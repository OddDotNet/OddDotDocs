---
layout: single
title: "Logs"
permalink: /documentation/concepts/logs/
sidebar:
  nav: "docs"
---
Before you dive in to the log concepts outlined below, review
these concepts in the official OpenTelemetry documentation, found
[here](https://opentelemetry.io/docs/concepts/signals/logs/).

# The ExportLogServiceRequest
All OpenTelemetry collectors receive OTLP logs from applications in the form
of an `ExportLogServiceRequest`. This is a well known message, and its
structure can be found in the official 
[opentelemetry-proto](https://github.com/open-telemetry/opentelemetry-proto/blob/main/opentelemetry/proto/collector/logs/v1/logs_service.proto) repository. The diagram below provides the general structure of 
this message (some properties omitted for brevity).

![ExportLogServiceRequest diagram]({{ "/assets/diagrams/ExportLogServiceRequest/ExportLogServiceRequest.svg" | relative_url }})

When the OpenTelemetry Collector exports logs to an endpoint, it typically batches those logs up and
sends them by resource (eg. a microservice), hence the list of resources in an `ExportLogServiceRequest`.
Each `ScopeLogs` will be associated with one resource, and the same goes for the `Logs`
that are associated with an instrumentation scope.

For each `Log`, then, you know what instrumentation library was used to create the `Log`, and you also 
know which `Resource` produced the `Log`. 

# What Are Logs?
The OpenTelemetry LogRecord is a data structure that includes details of the log message, and also 
correlates the log message to a `TraceId` and `SpanId` if the log was emitted within the context
of a span.

The available properties of a `Log`that can be queried are listed below:

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

message SeverityNumberProperty {
  odddotnet.proto.common.v1.EnumCompareAsType compare_as = 1;
  opentelemetry.proto.logs.v1.SeverityNumber compare = 2;
}
```

The proto files for Logs can be found [here](https://github.com/OddDotNet/OddDotProto/tree/main/odddotproto/proto/logs/v1).