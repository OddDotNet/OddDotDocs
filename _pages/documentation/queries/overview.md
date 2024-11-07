---
layout: single
title: "Queries"
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
    opentelemetry.proto.common.v1.InstrumentationScope instrumentationScope = 3;
    string resourceSchemaUrl = 4;
    string instrumentationScopeSchemaUrl = 5;
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
* Logs - Coming Soon

Here is the `Where` definition for Metrics, as an example:

```proto
message Where {
    oneof value {
        PropertyFilter property = 1;
        OrFilter or = 2;
        odddotnet.proto.common.v1.InstrumentationScopeFilter instrumentationScope = 3;
        odddotnet.proto.resource.v1.ResourceFilter resource = 4;
        odddotnet.proto.common.v1.StringProperty instrumentationScopeSchemaUrl = 5;
        odddotnet.proto.common.v1.StringProperty ResourceSchemaUrl = 6;
    }
}
```

Each filter has logic that evaluates to either true or false. If a filter is true, the
span currently being checked passes that filter and moves on to the next filter in line. 
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

The `PropertyFilter` of the `MetricQueryRequest.Wnere` includes a filter for `Gauge`:

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
        NumberDataPointFilter dataPoint = 1;
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
        odddotnet.proto.common.v1.UInt64Property startTimeUnixNano = 2;
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

#### InstrumentationScopeFilter
All signals, are generated using an instrumentation library. The 
`InstrumentationScopeFilter` allows for querying properties
related to the instrumentation scope.

#### ResourceFilter
All signals, are associated with a resource when they are created. This
resource can be queried using the `ResourceFilter`.

#### InstrumentationScopeSchemaUrl
This property is associated with the `Scope{SIGNAL}` message, which is a combination of the
instrumentation scope and the list of signals. It is listed as a separate property since it
is not directly part of the `InstrumentationScope` message.

#### ResourceSchemaUrl
This property is associated with the `Resource{SIGNAL}` message, which is a combination of the
resource and the list of `Scope{SIGNAL}`. It is listed as a separate property since it
is not directly part of the `Resource` message.