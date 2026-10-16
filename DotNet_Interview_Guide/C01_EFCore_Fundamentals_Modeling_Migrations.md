# C01 – EF Core Fundamentals, Modeling & Migrations

**How to use:** Read each topic's *Study notes* first, then try answering every question in the ladder out loud before reading the **Answer**. Climb rung by rung (Easy → Tricky); an interviewer typically stops where your depth ends.
**Scope:** EF Core 8 on SQL Server. Code uses C# 12 / .NET 8 unless noted.

## Table of Contents
1. [Topic 1: ORM concept – EF Core vs EF6 vs Dapper vs ADO.NET](#topic-1-orm-concept--ef-core-vs-ef6-vs-dapper-vs-adonet)
2. [Topic 2: DbContext lifetime & registration](#topic-2-dbcontext-lifetime--registration)
3. [Topic 3: DbSet & how LINQ translates to SQL](#topic-3-dbset--how-linq-translates-to-sql)
4. [Topic 4: Change tracking](#topic-4-change-tracking)
5. [Topic 5: SaveChanges internals](#topic-5-savechanges-internals)
6. [Topic 6: Model configuration – conventions, annotations, Fluent API & keys](#topic-6-model-configuration--conventions-annotations-fluent-api--keys)
7. [Topic 7: Relationships & cascade delete](#topic-7-relationships--cascade-delete)
8. [Topic 8: Owned types, complex types & inheritance mapping](#topic-8-owned-types-complex-types--inheritance-mapping)
9. [Topic 9: Value converters, JSON columns, shadow/backing fields, computed, defaults, sequences](#topic-9-value-converters-json-columns-shadowbacking-fields-computed-defaults-sequences)
10. [Topic 10: Global query filters, table splitting, keyless entities, views & stored procedures](#topic-10-global-query-filters-table-splitting-keyless-entities-views--stored-procedures)
11. [Topic 11: Migrations](#topic-11-migrations)
12. [Topic 12: Seeding & Code-first vs Database-first](#topic-12-seeding--code-first-vs-database-first)
13. [Topic 13: Loading related data – eager, lazy, explicit](#topic-13-loading-related-data--eager-lazy-explicit)
14. [Topic 14: Pooling, compiled models, multiple DbContexts & bounded contexts](#topic-14-pooling-compiled-models-multiple-dbcontexts--bounded-contexts)
15. [Topic 15: Logging & inspecting SQL](#topic-15-logging--inspecting-sql)

Shared sample model used in many snippets:

```csharp
public class Customer { public int Id { get; set; } public string Name { get; set; } = ""; public List<Order> Orders { get; set; } = new(); }
public class Order    { public int Id { get; set; } public int CustomerId { get; set; } public Customer Customer { get; set; } = null!;
                        public decimal Total { get; set; } public DateTime CreatedUtc { get; set; } public List<OrderLine> Lines { get; set; } = new(); }
public class OrderLine{ public int Id { get; set; } public int OrderId { get; set; } public string Sku { get; set; } = ""; public int Qty { get; set; } }
```

---

## Topic 1: ORM concept – EF Core vs EF6 vs Dapper vs ADO.NET

**Study notes**

An ORM (object-relational mapper) maps rows ↔ objects, translates object queries into SQL, tracks changes and generates INSERT/UPDATE/DELETE. It addresses the *impedance mismatch* (graphs vs tables, inheritance, identity, types).

```
Abstraction / convenience  (high)                              (low)
EF Core / EF6  >  Dapper (micro-ORM)  >  ADO.NET (SqlConnection/SqlCommand/SqlDataReader)
Performance control        (lower)                              (max)
```

| Feature | ADO.NET | Dapper | EF6 | EF Core |
|---|---|---|---|---|
| SQL authoring | Manual | Manual | Generated (LINQ) | Generated (LINQ) + raw SQL |
| Change tracking / unit of work | No | No | Yes | Yes |
| Mapping | Manual reader | Auto by column name | Full model | Full model |
| Platform | .NET FW + .NET | both | .NET Framework (EF 6.3+ also runs on .NET Core 3+) | .NET 8+ / cross-platform |
| Migrations | No | No | Yes | Yes |
| Extensibility | – | – | Limited | Interceptors, conventions, compiled models, bulk `ExecuteUpdate/Delete` |

EF Core is a rewrite (not an upgrade) of EF6: no EDMX/designer, no lazy loading by default, many-to-many without a join entity (since 5), `ExecuteUpdate`/`ExecuteDelete` (7), TPC (7), complex types (8), raw SQL for unmapped types (8).

### 1.1 [Easy] What is an ORM and why use one?
**Answer**
An ORM maps relational tables to object types and translates queries and changes between the two. It removes boilerplate SQL/mapping code, gives compile-time-checked queries (LINQ), and provides change tracking and a unit of work. The cost is less control over SQL and some overhead.

**Explanation**
Without an ORM you write SQL strings, open connections, read columns by ordinal/name and convert types. An ORM centralises that: a model describes the mapping, LINQ becomes parameterised SQL (protecting against SQL injection), and `SaveChanges` computes the diff. Trade-offs: leaky abstraction, N+1 pitfalls, generated SQL that may be sub-optimal, learning curve.

**Code**
```csharp
var big = await db.Orders.Where(o => o.Total > 100).OrderBy(o => o.CreatedUtc).ToListAsync();
```
```sql
SELECT [o].[Id], [o].[CustomerId], [o].[CreatedUtc], [o].[Total]
FROM [Orders] AS [o] WHERE [o].[Total] > 100.0 ORDER BY [o].[CreatedUtc]
```

### 1.2 [Follow-up] When would you choose Dapper or ADO.NET over EF Core?
**Answer**
Dapper for hot read paths, complex hand-tuned SQL, reporting, and stored-procedure-heavy code where you want near-ADO.NET speed with less boilerplate. ADO.NET for bulk copy (`SqlBulkCopy`), streaming huge results, or very specific provider features. EF Core is best for CRUD-heavy domain code where change tracking and migrations pay off.

**Explanation**
Many teams mix: EF Core for writes/commands, Dapper for read-side queries (a pragmatic CQRS). They can share the same connection and transaction via `db.Database.GetDbConnection()` and `db.Database.CurrentTransaction.GetDbTransaction()`. Modern EF Core with `AsNoTracking`, compiled queries and projections narrows the gap with Dapper considerably.

**Code**
```csharp
// Dapper sharing EF's connection and transaction
await using var tx = await db.Database.BeginTransactionAsync();
var conn = db.Database.GetDbConnection();
var rows = await conn.QueryAsync<OrderDto>(
    "SELECT Id, Total FROM Orders WHERE CustomerId = @id", new { id = 5 },
    transaction: tx.GetDbTransaction());
await tx.CommitAsync();
```

### 1.3 [Medium] What are the main differences between EF Core and EF6 that matter in practice?
**Answer**
EF Core is a cross-platform, modular rewrite with a different query pipeline, no EDMX, no automatic lazy loading, and different defaults (e.g. cascade/required semantics, no implicit many-to-many join entity before EF5). It added bulk `ExecuteUpdate/Delete`, TPC, JSON columns, interceptors and compiled models. Some EF6 features (e.g. spatial in older versions, complex-type mapping parity, `Database.SqlQuery` behaviors) differ and migration between them is a port, not a recompile.

**Explanation**
EF6 translated whole queries client-side-rarely; early EF Core silently evaluated parts on the client (removed in 3.0 – now throws). EF Core 3.0 also changed single-SQL-query with Include (cartesian explosion), later mitigated with `AsSplitQuery` (5). Mapping differences: `HasRequired/WithMany` → `HasOne/WithMany().IsRequired()`; many-to-many uses skip navigations. Migrations: EF Core's snapshot (`ModelSnapshot.cs`) replaces EF6's `__MigrationHistory` model blob.

**Code**
```csharp
// EF Core: set-based update without loading entities (EF6 couldn't)
await db.Orders.Where(o => o.CreatedUtc < cutoff).ExecuteDeleteAsync();
```
```sql
DELETE FROM [o] FROM [Orders] AS [o] WHERE [o].[CreatedUtc] < @cutoff
```

### 1.4 [Hard] What does an ORM cost you at runtime, and how do you mitigate it in EF Core?
**Answer**
Costs: model building at startup, LINQ expression-tree compilation and translation per query shape (cached), change-tracking snapshots, and materialization. Mitigate with `AsNoTracking`, projections (`Select` to DTOs), compiled queries (`EF.CompileAsyncQuery`), compiled models, `DbContext` pooling, split queries, and `ExecuteUpdate/Delete` for set-based work.

**Explanation**
First use of a `DbContext` type builds the model (can take hundreds of ms for large models) — compiled models (`dotnet ef dbcontext optimize`) reduce that. Each query: the expression tree is parameterised and looked up in a query cache keyed by shape; a cache miss costs translation. Tracking queries allocate entry objects and snapshots for every row. Reading only needed columns reduces I/O and tracker overhead.

**Code**
```csharp
static readonly Func<AppDb, int, IAsyncEnumerable<Order>> ByCustomer =
    EF.CompileAsyncQuery((AppDb db, int id) => db.Orders.AsNoTracking().Where(o => o.CustomerId == id));

await foreach (var o in ByCustomer(db, 5)) { /* ... */ }
```

### 1.5 [Tricky] "EF Core is slow" – what real mistakes usually cause it?
**Answer**
Almost always usage errors: N+1 queries, loading full entities when a projection suffices, tracking read-only data, `Include` chains producing cartesian explosions, client-side evaluation in final projection, calling `ToList()` too early and filtering in memory, and `SaveChanges` in a loop.

**Explanation**
Premature `ToList()`/`AsEnumerable()` turns the rest of the query into LINQ-to-Objects. `Contains` on large in-memory lists produces huge `IN`/`OPENJSON` (EF Core 8 uses `OPENJSON` for parameterised collections on SQL Server ≥ 2016 compat level 130). Always measure with logs/profiler before assuming the ORM is at fault.

**Code**
```csharp
// BAD: N+1 and in-memory filtering
var list = db.Customers.ToList().Where(c => c.Name.StartsWith("A"));
foreach (var c in list) Console.WriteLine(c.Orders.Count); // lazy/explicit loads per row

// GOOD: one query projection
var dto = await db.Customers.Where(c => c.Name.StartsWith("A"))
    .Select(c => new { c.Name, OrderCount = c.Orders.Count }).ToListAsync();
```
**Gotchas / follow-ups**
- Final-projection client evaluation is still allowed (only the top-level `Select`); anything elsewhere throws `InvalidOperationException`.
- Don't "fix" slowness by switching ORMs before reading the SQL and execution plan.

**Interview traps**
- Saying "EF Core is EF7/EF6 upgraded" – it's a rewrite.
- Claiming Dapper is always faster; for tiny queries the difference is micro-seconds, and EF adds tracking only if you let it.
- Forgetting that EF's LINQ → parameterised SQL protects from injection, while `FromSqlRaw` with string concatenation does not.
- Believing ORM means you needn't know SQL.

**Cheat sheet**
- ORM = mapping + query translation + change tracking + unit of work.
- ADO.NET (max control) → Dapper (fast, manual SQL) → EF (productive, generated SQL).
- Mix them: EF for writes, Dapper for reads, shared connection/transaction.
- EF Core ≠ EF6: no EDMX, no auto lazy loading, ExecuteUpdate/Delete, TPC, JSON, interceptors.
- Perf fixes: AsNoTracking, projection, compiled queries/models, pooling.

---

## Topic 2: DbContext lifetime & registration

**Study notes**

`DbContext` = a **unit of work + repository-ish gateway**: holds a connection (lazily opened/closed per operation), a change tracker (identity map), and the model. It is **lightweight to create, not thread-safe, short-lived**.

```
HTTP request ──► DI scope ──► AppDb (Scoped) ──► tracker + connection (from ADO pool)
                                  └── disposed at end of request
Background service (Singleton) ──► IDbContextFactory<AppDb> ──► CreateDbContext() per unit of work
```

- `AddDbContext<T>` registers `T` as **Scoped** (options are Scoped by default too).
- `AddDbContextPool<T>` reuses instances (reset between uses); lifetime still scoped from the consumer's perspective.
- `AddDbContextFactory<T>` registers `IDbContextFactory<T>` (singleton by default) – for Blazor Server, background services, parallel work.
- Connection pooling (ADO.NET) is separate from DbContext pooling.

### 2.1 [Easy] How do you register a DbContext and what is its default lifetime?
**Answer**
`services.AddDbContext<AppDb>(o => o.UseSqlServer(connString))`. It is registered as Scoped, so one instance per HTTP request (per DI scope).

**Explanation**
Scoped fits the unit-of-work pattern: all code in a request shares one tracker, so entities loaded in one service are the same instances elsewhere in that request, and a single `SaveChanges` commits the work. The context should be disposed at the end of the scope (the container does it).

**Code**
```csharp
builder.Services.AddDbContext<AppDb>(o =>
    o.UseSqlServer(builder.Configuration.GetConnectionString("Default"),
        sql => sql.EnableRetryOnFailure()));

public class AppDb(DbContextOptions<AppDb> options) : DbContext(options)
{
    public DbSet<Customer> Customers => Set<Customer>();
    public DbSet<Order> Orders => Set<Order>();
}
```

### 2.2 [Follow-up] Is DbContext thread-safe? What happens if you run two queries in parallel on it?
**Answer**
No. Concurrent use throws `InvalidOperationException` ("A second operation was started on this context instance before a previous operation completed"). Don't share a context across threads or fire un-awaited async calls; use one context per parallel operation (via a factory).

**Explanation**
The tracker and connection state are not synchronized. The classic bug is `Task.WhenAll(db.A.ToListAsync(), db.B.ToListAsync())` on the same instance, or forgetting an `await`. Parallel code should create separate contexts through `IDbContextFactory<T>`.

**Code**
```csharp
// WRONG
await Task.WhenAll(db.Orders.ToListAsync(), db.Customers.ToListAsync());

// RIGHT
var t1 = Task.Run(async () => { await using var c = await factory.CreateDbContextAsync(); return await c.Orders.ToListAsync(); });
var t2 = Task.Run(async () => { await using var c = await factory.CreateDbContextAsync(); return await c.Customers.ToListAsync(); });
await Task.WhenAll(t1, t2);
```

### 2.3 [Medium] When do you use AddDbContextFactory, and how is it different from AddDbContext?
**Answer**
Use it where there is no natural scope per operation: Blazor Server circuits (long-lived scope), hosted/background services (singleton), or parallel work. You inject `IDbContextFactory<T>` and create/dispose a short-lived context per operation. `AddDbContext` gives you one scoped instance.

**Explanation**
In Blazor Server a DI scope lives for the whole circuit, so a scoped context would live for minutes/hours, accumulating tracked entities and stale data, and component events can overlap (concurrency). A hosted `BackgroundService` is singleton; injecting a scoped `DbContext` into it fails DI validation ("Cannot consume scoped service from singleton"). Factory fixes both. You can also combine: `AddDbContextFactory` plus resolving scoped `AppDb` via `AddScoped(sp => sp.GetRequiredService<IDbContextFactory<AppDb>>().CreateDbContext())`. `AddPooledDbContextFactory` pools instances.

**Code**
```csharp
builder.Services.AddDbContextFactory<AppDb>(o => o.UseSqlServer(cs));

public class Cleanup(IDbContextFactory<AppDb> factory) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            await using var db = await factory.CreateDbContextAsync(ct);
            await db.Orders.Where(o => o.CreatedUtc < DateTime.UtcNow.AddYears(-7)).ExecuteDeleteAsync(ct);
            await Task.Delay(TimeSpan.FromHours(1), ct);
        }
    }
}
```

### 2.4 [Hard] How does AddDbContextPool work and what are its restrictions?
**Answer**
It keeps a pool of `DbContext` instances; on scope end the context is *reset* (tracker cleared, state reset) and returned instead of disposed/recreated. It saves allocation/initialisation cost under high throughput. Restrictions: the context must not hold per-request state in fields, and constructor injection of scoped services into the context is not supported.

**Explanation**
Because instances are reused, any custom state (e.g. a `TenantId` set in a field) leaks into the next request. Per-request values should be supplied via `OnConfiguring`-independent mechanisms – e.g. read from a scoped service through a lambda, interceptors that resolve from `RequestServices`, or by using a non-pooled context. The options are fixed for all pooled instances. Default pool size is 1024 (`poolSize` parameter). The gain is small (microseconds) – benchmark first. On return, the context's state (change tracker, etc.) is reset.

**Code**
```csharp
builder.Services.AddDbContextPool<AppDb>(o => o.UseSqlServer(cs), poolSize: 256);
```
**Gotchas / follow-ups**
- Don't make the context depend on scoped services via constructor; use `IDbContextFactory` or resolve at call time.
- A pooled context that you manually `Dispose` early still returns to the pool at scope end – don't touch it afterward.

### 2.5 [Tricky] A singleton service takes a scoped DbContext "and it works in dev". Why is this dangerous?
**Answer**
Scope validation is only on in Development (`ValidateScopes`), so in production a captive scoped context can end up living for the app lifetime. It becomes a shared, long-lived, non-thread-safe context with an ever-growing tracker and stale data.

**Explanation**
This is the *captive dependency* anti-pattern. Symptoms: random concurrency exceptions under load, memory growth, stale reads (tracker returns cached entities, not DB values), connection never released. Fix by injecting `IDbContextFactory<T>` or `IServiceScopeFactory` and creating a scope per operation. Also note `ValidateOnBuild` can detect captive dependencies at startup.

**Code**
```csharp
public class Worker(IServiceScopeFactory scopes)
{
    public async Task RunAsync()
    {
        using var scope = scopes.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDb>();
        // work
        await db.SaveChangesAsync();
    }
}
```

**Interview traps**
- "DbContext is a connection" – it's a unit of work; the connection is borrowed from the ADO.NET pool only while needed.
- Registering DbContext as Singleton "for performance".
- Fire-and-forget `SaveChangesAsync()` without await.
- Thinking `DbContextPool` and SQL connection pool are the same thing.
- Disposing a context while a returned `IQueryable` is still unmaterialised.

**Cheat sheet**
- AddDbContext → Scoped; one per request.
- Not thread-safe: one operation at a time per instance.
- Background/Blazor Server → `IDbContextFactory<T>`.
- Pooling = reuse + reset; no per-request state in context fields.
- Never capture a scoped context in a singleton.

---

## Topic 3: DbSet & how LINQ translates to SQL

**Study notes**

`DbSet<T>` is the query root + entity collection (`Add`, `Find`, `Remove`...). It implements `IQueryable<T>`: LINQ operators build an **expression tree**; nothing runs until enumeration (`ToList`, `First`, `Count`, `foreach`, `await`).

```
LINQ expression tree ─► parameter extraction (closure values → @p) ─► query cache lookup
   ─► Translation to SQL expression tree (relational) ─► SQL generation ─► DbCommand
   ─► DbDataReader ─► materialization (+ identity resolution if tracking)
```

- `IQueryable` operators translate to SQL; `IEnumerable` operators (after `AsEnumerable()`/`ToList()`) run in memory.
- Closure variables become **parameters** (query plan reuse); constants stay literal.
- Since EF Core 3.0 untranslatable expressions throw (except in the final top-level `Select`).
- `Find(key)` checks the tracker first, then queries; `FirstOrDefault` always hits the DB.
- `Skip/Take` → `OFFSET/FETCH` (requires `ORDER BY` for determinism).

### 3.1 [Easy] What is deferred execution and what triggers it to run?
**Answer**
A LINQ query on `IQueryable` only describes the query; SQL is sent when you enumerate it (`ToList`, `ToListAsync`, `First`, `Single`, `Count`, `Any`, `foreach`). Re-enumerating re-executes the query.

**Explanation**
Because the query is composable, you can add `Where`/`OrderBy` conditionally before executing. Beware that the closure is evaluated at execution, so changing a captured variable before enumeration changes the parameter value.

**Code**
```csharp
IQueryable<Order> q = db.Orders;               // no SQL yet
if (minTotal.HasValue) q = q.Where(o => o.Total >= minTotal);
var page = await q.OrderBy(o => o.Id).Skip(20).Take(10).ToListAsync();  // SQL runs here
```
```sql
SELECT [o].[Id], ... FROM [Orders] AS [o] WHERE [o].[Total] >= @__minTotal_0
ORDER BY [o].[Id] OFFSET @__p_1 ROWS FETCH NEXT @__p_2 ROWS ONLY
```

### 3.2 [Follow-up] IQueryable vs IEnumerable – what's the practical difference?
**Answer**
`IQueryable` carries an expression tree the provider translates to SQL; `IEnumerable` carries delegates executed in memory. Switching to `IEnumerable` early (e.g. returning `IEnumerable` from a repository or calling `AsEnumerable`) makes later filters run client-side after pulling all rows.

**Explanation**
`Where(Func<T,bool>)` (delegate) vs `Where(Expression<Func<T,bool>>)` (expression) is chosen by static type. Passing a `Func<>` into a method that calls `db.Orders.Where(func)` silently selects `Enumerable.Where` and loads the whole table. Use `Expression<Func<..>>` for reusable predicates.

**Code**
```csharp
Expression<Func<Order, bool>> big = o => o.Total > 100;    // translates
Func<Order, bool> bigFn = o => o.Total > 100;              // runs in memory!
var ok  = await db.Orders.Where(big).ToListAsync();
var bad = db.Orders.Where(bigFn).ToList(); // SELECT * FROM Orders, filters in C#
```

### 3.3 [Medium] How does EF Core handle a LINQ expression it can't translate? What about calling your own C# method?
**Answer**
Since 3.0 it throws `InvalidOperationException` ("could not be translated") rather than silently evaluating on the client. Custom methods can't be translated unless mapped with `HasDbFunction`/`[DbFunction]` or you move them to the final projection or pre-compute the value.

**Explanation**
Exception: the outermost `Select` projection may call arbitrary C# after the needed columns are fetched. Options for custom logic: `EF.Functions.*` (Like, DateDiffDay, Contains for full-text), map a SQL scalar UDF with `[DbFunction]`, use `AsEnumerable()` deliberately after reducing the dataset, or write raw SQL (`FromSql`). Also, `string.Contains` → `LIKE '%x%'` (or `CHARINDEX`), `ToLower()` → `LOWER()` (collation matters; SQL Server default collation is case-insensitive).

**Code**
```csharp
var q = db.Customers
    .Where(c => EF.Functions.Like(c.Name, "A%"))
    .Select(c => new { c.Id, Label = Format(c.Name) }); // Format runs client-side – top-level Select only

static string Format(string s) => s.Trim().ToUpperInvariant();
```

### 3.4 [Hard] Explain parameterisation and the query cache. Why do constants vs variables matter?
**Answer**
EF extracts captured variables into SQL parameters, so the same query shape with different values reuses the compiled query (EF cache) and SQL Server's plan. Inline constants in the expression tree become literals, creating different shapes and plans. Use `EF.Constant` / `EF.Parameter` (EF8) to override the default per-expression.

**Explanation**
Two caches: EF's compiled-query cache (keyed by expression-tree structure after parameter extraction) and SQL Server's plan cache (keyed by SQL text). Literal-heavy queries pollute both. Conversely, parameter sniffing can give a bad plan for skewed values; `EF.Constant(x)` forces a literal when a per-value plan is preferable. `Contains` over a local list: EF 8 on SQL Server sends the list as a single JSON parameter using `OPENJSON` so the SQL text is stable (earlier versions expanded `IN (@p1,@p2…)`, making plan-cache bloat).

**Code**
```csharp
var ids = new[] { 1, 2, 3 };
var q = db.Orders.Where(o => ids.Contains(o.Id));
// EF8: WHERE [o].[Id] IN (SELECT [i].[value] FROM OPENJSON(@__ids_0) WITH ([value] int '$') AS [i])
var q2 = db.Orders.Where(o => o.Total > EF.Constant(100m)); // literal in SQL
```
**Gotchas / follow-ups**
- Calling `.ToQueryString()` shows parameters declared at top; it's a debugging aid, not a copy-paste-safe SQL for production.
- EF's own cache is bounded; dynamically built expression shapes (e.g. varying numbers of predicates) can thrash it.

### 3.5 [Tricky] `Find` vs `FirstOrDefault` vs `SingleOrDefault` vs `Any` – differences and gotchas?
**Answer**
`Find` checks the change tracker by primary key first and only queries if absent; the others always query. `FirstOrDefault` adds `TOP(1)`; `SingleOrDefault` uses `TOP(2)` and throws if more than one match. `Any()` is the efficient existence check (`EXISTS`), not `Count() > 0`.

**Explanation**
`Find` can return a tracked entity with unsaved/stale in-memory values and also returns *Added* entities not yet in the DB; it also doesn't work with `AsNoTracking` or `Include`. `First` without `OrderBy` gives a non-deterministic row (EF warns). `Single` communicates uniqueness expectations and fails loudly. `Count()>0` forces a full count scan.

**Code**
```csharp
var a = await db.Customers.FindAsync(5);                          // tracker, then DB
var b = await db.Customers.FirstOrDefaultAsync(c => c.Id == 5);   // always DB
bool exists = await db.Orders.AnyAsync(o => o.CustomerId == 5);
```
```sql
SELECT CASE WHEN EXISTS (SELECT 1 FROM [Orders] AS [o] WHERE [o].[CustomerId] = 5) THEN CAST(1 AS bit) ELSE CAST(0 AS bit) END
```

**Interview traps**
- Returning `IQueryable` from repositories leaks persistence concerns and risks deferred execution after the context is disposed.
- Passing `Func<>` instead of `Expression<Func<>>`.
- `OrderBy` after `Skip/Take` or missing `OrderBy` before paging.
- Assuming `ToLower()` comparisons are needed for case-insensitivity on SQL Server (default collation is CI) – and they defeat index usage.
- Believing the final `Select` client evaluation also applies to `Where`.

**Cheat sheet**
- `DbSet<T>` : `IQueryable<T>` → expression tree → SQL on enumeration.
- Variables → parameters; constants → literals; `EF.Constant/Parameter` override (EF8).
- Untranslatable → exception (except top-level Select).
- `Find` = tracker first; `Any` = EXISTS.
- Always `OrderBy` before `Skip/Take`.

---

## Topic 4: Change tracking

**Study notes**

The `ChangeTracker` holds an `EntityEntry` per tracked entity with a **state**, current values and (for snapshot tracking) original values. It's also the **identity map**: one instance per (type, key) per context.

| State | Meaning | On SaveChanges |
|---|---|---|
| Detached | Not tracked | nothing |
| Added | New, no row yet | INSERT |
| Unchanged | Tracked, equals DB | nothing |
| Modified | ≥1 property changed | UPDATE (changed columns only) |
| Deleted | Marked for delete | DELETE |

```
Query (tracking) ─► entry Unchanged + snapshot copy of values
Set property     ─► DetectChanges() compares current vs snapshot ─► Modified (per property)
SaveChanges      ─► DetectChanges ─► generate commands ─► after success: Added/Modified→Unchanged, Deleted→Detached
```

- **Snapshot change tracking** (default): `DetectChanges` runs on `SaveChanges`, `Entries()`, `Find`, etc.
- **Change-tracking proxies** (`UseChangeTrackingProxies`) or `INotifyPropertyChanged` (`ChangeTrackingStrategy.ChangedNotifications`) avoid scanning.
- `AsNoTracking()` = no entries, no identity resolution (duplicate rows → duplicate objects). `AsNoTrackingWithIdentityResolution()` = no tracking but one instance per key within the query.
- `Attach` → Unchanged when the key is set, Added when the key is unset/default and store-generated. `Update` → Modified for all properties (Added if key unset). `Entry(e).State = …` sets the root only; graph methods (`Add/Attach/Update`) traverse reachable entities.

### 4.1 [Easy] What are the entity states and how does EF decide what to save?
**Answer**
Added, Modified, Unchanged, Deleted and Detached. On `SaveChanges` EF scans tracked entities, generates INSERT for Added, UPDATE for Modified, DELETE for Deleted, and nothing for Unchanged/Detached.

**Explanation**
For Modified entities loaded through a tracking query EF knows exactly which properties changed (snapshot compare) so the UPDATE only includes those columns. After saving, states reset to Unchanged (Deleted → Detached) and original values refresh.

**Code**
```csharp
var c = await db.Customers.FirstAsync(x => x.Id == 1);   // Unchanged
c.Name = "New";                                          // Modified after DetectChanges
Console.WriteLine(db.Entry(c).State);                    // calls DetectChanges -> Modified
await db.SaveChangesAsync();
```
```sql
UPDATE [Customers] SET [Name] = @p0 WHERE [Id] = @p1;
```

### 4.2 [Follow-up] When should you use AsNoTracking, and what is AsNoTrackingWithIdentityResolution?
**Answer**
Use `AsNoTracking` for read-only queries — less memory and faster. If the result graph contains the same entity multiple times (e.g. `Include` of a shared parent), plain `AsNoTracking` creates separate instances; `AsNoTrackingWithIdentityResolution` keeps a temporary per-query identity map so duplicates share one instance, still without tracking.

**Explanation**
You can also set the default at context level: `ChangeTracker.QueryTrackingBehavior = QueryTrackingBehavior.NoTracking`, then opt in with `.AsTracking()`. Projections to non-entity types (DTOs) are never tracked, but entities inside a projection are tracked unless AsNoTracking is used.

**Code**
```csharp
var orders = await db.Orders.AsNoTrackingWithIdentityResolution()
    .Include(o => o.Customer).ToListAsync();   // same Customer instance shared across its orders

db.ChangeTracker.QueryTrackingBehavior = QueryTrackingBehavior.NoTracking;
```

### 4.3 [Medium] Explain Attach, Update, Add and Entry().State for disconnected entities (e.g. from a Web API).
**Answer**
`Add` marks the graph Added; `Attach` marks it Unchanged (Added where keys are unset/generated); `Update` marks it Modified for *all* scalar properties (UPDATE every column) and Added for unset keys. `Entry(e).State = Modified` affects only that entity, not related entities. For partial updates, load then copy values or mark specific properties as modified.

**Explanation**
In disconnected scenarios, EF can't know what changed. `Update` overwrites every column, which can null out properties absent in the incoming DTO and breaks concurrency unless a rowversion is supplied. Safer patterns: load by id and `SetValues` / map from DTO; or `Attach` then set `IsModified = true` on individual properties.

**Code**
```csharp
// Patch just one column without loading
var stub = new Customer { Id = id, Name = dto.Name };
db.Attach(stub);
db.Entry(stub).Property(c => c.Name).IsModified = true;
await db.SaveChangesAsync();

// Load + copy values
var existing = await db.Customers.FindAsync(id);
db.Entry(existing!).CurrentValues.SetValues(dto);
await db.SaveChangesAsync();
```
```sql
UPDATE [Customers] SET [Name] = @p0 WHERE [Id] = @p1;
```

### 4.4 [Hard] How does snapshot tracking work and what is its cost? What alternatives exist?
**Answer**
On tracking a query result EF stores a snapshot of property values. `DetectChanges` iterates *all* tracked entities and compares current to snapshot (and detects relationship fixups), so cost is O(tracked entities) per call. Alternatives: change-tracking proxies/`INotifyPropertyChanged` notifications, disabling `AutoDetectChangesEnabled` for bulk loops, and using `AsNoTracking`/new short-lived contexts.

**Explanation**
`DetectChanges` is triggered by `SaveChanges`, `Entries`, `Find`, `Local`, `Add` (in older versions), etc. Adding many entities one by one in a loop with auto-detect on can become quadratic; batch via `AddRange` (one detect) or toggle `db.ChangeTracker.AutoDetectChangesEnabled = false` and call `DetectChanges()` manually. Proxies require virtual properties and the `Microsoft.EntityFrameworkCore.Proxies` package, and only entities created via `CreateProxy` or loaded by queries get them. Also `ChangeTracker.Clear()` (EF5+) detaches everything cheaply.

**Code**
```csharp
db.ChangeTracker.AutoDetectChangesEnabled = false;
foreach (var row in bigInput) db.Orders.Add(Map(row));
db.ChangeTracker.DetectChanges();
await db.SaveChangesAsync();
db.ChangeTracker.Clear();
```
**Gotchas / follow-ups**
- With auto-detect off, forgetting `DetectChanges()` before `SaveChanges` silently skips modifications.
- Each tracking query over big result sets costs memory; chunk and `Clear()`.

### 4.5 [Tricky] Two different instances with the same key get attached to one context – what happens? And what happens when you set a foreign-key navigation on a tracked entity?
**Answer**
EF throws `InvalidOperationException` ("another instance with the same key value is already being tracked"). The identity map allows one instance per key. Separately, relationship fixup: setting a navigation or FK on tracked entities is detected and EF syncs the other side (collection, FK value) at `DetectChanges`.

**Explanation**
Common source: a loaded entity plus a deserialized DTO-entity with the same Id both attached; fix by reusing the tracked instance (`Find`+`SetValues`), detaching the first, or using `AsNoTracking` for the earlier read. Another gotcha: `Update(order)` on a graph containing new child with key set (e.g. non-zero Guid generated client-side) → child treated as Modified, not Added, so the UPDATE hits zero rows → `DbUpdateConcurrencyException`. With int identity keys, key = 0 means Added; with client-generated Guids, EF can't tell, so use `Add` explicitly or `TrackGraph`.

**Code**
```csharp
var a = await db.Customers.FindAsync(1);
db.Attach(new Customer { Id = 1 });   // throws: already tracked

db.ChangeTracker.TrackGraph(order, e =>
{
    e.Entry.State = e.Entry.IsKeySet ? EntityState.Modified : EntityState.Added;
});
```

**Interview traps**
- Believing `Update()` sends only changed columns for a disconnected entity – it sends all.
- `AsNoTracking` then modifying and calling `SaveChanges` – nothing happens.
- Setting `Entry(e).State = Modified` and assuming the child collection is also saved.
- Thinking `Local` queries the database.
- Using `Update` with client-generated keys on new children.

**Cheat sheet**
- Five states; SaveChanges maps them to INSERT/UPDATE/DELETE/none.
- Tracker = identity map + snapshots; `DetectChanges` scans all entries.
- `AsNoTracking` for reads; `…WithIdentityResolution` shares duplicates.
- `Attach` = Unchanged; `Update` = all columns Modified; set `IsModified` per property for partial updates.
- `ChangeTracker.Clear()` resets a context in long loops.

---

## Topic 5: SaveChanges internals

**Study notes**

```
SaveChanges ─► DetectChanges ─► SavingChanges interceptor/event
  ─► build ModificationCommands per entry ─► topological sort by FK dependency (parents INSERT first, children DELETE first)
  ─► batch (MaxBatchSize default 42 on SQL Server; MERGE/ INSERT…OUTPUT) ─► BEGIN TRAN … COMMIT (implicit if none exists)
  ─► read back generated values (identity, computed, rowversion) ─► AcceptAllChanges ─► SavedChanges event
```

- All commands from one `SaveChanges` run in **one transaction** (atomic) unless you've started your own.
- Batching: multiple statements per round trip (`sp_executesql` with many statements); tune via `UseSqlServer(o => o.MaxBatchSize(n) / MinBatchSize)`.
- Generated values: read back with `OUTPUT INSERTED.[Id]`; tables with triggers can't use a plain `OUTPUT`, so declare them with `ToTable(t => t.HasTrigger("trg_name"))` and EF uses `OUTPUT INTO` a temp table.
- Hooks: `ISaveChangesInterceptor` (`SavingChanges`, `SavedChanges`, `SaveChangesFailed`), `override SaveChanges/SaveChangesAsync`, events `SavingChanges`/`SavedChanges`.
- Concurrency: `[Timestamp]`/`IsRowVersion` or `IsConcurrencyToken` adds `WHERE` check; mismatch → `DbUpdateConcurrencyException`.
- `SaveChanges(acceptAllChangesOnSuccess: false)` for distributed/outer transaction control.

### 5.1 [Easy] What does SaveChanges do, and is it transactional?
**Answer**
It persists all tracked changes to the database. Everything in one call is wrapped in a single transaction, so it's all-or-nothing. It returns the number of rows affected.

**Explanation**
If you didn't open a transaction, EF opens one for you (when more than one command is sent). On failure it rolls back and tracker state remains unchanged so you can fix and retry. If you need multiple `SaveChanges` calls atomic, use `Database.BeginTransaction` or `TransactionScope`.

**Code**
```csharp
db.Customers.Add(new Customer { Name = "A" });
db.Orders.Add(new Order { Customer = new Customer { Name = "B" }, Total = 5 });
int affected = await db.SaveChangesAsync();
```

### 5.2 [Follow-up] How does EF order operations, and what is batching?
**Answer**
EF builds a dependency graph from FKs, sorting so principals are inserted before dependents and dependents deleted before principals. Commands are grouped into batches sent in a single round trip, reducing latency.

**Explanation**
For SQL Server, batches are capped (default max 42 statements, configurable via `MaxBatchSize`; `MinBatchSize` controls the threshold below which statements aren't batched). Cyclic dependencies throw unless you break the cycle (insert first, then update FK in a second save). Identities generated for parents are propagated to child FKs in the same batch via parameter fix-up using values read back.

**Code**
```csharp
optionsBuilder.UseSqlServer(cs, o => o.MaxBatchSize(100));
```
```sql
-- one round trip
SET NOCOUNT ON;
INSERT INTO [Customers] ([Name]) OUTPUT INSERTED.[Id] VALUES (@p0);
INSERT INTO [Orders] ([CustomerId], [Total], ...) OUTPUT INSERTED.[Id] VALUES (@p1, @p2, ...);
```

### 5.3 [Medium] How do you implement automatic audit fields (CreatedAt/ModifiedAt/By)?
**Answer**
Use a `SaveChangesInterceptor` (preferred, reusable and DI-friendly) or override `SaveChangesAsync` and iterate `ChangeTracker.Entries<IAuditable>()`, setting timestamps by state before calling base. Don't rely on `DateTime.Now` in the database layer – use UTC and an injected clock/user provider.

**Explanation**
Interceptors register through `AddInterceptors` and run for all contexts using those options. `Entries()` calls `DetectChanges` itself, so states are current inside the interceptor. Don't set `Modified` fields on Added entries; prevent clients overwriting `CreatedAt` by marking `IsModified = false` on updates.

**Code**
```csharp
public interface IAuditable { DateTime CreatedUtc { get; set; } DateTime? ModifiedUtc { get; set; } }

public sealed class AuditInterceptor(TimeProvider clock) : SaveChangesInterceptor
{
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData e, InterceptionResult<int> result, CancellationToken ct = default)
    {
        foreach (var entry in e.Context!.ChangeTracker.Entries<IAuditable>())
        {
            var now = clock.GetUtcNow().UtcDateTime;
            if (entry.State == EntityState.Added) entry.Entity.CreatedUtc = now;
            else if (entry.State == EntityState.Modified)
            {
                entry.Property(x => x.CreatedUtc).IsModified = false;
                entry.Entity.ModifiedUtc = now;
            }
        }
        return base.SavingChangesAsync(e, result, ct);
    }
}
// registration
services.AddSingleton<AuditInterceptor>();
services.AddDbContext<AppDb>((sp, o) => o.UseSqlServer(cs).AddInterceptors(sp.GetRequiredService<AuditInterceptor>()));
```

### 5.4 [Hard] What happens on failures: DbUpdateException, concurrency conflicts, retries and execution strategies?
**Answer**
`DbUpdateException` wraps provider errors (constraint violations); `DbUpdateConcurrencyException` signals a rowversion/token mismatch (0 rows affected). With `EnableRetryOnFailure` EF uses an execution strategy that retries transient failures, but user-initiated transactions must be wrapped in `strategy.ExecuteAsync` so the whole unit retries.

**Explanation**
Concurrency handling: catch, inspect `ex.Entries`, call `GetDatabaseValues()`, then choose client-wins (refresh `OriginalValues` from DB values and retry), database-wins (`Reload`), or merge. A retrying strategy doesn't allow a manually-begun transaction outside `ExecuteAsync` ("The configured execution strategy does not support user-initiated transactions"). Retries can re-execute commits whose result was lost (ambiguous commit) – make operations idempotent.

**Code**
```csharp
var strategy = db.Database.CreateExecutionStrategy();
await strategy.ExecuteAsync(async () =>
{
    await using var tx = await db.Database.BeginTransactionAsync();
    db.Orders.Add(order);
    await db.SaveChangesAsync();
    await tx.CommitAsync();
});

try { await db.SaveChangesAsync(); }
catch (DbUpdateConcurrencyException ex)
{
    var entry = ex.Entries.Single();
    var dbVals = await entry.GetDatabaseValuesAsync();
    if (dbVals is null) { /* row deleted */ }
    else entry.OriginalValues.SetValues(dbVals);   // client wins; retry save
}
```
**Gotchas / follow-ups**
- Concurrency token declared as `[Timestamp] byte[] RowVersion` → `WHERE Id=@p AND RowVersion=@p`.
- On retry the tracker state may be partially changed; recreate work inside the delegate.

### 5.5 [Tricky] Why might SaveChanges insert in an unexpected order, or silently produce no UPDATE even though you changed something?
**Answer**
Insert order follows FK dependency, not the order of `Add` calls (within the same entity type, order generally follows add order, but not guaranteed). No UPDATE occurs if the entity isn't tracked (AsNoTracking), the value equals the snapshot, or you mutated a non-mapped/ignored property or changed an owned/JSON object in place without it being detected (e.g. mutating a mutable value-typed collection through a converter without a `ValueComparer`).

**Explanation**
Value-converted collections (e.g. `List<string>` stored as JSON/CSV) need a `ValueComparer` so EF can detect in-place mutations; otherwise changes go unnoticed. Another trap: modifying a `Detached` object. Also, triggers on tables conflict with `OUTPUT` – declare `ToTable(t => t.HasTrigger("trg"))`.

**Code**
```csharp
b.Property(e => e.Tags)
 .HasConversion(
    v => string.Join(',', v),
    v => v.Split(',', StringSplitOptions.RemoveEmptyEntries).ToList(),
    new ValueComparer<List<string>>(
        (a, c) => a!.SequenceEqual(c!),
        v => v.Aggregate(0, (h, s) => HashCode.Combine(h, s.GetHashCode())),
        v => v.ToList()));
```

**Interview traps**
- "Each entity is saved in its own transaction" – no, whole call is one.
- Calling `SaveChanges` inside loops (N round trips and transactions).
- Using a retry strategy together with manual `BeginTransaction` outside `ExecuteAsync`.
- Forgetting a `ValueComparer` for mutable converted types.
- Putting audit logic only in the controller instead of a central interceptor/override.

**Cheat sheet**
- One `SaveChanges` = one transaction, batched, FK-ordered.
- Hooks: `SaveChangesInterceptor`, override, events.
- Concurrency: rowversion → `DbUpdateConcurrencyException`; resolve via `GetDatabaseValues`.
- Retry strategies need `ExecuteAsync` around manual transactions.
- Mutable converted types need `ValueComparer`.

---

## Topic 6: Model configuration – conventions, annotations, Fluent API & keys

**Study notes**

Model building runs `OnModelCreating` after built-in **conventions**. Precedence (lowest → highest): **Convention < Data Annotation < Fluent API**. Fluent wins and is the only way to express everything (composite keys, alternate keys, indexes with options, TPT/TPC config, etc.).

| Concern | Convention | Annotation | Fluent |
|---|---|---|---|
| PK | prop `Id` / `<Type>Id` | `[Key]` | `HasKey` |
| Composite PK | – (not possible) | `[PrimaryKey(nameof(A), nameof(B))]` on class (EF7+) | `HasKey(e => new { e.A, e.B })` |
| Required | non-nullable (NRT on) | `[Required]` | `IsRequired()` |
| Length | – (nvarchar(max)) | `[MaxLength(100)]` | `HasMaxLength(100)` |
| Table/column | DbSet / property name | `[Table]`, `[Column]` | `ToTable`, `HasColumnName` |
| Index | FKs get index | `[Index]` (EF5+) | `HasIndex` |
| Alternate key | – | – | `HasAlternateKey` |

Where to put config: `IEntityTypeConfiguration<T>` classes + `modelBuilder.ApplyConfigurationsFromAssembly(...)`. Bulk conventions: `ConfigureConventions(ModelConfigurationBuilder)` (EF6+/EF Core 6) e.g. all `string` default max length, all `decimal` precision.

Keys on SQL Server: `int/long` → `IDENTITY`; `Guid` → client-generated **sequential GUID** (EF's `SequentialGuidValueGenerator`, SQL-Server-friendly ordering) when not set; or `HasDefaultValueSql("NEWSEQUENTIALID()")` for DB-generated.

### 6.1 [Easy] What is the order of precedence between conventions, data annotations and Fluent API?
**Answer**
Fluent API overrides data annotations, which override conventions. If the same facet is configured in several places the highest one wins.

**Explanation**
Conventions give defaults (e.g. property `Id` becomes PK; `string` → `nvarchar(max)`). Annotations are visible on the class (but couple the domain to EF namespaces). Fluent API is the most complete and keeps entities POCO. Teams usually standardise on Fluent in `IEntityTypeConfiguration<T>`.

**Code**
```csharp
public class Product { public int Id { get; set; } [MaxLength(50)] public string Name { get; set; } = ""; }

modelBuilder.Entity<Product>().Property(p => p.Name).HasMaxLength(200); // wins → nvarchar(200)
```
```sql
CREATE TABLE [Products] ([Id] int NOT NULL IDENTITY, [Name] nvarchar(200) NOT NULL, CONSTRAINT [PK_Products] PRIMARY KEY ([Id]));
```

### 6.2 [Follow-up] How do you configure composite keys and alternate keys? When do you need each?
**Answer**
Composite PK only via Fluent (`HasKey(e => new { e.A, e.B })`) or `[PrimaryKey]` on the class in EF7+; typical for join tables. An alternate key (`HasAlternateKey`) is a unique constraint that can be the target of FKs (a *principal key*), e.g. a business key like `Sku` or `Email`.

**Explanation**
Alternate keys also make EF treat the property as immutable once saved (can't change after the entity is tracked/saved), and enable `HasPrincipalKey` in relationships. If you only need uniqueness without being an FK target, `HasIndex(...).IsUnique()` suffices.

**Code**
```csharp
modelBuilder.Entity<OrderLine>().HasKey(l => new { l.OrderId, l.LineNo });
modelBuilder.Entity<Product>().HasAlternateKey(p => p.Sku);

modelBuilder.Entity<Review>().HasOne<Product>().WithMany().HasForeignKey(r => r.Sku).HasPrincipalKey(p => p.Sku);
```
```sql
ALTER TABLE [Products] ADD CONSTRAINT [AK_Products_Sku] UNIQUE ([Sku]);
```

### 6.3 [Medium] int identity vs Guid vs sequential Guid keys – trade-offs? What does EF do for each on SQL Server?
**Answer**
`int/long IDENTITY`: small, ordered, fast clustered index, but DB-generated (needs round trip, leaks counts, hard to merge across DBs). Random `Guid.NewGuid()` keys: client-generated, globally unique, but cause clustered-index page splits and fragmentation. Sequential GUIDs keep uniqueness with mostly-increasing order; EF generates them client-side by default for `Guid` keys on SQL Server.

**Explanation**
A clustered index on a random GUID inserts at random pages → fragmentation and larger (16 byte) keys in all non-clustered indexes. `NEWSEQUENTIALID()` is only valid as a column default (DB-side) and values are sequential only until server restart/reset; EF's client-side sequential GUID generator orders by the SQL Server GUID comparison semantics. Alternatives: UUIDv7-style or hi/lo (`UseHiLo`) for ints to pre-allocate ids and permit batching of dependent inserts without round trips, or keep int clustered key plus GUID public identifier (alternate unique index).

**Code**
```csharp
modelBuilder.Entity<Order>().Property(o => o.Id).HasDefaultValueSql("NEWSEQUENTIALID()");
modelBuilder.Entity<Customer>().Property(c => c.Id).UseHiLo("CustomerHiLo");
modelBuilder.Entity<Product>().HasKey(p => p.Id).IsClustered(false); // GUID PK non-clustered
modelBuilder.Entity<Product>().HasIndex(p => p.CreatedUtc).IsClustered();
```

### 6.4 [Hard] How do you apply conventions globally and keep config maintainable (ConfigureConventions, ApplyConfigurationsFromAssembly)? 
**Answer**
Use `IEntityTypeConfiguration<T>` per entity and `ApplyConfigurationsFromAssembly`. For cross-cutting rules (all `decimal` → `decimal(18,2)`, strings max length, enum-to-string) override `ConfigureConventions` and use `configurationBuilder.Properties<T>()`.

**Explanation**
Without explicit precision, `decimal` maps to `decimal(18,2)` on SQL Server with a model-validation warning that values may be truncated (precision warning). Setting it centrally prevents silent truncation. Pre-convention config applies as a *default* which entity-level Fluent still overrides. You can also write custom `IModelFinalizingConvention` for rules like "all FKs restrict".

**Code**
```csharp
protected override void ConfigureConventions(ModelConfigurationBuilder cb)
{
    cb.Properties<decimal>().HavePrecision(18, 4);
    cb.Properties<string>().HaveMaxLength(256);
    cb.Properties<DateTime>().HaveColumnType("datetime2");
    cb.Properties<OrderStatus>().HaveConversion<string>();
}
protected override void OnModelCreating(ModelBuilder mb) => mb.ApplyConfigurationsFromAssembly(typeof(AppDb).Assembly);

public class OrderConfig : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> b)
    {
        b.ToTable("Orders", "sales");
        b.HasIndex(o => new { o.CustomerId, o.CreatedUtc });
    }
}
```
**Gotchas / follow-ups**
- Convention/`ConfigureConventions` types apply to *all* properties of that CLR type, including those in owned types.
- Nullable reference types drive `IsRequired` convention: `string` = NOT NULL, `string?` = NULL.

### 6.5 [Tricky] Why can changing a key value on a saved entity fail, and what changes if you use an alternate key?
**Answer**
EF treats primary keys (and alternate keys) as immutable after the entity is saved/tracked: modifying them throws ("property is part of a key and so cannot be modified"). To change a key you delete and re-insert (or use raw SQL), and cascades must be considered.

**Explanation**
Keys are used for identity resolution; changing it would break the identity map and relationship fixup. Business keys that may change should not be PKs or alternate keys; use unique indexes instead. Beware also: composite key with nullable column is invalid; `HasNoKey` entities can't be tracked.

**Code**
```csharp
var p = await db.Products.FirstAsync();
p.Sku = "NEW";               // Sku is an alternate key
await db.SaveChangesAsync(); // InvalidOperationException
```

**Interview traps**
- Putting `[Required]` on non-nullable value types and expecting something – it's already required.
- Believing annotations beat Fluent API.
- Using `Guid.NewGuid()` as clustered PK without noting fragmentation.
- Forgetting decimal precision (silent truncation/warnings).
- Using alternate keys when a unique index would do.

**Cheat sheet**
- Convention < Annotation < Fluent.
- Composite key = Fluent (or class-level `[PrimaryKey]` in EF7+).
- Alternate key = unique + can be FK target.
- Guid key on SQL Server → sequential GUID by default; int → IDENTITY.
- `ConfigureConventions` for global type rules; `ApplyConfigurationsFromAssembly` for entity classes.

---

## Topic 7: Relationships & cascade delete

**Study notes**

A relationship = **principal** (has PK/alternate key) ← **dependent** (has FK). Built from: navigation properties, FK property, `HasOne/HasMany/WithOne/WithMany`, `HasForeignKey`.

```
1:N   Customer 1 ──< Order *         (FK on Order: CustomerId)
1:1   User 1 ──── 1 Profile          (FK on dependent; unique index on FK) – must say HasForeignKey<Profile>
N:N   Student *>──<* Course          (skip navigations + join table StudentCourse, no entity required)
```

- **Required** (FK non-nullable) vs **optional** (FK nullable).
- **Shadow FK**: if no FK property is declared, EF creates a shadow property `<NavName><PK>` (e.g. `CustomerId`).
- **Cascade (`DeleteBehavior`)** defaults: required → `Cascade`; optional → `ClientSetNull`.
  Values: `Cascade`, `Restrict`, `NoAction`, `SetNull`, `ClientSetNull` (default optional; DB has NO ACTION, EF nulls FK for *tracked* dependents), `ClientCascade`, `ClientNoAction`.
- SQL Server restriction: **multiple cascade paths** / cycles → error when creating FKs.
- Many-to-many (EF5+): `HasMany(...).WithMany(...)`; customise join via `UsingEntity`.

### 7.1 [Easy] How do you configure one-to-many and what's a shadow foreign key?
**Answer**
Declare `HasOne(o => o.Customer).WithMany(c => c.Orders)` and optionally `HasForeignKey(o => o.CustomerId)`. If you omit the FK property from the class EF creates a *shadow* FK column (still mapped, accessible via `Entry(e).Property("CustomerId")`).

**Explanation**
By convention EF discovers the relationship from navigations and an FK property named `<NavName>Id`/`<PrincipalType>Id`. Explicit config is clearer, especially with multiple relationships between the same types (otherwise EF throws ambiguity).

**Code**
```csharp
modelBuilder.Entity<Order>()
    .HasOne(o => o.Customer).WithMany(c => c.Orders)
    .HasForeignKey(o => o.CustomerId).IsRequired();

// shadow FK
modelBuilder.Entity<Order>().HasOne(o => o.Customer).WithMany(c => c.Orders).HasForeignKey("CustomerId");
```
```sql
ALTER TABLE [Orders] ADD CONSTRAINT [FK_Orders_Customers_CustomerId] FOREIGN KEY ([CustomerId]) REFERENCES [Customers] ([Id]) ON DELETE CASCADE;
CREATE INDEX [IX_Orders_CustomerId] ON [Orders] ([CustomerId]);
```

### 7.2 [Follow-up] How do you configure one-to-one and many-to-many?
**Answer**
1:1: `HasOne(...).WithOne(...).HasForeignKey<Dependent>(d => d.PrincipalId)` – you must say which side is dependent; EF adds a unique index on the FK. N:N: `HasMany(s => s.Courses).WithMany(c => c.Students)` creates a join table automatically; use `UsingEntity` to add payload columns or a join entity class.

**Explanation**
If the join table needs extra data (EnrolledOn, Grade), model an explicit join entity with two 1:N relationships (you can still expose skip navigations by combining `UsingEntity<Enrollment>`). Default join table name `CourseStudent` (alphabetical) with composite PK of both FKs.

**Code**
```csharp
modelBuilder.Entity<User>().HasOne(u => u.Profile).WithOne(p => p.User).HasForeignKey<Profile>(p => p.UserId);

modelBuilder.Entity<Student>().HasMany(s => s.Courses).WithMany(c => c.Students)
    .UsingEntity<Enrollment>(
        j => j.HasOne(e => e.Course).WithMany().HasForeignKey(e => e.CourseId),
        j => j.HasOne(e => e.Student).WithMany().HasForeignKey(e => e.StudentId),
        j => { j.HasKey(e => new { e.StudentId, e.CourseId }); j.Property(e => e.EnrolledOn).HasDefaultValueSql("GETUTCDATE()"); });
```

### 7.3 [Medium] Explain cascade delete behaviours and the default for required vs optional relationships.
**Answer**
Required relationships default to `Cascade` (DB `ON DELETE CASCADE`); optional ones to `ClientSetNull` (DB `NO ACTION`; EF only nulls FKs of *tracked* children, so untracked children cause an FK violation). `Restrict`/`NoAction` prevent deletion of principals with dependents; `SetNull` makes the DB null the FK.

**Explanation**
EF also does *client-side* cascade for tracked entities: deleting a tracked principal marks tracked dependents Deleted (Cascade) or nulls their FK (ClientSetNull/SetNull). Choosing `Restrict` for business-critical data avoids accidental mass deletes. Deleting orphans: `DeleteOrphansTiming` / removing a child from the parent's collection (required relationship) deletes the child at save.

**Code**
```csharp
modelBuilder.Entity<Order>().HasOne(o => o.Customer).WithMany(c => c.Orders)
    .OnDelete(DeleteBehavior.Restrict);

var c = await db.Customers.Include(x => x.Orders).FirstAsync(x => x.Id == 1);
db.Customers.Remove(c);          // tracked orders cascade (client + DB)
await db.SaveChangesAsync();
```

### 7.4 [Hard] SQL Server throws "may cause cycles or multiple cascade paths" on migration. Why, and how do you fix it?
**Answer**
SQL Server doesn't allow an FK graph where one delete could reach a table via more than one cascading path (or a cycle). Fix by setting one or more relationships to `Restrict`/`NoAction`/`ClientCascade` (so DB doesn't cascade) and handling deletion in code or by a trigger/SP.

**Explanation**
Typical: `Order` → `Customer` (cascade) and `Order` → `ShippingAddress` → `Customer` (cascade); or self-referencing hierarchies (`Employee.ManagerId`) where cascade is invalid. `ClientCascade` has EF delete tracked dependents without DB cascade, which still depends on loading dependents. Alternatives: soft delete, explicit deletes in a transaction, or `ExecuteDelete` in the right order.

**Code**
```csharp
modelBuilder.Entity<Employee>().HasOne(e => e.Manager).WithMany(m => m.Reports)
    .HasForeignKey(e => e.ManagerId).OnDelete(DeleteBehavior.Restrict);

modelBuilder.Entity<Order>().HasOne(o => o.ShipTo).WithMany().OnDelete(DeleteBehavior.NoAction);
```
**Gotchas / follow-ups**
- Optional self-reference with `ClientSetNull` default works because DB is NO ACTION.
- Required relationships in EF Core are cascade-by-default – `Restrict` must be opt-in.

### 7.5 [Tricky] Why does a 1:1 relationship misbehave when EF can't tell which side is the dependent, and what about required-dependent / optional-dependent semantics?
**Answer**
With two entity types each having a navigation to the other and no FK property, EF can't infer the dependent and throws asking you to call `HasForeignKey<T>`. Dependent side with the FK = the one that can't exist without the principal. Since EF Core 5, `Navigation(...).IsRequired()` on the dependent navigation (principal→dependent) makes the dependent required.

**Explanation**
Also note table splitting (1:1 sharing the same table) where the dependent's PK is also its FK and the dependent can be optional (`IsRequired(false)` implies all columns nullable). In 1:1 with shared PK (`Profile.Id` = `User.Id`) configure `HasKey(p => p.UserId)` and `HasForeignKey<Profile>(p => p.UserId)`. Querying principals without including the dependent leaves the navigation null – not "no row".

**Code**
```csharp
modelBuilder.Entity<User>().HasOne(u => u.Profile).WithOne(p => p.User)
    .HasForeignKey<Profile>(p => p.UserId);
modelBuilder.Entity<User>().Navigation(u => u.Profile).IsRequired();   // EF5+: principal requires dependent
modelBuilder.Entity<Profile>().HasKey(p => p.UserId);                  // shared-key 1:1
```

**Interview traps**
- Assuming `Cascade` is the default for optional relationships (it is `ClientSetNull`).
- Believing deleting an untracked principal null-ifies dependents with ClientSetNull – DB will reject it.
- Multiple relationships between same two types without explicit config → ambiguity.
- Thinking many-to-many needs an explicit join class (not since EF5).
- Forgetting to configure `OnDelete` explicitly, then being surprised by mass cascade deletes on required relationships.

**Cheat sheet**
- Principal ← dependent(FK); required → cascade, optional → ClientSetNull.
- 1:1 needs `HasForeignKey<TDependent>`.
- N:N via skip navigations; `UsingEntity` for payload.
- SQL Server multiple cascade paths → switch to Restrict/NoAction/ClientCascade.
- No FK property = shadow FK `<Nav><Key>`.

---

## Topic 8: Owned types, complex types & inheritance mapping

**Study notes**

**Owned entity types** (`OwnsOne/OwnsMany`) model value-object-like parts of an aggregate. They have a hidden key (owner FK, plus shadow key for `OwnsMany`), are always loaded with the owner (no `Include` needed), and are tracked as part of the owner.
- `OwnsOne` → columns in the owner's table (**table splitting**), named `Address_Street`.
- `OwnsMany` → separate table with a composite key (owner id + shadow id).
- Optional dependent: if all owned columns are NULL, the navigation is null.

**Complex types** (EF Core 8): `[ComplexType]` / `ComplexProperty(...)`. No identity/key, **value semantics** (copy on assign, compare by value), not tracked as separate entities, mapped to columns in the owner's table (EF8 limits: no collections, can't be null, and `ExecuteUpdate` can't target them as a whole – set it via normal tracking).

**Inheritance**
```
TPH (default)  one table: Animals(Id, Name, Discriminator, Bark?, Meow?)     fastest, nullable columns
TPT            Animals(Id,Name) ← Dogs(Id,Bark) / Cats(Id,Meow)               JOINs, normalized, slower
TPC (EF7+)     Dogs(Id,Name,Bark) / Cats(Id,Name,Meow), no base table         no joins for leaf queries, UNION for base
```
TPC can't use database `IDENTITY` across tables (keys must be unique across hierarchy) → use sequences/`HiLo`/Guids (EF defaults to a sequence for ints).

### 8.1 [Easy] What is an owned entity and how do you configure it?
**Answer**
An owned type is a dependent entity whose lifecycle is tied to its owner and which has no independent identity (e.g. `Address` of `Customer`). Configure with `OwnsOne`/`OwnsMany`; by default `OwnsOne` shares the owner's table.

**Explanation**
Good for value objects in DDD (Money, Address). The owned instance can't be queried via its own `DbSet` (use `Set<T>()` only in limited cases) and isn't shared between owners. Because it lives in the owner's table, there's no join.

**Code**
```csharp
public record Address(string Street, string City, string Zip);
public class Customer { public int Id { get; set; } public Address? ShipTo { get; set; } }

modelBuilder.Entity<Customer>().OwnsOne(c => c.ShipTo, a =>
{
    a.Property(x => x.Street).HasMaxLength(100).HasColumnName("ShipStreet");
});
```
```sql
CREATE TABLE [Customers] ([Id] int IDENTITY, [ShipTo_Street] nvarchar(100) NULL, [ShipTo_City] nvarchar(max) NULL, [ShipTo_Zip] nvarchar(max) NULL);
```

### 8.2 [Follow-up] Owned types vs complex types (EF8) – differences?
**Answer**
Owned types are entity types (hidden key, reference semantics, tracked, can be `OwnsMany`, can be null); complex types are keyless value types with value semantics (assigning copies; compare by value), simpler and cheaper, but in EF8 non-nullable and no collections.

**Explanation**
Complex types avoid the identity-related surprises of owned types: sharing one owned instance between two owners throws; with a complex type each owner gets its own copy. Use `record`/`struct` for natural value equality. Prefer complex types for pure value objects in EF8; keep owned for optional or collection scenarios.

**Code**
```csharp
[ComplexType] public record struct Money(decimal Amount, string Currency);

public class Invoice { public int Id { get; set; } public Money Total { get; set; } }
// or fluent:
modelBuilder.Entity<Invoice>().ComplexProperty(i => i.Total, p => p.Property(m => m.Currency).HasMaxLength(3));
var big = db.Invoices.Where(i => i.Total.Amount > 100);   // translates to column Total_Amount
```

### 8.3 [Medium] Compare TPH, TPT and TPC. Which is the default and when would you pick each?
**Answer**
TPH (default): one table with a discriminator; fastest queries, nullable columns for subtype properties. TPT: one table per type joined by PK; normalized but slower (joins). TPC (EF7+): one table per concrete type with all columns; fast for leaf queries, UNION for base queries. Pick TPH by default; TPT when strict schema/NOT NULL integrity matters; TPC when subtypes are mostly queried alone.

**Explanation**
Microsoft's own guidance recommends TPH unless required otherwise for performance. TPT's joins hurt; TPC with abstract base avoids a base table; polymorphic queries `UNION ALL` the tables. Relationships to the *base* type under TPC are problematic (FK can't target multiple tables).

**Code**
```csharp
modelBuilder.Entity<Animal>()
    .HasDiscriminator<string>("Kind").HasValue<Dog>("dog").HasValue<Cat>("cat");   // TPH

modelBuilder.Entity<Animal>().UseTptMappingStrategy();   // TPT
modelBuilder.Entity<Animal>().UseTpcMappingStrategy();   // TPC
var dogs = await db.Animals.OfType<Dog>().ToListAsync();
```
```sql
SELECT [a].[Id], [a].[Name], [a].[Bark] FROM [Animals] AS [a] WHERE [a].[Kind] = N'dog'
```

### 8.4 [Hard] How do owned collections (OwnsMany) behave, and what are common pitfalls?
**Answer**
`OwnsMany` creates a separate table keyed by (owner id + shadow int id). It's loaded automatically with the owner. Changing the collection by replacing items often results in delete + insert rather than update because items lack a stable identity unless you give them a key.

**Explanation**
Pitfalls: every query on the owner joins/loads the collection (can't opt out except projection or EF8 `ToJson`). Removing and re-adding "equal" items causes churn. In disconnected scenarios, if you `Update` a graph, EF may treat owned items as new. Consider making it a regular entity if you need to query it independently, or use JSON column mapping (`ToJson`) to store it inside one column.

**Code**
```csharp
modelBuilder.Entity<Order>().OwnsMany(o => o.Notes, n =>
{
    n.WithOwner().HasForeignKey("OrderId");
    n.Property<int>("Id");
    n.HasKey("Id");
    n.ToTable("OrderNotes");
});
```
**Gotchas / follow-ups**
- Owned types are loaded implicitly – no `Include` needed (or possible to opt out except via projection).
- Same owned CLR type used by two owners creates separate mapped types per owner.

### 8.5 [Tricky] TPH discriminator pitfalls: adding a subtype, querying with a base type, and nullable columns?
**Answer**
Adding a subtype adds nullable columns and a discriminator value via migration – safe but widens the table. Subtype properties must be nullable in the table (even if non-null in C#), so DB constraints are weaker. Querying `db.Animals` returns instances materialized per discriminator; unknown discriminator values throw at materialization.

**Explanation**
Sparse columns grow with hierarchy; shared column names between subtypes with the same type/facets are merged (`HasColumnName`). Use a check constraint to enforce subtype-specific NOT NULL rules. The discriminator can be mapped as a property to filter/seed (`HasDiscriminator(...).IsComplete(false)` allows unmapped values). Changing the concrete type of an existing row isn't possible through EF – you'd update the discriminator by SQL.

**Code**
```csharp
modelBuilder.Entity<Dog>().ToTable(t => t.HasCheckConstraint("CK_Dog_Bark", "[Kind] <> 'dog' OR [Bark] IS NOT NULL"));
```

**Interview traps**
- Thinking owned types need `Include`.
- Sharing one owned instance across owners.
- Choosing TPT "because it's cleaner" without considering join cost.
- Expecting TPC to support IDENTITY keys for the whole hierarchy.
- Treating complex types (EF8) as nullable or collections – not supported in EF8.

**Cheat sheet**
- Owned = entity-without-identity, aggregate part; OwnsOne → same table.
- Complex type (EF8) = value semantics, keyless, no collections.
- TPH default/fast; TPT joins; TPC (EF7+) no base table.
- TPC → no shared IDENTITY; use sequences/Guid.
- Owned collections → separate table (or `ToJson`).

---

## Topic 9: Value converters, JSON columns, shadow/backing fields, computed, defaults, sequences

**Study notes**

- **Value converter**: maps CLR ↔ provider type (`HasConversion`). Built-ins: enum↔string/int, bool↔int, Guid↔string, DateTime↔long, etc. Needs a **ValueComparer** for mutable reference types (arrays, lists).
- **Enums**: stored as int by default; `HasConversion<string>()` stores names (readable, safer on reorder, wider/slower index).
- **JSON columns** (EF7+/8): `OwnsOne(...).ToJson()` / `OwnsMany(...).ToJson()` → `nvarchar(max)`; queries translate to `JSON_VALUE`/`OPENJSON`; EF8 adds primitive collections.
- **Shadow properties**: exist in the model/DB but not on the class; access with `Entry(e).Property("X")` / `EF.Property<T>(e, "X")`.
- **Backing fields**: EF writes straight to fields (convention `_name`, `m_name`, `name`), good for encapsulation; `UsePropertyAccessMode`.
- **Computed columns**: `HasComputedColumnSql("...", stored: true)`. **Defaults**: `HasDefaultValue` / `HasDefaultValueSql`. **Sequences**: `HasSequence`, `NEXT VALUE FOR`.

### 9.1 [Easy] How do you store an enum as a string?
**Answer**
`HasConversion<string>()` on the property (or globally via `ConfigureConventions`). EF converts on write and read.

**Explanation**
Int storage is compact but renumbering the enum silently corrupts data; strings are self-describing but longer. Add a `HasMaxLength` and consider a check constraint or lookup table for integrity. Queries on the enum translate comparing strings.

**Code**
```csharp
modelBuilder.Entity<Order>().Property(o => o.Status).HasConversion<string>().HasMaxLength(20);
var open = await db.Orders.Where(o => o.Status == OrderStatus.Open).ToListAsync();
```
```sql
SELECT ... FROM [Orders] AS [o] WHERE [o].[Status] = N'Open'
```

### 9.2 [Follow-up] Write a custom value converter (e.g. strongly-typed ID or encrypted string). What are its limitations?
**Answer**
Subclass `ValueConverter<TModel,TProvider>` or use `HasConversion(toProvider, fromProvider)`. Limitations: cannot convert `null`s (converters don't see null by default), SQL translation of operations uses the provider type (no server-side functions on the converted value), and mutable types need a comparer.

**Explanation**
Strongly-typed IDs (`OrderId` record struct wrapping `Guid`) are a common use. Converted properties can be compared in LINQ (`o.Id == someOrderId`) since EF converts the parameter, but `o.Id.Value.ToString()` won't translate. Encrypted converters make `Contains/LIKE` searching impossible.

**Code**
```csharp
public readonly record struct OrderId(Guid Value);

public class OrderIdConverter : ValueConverter<OrderId, Guid>
{
    public OrderIdConverter() : base(id => id.Value, g => new OrderId(g)) { }
}
protected override void ConfigureConventions(ModelConfigurationBuilder cb) =>
    cb.Properties<OrderId>().HaveConversion<OrderIdConverter>();
```

### 9.3 [Medium] How do JSON columns work in EF Core 8 and when would you use them?
**Answer**
Map an owned type (or collection) with `ToJson()` and EF stores it as a JSON document in one `nvarchar(max)` column, materializing it with the owner and translating LINQ over its properties into `JSON_VALUE`/`OPENJSON`. Use for semi-structured, aggregate-local data that doesn't need relational joins/constraints.

**Explanation**
Benefits: no extra table/join, flexible schema. Drawbacks: weaker integrity, harder reporting, indexing needs computed columns on `JSON_VALUE`. EF8 supports `ToJson` on owned types, including nested and collections, and change tracking of individual JSON parts (updates via `JSON_MODIFY`). `ExecuteUpdate` can't set JSON properties in EF8.

**Code**
```csharp
public class Contact { public int Id { get; set; } public ContactDetails Details { get; set; } = new(); }
public class ContactDetails { public string Phone { get; set; } = ""; public List<Address> Addresses { get; set; } = new(); }

modelBuilder.Entity<Contact>().OwnsOne(c => c.Details, d =>
{
    d.ToJson();
    d.OwnsMany(x => x.Addresses);
});
var q = db.Contacts.Where(c => c.Details.Phone == "123");
```
```sql
SELECT ... FROM [Contacts] AS [c] WHERE JSON_VALUE([c].[Details], '$.Phone') = N'123'
```

### 9.4 [Hard] Computed columns, default values and generated-value semantics: what does EF do on insert/update, and what is the bool/enum "sentinel" gotcha?
**Answer**
`HasDefaultValue/Sql` → `ValueGeneratedOnAdd`: if the CLR value equals the type's default (sentinel; `0`, `false`, `null`) EF omits the column on INSERT so the DB default applies; otherwise sends your value. Computed columns are `ValueGenerated.OnAddOrUpdate` and read back after save; EF never writes them. The gotcha: a `bool` with default `true` – explicitly setting `false` equals the sentinel and gets replaced by `true`.

**Explanation**
Fixes: make the property nullable (`bool?`), use a backing-field-based nullable, or specify a different sentinel (`HasSentinel(true)` in EF8, so `true` = "not set" and `false` is sent). For computed columns, `stored: true` persists values (index-friendly). For `rowversion`, `IsRowVersion()` generated on add/update.

**Code**
```csharp
b.Property(o => o.IsActive).HasDefaultValue(true).HasSentinel(true);            // EF8 sentinel
b.Property(o => o.CreatedUtc).HasDefaultValueSql("SYSUTCDATETIME()");
b.Property(o => o.Total).HasComputedColumnSql("[Subtotal] + [Tax]", stored: true);
b.Property(o => o.RowVersion).IsRowVersion();
```
```sql
[CreatedUtc] datetime2 NOT NULL DEFAULT (SYSUTCDATETIME()),
[Total] AS ([Subtotal] + [Tax]) PERSISTED
```
**Gotchas / follow-ups**
- Computed columns are read back, never written; don't expect assigned values to persist.
- A trigger-maintained column needs `ToTable(t => t.HasTrigger(...))` so EF avoids plain `OUTPUT`.

### 9.5 [Tricky] Shadow properties, backing fields and sequences – how do you use each, and what traps exist?
**Answer**
Shadow props hold infra data (e.g. `LastModified`, tenant id) without polluting the entity: `Property<DateTime>("LastModified")`, query with `EF.Property<DateTime>(e, "LastModified")`. Backing fields let EF bypass setters to preserve invariants. Sequences give DB-generated numbers shared across tables: `HasSequence` + `HasDefaultValueSql("NEXT VALUE FOR …")`.

**Explanation**
Shadow properties live only in the tracker/DB: after a no-tracking query they're unavailable except through projection with `EF.Property`. Backing-field conventions require matching names; with read-only collection exposure (`IReadOnlyCollection<T> Items => _items`) EF populates `_items`. Sequences: values can have gaps (rollbacks); `NEXT VALUE FOR` in default requires `ValueGeneratedOnAdd`. `UseHiLo` is sequence-based block allocation reducing round trips.

**Code**
```csharp
modelBuilder.Entity<Order>().Property<DateTime>("LastModified");
modelBuilder.HasSequence<int>("OrderNumbers", s => s.StartsAt(1000).IncrementsBy(1));
modelBuilder.Entity<Order>().Property(o => o.Number).HasDefaultValueSql("NEXT VALUE FOR OrderNumbers");

public class Basket { private readonly List<Item> _items = new(); public IReadOnlyCollection<Item> Items => _items; }
modelBuilder.Entity<Basket>().Navigation(b => b.Items).UsePropertyAccessMode(PropertyAccessMode.Field);

var recent = db.Orders.Where(o => EF.Property<DateTime>(o, "LastModified") > cutoff);
```

**Interview traps**
- Enum as int + later reorder = corrupted data.
- Forgetting `ValueComparer` on list/array converters.
- `bool` with DB default `true` can't be saved as `false` (sentinel).
- Expecting JSON columns to have indexes/constraints automatically.
- Sequence values are not gap-free.

**Cheat sheet**
- `HasConversion<string>()` for enum names.
- Converters: no nulls by default, need comparers for mutables, limits SQL translation.
- `ToJson()` owned types → `JSON_VALUE/OPENJSON`.
- Defaults skip insert when value == sentinel; `HasSentinel` (EF8) fixes bool defaults.
- Shadow: `EF.Property<T>`; backing fields: `UsePropertyAccessMode`.

---

## Topic 10: Global query filters, table splitting, keyless entities, views & stored procedures

**Study notes**

- **Global query filter**: `HasQueryFilter(e => !e.IsDeleted)` is appended as a `WHERE` to every query of that entity (and navigations/`Include`s). EF Core 8: **one filter per entity type** (named filters arrive in EF10). Bypass with `IgnoreQueryFilters()`.
- **Table splitting**: several entity types → same table (1:1 with shared PK) so you can load a "light" and "heavy" slice.
- **Keyless entities**: `HasNoKey()`; never tracked; used for views/raw SQL results/aggregates. `ToView("vw_X")` → table not created by migrations (create the view yourself with `migrationBuilder.Sql`).
- **Raw SQL**: `FromSql/FromSqlInterpolated` (entity or keyless type), `Database.SqlQuery<T>` (EF8, any scalar/unmapped type), `Database.ExecuteSql`. Stored-proc results can't be composed with `Include`/server-side LINQ (non-composable) – wrap `AsEnumerable()` or use `SELECT * FROM (...)`-style inline TVF. EF7+ also supports mapping insert/update/delete to sprocs (`InsertUsingStoredProcedure`).

### 10.1 [Easy] What is a global query filter and where is it typically used?
**Answer**
A model-level predicate that EF automatically applies to all queries for an entity. Typical uses: soft delete (`IsDeleted == false`) and multi-tenancy (`TenantId == current`).

**Explanation**
It is applied in the query pipeline, also for related entities reached via navigation or `Include`, and when you compose LINQ over `FromSql`. It does *not* apply to `ExecuteSql`/pure raw SQL you write yourself, and `IgnoreQueryFilters()` removes it. `ExecuteDelete/ExecuteUpdate` do include the filter in their `WHERE`.

**Code**
```csharp
modelBuilder.Entity<Customer>().HasQueryFilter(c => !c.IsDeleted);
var all = await db.Customers.IgnoreQueryFilters().ToListAsync();
```
```sql
SELECT ... FROM [Customers] AS [c] WHERE [c].[IsDeleted] = CAST(0 AS bit)
```

### 10.2 [Follow-up] How do you implement multi-tenancy with a query filter, safely?
**Answer**
Expose a `TenantId` property/field on the DbContext (populated from an injected tenant provider) and reference it in the filter: `HasQueryFilter(o => o.TenantId == TenantId)`. EF treats it as a parameter evaluated per execution against the *context instance*, so each request sees its own tenant.

**Explanation**
Also enforce tenant on writes (set `TenantId` for Added entities in `SaveChanges` interceptor) – filters only restrict reads. For strong isolation consider separate schemas/DBs or SQL Server Row-Level Security (`SESSION_CONTEXT`). Pooled contexts require resetting tenant per lease – use a factory/`IDbContextFactory` with a tenant-aware wrapper, or avoid pooling when state is per request.

**Code**
```csharp
public class AppDb : DbContext
{
    private readonly ITenantProvider _tenant;
    public AppDb(DbContextOptions<AppDb> o, ITenantProvider tenant) : base(o) => _tenant = tenant;
    public Guid TenantId => _tenant.TenantId;

    protected override void OnModelCreating(ModelBuilder mb) =>
        mb.Entity<Order>().HasQueryFilter(o => o.TenantId == TenantId);
}
```

### 10.3 [Medium] Implement soft delete end-to-end. What are the pitfalls?
**Answer**
Add `IsDeleted` (and `DeletedUtc`), a query filter excluding deleted rows, and an interceptor/`SaveChanges` override that converts `Deleted` entries into `Modified` with `IsDeleted = true`. Pitfalls: unique indexes (need a filtered index), cascades, required navigations to filtered entities, and bulk `ExecuteDelete` which does a real delete.

**Explanation**
A required navigation to a soft-deleted principal returns no row for the dependent (EF warns: "required navigation… global query filter"). Mirror filters on dependents or make the relationship optional. Unique constraints must ignore deleted rows: `HasIndex(...).IsUnique().HasFilter("[IsDeleted] = 0")`. Cascade soft delete isn't automatic – DB cascades only fire on hard deletes.

**Code**
```csharp
public override async ValueTask<InterceptionResult<int>> SavingChangesAsync(DbContextEventData e, InterceptionResult<int> r, CancellationToken ct = default)
{
    foreach (var entry in e.Context!.ChangeTracker.Entries<ISoftDelete>().Where(x => x.State == EntityState.Deleted))
    {
        entry.State = EntityState.Modified;
        entry.Entity.IsDeleted = true;
        entry.Entity.DeletedUtc = DateTime.UtcNow;
    }
    return await base.SavingChangesAsync(e, r, ct);
}
modelBuilder.Entity<Customer>().HasIndex(c => c.Email).IsUnique().HasFilter("[IsDeleted] = 0");
```
```sql
CREATE UNIQUE INDEX [IX_Customers_Email] ON [Customers] ([Email]) WHERE [IsDeleted] = 0;
```

### 10.4 [Hard] Keyless entities, views and stored procedures – how do you map and query each, and what are the limits?
**Answer**
Map a view with `HasNoKey().ToView("vw_OrderTotals")` (or give it a key if it has one); query like any `DbSet`. Stored procs: `db.Set<T>().FromSql($"EXEC GetOrders {id}")` for entity/keyless results, `Database.SqlQuery<T>` for ad hoc types (EF8), `ExecuteSql` for non-queries. Sproc results are non-composable (no `Include`, and further LINQ operators run client-side).

**Explanation**
Keyless types are never tracked and don't participate in `SaveChanges`. `ToView` keeps migrations from creating a table (the view must be created manually via `migrationBuilder.Sql`). A mapped entity can use `ToTable` for writes in one model and views for reads via separate keyless types (CQRS style). For TVFs: `HasDbFunction` + `FromExpression` gives composable queries. Always use interpolated `FromSql` (parameterised); `FromSqlRaw` with concatenation = SQL injection.

**Code**
```csharp
public class OrderTotal { public int CustomerId { get; set; } public decimal Sum { get; set; } }
modelBuilder.Entity<OrderTotal>().HasNoKey().ToView("vw_OrderTotals");

var rows = await db.Set<OrderTotal>().Where(v => v.Sum > 1000).ToListAsync();      // composes on a view
var viaSp = await db.Orders.FromSql($"EXEC dbo.GetOrdersByCustomer {customerId}").ToListAsync();
var scalars = await db.Database.SqlQuery<int>($"SELECT Id FROM Orders WHERE Total > {min}").ToListAsync();
```
**Gotchas / follow-ups**
- `SqlQuery<T>` results are composable only when the SQL is a SELECT (EF wraps it in a subquery); `EXEC` isn't composable.
- Interpolated `FromSqlRaw($"...")` (with `$` but *Raw*) bypasses parameterisation – use `FromSql` / `FromSqlInterpolated`.

### 10.5 [Tricky] How does table splitting work, and why would you use it?
**Answer**
Map two entity types to the same table with a shared PK and a 1:1 relationship, e.g. `Product` and `ProductDetails` (large text/blob). You can query the lightweight entity without pulling heavy columns and load the details on demand.

**Explanation**
The dependent must share the PK as FK; if the dependent is optional all its non-key columns must be nullable. Both entities are tracked separately and update the same row, so concurrency tokens should be configured on the shared table consistently. It's an alternative to projection when you want entity semantics for the heavy part, or for different bounded contexts' views of one table.

**Code**
```csharp
modelBuilder.Entity<Product>().ToTable("Products");
modelBuilder.Entity<ProductDetails>().ToTable("Products");
modelBuilder.Entity<Product>().HasOne(p => p.Details).WithOne().HasForeignKey<ProductDetails>(d => d.Id);
```

**Interview traps**
- Assuming a query filter protects raw `ExecuteSql`, `IgnoreQueryFilters()` calls, or writes – it only restricts LINQ reads.
- Filters protecting reads only, with writes unguarded.
- `ExecuteDelete` on soft-delete entities hard-deletes.
- Calling `FromSqlRaw` with string interpolation/concatenation.
- Trying to `Include` after `FromSql("EXEC ...")`.

**Cheat sheet**
- `HasQueryFilter` (one per entity in EF8) + `IgnoreQueryFilters()`.
- Multi-tenant filter references a context member → per-instance parameter.
- Keyless: `HasNoKey().ToView()`; untracked.
- `FromSql` (interpolated, safe) / `SqlQuery<T>` (EF8) / `ExecuteSql`.
- Table splitting = many entities, one table, shared PK.

---

## Topic 11: Migrations

**Study notes**

```
Model (C#)  ──diff vs──  ModelSnapshot.cs  ──►  Add-Migration  ──►  YYYYMMDDHHMMSS_Name.cs (Up/Down) + Designer.cs
                                                                   │
dotnet ef database update / Migrate() / script / bundle ──► runs Up() for migrations not in [__EFMigrationsHistory]
```

- `__EFMigrationsHistory(MigrationId, ProductVersion)` tracks applied migrations.
- Commands: `dotnet ef migrations add X`, `migrations remove` (last unapplied), `database update [Target]` (use `0` to revert all), `migrations script [From] [To] --idempotent`, `migrations bundle` (self-contained `efbundle.exe`), `migrations list`.
- **Snapshot** = model state at the last migration; the next `add` diffs against it, *not* against the DB.
- Custom SQL: `migrationBuilder.Sql(...)`. Data fixes inside migrations are allowed; keep them deterministic.
- Production: generate idempotent scripts or bundles in CI/CD, review SQL, apply with a privileged identity; avoid `Database.Migrate()` at app start for multi-instance apps.

### 11.1 [Easy] Describe the migrations workflow.
**Answer**
Change the model, run `dotnet ef migrations add <Name>` to generate `Up`/`Down` code and an updated snapshot, then `dotnet ef database update` to apply pending migrations. EF records applied ones in `__EFMigrationsHistory`.

**Explanation**
The generated migration is code you should review and commit alongside the model changes. `Up` moves schema forward, `Down` reverts. The design-time tooling (`Microsoft.EntityFrameworkCore.Design`) builds the context via `IDesignTimeDbContextFactory` or host builder.

**Code**
```bash
dotnet ef migrations add AddOrderStatus --project Data --startup-project Api
dotnet ef database update --project Data --startup-project Api
```
```csharp
public partial class AddOrderStatus : Migration
{
    protected override void Up(MigrationBuilder mb) => mb.AddColumn<string>("Status", "Orders", "nvarchar(20)", nullable: false, defaultValue: "Open");
    protected override void Down(MigrationBuilder mb) => mb.DropColumn("Status", "Orders");
}
```

### 11.2 [Follow-up] How do you undo a migration, and what is the difference between `migrations remove` and `database update <Previous>`?
**Answer**
If not applied: `dotnet ef migrations remove` deletes the last migration and reverts the snapshot. If applied: first `dotnet ef database update <PreviousMigration>` (runs `Down`), then `migrations remove`. To revert everything: `database update 0`.

**Explanation**
`remove` refuses (without `--force`) to delete an applied migration to protect consistency. In production, don't run `Down` casually – data in dropped columns is gone; prefer a forward-fix migration. Also never hand-edit old, already-applied migrations.

**Code**
```bash
dotnet ef database update AddCustomers      # rolls back migrations after AddCustomers
dotnet ef migrations remove
```

### 11.3 [Medium] How do you deploy migrations to production? Compare Migrate(), SQL scripts and bundles.
**Answer**
Best practice: generate an **idempotent SQL script** (`migrations script --idempotent`) or an **efbundle** in the pipeline, review it, and run it as a deployment step with elevated permissions. `context.Database.Migrate()` at startup is convenient but risky: concurrent app instances, app needs DDL rights, long migrations block startup.

**Explanation**
Idempotent scripts check `__EFMigrationsHistory` per migration, so they work from any starting state. Bundles (`dotnet ef migrations bundle --self-contained`) produce an executable taking `--connection`, ideal for containerised CD without the SDK. `Migrate()` in EF8 has no distributed lock (added later), so two instances may race.

**Code**
```bash
dotnet ef migrations script --idempotent -o deploy.sql
dotnet ef migrations bundle --self-contained -r linux-x64 -o efbundle
./efbundle --connection "Server=...;Database=App;..."
```
```sql
IF NOT EXISTS (SELECT * FROM [__EFMigrationsHistory] WHERE [MigrationId] = N'20240101_AddOrderStatus')
BEGIN
    ALTER TABLE [Orders] ADD [Status] nvarchar(20) NOT NULL DEFAULT N'Open';
    INSERT INTO [__EFMigrationsHistory] ([MigrationId],[ProductVersion]) VALUES (N'20240101_AddOrderStatus', N'8.0.0');
END;
```

### 11.4 [Hard] How do you do zero-downtime schema changes (expand/contract) with EF migrations?
**Answer**
Split breaking changes across releases: **expand** (add new nullable column/table, deploy code that writes both/reads old), **backfill/migrate** data, switch reads, then **contract** (drop old column) in a later release. Each migration must be backward compatible with the currently running app version.

**Explanation**
Example rename `Name` → `FullName`: release 1 adds `FullName`, app writes both (trigger or code) and backfills; release 2 reads `FullName`; release 3 drops `Name`. EF can scaffold a rename as drop+add if it can't detect it – manually edit to `RenameColumn` to avoid data loss (though rename itself breaks old app versions). Large backfills run in batches outside the DDL transaction. Avoid long locks: add columns nullable/with defaults, create indexes `ONLINE = ON` (Enterprise/Azure SQL) via `migrationBuilder.Sql`/`IsCreatedOnline` (EF Core `.IsCreatedOnline()` for SQL Server).

**Code**
```csharp
protected override void Up(MigrationBuilder mb)
{
    mb.AddColumn<string>("FullName", "Customers", nullable: true);
    mb.Sql("UPDATE Customers SET FullName = Name WHERE FullName IS NULL");   // for big tables batch this
}
// index online
modelBuilder.Entity<Order>().HasIndex(o => o.CreatedUtc).IsCreatedOnline();
```
**Gotchas / follow-ups**
- Non-nullable column add without default fails on non-empty tables.
- Scaffolded drop+add column on rename = data loss: always read the generated code.
- Old app instances running against new schema is the real constraint.

### 11.5 [Tricky] Team conflicts, squashing and snapshot problems – how do you handle them?
**Answer**
Two branches each adding a migration both change `ModelSnapshot.cs`, causing merge conflicts and an inconsistent snapshot. Resolve by merging the snapshot to represent the union, or delete your branch's migration (and its Designer file), merge main, then re-run `migrations add` so it is diffed against the merged snapshot. Squashing: remove old migrations + snapshot and add a fresh `Initial`, then mark it applied in existing databases.

**Explanation**
Never edit the DB and the snapshot independently; if the snapshot drifts from the model, the next `add` generates unexpected operations (review every scaffolded migration; EF9 adds `dotnet ef migrations has-pending-model-changes` / `Database.HasPendingModelChanges()` for CI checks – on EF8 add a migration in CI and assert it's empty). For squashing on existing DBs, insert the new initial migration's id into `__EFMigrationsHistory` manually (and delete old rows) or generate script excluding create-table steps. Keep migrations in order of timestamp; renaming migrations breaks history matching.

**Code**
```bash
dotnet ef migrations remove            # on your branch before merge
git merge origin/main
dotnet ef migrations add MyChange      # regenerated against merged snapshot
```
```sql
-- after squashing, for an existing DB
DELETE FROM __EFMigrationsHistory;
INSERT INTO __EFMigrationsHistory (MigrationId, ProductVersion) VALUES (N'20250101000000_Initial', N'8.0.0');
```

**Interview traps**
- Running `Migrate()` on every instance at startup.
- Editing old applied migrations.
- Trusting the scaffolder on renames.
- Thinking `migrations add` diffs against the live database (it diffs against the snapshot).
- Resolving snapshot merge conflicts by "accept mine".

**Cheat sheet**
- Add → review → update; history in `__EFMigrationsHistory`; diff vs snapshot.
- Undo: `database update <prev>` then `migrations remove`.
- Prod: idempotent script or bundle, not `Migrate()` on startup.
- Zero-downtime = expand → backfill → contract across releases.
- Conflicts: drop your migration, merge, re-add; CI: assert no pending model changes.

---

## Topic 12: Seeding & Code-first vs Database-first

**Study notes**

**Seeding options**
| Approach | Where | Notes |
|---|---|---|
| `HasData` (model seeding) | `OnModelCreating` | Becomes `InsertData/UpdateData/DeleteData` in migrations; needs explicit PKs; no navigations (use FK values); best for small static reference data |
| `migrationBuilder.Sql/InsertData` | migration `Up` | Full control, one-off data fixes |
| Startup seeder | app code after `Migrate()` | Idempotent checks; dynamic/env-specific data; EF9 adds `UseSeeding/UseAsyncSeeding` |

**Code-first**: model classes are the source of truth → migrations produce the DB. **Database-first**: `dotnet ef dbcontext scaffold` reverse-engineers an existing DB into entity classes + context. There is no EDMX designer in EF Core.

### 12.1 [Easy] How do you seed reference data with HasData?
**Answer**
Call `HasData` on the entity builder with objects that include their keys. The next migration contains the insert statements; changes to seed values produce updates/deletes in later migrations.

**Explanation**
EF compares seed data in the model snapshot, so seed rows are tracked by key. Use it for lookup tables (countries, statuses). For related data, set FK properties (not navigations). Don't use non-deterministic values (`DateTime.Now`, `Guid.NewGuid()`) – each migration would detect changes.

**Code**
```csharp
modelBuilder.Entity<OrderStatus>().HasData(
    new OrderStatus { Id = 1, Name = "Open" },
    new OrderStatus { Id = 2, Name = "Shipped" });
```
```sql
INSERT INTO [OrderStatuses] ([Id],[Name]) VALUES (1, N'Open'), (2, N'Shipped');
```

### 12.2 [Follow-up] When is HasData the wrong choice?
**Answer**
For large datasets, environment-specific data, data with generated/non-deterministic values, data requiring logic or hashing (admin users with passwords), or when you need to update rows users may have edited. Use a startup seeder or custom migration SQL instead.

**Explanation**
HasData forces overwriting: if users changed a seeded row, a later migration changing the seed can overwrite it, and removing seed deletes rows (FK failures). It also bloats the snapshot. With identity PKs, HasData inserts explicit key values (`SET IDENTITY_INSERT` handled by migrations), so later app-created rows must not collide.

**Code**
```csharp
public static async Task SeedAsync(AppDb db)
{
    if (!await db.Roles.AnyAsync(r => r.Name == "Admin"))
    {
        db.Roles.Add(new Role { Name = "Admin" });
        await db.SaveChangesAsync();
    }
}
// startup (single-instance/dev): await db.Database.MigrateAsync(); await SeedAsync(db);
```

### 12.3 [Medium] Code-first vs Database-first: how does scaffolding work and how do you keep it manageable?
**Answer**
`dotnet ef dbcontext scaffold "<conn>" Microsoft.EntityFrameworkCore.SqlServer` generates entities/context from the DB. For a long-lived legacy DB, re-scaffold on schema change with `--force` and put customisations in **partial classes** (never edit generated files), or scaffold once and move to code-first with a baseline migration.

**Explanation**
Options: `--table`/`--schema` to limit, `--data-annotations` vs Fluent default, `--context-dir`/`--output-dir`, `--no-onconfiguring`, `--use-database-names`. Views are scaffolded as keyless types; tables without PK become keyless. Adopting an existing DB into migrations: scaffold, create an `Initial` migration, then make `Up` empty or insert its row into `__EFMigrationsHistory` manually so EF treats it as applied.

**Code**
```bash
dotnet ef dbcontext scaffold "Server=.;Database=Shop;Trusted_Connection=True;TrustServerCertificate=True" Microsoft.EntityFrameworkCore.SqlServer --output-dir Models --context-dir Data --context ShopDb --schema sales --force
```
```csharp
public partial class ShopDb { partial void OnModelCreatingPartial(ModelBuilder mb) => mb.Entity<Order>().HasQueryFilter(o => !o.IsDeleted); }
```

### 12.4 [Hard] Your DB is owned by a DBA team; how do you run EF Core against it responsibly?
**Answer**
Use database-first scaffolding (or hand-maintained model) without EF migrations deploying schema; map only what you need (views, selected tables), treat schema changes as DBA scripts, and add a CI check (scaffold diff or integration tests) to detect drift. Keep your DbContext thin, with `ExcludeFromMigrations` for tables you don't own.

**Explanation**
Drift symptoms: runtime `Invalid column name` errors. Mitigation: contract tests against a schema copy, naming conventions, and read-only access for reporting contexts. Where DBA scripts are authoritative, EF migrations can still be used to *generate* SQL for review but not applied automatically. Triggers/computed columns must be declared in the model to avoid broken `OUTPUT`/writes.

**Code**
```csharp
modelBuilder.Entity<LegacyCustomer>().ToTable("tblCust", t => t.ExcludeFromMigrations());
```

### 12.5 [Tricky] HasData with identity keys and navigation seeding – what goes wrong?
**Answer**
You must provide key values even for identity columns; for dependents you must set FK values (not navigation) and seed principals first (ordering handled by EF but FKs must reference existing keys). Changing a seeded key value results in delete + insert, which can violate FKs or lose data.

**Explanation**
Seeding owned types requires an anonymous object with the owner key: `OwnsOne(...).HasData(new { CustomerId = 1, Street = "x" })`. Also, seed rows are inserted when the migration runs; `EnsureCreated()` also inserts them, but `EnsureCreated` and migrations don't mix (EnsureCreated creates the schema without history).

**Code**
```csharp
modelBuilder.Entity<Customer>().HasData(new Customer { Id = 1, Name = "ACME" });
modelBuilder.Entity<Order>().HasData(new { Id = 1, CustomerId = 1, Total = 10m, CreatedUtc = new DateTime(2024, 1, 1) });
modelBuilder.Entity<Customer>().OwnsOne(c => c.ShipTo).HasData(new { CustomerId = 1, Street = "1 Main", City = "X", Zip = "1" });
```

**Interview traps**
- Using `DateTime.Now`/`Guid.NewGuid()` in HasData.
- Mixing `EnsureCreated()` with migrations.
- Editing scaffolded files directly (lost on re-scaffold).
- Seeding via `Migrate()+Seed` on multiple instances without idempotency.
- Using HasData for users with passwords.

**Cheat sheet**
- HasData = small static, keyed, deterministic data → shows up in migrations.
- Dynamic/large/env data → startup seeder or migration SQL.
- Scaffold: `dotnet ef dbcontext scaffold … --force`; customise via partial classes.
- `EnsureCreated` ≠ migrations.
- Not owner of schema → `ExcludeFromMigrations`, thin context, drift checks.

---

## Topic 13: Loading related data – eager, lazy, explicit

**Study notes**

| Strategy | How | Queries | Risk |
|---|---|---|---|
| Eager | `Include/ThenInclude` | 1 (JOIN) or several with `AsSplitQuery` | cartesian explosion, large payloads |
| Explicit | `Entry(x).Collection(c => c.Orders).Load()` / `.Query()` | on demand | forgetting; N+1 in loops |
| Lazy | `UseLazyLoadingProxies()` + `virtual` navs (or `ILazyLoader`) | one per first access | hidden N+1, sync I/O, disposed context errors |
| Projection | `Select` to DTO | 1, minimal columns | best default for reads |

- Navigation fix-up: tracked entities already in the context get their navigations populated automatically even without `Include`.
- Filtered include (EF5+): `Include(c => c.Orders.Where(o => o.Total > 100).OrderBy(...).Take(5))`.
- `AsSplitQuery()` / `UseQuerySplittingBehavior(SplitQuery)`: one SQL per collection, avoids row duplication but loses single-snapshot consistency and needs deterministic ordering.

### 13.1 [Easy] How do you load related data in EF Core?
**Answer**
Eagerly with `Include`/`ThenInclude`, explicitly with `Entry(...).Collection/Reference(...).Load()`, or lazily via proxies. EF Core does not lazy-load by default, so a navigation not included is null/empty.

**Explanation**
Prefer eager loading or projections where you know what you need. Lazy loading is opt-in because it hides queries and breaks outside a live context.

**Code**
```csharp
var c = await db.Customers.Include(x => x.Orders).ThenInclude(o => o.Lines).FirstAsync(x => x.Id == 1);

var c2 = await db.Customers.FirstAsync(x => x.Id == 1);
await db.Entry(c2).Collection(x => x.Orders).Query().Where(o => o.Total > 100).LoadAsync();
```

### 13.2 [Follow-up] How do you enable lazy loading and what are the downsides?
**Answer**
Install `Microsoft.EntityFrameworkCore.Proxies`, call `UseLazyLoadingProxies()`, and make navigation properties `virtual` (entities non-sealed). Alternatively inject `ILazyLoader`/a delegate into the entity to avoid proxies. Downsides: N+1 queries, synchronous DB calls, serialization triggering loads, and failures when the context is disposed.

**Explanation**
The proxy subclasses the entity and overrides navigations to load on first access, using the owning context. Returning proxies from APIs risks `ObjectDisposedException` or infinite JSON cycles. In async code, lazy loads are synchronous (blocking).

**Code**
```csharp
services.AddDbContext<AppDb>(o => o.UseSqlServer(cs).UseLazyLoadingProxies());
public class Customer { public int Id { get; set; } public virtual ICollection<Order> Orders { get; set; } = new List<Order>(); }

foreach (var cust in await db.Customers.ToListAsync())
    Console.WriteLine(cust.Orders.Count);   // 1 + N queries
```

### 13.3 [Medium] What is the cartesian explosion problem and how does AsSplitQuery help?
**Answer**
Including multiple collections in one query joins them, multiplying rows (orders × lines × notes), which inflates data transfer and materialization time. `AsSplitQuery()` issues one query per collection include, trading extra round trips for far less data.

**Explanation**
EF warns if you use multiple collection includes without specifying a behaviour. Split queries aren't atomic (data may change between statements; use snapshot isolation/transaction if it matters), and you need a unique ordering – EF adds PK ordering. Alternatively project to DTOs or split manually. `AsSingleQuery()` overrides a global split default.

**Code**
```csharp
var orders = await db.Orders.Include(o => o.Lines).Include(o => o.Notes).AsSplitQuery().ToListAsync();
// global: o.UseSqlServer(cs, s => s.UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery));
```
```sql
SELECT ... FROM [Orders] AS [o] ORDER BY [o].[Id];
SELECT [l].*, [o].[Id] FROM [Orders] AS [o] INNER JOIN [OrderLines] AS [l] ON [o].[Id] = [l].[OrderId] ORDER BY [o].[Id];
```

### 13.4 [Hard] Why can navigations be populated even without Include, and why can Include sometimes be ignored?
**Answer**
Relationship fix-up: when a tracking query loads entities, EF connects them to any related entities already tracked, so navigations appear populated. `Include` is ignored when the query's final shape isn't the entity (e.g. you project with `Select`, or call `GroupBy`/set operations that change the result type).

**Explanation**
This makes tests/behaviour flaky: code works in one request because another query loaded related data earlier, but fails in isolation. Use `AsNoTracking` (no fix-up beyond the query) to reveal missing includes. Collections loaded only partially (filtered fix-up) may give incomplete `Orders` lists.

**Code**
```csharp
var c = await db.Customers.FirstAsync(x => x.Id == 1);
var orders = await db.Orders.Where(o => o.CustomerId == 1).ToListAsync();
Console.WriteLine(c.Orders.Count);   // populated by fix-up although never Included
```

### 13.5 [Tricky] Filtered Include and Include on nullable/inheritance paths – what do you need to know?
**Answer**
Filtered include supports `Where`, `OrderBy/ThenBy`, `Skip`, `Take` but only once per navigation and can't be combined with different filters on the same navigation path. With inheritance, use `Include(x => ((Dog)x).Toys)` or `.OfType<Dog>().Include(d => d.Toys)`; string-based `Include("Orders.Lines")` is available but not type safe.

**Explanation**
Tracking + filtered include leaves the collection "incomplete" but marked as loaded; later fix-ups may add other orders. Explicit loading with `.Query()` gives similar flexibility. For top-N-per-group, a projection with `Take` is typically better.

**Code**
```csharp
var q = db.Customers.Include(c => c.Orders.Where(o => o.Total > 100).OrderByDescending(o => o.CreatedUtc).Take(3));
var dogs = db.Animals.OfType<Dog>().Include(d => d.Toys);
```

**Interview traps**
- Believing lazy loading is on by default (that was EF6).
- Using lazy loading and returning entities to JSON serializer.
- Multiple collection `Include`s without split query.
- Relying on fix-up as if it were an `Include`.
- Using `Include` and then `Select` and wondering where includes went.

**Cheat sheet**
- Eager (`Include`), explicit (`Load`), lazy (proxies, opt-in), projection (best).
- `AsSplitQuery` avoids cartesian explosion but not atomic.
- Filtered Include (EF5+) supports Where/OrderBy/Skip/Take.
- Fix-up populates navigations from already-tracked entities.
- Include ignored if result shape isn't the entity.

---

## Topic 14: Pooling, compiled models, multiple DbContexts & bounded contexts

**Study notes**

- **Context pooling** (`AddDbContextPool`): reuses context instances (see Topic 2). **Compiled queries** (`EF.CompileQuery`) skip expression processing. **Compiled models** pre-generate the model code to cut startup time for large models.
- **Compiled model**: `dotnet ef dbcontext optimize -o CompiledModels -n MyApp.CompiledModels` → `UseModel(AppDbModel.Instance)`. Regenerate whenever the model changes. Not all features supported (check docs for your version; e.g. global query filters and proxies are not supported in EF8).
- **Multiple DbContexts**: one per bounded context/module (Sales, Inventory, Identity), possibly same DB. Each has its own model, migrations (`--context`) and history table.

```
 Sales module ──► SalesDb ──┐
 Inventory ─────► InvDb ────┼──► same SQL Server DB / different schemas  (or different DBs)
 Identity ──────► AuthDb ───┘
```

### 14.1 [Easy] Why would you have more than one DbContext?
**Answer**
To separate bounded contexts/modules so each has a small focused model, independent migrations, and its own ownership; also to separate read-only from write contexts or different databases.

**Explanation**
A giant "god" context slows startup/model building and couples teams. Each context shows only what its module needs. Cross-context navigation properties aren't possible; reference other aggregates by ID.

**Code**
```csharp
services.AddDbContext<SalesDb>(o => o.UseSqlServer(cs));
services.AddDbContext<InventoryDb>(o => o.UseSqlServer(cs));
```
```bash
dotnet ef migrations add Init --context SalesDb --output-dir Migrations/Sales
```

### 14.2 [Follow-up] How do migrations work with multiple contexts sharing one database?
**Answer**
Each context needs its own migrations folder/assembly and a distinct history table (or schema) via `MigrationsHistoryTable`, and you target one with `--context`. Without separate history tables the contexts would conflict in `__EFMigrationsHistory`.

**Explanation**
Use default schemas (`HasDefaultSchema("sales")`) to prevent table-name collisions. If two contexts map the same table, only one should own migrations; use `ExcludeFromMigrations` in the other.

**Code**
```csharp
o.UseSqlServer(cs, s => s.MigrationsHistoryTable("__SalesMigrations", "sales"));
protected override void OnModelCreating(ModelBuilder mb) => mb.HasDefaultSchema("sales");
```

### 14.3 [Medium] How do you make two contexts participate in one transaction?
**Answer**
Share the same connection and transaction: begin the transaction on one context and call `Database.UseTransaction(tx.GetDbTransaction())` on the other (they must use the same `DbConnection`). Otherwise use `TransactionScope` (may escalate to distributed transaction – not supported on .NET Core for multiple connections) or the outbox pattern for eventual consistency.

**Explanation**
Both contexts must be created with the same connection object (`UseSqlServer(connection)`). For microservice or multi-DB scenarios, prefer sagas/outbox over distributed transactions.

**Code**
```csharp
await using var conn = new SqlConnection(cs);
await conn.OpenAsync();
await using var salesDb = new SalesDb(new DbContextOptionsBuilder<SalesDb>().UseSqlServer(conn).Options);
await using var invDb   = new InventoryDb(new DbContextOptionsBuilder<InventoryDb>().UseSqlServer(conn).Options);
await using var tx = await salesDb.Database.BeginTransactionAsync();
await invDb.Database.UseTransactionAsync(tx.GetDbTransaction());
salesDb.Orders.Add(order); await salesDb.SaveChangesAsync();
invDb.Stock.Add(movement); await invDb.SaveChangesAsync();
await tx.CommitAsync();
```

### 14.4 [Hard] What are compiled models and when are they worth it? What are their limits?
**Answer**
A compiled model is generated C# code that builds the EF model without runtime reflection/convention processing, reducing first-use cost (seconds → ms for models with hundreds of entities). Worth it for large models, serverless cold starts, or CLI tools. Limits: must be regenerated on each model change, and some features are unsupported (e.g. global query filters, lazy-loading/change-tracking proxies in EF8).

**Explanation**
Generation: `dotnet ef dbcontext optimize`. Use via `UseModel(...)`. Forgetting to regenerate causes runtime mismatch with the actual model: stale compiled model → missing columns/ignored changes, so wire generation into the build/CI. Compiled *queries* are a separate optimisation (`EF.CompileQuery/CompileAsyncQuery`) mostly useful for hot paths – EF already caches query compilation, so gain is the skipped cache lookup and expression visiting.

**Code**
```bash
dotnet ef dbcontext optimize --output-dir CompiledModels --namespace MyApp.CompiledModels
```
```csharp
services.AddDbContext<AppDb>(o => o.UseSqlServer(cs).UseModel(MyApp.CompiledModels.AppDbModel.Instance));
```
**Gotchas / follow-ups**
- Verify supported features in the docs before adopting; regenerate in CI after every model change.

### 14.5 [Tricky] Bounded contexts: can you JOIN across contexts, and how do you handle shared reference data?
**Answer**
Not in a single LINQ query across two different `DbContext` types (the LINQ provider only knows one model). Options: map the shared table (read-only, `ExcludeFromMigrations`) into both contexts, query a view, use raw SQL, or compose in memory via IDs. Prefer keeping aggregates referencing each other by ID.

**Explanation**
Duplicating a read-only entity mapping across contexts is OK (one writer). Keep one context as the single writer per table to avoid conflicting migrations/concurrency rules. Cross-context transactional integrity: shared connection/transaction (same DB) or outbox/saga (different DBs).

**Code**
```csharp
// InventoryDb maps Products owned by CatalogDb read-only
modelBuilder.Entity<ProductRef>().ToTable("Products", "catalog", t => t.ExcludeFromMigrations());
```

**Interview traps**
- Assuming a pooled context can carry per-request state.
- Sharing `__EFMigrationsHistory` among contexts.
- Expecting compiled models to auto-update.
- Joining two `DbContext` `IQueryable`s in one LINQ query.
- Using `TransactionScope` across two connections on .NET Core and expecting MSDTC.

**Cheat sheet**
- One context per bounded context; reference others by ID.
- Separate migrations history/schema per context; `--context`.
- Shared transaction: same connection + `UseTransaction`.
- Compiled model: `dbcontext optimize` + `UseModel`; regenerate on change.
- Compiled queries optimise hot paths only.

---

## Topic 15: Logging & inspecting SQL

**Study notes**

| Tool | What it gives | Notes |
|---|---|---|
| `query.ToQueryString()` | SQL (+ parameter declarations) of an `IQueryable` without running it | EF5+; debugging only |
| `optionsBuilder.LogTo(Console.WriteLine, LogLevel.Information)` | Simple logging without ILogger | filter by categories/events |
| `ILogger` category `Microsoft.EntityFrameworkCore.Database.Command` (Information) | Executed SQL + timing | configure via `appsettings` log levels |
| `EnableSensitiveDataLogging()` | Parameter values / entity key values in logs & exceptions | **dev only (PII)** |
| `EnableDetailedErrors()` | More detail on materialization errors | small overhead |
| `TagWith("…")` | Adds a SQL comment for tracing | find source in profiler |
| `ChangeTracker.DebugView.LongView`, `Model.ToDebugString()` | Tracker / model state | debugging |
| `DbCommandInterceptor` | Hook/modify commands, slow-query logging | production-grade |

### 15.1 [Easy] How can you see the SQL EF Core generates for a query?
**Answer**
Call `ToQueryString()` on the `IQueryable`, or enable logging with `LogTo(Console.WriteLine)` / the `Microsoft.EntityFrameworkCore.Database.Command` log category. SQL Server Profiler / Extended Events also show it.

**Explanation**
`ToQueryString` doesn't execute the query. Logging shows executed commands with execution time (`Executed DbCommand (3ms)`). Parameter *values* are hidden unless `EnableSensitiveDataLogging()` is on.

**Code**
```csharp
var q = db.Orders.Where(o => o.Total > 100).OrderBy(o => o.Id);
Console.WriteLine(q.ToQueryString());

services.AddDbContext<AppDb>(o => o.UseSqlServer(cs)
    .LogTo(Console.WriteLine, new[] { DbLoggerCategory.Database.Command.Name }, LogLevel.Information));
```
```sql
DECLARE @__p_0 decimal(18,2) = 100.0;
SELECT [o].[Id], [o].[Total] FROM [Orders] AS [o] WHERE [o].[Total] > @__p_0 ORDER BY [o].[Id]
```

### 15.2 [Follow-up] What does EnableSensitiveDataLogging do and why is it dangerous in production?
**Answer**
It includes application data (parameter values, key values) in logs and exception messages. In production that can leak PII/secrets to log stores, so restrict it to development.

**Explanation**
Without it, logs show `@p0='?'`. With it, you see real values, essential for diagnosing data-related errors (e.g. FK failures with key values). Pair with `EnableDetailedErrors()` (safe, more diagnostics for null-materialization problems).

**Code**
```csharp
if (env.IsDevelopment()) options.EnableSensitiveDataLogging().EnableDetailedErrors();
```
```json
{ "Logging": { "LogLevel": { "Microsoft.EntityFrameworkCore.Database.Command": "Information" } } }
```

### 15.3 [Medium] How do you find which line of code produced a slow SQL statement?
**Answer**
Add `TagWith("Orders.GetBigOrders")` (and `TagWithCallSite()` in EF Core 6+) which prepend a comment to the SQL; then locate it in logs, Query Store, or profiler.

**Explanation**
Tags are stable query identifiers in the SQL text (they don't change plan caching meaning but do change the text, so differing tags create separate cache entries). Combined with Query Store / Application Insights dependency telemetry you can attribute cost. Use `DbCommandInterceptor` to log commands exceeding a threshold.

**Code**
```csharp
var q = db.Orders.TagWith("Dashboard: big orders").TagWithCallSite().Where(o => o.Total > 1000);
```
```sql
-- Dashboard: big orders
-- File: C:\src\Api\Dashboard.cs:42
SELECT ... FROM [Orders] AS [o] WHERE [o].[Total] > 1000.0
```

### 15.4 [Hard] How do you implement slow-query detection and inspect change-tracker/model state?
**Answer**
Register a `DbCommandInterceptor` and in `ReaderExecutedAsync`/`NonQueryExecutedAsync` check `eventData.Duration`, logging commands above a threshold. For tracker/model debugging use `db.ChangeTracker.DebugView.LongView` and `db.Model.ToDebugString()`.

**Explanation**
Interceptors also let you modify commands (e.g. add query hints), but keep them cheap. Command text and parameters are on `DbCommand`; avoid logging parameter values in production. `ChangeTracker.DebugView.ShortView` prints entities with states, handy in unit tests to assert expected states.

**Code**
```csharp
public class SlowQueryInterceptor(ILogger<SlowQueryInterceptor> log) : DbCommandInterceptor
{
    public override ValueTask<DbDataReader> ReaderExecutedAsync(DbCommand cmd, CommandExecutedEventData e, DbDataReader r, CancellationToken ct = default)
    {
        if (e.Duration > TimeSpan.FromMilliseconds(500)) log.LogWarning("Slow query {Ms}ms: {Sql}", e.Duration.TotalMilliseconds, cmd.CommandText);
        return base.ReaderExecutedAsync(cmd, e, r, ct);
    }
}
Console.WriteLine(db.ChangeTracker.DebugView.ShortView);   // e.g. Customer {Id: 1} Modified
```
**Gotchas / follow-ups**
- Interceptors registered per context options; pooled contexts share them – make them stateless/thread-safe.
- Sync API calls hit the sync overloads (`ReaderExecuted`), so override both sync and async methods if both styles are used.

### 15.5 [Tricky] Why might ToQueryString output differ from what actually runs, and what else can mislead you?
**Answer**
`ToQueryString` generates SQL from the current expression with current parameter values and renders parameters as `DECLARE` statements; it doesn't reflect query splitting (`AsSplitQuery` runs several statements, so check the logs for all of them), the exact `sp_executesql` form used at runtime, or the terminal operator's SQL. Also, `ToQueryString` isn't available for `FirstOrDefault`-style scalar calls (they aren't `IQueryable`).

**Explanation**
For aggregate/scalar operators call `ToQueryString` on the query before the terminal operator, or rely on logging to see exactly what ran. Also remember plan differences: the logged SQL with literals pasted into SSMS can pick a different plan than the parameterised version (parameter sniffing), so test with parameters declared.

**Code**
```csharp
var baseQuery = db.Orders.Where(o => o.CustomerId == 5);
Console.WriteLine(baseQuery.ToQueryString());
var count = await baseQuery.CountAsync();   // actual SQL is SELECT COUNT(*) ...; see logs
```

**Interview traps**
- Leaving `EnableSensitiveDataLogging` on in production.
- Copy/pasting logged SQL with literals and assuming the same plan.
- Thinking `ToQueryString` executes or shows every statement of a split query.
- Logging at Information level in hot paths without filtering categories.
- Ignoring `TagWith` as a cheap observability tool.

**Cheat sheet**
- `ToQueryString()` = SQL without executing.
- `LogTo` or `Microsoft.EntityFrameworkCore.Database.Command` category for executed SQL + time.
- Sensitive data logging = dev only; detailed errors OK.
- `TagWith/TagWithCallSite` to trace origin.
- `DbCommandInterceptor` for slow queries; `ChangeTracker.DebugView` for state.

---

*End of C01.*

