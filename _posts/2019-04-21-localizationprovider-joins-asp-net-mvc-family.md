---
title: LocalizationProvider Client Side Feature now also for Asp.Net Mvc Apps
author: valdis
date: 2019-04-21 18:00:00 +0200
categories: [Add-On, .NET, ASP.NET, C#, Open Source, Localization Provider]
tags: [add-on, .net, asp.net, c#, open source, localization provider, localization]
---

There are sometimes moments when you just need to take deep inhale and add backward support for apps that most probably you will hardly see selected in "File > New Project". We do have some projects still based on "pure" Asp.Net Mvc that need client-side localization. Therefore adding support for this type of apps sounds like feature with "must have" label. At least for now. This applies to v5.5.1 and forward.

However, if you are targeting .NET Core - support is [already there](https://tech-fellow.eu/2019/03/09/client-side-localization-in-asp-net-core-using-dblocalizationprovider/) and you are good to go!

## Getting Started

In order to start localizing your applications you will need to install Asp.Net Mvc localization provider client-side support package:

```powershell
 Install-Package LocalizationProvider.JsResourceHandler
```

This package will drag down couple of dependencies as well, but believe me - those are required in order to run the library :)

Next is to register client-side resource provider within your app. This is done in `Startup.cs` file:

```csharp
public class Startup
{
    public void Configuration(IAppBuilder app)
    {
        // setup of the core functionality
        app.UseDbLocalizationProvider(...);

        // register client-side support
        app.UseDbLocalizationProviderJsHandler();
    }
}
```

## Adding Resources

Once package is installed, you can add localizable resources as you [would do it normally](https://github.com/valdisiljuconoks/LocalizationProvider/blob/master/docs/resource-types.md).

```csharp
namespace MyProject.Localization
{
    [LocalizedResource]
    public class HomePageResources
    {
        public static string Header => "This is home page header!";
        ...
    }
}
```

## Require Resource on Client-Side

Sometimes it is required to pull down resources on client-side and work with them from some scripting language. This is possible.

### Pulling Resources via HtmlHelpers

If you will pull down resources via `HtmlHelper` you are basically letting library to generate special `<script>` tag with link to handler endpoint for getting resources.

```razor
@Html.GetTranslations(typeof(MyProject.Localization.HomePageResources))
```

This will generate `script` tag with link to `/jsl10n/MyProject.Localization.HomePageResources`. By default client-side resource handler is installed on `/jsl10n` path. Currently it's not configurable and requires some manual labor to add support. If it's important for you - please create a GitHub issue!

### Pulling Resources as JSON

Sometimes you may need to pull down resources in JSON format and work with them totally differently and not necessarily pollute `window` global scope. Anyways so essentially - to get resources down on client-side as JSON, you need pull them via some XHR method (pick any library that does this).

```javascript
<script type="text/javascript">
    $(function() {
        $.ajax({
            url: '/jsl10n/MyProject.Localization.HomePageResources?json=true',
            method: 'GET'
        }).done(function (data) {
            document.getElementById('theJsPlaceholder1').innerHTML = data.MyProject.Localization.HomePageResources.Header;
        });
    });
</script>
```

Key here is to add `json=true` query string parameter. Also HTTP header detection will be supported, but this feature didn't make it to the first version.

Structure of the JSON returned is exactly the same as you would write resource translation access code on the server-side. It's basically [FQN](https://docs.microsoft.com/en-us/dotnet/framework/reflection-and-codedom/specifying-fully-qualified-type-names) of the property.

```javascript
...innerHTML = data.MyProject.Localization.HomePageResources.Header;
```

### Scoping

Scoping for the requested resources also works. For example, you might request only `/jsl10n/MyProject.Localization`. That should return all localizable resources in that namespace. Structure of the JSON returned will be the same as when requesting specific resource class.

### Note on 404.7 Error

We have seen that sometimes you might get 404.7 error back from IIS whatever you do. This turns out to be reserved [file extension request filtering](https://docs.microsoft.com/en-us/iis/configuration/system.webserver/security/requestfiltering/fileextensions/) in IIS. We noticed this with quite common folder name `Resources/`. So when you might be requesting all localized resources under `Resources/` folder - you will end up with url `.Resources`. This will most probably be blocked under IIS settings. Either you should allow this "file extension" or just request specific resource class.

### Nested Resource Classes

It's supported to have nested localizable classes which makes it quite nice resource grouping possibilities. By using nested classes resource key will contain `+` symbol. This symbol is not nicely supported in url, therefore - you should replace it with `---` (triple dash).

Example:

```csharp
namespace MyProject.Localization
{
    public class CommonResources
    {
        [LocalizedResource]
        public class HomePageResources
        {
            public static string Header => "This is home page header!";
            ...
        }
    }
}
```

To request `HomePageResources` class in client-side, you should construct url in following format: `/jsl10n/MyProject.Localization.CommonResources---HomePageResources`.

Happy localizing!

[*eof*]
