---
title: Overview of some of the less-known Optimizely Add-Ons
author: valdis
date: 2023-03-06 10:30:00 +0200
categories: [Add-On, Optimizely, Episerver, .NET, C#, Open Source]
tags: [Add-On, Optimizely, Episerver, .NET, C#, Open Source]
---

We have more than seven hundred packages (717) on Optimizely NuGet feed. Official Optimizely packages are part of this number, but the majority is community driven open-source packages enhancing and extending the platform, services and developer, administrator or editor experiences. It's a lot. And there is a great community behind this number.

I had a chance to talk more about Optimizely Add-Ons at last [Optimizely Happy Hour](https://world.optimizely.com/community/optimizely-dev-happy-hours/happy-hour-emea/).

If you want to go through the list at a slower speed, this blog post is covering topics and packages touched on in the session.

I ran through all the packages on the feed and selected some of less-known (either one I haven't seen/heard myself or with low download count). Read a bit about the package, added value and purpose of the package. I chose to filter only packages targeting .NET5 and above.

I could not select every single package and review it here, but I picked interesting ones and grouped them 4 categories:
* [General / Framework](#general-framework)
    * [Source Analyzers](#source-analyzers)
    * [Settings](#settings)
    * [Extension Methods](#extension-methods)
    * [Health Check](#health-check)
    * [Warm-up](#warm-up)
    * [Session](#session)
    * [Azure Explorer](#azure-explorer)
    * [Event Transports](#event-transports)
* [CMS 12](#cms-12)
    * [TimeSpan](#timespan)
    * [Extended External Links](#extended-external-links)
    * [PDF Preview](#pdf-preview)
    * [Grouping Headers](#grouping-headers)
    * [Better Content Areas](#better-content-areas)
    * [Enhanced Property List](#enhanced-property-list)
    * [Generic LinkItem Collection](#generic-linkitem-collection)
    * [Custom Quick Navigation](#custom-quick-navigation)
    * [Content Type Usage](#content-type-usage)
    * [Unpublish](#unpublish)
    * [Content Security Policy (CSP)](#content-security-policy-csp)
    * [Access Rights Audit Log](#access-rights-audit-log)
* [Commerce 14](#commerce-14)
    * [Currency Exchange Rates](#currency-exchange-rates)
    * [Geolocation Tools](#geolocation-tools)
    * [Product Feed](#product-feed)
* ["Dangerous" (Powerful) Packages](#dangerous-powerful-packages)
    * [SQL Studio](#sql-studio)
    * [Developer Tools](#developer-tools)
    * [Developer Console](#developer-console)


# General / Framework
## Source Analyzers
Source analyzer package allows you to get runtime exceptions quicker and exposes those as compile time errors.

![addons-1](https://tech-fellow.ghost.io/content/images/2023/03/addons-1.png)

Simply type the following command in your prompt, and you're ready to go!

```
> dotnet add package Stekeblad.Optimizely.Analyzers
```

Once installed, this package scans your source and runs analyzers throwing errors in your face.

![addons-1-1](https://tech-fellow.ghost.io/content/images/2023/03/addons-1-1.png)

## Settings
Settings package allows you to store various settings, configuration parameters or even maybe arguments to scheduled job as part of content repository.
It will provide familiar editorial UI for administrators or powerful editors to create and maintain settings.

![addons-2](https://tech-fellow.ghost.io/content/images/2023/03/addons-2.png)

Install the package by:
```
> dotnet add package AddOn.Episerver.Settings
```

Once installed, you need to define your model for the settings "container".

![addons-2-1](https://tech-fellow.ghost.io/content/images/2023/03/addons-2-1.png)

The new section will show up to create or maintain your setting values using well-known editorial UI from Optimizely.

![addons-2-2](https://tech-fellow.ghost.io/content/images/2023/03/addons-2-2.png)

Retrieval of the setting values is very easy - just inject `ISettingsService` (similar API as for `IContentLoader`) and retrieve settings by the type parameter.

![addons-2-3](https://tech-fellow.ghost.io/content/images/2023/03/addons-2-3.png)

We see this as great addition when you need to parametrize scheduled job execution and need to provide different parameter values (and preferably do this without restart of the site).

## Extension Methods
There are 2 packages worth looking at if you would like to improve productivity for your developers. Some of the methods overlap, but overall - both packages can be installed and used simultaneously.

The first package is from Blend Interactive.

![addons-3](https://tech-fellow.ghost.io/content/images/2023/03/addons-3.png)

Installation is as simple as the rest of the packages:
```
> dotnet add package Blend.Optimizely
```

And another package is from Geta Digital.

![addons-4](https://tech-fellow.ghost.io/content/images/2023/03/addons-4.png)

```
> dotnet add package Geta.Optimizely.Extensions
```

There are many extension methods for BCL (Base Class Library) types and also for Optimizely content and related types.

![addons-4-1](https://tech-fellow.ghost.io/content/images/2023/03/addons-4-1.png)

Browse around and see which ones you like and use most.

## Health Check
It's always a good idea to keep your site in-check. Health check package from Optimizely could be used to implement health endpoints and utilize those in your infrastructure monitoring routines.

![addons-5](https://tech-fellow.ghost.io/content/images/2023/03/addons-5.png)

Throw this command at your prompt to get package installed.

```
> dotnet add package EPiServer.Cms.HealthCheck
```

You need to register CMS endpoints in the pipeline.

![addons-5-1](https://tech-fellow.ghost.io/content/images/2023/03/addons-5-1.png)

Under the hood it's plugged into standard ASP.NET Core health check pipeline and uses `ISchemaUpdater` to get database version. Database version is read by executing stored procedure. So essentially health endpoint is checking if site can read database and if connection is successfully established.
You can also implement your own health checks and add to the collection to verify something specific your project needs.

## Warm-up
During cold weather we need to warm-up to keep us alive. However, in your Optimizely site you can use warm-up package to prepare site to serve requests to the end-users. Using warm-up package from Optimizely.

![addons-6](https://tech-fellow.ghost.io/content/images/2023/03/addons-6.png)

Installation:

```
> dotnet add package EPiServer.Cms.Warmup
```

By adding package to the site, it will register callback to `IApplicationLifetime.ApplicationStarted` event and will fire off requests to all sites' root url found in configuration.

![addons-6-1](https://tech-fellow.ghost.io/content/images/2023/03/addons-6-1.png)

You can also add additional paths for the warmup package to hint during startup. This is useful if you have special cache preparation endpoints and similar actions exposed to prepare your site before it's ready to serve.
And also you can add "Warmup Health Check" - to double check that warm-up procedure has been executed successfully.

## Session
Sometimes you need to keep a track of device or a user. Optimizely offers you small utility package to keep track of user's sessions.

![addons-7](https://tech-fellow.ghost.io/content/images/2023/03/addons-7.png)

As usual - paste this in your favorite command prompt:

```
> dotnet add package EPiServer.Session
```

![addons-7-1](https://tech-fellow.ghost.io/content/images/2023/03/addons-7-1.png)

The default implementation of the session storage uses browser cookies to transfer session identifier to the server.

## Azure Explorer
Usually when you run your site in DXP you have Azure Storage Account attached to the site to keep your assets safe. It's quite rare but still - it might be necessity to browse around and review storage content. Sometimes even maybe upload something that needs to be access from the project.

![addons-8](https://tech-fellow.ghost.io/content/images/2023/03/addons-8.png)

For this, head over to your command prompt and install Mark's Storage Explorer package:

```
> dotnet add package EPiServer.Azure.StorageExplorer
```

After installation your will have new admin section to access the addon.

![addons-8-1](https://tech-fellow.ghost.io/content/images/2023/03/addons-8-1.png)

Storage Explorer user interface allows to download or upload files to the blob storage. Comes very handy in case of emergency.

## Event Transports
If you want to emit or receive events using other transport type as `ServiceBus` (by default in DXP) you have few options available:

* `StefanOlsen.Optimizely.Events.Sockets`
* `EPiServer.Events.MassTransit`
* or `StefanOlsen.Optimizely.Events.Redis`

# CMS 12
## TimeSpan
It's much easier to work with data types (`TimeSpan`) that you really need and not subtract something every time you get property value (`DateTime`) to get to the actual value you need.

Using following command in your prompt to get `TimeSpan` data type with nice editorial interface as part of your content type definition:

```
> dotnet add package Advanced.CMS.TimeProperty
```

![addons-9](https://tech-fellow.ghost.io/content/images/2023/03/addons-9.png)

You can now define content property as `TimeSpan`.

![addons-9-1](https://tech-fellow.ghost.io/content/images/2023/03/addons-9-1.png)

And also set value using time selector in editorial user interface.

![addons-9-2](https://tech-fellow.ghost.io/content/images/2023/03/addons-9-2.png)

## Extended External Links
If you need to keep track of all outgoing external links in your site, you can expose all external links in a single list.

![addons-10-2](https://tech-fellow.ghost.io/content/images/2023/03/addons-10-2.png)

Head over to NuGet feed and install `ExtendedExternalLinks` package:

```
> dotnet add package ExtendedExternalLinks
```

It will give you a widget to summarize all your external links in the site and link to the content reference where it's used.

![addons-10-1-1](https://tech-fellow.ghost.io/content/images/2023/03/addons-10-1-1.png)

## PDF Preview
If you site is working with PDF documents and editors asking if they could preview those directly in Optimizely editorial user interface? Now you can just install PDF Preview package and you are good to go!

```
> dotnet add package EPiServer.PdfPreview
```

![addons-11-1](https://tech-fellow.ghost.io/content/images/2023/03/addons-11-1.png)

## Grouping Headers
When you have too many properties under single group ("tab" in editorial user interface), you can use this package to group properties under logical group in the same tab.

![addons-12-2](https://tech-fellow.ghost.io/content/images/2023/03/addons-12-2.png)

```
> dotnet add package Advanced.CMS.GroupingHeader
```

Then you need to decorate your group's first element (rest of the properties will follow the same semantics and will be added to the group).

![addons-12-1-1](https://tech-fellow.ghost.io/content/images/2023/03/addons-12-1-1.png)

In editorial interface now all properties under this group are together with nice group header to inform editor what kind of content properties are located in this group.

![addons-12-2-1](https://tech-fellow.ghost.io/content/images/2023/03/addons-12-2-1.png)

## Better Content Areas
We have at least two packages that tries to make content areas better by 2 cents.

* `AddOn.Optimizely.ContentAreaLayout`
* and `TechFellow.Optimizely.AdvancedContentArea`


### ContentAreaLayout
`ContentAreaLayout` plugin does exactly what it states - controlling layout of the content area.

![addons-13](https://tech-fellow.ghost.io/content/images/2023/03/addons-13.png)

Installation is super simple, as usual, punch this into your command line:

```
> dotnet add package AddOn.Optimizely.ContentAreaLayout
```

Next step is to define your block type which will act as controller for the following blocks and will determine the width of the blocks they occupy in the row.

![addons-13-1](https://tech-fellow.ghost.io/content/images/2023/03/addons-13-1.png)

Now you have to create new instance of the block and drop it in the content area (before the blocks you would like to control width of).

![addons-13-2](https://tech-fellow.ghost.io/content/images/2023/03/addons-13-2.png)

And lastly - you can see how layout block is controlling the width taken by following blocks in the content area.

![addons-13-3](https://tech-fellow.ghost.io/content/images/2023/03/addons-13-3.png)

Using this plugin you can create powerful content area layouts and avoid of the mistakes that editors might make.

### AdvancedContentArea
If you would like to give more options of the editors to choose layout themselves, then `AdvancedContentArea` might help you with this challenge.

![addons-14](https://tech-fellow.ghost.io/content/images/2023/03/addons-14.png)

Install it via command line:

```
> dotnet add package TechFellow.Optimizely.AdvancedContentArea
```

You register plugin in your project, add necessary `DisplayOptions` and you are pretty much ready to go.

![addons-14-1](https://tech-fellow.ghost.io/content/images/2023/03/addons-14-1.png)

Now editors have option to specify display option by themselves and setup content area in the way they need it to be.

![addons-14-2](https://tech-fellow.ghost.io/content/images/2023/03/addons-14-2.png)

Checking DOM later on, you can see that content area renderer added all required classes to look and work nicely in Bootstrap grid system.

![addons-14-3](https://tech-fellow.ghost.io/content/images/2023/03/addons-14-3.png)

**psst!** `AdvancedContentArea` can be used also to control layout of your Optimizely Forms elements.

![addons-14-4](https://tech-fellow.ghost.io/content/images/2023/03/addons-14-4.png)

## Enhanced Property List
Built-in property list is nice and all, but it's missing two important features - able to render image and content reference.

![addons-15](https://tech-fellow.ghost.io/content/images/2023/03/addons-15.png)

This package is exactly about to fix it.

```
> dotnet add package DoubleJay.Epi.EnhancedPropertyList
```

Imagine that we do have content in property list which has image and content reference.

![addons-15-1](https://tech-fellow.ghost.io/content/images/2023/03/addons-15-1.png)

By default, rendering row of this content type in property list, might not resolve image and content reference nicely.

![addons-15-2](https://tech-fellow.ghost.io/content/images/2023/03/addons-15-2.png)

Now, changing descriptor of the field, we can fix this issue.

![addons-15-3](https://tech-fellow.ghost.io/content/images/2023/03/addons-15-3.png)

Images and content references are rendered correctly now in the list:

![addons-15-4](https://tech-fellow.ghost.io/content/images/2023/03/addons-15-4.png)

## Generic LinkItem Collection
When you list item collection and need some small tweaks or extension, it's hard. This package might help you to get around.

![addons-16](https://tech-fellow.ghost.io/content/images/2023/03/addons-16.png)

This generic link item collection package, author is solving problem about not being able to extend link item collection property.

```
> dotnet add package Geta.Optimizely.GenericLinks
```

You need to define content type that will expand `LinkItem` and type in additional properties you would need to save on `LinkItem`.

![addons-16-1](https://tech-fellow.ghost.io/content/images/2023/03/addons-16-1.png)

You need to use new link collection content type in your content definition.

![addons-16-2](https://tech-fellow.ghost.io/content/images/2023/03/addons-16-2.png)

And now, editorial user interface has changed a bit for item editor.

![addons-16-3](https://tech-fellow.ghost.io/content/images/2023/03/addons-16-3.png)

As you can see, now editors are able to supply `LinkItem` with additional properties if needed.

## Custom Quick Navigation
If your project has a lot of shortcuts to other systems or you just might want to see available favorites always available at your fingertips, install `Quick Navigation` extension package.

![addons-17](https://tech-fellow.ghost.io/content/images/2023/03/addons-17.png)

```
> dotnet add package Epicweb.Optimizely.QuickNavExtension
```

![addons-17-1](https://tech-fellow.ghost.io/content/images/2023/03/addons-17-1.png)

And now you quick navigation menu turned into unrecognizable :)

![addons-17-2](https://tech-fellow.ghost.io/content/images/2023/03/addons-17-2.png)

## Content Type Usage
There are few packages that try to address this specific challenge - "Where this content is used?"

![addons-18](https://tech-fellow.ghost.io/content/images/2023/03/addons-18.png)

Try out content usage package.

```
> dotnet add package Eshn.ContentTypeUsage
```

If will answer almost all type usage questions you might have.

![addons-18-1](https://tech-fellow.ghost.io/content/images/2023/03/addons-18-1.png)

## Unpublish
Super tiny, but super useful package - gives you an option to unpublish your content.

![addons-19](https://tech-fellow.ghost.io/content/images/2023/03/addons-19.png)

Renders new menu item in the publish menu. Nothing more, nothing less :)

![addons-19-1](https://tech-fellow.ghost.io/content/images/2023/03/addons-19-1.png)

## Content Security Policy (CSP)
It's important to keep your site healthy. This also includes [content security policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP).
Before this package, usually I would add middle-ware or something which could affect my outgoing responses and add required CSP fields. I would need to build some configuration storage where to keep CSP settings, etc.
However, this is already taken care of and you can just install the package and enjoy your time off.

![addons-20](https://tech-fellow.ghost.io/content/images/2023/03/addons-20.png)

```
> dotnet add package Jhoose.Security.Admin
```

And now you are able to specify and maintain all CSP directives and configuration accordingly.

![addons-20-1](https://tech-fellow.ghost.io/content/images/2023/03/addons-20-1.png)

Nice plugin to keep your sites secure.

## Access Rights Audit Log
Talking about security, it's also important to audit changes in your site and be informed about any abnormal activity.

For this reason there is also great package to keep track of access right changes for the content.

![addons-21](https://tech-fellow.ghost.io/content/images/2023/03/addons-21.png)

```
> dotnet add package Swapcode.Optimizely.AuditLog
```

Package hooks into access right changes pipeline and writes events to built-in "Change Log".

![addons-21-1](https://tech-fellow.ghost.io/content/images/2023/03/addons-21-1.png)

Nice addition to keep your site secure.
# Commerce 14
For the Commerce part there are not actually that many packages yet. So it's space we all need to fill :)
Anyways I picked some of interesting packages worth mentioning.

## Currency Exchange Rates
When you are working with multi-currency sites, it's good to have exchange rates for currencies loaded and ready to use. By default, this functionality is not something that is built-in into the platform. But you can use addon to extend platform functionality and load necessary currency exchange rates automatically.

Paste this into your command line:

```
> dotnet add package EPi.Libraries.Commerce.ExchangeRates
```


![addons-22](https://tech-fellow.ghost.io/content/images/2023/03/addons-22.png)

Also you need to decide which source for the currency rate exchange system you will use to download data. Addon supports two sources:

* Fixer
* Currency Layer

There will be another package that you will need to install to get access to the exchange rates source data.

`EPi.Libraries.Commerce.ExchangeRates` will register scheduled job to load list of currencies and download and import list of exchange rates.

![addons-22-1](https://tech-fellow.ghost.io/content/images/2023/03/addons-22-1.png)

After job is ran, relative currency (ones used in the site) exchange rates will be downloaded and imported into the database.

![addons-22-2](https://tech-fellow.ghost.io/content/images/2023/03/addons-22-2.png)

## Geolocation Tools
Continuing multi-currency topic, most of the times you will also have to handle multi-market functionality. Very often requirement is to select "closest" market based on end-user data (IP address mostly).

You can get help from this package:

```
> dotnet add package Geta.Optimizely.GeolocationTools.Commerce
```

![addons-23](https://tech-fellow.ghost.io/content/images/2023/03/addons-23.png)

Using this package you will be able to automatically select market based on end-user location.

![addons-23-1](https://tech-fellow.ghost.io/content/images/2023/03/addons-23-1.png)

So again, functionality that is often requested by our customers is exposed and packed as reusable extension to the platform.

## Product Feed
Working with Commerce projects one of the standard requirement is to expose catalog data in one or another format for external consumption. Whether it's Google Product Feed or external service pulling in CSV format, or any other way - `ProductFeed` package got you covered.

```
> dotnet add package Geta.Optimizely.ProductFeed
```

![addons-24](https://tech-fellow.ghost.io/content/images/2023/03/addons-24.png)

`ProductFeed` package is highly customizable and extended, so it should suit your needs.

You can plugin various mappers and loaders and what's not to enable and build your exposure pipeline the way you need it to be.

![addons-24-1](https://tech-fellow.ghost.io/content/images/2023/03/addons-24-1.png)

After running scheduled job to build feed, it's available for consumption.

![addons-24-2](https://tech-fellow.ghost.io/content/images/2023/03/addons-24-2.png)

# "Dangerous" (Powerful) Packages
Packages and Addons in this category are coming with great power (and great responsibility).
**DISCLAIMER**: Use packages enlisted in this section on your own risk!

## SQL Studio
When you need back-door access to your database to perform some nasty clean-up action, or updating something in batch or fix the data - one way to achieve this is to have SQL Studio addon installed and accessible.

```
> dotnet add package Gulla.Episerver.SqlStudio
```

![addons-25](https://tech-fellow.ghost.io/content/images/2023/03/addons-25.png)

This addon allows you to access database directly and execute queries or commands for your convenience.

![addons-25-1](https://tech-fellow.ghost.io/content/images/2023/03/addons-25-1.png)

Package is configurable to allow only certain keywords and commands (preventing accidental damage to the database). These settings should be reviewed and approved before deployed to production (or any other environment).

## Developer Tools
Another tool for advanced users and administrators is `DeveloperTools`. This package allows to look into the application state, review startup times, check environment variables and even content of dependency injection container.

```
> dotnet add package EPiServer.DeveloperTools
```

![addons-26](https://tech-fellow.ghost.io/content/images/2023/03/addons-26.png)

Set of available tools in `DeveloperTools` package is growing and community is adding more and more useful tools there.

![addons-26-1](https://tech-fellow.ghost.io/content/images/2023/03/addons-26-1.png)

## Developer Console
Last, but not least - keep an eye on Allan's work.
One of the package that is scheduled soon to be released - `DeveloperConsole` ([link to the repo](https://github.com/CodeArtDK/CodeArt.Optimizely.DeveloperConsole)).

![addons-27](https://tech-fellow.ghost.io/content/images/2023/03/addons-27.png)

This is basically PowerShell in your Optimizely site.

# Feed Browser
Also, if you are wondering around NuGet feed and exploring what addons and packages are available on the feed, head over to David's [feed explorer](https://www.david-tec.com/optimizely-nuget-feed-explorer/) for more elegant browsing experience :)

![addons-28](https://tech-fellow.ghost.io/content/images/2023/03/addons-28.png)

# Summary
We have more than 700 packages on the feed. Most of the packages have been delivered by our beloved community and tireless community members and [OMVPs](https://world.optimizely.com/omvp/).

Bit shout to everyone improving Optimizely platform everyday!


Happy coding!
[*eof*]
