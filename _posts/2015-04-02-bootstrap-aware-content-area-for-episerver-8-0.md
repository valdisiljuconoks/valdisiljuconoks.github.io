---
title: Bootstrap aware Content Area for EPiServer 8.0
author: valdis
date: 2015-04-02 21:30:00 +0200
categories: [.NET, C#, Episerver, Optimizely, Bootstrap]
tags: [.net, c#, episerver, optimizely, bootstrap]
---

I got pull request from [Mathias Nohall](https://github.com/mathiasnohall) on GitHub asking for adding support for EPiServer v8.0. This is what we call open-sourcing :). I was thinking this is good moment to introduce some breaking changes in EPiBootstrapArea plugin and give you fresh package compatible with v8.0 of EPiServer and also cleaned-up from some left-overs (mainly pointed out by [Mark’s pull request](https://github.com/valdisiljuconoks/EPiBootstrapArea/pull/3)).

## Changes

Here is list of changes that has been applied for this release:

- New referenced EPiServer NuGet package is version 8.3;
- DDS stored DisplayModeFallback is now more optional;
- Primary list of DisplayModeFallbacks are retrieved from `EPiBootstrapArea.Providers.DisplayModeFallbackDefaultProvider` provider;
- It’s possible to provide additional display modes from the code or from the DDS storage;


## Supply Additional Display Modes

It’s not an exception that your project may require additional or totally another set of display modes. If this is your case and you need to provide additional display modes, all you need to do is swap default provider and override `GetAll()` method.

Swap default provider:

```csharp
[InitializableModule]
[ModuleDependency(typeof(EPiServer.Web.InitializationModule))]
public class RegisterDisplayModesModule : IConfigurableModule
{
    public void ConfigureContainer(ServiceConfigurationContext context)
    {
        context.Container.Configure(ConfigureContainer);
    }

    private static void ConfigureContainer(ConfigurationExpression container)
    {
        container.For<IDisplayModeFallbackProvider>()
                 .Use<DisplayModeFallbackCustomProvider>();
    }

    public void Initialize(InitializationEngine context)
    {
    }

    public void Uninitialize(InitializationEngine context)
    {
    }

    public void Preload(string[] parameters)
    {
    }
}
```

Implement your own set or add additional display modes:

```csharp
public class DisplayModeFallbackCustomProvider : DisplayModeFallbackDefaultProvider
{
    public override List<DisplayModeFallback> GetAll()
    {
        var original = base.GetAll();

        original.Add(new DisplayModeFallback
        {
            Name = "This is from code (1/12)",
            Tag = "one-12th-from-code",
            LargeScreenWidth = 12,
            MediumScreenWidth = 12,
            SmallScreenWidth = 12,
            ExtraSmallScreenWidth = 12
        });

        return original;
    }
}
```

## Adding Support for DDS Driven Storage

In order to get back DDS driven storage for display modes – you need to swap built-in provider with another build-in `EPiBootstrapArea.Providers.DisplayModeDdsFallbackProvider`.

```csharp
[InitializableModule]
[ModuleDependency(typeof(EPiServer.Web.InitializationModule))]
public class RegisterDisplayModesModule : IConfigurableModule
{
    public void ConfigureContainer(ServiceConfigurationContext context)
    {
        context.Container.Configure(ConfigureContainer);
    }

    private static void ConfigureContainer(ConfigurationExpression container)
    {
        container.For<IDisplayModeFallbackProvider>()
                 .Use<DisplayModeDdsFallbackProvider>();
    }

    public void Initialize(InitializationEngine context)
    {
    }

    public void Uninitialize(InitializationEngine context)
    {
    }

    public void Preload(string[] parameters)
    {
    }
}
```

Provider `DisplayModeDdsFallbackProvider` will only add modes that are not previously registered in DDS by comparing `Tag` properties for display modes.

If you need additional DisplayModes than default ones provided by the package – you can do that easily in overridden class:

```csharp
[InitializableModule]
[ModuleDependency(typeof(EPiServer.Web.InitializationModule))]
public class RegisterDisplayModesModule : IConfigurableModule
{
     private static void ConfigureContainer(ConfigurationExpression container)
    {
        container.For<IDisplayModeFallbackProvider>()
                 .Use<DisplayModeCustomDdsFallbackProvider>();
    }
}

public class DisplayModeCustomDdsFallbackProvider : DisplayModeDdsFallbackProvider
{
    protected override List<DisplayModeFallback> GetInitialData()
    {
        var original = base.GetInitialData();

        original.Add(new DisplayModeFallback
        {
            Name = "From code (1 / 12)",
            Tag = "from-code",
            LargeScreenWidth = 12,
            MediumScreenWidth = 12,
            SmallScreenWidth = 12,
            ExtraSmallScreenWidth = 12
        });

        return original;
    }
}
```

This should additionally add another display mode available for editor’s disposal.

![](/assets/img/2015/05/display-mode-from-code.png)

Thanks and happy coding!

[*eof*]
