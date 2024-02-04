---
title: Alternative Localization for Asp.Net Core Applications
author: valdis
date: 2018-01-27 23:30:00 +0200
categories: [Add-On, .NET, C#, Localization Provider]
tags: [add-on, .net, c#, open source, localization]
---

This is code fragment from [official documentation](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/localization) how to localize content using built-in functionality.

### App Content Localization

```csharp
[Route("api/[controller]")]
public class AboutController : Controller
{
    private readonly IStringLocalizer<AboutController> _localizer;

    public AboutController(IStringLocalizer<AboutController> localizer)
    {
        _localizer = localizer;
    }

    [HttpGet]
    public string Get()
    {
        return _localizer["About Title"];
    }
}
```

And if you are working with Html content that shouldn't be escaped during rendering - you are using `IHtmlLocalizer` implementation that returns `LocalizedHtmlString` instance.

```csharp
public class BookController : Controller
{
    private readonly IHtmlLocalizer<BookController> _localizer;

    public BookController(IHtmlLocalizer<BookController> localizer)
    {
        _localizer = localizer;
    }

    public IActionResult Hello(string name)
    {
        ViewData["Message"] = _localizer["<b>Hello</b><i> {0}</i>", name];

        return View();
    }
}
```

### View Localization

For the view localization - there is another injectable interface `IViewLocalizer`.

```razor
@inject IViewLocalizer Localizer

@{
    ViewData["Title"] = Localizer["About"];
}
```

## Alternative: Strongly-Typed DbLocalizationProvider

Where is my problem with built-in providers? They all are "stringly-typed". You have to provide string as either key or translation of the resource. I'm somehow more confident strongly-typed approach where I can use "Find All Usages", "Rename" or do any other static code operation that's would not be entirely possible in built-in approach.

Over the time I've been busy developing alternative localization provider for Asp.Net and Episerver (it's brilliant content management system) platforms specifically.

Thought getting that over to Asp.Net Core should not be hard. And it wasn't. So here we are - [DbLocalizationProvider](https://github.com/valdisiljuconoks/LocalizationProvider) for [Asp.Net Core](https://docs.microsoft.com/en-us/aspnet/core/).

### Getting Started

There are couple of things to setup first, before you will be able to start using strongly-typed localization provider.

First, you need to install the package (it will pull down other dependencies also).

```
PM> Install-Package LocalizationProvider.AspNetCore
```

Second you need to setup/configure services.
In your `Startup.cs` class you need to stuff related to Mvc localization (to get required services into DI container - service collection).

And then `services.AddDbLocalizationProvider()`. You can pass in configuration settings class and setup provider's behavior.

```csharp
public class Startup
{
    public Startup(IConfiguration configuration)
    {
        Configuration = configuration;
    }

    public void ConfigureServices(IServiceCollection services)
    {
        services.AddLocalization();

        services.AddMvc()
                .AddViewLocalization()
                .AddDataAnnotationsLocalization();

        services.AddDbLocalizationProvider(cfg =>
        {
            cfg...
        });
    }
}
```

After then you will need to make sure that you start using the provider:

```csharp
public class Startup
{
    public Startup(IConfiguration configuration)
    {
        Configuration = configuration;
    }

    public void ConfigureServices(IServiceCollection services)
    {
        ...
    }

    public void Configure(IApplicationBuilder app, IHostingEnvironment env)
    {
        ...

        app.UseDbLocalizationProvider();
    }
}
```

Using localization provider will make sure that resources are discovered and registered in the database (if this process will not be disabled via `AddDbLocalizationProvider()` method).

### App Content Localization

Localizing application content via `IStringLocalizer<T>` is similar as that would be done for regular Asp.Net applications.

You have to define resource container type:

```csharp
[LocalizedResource]
public class SampleResources
{
    public string PageHeader => "This is page header";
}
```

Then you can demand `IStringLocalizer<T>` is any place you need that one (f.ex. in controller):

```csharp
public class HomeController : Controller
{
    private readonly IStringLocalizer<SampleResources> _localizer;

    public HomeController(IStringLocalizer<SampleResources> localizer)
    {
        _localizer = localizer;
    }

    public IActionResult Index()
    {
        var smth = _localizer.GetString(r => r.PageHeader);
        return View();
    }
}
```

As you can see - you are able to use nice strongly-typed access to the resource type: `_localizer.GetString(r => r.PageHeader);`.

Even if you demanded strongly-typed localizer with specified container type `T`, it's possible to use also general/shared static resources:

```csharp
[LocalizedResource]
public class SampleResources
{
    public static string SomeCommonText => "Hello World!";
    public string PageHeader => "This is page header";
}

public class HomeController : Controller
{
    private readonly IStringLocalizer<SampleResources> _localizer;

    public HomeController(IStringLocalizer<SampleResources> localizer)
    {
        _localizer = localizer;
    }

    public IActionResult Index()
    {
        var smth = _localizer.GetString(() => SampleResources.SomeCommonText);
        return View();
    }
}
```

### View Localization

Regarding the views, story here is exactly the same - all built-in approach is supported:

```
@model UserViewModel
@inject IViewLocalizer Localizer
@inject IHtmlLocalizer<SampleResources> HtmlLocalizer

@Localizer.GetString(() => SampleResources.SomeCommonText)
@HtmlLocalizer.GetString(r => r.PageHeader)
```

### Data Annotations

Supported. Sample:

```csharp
[LocalizedModel]
public class UserViewModel
{
    [Display(Name = "User name:")]
    [Required(ErrorMessage = "Name of the user is required!")]
    public string UserName { get; set; }

    [Display(Name = "Password:")]
    [Required(ErrorMessage = "Password is kinda required :)")]
    public string Password { get; set; }
}
```

View.cshtml:

```razor
@model UserViewModel

<form asp-controller="Home" asp-action="Index" method="post">
    <div>
        <label asp-for="UserName"></label>
        <input asp-for="UserName"/>
        <span asp-validation-for="UserName"></span>
    </div>
    <div>
        <label asp-for="Password"></label>
        <input asp-for="Password" type="password"/>
        <span asp-validation-for="Password"></span>
    </div>
    ...
</form>

```

### Localization in Libraries

You can either rely on `IStringLocalizer` implementation that's coming from `Microsoft.Extensions.Localization` namespace and demand that one in your injections:

```csharp
using Microsoft.Extensions.Localization;

public class MyService
{
    public MyService(IStringLocalizer localizer)
    {
       ...
    }
}
```

Or you can also depend on `LocalizationProvider` class defined in `DbLocalizationProvider` namespace:


```csharp
using DbLocalizationProvider;

public class MyService
{
    public MyService(LocalizationProvider provider)
    {
       ...
    }
}
```

Both of these types provide similar functionality in terms how to retrieve localized content.

### Changing Culture

Sometimes you need to get translation for other language and not primary UI one.
This is possible either via built-in method:

```razor
@inject IHtmlLocalizer<SampleResources> Localizer

Localizer.WithCulture(new CultureInfo("no"))
         .GetString(() => SampleResources.SomeCommonText)
```

Or via additional extension method:

```razor
@inject IHtmlLocalizer<SampleResources> Localizer

Localizer.GetStringByCulture(() => SampleResources.SomeCommonText, new Culture("no"))
```

### Stringly-Typed Localization

For backward compatibility or even if you wanna go hardcore and supply resource keys manually (for reasons) stingly-typed interface is also supported:

```csharp
using Microsoft.Extensions.Localization;

public class MyService
{
    public MyService(IStringLocalizer localizer)
    {
       var header = localizer["MyProject.Resources.Header"];
    }
}
```

## Get More Info
If you want to know more about `DbLocalizationProvider` features and possibilities - good starting point would be [GitHub docs](https://github.com/valdisiljuconoks/LocalizationProvider/blob/master/README.md).

If you do have any issues or trouble setting up, please file an issue in [GitHub](https://github.com/valdisiljuconoks/LocalizationProvider/issues).

<br/>
Happy localizing!<br/>
[*eof*]
