---
title: Configure EPiServer Media Auto Publish on Upload
author: valdis
date: 2016-09-23 12:00:00 +0200
categories: [.NET, C#, Episerver, Optimizely]
tags: [.net, c#, episerver, optimizely]
---

In this post we will enhance media auto-publish option to gain more control over what's going to be automatically published and what will stay for manual publishing.

EPiServer gives site admins option to configure whether media file gets automatically published when uploading to Blob storage.

This is very nice, but the problem is - this setting is *global*. It means either all media files get automatically published, or none - on/off.

![](/assets/img/2016/09/2016-09-22_15-41-31-1.png)

For one of our customers requirement was to delay publish only certain file extensions (usually `.pdf` files with some data that will be valid only from certain date - usually in the future).

1st attempt was to try to see through what's going on during file upload. Browsing through `FileUploadController` type (one is responsible for logic what happens during file upload), I realized that it's not so easy to extend that guy (for instance, constructor demands `ContentService` class and not `IContentService` interface). Which makes overall difficult to intercept, decorate and do something with that dependency.

Another much simpler way is to subscribe to `SavedContent` event and check whether media should be published or not.

## Handling the Event

So let's start with event handler itself. This is pretty straightforward initializable module, that subscribes to `IContentEvent.CreatedContent` event and handles auto-publishing there.

```csharp
[InitializableModule]
[ModuleDependency(typeof(InitializationModule))]
public class MediaAutoPublishHandler : IInitializableModule
{
    public void Initialize(InitializationEngine context)
    {
        var events = context.Locate.Advanced.GetInstance<IContentEvents>();
        events.CreatedContent += OnContentSaved;
    }

    public void Uninitialize(InitializationEngine context)
    {
        var events = context.Locate.Advanced.GetInstance<IContentEvents>();
        events.CreatedContent -= OnContentSaved;
    }

    private void OnContentSaved(object sender, ContentEventArgs args)
    {
        var targetMedia = args.Content as MediaData;
        if(targetMedia == null)
            return;

        TryToPublishWhenContentIsCreated(targetMedia,
                                         ServiceLocator.Current.GetInstance<CurrentProject>(),
                                         ServiceLocator.Current.GetInstance<IContentRepository>());
    }

    private void TryToPublishWhenContentIsCreated
                       (MediaData content,
                        CurrentProject currentProject,
                        IContentRepository contentRepository)
    {
        if(MediaAutoPublishOptions.ShouldAutoPublishCallback == null)
            return;

        // apply logic only to checkout media files
        if(content.Status != VersionStatus.CheckedOut)
            return;

        // skip auto publish in we are in EPiServer project currently
        var isInsideProject = currentProject.ProjectId.HasValue;
        if(isInsideProject)
            return;

        var shouldPublish = MediaAutoPublishOptions.ShouldAutoPublishCallback(content);

        if(shouldPublish)
            contentRepository.Save(content.CreateWritableClone() as IContent,
                                   SaveAction.Publish);
    }
}
```

This was boring part.

It's the handler of the event that's doing all the work.
Magic is in following 2 types.
Auto-publish configuration that just holds a lambda delegate:

```csharp
public static class MediaAutoPublishOptions
{
    public static Func<MediaData, bool> ShouldAutoPublishCallback { get; set; }
}
```

And configuration of the media auto-publisher. This guy will determine whether current content should be published or not.

Why it's lambda and defined in the code? Because here - you can go to configuration file and look for your extensions of the file and maybe auto-publish based on media type (`.png` vs `.pdf` for instance), you can call webservice to check, you can check for time and date and not publish any media on 1st of January, or call grandpapa and ask him. Code opens much more flexibility and extensibility.

```csharp
[InitializableModule]
[ModuleDependency(typeof(InitializationModule))]
public class MediaAutoPublishConfigurator : IInitializableModule
{
    public void Initialize(InitializationEngine context)
    {
        // we would like to publish only images, other media needs manual publish
        MediaAutoPublishOptions.ShouldAutoPublishCallback =
             content => content is ImageData;
    }
    public void Uninitialize(InitializationEngine context) { }
}
```


## See in Action

Ordinary media file upload.

<iframe src="https://player.vimeo.com/video/183892146" width="800" height="468" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe>

No auto-publish on any type if currently editor is inside the project.

<iframe src="https://player.vimeo.com/video/183893166" width="800" height="468" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe>

Happy delayed publishing!

[*eof*]
