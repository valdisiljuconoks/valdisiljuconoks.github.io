---
title: Full support for Asp.Net Mvc areas in EPiServer 7.5
author: valdis
date: 2015-01-22 01:30:00 +0200
categories: [.NET, C#, Episerver, Optimizely]
tags: [.net, c#, episerver, optimizely]
---

This is a blog post about how to add Asp.Net Mvc areas back in town and add full support in EPiServer v7.5. I’ll not spend time to explain what Mvc area is, most probably if you are reading this, you already are looking for a way to add it or adjust it to your needs in your EPiServer project.
 I know that there has been some attempts to add [Mvc areas for EPiServer 7.5](http://world.episerver.com/Blogs/Tuan--Truong/Dates/2014/2/Upgrade-to-EPiServer-Commerce-75--Part-2-MVC-Areas-Support-For-EPiServer-7--above/). Either you have to modify your view engine or add new one. Latter I don’t like at all. [Someone tries](http://world.episerver.com/Blogs/Tuan--Truong/Dates/2014/2/Upgrade-to-EPiServer-Commerce-75--Part-2-MVC-Areas-Support-For-EPiServer-7--above/#comment3613) to inherit from some base controller class in each area and then somehow trick Mvc engine to tell in which area we currently are.

But there is a small challenge for any of methods mentioned and used above. For EPiServer blocks (or partial views in Mvc terms) there are two types of view template registration available:

**a)** Partial rendering registration: This is something that you can find in AlloyTech sample site under `TemplateCoordinator` and is used in cases when conventions based registration (b) is not possible – manually telling EPiServer which view template to use for what content type.

```csharp
[ServiceConfiguration(typeof(IViewTemplateModelRegistrator))]
public class TemplateCoordinator : IViewTemplateModelRegistrator
{
    public void Register(TemplateModelCollection viewTemplateModelRegistrator)
    {
        viewTemplateModelRegistrator.Add(typeof(JumbotronBlock), new TemplateModel
        {
            Tags = new[] { Global.ContentAreaTags.FullWidth },
            AvailableWithoutTag = false,
            Path = BlockPath("JumbotronBlockWide.cshtml")
        });
```

**b)** Conventions based registration: This is the case when EPiServer is asking underlying Mvc view engine collection to find matching partial view for particular content type. So basically what it means is when you will have block of name `EditorialBlock` and you do have a partial view `EditorialBlock.cshtml` (or any other engine powered view template) in some of partial view locations EPiServer will match them and will render that partial view directly when asked to render the block.

![](/assets/img/2015/01/epiareas-conventions.png)

So latter does not really work well in either view engine customization case or route’s data token adjustment case.
 As usual I’m looking for some sort of automation out of the box that does not need to be adjusted every time we add new area, change or delete existing ones.

## Adding Mvc Area via Visual Studio

So as we are using built-in Mvc areas scaffolding support in Visual Studio we are ending up with automatically generated area registration code:

![](/assets/img/2015/01/epiareas-add.png)

```csharp
public class SampleAreaAreaRegistration : AreaRegistration
{
    public override string AreaName
    {
        get
        {
            return "SampleArea";
        }
    }

    public override void RegisterArea(AreaRegistrationContext context)
    {
        context.MapRoute(
            "SampleArea_default",
            "SampleArea/{controller}/{action}/{id}",
            new { action = "Index", id = UrlParameter.Optional }
        );
    }
}
```

This is something we can use for further automation.

## Keep Track of Registered Areas

By taking a deeper look at what happens under the hood when you are registering an area, we can see that there is actually nothing we can use later for enumerating through all registered areas.
So I had to wrap around this code and added my method for registering areas:

```csharp
public class AreaConfiguration
{
    public static void RegisterAllAreas()
    {
        AreaRegistration.RegisterAllAreas();

        var types = Assembly.GetExecutingAssembly().GetExportedTypes();
        var areas = types.Where(t => IsTypeOf(t, typeof(AreaRegistration)));

        foreach (var area in areas)
        {
            var areaRegistration = AreaTable.AddArea(area);

            ...
        }
    }

    private static bool IsTypeOf(Type type, Type parentType)
    {
        return parentType.IsAssignableFrom(type);
    }
}

public class AreaTable
{
    private static readonly AreaCollection _instance = new AreaCollection();

    public static AreaCollection Areas
    {
        get
        {
            return _instance;
        }
    }

    internal static AreaRegistration AddArea(Type area)
    {
        if (area == null)
        {
            throw new ArgumentNullException("area");
        }

        var areaRegistration = (AreaRegistration)Activator.CreateInstance(area);
        Areas.Add(new Area(areaRegistration.AreaName));

        return areaRegistration;
    }
}
```

From the consumer point of view nothing really changes. You just need to change from


```csharp
public class EPiServerApplication : EPiServer.Global
{
    protected void Application_Start()
    {
        AreaRegistration.RegisterAllAreas();
        ...
    }
}
```

to

```csharp
public class EPiServerApplication : EPiServer.Global
{
    protected void Application_Start()
    {
        AreaConfiguration.RegisterAllAreas();
        ...
    }
}
```

Now this new method will give us tracking of registered areas which we will need later when registering conventions based partial views.

## Hook-in to Register Partial Views

While browsing around disassembled `EPiServer Framework` dark cellars I stumbled upon class `EPiServer.Web.Routing.ContentRoute`. Eventually this class turned out to responsible for firing view discovery and registration process. Happens this only in the first request and while content is being routed (I can imagine why this is needed..)

```csharp
public override RouteData GetRouteData(HttpContextBase httpContext)
{
    ...
    if (ContentRoute._isFirstRequest)
        this.FirstIncomingRequest(httpContext);

protected virtual void FirstIncomingRequest(HttpContextBase httpContext)
{
    ...
    if (this._viewRegistrator != null)
        this._viewRegistrator.RegisterViews(httpContext);
```

So we have to hook inside this class to do our step for registering view templates located in registered areas. I noticed that EPiServer is “raising an event” for this case when content is going to be routed:
`this.OnRoutingContent(routingEventArgs)`;


Unfortunately this seems like an event but it’s not. EPiServer provides a delegate here that consumer can set in order to get invoked by the framework (hope they will fix this in future versions).

```csharp
/// <summary>
/// Raised when outgoing virtual path has been created.
///
/// </summary>
public static EventHandler<RoutingEventArgs> RoutingContent;
/// <summary>
/// Raised when an incoming request have been routed to a content instance.
///
/// </summary>
public static EventHandler<RoutingEventArgs> RoutedContent;
```

So I ended up with following class that hooks-in, after first request has been issued and content is going to be routed (built-in view template registration has been already executed and most of block templates have been already discovered and registered) we can step in and finish registration process by walking around partial views locations in registered areas and trying to match against templates registered by run-time in `ContentTypeModelRepository`.

```csharp
[ModuleDependency(typeof(InitializationModule))]
public class AddMvcAreasSupportModule : IInitializableModule
{
    public void Initialize(InitializationEngine context)
    {
        ContentRoute.RoutingContent += OnRoutingContent;
    }

    public void Uninitialize(InitializationEngine context)
    {
    }

    public void Preload(string[] parameters)
    {
    }

    private void OnRoutingContent(object sender, RoutingEventArgs e)
    {
        PartialViewsInAreasRegistrar.Register(new HttpContextWrapper(HttpContext.Current));
        ContentRoute.RoutingContent -= OnRoutingContent;
    }
}
```

Registering partial view templates located in areas

Once we got chance to walk through, discover and register view templates for out blocks located in areas we need some built-in stuff from EPiServer. Therefore created small static factory method to get instance of registrar class:

```csharp
public class PartialViewsInAreasRegistrar
{
    private static volatile bool _isInitialized;
    private static readonly object _lockObj = new object();
    private readonly IContentTypeModelScanner _contentTypeModelScanner;
    private readonly TemplateModelRepository _templateModelRepository;
    private readonly CachingViewEnginesWrapper _viewEngineWrapper;

    public PartialViewsInAreasRegistrar(
        IContentTypeModelScanner contentTypeModelScanner,
        TemplateModelRepository templateModelRepository,
        CachingViewEnginesWrapper viewEngineWrapper)
    {
        _contentTypeModelScanner = contentTypeModelScanner;
        _templateModelRepository = templateModelRepository;
        _viewEngineWrapper = viewEngineWrapper;
    }

    public static void Register(HttpContextBase context)
    {
        lock (_lockObj)
        {
            if (_isInitialized)
            {
                return;
            }

            var reg = ServiceLocator.Current.GetInstance<PartialViewsInAreasRegistrar>();
            reg.RegisterPartials(context);

            _isInitialized = true;
        }
    }

    ...
}
```

And next these are methods that take all the heavy-lifting and walkthrough, discover and register view templates if any:

```csharp
public void RegisterPartials(HttpContextBase context)
{
    var controllerContext = new ControllerContext
    {
        RequestContext = new RequestContext
        {
            RouteData = new RouteData(),
            HttpContext = context
        },
        HttpContext = context
    };

    controllerContext.RouteData.Values["controller"] = "[Unknown]";

    foreach (var area in AreaTable.Areas)
    {
        controllerContext.RouteData.DataTokens["area"] = area.Name;
        FindPartialViewInArea(controllerContext);
    }
}

private void FindPartialViewInArea(ControllerContext controllerContext)
{
    foreach (var type in _contentTypeModelScanner.ContentTypes)
    {
        var contentType = type;
        if (
            _templateModelRepository.List(contentType)
                                    .Any(p => p.TemplateTypeCategory.IsCategory(TemplateTypeCategories.MvcPartialView) && string.IsNullOrEmpty(p.Path)))
        {
            continue;
        }

        // NOTE: this line in Release Mode was scanning only first area.
        // If there were more than one area and requested block would be located
        // in 2nd or any other area, EPiServer would add this template to noHit
        // cache and would assume that view does not exist at all.

        //  Replaced with view search directly via ViewEngines collection.
        // If this hits performance - we would need to search for another solution here.

        // var partialView = _viewEngineWrapper.FindPartialView(controllerContext, contentType.Name);

        var partialView = ViewEngines.Engines.FindPartialView(controllerContext, contentType.Name);

        if (partialView.View == null)
        {
            continue;
        }

        var templateModel = new TemplateModel
        {
            Name = contentType.Name,
            TemplateTypeCategory = TemplateTypeCategories.MvcPartialView
        };

        // This is UPDATED code fragment !
        var view = partialView.View as BuildManagerCompiledView;
        if (view != null)
        {
            templateModel.Path = view.ViewPath;
        }

        _templateModelRepository.RegisterTemplate(contentType, templateModel);
        partialView.ViewEngine.ReleaseView(controllerContext, partialView.View);
    }
}
```

Code fragment does few things:

**a)** First of all it creates artificial controller context, setting everything we have so far (including name of the controller).<br/>
**b)** Then iterates through our registered areas (remember I told that we will need list of areas at some point, well this is 1st usage) and asks to try to find partials that matches in this particular area.<br/>
The only thing Asp.Net needs in order to start looking for a template in areas folders is to set `DataToken` for route data:
`controllerContext.RouteData.DataTokens["area"] = area.Name`;<br/>
**c)** Then we iterate through all template models registered by runtime and we are looking for types that does not have `MvcPartialView` renderer without set path. By EPiServer conventions `MvcPartialView` is template model category used by blocks to render themselves. And if the path is not set for these renderers that means these are automatically conventions-based registered templates.<br/>
**d)** If content type does not have registered template with category `MvcPartialView` with empty path – we need to ask for underlying view engine collection to find partial view: `FindPartialView()`. We set area name in `DataToken` collection for the `RouteData` so Mvc will try to look in that particular area’s partial view locations.<br/>

**UPDATE!** (Added description for case – when you need to preview block located in area using Preview controller and view template located in root):

e) We try to cast to `BuildManagerCompiledView` to get view path to be used in template model information. If we succeed we can write down discovered template path for our template model.

## Return View from Controller’s Action

EPiServer does not have issues invoking controller’s action for particular content type even if controller is located in Mvc area. Problem for Asp.Net Mvc is to find proper view to render `ActionResult`. In this case we need to tell Mvc that we are currently in appropriate area. Someone suggests to do it in base controller for that area. But I know how it usually happens. We all are working in Google’s Copy-Paste department :) You may forget to change area name in that base controller. We need to find a more automated way to set this `DataToken`.
 We can use Mvc’s action filters to intercept call to controller’s action and setup stuff we need before executing action.
In our registrar registration (I know – sounds weird) module need to add another filter:

```csharp
[ModuleDependency(typeof(InitializationModule))]
public class AddMvcAreasSupportModule : IInitializableModule
{
    public void Initialize(InitializationEngine context)
    {
        GlobalFilters.Filters.Add(ServiceLocator.Current.GetInstance<DetectAreaAttribute>());
        ContentRoute.RoutingContent = OnRoutingContent;
        ...
    }
```

New filter `DetectAreaAttribute` (also weird that filter name ends with Attribute, it’s because it could be added directly to controller as well).
 Before we continue with filter we need to make small adjustment to our area registration process. For a sake of performance I decided to keep track of known controllers in area registered in `AreaTable` collection. New version of area registration process is following:


```csharp
public class AreaConfiguration
{
    public static void RegisterAllAreas()
    {
        AreaRegistration.RegisterAllAreas();

        var types = Assembly.GetExecutingAssembly().GetExportedTypes();
        var areas = types.Where(t => IsTypeOf(t, typeof(AreaRegistration)));

        foreach (var area in areas)
        {
            var areaRegistration = AreaTable.AddArea(area);

            var ns = area.Namespace;
            if (string.IsNullOrEmpty(ns))
            {
                continue;
            }

            var allTypesInArea = types.Where(t => t.Namespace != null
                                                  && t.Namespace.StartsWith(ns) && IsTypeOf(t, typeof(Controller)));

            allTypesInArea.ToList().ForEach(t => AreaTable.RegisterController(t.FullName, areaRegistration.AreaName));
        }
    }

    private static bool IsTypeOf(Type type, Type parentType)
    {
        return parentType.IsAssignableFrom(type);
    }
}

public class AreaTable
{
    private static readonly AreaCollection _instance = new AreaCollection();
    private static readonly Dictionary<string, string> _controllersMap = new Dictionary<string, string>();

    internal static AreaRegistration AddArea(Type area)...

    public static string GetAreaForController(string controllerName)
    {
        string value;
        return _controllersMap.TryGetValue(controllerName, out value) ? value : null;
    }

    internal static void RegisterController(string controllerName, string areaName)
    {
        if (!_controllersMap.ContainsKey(controllerName))
        {
            _controllersMap.Add(controllerName, areaName);
        }
    }
}
```


We wrote down all controllers’ `FullName` and area name in which they are located. This gives us a dictionary of controller and particular area. We will use this collection while trying to understand which area we are in while executing controller’s action. Normally this would be carried out by matched Mvc route already.
Filter code is pretty straight forward:


```csharp
public class DetectAreaAttribute : ActionFilterAttribute
{
    public override void OnActionExecuting(ActionExecutingContext filterContext)
    {
        var areaName = AreaTable.GetAreaForController(filterContext.Controller.GetType().FullName);
        if (areaName != null)
        {
            filterContext.RouteData.DataTokens["area"] = areaName;
        }
    }
}
```

Once we set `filterContext.RouteData.DataTokens["area"] = areaName`; Mvc is sure where to look for what. If `DataToken` is not set then Mvc view engines will follow default conventions and most probably will look for templates somewhere under ~/Views/... folder.

Now project and template structure may look and be organized like this:

![](/assets/img/2015/01/epiareas-fin.png)

## Duplicate Block Names

I’m sure that this is not the best style to organize your blocks, but if you have a case when two blocks are defined with the same name (but with different namespaces and GUIDs for sure) and each of them is located in different area – this code will not handle that properly. If you have such cases, please drop me a note – I’m looking for a way to decorate block definition with some sort of area name where template is located. But again, I’m strongly don’t recommend to introduce another misunderstanding in your project and avoid such cases. Most easiest way to get rid of this is to rename one of the blocks to other name and template respectively.

## Wrapping it up

In general this was interesting journey for me inside template registration process and how properly one should be implemented to hook in existing discovery and registration process.
I shuffled together a [sample project on github.com](https://github.com/valdisiljuconoks/EPiServer.MvcAreas/tree/master) where you can take a look at complete source code. It’s not production quality library yet. If you are interested in getting one with few configuration options you may expect as a consumer – drop me a note, we will definitely figure out something.

Happy coding!

[*eof*]
