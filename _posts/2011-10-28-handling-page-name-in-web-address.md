---
title: Handling page name in web address
author: valdis
date: 2011-11-28 23:35:00 +0200
categories: [.NET, C#, Episerver, Optimizely]
tags: [.net, c#, episerver, optimizely]
---

EPiServer provides a great feature to copy and paste pages in site structure and reuse already existing content or features of particular page as a basis for the new page. However during copy page name gets set to the same name as source page has and assigned some index to the name. So for instance if we take Alloy Tech sample site copying and later paste “Alloy Track” page to the same branch in the site structure new page will get page name in address “Alloy-Track1”.

![](/assets/img/2011/10/1.png)

This problem also applies to the process of rename of the page. Web address remains old one.

However fortunately there is another great feature in EPiServer – called “Rebuild Name for Web Addresses” which iterates over all pages and looks for these kind of pages and tries to fix them. If I leave name of the new page the same as source page then there would be two identical pages in the same branch – then of course rebuilding names would not fix that. If I rename the page to reflect new content name for the web address still will remain the same as assigned during copying it. Then rebuild will fix that and page web address will be renamed.

However there is a problem if this rebuild takes part after some time when site has been published in the air and users have been using the site for a while. Bookmarks, favorites, etc will be broken.

EPiServer module `Geta.ErrorHandler` latest version provides possibility to preserve these previous addresses for pages during rebuild and guarantee that users will be redirected to the new address even they will be asking for the previous one.

Module actually is too far from the complex implementation.

EPiServer provides incredible way to plug some custom code in almost any processing pipeline or listen to any event occurring in the system.

By inheriting from `EPiServer.PlugIn.PlugInAttribute` you can do everything you need to hook somewhere in static `Start()` method.

```csharp
public class PageUrlRenameHandler : PlugInAttribute
{
    public static void Start()
    {
        DataFactory.Instance.PublishingPage += OnPublishingPage;
    }
}
```

So to catch pages that are changed during page name rebuild we subscribe to publishing event.

During publishing event handler we can get access to old name and new name of the page.

This will give old address of the page.


```
e.Page.LinkURL
```

So by using friendly URL rewriter module we can get friendly URL for the page.


```
Global.UrlRewriteProvider.ConvertToExternal(oldUrl, e.Page.PageLink, Encoding.UTF8)
```

This will give new name of the page in web address (e is parameter PageEventArgs in event handler method).


```
e.Page.URLSegment
```

So using old URL and new segment we can make a decision either page is just saved and no web address rebuild has been invoked (if old and new address match) or page is being rebuilt (old and new URL segments do not match).

If page is being rebuilt then we just iterate over all language branches and using PageData.LinkURL property get old URL address and write it down that that particular page.

We are using DDS entity with following attributes to store set of old URLs attached to that particular page.

![](/assets/img/2011/10/2.png)

We are storing all previous URLs for particular page in appropriate language branch.

**NB!** The trick in rebuilding process is that EPiServer is touching *only* that particular page that has to be renamed. Child pages remain untouched.

For instance suppose we have following page structure:

![](/assets/img/2011/10/3.png)

And web address for page “1.2” would be `“/Parent-page/First-child/12”`.

If we rename `“Parent page”` to `“New parent page”` then EPiServer rebuild process will republish only this particular page leaving all child pages untouched.

We need to take care of rebuilding all child pages as well to write old URL for each of them – so the old web address for “1.3” page should still start with `“/Parent-page/..”`.

Therefore we just process all child pages in all language branches recursively.

Rename the top level page in a large site could influence page rebuild process performance a lot. That’s the down side of this approach.



Fortunately this feature can be turned off and `Geta.ErrorHandler` could be used to perform its original task – handling not found and other server side errors in friendly fashion.

Previous URL redirect feature can be disabled in `“Plug-In Manager”` page in admin mode.

![](/assets/img/2011/10/4.png)

Hope this helps!

Available via nuget.episerver.com

[*eof*]
