---
title: Episerver Scheduled Jobs - Under the Hood
author: valdis
date: 2020-12-07 21:00:00 +0200
categories: [Add-On, Optimizely, Episerver, .NET, C#, Scheduled Jobs]
tags: [add-on, optimizely, episerver, .net, c#, under the hood]
---

## Introduction

Have you ever wondered what really happens when you run your scheduled job in Episerver? What happens when Episerver misses to fire off your job, what is scheduler and how job status notifications are implemented? In order to better understand underlying platform - in my opinion, it's crucial to understand what's going on under the hood. That should help next time when the thing hits the fan and you have a glass of your favorite whisky with you at Friday evening and you are staring at the screenshot in support ticket.

As I see that nowadays scheduled jobs are quite [popular topic](https://www.gulla.net/en/blog/episerver-scheduled-jobs-make-it-fail/) :)

In this blog post we are going to try to peel off some of the details what actually is happening when Episerver needs to run your scheduled jobs.

**NB!** Most of the APIs I'm going to talk about are **internal**. Internal for us developers means - that we should not take dependency on those, should avoid using them in our code and overall stay away. I can't expose Episerver source code directly, therefore I'll describe behavior with sample code as close as I can.

This post is mostly for educational purpose. Some parts don't repeat at production! :)

## Scheduled Job Definition

### Recommended Implementation

Let's start with very simple job definition.

```csharp
[ScheduledPlugIn(DisplayName = "ScheduledJob1")]
public class ScheduledJob1 : ScheduledJobBase
{
    private bool _stopSignaled;

    public ScheduledJob1()
    {
        IsStoppable = true;
    }

    /// <summary>
    /// Called when a user clicks on Stop for a manually started job, or when ASP.NET shuts down.
    /// </summary>
    public override void Stop()
    {
        _stopSignaled = true;
    }

    /// <summary>
    /// Called when a scheduled job executes
    /// </summary>
    /// <returns>A status message to be stored in the database log and visible from admin mode</returns>
    public override string Execute()
    {
        //Call OnStatusChanged to periodically notify progress of job for manually started jobs
        OnStatusChanged(String.Format("Starting execution of {0}", this.GetType()));

        //Add implementation

        //For long running jobs periodically check if stop is signaled and if so stop execution
        if (_stopSignaled)
        {
            return "Stop of job was called";
        }

        return "Change to message that describes outcome of execution";
    }
}
```

This is default job definition created by Episerver Visual Studio template. Few things to note here:

* `[ScheduledPlugIn(DisplayName = "...")]` - you have to annotate your class with attribute in order to get Episerver attention during scanning and registration phase.
* `public class ScheduledJob1 : ScheduledJobBase` - you are inheriting from the base class. Also this plays some role in the whole infrastructure of scheduled jobs.
* `IsStoppable = true;` - this is telling Episerver that you do support graceful shutdown of your job. Whether you have to be good or bad at that time - we will experiment later around this.
* `OnStatusChanged(...)` - this is a way to tell UI users that job did something and here is the thing it did.

### Alternative

You like your parents but sometimes you just want to live alone of with somebody else - when you would like to inherit from different base class for example (or maybe do not want to inherit from anyone and be independent individualist). Then you can just implement `IScheduledJob` interface and you should be good to go.

```csharp
[ScheduledPlugIn(DisplayName = "ScheduledJob1")]
public class ScheduledJob1 : IScheduledJob
{
    public ScheduledJob1() { ID = Guid.NewGuid(); }

    public string Execute()
    {
        return "Great success!";
    }

    public Guid ID { get; }
}
```

### Don't Try this at Home Kids

This is recommended approach at all, but if you are willing to fly super alone - then the only thing you need to have is `static Execute` method. No other obligations required here.

```csharp
[ScheduledPlugIn(DisplayName = "ScheduledJob1")]
public class ScheduledJob1
{
    public static void Execute()
    {
        // do magic here
    }
}
```

**NB!** This is design is kept in Episerver *only* for backward compatibility. Please do not use it, you gonna loose a lot of built-in functionality that is available if you inherit from `ScheduledJobBase` type.

## Important Types

Here is a list of some important types and interfaces that will play role in scheduled jobs infrastructure:

* `ISchedulerService` - implementation responsible for following the given schedule and assign jobs for execution;
* `IScheduledJobExecutor` - the one who actually invokes jobs;
* `IScheduledJobLogRepository` - bookkeeper of the stuff happened;
* `ScheduledJobBase` (together with `IScheduledJob`) - base class or implementation which is recommended to use if you are implementing scheduled jobs;
* `SchedulerOptions` - some of the options you can customize to adjust behavior of the scheduling infrastructure;

Some internal key key role players:

* `IScheduledJobFactory` - this guy can create new instances of the jobs;
* `IScheduledJobLocator` - this guy can locate jobs in your type collection;
* `IScheduledJobScanner` - this guy can scan and find job definitions in your assemblies;

## Big Picture

You have find big picture of some of the most important parts of the scheduled jobs infrastructure.

![scheduled-jobs](/assets/img/2020/12/scheduled-jobs.png)

We are going to walkthrough each of them in sections below.

## Scanning & Registration

Now when we have defined our job and compile the solution, next time when we will run it "scanning and registration" phase will kick in which is responsible for making sure that all scheduled jobs are discovered and registered.

Everything starts from ordinary initializable module (`EPiServer.Scheduler.Internal.SchedulerInitialization`). Process starts with just a few service registrations in DI container. Then during initialization phase init module locates implementation of `IScheduledJobScanner` and kicks it off to thread pool (`Task.Run`) to get scanning process out of the way. This is interesting approach. It basically queues scanning of the jobs to the thread pool thread and then just continues.

Approach looks something like along these lines:

```csharp
[InitializableModule(UninitializeOnShutdown = true)]
[ModuleDependency(typeof (CmsRuntimeInitialization))]
public class SchedulerInitialization : IConfigurableModule, IInitializableModule
{
    public void Initialize(InitializationEngine context)
    {
        _scanner = Task.Run<IScheduledJobScanner>(() =>
        {
          var instance = context.Locate.Advanced.GetInstance<IScheduledJobScanner>();
          instance.Scan();

          return instance;
        });

        context.InitComplete += new EventHandler(OnInitComplete);
    }

    private void OnInitComplete(object sender, EventArgs e)
    {
        var scanner = _scanner.Result;

        ...
    }
}
```

`IScheduledJobScanner` is asking `IScheduledJobLocator` to find all  types with `ScheduledPlugInAttribute` attribute. Remember this attribute is present in sample job at introduction section? Attribute is playing important role here - it identifies types and helps Episerver find all jobs defined in code.

Following properties can be set for the job via this attribute (most of the properties are inherited from `PlugInAttribute` but these are specifically for scheduled jobs):

* `GUID` - unique identifier for your job. It's not mandatory to have a GUID set for your job, but it is **recommended**. This will help Episerver to understand what to do when you are renaming your job types or just moving to the different namespace.
* `IntervalType` - you can specify type of internal at which  job should be executed (e.g. "minutes").
* `IntervalLength` - specify "step" of the interval (e.g. "5").
* `InitialTime` - when job should be started for the 1st time.
* `Restartable` - is job restartable? This will affect what you can do and how job will behave when restart happens.
* `HelpFile` - if someone is interested in some manual or documentation, you can use this property to specify link for help page.


When list of discovered jobs is returned it also tries to detect any duplicates by the same GUID. If so, an exception will be thrown.

When scanning is done, job definitions are sync with database.
And now the scheduler service can kick in and launch scheduling process.

### Scheduler Configuration

Before scheduler takes over there is a check of whether scheduled jobs should be run on this machine. Yes, sometimes (especially in farm cluster scenario) you would like to disable scheduler on some designated machines. Ok, usually it's exactly vice versa - you would like to enable scheduler only on specific machine.

To control this, head over to `web.config` and look for `<episerver>` section.

```xml
<episerver>
    <applicationSettings "enableScheduler"="true" />
</episerver>
```

You can also configure some additional properties (container class `SchedulerOptions`) for the scheduler configuration (we will get back to these later):

* `PingTime` - ping period for the scheduler (default 30 secs).
* `MaximumExecutionAttempts` - how many attempts Episerver should try to execute the job if it's been interrupted. (default 10).
* `ContentCacheSlidingExpiration` - for how long content loaded in scheduled job context is going to live in the cache (default 60 seconds).

The best way how to configure these options is to use `services.Configure<T>()`:

```csharp
[InitializableModule]
public class ConfigureSchedulerInitialization : IConfigurableModule
{
    public void ConfigureContainer(ServiceConfigurationContext context)
    {
        context.Services.Configure<SchedulerOptions>(o =>
        {
            // provided here just as example
            // adjust values to your needs
            o.PingTime = TimeSpan.FromSeconds(5);
        });
    }

    public void Initialize(InitializationEngine context) { }

    public void Uninitialize(InitializationEngine context) { }
}
```

If everything is tip-top, Episerver kicks of The Job Scheduler (`EPiServer.Scheduler.Internal.SchedulerService`).

## Job Execution Pipeline

This job scheduler class is interesting by itself. The problem that it has to solve is basically - stay alive, look for the job which should be executed next and execute it (actually pass control over to another type that we will look at shortly).

What happens when scheduler service is run?

### Job Scheduler Service

It basically creates two wait handles - one for quitting the application (shutdown) and another on the moment of execution of the next job, and awaits in `while` loop to whichever hits first. How it knows when next job should be executed? Well, it reads and calculates it before awaiting.

```csharp
public virtual void Run()
{
    var waitHandles = new [] { _quit, _exec };
    while (!cancellationToken.IsCancellationRequested)
    {
        ReadNextScheduledItem();
        switch (WaitHandle.WaitAny(waitHandles, refreshInterval, true))
        {
            case 0:
                // quit wait handle hit first - exiting the loop
                return;
            case 1:
                // execution wait handle hit first - executing the job
                Execute();
                continue;
        }
    }
}
```

The actual execution wait handle is custom implementation that you can find at `EPiServer.Scheduler.Internal.WaitableTimer` and is quite interesting class by itself. It is using `System.Threading.Timer` and `System.Threading.AutoResetEvent` primitives to simulate wait handle set on the timer (you can wait for event to be raised after some period).

Nowadays cool kidz most probably would just write `await Task.Delay(...)` but.. if you are interested in how we did it in old days - this is class to check it out.

So basically job scheduler is responsible to be in tight loop, look for next job to execute at perfect timing and hand over control to `IScheduledJobExecutor` to invoke the job.

But before we jump to job executor, let's see what happens inside `ReadNextScheduledItem()` method.

### Detecting Next Job to Run

Now we come to the point where we need to decide which is the next scheduled job we need to execute. It's done in `ReadNextScheduledItem` method.

Following steps are done in this method:

* looping through all registered scheduled jobs (list is retrieved from `IScheduledJobRepository`)
* if job is enabled & is not registered as failed (we will get back to this in "Job Execution" section)
* if job is required to be executed immediately - it's selected as next scheduled job
* otherwise - job list is sorted ascending by `NextExecutionUTC` property value and first which is not running or missing pings is selected as next scheduled job to be executed.

What does it mean that job needs to be executed immediately? Well, there are few checks performed to determine if job needs to be executed immediately:

* if job is restartable (`[ScheduledPlugIn(Restartable = true, ...)]`)
* AND execution attempts are not exceeding threshold (default 10)
* OR job is missing pings (more in "Pinging Jobs" section)
* AND
* job status is `Running`
* OR `LastExecutionStatus` was set to `Aborted` (this is happening when hosting system is gracefully shutting down and notifying about this running jobs at that moment). More about this in "Stopping the Scheduler" section.

If job is not required to be executed immediately - then job closest by scheduled execution time is picked (winning job).

Winning job next scheduled execution date and time is set to our old friend `WaitableTimer _exec` and we basically return back to the `while` loop:

```csharp
public virtual void Run()
{
    var waitHandles = new [] { _quit, _exec };
    while (!cancellationToken.IsCancellationRequested)
    {
        ReadNextScheduledItem();
        switch (WaitHandle.WaitAny(waitHandles, refreshInterval, true))
        {
            ...
        }
    }
}
```

This means that if date and time is set for `_exec` awaitable - `switch` statement will be "resumed" once any of those 2 will reset event (remember `AutoResetEvent` class used there?).

If `_exec` wait handle won - job is carried out for the execution. Here we then switch over to `IScheduledJobExecutor` interface and specifically to  `DefaultScheduledJobExecutor` implementation.

**NB!** But before we switch to the execution - there is one note worth knowing. There could be situations when job scheduler misses scheduled time for the job - and therefore is missing also to pass it over to the executor. This situation is also handled in `ReadNextScheduledItem` method. However, if scheduler missed job execution, it will pick it up only after 1 minute. Because of this code in `ReadNextScheduledItem` method:

```csharp
if (CurrentScheduledItem.NextExec.AddMinutes(1.0) < DateTime.UtcNow)
{
    ...
}
```

### Starting Job Execution

When scheduler has picked up the job for the execution - control is passed over to implementation if `IScheduledJobExecutor` and specifically by default to `EPiServer.Scheduler.Internal.DefaultScheduledJobExecutor` to the `StartAsync` method.

`SchedulerService` creates new instance of `JobExecutionOptions` class, sets couple of properties and passes it as parameter to `StartAsync` method. `JobExecutionOptions` class contains one important property - `RunSynchronously` which is not set by the scheduler - and thus resulting with value `false`. This will be important to remember when we will look at how jobs are executed manually.

```csharp
_executor.StartAsync(
    job,
    new JobExecutionOptions()
    {
        Trigger = CurrentScheduledItem.Trigger,
        ...
    });
```

Before *actual* execution of the job executor does few checks:

* checks what is the type of the trigger for the execution
* if it's `Scheduler` OR `Restart` - then it's ordinary controlled scheduled execution of the job. in this case - if job status update in database succeeds - then job is executed. if not - job gets new status `UnableToStart`.
* otherwise - it checks for passed-in (from `SchedulerService`) execution date time and if it's not `DateTime.MinValue` but is in the past - job status is updated in the database and job is immediately executed.

Now we got to the point where all the prerequisites are satisfied, everything is  tip-top and we are now at `InternalStartAsync` method.

Executor keeps track of currently running jobs in `ConcurrentDictionary`. So this is a mechanism behind to avoid double executions for the same job.

Following is the code inside final execution method:

* check if currently scheduled job is not already executing - by checking the content of the `ConcurrentDictionary` of running jobs. If job is running - abort.
* create class instance of the job type. Here Episerver is using `IScheduledJobFactory` implementation.

### Job Factory

Scheduled jobs during scanning and registration phase are written down to database by capturing some of the type information. Like:

* Assembly name
* Type fully qualified name (FQN)
* Target method to invoke

Factory `IScheduledJobFactory` default implementation `DefaultScheduledJobFactory` will try to create new instance of your job type firstly via DI container. If that will fail - then will look for parameterless constructor.

Few things can go wrong here while creating new job instance:

* `FileNotFoundException` - specified file does not exist anymore
* `TypeLoadException` - specified job type could be not be found
* StructureMap specific exceptions on why class creation failed

By default Episerver wants you to inherit from `ScheduledJobBase` class (which is fine, because you get some things out of the box - like status updates).

```csharp
[ScheduledPlugIn(DisplayName = "ScheduledJob2"]
public class ScheduledJob2 : ScheduledJobBase
{
    public override string Execute()
    {
        ...
    }
}
```

But at the same time, Episerver also respects your design and allows to have static method.
Interesting is approach with static method. Of course, even if you ignore all goodies provided by Episerver infrastructure and not using `ScheduledJobBase` class as your master parent, Episerver needs to "represent" your `static` method somehow. This is done via `MethodProxyJob` class.

```csharp
var jobInstance = serviceLocator.GetInstance(typeOfJob);

if (jobInstance == null)
{
    return new MethodProxyJob(job.ID, type, method, job.InstanceData);
}
```

Instance data of the scheduled job class is binary serialized - in order to deserialize it later when `MethodProxyJob` will need instance to execute method on.

If factory will not find any of these definitions - it will just blow up then.
When factory is blowing up due to failure of the creation of the job type - it will mark job as failed via `FailedScheduledJobRegistry`. Marking jobs as failed is only in-memory storage. Meaning that on the next restart of the site - Episerver will start over and will try hard to get your job executed again.

### Executing Job

Once we have figured out which job to run, have created instance of it, now we are at the point of invoking actual `Execute` method finally.

Bu default this is scheduled on thread pool thread:

```csharp
public class DefaultScheduledJobExecutor
{
    public void InternalStartAsync()
    {
        ...
        Task.Run(() => Execute(...));
    }
}
```

First things first, when job starts running - it's important to shout this out loud to everyone:

```csharp
_dataAccess.UpdateRunningState(job.ID, true, false);
```

Then we open stopwatch and basically let it run:

```csharp
var sw = new Stopwatch();
sw.Start();

job.Execute();

sw.Stop();
```


After job is executed - then as last thing is to report status of this execution.

```csharp
await LogExecutionAsync(job.ID, ..., status, ...).ConfigureASwait(false);
```

### Pinging Job

It's important to control also what's going on with running scheduled job instance. This is done via additional Timer:

```csharp
isAliveTimer = new Timer(PingDatabaseWithRunningStateCallback,
                         job.ID,
                         _schedulerOptions.GetPingTime(job),
                         _schedulerOptions.GetPingTime(job));
```

Pinging basically means that time to time (`_schedulerOptions.GetPingTime`) executor updates running state of the job. By default this happens every 30 seconds. And seems like it's not able to change it right now.

Why it's important to ping running jobs? Last ping time will be used to determine whether job should be executed immediately after system restart.

It's assumed that job has crashed after pings are missed for 4 times. So if last ping is 2 minutes or older - Episerver assumes that job has crashed. Missing pings is just one of the parameters to determine if job should be executed immediately.

### Job Status Updates

Job status update is designed to inform about progress of the job, how far (or deep) it got - basically what's going on. So that interested personnel is informed about progress and can make appropriate decisions.

```csharp
public class DefaultScheduledJobExecutor
{
    private void Execute()
    {
        var jobBase = instance as ScheduledJobBase;
        if (jobBase != null)
        {
            jobBase.StatusChanged += JobInstance_StatusChanged;
        }

        instance.Execute();

        if (jobBase != null)
        {
            jobBase.StatusChanged -= JobInstance_StatusChanged;
        }
    }
}
```

Implementation of the status update is done again via `ScheduledDB` (`_dataAccess`).

```
this._dataAccess.UpdateCurrentStatusMessage(scheduledJobBase.ScheduledJobId, e.Message);
```

which at the end - updates `tblScheduledItem` table `CurrentStatusMessage` column for a given job record.

![scheduled-jobs-status-update](/assets/img/2020/12/scheduled-jobs-status-update.png)

Current status message is displayed in administration user interface for appropriate scheduled job.

### Manual Execution

Sometimes you need to execute scheduled job out of any schedule - just right now, right here.

For this purpose you are able to use manual execution feature. You can either kick-off job programmatically or from the user interface.

![scheduled-jobs-manual-exec](/assets/img/2020/12/scheduled-jobs-manual-exec.png)

Under the hood you will have to use our old friend - `IScheduledJobExecutor`. This is code that's being executed from Episerver CMS user interface:

```csharp
Executor.Service.StartAsync(_job, new JobExecutionOptions
{
    Trigger = ScheduledJobTrigger.User
});
```

There is also another "legacy" way to execute scheduled job:

```csharp
public class MyStuff
{
    private IScheduledJobRepository _repo;

    public MyStuff(IScheduledJobRepository repo)
    {
        _repo = repo;
    }

    public void DoMagic()
    {
        var job = _repo.Get(...);
        job.ExecuteManually();
    }
}
```

This approach is not recommended - it just doesn't feel right to call `Execute` method right on job type :) It's even decorated with `[Obsolete]` attribute to stress this.

Anyways, let's look inside `ExecuteManually` method:

```csharp
public void ExecuteManually()
{
    Executor.StartAsync(
        this,
        new JobExecutionOptions
        {
            Trigger = ScheduledJobTrigger.User,
            RunSynchronously = true,
            PreserveAsyncCompatibility = true
        }
    );
}
```

Look closer at `RunSynchronously` and `PreserveAsyncCompatibility` values. Even if Episerver sets `RunSynchronously` which should mean to execute job synchronously, it also enables async compatibility.

What does this all mean?

To answer - we have to return back to scheduled job executor.
Before deciding how to run the job, we have this code fragment:

```csharp
if (ShouldRunAsynchronously(instance, options))
{
    var task = Task.Run(() => Execute(....));
    return task;
}
else
{
    var task = Execute(...);
    return Task.FromResult(task.GetAwaiter().GetResult());
}
```

**NOTE!** The only reason (at least for now) why `Execute` is async - just to be able to await on log entries to be saved in database (`await LogExecutionAsync(...)`).

Now we have to tear apart `ShouldRunAsynchronously` method:

```csharp
private static bool ShouldRunAsynchronously(IScheduledJob instance, JobExecutionOptions options)
{
  if (!options.RunSynchronously) return true;

  return options.PreserveAsyncCompatibility && instance is ScheduledJobBase;
}
```

Here few things happen:

* If anyone set `RunSynchronously` to `false` -> job is executed asynchronously.
* If anyone set `PreserveAsyncCompatibility` to `true` AND job is a subtype of `ScheduledJobBase` -> job is executed asynchronously.

Here you can see - "normal" scheduled job is executed manually - and still gets queued on thread pool thread:

![scheduled-jobs-manual-exec-2](/assets/img/2020/12/scheduled-jobs-manual-exec-2.png)

On the other hand - we can experiment a bit.
If we manually execute "static" job which does not inherit from `ScheduledJobBase` parent - we will get the same thread from which manual execution was triggered.

```csharp
[ScheduledPlugIn(
    DisplayName = "ScheduledJob3",
    GUID = "016DB745-F2F1-41C0-A078-778D08F1B3CA")]
public class ScheduledJob3
{
    public static void Execute()
    {
        ...
    }
}
```

Here you can see that the same thread (I called it from ASP.NET MVC controller action) is used to execute our scheduled job:

![scheduled-jobs-manual-exec-3](/assets/img/2020/12/scheduled-jobs-manual-exec-3.png)

**NB!** So you have to be careful how your jobs are defined and how they are executed - it will depend what features you will get access to (`HttpContext` is one of the first type to step on).

## Stopping and Restarting, Error Handling

### Stopping the Job

It just feels much better to stop the job and not to `kill` it. There are situations when you are sick and tired of waiting and just want to move on. Then you will have to look for an opportunity to stop the job.

Again you can do it from Episerver CMS user interface or programmatically:

```csharp
this.Executor.Service.Cancel(_job.ID);
```

Before actually stopping the job, cancel event is raised via `IEventRegistry` (`ID:184468e9-9f0d-4460-aecd-3c08f652c73c`). This allows to "capture" stopping of the job on other Episerver instances if interested.

If job by given ID is still running AND again is child class of `ScheduledJobBase` AND is `IsStoppable = true` then `Stop()` method is called on the instance of the job.

What happens when you are "bad citizen"? For example if you tell Episerver that job is stoppable but actually do not respect this?

Here via `Stop` method class level field is set to `true` -> so this means that you should respect it and check regularly (if your job is quite heavy and intensive).

```csharp
[ScheduledPlugIn(GUID = "...")]
public class ScheduledJob1 : ScheduledJobBase
{
    private bool _stopSignaled;

    public ScheduledJob1()
    {
        IsStoppable = true;
    }

    public override void Stop()
    {
        _stopSignaled = true;
    }

    public override string Execute()
    {
        if (_stopSignaled)
        {
            return "Stop of the job was called";
        }

        return "Great success!";
    }
}
```

If you don't respect stop signal and keep running - probably no one will call you again. Basically there are no rules around this - how job should be killed forcibly.

### Stopping the Scheduler

What happens when the whole system does shutdown? Remember that whole scheduled job infrastructure is initialized in `IInitializableModule`. This means that Episerver is also calling `Uninitialize` method for graceful shutdowns (not guaranteed but happens sometimes).

So scheduler service is able to stop all currently running jobs also.

```csharp
public void Uninitialize(InitializationEngine context)
{
    ServiceLocator.Current.GetInstance<ISchedulerService>().Stop();
}
```

Inside `SchedulerService` when service is stopping `_quit` event is set `AutoResetEvent` - meaning that circuit breaker is set for scheduling loop.

```
var waitHandles = new { _quit, _exec.Event };
while (...)
{
    switch (WaitHandle.WaitAny(waitHandles, refreshInterval, true))
    {
        ...
    }
}
```

After scheduler loop is broken, Episerver gives **20 seconds** for everyone to pack their parachutes and jump!

If there were any currently running jobs -> those got killed and status is set to `ScheduledJobExecutionStatus.Aborted`. Status is saved in database.

### Restart

When system has been interrupted and at that moment running jobs got aborted, this is taken into account next time scheduler service starts.

When reading which job to execute there is a check should job be executed immediately. More info about what other checks are performed there in "Detecting Next Job to Run" section.

## What to Improve? Suggestions for Episerver

### Provide Access to Trigger Type

When my scheduled jobs runs I would like to access in what manner job has been executed.

Either through base class:

```csharp
[ScheduledPlugIn(GUID = "...")]
public class ScheduledJob1 : ScheduledJobBase
{
    public override string Execute()
    {
        if (Trigger == ScheduledJobTrigger.User)
        {
            // does some manual execution specific magic
            ...
        }

         return "Great success!";
    }
}
```

Or maybe even via `Execute()` method:

```csharp
[ScheduledPlugIn(GUID = "...")]
public class ScheduledJob1 : ScheduledJobBase
{
    public override string Execute(ScheduledJobTrigger trigger)
    {
        if (trigger == ScheduledJobTrigger.User)
        {
            // does some manual execution specific magic
            ...
        }

         return "Great success!";
    }
}
```

### Allow Custom Schedule Calculator

Despite that `SchedulerService` is registered by default in DI container as singleton, it has some `virtual` methods meaning that I would be able to override something if I would need to.

But browsing through scheduler service and job executor - it was clear that there is no concrete separation of scheduler, executor and schedule calculator - which would allow me to plug in my own `IScheduledJobExecutionTimeCalculator` (does not exist) implementation - allowing me to come up with my own calendar when job needs to be executed.

Some of the parts are done in `SchedulerService` (like deciding which is next job and when it needs to execute) and some parts are done in `IScheduledJobExecutor` which is responsible for deciding when is next time job should be running after execute.

### Access to Last Successful Execution

There are cases when you need to understand when job has executed successfully last time (for example if you have some differential sync jobs - deltas are sync from specific checkpoints). Of course nowadays we do have our own implementations of checkpoint saves (just like F5 in video games) but it would be nice if this would be available for the job instance.

For example:

```csharp
[ScheduledPlugIn(GUID = "...")]
public class ScheduledJob1 : ScheduledJobBase
{
    public override string Execute()
    {
        if (LastSuccessfulTimeUtc < _timeProvider.UtcNow.AddDays(-5))
        {
            // does some magic if job hasn't been ran for more than 5 days
            ...
        }

         return "Great success!";
    }
}
```

### Allow Async Execution

This is big question for jobs especially for those that do synchronizations, remote RPC calls (aka Web API) or any other async operations.

I would like to see this possible:

```csharp
[ScheduledPlugIn(GUID = "...")]
public class ScheduledJob1 : ScheduledJobBase
{
    public override async Task<string> ExecuteAsync()
    {
        var result = await _apiClient.CallAsync();

        return "Great success!";
    }
}
```

### Parametrized Execution

Scheduled job reminds me just a controlled method execution. Sometimes it's some references to checkpoints, sometimes it points you to the correct configuration, sometimes it gives you precise target object to work on.

Would be awesome if you could pass some arguments to that method.

Like along these lines:

```csharp
public class ParameterClass
{
    public DateTime LastSyncCheckpoint { get; set; }
}

[ScheduledPlugIn(GUID = "...")]
public class ScheduledJob1 : ScheduledJobBase<ParameterClass>
{
    public override string Execute(ParameterClass parameters)
    {
        if (parameters.LastSyncCheckpoint < TimeProvider.UtcNow.AddDays(-5))
        {
            // do some magic
        }

        return "Great success!";
    }
}
```

Would be great if job could mutate these arguments and would get back updated values on next execution cycle.

Some of the attempts to make this possible was done here at [Geta.ScheduledParameterJob](https://github.com/Geta/Geta.ScheduledParameterJob) module.

### Provide Interactive Overview

As site admin - I would like to see all scheduled jobs in single place, single table, with automatic updates of currently executing jobs and failed jobs, enable, disable, execute and stop all at the same table.

Someone also tried to fix this challenge [here](https://github.com/valdisiljuconoks/TechFellow.ScheduledJobOverview).

Things might change in next major version wave (aka Episerver on .NET Core).

## Summary

Scheduled job is not a beast in your site, it is designed to run in background and silently make sure that your jobs are following schedule, are executing and properly shutdown if needed.
You can let Episerver to handle everything, you can also change settings, execute and kill jobs yourself.
You can also override some of the system implementations if you are brave enough :)

Hope this gives you some more insights of what's going on when Episerver is asked to execute your job on given schedule.

Super big thanks to [Henrik Nyström](https://twitter.com/henriknystrom) for fixing all my bugs here :)

Happy scheduling!

[*eof*]
