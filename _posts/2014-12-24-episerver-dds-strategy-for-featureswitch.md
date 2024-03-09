---
title: EPiServer DDS Strategy for FeatureSwitch
author: valdis
date: 2014-12-24 11:30:00 +0200
categories: [.NET, C#, Feature Switch, Episerver, Optimizely]
tags: [.net, c#, feature switch, episerver, optimizely]
---

Thanks to [Paul Smith](http://world.episerver.com/blogs/paul-smith/) for adding some more [EPiServer integration for FeatureSwitch](https://github.com/valdisiljuconoks/FeatureSwitch) library. Now `FeatureSwitch` EPiServer integration library is equipped with DDS storage strategy. This was pretty easy to implement:

```csharp
public class EPiServerDatabaseStrategyImpl : BaseStrategyImpl
{
    public override bool Read()
    {
        var store = typeof(FeatureState).GetStore();
        var definition = store.Items<featurestate>().FirstOrDefault(d => d.Name == Context.Key);
        return definition != null && definition.Enabled;
    }

    public override void Write(bool state)
    {
        var store = typeof(FeatureState).GetStore();
        var definition = store.Items<featurestate>().FirstOrDefault(d => d.Name == Context.Key) ?? new FeatureState { Name = Context.Key };

        definition.Enabled = state;
        store.Save(definition);
    }
}
```

DDS storage strategy gives you more control over how feature state is saved and also makes it easier to alter it (for instance using [Geta’s DDS Admin](https://github.com/Geta/DdsAdmin) plugin).

![](/assets/img/2014/12/dds-strategy.png)

Here goes some more information about how to add new custom strategies and other stuff.

## Adding New Strategies

Currently `FeatureSwitch` comes with some [built-in strategies](https://github.com/valdisiljuconoks/FeatureSwitch/wiki#strategies). However it’s natural that there is a need to extend `FeatureSwitch` and add new strategies. This is really easy to implement.
`FeatureSwitch` recognizes two types of strategies:

- *Initializable*: these strategies take part only during `FeatureSwitch` initialization routine and are called while feature set context is built-up. To create initializable strategy you need to implement `FeatureSwitch.Strategies.IStrategy` interface.
- *Read-only*: read-only strategies are strategies that are able to read form the underlying storage initial value of the feature but not able to store or save it back. There will be disabled checkbox for features that are marked with these strategies in Asp.Net control panel. To create initializable strategy you need to implement `FeatureSwitch.Strategies.IStrategyStorageReader` interface.
- *Writeable*: these strategies are able to update underlying storage and save new state of the feature. Features marked with these strategies will have enabled checkbox in Asp.Net control panel. To create initializable strategy you need to implement `FeatureSwitch.Strategies.IStrategyStorageWriter` interface.

In order to implement your new initializable strategy it’s enough to implement IStrategy interface.

```csharp
public class EmptyStrategyImpl : IStrategy
{
    public void Initialize(ConfigurationContext configurationContext)
    {
    }
}
```

We will get back to `ConfigurationContext` a bit later. So as you can see this strategy is called when feature set context is built by passing configuration context to the Initialize method.
In order to implement your new read-only strategy it’s enough to implement `IStrategyStorageReader` interface.

```csharp
public class SampleReaderImpl : IStrategyStorageReader
{
    public void Initialize(ConfigurationContext configurationContext)
    {
        throw new System.NotImplementedException();
    }

    public bool Read()
    {
        throw new System.NotImplementedException();
    }
}
```

As you can see there is no feature identification mentioned in `Read` method. Identification key of the feature (one that is used in attribute definition).

```csharp
[AppSettings(Key = "MySampleFeatureKey")]
public class MySampleFeature : BaseFeature
{
```

You can access feature key via `ConfigurationContext.Key` (that is passed in `Initialize` method).

To get rid of `Initialize` method and not bothering yourself to store configurationContext somewhere, you can inherit from `FeatureSwitch.Strategies.Implementations.BaseStrategyReaderImpl`. It will take care of this.

```csharp
public class SampleReaderImpl : BaseStrategyReaderImpl
{
    public override bool Read()
    {
        throw new System.NotImplementedException();
    }
}
```

## Adding New Writable Strategies

To create writable strategy you need to either implement `IStrategyStorageWriter`:

```csharp
public class SampleWriterImpl : IStrategyStorageWriter
{
    public void Initialize(ConfigurationContext configurationContext)
    {
        throw new System.NotImplementedException();
    }

    public bool Read()
    {
        throw new System.NotImplementedException();
    }

    public void Write(bool state)
    {
        throw new System.NotImplementedException();
    }
}
```

Or you can inherit from `FeatureSwitch.Strategies.BaseStrategyImpl` to reduce code a bit:


```csharp
public class SampleWriterImpl : BaseStrategyImpl
{
    public override bool Read()
    {
        throw new System.NotImplementedException();
    }

    public override void Write(bool state)
    {
        throw new System.NotImplementedException();
    }
}
```

## Dependency Injection Support

Strategies are constructed using dependency injection services which means that strategies with constructor injections are supported out-of-box. So strategies like these will be constructed as long as necessary dependencies will be available in IoC (inversion of control) container.

```csharp
private readonly ISampleInjectedInterface _sample;

public StrategyWithParameterReader(ISampleInjectedInterface sample)
{
    _sample = sample;
}
```

## Enable Your Custom Strategy

When new strategy has been implemented it’s important that you enable it.
To enable new strategy you need to:

- Create associated attribute that will be used to map your new strategy to particular feature marking that this feature is backed up by your strategy
- Register this mapping in `FeatureSwitch` context.

To create mapping attribute – you need just new class inheriting from `FeatureStrategyAttribute`:

```csharp
public class MyNewStrategy : FeatureStrategyAttribute
{
}
```

And you need to register new strategy implementation with particular attribute (if you are in [EPiServer context](https://github.com/valdisiljuconoks/FeatureSwitch/wiki/EPiServer-Integration) – you can use EPiServer’s `InitializableModule`, or if you are on your own – you can use [my custom made initializable module infrastructure](https://github.com/valdisiljuconoks/InitializableModule) to register this mapping):

```csharp
var builder = new FeatureSetBuilder(new StructureMapDependencyContainer());
builder.Build(ctx =>
{
    ctx.ForStrategy<mynewstrategy>().Use<mynewstrategyimpl>();
});
```

And now you can use your new strategy on feature:

```csharp
[MyNewStrategy(Key = "MySampleFeatureKey")]
public class MySampleFeature : BaseFeature
{
```

# Configuration Context of the Strategy

`ConfigurationContext` is something that is needed for the strategy to initialize properly and get some info about feature "attached" to. `ConfigurationContext` is passed when strategy is initialized during feature set context build process and passed to `Initialize` method:

```csharp
public interface IStrategy
{
    void Initialize(ConfigurationContext configurationContext);
}
```

Currently there is no built-in support for customizing configuration context (providing ways to extend and hook into configuration context creation process), but this is something I’m looking into in future releases of `FeatureSwitch` library.

Happy coding!

[*eof*]
