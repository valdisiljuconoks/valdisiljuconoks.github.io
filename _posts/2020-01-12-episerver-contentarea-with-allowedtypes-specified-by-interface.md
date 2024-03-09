---
title: Episerver ContentArea with AllowedTypes Specified by Interface
author: valdis
date: 2020-01-12 21:45:00 +0200
categories: [Add-On, Optimizely, Episerver, .NET, C#, Open Source]
tags: [add-on, optimizely, episerver, .net, c#, open source]
---

## Background

Once an interesting question was asked on [Episerver Community Slack space](https://episervercommunity.slack.com/) how to work with ContentAreas and specifically `[AllowedTypes]` and specify restrictions based on interface.

![2020-01-12_21-23-34](/assets/img/2020/01/2020-01-12_21-23-34.png)

My answer wasn't quite helpful there and also I'm [quite passionate about looking into the foreign code](https://tech-fellow.ghost.io/tag/look-inside/) and understanding how platform is built - so I decided to write up a bit expanded version of the answer.

According to aforementioned question you might have for example page type with following `ContentArea` definition (based on AlloyTech sample site):

```csharp
[ContentType(GUID = "19671657-B684-4D95-A61F-8DD4FE60D559")]
public class SamplePage : SitePageData
{
    [AllowedTypes(
        AllowedTypes = new[] {typeof(ISpecificBlock)})]
    public virtual ContentArea MainContentArea { get; set; }

    ...
}
```

And some random blocks:

```csharp
public interface ISpecificBlock
{
}

[ContentType(DisplayName = "SpecificBlock", GUID = "...")]
public class SpecificBlock : BlockData, ISpecificBlock
{
}

[ContentType(DisplayName = "SpecificBlock 2", GUID = "...")]
public class SpecificBlock2 : BlockData, ISpecificBlock
{
}
```

If you will run this sample code in AlloyTech - you will see that once you created instance of either `SpecificBlock` or `SpecificBlock2` blocks - you can't really drag them onto `MainContentArea` editor for `StartPage`.

Seems like Episerver is not "expanding" interface definition for allowed types for content areas.

**NB!** Solutions may contain some workarounds and hacks. Use on your own risk!

## Getting It to Work - Registration

Built-in magic happens in class named `ContentDataAttributeScanningAssigner` which implements `IContentTypeModelAssigner` interface. Method called `AssignValuesToPropertyDefinition`.

In order for us to alter behavior - the easiest way would be to intercept call to original assigner and mix-in our logic along the way.

For this to work we will need to register interceptor. Not sure why Episerver is registering some of the dependencies at `ConfigurationComplete` event (might be due to fact that services registered then needs to be "last in the row"). This happens in module called `CmsRuntimeInitialization`. So we will need to even more last in the row and intercept assigner after it's being registered in service collection.

```csharp
[InitializableModule]
[ModuleDependency(typeof(CmsRuntimeInitialization))]
public class DependencyResolverInitialization : IConfigurableModule
{
    public void ConfigureContainer(ServiceConfigurationContext context)
    {
        context.ConfigurationComplete += (sender, args) =>
        {
            context.Services.Intercept<IContentTypeModelAssigner>(
                (sl, svc) =>
                    new IContentTypeModelAssignerInterceptor(
                        svc,
                        sl.GetInstance<ITypeScannerLookup>()));
        };
    }
}
```

## Getting It to Work - Intercepting Assignment

Next step is to intercept property definition model assignment and "expand" mentioned interface into concrete classes.

For that we will need access to original `IContentTypeModelAssigner` and also we will require `ITypeScannerLookup` to find concrete classes that implement mentioned interface.

```csharp
public class IContentTypeModelAssignerInterceptor : IContentTypeModelAssigner
{
    private readonly IContentTypeModelAssigner _inner;
    private readonly ITypeScannerLookup _typeScannerLookup;

    public IContentTypeModelAssignerInterceptor(
        IContentTypeModelAssigner inner,
        ITypeScannerLookup typeScannerLookup)
    {
        _inner = inner;
        _typeScannerLookup = typeScannerLookup;
    }

    public void AssignValues(ContentTypeModel contentTypeModel)
    {
        // we are not changing behavior for this method
        // so just pass-through

        _inner.AssignValues(contentTypeModel);
    }

    public void AssignValuesToPropertyDefinition(
        PropertyDefinitionModel propertyDefinitionModel,
        PropertyInfo property,
        ContentTypeModel parentModel)
    {
        // execute default behavior
        _inner.AssignValuesToPropertyDefinition(propertyDefinitionModel, property, parentModel);

        // check if property is of type `ContentArea`
        // if so - then we are looking for `AllowedTypes` attribute
        if(typeof(ContentArea).IsAssignableFrom(propertyDefinitionModel.Type))
        {
            var allowedAttribute = propertyDefinitionModel
                .Attributes
                .GetSingleAttribute<AllowedTypesAttribute>();

            if (allowedAttribute != null)
            {
                foreach (var allowedType in allowedAttribute.AllowedTypes)
                {
                    // let's do magic only when allowed type is interface
                    if (allowedType.IsInterface)
                    {
                        var concreteClasses = new List<Type>();

                        // we need to find list of classes implementing this interface
                        var foundChildClasses =
                            _typeScannerLookup.AllTypes.Where(t => allowedType.IsAssignableFrom(t));

                        foreach (var foundChildClass in foundChildClasses)
                        {
                            concreteClasses.Add(foundChildClass);
                        }

                        // assign back attribute with added concrete classes
                        allowedAttribute.AllowedTypes =
                            concreteClasses.Concat(allowedAttribute.AllowedTypes).ToArray();
                    }
                }
            }
        }
    }
}
```

After this code being executed in AlloyTech sample site - you can add block instances which are implementing `ISpecificBlock` interface to `SamplePage`'s content area named `MainContentArea`.

Happy content data modeling!

[*eof*]
