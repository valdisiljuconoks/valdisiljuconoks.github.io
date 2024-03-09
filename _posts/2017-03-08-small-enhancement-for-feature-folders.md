---
title: Small Enhancement for Feature Folders
author: valdis
date: 2017-03-08 15:45:00 +0200
categories: [Add-On, .NET, C#, Architecture]
tags: [add-on, .net, c#, architecture]
---

Great colleague of mine [Māris Krivtežs](https://github.com/marisks) wrote [blog post](http://marisks.net/2017/02/03/razor-view-engine-for-feature-folders/) about feature folders describing code organization into folders by features or functional areas rather than technology or different grouping.

However, one of cons from Maris' approach we couldn't reconcile with - name of the sub-feature folder needs to follow Mvc controller naming convention.

> "Another issue is related to the sub-feature folder naming. Sub-feature folder still should be called with **the same** name **as a controller.**"

So I made tiny adjustment to the view engine worth sharing. Adjusted code is below.

```csharp
public class SiteViewEngine : RazorViewEngine
{
    /*
     Placeholders:
     *      {2} - Name of the Mvc area
     *      {1} - Name of the controller
     *      {0} - Name of the action (name of the partial view)
     */

    public SiteViewEngine()
    {
        var featureFolders = new []
        {
            "~/Features/{0}.cshtml",
            "~/Features/{1}{0}.cshtml",
            "~/Features/{1}/{0}.cshtml",
            "~/Features/{1}/Views/{0}.cshtml",
            "~/Features/{1}/Views/{1}.cshtml",
        }
        .Union(SubFeatureFolders("~/Features"))
        .ToArray();

        ViewLocationFormats = ViewLocationFormats
                              .Union(featureFolders)
                              .ToArray();

        PartialViewLocationFormats = PartialViewLocationFormats
                                     .Union(featureFolders)
                                     .ToArray();
    }

    private IEnumerable<string> SubFeatureFolders(string rootFolder)
    {
        var rootPath = HostingEnvironment.MapPath(rootFolder);
        if(rootPath == null)
            return Enumerable.Empty<string>();

        var featureFolders = Directory.GetDirectories(rootPath)
                                      .Select(GetDirectory);

        var features = featureFolders.Select(a => new
                        {
                            a.Name,
                            Features = Directory.GetDirectories(a.FullName)
                                                .Select(GetDirectoryName)
                        });

        return features.SelectMany(feature =>
                {
                    return new[]
                    {
                        $"{rootFolder}/{feature.Name}/{{0}}.cshtml",
                        $"{rootFolder}/{feature.Name}/{{1}}{{0}}.cshtml",
                        $"{rootFolder}/{feature.Name}/Views/{{1}}{{0}}.cshtml",
                        $"{rootFolder}/{feature.Name}/Views/{{1}}/{{0}}.cshtml"
                    }
                    .Union(
                       feature.Features
                              .SelectMany(subFfeature => new[]
                              {
                                $"{rootFolder}/{feature.Name}/{subFfeature}/{{0}}.cshtml",
                                $"{rootFolder}/{feature.Name}/{subFfeature}/{{1}}{{0}}.cshtml",
                                $"{rootFolder}/{feature.Name}/{subFfeature}/Views/{{1}}/{{0}}.cshtml",
                                $"{rootFolder}/{feature.Name}/{subFfeature}/Views/{{1}}{{0}}.cshtml"
                              }));
                });
    }

    private string GetDirectoryName(string path)
    {
        return GetDirectory(path).Name;
    }

    private DirectoryInfo GetDirectory(string path)
    {
        return new DirectoryInfo(path);
    }
}
```

Basically I got rid of `{1}` placeholder in sub-feature segment, and instead just scanning folder structure and registering view path locations dynamically depending on folder structure there.

**NB!** All of the disadvantages and cons from Maris' post still applies (like view name uniqueness, Visual Studio complaints and others).

Below you can see various combinations of possible conventions and feature folder organizations in Visual Studio (that might give you more clear overview of possibilities):

![](/assets/img/2017/03/ff.png)

I'll revisit [Mvc Areas for EPiServer](https://github.com/valdisiljuconoks/MvcAreasForEPiServer) package once again, and will write about how to use these feature folders together with Mvc Areas to get even more flexibility in your projects.

Happy slicing!

[*eof*]
