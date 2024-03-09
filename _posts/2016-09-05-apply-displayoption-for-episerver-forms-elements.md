---
title: Apply DisplayOptions for EPiServer.Forms Elements
author: valdis
date: 2016-09-05 15:30:00 +0200
categories: [.NET, C#, Episerver, Optimizely]
tags: [.net, c#, episerver, optimizely]
---

Would you like to use `DisplayOptions` feature also for EPiServer.Forms?

![](/assets/img/2016/09/2016-08-31_11-52-33.png)

In this blog post I'm gonna show you how to do that.

## Background

It's really great to see platform evolving and adding new features. But unfortunately - I must say that `EPiServer.Forms` are pretty close for modification (which is good) but at the same time - not so open for extensions, that's why we will need to modify some of the built-in templates to get this done.

## Getting Started

* First of all, we will need to pull down a small package ([EPiBootstrapArea.Forms](http://nuget.episerver.com/en/OtherPages/Package/?packageId=EPiBootstrapArea.Forms)) for this to work.
This is small library that is available as part of [EPiBootstrapArea](https://github.com/valdisiljuconoks/EPiBootstrapArea) project.

* Second, we need to extract one built-in forms element template for modifications. Template is located at `{project-root}\packages\EPiServer.Forms.3.0.0.0\` and then `Content\modules\_protected\EPiServer.Forms\EPiServer.Forms.zip`. And you need to search for `\Views\ElementBlocks\FormContainerBlock.ascx` inside that `.zip` file.

* Then you need to copy over this `.ascx` file to your project's shared components views folder: `{web-project}\Views\Shared\ElementBlocks\` (you most probably will need to create new folder over there).

## Modifying the Template

To get things working we have to modify built-in template of EPiServer. As it seems like straightforward and easy way to customize - this approach has **huge downside**. Upon next EPiServer update - there is no guarantee that built-in templates are not changed from previous installed version. Which also means that upgrade process is "doable", but will probably cost more than usual and you have to keep in mind all the time - to peek into templates and look for changes.

When we are over this mental challenge, we need to add two things to the template:

a) add required `using`:

```aspnet
...
<%@ import namespace="EPiBootstrapArea.Forms" %>
...
```

b) then we need to go to `Ln 110` and change from this:

```aspnet
<%
    Html.RenderFormElements(i, step.Elements);
%>
```

to this:

```aspnet
<%
    Html.RenderFormElements(i, step.Elements, Model);
%>
```

This is extension method coming from `EPiBootstrapArea.Forms` package.

Or alternatively - you can [download this file](https://github.com/valdisiljuconoks/EPiBootstrapArea/blob/master/tests/EPiBootstrapArea.SampleWeb/Views/Shared/ElementBlocks/FormContainerBlock.ascx) from GitHub repo test project.

You may ask - why this is needed? Basically - original code was iterating over forms elements collection and calling content renderer directly in the loop. There is almost no place for customization and replacement (maybe I'm wrong - please comment,if you know better way to avoid these modifications). I'm overriding this method with my own implementation - also passing in `Model` e.g. `FormContainerBlock`.
In this case `Model` is needed because we need to access original `ContentAreaItem` that was used as form element base. There display option - selected by the editor - is available. By knowing which display mode was selected, library can wrap particular form element inside new markup element with proper Bootstrap classes (or even maybe with [custom Css classes](https://github.com/valdisiljuconoks/EPiBootstrapArea#customize-generated-css-classes)).

## Result

By referencing this small new package - as expected - editors are able to use display options also for form elements in on-page edit mode:

![](/assets/img/2016/09/2016-09-02_12-37-35.png)


The following markup for this particular form element will be something like this:

![](/assets/img/2016/09/2016-09-02_12-43-55.png)

If you have any issues, problems or feedback - please, leave it on [GitHub repo](https://github.com/valdisiljuconoks/EPiBootstrapArea/issues).

Happy forming!

[*eof*]
