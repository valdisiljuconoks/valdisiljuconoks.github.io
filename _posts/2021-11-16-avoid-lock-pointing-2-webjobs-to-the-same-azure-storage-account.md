---
title: Avoid Lock - Pointing Two WebJobs to The Same Azure Storage Account
author: valdis
date: 2021-11-16 22:30:00 +0200
categories: [.NET, C#, Open Source, Azure, WebJobs, Background Services, Dependency Injection]
tags: [.net, c#, open source, debugging, azure, webjobs]
---

Yes yes, while the whole world around us is serverless (read "servers running on thin air"), we are still running on concrete servers and utilizing real CPUs.

Recently we had a requirement to run multiple copies of our webjobs in parallel (with different settings and configuration) doing similar but different work. Webjobs runtime uses `Singleton` lock approach to coordinate the execution of the job across the farm (if you have scaled-out your application). This is the way how runtime avoids duplicate executions of the same job. Webjobs runtime uses Azure storage blobs to accomplish a locking mechanism across multiple nodes in your cluster.

We wanted to skip extra storage account creation for the copy of the webjobs, but instead - we would like to use the same account because webjobs are doing completely different tasks and it's OK if they run simultaneously. But when you point both copies of the job to the same storage account one of the jobs will not be able to acquire the lock and therefore will stall until another job will release the lock.

![](/assets/img/2021/11/image.png)

We had to handle this somehow and workaround "single host" limitation.

## Surfing WebJob Host Source Code to Find The Lock Source

We would need to find who is in charge of issuing the locks and who is responsible for generating the path for the locks (`ba0d.../Keeper.ScheduledJob.Run.Listener`).

When you configure your webjob runtime, you are adding a couple of services to the `IServiceCollection` via `IHostBuilder` extension method.

```csharp
var builder = new HostBuilder();
builder.ConfigureWebJobs((context, b) =>
    {
        b.AddAzureStorageCoreServices();
        b.AddAzureStorage();
        b.AddTimers();

        ...
    })
```

`AddTimers()` method plays important role in this game.

If you look inside - then at the end `JobHostService` is added as [background service](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/hosted-services?view=aspnetcore-6.0&tabs=visual-studio&ref=tech-fellow.eu).

```csharp
services.TryAddEnumerable(ServiceDescriptor.Singleton<IHostedService, JobHostService>());
```

Let's look inside `JobHostService`. Some bla bla housekeeping things and call to `((IJobHost)_jobHost).StartAsync(cancellationToken)`.

Let's see what job host is doing..

During the initialization `IJobHost` creates `JobHostContext` where all the information about found functions - yes, in webjobs context each webjob is actually called "Function" - no doubt that explains why Azure Functions SDK is based on Azure WebJobs SDK ;)

Each function is represented by `FunctionDescriptor` - very similar concept as MVC does controller/action thingy and how context is represented in various filters for example those could be plugged into the pipeline.

Job host context has function descriptors, logger factory and what's not.

Part of the host context creation (`JobHostContextFactory`) factory also creates various instances of `IListener` which in turn is responsible for triggering (executing) functions when external initiator "appears" (for example in queue trigger case - when a message has been dropped in the queue).

Listeners however are created via `HostListenerFactory` type. And one of the constructor arguments is `SingletonManager singletonManager`. Type sounds familiar. Let's take a closer look at what it does.

`HostListenerFactory` is responsible for creating all instances of `IListener` for found functions. We have only a single function (webjob) in the project:

```csharp
public async Task Run(
    [TimerTrigger(typeof(ScheduledJobTrigger), RunOnStartup = true)]
    TimerInfo timerInfo)
{
    ...
}
```

Job host has knowledge about every single function found in the project. Apart of the function metadata, the function descriptor has info also about what type of listener is required for the function to run.

Our job is triggered on a timer (CRON expression at the end). Remember about that `AddTimers()` method and the beginning of the post? Well, let's see what it does actually:

```csharp
public static IWebJobsBuilder AddTimers(this IWebJobsBuilder builder)
{
    if (builder == null)
    {
        throw new ArgumentNullException(nameof(builder));
    }

    builder.AddExtension<TimersExtensionConfigProvider>()
        .BindOptions<TimersOptions>();
    builder.Services.AddSingleton<ScheduleMonitor, StorageScheduleMonitor>();

    return builder;
}
```

Nothing really much of it. But `TimersExtensionConfigProvider` is important here.

`Microsoft.Azure.WebJobs.Extensions.Extensions.Timers.TimersExtensionConfigProvider`provides info to the host and runtime what exactly are timer triggers and how to get more information about them - like what kind of listener is required for the timers to go off and kick the job. This knowledge transfer in WebJobs world is called "binding rules". And the type that carries this info for the timers is `TimerTriggerAttributeBindingProvider` who essentially is informing runtime that `TimerTriggerBinding` should be used if the host needs info about timers and what kind of listener they need.

`ITriggerBinding` interface (which is implemented by `TimerTriggerBinding`) has the method - `CreateListenerAsync`. For the timers listener is `Microsoft.Azure.WebJobs.Extensions.Timers.Listeners.TimerListener`.

Let's look at the listener:

```csharp
[Singleton(Mode = SingletonMode.Listener)]
internal sealed partial class TimerListener : IListener
{
    ...
}
```

Well, here we are - this listener is requiring `Singleton` which has to be acquired before firing off this trigger.

Let's jump back to `HostListenerFactory`. When it's asked to create a new listener, there is a check for the `Singleton` attribute. If that's present - the original listener is wrapped in `SingletonListener` to ensure that lock is acquired before starting the inner listener. `SingletonListener` is using `SingletonManager` to actually get the lock from the Azure storage blob.

What's inside `SingletonManager`?

Loads of stuff.. and `FormatLockId(...)`. This sounds right. Let's see what's there.

```csharp
public static string FormatLockId(FunctionDescriptor descr, SingletonScope scope, string hostId, string scopeId)
{
    if (string.IsNullOrEmpty(hostId))
    {
        throw new ArgumentNullException("hostId");
    }

    string lockId = string.Empty;
    if (scope == SingletonScope.Function)
    {
        lockId += descr.FullName;
    }

    if (!string.IsNullOrEmpty(scopeId))
    {
        if (!string.IsNullOrEmpty(lockId))
        {
            lockId += ".";
        }
        lockId += scopeId;
    }

    lockId = string.Format(CultureInfo.InvariantCulture, "{0}/{1}", hostId, lockId);

    return lockId;
}
```

This is most important line there `lockId = string.Format(CultureInfo.InvariantCulture, "{0}/{1}", hostId, lockId);`

Locks consists of two segments:

- `hostId` - passed in as an argument
- `lockId` - type full name (FQDN) and `scopeId` (if has one)

Seems legit. Let's see where `hostId` is coming from.

```csharp
internal string HostId
{
    get
    {
        if (_hostId == null)
        {
            _hostId = _hostIdProvider.GetHostIdAsync(CancellationToken.None).Result;
        }
        return _hostId;
    }
}
```

NB! I hope Microsoft engineers are super sure what they are doing when call `.Result` of the `HostId` producing `Task`.

So as it turns out - there is such an extension as `IHostIdProvider`. If webjob is scaled out to multiple instances and timers need singleton lock - there should be something that generates "the same" host id for both of these jobs. Something that is the same for all webjobs of the same source project. Let's see what does `IHostIdProvider`.

```csharp
private string ComputeHostId()
{
    // Search through all types for the first job method.
    // The reason we have to do this rather than attempt to use the entry assembly
    // (Assembly.GetEntryAssembly) is because that doesn't work for WebApps, and the
    // SDK supports both WebApp and Console app hosts.
    MethodInfo firstJobMethod = null;
    foreach (var type in _typeLocator.GetTypes())
    {
        firstJobMethod = FunctionIndexer.GetJobMethods(type).FirstOrDefault();
        if (firstJobMethod != null)
        {
            break;
        }
    }

    // Compute hash and map to Guid
    // If the job host doesn't yet have any job methods (e.g. it's a new project)
    // then a default ID is generated
    string hostName = firstJobMethod?.DeclaringType.Assembly.FullName ?? "Unknown";
    Guid id;
    using (var md5 = MD5.Create())
    {
        var hash = md5.ComputeHash(Encoding.UTF8.GetBytes(hostName));
        id = new Guid(hash);
    }

    return id.ToString("N");
}
```

Well this makes total sense now. It generates `Guid` out of hash produced by the hosting assembly's full name. If you run the same webjob instance on another node - it will produce the same `Guid` - resulting in failure to acquire lock - and thus skipping the execution. Which is perfectly fine. But not in our case :)

We need to add some extra salt to distinguish between one and another webjob hosts.

### Generating Unique Lock Path by Overriding HostId

As it turned out - we have to inject another `HostIdProvider` to get a unique host id and therefore be able to run more than a single webjob instance of the same source project.

For us, there is a configuration knob in the settings file that dictates which job it is (set by automatic deployment pipeline for us). We could use this to add some salt to the `hostId` generation process.

Let's implement our own host id provider:

```csharp
public class HostTypeIdProvider : IHostIdProvider
{
    private readonly string _hostType;
    private string _hostId;

    public HostTypeIdProvider(string hostType)
    {
        _hostType = hostType;
    }

    public Task<string> GetHostIdAsync(CancellationToken cancellationToken)
    {
        _hostId ??= ComputeHostId();

        return Task.FromResult(_hostId);
    }

    private string ComputeHostId()
    {
        var hostName = "Keeper-" + _hostType;
        Guid id;
        using (var md5 = MD5.Create())
        {
            var hash = md5.ComputeHash(Encoding.UTF8.GetBytes(hostName));
            id = new Guid(hash);
        }

        return id.ToString("N");
    }
}
```

And the only thing left is to inject this into the webjobs runtime. As most of the things nowadays are retrieved from IoC - it's super easy:

```csharp
private static async Task Main(string[] args)
{
    var builder = new HostBuilder();
    builder.ConfigureWebJobs((context, b) =>
        {
            b.AddAzureStorageCoreServices();
            b.AddAzureStorage();
            b.AddTimers();

            var c = new JobConfig();
            context.Configuration.GetSection(nameof(JobConfig)).Bind(c);

            b.Services.AddSingleton<IHostIdProvider>(new HostTypeIdProvider(c.HostType ?? "Unknown"));
        });
}
```

So now we are able to run two (or even more) instances of our webjobs using the same Azure storage account. Each of them produces different `hostId` and therefore also different lock `path`. That allows each of them to acquire the singleton lock and start execution of the job functions.

### Summary

This is more like a note for me - when you are creating libraries or frameworks, please think about consumers of your masterpiece and how they will be able to override and change your opinionated approach or architecture :)

Happy coding!

[*eof*]
