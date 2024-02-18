---
title: ImageResizer.Plugin.EPiServerBlobReader
author: valdis
date: 2014-03-16 09:15:00 +0200
categories: [.NET, C#, Episerver, Optimizely, Image Resizer]
tags: [.net, c#, episerver, optimizely, image resizer]
---


`ImageResizer` is one of top choice library when it comes to optimizing media assets in web context. EPiServer 7.5 introduced really cool feature [Asset and media](http://world.episerver.com/Documentation/Items/Developers-Guide/EPiServer-CMS/75/Content/Assets-and-media/Assets-and-media/).

You need to thank and follow Martin Pickering [hard work](http://world.episerver.com/Code/Martin-Pickering/ImageResizingNet-integration-for-CMS75/) to get both of these worlds working together which involves a bit of hacking here and there.

Recently I got sick of repeating these steps from project to project and came up with a [package of NuGet](http://nuget.episerver.com/en/OtherPages/Package/?packageId=ImageResizer.Plugins.EPiServerBlobReader).

Package gently modifies your web.config file in order to add new plugin to the list of [ImageResizer plugins](http://imageresizing.net/plugins).

## Source Source

Source code is available either [here](http://world.episerver.com/Code/Martin-Pickering/ImageResizingNet-integration-for-CMS75/) or on [GitHub](https://github.com/valdisiljuconoks/ImageResizer.Plugins.EPiServerBlobReader).

Happy coding!

[*eof*]
