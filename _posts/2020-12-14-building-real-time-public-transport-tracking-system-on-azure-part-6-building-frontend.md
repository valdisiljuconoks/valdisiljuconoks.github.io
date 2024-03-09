---
title: Building Real-time Public Transport Tracking System on Azure - Part 6 - Building Frontend
author: valdis
date: 2020-12-14 13:30:00 +0200
categories: [.NET, C#, gRPC, Azure]
tags: [.net, c#, grpc, azure]
---

This is next post in series about real-time public transport tracking system development based on Azure infrastructure and services. This article is a part of Applied Cloud Stories initiative - [aka.ms/applied-cloud-stories](aka.ms/applied-cloud-stories).

Blog posts in this series:

* [Part 1 - Scene Background & Solution Architecture](https://tech-fellow.eu/2019/12/08/building-real-time-public-transport-tracking-system-on-azure-part1/)
* [Part 2 - Data Collectors & Composer](https://tech-fellow.eu/2020/01/21/building-real-time-public-transport-tracking-system-on-azure-part-2-data-collectors-composer/)
* [Part 3 - Event-Based Data Delivery](https://tech-fellow.eu/2020/04/08/building-real-time-public-transport-tracking-system-on-azure-part-3/)
* [Part 4 - Data Processing Pipelines](https://tech-fellow.eu/2020/08/30/building-real-time-public-transport-tracking-system-on-azure-part-4/)
* [Part 5 - Data Broadcast using gRPC](https://tech-fellow.eu/2020/11/01/building-real-time-public-transport-tracking-system-on-azure-part-5/)
* **Part 6 - Building Frontend**
* [Part 7 - Packing Everything Up](https://tech-fellow.eu/2021/01/21/building-real-time-public-transport-tracking-system-on-azure-part-6-packing-everything-up/)
* [Part 7.1 - Fixing Docker Image Metrics in Azure](https://tech-fellow.eu/2021/01/25/building-linux-docker-images-on-windows-machine/)

## Background

Now that we have received the data, ran through all required processing pipelines, prepared various profiles and have written data out to active (connected clients) gRPC channels, it's time to render snapshot on the screen.

## Solution

For this part we gonna ask for help from my colleague and good friend - **Dzulqarnain Nasir**. So let's head over to his blog series [here](https://dnasir.com/2020/11/11/realtime-web-apps-with-grpc-part-1/).

He is going to fill up space to write about how to build front-end application that uses gRPC channels to talk to the server, offload heavy processing to web workers (to free up CPU for rendering) and other quite cool things. Bookmark and keep an eye on this blog posts!


Happy frontending!

[*eof*]
