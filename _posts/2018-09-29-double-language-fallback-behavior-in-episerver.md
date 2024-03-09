---
title: Double Language Fallback Behavior in Episerver
author: valdis
date: 2018-09-29 17:15:00 +0200
categories: [Add-On, .NET, C#, Localization Provider, Episerver]
tags: [add-on, .net, c#, open source, localization, Episerver]
---

And here we are again to talk a little bit about localization stuff. Most probably you know already there are pretty neat settings when it comes to localization of the Episerver and how one should behave when resource translation is missing for requested language (or culture to be precise). You can define fallback behavior, fallback culture and maybe in the future even something else.

```xml
...
  <localization fallbackBehavior="FallbackCulture">
    <providers>
      <add name="db" type="DbLocalizationProvider.EPiServer.DatabaseLocaliza
      <add name="languageFiles" virtualPath="~/Resources/LanguageFiles" type
    </providers>
  </localization>
</episerver.framework>
```

At the same time, `DbLocalizationProvider` library also allows you to configure some defaults and fallback logic. Working only with invariant culture (you can set it up to do so) could come handy when you don't want to mess around with database and reduce resource table content. In cases when you assume that translations provider in code are invariant culture (even if texts are in English for example), you don't want to duplicate the same values and register also the same texts in English culture. Therefore, you can configure library to work with invariant culture as default culture and also perform fallback on invariant culture, when someone is looking for non-existing or not yet translated language.

```csharp
[InitializableModule]
[ModuleDependency(typeof(InitializationModule))]
public class InitLocalization : IInitializableModule
{
    public void Initialize(InitializationEngine context)
    {
        ConfigurationContext.Setup(_ =>
        {
            _.EnableInvariantCultureFallback = true;
            _.DefaultResourceCulture = CultureInfo.InvariantCulture;
            ...
        });
    }
}
```

If we check how Episerver is treating `<localization>` configuration element, we can see that actually there is no invariant culture feedback (and it makes sense when you think about fallback behavior in content delivery context - what is invariant culture for the content? .. an empty page, English content or maybe master language content?!)

```csharp
[ConfigurationProperty("fallbackCulture", DefaultValue = "en", IsRequired = false)]
public CultureInfo FallbackCulture { get { ... } set { ... }}
```

We can see that even Episerver is defaulting back to `en` language when one is not explicitly set.

So where is the problem?

## Experiment with InvariantCulture Fallback

Anyway, we can set fallback culture to empty string, and this indeed results in fallback culture being set to `CultureInfo.InvariantCulture`. I would expect things to be working just fine..

![](/assets/img/2018/09/2018-09-28_14-07-18.jpg)

Episerver's `LocalizationService` checks for the *real* fallback culture in `GetActualFallbackCulture` method. Regardless whether you ask for first level culture translations (like `lv`) or second level culture with region (like `lv-LV`), invariant culture will be "calculated" as fallback culture (because `this.FallbackCulture`  points to invariant):

```csharp
private CultureInfo GetActualFallbackCulture(
        FallbackBehaviors fallbackBehavior,
        CultureInfo requestedCulture)
{
    if(fallbackBehavior.HasFlag(FallbackBehaviors.FallbackCulture)
       && requestedCulture != this.FallbackCulture
       && requestedCulture.Parent != this.FallbackCulture)
    {
        return this.FallbackCulture;
    }

    return CultureInfo.InvariantCulture;
}
```

After Episerver's localization service has detected actual fallback culture for this resource translation lookup, translation retrieval is being performed in `LocalizationService.TryGetStringByCulture` method.

Essentially this method "climbs" up the culture "inheritance" path (well, that's not entirely inheritance, but child/parent relationships) and tries to figure out in which culture this translation exists. So if I'm asking for `lv-LV` culture resource and there is none, Episerver's localization service will look for `lv` culture as well, because it's parent of `lv-LV`. Having a following code fragment:

```csharp
...
bool flag = false;
for (; !CultureInfo.InvariantCulture.Equals(culture); culture = culture.Parent)
{
    // calls underlying provider for actual string in requested language
    ...

    if (localizedString != null)
        return true;

    if (culture.Equals(fallbackCulture))
        flag = true;
}

if (!flag && !CultureInfo.InvariantCulture.Equals(fallbackCulture))
{
    // final stage - load translation in configured fallback language
}

...
```

As you can see, Episerver will not try to fallback to `CultureInfo.InvariantCulture` even if configured so in `<localization>` element.

What's even worse, is that if we go back to `GetStringByCulture` method which is handling the whole retrieval process, there we can see code what happens when resource translation is not being found in requested or fallback culture (in our case - fallback culture is invariant and will not be even taken into account):

```csharp
public virtual string GetStringByCulture(string resourceKey, FallbackBehaviors fallbackBehavior, string fallback, CultureInfo culture)
{
   var actualFallbackCulture = GetActualFallbackCulture(fallbackBehavior,
                                                        culture);
   if (TryGetStringByCulture(resourceKey,
                             culture,
                             actualFallbackCulture,
                             out var localizedString))
       return localizedString;

   return GetMissingFallbackResourceValue(fallbackBehavior,
                                          fallback,
                                          resourceKey,
                                          culture);
}
```

Code basically tells you, if neither requested (or any of its parent) nor fallback culture have translation for resource you are looking, I'm going to return you "a missing fallback resource value":

```csharp
private string GetMissingFallbackResourceValue(FallbackBehaviors fallbackBehavior,
                                               string fallback,
                                               string resourceKey,
                                               CultureInfo culture)
{
    if (fallback != null)
        return fallback;

    if (fallbackBehavior.HasFlag(FallbackBehaviors.Echo))
        return resourceKey;

    if (fallbackBehavior.HasFlag(FallbackBehaviors.MissingMessage))
        return LocalizationService.GetMissingMessage(resourceKey, culture);

    if (fallbackBehavior.HasFlag(FallbackBehaviors.Null))
        return null;

    return string.Empty;
}
```

This code means if there is no specific Episerver localization configuration to return `null` values for the missed fallback (`FallbackBehaviors.Null`) - then empty string is returned. This makes perfect sense if you think about it - framework does not want you to have zillions of null reference exceptions all over the place where you didn't expect it.

For the DbLocalizationProvider package - there has to be somehow a way to detect that resource translation is not being found, and built-in fallback also failed - so invariant translation should be fetched if configured so.

Everything would be great in cases when Episerver localization is configured to return `null` in case of no translation. Also built-in `ProviderBasedLocalizationService` does checking on return value for each provider it's calling, and if return value is `null` it assumes that resource translation has not been found and continues to the next provider in the list. So the `string.Empty` is last return from the Episerver perspective.

But as a library author, I cannot rely on fact that site should be configured first to get fallback working. I would like to have a feature which would work in default configuration as well. There has to be way around this.

## DbLocalizationProvider Double Fallback Feature

Originally fallback to invariant culture was introduced in pure Asp.Net Mvc app context. But being a member of the platform, you have to behave well and need to respect surrounding behaviors and assumptions. So with some gentle pushes from colleagues think I need to support this fallback to invariant culture also when DbLocalizationProvider library is used in Episerver sites.

### CQS and Friends

W$%&.tfh heck CQS has common with Episerver and localization, you may ask? I still have pending blog post about how localization provider library is built internally and how the same piece of code is reused (with slight differences sometimes) between Episerver, Asp.Net Mvc and Asp.Net Core applications.. But that's a different story.

In this context, keeping long story short, basically there are 2 entries for the DbLocalizationProvider when someone is trying to get string in correct culture. By default, if one is using Episerver built-in localization service, call will end up at `DbLocalizationProvider.EPiServer.DatabaseLocalizationProvider` type, which plugged in common list of localization translation providers via `<localization>` element in configuration file.

This `DatabaseLocalizationProvider` will invoke then something called `GetTranslationQuery` which is basically player from CQS architecture - where commands, queries (sometimes also events) and relevant handlers are mix-matched all together.

On the another hand, there are a lot of various helpers and other extension methods that are not Episerver specific (for example, `typeof(Enum).Translate(..)`). These helper methods are located in shared part of the library - thus could be called within Episerver application context as well. Therefore, we do have `LocalizationProvider` type. Responsibility for this type is to resolve and return resource translations for various situations.

In order to play well together with Episerver and respect framework's settings regarding culture fallback, `LocalizationProvider` from shared library part is not calling directly underlying services, but instead - executes `GetTranslationQuery`. Who is going to handle this query and return translation - now depends on how DbLocalizationProvider library is setup. In Episerver context there is dedicated setup & init module that does this for you and appropriate Episerver specific translation retrieval handler is registered (instead of built-in one for example).

I know, it's hard to explain in words, that's why I'm saving this topic for dedicated blog post. Think it's worth sharing.

### Episerver's LocalizationProvider Fix Supporting Double Fallback

When resource translation is asked via Episerver built-in localization service, one of the providers in the list is also `DatabaseLocalizationProvider`. This type then could be responsible for double fallback checking and supporting also scenarios with invariant cultures.

And as it turned out, it's not actually that hard to implement and support double fallback.

```csharp
public override string GetString(string originalKey,
                                 string[] normalizedKey,
                                 CultureInfo culture)
{
    var foundTranslation = _originalHandler.Execute(new GetTranslation.Query(originalKey,
                     culture,
                     false));

    if(foundTranslation == null
       && LocalizationService.Current.FallbackBehavior.HasFlag(FallbackBehaviors.FallbackCulture)
       && ConfigurationContext.Current.EnableInvariantCultureFallback
       && (Equals(culture, LocalizationService.Current.FallbackCulture) || Equals(culture.Parent, CultureInfo.InvariantCulture)))
    {
        return _originalHandler.Execute(new GetTranslation.Query(originalKey, CultureInfo.InvariantCulture, false));
    }

    return foundTranslation;
}
```

And now tell me slowly, ok?! :)

So what happens here:

- first, database localization provider tries to resolve translation in the culture that has been asked
- if that fails (`foundTranslation == null`) and configured fallback behavior has configuration to fallback to different culture, and it's also configured in DbLocalizationProvider package, and requested culture is either fallback culture itself or parent of the requested culture is invariant culture - then library will explicitly look for invariant culture translation
- otherwise - result is returned as-is, meaning that library does not care either translation has been found or not.

### Built-in LocalizationProvider Fix

At the same time unfortunately library's built-in localization provider should be fixed as well, because there are definitely cases when built-in is called directly skipping Episerver's one - so that part has to be fixed as well.

When resource is retrieved via built-in provider, eventually `EPiServerGetTranslationHandler` is executed. So sounds like this is perfect place for the fix.

```csharp
public class Handler : IQueryHandler<GetTranslation.Query, string>
{
    private readonly LocalizationService _service;

    public Handler(LocalizationService service)
    {
        _service = service;
    }

    public string Execute(GetTranslation.Query query)
    {
        var foundTranslation = _service.GetStringByCulture(query.Key, query.Language);

        if(string.IsNullOrEmpty(foundTranslation)
           && service.FallbackBehavior.HasFlag(FallbackBehaviors.FallbackCulture)
           && query.UseFallback
           && (Equals(query.Language, service.FallbackCulture)) || Equals(query.Language.Parent, CultureInfo.InvariantCulture))
        {
            return _originalHandler.Execute(
                new GetTranslation.Query(query.Key,
                                         CultureInfo.InvariantCulture,
                                         false));
        }

        return foundTranslation;
    }
}
```

Here, the logic is quite similar to one found in Episerver's provider fix code fragment, so think maybe it's just worth to reuse it somehow.

But, built-in provider cannot just look for `foundTranslation == null` case. As we know now - Episerver *might* return back `string.Empty` even if fallback failed (in Episerver context). So here query handler unfortunately must check for `string.IsNullOrEmpty` value.

## Translate to InvariantCulture Explicitly

There is a tiny nuance when you call something line this on built-in Episerver service:

```csharp
LocalizationService.GetStringByCulture("...", CultureInfo.InvariantCulture)
```

and the same on DbLocalizationProvider library's one:

```csharp
LocalizationProvider.GetStringByCulture("...", CultureInfo.InvariantCulture)
```

Latter will have correct value as it's explicitly will be able to resolve invariant culture translation when asked. However, it's not entirely true for the Episerver - knowing behavior and logic inside `TryGetStringByCulture` method - invariant culture won't even have it's chance to act like culture to resolve translations for.

So just beware that there is a difference between DbLocalizationProvider localization provider and Episerver's localization service when working directly with invariant cultures.

And don't worry about all this stuff, it's already implemented in the library and everything should fallback as it's designed and configured.

Happy double fallbacking ;)

[*eof*]
