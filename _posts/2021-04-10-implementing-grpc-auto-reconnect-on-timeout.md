---
title: Implementing gRPC Auto-Reconnect on Timeout
author: valdis
date: 2021-04-10 22:10:00 +0200
categories: [.NET, C#, Open Source, gRPC, Azure]
tags: [.net, c#, open source, grpc, azure]
---

In this blog post we are going to implement feature to reconnect to gRPC stream if specific timeout has elapsed and no new data has been received from the server.

Part of our real-time [public transportation tracking system](https://tech-fellow.eu/2019/12/08/building-real-time-public-transport-tracking-system-on-azure-part1/) built on Microsoft Azure platform is also a gRPC service - providing near to the real-time data feed to the consuming parties. gRPC service is running as an experiment aside from [SignalR](https://dotnet.microsoft.com/apps/aspnet/signalr) service which has been around for a while in our infrastructure.

Over the time we see that data size transferred on the wire reduced and battery consumption dropped by using gRPC instead of SignalR stream. Which is a good excuse for use to keep going with gRPC.

We are continuing to explore possibilities gRPC infrastructure has to offer. One of the great features of SignalR library is its built-in option to [automatically reconnect](https://docs.microsoft.com/en-us/aspnet/signalr/overview/guide-to-the-api/handling-connection-lifetime-events#how-to-continuously-reconnect). gRPC has somewhat similar feature - ["retry policy"](https://docs.microsoft.com/en-us/aspnet/core/grpc/retries?view=aspnetcore-5.0). However retry policy applies only to unary calls. In our case unary calls are more like auxiliary invokes service some classifiers or static data. In gRPC we are using server side streaming - meaning that we can't use built-in retries and need to handle this ourselves.

One of the gRPC consuming applications is our own monitoring dashboard - where we are constantly consuming gRPC streams and any delays for receiving data packets means red code for us and the rescue team is most probably already on its way.
Monitoring dashboard is written in C# - so consuming gRPC service is super easy there.

## Task-based Reconnect

One of the first approach to implement retry was to fire off 2 tasks - one to receive packet on the stream from gRPC service, another - to control timeout. We would need to raise some sort of flag that no new data has been received for X amount of time, resulting in some alert action in monitoring system.

We gonna implement receiver as `BackgroundService` in .NET 5.

```csharp
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    var stream = _subscriber.GetStream();
    Task<bool> nextDataTask;

    while (!stoppingToken.IsCancellationRequested)
    {
        nextDataTask = stream.MoveNext(stoppingToken);

        // wait for the incoming data within some threshold
        // if threshold will be exceeded - we will subscribe again to the stream to get new connection
        var winningTask = await Task.WhenAny(nextDataTask, Task.Delay(_options.GrpcStreamTimeout, stoppingToken));
        if (winningTask == nextDataTask && nextDataTask.IsCompletedSuccessfully)
        {
            var gotNewItem = await nextDataTask;

            if (gotNewItem)
            {
                // do some magic with `stream.Current` item
            }
        }
        else
        {
            // we are because winning task is timeout task
            // or next data task has failed

            // OPTIONAL: we can look inside next data task (`nextDataTask.Exception?`)
            // maybe there is an exception
            // that should be good practice to log it at least

            // other than that - we just subscribe once again to the stream
            // this will result in new connection and old one will be dropped
            stream = _subscriber.GetStream();
        }
    }
}
```

And getting stream from the server is straight forward.

```csharp
public IAsyncStreamReader<DataItems> GetStream()
{
    _channel = GrpcChannel.ForAddress(...);
    _client = new GrpcDataFeed.GrpcDataFeedClient(_channel);
    _call = _client.SubscribeToStream();

    return _call.ResponseStream;
}
```

Code works, but it looks messy.
Also would like to wait a bit before retrying again - just to give some time for the server to recover (in case of disaster).

### Add Resilience with Polly

To help complete this task - we can rely on some retry policy libraries. One of the most popular in .NET world is [Polly](https://github.com/App-vNext/Polly).

Here is the same code, but with Polly retry forever giving recovery window for 15 seconds before each retry.

```csharp
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    var stream = _subscriber.GetStream();
    Task<bool> nextDataTask;

    var policy = Policy
        .Handle<Exception>()
        .WaitAndRetryForeverAsync(
            attempt => TimeSpan.FromSeconds(15),
            (exception, timespan) =>
            {
                _logger.LogError(exception, "Error handling gRPC stream. Reconnect.");

                // other than that - we just subscribe once again to the stream
                // this will result in new connection and old one will be dropped

                stream = _subscriber.GetStream();
            });

    while (!stoppingToken.IsCancellationRequested)
    {
        await policy.ExecuteAsync(async () =>
        {
            nextDataTask = stream.MoveNext(stoppingToken);

            // wait for the incoming data within some threshold
            // if threshold will be exceeded - we will subscribe again to the stream to get new connection
            var winningTask = await Task.WhenAny(nextDataTask, Task.Delay(_options.GrpcStreamTimeout, stoppingToken));
            if (winningTask == nextDataTask && nextDataTask.IsCompletedSuccessfully)
            {
                var gotNewItem = await nextDataTask;

                if (gotNewItem)
                {
                    _dataBuffer.SetNewOptimizedStreamState(stream.Current);
                }
            }
            else
            {
                // we are because winning task is timeout task
                // or next data task has failed

                // we can look inside next data task (`nextDataTask.Exception?`)
                // maybe there is an exception
                // that should be good practice to log it at least

                if (nextDataTask.IsCompleted && nextDataTask.Exception?.InnerException != null)
                {
                    throw new Exception("Error receiving data from gRPC endpoint.", nextDataTask.Exception?.InnerException);
                }
            }
        });
    }
}
```

## Better Approach - Cancellation Token based Reconnect

Task based code looks a bit messy. Think it could be improved.
Meanwhile we also can rewrite code to use async enumeration via `await foreach` statement.

This is the same logic but rewritten to use `CancellationToken` instead:

```csharp
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    var stream = _subscriber.GetStream();
    var cts = new CancellationTokenSource(_options.GrpcStreamTimeout);

    var policy = Policy
        .Handle<Exception>()
        .WaitAndRetryForeverAsync(
            attempt => TimeSpan.FromSeconds(15),
            (exception, timespan) =>
            {
                _logger.LogError(exception, "Error handling gRPC stream. Reconnecting...");

                // we haven't received data for more than allowed threshold
                // reconnecting to the service again
                stream = _subscriber.GetStream();

                // here we are recreating cancellation token
                // and need to take into consideration that parameter `timespan`
                // is wait duration for the next execution.

                // meaning that we have to set cancellation token expiration
                // to `waitDuration + timeout`
                // otherwise token will be already cancelled when retry policy will execute

                cts = new CancellationTokenSource(timespan + _options.GrpcStreamTimeout);
            });

    await policy.ExecuteAsync(async () =>
    {
        await foreach (var message in stream.ReadAllAsync(cts.Token))
        {
            _dataBuffer.SetNewOptimizedStreamState(message);
            cts.CancelAfter(_options.GrpcStreamTimeout);
        }
    });
}
```

Hope this helps!
Happy reconnecting!

[*eof*]
