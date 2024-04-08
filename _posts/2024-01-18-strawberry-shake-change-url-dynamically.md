---
title: Change Url Dynamically for GraphQL Client (Strawberry Shake)
author: valdis
date: 2024-04-08 14:00:00 +0200
categories: [.NET, C#, Open Source]
tags: [.net, c#, open source]
---

[Strawberry client](https://strawberry.rocks/) package is a great library if you need to talk to some GraphQL endpoints.

Sometimes you need to decide of different endpoint address depending on some condition or even do it fully dynamically (during runtime per call).

So let's assume that we have two different endpoint addresses that we would need to switch between depending on some condition.

```json
{
    "AppConfig": {
        "NormalEndpointBaseAddress": "https://normal-server.com",
        "DebugEndpointBaseAddress": "https://debug-server.com"
    }
}
```

In our case we had to switch between endpoints depending on incoming request parameters - one endpoint was used in "normal" and other in "debug" mode.
Mode switch is just a simple payload parameter in the request:

```json
{
    ....
    "isDebugMode": true
}
```

When you add Strawberry client to your service collection, it returns `IClientBuilder<T>` type which in turn has `ConfigureHttpClient()` extension:

```csharp
services
    .AddSomeGraphQLServiceClient()        // name of the method depends on the Strawberry config for your service
    .ConfigureHttpClient((provider, client) =>
    {
    });
```

Inside `ConfigureHttpClient()` we can make a decision what endpoint base address will be.

```csharp
services
    .AddSomeGraphQLServiceClient()
    .ConfigureHttpClient((provider, client) =>
    {
        // if no one set the mode -> running in normal mode
        var isDebug = ContextProvider.GetState()?.IsDebugMode ?? false;
        var appConfig = provider.GetRequiredService<IOptions<AppConfig>>().Value;

        client.BaseAddress = new Uri(!isDebug ? appConfig.NormalEndpointBaseAddress : appConfig.DebugEndpointBaseAddress);
    });
```

Now we need to look inside `ContextProvider` and understand what is that.

```csharp
public static class ContextProvider
{
    private static readonly AsyncLocal<CustomExecutionContext> State = new();

    public static void SetDebugMode(bool state)
    {
        State.Value = new CustomExecutionContext(state);
    }

    public static CustomExecutionContext? GetState() => State.Value;
}

public class CustomExecutionContext(bool state)
{
    public bool IsDebugMode { get; } = state;
}
```

Key takeaway here is `AsyncLocal<T>` - it is a structure will survive async operations and will always point you to the correct instance of the custom execution context.

Next thing we can implement is action filter and detect is incoming request contains debug mode flag and set it accordingly.

```csharp
builder.Services
    .AddControllersWithViews(options =>
    {
        options.Filters.Add<DebugModeDetectorFilter>();
    });
```

And filter itself:

```csharp
public class DebugModeDetectorFilter : IActionFilter
{
    public void OnActionExecuting(ActionExecutingContext context)
    {
        foreach (var argument in context.ActionArguments.Select(a => a.Value).Where(v => v is BaseServiceRequestDto))
        {
            if (argument != null)
            {
                // set debug mode in state - this will be used later in Strawberry HttpClient configuration to set correct service base address
                ContextProvider.SetDebugMode(((BaseServiceRequestDto)argument).IsDebugMode ?? false);
            }
        }
    }

    public void OnActionExecuted(ActionExecutedContext context)
    {
    }
}
```

In our solution every single request DTO object has base class `BaseServiceRequestDto` - then we can reuse the same common payload parameters for every single request if we need to.

```csharp
using TypeGen.Core.TypeAnnotations;

[ExportTsInterface]
public abstract class BaseServiceRequestDto
{
    [TsOptional]
    public bool? IsDebugMode { get; set; }
}
```

With this in place - now every time front-end will send in `isDebugMode: true` field - back-end will use `https://debug-server.com` as base address for GraphQL services.

Happy coding!

[*eof*]
