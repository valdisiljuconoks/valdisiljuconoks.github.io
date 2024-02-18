---
title: Eliminate Wrapping Element for EPiServer Content Area Item
author: valdis
date: 2014-06-23 18:10:00 +0200
categories: [.NET, C#, Episerver, Optimizely]
tags: [.net, c#, episerver, optimizely]
---

This blog posts describes some new features added to latest [EPiBootstrapArea](https://github.com/valdisiljuconoks/EPiBootstrapArea) plugin library.

## Register New Display Mode from Code
Using `EPiBootstrapArea` plugin it’s possible to rely on some of the automatically generated display modes. Those are covering almost all portions of the area (starting from whole width ending with one-quarter). However not always default fallback modes suited well for various projects.

In latest `1.2` version you may register new display mode by code in EPiServer init module using [Registrar](https://github.com/valdisiljuconoks/EPiBootstrapArea/blob/master/Registrar.cs).

Following code registers new display mode named “The real half” that takes *really* just a half of the area on all device screen sizes.

```csharp
using EPiBootstrapArea;
using EPiServer.Framework;
using EPiServer.Framework.Initialization;
using InitializationModule = EPiServer.Web.InitializationModule;

namespace EPiServer.Templates.Alloy.Business.Initialization
{
    [ModuleDependency(typeof(InitializationModule))]
    public class CustomDisplayModes : IInitializableModule
    {
        public void Initialize(InitializationEngine context)
        {
            Registrar.Register(new DisplayModeFallback
            {
                Name = “The real half”,
                LargeScreenWidth = 6,
                MediumScreenWidth = 6,
                SmallScreenWidth = 6,
                ExtraSmallScreenWidth = 6,
                Tag = “sample-tag”
            });
        }

        public void Uninitialize(InitializationEngine context)
        {
        }

        public void Preload(string[] parameters)
        {
        }
    }
}
```

Order of initialization modules doesn’t matter as eventually both default mode registration process and custom modes registered through [Registrar](https://github.com/valdisiljuconoks/EPiBootstrapArea/blob/master/Registrar.cs) uses display mode `Comparer` to distinguish identity of the mode (it compares all property values). If you will register the same mode twice or more times it will be just skipped.

Unfortunately currently you can’t remove display mode from the code. You have to do it either directly in database or you can use [tool](https://github.com/Geta/DdsAdmin) to manipulate data in `Dynamic Data Store (DDS)`.

## Content Area Item Wrapping Element

Another feature that came in was to provide ability not to render wrapping `div` element if block content is empty. For instance request was based on some sort of block that enlists items. There could be cases when list is empty (for instance no news items returned). In that case block should not render any markup at all. As you may know EPiServer adds wrapping element around block while it’s rendering (good insight of what happens when content is rendered could be found in [article by Joel](http://joelabrahamsson.com/how-episervers-html-helper-propertyfor-works/)).
So eventually I ended up with following “consumer-side” code:

```csharp
[ContentType(DisplayName = “ReallyEmptyBlock”, GUID = “…”)]
public class ReallyEmptyBlock : BlockData, IControlVisibility
{
    public bool HideIfEmpty
    {
        get
        {
            return true;
        }
    }
}
```

Essentially you will need to implement `IControlVisibility` interface that will take care of wrapping element around block content. If block eventually returns something that satisfies `string.IsNullOrEmpty` method (in other words returns completely nothing) no markup will be rendered on content area either.

### Behind the Scene

In order to make this happen I had to override `RenderContentAreaItem` method for content area render.
The tricky part about this one is that base render method is pretty closed when it comes to customization of its behavior and other aspects. If you take a look at where exactly rendered markup is output you can see that it takes current renderer from `HtmlHelper` ():

```csharp
namespace EPiServer.Web.Mvc.Html
{
    public static class IContentDataExtensions
    {
        public static void RenderContentData(this HtmlHelper html, …)
        {
            html.ViewContext.Writer.Write(…)
        }
    }
}
```

This one reminds me of [ServiceLocator anti-pattern](http://blog.ploeh.dk/2010/02/03/ServiceLocatorisanAnti-Pattern/) defined and heavily defended by Mark Seeman. It means that you as child class actor can’t really control final output destination of markup rendered. It would be so much easier if `RenderContetAreaItem` would expect to receive `TextWriter` as one of the parameters for the method.
Anyway base `EPiServer` Api comes AS-IS. The only way how we can trick renderer is to swap `TextWriter` and inject temporary writer at the moment when content of the block is rendered and then inspect and analyze it. If content is empty then just don’t add anything to the original writer and eventually we need to remember to swap it back (in order to continue proper rendering of other items on the content area).

Code fragment that takes care of this is shown below. In this case I’m just using [HtmlAgilityPack](http://htmlagilitypack.codeplex.com/) to inspect content of wrapping element correctly. I know that this needs additional dependency to another package, but you know we are all living in chaos of NuGet dependencies.. :)

```csharp
protected override void RenderContentAreaItem(
                        HtmlHelper htmlHelper,
                        ContentAreaItem contentAreaItem,
                        string templateTag,
                        string htmlTag,
                        string cssClass)
{
    var originalWriter = htmlHelper.ViewContext.Writer;
    var tempWriter = new StringWriter();

    htmlHelper.ViewContext.Writer = tempWriter;
    var content = contentAreaItem.GetContent(ContentRepository);

    try
    {
        base.RenderContentAreaItem(htmlHelper, contentAreaItem, templateTag, htmlTag, cssClass);

        var contentItemContent = tempWriter.ToString();
        var shouldRender = IsInEditMode(htmlHelper);

        if (!shouldRender)
        {
            var doc = new HtmlDocument();
            doc.Load(new StringReader(contentItemContent));
            var blockContentNode = doc.DocumentNode.ChildNodes.FirstOrDefault();

            if (blockContentNode != null)
            {
                shouldRender = !string.IsNullOrEmpty(blockContentNode.InnerHtml);
                if (!shouldRender)
                {
                    // ReSharper disable once SuspiciousTypeConversion.Global
                    var visibilityControlledContent = content as IControlVisibility;
                    shouldRender = (visibilityControlledContent == null)
                                    || (!visibilityControlledContent.HideIfEmpty);
                }
            }
        }

        if (shouldRender)
        {
            originalWriter.Write(contentItemContent);
        }
    }
    finally
    {
        // restore original writer to proceed further with rendering pipeline
        htmlHelper.ViewContext.Writer = originalWriter;
    }
}
```

You can grab it on [EPiServer NuGet](http://nuget.episerver.com/en/OtherPages/Package/?packageId=EPiBootstrapArea).


Happy coding!

[*eof*]
