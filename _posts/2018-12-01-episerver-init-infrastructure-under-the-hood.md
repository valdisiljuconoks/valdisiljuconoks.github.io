---
title: Episerver Init Infrastructure - Under the Hood
author: valdis
date: 2018-12-01 21:00:00 +0200
categories: [Add-On, Optimizely, Episerver, .NET, C#, Scheduled Jobs]
tags: [add-on, optimizely, episerver, .net, c#, under the hood]
---

From the consumer perspective it looks like just a bunch of `IConfigurableModule` or maybe some of the `IInitializableModule` and the rest of the magic is hidden. Having couple of these in your Episerver solution - and everything works as expected.
But what really happens under the hood and how Episerver initialization system really works? This is a blog post about it.

## Consumer Perspective

From the consumer perspective, having a initializable module in Episerver platform is pretty straight forward. You create class, implement interface and decorate it with some metadata to be visible to Episerver scanning process and you are done.
There are 3 types of modules you can have:

* configurable module (`IConfigurableModule`) - this giving you a chance to configure service collection (aka IoC container);
* initializable module (`IInitializableModule`) - classical way of doing to stuff once and at the beginning of the site startup;
* http events aware initializable module (`IInitializableHttpModule`) - the same as classical initializable module, but allows also access to `HttpApplication` instance. This is useful if you need to subscribe to some web application lifetime events;

Module looks like this:

```csharp
[InitializableModule]
[ModuleDependency(typeof(InitializationModule))]
public class MyCustomInitModule : IInitializableModule
{
    public void Initialize(InitializationEngine context)
    {
        ...
    }

    public void Uninitialize(InitializationEngine context)
    {
        ...
    }
}
```

But how that everything sticks together and how Episerver is initializing whole infrastructure? That's interesting for me.

## Where Init Phase Starts?

When you sneak peek into your project's `Global.asax.cs` file you will notice that this type inherits from `EPiServer.Global` type (part of the `EPiServer.Cms.AspNet.dll`). This is done by Visual Studio templates automatically. When you are creating site from really "scratch" (which you probably should not do) - you will need to inherit from that type yourself.
When `EPiServer.Global` type is being created (in constructor), what we see there is following line:

```csharp
InitializationModule.FrameworkInitialization(HostType.WebApplication);
```

Body of the `FrameworkInitialization` method calls `InitializationEngine` setup.

![Untitled](/assets/img/2018/12/Untitled.png)

This engine then is responsible for setting up everything in order to initialize modules and setup couple of other framework stuff before Episerver is ready to serve requests.

## Initialization Engine Setup

There are couple of notable things that happens while `Global.asax.cs` sets up initialization engine.

### AppDomain Setup

Think this is pretty interesting as well, but is beyond the scope of this blog post. So I'll will dedicate additional post for this topic. But basically here in this step Episerver makes sure that assemblies are found and could be resolved in current AppDomain (this is **not** yet assembly scanning process).

### Legacy System Presence Check

Check of the presence for the old system assemblies:

* An exception will be thrown if there is an old Add-On system libraries (`EPiServer.Packaging.UI.dll` 3.0)
* Mirroring System is old (`EPiServer.MirroringService.dll` major version is not the same as running Episerver assembly version)
* Shell is old (`EPiServer.Shell.dll` major version is lower than `10`)
* Legacy assemblies are present (if any one of these are found in output directory then exception is raised `EPiServer.Implementation`, `EPiServer.BaseLibrary` or `EPiServer.WorkflowFoundation`). These are most of the time left-overs from any major Episerver upgrade process.

If any of these predicates turns out to be `true`, then initialization process is aborted.

## Engine Initialization (Execute Transition)

Initialization engine needs to scan all assemblies and register found modules and then initialize them (this is done through engine's state transitions).

Two states which engine needs to transits through:

* `PreInitialize` - Initializing `IConfigurableModule`s (this is uninitialized engine state which means that nothing has been done to the engine and it's ready to start calling configurable modules for setup)
* `Initializing` - Initializing `IInitializableModule`s (here engine is responsible to setup all initializable modules)

### Scanning Assemblies

There are couple of notable things that are done during assembly scanning process (in order to find all modules in current app domain):

* assembly type scanner is setup - this step basically sets up scanner to go through all available assemblies in current app domain and create lookup tables for found known types there (I think assembly and type scanner is worth its own blog post..?!);
* capturing all types that has one of the attributes - either `InitializableModuleAttribute` or `ModuleDependencyAttribute`

**NB!** Also pay attention that after all types with these attributes are found, instance of that type is created via `Activator.CreateInstance(..)`. Which prevents and makes it impossible to use construction injection pattern for managing dependencies for the initialization modules. This basically is "[constrained construction](http://blog.ploeh.dk/2011/04/27/Providerisnotapattern/#4c7b89305e744b22b9cadb8fd4f1d74e)" anti-pattern and should be avoided.

Instance is created and casted to `IInitializableModule` interface. You might be wondering - how Episerver is able to find all configurable modules? Those fall under the same category because:

```csharp
public interface IConfigurableModule : IInitializableModule { ... }
```

### Configurable Modules

Before `IConfigurableModule` modules can be configured, those should be sorted out by dependencies. Basically Episerver looks at all found modules and figures out which modules need to be called after which - establishing call order. For example:

```csharp
namespace EPiServer.Data
{
  [InitializableModule]
  [ModuleDependency(typeof (ServiceContainerInitialization))]
  public class DataInitialization : IConfigurableModule, IInitializableModule
```

This module needs to be called after `ServiceContainerInitialization` module has finished its configuration method.
Once modules are sorted by dependencies, there is nothing more special about configuring modules except ServiceLocator setup (described in next section). `ConfigureContainer` method is called passing in current configuration context (with access to service collection or IoC container).
Once whole configuration phase is finished, event `ConfigurationComplete` is raised - for anyone interested to perform any special action after service collection is setup and configurable modules are initialized.

### ServiceLocator Provider

Essential part around service configuration - is creation and setup of the ServiceLocator. This still is important part of the Episerver which ensures that developers can continue to use this "[anti-pattern](http://blog.ploeh.dk/2010/02/03/ServiceLocatorisanAnti-Pattern/)" :) Under the hood - yes, of course service locator most probably is required, but in my opinion, it shouldn't be exposed to public. But it definitely takes time to refactor this.
Anyway, default service locator provider implementation in Episerver is based on [StructureMap](http://structuremap.github.io/) library. So then how Episerver is able to find this provider and use it as a default one?
While Episerver is configuring modules - it needs to understand who is going to fulfill service locator responsibilities. It's done by scanning assemblies and looking for `[ServiceLocatorFactory]` attribute on assembly level.
There is an attribute in `EPiServer.ServiceLocation.StructureMap.dll` assembly (which you get when install `EPiServer.ServiceLocation.StructureMap` NuGet package):

```csharp
[assembly: ServiceLocatorFactory(typeof (StructureMapServiceLocatorFactory))]
```

This factory must implement `IServiceLocatorFactory` interface and will be used during service locator infrastructure setup. Interesting is fact that exception with message `"There is no dependency injection framework installed. Resolve this issue by installing NuGet package 'EPiServer.ServiceLocation.StructureMap'."` is thrown if Episerver is not able to find any assembly with this attribute. Should message change at the moment when someone will implement other IoC container adapter? :)

![Untitled2](/assets/img/2018/12/Untitled2.png)

After initialization context is constructed and everyone had their chance to configure it, `IServiceLocator` implementation is created via this `IServiceLocatorFactory` and stored as "famously" known `ServiceLocator.Current` instance.

###  Initializable Modules

Later engine transits into next phase - "Initializing". During this phase - main objective is to initialize all `IInitializableModule` instances. Engine itself is passed to each of the modules as argument of main initialization method - ensuring that initializable modules have access to service locator if needed and other types from the engine.
Order of the call is already known and sorted out in previous phase - when sorting was applied for configurable module list before the call.

Interesting part of this step is - transformations. It looks like a way to apply various transformations before module initialization. One of the built-in configuration transformation I found is - `LegacyDatabaseHandlerSetup`.

![Untitled3](/assets/img/2018/12/Untitled3.png)

Honestly - I'm not quite sure I understand full purpose of this step and it looks like something obsolete and leftover from previous generations.

### Engine States

These are main `InitializationEngine` states (there are more, but usage seems to be limited):

![Untitled4](/assets/img/2018/12/Untitled4.png)

* `PreInitialize` - this is default start position for the engine. At this state engine would start to configure all modules;
* `Initializing` - here engine would be ready to initialize known modules;
* `InitializeFailed` - if one of the module fails, engine sets this status and basically blows up with an exception;
* `InitializeDelayed` - this state we will look at a bit later;
* `InitializeComplete` - engine enters into this state when everything is done with the modules and `InitComplete` event handlers are being called;
* `Initialized` - everything is tip-top and Episerver is ready to proceed with other business before serving requests to visitors;

### Initializing Http Events

There are other type of initializable modules - `IInitializableHttpModule`. You might be asking what's special about them? Most probably nothing expect that they do have access to `HttpApplication` instance - which gives them opportunity to subscribe to any web application lifetime event if needed.

```csharp
public void InitializeHttpEvents(HttpApplication application) { ... }
```

You might be wondering why this is not responsibility of the `InitializationEngine`? I would bet that this is part of `InitializationModule` due to fact that module has access to `HttpApplication` and Episerver most probably wanted to keep engine "host agnostic" - meaning the fact that Episerver is running in IIS actually is just host "context" and not part of the engine itself. Engine could be called from unit tests, console application or any other host type.

## Delaying Initialization

There is a special mechanism built into initialization engine to delay execution of particular module (and all it's dependent child modules).

### Delaying Init Module

In case when you cannot continue for some reasons to initialize your logic further and need to wait until "normal" request is made to the site, you can do that by throwing particular type of the exception. Initialization engine reacts on `TerminateInitializationException` and behaves appropriately.

So, for example, you will write following code:

```csharp
[InitializableModule]
[ModuleDependency(typeof(EPiServer.Web.InitializationModule))]
public class DelayedInitModule : IInitializableModule
{
    public void Initialize(InitializationEngine context)
    {
        var resolver = context.Locate.Advanced.GetInstance<IUrlResolver>();
        var url = resolver.GetUrl(ContentReference.StartPage);
    }

    public void Uninitialize(InitializationEngine context) { }
}
```

Variable `url` will always contain `null`. Because `UrlResolver` is able to do its business only after routes have been registered. However, routes are registered on 1st "real" request to the site. Thus you cannot get address of the content until routes are registered.
However, there is a way to workaround this behavior and "continue" initialization routine after 1st real request to the site by throwing this exception:

```csharp
[InitializableModule]
[ModuleDependency(typeof(InitializationModule))]
public class DelayedInitModule : IInitializableModule
{
    public void Initialize(InitializationEngine context)
    {
        var resolver = context.Locate.Advanced.GetInstance<IUrlResolver>();
        var url = resolver.GetUrl(ContentReference.StartPage);

        if(url == null)
            throw new TerminateInitializationException("Astalavista baby! I'll be back..");

        // do business with `url`
    }

    public void Uninitialize(InitializationEngine context) { }
}
```

This is the way how you can tell Episerver that you want to continue later. But how framework is able to resume delayed modules?

### Initialization IIS Module

If one of the module throws this `TerminateInitializationException` exception (actually you can throw any exception and Episerver will try to resume init process anyways, but think it's good to throw appropriate exception showing your intention) - whole process is stopped and Episerver "continues" do other business but not initializing modules further.

Resume option comes from `InitializationModule` IIS module. You can find one in `web.config` file:

```xml
  <system.webServer>
    ..
    <modules runAllManagedModulesForAllRequests="true">
      <add name="InitializationModule" type="EPiServer.Framework.Initialization.InitializationModule, EPiServer.Framework.AspNet" preCondition="managedHandler" />
      ...
    </modules>
```

Episerver initialization IIS module is responsible for resuming any delayed initialization. Also it's responsible to initialize any HttpEvent aware modules in the system (implementing `IInitializableHttpModule` interface).

**NB!** But beware that module which is delaying kinda it's own initialization is actually acting as "circuit-breaker". Meaning that - if there are other modules in the list which are "sorted behind" delayed module (even without explicit dependency on delayed module) - those will not be called on "1st" engine initialization cycle.

## Uninitialize

Here in uninitialization phase is not a big deal. You have to be good citizen and close any connections, dispose all garbage you have collected so far, release any resources that you have acquired, etc. Basically exactly what you would do in `IDisposable` interface implementation.

Also it's good idea to unsubscribe from any events you have subscribed to before.

## Replacing InitializationEngine

Once I [had a challenge](https://world.episerver.com/forum/developer-forum/-Episerver-75-CMS/Thread-Container/2018/9/how-do-i-remove-or-hide-the-customized-search-block-type-from-editors/) to make it possible for initializable module to support constructor injection and avoid using ServiceLocator.
Despite initialization engine implements also interface (`IInitializationEngine`), it's not possible to replace engine implementation via any official way (except reflection) to plug your own engine.

```csharp
public class InitializationEngine : IInitializationEngine { ... }
```

But inside `InitializationModule` class we see following:

```csharp
if (InitializationModule._engine == null)
{
  ApplicationDomainInitializer.Instance.SetupAppDomain(hostType);
  InitializationModule._engine = new InitializationEngine(...);
```

And this:

```csharp
/// <summary>Exposed for (shady) unit test reasons</summary>
internal static InitializationEngine Engine
{
  get => InitializationModule._engine;
  set => InitializationModule._engine = value;
}
```

And also reference to engine is stored in field:

```csharp
private static InitializationEngine _engine;
```

Which basically means that it's impossible to add support to DI "from outside" unless you replace whole initialization engine.

However, looking at this problem from the other perspective, having service locator usage in initializable module - is not SO bad, as it's part of the composition root for the application, is called once and it could be that "dark" nasty corner of the app which none wants to walk close by.


Happy initializing!

[*eof*]
