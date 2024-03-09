---
title: What New APIs Are Coming in Optimizely Content Cloud vNext?
author: valdis
date: 2021-08-19 00:10:00 +0200
categories: [Add-On, Optimizely, Episerver, .NET, C#, Open Source]
tags: [add-on, optimizely, episerver, .net, c#, open source]
---

## Intro
I was wondering what new assemblies and public APIs are coming in new version of Content Management System from Optimizely (ex. EPiServer).
I took standard AlloyTech site created via Episerver VS integration plugin (for CMS11) and [.NET Core Preview](https://github.com/episerver/netcore-preview) site (**v12.0.1-pre-022064**) and ran some comparison script I hacked together at evening.

## Assemblies
These are brand new assemblies coming in CMS v12 (aka Optimizely Content Clound). Note that package and assmeblies names still contain prefix "EPiServer". This is by design and it will make sure that upgrade process is much smoother.

* EPiServer.CMS
* EPiServer.Cms.AspNetCore
* EPiServer.Cms.AspNetCore.HtmlHelpers
* EPiServer.Cms.AspNetCore.Mvc
* EPiServer.Cms.AspNetCore.Routing
* EPiServer.Cms.AspNetCore.Templating
* EPiServer.Cms.UI.Admin
* EPiServer.Cms.UI.VisitorGroups
* EPiServer.Framework.AspNetCore
* EPiServer.Hosting


Assemblies left in CMS 11:

* EPiServer.Cms.AspNet
* EPiServer.Configuration
* EPiServer.Data.Cache
* EPiServer.Framework.AspNet
* EPiServer.LinkAnalyzer
* EPiServer.Web.WebControls

![asm](/assets/img/2021/08/asm.png)

This is full list comparison:

![asm-full](/assets/img/2021/08/asm-full.png)

## New Public Types
Here it's interesting to compare what new types have showned up in assemblies that are still present from CMS11.

Some of the types have been moved from `Internal` (CMS11) to public namespace (CMS12). Also some of types have been moved from one assembly to another (for example, `EPiServer.Web.QuickNavigatorMenu` defined in `EPiServer.Cms.AspNet.dll` moved to `EPiServer.Shell.UI`).

Some of the types have been transformed from `internal` to `public`.

I haven't been digging deeper for each of the type and it's goal in the platform.

Below you can find all new types in CMS12 but defined or moved to assemblies that exist CMS11 as well:
