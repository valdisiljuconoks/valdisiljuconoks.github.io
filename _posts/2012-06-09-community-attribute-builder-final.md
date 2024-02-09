---
title: Community Attribute Builder final
author: valdis
date: 2012-06-09 19:15:00 +0200
categories: [.NET, C#, Add-On, Episerver, Optimizely]
tags: [.net, c#, add-on, episerver, optimizely]
---

If you are working with Relate+ platform most probably you are using entities attributes to attach some additional info to particular entity type. This is a great way to extend built-in functionality but however involved a bit of manual work:

* You have to define attribute for each of the entity in Relate+ administrative UI (which honestly speaking could be made more user friendly).
* You have to refer to the attribute via stringly-typed interface.

Latter for me is most embarrassing.

```csharp
instance.SetAttributeValue("attributeName", attributeValue);
```

This approach could give you few real scenarios where such is unacceptable:

* When you will margin for a grammar error when typing attribute name – that could be discovered only during runtime.
* You cannot use static code analysis tools. Like to search for all attribute usages.
* You cannot use some of the refactoring tools –> like rename attribute name.
* You can easily change type of the attribute and forget about code that uses that, effect –> runtime error(s).
* Use cannot leverage all the power of Visual Studio (for instance to provide Intellisense over available attributes)


So therefore I created package that should relief tasks with Community entity’s attributes – **Community Attribute Builder**.

There were already some posts on this topic [here](http://world.episerver.com/Blogs/Valdis-Iljuconoks/Dates/2012/2/Synchronize-Relate-entitys-attributes-in-a-strongly-typed-way), [here](http://world.episerver.com/Blogs/Valdis-Iljuconoks/Dates/2012/2/Metadata-driven-Community-queries) and [there](http://world.episerver.com/Blogs/Valdis-Iljuconoks/Dates/2012/5/Limit-assemblies-for-scanning-for-Community-Entity-Attributes-v03). Latest version (1.0) is out and this is just a summary of what’s supported.

1. Describe community entity’s attributes in code. Those attributes defined in metadata class are synced during application start-up and added to the target entity:

```csharp
[CommunityEntity(TargetType = typeof(IUser))]
public class SampleAttributeMetadata
{
    [CommunityEntityMetadata]
    public virtual IList<SampleType> SampleAttribute { get; set; }
}
```

2. Limit scanning of the assemblies:


```xml
<configuration>
  <configSections>
    <section name="entityAttributeBuilder" type="Geta.Community.EntityAttributeBuilder.EntityAttributeBuilderConfiguration, Geta.Community.EntityAttributeBuilder" />
  </configSections>

  <entityAttributeBuilder>
    <scanAssembly>
      <add assembly="EntityAttributes" />
      <remove assembly="RemoveAssembly" />
    </scanAssembly>
  </entityAttributeBuilder>

</configuration>
```

3. You can set or get entity’s attribute value(s) using strongly-typed interface.


```csharp
var metadata = instance.AsAttributeExtendable<SampleAttributeMetadata>();
metadata.SampleAttribute = null;
```

4. Support for strongly typed queries

```csharp
var query = new UserQuery();
var qmetadata = query.AsMetadataQuery<SampleAttributeMetadata>();
qmetadata[m => m.SampleAttribute] = new StringCriterion { Value = "the attribute value" };
```

5. Added support for IList<T> type attributes. You can assign via local variable assignment and modification or using line modifications.

```csharp
var entity = instance.AsAttributeExtendable<SampleAttributeMetadata>();

// getter
var collection = entity.SampleAttribute;
collection.Add(new SampleType(100));
// setter
entity.SampleAttribute = collection;
```

or you can use more convenient syntax:

```csharp
var entity = instance.AsAttributeExtendable<SampleAttributeMetadata>();
entity.SampleAttribute.Add(new SampleType(100));
```

Rename, deletion and content migration for the community entity attributes need to me done manually.

Can be found [here](http://nuget.episerver.com/en/OtherPages/Package/?packageId=Geta.Community.EntityAttributeBuilder) (or search by “community” in NuGet package manager).


Hope this helps!

[*eof*]
