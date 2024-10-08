---
layout: single
classes: wide
title: Getting Started
permalink: /quick-starts/
sidebar:
  nav: "samples"
---
The examples on the left provide instructions for getting started with the OddDotNet
OpenTelemetry test harness. Each example (will eventually!) include a video 
describing the project, along with code snippets and written descriptions.

## Prerequisites
### OddDotNet Container
All of the examples require the OddDotNet container, with an exposed port for gRPC
(and eventually HTTP). 

```bash
docker run -p 4317:4317 ghcr.io/odddotnet/odddotnet:latest
```

This will start the container up and expose port 4317 on your host machine. You will
omit this step if you are managing the container in code instead, such as with a test
that uses [Testcontainers](https://testcontainers.com/) to spin up dependencies.

### Protobuf
In addition to the container, you will also need a client for accessing the various
query services (currently only `SpanQueryService` exists, more to come later).

#### Pre-built Client
Currently only C# has a pre-built client. It can be pulled in as a NuGet package:

```bash
dotnet add package OddDotNet.Client --version 0.0.4
```

The source code can be found at [OddDotCSharp](https://github.com/OddDotNet/OddDotCSharp).

The pre-built C# client also includes a builder for easily constructing queries. See any of
C# quick starts for instructions.

#### From Source
If you're using a language that does not have a pre-built client, the messages and clients can be generated using 
the language-specific protobuf compiler. Instructions can be found [here](https://grpc.io/docs/languages/).
The `.proto` files themselves are located at [OddDotProto](https://github.com/OddDotNet/OddDotProto).

Most languages also have libraries to automatically build
`.proto` files and include them in your project, eg. C#/.NET has the `Grpc.Tools` and 
`Grpc.AspNetCore` NuGet packages to build your proto files automatically.

***TIP!*** OddDotNet and OddDotCSharp make extensive use of `git submodules` to automatically include
the `.proto` files in the project. Check out the source code to see how this is set up.
{: .notice--primary}

***NOTE:*** Support for more languages is on the roadmap. The "From Source" approach will work just
fine, but it is more convenient to have a pre-built client if possible. Java and Go are
the likely next candidates.
{: .notice--info}