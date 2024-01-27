---
title: Azure WebJobs - Eliminate Gap between Runs
author: valdis
date: 2021-03-23 07:10:00 +0200
categories: [.NET, C#, Azure, WebJobs]
tags: [.net, c#, azure, webjobs]
---

Our real-time public transportation tracking systems consists of few Azure WebJobs (yes, web jobs - even in modern "execute your code on some random server you have never seen before and then dispose it" world).

Component which is based on Azure WebJobs is doing heavy calculations and we would like to max-out parallelization as much as possible without paying penalty for leaving the process boundaries (like potentially in for example Azure Functions fan-out/fan-in scenarios).

It's possible to define timer trigger step by specifying it in attribute usage.

```csharp
public static void DoTheWork(
    [TimerTrigger("*/1 * * * * *")]
    TimerInfo timerInfo)
{
    // heavy work ahead
}
```

As you can see using CRON expressions it's not possible to specify to run job under 1 second. The lowest step you can get is single second (1s) - `*/1 * * * * *`.

It's fine for the most use cases probably. But our requirement was to execute as frequent as possible. Every millisecond counts for us - so we were looking for a way how to reduce "pauses" between cycles.

It turned out that `TimerTrigger` also supports `TimeSpan` argument (which is represented as `string`). For example:

```csharp
public static void DoTheWork(
    [TimerTrigger("00:00:01")]
    TimerInfo timerInfo)
{
    // heavy work ahead
}
```

This is equivalent to run job every second.

As `string` is passed down to `TimeSpan` for parsing, you can also specify whichever expression you need:

```csharp
public static void DoTheWork(
    [TimerTrigger("00:00:00.0001")]
    TimerInfo timerInfo)
{
    // heavy work ahead
}
```

This will tell WebJobs runtime host that job needs to be executed every **1 millisecond** resulting in job being executed almost all the time.

Also by inheriting from `TimerSchedule` class and specifying step yourself.

```csharp
public class EverySecondTrigger : TimerSchedule
{
    public override bool AdjustForDST { get; } = false;

    public override DateTime GetNextOccurrence(DateTime now)
    {
        return now.AddMilliseconds(1);
    }
}
```

```csharp
public static void DoTheWork(
    [TimerTrigger(typeof(EverySecondTrigger))]
    TimerInfo timerInfo)
{
    // heavy work ahead
}
```

Your own custom trigger might be handy in cases when you need to control whether your job should be executed or not (changing configuration file for example) without recompiling the solution.

For example reading setting from `appsettings.json` file).

Program.cs:

```csharp
internal class Program
{
    public static IServiceProvider Services { get; set; }

    private static async Task Main(string[] args)
    {
        var builder = new HostBuilder();
        builder.ConfigureWebJobs(b =>
            {
                b.AddAzureStorageCoreServices();
                b.AddAzureStorage();
                b.AddTimers();
            })
            .ConfigureServices((context, services) =>
            {
                services.Configure<JobSettings>(context.Configuration.GetSection(nameof(JobSettings)));
            })
            .UseConsoleLifetime();

        var host = builder.Build();
        Services = host.Services;

        using (host)
        {
            await host.RunAsync();
        }
    }
}
```

EverySecondTrigger.cs:

```csharp
public class EverySecondTrigger : TimerSchedule
{
    private readonly bool _enabled;

    public EverySecondTrigger()
    {
        _enabled = Program.Services.GetService<IOptions<JobSettings>>().Value.TriggerEnabled;
    }

    public override bool AdjustForDST { get; } = false;

    public override DateTime GetNextOccurrence(DateTime now)
    {
        return _enabled ? now.AddMilliseconds(1) : DateTime.MaxValue.AddMilliseconds(-1);
    }
}
```


Happy scheduling!
[*eof*]
