---
title: Asp.Net Mvc Areas - Packed as NuGet
author: valdis
date: 2015-08-15 08:30:00 +0200
categories: [.NET, C#, Episerver, Optimizely]
tags: [.net, c#, episerver, optimizely]
---

If you would want to add Asp.Net Areas support to your EPiServer site, you would need to copy some files from my [previous blog post](https://tech-fellow.eu/2015/01/21/full-support-for-asp-net-mvc-areas-in-episerver-7-5/). Copying files from someone's blog posts seems to be good idea, but it's a bit problematic in case of updates or changes. If so, then we share common vision that by installing NuGet package, your project gets up-to-date support for Asp.Net Mvc Areas.

![](/assets/img/2015/08/nuget-2.png)

For that reason I packed up Asp.Net Mvc Areas support into NuGet package, for you guys. You may just need to install it and configure a bit if needed, to get things up and running.

## Getting Started

1) Install NuGet package:

```
PM> Install-Package MvcAreasForEPiServer
```

2) Register Mvc Areas. You have plenty of places to register this code fragment (`Application_Start`, `Global.RegisterRoutes`, `InitializableModule`, etc).

```csharp
AreaConfiguration.RegisterAllAreas();
```

By default configuration setting to detect Mvc Area by controller (`EnableAreaDetectionByController`) will be enabled.

## Configuration

If you want to disable Mvc Area detection by controller, you can choose configuration settings to customize this behavior during areas registration procedure.

```csharp
[InitializableModule]
[ModuleDependency(typeof(EPiServer.Web.InitializationModule))]
public class ConfigureAreasSupportModule : IInitializableModule
{
    public void Initialize(InitializationEngine context)
    {
        AreaConfiguration.RegisterAllAreas(config =>
        {
            config.EnableAreaDetectionByController = true;
            config.EnableAreaDetectionBySite = true;
        });
    }

    public void Uninitialize(InitializationEngine context)
    {
    }
}
```

There are two configuration settings:

* `EnableAreaDetectionByController`: this setting will make sure that Mvc Area is detected while executing controller for particular content type in that area;
* `EnableAreaDetectionBySite`: this setting will make sure that Mvc Area is detected when accessing EPiServer site that is also defined as Mvc Area;

More info about **general insights** of Mvc Area support for EPiServer sites: [here](http://tech-fellow.eu/2015/01/22/full-support-asp-net-mvc-areas-episerver-7-5/)
More info about using Mvc Areas as EPiServer's **site discriminators**: [here](https://tech-fellow.eu/2015/08/10/asp-net-mvc-areas-in-episerver-part-2/)


Happy slicing and dicing into areas!

[*eof*]
