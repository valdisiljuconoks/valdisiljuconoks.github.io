---
title: Localized Episerver Enum Properties
author: valdis
date: 2018-12-29 00:30:00 +0200
categories: [Add-On, .NET, C#, Episerver, Optimizely, Localization Provider]
tags: [add-on, .net, c#, open source, episerver, optimizely, localization]
---

Time to time I receive questions on how to properly translate `System.Enum` based properties on Episerver using DbLocalizationProvider package.

The support has been around for quite while, but I've never really blogged about it properly. So thought this is a good time to finish-up this amazing year and write how to utilize built-in support from the package.

Let's assume that you do have a page defined as following:

```csharp
[ContentType(DisplayName = "...", GUID = ".."]
public class StartPage : PageData
{
    ...

    [BackingType(typeof(PropertyNumber))]
    public virtual SomeValuesEnum SomeValue { get; set; }
}

public enum SomeValuesEnum
{
    None = 0,
    FirstValue = 1,
    SecondValue = 2,
    ThirdOne = 3
}
```

This is pretty straight forward approach how to do the enumeration based property definition in Episerver. However, when it comes to how UI is localized and how Episerver is able to get proper display values from the drop-downs - it might get a bit tricky to line up all parts.

DbLocalizationPackage however, is here to rescue you an ease this process with couple of decorations and that's it.

1) you will need to decorate enumeration type with `[LocalizedResource]` attribute for the package to pick it up and register resources and corresponding translations.

```csharp
[LocalizedResource]
public enum SomeValuesEnum
{
    [Display(Name = "NOONE!")]
    None = 0,

    [Display(Name = "1st value")]
    FirstValue = 1,

    [Display(Name = "This is second")]
    SecondValue = 2,

    [Display(Name = "And here comes last (3rd)")]
    [TranslationForCulture("Tredje", "no")]
    [TranslationForCulture("Third (EN)", "en")]
    ThirdOne = 3
}
```

You can use [any of the supported attributes](https://github.com/valdisiljuconoks/LocalizationProvider/blob/master/docs/translate-enum-net.md?ref=tech-fellow.ghost.io) here to customize translations for each enumeration value and even provide alternative translation for specific languages.

2) You will need to decorate property itself on page model class to indicate Episerver, that this is a bit special property and needs some attention before rendering (decorate with `[LocalizedEnum]` attribute):

```csharp
[ContentType(DisplayName = "...", GUID = ".."]
public class StartPage : PageData
{
    ...

    [LocalizedEnum(typeof(SomeValuesEnum))]
    [BackingType(typeof(PropertyNumber))]
    public virtual SomeValuesEnum SomeValue { get; set; }
}
```

What's happening behind the scene is that `[LocalizedEnum]` attribute is actually metadata provider for the Episerver (inheriting from `[SelectOneAttribute]` attribute).

```csharp
public class LocalizedEnumAttribute : SelectOneAttribute, IMetadataAware
{
    public LocalizedEnumAttribute(Type enumType)
    {
        EnumType = enumType ?? throw new ArgumentNullException(nameof(enumType));
    }

    public Type EnumType { get; set; }

    public new void OnMetadataCreated(ModelMetadata metadata)
    {
        SelectionFactoryType = typeof(LocalizedEnumSelectionFactory<>).MakeGenericType(EnumType);
        base.OnMetadataCreated(metadata);
    }
}
```

It instructs Episerver to use generic `LocalizedEnumSelectionFactory` type when rendering values on Episerver UI.

![](/assets/img/2018/12/2018-12-28_22-47-30.png)

And selection factory is even more straight forward by just iterating over given enumeration type and translating each value.

```csharp
public class LocalizedEnumSelectionFactory<TEnum> : ISelectionFactory where TEnum : struct
{
    public IEnumerable<ISelectItem> GetSelections(ExtendedMetadata metadata)
    {
        var values = Enum.GetValues(typeof(TEnum))
                         .Cast<Enum>();

        foreach(var value in values)
        {
            yield return new SelectItem
                         {
                             Text = value.Translate(),
                             Value = value
                         };
        }
    }
}
```

And while writing this post I just realized that there were no support for multi choice (aka `[SelectMany]` attribute). So I just added that to the package. So if you need multiple-selection support as well, please wait couple days while package being approved.

To enable multi-selection, just provide `true` as second argument for the attribute:

```csharp
[ContentType(DisplayName = "...", GUID = ".."]
public class StartPage : PageData
{
    ...

    [LocalizedEnum(typeof(SomeValuesEnum), true)]
    [BackingType(typeof(PropertyNumber))]
    public virtual SomeValuesEnum SomeValue { get; set; }
}
```

Happy localizing!

[*eof*]
