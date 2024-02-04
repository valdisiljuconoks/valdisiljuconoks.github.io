---
title: Scheduled Jobs Updates
author: valdis
date: 2016-12-28 22:45:00 +0200
categories: [Add-On, .NET, C#, Localization Provider, Episerver, Optimizely]
tags: [add-on, .net, c#, open source, localization, episerver, optimizely]
---

Some great updates have been delivered to EPiServer Scheduled Jobs infrastructure within [version 10.3](http://world.episerver.com/releases/episerver---update-143/).
In this blog post we will review some of the important changes.

## Scanning Process

Property `IsStoppable` is now calculated during scanning process. Scheduled jobs scanner detects whether job is stoppable or not. If so - it's written down to the database and used later when job is being triggered.

Also - now with v10.3 you don't need to inherit from base class (`EPiServer.Scheduler.ScheduledJobBase`). In this case all you need is to have static `string Execute()` method:


```csharp
[ScheduledPlugIn(DisplayName = "My Custom Job"
                 GUID = "...")]
public class MyCustomJob
{
    public static string Execute()
    {
        ...
    }
}
```

Not really sure why you would like to escape `ScheduledJobBase` class and just have simple class with execution method tho.

Use case that pops in my mind could be when you want to reduce dependency on EPiServer infrastructure and libraries and have just simple class that serves as shell for the actual service or component. In this case scheduled job would become tiny fraction of [outer thin slice of the system](https://tech-fellow.eu/2016/10/17/baking-round-shaped-software/) - with almost no dependencies on underlying execution engine.


## Job Factory - DI Friendly!

Finally! Another great addition is `IScheduledJobFactory` interface and corresponding implementation. As you might guess from the name of the type - it's factory for the scheduled jobs.

It's possible to override this factory with our own stuff, but mostly all of necessary functionality is there.

Most importantly - factory is DI ([Dependency Injection](https://en.wikipedia.org/wiki/Dependency_injection)) aware now!
So you can write (not only specific to "method invocation based" jobs - with static `string Execute()` method):

```csharp
[ScheduledPlugIn(DisplayName = "My Custom Job"
                 GUID = "...")]
public class MyCustomJob
{
    public MyCustomJob(IContentTypeRepository repo)
    {
        ...
    }

    public static string Execute()
    {
        ...
    }
}
```

Now you will not need to use [ServiceLocator anti-pattern](http://blog.ploeh.dk/2010/02/03/ServiceLocatorisanAnti-Pattern/) anymore in scheduled jobs. It was not [good design](http://marisks.net/2016/12/01/dependency-injection-in-episerver/).

## Execution Model Changes

Another great change in v10.3 is addition of `IScheduledJobExecutor` interface and corresponding implementation. This guy makes sure that job is executed correctly, keeps track of running jobs and gives opportunity to cancel (aka *Stop*) the job.

It doesn't matter anymore how you execute the job (scheduler or manually) - with the help of this new interface now both modes are the same. You specify trigger options when you are firing the job (`User` - manual execution):


```csharp
private IScheduledJobExecutor _executor;

...

_executor.StartAsync(jobInstance,
                     new JobExecutionOptions
                     {
                         Trigger = ScheduledJobTrigger.User
                     });
```

Among other things, you can also specify whether you want to execute job asynchronous (default) or wait for the results immediately:


```csharp
private IScheduledJobExecutor _executor;

...

_executor.StartAsync(jobInstance,
                     new JobExecutionOptions
                     {
                         Trigger = ScheduledJobTrigger.User,
                         RunSynchronously = true
                     });
```

## Rename Jobs Safely

-- _What is hardest thing in IT?_<br/>
-- _Right, naming!_


To name scheduled job with first shot is pretty challenging and almost impossible :)


Before EPiServer v10.3 renaming of the scheduled jobs resulted in orphan jobs - ghost ones that are hanging around as obsolete records in the database and make unwanted noise in jobs listing. You can of course delete them from [scheduled jobs overview plugin](http://nuget.episerver.com/en/OtherPages/Package/?packageId=TechFellow.ScheduledJobOverview) safely, but still - not so nice!

Now with support of `[ScheduledPlugIn(GUID=...)]` attribute it's possible to rename jobs safely without worrying to make stale data in the database.

```csharp
[ScheduledPlugIn(DisplayName = "My Custom Job"
                 GUID = "ac034ea9-91d0-471b-821d-456b9072b465")]
public class MyCustomJob
{
    ...
}
```

We all are good copy/pasters. But, you will get an exception if you will forget to change `GUID` of the copied job.

## Measure Execution Cycle

Precise measure of job run cycle in some cases could be pretty important performance indicator. Now with latest version update it's possible to get statistics about how long each run cycle took. It's available in log viewer.

![](/assets/img/2016/12/2016-12-28_17-45-42.png)


## Increased Log Size

If job runs frequently - last 100 log entries might be not sufficient. Now this limitation is gone and you can jump back to history far enough.

## Some Final Thoughts

Awesome to see improvements in stable infrastructure components as well. However - here are some thoughts from my side.

* Still waiting for something similar as [ScheduledJobs Overview](https://github.com/valdisiljuconoks/TechFellow.ScheduledJobOverview) plugin already built-in into EPiServer platform. This is one of the issue among others.

* Would be helpful to see last job's run cycle duration somewhere in job's details page. Fortunately, this is visible in jobs overview page:

![](/assets/img/2016/12/2016-12-28_17-44-56.png)

* Not sure why scanning process is kicked off from the attribute (`EPiServer.PlugIn.ScheduledPlugInAttribute`) and not some scanner implementation. As scheduled job is just yet another plugin for EPiServer - might be that this is coming from Plug-In system.


```csharp
public class ScheduledPlugInAttribute : PlugInAttribute
{
    internal static void Initialize()
    {
        ServiceLocator.Current
                      .GetInstance<IScheduledJobScanner>()
                      .Scan();
    }
}
```

Even more, I couldn't find any references to `Start()` and `AsyncStart()` methods within EPiServer code. Most probably they are invoked dynamically via method invocation. Which is even worse. But that's a topic for another blog post..

Back to the `[ScheduledPlugIn]`. As we know - attributes are just metadata. It should [not have any behavior](http://blog.ploeh.dk/2014/06/13/passive-attributes/).

* I'm missing some nice overview of duration of last X cycles. That would help me to look for patterns and/or peeks in job execution cycles. Fortunately, [overview plugin](https://github.com/valdisiljuconoks/TechFellow.ScheduledJobOverview) has one :)


![](/assets/img/2016/12/2016-12-28_17-45-22.png)

Happy scheduling!

[*eof*]
