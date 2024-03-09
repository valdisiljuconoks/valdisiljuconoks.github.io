---
title: Object Cache Viewer Joins EPiServer Developer Tools
author: valdis
date: 2019-03-03 10:05:00 +0200
categories: [Add-On, Optimizely, Episerver, .NET, C#, Open Source, Localization Provider]
tags: [add-on, optimizely, episerver, .net, c#, open source, localization provider, localization]
---

It's super cool to work together with the community and unite effort across all areas to strengthen EPiServer ecosystem and toolsets.

After chat with [Joe Mayberry](https://github.com/jaytem) we decided that his local object cache viewer would be great addition to [EPiServer DeveloperTools](https://nuget.episerver.com/package/?id=EPiServer.DeveloperTools).

Thanks to Joe's contributions local object cache viewer is now part of EPiServer DeveloperTools package. During the merge we added small extra to the tool - you can now also see approx. size of the cache entry (might be sometimes useful to know who ate the cake).

![cache-viewer](/assets/img/2019/03/cache-viewer.png)

Code to calculate cache entry size is quite naïve, but seems to give good enough numbers:

```csharp
private static long GetObjectSize(object obj)
{
    if(obj == null)
        return 0;

    try
    {
        using (Stream s = new MemoryStream())
        {
            var formatter = new BinaryFormatter();
            formatter.Serialize(s, obj);

            return s.Length;
        }
    }
    catch (Exception)
    {
        return -1;
    }
}
```

There are also couple of other features that we need to add to cache viewer / manager, but those are coming :)

Cache viewer / manager will be available starting from v3.4

Happy caching!

[*eof*]
