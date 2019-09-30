---
title: Client vs. Server Evaluation - EF Core
author: smitpatel
ms.date: 10/03/2019
ms.assetid: 8b6697cc-7067-4dc2-8007-85d80503d123
uid: core/querying/client-eval
---
# Client vs. Server Evaluation

As a general rule, Entity Framework Core attempts to evaluate query on server as much as possible. EF Core converts parts of the query to parameters, which it can evaluate on client side. Rest of the query with generated parameters is given to database providers to determine what should be equivalent database query to do evaluation on the server. EF Core supports partial client evaluation in top-level projection (essentially, the last call to `Select()`). If parts of query in top-level projection can't be translated to server, EF Core will fetch required data from server and evaluate it on client. If EF Core detects expression, which can't be translated to server in any other place, then it throws runtime exception. See [how query works](xref:core/querying/how-query-works) to understand how EF Core determines what can't be translated to server.

> [!NOTE]
> Prior to version 3.0, Entity Framework Core supported client evaluation anywhere in the query. For more information, see the [previous versions section](#previous-versions).

## Client evaluation in top-level projection

> [!TIP]
> You can view this article's [sample](https://github.com/aspnet/EntityFramework.Docs/tree/master/samples/core/Querying) on GitHub.

In the following example, a helper method is used to standardize URLs for blogs, which are returned from a SQL Server database. Since the SQL Server provider has no insight into how this method is implemented, it isn't possible to translate it into SQL. All other aspects of the query are evaluated in the database, but passing the returned `URL` through this method is done on the client.

[!code-csharp[Main](../../../samples/core/Querying/ClientEval/Sample.cs#ClientProjection)]

[!code-csharp[Main](../../../samples/core/Querying/ClientEval/Sample.cs#ClientMethod)]

## Unsupported client evaluation

While client evaluation is useful, it can result in poor performance sometimes. Consider the following query, in which the helper method is now used in a where filter. Because filter can't be applied in the database, all the data needs to be pulled into memory to apply filter on the client. Based on the amount of data on the server and the filter, client evaluation could result in poor performance. So Entity Framework Core blocks such client evaluation and throws a runtime exception.

[!code-csharp[Main](../../../samples/core/Querying/ClientEval/Sample.cs#ClientWhere)]

## Explicit client evaluation

User may need to force into client evaluation explicitly in certain cases like following

- Data is small so that doing evaluation on client doesn't cause huge performance penalty.
- LINQ operator being used has no server-side translation.

In such cases, user can explicitly opt into client evaluation by calling methods like `AsEnumerable` or `ToList` (`AsAsyncEnumerable` or `ToListAsync` for async). By using `AsEnumerable` you would be streaming the results, but using `ToList` would cause buffering by creating the list, which also takes additional memory. Though if you're enumerating multiple times, then storing results in a list helps more since there's only one query to database. Depending on the particular usage, you should evaluate which method is more useful for the case.

[!code-csharp[Main](../../../samples/core/Querying/ClientEval/Sample.cs#ExplicitClientEval)]

## Potential memory leak in client evaluation

Since query translation and compilation are expensive, EF Core caches the compiled query plan. The cached delegate may use client code while doing client evaluation of top-level projection. EF Core generates parameters for client parts of the tree, to reuse the query plan by replacing parameter values. But certain constants in expression tree can be converted to parameters. If the cached delegate contains such constants, then those objects can't be garbage collected as they're still being referenced. If such object contains DbContext or other services inside it, then it could cause memory usage of the app to grow over time. It's generally sign of memory leak. EF Core throws exception whenever it comes across constants of type that can't be mapped using current database providers. Common causes and their solutions are as follows:

- **Using instance method**: When using instance methods in client projection, the expression tree contains constant of the instance. If your method doesn't use any data from instance, consider making the method static. If you need instance data in the method body, then pass specific data as an argument to the method.
- **Passing constant argument to method**: This case arises generally by using `this` in argument to client method. Consider splitting argument in to multiple scalar arguments, which can be mapped by database provider.
- **Other constants**: If constant is seen in any other case, then you can evaluate if constant is needed in processing. If it's necessary to have the constant or if you can't use solution to above cases, then create a local variable to store the value and use local variable in the query. EF Core will convert it to parameter.

## Previous versions

The following section applies to EF Core versions before 3.0.

Older EF Core versions supported client evaluation in any part of query not just top-level projection. That's why queries similar to one posted under [Unsupported client evaluation](#unsupported-client-evaluation) section worked correctly. Since this behavior could cause unnoticed performance issue, EF Core logged client evaluation warning. For more information on viewing logging output, see [Logging](xref:core/miscellaneous/logging).

Optionally EF Core allowed to change default behavior to throw exception or do nothing when doing client evaluation except for projection. The exception throwing would make it similar to the behavior in 3.0. To change the behavior, you need to configure warnings while setting up the options for your context - typically in `DbContext.OnConfiguring`, or in `Startup.cs` if you're using ASP.NET Core.

```csharp
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder
        .UseSqlServer(@"Server=(localdb)\mssqllocaldb;Database=EFQuerying;Trusted_Connection=True;")
        .ConfigureWarnings(warnings => warnings.Throw(RelationalEventId.QueryClientEvaluationWarning));
}
```
