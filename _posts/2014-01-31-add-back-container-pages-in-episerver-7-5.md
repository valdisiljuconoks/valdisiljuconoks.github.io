---
title: Add back Container Pages in EPiServer 7.5
author: valdis
date: 2014-01-31 16:35:00 +0200
categories: [.NET, C#, Add-On, Episerver, Optimizely]
tags: [.net, c#, add-on, episerver, optimizely]
---

Container pages in EPiServer 7.5 is [no more a built-in system type](http://world.episerver.com/Documentation/Items/Upgrading/EPiServer-CMS/75/Breaking-changes/) which is actually good because now everything in based on template implementation.

If you are still looking for a solution for container pages code snippets below could be useful.

I kind of liked solution found in AlloyTech Mvc sample site for 7.0 – when you define interface and your page type is “implementing” that eventually your page instance SelectedTemplate is set to null – resulting in so called container pages.


```csharp
[ContentType(GroupName = GroupNames.MyGroup,
             DisplayName = “Sample page”,
             GUID = “9B02B266-D3C9-409E-93D2-21D26657F410″,
             AvailableInEditMode = true)]
public class SamplePage : BasePageData, IContainerPage
{
    …
```

In EPiServer 7.5 solution is much more cleaner. All you need to do is register new [UIDescriptorInitializer](http://world.episerver.com/Documentation/Class-library/?documentId=framework/7.5/47dc5e02-00ec-57c8-cc83-a4f43869b5ac).

```csharp
[ServiceConfiguration(typeof(IUIDescriptorInitializer))]
public class ContainerPageUIDescriptorInitializer : IUIDescriptorInitializer
{
    public void Initialize(UIDescriptorRegistry registry)
    {
        var containerTypes = TypeAttributeHelper.GetTypesChildOf<IContainerPage>();
        foreach (var type in containerTypes)
        {
            registry.Add(new UIDescriptor(type)
                {
                    DefaultView = CmsViewNames.AllPropertiesView,
                    IconClass = ContentTypeCssClassNames.Container
                });
        }
    }
}
```

This initializer searches for all types those are implementing `IContainerPage` interface and registers container page like UI descriptor for them. Interface could be of course defined by you.

```csharp
TypeAttributeHelper.GetTypesChildOf<IContainerPage>();
```

This is just a small helper class taken from other my plugins that finds all types that is child of specified type.

```csharp
public class TypeAttributeHelper
{
    public static IEnumerable GetTypesChildOf()
    {
        var allTypes = new List();
        foreach (var assembly in GetAssemblies())
        {
            allTypes.AddRange(GetTypesChildOfInAssembly(typeof(T), assembly));
        }

        return allTypes;
    }

    private static IEnumerable GetAssemblies()
    {
        return AppDomain.CurrentDomain.GetAssemblies();
    }

    private static IEnumerable GetTypesChildOfInAssembly(Type type, Assembly assembly)
    {
        try
        {
            return assembly.GetTypes().Where(t => t.IsSubclassOf(type) && !t.IsAbstract);
        }
        catch (Exception)
        {
            // there could be situations when type could not be loaded
            // this may happen if we are visiting *all* loaded assemblies in application domain
            return Enumerable.Empty();
        }
    }
}
```

Happy containering!

[*eof*]
