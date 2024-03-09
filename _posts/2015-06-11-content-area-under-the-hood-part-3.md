---
title: Content Area - Under the Hood, Part 3
author: valdis
date: 2015-06-11 08:15:00 +0200
categories: [.NET, C#, Episerver, Optimizely]
tags: [.net, c#, episerver, optimizely, under the hood]
---

This is last blog post in series under the hood of Content Area. In this blog post we will take a look at how and what exactly could be customizable while content area is being rendered. Where and how developer can hook in, what can be overridden and controlled.

Parts in this series:

– [Part 1: Registration and Discovery](https://tech-fellow.eu/2015/04/26/content-area-under-the-hood-part-1/)
– [Part 2: Template Selection](https://tech-fellow.eu/2015/05/30/content-area-under-the-hood-part-2/)
– **Part 3: Extensibility**

## Overview

Following parts and pieces can be customizable, changeable or replaced by other code completely:

* *Change renderer* - It's possible to change they guy who is responsible for whole process in the first place. You can swap content area renderer completely and replace with your own code. You would need to inherit from built-in `ContentAreaRenderer`, swap it out in IoC container and do your stuff;
* *Change display template* - Change view (`ContentArea.ascx`) for the content area type;
* *Customize rendering* - It's possible to add and specify various settings while rendering then content area (`Html.PropertyFor(..., new { }`);
* *Handle events* - As  we got to know in [Part 2](https://tech-fellow.eu/2015/05/30/content-area-under-the-hood-part-2/) `TemplateResolver` fires few events. We are able to hook on those events and customize templates selected as we need.

## Replace Renderer

If something does not work as expected, you need some customization that is not achievable by other methods, want to add some extra stuff on top of existing content area renderer? It's possible to inherit from built-in `ContentAreaRenderer`, replace methods with your own and register new implementation in IoC container.

### Inherit from Built-in Renderer

Inheritance is simple:

```csharp
public class MyOwnContentAreaRenderer : ContentAreaRenderer
{
}
```

These are all methods that make sense to override and change behavior (most probably there is no point to override method that checks if currently CMS context is set to `Edit` mode):

```csharp
void Render(HtmlHelper htmlHelper, ContentArea contentArea)

void RenderContentAreaItems(
            HtmlHelper htmlHelper,
            IEnumerable<ContentAreaItem> contentAreaItems)

void RenderContentAreaItem(
            HtmlHelper htmlHelper,
            ContentAreaItem contentAreaItem,
            string templateTag,
            string htmlTag,
            string cssClass)

void BeforeRenderContentAreaItemStartTag(
            TagBuilder tagBuilder,
            ContentAreaItem contentAreaItem)

TemplateModel ResolveTemplate(
            HtmlHelper htmlHelper,
            IContent content,
            string templateTag)

string GetContentAreaTemplateTag(HtmlHelper htmlHelper)

string GetContentAreaItemTemplateTag(
            HtmlHelper htmlHelper,
            ContentAreaItem contentAreaItem)

string GetContentAreaHtmlTag(
            HtmlHelper htmlHelper,
            ContentArea contentArea)

string GetContentAreaItemHtmlTag(
            HtmlHelper htmlHelper,
            ContentAreaItem contentAreaItem)

string GetContentAreaItemCssClass(
            HtmlHelper htmlHelper,
            ContentAreaItem contentAreaItem)
```

Let's go through the call sequence of each of these methods.

![](/assets/img/2015/06/p3-1.png)

Eventual impact of each method in resulting markup is shown in diagram below:

![](/assets/img/2015/06/p3-2.png)

If you are looking for an easy way to customize resulting markup of `ContentArea`, without overriding almost everything in `ContentAreaRenderer`, head to "*Customize Rendering*" section.

Few observations:

* Why CSS `class=""` for content area itself can't be controlled via `virtual` method in the same way as other customization for `Html` tags and `Css class` for area item?
* Why there is no `AfterRenderContentAreaItemEndTag` method? Where you could hook in and add some necessary stuff *after* content area item has closed its tag element?


### Swap Default Renderer
So once you are ready with your own content area renderer you will need to set it in action. In order to swap out built-in renderer and replace it with your own, all you need to do is to replace one in IoC container:

```csharp
[ModuleDependency(typeof(ServiceContainerInitialization))]
[InitializableModule]
public class SwapRendererInitModule : IConfigurableModule
{
    public void ConfigureContainer(ServiceConfigurationContext context)
    {
        context.Container.Configure(container =>
            container.For<ContentAreaRenderer>()
                     .Use<MyOwnContentAreaRenderer>());
    }

    public void Initialize(InitializationEngine context)
    {
    }

    public void Uninitialize(InitializationEngine context)
    {
    }
}
```

Now everything and anywhere where `ContentArea` property is rendered - you will get full access and control of workflow and can customize it according to site's requirements.

## Change Display Template

If you want to become real hacker and apply your logic to every single content area and also do some black magic with markup and the way how content areas are rendered, you may need to swap out existing built-in template for `ContentArea` by just creating "more specific" display template in your `Shared/DisplayTemplates` folder (or anywhere where Asp.Net can find it):

![](/assets/img/2015/06/p3-3-1.png)

I've seen projects where this was implemented. Fun times to try to find it out.. Custom display template for EPiServer built-in type was the last idea that came to mind.

## Customize Rendering
There are also some extension points on how you can customize content area resulting markup.

Basically you can pass in few known to EPiServer keys in `ViewData` when executing `@Html.PropertyFor()`. For instance:

```csharp
...
@Html.PropertyFor(m => m.ContentArea, new { tag = "some-tag" })
...
```

Following are known keys to EPiServer you can pass in:

* `Tag` - I think one of the most important `ViewData` key while rendering content area. This key regulates what templates may be selected for the content area and potentially for its items. For more info on template selection - please read [Part 2](https://tech-fellow.eu/2015/05/30/content-area-under-the-hood-part-2/);
* `HasContainer` - key indicates whether to generate wrapping element for the content area at all;
* `CustomTag` - wrapping element name for the content area (defaults to "`div`") if `hascontainer = true`. Small observation - a bit confusing and could be mismatched with `tag`;
* `CssClass` - CSS class(-es) to add to content area wrapping element
* `ChildrenCustomTagName` - element name for every content area item (defaults to "`div`"). Small observation - naming for area element name and child element name should be consistent. Let's say: either `customtagname` for content area itself or `childerencustomtag` for item;
* `ChildrenCssClass` - CSS class(-es) name to add to every content area item

So this are list of keys you can pass in:

```csharp
@Html.PropertyFor(m => m.ContentArea, new {
    Tag = "some-tag",
    HasContainer = true,
    CustomTag = "ul",
    CssClass = "this-is-a-list-class",
    ChildrenCustomTagName = "li",
    ChildrenCssClass = "this-is-list-item-class"
})
```

In the results you will get something like this assuming that there are few items added to the content area:

```html
<ul class="this-is-a-list-class">
  <li class="this-is-list-item-class">...</li>
  <li class="this-is-list-item-class">...</li>
  <li class="this-is-list-item-class">...</li>
</ul>
```

Which gives you pretty much flexibility to render content area as you want and need. This reminds me of [Per Magne's](http://world.episerver.com/blogs/Per-Magne-Skuseth/) approach to build [menu driven by content area](http://world.episerver.com/blogs/Per-Magne-Skuseth/Dates/2013/8/Developing-a-drag-and-droppable-menu-MVC/).

## Handle Resolving Events

There 2 type of events that could be handled during content area rendering:

* `TemplateResolving` - from the name of the event it's clear that this event is raised *before* template is resolved;
* `TemplateResolved` - however this event on the other hand is raised after EPiServer has finished template resolving process;

Both of these events are exposed from EPiServer API in order for us developers to hook in, handle and customize selection of the template.

Event handling for the template selection is pretty straight forward:

```csharp
[ModuleDependency(typeof(InitializationModule))]
public class CustomizedRenderingInitialization : IInitializableModule
{
    public void Initialize(InitializationEngine context)
    {
        context.Locate.TemplateResolver().TemplateResolving +=
            TemplateCoordinator.OnTemplateResolving;

        context.Locate.TemplateResolver().TemplateResolved +=
            TemplateCoordinator.OnTemplateResolved;
    }

    public void Uninitialize(InitializationEngine context)
    {
        ServiceLocator.Current.GetInstance<TemplateResolver>().TemplateResolving -=
            TemplateCoordinator.OnTemplateResolving;

        ServiceLocator.Current.GetInstance<TemplateResolver>().TemplateResolved -=
            TemplateCoordinator.OnTemplateResolved;
    }
}

public class TemplateCoordinator
{
    public static void OnTemplateResolved(object sender, TemplateResolverEventArgs args)
    {
        // handle event - you may change selected template by:
        // args.SelectedTemplate = ....;
    }

    public static void OnTemplateResolving(object sender, TemplateResolverEventArgs args)
    {
        // handle event - you may change selected template by:
        // args.SelectedTemplate = ....;
    }
}
```

This code fragment is taken from our old friend AlloyTech sample site. However I haven't dig into on why during `Initialize` service locator is access as `context.Locate`, but in `Uninitialize` we are referring to service locator as `ServiceLocator.Current`. I'm sure there is a reason behind this.

For more info about template model selection and it's internals - please read [Part 2](https://tech-fellow.eu/2015/05/30/content-area-under-the-hood-part-2/) of these series.

## Summary

Overall `ContentArea` was one of the most interesting improvement over EPiServer's 6.x `ComposerBlock` concept. It's really powerful and you may achieve high flexibility and utilize it's potential in various scenarios.
However - I guess that `ContentArea` needs a bit more love when it comes to discovery of magic behind the scene - that was main goal for these series to spread knowledge I gain while research EPiServer libraries.
The most challenging and interesting for me was [Part 2](https://tech-fellow.eu/2015/05/30/content-area-under-the-hood-part-2/) where I recapped how template model is working and how models are selected based on environmental settings. This part I find most unclear for developers (Usually questions are - "I do have a view for the block and I do have a controller for the block. Why my controller is not invoked, but instead view is rendered directly?").

Hope that these series will give you some more insightful look at what's going on when EPiServer scans your template models, when you execute `@Html.PropertyFor(m => m.ContentArea)`, which template model will be chosen and how to customize other stuff for the content area rendering pipeline.

Happy coding!

[*eof*]
