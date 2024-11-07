---
layout: single
classes: wide
title: "Metrics"
permalink: /documentation/concepts/metrics/
sidebar:
  nav: "docs"
---
Before you dive in to the metrics concepts outlined below, review
these concepts in the official OpenTelemetry documentation, found
[here](https://opentelemetry.io/docs/concepts/signals/metrics/).

# The ExportMetricServiceRequest
All OpenTelemetry collectors receive OTLP metrics from applications in the form
of an `ExportMetricServiceRequest`. This is a well known message, and its
structure can be found in the official 
[opentelemetry-proto](https://github.com/open-telemetry/opentelemetry-proto/blob/main/opentelemetry/proto/collector/metrics/v1/metrics_service.proto) repository. The diagram below provides the general structure of 
this message (some properties omitted for brevity).

![ExportMetricServiceRequest diagram]({{ "/assets/diagrams/ExportMetricServiceRequest/ExportMetricServiceRequest.svg" | relative_url }})

When the OpenTelemetry Collector exports metrics to an endpoint, it typically batches those metrics up and
sends them by resource (eg. a microservice), hence the list of resources in an `ExportMetricServiceRequest`.
Each `ScopeMetrics` will be associated with one resource, and the same goes for the `Metrics`
that are associated with an instrumentation scope.

For each `Metric`, then, you know what instrumentation library was used to create the `Metric`, and you also 
know which `Resource` produced the `Metric`. 

# What Are Metrics?
The OpenTelemetry Metric consists of the name of the meter, an optional `description` and `unit`, and the `kind`
of meter. The kinds of meters are:
* Sum
* Gauge
* Histogram
* ExponentialHistogram
* Summary

The available properties of a `Metric`that can be queried are listed below:

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

message PropertyFilter {
    reserved 4, 6, 8;
    oneof value {
        odddotnet.proto.common.v1.StringProperty name = 1;
        odddotnet.proto.common.v1.StringProperty description = 2;
        odddotnet.proto.common.v1.StringProperty unit = 3;
        GaugeFilter gauge = 5;
        SumFilter sum = 7;
        HistogramFilter histogram = 9;
        ExponentialHistogramFilter exponentialHistogram = 10;
        SummaryFilter summary = 11;
        odddotnet.proto.common.v1.KeyValueProperty metadata = 12;
    }
}
```

The `PropertyFilter` includes a number of additional filters for the different meter types, such as
the `GaugeFilter` for querying properties associated with the `Gauge`. See 
[Metric Queries]({{ "/documentation/queries/metrics/" | relative_url }}) for more information around
querying metrics.

The proto files for Metrics can be found [here](https://github.com/OddDotNet/OddDotProto/tree/tkenna/feature/metrics/odddotproto/proto/metrics/v1).