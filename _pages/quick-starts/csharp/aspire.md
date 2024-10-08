---
layout: single
# classes: wide
toc: true
title: OddDotNet / .NET Aspire
sidebar:
  nav: "samples"
permalink: /quick-starts/csharp/aspire/
---
## Repository
[Sample-AspireTodoApp](https://github.com/OddDotNet/Sample-AspireTodoApp)

## Summary
.NET Aspire is "an opinionated, cloud ready stack for building observable, 
production ready, distributed applications." This sample project outlines
how to use .NET Aspire's features to configure the OddDotNet OpenTelemetry
test harness and use it in your automated tests.

`TodoWebApp` is a very basic WebAPI project that enables you to create
TODO items, and then retrieve those TODO items using the ID. Requests 
to retrieve the same TODO item are cached for 30 seconds.

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

### Project Structure
* Some files removed for brevity
```
.
├── Sample-AspireTodoApp.sln
├── src
│   ├── TodoAppHost
│   │   ├── Program.cs
│   │   ├── TodoAppHost.csproj
│   │   └── appsettings.json
│   ├── TodoServiceDefaults
│   │   ├── Extensions.cs
│   │   └── TodoServiceDefaults.csproj
│   └── TodoWebApp
│       ├── Controllers
│       │   └── TodosController.cs
│       ├── Database.cs
│       ├── Models
│       │   ├── CreateTodoItemRequest.cs
│       │   └── TodoItemModel.cs
│       ├── Program.cs
│       ├── TodoWebApp.csproj
│       └── appsettings.json
└── tests
    ├── TodoTestAppHost
    │   ├── Program.cs
    │   ├── TodoTestAppHost.csproj
    │   ├── appsettings.json
    └── TodoTests
        ├── GetTodosShould.cs
        └── TodoTests.csproj
```

#### TodoAppHost
The `TodoAppHost` project is a very simple .NET Aspire AppHost project for running
the `TodoWebApp` manually, taking advantage of the dashboards and automatic
instrumentation that .NET Aspire provides out of the box. 

There's not much to look at here, but if you want to run the app on your own, running
the `TodoAppHost` and then navigating to the dashboard and the `TodoWebApp` Swagger
page should give you everything you need.

#### TodoServiceDefaults
Again, there's not much here. .NET Aspire recommends a "Service Defaults" project
that all your services depend on. This project centralizes the configuration of 
things like OpenTelemetry and Service Discovery.

#### TodoWebApp
A simple WebAPI for creating new TODO items, and retrieving a TODO item by ID.

This project uses `EntityFrameworkCore` with an in-memory SQLite database. It also
includes `IMemoryCache` to simulate cache hits vs. database hits.

While you can run this project on its own, it's recommended that you use the `TodoAppHost`
to spin everything up instead.

#### TodoTestAppHost
This project is used by the `TodoTests` project to spin up the `TodoWebApp` and the
OddDotNet container. It also configures `TodoWebApp` to send its telemetry to the 
test harness rather than the .NET Aspire collector. It's not recommended that you
run this project on its own.

#### TodoTests
This `.NET Aspire xUnit` test project contains a single test that spins up the System
Under Test (SUT, in this case `TodoWebApp`) and the `OddDotNet` test harness, and then
uses the `SpanQueryService` to make assertions on telemetry data.

### Rundown
Take a look at the test in `GetTodosShould.cs`. The name of the test is `UseCacheOnSecondRequest`.

In our simulated scenario, we want to enable caching functionality for TODO items so
that the database doesn't get hit as often. In `TodoWebApp/Controllers/TodosController.cs`, we provide
two endpoints for the consumer of our app: `POST CreateTodoItem`, and `GET GetTodoItem`. When getting
a TODO item, the in-memory cache is first checked and then, if the item doesn't exist in cache, the
database is queried, with the results cached for the next attempt. 

How would you normally test this functionality? Typically, you would mock/stub out the 
`IMemoryCache` and the repository that is wrapping your `DbContext`. Mocking is generally
required here because you need to be able to `.Verify()` the call was made (or not made).

In the `UseCacheOnSecondRequest()` test, however, we're not mocking anything. In fact,
this is an integration test. We are using the `TodoWebApp` application exactly as it
is written. The only thing we're doing different in our application is supplying it
with a different value for the `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT` environment
variable, which directs the OpenTelemetry library to send traces to our test harness
rather than the regular OTel collector from .NET Aspire. 

The steps for our test, then, are as follows:
1. Build and start the .NET Aspire `TodoTestAppHost` and ensure our SUT and the OddDotNet
test harness are up and healthy.
2. Create a new TODO Item using the POST API.
3. Generate two TraceIds and two SpanIds
    - Set the "traceparent" header of the http request client to the first traceId/spanId and send the request
    - Re-set the "traceparent" header of the http request client to the second traceId/spanId and send the request
4. Build the `SpanQueryRequest`
    - `TakeAll()` spans that you find
    - `Wait(TimeSpan.FromSeconds(1))` for up to 1 second after you begin the query
    - `Where(...)`
        - Add an `OrFilter`, and provide it with two values
            - Find any spans where the traceId matches the first traceId, OR
            - Find any spans where the traceId matched the second traceId
5. Send the request
6. Assert that `EntityFrameworkCore` was called in the first request, and that it was NOT called in the second request.

Because the application has been configured to generate telemetry data when Entity Framework Core
requests are made (`AddEntityFrameworkCoreInstrumentation()` in the Service Defaults project), we
know that database queries will result in corresponding telemetry, specifically telemetry
generated by the `OpenTelemetry.Instrumentation.EntityFrameworkCore` instrumentation scope.

The OddDotNet span model includes the Instrumentation Scope values that were used to 
create the span, in this case the `EntityFrameworkCore` instrumentation library. We can
check the `span.InstrumentationScope.Name` property to see what the name of the instrumentation scope
was that generated the span. 

In the `orFilters` you'll also notice that there is an extension method called `AddTraceIdFilter`
that takes the traceId byte array as an argument, along with an enum that specifies what type
of comparison you'd like to perform (in this case an EQUALS comparison).

## YouTube
TODO: Insert youtube link here...

## Credit
Credit where credit is due. This sample project, and really the entire 
OddDotNet OpenTelemetry test harness, draws inspiration from a presentation
by [Martin Thwaites](https://github.com/martinjt) at the NDC London 2023
conference, and his corresponding `todo-odd` code sample found
[here](https://github.com/martinjt/todo-odd/tree/main).

The presentation can be viewed [here](https://youtu.be/prLRI3VEVq4).