---
title: Capture Exception in Azure Functions Poison Queue Trigger
author: valdis
date: 2019-02-06 09:45:00 +0200
categories: [Add-On, .NET, C#, Azure]
tags: [add-on, .net, c#, open source, azure]
---

I'll not talk about how cool Azure Functions are (because they are cool), but will focus on pretty tiny but very important aspect when running functions - how to get exception details out of poison message when using `[QueueTrigger]` trigger on Azure Storage Queues and "normal" handling of the message just fails and runtime decides to move message to poison queue. Code samples are provided for Azure Functions V2, but guess it applies to previous version as well.

## Queue Message Handler

To have a queue message handler in Azure Functions is damn pretty simple.

```csharp
[FunctionName("HandleMessageFunction")]
public static async Task RunAsync(
    [QueueTrigger("incoming-queue")]
    MyQueueMessageObject message,
    ILogger log,
    ExecutionContext executionContext)
{
    // handling message here
}
```

What you do with your incoming message - that's entirely up to you.
If you fail for some reason to handle message properly (read - you throw exceptions during the handling), runtime will decide at some point to move your message to poison queue. You can read more about this mechanism [here](https://docs.microsoft.com/en-us/azure/azure-functions/functions-bindings-storage-queue#trigger---poison-messages). You can of course do `try\catch` option here as well, but then question is what exactly are you going to do inside `catch` to make a retry later? Best option is to just let an exception fly up to the runtime and delegate dequeue on next round.

## Handling Poison Messages

When you fail to handle message properly (there is even a threshold how many times runtime will retry to give you your message for processing before moving to the poison queue):

```json
{
    "version": "2.0",
    ...
    "extensions": {
        "queues": {
            "maxDequeueCount": 3
        }
    }
}
```

To get notifications about bad messages that end up in poison queue (max dequeue threshold reached) you can create another function with trigger on `{your-queue-name}-poison`:

```csharp
[FunctionName("HandleErrorFunction")]
public static async Task RunAsync(
    [QueueTrigger("incoming-queue-poison", Connection = "AzureWebJobsStorage")]
    MyQueueMessageObject poisonMessage,
    ILogger log,
    ExecutionContext executionContext)
{
    ...
}
```

Again, here what you do with poison message - is up to you.

But how to get exception details from failed "normal" handling process?

There are [couple of hacky solutions provided](https://stackoverflow.com/questions/46852385/how-to-determine-reason-for-poison-queue-message) (including implementing your own interceptor - `FunctionInvocationFilterAttribute` or implementing custom `IQueueProcessorFactory` which is responsible for the logic how message is moved to the poison queue.

## Getting Exception Details

There is another alternative to get an exception details that was thrown while handling incoming message. Still might involve some hacky workarounds, but thought it's worth sharing.

In [hacky solutions in SO](https://stackoverflow.com/questions/46852385/how-to-determine-reason-for-poison-queue-message) there was a need to store exception details somewhere (if you choose `FunctionInvocationFilterAttribute` option). I would like to see exception together with incoming poison message - it's easier to reason about and handle further notification if needed.

For this work, we will have to return to original "normal" message handling function.

Note that we were asking runtime to bind our incoming message to our strongly typed object model:

```csharp
[FunctionName("HandleMessageFunction")]
public static async Task RunAsync(
    ...
    MyQueueMessageObject message,
    ...)
{
    // handling message here
}
```

There are couple of known types to which runtime can bind your incoming queue message. One of the possibility is to bind to very base class - `CloudQueueMessage`.

```csharp
[FunctionName("HandleMessageFunction")]
public static async Task RunAsync(
    ...
    CloudQueueMessage queueMessage,
    ...)
{
    // handling message here
}
```

This type will be needed later.
**NB!** The only disadvantage of the approach is that if we want to work in strongly-typed object model for the queue message - we need to deserialize it back.

```csharp
[FunctionName("HandleMessageFunction")]
public static async Task RunAsync(
    ...
    CloudQueueMessage queueMessage,
    ...)
{
    var message =
        JsonConvert.DeserializeObject<MyQueueMessageObject>(queueMessage.AsString);

    // handling message here
    ...
}
```

Now in order to capture exception details we need of course back `try/catch` statement:

```csharp
[FunctionName("HandleMessageFunction")]
public static async Task RunAsync(
    ...
    CloudQueueMessage queueMessage,
    ...)
{
    var message =
        JsonConvert.DeserializeObject<MyQueueMessageObject>(queueMessage.AsString);

    try
    {
        // handling message here
        ...
    catch (Exception ex)
    {
        message.ExceptionDetails = e.ToString();
        queueMessage.SetMessageContent(
            JsonConvert.SerializeObject(composedMessageItem));

        throw;
    }
}
```

Now we need to add `ExceptionDetails` property to our object model:

```csharp
public class MyQueueMessageObject
{
    ...
    public string ExceptionDetails { get; set; }
}
```

I haven't tried message object model `Exception` type property - might not work. String representation was enough.

Now you are able to get exception details inside your poison queue handler and decide where and how you are going to deliver exception to the responsible personnel.

```csharp
[FunctionName("HandleErrorFunction")]
public static async Task RunAsync(
    [QueueTrigger("incoming-queue-poison", Connection = "AzureWebJobsStorage")]
    MyQueueMessageObject poisonMessage,
    ILogger log,
    ExecutionContext executionContext)
{
    ...
    var notif = new MailMessage
                {
                    Body = poisonMessage.ExceptionDetails
                }
}
```

**NB!** When you will receive `CloudQueueMessage` on subsequent retries (after you failed to process it before) `ExceptionDetails` property will not be filled in. It gets "filled in" only on last retry before message is moved to the poison queue. I'm just speculating here (and haven't checked source code of the queue processor) but it looks like that message "as whole object" is updated and stored back in storage only when it's being moved to the poison queue. During "normal" handling retry cycles runtime just updates `dequeue` count of the message but body remains the same.

There are couple of ways to solve this problem, we liked this approach as it involved less code, exception details were always present within poison message and we do not need to inject some custom processors and interceptors in runtime to get this done.

May no poison messages be with you!

[*eof*]
