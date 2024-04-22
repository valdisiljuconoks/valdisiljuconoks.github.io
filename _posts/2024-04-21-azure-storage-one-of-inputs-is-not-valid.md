---
title: How to fix Azure.RequestFailedException - One of the request inputs is not valid. exception
author: valdis
date: 2024-04-21 17:00:00 +0200
categories: [.NET, C#, Open Source, Azure, Table Storage]
tags: [.net, c#, open source, azure]
---

When you are working on APIs and want to secure access by using well-known header field names:

```csharp
[HttpGet]
[Route("verify-token")]
[Produces("application/json")]
public async Task<ActionResult> VerifyToken(
    [FromHeader(Name = "Authorization")] string? token)
{
    var t = await accessTokenRepository.GetByTokenValueAsync(token);

    if (t == null)
    {
        return Unauthorized();
    }

    return Ok();
}
```

From fist sight it might look like innocent API access control mechanism - get the token value from the header, verify against the storage - if not found, fail.

Code that looks for the token in storage is nothing fancy:

```csharp
private async Task<AccessTokenEntity> GetEntityByValueAsync(string value)
{
    var query = CreateTokenByValueFilter(value);
    var client = await GetTable();
    var result = await client.ExecuteQueryAsync<AccessTokenEntity>(query);

    return result.FirstOrDefault();
}

private static string CreateTokenByValueFilter(string value)
{
    return TableClient.CreateQueryFilter<AccessTokenEntity>(
        e => e.PartitionKey == AccessTokenEntity.PartitionKeyValue && e.Token == value);
}
```

Even if you run your sample code locally against Azure Storage Emulator (aka [azurite](https://learn.microsoft.com/en-us/azure/storage/common/storage-use-azurite?tabs=visual-studio%2Cblob-storage)) - everything might be working perfectly.

However, when you do deploy to actual Azure environment (or point your local environment to Azure Storage services) - you most probably will hit exception along these lines:

```
2024-04-19 12:24:04.473 +00:00 [ERR] Connection ID "....", Request ID "....": An unhandled exception was thrown by the application.
Azure.RequestFailedException: One of the request inputs is not valid.
RequestId:.....
Time:2024-04-19T12:24:04.4525536Z
Status: 400 (Bad Request)
ErrorCode: InvalidInput

Content:
{"odata.error":{"code":"InvalidInput","message":{"lang":"en-US","value":"One of the request inputs is not valid.\nRequestId:...\nTime:2024-04-19T12:24:04.4525536Z"}}}

Headers:
Cache-Control: no-cache
Transfer-Encoding: chunked
Server: Windows-Azure-Table/1.0 Microsoft-HTTPAPI/2.0
x-ms-request-id: ...
x-ms-client-request-id: ...
x-ms-version: REDACTED
X-Content-Type-Options: REDACTED
Date: Fri, 19 Apr 2024 12:24:04 GMT
Content-Type: application/json;odata=minimalmetadata;streaming=true;charset=utf-8

   at Azure.Data.Tables.TableRestClient.QueryEntitiesAsync(String table, Nullable`1 timeout, String nextPartitionKey, String nextRowKey, QueryOptions queryOptions, CancellationToken cancellationToken)
   at Azure.Data.Tables.TableClient.<>c__DisplayClass55_0`1.<<QueryAsync>b__0>d.MoveNext()
--- End of stack trace from previous location ---
   at Azure.Core.PageableHelpers.FuncAsyncPageable`1.AsPages(String continuationToken, Nullable`1 pageSizeHint)+MoveNext()
   at Azure.Core.PageableHelpers.FuncAsyncPageable`1.AsPages(String continuationToken, Nullable`1 pageSizeHint)+System.Threading.Tasks.Sources.IValueTaskSource<System.Boolean>.GetResult()
   ...
```

## The Solution

I did not capture network traffic and did not analyze it (as I assume that outgoing request might be the same as running locally), but removing header binding and receiving token value by url:

```csharp
[HttpGet]
[Route("verify-token")]
[Produces("application/json")]
public async Task<ActionResult> VerifyToken(string? token)
{
    var t = await accessTokenRepository.GetByTokenValueAsync(token);

    if (t == null)
    {
        return Unauthorized();
    }

    return Ok();
}
```

or binding to another header field:

```csharp
[HttpGet]
[Route("verify-token")]
[Produces("application/json")]
public async Task<ActionResult> VerifyToken(
    [FromHeader(Name = "Token")] string? token)
{
    var t = await accessTokenRepository.GetByTokenValueAsync(token);

    if (t == null)
    {
        return Unauthorized();
    }

    return Ok();
}
```

solves the problem.

I did not see that coming!

Happy coding!

[*eof*]
