---
title: If all you need is just Dynamic Data Store..
author: valdis
date: 2011-11-04 19:00:00 +0200
categories: [.NET, C#, Add-On, Episerver, Optimizely]
tags: [.net, c#, add-on, episerver, optimizely]
---

We had recently task to make a module data store agnostic. This means that it shouldn’t be a problem to switch from Dynamic Data Store to some other storage implementation. Back and forth. No big deal.

As we currently are developing in our sandbox and making some prototypes only, idea was to bring DDS in our sandbox. We didn’t wanted to deal with all that complexity of EPiServer for now just to be able to use DDS for out testing purposes.

Things to happen, first we need to bring in section from web.config file.


```xml
<section name="episerver.dataStore" type="EPiServer.Data.Configuration.EPiServerDataStoreSection, EPiServer.Data"/>
```

And then we need to copy section and its content as well (for testing purposes we disabled caching):


```xml
<episerver.dataStore>
  <dataStore defaultProvider="EPiServerSQLServerDataStoreProvider">
    <providers>
      <add name="EPiServerSQLServerDataStoreProvider" description="SQL Server implementation of Data Store" type="EPiServer.Data.Dynamic.Providers.SqlServerDataStoreProvider, EPiServer.Data" connectionStringName="EPiServerDB"/>
    </providers>
    <cache defaultProvider="nullCacheProvider">
      <providers>
        <add name="httpCacheProvider" description="Http Cache implementation for DataStore" type="EPiServer.Data.Cache.HttpRuntimeCacheProvider,EPiServer.Data.Cache"/>
        <add name="nullCacheProvider" description="Null Cache implementation for DataStore" type="EPiServer.Data.Cache.NullCacheProvider,EPiServer.Data"/>
      </providers>
    </cache>
  </dataStore>
</episerver.dataStore>
```

Remember to add connection string in <connectionStrings> section as well.


```
connectionStringName="EPiServerDB"
```

That’s almost it.

We would need to reference `EPiServer.CMS.Core` and `EPiServer.Framework` NuGet packages to get `EPiServer.Data.Dynamic` namespace in codebase.



Next thing is to initialize store to get a working copy. After sniffing around EPiServer source code we found that particular initialization module is doing required stuff.


```csharp
EPiServer.Web.InitializationModule._initializationEngine.Add((System.Action) (() => EPiServer.Web.InitializationModule.InitializeDynamicDataStore()), new System.Action(EPiServer.Web.InitializationModule.UninitializeDynamicDataStore));
```

And particularly this method:


```csharp
private static void InitializeDynamicDataStore()
{
  ...
    DynamicDataStoreFactory.Instance = (DynamicDataStoreFactory) new EPiServerDynamicDataStoreFactory();

  ...
}
```


And looking at TypeExtensions that provide convenient way to get store instance:


```csharp
public static DynamicDataStore GetOrCreateStore(this Type type)
{
  return DynamicDataStoreFactory.Instance.GetStore(type) ?? DynamicDataStoreFactory.Instance.CreateStore(type);
}
```

We found a way if we set store instance property to EPiServerDynamicDataStoreFactory. instance – everything works correctly and you are ready to go with DDS support in your very disconnected from the EPiServer sandbox project.

But to make it more easier for testing, imagine that it would classical requirement to be able to mock store and test against some simpler or even mocked instance of the DynamicDataStore type. We adjusted our data store facade to be able to accept various kinds of factories which would make instances of data store as required.


```csharp
private readonly DynamicDataStoreFactory storeFactory;

public OurEPiServerStore()
{
    if(DynamicDataStoreFactory.Instance == null)
    {
        this.storeFactory = DynamicDataStoreFactory.Instance = new EPiServerDynamicDataStoreFactory();
    }

    this.storeFactory = DynamicDataStoreFactory.Instance;
}

public OurEPiServerStore(DynamicDataStoreFactory factory)
{
    this.storeFactory = factory;
}
```

And then it’s just more easier to unit test our store behaviour and surrounding environment with something like this:

```csharp
var factoryMock = new Mock<DynamicDataStoreFactory>();
var storeMock = new Mock<DynamicDataStore>();

factoryMock.Setup(f => f.GetStore(typeof(EmailModel))).Returns(storeMock.Object);

var store = new NewsletterStore<ModelType>(factoryMock.Object);
```

That’s it!
Hope this helps!

[*eof*]
