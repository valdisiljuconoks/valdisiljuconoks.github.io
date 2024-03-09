---
title: EPiServer Database Localization Provider - Released!
author: valdis
date: 2016-03-01 20:05:00 +0200
categories: [Add-On, .NET, C#, Localization Provider, Episerver, Optimizely]
tags: [add-on, .net, c#, open source, localization, episerver, optimizely]
---

Why on the Earth I would ever want to change my EPiServer localization system?
EPiServer provides you with a way to organize localization resources using set of Xml files. Working on multi-lingual projects and seeing how support teams are struggling and bored with changing resource value - thought: there has to be space for improvement..


### We don't need localization

Lately I've been working more and more with international sites where single language is not the case at all.

> Localization is not needed - because we have only single language.

In my opinion this is absolutely no excuse - even if project requires just single language, at some point editors will ask to change some of the texts somewhere on the website, change value of the resource. If texts are coded inside the views, services and controllers - does it mean that every time editor will ask for the change you may end up with new version roll-out?!


Example when taking localization into account is pretty important, even usage is in single language sites. This is extremely important if you are building library that is close to the core.

Localization of the EPiServer Commerce workflows..

![](/assets/img/2016/03/2016-02-27_20-15-17-1.png)

I want to show precise error message what happened in workflow to the end-user and also give editors possibility to change this message to anything meaningful for end-users.

And I don't want to download workflow source code project, rebuild it and use my own version only because my site is multi-lingual.

### Noise in the Model

Yes.. I've been there and made my own set of overridden `DataAnnotation` attributes to be applied for the model in order to get localized strings out of `LocalizationService` and pass them back to Mvc model meta data provider for further usage.

```csharp
public class LoginViewModel
{
    [LocalizedDisplay(Name = "/login/viewmodel/username")]
    [LocalizedRequired(ErrorMessage = "/login/viewmodel/username-required")]
    public string UserName { get; set; }

    ...
}
```

This of course does not look sexy and there is a lot of "noise" in the model. I don't see *the model* any more, I see some stuff around it that makes sure proper error message is shown on the form. It's stuff for [adapter or port](http://blog.ploeh.dk/2013/12/03/layers-onions-ports-adapters-its-all-the-same/) not on my level, I don't need it here - my domain. Validation attributes is something that's interesting for the domain modeler.

### Eh.. Can You change this?

Even working on single language sites I often receive requests to change text from "`xxx`" to "`yyyy`".

Then after a while:

-- *Can you change this text "yyyy" now to "zzzz", please?*

-- Of course, no problem.

..

-- *Everything looks OK, but you know - we found some grammar mistake. Can you change this again?*

..

-- *You know, is it possible for us to change these localization resources ourselves?*

-- Of course, here you go a Xml file you will need to translate.

![](/assets/img/2016/03/2016-02-27_20-47-00.png)

-- *Whew... We spent whole day translating those damn resources. But anyway -> here you go!*

![](/assets/img/2016/03/2016-02-27_20-52-03.png)

..

-- *And by the way, how can we overview all our translation resources and see what messages are not translated and which have wrong translations?*


### Attempts to Solve this

Martin Pickering [tried to squeeze in MVC Data Annotation subsystem](http://world.episerver.com/blogs/devabees/Dates/2014/3/Integrating-LocalizationService-with-MVC-DataAnnotations/) to fetch resources out of Localization Service and localize either display name or validation messages during runtime.

This makes it possible to have more "pure" models in your code base:

```csharp
public class LoginViewModel
{
    [Display(Name = "/login/username")]
    [Required(ErrorMessage = "/login/username-required")]
    public string UserName { get; set; }
}
```

However - I find it still "noisy" enough to continue searching for better solution.
I know that I'm in LoginViewModel, I know that I'm on the `UserName` property and also I know that I need to show `DisplayName` for this property if somebody will ask me `@Html.DisplayNameFor(m => m.UserName)` or even when asking for validation message `@Html.ValidationMessageFor(m => m.UserName)` - I know that this property is required and maybe has some other .

One of the greatest plugins seen so far is from my old fellow and friend and EMVP - Jeroen Stemerdink - [EPi.Libraries.Localization](https://github.com/jstemerdink/EPi.Libraries.Localization/tree/master/EPi.Libraries.Localization).

However, Jeroen's solution is ideal when it comes to editor story - editing UI, permissions, search everything is in sort in place already. But I missed a bit developer experience story here - how I as a developer am able start generating new resources, how do I easily reference these resources in my services, etc.

I was trying to solve these challenges and put localization provider on steroids - in this new **Database Driven Localization Provider** (`DbLocalizationProvider`).

## Getting Started
Localization Provider consists from few components:

* `DbLocalizationProvider` - this is core package and gives you EPiServer localization provider, `DataAnnotation` attributes, model and resource synchronization process and other core features.
* `DbLocalizationProvider.AdminUI` - administrator user interface for editors and administrators to overview resources, make translations, import / export and do other management stuff.
* `DbLocalizationProvider.MigrationTool` - tool gives you possibility to generate JSON format data out of Xml language files later to import these resources into this new DbLocalizationProvider.


### Installing Provider

Installation nowadays can't be more simpler as just adding NuGet package(s).

```
PM> Install-Package DbLocalizationProvider
PM> Install-Package DbLocalizationProvider.AdminUI
PM> Install-Package DbLocalizationProvider.MigrationTool
```

**NB!** Currently `DbLocalizationProvider` NuGet package has naïve `web.config` transformation file assuming that `<episerver.framework>` section is not extracted into separate file (this was usual case for older versions of AlloyTech sample sites).

New localization provider needs to be "registered" and added to the list of the localization providers configured for EPiServer Framework. Section may look like this:

```xml
<episerver.framework>
  ..
  <localization>
    <providers>
      <add name="db"
           type="DbLocalizationProvider.DatabaseLocalizationProvider, DbLocalizationProvider" />
      <add name="languageFiles" virtualPath="~/lang"
           type="EPiServer.Framework.Localization.XmlResources.FileXmlLocalizationProvider, EPiServer.Framework" />
    </providers>
  </localization>
  ..
</episerver.framework>
```

**NB!** If you do have extracted `<episerver.framework>` section into separate file (usually `episerver.framework.config`), please clean-up web.config after NuGet package installation and move content of the `<localization>` section to that separate physical file.


## DbLocalizationProvider Features

All resources in DbLocalizationProvider system are divided into 2 groups:

* **Resources** - localized resources are just list of key / value pairs. You may have a key for the resource and value is its translation in specific language. Resources are designed as just POCO objects.
* **Models** - models are usually view models that may have `DataAnnotation` attributes attached to them (like, `Display`, `Required`, etc).


### Localized Resources

Localized resource is straight forward way to define list of properties that are localizable. Localized resource is simple POCO class that defines list of properties:

```csharp
namespace MySampleProject {

    [LocalizedResource]
    public class MyResources
    {
        public static string SampleResource => "This is default value";
    }
}
```

So what happens now is that localization provider knows that there is a resource that needs to be registered in data storage. This scanning and discovery of the types is done during startup (more about this will be in "*Part 1: Model and Resource Discovery Process*").

Localization Provider scanning process looks for `[LocalizedResource]` attributes and registers these classes in its internal type list for further inspection and discovery.

If we take a look at resource class definition key (unique identifier of the resource) is defined as: `{namespace}.{container-type}.{property-name}`. So in this case key of the resource is: `MySampleProject.MyResources.SampleResource`.

Now when you have your resource registered and discovered - default value for the preferred content language (`ContentLanguage.PreferredCulture` value during application startup) is `"This is default value"`. Later of course editors via AdminUI will be able to change this.

Now, you may use one of the ways to output this resource to the end-users:

```razor
@using DbLocalizationProvider

<div>
    @Html.Translate(() => MyResources.SampleResource)
</div>
```


### Nested Localized Resources

Btw, these kind of resource definitions are also supported:

```csharp
namespace MySampleProject {

    [LocalizedResource]
    public class MyResources
    {
        public static string SampleResource => "This is default value";
        public static SubResource AnotherSet { get; set; }
    }

    public class SubResource
    {
        public string AnotherResource => "Please, translate!";
    }
}
```

Now you will have 2 resources discovered with following keys:

```
MySampleProject.MyResources.SampleResource
MySampleProject.MyResources.AnotherSet.AnotherResource
```

Now you can access this like following:

```razor
@using DbLocalizationProvider

<div>
    @Html.Translate(() => MyResources.AnotherSet.AnotherResource)
</div>
```

This is why during resource scanning process, library "preserves" nested resource context (`AnotherSet.AnotherResource`) and not setting resource key as `SubResource.AnotherResource` - to match with used `LambdaExpression` to reach that resource via `HtmlHelper` or any other usage.


More about this will be in up coming post - "*Part 1: Model and Resource Discovery Process*".

### Localized Models

Another more interesting way and usage is to define localizable view models.

Localizable view model means that `DbLocalizationProvider` library will search for `[LocalizedModel]` attributes and will discover all models and further discovery of the resources there.

Now you may define following view model:

```csharp
namespace MySampleProject {

    [LocalizedModel]
    public class MyViewModel
    {
        [Display(Name = "This is default value")]
        public string SampleProperty { get; set; }
    }
}
```

Now you will have similar resource discovered as it was with `[LocalizedModel]`:

```
MySampleProject.MyViewModel.SampleProperty
```

Default value for the resource will be `"This is default value"` (this is taken from `[Display]` attribute).

**NB!** If there will be no `[Display]` attribute for the property - name of the property (`SampleProperty`) will be used as default resource value.

### Model Data Annotations

Usually view models are decorated with various `DataAnnotation` attributes to get model validation into the Asp.Net Mvc request processing pipeline. Which is very fine and `DbLocalizationProvider` aware of these attributes once scanning for localized models.

So if you add bit more attributes to initial view model:

```csharp
namespace MySampleProject {

    [LocalizedModel]
    public class MyViewModel
    {
        [Display(Name = "This is default value")]
        [Required]
        [StringLength(5)]
        public string SampleProperty { get; set; }
    }
}
```

Following resources will be discovered:

```
MySampleProject.MyViewModel.SampleProperty
MySampleProject.MyViewModel.SampleProperty-Required
MySampleProject.MyViewModel.SampleProperty-StringLength
```

Which gives you a possibility to translation precise error messages shown when particular property has invalid value for this model.

So you can easily continue using Html helper extensions like:


```razor
<div>
    @Html.LabelFor(m => m.SampleProperty)
    @Html.EditorFor(m => m.SampleProperty)
</div>
```

More about this will be in up coming post - "*Part 1: Model and Resource Discovery Process*".


Now if I take a look at one of my view model before and after adding `DbLocaliztionProvider` - seems like new version of the model is much more cleaner and readable.

![](/assets/img/2016/03/2016-02-25_15-15-35.png)

## Import Language Xml Files

This is a natural situation when adding `DbLocalizationProvider` to the project - there already will be Xml language files present.

Even after `DbLocalizationProvider` library is added - old Xml language files will continue to work. However - if you would like to get rid of language files completely - you will need to migrate them to the database.

Migration may happen in 2 ways:

* via SQL script generated by migration tool
* sometimes direct access (via `SqlConnection` object) may not be applicable. That's why it's possible to extract all Xml language file resources and later import them into database via Administration UI (AdminUI).


### Migration with SQL script

Migration via SQL script is pretty simple. First of all you need to install `DbLocalizationProvider.MigrationTool` package. Then you can execute `.exe` file inside package `tools/` folder.


```
> DbLocalizationProvider.MigrationTool.exe -s="{path-to-project}" -e -i
```

Explanation of switches used:

* `-s` - specifies **s**ource directory for the project (your web root)
* `-e` - asks migration tool to **e**xport all resources from language Xml files into SQL format file (resulting file `localization-resource-translations.sql` by default is located in the same folder specified by `-s`)
* `-i` - asks migration tool to **i**mport resources from exported file into directly database (using connection string named `"EPiServerDB"`)

![](/assets/img/2016/03/2016-02-27_22-48-32.png)


### Migration via JSON file

Using JSON format export file is similar as with SQL script file, you just need to provide switch to use JSON file (import from AdminUI is supported only with JSON format files).


```
> DbLocalizationProvider.MigrationTool.exe -s="{path-to-project}" -e --json
```

Commands:

* `-s` - specifies **s**ource directory for the project (your web root)
* `-e` - asks migration tool to **e**xport all resources from language Xml files (resulting file `localization-resource-translations.json` by default is located in the same folder specified by `-s`)
* `--json` - to use JSON format

![](/assets/img/2016/02/2016-02-27_22-54-11.png)

Now you can use result JSON file and import it from AdminUI.


## Manage Resources in AdminUI

There are 2 type of actors in localization module administration UI:

* **Editors** - users in this role can access resources, change translations and do some basic setup of the AdminUI (like, pick only languages and translations in interest). Following roles classifies as editors - `WebEditors`, `CmsEditors`, `LocalizationEditors`.
* **Administrators** - users in this role has access to more features in AdminUI, like Import/Export, create new or delete manually resource, etc. Users in these roles classifies as administrators - `Administrators`, `CmsAdmins`, `WebAdmins`, `LocalizationAdmins`.


## See it in Action!

Few sample videos to see new provider in action.

Simple resource definition:

<iframe src="https://player.vimeo.com/video/156990357" width="800" height="520" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe>

Simple View Model definition:

<iframe src="https://player.vimeo.com/video/157056409" width="800" height="520" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe>

Newly added language branch awareness:

<iframe src="https://player.vimeo.com/video/157062616" width="800" height="520" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe>


## Where do I get it?

Nothing is more easy as installing [NuGet package](http://nuget.episerver.com/en/OtherPages/Package/?packageId=DbLocalizationProvider) :)

## Read More
More about new EPiServer Localization Provider in this blog post series (*upcoming*):

* [Part 1: Resources and Models](https://tech-fellow.eu/2016/03/16/db-localization-provider-part-1-resources-and-models/)
* [Part 2: Configuration and Extensions](https://tech-fellow.eu/2016/04/21/db-localization-provider-part-2-configuration-and-extensions/)
* [Part 3: Import and Export](https://tech-fellow.eu/2017/02/22/localization-provider-import-and-export-merge/)
* [Part 4: Resource Refactoring and Migrations](https://tech-fellow.eu/2017/10/10/localizationprovider-tree-view-export-and-migrations/)


Library is open sourced and and you can find it @ [GitHub](https://github.com/valdisiljuconoks/LocalizationProvider).


Happy localizing!

[*eof*]
