---
layout: single
toc: true
title: "Span Queries"
permalink: /documentation/queries/spans
sidebar:
  nav: "docs"
---
The `Span`-specific properties that can be queried are listed below.

## SpanQueryRequest
```proto
message SpanQueryRequest {
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
    odddotnet.proto.common.v1.InstrumentationScopeFilter instrumentationScope = 3;
    odddotnet.proto.resource.v1.ResourceFilter resource = 4;
    odddotnet.proto.common.v1.StringProperty instrumentationScopeSchemaUrl = 5;
    odddotnet.proto.common.v1.StringProperty ResourceSchemaUrl = 6;
  }
}
```

### PropertyFilter
```proto
message PropertyFilter {
  oneof value {
    odddotnet.proto.common.v1.ByteStringProperty traceId = 1;
    odddotnet.proto.common.v1.ByteStringProperty spanId = 2;
    odddotnet.proto.common.v1.StringProperty traceState = 3;
    odddotnet.proto.common.v1.ByteStringProperty parentSpanId = 4;
    odddotnet.proto.common.v1.StringProperty name = 5;
    SpanKindProperty kind = 6;
    odddotnet.proto.common.v1.UInt64Property startTimeUnixNano = 7;
    odddotnet.proto.common.v1.UInt64Property endTimeUnixNano = 8;
    odddotnet.proto.common.v1.KeyValueProperty attribute = 9;
    odddotnet.proto.common.v1.UInt32Property droppedAttributesCount = 10;
    EventFilter event = 11;
    odddotnet.proto.common.v1.UInt32Property droppedEventsCount = 12;
    LinkFilter link = 13;
    odddotnet.proto.common.v1.UInt32Property droppedLinksCount = 14;
    StatusFilter status = 15;
    odddotnet.proto.common.v1.UInt32Property flags = 16;
  }
}
```

### EventFilter
```proto
message EventFilter {
  oneof value {
    odddotnet.proto.common.v1.UInt64Property timeUnixNano = 1;
    odddotnet.proto.common.v1.StringProperty name = 2;
    odddotnet.proto.common.v1.KeyValueProperty attribute = 3;
    odddotnet.proto.common.v1.UInt32Property droppedAttributesCount = 4;
  }
}
```

### LinkFilter
```proto
message LinkFilter {
  oneof value {
    odddotnet.proto.common.v1.ByteStringProperty traceId = 1;
    odddotnet.proto.common.v1.ByteStringProperty spanId = 2;
    odddotnet.proto.common.v1.StringProperty traceState = 3;
    odddotnet.proto.common.v1.KeyValueProperty attribute = 4;
    odddotnet.proto.common.v1.UInt32Property droppedAttributesCount = 5;
    odddotnet.proto.common.v1.UInt32Property flags = 6;
  }
}
```

### StatusFilter
```proto
message StatusFilter {
  reserved 1;

  oneof value {
    odddotnet.proto.common.v1.StringProperty message = 2;
    SpanStatusCodeProperty code = 3;
  }
}
```

## SpanQueryServiceClient
The `SpanQueryServiceClient` is automatically generated from the `.proto` files for the
`SpanQueryService` when you build for a client. Each language will handle this slightly
different. In C#, the following can be used:

```csharp
var channel = GrpcChannel.ForAddress(app.GetEndpoint("odddotnet", "grpc")); // If using .NET Aspire, or just a regular Uri
var spanQueryServiceClient = new SpanQueryService.SpanQueryServiceClient(channel);
```

The client contains two functions: Query, and Reset.

### Query
Passing a `SpanQueryRequest` to the Query endpoint will kick off the request and
return once all spans requested have been matched or the duration has timed out.

### Reset
In certain scenarios it might be necessary to clear the cache of spans, such as when
the traces generated from two separate tests may conflict with each other and the
tests must be ran sequentially. You may reset the list of spans between sequential
test runs so that each test is working with a blank slate.

See the [Query Overview]({{ "/documentation/queries/" | relative_url }}) section for more details around 
how the test harness caches spans.

## An Example
The following code, written in C#, shows how to manually build a request and send it
to the test harness. Naturally, the syntax in other languages will be slightly different.
Additional examples in other languages are on the roadmap and will be provided soon.

This example also assumes you have the OddDotNet test harness spun up and ready to accept
telemetry data. For ideas around how to do this, see the [Quick Starts]({{ "/quick-starts/" | relative_url }}).

```csharp
// ARRANGE
// Look for spans that have the name "GET /healthz"
var nameFilter = new PropertyFilter
{
  Name = new StringProperty
  {
    Compare = "GET /healthz",
    CompareAs = StringCompareAsType.Equals
  }
};

// Look for spans that were generated using the EFCore instrumentation library
var scopeFilter = new InstrumentationScopeFilter
{
  Name = new StringProperty
  {
    Compare = "OpenTelemetry.Instrumentation.EntityFrameworkCore",
    CompareAs = StringCompareAsType.Equals
  }
};

// We want spans that match either of those, so we're going to add them to an OR filter
var orFilter = new OrFilter
{
  Filters = { nameFilter, scopeFilter }
};

// Create the request
var request = new SpanQueryRequest
{
  Take = new Take
  {
    TakeFirst = new TakeFirst() // Take the first span found
  },
  Duration = new Duration
  {
    Milliseconds = 1000 // Wait up to 1 second to find the span
  },
  Filters = { orFilter } // Add our OR filter to it (which contains the two property filters)
};

// ACT
await TriggerWorkflowThatGeneratesSpans();

// ASSERT
var channel = GrpcChannel.ForAddress("http://localhost:4317");
var client = new SpanQueryService.SpanQueryServiceClient(channel);

SpanQueryResponse response = await client.QueryAsync(request);

// Make some assertions on the spans returned
Assert.Contains(response.Spans, span => span.InstrumentationScope.Name == "OpenTelemetry.Instrumentation.EntityFrameworkCore");
Assert.Contains(response.Spans, span => span.Span.Name == "GET /healthz");
```

## SpanQueryRequestBuilder
The above code snippet is rather verbose. Because the `SpanQueryRequest` is 
defined as a message in protobuf, we're limited in the convenience methods
available. The `SpanQueryRequestBuilder` can simplify the process of building
queries.

### CSharp
The [OddDotNet.Client](https://www.nuget.org/packages/OddDotNet.Client/) NuGet
package can be added to your project to leverage the pre-built gRPC client and
the `SpanQueryRequestBuilder`. 

The builder can streamline the generation of requests to the `SpanQueryService`.
the above snippet of code in `An Example` can be simplified using the builder.

```csharp
// ARRANGE
var request = new SpanQueryRequestBuilder()
  .TakeFirst() // Take the first span found
  .Wait(TimeSpan.FromSeconds(1)) // Wait up to 1 second to find the span
  .Where(filters => 
  {
    filters.AddOrFilter(orFilters => 
    {
      orFilters
        .AddNameFilter("GET /healthz", StringCompareAsType.Equals)
        .InstrumentationScope.AddNameFilter("OpenTelemetry.Instrumentation.EntityFrameworkCore", StringCompareAsType.Equals);
    })
  })
  .Build();

// ACT
await TriggerWorkflowThatGeneratesSpans();

// ASSERT
var channel = GrpcChannel.ForAddress("http://localhost:4317");
var client = new SpanQueryService.SpanQueryServiceClient(channel);

SpanQueryResponse response = await client.QueryAsync(request);

// Make some assertions on the spans returned
Assert.Contains(response.Spans, span => span.InstrumentationScope.Name == "OpenTelemetry.Instrumentation.EntityFrameworkCore");
Assert.Contains(response.Spans, span => span.Span.Name == "GET /healthz");
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