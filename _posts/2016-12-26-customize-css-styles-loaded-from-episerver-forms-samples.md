---
title: Customize Css styles loaded from EPiServer.Forms.Samples
author: valdis
date: 2016-12-26 23:45:00 +0200
categories: [Add-On, .NET, C#, Episerver, Optimizely]
tags: [add-on, .net, c#, open source, episerver, optimizely]
---

This blog post will describe a way how to customize styles and scripts loaded by `EPiServer.Forms.Samples` package.

## Background

Unfortunately EPiServer.Forms package does not have `DateTime` picker available in standard package. So you need to download and reference sample package that contains reference scripts and form elements for some additional stuff. Among others one of them is date time picker. It's based on jQuery UI.

More or less everything is OK, until you need to customize look & feel of the date time picker.

The way how it's implemented in the sample package - is via `IViewModeExternalResources` interface. In the end it loads up bunch of style sheet files:

![](/assets/img/2016/12/2016-12-26_23-18-02.png)

Now the problem is that it's not always accepted to have built-in jQuery UI styles and project requires its own styling for date time picker of any other form element from that package.


## Solution 1

One of the way is to decompile source code from the package and reference it in the project directly. Or just fork [GitHub repo](https://github.com/episerver/EPiServer.Forms.Samples) and copy over all files.

## Solution 2

Looking into `IViewModeExternalResources` interface usage code (EPiServer is looking for all instances using `ServiceLocator.GetAllInstances<T>`). Which means that we can't just write something like:


```csharp
container.For<IViewModeExternalResources>().Use<CustomResources>();
```


This would just make EPiServer to use both: `EPiServer.Forms.Samples.ViewModeExternalResources` and `CustomResources` providers.

Will not even mention that `EPiServer.Forms.Samples.ViewModeExternalResources` uses `[ServiceConfiguration]` attribute, which is pretty [hard to spot](http://marisks.net/2016/12/01/dependency-injection-in-episerver/).

Straight forward solution for this problem is to "replace" existing mode resource provider with your own custom one. For this you can use StructureMap's `IContainer.Inject()` method, but it's not recommended.

Instead, there is better approach. You can decorate interface with your own "interceptor".

```csharp
[InitializableModule]
[ModuleDependency(typeof(InitializationModule))]
public class InitializationModule1 : IConfigurableModule
{
    public void Initialize(InitializationEngine context) { }

    public void Uninitialize(InitializationEngine context) { }

    public void ConfigureContainer(ServiceConfigurationContext context)
    {
        context.Container.Configure(
            cfg =>
                    cfg.For<IViewModeExternalResources>()
                       .DecorateAllWith<CustomResources>());
    }
}

public class CustomResources : IViewModeExternalResources
{
    public IEnumerable<Tuple<string, string>> Resources
    {
        get
        {
            ...

            yield return new Tuple<string, string>(
                "css",
                "/ClientResources/custom.css");
        }
    }
}
```

Key line is this:


```csharp
context.Container.Configure(
    cfg =>
            cfg.For<IViewModeExternalResources>()
               .DecorateAllWith<CustomResources>());
```


**NB!** The only downside now - all `IViewModeExternalResources` instances will be "intercepted". This might lead to some different issues when referencing other packages that do view mode client resource registration.

To fix this, you can check for the inner ("actual intercepted") provider and do replacement only if correct provider is being intercepted.

```csharp
public class CustomResources : IViewModeExternalResources
{
    private readonly IViewModeExternalResources _inner;
    private readonly bool _replace;

    public CustomResources(IViewModeExternalResources inner)
    {
        _inner = inner;
        if(inner is ViewModeExternalResources)
            _replace = true;
    }

    public IEnumerable<Tuple<string, string>> Resources
    {
        get
        {
            if(!_replace)
                return _inner.Resources;

            return new[]
            {
                new Tuple<string, string>("css",
                                          "/ClientResources/custom.css")
            };
        }
    }
}
```

This is pretty easy way to "replace" default styles for form elements that are not part of original package and those scripts and resources are loaded via resource provider that is "compiled into" .dll file.

Happy forming!

[*eof*]
