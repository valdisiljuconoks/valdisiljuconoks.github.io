---
title: Asp.Net Mvc Areas in EPiServer - Part 2
author: valdis
date: 2015-08-10 13:30:00 +0200
categories: [.NET, C#, Episerver, Optimizely]
tags: [.net, c#, episerver, optimizely]
---

Asp.Net Mvc Areas is a great way to organize and structure your EPiServer site. Once you have multi-tenant site where the same EPiServer installation with the same set of libraries, styles, script set and other assets is handling more than single site - Asp.Net Mvc Areas become really handy to group particular site's specific assets and other stuff together under single folder. This also gives developers more obvious and more clear picture of the project setup.

However we all know that there is no *native* support for Asp.Net Mvc Areas inside EPiServer. I've been [trying to add](https://tech-fellow.eu/2015/01/21/full-support-for-asp-net-mvc-areas-in-episerver-7-5/) some basic support for EPiServer installations to utilize Asp.Net Mvc Areas for developers to organize their stuff.
Anyway there is room for improvement. This blog post is about how to customize Asp.Net Mvc Areas usage in EPiServer projects even more or utilize Mvc Areas as EPiServer's sites.

## Mvc Area as Site Discriminators

In order for us to be on the same context, let me enlist requirements for areas and its usage when serving multi-tenant sites:

* There are two sites in the solution - `site1` and `site2`;

![](/assets/img/2015/08/ar-0.png)

* Every site should be implemented as Mvc Area;
* All site related assets should be located inside that area;
* Should be possible to have content type only defined and used in particular site/area;
* Should be possible to have `common content type` defined in root scope (not in particular site/area);
* Should be possible to have different template for the same common content type for each site. So basically as an editor I should be able to create 2 instances of `common block` and use each of the instance in each site;
* Should be possible to define rendering template for common block for each site. So basically when visitors are accessing `site1` - render template from `site1` should be used, and similarly when accessing `site2` - render template from `site2` should be used;
* Should be possible to choose correct layout page while previewing (when editors press "Edit" in content area context menu for particular block) the block instance. It means - if I'm in `site1` and press "Edit" on one of the blocks in content area, EPiServer should pick up correct preview template that may be defined in each site;

![](/assets/img/2015/08/ar-0-11.png)

So following control-flow is needed:

* if request is made for area specific controller - we set context to that area;
* if request is made against controller defined in "root" scope", but there is matching Mvc Area for current EPiServer site, we need to set context to that particular area while executing template rendering;
* otherwise request flows through ordinary Asp.Net processing pipeline conventions;

Hope that requirements for our tiny "multiple-sites-organized-by-areas" project is now more clear. Let's start implementing necessary moving parts.

## Implementing Requirements

### Site Specific Content Type

There are no special requirements for the content type defined inside Mvc Area with renderer, or without it - only with template.
This is already supported out of the box using code snippets from [my earlier attempt](https://tech-fellow.eu/2015/01/21/full-support-for-asp-net-mvc-areas-in-episerver-7-5/) to add support for Mvc Areas inside EPiServer projects.

![](/assets/img/2015/08/ar-0-1.png)


### Content Type with Site Specific Template

Using following code snippets from [my earlier attempts](https://tech-fellow.eu/2015/01/21/full-support-for-asp-net-mvc-areas-in-episerver-7-5/) to add Asp.Net Mvc Areas to EPiServer projects, you can easily define content type in "root" scope or any other project or namespace and have only single template in your particular area - either for `site1` or `site2`.

![](/assets/img/2015/08/ar-1.png)

### Generic Content with Templates for each Site

More common case is when you have `GenericContentType` defined somewhere in "root" scope or any other project/namespace, and you have multiple sites set up as Mvc Areas. You define content type once and would like to have different template for every site.

In this case, when there is no renderer (controller) for the content type, proper context for template selection is already done by hosting page.
So in other words if such content is hosted inside the page that resides in particular Mvc Area, instructions to look for templates inside that area (`RouteData.DataTokens["area"]`) is already set by the page or [action filter specifically](https://github.com/valdisiljuconoks/EPiServer.MvcAreas/blob/master/EPiServer.MvcAreas/DetectAreaAttribute.cs).

![](/assets/img/2015/08/ar-2.png)

### Generic Content with Renderer and Templates for each Site

Another case is when you have common renderer (content type controller) and templates for each site. Visually it looks something like this:

![](/assets/img/2015/08/ar-3.png)

For this particular case thing that we need to do - is to ensure that after content type renderer (controller) has been executed, we need to set context to specific site as Mvc Area in order to Asp.Net view engine collection to find proper views in located in area.
This could be done easily with few lines of code:

```csharp
// if there is an area with the same name of the site - switch to that area
if (AreaTable.Areas.Contains(SiteDefinition.Current.Name))
{
    RouteData.DataTokens["area"] = SiteDefinition.Current.Name;
}
```

So it means that following content type action:

```csharp
public override ActionResult Index(GenericBlockWithCtrl currentBlock)
{
    // if there is an area with the same name of the site - switch to that area
    if (AreaTable.Areas.Contains(SiteDefinition.Current.Name))
    {
        RouteData.DataTokens["area"] = SiteDefinition.Current.Name;
    }

    return PartialView("_GenericBlockWithCtrl", currentBlock);
}
```

will make sure that once action has been executed we instruct view engine collection that current context is in specific area. But do this only if we are currently accessing valid and known EPiServer site.

This also could be refactored into a action filter for more convenient usage:

```csharp
public class GenericBlockWithCtrlController : BlockController<GenericBlockWithCtrl>
{
    [SwitchToArea]
    public override ActionResult Index(GenericBlockWithCtrl currentBlock)
    {
        return PartialView("_GenericBlockWithCtrl", currentBlock);
    }
}

public class SwitchToAreaAttribute : ActionFilterAttribute
{
    public override void OnActionExecuting(ActionExecutingContext filterContext)
    {
        base.OnActionExecuting(filterContext);

        // if there is an area with the same name of the site - switch to that area
        if (AreaTable.Areas.Contains(SiteDefinition.Current.Name))
        {
            filterContext.RouteData.DataTokens["area"] = SiteDefinition.Current.Name;
        }
    }
}
```

## Handling Preview Controller

Preview controller is not exactly what it says. EPiServer uses `Preview` category for its content templates to control who will be responsible for rendering the content when editor will edit the block content.

![](/assets/img/2015/08/ar-4.png)

Preview controller is defined as single instance inside "root" scope. So in order for this guy to detect proper EPiServer site and switch to that Mvc Area - we need to add our newly created filter attribute:

```csharp
[TemplateDescriptor(
    Inherited = true,
    TemplateTypeCategory = TemplateTypeCategories.MvcController, //Required as cont
    Tags = new[] {RenderingTags.Preview, RenderingTags.Edit},
    AvailableWithoutTag = false)]
[VisitorGroupImpersonation]
public class PreviewController : ActionControllerBase, IRenderTemplate<BlockData>,
{
    public PreviewController(...)
    {
        ....
    }

    [SwitchToArea]
    public ActionResult Index(IContent currentContent)
    {
        ....
        return View(model);
    }
```

Also if you want to have multiple preview layout pages - let say different preview layout page for each site (that will definitely set editor in proper look & feel when editing particular site's block) you need to add view templates for preview controller inside your areas:

![](/assets/img/2015/08/ar-5.png)

The way how I prefer to setup layout page for views inside area is by using `_viewstart.cshtml` page:

```csharp
@{
    Layout = "Shared/Layouts/_SiteLayout.cshtml";
}
```

## Summary

Asp.Net Mvc Areas is an ideal way to organized and structure your site by features - you get all your feature related stuff under single folder structure that makes it really easy to find and understand the project.
However - you can also organize EPiServer's project structure in such way that EPiServer site appears as separate Asp.Net Mvc Area - that also gives you a way to organize and structure your sites and projects. Latter needs a bit adjustments to make it work properly.

Source code for Asp.Net Mvc Areas and added support for EPiServer sites can be found in [GitHub](https://github.com/valdisiljuconoks/EPiServer.MvcAreas) as well!

Happy coding in one of your areas!

[*eof*]
