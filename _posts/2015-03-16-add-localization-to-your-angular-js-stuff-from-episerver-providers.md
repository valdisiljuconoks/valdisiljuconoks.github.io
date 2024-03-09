---
title: Add localization to your angular.js stuff from EPiServer providers
author: valdis
date: 2015-03-16 20:05:00 +0200
categories: [Add-On, .NET, C#, Localization Provider, Episerver, Optimizely]
tags: [add-on, .net, c#, open source, localization, episerver, optimizely]
---

Sneaking inside default EPiServer’s sample sites (like AlloyTech) we can notice that Xml based localization provider is being used. It’s nice approach to keep all localization resources in single place, well-structured using XPath query to fetch a value and group into multiple languages.
 We can argue here about various drawbacks for this provider (like, stringly typed access – actually there is a small poly fill for this [OpenWaves.EPiServer.Localization](https://openwaves.codeplex.com/SourceControl/latest#OpenWaves.EPiServer.Localization/trunk/src/), lack of possibility to make new authoring of localization resources by editors – also we can find some attempts to fix this – [CMS75XmlResourcesTool](https://github.com/PNergard/CMS75XmlResourcesTool)) but this is not the point of the blog post.

In this blog post we are going through steps how to move server-side EPiServer Xml based resource provider content over to client-side and localize angular.js controllers, services or bindings – basically how to let angular.js to know that there is localization enabled and it’s located on the server and it’s in Xml format understandable by EPiServer providers.

## Adding Localization Service to Angular.js Application

First of all we would need to add localization support to angular.js side. For this we can pick [angular-localization on github.com package](https://github.com/doshprompt/angular-localization) (execute this in your directory where front-end stuff is located):

```
bower install angular-localization
```

Next we need to add it to our application. Simple:

```javascript
<script src="{front-end}/angular/angular.js"></script>
<script src="{front-end}/angular-localization/angular-localization.js"></script>

angular.module('myApp', ['ngLocalize'])
```

That’s it. Next we need to connect it to the server-side.

## Connect localization service to WebApi

By default angular-localization will try to fetch localization resource file (Json format) based on decision how you request resource keys. If you request `locale.getString('shared.sampleKey')` plugin will assume that there must be file named `shared.lang.json` in your resource directory under culture subdirectory.

For instance you should then have a `.lang.json` file located in `languages/en-US/shared.lang.json` with following content:

```json
{
    sampleKey: "sample value"
}
```

If you request localization resource value by following key (shared.sampleKey):

```javascript
angular.module('myApp')
       .controller('myController', ['$scope', 'locale', function($scope, locale) {

           function init() {
                var message = locale.getString('shared.sampleKey');
           }

        }]);
```

Localization service will try to fetch `shared.lang.json` file (because shared in the name of the bundle for the service – bundle = set of resources located in the file with bundle name) from resource directory based on default localization service settings.

**NB!** Actually you need to use locale.ready() to workaround asynchronous call to fetch the resources:

```javascript
angular.module('myApp')
       .controller('myController', ['$scope', 'locale', function($scope, locale) {

           function init() {
                locale.ready('shared', function() {
                    message = locale.getString('shared.sampleKey');
                });
           }

       }]);
```

Instead what we want to do is to redirect angular-localization service to get Json content directly from our `WebApi` controller.

Configuring localization service

First of all we need to configure localization service to connect to our not yet existing controller to fetch resources.

```javascript
angular.module('myApp', ['ngLocalze', 'ngLocalize.Config'])
       .value('localeConf', {
            basePath: '/api/localization/get-translation',
            fileExtension: ''
       })
```

After these tweaks angular-localization service will try to reach for similar url: `/api/localization/get-translation/{language-code}/{name-of-bundle}` to fetch localization resources.

Next we need to create Api controller itself. Actually not a big deal:

```csharp
[RoutePrefix("api/localization")]
public class LocalizationApiController : BaseApiController
{
    [Route("get-translation/{language}/{bundle}")]
    [HttpGet]
    public Dictionary<string, string> GetTranslation()
    {
        var language = RequestContext.RouteData.Values["language"];
        var bundle = RequestContext.RouteData.Values["bundle"];

        var results = new Dictionary<string, string>();

        return results;
    }
```

This small controller will make sure that if request is coming to the controller to fetch Json data (expected as key/value format) – it’s is returned as this is automatically done by Json serializer if we are returning `Dictionary<k,v>`.

Next thing is completely up to you (as you have both – requested language and bundle name) I’ll describe our approach to fetch resource translations only for that particular bundle.

Use strongly-typed interface to Xml resources

If we install `OpenWaves.EPiServer.Localization` plugin – it makes sure that content from Xml file gets converted into object graph that could be traversed and walked around.

Let’s say we have Xml file with following resource keys:

```xml
<languages>
  <language name="English" id="en">
    <Shared>
      <SampleKey>This is sample value</SampleKey>
    </Shared>
  </language>
</languages>
```

`OpenWaves.EPiServer.Localization` will generate `TranslationKeys.Shared` property that will contain collection of resources keys defined below that element.
 Using this approach we can traverse keys and generate json formatted reply in our controller:

```csharp
[Route("get-translation/{language}/{bundle}")]
[HttpGet]
public Dictionary<string, string> GetTranslation()
{
    var results = new Dictionary<string, string>();
    var language = RequestContext.RouteData.Values["language"];

    TraverseTranslations(TranslationKeys.Shared, results, new CultureInfo(language));
    return results;
}

private static void TraverseTranslations(ITranslationEntry rootEntry,
                                         IDictionary<string, string> results,
                                         CultureInfo culture)
{
    var collection = rootEntry as TranslationKeyCollection;

    if (collection != null)
    {
        collection.Entries.ForEach(x => TraverseTranslations(x, results, culture));
    }

    var key = rootEntry as TranslationKey;

    if (key != null)
    {
        results.Add(key.Path, key.GetStringByCulture(culture));
    }
}
```

**NB!** Recommend not to use `{bundle}` route data to fetch resources – this may open some vulnerability for your site’s “power users”. Instead use hard-coded route path to fetch resources for specific bundle / area:

```csharp
[Route("get-translation/{language}/shared")]
[HttpGet]
public Dictionary<string, string> GetTranslation()
{
    var results = new Dictionary<string, string>();
    var language = RequestContext.RouteData.Values["language"];

    TraverseTranslations(TranslationKeys.Shared, results, new CultureInfo(language));
    return results;
}
```

Fixing resource key names

By default using `OpenWaves.EPiServer.Localization` plugin key.Path will be `XPath` syntax – this will not play nicely with angular-localization service.
Let’s say we have following Xml file:

```xml
<languages>
  <language name="English" id="en">
    <Shared>
      <SampleKey>
        <AnotherSampleKey>
          <ThirdKey>This is the value</ThirdKey>
        </AnotherSampleKey>
      </SampleKey>
    </Shared>
  </language>
</languages>
```

Asking for resources through our newly created controller we will get back flatten Json:

```json
{
    "/Shared/SampleKey/AnotherSampleKey/ThirdKey": "This is the value"
}
```

We need to apply few fixes in order to get service running:

```csharp
[Route("get-translation/{language}/shared")]
[HttpGet]
public Dictionary<string, string> GetTranslation()
{
    var results = new Dictionary<string, string>();
    var language = RequestContext.RouteData.Values["language"];

    TraverseTranslations(TranslationKeys.Shared,
                         results,
                         new CultureInfo(language),
                         TranslationKeys.Shared.Name);

    return results;
}

private static void TraverseTranslations(ITranslationEntry rootEntry,
                                         IDictionary<string, string> results,
                                         CultureInfo culture,
                                         string groupName)
{
    var collection = rootEntry as TranslationKeyCollection;

    if (collection != null)
    {
        collection.Entries.ForEach(x => TraverseTranslations(x, results, culture, rootEntry.Name));
    }

    var key = rootEntry as TranslationKey;

    if (key != null)
    {
        var path = key.Path.Replace("/" + groupName + "/", "").Replace("/", "-");
        results.Add(path, key.GetStringByCulture(culture));
    }
}
```

This small patch will make json look like following:

```
{
    "SampleKey-AnotherSampleKey-ThirdKey": "This is the value"
}
```

Now we can safely use new service using a bit different naming conventions for resource keys:

```javascript
angular.module('myApp')
       .controller('myController', ['$scope', 'locale', function($scope, locale) {

           function init() {
                locale.ready('shared', function() {
                    message = locale.getString('shared.SampleKey-AnotherSampleKey-ThirdKey');
                });
           }

       }]);
```

## Summary

In order to localize angular.js stuff on client-side based on server-side Xml resource providers you need to:

a) Add localization service to angular application;
b) Configure service to fetch resources from WebApi route;
c) Add WebApi controller that will listen on that route;
d) optional: add OpenWaves localization plugin to enable strongly-typed access to resources;
e) Based on requested bundle and culture generate set of translation keys back in reply as Dictionary<k,v> entries get them serialized automatically into json key/value format;

Good luck localizing!

[*eof*]
