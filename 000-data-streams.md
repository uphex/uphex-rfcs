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

_Data Points_ are very similar to observations in the current model.

[Consider merging metrics with observations]

[Consider storing all data in this table (including provider name and API
identifier) so it exists independent of the rest of the system]

### Sources

### Streams

### Predictions

