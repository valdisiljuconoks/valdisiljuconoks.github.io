---
title: Some smaller updates for DbLocalizationProvider
author: valdis
date: 2018-01-31 23:30:00 +0200
categories: [Add-On, .NET, C#, Localization Provider]
tags: [add-on, .net, c#, open source, localization]
---

Localization provider is well and alive. Lately I've been heads down busy with porting some of the parts over to [.Net Core](https://tech-fellow.eu/2018/01/27/dblocalizationprovider-in-asp-net-core/) targeting .Net Standard version 2.0. Anyway just wanted to let you know some of the smaller updates and fixes for the latest version of the provider packages.


## Multi-Target -> NetFx and Core

Now there are specific multi targets for .Net Framework and .Net Core for `` and `` packages. This was required due to reported issues when package was used context together with WebApi (to be more specific - there was weird issues when referencing `System.Net.Http` from netstandard in framework targeted project). It's quite lengthy to explain here. Maybe another post would make it more clear.

## ResourceKey Attribute with ViewModels

Now you can use `[ResourceKey]` attribute also with view models. This addresses some of the use cases when there is defined convention for the resource keys or there are already migrated resources from (well, guess!?!) XML files with predefined structure.
If you do have following view model:

```csharp
[LocalizedModel]
public class ModelWithDataAnnotationsAndResourceKey
{
    [ResourceKey("the-key")]
    [Display(Name = "Something")]
    [Required (ErrorMessage = "This unfortunately is required")]
    public string UserName { get; set; }
}
```

then now library will respect `[ResourceKey]` attribute and will generate for instance following resources:

* `the-key` (`[Display]` value will be used here as translation)
* `the-key-Required` - `ErrorMessage` value will be used as translation

## JavaScript Resource Handler

JavaScript Resource Handler now correctly handles deep object cloning. What it means? Previously issue was with multiple client-side resource includes:

```razor
<head>
    @Html.GetTranslations(typeof(ResourceClass))
    @Html.GetTranslations(typeof(AnotherResource))
</head>
```

With this code it would result in "last wins" situation - meaning that under `window.jsl10n` key would be **only** last resource class - `AnotherResource`.

This has been fixed in latest version and all of the client-side resources are merged into single object:

![](/assets/img/2018/01/2018-01-20_00-45-04.jpg)

Note, that there `...ResouceWithSwedish..` but returns English text? This is just debug sample code for invariant fallback scenario :)

Also now `JsResourceHandler` will include also fallback for invariant culture (if it will be enabled via `ConfigurationContext`).

## Editor UI

Now you are able to hide either `Table` or `Tree` view if you need to.

```csharp
[InitializableModule]
[ModuleDependency(typeof(InitializationModule))]
public class InitLocalizationModule : IInitializableModule
{
    public void Initialize(InitializationEngine context)
    {
        UiConfigurationContext.Setup(_ =>
        {
            ...
            _.DisableView(ResourceListView.Table);
        });
    }

   ...
```

Also `New Resource` command has been removed from the UI. Now you would not be able to create new resources from UI. All discovery and synchronization of resources should happen via code.

![](/assets/img/2018/01/2018-01-31_23-42-16.png)

<br/>
Happy localizing!
[*eof*]
