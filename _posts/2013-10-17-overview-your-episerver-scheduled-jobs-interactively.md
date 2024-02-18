---
title: Overview your EPiServer scheduled jobs interactively
author: valdis
date: 2013-10-17 09:00:00 +0200
categories: [.NET, C#, Add-On, Episerver, Optimizely]
tags: [.net, c#, add-on, episerver, optimizely]
---

EPiServer scheduled jobs list in larger enterprise scale projects may get huge enough in order not to be able to overview them.

Few pain points around scheduled jobs presentation in administrative EPiServer interface:

* Section for listing scheduled jobs usually presents list of jobs in some unpredictable order.
* It’s hard to tell details of the job, which of them is enabled, which of them failed last time, what is the schedule interval, etc.
* It’s also not easy to tell which of the jobs are running at any given moment.

These are just a few points that lead me to develop EPiServer scheduled jobs overview plugin.

Everything you need to do in order to add a plugin is to install it via NuGet (just search for `scheduledjobs` in EPiServer NuGet feed – [http://nuget.episerver.com](http://nuget.episerver.com)).

![](/assets/img/2013/10/1.png)

After package installation and site build completes plugin is accessible in “Tools” section in Admin page.

![](/assets/img/2013/10/2.png)

Plugin shows you all registered scheduled jobs in the system in one table with additional essential information.

![](/assets/img/2013/10/3.png)

It’s also possible to get information on which job is currently running (if `Auto-refresh` is enabled):

![](/assets/img/2013/10/4.png)

Plugin is available for following versions:
1.CMS 6 (Id: `TechFellow.ScheduledJobsOverview.CMS6`)
2.CMS 7 (Id: `TechFellow.ScheduledJobsOverview`)
3.CMS 7 as AddOn ([on GitHub](https://github.com/valdisiljuconoks/TechFellow.ScheduledJobOverview/raw/master/ScheduledJobOverview/Packages/TechFellow.ScheduledJobOverview.AddOn.1.4.0.nupkg))

Source located in GitHub – [https://github.com/valdisiljuconoks/TechFellow.ScheduledJobOverview](https://github.com/valdisiljuconoks/TechFellow.ScheduledJobOverview)

## Behind the scene

While developing the plugin I discovered a few interesting things to share in following sections.

### Registering Routes in CMS7 as plugin

In order to register some custom routes in EPiServer 7 CMS there is a special method in `EPiServer.Global` class which you can override and add your own custom routes.

```csharp
public class Global : EPiServer.Global
{
    protected override void RegisterRoutes(RouteCollection routes)
    {
        base.RegisterRoutes(routes);
        routes.MapSomething();
    }
}
```

However if you are developing a plugin, usually you don’t want to make your library users to add something somewhere explicitly and then upon removal you as a user of the library need to remove those additions, etc.

My idea is that plugin must add everything it needs and remove everything it needed upon removal. As EPiServer CMS does not provide any events or other kind of extension point around route registration what I came up is to use Http module for this which is dynamically registered using Microsoft.Web.Infrastructure library.

First what you need to do is to dynamically register module:

```csharp
[ModuleDependency(typeof(InitializationModule))]
[InitializableModule]
public class InitializeModule : IInitializableModule
{
    public void Preload(string[] parameters)
    {
    }

    public void Initialize(InitializationEngine context)
    {
#if !ADDON && !CMS6
        DynamicModuleUtility.RegisterModule(typeof(WorkaroundRouteRegistrationHttpModule));
#endif
     }

     public void Uninitialize(InitializationEngine context)
     {
     }
}
```

Compile time constants (ADDON and CMS6) are introduced in order to support multiple CMS versions and installation scenario within the same code-base (more that below).

Then workaround module itself is just a registration of the necessary routes:

```csharp
public class WorkaroundRouteRegistrationHttpModule : IHttpModule
{
    private static bool isInitialized;

    public void Init(HttpApplication context)
    {
        if (isInitialized)
        {
            return;
        }

        RouteTable.Routes.MapRoute(..);

        lock (context)
        {
            isInitialized = true;
        }
    }

    public void Dispose()
    {
    }
}
```

### Supporting multiple versions within the same code-base

Goal was to enable multiple version support within the same code-base. For multiple version support I had to create separate assemblies (projects) to target different EPiServer assemblies and to define different compile time constants.

Following is a summary of exceptions I needed to take to workaround issues related to multiple versions:

1.*Route registration*. need to register routes in separate `Http module` if running in CMS7.
2.*Admin Plugin registration*. If you are supporting AddOn installation mode as well you need to register plugin with different settings.

```csharp
namespace TechFellow.ScheduledJobOverview
{
#if ADDON
    [GuiPlugIn(DisplayName = "Scheduled jobs overview", UrlFromModuleFolder = "Overview", Area = PlugInArea.AdminMenu)]
#else
    [GuiPlugIn(DisplayName = "Scheduled jobs overview", Url = "~/modules/" + Const.ModuleName + "/Overview", Area = PlugInArea.AdminMenu)]
#endif
    public class ToolsPluginRegistration
    {
    }
}
```

3.*Client resource delivery*. There is difference when you install plugin as AddOn you need to deliver client side resources (like scripts, images) using `EPiServer.Shell.Paths` class to resolve location of the resource. When plugin is installed as NuGet package – files are delivered from assembly itself (more on this in section “Single-file deployment”).

For instance to include script on the page it’s convenient to create some helper methods:

```csharp
public static string GetJavascriptIncludes(this ClientScriptManager clientScript, string file)
{
    return string.Format("",
                         RuntimeInfo.IsModule()
                                 ? Paths.ToClientResource(Const.ModuleName + ".AddOn", "Scripts/" + file)
                                 : clientScript.GetWebResourceUrl(typeof(ClientScriptManagerExtensions),
                                                                  ResourceProvider.CreateResourceUrl("Scripts", file)));
}
```

To understand if the plugin is running in `AddOn` mode, I came up with following helper method:

```csharp
using EPiServer.ServiceLocation;
using EPiServer.Shell.Modules;

namespace TechFellow.ScheduledJobOverview
{
    public static class RuntimeInfo
    {
        private static bool isInitialized;
        private static bool isModule;
        private static readonly object syncRoot = new object();

        public static bool IsModule()
        {
            if (isInitialized)
            {
                return isModule;
            }

#if !CMS6
            if (ServiceLocator.Current == null)
            {
                return isModule;
            }

            lock (syncRoot)
            {
                ShellModule module;
                isModule = ServiceLocator.Current.GetInstance().TryGetModule(typeof(RuntimeInfo).Assembly, out module);
            }
#endif
            isInitialized = true;
            return isModule;
        }
    }
}

namespace EPiServer.ServiceLocation
{
}
```

**NB!** The last line with empty namespace is there just because of some `ReSharper` or any other code clean-up tool. If code will be opened in solution that is targeting different CMS version (v6) it will not compile code fragment around ServiceLocator but usage on top (`EPiServer.ServiceLocation`) will break. If you will fully qualified type name for `ServiceLocator` -> next time when you will reformat code in solution that is targeting CMS 7 for instance, usually namespace before the type will be moved out to usage list. And we will be back with the same issue :)

### Enabling client-side interactivity

Feature “Auto-refresh”  in plugin is using client side interactivity based on `AngluarJs` library that is used to bind on the page Json object received back from the server. `Angular.js` library is packed together with plugin (even this is not correct) just to enable single file deployment scenarios.

To enable “Auto-refresh” feature we need to create 2 parts:

1.*Create a service*. Service that will deliver information about current state of the plugin subsystem:

```csharp
[AcceptVerbs(HttpVerbs.Get)]
public JsonResult GetList()
{
    var repository = new JobRepository();
    return Json(repository.GetList(), JsonRequestBehavior.AllowGet);
}
```

Repository is pretty straight forward (you need “join” scheduled job list with discovered plugins just for the reason that there might be jobs that are not yet ran and they are not registered as scheduled jobs but are discovered just as plugins):

```csharp
public List GetList()
{
    var plugIns = PlugInLocator.Search(new ScheduledPlugInAttribute()).ToList();

    return (from plugin in plugIns
            let job = ScheduledJob.List().FirstOrDefault(j => j.TypeName == plugin.TypeName &&
                                                              j.AssemblyName == plugin.AssemblyName)
            let attr = plugin.GetAttribute()
            select new JobDescriptionViewModel
                       {
                               Id = plugin.ID,
                               InstanceId = job != null ? job.ID : Guid.Empty,
                               Name = attr.DisplayName,
                               Description = attr.Description,
                               IsEnabled = (job != null && job.IsEnabled),
                               Interval = job != null ? string.Format("{0} ({1})", job.IntervalLength, job.IntervalType) : "",
                               IsLastExecuteSuccessful = (job != null && !job.HasLastExecutionFailed ? true : (bool?)null),
                               LastExecute = job != null ? job.LastExecution : (DateTime?)null,
                               AssemblyName = plugin.AssemblyName,
                               TypeName = plugin.TypeName,
                               IsRunning = job != null && ScheduledJob.IsJobRunning(job.ID),
                       }).OrderBy(j => j.Name).ToList();
}
```

2.*Create a view*. Client side view that will be used as template to bind received data back.
And client-side script that enables automatic fetching of current scheduled job subsystem:

```js
;(function() {
    angular.module('schApp', [])
        .controller('scheduledJobsController', function($scope, $http, $window,
                                                        $timeout, serviceUrl, detailsUrl) {
            $scope.fetch = function() {
                if ($scope.autoRefresh) {
                    $http.get(serviceUrl + 'GetList').success(function (data) {
                        try {
                            $scope.jobs = angular.fromJson(data);
                        } catch (e) {
                            // error may occur when service returns html for login page instead of json
                            // (unauthorized access, session expired, etc)
                        }
                    });
                    $timeout(function() { $scope.fetch(); }, 5000);
                }
            };

            $scope.$watch('autoRefresh', function(newValue) {
                if (newValue) {
                    $scope.fetch();
                }
            });

            $scope.autoRefresh = true;

        })
})();
```

### Customizing EPiServer's ScheduledJob details page
When accessing scheduled jobs details page new button appears that enables administrator to go back to overview page.

![](/assets/img/2013/10/5.png)

It’s done using Asp.Net control adapters infrastructure.
You just need to deploy `YourAdapterMappings.browser` file to `App_Browsers` folder in your target web site.

```xml
<browsers>
    <browser refID="Default">
        <controlAdapters>
            <adapter controlType="EPiServer.UI.Admin.DatabaseJob"
                     adapterType="TechFellow.ScheduledJobOverview.DatabaseJobAdapter" />
        </controlAdapters>
    </browser>
</browsers>
```

DatabaseJobAdapter.cs file contains logic that adds extra button on page without changing source code of the built-in job details page.

```csharp
protected override void OnInit(EventArgs e)
{
    base.OnInit(e);

    var navigateButton = new Button
                             {
                                     Text = "Scheduled job overview",
                                     ToolTip = "Navigate to scheduled job overview page.",
                                     CssClass = "epi-cmsButton-tools epi-cmsButton-text epi-cmsButton-Report"
                             };

    navigateButton.Click += NavigateButtonOnClick;

    var span = new HtmlGenericControl("span");
    span.Attributes.Add("class", "epi-cmsButton");
    span.Attributes.Add("style", "float: right;");
    span.Controls.Add(navigateButton);

    Control.FindControlRecursively("GeneralSettings").Controls.Add(span);
}
```

### Execute Scheduled job manually

To provide “Execute manually” button on overview page we have to deal with the case when plugin may not be ran before – therefore it’s not yet registered as scheduled job but is just discovered by plugin subsystem (the same case when retrieving list of jobs).

This is client-side code that is invoked when user presses the button:

```js
$scope.executeJob = function (id) {
    $window.location.href = serviceUrl + '/Execute/?jobId=' + id;
};
```

Which just executes “Execute” action on controller passing in job id.

This is server side code that is invoked when script executes the action.


```csharp
var repository = new JobRepository();
var job = repository.GetList().FirstOrDefault(j => j.Id == int.Parse(jobId));

if (job != null)
{
    try
    {
        ScheduledJob jobInstance;
        if (job.InstanceId != Guid.Empty)
        {
            jobInstance = ScheduledJob.Load(job.InstanceId);
        }
        else
        {
            jobInstance = new ScheduledJob
                              {
                                      IntervalType = ScheduledIntervalType.Days,
                                      IsEnabled = false,
                                      Name = job.Name,
                                      MethodName = "Execute",
                                      TypeName = job.TypeName,
                                      AssemblyName = job.AssemblyName,
                                      IsStaticMethod = true
                              };

            if (jobInstance.NextExecution == DateTime.MinValue)
            {
                jobInstance.NextExecution = DateTime.Today;
            }

            jobInstance.Save();
        }

        if (jobInstance != null)
        {
            jobInstance.ExecuteManually();
        }
    }
    catch
    {
    }
}
```

1.First thing you need to verify that passed in job Id exists and by that id you are able to load the job instance.
2.If instance Guid is empty that means that job has not been ran before and you need to create job instance manually, pass all necessary data and then save it.
3.Then either job was loaded by `Id` or just recently created you can call `.ExecuteManually()` method.

## Single-file deployment

I’m a big fan of single file deployment especially when it comes to plugins and various other extensions. In this case plugin is deployed as single file only when its installed as NuGet package. AddOn is running normally and all necessary dependencies are deployed as physical files in AddOn installation folder.

1.*Embed resources*. In order to enable single file deployment everything your plugin depends on must be embedded in assembly itself and delivered via registered handler.

![](/assets/img/2013/10/6.png)

2.*Include resource in page*. I created small helper method in order to include scripts on page taking into account that the same code base version may run in different scenarios (different CMS version, AddOn installation mode, etc).

```js
<%= Page.ClientScript.GetJavascriptIncludes("site.js") %>
```

```csharp
public static string GetJavascriptIncludes(this ClientScriptManager clientScript,
                                           string file)
{
    return string.Format("",
                         RuntimeInfo.IsModule()
                                 ? Paths.ToClientResource(Const.ModuleName + ".AddOn", "Scripts/" + file)
                                 : clientScript.GetWebResourceUrl(typeof(ClientScriptManagerExtensions),
                                                                  ResourceProvider.CreateResourceUrl("Scripts", file)));
}
```

3.*Look-up resource*. If plugin is running in AddOn installation mode we are using `EPiServer.Shell.Paths` class to resolve path to resource, otherwise if plugin was installed via NuGet – we are in single file deployment case and need to look-up resource in assembly’s embedded resources by using `GetWebResourceUrl()` which results in something similar:

4.*Different kind of resources*. The same applies for other type of resources (images for instance):

```html
alt="{{job.IsRunning}}" data-ng-show="job.IsRunning" />
```

```csharp
public static string GetImageIncludes(this ClientScriptManager clientScript, string file)
{
    return RuntimeInfo.IsModule()
                   ? Paths.ToClientResource(Const.ModuleName + ".AddOn", "Images/" + file)
                   : clientScript.GetWebResourceUrl(typeof(ClientScriptManagerExtensions), ResourceProvider.CreateResourceUrl("Images", file));
}
```

5.*Register resource provider*. In order to render view back to the user in single file deployment case – view markup file itself is embedded into assembly’s resources and has to be extracted and sent back to the user. For this reason we need to register custom resource provider that will understand those requests and will look-up resource in embedded resource dictionary and will serve the file as it was read from the disk or any other storage.

```csharp
[ModuleDependency(typeof(InitializationModule))]
[InitializableModule]
public class InitializeModule : IInitializableModule
{
    public void Preload(string[] parameters)
    {
    }

    public void Initialize(InitializationEngine context)
    {
        if (RuntimeInfo.IsModule())
        {
            return;
        }

        GenericHostingEnvironment.Instance.RegisterVirtualPathProvider(new ResourceProvider());
        ViewEngines.Engines.Add(new CustomViewEngine());
    }

    public void Uninitialize(InitializationEngine context)
    {
    }
}
```

There is a new `Virtual Path Provider` registered in the system – `ResourceProvider`.

These are steps resource provider is responsible for:

1. It caches all embedded assembly resources just for sake of performance.
2. Translates incoming requested file name to resource name (`/Dir1/SubDir2/File1.txt` to `Dir1.SubDir2.File1.txt`).
3. Checks if requested file is in the list of the cached resource files.
4. If file exists new instance of `ResourceVirtualFile` is returned.

```csharp
public class ResourceProvider : VirtualPathProvider
{
    private static readonly List resources = typeof(ResourceProvider).Assembly.GetManifestResourceNames().ToList();

    public override bool FileExists(string virtualPath)
    {
        return ShouldHandle(virtualPath) || base.FileExists(virtualPath);
    }

    public override CacheDependency GetCacheDependency(string virtualPath, IEnumerable virtualPathDependencies, DateTime utcStart)
    {
        return ShouldHandle(virtualPath)
                       ? new CacheDependency((string)null)
                       : base.GetCacheDependency(virtualPath, virtualPathDependencies, utcStart);
    }

    public override VirtualFile GetFile(string virtualPath)
    {
        return ShouldHandle(virtualPath)
                       ? new ResourceVirtualFile(virtualPath)
                       : base.GetFile(virtualPath);
    }

    public static string CreateResourceUrl(string type, string resource)
    {
        return string.Format("{0}.{1}.{2}", Const.ModuleName, type, TranslateToResource(resource));
    }

    public static string TranslateToResource(string url)
    {
        if (url.Contains(Const.ModuleName))
        {
            url = url.Substring(url.IndexOf(Const.ModuleName, StringComparison.Ordinal));
        }

        if (url.StartsWith("/"))
        {
            url = url.Substring(1);
        }

        return url.Replace('/', '.');
    }

    private static bool ShouldHandle(string virtualPath)
    {
        return VirtualPathUtility.ToAppRelative(virtualPath).Contains(Const.ModuleName)
               && resources.Contains(TranslateToResource(virtualPath));
    }
}

internal class ResourceVirtualFile : VirtualFile
{
    private readonly string fileName;

    public ResourceVirtualFile(string virtualPath) : base(virtualPath)
    {
        this.fileName = VirtualPathUtility.ToAppRelative(virtualPath);
    }

    public override Stream Open()
    {
        return GetType().Assembly.GetManifestResourceStream(ResourceProvider.TranslateToResource(this.fileName));
    }
}
```

## Summary

It’s pretty interesting to maintain single code-base to support various target platforms. I ran into few of the cases where you need to explicitly state which platform or installation / deployment mode you are supporting now and handle various cases a bit differently. In general to provide and share the same code-base for different platforms is doable and not a huge deal.

If you would like to support single file deployment – there are special steps you need to consider and take to enable assembly embedded resource delivery via new virtual path provider and resource virtual file classes.

Would be also great if EPiServer provide NuGet feed or something similar mechanism for delivering AddOn modules for any developers than developers.

Hope this helps!

[*eof*]
