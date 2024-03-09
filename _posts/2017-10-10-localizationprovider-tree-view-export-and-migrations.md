---
title: LocalizationProvider - Tree View, Export and Migrations
author: valdis
date: 2017-10-10 22:00:00 +0200
categories: [Add-On, .NET, C#, Localization Provider]
tags: [add-on, .net, c#, open source, localization]
---

It's been a while since last blog post about localization provider. It just means that I've been heads down busy with implementing some great features :)

Anyway - these are most probably last features before I'm switching over to .Net Core. I know that you might ask - why you need .Net Core if Episerver is still not there yet!? I'm planning to migrate to .Net Core because Episerver plugin is just small fraction of localization provider and actually you can use it outside of Episerver also - just in regular Asp.Net or Asp.Net Core website to localize standard stack.

These were original series over localization provider features.

* [Part 1: Resources and Models](https://tech-fellow.eu/2016/03/16/db-localization-provider-part-1-resources-and-models/)
* [Part 2: Configuration and Extensions](https://tech-fellow.eu/2016/04/21/db-localization-provider-part-2-configuration-and-extensions/)
* [Part 3: Import and Export](https://tech-fellow.eu/2017/02/22/localization-provider-import-and-export-merge/)
* **Part 4: Resource Refactoring and Migrations**

There has been [many other updates](https://tech-fellow.eu/tag/localization/) in between, but last part was planned to be just about migrations of resources, but turned out to include other features that you might find interesting.

Below is a list of some of the feature highlights from the latest version from [NuGet feed](https://www.nuget.org/packages?q=dblocalizationprovider) and [Episerver feed](http://nuget.episerver.com/en/?search=dblocalizationprovider&sort=MostDownloads&page=1&pageSize=10).

## XLIFF Export/Import

With some of the latest versions it's now possible to export and import translations using XLIFF format.

![](/assets/img/2017/10/2017-08-04_16-00-04.jpg)

Developers working on projects that involve 3rd party translation companies could find this valuable. As request to add XLIFF support came just from developer in that situation. XLIFF format is supported for both - export of resources and also import. In order to add support for XLIFF format you will need to install following package - [DbLocalizationProvider.AdminUI.EPiServer.Xliff](http://nuget.episerver.com/en/OtherPages/Package/?packageId=DbLocalizationProvider.AdminUI.EPiServer.Xliff) from Episerver nuget freed.

## Tree View

Sometimes (especially when you have huge list of resources and deep hierarchical relationships between classes, you might find it clumsy to work with resources as keys might get pretty long.

![](/assets/img/2017/10/2017-10-09_22-54-56.png)

Now you can switch to tree view and resources will be split into hierarchy by separator (usually `"."`) and represented to the editor in tree view way.

![](/assets/img/2017/10/2017-10-09_22-56-16.png)

It's also worth mention that if you have migrated old legacy resources to db localization provider and you have enabled `LegacyMode` then XPath style resources will collapse into tree view as well - making it more pleasant for the editor to work with.

First you need to enable legacy mode:

```csharp
[InitializableModule]
[ModuleDependency(typeof(InitializationModule))]
public class InitLocalization : IInitializableModule
{
    public void Initialize(InitializationEngine context)
    {
        ConfigurationContext.Setup(cfg =>
        {
            cfg.ModelMetadataProviders.EnableLegacyMode = () => true;
        });
    }

    public void Uninitialize(InitializationEngine context) { }
}
```

Then collapse will work correctly in Admin UI:

![](/assets/img/2017/10/2017-10-09_22-59-05-1.png)

## Migrations

Last but not least, there is now (FINALLY!) added support for refactoring scenarios for the resources (resource classes and localized models as well).
It's been hanging around for a while on GitHub and I never had chance to implemented it properly and I was always thinking about that problem deeper as it should be.. But I finally got question in Amsterdam about this tool - "what if I misspell resource class name, what about it property name that holds a translation is wrongly named at the beginning?" Meaning that if you just rename the resource container class - all the resources underneath will be treated as new set of resources - hence, loosing already added translation (if any). There are also scenarios when developers just want to shuffle around classes and move those in appropriate namespaces. Unfortunately - as database localization provider is heavily based on class structure and metadata found there - renames or class movement around to different namespace is disaster for the localization provider.

There were no support for these kind of refactoring tasks util now.

So it latest version there are bunch of new ways to tackle this complexity of migrations of the resources preserving translations for all added languages.

First, this code snippet might demo you basic support for the refactorings:

```csharp
using DbLocalizationProvider.Abstractions.Refactoring;

namespace DbLocalizationProvider.Tests.Refactoring
{
    [LocalizedModel]
    [RenamedResource("OldModelClass")]
    public class RenamedModelClass
    {
        public string NewProperty { get; set; }
    }
}
```

When provider will discover this resource newly generated key will be:

 * `DbLocalizationProvider.Abstractions.Refactoring.RenamedModelClass.NewProperty`

However, as resource class is decorated with `[RenamedResource]` attribute, there will be also another resource key discovered:

 * `DbLocalizationProvider.Abstractions.Refactoring.OldModelClass.NewProperty`

Meaning that latter will be used before synchronization. Provider will look for `..OldModelClass.NewProperty` resource in database and will update key to `..RenamedModelClass.NewProperty`.

And only then will proceed with standard resource synchronization.

Various kinds of refactoring scenarios are covered, starting from simple class or resource property name, ending with nested resource class property and class and namespace (for declaring type) renames. This tiny fragment from unit test suite should give you pretty clear picture of supported cases.

```csharp
namespace DbLocalizationProvider.Tests.Refactoring
{
    [LocalizedResource]
    [RenamedResource("OldParentContainerClassAndNamespace", OldNamespace = "In.Galaxy.Far.Far.Away")]
    public class RenamedParentContainerClassAndNamespace
    {
        [LocalizedResource]
        [RenamedResource("OldNestedResourceClass")]
        public class RenamedNestedResourceClass
        {
            public static string NewResourceKey => "New Resource Key";
        }

        [LocalizedResource]
        [RenamedResource("OldNestedResourceClass")]
        public class RenamedNestedResourceClassAndProperty
        {
            [RenamedResource("OldResourceKey")]
            public static string NewResourceKey => "New Resource Key";
        }
    }
}
```

This might come handy if you are dealing with requirements to rename resources or models and want to preserve existing translations.

**NB!** Once migration is done and executed successfully, it's recommended that you remove obsolete migrations as they might impact startup performance performing not necessary lookups in database trying to figure out which resource to rename and also because resource class looks ugly with all those attributes and ceremony.

<br/>
Happy localizing!

[*eof*]
