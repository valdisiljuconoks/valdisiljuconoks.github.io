---
title: EPiServer DbLocalizationProvider - fresh 2.0 is out!
author: valdis
date: 2016-11-03 20:05:00 +0200
categories: [Add-On, .NET, C#, Localization Provider, Episerver, Optimizely]
tags: [add-on, .net, c#, open source, localization, episerver, optimizely]
---

## Why?

You might ask, but changes I guess is the only constant thing in this industry. DbLocalizationProvider is witnessing new version and some cool features covered in more details below. Also internally provider survived quite huge refactoring activities, while I was redesigning from central services and repositories to more granular command and queries. But that’s the story for another blog post.

## New Features in 2.0

Here goes the list of new features available in version 2.0. List is not sorted in any order.

### Translate System.Enum

It's quite often that you do have a enumeration in the domain to ensure that your entity might have value only from predefined list of values - like `Document.Status` or `PurchaseOrder.Shipment.Status`. These values are usually defined as `System.Enum`. And it's also quite often case when you need to render a list of these available values on the page or anywhere else for the user of the application to choose from. Now in 2.0 enumeration translation is easily available.

```csharp
[LocalizedResource]
public enum DocumentStatus
{
    None,
    New,
    Pending.
    Active,
    Closed
}

[LocalizedModel]
public class Document
{
    ...

    public DocumentStatus Status { get; }
}
```

Now you can use following snippet to give end-user localized dropdown list of available document statuses:

```razor
@using DbLocalizationProvider
@model Document

@{
    var statuses = Enum.GetValues(typeof(DocumentStatus))
                       .Cast<DocumentStatus>()
                       .Select(s => new SelectListItem
                                        {
                                            Value = s.ToString(),
                                            Text = s.Translate()
                                        });
}

@Html.DropDownListFor(m => m.Status, statuses)
```

Or if you just need to output current status of the document to the end-user:

```razor
@using DbLocalizationProvider
@model Document

@Model.Status.Translate()
```

### Templates with Placeholders

Index based `string.Format` style arguments for localized message is very nice and flexible approach. However - it's readable most probably only by developer or somebody who understands why first element starts with `{0}` and not `{1}`.
When you need to give access to the resources to editors or anybody else with even enough technical background, you might receive questions back - "What is `{0}` and what will be placed in `{4}`?" Pretty tricky question if you need to open source code and look for passed in format arguments.

Now in v2.0 you can pass in anonymous object with named properties and use those in your localized resource.

For example, greeting message for end-users (I really hate these kind of greetings.. so simple to invest in vocative case):

```csharp
[LocalizedResource]
public class StartPageResources
{
    public static string GreetingMessage => "Hi {Firstname} {Lastname}, where would you like to click today?";
}


@Html.Translate(() => StartPageResources.GreetingMessage, new { Firstname = "John", Lastname = "Doe" })
```

Or you may have a view model as basis for some translated message, so you can pass in directly that model:

```csharp
public class Document
{
    public string Nr { get; }
    public string Author { get; }
}

[LocalizedResource]
public class DocumentResources
{
    public static string SharedTo => "{Author}, somebody shared your '{Nr}' document!";
}
...

@model Document

@Html.Translate(() => StartPageResources.GreetingMessage, Model)
```

Now it should be much easier for the editors to understand what value goes in which placeholders.


### Custom Resource Keys

Back in time [Linus blogged](http://world.episerver.com/blogs/Linus-Ekstrom/Dates/2013/12/New-standardized-format-for-content-type-localizations/) about new standardized way to localize Cms content types and properties. As originally EPiServer is based on Xml and XPath resource key conventions - there was no support in DbLocalizationProvider to make it happen and supply these specially generated resource keys via strongly typed resources.

Now in v2.0 you can do that by specifying `ResourceKey` directly on property of the `PageType`:

```csharp
...
using DbLocalizationProvider;

[ContentType(GUID = "....")]
[LocalizedModel(KeyPrefix = "/contenttypes/startpage/")]
[ResourceKey("name", Value = "This is StartPage!")]
public class StartPage : PageData
{
    [ResourceKey("properties/headertitle/caption", Value = "Title of the page")]
    public virtual string HeaderTitle { get; set; }
}
```

This class should register 2 resources with following keys (that will be picked up by EPiServer automatically):

* `/contenttypes/startpage/name`
* `/contenttypes/startpage/properties/headertitle/caption`

You may also need to specify descriptions or any other resources for that Cms content type property, you can have multiple `ResourceKey` attributes:

```csharp
...
using DbLocalizationProvider;

[ContentType(GUID = "....")]
[LocalizedModel(KeyPrefix = "/contenttypes/startpage/")]
[ResourceKey("name", Value = "This is StartPage!")]
[ResourceKey("description", Value = "This is StartPage!")]
public class StartPage : PageData
{
    [ResourceKey("properties/headertitle/caption", Value = "Title of the page")]
    [ResourceKey("properties/headertitle/help", Value = "Enter some meaningful title of the page")]
    public virtual string HeaderTitle { get; set; }
}
```

### Support for Nullable properties

This is pretty small addition, but previously in 1.x version nullable properties where not supported.
Now following property will be discovered and added to localized resources:

```csharp
[LocalizedModel]
public class Document
{
    public DateTime? DeletedWhen { get; }
}
```

Following data types are treated as simple and thus - added to list of resources to synchronize:

* `typeof(Enum)`,
* `typeof(string)`,
* `typeof(char)`,
* `typeof(Guid)`,
* `typeof(bool)`,
* `typeof(byte)`,
* `typeof(short)`,
* `typeof(int)`,
* `typeof(long)`,
* `typeof(float)`,
* `typeof(double)`,
* `typeof(decimal)`,
* `typeof(sbyte)`,
* `typeof(ushort)`,
* `typeof(uint)`,
* `typeof(ulong)`,
* `typeof(DateTime)`,
* `typeof(DateTimeOffset)`,
* `typeof(TimeSpan)`

And their `Nullable<>` counterpart.

### [Display(Description = "...")]

Also small addition to `ModelMetadataProvider` infrastructure available for Asp.Net Mvc pipeline. Now you can also localize description for the property via `DataAnnotations` attributes:

```csharp
namespace MyProject
{
    public class MyViewModel
    {
        [Display(Name = "Login name", Description = "Login name for the user is email.")]
        public string Username { get; set; }
    }
}
```

Will generate following resource `MyProject.MyViewModel.Username-Description` *only* if `Description` property of `Display` attribute will not be `string.Empty`. You can localize it via AdminUI and set new value if needed.

When you need to use this value in your display or editor templates you can access it via `ViewData`:


```html
<div>
    ...
    <span class="field-description">@ViewData.ModelMetadata.Description</span>
    ...
</div>
```

### Mark Required Fields

*NB!* This is experimental feature, so feedback is welcome.
Got a request from one of our projects to indicate all required fields in the system with some sort of prefix (e.g., asterix `"*"` or anything like that). We were considering to create some HtmlHelper extensions for this and revisit its usage across the pages. However, using new DbLocalizationProvider all calls for model metadata is going through `ModelMetadataProvider` infrastructure and there is single point of responsibility for providing value for code snippet like this `@Html.LabelFor(...)`.

So I decided to add this experimental feature to the localization provider to give single configuration option for the developers to enable this requirement. Might not be directly related to responsibility of localization provider, that's why it's still experimental and not sure whether a good idea to add it here. Anyhow, here is the way how to achieve this:

```csharp
[LocalizedResources]
public class Common
{
    public static string RequiredIndicator => " *";
}

[InitializableModule]
[ModuleDependency(typeof(InitializationModule))]
public class SetupLocalization : IInitializableModule
{
    public void Initialize(InitializationEngine context)
    {
        ConfigurationContext.Setup(ctx =>
        {
            ctx.ModelMetadataProviders.MarkRequiredFields = true;
            ctx.ModelMetadataProviders.RequiredFieldResource =
                () => Common.RequiredIndicator;
        });
    }

    public void Uninitialize(InitializationEngine context) { }
}

public class MyViewModel
{
    [Required]
    public string Username { get; set; }
}
```

Now running in this context, if you type in:

```razor
@model MyViewModel

@Html.Label(m => m.Username)
```

With no modifications to default resource translations, it should output:

```html
<label for="...">Username *</label>
```

## Upgrade from version 1.x

If this is the case and you are using 1.x version, first of all - I would like to thank you for trying it out!

Secondly, I tried to make upgrade as transparent as possible for the developers using 1.x.

So currently, you should have following packages installed:

* `DbLocalizationProvider`, `v1.3.2`
* `DbLocalizationProvider.AdminUI`, `v1.3.0`

So you should look for newer versions for these packages:

* `DbLocalizationProvider`, `v1.3.3`
* `DbLocalizationProvider.AdminUI`, `v1.3.1`

These latest versions are so called ghost packages that do not have any content, but have proper references to new dependencies of new package names. By upgrading to this version, you should also get pulled down additional packages:

* `DbLocalizationProvider.EPiServer`, `v2.0.0`
* `DbLocalizationProvider.AdminUI.EPiServer`, `v2.0.0`
* `LocalizationProvider`, `v2.0.0`

Latter package is related with new feature to host localization provider outside of EPiServer.

So theoretically you can remove old versions of `DbLocalizationProvider` and also `DbLocalizationProvider.AdminUI` packages.

Also, by completely removing old `DbLocalizationProvider*` packages and adding directly `DbLocalizationProvider.EPiServer` and/or `DbLocalizationProvider.AdminUI.EPiServer` you should be on the safe side.

## Hosting outside of EPiServer

I guess the most important feature of new 2.0 version is ability to host it outside of the EPiServer.

So you should be able to add reference to [LocalizatioProvider package](https://www.nuget.org/packages/LocalizationProvider/) in pure vanilla Asp.Net Mvc project and you are settle and ready to go. Of course, if you will need to configure and tweak library, that's available as usual - via `AppBuilder` interface (for instance here, to specify which database connection string to use):

```csharp
[assembly: OwinStartup(typeof(Startup))]

namespace DbLocalizationProvider.MvcSample
{
    public class Startup
    {
        public void Configuration(IAppBuilder app)
        {
            app.UseDbLocalizationProvider(ctx =>
            {
                ctx.ConnectionName = "MyConnectionString";
            });
        }
    }
}

```

The only show stopper currently is that I'm not finished with AdminUI for Mvc projects. Just need to put final pieces together and publish.

New blog post will be published once package will become available.

## Thanks!

Thanks for taking time to read through and probably trying out `DbLocalizationProvider` library.

If you have any comments, suggestion, feedback, complaints - please leave it on [GitHub](https://github.com/valdisiljuconoks/LocalizationProvider).

Happy localizing!

[*eof*]
