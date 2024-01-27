---
title: Real-time Transport Tracking System on Azure - Part 7.1 - Fixing Docker Image Metrics in Azure
author: valdis
date: 2021-01-25 23:15:00 +0200
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
* [Part 6 - Building Frontend](https://tech-fellow.eu/2020/12/14/building-real-time-public-transport-tracking-system-on-azure-part-6-building-frontend/)
* [Part 7 - Packing Everything Up](https://tech-fellow.eu/2021/01/21/building-real-time-public-transport-tracking-system-on-azure-part-6-packing-everything-up/)
* **Part 7.1 - Fixing Docker Image Metrics in Azure**

## Background

We finished previous blog post with disadvantage of running Windows based image in ACI by stating that guest OS metrics are not exposed in Azure Portal.

![grpc-aci-no-metrics-2](/assets/img/2021/01/grpc-aci-no-metrics-2.png)

This is even mentioned in [official docs](https://docs.microsoft.com/en-us/azure/container-instances/container-instances-monitor#preview-limitations). Metrics currently are only supported for Linux based images.

So let's fix this by trying to build Linux image on Windows development machine. For this to happen - we gonna need some tools upfront.

## Preparing Build Environment

### Install WSL2

First, we would need something called "Windows Subsystem for Linux (v2)". It's easy installable via `wsl` command if you have joined ["Windows Insiders Program"](https://insider.windows.com/en-us/getting-started) or you can just follow manual installation steps [here](https://docs.microsoft.com/en-us/windows/wsl/install-win10#manual-installation-steps).

Being able to see this screen is thrilling :)

![ubuntu](/assets/img/2021/01/ubuntu.png)

### Installing "Remote WSL" VSCode Extension

After we have installed Linux distro of your choice, we need to install VSCode extension (assuming that you do development in VSCode and know how to install that).

Search for "remote wsl" in extension list.

![remote-wsl](/assets/img/2021/01/remote-wsl.png)

Install it.

### Preparing Docker Engine to Talk to WSL2

You also will need to prepare Docker for Desktop engine to talk to WSL2.
Do following steps:

* Head over to Docker Settings and enable WSL2 integration in "General" tab.

![docker-wsl2](/assets/img/2021/01/docker-wsl2.png)

* Then under "Resources > WSL Integration" make sure that you see your distro and it's enabled.

![docker-wsl2-ubuntu](/assets/img/2021/01/docker-wsl2-ubuntu.png)

## Building Image

### Launching VSCode in WSL Mode

Now to get started:

* Open VS Code
* Open terminal
* Type `wsl`

![wsl-in-vscode](/assets/img/2021/01/wsl-in-vscode.png)

* Then type `code .`

![code-opened-in-wsl](/assets/img/2021/01/code-opened-in-wsl.png)

This basically means that new instance of VSCode is opened in WSL context (pretending that you are actually at Linux `bash`). A bit of mind-blowing, but I can chew that.

### Changing Image Stages

Now we need to change image for the `build` and `final` stages - we need to switch from "Windows" to "Linux".

Which distro you choose that's entirely up to you, but I chose the very basic and straight forward edition - "Focal".

We have to change from

```
FROM mcr.microsoft.com/dotnet/core/sdk:3.1 AS build
...
FROM mcr.microsoft.com/dotnet/core/aspnet:3.1-nanoserver-1809 AS final
...
```

to

```
FROM mcr.microsoft.com/dotnet/sdk:3.1-focal AS build
...
FROM mcr.microsoft.com/dotnet/aspnet:3.1-focal AS final
...
```

List of available SDK images you can get [here](https://hub.docker.com/_/microsoft-dotnet-sdk) and runtimes [here](https://hub.docker.com/_/microsoft-dotnet-aspnet/).


### Build the Image

Building the image inside WSL2 is exactly the same as we saw it already when we were building Docker image inside Windows OS.

Remote WSL makes it super easy. All `docker` commands are working and it feels almost like *magic*.

![vscode-wsl-docker](/assets/img/2021/01/vscode-wsl-docker.png)

## Deploying Image to Azure

Again, the deployment process to Azure is exactly the same as with Windows based images. Just if you would like to override OS type for the ACI - you will have to delete ACI resource before proceeding (as you just can't change OS type on-fly).

After successful deployment and uptime for some period, we now do have nice metrics for the app.

![grpc-metrics-in-portal](/assets/img/2021/01/grpc-metrics-in-portal.png)

Hope this helps!
Happy containerizing!

[*eof*]
