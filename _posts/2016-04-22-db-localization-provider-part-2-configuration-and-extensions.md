---
title: DbLocalizationProvider - Part 2: Configuration and Extensions
author: valdis
date: 2016-04-22 20:05:00 +0200
categories: [Add-On, .NET, C#, Localization Provider, Episerver, Optimizely]
tags: [add-on, .net, c#, open source, localization, episerver, optimizely]
---

In this blog post we are going through some of the configuration options available for DbLocalizationProvider for EPiServer. You might be wondering what can and should be configurable for localization provider in the context of the EPiServer, but there are few things.

## Configuration Setup for Libraries

Developing libraries as an author you always need to keep in mind your consuming projects and usage scenarios, need to think about how your library will be setup, used and extended if needed.

I really liked [Mark Seeman's blog posts](http://blog.ploeh.dk/2014/05/19/di-friendly-library/) about developing libraries and frameworks and what you should keep in mind as an author. Libraries are something that might or might not be used in target project and adding a package of the library shouldn't affect projects behavior. Consuming project's codebase should be able to *use* library *if*, *when* and *how* needed.


## Configuring DbLocalizationProvider

By default if you install the package, some of the things have been setup for you already, however - you have a possibility to change that.

As I wrote in [blog post](https://tech-fellow.eu/2016/02/22/create-your-library-configuration-api-for-episerver/) about creating configuration API for you library, expected approach for EPiServer to setup library would be through initializable module interface:

```csharp
...
using DbLocalizationProvider;

[InitializableModule]
public class InitializationModule1 : IInitializableModule
{
    public void Initialize(InitializationEngine context)
    {
        ConfigurationContext.Setup(ctx => ... );
    }

    public void Uninitialize(InitializationEngine context) { }
}
```

Type `ConfigurationContext` is coming from `DbLocalizationProvider` and is used as settings container to configure and setup localization provider. Following setup properties are available:

* `DiscoverAndRegisterResources` - this setting will control whether library does localized models and resource lookup and discovery during application startup. Sometimes you just don't want to spend time on discovery process (it might come handy in cases when you know that there will be no new resources or models added to the codebase). Scanning and registration process is enabled by default;

* `DefaultResourceCulture` - while scanning the resources or models, default value will be written to the database and initial translation for particular resource. If default culture is not set, by default library will try to use `ContentLanguage.PreferredCulture`, if that fails - `English` will be used as initial translation culture:

```csharp
ConfigurationContext.Current.DefaultResourceCulture != null
    ? ConfigurationContext.Current.DefaultResourceCulture.Name
    : (ContentLanguage.PreferredCulture != null ? ContentLanguage.PreferredCulture.Name : "en");
```

* `PopulateCacheOnStartup` - after scanning, discovery and registration process has ended (if one was enabled), this setting will control whether cache will be populated during the startup. Enabled by default. If cache pre-population is disabled, library will "eventually" fill up cache with resources "on-demand" when somebody will be asking for particular resource, one will be added to the cache afterwards;

* `ReplaceModelMetadataProviders` - one of the cool features of `DbLocalizationProvider` library is that it also "plugs in" in Asp.Net Model Metadata pipeline and can replace default model's metadata providers to fetch localization resources necessary for things like `Html.LabelFor(m => m.Username)`. This configuration setting allows you to disable this replacement and continue with whatever providers you might have in your project. This setting is enabled by default.

* `UseCachedModelMetadataProviders` - cached model metadata provider gives you performance by caching model metadata and avoiding scanning of the attributes and building metadata every time somebody needs to generate label (e.g. `Html.LabelFor(m => m.Username)`) or any other metadata dependent element. **NB!** However there is a catch using cached model metadata provider - it's not aware of multi-language cases. Meaning that - once model metadata is cached - it is not changed anymore (cache is not invalidated). Which also means, if you have multi-language project and user changes UI language, Asp.Net Mvc model metadata provider pipeline will not call metadata provider once again to get new `DisplayName` for the model property - because it's already cached. So **recommendation** here is - if you have only single language, you can enable cached model metadata provider which will give you some performance boost, but otherwise - in multi-language projects - this should not be enabled. This configuration setting is disabled by default.

* `EnableLegacyMode` - legacy mode configuration setting is used in cases when you migrated from EPiServer XPath based resources to DbLocalizationProvider using MigrationTool.

For example, when you had some sort of hacked model metadata annotations (there are [few ways](http://world.episerver.com/blogs/devabees/Dates/2014/3/Integrating-LocalizationService-with-MVC-DataAnnotations/) to achieve this) using EPiServer's XPath based access to resources:

```csharp
public class MyViewModel
{
    [LocalizedDisplayName("/path/to/langauge/resource")]
    public string Username { get; set; }
}
```

and you used DbLocalizationProvider MigrationTool to generate resources in database. As you might add `DbLocalizationProvider` library to existing project, there could be lot of resources that would need refactoring.
However after migrating resources from XML files to database, resource keys still remain in EPiServer XPath notation.

Your new model might look like this:

```csharp
public class MyViewModel
{
    [Display(Name = "/path/to/language/resource")]
    public string Username { get; set; }
}
```

Setting `EnableLegacyMode` configuration key to `true` this will ensure that `DbLocalizationProvider` library will also try to find resource translation using EPiServer's XPath notation that is used in the attribute. Meaning that you don't need to move translation from XPath resource to *actual* resource (e.g. `LocalizationSample.Models.MyViewModel.Username`). This setting is disabled by default.

* `EnableLocalization` - this configuration setting is lambda expression. If lambda expression returns `true`, actual translation for requested language is returned, otherwise - resource key is return.
This configuration setting is used in cases when editor needs to understand what kind of resource is used in particular location on the page or anywhere else.

You might wondering why this setting is `LambdaExpression` or `Func<bool>` to be more precise?!
`Expression` type setting key will give possibility to enable or disable localization during runtime if that's needed.

One of the possibility how to configure this setting would be to use for instance `FeatureSwitch` library to provide runtime switching possibility for editors. In this case, setting setup might look like this:

```csharp
[ModuleDependency(typeof (InitializationModule))]
public class LocalizationInitialization : IInitializableModule
{
    public void Initialize(InitializationEngine context)
    {
        ConfigurationContext.Setup(ctx =>
        {
            ctx.EnableLocalization = CheckLocalization;
        });
    }

    public void Uninitialize(InitializationEngine context) { }

    private bool CheckLocalization()
    {
        return !FeatureContext.IsEnabled<DisableLocalization>();
    }
}

[QueryString(Key = "localization-disabled")]
public class DisableLocalization : BaseFeature { }
```
Here is a small demo video:

<iframe src="https://player.vimeo.com/video/163752727" width="800" height="520" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe>

## More info

Other posts in this series:

* [Part 1: Resources and Models](https://tech-fellow.eu/2016/03/16/db-localization-provider-part-1-resources-and-models/)
* **Part 2: Configuration and Extensions**
* [Part 3: Import and Export](https://tech-fellow.eu/2017/02/22/localization-provider-import-and-export-merge/)
* [Part 4: Resource Refactoring and Migrations](https://tech-fellow.eu/2017/10/10/localizationprovider-tree-view-export-and-migrations/)

If you have any ideas, thoughts or complaints, please leave them in [GitHub repo](https://github.com/valdisiljuconoks/LocalizationProvider/) or in comments section below.

Happy localizing!

[*eof*]
