---
title: What's new in FeatureSwitch?
author: valdis
date: 2015-09-15 21:15:00 +0200
categories: [.NET, C#, Open Source, Feature Switch]
tags: [.net, c#, open source, feature switch]
---

It's been while since last FeatureSwitch library release. Just pushed out a bit of stuff held under the hood. Some of them are requested by my lovely consumers :)
Here are enlisted some of the latest version features in the library.

1) *EPiServer Site Strategy*. Now you can use EPiServer's site strategy to enable or disable feature per particular EPiServer site. With example below feature `ThisFeatureIsOnlyForSite1` will only be enabled if request is made against site named "Site1", otherwise - feature will be disabled.

```csharp
[EPiServerSite(Key = "Site1")]
public class ThisFeatureIsOnlyForSite1 : BaseFeature {}
```

In some enterprise solutions you can actually use multiple sites if needed, separated by comma.

```csharp
[EPiServerSite(Key = "Site1, Site2")]
public class ThisFeatureIsOnlyForSite1 : BaseFeature {}
```

2) *Authorized Strategy*. Now there is a strategy that you can enable for users in specific role. With code below seems like that feature is really important as it will be enabled only for users in "Administrators" group.

```csharp
[AuthorizedFor(Key = "Administrators")]
public class AuthorizedFeature : BaseFeature {}
```

And also you can "relax" feature a bit and make it available for "Administrators" and also for "CmsAdmins".

```csharp
[AuthorizedFor(Key = "Administrators, CmsAdmins")]
public class AuthorizedFeature : BaseFeature {}
```

Feature will be disabled for users in any other role (also for users that are not in any role).

3) *Cloud Configuration Support*. As we are moving more and more to skies, thought it may be valuable to have some tiny cloud (Microsoft Azure) support in FeatureSwitch library. Now by installing `FeatureSwitch.Azure` package you can get access to features that store their state in cloud and is accessible via `CloudConfigurationManager`.

```csharp
[CloudConfiguration(Key = "CloudFeature")]
public class ThisFeatureIsInTheSkies : BaseFeature {}
```

Settings are easily available via Azure management portal.

![](/assets/img/2015/09/cloud-feature-1.png)

4) *EPiServer 6.0 support dropped*. Your open source code at the moment of writing already turns into liability you commit yourself to support and maintain. At some point you just feel that you need to let some things go. In this release of `FeatureSwitch` library EPiServer v6.0 support was thing that was dropped.
For developers it means that package `FeatureSwitch.EPiServer` from version [2.4.0](http://nuget.episerver.com/en/OtherPages/Package/?packageId=FeatureSwitch.EPiServer&packageVersion=2.4.0) is now requiring EPiServer v8.0 version.

If you have previously used `FeatureSwitch.EPiServer.Cms75` package (up to [v2.3.2](http://nuget.episerver.com/en/OtherPages/Package/?packageId=FeatureSwitch.EPiServer.Cms75&packageVersion=2.3.2)) to target specifically version 7.x of EPiServer, you can now either update the package or remove it and install `FeatureSwitch.EPiServer`. Updated version of the package is empty and is referencing `FeatureSwitch.EPiServer` package (this is required for backward compatibility and easier upgrade path for consumers of Cms75 package).

Anyway, as always feedback is much appreciated.

Happy featuring!

[*eof*]
