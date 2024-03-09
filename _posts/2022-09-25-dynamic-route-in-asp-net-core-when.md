---
title: Dynamic Route in ASP.NET Core When MapDynamicControllerRoute Does Not Work
author: valdis
date: 2022-11-01 10:00:00 +0200
categories: [Add-On, Optimizely, Episerver, .NET, C#, Open Source]
tags: [add-on, optimizely, episerver, .net, c#, open source]
---

## Background ##
Creating one of the add-on for Optimizely I had to deal with challenge to register dynamically route for the API controller. Dynamic route here means that user is able to provide own url segment on top of which add-on user interface and supporting API service endpoints will be registered.

There have been also some [bug reports](https://github.com/valdisiljuconoks/localization-provider-epi/issues/150) around this area. So we need to address this somehow.

Why this is needed? Add-on is used in content management systems where you as the author of add-on can't control url and therefore your chosen url for add-on might collide with user content address.

Code sample is as following:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services
        .AddDbLocalizationProviderAdminUI(_ =>
        {
            _.RootUrl = "/localization-admin-ui";
        });
}

public void Configure(IApplicationBuilder app)
{
    app.UseEndpoints(endpoints =>
    {
        // other endpoint registration
        ..

        endpoints.MapDbLocalizationAdminUI();
    });
}
```

This should assure that add-on is available at `/localization-admin-ui`, which also requires supporting API services to be available under `localization-admin-ui/api/*` address.

You might be thinking that route pattern could be moved mapping method - `MapDbLocalizationAdminUI(string pattern)`. However this is not entirely possible due to fact that add-on setup and initialization code needs to know `RootUrl` way before it's mapped at endpoint collection.

Dynamic route requirement means that we can't use `[Route]` attribute on the API service controller as it requires compilation time constant for the route pattern.

So this is not possible:

```csharp
namespace DbLocalizationProvider.AdminUI.AspNetCore;

[Route(RootUrl + "/api/service")]
public class ServiceController : ControllerBase
{
    ...
}
```

## Try with Dependency Injection for Attribute ##
One of the approaches would be to pass in `UiConfigurationContext` where `RootUrl` property is located to the attribute somehow.

```csharp
public class DynamicRouteAttribute : RouteAttribute
{
    public DynamicRouteAttribute(string pattern)
        : base(pattern, ???)
    {
    }

    public DynamicRouteAttribute(string pattern, UiConfigurationContext context)
        : base(context.RootUrl + pattern)
    {
    }
}

[DynamicRoute("/api/service")]
public class ServiceController : ControllerBase
{
    ...
}
```

This would allow us to use ordinary route attribute with route pattern and also respect user's configuration for the root address of the add-on. Dependency Injection for an attribute is against all good practices to split [meta-data and behavior](https://blog.ploeh.dk/2014/06/13/passive-attributes/) and should not be used.

Also as add-on (library) author you have no idea how to access `UiConfigurationContext` from the attribute context. `UiConfigurationContext` is added to service collection (dependency injection container) but we have limited options how to access it here. We could use some static `ServiceLocator`, `IServiceProvider` or any other method - but that would require some actions from hosting project setup code, like - preserving service factory somehow and use it later.

However this is against all best practices and requires some static context. Not very "dependency injection-ish".

Let's try another approach.

## Mapping Route with DynamicControllerRoute ##
There is a way to dynamically change matched route before it's handled. This is done by `MapDynamicControllerRoute()`.

```csharp
public void Configure(IApplicationBuilder app)
{
    app.UseEndpoints(endpoints =>
    {
        // other endpoint registration
        ..

        endpoints.MapDbLocalizationAdminUI();
    });
}

public static IEndpointRouteBuilder MapDbLocalizationAdminUI(
    this IEndpointRouteBuilder builder)
{
    var context = builder.ServiceProvider.GetService<UiConfigurationContext>();

    builder.MapDynamicControllerRoute<AdminUIDynamicRouteValueTransformer>(context.RootUrl + "/api/service/{action}");

    return builder;
}
```

Here we are transforming incoming route on `context.RootUrl + "/api/service/*"` route. Transformer does nothing fancy - just sets correct controller for the request:

```csharp
public  class AdminUIDynamicRouteValueTransformer: DynamicRouteValueTransformer
{
    private readonly UiConfigurationContext _context;

    public AdminUIDynamicRouteValueTransformer(UiConfigurationContext context)
    {
        _context = context;
    }

    public override ValueTask<RouteValueDictionary> TransformAsync(
        HttpContext httpContext, RouteValueDictionary values)
    {
        values["controller"] = "Service";
        return new ValueTask<RouteValueDictionary>(values);
    }
}
```

This looks OK'ish. Transformer is created via dependency injection, so we can get to our services in normal way.

However, problems start when you install other citizens to the project who also does route registration - [Optimizely Service API](https://docs.developers.optimizely.com/commerce/v1.3.0-service-api-developer-guide/docs) in this case.

Exception is somewhat odd, but I coudn't figure out where exactly was the problem.

![](/assets/img/2022/09/route-exception-1.png)


My speculation - `ServiceController` does not have `[Route]` attribute but is still selected via dynamic controller route transformer. In result - interfering with "normal" API service attribute-based registrations. I don't know.

If you have any idea - please comment! Thx

## ApplicationModel to the Rescue ##
There is another approach how to influence application behavior after runtime has finished building behavioral model for the application (scanning and registering different parts of the application).

When application starts up there is a phase in the pipeline - build application model. This is process when runtime decides how application should behave - what controllers we have, actions for each controller, parameters, etc.
To influence this phase we can use `IApplicationModelProvider` interface. First we need to register this in IoC. Best location - when service collection is built:

```csharp
public static IDbLocalizationProviderAdminUIBuilder AddDbLocalizationProviderAdminUI(
    this IServiceCollection services,
    Action<UiConfigurationContext> setup = null)
{
    ...
    services.TryAddEnumerable(ServiceDescriptor.Transient
                              <IApplicationModelProvider, ServiceControllerDynamicRouteProvider>());
}
```

And dynamic route model provider:

```csharp
public class ServiceControllerDynamicRouteProvider : IApplicationModelProvider
{
    private readonly UiConfigurationContext _context;

    public ServiceControllerDynamicRouteProvider(UiConfigurationContext context)
    {
        _context = context;
    }

    public void OnProvidersExecuting(ApplicationModelProviderContext context) { }

    public void OnProvidersExecuted(ApplicationModelProviderContext context)
    {
        var serviceControllerModel =
            context.Result.Controllers.FirstOrDefault(c => c.ControllerType.IsAssignableFrom(typeof(ServiceController)));

        if (serviceControllerModel == null)
        {
            return;
        }

        var selectorModel = serviceControllerModel.Selectors.FirstOrDefault();
        if (selectorModel is { AttributeRouteModel: { } })
        {
            selectorModel.AttributeRouteModel.Template = _context.RootUrl + "/api/service";
        }
    }

    public int Order => -1000 + 10;
}
```

Here are few important things to mention:

* it does not really matter whether you do configuration on `OnProvidersExecuted` or `OnProvidersExecuting`
* however it's important to keep correct `Order` of the part model provider. Default application model provider execution order is set to `-1000`. If you set anything beyond this - it will be executed first. So increasing by `10` - we can be sure that our model will be executed after the default one. This reminds me of delicate poking of middle-wares to get the correct order.

So when the default application model provider has been executed - all controllers have been scanned and registered. Each controller has also meta-data about how to reach out this controller - thus controller's `Selectors`. We have an opportunity to mutate this selector data to change routing for our `ServiceController`.

However this will require `ServiceController` to have `[Route]` attribute set - so the controller is registered in the application model. But we can set any pattern in the attribute - as it will be rewritten anyways.

```csahrp
[Route("/localization-admin/api/service", Name = "LocAdminUIRoute")]
public class ServiceController : ControllerBase
{
    ...
}
```

<br/>

Happy routing!
[*eof*]
