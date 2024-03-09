---
title: Episerver Images got Responsive Resize
author: valdis
date: 2018-10-25 15:30:00 +0200
categories: [Add-On, Optimizely, Episerver, .NET, C#, Image Resizer]
tags: [add-on, optimizely, episerver, .net, c#, image resizer]
---

Latest [ImageResizer](http://imageresizing.net/?ref=tech-fellow.eu) library version now can also generate `<picture>` element. Thanks to lately movement by [Erik](https://hacksbyme.net/about/?ref=tech-fellow.eu) and [Vincent](https://world.episerver.com/System/Users-and-profiles/Community-Profile-Card/?userid=be03d326-b35f-e611-9afb-0050568d2da8&ref=tech-fellow.eu).

When you need to deal with images and want to adapt to various screen sizes - it's time to switch to `<picture>` element.

Note that new method to generate picture element with all the corresponding sources and source sets is as simple as following:

```razor
@Html.ResizePicture(Model.CurrentPage.MainImage, PictureProfiles.SampleImage)
```

Current page's `MainImage` property is of type `ContentReference`. Basically simple straight forward Episerver property for image definition.

-- What are those picture profiles you passed as last element?

Picture profile is an entity that describes how image should be scaled and rendered in various cases. For example in this code sample we have `SampleImage` profile:

```csharp
public static PictureProfile SampleImage =
    new PictureProfile
    {
        SrcSetWidths = new[] { 480, 768, 992, 1200 },
        SrcSetSizes = new[]
        {
            "50vw",
        },
       DefaultWidth = 992
    };
```

Here we can specify couple of properties to customize `<picture>` element:

- Source set sizes (**SrcSetSizes**) - this regulates image size for various [media conditions](https://developer.mozilla.org/en-US/docs/Learn/HTML/Multimedia_and_embedding/Responsive_images?ref=tech-fellow.eu).
- Source set widths (**SrcSetWidths**) - this regulates various image sizes (resized by width specified here). Used to generate *srcset* attribute.
- Default width (**DefaultWidth**) - what is default width of the image. This is for old-school browsers those have no clue about `<picture>` element existence.

Code above generates following markup:

```html
<picture>
    <source sizes="50vw"
            srcset="/globalassets/batman.jpg?w=480 480w,
                    /globalassets/batman.jpg?w=768 768w,
                    /globalassets/batman.jpg?w=992 992w,
                    /globalassets/batman.jpg?w=1200 1200w">
    <img alt="" src="/globalassets/batman.jpg?w=992">
</picture>
```

Simple enough.

### Customized Image Url

You might be wondering, if helper method takes in `ContentReference` property, how am I going to specify various parameters for my images if I need to? For example "quality" is one of the example.

Roger! There is also a `UrlBuilder` overload, that is able to work with any arbitrary url. So, for example if I need to specify quality for all my images when generating `<picture>` element, I can actually resize "inline" and then render urls based on resulting image path:

```razor
@Html.ResizePicture(Model.CurrentPage.MainImage
                                     .ResizeImage()
                                     .Quality(10),
                    PictureProfiles.SampleImage)
```

This code will generate following markup:

```html
<picture>
    <source sizes="50vw"
            srcset="/globalassets/batman.jpg?quality=10&w=480 480w,
                    /globalassets/batman.jpg?quality=10&w=768 768w,
                    /globalassets/batman.jpg?quality=10&w=992 992w,
                    /globalassets/batman.jpg?quality=10&w=1200 1200w">
    <img alt="" src="/globalassets/batman.jpg?quality=10&w=992">
</picture>
```

### Multiple Picture Sources

Now imagine that for some silly reasons you need to supply multiple picture sources for various viewport widths or under any other conditions. Might sound silly, but I've seen websites like this.

Roger! This is also supported by the package. Now this can be implemented like this.

First, we need to define multiple properties to use in various media condition cases (in this case I'm just following [Bootstrap grid system](https://getbootstrap.com/docs/4.0/layout/grid/?ref=tech-fellow.eu) setup):

```csharp
public class StartPage : SitePageData
{
    [UIHint(UIHint.Image)]
    [Display(Description = "XS (< 576)", Order = 200)]
    public virtual ContentReference XSImage { get; set; }

    [UIHint(UIHint.Image)]
    [Display(Description = "Small (>= 576)", Order = 201)]
    public virtual ContentReference SMImage { get; set; }

    [UIHint(UIHint.Image)]
    [Display(Description = "Medium (>= 768)", Order = 202)]
    public virtual ContentReference MDImage { get; set; }

    [UIHint(UIHint.Image)]
    [Display(Description = "Medium (>= 992)", Order = 203)]
    public virtual ContentReference LGImage { get; set; }

    [UIHint(UIHint.Image)]
    [Display(Description = "Extra Large (>= 1200)", Order = 204)]
    public virtual ContentReference XLImage { get; set; }
```

Next we would need to define our picture profile:

```csharp
public static PictureProfile BootstrapGrid =
    new PictureProfile
    {
        DefaultWidth = 576,
        SrcMedias = new[]
                    {
                        "(min-width: 1200px)",
                        "(min-width: 992px)",
                        "(min-width: 768px)",
                        "(min-width: 576px)",
                        "(max-width: 575.98px)"
                    },
        SrcSetWidths = new[]
                       {
                           1200,
                           1200,
                           992,
                           768,
                           576
                       }
    };
```

Note, that we have now possibility to define various medias which are specified for every source element.

**NB!** Specified media attribute is generated in the sequence as specified here in picture profile. Note that CSS is cascade and first match top down will win.

And now the last piece in the puzzle - markup generation method (here also the order of the images is important to match source media order):

```razor
@Html.ResizePictures(new[]
                     {
                         Model.CurrentPage.XLImage,
                         Model.CurrentPage.LGImage,
                         Model.CurrentPage.MDImage,
                         Model.CurrentPage.SMImage,
                         Model.CurrentPage.XSImage
                     }, PictureProfiles.BootstrapGrid)
```

Following markup is generated here:


```html
<picture>
    <source media="(min-width: 1200px)" srcset="/globalassets/xl.jpg?w=1200">
    <source media="(min-width: 992px)" srcset="/globalassets/lg.jpg?w=1200">
    <source media="(min-width: 768px)" srcset="/globalassets/md.jpg?w=992">
    <source media="(min-width: 576px)" srcset="/globalassets/sm.jpg?w=768">
    <source media="(max-width: 575.98px)" srcset="/globalassets/xs.jpg?w=576">
    <img alt="" src="/globalassets/xl.jpg?w=576">
</picture>
```

Also arbitrary image urls are supported (in cases when you need to fine tuning each image):

```razor
@Html.ResizePictures(new[]
                     {
                         Html.ResizeImage(Model.CurrentPage.XLImage).Quality(10),
                         Html.ResizeImage(Model.CurrentPage.LGImage).Quality(20),
                         Html.ResizeImage(Model.CurrentPage.MDImage).Quality(30),
                         Html.ResizeImage(Model.CurrentPage.SMImage).Quality(40),
                         Html.ResizeImage(Model.CurrentPage.XSImage).Quality(50)
                     }, PictureProfiles.BootstrapGrid)
```

### Fallback in Case of /dev/null/

It's not good if you generate empty `<img>` tags. I know it's been patched to avoid sub-sequential web requests in most of the browsers, but still.. Why should I render empty images for my site if I could fallback to some nice looking "no image" placeholder?

This is also supported:

```razor
@Html.ResizePictureWithFallback(Model.CurrentPage.EmptyImage,
                                PictureProfiles.SampleImage,
                                "/noimage.png")
```

## See it in Action!

Code is nice, but think it's much better to understand new possibilities when you see it in action!

<iframe width="846" height="580" src="https://www.youtube.com/embed/PTOYiL1AABk" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>

Happy responsiveness!

[*eof*]
