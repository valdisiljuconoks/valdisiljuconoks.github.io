---
title: How-to Pack Your Shell Module for Optimizely Content Cloud
author: valdis
date: 2021-09-03 19:00:00 +0200
categories: [Add-On, Optimizely, Episerver, .NET, C#, Open Source]
tags: [add-on, optimizely, episerver, .net, c#, open source]
---

Optimizely Content Cloud is approaching fast and partners are working on getting modules out on the feed for the rest of us to test.

As you know - Optimizely vNext is targeting .NET 5 and with this fundamental platform change - there are few changes also how packaging of the module's assets should be done.

This blog post should cover the most common tasks for you to pack properly Optimizely Content Cloud module.

Thanks to [Mark](https://world.optimizely.com/System/Users-and-profiles/Community-Profile-Card/?userId=c739f674-8f6f-e111-968f-0050568d002c) and [Māris](https://world.optimizely.com/System/Users-and-profiles/Community-Profile-Card/?userId=e6d9e7b4-1069-4217-81b4-1b2346a0150a) for initial version and ideas around how to pack our stuff.

If your module has Views - story is simpler there. More details [here](https://github.com/episerver/netcore-preview#compiled-views-for-shell-modules).

## Todo List
During module packaging process there are few things that we need to do:

* copy `module.config` any client-side assets to temporary directory (assets usually go into directory named after package version);
* if your module has client-side assets - you have to ensure that `module.config` file points to correct path (including version number) e.g.:

```xml
<module
        loadFromBin="false"
        clientResourceRelativePath="7.0.0-pre-0002"
        ...
```

* pack everything up into `.zip` file

**NB!** Even if your module does not have any client-side assets - you still need to include `module.config` file in `.zip` file. Here is [a tracking issue](https://github.com/episerver/netcore-preview/issues/17) (to install module without `module.config` file). And install module provider sounds hackish anyway :)

* include `.zip` file in target NuGet package under specific paths.

So let's get started.

## The Pack Script
### 0. Create Build File and Include in Project
To get things unified and reusable - let's define `.proj` file that we will include in `.csproj` file - for the project that requires to be packed.

Let's name file `pack.proj` and place it in `src/` folder.

Now we can include that one in `.csproj` file:

```
<Import Project="$(SolutionDir)\src\pack.proj" Condition="Exists('$(SolutionDir)\src\pack.proj')" />
```

### 1. Copy Stuff To Temp Directory
First we need to define what is temp directory

```xml
<PropertyGroup>
    <TmpOutDir>$(SolutionDir)\tmp</TmpOutDir>
</PropertyGroup>
```

Then we have to script how stuff is copied over to temp directory. This target will be called always after successful `Build` target invocation - basically when project is built.

This script assumes that your Optimizely client-side assets are in `module\` folder in project structure.

```xml
  <Target Name="CreateZip" AfterTargets="Build">

    <MakeDir Directories="$(TmpOutDir)\content\$(Version)" />

    <ItemGroup>
      <ClientResources Include="$(SolutionDir)\src\$(MSBuildProjectName)\module\ClientResources\**\*" />
    </ItemGroup>

    <Copy SourceFiles="@(ClientResources)" DestinationFiles="@(ClientResources -> '$(TmpOutDir)\content\$(Version)\ClientResources\%(RecursiveDir)%(Filename)%(Extension)')" />

    <Copy SourceFiles="$(SolutionDir)\src\$(MSBuildProjectName)\module\module.config" DestinationFolder="$(TmpOutDir)\content" />

  </Target>
```

Notes:

* we define `ClientResources` variable pointing to all files under `module/ClientResources` folder;
* copy over those in `tmp/content/{version}/`
* copy over also `module.config` file un `tmp/content/` folder;

### 2. Set Client Resource Relative Path
Before we pack them up - we need to set correct client-side asset relative path in `module.config` file. This could be done by `XmlPoke` task:

```xml
<XmlPoke
    XmlInputPath="$(TmpOutDir)\content\module.config"
    Query="/module/@clientResourceRelativePath"
    Value="$(Version)" />
```

### 3. Zip It Up
Now we are ready to zip it up and remove temp folder.

```xml
<UsingTask TaskName="ZipDirectory" TaskFactory="RoslynCodeTaskFactory" AssemblyFile="$(MSBuildToolsPath)\Microsoft.Build.Tasks.Core.dll">
  <ParameterGroup>
    <InputPath ParameterType="System.String" Required="true" />
    <OutputFileName ParameterType="System.String" Required="true" />
    <OverwriteExistingFile ParameterType="System.Boolean" Required="false" />
  </ParameterGroup>
  <Task>
    <Using Namespace="System.IO" />
    <Using Namespace="System.IO.Compression" />
    <Code Type="Fragment" Language="cs">
      <![CDATA[
        if(this.OverwriteExistingFile) {
          File.Delete(this.OutputFileName);
        }

        ZipFile.CreateFromDirectory(this.InputPath, this.OutputFileName);
      ]]>
    </Code>
  </Task>
</UsingTask>

<ZipDirectory
  InputPath="$(TmpOutDir)\content"
  OutputFileName="$(OutDir)\$(MSBuildProjectName).zip"
  OverwriteExistingFile="true" />
 <RemoveDir Directories="$(TmpOutDir)" />
```

`$(OutDir)` - points to target folder where project is being packed.

### 4. Include .zip File in NuGet Package
In order to include `.zip` file in NuGet package we have to specify that file is part of the package and also specify its location within the package folder tree system. This is accomplished with help of `Pack` and `PackagePath` properties defined for the `Content`.

```xml
<Target Name="CreateZip" AfterTargets="Build">
  <MakeDir Directories="$(TmpOutDir)\content\$(Version)" />
  <ItemGroup>
    <Content Include="$(OutDir)\$(MSBuildProjectName).zip" >
    <Pack>true</Pack>
    <PackagePath>content\modules\_protected\$(MSBuildProjectName)\;contentFiles\any\net5.0\modules\_protected\$(MSBuildProjectName)\</PackagePath>
</Content>
```

**NB!** Shell `.zip` files must be present in two locations:

* under `content\modules\_protected\..`
* and under `contentFiles\any\net5.0\modules\_protected\..`

## Copy Module Files On Host Project Build
Here "host" project is consuming project - one where the package has been installed.

There is a catch. When you install NuGet package and want to include some files in the host project - files are added as "shortcuts":

![added-module-shortcut](/assets/img/2021/09/added-module-shortcut.png)

Checking file properties you can see that files are added (as shortcut) from NuGet cache folder: `C:\Users\{user}\.nuget\packages\{module}\{version}\contentFiles\any\net5.0\modules\_protected\{module}\{module}.zip`

Obviously this file is required to be present on the disk under project folder. We need additional file to get this file copied. We have to create `CopyZipFiles.targets` file. This file hook into different host project events and perform some actions. This time - we will copy over file *before* `Build` target.

```xml
<?xml version="1.0" encoding="utf-8"?>
<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003" ToolsVersion="4.0">
  <ItemGroup>
    <SourceScripts
      Include="$(MSBuildThisFileDirectory)..\..\contentFiles\any\net5.0\modules\_protected\**\*.zip"/>
  </ItemGroup>

  <Target Name="CopyZipFiles" BeforeTargets="Build">
    <Copy
      SourceFiles="@(SourceScripts)"
      DestinationFolder="$(MSBuildProjectDirectory)\modules\_protected\%(RecursiveDir)"
/>
  </Target>
</Project>
```

Here we basically take all files in NuGet package `contentFiles\any\net5.0\modules\_protected\` folder and copy to host project file system.

For the package to be able to execute some actions "inside" host project context - this target file also needs to be included in NuGet package with specific name (same as package name) and under specific folder (`build`).

To achieve this - we can use `Pack` and `PackagePath` settings:

```xml
<ItemGroup>
  <Content Include="$(SolutionDir)\src\$(MSBuildProjectName)\CopyZipFiles.targets" >
    <Pack>true</Pack>
    <PackagePath>build\net5.0\$(MSBuildProjectName).targets</PackagePath>
  </Content>
</ItemGroup>
```

## Resulting NuGet Package
Packaging process is exactly the same as for any other .NET project:

```
> dotnet pack
```

At the end NuGet package should look something like this:

![package](/assets/img/2021/09/package.png)

I hope you will use other package ID as shown above :) otherwise we might collide..

## Sample Working Project
Localization Provider beta version for upcoming Optimizely version is using this approach. Here is reference to [GitHub repo](https://github.com/valdisiljuconoks/localization-provider-epi/tree/main/src/DbLocalizationProvider.AdminUI.EPiServer).

## Full File Versions
### The Pack Script
Here is full pack script file for completeness:

```xml
<?xml version="1.0" encoding="utf-8"?>

<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003" ToolsVersion="15.0">

  <UsingTask TaskName="ZipDirectory" TaskFactory="RoslynCodeTaskFactory" AssemblyFile="$(MSBuildToolsPath)\Microsoft.Build.Tasks.Core.dll">
    <ParameterGroup>
      <InputPath ParameterType="System.String" Required="true" />
      <OutputFileName ParameterType="System.String" Required="true" />
      <OverwriteExistingFile ParameterType="System.Boolean" Required="false" />
    </ParameterGroup>
    <Task>
      <Using Namespace="System.IO" />
      <Using Namespace="System.IO.Compression" />
      <Code Type="Fragment" Language="cs">
        <![CDATA[
          if(this.OverwriteExistingFile) {
            File.Delete(this.OutputFileName);
          }
          ZipFile.CreateFromDirectory(this.InputPath, this.OutputFileName);
        ]]>
      </Code>
    </Task>
  </UsingTask>

  <PropertyGroup>
    <SolutionDir Condition="$(SolutionDir) == ''">$(MSBuildProjectDirectory)\..\</SolutionDir>
    <TmpOutDir>$(SolutionDir)\tmp</TmpOutDir>
  </PropertyGroup>

  <Target Name="CreateZip" AfterTargets="Build">
    <MakeDir Directories="$(TmpOutDir)\content\$(Version)" />
    <ItemGroup>
      <ClientResources Include="$(SolutionDir)\src\$(MSBuildProjectName)\module\ClientResources\**\*" />
    </ItemGroup>
    <Copy SourceFiles="$(SolutionDir)\src\$(MSBuildProjectName)\module\module.config" DestinationFolder="$(TmpOutDir)\content" />
    <Copy SourceFiles="@(ClientResources)" DestinationFiles="@(ClientResources -> '$(TmpOutDir)\content\$(Version)\ClientResources\%(RecursiveDir)%(Filename)%(Extension)')" />
    <XmlPoke XmlInputPath="$(TmpOutDir)\content\module.config" Query="/module/@clientResourceRelativePath" Value="$(Version)" />
    <ZipDirectory
      InputPath="$(TmpOutDir)\content"
      OutputFileName="$(OutDir)\$(MSBuildProjectName).zip"
      OverwriteExistingFile="true" />

    <!-- <RemoveDir Directories="$(TmpOutDir)" /> -->
  </Target>

  <ItemGroup>
    <Content Include="$(OutDir)\$(MSBuildProjectName).zip" >
      <Pack>true</Pack>
      <PackagePath>content\modules\_protected\$(MSBuildProjectName)\;contentFiles\any\net5.0\modules\_protected\$(MSBuildProjectName)\</PackagePath>
    </Content>
    <Content Include="$(SolutionDir)\src\$(MSBuildProjectName)\CopyZipFiles.targets" >
      <Pack>true</Pack>
      <PackagePath>build\net5.0\$(MSBuildProjectName).targets</PackagePath>
    </Content>
  </ItemGroup>
</Project>
```

### Copy Zip File Script

```xml
<?xml version="1.0" encoding="utf-8"?>
<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003" ToolsVersion="4.0">
  <ItemGroup>
    <SourceScripts
      Include="$(MSBuildThisFileDirectory)..\..\contentFiles\any\net5.0\modules\_protected\**\*.zip"/>
  </ItemGroup>

  <Target Name="CopyZipFiles" BeforeTargets="Build">
    <Copy
      SourceFiles="@(SourceScripts)"
      DestinationFolder="$(MSBuildProjectDirectory)\modules\_protected\%(RecursiveDir)"
/>
  </Target>
</Project>
```

Happy packaging!
[*eof*]
