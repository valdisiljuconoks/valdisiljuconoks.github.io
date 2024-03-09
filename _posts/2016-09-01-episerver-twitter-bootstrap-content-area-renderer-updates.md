---
title: EPiServer Twitter Bootstrap Content Area Renderer Updates
author: valdis
date: 2016-09-01 10:30:00 +0200
categories: [.NET, C#, Episerver, Optimizely]
tags: [.net, c#, episerver, optimizely]
---

Here comes small updates for the Content Area renderer in the latest version:

* Now `EPiServer.Forms` are supported
* Possibility to have your own Css classes
* `DisplayOption` is now available in block template as well (if needed)
* Some smaller bug fixes

## Naming Challenges

As you might know - I'm still struggling with naming of this package. Originally it was planned to be as `Twitter Bootstrap Content Area Renderer`, but apparently over time package feature set deviated from original idea as now package gives so much more than just adding necessary Css classes to the content area items.
But naming is hard, so I'll just leave it here :)

## Support for EPiServer.Forms

With the latest package version + a smaller new package (EPiBootstrapArea.Forms) now `DisplayOption` can be applied to EPiServer Forms elements as well.

![](/assets/img/2016/08/2016-08-31_11-52-33-2.png)

How this is done - I guess it needs separate blog post. Will publish soon.

## Add Your Own Css Classes

Latest version now also gives you possibility to add your own Css classes for the cases when Bootstrap ones are not the thing you were looking for.

So all you need to do is to provide your own Css class pattern while registering your [custom display options](https://github.com/valdisiljuconoks/EPiBootstrapArea#register-custom-provider):


```csharp
public class DisplayModeFallbackCustomProvider : DisplayModeFallbackDefaultProvider
{
    public override List<DisplayModeFallback> GetAll()
    {
        var original = base.GetAll();

        original.Add(new DisplayModeFallback
        {
            Name = "This is from code (1/12)",
            Tag = "one-12th-from-code",
            LargeScreenWidth = 1,
            LargeScreenCssClassPattern = "large-{0}",
            MediumScreenWidth = 2,
            MediumScreenCssClassPattern="medium-{0}-the-size",
            SmallScreenWidth = 3,
            SmallScreenCssClassPattern = "small-{0}",
            ExtraSmallScreenWidth = 4,
            ExtraSmallScreenCssClassPattern = "xs"
        });

        return original;
    }
}
```

In turn, this will produce following Css classes for the particular content area item with selected display option:

```html
<div class="... large-1 medium-2-the-size small-3 xs ...">
```

Of course `1`, `2`, `3` and `4` are fictive numbers, but still - I guess you understood the pattern and its usage.

Btw, if you will mismatch `string.Format()` placeholders and use for instance `{5}` - no Css class will be added for particular screen form factor.

## Get Selected DisplayOption

Sometimes it's required to access selected `DisplayOption` from the block template itself - one of the application case was that template needed to make decision about markup generated based on selected display mode (no only just applying necessary Css classes to content area item wrapping element). You might need to write some clumsy code to get to this information.. But not anymore :) Now you can just use following code to retrieve selected `DisplayOption` for current item (`SampleBlock.cshtml`):

```razor
@using EPiBootstrapArea
@model SampleBlock

...

@Html.GetDisplayOption(Model)
```

Happy bootstrapping!

[*eof*]
