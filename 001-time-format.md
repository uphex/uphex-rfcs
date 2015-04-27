# Source Point Time Format

## Summary

Standardize the format of time information in source points to facilitate operations.

## Motivation

The time data of source points is stored according to its type:

- Instant/snapshot points: hashmap with a single key "time";
- Interval points: hashmap with keys "start" and "end".

In order to operate on source points, it is required to either know in advance how the time information is stored or to [handle each different format](http://git.io/vf7GI).

Operations such as sorting and selecting a subset of points could benefit from a standardized format.

## Design

The time information of source points can be reduced to a single timestamp plus some additional metadata:

- Instant/snapshot points: just a timestamp, no additional metadata required;
- Interval points: a single timestamp and a duration metadata.

By giving a common name to the timestamp key of the time hashmap regardless of the source point type, it is possible to perform the operations described above without knowing the source point type in advance or handling all possible types.

## Implementation

Example of an instant source point with standardized time format:

```ruby
instant_point  = {value: {...}, time: {stamp: "2015-04-04T22:23:24Z"}}
interval_point = {value: {...}, time: {stamp: "2015-05-05T00:00:00Z", duration: 86400}}
```

The "standardization" is performed at the provider client layer before saving the source point.
