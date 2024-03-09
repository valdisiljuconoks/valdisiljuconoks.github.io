---
title: Baking Round Shaped Apps with MediatR
author: valdis
date: 2018-03-24 14:30:00 +0200
categories: [.NET, C#, ASP.NET, Architecture, Pizza Architecture, Software Design]
tags: [.net, c#, open source, asp.net, architecture, pizza architecture, software design]
---

Back in days I spent time to describe something that I called [pizza architecture](https://medium.com/@benorama/the-evolution-of-software-architecture-bd6ea674c477) - [showing off](https://tech-fellow.ghost.io/2016/10/31/baking-round-shaped-software-mapping-to-the-code/) how to organize and design applications in round shape with domain in the middle and less important transport specific channels (like web or batch environment) closer to the border of the pizza.
Originally I spent time to developer self-brewed messaging pipeline inside the application to show how internals are working in this type of architecture. And promised to convert application to use some out-of-the box solution - like [MediatR](https://github.com/jbogard/MediatR/).

![](/assets/img/2018/03/FreshPaint-21-2016.09.25-10.45.57.png)

[This PR](https://github.com/valdisiljuconoks/PizzaArchitecture/commit/e7256dd55fb9594037de0c5e36f19214049e1327) shows all the necessary changes I had to make to convert original solution to use MediatR library (+ also converted to NetCoreApp 2.0).

## Naming is Hard

Naming is our industry is always hard part and we have to stick to basic English range. I personally like idea that my types express intent and are named accordingly - commands and queries. However, when we are going to switch to MediatR library, it regulates other naming schema - `Request` (aka query or command) and `Notification`. It's easy to remember which is what by understanding that request has always result (the same way as queries and/or commands) and notification is more like fire & forget. So, my definition:

* `Request<T>` - under this type you might hide commands and queries. Most of the time you don't need to distinguish between the two. You send the request to the mediator and expect results. When you don't have any results - just use `Unit` as `T`;
* `Notification` - under this type you might implement something similar to domain events. Using notifications you are expected to notify other parties that are interested in your business about stuff that happens inside your domain.

## Register Phase

Look for registries:

```csharp
container.Configure(_ => { _.Scan(s => { s.LookForRegistries(); }));
```

This for some reason is not really working in .Net Core 2.0 app. Registries were not found and added to the StructureMap. Instead, I had to manually add those to the list:

```csharp
container.Configure(_ => { _.AddRegistry<MediatorRegistrations>(); }));
```

## DomainEvent -> INotification

When switching to MediatR library -> there is no explicit different between commands, queries, domain events and other DDD terms. Instead setup is quite simple: you do have requests with potential result and you do have notifications with no result.

In order to migrate to MediatR library I had to rename all occurrences from `IDomainEvent` (and domain event handler) to just `INotification` (and appropriate handlers). Think this simplification is much better and you don't need to think about whether you are dispatching event, query or command.

## Messaging Pipeline -> Mediator Behaviors

In previous post we extended mediator implementation itself and decorated it with another instance with running validation routines against passed in commands or queries. This makes sense in self-made mediator implementation. But here, when we are switching to MediatR library - it provides [another way](https://github.com/jbogard/MediatR/wiki/Behaviors) to extend messaging pipelines and add some extra logic while handling commands or queries (requests).

### Adding Behaviors to Pipelines

Adding various behaviors to the messaging pipelines is quite simple and requires just a couple registrations in container.

```csharp
public class MediatorRegistrations : Registry
{
    public MediatorRegistrations()
    {
        For(typeof(IPipelineBehavior<,>)).Add(typeof(...));
        ...
    }
}
```

Behaviors are very similar to the Asp.Net middlewares sitting in the middle of the request processing pipeline and waiting for a chance to kick in.

**NB!** Order of the registrations matter here as mediator will run through behaviors in order they are registered in IoC container (actually it's how those registrations are returned from IoC container implementation). So it *might* depend from particular IoC implementation.

### Input Validation with FluentValidation

For instance, we can take a look at how validation is added to the overall messaging pipeline.

First, we need to add fluent validation registration if not done already (`Startup.cs`):

```csharp
public IServiceProvider ConfigureServices(IServiceCollection services)
{
    services.AddMvc()
            .AddFluentValidation(_ =>
                {
                    _.RegisterValidatorsFromAssemblyContaining<Startup>());
                });

    return UseStructureMapContainer(services);
}
```

Then we need to add validation behavior to the pipeline (IoC registry):

```csharp
For(typeof(IPipelineBehavior<,>)).Add(typeof(ValidationBehavior<,>));
```

And actual implementation:

```csharp
public class ValidationBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
{
    private readonly IValidatorFactory _validationFactory;
    private readonly ILogger _logger;

    public ValidationBehavior(IValidatorFactory validationFactory,
                              ILoggerFactory loggingFactory)
    {
        _validationFactory = validationFactory;
        _logger = loggingFactory.CreateLogger("ValidationBehavior");
    }

    public async Task<TResponse> Handle(TRequest request,
                                        CancellationToken cancellationToken,
                                        RequestHandlerDelegate<TResponse> next)
    {
        var validator = _validationFactory.GetValidator(request.GetType());
        var result = validator?.Validate(request);

        if(result != null && !result.IsValid)
            throw new ValidationException(result.Errors);

        var response = await next();
        return response;
    }
}
```

The same way as middle-wares behavior implementation receives `next` behavior and can decide whether to call it or break the circuit.

### Command Pre-Execute/Post-Execute Handlers

In our custom implementation we had special command pipeline that was responsible for finding and running pre-command handlers (anyone that would like to have control before the command is carried out).

```csharp
public class CommandPipeline<TCommand>
             : ICommandHandler<TCommand> where TCommand : ICommand
{
    private readonly ICommandHandler<TCommand> _inner;
    private readonly IPreExecuteCommandHandler<TCommand>[] _preHandlers;

    public CommandPipeline(ICommandHandler<TCommand> inner,
                           IPreExecuteCommandHandler<TCommand>[] preHandlers)
    {
        _inner = inner;
        _preHandlers = preHandlers;
    }

    public void Handle(TCommand command)
    {
        if(_preHandlers != null && _preHandlers.Any())
        {
            foreach (var handler in _preHandlers)
            {
                handler.Handle(command);
            }
        }

        _inner.Handle(command);
    }
}
```

Now using MediatR library it's actually pretty easy to get the same functionality (and also post processors) out of the box.

What we need to do is to register special built-in behavior in message pipeline via IoC registrations (in this case StructureMap registry):

```csharp
public class MediatorRegistrations : Registry
{
    public MediatorRegistrations()
    {
        // order of the registration matters here
        For(typeof(IPipelineBehavior<,>))
           .Add(typeof(RequestPreProcessorBehavior<,>));
        ...
        For(typeof(IPipelineBehavior<,>))
           .Add(typeof(RequestPostProcessorBehavior<,>));
    }
}
```

Both types `RequestPreProcessorBehavior` and `RequestPostProcessorBehavior` come from mediator library and is very easy to use.
The only thing left here is pre/post-handler implementation:

```csharp
public class NeedToExecuteBeforeCommand : IRequestPreProcessor<Command>
{
    public Task Process(Command request,
                        CancellationToken cancellationToken)
    {
        // very complex logic goes here...

        return Task.CompletedTask;
    }
}
```

Library is taking care of all the heavy generic type lifting in order to find matching handlers for given request.

### Only RequestHandlers Supported in Processing Pipeline

From MediatR docs:

> The pipeline behaviors are only compatible with IRequestHandler<TRequest,TResponse> and can't be used with INotificationHandler<TRequest>.

Just to be aware that you are able to adjust behavior of the message processing pipeline if you are sending `Request<T>` type of the message. Notifications are not adjustable and they are just dispatched with no custom behavior attached.

## Summary

Implementing your own mediator and understanding all internals how it's working was definitely fun and educational. While converting now to MediatR library - I understood that it was mistake to decorate mediator itself in original post to get custom behavior. Instead - it's much more elegant solution to add behaviors to the processing pipeline and drip custom logic there compared to decorations around core dispatcher object. Using behaviors it's easier to add/remove custom logic, see all custom processing behaviors in one place, easy change order and unit test those.

Also understood that it does not matter to be DDD compliant 100% and talk about command and queries, domain events and other terms explicitly in code. It's easier to think about architecture in simple terms - requests and notifications. Keeping architecture simple.

Even if I would be using home-brewed mediator implementation in production, upon next project - it most probably would scream to be extracted and packed and reusable library. At that moment consideration about using something already ready made would come at ease.

If you are interested more to see the code it's available [in GitHub](https://github.com/valdisiljuconoks/PizzaArchitecture/tree/mediatr).

Happy mediatoring! :)

[*eof*]
