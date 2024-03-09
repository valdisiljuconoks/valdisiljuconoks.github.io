---
title: Localization Provider Update
author: valdis
date: 2020-07-09 18:10:00 +0200
categories: [Add-On, Optimizely, Episerver, .NET, C#, Open Source, Localization Provider]
tags: [add-on, optimizely, episerver, .net, c#, open source, localization provider, localization]
---

Hi all,

Not a huge pile of news and features (those I've saved for v7.0).

It's been a while since v1.0 of LocalizationProvider. Originally it has its inception during [my company's](https://getadigital.com) hackathon - we were trying to solve some of the outstanding issues that we came across while working with localized resources based on XML files in [Episerver CMS](https://world.episerver.com) platform. It was few years ago.

At first, package back then was humble enough only to target Episerver as supported runtime. But over the time - I saw opportunity (and actually requirement in one of our projects) to support also classical ASP.NET applications (MVC). And of course - [ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/) came as logical continuation of the journey.

Since then I was working mostly alone on the project. But somehow during v6 development - I started to receive some really great contributions from the community.

For example, in v6 I extracted underlying SQL Server dependency (and also threw out EF dependency - which actually unfortunately led to startup performance improvements). Unfortunately - because IMHO EF has its potential but there are definitely [space for improvement](https://twitter.com/isaac_abraham/status/1279817037255229440). By extracting dependency into its own package - I introduced provider-based infrastructure (still some things needs to be cleaned-up and moved to base library). And this extraction led to new package - storage implementation based on [PostgreSQL database engine](https://www.nuget.org/packages/LocalizationProvider.Storage.PostgreSql/). For which I just had to do some minor code reformatting and merging into master.

Probably I should blog about provider model and CQRS approach which resulted in quite nice extensible / overwritable feature.

Just wanted to shout out big thanks to everyone who has contributed to the project!!

In latest v6.2 there are some other smaller features and bug fixes (also - mostly contributed by community). You can follow list of issues [here](https://github.com/valdisiljuconoks/LocalizationProvider/milestone/20), [here](https://github.com/valdisiljuconoks/localization-provider-core/milestone/6) and [here](https://github.com/valdisiljuconoks/localization-provider-opti/milestone/12). So if you have stumbled upon one of them - please take time and update packages.

Some sneak peek for v7:

* usability features for Episerver and .NET Core AdminUI
* end of support for classical ASP.NET MVC runtime (relevant code base will be merged into base package)
* import/preview for .NET Core AdminUI
* support for adding notes / comments for resources
* proper support for multi-instance startup in Azure
* storage implementation for [Azure Table Storage](https://docs.microsoft.com/en-us/azure/cosmos-db/table-storage-overview)
* merge of GitHub repositories back to single one (over the time its proven that submodules are not necessary in this project)

If you have any additional feature request / bug report - please file it in [GitHub](https://github.com/valdisiljuconoks/localizationprovider/issues).

Happy localizing and hold-on for the v7 :)

[*eof*]
