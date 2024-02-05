---
title: Playing with Fire - Localized EPiServer View Models
author: valdis
date: 2016-11-03 20:05:00 +0200
categories: [Add-On, .NET, C#, Localization Provider, Episerver, Optimizely]
tags: [add-on, .net, c#, open source, localization, episerver, optimizely]
---

Recently released driven localization provider for EPiServer had main focus on more complex view models along the other smaller bug fixes and features.

And specifically - there were few unsupported scenarios for the view models with base/parent class.

Here is the list of all new features in **v2.1**.

## Extract Abstractions

Until now, if I do need to decorate some of the resources that are located in some [core area assembly](https://tech-fellow.eu/2016/10/30/baking-round-shaped-software-mapping-to-the-code/) - you need to reference `DbLocalizationProvider` library that had some undesired dependencies (for instance, why do I need to reference `Owin` assembly in my core area project). Now `DbLocalizationProvider.Abstractions` is extracted and it contains only bare minimum for the resource attribution and controlling resource discovery and naming conventions. So if you need just to mark resources somewhere in inner circle projects - you can just reference `DbLocalizationProvider.Abstractions` [package](https://www.nuget.org/packages/LocalizationProvider.Abstractions/).

## Register Only Included Resources

Sometimes when you do your own class that has a lot of properties, or class that inherits from some other class with lots of properties, and you need to register just a few of the resources.

Let's look at sample. Image you have following class:

```csharp
namespace Sample
{
    [LocalizedModel]
    public class MyPage : PageData
    {
        public string SomeProperty { get; set; } = "Default translation";
    }
}
```

What will happen is - along the `Sample.MyPage.SomeProperty` registration, **all** `PageData` public properties will be discovered and registered as well. This might not be desired - will be huge noise and bunch of unnecessary properties (well, maybe sometimes inherited `PageData` properties needs to be translated). And as you don't have access to source code of the `PageData` type - you can't really set `[Ignore]` attributes there either.

Anyway - there is a solution for this problem. You can ask resource scanner to include only those resources that are marked explicitly with `[Include]` attribute. Here is code:

```csharp
namespace Sample
{
    [LocalizedModel(OnlyIncluded = true)]
    public class MyPage : PageData
    {
        [Include]
        public string SomeProperty { get; set; } = "Default translation";

        public string ThisWillBeIgnored { get; set; } = "Whatever";
    }
}
```

Only `Sample.MyPage.SomeProperty` resource will be registered from this type.

## Register Only "My" Resources

### Background for Feature

This feature will make sure that only translation resources on current `Type` will be registered. It's supported in "ordinary" case and also with "generics". Read on.

Motivation behind this feature was case when there was base viewmodel class with lot of properties that are registered as resources and also at the same time lot child viewmodels. As result with existing set of features was fact that resources from base class was registered as many times as there were child classes. Basically - every time scanner will discover child class - all parent/base properties were also registered. But within that child class "context".

For example:

```csharp
[LocalizedModel]
public class BaseViewModel
{
    public string CustomMessage { get; } = "Some value";
}

[LocalizedModel]
public class HomeViewModel : BaseViewModel
{
    [Display(Name = "Also your email")]
    public string Username { get; set; }
}

[LocalizedModel]
public class ArticleViewModel : BaseViewModel
{
    ...
}
```

Following resources will be registered:

```
HomeViewModel.CustomMessage
HomeViewModel.Username
ArticleViewModel.CustomMessage
```

Sometimes you just want to "freeze" property definition container type and only register base class resources once.


This feature does exactly that.

### For Non-Generic Models

To use this feature you need to set attribute property `Inherited` to `false` - `[LocalizedModel(Inherited = false)]`. This will instruct scanner to preserve property definition container type.

```csharp
[LocalizedModel]
public class BaseViewModel
{
    [Display(Name = "This is message")]
    public string Message { get; set; }

    public string CustomMessage { get; } = "Some value";
}

[LocalizedModel(Inherited = false)]
public class HomeViewModel : BaseViewModel
{
    [Display(Name = "Also your email")]
    public string Username { get; set; }
}
```

Following resources will be registered this time:

```
BaseViewModel.Message
BaseViewModel.CustomMessage
HomeViewModel.Username
```

So when you will try to use this new property somewhere during "runtime":

```razor
@model HomeViewModel

@Html.TranslateFor(() => m.CustomMessage)
```

Localization provider will look for resource with key `BaseViewModel.CustomMessage` and not for the key `HomeViewModel.CustomMessage` (even if `TranslateFor()` context is current view model - `HomeViewModel`). This is because while scanning and registering resources - provider discovered that `HomeViewModel` does not want to register inherited resources from the parent class(-es).

By applying this feature you prevent pollution of the resources.

### Playing with Fire - Generic Models

The tricky part starts when viewmodels are defined as generic types. And this is the time when database localization provider starts playing with fire :)
For instance, let's have following view model defined:

```csharp
namespace Sample
{
    [LocalizedModel]
    public class BaseOpenViewModel<T>
    {
        [Display]
        [Required]
        public string BaseProperty { get; set; }
    }

    [LocalizedModel(Inherited = false)]
    public class SampleViewModelWithClosedBase : BaseOpenViewModel<SomeType>
    {
        public string ChildProperty { get; set; }
    }
}
```

By default following resource keys will be discovered:


```
Sample.BaseOpenViewModel`1.BaseProperty
Sample.BaseOpenViewModel`1.BaseProperty-Required
Sample.SampleViewModelWithClosedBase.ChildProperty
```

Now image that we are asking for a label for the parent class property:

```razor
@model SampleViewModelWithClosedBase

@Html.LabelFor(m => m.BaseProperty)
```

Method call `LabelFor()` will go through Asp.Net Mvc model meta data provider. Container type for the requested property will be `SampleViewModelWithClosedBase`, well because that's the model of the view.
Metadata provider will recognize that container type has attribute with `Inherited` property set to `false`. Which essentially means that if property is not found on the given container type "level", property definition should be searched within upper levels - through the inheritance chain up to the very base type - `System.Object`.

At the runtime while metadata provider tries to find parent type where property is defined, in this case parent will be `BaseOpenViewModel<SomeType>` and not `BaseOpenViewModel<T>` as it was discovered during scanning process.

In other words: parent class is **open generic** during scanning, but **closed generic** when resource translation is requested afterwards. Interesting - but at the same time pretty simple to resolve. We need to ignore type parameter and look only for actual type definition. Fortunately this type information in available from .Net Framework. We just need to consume it and act accordingly.


This code fragment will look for resource with key `Sample.BaseOpenViewModel.BaseProperty` despite that model of the view is `SampleViewModelWithClosedBase`:

```razor
@model SampleViewModelWithClosedBase

@Html.LabelFor(m => m.BaseProperty)
```


## Log Missing Keys

Many thanks to my friend [Petter Sørby](http://world.episerver.com/blogs/sorby/) from BVN/EPiServer for a great idea and copyright of the feature.

I just couldn't image better place to add diagnostics.

If you want to see which keys are missing (it's unlikely that you will hit this problem if you are following "*strongly-typed*" approach and avoid "*stringly-typed*"), then you just need to enable diagnostics for database localization provider and then grep your EPiServer log files. You can enable diagnostics my adding following line somewhere in your initialization modules:

```csharp
using DbLocalizationProvider;

...

ConfigurationContext.Setup(cfg =>
                          {
                              cfg.DiagnosticsEnabled = true;
                          });
```

Also maybe some other stuff will be added to be logged under this setting.

## EPiServer v10

Support for EPiServer v10 is coming soon! Pretty close.

Happy localizing!

[*eof*]
