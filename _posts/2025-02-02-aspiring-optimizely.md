---
title: Aspiring Optimizely
author: valdis
date: 2025-02-02 22:00:00 +0200
categories: [.NET, C#, Aspire, Optimizely]
tags: [.net, c#, aspire, optimizely]
---

Adding .NET Aspire application host to your Optimizely project (even if it's just for local development) is really a time-saver when it comes to setting up your local development environment, spinning up resources you need and tracing it during runtime.

You might ask - why do I need to add Aspire if I can just F5? True, but what is your SQL server strategy? How do you restore databases? How long it takes to setup environment on new machine?

Using Aspire hosting model - you save time for this all. Let's dive into how to get started to add Aspire to you Optimizely project.

## Adding Aspire

Adding Aspire is as easy as create new project. I'll use Visual Studio for this exercise.

* first you need to open your Optimizely solution (I'll use simple Alloy here targeted to `net8.0`)
* then you need to add new **.NET Aspire Empty App**

![](/assets/img/2025/02/opti-aspire-1.png)

* Visual Studio will ask few questions
* then `*.AppHost` and `*.ServiceDefaults` projects should be added to your solution

`*.AppHost` is the actual startup project - host and orchestrator of the solution and `*.ServiceDefaults` is utility project which we gonna need to modify a bit to fit Optimizely setup.

## Adding SQL Server

Now when we have application host, we can start adding dependency projects and/or services to our distributed application (Aspire calls it distributed - as it makes sense in the context when you are running multiple services for your solution at the same time).

* first we need to add SQL Server NuGet package reference to Aspire host project

```xml
<PackageReference Include="Aspire.Hosting.SqlServer" Version="9.0.0" />
```

This NuGet package ensures that we are able now to add SQL Server to our host.

* now in `Program.cs` file we can add SQL Server as dependency for our host

```csharp
var builder = DistributedApplication.CreateBuilder(args);

var sql = builder
    .AddSqlServer("alloy-sql")
    // Configure the container to store data in a volume so that it persists across instances.
    .WithDataVolume()
    .WithLifetime(ContainerLifetime.Persistent);

var db = sql.AddDatabase("EPiServerDB", databaseName: "alloy-db-cms");

await builder.Build().RunAsync();
```

To save even more time as you can see - I set server to `.WithLifetime(ContainerLifetime.Persistent)`. This is instructing Aspire not to kill Docker container and delete image, but preserve SQL Server instance across app launches.

Theoretically we are able to run our Aspire solution and see that SQL Server is started automatically and mapped to the solution.

![](/assets/img/2025/02/opti-aspire-2.png)

We can also check local Docker container list.

![](/assets/img/2025/02/opti-aspire-3.png)

However, connecting to SQL Server and checking databases, we can see that server is empty.

![](/assets/img/2025/02/opti-aspire-4.png)

Unfortunately SQL Server spin-up does not handle database creation automatically, so we gonna help Aspire here a bit.

## Create Optimizely CMS Database Automatically

In order to create database automatically, we need couple scripts for our Docker image.

* create new directory in `*.AppHost` project called `/sqlserverconfig` in project's root directory
* mount that directory to the image with

```csharp
// SQL doesn't support any auto-creation of databases or running scripts on startup so we have to do it manually.
var sql = builder
    .AddSqlServer("alloy-sql")
    // Mount the init scripts directory into the container.
    .WithBindMount(@".\sqlserverconfig", "/usr/config")
```

* create `entrypoint.sh` file in `/sqlserverconfig` directory which will execute database creation script

```bash
#!/bin/bash

# Start the script to create the DB and user
/usr/config/configure-db.sh &

# Start SQL Server
/opt/mssql/bin/sqlservr
```

* create `configure-db.sh` file in the same `/sqlserverconfig` directory

```bash
#!/bin/bash

# set -x

# Wait 120 seconds for SQL Server to start up by ensuring that
# calling SQLCMD does not return an error code, which will ensure that sqlcmd is accessible
# and that system and user databases return "0" which means all databases are in an "online" state

dbstatus=1
errcode=1
start_time=$SECONDS
end_by=$((start_time + 120))

echo "Starting check for SQL Server start-up at $start_time, will end at $end_by"

while [[ $SECONDS -lt $end_by && ( $errcode -ne 0 || ( -z "$dbstatus" || $dbstatus -ne 0 ) ) ]]; do
    dbstatus="$(/opt/mssql-tools/bin/sqlcmd -h -1 -t 1 -U sa -P "$MSSQL_SA_PASSWORD" -C -Q "SET NOCOUNT ON; Select SUM(state) from sys.databases")"
    errcode=$?
    sleep 1
done

elapsed_time=$((SECONDS - start_time))
echo "Stopped checking for SQL Server start-up after $elapsed_time seconds (dbstatus=$dbstatus,errcode=$errcode,seconds=$SECONDS)"

if [[ $dbstatus -ne 0 ]] || [[ $errcode -ne 0 ]]; then
    echo "SQL Server took more than 120 seconds to start up or one or more databases are not in an ONLINE state"
    echo "dbstatus = $dbstatus"
    echo "errcode = $errcode"
    exit 1
fi

# Loop through the .sql files in the /docker-entrypoint-initdb.d and execute them with sqlcmd
for f in /docker-entrypoint-initdb.d/*.sql
do
    echo "Processing $f file..."
    /opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P "$MSSQL_SA_PASSWORD" -C -d master -i "$f"
done
```

This script basically ensures that SQL Server is up & running and will execute all script files in `/docker-entrypoint-initdb.d/` directory in the image (which we gonna bind just in a moment)

* add this script to the Docker image

```csharp
var sql = builder
    .AddSqlServer("alloy-sql")
    // Mount the init scripts directory into the container.
    .WithBindMount(@".\sqlserverconfig", "/usr/config")
    // Run the custom entrypoint script on startup.
    .WithEntrypoint("/usr/config/entrypoint.sh")
```

* create new directory `/App_Data/sqlserver` in in your Alloy CMS project
* create new file `init.sql` in that `/App_Data/sqlserver` directory

```sql
-- SQL Server init script

-- Create the AddressBook database
IF NOT EXISTS (SELECT * FROM sys.databases WHERE name = N'alloy-db-cms')
BEGIN
  CREATE DATABASE [alloy-db-cms];
END;
GO
```

* and finally we need to add directory to Docker image

```csharp
var sql = builder
    .AddSqlServer("alloy-sql")
    // Mount the init scripts directory into the container.
    .WithBindMount(@".\sqlserverconfig", "/usr/config")
    // Mount the SQL scripts directory into the container so that the init scripts run.
    .WithBindMount(@"..\..\AlloyCms12New\App_Data\sqlserver", "/docker-entrypoint-initdb.d")
    // Run the custom entrypoint script on startup.
    .WithEntrypoint("/usr/config/entrypoint.sh")
```

Overall file structure is following:

![](/assets/img/2025/02/opti-aspire-5.png)

Full SQL Server setup code:

```csharp
// SQL doesn't support any auto-creation of databases or running scripts on startup so we have to do it manually.
var sql = builder
    .AddSqlServer("alloy-sql")
    // Mount the init scripts directory into the container.
    .WithBindMount(@".\sqlserverconfig", "/usr/config")
    // Mount the SQL scripts directory into the container so that the init scripts run.
    .WithBindMount(@"..\..\AlloyCms12New\App_Data\sqlserver", "/docker-entrypoint-initdb.d")
    // Run the custom entrypoint script on startup.
    .WithEntrypoint("/usr/config/entrypoint.sh")
    // Configure the container to store data in a volume so that it persists across instances.
    .WithDataVolume()
    .WithLifetime(ContainerLifetime.Persistent);
```

Now when we run the `*.AppHost` again - database named `alloy-cms-db` is automatically created.

![](/assets/img/2025/02/opti-aspire-6.png)

## Adding Alloy CMS Project

Now we are ready to add Alloy CMS project to our Aspire host.
This is much easier than it sounds.

```csharp
var db = sql.AddDatabase("EPiServerDB", databaseName: "alloy-db-cms");
builder
    .AddProject("alloy-cms", @"..\..\AlloyCms12New\AlloyCms12New.csproj")
    .WithReference(db)
    .WaitFor(db);
```

Let's build everything and run the host.

![](/assets/img/2025/02/opti-aspire-7.png)

As you can see - Alloy is up & running and part of our distributed solution.

We are also able to see console logs as part of Aspire Dashboard.

![](/assets/img/2025/02/opti-aspire-8.png)

However, structured logs, traces and metrics are not found for this app.

![](/assets/img/2025/02/opti-aspire-9.png)

Now we need to do some tricks in ServiceDefaults project which was added by Aspire project template.

## Adding Logging and Tracing (via Service Defaults)

If you check `*.ServiceDefaults` project you will see that all the extension methods to add logging and other magic to the project is for `IHostApplicationBuilder` which basically is `Microsoft.AspNetCore.Builder.WebApplicationBuilder`.

```csharp
public static TBuilder AddServiceDefaults<TBuilder>(this TBuilder builder)
    where TBuilder : IHostApplicationBuilder
```

You can have access to this object when you are building web application using

```csharp
var builder = WebApplication.CreateBuilder(args);
...
```

However, by default Optimizely is using older model - `Microsoft.Extensions.Hosting.IHostBuilder`.

```csharp
public class Program
{
    public static void Main(string[] args) => CreateHostBuilder(args).Build().Run();

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureCmsDefaults()
            .ConfigureWebHostDefaults(webBuilder => webBuilder.UseStartup<Startup>());
}
```

Using new hosting model is still work-in-progress and [not yet fully supported by Optimizely](https://world.optimizely.com/forum/developer-forum/cms-12/thread-container/2022/5/-net-6-new-minimal-hosting-model-with-cms-12-giving-exception-at-startup/).

Actually all it does is accessing `builder.Services` and configuring logging for the host.
We would need to do a bit of black magic for ServiceDefaults to get the same stuff into older hosting model.

* create a copy of `Extensions.cs` file (I chose the most creative name I could figure out - `Extensions2.cs`).
* modify `AddServiceDefaults` method to extend `IServiceCollection services` instead of `TBuilder builder` (we also gonna need `IConfiguration` and `applicationName` as well)

```csharp
public static IServiceCollection AddServiceDefaults(this IServiceCollection services, IConfiguration configuration, string applicationName)
{
    ...
}
```

* fix compilation errors - instead of `builder.Service.` use `services.`
* do the same changes for the rest of the methods
* use `applicationName` parameter in `ConfigureOpenTelemetry` method to configure tracing
* add `*.ServiceDefaults` project reference to AlloyCms project

```xml
  <ItemGroup>
    <ProjectReference Include="..\AspireApp1\AspireApp1.ServiceDefaults\AspireApp1.ServiceDefaults.csproj" />
  </ItemGroup>
```

* inject `IConfiguration` in `Startup.cs`

```csharp
public class Startup(IWebHostEnvironment webHostingEnvironment, IConfiguration configuration)
```

* use service defaults to configure tracing and metrics for AlloyCms project

```csharp
    public void ConfigureServices(IServiceCollection services)
    {
        ...

        services.AddServiceDefaults(configuration, webHostingEnvironment.ApplicationName);
    }
```

* and lastly configure logging for AlloyCms project (in `Program.cs` file)

```csharp
public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .ConfigureCmsDefaults()
        .ConfigureLogging(builder =>
        {
            builder.AddOpenTelemetry(logging =>
            {
                logging.IncludeFormattedMessage = true;
                logging.IncludeScopes = true;
            });
        })
        .ConfigureWebHostDefaults(webBuilder => webBuilder.UseStartup<Startup>());
```

## Running the Orchestrator

Now we are able to run Alloy again and enjoy logging and tracing in Aspire Dashboard.

![](/assets/img/2025/02/opti-aspire-10.png)

We can also check out traces and structured logging.

![](/assets/img/2025/02/opti-aspire-11.png)

## Checkout the Source Code

If you got lost in source code modification exercise - you can also checkout [reference project](https://github.com/valdisiljuconoks/aspiring-optimizely) for any hints you may need.


Happy asping!


[*eof*]
