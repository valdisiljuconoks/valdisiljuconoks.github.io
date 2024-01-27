---
title: Building Real-time Public Transport Tracking System on Azure - Part 4
author: valdis
date: 2020-08-30 10:00:00 +0200
categories: [.NET, C#, gRPC, Azure]
tags: [.net, c#, grpc, azure]
---

This is next post in series about real-time public transport tracking system development based on Azure infrastructure and services. This article is a part of Applied Cloud Stories initiative - [aka.ms/applied-cloud-stories](aka.ms/applied-cloud-stories).

Blog posts in this series:

* [Part 1 - Scene Background & Solution Architecture](https://tech-fellow.eu/2019/12/08/building-real-time-public-transport-tracking-system-on-azure-part1/)
* [Part 2 - Data Collectors & Composer](https://tech-fellow.eu/2020/01/21/building-real-time-public-transport-tracking-system-on-azure-part-2-data-collectors-composer/)
* [Part 3 - Event-Based Data Delivery](https://tech-fellow.eu/2020/04/08/building-real-time-public-transport-tracking-system-on-azure-part-3/)
* **Part 4 - Data Processing Pipelines**
* [Part 5 - Data Broadcast using gRPC and SignalR](https://tech-fellow.eu/2020/11/01/building-real-time-public-transport-tracking-system-on-azure-part-5/)
* [Part 6 - Building Frontend](https://tech-fellow.eu/2020/12/14/building-real-time-public-transport-tracking-system-on-azure-part-6-building-frontend/)
* [Part 7 - Packing Everything Up](https://tech-fellow.eu/2021/01/21/building-real-time-public-transport-tracking-system-on-azure-part-6-packing-everything-up/)
* [Part 7.1 - Fixing Docker Image Metrics in Azure](https://tech-fellow.eu/2021/01/25/building-linux-docker-images-on-windows-machine/)

## Broadcast Phase

We ended [last part of this blog post series](https://tech-fellow.ghost.io/2020/04/08/building-real-time-public-transport-tracking-system-on-azure-part-3/) with a open question what to do next once we have received EventGrid event.

We are now in the phase when data needs to be prepared and broadcasted to connected clients (either mobile apps, browsers, specific data feeds or any other consumer application).

Following architecture is set for  broadcast phase of our real-time public transport tracking system:

![brtts-p4-1](/assets/img/2020/08/brtts-p4-1.png)

## Receiving EventGrid Events

Now we need to distribute if further to connected clients (whoever it might be).

Code fragment that we now need to implement is between these lines:

```csharp
if (eventGridEvent.Data is StorageBlobCreatedEventData)
{
    // TODO: magic happens here
}
```

If we inspect received EventGrid data - we see that there is only a reference to blob that has been generated and saved in Azure Storage **and not** the actual content.

```json
[
  {
    "topic": "/subscriptions/{subscription-id}/resourceGroups/Storage/providers/Microsoft.Storage/storageAccounts/xstoretestaccount",
    "subject": "/blobServices/default/containers/oc2d2817345i200097container/blobs/oc2d2817345i20002296blob",
    "eventType": "Microsoft.Storage.BlobCreated",
    "eventTime": "2017-06-26T18:41:00.9584103Z",
    "id": "831e1650-001e-001b-66ab-eeb76e069631",
    "data": { ... },
    "dataVersion": "",
    "metadataVersion": "1"
  }
]
```

This is by design how EventGrid is working - it's event messaging infrastructure and not data transport. It suppose to trigger an event processing pipeline on the receiver side.

So to get content - we need to fetch it via Storage SDK. It's quite simple and straight forward.

```csharp
var storageAccount = CloudStorageAccount.Parse(connectionString);
var client = storageAccount.CreateCloudBlobClient();
var container = client.GetContainerReference(containerName);
var blob = container.GetBlockBlobReference(blobName);

using var memoryStream = new MemoryStream(initialCapacity);
await blob.DownloadToStreamAsync(memoryStream);
var content = memoryStream.ToArray();
```

To reply back to EventGrid runtime that we have received ("acknowledged") the message - we have to act quickly. This usually means that we can't really download the data from the storage, start data processing pipeline, data slicing and preparing for the broadcast and actually do the broadcast - as it all takes time and "blocks" request receiver to send back 200OK message to Azure EventGrid infrastructure.

We have to somehow "buffer" the incoming messages and reply quickly as possible back to EventGrid.

For this purpose we can utilize interesting data structure - `Channel<T>` (super cool intro [here](https://devblogs.microsoft.com/dotnet/an-introduction-to-system-threading-channels/)).

This allows us to split apart receiver side (writer on the channel) and processor side (reader on the channel).

Definition of the channel is pretty simple:

```csharp
private readonly Channel<SnapshotBroadcastWorkItem.Trigger> _buffer =
    Channel.CreateUnbounded<SnapshotBroadcastWorkItem.Trigger>(new UnboundedChannelOptions
    {
        AllowSynchronousContinuations = false,
        SingleReader = true,
        SingleWriter = false
    });
```

This definition will create data structure that will be used to create buffer between producer of the data (trigger items on the channel) and subscriber of the data. Each of these parties can act on their own frequencies - meaning that you can write to the channel without knowing whether there is anybody on the other side or not (and you actually should not care as long as channel is not empty or you run out of memory).

What I usually like to do - is to hide some implementation details behind abstractions to avoid leaky dependencies outside - no one really cares that underlying transport is `Channel<T>`.

Let's define abstraction around the `Channel<T>` data structure:

```csharp
public class SnapshotReceiverBuffer
{
    private readonly Channel<SnapshotBroadcastWorkItem.Trigger> _buffer =
        Channel.CreateUnbounded<SnapshotBroadcastWorkItem.Trigger>(new UnboundedChannelOptions
        {
            AllowSynchronousContinuations = false,
            SingleReader = true,
            SingleWriter = false
        });

    public bool TryAdd() => _buffer.Writer.TryWrite(new SnapshotBroadcastWorkItem.Trigger());

    public ValueTask<bool> WaitToReadAsync(CancellationToken stoppingToken) =>
        _buffer.Reader.WaitToReadAsync(stoppingToken);

    public bool TryRead(out SnapshotBroadcastWorkItem.Trigger message) =>
        _buffer.Reader.TryRead(out message);

    public void MarkAllAsRead()
    {
        while (_buffer.Reader.TryRead(out _)) { }
    }
}
```

So you can configure your favorite DI container to keep this as singleton and inject all the time the same instance into services.

So now with an abstraction in place we can go back to our EventGrid receiver code and implement buffer there:

```csharp
[ApiController]
[Route("api")]
public class EventGridReceiverController : ControllerBase
{
    private readonly SnapshotReceiverBuffer _buffer;

    public EventGridReceiverController(SnapshotReceiverBuffer buffer)
    {
        _buffer = buffer;
    }

    [Route("eventgridreceiver")]
    [HttpPost]
    public ActionResult ReceiveEventGridPost([FromBody] object content)
    {
        return HandleEvent(content.ToString());
    }

    public ActionResult HandleEvent(string content)
    {
        var eventGridSubscriber = new EventGridSubscriber();
        var eventGridEvents = eventGridSubscriber.DeserializeEventGridEvents(content);

        foreach (var eventGridEvent in eventGridEvents)
        {
            // EventGrid will send verification request and we need to reply with given code
            // otherwise event subscription creation will fail in Azure and no data will be sent to this endpoint
            if (eventGridEvent.Data is SubscriptionValidationEventData eventData)
            {
                return Ok(new SubscriptionValidationResponse { ValidationResponse = eventData.ValidationCode });
            }

            if (eventGridEvent.Data is StorageBlobCreatedEventData)
            {
                _buffer.TryAdd();
            }
        }

        return Ok();
    }
}
```

With this buffer in place responses are sent almost immediately after request is received in this EventGrid receiver controller.

Visually our architecture is now looking something like this:

![brtts-p4-2](/assets/img/2020/08/brtts-p4-2.png)

We are missing some of the parts that we are going to fill in soon.

## Data Processing Pipeline

Now as we have data inside channel - we can build reader part to execute some piece of code every time EventGrid receiver controller will write new data to the channel.

Reader side on the channel is long-running process that needs to sit in the background and process items only when there is something on the channel.

This really sounds a perfect use case for new .NET Core feature - [BackgroundService](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/hosted-services?view=aspnetcore-3.1&tabs=visual-studio).

Let's start with empty template for the service:

```csharp
public class SnapshotDispatcherBackgroundService : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken) { }
}
```

Second, in order for us to the this service up & running, we need to register it in the service collection (usually this is done in `Startup.cs` file):

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddHostedService<SnapshotDispatcherBackgroundService>();
}
```

Having abstractions around `Channel<T>` data structure we can provide with some nice methods for the reader part. Essentially what we need to get - is a message pump that is invoked every time there is a new item in the channel and preferably not to block the thread.

```csharp
public SnapshotDispatcherBackgroundService(SnapshotReceiverBuffer buffer)
{
    _buffer = buffer;
}

protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    try
    {
        while (await _buffer.WaitToReadAsync(stoppingToken).ConfigureAwait(false))
        {
            try
            {
                // we have received data
                _buffer.MarkAllAsRead();

                // fetch data from storage
                var latestData = await FetchLatestSnapshot().ConfigureAwait(false);

                // prepare data for profiles
                // TODO

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

Here we are using cool feature - async `while` loop. We can run `while` loop and await asynchronously till the moment then `Channel<T>` reader gives us signal that it's now time to start reading message from the channel - because somebody wrote one or more messages there. Thanks to `await` - we are not blocking here.

Now we have something more in our architecture:

![brtts-p4-3-1](/assets/img/2020/08/brtts-p4-3-1.png)

But before we get started with data broadcast, we have to cover data processing step here before broadcast - data profiles.

### Data Profiles

When data is received from EventGrid we are going to broadcast it to the connected clients (either mobile app, browser or any other system).
Before data is sent to clients - some processing has to be done. For example - we should do simple filtering - for example remove all vehicles with invalid coordinates(`Lat = 0, Lng = 0`). These vehicles otherwise would be located on "Null Island". This is step that could be shared across all connected clients and we can execute `InvalidCoordinatesFilter` on the server.

Another filtering type - `VehicleStatusFilter`. Every vehicle in the snapshot has different status - some of them are on service journey (status = on duty), some of them are parked at depots (status = parked), etc. Depending on visitor "access level" you might see only vehicles with status "on duty", or you might see all vehicles. Depending who you are - you have specific "data profile" assigned on first visit on the map page.

This is what we call - "Data Profiles". It's the shape of the data that is going to be sent to the clients.

Profiles are composed out of individual processing options. Processing options are part of the request to the broadcast service. Either filter is enabled or disabled. In order to avoid duplicate processing and waste of resources - we are going to analyze and compare combination of supplied processing options and construct new data profile only in those cases when there is no one else with the same combination. This would allow us to construct unique data profile and do filtering and processing only once.

We will get to the actual implementation of gRPC service and client proxy code generation, but for now let's assume that client application is able to pass in processing options somehow.

This is gRPC method that's being invoked from the client code:

```csharp
public override async Task Subscribe(
    SubscribeRequest request,
    IServerStreamWriter<SubscribeResponse> responseStream,
    ServerCallContext context)
{
    _processingOptions = request.DataProcessingOptions;
    _profileStore.TryRegister(_processingOptions);
}
```

Here essentially we have implemented a method that is invoked every time new client is connected to the gRPC service and subscribing to vehicle data stream. Don't worry about all the unknowns there - we will cover those in more details soon.

Client has supplied processing options when connecting to the gRPC service, and from these options we are constructing specific data profile.

This is pretty naive implementation of profile store. It does not support yet data profile teardown - or process when last client with specific profile disconnects from the server - we should also remove this profile from the profile store, as there are no reasons to "prepare" data for that profile anymore.

```csharp
public class MapDataProfileStore
{
    private readonly ConcurrentDictionary<DataProcessingOptions, ProfileData> _profiles =
        new ConcurrentDictionary<DataProcessingOptions, ProfileData>();

    public int Count => _profiles.Count;

    public bool TryRegister(DataProcessingOptions processingOptions)
    {
        if (processingOptions == null) return false;

        bool result;

        if (!_profiles.ContainsKey(processingOptions))
        {
            var profile = new ProfileData();
            result = _profiles.TryAdd(processingOptions, profile);
        }
        else
        {
            result = true;
        }

        return result;
    }
}
```

Now our system architecture has following look:

![brtts-p4-4](/assets/img/2020/08/brtts-p4-4.png)

This profile store allows us to do bookkeeping of data profiles across all connected clients.

### Shaping Data for Profiles

Now when we have defined what data profiles are - we can slice and shape received data from EventGrid to appropriate profilers.

For that - we will ask profile store to do the work. Our receiver code now is following:

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
        while (await _buffer.WaitToReadAsync(stoppingToken).ConfigureAwait(false))
        {
            try
            {
                // we have received data
                _buffer.MarkAllAsRead();

                // fetch data from storage
                var latestData = await FetchLatestSnapshot().ConfigureAwait(false);

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

Profile store actually has no idea how to process data, but this job is delegated further to data processing pipeline:

```csharp
public class MapDataProfileStore
{
    private readonly DataProcessingPipeline _processorPipeline;

    private readonly ConcurrentDictionary<MapDataOptions, ProfileData> _profiles =
        new ConcurrentDictionary<MapDataOptions, ProfileData>();

    public MapDataProfileStore(DataProcessingPipeline processorPipeline)
    {
        _processorPipeline = processorPipeline;
    }

    public void PrepareData(SnapshotBroadcastWorkItem.Data latestSnapshot)
    {
        Parallel.ForEach(
            _profiles,
            pair =>
            {
                // create working copy of the data
                var copy = latestSnapshot.Clone();
                var (options, value) = pair;

                // do the actual data processing
                value.Snapshot = _processorPipeline.Process(copy.TransportDataModel, options);
            });
    }
}
```

As data processing is more or less CPU intensive usually - we made it a bit faster by running as much parallel processing as we could by using `Parallel.ForEach()` feature originally coming form TPL (Task Parallel Library).

And data processor pipeline is more or less straight process to run through various registered processors and filters:

```csharp
public class DataProcessingPipeline
{
    private readonly IEnumerable<IVehicleDataProcessor> _processors;
    private readonly IEnumerable<IVehicleDataFilter> _filters;

    public DataProcessingPipeline(
        IEnumerable<IVehicleDataProcessor> processors,
        IEnumerable<IVehicleDataFilter> filters)
    {
        _processors = processors ?? new List<IVehicleDataProcessor>();
        _filters = filters ?? new List<IVehicleDataFilter>();
    }

    public TransportDataModel Process(TransportDataModel model, MapDataOptions options)
    {
        var result = model;

        foreach (var processor in _processors)
        {
            result = processor.Process(ref result, options);
        }

        foreach (var processor in _filters)
        {
            result = processor.Filter(result, options);
        }

        return result;
    }
}
```

List of processors and filters are configured as multiple implementations of specific interfaces:

```csharp
// vehicle data processing pipeline
services.AddSingleton<DataProcessingPipeline>();
services.AddSingleton<IVehicleDataProcessor, DelayStatusProcessor>();
services.AddSingleton<IVehicleDataFilter, InvalidCoordinatesFilter>();
services.AddSingleton<IVehicleDataFilter, UnknownStatusesFilter>();
services.AddSingleton<IVehicleDataFilter, ParkedBusFilter>();
services.AddSingleton<IVehicleDataFilter, VehicleTypesFilter>();
```

If we zoom in following are internals of the single profile data creation:

![brtts-p4-5-1](/assets/img/2020/08/brtts-p4-5-1.png)

Now when have got through data processing pipeline and prepared data for various profiles we are finally ready for data broadcast to connected clients.

This is story for another gRPC and SignalR focused post.

Happy coding!

[*eof*]
