---
title: Pragmatic problem solving – Answer to the EPiServer forum question
author: valdis
date: 2014-04-03 23:25:00 +0200
categories: [.NET, C#, Episerver, Optimizely]
tags: [.net, c#, episerver, optimizely]
---

At some point I was questioned about how I’m learning and finding the answers.
Generally for the learning process a huge inspiration came from “Pragmatic Thinking and Learning: Refactor Your Wetware” by Andy Hunt.

![](/assets/img/2014/04/ahptl.jpg)

> Software development happens in your head. Not in an editor, IDE, or design tool. You’re well educated on how to work with software and hardware, but what about wetware—our own brains? Learning new skills and new technology is critical to your career, and it’s all in your head.

## Walk-through learning and research
I want you to walk with me through my learning and research process and activities while answering to one of interesting questions on [stackoverflow.com](http://stackoverflow.com/questions/18640042/structuremap-exception-code-202-when-opening-visitor-group-tab-within-episerver).
Question essentially is pretty interesting in its own nature. It was asked at the beginning of 2013 September. Question was still hanging around up till 2014 April with no answer to it. Either it was too hard (which I don’t believe was the case) or it just flushed away together with other stuff that is screaming for our attention (read – spam).
Anyway: discovery process for the answer was thrilled enough to pick it up for adding more details to it.

## Original question
Original question was about `VisitorGroups` and custom criteria for them.
xlevel asked on SO: *”I’ve created a visitor group and I’m trying to inject a class into it.I have the class all wired up and running fine in the site where I am injecting it into a block.”*

Author also provided some hints on idea he had behind the error: *”I suspect the issue is that the Modules area have their own `StructureMap Container`. Is this the case? And if so, how is the best way to make sure your mappings are carried through?”*

## Finding the answer
### Verification
My first rule on find the answer is to verify existing theorems. So I should verify idea that author gave in order to understand is this the real situation or just a wild guess. Theory is:  dependency container **is not shared** between *ordinary* CMS parts (blocks, pages, etc) and *module* – like VisitorGroup section in CMS.

First thing that I needed to find is a way how EPiServer is creating a controller for visitor groups because original exception is located somewhere deep in dark corridors of EPiServer Framework platform code:

```csharp
…
[ActivationException: Activation error occurred while trying to get instance of type VisitorGroupsController, key “”]
EPiServer.ServiceLocation.ServiceLocatorImplBase.GetInstance(Type serviceType, String key) +156
 EPiServer.Shell.Web.Mvc.ModuleControllerFactory.CreateController(RequestContext requestContext, String controllerName) +351
 EPiServer.Shell.Web.Mvc.ModuleMvcHandler.ProcessRequestInit(HttpContextBase httpContext) +105
 EPiServer.Shell.Web.Mvc.ModuleMvcHandler.BeginProcessRequest(HttpContextBase httpContext, AsyncCallback callback, Object state) +17
 System.Web.CallHandlerExecutionStep.System.Web.HttpApplication.IExecutionStep.Execute() +12272575
 System.Web.HttpApplication.ExecuteStep(IExecutionStep step, Boolean& completedSynchronously) +288
```

From exception stacktrace it’s obvious that we need to trace down how `VisitorGroupsController` is constructed via `ModuleControllerFactory`.
Let’s sneak peek in EPiServer source code for evidence:

```csharp
public IController CreateController(RequestContext requestContext, string controllerName)
{
    …
    IController controller = this._serviceLocator.GetInstance(serviceType) as IController;
    …
}
```

Service locator is given as dependency while constructing a type (constructor dependency injection):

```csharp
public ModuleControllerFactory(IServiceLocator serviceLocator)
{
    this._serviceLocator = serviceLocator;
}
```

Next I need to check how exactly `ModuleControllerFactory` is constructed (most probably via some automatic type creation – but just to verify).
Using cool Visual Studio features you are able to find any reference to this type. I found one in `ShellInitialization` module that is responsible for various type initialization routine and setup. during app startup. And found our type in interest:

```csharp
public void ConfigureContainer(ServiceConfigurationContext context)
{
    context.Container.Configure((Action) (ce =>
    {
        ce.For<ModuleTable>().Singleton();
        ce.For<IModuleControllerFactory>().Use<ModuleControllerFactory>();
        …
```

As `Shell` is initialized via `InitializableModule` infrastructure and IoC container is configured in the same way as other modules are (even custom) this rejects idea that dependency container is not shared across various parts of the EPiServer CMS platform.

### Let’s look at origin
Next thing in search and finding answer is to return to origin of the issue – in this case back to exception that is thrown and guy who is causing the exception.

```
StructureMap Exception Code: 202
 No Default Instance defined for PluginFamily EPiServer.ServiceLocation.ServiceAccessor`1[[EPiServer.Templates.Alloy.Business.Initialization.IMemberFactory, EPiServer.Templates.Alloy.Mvc, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null]], EPiServer.Framework, Version=7.5.1003.0, Culture=neutral, PublicKeyToken=8fe83dea738b45b7
```

This is *really* something wrong with IoC container and registered types there.
Let’s return to our guy who is causing the error:

```csharp
[VisitorGroupCriterion(Category = “Custom Criterion”,
DisplayName = “IsMemberCriterion”,
Description = “IsMemberCriterion”)]
public class IsMemberCriterion : CriterionBase
{
    public Injected MemberFactory { get; set; }

    public override bool IsMatch(IPrincipal principal, HttpContextBase httpContext)
    {
        return MemberFactory.Service.Match(httpContext);
    }
}
```

One of the trick would be to refactor injection from `Injected` to constructor injection:

```csharp
public IsMemberCriterion(IMemberFactory factory)
{
    _factory = factory;
}
```

This is cool except that EPiServer VisitorGroup controller does not like this while constructing UI section and trying to create criterion instances:

```csharp
 [MissingMethodException]: No parameterless constructor defined for this object.
 at System.RuntimeTypeHandle.CreateInstance(RuntimeType type, Boolean publicOnly, Boolean noCheck, Boolean& canBeCached, RuntimeMethodHandleInternal& ctor, Boolean& bNeedSecurityCheck)
 at System.RuntimeType.CreateInstanceSlow(Boolean publicOnly, Boolean skipCheckThis, Boolean fillCache, StackCrawlMark& stackMark)
 at System.RuntimeType.CreateInstanceDefaultCtor(Boolean publicOnly, Boolean skipCheckThis, Boolean fillCache, StackCrawlMark& stackMark)
 at System.Activator.CreateInstance(Type type, Boolean nonPublic)
 at System.Activator.CreateInstance(Type type)
 at EPiServer.Cms.Shell.UI.Controllers.VisitorGroupsController.CriteriaUI(String typeName)
 at lambda_method(Closure , ControllerBase , Object[] )
 at System.Web.Mvc.ReflectedActionDescriptor.Execute(ControllerContext controllerContext, IDictionary`2 parameters)
 at System.Web.Mvc.ControllerActionInvoker.InvokeActionMethod(ControllerContext controllerContext, ActionDescriptor actionDescriptor, IDictionary`2 parameters)
 at System.Web.Mvc.ControllerActionInvoker.<>c__DisplayClass13.b__10()
 at System.Web.Mvc.ControllerActionInvoker.InvokeActionMethodFilter(IActionFilter filter, ActionExecutingContext preContext, Func`1 continuation)
 at System.Web.Mvc.ControllerActionInvoker.InvokeActionMethodFilter(IActionFilter filter, ActionExecutingContext preContext, Func`1 continuation)
 at System.Web.Mvc.ControllerActionInvoker.InvokeActionMethodWithFilters(ControllerContext controllerContext, IList`1 filters, ActionDescriptor actionDescriptor, IDictionary`2 parameters)
 at System.Web.Mvc.ControllerActionInvoker.InvokeAction(ControllerContext controllerContext, String actionName)
 at System.Web.Mvc.Controller.ExecuteCore()
 at System.Web.Mvc.ControllerBase.Execute(RequestContext requestContext)
 at EPiServer.Shell.Web.Mvc.ModuleMvcHandler.ProcessController(IController controller)
 at EPiServer.Shell.Web.Mvc.ModuleMvcHandler.BeginProcessRequest(HttpContextBase httpContext, AsyncCallback callback, Object state)
 at System.Web.HttpApplication.CallHandlerExecutionStep.System.Web.HttpApplication.IExecutionStep.Execute()
 at System.Web.HttpApplication.ExecuteStep(IExecutionStep step, Boolean& completedSynchronously)
```

There is another workaround to this problem – let’s define parameter-less constructor and get instance of required type manually:

```csharp
public class IsMemberCriterion : CriterionBase
{
    private readonly IMemberFactory _factory;

public IsMemberCriterion() : this(ServiceLocator.Current.GetInstance<IMemberFactory>())
{
}

public IsMemberCriterion(IMemberFactory factory)
{
    _factory = factory;
}
```

This works and now everybody is happy: `IMemberFactory` is injected from IoC container, controller is able to create new criterion instance. Maybe everybody is satisfied except me – for me this really seems more like workaround and not proper solution. So let’s keep digging..

### Finding the Solution
According to pragmatic learning theory – you have to be present in surroundings of the issue and imagine where the problem could be originating from.
What didn’t left me was an idea that original problem is in the definition line of `Injected`. I’ve seen a lot of these style of injection in EPiServer source code and also in our own code.

```csharp
public Injected MemberFactory { get; set; }
```

Looking at source code of `Injected` we can see that somebody will have to provide `ServiceAccessor` type while constructing instance of injected property. `ServiceAccessor` is just an ordinary .Net delegate.

```csharp
public delegate TService ServiceAccessor();
```

Next thing what we can do is to try find how `ServiceAccessor` is used across EPiServer libraries and other various modules. Again using some cool Visual Studio features like `Find All References` you can easily spot usages of the type. For the `ServiceAccessor` there are zillions of various usages but one interesting usage just popup. It was located in EPiServer.Framework assembly:

```csharp
...
public class FrameworkInitialization : IConfigurableModule, ...
{
    public void ConfigureContainer(ServiceConfigurationContext context)
    {
        context.Container.Configure(ce =>
            ce.For<ServiceAccessor<HttpContextBase>>()
              .Use((ServiceAccessor<HttpContextBase>)(() => ...)));
}
```

This fragment `ce.For<ServiceAccessor<HttpContextBase>>` definitely approves that somebody is really registering service accessors for various types.
The next usage of `ServiceAccessor` finally gave me more insight of what’s really going on under the hood. That was `ServiceContainerInitialization` module.

I landed on small internal class:

```csharp
private static class ServiceAccessorFactory
```

That is responsible for providing factory style methods for creating new `ServiceAccessor`s.
Few steps up on usage and calling chain I ended in method `public void Process(Type type, Registry registry)` that takes care of service registration into the IoC container.
Method is asking for a list of types that are decorated with something that is implementing `IServiceConfiguration` interface.

```csharp
foreach (IServiceConfiguration attribute in type.GetCustomAttributes(typeof (IServiceConfiguration), true))
{
    …
```

### Formulating the answer
Next step in research process naturally is structuring my findings and formulating the answer to the original question.
And here goes what I understood: *”If you decorate something with attribute that is implementing `IServiceConfiguration` then somebody else takes care of that and registers service accessor in IoC container for this class according to specified life-cycle and creation settings.”*.

So in order to get your custom class as target for `Injected` you have to tell EPiServer how that `Type` should be constructed, for how long it should live and maybe other infrastructure level information in order to instantiate that accordingly. The final solution to the problem is following code snippet:

```
 [ServiceConfiguration(Lifecycle = ServiceInstanceScope.Unique, ServiceType = typeof(IMemberFactory))]
 public class MemberFactory : IMemberFactory
```

As it turned out the solution was pretty simple and easy but the way to get to the solution wasn’t so straight..

Happy coding!

[*eof*]
