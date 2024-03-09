---
title: Building Real-time Public Transport Tracking System on Azure - Part 5 - Data Broadcast
author: valdis
date: 2020-11-01 13:30:00 +0200
categories: [.NET, C#, gRPC, Azure]
tags: [.net, c#, grpc, azure]
---

This is next post in series about real-time public transport tracking system development based on Azure infrastructure and services. This article is a part of Applied Cloud Stories initiative - [aka.ms/applied-cloud-stories](aka.ms/applied-cloud-stories).

Blog posts in this series:

* [Part 1 - Scene Background & Solution Architecture](https://tech-fellow.eu/2019/12/08/building-real-time-public-transport-tracking-system-on-azure-part1/)
* [Part 2 - Data Collectors & Composer](https://tech-fellow.eu/2020/01/21/building-real-time-public-transport-tracking-system-on-azure-part-2-data-collectors-composer/)
* [Part 3 - Event-Based Data Delivery](https://tech-fellow.eu/2020/04/08/building-real-time-public-transport-tracking-system-on-azure-part-3/)
* [Part 4 - Data Processing Pipelines](https://tech-fellow.eu/2020/08/30/building-real-time-public-transport-tracking-system-on-azure-part-4/)
* **Part 5 - Data Broadcast using gRPC**
* [Part 6 - Building Frontend](https://tech-fellow.eu/2020/12/14/building-real-time-public-transport-tracking-system-on-azure-part-6-building-frontend/)
* [Part 7 - Packing Everything Up](https://tech-fellow.eu/2021/01/21/building-real-time-public-transport-tracking-system-on-azure-part-6-packing-everything-up/)
* [Part 7.1 - Fixing Docker Image Metrics in Azure](https://tech-fellow.eu/2021/01/25/building-linux-docker-images-on-windows-machine/)

## Distribute Data to Connected Clients

We finished previous post with data processing pipelines and projecting data into profiles in order to reduce unnecessary data processing for every connected client to the application.

![brtts-p4-4](/assets/img/2020/11/brtts-p4-4.png)

Now when we have data profiles in place, let's talk now on how do get those clients connected. Our application is web based real-time map. In transportation business "real-time" is really important aspect. It affects passengers and their potential decisions made on the data operator provides. Whether it's delay information for the bus or real-time location updates, all data that helps to make a decision for the passenger or just to stay informed - is valuable.

In order to distribute data to connected clients and provide data stream in near real-time, we have chosen [gRPC](https://grpc.io/) technology to accomplish this.

## Getting Started with gRPC

I'm not gonna tell you what is gRPC and why it was developed, there are plenty of articles out there that will explain concept in much better way than I could possibly do.

So let's skip intro and get started with gRPC.
Before we jump onto actual data contract definition by using `.proto` files, we need to install supporting tools to do so.

```
PM> dotnet add package Grpc.AspNetCore
```

This package adds bunch of other packages including important package for the build time - `Grpc.Tools`.

### Creating the File

First thing we need to do is to define the shape of our services. In gRPC world this is done by defining Protobuf file.
Head to **Project > Add New Item...** and search for "proto".

![brtts-p5-proto](/assets/img/2020/11/brtts-p5-proto.png)

It creates new `.proto` file in the solution.
Note that if you inspect file properties, `BuildAction` it's set to `"None"`.

![brtts-p5-proto-actions](/assets/img/2020/11/brtts-p5-proto-actions.png)

We have to change it to `"Protobuf compiler"`.

![brtts-p5-proto-actions-2](/assets/img/2020/11/brtts-p5-proto-actions-2.png)

Notice that when we changed to `"Protobuf compiler"` there are few additional properties shown. Specifically interesting is "gRPC Stub Classes"

![brtts-p5-proto-actions-3](/assets/img/2020/11/brtts-p5-proto-actions-3.png)

Stub classes:
* `"Client and Server"` - this is telling build system - please generate both server side and client side classes.
* `"Client only"` - generate only client side classes.
* `"Server only"` - I'm implementing only server here.
* `"Do not generate"` - I don't care about neither server nor client.

Which one to use pick? It depends on where you `.proto` file is located.

### Linking the Proto File

We tend to follow file organization where `.proto` file itself physically is located in `.Abstractions` / `.Models` or any other isolated project which can be shared later on.

When creating `.proto` file in separate project you need to install `Grpc.Tools` package. Then we usually set following properties for the `.proto` file:

![brtts-p5-proto-actions-4](/assets/img/2020/11/brtts-p5-proto-actions-4.png)

Now when you have chosen location of your `.proto` file, you need to link that to gRPC server project.
One option is to use ["gRPC .NET Global Tool"](https://docs.microsoft.com/en-us/aspnet/core/grpc/dotnet-grpc?view=aspnetcore-3.1).

```
> dotnet-grpc add-file ..\Grpc.Models\protofile.proto
```

Another (maybe even easier) is to link file manually.
Head to the gRPC server-side project and add following line in your `.csproj` file:

```xml
<ItemGroup>
    <Protobuf Include="..\Grpc.Models\protofile.proto" GrpcServices="Server" Link="protos\v1\protofile.proto" />
</ItemGroup>
```

Now we have made following structure in our solution:

![brtts-p5-proto-link](/assets/img/2020/11/brtts-p5-proto-link.png)

**Note!** that it's good to start versioning your gRPC services from the very beginning of their lifespan. More info about versioning of the gRPC services could be found [here](https://docs.microsoft.com/en-us/aspnet/core/grpc/versioning?view=aspnetcore-3.1).

### Defining gRPC Service Shape

Now when we have done with file organization, we can finally jump to defining shape of our service.

Let's start with prerequisite header:

```proto
syntax = "proto3";
package vehicle_hub.v1;
```

This is the header for the `.proto` file instructing that we are running on v3 of the protocol and also dictating name of the package.

Next thing we need to do - define actual service which will feed the data.

```proto
syntax = "proto3";
package vehicle_hub.v1;

service DataFeed {
    rpc SubscribeToVehicles (Vehicles.GetAllRequest) returns (stream Vehicles.GetAllResponse);
}

message Vehicles {
    message GetAllRequest {
        RequestOptions options = 1;
    }

    message GetAllResponse {
        repeated Vehicle data = 1;
        int64 timestamp = 2;
    }

    message RequestOptions {
        repeated TransportMode.Values transport_modes = 1;
    }

    message Vehicle {
        string id = 1;
        double latitude = 2;
        double longitude = 3;
    }
}

message TransportMode {
    enum Values {
        None = 0;
        Bus = 1;
        Ferry = 2;
    }
}
```

What is cool about this service - is return type of the method (`stream`):

```proto
rpc SubscribeToVehicles (Vehicles.GetAllRequest) returns (stream Vehicles.GetAllResponse);
```

We will get back this when implementing server-side service.

Also you can notice that we are using "nested" classes for organizing messages and their types. This is a nice way to group related types together.

## Implementing gRPC Server-Side (Service)

### Creating gRPC Service Project

The easiest way to create new project is by using Visual Studio templates for this job.

Look for "grpc" in new project wizard window.

![brtts-p5-new-project](/assets/img/2020/11/brtts-p5-new-project.png)

Template is adding all necessary plumbing to get gRPC service up and running and hooked into ASP.NET Core pipeline.

### Service Implementation Placeholder

Now when shape of the service is defined with the help of the `.proto` file, it's time to implement the service.

First - we need to create new class and inherit from base type been generated for the gRPC server side:

```csharp
using System.Threading.Tasks;
using Grpc.Core;
using Grpc.Models;

namespace Grpc.Server.Services
{
    public class DataFeedService : DataFeed.DataFeedBase
    {
        public override Task SubscribeToVehicles(
            Vehicles.Types.GetAllRequest request,
            IServerStreamWriter<Vehicles.Types.GetAllResponse> responseStream,
            ServerCallContext context)
        {
            // do magic here

            // 1. register incoming request in profile storage
            var options = request.Options;
            _profileStore.TryRegister(options);

            2. TODO: subscribe to stream from EventGrid
        }
    }
}
```

The most interesting parameter to this method is `responseStream`.
More or less it has only one method - `WriteAsync`.

![brtts-p5-response-stream](/assets/img/2020/11/brtts-p5-response-stream.png)

We registered incoming request to the data profile store (which was introduced in [part 4](https://tech-fellow.ghost.io/2020/08/30/building-real-time-public-transport-tracking-system-on-azure-part-4/)) and then we need to start writing to the response stream.
Our goal basically is to keep it "alive" and write to it as data comes from the Azure EventGrid prepared through data processing pipeline and profiles straight to this response stream.

gRPC services by default are transient - meaning that each connected client will get its own gRPC service instance. This is nice from the service isolation perspective.

Noticed also that we have been provided by server call `context` parameter? This is important parameter also - as it supplies us with `CancellationToken` property. This token gives information to the server about client being disconnected, network issues, etc. - meaning that the server can abort and finish-up the stream because it's not needed anymore. On the other hand - `CancellationToken` also provides a way for the server to tell connected client that stream is being cancelled and they can do all necessary disposal and maybe reconnect if that's needed.

But before we continue - we need to convert this token into cancellable task completition source.

### Create Task Completition Source

So in order to play nicely with async world and keep gRPC service "alive" while we are streaming - we are going to convert passed in `CancellationToken` into cancellable task completition source that we can `await` later.

For that - we gonna need a small auxiliary (extension) method:

```csharp
public static class CancellationTokenExtensions
{
    public static TaskCompletionSource<object> CreateCancellable(this CancellationToken cancellationToken)
    {
        var taskCompletionSource = new TaskCompletionSource<object>(TaskCreationOptions.RunContinuationsAsynchronously);
        cancellationToken.Register(state => { ((TaskCompletionSource<object>)state).TrySetResult(null); },
                                   taskCompletionSource,
                                   false);

        return taskCompletionSource;
    }
}
```

This small extension method will give us possibility to create new task completition source allowing us to await on task which will be completed when token is cancelled.

So now we are able to continue with gRPC streaming method implementation.

### Publishing Data to gRPC Service After Processing

In part 4 we implemented background dispatcher service till the phase where we had to implement data broadcast further to gRPC service after it has been processed (`TODO` comment).

```csharp
public SnapshotDispatcherBackgroundService(
    SnapshotReceiverBuffer buffer,
    MapDataProfileStore profileStore)
{
    _buffer = buffer;
    _profileStore = profileStore;
}

protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    try
    {
        while (await _buffer.WaitToReadAsync(stoppingToken))
        {
            try
            {
                // we have received data
                _buffer.MarkAllAsRead();

                // fetch data from storage
                var latestData = await FetchLatestSnapshot();

                // prepare data for profiles
                _profileStore.PrepareData(latestData);

                // we are ready now to broadcast the data
                // TODO
            }
            catch (Exception e)
            {
                // log
            }
        }
    }
    catch (Exception ex)
    {
        // log
    }
}
```

As each gRPC service is transient and new instance is created for each connected client - we need to find a way to implement multi-broadcast pub/sub here. Meaning if two clients are connected - we would need to subscribe to data twice and each gRPC service should receive its own "copy" of the processed data to broadcast further down to the connected clients.

It would be quite interesting and probably challenging to implement this ourselves, but for sake of saving time and do something interesting, let's rely on simple yet powerful library that is designed just for this reason - ["Easy.MessageHub"](https://www.nuget.org/packages/Easy.MessageHub/) ([GitHub repo](https://github.com/NimaAra/Easy.MessageHub)).

So after installation of the package we can now finalize our `ExecuteAsync` method and implement publishing step of the processed data.

But first, we need to do a registration in DI container (Startup.cs):

```csharp
public static IServiceCollection AddMessageBus(this IServiceCollection services)
{
    var bus = new MessageHub();
    services.AddSingleton<IMessageHub>(bus);

    return services;
}

public void ConfigureServices(IServiceCollection services)
{
    ...
    services.AddMessageBus();
}
```

Now we can request `IMessageHub` type from container and invoke `Publish` later on when we need it.

```csharp
private readonly SnapshotReceiverBuffer _buffer;
private readonly MapDataProfileStore _profileStore;
private readonly IMessageHub _hub;

public SnapshotDispatcherBackgroundService(
    SnapshotReceiverBuffer buffer,
    MapDataProfileStore profileStore,
    IMessageHub hub)
{
    _buffer = buffer;
    _profileStore = profileStore;
    _hub = hub;
}

protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    try
    {
        while (await _buffer.WaitToReadAsync(stoppingToken))
        {
            try
            {
                // we have received data
                _buffer.MarkAllAsRead();

                // fetch data from storage
                var latestData = await FetchLatestSnapshot();

                // prepare data for profiles
                _profileStore.PrepareData(latestData);

                // we are ready now to broadcast the data
                _hub.Publish(latestData);
            }
            catch (Exception e)
            {
                // log
            }
        }
    }
    catch (Exception ex)
    {
        // log
    }
}
```

We are able now to subscribe to publish data and finally start broadcasting to connected clients via gRPC runtime provided `IServerStreamWriter` type.

### Awaiting on Published EventGrid Data

```csharp
public class DataFeedService : DataFeed.DataFeedBase
{
    public override Task SubscribeToVehicles(
        Vehicles.Types.GetAllRequest request,
        IServerStreamWriter<Vehicles.Types.GetAllResponse> responseStream,
        ServerCallContext context)
    {
        // 1. register incoming request in profile storage
        var options = request.Options;
        _profileStore.TryRegister(options);

        2. subscribe to stream from EventGrid
        var taskCompletionSource = context.CancellationToken.CreateCancellable();

        var subscriptionToken = _hub.Subscribe<SnapshotBroadcastWorkItem.Data>(...);

        try
        {
            await taskCompletionSource.Task;
        }
        catch(TaskCanceledException e)
        {
            // when hub subscription is cancelled due to some kind of exception
            // TODO: implement error hanlding
        }
        finally
        {
            _hub.Unsubscribe(subscriptionToken);
        }
    }
}
```

Here few things are happening  worth describing.

1. After we created cancellable task completition source, we are subscribing to hub published messages (`_hub.Subscribe<T>()`). We are capturing messaging hub subscription token.
2. After subscription is made - we need to "keep service alive". This is done by just awaiting on our earlier made completition source wrapped task. This is ensuring that service scope is not ended until task is finished.

```csharp
await taskCompletionSource.Task.ConfigureAwait(false);
```

3. After service has resumed execution (cancellation token has been cancelled) we are unsubscribing from the "Easy.MessageHub":

```csharp
...
finally
{
    _hub.Unsubscribe(subscriptionToken);
}
```

Now question is - what we are going to do inside subscribe method?

```csharp
var subscriptionToken = _hub.Subscribe<SnapshotBroadcastWorkItem.Data>(...);
```

To continue - we need to define yet another extension method (a small helper that will await on `async` method and in case of exception will invoke provided callback):

```csharp
public static async Task FireAndRunOnExceptionAsync(this Task task, Action<Exception> callback)
{
    try
    {
        await task.ConfigureAwait(false);
    }
    catch (Exception e)
    {
        callback.Invoke(e);
    }
}
```

Let's finally implement gRPC streaming method completely:

```csharp
public class DataFeedService : DataFeed.DataFeedBase
{
    public override Task SubscribeToVehicles(
        Vehicles.Types.GetAllRequest request,
        IServerStreamWriter<Vehicles.Types.GetAllResponse> responseStream,
        ServerCallContext context)
    {
        // 1. register incoming request in profile storage
        var options = request.Options;
        _profileStore.TryRegister(options);

        2. subscribe to stream from EventGrid
        var taskCompletionSource = context.CancellationToken.CreateCancellable();

        var subscriptionToken = _hub.Subscribe<SnapshotBroadcastWorkItem.Data>(
            latestData => AsyncHelper.RunSync(
                async () => await OnNewVehiclesData(options, responseStream, context)
                    .FireAndRunOnExceptionAsync(e =>
                    {
                        lastException = e;
                        if (!taskCompletionSource.Task.IsCompleted)
                        {
                            taskCompletionSource.SetCanceled();
                        }
                    })));

        try
        {
            await taskCompletionSource.Task;
        }
        catch(TaskCanceledException e)
        {
            // when hub subscription is cancelled due to some kind of exception
            var message = e.Message;
            if (lastException != default)
            {
                message = lastException.Message;
            }

            throw new RpcException(new Status(StatusCode.Internal, message));
        }
        finally
        {
            _hub.Unsubscribe(subscriptionToken);
        }
    }
}
```

We also extracted actual callback on new data into separate method - `OnNewVehiclesData`. Method just moves code from `Subscribe` method to its own just for brevity.

```csharp
private async Task OnNewVehicles(
    ProcessorOptions options,
    IServerStreamWriter<Vehicles.Types.GetAllResponse> responseStream,
    ServerCallContext context)
{
    // get transport model from the profile store
    var snapshot = _profileStore.GetData(options);

    // prepare result
    var result = new Vehicles.Types.GetAllResponse
    {
        Timestamp = snapshot.GeneratedAt.ToTimestamp()
    };

    // we can't use object initializer as gRPC requires you to add items separately
    // it generates getter only
    result.Data.AddRange(snapshot.Vehicles.ToGrpcModel());

    await responseStream.WriteAsync(result);
}
```

Note that we are executing special callback that we configure in `FireAndRunOnExceptionAsync()` in case of exception during `OnNewVehiclesData` method - we are capturing exception and cancelling completition source. This will ensure that `await taskCompletionSource.Task;` is resumed.

## Wrapping Up

We implemented our system till the phase where sent data from Azure EventGrid is passed through data processing pipeline and various profiles have been created, and later data is published via messaging hub to the gRPC service for further broadcast to connected clients to the app.

Our architecture now looks like this:

![brtts-p5-arch](/assets/img/2020/11/brtts-p5-arch.png)

We implemented gRPC server side streaming which is quite cool if you think about various possibilities how to talk to connected clients to your app in real-time.
Also liked that gRPC is "protocol-first" approach - meaning that you define the shape of your service and client stubs could be generated ion many different languages - C# and .NET are not the only options.

Next part we will need to implement frontend for the gRPC service. We gonna do a web based interface (where vehicles could be draw on the map canvas) and another frontend will be c# based client for the monitoring of the health of the service.

Stay tuned!

Happy streaming!
[*eof*]
