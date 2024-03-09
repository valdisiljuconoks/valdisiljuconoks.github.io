---
title: Episerver Localization Provider - Major v6 Released!
author: valdis
date: 2020-03-17 18:00:00 +0200
categories: [Add-On, Optimizely, Episerver, .NET, C#, Open Source, Localization Provider]
tags: [add-on, optimizely, episerver, .net, c#, open source, localization provider, localization]
---

## Introduction

I'm pleased to announce that v6 of DbLocalizationProvider is finally out to the wild. This stressful and lots of unknowns period was great timing for me to sit down and finish started journey. It's been a bit bumpy road and longer trip than expected, but here we are..

This post will guide you through some of the most noteworthy changes since last major version.

![v6-2](/assets/img/2020/03/v6-2.png)

## Major Changes in v6

* Library is now licenses under Apache 2.0 license
* Increased lower runtime version up to `net472`
* Jumped to [JSON.NET v11.0.2](https://www.nuget.org/packages/Newtonsoft.Json/12.0.1)
* Jumped to [Episerver CMS 11.13.1](https://nuget.episerver.com/package/?id=EPiServer.CMS&v=11.13.1) as lower version
* MSSQL as separate package (this opens up extensibility to plugin additional providers). No EF / EFCore dependency anymore.
* Language fallback configuration
* Added interface `ILocalizationProvider` for easier unit testing
* Read-only apps
* Logging added to unify functionality across platforms and runtimes
* Smaller bug fixes here and there, as result some of the obsolete query and command handlers were deleted in favor of unified logic across all runtimes (Episerver, ASP.NET & ASP.NET Core)

Let's cover each of the features in more details.

## New Features

### MS SQL Storage as Separate Package

One of the biggest change in v6 is that DbLocalizationProvider by default does not have any dependency on EntityFramework | EFCore anymore and therefore by default if you already have project running on v5.x -> just by upgrading packages to v6 will not solve all your problems.

You will need to install additional package with MSSQL Server storage implementation.

```cmd
PM> Install-Package LocalizationProvider.Storage.SqlServer
```

Once this is done, you need to configure connectionString for the SQL Server package (usually in your `Startup.cs` file):

```csharp
[InitializableModule]
[ModuleDependency(typeof(ServiceContainerInitialization))]
public class DbLocalizationProviderConnectionSetupModule : IConfigurableModule
{
    public void Initialize(InitializationEngine context) { }

    public void Uninitialize(InitializationEngine context) { }

    public void ConfigureContainer(ServiceConfigurationContext context)
    {
        var connection = ConfigurationManager.ConnectionStrings["EPiServerDB"].ConnectionString;
        ConfigurationContext.Setup(_ =>
        {
            ...
            _.UseSqlServer(conection);
        });
    }
}
```

**NB!** In this example you can see that connection name (`EPiServerDB`) has been hard-coded. It's common name of the connection strings in Episerver. But if you happen (for any reason) to have different connection name of would not like to hard-code it here, then you have retrieve name of the connection from Episerver configuration:

```csharp
var name = EPiServerDataStoreSection.Instance.DataSettings.ConnectionStringName;
var connection = ConfigurationManager.ConnectionStrings[name].ConnectionString;

_.UseSqlServer(connection);
```

By executing this `UseSqlServer` method library will make sure that all necessary query and command handlers are registered to talk to specified database.

At this moment there is no option to customize schema used there (like mapping from your own custom schema to provider's one). But that might come as I see feature requests on [GitHub](https://github.com/valdisiljuconoks/localization-provider-core/issues/32) for this. Vote if you see necessity for your project as well.

### Fallback Languages

Until v6 fallback languages where somewhat broken and not fully (read "properly") implemented. I tried to rely on Episerver's `LocalizationService` behavior and rely on fallback mechanism there. But it turned out to be more tricker that initially might sound. Therefore fallback is implemented straight into the library. Which also allows me to provide this feature for other runtimes as well.

By default Episerver initialization module will read fallaback culture value from `web.config` file (or which ever other file containing `<episerver.framework>` element).

For example (if we take this fragment from config file):

```xml
<episerver.framework>
  ..
  <localization fallbackBehavior="FallbackCulture, Echo" fallbackCulture="en">
    <providers>
      <add name="db" type="DbLocalizationProvider.EPiServer.DatabaseLocalizationProvider, DbLocalizationProvider.EPiServer" />
    </providers>
  </localization>
</episerver.framework>
```

Provider initialization module will read this section and make appropriate settings in configuration context:

* Fallback behavior is checked. If it does not contain `FallbackCulture` no further configuration is done. If behavior contains fallback, then initialization module continues configuring provider.
* `English` (`"en"`) will be used as fallback language.

Also you are able to set invariant culture fallback (this means that `CultureInfo.InvariantCulture` will be used as last chance for the retrieval of the translation). Invariant culture fallback can be set by:

```csharp
[InitializableModule]
[ModuleDependency(typeof(ServiceContainerInitialization))]
public class DbLocalizationProviderConnectionSetupModule : IConfigurableModule
{
    public void Initialize(InitializationEngine context) { }

    public void Uninitialize(InitializationEngine context) { }

    public void ConfigureContainer(ServiceConfigurationContext context)
    {
        ConfigurationContext.Setup(_ =>
        {
            ...
            _.EnableInvariantCultureFallback = true;
        });
    }
}
```

LocalizationProvider gives you option to configure fallback languages for the library from the code also. If translation in requested language does not exist, list of fallback languages is used to decide which language to try next until either succeeds or fails with no translation found.

To configure fallback languages from code use snippet below:

```csharp
[InitializableModule]
[ModuleDependency(typeof(ServiceContainerInitialization))]
public class DbLocalizationProviderConnectionSetupModule : IConfigurableModule
{
    public void Initialize(InitializationEngine context) { }

    public void Uninitialize(InitializationEngine context) { }

    public void ConfigureContainer(ServiceConfigurationContext context)
    {
        ConfigurationContext.Setup(_ =>
        {
            ...
            _.FallbackCultures
                .Try(new CultureInfo("sv"))
                .Then(new CultureInfo("no"))
                .Then(new CultureInfo("en"));

        });
    }
}
```

Following logic will be used during translation lookup:

1) Developer requests translation in French culture (`"fr"`) using `ILocalizationProvider.GetString(() => ...)` method.
2) If translation does not exist -> provider is looking for translation in Swedish  language (`"sv"` - first language in the fallback list).
3) If translation does not exist -> provider is looking for translation in Norwegian language (`"no"` - second language in the fallback list).
4) If translation is found - one is returned; if not - provider continues process and is looking for translation in English (`"en"`).
5) If there is no translation in English -> depending on `ConfigurationContext.EnableInvariantCultureFallback` setting -> translation in InvariantCulture may be returned.

**NB!** Also worth mentioning that if you would be requesting translation in Norwegian (`"no"`) then localization provider is smart enough to understand that requested language is one of the fallback languages and will proceed only with the "rest" of the fallback languages - it this case only `"en"` language would be searched and invariant language (if configured so).

### Localization Provider Hides Behind Interface

With request from the community - you can now access provider also via interface `ILocalizationProvider`.

```csharp
public class SamplePageController : PageController<SamplePage>
{
    private ILocalizationProvider _provider;

    public SamplePageController(ILocalizationProvider provider)
    {
        _provider = provider;
    }

    public ActionResult Index(StartPage currentPage)
    {
        var someTranslation = _provider.GetString(() => SomeResource.SomeProperty);
    }
}
```

Interface mapping and instance is automatically added to Episerver container. Extra registration is not required.

Testing your components should be now much easier.

### Logging Added to All Runtimes

I made small effort to unify all runtimes and provider universal API to my own code to do the logging across multiple runtimes. Image SQL Server storage library. That one could be used in Episerver context, pure AS.NET or ASP.NET Core applications. Methods to do the logging should be the same (at least from the library perspective itself). Like:

```csharp
public class SomeLogicInSqlLibrary
{
    private ILogger _logger;

    public SomeLogicInSqlLibrary(ILogger logger)
    {
        _logger = logger;
    }

    public void SomeMethod()
    {
        // perform some action which requires logging
        ...

        _logger.Info("This is DONE now!");
    }
}
```

To make this happen adapters for various runtime logging infrastructures are required (to forward logged messages from  the library to underlying logging infrastructure).

This is done for Episerver if you are using official `EPiServer.Logging` APIs.

### Read-Only Apps

Next major version of localization provider also allows interesting scenario that was not possible (or hard to achieve) - reader/writer and read-only apps.

![dbloc-readonly-app](/assets/img/2020/03/dbloc-readonly-app.png)

I've seen common case when one of the applications only had to use localization provider without any other functionality - for example [Azure Function](https://azure.microsoft.com/en-us/services/functions/)  which required to send out email to customers and use localized resources to compose email body. Function itself has no resources not wants to scan and register anything in underlying database - it requires only to read up content of the database and use resources.

For this reason you can just set configuration setting `a` to `false`:

For example in Azure Function:

```csharp
public class Startup : FunctionsStartup
{
    public override void Configure(IFunctionsHostBuilder builder)
    {
        builder.Services.AddDbLocalizationProvider(_ =>
        {
            _.DiscoverAndRegisterResources = false;
            ...
        });

        InitializationExtensions.UseDbLocalizationProvider();
    }
}
```

Here we have to call directly `UseDbLocalizationProvider()` method on `InitializationExtensions` as there is no extension method specifically for Azure Functions runtime.

## Changes from v5.x

### Episerver IDisplayResolution Names

If you have `IDisplayResolution` implementations which are using `LocalizationService` to retrieve display names (usually found in AlloyTech sample sites) special treatment is required in order to get text from `DbLocalizationProvider`.

Assume that you have following display resolution class:

```csharp
public class AndroidVerticalResolution : DisplayResolutionBase
{
    public AndroidVerticalResolution() : base("/resolutions/androidvertical", 480, 800) { }
}
```

As you can see we are inheriting from `DisplayResolutionBase` which in turn tries to find display name for child class during constructor (default implementation in AlloyTech provided sample project).

```csharp
public abstract class DisplayResolutionBase : IDisplayResolution
{
    private Injected<LocalizationService> LocalizationService { get; set; }

    protected DisplayResolutionBase(string name, int width, int height)
    {
        Id = GetType().FullName;
        Name = Translate(name);
        Width = width;
        Height = height;
    }

    ...

    private string Translate(string resurceKey)
    {
        string value;

        if(!LocalizationService.Service.TryGetString(resurceKey, out value))
        {
            value = resurceKey;
        }

        return value;
    }
}
```

Episerver initializes display resolutions during `CmsRuntimeInitialization` module.
As DbLocalizationProvider allows setup code to be located in "ordinary" initialization module, there is a timing issue between `CmsRuntimeInitialization` module and your own custom module invoke moments. There *could* be situations (and usually are) when custom module that does setup of DbLocalizationProvider library is executed **after** `CmsRuntimeInitialization` module.
This results into missing display names for your implementations of `IDisplayResolution` interfaces (because when display resolutions are initialized, display name is loaded - DbLocalizationProvider library is *not yet* setup - hence translations are missing).

Workaround for this case is to do DbLocalizationProvider storage initialization a bit earlier (earlier than `CmsRuntimeInitialization` is invoked). The easiest way I found is to implement `IConfigurableModule` than has dependency on `ServiceContainerInitialization`. This more or less guarantees that your setup module is invoked one of the first in the list immediately after IoC setup:

```csharp
[InitializableModule]
[ModuleDependency(typeof(ServiceContainerInitialization))]
public class DbLocalizationProviderConnectionSetupModule : IConfigurableModule
{
    public void Initialize(InitializationEngine context) { }

    public void Uninitialize(InitializationEngine context) { }

    public void ConfigureContainer(ServiceConfigurationContext context)
    {
        var connection = ConfigurationManager.ConnectionStrings["EPiServerDB"].ConnectionString;
        ConfigurationContext.Setup(_ =>
        {
            _.ModelMetadataProviders.EnableLegacyMode = () => true;
            _.EnableInvariantCultureFallback = true;
            _.UseSqlServer(conection);
        });
    }
}
```

**NB!** Don't forget to enable `LegacyMode` if your display name resource starts with `/` - for example `/resolutions/androidvertical` (sample from Alloy).

### Manual Storage Schema Update

Sometimes it's hard to control timing of initialization chain. Are you sure where and when your startup code will be called?
If you need to control timing and call initialization code yourself - you can use `Synchronizer` class to trigger at least underlying schema updates (if needed) and perform resource synchronization.

```csharp
using System.Configuration;
using DbLocalizationProvider.Storage.SqlServer;
using DbLocalizationProvider.Sync;

[InitializableModule]
[ModuleDependency(typeof(ServiceContainerInitialization))]
public class DbLocalizationProviderConnectionSetupModule : IConfigurableModule
{
    public void Initialize(InitializationEngine context) { }

    public void Uninitialize(InitializationEngine context) { }

    public void ConfigureContainer(ServiceConfigurationContext context)
    {
        ConfigurationContext.Setup(_ =>
        {
            var connectionName = EPiServerDataStoreSection.Instance.DataSettings.ConnectionStringName;
            var connectionString = ConfigurationManager
                                   .ConnectionStrings[connectionName]
                                   .ConnectionString;

            _.UseSqlServer(connectionString);
        });

        // manually sync storage schema - this is required as schema update happens later in startup pipeline
        // here - as localization provider is called way TOOOOOO early - we make sure that schema is OK and queries do not fail
        var sync = new Synchronizer();
        sync.UpdateStorageSchema();
    }
}
```

I hope that this will not be necessary for you! But just in case.. :)

Happy localizing!
Stay safe!

[*eof*]
