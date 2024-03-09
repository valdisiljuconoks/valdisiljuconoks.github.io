---
title: Format Your Exception Message Properly
author: valdis
date: 2014-08-12 23:45:00 +0200
categories: [.NET, C#]
tags: [.net, c#]
---

## Not so Nice Failure
I was hacking around [NServiceBus](http://particular.net/service-platform) (NSB) application and came across pretty unpleasant failure from `NSB`. So in short we were using `Unicast` bus that basically means that producer-side of the message has to have a configuration set to which `NSB` endpoint particular message should be sent. In this case we were using [Microsoft Azure Storage Queues](http://azure.microsoft.com/en-us/documentation/articles/storage-dotnet-how-to-use-queues/#what-is) as physical transport level.
In case when you don’t have a particular configuration to instruct `NSB` where the message has to be delivered you may encounter following exceptional message:

![](/assets/img/2014/08/nsb-exc.png)

Which does not tell much about root cause of the exception.
Stack trace looks even more obscure:

```
at NServiceBus.Azure.Transports.WindowsAzureStorageQueues.AzureMessageQueueSender.Send(TransportMessage message, Address address) in y:/BuildAgent/work/ba77a0c29cee2af1/src/NServiceBus.Azure.Transports.WindowsAzureStorageQueues/AzureMessageQueueSender.cs:line 37
 at NServiceBus.Unicast.Behaviors.DispatchMessageToTransportBehavior.Invoke(SendPhysicalMessageContext context, Action next) in y:/BuildAgent/work/31f8c64a6e8a2d7c/src/NServiceBus.Core/Unicast/Behaviors/DispatchMessageToTransportBehavior.cs:line 77
 at NServiceBus.Pipeline.BehaviorChain`1.InvokeNext(T context) in y:/BuildAgent/work/31f8c64a6e8a2d7c/src/NServiceBus.Core/Pipeline/BehaviorChain.cs:line 51 at NServiceBus.Pipeline.BehaviorChain`1.Invoke(T context) in y:/BuildAgent/work/31f8c64a6e8a2d7c/src/NServiceBus.Core/Pipeline/BehaviorChain.cs:line 36
 at NServiceBus.Pipeline.PipelineExecutor.Execute[T](BehaviorChain`1 pipelineAction, T context) in y:/BuildAgent/work/31f8c64a6e8a2d7c/src/NServiceBus.Core/Pipeline/PipelineExecutor.cs:line 139
 at NServiceBus.Pipeline.PipelineExecutor.InvokeSendPipeline(SendOptions sendOptions, TransportMessage physicalMessage) in y:/BuildAgent/work/31f8c64a6e8a2d7c/src/NServiceBus.Core/Pipeline/PipelineExecutor.cs:line 97
 at NServiceBus.Unicast.Behaviors.CreatePhysicalMessageBehavior.Invoke(SendLogicalMessagesContext context, Action next) in y:/BuildAgent/work/31f8c64a6e8a2d7c/src/NServiceBus.Core/Unicast/Behaviors/CreatePhysicalMessageBehavior.cs:line 57
 at NServiceBus.Pipeline.BehaviorChain`1.InvokeNext(T context) in y:/BuildAgent/work/31f8c64a6e8a2d7c/src/NServiceBus.Core/Pipeline/BehaviorChain.cs:line 51 at NServiceBus.Pipeline.BehaviorChain`1.Invoke(T context) in y:/BuildAgent/work/31f8c64a6e8a2d7c/src/NServiceBus.Core/Pipeline/BehaviorChain.cs:line 36
 at NServiceBus.Pipeline.PipelineExecutor.Execute[T](BehaviorChain`1 pipelineAction, T context) in y:/BuildAgent/work/31f8c64a6e8a2d7c/src/NServiceBus.Core/Pipeline/PipelineExecutor.cs:line 139
 at NServiceBus.Pipeline.PipelineExecutor.InvokeSendPipeline(SendOptions sendOptions, IEnumerable`1 messages) in y:/BuildAgent/work/31f8c64a6e8a2d7c/src/NServiceBus.Core/Pipeline/PipelineExecutor.cs:line 78
 at NServiceBus.Unicast.UnicastBus.InvokeSendPipeline(SendOptions sendOptions, List`1 messages) in y:/BuildAgent/work/31f8c64a6e8a2d7c/src/NServiceBus.Core/Unicast/UnicastBus.cs:line 749
 at NServiceBus.Unicast.UnicastBus.SendMessages(SendOptions sendOptions, List`1 messages) in y:/BuildAgent/work/31f8c64a6e8a2d7c/src/NServiceBus.Core/Unicast/UnicastBus.cs:line 725
 at NServiceBus.Unicast.UnicastBus.Send(Object[] messages) in y:/BuildAgent/work/31f8c64a6e8a2d7c/src/NServiceBus.Core/Unicast/UnicastBus.cs:line 571
 at NServiceBus.Unicast.UnicastBus.Send(Object message) in y:/BuildAgent/work/31f8c64a6e8a2d7c/src/NServiceBus.Core/Unicast/UnicastBus.cs:line 555
….
at lambda_method(Closure , ControllerBase , Object[] )
 at System.Web.Mvc.ActionMethodDispatcher.Execute(ControllerBase controller, Object[] parameters)
 at System.Web.Mvc.ReflectedActionDescriptor.Execute(ControllerContext controllerContext, IDictionary`2 parameters)
 at System.Web.Mvc.ControllerActionInvoker.InvokeActionMethod(ControllerContext controllerContext, ActionDescriptor actionDescriptor, IDictionary`2 parameters)
 at System.Web.Mvc.Async.AsyncControllerActionInvoker.ActionInvocation.InvokeSynchronousActionMethod()
 at System.Web.Mvc.Async.AsyncControllerActionInvoker.b__39(IAsyncResult asyncResult, ActionInvocation innerInvokeState)
 at System.Web.Mvc.Async.AsyncResultWrapper.WrappedAsyncResult`2.CallEndDelegate(IAsyncResult asyncResult)
 at System.Web.Mvc.Async.AsyncResultWrapper.WrappedAsyncResultBase`1.End()
 at System.Web.Mvc.Async.AsyncResultWrapper.End[TResult](IAsyncResult asyncResult, Object tag)
 at System.Web.Mvc.Async.AsyncControllerActionInvoker.EndInvokeActionMethod(IAsyncResult asyncResult)
 at System.Web.Mvc.Async.AsyncControllerActionInvoker.AsyncInvocationWithFilters.b__3f()
 at System.Web.Mvc.Async.AsyncControllerActionInvoker.AsyncInvocationWithFilters.<>c__DisplayClass48.b__41()
```

## Traceable Failure Sample

This reminds me how important actually is to capture failure case and provide meaningful exception message to the consumer (in this case DevOps guy who is monitoring the system or Dev who is developing the site) in order to relief search of exception origin.

Once I was refactoring default `Asp.Net Mvc` 5 sample file with built-in authentication.

You may have seen `AccountController` with multiple constructors. One of them is accepting instance of `UserManager` another constructor is parameter-less providing instance of `UserManager` class by constructing it manually:

```csharp
public AccountController() :
    this(new UserManager(new UserStore(new ApplicationDbContext())))
{
}

public AccountController(UserManager userManager)
{
    UserManager = userManager;
}
```

I thought it should be good actually to utilize `Inversion of Control` capabilities and inject proper instance of ‘UserManager’ class when requested by dependency resolver. So I got rid of this extra constructor, but forgot to configure IoC container properly.
In a return what I got was a nicely formatted exception message that IoC (in this case [StructureMap](http://docs.structuremap.net/)) is not able to resolve requested service type and here goes why:

![](/assets/img/2014/08/sm-exc.png)

Highlighted lines even shows which constructor was picked up by resolver and what kind of argument instance was used (`* Default of …` means that `IoC` library tries to resolve requested type further down in container).

Reading this particular exception message I can fully understand where is the case and can start to investigate why IoC was not able to resolve requested type.

Forgot to instruct how to get instance of `UserManager` class:

```csharp
For<UserManager>()
    .Use(ctx => new UserManager(new UserStore(ctx.GetInstance())));
```


Prepare to fail.

![](http://www.cs.cmu.edu/~fgandon/miscellaneous/japan/Image2.gif)
 > Nana korobi, ya oki
— Japanese proverb

Happy coding!

[*eof*]
