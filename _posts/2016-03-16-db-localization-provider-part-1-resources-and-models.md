---
title: DbLocalizationProvider - Part 1: Resources and Models
author: valdis
date: 2016-03-16 20:05:00 +0200
categories: [Add-On, .NET, C#, Localization Provider, Episerver, Optimizely]
tags: [add-on, .net, c#, open source, localization, episerver, optimizely]
---

# Resources and Models

This is first post about more details inside DbLocalizationProvider for EPiServer. In this post we will go through localization approach for arbitrary resources that can be used from the code to provide some message to the user or anywhere else where localization is required.
Another type of localization target is model. By model we can assume any kind of class that is used to render a page, or collect posted back data from the user. Usually models are decorated with various data annotation attributes indicating underlying data type or required validation procedures attached to the model properties.

## Why Different Types?

You may ask why need to differentiate these types as they seems to be the same - subject for the localization?
Main reason why these types need to be different is for the scanning and discovery process.
Localized resources are just a bunch of strings that you target for localization. They are similar to language files known today for EPiServer developers.
However - models are more complex subject for localization as additional metadata needs to be taken into account.

## Localized Resources

As described above localized resource is straight forward type that contains set of properties that you can use to provide simple message to the user or any other "consumer" that might be localized by editors.

Simple definition of the resource is following:

```csharp
namespace MyProject
{
    [LocalizedResource]
    public class MyPageResources
    {
        public static string ThisIsErrorMessage => "This is default value from code";
    }
}
```

Localized resource is simple POCO type that contains properties for localization.
Usage of this resource class is also straight forward (assuming that you want to translate some message in the Razor view):

```razor
@Html.Translate(() => MyProject.MyPageResources.ThisIsErrorMessage)
```

Localized resource scanning process that is executed at the app startup will look for types decorated with `[LocalizedResource]` and register it within localization storage (by default database configured under `EPiServerDB` connection string).

Key for the localized resource is calculated as `FQN` ("Fully Qualified Name") of each property.
After scanning process there should be new resource with key:

```
MyProject.MyPageResources.ThisIsErrorMessage
```

If property return type is `string` library will try to get value for resource translation and will save it for EPiServer's `ContentLanguage.PreferredCulture` while scanning and registration process executes. If `ContentLanguage.PreferredCulture` cannot be determined at time of scanning – translation will be registered for "English" language.

When `Html.Translate(() => ...)` method is invoked with `LambdaExpression` this localization extension method "calculates" FQN for given property (`MemberAccessExpression`) and resource with this key is searched in the storage.


### Nested Localized Resources

Localized resources are not limited to "single level" container properties. It means that you can design your resource hierarchy as you wish and what makes more sense for your project and editors.

So for instance you may have following structure for your resources:

```csharp
namespace MyProject
{
    [LocalizedResource]
    public class MyPageResources
    {
        public static string ThisIsErrorMessage => "This is default value from code";
        public static HeaderResources Header { get; }

        public class HeaderResources
        {
            public string HelloMessage => "Well, hello there!";
        }
    }
}

```

This approach avoids to "pollute" global namespace with types that have really narrow usages - only to group related properties under related parent resource. Scanning this kind of structure, following keys will be discovered:

```
MyProject.MyPageResources.ThisIsErrorMessage
MyProject.MyPageResources.Header.HelloMessage
```

Scanning process makes sure that resource keys for nested resources "follows" usage context.
**NB!** Note that there is no `static` keyword added to `HelloMessage` property in type `HeaderResources`.

For example, how you might use nested resource is following:

```razor
@Html.Translate(() => MyPageResources.Header.HelloMessage)
```

Last part of the `LambdaExpression` (e.g. `.HelloMessage`) is accessible ("compilable") **only if** property `HelloMessage` for type `HeaderResources` is marked as "instance property" (no `static` property).

If you define `static` property accessor for `HelloMessage` like this:

```csharp
namespace MyProject
{
    [LocalizedResource]
    public class MyPageResources
    {
        public static HeaderResources Header { get; }

        public class HeaderResources
        {
            public static string HelloMessage => "Well, hello there!";
        }
    }
}
```

Then usage ("path" to property) is different:

```razor
@Html.Translate(() => MyPageResources.HeaderResources.HelloMessage)
```

Which makes it difficult to find correct path to corresponding resources.
So library tries to follow usage of the resource from code perspective and makes it easier for editor and developer to understand the context and usage of the localized resource.

## Localized Models

What about localized models? Models are types that we tend to use as `ViewModels` in classical Mvc architecture.
These types are decorated with various data annotation attributes for display names, underlying data types or sometimes even `UIHints` to instruct Asp.Net Mvc pipeline how to render the page for this viewmodel, or how to validate incoming page postback represented with this viewmodel.

From localization resource scanning and discovery process perspective localized models need to be treated a bit differently. Scanning process needs to discover and register properties, its display names and related validation attributes.

This is pretty simple view model decorated with `[LocalizedModel]` for provider to recognize and register:

```csharp
namespace MyProject
{
    [LocalizedModel]
    public class MyViewModel
    {
        [Display(Name = "Username or email")]
        public string UserName { get; set; }
    }
}
```

In this case (when only `Display` attribute is added) only single resource key will be discovered:

```
MyProject.MyViewModel.UserName
```

After you decorate your viewmodel with `[LocalizedModel]` you can use all built-in Asp.Net Mvc helper methods to render your page:


```razor
@model MyViewModel

@Html.LabelFor(m => m.UserName)
```

### Localized Model Validation

As you might expect - all built-in Asp.Net Mvc model validation (or correctly would be to call it "Data Annotation Validation" as it's not Mvc specific) is supported as well.
You just need to add necessary validation attributes on top of the properties and you are ready to go:

```csharp
namespace MyProject
{
    [LocalizedModel]
    public class MyViewModel
    {
        [Display(Name = "Username or email")]
        [Required(ErrorMessage = "Username is required")]
        [StringLength(5)]
        public string UserName { get; set; }
    }
}
```

Following resource keys will be discovered:

```
MyProject.MyViewModel.Username                // for [Display]
MyProject.MyViewModel.Username-Required       // for [Required]
MyProject.MyViewModel.Username-StringLength   // for [StringLength]
```

As expected you can use built-in Mvc helpers to render your page:

```razor
@model MyViewModel

...
@Html.LabelFor(m => m.UserName)
@Html.ValidationMessageFor(m => m.UserName)
@Html.EditorFor(m => m.UserName)
```

Library will plugin its own model metadata providers and will make sure that proper resource is picked-up when Asp.Net Mvc pipeline will ask for metadata for particular model.

There are few configuration settings for DbLocalizationProvider library that you might use to control is and how model metadata providers are plugged in Asp.Net Mvc pipeline (will be covered in "Part 2: Configuration and Extensions").

### Nested Models

Single level, single hierarchy models rarely exist in real life. That's why nested models are no exception for DbLocalizationProvider library.

You may have seen this kind of view model:

```csharp
namespace MyProject
{
    [LocalizedModel]
    public class MyViewModel
    {
        [Display(Name = "Username or email")]
        [Required(ErrorMessage = "Username is required")]
        [StringLength(5)]
        public string UserName { get; set; }

        public AddressModel BillingAddress { get; set; }
    }

    [LocalizedModel]
    public class AddressModel
    {
        [Display(Name = "Line 1")]
        [Required]
        public string Line1 { get; set; }
    }
}
```

If nested model (`AddressModel`) is also decorated with `[LocalizedModel]` it will be included in scanning and discovery process and its properties will be registered in localization resource storage.

On sample model above following resource keys will be registered:

```
MyProject.MyViewModel.UserName
MyProject.MyViewModel.UserName-Required
MyProject.MyViewModel.UserName-StringLength

MyProject.AddressModel.Line1
MyProject.AddressModel.Line1-Required
```

Now you can use nested models and they are localized as well. Sample usage:

```razor
@model MyViewModel

@Html.LabelFor(m => m.UserName)
@Html.ValidationMessageFor(m => m.UserName)
@Html.EditorFor(m => m.UserName)

@Html.LabelFor(m => m.BillingAddress.Line1)
@Html.ValidationMessageFor(m => m.BillingAddress.Line1)
@Html.EditorFor(m => m.BillingAddress.Line1)
```

**NB!** Note that resource key names for property `BillingAddress` are different as it was for nested resources.
You might expect that resource key should be `MyProject.MyViewModel.BillingAddress.Line1`, but actually it's: `MyProject.AddressModel.Line1`.

This is related to how Asp.Net Mvc framework is building model metadata structures.
When you use following helper method:

```razor
@Html.LabelFor(m => m.BillingAddress.Line1)
```

metadata for nested property `Line1` of type `AddressModel` defined as property `BillingAddress` for type `MyViewModel` is required. Asp.Net Mvc model metadata provider infrastructure will give us `AddressModel` as "container type" for the requested `Line1` property. In other words from model metadata provider perspective Asp.Net Mvc does not care how model was declared in the view model (whether is has property name `BillingAddress` or `ShippingAddress`) - it's using *actual* type (`AddressModel`) as container type and not preserving "context" where it was declared.
That's why it's pointless to keep context about how property was declared as what matters is only actual model container type - the nested viewmodel.

### EPiServer Compatibility Mode

If you follow [Martin's approach](http://world.episerver.com/blogs/devabees/Dates/2014/3/Integrating-LocalizationService-with-MVC-DataAnnotations/) it's possible eventually to define your viewmodel with following attributes:

```csharp
namespace MyProject
{
    [LocalizedModel]
    public class MyViewModel
    {
        [Display(Name = "/mypageview/myviewmodel/username")]
        [Required(ErrorMessage = "/mypageview/myviewmodel/username-required")]
        [StringLength(5, ErrorMessage = "/mypageview/myviewmodel/username-length")]
        public string UserName { get; set; }
    }
}
```

However doing this, it's required to add bunch of adapters and later register them to the Asp.Net Mvc validation attribute adapter collection. I'm too lazy and I want everything to be working out of the box with no extra effort.

If you already have EPiServer [Language Files](http://world.episerver.com/documentation/Items/Developers-Guide/Episerver-CMS/9/Globalization/Localization-service/) then you have these keys in Xml files as well.

First step to migrate existing language files to new DbLocalizationProvider is to use [DbLocalizationProvider Migration Tool](http://nuget.episerver.com/en/OtherPages/Package/?packageId=DbLocalizationProvider.MigrationTool).

Once you have exported resources from Xml files and imported them into new provider you will have already resources with following keys (most probably among the other ones):

```
/mypageview/myviewmodel/username
/mypageview/myviewmodel/username-required
/mypageview/myviewmodel/username-length
```

Then upon 1st request to the application (when scanning and discovery process will kick in) your viewmodel with `[LocalizedModel]` annotation will be picked up, and all properties will be registered in resource storage.
So once application finished initialization, you will end up with following resource keys:

```
/mypageview/myviewmodel/username
/mypageview/myviewmodel/username-required
/mypageview/myviewmodel/username-length

MyProject.MyViewModel.UserName
MyProject.MyViewModel.UserName-Required
MyProject.MyViewModel.UserName-StringLength
```

It looks like duplication, but that's what library knows about your model.

Translations for new resources are taken either from `Name = "..."` property in case of `[DisplayAttribute]` or `ErrorMessage = "..."` in case of some kind of `ValidationAttribute`. Translations will be following:

```
MyProject.MyViewModel.UserName = "/mypageview/myviewmodel/username"
MyProject.MyViewModel.UserName-Required = "/mypageview/myviewmodel/username-required"
MyProject.MyViewModel.UserName-StringLength = "/mypageview/myviewmodel/username-length"
```

Once `LegacyMode` is enabled (it’s **enabled by default**, but you may turn it off – more info in “Part 2: Configuration and Extensions”), library will make sure that actual translation for property `UserName` of viewmodel `MyViewModel` is taken from EPiServer's `LocalizationService` with key `/mypageview/myviewmodel/username`. Which means that you don't need to make bunch of new translations to make new provider working, but instead new provider will try to work in legacy mode and will try to look for "XPath resource" as it's used in EPiServer if using built-in Xml language files.

### Additional Model Attributes

There are 2 additional attributes you can use to instruct scanning and discovery process how it's registering resources for your model:

* `[Include]` - attribute defined as `DbLocalizationProvider.Sync.IncludeAttribute`
* `[Ignore]` - defined as `EPiServer.DataAnnotations.IgnoreAttribute`

`Ignore` attribute is kind of self-documenting. Once applied to the property - you are instructing library not to register any resources associated with this property. It will be just completely ignored.

`Include` attribute usage is more interesting.
Let's say you have following viewmodel:

```csharp
namespace MyProject
{
    [LocalizedModel]
    public class MyViewModel
    {
        [Display(Name = "Username or email")]
        [Required(ErrorMessage = "Username is required")]
        [StringLength(5)]
        public string UserName { get; set; }

        public AddressModel BillingAddress { get; set; }
    }

    [LocalizedModel]
    public class AddressModel
    {
        [Display(Name = "Line 1")]
        [Required]
        public string Line1 { get; set; }
    }
}
```

And you would like to use following markup in your view (for instance creating section for billing address with title):


```razor
@model MyViewModel

...
<div class="block-section">
    <div class="title">@Html.DisplayNameFor(m => m.BillingAddress)</div>
    ...
</div>
...
```

However, you will end up with not localized `<div>` element content for the title.
By default library will not include "declaring property" localization resource, but instead resources for `AddressModel` will be registered. Usage for nested viewmodels is usually something like this:


Index.cshtml:

```razor
@model MyViewModel

...
@Html.LabelFor(m => m.UserName)
...

@Html.Partial("Address", m.BillingAddress)     @* or you can use any of partials approach *@
...
```

Address.cshtml:

```razor
@model AddressModel

@Html.LabelFor(m => m.Line1)
```

From this case, it's obvious that there is no need to "pollute" resource list with translation for "declaring property" of nested viewmodel.
But if you need to translate also "declaring property" you can do this by decorating this property with `[Include]` attribute:

```csharp
namespace MyProject
{
    [LocalizedModel]
    public class MyViewModel
    {
        ...

        [Include]
        public AddressModel BillingAddress { get; set; }
    }
}
```

This attribute will make sure that you have resource with key `MyProject.MyViewModel.BillingAddress` and then you can use this resource to localize "declaring property" with the following code:

```razor
@model MyViewModel

@Html.DisplayNameFor(m => m.BillingAddress)
```

## What's Next?

More blog posts in this series (*upcoming*):

* **Part 1: Resources and Models**
* [Part 2: Configuration and Extensions](https://tech-fellow.eu/2016/04/21/db-localization-provider-part-2-configuration-and-extensions/)
* [Part 3: Import and Export](https://tech-fellow.eu/2017/02/22/localization-provider-import-and-export-merge/)
* [Part 4: Resource Refactoring and Migrations](https://tech-fellow.eu/2017/10/10/localizationprovider-tree-view-export-and-migrations/)

If you have any ideas, suggestions or complaints please post them to library's [GitHub repo](https://github.com/valdisiljuconoks/LocalizationProvider/issues).

Happy localizing!

[*eof*]
