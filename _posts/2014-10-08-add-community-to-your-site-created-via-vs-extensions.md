---
title: Add Community to your site created via VS Extensions
author: valdis
date: 2014-10-08 09:10:00 +0200
categories: [.NET, C#, Episerver, Optimizely]
tags: [.net, c#, episerver, optimizely]
---

## Set the playground

After you have successfully followed [Ted’s tutorial](http://tedgustaf.com/blog/2014/4/installing-episerver-cms-75/) on how to create a site using almost nothing else than Visual Studio. Next thing what is in your work queue is to add EPiServer Community functionality on top of this.

## Add Community via Deployment Center

One of the way is old-school – via Deployment Center.
First of all you may even be able not to see your site there (regardless that you have added it to IIS).

While surfing of EPiServer source code using surfboard with [Reflector](http://www.red-gate.com/products/dotnet-development/reflector/) sticker on back, I noticed that EPiServer install library that is responsible for enlisting applicable sites in the site tree is checking that there shouldn’t be `EPiServer.Community.dll` file in target app’s `bin` folder and `EPiServer.dll` *has* to be with 7.5 version and major and minor ones respectively.

Which means that in order to enlist your newly created site in EPiServer installation wizard’s site tree – you have to copy over `EPiServer.dll` file from some other application (preferably from app installed via Deployment Center).

After you copied over older version of `EPiServer.dll` file you may receive following error message while installing Community for your site:

![](/assets/img/2014/10/community-err-1-.png)

To be honest, error message looks weird and does tell exactly **nothing**.

Again, browsing installation source code I discovered that this particular error may occur for the sites created via Visual Studio plugin that has this element in web.config file:

```xml
<configuration>
  <episerver>
    <workflowSettings disable="true" />
    ...
```

What is needed for the installation scripts is old style episerver.config file that is mentioned as a source for the `episerver` element.

```xml
<episerver configSource="episerver.config" />
```

So you just have to extract this element into its own file (usually `episerver.config`). And don’t forget to add a namespace for this element (it’s used in installation scripts as well):

```xml
<episerver xmlns="http://EPiServer.Configuration.EPiServerSection">
  <applicationSettings globalErrorHandling="RemoteOnly" ...
```

## Launch the site

Now you can continue installation of the Community via Deployment Center (this will make sure that database structures and procedures and other stuff is added).
 Before you launch the site I noticed that after installation of the Community module `connectionStrings` element from inline listing of the connections was converted to:

```xml
<connectionStrings configSource="connectionStrings.config" />
```

File named `connectionStrings.config` does not exist of course. So be careful – you may need to revert back your connection strings.

After you launched the site you may immediately receive an error (or sometimes when you are accessing AdminUI) that `EPiServer.Shell.UI.dll` version `7.8.0.0` does not exist or could not be found.

All you need to do is to add assembly redirect under `runtime` element:

```xml
<runtime>
<assemblyBinding xmlns="urn:schemas-microsoft-com:asm.v1">
  <dependentAssembly>
    <assemblyIdentity name="EPiServer.Shell.UI" publicKeyToken="8fe83dea738b45b7" culture="neutral" />
    <bindingRedirect oldVersion="0.0.0.0-7.8.0.0" newVersion="7.8.0.0" />
  </dependentAssembly>
```

## Access AdminUI (optional)

After you have successfully installed Community on top of your site created via Visual Studio plugin accessing AdminUI you may receive an error blaming that there is no `LocalSqlServer` connection string:

```
Configuration Error

Description: An error occurred during the processing of a configuration file required to service this request. Please review the specific error details below and modify your configuration file appropriately.

Parser Error Message: The connection name ‘LocalSqlServer’ was not found in the applications configuration or the connection string is empty.

Source File: C:/Windows/Microsoft.NET/Framework64/v4.0.30319/Config/machine.config Line: 252
```

Things you need to do is pretty simple – just add `clear` element in the `profiles/providers` section to clear out any other foreign providers and live just with EPiServer ones.

```xml
<system.web>
  <profile>
    <providers>
      <clear/>
      ...
```

Happy communiting!

[*eof*]
