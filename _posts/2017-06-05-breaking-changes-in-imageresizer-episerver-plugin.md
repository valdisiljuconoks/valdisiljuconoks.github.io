---
title: Breaking Change in ImageResizer EPiServer Plugin
author: valdis
date: 2017-06-05 13:30:00 +0200
categories: [Add-On, .NET, C#, Episerver, Optimizely, Image Resizer]
tags: [add-on, .net, c#, open source, episerver, optimizely, image resizer]
---

Usually I don't post small changes in separate blog post, but I think that it's worth to mention breaking changes in latest ImageResizer (IR) plugin for EPiServer.

## Before

If you are using [Fluent API](https://github.com/valdisiljuconoks/ImageResizer.Plugins.EPiServerBlobReader#render-image-markup-fluent) to resize the image, then you may face inconsistency in API.
Passing in `null` or `ContentReference.EmptyReference` you will get an `ArgumentNullException` exception:

```razor
@Html.ResizeImage(null, 100, 100)
```

However, if passing in `string.Empty` for the another overload - you would get back empty string:

```razor
@Html.ResizeImage(string.Empty, 100, 100)
```

Why this is bad - you can read more [here](https://www.nczonline.net/blog/2009/11/30/empty-image-src-can-destroy-your-site/). Thx to [Kaspars](https://github.com/kaspars-ozols) for the link.
This is inconsistent API and should be fixed.

## After

Now (in v6.0 and after) in any case when markup could potentially end up with empty `src` attribute - `ResizeImage` **will** throw an exception. Both invokes will blow up:

```razor
@Html.ResizeImage(null, 100, 100)
@Html.ResizeImage(string.Empty, 100, 100)
```

## Render with Fallback

Now, to render image from the page property could be cumbersome. You also would need to check for existence of the content reference.

```razor
@{
    var imageUrl = Model.ImageLink.IsNullOrEmpty()
        ? new UrlBuilder(string.Empty)
        : Html.ResizeImage(Model.ImageLink, 1600, 670);
}

<div data-src="@imageUrl">
```

To make life easier, there is a new overload in the library:

```razor
<div data-src="@Html.ResizeImageWithFallback(Model.ImageLink, "", 1600, 670)">
```

Anyways, just **be sure** to pass in valid fallback image path if you still resizing for `img` element, otherwise - you still might destroy your site.

Happy resizing!

[*eof*]
