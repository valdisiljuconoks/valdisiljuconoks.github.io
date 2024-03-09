---
title: Baking Round Shaped Software - Mapping to the Code
author: valdis
date: 2016-10-31 14:30:00 +0200
categories: [.NET, C#, Architecture, Pizza Architecture, Software Design]
tags: [.net, c#, architecture, pizza architecture, software design]
---

This part of the round shaped software blog posts as promised will be about mapping the theoretical terms and ideas mentioned in [previous post](https://tech-fellow.eu/2016/10/17/baking-round-shaped-software/) to the actual code fragments and components.

![](/assets/img/2016/10/code.png)

## Application for the Post

To move further and map theory to the code, I've created really simple sample application for managing the users. To demonstrate mediator internals and round shaped architecture - we will stick with hand crafted mediator. Mediator will be able to handle commands, queries and events.

![](/assets/img/2016/10/2016-10-29_23-29-02-1.png)

In the application you can:

* enlist existing users (for the sake of simplicity - users will be stored in memory)
* create new user
* handle event for the newly created user (this might be handy if you need to notify somebody else about the fact)

Sample application is based on [Asp.Net Core Framework](https://www.asp.net/core).

## Notes About DDD
Some notes regarding Domain Driven Design (DDD).

### Handlers as Application Services
We had discussions about where handlers belong. It looks like handlers should be located in the outermost circle - somewhere in UI - it's not part of the domain. It's partially true. Handlers are not part of the domain. But neither they are part of the user interface. User interface is just a delivery mechanism.

For me - handlers look alike process orchestrators, like application services, like [Interactors](http://ebi.readthedocs.io/en/latest/index.html). They provide fuel for the use cases - only handlers know which use case they are handling, which domain services and models they need to operate with or upon to carry out use case. Handlers are implemented **medium agnostic**. They should not care from which delivery mechanism they were accessed.

### Feature Slicing in DDD Context

Slicing by features is one of the concept how to organize larger code bases, even in round shaped softwares. For the sake of simplicity - I left all the source code in single assembly, but sliced it by feature folders. In real application, code organization should be split at least into 2 group of projects:

* *user interface project* - this project would contain all delivery mechanism related stuff - like forms, views, controllers, all that stuff. Organization by feature folders here would make it more organized.
* *core domain project (each for every feature)* - these projects would contain core domain logic for each feature, including use cases, handlers, services, events, domain models, etc.

References (dependencies) should point from delivery mechanism projects inwards to domain projects (center of the architecture).

## Web is Delivery Mechanism

From the architecture perspective - I still think that web is just a delivery mechanism, ceremony layer around business core area. But don't get me wrong - I love web, I love JavaScript (sometimes) and love Ajax and Json, and the rest of the [zoo inhabitants](https://hackernoon.com/how-it-feels-to-learn-javascript-in-2016-d3a717dd577f#.9futv0n7m). But still - looking from another side of the fence - web is just yet another consumer of the application, just another incoming channel - like unit tests, or integration tests (if you have one), batch processes, scheduled jobs or somebody else.

![](/assets/img/2016/10/FreshPaint-22-2016.10.09-08.49.12-1.png)

Delivery mechanism might be changed from version X to Y, it might be refactored from WebForms to Asp.Net Mvc. It even might be supplemented by another user interface (like mobile or watch, or anything else). Despite all the changes around, the core is main reason why application exists. It's your main focus as developer/architect.

## Code Definitions

As we covered in [previous post](https://tech-fellow.ghost.io/2016/10/17/baking-round-shaped-software/) there are 3 types of messages that could be passed through mediator:

* *commands* - something that mutes state of the application;
* *queries* - something that does not mutate state of the application, but returns value instead;
* *events* - something that notifies other parties of the fact inside the domain;

### Commands, Queries and Events

Code definition for each of them is pretty straight forward.

For the commands, queries and events in sample application here - it's more like marker interfaces.

```csharp
public interface ICommand { }

public interface IQuery<out TResult> { }

public interface IDomainEvent { }
```

Regarding of commonly shared attributes for all message types - is really up to you and project, how constraint you want to be regarding messages going through the system and what kind of common characteristics are shared across all message types. Currently interfaces for message types are more like marker interfaces, but they may contain various properties that all messages of particular type must implement. For instance all events should have occourance date and time:

```csharp
public interface IDomainEvent
{
    DateTime OccouredAt { get; }
}
```

### Message Handlers

On the other end - there are handlers for each type of the message.

For commands:

```csharp
public interface ICommandHandler<in TCommand>
                                where TCommand : ICommand
{
    void Handle(TCommand command);
}
```

One for queries:
```csharp
public interface IQueryHandler<in TQuery, out TResult>
                              where TQuery : IQuery<TResult>
{
    TResult Handle(TQuery query);
}
```

And for events:
```csharp
public interface IDomainEventHandler<in TEvent>
                                    where TEvent : IDomainEvent
{
    void Handle(TEvent @event);
}
```

Most of them are all the same: incoming message type passed to `Handle()` method.
Note that these are just abstractions still. No concrete command or handler is shown yet.

### The Mediator

Last piece in the puzzle is component that will "glue" together message types with corresponding handlers.
From source code perspective, mediator is an interface exposed to consumers, so they can use mediator to handle specific messags. Execution of the message means finding specific handler (or maybe more handler**s**) for this specific message and asking handler to handle the message.

Mediator interface is defined as following:

```csharp
public interface IMediator
{
    void Execute<T>(T command) where T : ICommand;

    void Publish<T>(T @event) where T : IDomainEvent;

    TResult Query<TResult>(IQuery<TResult> query);
}
```

Note that there is separate method to execute each type of the message - one for commands, one for queries and last for events. We will return back to this later, when concrete messages will be passed around the system.

## DI Container Setup aka "Register Phase"

"Registration Phase" In [RRR pattern](http://blog.ploeh.dk/2010/09/29/TheRegisterResolveReleasepattern/) is quite important just because that's the place where composition root will look for mappings from the requested abstrations to the concrete implementations. Maybe in sample applications it's better to use [Pure DI](http://blog.ploeh.dk/2014/06/10/pure-di/), but I'll stick with my favorite container - [StructureMap (SM)](http://structuremap.github.io/) - just for convenience and also to demonstrate some of the cool features modern containers could offer.

### Setup IServiceProvider

[StructureMap](http://structuremap.github.io/) is powerful enough to find types my any mean (I'm pretty sure that other containers can do that as well - I'm just not so deep into other containers to blog about that in more details).

So let's start with swapping out default `IServiceProvider` (we need to replace Asp.Net Core default one to enable more black magic in type scanning and object wiring process).

We need to add following code in `ConfigureServices()` method in `Startup.cs`:

```csharp
public IServiceProvider ConfigureServices(IServiceCollection services)
{
    return UseStructureMapContainer(services);
}

private IServiceProvider UseStructureMapContainer(IServiceCollection services)
{
    ....

    var container = new StructureMap.Container();

    container.Populate(services);
    container.Configure(config =>
    {
        config.Scan(scanner =>
        {
            scanner.TheCallingAssembly();
            scanner.WithDefaultConventions();
            scanner.LookForRegistries();
        });
    });

    return container.GetInstance<IServiceProvider>();
}
```

### Scanning for Handlers

When we have StructureMap registries in place, we can add SM registry to register mediator types in container.

```csharp
public class MediatorRegistrations : Registry
{
    public MediatorRegistrations()
    {
        Scan(scanner =>
        {
            scanner.TheCallingAssembly();
            scanner.WithDefaultConventions();

            scanner.AddAllTypesOf(typeof(ICommandHandler<>));
            scanner.AddAllTypesOf(typeof(IQueryHandler<,>));
            scanner.AddAllTypesOf(typeof(IDomainEventHandler<>));
        });
    }
}
```

This SM feature (`AddAllTypesOf()`) will make sure that **all** types will be registered in the DI container.

### Register Handler Factories

DI container will have information about list of handlers for various message types. However - during runtime it will be required to create instance of those handlers. For this reason - we need to register additional types - abstract factory and concrete message type handler factories. Under the hood handler factories will use abstract type factory. Let's just see the code.

```csharp
public class MediatorRegistrations : Registry
{
    public MediatorRegistrations()
    {
        Scan(...);

        For<IAbstractFactory>().Use<AbstractFactory>()
                               .ContainerScoped();
        For<IQueryHandlerFactory>().Use<QueryHandlerFactory>()
                                   .ContainerScoped();
        For<ICommandHandlerFactory>().Use<CommandHandlerFactory>()
                                     .ContainerScoped();
        For<IDomainEventHandlerFactory>().Use<DomainEventHandlerFactory>()
                                         .ContainerScoped();

        ...
    }
}
```

[Container scoped](http://structuremap.github.io/the-container/nested-containers/) instances (`ContainerScoped()`) is needed for disposable dependencies. Every time for new web request nested container is created. Which will ensure that types created via `AbstractTypeFactory` are disposed at the end of the request. I don't need to do that manually - as SM manages nested containers automatically.

### Abstract Type Factory

Abstract type factory actually is just a wrapper around `HttpContext.RequestServices`.

```csharp
public interface IAbstractFactory
{
    object GetService(Type serviceType);
    T GetService<T>();
    IEnumerable<object> GetServices(Type serviceType);
    IEnumerable<T> GetServices<T>();
}

internal class AbstractFactory : IAbstractFactory
{
    private readonly IHttpContextAccessor _contextAccessor;

    public AbstractFactory(IHttpContextAccessor contextAccessor)
    {
        if(contextAccessor == null)
            throw new ArgumentNullException(nameof(contextAccessor));

        _contextAccessor = contextAccessor;
    }

    public object GetService(Type serviceType)
    {
        return _contextAccessor.HttpContext.RequestServices.GetService(serviceType);
    }

    public T GetService<T>()
    {
        return _contextAccessor.HttpContext.RequestServices.GetService<T>();
    }

    public IEnumerable<object> GetServices(Type serviceType)
    {
        return _contextAccessor.HttpContext.RequestServices.GetServices(serviceType);
    }

    public IEnumerable<T> GetServices<T>()
    {
        return _contextAccessor.HttpContext.RequestServices.GetServices<T>();
    }
}
```

### Command Handler Factory

Specific message type handler factories are pretty simple (they mostly rely on the given `AbstractTypeFactory`).

```csharp
internal interface ICommandHandlerFactory
{
    ICommandHandler<T> GetHandler<T>(T command) where T : ICommand;
}

internal class CommandHandlerFactory : ICommandHandlerFactory
{
    private readonly IAbstractFactory _abstractFactory;

    public CommandHandlerFactory(IAbstractFactory abstractFactory)
    {
        if(abstractFactory == null)
            throw new ArgumentNullException(nameof(abstractFactory));

        _abstractFactory = abstractFactory;
    }

    public ICommandHandler<T> GetHandler<T>(T command) where T : ICommand
    {
        return _abstractFactory.GetService<ICommandHandler<T>>();
    }
}
```

It's a lot of ceremony to just create types at the runtime, but that's the life :)

While it's quite easy to create `ICommandHandler` handler factories, dealing with `IQueryHandler` and `IDomainEventHandler` requires more special treatment.

### Note About Covariance for QueryHandlers

Challenge here with `IQueryHandler<IQuery<TResult>>` handlers is related with generics.

Let's see what happens when we need to create handler for `Commands`.

```csharp
public ICommandHandler<T> GetHandler<T>(T command) where T : ICommand
{
    return _abstractFactory.GetService<ICommandHandler<T>>();
}
```

Concrete type of the command is passed as generic type parameter. Meaning that specific handler will be requested from the container, e.g., `ICommandHandler<CreateUser>`. This type will be present - due to `AddAllTypesOf()` - so called [closing open generics](http://structuremap.github.io/generics/).

The same principle for closing open generics applies to query handlers. Now look at potential factory method to create the instance of query handler:

```csharp
public IQueryHandler<TQuery, TResult> GetHandler<TQuery, TResult>(IQuery<TResult> query)
                                                 where TQuery : IQuery<TResult>
{
    ....
}
```

First of, there is no type parameter to identify for which query the handler is required (e.g., concrete type of the query - `GetAllUsers.Query`). We do have only `IQuery<TResult>` type information. It means that we need to create target handler type "dynamically" and pass it to container.

Even if we could use this approach to create type of the target handler and ask underlying abstract type factory (using *untyped* method overload) to give us back handler instance:

```csharp
var handlerType = typeof(IQueryHandler<,>).MakeGenericType(query.GetType(), typeof(TResult));
var handler = _abstractFactory.GetService(handlerType);
```

There is no way we would be able to cast received `System.Object` back to `IQueryHandler<,>`. It's just not the way C# works for covariant interfaces.

Therefore, I'm just relying on [smart guy](https://twitter.com/jbogard) here.
Method for create `IQueryHandler<,>` instances and further work with that handler, there is [pretty neat](https://github.com/jbogard/MediatR/blob/master/src/MediatR/Internal/RequestHandlerWrapper.cs) way to encapsulate that and return intermediate type:

```csharp
public QueryHandlerWrapper<TResult> GetHandler<TResult>(IQuery<TResult> query)
{
    var handlerType = typeof(IQueryHandler<,>)
                          .MakeGenericType(query.GetType(), typeof(TResult));
    var handlerWrapperType = typeof(QueryHandlerWrapper<,>)
                                 .MakeGenericType(query.GetType(), typeof(TResult));

    var handler = _abstractFactory.GetService(handlerType);

    var result = (QueryHandlerWrapper<TResult>) Activator.CreateInstance(handlerWrapperType, handler);

    return result;
}
```

And the wrapper class:

```csharp
internal abstract class QueryHandlerWrapper<TResult>
{
    public abstract TResult Handle(IQuery<TResult> query);
}

internal class QueryHandlerWrapper<TQuery, TResult>
               : QueryHandlerWrapper<TResult> where TQuery : IQuery<TResult>
{
    private readonly IQueryHandler<TQuery, TResult> _inner;

    public QueryHandlerWrapper(IQueryHandler<TQuery, TResult> inner)
    {
        _inner = inner;
    }

    public override TResult Handle(IQuery<TResult> query)
    {
        return _inner.Handle((TQuery) query);
    }
}
```

I could go into more details here on why this is needed, but I guess that might span across multiple blog posts.

### Generic Event Dispatch

Similar situation is for domain event dispatch. Imagine when you are manipulating aggregate root, calling one service, then another one, then call some other stuff. As the result of all the manipulations there will be a collected of domain events. Events should be dispatched at some point.

From the code perspective it means that mediator will receive collection of generic `IDomainEvent` and somehow will need to find **concrete** handler for underlying domain event. Do you think approach like this will work here?

```csharp
// similar method as for commands
public IEnumerable<IDomainEventHandler<T>> GetHandlers<T>(T @event) where T : IDomainEvent
{
    return _abstractFactory.GetServices<IDomainEventHandler<T>>();
}
```

Generic type parameter `T` will be `IDomainEvent`. What would you expect this method to return?

```csharp
factory.GetServices<IDomainEventHandler<T>>();
```

This is ambiguous. Which handlers exactly you need? Mediator has information only about `IDomainEvent`. Do you want me to return all handlers for all events?

Again, I can just rely on [smart people](https://github.com/jbogard/MediatR/blob/master/src/MediatR/Internal/RequestHandlerWrapper.cs) here and follow their solutions.

```csharp
public IEnumerable<DomainEventHandlerWrapper> GetHandlers<TEvent>(TEvent @event)
                                                         where TEvent : IDomainEvent
{
    var handlerType = typeof(IDomainEventHandler<>)
                          .MakeGenericType(@event.GetType());
    var wrapperType = typeof(DomainEventHandlerWrapper<>)
                          .MakeGenericType(@event.GetType());

    var handlerWrappers = _abstractFactory
                              .GetServices(handlerType)
                              .Select(handler => Activator.CreateInstance(wrapperType, handler))
                              .Cast<DomainEventHandlerWrapper>().ToList();

    return handlerWrappers;
}
```

Now we have all the required factories in place and we can even hide them with `internal`. I recognize this whole code as just a ceremony required to properly create handlers based on information that you have at that given moment.

## Mediator Internals

Now as we know how parts are assembled together and our mediator is built, we can advance further and see how mediator works internally.

![](/assets/img/2016/10/FreshPaint-24-2016.10.05-11.55.33-1.png)

We are particularly interested in how messages will be handled.

### IMediator Implementation

Internal working of the mediator is also straight forward - all it has to do is understand incoming message type, find the handler for this message type and pass control over to `Handle()` method. Single responsibility. Other interesting responsibilities we will add later in the blog post.

```csharp
internal class DefaultMediator : IMediator
{
    ...

    public DefaultMediator(IQueryHandlerFactory queryHandlerFactory,
                           ICommandHandlerFactory commandHandlerFactory,
                           IDomainEventHandlerFactory eventHandlerFactory)
    {
        ...
    }

    public void Execute<T>(T command) where T : ICommand
    {
        var handler = _commandHandlerFactory.GetHandler(command);
        if(handler == null)
            throw new InvalidOperationException(...);

        handler.Handle(command);
    }

    public void Publish<T>(T @event) where T : IDomainEvent
    {
        var handlers = _eventHandlerFactory.GetHandlers(@event);
        if(handlers == null)
            return;

        foreach (var handler in handlers)
            handler.Handle(@event);
    }

    public TResult Query<TResult>(IQuery<TResult> query)
    {
        var handler = _queryHandlerFactory.GetHandler(query);
        if(handler == null)
            throw new InvalidOperationException(...);

        return handler.Handle(query);
    }
}
```

Following best DI practices - we require that somebody passes in `IMediator` interface in our Asp.Net controllers:

```csharp
private readonly IMediator _mediator;

public UserManagementController(IMediator mediator)
{
    _mediator = mediator;
}
```

It's composition root (in Asp.Net Mvc case it's controller factory) who is responsible for building object graph and finding all required dependencies.

### Query - List Users

Let's start with the simplest one - we need to enlist all the users from the database and show on the screen?

First of all, we need to define use case class - something that will represent the query itself:

```csharp
public class GetAllUsersList
{
    public class Query : IQuery<ICollection<User>> { }
}
```

Class `GetAllUsersList` represents use case as such and `Query` class represents query message type.

From the very outer layer of the application - from the Asp.Net controller - code to get user list is nothing else as calling mediator and then mapping results back to the view model.

```csharp
public ActionResult List(GetAllUsersList.Query query)
{
    var result = _mediator.Query(query);
    var model = new GetAllUsersList.ViewModel(result);

    return View(model);
}
```

View model is also part of the use case in this sample application:

```csharp
public class GetAllUsersList
{
    public class Query : IQuery<ICollection<User>> { }

    public class ViewModel
    {
        public ViewModel(ICollection<User> users)
        {
            if(users == null)
                throw new ArgumentNullException(nameof(users));

            Users = users;
        }

        public ICollection<User> Users { get; private set; }
    }
}
```

For the sake of simplicity viewmodel contains the same collection of users received from the query handler. In real life application most probably here could be mapping from domain model to viewmodel. From my experience usually viewmodels are superset of domain model - they contain more information, they are enriched and crafted for particular view needs.

If we just execute this code above - we will get an exception from the mediator complaining that there is no handler defined for `GetAllUsersList.Query` query. It's time to add handler (also as member of `GetAllUsersList` use case class):

```csharp
public class GetAllUsersList
{
    ...

    public class Handler : IQueryHandler<Query, ICollection<User>>
    {
        private readonly IUserStorage _storage;

        public Handler(IUserStorage storage)
            {
                _storage = storage;
            }

        public ICollection<User> Handle(Query command)
        {
            return _storage.Users;
        }
    }

    ...
}
```

Note that handlers are just another piece in the system that can require dependencies. Previously described `AbstractTypeFactory` together with `QueryHandlerFactory` are able to create handlers via `IServiceProvider` giving all required dependencies if needed.

Let's see how we can add new user.

### Command - Create User

Start with outer most layer of the application - Mvc controller. Method that draws form on the screen is really boring:

```csharp
public ActionResult Create()
{
    var model = new CreateUser.Command();
    return View(model);
}
```

I'm not a UI designer :)

![](/assets/img/2016/10/2016-10-29_07-00-51.png)

Note about viewmodel in create operation. If you look at the Mvc action - the model of the view is used the same `CreateUser.Command` class. Viewmodel and command classes are exactly the same - there is no reason to split apart. But in real life - most probably viewmodel will be used to draw rich UI and command object will be used to carry only data required to execute this command.

Now let's look at postback handler - Mvc action with `[HttpPost]` attribute:

```csharp
[HttpPost]
public ActionResult Create(CreateUser.Command command)
{
    _mediator.Execute(command);

    return this.RedirectToActionJson("List");
}
```

Note that I'm already expecting `CreateUser.Command` as parameter for the action. Even if initially form would be drawn using viewmodel class as model, I would still expect `CreateUser.Command` as parameter. Why would I receive viewmodel from which I would need to create command and then execute the command. Does not make sense. I'm expecting command as parameter.

Why there is `RedirectToActionJson()` return statement?! We will get to that in "*Client-Side Communications via Ajax*" chapter.

Now let's look at use case class.

```csharp
public class CreateUser
{
    public class Command : ICommand
    {
        public string Username { get; set; }
        public string Email { get; set; }
        public string FirstName { get; set; }
        public string LastName { get; set; }
    }
}
```

And this is the event handler:

```csharp
public class CreateUser
{
    ...

    public class Handler : ICommandHandler<Command>
    {
        public Handler(IUserStorage storage)
        {
            ...
        }

        public void Handle(Command command)
        {
            // perform some business logic
            var newUser = User.Create(command.Username,
                                    command.Email,
                                    command.FirstName,
                                    command.LastName);

            _storage.Users.Add(newUser);
            _storage.Save(newUser);
        }
    }
}
```

Here `IUserStorage` is simulation of the repository that will be able to store and retrieve users. In this sample composition root injects `InMemoryStorage` class - that contains user list in memory. We can pretend that underlying storage has collection of `User` where you can add new ones. Small downside is that list resets everytime we restart the app :)

We haven't seen `User` domain object model so far. Let's take a look.

```csharp
public class User
{
    protected User() { }
    public Guid Id { get; private set; }
    public string Username { get; private set; }
    public string Email { get; private set; }
    public string FirstName { get; private set; }
    public string LastName { get; private set; }

    public static User Create(string username, string email,
                              string firstName, string lastName)
    {
        ...
        var result = new User
                     {
                         Id = Guid.NewGuid(),
                         Username = username,
                         Email = email,
                         FirstName = firstName,
                         LastName = lastName
                     };
        return result;
    }
}
```

Nothing fancy - ordinary C# class with some functionality.

As described earlier - we need to send email notification to newly created users. Creation of the user and sending email notification looks like two different things that would be nice to split and isolate one from another. For this reason we can use domain events to notify other use cases or other features about what just happened. As you can see here - there is no event dispatch in command handler.

### Domain Event - User Created

There are two type of messages - one of them are just executed and that's it. Another part needs to communicate with other components, notify other use cases in other features, etc.
For this reason mediator provides so called event broadcasting (or dispatch).

My understanding regarding domain events is: aggregate root is responsible for "**raising**" events and somebody is responsible for "**dispatching**" them. *Raising* here actually means that the aggregate root is responsible for registering events in some event collection. They will not be immediately dispatched. While performing operations on aggregate root - events are just collected. Once operation is performed and aggregate root needs to be persisted to the underlying store - so to say, operation will be finalized, there has to be somebody who is responsible to dispatch these collected events, find appropriate handlers and give them control to handle the event. So pattern is called - [**double dispatch**](https://lostechies.com/jimmybogard/2010/03/30/strengthening-your-domain-the-double-dispatch-pattern/) pattern.

Let's see how this will look in the code.

First of all, we need to define where events will be stored. For this I chose aggregate root.

```csharp
public class AggregateRoot
{
    public ICollection<IDomainEvent> Events { get; } = new List<IDomainEvent>();
}
```

As we now have a place where to "*raise*" events - we need to update our `User` class.

```csharp
public class User : AggregateRoot
{
    public static User Create(string username, string email,
                              string firstName, string lastName)
    {
        var result = new User
                     {
                         Id = Guid.NewGuid(),
                         ....
                     };

        result.Events.Add(new UserCreatedEvent(username,  email));
        return result;
    }
}
```

Why we need double dispatch of events? One of the reasons is that directly dispatching events from the domain model to handlers makes them to execute immediately. Events are usually with side effects. So it makes testing quite a bit simpler. We can call operations on domain model and then inspect `Events` collection afterwards.

It really depends on the project and particular application architecture, but I tend to think that this "somebody" most of the time is component that pushes new mutated state of the domain model back to the underlying storage. Either you follow `IRepository` pattern or you do manual database access (it actually does not matter) - you have to choose a place where events will be  dispatched.

In our sample application event dispatcher will be done by mediator (with the help from `IDomainEventHandlerFactory`).
Let's say that there is `Save()` method that needs to be called to push data back to the underlying storage. So this is perfect place to do dispatch.


```csharp
internal class InMemoryUserStorage : IUserStorage
{
    ...

    public InMemoryUserStorage(IMediator mediator)
    {
        ...
    }

    public void Save(AggregateRoot root)
    {
        // simulation of the actual database call
        // ...

        // dispatch of the events
        if(!root.Events.Any())
            return;

        while (root.Events.Any())
        {
            var @event = root.Events.FirstOrDefault();
            if(@event == null)
                continue;

            _mediator.Publish(@event);
            root.Events.Remove(@event);
        }
    }
}
```

Depending on your implementation of data access - this might be some hook in [Entity Framework based repository](https://www.pluralsight.com/courses/efarchitecture). If it's your case then implementation of the dispatching hook could look like this:

```csharp
public override int SaveChanges()
{
    var events = ChangeTracker.Entries<AggregateRoot>()
                              .Where(po => po.Events.Any())
                              .ToList();
    ...
}
```

Or maybe you might go EF interceptor way. Depends on your preferences and practices.

### Domain Events vs Application Events

From the DDD theory there should be two type of events - domain events and application events. Latter is optional. From my experience and using mediator as pattern - I rarely need to implement application events, mostly domain events. However - in this architecture I don't see any objection why domain and application events couldn't co-exist. Domain is responsible for "raising" domain events. Somebody is responsible for dispatching those domain events. If you need to implement application wide events - I would go with approach where corresponding use case handler is responsible for dispatching application events. Being a process orchestrator, only handler knows when process has started, when ended or when process transits into different state.
If needed, handler can demand `IMediator` instance via constructor injection and use that to dispatch application events. I think it's perfectly fine if handler has it's own dependencies.

**Update**: as one attentive reader pointed out - to request the same `IMediator` interface to dispatch either domain or application events might violate [ISP](https://en.wikipedia.org/wiki/Interface_segregation_principle) principle. Better would be to require something like `IEventDispatcher<T>` or something like that.

## Client-Server Communications via Ajax

While we were reviewing Mvc action body there was note about *-Json return types from the actions.

Btw, why there is `return this.RedirectToActionJson("List")` and not something like this `return this.RedirectToActionJson(() => List(new GetAllUsersList.Query()));`? Because I'm too lazy now to implement proper `LambdaEpxression` walker and generate Url with action name and all passed arguments. Let smarter guys do it.

Main reason for this is that all forms are sent back to the server via Ajax pipeline. There is no full page postback. Why? Here are some reasons:

* You don't need to deal with `ModelState` object in your Mvc actions anymore. No more ridiculous checks for `ModelState.IsValid` all over the place. Invalid model state is infrastructure thingy, I don't need to think about it in my Mvc action. Does it makes sense to continue to execute Mvc action if `ModelState` is not valid? From the experience I almost always see code that returns the same `View(model)` is state is invalid, otherwise continues with actual business process. For me - it's unnecessary noise.
* If it happens so that client made a request with `ModelState.IsValid = false`, I can return this model state with all the found errors back to the client as `JSON` object. You might think why that's needed? Image if I do have all invalid posted data back as traversable data tree - I can do some black magic and provide nice and rich user interface to go through all the invalid input fields with `<<` and `>>` links (this was actually real requirement in one of the projects).
* Most of the time viewmodel that was used to render the page is "more rich" compared to one that is posted back (I'm referring to the filled in classifiers dropdowns and other enrichments). Using classical postback handling - I will need to "enrich" again received view model in order to return it back to the client (in both error and success cases) within the view. Using Ajax - I don't even need to think about it. Page is never actually posted back to the server, meaning that I don't need to redraw page once again.

Client side code is not complex. Here is snapshot:

```javascript
$(function () {
    var redirect = function (data) {
        data = JSON.parse(data);
        if (data.redirect) {
            window.location = data.redirect;
        } else {
            window.scrollTo(0, 0);
            window.location.reload();
        }
    };
    var showException = function (data) {
        data = JSON.parse(data.responseJSON);
        alert(data.Error);
    };
    var highlightErrors = function (response) {
        var data = response.responseJSON;
        $.each(Object.keys(data),
            function (ix, el) {
                var errors = data[el].Errors,
                    fieldId = el.replace('.', '_');
                if (errors.length != 0) {
                    var $field = $('#' + fieldId);
                    $field.closest('.form-group').addClass('has-error');
                    $field.next('.field-validation-valid').text(errors[0].ErrorMessage);
                }
            });
    };
    var $form = $('form.ajax-form[method=post]');
    $form.on('submit',
        function () {
            var $submitButton = $(this).find('input[type=submit]');
            $submitButton.attr('disabled', true);
            $(window).unbind();
            $.ajax({
                url: $form.attr('action'),
                type: 'post',
                headers: {
                    'X-Ajax-Form': true
                },
                data: new FormData(this),
                cache: false,
                processData: false,
                contentType: false,
                dataType: 'json',
                statusCode: {
                    200: redirect,
                    400: highlightErrors,
                    500: showException
                },
                complete: function () {
                    $submitButton.attr('disabled', false);
                }
            });
            return false;
        });
});
```

Basically what it does is:

* if forms has class `ajax-form` it will be submitted via ajax;
* all form data is being serialized and posted to the server via ajax (no actual postback happens);
* then depending on result of the Http request Javascript does something
* on *success* (`200`) - redirect to some target page
* *bad request* (`400` - usually validation errors) - highlight errors on the form
* in case or *server error* (`500`) - show the error.

To generate proper responses back from the server, we will need to add input validation. This we will do in "*Extending Message Pipeline*" chapter.

## Extending Messaging Pipeline

Remember that mediator in our pizza architecture is central gatekeeper - nobody can pass this guy to reach inner circle - core of the application.

![](/assets/img/2016/10/FreshPaint-5-2016.10.30-11.30.04.png)

We can use this in our advantage and add some interesting mixins there.

### Adding Input Validation

Let's say that it would be nice to have some sort of input value validation before passing it further to domain model. One of the way how to achieve this would be to decorate all viewmodels/command/whatever with `DataAnnotation` attributes and let the Asp.Net Mvc model binder to do the job and then inpsect `ModelState`. I probably go with this solution if the only consumer of the domain logic would the Asp.Net Mvc application. That would mean that I can push all validation stuff to Asp.Net level and not think about it in inner circles. But it's rarely the case in bigger enterprise applications - where consumers of the the same domain logic might be coming from different places (scheduled batch processes, unit tests, maybe integration tests, etc).

Another way how to implement validation is - by extending mediator messaging pipeline. As all messages go through this channel - we can add some inspectors to do the validation.

Let's start with message type validators. I choose here [FluentValidation](https://www.nuget.org/packages/FluentValidation/6.2.0), but it's really up to your preferences.

Let's add `CreateUser.Command` validator:


```csharp
public class Validator : AbstractValidator<Command>
{
    public Validator()
    {
        RuleFor(t => t.Username).NotEmpty();
        RuleFor(t => t.Email).EmailAddress();
    }
}
```

Then we will need to extend mediator pipeline to call this validator if somebody tries to execute `CreateUser` command.

With the power of polymorphism, DI and IoC containers, it's quite easy. We decorate default mediator and wrap it with mediator that understands how to execute validations (`IValidatorFactory` is required by `FluentValidation` library):


```csharp
public class MediatorRegistrations : Registry
{
    public MediatorRegistrations()
    {
        Scan(scanner =>
             {
                 ...
             });

        For<IValidatorFactory>().Use<ServiceProviderValidatorFactory>().Singleton();

        For<IMediator>().DecorateAllWith<MediatorWithValidation>();
        For<IMediator>().Use<DefaultMediator>().ContainerScoped();
    }
}
```

And mediator with validations:


```csharp
internal class MediatorWithValidation : IMediator
{
    public MediatorWithValidation(IMediator inner, IValidatorFactory factory)
    {
        _inner = inner;
        _factory = factory;
    }

    public TResult Query<TResult>(IQuery<TResult> query)
    {
        var validator = _factory.GetValidator(query.GetType());
        var result = validator?.Validate(query);

        if((result != null) && !result.IsValid)
            throw new ValidationException(result.Errors);

        return _inner.Query(query);
    }

    public void Execute<T>(T command) where T : ICommand
    {
        var validator = _factory.GetValidator(command.GetType());
        var result = validator?.Validate(command);

        if((result != null) && !result.IsValid)
            throw new ValidationException(result.Errors);

        _inner.Execute(command);
    }

    public void Publish<T>(T @event) where T : IDomainEvent
    {
        _inner.Publish(@event);
    }
}
```

When validation will fail in the new extended pipeline, `ValidationException` exception will be thrown. To handle this correctly and set Http response code to `400` we will need custom filter added to the Mvc filters collection.


```csharp
private IServiceProvider UseStructureMapContainer(IServiceCollection services)
{
    ...
    services.AddMvc()
            .AddMvcOptions(config => { config.Filters.Add(typeof(ExceptionToJsonFilter));
    ...
})
```

```csharp
public class ExceptionToJsonFilter : IExceptionFilter
{
    public void OnException(ExceptionContext context)
    {
        var validationException = context.Exception as ValidationException;

        if(validationException != null)
            FormatValidationResponse(context, validationException);
        else
            FormatErrorResponse(context);
    }

    ....
    private static void FormatValidationResponse(ExceptionContext context,
                                                 ValidationException validationException)
    {
        validationException.Errors.AddToModelState(context.ModelState);

        var result = new ContentResult
                     {
                         Content =
                             JsonConvert.SerializeObject(context.ModelState),
                         ContentType = "application/json"
                     };

        context.Result = result;
        context.HttpContext.Response.StatusCode = (int) HttpStatusCode.BadRequest;
    }
```

**Why** decorate? Well - it makes clear, that we need to split concerns. Actual (inner) mediator should not be aware of any validation as such - it might or might not be enabled in the application. The validator mediator is just another plugin in overall messaging pipeline. Using this approach - you can add different cross-cutting concerns as well, like logging, performance counters, unit of work.

### Command Pre-Execute Handlers

You can decorate whole `IMediator` interface - to change behavior for all message types. But at the same time, you can decorate and wrap only particular message type pipeline. For instance - `Command` pipeline alone:

Define pre-execute handler interface:

```csharp
public interface IPreExecuteCommandHandler<in TCommand>
                                          where TCommand : ICommand
{
    void Handle(TCommand command);
}
```

Then we need to fill up container with proper registrations:


```csharp
public class MediatorRegistrations : Registry
{
    public MediatorRegistrations()
    {
        Scan(scanner =>
             {
                 ...
                 scanner.AddAllTypesOf(typeof(IPreExecuteCommandHandler<>));
             });

        ...
        For(typeof(ICommandHandler<>)).DecorateAllWith(typeof(CommandPipeline<>));
    }
}
```

We know already decorator pattern. Here we are wrapping all `ICommandHandler` instances with this guy:

```csharp
public class CommandPipeline<TCommand> : ICommandHandler<TCommand>
                            where TCommand : ICommand
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

And we are ready to go. Here is fake pre-execute command handler:

```csharp
public class NeedToExecuteBeforeCommand : IPreExecuteCommandHandler<Command>
{
    public void Handle(Command command)
    {
        // do some voodoo black magic here
    }
}
```

Btw, notice that I do not touch original mediator, nor any other related class. This is power of composition and goes hand by hand with Open/Closed Principle from [S.O.L.I.D](https://en.wikipedia.org/wiki/SOLID_(object-oriented_design)).

## OOTB - MediatR

Reading all posts you might be wondering is there any Out-Of-The-Box solution for dealing with these kind of patterns? Yes, fortunately there are smart guys working on library called [MediatR](https://github.com/jbogard/MediatR). Awesome people and smart techies. Will post soon the same application based on `MediatR` library. You don't need to think about lot of details then, but concepts remain the same.

## Summary

Wuhuuu... This post is tremendously long! I wanted to dump all my thoughts, all ideas and other concerns in single post. Sorry about that. If it's hard to read and too much - please give me feedback - I'll split it up..

Anyway - main idea here in these series of blog posts is to show how can you think about software architecture from different angle, look at set of requirements and business rules from the software maintainer perspective. If you are concerned about the future of the software you are building, you should think about how to make it more maintainable, extendable and readable. Just for your future you. With the power of DI, various patterns and practices - I believe that it is possible.

## Source Code

We have proverb that "*seeing in action is worth thousand words*". Show me the code!
It's available here in [GitHub](https://github.com/valdisiljuconoks/PizzaArchitecture). Fork it down and see for yourself. Give me feedback of what you think!


<br/>
Happy baking round shaped software!
[*eof*]
