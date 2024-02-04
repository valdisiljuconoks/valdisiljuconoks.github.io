---
title: Localizing your React Components with DbLocalizationProvider
author: valdis
date: 2018-06-02 23:15:00 +0200
categories: [Add-On, .NET, C#, ASP.NET, Localization]
tags: [add-on, .net, c#, open source, asp.net, localization]
---

With latest hype around Episerver with no head (aka [headless](https://www.episerver.com/products/features/all-features/headless-api)) - it's becoming more interesting to escape casual back-end stack and see what's happening in other fronts. And it's not surprise that Episerver content could be consumed in other [client-side platforms](https://blog.mathiaskunto.com/2018/03/23/react-and-episerver-moving-to-episerver-headless-episerver-contentdeliveryapi-with-friendly-urls-quick-poc/).
So this time decided to explore a bit more [my colleagues](https://github.com/patkleef) work and look how React is doing nowadays.<br/>
Anyway at some point React component will come to conclusion that there is a need for getting some translations from the server to render localized components. And this just sounds like a great idea to enhance and improve DbLocalizationProvider package.

**Disclaimer!** However this is just one of the possibilities to localize your React components. I believe there are other alternatives out there as well..

## Define Localized Resources

First, we need to define where and how translations will to stored - meaning that there has to be some kind of data container.

```csharp
public class ComponentResources
{
    public SampleComponentResources ComponentTranslations { get; set; }
    ...
}

[LocalizedResource]
public class SampleComponentResources
{
    public static string Property1 { get; set; } = "Some translation";
    public static string Property2 { get; set; } = "Other translation..";
}
```

**Note!** Translation are not defined as expression-bodied properties (`public static string Property1 => "Some translation"`) but as casual properties with setters and getters. We will need this later when we translate object to use on client-side.

## Define Interface on Client-Side

If you are same as me and like strongly-typed access (instead of "stringly"-typed) then you most probably are using TypeScript for your type enforcement on client-side. For this to work as expected, first we need to define interface that will declare what types and properties are available. To make things easier and speed-up (actually to keep server-side and client-side in sync) you can use [TypeLite](http://type.litesolutions.net/) or similar library.

Basically we eventually get interface definition exported similar to this:

```csharp
export interface ComponentResources {
    componentTranslations: SampleComponentResources;
    ...
}

export interface SampleComponentResources {
    property1: string;
    property2: string;
}
```

## Render Translations in Markup
One of the easiest ways to render required resource translations in markup is via special action on let's say `Global` controller:

```razor
@{ Html.RenderAction("ReactComponentTranslations", "Global"); }
```

Action `ReactComponentTranslations` on `Global` controller is quite straight-forward. We use database localization provider *new feature* to translate whole object and not just a property of the resource class. This will give us option to return translated object with all proper translations back to the client-side.

```csharp
private readonly LocalizationProvider _provider;

public GlobalConntroller(LocalizationProvider provider)
{
    _provider = provider;
}

public ActionResult ReactComponentTranslations()
{
    var result = new ComponentResources
    {
        ComponentTranslations = _provider.Translate<SampleComponentResources>();
    };

    var json = JsonConvert.SerializeObject(result, new JsonSerializerSettings
    {
        ContractResolver = new StaticPropertyContractResolver()
    });

    return Content($"<div data-translations='{json}'></div>");
}
```

Code mentioned above will basically emit HTML element with translations from `ComponentResource` class:

```html
<div data-translations='{"componentTranslations": {"property1":"..", ...}}'></div>
```

## Consume Translations in React Component

So now we need to read emitted translations and pass in to our React component. One way would be to do this [via props](https://reactjs.org/docs/components-and-props.html). However, this would require some responsibility of the "middle components" when you would like to consume translations in child of the child components. Thanks to [another colleague](https://github.com/eirhor) who was kind enough and showed me [Context API](https://reactjs.org/docs/context.html) from the React framework. Think this is much nicer and cleaner approach compared to for example Redux state machine.

Ok, let's continue. First, we need to define function that will read translations from the element, this is not hard:

```js
const getTranslations = (): ComponentResources => {
    const element = window.document.querySelector('[data-translations]');
    if (element) {
        return JSON.parse(element.getAttribute('data-translations'));
    }

    return {} as ComponentResources;
};
```

Once we have translations as object in JS, we can create context out of it:

```js
const TranslationsContext = React.createContext(getTranslations());
```

After creation of the context, it's quite easy to start using it in components.
**Note!** That context has set it's default value from the generated element with translations. There are few rules around default values for the contexts. As we would like to use generated translations from the server side and not override them while creating component tree - we will define only consumer of the context, omitting provider (as provide will have to provide concrete values for the context, even passing in `undefined` [will not make](https://reactjs.org/docs/context.html#reactcreatecontext) use of default values).

## Use Translations in your React Component
The essence of getting hold of initialized resource translations is to make use that component is aware of translation context defined above and renders itself "within" context:

```js
export class ComponentContainer extends React.Component {
    constructor(props) {
        super(props);
    }

    render() {
        return(
            <TranslationsContext.Consumer>
            { t =>
                <div>
                    <input type="text"
                           placeholder="{t.componentTranslations.property1}"
                           ... />
                </div>
            }
            </TranslationsContext.Consumer>
        );
    }
}
```

## Conclusion

Generating server-side resource translations and passing them down to the client-side for easier access from the React components is achievable via new [Context API](https://reactjs.org/docs/context.html). You can use at any "depth" in the component tree and you don't need to pollute props definitions to pass down translations from grand to child component. Code was not so terrible actually (even for elder backender as I am).

## Grabbing the Code
As usual DbLocalizationProvider packages are available on [Episerver Nuget feed](https://nuget.episerver.com/package/?id=DbLocalizationProvider.EPiServer).
If you do have any feedback, please leave in comments or ask on [Github](https://github.com/valdisiljuconoks/localization-provider-opti/issues).


Happy localized Reacting!

[*eof*]
