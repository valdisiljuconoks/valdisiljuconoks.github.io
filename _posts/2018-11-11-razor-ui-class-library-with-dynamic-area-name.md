---
title: Razor UI Class Library with Dynamic Area Name
author: valdis
date: 2018-11-11 00:30:00 +0200
categories: [Add-On, .NET, C#, Episerver, Optimizely, Localization Provider, ASP.NET]
tags: [add-on, .net, c#, open source, episerver, optimizely, localization, asp.net]
---

## New Packaging Process

In general when working with packages that contains artifacts (like client side resources, scripts and what not) I'm a big fan of single-file packaging. Which means - stuff required by package is "packed" together with primary output. Essentially resulting in simpler installation, upgrade or sometimes even removal process.

Earlier (in Asp.Net MVC times) I was messing around with embedding files together with primary assembly and then later during runtime convincing [Owin FileServer](https://www.nuget.org/packages/Microsoft.Owin.StaticFiles/) to serve my files as part of the hosting application. It was doable (take a [look at DbLocalizationProvider](https://github.com/valdisiljuconoks/localization-provider-aspnet/blob/master/src/DbLocalizationProvider.AdminUI/AppBuilderExtensions.cs#L18) AdminUI for Asp.Net MVC apps). Despite that it's achievable, still it felt a little bit of hacking here and there.

## Razor Class Libraries

As I had to move forward with .Net Core implementation for the localization provider, this was a great chance to jump straight to `netcore21` and try new feature - [Razor Class Library](https://docs.microsoft.com/en-us/aspnet/core/razor-pages/ui-class?view=aspnetcore-2.1&tabs=visual-studio).

So, everything start with new project:

![2018-11-09_13-26-20](/assets/img/2018/11/2018-11-09_13-26-20.png)

What is changes is who is the main responsible party for the compilation process of the project (replace `Microsoft.NET.Sdk` with `Microsoft.NET.Sdk.Razor`):

```xml
<Project Sdk="Microsoft.NET.Sdk.Razor">
```

Once the project is setup and ready, you can add new Razor Page to it:

![2018-11-09_13-38-27](/assets/img/2018/11/2018-11-09_13-38-27.png)

If you peek under the hood, firstly Razor markup has new directive `@page`.
Also it generates new `.cshtml.cs` file which contains definition of the page model. This is typical C# class inheriting from `Microsoft.AspNetCore.Mvc.RazorPages.PageModel` type. Base class gives you much shortcuts and access to various environmental types - like `Request`. You will find much similarities with `Controller` from Asp.Net MVC days. It also somehow reminds a bit WebForms code-behind classes (oh, this was retro). There are couple of [auto-wired handler methods](https://docs.microsoft.com/en-us/aspnet/core/razor-pages/?view=aspnetcore-2.1&tabs=visual-studio#writing-a-basic-form) that you can implement and those are guaranteed to be called during page life-cycle.

## Routing to Razor Page

Now, when class library has bunch of of these Razor pages, those got compiled into `*.Views.dll` file. This extra separate file contains your Razor page compiled version. It's much very similar of what `BuildMvcViews` did in Asp.Net MVC context.

Sneak peek in `*.Views.dll` file:

![2018-11-09_13-51-34](/assets/img/2018/11/2018-11-09_13-51-34.png)

Idea how get access to the Razor page from hosting project is via its "areas" (don't mix with [Asp.Net MVC areas](https://docs.microsoft.com/en-us/aspnet/mvc/videos/mvc-2/how-do-i/aspnet-mvc-2-areas) - those we used to organize code into some sort of feature folders back then).

This said, having file organization and area names like these, we can access Razor page via exactly the same url under which page was stored in class library.

![2018-11-09_18-10-51](/assets/img/2018/11/2018-11-09_18-10-51.png)

## Problem with Routing

Issue using Razor pages is that area name is more or less hard-coded (chosen by package owner) and should be used when accessing the pages in hosting project. This is no issue if package owner and hosting project "agree" on path to be used to access pages.
However, for example in localization provider Admin UI package my idea is to allow completely dynamic path to the administrative user interface. Hosting project owner can choose what name will be used in url:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddDbLocalizationProviderAdminUI(_ =>
    {
        _.RootUrl = "/localization-admin";
        ...
    });
}
```

Configuration above will make sure that DbLocalizationProvider AdminUI is "mounted" to url "`https://{something-something}/localization-admin`". Of course I could choose path to hard-code this url and always "mount" to that url (which is actually by default if you don't specify anything), but thought that if [Asp.Net MVC package](https://www.nuget.org/packages/LocalizationProvider.AdminUI/) or [EPiServer counterpart](https://nuget.episerver.com/package/?id=DbLocalizationProvider.AdminUI.EPiServer) allows this configuration, why .Net Core package would be exclusion?

But, as you can image - we somehow have to replace "static" hard-coded path with configured one during the runtime.

## Dynamic Area Name for Razor Pages

After couple hours investigation and research (it's pretty amazing when underlying framework is open source) found a solution for our issue here.

First, we need to give area name anyways. This could be anything, any junk. I just picked some random generated GUID for the area name.

![2018-11-09_18-26-15](/assets/img/2018/11/2018-11-09_18-26-15.png)

Second, we need to play around a bit with Mvc conventions and reconfigure (or add new options to be precise) our area name to route to other (configured path).

```csharp
public static IServiceCollection AddDbLocalizationProviderAdminUI(
    this IServiceCollection services,
    Action<UiConfigurationContext> setup = null)
{
    // add support for admin ui razor class library pages
    services.Configure<RazorPagesOptions>(_ =>
    {
        _.Conventions.AddAreaPageRoute("4D5A2189D188417485BF6C70546D34A1",
                                        "/AdminUI",
                                        UiConfigurationContext.Current.RootUrl);
    });

    return services;
}
```

This is also one of the reasons why path to the AdminUI should be configured in `ConfigureServices` stage and not `Configure` stage in `Startup.cs` file. If we would be adding `RazorPageOptions` after services are configured - it will have not effects on Asp.Net Mvc pipeline what so ever.

Code above basically "remaps" all requests that will be sent to `UiConfigurationContext.Current.RootUrl` (which by default is `localization-admin`) to our randomly generated area.

Now we are able to customize address of the AdminUI to whatever we want.

![2018-11-09_18-36-23](/assets/img/2018/11/2018-11-09_18-36-23.png)

Happy routing!

[*eof*]
