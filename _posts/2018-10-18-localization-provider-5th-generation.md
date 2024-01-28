---
title: Localization Provider - 5th generation out!
author: valdis
date: 2018-10-18 00:30:00 +0200
categories: [Add-On, .NET, C#, Localization Provider, Episerver]
tags: [add-on, .net, c#, open source, localization, Episerver]
---

Freshly baked v5.0 is out now. Changes in this version (some of them are breaking ones). Mostly release was focused on .Net Core AdminUI improvements and new features. As I'm preparing for something bigger for the library, so I need to do some prerequisites :)

General:

- strong naming of all provider assemblies. You might be wondering why? [because reasons](https://docs.microsoft.com/en-us/dotnet/standard/library-guidance/?ref=tech-fellow.eu).
- Added [SourceLink](https://github.com/dotnet/sourcelink/blob/master/README.md?ref=tech-fellow.eu) support for main library (["LocalizationProvider.Abstractions"](https://www.nuget.org/packages/LocalizationProvider.Abstractions/?ref=tech-fellow.eu) and "[LocalizationProvider](https://www.nuget.org/packages/LocalizationProvider/?ref=tech-fellow.eu)") and .Net Core ("[LocalizationProvider.AspNetCore](https://www.nuget.org/packages/LocalizationProvider.AspNetCore/?ref=tech-fellow.eu)"). If SourceLink support will be requested for Framework libraries (Episerver and Asp.Net Mvc) - I can work on that as well.
- various bug fixes in main package mostly related to language proper fallback in cases of missing translation for requested language

Episerver:

- Add new resource manually is now back in Episerver AdminUI. Special thanks goes to [@ev2000](https://twitter.com/ev2000?ref=tech-fellow.eu) for motivation :)

![](/assets/img/2018/10/2018-10-16_17-33-25.jpg)

Asp.Net Core:

- Target to .Net Core App 2.1
- User interface is rewritten to [Razor Class Library](https://docs.microsoft.com/en-us/aspnet/core/razor-pages/ui-class?view=aspnetcore-2.1&tabs=visual-studio&ref=tech-fellow.eu)
- You can now export resources from UI (import with diff merge is coming, that's a bit bigger chunk of work)
- You can now also customize look and feel of the admin UI via custom external CSS file (UiConfigurationContext.CustomCssPath setting)
- `Localized AdminUI` itself and added all those resources [as hidden](https://github.com/valdisiljuconoks/LocalizationProvider/blob/master/docs/hidden-resources.md?ref=tech-fellow.eu) (so there will be less noise by default in the resource list)
- Proper support for role based authorization. Be sure to [enable roles](https://github.com/aspnet/Identity/issues/1813?ref=tech-fellow.eu) in .Net Core 2.1 properly

Good luck and Happy localizing!

[*eof*]
