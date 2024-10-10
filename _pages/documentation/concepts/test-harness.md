---
layout: single
title: "OpenTelemetry Test Harness"
permalink: /documentation/concepts/test-harness/
sidebar:
  nav: "docs"
---
The [OddDotNet OpenTelemetry Test Harness](https://github.com/OddDotNet/OddDotNet/pkgs/container/odddotnet) 
is a container image that acts as a OpenTelemetry Collector with an OTLP receiver. 
In addition, it includes a query language and service for querying telemetry signals
that it has received in real time. It currently only supports traces/spans, but 
metrics and logs are on the road map and should be available soon.

# Running
The images exposes port `4317`, the default port for OTLP over gRPC. To send telemetry
to the container from your local machine, expose your desired host port when running
the container.

`docker run -p 4317:4317 ghcr.io/odddotnet/odddotnet:latest`

## Health Check
The container provides a health check endpoint on port 4318 at `/healthz` and can be
used to confirm the application is up first before continuing. 