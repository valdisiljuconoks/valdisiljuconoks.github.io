---
title: Responsive EPiServer Tables
author: valdis
date: 2014-03-06 12:35:00 +0200
categories: [.NET, C#, Add-On, Episerver, Optimizely]
tags: [.net, c#, add-on, episerver, optimizely]
---

EPiBootstrapArea
================

Bootstrap aware EPiServer content area renderer. Provides easy way to register display options used to customize look and feel of the blocks inside a EPiServer content area.

## Getting Started

You would need to install package from EPiServer's NuGet [feed](http://nuget.episerver.com/) to start using Twitter's Bootstrap aware EPiServer Content Area renderer:

```
PM> Install-Package EPiBootstrapArea
```

Default built-in display options are regsitered automatically. Below is described some built-in features.

## Available Built-in Display Options

Following display options are registered by default:

* Full width (`displaymode-full`)
* Half width (`displaymode-half`)
* One-third width (`displaymode-one-third`)
* Two-thirds width (`displaymode-two-thirds`)
* One-quarter width (`displaymode-one-quarter`)
* Three-quarter width (`displaymode-three-quarters`)

![](/assets/img/2014/02/display-modes.png)

### Display Option Fallbacks
For every display option there are 4 fallback width for various screen sizes based on Bootstrap grid system. According to Bootstrap v3 [specification](http://getbootstrap.com/css/#grid-options) following screen sizes are defined:

* Large screen (>= 1200px)
* Medium devices (>= 992px && < 1200px)
* Small devices (>= 768px && < 992px)
* Extra small devices (< 768px)

These numbers are added at the end of Bootstrap grid system class (for instance 12 for Large screen -> `'col-lg-12'`)

<table><thead><th>Display Mode Name</th><th>Extra small devices (xs)</th><th>Small devices (sm)</th><th>Medium devices (md)</th><th>Large devices (lg)</th><thead><tbody><tr><td>`Full width`</td><td>12</td><td>12</td><td>12</td><td>12</td></tr><tr><td>`Half width`</td><td>12</td><td>12</td><td>6</td><td>6</td></tr><tr><td>`One third`</td><td>12</td><td>12</td><td>6</td><td>4</td></tr><tr><td>`Two thirds`</td><td>12</td><td>12</td><td>6</td><td>8</td></tr><tr><td>`One quarter`</td><td>12</td><td>12</td><td>6</td><td>3</td></tr><tr><td>`Three quarters`</td><td>12</td><td>12</td><td>6</td><td>9</td></tr></tbody><table>

Eventually if you choose `Half-width` display option for a block of type `EditorialBlockWithHeader` following markup will be generated:

```xml
<div class="block editorialblockwithheader col-lg-6 col-md-6 col-sm-12 col-xs-12 displaymode-half">
    ...
</div>
```

Breakdown of added classes:

* `block` : generic class added to identify a block
* `{block-name}` : name of the block type is added (in this case `EditorialBlockWithHeader`)
* `col-xs-12` : block will occupy whole width of the screen on extra small devices
* `col-sm-12` : block will occupy whole width of the screen on small devices
* `col-md-6` : block will occupy one half of the screen on medium devices
* `col-lg-6` : block will occupy one half of the screen on desktop
* `displaymode-half` : chosen display option name is added

### Example
Let's take a look at `One quarter width` block.
This is a block layout in EPiServer content area on-page edit mode (desktop view - large screen `col-lg-3`):

![](/assets/img/2014/02/one-qrt-1.png)

This is a block layout in EPiServer content area on medium devices - `col-md-6`:

![](/assets/img/2014/02/one-qrt-2.PNG)

This is a block layout in EPiServer content area on small and extra small devices - `col-sm-12` and `col-xs-12`:

![](/assets/img/2014/02/one-qrt-3.png)


## Customize Bootstrap Content Area
In order to customize available display options you need to add new ones through provider model.

### Provider Model
There is a tiny provider model inside this library to control how list of supported display modes is found. By default `DisplayModeFallbackDefaultProvider` provider is registered with following module:

```csharp
[ModuleDependency(typeof(ServiceContainerInitialization))]
[InitializableModule]
public class DisplayModeFallbackProviderInitModule : IConfigurableModule
{
    void IConfigurableModule.ConfigureContainer(ServiceConfigurationContext context)
    {
        context.Container.Configure(x => x.For<IDisplayModeFallbackProvider>()
                                          .Use<DisplayModeFallbackDefaultProvider>());
    }

    public void Initialize(InitializationEngine context)
    {
    }

    public void Uninitialize(InitializationEngine context)
    {
    }
}
```

### Register Custom Provider

You can for instance create new module and register your own new custom provider:

```csharp
[ModuleDependency(typeof(DisplayModeFallbackProviderInitModule))]
[InitializableModule]
public class CustomDisplayModeFallbackProviderInitModule : IConfigurableModule
{
    void IConfigurableModule.ConfigureContainer(ServiceConfigurationContext context)
    {
        context.Container.Configure(x => x.For<IDisplayModeFallbackProvider>()
                                          .Use<DisplayModeFallbackCustomProvider>());
    }

    public void Initialize(InitializationEngine context)
    {
    }

    public void Uninitialize(InitializationEngine context)
    {
    }
}
```

**NB!** In order to run after built-in initializable module you will need to add dependency to it in your module.

```csharp
...
[ModuleDependency(typeof(DisplayModeFallbackProviderInitModule))]
public class CustomDisplayModeFallbackProviderInitModule : IConfigurableModule
{
```

And then in your custom provider you need to specify list of available display modes by overridding `GetAll()` method.

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
            LargeScreenWidth = 12,
            MediumScreenWidth = 12,
            SmallScreenWidth = 12,
            ExtraSmallScreenWidth = 12
        });

        return original;
    }
}
```

There is also backward compatibility with DDS storage. You will need to switch to that provider manually:

```csharp
...
context.Container.Configure(x => x.For<IDisplayModeFallbackProvider>()
                                  .Use<DisplayModeDdsFallbackProvider>());
```

Registered display options will be stored in Dynamic Data Store under `EPiBootstrapArea.DisplayModeFallback` store type. Currently there is no built-in support for editing DisplayOptions on fly from EPiServer UI. For this reason you can choose for instance [Geta.DDSAdmin](https://github.com/Geta/DdsAdmin) plugin.


### Additional Styles
Similar to EPiServer AlloyTech sample site it's possible to define custom styles for block. You have to implement `EPiBootstrapArea.ICustomCssInContentArea` interface.

```csharp
[ContentType(GUID = "EED33EA7-D118-4D3D-BD7F-88C012DFA1F8", GroupName = SystemTabNames.Content)]
public class Divider : BaseBlockData, EPiBootstrapArea.ICustomCssInContentArea
{
    public string ContentAreaCssClass
    {
        get
        {
            return "block-with-round-borders";
        }
    }
}
```

### Localized Display Option Names
You will need to add few localization resource entries in order to get localized DisplayOptions. Following entry has to be added to get localized names for default display options:

```xml
<?xml version="1.0" encoding="utf-8" standalone="yes"?>
<languages>
  <language name="English" id="en">
    <displayoptions>
      <displaymode-full>Full (1/1)</displaymode-full>
      <displaymode-half>Half (1/2)</displaymode-half>
      <displaymode-one-third>One third (1/3)</displaymode-one-third>
      <displaymode-two-thirds>Two thirds (2/3)</displaymode-two-thirds>
      <displaymode-one-quarter>One quarter (1/4)</displaymode-one-quarter>
      <displaymode-three-quarters>Three quarters (3/4)</displaymode-three-quarters>
    </displayoptions>
  </language>
</languages>
```

Happy Bootstrapping!

[*eof*]
