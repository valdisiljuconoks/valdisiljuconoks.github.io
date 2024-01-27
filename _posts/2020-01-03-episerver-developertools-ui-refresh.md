---
title: Episerver DeveloperTools - UI Refresh
author: valdis
date: 2020-01-03 21:45:00 +0200
categories: [Add-On, Optimizely, Episerver, .NET, C#, Open Source]
tags: [add-on, optimizely, episerver, .net, c#, open source]
---

Been busy with some UI refreshment effort on [Episerver DeveloperTools](https://nuget.episerver.com/package/?id=EPiServer.DeveloperTools).

Latest version (v3.5) released on NuGet feed has following changes:

* now UI rewritten in Razor (yes! Episerver developer tools were still written in WebForms).
* With rewrite to Razor view engine and due to some issue in Episerver module delivery mechanism (specifically when module is served from Zip archive) included `web.config` file is ignored resulting in runtime errors (more info [here](https://world.episerver.com/forum/developer-forum/Developer-to-developer/Thread-Container/2019/4/issue-with-razor-views-while-developing-custom-add-on/#204171)). Because of this change - `Episerver.DeveloperTools` is served as "expanded" module - meaning that all view files are copied to target project under appropriate protected module directory. **NB!** Upgrade from previous versions have been tested in sandbox project, but if you face some runtime issues - so just you know of this change!
* also dependency is now set on Episerver UI `[11.21.1, 12)`. Which means that UI will be according to new blue-ish theme.

![dev-tools-1](/assets/img/2019/11/dev-tools-1.png)

* DeveloperTools package have many menu items and might be so that you will have to scroll in order to get to tools outside visible menu area
* Some smaller UI fixes are applied (like adding icons to buttons, etc).

**NB!** Idea in next iteration is to split all tools apart into separate packages. For example - `EPiServer.DeveloperTools.IoC` or `EPiServer.DeveloperTools.LoadedAssemblies`. Split would give you an option to install only what you need to use. Still `EPiServer.DeveloperTools` package would have reference to all the other packages (very similar as we have seen in `Microsoft.AspNetCore.App` package) and would act as "umbrella" package to pull all required packages. Please give me feedback on this idea (pros/cons) in comments below.

If you have any other idea what would be great addition to the tool belt - please reach me out!

To finish this post I'm adding module dependency tool just to remind you how beautiful your Episerver project module dependencies might look :)

![dev-tools-2](/assets/img/2019/11/dev-tools-2.png)

Happy tooling!

[*eof*]
