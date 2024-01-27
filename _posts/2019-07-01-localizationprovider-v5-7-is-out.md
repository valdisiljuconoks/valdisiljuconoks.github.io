---
title: LocalizationProvider v5.7 Is Out
author: valdis
date: 2019-07-01 18:00:00 +0200
categories: [Add-On, .NET, C#, Open Source, Localization Provider]
tags: [add-on, .net, c#, open source, localization provider, localization]
---

It's been a while since last Episerver `DbLocalizationProvider` package update. Fortunately this release includes couple bug fixes delivered by amazing developer community and also few new features.

What's included in this release:
* "Transient" referenced projects are now properly respected and scanned ([#github/28](https://github.com/valdisiljuconoks/localization-provider-core/issues/23))
* Resource import in Episerver was not polite and threw up when importing not active/disabled languages ([#github/30](https://github.com/valdisiljuconoks/localization-provider-epi/issues/30))
* Duplicate keys are ignored and first one is taken during MigrationTool process ([#pull/25](https://github.com/valdisiljuconoks/localization-provider-epi/pull/25))

There are also some other smaller bug fixes, but who really reads release notes?! So will not bother you with bureaucracy here.

Most of the effort went to implement long requested feature of tree view in package version for AspNet Core platform. Had some "what da heck?" moments while digging my details through [Vue.js](https://vuejs.org/) framework. Hope it works well and supports basic features you might need. It's still far from perfect, but hey - I have to leave something for the future to improve, otherwise I would be out of business ;)

![tree-view](/assets/img/2019/07/tree-view.png)

There were also thing or two that I unfortunately had to move to next version. Sorry for that. Will work on those within next version. Otherwise v5.7 would stay in "cookings" even longer..

Next major version (v6.0) is already in the oven and I believe features added there will make somebody happy while someone else might go crazy and/or swear at me :) Well.. that's what major versions are designed for, right?!

As usual, if you do have any feedback please share it on [GitHub](https://github.com/valdisiljuconoks/LocalizationProvider). You can use main package repository and I'll move issue to appropriate repository if needed.

Open-source package developers / maintainers - keep up good work! Community appreciate it!

Happy localizing!

[*eof*]
