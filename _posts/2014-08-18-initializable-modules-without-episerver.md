---
title: Initializable modules without EPiServer
author: valdis
date: 2014-08-18 11:30:00 +0200
categories: [.NET, C#, Episerver, Optimizely]
tags: [.net, c#, episerver, optimizely]
---

[EPiServer initialization modules](http://world.episerver.com/Documentation/Items/Developers-Guide/EPiServer-CMS/75/Initialization/Creating-an-initialization-module/) is a great feature to provide plugin facilities for the hosting application. I find initialization modules pretty handful in cases when you need to initialize some library, configure some stuff or do whatever needed at application start-up.
 However there are times when you have to develop site without wonderful backup platform that takes all boring boilerplate code away and provides with much awesome stuff and features.
 Let’s switch to ordinary Asp.Net Mvc application initialization code:

```csharp
[assembly: OwinStartupAttribute(typeof(WebApplication1.Startup))]
namespace WebApplication1
{
    public partial class Startup
    {
        public void Configuration(IAppBuilder app)
        {
            ConfigureAuth(app);
        }
    }
}
```

In order to add new steps for initialization you may need to add new method invocation under or above `ConfigureAuth(app)` method:

```csharp
public void Configuration(IAppBuilder app)
{
    ConfigureAuth(app);
    // OtherSetupMethod();
}
```

Then you may need to create new `Startup` class partial fragment that is located under `App_Start` folder just to follow used conventions in default project template. Tiny place for a small library :)

## Initialization Modules

So I took inspiration EPiServer gave and created a small initialization module library that does exactly what it says – you are able to create a initialization modules which will be called during application start-up or at any your preferred time during app life cycle.

So by using [InitializationModule library](https://github.com/valdisiljuconoks/InitializableModule) you are able to rewrite user authentication setup in default Asp.Net Mvc template with following code.

```csharp
[assembly: OwinStartupAttribute(typeof(WebApplication1.Startup))]
namespace WebApplication1
{
    public partial class Startup
    {
        public void Configuration(IAppBuilder app)
        {
            var process = new ModuleExecutionProcess(app);
            var context = process.Execute();
        }
    }
}
```

You can now rewrite authentication setup code using initialization modules:

```csharp
namespace WebApplication1
{
    public partial class Startup : IInitializableModule
    {
        private readonly IAppBuilder _app;

        public Startup(IAppBuilder app)
        {
            this._app = app;
        }

        public void Initialize()
        {
            // Configure the db context, user manager and signin manager to use a single instance per request
            _app.CreatePerOwinContext(ApplicationDbContext.Create);
            _app.CreatePerOwinContext(ApplicationUserManager.Create);
            _app.CreatePerOwinContext(ApplicationSignInManager.Create);

            // …
```

Taking a look at `Global.asacx.cs` file content it seems very good candidate for a initialization modules:

```csharp
public class MvcApplication : System.Web.HttpApplication
{
    protected void Application_Start()
    {
        AreaRegistration.RegisterAllAreas();
        GlobalConfiguration.Configure(WebApiConfig.Register);
        FilterConfig.RegisterGlobalFilters(GlobalFilters.Filters);
        RouteConfig.RegisterRoutes(RouteTable.Routes);
        BundleConfig.RegisterBundles(BundleTable.Bundles);
    }
}
```

## Dependencies between modules

Initialization modules provides built-in support for specifying cross-module dependencies. Let’s say we do have following modules:

```csharp
public class Module1 : IInitializableModule
{
    public void Initialize()
    {
    }
}

[ModuleDependency(typeof(Module1))]
public class Module2 : IInitializableModule
{
    public void Initialize()
    {
    }
}

[ModuleDependency(typeof(Module2))]
public class Module3 : IInitializableModule
{
    public void Initialize()
    {
    }
}

[ModuleDependency(typeof(Module2))]
public class Module4 : IInitializableModule
{
    public void Initialize()
    {
    }
}

[ModuleDependency(typeof(Module4))]
public class Module5 : IInitializableModule
{
    public void Initialize()
    {
    }
}
```

Using `ModuleDependency` attribute you can specify that one module is dependent on others’ module execution result, e.g., `Module3` will not be executed before `Module2`, `Module5` before `Module4`, etc.

Graphically this looks like this:

![](/assets/img/2014/08/depend.PNG)

Engine takes care of proper execution order for the dependent modules. However – it’s defined which module will be executed first – module `Module3` or module `Module4`. If you need to define a specific execution order for the modules on "the same level" – probably they are not on the same level – somebody has dependency on other.

## Module resolution

Initialization modules sometimes may depend not only on other modules but they may require something from outer world as well - it’s an ordinary case in enterprise applications – some dependencies injected.

It’s possible to provide module resolution function to [InitializationModule library](https://github.com/valdisiljuconoks/InitializableModule) in order to specify how modules are resolved in [RRR](http://blog.ploeh.dk/2010/09/29/TheRegisterResolveReleasepattern/) cycle.

Let’s say in this particular case we are using `StructureMap DI` container (particular implementation of container does not play any role in this case as library does not care about implementation - power of `System.Func`).

```csharp
var process = new ModuleExecutionProcess(ObjectFactory.Container.GetInstance);
var context = process.Execute();
```

In this case every discovered module will be resolved using specified function (`Func`).
It may seem a bit of [Service locator anti-pattern](http://blog.ploeh.dk/2010/02/03/ServiceLocatorisanAnti-Pattern/). But it’s a bit better than creating a [conforming container](http://blog.ploeh.dk/2014/05/19/conforming-container/) for every DI framework out there.

[InitializationModule library](https://github.com/valdisiljuconoks/InitializableModule) is available in [nuget.org feed](https://www.nuget.org/packages/TechFellow.InitializableModule/) and source code is  pushed to [GitHub](https://github.com/valdisiljuconoks/InitializableModule).

[*eof*]
