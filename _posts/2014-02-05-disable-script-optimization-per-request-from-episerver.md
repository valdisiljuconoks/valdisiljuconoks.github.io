---
title: Disable script optimization per request from EPiServer
author: valdis
date: 2014-02-05 13:35:00 +0200
categories: [.NET, C#, Add-On, Episerver, Optimizely, Feature Switch]
tags: [.net, c#, add-on, episerver, optimizely, feature switch]
---

Image that you are running the site in production environment and following best practices for client-side page performance optimization. You are using script or style bundling and minification approach to reduce size of assets to download decreasing time to load for the page.

```razor
 @System.Web.Optimization.Styles.Render(“~/Content/css”)
```

Let’s assume that you received a bug report from the customer complaining that one of the page is not loading correctly. You navigate to the page and see something similar in console output:

![](/assets/img/2014/02/error-stack.png)

At the top of the call stack is guilty target.

```
ReferenceError: ‘a’ is not defined in /site-scripts?v=P4j3J… Line 1: Column 45669
```

Viewing the source of the file does not help much either.

![](/assets/img/2014/02/view-source.png)

Question is **now what**?

Looking at `System.Web.Optimization.Styles|Scripts.Render()` method source code we can see that eventually code is using `BundleTable.EnableOptimizations` property which on another hand is affecting all script or style rendering – it means that be enabled or disabling optimizations – you are affecting all site users.
The case is that actually you want script or style optimization to be disabled only for your particular debugging request without affecting other site users. It means that you have to turn feature on or off conditionally.

## Styles and Scripts conditional optimization

Using [FeatureSwitch](https://github.com/valdisiljuconoks/FeatureSwitch/wiki#overview) library it’s easily possible to turn optimization on or off only for particular request, session, context, whatever..
By using FeatureSwitch [strategies](https://github.com/valdisiljuconoks/FeatureSwitch/wiki#strategies) you can define your own feature that relies for instance on [QueryString](http://msdn.microsoft.com/en-us/library/system.web.httprequest.querystring(v=vs.110).aspx) value provided within the request.

Or you can use built-in features for disable script and style optimizations:

```csharp
namespace FeatureSwitch.Web.Optimization {

    public class DisableScriptOptimization { }
    public class DisableStyleOptimization { }
}
```

These 2 features are waiting for particular query string and writing selection in Http session if available. Feature is defined as shown below:

```csharp
namespace FeatureSwitch.Web.Optimization
{
    [QueryString(Key = “DisableScriptOptimization”)]
    [HttpSession(Key = “FeatureSwitch_DisableScriptOptimization”, Order = 1)]
    public class DisableScriptOptimization : BaseFeature
    {
    }
}
```

`FeatureSwitch.Web.Optimization` plugin provides new Styles and Scripts types for rendering bundles and can be used in following way:

```razor
@using Scripts = FeatureSwitch.Web.Optimization.Scripts
@using Styles = FeatureSwitch.Web.Optimization.Styles

…

@(Styles.Render<DisableStyleOptimization>(“~/Content/css”))
@(Scripts.Render<DisableScriptOptimization>(“~/bundles/modernizr”))
```

### Local Debug environment
Behind the scenes if feature is not enabled `FeatureSwitch.Web.Optimization` library relies on `System.Web.Optimization` functionality meaning that locally if you are running site in debug environment (`HttpContext.Current.IsDebuggingEnabled` is returning true) scripts or styles will not be bundled.

## EPiServer Control Panel support
FeatureSwitch EPiServer integration library provides an easy way to integration UI Control Panel into [EPiServer CMS](http://www.episerver.com) platform for enabling or disabling particular feature on demand directly from EPiServer Admin Panel.

![](/assets/img/2014/02/epi-control-panel.png)

### Setup
No special setup is needed in order to integrate `FeatureSwitch` EPiServer's library it’s done automatically by `InitializableModule` ([code](https://github.com/valdisiljuconoks/FeatureSwitch/blob/master/FeatureSwitch.EPiServer/FeatureSwitchInit.cs))

### Security Setup
 Currently UI Control Panel is mapped to `~/modules/FeatureSwitch` route and is secured by `Administrators` user group access security policy.
 Currently if you are using `InitializableModule` there is no easy way to customize this other than registering route and securing location by yourself (copy code from `FeatureSwitchInit.cs`).

## Availability
EPiServer integration library is available on [nuget.episerver.com](http://nuget.episerver.com/en/OtherPages/Package/?packageId=FeatureSwitch.EPiServer.Cms75) and Web.Optimization package is available on [nuget.org](https://www.nuget.org/packages/FeatureSwitch.Web.Optimization/) feed appropriately.
FeatureSwitch package is available for EPiServer 7.x version.

## Feedback
Feedback is always welcome!

Happy feature switching!

[*eof*]
