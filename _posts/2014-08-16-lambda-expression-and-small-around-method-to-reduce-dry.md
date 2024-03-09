---
title: Lambda expression and small “around” method to reduce DRY
author: valdis
date: 2014-08-16 11:30:00 +0200
categories: [.NET, C#, Lisp]
tags: [.net, c#, lisp]
---

When you are acting as façade or in any other “forward-like” role sometimes error handling may come pretty boring stuff to deal with.
For instance if you are acting as front-end client proxy class for some `REST Api` behind the scenes, code below may look familiar – that you have to replicate yourself all the time and catch for instance in this case IO error and response content format error (if they return invalid formatted `Json` object).

```csharp
protected void Operation1()
{
    try
    {
        // some black magic happens here..
    }
    catch (HttpRequestException e)
    {
        // handle IO issue
        return …;
    }
    catch (JsonReaderException e)
    {
        // handle response content issue
        return …;
    }
}

protected void FancyOperation2()
{
    try
    {
        // some black magic happens here..
    }
    catch (HttpRequestException e)
    {
        // handle IO issue
        return …;
    }
    catch (JsonReaderException e)
    {
        // handle response content issue
        return …;
    }
}
```

We are know that repetition is anything about to remember it better, but for me as design practitioner and `DRY` principle defender this seems to be too much. There is definitely a space to improve and reduce some of the duplicates.
The most simplest solution is to utilize Lambda expressions and refactor these methods to use “around” method (for me it drives small similarities with [Lisp macros](http://stackoverflow.com/questions/267862/what-makes-lisp-macros-so-special)).
So what we could do better in this case is to create new method that receives Action and does all boring stuff like handling the error cases:

```csharp
protected void Try(Action action)
{
    try
    {
        return action();
    }
    catch (HttpRequestException e)
    {
        // handle IO issue
        return …;
    }
    catch (JsonReaderException e)
    {
        // handle response content issue
        return …;
    }
}
```

So `Operation1` and `FancyOperation2` now looks much better and cleaner.

```csharp
 protected void Operation1()
{
    Try(() => {
        // some black magic happens here..
    });
}

protected void FancyOperation2()
{
    Try(() => {
        // some black magic happens here..
    });
}
```


I guess it’s a bit better now.


Happy Coding!
[*eof*]
