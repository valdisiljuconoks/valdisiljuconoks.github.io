---
title: Chasing for NullReferenceException in EPiServer Relate+
author: valdis
date: 2014-09-03 11:30:00 +0200
categories: [.NET, C#, Episerver, Optimizely, Debugging]
tags: [.net, c#, episerver, optimizely, debugging]
---

This is a blog post telling a short story about one of the developer’s nightmare beast hunting case – `System.NullReferenceException`.

## Hunting Game Starts

Once up on a time I received an interesting issue regarding user authentication in one of the projects.
Issue description was telling that it was not possible for the users to login to the site. We were using EPiServer Relate+ in that particular area of the application.
At least I had something for the exceptional case on production server. Following message was observed in local Windows Event store:

```
Exception information:
 Exception type: NullReferenceException
 Exception message: Object reference not set to an instance of an object.
 at EPiServer.Common.Security.Data.SecurityFactory.<>c__DisplayClass5d.b__5c()
 at EPiServer.Data.Providers.SqlDatabaseHandler.<>c__DisplayClass4.b__3()
 at EPiServer.Data.Providers.SqlDatabaseHandler.<>c__DisplayClass7`1.b__6()
 at EPiServer.Data.Providers.SqlTransientErrorsRetryPolicy.Execute[TResult](Func`1 method)
 at EPiServer.Common.Security.SecurityHandler.UpdateUser(IUser user)
 at EPiServer.Common.Web.Authorization.Integrator.SynchronizeUser(MembershipUser membershipUser, String password, Boolean enableCreateNew)
 at System.Web.HttpApplication.SyncEventExecutionStep.System.Web.HttpApplication.IExecutionStep.Execute()
 at System.Web.HttpApplication.ExecuteStep(IExecutionStep step, Boolean& completedSynchronously)
```

At least

![](/assets/img/2014/09/something.png)

## Let’s do some hardcore

One of the best friends of yours is [WinDbg](http://msdn.microsoft.com/en-us/library/windows/hardware/ff551063(v=vs.85).aspx) on production environment when it comes to debugging and hunting down something.
Let’s fire that guy up on the server (actually you just need to make `xcopy` over to server at least for single `.exe` file – but it may depend if you need some additional symbols or extensions to use).
 First of all you need to tell that you are actually a .Net developer:

```
 .loadby sos clr
```

As from Windows Event log we can see that this is a `NullReferenceException` we can set explicitly a breakpoint for WinDbg instructing it that we need a notification only this guy gets thrown and don’t bother me with any other first chance junk you will come across:

```
 !soe -Create System.NullReferenceException
```

Now we are free to reproduce a test scenario when users are not able to login in application. And "voilà" – we get back to WinDbg with breakpoint hit.

```
0:033> !clrstack
 OS Thread Id: 0x6d4 (33)
 Child SP IP Call Site
 0000006da87ada10 00007ff95ee78162 EPiServer.Common.Security.Data.SecurityFactory+<>c__DisplayClass5d.b__5c()
 0000006da87adb40 00007ff95c1c1aff *** ERROR: Module load completed but symbols could not be loaded for C:/Windows/Microsoft.NET/Framework64/v4.0.30319/Temporary ASP.NET Files/root/ab1314ba/3ae9e6be/assembly/dl3/49a2918e/d9a3f7fc_f08fcf01/EPiServer.Data.dll
 EPiServer.Data.Providers.SqlDatabaseHandler+<>c__DisplayClass4.b__3()
 0000006da87adb70 00007ff95c1c1790 EPiServer.Data.Providers.SqlDatabaseHandler+<>c__DisplayClass7`1[[System.Boolean, mscorlib]].b__6()
 0000006da87adbd0 00007ff95b9a2622 EPiServer.Data.Providers.SqlTransientErrorsRetryPolicy.Execute[[System.Boolean, mscorlib]](System.Func`1)
 0000006da87adcf0 00007ff95ee76cff EPiServer.Common.Security.SecurityHandler.UpdateUser(EPiServer.Common.Security.IUser)
 0000006da87adda0 00007ff95ee5e56c *** ERROR: Module load completed but symbols could not be loaded for C:/Windows/Microsoft.NET/Framework64/v4.0.30319/Temporary ASP.NET Files/root/ab1314ba/3ae9e6be/assembly/dl3/ad46c74b/15e1f2fc_f08fcf01/EPiServer.Common.Web.Authorization.dll
 EPiServer.Common.Web.Authorization.Integrator.SynchronizeUser(System.Web.Security.MembershipUser, System.String, Boolean)
 0000006da87adf10 00007ff9b258b680 *** WARNING: Unable to verify checksum for C:/Windows/assembly/NativeImages_v4.0.30319_64/System.Web/2e2a972f876941b5d11da10ae390fdf0/System.Web.ni.dll
 *** ERROR: Module load completed but symbols could not be loaded for C:/Windows/assembly/NativeImages_v4.0.30319_64/System.Web/2e2a972f876941b5d11da10ae390fdf0/System.Web.ni.dll
 System.Web.HttpApplication+SyncEventExecutionStep.System.Web.HttpApplication.IExecutionStep.Execute()
 0000006da87adf70 00007ff9b256bee5 System.Web.HttpApplication.ExecuteStep(IExecutionStep, Boolean ByRef)
 0000006da87ae010 00007ff9b258954a System.Web.HttpApplication+PipelineStepManager.ResumeSteps(System.Exception)
 0000006da87ae160 00007ff9b256c0f3 System.Web.HttpApplication.BeginProcessRequestNotification(System.Web.HttpContext, System.AsyncCallback)
 0000006da87ae1b0 00007ff9b256613e System.Web.HttpRuntime.ProcessRequestNotificationPrivate(System.Web.Hosting.IIS7WorkerRequest, System.Web.HttpContext)
 0000006da87ae250 00007ff9b256efb1 System.Web.Hosting.PipelineRuntime.ProcessRequestNotificationHelper(IntPtr, IntPtr, IntPtr, Int32)
 0000006da87ae460 00007ff9b256e9e2 System.Web.Hosting.PipelineRuntime.ProcessRequestNotification(IntPtr, IntPtr, IntPtr, Int32)
 0000006da87ae4b0 00007ff9b2cc28d1 DomainNeutralILStubClass.IL_STUB_ReversePInvoke(Int64, Int64, Int64, Int32)
 0000006da87aecc8 00007ff9b9e3736e [InlinedCallFrame: 0000006da87aecc8] System.Web.Hosting.UnsafeIISMethods.MgdIndicateCompletion(IntPtr, System.Web.RequestNotificationStatus ByRef)
 0000006da87aecc8 00007ff9b261838b [InlinedCallFrame: 0000006da87aecc8] System.Web.Hosting.UnsafeIISMethods.MgdIndicateCompletion(IntPtr, System.Web.RequestNotificationStatus ByRef)
 0000006da87aeca0 00007ff9b261838b DomainNeutralILStubClass.IL_STUB_PInvoke(IntPtr, System.Web.RequestNotificationStatus ByRef)
 0000006da87aed70 00007ff9b256f19f System.Web.Hosting.PipelineRuntime.ProcessRequestNotificationHelper(IntPtr, IntPtr, IntPtr, Int32)
 0000006da87aef80 00007ff9b256e9e2 System.Web.Hosting.PipelineRuntime.ProcessRequestNotification(IntPtr, IntPtr, IntPtr, Int32)
 0000006da87aefd0 00007ff9b2cc28d1 DomainNeutralILStubClass.IL_STUB_ReversePInvoke(Int64, Int64, Int64, Int32)
 0000006da87af1d8 00007ff9b9e375c3 [ContextTransitionFrame: 0000006da87af1d8]
```

Well, this doesn’t give much more info about what’s really is happening on the server. Stacktrace is almost the same except that now we do have much more noise.
 One of the downside of this exception is that it occurs in EPiServer module where we do have limited access to source code and debug symbols.
 So, if you are lucky and you have a `RedGate Reflector Pro` version, that incredible tool has VS plugin that allows you “`Enable Debugging`” for even 3rd party libs to which you don’t have a source code available. RedGate Reflector disassemble `.dll` file and then generates `.pdb` file out also. This enables you to copy generated debug symbol files to target application site and inspect where excatly this happens in source code.

```
0:033> !clrstack
 OS Thread Id: 0x6d4 (33)
 Child SP IP Call Site
 0000006da87ada10 00007ff95ee78162 EPiServer.Common.Security.Data.SecurityFactory+<>c__DisplayClass5d.b__5c() [C:/Users/valdis/AppData/Local/Red Gate/.NET Reflector 8/Cache/0/EPiServer.Common.Security.pdb.source/EPiServer/Common/Security/Data/SecurityFactory.cs @ 1020]
 0000006da87adb40 00007ff95c1c1aff *** ERROR: Module load completed but symbols could not be loaded for C:/Windows/Microsoft.NET/Framework64/v4.0.30319/Temporary ASP.NET Files/root/ab1314ba/3ae9e6be/assembly/dl3/49a2918e/d9a3f7fc_f08fcf01/EPiServer.Data.dll
 EPiServer.Data.Providers.SqlDatabaseHandler+<>c__DisplayClass4.b__3()
 0000006da87adb70 00007ff95c1c1790 EPiServer.Data.Providers.SqlDatabaseHandler+<>c__DisplayClass7`1[[System.Boolean, mscorlib]].b__6()
 0000006da87adbd0 00007ff95b9a2622 EPiServer.Data.Providers.SqlTransientErrorsRetryPolicy.Execute[[System.Boolean, mscorlib]](System.Func`1)
 0000006da87adcf0 00007ff95ee76cff EPiServer.Common.Security.SecurityHandler.UpdateUser(EPiServer.Common.Security.IUser) [C:/Users/valdis/AppData/Local/Red Gate/.NET Reflector 8/Cache/0/EPiServer.Common.Security.pdb.source/EPiServer/Common/Security/SecurityHandler.cs @ 1223]
….
```

Finally we can spot a source of our exception. The given source line number actually as it turned out wasn’t really precise – it pointed to following region in our `UpdateUser` method where `OpenIDCollection` is constructed:

```csharp
protected internal virtual void UpdateUser(User user)
{
    User tmpUser = this.GetUser(user.ID, true);
    ...

    databaseHandler.ExecuteTransaction((Action) (() =>
    {
        …
        OpenIDCollection local_3 = new OpenIDCollection();

        if(tmpUser.OpenIDs.Count > 0)
        {
            …
        }

        …
    }));
}
```

Browsing constructor of the custom collection – we can be sure there is actually nothing that could be `null` and/or throw an `Exception`.

## Inspecting method parameters

Well, next thing that we could do is to browse a bit around executing environment of the method.
As `UpdateUser` uses lambda expressions it starts to get a bit more trickier and more fun.
To get a parameters for given stacktrace we can use `-p` switch for `clrstack` command:

```
0:033> !clrstack -p
 OS Thread Id: 0x6d4 (33)
 Child SP IP Call Site
 0000006da87ada10 00007ff95ee78162 EPiServer.Common.Security.Data.SecurityFactory+<>c__DisplayClass5d.b__5c() [C:/Users/valdis/AppData/Local/Red Gate/.NET Reflector 8/Cache/0/EPiServer.Common.Security.pdb.source/EPiServer/Common/Security/Data/SecurityFactory.cs @ 1020]
 PARAMETERS:
 this (0x0000006da87adb40) = 0x0000006c59f138f0

0000006da87adb40 00007ff95c1c1aff EPiServer.Data.Providers.SqlDatabaseHandler+<>c__DisplayClass4.b__3()
 PARAMETERS:
 this =

0000006da87adb70 00007ff95c1c1790 EPiServer.Data.Providers.SqlDatabaseHandler+<>c__DisplayClass7`1[[System.Boolean, mscorlib]].b__6()
 PARAMETERS:
 this =

0000006da87adbd0 00007ff95b9a2622 EPiServer.Data.Providers.SqlTransientErrorsRetryPolicy.Execute[[System.Boolean, mscorlib]](System.Func`1)
 PARAMETERS:
 this (0x0000006da87adcf0) = 0x0000006c59df9c68
 method (0x0000006da87adcf8) = 0x0000006c59f1d5a0
…
```

We can dump lambda expression generated method parameter object and see what’s inside there:

```
0:033> !do 0x0000006c59f138f0
 Name: EPiServer.Common.Security.Data.SecurityFactory+<>c__DisplayClass5d
 MethodTable: 00007ff95ef12578
 EEClass: 00007ff95ef07f00
 Size: 56(0x38) bytes
 File: C:/Windows/Microsoft.NET/Framework64/v4.0.30319/Temporary ASP.NET Files/root/ab1314ba/3ae9e6be/assembly/dl3/3230184a/15e1f2fc_f08fcf01/EPiServer.Common.Security.dll
 Fields:
 MT Field Offset Type VT Attr Value Name
 00007ff95b4d7048 400010e 8 …mon.Security.User 0 instance 0000000000000000 tmpUser
 00007ff95bdd1958 400010f 10 …ativeAccessRights 0 instance 0000006c59f1ba38 accessRights
 00007ff95b65dfd8 4000110 18 ….IDatabaseHandler 0 instance 0000006c59df9bd0 databaseHandler
 00007ff95bdbd158 4000111 20 …a.SecurityFactory 0 instance 0000006b547918a0 <>4__this
 00007ff95b4d7048 4000112 28 …mon.Security.User 0 instance 0000006c59f10308 user
```

This gets more interesting. We can see that `tmpUser` actually is `null` – `Value = 0000000000000000`.
If we take a closer look at body of the method we can see that `tmpUser` is used later on to understand what kind of changes have been made to the user in order to calculate necessary updates to be sent to database.

```
User tmpUser = this.GetUser(user.ID, true);
...

if(tmpUser.OpenIDs.Count > 0)
{
    ...
```

We can verify once again the passed in user object to the method (dump `user` from the lambda expression parameter object):

```
0:033> !do 0000006c59f10308
 Name: EPiServer.Common.Security.User
 MethodTable: 00007ff95b4d7048
 EEClass: 00007ff95b49fc30
 Size: 272(0x110) bytes
 File: C:/Windows/Microsoft.NET/Framework64/v4.0.30319/Temporary ASP.NET Files/root/ab1314ba/3ae9e6be/assembly/dl3/3230184a/15e1f2fc_f08fcf01/EPiServer.Common.Security.dll
 Fields:
 MT Field Offset Type VT Attr Value Name
 00007ff9b8e737c8 4000001 18 System.Int32 1 instance 95887 m_id
 00007ff9b8e87978 4000002 20 System.Guid 1 instance 0000006c59f10328 m_uniqueId
 00007ff9b8e711b8 4000003 8 System.Object 0 instance 0000006a5ade9d88 _syncObj
 00007ff9b8e6f370 4000004 1c System.Boolean 1 instance 0 m_ownedObjectsLoaded
 00007ff9b8e711b8 4000005 10 System.Object 0 instance 0000006a5ade9b88 m_master
 00007ff95b483748 4000006 30 ….IFrameworkEntity 0 instance 0000006a5ade9b88 m_entity
 ...
```

We can see that `user` variable seems like to be valid. It has it’s own ID, Master object, etc.

## Inspecting last chain – database

Well, if we take a look at `GetUser(int, bool)` method that’s filling up `tmpUser` variable we can see that one of the way that this method returns `null` value is if it fails to read SP result or there is no results at all.
So it’s quite interesting what would stored procedure return and why `GetUser` returns null in the first place?

Sneak peek into stored procedure and executing query we can get confirmation that there is no such user with `ID` (`95887`) given by `user` variable that was passed in as method parameter for our lambda expression.

```
select * from tblEPiServerCommonUser where intID = 95887

(0 row(s) affected)
```

Now it get’s really confusing.
Let’s get back to facts that we know already:

* When user is logging in we got an exception
* New users are not saved in the database
* Synchronization process happens for the Relate+ user anyway

Our project specifics were that during initial process when we need to create new user, it gets synced once again after domain has raised an event and handler received roles from external system.
During this second round `UpdateUser` somehow fails with `NullReferenceException`.
Digging more into SQL queries and its results finally I noticed following line:

![](/assets/img/2015/06/null-ref-excp-1.png)

This was the key to the solution. By executing manually failing command we got response from the SQL Server:

```
exec spEPiServerCommunityBlogAddBlog @strHeader=N’MyPage Blog for User ”…”’,@strBody=N”,@intAuthorID=…,@blnPingEnabled=0,@strLanguageID=NULL,@intStatus=1

Msg 515, Level 16, State 2, Procedure spEPiServerCommunityBlogAddBlog, Line 11
 Cannot insert the value NULL into column ‘intPreviousStatus’, table ‘dbEPiServer.dbo.tblEPiServerCommunityBlog'; column does not allow nulls. INSERT fails.
 The statement has been terminated.
```

As it’s turned out – you have to double check what’s going in your database after you ran EPiServer Relate+ database upgrade scripts. Not complete database schema was upgraded correctly!

Have fun!

[*eof*]
