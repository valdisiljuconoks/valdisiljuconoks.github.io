---
title: Disposable Dependency for Azure WebJob
author: valdis
date: 2017-04-12 15:45:00 +0200
categories: [Add-On, .NET, C#, Azure]
tags: [add-on, .net, c#, azure]
---

Recently we had experience with Azure WebJobs hosting system and specificely - with disposable jobs. This blog post will describe how to properly handle disposable job dependency.

## Why Dependency?

You might ask, why I need to have dependencies for the Azure WebJob. The only purpose for WebJob would be to kick-off rest of the services and transfer control to them to do the work. According to some software architectural theories - WebJob is just the delivery mechanism, it's a trigger for other services to step up and carry out the business task. However, in this [pizza's crunchy outer edge](https://tech-fellow.eu/2016/10/17/baking-round-shaped-software/), composition of the services and other involved parties happens. Even if **WebJob** does not do much, it's still the [composition root](http://blog.ploeh.dk/2011/07/28/CompositionRoot/) which **plays** important **role** in whole application. It's the place where composition of object graph happens.

## Why Disposable Dependency?

There might be various types of dependencies required for the WebJob to run successfully. Some of them should be created every time requested, some of them should be singleton all the time. But some of them should be disposable.

By disposable - I mean that dependency is singleton while job instance is running, but gets new instance once new job is executed. So, if job `A` starts new instance of this job is created - `A1`, instance of disposable dependency `D` is also created - `D1`. If there are any service or anybody else requires this dependency, the same instance `D1` is passed to the service. But then, next time job starts - `A2` instance is created and also `D2` is being created. But `D2` is shared across all services within `A2` run cycle.

One of our **usage for disposable dependency** was tiny profiler class that we can use to collect more diagnostics data as we go along, and "dump" all stuff collected during web job run cycle to some more persistant storage. So we needed to maintain "singleton" instance behavior between various services, make it unique within every web job run cycle and have it disposable, so we could write down collected data to some storage.

One of the easiest way to accomplish this requirement - to utilize [nested containers](http://structuremap.github.io/the-container/nested-containers/) in StructureMap library. Our idea was to create nested container for every job run cycle and request services from there. Meaning that we can require the same dependency over and over again from nested container - the same instance should be returned. In StructureMap - nested container itself is treated as `Transient` scoped lifetime object.


## How Azure WebJob Composes Object Graph?

Now when we know that WebJob is composition root, question is how it composes needed object graph for the job?

When configuring Azure WebJobs, there is possibility to setup activator that will be used to create new instances of web jobs. In this case I'm using one of my favorite IoC libs [StructureMap](http://structuremap.github.io/). Code is simple enough (in `Program.cs`):

```csharp
static void Main()
{
    var container = ComposeContainer();
    var config = new JobHostConfiguration
        {
            JobActivator = new StructureMapActivator(container)
        };

    if(DataPumpConfig.IsDevelopment)
        config.UseDevelopmentSettings();

    config.UseTimers();

    var host = new JobHost(config);
    host.RunAndBlock();
}

private static IContainer ComposeContainer()
{
    var container = new Container();
    // here you configure everything needed for your container
    ....

    return container;
}
```

StructureMapActivator.cs:

```csharp
public class StructureMapActivator : IJobActivator
{
    private readonly Container _container;

    public StructureMapActivator(Container container)
    {
        if(container == null)
            throw new ArgumentNullException(nameof(container));

        _container = container;
    }

    public T CreateInstance<T>()
    {
        var function = _container.GetInstance<T>();
        return function;
    }
}
```

This is the way how WebJob host can compose functions using StructureMap IoC library. Of course you can also go with [Pure DI](http://blog.ploeh.dk/2014/06/10/pure-di/) here. It's just a question of taste.

## How Azure WebJob Host Handles Disposable?

Question is how Azure WebJob host handles disposable objects? Fiddling around source code of the web job hosting environment, found out that there is class named `FunctionInvoker<T>` that's responsible for maintaining job instances upon demand when executing one. Code is similar to this one:

```csharp
public async Task InvokeAsync(object[] arguments)
{
  var instance = this._instanceFactory.Create();
  using ((object) instance as IDisposable)
    await _methodInvoker.InvokeAsync(instance, arguments);
}
```

Meaning that if host can cast job instance to `IDisposable` - it's then wrapped into `using` statement. This will ensure that job has possibility to release all necessary resources when disposing job instance.

So you can have this job for instance:

```csharp
public class MyDisposableJob : IDisposable
{
    public MyDisposableJob(IDisposableService service)
    {
        ...
    }

    public void DoStuff([TimerTrigger("00:00:01")]TimerInfo timerInfo)
    {
        ...
    }

    public void Dispose()
    {
        // release all held resources
    }
}
```

So we do have now `IJobActivator` that is responsible creating job instances and also function invoke that is responsible for releasing disposable web job instances. Question is - when exactly and how do I need to release disposable dependencies? Who is responsible for that?

## When to Release (Issue with IJobActivator)?

Following best dependency injection practices - there is [triple "R"](http://blog.ploeh.dk/2010/09/29/TheRegisterResolveReleasepattern/) pattern when it comes to Dependency Injection. Meaning that there has to be "*Register*", "*Resolve*" and "*Release*" stages during host lifetime. "*Register*" might happen only once - in `JobHost` creation - when we were composing container. "*Resolve*" happens in `IJobActivator` - when host is asking to create new instance of the job. We are **missing** "*Release*"!

There has to be "*Release*" stage to properly dispose nested containers. `IJobActivator` is composition root. `FunctionInvoker<T>` is responsible for asking to make job instances via `CreateInstance()` method - which means there has to be "release hook" for the job activator to handle when function invoker has done its work and job is about being released (disposed).

Control flow is following:

![](/assets/img/2017/04/Drawing1.png)

So there is no explicit moment when job activator or whoever else would be able to handle job dispose event and do some black magic there. I came up with hacky workaround. Would love to hear any feedback..

## Releasing Disposable Dependency

How to resolve and then release these disposable dependencies from the nested container?
Here is what I came up with. First, we need to modify a bit our `StructureMapActivator`. We need to create nested container and then resolve requested job from that container. This is needed because of "singleton" instances for dependencies within the same job execution cycle.

```csharp
public class StructureMapActivator : IJobActivator
{
    private readonly Container _container;

    public StructureMapActivator(Container container)
    {
        if(container == null)
            throw new ArgumentNullException(nameof(container));

        _container = container;
    }

    public T CreateInstance<T>()
    {
        var nestedContainer = _container.GetNestedContainer();
        var function = nestedContainer.GetInstance<T>();

        return function;
    }
}
```

Anyway, there - we are still missing this "*Release*" stage - there is no room for the activator to know when function invoker has done its work and job is going to be disposed.

Knowing that all disposable jobs are handled correctly from job host perspective  - we introduce new type of jobs: a base class for all your jobs:

```csharp
public class BaseFunction : IDisposable
{
    ...
}
```

Idea is that we can pass over `nestedContainer` instance to job itself and at least job instance will be responsible for releasing that container - when function invoker has done its work..

```csharp
public class StructureMapActivator : IJobActivator
{
    ...

    public T CreateInstance<T>()
    {
        var nestedContainer = _container.GetNestedContainer(typeof(T).Name);
        var function = nestedContainer.GetInstance<T>();

        var disposableFunction = function as BaseFunction;
        disposableFunction?.SetChildContainer(nestedContainer);

        return function;
    }
}
```

and now we need to implement `SetChildContainer` method in our base job:

```csharp
public class BaseFunction : IDisposable
{
    private IContainer _childContainer;

    public void Dispose()
    {
        _childContainer?.Dispose();
    }

    public void SetChildContainer(IContainer nestedContainer)
    {
        _childContainer = nestedContainer;
    }
}
```

Here is our base job, we know precisely when job is being disposed - so we have possibility to hook into and release nested container when needed.


## Diagnostics Tracer as Disposable Dependency

We had requirement to collect tiny diagnostics around some of the our business processes to write them down to diagnostics storage. What we did is introduced interface for the rest of the services to rely on:

```csharp
public interface IProfilerWriter
{
    void Write(string message);

    IDisposable Measure(string messagePattern);
}
```

Method `Measure()` is really useful when you would like to **explicitly** measure performance of some of the methods or code block. You could write:

```csharp
public class MyDisposableJob : IDisposable
{
    public MyDisposableJob(IDisposableService service, IProfilerWriter profiler)
    {
        ...
        _profiler = profiler;
    }

    public void DoStuff([TimerTrigger("00:00:01")]TimerInfo timerInfo)
    {
        ...
        using(_profiler.Measure("Get records took: {0}"))
        {
            // retrieve records from somewhere
        }
    }
}
```

Then we have profiler implementation:

```csharp
public class ProfilerWriter : IProfilerWriter, IDisposable
{
    private readonly ConcurrentDictionary<DateTime, string> _messages =
        new ConcurrentDictionary<DateTime, string>();

    public void Dispose()
    {
        ...
    }

    public void Write(string message)
    {
        _messages.TryAdd(DateTime.UtcNow, message);
    }

    public IDisposable Measure(string messagePattern)
    {
        return new ProfilerCaptureScope(this, messagePattern);
    }
}
```

**NB!** Usually profiler measurements should be implemented as infrastructure code (decorators, interceptors, whatever). But sometimes you need to capture also small fraction of bigger process, so then explicitly taking samples could be effective.

Profiler scope class is another disposable that's running timer once it's created and stops it when disposed:

```csharp
public class ProfilerCaptureScope : IDisposable
{
    private readonly IProfilerWriter _profiler;
    private readonly string _messagePattern;
    private readonly Stopwatch _clock = new Stopwatch();

    public ProfilerCaptureScope(IProfilerWriter profiler, string messagePattern)
    {
        if(profiler == null)
            throw new ArgumentNullException(nameof(profiler));

        if(string.IsNullOrWhiteSpace(messagePattern))
            throw new ArgumentNullException(nameof(messagePattern));

        _profiler = profiler;
        _messagePattern = messagePattern;
        _clock.Start();
    }

    protected virtual void Dispose(bool disposing)
    {
        if(disposing)
        {
            _clock.Stop();
            _profiler.Write(string.Format(_messagePattern, _clock.ElapsedMilliseconds));
        }
    }

    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }
}
```

And then in `ProfilerWriter.Dispose()` method you can use any storage if you need to persist diagnostics trace for that particular job execution cycle (we are using Azure Table Storage for that). I'll skip implementation for write here..

## Proper Solution (Instead of Workaround)

Wondering what would be solution for this hacky workaround? If framework is going to ask you to create new instance of anything, it has to tell you also when it has done its work and is about to release created instance.

New definition of `IJobActivator` interface might look like this:

```csharp
public interface IJobActivator
{
  T CreateInstance<T>();

  ReleaseInstance<T>(T instance);
}
```

There is explicit "*Resolve*" and "*Release*" phase you can hook into and perform your tasks.

Happy web jobbing! :)

[*eof*]
