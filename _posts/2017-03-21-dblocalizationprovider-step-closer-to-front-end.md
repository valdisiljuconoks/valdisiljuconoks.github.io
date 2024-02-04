---
title: DbLocalizationProvider step closer to front-end
author: valdis
date: 2017-03-21 15:45:00 +0200
categories: [Add-On, .NET, C#, Localization Provider, Episerver, Optimizely]
tags: [add-on, .net, c#, open source, localization, episerver, optimizely]
---

Along with other smaller bug fixes, database-driven localization provider for EPiServer got closer to front-end. We joined forces together with my old pal and friend [Arve Systad](https://github.com/ArveSystad) and made it possible to add translations to client side resources as well.

## Setup

So setup for the client-side resource localization with help of DbLocalizationProvider plugin is more or less straight forward. You will need to install `DbLocalizationProvider.EPiServer.JsResourceHandler` package from [EPiServer feed](http://nuget.episerver.com/en/?search=localization) and add corresponding `<script>` include in your markup file to fetch translations from the server:

```html
<script src="/jsl10n/{beginning-of-resource-key(s)}">
```

You have to specify beginning of resource keys to fetch those from the server. For instance you have following resources defined in your code:

```csharp
namespace MyProject
{
    [LocalizedResource]
    public class MyResources
    {
        public static string FirstProperty => "One";
        public static string SecondProperty => "Two";
    }
}
```

You can use Html helper to get translations:

```razor
@Html.GetTranslations(typeof(MyProject.MyResources))
```

However if you have more than single resource container type:

```csharp
namespace MyProject
{
    [LocalizedResource]
    public class MyResources
    {
        public static string FirstProperty => "One";
        public static string SecondProperty => "Two";
    }

    [LocalizedResource]
    public class AlternativeResources
    {
        public static string ThirdProperty => "Three";
        public static string FothProperty => "Four";
    }
}
```

following resources will be registered:

```
MyProject.MyResources.FirstProperty
MyProject.MyResources.SecondProperty
MyProject.AlternativeResources.ThirdProperty
MyProject.AlternativeResources.FothProperty
```

To include all resources from those classes, you will need to specify *parent* container name:

```html
<script src="/jsl10n/MyProject"></script>
```

All resources from both classes will be retrieved:

![](/assets/img/2017/03/2017-03-19_23-54-26.png)

Resources retrieved in this way are accessible via `jsl10n` global variable:

```html
<script type="text/javascript">
   alert(window.jsl10n.MyProject.AlternativeResources.ThirdProperty);
</script>
```

**NB!** Notice that naming notation of the resource is exactly the same as it's on the server-side. This notation should reduce confusion and make is less problematic to switch from server-side code to front-end, and vice versa.

## Aliases

Sometimes it's required to split resources into different groups and have access to them separately. Also the same problem will occur when you would like to retrieve resources from two different namespaces in single page. Therefore aliasing particular group of resources might come handy. You have to specify `alias` query string parameter:

```html
<script src="/jsl10n/MyProject.MyResources?alias=m"></script>
<script src="/jsl10n/MyProject.AlternativeResources?alias=ar"></script>

<script type="text/javascript">
   alert(window.m.MyProject.MyResources.FirstProperty);

   alert(window.ar.MyProject.AlternativeResources.ForthProperty);
</script>
```

## Explicit Translation Culture

Sometimes it's also necessary to fetch translations for other language. This is possible specifying `lang` query parameter.

For single container translations:

```razor
@Html.GetTranslations(typeof(MyProject.MyResources), new CultureInfo("en").Name)
```

Or for multiple containers:

```html
<script src="/jsl10n/MyProject?lang=no"></script>
```

**Note:** Those resources that do not have translation in requested language will not be emitted in resulting `json` response.

Happy front-end localization!

[*eof*]
