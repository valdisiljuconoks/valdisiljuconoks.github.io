---
title: Year in Review
author: valdis
date: 2015-12-30 23:30:00 +0200
categories: [Year Review]
tags: [.net, c#, open source, localization, year review]
---

This is perfect timing for a busy people like us in IT industry to stop by and look back on the path that has been walked and some good things accomplished.

Below is a short summary of things that I hope you will find useful and beneficial in your own way. This is an experience had this year within my lovely industry, company I'm working in and with people around me.

## Github Move

I was using `git` and GitHub particularly as open source project repository quite a while. Once you will step over the peek of the learning curve, command-line interface and `git` command will become your new comfort zone, you may become a true fan of distributed source control system. This is great especially in open source projects where a lot of people may be involved and development and implementation of the project's new features may happen in parallel or in even more tracks at the same time.

I thought this is great for open source project and couldn't image to use it everyday in enterprise scenarios - when your company's main source code repo in under `git` control. This turned out to be truly amazing journey and enjoyed its every inch.

## OpenSource libraries

By reviewing GitHub activity over last years - I can see the pattern that most productive and active period happens during winter months. Who loves hacking while beach or [inline](https://en.wikipedia.org/wiki/Aggressive_inline_skating) is calling you out? Is it related to daylight, when it's over already at 4:00 PM? Not sure, but anyway - I find my self hacking again something new together to help fellow developers during the evenings and nights when darkness already covered our street and you are folded into stillness.

![](/assets/img/2015/12/2015-12-28_23-05-11.png)

This is good way to give you an overview of stuff I've been working on lately (over a year already) in order for you to decide whether this is something valuable and useful for you and projects you are working on.

These modules, libraries and frameworks have seen daylight only thanks to my library's consumers, company customers, project's contributors and my personal life motivators.

I would like to **thank you all!**

### FeatureSwitch

Looking at usage, downloads, questions and shown interest - in my opinion this is one of the most successful projects I've been busy with.

I was set on the project where a lot of `if/else` statements were typed just to check if this piece of code, that brings some value or adds some functionality to the project, is accessible and executable by the user, batch job and any other consumer of that code fragment. At that point an idea was born about more automatic and easier way to check these features and enable or disable those depending on outer environment, triggers or some arbitrary piece of code that tells library if feature is accessible in this context. Over the years - FeatureSwitch library grew into separate modules and adapters for various purposes. Most common used is plugin controlling feature states from the [EPiServer](http://nuget.episerver.com/en/OtherPages/Package/?packageId=FeatureSwitch.EPiServer) administration UI. Even if you are diagnosing your web site using [Glimpse](http://getglimpse.com/) platform, you can now turn on/off your features directly from [Glimpse Tab]().

There is still a lot of featurs to implement on my roadmap, but I'm always open to your ideas.

More info: [here](https://github.com/valdisiljuconoks/FeatureSwitch).

### ImageResizer.Plugins.EPiServerBlobReader

Optimizing images for your site and thinking about mobile users is important topic nowadays. One of the most popular library for working with images in web is [ImageResizer](http://imageresizing.net/).

You need to hack some pieces together to tell ImageResizer how to work with EPiServer Blob reader in order to serve images from that storage. Martin Pickering has done brilliant job and stick together [required parts](http://world.episerver.com/Code/Martin-Pickering/ImageResizingNet-integration-for-CMS75/) for the EPiServer.

I was really sick and tired of making the same stuff all over again and again - copying code from episever.com and adding to different projects. It was enough and just decided to take a duct tape, and wrap it up into a [NuGet package](http://nuget.episerver.com/en/OtherPages/Package/?packageId=ImageResizer.Plugins.EPiServerBlobReader).

More info: [here](https://github.com/valdisiljuconoks/ImageResizer.Plugins.EPiServerBlobReader).

### Twitter Bootstrap aware EPiServer ContentArea

Another smaller, but still interesting project is about ContentArea and [Twitter Bootstrap](http://getbootstrap.com/). I would dare to agree that Bootstrap may be the world's most popular mobile-first and responsive front-end framework.

![](/assets/img/2015/12/2015-12-30_22-13-41.png)

EPiServer demo site ([AlloyTech](http://world.episerver.com/download/Demo-Sites/Alloy-Technologies/)) at some point has support for Bootstrap grid system to control the width of the blocks and the content on the content area. The lacking feature was to control the fallback on different form factors (screen sizes).

I had some working proof of concept stuff in few projects. Eventually - what turned out is a [NuGet package](https://nuget.episerver.com/en/OtherPages/Package/?packageId=EPiBootstrapArea) that gives you possibility to control display options for every content type on the ContentArea.

![](/assets/img/2015/12/2015-12-28_23-55-34.png)

More info: [here](https://github.com/valdisiljuconoks/EPiBootstrapArea)

### Scheduled Jobs Overview

Once I noticed an pretty interesting question on EPiServer forums where somebody were asking - what about scheduled jobs sort order in Admin menu. Once they are sorted in one way, next time you check Admin UI - scheduled jobs may be sorted the other way. Hard to find job you are looking for if the list is long enough.

This may become a bit annoying when you are dealing with EPiServer project where scheduled job count may exceed Content Type count :)

In the same EPiServer project we had pretty important scheduling times for our jobs, list of jobs hardly fitted on single page and there were no easy way to double-check whether job is enabled, was the last run cycle successful and when is next scheduled execution time. These were regular complaints from DevOps.

Decided to create some plugin that may display all these values on single page, giving DevOps an executive overview of all scheduled jobs, their statuses and other important information.

![](/assets/img/2015/12/4-1-.png)

ScheduledJobsOverview project was created pretty early in my EPiServer career and was one of the first projects were I tried to maintain the code base for ordinary [NuGet package](http://nuget.episerver.com/en/OtherPages/Package/?packageId=TechFellow.ScheduledJobOverview) and [EPiServer AddOn](http://world.episerver.com/add-ons-page/) at the same time. That made sense for versions where editor (actually packing admin) may install modules themselves. Nowadays - adding a package within developer workflow makes perfect sense. So much hustle back in times when at the moment of deployment you need to deal with changed environment by somebody else.

Kudos goes to [Mathias Kunto](http://world.episerver.com/System/Users-and-profiles/Community-Profile-Card/?userid=effcb213-0e04-de11-b9c4-0018717a8c82) for giving an idea on how to augment existing EPiServer Admin UI page and add your own stuff using WebForms Page adapters. Brilliant idea.

More info: [here](https://github.com/valdisiljuconoks/TechFellow.ScheduledJobOverview)

### Asp.Net Mvc Areas for EPiServer

I'm a big fan of structuring and organizing source code. It was awesome to see that EPiServer reference architecture site for the Commerce part ([Quicksilver](https://github.com/episerver/Quicksilver)) introduced so called feature folders. Where they organize code and other assets related to that particular area under single folder. This makes so much easier to reason about code, structure and architecture of the solution.
I was missing only one thing - views or content type renderer templates were located under root folder - where default view engine is looking for. This still felt not so clean and feature folders were implemented only partially.

One of the potential way how to organize code in your web projects - is to use Mvc areas as functionality discriminators.

After a long research and series of experiments (that included several [blog posts](https://tech-fellow.ghost.io/tag/look-inside/) from the "insight into EPiServer" series) came to conclusion that Mvc area could be used as grouping element to bundle related functionality together and also as discriminator between various EPiServer sites. Mvc area as discriminator for EPiServer sites is an interesting and easy way how you may organize your project code, content types and other supporting code for particular site.


Eventually [NuGet package](http://nuget.episerver.com/en/OtherPages/Package/?packageId=MvcAreasForEPiServer) was rolled out to be beneficial for other as well.

More info: [here](https://github.com/valdisiljuconoks/MvcAreasForEPiServer)

### Initialization Modules

Sounds familiar? Well, I was involved in project where EPiServer was not an option, we were building pure Asp.Net Mvc application. Our startup code had some dependencies and also we pulled in some 3rd party libraries that had their own initialization approach. We tried to keep this code simple and modular, but needed to fulfill execution order requirements.

That reminded me EPiServer initialization engine kicking off at the very beginning of the app life cycle. EPiServer served as great piece of inspiration while creating this library.

Package represents my attempt to resolve this requirement and come up with something that others may find useful in their projects as well.

More info: [here](https://github.com/valdisiljuconoks/InitializableModule)

### Form Multipart Media Formatter for WebApi

And last but not the least..

Have you tried to upload file from your favorite front-end framework to Asp.Net WebApi? Sounds simple, but turned out not to be. Missing part in WebApi architecture was multipart form media binder that should be responsible for understanding incoming request and bind values properly and create [HttpPostedFile](https://msdn.microsoft.com/en-us/library/system.web.httppostedfile(v=vs.110).aspx) collection or similar data structure that represents list of files uploaded with incoming request.

Fortunately guy from Ukraine ([Alexander Kozlovskiy](https://github.com/iLexDev)) made this happen. My 2 cents were just to pack it as [NuGet package](https://www.nuget.org/packages/MultipartDataMediaFormatter/) and use it.


### Payback for being an Open Sourcer

The case here is interesting. The other day I saw an advertisement on Facebook promoting WebSummit conference and urged you to submit [GitHub account link](https://github.com/valdisiljuconoks) to enter contest to find free tickets to next summit in [Lisbon in 2016](https://websummit.net/). I'm not a big fan of such activities as I never won anywhere. Well, I haven't played hard either..

Anyway, receiving mail like this makes me smile:

![](/assets/img/2015/12/2015-12-28_23-20-39.png)

Read more about [Open Source contributors](https://websummit.net/open-source-contributors) contest to get free tickets to next for the WebSummit.

## EPiServer Conference

Close to the end of the year, EPiServer held an amazing conference in Vegas. Interesting and challenging for me this conference was because it was first time when I had a chance to polish speaking skills in other continent. It was really a fast forward through EPiServer Commerce content in 80 minutes counting in workshop tasks and questions and answers. Anyway - feedback from the audience was quite positive which makes me think that next time I should focus more on some deep dive advanced topic. This is the subject [John Sonmez](http://simpleprogrammer.com/) is talking in his book - you need to find your niche, your area you are passion about. Debugging..?! That's something I'm in.

![](/assets/img/2015/12/emvp-summit-2015-1-.jpg)
*Photo: Allan Thraen / Marija Jemuovic Delibasic*

## Books Read

It's pretty hard to imagine professional career without reading the books. Most probably there will be technical and non-technical books in your list. We have to keep balance and not be geeky all the time.

Just to keep list short, would like to share my latest discoveries (first was gifted by an great colleague of mine - [Alexander Haneng](https://twitter.com/ahaneng)). Sorted by influence on my career, professional and personal life.

* [How to win friends and influence people](http://www.amazon.com/How-Win-Friends-Influence-People/dp/0671027034) (by Dale Carnegie)
* [Soft Skills: The Software Developer's Life Manual](https://www.manning.com/books/soft-skills) (by John Z. Sonmez)
* [Software Architecture for Developers](https://leanpub.com/software-architecture-for-developers) (by Simon Brown)
* [Antifragile Software](https://leanpub.com/antifragilesoftware) (by Russ Miles)

These books accompanied me on the plane, in the train and while waiting in the queue. My black Kindle is always with me and becomes best buddy in these situations.

Dale Carnegie changed the way I'm dealing with people. I could quote the whole book here, but to be short, the most influential principle from the book that I'm taking with me everywhere:

> Don't criticize anyone.

[John Sonmez](http://simpleprogrammer.com/) gives you an elegant overview of your profession questioning you all the time - who would you like to become? What's your goal and how you are going to achieve it? Giving you descriptions of various topics that software developer may wonder about in his/her career. The most interesting, important and influential question you need to ask:

> How you are going to sell yourself?

## Socializing

People skills is an interesting topic. How to deal and work together with them. All the time we are giving with an option to make friends, interact with others and be open.

Conferences is a great possibility to find confederates and friends. Make use of it and enjoy.

Another interesting way to "socialize" is be active in the public, in community you are belonging to. My case is great Microsoft and EPiServer community. Great developers and amazon people. Through these channels, being a *Most Valuable Professional* (**MVP**) both - in [Microsoft](https://mvp.microsoft.com/en-us/mvp/Valdis%20Iljuconoks-4024626) and in [EPiServer](http://world.episerver.com/Articles/Items/EMVP-Announcement/) - opens doors to interesting connections and activities.

![](/assets/img/2015/12/2015-12-30_22-50-11.png)

Helping to your community could be done in various ways. My way is to help developers, help them to solve problems, overcome difficulties and achieve greater goals. Most easiest way is to be involved on forums, mentor and help other fellows to solve issues they are facing while working on projects. Be aware that may lead to some "complaints" as well :)

![](/assets/img/2015/12/2015-12-30_14-58-49.png)

## \#crapmanship

These days it's hard to stick to only single area of interest - whether it's front-end, backend, or something in between. Modern software craftsmanship practitioner runs through all the layers, learns all required tools, observes and understands best practices and patterns. It's just natural when you are becoming a [full-stack developer](https://en.wikipedia.org/wiki/Full-stack_developer) or someone who knows what tool suits best in particular situation.

However recent years show that customers require more and more value to be delivered almost by the same time and money. Leaving only the last vertex of the software project's [triangle](https://en.wikipedia.org/wiki/Project_management_triangle) flexible - **quality**.

Usually "quality" of the first variable that is cut from software development project management equation. This brings up the question - what are we creating in the first place? Is it a craftsmanship or crapmanship we are practicing here?

During the year more and more I was pulled in topics when it comes to estimates. Either it was over energized discussions about [\#noestimates](https://twitter.com/search?q=%23noestimates) in some conference coffee breaks or just a fact you face in the industry.

Anyhow I can not find the source for this quote, but while reading its origin - it just burned in my brain cells:

> If there is no probability to finish 30% ahead of the schedule - the schedule is goal, not an estimate.

It's a hash tag `#crapmanship` I'll be using to indicate some "interesting" cases while drifting through professional career and stumble upon this topic.


## What's next?

There is plenty of things I would like to do to in the community and help other fellows to be more productive, help customers and partners.

First project that is scheduled for release in early January is something about EPiServer localization. Hope that will make lot of things easier, more productive and less annoying. Stay tuned!

Following John Sonmez recommendations I'm still searching for my niche in this industry and still considering to really deep dive into .Net application debugging, and especially EPiServer projects debugging. I know that it's not common topic among EPiServer developers as most of the hardcore part has been done by the platform, but still - find it interesting and challenging for me and hope helpful for others as well. So will see. Some blog posts are in draft state already.


EPiServer Commerce is on the radar. Really hope and dive into this platform and become comfortable to talk about really advanced stuff there and main goal is to help out community on EPiServer World forums.


So all in all, I'm confident that 2016 year will bring us more interesting and challenging tasks to complete, .. and goals to achieve.


As usual - happy coding and Happy New Year! :)

[*eof*]
