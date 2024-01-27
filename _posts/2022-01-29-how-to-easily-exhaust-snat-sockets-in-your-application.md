---
title: How To Easily Exhaust SNAT Sockets in Your Azure Function
author: valdis
date: 2022-01-29 00:15:00 +0200
categories: [Add-On, Optimizely, Episerver, .NET, C#, Open Source, DeveloperTools]
tags: [add-on, optimizely, episerver, .net, c#, open source]
---

So here we are again - late Friday evening, some sort of hotel lobby music in my headphones (I miss travel) and this chart on my screen. I'm reaching for my keyboard to type "studio" in my [PowerToys](https://github.com/microsoft/PowerToys?ref=tech-fellow.ghost.io) power-run bar. We are running out of network sockets to talk to any service, any API - basically anything. We are at SNAT port exhaustion.

## Queue is Full

It started out of sudden. Yes, we were in the middle of packing up for the release and there were quite of things to be released. So must be in between the lines of the new code. But there were many new things.

We noticed that out of the blue sky our azure functions started to fail with a somewhat weird error message:

```
2022-01-28 08:32:11.282 +00:00 [ERR] An unhandled exception has occurred while executing the request.
Microsoft.Data.SqlClient.SqlException (0x80131904): A network-related or instance-specific error occurred while establishing a connection to SQL Server. The server was not found or was not accessible. Verify that the instance name is correct and that SQL Server is configured to allow remote connections. (provider: TCP Provider, error: 0 - An operation on a socket could not be performed because the system lacked sufficient buffer space or because a queue was full.)
 ---> System.ComponentModel.Win32Exception (10055): An operation on a socket could not be performed because the system lacked sufficient buffer space or because a queue was full.
```

Yeah, maybe SQL server is dead (or it's doing some circus tricks as we are scaling and taking backups from time to time).

But then after some time, other services start to fail. This time access to Azure Storage tables:

```
2022-01-28 08:33:38.583 +00:00 [ERR] HTTP GET /api/... responded 500 in 33791.4264 ms
Microsoft.Azure.Cosmos.Table.StorageException: An operation on a socket could not be performed because the system lacked sufficient buffer space or because a queue was full. (mytable.table.core.windows.net:443)
 ---> System.Net.Http.HttpRequestException: An operation on a socket could not be performed because the system lacked sufficient buffer space or because a queue was full. (mytable.table.core.windows.net:443)
 ---> System.Net.Sockets.SocketException (10055): An operation on a socket could not be performed because the system lacked sufficient buffer space or because a queue was full.
```

This time error message is exactly the same: An operation on a socket could not be performed because the system lacked sufficient buffer space or because a queue was full.

Error message quickly leads to [similar issue](https://github.com/Azure/Azure-Functions/issues/1127?ref=tech-fellow.ghost.io) others have experienced. Someone is using too much..

## Looking For Analysis Report in Portal

Microsoft Azure portal provides really useful reporting and analysis tools to get started with the issue.

This is done via "Diagnose and solve problems" section in your function app. Choose "Availability and Performance" and then "SNAT Port Exhaustion".

![](/assets/img/2022/01/snat-report.png)

We can also even see a number of connections fail.

![](/assets/img/2022/01/Screenshot-2022-01-28-195148-1.png)

But unfortunately Azure for some reason is not able to show which endpoints are failing.

![](/assets/img/2022/01/Screenshot-2022-01-28-195213.png)

## Analyzing the Dump

We need to take a dump (memory) to move forward.

This is super easy done via "Diagnose and solve problems" section in your function app. Choose "Diagnostics tools" and then "Collect Memory Dump".

![](/assets/img/2022/01/menu.png)

Let's see what's inside the dump.

**Visual Studio** is just a perfect tool for the job of analyzing memory. After opening .dmp file you have to choose "Debug Managed Memory". It will take time. I've been analyzing memory dumps of size 14GB. Meanwhile, I could prepare my coffee. Twice.

First view what Visual Studio is opening shows heap view - basically, all top objects that reside in memory and GC is not able to wipe them off.

![](/assets/img/2022/01/top-types-in-mem.png)

**Timer**, **TimeQueueTimer**, **Amqp**, **EventHandler** and other types are WAAAYYY OFF above the normal count for a healthy application.

![](/assets/img/2022/01/amqp.png)

5k Amqp sessions to Azure ServiceBus?? There definitely is something wrong going on.

Let's inspect some of the instances..

Inspecting some first couple sessions - everything seems to be just fine, functions who need access to ServiceBus - is having a session.

Then I opened up some 29xxx th instance (which is way off the range).

![](/assets/img/2022/01/amqp-session-instance.png)

In order to understand to which SB topic/subscription this session is created, you should dig deeper and look for **Links[x].Settings.Target**.

![](/assets/img/2022/01/amqp-session-target.png)

Ok, now we know to which topic/subscription this session is pointing to. We can check out the function code.

## Review The Code

Our function is opening the listener to ServiceBus and receiving the messages in batches, process those, execution ends. No, we don't want to use binding here are we need to control the frequency of the function execution and not invoke every time a new message arrives on the topic.

```csharp
using Microsoft.Azure.ServiceBus;
using Microsoft.Azure.ServiceBus.Core;

private readonly MessageReceiver _messageReceiver;

public Function1()
{
    _messageReceiver =
        new MessageReceiver(..., EntityNameHelper.FormatSubscriptionPath(..., ...));
}

[FunctionName("Pump")]
public async Task Run([TimerTrigger("*/15 * * * * *")] TimerInfo myTimer, ILogger log)
{
    var messages = await _messageReceiver.ReceiveAsync(...);
    // process the messages...
}
```


Looks kinda legit. Creates a new instance of the receiver, receives all messages from the topic subscription, processes them, and disposes receiver. During the next execution - the same.

However - we are missing one important detail here. We are **NOT** reusing connection to the ServiceBus, but instead - opening a new one **EVERY** time function executes.

According to best practices - you should strive to do pooling, reusing, or any sort of action to avoid socket exhaustion.

## Fixing The Code

Let's try to fix the code to avoid this problem.

### Attempt #1

One way is to convert the message receiver to `static` field.

```csharp
using Microsoft.Azure.ServiceBus;
using Microsoft.Azure.ServiceBus.Core;

private static string _connectionString
private static string _topicName { get; set; }
private static string _subscriptionName { get; set; };

private static readonly Lazy<IMessageReceiver> _messageReceiver =
    new Lazy<IMessageReceiver>(() =>
        new MessageReceiver(
          _connectionString,
          EntityNameHelper.FormatSubscriptionPath(_topicName, _subscriptionName)));

public Function1(YourServiceOptions options)
{
    _serviceBusConnectionString = ...;
    _topicName = ...;
    _subscriptionName = ...;
}

[FunctionName("Pump")]
public async Task Run([TimerTrigger("*/15 * * * * *")] TimerInfo myTimer, ILogger log)
{
    var messages = await _messageReceiver.Value.ReceiveAsync(...);
    // process the messages...
}
```

We are declaring a message receiver as static `Lazy<T>` meaning that it should be initialized once and then reused every time we request for a `Value`.

This works, but it's considered to be [code smell](https://rules.sonarsource.com/csharp/RSPEC-3963?ref=tech-fellow.eu). Static fields should be initialized inline.

### Attempt #2

A nicer way to get around this issue I found was to hide the message receiver behind some abstraction and configure this abstraction in your dependency container to be singleton per application.

Define the receiver:

```csharp
public class ServiceBusMessageReceiver
{
    private readonly MessageReceiver _receiver;

    public EnturServiceBusMessageReceiver(string connectionString, string topicName, string subscriptionName)
    {
        _receiver = new MessageReceiver(serviceBusConnectionString,
                                        EntityNameHelper.FormatSubscriptionPath(topicName, subscriptionName));
    }

    public Task<IList<Message>> ReceiveAsync(int maxMessageCount, TimeSpan operationTimeout) =>
        _receiver.ReceiveAsync(maxMessageCount, operationTimeout);
}
```

Configure DI container:

```csharp
services.AddSingleton(sp =>
{
    var options = sp.GetRequiredService<YourServiceOptions>();

    return new ServiceBusMessageReceiver(
        options.ServiceBusConnectionString,
        options.TopicName,
        options.SubscriptionName);
});
```

And then you are able to use this service as any other injected dependency.

```csharp
private readonly ServiceBusMessageReceiver receiver;

public Function1(ServiceBusMessageReceiver receiver)
{
    _messageReceiver = receiver;
}

[FunctionName("Pump")]
public async Task Run([TimerTrigger("*/15 * * * * *")] TimerInfo myTimer, ILogger log)
{
    var messages = await _messageReceiver.ReceiveAsync(...);
    // process the messages...
}
```


## Pit of Success

This is of course is not functional architecture and there are no ports or adapters, but still - using the second approach (by defining your service as singleton dependency) you are falling into a [pit of success](https://blog.ploeh.dk/2016/03/18/functional-architecture-is-ports-and-adapters/?ref=tech-fellow.eu) as there is less room for errors that developer can make, less room for how developer instantiates dependencies. Everything service needs get injected into it.

Now we are happy to stare at the stabilization line after the late-night patch..

![](/assets/img/2022/01/stabilize.png)

Happy coding! Stay safe!

[*eof*]
