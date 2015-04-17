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
shortcomings of our current model. This new model can be achieved by introcuding
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

- **Data Points** represent discrete values persisted to the database with
  minimal processing;

- **Streams** represent series of _Data Points_ originated at either a _Source_
  or another _Stream_. From this perspective, a _Source_ is indistinguishable
  from a _Stream_;

- **Predictions** represent values computed from a time series extracted from a
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
    |    PREDICTIONS    |
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

_Predictions_ are equivalent to a combination of observations and predictions
from the existing model.

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

### Predictions

_Predictions_ extend predictions from our current model by including the actual
observed value with the computed prediction. This is needed because the data
flowing through streams is transient. Effectively, we store the raw _Data
Points_ and the fully processed _Predictions_.

In addition to the observed values, _Predictions_ also store metadata describing
the stream pipeline used to process raw data into predictable observations.

This is the schema for _Predictions_:

```ruby
create_table :predictions do |t|
  t.jsonb      :time
  t.jsonb      :observed_value
  t.jsonb      :predicted_value
  t.jsonb      :lower_limit_value
  t.jsonb      :upper_limit_value
  t.references :metadata
end
```

Channels and metrics from the current model can serve as metadata for
_Predictions_. This part of the model remains to be discussed.
