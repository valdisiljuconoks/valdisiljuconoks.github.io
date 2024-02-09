---
title: Analyzing EPiServer page type source code
author: valdis
date: 2012-03-02 05:15:00 +0200
categories: [.NET, C#, Episerver, Optimizely]
tags: [.net, c#, episerver, optimizely]
---

Creating pages in EPiServer using `PageTypeBuilder` is pretty simple and very convenient. However sometimes I run into issue that forgetting to mark page type property as virtual (I suppose this is required for property access interception).

Playing around `Microsoft Roslyn` project I came up with an idea to analyze the source code for page type builder classes and provide some feedback on analyze results.

![](/assets/img/2012/03/1.png)

Hovering over the property declaration VS gives you a small tooltip that property has to be marked as virtual otherwise runtime exception will be thrown.

![](/assets/img/2012/03/2.png)

Plugin scans only classes that are decorated with `PageType` attribute and  properties decorated with `PageTypeProperty` attributes.

There is a space to expand and enrich functionality of such a plugin as for instance develop some `FxCop` rules designed for PTB specifics, `CodeActions` to execute if code issue is found (e.g. converting property to virtual property).

Currently it’s only sandbox project utilizing `Roslyn` possibilities and can be distributed as VS extension (`.vsix`).

This is just a sneak preview of what’s could be accomplish but I wanted to ask you guys is this something that would be useful and are interested in?
