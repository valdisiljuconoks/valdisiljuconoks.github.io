---
title: Localization Provider - Major 6 Released!
author: valdis
date: 2020-02-22 18:00:00 +0200
categories: [Add-On, .NET, C#, Open Source, Localization Provider]
tags: [add-on, .net, c#, open source, localization provider, localization]
---

## Introduction

I'm pleased to announce that v6 of DbLocalizationProvider is finally out to the wild. It's been a bit bumpy road and longer journey as expected, but here we are..

This post will guide you through some of the most noteworthy changes since last major version.

![2020-02-22_01-07-43](/assets/img/2020/02/2020-02-22_01-07-43.png)

## Major Changes in v6

* Apache 2.0 license
* AdminUI registration changes
* Jumped to `netcore31` (.NET Core 3.1) version
* Jumped to JSON.NET v11.0.2
* MSSQL as separate package (this opens up extensibility to plugin additional providers). No EF / EFCore dependency anymore.
* Some smaller fixes for AdminUI (like localizing error messages :)
* Language fallback configuration
* Added interface `ILocalizationProvider` for easier unit testing
* Logging added to unify functionality across platforms and runtimes

## Getting Started with v6

There are few changes in how to get started with v6 of DbLocalizationProvider.
Let me guide you through that process.

### Install Storage Implementation Package

One of the biggest change in v6 is that DbLocalizationProvider by default does not have dependency on EntityFramework | EFCore and therefore by default if you already have project running on v5.x -> just by upgrading packages to v6 will not solve all your problems.

You will need to install additional package with MSSQL Server storage implementation.

```cmd
> dotnet add package LocalizationProvider.Storage.SqlServer
```

Once this is done, you need to configure connectionString for the SQL Server package (usually in your `Startup.cs` file):

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddDbLocalizationProvider(ctx =>
    {
        ...
        ctx.UseSqlServer(Configuration.GetConnectionString("DefaultConnection"));
    });
}
```

### Configure Fallback Languages

LocalizationProvider gives you option to configure fallback languages for the library.
It means that provider will try to get translation in requested language. And if it does not exist in that language, fallback language list is used to decide which language to try next until either succeeds or fails with no translation found.

To configure fallback languages use code below:

```csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddDbLocalizationProvider(_ =>
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

This means that following logic will be used during translation lookup:

1) Developer requests translation in Swedish culture (`"sv"`) using `ILocalizationProvider.GetString(() => ...)` method.
2) If translation does not exist -> provider is looking for translation in Norwegian language (`"no"` - second language in the fallback list).
3) If translation is found - one is returned; if not - provider continues process and is looking for translation in English (`"en"`).
4) If there is no translation in English -> depending on `ConfigurationContext.EnableInvariantCultureFallback` setting -> translation in InvariantCulture may be returned.

### Mapping AdminUI and Clientside Resource Handler

ASP.NET MVC back in 2.2 days introduced different routing mechanism - called [EndpointRouting](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/routing?view=aspnetcore-3.1).

If you create new web application using templates - application will use endpoint routing by default. However if you are upgrading existing projects - opt-in to use endpoint routing is optional.

So depending on what routing system your application is using, you might need to register AdminUI and Clientside resource handler differently.

For **old MVC Routing**:

```csharp
public void ConfigureServices(IServiceCollection services)
{
   services
       .AddControllersWithViews(opt => opt.EnableEndpointRouting = false)
       .AddMvcLocalization();

   services.AddRouting();
   ...
}

public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    ...
    app.UseMvc(routes =>
    {
        routes.MapDbLocalizationAdminUI();
        routes.MapDbLocalizationClientsideProvider();

        routes.MapRoute(
            name: "default",
            template: "{controller=Home}/{action=Index}/{id?}");
    });
}
```

For **Endpoint routing**:

```csharp
public void ConfigureServices(IServiceCollection services)
{
   services.AddRouting();
   ...
}

public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    app.UseRouting();
    ...
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllerRoute("default", "{controller=Home}/{action=Index}/{id?}");
        ...

        endpoints.MapDbLocalizationAdminUI();
        endpoints.MapDbLocalizationClientsideProvider();
    });
}
```

For more information and sample setup code you can checkout [sample Mvc project](https://github.com/valdisiljuconoks/localization-provider-core/tree/master/tests/DbLocalizationProvider.Core.AspNetSample) on github.

### Securing AdminUI

AdminUI by default is secured via roles which you can configure yourself via `Configure` method on startup:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddDbLocalizationProviderAdminUI(_ =>
    {
        ...
        _.AuthorizedAdminRoles.Add("Admins");
        _.AuthorizedEditorRoles.Add("Translators");
    });
}
```

In order for you to get this working, you need to enable roles based access in your ASP.NET identity setup:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddDbContext<...>(...);

    services
        .AddDefaultIdentity<...>(...)
        .AddRoles<IdentityRole>();
}
```

### Bare Minimum Startup.cs to Start With

This seems to be bare minimum for the localization provider start functioning OK-ish.

```csharp
using System.Collections.Generic;
using System.Globalization;
using DbLocalizationProvider.AdminUI.AspNetCore;
using DbLocalizationProvider.AdminUI.AspNetCore.Routing;
using DbLocalizationProvider.AspNetCore;
using DbLocalizationProvider.AspNetCore.ClientsideProvider.Routing;
using DbLocalizationProvider.Core.AspNetSample.Data;
using DbLocalizationProvider.Core.AspNetSample.Resources;
using DbLocalizationProvider.Storage.SqlServer;
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Identity;
using Microsoft.AspNetCore.Localization;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;

namespace SampleApp
{
    public class Startup
    {
        public Startup(IConfiguration configuration)
        {
            Configuration = configuration;
        }

        public IConfiguration Configuration { get; }

        public void ConfigureServices(IServiceCollection services)
        {
            services.AddDbContext<ApplicationDbContext>(
                options => options.UseSqlServer(Configuration.GetConnectionString("DefaultConnection")));

            services
                .AddDefaultIdentity<IdentityUser>()
                .AddRoles<IdentityRole>()
                .AddEntityFrameworkStores<ApplicationDbContext>();

            services
                .AddControllersWithViews()
                .AddMvcLocalization();

            services.AddAuthorization();
            services.AddRazorPages();
            services.AddRouting();

            var supportedCultures = new List<CultureInfo> { new CultureInfo("sv"), new CultureInfo("no"), new CultureInfo("en") };

            services.Configure<RequestLocalizationOptions>(opts =>
            {
                opts.DefaultRequestCulture = new RequestCulture("en");
                opts.SupportedCultures = supportedCultures;
                opts.SupportedUICultures = supportedCultures;
            });

            services.AddDbLocalizationProvider(_ =>
            {
                _.EnableInvariantCultureFallback = true;
                _.ScanAllAssemblies = true;
                _.FallbackCultures.Try(supportedCultures);
                _.UseSqlServer(Configuration.GetConnectionString("DefaultConnection"));
            });

            services.AddDbLocalizationProviderAdminUI(_ =>
            {
                _.RootUrl = "/localization-admin";
                _.ShowInvariantCulture = true;
                _.ShowHiddenResources = false;
                _.DefaultView = ResourceListView.Tree;
            });
        }

        public void Configure(IApplicationBuilder app, IHostingEnvironment env)
        {
            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
            }
            else
            {
                app.UseExceptionHandler("/Home/Error");
                app.UseHsts();
            }

            var options = app.ApplicationServices.GetService<IOptions<RequestLocalizationOptions>>();
            app.UseRequestLocalization(options.Value);

            app.UseRouting();
            app.UseHttpsRedirection();
            app.UseStaticFiles();
            app.UseAuthentication();
            app.UseAuthorization();

            app.UseDbLocalizationProvider();
            app.UseDbLocalizationProviderAdminUI();
            app.UseDbLocalizationClientsideProvider();

            app.UseEndpoints(endpoints =>
            {
                endpoints.MapControllerRoute("default", "{controller=Home}/{action=Index}/{id?}");
                endpoints.MapRazorPages();

                endpoints.MapDbLocalizationAdminUI();
                endpoints.MapDbLocalizationClientsideProvider();
            });
        }
    }
}
```

## Special Notes

### Overwriting Query/Command Handlers

This applies to use-cases for overwriting command and/or query handlers (by having this code in `Startup.cs` or any other global composition root).

```csharp
app.UseDbLocalizationProvider(ctx =>
    {
        ctx.TypeFactory
            .ForQuery<AvailableLanguages.Query>()
            .SetHandler<SampleAvailableLanguagesHandler>();
    });
```

It's important to note that if you do overwrite query and/or command handlers to customize localization provider to your needs ***and*** at the same time using SQL Server to storage your resources - you have to overwrite handlers **after** you have configured to use SQL Server.

This is due to fact that extension method `UseSqlServer()` itself overwrites and sets some of the handlers required to successfully implement storage.

Instead of this:

```csharp
app.UseDbLocalizationProvider(ctx =>
    {
        ctx.TypeFactory
            .ForQuery<AvailableLanguages.Query>()
            .SetHandler<SampleAvailableLanguagesHandler>();

        ctx.UseSqlServer(...);
    });
```

You have to rewrite to this:

```csharp
app.UseDbLocalizationProvider(ctx =>
    {
        ctx.UseSqlServer(...);

        ctx.TypeFactory
            .ForQuery<AvailableLanguages.Query>()
            .SetHandler<SampleAvailableLanguagesHandler>();
    });
```

## Breaking Changes

Major versions are perfect timing to break something.

### ConfigurationContext.Connection Property Is Gone

With v6 there is no storage management inside core libraries. Currently only MSSQL Server implementation is done. Therefore you have to now explicitly register your storage implementation with LocalizationProvider during startup.

Following code will not work anymore:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddDbLocalizationProvider(_ =>
    {
        _.Connection = "....";
    });
}
```

Instead you need to explicitly register MSSQL storage:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddDbLocalizationProvider(_ =>
    {
        _.UseSqlServer(Configuration.GetConnectionString("..."));
    });
}
```

### AdminUI Registration for ASP.NET Applications

Old:

```csharp
app.Map(
    "/localization-admin",
    b => b.UseDbLocalizationProviderAdminUI(_ =>
    {
        _.ShowInvariantCulture = true;
    }));
```

New: depends on your routing system used. Please refer to section above for more detailed description on how to map AdminUI in your app.

### Reversed ResourceLookupFilter Logic

In some late v5.x version extra parameter for filter resource lookup was added `ConfigurationContext.ResourceLookupFilter`.
First implementation of the filter was created to return `false` if resource should not be searched.
However, after inspecting naming of the property - it didn't match the meaning of the property, therefore in new version v6 logic of this property has been reversed. Now if `true` is returned - then resource lookup happens.

If you have used this property and have implemented your logic - please make sure that you revert your logic.

### LegacyModeEnabled Has Moved

Property to control if legacy mode is enabled or not has been moved from `ConfigurationContext.ModelMetadataProviders.EnableLegacyMode` to `ConfigurationContext.EnableLegacyMode`.

### CreateNewResource Command Changes

Command `CreateNewResource` has been removed and instead now you can create new resources in batch by using `CreateNewResources` command. If you still need to create single resource - execute batch command with single resource in collection.

## Give It a Try!

I'm happy to get to this milestone. v6 is next major version that includes many tiny changes inside. Things might get lost or broken while you are commuting between versions..

Would be awesome if you share your feedback!

Hope this helps!
Happy localization!

[*eof*]
