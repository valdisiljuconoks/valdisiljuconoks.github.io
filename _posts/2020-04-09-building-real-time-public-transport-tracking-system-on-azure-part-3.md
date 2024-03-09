---
title: Building Real-time Public Transport Tracking System on Azure - Part 3
author: valdis
date: 2020-04-09 10:00:00 +0200
categories: [.NET, C#, gRPC, Azure]
tags: [.net, c#, grpc, azure]
---

This is next post in series about real-time public transport tracking system development based on Azure infrastructure and services. This article is a part of Applied Cloud Stories initiative - [aka.ms/applied-cloud-stories](https://aka.ms/applied-cloud-stories).


Blog posts in this series:

* [Part 1 - Scene Background & Solution Architecture](https://tech-fellow.eu/2019/12/08/building-real-time-public-transport-tracking-system-on-azure-part1/)
* [Part 2 - Data Collectors & Composer](https://tech-fellow.eu/2020/01/21/building-real-time-public-transport-tracking-system-on-azure-part-2-data-collectors-composer/)
* **Part 3 - Event-Based Data Delivery**
* [Part 4 - Data Processing Pipelines](https://tech-fellow.eu/2020/08/30/building-real-time-public-transport-tracking-system-on-azure-part-4/)
* [Part 5 - Data Broadcast using gRPC and SignalR](https://tech-fellow.eu/2020/11/01/building-real-time-public-transport-tracking-system-on-azure-part-5/)
* [Part 6 - Building Frontend](https://tech-fellow.eu/2020/12/14/building-real-time-public-transport-tracking-system-on-azure-part-6-building-frontend/)
* [Part 7 - Packing Everything Up](https://tech-fellow.eu/2021/01/21/building-real-time-public-transport-tracking-system-on-azure-part-6-packing-everything-up/)
* [Part 7.1 - Fixing Docker Image Metrics in Azure](https://tech-fellow.eu/2021/01/25/building-linux-docker-images-on-windows-machine/)

## Background

In [previous blog post](/2020/01/21/building-real-time-public-transport-tracking-system-on-azure-part-2-data-collectors-composer/), we ended story with state when data is written to the Azure Storage table and blob combination. Collector functions together with Composer WebJob is running smoothly on more or less constant frequency and producing data steadily.

Now we have a task in front of us to deliver this information further to our beloved subscribers. Subscriber here means any other system that is interested in receiving this information and process it further - for example, push data down to the connected clients via browsers.

Of course subscribers could just periodically check for the new data by asking server every X seconds, for example. But then in this case I would not call them subscribers, but spammers. Azure infrastructure should be responsible for delivering this information to subscribers that new snapshot has been generated. This is reactive event-based system architecture.

Scanning [feature set documentation](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-blob-event-overview) for Azure Storage service you might find **EventGrid** keyword.

Essentially we will be implementing now everything that's needed from blue icon on the right side:

![5.2](/assets/img/2020/04/5.2.png)

## Reactive Event Publishing

For creating reactive event-based systems in Azure infrastructure EventGrid is ideal recipe for this.

But before we can proceed with EventGrid setup in Azure, we need to establish listener on the receiving side. This is required before setting up EventGrid because during EventGrid setup in Azure data is sent to the endpoint - so there has to be somebody listening on that address.

Otherwise you will end up with situation that endpoint verification just fails.

![eventgrid-endpoint-validation](/assets/img/2020/04/eventgrid-endpoint-validation.png)

### Create EventGrid Receiver

Before you can proceed with EventGrid subscription setup in Azure, you need to establish receiving endpoint otherwise endpoint validation will fail. There has to be somebody answering on that endpoint.

For this we will need a simple web application. Let's create WebAPI placeholder endpoint for this:

```csharp
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;

namespace MySampleProject.EventGridReceiver
{
    [ApiController]
    [Route("api/eg")]
    public class EventGridReceiverController : ControllerBase
    {
        [Route("receiver")]
        [HttpPost]
        public async Task<ActionResult> ReceiveEventGridPost([FromBody] object content /* not sure why but we need object for ModelBinder to work */)
        {
            await HandleEventAsync(content.ToString());

            return Ok();
        }

        public async Task<ActionResult> HandleEventAsync(string content)
        {
            // do the magic with received event
        }
    }
}
```

So far so good. Looks like endpoint is setup and it's ready to receive requests from EventGrid.

### Setup EventGrid in Azure

One of the way to create new EventGrid subscription is doing that in Azure Portal:
![Picture1](/assets/img/2020/04/Picture1.png)
Another way (when you feel hack-ish) is via Azure CLI.

If you haven't used EventGrid services before in your subscription, you might need to register this provider.

```
az login
az provider register --namespace Microsoft.EventGrid
```

Then we need to fetch FQN of the storage account which we would be using as source for the events (in our case it's "buffer" storage):

```powershell
$storageid=$(az storage account show --name buffer --resource-group <resource-group> --query id --output tsv)
```

You can define your endpoint as variable (`$endpoint`) or just pass inline.

```powershell
az eventgrid event-subscription create \
    --source-resource-id $storageid \
    --subject-begins-with 'transportData' \
    --name to-rt-map-endpoint \
    --endpoint $endpoint
    --endpoint-type webhook
```

However, after setup of this EventGrid subscription and waiting for a bit (you know, just in case) nothing really happens..
![Picture2-1](/assets/img/2020/04/Picture2-1.png)
I see no heartbeats (being sure that blobs are updated regularly in storage), nothing. It's single dead line.

### EventGrid Endpoint Verification

If I wouldn't be so lazy and would read what's been thrown at me, then I would be aware that in order to setup EventGrid event submission to custom webhook endpoint, you have to pass endpoint verification procedure.

Meaning that we need to reply with received verification token prior we are able to receive any events from EventGrid.

So we need to change our endpoint a bit:

```csharp
[ApiController]
[Route("api/eg")]
public class EventGridReceiverController : ControllerBase
{
    [Route("receiver")]
    [HttpPost]
    public async Task<ActionResult> ReceiveEventGridPost([FromBody] object content /* not sure why but we need object for ModelBinder to work */)
    {
        return await HandleEventAsync(content.ToString());
    }

    public async Task<ActionResult> HandleEventAsync(string content)
    {
        var eventGridSubscriber = new EventGridSubscriber();
        var eventGridEvents = eventGridSubscriber.DeserializeEventGridEvents(content);

        foreach (var eventGridEvent in eventGridEvents)
        {
            // EventGrid will send verification request and we need to reply with given code
            // otherwise event subscription creation will fail in Azure and no data will be sent to this endpoint
            if (eventGridEvent.Data is SubscriptionValidationEventData eventData)
                return Ok(new SubscriptionValidationResponse
                {
                    ValidationResponse = eventData.ValidationCode
                });

            if (eventGridEvent.Data is StorageBlobCreatedEventData)
            {
                // do the magic with event
            }
        }

        return Ok();
    }
}
```

Now it seems alive.

![Picture4-2](/assets/img/2020/04/Picture4-2.png)

## Max Delivery Attempts

**NB!** Be aware of default settings when creating EG subscriber endpoints.
[By default](https://docs.microsoft.com/en-us/cli/azure/eventgrid/event-subscription?view=azure-cli-latest#optional-parameters) EG is created with re-delivery mechanism in mind.

EventGrid will be instructed to keep an eye on failed messages. If infrastructure discovers that message could not be delivered (endpoint returns not 200 OK response) - message is scheduled for redelivery. By default redelivery count will be set to `30`.

At some point when your test environment was down, or even shut off and you become back online.. That moment when you are banging your head against table trying to find out the reason why your server suddenly is dying in the middle of the night, where there is almost no visitors.. :)

![Picture3](/assets/img/2020/04/Picture3.png)

As we are dealing with real-time data and lost events in our case for delivery sites does not matter. I mean if we lost event few minutes or even hours ago - no one really cares because visitors are interested in the most recent data. So we can easily ignore lost packages and not bother EG for scheduling redelivery attempt. You can configure redelivery attempt count while creating subscription:

```
az eventgrid event-subscription create \
    ...
    --max-delivery-attempts 1
```

## Next Steps

Now when we have received event from EG, we able to start implementing handler method. Basically we will need to fill in between the lines here:

```csharp
if (eventGridEvent.Data is StorageBlobCreatedEventData)
{
    // magic happens here
}
```

Happy eventing and stay safe!

[*eof*]
