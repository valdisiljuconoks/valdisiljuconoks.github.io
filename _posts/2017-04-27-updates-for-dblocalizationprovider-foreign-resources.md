---
title: Updates for DbLocalizationProvider - Foreign Resources
author: valdis
date: 2017-04-27 15:45:00 +0200
categories: [Add-On, .NET, C#, Localization Provider, Episerver, Optimizely]
tags: [add-on, .net, c#, open source, localization, episerver, optimizely]
---

Blog post about latest updates for database localization provider project. Release mainly focuses on foreign affairs - foreign, hidden and referenced resources. Read further for more info.

## Foreign Resources

Good [friend and colleague](http://marisks.net/) of mine asked once: "How do I register localized resources for [ValidationIssue](http://world.episerver.com/documentation/Items/Developers-Guide/Episerver-Commerce/9/Orders/order-processing/) enum from EPiServer Commerce assembly?" Short answer back then was - you can't.
Until now.

With latest database localization provider now you can tell provider to get familiar with so called foreign resources and include those in sync process as well - even they are not decorated with `[LocalizedResource]` attribute. Actually attribute is only needed for provider to scan through and understand which types to include in sync process and which not.

Let's assume for some reason you need to localize foreign resource (`EPiServer.Core.VersionStatus`) for what you don't have source code and you cannot decorate type with required attributes. You will need to add similar code to register foreign resources:

```csharp
namespace DbLocalizationProvider.Sample
{
    [InitializableModule]
    [ModuleDependency(typeof(InitializationModule))]
    public class InitLocalization : IInitializableModule
    {
        public void Initialize(InitializationEngine context)
        {
            ConfigurationContext.Setup(cfg =>
            {
                ...

                cfg.ForeignResources.Add(typeof(VersionStatus));
            });
        }

        public void Uninitialize(InitializationEngine context) { }
    }
}
```

Using `ForeignResources` collection you are telling provider to include those types in resource sync process.
There are couple of extension methods as well for your convenience helping to add types to the collection.

Still you are able to use newly registered foreign resource in the same way as the rest of resources:

```csharp
[LocalizedModel]
public class MyCustomViewModel
{
    public VersionStatus Status { get; set; }
}



@model MyCustomViewModel

...razor
@Html.TranslateFor(m => m.Status)
```

Resource key naming conventions are the same as for ordinary discovered resources via `[LocalizedResource]` attribute.

## Hidden Resources

Sometimes you need to have some resources localizable but not visible by default in administration user interface.
No additional permissions are applied for hidden resources - they are just hidden by default from the UI.
If you need hidden & **restricted resources** - please ping me, will see what we can do about that.

To register hidden resource - you just apply `[Hidden]` attribute to particular resource.

You can apply attribute to specific member (only that resource will be hidden from UI):

```csharp
[LocalizedModel]
public class SomeModelWithHiddenProperty
{
    [Hidden]
    public string SomeProperty { get; set; }
}

[LocalizedResource]
public enum SomeEnumWithHiddenResources
{
    None,
    [Hidden] Some,
    Another
}
```

Or you can actually apply to whole class as well (then all discovered resources from that type will be hidden):

```csharp
[LocalizedModel]
[Hidden]
public class SomeModelWithHiddenPropertyOnClassLevel
{
    public string SomeProperty { get; set; }
}

[LocalizedResource]
[Hidden]
public enum SomeEnumWithAllHiddenResources
{
    None,
    Some,
    Another
}
```

Translation still works even for hidden resources.
Also, `[Hidden]` attribute can be applied to existing resources - metadata in the database will be updated to match the source code.

## Referenced Resources

Sometimes in your model you need to reference resource from the other - usually common shared resources.

Meaning that there are models with properties that are common for the project (for instance - `Yes`, `No`, `First name`, etc.). If you will just annotate view model with `[LocalizedModel]` and will have multiple models with property `FirstName` - by default all properties from all models will become localizable resources.
This might pollute your resource list with unnecessary items you would want to avoid.

Now you can "reference" different resource and use that instead of annotated resource.

For example, you have some common resources:

```csharp
namespace DbLocalizationProvider.Sample
{
    [LocalizedResource]
    public class CommonResources
    {
        public static string CommonProp => "Common Value";
    }
}
```

Using new attribute you can decorate your model's property with `[UseResource]` attribute (you can also just reference resource using string - `"CommonProp"`, I prefer `nameof`):


```csharp
namespace DbLocalizationProvider.Sample
{
    [LocalizedModel]
    public class ModelWithOtherResourceUsage
    {
        [UseResource(typeof(CommonResources),
                     nameof(CommonResources.CommonProp))]
        public string SomeProperty { get; set; }
    }
}
```

So, now if you will try to localize `SomeProperty` for this view model - common resource will be returned:

```razor
@model ModelWithOtherResourceUsage

...
@Html.TranslateFor(m => m.SomeProperty)
```

Instead of `ModelWithOtherResourceUsage.SomeProperty` resource, `CommonResources.CommonProp` key will be used.
There will be no resource for key `ModelWithOtherResourceUsage.SomeProperty` registered in database.
Whenever you will change translation for `CommonResources.CommonProp` resource - it will be used everywhere you referenced it using `[UseResource]` attribute.

## Where to get?

As usual - new version is available on [nuget.org](https://www.nuget.org/packages?q=dblocalization) and [EPiServer feeds](http://nuget.episerver.com/en/?search=dblocalization&sort=MostDownloads&page=1&pageSize=10).

<br/>
Happy localizing!<br/>
[*eof*]
