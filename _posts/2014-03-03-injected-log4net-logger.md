---
title: Injected log4net Logger
author: valdis
date: 2014-03-03 20:45:00 +0200
categories: [.NET, C#, Episerver, Optimizely]
tags: [.net, c#, episerver, optimizely]
---

Blog post is for personal reference for the cases when you need to get instance of some sort of logger from the Logging library you are using.

It depends on logging library that you use but usually you may get instance of logger itself by providing some metadata about calling site or producer of the log entries.

Code usually looks somewhat similar:

```csharp
private readonly ILog logger = LogManager.GetLogger(MethodBase.GetCurrentMethod().DeclaringType);
```

As you can see it’s not really ready for injection as it requires concrete type in order to initialize an instance.
I would expect to have possibility to wait for somebody to give me correct logger already configured with concrete producer of the log entries.

```csharp
public class MyController : Controller
{
    public MyController(ILog logger)
    {
    }
}
```

## Instruct DependencyResolver to create a logger

It depends on type of IoC framework you are using but majority of main players in that field provide a way to initialize and customize creation of a container.
Code snippet below uses [StructureMap](http://docs.structuremap.net/) library.

```csharp
public class StructureMapRegistry : Registry
{
    public StructureMapRegistry()
    {
        For().AlwaysUnique().Use(ctx =>
            LogManager.GetLogger(ctx.ParentType ?? ctx.BuildStack.Current.ConcreteType));
    }
}
```

So whenever somebody will ask for `ILog` interface from IoC container it will be constructed with proper producer type.

Happy logging!

[*eof*]
