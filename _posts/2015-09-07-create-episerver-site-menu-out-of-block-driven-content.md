---
title: Creating EPiServer Site Menu out of Block Driven Content
author: valdis
date: 2015-09-07 08:30:00 +0200
categories: [.NET, C#, Episerver, Optimizely]
tags: [.net, c#, episerver, optimizely]
---

For one of our great customers we needed to make site out of just only few pages and lots of blocks. It reminds sort of single page application, just there is no client-side behavior as original SPA projects have.
Anyway, in the middle of the project we came to task to make site menu. As we know it's quite easy to build site menu out of the pages and subpages and children pages of these subpages. But this time, we had almost no page structure in CMS, but instead, we have lots of `ContentArea` items. So we decided to play around CA items and try to build site's menu out of these items.

## Site Structure

This is a prototype of project we had for the customer.

![](/assets/img/2015/09/2015-09-06_21-23-59.png)

Almost everything was driven by blocks and there were just few pages types only.

## Menu Structure

It of course depends on precise project's requirements, but site menu may look something like this:

```xml
<ul>
    <li>Menu Item 1
        <ul>
            <li><a href="...">Submenu item 1</a></li>
            <li><a href="...">Submenu item 2</a></li>
            <li><a href="...">Submenu item 3</a></li>
        </ul>
    </li>
    <li>Menu Item 2
        ...
    </li>
    ...
</ul>
```

Visually it should look like this:

![](/assets/img/2015/09/2015-09-06_21-38-19-1.png)

You see that we will need to generate links directly to blocks on the page. For instance `Page2` should be not generated in menu at all.

## Preparing content types to be in menu

### Adjusting Page Types

We need add special "marker" interface to the pages which should be part of the menu and those content should be as menu items to which visitor can navigate to.
Adding this interface and marking necessary page types with it - menu generator will be able to find where exactly content of the page is located and then proceed with submenu item generation out of the blocks.

```csharp
public interface IHaveContent
{
    ContentArea MainContentArea { get; set; }
}

public class ArticlePageWithContent : PageBase, IHaveContent
{
    [CultureSpecific]
    [Display(Name = "Content",
        GroupName = SystemTabNames.Content,
        Order = 10)]
    public virtual ContentArea MainContentArea { get; set; }
}
```

### Adjusting Block Types

While it's easy to make decision whether page should be part of the menu (based on its `DisplayInMenu` attribute that is easily available for the editors), we need to add something similar for the blocks that will be placed inside Content Area - to filter out blocks that we do not need while generating menu content.

For this reason we can create "marker" interface - that will help us to distinguish between and understand which blocks we need to filter away.

```csharp
public interface IMenuItem
{
    bool ShowInMenu { get; set; }
    string DisplayNameInMenu { get; set; }
}
```

Later those block types that need to be part of the menu - can implement this interface, so menu generator can filter be these.

```csharp
[ContentType(DisplayName = "EditorialBlock", GUID = "...")]
public class EditorialBlock : BlockDataBase, IMenuItem
{
    [CultureSpecific]
    [Display(Name = "Show in menu",
        GroupName = SystemTabNames.Content,
        Order = 10)]
    public virtual bool ShowInMenu { get; set; }

    [CultureSpecific]
    [Display(Name = "Display name in menu",
        GroupName = SystemTabNames.Content,
        Order = 20)]
    public virtual string DisplayNameInMenu { get; set; }

    ...
}
```

However - this interface is to restricted to block types only. Menu item generation also applies (and works) for the page types that are partially rendered through `ContentArea`.

## Generating the Menu

### Menu View model

Next we need to produce view-model for the menu to generate markup from. This is straight forward:

```csharp
public class MenuItem
{

    public MenuItem(string name)
    {
        Items = new List<SubMenuItem>();
        DisplayName = name;
    }

    public string DisplayName { get; set; }
    public List<SubMenuItem> Items { get; internal set; }
}

public class SubMenuItem
{
    public SubMenuItem(string blockId, string displayName, string targetPageAddress)
    {
        DisplayName = displayName;
        MenuLink = targetPageAddress + "#" + blockId;
    }

    public string DisplayName { get; set; }
    public string MenuLink { get; set; }
}
```

Don't worry - we will get back to the bookmark field in just a second.

### Menu Controller

This is personal preference, but we usually extract common rendering (like menu) into its own controller (or merge together with other "common" part of the site) and let Asp.Net Mvc to invoke it correctly.

In your `_SiteLayout.cshtml` file you may write something similar:

```csharp
...
<body>
    @Html.Action("Menu", "Common)
    ...
```

Which in turn will invoke this action from `CommonController.cs`:

```csharp
public class CommonController : Controller
{
    [ChildActionOnly]
    public ActionResult Menu()
    {
        ...
    }
}
```

This will be the place where we will put our menu generation code.

### Get list of pages to show in menu

First of all we need to get list of pages as 1st level menu items:

```csharp
[ChildActionOnly]
public ActionResult Menu()
{
    var menuItems = new List<MenuItem>();
    var filter = new FilterContentForVisitor();

    var pages = _loader.GetChildren<IContent>(ContentReference.StartPage).ToList();
    filter.Filter(pages);
    var filteredPages = pages.Cast<PageData>().Where(p => p.VisibleInMenu);

    foreach (var page in filteredPages.OfType<IHaveContent>())
    {
        ....
    }
```

Then we need to access page's content area and generate submenu items out of those blocks (code is long enough as it's a bit defensive):

```csharp
foreach (var page in filteredPages.OfType<IHaveContent>())
{
    var contentArea = page.MainContentArea;
    if (contentArea == null || contentArea.FilteredItems == null || !contentArea.FilteredItems.Any())
    {
        continue;
    }

    var pageData = page as PageData;
    if (pageData == null)
    {
        continue;
    }

    var menuItem = new MenuItem(pageData.Name);

    foreach (var block in contentArea.FilteredItems.Select(b => _loader.Get<IContent>(b.ContentLink)).OfType<IMenuItem>())
    {
        if (!block.ShowInMenu)
        {
            continue;
        }

        var blockInstance = block as IContent;

        if (blockInstance == null)
        {
            continue;
        }

        var pageAddress = _resolver.GetUrl(pageData.ContentLink);

        var subMenuItem = new SubMenuItem(blockInstance.GetContentBookmarkName(),
                                          !string.IsNullOrEmpty(block.DisplayNameInMenu)
                                              ? block.DisplayNameInMenu
                                              : blockInstance.Name,
                                          pageAddress);

        menuItem.Items.Add(subMenuItem);
    }

    menuItems.Add(menuItem);
}
```

Basically code fragment above will look for all blocks inside our known `ContentArea`, will filter out only those blocks that expressed willingness to be part of the menu, and out of these blocks new `SubMenuItem` object instances are added to the Menu's `ViewModel`.

### Generating the Markup

Next what we need is actual markup that will be used to render the menu. I'm not a web designer and this is the best what I could come up with :)

```razor
@model List<DynamicMenu.Controllers.MenuItem>

<div>
    Menu:

    <div>
        <ul>
            @foreach(var menuItem in Model)
            {
                <li>
                    @menuItem.DisplayName
                    @if(menuItem.Items.Any())
                    {
                        <ul>
                            @foreach(var subMenuItem in menuItem.Items)
                            {
                                <li><a href="@subMenuItem.MenuLink">@subMenuItem.DisplayName</a></li>
                            }
                        </ul>
                    }
                </li>
            }
        </ul>
    </div>
</div>
```

### Bookmarking the content

So far so good. We now have view-model that contains data about the menu, we got controller that fills in view-model, we have markup that is rendering the menu.
Next thing that we need to do is to make sure that visitors will be able to navigate to particular block on particular page.
For this to work we need to make sure that links in menu are generated with bookmarks.
Let's start from beginning. First, we need to fill in view-model data with submenu items containing some sort of bookmark name (Html bookmark links starts with `#` sign followed by name of the bookmark).
For this, I introduced extensions method for the `IContent`:

```csharp
public static class ContentExtensions
{
    public static string GetContentBookmarkName(this IContent content)
    {
        return content.GetOriginalType().Name.ToLowerInvariant()
               + "_"
               + content.ContentLink;
    }
}
```

This extension method generates bookmark name similar to `"{name of the block type}_{id of the block}"`, e.g. `"editorialblock_556"`. This combination should ensure uniquality of the bookmarks across the page.

So submenu item markup may look like this:

```xml
<li><a href="/targetpage#editorialblock_556">Block 1</a></li>
```

Next (most interesting part for me), we need to somehow make EPiServer to generate these unique bookmark names exactly at the DOM location where block starts to render its content.
I couldn't find more ideal candidate for this ar `ContentAreaRenderer` (you can read more about what exactly is customizable in ContentArea rendering pipeline [in my post](https://tech-fellow.eu/2015/06/11/content-area-under-the-hood-part-3/)).

So one of the possibility is to completely override `ContentAreaRenderer` and add `id` attribute for the block tag element.
Another way (as most of our projects use Bootstrap), I used [EPiBootstrapArea](http://nuget.episerver.com/en/OtherPages/Package/?packageId=EPiBootstrapArea) library and extended `ContentAreaRenderer` used there:

```csharp
[ModuleDependency(typeof (SwapRendererInitModule))]
[InitializableModule]
public class SwapBootstrapRendererInitModule : IConfigurableModule
{
    public void ConfigureContainer(ServiceConfigurationContext context)
    {
        context.Container.Configure(container => container
                                        .For<ContentAreaRenderer>()
                                        .Use<AnotherBootstrapAwareContentAreaRenderer>());
    }

    public void Initialize(InitializationEngine context) {}

    public void Uninitialize(InitializationEngine context) {}
}

public class AnotherBootstrapAwareContentAreaRenderer : BootstrapAwareContentAreaRenderer
{
    public AnotherBootstrapAwareContentAreaRenderer()
    {
        SetElementStartTagRenderCallback(GenerateIdAtBlockElement);
    }

    private void GenerateIdAtBlockElement(HtmlNode blockElement, ContentAreaItem contentAreaItem, IContent content)
    {
        blockElement.Attributes.Add("id", content.GetContentBookmarkName());
    }
}
```

Method `SetElementStartTagRenderCallback` gives me possibility to hook inside `ContentAreaItem` rendering process and do some magic with element DOM object (`HtmlNode` is coming from [HtmlAgilityPack](https://www.nuget.org/packages/HtmlAgilityPack) library).

So eventually what we will get is `ContentAreaItem` starting with following markup:

```html
<div class="..." id="editorialblock_556" ...>
    ...
```

Which means that we now can connect menu item link with block on the page through this auto-generated bookmark name.

## Summary

To generate site menu out of content driven mostly by blocks you will need:

* to prepare page and block types to take part in menu generation process (use some "marker" interfaces)
* find pages that will be added to the menu
* filter out content (list of `IContent` from which particular page consists of)
* also use another "marker" interface to extract data out of content required for the menu (like, menu item display name)
* make sure that you generate links in the menu with bookmark that will be later generated into `ContentAreaItem` starting tag element
* adjust `ContentAreaRenderer` to include block's bookmark name as `id` attribute of the block's start tag element;

Happy site menu'ing!

[*eof*]
