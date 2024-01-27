---
title: Building Real-time Public Transport Tracking System on Azure - Part 1
author: valdis
date: 2019-12-08 07:00:00 +0200
categories: [.NET, C#, Azure Functions, Azure, WebJobs, gRPC, SignalR]
tags: [.net, c#, grpc, azure, azure functions, webjobs, grpc, signalr]
---

This is next post in series about real-time public transport tracking system development based on Azure infrastructure and services. This article is a part of Applied Cloud Stories initiative - [aka.ms/applied-cloud-stories](https://aka.ms/applied-cloud-stories).

This is 1st post in series about real-time public transport tracking system development based on Azure infrastructure and services.

Blog posts in this series:

* **Part 1 - Scene Background & Solution Architecture**
* [Part 2 - Data Collectors & Composer](https://tech-fellow.eu/2020/01/21/building-real-time-public-transport-tracking-system-on-azure-part-2-data-collectors-composer/)
* [Part 3 - Event-Based Data Delivery](https://tech-fellow.eu/2020/04/08/building-real-time-public-transport-tracking-system-on-azure-part-3/)
* [Part 4 - Data Processing Pipelines](https://tech-fellow.eu/2020/08/30/building-real-time-public-transport-tracking-system-on-azure-part-4/)
* [Part 5 - Data Broadcast using gRPC and SignalR](https://tech-fellow.eu/2020/11/01/building-real-time-public-transport-tracking-system-on-azure-part-5/)
* [Part 6 - Building Frontend](https://tech-fellow.eu/2020/12/14/building-real-time-public-transport-tracking-system-on-azure-part-6-building-frontend/)
* [Part 7 - Packing Everything Up](https://tech-fellow.eu/2021/01/21/building-real-time-public-transport-tracking-system-on-azure-part-6-packing-everything-up/)
* [Part 7.1 - Fixing Docker Image Metrics in Azure](https://tech-fellow.eu/2021/01/25/building-linux-docker-images-on-windows-machine/)


We are working with transportation company where real-time systems is everyday mindset for each team member. Over few years we have been part of building this system. There were few generations (differences between systems were major refactorings) of the system in this period. Think that we have finally landed on more or less stable implementation. But it's never frozen - system changes all the time as new services and data become available.

We have been following new technologies (like .NET Core) and data distribution mechanisms (like gRPC). Our customer is keen to try out new features and libraries and these technologies are not exclusion.

Let's see how this system can be implemented.

## Architecture

Big picture is quite simple. We have to show data coming from buses to the passengers.

![1](/assets/img/2019/12/1.png)

Picture might look like simple enough, but there are a lot of small moving parts, components and data processing pipelines in the solution. We have to deal with a couple of vendor systems that are used to send data points to centralized aggregator component which is responsible for composing unified transport data snapshot for all running (and also standing still) buses in the region.

Part 2 will focus more on data collection and aggregation process, but from architecture perspective - we built it in the way that new vendors can be plugged-in and obsolete ones can be removed easily. Underlying architecture ideology is based on [pipes and filters architectural style](https://www.enterpriseintegrationpatterns.com/patterns/messaging/PipesAndFilters.html).

We do have many vendors (in picture below showed only vendors named A, B, C, D) to talk to and collect data from. Each of them might be totally different shape and behavior.

![3](/assets/img/2019/12/3.png)

Each vendor supplies data to each processing pipeline where later unified transport data snapshot is produced.
Unified snapshot is then saved in buffer storage. Azure EventGrid subsystem is used to push data further into distribution systems - web application servers to which browsers and other devices are connected to in order to receive real-time updates.

![5.2](/assets/img/2019/11/5.2.png)

Currently we are using [SignalR](https://dotnet.microsoft.com/apps/aspnet/signalr) to provide reliable connection to browser to push data through.

Unfortunately due to underlying platform, we are still running on .NET Framework (we have to host SignalR hub on .NET Framework runtime). Exploring new features is part of the deal, therefore we extracted some of the parts of the system into .NET Standard 2.0 projects and started to build new .NET Core powered gRPC Server.

Specifically we want to change this part and try out new protocols and distribution mechanisms.

![7](/assets/img/2019/11/7.png)

So what we trying now is to add additional webhooks to EventGrid registry to forward events also to gRPC server running in [AKS](https://docs.microsoft.com/en-us/azure/aks/) for additional distribution scenarios.

![6.1](/assets/img/2019/12/6.1.png)

Stay tuned for more in depth posts about each of the subsystems that take part in the big picture!

Happy coding!

[*eof*]
