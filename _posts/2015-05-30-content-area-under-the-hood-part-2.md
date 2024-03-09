---
title: Content Area - Under the Hood, Part 2
author: valdis
date: 2015-05-30 08:15:00 +0200
categories: [.NET, C#, Episerver, Optimizely]
tags: [.net, c#, episerver, optimizely, under the hood]
---

This is second post in series of EPiServer Content Area feature. This time we will take a look at how render templates are selected. In the [first part](https://tech-fellow.eu/2015/04/26/content-area-under-the-hood-part-1/) we looked closer on how templates get discovered and registered in Template Model Repository.
Selection of the template is one of the most important fragment of whole rendering pipeline for the content area. This part of the process may turn out to be one of the most non-understandable and kind of black box component (reviewing questions on the EPiServer World forum). But it's crucial to understand what's really going on under the hood. And this blog post is all about that.

![](/assets/img/2015/05/p2-1.png)

Parts in this series:

– [Part 1: Registration and Discovery](https://tech-fellow.eu/2015/04/26/content-area-under-the-hood-part-1/)
– **Part 2: Template Selection**
– [Part 3: Extensibility](https://tech-fellow.eu/2015/06/11/content-area-under-the-hood-part-3/)

## Template Resolver

As we looked of template registration step of the rendering process of content area in previous post, selection of the template to use for the rendering starts in ContentAreaRenderer:

```csharp
ResolveTemplate(htmlHelper, content, templateTag);
```

Which delegates control further down to `ITemplateResolver` (actually `TemplateResolverImplementation`):

```csharp
Resolve(htmlHelper.ViewContext.HttpContext,
        ..,
        ...,
        TemplateTypeCategories.MvcPartial,
        templateTag);
```

Enumeration `TemplateTypeCategories.MvcPartial` actually points to two other enumeration members: `MvcPartialView` and `MvcPartialController`, so EPiServer is looking for template models within either partial view or partial controller categories.


## Handling Events during Template Resolve

`Resolve` method is wrapped around 2 events which are implemented in order for the website code to be able to control a bit how and which template is selected:

- `TemplateResolving`: this event is raised at the beginning of the template selection. If you subscribe to this event, you have a control of which template should be selected before even EPiServer will try to resolve. If you modify event arguments and assign `SelectedTemplate` property, you basically telling EPiServer that it should skip further selection process and use your template. EPiServer will continue template model selection process if `SelectedTemplate` is null after raising the event (basically if there is no event handler or somebody set it to `null` for some reason).

```csharp
[ModuleDependency(typeof(InitializationModule))]
public class CustomizedRenderingInitializationModule : IInitializableModule
{
    public void Initialize(InitializationEngine context)
    {
        context.Locate.TemplateResolver()
            .TemplateResolving += TemplateCoordinator.OnTemplateResolving;
    }

    public void Uninitialize(InitializationEngine context)
    {
        ServiceLocator.Current.GetInstance<TemplateResolver>()
            .TemplateResolving -= TemplateCoordinator.OnTemplateResolving;
    }
}

public class TemplateCoordinator
{
    public static void OnTemplateResolving(object sender, TemplateResolverEventArgs args)
    {
        var repo = ServiceLocator.Current.GetInstance<TemplateModelRepository>();
        args.SelectedTemplate = repo.List(args.ContentType.ModelType).First();  // put actual code here ...
    }
}
```

*NB!* Implementation of event handler for selected template is not ready for production use. Don't copy the code as it may mess around with your templates. Fragment is provided just as code sample :)

- `TemplateResolved`: this is another event raised after selection of the template has ended and EPiServer has done its job. This is the place to override selected template and instruct EPiServer to use other template if it's necessary.

![](/assets/img/2015/05/p2-2.png)

## Template Model
Each discovered and registered template model has following data model that is stored in template model repository. Almost all properties are playing important role in template selection. We will touch role of those properties in sections below.

<table><thead><th>Field Name</th><th>Description</th><thead><tbody><tr><td>Name</td><td>Name of the template model. Gets type name from the content type it's rendering.</td></tr><tr><td>Default</td><td>Flag specifying is this template default.</td></tr><tr><td>Inherit</td><td>Flag indicating whether this model allows inheritance to other child content types.</td></tr><tr><td>TemplateType</td><td>Actual type (class) that may handle the execution of rendering of the template. Not necessarily that each template must have template type.</td></tr><tr><td>TemplateTypeCategory</td><td>What type of template is this? In our case we are interested in `MvcPartialView` and `MvcPartialController`.</td></tr><tr><td>Path</td><td>If path was resolved or set explicitly for the template, it's stored here.</td></tr><tr><td>AvailableWithoutTags</td><td>Flag indicating whether template can also be used without tags. Frequently used together with tagged templates.</td></tr><tr><td>Tags</td><td>List of tags associated with this template model (see "Tagging Templates" section).</td></tr></tbody><table>

## Tagging the Templates
Before proceeding further with template selection it's worth to revisit tagging mechanism for template models.
Main idea behind the tagging was that you may have a single block type, but you would need to customize look & feel of the block in some cases by switching templates. This is easily achievable if you specify `tag` while rendering content area. I've seen quite frequent usage of tags when you may have some shared block and it should look “green with 2 buttons” on one page, and “yellow without buttons” on the other page.
There are two ways how you can get tags attached to the template model:

- by using `TemplateDescriptor` attribute:

```csharp
[TemplateDescriptor(
    Default = true,
    TemplateTypeCategory = TemplateTypeCategories.MvcPartialController,
    Tags = new[] { "SampleTag" },
    ModelType = typeof(SampleBlock),
    AvailableWithoutTag = false)]
public class MyUltraBlockController : ...
{ }
```

- or by manually registering required template with associated tags:

```csharp
[ServiceConfiguration(typeof(IViewTemplateModelRegistrator))]
public class TemplateCoordinator : IViewTemplateModelRegistrator
{
    public void Register(TemplateModelCollection registrator)
    {
        registrator.Add(typeof(SampleBlock),
                                     new TemplateModel
                                     {
                                         Tags = new[] { "full-width" },
                                         AvailableWithoutTag = false,
                                         Default = true,
                                         Name = "SampleBlockFullWidth",
                                     });

```

Then you can render content area with specified `Tag` added to the area:

```razor
@Html.PropertyFor(m => m.MainContentArea, new { Tag = "SampleTag" })
```

I'm thinking of tags something like setting the context for the rendering cycle. Using tags you may get totally different look and / or feel of the same content type - just by choosing different templates to be used while rendering the content.
Tags also are playing important role in template selection process.


## Template Selection Process

After `TemplateResolving` event was raised and no template was selected, EPiServer will continue selection process. Doing two things:

- it will try to fetch template based on the `tag` that was used while rendering the content area (if any were used).
- if no template was found based on tags, EPiServer will continue on default logic for selecting the template for the content type.

The overall process looks like as illustrated in diagram below. We will dig into each of the steps in more details.

![](/assets/img/2015/05/p2-3.png)


Flowchart `Yes` paths are straight forward - if at some point template is selected/found - EPiServer will just use that one without further selection.
Interesting paths and steps are `try to get by tag` and `select template with preferred model`.

### Choosing Template Models by Tag

So what really happens under `TryGetTemplateByTag` method?

![](/assets/img/2015/05/p2-4.png)

There are two important input parameters to this method:

- list of supported templates: supported templates are templates that are registered for particular content type (`ModelType` while registering template model in repository via attribute or view registrator)

- list of all active display channels for current request

Display channels is feature to make it possible to adopt content rendering to specific cases during handling of the request. More info about display channels - [EPiServer SDK Documentation](http://world.episerver.com/documentation/Items/Developers-Guide/EPiServer-CMS/8/Rendering/Display-channels/Display-channels/).

This is the logic of the template selection by `tag`:
**a)** from the list of supported templates ones those have matching `Tag` with any current active channels are selected. If there is no template models marked with `Tag` as any current active channels EPiServer continues with list of supported templates;
**b)** then out of filtered template models by active channel, EPiServer is taking only those template models that have the same `tag` that is used in current content area rendering.
**c)** after the list of possible candidate templates is built, the single model is selected from the list (see "Selecting the Model" section below). Here templates available only within tag are filtered (`AvailableWithoutTags` property of the template model should be set to `false`);


For example if you have following template models registered for particular content type:

<table><thead><th>Id</th><th>Tags</th><th>AvailableWithoutTags</th><th>Default</th><thead><tbody><tr><td>T1</td><td>"SampleTag", "mobile"</td><td>`false`</td><td>`true`</td></tr><tr><td>T2</td><td>"mobile"</td><td>`true`</td><td>`true`</td></tr><tr><td>T3</td><td>"SampleTag"</td><td>`false`</td><td>`true`</td></tr><tr><td>T4</td><td>"SampleTag", "AnotherTag", mobile"</td><td>`true`</td><td>`true`</td></tr><tr><td>T5</td><td>[ ]</td><td>`true`</td><td>`true`</td></tr><tr><td>T6</td><td>"SampleTag", "mobile"</td><td>`false`</td><td>`false`</td></tr></tbody><table>

Assuming that you have `mobile` as active channel during the render and your content area is rendered with `SampleTag` as `Tag`:

```
@Html.PropertyFor(m => m.MainContentArea, new { Tag = "SampleTag" })
```

Choice of templates are following:
**a)** `T1`, `T2`, `T4` and `T6` will be chosen (have matching `Tag` - "mobile");
**b)** `T1`, `T4` and `T6` will be chosen (they have matching `tag` that was used to render content area - "SampleTag");
**c)** `T1` and `T6` are passed further to selection of the model (as they have `AvailableWithoutTag` set to `false`);


### Choosing Template Models with Preferred Model

If template model choice by `Tag` failed (either there was no `tag` used during rendering of the content area or there were no matching templates by tag), EPiServer tries to choose template models from list of supported templates.


![](/assets/img/2015/05/p2-5.png)


Logic behind choosing template models with preferred model is following:
**a)** all template models that have `Tag` the same as any of current active channels are also added to the list of supported templates;
**b)** if there is no template that match any current active display channel and `ContentType` has defined default Mvc view or default Mvc controller - one is chosen (default template has to be `MvcPartialView` or `MvcPartialController`). Default Mvc view or controller you can define here:


![](/assets/img/2015/05/p2-6.png)


**c)** if there is no templates matched active display channel, or content type has no default Mvc view or controller defined, EPiServer continues with selecting the model out of supported template list (see "Selecting the Model" section below). Here also templates that are available without tags are considered and taken into account (`AvailableWithoutTags` property of the template model is set to `true`);


### Selecting the Model

After EPiServer has built a list of templates, selection of the one and only model is made.

Selection of the template is fairly easy task for the EPiServer. Logic behind of selection is as following:
**a)** list of chosen templates is sorted by `Category` key. Which means looking at `TemplatesTypeCategories` enumeration - partial controllers will be on top followed by partial views;
**b)** Template marked as default (`Default = true`) and not inherited (`IsInherited() = false`) is selected;
**c)** If there is no such template (default and not inherited), just default template is selected instead;
**d)** if there is no default template defined - first template from the ordered list of chosen templates is selected.

For example, if we continue with our sample template model list, `T1` and `T6` were passed to selection logic:

<table><thead><th>Id</th><th>Tags</th><th>AvailableWithoutTags</th><th>Default</th><thead><tbody><tr><td>T1</td><td>"SampleTag", "mobile"</td><td>`false`</td><td>`true`</td></tr><tr><td>T6</td><td>"SampleTag", "mobile"</td><td>`false`</td><td>`false`</td></tr></tbody><table>

So assuming that none of them are inherited, the final selected model will be `T1` (as it's default template for this content type - `Default = true`).

## What happens after template is selected?

After EPiServer has selected the template model to use for rendering the content area item, there still happens small black magic here and there. Control flows further to `IContentRenderer`.


![](/assets/img/2015/05/p2-7.png)


Content renderer first decides what type of template model it is - partial view or controller. If template is partial view - view is rendered, if partial controller - child action for the controller is invoked via Asp.Net Mvc standard request handling pipeline (`TemplateType` points to actual controller type to create).

Interesting part is how EPiServer handles rendering of the template if it's of category partial view. EPiServer heavily relies on `ViewEngineCollection` to find particular template on the disk and execute it (by consuming `ViewEngineCollection.FindPartialView` method via its own cached view engine collection).

Actually there is simple logic underneath:
**a)** First, EPiServer anyway will try to find view with name "`Model.Name`" + "`.`" + "`Tag`";
**b)** If this fails, EPiServer will try to find view with the name of the template (`TemplateModel.Name`);
**c)** If that fails, EPiServer will try to find a view by `TemplateModel.Path` property value (if set any).

So we can illustrate this process with new diagram:


![](/assets/img/2015/05/p2-8.png)


*NB!* However there is a small catch with step *a)*.
Assume that you have following template model registered:

```csharp
viewTemplateModelRegistrator.Add(typeof(SampleBlock),
     new TemplateModel
     {
         Tags = new[] { "mobile" },
         AvailableWithoutTag = false,
         Default = true,
         Name = "SampleBlockMobile",
         Path = "~/Views/Shared/Blocks/_SampleBlockMobile.cshtml"
     });
```

Assume that content area was rendered with custom tag:

```razor
@Html.PropertyFor(m => m.MainContentArea, new { Tag = "supertag" })
```

Assume that we do have an active display channel named "mobile" while rendering the content area. So form all choice and selection logic described above, template is matched and recognized as good enough.
As there are a lot of moving parts during template selection, this may get a bit tricky. Let’s assume that you have following file in your partial views location:

```
"~/Views/Shared/Blocks/_SampleBlockMobile.supertag.cshtml"
```

I'll leave it as your homework to decide whether "_SampleBlockMobile.cshtml" or "_SampleBlockMobile.supertag.cshtml" will be selected as the one and only template :)

## Whole Process

So in total eventually the whole process of template selection looks like this:


![](/assets/img/2015/05/p2-9.png)


## Verdict

Content type template model [discovery and registration](http://tech-fellow.eu/2015/04/26/content-area-under-the-hood-part-1/) is pretty easy task for EPiServer. However choice, filtering the templates and selecting the one may turn out to be tricky component to deal with during EPiServer site implementation.

If you get unsure if template got registered, review properties of the template model inside template model repository, browse the content of the repository or want to understand more, I really recommend [Developers Tools from EPiServer](https://github.com/episerver/DeveloperTools). Under section `Templates` there is a section with all discovered and registered templates for any content type. It’s fantastic tool!


## Next Part

In next blog post we will cover various extensibility points added to the pipeline where you can hook in and add your stuff.

Happy rendering!

[*eof*]
