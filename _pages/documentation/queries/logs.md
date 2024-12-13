---
layout: single
toc: true
toc_sticky: true
title: "Log Queries"
permalink: /documentation/queries/logs
sidebar:
  nav: "docs"
---
The `LogRecord`-specific properties that can be queried are listed below.

## LogQueryRequest
```proto
message LogQueryRequest {
  repeated Where filters = 1;
  odddotnet.proto.common.v1.Take take = 2;
  optional odddotnet.proto.common.v1.Duration duration = 3;
}
```

### Where
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

### PropertyFilter
```proto
message PropertyFilter {
  reserved 4;
  oneof value {
    odddotnet.proto.common.v1.UInt64Property time_unix_nano = 1;
    odddotnet.proto.common.v1.UInt64Property observed_time_unix_nano = 11;
    SeverityNumberProperty severity_number = 2;
    odddotnet.proto.common.v1.StringProperty severity_text = 3;
    odddotnet.proto.common.v1.AnyValueProperty body = 5;
    odddotnet.proto.common.v1.KeyValueListProperty attributes = 6;
    odddotnet.proto.common.v1.UInt32Property dropped_attributes_count = 7;
    odddotnet.proto.common.v1.UInt32Property flags = 8;
    odddotnet.proto.common.v1.ByteStringProperty trace_id = 9;
    odddotnet.proto.common.v1.ByteStringProperty span_id = 10;
  }
}
```

### SeverityNumberProperty
```proto
message SeverityNumberProperty {
  odddotnet.proto.common.v1.EnumCompareAsType compare_as = 1;
  opentelemetry.proto.logs.v1.SeverityNumber compare = 2;
}
```

## An Example
The following code, written in C#, shows how to manually build a request and send it
to the test harness.

This example assumes you have the OddDotNet test harness spun up and ready to accept
telemetry data. For ideas around how to do this, see the [Quick Starts]({{ "/quick-starts/" | relative_url }}).

```csharp
// ARRANGE
// Look for any logs associated with a specific TraceId
byte[] traceId = ...; // Some TraceId
var traceIdFilter = new PropertyFilter
{
  TraceId = new ByteStringProperty
  {
    CompareAs = ByteStringCompareAsType.Equals,
    Compare = traceId
  }
};

// Create the request
var request = new LogQueryRequest
{
  Take = new Take
  {
    TakeAll = new TakeAll()
  },
  Duration = new Duration
  {
    Milliseconds = 1000 // Wait up to 1 second to allow logs to flow in
  },
  Filters = { traceIdFilter } // Add our filter
};

// ACT
await TriggerWorkflowThatGeneratesLogs();

// ASSERT
var channel = GrpcChannel.ForAddress("http://localhost:4317");
var client = new LogQueryService.LogQueryServiceClient(channel);

LogQueryResponse response = await client.QueryAsync(request);

// Make some assertions on the logs returned
Assert.NotEmpty(response.Logs);
```

## LogQueryRequestBuilder
The above code snippet is rather verbose. Because the `LogQueryRequest` is 
defined as a message in protobuf, we're limited in the convenience methods
available. The `LogQueryRequestBuilder` can simplify the process of building
queries.

### CSharp
The [OddDotNet.Client](https://www.nuget.org/packages/OddDotNet.Client/) NuGet
package can be added to your project to leverage the pre-built gRPC client and
the `LogQueryRequestBuilder`. 

The builder can streamline the generation of requests to the `LogQueryService`.
the above snippet of code in `An Example` can be simplified using the builder.

```csharp
// ARRANGE
var request = new LogQueryRequestBuilder()
  .TakeAll() // Take all logs within the timeframe
  .Wait(TimeSpan.FromSeconds(1)) // Wait up to 1 second to allow logs to flow in
  .Where(filters => 
  {
    filters.AddTraceIdFilter(traceId, ByteStringCompareAsType.Equals); // Add our TraceId filter
  })
  .Build();

// ACT
await TriggerWorkflowThatGeneratesLogs();

// ASSERT
var channel = GrpcChannel.ForAddress("http://localhost:4317");
var client = new LogQueryService.LogQueryServiceClient(channel);

LogQueryResponse response = await client.QueryAsync(request);

// Make some assertions on the logs returned
Assert.NotEmpty(response.Logs);
```

### Java
Not yet built. On the roadmap.

### Go
Not yet built. On the roadmap.

### Other languages
Support for other languages is not currently on the roadmap, but it is desired.
Check back often to see the progress on your favorite language, or create a 
[Discussion](https://github.com/OddDotNet/OddDotNet/discussions) to advocate for the next supported language.

Remember, this is just for the `Builder`. Any language that supports gRPC and
protobuf (and eventually HTTP and JSON) can use the test harness, it just won't
have the convenience of the `Builder`, so queries will need to be constructed manually,
as outlined in `An Example` above.