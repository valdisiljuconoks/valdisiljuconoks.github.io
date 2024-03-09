---
title: Content Area - Under the Hood, Part 1
author: valdis
date: 2015-04-26 08:35:00 +0200
categories: [.NET, C#, Episerver, Optimizely]
tags: [.net, c#, episerver, optimizely, under the hood]
---

Lately I was visiting and revisiting content area, code and functionality around this feature in EPiServer and decided to revisit it once again and take a closer look at what’s really inside. So the blog post is not about what content area is, but how it works, how responsible party for rendering and templating is selected when EPiServer needs to render this one of the most powerful content editing feature.

This is a multi-part blog post series about what happens under `ContentArea` property in EPiServer.

Parts in this series:
– **Part 1: Registration and Discovery**
– [Part 2: Template Selection](https://tech-fellow.eu/2015/05/30/content-area-under-the-hood-part-2/)
– [Part 3: Extensibility](https://tech-fellow.eu/2015/06/11/content-area-under-the-hood-part-3/)

## Overview
Everything starts with property definition on content data:

```csharp
[ContentType(GUID = "265F52C3-5E88-4AFE-90F8-9A42819D9EAF")]
public class StartPage : PageData
{
    [Display(GroupName = SystemTabNames.Content, Order = 10, Name = “Content area”)]
    public virtual ContentArea MainContentArea { get; set; }
}
```

From the technical perspective `ContentArea` property is still just an ordinary page property, in this case it’s just yet another `XhtmlString` type of property.
Interesting part about content area is that all the content and some of the settings for items EPiServer is storing as semi html data. For instance this area:

![](/assets/img/2015/05/content-area-100.png)

Is stored as html data:

```
data-contentguid="e51f496b-ebe..."
data-contentlink="6"
data-contentname="sample block 1">{}
data-contentguid="5dd9a357-761...."
data-contentlink="7"
data-contentname="with controller"
data-epi-content-display-option="full">{}
data-contentguid="08399f13-9339-46...."
data-contentlink="10"
data-contentname="with iview"
data-epi-content-display-option="full">{}
```

Interesting part starts with what exactly EPiServer is doing when it’s given a command to render a content area inside content template:

```razor
@Html.PropertyFor(m => m.MainContentArea)
```

## Rendering Pipeline
Let’s take a look at what happens under the hood when EPiServer encounters `PropertyFor(ContentArea)`. Method basically goes through few discovery steps to pick up template to be used for content area rendering. If nothing has been modified and there are no deviations from built-in functionality – eventually method will call `DisplayFor` (from Asp.Net Mvc).

Below is short overview of pipeline what happens during content area rendering:

![](/assets/img/2015/05/content-area-200.png)

Asp.Net Mvc `DisplayFor` finally hits `ContentArea.ascx` file which calls `Html.RenderContentArea()`. `ContentArea.ascx` file is located in `EPiServer.Cms.Shell.UI.zip` file which is part of CMS package and is located in protected modules directory.

There are 2 essential steps to let Asp.Net Mvc hit `ContentArea.ascx`:
a) Special view engine is registered together with EPiServer site that has access to this `.zip` file and knows about content of the file;
b) View engine is registered by `UIInititialization` module;

File `ContentArea.ascx` does not contain much code – immediately passes control over to `ContentAreaRenderer` the guy who is responsible for rest of the process.
`ContentAreaRender` is responsible also for selecting (actually not the selection itself, but at least calling code that will do the stuff) which rendering template will be used for particular content area item. All registered templates are stored in `TemplateModelRepository` and you can get direct access to these templates via this class. We will take a look at who and how is filling up template model repository a bit later in section about scanning and discovery of the renderers.

## Types of Template Renderers
There are few types of template renderers available to render content of area item.

![](/assets/img/2015/05/content-area-300.png)

Available render templates:
a) *markup view* – just an ordinary markup view
b) *IView* – something that implements `IView` interface directly
c) *partial content controller* – type derived from `PartialContentController`

### Asp.Net Mvc View as Render Template
The easiest and fastest way to get content out to page is “register” ordinary Asp.Net Mvc view as rendering template. EPiServer will call view directly passing in content type it is rendering. Registration is set into quotes because it’s not always explicit manual registration. There is also automatic discovery process in place, but we will take a look at that a bit later.
Model of the view must be the same as content type it’s handling a rendering for:

```csharp
@model SampleProject.Models.Cms.Blocks.SampleBlock
…
```

### IView as Render Template
Sometimes it’s not enough with just dumb view template, maybe some code should be executed before actual rendering of the markup, some calculations or anything.
For this reason there is a possibility to register something that implements `IView` interface as render template for the content.
So the render template itself is pretty straight forward (assuming that there is a block of type `SampleBlockWithIView`):

```csharp
public class SampleBlockWithIViewRenderer : IView,
                                            IRenderTemplate
{
    public void Render(ViewContext viewContext, TextWriter writer)
    {
        …
    }
}
```

*NB!* There are issues for passed in `ViewContext` – about what exactly you [will receive](http://world.episerver.com/Modules/Forum/Pages/Thread.aspx?id=114901). But that's minor issue for now.

How to get registered? Wait till discovery and registration section on this blog post.

### BlockController as Render Template
The most Mvc-ish way to handle rendering of the block is to create a controller that is inheriting from `BlockController`

```csharp
public class SampleBlockWithCtrlController : BlockController
{
    public override ActionResult Index(SampleBlockWithCtrl currentBlock)
    {
        …
        return PartialView(…);
    }
}
```

*NB!* However EPiServer is complaining that this implementation of block render template is [very slow](http://world.episerver.com/blogs/Jonas-Bergqvist/Dates/2013/6/Hidden-template-functionality-in-the-MVC-implementation/) and is desired to be avoided.

## Template Scanning and Discovery Process
All discovered templates and renders are stored in template model repository. This section will describe how exactly templates are discovered and then stored in this repository.

![](/assets/img/2015/05/content-area-400.png)

Template model repository is main data source for template selector when `ContentAreaRenderer` needs to find what template to use when rendering the content.

![](/assets/img/2015/05/content-area-500.png)

There are two main responsible parties for filling up the repository:
a) `RenderTemplateScanner`
b) `ViewRegistrator`

### RenderTemplate Scanning Process
`RenderTemplateScanner` gets kicked in discovery process by `ModelSyncInitialization` initialization module at the very beginning of the request processing pipeline.

There are few main responsibilities for `RenderTemplateScanner` process:
a) It looks for everybody that implements marker interface `IRenderTemplate` and registers it. It means that all `PageController`s and `BlockController`s are being registered by this guy;
b) It also looks for `TemplateDescriptorAttribute`s and registers mentioned template model explicitly in the repository;

For instance in this sample code from AlloyTech sample site – will tell EPiServer to register this as render template with various settings (read more in [Part 2](https://tech-fellow.eu/2015/05/28/content-area-under-the-hood-part-2/): about template selection).

```csharp
[TemplateDescriptor(Inherited = true,
                    TemplateTypeCategory = TemplateTypeCategories.MvcController,
                    Tags = new[] { RenderingTags.Preview, RenderingTags.Edit },
                    AvailableWithoutTag = false)]
[VisitorGroupImpersonation]
public class PreviewController : ActionControllerBase, IRenderTemplate
{
    …
}
```

### Asp.Net Mvc View Scanning Process
Second part of the template model discovery process is view registration. It may create template models in the repository (for not previously registered model types) or it may update already existing ones with an newly discovered information about available template(-s).

*NB!* Interesting part about Mvc view template discovery is that it gets kicked in only on first request by `ContentRoute`. I can just speculate that this delay is needed to let Mvc and other moving parts to finish it’s registration and initialization and to discover Mvc views in EPiServer’s initialization module could be pretty early in the pipeline.

So what does `ViewRegistrator` do? It’s responsible for 2 processes:
a) Manual template model registration (see "Manual Registration" below);
b) Convention based template model registration (see "Conventions Based Registration" below);

### Manual Registration

There is possibility to skip all automatic or semi-automatic discovery processes and do manual template model registration. This is supported by `ViewRegistrator` which looks for types implementing `IViewTemplateModelRegistrator` interface and calls `Register` method.

```csharp
[ServiceConfiguration(typeof(IViewTemplateModelRegistrator))]
public class TemplateCoordinator : IViewTemplateModelRegistrator
{
    public void Register(TemplateModelCollection viewTemplateModelRegistrator)
    {
        …
    }
}
```

Then it’s really up to the developer how and which templates are registered.
Here developer can provide every single detail to the template model needed to be registered.
For instance this registration can’t be discovered with any automatic registration and therefore done manually:

```csharp
public void Register(TemplateModelCollection registrator)
{
    registrator.Add(typeof(SampleBlock),
                    new TemplateModel
                    {
                        Tags = new[] { "full-width" },
                        AvailableWithoutTag = false,
                        Default = true,
                        Name = "SampleBlockFullWidth",
                        Path = "~/Views/Shared/Blocks/_SampleBlockFullWidth.cshtml"
                    });
}
```

It’s also possible to register any template type (actual code that will handle rendering) within template model. For instance you can manually register `IView` template as template type:

```csharp
[ServiceConfiguration(typeof(IViewTemplateModelRegistrator))]
public class TemplateCoordinator : IViewTemplateModelRegistrator
{
    public void Register(TemplateModelCollection registrator)
    {
        registrator.Add(
            typeof(SampleBlockWithIView),
            new TemplateModel
            {
                Default = true,
                AvailableWithoutTag = true,
                TemplateTypeCategory = TemplateTypeCategories.MvcPartialView,
                Name = "SampleBlockWithIViewRenderer",
                TemplateType = typeof(SampleBlockWithIViewRenderer)
            });
    }
}
```

Explicit registration gives you more control over how and what template renderer to register comparing to automatic discovery.

### Conventions Based Registration

Another part of `ViewRegistrator` is to discover templates based on default naming conventions.
So for instance when you will have a partial view with the same name as model type that is inside your partial view locations (probably registered by custom `ViewEngine`) it’s registered as template render for this model type.

![](/assets/img/2015/05/content-area-600.png)

## Next Part

In next blog post we will cover how templates are being selected based on various settings inside registered `TemplateModel`.

[*eof*]
