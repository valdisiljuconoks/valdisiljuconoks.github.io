---
title: Optimizely DeveloperTools are back!
author: valdis
date: 2022-12-13 12:00:00 +0200
categories: [Add-On, Optimizely, Episerver, .NET, C#, Open Source]
tags: [Add-On, Optimizely, Episerver, .NET, C#, Open Source]
---

Experimental project with handy tools for developers is back for Optimizely v12.

# DISCLAIMER!
Remember, use at your own risk - this is **not** a supported product!

## Current Features

* View contents of the Dependency Injection container
* View Content Type sync state between Code and DB
* View rendering templates for content types
* View ASP.NET routes
* View loaded assemblies in AppDomain
* View startup time for initialization modules
* View remote event statistics, provider and servers
* View all registered view engines
* View local object cache content (with option to remove items)
* View initialization module dependencies as graph

## Getting Started

To get started with Optimizely developer tools - all you need to do is to add it to your project and use it :)

```csharp
public void ConfigureServices(IServiceCollection services)
{
    ...
    services.AddOptimizelyDeveloperTools();
}

public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    ...
    app.UseOptimizelyDeveloperTools();
}
```

When installed and configured - you must be part of the "Administrators" group to access the tool. New menu "Developer Tools" should appear in the top menu.

![](/assets/img/2022/12/opti-devtools.png)


## How Risky it is to install on production?
You can read more in-depth analysis of toolset and it's side-effects [here](https://tech-fellow.ghost.io/2019/02/14/how-risky-are-episerver-developertools-on-production-environment/).

## Try It Out!
Download the latest build on [NuGet](https://nuget.optimizely.com/package/?id=EPiServer.DeveloperTools) or [under releases](https://github.com/episerver/DeveloperTools/releases)


<br/>

Happy tooling!

[*eof*]
