# B05 — .NET Performance, Resilience, Real-time & Testing

**How to use:** Read the *Study notes* first, then cover the answers and try each ladder question in order (Easy → Tricky).
Say the **Answer** aloud in 2-4 sentences, then check the **Explanation** and **Code**; finish each topic with the *Interview traps* and *Cheat sheet*.

## Table of Contents
1. [Caching (IMemoryCache, Distributed, HybridCache, Output/Response caching)](#topic-1-caching)
2. [HttpClient & IHttpClientFactory](#topic-2-httpclient--ihttpclientfactory)
3. [Resilience (Polly / Microsoft.Extensions.Resilience)](#topic-3-resilience-polly--microsoftextensionsresilience)
4. [Response Compression & HTTP Performance Tuning](#topic-4-response-compression--http-performance-tuning)
5. [Profiling & Diagnosing Slow APIs](#topic-5-profiling--diagnosing-slow-apis)
6. [Memory Leak & High-CPU Investigation in Production](#topic-6-memory-leak--high-cpu-investigation-in-production)
7. [BackgroundService / IHostedService / Worker Services](#topic-7-backgroundservice--ihostedservice--worker-services)
8. [Hangfire & Quartz Job Scheduling](#topic-8-hangfire--quartz-job-scheduling)
9. [Message Queues (RabbitMQ, Azure Service Bus)](#topic-9-message-queues-rabbitmq-azure-service-bus)
10. [SignalR](#topic-10-signalr)
11. [gRPC](#topic-11-grpc)
12. [WebSockets & Server-Sent Events](#topic-12-websockets--server-sent-events)
13. [Testing Pyramid, xUnit/NUnit & Mocking](#topic-13-testing-pyramid-xunitnunit--mocking)
14. [Integration Testing & Testing EF Core](#topic-14-integration-testing--testing-ef-core)
15. [TDD/BDD, Coverage, Mutation & Load Testing](#topic-15-tddbdd-coverage-mutation--load-testing)
16. [Docker, Concurrency Tuning & Graceful Degradation](#topic-16-docker-concurrency-tuning--graceful-degradation)

---

## Topic 1: Caching

**Study notes**

Caching trades freshness for speed. Layers (closest to the user first):

```
Browser/CDN  -> Response caching (Cache-Control headers)
Kestrel      -> Output caching (server stores whole responses)
App process  -> IMemoryCache (per-instance, fastest, lost on restart)
Shared       -> IDistributedCache (Redis / SQL Server) - shared across instances, serialized bytes
L1+L2        -> HybridCache (.NET 9+; Microsoft.Extensions.Caching.Hybrid) = memory + distributed + stampede protection
DB
```

- **IMemoryCache**: in-process, stores object references (no serialization). Entries have absolute/sliding expiration, priority, size (only enforced if `SizeLimit` is set — then every entry *must* set `Size`). Use `GetOrCreateAsync` (extension). Thread-safe dictionary, but the factory is **not** run once-only → stampede possible.
- **IDistributedCache**: `byte[]` API (`GetAsync/SetAsync/RefreshAsync/RemoveAsync`; string helpers `GetStringAsync`). Implementations: `AddStackExchangeRedisCache`, `AddDistributedSqlServerCache`, in-memory (dev only). No atomic "get or create".
- **HybridCache** (`AddHybridCache()`, .NET 9 / package `Microsoft.Extensions.Caching.Hybrid`): `GetOrCreateAsync(key, factory, options, tags)`; two tiers, serializer handled for you (System.Text.Json default), **stampede protection** (concurrent callers for same key share one factory call), tag-based invalidation (`RemoveByTagAsync`). Uses any registered `IDistributedCache` as L2.
- **Output caching** (`AddOutputCache`, `UseOutputCache`, .NET 7+): server-side storage of responses, policies, tag eviction (`IOutputCacheStore.EvictByTagAsync`), vary by query/header. **Response caching** (`AddResponseCaching`) honours HTTP headers and is mostly for client/proxy caches; rarely sufficient server-side.
- **Cache-aside**: read cache → miss → load from DB → populate cache. Writes: update DB then invalidate (delete) the key.
- **Stampede (thundering herd)**: many concurrent misses hit the DB. Fixes: single-flight lock per key, HybridCache, jittered TTLs, early refresh.
- **Invalidation**: TTL, explicit remove on write, tags, `CancellationChangeToken`/`IChangeToken`, pub/sub (Redis) for L1 coherence.

### 1.1 [Easy] What is IMemoryCache and how does it differ from IDistributedCache?

**Answer**
`IMemoryCache` is a per-process in-memory cache that stores live object references; it is fastest but not shared between instances and is lost on restart. `IDistributedCache` stores serialized `byte[]` in an external store such as Redis, so all instances share it and it survives app restarts, at the cost of network latency and serialization.

**Explanation**
In a multi-instance deployment (load-balanced, Kubernetes), an `IMemoryCache` write on one pod is invisible to others and invalidations don't propagate. A distributed cache gives consistency across instances. Many production systems use both (L1 + L2), which is exactly what `HybridCache` formalises. Because `IMemoryCache` holds references, mutating a cached object mutates it for everybody — treat cached objects as immutable.

**Code**
```csharp
using Microsoft.Extensions.Caching.Memory;
using Microsoft.Extensions.Caching.Distributed;
using System.Text.Json;

public record Product(int Id, string Name);

public class ProductService(IMemoryCache memory, IDistributedCache distributed)
{
    public async Task<Product?> GetLocalAsync(int id, Func<int, Task<Product?>> load)
        => await memory.GetOrCreateAsync($"product:{id}", entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5);
            return load(id);
        });

    public async Task<Product?> GetSharedAsync(int id, Func<int, Task<Product?>> load, CancellationToken ct)
    {
        var key = $"product:{id}";
        var bytes = await distributed.GetAsync(key, ct);
        if (bytes is not null) return JsonSerializer.Deserialize<Product>(bytes);

        var product = await load(id);
        if (product is not null)
            await distributed.SetAsync(key, JsonSerializer.SerializeToUtf8Bytes(product),
                new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5) }, ct);
        return product;
    }
}
```

### 1.2 [Follow-up] Explain the cache-aside pattern and how you invalidate on writes.

**Answer**
In cache-aside the application owns the logic: read the cache, on a miss load from the database, then populate the cache. On writes, update the database first and then remove (not update) the cache key so the next read repopulates it.

**Explanation**
Deleting is safer than updating because two concurrent writers could write the cache in the opposite order from the DB commit and leave stale data. Even delete-after-write has a small race (a reader that loaded the old value before the write can repopulate after the delete), so keep a TTL as a safety net. For related entries (e.g. a product list and detail), use tags (HybridCache/OutputCache) or key versioning.

**Code**
```csharp
using Microsoft.Extensions.Caching.Memory;

public class Repo(IMemoryCache cache)
{
    private readonly Dictionary<int, string> _db = new() { [1] = "Old" };

    public string? Get(int id) => cache.GetOrCreate($"item:{id}", e =>
    {
        e.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(1); // TTL safety net
        return _db.GetValueOrDefault(id);
    });

    public void Update(int id, string value)
    {
        _db[id] = value;               // 1. write the source of truth
        cache.Remove($"item:{id}");    // 2. invalidate
    }
}
// Get(1) -> "Old" (cached); Update(1,"New"); Get(1) -> "New"
```

### 1.3 [Medium] What is a cache stampede and how does HybridCache protect against it?

**Answer**
A stampede happens when a hot key expires and many concurrent requests all miss and hit the database simultaneously. `HybridCache.GetOrCreateAsync` coalesces concurrent calls for the same key so only one runs the factory while others await its result.

**Explanation**
`IMemoryCache.GetOrCreateAsync` does *not* guarantee single execution: two threads can both see a miss and both run the factory. Manual protection uses a per-key `SemaphoreSlim`. HybridCache does this for you across L1/L2, handles serialization, and supports tags. Additional mitigations: randomised (jittered) TTLs so keys don't expire together, and background refresh before expiry. Note the stampede protection is per-process; across many instances each instance can still call the factory once.

**Code**
```csharp
using Microsoft.Extensions.Caching.Hybrid;
using Microsoft.Extensions.DependencyInjection;

// Program.cs: builder.Services.AddHybridCache(o =>
//   o.DefaultEntryOptions = new HybridCacheEntryOptions
//   { Expiration = TimeSpan.FromMinutes(5), LocalCacheExpiration = TimeSpan.FromMinutes(1) });

public class PriceService(HybridCache cache)
{
    public ValueTask<decimal> GetPriceAsync(int id, CancellationToken ct) =>
        cache.GetOrCreateAsync(
            $"price:{id}",
            async token => await LoadFromDbAsync(id, token), // runs once per key even under concurrency
            tags: ["prices"],
            cancellationToken: ct);

    public ValueTask InvalidateAllAsync(CancellationToken ct) => cache.RemoveByTagAsync("prices", ct);

    private static Task<decimal> LoadFromDbAsync(int id, CancellationToken ct) => Task.FromResult(9.99m);
}
```

### 1.4 [Medium] Output caching vs response caching — when to use which?

**Answer**
Response caching only sets/honours HTTP cache headers and a limited server cache that obeys client headers like `Cache-Control: no-cache`; Output caching is a server-side cache you control with policies, so clients can't bypass it, and you can evict by tag. Prefer output caching for server-side caching of GET endpoints.

**Explanation**
Output caching (.NET 7+) supports policies (`CacheOutput`), vary-by-query/header/route, resource locking (stampede protection for the same URL), tag eviction, and pluggable stores (Redis store available via package). It doesn't automatically cache authenticated responses or responses that set cookies (default policy only caches 200 GET/HEAD without `Authorization`/cookies). Response caching is useful when you want browsers/CDNs to cache (`ResponseCache` attribute emits headers).

**Code**
```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddOutputCache(o =>
{
    o.AddBasePolicy(b => b.Expire(TimeSpan.FromSeconds(30)));
    o.AddPolicy("Products", b => b.Expire(TimeSpan.FromMinutes(5)).Tag("products").SetVaryByQuery("category"));
});
var app = builder.Build();
app.UseOutputCache();

app.MapGet("/products", () => Results.Ok(new[] { "a", "b" })).CacheOutput("Products");
app.MapPost("/products", async (Microsoft.AspNetCore.OutputCaching.IOutputCacheStore store, CancellationToken ct) =>
{
    await store.EvictByTagAsync("products", ct); // invalidate
    return Results.NoContent();
});
app.Run();
```

### 1.5 [Hard] IMemoryCache pitfalls: SizeLimit, eviction, and caching Tasks/closures

**Answer**
If you set `MemoryCacheOptions.SizeLimit`, every entry must specify `Size` or `Set` throws; entries are evicted by compaction/expiration, not by GC pressure alone. Also, unbounded caches keyed by user input are a classic memory leak, and caching a `Task` caches failures too.

**Explanation**
- `SizeLimit` has no unit — you define it (e.g., entry count or KB). Without a limit, `IMemoryCache` grows until the process is out of memory (it does react to GC memory pressure via compaction only in some hosting configs; don't rely on it).
- Sliding expiration without an absolute expiration can keep a hot item forever (stale). Combine both.
- Use `PostEvictionCallbacks` for diagnostics; `MemoryCacheStatistics` (`TrackStatistics = true`, `cache.GetCurrentStatistics()`) exposes hits/misses.
- Caching a `Task<T>` stores the failed task if the factory throws — future callers get the cached exception. Cache the result, not the task, or remove on fault.
- Don't register multiple `IMemoryCache` instances unintentionally: `AddMemoryCache()` is a singleton; creating `new MemoryCache(...)` per request defeats caching and leaks.

**Code**
```csharp
using Microsoft.Extensions.Caching.Memory;

var cache = new MemoryCache(new MemoryCacheOptions { SizeLimit = 2, TrackStatistics = true });
var opts = new MemoryCacheEntryOptions { Size = 1, AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(1) };

cache.Set("a", 1, opts);
cache.Set("b", 2, opts);
cache.Set("c", 3, opts);          // exceeds limit: entry is NOT stored (compaction scheduled)
Console.WriteLine(cache.TryGetValue("c", out _)); // Output: False (typically)

try { cache.Set("d", 4); }       // no Size while SizeLimit is set
catch (InvalidOperationException) { Console.WriteLine("Size required"); } // Output: Size required
```

**Gotchas / follow-ups**
- Over-limit `Set` silently doesn't add the entry; it doesn't throw. Missing `Size` does throw.
- Does `SlidingExpiration` extend beyond `AbsoluteExpiration`? No — absolute wins.

### 1.6 [Tricky] Redis-backed distributed cache: what goes wrong in practice?

**Answer**
Common issues are serialization mismatches across versions, missing TTLs (keys live forever), cache-and-DB inconsistency, treating `IDistributedCache` as atomic, and a Redis outage taking the app down because cache errors aren't handled. Cache failures should degrade to the database, not fail the request.

**Explanation**
`IDistributedCache` has no atomic get-or-create or increment; use `IConnectionMultiplexer` directly for `SET NX`, Lua, or locks. Sliding expiration is implemented via hash fields (`absexp`, `sldexp`) in the Redis implementation. Use a key prefix (`InstanceName`) to separate apps. Wrap calls in try/catch + circuit breaker/timeout so a slow Redis doesn't become a bottleneck. Keep `StackExchange.Redis` `ConnectionMultiplexer` as a singleton (see HttpClient topic for the same pattern idea).

**Code**
```csharp
builder.Services.AddStackExchangeRedisCache(o =>
{
    o.Configuration = builder.Configuration.GetConnectionString("Redis");
    o.InstanceName = "shop:";
});

public class SafeCache(IDistributedCache cache, ILogger<SafeCache> log)
{
    public async Task<string?> TryGetAsync(string key, CancellationToken ct)
    {
        try { return await cache.GetStringAsync(key, ct); }
        catch (Exception ex) { log.LogWarning(ex, "Cache read failed; falling back to DB"); return null; }
    }
}
```

**Gotchas / follow-ups**
- Where does L1 invalidation come from in a hybrid setup? HybridCache's L1 is time-limited (`LocalCacheExpiration`); cross-node eviction of L1 isn't automatic — keep local TTL short.

**Interview traps**
- "Cache everything" — caching mutable, user-specific data under a shared key leaks data between users (always include the user/tenant in the key).
- `IMemoryCache` in a scaled-out app gives inconsistent data between instances.
- Updating the cache after DB write instead of deleting causes race-condition staleness.
- Forgetting HybridCache is .NET 9+ / separate NuGet package (also works on .NET 8 via package).
- Cached `null`: `GetOrCreate` factory returning null is cached as null in IMemoryCache (and avoids repeated misses, but may hide data).

**Cheat sheet**
- L1 = IMemoryCache (refs, fast), L2 = IDistributedCache (bytes, shared), HybridCache = both + stampede protection + tags.
- Cache-aside: read → miss → load → set; write → DB → remove key.
- Always set TTL; add jitter to avoid synchronized expiry.
- Output cache = server-side, policy/tags; Response cache = HTTP headers.
- `SizeLimit` ⇒ every entry needs `Size`.

---

## Topic 2: HttpClient & IHttpClientFactory

**Study notes**

`HttpClient` is a thin wrapper over a handler chain ending in `HttpMessageHandler` (`SocketsHttpHandler` on .NET Core 2.1+), which owns the connection pool.

```
HttpClient -> DelegatingHandler(s) -> SocketsHttpHandler (connection pool per endpoint) -> TCP/TLS
```

- **Misuse #1 – `new HttpClient()` per request with `using`**: each disposes/creates a pool; closed sockets linger in `TIME_WAIT` (~240 s on Windows) → **socket exhaustion** (`SocketException: Only one usage of each socket address...`).
- **Misuse #2 – static/singleton HttpClient forever**: reuses connections but **never respects DNS changes** (connections stay open indefinitely). Fix: `SocketsHttpHandler.PooledConnectionLifetime` (e.g. 2 min) or IHttpClientFactory.
- **IHttpClientFactory** (`AddHttpClient`): pools and recycles `HttpMessageHandler`s (default lifetime **2 minutes**), integrates with DI, logging, Polly/resilience handlers. The `HttpClient` instances it hands out are cheap/transient; the handlers are pooled.
- Flavours: **basic** (`CreateClient()`), **named** (`CreateClient("github")`), **typed** (`AddHttpClient<GitHubClient>()` — client injected into your class), **generated** (Refit).
- A typed client should be registered as **transient**; injecting it into a singleton captures one handler for the singleton's lifetime (loses rotation — use `PooledConnectionLifetime` in that case).
- Use `HttpCompletionOption.ResponseHeadersRead` for streaming large bodies; set `Timeout` (default 100 s); pass `CancellationToken`.

### 2.1 [Easy] Why is `using var client = new HttpClient()` per request a problem?

**Answer**
Each new `HttpClient` creates its own handler and connection pool; disposing it closes the connection but the OS socket stays in `TIME_WAIT`, so under load you run out of ephemeral ports (socket exhaustion). Also you lose connection reuse and pay TLS handshakes every time.

**Explanation**
`HttpClient` is designed to be long-lived and shared. The problem isn't the object allocation, it's the underlying connection. The fix is `IHttpClientFactory` (or a static/singleton client with `PooledConnectionLifetime`).

**Code**
```csharp
// BAD
async Task<string> BadAsync(string url)
{
    using var client = new HttpClient();   // new pool every call
    return await client.GetStringAsync(url);
}

// GOOD
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddHttpClient();          // registers IHttpClientFactory
// inject IHttpClientFactory and call CreateClient()
```

### 2.2 [Follow-up] What problem does a static HttpClient have, and how does IHttpClientFactory solve both problems?

**Answer**
A static client keeps connections open indefinitely, so it never picks up DNS changes (blue/green, failover). `IHttpClientFactory` pools handlers and recycles them every 2 minutes by default, giving both connection reuse and periodic DNS refresh.

**Explanation**
Alternative without DI: a static `HttpClient` created with `new SocketsHttpHandler { PooledConnectionLifetime = TimeSpan.FromMinutes(2) }`. Factory advantages: central configuration, named/typed clients, `DelegatingHandler` pipeline, resilience integration, and logging per client name.

**Code**
```csharp
static readonly HttpClient Shared = new(new SocketsHttpHandler
{
    PooledConnectionLifetime = TimeSpan.FromMinutes(2),
    PooledConnectionIdleTimeout = TimeSpan.FromMinutes(1)
});

// Factory configuration
builder.Services.AddHttpClient("github", c =>
{
    c.BaseAddress = new Uri("https://api.github.com/");
    c.DefaultRequestHeaders.UserAgent.ParseAdd("demo");
})
.SetHandlerLifetime(TimeSpan.FromMinutes(5));
```

### 2.3 [Medium] Typed vs named clients — and how do you add a DelegatingHandler?

**Answer**
A named client is retrieved by string from the factory; a typed client is a class that receives a configured `HttpClient` through its constructor, giving you strong typing and encapsulating the API. Cross-cutting concerns (auth tokens, correlation IDs) go in `DelegatingHandler`s added with `AddHttpMessageHandler`.

**Explanation**
DelegatingHandlers must be registered as **transient** services. Handlers run in the order added (outermost first). Typed clients are transient; don't inject them into singletons without understanding the handler-lifetime implications.

**Code**
```csharp
public class CorrelationHandler : DelegatingHandler
{
    protected override Task<HttpResponseMessage> SendAsync(HttpRequestMessage req, CancellationToken ct)
    {
        req.Headers.TryAddWithoutValidation("X-Correlation-Id", Guid.NewGuid().ToString());
        return base.SendAsync(req, ct);
    }
}

public class GitHubClient(HttpClient http)
{
    public async Task<string> GetRepoAsync(string repo, CancellationToken ct)
    {
        using var resp = await http.GetAsync($"repos/{repo}", ct);
        resp.EnsureSuccessStatusCode();
        return await resp.Content.ReadAsStringAsync(ct);
    }
}

builder.Services.AddTransient<CorrelationHandler>();
builder.Services.AddHttpClient<GitHubClient>(c => c.BaseAddress = new Uri("https://api.github.com/"))
                .AddHttpMessageHandler<CorrelationHandler>();
```

### 2.4 [Hard] How does handler pooling/rotation work internally, and what's the "typed client in a singleton" trap?

**Answer**
The factory keeps `HttpMessageHandler` entries per client name with an expiry timer; when expired, a new handler is used for new clients, while the old one is disposed only after all outstanding clients using it are garbage-collected/released. A typed client captured in a singleton keeps its original handler forever, so DNS never refreshes.

**Explanation**
Internally: `DefaultHttpClientFactory` holds `ActiveHandlerTrackingEntry` (expiring) and `ExpiredHandlerTrackingEntry` (held via weak reference until cleanup). The `HttpClient` returned wraps a `LifetimeTrackingHttpMessageHandler` so `Dispose()` on the client doesn't dispose the pooled handler. The `HttpClient.Dispose` is therefore harmless with the factory. In a singleton (e.g., a `BackgroundService`) either resolve `IHttpClientFactory` and call `CreateClient` per iteration, or configure `PooledConnectionLifetime` on a `SocketsHttpHandler` via `ConfigurePrimaryHttpMessageHandler`.

**Code**
```csharp
builder.Services.AddHttpClient<GitHubClient>()
    .ConfigurePrimaryHttpMessageHandler(() => new SocketsHttpHandler
    {
        PooledConnectionLifetime = TimeSpan.FromMinutes(2),
        MaxConnectionsPerServer = 50
    })
    .SetHandlerLifetime(Timeout.InfiniteTimeSpan); // rely on SocketsHttpHandler's lifetime instead

public class Poller(IHttpClientFactory factory) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            var client = factory.CreateClient("github"); // fresh client per iteration
            await client.GetAsync("/", ct);
            await Task.Delay(TimeSpan.FromSeconds(30), ct);
        }
    }
}
```

**Gotchas / follow-ups**
- Mutating `DefaultRequestHeaders` on a shared singleton client is not thread-safe; set per-request headers on `HttpRequestMessage`.
- `HttpClient.Timeout` can only be set before the first request.

### 2.5 [Tricky] You see `TaskCanceledException` — is it a timeout or caller cancellation? Plus: why dispose `HttpResponseMessage`?

**Answer**
In .NET 5+ a timeout surfaces as `TaskCanceledException` with an inner `TimeoutException`, whereas caller cancellation has the token's `IsCancellationRequested == true` and no `TimeoutException`. Always dispose `HttpResponseMessage` (or use `ResponseHeadersRead` carefully) so the connection returns to the pool.

**Explanation**
Un-disposed responses with buffered content aren't a leak by themselves (content is buffered and the connection released), but with `ResponseHeadersRead` the connection stays checked out until the stream is read/disposed → pool starvation. Also, `HttpClient.Timeout` applies to the entire request incl. reading buffered content; per-try timeouts should use resilience handlers.

**Code**
```csharp
try
{
    using var resp = await http.GetAsync(url, HttpCompletionOption.ResponseHeadersRead, ct);
    resp.EnsureSuccessStatusCode();
    await using var stream = await resp.Content.ReadAsStreamAsync(ct);
    // process stream
}
catch (TaskCanceledException ex) when (ex.InnerException is TimeoutException)
{
    Console.WriteLine("HTTP timeout");          // client timeout
}
catch (OperationCanceledException) when (ct.IsCancellationRequested)
{
    Console.WriteLine("Caller cancelled");
}
```

**Interview traps**
- "HttpClient implements IDisposable so I must `using` it" — not with the factory (and not per-request in general).
- A singleton `HttpClient` without `PooledConnectionLifetime` ignores DNS changes.
- Setting `BaseAddress` without trailing slash drops the last path segment when combining with relative URIs.
- Retrying a request whose `HttpRequestMessage` was already sent: messages can't be re-sent; recreate (resilience handlers do this via factory).

**Cheat sheet**
- Never `new HttpClient()` per call; use `IHttpClientFactory` or a singleton with `PooledConnectionLifetime`.
- Default handler lifetime = 2 min.
- Typed clients are transient; DelegatingHandlers transient.
- Dispose responses; use `ResponseHeadersRead` for streaming.
- Timeout → `TaskCanceledException` + inner `TimeoutException`.

---

## Topic 3: Resilience (Polly / Microsoft.Extensions.Resilience)

**Study notes**

Polly v8 (`Polly.Core`) uses **ResiliencePipeline** (replaces v7 policies). Microsoft packages: `Microsoft.Extensions.Resilience` (DI integration: `AddResiliencePipeline`) and `Microsoft.Extensions.Http.Resilience` (HttpClient handlers: `AddStandardResilienceHandler`, `AddResilienceHandler`).

| Strategy | Purpose | Polly v8 API |
|---|---|---|
| Retry | Transient faults | `AddRetry(RetryStrategyOptions)` |
| Circuit breaker | Stop hammering a failing dependency | `AddCircuitBreaker(CircuitBreakerStrategyOptions)` |
| Timeout | Bound latency | `AddTimeout(TimeSpan)` |
| Bulkhead (isolation) | Limit concurrency | `AddConcurrencyLimiter(permits, queue)` (rate-limiter based) |
| Fallback | Degraded response | `AddFallback(FallbackStrategyOptions<T>)` |
| Hedging | Parallel speculative attempts | `AddHedging` |

```
Request -> [Total timeout] -> [Retry] -> [Circuit breaker] -> [Attempt timeout] -> call
              (outer)                                                           (inner)
```

- Order matters: strategies added first are **outermost**. Retry wraps breaker so each attempt is counted by the breaker; the attempt timeout is innermost.
- **Circuit breaker states**: Closed → (failure ratio over `SamplingDuration` with ≥ `MinimumThroughput`) → Open (rejects with `BrokenCircuitException` for `BreakDuration`) → Half-Open (one probe) → Closed/Open.
- **Jitter** (`UseJitter = true`) randomises retry delays so clients don't retry in lockstep (retry storms).
- `AddStandardResilienceHandler()` = rate limiter + total timeout (30 s) + retry (3 attempts, exponential, jitter) + circuit breaker + attempt timeout (10 s) — sensible defaults for HTTP.
- Only retry **idempotent** operations (or use idempotency keys).

### 3.1 [Easy] What do retry, circuit breaker and timeout each do, and why combine them?

**Answer**
Retry re-attempts transient failures, a timeout bounds how long a call may take, and a circuit breaker stops calling a dependency that is clearly failing so it can recover and callers fail fast. Combined, they absorb blips without piling load onto an unhealthy service or holding threads indefinitely.

**Explanation**
Retry alone can amplify an outage (retry storm). Breaker without timeout can still hang. Timeouts convert slow failures into fast ones that breakers can count. Add fallback to return degraded data when everything fails.

**Code**
```csharp
using Polly;
using Polly.Retry;

var pipeline = new ResiliencePipelineBuilder()
    .AddRetry(new RetryStrategyOptions
    {
        MaxRetryAttempts = 3,
        Delay = TimeSpan.FromMilliseconds(200),
        BackoffType = DelayBackoffType.Exponential,
        UseJitter = true,
        ShouldHandle = new PredicateBuilder().Handle<HttpRequestException>()
    })
    .AddTimeout(TimeSpan.FromSeconds(5))
    .Build();

await pipeline.ExecuteAsync(async ct =>
{
    await Task.Delay(10, ct); // call dependency
});
```

### 3.2 [Follow-up] How do you configure resilience for an HttpClient in .NET 8?

**Answer**
Install `Microsoft.Extensions.Http.Resilience` and call `.AddStandardResilienceHandler()` on the `IHttpClientBuilder` for defaults, or `.AddResilienceHandler("name", b => ...)` for a custom pipeline. Options of the standard handler (e.g., `Retry.MaxRetryAttempts`) can be tweaked in a configure delegate.

**Explanation**
The standard handler already includes jitter and treats 5xx, 408, 429 and `HttpRequestException`/timeouts as failures. Don't stack it with a second hand-rolled retry or you get multiplicative retries (3 × 3 = 9). Total request timeout (default 30 s) should comfortably exceed attempts × attempt timeout (default 3 × 10 s), otherwise the total budget cuts retries short; and the circuit breaker `SamplingDuration` must be at least double the attempt timeout (validated at startup).

**Code**
```csharp
using Polly;
using Microsoft.Extensions.Http.Resilience;

builder.Services.AddHttpClient<GitHubClient>(c => c.BaseAddress = new Uri("https://api.github.com/"))
    .AddStandardResilienceHandler(o =>
    {
        o.Retry.MaxRetryAttempts = 4;
        o.AttemptTimeout.Timeout = TimeSpan.FromSeconds(3);
        o.TotalRequestTimeout.Timeout = TimeSpan.FromSeconds(20);
        o.CircuitBreaker.SamplingDuration = TimeSpan.FromSeconds(30); // must be >= 2 x AttemptTimeout
    });

// Custom pipeline instead:
builder.Services.AddHttpClient("custom")
    .AddResilienceHandler("my-pipeline", b =>
    {
        b.AddTimeout(TimeSpan.FromSeconds(10));
        b.AddRetry(new HttpRetryStrategyOptions { MaxRetryAttempts = 2, UseJitter = true });
    });
```

### 3.3 [Medium] Explain circuit breaker configuration and the half-open state.

**Answer**
The breaker opens when the failure ratio exceeds `FailureRatio` within `SamplingDuration` and at least `MinimumThroughput` calls were made. It stays open for `BreakDuration`, then goes half-open and allows a probe call: success closes it, failure re-opens it.

**Explanation**
`MinimumThroughput` prevents opening on 1-of-2 failures at low traffic. While open, calls throw `BrokenCircuitException` immediately (no network). Breaker state is per pipeline *instance* — in an HttpClient scenario it's per named client (and with `AddResilienceHandler` per endpoint selector for standard handler). Multiple app instances each have their own breaker; there's no shared state.

**Code**
```csharp
using Polly;
using Polly.CircuitBreaker;

var pipeline = new ResiliencePipelineBuilder()
    .AddCircuitBreaker(new CircuitBreakerStrategyOptions
    {
        FailureRatio = 0.5,
        MinimumThroughput = 10,
        SamplingDuration = TimeSpan.FromSeconds(30),
        BreakDuration = TimeSpan.FromSeconds(15),
        ShouldHandle = new PredicateBuilder().Handle<HttpRequestException>(),
        OnOpened = args => { Console.WriteLine("Circuit opened"); return default; }
    })
    .Build();

try { await pipeline.ExecuteAsync(_ => throw new HttpRequestException()); }
catch (HttpRequestException) { }
// After enough failures: next call throws BrokenCircuitException
```

### 3.4 [Medium] How do you implement bulkhead and fallback in Polly v8?

**Answer**
Polly v8 has no `AddBulkhead`; use `AddConcurrencyLimiter(permitLimit, queueLimit)` to cap parallel executions, and reject with `RateLimiterRejectedException` when full. Fallback uses `AddFallback` with a `FallbackAction` producing a substitute result.

**Explanation**
Bulkhead isolation prevents one slow dependency from consuming all threads/connections (give each dependency its own limiter). Fallback should return stale/cached/default data and log; don't hide errors silently. Fallback is generally the outermost strategy so it catches exceptions from all inner ones (including timeouts and `BrokenCircuitException`).

**Code**
```csharp
using Polly;
using Polly.Fallback;

var pipeline = new ResiliencePipelineBuilder<string>()
    .AddFallback(new FallbackStrategyOptions<string>
    {
        ShouldHandle = new PredicateBuilder<string>().Handle<Exception>(),
        FallbackAction = _ => Outcome.FromResultAsValueTask("cached-value")
    })
    .AddConcurrencyLimiter(permitLimit: 5, queueLimit: 10)
    .Build();

var result = await pipeline.ExecuteAsync<string>(_ => throw new InvalidOperationException("simulated"));
Console.WriteLine(result); // Output: cached-value
```

### 3.5 [Hard] DI-registered pipelines, and why retries without jitter or idempotency are dangerous

**Answer**
Register named pipelines with `services.AddResiliencePipeline("key", b => ...)` and resolve them via `ResiliencePipelineProvider<string>`. Retries without jitter synchronise clients into retry storms, and retrying non-idempotent calls (POST payments) can duplicate side effects unless guarded by an idempotency key.

**Explanation**
Pipelines are thread-safe and reusable; create once, execute many times. Use `ResilienceContext` to pass data. For HTTP, `HttpRetryStrategyOptions` by default retries all methods — set `DisableForUnsafeHttpMethods()` (the standard handler's `Retry.DisableForUnsafeHttpMethods()`) for POST/PATCH/DELETE. Also honour `Retry-After` for 429 (`ShouldRetryAfterHeader = true` on `HttpRetryStrategyOptions`).

**Code**
```csharp
using Polly;
using Polly.Registry;

builder.Services.AddResiliencePipeline("db", b =>
    b.AddRetry(new() { MaxRetryAttempts = 3, UseJitter = true, BackoffType = DelayBackoffType.Exponential })
     .AddTimeout(TimeSpan.FromSeconds(2)));

public class Worker(ResiliencePipelineProvider<string> provider)
{
    public async Task RunAsync(CancellationToken ct)
    {
        var pipeline = provider.GetPipeline("db");
        await pipeline.ExecuteAsync(async token => await Task.Delay(1, token), ct);
    }
}

// HTTP: retry only safe methods
builder.Services.AddHttpClient("api").AddStandardResilienceHandler(o => o.Retry.DisableForUnsafeHttpMethods());
```

**Gotchas / follow-ups**
- Retry + timeout per attempt vs total: a hung attempt without attempt-timeout burns the whole total timeout.
- Retrying inside a transaction/EF execution strategy duplicates retry layers.

### 3.6 [Tricky] Order of strategies: what's the difference between timeout-outside-retry and timeout-inside-retry?

**Answer**
Timeout added before retry is a *total* budget across all attempts; timeout added after retry is a *per-attempt* limit. Getting them swapped is a frequent bug: you may think each try gets 5 s but the whole operation is capped at 5 s.

**Explanation**
Strategies are executed outer→inner in the order added. Standard handler uses both: `TotalRequestTimeout` (outer), then retry, breaker, then `AttemptTimeout` (inner). Breaker outside retry would count one failure per whole retried operation rather than per attempt — usually undesirable.

**Code**
```csharp
var p = new ResiliencePipelineBuilder()
    .AddTimeout(TimeSpan.FromSeconds(30))     // total budget (outer)
    .AddRetry(new() { MaxRetryAttempts = 3 })
    .AddTimeout(TimeSpan.FromSeconds(5))      // per attempt (inner)
    .Build();
```

**Interview traps**
- Using Polly v7 terms (`Policy.Handle<>().WaitAndRetryAsync`) — still works via Polly v7 API but v8 `ResiliencePipeline` is current.
- Retrying on every exception including `OperationCanceledException`/validation errors.
- Stacked retries (client + gateway + service mesh) amplify load.
- Breaker state is local per process, not distributed.

**Cheat sheet**
- Outer→inner: Fallback → total timeout → retry → breaker → attempt timeout.
- Exponential backoff + jitter always.
- Bulkhead in v8 = `AddConcurrencyLimiter`.
- `AddStandardResilienceHandler()` for HttpClient defaults.
- Only retry idempotent operations.

---

## Topic 4: Response Compression & HTTP Performance Tuning

**Study notes**

- **Response compression middleware**: `AddResponseCompression()` + `UseResponseCompression()` (before anything that writes responses). Providers: **Brotli** (best ratio, slower) and **Gzip**; default enabled providers when none specified: Brotli + Gzip. `EnableForHttps` is **false** by default because of **BREACH/CRIME** side-channel attacks (compressing secrets alongside attacker-controlled input over TLS).
- Prefer compressing at the reverse proxy/CDN (IIS, nginx, Front Door) — better, and offloads the app. Don't double-compress. Already-compressed formats (JPEG, PNG, zip) aren't compressed (MIME-type list controls it; defaults include text/json/xml/js/svg).
- `UseRequestDecompression()` handles `Content-Encoding` on requests.
- **Other tuning levers**: async I/O end-to-end; `AsNoTracking`/projections; pagination; avoid `ToList` of giant sets; `System.Text.Json` source generation; streaming `IAsyncEnumerable<T>`; `ArrayPool<T>`/`Span<T>`; Server GC (default for ASP.NET Core apps) and `TieredPGO` (on by default .NET 8), ReadyToRun; HTTP/2 & HTTP/3; keep-alive; ETag/conditional GET; caching; connection pooling to DB; avoiding sync-over-async; reducing payload size (DTOs, field selection).

### 4.1 [Easy] How do you enable response compression in ASP.NET Core?

**Answer**
Call `AddResponseCompression` in services and `UseResponseCompression` early in the pipeline. The client's `Accept-Encoding` header decides whether Brotli or Gzip is used.

**Explanation**
The middleware negotiates by `Accept-Encoding` and sets `Content-Encoding` and `Vary: Accept-Encoding`. Configure providers to set the compression level; `CompressionLevel.Fastest` is usually the best server trade-off.

**Code**
```csharp
using Microsoft.AspNetCore.ResponseCompression;
using System.IO.Compression;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddResponseCompression(o =>
{
    o.Providers.Add<BrotliCompressionProvider>();
    o.Providers.Add<GzipCompressionProvider>();
    o.MimeTypes = ResponseCompressionDefaults.MimeTypes.Concat(["application/problem+json"]);
});
builder.Services.Configure<BrotliCompressionProviderOptions>(o => o.Level = CompressionLevel.Fastest);

var app = builder.Build();
app.UseResponseCompression();
app.MapGet("/", () => new string('a', 10_000));
app.Run();
```

### 4.2 [Follow-up] Why is compression over HTTPS disabled by default, and when should it be enabled?

**Answer**
Compressing HTTPS responses can enable BREACH-style attacks where an attacker infers secrets (CSRF tokens) from compressed response sizes when they can inject reflected content. Enable it (`EnableForHttps = true`) only for responses without secrets mixed with user-controlled data, or mitigate (e.g. per-request token masking).

**Explanation**
In practice, most APIs returning JSON with no reflected secrets enable it; HTML pages with anti-forgery tokens and reflected input are riskier (ASP.NET Core's antiforgery tokens are masked per request which mitigates BREACH). Terminating TLS and compressing at the proxy is the common architecture.

**Code**
```csharp
builder.Services.AddResponseCompression(o => o.EnableForHttps = true);
```

### 4.3 [Medium] Beyond compression, list the highest-impact ways to make an ASP.NET Core API faster.

**Answer**
Fix the biggest cost first: database (indexes, `AsNoTracking`, projection, paging, avoiding N+1), eliminate sync-over-async, cache hot reads, reduce payload size, and reuse connections (HttpClientFactory, DB pooling). Measure with a profiler before optimizing.

**Explanation**
Typical distribution: 70-90% of latency is I/O (DB/HTTP). Micro-optimizations (Span, struct) matter on hot CPU-bound paths only. .NET 8 enables dynamic PGO and tiered compilation; `ReadyToRun`/Native AOT reduce startup. `IAsyncEnumerable` streaming reduces memory for large results. Use `System.Text.Json` source generators to avoid reflection and for AOT.

**Code**
```csharp
using System.Text.Json.Serialization;

public record OrderDto(int Id, decimal Total);

[JsonSerializable(typeof(OrderDto[]))]
internal partial class AppJsonContext : JsonSerializerContext;

// Program.cs
builder.Services.ConfigureHttpJsonOptions(o => o.SerializerOptions.TypeInfoResolverChain.Insert(0, AppJsonContext.Default));

app.MapGet("/orders", async (AppDb db) =>
    await db.Orders.AsNoTracking()
        .Where(o => o.Total > 100)
        .Select(o => new OrderDto(o.Id, o.Total))   // projection: only needed columns
        .Take(100)
        .ToArrayAsync());
```

### 4.4 [Hard] Sync-over-async and thread pool starvation: how does it manifest in an API?

**Answer**
Blocking on `.Result`/`.Wait()` inside request handling ties up thread-pool threads while the awaited I/O completes; under load the pool runs out, queue length grows, the pool injects threads slowly (~1–2 per second after the minimum), and latency spikes while CPU stays low.

**Explanation**
ASP.NET Core has no `SynchronizationContext`, so it doesn't deadlock like old ASP.NET, but still starves the pool. Symptoms: low CPU, high `threadpool-queue-length`, rising `threadpool-thread-count`, timeouts. Fix: `async` all the way, `ConfigureAwait(false)` in libraries, never `Task.Run` to wrap I/O in server code. Raising `ThreadPool.SetMinThreads` is a band-aid.

**Code**
```csharp
// BAD: blocks a pool thread per request
app.MapGet("/bad", (HttpClient c) => c.GetStringAsync("https://example.com").Result);

// GOOD
app.MapGet("/good", async (HttpClient c, CancellationToken ct) =>
    await c.GetStringAsync("https://example.com", ct));
```

**Gotchas / follow-ups**
- `Task.Run(() => syncWork)` in a controller just moves work to another pool thread; no scalability win.

### 4.5 [Tricky] Compression meets caching: what is `Vary: Accept-Encoding` for, and what goes wrong without it?

**Answer**
Any shared cache (CDN, proxy, output cache) must key on `Accept-Encoding`, otherwise a gzip/brotli body cached for one client can be served to a client that can't decode it. The compression middleware adds `Vary: Accept-Encoding`; custom headers or CDN rules that strip `Vary` break this.

**Explanation**
Strong ETags also differ per encoding (the representation differs), so conditional requests need encoding-aware validators. Middleware order affects what a server-side cache stores: verify with a test using `Accept-Encoding: gzip` and without, and read the Microsoft docs' middleware-ordering table for your versions rather than assuming. Also don't compress responses that are already compressed or tiny.

**Code**
```csharp
using var http = new HttpClient(new HttpClientHandler { AutomaticDecompression = System.Net.DecompressionMethods.None });
var req = new HttpRequestMessage(HttpMethod.Get, "https://localhost:5001/");
req.Headers.AcceptEncoding.ParseAdd("br");
var resp = await http.SendAsync(req);
Console.WriteLine(string.Join(",", resp.Content.Headers.ContentEncoding)); // Output: br (if enabled)
Console.WriteLine(string.Join(",", resp.Headers.Vary));                    // Output: Accept-Encoding
```

**Interview traps**
- Compressing small payloads (< ~1 KB) wastes CPU.
- Enabling both proxy and app compression → double work or double-encoding.
- Brotli at max level on every dynamic response burns CPU — use Fastest or precompress static assets.
- Assuming async fixes slow queries — it only frees threads.

**Cheat sheet**
- `AddResponseCompression` + `UseResponseCompression` (early); Brotli > Gzip ratio.
- HTTPS compression off by default (BREACH).
- Compress at proxy/CDN when possible.
- Biggest wins: DB, caching, payload size, no sync-over-async.
- Measure first (counters/trace), then tune.

---

## Topic 5: Profiling & Diagnosing Slow APIs

**Study notes**

Method: **Observe → Hypothesise → Measure → Fix → Verify**. Start broad (is it CPU, memory, threads, or waiting on I/O?) then narrow.

| Tool | Use | Key command |
|---|---|---|
| `dotnet-counters` | Live metrics (CPU, GC, thread pool, requests/s) | `dotnet-counters monitor -p <pid> --counters System.Runtime,Microsoft.AspNetCore.Hosting` |
| `dotnet-trace` | CPU sampling / events → `.nettrace`/speedscope | `dotnet-trace collect -p <pid> --profile cpu-sampling` |
| `dotnet-dump` | Heap/thread analysis of a dump | `dotnet-dump collect -p <pid>` / `dotnet-dump analyze x.dmp` |
| `dotnet-gcdump` | Lightweight managed heap snapshot | `dotnet-gcdump collect -p <pid>` |
| PerfView | Deep ETW/event analysis (GC, CPU stacks, allocations) | GUI |
| Application Insights | Requests, dependencies, exceptions, Live Metrics, Profiler, Snapshot Debugger | Azure Monitor |
| OpenTelemetry | Vendor-neutral traces/metrics/logs | `AddOpenTelemetry()` |
| EF Core logging / MiniProfiler | Slow SQL, N+1 | `LogTo`, `EnableSensitiveDataLogging` |

Typical diagnosis tree:
```
Slow?  -> High CPU?      -> dotnet-trace / PerfView (hot methods), GC % time
       -> Low CPU?       -> waiting: DB/HTTP dependency (App Insights dependency durations),
                            thread pool starvation (counters), lock contention
       -> Memory growing -> gcdump/dump (see Topic 6)
```
Custom metrics use `System.Diagnostics.Metrics` (`Meter`, `Counter<T>`, `Histogram<T>`), traces use `ActivitySource`.

### 5.1 [Easy] How do you start investigating a slow API endpoint in production?

**Answer**
First check telemetry (Application Insights/OpenTelemetry) for the slow operation's end-to-end trace to see where time goes — usually a dependency such as SQL or an HTTP call. Then use `dotnet-counters` to see CPU, GC and thread-pool health, and drill down with `dotnet-trace` or dumps if needed.

**Explanation**
Distributed traces break down request time into spans (SQL, HTTP, Redis). If the app itself is slow with low dependency time, check CPU-bound hotspots, thread pool starvation, GC pauses, or lock contention. Always compare to a baseline and percentile (p95/p99), not just average.

**Code**
```csharp
using System.Diagnostics;
using System.Diagnostics.Metrics;

public static class Telemetry
{
    public static readonly ActivitySource Source = new("Shop.Api");
    public static readonly Meter Meter = new("Shop.Api");
    public static readonly Histogram<double> OrderMs = Meter.CreateHistogram<double>("order.duration", "ms");
}

public class OrderService
{
    public async Task PlaceAsync()
    {
        using var activity = Telemetry.Source.StartActivity("PlaceOrder");
        var sw = Stopwatch.StartNew();
        await Task.Delay(20);
        Telemetry.OrderMs.Record(sw.Elapsed.TotalMilliseconds);
    }
}
```

### 5.2 [Follow-up] Which dotnet-counters tell you what, and how do you use them?

**Answer**
`dotnet-counters monitor` shows live metrics from the `System.Runtime` provider: `cpu-usage`, `gc-heap-size`, `gen-0/1/2-gc-count`, `time-in-gc`, `threadpool-thread-count`, `threadpool-queue-length`, `monitor-lock-contention-count`, `exception-count`, `alloc-rate`. ASP.NET Core adds `Microsoft.AspNetCore.Hosting` (requests-per-second, current-requests, failed-requests).

**Explanation**
Interpretation: high `time-in-gc` or Gen2 counts → allocation/leak; growing `threadpool-queue-length` with low CPU → starvation; high `exception-count` → exceptions as flow control; high `monitor-lock-contention-count` → locks. It attaches to a running process without restarting and works in containers (diagnostic IPC socket in `/tmp` — mount shared volume or use `kubectl exec`).

**Code**
```csharp
// Shell commands (comments), then in-process listening to a counter
// dotnet-counters ps
// dotnet-counters monitor -p 1234 --counters System.Runtime,Microsoft.AspNetCore.Hosting
// dotnet-counters collect -p 1234 --format json -o counters

using System.Diagnostics.Metrics;
var meter = new Meter("Demo");
var counter = meter.CreateCounter<int>("items.processed");
counter.Add(1); // visible with: dotnet-counters monitor --counters Demo
```

### 5.3 [Medium] CPU-bound vs I/O-bound: how do you tell, and how do you capture a CPU profile?

**Answer**
CPU-bound means `cpu-usage` is high and the thread pool is busy running code; I/O-bound shows low CPU with long wall time in dependencies. Capture a CPU sample with `dotnet-trace collect --profile cpu-sampling`, then open the `.nettrace` in PerfView or convert to speedscope (`--format speedscope`).

**Explanation**
Sampling profilers capture stacks about every 1 ms for running threads only; they won't show time spent waiting. For async waits use traces with the TPL/event providers or Application Insights Profiler, which does capture wall-clock hot paths. Flame graphs show hot call paths; look for JSON serialization, regex, LINQ in loops, string concatenation, excessive allocations.

**Code**
```csharp
// dotnet-trace collect -p 1234 --profile cpu-sampling --duration 00:00:30 --format speedscope
// Hot path example to look for in a flame graph:
static int Slow(IEnumerable<int> data)
{
    var total = 0;
    foreach (var x in data)
        if (data.Contains(x)) total++;   // O(n^2) hidden in LINQ - shows as Enumerable.Contains hot spot
    return total;
}
```

### 5.4 [Hard] What does Application Insights give you, and what are its pitfalls?

**Answer**
App Insights collects requests, dependencies (SQL, HTTP), exceptions, custom metrics, and an application map; Live Metrics, Profiler and Snapshot Debugger help diagnose in production. Pitfalls: adaptive **sampling** drops telemetry, high-cardinality custom dimensions cost money, and the SDK's default behaviour can hide the true request rate.

**Explanation**
Use `Azure.Monitor.OpenTelemetry.AspNetCore` (`UseAzureMonitor()`) for the modern OpenTelemetry-based distro; the classic `Microsoft.ApplicationInsights.AspNetCore` SDK also works. Correlation uses W3C `traceparent`. Log slow queries via dependency telemetry with durations. Remember telemetry is async-batched and may be lost when the process crashes (call flush on shutdown).

**Code**
```csharp
// Azure.Monitor.OpenTelemetry.AspNetCore
builder.Services.AddOpenTelemetry().UseAzureMonitor();   // reads APPLICATIONINSIGHTS_CONNECTION_STRING

// Classic SDK alternative
// builder.Services.AddApplicationInsightsTelemetry();
```

### 5.5 [Tricky] Your endpoint is slow only under load and CPU is 15%. What are the likely causes?

**Answer**
Low CPU with high latency under load points to waiting: thread-pool starvation (sync-over-async), connection pool exhaustion (DB `Max Pool Size`, HttpClient limits), lock contention, or a downstream throttle. Check `threadpool-queue-length`, DB pool waits and dependency latency.

**Explanation**
Check in order: (1) thread pool counters; (2) SQL connection pool — "Timeout expired. The timeout period elapsed prior to obtaining a connection from the pool" means leaked/unreleased connections or too small pool; (3) `dotnet-dump analyze` → `clrstack -all` / `threads` to see many threads blocked on `Task.Wait` or `Monitor.Enter`; (4) Kestrel `MaxConcurrentConnections`/rate limiter queueing. Reproduce with load test.

**Code**
```csharp
// dotnet-dump analyze dump.dmp
//   > threads
//   > clrstack -all           (look for many identical blocked stacks)
//   > syncblk                 (monitor lock owners)
ThreadPool.GetAvailableThreads(out var worker, out var io);
ThreadPool.GetMinThreads(out var minW, out var minIo);
Console.WriteLine($"pending={ThreadPool.PendingWorkItemCount} threads={ThreadPool.ThreadCount} availW={worker}");
```

**Interview traps**
- Profiling in Debug build or with the debugger attached gives misleading data.
- Only watching averages — tail latency (p99) is where users hurt.
- Blaming GC without checking `% time in GC`.
- Sampling profilers don't see async waiting time.

**Cheat sheet**
- counters (what) → trace (where, CPU) → dump (why, memory/threads).
- Low CPU + slow = waiting (pool starvation, DB pool, locks).
- `dotnet-trace collect --profile cpu-sampling`.
- `dotnet-gcdump` is light; `dotnet-dump` is full.
- Use `Meter`/`ActivitySource` for custom telemetry.

---

## Topic 6: Memory Leak & High-CPU Investigation in Production

**Study notes**

Managed "leaks" = objects still **rooted** (reachable) though no longer needed. Common roots: static collections/caches without eviction, event handlers never unsubscribed, `Timer`s, long-lived singletons capturing scoped objects, `HttpClient` per request, `Task`s never completing, `ThreadLocal`/`AsyncLocal`, finalizer queue backlog, unmanaged/native leaks (not visible in GC heap), `IDisposable` not disposed.

```
Memory growing?
  gc-heap-size grows, Gen2 grows  -> managed leak -> gcdump/dump: dumpheap -stat -> gcroot
  working-set grows, heap flat    -> native leak / fragmentation / LOH / pinned / container limit
  Many Gen0 GCs, high alloc-rate  -> allocation churn (not a leak)
```

Workflow (dotnet-dump): `dotnet-dump collect -p <pid>` → `dotnet-dump analyze x.dmp` → `dumpheap -stat` (biggest types) → `dumpheap -type Foo -min 1000` → `gcroot <address>` (who keeps it alive). Take **two** gcdumps minutes apart and diff (Visual Studio / PerfView) to see growth. High CPU: `dotnet-counters` (cpu-usage, time-in-gc) → `dotnet-trace` (cpu-sampling) → hot method; or dump → `threads`, `clrstack`. On Linux containers use `DOTNET_gcServer`, `DOTNET_GCHeapHardLimit`, and enable createdump (`DOTNET_DbgEnableMiniDump=1`) for crash dumps. Container OOMKilled usually means the cgroup limit was exceeded: .NET respects it (GC heap hard limit defaults to 75% of container limit).

### 6.1 [Easy] What counts as a memory leak in a garbage-collected runtime?

**Answer**
A leak is memory that is still reachable from a GC root but no longer needed, so the GC can't reclaim it. Typical causes are static collections, un-removed event handlers, and unbounded caches.

**Explanation**
GC collects unreachable objects only. A `static Dictionary` that only grows, or a publisher event holding a subscriber, keeps whole object graphs alive. Native resources (file handles, unmanaged memory) leak when `Dispose`/`SafeHandle` isn't used.

**Code**
```csharp
public class Publisher { public event EventHandler? Changed; }

public class Subscriber : IDisposable
{
    private readonly Publisher _p;
    public Subscriber(Publisher p) { _p = p; _p.Changed += OnChanged; }   // publisher now references subscriber
    private void OnChanged(object? s, EventArgs e) { }
    public void Dispose() => _p.Changed -= OnChanged;                    // fix: unsubscribe
}

static class Leaky { public static readonly List<byte[]> Cache = new(); } // grows forever
```

### 6.2 [Follow-up] How do you find what's leaking in a running production process?

**Answer**
Watch `gc-heap-size`/Gen2 size with `dotnet-counters`; capture a `dotnet-gcdump` or `dotnet-dump`, run `dumpheap -stat` to find the dominant types, and use `gcroot` on a sample instance to see the reference chain keeping it alive. Comparing two snapshots shows what's growing.

**Explanation**
`gcdump` is smaller and faster (heap graph only, no data) and suitable for production; a full dump includes memory contents (sensitive!) and pauses the process while captured. Analyze offline in `dotnet-dump`, Visual Studio, or PerfView. Fix then confirm via counters that the heap plateaus after Gen2 GCs.

**Code**
```csharp
// dotnet-counters monitor -p 1234 System.Runtime[gc-heap-size,gen-2-gc-count]
// dotnet-gcdump collect -p 1234 -o before.gcdump   ... wait ... -o after.gcdump
// dotnet-dump analyze app.dmp
//   > dumpheap -stat
//   > dumpheap -type MyApp.OrderCache -min 85000
//   > gcroot 000001f2a3b4c5d0
GC.Collect();
Console.WriteLine(GC.GetTotalMemory(forceFullCollection: true));  // use sparingly, diagnostics only
```

### 6.3 [Medium] Common .NET leak sources in ASP.NET Core apps

**Answer**
Captive dependencies (singleton holding scoped `DbContext`), unbounded `IMemoryCache`/static dictionaries, event handlers, `Timer`/`CancellationTokenSource.CreateLinkedTokenSource` not disposed, `HttpClient` created per call, and large-object-heap fragmentation from big arrays/strings. Each pins memory for the life of the root.

**Explanation**
A scoped `DbContext` captured by a singleton accumulates tracked entities forever. `CancellationTokenSource` linked sources registered to a long-lived parent token leak unless disposed. Strings/arrays ≥ 85,000 bytes go to LOH (collected only with Gen2); use `ArrayPool<T>`, streaming, and `GCSettings.LargeObjectHeapCompactionMode` (rarely needed). Enable scope validation (`ValidateScopes`, on in Development) to catch captive dependencies.

**Code**
```csharp
builder.Host.UseDefaultServiceProvider(o => { o.ValidateScopes = true; o.ValidateOnBuild = true; });

public class Reporter(IServiceScopeFactory scopes)   // singleton-safe pattern
{
    public async Task RunAsync()
    {
        using var scope = scopes.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDb>();
        await db.Orders.AsNoTracking().CountAsync();
    }
}
```

### 6.4 [Hard] The process memory (working set) keeps growing but the GC heap is flat. What now?

**Answer**
That suggests native memory (unmanaged allocations, native libraries, image/PDF libs), heap fragmentation/not-yet-decommitted GC memory, thread stacks, or the GC deliberately holding memory (Server GC retains more). Look at GC heap vs working set, `GCHeapAffinitizeMask`/`GCConserveMemory`, and use native tools (dotnet-dump `eeheap`, `dumpheap` for pinned handles, `perf`/WinDbg/`!address`).

**Explanation**
Server GC with many cores uses a heap per core and holds on to memory — in containers set `DOTNET_GCHeapCount`, `DOTNET_GCHeapHardLimit`, or use Workstation concurrent GC for small pods. Pinned objects block compaction, causing fragmentation. Excess threads each reserve stack space. Libraries like `System.Drawing`/SkiaSharp/native SDKs need explicit disposal. `GC.GetGCMemoryInfo()` gives fragmentation and committed numbers.

**Code**
```csharp
var info = GC.GetGCMemoryInfo();
Console.WriteLine($"Heap: {info.HeapSizeBytes / 1_048_576} MB");
Console.WriteLine($"Fragmented: {info.FragmentedBytes / 1_048_576} MB");
Console.WriteLine($"Committed: {info.TotalCommittedBytes / 1_048_576} MB");
Console.WriteLine($"Limit: {info.TotalAvailableMemoryBytes / 1_048_576} MB");
```

**Gotchas / follow-ups**
- `GC.Collect()` in production as a "fix" masks leaks and hurts latency.
- Generation 2 not shrinking right after a leak fix is normal until a full GC occurs.

### 6.5 [Tricky] CPU is pegged at 100% — how do you find the culprit without restarting, and what are common causes?

**Answer**
Confirm with `dotnet-counters` (cpu-usage, `time-in-gc`), then `dotnet-trace collect --profile cpu-sampling` for 30–60 s and read the hot stacks. Frequent culprits: infinite/tight loops (`while(true)` without delay), regex backtracking, excessive GC from allocations, serialization, exception storms, and spin-waiting on locks.

**Explanation**
If `time-in-gc` is high, the "CPU" is the GC → fix allocations/leak. If a single thread is hot, take a dump and run `clrstack` on the hot thread (`threads`, then `setthread`). Catastrophic regex backtracking can be prevented with `RegexOptions.NonBacktracking` (.NET 7+) or a `matchTimeout`. Exceptions are expensive (~10–50 µs each); thousands/sec burn CPU (`exception-count` counter).

**Code**
```csharp
using System.Text.RegularExpressions;

// Vulnerable to catastrophic backtracking on "aaaa...!":
var bad = new Regex("^(a+)+$", RegexOptions.None, TimeSpan.FromMilliseconds(200)); // timeout protects
// Safe, linear-time engine:
var safe = new Regex("^(a+)+$", RegexOptions.NonBacktracking);
Console.WriteLine(safe.IsMatch(new string('a', 50) + "!")); // Output: False
```

**Interview traps**
- Saying "GC will handle it" for event handlers/statics.
- Full memory dumps contain secrets/PII — handle them as sensitive data.
- Treating high working set as a leak without checking GC settings/container limits.
- Using `GC.Collect()` or `GC.AddMemoryPressure` as production fixes.

**Cheat sheet**
- Leak = still-rooted garbage; find the root with `gcroot`.
- Heap grows → managed; working set grows, heap flat → native/fragmentation.
- `dumpheap -stat` → `gcroot` is the core loop.
- High CPU → counters → `dotnet-trace cpu-sampling`.
- Check `time-in-gc` before blaming your code.

---

## Topic 7: BackgroundService / IHostedService / Worker Services

**Study notes**

- `IHostedService` (`StartAsync`, `StopAsync`) — managed by the Generic Host. `BackgroundService` is an abstract base class implementing it with a single `ExecuteAsync(CancellationToken)`. .NET 8 adds `IHostedLifecycleService` (StartingAsync/StartedAsync/StoppingAsync/StoppedAsync).
- Hosted services are started **sequentially in registration order** and stopped in reverse. In .NET 8 and earlier, the synchronous part of `ExecuteAsync` (before the first `await`) **blocks host startup** (so don't do long sync work first; `await Task.Yield()` if needed).
- **Exceptions**: since .NET 6, an unhandled exception in `ExecuteAsync` stops the host (`BackgroundServiceExceptionBehavior.StopHost`, default); `Ignore` is opt-in.
- **Shutdown**: on SIGTERM/Ctrl+C the host triggers `stoppingToken` and waits up to `HostOptions.ShutdownTimeout` (**30 s** default since .NET 6; was 5 s) for services to finish.
- Hosted services are **singletons** → use `IServiceScopeFactory` to get scoped services (DbContext).
- Periodic work: `PeriodicTimer` (async-friendly, no overlapping ticks) beats `Timer`/`Task.Delay` loops. Queueing work from requests: `Channel<T>` producer/consumer.
- Worker Service template: `Host.CreateApplicationBuilder(args)`; run as Windows Service (`AddWindowsService`) or systemd (`AddSystemd`) or container.

### 7.1 [Easy] How do you create a background task in ASP.NET Core?

**Answer**
Derive from `BackgroundService`, implement `ExecuteAsync`, and register with `AddHostedService<T>()`. The host starts it with the app and signals the cancellation token at shutdown.

**Explanation**
Use it for polling, queue consumers, cache warmers. Always loop on `stoppingToken` and pass it to awaited calls so shutdown is prompt.

**Code**
```csharp
public class HeartbeatService(ILogger<HeartbeatService> log) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromSeconds(10));
        while (await timer.WaitForNextTickAsync(stoppingToken))
            log.LogInformation("alive at {Time}", DateTimeOffset.UtcNow);
    }
}

builder.Services.AddHostedService<HeartbeatService>();
```

### 7.2 [Follow-up] How do you use a scoped service (DbContext) inside a BackgroundService?

**Answer**
Inject `IServiceScopeFactory`, create a scope per unit of work, resolve the scoped service from it, and dispose the scope afterwards. Injecting the scoped service directly fails (or captures a stale instance).

**Explanation**
Hosted services are singletons; scoped services injected into them trigger a scope-validation error in Development. A scope per iteration/message also gives you a fresh `DbContext` (no ever-growing change tracker).

**Code**
```csharp
public class CleanupService(IServiceScopeFactory scopeFactory) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromMinutes(1));
        while (await timer.WaitForNextTickAsync(ct))
        {
            using var scope = scopeFactory.CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<AppDb>();
            await db.Sessions.Where(s => s.Expires < DateTime.UtcNow).ExecuteDeleteAsync(ct);
        }
    }
}
```

### 7.3 [Medium] Implement an in-process work queue with `Channel<T>` consumed by a BackgroundService

**Answer**
Register a bounded `Channel<T>` as a singleton; request handlers `WriteAsync` to it, and a `BackgroundService` reads via `ReadAllAsync`. Bounded capacity provides back-pressure.

**Explanation**
This decouples request latency from slow work, but the queue is **in-memory**: items are lost on crash/restart, so use it for best-effort work only; otherwise use a durable queue (Service Bus/RabbitMQ) or outbox. `BoundedChannelFullMode` decides whether writers wait or drop.

**Code**
```csharp
using System.Threading.Channels;

public record EmailJob(string To);

builder.Services.AddSingleton(Channel.CreateBounded<EmailJob>(
    new BoundedChannelOptions(100) { FullMode = BoundedChannelFullMode.Wait }));
builder.Services.AddHostedService<EmailWorker>();

app.MapPost("/email", async (Channel<EmailJob> ch, string to) =>
{
    await ch.Writer.WriteAsync(new EmailJob(to));
    return Results.Accepted();
});

public class EmailWorker(Channel<EmailJob> ch, ILogger<EmailWorker> log) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        await foreach (var job in ch.Reader.ReadAllAsync(ct))
        {
            try { log.LogInformation("Sending to {To}", job.To); await Task.Delay(100, ct); }
            catch (Exception ex) when (ex is not OperationCanceledException) { log.LogError(ex, "Failed"); }
        }
    }
}
```

### 7.4 [Hard] What happens on unhandled exceptions and on shutdown? How do you make shutdown graceful?

**Answer**
An unhandled exception in `ExecuteAsync` logs and stops the host by default (.NET 6+), so catch and handle per-iteration errors inside the loop. On shutdown, the token is cancelled and the host waits up to `ShutdownTimeout` (30 s) for `ExecuteAsync` to complete; services that ignore the token get abandoned.

**Explanation**
Wrap the loop body in try/catch (excluding `OperationCanceledException`) so one bad message doesn't kill the worker. For in-flight work, let it finish with a token that isn't cancelled until the timeout (use a separate CTS for work if needed). In Kubernetes, set `terminationGracePeriodSeconds` ≥ `ShutdownTimeout`. `StopAsync` override can flush buffers after `base.StopAsync`.

**Code**
```csharp
builder.Services.Configure<HostOptions>(o =>
{
    o.ShutdownTimeout = TimeSpan.FromSeconds(45);
    o.BackgroundServiceExceptionBehavior = BackgroundServiceExceptionBehavior.StopHost; // default
});

public class Resilient(ILogger<Resilient> log) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            try { await DoWorkAsync(ct); }
            catch (OperationCanceledException) when (ct.IsCancellationRequested) { break; }
            catch (Exception ex) { log.LogError(ex, "Iteration failed"); await Task.Delay(5000, ct); }
        }
    }
    private static Task DoWorkAsync(CancellationToken ct) => Task.Delay(1000, ct);
}
```

### 7.5 [Tricky] Why does my app not start (or hang) because of a BackgroundService?

**Answer**
In .NET 8 and earlier the host awaits `StartAsync`, which runs `ExecuteAsync` synchronously until its first `await`. If your code does long synchronous work or `.Wait()`s before awaiting, startup (and readiness) is blocked.

**Explanation**
`BackgroundService.StartAsync` calls `ExecuteAsync`, stores the task, and returns it only if already completed; otherwise returns `Task.CompletedTask`. So everything up to the first truly asynchronous await runs on the startup path. Add `await Task.Yield()` at the top for heavy initialization, or perform it in `StartAsync`/`StartedAsync` deliberately. Also: hosted services registered *before* the web server start first in a WebApplication (the server starts last in .NET 8 generic host ordering for `GenericWebHostService`), so dependency on the HTTP server being up isn't guaranteed.

**Code**
```csharp
protected override async Task ExecuteAsync(CancellationToken ct)
{
    await Task.Yield();             // return control so host startup continues
    HeavySyncInit();                // now off the startup path
    while (!ct.IsCancellationRequested) await Task.Delay(1000, ct);
}
static void HeavySyncInit() => Thread.Sleep(5000);
```

**Interview traps**
- `Task.Delay` loops drift and overlap-proof differently vs `PeriodicTimer`.
- Fire-and-forget `Task.Run` in controllers instead of a hosted service — lost on shutdown, unobserved exceptions.
- Running multiple instances → duplicate execution (need leader election/distributed lock).
- Injecting scoped services directly.

**Cheat sheet**
- `BackgroundService.ExecuteAsync(stoppingToken)`; register via `AddHostedService`.
- Scoped deps → `IServiceScopeFactory`.
- Unhandled exception stops host (.NET 6+); `ShutdownTimeout` = 30 s.
- `PeriodicTimer` for periodic loops; `Channel<T>` for in-process queues.
- Sync code before first await blocks startup (≤ .NET 8).

---

## Topic 8: Hangfire & Quartz Job Scheduling

**Study notes**

| | Hangfire | Quartz.NET |
|---|---|---|
| Model | Persistent background jobs (fire-and-forget, delayed, recurring, continuations, batches*) | Scheduler with Jobs + Triggers |
| Storage | SQL Server (also Redis/Postgres via packages), required | In-memory (RAMJobStore) or ADO.NET (AdoJobStore) |
| Definition | Expression `Enqueue(() => svc.Do(x))` – method + serialized args stored | `IJob` classes; `JobDataMap` |
| Dashboard | Built-in | Third-party |
| Retries | Automatic (default 10 attempts, `[AutomaticRetry]`) | Manual (job handles / listeners) |
| Cron | 5-field (Cronos) | Quartz cron: seconds first, 6–7 fields |
| Clustering | Multiple servers share storage, jobs locked | AdoJobStore `Clustered = true` |

(*batches are in Hangfire Pro.) Hangfire serializes the **method call and arguments** as JSON; keep arguments small/primitive (IDs, not entities). Jobs get DI via the activator (`UseSimpleInjector`/ASP.NET Core integration creates a scope per job). Jobs must be **idempotent** (at-least-once; may run twice after server crash). Concurrency control: `[DisableConcurrentExecution(timeoutInSeconds)]` (Hangfire), `[DisallowConcurrentExecution]` (Quartz).

### 8.1 [Easy] What job types does Hangfire support?

**Answer**
Fire-and-forget (`BackgroundJob.Enqueue`), delayed (`Schedule`), recurring (`RecurringJob.AddOrUpdate` with cron), and continuations (`ContinueJobWith`). Jobs are persisted in storage (e.g. SQL Server), so they survive app restarts.

**Explanation**
Server threads poll storage, fetch a job, and execute it with automatic retries on exception. The dashboard (`/hangfire`) shows state; secure it with an authorization filter (by default only localhost is allowed).

**Code**
```csharp
builder.Services.AddHangfire(c => c.UseSqlServerStorage(builder.Configuration.GetConnectionString("Hangfire")));
builder.Services.AddHangfireServer();

var app = builder.Build();
app.UseHangfireDashboard("/hangfire");

BackgroundJob.Enqueue<IReportService>(s => s.GenerateAsync(42));
BackgroundJob.Schedule<IReportService>(s => s.GenerateAsync(43), TimeSpan.FromMinutes(10));
RecurringJob.AddOrUpdate<IReportService>("nightly-report", s => s.GenerateAsync(0), "0 2 * * *");
```

### 8.2 [Follow-up] How are Hangfire job arguments handled, and what are the DI/lifetime implications?

**Answer**
Hangfire stores the type, method and JSON-serialized arguments, then reconstructs the call on a worker, resolving the class through DI in a new scope per job. So pass IDs/small DTOs, not entities, `DbContext`, or `HttpContext`.

**Explanation**
Changing a method signature or renaming a type while jobs are queued can make old jobs fail deserialization. Use interface-based enqueueing (`Enqueue<IService>`) to avoid binding to implementation. A `CancellationToken` parameter is supported specially (`IJobCancellationToken` / `CancellationToken` is replaced at execution time).

**Code**
```csharp
public interface IReportService { Task GenerateAsync(int reportId, CancellationToken ct = default); }

public class ReportService(AppDb db) : IReportService
{
    [AutomaticRetry(Attempts = 3, DelaysInSeconds = [30, 120, 600])]
    [DisableConcurrentExecution(timeoutInSeconds: 600)]
    public async Task GenerateAsync(int reportId, CancellationToken ct = default)
    {
        var r = await db.Reports.FindAsync([reportId], ct);
        // idempotent processing
    }
}
```

### 8.3 [Medium] Quartz.NET: job, trigger, cron and misfire

**Answer**
In Quartz, an `IJob` holds the work and a trigger (simple or cron) defines when; the scheduler matches them. A *misfire* happens when a trigger could not fire on time (scheduler down/busy) and the misfire instruction decides whether to fire now, skip, or reschedule.

**Explanation**
Quartz cron expressions have **seconds** as the first field (e.g. `0 0/5 * * * ?` = every 5 minutes) and require `?` in day-of-month or day-of-week. Use `[DisallowConcurrentExecution]` to avoid overlapping runs of the same job key. For clustering, use AdoJobStore with `quartz.jobStore.clustered = true`; instances coordinate through DB locks, so each fire happens on one node.

**Code**
```csharp
using Quartz;

[DisallowConcurrentExecution]
public class CleanupJob(ILogger<CleanupJob> log) : IJob
{
    public Task Execute(IJobExecutionContext context)
    {
        log.LogInformation("Cleanup at {t}", DateTimeOffset.Now);
        return Task.CompletedTask;
    }
}

builder.Services.AddQuartz(q =>
{
    var key = new JobKey("cleanup");
    q.AddJob<CleanupJob>(o => o.WithIdentity(key));
    q.AddTrigger(t => t.ForJob(key).WithIdentity("cleanup-trigger")
        .WithCronSchedule("0 0/5 * * * ?", x => x.WithMisfireHandlingInstructionDoNothing()));
});
builder.Services.AddQuartzHostedService(o => o.WaitForJobsToComplete = true);
```

### 8.4 [Hard] When would you choose Hangfire vs Quartz vs a plain BackgroundService vs a queue?

**Answer**
Hangfire for ad-hoc persisted background work with retries and a dashboard; Quartz for complex calendar/cron scheduling and clustered schedules; `BackgroundService` for simple in-process loops; a message queue for decoupled, scalable, cross-service processing. They aren't mutually exclusive.

**Explanation**
Decision factors: durability (in-memory loses work), observability (Hangfire dashboard), schedule complexity (Quartz calendars/misfire policies), horizontal scaling (queue consumers scale out easily; recurring jobs need single-fire guarantees), operational cost (Hangfire needs DB polling; SQL storage uses distributed locks). Cloud alternatives: Azure Functions timer triggers, Kubernetes CronJob, Azure Container Apps jobs.

**Code**
```csharp
// Rule of thumb in code comments:
// - "Send email after signup"            -> Hangfire Enqueue (durable, retry)
// - "Every weekday 09:00 in tz X, skip holidays" -> Quartz CronTrigger + calendar
// - "Refresh cache every 30s"            -> BackgroundService + PeriodicTimer
// - "Process 10k orders/sec, multiple services" -> Service Bus / RabbitMQ consumers
RecurringJob.AddOrUpdate<IReportService>("weekday-9", s => s.GenerateAsync(1, default),
    "0 9 * * 1-5", new RecurringJobOptions { TimeZone = TimeZoneInfo.FindSystemTimeZoneById("UTC") });
```

### 8.5 [Tricky] A recurring job ran twice (or overlapped). Why, and how do you prevent it?

**Answer**
Causes: multiple app instances each running their own scheduler against in-memory stores, Quartz without clustering, a Hangfire worker crash mid-job (job re-queued after invisibility timeout → at-least-once), or a long job overlapping the next occurrence. Prevent with a shared persistent store/clustering, disable concurrent execution, and make jobs idempotent.

**Explanation**
Hangfire's `DisableConcurrentExecution` uses a distributed lock (SQL app lock) and *waits* the timeout then fails the job; it doesn't skip. For skip-if-running semantics, check a lock yourself (e.g., `sp_getapplock` with 0 timeout, Redis `SET NX`). Idempotency (e.g., unique constraint keyed by job run id) is the only real guarantee.

**Code**
```csharp
public async Task NightlyAsync(CancellationToken ct)
{
    var runKey = $"nightly:{DateTime.UtcNow:yyyyMMdd}";
    if (!await _locks.TryAcquireAsync(runKey, TimeSpan.FromHours(1), ct)) return; // another node/run owns it
    await ProcessAsync(ct);
}
```

**Interview traps**
- Passing a `DbContext`/entity as a Hangfire argument.
- Assuming recurring jobs run exactly once across instances with in-memory storage.
- Quartz cron ≠ standard cron (seconds field, `?`).
- Time zones/DST: recurring jobs default to UTC in Hangfire.

**Cheat sheet**
- Hangfire: persisted, dashboard, auto-retry 10×, args serialized.
- Quartz: Job + Trigger, cron with seconds, AdoJobStore clustering.
- Jobs must be idempotent (at-least-once).
- `[DisableConcurrentExecution]` / `[DisallowConcurrentExecution]`.
- Simple in-process loop → BackgroundService.

---

## Topic 9: Message Queues (RabbitMQ, Azure Service Bus)

**Study notes**

Queues decouple producers and consumers in time, scale and failure. Core concepts:

```
Producer -> [Exchange] --routing--> [Queue] -> Consumer(s)         (RabbitMQ)
Producer -> [Queue | Topic -> Subscriptions] -> Consumer(s)        (Service Bus)
                          failed N times -> Dead-Letter Queue (DLQ)
```

- **Delivery semantics**: at-most-once (ack before processing; may lose), **at-least-once** (ack after processing; may duplicate — the default practical choice), exactly-once (not truly achievable end-to-end; approximate with idempotent consumers + dedup).
- **Acknowledgement**: RabbitMQ `basic.ack/nack/reject`, unacked messages are redelivered when the channel closes; `prefetch` (QoS) limits in-flight messages. Service Bus: `PeekLock` (default; Complete/Abandon/DeadLetter/Defer) vs `ReceiveAndDelete`; lock duration (default 30 s, renewable), `MaxDeliveryCount` (default 10) → auto dead-letter.
- **Dead-letter**: messages that are rejected, expired (TTL), or exceed retries go to a DLQ (Service Bus built-in sub-queue `<queue>/$deadletterqueue`; RabbitMQ via DLX *dead-letter exchange* args).
- **Idempotent consumer**: store processed `MessageId` (inbox table) or make the operation naturally idempotent. **Outbox pattern**: write the message into the same DB transaction as the business change and publish from the outbox, avoiding dual-write inconsistency.
- **Ordering**: per-queue FIFO only with a single consumer; Service Bus **sessions** give ordered per-key processing; competing consumers break ordering. **Poison messages**: always fail → DLQ so they don't block.
- Abstractions: MassTransit, NServiceBus, Wolverine, CAP. RabbitMQ.Client 7.x is **async-only** (`CreateConnectionAsync`, `BasicPublishAsync`); 6.x had sync APIs.

### 9.1 [Easy] What is a message queue good for, and what's the difference between a queue and a topic?

**Answer**
A queue provides asynchronous, durable buffering between producer and consumer so work is decoupled, load-levelled, and retried independently. A queue delivers each message to one consumer (competing consumers); a topic delivers a copy to each subscription (pub/sub).

**Explanation**
Benefits: temporal decoupling, resilience, scale-out consumers, back-pressure. Costs: eventual consistency, operational complexity, duplicates, ordering challenges. In Service Bus, topics have subscriptions with filters; in RabbitMQ, exchanges (direct/topic/fanout/headers) route to queues.

**Code**
```csharp
using Azure.Messaging.ServiceBus;

await using var client = new ServiceBusClient("<connection-or-namespace>");
ServiceBusSender sender = client.CreateSender("orders");
await sender.SendMessageAsync(new ServiceBusMessage(BinaryData.FromString("{\"id\":1}"))
{
    MessageId = "order-1",                  // enables duplicate detection if enabled on the queue
    ContentType = "application/json"
});
```

### 9.2 [Follow-up] Explain at-least-once delivery and how a consumer should be written to cope with it

**Answer**
At-least-once means a message is acknowledged only after successful processing, so crashes or lock expiry cause redelivery and duplicates are possible. Consumers must be idempotent — safe to process the same message multiple times.

**Explanation**
Techniques: unique business keys with DB unique constraints, an inbox table `(MessageId, ProcessedAt)` checked in the same transaction as the side effect, upserts, version/ETag checks. Don't `ack` before the work completes unless losing the message is acceptable. Set processing time within the lock duration or renew locks (the Service Bus processor auto-renews up to `MaxAutoLockRenewalDuration`, default 5 min).

**Code**
```csharp
public async Task HandleAsync(OrderPlaced msg, string messageId, CancellationToken ct)
{
    await using var tx = await _db.Database.BeginTransactionAsync(ct);
    if (await _db.Inbox.AnyAsync(i => i.MessageId == messageId, ct)) return;   // duplicate -> ignore, then ack

    _db.Orders.Add(new Order { Id = msg.OrderId });
    _db.Inbox.Add(new InboxEntry { MessageId = messageId });
    await _db.SaveChangesAsync(ct);
    await tx.CommitAsync(ct);
}
```

### 9.3 [Medium] Write a Service Bus processor with complete / abandon / dead-letter

**Answer**
Create a `ServiceBusProcessor` with `AutoCompleteMessages = false`, handle `ProcessMessageAsync` and `ProcessErrorAsync`, and explicitly complete on success, dead-letter poison messages with a reason, and let transient exceptions abandon (retry) the message until `MaxDeliveryCount`.

**Explanation**
With `AutoCompleteMessages = true` (default) the SDK completes after your handler returns without throwing, and abandons if it throws. `MaxConcurrentCalls` controls parallelism. Dead-lettered messages are read with a receiver whose options use `SubQueue = SubQueue.DeadLetter`.

**Code**
```csharp
using Azure.Messaging.ServiceBus;

await using var client = new ServiceBusClient("<conn>");
await using var processor = client.CreateProcessor("orders", new ServiceBusProcessorOptions
{
    AutoCompleteMessages = false,
    MaxConcurrentCalls = 8,
    PrefetchCount = 16
});

processor.ProcessMessageAsync += async args =>
{
    try
    {
        var body = args.Message.Body.ToString();
        if (string.IsNullOrWhiteSpace(body))
        {
            await args.DeadLetterMessageAsync(args.Message, "EmptyBody", "Message had no payload");
            return;
        }
        // process...
        await args.CompleteMessageAsync(args.Message);
    }
    catch (TimeoutException)
    {
        await args.AbandonMessageAsync(args.Message);   // retry; DeliveryCount increments
    }
};
processor.ProcessErrorAsync += args => { Console.WriteLine(args.Exception.Message); return Task.CompletedTask; };

await processor.StartProcessingAsync();
```

### 9.4 [Medium] RabbitMQ producer/consumer with manual ack (client 7.x)

**Answer**
Declare a durable queue, publish persistent messages, consume with `autoAck: false`, set a prefetch via `BasicQosAsync`, and `BasicAckAsync` after processing. On failure `BasicNackAsync(requeue: false)` routes to a dead-letter exchange if configured.

**Explanation**
Auto-ack removes the message on delivery, so a crash loses it (at-most-once). Without a prefetch limit a fast broker floods a slow consumer. Requeue-on-failure without a retry cap creates infinite poison loops; use DLX + delayed retry. Publisher confirms (`ConfirmSelect`/tracking in 7.x via `CreateChannelOptions`) ensure the broker has persisted the message.

**Code**
```csharp
using RabbitMQ.Client;
using RabbitMQ.Client.Events;
using System.Text;

var factory = new ConnectionFactory { HostName = "localhost" };
await using var conn = await factory.CreateConnectionAsync();
await using var ch = await conn.CreateChannelAsync();

await ch.QueueDeclareAsync("orders", durable: true, exclusive: false, autoDelete: false,
    arguments: new Dictionary<string, object?> { ["x-dead-letter-exchange"] = "orders.dlx" });

await ch.BasicPublishAsync("", "orders", mandatory: false,
    basicProperties: new BasicProperties { Persistent = true },
    body: Encoding.UTF8.GetBytes("hello"));

await ch.BasicQosAsync(0, prefetchCount: 10, global: false);
var consumer = new AsyncEventingBasicConsumer(ch);
consumer.ReceivedAsync += async (_, ea) =>
{
    try
    {
        Console.WriteLine(Encoding.UTF8.GetString(ea.Body.Span));
        await ch.BasicAckAsync(ea.DeliveryTag, multiple: false);
    }
    catch
    {
        await ch.BasicNackAsync(ea.DeliveryTag, multiple: false, requeue: false); // -> DLX
    }
};
await ch.BasicConsumeAsync("orders", autoAck: false, consumer);
```

### 9.5 [Hard] The dual-write problem and the Outbox pattern

**Answer**
Saving to the database and publishing to a broker are two separate resources; if one succeeds and the other fails you get lost or phantom events. The outbox pattern writes the event to an `Outbox` table in the same DB transaction as the business change, and a separate relay publishes and marks it sent (at-least-once).

**Explanation**
Distributed transactions (MSDTC/2PC) are avoided in cloud messaging. The relay may publish twice if it crashes between publish and marking, so consumers must dedupe (Inbox). MassTransit/NServiceBus/Wolverine/CAP ship ready-made EF Core outbox implementations. Alternative: change-data-capture (Debezium).

**Code**
```csharp
public async Task PlaceOrderAsync(Order order, CancellationToken ct)
{
    _db.Orders.Add(order);
    _db.Outbox.Add(new OutboxMessage
    {
        Id = Guid.NewGuid(),
        Type = nameof(OrderPlaced),
        Payload = JsonSerializer.Serialize(new OrderPlaced(order.Id)),
        CreatedUtc = DateTime.UtcNow
    });
    await _db.SaveChangesAsync(ct);   // atomic: order + outbox row in one transaction
}
// A BackgroundService polls Outbox where ProcessedUtc IS NULL, publishes, then sets ProcessedUtc.
```

### 9.6 [Tricky] Poison messages, ordering and "exactly-once" — what are the real guarantees?

**Answer**
A poison message fails repeatedly and must be capped by max delivery count and dead-lettered, else it blocks throughput. FIFO ordering holds only with a single consumer (or Service Bus sessions/RabbitMQ single active consumer); "exactly-once" is really at-least-once plus idempotent processing.

**Explanation**
Service Bus has duplicate detection (by `MessageId` within a time window, 10 min default) for the sender side, and "exactly-once"-like transactional receive+send within a namespace, but not end-to-end with your database. Retry with delay: Service Bus `ScheduleMessageAsync` or re-send with `ScheduledEnqueueTime`; RabbitMQ TTL + DLX queues for delayed retry. Monitor DLQ depth and alert; build a replay tool.

**Code**
```csharp
await using var receiver = client.CreateReceiver("orders", new ServiceBusReceiverOptions
{
    SubQueue = SubQueue.DeadLetter
});
await foreach (var m in receiver.ReceiveMessagesAsync())
{
    Console.WriteLine($"{m.MessageId}: {m.DeadLetterReason} / {m.DeadLetterErrorDescription}");
    // fix & resubmit: await sender.SendMessageAsync(new ServiceBusMessage(m)); await receiver.CompleteMessageAsync(m);
}
```

**Interview traps**
- "Exactly-once delivery" claims — say at-least-once + idempotency.
- Auto-ack/ReceiveAndDelete loses messages on crash.
- Requeue forever on failure → hot poison loop.
- Publishing to the broker inside the DB transaction (dual-write) instead of outbox.
- Lock expires mid-processing → duplicate processing by another consumer.

**Cheat sheet**
- Ack after work; consumers idempotent.
- Service Bus: PeekLock, MaxDeliveryCount 10 → DLQ, sessions for order.
- RabbitMQ: durable queue + persistent msg + manual ack + prefetch + DLX.
- Outbox for publish-with-DB-commit; Inbox for dedupe.
- Monitor DLQ depth.

---

## Topic 10: SignalR

**Study notes**

SignalR is ASP.NET Core's real-time abstraction: the server calls client methods and clients call hub methods over a persistent connection, with automatic transport negotiation.

- **Transports** (best → fallback): **WebSockets**, **Server-Sent Events**, **Long Polling**. The client first POSTs `/hub/negotiate`, gets a connection id + available transports, then connects.
- **Hub**: class deriving from `Hub` (or `Hub<TClient>` for strongly typed client calls). A **new Hub instance is created per invocation** (transient) — don't keep state in fields. Per-connection data: `Context.ConnectionId`, `Context.UserIdentifier`, `Context.Items`.
- **Targets**: `Clients.All`, `Caller`, `Others`, `Client(id)`, `Group(name)`, `User(userId)`, `AllExcept(ids)`. **Groups** are server-side, in-memory (per server) and **not persisted**; membership is lost when the connection drops (a reconnect gets a *new* connection id → re-add to groups).
- Send from outside hubs with `IHubContext<THub>` (singleton, e.g. from controllers/background services).
- **Scale-out**: each server only knows its own connections. Use a **backplane** — Redis (`AddStackExchangeRedis`) — or **Azure SignalR Service** (offloads connections). With multiple servers and non-WebSocket transports you need **sticky sessions** (negotiate and subsequent requests must hit the same server), unless clients use WebSockets only with `SkipNegotiation = true`.
- Streaming: server→client `IAsyncEnumerable<T>`/`ChannelReader<T>`, client→server streams too.
- Client reconnect: `.WithAutomaticReconnect()` (default delays 0, 2, 10, 30 s then gives up).
- Auth: bearer token passed as `access_token` query string for WebSockets/SSE (browsers can't set headers) → configure `JwtBearerEvents.OnMessageReceived`.

### 10.1 [Easy] What is SignalR and when would you use it vs polling?

**Answer**
SignalR enables bi-directional real-time communication between server and clients (chat, dashboards, notifications) using WebSockets with automatic fallback to SSE/long polling. It beats polling by pushing updates immediately with far less overhead.

**Explanation**
It abstracts transport negotiation, reconnection, serialization (JSON or MessagePack) and user/group targeting. Use it when the server must push; use plain HTTP + caching when updates are rare.

**Code**
```csharp
using Microsoft.AspNetCore.SignalR;

public class ChatHub : Hub
{
    public async Task Send(string user, string message)
        => await Clients.All.SendAsync("ReceiveMessage", user, message);
}

// Program.cs
builder.Services.AddSignalR();
var app = builder.Build();
app.MapHub<ChatHub>("/chat");
```

### 10.2 [Follow-up] How do groups work and what happens to them on reconnect?

**Answer**
`Groups.AddToGroupAsync(Context.ConnectionId, "room1")` adds a connection to a named group; `Clients.Group("room1")` targets it. Groups are not persisted and tie to connection IDs, so after a reconnect (new connection ID) the client must rejoin.

**Explanation**
A group is created implicitly on first add and removed when empty; no API enumerates groups/members natively (maintain your own store if needed). Re-join in `OnConnectedAsync` from persisted data (e.g., DB of user rooms) or from the client after `onreconnected`.

**Code**
```csharp
public class RoomHub : Hub
{
    public Task Join(string room) => Groups.AddToGroupAsync(Context.ConnectionId, room);
    public Task Leave(string room) => Groups.RemoveFromGroupAsync(Context.ConnectionId, room);
    public Task Say(string room, string msg) => Clients.Group(room).SendAsync("Msg", Context.UserIdentifier, msg);

    public override async Task OnConnectedAsync()
    {
        // restore memberships (e.g., from DB) because groups don't survive reconnects
        await Groups.AddToGroupAsync(Context.ConnectionId, "lobby");
        await base.OnConnectedAsync();
    }
}

// .NET client
// var conn = new HubConnectionBuilder().WithUrl("https://host/room").WithAutomaticReconnect().Build();
// conn.Reconnected += async id => await conn.InvokeAsync("Join", "room1");
```

### 10.3 [Medium] Send messages from outside a hub, and use a strongly typed hub

**Answer**
Inject `IHubContext<THub>` (or `IHubContext<THub, TClient>`) into controllers or background services to push messages. A strongly typed `Hub<IChatClient>` replaces magic method-name strings with an interface, giving compile-time safety.

**Explanation**
`IHubContext` is a singleton and can't access `Context` (no caller). Hub methods are invoked on a transient hub per call, so inject scoped services in the constructor safely.

**Code**
```csharp
public interface IChatClient { Task ReceiveMessage(string user, string message); }

public class TypedChatHub : Hub<IChatClient>
{
    public Task Send(string user, string msg) => Clients.All.ReceiveMessage(user, msg);
}

public class Broadcaster(IHubContext<TypedChatHub, IChatClient> hub) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        using var t = new PeriodicTimer(TimeSpan.FromSeconds(5));
        while (await t.WaitForNextTickAsync(ct))
            await hub.Clients.All.ReceiveMessage("system", $"tick {DateTime.UtcNow:T}");
    }
}
```

### 10.4 [Hard] How do you scale SignalR across multiple servers?

**Answer**
Add a backplane so servers forward messages to each other — Redis (`AddStackExchangeRedis`) — or use Azure SignalR Service which terminates client connections for you. Also enable sticky sessions (ARR affinity) when not using WebSockets-only, because the negotiate request and the connection must reach the same node.

**Explanation**
Without a backplane, `Clients.All` only reaches clients connected to the local node. The Redis backplane publishes every send to Redis pub/sub channels (cost grows with message volume and servers). Azure SignalR Service scales to many thousands of connections and removes the need for sticky sessions on the app server. Connection state (e.g. `Context.Items`) is per server. Also configure keep-alive/timeouts (`KeepAliveInterval` 15 s, `ClientTimeoutInterval` 30 s defaults) and reverse proxy idle timeouts/WebSocket upgrade support.

**Code**
```csharp
builder.Services.AddSignalR(o =>
{
    o.KeepAliveInterval = TimeSpan.FromSeconds(15);
    o.ClientTimeoutInterval = TimeSpan.FromSeconds(30);
    o.MaximumReceiveMessageSize = 64 * 1024;       // default 32 KB
})
.AddStackExchangeRedis("localhost:6379", o => o.Configuration.ChannelPrefix = RedisChannel.Literal("myapp"));
// or: .AddAzureSignalR("<connection string>");

// WebSockets-only client (no sticky session needed):
// new HubConnectionBuilder().WithUrl(url, HttpTransportType.WebSockets | ..., o => { o.SkipNegotiation = true; o.Transports = HttpTransportType.WebSockets; })
```

**Gotchas / follow-ups**
- Redis backplane does not store messages — a node that's down simply misses them.
- `ServerTimeout`/`KeepAlive` mismatches cause spurious disconnects behind proxies.

### 10.5 [Tricky] Authentication over WebSockets and per-user targeting

**Answer**
Browsers can't set an `Authorization` header on WebSocket/SSE connections, so the JWT is sent as the `access_token` query parameter and extracted in `JwtBearerEvents.OnMessageReceived` for hub paths. `Clients.User(id)` targets all connections of a user based on `IUserIdProvider` (defaults to `ClaimTypes.NameIdentifier`).

**Explanation**
Query-string tokens can leak into logs — mask them and use short-lived tokens. Mark hubs with `[Authorize]`. Multiple tabs/devices = multiple connections for the same user; `Clients.User` hits all. If no NameIdentifier claim, `Context.UserIdentifier` is null and `Clients.User` goes nowhere.

**Code**
```csharp
builder.Services.AddAuthentication("Bearer").AddJwtBearer("Bearer", o =>
{
    o.Events = new JwtBearerEvents
    {
        OnMessageReceived = ctx =>
        {
            var token = ctx.Request.Query["access_token"];
            if (!string.IsNullOrEmpty(token) && ctx.HttpContext.Request.Path.StartsWithSegments("/chat"))
                ctx.Token = token;
            return Task.CompletedTask;
        }
    };
});

[Authorize]
public class SecureHub : Hub
{
    public Task Notify(string userId, string text) => Clients.User(userId).SendAsync("Notice", text);
}
```

**Interview traps**
- Storing per-user state in Hub fields (new instance per call).
- Assuming groups survive reconnects or app restarts.
- Forgetting the backplane/sticky sessions when scaling out.
- Blocking in hub methods (hub message processing for one connection is serialized by default — a slow method delays that client's next messages).

**Cheat sheet**
- Transports: WebSockets → SSE → Long Polling (negotiate first).
- Hub is transient; use `IHubContext` outside; `Hub<T>` for typing.
- Groups: server-side, in-memory, lost on reconnect.
- Scale-out: Redis backplane or Azure SignalR Service (+ sticky sessions unless WS-only).
- Token for WS via `access_token` query string.

---

## Topic 11: gRPC

**Study notes**

gRPC = contract-first RPC using **Protocol Buffers** (binary) over **HTTP/2**.

```
.proto  --protoc/Grpc.Tools-->  C# base class (server) + client stub
Client stub -> HTTP/2 stream (multiplexed on one connection) -> Server
```

| Call type | Proto | Use |
|---|---|---|
| Unary | `rpc Get(Req) returns (Res)` | like a normal request |
| Server streaming | `returns (stream Res)` | feeds/progress |
| Client streaming | `rpc Up(stream Req) returns (Res)` | uploads |
| Bidirectional | `stream` both | chat/telemetry |

- Field numbers identify fields on the wire — never reuse or change them; add new fields, `reserved` removed ones (backward/forward compat).
- Packages: server `Grpc.AspNetCore` (`AddGrpc()`, `MapGrpcService<T>()`); client `Grpc.Net.Client` (`GrpcChannel.ForAddress`), `Grpc.Net.ClientFactory` (`AddGrpcClient<T>`), `Grpc.Tools` (codegen).
- Errors: `RpcException` with `StatusCode` (e.g., `NotFound`, `DeadlineExceeded`, `Unavailable`); **deadlines** propagate; cancellation via `CancellationToken`.
- **vs REST**: faster/smaller, strongly typed, streaming, great for service-to-service; but not directly browser-callable (needs **gRPC-Web**), harder to debug/cache, less human-readable. **JSON transcoding** (.NET 7+) exposes gRPC services as REST endpoints.
- Load balancing: HTTP/2 multiplexes many calls on **one long-lived connection**, so L4 balancers pin all traffic to one server — use L7 (Envoy/YARP/ingress) or client-side load balancing.
- Channels are expensive and thread-safe: create once, reuse.

### 11.1 [Easy] What is gRPC and how does it differ from REST?

**Answer**
gRPC is a contract-first RPC framework using Protobuf over HTTP/2, producing compact binary messages, strongly typed clients and built-in streaming. REST is resource-oriented over HTTP/1.1 or 2 with usually JSON, easier for browsers and public APIs.

**Explanation**
Choose gRPC for internal microservice calls, low latency, polyglot teams, and streaming; choose REST for public/browser-facing APIs, caching and ease of exploration. Protobuf payloads are typically 3–10× smaller than JSON and faster to (de)serialize.

**Code**
```csharp
// greet.proto
// syntax = "proto3";
// option csharp_namespace = "Demo";
// service Greeter { rpc SayHello (HelloRequest) returns (HelloReply); }
// message HelloRequest { string name = 1; }
// message HelloReply   { string message = 1; }

public class GreeterService : Demo.Greeter.GreeterBase
{
    public override Task<Demo.HelloReply> SayHello(Demo.HelloRequest request, Grpc.Core.ServerCallContext ctx)
        => Task.FromResult(new Demo.HelloReply { Message = $"Hello {request.Name}" });
}

// Program.cs
builder.Services.AddGrpc();
app.MapGrpcService<GreeterService>();
```

### 11.2 [Follow-up] Show the client side and error/deadline handling

**Answer**
Create a `GrpcChannel` (reused), instantiate the generated client, and pass a `deadline` so calls don't hang; errors surface as `RpcException` with a `StatusCode`. Use `AddGrpcClient<T>` with the client factory in ASP.NET Core.

**Explanation**
Deadlines are absolute UTC times, propagated to the server (`ServerCallContext.Deadline`) and cancel the call when exceeded (`StatusCode.DeadlineExceeded`). Server throws `new RpcException(new Status(StatusCode.NotFound, "..."))` to return errors; add rich details via `Grpc.StatusProto`/metadata.

**Code**
```csharp
using Grpc.Core;
using Grpc.Net.Client;

using var channel = GrpcChannel.ForAddress("https://localhost:5001");
var client = new Demo.Greeter.GreeterClient(channel);
try
{
    var reply = await client.SayHelloAsync(new Demo.HelloRequest { Name = "Ann" },
        deadline: DateTime.UtcNow.AddSeconds(3));
    Console.WriteLine(reply.Message); // Output: Hello Ann
}
catch (RpcException ex) when (ex.StatusCode == StatusCode.DeadlineExceeded)
{
    Console.WriteLine("Too slow");
}

// DI:  builder.Services.AddGrpcClient<Demo.Greeter.GreeterClient>(o => o.Address = new Uri("https://svc"));
```

### 11.3 [Medium] Implement server streaming

**Answer**
Declare `returns (stream Res)` in the proto; the generated server method receives an `IServerStreamWriter<Res>` and writes messages with `WriteAsync` until done. The client consumes with `ResponseStream.ReadAllAsync`.

**Explanation**
Honour `context.CancellationToken` so the loop ends when the client disconnects or the deadline expires. Streams hold a connection and server resources; limit concurrency (Kestrel HTTP/2 `MaxStreamsPerConnection`, default 100).

**Code**
```csharp
// proto: rpc Ticks (TickRequest) returns (stream Tick);  message Tick { int32 n = 1; }

public class TickService : Demo.Ticker.TickerBase
{
    public override async Task Ticks(Demo.TickRequest req, IServerStreamWriter<Demo.Tick> stream, ServerCallContext ctx)
    {
        for (var i = 1; !ctx.CancellationToken.IsCancellationRequested; i++)
        {
            await stream.WriteAsync(new Demo.Tick { N = i });
            await Task.Delay(1000, ctx.CancellationToken);
        }
    }
}

// Client
using var call = client.Ticks(new Demo.TickRequest());
await foreach (var t in call.ResponseStream.ReadAllAsync())
    Console.WriteLine(t.N); // Output: 1, 2, 3, ...
```

### 11.4 [Hard] Why does gRPC behind a plain load balancer not balance, and how do you fix it?

**Answer**
HTTP/2 keeps one TCP connection and multiplexes all calls over it, so an L4 (connection-level) balancer sends every call to the same backend. Fix with an L7 proxy that understands HTTP/2 streams, or client-side load balancing (`dns:///` address + `ServiceConfig` with a `RoundRobinLoadBalancingConfig`, or custom resolver).

**Explanation**
In Kubernetes a ClusterIP service has the same problem; use a headless service with client-side round-robin, or a service mesh/ingress (Envoy, Linkerd, YARP). Client config below needs `Grpc.Net.Client` and a `dns:` scheme address so the built-in DNS resolver returns all pod IPs. Also set `SocketsHttpHandler.PooledConnectionLifetime` and `EnableMultipleHttp2Connections = true` to avoid the 100-stream limit bottleneck.

**Code**
```csharp
using Grpc.Net.Client;
using Grpc.Net.Client.Configuration;

var channel = GrpcChannel.ForAddress("dns:///my-service:5001", new GrpcChannelOptions
{
    ServiceConfig = new ServiceConfig { LoadBalancingConfigs = { new RoundRobinConfig() } },
    HttpHandler = new SocketsHttpHandler
    {
        PooledConnectionIdleTimeout = Timeout.InfiniteTimeSpan,
        KeepAlivePingDelay = TimeSpan.FromSeconds(60),
        KeepAlivePingTimeout = TimeSpan.FromSeconds(30),
        EnableMultipleHttp2Connections = true
    }
});
```

**Gotchas / follow-ups**
- Without keep-alive pings, idle connections may be dropped silently by intermediaries.
- gRPC retry policy is available via `ServiceConfig` (`RetryPolicy`) — only for idempotent status codes you list.

### 11.5 [Tricky] Can a browser call gRPC? And how do you evolve a proto safely?

**Answer**
Browsers can't use raw gRPC (no control over HTTP/2 framing/trailers), so use **gRPC-Web** (`Grpc.AspNetCore.Web`, `app.UseGrpcWeb()`, `.EnableGrpcWeb()`) or JSON transcoding. Evolve protos by only adding new fields with new numbers and `reserved`-ing removed ones; never renumber or change types.

**Explanation**
gRPC-Web doesn't support client or bidirectional streaming (server streaming works). Unknown fields are preserved/ignored by old parsers, giving forward compatibility; default values (0, "") are indistinguishable from unset in proto3 unless you use `optional` or wrapper types. Renaming a field is wire-safe (only number matters) but breaks JSON.

**Code**
```csharp
// message User {
//   reserved 3; reserved "legacy_name";
//   int32 id = 1;
//   string name = 2;
//   optional string nickname = 4;   // 'optional' gives HasNickname presence
// }
var user = new Demo.User { Id = 1, Name = "Ann" };
Console.WriteLine(user.HasNickname); // Output: False
```

**Interview traps**
- Creating a new `GrpcChannel` per call (connection churn).
- Reusing/renumbering proto field numbers.
- Forgetting deadlines → hung calls.
- Expecting L4 load balancers to spread calls.
- Using gRPC for public browser APIs without gRPC-Web.

**Cheat sheet**
- Proto3 + HTTP/2; 4 call types.
- Field numbers are the contract; add, don't change.
- Reuse channels; always set deadlines.
- Needs L7/client-side LB; gRPC-Web for browsers.
- Errors via `RpcException`/`StatusCode`.

---

## Topic 12: WebSockets & Server-Sent Events

**Study notes**

| | WebSockets | SSE (Server-Sent Events) | Long polling |
|---|---|---|---|
| Direction | Full duplex | Server → client only | Simulated push |
| Protocol | Upgrade from HTTP/1.1 (`101 Switching Protocols`), then framed TCP | Plain HTTP response `text/event-stream`, kept open | Repeated HTTP requests |
| Reconnect | Manual | **Built-in** `EventSource` auto-reconnect + `Last-Event-ID` | Manual |
| Data | Text/binary | UTF-8 text | Any |
| Browser API | `WebSocket` | `EventSource` (GET only, no custom headers) | fetch |
| Proxies | Need upgrade support | Friendly (plain HTTP), but buffering must be disabled | Friendly |

- ASP.NET Core raw WebSockets: `app.UseWebSockets()`, then `context.WebSockets.IsWebSocketRequest` → `AcceptWebSocketAsync()`; you loop `ReceiveAsync` until `Close`, and **you must keep the request pipeline alive** (don't return from the middleware while the socket is open). You handle framing, reconnects, heartbeats, auth, scaling yourself — SignalR does this for you.
- SSE message format: lines `event: name`, `data: payload`, optional `id: n`, `retry: ms`, terminated by a blank line. Response headers: `Content-Type: text/event-stream`, `Cache-Control: no-cache`. Flush after each event.
- Each connection holds a Kestrel connection/thread-free async loop; limit & monitor connection counts. Over HTTP/1.1 browsers cap ~6 connections per domain (SSE counts), HTTP/2 multiplexes.

### 12.1 [Easy] When do you pick WebSockets vs SSE?

**Answer**
Use WebSockets when you need two-way, low-latency messaging (games, chat, collaborative editing). Use SSE when only the server pushes updates (notifications, live feeds) because it is simpler, runs over plain HTTP and auto-reconnects.

**Explanation**
SSE is text-only and unidirectional but gets proxy/CDN/HTTP2 compatibility and `EventSource` automatic retry with `Last-Event-ID` resume. SignalR can use either under the hood, adding groups/users/reconnect semantics.

**Code**
```csharp
// Browser (comment): const es = new EventSource('/events'); es.onmessage = e => console.log(e.data);
app.MapGet("/events", async (HttpContext ctx, CancellationToken ct) =>
{
    ctx.Response.Headers.ContentType = "text/event-stream";
    ctx.Response.Headers.CacheControl = "no-cache";
    for (var i = 1; !ct.IsCancellationRequested; i++)
    {
        await ctx.Response.WriteAsync($"id: {i}\ndata: tick {i}\n\n", ct);
        await ctx.Response.Body.FlushAsync(ct);
        await Task.Delay(1000, ct);
    }
});
```

### 12.2 [Follow-up] Implement an echo WebSocket endpoint in ASP.NET Core

**Answer**
Enable `UseWebSockets`, accept the upgrade, then loop `ReceiveAsync` into a buffer and `SendAsync` the data back until the client sends a Close frame, completing the close handshake.

**Explanation**
`ReceiveAsync` may return partial messages (`EndOfMessage == false`); real code accumulates fragments. Always honour the `CancellationToken` and call `CloseAsync`/`CloseOutputAsync`. The middleware must not complete before the socket closes or the connection is aborted. Configure `KeepAliveInterval` (default 2 min in ASP.NET Core 8) and `AllowedOrigins` for origin checks (WebSockets aren't covered by CORS).

**Code**
```csharp
using System.Net.WebSockets;

app.UseWebSockets(new WebSocketOptions { KeepAliveInterval = TimeSpan.FromSeconds(30) });

app.Map("/ws", async ctx =>
{
    if (!ctx.WebSockets.IsWebSocketRequest) { ctx.Response.StatusCode = 400; return; }
    using var ws = await ctx.WebSockets.AcceptWebSocketAsync();
    var buffer = new byte[4096];

    var result = await ws.ReceiveAsync(buffer, ctx.RequestAborted);
    while (!result.CloseStatus.HasValue)
    {
        await ws.SendAsync(buffer.AsMemory(0, result.Count), result.MessageType, result.EndOfMessage, ctx.RequestAborted);
        result = await ws.ReceiveAsync(buffer, ctx.RequestAborted);
    }
    await ws.CloseAsync(result.CloseStatus.Value, result.CloseStatusDescription, CancellationToken.None);
});
```

### 12.3 [Medium] How does SSE resume after a disconnect, and what are its limits?

**Answer**
The browser's `EventSource` reconnects automatically (default ~3 s, overridable with `retry:`) and sends the last received `id` in the `Last-Event-ID` header so the server can replay missed events. Limits: GET only, no custom headers (use cookies/query token), text only, and connection caps over HTTP/1.1.

**Explanation**
You must buffer recent events server-side (per stream) to support replay. Proxies that buffer responses (nginx `proxy_buffering on`) break SSE — disable buffering (`X-Accel-Buffering: no`). Send periodic comments (`: keepalive\n\n`) to keep idle connections open through proxies.

**Code**
```csharp
app.MapGet("/feed", async (HttpContext ctx, CancellationToken ct) =>
{
    ctx.Response.Headers.ContentType = "text/event-stream";
    ctx.Response.Headers.CacheControl = "no-cache";
    ctx.Response.Headers["X-Accel-Buffering"] = "no";

    int.TryParse(ctx.Request.Headers["Last-Event-ID"], out var last);
    for (var id = last + 1; id < last + 4; id++)     // replay/produce next events
        await ctx.Response.WriteAsync($"id: {id}\nevent: update\ndata: {{\"n\":{id}}}\nretry: 5000\n\n", ct);

    await ctx.Response.WriteAsync(": keepalive\n\n", ct);
    await ctx.Response.Body.FlushAsync(ct);
});
```

### 12.4 [Hard] Scaling and operating long-lived connections

**Answer**
Each open connection costs memory (buffers, state) and a file descriptor but no thread when you use async I/O; scale by limiting per-connection buffers, setting Kestrel `MaxConcurrentConnections`/`MaxConcurrentUpgradedConnections`, tuning OS limits, and using a message backplane (Redis pub/sub) so any node can broadcast to all sockets.

**Explanation**
Upgraded connections (WebSockets) aren't counted in `MaxConcurrentConnections`; they use `MaxConcurrentUpgradedConnections`. Load balancers need sticky/idle timeouts compatible with heartbeats. Deploys drop all sockets → clients need jittered reconnect backoff to avoid a reconnect storm. Track connection count, message rate and backlog via metrics. In practice, prefer SignalR (or Azure Web PubSub) rather than hand-rolling scale-out.

**Code**
```csharp
builder.WebHost.ConfigureKestrel(k =>
{
    k.Limits.MaxConcurrentConnections = 20_000;
    k.Limits.MaxConcurrentUpgradedConnections = 20_000;   // WebSockets
    k.Limits.KeepAliveTimeout = TimeSpan.FromMinutes(2);
});
```

### 12.5 [Tricky] "Why does my WebSocket/SSE endpoint close immediately or stop streaming?"

**Answer**
Common causes: the middleware returns before the socket loop ends (pipeline completes → connection closed), response buffering/compression middleware holds SSE data, a proxy/IIS timeout or missing upgrade support, or CORS/origin mismatch. Fix by awaiting the loop, flushing each SSE write, excluding `text/event-stream` from compression, and aligning proxy timeouts.

**Explanation**
Response compression buffers by default; add `Flush` or exclude the MIME type (`text/event-stream` is not in default MIME list so it's fine unless you add it). IIS needs the WebSocket Protocol feature installed. Azure App Service requires WebSockets enabled in settings, and Front Door/App Gateway have idle timeouts (~230 s on Azure LB/App Service front end). Use heartbeat messages.

**Code**
```csharp
// WRONG: returns immediately -> socket is disposed
app.Map("/ws-bad", async ctx =>
{
    var ws = await ctx.WebSockets.AcceptWebSocketAsync();
    _ = Task.Run(() => Echo(ws));      // fire-and-forget; middleware completes, connection torn down
});

// RIGHT: await the loop inside the request delegate
app.Map("/ws-good", async ctx =>
{
    using var ws = await ctx.WebSockets.AcceptWebSocketAsync();
    await Echo(ws);
});
static Task Echo(WebSocket ws) => Task.CompletedTask;
```

**Interview traps**
- WebSockets are not subject to CORS — validate `Origin` yourself.
- Assuming SSE supports sending data/headers from browser.
- Forgetting heartbeats → proxies kill idle connections.
- Reconnect storms after deploys without jitter.

**Cheat sheet**
- WS = duplex, framed after HTTP upgrade (101); SSE = server→client text over HTTP.
- SSE: `text/event-stream`, `data:`/`id:`/`event:`/`retry:`, blank line ends event, flush.
- `UseWebSockets` → `AcceptWebSocketAsync` → receive loop (keep request alive).
- Backplane + heartbeats + jittered reconnect for scale.
- Prefer SignalR for real apps.

---

## Topic 13: Testing Pyramid, xUnit/NUnit & Mocking

**Study notes**

```
        /\        E2E / UI  (few, slow, brittle)           Playwright, Selenium
       /  \       Integration / API (some)                  WebApplicationFactory, Testcontainers
      /----\      Unit (many, fast, isolated)               xUnit/NUnit + mocks
```
Unit tests: milliseconds, no I/O. Integration: real wiring/DB. E2E: whole system. A "testing trophy" variant weights integration tests more heavily for web APIs. Good tests follow **AAA** (Arrange-Act-Assert), are deterministic, independent, and named for behaviour (`Method_Scenario_Expected`).

**xUnit**
- A **new test-class instance is created per test** → constructor = per-test setup, `IDisposable.Dispose` = teardown (async: `IAsyncLifetime`).
- `[Fact]` = parameterless test; `[Theory]` + `[InlineData]`/`[MemberData]`/`[ClassData]` = data-driven.
- **Fixtures** share expensive setup: `IClassFixture<T>` (once per class), `ICollectionFixture<T>` + `[Collection("name")]` (once across classes in a collection).
- Parallelism: **different collections run in parallel; tests within a collection run sequentially.** Each class is its own implicit collection.
- `Assert.Equal(expected, actual)` (expected first), `Assert.Throws<T>` / `ThrowsAsync<T>`.

**NUnit differences**: `[Test]`, `[TestCase]`, `[SetUp]`/`[TearDown]` (per test), `[OneTimeSetUp]`/`[OneTimeTearDown]` (per fixture), **one fixture instance shared by all tests** (so fields persist across tests unless reset), constraint model `Assert.That(actual, Is.EqualTo(expected))`, parallel execution is opt-in (`[Parallelizable]`). MSTest: `[TestMethod]`, `[DataRow]`.

**Test doubles**: *dummy* (placeholder), *stub* (canned answers), *fake* (working lightweight impl), *spy* (records calls), *mock* (verifies interactions). Libraries: **Moq**, **NSubstitute**, FakeItEasy; for time use `TimeProvider`/`FakeTimeProvider`.

### 13.1 [Easy] What is the testing pyramid and what belongs at each level?

**Answer**
The pyramid says to have many fast unit tests, fewer integration tests, and very few end-to-end tests. Unit tests verify logic in isolation, integration tests verify components work together (DB, HTTP pipeline), and E2E tests verify user journeys.

**Explanation**
Lower levels give faster feedback and pinpoint failures; higher levels give more confidence but are slower and flakier. For EF Core/ASP.NET apps, integration tests against a real DB usually give more value than heavily mocked unit tests of thin controllers/repositories.

**Code**
```csharp
using Xunit;

public class PriceCalculator
{
    public decimal Total(decimal price, int qty, decimal discount = 0m) => price * qty * (1 - discount);
}

public class PriceCalculatorTests
{
    [Fact]
    public void Total_AppliesDiscount()
    {
        var sut = new PriceCalculator();               // Arrange
        var result = sut.Total(10m, 3, 0.1m);          // Act
        Assert.Equal(27m, result);                     // Assert
    }
}
```

### 13.2 [Follow-up] `[Fact]` vs `[Theory]`, and the ways to supply data

**Answer**
`[Fact]` is a single fixed test; `[Theory]` runs once per data row. Data comes from `[InlineData]` (constants), `[MemberData]` (static member returning rows), or `[ClassData]`; use `TheoryData<T>` for compile-time type safety.

**Explanation**
Inline data is limited to compile-time constants (no `DateTime`, objects). `MemberData` can return complex objects. Each row displays as a separate test; override display names with `DisplayName`.

**Code**
```csharp
public class MathTests
{
    [Theory]
    [InlineData(2, 3, 5)]
    [InlineData(-1, 1, 0)]
    public void Add(int a, int b, int expected) => Assert.Equal(expected, a + b);

    public static TheoryData<string, bool> Emails => new()
    {
        { "a@b.com", true },
        { "bad", false }
    };

    [Theory]
    [MemberData(nameof(Emails))]
    public void IsEmail(string input, bool expected) => Assert.Equal(expected, input.Contains('@'));
}
```

### 13.3 [Medium] xUnit lifecycle and fixtures: constructor vs IClassFixture vs ICollectionFixture

**Answer**
xUnit creates a new instance of the test class for every test, so the constructor and `Dispose` run per test. For expensive shared state (DB container, web app), use `IClassFixture<T>` (shared by tests in one class) or `ICollectionFixture<T>` with `[Collection]` (shared across classes).

**Explanation**
Because instances are per-test, fields don't leak state between tests (unlike NUnit). Fixture constructors run once; implement `IAsyncLifetime` for async init. Tests in the same collection never run in parallel, which is also a way to serialize tests that share a DB.

**Code**
```csharp
public class DbFixture : IAsyncLifetime
{
    public string ConnectionString { get; private set; } = "";
    public Task InitializeAsync() { ConnectionString = "Server=...;"; return Task.CompletedTask; }  // once
    public Task DisposeAsync() => Task.CompletedTask;
}

[CollectionDefinition("db")]
public class DbCollection : ICollectionFixture<DbFixture>;

[Collection("db")]
public class OrderTests(DbFixture fx)           // fixture injected; class instance still new per test
{
    private readonly Guid _id = Guid.NewGuid(); // runs per test
    [Fact] public void Uses_shared_fixture() => Assert.NotNull(fx.ConnectionString);
}
```

### 13.4 [Medium] Mocking with Moq vs NSubstitute, and what to verify

**Answer**
Both create runtime proxies of interfaces/virtual members: Moq uses `Setup(...).Returns(...)` and `Verify(...)`; NSubstitute uses `sub.Method().Returns(...)` and `sub.Received()`. Mock outward dependencies (email, clock, repository) and assert on **outcomes** first, interactions only when the interaction *is* the behaviour.

**Explanation**
Over-mocking couples tests to implementation and makes refactoring painful. Don't mock types you don't own (e.g. `DbContext`, `HttpClient` — use a fake `HttpMessageHandler` or real DB). Moq cannot mock non-virtual/sealed members or static methods; NSubstitute same limitations (use wrapper interfaces). Note: Moq 4.20.0 briefly shipped the SponsorLink analyzer, causing community controversy — many teams pinned older versions or moved to NSubstitute.

**Code**
```csharp
using Moq;
using NSubstitute;

public interface IEmailSender { Task SendAsync(string to, string body); }
public interface IUserRepo { Task<string?> GetEmailAsync(int id); }

public class Notifier(IUserRepo repo, IEmailSender email)
{
    public async Task<bool> NotifyAsync(int id)
    {
        var to = await repo.GetEmailAsync(id);
        if (to is null) return false;
        await email.SendAsync(to, "hi");
        return true;
    }
}

public class NotifierTests
{
    [Fact]
    public async Task Moq_Version()
    {
        var repo = new Mock<IUserRepo>();
        repo.Setup(r => r.GetEmailAsync(1)).ReturnsAsync("a@b.com");
        var email = new Mock<IEmailSender>();

        Assert.True(await new Notifier(repo.Object, email.Object).NotifyAsync(1));
        email.Verify(e => e.SendAsync("a@b.com", "hi"), Times.Once);
    }

    [Fact]
    public async Task NSubstitute_Version()
    {
        var repo = Substitute.For<IUserRepo>();
        repo.GetEmailAsync(2).Returns((string?)null);
        var email = Substitute.For<IEmailSender>();

        Assert.False(await new Notifier(repo, email).NotifyAsync(2));
        await email.DidNotReceive().SendAsync(Arg.Any<string>(), Arg.Any<string>());
    }
}
```

### 13.5 [Hard] How does NUnit differ from xUnit in lifecycle and shared state — and what bug does that cause?

**Answer**
NUnit creates **one instance of the fixture class** (by default) and runs all its tests on it, so instance fields persist between tests; `[SetUp]` must reset state. xUnit creates a new instance per test, so state is naturally isolated. Migrating tests between frameworks can therefore expose hidden shared-state bugs or order dependencies.

**Explanation**
NUnit mapping: `[SetUp]` ≈ xUnit constructor, `[TearDown]` ≈ `Dispose`, `[OneTimeSetUp]` ≈ `IClassFixture` ctor. NUnit's `[TestFixtureSource]`/`[TestCaseSource]` vs xUnit `MemberData`. NUnit's `[Parallelizable(ParallelScope.All)]` opt-in vs xUnit parallel-by-collection default. NUnit 3.x offers `[FixtureLifeCycle(LifeCycle.InstancePerTestCase)]` to emulate xUnit.

**Code**
```csharp
using NUnit.Framework;

[TestFixture]
public class CounterTests
{
    private int _count;                       // shared across tests unless reset

    [SetUp] public void Reset() => _count = 0;

    [Test] public void A() { _count++; Assert.That(_count, Is.EqualTo(1)); }
    [Test] public void B() { _count++; Assert.That(_count, Is.EqualTo(1)); }   // passes only due to [SetUp]

    [TestCase(1, 2, 3)]
    [TestCase(0, 0, 0)]
    public void Add(int a, int b, int exp) => Assert.That(a + b, Is.EqualTo(exp));
}
```

**Gotchas / follow-ups**
- xUnit parallelism + shared static state or shared DB → flaky tests; disable with `[assembly: CollectionBehavior(DisableTestParallelization = true)]` or use collections.
- `async void` tests are not awaited — always `async Task`.

### 13.6 [Tricky] Testing time, randomness and async code deterministically

**Answer**
Abstract non-determinism: inject `TimeProvider` (built in since .NET 8) and use `FakeTimeProvider` from `Microsoft.Extensions.TimeProvider.Testing` to control the clock and timers. Never use `Thread.Sleep`/`Task.Delay` in tests to "wait"; await the operation or advance the fake clock.

**Explanation**
`TimeProvider.GetUtcNow()` replaces `DateTime.UtcNow`; `TimeProvider.CreateTimer` and `Task.Delay(TimeSpan, TimeProvider)` are controllable. `FakeTimeProvider.Advance` fires timers synchronously. For randomness inject `Random`/seed it. Test exceptions with `await Assert.ThrowsAsync<T>(() => ...)`.

**Code**
```csharp
using Microsoft.Extensions.Time.Testing;

public class TokenService(TimeProvider clock)
{
    private readonly DateTimeOffset _issued = clock.GetUtcNow();
    public bool IsExpired() => clock.GetUtcNow() - _issued > TimeSpan.FromMinutes(30);
}

public class TokenTests
{
    [Fact]
    public void Expires_after_30_minutes()
    {
        var clock = new FakeTimeProvider(new DateTimeOffset(2025, 1, 1, 0, 0, 0, TimeSpan.Zero));
        var sut = new TokenService(clock);
        Assert.False(sut.IsExpired());
        clock.Advance(TimeSpan.FromMinutes(31));
        Assert.True(sut.IsExpired());
    }
}
```

**Interview traps**
- Mocking everything → tests only prove the mocks (and break on refactor).
- Asserting on `ToString()`/order of unordered results.
- Shared static state with parallel xUnit collections.
- `Assert.Equal(actual, expected)` argument order (messages become confusing).
- `async void` test methods.

**Cheat sheet**
- Pyramid: many unit, some integration, few E2E.
- xUnit: new instance per test; fixtures for sharing; collections serialize.
- NUnit: one fixture instance; `[SetUp]` resets state.
- Mock outward deps; verify behaviour, not implementation.
- `TimeProvider` + `FakeTimeProvider` for time.

---

## Topic 14: Integration Testing & Testing EF Core

**Study notes**

**WebApplicationFactory<TEntryPoint>** (`Microsoft.AspNetCore.Mvc.Testing`) boots your real app in-memory with **TestServer** (no sockets) and gives `CreateClient()`. With minimal APIs/top-level statements, expose the entry point: `public partial class Program { }`. Override services with `builder.ConfigureTestServices(...)` (runs after the app's registrations) or `ConfigureServices`; override config with `UseSetting`/`ConfigureAppConfiguration`; set environment with `UseEnvironment("Testing")`.

**Testcontainers** (`Testcontainers.MsSql`, `Testcontainers.PostgreSql`, …) spins up real Docker containers per test run (needs Docker). Use `IAsyncLifetime` fixtures; **Respawn** resets data between tests quickly.

**Testing EF Core — which provider?**

| Option | Pros | Cons |
|---|---|---|
| `UseInMemoryDatabase` | Fast, no setup | **Not relational**: no FK/unique constraints enforcement, no transactions, no raw SQL, different query semantics (client eval, case-sensitivity). Microsoft advises against it for testing providers. |
| SQLite in-memory | Relational, constraints, transactions, fast | Different SQL dialect/types (no `decimal` precision, limited `ALTER`, no schemas, collation differences); keep the connection open or the DB vanishes |
| Real DB (Testcontainers / LocalDB) | Highest fidelity (same engine, collations, SQL Server functions, concurrency tokens) | Slower, needs Docker/infra |
| Mock `DbContext`/repository | Fast for pure logic | Doesn't test queries/mapping |

### 14.1 [Easy] What is WebApplicationFactory and why use it?

**Answer**
`WebApplicationFactory<TEntryPoint>` hosts your actual ASP.NET Core app in-process using TestServer, and gives an `HttpClient` that calls it without network. It lets you test routing, middleware, filters, serialization and DI end-to-end quickly.

**Explanation**
Add `public partial class Program { }` at the bottom of `Program.cs` so the test project can reference it. Implement `IClassFixture<WebApplicationFactory<Program>>` to share the factory per class (the host is built once per factory instance).

**Code**
```csharp
using Microsoft.AspNetCore.Mvc.Testing;

public class ApiTests(WebApplicationFactory<Program> factory) : IClassFixture<WebApplicationFactory<Program>>
{
    [Fact]
    public async Task Health_returns_200()
    {
        var client = factory.CreateClient();
        var resp = await client.GetAsync("/health");
        Assert.Equal(System.Net.HttpStatusCode.OK, resp.StatusCode);
    }
}
// Program.cs ends with:  public partial class Program { }
```

### 14.2 [Follow-up] Replace services (DB, external HTTP, auth) for tests

**Answer**
Subclass the factory (or use `WithWebHostBuilder`) and call `ConfigureTestServices` to remove the production registration and add a test one — typically swap the `DbContext` to a test database and replace external clients with fakes. Auth is usually replaced with a test authentication handler.

**Explanation**
Remove the existing `DbContextOptions<AppDb>` descriptor before re-adding; otherwise both registrations apply (in EF Core 9 also remove `IDbContextOptionsConfiguration<AppDb>`). Fake external services with in-process fakes or WireMock.Net rather than hitting the internet.

**Code**
```csharp
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Mvc.Testing;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.DependencyInjection.Extensions;

public class TestFactory(string connectionString) : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.UseEnvironment("Testing");
        builder.ConfigureTestServices(services =>
        {
            services.RemoveAll<DbContextOptions<AppDb>>();
            services.AddDbContext<AppDb>(o => o.UseSqlServer(connectionString));
            services.RemoveAll<IEmailSender>();
            services.AddSingleton<IEmailSender, FakeEmailSender>();
        });
    }
}
```

### 14.3 [Medium] Use Testcontainers for a real SQL Server in tests

**Answer**
Start an `MsSqlContainer` in an `IAsyncLifetime` fixture, pass its connection string to the app (via the factory) and run migrations; the container is destroyed afterwards. This tests against the real engine with real constraints and SQL translation.

**Explanation**
Container start is slow (several seconds, SQL Server image is large) so share it per collection/assembly, not per test, and reset data between tests with **Respawn** or transactions. CI agents need Docker. Ryuk (resource reaper) cleans up leftover containers. Pin the image tag.

**Code**
```csharp
using Testcontainers.MsSql;

public class SqlFixture : IAsyncLifetime
{
    private readonly MsSqlContainer _sql = new MsSqlBuilder()
        .WithImage("mcr.microsoft.com/mssql/server:2022-latest")
        .Build();

    public TestFactory Factory { get; private set; } = default!;
    public string ConnectionString => _sql.GetConnectionString();
    public async Task InitializeAsync()
    {
        await _sql.StartAsync();
        Factory = new TestFactory(_sql.GetConnectionString());
        using var scope = Factory.Services.CreateScope();
        await scope.ServiceProvider.GetRequiredService<AppDb>().Database.MigrateAsync();
    }

    public async Task DisposeAsync()
    {
        await Factory.DisposeAsync();
        await _sql.DisposeAsync();
    }
}
```

### 14.4 [Medium] EF Core InMemory vs SQLite vs real DB — which and why?

**Answer**
Prefer a real database (Testcontainers) for repository/query tests; use SQLite in-memory when Docker isn't available and you only need relational behaviour; avoid the InMemory provider for anything that depends on constraints, transactions or SQL translation. InMemory can give false confidence because it isn't a relational database.

**Explanation**
InMemory doesn't enforce FKs/uniqueness, ignores `FromSql`, `ExecuteUpdate/Delete` limitations, and treats things like case-sensitivity/ordering differently than SQL Server. SQLite in-memory is relational but its type system differs (e.g., `decimal` stored as TEXT/REAL; `DateTimeOffset` ordering/translation limitations). If using SQLite in-memory you must open the connection manually and keep it open for the DB's lifetime.

**Code**
```csharp
using Microsoft.Data.Sqlite;
using Microsoft.EntityFrameworkCore;

using var conn = new SqliteConnection("DataSource=:memory:");
conn.Open();                                       // DB lives as long as this connection is open

var options = new DbContextOptionsBuilder<AppDb>().UseSqlite(conn).Options;
using (var db = new AppDb(options)) db.Database.EnsureCreated();

using (var db = new AppDb(options))
{
    db.Customers.Add(new Customer { Email = "a@b.com" });
    db.Customers.Add(new Customer { Email = "a@b.com" });       // unique index defined in OnModelCreating
    try { db.SaveChanges(); }
    catch (DbUpdateException) { Console.WriteLine("Unique violation caught"); } // InMemory provider would NOT throw
}
```

### 14.5 [Hard] Keeping integration tests isolated, fast and non-flaky

**Answer**
Isolate data per test using Respawn (reset to empty), a transaction rolled back at the end, or unique tenant/keys; share the container and factory across tests; and run tests that mutate shared state serially. Avoid depending on execution order or on `Thread.Sleep` for eventual consistency.

**Explanation**
Per-test transactions don't work if the code under test commits its own transaction or runs through a separate HTTP-hosted app instance using a different connection; Respawn works regardless. Seed minimal data in each test (builders/object mothers) instead of a giant shared seed. Run migrations once. For background processing, poll with a timeout (`Polly`/custom `Eventually`) rather than fixed sleeps. In CI, cache images and run in parallel by database-per-test-class if needed.

**Code**
```csharp
using Respawn;

public class DbReset(string connectionString)
{
    private Respawner? _respawner;

    public async Task ResetAsync()
    {
        await using var conn = new Microsoft.Data.SqlClient.SqlConnection(connectionString);
        await conn.OpenAsync();
        _respawner ??= await Respawner.CreateAsync(conn, new RespawnerOptions
        {
            TablesToIgnore = [new Respawn.Graph.Table("__EFMigrationsHistory")]
        });
        await _respawner.ResetAsync(conn);
    }
}
// In a test class: IAsyncLifetime.InitializeAsync => await new DbReset(fixture.ConnectionString).ResetAsync();
```

**Gotchas / follow-ups**
- `WebApplicationFactory` runs the app's startup, so hosted services (e.g. Hangfire/consumers) start in tests — disable them in `ConfigureTestServices`.
- `CreateClient()` follows redirects by default; set `WebApplicationFactoryClientOptions { AllowAutoRedirect = false }` to assert 302s.

### 14.6 [Tricky] Test authentication and the "test passes locally but fails in CI" trap

**Answer**
Register a fake authentication scheme (`AddAuthentication("Test").AddScheme<AuthenticationSchemeOptions, TestAuthHandler>`) so tests can authenticate without a real IdP. CI-only failures usually come from Docker not available, port/timing assumptions, time zone/culture differences, or shared state with parallel runs.

**Explanation**
Make the default scheme the test one in `ConfigureTestServices`. Pin culture/time zone-sensitive assertions (`CultureInfo.InvariantCulture`), avoid file-path assumptions (`/` vs `\`), and avoid ordering dependencies. Print container logs on failure for diagnosis.

**Code**
```csharp
using Microsoft.AspNetCore.Authentication;
using Microsoft.Extensions.Options;
using System.Security.Claims;
using System.Text.Encodings.Web;

public class TestAuthHandler(IOptionsMonitor<AuthenticationSchemeOptions> o, ILoggerFactory l, UrlEncoder e)
    : AuthenticationHandler<AuthenticationSchemeOptions>(o, l, e)
{
    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        var identity = new ClaimsIdentity([new Claim(ClaimTypes.NameIdentifier, "u1"), new Claim(ClaimTypes.Role, "Admin")], "Test");
        return Task.FromResult(AuthenticateResult.Success(
            new AuthenticationTicket(new ClaimsPrincipal(identity), "Test")));
    }
}

// in ConfigureTestServices:
// services.AddAuthentication("Test").AddScheme<AuthenticationSchemeOptions, TestAuthHandler>("Test", _ => { });
// services.PostConfigure<AuthenticationOptions>(o => { o.DefaultAuthenticateScheme = "Test"; o.DefaultChallengeScheme = "Test"; });
```

**Interview traps**
- Using the InMemory provider and believing constraints/transactions work.
- Forgetting `public partial class Program` (type inaccessible/internal).
- Spinning a new container per test (very slow).
- Hosted services/background jobs running during integration tests.
- SQLite connection closed → database disappears.

**Cheat sheet**
- `WebApplicationFactory<Program>` + `ConfigureTestServices` to swap services.
- Testcontainers for real DB; Respawn for resets.
- InMemory ≠ relational; SQLite in-memory needs an open connection.
- Share containers per collection; isolate data per test.
- Fake auth with a test `AuthenticationHandler`.

---

## Topic 15: TDD/BDD, Code Coverage, Mutation Testing & Load Testing

**Study notes**

- **TDD**: Red (write failing test) → Green (simplest code to pass) → Refactor. Drives design toward small, testable units; "London" (mock-heavy, outside-in) vs "Chicago/Detroit" (state-based, inside-out) schools.
- **BDD**: behaviour specified in business language (**Given / When / Then**, Gherkin `.feature` files). .NET tools: **Reqnroll** (open-source successor of SpecFlow, which reached end of life in 2024), or plain xUnit with behaviour-style names.
- **Code coverage**: % of code executed by tests. Tools: **Coverlet** (`dotnet test --collect:"XPlat Code Coverage"`), Visual Studio Enterprise, dotCover; **ReportGenerator** builds HTML reports. Types: line, branch (each `if` outcome), method. Coverage says what was *executed*, not what was *verified*.
- **Mutation testing**: tool modifies your code (flip `>` to `>=`, `+` to `-`, remove statements) and re-runs tests; a mutant that no test fails is a **survivor** → weak assertions. **Stryker.NET** (`dotnet tool install -g dotnet-stryker`, run `dotnet stryker`). **Mutation score** = killed / total (excluding ignored/no-coverage per config).
- **Load testing**: measure throughput/latency under load. Types: load, stress (beyond capacity), spike, soak (long duration). Tools: **k6** (JavaScript scripts, Go engine, thresholds), **NBomber** (C#, scenarios + load simulations), JMeter, Azure Load Testing, Bombardier/wrk for quick tests. Look at **percentiles** (p95/p99), error rate, RPS, and resource saturation. **Open model** (fixed arrival rate) vs **closed model** (fixed concurrent users): closed models under-report latency when the system slows (*coordinated omission*).

### 15.1 [Easy] Describe the TDD cycle with a tiny example

**Answer**
Write a failing test first (red), write the minimum code to make it pass (green), then refactor with tests as a safety net. Repeat in small steps.

**Explanation**
The test defines behaviour before implementation, keeping code testable and requirements explicit. The refactor step is crucial — TDD without refactoring just produces tests.

**Code**
```csharp
using Xunit;

// 1. RED: FizzBuzz does not exist -> compile error / failing test
public class FizzBuzzTests
{
    [Theory]
    [InlineData(3, "Fizz")]
    [InlineData(5, "Buzz")]
    [InlineData(15, "FizzBuzz")]
    [InlineData(7, "7")]
    public void Convert(int n, string expected) => Assert.Equal(expected, FizzBuzz.Convert(n));
}

// 2. GREEN: simplest implementation
public static class FizzBuzz
{
    public static string Convert(int n) => (n % 3, n % 5) switch
    {
        (0, 0) => "FizzBuzz",
        (0, _) => "Fizz",
        (_, 0) => "Buzz",
        _ => n.ToString()
    };
}
// 3. REFACTOR with tests green.
```

### 15.2 [Follow-up] Does 100% code coverage mean the code is well tested?

**Answer**
No. Coverage only shows which lines ran; a test with no assertions can produce 100% coverage. Use it to find untested areas, prefer branch coverage, and complement it with mutation testing to check test quality.

**Explanation**
Chasing a coverage number leads to low-value tests (getters, DI wiring). Typical sensible targets: 70–80% on business logic while excluding generated code (`[ExcludeFromCodeCoverage]`, Coverlet exclude filters). Collect with `dotnet test --collect:"XPlat Code Coverage"` and render via `reportgenerator -reports:**/coverage.cobertura.xml -targetdir:cov`.

**Code**
```csharp
using System.Diagnostics.CodeAnalysis;

public class Discount
{
    public decimal Apply(decimal price, bool vip) => vip ? price * 0.9m : price;
}

public class WeakTests
{
    [Fact] public void Runs_both_branches_without_checking()
    {
        var d = new Discount();
        d.Apply(100, true);     // 100% line & branch coverage...
        d.Apply(100, false);    // ...but zero assertions => nothing verified
    }
}

[ExcludeFromCodeCoverage] public class GeneratedDto { public int Id { get; set; } }
```

### 15.3 [Medium] How does mutation testing work and what does a surviving mutant tell you?

**Answer**
Stryker creates many small variants (mutants) of your code and runs your tests against each; if tests still pass, the mutant "survived", meaning your tests don't verify that behaviour. The mutation score summarises how many mutants your tests killed.

**Explanation**
Example: the code `if (age >= 18)` mutated to `age > 18` — if no test checks age exactly 18, the mutant survives (boundary missing). Mutation testing is slow (runs tests many times), so run it on changed files in CI (`--since`) or nightly. Configure `stryker-config.json` (thresholds high/low/break).

**Code**
```csharp
public static class Age
{
    public static bool IsAdult(int age) => age >= 18;
}

public class AgeTests
{
    [Fact] public void Adult()  => Assert.True(Age.IsAdult(30));
    [Fact] public void Minor()  => Assert.False(Age.IsAdult(5));
    // Mutant ">= -> >" SURVIVES: neither test uses 18. Add the boundary test to kill it:
    [Fact] public void Boundary() => Assert.True(Age.IsAdult(18));
}
// dotnet tool install -g dotnet-stryker ; cd tests/MyApp.Tests ; dotnet stryker
```

### 15.4 [Medium] Load test an API with NBomber (and the equivalent k6 idea)

**Answer**
Define a scenario that calls the endpoint, attach a load simulation (e.g., inject 100 requests/second for 30 s) and let the tool report RPS, latency percentiles and failures. k6 does the same with a JS script (`stages`, `thresholds`).

**Explanation**
Use *arrival-rate* (open) simulations to avoid coordinated omission. Always warm up, run against a production-like environment with realistic data, and monitor the server (CPU, GC, DB) concurrently. Load-generator saturation produces misleading numbers — run it on a separate machine.

**Code**
```csharp
using NBomber.CSharp;
using NBomber.Http.CSharp;

var http = new HttpClient();

var scenario = Scenario.Create("get_products", async ctx =>
{
    var request = Http.CreateRequest("GET", "https://localhost:5001/products");
    var response = await Http.Send(http, request);
    return response;
})
.WithWarmUpDuration(TimeSpan.FromSeconds(5))
.WithLoadSimulations(Simulation.Inject(rate: 100, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromSeconds(30)));

NBomberRunner.RegisterScenarios(scenario).Run();

// k6 equivalent (script.js):
//   export const options = { scenarios: { api: { executor: 'constant-arrival-rate', rate: 100, timeUnit: '1s',
//       duration: '30s', preAllocatedVUs: 50 } }, thresholds: { http_req_duration: ['p(95)<300'], http_req_failed: ['rate<0.01'] } };
//   export default function () { http.get('https://localhost:5001/products'); }
```

### 15.5 [Hard] Interpreting load-test results: why does latency explode at some throughput?

**Answer**
Past the saturation point queues form (thread pool, DB pool, CPU), so latency rises non-linearly while throughput plateaus or drops. Find the bottleneck resource, not just the symptom: compare RPS vs p99, CPU, GC, thread pool queue length, DB wait stats and connection-pool waits.

**Explanation**
Little's Law: concurrency = throughput × latency. If latency doubles at fixed concurrency, throughput halves. Closed-model tests hide this (virtual users slow down with the server — coordinated omission), while arrival-rate tests show the backlog. Also test with realistic data volume (index/cache behaviour), include think-time only if modelling users, and test *failure* (stress/soak) for leaks. Compare p50 vs p99: a big gap suggests GC pauses, lock contention, or cold paths.

**Code**
```csharp
// Little's Law helper for capacity reasoning
static double RequiredConcurrency(double requestsPerSecond, double avgLatencySeconds)
    => requestsPerSecond * avgLatencySeconds;

Console.WriteLine(RequiredConcurrency(500, 0.2)); // Output: 100  (need ~100 concurrent in-flight requests)
```

**Gotchas / follow-ups**
- Average latency hides tail; report p95/p99 and max.
- Test environment ≠ production sizing (smaller DB) invalidates conclusions.

### 15.6 [Tricky] BDD with Gherkin: when does it help and when is it overhead?

**Answer**
BDD helps when non-developers (product/QA) collaborate on acceptance criteria and the feature files are living documentation; it's overhead when only developers write and read the scenarios, where plain xUnit tests are simpler. Step definitions bind Gherkin text to code (Reqnroll/SpecFlow-style).

**Explanation**
Gherkin scenarios should describe behaviour, not UI clicks ("imperative" scenarios become brittle). Reqnroll uses `[Binding]` classes with `[Given]/[When]/[Then]` attributes and integrates with xUnit/NUnit/MSTest. Beware step-definition explosion and slow tests when scenarios drive the UI.

**Code**
```csharp
// Feature file (Gherkin):
//   Scenario: VIP discount
//     Given a VIP customer
//     When they buy an item priced 100
//     Then the total is 90

using Reqnroll;
using Xunit;

[Binding]
public class DiscountSteps
{
    private bool _vip; private decimal _total;

    [Given("a VIP customer")] public void GivenVip() => _vip = true;
    [When("they buy an item priced {int}")] public void WhenBuy(int price) => _total = _vip ? price * 0.9m : price;
    [Then("the total is {int}")] public void ThenTotal(int expected) => Assert.Equal(expected, _total);
}
```

**Interview traps**
- "100% coverage = no bugs."
- Calling any test written after the code "TDD".
- Running load tests from the same machine as the server.
- Quoting average latency instead of percentiles.
- Treating SpecFlow as actively maintained (Reqnroll is the community successor).

**Cheat sheet**
- TDD: Red → Green → Refactor.
- Coverage = executed, not verified; use branch coverage.
- Mutation testing (Stryker.NET) measures test strength; survivors = missing assertions.
- k6 (JS) / NBomber (C#); prefer arrival-rate model; read p95/p99.
- Little's Law: concurrency = throughput × latency.

---

## Topic 16: Docker, Concurrency Tuning & Graceful Degradation

**Study notes**

**Docker for .NET**: build with the SDK image, run on the smaller runtime image (**multi-stage**). Official images on `mcr.microsoft.com/dotnet/{sdk,aspnet,runtime}`. Since .NET 8: containers run **non-root** by default option (`USER $APP_UID`), apps listen on **port 8080** (not 80). **Chiseled** (Ubuntu) images (`8.0-jammy-chiseled`) are tiny, distroless-like: no shell, no package manager, non-root, smaller attack surface; trade-off: harder to debug (use `dotnet-monitor`/ephemeral debug containers). Layer caching: copy `.csproj` and `dotnet restore` before copying source. `dotnet publish` can also build container images without a Dockerfile (`PublishContainer` — .NET SDK 7.0.100+/8).

**Concurrency & throughput**
- **Thread pool**: starts with min threads = processor count; beyond that injects ~1–2 threads/sec (hill-climbing) → starvation under sync-over-async bursts. Tune `ThreadPool.SetMinThreads` only as a mitigation.
- **Kestrel limits** (`ConfigureKestrel(k => k.Limits...)`): `MaxConcurrentConnections` (default unlimited), `MaxRequestBodySize` (30 MB default), `RequestHeadersTimeout` (30 s), `KeepAliveTimeout` (130 s), `Http2.MaxStreamsPerConnection` (100), `MinRequestBodyDataRate`.
- **Rate limiting middleware** (.NET 7+): fixed window, sliding window, token bucket, **concurrency** limiter.
- **GC mode**: Server GC (throughput, more memory, one heap per core) vs Workstation (default for non-ASP.NET apps; lower footprint). In containers, CPU/memory **limits** change `Environment.ProcessorCount` and GC heap hard limits.

**Graceful degradation**: keep the core function working when parts fail — timeouts, circuit breakers, fallbacks/stale cache, feature flags to disable non-critical features, **load shedding** (reject early with 429/503 + `Retry-After`), queue-based load leveling, health checks (liveness vs readiness), and graceful shutdown on SIGTERM.

### 16.1 [Easy] Write a multi-stage Dockerfile for an ASP.NET Core 8 app. Why multi-stage?

**Answer**
Multi-stage builds compile with the large SDK image and copy only the published output into the small ASP.NET runtime image, so the final image is smaller, faster to pull, and has fewer attack surface. Copy the `.csproj` first to cache the restore layer.

**Explanation**
Final image has no SDK/compilers. Use `.dockerignore` (bin/obj/.git). .NET 8 images expose port 8080 and include a non-root `app` user (`$APP_UID`).

**Code**
```dockerfile
# syntax=docker/dockerfile:1
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY MyApp/MyApp.csproj MyApp/
RUN dotnet restore MyApp/MyApp.csproj          # cached unless csproj changes
COPY . .
RUN dotnet publish MyApp/MyApp.csproj -c Release -o /app/publish --no-restore

FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
WORKDIR /app
COPY --from=build /app/publish .
USER $APP_UID
EXPOSE 8080
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

### 16.2 [Follow-up] What are chiseled images and what's the catch?

**Answer**
Chiseled images (e.g. `mcr.microsoft.com/dotnet/aspnet:8.0-jammy-chiseled`) are minimal Ubuntu-based images with only what .NET needs: no shell, no package manager, non-root by default. They cut size and CVEs; the catch is you can't `exec` into a shell or `apt install` tools, so debug with logs, `dotnet-monitor`, or ephemeral debug containers.

**Explanation**
Variants include `-chiseled-extra` (adds ICU/tzdata for globalization) — the plain chiseled runs in **invariant globalization** mode unless ICU is present, which can surprise culture/time-zone code. Native AOT apps can use `runtime-deps:8.0-jammy-chiseled-aot`. Healthchecks via `HEALTHCHECK curl` won't work (no curl) — use orchestrator probes instead.

**Code**
```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish -c Release -o /app

FROM mcr.microsoft.com/dotnet/aspnet:8.0-jammy-chiseled-extra AS final
WORKDIR /app
COPY --from=build /app .
EXPOSE 8080
ENTRYPOINT ["dotnet", "MyApp.dll"]     # exec form required: no shell to interpret strings
```

### 16.3 [Medium] Which Kestrel and thread-pool settings matter for throughput, and how do you set them?

**Answer**
Set Kestrel `Limits` (max connections, body size, timeouts, HTTP/2 stream limits) to protect the server, and understand the thread pool's minimum threads and slow ramp-up. Fixing async code is better than raising `SetMinThreads`.

**Explanation**
Use `MaxRequestBodySize` per endpoint for uploads. Bounded connection limits avoid memory exhaustion from slowloris-style clients. For bursty workloads that legitimately use blocking calls, `ThreadPool.SetMinThreads` (or the `System.Threading.ThreadPool.MinThreads` runtimeconfig setting) can remove the ramp-up delay.

**Code**
```csharp
builder.WebHost.ConfigureKestrel(k =>
{
    k.Limits.MaxConcurrentConnections = 10_000;
    k.Limits.MaxRequestBodySize = 10 * 1024 * 1024;
    k.Limits.RequestHeadersTimeout = TimeSpan.FromSeconds(15);
    k.Limits.Http2.MaxStreamsPerConnection = 100;
});

ThreadPool.GetMinThreads(out var w, out var io);
Console.WriteLine($"Min worker={w}, io={io}");        // Output: equals Environment.ProcessorCount by default
ThreadPool.SetMinThreads(Math.Max(w, 64), io);        // mitigation only
```

### 16.4 [Hard] How do container CPU/memory limits affect .NET throughput and GC?

**Answer**
.NET reads cgroup limits: `Environment.ProcessorCount` reflects the CPU limit (rounded up), so a 0.5-CPU limit gives 1 core worth of thread-pool min threads and 1 GC heap; the GC heap hard limit defaults to 75% of the memory limit. Under-provisioned CPU causes CFS throttling → latency spikes even when "CPU usage" looks low.

**Explanation**
Server GC creates one heap/thread per logical core — wasteful in small pods; use `DOTNET_gcServer=0` (workstation) or `DOTNET_GCHeapCount`/`GCHeapAffinitizeMask` to cap, and `DOTNET_GCConserveMemory` for tighter footprints. CPU throttling shows in cgroup `cpu.stat` (`nr_throttled`). Set requests ≈ typical usage and limits generously (or none for CPU) to avoid throttling; always set memory limit for predictable OOM behaviour. Few cores → thread pool min threads small → starvation arrives earlier.

**Code**
```csharp
Console.WriteLine($"CPUs: {Environment.ProcessorCount}");
Console.WriteLine($"Server GC: {System.Runtime.GCSettings.IsServerGC}");
var info = GC.GetGCMemoryInfo();
Console.WriteLine($"Memory limit seen by GC: {info.TotalAvailableMemoryBytes / 1_048_576} MB");
// runtimeconfig / env: DOTNET_gcServer=0, DOTNET_GCHeapHardLimitPercent=0x4B (hex: 75%)
```

**Gotchas / follow-ups**
- Env var values for GC settings are **hex** (e.g. `DOTNET_GCHeapHardLimit=0x10000000`).
- Container support (cgroup limits) is built into .NET Core 3.0+ with no flag needed.

### 16.5 [Medium] Add load shedding with the built-in rate limiter

**Answer**
Use `AddRateLimiter` with a concurrency or token-bucket limiter, attach it with `RequireRateLimiting`, and return 429 (optionally with `Retry-After`) when saturated. This protects a server by rejecting excess work early instead of queuing until timeouts.

**Explanation**
Limit by partition (per user/IP/API key) with `PartitionedRateLimiter`. Concurrency limiter bounds in-flight work; a small `QueueLimit` smooths bursts but a big queue just adds latency. Per-instance only — for global limits use a gateway or distributed limiter.

**Code**
```csharp
using System.Threading.RateLimiting;
using Microsoft.AspNetCore.RateLimiting;

builder.Services.AddRateLimiter(o =>
{
    o.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
    o.OnRejected = (ctx, _) =>
    {
        ctx.HttpContext.Response.Headers.RetryAfter = "5";
        return ValueTask.CompletedTask;
    };
    o.AddConcurrencyLimiter("heavy", c =>
    {
        c.PermitLimit = 20;
        c.QueueLimit = 10;
        c.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
    });
});

var app = builder.Build();
app.UseRateLimiter();
app.MapGet("/report", () => "ok").RequireRateLimiting("heavy");
```

### 16.6 [Tricky] Design graceful degradation and graceful shutdown for an e-commerce API

**Answer**
Keep checkout working when recommendations, reviews or search are down: wrap each dependency with timeout + circuit breaker + fallback (stale cache/empty list), use feature flags to switch off non-critical features, shed load with 429/503, and expose separate liveness and readiness health checks. On SIGTERM, fail readiness first, drain in-flight requests within the grace period, then stop.

**Explanation**
Liveness answers "restart me?" (cheap, no dependency checks); readiness answers "send me traffic?" (checks DB/cache). Mixing them causes restart loops when a dependency blips. During shutdown Kubernetes removes the pod from endpoints asynchronously, so a short `preStop` sleep + `HostOptions.ShutdownTimeout` shorter than `terminationGracePeriodSeconds` avoids dropped requests. Prioritise critical flows (bulkheads) so non-critical traffic can't starve checkout.

**Code**
```csharp
builder.Services.AddHealthChecks()
    .AddCheck("self", () => Microsoft.Extensions.Diagnostics.HealthChecks.HealthCheckResult.Healthy(), tags: ["live"])
    .AddSqlServer(builder.Configuration.GetConnectionString("Db")!, tags: ["ready"]);   // AspNetCore.HealthChecks.SqlServer

builder.Services.Configure<HostOptions>(o => o.ShutdownTimeout = TimeSpan.FromSeconds(25));

app.MapHealthChecks("/health/live", new() { Predicate = r => r.Tags.Contains("live") });
app.MapHealthChecks("/health/ready", new() { Predicate = r => r.Tags.Contains("ready") });

app.MapGet("/product/{id}", async (int id, IRecommendations rec, ILogger<Program> log) =>
{
    IReadOnlyList<int> related;
    try { related = await rec.GetAsync(id).WaitAsync(TimeSpan.FromMilliseconds(300)); }   // timeout
    catch (Exception ex) { log.LogWarning(ex, "Recommendations degraded"); related = []; } // fallback
    return Results.Ok(new { id, related });
});
```

**Interview traps**
- Putting dependency checks in the liveness probe → cascading restarts.
- Raising `MaxConcurrentConnections`/min threads instead of fixing blocking code.
- Server GC in a tiny container.
- Chiseled image + `HEALTHCHECK CMD curl` or shell-form `ENTRYPOINT` (no shell).
- Large rate-limiter queues that convert overload into long latency instead of fast failure.

**Cheat sheet**
- Multi-stage: SDK build → `aspnet` runtime; copy csproj first for caching; port 8080, non-root.
- Chiseled = tiny, no shell, invariant globalization unless `-extra`.
- Thread pool ramps slowly; fix sync-over-async rather than `SetMinThreads`.
- Container CPU limit → `ProcessorCount`, GC heaps, throttling.
- Degrade: timeout + breaker + fallback + flags + shed load; liveness ≠ readiness.

---

*End of B05.*
