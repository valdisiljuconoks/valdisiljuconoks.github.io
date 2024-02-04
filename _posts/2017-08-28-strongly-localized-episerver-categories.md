---
title: Strongly Localized EPiServer Categories
author: valdis
date: 2017-08-28 13:30:00 +0200
categories: [Add-On, .NET, C#, Localization Provider, Episerver, Optimizely]
tags: [add-on, .net, c#, open source, localization, episerver, optimizely]
---

Initially [strongly typed localization provider](https://nuget.episerver.com/en/OtherPages/Package/?packageId=DbLocalizationProvider.EPiServer) was not planned to be used everywhere in EPiServer, but localizing [categories](https://webhelp.episerver.com/latest/cms-admin/editing-categories.htm?Highlight=categories) came as complimentary feature:

```csharp
[LocalizedResource(KeyPrefix = "/categories/")]
public class Categories
{
    [ResourceKey("category[@name=\"" + nameof(SampleCategory) + "\"]/description")]
    public static string SampleCategory => "Some Category !";
}
```

Still there is lot of ceremony to get things right..

Sick and tired of generating proper resource keys for localizing EPiServer categories? Say no more. Localization provider gives you now (within latest version) possibility to decorate your class with attribute and necessary translations will be added [automagically](https://en.wikipedia.org/wiki/Magic_(programming)).

Here is a sample category:


```csharp
namespace MyProject
{
    [LocalizedCategory]
    public class SomeCategory : Category { }
}
```

By adding `[LocalizedCategory]` attribute to your class, provider will generate following resource key:

```csharp
$"/categories/category[@name=\"{categoryName}\"]/description"
```

**NB!** Class you decorate with `[LocalizedCategory]` must inherit from `EPiServer.DataAbstraction.Category` class.

This is the key EPiServer will look for when [translating categories for UI](https://world.episerver.com/blogs/Linus-Ekstrom/Dates/2013/12/New-standardized-format-for-content-type-localizations/).
By default **class name** of the category type will be taken as translation for default culture.
You can change this by setting `Name` of the category (you might receive warning - that virtual member is used in constructor):

```csharp
namespace MyProject
{
    [LocalizedCategory]
    public class SomeCategory : Category
    {
        public SomeCategory()
        {
            Name = "Some Category !";
        }
    }
}
```

**Pst!** Thinking why categories should be defined in code and why my category class should be even inheriting from EPiServer category built-in class?? You definitely wanna check out [blog post](https://www.patrickvankleef.com/2016/03/28/strongly-typed-categories) and effort made by brilliant colleague of mine! **Patrick** made it possible to auto register discovered categories in EPiServer from code with no need to access UI to get the boring task done!

Hope this helps! Happy categorizing!

[*eof*]
