---
layout: single
toc: true
title: "Span Queries"
permalink: /documentation/queries/spans
sidebar:
  nav: "docs"
---
Once the OddDotNet OpenTelemetry Test Harness has received `ExportTraceServiceRequest`s
from your application, the next step is to query the harness to confirm that
the correct spans have been received.

## SpanQueryService
The `SpanQueryService` handles queries for spans. Queries can be made using
gRPC or (eventually) HTTP requests. The `SpanQueryRequest` is used for every query
related to spans, and each request returns a `SpanQueryResponse`.

## SpanQueryResponse
Every query for spans will return a `SpanQueryResponse` message. The structure of that
message is:

```protobuf
message SpanQueryResponse {
  repeated Span spans = 1;
}
```

The list of spans could be empty if no spans match the filters you provide
within the timeout specified. 

## SpanQueryRequest
The `SpanQueryRequest` object is a simple message that encapsulates all the
query language needed to make a request. 

```protobuf
message SpanQueryRequest {
  repeated WhereSpanFilter filters = 1;
  Take take = 2;
  optional Duration duration = 3;
}
```

### Take
The `Take` property supports three options.

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

`TakeFirst` finds the first matching span based on the filters you provide. The
query will return immediately upon finding a match, with the matching span
contained in the `SpanQueryResponse` message.

`TakeAll` will continue to add spans that match your filters until the request
times out.

Finally, `TakeExact` has a single property `count` that instructs the test harness
to continue to match spans until it has reached the number specified in `count`.

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

### WhereSpanFilter
The `filters` property of the request defines the criteria used to match a span. There
are two types of filters for spans: "property" filters and "or" filters.

```protobuf
message WhereSpanFilter {
  oneof value {
    WhereSpanPropertyFilter spanProperty = 1;
    WhereSpanOrFilter spanOr = 2;
  }
}
```

Each filter has logic that evaluates to either true or false. If a filter is true, the
span currently being checked passes that filter and moves on to the next filter in line. 
If all filters pass, the span is considered a match and is included in the results.

#### WhereSpanPropertyFilter
The `WhereSpanPropertyFilter` provides the ability to check a specific property of a span
using the correct data type and a comparison that makes sense for that property. See the
concept docs around [Traces and Spans]({{ "/documentation/concepts/traces/" | relative_url }}) for more 
information around the properties available. 

#### WhereSpanOrFilter
When you are adding a `WhereSpanPropertyFilter` to the request, each filter is added as
an "AND". This is not always desireable, as you may want to check if property 'x' matches
"OR" property 'y' matches. The `WhereSpanOrFilter` provides this ability. 

```protobuf
message WhereSpanOrFilter {
  repeated WhereSpanFilter filters = 1;
}
```

The "OR" filter is just a list of filters, and it returns true if *any* of the filters
within return true.

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
Additional examples in other languages is on the roadmap and will be provided soon.

This example also assumes you have the OddDotNet test harness spun up and ready to accept
telemetry data. For ideas around how to do this, see the [Quick Starts]({{ "/quick-starts/" | relative_url }}).

```csharp
// ARRANGE
// Look for spans that have the name "GET /healthz"
var nameFilter = new WhereSpanPropertyFilter
{
  Name = new StringProperty
  {
    Compare = "GET /healthz",
    CompareAs = StringCompareAsType.Equals
  }
};

// Look for spans that were generated using the EFCore instrumentation library
var scopeFilter = new WhereSpanPropertyFilter
{
  InstrumentationScopeName = new StringProperty
  {
    Compare = "OpenTelemetry.Instrumentation.EntityFrameworkCore",
    CompareAs = StringCompareAsType.Equals
  }
};

// We want spans that match either of those, so we're going to add them to an OR filter
var orFilter = new WhereSpanOrFilter
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
Assert.Contains(response.Spans, span => span.Name == "GET /healthz");
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
        .AddSpanNameFilter("GET /healthz", StringCompareAsType.Equals)
        .AddInstrumentationScopeNameFilter("OpenTelemetry.Instrumentation.EntityFrameworkCore", StringCompareAsType.Equals);
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
Assert.Contains(response.Spans, span => span.Name == "GET /healthz");
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