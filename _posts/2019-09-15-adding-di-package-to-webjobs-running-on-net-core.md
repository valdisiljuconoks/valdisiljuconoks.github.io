---
title: Converting Azure WebJobs to .NET Core
author: valdis
date: 2019-09-15 07:00:00 +0200
categories: [.NET, C#, Azure, WebJobs]
tags: [.net, c#, grpc, azure, webjobs]
---

## Motivation

Migrating something to .NET Core (while stuck with .NET Framework for a while due to surrounding platform dependency constraints) sounds intriguing and challenging at the same time. Our main motivator for the migration has been [performance improvements](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-core/), [performance improvements](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-core-2-1/) and upcoming [performance improvements](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-core-3-0/) seen throughout .NET Core.

This blog post will walk you through steps we did for migration for one our [web jobs](https://docs.microsoft.com/en-us/azure/app-service/webjobs-create) over to .NET Core.

As seen from pull request statistics - it's actually more removal that adding new code. Throwaway always feels good.

![2019-07-17_20-53-51](/assets/img/2019/09/2019-07-17_20-53-51.png)

Path wasn't smooth, a bit bumpy - but at the end we reached our destination.

## Background

We are building near real-time tracking system for public transportation company. WebJob (still) has one of central role in this system. It does great work by sticking together data from various sources and composing unified data model for later consumption. One of the reason (partially nowadays just historical) we need WebJob in our infrastructure is because running instance needs access to in-memory cache (job running currently requires access to results composed by previous run). Yes - we are storing data in persistent storage. Yes - we know about some "more friendly / managed cache services". But the fastest way to access results produced by previous run is required. Using in-memory cache showed great results for performance. Hosting options might change of course overtime but for now - WebJob is fair enough for us.

Let's get started.

## Planning

Before you actually jump to the action and perform real migration tasks, it's good idea to cross-check project source code and dependencies readiness for migration to .NET Core or .NET Standard. Which target to choose depends on how library is going to be used. As far as I understood:
* choose .NET Core is project is going to be "host". Some executable - either console application or ASP.NET Core web application;
* for the rest - pick .NET Standard (if used APIs are supported there)

Great tool for this task is called "Portability Analyzer." You can grab it from [Visual Studio Marketplace](https://marketplace.visualstudio.com/items?itemName=ConnieYau.NETPortabilityAnalyzer).

![portability-analyzer](/assets/img/2019/09/portability-analyzer.png)

Idea for the tool is to get you prepared before acting. It checks what code is used in project  or solution, also dependencies are checked. Compatibility level for code-base to be migrated to .NET Core or .NET Standard is shown as result report. Tool outputs results in Excel which makes it easy for filtering and reviewing.

## Transition to .NET Core

### Convert WebJob Project File

Converting from .NET Framework to .NET Core project file is quite dramatic (in a good way). Zillions of code lines are removed from `.csproj` file and only essentials are left.

![Annotation-2019-08-13-233551](/assets/img/2019/09/Annotation-2019-08-13-233551.png)

Before converting, I recommend to migrate from NuGet reference (old-school package references) to "Package Reference" format (even while you are still in .NET Framework). There is a built-in tool inside Visual Studio that can help you with this.

![migrate-to-pack-ref](/assets/img/2019/09/migrate-to-pack-ref.png)

It can be done manually as well of course, but with lots of references - it might get boring at some point.
Tool helps top convert from this:

```xml
<ItemGroup>
    <Reference Include="Newtonsoft.Json, Version=10.0.0.0, Culture=neutral, PublicKeyToken=30ad4fe6b2a6aeed, processorArchitecture=MSIL">
      <HintPath>..\..\packages\Newtonsoft.Json.10.0.3\lib\net45\Newtonsoft.Json.dll</HintPath>
    </Reference>
</ItemGroup>
```

to this:

```xml
<ItemGroup>
    <PackageReference Include="Newtonsoft.Json" version="10.0.3" />
</ItemGroup>
```

As you can see - we now only need version of the package. You can read more about SDK format type [here](https://docs.microsoft.com/en-us/dotnet/core/migration/).
At the end `.csproj` file is much cleaner and have just important parts to get project running.

![2019-08-13_23-14-11](/assets/img/2019/09/2019-08-13_23-14-11.png)

Next step is to upgrade WebJob infrastructure packages to get it running under .NET Core. Following dependencies are required for WebJob to run on .NET Core (at the moment of writing):

* Microsoft.Azure.WebJobs (v3.x)
* Microsoft.Azure.WebJobs.Extensions (v3.x)
* Microsoft.Azure.WebJobs.Extensions.Storage (v3.x)
* Microsoft.Extensions.DependencyInjection (v2.2.x)
* Microsoft.Extensions.Options.ConfigurationExtensions (v2.2.x)

Optional references:

* Microsoft.Extensions.Configuration.CommandLine (v2.2.x)
* Microsoft.Extensions.Logging.Console (v2.2.x)

### Leaving Dependencies in .NET Framework

In cases when you can't convert dependency to .NET Standard or .NET Core, we extracted sharable pieces into separate project and converted that to .NET Standard and leaving rest of the code in .NET Framework. By doing this we could ensure that code is reused as much as possible (like data structures, model definitions, anything that can be converted to .NET Standard, etc.) and framework specific thingies are left in old project. Results is architecture where part of the system is running on .NET Framework (by referencing .NET Standard targeted shared project) and other part of the system is capable of running on .NET Core (also using the same shared components targeting .NET Standard).

In our case - we had WCF service client reading data from remote API endpoint, processing data and then writing it to the Azure Storage.

![fx-netstd-core-1-3](/assets/img/2019/09/fx-netstd-core-1-3.png)

Later data was read from Azure Storage and processed dureing composition process. However - the same WCF service client library was used to read data from Azure Storage.

![fx-netstd-core-2](/assets/img/2019/09/fx-netstd-core-2.png)

As we realized WCF client API library is not quite movable to .NET Core or .NET Standard target, we have extract some of the shared parts and reuse that in composition process (which will be running on .NET Core).

![fx-netstd-core-3](/assets/img/2019/09/fx-netstd-core-3.png)

Azure Storage reader code fragment was taken out and moved .NET Standard. This extraction also made it possible to read data from other .NET Standard or .NET Core projects in our system.

### Shared Library and Transient Dependencies

In our structure there are some applications still running on .NET Framework. In .NET Framework project it is possible to reference .NET Standard targeted project. You just have to be aware that there are some issues for .NET Framework based project to "collect" transient dependencies coming from .NET Standard project(s) and copy those in output directory.
Meaning that if you have structure `A (netfx) -> B (netstandard) -> C (netstandard)` there are high chances that project `C (netstandard)` which is transient dependency to project `A (netfx)` will not be copied to output directory of `A (netfx)`. This will result in runtime errors blaming that required types could not be loaded.
Workaround for this is to just reference project `C (netstandard)` directly from project `A (netfx)` => `A (netfx) -> C (netstandard)`. Cumbersome, but this works!

### Converting Program.cs

WebJob is just ordinary console application that is running in Azure WebJobs host.
Looking at old WebJob `Program.cs` file you can see that there is not so much configuration (for example - you can't really see how logging is done):

```csharp
class Program
{
    static void Main()
    {
        var container = ComposeContainer();
        var config = new JobHostConfiguration();

        if(ScheduledTimeTableConfig.IsDevelopment)
            config.UseDevelopmentSettings();

        config.UseTimers();

        var host = new JobHost(config);
        host.RunAndBlock();
    }
}
```

WebJobs based .NET Framework have lots of configuration located in `App.config` file.
New .NET Core WebJobs `Program.cs` file is just yet another .NET Core application with all the required configuration in code (which is very nice). It helps to understand what is being used by looking at the code:

```csharp
internal class Program
{
    private static async Task Main(string[] args)
    {
        var builder = new HostBuilder();
        builder.ConfigureWebJobs(b =>
                {
                    b.AddAzureStorageCoreServices();
                    b.AddAzureStorage();
                    b.AddTimers();
                })
               .ConfigureAppConfiguration(b =>
                {
                    // Adding command line as additional configuration source
                    b.AddCommandLine(args);
                })
               .ConfigureLogging((context, b) =>
                {
                    // here we can access context.HostingEnvironment.IsDevelopment() yet
                    if(context.Configuration["environment"] == EnvironmentName.Development)
                    {
                        b.SetMinimumLevel(LogLevel.Debug);
                        b.AddConsole();
                    }
                    else
                    {
                        b.SetMinimumLevel(LogLevel.Information);
                    }

                    // configure CommonLogging to use Serilog
                    var logConfig = new LogConfiguration();
                    context.Configuration.GetSection("LogConfiguration").Bind(logConfig);
                    LogManager.Configure(logConfig);

                    var log = new LoggerConfiguration()
                                      .WriteTo
                                      .File("webjob-log.txt", rollingInterval: RollingInterval.Day)
                                      .CreateLogger();
                    Log.Logger = log;
                })
               .ConfigureServices((context, services) =>
                {
                    services.AddSingleton(context.Configuration);
                    services.AddMemoryCache();

                    // other DI configuration here
                })
               .UseConsoleLifetime();

        var host = builder.Build();
        Services = host.Services;

        using(host)
        {
            await host.RunAsync();
        }
    }
}
```

Following happens in this code fragment:

* creating new instance of generic host builder (`HostBuilder`)
* as you can see we are not explicitly setting configuration (loading `.json` file or using environment variables) - this is done by default already by WebJob configuration (`.ConfigureWebJobs()`). Source for this is [available here](https://github.com/Azure/azure-webjobs-sdk/blob/137196291b5edd421e34dbd283486b4640a9334b/src/Microsoft.Azure.WebJobs.Host/Hosting/WebJobsHostBuilderExtensions.cs#L36). Only thing we need to configure is to add command-line argument support. This comes handy when you are debugging and need to pass-in various settings on-fly.
* next need to add logging support. Historically our platform is based on `log4net` library to provide logging capabilities for our systems. Over the time we have been able to move away to some more generic abstractions like `Common.Logging` in order to remove direct dependency on `log4net` library. However this time decided that we have opportunity to remove this dependency at all. We researched couple other logging platforms and decided to go with [Serilog](https://serilog.net/).
* next comes DependencyInjection -> here we are adding our services and its lifetime configuration to service collection.
* then we need to tell host we are in console mode (it adds `ConsoleLifetime` configuration as `IHostLifetime` implementation). This lifetime manager ensures that application is alive and going to die only by receiving `Ctrl+C` keystroke or `SIGTERM` and only then initiates shutdown.
* then we are building host itself (`builder.Build()`)
* capturing the services from host (`IServiceCollection`). This will be required later for job timer functionality. More details below.
* then we need to build the host and launch it

### Adding DI Support

We've been using StructureMap dependency injection library for ages, and built practices around it. Team used to APIs and behavior of the StructureMap. Therefore we thought it makes sense to continue to use it and give Microsoft.Extensions.DependencyInjection (aka "MSDI") package another chance in some different project.

To make StructureMap working under .NET Core infrastructure we will need StructureMap adapter for Microsoft dependency injection framework.

This can be added by following package:
```
<PackageReference Include="StructureMap.Microsoft.DependencyInjection" Version="2.0.0" />
```

We can configure and setup StructureMap container and tell Microsoft DI infrastructure about services that are configured in StructureMap to be used.

```csharp
internal class Program
{
    private static async Task Main(string[] args)
    {
        var builder = new HostBuilder();
        builder.ConfigureWebJobs(...)
               .UseServiceProviderFactory(
                   new StructureMapServiceProviderFactory(ComposeContainer))
               ...
    }

    private static Container ComposeContainer()
    {
        var container = new Container(_ =>
        {
            // here goes code to configure container..
            // for example:
            //
            // _.For<ISomeInterface>().Use<SomeServiceImpl>().Singleton();
        });

        return container;
    }
}
```

What's left is to implement builder for `IServiceProvider` implementation. For this we will need access to configured StructureMap container.

```csharp
internal class StructureMapServiceProviderFactory : IServiceProviderFactory<IContainer>
{
    private readonly Func< IContainer> _containerBuilder;

    public StructureMapServiceProviderFactory(Func<IContainer> containerBuilder)
    {
        _containerBuilder = containerBuilder;
    }

    public IContainer CreateBuilder(IServiceCollection services)
    {
        var container = _containerBuilder();
        container.Populate(services);

        return container;
    }

    public IServiceProvider CreateServiceProvider(IContainer containerBuilder)
        => containerBuilder.GetInstance<IServiceProvider>();
}
```

Black magic to "glue" MSDI world with StructureMap one is in this line `container.Populate(services);`. During this `Populate()` method specific service provider and scope factory are added:

```csharp
public static void Populate(this Registry registry,
                            IEnumerable<ServiceDescriptor> descriptors,
                            bool checkDuplicateCalls)
{
    registry.For<IServiceProvider>()
            .LifecycleIs(Lifecycles.Container)
            .Use<StructureMapServiceProvider>();

    registry.For<IServiceScopeFactory>()
            .LifecycleIs(Lifecycles.Container)
            .Use<StructureMapServiceScopeFactory>();
}
```

This is what happens when you look under the hood for these lines:

```csharp
builder.UseServiceProviderFactory();  // .UseSPF() in diagram
..
builder.Build();
```

![fx-netstd-core-4](/assets/img/2019/09/fx-netstd-core-4.png)

Now every time WebJob infrastructure will require service provider - it will be built from StructureMap container. This is nice addition to be able to use StructureMap advanced type lookup or scanning features instead of relying on explicit service registration in MSDI case.


### IConfiguration Access in IServiceProvider Factory

Sometimes you might need access to `IConfiguration` interface to fetch settings to conditionally configure container.
I haven't figure out better approach to do this, so sharing what I've got here.
First, we need to change signature of `ComposeContainer()` method  to accept now `IConfiguration configuration` as parameter:

```csharp
private static Container ComposeContainer(IConfiguration configuration)
{
    ...
}
```

Then we need to change signature of factory constructor:

```csharp
public StructureMapServiceProviderFactory(
    Func<IConfiguration, IContainer> containerBuilder)
{
    ....
}
```

And now while we create container instance we need to get access to `IConfiguration` implementation and pass that into the function invocation.

```csharp
public IContainer CreateBuilder(IServiceCollection services)
{
    // temporary build service provider
    // to get access to IConfiguration implementation
    var sp = services.BuildServiceProvider();
    var container = _containerBuilder(sp.GetService<IConfiguration>());
    container.Populate(services);

    return container;
}
```

I'm not quite sure that this is the best approach to get instance of `IConfiguration` implementation is whether this is good idea in general to call `services.BuildServiceProvider()`.

Later by having access to `IConfiguration` instance, container setup logic can conditionally add some services to the container or configure instance specific settings from configuration.

```csharp
private static Container ComposeContainer(IConfiguration configuration)
{
    var config = configuration.GetSection("ConfigSection").Get<SomeConfig>();
    var container = new Container(_ => { ... });

    if(config.IsSomethingEnabled)
        container.Configure(...);

    return container;
}
```

### StructureMap Constructor Selector Policy

Once StructureMap is properly configured and ready to roll, you have to keep in mind that StructureMap's MSDI adapter default constructor selector policy looks for "the most specific constructor". Source code can be [found here](https://github.com/structuremap/StructureMap.Microsoft.DependencyInjection/blob/master/src/StructureMap.Microsoft.DependencyInjection/ContainerExtensions.cs#L98).

Meaning that there might be some runtime issues once you get your WebJob up & running.

For example, when you add `ConsoleLogger` to the configuration, there will be a case when somewhere deep down in rabbit hole someone will require instance of `ConsoleLoggerProvider`. Which in turn (hopping over some stacks here) will create new instance of `WebJobsOptionsFactory` class.
This class has two constructors:

```csharp
internal class WebJobsOptionsFactory<TOptions> :
    IOptionsFactory<TOptions> where TOptions : class, new()
{
    private readonly OptionsFactory<TOptions> _innerFactory;
    private readonly IOptionsLoggingSource _logSource;
    private readonly IOptionsFormatter<TOptions> _optionsFormatter;

    public WebJobsOptionsFactory(
        IEnumerable<IConfigureOptions<TOptions>> setups,
        IEnumerable<IPostConfigureOptions<TOptions>> postConfigures,
        IOptionsLoggingSource logSource) : this(setups, postConfigures, logSource, null)
    {
        ...
    }

    public WebJobsOptionsFactory(
        IEnumerable<IConfigureOptions<TOptions>> setups,
        IEnumerable<IPostConfigureOptions<TOptions>> postConfigures,
        IOptionsLoggingSource logSource,
        IOptionsFormatter<TOptions> optionsFormatter)
    {
        ...
    }
}
```

As you can see first constructor relies on second one just by providing `null` to the `IOptionsFormatter<TOptions>` implementation.
However, StructureMap does not support this kind of constructor resolution and will invoke "the most specific" constructor -> constructor with most parameters will be selected - looking for implementation of `IOptionsFormatter<TOptions>`.

Which will result in following runtime error:

```
No default Instance is registered and cannot be automatically determined for type 'IOptionsFormatter<ConsoleLoggerOptions>'

There is no configuration specified for IOptionsFormatter<ConsoleLoggerOptions>

1.) new WebJobsOptionsFactory`1(*Default of IEnumerable<IConfigureOptions<ConsoleLoggerOptions>>*, *Default of IEnumerable<IPostConfigureOptions<ConsoleLoggerOptions>>*, *Default of IOptionsLoggingSource*, *Default of IOptionsFormatter<ConsoleLoggerOptions>*)
2.) WebJobsOptionsFactory<ConsoleLoggerOptions> ('dabbde68-0a4c-4636-b291-4fb356b67a14')
3.) Instance of IOptionsFactory<ConsoleLoggerOptions> ('dabbde68-0a4c-4636-b291-4fb356b67a14')
4.) new OptionsMonitor`1(*Default of IOptionsFactory<ConsoleLoggerOptions>*, *Default of IEnumerable<IOptionsChangeTokenSource<ConsoleLoggerOptions>>*, *Default of IOptionsMonitorCache<ConsoleLoggerOptions>*)
5.) OptionsMonitor<ConsoleLoggerOptions> ('c3ab42b9-4c41-4a07-af71-3c9df514ce96')
6.) Instance of IOptionsMonitor<ConsoleLoggerOptions> ('c3ab42b9-4c41-4a07-af71-3c9df514ce96')
7.) new ConsoleLoggerProvider(*Default of IOptionsMonitor<ConsoleLoggerOptions>*)
8.) Microsoft.Extensions.Logging.Console.ConsoleLoggerProvider
9.) Instance of Microsoft.Extensions.Logging.ILoggerProvider (Microsoft.Extensions.Logging.Console.ConsoleLoggerProvider)
10.) All registered children for IEnumerable<ILoggerProvider>
11.) Instance of IEnumerable<ILoggerProvider>
12.) new LoggerFactory(*Default of IEnumerable<ILoggerProvider>*, *Default of IOptionsMonitor<LoggerFilterOptions>*)
13.) Microsoft.Extensions.Logging.LoggerFactory
14.) Instance of Microsoft.Extensions.Logging.ILoggerFactory (Microsoft.Extensions.Logging.LoggerFactory)
15.) new Logger`1(*Default of ILoggerFactory*)
16.) Logger<ApplicationLifetime> ('bf048bf3-567a-459e-9634-8b6d277e6507')
17.) Instance of ILogger<ApplicationLifetime> ('bf048bf3-567a-459e-9634-8b6d277e6507')
18.) new ApplicationLifetime(*Default of ILogger<ApplicationLifetime>*)
19.) Microsoft.Extensions.Hosting.Internal.ApplicationLifetime
20.) Instance of Microsoft.Extensions.Hosting.IApplicationLifetime (Microsoft.Extensions.Hosting.Internal.ApplicationLifetime)
21.) new Host(*Default of IServiceProvider*, *Default of IApplicationLifetime*, *Default of ILogger<Host>*, *Default of IHostLifetime*, *Default of IOptions<HostOptions>*)
22.) Microsoft.Extensions.Hosting.Internal.Host
23.) Instance of Microsoft.Extensions.Hosting.IHost (Microsoft.Extensions.Hosting.Internal.Host)
24.) Container.GetInstance(Microsoft.Extensions.Hosting.IHost)
```

The same applies for `IWebHookProvider` for example.

```
StructureMap.StructureMapConfigurationException: 'No default Instance is registered and cannot be automatically determined for type 'Microsoft.Azure.WebJobs.Host.Config.IWebHookProvider''
```

These are two dependencies that I discovered were not registered in container but required to build instance of WebJob host with console logging configured.

Workaround for this is to "silence" or fake implementations for these types. It's doable by configuring StructureMap's container:

```csharp
private static Container ComposeContainer(...)
{
    var container = new Container(_ =>
    {
        _.For(typeof(IOptionsFormatter<>)).Use(ctx => null);
        _.For(typeof(IWebHookProvider)).Use(ctx => null);

        ...
    };

    return container;
```

### Converting Custom Timers

In our WebJob solution we do have separate timer trigger for each of the jobs.

```csharp
public class SomeJob
{
    public async Task Run([TimerTrigger(typeof(SomeJobTrigger))]
                            TimerInfo timerInfo,
                            ILogger log)
    {
        // job logic goes here...
    }
}
```

Trigger itself:

```csharp
public class SomeJobTrigger : TimerSchedule
{
    private readonly TimeSpan _interval = TimeSpan.Parse("00:05:00");

    public override DateTime GetNextOccurrence(DateTime now)
    {
        // here `_config` might be any implementation
        // which is able to read config from somewhere
        // we are for now just using ConfigurationManager.AppSettings[""]
        if(!_config.SomeJobTriggerEnabled)
            return DateTime.MaxValue.AddYears(-100);

        var timeSpan = _interval;
        return now + timeSpan;
    }
}
```

It's nice feature for Azure Functions to have possibility to enable or disable individual functions via portal.

![func-on-off](/assets/img/2019/09/func-on-off.png)

We wanted something similar for our WebJobs as well. This would allow us to have possibility to enable / disable specific job without redeploying whole solution (which requires decorate job class with `[Disabled]` attribute). We can change configuration and restart job host instance without redeployments.

In the new .NET Core world we have to have access to `IServiceCollection` in order to get configuration options out of it.

Can add `IOptions<T>` instance to the service container:

```csharp
internal class Program
{
    private static async Task Main(string[] args)
    {
        var builder = new HostBuilder();
        builder.ConfigureWebJobs(...)
               .ConfigureServices((context, services) =>
                {
                    services.AddSingleton(context.Configuration);
                    services.AddMemoryCache();

                    // other DI configuration here
                    services.Configure<SomeJobConfig>(context.Configuration.GetSection(nameof(SomeJobConfig)))
                });

        // run the host
        ...
        Services = host.Services;
    }

    /// <summary>
    /// We need access to service provider later in TimedTrigger - to get data from the config file
    /// </summary>
    public static IServiceProvider Services { get; set; }
}
```

And settings file itself (`SomeJobConfig.cs`):

```csharp
public class SomeJobConfig
{
    public bool SomeJobTriggerEnabled { get; set; }
}
```

Now when we do have options configured for our WebJobs, access to it via service collection:

```csharp
public class SomeJobTrigger : TimerSchedule
{
    private readonly TimeSpan _interval = TimeSpan.Parse("00:05:00");

    public SomeJobTrigger()
    {
        _config = Program.Services.GetService<IOptions<SomeJobConfig>>().Value;
    }

    public override DateTime GetNextOccurrence(DateTime now)
    {
        // got access to configuration settings via IOptions<T>
        if(!_config.SomeJobTriggerEnabled)
            return DateTime.MaxValue.AddYears(-100);

        var timeSpan = _interval;
        return now + timeSpan;
    }
}
```

### Getting ConnectionStrings

As previous version of WebJobs (and including some common shared libraries) were on .NET Framework, we had `ConfigurationManager` usage in our code-base wherever we needed access to some configuration data. It's still possible to use `ConfigurationManager` in .NET Standard libraries via [Platform Compatibility Pack](https://devblogs.microsoft.com/dotnet/announcing-the-windows-compatibility-pack-for-net-core/), but in this case we wanted to go full native and access configuration from `IConfiguration` interface instance directly.

We had these code fragments all around the code-base:

```csharp
public class SomeDataAccessThingy
{
    public void DoMagic()
    {
        // need to get connection string first
        var connection = ConfigurationManager.ConnectionStrings[name].ConnectionString;

        // magic continues here..
    }
}
```

As you can see this is not very friendly for the migration.
We extracted access to `ConfigurationManager` API into separate class in order to isolate and locate all usages of these APIs.

```csharp
public interface IConnectionStringAccessor
{
    string GetConnectionStringMyName(string name);
}
```

Next we need to provide two implementation of this interface - one for .NET Framework applications (we still have couple of them in our infrastructure) and another one for the .NET Core platform.

```csharp
// this one will be used in .NET Framework applications
// just by injecting proper implementation for IConnectionStringAccessor interface
public class DefaultConnectionStringAccessor : IConnectionStringAccessor
{
    public string GetConnectionStringMyName(string name) => ConfigurationManager.ConnectionStrings[name].ConnectionString;
}
```

And for .NET Core:

```csharp
// this one will be used in .NET Core applications
public class NetCoreConnectionStringAccessor : IConnectionStringAccessor
{
    private readonly IConfiguration _configuration;

    public NetCoreConnectionStringAccessor(IConfiguration configuration)
    {
        _configuration = configuration;
    }

     public string GetConnectionStringMyName(string name) => _configuration.GetConnectionString(name);
}
```

Now instead of accessing `ConfigurationManager` directly - we use our accessor that abstracts away actual implementation of how to get connection strings from platform APIs - meaning that common shared libraries are now more "cross-platform" ready.

```csharp
private CloudBlobClient GetBlobClient(string connectionName)
{
    var connectionString = _connectionStringAccessor
                               .GetConnectionStringMyName(connectionName);
    var storageAccount = CloudStorageAccount.Parse(connectionString);

    return storageAccount.CreateCloudBlobClient();
}
```

### Container vs Pure DI (aka Poor Man's DI) Principle

Dependency Injection is a principle when writing your class library code you don't think about how you are going to get implementation of the interface you need, or some sort of configuration for your service to run successfully. You can just assume that someone will pass it in while constructing instance of your class. This is [constructor dependency injection](https://en.wikipedia.org/wiki/Dependency_injection#Constructor_injection).

With introduction of additional dependency (to retrieve connection strings from different sources depending on target running platform) for various services - there is now a change in constructor signature. This of course should be taken into account and solved by dependency injection framework (aka IoC).

Before:

```csharp
public class SomeServiceImpl : ISomeService
{
    public SomeServiceImpl(IDependency dep)
    {
        // .. capture passed in deps
    }

    public void DoStuff()
    {
        // ...
    }
}
```

Now:

```csharp
public class SomeServiceImpl : ISomeService
{
    public SomeServiceImpl(IDependency dep, IConnectionStringAccessor accessor)
    {
        // .. capture passed in deps
    }

    public void DoStuff()
    {
        // ...
    }
}
```

Everything compiles and you can run the app. What you will get is an error telling you that dependency injection framework can't resolve `IConnectionStringAccessor` dependency. Funny part - this is **not** compile time error. Exception is thrown only at runtime. Meaning everything needs to be double checked (hoping that service object graph is build and verified at app startup and not at later stages on demand).

Pure DI (also known as "Poor Man's DI") is concept that I read about from Mark Seeman's blog posts. It didn't quite catch me at the beginning and left me wondering who would on Earth would like to keep track of all constructor signature changes and adjust it every time.

However, now I do understand ["strongly-typed" part](https://blog.ploeh.dk/2012/11/06/WhentouseaDIContainer/) of the Pure DI principle.
If you are relying on conventions to build your service instances and registration is something like this:

```csharp
public void Configureservices(ServiceCollection services)
{
    services.Add<ISomeService, SomeServiceImpl>();
}
```

There is no compile time check about all required dependencies or whether instance of `SomeServiceImpl` is constructable at all.
Compared to following code when object graph is constructed manually:

```csharp
public void Configureservices(ServiceCollection services)
{
    services.Add<ISomeService>(_ => () =>
    {
        return new SomeServiceImpl(new Dependency() /* missing connection string accessor */)
    });
}
```

You will immediately see compilation error and you will not able to construct new instance of `SomeServiceImpl` without supplying all required dependencies.
Of course, as Mark mentioned it out, using Pure DI - requires much higher level of self-discipline and maintainability of the code drops dramatically. But it has its strengths.

## Running .NET Core App as WebJob in Azure

Once you have done converting from .NET Framework to .NET Core / .NET Standard, the easiest way to verify that everything is working - just by launching project locally. That will open up Console window and output all logging directly there.
However - to get it running under Azure WebJobs host - requires a bit of work to be done upfront.
You have to have `run.cmd` file (or similar executable that follows [naming conventions](https://github.com/projectkudu/kudu/wiki/WebJobs) for Azure Functions host). This executable file will ensure that WebJob is launched properly using `dotnet.exe` tool.

```
dotnet {name-of-your-entry-project}.dll
```

**NB!** Note that you don't need to include `run` (like `dotnet run {name-of-your-entry-project}.dll`).

Locate this file in the root of WebJobs folder and deploy together with your application.

## Some Practical Gotchas

### Switching Between Targeted Platforms

While you are in transition phase between .NET Framework and .NET Core you might need to switch branches and work on some bug fixes in old project version that is still targeting .NET Framework. It's sounds like easy task to do, but when you see errors like this:

![2019-09-02_22-11-45](/assets/img/2019/09/2019-09-02_22-11-45.png)

It might take some brain power of yours to figure out what's going on here.

What we figured out is requirement to delete `obj/` folders. If you have many projects just like we do, this small snippet might become handy to remove all.

```powershell
Get-ChildItem -Path */obj -Attributes Directory | Remove-Item -Recurse
```

Not sure what causes this error, but we are glad to solve it.

### Clean State

Be sure that you start you journey in "clean state". Meaning that it's recommended to "re-clone" repository in different location and have the latest source code straight from origin (source code version control repository). Once you convert your project from .NET Framework project file format to SDK based - there is no explicit file includes anymore (enlisting which files should be included in resulting assembly). All files that are found on the disk in project folder are included in final build by default.
For this mistake it cost me couple hours of effort to fix compilation errors and keeping code up-to-date for API changes caused by some dependency library. At the end it turned out that file was deleted long time ago and my changes to the file are basically waste of time.. Always keep your workbench clean and work on code base that is the latest.

## Summary

Converting to .NET Core technically is not complex task. However, it takes significant amount of time to verify and test converted application to be 200% sure that application is running properly and is ready for production launch. By default project should target .NET Standard if possible.

Looking forward to migrate further to [.NET Core v3.0](https://devblogs.microsoft.com/dotnet/announcing-net-core-3-0-preview-9/).

Happy converting!

[*eof*]
