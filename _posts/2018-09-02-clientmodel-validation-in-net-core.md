---
title: Fixing ClientModel Validation in Asp.Net Core
author: valdis
date: 2018-09-02 00:15:00 +0200
categories: [Add-On, .NET, C#, ASP.NET, Localization]
tags: [add-on, .net, c#, open source, asp.net, localization]
---

## Fixing ClientModel Validation in Asp.Net Core

Story begins with small issue registered in GitHub telling that LocalizationProvider does its job excellent when model is submitted to the server and validated there. Resources are found and used then. But localization provider is not so great when `data-` attributes are generated. Issue described that text from `[Required(ErrorMessage = "...")]` or any other validation attribute was rendered in resulting markup and localization provider was not even involved.

So this view model:

```csharp
[LocalizedModel]
public class UserViewModel
{
    [Display(Name = "User name:")]
    [Required(ErrorMessage = "Name of the user is required!")]
    public string UserName { get; set; }
}
```

Would generate:

```html
<form action="/" method="post" novalidate="novalidate">
    <div>
        <label for="UserName">User name:</label>
        <input name="UserName" id="UserName" type="text" value="" data-val-required="Name of the user is required!" data-val="true">
        <span class="field-validation-valid" data-valmsg-replace="true" data-valmsg-for="UserName"></span>
    </div>
    ...
```

Which looks OK from first sight, but when you change associated resource for required attribute in [AdminUI](https://github.com/valdisiljuconoks/localization-provider-core/blob/master/docs/getting-started-adminui.md?ref=tech-fellow.eu), changed text for that resources is not reflected back to generated markup.

That surprised me and I needed to look inside what's going on when Asp.Net Core Mvc is generating markup and trying to figure out what and how to generate client model validation messages.

## How Built-in Provider is Working

So, when you would like to use default built-in provider to localize models via data annotations attributes (required, string length, etc.) this is done by configuring data annotation localization options (`Startup.cs`):

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddMvc()
            .AddViewLocalization()
            .AddDataAnnotationsLocalization();
}
```

Somewhere along the lines will be registration for `MvcDataAnnotationsLocalizationOptions` type. This type is responsible for holding the reference to a factory method for creating new string localizers (`IStringLocalizer`):

```csharp
public class MvcDataAnnotationsLocalizationOptions
{
    public Func<Type, IStringLocalizerFactory, IStringLocalizer> DataAnnotationLocalizerProvider;
}
```

Simply the following lambda is being registered for this purpose:

```csharp
(modelType, factory) => factory.Create(modelType);
```

Which is essentially just a shortcut for `ResourceManagerStringLocalizer` creation based on given model type. However `ResourceManagerStringLocalizer` is just a wrapper class around `System.Resources.ResourceManager` which is responsible to keep track of available embedded `.resx` files within the assembly and look for the resource based on resource key. More info can be found [here](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/localization?view=aspnetcore-2.1&ref=tech-fellow.eu) so I will not go deeper on this topic.

However, this is just how the localization features are registered within Mvc pipeline and where resources are stored.

Client model validation providers are registered as part of `services.AddMvc()` call usually found in `Startup.cs` and later in `builder.AddViews()` to be more precise, further down in `builder.AddViewServices()`. This type is finally responsible for registered client model validation providers: `Microsoft.AspNetCore.Mvc.ViewFeatures.Internal.MvcViewOptionsSetup`.

![](/assets/img/2018/08/seq.jpg)

List of client model validator providers are stored in `MvcViewOptions.ClientModelValidatorProviders` collection.

There are 3 providers registered by default:

- `DefaultClientModelValidatorProvider` - this is default implementation of the interface and provides validators in model validators metadata
- `DataAnnotationsClientModelValidatorProvider` - data annotation attributes driven validation provider
- `NumericClientModelValidatorProvider` - something special about `float`, `double` and `decimal` validators

We are interested in the middle one.

## What's Wrong with Built-in?

So what's wrong with built-in provider and why I can't just use it out of the box?

When provider is asked to create a validator for [given validator context](https://github.com/aspnet/Mvc/blob/master/src/Microsoft.AspNetCore.Mvc.DataAnnotations/Internal/DataAnnotationsClientModelValidatorProvider.cs?ref=tech-fellow.ghost.io#L52) (when particular model is being validated or validation attributes being generated on client-side) - validator provider is creating `IStringLocalizer` instance based on model type alone:

```
// This will pass first non-null type (either containerType or modelType) to delegate.
// Pass the root model type(container type) if it is non null, else pass the model type.
stringLocalizer = _options.Value.DataAnnotationLocalizerProvider(
                      context.ModelMetadata.ContainerType ?? context.ModelMetadata.ModelType,
                      _stringLocalizerFactory);
```

As you can see - there is no metadata available for which actually property is going to be localized. String localizer is created based on just container type (actual class within which property is validated).

Let's go further - particular adapter is also created based on what kind of data annotation attribute is being validated (`[StringLength]`, `[Required]`, etc):

```
var adapter = _validationAttributeAdapterProvider.GetAttributeAdapter(attribute, stringLocalizer);
```

Again - there is just a information of attribute itself (with no metadata information about property itself on which attribute is being placed) and previously generated string localizer - which is created only based on container type (model class).

And all of the actual data annotation attribute adapters who are responsible for actually generating the error message in case of emergency - has no info about property on which validation attribute is set. All those adapters do is just generate error message based on available metadata:

```csharp
public class StringLengthAttributeAdapter : ...
{
    ...
    public override string GetErrorMessage(ModelValidationContextBase validationContext)
    {
        return GetErrorMessage(
            validationContext.ModelMetadata,
            validationContext.ModelMetadata.GetDisplayName(),
            Attribute.MaximumLength,
            Attribute.MinimumLength);
    }
}
```

which invokes `GetErrorMessage`  method on base class:

```csharp
protected virtual string GetErrorMessage(
    ModelMetadata modelMetadata,
    params object[] arguments)
{
    if (modelMetadata == null)
    {
        throw new ArgumentNullException(nameof(modelMetadata));
    }

    if (_stringLocalizer != null &&
        !string.IsNullOrEmpty(Attribute.ErrorMessage) &&
        string.IsNullOrEmpty(Attribute.ErrorMessageResourceName) &&
        Attribute.ErrorMessageResourceType == null)
    {
        return _stringLocalizer[Attribute.ErrorMessage, arguments];
    }

    return Attribute.FormatErrorMessage(modelMetadata.GetDisplayName());
}
```

Thus as we can see, if you do have data annotation validation attribute with error message set:

```csharp
[LocalizedModel]
public class UserViewModel
{
    [Display(Name = "User name:")]
    [Required(ErrorMessage = "Name of the user is required!")]
    public string UserName { get; set; }
}
```

eventually this will end as call to `IStringLocalizer` looking for resource key:

```csharp
return _stringLocalizer["Name of the user is required!", arguments];
```

Naturally that there is no such a resource with this key.

We have to find another way around to fix this issue.

## Fixing Built-in Stuff in LocalizationProvider Way

In order this shortcoming - we need to dig pretty deep in Mvc pipeline and change couple of things before we can fix localization issue mentioned at the beginning of the post. Latest version of [LocalizationProvider for .Net Core](https://www.nuget.org/packages/LocalizationProvider.AspNetCore/4.3.0?ref=tech-fellow.eu) fixes this issue, so this post is just a recap of things that needed to be changed.

First things first. The most proper location for the fix would be during localization provider initialization:

```csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddDbLocalizationProvider(_ =>
                                           {
                                               ...
                                           });
    }
}
```

During this initialization code we have to add new Mvc View option configurator:

```csharp
services.TryAddEnumerable(ServiceDescriptor.Transient<IConfigureOptions<MvcViewOptions>,
                          ConfigureMvcViews>());
```

Mvc View configurator is responsible now for "injecting" proper client model validator provider with support with more metadata (context in with validation is performed - capturing property name on which validation attribute is decorated).

```csharp
public class ConfigureMvcViews : IConfigureOptions<MvcViewOptions>
{
    private readonly IValidationAttributeAdapterProvider _validationAttributeAdapterProvider;

    public ConfigureMvcViews(IValidationAttributeAdapterProvider validationAttributeAdapterProvider)
    {
        _validationAttributeAdapterProvider = validationAttributeAdapterProvider;
    }

    public void Configure(MvcViewOptions options)
    {
        options.ClientModelValidatorProviders.Insert(
          0,
          new LocalizedClientModelValidator(_validationAttributeAdapterProvider));
    }
}
```

Type `LocalizedClientModelValidator` is now responsible for creating instances of `IStringLocalizer` type with captured proper metadata.

So this is essentially the code that's needed:

```csharp
var attributeAdapter = _validationAttributeAdapterProvider
    .GetAttributeAdapter(validatorMetadata,
                         new ValidationStringLocalizer(type,
                                                       context.ModelMetadata.PropertyName,
                                                       validatorMetadata));
```

We were missing `context.ModelMetadata.PropertyName` fragment.

And once we are aware of actual property being validated - we can now get access to already available helper classes and ask to generate resource key to look for localized resource:

```csharp
public class ValidationStringLocalizer : IStringLocalizer
{
    private readonly Type _containerType;
    private readonly CultureInfo _culture;
    private readonly string _propertyName;
    private readonly ValidationAttribute _validatorMetadata;

    public ValidationStringLocalizer(Type containerType,
                                     string propertyName,
                                     ValidationAttribute validatorMetadata) : ...
{

   ...

   public LocalizedString this[string name]
   {
       get
       {
           return LocalizationProvider.GetString(
                         ResourceKeyBuilder.BuildResourceKey(_containerType,
                                                             _propertyName,
                                                             _validatorMetadata));
       }
   }
}
```

Aaand that's it!

The only piece in whole pipeline was `ModelMetaData.PropertyName` that was missing from `IStringLocalizer` type which was in charge of returning localized validation error messages.

In order to pass in this information to actual localizer we had to change a quite a bit from the pipeline by inserting validation attribute adapter provider in client model validation provider collection and re-implementing the rest of the pipeline under that type.

However, fixing this bug gave me more insights and understanding about Asp.Net Core Mvc internals and view configuration and model validation in particular, meaning that I do have now a bit more info to help others.

Happy localizing .Net Core web apps!

[*eof*]
