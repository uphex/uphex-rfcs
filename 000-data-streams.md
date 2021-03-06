# Data Streams

## Summary

Remove coupling between observations and data fetched from providers by
introducing the idea of data streams.

## Motivation

There is a direct relationship between what is fetched from providers and what
is saved as observation. This model works well for metrics that involve a single
data source (e.g. Facebook page visits), but doesn't allow for multi-source
metrics (e.g. Shopify revenue by number of Twitter followers).

In addition, data fetched from providers do not necessarily map one-to-one to
observations, and even though we have the ability to further process it before
saving observations, we can't reuse it to compose multidimensional metrics (e.g.
new Twitter followers by day, Facebook page visits by day/week/month).

Treating fetched data and saved observations as distinct entities addresses the
shortcomings of our current model. This new model can be achieved by introducing
data streams and sources.

## Design

### Current Model

- **Channels** are collections of metrics provided by an external service and
  grouped by an API identifier;

- **Metrics** are series of observations;

- **Observations** are products of processing data points fetched from a metric;

- **Predictions** are values computed from a time series composed of
  observations.

### Proposed Model

- **Sources** emit _Data Points_. They are the origin of all data consumed by
  the system. _Data Points_ emitted from a _Source_ cav have different semantics
  (e.g interval aggregates, snapshots, discrete);

- **Data Points** represent discrete input points persisted to the database with
  minimal processing;

- **Streams** represent series of _Data Points_ originated at either a _Source_
  or another _Stream_. From this perspective, a _Source_ is indistinguishable
  from a _Stream_;

- **Observations** represent values computed from a time series extracted from a
  _Stream_. They are also persisted to the database.

Some of the ideas on the new model exist on the current model but receive a new
name in order to avoid confusion.

The structure of the proposed model looks like this:


    +-------------------+      +-------------------+
    |      SOURCE       | <--- |    DATA POINTS    |
    +-------------------+      +-------------------+
              |
              |
              V
    +-------------------+      +-------------------+      +------------
    |      STREAM       |      |      STREAM       | <--- |     SOUR...
    +-------------------+      +-------------------+      +------------
              |                         |
              |-------------------------+
              V
    +-------------------+
    |      STREAM       |
    +-------------------+
              |
              |
              V
    +-------------------+
    |    OBSERVATIONS   |
    +-------------------+


_Data Points_ are created by recurring fetch jobs and happen outside of the Data
Streams model. _Sources_ are used to pull _Data Points_ out of the database into
a _Stream_.

_Streams_ and _Sources_ share the same interface (e.g. Enumerable). A _Source_
can exist by itself while a _Stream_ cannot exist without a _Source_.

_Sources_ are, in practice, abstracted queries to the _Data Points_ database
table.

_Streams_ are the most complex entities in this model. They are initialized with
a set of _Streams_ and/or _Sources_ and the relationships between them.

## Implementation

### Data Points

_Data Points_ are analogous to observations in the current model. They are
stored in the database with the following schema:

```ruby
create_table :data_points do |t|
  t.jsonb      :time
  t.jsonb      :value
  t.references :metadata
end
```

In addition to their time and value data, a _Data Point_ has metadata associated
with it which describes the meaning of the other two fields plus any other
information common to a collection of data points (e.g. channel, metric, API
identifiers, etc.)

Channels and metrics from the current model can serve as metadata for _Data
Points_. This part of the model remains to be discussed.

### Sources

_Sources_ are implemented in code, and they use the metadata described in
_Data Points_ to fetch and make sense out of the data stored in the database.

A _Source_ may look like this:

```ruby
class TwitterFollowers < Source
  def initialize(metadata)
    # ...
  end

  def type
  end

  def each
  end
end

# >> source = TwitterFollowers.new(handle: "@uphex")
# => #<TwitterFollowers:0xABCDEF>
# >> source.each.to_a
# => [{time: "2015-03-03T10:20:30Z", value: 45}, ...]
```

### Streams

_Streams_ are "middleware". They wrap around _Sources_ and other _Streams_ so
they also exist in code. Here is an example:

```ruby
class TwitterFollowersByDay
  def initialize(input)
  end

  def type
  end

  def each
  end
end

# >> stream = TwitterFollowersByDay.new(source)
# => #<TwitterFollowersByDay:0xABCDEF>
# >> stream.each.to_a
# => [{day: "2015-03-03", value: 4}, {day: "2015-03-04", value: -2}
```

The idea behind _Streams_ is that the system knows about a predefined set of
stream types and how to operate on them.

Among the possible stream types and operations there are:

| Operation | Input                     | Output         |  Examples            |
|:---------:|:-------------------------:|:--------------:|:--------------------:|
|delta      | snapshot                  |  interval:(x)  | Followers by day     |
|agreggate  | discrete                  |  interval:(x)  | Sales by day         |
|agreggate  | interval:(x)              |  interval:(y)  | Monthly page visits  |
|ratio      | interval:(x),interval:(x) |  interval:(x)  | Revenue by followers |

## Challenges

### Stream Discovery & Combination

The number of available data streams changes as the user adds new data sources.

A UI for combining data sources and streams is displayed each time a user adds a
new source. The same UI is used to create streams when not adding a data source.
The UI takes a list of all possible combinations and lets users to select/filter
streams. (After adding a data source, the new inputs are automatically selected
for convenience.)

A Stream discover class is responsible for collecting input data sources &
streams and listing all possible stream combinations.

Each new stream known to the system is saved to the database. When a user "adds"
a stream, a new association between a user and a stream is created.

Similarly, each new data source is saved to the database and associated with a
user when "added".

### Stream-Source Dependency

Consider whether streams should depende on their bottom level data sources.
This would ensure a stream can actually be fetched without computing the stream
pipeline. An array column with the data source ids can be used used.

### Model Migration

Channels and metrics are essentially merged into data sources. Metrics are
moved to code under lib/provider. Data sources become associated with a user and
an access token.

Table user_channels become user_data_streams.

Updating channels is no longer needed. Each time the stream selection UI is
rendered, a CombineStreams job is dispatched to add all new streams known to the
system.

Existing observations are migrate to input data points and converted to
observations once streams are in place.
