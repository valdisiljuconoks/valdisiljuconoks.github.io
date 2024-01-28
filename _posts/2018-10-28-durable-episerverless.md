---
title: Durable Episerverless
author: valdis
date: 2018-10-28 15:30:00 +0200
categories: [Add-On, Optimizely, Episerver, .NET, C#, Azure, Azure Functions]
tags: [add-on, optimizely, episerver, .net, c#, azure, azure functions]
---

There been couple of times when we together with friend of mine ([Henrik Fransas](https://twitter.com/HenrikFransas)) have presented [Episerver and Azure Functions](/2017/11/21/episerver-azure-functions-episerverless/) (aka "episerverless") and how they play well together. However, looking at overall our sample application architecture - it still seems to be a bit brittle and composed together out of some small moving parts.

This is our initial architecture:

![](/assets/img/2018/10/2018-10-28_14-36-31.png)

As you can see there are lots of small parts, composed together, having queues in between functions (be sure to spell them correctly) and even also Service Bus topic to get out of the parallel processing (by having subscriptions to the topic from the Function2 and Function4).

It's working, but knowing me - I suppose there must be a better way to accomplish the same task.

## Durable Functions to the Rescue

[Azure Durable Functions](https://docs.microsoft.com/en-us/azure/azure-functions/durable-functions-overview) made it debut earlier this year. Thought it's good chance to try it and see how architecture might change when introducing this piece of black magic here.

So, let's get started.

### Entry Point Function

First of all - connection from Episerver up to Azure Function endpoint (to initiate whole pipeline of image process) does not change much. We still are talking to function via HttpClient instance:

```csharp
private async Task<HttpResponseMessage> CallFunctionAsync(string contentReference,
    byte[] byteData,
    string imageUrl)
{
    var req = new ProcessingRequest
    {
        FileId = contentReference,
        Content = byteData,
        Width = 150,
        ImageUrl = imageUrl
    };

    using (var content = new StringContent(JsonConvert.SerializeObject(req)))
    {
        content.Headers.ContentType = new MediaTypeHeaderValue("application/json");
        return await
            Global.HttpClient.Value.PostAsync(
                _settings.Settings.RequestFunctionUri,
                content).ConfigureAwait(false);
    }
}
```

Difference from the ordinary function (nondurable one) is that entry point function needs to initiate orchestractor, kick off durable process and return "something". This "something" is nothing else as simple `HttpResponseMessage` containg some magic auto-generated url end-points for consumer party to check durable process status. It's up to the consumer side of course to use thes endpoints of not, but just makes it possible to display some sort of progress report if needed.

This is now Function1 body:

```csharp
[FunctionName("Function1")]
public static async Task<HttpResponseMessage> Run(
    [HttpTrigger(AuthorizationLevel.Anonymous, "post")] ProcessingRequest request,
                                                        HttpRequestMessage req,
    [Blob("%input-container%/{FileId}")]                CloudBlockBlob outBlob,
    [OrchestrationClient]                               DurableOrchestrationClient starter,
                                                        TraceWriter log)
{
    log.Info("(Fun1) Received image for processing...");

    await outBlob.UploadFromByteArrayAsync(request.Content, 0, request.Content.Length);
    var analysisRequest = new AnalysisReq
    {
        BlobRef = outBlob.Name,
        Width = request.Width,
        ImageUrl = request.ImageUrl
    };

    var instanceId = await starter.StartNewAsync(nameof(ProcessingSequence), analysisRequest);
    var result = starter.CreateCheckStatusResponse(req, instanceId);

    return result;
}
```

Note, that we demanded from the Azure Functions runtime to give us something call `DurableOrchestrationClient`. This type isresponsible for allowing us to kick-off some durable process. Throught the orchestration client, we are able to start our process:

```csharp
var instanceId = await starter.StartNewAsync(nameof(ProcessingSequence), analysisRequest);
```

and also return response to the caller about how we are doing:

```csharp
var result = starter.CreateCheckStatusResponse(req, instanceId);

return result;
```

Once durable process is started, consumer party gets back similar response (instance `id` and access `code` will be different, also note that url themselves might be different depending on which runtime version you are running function on, this is v1.0):

```json
{
  "id": "b4950....2fdaa8",
  "statusQueryGetUri": "http://localhost:7071/admin/extensions/DurableTaskExtension/instances/b4950....2fdaa8?taskHub=DurableFunctionsHub&connection=Storage&code=gj1VN...nILg==",
  "sendEventPostUri": "http://localhost:7071/admin/extensions/DurableTaskExtension/instances/b4950....2fdaa8/raiseEvent/{eventName}?taskHub=DurableFunctionsHub&connection=Storage&code=gj1VN...nILg==",
  "terminatePostUri": "http://localhost:7071/admin/extensions/DurableTaskExtension/instances/b4950....2fdaa8/terminate?reason={text}&taskHub=DurableFunctionsHub&connection=Storage&code=gj1VN...nILg=="
}
```

Here is couple of notes to make:
* `id` - this is unique identifier of the orchestration instance (durable process)
* `statusQueryGetUri` - here we cna get more info about the status of the instance if needed
* `sendEventPostUri` - here we can send in custom event to the instance if needed
* `terminatePostUri` - and if none cares about this instance anymore, here we can kill it

Returning status check response makes it possible to implement something known as "[Async HTTP API](https://docs.microsoft.com/en-us/azure/azure-functions/durable-functions-http-api)" pattern.

### Orchestrator Function

Next, wex should look at orchestrator function itself.

```csharp
public static class ProcessingSequence
{
    [FunctionName(nameof(ProcessingSequence))]
    [StorageAccount("my-storage-connection")]
    [return: Queue("done-images")]
    public static async Task<AsciiArtResult> Run(
        [OrchestrationTrigger] DurableOrchestrationContext context,
        TraceWriter log)
    {
        var input = context.GetInput<AnalysisReq>();

        ...
    }
}
```

As you can see, core part of the function via which whole durable process can happen is `DurableOrchestrationContext` parameter.
We can now start to implement our workflow (and what is nice about whole this setup, is that workflow definition is code-first).

We need first to get out from the context incoming parameters (`analysisRequest`) that were sent to the function via `starter`:

```csharp
public static class Function1
{
    [FunctionName("Function1")]
    public static async Task<HttpResponseMessage> Run(
        ...
        [OrchestrationClient]                               DurableOrchestrationClient starter
        ...)
    {
        ...
        var instanceId = await starter.StartNewAsync(nameof(ProcessingSequence), analysisRequest);
        ...
    }
}
```

This is doable fromt the `context`:

```csharp
var input = context.GetInput<AnalysisReq>();
```

Now we are ready to proceed with `Function2`:

```csharp
[FunctionName(nameof(ProcessingSequence))]
[StorageAccount("my-storage-connection")]
[return: Queue("done-images")]
public static async Task<AsciiArtResult> Run(
    [OrchestrationTrigger] DurableOrchestrationContext context,
    TraceWriter log)
{
    var input = context.GetInput<AnalysisReq>();

    var visionResult = await context.CallActivityAsync<(string Description, string[] Tags)>(nameof(Function2), input);
    ...
```

Note that functions that take part in orchestration are not called functiosn anymore, but `Activity`. So call to the other function or activity is just as simple as `context.CallActivityAsync<TResponse>(...)`.

Knowing this we can now implement whole workflow quite easily:

```csharp
[FunctionName(nameof(ProcessingSequence))]
[StorageAccount("my-storage-connection")]
[return: Queue("done-images")]
public static async Task<AsciiArtResult> Run(
    [OrchestrationTrigger] DurableOrchestrationContext context,
    TraceWriter log)
{
    var input = context.GetInput<AnalysisReq>();

    var visionResult = await context.CallActivityAsync<(string Description, string[] Tags)>(nameof(Function2), input);
    var asciiResult = await context.CallActivityAsync<AsciiArtResult>(nameof(Function3), input);

    var adultContentResult = await context.CallActivityAsync<bool>(nameof(Function4), input);
    if(adultContentResult)
        await context.CallActivityAsync<TwilioSmsAttribute>(nameof(Function5), input);

    var result = new AsciiArtResult(asciiResult.BlobRef,
                                    ConfigurationManager.AppSettings["output-container"],
                                    visionResult.Description,
                                    visionResult.Tags);

    log.Info($"({nameof(ProcessingSequence)}) Finished processing the image.");

    return result;
}
```

### Activity Definition

Activity body implementation itself is not changed much from the days when it was ordinary function. The only thing we must change is how we decorate incoming parameters to the activity - now we have to use `[ActivityTrigger]` instead of other trigger types supported by Azure Functions.

**NB!** Parameters to the activities cannot be complex thingies, those should be serializable (due to orchestrator implementation and durable functions architecture - state of the orchestration instance is saved in Azure Storage along with incoming parameters):

```csharp
[FunctionName(nameof(Function2))]
public static async Task<(string Description, string[] Tags)> Run(
    [ActivityTrigger]                                      AnalysisReq request,
                                                           TraceWriter log)
{
    log.Info($"({nameof(Function2)}) Running image analysis...");
    ...
```

The rest of the activities should be changed in similar way.

### New Architecture Picture

So, once we have finished transition to durable functions, architecture picture changed dramatically:

![](/assets/img/2018/10/2018-10-28_14-38-09.png)

Happy orchestrating!

[*eof*]
