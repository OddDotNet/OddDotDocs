---
layout: single
toc: true
toc_sticky: true
title: OddDotNet / OpenTelemetry Collector Testing
sidebar:
  nav: "samples"
permalink: /quick-starts/csharp/otelcol/
---
## Repository
[Sample-OtelCollector](https://github.com/OddDotNet/Sample-OtelCollector)

## Summary
Have you ever broken your OpenTelemetry Collector because something was misconfigured? 

Have you ever lost telemetry signals that were critical for troubleshooting your 
application because a regression was introduced to the collector?

Configuring the collector can be a very complicated endeavor. For trivial use cases, 
the out of the box configuration is straightforward, but the moment you begin to add 
additional receivers, processors, and pipelines, great care must be taken to ensure 
your collector is performing and behaving as expected and that regressions haven't been introduced.

How do you guarantee that you have configured your collector correctly? 
How do you catch regressions before they make it out to production?

Traditionally, collector configuration and testing has been a manual process: 

1. Edit the configuration file. 
2. Restart the collector.
3. Send in some signals using either a real application or some script you've written.
4. Check the debug logs of the collector, or look at your observability vendor to verify things are working.

Manual testing like this is useful but slow, and it is very error-prone.

This example shows how OddDotNet can help automate the testing and validation of your
collector configuration.

## Description
A basic understanding of OpenTelemetry, gRPC, Docker, and .NET Aspire is
recommended before diving into this sample.

- [OpenTelemetry](https://opentelemetry.io/)
- [gRPC](https://grpc.io/)
- [Docker](https://www.docker.com/)
- [.NET Aspire](https://learn.microsoft.com/en-us/dotnet/aspire/get-started/aspire-overview)

## Prequisites
- .NET 8
- .NET Aspire workload installed

## Rundown
This sample project outlines how to use OddDotNet to test an OTel Collector
that is configured to use [tail_sampling](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/processor/tailsamplingprocessor/README.md).

***NOTE:*** The tail_sampling processor provides the ability to make sampling decisions
based on the entire contents of a trace (all spans). Configuration of the sampler
is achieved by adding "policies", which can quickly become complicated.
{: .notice--info}

The project enables 3 policies for the tail_sampling processor:
1. Teams that have not yet "opted in" will have all their traces sampled.
2. Traces that are part of a "readiness" probe should be sampled at a reduced rate.
3. Traces with errors should always be sampled.

Each policy is tested by sending traces through the collector that match the criteria
we are looking for. For example, the first test sends two traces to the collector:
one trace where the Resource's `service.name` attribute is "service-1", and another for
"service-4" (where "service-4" belongs to a team that has not yet "opted in").

The traces are sent to the collector using the `TraceService.TraceServiceClient` client,
which is defined in the official [opentelemetry-proto](https://github.com/open-telemetry/opentelemetry-proto)
repository. The C# client library [OddDotNet.Client](https://www.nuget.org/packages/OddDotNet.Client)
comes with this client pre-built, but you can build your own in your language of choice
using the [Protocol Buffer Compiler](https://grpc.io/docs/protoc-installation/).

In addition, traces are queried against OddDotNet using the `SpanQueryService.SpanQueryServiceClient`
client, also part of the C# library. This client can also be generated in your language
of choice using `protoc`. The proto files are located in the official 
[OddDotProto](https://github.com/OddDotNet/OddDotProto) repository.

## YouTube
Watch the video [here](https://youtu.be/NQCMFN6njQM).