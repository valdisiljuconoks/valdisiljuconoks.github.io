---
title: Switch Content Language Everywhere in EPiServer
author: valdis
date: 2015-04-09 12:30:00 +0200
categories: [.NET, C#, Episerver, Optimizely, Commerce]
tags: [.net, c#, episerver, optimizely, commerce]
---

Nowadays lot of sites are localized or globalized (depending from which point of view you are looking at). Usually it means that developer has to provide some sort of language switcher for the end-user to pick another content language. This blog post is about small trick that I’m using when need to provide switcher on web sites driven by EPiServer.

I’ll not dig into details on how to get started with globalization and how localization engine and services are working (you can find lot of information in [SDK](http://world.episerver.com/documentation/Items/Developers-Guide/EPiServer-Framework/7/Localization/)).

## Common Features

It really depends on each site’s requirements, but here are some common cases that regular site may have:

* provide content in different language;
* give possibility for end-user to switch language;
* enlist all available languages defined in the site;
* if current page does not exist in target language (one that end-user is going to switch to) – redirect user to start page;
* if current page exists in target language – redirect user to new page version in requested content language;
* preserve all query parameters from the current page (not to loose contextual information while switching);
* optional: persist selected language somewhere (sometimes auto-select content language is needed when user accessing site next time);


## Enlisting Available Languages

This is really easy. Type that will give you everything is `LanguageBranchRepository`:

```csharp
IList<LanguageBranch> languages =
          ServiceLocator.Current
                .GetInstance<LanguageBranchRepository>()
                .ListEnabled();
```

If you need to check whether current language while looping is the same as current page, this is also easy:

```csharp
@foreach (var lang in languages)
{
    if (string.Equals(Model.CurrentPage.Language.Name, lang.LanguageID,
                      StringComparison.CurrentCultureIgnoreCase))
    { }
```

## Generating Target Page Address

Next thing we need is to generate target page address in particular content language that will be used in language switcher. There could be cases when end-user is on page that does not exist in other languages, regardless of that site most probably needs to provider language switcher anyway. Target address in this case may be start page. It’s simple condition in code (assuming that Model in the view is view-model with access to current page from EPiServer).

```csharp
var contentLoader = ServiceLocator.Current.GetInstance<IContentLoader>();

@foreach (var lang in languages)
{
    var languageSelector = new LanguageSelector(lang.LanguageID);
    var alternatePage =
             contentLoader.Get<PageData>(Model.CurrentPage.ContentLink, languageSelector)
             ?? contentLoader.Get<PageData>(ContentReference.StartPage, languageSelector);
}
```

Once we got target page content reference, we can generate url for that page:

```csharp
...
var resolver = ServiceLocator.Current.GetInstance<UrlResolver>();

@foreach (var lang in languages)
{
    ...
    var alternatePageAddress = resolver.GetUrl(alternatePage.ContentLink,
                                               languageSelector.LanguageBranch);
}
```

### Preserving Query Parameters

Very simple solution to preserve any query parameter if applicable.

```csharp
@foreach (var lang in languages)
{
    ...
    var parameters = Request.Url != null ? Request.Url.Query : string.Empty;
```

Now we can generate target page address:

```csharp
<a href="@(alternatePageAddress + "switchlanguage" + parameters)">@lang.Name</a>

// this will give us link for 'href':
//  "/{language-code}/target-page/switchlanguage?parameter1=value1&.."
```

## `SwitchLanguage' Action Handler

Using generated link we should switch language wherever end-user is in the site, on any page with any query parameters already added to the url. It means that we can safely provider language switcher link everywhere on the site.
Action `SwitchLanguage` is special action that will be invoked by Asp.Net Mvc together with EPiServer when action look-up will fail.

You may ask why da heck do I need to create new action for handling content language and how this differs from ordinary target link to target language? One of the reason why I like special handler is that I do have freedom of what exactly happens when user explicitly chooses to change content language and not just visiting page in particular language.

So who is handling this `SwitchLanguage` action?

I wasn’t aware that there is a small extensibility point in Asp.Net Mvc – called `Handle Unknown Action'. Unknown action handler is implemented in base Mvc controller and overwritten in EPiServer base action controller.
You will need to register this unknown action handler and tell EPiServer to register this in it’s internal list of handlers:

```csharp
[ServiceConfiguration(typeof(IUnknownActionHandler))]
public class LanguageSwitcherHandler : IUnknownActionHandler
{
    public string ActionName
    {
        get
        {
            return "changelanguage";
        }
    }

    public ActionResult HandleAction(Controller controller)
    {
    }
}
```

This will tell EPiServer: whenever you will receive a message from Asp.Net Mvc that action was not dispatched (because there is no method implemented that may handle requested action from request) please invoke this action handler if action name matches `ActionName` property value.

As request has been already made to target page in target content language you can easily extract language out of `RouteData`:

```csharp
public class LanguageSwitcherHandler : IUnknownActionHandler
{
    ...

    public ActionResult HandleAction(Controller controller)
    {
        var language = controller.RouteData.Values["language"].ToString();
    }
```

What you do with this language is really up to you and site’s requirements. Usually it may be preserved in cookie, session, cloud storage or whatever other persistent media for later usage.

## What to do next when language is switched?

Next what you may need to do is to actually process originally requested page or action.
I know that this is not ideal solution, but one of the easiest way was to generate new url without `“/switchlanguage”` action and send back redirect action result to client, to force browser to request new page once again within new browser state (assuming that maybe you may persist new selected language in browser’s cookies).

**NB!** Sounds like not an ideal solution for production site?! :) Most probably. The proper way you may need to do is actually something like Server.Execute() did in old good days.

## Summary

So using unknown action handlers it’s possible:

* give possibility for end-user to switch language wherever he or she is in the site;
* developer precise moment and handler that will be invoked when end-user will switch content language, and not just requesting page in particular content language;
* query parameters are preserved if any;

Happy coding!

[*eof*]
