---
title: Episerverless = Episerver + Azure Functions
author: valdis
date: 2017-11-21 15:30:00 +0200
categories: [Add-On, Optimizely, Episerver, .NET, C#]
tags: [add-on, optimizely, episerver, .net, c#]
---

I did talk about this topic couple times solo, and also together with [Henrik Fransas](https://world.episerver.com/System/Users-and-profiles/Community-Profile-Card/?userid=9127cdd8-f26f-dc11-8e6c-0018717a8c82) in Episerver Partner Close-up in Stockholm this year. Received lot of questions regarding source code availability and also about recordings of the session(s). Unfortunately there are no recordings of the conferences I/we were. So I decided just compile together a blog post.

## Objective

Idea for the conference talk(s) was to show how you can integrate Azure Functions with Episerver and how that nicely plays together.
There are zillions problems to solve with Azure Functions and also zillions of ways of doing that. However, I've chosen pretty straight forward problem to show how you can leverage cloud services and how to integrate those into Episerver solutions.

*Problem description*: we are building hacker-ish style website where images are not images, but instead converted to [ASCII art](https://en.wikipedia.org/wiki/ASCII_art) green on black background. However challenge is with editors - they are quite lazy and almost always refuse to write description of the images. Also, there has been couple of incidents when editors are uploading inappropriate content images. In these cases - administrator of the website has to be notified (notification transport channel is no important - or may change in the future).

For me, as one of the developers of the website, this seems to be perfect candidate for tryout of the Azure Functions and some other cloud services.

So let's start with laying out components and solution architecture for this fictive website.

## Solution Architecture

There are 2 main players in the solution: website and cloud services. Website will act as consumer of the cloud services. Azure Functions are ideal candidate for delivering this functionality. Also Azure Cognitive Services will be good stuff when it comes to processing and analysis of images. Notification delivery will be done via Twilio service (which by the way has [native support](https://docs.microsoft.com/en-us/azure/azure-functions/functions-bindings-twilio) in Azure Functions).

Following functionality could be split into separate functions (remember that function has to do only single thing and not more).

* *Functionality 1* - this could be as entrance gate for whole functionality. This component will be responsible for retrieving requests for ASCII art convert and image analysis
* *Functionality 2* - this component will be responsible for asking Azure Cognitive Services to analyze image and describe it.
* *Functionality 3* - Third link in the chain would be responsible for actually converting image to ASCII art. If we would convert image to ASCII art and then run vision analysis - most probably results would not be very intelligent.
* *Functionality 4* - Here we could slide in service responsible for analyzing image for inappropriate content (content moderation as they call it).
* *Functionality 5* - if somebody uploaded inadequate image (decision made by Function4) this service would make it possible to send actual notification to the administrator.

Let's try to draw architecture for our imagined solution:

![](/assets/img/2017/11/overall-1.png)

As you can see - inter-function communication is done either via **ServiceBus** or **Storage Queues**. These are most easiest communication channels to use to "stick" together Azure Functions.

ServiceBus topic functionality between function 1 and functions 2 & 4 is necessary because there are multiple "consumers" of the same "signal" - that image has been uploaded for the processing. If using Storage queues there is only single "consumer" of the queue item. Once it's handled - message disappears from the queue - making it impossible for the other function to pick it up and do stuff also. Therefore most appropriate communication mechanism here would be to use durable pub/sub - Azure ServiceBus topics.

For the content moderator cognitive services (to evaluate content) you will need [signup here](https://contentmoderator.cognitive.microsoft.com/).

## Detailed Description

Below is more detailed description of each of the functions and related components.

### Function1 - Accept

First function is acting as facade service for the whole image analysis and processing pipeline. Main purpose (and the only one) for this function is to capture incoming request for the image processing, "register" request and return status code to the caller. Registering request in this context means to create an item in ServiceBus topic. Topic here is required because there will be more than single consumer of the item - Function2 (proceeding with vision analysis of the image) and Function4 (proceeding with content review analysis of the image).

Upload (or call) of the Function1 from the Episerver is straight forward. We subscribe to media upload event in Episerver and then post image `byte[]` to the Azure Function (I know about `ServiceLocator` - I've been active [preaching to avoid this](http://www.episerver.se/events--rapporter/event/evenemangslista/episerver-partner-close-up-2017/epi-partner-close-up-20172/agenda/agenda-utvecklarspar/), but there are cases when it's not entirely possible just because of the underlying framework).

```csharp
[InitializableModule]
[ModuleDependency(typeof(InitializationModule))]
public class EventHandlerInitModule : IInitializableModule
{
    private IAsciiArtUploader _uploader;
    private UrlHelper _urlHelper;

    public void Initialize(InitializationEngine context)
    {
        var canon = ServiceLocator.Current.GetInstance<IContentEvents>();
        _uploader = ServiceLocator.Current.GetInstance<IAsciiArtUploader>();
        _urlHelper = ServiceLocator.Current.GetInstance<UrlHelper>();

        canon.CreatedContent += OnImageCreated;
    }

    public void Uninitialize(InitializationEngine context)
    {
        var canon = ServiceLocator.Current.GetInstance<IContentEvents>();
        canon.CreatedContent -= OnImageCreated;
    }

    private void OnImageCreated(object sender, ContentEventArgs args)
    {
        if (!(args.Content is ImageData img))
            return;

        using (var stream = img.BinaryData.OpenRead())
        {
            var bytes = stream.ReadAllBytes();
            _uploader.Upload(img.ContentGuid.ToString(),
                             bytes,
                             _urlHelper.ContentUrl(img.ContentLink));
        }
    }
}
```

There is also a small helper class that's dealing with physical file upload:

```csharp
internal class AsciiArtUploader : IAsciiArtUploader
{
    private readonly IAsciiArtServiceSettingsProvider _settings;

    public AsciiArtUploader(IAsciiArtServiceSettingsProvider settings)
    {
        _settings = settings;
    }

    public void Upload(string fileId, byte[] bytes, string imageUrl)
    {
        AsyncHelper.RunSync(() => CallFunctionAsync(fileId, bytes, imageUrl));
    }

    private async Task<string> CallFunctionAsync(string contentReference,
        byte[] byteData,
        string imageUrl)
    {
        var req = new ProcessingRequest
        {
            FileId = contentReference,
            Content = byteData,
            Width = 150,
            ImageUrl = imageUrl
        };

        using (var content = new StringContent(JsonConvert.SerializeObject(req)))
        {
            content.Headers.ContentType = new MediaTypeHeaderValue("application/json");
            var response = await
                Global.HttpClient.Value.PostAsync(
                    _settings.Settings.RequestFunctionUri,
                    content).ConfigureAwait(false);

            return await response.Content.ReadAsStringAsync()
                                         .ConfigureAwait(false);
        }
    }
}
```

Class `IAsciiArtServiceSettingsProvider` is just another small helper class that's providing various settings as it's not quite good idea to hard-code URL to the functions or any other "dynamic" part of the required settings for the component.

Also, as Episerver event handler is invoked in a sync method, but we need to invoke image upload via async rest client (`HttpClient`) we need to do some black async magic:

```csharp
/// <summary>
/// Credits:
/// http://stackoverflow.com/questions/9343594/how-to-call-asynchronous-method-from-synchronous-method-in-c
/// </summary>
public static class AsyncHelper
{
    private static readonly TaskFactory MyTaskFactory =
        new TaskFactory(CancellationToken.None,
            TaskCreationOptions.None,
            TaskContinuationOptions.None,
            TaskScheduler.Default);

    public static TResult RunSync<TResult>(Func<Task<TResult>> func)
    {
        return MyTaskFactory.StartNew(func)
            .Unwrap()
            .GetAwaiter()
            .GetResult();
    }

    public static void RunSync(Func<Task> func)
    {
        MyTaskFactory.StartNew(func)
            .Unwrap()
            .GetAwaiter()
            .GetResult();
    }
}
```

**Note!** That you might be using `HttpClient` [in a wrong way](https://aspnetmonsters.com/2016/08/2016-08-27-httpclientwrong/).

Interaction from the Episerver website to function in our architecture diagram is this particular fragment:

![](/assets/img/2017/11/f1-1.png)


Also function definition is not hard to grasp - the only purpose for the function to exist is to register incoming requests for the image processing, save it to the topic and return Http status code with the reference to saved item.

```csharp
[StorageAccount("my-storage-connection")]
[ServiceBusAccount("my-servicebus-connection")]
public static class Function1
{
    [FunctionName("Function1")]
    public static HttpResponseMessage Run(
        [HttpTrigger(AuthorizationLevel.Anonymous, "post")] ProcessingRequest request,
        [Blob("%input-container%/{FileId}")]                CloudBlockBlob outBlob,
                                                            TraceWriter log,
        [ServiceBus("mytopic", AccessRights.Manage)]        out BrokeredMessage topicMessage)
    {
        log.Info("(Fun1) Received image for processing...");

        AsyncHelper.RunSync(async () =>
        {
            await outBlob.UploadFromByteArrayAsync(request.Content,
                                                   0,
                                                   request.Content.Length);
        });

        var analysisRequest = new AnalysisReq
        {
            BlobRef = outBlob.Name,
            Width = request.Width,
            ImageUrl = request.ImageUrl
        };

        topicMessage = new BrokeredMessage(analysisRequest);

        return new HttpResponseMessage(HttpStatusCode.OK)
        {
            Content = new StringContent(outBlob.Name)
        };
    }
}
```

As you can see, creating functions with [Visual Studio Azure Functions](https://blogs.msdn.microsoft.com/webdev/2017/05/10/azure-function-tools-for-visual-studio-2017/) templates might a bit verbose time to time. There has to be a lot of ceremony with `Attribute` classes all over the place, but somehow template engine needs to understand what kind of bindings you have for your functions and generate appropriate `function.json` [file](https://github.com/Azure/azure-webjobs-sdk-script/wiki/function.json) at the end. Of course, there is also an option to create (and maintain) this file manually, but I prefer tooling for that.
Basically function decorations tells following:

* it's `HttpTrigger`ed function - meaning that this function is just sitting there and waiting for incoming requests. For the demo purposes this function is not authorized - so anyone can post anything, but in real-life you should definitely secure your functions [with invoke keys](https://github.com/Azure/azure-webjobs-sdk-script/wiki/Http-Functions#authentication-and-authorization)
* it has connection strings to storage (`[StorageAccount]`) and servicebus (`[ServiceBusAccount]`) accounts
* it also has a reference to Storage Blob container (well, just because image itself might not be able to store in topic item)
* and an output of the function `BrokeredMessage` will be produced (`out` parameter) - this will be stored in ServiceBus topic
* and as result of the function call `HttpResponseMessage` will be returned to the caller


Interesting to note here is `[Blob("%input-container%/{FileId}")] CloudBlockBlob outBlob` argument.
If we look at incoming Http trigger parameter `ProcessingRequest` (body of the request):

```csharp
 public class ProcessingRequest
{
    public ProcessingRequest()
    {
        Width = 100;
    }

    public string FileId { get; set; }
    public string ImageUrl { get; set; }
    public int Width { get; set; }
    public byte[] Content { get; set; }
}
```

From the `Blob` attribute argument you can read is as following:

* `%input-container%` - this is reference to the environment variables. You can reference also values from `settings.json` file as environment variables:

```json
{
  "Values": {
    ...
    "input-container": "in-container",
    ...
  }
}
```

* However `{FileId}` argument is a reference to the property with the same name in incoming request (class `ProcessingRequest`). This technique is very nice and flexible. If you look at how request object was filled up in Episerver - it's content `Guid` of media file in Episerver:

```csharp
private void OnImageCreated(object sender, ContentEventArgs args)
{
    if (!(args.Content is ImageData img))
        return;

    using (var stream = img.BinaryData.OpenRead())
    {
        var bytes = stream.ReadAllBytes();
        _uploader.Upload(img.ContentGuid.ToString(),
                         bytes,
                         _urlHelper.ContentUrl(img.ContentLink));
    }
}
```

In order for this function to operate there has to be a variable with name `my-servicebus-connection` in `settings.json` (or `local.settings.json` file if you are running locally) that's pointing to Azure ServicieBus service. And you will need to create new topic named `mytopic` (or whatever name you choose - just need to match one in decoration of the Function1).

![](/assets/img/2017/11/sb-topic.png)

### Function2 - Analysis

Once incoming request has been accepted and saved in ServiceBus topic - it could processed in parallel - one branch for vision analysis and ASCII art convert, another - for content review.

![](/assets/img/2017/11/f1-f2-f4.png)

Function2 is doing vision analysis of the image using Azure Cognitive Services. I was surprised how easy and elegant consumption of these state-of-art services are:

```csharp
[StorageAccount("my-storage-connection")]
[ServiceBusAccount("my-servicebus-connection")]
public static class Function2
{
    [FunctionName("Function2")]
    [return: Queue("to-ascii-conversion")]
    public static async Task<CloudQueueMessage> Run(
        [ServiceBusTrigger("mytopic", "to-ascii", AccessRights.Manage)] AnalysisReq request,
        [Blob("%input-container%/{BlobRef}", FileAccess.Read)]          Stream inBlob,
                                                                        TraceWriter log)
    {
        log.Info("(Fun2) Running image analysis...");

        var subscriptionKey = ConfigurationManager.AppSettings["cognitive-services-key"];
        var serviceUri = ConfigurationManager.AppSettings["cognitive-services-uri"];
        var client = new VisionServiceClient(subscriptionKey, serviceUri);

        var result = await client.AnalyzeImageAsync(inBlob,
                                                    new[]
                                                    {
                                                        VisualFeature.Categories,
                                                        VisualFeature.Color,
                                                        VisualFeature.Description,
                                                        VisualFeature.Faces,
                                                        VisualFeature.ImageType,
                                                        VisualFeature.Tags
                                                    });

        var asciiArtRequest = new AsciiArtRequest
                              {
                                  BlobRef = request.BlobRef,
                                  Width = request.Width,
                                  Description = string.Join(",", result.Description.Captions.Select(c => c.Text)),
                                  Tags = result.Tags.Select(t => t.Name).ToArray()
                              };

        log.Info("(Fun2) Finished image analysis.");

        return asciiArtRequest.AsQueueItem();
    }
}
```

So what we can read form the function decorations:

* again Function2 needs access to Storage and ServiceBus (to read incoming topic message) services
* trigger of the function is item in the topic (via `[ServiceBusTrigger]` attribute)
* also in the same way as Function1 - it's possible to get reference to Blob block to retrieve actual `byte[]` for the image (with help of `[Blob("...")]` attribute)
* return of the function is item in the Storage queue (`return` value of the function will be used). Decoration `[return: Queue("...")]` tells Visual Studio Azure Functions tooling to generate binding for the return value of the function to post to queue named `to-ascii-conversion` in storage configured under `my-storage-connection` variable (in `settings.json` file). This is will be trigger for Function3.

To work with Azure Cognitive Services you will need to install ProjectOxford package:

```
PM> install-package Microsoft.ProjectOxford.Vision
```

Actual invocation of the Cognitive Services is easy:

```csharp
var result = await client.AnalyzeImageAsync(...);
```

So having captured all available data from Cognitive Services we take those and combine together within next request for the ASCII art. I find this more flexible as next function in the chain is dependent on previous function (to fetch data from somewhere). Data messages are enriched as they flow through the application leaving no artifacts behind, but absorbing all the necessary data.

### Function3 - Convert

When Function2 has finished its product (queue item) is placed in Storage queue upon which next function is triggered. Function3 is responsible for converting image to ASCII art and placing result in next queue (final queue).

![](/assets/img/2017/11/f2-f3.png)

I'll skip actual code for converting to ASCII art (you can actually find lot of interesting implementations on the net), but will post here function itself:

```csharp
[StorageAccount("my-storage-connection")]
public static class Function3
{
    [FunctionName("Function3")]
    [return: Queue("done-images")]
    public static async Task<CloudQueueMessage> Run(

        [QueueTrigger("to-ascii-conversion")]                        AsciiArtRequest request,
        [Blob("%input-container%/{BlobRef}", FileAccess.Read)]       Stream inBlob,
        [Blob("%output-container%/{BlobRef}", FileAccess.Write)]     Stream outBlob,
                                                                     TraceWriter log)
    {
        log.Info("(Fun3) Starting to convert image to ASCII art...");

        var convertedImage = ConvertImageToAscii(inBlob, request.Width);
        await outBlob.WriteAsync(convertedImage, 0, convertedImage.Length);

        var result = new AsciiArtResult(request.BlobRef,
                                        ConfigurationManager.AppSettings["output-container"],
                                        request.Description,
                                        request.Tags);

        log.Info("(Fun3) Finished converting image.");

        return result.AsQueueItem();
    }
}
```

In this case function needs reference to input blob block (actual image that's been uploaded in Episerver and analyzed in Cognitive Services) and also reference to output blob - where image will be stored after ASCII conversion. And also result of the function (`AsciiArtResult` object) will be placed in Storage queue named `done-images`.

## Download to Episerver

Now when image is processed (analyzed and converted) - it's time to download it back to Episerver. There had been discussions in couple of presentations whether Episerver should download the image or function should "push" processing result to Episerver. I picked solution that function is not doing anything - as just writing down result of processing. "Consuming" side (in this case Episerver) is responsible for getting that image and saving in its own blob storage.

So the easiest way in Episerver to do something periodically (to check for new processing results) is by implementing [scheduled jobs](https://world.episerver.com/documentation/Items/Developers-Guide/Episerver-CMS/9/Scheduled-jobs/Scheduled-jobs/).

![](/assets/img/2017/11/f3-job-1.png)

There is nothing interesting to see in scheduled job itself. I don't like to keep lot of my stuff physically inside my scheduled job. I tend to think about it as just hosting/scheduling environment for my service. Also separate (read: service isolated from the underlying framework as much as possible) is a bit easier to test.

```csharp
[ScheduledPlugIn(DisplayName = "Download AsciiArt Images")]
public class DownloadAsciiArtImagesJob : ScheduledJobBase
{
    private readonly IAsciiResponseDownloader _downloader;

    public DownloadAsciiArtImagesJob(IAsciiResponseDownloader downloader)
    {
        _downloader = downloader;
    }

    public override string Execute()
    {
        OnStatusChanged($"Starting execution of {GetType()}");

        var log = new StringBuilder();

        _downloader.Download(log);

        return log.ToString();
    }
}
```

Basically job gets the access to storage queue (again, for demo purposes everyone can access that queue, but in real-life I would recommend to handle this [with SAS tokens](https://docs.microsoft.com/en-us/azure/storage/common/storage-dotnet-shared-access-signature-part-1)) and checks for any new items in the queue. If there is anything - this means that there is new response from ASCII art processing service app.
Almost all the business logic is hidden again in helper services.

Actual downloader of the image:

```csharp
internal class CloudQueueAsciiResponseDownloader : IAsciiResponseDownloader
{
    private readonly IAsciiArtImageProcessor _processor;
    private readonly IAsciiArtServiceSettingsProvider _settingsProvider;

    public CloudQueueAsciiResponseDownloader(
        IAsciiArtServiceSettingsProvider settingsProvider,
        IAsciiArtImageProcessor processor)
    {
        _settingsProvider = settingsProvider;
        _processor = processor;
    }

    public void Download(StringBuilder log)
    {
        var settings = _settingsProvider.Settings;
        var account = CloudStorageAccount.Parse(settings.StorageUrl);
        var queueClient = account.CreateCloudQueueClient();
        var queue = queueClient.GetQueueReference(settings.DoneQueueName);

        var draftMsg = queue.GetMessage();
        if (draftMsg == null)
        {
            log.AppendLine("No messages found in the queue");
            return;
        }

        while (draftMsg != null)
        {
            var message = JsonConvert.DeserializeObject<AsciiArtResult>(draftMsg.AsString);

            log.AppendLine($"Started processing image ({message.BlobRef})...");

            try
            {
                _processor.SaveAsciiArt(account, message);
                queue.DeleteMessage(draftMsg);

                log.AppendLine($"Finished image ({message.BlobRef}).");
            }
            catch (Exception e)
            {
                log.AppendLine($"Error occoured: {e.Message}");
            }

            draftMsg = queue.GetMessage();
        }
    }
}
```

And once response from the image processing function app is retrieved - we need to save meta data about the image and actual ASCII art as well. This is done in response processor:

```csharp
internal class AsciiArtImageProcessor : IAsciiArtImageProcessor
{
    private readonly IContentRepository _repository;

    public AsciiArtImageProcessor(IContentRepository repository)
    {
        _repository = repository;
    }

    public void SaveAsciiArt(CloudStorageAccount account, AsciiArtResult result)
    {
        var blobClient = account.CreateCloudBlobClient();
        var container = blobClient.GetContainerReference(result.Container);
        var asciiBlob = container.GetBlobReference(result.BlobRef);

        var image = _repository.Get<ImageFile>(Guid.Parse(result.BlobRef));
        var writable = image.MakeWritable<ImageFile>();

        using (var stream = new MemoryStream())
        {
            asciiBlob.DownloadToStream(stream);
            var asciiArt = Encoding.UTF8.GetString(stream.ToArray());

            writable.AsciiArt = asciiArt;
            writable.Description = result.Description;
            writable.Tags = string.Join(",", result.Tags);

            _repository.Save(writable, SaveAction.Publish, AccessLevel.NoAccess);
        }
    }
}
```

Now the image vision branch is completed. Let's jump over to content review branch.

### Function4 - Evaluate

For the Function4 to work, you will to signup for the content cognitive services (for now, this is done separately from Cognitive Services setup in Azure). You can signup [here](https://contentmoderator.cognitive.microsoft.com/).


![](/assets/img/2017/11/f2-f4-cogn.png)

I was expecting to have the same .Net client API libraries as for image vision services (`ProjectOxford`). But this is not true for content cognitive services. I'll explain this more in details later in the post (when talking about targeting frameworks issues).

So when somebody uploads image also new message in the topic is emitted. Function4 picks it up (aside from Function2) and starts processing of the content for this image. In particular we are looking for any adult content posted. And if so - administrator should be notified.

```csharp
[StorageAccount("my-storage-connection")]
[ServiceBusAccount("my-servicebus-connection")]
public static class Function4
{
    [FunctionName("Function4")]
    [return: Queue("to-admin-notif")]
    public static async Task<CloudQueueMessage> Run(
        [ServiceBusTrigger("mytopic", "to-eval", AccessRights.Manage)]   AnalysisReq request,
        [Blob("%input-container%/{BlobRef}", FileAccess.Read)]           Stream inBlob,
                                                                         TraceWriter log)
    {
        log.Info("(Fun4) Running image approval analysis...");

        try
        {
            var subscriptionKey =
                ConfigurationManager.AppSettings["cognitive-services-approval-key"];

            var client = new HttpClient();

            // Request headers
            client.DefaultRequestHeaders.Add("Ocp-Apim-Subscription-Key", subscriptionKey);

            // Request parameters
            var uri = ConfigurationManager.AppSettings["cognitive-services-approval-uri"];

            using (var ms = new MemoryStream())
            {
                inBlob.CopyTo(ms);
                var byteArray = ms.ToArray();
                Task<string> result;

                using (var content = new ByteArrayContent(byteArray))
                {
                    content.Headers.ContentType = new MediaTypeHeaderValue("image/jpeg");
                    var response = await client.PostAsync(uri, content);
                    result = response.Content.ReadAsStringAsync();
                }

                if(result.Result != null)
                {
                    var resultAsObject = JsonConvert.DeserializeObject<ContentModeratorResult>(result.Result);

                    if(resultAsObject.IsImageAdultClassified)
                    {
                        log.Warning("(Fun4) Inappropriate content detected. Sending notification...");
                        return request.AsQueueItem();
                    }
                }
            }
        }
        catch (Exception e)
        {
            log.Info("Error on Running image approval..." + e.Message);
        }

        return null;
    }
}
```

Very similar as for the Function2, this Function4 needs access to original image blob (via blob block reference).

Invocation of the content cognitive service is not so nice as for image vision services, I'll cover this a bit later in the post. Basically we are creating new REST request to the content cognitive service endpoint and posting image for the analysis:

```ccsharp
using (var content = new ByteArrayContent(byteArray))
{
    content.Headers.ContentType = new MediaTypeHeaderValue("image/jpeg");
    var response = await client.PostAsync(uri, content);
    result = response.Content.ReadAsStringAsync();
}

if(result.Result != null)
{
    var resultAsObject =
        JsonConvert.DeserializeObject<ContentModeratorResult>(result.Result);

    if(resultAsObject.IsImageAdultClassified)
    {
        log.Warning("(Fun4) Inappropriate content detected. Sending notification...");
        return request.AsQueueItem();
    }
}
```

And back we get simple type describing results (this is just a projection of whole response):

```csharp
public class ContentModeratorResult
{
    public bool IsImageAdultClassified { get; set; }
    public bool IsImageRacyClassified { get; set; }

}
```

As you can see from the decorations of the function - return value of the function will be used as Storage queue item. So if you don't want to put anything in the queue - return `null`.

### Function5 - Notify

And the last but not least, Function5 - responsible for delivering notifications to administrator(s) if inappropriate content detected.

```csharp
[StorageAccount("my-storage-connection")]
public static class Function5
{
    [FunctionName("Function5")]
    [return: TwilioSms(AccountSidSetting = "twilio-account-sid",
                       AuthTokenSetting = "twilio-account-auth-token")]
    public static SMSMessage Run(
        [QueueTrigger("to-admin-notif")]         AnalysisReq request,
                                                 TraceWriter log)
    {
        log.Info("(Fun5) Sending SMS...");

        var baseUrl = ConfigurationManager.AppSettings["base-url"];
        var from = ConfigurationManager.AppSettings["twilio-from-number"];
        var to = ConfigurationManager.AppSettings["twilio-to-number"];

        return new SMSMessage
        {
            From = from,
            To = to,
            Body = $@"Someone uploaded an non appropriated image to your site.
    The image url Id is {request.BlobRef},
    url is {baseUrl + request.ImageUrl}"

        };
    }
}
```

Again for the demo purpose, `to` parameter (actual phone number to send SMS to) is hard-coded in `settings.json` file.

Trigger of the function is Storage queue - however, if there is no item in the queue, this function will not be invoked at all (you will not be charged if running on "consumption pricing plan").

![](/assets/img/2017/11/f4-f5.png)

As you see, there is no interaction with Episerver anymore during content cognitive services branch - if inappropriate content detected - just SMS message is sent.

## Challenge with Target Frameworks

### Upgrade to .Net Standard 2.0

So why there is challenge using various built-in packages for working with strongly typed bindings and other goodies out of the box? Let's try to go through this together.

Imagine that we are building simple Azure Function App using Visual Studio Tools for Azure Functions (latest at the moment of writing - 15.0.30923.

By default with that version of templates `Microsoft.NET.Sdk.Functions` package is referenced and version is set to `1.0.2`. As you might heard, running on .Net Standard in the new thing, so let's try to [migrate there](https://blogs.msdn.microsoft.com/appserviceteam/2017/09/25/develop-azure-functions-on-any-platform/) (as VS Tooling has no support yet). We need to upgrade to `1.0.6` of `Microsoft.NET.Sdk.Functions` package. This is easy. And also need to change target framework in .csproj file to `netstandard2.0`.

I'm not after proper Nuget package restore getting some dependency errors, but oh well.. yeah... I don't know even what that dependency is coming from, why it's needed and most importantly why it's not restoring for .Net Standard.

```
Package 'Microsoft.AspNet.WebApi.Client 5.2.2' was restored using '.NETFramework,Version=v4.6.1'
```

Also I see yellow exclamation mark besides that dependency in solution explorer.

![](/assets/img/2017/11/webapi-restore-1.png)

Function created in sample app is coming AS-IS from VS Tools for Azure Functions template:

```csharp
public static class Function1
{
    [FunctionName("Function1")]
    public static void Run(
        [TimerTrigger("0 */5 * * * *")]    TimerInfo myTimer,
                                           TraceWriter log)
    {
        log.Info($"C# Timer trigger function executed at: {DateTime.Now}");
    }
}
```

Now running this simple function app targeted against .Net Standard, gives another error:

![](/assets/img/2017/11/log-error.png)

Should be easy to fix according to [this GitHub issue](https://github.com/Azure/Azure-Functions/issues/293).
Running again but this time with `ILogger` from `Microsoft.Extensions.Logging`.

```csharp
public static class Function1
{
    [FunctionName("Function1")]
    public static void Run(
        [TimerTrigger("0 */5 * * * *")]    TimerInfo myTimer,
                                           ILogger log)
    {
        log.LogInformation($"C# Timer trigger function executed at: {DateTime.Now}");
    }
}
```

Gives exactly the same error - failing to provide binding value for `ILogger` interface.
There is also some traces on [GitHub repo](https://github.com/Azure/Azure-Functions/issues/293) as well, and couple of issues still open ([#452](https://github.com/Azure/Azure-Functions/issues/452) on Azure-Functions or [#573](https://github.com/Azure/azure-webjobs-sdk-script/issues/573) on azure-webjobs-sdk-script).

Actually it turned out that we need to download new version of Azure Function SDK and set our project to run under that (that was not obvious at first sight).

Action plan here:

* Download Azure Core tools (node):

```
> npm i -g azure-functions-core-tools@core
```

* Set project to execute within new runtime:

![](/assets/img/2017/11/new-runtime-proj-settings.png)

Now we are able to resolve `ILogger` at least.

![](/assets/img/2017/11/runtime-20.png)

However, local run gave no results in console output. I'm not [the only one](https://github.com/serilog/serilog/issues/983).

Ok, for the logging - let's hope it might work at some point.

Now we need to do some Storage and ServiceBus things. With the Storage everything is easy - it's already part of `Microsoft.Azure.WebJobs` package (main package for doing Azure Functions - remember that functions are mode on top of webjobs). On the anther hand - to do a ServiceBus thingies - we need separate package - `Microsoft.Azure.WebJobs.ServiceBus`. For this package last available version is `2.1.0-beta4`. Which however requires `Microsoft.Azure.WebJobs` version `2.1.0-beta4`. Which will give following error if you attempt to install:

```
Package 'Microsoft.Azure.WebJobs.ServiceBus 2.1.0-beta4' was restored using '.NETFramework,Version=v4.6.1' instead of the project target framework '.NETStandard,Version=v2.0'. This package may not be fully compatible with your project.
```

ServiceBus extensions package requires us to fallback to .Net Framework 4.6.1 at least. Ok, also not a problem. We can survive without latest and greatest.

Next thing - we need to talk to Content Cognitive services (preferably via strongly typed client API library and not REST interface). Let's try to [look for this package](https://www.nuget.org/profiles/ProjectOxfordSDK):

![](/assets/img/2017/11/oxford-sdk.png)

If we look for Content Cognitive Services client API library - there is a [unofficial package]()https://www.nuget.org/packages/Microsoft.ProjectOxford.ContentModerator.DotNetCore/) for that. But it's targeting .Net Core (v1.1.1).

Not sure whether .Net Core is supported in Azure Functions runtime at all, but we cannot do that because we are constrained with `net461` target framework because of ServiceBus extension library. We switch back to .Net Framework (also remember to remove project settings not to run under Azure Runtime 2.0).

So I guess we will need to wait a bit still while all the packages will align and .Net Standard 2.0 will be supported everywhere.

## Where to Grab the Stuff?

If you wanna play around with source code - you can [fork one here](https://github.com/valdisiljuconoks/episerverless).

If you wanna see original presentation - you can [grab one here](https://github.com/valdisiljuconoks/episerverless/blob/master/presentation.pptx).

See in action:

<iframe width="800" height="520" src="https://www.youtube.com/embed/dUlbTpRBKvU" frameborder="0" gesture="media" allowfullscreen></iframe>

Happy functioning!

[*eof*]
