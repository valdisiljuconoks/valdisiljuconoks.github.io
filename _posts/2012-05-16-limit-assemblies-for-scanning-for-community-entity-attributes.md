---
title: Limit assemblies for scanning for Community Entity Attributes (v0.3)
author: valdis
date: 2012-05-16 09:35:00 +0200
categories: [.NET, C#, Episerver, Optimizely]
tags: [.net, c#, episerver, optimizely]
---

If you are working with EPiServer Relate+ product then you know that there is a extendable entities in the framework that allows you to add new attributes to existing entities to store some additional info if required.



Default approach to set and get values from those attributes are using stringly-typed interface, like:



```csharp
return user.GetAttributeValue<string>("SkypeUsername");
```

To get rid of this and to access entity in strongly-typed manner can be accomplished either using:

* CustomEntityProvider
* Or by using some custom interceptors.


Later one is implemented in `Geta.Community.EntityAttributeBuilder` library. Using this lib you can get access to entities attributes in following manner:


```csharp
var userAttr = user.AsAttributeExtendable<UserAttributes>();
userAttr.FatherName = "John";
```

Library had to scan all the assemblies in current application domain and look for particular types decorated with known attributes for `Geta.Community.EntityAttributeBuilder`.

Building and using this library in a large projects where assemblies may reach up to 250 and more may cause some startup delays as it was scanning everything and trying to sync attributes. Usually such meta data is stored in few assemblies and no need for extra all assemblies scanning process to find ones.



Latest version of `Geta.Community.EntityAttributeBuilder` library (0.3) is released with feature that limits scanning process to known list of assemblies. One that you can found in `<episerver.framework>` element `<scanAssembly>`.



So using latest version you can now configure to limit scanning. Add this to your configuration file:


```xml
<configSections>
  <section name="entityAttributeBuilder" type="Geta.Community.EntityAttributeBuilder.EntityAttributeBuilderConfiguration, Geta.Community.EntityAttributeBuilder" />
</configSections>
```

Add add configuration section:


```xml
<entityAttributeBuilder>
  <scanAssembly>
    <add assembly="YourAssemblyNameWhereAttributesAreLocated" />
    <add assembly="YourAnotherAssemblyName" />
  </scanAssembly>
</entityAttributeBuilder>
```

Package as usual can be found in [nuget.episerver.com](https://nuget.episerver.com) (Id: `Geta.Community.EntityAttributeBuilder`).

Hope this helps!

[*eof*]
