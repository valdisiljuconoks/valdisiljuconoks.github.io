---
title: Specify Default DisplayOption in the Content Area
author: valdis
date: 2016-05-09 10:30:00 +0200
categories: [.NET, C#, Episerver, Optimizely]
tags: [.net, c#, episerver, optimizely]
---

Time to time when working on various projects we come across requirement to control somehow which `DisplayOption` will be selected as default, once content is placed inside particular `ContentArea`.

![](/assets/img/2016/05/2016-05-08_22-46-18-2.png)

As most of our projects are running under **Twitter Bootstrap** system - this is ideal feature request for [EPiServer Bootstrap Content Area](http://nuget.episerver.com/en/OtherPages/Package/?packageId=EPiBootstrapArea) plugin. As I'm always trying to be developer friendly - I would like to be able to define these default display option rules in the code. And finally **version 3.3** got these features.

## Default DisplayOption for Block

So now with latest version you can specify which display option to use if block is dropped inside content area:

```csharp
using EPiBootstrapArea;

public static Class ContentAreaTags
{
    public const string HalfWidth = "half-width";
}

[DefaultDisplayOption(ContentAreaTags.HalfWidth)]
public class SomeBlock : BlockData
{
    ...
}
```

Constant `"half-width"` is `Tag` of the display option registered within EPiServer bootstrap content area renderer (either [by code](https://github.com/valdisiljuconoks/EPiBootstrapArea/blob/master/Providers/DisplayModeFallbackDefaultProvider.cs) or [manually](https://github.com/valdisiljuconoks/EPiBootstrapArea/blob/master/IDisplayModeFallbackProvider.cs)).

This attribute will make sure that if block is dropped inside content area - display option registered with tag `"half-width"` is used.

Editor of course can override this and set display option explicitly.

![](/assets/img/2016/05/2016-05-09_18-12-42.png)

## Default DisplayOption for Content Area

The same attribute can be used in `ContentArea` property definition:


```csharp
using EPiBootstrapArea;

[ContentType(DisplayName...]
public class StandardPage : PageData
{
    [DefaultDisplayOption(ContentAreaTags.HalfWidth)]
    public virtual ContentArea MainContentArea { get; set; }
    ...
}
```

Using this attribute - you are ensuring that any content dropped inside this particular content area will use display option tagged as "half-width" if editor haven't specified otherwise.

## Default DisplayOption for Tagged Block

[Template tags](http://world.episerver.com/documentation/Items/Developers-Guide/Episerver-CMS/9/Rendering/Selecting-template-based-on-tag/) is pretty powerful way to customize the same block to render differently based on where exactly it's placed.

Let's assume that we do have a block definition:

```csharp
public class SomeBlock : BlockData
{
    ...
}
```

And we are rendering content area on the page with specific tag:

```razor
...
@Html.PropertyFor(m => m.MainContentArea, new { tag = "ca-tag" })
...
```

And let's say we do have a requirement - if this block is placed inside *tagged* content area - we need to use different display option compared to default display option that might be specified for content areas without any tags. Fortunately this feature is available in `EpiBootstrapArea` plugin.

Now you can specify default display option for specific `tag` (`"cs-tag"`):


```csharp
using EPiBootstrapArea;

[DefaultDisplayOptionForTag("ca-tag", ContentAreaTags.OneThirdWidth)]
public class SomeBlock : BlockData
{
    ...
}
```

Now `ContentAreaTags.OneThirdWidth` display option should be used by default if editor drops block inside content area and do not specify display option explicitly.

You can also mix-match with default display option and default display option for tagged cases:

```csharp
using EPiBootstrapArea;

[DefaultDisplayOption(ContentAreaTags.HalfWidth)]
[DefaultDisplayOptionForTag("ca-tag", ContentAreaTags.OneThirdWidth)]
[DefaultDisplayOptionForTag("ca-tag-other", ContentAreaTags.ThreeQuartersWidth)]
public class SomeBlock : BlockData
{
    ...
}
```

**NB!** This feature is not yet available for ContentArea property definition. You can use only `DefaultDisplayOption` there at the moment.


## Block vs ContentArea DisplayOption

In cases when there are conflicts between block's definition and content area definition, block's definition always wins.
Let's take a look at same. We do have following definitions:


```csharp
using EPiBootstrapArea;

[DefaultDisplayOption(ContentAreaTags.HalfWidth)]
[DefaultDisplayOptionForTag("ca-tag", ContentAreaTags.OneThirdWidth)]
[DefaultDisplayOptionForTag("ca-tag-other", ContentAreaTags.ThreeQuartersWidth)]
public class SomeBlock : BlockData
{
    ...
}

[ContentType(DisplayName...]
public class StandardPage : PageData
{
    [DefaultDisplayOption(ContentAreaTags.FullWidth)]
    public virtual ContentArea MainContentArea { get; set; }
    ...
}

@Html.PropertyFor(m => m.MainContentArea)
```

Content area is requesting content to take `ContentAreaTags.FullWidth` display option by default, but block is specifying that by default it will occupy only `ContentAreaTags.HalfWidth`.

In this case when instance of `SomeBlock` block is dropped on this content area, it will use `ContentAreaTags.HalfWidth` as default display option.

Happy tagging!

[*eof*]
