---
title: How Risky are EPiServer.DeveloperTools on Production Environment?
author: valdis
date: 2019-02-14 10:00:00 +0200
categories: [Add-On, Optimizely, Episerver, .NET, C#, Open Source, DeveloperTools]
tags: [add-on, optimizely, episerver, .net, c#, open source]
---

We had a great conversation at the EPiServer Partner Close-Up conference with one and only [Allan Thræn](https://twitter.com/athraen) about future plans and such. And one of the topics we chatted about was that in our opinion no one really knows whether ["EPiServer.DeveloperTools"](https://nuget.episerver.com/package/?id=EPiServer.DeveloperTools) package is safe to install to production environment or not. Will it do any harm? It's great for local development and probably staging also. But what about production? What exactly the package does, what it intercepts, mimics or changes in the running code?!
Therefore I thought - it's good timing to shed some light on the package and look inside - what exactly it is.

![](/assets/img/2019/02/nuget.png)

## Features of DeveloperTools Package
This is most probably the most descriptive list known about features that package offers:

* **IoC Viewer** - this feature should help you to answer questions what kind of service or interface registration you do have in your StructureMap's (or any other supported dependency injection) container. Handy time to time;
* **Content Type Analyzer** - don't you agree, that when weird stuff happening around content types and code -> database synchronization, reviewing content type definition in admin ui one-by-one is not super exciting task. Table view of the content types and sync status is great addition;
* **Loaded Assemblies** - have you wondered - what exactly website is loading and which versions of the 3rd party dependencies we have? If so, then loaded assemblies will give you list of almost all files you could potentially find in `bin/` directory;
* **Local Object Cache Manager** - sometimes it's cool to see who is using your memory and also be able to throw somebody out of shared space. this feature will give you these powers;
* **LogViewer** - watch your logs in almost real-time. This is pretty awesome and **dangerous** at the same time. "DON'T do it at production!" is stated as disclaimer, it **will** slow your site down. But if there is no other option and site is dead anyways - this is useful;
* **Memory Dump** - if you like nerd-like hard-core and love [WinDbg](https://www.microsoft.com/en-us/p/windbg-preview/9pgjgd53tn86), this command is just for you!
* **Modules Dependency Graph** - configurable and initializable modules are great addition to your EPiServer website, but it's not clear which module needs what? There is a chance you might find the answer using this tool;
* **Remote Event** - statistics about sent and received events. Practical when you are dealing with "remote server out of sync" cases;
* **Revert to Default** - this is going to be [merged together](https://github.com/episerver/DeveloperTools/issues/16) with "Content Type Analyzer", at some point :troll:
* **Routes** - tool to test your skills in "guess the route handler for incoming request" game;
* **Startup Performance** - if you are all heads down into the performance of your site and wonder why my EPiServer startup takes 5 minutes - here is a list of all the modules that's are invoked during startup and actual timing for each of them;
* **Templates** - do you know which template renderer is responsible for rendering your content? If not - come here and check;
* **View Engines/Locations** - "Content type could not be rendered. View with name X is not found in these locations: ..."? Sounds familiar? Check your view engine registrations and view locations here.

All of these features are available under "Developer" menu.

![2019-02-13_23-28-45](/assets/img/2019/02/2019-02-13_23-28-45.png)


### IoC Viewer
EPiServer dependency injection techniques initially were based on hard reference to StructureMap library. Can't recall precise EPiServer version, but at some point StructureMap was made more like an optional dependency allowed other IoC libraries to step in and provide dependency injection feature.
However, sometimes it's very important to understand and check what kind of registrations (even more important is possibility to check life of the dependency). Using this feature from DeveloperTools - it's possible to see output from `container.WhatDoIHave()` and then use searchable table to find stuff you were looking for.

![ioc](/assets/img/2019/02/ioc.png)

The only hacky trick DeveloperTools package uses - the way how it gets to the StructureMap container (not EPiServer's `ServiceCollection` container, but StructureMap one).
It was found that reference to the StructureMap container holds `StructureMapConfiguration` which available as `Services` private property of `InitializationEngine` which however is stored as private field on `InitializationModule` module (it's `IHttpModule`). You can read more about whole chain and EPiServer's initialization process [here](/2018/12/01/episerver-init-infrastructure-under-the-hood/). Code is dirty, but it works:

```csharp
var ie = (InitializationEngine) typeof(InitializationModule).GetField("_engine", BindingFlags.NonPublic | BindingFlags.Static)
                                                            .GetValue(null);
var services = ie.GetType()
                 .GetProperty("Services", BindingFlags.NonPublic | BindingFlags.Instance)
                 .GetValue(ie, null) as StructureMapConfiguration;
var container = services.Container;
...
```

### Content Type Analyzer
Have you had an issue when you defined content types are just not synced with EPiServer and you are pulling your hairs out of your head to understand why? What is wrong, which property(-ies) are not synced and why this is happening? If yes, this feature of DeveloperTools might be handy for you.

![content-type-analyzer](/assets/img/2019/02/content-type-analyzer.png)

It's basically a searchable table of all content types and properties with sync status. Feature just uses data available from `ContentTypeModelRepository`, no fancy hacky way to get to the content type info.
If you will have conflict during sync operation - it will be nicely shown there as well.

![content-type-analyzer--error](/assets/img/2019/02/content-type-analyzer--error.png)

Thanks [Māri for the code](http://marisks.net/2018/05/14/finding-content-type-conflict-reasons/).

### Loaded Assemblies
Sometimes things might get messy and weird and it could turn out that some unwanted assemblies are loaded that shouldn't be there. "Loaded Assemblies" feature enlists everyone that's found in `AppDomain` using `AppDomain.CurrentDomain.GetAssemblies()`. Location of the assembly is nice addition as well.

![loaded-asm](/assets/img/2019/02/loaded-asm.png)

Extra bonus - you are also able to read all discovered `EnvironmentVariables` and a sneak peak into couple of properties for the current `HttpRequest`.

### Local Object Cache Manager

Thanks to Joe's contributions local object cache viewer is now part of EPiServer DeveloperTools package. During the contribution we added small extra to the tool - you can now also see approx. size of the cache entry (might be sometimes useful).

![cache-viewer](/assets/img/2019/03/cache-viewer.png)

This feature will get some UX polishing, but other than that - it's works great and gives you possibility to throw something out even from "remote" cache as well.

### LogViewer
Do the action in the site and see EPiServer's log entries on fly. No need to remote session into server (sometimes it's not even possible).

Trick here is to add `InMemoryAppender` for log4net. Yes, DeveloperTools package use direct reference to log4net at the moment. Why not? :) If you do other logging mechanism - this will not work.

![logging](/assets/img/2019/02/logging.png)

This small code snipper basically ensures that in-memory rolling appender is added to the log4net list so then it will be possible to "intercept" and output those log entries on the page:

```csharp
private void CreateDefaultMemoryAppender()
{
    _memoryAppender = new RollingMemoryAppender { Name = "DeveloperToolsLogViewer" };
    _memoryAppender.ActivateOptions();
    var repository = LogManager.GetRepository() as Hierarchy;

    if(repository != null)
    {
        repository.Root.AddAppender(_memoryAppender);
        repository.Root.Level = Level.All;
        repository.Configured = true;
        repository.RaiseConfigurationChanged(EventArgs.Empty);
    }
}
```

### Memory Dump
I hope everyone loves [WinDbg](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/) or similar hardcore!

If you encounter an error on production (let's say memory leak) and the only way to diagnose it is by analyzing memory dump, one of the easy way is to use this tool to get things done.

![mem-dump](/assets/img/2019/02/mem-dump.png)

It uses couple of Win32 imports to get to the some internals, but that's not a big deal.

```csharp
public class NativeMethods
{
    [DllImport("dbghelp.dll",
         EntryPoint = "MiniDumpWriteDump",
         CallingConvention = CallingConvention.StdCall,
         CharSet = CharSet.Unicode,
         ExactSpelling = true,
         SetLastError = true)]
    public static extern bool MiniDumpWriteDump(IntPtr hProcess,
                                                uint processId,
                                                IntPtr hFile,
                                                uint dumpType,
                                                ref MiniDumpExceptionInformation expParam,
                                                IntPtr userStreamParam,
                                                IntPtr callbackParam);

    [DllImport("kernel32.dll", EntryPoint = "GetCurrentThreadId", ExactSpelling = true)]
    public static extern uint GetCurrentThreadId();

    [DllImport("kernel32.dll", EntryPoint = "GetCurrentProcess", ExactSpelling = true)]
    public static extern IntPtr GetCurrentProcess();
}

[StructLayout(LayoutKind.Sequential, Pack = 4)]
public struct MiniDumpExceptionInformation
{
    public uint ThreadId;
    public IntPtr ExceptionPointers;
    [MarshalAs(UnmanagedType.Bool)] public bool ClientPointers;
}

[Flags]
public enum DumpType : uint
{
    MiniDumpNormal = 0x00000000,
    MiniDumpWithDataSegs = 0x00000001,
    MiniDumpWithFullMemory = 0x00000002,
    MiniDumpWithHandleData = 0x00000004,
    MiniDumpFilterMemory = 0x00000008,
    MiniDumpScanMemory = 0x00000010,
    MiniDumpWithUnloadedModules = 0x00000020,
    MiniDumpWithIndirectlyReferencedMemory = 0x00000040,
    MiniDumpFilterModulePaths = 0x00000080,
    MiniDumpWithProcessThreadData = 0x00000100,
    MiniDumpWithPrivateReadWriteMemory = 0x00000200,
    MiniDumpWithoutOptionalData = 0x00000400,
    MiniDumpWithFullMemoryInfo = 0x00000800,
    MiniDumpWithThreadInfo = 0x00001000,
    MiniDumpWithCodeSegs = 0x00002000,
    MiniDumpWithoutAuxiliaryState = 0x00004000,
    MiniDumpWithFullAuxiliaryState = 0x00008000,
    MiniDumpWithPrivateWriteCopyMemory = 0x00010000,
    MiniDumpIgnoreInaccessibleMemory = 0x00020000,
    MiniDumpValidTypeFlags = 0x0003ffff
}

public sealed class MiniDump
{
    public static void WriteDump(string fileName, DumpType typeOfdumpType)
    {
        MiniDumpExceptionInformation info;
        info.ThreadId = NativeMethods.GetCurrentThreadId();
        info.ClientPointers = false;
        info.ExceptionPointers = Marshal.GetExceptionPointers();

        using (var fs = new FileStream(fileName, FileMode.Create, FileAccess.Write, FileShare.None))
        {
            var processId = (uint) Process.GetCurrentProcess().Id;
            var processHandle = Process.GetCurrentProcess().Handle;
            var dumpType = (uint) typeOfdumpType;
            NativeMethods.MiniDumpWriteDump(processHandle2,
                                            processId,
                                            fs.SafeFileHandle.DangerousGetHandle(),
                                            dumpType,
                                            ref info,
                                            IntPtr.Zero,
                                            IntPtr.Zero);
        }
    }
}
```

But be aware, that while dump is being made, your site is basically "frozen". All the threads are stopped, nothing happens. If memory usage is huge (as it's usually the case for the leaks) - site might be unresponsive for longer period of time. But who really cares then when site is just crushing all the time when memory limit is reached?

Yes, and also I do agree that EPiServer DeveloperTools could deliver this file directly back to the browser as downloadable content.. Experience would be very similar to the one you could experience using Kudu console in Azure.

![kudu](/assets/img/2019/02/kudu.png)

### Modules Dependency Graph
This is more like nice-to-look-at feature. No huge interaction or features there, but sometimes it's nice to see your configurable or initializable module dependencies and see who is the man in the center.

![modules-dep](/assets/img/2019/02/modules-dep.png)

You can either filter on all modules (including EPiServer's ones as well) or just look at your own custom defined ones.

![mod-dep-all](/assets/img/2019/02/mod-dep-all.png)

With all the EPiServer's modules included - things are getting hairy there.

### Remote Event
Part of the DeveloperTools is also feature to deal with remote events. It's very convenient to cross-check health and statistics of particular events when needed. This is important when you are debugging remoting related case.

![remote-events](/assets/img/2019/02/remote-events.png)

You can also invoke (send) particular event - kinda mimic the server behavior.
Feature uses `IEventRegistry`, `EventProviderService` and `ServerStateService` services.

### Revert to Default
This particular feature is going to be merged together with "Content Type Analyzer" as those two are pretty closely related.

This part of the library invokes following code for selected content type(-s):

```csharp
var ct = contentTypeRepository.Load(id);
var writableContentType = ct.CreateWritableClone() as ContentType;
writableContentType.ResetContentType();

foreach (var propDef in writableContentType.PropertyDefinitions)
{
    propeDef.ResetPropertyDefinition();
    _propertyDefinitionRepository.Save(propeDef);
}
```

![revert](/assets/img/2019/02/revert.png)

Uses `IContentTypeRepository` & `IPropertyDefinitionRepository` services.
Becomes handy when you messed around with your content types.

### Routes
Do you know all the routes that are registered in `RouteTable` and through which Asp.Net Mvc is trying to lookup handler for your incoming request? Do you know why your controller is not invoked? Or do you know why some of the parameters are recognizable? This feature will be your 1st tool to understand what's going on in your routes.

![routes](/assets/img/2019/02/routes.png)

Feature also gives you overview of all defaults for the route definition. Nice!

### Startup Performance
Why your startup is slow? Here you might find answers. This feature uses pretty cool type that I was not aware before: `TimeMeters.GetAllRegistered();`. It gives you timing registries that were collected during startup (initialization engine setup and invoke of all discovered modules).

![perf](/assets/img/2019/02/perf.png)

You can see who is the winner for this instance of the app.

### Templates
Very helpful information that you can get out of `ITemplateRepository` and `ContentTypeModelRepository` services. It may answer question about why my content type is rendered as it's rendered? Why some of the blocks look differently? And how my content will look like when I will use tags?

![templates](/assets/img/2019/02/templates.png)

Sometimes it might get really hot to understand all the mechanics behind the process which is responsible for deciding which of the template will be used for rendering of particular content.

### View Engines/Locations
Last but not least, this feature gives you an overview of all registered view locations where `ViewEngine`-s might look at. This had been convenient couple times when you are looking why block is not rendered, why view is not picked-up and why Asp.Net Mvc is using totally wrong view to render the content.

![views](/assets/img/2019/02/views.png)

This becomes practical together with "Templates" section.

## Is It Risky?

**"USE AT YOUR OWN RISK!"** This is basically disclaimer of the tools package. EPiServer or any other party do not take any responsibility for damage made to the websites when package is installed. With great power comes great responsibility! EPiServer initially didn't want to release this as NuGet package on the feed. But I saw opportunity to pack it up and distribute for other fellow developers as well. This makes life a bit easier for packaging, updates and fixing the bugs.

There is no configurable or initializable modules in the package that might alter the running code. Package provides functionality based on existing features or hooks inside EPiServer platform. However, there are couple features that are risky to enable or use. So you should be careful with those.

Worth to note that all of this is availably only for users with "AdminAccess" permissions. In my opinion all the these features are more or less read-only and could not do big harm except "Revert to Default" & "Memory Dump" features. User passwords and any other secrets might just be exposed during memory dump analysis. So be aware.

Hope you will not get into the situation when you will need DeveloperTools on production server.

Happy debugging!
[*eof*]
