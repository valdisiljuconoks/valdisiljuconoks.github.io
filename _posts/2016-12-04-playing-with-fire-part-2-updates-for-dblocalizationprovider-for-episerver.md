---
title: Playing with Fire - Part 2, updates for DbLocalizationProvider for EPiServer
author: valdis
date: 2016-12-04 00:35:00 +0200
categories: [Add-On, .NET, C#, Localization Provider, Episerver, Optimizely]
tags: [add-on, .net, c#, open source, localization, episerver, optimizely]
---

Story continues with fire and localization cases inside EPiServer sites. This time smaller update for database localization provider, but still - worth sharing.

## Support for Custom Attributes

Currently database provider supports and understands just a few `System.Attribute` child classes:

* `[LocalizedModel]` - this is main marking attribute for making sure that provider resource sync process will pickup decorated class and scan resources there;
* `[LocalizedResource]` - this is similar as `LocalizedModel`, but makes sure that resources discovered at this class will take localized resource semantics. [Read more here](https://tech-fellow.eu/2016/03/16/db-localization-provider-part-1-resources-and-models/) about differences;
* `[ResourceKey]` - this attribute will make sure that particular resource gets specified key, instead of auto-generated from the synchronizer. This comes handy when you need to control under which key resource will be registered (usually because of some EPiServer built-in conventions for translating their UI - page types or categories for instance);
* `[Include]` - this attribute will make sure that particular property decorated with this will be added to the discovered resource list from that class;
* `[Ignore]` - this is exactly opposite to `[Include]` attribute;
* `[Display]` or `[DisplayName]` - these are attributes mostly used during model metadata generation via Asp.Net pipeline. So when you write `@Html.EditorFor(m => m.Username)`, either `[Display]` or `[DisplayName]` attribute will be used to get label for that field;
* `[StringLength]`, `[Required]` or any other `ValidationAttribute` (from `System.ComponentModel.DataAnnotations` namespace) - these attributes will play their roles during model metadata generation process deep inside Asp.Net pipeline when somebody will ask for editor markup using `@Html.EditorFor(m => ...)`. Data validation attributes will make sure that all necessary Html attributes are added to the field input element for client-side validation library to pick it up.

Almost each of these attributes have additional properties to control and adjust scanning and registration process.
However - it's still not enough.

Sometimes there is need for arbitrary attribute to be recognized by the provider.


### [HelpText] Scenario

In particular, there was requirement in one of our projects to add help text or small hint block for visitors to understand more about the meaning of the input field.

![](/assets/img/2016/12/2016-12-04_00-24-40.png)

One of the potential solution for this case that we were thinking about - to generate resources with specifc key conventions (like `/Form/HelpTexts/{FieldName}`) using existing `[ResourceKey]` attributes. And then inside field editor or display templates ask localization provider for a specific key:

```csharp
...
@LocalizationService.Current.GetString($"/Form/HelpTexts/{ViewData.ModelMetadata.PropertyName}");
```

As this seems to be viable solution, it doesn't feel right... Whole idea of the database localization provider for EPiServer was to replace "*stringly* typed access for the resources" to "**strongly** typed one". And now - just to support additional custom attributes we would need to jump back to stringly interface.

So this requires some tweaks in the library.

### Implementation Details

So what's now supported is something called `CustomAttributeDescriptors` (you can guess where naming comes from).
The way you would ask localization provider to include your own `System.Attribite` classes in scanning and discovery process - is by adding descriptors to configuration context. The most convenient way - via EPiServer [initialization module](http://world.episerver.com/documentation/developer-guides/CMS/initialization/).

```csharp
[InitializableModule]
[ModuleDependency(typeof(WebModule))]
public class InitLocalization : IInitializableModule
{
    public void Initialize(InitializationEngine context)
    {
        ConfigurationContext.Setup(cfg =>
        {
            cfg.DiagnosticsEnabled = true;
            cfg.CustomAttributes = new[]
            {
                new CustomAttributeDescriptor(typeof(HelpTextAttribute))
            };
        });
    }

    public void Uninitialize(InitializationEngine context) { }
}
```

Or if you use localization provider outside of EPiServer site:

```csharp
public class Startup
{
    public void Configuration(IAppBuilder app)
    {
        app.UseDbLocalizationProvider(ctx =>
        {
            ctx.ConnectionName = "...";
            ctx.CustomAttributes = new[]
            {
                new CustomAttributeDescriptor(typeof(HelpTextAttribute))
            };
        });

        app.Map("/localization-admin", b => b.UseDbLocalizationProviderAdminUI());
    }
}

```

Now we added `[HelpText]` to localization provider list of custom attributes.
Attribute definition itself is nothing than simple C# class:

```csharp
public class HelpTextAttribute : Attribute
```

How you use new attribute? Simply decorate property with it:

```csharp
namespace MyProject.Models
{
    [LocalizedModel]
    public class HomeViewModel
    {
        [Display(Name = "User name:")]
        [UIHint("Username")]
        [HelpText]
        public string Username { get; set; }
    }
}
```

Resource key naming convention by default is the same as for other attributes - `$"{FQN of the property}-{attribute type name}"`. So new resource key for help text will be: `MyProject.Models.HomeViewModel.Username-HelpText`.

**NB!** Well actually resource key name depends on whether you registering resources for child classes within its own context or preserving base class context. Read more about these semantics [here](https://tech-fellow.eu/2016/11/03/playing-with-fire-localized-episerver-view-models/).


Now question is how we can access translation for this custom attribute resource?
I made few helper methods for easier retrieval.
Imagine that you are inside your field's (view model property) display or editor template and you need to retrieve this help text, a resource based on `[HelpText]` attribute. It's easy.

Index.cshtml:

```razor
...
@Html.EditorFor(m => m.Username)
```

Username.cshtml (because of `[UIHint]`):

```razor
@model string

...
<div>
    @Html.TranslateFor(m => m, typeof(HelpTextAttribute))
</div>
```

That's it. Pretty easy, ah?! :)

### Optional HelpText

With solution above there is a small catch. By default all properties where `[HelpText]` was used - resource translation will be generated (by default - last segment after `.` from the resource key).

However, another requirement in the project was to support *optional* resources - if translation is specified for the resource in specific culture - then visitor will see hint text, otherwise - no hint button should be generated.

With 1st version of custom attributes solution - this problem cannot be solved. So there has to be some adjustments.

When you are registering custom attribute descriptor - you can actually specify translation existence.

```csharp
[InitializableModule]
[ModuleDependency(typeof(WebModule))]
public class InitLocalization : IInitializableModule
{
    public void Initialize(InitializationEngine context)
    {
        ConfigurationContext.Setup(cfg =>
        {
            cfg.CustomAttributes = new[]
            {
                new CustomAttributeDescriptor(typeof(HelpTextAttribute), false)
            };
        });
    }

    public void Uninitialize(InitializationEngine context) { }
}
```

Note last parameter for `CustomAttributeDescriptor` constructor:


```csharp
{
    new CustomAttributeDescriptor(typeof(HelpTextAttribute), false)
}
```

Last parameter controls whether translation should (`true`) or should not (`false`) be generated for this resource.

By specifying `false` - resource itself will be discovered and registered, but translation will be empty. Which gives possibility for editors to add translation for that resource if needed.
This means that in your field template you can check for this and if translation is not empty, only then render markup for the hint.

Username.cshtml:

```razor
@{
    var hint = Html.TranslateFor(m => m, typeof(HelpTextAttribute));
}
@if (hint != MvcHtmlString.Empty)
{
    <div class="hint-text">@hint</div>
}
```

Needless to say, that `[Dislay(Description = "...")` attribute usage is [also supported](https://tech-fellow.eu/2016/06/06/episerver-dblocalizationprovider-fresh-2-0-is-out/#displaydescription) out of the box. So you can just look for description of the field inside your templates:

```razor
@ViewData.ModelMetadata.Description
```

Field description follows the same mechanics as custom attributes - you can leave it empty (`""`) and no translation will be added for that resource.

Happy localizing!

[*eof*]
