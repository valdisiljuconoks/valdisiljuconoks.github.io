---
title: FeatureSwitch meets Glimpse
author: valdis
date: 2015-01-02 22:55:00 +0200
categories: [.NET, C#, Feature Switch]
tags: [.net, c#, feature switch]
---

`FeatureSwitch` library got yet another plugin in its family. Glimpse is an amazing framework for troubleshooting and tracing your web application, even in production environment. If by coincidence you are a consumer of this framework and also use `FeatureSwitch` library, then now they run together. You can access your features through Glimpse control panel.

## Installation
You need to install a package from [nuget.org](https://www.nuget.org/packages/FeatureSwitch.Glimpse/) feed:

```
> Install-Package FeatureSwitch.Glimpse
```

Using `FeatureSwitch.Glimpse` package you can get an overview of your features and its state through `FeatureSwitch` tab.

![](/assets/img/2015/01/glimpse-1-1.png)

There is also an Glimpse’s resource which provides you an access to `FeatureSwitch` control panel. You can enable/disable particular feature.

![](/assets/img/2015/01/glimpse-2.png)

If check-box for the feature is disabled in Glimpse control panel it means that there are only read-only strategies configured for feature.

## Fix for incorrect feature state

There could be a case when incorrect feature state is shown in Glimpse’s resource `FeatureSwitch` Config. If you your feature has strategy based on `HttpSession` then there is a high chance that state for the feature will be incorrect. This is due to the fact that Glimpse `HttpHandler` is not session state enabled. While executing Glimpse resource Session is set to null which means that `FeatureSwitch` context detects this feature as disabled.
 If you need correct state this is easy to fix. Find following line in web.config file:

```xml
<system.webServer>
  <handlers>
    <add name="Glimpse" path="glimpse.axd" verb="GET" type="Glimpse.AspNet.HttpHandler, Glimpse.AspNet" preCondition="integratedMode" />
  </handlers>
</system.webServer>
```

This controls what is invoked when accessing Glimpse. You need to change Glimpse’s default `HttpHandler` to `FeatureSwitch` custom one:

```xml
<system.webServer>
  <handlers>
    <add name="Glimpse" path="glimpse.axd" verb="GET" type="FeatureSwitch.Glimpse.SessionHttpHandler, FeatureSwitch.Glimpse" preCondition="integratedMode" />
  </handlers>
</system.webServer>
```

Now correct state should show up in `FeatureSwitch` control panel as `HttpSession` is now available for Glimpse’s resources.


Happy coding!

[*eof*]
