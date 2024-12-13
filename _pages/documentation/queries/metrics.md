---
layout: single
toc: true
toc_sticky: true
title: "Metric Queries"
permalink: /documentation/queries/metrics
sidebar:
  nav: "docs"
---
The `Metric`-specific properties that can be queried are listed below.

## MetricQueryRequest
```proto
message MetricQueryRequest {
  repeated Where filters = 1;
  odddotnet.proto.common.v1.Take take = 2;
  optional odddotnet.proto.common.v1.Duration duration = 3;
}

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
  reserved 4, 6, 8;
  oneof value {
    odddotnet.proto.common.v1.StringProperty name = 1;
    odddotnet.proto.common.v1.StringProperty description = 2;
    odddotnet.proto.common.v1.StringProperty unit = 3;
    GaugeFilter gauge = 5;
    SumFilter sum = 7;
    HistogramFilter histogram = 9;
    ExponentialHistogramFilter exponential_histogram = 10;
    SummaryFilter summary = 11;
    odddotnet.proto.common.v1.KeyValueListProperty metadata = 12;
  }
}

message GaugeFilter {
  oneof value {
    NumberDataPointFilter data_point = 1;
  }
}

message SumFilter {
  oneof value {
    NumberDataPointFilter data_point = 1;
    AggregationTemporalityProperty aggregation_temporality = 2;
    odddotnet.proto.common.v1.BoolProperty is_monotonic = 3;
  }
}

message HistogramFilter {
  oneof value {
    HistogramDataPointFilter data_point = 1;
    AggregationTemporalityProperty aggregation_temporality = 2;
  }
}

message ExponentialHistogramFilter {
  oneof value {
    ExponentialHistogramDataPointFilter data_point = 1;
    AggregationTemporalityProperty aggregation_temporality = 2;
  }
}

message SummaryFilter {
  oneof value {
    SummaryDataPointFilter data_point = 1;
  }
}

message HistogramDataPointFilter {
  reserved 1;
  oneof value {
    odddotnet.proto.common.v1.KeyValueListProperty attributes = 9;
    odddotnet.proto.common.v1.UInt64Property start_time_unix_nano = 2;
    odddotnet.proto.common.v1.UInt64Property time_unix_nano = 3;
    odddotnet.proto.common.v1.UInt64Property count = 4;
    odddotnet.proto.common.v1.DoubleProperty sum = 5;
    odddotnet.proto.common.v1.UInt64Property bucket_count = 6;
    odddotnet.proto.common.v1.DoubleProperty explicit_bound = 7;
    ExemplarFilter exemplar = 8;
    odddotnet.proto.common.v1.UInt32Property flags = 10;
    odddotnet.proto.common.v1.DoubleProperty min = 11;
    odddotnet.proto.common.v1.DoubleProperty max = 12;
  }
}

message ExponentialHistogramDataPointFilter {
  oneof value {
    odddotnet.proto.common.v1.KeyValueListProperty attributes = 1;
    odddotnet.proto.common.v1.UInt64Property start_time_unix_nano = 2;
    odddotnet.proto.common.v1.UInt64Property time_unix_nano = 3;
    odddotnet.proto.common.v1.UInt64Property count = 4;
    odddotnet.proto.common.v1.DoubleProperty sum = 5;
    odddotnet.proto.common.v1.Int32Property scale = 6;
    odddotnet.proto.common.v1.UInt64Property zero_count = 7;
    BucketFilter positive = 8;
    BucketFilter negative = 9;
    odddotnet.proto.common.v1.UInt32Property flags = 10;
    ExemplarFilter exemplar = 11;
    odddotnet.proto.common.v1.DoubleProperty min = 12;
    odddotnet.proto.common.v1.DoubleProperty max = 13;
    odddotnet.proto.common.v1.DoubleProperty zero_threshold = 14;
  }
}

message BucketFilter {
  oneof value {
    odddotnet.proto.common.v1.Int32Property offset = 1;
    odddotnet.proto.common.v1.UInt64Property bucket_count = 2;
  }
}

message NumberDataPointFilter {
  reserved 1;
  oneof value {
    odddotnet.proto.common.v1.KeyValueListProperty attributes = 7;
    odddotnet.proto.common.v1.UInt64Property start_time_unix_nano = 2;
    odddotnet.proto.common.v1.UInt64Property time_unix_nano = 3;
    odddotnet.proto.common.v1.DoubleProperty value_as_double = 4;
    odddotnet.proto.common.v1.Int64Property value_as_int = 6;
    ExemplarFilter exemplar = 5;
    odddotnet.proto.common.v1.UInt32Property flags = 8;
  }
}

message SummaryDataPointFilter {
  reserved 1;
  oneof value {
    odddotnet.proto.common.v1.KeyValueListProperty attributes = 7;
    odddotnet.proto.common.v1.UInt64Property start_time_unix_nano = 2;
    odddotnet.proto.common.v1.UInt64Property time_unix_nano = 3;
    odddotnet.proto.common.v1.UInt64Property count = 4;
    odddotnet.proto.common.v1.DoubleProperty sum = 5;
    ValueAtQuantileFilter quantile_value = 6;
    odddotnet.proto.common.v1.UInt32Property flags = 8;
  }
}

message ExemplarFilter {
  oneof value {
    odddotnet.proto.common.v1.KeyValueListProperty filtered_attributes = 1;
    odddotnet.proto.common.v1.UInt64Property time_unix_nano = 2;
    odddotnet.proto.common.v1.DoubleProperty value_as_double = 3;
    odddotnet.proto.common.v1.Int64Property value_as_int = 4;
    odddotnet.proto.common.v1.ByteStringProperty span_id = 5;
    odddotnet.proto.common.v1.ByteStringProperty trace_id = 6;
  }
}

message OrFilter {
  repeated Where filters = 1;
}

message AggregationTemporalityProperty {
  odddotnet.proto.common.v1.EnumCompareAsType compare_as = 1;
  opentelemetry.proto.metrics.v1.AggregationTemporality compare = 2;
}

message ValueAtQuantileFilter {
  oneof property {
    odddotnet.proto.common.v1.DoubleProperty quantile = 1;
    odddotnet.proto.common.v1.DoubleProperty value = 2;
  }
    
}
```

## An Example
The following code, written in C#, shows how to manually build a request and send it
to the test harness. Naturally, the syntax in other languages will be slightly different.
Additional examples in other languages are on the roadmap and will be provided soon.

This example also assumes you have the OddDotNet test harness spun up and ready to accept
telemetry data. For ideas around how to do this, see the [Quick Starts]({{ "/quick-starts/" | relative_url }}).

```csharp
// ARRANGE
// Look for metrics that have the StartTimeUnixNano of the Gauge.NumberDataPoint equal to 123
var startTimeUnixNanoFilter = new PropertyFilter
{
  Gauge = new GaugeFilter
  {
    DataPoint = new NumberDataPointFilter
    {
        StartTimeUnixNano = new UInt64Property
        {
            CompareAs = NumberCompareAsType.Equals,
            Compare = 123
        }
    }
  }
};

// Create the request
var request = new MetricQueryRequest
{
  Take = new Take
  {
    TakeFirst = new TakeFirst() // Take the first metric found
  },
  Duration = new Duration
  {
    Milliseconds = 1000 // Wait up to 1 second to find the metric
  },
  Filters = { startTimeUnixNanoFilter } // Add our filter
};

// ACT
await TriggerWorkflowThatGeneratesMetrics();

// ASSERT
var channel = GrpcChannel.ForAddress("http://localhost:4317");
var client = new MetricQueryService.MetricQueryServiceClient(channel);

MetricQueryResponse response = await client.QueryAsync(request);

// Make some assertions on the metrics returned
Assert.NotEmpty(response.Metrics);
```

## MetricQueryRequestBuilder
The above code snippet is rather verbose. Because the `MetricQueryRequest` is 
defined as a message in protobuf, we're limited in the convenience methods
available. The `MetricQueryRequestBuilder` can simplify the process of building
queries.

### CSharp
The [OddDotNet.Client](https://www.nuget.org/packages/OddDotNet.Client/) NuGet
package can be added to your project to leverage the pre-built gRPC client and
the `MetricQueryRequestBuilder`. 

The builder can streamline the generation of requests to the `MetricQueryService`.
the above snippet of code in `An Example` can be simplified using the builder.

```csharp
// ARRANGE
var request = new MetricQueryRequestBuilder()
  .TakeFirst() // Take the first metric found
  .Wait(TimeSpan.FromSeconds(1)) // Wait up to 1 second to find the metric
  .Where(filters => 
  {
    filters.Gauge.DataPoint.AddStartTimeUnixNanoFilter(123, NumberCompareAsType.Equals);
  })
  .Build();

// ACT
await TriggerWorkflowThatGeneratesMetrics();

// ASSERT
var channel = GrpcChannel.ForAddress("http://localhost:4317");
var client = new MetricQueryService.MetricQueryServiceClient(channel);

MetricQueryResponse response = await client.QueryAsync(request);

// Make some assertions on the metrics returned
Assert.NotEmpty(response.Metrics);
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