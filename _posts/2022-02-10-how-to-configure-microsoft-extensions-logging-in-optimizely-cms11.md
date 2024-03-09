---
title: How to Configure Microsoft.Extensions.Logging in Optimizely CMS11
author: valdis
date: 2022-02-10 00:00:00 +0200
categories: [Add-On, Optimizely, Episerver, .NET, C#, Open Source, DeveloperTools, Logging]
tags: [add-on, optimizely, episerver, .net, c#, open source, logging]
---

You might be thinking that `Microsoft.Extensions.Logging` and related packages are useful only for .NET Core and above. That's not entirely true. While `Microsoft.Extensions.Logging` is target .NET Standard it's possible to use it even on .NET Framework 4.x. This is de-facto logging library that class libraries should use at very least. Where Optimizely (ex. Episerver) CMS11 (running on .NET Framework) would collide with anything from the "new world" (.NET5)?

![ms.logging-in-cms11-1](/assets/img/2022/02/ms.logging-in-cms11-1.png)

We do have many shared libraries across our codebase. As by design and name states - class libraries contain business logic and other sweet stuff. Main goal is to reuse and share the same business login across many rutimes and platforms as possible (and applicable).

Few years ago we refactored our shared class libraries to use `Common.Logging` as logging abstractions. That would allow us to reduce dependency on `log4net` coming from CMS platform. The same class libraries are using in other projects in our solution. Some of them even on .NET 6. To get everything in order and no exception during runtime - we had to pull in also `Common.Logging` dependency to get things working properly.

Depending on runtime (ASP.NET Core or Azure Runtime, or whatever) you might get to work with different logging abstractions and therefore you have to make sure that all moving parts are working properly together and there are all required adapters registered.

It is time for us to say good-bye to `Common.Logging`.

However if we throw out `Common.Logging` from our shared libraries and use `Microsoft.Extensions.Logging` abstractions, how can we be sure that we are not losing log entries when library will be used in CMS11 context where we have log4net and friends?

Therefore we have to trick DI container to pretend that it's possible to create `ILogger<T>` instances when anyone from shared libraries is requesting instance of it.

## Train StructureMap About MS.Extensions.Logging

First thing we have to do is to tell StructureMap what to do about `ILogger<T>` open generic (when someone requests this dependency).

```csharp
[InitializableModule]
public class DependencyResolverInitialization : IConfigurableModule
{
    public void ConfigureContainer(ServiceConfigurationContext context)
    {
        context.ConfigurationComplete += (o, e) =>
        {
            context.StructureMap().Configure(cfg =>
            {
                cfg.For(typeof(ILogger<>)).Use(new LoggerAdapterFactory());
            });
        };
    }

    public void Initialize(InitializationEngine context) { }

    public void Uninitialize(InitializationEngine context) { }
}
```

Type `LoggerAdapterFactory` is special StructureMap type that will instruct DI library what to do next.

It is of type `Instance` that is used by the library when StructureMap will have to "close" open generic.

```csharp
public class LoggerAdapterFactory : Instance
{
    public override Instance CloseType(Type[] types)
    {
        var instanceType = typeof(LoggerInstance<>).MakeGenericType(types);
        return Activator.CreateInstance(instanceType) as Instance;
    }

    // ignore
    public override IDependencySource ToDependencySource(Type pluginType) { throw new NotImplementedException(); }

    // this is just for the debugging
    public override string Description => "Build ILogger<T> with LoggerAdapterFactory";

    // what types this instance handles
    public override Type ReturnedType => typeof(ILogger<>);
}
```

We are "outsorcing" actual creation of the logger classes to `LoggerInstance`.
Let's take a look at `LoggerInstance`.

```csharp
public class LoggerInstance<TCategoryName> : LambdaInstance<ILogger<TCategoryName>>
{
    public LoggerInstance()
        : base(() => new ExtensionsLogger<TCategoryName>()) { }

    // just made debugging easier
    public override string Description =>
        "via LoggerInstance<" + typeof(TCategoryName).Name + ">";
}
```

Everytime StructureMap will see `ILogger<T>` dependency it will use these 2 types to construct instance of the requested dependency.

And `ExtensionsLogger` is actual bridge between `MS.Extensions.Logging` logger and `ILogger` from Optimizely CMS.

## Implement the Bridge

Actual implementation of logging bridge is quite simple -> we just have to translate calls from `Microsoft.Extensions.Logging` to Optimizely logger API.

```csharp
public class ExtensionsLogger<T> : ILogger<T>
{
    private readonly EPiServer.Logging.ILogger _innerLogger;

    public ExtensionsLogger()
    {
        _innerLogger = LogManager.GetLogger(typeof(T));
    }

    public void Log<TState>(LogLevel logLevel, EventId eventId, TState state, Exception exception,
        Func<TState, Exception, string> formatter)
    {
        switch (logLevel)
        {
            case LogLevel.Trace:
                _innerLogger.Trace(formatter(state, exception));
                break;
            case LogLevel.Debug:
                _innerLogger.Debug(formatter(state, exception));
                break;
            case LogLevel.Information:
                _innerLogger.Information(formatter(state, exception));
                break;
            case LogLevel.Warning:
                _innerLogger.Warning(formatter(state, exception));
                break;
            case LogLevel.Error:
                _innerLogger.Error(formatter(state, exception));
                break;
            case LogLevel.Critical:
                _innerLogger.Critical(formatter(state, exception));
                break;
        }
    }

    public bool IsEnabled(LogLevel logLevel)
    {
        switch (logLevel)
        {
            case LogLevel.Trace:
                return _innerLogger.IsTraceEnabled();
            case LogLevel.Debug:
                return _innerLogger.IsDebugEnabled();
            case LogLevel.Information:
                return _innerLogger.IsInformationEnabled();
            case LogLevel.Warning:
                return _innerLogger.IsWarningEnabled();
            case LogLevel.Error:
                return _innerLogger.IsErrorEnabled();
            case LogLevel.Critical:
                return _innerLogger.IsCriticalEnabled();
        }

        return false;
    }

    // no support for scopes for now
    public IDisposable BeginScope<TState>(TState state) { throw new NotImplementedException(); }
}
```

As we know category name (or `<T>` parameter for `ILogger`) we can ask `LogManager` to give us logger of that type: `LogManager.GetLogger(typeof(T));`.
This will ensure that log entries are decorated with correct category.

## Sample Usage

After configuring this properly in your app, you can ask for `ILogger<T>` dependnecy
 directly in your controller constructor, or you use class library that does this. It doesn't matter. Infrastructure is set up properly and logging is working as expected:

```csharp
public class StartPageController : PageControllerBase<StartPage>
{
    private readonly ILogger<StartPageController> _logger;

    public StartPageController(ILogger<StartPageController> logger)
    {
        _logger = logger;
    }

    public ActionResult Index(StartPage currentPage)
    {
        _logger.LogInformation("TEST FROM START PAGE");

        ...
    }
}
```

`ILogger<T>` interface is coming from `Microsoft.Extensions.Logging` namespace.

![ms.logging-in-cms11-2](/assets/img/2022/02/ms.logging-in-cms11-2.png)

After couple of seconds log entries show up also in log files (you will need to configure different settings from default ones to see also `Information` severity entries).

![ms.logging-in-cms11-3](/assets/img/2022/02/ms.logging-in-cms11-3.png)


Happy coding!

Smooth migrating to .NET5!

[*eof*]
