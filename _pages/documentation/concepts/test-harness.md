---
layout: single
title: "OpenTelemetry Test Harness"
toc: true
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

# Environment Variables
The following environment variables are recognized by OddDotNet and can be used to 
configure various things such as cache expiration and cleanup intervals.

All variables begin with `ODD`. Underscores can be single `_` or double `__`, eg.
`ODD_CACHE_EXPIRATION` and `ODD__CACHE__EXPIRATION` will both work. Single underscores
take precedence.

You can pass environment variables to the container using the `-e` flag. eg. 

`docker run -p 4317:4317 -e ODD_CACHE_EXPIRATION=5000 ghcr.io/odddotnet/odddotnet:latest`

## ODD_CACHE_CLEANUP_INTERVAL (ms)
The container includes a `BackgroundService` responsible for pruning the signal lists
of any expired signals. This background service runs once per second (`default = 1000`),
but can be configured to run at a different interval if needed.

## ODD_CACHE_EXPIRATION (ms)
Determines how long a telemetry signal remains in cache before being pruned. Signals may
be received by the container before a query request has been started. As a result, it is
possible to "miss" some of the signals because they came in before the query began. The
container will cache signals, and the query will check for previously received signals
first to find a match. 

The default value is 30,000 milliseconds.

If OddDotNet is receiving a large number of signals within a 30 second period, memory
pressure may become an issue. If a full 30 seconds of signal history is not necessary,
lowering this value can help reduce the container's memory footprint.