---
title: Create Your Library Configuration API for EPiServer
author: valdis
date: 2016-02-23 10:30:00 +0200
categories: [.NET, C#, Episerver, Optimizely]
tags: [.net, c#, episerver, optimizely]
---

Time comes when you need to extent EPiServer functionality by providing your own library with new or modified functionality.
I'm big fan of configuration via code. Which may give you possibility to configure some features with `LambdaExpression`, e.g. `() => true` to ensure that you are able to set feature's state `on` or `off` during runtime. This opens up quite interesting scenarios that I'm going to cover soon enough in other blog post.

Anyway, what I'm looking for is to make sure that my library consumers can do something like this:

```csharp
[ModuleDependency(typeof (InitializationModule))]
public class MyLibInit : IInitializableModule
{
    public void Initialize(InitializationEngine context)
    {
        ConfigurationContext.Setup(ctx =>
                                  {
                                      ctx.ConfigOption = true;
                                  });
    }

    public void Uninitialize(InitializationEngine context) { }
}
```

From the "end-user" perspective everything seems quite logic -> following best practices for EPiServer development - startup initialization is moved to `InitializableModule`.

## Module Call Order

Regardless that "end-user" code (`MyLibInit.cs`) seems OK, this makes it a bit challenging to set everything up correctly on library's side. The issue here is with order of calls for `InitializableModule` initialization code. Following best practices for EPiServer development also on library's side - we would need to setup our library via `InitializableModule` as well (`LibInit.cs`).

This is not guaranteed in EPiServer - which means that sometimes your "end-user" module may be called *before* library's init module - this is perfect timing of the modules - library's module can now read configuration values setup by library consumer code:


![](/assets/img/2016/02/2016-02-22_23-35-50-2.png)


However, sometimes EPiServer may call library init module *before* "end-user" module - which means that configuration settings will have no effect (as they are set after library is initialized):


![](/assets/img/2016/02/2016-02-22_23-42-20.png)



## Create Mid-Dependency Module

One thought that I gave chance to prove itself was to create intermediate fake dependency module - that library "end-user" module could depend upon and then create another module internally that would also be dependent on this fake module. And then add "actual" initialization module (`LibInit.cs`) as dependent module for this fake module. Visually it may look something like this:


![](/assets/img/2016/02/2016-02-22_23-49-00-1.png)


However, in this case (and I haven't dig into more details) case was so that EPiServer made sure that `LibInit` is called before `MyLibInit`. I'm speculating here that EPiServer makes sure that "longest" dependency chain gets called first.


## Solution

Actually solution for this case of dependency juggling is **really** simple. Fortunately EPiServer exposes few events that you can hook onto and add your logic after all of the modules have done their job:


![](/assets/img/2016/02/2016-02-22_23-57-58-1.png)


Using event subscription - it's guaranteed that your event handler will be called *after* all modules have done their job - including one that setup your configuration context.

In code it looks really simple:

```csharp
[InitializableModule]
[ModuleDependency(typeof (InitializationModule))]
public class LibInit : IInitializationModule
{
    private bool _eventHandlerAttached;

    public void Initialize(InitializationEngine context)
    {
        if (_eventHandlerAttached)
        {
            return;
        }

        context.InitComplete += ContextOnInitComplete;
        _eventHandlerAttached = true;
    }

   private void ContextOnInitComplete(object sender, EventArgs eventArgs)
   {
        ...
```

## Solution with IContainer Access

Solution seems to be simple enough to configure your own library behavior based on what "end-user" has configured.

Sometimes you may need to swap out some stuff from IoC container and replace with your own stuff (for instance, if "end-user" is willing to do so and had configured your library with that setting).

`IContainer` is not accessible in `InitializableModule` interface. But EPiServer gives access to dependency container while executing collection of `IConfigurableModule`.

Which means that we just need to refactor our module to implement `IConfigurableModule` and then capture instance of `IContainer` and inject things we need later on:

```csharp
[InitializableModule]
[ModuleDependency(typeof (InitializationModule))]
public class LibInit : IConfigurableModule
{
    private IContainer _container;
    private bool _eventHandlerAttached;

    public void Initialize(InitializationEngine context)
    {
        if (_eventHandlerAttached)
        {
            return;
        }

        context.InitComplete += ContextOnInitComplete;
        _eventHandlerAttached = true;
    }

    public void ConfigureContainer(ServiceConfigurationContext context)
    {
        _container = context.Container;
    }

    private void ContextOnInitComplete(object sender, EventArgs eventArgs)
    {
        _container.Configure(ctx => ctx.For<X>().Use<Y>());
    }
}
```

Subscription to the `InitComplete` event makes it possible to do things in your library after your consumers have configured stuff via their initializable modules.

Happy coding!

[*eof*]
