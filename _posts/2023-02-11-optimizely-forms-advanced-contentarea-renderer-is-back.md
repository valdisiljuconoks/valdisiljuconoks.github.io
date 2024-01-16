---
title: Optimizely Advanced ContentArea Renderer for Forms is Back!
author: valdis
date: 2023-02-11 17:00:00 +0200
categories: [Add-On, Optimizely, Episerver, .NET, C#, Open Source]
tags: [Add-On, Optimizely, Episerver, .NET, C#, Open Source]
---

Advanced Content Area Renderer is now available for Optimizely content. However, it would be nice to apply display options to Optimizely Forms elements as well.

![](/assets/img/2023/02/acar-forms-1.png)

## Background

`Optimizely.Forms` are more or less closed for modification and extensions. Templates are bundled together with the package and copied into your `modules/` folder.
To get Advanced Content Area Renderer working together `Optimizely.Forms` we will need to modify forms container element view template. We have to change just a single line, but still - this is something that needs to be kept in mind when upgrading forms packages again.

## Getting Started

* First of all, we will need to pull down `AdvancedContentArea` package for `Optimizely.Forms`.

```
> dotnet add package TechFellow.Optimizely.AdvancedContentArea.Forms
```

This is a small library that is available as part of [AdvancedContentArea](https://github.com/valdisiljuconoks/optimizely-advanced-contentarea) project.

* Second, extract built-in forms container block element template for modification. Template is located at `{web-project-root}\modules\_protected\EPiServer.Forms\EPiServer.Forms.zip`. Look for `\Views\ElementBlocks\Components\FormContainerBlock\FormContainerBlock.cshtml` file inside this `.zip` file.

* Copy this `.cshtml` file to your web project's shared components views folder: `{web-project}\Views\Shared\ElementBlocks\Components\FormContainerBlock` (create a folder if it does not exist).

## Modify the Template

To get things working we have to modify built-in template for `Optimizely.Forms` container element. As it seems like straightforward and easy way to customize - this approach has **huge downside** - you have to keep in mind that updating `Optimizely.Forms` pacakge, template might be changed and your copied / modified template might not be compatible anymore with new `Optimizely.Forms` version. In such case - just copy over again new version of the template and do the same modifications as before.

Now we need to add two things to the template:

a) add required `using` (`Ln 18`):

```razor
...
@using TechFellow.Optimizely.AdvancedContentArea.Forms
...
```

b) then we need to go to `Ln 98` and change from this:

```razor
Html.RenderElementsInStep(step.i, step.value.Elements);
```

to this:

```razor
Html.RenderFormElements(step.i, step.value.Elements, Model);
```

![](/assets/img/2023/02/acar-forms-4.png)


This is extension method coming from `TechFellow.Optimizely.AdvancedContentArea.Forms` package.

Or alternatively - you can [download this file](https://github.com/valdisiljuconoks/optimizely-advanced-contentarea/blob/master/samples/AlloySampleSite/Views/Shared/ElementBlocks/Components/FormContainerBlock/FormContainerBlock.cshtml) from GitHub repo test project.

You may ask - why this is needed? Basically - original code was iterating over forms elements collection and calling content renderer directly in the loop. There is almost no place for customization and replacement (maybe I'm wrong - please comment, if you know better way to avoid these modifications). I'm overriding this method with my own implementation - also passing in `Model` e.g. `FormContainerBlock`.
In this case `Model` is needed because we need to access original `ContentAreaItem` that was used as form element base to get selected display options by the editor. By knowing which display option was selected, library can wrap form element in new markup element with configured classes.

## Result

By referencing this small package - editors now are able to use display options also for Optimizely form elements on-page edit mode:

![](/assets/img/2023/02/acar-forms-1-1.png)

The following markup for this particular form element will be applied according to `DisplayOptions` settings:

![](/assets/img/2023/02/acar-forms-5.png)

If you have any issues, problems, or feedback - please, leave it on [GitHub repo](https://github.com/valdisiljuconoks/optimizely-advanced-contentarea/issues).
<br/>

Happy forming!

[*eof*]
