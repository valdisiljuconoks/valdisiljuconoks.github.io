---
title: Module Dependencies in EPiServer DeveloperTools
author: valdis
date: 2018-05-10 22:30:00 +0200
categories: [Add-On, Optimizely, Episerver, .NET, C#]
tags: [add-on, optimizely, episerver, .net, c#]
---

After gratefully accepted invitation to contribute to the [Episerver Developer Tools](https://github.com/episerver/DeveloperTools) package, I can safely do this:

![2018-05-06_01-55-28-1](/assets/img/2018/05/2018-05-06_01-55-28-1.png)

All credits to the amazing Episerver team, I'm just contributing here :) So that's why happy to share some of the updates in latest version.

While reviewing performance for startup modules (either configurable or initializable ones) to find out some bottlenecks on the site..

![startup-perf](/assets/img/2018/05/startup-perf.png)

I had hard time to find the answer on question - what is the order/dependencies of modules in this list and how they are interconnected?

As I do like data visualization a lot, thought - this might be interesting data source to play around with. It would be great if you could enlist your custom modules and see what are dependencies between them.

So in latest version (v3.2) you have now section "Module Dependencies".

![menu](/assets/img/2018/05/menu.png)

There you can see all the custom (non EPiserver) modules and their connections (dependencies in code):


```csharp
[ModuleDependency(typeof(TinyMceInitialization))]
public class ExtendedTinyMceInitialization : IConfigurableModule
{
    public void Initialize(InitializationEngine context)
    ...
```

This is how it looks in AlloyTech site.

![custom-modules](/assets/img/2018/05/custom-modules.png)

Not so impressive and interesting ;)
Much more fun comes when you enable all modules (including EPiServer built-in ones):

![epi-modules](/assets/img/2018/05/epi-modules.png)

If you do have any feedback or questions, please visit EPiServer.DeveloperTools [GitHub page](https://github.com/episerver/DeveloperTools).
I'm still not quite confident with the choice of the forced directed graph library, but at least it did its job pretty OK.



May the forced directed graphs be with you!<br/>
Happy coding!
[*eof*]
