---
title: Row support added to Bootstrap ContentArea
author: valdis
date: 2015-09-20 21:30:00 +0200
categories: [.NET, C#, Episerver, Optimizely, Bootstrap]
tags: [.net, c#, episerver, optimizely, bootstrap]
---

With helpful support from Huilaaja ([@huilaaja](https://twitter.com/Huilaaja)) [Bootstrap](http://getbootstrap.com/) aware `ContentArea` renderer (or in other words - [EPiBootstrapArea](http://nuget.episerver.com/en/OtherPages/Package/?packageId=EPiBootstrapArea)) got row markup support.
Originally row support documentation for ContentArea you can find in author's [blog post](http://blog.huilaaja.net/2015/09/14/episerver-content-area-renderer-with-row-support/).
With author's permission now row support is added to Bootstrap aware content area renderer package.

## Enable Row Support

For backward compatibility and general idea of Bootstrap content area renderer - row support is disabled by default. You need to enable it explicitly.
If you want to enable row markup support "globally" on every content area, you should configure renderer in following way:

```csharp
[ModuleDependency(typeof (SwapRendererInitModule))]
[InitializableModule]
public class SwapBootstrapRendererInitModule : IConfigurableModule
{
    public void ConfigureContainer(ServiceConfigurationContext context)
    {
        context.Container.Configure(container => container
                                        .For<ContentAreaRenderer>()
                                        .Use<BootstrapAwareContentAreaRenderer>()
                                        .SetProperty(i => i.RowSupportEnabled = true));
    }

    public void Initialize(InitializationEngine context) {}

    public void Uninitialize(InitializationEngine context) {}
}
```

Or you can also enable or disable row markup support on individual `ContentArea` property level (request renderer to add row markup for property even it's globally disabled):

```csharp
@Html.PropertyFor(m => m.MainContentArea, new { rowsupport = true })
```

## Disable Row Support

You can disable row markup support with following methods:

* Set configuration property to `false`:

```csharp
[ModuleDependency(typeof (SwapRendererInitModule))]
[InitializableModule]
public class SwapBootstrapRendererInitModule : IConfigurableModule
{
    public void ConfigureContainer(ServiceConfigurationContext context)
    {
        context.Container.Configure(container => container
                                        .For<ContentAreaRenderer>()
                                        .Use<BootstrapAwareContentAreaRenderer>()
                                        .SetProperty(i => i.RowSupportEnabled = false));
    }

    public void Initialize(InitializationEngine context) {}

    public void Uninitialize(InitializationEngine context) {}
}
```

* Do it on property level (even it's enabled globally):

```csharp
@Html.PropertyFor(m => m.MainContentArea, new { rowsupport = false })
```

* Remove custom initializable module - row markup support is disabled by default.

## Row Support Drawback

Beware that adding row markup support either globally or locally for particular area you are avoiding original flexibility that was behind creation of this Bootstrap aware content area renderer. Flexibility of fall back to different width on various screen sizes.
For instance, consider following markup:

```html
<div>
    <div class="row row0">
        <div class="block editorialblock col-lg-4 col-md-6">
            ...
        </div>
        <div class="block editorialblock col-lg-4 col-md-6">
            ...
        </div>
        <div class="block editorialblock col-lg-4 col-md-6">
            ...
        </div>
    </div>
    <div class="row row1">
        <div class="block editorialblock col-lg-6 col-md-12">
            ...
        </div>
        <div class="block editorialblock col-lg-6 col-md-12">
            ...
        </div>
    </div>
</div>
```

Visually on large screen it looks like this:

![](/assets/img/2015/09/2015-09-20_22-46-00.png)

From markup it's totally perfect to wrap both first elements into additional row because each of them occupies only half of the screen. But issue here is with smaller screen sizes - let's say `col-md-*`. It's said that blocks [1-3] should occupy half of the screen size on smaller devices (medium screen size). Then - there is no reason to wrap all 3 blocks inside the same row as on large screens.

![](/assets/img/2015/09/2015-09-20_22-57-09.png)

**NB!** This means that it's not necessary wrap them into single row. Adding row support may end up with fancy and interesting effects. **So be careful!** Portion for block sizes to fill up row is taken from **block's large screen** size (`col-lg-*`).

Happy rowing!

[*eof*]
