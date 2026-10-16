# Z08 - Redis & Caching on Azure (for .NET developers)

**How to use:** Read each topic's *Study notes*, then work the difficulty ladder top-down (Easy -> Tricky); cover the Answer and say it aloud before reading the Explanation.
**Caveat:** Azure Redis offerings, SKUs, retirement dates, limits and prices change often - **verify in current docs** anything marked (verify).

## Table of contents
1. Why cache
2. Caching patterns
3. Redis basics and data structures
4. Key design, TTL, eviction, maxmemory
5. Azure Redis offerings (ACR tiers vs Azure Managed Redis)
6. StackExchange.Redis: connection and resilience
7. IDistributedCache, HybridCache, serialization
8. Invalidation, stampede, penetration, avalanche
9. Distributed locks
10. Rate limiting, counters, leaderboards
11. Pub/Sub vs Streams vs Service Bus; session and output cache
12. Transactions, Lua, pipelining, clustering and hash slots
13. Persistence, HA, failover
14. Security: keys vs Entra ID, TLS, private endpoints
15. Performance troubleshooting, sizing, cost
16. Testing, local dev, Redis vs alternatives
17. Production scenarios

---

## Topic 1: Why cache

**Study notes**
A cache stores computed/fetched data closer or cheaper to reach than the source of truth. Goals: lower latency, reduce load/cost on the backing store, absorb spikes, improve resilience.

```
Client -> App -> [Cache: ~sub-ms to few ms] --miss--> [DB / API: 10-500ms, limited RU/DTU]
```
| Cache kind | Scope | Pros | Cons |
|---|---|---|---|
| In-process (IMemoryCache) | one instance | fastest, no network | per-instance copies, inconsistent, lost on restart |
| Distributed (Redis) | shared | consistent across instances, survives app restarts | network hop, ops cost, serialization |
| CDN / HTTP cache | edge | offloads whole requests | coarse invalidation |

Good candidates: read-heavy, expensive to compute, tolerant of slight staleness. Bad: write-heavy, must-be-exact data (balances), highly unique keys (low hit rate).

### 1.1 [Easy] What is a cache and why use one?
**Answer**
A cache is a fast temporary store for data that is expensive to fetch or compute. It cuts latency and protects the backing store from load. It trades some freshness and complexity for speed.
**Explanation**
Hit ratio = hits/(hits+misses) is the key metric. Even a 90% hit ratio cuts DB reads 10x. Caches should always be treated as disposable: the app must work (slower) if the cache is empty or down.
**Code / CLI / Bicep**
```csharp
// Simplest in-process caching
var value = await memoryCache.GetOrCreateAsync("product:42", async e =>
{
    e.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5);
    return await repo.GetProductAsync(42);
});
```

### 1.2 [Follow-up] When do you choose Redis over IMemoryCache?
**Answer**
Choose Redis when multiple instances must share cached data or consistency matters, or when cache must survive app restarts/scale-out. IMemoryCache is faster but per-instance. Often you use both (L1 + L2), which is what HybridCache does.
**Explanation**
With N instances in-memory, each caches separately: N times the memory, N misses per key, and invalidation does not propagate. Redis gives one copy, atomic ops (INCR, locks), pub/sub. Cost: network RTT (~0.5-2ms in-region), serialization, and a new failure dependency.
**Code / CLI / Bicep**
```csharp
builder.Services.AddStackExchangeRedisCache(o =>
{
    o.Configuration = builder.Configuration["Redis:Connection"];
    o.InstanceName = "orders:";
});
```

### 1.3 [Medium] What are the costs/risks of adding a cache?
**Answer**
Staleness, invalidation complexity, a new failure mode, cold-start load on the DB, and added operations/cost. A cache can hide a slow query rather than fix it. You must design for cache-down and cold-cache scenarios.
**Explanation**
Classic outage: cache restarts/fails over, hit ratio drops to 0, DB receives full traffic and collapses. Mitigate with fallback timeouts, circuit breakers, stampede protection, DB capacity planning for cold cache, and cache warm-up.
**Code / CLI / Bicep**
```csharp
try { cached = await db.StringGetAsync(key); }
catch (RedisException ex) { logger.LogWarning(ex, "Cache unavailable, falling back"); }
// fall through to DB; never let a cache failure fail the request
```

### 1.4 [Hard] How do you decide what TTL / staleness is acceptable?
**Answer**
Derive it from business tolerance: how wrong can the data be and for how long. Combine a TTL as a safety net with explicit invalidation on writes when freshness matters. Measure hit ratio and source load to tune.
**Explanation**
Reference data (country list): hours/days. Product catalog: minutes + event invalidation. Per-user permissions: short TTL, because stale authorization is a security issue. Never rely only on invalidation (messages can be lost); TTL bounds the damage.
**Code / CLI / Bicep**
```csharp
await db.StringSetAsync(key, json, TimeSpan.FromMinutes(10) + TimeSpan.FromSeconds(Random.Shared.Next(0, 60)));
```
**Gotchas / follow-ups**
- Caching authorization/entitlement data with long TTL = privilege staleness.
- Caching per-user data under a shared key = data leak between users.

**Interview traps**
- "Cache makes everything faster" - not if hit ratio is low or serialization dominates.
- The cache must never be the only copy of data.
- Forgetting cold-cache capacity of the DB.

**Cheat sheet**
- Cache = disposable, faster, possibly stale.
- Measure hit ratio.
- TTL always + invalidation when needed.
- Fail open to source with timeouts.

---

## Topic 2: Caching patterns

**Study notes**
```
Cache-aside:   App -> cache? miss -> App -> DB -> App writes cache
Read-through:  App -> cache layer -> (miss) cache loads from DB itself
Write-through: App -> cache layer -> writes cache AND DB synchronously
Write-behind:  App -> cache -> (async later) DB   (batching)
Refresh-ahead: cache proactively reloads hot entries before TTL expiry
```
| Pattern | Pros | Cons |
|---|---|---|
| Cache-aside | simple, resilient, only caches what is used | stale on write, miss penalty, stampede |
| Read-through | app code simpler | needs a provider/library; Redis itself doesn't do it natively |
| Write-through | cache consistent with DB | write latency, caches unread data |
| Write-behind | fast writes, batching | data loss risk if cache dies, ordering complexity |
| Refresh-ahead | no miss latency for hot keys | wasted loads, complexity |

Redis OSS has no native read/write-through; you implement in code (or via libraries / Redis Enterprise-type features - verify). Cache-aside is the .NET default.

### 2.1 [Easy] Explain cache-aside.
**Answer**
App checks the cache; on miss it loads from the DB, stores in the cache with a TTL, and returns. On writes, it updates the DB and invalidates (or updates) the cache entry. The application owns cache logic.
**Explanation**
Resilient: cache down = slower, not broken. Prefer *delete* on write over *update* to reduce race conditions where an older value overwrites a newer one.
**Code / CLI / Bicep**
```csharp
public async Task<Product?> GetAsync(int id)
{
    var key = $"product:v1:{id}";
    var hit = await _db.StringGetAsync(key);
    if (hit.HasValue) return JsonSerializer.Deserialize<Product>((string)hit!);

    var p = await _repo.GetAsync(id);
    if (p is not null)
        await _db.StringSetAsync(key, JsonSerializer.Serialize(p), TimeSpan.FromMinutes(10));
    return p;
}
public async Task UpdateAsync(Product p)
{
    await _repo.UpdateAsync(p);
    await _db.KeyDeleteAsync($"product:v1:{p.Id}");
}
```

### 2.2 [Follow-up] Read-through vs cache-aside? Write-through vs write-behind?
**Answer**
Read-through moves the load-on-miss logic into the cache layer; cache-aside keeps it in the app. Write-through writes cache and DB synchronously (consistent, slower); write-behind acknowledges after the cache and flushes to the DB later (fast, risk of loss).
**Explanation**
Write-behind with plain Redis on Azure is risky: a failover or eviction before flush loses writes. If needed, use Redis Streams or Service Bus as a durable buffer and a worker to write to DB instead.
**Code / CLI / Bicep**
```csharp
// Write-through (manual)
await _repo.SaveAsync(item);            // source of truth first
await _db.StringSetAsync(key, json, ttl);
```

### 2.3 [Medium] Race condition in cache-aside: update vs read.
**Answer**
Reader misses, reads old DB value; writer updates DB and deletes the cache; reader then writes the stale value into the cache. Stale data persists until TTL. Mitigate with TTL, versioning, or short TTLs on hot data.
**Explanation**
Sequence: R1 reads DB (old) -> W updates DB -> W deletes key -> R1 sets cache (old). Rare but real. Fixes: version column in the cache value and conditional set via Lua (set only if version is newer), delayed double-delete, or just accept TTL-bounded staleness.
**Code / CLI / Bicep**
```csharp
// Set only if our version is newer (atomic via Lua)
const string script = @"
local cur = redis.call('HGET', KEYS[1], 'v')
if (not cur) or tonumber(cur) < tonumber(ARGV[1]) then
  redis.call('HSET', KEYS[1], 'v', ARGV[1], 'd', ARGV[2])
  redis.call('PEXPIRE', KEYS[1], ARGV[3])
  return 1
end
return 0";
await _db.ScriptEvaluateAsync(script, new RedisKey[]{key}, new RedisValue[]{ver, json, 600000});
```
**Gotchas / follow-ups**
- In clusters, only keys passed in KEYS are slot-checked; keep a single key per script.

### 2.4 [Hard] When is refresh-ahead worth it, and how do you do it with Redis?
**Answer**
For hot keys where a miss is costly (slow computation) and access is predictable. Check remaining TTL on read; if below a threshold, trigger a background refresh while serving the current value. HybridCache and probabilistic early refresh give similar benefits.
**Explanation**
Use `KeyTimeToLiveAsync` and refresh asynchronously guarded by a lock so only one instance refreshes. Cost: extra TTL call per read, and refreshing keys nobody asks for.
**Code / CLI / Bicep**
```csharp
var ttl = await _db.KeyTimeToLiveAsync(key);
if (ttl is { } t && t < TimeSpan.FromSeconds(30))
    _ = Task.Run(() => RefreshWithLockAsync(key));   // lock prevents N refreshers
```
**Gotchas / follow-ups**
- Fire-and-forget needs exception handling/logging; otherwise errors vanish.

**Interview traps**
- Saying Redis "does" read-through natively.
- Update-on-write vs delete-on-write races.
- Write-behind data loss on failover.

**Cheat sheet**
- Default: cache-aside + TTL + delete on write.
- Write-behind needs a durable buffer.
- Refresh-ahead = lock + background reload.

---

## Topic 3: Redis basics and data structures

**Study notes**
Redis is an in-memory data-structure server. Commands execute on a **single main thread** (atomic per command; no parallel command execution), I/O threading (Redis 6+) parallelizes network read/write only. So one slow command (KEYS *, big SMEMBERS, long Lua) blocks everyone.

| Type | Use | Key commands |
|---|---|---|
| String | cache blobs, counters, locks | GET/SET/INCR/SET NX PX |
| Hash | object fields | HSET/HGET/HINCRBY |
| List | queue/stack | LPUSH/RPOP/BLPOP |
| Set | uniqueness, tags | SADD/SISMEMBER/SINTER |
| Sorted set | leaderboard, sliding window, scheduling | ZADD/ZRANGE/ZRANGEBYSCORE |
| Stream | append log + consumer groups | XADD/XREADGROUP/XACK |
| HyperLogLog | approx distinct count (~0.81% error, 12KB) | PFADD/PFCOUNT |
| Bitmap | flags per id, DAU | SETBIT/BITCOUNT |
| Geo | radius queries (on a sorted set) | GEOADD/GEOSEARCH |

### 3.1 [Easy] What is Redis and what makes it fast?
**Answer**
Redis is an in-memory key-value/data-structure store with optional persistence. It is fast because data lives in RAM and the single-threaded command loop avoids locking overhead. Typical operations are sub-millisecond server-side.
**Explanation**
Network RTT usually dominates, hence pipelining/batching matters. Commands are atomic because only one runs at a time.
**Code / CLI / Bicep**
```bash
redis-cli -h <name>.redis.cache.windows.net -p 6380 --tls -a <key> PING
```

### 3.2 [Follow-up] Which data structure for: object cache, unique visitors, leaderboard, job queue?
**Answer**
Object cache: string (JSON) or hash if partial field updates are needed. Unique visitors: set (exact) or HyperLogLog (approximate, tiny). Leaderboard: sorted set. Queue: list or stream (with consumer groups for reliability).
**Explanation**
Hash allows HINCRBY and reading one field without deserializing everything, but per-field TTL isn't generally available (only whole-key; newer Redis versions add hash field expiration - verify). Lists lose a message if a consumer crashes after pop; streams with XACK/XPENDING fix that.
**Code / CLI / Bicep**
```csharp
await db.HashSetAsync("user:1", new HashEntry[]{ new("name","Asha"), new("tier","gold") });
await db.HashIncrementAsync("user:1", "logins");
await db.HyperLogLogAddAsync("uv:2026-10-16", userId);
long approx = await db.HyperLogLogLengthAsync("uv:2026-10-16");
await db.SortedSetAddAsync("lb", "asha", 1200);
```

### 3.3 [Medium] Explain Redis's threading model and its production implications.
**Answer**
Commands run on one main thread, so each is atomic but long ones block all clients. Redis 6+ can use I/O threads for socket handling. Avoid O(N) commands on big collections (KEYS, SMEMBERS, HGETALL on huge hashes, DEL of huge key - use UNLINK).
**Explanation**
Latency for everyone spikes when one client runs a slow command; this shows up as client-side timeouts on unrelated operations. Use SCAN (cursor, incremental), UNLINK (async free), and break up big collections. Managed offerings may use multiple vCPUs for background tasks/proxy, but command execution is still serialized per shard.
**Code / CLI / Bicep**
```csharp
// Use SCAN via server (StackExchange.Redis uses SCAN under the hood for Keys())
var server = mux.GetServer(endpoint);
await foreach (var k in server.KeysAsync(pattern: "session:*", pageSize: 500)) { /* ... */ }
await db.KeyDeleteAsync(key, CommandFlags.FireAndForget); // DEL; use UNLINK via ExecuteAsync for big keys
```
**Gotchas / follow-ups**
- Don't run KeysAsync against production on a hot path; on replicas/each shard separately in clusters.
- Big-key DEL blocks; use `db.ExecuteAsync("UNLINK", key)`.

### 3.4 [Tricky] Are Redis operations "transactional"/isolated across multiple commands? What about atomicity of read-modify-write from client code?
**Answer**
Single commands are atomic; a client-side GET then SET is not. Use atomic commands (INCR, SET NX), MULTI/EXEC with conditions, or Lua scripts for read-modify-write. MULTI/EXEC provides isolation but no rollback.
**Explanation**
Two instances doing GET -> +1 -> SET lose updates. INCR avoids it. Condition checks in StackExchange.Redis use `CreateTransaction()` + `AddCondition` (WATCH-based optimistic concurrency).
**Code / CLI / Bicep**
```csharp
var tran = db.CreateTransaction();
tran.AddCondition(Condition.StringEqual("owner", "me"));
var t1 = tran.StringSetAsync("owner", "me2");
bool committed = await tran.ExecuteAsync();   // false if condition failed
```

**Interview traps**
- "Redis is multi-threaded" - commands aren't (I/O threads aside).
- KEYS in production.
- Assuming transactions roll back.

**Cheat sheet**
- Single-threaded commands -> one slow command hurts all.
- Know the 9 types and a use-case each.
- Prefer SCAN/UNLINK.
- Atomic = single command / Lua / MULTI.

---

## Topic 4: Key design, TTL, eviction, maxmemory

**Study notes**
Key convention: `app:entity:version:id[:qualifier]`, e.g. `shop:product:v3:42`. Keep keys short but readable; use `:` separators; include a schema version; include tenant/user when data is scoped.

TTL: `EXPIRE/PEXPIRE/SET ... EX|PX`. Expiry is lazy (on access) + active (sampling), so expired keys may briefly occupy memory. `SET` without TTL on an existing key **removes** the TTL (unless KEEPTTL).

Eviction when `maxmemory` is reached:
| Policy | Behavior |
|---|---|
| noeviction | writes fail with OOM error, reads fine |
| allkeys-lru / allkeys-lfu | evict any key (approximate LRU/LFU) |
| volatile-lru / -lfu / -ttl / -random | evict only keys with a TTL |
| allkeys-random | random |

Azure Cache for Redis default is typically `volatile-lru` (verify); with Azure Managed Redis the eviction policy choice is set on the cluster/database (verify in current docs).

### 4.1 [Easy] How should you name Redis keys?
**Answer**
Use a consistent hierarchical scheme such as `app:entity:version:id`. Include a version segment and any tenant identifier. Keep keys short but not cryptic.
**Explanation**
Consistency enables SCAN patterns, metrics, and safe version bumps. Keys are stored in memory for every entry; millions of 200-byte keys cost real memory.
**Code / CLI / Bicep**
```csharp
static string ProductKey(int id) => $"shop:product:v3:{id}";
static string UserCartKey(Guid tenant, Guid user) => $"shop:cart:v1:{tenant:N}:{user:N}";
```

### 4.2 [Follow-up] How does TTL work, and what resets it?
**Answer**
TTL sets a countdown after which the key is deleted. Overwriting with plain SET clears the TTL; INCR/HSET on existing keys keep it. Check with TTL/PTTL.
**Explanation**
In StackExchange.Redis `StringSetAsync(key, value)` without expiry discards the previous TTL - a classic bug that creates immortal keys. Use `keepTtl: true` where applicable.
**Code / CLI / Bicep**
```csharp
await db.StringSetAsync(key, v, TimeSpan.FromMinutes(5));
await db.StringSetAsync(key, v2, keepTtl: true);   // preserve remaining TTL
```
```bash
TTL shop:product:v3:42
```

### 4.3 [Medium] Which eviction policy for a pure cache vs mixed use?
**Answer**
For a pure cache, allkeys-lru (or lfu) lets Redis free memory whatever the keys are. If you mix cache with data that must not be evicted (locks, counters), use volatile-* so only keys with TTL are evictable - but then ensure everything evictable has a TTL, otherwise you get OOM errors. noeviction is for Redis used as a store.
**Explanation**
Under volatile-lru with no TTL'd keys left, writes fail with `OOM command not allowed when used memory > 'maxmemory'`. Also on Azure, part of memory is reserved (`maxmemory-reserved`, `maxfragmentationmemory-reserved`) for non-cache overhead like replication buffers and fragmentation; setting it too low can cause timeouts during failover/save (verify current guidance).
**Code / CLI / Bicep**
```bash
# Azure Cache for Redis: patch settings (verify exact property names in docs)
az redis update -n mycache -g rg --set "redisConfiguration.maxmemory-policy=allkeys-lru"
```

### 4.4 [Hard] Why does memory grow although you set TTLs? Key explosion causes?
**Answer**
Causes: keys without TTL, unbounded key cardinality (per-request/user/query-string keys), large values, fragmentation, and TTL too long. Fix with bounded keyspace, TTLs everywhere, size limits, and monitoring of key counts/evictions.
**Explanation**
Expiry is probabilistic, so expired-but-unvisited keys linger briefly. Keys built from unbounded input (full URL with query string, timestamps) explode. Track `INFO keyspace` (keys, expires), `MEMORY USAGE key`, `--bigkeys` scan.
**Code / CLI / Bicep**
```bash
redis-cli INFO keyspace        # db0:keys=1200000,expires=300000  <- 900k keys w/o TTL
redis-cli --bigkeys
redis-cli MEMORY USAGE shop:product:v3:42
```
**Gotchas / follow-ups**
- Normalize inputs (lowercase, sorted query params, hashed long inputs) before building keys.
- Hash long keys (SHA-256 -> base64url) but keep a readable prefix.

**Interview traps**
- SET without TTL silently clears TTL.
- noeviction on a cache = outage when full.
- volatile-* with non-TTL keys = OOM.

**Cheat sheet**
- Key = app:entity:ver:id.
- Always TTL; add jitter.
- Cache -> allkeys-lru; mixed -> volatile-lru + TTL discipline.
- Watch evictions + INFO keyspace.

---

## Topic 5: Azure Redis offerings (verify in current docs)

**Study notes**
**Status is changing - verify current docs for SKUs, regions, retirement dates, limits and prices.** As of my knowledge:
- **Azure Cache for Redis (ACR)**: Basic (single node, no SLA, dev/test), Standard (primary+replica, SLA), Premium (clustering, persistence, VNet injection, geo-replication, zone redundancy), plus **Enterprise / Enterprise Flash** (Redis Inc. modules, active geo-replication) delivered via Azure Marketplace.
- **Azure Managed Redis (AMR)**: the newer service built on Redis Enterprise, with tiers Memory Optimized, Balanced, Compute Optimized, Flash Optimized (preview/GA status - verify). Microsoft has announced a retirement path for Azure Cache for Redis, steering customers to AMR; dates differ per tier (Basic/Standard/Premium vs Enterprise) - **check the official retirement announcement and migration guide**.

| Feature | Basic | Standard | Premium | AMR (verify) |
|---|---|---|---|---|
| Replica / SLA | no | yes | yes | yes (HA default) |
| Clustering | no | no | yes (shards) | yes (OSS or Enterprise cluster policy) |
| Persistence | no | no | RDB/AOF | data persistence options (verify) |
| Zone redundancy | no | no | yes | yes (verify per region) |
| Geo-replication | no | no | passive link | active geo-replication (verify) |
| Private link | yes | yes | yes | yes |

### 5.1 [Easy] Which Azure services provide Redis?
**Answer**
Azure Cache for Redis (Basic/Standard/Premium and Enterprise tiers) and the newer Azure Managed Redis. Microsoft is moving customers toward Azure Managed Redis, so new workloads should evaluate AMR first. Verify current retirement dates.
**Explanation**
Interviewers want awareness of the lifecycle: don't pick Basic for production (no replica, no SLA), and plan migration for existing ACR instances.
**Code / CLI / Bicep**
```bicep
resource redis 'Microsoft.Cache/redis@2024-03-01' = {
  name: 'myredis'
  location: resourceGroup().location
  properties: {
    sku: { name: 'Standard', family: 'C', capacity: 1 }
    enableNonSslPort: false
    minimumTlsVersion: '1.2'
    redisConfiguration: { 'maxmemory-policy': 'allkeys-lru' }
  }
}
// API versions evolve - verify current apiVersion. AMR uses Microsoft.Cache/redisEnterprise (verify).
```

### 5.2 [Follow-up] Compare Standard vs Premium for a production workload.
**Answer**
Standard gives primary/replica with an SLA but a single shard and fewer features. Premium adds clustering for scale-out, persistence, VNet injection, zone redundancy, and passive geo-replication. Choose Premium (or AMR) when you need throughput scaling, persistence, or zonal resilience.
**Explanation**
Redis is single-threaded per shard, so scaling CPU-bound workloads needs sharding (clustering) or a larger tier. Check current SKU matrices because sizes (C0-C6, P1-P5) and features are subject to change.
**Code / CLI / Bicep**
```bash
az redis create -n mycache -g rg -l westeurope --sku Premium --vm-size p1 --shard-count 2 --zones 1 2 3
```

### 5.3 [Medium] Explain persistence (RDB vs AOF), zone redundancy and geo-replication on Azure.
**Answer**
RDB = periodic snapshots to storage; AOF = append-only log of writes (smaller loss window, more overhead). Zone redundancy spreads primary/replica across availability zones; geo-replication replicates to another region for DR. None of these make Redis a system of record.
**Explanation**
RDB loses data since the last snapshot; AOF (every second on Azure) loses up to ~1s. Replication is asynchronous, so failover can lose recent writes. Passive geo-replication (Premium): secondary is read-only, failover is manual/unlinking; active geo-replication (Enterprise/AMR) supports multi-region writes with conflict-free data types (verify).
**Code / CLI / Bicep**
```bash
az redis update -n mycache -g rg --set "redisConfiguration.rdb-backup-enabled=true" \
  "redisConfiguration.rdb-backup-frequency=60" "redisConfiguration.rdb-storage-connection-string=<blob-conn>"
```
**Gotchas / follow-ups**
- Persistence storage account recommendations (e.g. storage auth) change - verify.
- Async replication => replica reads can be stale.

### 5.4 [Tricky] Redis versions and upgrades - what should you watch for?
**Answer**
Service versions move (ACR offered 4/6; newer versions via AMR e.g. 7.x - verify). Upgrades can change client behavior, command availability and TLS/auth requirements, so test in a non-prod cache and pin client versions.
**Explanation**
Old ACR versions are retired on a schedule; Microsoft may force upgrades. Deprecated features (non-TLS port 6379, TLS < 1.2) get disabled. Cluster mode on AMR has "OSS cluster" vs "Enterprise cluster" policies that influence client support - OSS cluster API requires cluster-aware clients (StackExchange.Redis supports it) (verify).
**Code / CLI / Bicep**
```bash
az redis show -n mycache -g rg --query "{ver:redisVersion, sku:sku.name, tls:minimumTlsVersion}"
```

**Interview traps**
- Recommending Basic for prod.
- Not knowing the ACR -> AMR direction.
- Treating geo-replication as zero-RPO.

**Cheat sheet**
- Basic = dev only; Standard = HA; Premium = cluster/persist/zones.
- AMR = new strategic offering (verify).
- Replication async -> possible data loss.
- Always check retirement docs.

---

## Topic 6: StackExchange.Redis - connection and resilience

**Study notes**
`ConnectionMultiplexer` is designed to be **created once and shared** (thread-safe, multiplexes many commands over few connections). Creating one per request causes connection storms, port exhaustion, TLS handshake cost, and server connection limits. `IDatabase` is a lightweight pass-through - get it per use.

Key options: `abortConnect=false` (keep retrying at startup instead of throwing), `connectTimeout`, `syncTimeout`, `asyncTimeout`, `connectRetry`, `reconnectRetryPolicy` (ExponentialRetry), `keepAlive`, `ssl=true`, `defaultDatabase`. Azure connection string: `<name>.redis.cache.windows.net:6380,password=<key>,ssl=True,abortConnect=False`.

### 6.1 [Easy] How should you register the Redis connection in ASP.NET Core / Functions?
**Answer**
Register `IConnectionMultiplexer` as a singleton (ideally lazily created) and inject it. Never create a multiplexer per request or per function invocation.
**Explanation**
In Azure Functions, a static field or DI singleton is reused across invocations within the same host instance; a new connection per invocation causes the classic "too many connections"/timeouts. Lazy<T> defers connect until first use and ensures thread-safe single creation.
**Code / CLI / Bicep**
```csharp
public static class RedisConn
{
    private static readonly Lazy<ConnectionMultiplexer> _lazy = new(() =>
    {
        var opts = ConfigurationOptions.Parse(Environment.GetEnvironmentVariable("RedisConnection")!);
        opts.AbortOnConnectFail = false;
        opts.ConnectRetry = 5;
        opts.ReconnectRetryPolicy = new ExponentialRetry(5000);
        return ConnectionMultiplexer.Connect(opts);
    });
    public static ConnectionMultiplexer Connection => _lazy.Value;
}

// or DI
builder.Services.AddSingleton<IConnectionMultiplexer>(_ =>
    ConnectionMultiplexer.Connect(builder.Configuration["Redis:Connection"]!));
```

### 6.2 [Follow-up] What does AbortOnConnectFail=false do, and why use it?
**Answer**
By default the initial Connect throws if no server is reachable. With false, it returns a multiplexer that keeps trying in the background, so the app starts even if Redis is briefly down. Commands fail fast until it connects. Azure connection strings use `abortConnect=False` by default for this reason.
**Explanation**
Combined with fail-open cache logic, the app survives Redis outages and restarts. Without it, a startup race (Redis patching) crashes the host in a restart loop.
**Code / CLI / Bicep**
```csharp
var mux = await ConnectionMultiplexer.ConnectAsync(
    "mycache.redis.cache.windows.net:6380,ssl=True,abortConnect=False,connectTimeout=10000,syncTimeout=5000");
if (!mux.IsConnected) logger.LogWarning("Redis not connected yet; running degraded");
```

### 6.3 [Medium] Explain batching/pipelining and async usage in StackExchange.Redis.
**Answer**
The multiplexer pipelines naturally: multiple async calls in flight share the connection without waiting on each reply. Use `CreateBatch()` to send grouped commands together, or fire several tasks then `Task.WhenAll`. Never use `.Result`/`.Wait()` on these tasks.
**Explanation**
Sync-over-async and blocking calls plus a starved thread pool produce timeout errors that look like Redis problems but are client-side. Batch does not provide atomicity (use transaction for that). `CommandFlags.FireAndForget` skips waiting for the reply.
**Code / CLI / Bicep**
```csharp
var batch = db.CreateBatch();
var tasks = ids.Select(id => batch.StringGetAsync($"p:{id}")).ToList();
batch.Execute();
var values = await Task.WhenAll(tasks);

// simpler: multi-key GET in one command (same slot in cluster)
RedisValue[] vals = await db.StringGetAsync(ids.Select(i => (RedisKey)$"p:{i}").ToArray());
```
**Gotchas / follow-ups**
- Multi-key MGET in a cluster across slots: StackExchange.Redis splits it per slot for you for `StringGet(RedisKey[])` (verify), but raw commands/Lua do not.

### 6.4 [Hard] You see `RedisTimeoutException` under load. What are the causes and how do you investigate?
**Answer**
Common causes: thread-pool starvation, CPU-saturated client/server, large values or slow commands blocking the server, network bursts, too-small `syncTimeout`, or Redis failover/patching. Read the exception text: it contains queue lengths, `IOCP/WORKER` thread-pool stats, and server/client CPU.
**Explanation**
The message includes `inst`, `qu`, `qs`, `in`, `mgr`, `IOCP: Busy=..., Min=...`. If Busy > Min, pool is growing slowly (adds ~1-2 threads/sec): raise `ThreadPool.SetMinThreads` or, better, remove blocking calls. If `qs` (queue sent) high with `in` (bytes in) high: large payloads/bandwidth. Then check server side: slowlog, server load, CPU, evictions, big keys.
**Code / CLI / Bicep**
```csharp
ThreadPool.GetMinThreads(out var w, out var io);
ThreadPool.SetMinThreads(Math.Max(w, 200), Math.Max(io, 200)); // stopgap; fix blocking code
```
```bash
redis-cli SLOWLOG GET 10
redis-cli INFO commandstats
redis-cli CLIENT LIST
```
**Gotchas / follow-ups**
- Don't just bump `syncTimeout` - it hides the problem and lengthens the queue.
- Use async APIs end to end.

### 6.5 [Tricky] After a failover/patch your app keeps throwing errors until restart. Why, and how do you build resilience?
**Answer**
Usually a stale/broken multiplexer that did not reconnect (old DNS, a bug in an old client version, or a long-lived connection with no keepalive). Use a current client, `abortConnect=false`, `ExponentialRetry`, and optionally recreate the multiplexer after repeated failures with a throttled reconnect pattern.
**Explanation**
Microsoft documents a "ForceReconnect" pattern: on `RedisConnectionException`/`SocketException` repeated within a window, dispose and recreate the multiplexer, guarded by min interval to avoid reconnect storms. Add Polly retry with jitter for transient errors; keep command timeouts modest and fail-open.
**Code / CLI / Bicep**
```csharp
// Simplified ForceReconnect pattern
static DateTimeOffset _lastReconnect = DateTimeOffset.MinValue;
static Lazy<ConnectionMultiplexer> _mux = Create();
static Lazy<ConnectionMultiplexer> Create() => new(() => ConnectionMultiplexer.Connect(Config));

public static void ForceReconnect()
{
    var now = DateTimeOffset.UtcNow;
    if (now - _lastReconnect < TimeSpan.FromSeconds(30)) return;   // throttle
    var old = _mux;
    _mux = Create();
    _lastReconnect = now;
    try { old.Value.Dispose(); } catch { }
}
```
**Gotchas / follow-ups**
- Multiple threads calling ForceReconnect: use a lock/Interlocked.
- Reconnect storms from hundreds of instances amplify an outage: add jitter.

**Interview traps**
- "Create ConnectionMultiplexer per request is fine since it's pooled" - false.
- `.Result` on Redis tasks.
- Raising timeouts instead of finding the cause.

**Cheat sheet**
- One multiplexer, Lazy<T>, singleton.
- abortConnect=false, ExponentialRetry.
- Read timeout message: IOCP/WORKER, qu/qs.
- Async all the way; fail open.

---

## Topic 7: IDistributedCache, HybridCache, serialization

**Study notes**
- `IDistributedCache` (Microsoft.Extensions.Caching.StackExchangeRedis): byte[]/string get/set, sliding/absolute expiration, no typed API, no stampede protection. Registered via `AddStackExchangeRedisCache`.
- `HybridCache` (Microsoft.Extensions.Caching.Hybrid, .NET 9 era; verify package version/GA): typed `GetOrCreateAsync`, **L1 (in-memory) + L2 (IDistributedCache, e.g. Redis)**, built-in **stampede protection** (concurrent calls for the same key share one factory execution), configurable serializers, tags (tag-based invalidation support evolving - verify).

```
GetOrCreateAsync(key) -> L1 hit? -> return
                      -> L2 (Redis) hit? -> populate L1 -> return
                      -> one factory call (others wait) -> write L2 + L1
```
Serialization: System.Text.Json (default for HybridCache, readable, larger), MessagePack (compact, fast, needs contracts), protobuf (compact, schema-evolution friendly). Smaller payload = less memory, bandwidth, latency.

### 7.1 [Easy] What is IDistributedCache and how do you set it up with Redis?
**Answer**
An abstraction for a shared cache that stores bytes/strings by key with expiration. The Redis implementation is added with `AddStackExchangeRedisCache`. It hides the Redis client but exposes limited features.
**Explanation**
It prefixes keys with `InstanceName`. It stores entries as hashes (with absolute expiration/sliding data) internally, so keys are not plain strings - don't read them with raw `GET`. Great for portability, poor for Redis-specific features (locks, sorted sets).
**Code / CLI / Bicep**
```csharp
builder.Services.AddStackExchangeRedisCache(o =>
{
    o.Configuration = "mycache.redis.cache.windows.net:6380,password=...,ssl=True,abortConnect=False";
    o.InstanceName = "app1:";
});

await cache.SetStringAsync("greeting", "hi", new DistributedCacheEntryOptions
{ AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5) });
var s = await cache.GetStringAsync("greeting");
```

### 7.2 [Follow-up] When use IDistributedCache vs raw IDatabase?
**Answer**
IDistributedCache for plain cache/session payloads where abstraction and swapability matter. Raw StackExchange.Redis `IDatabase` when you need data structures, atomic ops, scripts, pub/sub. Many apps use both, sharing the same multiplexer.
**Explanation**
Can share the multiplexer via `ConnectionMultiplexerFactory` option. Avoid mixing key namespaces with unexpected prefixes.
**Code / CLI / Bicep**
```csharp
builder.Services.AddSingleton<IConnectionMultiplexer>(sp => ConnectionMultiplexer.Connect(conn));
builder.Services.AddStackExchangeRedisCache(o =>
{
    o.ConnectionMultiplexerFactory = () => Task.FromResult(sp.GetRequiredService<IConnectionMultiplexer>());
});
// (build sp via a configured options pattern; shown for concept)
```

### 7.3 [Medium] What does HybridCache add, and how do you use it?
**Answer**
It combines in-memory L1 with a distributed L2, gives typed `GetOrCreateAsync`, and coalesces concurrent misses so the factory runs once per key per process (stampede protection). It works with Redis as L2 via the registered IDistributedCache.
**Explanation**
Stampede protection is per-process (and L2 reads are shared) - with N instances you may still get up to N factory calls per cold key. L1 data can be stale relative to L2 on other instances until L1 expiry (no cross-node L1 invalidation by default - verify), so keep L1 expiration short.
**Code / CLI / Bicep**
```csharp
builder.Services.AddStackExchangeRedisCache(o => o.Configuration = conn);
builder.Services.AddHybridCache(o =>
{
    o.DefaultEntryOptions = new HybridCacheEntryOptions
    {
        Expiration = TimeSpan.FromMinutes(10),            // L2
        LocalCacheExpiration = TimeSpan.FromMinutes(1)    // L1
    };
});

var p = await hybrid.GetOrCreateAsync($"product:v1:{id}",
    async ct => await repo.GetAsync(id, ct),
    cancellationToken: ct);
await hybrid.RemoveAsync($"product:v1:{id}");
```
**Gotchas / follow-ups**
- Keys must not contain user-controlled unbounded garbage (key length limits).
- Cached type must be serializable by the configured serializer.

### 7.4 [Hard] Choose a serializer for cached objects and discuss versioning.
**Answer**
System.Text.Json: easy, readable, bigger, tolerant to added fields. MessagePack/protobuf: compact and fast but need stable contracts (protobuf field numbers; MessagePack keys). Always include a version in the key or payload so deploys with changed shapes don't read old-format entries.
**Explanation**
Serialization version break scenario: new release renames a property; old entries deserialize with nulls/defaults or throw. Solutions: bump key version (`v2`), tolerate missing fields, treat deserialization failure as a miss (and delete the key), roll out additive changes only. Compression (e.g. Brotli/Gzip) helps for large values at CPU cost; keep values small (ideally KBs, avoid >100KB-1MB).
**Code / CLI / Bicep**
```csharp
async Task<T?> TryGetAsync<T>(string key)
{
    var raw = await _db.StringGetAsync(key);
    if (!raw.HasValue) return default;
    try { return JsonSerializer.Deserialize<T>((byte[])raw!); }
    catch (JsonException) { await _db.KeyDeleteAsync(key); return default; } // treat as miss
}

// MessagePack
byte[] bytes = MessagePackSerializer.Serialize(dto);
```
**Gotchas / follow-ups**
- Don't use BinaryFormatter (insecure, removed).
- Polymorphic deserialization of untrusted cache data = security risk; cache content should be trusted but limit type handling.

### 7.5 [Tricky] Why might two app versions in a rolling deploy corrupt each other's cache usage?
**Answer**
During rolling/blue-green deploys both versions share the Redis keyspace but may expect different payload shapes. Version the keys (or payloads) so each version reads only its own format.
**Explanation**
Key pattern `app:entity:v{schemaVersion}:id` isolates versions; old keys expire by TTL. Alternatively use a separate cache per slot - but then deployment slot swaps need warm-up. Also watch for Functions slots sharing the same app settings.
**Code / CLI / Bicep**
```csharp
const int SchemaVersion = 4;
string Key(int id) => $"shop:product:v{SchemaVersion}:{id}";
```

**Interview traps**
- Assuming IDistributedCache has stampede protection (it doesn't; HybridCache does).
- Assuming HybridCache L1 invalidates across nodes automatically.
- Reading IDistributedCache keys with raw Redis GET.

**Cheat sheet**
- IDistributedCache = bytes + expiry; HybridCache = typed, L1+L2, coalescing.
- Short L1 TTL.
- Version keys/payloads; deserialize failure = miss.
- Prefer compact serializers for big/hot data.

---

## Topic 8: Invalidation, stampede, penetration, avalanche

**Study notes**
"There are only two hard things: cache invalidation and naming things."

| Problem | Meaning | Mitigation |
|---|---|---|
| Stampede / thundering herd / dogpile | hot key expires, many requests hit DB at once | single-flight lock, HybridCache, early/probabilistic refresh, serve-stale |
| Penetration | requests for non-existent keys always miss | negative caching (short TTL), Bloom filter, validation |
| Avalanche | many keys expire together (or cache restarts) | jittered TTL, staggered warmup, multi-layer |
| Breakdown (hotspot) | single hot key | never expire hot keys + background refresh, L1 cache |

Invalidation strategies: TTL only; delete on write; event-driven (Service Bus/Event Grid/Change Feed -> delete keys); **versioned keys / namespace versions** (bump a version number instead of deleting many keys).

### 8.1 [Easy] What is a cache stampede?
**Answer**
When a popular key expires, many concurrent requests miss simultaneously and all hit the database. It can overload the backing store. Mitigate with single-flight locking, jittered TTLs and early refresh.
**Explanation**
Within one process, SemaphoreSlim per key or HybridCache coalesces. Across instances, use a short Redis lock (SET NX PX) so one rebuilds while others wait briefly or serve stale data.
**Code / CLI / Bicep**
```csharp
static readonly ConcurrentDictionary<string, SemaphoreSlim> _locks = new();
async Task<T> GetOrLoadAsync<T>(string key, Func<Task<T>> load)
{
    var cached = await TryGetAsync<T>(key);
    if (cached is not null) return cached;
    var gate = _locks.GetOrAdd(key, _ => new SemaphoreSlim(1, 1));
    await gate.WaitAsync();
    try
    {
        cached = await TryGetAsync<T>(key);           // double-check
        if (cached is not null) return cached;
        var v = await load();
        await _db.StringSetAsync(key, JsonSerializer.Serialize(v), TimeSpan.FromMinutes(10));
        return v;
    }
    finally { gate.Release(); }
}
```

### 8.2 [Follow-up] Explain penetration and negative caching.
**Answer**
Penetration is repeated lookups for data that doesn't exist (bug or attack), bypassing the cache every time. Cache a "not found" marker with a short TTL, validate input, or use a Bloom filter in front.
**Explanation**
Keep negative TTL short (30-60s) so newly created items appear soon; or invalidate on create. Use a sentinel, not null, because Redis can't store null.
**Code / CLI / Bicep**
```csharp
const string NotFound = "__nf__";
var v = await _db.StringGetAsync(key);
if (v == NotFound) return null;
var item = await _repo.FindAsync(id);
await _db.StringSetAsync(key, item is null ? NotFound : Serialize(item),
                         item is null ? TimeSpan.FromSeconds(30) : TimeSpan.FromMinutes(10));
```

### 8.3 [Medium] How do you invalidate many related keys (e.g. all cached pages for a tenant)?
**Answer**
Avoid scanning and deleting. Use a version/namespace token in the key: bump `tenant:{id}:ver` and old keys become unreachable and expire by TTL. Alternatively maintain a set of keys per group, or use HybridCache tags where supported (verify).
**Explanation**
SCAN+DEL is O(N) over the whole keyspace and non-atomic. Versioned namespace makes invalidation O(1) at the cost of orphan keys until TTL (memory).
**Code / CLI / Bicep**
```csharp
async Task<string> NsKey(Guid tenant, string suffix)
{
    var ver = await _db.StringGetAsync($"ns:{tenant:N}") ;
    var v = ver.HasValue ? (long)ver : 0;
    return $"app:t:{tenant:N}:v{v}:{suffix}";
}
Task InvalidateTenant(Guid t) => _db.StringIncrementAsync($"ns:{t:N}");
```
**Gotchas / follow-ups**
- Namespace key itself must not expire/evict (use volatile policy carefully or give it a long TTL and handle absence).

### 8.4 [Hard] Describe probabilistic early expiration (XFetch) and serve-stale.
**Answer**
On each read, with probability increasing as expiry nears, a request recomputes the value early so the key rarely actually expires. Serve-stale keeps an old copy beyond logical TTL and refreshes in the background. Both avoid synchronized misses without a global lock.
**Explanation**
XFetch: recompute if `now - delta * beta * ln(rand) >= expiry`, where delta is recompute cost. Implementation: store value + logical expiry + delta; physical TTL is longer than logical TTL.
**Code / CLI / Bicep**
```csharp
bool ShouldRefresh(DateTimeOffset expiry, TimeSpan delta, double beta = 1.0)
{
    var r = Random.Shared.NextDouble();
    return DateTimeOffset.UtcNow - delta * (beta * Math.Log(r)) >= expiry;  // log(r) <= 0
}
```
**Gotchas / follow-ups**
- TimeSpan multiplication: `delta * double` is supported in .NET Core 3.0+.
- Clock skew across nodes: store expiry from a single source or accept error.

### 8.5 [Tricky] Avalanche after a Redis restart/failover - how do you survive it?
**Answer**
A cold cache sends all traffic to the DB. Use L1 in-memory cache, request coalescing, rate limiting/bulkheads on DB calls, circuit breakers, and a warm-up of critical keys; size the DB for cold cache or degrade gracefully.
**Explanation**
Add jitter to TTLs so keys don't expire together, and stagger deploys. Persistence (RDB/AOF) can restore data after restart but not after all failover scenarios; don't rely on it.
**Code / CLI / Bicep**
```csharp
TimeSpan Ttl(TimeSpan baseTtl) => baseTtl + TimeSpan.FromSeconds(Random.Shared.NextDouble() * baseTtl.TotalSeconds * 0.1);
```

**Interview traps**
- Mixing up stampede (one hot key) vs avalanche (many keys) vs penetration (missing keys).
- Locks without timeouts causing request pile-ups.
- Invalidate by SCAN in production.

**Cheat sheet**
- Stampede: single-flight/HybridCache/XFetch.
- Penetration: negative cache/Bloom.
- Avalanche: jitter + coalescing + DB protection.
- Mass invalidation: version token.

---

## Topic 9: Distributed locks

**Study notes**
Basic lock: `SET lock:key <unique-token> NX PX 30000`. Release only if the token matches (Lua compare-and-delete). StackExchange.Redis wraps this: `LockTakeAsync(key, token, expiry)` / `LockReleaseAsync(key, token)` / `LockExtendAsync`.

Lock safety: TTL prevents deadlock but a paused holder (GC, network) may continue after expiry -> two holders. Use **fencing tokens** (monotonic counter checked by the protected resource) when correctness matters. Redlock (acquire across N independent Redis masters) is debated (Kleppmann vs Antirez): it doesn't give fencing guarantees; for correctness-critical mutual exclusion prefer a consensus system or DB constraints/optimistic concurrency; Redis locks are fine for efficiency (avoid duplicate work).

Azure alternatives: **Blob lease** (60s max or infinite), Service Bus sessions, Durable Functions entities, DB row locks/ETags.

### 9.1 [Easy] How do you implement a simple distributed lock with Redis?
**Answer**
`SET key token NX PX ttl` acquires only if absent and auto-expires. Release by deleting only if the stored token equals yours. In .NET use `LockTakeAsync` and `LockReleaseAsync`.
**Explanation**
Unique token (GUID) prevents releasing someone else's lock after yours expired.
**Code / CLI / Bicep**
```csharp
var token = Guid.NewGuid().ToString();
if (await db.LockTakeAsync("lock:invoice:42", token, TimeSpan.FromSeconds(30)))
{
    try { await DoWorkAsync(); }
    finally { await db.LockReleaseAsync("lock:invoice:42", token); }
}
else { /* someone else is working; skip or retry */ }
```

### 9.2 [Follow-up] Why must release check the token, and how is it atomic?
**Answer**
Otherwise a client whose lock expired could delete the new owner's lock. Check-and-delete must be atomic via Lua (or a WATCH transaction), because GET then DEL has a race.
**Explanation**
`LockReleaseAsync` in StackExchange.Redis does this with a transaction condition. Manual Lua version shown.
**Code / CLI / Bicep**
```csharp
const string Unlock = "if redis.call('get',KEYS[1])==ARGV[1] then return redis.call('del',KEYS[1]) else return 0 end";
await db.ScriptEvaluateAsync(Unlock, new RedisKey[]{"lock:x"}, new RedisValue[]{token});
```

### 9.3 [Medium] The work takes longer than the lock TTL. What do you do?
**Answer**
Extend the lock periodically (a watchdog) while the work runs, only if you still hold the token, or size TTL conservatively and make the work idempotent. If extension fails, abort the work.
**Explanation**
Extension must also be token-checked (`LockExtendAsync`). Always make protected operations idempotent since locks can be lost.
**Code / CLI / Bicep**
```csharp
using var cts = new CancellationTokenSource();
var renew = Task.Run(async () =>
{
    while (!cts.IsCancellationRequested)
    {
        await Task.Delay(TimeSpan.FromSeconds(10), cts.Token);
        if (!await db.LockExtendAsync(key, token, TimeSpan.FromSeconds(30))) { /* lost lock: cancel work */ break; }
    }
});
```

### 9.4 [Hard] Is a Redis lock safe? Explain Redlock and fencing tokens.
**Answer**
A single-instance Redis lock is not fully safe: replication is asynchronous (failover can lose the lock) and process pauses can outlive the TTL. Redlock uses majority of independent nodes but still relies on timing assumptions and has no fencing token. For strict correctness use fencing tokens validated by the resource, or a lease-based store with proper consensus.
**Explanation**
Fencing: each lock acquisition returns an incrementing number (INCR); the storage rejects writes with a lower number than seen. On Azure, a managed Redis HA pair is not independent masters, so Redlock isn't straightforwardly applicable. For "only one instance runs this timer job", a Blob lease is simpler and stronger.
**Code / CLI / Bicep**
```csharp
long fence = await db.StringIncrementAsync("fence:invoice:42");
// pass 'fence' to DB: UPDATE ... SET ..., LastFence=@f WHERE Id=@id AND LastFence < @f
```
```bash
# Blob lease alternative
az storage blob lease acquire -c locks -b job1 --lease-duration 60 --connection-string "$CS"
```
**Gotchas / follow-ups**
- Lock lost on failover: both nodes think they own it.
- Clock jumps affect Redlock validity.

**Interview traps**
- Lock without token / without TTL.
- Believing Redlock gives correctness guarantees.
- Not making work idempotent.

**Cheat sheet**
- SET NX PX + token + Lua release.
- Extend with watchdog, check token.
- Efficiency lock = Redis OK; correctness = fencing/DB/lease.

---

## Topic 10: Rate limiting, counters, leaderboards

**Study notes**
| Algorithm | Redis implementation | Notes |
|---|---|---|
| Fixed window | `INCR key` + `EXPIRE` on first hit | simple, boundary burst (2x at window edge) |
| Sliding window log | sorted set of timestamps: ZREMRANGEBYSCORE, ZCARD, ZADD | accurate, memory per request |
| Sliding window counter | two fixed windows weighted | cheap approx |
| Token bucket | hash (tokens, ts) in Lua | allows bursts |

Leaderboards: sorted set `ZADD/ZINCRBY/ZREVRANGE/ZREVRANK`. Counters: `INCR/INCRBY/HINCRBY`.
Also see built-in ASP.NET Core rate limiting middleware (in-memory per instance; Redis needed for global limits).

### 10.1 [Easy] Implement fixed-window rate limiting.
**Answer**
INCR a per-client per-window key and set an expiry on first increment; reject when count exceeds limit. Make INCR + EXPIRE atomic to avoid immortal keys.
**Explanation**
If the app crashes between INCR and EXPIRE the key never expires. Use a Lua script or transaction.
**Code / CLI / Bicep**
```csharp
const string Script = @"
local c = redis.call('INCR', KEYS[1])
if c == 1 then redis.call('PEXPIRE', KEYS[1], ARGV[1]) end
return c";
var window = DateTimeOffset.UtcNow.ToUnixTimeSeconds() / 60;
var count = (long)await db.ScriptEvaluateAsync(Script,
    new RedisKey[]{$"rl:{clientId}:{window}"}, new RedisValue[]{60000});
bool allowed = count <= 100;
```

### 10.2 [Follow-up] Fixed window's weakness and the sliding window fix?
**Answer**
A client can send the full limit at the end of one window and again at the start of the next (2x burst). Sliding window uses a sorted set of timestamps (or weighted counters) for a true rolling limit at higher memory/CPU cost.
**Explanation**
Sorted-set approach: remove entries older than window, count remaining, add current if under limit - all in Lua for atomicity.
**Code / CLI / Bicep**
```csharp
const string Sliding = @"
redis.call('ZREMRANGEBYSCORE', KEYS[1], 0, ARGV[1] - ARGV[2])
local n = redis.call('ZCARD', KEYS[1])
if n < tonumber(ARGV[3]) then
  redis.call('ZADD', KEYS[1], ARGV[1], ARGV[4])
  redis.call('PEXPIRE', KEYS[1], ARGV[2])
  return 1
end
return 0";
var now = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds();
var ok = (int)await db.ScriptEvaluateAsync(Sliding, new RedisKey[]{$"rl:{id}"},
    new RedisValue[]{now, 60000, 100, Guid.NewGuid().ToString()}) == 1;
```

### 10.3 [Medium] Build a leaderboard.
**Answer**
Use a sorted set: ZADD/ZINCRBY to update scores, ZREVRANGE for top N, ZREVRANK for a user's rank. All are O(log N)-ish, making it very efficient.
**Explanation**
Ties are broken lexicographically by member. For huge leaderboards shard by time window (daily/weekly keys with TTL).
**Code / CLI / Bicep**
```csharp
await db.SortedSetIncrementAsync("lb:2026-10", "asha", 50);
var top = await db.SortedSetRangeByRankWithScoresAsync("lb:2026-10", 0, 9, Order.Descending);
long? rank = await db.SortedSetRankAsync("lb:2026-10", "asha", Order.Descending);
```

### 10.4 [Hard] Rate limiting in a cluster / with Redis down - what are the issues?
**Answer**
All keys for one limiter must hit one slot (they do if one key; use hash tags for multiple). If Redis is down, decide fail-open (availability) vs fail-closed (protection); typically fail-open with a local in-memory fallback limiter.
**Explanation**
Each Redis round trip adds latency to every request; consider local token prefetch/batching. Clock source: use Redis `TIME` inside Lua (allowed in Redis 5+ with effects replication) or pass app time carefully.
**Code / CLI / Bicep**
```csharp
try { allowed = await RedisLimiter.TryAsync(id); }
catch (RedisException) { allowed = LocalLimiter.TryAcquire(id); } // degraded mode
```
**Gotchas / follow-ups**
- Lua scripts touching undeclared keys break in cluster.
- Hot client key = hot shard.

**Interview traps**
- INCR then EXPIRE non-atomically.
- Ignoring fail-open/closed decision.
- Using in-memory limiter across scaled-out instances and expecting global limits.

**Cheat sheet**
- Fixed: INCR+EXPIRE (Lua). Sliding: ZSET+Lua.
- Leaderboard: ZINCRBY/ZREVRANGE.
- Decide fail-open vs closed.

---

## Topic 11: Pub/Sub vs Streams vs Service Bus; session and output cache

**Study notes**
| Feature | Redis Pub/Sub | Redis Streams | Azure Service Bus |
|---|---|---|---|
| Delivery | fire-and-forget, at-most-once | persisted log, consumer groups, at-least-once (XACK) | durable broker, at-least-once / exactly-once-ish via sessions/dedup |
| Offline subscriber | message lost | can read later | queued |
| DLQ, retries, scheduling, sessions, transactions | no | manual (XPENDING/XCLAIM) | yes |
| Retention | none | bounded by memory/MAXLEN | up to TTL, GBs |
| Use | cache invalidation broadcast, ephemeral signals | lightweight event log/work queues | business-critical messaging |

Redis data is in memory and may be lost on failover, so Redis messaging is not a substitute for Service Bus on critical flows.

Session state: `AddStackExchangeRedisCache` + `AddSession`. Output caching: `AddStackExchangeRedisOutputCache` (.NET 8+) stores full responses.

### 11.1 [Easy] What is Redis Pub/Sub and its limitation?
**Answer**
Publishers send messages to channels; current subscribers receive them instantly. If no one is subscribed (or a subscriber disconnects), the message is lost. It's at-most-once with no persistence.
**Explanation**
Good for broadcasting cache-invalidation notices to L1 caches on all instances, where a miss only costs freshness bounded by TTL.
**Code / CLI / Bicep**
```csharp
var sub = mux.GetSubscriber();
await sub.SubscribeAsync(RedisChannel.Literal("inval"), (ch, msg) => localCache.Remove((string)msg!));
await sub.PublishAsync(RedisChannel.Literal("inval"), "product:v1:42");
```

### 11.2 [Follow-up] What are Redis Streams and consumer groups?
**Answer**
A stream is an append-only log (XADD). Consumer groups let multiple consumers share work: each entry goes to one consumer, which must XACK it; unacked entries stay in the pending list for retry/claim.
**Explanation**
Gives at-least-once semantics, so consumers must be idempotent. Trim with MAXLEN to bound memory. Reclaim stuck messages with XAUTOCLAIM/XCLAIM.
**Code / CLI / Bicep**
```csharp
await db.StreamAddAsync("orders", "payload", json, maxLength: 100_000, useApproximateMaxLength: true);
try { await db.StreamCreateConsumerGroupAsync("orders", "workers", StreamPosition.NewMessages); } catch (RedisServerException) { /* exists */ }
var entries = await db.StreamReadGroupAsync("orders", "workers", "c1", ">", count: 10);
foreach (var e in entries) { await Handle(e); await db.StreamAcknowledgeAsync("orders", "workers", e.Id); }
```

### 11.3 [Medium] Redis vs Service Bus for messaging - how do you choose?
**Answer**
Service Bus for durable, business-critical messaging needing DLQ, retries, sessions, scheduling, transactions and geo-DR. Redis Streams/pub-sub for low-latency, ephemeral or best-effort signals where loss is tolerable. Mixing: cache invalidation via pub/sub, order processing via Service Bus.
**Explanation**
Service Bus is integrated with Functions triggers, managed identity and monitoring. Redis stream processing needs you to build poison handling, scaling and monitoring.
**Code / CLI / Bicep**
```csharp
// Durable business event -> Service Bus
await sender.SendMessageAsync(new ServiceBusMessage(json) { MessageId = orderId });
```

### 11.4 [Hard] Configure session state and output cache with Redis; what are the gotchas?
**Answer**
Register Redis via `AddStackExchangeRedisCache`, then `AddSession`; for output cache use `AddStackExchangeRedisOutputCache`. Sessions are cache entries so can be evicted; keep them small and treat them as losable. Output caching must vary correctly or you leak per-user content.
**Explanation**
Session cookies hold only the id; data lives in Redis (loaded per request, saved at end). Concurrent requests for one session can race (last writer wins). Authenticated responses are not cached by default in output cache; don't override without vary-by user.
**Code / CLI / Bicep**
```csharp
builder.Services.AddStackExchangeRedisCache(o => o.Configuration = conn);
builder.Services.AddSession(o => { o.IdleTimeout = TimeSpan.FromMinutes(20); o.Cookie.HttpOnly = true; o.Cookie.IsEssential = true; });

builder.Services.AddStackExchangeRedisOutputCache(o => o.Configuration = conn);
builder.Services.AddOutputCache(o => o.AddPolicy("Products", b => b.Expire(TimeSpan.FromMinutes(1)).Tag("products")));
app.UseSession(); app.UseOutputCache();
// Invalidate: await store.EvictByTagAsync("products", ct);
```
**Gotchas / follow-ups**
- Eviction logs users out - use sufficient memory/policy.
- Prefer stateless tokens over server sessions when possible.

**Interview traps**
- Using pub/sub for must-not-lose messages.
- Streams without idempotent consumers or MAXLEN.
- Caching personalized output without vary.

**Cheat sheet**
- Pub/sub = ephemeral; Streams = log + groups; Service Bus = durable enterprise.
- XACK + XAUTOCLAIM.
- Session/output cache via Redis extensions.

---

## Topic 12: Transactions, Lua, pipelining, clustering and hash slots

**Study notes**
- `MULTI/EXEC`: queued commands run atomically (no interleaving), **no rollback** on runtime errors; `WATCH` gives optimistic locking.
- **Lua** (`EVAL`/`EVALSHA`): atomic script; blocks the server while running - keep short.
- **Pipelining**: send many commands without waiting for replies (not atomic).
- **Clustering**: keyspace split into **16384 hash slots** (CRC16(key) mod 16384), each owned by a shard. Multi-key operations (MGET, transactions, Lua, SUNION) require all keys in the **same slot**, else `CROSSSLOT` error. **Hash tags**: only the part in `{}` is hashed - `{user:42}:cart` and `{user:42}:profile` share a slot.

```
key -> CRC16 -> slot 0..16383 -> shard A (0-5460) | B (5461-10922) | C (10923-16383)
```

### 12.1 [Easy] What is a Redis transaction?
**Answer**
MULTI queues commands and EXEC runs them atomically without other clients' commands interleaving. It doesn't roll back if one command fails at runtime. Use WATCH-based conditions for check-and-set.
**Explanation**
In StackExchange.Redis, `CreateTransaction()` buffers commands and `ExecuteAsync()` sends them with conditions; don't await individual commands before Execute (deadlock).
**Code / CLI / Bicep**
```csharp
var tx = db.CreateTransaction();
tx.AddCondition(Condition.KeyExists("acct:1"));
_ = tx.HashDecrementAsync("acct:1", "bal", 10);
_ = tx.HashIncrementAsync("acct:2", "bal", 10);
bool ok = await tx.ExecuteAsync();
```

### 12.2 [Follow-up] When use Lua over a transaction?
**Answer**
When logic depends on values read within the operation (read-modify-write with branching), since a transaction can't branch. Lua runs atomically on the server in one round trip. Keep scripts short and pass keys through KEYS.
**Explanation**
StackExchange.Redis `LuaScript.Prepare` supports named parameters and caches via EVALSHA. Redis scripts block the main thread; avoid loops over big sets.
**Code / CLI / Bicep**
```csharp
var script = LuaScript.Prepare("local v = redis.call('GET', @key) if v == @expected then redis.call('SET', @key, @new) return 1 end return 0");
var r = (int)await db.ScriptEvaluateAsync(script, new { key = (RedisKey)"cfg", expected = "a", @new = "b" });
```

### 12.3 [Medium] What is a CROSSSLOT error and how do you fix it?
**Answer**
In cluster mode, a multi-key command whose keys hash to different slots fails with `CROSSSLOT Keys in request don't hash to the same slot`. Fix by using hash tags so related keys share a slot, or split the operation per slot.
**Explanation**
Overusing one hash tag (e.g. `{tenant}`) piles data on one shard (hot shard). Design tags at the finest granularity that still groups what must be atomic.
**Code / CLI / Bicep**
```csharp
var k1 = (RedisKey)"{order:42}:items";
var k2 = (RedisKey)"{order:42}:status";
await db.StringGetAsync(new[] { k1, k2 });   // same slot - OK
```
```bash
redis-cli -c -h host -p 10000 CLUSTER KEYSLOT "{order:42}:items"
```

### 12.4 [Hard] Clustering behaviors on Azure to watch for (verify per tier).
**Answer**
Azure Premium cluster and AMR expose different cluster endpoints/policies; OSS-cluster mode needs a cluster-aware client (StackExchange.Redis is), while an enterprise proxy mode presents a single endpoint (AMR "Enterprise cluster policy" - verify). Scaling shards causes slot migration and brief latency; keys may get `MOVED/ASK` redirects handled by the client.
**Explanation**
SCAN/KEYS must be run per shard. Database selection (`SELECT n`) is not supported in cluster mode (only db 0). Pub/sub in clusters broadcasts across nodes. Test client behavior during scale operations.
**Code / CLI / Bicep**
```bash
az redis create -n c1 -g rg -l westeurope --sku Premium --vm-size p1 --shard-count 3
az redis update -n c1 -g rg --set shardCount=4     # scale out (causes slot rebalancing)
```
**Gotchas / follow-ups**
- `defaultDatabase` other than 0 fails on clusters.
- Lua with undeclared keys = cross-slot risk.

**Interview traps**
- "EXEC rolls back" - no.
- Hash tag everything with one tag = hot shard.
- Using KEYS across shard cluster.

**Cheat sheet**
- 16384 slots; same slot for multi-key ops.
- Hash tag `{x}`.
- Lua for branching atomic logic.
- Transactions: no rollback.

---

## Topic 13: Persistence, durability, HA, failover

**Study notes**
Redis is a cache first: it may lose data on eviction, restart, failover. Persistence (RDB snapshots / AOF) reduces loss but replication is asynchronous.

Azure HA (Standard+): primary + replica; Microsoft patches/updates the nodes with failover. Expect **connection drops and a few seconds of errors** during planned maintenance and failover (client must reconnect). Use zone redundancy for AZ failures (verify tiers). Schedule patching windows where available (verify).

```
App --> Primary ==async repl==> Replica
         |  failover: replica promoted; clients reconnect; recent unreplicated writes may be lost
```

### 13.1 [Easy] Is Redis a database? Can you rely on its data?
**Answer**
It can persist data, but on Azure treat it as a cache: data can be evicted, lost on failover, or flushed. The source of truth must be a durable store like SQL/Cosmos/Blob.
**Explanation**
Even with AOF, up to ~1 second of writes can be lost and memory limits cause evictions. Using it as a primary store requires AMR/Enterprise features and a careful durability review (verify).
**Code / CLI / Bicep**
```csharp
// Cache-miss path always rebuilds from the source of truth
var v = await cache.GetAsync(key) ?? await repo.LoadAndCacheAsync(key);
```

### 13.2 [Follow-up] What happens to clients during a failover or patch?
**Answer**
Connections break for seconds, commands fail/time out, then the client reconnects to the promoted node. Resilient apps retry transient failures, use abortConnect=false, and fail open to the source.
**Explanation**
DNS name stays the same; the client re-resolves. Keep client libraries current. Use Azure "Reboot"/failover test in non-prod to validate.
**Code / CLI / Bicep**
```bash
az redis force-reboot -n mycache -g rg --reboot-type PrimaryNode   # test failover in non-prod only
```
```csharp
var policy = Policy.Handle<RedisException>().WaitAndRetryAsync(3, i => TimeSpan.FromMilliseconds(100 * Math.Pow(2, i)) + TimeSpan.FromMilliseconds(Random.Shared.Next(0, 100)));
```

### 13.3 [Medium] Replication lag and read replicas.
**Answer**
Replication is asynchronous, so replicas can serve stale data and a failover can drop the latest writes. Don't read-your-writes from a replica; if using `CommandFlags.PreferReplica`, accept staleness.
**Explanation**
Azure Standard replica isn't directly addressable for reads in the basic model; clustered/AMR replicas may be. Use primary for correctness-critical reads (locks, counters).
**Code / CLI / Bicep**
```csharp
var v = await db.StringGetAsync(key, CommandFlags.PreferReplica);   // staleness OK only for cache reads
```

### 13.4 [Hard] Design a DR strategy for Redis-backed caching.
**Answer**
Since it's a cache, DR can be "rebuild": deploy an empty cache in the secondary region and let the app re-warm it, with a DB sized for cold start. If warm data is needed, use geo-replication (passive on Premium; active on Enterprise/AMR - verify) and test failover runbooks.
**Explanation**
Geo-replication cost and complexity are only justified when a cold cache cannot be tolerated. Keep connection strings in Key Vault/app config so repointing is a config change; validate TLS/auth in the secondary.
**Code / CLI / Bicep**
```bash
az redis server-link create -n primary -g rg --server-to-link secondaryId --replication-role Secondary
```
**Gotchas / follow-ups**
- Geo-failover may need unlinking and DNS/connection changes.
- Persisted RDB in storage is not DR for in-flight writes.

**Interview traps**
- Treating Redis as the system of record.
- No retry/fallback for failover blips.
- Assuming sync replication.

**Cheat sheet**
- Cache = disposable; rebuild from source.
- Expect brief errors on patch/failover.
- Replication async; stale replica reads.
- DR = empty cache + warm-up, or geo-replication.

---

## Topic 14: Security - keys vs Entra ID, TLS, private endpoints

**Study notes**
- **Access keys** (primary/secondary): simple, full access, must be rotated; store in Key Vault; don't commit.
- **Microsoft Entra ID auth** (token-based, supports managed identities SAMI/UAMI and RBAC-like "data access policies"; available on ACR and AMR - verify tier support). Tokens expire (~1 hour); the client must re-authenticate/refresh; the **Microsoft.Azure.StackExchangeRedis** NuGet package extends StackExchange.Redis `ConfigurationOptions` to acquire/refresh tokens (methods like `ConfigureForAzureWithUserAssignedManagedIdentityAsync` / `ConfigureForAzureWithSystemAssignedManagedIdentityAsync` - verify current signatures).
- TLS: port 6380 (non-TLS 6379 disabled by default), min TLS 1.2.
- Network: public access + firewall rules, **private endpoint** (Private Link, `privatelink.redis.cache.windows.net` DNS zone - verify for AMR), or VNet injection (Premium, legacy).

### 14.1 [Easy] How do apps authenticate to Azure Redis?
**Answer**
With access keys in the connection string, or with Microsoft Entra ID tokens via a managed identity. Entra auth is preferred since there are no secrets to store or rotate. Both use TLS.
**Explanation**
Entra auth may need enabling on the cache (`aad-enabled`) and creating a data access policy assignment for the identity (verify exact property names). Disable access-key auth once all clients migrate.
**Code / CLI / Bicep**
```bash
az redis update -n mycache -g rg --set "redisConfiguration.aad-enabled=true"
az redis access-policy-assignment create -n mycache -g rg --policy-name "Data Contributor" \
  --access-policy-assignment-name myapp --object-id <principalId> --object-id-alias myapp
# verify commands/flags in current az CLI docs
```

### 14.2 [Follow-up] Show Entra ID (managed identity) auth from .NET.
**Answer**
Use `Microsoft.Azure.StackExchangeRedis` to configure the options with a managed identity; it fetches and renews tokens. The username is the identity's object (principal) ID.
**Explanation**
UAMI needs client id; SAMI works without. Local dev can use developer credentials (Azure CLI/VS) via `DefaultAzureCredential`.
**Code / CLI / Bicep**
```csharp
var opts = await ConfigurationOptions.Parse("mycache.redis.cache.windows.net:6380")
    .ConfigureForAzureWithUserAssignedManagedIdentityAsync(clientId);   // verify signature in current package
opts.AbortOnConnectFail = false;
var mux = await ConnectionMultiplexer.ConnectAsync(opts);
```

### 14.3 [Medium] Token expiry - what goes wrong?
**Answer**
Tokens expire (~1h); an old client that authenticated once keeps its connection until it drops, then reconnect with an expired token fails with `NOAUTH`/`WRONGPASS`. The Azure extension package refreshes and re-authenticates connections; custom code must do it itself.
**Explanation**
Symptoms: errors appearing roughly hourly or after a failover. Ensure the client library version supports re-auth, and that managed identity assignment propagates (role assignments can take minutes). Verify current guidance on token refresh behaviours.
**Code / CLI / Bicep**
```csharp
// Manual token (concept) - prefer the Azure extension package
var token = await new DefaultAzureCredential().GetTokenAsync(
    new TokenRequestContext(new[] { "https://redis.azure.com/.default" }));   // scope: verify
```

### 14.4 [Hard] Lock the cache down: networking and secrets checklist.
**Answer**
Disable public network access, use a private endpoint with the private DNS zone, require TLS 1.2+, disable non-TLS port, use Entra auth (or Key Vault-held keys with rotation), and restrict by identity/policy. Monitor with diagnostic logs.
**Explanation**
Private endpoint DNS misconfiguration is the most common failure: the name resolves to the public IP and gets blocked. Functions need VNet integration (regional) to reach private endpoints. Key rotation: regenerate secondary, switch apps, regenerate primary.
**Code / CLI / Bicep**
```bicep
resource redis 'Microsoft.Cache/redis@2024-03-01' = {
  name: 'myredis'
  location: resourceGroup().location
  properties: {
    sku: { name: 'Premium', family: 'P', capacity: 1 }
    publicNetworkAccess: 'Disabled'
    enableNonSslPort: false
    minimumTlsVersion: '1.2'
    redisConfiguration: { 'aad-enabled': 'true' }
  }
}
// plus Microsoft.Network/privateEndpoints + privateDnsZones privatelink.redis.cache.windows.net (verify)
```
**Gotchas / follow-ups**
- Keys in app settings = visible to contributors; use Key Vault references.
- Rotate keys without downtime using two keys.

**Interview traps**
- Leaving access-key auth on while claiming "passwordless".
- Forgetting VNet integration for Functions.
- Token refresh ignored in custom code.

**Cheat sheet**
- Entra ID + MI preferred; keys in Key Vault otherwise.
- TLS 1.2, 6380.
- Private endpoint + DNS zone.
- Rotate via secondary key first.

---

## Topic 15: Performance troubleshooting, sizing, cost

**Study notes**
Azure metrics (Azure Monitor): **Used Memory / Used Memory RSS**, **Connected Clients**, **CPU**, **Server Load** (how busy the single Redis thread is; sustained >80% risks latency), **Cache Hits / Misses**, **Evicted Keys**, **Expired Keys**, **Cache Read/Write (bandwidth)**, **Operations per Second**, **Errors** (failover, etc.). Alert on server load, memory %, evictions, connected clients.

Redis tools: `SLOWLOG GET`, `INFO` (memory, stats, clients, commandstats), `CLIENT LIST`, `LATENCY DOCTOR` (verify availability), `MEMORY USAGE`, `--bigkeys`, `MONITOR` (avoid in prod: heavy), `redis-benchmark`.

Fragmentation ratio = used_memory_rss / used_memory; >1.5 = fragmentation; <1 = swapping.

### 15.1 [Easy] Which metrics do you watch first?
**Answer**
Server Load, CPU, Used Memory %, Evicted Keys, Connected Clients, and cache hit/miss. Together they tell if Redis is CPU-bound, memory-bound, or inefficient.
**Explanation**
High evictions => undersized or wrong TTLs. High misses => bad keys/TTLs. High server load => slow commands/hot keys or needs scaling/sharding.
**Code / CLI / Bicep**
```bash
az monitor metrics list --resource <redisId> --metric "serverLoad" "usedmemorypercentage" "evictedkeys" "connectedclients" --interval PT1M
```

### 15.2 [Follow-up] How do you find a slow or heavy command?
**Answer**
Run SLOWLOG GET to see slow commands and `INFO commandstats` for per-command totals; `CLIENT LIST` shows connections and their last commands. Fix by replacing O(N) commands, reducing value sizes and batching.
**Explanation**
The slowlog threshold is configurable (`slowlog-log-slower-than`, microseconds). Commands such as KEYS, HGETALL on large hashes, SMEMBERS, big MGET, large Lua are common culprits.
**Code / CLI / Bicep**
```bash
redis-cli -h host -p 6380 --tls -a $KEY SLOWLOG GET 10
redis-cli ... INFO commandstats
redis-cli ... CLIENT LIST
redis-cli ... --bigkeys
```

### 15.3 [Medium] KEYS vs SCAN, big keys and hot keys.
**Answer**
KEYS is O(N) and blocks the server; SCAN iterates incrementally with a cursor (may return duplicates, no snapshot guarantee). Big keys (MBs) cause latency and uneven memory; hot keys overload one shard. Split big keys, use L1 caching or replicate hot keys with suffixes.
**Explanation**
Hot key mitigation: local in-memory cache with short TTL, key sharding (`hot:{n}` for n in 0..K-1, random pick), HybridCache.
**Code / CLI / Bicep**
```csharp
var hotKey = $"hot:config:{Random.Shared.Next(0, 8)}";   // read from random replica key
```

### 15.4 [Hard] Latency spikes with low CPU. Diagnose.
**Answer**
Possibly large values/bandwidth limits (cache tier throttles network), fork during RDB/AOF rewrite, memory fragmentation/swap, client-side thread-pool starvation, many connections, or maintenance/failover events. Correlate Azure metrics with client exception details.
**Explanation**
Check Cache Read/Write bandwidth vs tier limits (verify), `used_memory_rss` vs `used_memory`, `latest_fork_usec`, `connected_clients`, and client CPU. Large Basic/Standard instances at high throughput hit network caps. Check `maxmemory-reserved` headroom.
**Code / CLI / Bicep**
```bash
redis-cli ... INFO memory   # mem_fragmentation_ratio
redis-cli ... INFO stats    # latest_fork_usec, evicted_keys
redis-cli ... LATENCY LATEST
```
**Gotchas / follow-ups**
- Azure restricts some commands (CONFIG partly, etc.) - verify per tier.
- Don't MONITOR in production.

### 15.5 [Tricky] How do you size and cost a cache?
**Answer**
Estimate: items x (avg value size + key + ~overhead 60-100B) x replication/headroom (keep ~20-30% free), then throughput (ops/s, bandwidth) vs tier limits, then HA/clustering needs. Load test with realistic payloads; verify prices and SKU limits in current docs.
**Explanation**
Over-provisioned memory is wasteful; under-provisioned causes evictions and misses that cost more at the DB. Compress/trim values, reduce TTLs, avoid caching rarely-used data. Compare AMR tiers (memory- vs compute-optimized - verify) for workload shape.
**Code / CLI / Bicep**
```bash
# rough memory estimate: 5M keys * (500B value + 80B key/overhead) ~ 2.9 GB -> with 30% headroom ~ 3.8 GB
redis-benchmark -h host -p 6380 --tls -a $KEY -t get,set -d 512 -n 200000 -c 50 -P 16
```

**Interview traps**
- Running KEYS/MONITOR in prod.
- Bumping timeouts without checking server load.
- Sizing only on data size, ignoring throughput/bandwidth.

**Cheat sheet**
- Metrics: server load, memory, evictions, clients, hit/miss.
- SLOWLOG, INFO, CLIENT LIST, bigkeys.
- Hot key => L1 / shard keys.
- Keep 20-30% memory headroom.

---

## Topic 16: Testing, local dev, Redis vs alternatives

**Study notes**
- Local: `docker run -p 6379:6379 redis:7` (or redis-stack). Connect without TLS locally.
- Integration tests: **Testcontainers for .NET** (`Testcontainers.Redis`) starts real Redis per test run - preferred because Lua/transactions/TTL behave realistically.
- Unit tests: abstract behind your own `ICache` interface or use `IDistributedCache` with `MemoryDistributedCache`; mocking `IDatabase` (NSubstitute/Moq) works for simple logic but doesn't validate Redis semantics. Beware in-memory fakes lacking Lua/streams.

| Option | Best for | Not for |
|---|---|---|
| IMemoryCache | single instance, tiny data | multi-instance consistency |
| Redis | shared cache, counters, locks, rate limits | system of record, complex queries |
| Cosmos DB | durable, globally distributed docs, point reads ~ms | sub-ms latency at lowest cost |
| SQL | relational truth, queries | very high read fan-out without cache |

### 16.1 [Easy] How do you run Redis locally for dev?
**Answer**
Run the official Redis Docker image and connect to localhost:6379 without TLS. Keep the connection string in config so Azure uses TLS/Entra instead.
**Explanation**
Parity: match Redis major version to your Azure cache version where possible (verify). Use user-secrets for keys.
**Code / CLI / Bicep**
```bash
docker run -d --name redis -p 6379:6379 redis:7
redis-cli -p 6379 PING
```

### 16.2 [Follow-up] How do you integration-test code using Redis?
**Answer**
Use Testcontainers to spin up a real Redis container in the test fixture and point the multiplexer at it. This validates TTL, Lua scripts and atomicity that fakes can't.
**Explanation**
Share one container per test class/collection and flush between tests with FLUSHALL (`allowAdmin=true`) to keep tests fast and isolated.
**Code / CLI / Bicep**
```csharp
public class RedisFixture : IAsyncLifetime
{
    private readonly RedisContainer _c = new RedisBuilder().WithImage("redis:7").Build();
    public IConnectionMultiplexer Mux = default!;
    public async Task InitializeAsync()
    {
        await _c.StartAsync();
        Mux = await ConnectionMultiplexer.ConnectAsync(_c.GetConnectionString());
    }
    public async Task DisposeAsync() { Mux.Dispose(); await _c.DisposeAsync(); }
}
```
```bash
dotnet add package Testcontainers.Redis
```

### 16.3 [Medium] Redis vs in-memory vs Cosmos/SQL - justify a choice for product catalog reads.
**Answer**
Keep SQL/Cosmos as the source of truth; add Redis (or HybridCache L1+L2) for hot read paths shared across instances. For very small, rarely changing data, IMemoryCache alone may suffice. Cosmos with its integrated cache or point reads can be enough if latency/RU budget allows.
**Explanation**
Decision factors: read:write ratio, tolerable staleness, data size, number of instances, cost (RU vs cache SKU), operational complexity. Measure first; don't add a cache to cover a missing index.
**Code / CLI / Bicep**
```csharp
// HybridCache gives L1 (fast) + L2 (shared) with one API
var cat = await hybrid.GetOrCreateAsync("catalog:v2", async ct => await repo.LoadCatalogAsync(ct), cancellationToken: ct);
```

### 16.4 [Hard] How do you test failure behavior (Redis down, slow, evicted)?
**Answer**
Stop the Testcontainers container mid-test, inject latency (Toxiproxy) and assert that the app falls back to the source within time budgets, and logs/metrics emit. In Azure, test with a non-prod cache failover reboot.
**Explanation**
Chaos tests verify abortConnect/timeouts/circuit-breaker settings, that no request hangs for the full default timeout, and that recovery happens automatically without restart.
**Code / CLI / Bicep**
```csharp
await _c.StopAsync();
var sw = Stopwatch.StartNew();
var result = await svc.GetProductAsync(42);          // must still succeed via DB
Assert.True(sw.Elapsed < TimeSpan.FromSeconds(3));
```

**Interview traps**
- Mocking IDatabase and believing Redis semantics are tested.
- Cache fakes lacking Lua/TTL behavior.
- Testing only the happy path.

**Cheat sheet**
- docker redis:7 locally.
- Testcontainers.Redis for integration.
- Test Redis-down fallback.
- Cache is an optimization, DB stays truth.

---

## Topic 17: Production scenarios

**Study notes**
Pattern for each: symptom -> likely cause -> verify -> fix -> prevent.

### 17.1 [Medium] Timeouts under load (RedisTimeoutException spikes)
**Answer**
Likely client thread-pool starvation, big values, slow server commands or hitting tier bandwidth/CPU limits. Verify with exception details, server load, SLOWLOG, thread pool counters; fix blocking calls, reduce payloads, scale up/out.
**Explanation**
Prevent: async-only code, min threads set sensibly, load tests, alerts on server load > 70-80%.
**Code / CLI / Bicep**
```bash
az monitor metrics list --resource <id> --metric serverLoad cpu connectedclients --interval PT1M --aggregation Maximum
```

### 17.2 [Medium] Connection storm (hundreds of connections, errors at start)
**Answer**
Typically a ConnectionMultiplexer created per request/invocation or aggressive scale-out/restarts reconnecting simultaneously. Make it a singleton, stagger/jitter reconnects, cap instance count, check tier client limits (verify).
**Explanation**
Functions Consumption scale-out multiplies connections: instances x connections each. Check `connectedclients` metric and `CLIENT LIST` for client addresses.
**Code / CLI / Bicep**
```csharp
// Functions isolated worker: register once
services.AddSingleton<IConnectionMultiplexer>(_ => ConnectionMultiplexer.Connect(opts));
```

### 17.3 [Medium] Stale data after deploy
**Answer**
New code changed meaning/shape of cached data but old keys persist, or old and new versions share keys. Version keys, bump namespace, or flush selectively; treat deserialize errors as misses.
**Explanation**
Prevent: key schema version constant, backward-compatible additive changes, and deployment checklists noting cache impact.
**Code / CLI / Bicep**
```csharp
const string Ver = "v5";   // bump when cached shape changes
```

### 17.4 [Hard] Memory full / evictions climbing
**Answer**
Check key count, TTL coverage (`INFO keyspace`), big keys, eviction policy, and growth. Fix by adding TTLs, trimming values, bounding key cardinality, scaling up/out, or changing eviction policy. Alert before 80% memory.
**Explanation**
If noeviction/volatile-* with non-TTL keys: OOM write errors. If allkeys-lru: evictions raise miss rate and DB load. Don't FLUSHALL blindly - it triggers an avalanche.
**Code / CLI / Bicep**
```bash
redis-cli ... INFO keyspace
redis-cli ... INFO memory | grep -E "used_memory_human|maxmemory_human|mem_fragmentation_ratio"
redis-cli ... --bigkeys
```
**Gotchas / follow-ups**
- Fragmentation: used_memory_rss >> used_memory; restart/reboot or defrag per version (verify).

### 17.5 [Hard] Failover blips and key explosion
**Answer**
Failover: expect seconds of errors; mitigate with retries, fail-open, reconnect policy; correlate with Azure Service Health/maintenance. Key explosion: unbounded cardinality (e.g. keys per raw URL); normalize inputs, add TTLs, cap cache scope, monitor key count.
**Explanation**
Add dashboards: hit ratio, evictions, key count, P99 Redis latency (app-side telemetry via Application Insights dependency tracking - StackExchange.Redis can be instrumented with OpenTelemetry/`StackExchange.Redis` instrumentation packages).
**Code / CLI / Bicep**
```kusto
dependencies
| where type == "Redis" or target contains "redis.cache.windows.net"
| summarize p95=percentile(duration,95), failures=countif(success==false) by bin(timestamp,5m)
```

### 17.6 [Tricky] Serialization version break during rolling deploy
**Answer**
Old instances write v-old payloads while new ones expect v-new (or vice versa) causing deserialization exceptions and 500s. Use versioned keys, additive schema changes, and treat failures as cache misses with delete. Deploy with backward and forward-compatible readers first.
**Explanation**
Two-phase rollout: release N reads both formats but writes old; release N+1 writes new. Alternatively separate key version per release so there is no overlap.
**Code / CLI / Bicep**
```csharp
try { return Deserialize(bytes); }
catch (Exception ex) when (ex is JsonException or MessagePackSerializationException)
{ _log.LogWarning(ex, "Bad cache payload {Key}", key); await _db.KeyDeleteAsync(key); return null; }
```

**Interview traps**
- Flushing the whole cache to "fix" things.
- Blaming Redis when the client thread pool is starved.
- Not alerting on evictions and server load.

**Cheat sheet**
- Timeouts: client threads, big values, server load.
- Storms: singleton multiplexer, jitter.
- Deploy: versioned keys.
- Memory: TTL coverage, big keys, scale.
- Always: fail open, retry with jitter, observe.
