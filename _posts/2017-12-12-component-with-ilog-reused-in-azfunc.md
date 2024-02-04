---
title: Fix Logging in Azure Functions when Reusing Your Component
author: valdis
date: 2017-12-12 23:30:00 +0200
categories: [.NET, C#, Logging, Azure Functions]
tags: [.net, c#, logging, azure functions]
---

There are cases when your project follows hype and you face the case when you need to reuse your component in serverless world. This blog post is about how to fix logging (I picked `Common.Logging`, but actual implementation does not matter) when reusing some of your components in Azure Functions.

## Existing Component

Most of the time I see that transition to functions or serverless computing is not done by rewriting component that does black magic and delivers business logic, but instead - just referencing it and invoking from function. In this scenario function became as just a hosting environment for the execution of the business logic component.

Let's say that we do have a component (or even list of components) that demands logging dependency via constructor injection:

```csharp
using Common.Logging;

public class SomeComponent
{
    private readonly ILog _logger;

    public SomeComponent(ILog logger)
    {
        _logger = logger;
    }

    public void DoSomeStuff()
    {
        _logger.Debug("Starting to do some stuff...");
        ...
    }
}
```

In order to initialize this component and use it - we need to obtain instance of `ILog` from `Common.Logging` library. It could be also any other logging library implementation. Let's jump to function it self and see how we can setup environment properly.

## Hosting Function

Any Azure Function that needs to do some kind of logging can "demand" `TraceWriter` to be injected.

```csharp
[FunctionName("Function1")]
public static void Run([TimerTrigger("*/5 * * * * *")]
                       TimerInfo myTimer,
                       TraceWriter log)
{
     log.Info($"C# Timer trigger function executed at: {DateTime.UtcNow}");
}
```

Messages send to `TraceWriter` do appear in console (if [required levels](https://docs.microsoft.com/en-us/azure/azure-functions/functions-host-json#tracing) for the tracing are set).

![](/assets/img/2017/12/2017-12-11_00-43-45.png)

So if I would construct component instance manually - supplied logger would be `NoOpLogger` (or similar type) - basically meaning that there is no logger for the component.

```csharp
var svc = new SomeComponent(LogManager.GetLogger(typeof(Function1)));
```

We need to stick together `TraceLogger` with `ILog` and forward all logging entries to console or file (during runtime).

## Creating Composition Root

Here in sample we do have just one component class that demands single dependency to `ILog` instance. However in real life there might be much more complex object graphs to compose. There are 2 options when it comes to object composition and dependency injection:

* you do everything manually and be happy with [pure DI](http://blog.ploeh.dk/2014/06/10/pure-di/)
* or you can leverage any object composition library (aka [DI containers](http://blog.ploeh.dk/2012/11/06/WhentouseaDIContainer/))

This time we could be lazy and go with composition library (again I picked one from my stack - [StructureMap](http://structuremap.github.io/)).

We will need following ingredients:

* **R**egistration phase - when we instruct container what are our mappings between abstractions and implementations
* **R**esolution phase - where we actually will be created requested service and related object graph (aka Composition Root)
* **R**elease - where we let it go

In my sample - registration is super simple - we just need to map single type and tell container how to obtain `ILog` instance.

```csharp
private static readonly Lazy<Container> _containerBuilder =
    new Lazy<Container>(() =>
        new Container(_ =>
                     {
                         _.For<ILog>().AlwaysUnique()
                          .Use(ctx => LogManager.GetLogger(ctx.ParentType));
                     })
        );
```

I'm holding container initialization inside `Lazy<T>` here just because I don't want to pay function startup fee - execute code and initialize container only when it's needed.

In function "entry point" (`static Run` method) we do have access to `TraceWriter` instance passed in by the hosting environment. We can make use of it now.

To hide complexity and increase reusability - I moved logic to new `ServiceBuilder` class:

```csharp
public class ServiceBuilder
{
    public ServiceBuilder(IContainer container, TraceWriter log)
    {
    }

    public T GetInstance<T>()
    {
        ...
    }
}
```

In order to maintain any disposable dependencies or more like singleton ones - it's good idea to create child container for every usage of the container - function run. Meaning that `ServiceBuilder` needs to create child container and also implement `IDisposable` pattern - to properly release the container.

```csharp
public class ServiceBuilder : IDisposable
{
    private readonly Lazy<IContainer> _childContainer;

    public ServiceBuilder(IContainer container, TraceWriter log)
    {
        if (container == null) throw new ArgumentNullException(nameof(container));
        if (log == null) throw new ArgumentNullException(nameof(log));

        _childContainer = new Lazy<IContainer>(container.CreateChildContainer);
    }

    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }

    public T GetInstance<T>()
    {
        return _childContainer.Value.GetInstance<T>();
    }

    protected virtual void Dispose(bool disposing)
    {
        if (disposing)
            _childContainer?.Value?.Dispose();
    }
}
```

When you are dealing with disposable objects - it's good idea to dispose those as well. Meaning that if you create dependency - in this case `ServiceBuilder`, as consumer you should but it into `using` statement. So this is how it would look in Azure Function now:

```csharp
[FunctionName("Function1")]
public static void Run([TimerTrigger("*/5 * * * * *")]
                       TimerInfo myTimer,
                       TraceWriter log)
{
    using (var builder = new ServiceBuilder(_containerBuilder.Value, log))
    {
        var service = builder.GetInstance<SomeComponent>();
        service.DoSomeStuff();
    }
}
```

So all the necessary dependencies for `ServiceBuilder` are provided and passed in via constructor.

## Fix Logging

Now - when we have `TraceWriter` available from Azure Function host and all services that will be used during function execution will be created via `ServiceBuilder` - we can fix logging and make it to use `TraceWriter` to output messages sent to `Common.Logging` library to be visible in console (if running locally) or function log files (during normal runtime).

For this happen we will need adapter (or actually factory) for the `Common.Logging` to create logger via that.

This is new constructor of `ServiceBuilder` now (here is only important line of code):

```csharp
public ServiceBuilder(IContainer container, TraceWriter log)
{
    ...
    LogManager.Adapter = new TraceWriterLoggerFactory(log);
}
```

And `TraceWriterLoggerFactory` is simple factory pattern class which is able to construct new loggers with passed in `TraceWriter` as final output.


```csharp
public class TraceWriterLoggerFactory : ILoggerFactoryAdapter
{
    private readonly TraceWriter _log;

    public TraceWriterLoggerFactory(TraceWriter log)
    {
        _log = log;
    }

    public ILog GetLogger(Type type)
    {
        return GetLogger(type.Name);
    }

    public ILog GetLogger(string key)
    {
        return new LoggerTraceWriterAdapter(key, _log);
    }
}
```

And `LoggerTraceWriterAdapter` adapter itself is just a class that implements `ILog` interface and has A LOT of methods to write all severity messages to the output writer.

```csharp
public class LoggerTraceWriterAdapter : ILog
{
    private readonly string _parentType;
    private readonly TraceWriter _actualLogger;

    public LoggerTraceWriterAdapter(string parentType, TraceWriter actualLogger)
    {
        _parentType = parentType;
        _actualLogger = actualLogger;
    }

    public void Info(object message)
    {
        _actualLogger.Info($"INFO {_parentType}: {message}");
    }

    public void Info(object message, Exception exception)
    {
        _actualLogger.Info($"INFO {_parentType}: {message}. Exception: {exception}");
    }

    ...

}
```

## Missing Features

Using this approach there are couple of missing features as well:

* at the moment there us no way to configure output format of the message (similar as you have seen probably in `log4net` config files)
* there is no way to configure severity levels for specific `ILog` instance (but this is possible via `tracing` element in `host.json` file).
* it's not possible to configure rolling strategy for log files (but looking at Azure Function machine file system via [Kudu console](https://github.com/projectkudu/kudu/wiki/Kudu-console) seems like log file rolling strategy is already in place)

## Summary

Anyway regardless of missing features, this approach gave us opportunity to "redirect" `ILog` messages from `Common.Logging` library (which was used all over the place in our components) to leverage `TraceWriter` from Azure Functions host without rewriting any line of our components.
And also gave us possibility to unify our service composition and object graph creation. Now all necessary injections and "preparation" work is done in `Service Builder` instead of each function itself.

Happy logging!

[*eof*]
