# C02 — EF Core Querying, Performance & Patterns (EF Core 8, SQL Server)

**How to use:** Read the *Study notes* first, then climb each difficulty ladder (Easy → Tricky) and answer aloud before reading the **Answer**.
Code targets EF Core 8 + SQL Server; generated SQL is representative (aliases/whitespace trimmed).

## Table of Contents
1. IQueryable vs IEnumerable (client vs server evaluation)
2. Include / ThenInclude, filtered include, cartesian explosion, AsSplitQuery
3. The N+1 problem
4. Projection with Select / DTOs
5. Pagination (Skip/Take vs keyset) and Count performance
6. GroupBy, joins, subqueries translation
7. Raw SQL, stored procedures, views, SQL-injection safety
8. Compiled queries & query caching
9. Bulk operations (ExecuteUpdate/Delete, AddRange, bulk libs), tracking costs & tuning checklist
10. Concurrency control (rowversion, DbUpdateConcurrencyException)
11. Transactions, execution strategies, locking (optimistic vs pessimistic)
12. Async, cancellation, deadlocks & timeouts
13. Interceptors, auditing, soft delete, multi-tenancy
14. EF Core 8/9 features (JSON, complex types, HierarchyId, primitive collections)
15. EF vs Dapper, Repository/UoW, Specification, CQRS, DDD aggregates, testing
16. Large-table migrations & production scenario ladders

---

## Topic 1: IQueryable vs IEnumerable (client vs server evaluation)

**Study notes**

- `IQueryable<T>` holds an **expression tree** (`Expression<Func<>>`); the provider translates it to SQL when enumerated. `IEnumerable<T>` holds **delegates** (`Func<>`) and runs in memory (LINQ to Objects).
- Query is **deferred**: it executes on `foreach`, `ToList`, `First`, `Count`, `Any`, `Single`, etc.
- Once you call `AsEnumerable()`, `ToList()`, `ToArray()` or assign to `IEnumerable<T>` and keep chaining, everything after runs in memory.
- EF Core 3.0+ **no longer silently client-evaluates** an untranslatable predicate: it throws `InvalidOperationException` ("could not be translated"). The only client-evaluated part allowed is the **final top-level `Select` projection**.
- Passing a `Func<>` instead of `Expression<Func<>>` to a `Where` binds to `Enumerable.Where` → whole table is loaded.

```
 db.Orders            .Where(expr)     .Select(expr)     .ToList()
 IQueryable<Order> -> IQueryable    -> IQueryable    -> executes SQL, materializes
   (expression tree built up, nothing sent to DB yet)

 .AsEnumerable() = the "fence": SQL stops here, rest is LINQ to Objects
```

### 1.1 [Easy] What is the difference between IQueryable<T> and IEnumerable<T>?
**Answer**
`IQueryable<T>` builds an expression tree that the provider (EF Core) translates into SQL, so filtering/sorting happen in the database. `IEnumerable<T>` works on in-memory objects with compiled delegates. Using the wrong one can pull an entire table into memory.

**Explanation**
`IQueryable<T>` extends `IEnumerable<T>` but its `Where/Select` extension methods (`Queryable.*`) take `Expression<Func<..>>`. EF's provider visits the tree and generates SQL on enumeration. `Enumerable.*` methods take `Func<..>` and loop over rows already fetched.

**Code**
```csharp
// Server-side: filter in SQL
var cheap = await db.Products.Where(p => p.Price < 10).ToListAsync();

// Client-side: loads ALL products, then filters in memory
IEnumerable<Product> all = db.Products;
var cheap2 = all.Where(p => p.Price < 10).ToList();
```
```sql
-- first query
SELECT [p].[Id], [p].[Name], [p].[Price] FROM [Products] AS [p] WHERE [p].[Price] < 10.0
-- second query
SELECT [p].[Id], [p].[Name], [p].[Price] FROM [Products] AS [p]
```

### 1.2 [Follow-up] What does "deferred execution" mean and when does the query actually run?
**Answer**
The query is only defined, not run, until something enumerates it: `foreach`, `ToList/ToArray`, `First/Single`, `Count/Any/Sum`, `AsAsyncEnumerable`, etc. Re-enumerating re-executes the SQL each time.

**Explanation**
A common bug is enumerating the same `IQueryable` twice (e.g. `if (q.Any()) foreach (var x in q)`) which hits the DB twice and may return different data. Materialize once with `ToListAsync` when you need to reuse the results. Also, composition after `ToList()` is no longer SQL.

**Code**
```csharp
var q = db.Orders.Where(o => o.Status == OrderStatus.Open); // no SQL yet
var count = await q.CountAsync();      // SQL #1
var list  = await q.ToListAsync();     // SQL #2
```

### 1.3 [Medium] What happens in EF Core 3+ when a predicate contains a method EF cannot translate? How do you fix it?
**Answer**
EF Core throws an `InvalidOperationException` saying the LINQ expression could not be translated (it no longer silently loads everything). Fix by rewriting using translatable members/functions (`EF.Functions`, `string.Contains`, etc.), mapping a DB function, or deliberately switching to client evaluation after narrowing server-side.

**Explanation**
Options in order of preference: (1) rewrite the expression; (2) map a scalar function via `HasDbFunction` / `EF.Functions.*`; (3) filter in SQL as much as possible, then `AsEnumerable()` and apply the C# logic on the reduced set; (4) raw SQL. Client evaluation is still allowed in the **last** `Select`.

**Code**
```csharp
// Throws: IsValid is a custom C# method
var bad = await db.Customers.Where(c => IsValid(c.Email)).ToListAsync();

// Narrow in SQL, then finish in memory
var good = db.Customers
    .Where(c => c.Email != null && c.CreatedAt > cutoff)   // SQL
    .AsEnumerable()
    .Where(c => IsValid(c.Email))                           // memory
    .ToList();

// Translatable alternative
var like = await db.Customers
    .Where(c => EF.Functions.Like(c.Name, "A%")).ToListAsync();
```
```sql
SELECT ... FROM [Customers] AS [c] WHERE [c].[Name] LIKE N'A%'
```

### 1.4 [Hard] Where exactly can EF Core still do client evaluation, and what are the risks?
**Answer**
Only in the outermost projection (final `Select`), where a custom C# method is applied after rows are fetched. The risk is that the method runs per row, can't use DB indexes, and may hide the fact that you are fetching more columns than needed.

**Explanation**
EF translates what it can into the SELECT list and evaluates the rest in the shaper. Anything in `Where`, `OrderBy`, `GroupBy`, `Join` must translate. Be careful that a client-evaluated method inside `Select` that itself touches `DbContext` creates hidden N+1 queries and wrongly mixes concerns.

**Code**
```csharp
var dtos = await db.Orders
    .Select(o => new OrderDto
    {
        Id = o.Id,
        Label = FormatLabel(o.Number, o.Date) // runs in memory per row
    })
    .ToListAsync();
```
```sql
SELECT [o].[Id], [o].[Number], [o].[Date] FROM [Orders] AS [o]
```

**Gotchas / follow-ups**
- Moving `FormatLabel` into a `Where` after this throws – only top-level projection is exempt.
- Ordering by `FormatLabel(...)` throws too.
- Follow-up: "How would you detect hidden client eval?" → log SQL (`LogTo`) and compare to the intended query.

### 1.5 [Tricky] Why does `Func<>` vs `Expression<Func<>>` in a repository method silently destroy performance?
**Answer**
If a method accepts `Func<T,bool>`, the call resolves to `Enumerable.Where`, so EF loads the whole table and filters in memory. Accepting `Expression<Func<T,bool>>` keeps it as `IQueryable` translation.

**Explanation**
Lambdas convert to either type; the parameter type decides. This is the classic generic-repository bug. Also returning `IEnumerable<T>` from a repository method (instead of `IQueryable<T>` or a materialized list) makes later `.Where` calls client-side.

**Code**
```csharp
// BAD - whole table loaded, then filtered
public IEnumerable<Order> Find(Func<Order, bool> predicate) => db.Orders.Where(predicate);

// GOOD - translated to SQL
public Task<List<Order>> FindAsync(Expression<Func<Order, bool>> predicate, CancellationToken ct)
    => db.Orders.Where(predicate).ToListAsync(ct);
```

**Gotchas / follow-ups**
- Exposing `IQueryable` from a repository leaks the abstraction but preserves composability; exposing `IEnumerable` kills composability silently.
- `ToList()` too early + `Take(10)` later = loads all rows.

**Interview traps**
- Saying "`IQueryable` is faster" — it's faster only because work moves to the DB.
- Forgetting that `AsEnumerable()` ends SQL composition but doesn't execute immediately (it streams).
- `Func<>` parameter in repositories.
- Calling `.ToList()` before `.Where/.Take/.Count`.
- Believing EF Core 3+ still falls back to client evaluation everywhere.

**Cheat sheet**
- `IQueryable` = expression tree → SQL; `IEnumerable` = delegates → memory.
- Execution is deferred until enumeration; each enumeration re-queries.
- Untranslatable query → exception (EF Core 3+), except in the last `Select`.
- `AsEnumerable()` is the SQL/memory boundary.
- Use `Expression<Func<..>>` in reusable APIs.

---

## Topic 2: Include / ThenInclude, filtered include, cartesian explosion, AsSplitQuery

**Study notes**

- `Include` eagerly loads related data in the same query (JOIN). `ThenInclude` goes deeper. Works only for **tracked or no-tracking entity** queries; ignored when you project with `Select` to non-entities.
- **Filtered include** (EF Core 5+): `Include(c => c.Orders.Where(...).OrderBy(...).Take(n))` — allowed operators: `Where`, `OrderBy/ThenBy` (and descending), `Skip`, `Take`, `AsSplitQuery` etc.; each navigation can have only one distinct filter per query.
- **Cartesian explosion**: joining several collection navigations in a single query multiplies rows (Orders × Payments × Items) → huge result sets, duplicated parent data.
- **Single query** (default): one SQL, consistent snapshot, but risk of explosion. **Split query** (`AsSplitQuery()` or `UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery)`): one SQL per collection, no explosion but multiple roundtrips and no atomic snapshot (unless in a serializable/snapshot transaction).
- Collection includes on a single query without a configured behavior trigger warning `MultipleCollectionIncludeWarning`.

```
Single query (JOINs)                 Split query
Blog ⋈ Posts ⋈ Tags  -> P x T rows    SELECT Blogs
                                      SELECT Posts   JOIN Blogs (same filter/order)
                                      SELECT Tags    JOIN Posts JOIN Blogs
```

### 2.1 [Easy] What do Include and ThenInclude do?
**Answer**
They eagerly load related entities along with the main entity in the same query, so navigation properties are populated without extra roundtrips. `ThenInclude` loads the next level from the previously included navigation.

**Explanation**
Without them, navigations are null/empty (no lazy loading by default). Relationship fix-up also fills navigations from already-tracked entities even without Include, which can be confusing.

**Code**
```csharp
var orders = await db.Orders
    .Include(o => o.Customer)
    .Include(o => o.Lines).ThenInclude(l => l.Product)
    .ToListAsync();
```
```sql
SELECT o.*, c.*, l.*, p.*
FROM Orders o
INNER JOIN Customers c ON o.CustomerId = c.Id
LEFT JOIN OrderLines l ON o.Id = l.OrderId
LEFT JOIN Products p ON l.ProductId = p.Id
ORDER BY o.Id, c.Id, l.Id
```

### 2.2 [Follow-up] What is a filtered include and what can you put in it?
**Answer**
It lets you load only part of a collection navigation: `Include(x => x.Items.Where(...).OrderBy(...).Take(n))`. Supported operations are Where, OrderBy/ThenBy (and Descending), Skip, Take.

**Explanation**
Useful for "customer with 5 latest orders". The filter applies to the related collection only; it doesn't filter the parents. You can't use two different filters on the same navigation in the same query. Note that fix-up of already tracked entities could add more items than your filter, so filtered include with tracking is subtle.

**Code**
```csharp
var customers = await db.Customers
    .Include(c => c.Orders
        .Where(o => o.Status == OrderStatus.Open)
        .OrderByDescending(o => o.CreatedAt)
        .Take(5))
    .ToListAsync();
```

### 2.3 [Medium] What is cartesian explosion and how do you handle it?
**Answer**
When one query joins multiple collection navigations, the row count is the product of child counts, duplicating parent columns and wasting bandwidth/memory. Fix with `AsSplitQuery()`, projecting only what you need, or separate queries.

**Explanation**
Blog with 100 Posts and 50 Tags loaded in one query returns 5,000 rows per blog. Split query runs one SELECT per collection. Alternatively reduce through projection (`Select`) with aggregated counts or run explicit separate queries and combine.

**Code**
```csharp
var blogs = await db.Blogs
    .Include(b => b.Posts)
    .Include(b => b.Tags)
    .AsSplitQuery()
    .ToListAsync();

// Global default
services.AddDbContext<AppDb>(o => o.UseSqlServer(cs,
    s => s.UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery)));
// then opt back in per query: .AsSingleQuery()
```
```sql
SELECT [b].[Id], [b].[Name] FROM [Blogs] AS [b] ORDER BY [b].[Id];
SELECT [p].*, [b].[Id] FROM [Blogs] AS [b] INNER JOIN [Posts] AS [p] ON [b].[Id] = [p].[BlogId] ORDER BY [b].[Id];
SELECT [t].*, [b].[Id] FROM [Blogs] AS [b] INNER JOIN [Tags] AS [t] ON [b].[Id] = [t].[BlogId] ORDER BY [b].[Id];
```

### 2.4 [Hard] What are the trade-offs of AsSplitQuery versus a single query?
**Answer**
Split avoids row multiplication and large payloads but costs extra roundtrips, and the queries are not atomic, so concurrent writes can produce inconsistent results across the splits. Single query is one consistent statement but can explode. Choose by cardinality of the collections and latency to the DB.

**Explanation**
- Split queries need a deterministic ordering; EF adds ordering by key. With `Skip/Take` you must ensure a unique order, otherwise pages can differ between splits.
- Without a surrounding transaction at snapshot/serializable isolation, data may change between the statements.
- Single reference (non-collection) includes never need splitting — they don't multiply rows.
- Single query with one collection Include is generally fine.
- Split queries issue multiple commands per LINQ query, so latency to the DB multiplies the roundtrip cost.

**Code**
```csharp
await using var tx = await db.Database.BeginTransactionAsync(IsolationLevel.Snapshot, ct);
var data = await db.Blogs.Include(b => b.Posts).Include(b => b.Tags)
    .AsSplitQuery().OrderBy(b => b.Id).Take(20).ToListAsync(ct);
await tx.CommitAsync(ct);
```

**Gotchas / follow-ups**
- Snapshot isolation must be enabled on the DB (`ALLOW_SNAPSHOT_ISOLATION ON`).
- `Include` + `Select` to anonymous types: Include is ignored, so use projection directly.
- Splitting is per-query decided at translation; `AsSingleQuery()` overrides a global split default.

### 2.5 [Tricky] Why might an Include not appear to work, or return extra related rows?
**Answer**
Include is ignored when the query result type changes (e.g. `Select` to a DTO/anonymous type) and navigations can also be populated by fix-up from other tracked entities, so you may see related data you didn't include. With filtered includes on tracked queries, entities already tracked can appear in the collection regardless of the filter.

**Explanation**
Use `AsNoTracking()` or a fresh context to get deterministic shapes. Also, `AsNoTracking()` queries do **not** do identity resolution: the same related entity is materialized as separate instances unless you use `AsNoTrackingWithIdentityResolution()`, which matters for large graphs with shared references.

**Code**
```csharp
var orders = await db.Orders
    .AsNoTrackingWithIdentityResolution()
    .Include(o => o.Customer) // shared Customer instances de-duplicated
    .ToListAsync();

var wrong = await db.Orders.Include(o => o.Lines)
    .Select(o => new { o.Id }) // Include ignored
    .ToListAsync();
```

**Gotchas / follow-ups**
- Circular graphs from no-tracking queries w/o identity resolution can serialize weirdly or duplicate memory.
- `System.Text.Json` cycle errors come from returning entity graphs — return DTOs.

**Interview traps**
- "Include does a LEFT JOIN always" — required relationships use INNER JOIN.
- Using Include for read-only data you only need two columns from.
- Assuming split queries are consistent snapshots.
- Forgetting a deterministic OrderBy with Skip/Take and split queries.
- Many Includes in a single query without noticing the `MultipleCollectionIncludeWarning`.

**Cheat sheet**
- Include/ThenInclude = eager load via JOIN; ignored after projection.
- Filtered include: Where/OrderBy/Skip/Take only.
- Cartesian explosion = multiple collection joins multiplying rows.
- `AsSplitQuery` = N+1 statements (fixed count), not atomic; `AsSingleQuery` overrides default.
- Prefer projection when you don't need entities.

---

## Topic 3: The N+1 problem

**Study notes**

- **N+1**: 1 query for the parent list + N queries, one per parent, to get children.
- Causes in EF: **lazy-loading proxies** (`UseLazyLoadingProxies`, virtual navigations), **explicit loading in a loop** (`Entry(x).Collection(...).Load()`), a service method called per row that queries the DB, or client-evaluated `Select` that calls a method hitting the DB.
- Detection: SQL logging (`LogTo`, `EnableSensitiveDataLogging` in dev), SQL Profiler/Extended Events, Application Insights dependency counts, MiniProfiler, test asserting command count with a `DbCommandInterceptor`.
- Fixes: `Include`, projection (`Select`) that traverses navigations in one SQL, batched `Where(x => ids.Contains(x.ParentId))` (explicit "batch loading"), `AsSplitQuery` for multiple collections.

```
Lazy loading:  SELECT * FROM Orders            -- 1
               SELECT * FROM Lines WHERE OrderId=1   -- +1
               SELECT * FROM Lines WHERE OrderId=2   -- +1  ... N times
```

### 3.1 [Easy] What is the N+1 problem?
**Answer**
It's issuing one query to fetch N parent rows and then one additional query per parent to fetch its related data, giving N+1 roundtrips. It scales poorly with data size and network latency.

**Explanation**
Each roundtrip has fixed overhead (latency, parsing, plan lookup), so 1,000 parents means 1,001 roundtrips versus one or two queries.

**Code**
```csharp
var orders = await db.Orders.ToListAsync();
foreach (var o in orders)
{
    // N extra queries
    var lines = await db.OrderLines.Where(l => l.OrderId == o.Id).ToListAsync();
}
```

### 3.2 [Follow-up] How do you fix it?
**Answer**
Load the related data in the same or a constant number of queries: `Include`, a projection to a DTO, or a batched query using `Contains` on the parent IDs. Pick projection when you don't need entities.

**Explanation**
Projection often yields the best SQL (only needed columns, aggregated subqueries). Batched loading with `Contains` is useful when a join would explode rows.

**Code**
```csharp
// Include
var a = await db.Orders.Include(o => o.Lines).ToListAsync();

// Projection
var b = await db.Orders.Select(o => new OrderDto
{
    Id = o.Id,
    Total = o.Lines.Sum(l => l.Qty * l.UnitPrice),
    LineCount = o.Lines.Count
}).ToListAsync();

// Batched
var ids = orders.Select(o => o.Id).ToList();
var lines = await db.OrderLines.Where(l => ids.Contains(l.OrderId)).ToListAsync();
```
```sql
SELECT [o].[Id], (SELECT SUM(l.Qty*l.UnitPrice) FROM OrderLines l WHERE l.OrderId=o.Id) AS Total,
       (SELECT COUNT(*) FROM OrderLines l WHERE l.OrderId=o.Id) AS LineCount
FROM Orders o
```

### 3.3 [Medium] How does lazy loading cause N+1 and how do you control it?
**Answer**
With `Microsoft.EntityFrameworkCore.Proxies` and `UseLazyLoadingProxies()` (or `ILazyLoader` injection), touching an unloaded navigation fires a query. In loops this becomes N+1. Avoid it by not enabling lazy loading, or by disabling it and loading deliberately, and by using `Include`/projection.

**Explanation**
Requirements for proxies: navigations are `virtual`, classes non-sealed. Lazy loading is also sync I/O (no async), and fails outside the context lifetime (ObjectDisposedException) — e.g. during JSON serialization after the request scope. Many teams ban it in APIs. Use `ConfigureWarnings` or analyzers to catch it in tests.

**Code**
```csharp
services.AddDbContext<AppDb>(o => o
    .UseLazyLoadingProxies() // opt-in; avoid in APIs
    .UseSqlServer(cs));

public class Order { public virtual ICollection<OrderLine> Lines { get; set; } = new List<OrderLine>(); }

foreach (var o in db.Orders.ToList())
    Console.WriteLine(o.Lines.Count); // each access = query
```

### 3.4 [Hard] How would you detect N+1 in an existing production application?
**Answer**
Look at traces/telemetry: a single request issuing hundreds of near-identical parameterized commands. Locally, enable `LogTo` or an interceptor that counts commands per request, and add integration tests that assert a maximum command count.

**Explanation**
In SQL Server, Query Store / `sys.dm_exec_query_stats` shows queries with huge execution counts and tiny durations. APM tools (App Insights dependency calls, OpenTelemetry EF instrumentation) show the repeating pattern. A `DbCommandInterceptor` incrementing a counter in a scoped service can fail tests above a threshold.

**Code**
```csharp
public sealed class CommandCounter : DbCommandInterceptor
{
    public int Count;
    public override ValueTask<InterceptionResult<DbDataReader>> ReaderExecutingAsync(
        DbCommand command, CommandEventData eventData,
        InterceptionResult<DbDataReader> result, CancellationToken ct = default)
    {
        Interlocked.Increment(ref Count);
        return base.ReaderExecutingAsync(command, eventData, result, ct);
    }
}

optionsBuilder.AddInterceptors(counter);
optionsBuilder.LogTo(Console.WriteLine, LogLevel.Information);
```

**Gotchas / follow-ups**
- Counting commands per *request* needs a scoped/AsyncLocal counter, not a singleton shared across requests.
- Not every repeating query is N+1; caching misses can look the same.

### 3.5 [Tricky] Can an N+1 occur with no lazy loading and no loops visible in your code?
**Answer**
Yes: a client-evaluated method in the final `Select` that calls a service hitting the DB, a mapper (AutoMapper/Mapster) invoked on entities that touches unloaded navigations through a lazy-loading proxy, or a serializer enumerating lazy navigations. Also repository methods called inside a `foreach`.

**Explanation**
Also, explicit loading APIs (`Reference().Load()`) in loops. Fix by moving the data access out of the projection and using `ProjectTo`/server-translated projections.

**Code**
```csharp
// Hidden N+1: GetDiscount queries the DB for each row
var list = await db.Orders
    .Select(o => new { o.Id, Discount = discountService.GetDiscount(o.CustomerId) })
    .ToListAsync();

// Fix: join/precompute in one query
var discounts = await db.Discounts.ToDictionaryAsync(d => d.CustomerId, d => d.Percent);
var rows = await db.Orders.Select(o => new { o.Id, o.CustomerId }).ToListAsync();
var result = rows.Select(r => new { r.Id, Discount = discounts.GetValueOrDefault(r.CustomerId) });
```

**Gotchas / follow-ups**
- Fixing N+1 with a giant Include can swap it for cartesian explosion; measure both.
- `Contains` with thousands of ids: EF 8 sends it as a JSON parameter via `OPENJSON`, avoiding the 2,100-parameter limit.

**Interview traps**
- "N+1 is only caused by lazy loading."
- Fixing with Include on every query regardless of need.
- Not measuring: replacing N+1 with a 10M-row cartesian product.
- Ignoring that async doesn't reduce roundtrips.

**Cheat sheet**
- N+1 = 1 parent query + N child queries.
- Detect: logs, interceptor counters, APM, Query Store.
- Fix: Include, projection, batched `Contains`, split query.
- Lazy loading = opt-in proxies, virtual navs; avoid in APIs.
- Always verify the generated SQL after the fix.

---
## Topic 4: Projection with Select / DTOs

**Study notes**

- `Select` into a DTO/anonymous type makes EF select **only the needed columns**, can traverse navigations (EF adds the JOINs/subqueries), and the result is **not tracked** (non-entity types aren't tracked; entities nested inside an anonymous type *are* tracked unless `AsNoTracking`).
- Benefits: less IO, covering indexes can satisfy the query, no change-tracking overhead, no accidental lazy loading, stable API contracts.
- Use `new Dto { ... }` initializers or constructors; both translate. Methods in the projection run client-side (final Select only).
- AutoMapper `ProjectTo<T>` / Mapster `ProjectToType` build the `Select` expression for you.
- Don't `Include` with projection — it's ignored.

```
Entity query:  SELECT Id, Name, Bio, Photo(varbinary(max)), ... (20 cols)   + tracking snapshot
Projection:    SELECT Id, Name                                              (no tracking)
```

### 4.1 [Easy] Why project to a DTO instead of returning entities?
**Answer**
Projection fetches only the columns you need, skips change tracking, and decouples your API from the DB model. It also avoids over-posting/serialization cycles and lazy-loading surprises.

**Explanation**
Wide rows (blobs, long text) hurt. A projection also lets SQL Server use a narrower covering index.

**Code**
```csharp
var dtos = await db.Customers
    .Where(c => c.IsActive)
    .Select(c => new CustomerDto(c.Id, c.Name))
    .ToListAsync(ct);
```
```sql
SELECT [c].[Id], [c].[Name] FROM [Customers] AS [c] WHERE [c].[IsActive] = CAST(1 AS bit)
```

### 4.2 [Follow-up] Does a projection do tracking? What about an anonymous type containing an entity?
**Answer**
Pure DTO/scalar projections are not tracked. If the projection includes entity instances (e.g. `new { Order = o, Total = ... }`), those entities are tracked unless you use `AsNoTracking`.

**Explanation**
This surprises people who think "Select = no tracking". Use `AsNoTracking()` when the entity is only for reading.

**Code**
```csharp
var rows = await db.Orders
    .Select(o => new { Order = o, LineCount = o.Lines.Count }) // Order tracked
    .ToListAsync();
```

### 4.3 [Medium] How do navigations in a Select translate, and how do you avoid Include?
**Answer**
Navigating a reference becomes a JOIN; navigating a collection with an aggregate becomes a correlated subquery; selecting a nested collection into a list uses a JOIN with ordering (like Include) or subqueries. Because you pick exactly what you need, you rarely need Include.

**Explanation**
Nested collection projection (`Lines = o.Lines.Select(...).ToList()`) is translated similarly to Include (join + ordering and client shaping); multiple nested collections can cause the same row-multiplication as Include — `AsSplitQuery` applies to projections too.

**Code**
```csharp
var orders = await db.Orders.Select(o => new OrderDto
{
    Id = o.Id,
    Customer = o.Customer.Name,
    Lines = o.Lines.Select(l => new LineDto { Sku = l.Product.Sku, Qty = l.Qty }).ToList()
}).ToListAsync();
```
```sql
SELECT [o].[Id], [c].[Name], [p].[Sku], [l].[Qty]
FROM [Orders] o INNER JOIN [Customers] c ON o.CustomerId = c.Id
LEFT JOIN ([OrderLines] l INNER JOIN [Products] p ON l.ProductId = p.Id) ON o.Id = l.OrderId
ORDER BY o.Id, l.Id
```

### 4.4 [Hard] What are the pitfalls of AutoMapper ProjectTo and of mapping after materialization?
**Answer**
`ProjectTo` generates a translatable `Select` (good) but custom resolvers/`MapFrom` with C# methods can fail translation. Calling `Map` after `ToList()` loads entire entities and may trigger lazy-loading N+1.

**Explanation**
Always prefer `ProjectTo<TDto>(config)` on the `IQueryable`. Keep mapping expressions simple (member access, arithmetic, conditionals). Constructors with parameters are supported, but `ForMember(..., o => o.MapFrom(src => Helper(src.X)))` with arbitrary methods in non-final positions won't translate.

**Code**
```csharp
var dtos = await db.Orders
    .Where(o => o.CustomerId == id)
    .ProjectTo<OrderDto>(mapper.ConfigurationProvider)
    .ToListAsync(ct);
```

**Gotchas / follow-ups**
- Null-propagation (`o.Customer?.Name`) isn't allowed in expression trees; use `o.Customer.Name` (EF handles nulls with LEFT JOIN).
- Projecting `Include`-less entities into DTOs through a repository returning entities defeats the purpose.

### 4.5 [Tricky] When is projection NOT the better choice?
**Answer**
When you need to update the entity (you need tracked entities), or the aggregate is small and the projection would be reused across many places. Also when the projection shape is so complex that the generated SQL gets worse than a simple query plus in-memory composition.

**Explanation**
Update scenario: load the entity, mutate, `SaveChanges`. For partial updates without loading, use `Attach` + set modified properties or `ExecuteUpdate`. Always compare the SQL and plan.

**Code**
```csharp
// Update without loading the whole row
var stub = new Customer { Id = id, Name = newName };
db.Attach(stub);
db.Entry(stub).Property(c => c.Name).IsModified = true;
await db.SaveChangesAsync();
```
```sql
UPDATE [Customers] SET [Name] = @p0 WHERE [Id] = @p1;
```

**Gotchas / follow-ups**
- Attach with a stub and required properties: other columns are untouched since only `Name` is flagged modified.
- Concurrency tokens must also be set on the stub.

**Interview traps**
- Thinking `Select` always disables tracking.
- Using `Include` plus `Select`.
- Calling instance methods in the projection that are not final-Select-safe.
- Returning entities from controllers (cycles, over-posting, leaks of columns).

**Cheat sheet**
- Projection = fewer columns, no tracking (for non-entities), better indexes use.
- Navigations in `Select` → JOIN / subqueries.
- `ProjectTo` keeps mapping server-side.
- Entities inside anonymous types are still tracked.
- Use entities only when you need to update.

---

## Topic 5: Pagination (Skip/Take vs keyset) and Count performance

**Study notes**

- **Offset paging**: `OrderBy(...).Skip(n).Take(m)` → `ORDER BY ... OFFSET n ROWS FETCH NEXT m ROWS ONLY`. Needs a **deterministic ORDER BY** (add a unique tiebreaker like Id). Cost grows with offset — SQL Server must still read/skip `n` rows.
- **Keyset (seek) paging**: remember the last row's sort key(s) and query `WHERE key > @last ORDER BY key FETCH NEXT m` — constant cost with a supporting index; no "jump to page 500" and unstable under deletes only in the sense that it's *more stable* than offset (no skipped/duplicated rows on inserts).
- **Count**: `Count()` → `SELECT COUNT(*)`; for big tables it scans an index. Use `Any()` for existence (`SELECT CASE WHEN EXISTS`). A total count for every page doubles the cost; cache it, approximate it, or skip it ("has next page" by fetching `m+1`).
- `Count()` vs `LongCount()` → `COUNT` vs `COUNT_BIG`.

```
Offset: page 10000 -> read & discard 999,900 rows, return 100.  O(offset)
Keyset: index seek to (CreatedAt, Id) > (last)  -> read 100 rows. O(page)
```

### 5.1 [Easy] How do you paginate with EF Core?
**Answer**
Order the query deterministically and apply `Skip((page-1)*size).Take(size)`. EF translates it to `OFFSET/FETCH` on SQL Server.

**Explanation**
Without `OrderBy` the result order is undefined and EF warns (SQL Server requires ORDER BY for OFFSET; EF adds `ORDER BY (SELECT 1)`). Always include a unique column in the ordering.

**Code**
```csharp
var items = await db.Products
    .OrderBy(p => p.Name).ThenBy(p => p.Id)
    .Skip((page - 1) * size).Take(size)
    .AsNoTracking()
    .ToListAsync(ct);
```
```sql
SELECT [p].* FROM [Products] AS [p]
ORDER BY [p].[Name], [p].[Id]
OFFSET @__p_0 ROWS FETCH NEXT @__p_1 ROWS ONLY
```

### 5.2 [Follow-up] Why is Skip/Take slow on deep pages, and what is keyset pagination?
**Answer**
SQL Server must scan and discard all skipped rows, so cost is proportional to the offset. Keyset pagination filters by the last seen sort key (`WHERE key > last`) and uses an index seek, so each page costs the same.

**Explanation**
EF has no row-value comparison, so for a composite key (CreatedAt, Id) write the expanded predicate. The index must match the ORDER BY: `(CreatedAt DESC, Id DESC)`.

**Code**
```csharp
var q = db.Orders.AsNoTracking();
if (cursor is not null)
    q = q.Where(o => o.CreatedAt < cursor.CreatedAt
                  || (o.CreatedAt == cursor.CreatedAt && o.Id < cursor.Id));

var page = await q.OrderByDescending(o => o.CreatedAt).ThenByDescending(o => o.Id)
                  .Take(size + 1)   // +1 tells us if there is a next page
                  .ToListAsync(ct);
var hasNext = page.Count > size;
```
```sql
SELECT TOP(@size) ... FROM Orders o
WHERE o.CreatedAt < @c OR (o.CreatedAt = @c AND o.Id < @id)
ORDER BY o.CreatedAt DESC, o.Id DESC
```

### 5.3 [Medium] How do you make a paged API with total count efficient?
**Answer**
Avoid running a full `COUNT(*)` with heavy filters on every page request: run it only for the first page, cache it, or return `hasNext` via fetching one extra row. If total count is mandatory, make sure the filter is index-supported.

**Explanation**
Two queries (count + page) don't share a snapshot. A window function `COUNT(*) OVER()` per row isn't directly translatable in LINQ (use raw SQL). Count with joins from Includes is wasteful — count the base query without Includes/ordering.

**Code**
```csharp
var baseQ = db.Orders.Where(o => o.Status == status);   // no Include/OrderBy
var total = await baseQ.CountAsync(ct);
var items = await baseQ.OrderBy(o => o.Id).Skip(skip).Take(take).ToListAsync(ct);
```
```sql
SELECT COUNT(*) FROM [Orders] AS [o] WHERE [o].[Status] = @status;
```

### 5.4 [Hard] Any() vs Count() > 0, and Count vs LongCount — what does EF generate and why does it matter?
**Answer**
`Any()` becomes `EXISTS`, which stops at the first match; `Count() > 0` scans all matching rows. Use `Any()` for existence. `LongCount()` generates `COUNT_BIG` for results that may exceed `int.MaxValue`.

**Explanation**
In SQL Server, `COUNT(*)` can use the smallest nonclustered index. A `Count() == 0`/`> 0` check on a large filtered set is a classic perf bug. Also `FirstOrDefault` vs `Single`: `Single` fetches `TOP(2)` to verify uniqueness; `First` is `TOP(1)`.

**Code**
```csharp
bool exists = await db.Users.AnyAsync(u => u.Email == email, ct);
var user = await db.Users.SingleOrDefaultAsync(u => u.Id == id, ct);   // TOP(2)
```
```sql
SELECT CASE WHEN EXISTS (SELECT 1 FROM [Users] AS [u] WHERE [u].[Email] = @email) THEN CAST(1 AS bit) ELSE CAST(0 AS bit) END
```

**Gotchas / follow-ups**
- `FindAsync(id)` checks the change tracker first and avoids SQL if tracked.
- `Count()` on a heap or without a suitable index = table scan.

### 5.5 [Tricky] Why can offset paging return duplicate or missing rows, and what must you do with Skip/Take plus Include collections?
**Answer**
Non-unique or missing ordering makes row order non-deterministic, so rows can repeat across pages; concurrent inserts/deletes shift offsets. With collection `Include` in a single query, EF paginates the *parent* rows using a subquery, but with `AsSplitQuery` you must have a unique ordering so all splits see the same parent page.

**Explanation**
Ordering by a non-unique column like `CreatedAt` where many rows share the same value gives unstable pages. Keyset paging with a tiebreaker fixes it. For parent+children, paginate parents first (`Select(Ids)`), then load children for those IDs.

**Code**
```csharp
var ids = await db.Orders.OrderByDescending(o => o.CreatedAt).ThenBy(o => o.Id)
    .Skip(skip).Take(size).Select(o => o.Id).ToListAsync(ct);

var orders = await db.Orders.Include(o => o.Lines)
    .Where(o => ids.Contains(o.Id)).AsSplitQuery().ToListAsync(ct);
```

**Gotchas / follow-ups**
- Re-sort in memory by the id order from the first query (the second query has no guaranteed order).
- Skip/Take parameters are parameterized so the plan is reused; large `OFFSET` still hurts.

**Interview traps**
- Paging without a unique order key.
- Running `Count()` on a query with Includes/OrderBy.
- Using `Count() > 0` instead of `Any()`.
- Claiming keyset paging supports "jump to page N" (it doesn't).
- Calling `Skip/Take` after `ToList()`.

**Cheat sheet**
- Skip/Take → OFFSET/FETCH; needs deterministic ORDER BY.
- Keyset = `WHERE (sortKey) beyond last` + index; constant cost.
- `Any()` = EXISTS; `Count()` = full count; `LongCount` = COUNT_BIG.
- Fetch `size+1` for "has next page".
- Keep the paged index aligned with ORDER BY.

---

## Topic 6: GroupBy, joins, subqueries translation

**Study notes**

- `GroupBy` translates to SQL `GROUP BY` **only when the result projects the key and aggregates** (`Count`, `Sum`, `Min`, `Max`, `Average`) from the group. Selecting the group elements themselves (e.g. `g.ToList()`) isn't translatable in EF Core 3+ (older versions client-evaluated it).
- `Join`/`GroupJoin` → `INNER JOIN`/`LEFT JOIN`. Navigation properties are preferred over manual joins in most cases. `GroupJoin + SelectMany + DefaultIfEmpty` = left join (EF 10 adds `LeftJoin` operator).
- Collection navigation aggregates (`o.Lines.Sum(...)`) and `Any/All/Contains` become correlated subqueries or `EXISTS`.
- `Contains(list)` → `IN (...)` constants or, in EF 8 on SQL Server, a parameter sent as JSON with `OPENJSON`.
- Set ops: `Union`, `Concat`, `Intersect`, `Except` translate. `Distinct` → `DISTINCT`.
- Window functions (`ROW_NUMBER`) aren't first-class LINQ in EF 8; use correlated subqueries, `FromSql`, or SQL views.

### 6.1 [Easy] How does GroupBy translate in EF Core?
**Answer**
When you group and project the key with aggregates, EF generates `GROUP BY` with aggregate functions in the SELECT list. Group by multiple columns using an anonymous type key.

**Explanation**
The result is a flat set of rows, so it's efficient. A `HAVING` clause is produced when you `Where` on the aggregate after `GroupBy`.

**Code**
```csharp
var stats = await db.Orders
    .GroupBy(o => o.CustomerId)
    .Where(g => g.Count() > 5)
    .Select(g => new { CustomerId = g.Key, Orders = g.Count(), Total = g.Sum(o => o.Total) })
    .ToListAsync();
```
```sql
SELECT [o].[CustomerId], COUNT(*) AS [Orders], SUM([o].[Total]) AS [Total]
FROM [Orders] AS [o]
GROUP BY [o].[CustomerId]
HAVING COUNT(*) > 5
```

### 6.2 [Follow-up] When do you use Join vs navigation properties vs GroupJoin?
**Answer**
Use navigation properties when a relationship is mapped — EF generates the JOIN for you. Use explicit `Join` for unrelated entities without mapped relationships, and `GroupJoin`+`DefaultIfEmpty` for left joins.

**Explanation**
Navigation use reads cleaner and benefits from model-level configuration (required → INNER JOIN, optional → LEFT JOIN).

**Code**
```csharp
var q = from c in db.Customers
        join o in db.Orders on c.Id equals o.CustomerId into grp
        from o in grp.DefaultIfEmpty()
        select new { c.Name, OrderId = (int?)o.Id };
```
```sql
SELECT [c].[Name], [o].[Id] FROM [Customers] AS [c] LEFT JOIN [Orders] AS [o] ON [c].[Id] = [o].[CustomerId]
```

### 6.3 [Medium] How do you get "latest/top-N per group" and what happens if you try `GroupBy(...).Select(g => g.OrderBy(...).First())`?
**Answer**
Prefer a correlated subquery from the parent entity: `customers.Select(c => c.Orders.OrderByDescending(o => o.CreatedAt).FirstOrDefault())`. Group-then-take-first patterns may translate in newer EF versions only for some shapes; if it fails with a translation error, use the correlated approach or raw SQL with `ROW_NUMBER()`.

**Explanation**
Always inspect the generated SQL; correlated `TOP(1)` with `OUTER APPLY`/subselect is index-friendly with an index on `(CustomerId, CreatedAt DESC)`.

**Code**
```csharp
var latest = await db.Customers
    .Select(c => new
    {
        c.Id,
        LastOrder = c.Orders.OrderByDescending(o => o.CreatedAt)
                            .Select(o => new { o.Id, o.CreatedAt })
                            .FirstOrDefault()
    }).ToListAsync();
```
```sql
SELECT [c].[Id], [t0].[Id], [t0].[CreatedAt]
FROM [Customers] AS [c]
LEFT JOIN (
    SELECT [t].[Id], [t].[CreatedAt], [t].[CustomerId]
    FROM (SELECT [o].*, ROW_NUMBER() OVER(PARTITION BY [o].[CustomerId] ORDER BY [o].[CreatedAt] DESC) AS [row] FROM [Orders] AS [o]) AS [t]
    WHERE [t].[row] <= 1
) AS [t0] ON [c].[Id] = [t0].[CustomerId]
```

### 6.4 [Hard] How does `Contains` with a list translate, and what limits apply?
**Answer**
Before EF 8, a list was inlined as `IN (1,2,3)` constants (or parameters), hitting plan-cache bloat and the 2,100-parameter limit. EF Core 8 on SQL Server by default sends the list as one JSON parameter and uses `OPENJSON`, so one plan is reused and parameter limits are avoided.

**Explanation**
Trade-off: `OPENJSON` gives the optimizer poor cardinality estimates for the JSON rowset (often fixed guess), which can produce bad plans for big lists; EF 8 lets you revert via `UseSqlServer(o => o.UseCompatibilityLevel(120))`. Don't pass tens of thousands of IDs — use a temp/table-valued approach or join to a staging table.

**Code**
```csharp
var ids = new[] { 1, 2, 3 };
var items = await db.Products.Where(p => ids.Contains(p.Id)).ToListAsync();
```
```sql
SELECT [p].[Id], [p].[Name] FROM [Products] AS [p]
WHERE [p].[Id] IN (SELECT [i].[value] FROM OPENJSON(@__ids_0) WITH ([value] int '$') AS [i])
```

**Gotchas / follow-ups**
- Empty list → `WHERE 0 = 1`-like short circuit, fine.
- If using older compatibility level the translation is different; verify in the logs.

### 6.5 [Tricky] Why can a query with `GroupBy` return different results client vs server, and what is the trap with `string.Join`/`ToList` inside GroupBy?
**Answer**
Grouping semantics are the same, but anything that needs the group's elements (e.g. `g.Select(...).ToList()`, `string.Join`) cannot be translated as a plain `GROUP BY` and either throws or relies on a split into key query + element query depending on shape. Case-sensitivity also differs: SQL Server collation (case-insensitive) groups "a" and "A" together, while C# `GroupBy` doesn't.

**Explanation**
Collation-driven semantics are a classic difference between LINQ-to-Objects and LINQ-to-SQL. Comparison with `==` on strings uses DB collation in SQL but ordinal in memory. For grouping and join keys, consider whether trailing spaces and accent/case matter.

**Code**
```csharp
// SQL (CI collation): "Ann" and "ann" are one group.
var q = db.Users.GroupBy(u => u.Name).Select(g => new { g.Key, N = g.Count() });

// In memory: two groups.
var m = db.Users.AsEnumerable().GroupBy(u => u.Name).Select(g => new { g.Key, N = g.Count() });
```

**Gotchas / follow-ups**
- Fix by specifying collation explicitly (`EF.Functions.Collate`, EF 5+) or normalizing data.
- Trailing spaces: SQL Server ignores trailing spaces in `=` comparisons.

**Interview traps**
- Expecting `GroupBy(...).ToList()` of full group elements to run in SQL.
- Hand-writing joins when navigations exist.
- Forgetting that DB collation drives string equality.
- Large `Contains` lists (and the 2,100 parameter limit pre-EF8).

**Cheat sheet**
- GroupBy + aggregate projection → GROUP BY/HAVING.
- Left join = GroupJoin + SelectMany + DefaultIfEmpty.
- Collection aggregates → correlated subqueries; `Any` → EXISTS.
- EF 8 `Contains(list)` → `OPENJSON` on SQL Server.
- Top-1 per group: correlated `OrderBy.FirstOrDefault()` or raw SQL with ROW_NUMBER.

---
## Topic 7: Raw SQL, stored procedures, views, SQL-injection safety

**Study notes**

| API (EF Core 7/8) | Returns | Use for |
|---|---|---|
| `DbSet.FromSql($"...")` / `FromSqlInterpolated` | `IQueryable<TEntity>` | Entity queries; **interpolated values become parameters** |
| `DbSet.FromSqlRaw(sql, params)` | `IQueryable<TEntity>` | Dynamic SQL text — **you** must parameterize |
| `Database.SqlQuery<T>($"...")` | `IQueryable<T>` (T = scalar in EF7; any unmapped type in EF 8) | Non-entity results, DTOs |
| `Database.ExecuteSql($"...")` / `ExecuteSqlInterpolated` / `ExecuteSqlRaw` | `int` rows affected | INSERT/UPDATE/DELETE/DDL |

- `FromSql` must return **all columns** of the entity type with matching names; it can be **composed** with LINQ (`Where`, `OrderBy`, `Include`) only when the SQL is a plain `SELECT` (EF wraps it in a subquery). Stored procedure calls (`EXEC`) and anything with `ORDER BY`/CTE at top level can't be composed (use `AsEnumerable()` after).
- Views: map with `ToView("vw_X")` and usually `HasNoKey()` (keyless entity). Keyless entities are never tracked. Migrations skip views.
- **SQL injection**: `FromSql`/`ExecuteSql` (interpolated) are safe because each `{value}` becomes a `DbParameter`. `FromSqlRaw($"...{x}...")` with a C# interpolated string is the dangerous mistake (string built before EF sees it).
- Identifiers (table/column names, ORDER BY direction) can't be parameters — whitelist them.

### 7.1 [Easy] How do you run raw SQL in EF Core, safely?
**Answer**
Use `FromSql` / `FromSqlInterpolated` for entity queries and `Database.ExecuteSql` for commands; interpolated values are converted to SQL parameters automatically, preventing injection.

**Explanation**
In EF Core 7+, `FromSql($"...")` takes a `FormattableString`; the compiler passes the format and arguments separately, so EF builds `@p0`.

**Code**
```csharp
var name = "Ann";
var customers = await db.Customers
    .FromSql($"SELECT * FROM Customers WHERE Name = {name}")
    .Where(c => c.IsActive)          // composable
    .ToListAsync(ct);
```
```sql
SELECT [c].* FROM (SELECT * FROM Customers WHERE Name = @p0) AS [c] WHERE [c].[IsActive] = CAST(1 AS bit)
```

### 7.2 [Follow-up] What's the difference between FromSql, FromSqlInterpolated and FromSqlRaw?
**Answer**
`FromSql` (EF 7+) and `FromSqlInterpolated` both take interpolated strings and parameterize them. `FromSqlRaw` takes a plain string plus explicit parameters and is only safe if you never concatenate user input into the string.

**Explanation**
`FromSql` in EF 7+ is the preferred spelling; analyzers (EF1002) warn when risky concatenation reaches `FromSqlRaw`/`ExecuteSqlRaw`.

**Code**
```csharp
// SAFE
await db.Customers.FromSqlRaw("SELECT * FROM Customers WHERE Name = {0}", name).ToListAsync();
await db.Customers.FromSqlRaw("SELECT * FROM Customers WHERE Name = @n", new SqlParameter("n", name)).ToListAsync();

// UNSAFE - C# interpolation builds the string first
await db.Customers.FromSqlRaw($"SELECT * FROM Customers WHERE Name = '{name}'").ToListAsync();
```

### 7.3 [Medium] How do you call stored procedures and map non-entity results?
**Answer**
Use `FromSql($"EXEC dbo.GetOrders {id}")` for entity-shaped results (non-composable), `Database.SqlQuery<T>` for DTOs/scalars (EF 8 supports arbitrary unmapped types), and `ExecuteSql` for procs with no result set. Output parameters require `SqlParameter` with `Direction = Output` and `FromSqlRaw/ExecuteSqlRaw`.

**Explanation**
For results with no matching entity, either map a keyless entity (`HasNoKey().ToView(null)`) or use `SqlQuery<T>`. With `SqlQuery<T>` scalar results must be aliased `Value`. EF 8 `SqlQuery` can be composed with LINQ.

**Code**
```csharp
var dtos = await db.Database
    .SqlQuery<OrderSummaryDto>($"EXEC dbo.GetOrderSummary {customerId}")
    .ToListAsync(ct);

var ids = await db.Database.SqlQuery<int>($"SELECT Id AS Value FROM Orders WHERE Status = {status}")
    .ToListAsync(ct);

var rows = await db.Database.ExecuteSqlAsync($"EXEC dbo.ArchiveOrders {cutoff}", ct);

var total = new SqlParameter("@total", SqlDbType.Int) { Direction = ParameterDirection.Output };
await db.Database.ExecuteSqlRawAsync("EXEC dbo.CountOrders @customerId, @total OUTPUT",
    new SqlParameter("@customerId", id), total);
```

### 7.4 [Hard] How do views and keyless entities work in EF Core, and what are the limitations?
**Answer**
A view is mapped with `ToView("vw")`; typically as a keyless entity (`HasNoKey`) which is never tracked and can't be updated through EF. Migrations don't create the view — you create it with `migrationBuilder.Sql` or raw scripts.

**Explanation**
Keyless entities can't participate as dependents in relationships in the same way, have no identity resolution, and `Include` of them from keyed entities is limited (navigations *to* keyless entities are allowed one-way from an entity). If the view has a unique key you can map it as a normal entity with `ToView` and still avoid writes by never calling SaveChanges on it (EF doesn't prevent writes — the DB will fail).

**Code**
```csharp
modelBuilder.Entity<OrderSummary>(e =>
{
    e.HasNoKey();
    e.ToView("vw_OrderSummary");
});

// Migration: create the view
migrationBuilder.Sql("CREATE VIEW dbo.vw_OrderSummary AS SELECT o.Id AS OrderId, SUM(l.Qty*l.UnitPrice) AS Total FROM Orders o JOIN OrderLines l ON l.OrderId=o.Id GROUP BY o.Id");

var rows = await db.OrderSummaries.Where(s => s.Total > 1000).ToListAsync();
```

**Gotchas / follow-ups**
- Mapping the same entity to both a table (`ToTable`) and view (`ToView`) is allowed: use view for reads, table for writes (EF 5+).
- Indexed (materialized) views have strict requirements (SCHEMABINDING).

### 7.5 [Tricky] Why does `FromSql` + `Include` or `OrderBy` sometimes fail, and how can dynamic ORDER BY be made safe?
**Answer**
EF wraps the raw SQL as a subquery, so SQL that ends in `ORDER BY`, uses `EXEC`, or has a trailing CTE/semicolon is not valid for composition. Dynamic column names can't be parameterized; compose with LINQ (`OrderBy` using a whitelist mapping) instead of concatenating.

**Explanation**
Put ordering in LINQ after `FromSql`. If you must, use `AsEnumerable()` to stop composition (client side). For stored procs, finish with `ToListAsync` immediately.

**Code**
```csharp
var sortMap = new Dictionary<string, Expression<Func<Customer, object>>>
{
    ["name"] = c => c.Name, ["created"] = c => c.CreatedAt
};
if (!sortMap.TryGetValue(sortBy, out var key)) key = sortMap["name"];

var q = db.Customers.FromSql($"SELECT * FROM Customers WHERE Region = {region}")
                    .OrderBy(key);
```

**Gotchas / follow-ups**
- `OrderBy(Expression<Func<T,object>>)` boxes value types → EF handles the `Convert` in many cases, but for value types verify translation, or build per-type expressions.
- `LIKE` patterns: still parameters, but escape `%`/`_` from user input.
- Never log parameter values in prod (`EnableSensitiveDataLogging` off).

**Interview traps**
- `FromSqlRaw($"...{x}")` — looks parameterized, isn't.
- Expecting `Include` to work on a stored-procedure call.
- Forgetting `FromSql` must return every column of the entity.
- Thinking views are created/updated by migrations automatically.

**Cheat sheet**
- Interpolated `FromSql`/`ExecuteSql` = parameterized = safe.
- `FromSqlRaw` = you own parameterization.
- `SqlQuery<T>` for DTOs (EF 8 arbitrary types), scalar alias `Value`.
- SP results aren't composable; plain SELECTs are.
- Views: `ToView` + `HasNoKey`, create via `migrationBuilder.Sql`.

---

## Topic 8: Compiled queries & query caching

**Study notes**

- Each LINQ query goes through: expression tree → **parameter extraction** (closures become `@__p`) → **query cache lookup** (keyed by the parameterized expression shape) → compile to SQL + shaper (expensive, first time only) → execute.
- EF Core **already caches** compiled queries internally, but still pays the cost of **expression tree building + cache-key hashing/comparison** per call.
- `EF.CompileQuery` / `EF.CompileAsyncQuery` produce a static delegate that **skips** the tree walk; gains are measurable only in very hot paths (microseconds). Pass the context and parameters as delegate args.
- Compiled queries can return `IEnumerable<T>`/`IAsyncEnumerable<T>`, `Task<T>` (single) — not `IQueryable` — so no further composition.
- SQL Server also has its own **plan cache**: parameterized queries reuse plans; constants (`EF.Constant`) create separate plans. **Parameter sniffing** can make one cached plan bad for other values.
- Model/service-provider caching: don't create a new `ServiceProvider`/options per context (e.g. `UseInternalServiceProvider` misuse or per-request changing config) — it causes "ManyServiceProvidersCreatedWarning" and memory growth.

### 8.1 [Easy] What is a compiled query in EF Core?
**Answer**
It's a precompiled query delegate created with `EF.CompileQuery`/`EF.CompileAsyncQuery`. It caches the translation work so repeated calls skip building and analyzing the expression tree.

**Explanation**
Define it as a `static readonly` field once; call it with a context and parameters.

**Code**
```csharp
private static readonly Func<AppDb, int, Task<Customer?>> GetCustomerById =
    EF.CompileAsyncQuery((AppDb db, int id) =>
        db.Customers.AsNoTracking().FirstOrDefault(c => c.Id == id));

var customer = await GetCustomerById(db, 42);
```
```sql
SELECT TOP(1) [c].[Id], [c].[Name] FROM [Customers] AS [c] WHERE [c].[Id] = @__id_0
```

### 8.2 [Follow-up] Does EF Core cache queries without CompileQuery? What's the benefit of explicitly compiling?
**Answer**
Yes, EF caches the compiled query by expression shape in an internal memory cache; only the first execution of a shape pays the translation cost. `CompileQuery` saves the remaining per-call overhead (building the tree, parameter extraction, cache lookup), which only matters for very hot, very cheap queries.

**Explanation**
Benchmark before using — for typical queries DB time dominates. Different closure *values* reuse the cache; different query *shapes* (e.g. conditionally composed `Where`s) create new entries.

**Code**
```csharp
// Same shape, different values -> one cache entry
db.Orders.Where(o => o.CustomerId == a);
db.Orders.Where(o => o.CustomerId == b);
```

### 8.3 [Medium] What are the limitations of compiled queries?
**Answer**
They can't be composed after compilation, the query shape is fixed (no conditional Where inside), the result must be materialized (`IEnumerable`/`IAsyncEnumerable`/single value), and they have a limited number of parameters supported by the delegate overloads. Dynamic filtering needs several compiled queries or non-compiled queries.

**Explanation**
For streaming results use `IAsyncEnumerable<T>` return and `await foreach`. For cancellation use the `CancellationToken` as an extra parameter (EF Core 8 supports it in compiled async queries).

**Code**
```csharp
private static readonly Func<AppDb, int, IAsyncEnumerable<Order>> OrdersByCustomer =
    EF.CompileAsyncQuery((AppDb db, int customerId) =>
        db.Orders.Where(o => o.CustomerId == customerId));

await foreach (var o in OrdersByCustomer(db, id)) { /* stream */ }
```

### 8.4 [Hard] How do query caching and SQL Server plan caching interact, and what is parameter sniffing in this context?
**Answer**
EF's cache holds the translated SQL text; SQL Server caches an execution plan per distinct SQL text. EF sends parameterized SQL, so one plan is compiled for the first parameter values (sniffed) and reused — good for reuse, bad when data skew makes that plan poor for other values.

**Explanation**
Mitigations: `OPTION (RECOMPILE)` via a command interceptor or raw SQL, `OPTIMIZE FOR UNKNOWN`, better statistics/indexes, query hints with Query Store, splitting hot/cold values into different queries. `EF.Constant(x)` inlines the value so each value gets its own plan (risk of plan cache bloat and SQL-injection concerns for strings are handled by EF escaping, but use sparingly).

**Code**
```csharp
// Different plan per literal; use only for low-cardinality skewed values
var q = db.Orders.Where(o => o.Status == EF.Constant(OrderStatus.Cancelled));

public class RecompileInterceptor : DbCommandInterceptor
{
    public override InterceptionResult<DbDataReader> ReaderExecuting(
        DbCommand c, CommandEventData e, InterceptionResult<DbDataReader> r)
    {
        if (c.CommandText.StartsWith("-- RECOMPILE", StringComparison.Ordinal))
            c.CommandText += " OPTION (RECOMPILE)";
        return r;
    }
}

db.Orders.TagWith("RECOMPILE").Where(...); // TagWith emits "-- RECOMPILE" before the SQL
```

**Gotchas / follow-ups**
- `TagWith` makes the SQL text unique per tag (separate plan/cache entry) — keep tags static.
- Prefer fixing indexes/stats before hints.

### 8.5 [Tricky] What causes EF's internal query cache or service provider to grow without bound?
**Answer**
Building queries that differ in *shape* each time (e.g. dynamically generated expression trees with constants, inlining values with `EF.Constant`, or embedding a captured value as a constant instead of a closure) creates new cache entries per variation. Also creating a new internal service provider per DbContext (`UseInternalServiceProvider` misuse) or per-request `OnConfiguring` model-affecting options causes repeated model building.

**Explanation**
Parameterize through closures/local variables (EF turns them into parameters). A `Contains` with a *constant array* creates a different expression shape than a captured variable. Watch `ManyServiceProvidersCreatedWarning` in logs.

**Code**
```csharp
// Good: closure variable -> parameter, same cache entry
var s = status;
db.Orders.Where(o => o.Status == s);

// Bad pattern: building Expression.Constant(value) dynamically each call
var p = Expression.Parameter(typeof(Order));
var body = Expression.Equal(Expression.Property(p, "Status"), Expression.Constant(status)); // constant -> distinct cache key per value
```

**Gotchas / follow-ups**
- `DbContext` pooling (`AddDbContextPool`) reuses instances; context state must be resettable and avoid per-request fields not reset (use `IDbContextPoolable` rules; scoped services can't be injected into a pooled context).
- Model cache is keyed by context type + `IModelCacheKeyFactory`; per-tenant model changes need a custom key factory.

**Interview traps**
- "EF re-translates every query every time."
- Compiling queries for everything without benchmarking.
- Thinking compiled queries return `IQueryable`.
- Hardcoding values as constants → plan cache bloat; or ignoring parameter sniffing.

**Cheat sheet**
- EF caches the translation by expression shape; closures become parameters.
- `EF.CompileAsyncQuery` = static delegate, skips tree work; non-composable.
- SQL Server plan cache is separate; parameter sniffing can bite.
- `EF.Constant` → literal in SQL (own plan).
- Don't create service providers per context.

---

## Topic 9: Bulk operations (ExecuteUpdate/Delete, AddRange, bulk libs), tracking costs & tuning checklist

**Study notes**

- **`ExecuteUpdate` / `ExecuteDelete`** (EF Core 7+): set-based `UPDATE`/`DELETE` executed **immediately**, one SQL statement, **no tracking, no SaveChanges**, no loading. They do **not** update already-tracked entities in memory, do **not** check concurrency tokens, and don't trigger `SaveChanges` interceptors (command interceptors still run). They are not auto-wrapped with other operations — use an explicit transaction to combine. EF 8 allows setting properties with non-constant composed setters and ExecuteUpdate on TPT/owned/complex mappings has limits (TPT/TPC multi-table not supported for update spanning tables).
- **SaveChanges batching**: EF groups INSERT/UPDATE/DELETE into batches (SQL Server default `MaxBatchSize` 42, and up to ~2,100 parameters) and sends them in one roundtrip; inserts with generated keys use `MERGE`/`INSERT ... OUTPUT`. `AddRange` doesn't change batching semantics; it just avoids repeated `DetectChanges`.
- **Bulk libraries**: `EFCore.BulkExtensions`, `Z.EntityFramework.Extensions` (commercial), or `SqlBulkCopy` directly (fastest for inserts; bypasses EF, no change tracking).
- **Tracking cost**: change tracker keeps a snapshot per entity and runs `DetectChanges` on `SaveChanges` (O(n)). Large tracked graphs = memory + CPU.

```
Tuning checklist: measure first (logs/Query Store) ->
 1 AsNoTracking for reads   2 Project only needed columns   3 Filter/page in SQL
 4 Right indexes (cover, filtered)  5 Avoid N+1 / cartesian  6 Batch writes
 7 ExecuteUpdate/Delete for set-based  8 Short-lived contexts, pooling
 9 Compiled queries only for hot paths  10 Async + cancellation, no sync-over-async
```

### 9.1 [Easy] What do ExecuteUpdate and ExecuteDelete do?
**Answer**
They run a set-based `UPDATE`/`DELETE` directly in the database, without loading entities or using the change tracker. They execute immediately, returning the number of affected rows.

**Explanation**
Ideal for "mark all expired as inactive" or purge jobs. Because they run in a single statement they're orders of magnitude faster than load–modify–save for large sets.

**Code**
```csharp
var n = await db.Orders
    .Where(o => o.Status == OrderStatus.Pending && o.CreatedAt < cutoff)
    .ExecuteUpdateAsync(s => s
        .SetProperty(o => o.Status, OrderStatus.Expired)
        .SetProperty(o => o.UpdatedAt, DateTime.UtcNow), ct);

await db.AuditLogs.Where(a => a.CreatedAt < cutoff).ExecuteDeleteAsync(ct);
```
```sql
UPDATE [o] SET [o].[Status] = 3, [o].[UpdatedAt] = @p FROM [Orders] AS [o]
WHERE [o].[Status] = 0 AND [o].[CreatedAt] < @cutoff;
DELETE FROM [a] FROM [AuditLogs] AS [a] WHERE [a].[CreatedAt] < @cutoff;
```

### 9.2 [Follow-up] What are the gotchas of ExecuteUpdate/ExecuteDelete compared with SaveChanges?
**Answer**
They bypass the change tracker (tracked entities become stale), skip concurrency-token checks, don't run `SaveChanges` interceptors/auditing/soft-delete logic, and aren't batched with other changes — each call is its own statement and, unless in a transaction, its own implicit transaction.

**Explanation**
After `ExecuteUpdate`, tracked instances still hold old values; use `ChangeTracker.Clear()` or reload. Cascade behavior is DB-level only (EF doesn't cascade client-side); with `ExecuteDelete` on a principal, the database FK cascade must be configured or it fails with an FK violation. Row `rowversion` is auto-updated by SQL Server but EF doesn't verify the original.

**Code**
```csharp
await using var tx = await db.Database.BeginTransactionAsync(ct);
await db.Orders.Where(o => o.CustomerId == id).ExecuteDeleteAsync(ct);
await db.Customers.Where(c => c.Id == id).ExecuteDeleteAsync(ct);
await tx.CommitAsync(ct);
```

### 9.3 [Medium] How does SaveChanges batching work and how do you insert 100k rows efficiently?
**Answer**
EF batches multiple statements per roundtrip using parameterized commands (limited by `MaxBatchSize` and SQL Server's 2,100-parameter limit). For 100k rows: use `AddRange` in chunks (e.g. 1–5k), clear the tracker after each chunk, disable auto-detect, or use `SqlBulkCopy`/`BulkExtensions` for the fastest path.

**Explanation**
Single big SaveChanges holds a huge tracked graph and transaction. Chunking keeps memory bounded. If you need generated IDs back, EF/`BulkExtensions` handle it; raw `SqlBulkCopy` doesn't. Consider `ChangeTracker.AutoDetectChangesEnabled = false` for pure inserts.

**Code**
```csharp
db.ChangeTracker.AutoDetectChangesEnabled = false;
foreach (var chunk in rows.Chunk(2000))
{
    db.Products.AddRange(chunk);
    await db.SaveChangesAsync(ct);
    db.ChangeTracker.Clear();
}

// Third-party alternative (EFCore.BulkExtensions)
await db.BulkInsertAsync(rows, cancellationToken: ct);
```

### 9.4 [Hard] What are the costs of change tracking and what's your read/write tuning checklist?
**Answer**
Tracking costs memory (entity + snapshot per tracked instance), identity-resolution lookups, and `DetectChanges` cost proportional to tracked entity count on each SaveChanges. Use `AsNoTracking` for reads, short-lived contexts, projection, and clear the tracker in long loops.

**Explanation**
Checklist: (1) log slow queries; (2) `AsNoTracking`/projection; (3) eliminate N+1 & cartesian; (4) check execution plans, add covering/filtered indexes, avoid functions on indexed columns (non-sargable like `.Date`, `ToLower()`, `Substring`); (5) paginate; (6) batch writes / ExecuteUpdate; (7) keep transactions short; (8) `AddDbContextPool` for high throughput; (9) turn off `AutoDetectChanges` for bulk loads; (10) set default tracking via `UseQueryTrackingBehavior(QueryTrackingBehavior.NoTrackingWithIdentityResolution)` for read services.

**Code**
```csharp
services.AddDbContext<ReadDb>(o => o
    .UseSqlServer(cs)
    .UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking));

// Non-sargable vs sargable
var bad  = db.Orders.Where(o => o.CreatedAt.Date == day);                 // CONVERT(date, ...) - scan
var good = db.Orders.Where(o => o.CreatedAt >= day && o.CreatedAt < day.AddDays(1));
```

**Gotchas / follow-ups**
- `AsNoTracking` + modify + `SaveChanges` does nothing (not tracked).
- `ToLower()` in predicates defeats indexes and is unnecessary with CI collation.

### 9.5 [Tricky] An import job using SaveChanges in a loop grows to GBs of RAM and gets slower over time. Why and how do you fix it?
**Answer**
The same long-lived DbContext keeps every entity it ever saw tracked, so memory grows and `DetectChanges` cost increases per iteration. Fix by using a new (or `Clear()`ed) context per batch, `AsNoTracking` for reads, chunking, and set-based operations where possible.

**Explanation**
Also `AddRange` of already-tracked graphs repeatedly rescans navigations. Scoped DI contexts in background services (a singleton `BackgroundService` holding one context) are a typical cause — create scopes with `IServiceScopeFactory` or use `IDbContextFactory<T>`.

**Code**
```csharp
foreach (var batch in source.Chunk(1000))
{
    await using var ctx = await factory.CreateDbContextAsync(ct);
    ctx.Items.AddRange(batch.Select(Map));
    await ctx.SaveChangesAsync(ct);
}
```

**Gotchas / follow-ups**
- Don't `ChangeTracker.Clear()` while holding entities you still intend to save.
- Bulk-copy bypasses triggers unless `FIRE_TRIGGERS` is set and bypasses EF validations/interceptors.

**Interview traps**
- Expecting `ExecuteUpdate` to update in-memory entities or check rowversion.
- Assuming `AddRange` is a bulk insert.
- Applying `AsNoTracking` and then `SaveChanges`.
- Optimising without measuring (premature tuning).
- `.Date`, `ToLower()`, `Contains` leading wildcards making predicates non-sargable.

**Cheat sheet**
- `ExecuteUpdate/Delete` = set-based, immediate, untracked, no interceptors/concurrency.
- SaveChanges batches commands (default max 42 statements/batch on SQL Server).
- Bulk load: chunk + clear tracker, or SqlBulkCopy/BulkExtensions.
- Reads: `AsNoTracking` + projection + paging + indexes.
- Long jobs: new context per batch (`IDbContextFactory`).

---
## Topic 10: Concurrency control (rowversion, DbUpdateConcurrencyException)

**Study notes**

- **Optimistic concurrency**: don't lock; detect that someone else changed the row since you read it. EF adds the original token value to the `WHERE` clause of `UPDATE`/`DELETE`; if **0 rows** are affected, it throws `DbUpdateConcurrencyException`.
- Token options: `[Timestamp] byte[] RowVersion` / `.IsRowVersion()` (SQL Server `rowversion`, DB-generated on every update — preferred), or `[ConcurrencyCheck]` / `.IsConcurrencyToken()` on ordinary columns (EF compares original values of those columns).
- Handling strategies: **store wins** (discard the client's changes, reload), **client wins** (overwrite with the client's values, retry with refreshed original token), **merge** (per-property decision/ask user).
- Exception exposes `Entries` (the `EntityEntry`s that failed); `entry.GetDatabaseValues()` returns current DB values (null if the row was deleted); `entry.OriginalValues` / `CurrentValues`.
- In disconnected APIs (ETag/If-Match), set the original token from the client: `entry.Property(e => e.RowVersion).OriginalValue = dto.RowVersion;`.

```
T1 reads Row(v=5)       T2 reads Row(v=5)
T1: UPDATE ... WHERE Id=1 AND RowVersion=5  -> 1 row, v=6
T2: UPDATE ... WHERE Id=1 AND RowVersion=5  -> 0 rows -> DbUpdateConcurrencyException
```

### 10.1 [Easy] How do you configure optimistic concurrency in EF Core?
**Answer**
Add a `rowversion` property marked `[Timestamp]` (or `.IsRowVersion()`), or mark specific properties as concurrency tokens. EF includes the token in the WHERE of updates/deletes and throws `DbUpdateConcurrencyException` when no row matches.

**Explanation**
`rowversion` changes automatically on every row modification and covers all columns, so you don't need to enumerate fields.

**Code**
```csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    [Timestamp] public byte[] RowVersion { get; set; } = null!;
}

modelBuilder.Entity<Product>().Property(p => p.RowVersion).IsRowVersion();
```
```sql
UPDATE [Products] SET [Name] = @p0
OUTPUT INSERTED.[RowVersion]
WHERE [Id] = @p1 AND [RowVersion] = @p2;
```

### 10.2 [Follow-up] What happens on conflict and how do you handle it?
**Answer**
`SaveChanges` throws `DbUpdateConcurrencyException`. Inspect the conflicting entries, load current DB values, and apply your strategy: store-wins (reload and tell the user), client-wins (overwrite) or merge, then retry (usually bounded retries).

**Explanation**
Never swallow the exception — that silently loses updates. In web APIs, map it to HTTP 409 (or 412 with `If-Match`).

**Code**
```csharp
try { await db.SaveChangesAsync(ct); }
catch (DbUpdateConcurrencyException ex)
{
    foreach (var entry in ex.Entries)
    {
        var dbValues = await entry.GetDatabaseValuesAsync(ct);
        if (dbValues is null) { /* row deleted by someone else */ return Results.NotFound(); }

        // Client wins: take DB's token as the new "original", keep my current values
        entry.OriginalValues.SetValues(dbValues);
    }
    await db.SaveChangesAsync(ct);   // retry once (guard with a loop limit)
}
```

### 10.3 [Medium] Implement store-wins and a property-level merge.
**Answer**
Store wins: `await entry.ReloadAsync()` discards local changes. Merge: compare original, current and database values per property; keep the user's change only for properties they changed and take the database value for others, then set the original values to the database values and save.

**Explanation**
`entry.Properties` enumerates `PropertyEntry` with `OriginalValue`, `CurrentValue`, `IsModified`.

**Code**
```csharp
catch (DbUpdateConcurrencyException ex)
{
    var entry = ex.Entries.Single();
    var dbValues = await entry.GetDatabaseValuesAsync(ct);

    if (dbValues is null) throw;                 // deleted

    foreach (var p in entry.Properties.Where(p => !p.Metadata.IsPrimaryKey()))
    {
        var mine = p.CurrentValue; var original = p.OriginalValue; var theirs = dbValues[p.Metadata];
        if (!Equals(mine, original) && !Equals(original, theirs)) { /* true conflict: pick or ask user */ }
        else if (Equals(mine, original)) p.CurrentValue = theirs;  // I didn't change it -> take theirs
    }
    entry.OriginalValues.SetValues(dbValues);   // refresh token
    await db.SaveChangesAsync(ct);
}

// Store wins
await entry.ReloadAsync(ct);
```

### 10.4 [Hard] How do you apply concurrency checks in a disconnected API (DTO from client)?
**Answer**
Return the rowversion to the client (Base64 or ETag); when updating, load the entity and assign the client's value to the property's `OriginalValue` before saving. EF will use it in the WHERE, so a stale token raises the exception.

**Explanation**
Just copying the DTO's `RowVersion` into `entity.RowVersion` doesn't work: EF uses the *original* value (from the load) for the WHERE. You must set `OriginalValue` explicitly.

**Code**
```csharp
var product = await db.Products.FindAsync(new object[] { dto.Id }, ct);
if (product is null) return Results.NotFound();

db.Entry(product).Property(p => p.RowVersion).OriginalValue = dto.RowVersion;
product.Name = dto.Name;

try { await db.SaveChangesAsync(ct); }
catch (DbUpdateConcurrencyException) { return Results.Conflict(); }
```

**Gotchas / follow-ups**
- With `Attach` + stub, set the token on the stub itself.
- Rowversion is `byte[]` → in JSON becomes Base64 string; compare as bytes.
- Without the original token being set, there's a race window between your load and save only (small), not the user's whole editing session.

### 10.5 [Tricky] Edge cases: concurrent deletes, `ExecuteUpdate`, non-SQL-Server tokens, and "retry until success" loops.
**Answer**
A concurrently deleted row also gives 0 rows affected → `DbUpdateConcurrencyException` (and `GetDatabaseValues()` returns null). `ExecuteUpdate`/`ExecuteDelete` ignore concurrency tokens. On providers without rowversion, use a `[ConcurrencyCheck]` Guid/int you update yourself. Unbounded retry loops can livelock under contention.

**Explanation**
Retries should be bounded with backoff, and the whole read-modify-write must be redone (re-evaluate business rules against fresh data) rather than just re-saving stale decisions — especially for things like inventory counts, where a better approach may be an atomic `UPDATE ... SET Stock = Stock - @n WHERE Stock >= @n`.

**Code**
```csharp
var rows = await db.Products
    .Where(p => p.Id == id && p.Stock >= qty)
    .ExecuteUpdateAsync(s => s.SetProperty(p => p.Stock, p => p.Stock - qty), ct);
if (rows == 0) throw new InsufficientStockException();
```
```sql
UPDATE [p] SET [p].[Stock] = [p].[Stock] - @qty FROM [Products] AS [p] WHERE [p].[Id] = @id AND [p].[Stock] >= @qty
```

**Gotchas / follow-ups**
- Two concurrency tokens on one entity are OK; all appear in WHERE.
- Concurrency checks cover only rows EF writes; they don't prevent write skew across rows (use constraints/serializable).
- Owned/table-split entities share the owner's rowversion.

**Interview traps**
- Setting `entity.RowVersion = dto.RowVersion` instead of `OriginalValue`.
- Catching the exception and ignoring it.
- Expecting `ExecuteUpdate` to enforce the token.
- Assuming optimistic concurrency prevents all anomalies (e.g. write skew).

**Cheat sheet**
- `[Timestamp]`/`IsRowVersion` = rowversion token in WHERE.
- 0 rows affected → `DbUpdateConcurrencyException`.
- `GetDatabaseValues()` null = row deleted.
- Client wins: `OriginalValues.SetValues(dbValues)`; store wins: `Reload()`.
- Disconnected: set `.Property(..).OriginalValue`.

---

## Topic 11: Transactions, execution strategies, locking (optimistic vs pessimistic)

**Study notes**

- `SaveChanges` is **implicitly transactional**: all changes in one call commit or roll back together.
- **Explicit**: `await using var tx = await db.Database.BeginTransactionAsync(); ... await tx.CommitAsync();` — disposing without commit rolls back. Share across contexts with `UseTransaction(dbTransaction)`.
- **Savepoints** (EF Core 5+): `tx.CreateSavepoint("x")`, `tx.RollbackToSavepoint("x")`. EF automatically uses savepoints inside `SaveChanges` when you're in a user transaction (SQL Server), so a failing SaveChanges can be retried without aborting the transaction (`Database.AutoSavepointsEnabled`).
- **TransactionScope**: ambient transaction; needs `TransactionScopeAsyncFlowOption.Enabled` for async. Can escalate to a **distributed transaction (MSDTC)** when more than one connection/resource is enlisted; supported on .NET (Windows) since .NET 7 but avoid where possible.
- **Execution strategy / retry**: `EnableRetryOnFailure()` retries transient failures (including deadlock 1205, timeouts, connection drops). With it, user transactions must run **inside** `strategy.ExecuteAsync(...)` so the whole unit can be replayed, and the work must be idempotent/re-runnable.
- **Pessimistic locking**: SQL Server has no EF API; use raw SQL hints (`UPDLOCK, ROWLOCK, HOLDLOCK`) inside a transaction, `sp_getapplock`, or higher isolation levels. **Optimistic** = rowversion (Topic 10).

| Approach | Blocks others? | Good for | Cost |
|---|---|---|---|
| Rowversion (optimistic) | No | Low contention, long think-time | Retry on conflict |
| `UPDLOCK` in tx | Yes (writers/other UPDLOCK) | High contention, short critical section | Blocking/deadlocks |
| Serializable | Yes (range locks) | Phantom prevention | Heavy |
| Snapshot/RCSI | Readers don't block | Read-heavy | tempdb/version store |

### 11.1 [Easy] When do you need an explicit transaction in EF Core?
**Answer**
When multiple `SaveChanges` calls (or SaveChanges plus raw SQL/`ExecuteUpdate`) must succeed or fail together. A single `SaveChanges` is already atomic.

**Explanation**
Common mistake: wrapping one `SaveChanges` in a transaction for no reason, or wrapping several without commit. Keep transactions short and avoid user I/O/HTTP calls inside.

**Code**
```csharp
await using var tx = await db.Database.BeginTransactionAsync(ct);
try
{
    db.Orders.Add(order);
    await db.SaveChangesAsync(ct);                       // gets order.Id

    await db.Database.ExecuteSqlAsync($"UPDATE Stock SET Qty = Qty - {qty} WHERE Sku = {sku}", ct);
    await tx.CommitAsync(ct);
}
catch { await tx.RollbackAsync(ct); throw; }
```

### 11.2 [Follow-up] What are savepoints and how does EF use them?
**Answer**
A savepoint marks a point within a transaction you can roll back to without aborting the whole transaction. EF uses them automatically when `SaveChanges` runs inside an existing user transaction so a failed save doesn't invalidate the outer transaction; you can also create your own.

**Explanation**
Useful for "try this step, fall back if it fails". Savepoints are provider-supported (SQL Server `SAVE TRANSACTION`).

**Code**
```csharp
await using var tx = await db.Database.BeginTransactionAsync(ct);
await tx.CreateSavepointAsync("BeforeOptional", ct);
try
{
    db.Add(optionalEntity);
    await db.SaveChangesAsync(ct);
}
catch (DbUpdateException)
{
    await tx.RollbackToSavepointAsync("BeforeOptional", ct);
}
await tx.CommitAsync(ct);
```
```sql
SAVE TRANSACTION [BeforeOptional];
-- ... ROLLBACK TRANSACTION [BeforeOptional];
```

### 11.3 [Medium] How do execution strategies (retry on failure) interact with user transactions?
**Answer**
With `EnableRetryOnFailure`, EF throws `InvalidOperationException` if you start your own transaction outside the strategy. You must wrap the whole unit of work in `CreateExecutionStrategy().ExecuteAsync(...)` so the strategy can replay it from the start on a transient failure.

**Explanation**
The delegate must be safe to re-run: create contexts/entities inside it or reset the tracker, avoid non-idempotent side effects (emails, queue messages). For commit-ambiguity (connection lost during commit) EF can't know the outcome, so replay may duplicate work — use idempotency keys or verify via `ExecuteInTransactionAsync(operation, verifySucceeded)`.

**Code**
```csharp
services.AddDbContext<AppDb>(o => o.UseSqlServer(cs, s => s.EnableRetryOnFailure(
    maxRetryCount: 5, maxRetryDelay: TimeSpan.FromSeconds(10), errorNumbersToAdd: null)));

var strategy = db.Database.CreateExecutionStrategy();
await strategy.ExecuteAsync(async () =>
{
    await using var tx = await db.Database.BeginTransactionAsync(ct);
    db.Orders.Add(NewOrder());
    await db.SaveChangesAsync(ct);
    await tx.CommitAsync(ct);
});
```

**Gotchas / follow-ups**
- `TransactionScope` is not compatible with retrying strategies in the same way; the ambient transaction must also be inside the strategy delegate.
- If the delegate throws a non-transient exception, the original is surfaced; after retry exhaustion `RetryLimitExceededException`.

### 11.4 [Hard] How do you implement pessimistic locking with EF Core on SQL Server?
**Answer**
Start a transaction, then read the row with a lock hint through raw SQL (`WITH (UPDLOCK, ROWLOCK)`), modify it, `SaveChanges`, and commit. Other transactions requesting `UPDLOCK` on that row wait until you commit.

**Explanation**
`UPDLOCK` takes update locks held until the transaction ends, preventing two readers-then-writers from both proceeding (which would deadlock on conversion to X locks under plain shared locks). Add `HOLDLOCK` (serializable semantics) when locking a *range/absent row* (e.g. "insert if not exists"). Keep the critical section short, access resources in a consistent order. `sp_getapplock` gives application-level named locks. Alternatives: `SET TRANSACTION ISOLATION LEVEL`, optimistic + retry.

**Code**
```csharp
await using var tx = await db.Database.BeginTransactionAsync(IsolationLevel.ReadCommitted, ct);

var account = await db.Accounts
    .FromSql($"SELECT * FROM Accounts WITH (UPDLOCK, ROWLOCK) WHERE Id = {id}")
    .SingleAsync(ct);

account.Balance -= amount;
await db.SaveChangesAsync(ct);
await tx.CommitAsync(ct);

// Application lock
await db.Database.ExecuteSqlAsync(
    $"EXEC sp_getapplock @Resource = {key}, @LockMode = 'Exclusive', @LockOwner = 'Transaction', @LockTimeout = 5000", ct);
```
```sql
SELECT * FROM Accounts WITH (UPDLOCK, ROWLOCK) WHERE Id = @p0;
UPDATE [Accounts] SET [Balance] = @p1 WHERE [Id] = @p0;
```

**Gotchas / follow-ups**
- `FromSql` results that are tracked work fine, as the query is composable only when plain SELECT; here we finish with `SingleAsync`, which wraps it in a subquery (`SELECT TOP(2) ... FROM (SELECT ... WITH (UPDLOCK...))`) — hints inside a derived table still apply, but verify in logs.
- Lock held only within the transaction; no transaction → lock released immediately (auto-commit).
- Optimistic is usually preferred for web apps unless contention is high.

### 11.5 [Tricky] Distributed transactions, `TransactionScope` and async pitfalls — what should you know?
**Answer**
`TransactionScope` must be created with `TransactionScopeAsyncFlowOption.Enabled` or the ambient transaction doesn't flow across `await`. Opening two connections (e.g. two DbContexts with different connection strings or two servers) can escalate to MSDTC, which isn't available on Linux/containers. Prefer one connection/transaction via `UseTransaction` or design with outbox/sagas instead of distributed transactions.

**Explanation**
Outbox pattern: write the domain change and an outbox message in the same local transaction; a background publisher forwards messages (at-least-once, consumers idempotent). Two EF contexts to the same DB can share a transaction using `context2.Database.UseTransaction(context1.Database.CurrentTransaction.GetDbTransaction())` (same connection).

**Code**
```csharp
using var scope = new TransactionScope(TransactionScopeOption.Required,
    new TransactionOptions { IsolationLevel = IsolationLevel.ReadCommitted, Timeout = TimeSpan.FromSeconds(30) },
    TransactionScopeAsyncFlowOption.Enabled);

await db1.SaveChangesAsync(ct);
await db2.SaveChangesAsync(ct);   // second connection -> may escalate to MSDTC
scope.Complete();

// Share one local transaction instead
await using var tx = await db1.Database.BeginTransactionAsync(ct);
await db2.Database.UseTransactionAsync(tx.GetDbTransaction(), ct);
```

**Gotchas / follow-ups**
- Default TransactionScope isolation is **Serializable** unless you pass options — a classic perf bug.
- Default timeout 1 minute (machine max 10).
- Don't call external HTTP/queues inside the transaction scope.

**Interview traps**
- Wrapping a single SaveChanges in a transaction "for safety".
- Using `BeginTransaction` with `EnableRetryOnFailure` without `CreateExecutionStrategy`.
- `TransactionScope` without async flow option; default Serializable isolation.
- Assuming EF has a built-in pessimistic lock API.
- Believing retries are free: non-idempotent side effects get duplicated.

**Cheat sheet**
- One `SaveChanges` = one transaction.
- `BeginTransactionAsync` + `CommitAsync`; dispose = rollback.
- Savepoints: `CreateSavepoint`/`RollbackToSavepoint`.
- Retry strategy: wrap the whole unit in `CreateExecutionStrategy().ExecuteAsync`.
- Pessimistic = raw `WITH (UPDLOCK, ROWLOCK)` in a tx; optimistic = rowversion.
- Distributed tx → prefer outbox/sagas.

---

## Topic 12: Async, cancellation, deadlocks & timeouts

**Study notes**

- Use `*Async` EF methods (`ToListAsync`, `SaveChangesAsync`, `FirstOrDefaultAsync`...) for I/O-bound DB calls so threads return to the pool; async doesn't make a single query faster.
- Pass `CancellationToken` (ASP.NET Core: `HttpContext.RequestAborted`) down to EF calls. Cancelling aborts the command (`OperationCanceledException`/`TaskCanceledException`); an in-flight `SaveChanges` that is cancelled rolls back its transaction, but if the command already reached the server and committed, you can't be sure — design idempotently.
- **`DbContext` is not thread-safe**: never run two operations on the same instance concurrently (`Task.WhenAll` over one context throws `InvalidOperationException: A second operation was started...`). Use `IDbContextFactory<T>` for parallel work.
- Don't block: `.Result`/`.Wait()`/`GetAwaiter().GetResult()` → thread-pool starvation/deadlocks. `ConfigureAwait(false)` is unnecessary in ASP.NET Core but fine in libraries.
- Timeouts: **command timeout** (default 30 s; `Database.SetCommandTimeout` or `CommandTimeout(...)` in provider options) vs connection timeout vs HTTP timeout. SQL error **-2** = timeout; **1205** = deadlock victim; **1222** = lock request timeout.
- **Deadlock**: two transactions hold locks the other needs. SQL Server picks a victim (error 1205). Fix: consistent access order, short transactions, proper indexes (fewer locked rows), RCSI/snapshot for readers, `UPDLOCK` for read-then-write, retry the victim.

### 12.1 [Easy] Why use async EF Core methods?
**Answer**
They free the thread while waiting for the database, improving scalability of web servers under concurrent load. They don't reduce the DB's work or latency for a single request.

**Explanation**
Use them consistently end-to-end (controller → service → EF); mixing sync-over-async blocks threads.

**Code**
```csharp
[HttpGet("{id}")]
public async Task<ActionResult<OrderDto>> Get(int id, CancellationToken ct)
{
    var dto = await db.Orders.Where(o => o.Id == id)
        .Select(o => new OrderDto(o.Id, o.Total)).SingleOrDefaultAsync(ct);
    return dto is null ? NotFound() : dto;
}
```

### 12.2 [Follow-up] How should you use CancellationToken with EF Core and what happens when cancelled?
**Answer**
Accept the token in your methods and pass it to every async EF call. On cancellation EF cancels the command and throws `OperationCanceledException`; for SaveChanges the transaction is rolled back if not yet committed.

**Explanation**
Handle `OperationCanceledException` specially (don't log as error, don't map to 500). In ASP.NET Core a client disconnect triggers `RequestAborted`. For writes that must complete regardless, use `CancellationToken.None`.

**Code**
```csharp
try { await db.SaveChangesAsync(ct); }
catch (OperationCanceledException) when (ct.IsCancellationRequested) { /* client left */ throw; }
```

### 12.3 [Medium] Why does running two queries in parallel on the same DbContext fail, and how do you parallelize correctly?
**Answer**
`DbContext` holds mutable state (change tracker, one connection) and isn't thread-safe; concurrent use throws an `InvalidOperationException`. Create one context per parallel task via `IDbContextFactory<T>` (or sequence them).

**Explanation**
Parallel DB queries rarely speed things up much and add connection-pool pressure; ensure the pool size (default max 100) supports it. Also avoid awaiting inside an unawaited fire-and-forget on a scoped context — the scope may be disposed.

**Code**
```csharp
async Task<List<T>> Run<T>(Func<AppDb, Task<List<T>>> q)
{
    await using var ctx = await factory.CreateDbContextAsync(ct);
    return await q(ctx);
}

var (orders, customers) = (await Run(c => c.Orders.ToListAsync(ct)), await Run(c => c.Customers.ToListAsync(ct)));
// Parallel:
var t1 = Run(c => c.Orders.AsNoTracking().ToListAsync(ct));
var t2 = Run(c => c.Customers.AsNoTracking().ToListAsync(ct));
await Task.WhenAll(t1, t2);
```

### 12.4 [Hard] You see deadlock errors (1205) from EF in production. How do you diagnose and handle them?
**Answer**
Capture the deadlock graph (system_health Extended Events / `xml_deadlock_report`), identify the two statements and resources, then fix the root cause: consistent lock order, shorter transactions, proper indexes, RCSI, `UPDLOCK` for read-modify-write. Also add retry for 1205 (the retrying execution strategy treats it as transient).

**Explanation**
Typical EF-induced deadlock: two transactions each `SELECT` then `UPDATE` the same rows (S lock → U conversion) → fix with `UPDLOCK`/atomic `UPDATE`. Another: missing index causing scans that lock many rows, or updates touching parent/child tables in different orders. `SaveChanges` orders commands by dependency graph (stable), but separate SaveChanges in different orders can deadlock. A retry only treats the symptom; make the transaction replayable.

**Code**
```csharp
services.AddDbContext<AppDb>(o => o.UseSqlServer(cs, s => s.EnableRetryOnFailure(
    maxRetryCount: 5, maxRetryDelay: TimeSpan.FromSeconds(5), errorNumbersToAdd: new[] { 1205 })));

catch (DbUpdateException ex) when (ex.InnerException is SqlException { Number: 1205 })
{
    // deadlock victim (if not using a retrying strategy)
}
```
```sql
-- Enable RCSI so readers don't block writers (DB-level change; test first)
ALTER DATABASE [MyDb] SET READ_COMMITTED_SNAPSHOT ON WITH ROLLBACK IMMEDIATE;
```

**Gotchas / follow-ups**
- 1205 is already in the SQL Server provider's transient list; adding it is harmless/explicit.
- RCSI changes read semantics (no blocking on read, uses tempdb version store) — measure.

### 12.5 [Tricky] A query times out intermittently; raising CommandTimeout "fixes" it. What's wrong with that, and what do you do instead?
**Answer**
A larger timeout hides the problem and holds connections/locks longer, risking pool exhaustion and blocking. Identify the cause: blocking (another transaction), bad plan/parameter sniffing, missing index, or large result set; then fix it, and set timeouts deliberately per operation (e.g. long for reports, short for APIs).

**Explanation**
Check `sys.dm_exec_requests`/`blocking_session_id`, Query Store regressions, actual execution plans. Timeouts surface as `SqlException` Number -2 (wrapped), commonly with `Execution Timeout Expired`. Pool exhaustion shows as "Timeout expired. The timeout period elapsed prior to obtaining a connection from the pool" — a *different* timeout (leaked contexts/unawaited calls/long transactions).

**Code**
```csharp
db.Database.SetCommandTimeout(TimeSpan.FromMinutes(5));  // only for the report context

try { await db.SaveChangesAsync(ct); }
catch (Exception ex) when (ex.InnerException is SqlException { Number: -2 }) { /* command timeout */ }
```
```sql
SELECT r.session_id, r.blocking_session_id, r.wait_type, r.wait_time, t.text
FROM sys.dm_exec_requests r CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
WHERE r.blocking_session_id <> 0;
```

**Gotchas / follow-ups**
- Cancellation (`ct`) with a client-side timeout vs. server command timeout: both can fire; distinguish by token state.
- Pooled `DbContext` with changed `CommandTimeout` — reset per use.

**Interview traps**
- "Async makes queries faster."
- `Task.WhenAll` on one DbContext.
- `.Result` on EF calls in controllers.
- Just increasing timeouts or just retrying deadlocks.
- Confusing connection-pool timeout with command timeout.

**Cheat sheet**
- Async = scalability, not speed; always pass `CancellationToken`.
- `DbContext` isn't thread-safe → `IDbContextFactory` for parallel.
- Deadlock 1205 → fix order/indexes/RCSI; retry as safety net.
- Timeout -2 → find blocking/plan issues before raising limits.
- Pool exhaustion ≠ command timeout.

---
## Topic 13: Interceptors, auditing, soft delete, multi-tenancy

**Study notes**

- **Interceptors** hook into EF operations; register with `optionsBuilder.AddInterceptors(...)`. Main kinds: `SaveChangesInterceptor` (`SavingChanges[Async]`, `SavedChanges`, `SaveChangesFailed`), `DbCommandInterceptor` (`ReaderExecuting[Async]`, `NonQueryExecuting`, `CommandCreated`...), `DbConnectionInterceptor` (`ConnectionOpening/Opened`), `DbTransactionInterceptor`, `IMaterializationInterceptor`, `IQueryExpressionInterceptor`.
- Interceptors can inspect/modify the command or **suppress** execution by returning a result (`InterceptionResult.SuppressWithResult`). Interceptors are typically singletons; resolve scoped services (e.g. current user) from `eventData.Context.GetService<>()` or from the `DbContext` itself, or register with the `(sp, options)` overload and keep them stateless.
- **Auditing**: in `SavingChanges`, iterate `ChangeTracker.Entries()` and stamp `CreatedAt/By`, `ModifiedAt/By`, or write audit rows. Doesn't cover `ExecuteUpdate/Delete` or raw SQL. Alternative: SQL Server temporal tables (`IsTemporal()`), CDC, triggers.
- **Soft delete**: `IsDeleted` flag + **global query filter** `HasQueryFilter(e => !e.IsDeleted)` + interceptor converting `Deleted` into `Modified`. Use `IgnoreQueryFilters()` to see deleted rows.
- **Multi-tenancy**: (a) shared DB + `TenantId` column + global query filter (+ set TenantId on save), (b) schema per tenant, (c) database per tenant (connection string chosen at runtime). Query filters referencing a DbContext property are re-evaluated per context instance.

```
Request -> middleware resolves TenantId -> scoped ITenantProvider
        -> DbContext(ctor gets ITenantProvider)
              HasQueryFilter(e => e.TenantId == _tenantId)   // reads

        -> SaveChangesInterceptor sets TenantId/audit fields // writes
```

### 13.1 [Easy] What is an EF Core interceptor and give a use case?
**Answer**
An interceptor is a hook that observes or modifies EF operations such as `SaveChanges`, command execution or connection opening. Typical use: automatically stamping audit fields or logging slow SQL.

**Explanation**
They're registered per context options and run for every context built with them. They're more targeted than overriding `SaveChanges` (they also work with `DbContext` pooling and can be shared).

**Code**
```csharp
public sealed class AuditInterceptor : SaveChangesInterceptor
{
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData, InterceptionResult<int> result, CancellationToken ct = default)
    {
        var ctx = eventData.Context!;
        var now = DateTime.UtcNow;
        foreach (var e in ctx.ChangeTracker.Entries<IAuditable>())
        {
            if (e.State == EntityState.Added)    e.Entity.CreatedAt = now;
            if (e.State is EntityState.Added or EntityState.Modified) e.Entity.UpdatedAt = now;
        }
        return base.SavingChangesAsync(eventData, result, ct);
    }
}

services.AddSingleton<AuditInterceptor>();
services.AddDbContext<AppDb>((sp, o) => o.UseSqlServer(cs).AddInterceptors(sp.GetRequiredService<AuditInterceptor>()));
```

### 13.2 [Follow-up] How do you implement soft delete?
**Answer**
Add an `IsDeleted` column, configure a global query filter to exclude deleted rows, and intercept `SaveChanges` to turn `Deleted` entries into `Modified` with `IsDeleted = true`. Use `IgnoreQueryFilters()` to include deleted rows.

**Explanation**
Add an index on the flag (filtered index `WHERE IsDeleted = 0` for hot paths). Beware: unique indexes must account for soft-deleted rows, cascade deletes don't apply to soft deletes (you must handle children), and `ExecuteDelete` hard-deletes.

**Code**
```csharp
modelBuilder.Entity<Product>().HasQueryFilter(p => !p.IsDeleted);
modelBuilder.Entity<Product>().HasIndex(p => p.Sku).IsUnique().HasFilter("[IsDeleted] = 0");

public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
    DbContextEventData e, InterceptionResult<int> r, CancellationToken ct = default)
{
    foreach (var entry in e.Context!.ChangeTracker.Entries<ISoftDelete>()
                 .Where(x => x.State == EntityState.Deleted))
    {
        entry.State = EntityState.Modified;
        entry.Entity.IsDeleted = true;
    }
    return base.SavingChangesAsync(e, r, ct);
}

var all = await db.Products.IgnoreQueryFilters().ToListAsync();
```
```sql
SELECT ... FROM [Products] AS [p] WHERE [p].[IsDeleted] = CAST(0 AS bit)
```

### 13.3 [Medium] How do you implement multi-tenancy with a shared database?
**Answer**
Add a `TenantId` to tenant-owned entities, configure a global query filter comparing it to the current tenant from a scoped provider injected into the DbContext, and set `TenantId` on insert (interceptor/`SaveChanges`). The filter is evaluated per context instance, so the same compiled model serves all tenants.

**Explanation**
The filter must reference a **field/property of the DbContext** (not a captured static) to be parameterized per instance. DB-per-tenant: choose the connection string at context creation (`IDbContextFactory` + tenant resolver); migrations must run per tenant. Add `TenantId` as the leading column in indexes. For defense in depth use SQL Server **Row-Level Security** with `SESSION_CONTEXT` set from a `DbConnectionInterceptor`.

**Code**
```csharp
public class AppDb : DbContext
{
    private readonly ITenantProvider _tenant;
    public AppDb(DbContextOptions<AppDb> o, ITenantProvider tenant) : base(o) => _tenant = tenant;
    private Guid TenantId => _tenant.TenantId;

    protected override void OnModelCreating(ModelBuilder mb)
        => mb.Entity<Invoice>().HasQueryFilter(i => i.TenantId == TenantId);
}
```
```sql
SELECT ... FROM [Invoices] AS [i] WHERE [i].[TenantId] = @__ef_filter__TenantId_0
```

### 13.4 [Hard] Use a connection interceptor for SQL Server Row-Level Security and discuss what interceptors cannot see.
**Answer**
In `ConnectionOpened` run `EXEC sp_set_session_context 'TenantId', @id`, and let an RLS predicate filter rows in the DB. Interceptors don't cover raw operations outside EF, and `SaveChanges` interceptors don't fire for `ExecuteUpdate/ExecuteDelete`, so rely on DB-level enforcement for strong isolation.

**Explanation**
With connection pooling, session context persists across reuse unless the pool resets it — SqlClient calls `sp_reset_connection`, which clears `SESSION_CONTEXT`, so set it on every open. Pooled `DbContext` + scoped tenant requires care: don't capture tenant in constructor state that isn't reset.

**Code**
```csharp
public sealed class TenantConnectionInterceptor(ITenantAccessor tenants) : DbConnectionInterceptor
{
    public override async Task ConnectionOpenedAsync(DbConnection connection,
        ConnectionEndEventData eventData, CancellationToken ct = default)
    {
        await using var cmd = connection.CreateCommand();
        cmd.CommandText = "EXEC sp_set_session_context @key = N'TenantId', @value = @v, @read_only = 1";
        var p = cmd.CreateParameter(); p.ParameterName = "@v"; p.Value = tenants.CurrentTenantId; cmd.Parameters.Add(p);
        await cmd.ExecuteNonQueryAsync(ct);
    }
}
```
```sql
CREATE FUNCTION Security.fn_tenant(@TenantId uniqueidentifier) RETURNS TABLE WITH SCHEMABINDING AS
RETURN SELECT 1 AS ok WHERE @TenantId = CAST(SESSION_CONTEXT(N'TenantId') AS uniqueidentifier);
CREATE SECURITY POLICY Security.TenantPolicy ADD FILTER PREDICATE Security.fn_tenant(TenantId) ON dbo.Invoices;
```

**Gotchas / follow-ups**
- `@read_only = 1` means the key can't be changed again on that connection until reset.
- A singleton interceptor needs a way to get the current tenant (e.g. `IHttpContextAccessor` / AsyncLocal accessor), not a scoped service injected directly.

### 13.5 [Tricky] Where do global query filters surprise people?
**Answer**
They apply to navigations and `Include`, and compose on top of `FromSql`. A required navigation to a filtered-out principal makes the dependent row disappear (EF warns about this). They also apply to the query part of `ExecuteUpdate/Delete`, need `IgnoreQueryFilters()` for admin paths, and EF 8 allows only one filter per entity (combine with `&&`).

**Explanation**
Required navigation + filtered principal: `Order` with required `Customer` and a soft-deleted customer → the order is silently excluded. Fix by making the filters consistent on both sides or the navigation optional. Named filters (several per entity) arrived in EF 10, so in EF 8 combine conditions manually.

**Code**
```csharp
modelBuilder.Entity<Order>().HasQueryFilter(o => !o.IsDeleted && o.TenantId == TenantId);
modelBuilder.Entity<Customer>().HasQueryFilter(c => !c.IsDeleted && c.TenantId == TenantId);

// ExecuteDelete respects filters in its query
await db.Products.Where(p => p.Discontinued).ExecuteDeleteAsync(ct);   // only non-deleted, current tenant
```

**Gotchas / follow-ups**
- Filtered unique index needed so a soft-deleted SKU doesn't block a new one.
- Query filters aren't a security boundary on their own; combine with RLS.
- Auditing via interceptor misses bulk operations; use temporal tables/CDC for complete history.

**Interview traps**
- Believing a `SaveChanges` interceptor catches `ExecuteUpdate/Delete`.
- Capturing tenant id in a static/singleton → cross-tenant data leaks.
- Soft delete + unique index collisions.
- Injecting scoped services into singleton interceptors.
- Thinking query filters = security.

**Cheat sheet**
- Interceptors: `AddInterceptors`, kinds: SaveChanges, Command, Connection, Transaction.
- Soft delete = `HasQueryFilter` + intercept `Deleted → Modified`.
- `IgnoreQueryFilters()` bypasses filters.
- Multi-tenant: filter on DbContext field + set TenantId on save; RLS for defense in depth.
- Bulk ops bypass SaveChanges interceptors.

---

## Topic 14: EF Core 8/9 features (JSON, complex types, HierarchyId, primitive collections)

**Study notes**

- **JSON columns** (EF 7+; richer in 8): map owned types to a JSON column with `OwnsOne(...).ToJson()` / `OwnsMany(...).ToJson()`; query into JSON with LINQ (translates to `JSON_VALUE`/`OPENJSON`). EF 8 adds arrays/primitive collections within JSON; `ExecuteUpdate` on JSON-mapped properties is not supported in EF 8 (added later). Stored as `nvarchar(max)` (or `json` type on SQL Server 2025/Azure with EF 10).
- **Complex types** (EF 8): `ComplexProperty` / `[ComplexType]` — value objects with **no identity**, stored in the owner's table columns, copied by value (not shared references), can be structs/records; no change-tracking identity, no relationships; collections of complex types came later (EF 10).
- **Primitive collections** (EF 8): `List<int>`, `int[]`, `List<string>` properties map to a JSON `nvarchar(max)` column; queries translate with `OPENJSON` (`Contains`, `Count`, indexing, etc.).
- **HierarchyId** (EF 8 via `Microsoft.EntityFrameworkCore.SqlServer.HierarchyId`): `UseHierarchyId()`; supports `IsDescendantOf`, `GetLevel`, `GetParent`, etc.
- **Other EF 8/9 notes**: raw SQL for unmapped types (`SqlQuery<T>`), sentinel-value changes for DB defaults, `DateOnly/TimeOnly`, lazy loading w/o tracking, `ExecuteUpdate` more flexible, `Contains` via OPENJSON. EF 9: `UseSeeding/UseAsyncSeeding`, migrations take an application lock (prevents concurrent migrate), `Migrate` warns/throws when model has pending changes, improved `GroupBy` and subquery translation, `HasData` improvements, `AOT/precompiled` work.

| Feature | Storage | Key trait |
|---|---|---|
| Owned entity (table split) | Owner's columns | Has shadow key; can be collection (separate table) |
| Complex type | Owner's columns | Value semantics, no identity, EF 8 |
| JSON column | nvarchar(max) JSON | Query inside JSON; flexible schema |
| Primitive collection | nvarchar(max) JSON | `List<int>` etc. |

### 14.1 [Easy] What's a complex type and how does it differ from an owned entity?
**Answer**
A complex type (EF 8) is a value object stored in the owner's table with no identity and value-equality semantics. An owned entity is still an entity with an implicit key and relationship tracking, so it's heavier and has reference semantics.

**Explanation**
Complex types can be assigned/copied freely (copy semantics), may be structs or records, and can't be shared between two properties by reference. They're ideal for `Address`, `Money`, `DateRange`.

**Code**
```csharp
public record Address(string Street, string City, string Zip);
public class Customer { public int Id { get; set; } public Address Billing { get; set; } = null!; }

modelBuilder.Entity<Customer>().ComplexProperty(c => c.Billing);
// or: [ComplexType] on the Address type
```
```sql
CREATE TABLE [Customers] ([Id] int, [Billing_Street] nvarchar(max), [Billing_City] nvarchar(max), [Billing_Zip] nvarchar(max));
```

### 14.2 [Follow-up] How do JSON columns work in EF Core?
**Answer**
Map an owned type (or collection) with `ToJson()`; EF stores it as JSON in one column and translates LINQ over its properties into `JSON_VALUE`/`OPENJSON` SQL. It's good for semi-structured, rarely joined data.

**Explanation**
Trade-offs: no FK constraints inside JSON, indexing requires computed columns on `JSON_VALUE(...)`, updates rewrite the JSON (EF 8 updates changed paths via `JSON_MODIFY`). Don't use it to avoid proper modeling of relational data.

**Code**
```csharp
public class Order { public int Id { get; set; } public ShippingInfo Shipping { get; set; } = null!; }
public class ShippingInfo { public string Carrier { get; set; } = ""; public List<string> Tags { get; set; } = new(); }

modelBuilder.Entity<Order>().OwnsOne(o => o.Shipping, b => b.ToJson());

var dhl = await db.Orders.Where(o => o.Shipping.Carrier == "DHL").ToListAsync();
```
```sql
SELECT [o].[Id], [o].[Shipping] FROM [Orders] AS [o] WHERE JSON_VALUE([o].[Shipping], '$.Carrier') = N'DHL'
```

### 14.3 [Medium] What are primitive collections and how do queries on them translate?
**Answer**
EF 8 maps `List<int>`, `string[]`, etc. as properties stored in a JSON column; `Contains`, `Count`, `Any` and indexers translate via `OPENJSON`/`JSON_VALUE`. Also, parameter collections in `Contains` are sent as JSON.

**Explanation**
You cannot index individual elements efficiently (OPENJSON scan per row) — for large or frequently filtered sets use a real child table. Convenient for tags/feature flags.

**Code**
```csharp
public class Post { public int Id { get; set; } public List<string> Tags { get; set; } = new(); }

var q = await db.Posts.Where(p => p.Tags.Contains("efcore")).ToListAsync();
```
```sql
SELECT [p].[Id], [p].[Tags] FROM [Posts] AS [p]
WHERE N'efcore' IN (SELECT [t].[value] FROM OPENJSON([p].[Tags]) WITH ([value] nvarchar(max) '$') AS [t])
```

### 14.4 [Hard] How do you use HierarchyId and when is it appropriate?
**Answer**
Install the HierarchyId package, enable `UseHierarchyId()` in the SQL Server options and map a `HierarchyId` property. Use it for tree structures where you query descendants/ancestors efficiently (org charts, category trees) with methods like `IsDescendantOf`.

**Explanation**
It stores a path (e.g. `/1/3/`) in a compact binary; queries on subtrees use an index on the hierarchyid (depth-first) or breadth-first indexes. Alternatives: adjacency list + recursive CTE (via raw SQL), materialized path, nested sets, closure table. Moving subtrees needs `GetReparentedValue`. Tradeoff: SQL Server specific, app-level maintenance of paths.

**Code**
```csharp
// <PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer.HierarchyId" />
services.AddDbContext<AppDb>(o => o.UseSqlServer(cs, x => x.UseHierarchyId()));

public class Category { public int Id { get; set; } public HierarchyId Path { get; set; } = null!; public string Name { get; set; } = ""; }

var root = await db.Categories.SingleAsync(c => c.Name == "Electronics");
var subtree = await db.Categories.Where(c => c.Path.IsDescendantOf(root.Path)).ToListAsync();
```
```sql
SELECT [c].[Id], [c].[Path], [c].[Name] FROM [Categories] AS [c] WHERE [c].[Path].IsDescendantOf(@__root_Path_0) = CAST(1 AS bit)
```

### 14.5 [Tricky] What are the gotchas when choosing between JSON columns, owned types, complex types and separate tables?
**Answer**
Choose by identity and query needs: complex types for value objects always loaded with the owner; owned entities for dependent entities needing identity/collections; JSON for schemaless/rare-lookup nested data; tables for anything with its own lifecycle, FKs, or heavy queries.

**Explanation**
Gotchas: complex types in EF 8 can't be used in collections (that arrived in EF 10), optional (nullable) complex properties had limitations in EF 8 — prefer non-null complex properties and verify against the docs for your version — and changing a JSON model requires migrating JSON content, not DDL. Owned collections default to a separate table with a composite key.

**Code**
```csharp
modelBuilder.Entity<Customer>(b =>
{
    b.ComplexProperty(c => c.Billing, a => a.Property(x => x.Zip).HasMaxLength(10));
    b.OwnsMany(c => c.Phones, p => p.ToJson());        // JSON collection
});
```

**Gotchas / follow-ups**
- JSON mapping in EF 8 requires the owned type to be configured with `ToJson()`; changes to JSON columns update the whole or partial JSON depending on provider support.
- Primitive collections: replacing the collection is the safest way to get a change detected; verify in-place mutation behavior on your EF version.
- Always check the version docs: EF 9 and 10 changed several of these limits.

**Interview traps**
- Treating complex types as owned entities.
- Putting relational data in JSON to avoid migrations.
- Querying large primitive collections and expecting index use.
- Using `HierarchyId` without the extra package.

**Cheat sheet**
- Complex type = value object, no identity (EF 8).
- JSON column = `OwnsOne/OwnsMany(...).ToJson()`; translates to `JSON_VALUE/OPENJSON`.
- Primitive collections (`List<int>`) → JSON + OPENJSON.
- HierarchyId = extra NuGet + `UseHierarchyId()`.
- EF 9: seeding hooks, migration lock, pending-model-changes check.

---
## Topic 15: EF vs Dapper, Repository/UoW, Specification, CQRS, DDD aggregates, testing

**Study notes**

- **EF Core**: full ORM — change tracking, LINQ, migrations, relationships, concurrency. **Dapper**: micro-ORM — maps SQL result sets to objects, you write SQL, no tracking/migrations. Dapper is faster on raw materialization and gives full SQL control; EF wins productivity/safety for writes. Mixing is common: EF for commands (writes), Dapper (or EF `SqlQuery`) for heavy reads/reports, sharing the same connection/transaction.
- **`DbContext` already is a Unit of Work** (change tracker + `SaveChanges`), and **`DbSet<T>` is a repository**. A generic repository on top adds little and often hurts (leaky `IQueryable`, lost `Include` control). Valid reasons for abstraction: domain-specific query methods, aggregate-root persistence boundaries, test seams — not "so we can swap ORMs".
- **Specification pattern**: encapsulate query criteria (`Expression<Func<T,bool>>`, includes, order, paging) in reusable objects applied to an `IQueryable` (e.g. Ardalis.Specification).
- **CQRS**: separate write model (rich aggregates, tracked, transactional) from read model (DTO projections, `AsNoTracking`, Dapper/views, maybe a replica). Doesn't require separate databases or event sourcing.
- **DDD aggregates & EF**: map aggregate roots as entities; children as owned types/entities reachable only via the root; private setters/backing fields (`UsePropertyAccessMode`, `HasField`), value objects as complex types/owned/value converters; one `SaveChanges` per aggregate/transaction; references between aggregates by ID, not navigation.
- **Testing data access**: EF InMemory provider is **not relational** (no constraints, transactions, SQL translation) — discouraged for logic that depends on SQL. Prefer **SQLite in-memory** (relational but different dialect) or, best, **real SQL Server via Testcontainers/LocalDB** with Respawn for cleanup; unit-test domain logic without EF.

```
Commands -> Application service -> Aggregate (domain rules) -> DbContext.SaveChanges (UoW)
Queries  -> Query handler -> AsNoTracking projection / Dapper / view -> DTO (no domain model)
```

### 15.1 [Easy] When would you choose Dapper over EF Core, or vice versa?
**Answer**
Choose EF Core for CRUD, relationships, change tracking, migrations and developer productivity; choose Dapper for hand-tuned SQL, complex reporting queries, bulk reads or hot paths where you want minimal overhead. Many systems use both.

**Explanation**
Modern EF Core with `AsNoTracking` and compiled queries closes most of the gap; the real differentiator is control of SQL (window functions, CTEs, hints, table-valued parameters) and simplicity. Dapper has no change tracking — you write INSERT/UPDATE yourself.

**Code**
```csharp
var rows = await connection.QueryAsync<OrderReportRow>(
    new CommandDefinition(
        "SELECT CustomerId, SUM(Total) AS Total FROM Orders WHERE CreatedAt >= @from GROUP BY CustomerId",
        new { from }, cancellationToken: ct));
```

### 15.2 [Follow-up] How do you mix EF Core and Dapper in one transaction?
**Answer**
Start the transaction on the EF context, then give Dapper the same `DbConnection` and `DbTransaction`. Both operations then commit or roll back together.

**Explanation**
Make sure to pass the `transaction` argument to Dapper — otherwise SQL Server raises "requires the command to have a transaction". Also remember Dapper writes bypass EF tracking, so tracked entities can be stale.

**Code**
```csharp
await using var tx = await db.Database.BeginTransactionAsync(ct);
var conn = db.Database.GetDbConnection();
var dbTx = tx.GetDbTransaction();

await conn.ExecuteAsync(
    new CommandDefinition("UPDATE Stock SET Qty = Qty - @q WHERE Sku = @sku", new { q, sku },
        transaction: dbTx, cancellationToken: ct));

db.Orders.Add(order);
await db.SaveChangesAsync(ct);
await tx.CommitAsync(ct);
```

### 15.3 [Medium] Do you need a Repository and Unit of Work over EF Core?
**Answer**
Usually not: `DbContext` is a unit of work and `DbSet` is a repository. A thin generic wrapper adds indirection without value; use repositories only for aggregate-specific persistence logic or when you need a clean seam, and use query objects/specifications for reads.

**Explanation**
Arguments against: leaks `IQueryable`, hides `Include`/tracking choices, duplicates APIs, makes it hard to share one DbContext/transaction across repositories without a UoW wrapper that merely forwards `SaveChanges`. Arguments for: enforcing aggregate boundaries, simpler unit tests of services (mock), centralizing complex queries. Test with a real DB instead of mocking `DbSet`.

**Code**
```csharp
// Aggregate-focused repository (not generic CRUD)
public interface IOrderRepository
{
    Task<Order?> GetAsync(OrderId id, CancellationToken ct);   // loads aggregate with children
    void Add(Order order);
}
public class OrderRepository(AppDb db) : IOrderRepository
{
    public Task<Order?> GetAsync(OrderId id, CancellationToken ct)
        => db.Orders.Include(o => o.Lines).SingleOrDefaultAsync(o => o.Id == id, ct);
    public void Add(Order order) => db.Orders.Add(order);
}
// UoW = DbContext.SaveChangesAsync (exposed via IUnitOfWork if needed)
```

### 15.4 [Hard] Show a Specification pattern and how it avoids the generic-repository problems.
**Answer**
A specification object holds criteria, includes, ordering and paging; a small evaluator applies it to `IQueryable<T>`. It keeps reusable query logic testable, composable, and translated to SQL (since criteria are expression trees).

**Explanation**
Keeps the repository to `ListAsync(spec)`/`FirstOrDefaultAsync(spec)`. Specs can also be tested against a real DB. Don't over-engineer: for simple apps, extension methods on `IQueryable<T>` achieve the same.

**Code**
```csharp
public abstract class Specification<T>
{
    public Expression<Func<T, bool>>? Criteria { get; protected set; }
    public List<Expression<Func<T, object>>> Includes { get; } = new();
    public Expression<Func<T, object>>? OrderBy { get; protected set; }
}

public class ActiveOrdersForCustomer : Specification<Order>
{
    public ActiveOrdersForCustomer(int customerId)
    {
        Criteria = o => o.CustomerId == customerId && o.Status == OrderStatus.Active;
        Includes.Add(o => o.Lines);
        OrderBy = o => o.CreatedAt;
    }
}

public static class SpecEvaluator
{
    public static IQueryable<T> Apply<T>(IQueryable<T> q, Specification<T> s) where T : class
    {
        if (s.Criteria is not null) q = q.Where(s.Criteria);
        foreach (var inc in s.Includes) q = q.Include(inc);
        if (s.OrderBy is not null) q = q.OrderBy(s.OrderBy);
        return q;
    }
}

// Cheaper alternative: extension methods
public static IQueryable<Order> Active(this IQueryable<Order> q) => q.Where(o => o.Status == OrderStatus.Active);
```

**Gotchas / follow-ups**
- `Expression<Func<T, object>>` for ordering boxes value types — EF handles the convert for common types, but check SQL.
- Specifications that return entities still track; add `AsNoTracking` for reads.

### 15.5 [Hard] How do you map a DDD aggregate (private setters, value objects, child entities) with EF Core?
**Answer**
Make the root expose behavior, keep collections private via backing fields, map value objects as complex/owned types or with value converters, and reference other aggregates by ID. Load and save one aggregate per transaction through the root.

**Explanation**
EF can set private setters via reflection; for collections use `HasField` with `PropertyAccessMode.Field`; use a protected/private parameterless constructor for materialization. For strongly-typed IDs use `HasConversion`. Concurrency token on the root protects the whole aggregate. Domain events can be collected on the root and dispatched in a `SaveChangesInterceptor` or after `SaveChanges` (outbox for reliability).

**Code**
```csharp
public class Order
{
    private readonly List<OrderLine> _lines = new();
    private Order() { }                                   // EF
    public Order(OrderId id, CustomerId customerId) { Id = id; CustomerId = customerId; }
    public OrderId Id { get; private set; }
    public CustomerId CustomerId { get; private set; }
    public Money Total { get; private set; }
    public IReadOnlyCollection<OrderLine> Lines => _lines.AsReadOnly();
    public byte[] RowVersion { get; private set; } = null!;
    public void AddLine(Sku sku, int qty, Money price) { _lines.Add(new OrderLine(sku, qty, price)); Total += price * qty; }
}

modelBuilder.Entity<Order>(b =>
{
    b.HasKey(o => o.Id);
    b.Property(o => o.Id).HasConversion(id => id.Value, v => new OrderId(v));
    b.Property(o => o.CustomerId).HasConversion(id => id.Value, v => new CustomerId(v));
    b.ComplexProperty(o => o.Total);                       // Money as complex type (EF 8)
    b.HasMany(o => o.Lines).WithOne().HasForeignKey("OrderId");
    b.Navigation(o => o.Lines).UsePropertyAccessMode(PropertyAccessMode.Field);
    b.Property(o => o.RowVersion).IsRowVersion();
});
```

**Gotchas / follow-ups**
- Don't navigate between aggregates; it causes huge graphs and transactional coupling.
- Strongly-typed ID converters need `ValueConverter` mapping in query translation — usually OK with `HasConversion`, but custom comparers/`ToString` in queries may not translate.

### 15.6 [Tricky] How do you test EF data access reliably, and why is the InMemory provider discouraged?
**Answer**
Test against the real database engine (Testcontainers SQL Server or LocalDB), reset data between tests (Respawn or transaction rollback), and unit-test domain logic without EF. InMemory ignores relational behavior (FK constraints, unique indexes, transactions, raw SQL, SQL translation differences, collation), so tests pass that would fail in production.

**Explanation**
SQLite in-memory is relational but has a different dialect/type system (e.g. no `rowversion`, different string comparison, limited `DateTimeOffset` ordering). Use it only for fast smoke tests. Mocking `DbSet<T>` doesn't reproduce LINQ-to-SQL translation. Use `WebApplicationFactory` + test DB for integration tests; seed via migrations.

**Code**
```csharp
public class SqlServerFixture : IAsyncLifetime
{
    private readonly MsSqlContainer _sql = new MsSqlBuilder().Build();   // Testcontainers.MsSql
    public string ConnectionString => _sql.GetConnectionString();
    public async Task InitializeAsync()
    {
        await _sql.StartAsync();
        await using var db = CreateDb();
        await db.Database.MigrateAsync();
    }
    public AppDb CreateDb() => new(new DbContextOptionsBuilder<AppDb>().UseSqlServer(ConnectionString).Options);
    public Task DisposeAsync() => _sql.DisposeAsync().AsTask();
}

[Fact]
public async Task Duplicate_sku_violates_unique_index()
{
    await using var db = fx.CreateDb();
    db.Products.AddRange(new Product { Sku = "A" }, new Product { Sku = "A" });
    await Assert.ThrowsAsync<DbUpdateException>(() => db.SaveChangesAsync());  // InMemory would not throw
}
```

**Gotchas / follow-ups**
- Don't share one DbContext between arrange and assert steps — tracked data hides query problems; use a fresh context to verify.
- Containers need Docker in CI.
- Run migrations in tests to also validate them.

**Interview traps**
- "Repository pattern is mandatory to be able to unit test."
- Mocking `DbSet<T>` or `IQueryable` to test queries.
- Treating InMemory provider as a faithful SQL Server substitute.
- Cross-aggregate navigations in DDD models.
- Using Dapper writes inside EF tx without passing the transaction.

**Cheat sheet**
- `DbContext` = UoW, `DbSet` = repository.
- Mix EF + Dapper by sharing `DbConnection` + `DbTransaction`.
- Specification = reusable query object applied to `IQueryable`.
- CQRS: tracked aggregates for writes, `AsNoTracking` projections/Dapper for reads.
- Test with real SQL Server (Testcontainers) + Respawn; avoid InMemory.

---

## Topic 16: Large-table migrations & production scenario ladders

**Study notes**

- **Migrations basics**: `dotnet ef migrations add`, `database update`, `migrations script --idempotent`, and **migration bundles** (`dotnet ef migrations bundle`) for CI/CD. Avoid `Database.Migrate()` at app startup in multi-instance/blue-green deployments (race conditions; EF 9 adds a migration lock but still prefer a dedicated deploy step).
- **Large tables**: operations that rewrite or lock (ALTER COLUMN type change, adding NOT NULL column with default on older SQL Server editions, index builds, FK validation with `WITH CHECK`) can block for hours. Use **expand/contract**: add nullable column → deploy code writing both → backfill in batches → add constraint → switch reads → drop old.
- **Online index operations**: SQL Server Enterprise/Azure SQL: `CreateIndex(...).Annotation("SqlServer:Online", true)`. Add `WITH (MAXDOP, SORT_IN_TEMPDB)` via raw SQL when needed.
- **Renames**: EF may scaffold drop+add for a rename (data loss!) — edit to `RenameColumn/RenameTable`.
- Always review the generated migration and script; test against a production-sized copy.

### 16.1 [Easy] How do you apply EF migrations safely in production?
**Answer**
Generate an idempotent SQL script (or a migration bundle) in CI, review it, and apply it in a controlled deployment step with proper permissions — not by calling `Migrate()` from every app instance at startup. Take backups and have a rollback plan (down migrations are rarely safe for data).

**Explanation**
A script lets a DBA review and tune locking; a bundle is a self-contained executable that applies pending migrations. Run with a privileged identity separate from the app's runtime identity (least privilege).

**Code**
```bash
dotnet ef migrations script --idempotent --output deploy.sql
dotnet ef migrations bundle --self-contained -r win-x64 --output efbundle.exe
./efbundle.exe --connection "<connection string>"
```

### 16.2 [Follow-up] How do you add a NOT NULL column to a 500M-row table without downtime?
**Answer**
Use expand/contract: add the column nullable (metadata-only), have the app write it, backfill existing rows in small batches, then add the NOT NULL constraint (and default) once complete. Avoid a single huge `UPDATE` that bloats the log and holds locks.

**Explanation**
Batched backfill with `TOP (n)` loops or key ranges, with brief delays/`WAITFOR`, keeps transactions short and replication/log manageable. Add constraints `WITH NOCHECK` then validate with `CHECK CONSTRAINT` when needed (validation scans but doesn't block like full rewrite in modern versions; still plan it).

**Code**
```csharp
migrationBuilder.AddColumn<string>(name: "Region", table: "Orders", type: "nvarchar(20)", nullable: true);

// Separate migration/job after deploy of new code:
migrationBuilder.Sql(@"
WHILE 1 = 1
BEGIN
    UPDATE TOP (5000) Orders SET Region = 'EU' WHERE Region IS NULL;
    IF @@ROWCOUNT = 0 BREAK;
    WAITFOR DELAY '00:00:00.100';
END", suppressTransaction: true);

// Final migration:
migrationBuilder.AlterColumn<string>(name: "Region", table: "Orders", type: "nvarchar(20)", nullable: false, defaultValue: "EU");
```

### 16.3 [Medium] How do you create or change indexes on a big table with minimal blocking?
**Answer**
Create indexes `ONLINE = ON` (edition permitting), do it off-peak, consider `RESUMABLE`, and verify via the actual plan that the new index is used. EF exposes the online option via an annotation on `CreateIndex`.

**Explanation**
Online builds still need brief schema locks and extra tempdb/log. For nullable-column filters or soft-delete flags, filtered indexes keep indexes small. Index changes are also expressed in the model (`HasIndex(...).IncludeProperties(...)`, `.HasFilter(...)`) so the model snapshot stays in sync.

**Code**
```csharp
modelBuilder.Entity<Order>().HasIndex(o => new { o.CustomerId, o.CreatedAt })
    .IncludeProperties(o => o.Total);

migrationBuilder.CreateIndex(
    name: "IX_Orders_CustomerId_CreatedAt", table: "Orders",
    columns: new[] { "CustomerId", "CreatedAt" })
    .Annotation("SqlServer:Include", new[] { "Total" })
    .Annotation("SqlServer:Online", true);
```
```sql
CREATE INDEX [IX_Orders_CustomerId_CreatedAt] ON [Orders] ([CustomerId], [CreatedAt]) INCLUDE ([Total]) WITH (ONLINE = ON);
```

### 16.4 [Hard] A property rename generated a DROP COLUMN + ADD COLUMN in the migration. What do you do, and what other migration hazards exist?
**Answer**
Edit the migration to use `RenameColumn` (or `RenameTable`) so data is preserved. Always read generated migrations: other hazards include type changes causing table rewrites, dropping columns/tables, adding FKs with validation on huge tables, and default-value changes.

**Explanation**
EF can't know a rename vs drop/add unless it's declared; it detects renames only sometimes by similar names. Data-moving migrations should be tested on a copy with production volume. Squashing migrations and model snapshot conflicts in merged branches are other team-level problems; resolve snapshot conflicts by removing and re-adding the latest migration.

**Code**
```csharp
// Replace generated drop/add
migrationBuilder.RenameColumn(name: "FullName", table: "Customers", newName: "DisplayName");

// FK on big table: add without validating existing rows, validate later
migrationBuilder.Sql("ALTER TABLE Orders WITH NOCHECK ADD CONSTRAINT FK_Orders_Customers FOREIGN KEY (CustomerId) REFERENCES Customers(Id)");
migrationBuilder.Sql("ALTER TABLE Orders WITH CHECK CHECK CONSTRAINT FK_Orders_Customers");
```

**Gotchas / follow-ups**
- Untrusted (NOCHECK) constraints are ignored by the optimizer until re-validated.
- Rollback: write explicit `Down` only if data-safe; prefer forward fixes.

### 16.5 [Tricky] [Production] A query that ran fast yesterday is suddenly slow. How do you investigate?
**Answer**
Confirm what changed (data volume, deployment, stats, parameter values, plan regression). Compare the current plan to the previous one in Query Store, look for parameter sniffing, outdated statistics, missing/changed indexes, blocking, or an EF change that altered the SQL (e.g. a new Include/filter, `Contains` via `OPENJSON`).

**Explanation**
Steps: (1) get the actual SQL from logs/`TagWith`/APM; (2) run it in SSMS with actual parameter values and the actual plan; (3) Query Store → regressed queries, force the last good plan as a stop-gap; (4) `UPDATE STATISTICS`; (5) check blocking/waits (`sys.dm_exec_requests`, `sys.dm_os_wait_stats`); (6) check for implicit conversions (e.g. `nvarchar` parameter vs `varchar` column → index scan; map with `IsUnicode(false)`); (7) fix permanently (index, rewrite query, hint) and add a regression test/alert.

**Code**
```csharp
// Implicit conversion fix: column is varchar(20), EF sends nvarchar by default
modelBuilder.Entity<Customer>().Property(c => c.Code).HasMaxLength(20).IsUnicode(false);

var orders = await db.Orders.TagWith("OrdersByCustomer - Orders API").Where(o => o.CustomerId == id).ToListAsync();
```
```sql
-- Query Store: top regressed queries
SELECT TOP 10 q.query_id, qt.query_sql_text, rs.avg_duration, rs.count_executions
FROM sys.query_store_query q
JOIN sys.query_store_query_text qt ON q.query_text_id = qt.query_text_id
JOIN sys.query_store_plan p ON q.query_id = p.query_id
JOIN sys.query_store_runtime_stats rs ON p.plan_id = rs.plan_id
ORDER BY rs.avg_duration DESC;
EXEC sp_query_store_force_plan @query_id = 42, @plan_id = 7;
```

**Gotchas / follow-ups**
- Same query slow only for some parameter values = sniffing/skew.
- Slow only from the app, fast in SSMS = different SET options (ARITHABORT) → different cached plan.
- Don't "fix" with index spam; each index slows writes.

### 16.6 [Tricky] [Production] Memory spikes in the API after a release. What do you look at in EF Core?
**Answer**
Look for queries now returning large result sets (missing paging, new `Include`s), tracking of big graphs, a long-lived or singleton DbContext, and materializing with `ToList()` instead of streaming. Capture a memory dump/`dotnet-gcdump` and check which entity types dominate.

**Explanation**
Fixes: `AsNoTracking`, projection, paging, `AsAsyncEnumerable()` streaming for exports, short DbContext lifetime (scoped per request, `IDbContextFactory` in background services), `ChangeTracker.Clear()` in batch loops. Also check large object heap (LOH) allocations from big `byte[]`/string columns — don't select blob columns unless needed (or use streaming `GetStream`). Check the number of cached query shapes/service providers.

**Code**
```csharp
await foreach (var row in db.Orders.AsNoTracking()
    .Where(o => o.CreatedAt >= from)
    .Select(o => new OrderExportRow(o.Id, o.Total))
    .AsAsyncEnumerable().WithCancellation(ct))
{
    await writer.WriteRowAsync(row, ct);   // stream, don't buffer
}
```
```
dotnet-gcdump collect -p <pid>      # see top types by size
dotnet-counters monitor -p <pid> System.Runtime Microsoft.EntityFrameworkCore
```

**Gotchas / follow-ups**
- Streaming keeps the connection open during enumeration; slow consumers hold the connection.
- With `AsAsyncEnumerable` + split query or EnableRetryOnFailure, buffering rules/limitations may apply (retry strategies buffer or disallow streaming mid-failure).

### 16.7 [Tricky] [Production] Users see stale or wrong data. Cache? DbContext lifetime? How do you find the cause?
**Answer**
Common causes: a long-lived/singleton DbContext whose change tracker returns tracked (stale) instances instead of fresh DB values, application-level caching (`IMemoryCache`) without invalidation, `ExecuteUpdate`/Dapper/other services changing rows behind tracked entities, read replicas lagging, or isolation-level/transaction assumptions.

**Explanation**
EF's identity resolution means a query returns the *already tracked instance* and does not overwrite its loaded property values with new DB values (the row is read, but the tracked entity's current values stay). Fix with scoped contexts, `AsNoTracking`, `entry.ReloadAsync()`, or `ChangeTracker.Clear()`. For caches: define TTL and explicit invalidation, include tenant/user in keys, don't cache tracked entities (cache DTOs). For replicas: read-your-writes routing. Also check that `ExecuteUpdate` was followed by a clear/reload.

**Code**
```csharp
// Stale because 'product' is tracked from an earlier read in the same long-lived context
var product = await db.Products.FindAsync(id);                     // returns tracked copy
await db.Products.Where(p => p.Id == id).ExecuteUpdateAsync(s => s.SetProperty(p => p.Price, 99m), ct);
var again = await db.Products.SingleAsync(p => p.Id == id);        // STILL old Price (tracked instance)

await db.Entry(again).ReloadAsync(ct);                             // refresh
// or: var fresh = await db.Products.AsNoTracking().SingleAsync(p => p.Id == id, ct);

// Cache DTOs with explicit invalidation, not entities
var dto = await cache.GetOrCreateAsync($"product:{tenantId}:{id}", async e =>
{
    e.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5);
    return await db.Products.AsNoTracking().Where(p => p.Id == id).Select(p => new ProductDto(p.Id, p.Price)).SingleAsync(ct);
});
```

**Gotchas / follow-ups**
- Registering `DbContext` as Singleton or injecting a scoped context into a singleton (captive dependency) is the #1 source.
- `AddDbContextPool` resets state, but custom per-request state you add yourself must be reset.
- Caching with `IMemoryCache` in a multi-instance deployment needs distributed invalidation (Redis, pub/sub) or short TTLs.

**Interview traps**
- `Migrate()` at startup in every instance of a scaled-out app.
- Letting EF scaffold a rename as drop/add.
- Blaming "EF is slow" before checking the actual SQL/plan/implicit conversions.
- Assuming a query refreshes already-tracked entities' values.
- Fixing slowness by only raising timeouts or adding indexes blindly.

**Cheat sheet**
- Deploy: idempotent script or bundle, not startup `Migrate()`.
- Big tables: expand/contract, batched backfill, `ONLINE = ON` indexes, `NOCHECK` then validate.
- Renames: hand-edit to `RenameColumn/RenameTable`.
- Slow query triage: actual SQL → actual plan → Query Store/stats/sniffing/implicit conversion → blocking.
- Memory: `AsNoTracking`, projection, paging, streaming, short-lived contexts.
- Stale data: tracked instances win; reload/clear/scoped context; cache DTOs with invalidation.

---

*End of C02.*
