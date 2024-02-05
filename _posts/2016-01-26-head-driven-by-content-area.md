---
title: <head> driven by Content Area
author: valdis
date: 2016-01-26 10:30:00 +0200
categories: [.NET, C#, Episerver, Optimizely]
tags: [.net, c#, episerver, optimizely]
---

For some powerful sites and editors sometimes it's wise to give them power to manipulate some stuff in `<head>` area. One of the simplest way to give editors this possibility is to create ContentArea where editors could create particular set of available blocks that would output themselves between `<head>` elements.

By default EPiServer will generate wrapping element around content area (`div` tag name is actually controllable as well, more info [here](https://tech-fellow.eu/2015/06/11/content-area-under-the-hood-part-3/)):

```razor
@Html.PropertyFor(m => m.PageHeaderArea)
```

Resulting in:

```html
<div>                 <!-- CA wrapper element -->
    <div ...>         <!-- Block element -->
        <...>         <!-- Actual content of the block -->
    </div>
</div>
```

EPiServer gives you an option to skip wrapper element generation - resulting only in set of blocks added to the content area.

```razor
@Html.PropertyFor(m => m.PageHeaderArea, new { HasContainer = false })
```

Resulting in:

```html
<div ...>         <!-- Block element -->
    <...>         <!-- Actual content of the block -->
</div>
```

However, we still see that wrapping `<div>` element is not desired in `<head>` area.

Looking for the best place to add feature to skip even further - not to generate block wrapping element, but only content of the block itself.. Found that [Twitter Bootstrap aware ContentAreaRender](http://nuget.episerver.com/en/OtherPages/Package/?packageId=EPiBootstrapArea) could be a perfect spot for new functionality.

So with latest version (v3.3.3) you can now write markup something like shown below.

**UPDATE!** You need to use `HasEditContainer = false` to avoid rendering edit container for the `ContentArea`. Otherwise you may end up with invalid markup that `<head>` element contains `<div>` elements inside.

```razor
@Html.PropertyFor(m => m.PageHeaderArea,
                  new
                  {
                      HasContainer = false,
                      HasItemContainer = false,
                      HasEditContainer = false
                  })
```

Resulting in:

```html
<...>         <!-- Only actual content of the block -->
```

## Use Case - Meta Data Block

One of the use cases we had - to give editors possibility to add minifest files for the page by themselves.

So markup for the block is pretty straight forward:

```razor
@model DynamicMenu.Models.Blocks.MetaDataBlock

<link rel="@Model.RelationshipName" href="@Url.ContentUrl(Model.Link)">
```

ContentArea definition for the start page:

```csharp
public class StartPage : PageData
{
    [AllowedTypes(typeof(MetaDataBlock))]
    public virtual ContentArea HeaderContentArea { get; set; }
}
```

Then in `_SiteLayout.cshtml` (or any other layout page where you define `<head>`) you can write:

```html
<html>
<head>
    @Html.PropertyFor(m => startPage.HeaderContentArea,
                      new {
                            HasContainer = false,
                            HasItemContainer = false,
                            HasEditContainer = false
                          })
    ...
```

Resulting in:

```html
<html>
<head>
    <link rel="manifest" href="http://someserver/somefile.json">
    ...
```

Happy heading!

[*eof*]
