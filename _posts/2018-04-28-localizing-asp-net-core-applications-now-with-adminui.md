---
title: Localizing Asp.Net Core Applications now with AdminUI
author: valdis
date: 2018-04-28 00:30:00 +0200
categories: [Add-On, .NET, C#, Localization Provider]
tags: [add-on, .net, c#, open source, localization]
---

Finally it's time to release administrative user interface for complementing Asp.Net Core application localizing process.

It's been a while since localization provider has been released for Asp.Net Core applications, but until now - actual translation of the resources was limited due to absence of the user interface through which editor can make adjustments to the translations.

I'm happy to announce that AdminUI has been rewritten to utilize Vue.js framework and to grind some of the rough edges of the library.

## Installation

The only thing you need to do to get started is to install following package.

```
PM> Install-Package LocalizationProvider.AdminUI.AspNetCore
```

It will also bring down all the other necessary packages for library to work correctly.

## Setup & Customization

Essentially there are 2 parts of the whole setup process:

* Configure Services
* Configure Library

Configuration of the services is part of the Asp.Net Core dependency injection process setup. So to add AdminUI to you application you need (in `Startup.cs`):

### Enable Built-In Localization Support

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddLocalization();
    services.AddMvc()
        .AddViewLocalization()
        .AddDataAnnotationsLocalization();

    // just adding English and Norwegian support
    services.Configure<RequestLocalizationOptions>(opts =>
    {
        var supportedCultures = new List<CultureInfo>
                                {
                                    new CultureInfo("en"),
                                    new CultureInfo("no")
                                };

        opts.DefaultRequestCulture = new RequestCulture("en");
        opts.SupportedCultures = supportedCultures;
        opts.SupportedUICultures = supportedCultures;
    });
}

public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    var options = app.ApplicationServices.GetService<IOptions<RequestLocalizationOptions>>();
    app.UseRequestLocalization(options.Value);

}
```

### Setup Library

And when built-in support is configured, you can now add support for DbLocalizationProvider library (again in `Startup.cs`):

```csharp
public void ConfigureServices(IServiceCollection services)
{
   services.AddDbLocalizationProvider(_ =>
   {
       ...
   });

   services.AddDbLocalizationProviderAdminUI(_ =>
   {
       ...
   });
}
```

Through these methods you can customize behavior for the library and AdminUI component.

And when you are done with customization, you need to add those to the application:

```csharp
public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
   app.UseDbLocalizationProvider();
   app.UseDbLocalizationProviderAdminUI();
}
```

## Accessing UI

When everything is setup correctly and Asp.Net Core runtime does not blame you for incorrect configuration, you may access AdminUI via `.../localization-admin` url (by default).

![aspnetcore-admin-ui](/assets/img/2018/04/aspnetcore-admin-ui.jpg)

## Get More Info

AdminUI itself will be developed further and some of the cool features (like import with merge preview) from the other target platform will be pulled over to Asp.Net Core component.

As usual - to find more info and to file any issue - look for [Github repo](https://github.com/valdisiljuconoks/localization-provider-core).

<br/>
Happy localizing!
