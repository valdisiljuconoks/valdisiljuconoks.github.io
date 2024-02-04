---
title: Feedback Taken - EPiServer Localization Provider Updates
author: valdis
date: 2017-07-07 13:30:00 +0200
categories: [Add-On, .NET, C#, Localization Provider, Episerver, Optimizely]
tags: [add-on, .net, c#, open source, localization, episerver, optimizely]
---

Listen to the audience. I was more than lucky to be on some local user groups to talk about strongly typed localization provider for EPiServer. Feedback from the audience is more likely to be provided than awaiting comments in blog posts. You have to listen to the audience, you really have to. They are your consumers, they know more about the context and projects where and how your library might be used.

Over couple of last weeks I've been heads down busy with some of the feedback implementation. Here is a short list of updates added to latest versions of library (both parts - on [EPiServer feed](http://nuget.episerver.com/en/?search=dblocalization&sort=MostDownloads&page=1&pageSize=10) (v3.4) and on [NuGet feed](https://www.nuget.org/packages?q=dblocalization) (v2.7)):

* Added [table sorting](#tablesortinginadminui) in AdminUI
* Added [tree view for resources](#treeviewinadminui) in AdminUI
* Ability to generate resources for [additional cultures](#additionalcultures)
* [Cache events](#cacheevents)
* JsResourceHandler [caches generated script](#jsresourcehandlercachesgeneratedscript)

## Table Sorting in AdminUI

-- "Can I sort resources somehow? By key, translation..? Anything..?"<br/>
-- "..Eh.. No. Not yet.."

Table sorting by any column is now available. Should be much easier for the editors to find what they are looking for.

![](/assets/img/2017/07/2017-07-05_23-15-59.png)

## Tree View in AdminUI

Also another request from audience - make it easier for the editors to work with resources (which is quite logical request looking at current flat list with cryptic names there).

Now with the latest version is possible to switch to table view and work with resources there.

![](/assets/img/2017/07/2017-07-05_23-21-19.png)

Unfortunately some of the features (like resource delete) is not properly working yet in the tree view, for that operation - you have to switch back to flat table view and execute there.

## Additional Cultures

If you need to register resources for other languages as well (not only for default one), it's possible using following attribute:

```csharp
namespace DbLocalizationProvider.Demo
{
    [LocalizedResource]
    public class CommonResources
    {
        [TranslationForCulture("Navn", "no")]
        [TranslationForCulture("Namn", "sv")]
        public static string UserName => "Name";
    }
}
```

**NB!** If there will be duplicate resource translations for the same language - an **exception** will be **thrown**.
Which also means - that theoretically an exception might be thrown if your default language is let's say "en" and you have additional translations via attribute also set to "en". So be careful.

## Cache Events

Got a request after one of my presentation from audience to **expose events** around cache - when item is added to the cache, when removed, etc. Reason for this was manual cache event propagation further to other nodes in the cluster.

Now you are able to subscribe to cache events via `ConfigurationContext`:

```
[assembly: OwinStartup(typeof(Startup))]

namespace DbLocalizationProvider.MvcSample
{
    public class Startup
    {
        public void Configuration(IAppBuilder app)
        {
            app.UseDbLocalizationProvider(ctx =>
            {
                ...
                ctx.CacheManager.OnRemove += CacheManagerOnOnRemove;
            });
        }

        private void CacheManagerOnOnRemove(CacheEventArgs args)
        {
            // black magic happens here..
        }
    }
}
```

You will be able to get info about cache event operation (`Insert`, `Remove`, etc) via `CacheEventArgs` argument.

## JsResourceHandler caches generated script

Until now every time you would be requesting [translations for client-side](https://tech-fellow.ghost.io/2017/03/20/dblocalizationprovider-step-closer-to-front-end/) it would generate script content from fresh start. This is not great! With the help of cache events - now Javascript Resource Handler can easily cache generated script content for the requested translations and store it in memory. Every time any resource that was included in the generated script changes (either by editor or code) event is raised that will signal resource handler to invalidate particular cache entries and build up it upon next request. This greatly increases site performance!

<br/>
If you have any feedback or ideas to be added to provider - please file them in [GitHub repo](https://github.com/valdisiljuconoks/LocalizationProvider).

<br/>
Happy localizing..!
<br/>
[*eof*]
