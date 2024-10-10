---
layout: single
title: "Traces and Spans"
permalink: /documentation/concepts/traces/
sidebar:
  nav: "docs"
---
Before you dive in to the "traces and spans" concepts outlined below, review
these concepts in the official OpenTelemetry documentation, found
[here](https://opentelemetry.io/docs/concepts/signals/traces/).

# The ExportTraceServiceRequest
All OpenTelemetry collectors receive OTLP traces from applications in the form
of an `ExportTraceServiceRequest`. This is a well known message, and its
structure can be found in the official 
[opentelemetry-proto](https://github.com/open-telemetry/opentelemetry-proto/blob/main/opentelemetry/proto/collector/trace/v1/trace_service.proto) repository. The diagram below provides the general structure of 
this message (some properties omitted for brevity).

![ExportTraceServiceRequest diagram]({{ "/assets/diagrams/ExportTraceServiceRequest/ExportTraceServiceRequest.svg" | relative_url }})

When the OpenTelemetry Collector exports traces to an endpoint, it typically batches those traces up and
sends them by resource (eg. a microservice), hence the list of resources in an `ExportTraceServiceRequest`.
Each `InstrumentationScopeSpan` will be associated with one resource, and the same goes for the `Spans`
that are associated with an instrumentation scope.

For each `Span`, then, you know what instrumentation library was used to create the `Span`, and you also 
know which `Resource` produced the `Span`. 

# What are Traces and Spans?
A trace is really just an ID attached to each individual telemetry signal that your application is producing that ties a request
together. Think of it like the execution of a workflow, where the trace ID of the workflow is added to each step.

Spans represent the individual steps within the workflow. They contain numerous properties, such as when they started and ended,
whether or not they completed successfully, and whether they belong to a parent. The full structure of the span can again be found
in the official 
[opentelemetry-proto](https://github.com/open-telemetry/opentelemetry-proto/blob/main/opentelemetry/proto/trace/v1/trace.proto) 
repository.

In total, including the instrumentation scope properties and the resource properties, there are 25 properties associated with every
span. You can find OddDotNet's span definition [here](https://github.com/OddDotNet/OddDotProto/blob/main/proto/spans/v1/span.proto)
and in the [WhereSpanPropertyFilter](https://github.com/OddDotNet/OddDotProto/blob/main/proto/spans/v1/span_query_request.proto).
Here's a code snippet for reference:

```proto
message WhereSpanPropertyFilter {
  oneof value {
    StringProperty name = 1;
    ByteStringProperty spanId = 2;
    ByteStringProperty traceId = 3;
    ByteStringProperty parentSpanId = 4;
    UInt64Property startTimeUnixNano = 5;
    UInt64Property endTimeUnixNano = 6;
    SpanStatusCodeProperty statusCode = 7;
    SpanKindProperty kind = 8;
    KeyValueProperty attribute = 9;
    UInt64Property eventTimeUnixNano = 10;
    StringProperty eventName = 11;
    ByteStringProperty linkTraceId = 12;
    ByteStringProperty linkSpanId = 13;
    StringProperty linkTraceState = 14;
    UInt32Property linkFlags = 15;
    UInt32Property flags = 16;
    StringProperty traceState = 17;
    KeyValueProperty eventAttribute = 18;
    KeyValueProperty linkAttribute = 19;
    StringProperty instrumentationScopeName = 20;
    KeyValueProperty instrumentationScopeAttribute = 21;
    StringProperty instrumentationScopeVersion = 22;
    StringProperty instrumentationScopeSchemaUrl = 23;
    StringProperty ResourceSchemaUrl = 24;
    KeyValueProperty resourceAttribute = 25;
  }
}
```