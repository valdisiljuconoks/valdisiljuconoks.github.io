---
title: Parse and Validate Quartz JobDataMap
author: valdis
date: 2013-10-17 22:55:00 +0200
categories: [.NET, C#]
tags: [.net, c#]
---

Once you start to implement web site driven by EPiServer Commerce the necessity of scheduled background job(s) sooner or later will rise. A nice approach and existing framework is used in EPiServer Commerce solutions – background jobs are scheduled and [Quartz.Net](http://www.quartz-scheduler.net/).

As we all know scheduled background jobs are sort of agents or workers that do the dirty job over night or any allowed time window to fetch, process, calculate, recalculate, push or do whatever other fancy complex operations that may take time and valuable resources to complete.

Nice advantage of Quartz jobs over `EPiServer CMS` scheduled jobs is option that Quartz jobs may run on separate node and not steal compute power from main node which is intended to be used to serve web pages for end-users.

An example of scheduled job from 1000 feet may look like this:

```csharp
public class SampleQuartzBackgroundJob : IJob
{
    public void Execute(IJobExecutionContext context)
    {
        throw new System.NotImplementedException();
    }
}
```

We are implementing an `IJob` ([info here](http://quartznet.sourceforge.net/tutorial/lesson_2.html)) interface which is pretty straight forward – have a single operation to execute which is the entry point for our job.

However sooner or later we may come up with requirement that claims for job parameterization – for instance, if we discover that we can reuse the same job implementation (type)  for different jobs (instances). Then we would need a way to supply instance with particular parameters.

## Here is your guide – a data map

Job parameterization is implemented using job’s data map – which is a simple list of key / value pairs to supply job instance for behavior customization.

```
    SampleJob
    SampleGroup
    SampleQuartzJobLibrary.SampleQuartzBackgroundJob, SampleQuartzJobLibrary


        Parameter1
        Value 1

        Parameter2
        7
```

Later when the instance of the job is executed we can access job map via `JobExecutionContext`.

```csharp
public void Execute(IJobExecutionContext context)
{
    var map = context.JobDetail.JobDataMap;
}
```

In this case map is just a dictionary with “`IsDirty`” flag feature which gets set to `true` if collection has been changed.

The values of the map is of type `System.Object`. This is fine for storage – but when working with map it’s better to access each value as intended type – `int`, `string`, etc.

```csharp
public void Execute(IJobExecutionContext context)
{
    var map = context.JobDetail.JobDataMap;
    var param1 = map.GetString("Parameter1");
    var param2 = map.GetInt("Parameter2");
}
```

Job’s data map provides an easy way to access map data in “stringly-typed” way.

Usually data provided in job data map is the same as end-user provided in web site – that is required to be validated.

## Is map pointing to the North?

So data which job author or administrator provided in job data map is the same candidate for data validation as submitted data to web server by end-user.

Easy, right?!

```csharp
public void Execute(IJobExecutionContext context)
{
    var map = context.JobDetail.JobDataMap;
    var param1 = map.GetString("Parameter1");
    var param2 = map.GetInt("Parameter2");

    if (string.IsNullOrEmpty(param1))
    {
        throw new ConfigurationErrorsException("Parameter1 is null or empty");
    }

    if (param2 <= 0 || param2 > 15)
    {
        throw new ConfigurationErrorsException("Parameter2 is out of range (1..15)");
    }
}
```

This may be “standard” value validation approach in Quartz jobs and probably somewhere else in .Net universe.

Still for me it seems boring / repetitive code that must be implemented all over the place in almost every job where job data map values would be used.

## I’m going to South actually

So we will take similar approach as MVC model binders took – we will let [`DataAnnotations`](http://msdn.microsoft.com/en-us/library/system.componentmodel.dataannotations.aspx) namespace and particularly `ObjectValidator` to do the job.

Instead of endless if() statements code snippet below would be much better.

```csharp
public void Execute(IJobExecutionContext context)
{
    var map = context.JobDetail.JobDataMap;
    var mapTyped = map.AsTyped();
}
```

Where `SampleJobParameters` is ordinary class that represents possible map parameters and those data types.

```csharp
public class SampleJobParameters
{
    public string Parameter1 { get; set; }

    public int Parameter2 { get; set; }
}
```

Then we will need a logic that will map from `JobDataMap` to our job parameter class.

```csharp
foreach (var key in map.Keys)
{
    var propInfo = result.GetType().GetProperty(key);
    if (propInfo != null)
    {
        if (propInfo.PropertyType.IsEnum)
        {
            var converter = TypeDescriptor.GetConverter(propInfo.PropertyType);
            propInfo.SetValue(result, converter.ConvertFrom(map[key]), null);
        }
        else
        {
            propInfo.SetValue(result, Convert.ChangeType(map[key], propInfo.PropertyType), null);
        }
    }
}
```

Iteration over map keys instead of parameter properties is by design – job data map values have high priority over particular parameter class. Here a reaction of absence of particular property in parameter class could also be easy added – anyway I leave the code as simple as possible for a sake of readability.

Now we may get back an instance of out property class

```csharp
public void Execute(IJobExecutionContext context)
{
    var map = context.JobDetail.JobDataMap;
    var mapTyped = map.AsTyped();

    var command = new SampleCommand();
    command.Execute(param1, param2);
}
```

Let’s go one step further and add a data validation to the parameter class.

```csharp
public class SampleJobParameters
{
    [Required]
    public string Parameter1 { get; set; }

    [Range(1, 15)]
    public int Parameter2 { get; set; }
}
```

Now we need a code that will handle the validation of data map / parameter class data.

Pretty easy!

```csharp
var validationContext = new ValidationContext(result, null, null);
var results = new List();

if (!Validator.TryValidateObject(result, validationContext, results, true))
{
    var msg = string.Format("Error in configuration.{0}",
                            string.Join(Environment.NewLine, results.Select(r => r.ErrorMessage)));

    logger.Error(msg);
    throw new ConfigurationErrorsException(msg);
}
```

Now if we supply invalid data types in job data map or provide incorrect data (out of range, missing, empty, etc) those will be validated.

Complete listing for parsing and validating job data map:

```csharp
public static class JobDataMapExtensions
{
    public static T Read(this JobDataMap map) where T : new()
    {
        var result = new T();
        foreach (var key in map.Keys)
        {
            var propInfo = result.GetType().GetProperty(key);
            if (propInfo != null)
            {
                if (propInfo.PropertyType.IsEnum)
                {
                    var converter = TypeDescriptor.GetConverter(propInfo.PropertyType);
                    propInfo.SetValue(result, converter.ConvertFrom(map[key]), null);
                }
                else
                {
                    propInfo.SetValue(result, Convert.ChangeType(map[key], propInfo.PropertyType), null);
                }
            }
        }

        var validationContext = new ValidationContext(result, null, null);
        var results = new List();

        if (!Validator.TryValidateObject(result, validationContext, results, true))
        {
            var msg = string.Format("Error in configuration.{0}",
                                    string.Join(Environment.NewLine, results.Select(r => r.ErrorMessage)));
            throw new ConfigurationErrorsException(msg);
        }

        return result;
    }
}
```

Benefits of such approach:

* Code reusability in all job types.
* Possible custom validation logic by implementing custom ValidationAttribute classes.
* Strongly typed access over stringly typed access for job parameters.
* Sort of documentation for job data map parameters, types, allowed values, etc.

Hope this helps!
Happy coding!

[*eof*]
