---
title: FeatureSwitch library 1.0 – Released!
author: valdis
date: 2014-03-28 23:40:00 +0200
categories: [.NET, C#, Feature Switch]
tags: [.net, c#, feature switch]
---

Have you ever wrote the code like following to verify that either you have to disable or enable some functionality base on set of conditions (usually different sources):

```csharp
if(ConfigurationManager.AppSettings["MyKey"] == "true")
{
    // feature is enabled - do something
    ...;
} else {
    // feature is disabled - do something
    ...;
}
```

or something like this:

```csharp
if(HttpContext.Current != null
   && HttpContext.Current.Session != null
   && HttpContext.Current.Session["MyKey"] == "true")
{
    return ...;
}
```

`FeatureSwitch` library should reduce amount of time and code needed to implement feature toggle in unified way. `FeatureSwitch` library is easily adoptable and extendable.

## Overview

`FeatureSwitch` library is based on two basic aspects [features](https://github.com/valdisiljuconoks/FeatureSwitch/wiki#features) and [strategies](https://github.com/valdisiljuconoks/FeatureSwitch/wiki#strategies). In combination they provide enough input data for feature set builder to construct feature context to be used later to check whether particular feature is enabled or disabled.
There are few integration and helper libraries as well around `FeatureSwitch` core lib:

* [Mvc Control Panel](https://github.com/valdisiljuconoks/FeatureSwitch/wiki/Asp.Net-MVC-Integration) – `FeatureSwitch` control panel where you can visually see state of the each feature and change it
* [Web Optimization](https://github.com/valdisiljuconoks/FeatureSwitch/wiki/Web-Optimization-Helpers) – Helpers for styles and scripts conditional bundling and minification
* [EPiServer integration](https://github.com/valdisiljuconoks/FeatureSwitch/wiki/EPiServer-Integration) – Module to integrate into EPiServer CMS platform for easier access to UI Control Panel

## Where do I get packages?

All packages are available on NuGet feeds:

* [Core library](https://www.nuget.org/packages/FeatureSwitch/)
* [Asp.Net Mvc Integration](https://www.nuget.org/packages/FeatureSwitch.AspNet.Mvc/)
* [Asp.Net Mvc 5 Integration / Owin](https://www.nuget.org/packages/FeatureSwitch.AspNet.Mvc5/)
* [Web Optimization pack](https://www.nuget.org/packages/FeatureSwitch.Web.Optimization/)
* [EPiServer integration](http://nuget.episerver.com/en/OtherPages/Package/?packageId=FeatureSwitch.EPiServer)

## Features

Feature is main aspect of `FeatureSwitch` library it’s your logical feature representation – either it’s enabled or disabled. You will need to define your features to work with `FeatureSwitch` library.

To define your feature you have to create class that inherits from `FeatureSwitch.BaseFeature` class:

```csharp
public class MySampleFeature : FeatureSwitch.BaseFeature
{
}
```

Keep in mind that you will need to add at least one strategy for the feature in order to enable it.

By default every feature is **disabled**!

## Setup (FeatureSet builder)

To get started with `FeatureSwitch` you need to kick-off [`FeatureSetBuilder`](https://github.com/valdisiljuconoks/FeatureSwitch/blob/master/FeatureSwitch/FeatureSetBuilder.cs) by calling `Build()` instance method:

```csharp
var builder = new FeatureSetBuilder();
builder.Build();
```

By calling `Build()` method you are triggering auto-discovery of features in current application domain loaded assemblies. Auto-discovery will look for classes inheriting from `FeatureSwitch.BaseFeature` class. Those are assumed to be features.

## Check if feature is enabled?

After features have been discovered and set has been built you are able to check whether feature is enabled or not:

```csharp
var isEnabled = FeatureContext.IsEnabled();
```

You can also use some of the `IsEnabled` overloads for other usages.

## Strategies

Whereas strategies are aspect in the library that is controlling either feature is enabled or disabled in the current circumstances. Strategy is an attribute you decorate your feature with:

```csharp
[FeatureSwitch.Strategies.AppSettings(Key = "MySampleFeatureKey")]
public class MySampleFeature : BaseFeature
{
}
```

Currently FeatureSwitch release contains following built-in strategies:

* `AlwaysFalse` – by using this strategy your feature will be always disabled
* `AlwaysTrue` – this strategy will always make your feature shine
* `AppSettings` – key under `<appSettings>` element either in `web.config` or `app.config` will control this feature state
* `HttpSession` – if you need more permanent storage for your feature’s state you can make use of Http session
* `QueryString` – if you will provide magic key in query string, feature may enable or disable.

### Multiple Strategies

Single feature can support more that one strategy. For instance:

```csharp
[AppSettings(Key = "StyleOptimizationDisabled")]
[QueryString(Key = "DisableStyleOptimization", Order = 1)]
public class StyleOptimizationDisabled : BaseFeature
{
}
```

This feature is controlled by `StyleOptimizationDisabled` element key in `web.config` file and then also by the query string passed in during request. It means that feature is disabled by `AppSettings` strategy but could be enabled by `QueryString` strategy (providing proper query string parameter `?DisableStyleOptimization=true`) – feature is eventually enabled.

In order to attach multiple strategies to single feature you need to define order for the strategy using `Order` property. `FetureSetBuilder` will fail if it founds multiple strategies with undefined or equal order.

When feature is asked for enabled state all strategies in ascending order are used to query for feature enabled state. If any of those return `true` feature is enabled.

### Writable Strategies

Sometimes it’s required to store feature enabled state in more permanent storage (session, cache, etc). For this reason you can use any of `FeatureSwitch.Strategies.BaseStrategyImpl` implementations.

Currently available writable strategies:

* `HttpSession` – using this strategy you can enable or disable feature and it will remain in this state as long as session is alive

## Advanced FeatureSwitch Context Constructions

At moment of current release of FeatureSwitch library we do have dependency on [StructureMap](https://www.nuget.org/packages/structuremap/) in order to keep track of strategies and dependencies and also to support [constructor injection](http://en.wikipedia.org/wiki/Dependency_injection) for custom strategies if needed (this is subject for refactoring).

Using dependency injection you are able to instruct `FeatureSetBuilder` how to construct context and which strategies to use and how to initialize them.

## Disable Auto-Discovery

Sometimes it’s required to disable auto-discovery and use only predefined list of features.
This can be accomplished using `action` parameter of `Build()` method:

```csharp
var builder = new FeatureSetBuilder();
builder.Build(ctx => ctx.AddFeature<MySampleFeature>());
```

This will add only `MySampleFeature` to the context and will not perform auto-discovery of features.

## Strategy Swap

Let’s assume that for any reason you are required to change / swap built-in strategy with some other implementation. This can be used by using configuration expression while building feature set (`MySampleFeature` is configured to use `AppSettings` strategy):


```csharp
var builder = new FeatureSetBuilder();
var container = builder.Build(ctx =>
{
    ctx.AddFeature();
    ctx
      .ForStrategy<AppSettings>()
      .Use<SomeFancyStrategyImplementation>();
});
```

## Custom Strategy With Dependency Injection

Let’s assume that we have a fancy strategy that is depended on even more fancier service to work properly:

```csharp
public class StrategyWithConstructorParameterReader : BaseStrategyImpl
{
    public StrategyWithConstructorParameterReader(ISampleInjectedInterface sample)
    {
...
```

So to properly configure dependency composition root we need to configure `FeatureSetBuilder` to inject `ISampleInjectedInterface` properly.

To do so you can use `dependencyConfiguration` parameter of `Build()` method:

```csharp
builder.Build(dependencyConfiguration: ex => ex.For<ISampleInjectedInterface>().Use<SampleInjectedInterface>());
```

## More Information

More information on [extending](https://github.com/valdisiljuconoks/FeatureSwitch/wiki/Extending-FeatureSwitch-library) FeatureSwitch library.

Happy switching the features!

[*eof*]
