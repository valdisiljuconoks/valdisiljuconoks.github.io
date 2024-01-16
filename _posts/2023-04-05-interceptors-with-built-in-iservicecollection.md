---
title: How to Build Interceptor with built-in Microsoft DI - IServiceProvider
author: valdis
date: 2023-04-05 14:00:00 +0200
categories: [C#, .NET, Dependency Injection, IoC, Containers, Interceptor]
tags: [c#, .NET]
---

**Service Interception** is known as an altering process to make it look like service instance is delivered to the calling site by contract, but it could altered and totally another type might be hidden by the interface.

Interception of the services is not a built-in feature of ServiceProvider (aka Microsoft Dependency Injection framework).
However it's quite simple to implement one yourself. This blog post describes how to implement one and use it.

## Registration API Prototype
Registration of the interceptor should be made when service collection is built. At this point service registrations is just a list of the service descriptors specifying implementing type, registered type, lifetime and other characteristics of the service.

Let's start with prototype code how consumers would register interceptors in service collection:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddTransient<IEngine, TheEngine>();
builder.Services.Intercept<IEngine, EngineWithTelemetry>();
```

This extension method should register new service interceptor - `EngineWithTelemetry` implementing `IEngine` interface and "behaving" according to the interface.

Whoever would request instance of `IEngine` would get `EngineWithTelemetry`.

Usually signature of the interceptor service constructor requires an instance of "intercepted" service (you can call it `inner` to make things more clear).

```csharp
public EngineWithTelemetry(IEngine inner)
{
    _inner = inner;
}
```

Now we have interceptor and intercepted services. What happens in interceptor is your logic, but usually following requirements deem to look for interceptors:

* logging
* telemetry
* some data stream correlation requirements
* caching
* auditing
* etc.

One of the easiest ways to implement these requirements is to implement cross-cutting concern and apply implicitly to required services.

This also makes it possible not to change original service's source code and not flood it with different responsibilities to do many things not entirely related to the service itself.

Also regarding constructor of the interceptor - it should of course also support other injections (not only intercepted service). So following code should work as well (assuming that `ITelemetryClient` is registered service in IoC):

```csharp
public EngineWithTelemetry(IEngine inner, ITelemetryClient telemetryClient)
{
    _inner = inner;
}
```

## Implementation
Now let's get back to the interceptor registration and get it implemented. 
This is our prototype method:

```csharp
public static IServiceCollection Intercept<TService, TInterceptor>(this IServiceCollection services)
{
    ...
}
```

Remember that service registration while container is built - is just a list of descriptors that are easily adjustable (or replaced completely).

First of all we need to check if there are any registrations of the service to intercept, if not - we can safely return.

```csharp
var targetServices = services.Where(s => s.ServiceType == typeof(TService)).ToList();

if (!targetServices.Any())
{
    return services;
}
```

Next we need to iterate over list of target service registrations and understand if we have implementation factory for the service or not - having factory it is required to invoke that to get instance of the intercepted service.

```csharp
foreach (var service in targetServices)
{
    var ix = services.IndexOf(service);
    
    if (service.ImplementationFactory == null)
    {
        ...
    }
    else
    {
        ...
    }
}
```

Now when we have access to the intercepted service descriptor - we need to **REPLACE** it.

```csharp
if (service.ImplementationFactory == null)
{
    services[ix] = new ServiceDescriptor(
        typeof(TService),
        provider => ActivatorUtilities.CreateInstance(
            provider,
            typeof(TInterceptor),
            ActivatorUtilities.GetServiceOrCreateInstance(provider, service.ImplementationType)),
        service.Lifetime);
}
```

So what happens here? Short description:

* we replace existing service descriptor with the new one
* new descriptor registers the same type (`typeof(TService)`) in service collection
* new descriptor will use `ActivatorUtilities.CreateInstance` to create an instance of the interceptor (`typeof(TInterceptor)`)
* original service instance is created with help of `ActivatorUtilities.GetServiceOrCreateInstance(provider, service.ImplementationType)`
* and it is passed as parameter to the constructor of the interceptor (thus we are able to receive `inner` instance of the original service)

Very similar code also for the case with implementing factory for the intercepted service:

```csharp
if (service.ImplementationFactory == null)
{
    ...
}
else
{
    services[ix] = new ServiceDescriptor(
        typeof(TService),
        provider => ActivatorUtilities.CreateInstance(
            provider,
            typeof(TInterceptor),
            service.ImplementationFactory.Invoke(provider)),
        service.Lifetime);
}
```

But here we have to invoke `service.ImplementationFactory.Invoke(provider)` to the instance of the original service.

Whole interceptor service registration implementation for completeness:

```csharp
public static IServiceCollection Intercept<TService, TInterceptor>(this IServiceCollection services)
{
    var targetServices = services.Where(s => s.ServiceType == typeof(TService)).ToList();

    if (!targetServices.Any())
    {
        return services;
    }

    foreach (var service in targetServices)
    {
        var ix = services.IndexOf(service);

        if (service.ImplementationFactory == null)
        {
            services[ix] = new ServiceDescriptor(
                typeof(TService),
                provider => ActivatorUtilities.CreateInstance(
                    provider,
                    typeof(TInterceptor),
                    ActivatorUtilities.GetServiceOrCreateInstance(provider, service.ImplementationType)),
                service.Lifetime);
        }
        else
        {
            // register descriptor for the service with factory
            services[ix] = new ServiceDescriptor(
                typeof(TService),
                provider => ActivatorUtilities.CreateInstance(
                    provider,
                    typeof(TInterceptor),
                    service.ImplementationFactory.Invoke(provider)),
                service.Lifetime);
        }
    }

    return services;
}
```


Happy intercepting!
[*eof*]
