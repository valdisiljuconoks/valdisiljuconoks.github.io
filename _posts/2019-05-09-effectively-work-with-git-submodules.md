---
title: Effectively Working with Git Submodules
author: valdis
date: 2019-05-09 07:00:00 +0200
categories: [.NET, C#, Git]
tags: [.net, c#, git]
---

## Background

During development of `DbLocalizationProvider` I had single repository in GitHub containing  more that one package as result of the build. Initially there was just a EPiServer package to add support for database driven localization resources. Later realized that there is actually not so much to do to add support for Asp.Net Mvc (.NET Framework) applications and later also for .NET Core applications.

This results into:

* packages for EPiServer applications
* packages for Asp.Net Mvc applications
* packages for Asp.Net Core applications
* abstract/core packages containing only general purpose functionality (like expression tree walker or resource definition attributes)

![submod-0](/assets/img/2019/05/submod-0.png)

As you can see there might be some issue with having multiple purpose packages (with different life-cycles and versions) located in single Git repository.
So decided to split whole code-base into git submodules and setup separate repositories for each of the sub-systems. This is a blog post about the stuff I had to do.

## Draw Module Boundaries

It's important to understand where each module ends and where next begins as you be referencing each other through sort of NuGet package references. Decision has to be made around type locations - where each type should go and which project will be used where.
Actually for development purposes using ordinary project reference is much more preferred way to work with. This gives nicer debugger experience for developer without any hustle to enable symbols and be able to "step into" the package source code.

## Create Repositories with Submodules

### Create Main Module Repo

Now when you have decided module boundaries it's time to create repositories for each of the module. You start with top-level module repo which does not have any external dependencies on any other module (core/common functionality).

We start in the following order:

* `main` package repository (core/common functionality)
* `aspnet` (repository for Asp.Net Mvc applications, has dependency on `main` module)
* `epi` (repository for EPiServer integration, has dependency on `main` and `aspnet`)
* `netcore` (.NET Core repository, has dependency on `main`)

We create repository for localization provider core/common module.

```
$ git init main
```

After whole code-base has been moved in, we can push it out to GitHub.

### Create Repo With Submodule

After we have `main` module repository available, we can create new one for Asp.Net Mvc (`aspnet`) packages.

```
$ git init aspnet
```

Now tricky part is that I would like to include source from  `main` repository as part of the solution - like normal projects. Then I would be able to debug it through, change and adjust on-demand if needed, etc.
We need to "*include*" projects from `main` repository into `aspnet` repository. This is achievable by "linking" `main` repository content into `aspnet`. Keep in mind that linking is possible by pointing to some directory under which linked repository content will be "cloned".

```
$ git submodule add {main-repo-url} lib/localization-provider
```

This will link `main` package current HEAD to the `aspnet` repository under `lib/localization-provider` resulting content of the `main` repository becoming as part of (submodule) `aspnet` repository.

Later we can repeat the same exercise for EPiServer repository as well.

Information about just added submodule is written in `.gitmodules` file (in root folder of parent repository).

```
[submodule "lib/localization-provider"]
    path = lib/localization-provider
    url = https://github.com/valdisiljuconoks/LocalizationProvider.git
```

### Detached HEAD

**NB!** This is very important to remember that submodule linking sets included submodule into *detached HEAD* state.

```
$ cd lib/localization-provider
$ git status
HEAD detached at 8d6dc87
nothing to commit, working tree clean
```

Meaning that when working with submodules - this rare git case becomes norm. As you would like to refer to particular commit and not tracking branch, which is essentially a moving target. You want to "lock down" yourself to specific version of the submodule.

### Adjust Package References for Projects in Asp.Net Repo (NetFx)

This applies only to .NET Framework project types (where SDK project style `<PackageReference>` is not supported).

Once you have linked `main` under `aspnet` - you need to make couple of changes for the IDE to understand new structure.
Adding reference to NuGet package in Asp.Net Mvc repository Visual Studio will create following reference hint path:

```
<Reference>
    <HintPath>
        ..\..\packages\Newtonsoft.Json.9.0.1\lib\net45\Newtonsoft.Json.dll.
    </HintPath>
</Reference>
```

Which looks quite good when working in Asp.Net Mvc repository. However, this will become problem when we will link `aspnet` repository one level up - in `epi`. Then references will be broken. Also NuGet package restore works in that way - when packages are restored on the level of solution file (under `packages` folder).

```
epi/
    |-libs/
        |-aspnet/
            |-packages/      << Asp.Net projects would be referencing to this folder
            |-libs/
                |-localization-provider
    |-packages/              << packages are restored here
    LocalizationProvider.EPiServer.sln
```

So to fix this issue - you need to change reference path to NuGet folder.

```
<Reference>
    <HintPath>
        $(SolutionDir)\packages\Newtonsoft.Json.9.0.1\lib\net45\Newtonsoft.Json.dll.
    </HintPath>
</Reference>
```

This `$(SolutionDir)` variable will ensure that top-level directory is used when looking for referenced NuGet packages.

### Publish New Repos

Publishing and pushing changes to upstream is exactly the same way as you would do for ordinary repository. There is no special magic when it comes to submodules and committing changes.

```
$ git push
```

## Cloning Repo with Submodules

When you start from scratch and need to close repository that includes submodules, you have to explicitly tell that:

```
$ git clone --recurse-submodules git://github.com/...
```

Or sometimes repository is already cloned (but you forgot to include submodules). No worries - this scenario is also covered:

```
$ git submodule update --init --recursive
```

**NB!** Sometimes you might fail to clone repo with submodules due to some weird access restrictions when submodule is configured to link to SSH instead of HTTPS. There are many resolution attempts [on internets](https://www.google.com/search?q=failed+to+clone+git+with+submodules+access+restricted+publickey).

Trick here was to explicitly set `git` protocol for the linkage (and not just `ssh`).

```
[submodule "lib/localization-provider"]
    path = lib/localization-provider
    url = git@github.com:valdisiljuconoks/LocalizationProvider.git
```

## Building Packages with Git Submodules

I've tried couple of online (Travis, Appvoyer, etc)  build systems to get around git submodules. I could not get it working properly with git submodules (there were some checkout issues for repos with submodules). Looked around and realized that there is absolutely no issues to run build with submodules on VSTS (now Azure DevOps) infrastructure. Everything worked out of the box.

![submod-2](/assets/img/2019/05/submod-2.png)

Resulting in build log entries:

![submod-1](/assets/img/2019/05/submod-1.png)

### Strongly Naming Submodule Assemblies

It's [good practice](https://channel9.msdn.com/Events/dotnetConf/2018/S107) to have strong name for your OSS projects.
My recommendation here is to have a strong name key in core/common repository and then whoever needs to sign would be referencing to the path where core/common content is linked as submodule.
For example `DbLocalizationProvider.EPiServer.csproj` file content for signing:

```
<PropertyGroup>
  <AssemblyOriginatorKeyFile>..\..\lib\aspnet\lib\localization-provider\strongname.snk</AssemblyOriginatorKeyFile>
</PropertyGroup>
```

## Update Repo with Latest Submodule Version

### Checking for Changes in Submodule

At the end of the day linked submodule is normal git repository with just a special status in linked parent repo. Other than that - you can perform standard git commands against that repo.

For example, when you need to understand whether there are any new changes from the moment when you added submodule - you can use `git fetch`.

```
$ cd lib/localization-provider
$ git fetch
remote: Enumerating objects: 7, done.
remote: Counting objects: 100% (7/7), done.
remote: Compressing objects: 100% (4/4), done.
remote: Total 4 (delta 3), reused 0 (delta 0), pack-reused 0
Unpacking objects: 100% (4/4), done.
From github.com:valdisiljuconoks/LocalizationProvider
   9ecf9eb..3a6ab2a  master     -> origin/master
```

### Moving Submodule to New Version

When you have decided to move on with submodule and include newer version in your parent repository you can safely execute this command:

```
$ git submodule update --remote
```

Or if you want to pull changes from specific branch (and be lazy and do it for all submodules if you have more than one):

```
git submodule foreach git pull origin master
```

Every time you do changes to submodule reference, it becomes as ordinary "parent" repository change that you must include and commit in order to "publish" new linkage. Kinda annoying sometimes, but that's part of the deal.

```
$ git submodule update --remote
remote: Enumerating objects: 7, done.
remote: Counting objects: 100% (7/7), done.
remote: Compressing objects: 100% (4/4), done.
Unpacking objects: 100% (4/4), done.
remote: Total 4 (delta 2), reused 0 (delta 0), pack-reused 0
From github.com:valdisiljuconoks/LocalizationProvider
   8d6dc87..3bcc663  develop    -> origin/develop
Submodule path 'localization-provider': checked out '3bcc6630a7d8b76e51c499574dcb10ef85064a2a'

$ git status
On branch develop
Your branch is up to date with 'origin/develop'.

Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

        modified:   localization-provider (new commits)

no changes added to commit (use "git add" and/or "git commit -a")
```

## Changing Submodule

As submodule is ordinary git repository that's being linked to some parent, you can do any changes you need to directly from parent repository.
However there is a catch here. Currently it's lack of proper tooling.
For example, if I change file located in core/common repository, Visual Studio will show just changes in that submodule (without specifying what files were changed).

![submod-3](/assets/img/2019/05/submod-3.png)

Also files will not be marked as changed in Solution Explorer.

But I was quite surprised about capabilities of VSCode. That amazing piece of source code is able to show changes made to more than one git repo:

![submod-4](/assets/img/2019/05/submod-4.png)

## Branching With Submodules

As written above when adding submodule to the repository, current HEAD is used as reference for the submodule (latest commit is linked).

However sometimes it's required to track only specific branch and not the most recent commit. To do so, you will need to specify branch while adding submodule to the repository. This is achievable with `-b` parameter followed by branch name.

```
$ git submodule add -b master {main-repo-url} lib/localization-provider
```

Now after adding submodule with tracking `master` branch, HEAD from that branch will be used as linkage for the submodule. Which means that if you do `$ git submodule update --remote` and there are new commits in other branches - those will be ignored and only `master` branch will be observed for the changes.

Also sometimes it's convenient to follow conventions and track the same branch in submodule as in "parent repository". For example, if you are in `develop` branch in parent repository and you want to track also `develop` branch in submodule, then you have to specify branch name as `.` symbol. This feature is available since [git v2.10+](https://github.com/git/git/commit/c8386962d6590ec6f71f353bc5b1b573b1e71b16).

But you have to do a bit of manual steps before because there is no branch named `.` and you cannot link to that branch.

```
$ git submodule add -b . {main-repo-url} lib/localization-provider
```

Line above will give you an error.

Steps you need to do to add "current" parent repository branch to be the same for submodule branch as well:

* `git submodule add -b master {main-repo-url} lib/localization-provider` - adding `master` branch as tracking branch for the submodule
* edit your `.gitmodules` file and change branch name from `master` to `.`
* `git submodule update --remote`

This is new `.gitmodules` file for `aspnet` repository:

```
[submodule "lib/localization-provider"]
    path = lib/localization-provider
    url = https://github.com/valdisiljuconoks/LocalizationProvider.git
    branch = .
```

## Best Practice to Modify Submodule

After working with repositories containing submodules we realized that it's better to avoid direct changes to submodule source code from the parent repository. There might be specific check-in policies for the submodule repository, maybe special rules how code is added, how PR are created and reviewed, branching policy, etc.

Therefore we found it better to switch to submodule repository and create branch there, make changes, add unit tests, verify, create PR, review code, merge, etc. Or whatever your git flow might look like.. And then after new changes have been stabilized and merged, then switch back to parent repository and pull new changes in by doing `git submodule update --remote`.

Happy submoduling!

[*eof*]
