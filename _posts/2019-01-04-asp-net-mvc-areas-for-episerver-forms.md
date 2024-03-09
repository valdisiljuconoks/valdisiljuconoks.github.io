---
title: ASP.NET Mvc Areas for EPiServer Forms
author: valdis
date: 2019-01-04 14:30:00 +0200
categories: [Add-On, .NET, C#, Azure, Episerver, Optimizely]
tags: [add-on, .net, c#, open source, azure, episerver, optimizely]
---

## Forms & Mvc Areas

According to [Episerver documentation](https://world.episerver.com/documentation/developer-guides/forms/creating-a-custom-form-block/) it's possible to override templates used for form elements by creating appropriate views under `~/Views/Shared/ElementBlocks` folder (or any other folder if you have customized location of the form elements views).
Or you can also provide additional folders for Episerver Forms library to look into by registering new `ICustomViewLocation` instance(-s). You can even control order of the locations:

```csharp
[ServiceConfiguration(typeof(ICustomViewLocation))]
public class MyFormsViewLocation : CustomViewLocationBase
{
    public override int Order { get => 1; set { } }

    public override string[] Paths => new[]
                                      {
                                          "..."
                                      };
}
```

Apparently there are folks out there that still are using Asp.Net Mvc Areas to organize project structure (despite that there is also an alternative ["feature folders"](http://marisks.net/2017/12/17/better-feature-folders/)). Most probably because of possibility to have folders as discriminators for sites or built in support from Visual Studio to work with areas:

![2019-01-03_11-34-33](/assets/img/2019/01/2019-01-03_11-34-33.png)

If you have other reasons why you are using Asp.Net Mvc Areas for your Episerver project - let me know, I would like to hear your story!

However using Mvc Areas and Episerver Forms - this approach does not work out well. For supporting Mvc Areas you will need to drop support using tiny [NuGet package](https://nuget.episerver.com/package/?id=MvcAreasForEPiServer.Forms) I just ducttaped together.

## Forms Elements Customization Test Drive

I'm playing around with forms element customization options in my sandbox test project. Would like to walk you through to get a picture of customizations available for Episerver forms elements.
Currently site is built with following features:

* it's based on AlloyTech sample project
* project handles 2 separate sites (distinguished by domain) - and each site has its own Mvc area (`Site1` and `Site2`)
* project also has ordinary page with controller in area with name `Area1`
* and there are bunch of standard AlloyTech pages as well (those views are in root folder)

Sandbox project customizes `TextboxElementBlock` element for each of these areas. We have following overrides:

* common/root text box element for the forms has <span style="background-color:green">green</span> background
* text box element located in `Area1` has <span style="background-color:pink">pink</span> background
* element for `Site1` has <span style="background-color:purple">purple</span> background
* element for `Site2` has <span style="background-color:yellow">yellow</span> background

In project system it looks something like this:

![2019-01-03_10-50-03](/assets/img/2019/01/2019-01-03_10-50-03.png)

### Common Form Element

When you are having a form in ordinary page which is located in root folders, Episerver will pick up template that is located under `Views/` folder.

![2019-01-03_10-56-53](/assets/img/2019/01/2019-01-03_10-56-53.png)

### Form Element in Page located in Area

When you are using Mvc Areas in your project, it's possible to configure infrastructure to support Mvc Areas for ordinary Episerver pages (area will be detected by location of the controller for the page). If you have a form on the page that is located in area (this time in `Area1`) rendering of the form elements should be based on templates that are found in that area (or fallback on common/root templates).

![2019-01-03_11-00-35](/assets/img/2019/01/2019-01-03_11-00-35.png)

### Form Element in Different Sites

And exactly the same support is available with you are hosting multiple sites on your Episerver installation and have decided to split your project into Mvc Areas each for separate site. And having templates for form elements there - Episerver should pick the right ones (those psychedelic colors are just there for me to distinguish between different areas).

![2019-01-03_11-03-39](/assets/img/2019/01/2019-01-03_11-03-39.png)

## Where to get it?

Asp.Net Mvc Areas support for Episerver forms is packed and released as ["MvcAreasForEPiServer.Forms"](https://nuget.episerver.com/package/?id=MvcAreasForEPiServer.Forms) NuGet package.

## Known Issues

Nuget package references `EPiServer.Forms` package version 4.12.0 (lowest that targets Episerver v11). If you install newer `EPiServer.Forms` package in your project, it does not add assembly version binding redirects. So you might run into an issue that runtime is not able to find appropriate versions of some of the .dll files. If so, then you need to add couple of redirects manually (here assumption is that 4.21.0 version is installed in your project):

```xml
<configuration>
  <runtime>
    <assemblyBinding>
      <dependentAssembly>
        <assemblyIdentity name="EPiServer.Forms" publicKeyToken="8fe83dea738b45b7" culture="neutral" />
        <bindingRedirect oldVersion="0.0.0.0-4.21.0.0" newVersion="4.21.0.0" />
      </dependentAssembly>
        <assemblyIdentity name="EPiServer.Forms.Core" publicKeyToken="8fe83dea738b45b7" culture="neutral" />
        <bindingRedirect oldVersion="0.0.0.0-4.21.0.0" newVersion="4.21.0.0" />
      </dependentAssembly>
        <assemblyIdentity name="EPiServer.Forms.UI" publicKeyToken="8fe83dea738b45b7" culture="neutral" />
        <bindingRedirect oldVersion="0.0.0.0-4.21.0.0" newVersion="4.21.0.0" />
      </dependentAssembly>
    </assemblyBinding>
  </runtime>
</configuration>
```

## Source Code

As usual source code for the package could be found on GitHub ([https://github.com/valdisiljuconoks/MvcAreasForEPiServer](https://github.com/valdisiljuconoks/MvcAreasForEPiServer)).


Happy forming into areas! :)
[*eof*]
