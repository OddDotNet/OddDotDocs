---
layout: single
title: "Queries"
toc: true
toc_sticky: true
permalink: /documentation/queries/
sidebar:
  nav: "docs"
---
Once the OddDotNet OpenTelemetry Test Harness has received an `Export{SIGNAL}ServiceRequest`
from your application (where `SIGNAL` is either `Trace` - see note, `Metric`, or `Log`), the next step 
is to query the harness to confirm that the correct signals have been received.

***NOTE:*** OddDotNet uses `Trace` and `Span` interchangeably under most circumstances. However,
since the return type of `SpanQueryService` is a `Span` and not a full `Trace`, `Span` is used
instead. The name of the request to ingest a trace/span is `ExportTraceServiceRequest`, and the
name of the service used to query an individual span is `SpanQueryService`. Under most 
circumstances, you will not see the `Export{SIGNAL}ServiceRequest`, so the `SIGNAL` values for
`{SIGNAL}QueryService` are `Span`, `Metric`, and `Log`.
{: .notice--info}

## Signal Caching
All signals are cached by OddDotNet for 30 seconds. A background service is responsible for cleaning
up "expired" signals on a 1-second interval.

The purpose of signal caching is to allow for querying of signals that may have already been 
received by OddDotNet before the query begins. For example:

```csharp
// generate some telemetry data as part of a request
await SomeWorkflowThatGeneratesSignals();

// Signals are generated asynchronously, but some time may pass before the query begins
await Task.Delay(TimeSpan.FromSeconds(1));

// Now query for the signal
var response = await client.QueryAsync(request);
```

Without signal caching, the above code would miss any signals that came in during the 1 second
delay between generating the signals and querying for those signals. 

When a query is made, the previous 30 seconds of signals are checked in cache first to see if any 
matches are found. If they are, they are included in the results. The query then continues to 
check more signals as they come in, up to the `Take` and `Duration` values specified.

Both the signal expiration and the cleanup interval are configurable, although care should be taken
when modifying these values as they can have a direct impact on performance. See 
[ODD_CACHE_CLEANUP_INTERVAL]({{ "/documentation/concepts/test-harness/#odd_cache_cleanup_interval-ms" | relative_url }}) 
and [ODD_CACHE_EXPIRATION]({{ "/documentation/concepts/test-harness/#odd_cache_expiration-ms" | relative_url }}) for details.

## {SIGNAL}QueryService
The `{SIGNAL}QueryService` handles queries for the specified signal (`Span`, `Metric`, `Log`). 
Queries can be made using gRPC or (eventually) HTTP requests. The `{SIGNAL}QueryRequest` is used 
for every query related to a signal, and each request returns a `{SIGNAL}QueryResponse`.

## {SIGNAL}QueryResponse
Every query for a signal will return a `{SIGNAL}QueryResponse` message. The structure of that
message is (using `Span`):

```protobuf
message SpanQueryResponse {
  repeated FlatSpan spans = 1;
}

message FlatSpan {
    opentelemetry.proto.trace.v1.Span span = 1;
    opentelemetry.proto.resource.v1.Resource resource = 2;
    opentelemetry.proto.common.v1.InstrumentationScope instrumentation_scope = 3;
    string resource_schema_url = 4;
    string instrumentation_scope_schema_url = 5;
}
```
The signal is de-normalized or "flattened", hence the `Flat{SIGNAL}` name. The
OpenTelemetry data structure for a signal includes the resource that created the signal and the
instrumentation library that generated the signal. To avoid traversing this data structure as 
part of OddDotNet testing, the resource and the scope are de-normalized.

The list of signals could be empty if no signals match the filters you provide
within the timeout specified. 

## {SIGNAL}QueryRequest
The `SIGNALQueryRequest` object is a simple message that encapsulates all the
query language needed to make a request. 

```protobuf
message SpanQueryRequest {
  repeated Where filters = 1;
  odddotnet.proto.common.v1.Take take = 2;
  optional odddotnet.proto.common.v1.Duration duration = 3;
}

message MetricQueryRequest {
  repeated Where filters = 1;
  odddotnet.proto.common.v1.Take take = 2;
  optional odddotnet.proto.common.v1.Duration duration = 3;
}

message LogQueryRequest {
  repeated Where filters = 1;
  odddotnet.proto.common.v1.Take take = 2;
  optional odddotnet.proto.common.v1.Duration duration = 3;
}
```

### Take
The `Take` property supports three options. `Take` is optional, so if no value
is supplied, the query will default to `TakeFirst`.

```protobuf
message Take {
  oneof value {
    TakeFirst takeFirst = 1;
    TakeAll takeAll = 2;
    TakeExact takeExact = 3;
  }
}

message TakeFirst {}

message TakeAll {}

message TakeExact {
  int32 count = 1;
}
```

`TakeFirst` finds the first matching signal based on the filters you provide. The
query will return immediately upon finding a match, with the matching signal
contained in the `{SIGNAL}QueryResponse` message.

`TakeAll` will continue to add signals that match your filters until the request
times out.

Finally, `TakeExact` has a single property `count` that instructs the test harness
to continue to match signals until it has reached the number specified in `count`.

### Duration
Duration specifies how long to wait *after the query has started* before returning
the results.

```protobuf
message Duration {
  int32 milliseconds = 1;
}
```

Duration is optional. If a value is not supplied, the query will continue to match spans
until the `Take` amount has been met. When supplying a duration, the value is in milliseconds. 

If a duration value is supplied, then the query will return either when the `Take`
criteria has been met, or when the `Duration` is met, whichever is first. This means,
for example, that a query with a `TakeExact(100)` might not return all 100 if
matches are not found within the `Duration`.

### Where
The `filters` property of the request defines the criteria used to match a signal. For the list
of `SIGNAL`-specific filters, see the corresponding query page:

* [Spans]({{ "/documentation/queries/spans/" | relative_url }})
* [Metrics]({{ "/documentation/queries/metrics/" | relative_url }})
* [Logs]({{ "/documentation/queries/logs/" | relative_url }})

Here is the `Where` definition for Metrics, as an example:

```proto
message Where {
    oneof value {
        PropertyFilter property = 1;
        OrFilter or = 2;
        odddotnet.proto.common.v1.InstrumentationScopeFilter instrumentation_scope = 3;
        odddotnet.proto.resource.v1.ResourceFilter resource = 4;
        odddotnet.proto.common.v1.StringProperty instrumentation_scope_schema_url = 5;
        odddotnet.proto.common.v1.StringProperty resource_schema_url = 6;
    }
}
```

Each filter has logic that evaluates to either true or false. If a filter is true, the
signal currently being checked passes that filter and moves on to the next filter in line. 
If all filters pass, the span is considered a match and is included in the results.

Each signal contains the same values that can be queried:
* PropertyFilter - the properties available are specific to each signal
* OrFilter
* InstrumentationScope - for querying details related to the instrumentation library used to create the signal
* Resource - for querying details related to the resource that created the signal
* InstrumentationScopeSchemaUrl
* ResourceSchemaUrl

#### PropertyFilter
The `PropertyFilter` provides the ability to check a specific property of a signal
using the correct data type and a comparison that makes sense for that property.

Notice that the `PropertyFilter` contains a map of possible values that are either additional
`Filter` types or some data type `Property`. When querying a signal, the goal is to build 
a query that reduces down to a single property that is being queried. Take metrics as an example.

The `PropertyFilter` of the `MetricQueryRequest.Where` includes a filter for `Gauge`:

```proto
message PropertyFilter {
    reserved 4, 6, 8;
    oneof value {
        ...
        GaugeFilter gauge = 5;
        ...
    }
}
```

Because `Gauge` is a `GaugeFilter` and not a `GaugeProperty`, this indicates that there are additional
levels that can be queried. Here's the `GaugeFilter`:

```proto
message GaugeFilter {
    oneof value {
        NumberDataPointFilter data_point = 1;
    }
}
```

Once again, the only property of the `GaugeFilter` is itself another filter, the `NumberDataPointFilter`.
This filter includes a `UInt64Property` called `StartTimeUnixNano`. 

```proto
message NumberDataPointFilter {
    reserved 1;
    oneof value {
        ...
        odddotnet.proto.common.v1.UInt64Property start_time_unix_nano = 2;
        ...
    }
}
```

To query for this property with a value of 123, the request would look like the following:

```json
{
  "filters": [
    {
      "property": {
        "gauge": {
          "dataPoint": {
            "startTimeUnixNano": {
              "compareAs": "NUMBER_COMPARE_AS_TYPE_EQUALS",
              "compare": 123
            }
          }
        }
      }
    }
  ]
}
```

#### OrFilter
When you are adding a `PropertyFilter` to the request, each filter is added as
an "AND". This is not always desireable, as you may want to check if property 'x' matches
"OR" property 'y' matches. The `OrFilter` provides this ability. 

```protobuf
message OrFilter {
  repeated Where filters = 1;
}
```

The "OR" filter is just a list of filters, and it returns true if *any* of the filters
within return true.

#### Generic Filters
Each signal contains various filters that are generic to all signals. 

##### InstrumentationScopeFilter
All signals are generated using an instrumentation library. The 
`InstrumentationScopeFilter` allows for querying properties
related to the instrumentation scope.

```proto
message InstrumentationScopeFilter {
  oneof value {
    odddotnet.proto.common.v1.StringProperty name = 1;
    odddotnet.proto.common.v1.KeyValueListProperty attributes = 2;
    odddotnet.proto.common.v1.StringProperty version = 3;
    odddotnet.proto.common.v1.UInt32Property dropped_attributes_count = 4;
  }
}
```

##### ResourceFilter
All signals are associated with a resource when they are created. This
resource can be queried using the `ResourceFilter`.

```proto
message ResourceFilter {
  oneof value {
    odddotnet.proto.common.v1.KeyValueListProperty attributes = 1;
    odddotnet.proto.common.v1.UInt32Property dropped_attributes_count = 2;
  }
}
```

#### Generic Properties
The intent behind all filters is to narrow down your query to a specific property. Some
properties are specific to a signal, such as the `SpanKindProperty` of a span. Most
properties, however, are simple data types.

##### StringProperty
Allows for querying of string data types. The property expects the string to be compared
and the comparison to perform.

```proto
message StringProperty {
  StringCompareAsType compare_as = 1;
  optional string compare = 2;
}

enum StringCompareAsType {
  STRING_COMPARE_AS_TYPE_NONE_UNSPECIFIED = 0;
  STRING_COMPARE_AS_TYPE_EQUALS = 1;
  STRING_COMPARE_AS_TYPE_NOT_EQUALS = 2;
  STRING_COMPARE_AS_TYPE_CONTAINS = 3;
  STRING_COMPARE_AS_TYPE_NOT_CONTAINS = 4;
  STRING_COMPARE_AS_TYPE_IS_EMPTY = 5;
  STRING_COMPARE_AS_TYPE_IS_NOT_EMPTY = 6;
}
```

##### ByteStringProperty
Similar to a `StringProperty`, allows for querying of `ByteString` data types such as 
`traceId` and `spanId`. Unlike strings, `Contains` and `DoesNotContain` are not included
as `ByteString` comparisons are typically all or none.

```proto
message ByteStringProperty {
  ByteStringCompareAsType compare_as = 1;
  optional bytes compare = 2;
}

enum ByteStringCompareAsType {
  BYTE_STRING_COMPARE_AS_TYPE_NONE_UNSPECIFIED = 0;
  BYTE_STRING_COMPARE_AS_TYPE_EQUALS = 1;
  BYTE_STRING_COMPARE_AS_TYPE_NOT_EQUALS = 2;
  BYTE_STRING_COMPARE_AS_TYPE_EMPTY = 3;
  BYTE_STRING_COMPARE_AS_TYPE_NOT_EMPTY = 4;
}
```

##### BoolProperty
Allows for querying bool property types.

```proto
message BoolProperty {
  BoolCompareAsType compare_as = 1;
  optional bool compare = 2;
}

enum BoolCompareAsType {
  BOOL_COMPARE_AS_TYPE_NONE_UNSPECIFIED = 0;
  BOOL_COMPARE_AS_TYPE_EQUALS = 1;
  BOOL_COMPARE_AS_TYPE_NOT_EQUALS = 2;
}
```

##### Number Properties
The following numeric data types are supported: `ulong|UInt64`, `uint|UInt32`,
`long|Int64`, `int|Int32`, and `double`. They all use the same `NumberCompareAsType`.

```proto
message UInt64Property {
  NumberCompareAsType compare_as = 1;
  optional fixed64 compare = 2;
}
  
message UInt32Property {
  NumberCompareAsType compare_as = 1;
  optional uint32 compare = 2;
}
  
message Int64Property {
  NumberCompareAsType compare_as = 1;
  optional int64 compare = 2;
}
  
message Int32Property {
  NumberCompareAsType compare_as = 1;
  optional int32 compare = 2;
}
  
message DoubleProperty {
  NumberCompareAsType compare_as = 1;
  optional double compare = 2;
}

enum NumberCompareAsType {
  NUMBER_COMPARE_AS_TYPE_NONE_UNSPECIFIED = 0;
  NUMBER_COMPARE_AS_TYPE_EQUALS = 1;
  NUMBER_COMPARE_AS_TYPE_NOT_EQUALS = 2;
  NUMBER_COMPARE_AS_TYPE_GREATER_THAN = 3;
  NUMBER_COMPARE_AS_TYPE_GREATER_THAN_EQUALS = 4;
  NUMBER_COMPARE_AS_TYPE_LESS_THAN = 5;
  NUMBER_COMPARE_AS_TYPE_LESS_THAN_EQUALS = 6;
}
```

##### Enum Properties
Signals contain enum types, such as the `SpanKind` enum property for spans,
or the `SeverityNumber` of a log message. Each enum property contains
the property being checked, and an `EnumCompareAsType` to define the 
comparison.

```proto
enum EnumCompareAsType {
  ENUM_COMPARE_AS_TYPE_NONE_UNSPECIFIED = 0;
  ENUM_COMPARE_AS_TYPE_EQUALS = 1;
  ENUM_COMPARE_AS_TYPE_NOT_EQUALS = 2;
}
```

##### Collections
Querying becomes significantly more complicated when dealing with collections.
Collections are typically associated with `Metadata` of a metric, the `Body`
of a log message, or the `Attributes` of any of the signals. 

When querying a collection, you first specify that the data type of the property
*IS* a collection. Let's take `Body` as an example:

```proto
odddotnet.proto.common.v1.AnyValueProperty body = 5;
```

The `Body` of a log message is defined as an `AnyValue`. This message type
is defined by OpenTelemetry as:

```proto
message AnyValue {
  oneof value {
    string string_value = 1;
    bool bool_value = 2;
    int64 int_value = 3;
    double double_value = 4;
    ArrayValue array_value = 5;
    KeyValueList kvlist_value = 6;
    bytes bytes_value = 7;
  }
}
```

So the `Body` property could be a collection of type `ArrayValue` or `KeyValueList`,
but it could also just as well be a `string` instead. Therefore, you must first
verify that the property you're checking is a collection, and then you can filter
on the contents of that collection. Data type mismatches, such as saying an `AnyValue`
property is a string when it's really a collection, will cause the filter to return
false. Be sure to verify what data type your `AnyValue` property is so that the filter
matches when found.

When performing this comparison, OddDotNet will very that *each and every* item you have
listed in your filter exists in the signal being checked. In other words, performing
`ArrayValue` and `KeyValueList` comparisons will check to confirm that the collection you
have specified is a *subset* of the signal's collection.

The following query will confirm that the `Body` of a log signal is a `KeyValueList`, and
that it contains a `KeyValue` where the `key` is `"test"` and the value is an `AnyValue`
with a `double` property equal to `123.0`.

```csharp
// Create a new Where filter
var filter = new Where
{
  Property = new PropertyFilter // We're interested in a property of the Log
  {
    Body = new AnyValueProperty // We're looking for the Body, which is an AnyValueProperty
    {
      KvlistValue = new KeyValueListProperty // We believe the data type of the Body is a KeyValueList
      {
        Values = // And we want to match if the following is present
        {
          new KeyValueProperty
          {
            Key = "test" // A KeyValue whose key is "test",
            Value = new AnyValueProperty
            {
              DoubleValue = new DoubleProperty // and whose Value is a Double...
              {
                CompareAs = NumberCompareAsType.Equals, // that is equal to...
                Compare = 123.0 // this double value
              }
            }
          }
        }
      }
    }
  }
}
```

Given the above filter, this log would match:
```json
{
  "body": {
    "kvlistValue": {
      "values": [
        {
          "key": "test",
          "value": {
            "doubleValue": 123.0
          }
        }
      ]
    }
  }
}
```

This log would not match, because the body is the wrong type:
```json
{
  "body": {
    "stringValue": "Some unstructured log message"
  }
}
```

This log also would not match, because the key is different:
```json
{
  "body": {
    "kvlistValue": {
      "values": [
        {
          "key": "other",
          "value": {
            "doubleValue": 123.0
          }
        }
      ]
    }
  }
}
```

OddDotNet allows for nesting, so if you have a property that is, say, an `ArrayValue`
inside of another `ArrayValue`, they will be handled appropriately.

###### AnyValueProperty
`KeyValueProperty` and `ArrayValueProperty` reference `AnyValueProperty`. As it's name implies, 
properties of this type can hold one of a number of different data types.

```proto
message AnyValueProperty {
  oneof value {
    StringProperty string_value = 1;
    BoolProperty bool_value = 2;
    Int64Property int_value = 3;
    DoubleProperty double_value = 4;
    ArrayValueProperty array_value = 5;
    KeyValueListProperty kvlist_value = 6;
    ByteStringProperty byte_string_value = 7;
  }
}
```

###### ArrayValueProperty
These are simply repeated `AnyValueProperty`. Each item added to this property
must exist in the signal property being checked.

```proto
message ArrayValueProperty {
  repeated AnyValueProperty values = 1;
}
```

###### KeyValue Properties
The same applies to a `KeyValueListProperty`: each item add to the filter's
list must exist in the signal being checked.
```proto
message KeyValueListProperty {
  repeated KeyValueProperty values = 1;
}

message KeyValueProperty {
  string key = 1;
  AnyValueProperty value = 2;
}
```