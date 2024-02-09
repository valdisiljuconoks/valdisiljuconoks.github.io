---
title: Story behind custom error handler development
author: valdis
date: 2011-11-27 23:15:00 +0200
categories: [.NET, C#, Add-On, Episerver, Optimizely]
tags: [.net, c#, add-on, episerver, optimizely]
---

One of the first task on my brand new path in EPiServer world was to develop custom error pages. Nothing special and no big deal at looking at the task from 10k feat.

We decided to take a Http module path – to plug in our module in request processing pipeline to inspect the situation on the server and make all the stuff needed to render the error page for appropriate HTTP status code.

Enabling custom error handler is as easy as adding following lines to your web.config file:


```xml
<add name="CustomErrorHandlerModule" type="Geta.ErrorHandler.ErrorHandlerModule, Geta.ErrorHandler" />
```

The first trick was about Http module and HttpApplication Error event. This event is raised only when there is a unhandled exception on the server by the Asp.Net runtime. The trick is to get back to the server processing pipeline even end-user requested resource that is not really associated with Asp.Net (for instance, user is requesting http://<server>/<app>/documenthatdoesnotexist.doc).

What we can do about this is to instruct IIS to execute some .aspx file for 404 Http status code. Trick here is that this file should not exist on the server therefore our module will be called and application will raise error blaming that our provided IIS 404 error displaying page does not exist.


```xml
<httpErrors errorMode="Custom">
  <remove statusCode="401" subStatusCode="-1" />
  <error statusCode="401" path="/CustomErrorHandler.aspx" prefixLanguageFilePath="" responseMode="ExecuteURL" />
```

By calling `HttpContext.Current.Server.GetLastError().GetBaseException()` we can get back to the original error occurred in “previous” request. I quoted “previous” request because request has not been completed and response is not served yet (because of Http error “ExecuteURL” mode).

If there are some issues with IIS and custom error pages (usually issue is that IIS is not allow use custom error pages) look at feature delegation configuration item in IIS server root. Search for “Error pages” and verify that it’s set to “Read/Write”.

After we get back on track we can investigate an exception occurred previously and make assumption on what status code to set for the response and which page to render.

Of course before rendering error page some checks should be made: for detecting infinite loops, request for some well known resources, etc.



So what the module is doing actually later when decision on which page to render is made? Module has following logic:

* It looks for `“Error{0}PageReference”` property of the site start page. Where place holder {0} is left for Http status code value (`“Error404PageReference”`, `“Error500PageReference”`, etc).
* If administrator or editor has not set page reference module is trying to locate error display page using convention: “error page should be named in name of Http error and located under root folder”. So for instance 404 error happens on the server, so module is trying to locate `/404` page (using friendly Url).
* If this fails again, module tries to render `“/{0}.htm”` page which is static Html fallback if everything else fails.
* The last step is to allow IIS to handle the exception by itself. This means that there is the possibility that module is enabled but no pages exists to render in user friendly fashion – IIS can show yellow screen of death or something unwanted information to the end-user.


Lastly if the EPiServer kind page was found (step 1. or 2.) Trick was to render it correctly.

We ended with the following implementation:


```csharp
private void Transfer(PageReference reference, int httpErrorCode)
{
    var page = DataFactory.Instance.GetPage(reference, new LanguageSelector(ContentLanguage.PreferredCulture.ToString())) ??
               DataFactory.Instance.GetPage(reference);

    if(httpErrorCode == 500)
    {
        this.context.Server.TransferRequest(page.LinkURL);
    }
    else
    {
        this.context.Server.Execute(page.LinkURL);
    }
}
```

More info in insights of `Server.TransferRequest()` and alternatives can be found here. We figured out that `Server.Execute()` does not work very well for 500 status pages. And somehow EPiServer is loosing preferred UI culture/language if method to render page is other than `Response.Redirect()`. Looking at `EPiServer.PageBase` class `InitializeCulture()` method suspicious line seemed to be:


```csharp
ILanguageSelectionSource languageSelectionSource = this.CurrentPageHandler as ILanguageSelectionSource;
```

Haven’t so much time to investigate all this.



Module as NuGet package is available at – [nuget.episerver.com](https://nuget.episerver.com). Id: **Geta.ErrorHandler**.

Package also contains:

* Some generic static Html error pages (for 401, 403, 404 and 500 statuses). Some very basic styling. Don’t blame, I’m not a web designer :)
* Custom page type (used PageTypeBuilder) that can be used to store view model for error pages. By default page type looks for “/Templates/Pages/GetaErrorHandlerPage.aspx” file. Contains just heading and body of the page.
* Base page to be used for custom error pages(Geta.ErrorHandler.BaseErrorPage<T>). Custom base page should be used to get Server.TransferRequest() and Server.Execute() work properly. We discovered that ContextMenu.OptionFlag should be turned off for target error page. Not yet figured out why. TransferRequest() seems to be very fragile and magic method to use. Avoid that.


It uses log4net as internal logging transport according to EPiServer approach.



What it currently does not support is:

* We do not support HTTP status subcodes. That’s most probably is coming soon.
* Appropriate support for AJAX requests. Should be given nice JSON object back to the caller indicating that something wrong happened on the server, so clients scripts can act appropriately.

Hope this helps!

[*eof*]
