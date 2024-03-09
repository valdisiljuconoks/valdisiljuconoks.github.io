---
title: ImageResizer Plugin for EPiServer v10
author: valdis
date: 2016-10-31 10:30:00 +0200
categories: [Add-On, Optimizely, Episerver, .NET, C#, Image Resizer]
tags: [add-on, optimizely, episerver, .net, c#, image resizer]
---

ImageResizer.Net Plugin for EPiServer v10 **is released**. `ImageResizer.Plugins.EPiServerBlobReader` got version 5.0.0.

"Old" NuGet package for EPiServer 9.x is updated with upper version constraint fix. Now plugin package v4.2 has upper EPiServer version set under v10 exclusive.

## Note About Image Preview

That was quite "interesting" experience to realize changes in `EPiServer.Web.BlobHttpHandler`. This is eventually the guy who is responsible for returning Blob back to the client. A bit more details below.

### Background

First of all a bit background. Images in EPiServer EditUI gets different Url than on website "runtime". If image is `/globalassets/image1.jpg` then when you will ask `Url.ContentUrl()` in EPiServer EditUI it will become something like `/episerver/cms/Content/globalassets/image1.jpg,,x_y?epieditmode=False....`. Here is nothing fancy, but as far as I understand - ImageResizer (IR) does not kick in for the Urls containing commas "`,`". I opened [GitHub issue](https://github.com/imazen/resizer/issues/195) for them. Awaiting authors response.

### "Solution" (more like Workaround)

While debugging and joggling with internals and decompiled code in `EPiServer.Web.BlobHttpHandler`, I came to the conclusion that there are changes how handler decides what to do with requested `Blob`.

While ImageResizer haven't provided official reply on issue for handling urls with command "`,`", we have to remove "`,,x_y`" part from the Url to give possibility for IR to kick in and do proper image resizing even in EditUI mode.

There also have been [lengthy discussions over GitHub](https://github.com/valdisiljuconoks/ImageResizer.Plugins.EPiServerBlobReader/issues/10) about this issue and whether to implement it or not.

That's what I would at least expect being an editor, and developer. Using built-in templates like (`Image.cshtml` in AlloyTech):

```razor
@model EPiServer.Core.ContentReference
@if (Model != null)
{
    <img src="@Url.ContentUrl(Model)" alt="" />
}
```

I would expect that image acts properly if I apply IR commands to it. Even for instance using own helpers:

```razor
@using ImageResizer.Plugins.EPiServer

<img src="@Html.ResizeImage(Model.CurrentPage.MainImage, 100, 100)"/>
```

### The Issue with Unpublished Images

Unfortunately there is logic inside `BlobHttpHandler` that determines what to do with Blob after it's being retrieved from underlying store. Method is call `ProccessBlobRequest()`. I can't paste whole EPiServer source code here due to copyright issues, but snippet looks something like this (inside this `ProccessBlobRequest()`):

```csharp
var evaluator = ServiceLocator.Current.GetInstance<IRoutableEvaluator>();
if (content == null || !evaluator.IsRoutable(content))
{
    return false;
}
```

By returning `false` from this method, `BlobHttpHandler` decides to return `404` status code for the whole Http request - resulting in `File not found` error.

And the issue is inside this `IsRoutable()` method. Request is "routable" to current content if:

* if current content is published
* **or** current context mode is `Edit` or `Preview`.

As we are removing `,,x_y` segment from the Url while serving images due to lack of IR to kick-in -> we are losing ContextMode and EPiServer now thinks that context mode is `Default`.

As result - if image is not yet published (or you are browsing next draft of the content) - image will not be rendered in Preview.

**However**, sometimes I got image rendered within the same circumstances. It is pretty weird.

I'm not quite sure where exactly is the issue and even more - how to resolve this.


Anyway, 99% of cases seems to be working as expected.
Would be awesome if somebody from product team could comment on this issue. Thanks!

Happy resizing!

[*eof*]
