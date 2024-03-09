---
title: Bootstrap Content Area Renderer Update for Episerver
author: valdis
date: 2020-09-06 09:15:00 +0200
categories: [Add-On, Optimizely, Episerver, .NET, C#, Open Source, DeveloperTools, Bootstrap]
tags: [add-on, optimizely, episerver, .net, c#, open source, bootstrap]
---

Twitter Bootstrap Content Area package hasn't been updated for a while which might mean either no one is using it or it's mature enough that does not require new features :)

Not sure if anyone still use it or should I upgrade to any other CSS framework out there..?! Package is going strong with more than 18k downloads for the last version.

Anyhow, I finally got time to sit down and scan through some of the tickets reported for the package that has been hanging around for a while.

## Registering Renderer in IoC

Interestingly, bug reports were quite odd cases (assuming that Bootstrap renderer is installed and configured properly). Like `BlockIndex` returns always `-1` (report [here](https://github.com/valdisiljuconoks/EPiBootstrapArea/issues/59)).
Also modification of the start element (comes quite handy when you need to anchor content area items with some bookmark and later link to it) of the content area item was not working.

As it turned out, Bootstrap Content Area renderer was not properly registered in IoC container and therefore did not have a chance to plug in and add its own behavior to the rendering pipeline.

Registration of renderer happened like this:

```csharp
[ModuleDependency(typeof(InitializationModule))]
[ModuleDependency(typeof(ServiceContainerInitialization))]
[InitializableModule]
public class SetupBootstrapRenderer : IConfigurableModule
{
    private ServiceConfigurationContext _context;

    void IConfigurableModule.ConfigureContainer(ServiceConfigurationContext context)
    {
        _context = context;
    }

    public void Initialize(InitializationEngine context)
    {
        context.InitComplete += ContextOnInitComplete;
    }

    public void Uninitialize(InitializationEngine context)
    {
        context.InitComplete -= ContextOnInitComplete;
    }

    private void ContextOnInitComplete(object sender, EventArgs eventArgs)
    {
        displayOptions = GetAllDisplayOptions();

        ...
        _context.Services.AddTransient<ContentAreaRenderer>(
            _ => new BootstrapAwareContentAreaRenderer(displayOptions));
        ...
    }
}
```

This is not going to replace `ContentAreaRenderer` in the container. Nor does this code (I'm not sure why tho):

```csharp
_context.Services.RemoveAll<ContentAreaRenderer>();
_context.Services.AddTransient<ContentAreaRenderer>(...);
```

The only way (at least for now) I found -> is to intercept renderer, "silence" it by ignoring given instance and just carry out custom Bootstrap content area renderer logic:

```csharp
private void ContextOnInitComplete(object sender, EventArgs eventArgs)
{
    displayOptions = GetAllDisplayOptions();

    _context.Services.Intercept<ContentAreaRenderer>((_, __) =>
    {
        return new BootstrapAwareContentAreaRenderer(displayOptions);
    });
}
```

## Modify Block Element Fix

Another interesting [issue](https://github.com/valdisiljuconoks/EPiBootstrapArea/issues/60) was that provided way to modify block start element was working [as documented](https://github.com/valdisiljuconoks/EPiBootstrapArea/blob/master/README.md#modify-block-start-element). I would not expect much from the library if one is developed and tested during night shifts :)

Idea for the feature is that you are able to override renderer (by inheriting from built-in) and add your own custom logic how block start element is processed. This was nice feature back in days when we had to generate site menu out of content area blocks - meaning that each block start element should have anchor attribute so that menu items could properly link to page bookmarks. Feature was not working in latest version.

At the end it turned out that again improper IoC registration was done. While registering `ContentAreaRenderer` in container we have to make sure that we check who is sitting there and respect if renderer is somebody from `BootstrapAwareContentAreaRenderer` object family :)

So this is final code for the renderer registration:

```csharp
private void ContextOnInitComplete(object sender, EventArgs eventArgs)
{
    displayOptions = GetAllDisplayOptions();

    // setup proper renderer with all registered fallback (+custom ones as well)
    _context.Services.Intercept<ContentAreaRenderer>((_, render) =>
    {
        // existing renderer is not familiar to us - so we can just "swap it out"
        // this is not the actual swap - as just registering new instance of the renderer does not do the trick
        // we are here "silencing" original - by intercepting it and forgetting about it :)
        if (!(render is BootstrapAwareContentAreaRenderer))
        {
            return new BootstrapAwareContentAreaRenderer(displayOptions);
        }

        // registered renderer is somebody from our family
        // this usually happens when somebody wants to do some extension of the Bootstrap aware renderer
        // therefore inheriting from default one and extending via some virtual methods
        // do after we have collected all display options here - we have to set that to the original renderer
        ((BootstrapAwareContentAreaRenderer)render).SetDisplayOptions(displayOptions);
        return render;
    });
}
```

## None Display Option

Sometimes you would like to set display option that does nothing - none of the CSS classes would be added that could mess up your site design. This was actually requested by somebody who has [JasonRodman](https://github.com/valdisiljuconoks/EPiBootstrapArea/issues/57) identity at GitHub.

For this reason there is a new built-in display option - `None`.

You can add it to the supported `DisplayOptions`:

```csharp
[ModuleDependency(typeof(InitializationModule))]
public class CustomizedRenderingInitialization : IInitializableModule
{
    public void Initialize(InitializationEngine context)
    {
        ConfigurationContext.Setup(ctx =>
        {
            ctx.CustomDisplayOptions
               .Add<DisplayModeFallback.None>();

            ...
        });
    }

    public void Uninitialize(InitializationEngine context) { }
}
```

If you set this display option on the block (in this example `"Teaser Block"` in Alloy sample site) only following classes will be added to the block container element:

```html
<div class="block teaserblock displaymode-none">
    <!-- block content -->
</div>
```

Bootstrap aware Episerver content area renderer package v5.3 includes other smaller fixes also.

Happy bootstrapping!

[*eof*]
