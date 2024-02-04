---
title: Localization Provider - Import and Export (Merge)
author: valdis
date: 2017-02-23 00:35:00 +0200
categories: [Add-On, .NET, C#, Localization Provider, Episerver, Optimizely]
tags: [add-on, .net, c#, open source, localization, episerver, optimizely]
---

It's been awhile since last update for **DbLocalizationProvider** for EPiServer. Anyway things been in my backlog. Period around New Year is not the most productive for me :) This blog post will describe some of the interesting stuff I've been lately busy with.

## Export & Import - View Diff

Once upon a time I got sick of merging and importing new resources from staging environment (with adjusted translations by editors) into production database where on the other hand editors already made changes to existing resources. Up till now import process only supported *partial import* or *full import*. Partial import was dealing with only new resources (those ones that were not in target database), however full import - on the other hand - was used to flush everything and import stuff from exported file. Kinda worked, but it was v0.1 of this export/import component. Pretty green.

Updates to this story was one of the main focus for this upcoming version. Eventually if you are importing resources from other environment, now your import experience might look like this:

![](/assets/img/2017/02/dblocexportimport.png)

To share the code would be overkill here in this post. Better sit back and relax watching import in action. Enjoy!

<iframe src="https://player.vimeo.com/video/205294678" width="800" height="520" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe>
<p><a href="https://vimeo.com/205294678">DbLocalizationProvider - Export_Import</a> from <a href="https://vimeo.com/user49426707">Valdis Iljuconoks</a> on <a href="https://vimeo.com">Vimeo</a>.</p>

## Resource Keys for Enum

Sometimes (usage for me was EPiServer visitor groups) you just need to control naming for each individual `Enum` item. Now you can do that with `[ResourceKey]` attribute. Hope code explains how it might look:

```csharp
[LocalizedResource(KeyPrefix = "/this/is/prefix/")]
public enum ThisIsMyStatus
{
    [ResourceKey("nothing")]
    None,

    [ResourceKey("something")]
    Some,

    [ResourceKey("anything")]
    Any
}
```

For instance you have following visitor group criteria:

```csharp

namespace My.Project.Namespace
{

[VisitorGroupCriterion(..,
    LanguagePath = "/visitorgroupscriterias/usernamecriterion")]
public class UsernameCriterion : CriterionBase<UsernameCriterionModel>
{
    ...
}

public class UsernameCriterionModel : CriteriaPackModelBase
{
    [Required]
    [DojoWidget(SelectionFactoryType = typeof(EnumSelectionFactory))]
    public UsernameValueCondition Condition { get; set; }

    public string Value { get; set; }
}

public enum UsernameValueCondition
{
    Matches,
    StartsWith,
    EndsWith,
    Contains
}
```

So EPiServer will look after resources with following key:

```
/enums/my/project/namespace/usernamecriterion/usernamevaluecondition
```

and then name of the `Enum`. So you can control this and decorate each enum member with `ResourceKey` attribute to generate specific keys if needed.

```csharp
namespace My.Project.Namespace
{
    [LocalizedResource(KeyPrefix = "/enums/my/project/namespace/usernamecriterion/usernamevaluecondition")]
    public enum UsernameValueCondition
    {
        [ResourceKey("matches")]
        Matches,
        [ResourceKey("startswith")]
        StartsWith,
        [ResourceKey("endswith")]
        EndsWith,
        [ResourceKey("contains")]
        Contains
    }
}
```

Maybe it's worth just to create new attribute - like `[EPiServerEnumResource]` or something - namespace and member resource key calculations would be done for me?! I know - I'm lazy.. Sorry..

## Class Fields

This is smaller update, but still important. Developer writing code below would expect at least one resource to be generated. But it would not happen (with older library versions).

```csharp
namespace My.Project.Namespace
{
    [LocalizedResource]
    public class MyResources
    {
        public static PageHeader = "This is page header";
    }
}
```

Seems legit, but issue here is that `PageHeader` is class field and not property.
Thanks to my [colleague](https://github.com/klavsi) who discovered this. He just forgot to add "`>`" symbol at the end of the assignment to convert from field to property:

```csharp
namespace My.Project.Namespace
{
    [LocalizedResource]
    public class MyResources
    {
        public static PageHeader => "This is page header";
    }
}
```

Now this has been fixed and class fields are also supported. Static and instance ones.

## Distributed Concurrency Issue

Everything went great and localization provider proved to be solid and developer friendly approach to localize EPiServer (and not only EPiServer - provider [runs outside of it just fine](https://www.nuget.org/packages/LocalizationProvider/)) applications. Until we deployed application with new pending resources to Azure deployment slot with more than 1 instance.. At the end duplicate resource keys were registered. Thanks to another [colleague](https://github.com/ajuris) for giving some hints. So over the weekend while testing **10** concurrent instances with **10,000** resources - I came also to some performance issues. However, not sure if this might ever be actual real life case tho... At least stress testing gave some ideas how to improve performance in "normal" cases with just a few hundred resources. Startup performance boost average is about **30%**.

I'm still struggling a bit with the best approach for distributed concurrency and how to properly address it, but at least - duplicate resource keys (which would be disaster) should not be registered.

## More info

* [Part 1: Resources and Models](https://tech-fellow.eu/2016/03/16/db-localization-provider-part-1-resources-and-models/)
* [Part 2: Configuration and Extensions](https://tech-fellow.eu/2016/04/21/db-localization-provider-part-2-configuration-and-extensions/)
* **Part 3: Import and Export**
* [Part 4: Resource Refactoring and Migrations](https://tech-fellow.eu/2017/10/10/localizationprovider-tree-view-export-and-migrations/)

Happy localizing!

[*eof*]
