---
title: DbLocalizationProvider v8.0 Released
author: valdis
date: 2024-02-28 15:00:00 +0200
categories: [Add-On, Optimizely, Episerver, .NET, C#, Open Source, Localization, Localization Provider]
tags: [add-on, optimizely, episerver, .net, c#, open source, localization, localization provider]
---

I'm pleased to announce that Localization Provider v8.0 is finally out.

Again - took a bit longer than expected :)

## What's new?

* .NET8 set as default target
* Added provider model for external translation services
* Various bug fixes
* Some performance improvements
* `ConfigurationContext` now supports config configuration as well (you can change some settings after you have added and configured default settings for localization provider). This is very useful in unit test scenarios when you need to adjust some settings for specific test.
* Improved resource key comparison, gaining a bit of performance in hot paths
* Added pagination in Admin UI
* Security improvements (by default upgrading insecure connections)
* Dependencies upgrade

## Cloud Translator

If your editors are having timing issues translating the content it's now possible to add cloud based translators and leverage power of cloud.

Currently supported translators:

* Azure [Cognitive Services](https://learn.microsoft.com/en-us/azure/ai-services/translator/translator-overview)

First you will need to install Azure Cognitive AI integration package:

```
dotnet add package LocalizationProvider.Translator.Azure
```

To enable Microsoft Azure Cognitive Services as translator for Localization Provider follow these steps:

* Create Cognitive Services instance in Azure
* Copy **Access Key** and **Region**

![](/assets/img/2024/02/auto-translate-1.png)

* Add Azure Cognitive Services registration to your Localization Provider setup code:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddDbLocalizationProvider(x =>
    {
        x.UseAzureCognitiveServices("{access-key}", "{region}");

        // rest of the provider configuration...
    });
}
```

You should better read values from `appsettings.json` file or any other secure storage of your choice (like [Azure KeyVault](https://azure.microsoft.com/en-us/products/key-vault)).

After you have configured cloud translator successfully, editors will have easy and fast way to translate resources with a magic wand.

![](/assets/img/2024/02/auto-translate-2.png)

### Post Configuration

It is also possible to perform post configuration (after you have called `AddDbLocalizationProvider()`) of the localization provider.
This is useful when you are unit testing your web app, after Startup code is executed and you want to make sure that some post configuration settings are applied for your unit tests to execute correctly.

**NB!** Please note, that it is *not* possible to configure any types, scanners or anything else that is added to the DI container during `AddDbLocalizationProvider()` call. This limitation is due to fact that types are added to DI container and post configuration is called afterwards (when `IServiceProvider` is already built).

To post configure localization provider you have to follow standard [.NET Options](https://learn.microsoft.com/en-us/dotnet/core/extensions/options) pattern:

```csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        ...

        services.AddDbLocalizationProvider(cfg =>
        {
            ...
        });

        // post configuring provider
        services.Configure<ConfigurationContext>(ctx =>
        {
            ctx.EnableInvariantCultureFallback = false;
        });
    }
}
```

## Pagination in AdminUI

This feature should be turned on if you are experiencing performance issues in AdminUI (page load times, resource update times, etc).

To enable pagination - set `EnableDbSearch` to `true`.

```csharp
services.AddDbLocalizationProviderAdminUI(x =>
{
    // enable "server-side" pagination
    x.EnableDbSearch = true;

    // optionally you can set number of resources
    // to return when pagination is enabled
    x.PageSize = 50;

    // rest of the configuration...
});
```

When pagination is enabled, AdminUI by default will not load any resources. The only way to get resources in result - would be to execute search specifying any part of the resource key.

## Contributing

IF you liked the library and want to make it even better - consider contributing. The best place to start is to checkout out [the issue list](https://github.com/valdisiljuconoks/LocalizationProvider/issues) for the packages.

## Got Idea?

IF you got any idea how to improve  the library, or what features are you missing the most, consider leaving feedback on [the issue list](https://github.com/valdisiljuconoks/LocalizationProvider/issues).

As always - happy coding, happy localizing!

[*eof*]
