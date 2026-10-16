# S06 - Scalability, Performance & Data Strategy (Senior Engineer Interview Guide)

**How to use:** Read the study notes, then answer each ladder question aloud before reading the strong answer. Practise saying numbers and assumptions out loud; seniors are judged on reasoning order (requirements -> options -> trade-offs -> decision -> risks), not on memorised facts.
Numbers quoted are illustrative unless stated; always say "assuming..." in an interview.

## Table of Contents
1. Scalability fundamentals
2. Finding the bottleneck; premature optimisation
3. Capacity planning & back-of-envelope estimation (and giving estimates in interviews)
4. Caching strategy as an architecture decision
5. Read scaling & write scaling
6. SQL Server vs NoSQL vs polyglot persistence
7. Data modelling, indexing & pagination at scale
8. Async processing, queues & event-driven scale
9. Connection/thread management & API performance design
10. Multi-tenant scale, rate limiting & fairness
11. Global distribution
12. Cost-aware architecture (FinOps)
13. Performance budgets, SLOs & load testing
14. Database scaling pains, reporting & analytics without hurting OLTP
15. Search, autocomplete & file/media handling at scale
16. Scenario round

---

## Topic 1: Scalability fundamentals

**Study notes**
- Vertical (scale up): bigger box. Simple, no code change, but ceiling, downtime on resize, single point of failure. Horizontal (scale out): more instances. Needs stateless services + shared state elsewhere.
- Stateless service: any request can hit any instance. Session/state goes to Redis, DB, or a signed token (JWT/cookie). Avoid sticky sessions unless unavoidable (ARR affinity in App Service is a crutch).
- Latency = time for one request. Throughput = requests/second. They trade off: batching raises throughput, raises latency.
- Little's law: `L = lambda x W` (concurrent items in system = arrival rate x time each spends in system). 200 req/s x 0.25 s = ~50 requests in flight. Use it to size pools/threads/instances.
- Amdahl's law: speedup is capped by the serial fraction. If 10% is serial (a lock, a single DB writer), max speedup is 10x no matter how many nodes.
- Utilisation vs latency: queues grow non-linearly as utilisation nears 100%; run hot paths well below saturation (commonly 60-70% target).

```
Vertical:   [ small VM ] -> [ BIG VM ]          (ceiling, one failure domain)
Horizontal: LB -> [app][app][app]  -> shared state (Redis / SQL / Cosmos)
```

### 1.1 [Easy] What is the difference between scaling up and scaling out, and when would you pick each?
**What the interviewer is really testing**: Do you see it as a trade-off, not "out is always better"?
**Typical mid-level answer**: "Out is better because cloud." Falls short: ignores the state problem, cost, and that the database often can only scale up first.
**Strong senior answer**: I would ask what is failing: CPU, memory, or a single stateful component. For a stateless ASP.NET Core API on App Service or AKS, scale out is cheap and gives availability too, so I default to it, with autoscale rules on CPU/queue depth/request rate. For SQL Server or Azure SQL, scaling up (more vCores, Hyperscale, higher tier) is often the first and quickest lever because writes are single-primary; scale out comes later via replicas, read models or partitioning, which cost real design effort. Scale up is also right when the workload is a single-threaded or lock-bound one that more nodes cannot help. The decision is: use the simplest lever that meets the next 6-12 months of projected load, and keep the path to the next lever open. Risk of scale up: ceiling and a resize that may cause a brief failover/connection drop; risk of scale out: hidden shared state and cost of coordination.
**Follow-up traps**: "What breaks when you go from 1 to 3 instances?" (in-memory cache, local files, timers/background jobs running 3x, in-proc sessions.)

### 1.2 [Follow-up] What makes a service "stateless", and what do you do with state you cannot remove?
**What the interviewer is really testing**: Practical handling of state, not the textbook definition.
**Typical mid-level answer**: "Store sessions in Redis." Partial: ignores caches, uploads, background work, and websockets.
**Strong senior answer**: Stateless means no instance holds data that another instance needs for correctness; instance loss only loses cache/warm-up. I would inventory state: user session (move to token or Azure Cache for Redis), uploads (Blob Storage, not local disk), in-memory caches (acceptable if they are only an optimisation, with a plan for cross-instance invalidation), scheduled jobs (use a leader/lock or a single scheduler such as Functions timer, or distributed lock via blob lease), and SignalR connections (Azure SignalR Service). For state that must stay (e.g. a stateful game or stream processing), partition it by key so one owner handles each key (Service Fabric, Orleans, Durable Entities) and make ownership failover explicit. Trade-off: externalising state adds a network hop and a new dependency to keep highly available.
**Follow-up traps**: "Is a JWT stateless?" (yes, but revocation then needs state or short lifetimes.) "Sticky sessions?" (uneven load, lose on scale-in.)

### 1.3 [Medium] Explain Little's law and use it to size something.
**What the interviewer is really testing**: Can you turn theory into a sizing number?
**Typical mid-level answer**: Cannot recall or just states the formula.
**Strong senior answer**: In plain words: the number of things in the system equals arrival rate times time spent. If an API gets 500 req/s and each calls SQL for ~40 ms, then about 500 x 0.04 = 20 connections are busy on average; peaks and variance mean I would size the pool for maybe 2-3x that, while checking the database can actually handle that concurrency (worker threads, DTU/vCore limits). It also tells me that if latency of a downstream dependency rises from 40 ms to 400 ms, in-flight work grows 10x at the same arrival rate, so threads/connections exhaust and a slow dependency becomes an outage. That justifies timeouts, bulkheads, and circuit breakers. I'd state the assumptions (average vs peak, p50 vs p99 service time) because averages hide tails.
**Follow-up traps**: "Which latency do you use, p50 or p99?" (mean service time for average load; p99 for worst-case headroom.)

### 1.4 [Medium] A manager says "just add 10 more servers" to fix slowness. Respond.
**What the interviewer is really testing**: Amdahl-thinking and not blindly scaling.
**Typical mid-level answer**: "Ok, autoscale." Falls short: no diagnosis.
**Strong senior answer**: Adding nodes only helps if the bottleneck is parallelisable compute in front of the constraint. If the slowness is a lock-contended table, a single hot partition, or a downstream rate limit, extra nodes increase contention and can make it worse (Amdahl: the serial part caps the gain). I would first measure where time goes (App Insights dependency durations, SQL wait stats, CPU of each tier). If app CPU is high and DB is idle, adding instances is the right and cheap move; if the DB is at its ceiling, more app nodes just open more connections and amplify the problem. I would explain this to the manager as "we will test the hypothesis in a day; adding nodes is the fast experiment only if the data supports it". Risk: cost with zero benefit, and masked root cause.
**Follow-up traps**: "How would the extra servers hurt the DB?" (more connections, more concurrent blockers.)

### 1.5 [Tricky] Throughput went up 3x after you introduced batching, but users complain it feels slower. Why?
**What the interviewer is really testing**: Throughput vs latency distinction.
**Typical mid-level answer**: "Maybe a bug."
**Strong senior answer**: Batching amortises per-call overhead (round trips, log flushes, TLS) and so lifts throughput, but each item now waits for the batch to fill or a timer to fire, adding latency. Interactive paths care about latency; background paths care about throughput. I would keep the user-facing path unbatched or use a tiny time/size window (e.g. flush at 50 items or 20 ms), and batch only the async tail (inserts via SqlBulkCopy/table-valued parameters, Service Bus batch sends). I'd verify with p95/p99 latency rather than only averages, since batching also produces bursty load. Risk: larger blast radius per failed batch, so I need idempotency and poison-item handling.
**Follow-up traps**: "How do you choose batch size?" (empirically, find the knee where throughput gains flatten and latency/memory costs rise.)

**Red flags in answers**
- "Scale out always wins" with no mention of state or the database.
- Quoting Amdahl/Little formulas but unable to apply them to a number.
- Treating average latency as the only metric.

**Cheat sheet**
- Up first for DBs, out first for stateless tiers.
- State inventory: session, files, cache, timers, websockets.
- Little: in-flight = rate x latency.
- Amdahl: serial fraction caps speedup.
- Batching: throughput up, latency up.
- Run hot paths at ~60-70% utilisation, not 95%.

---

## Topic 2: Finding the bottleneck; premature optimisation

**Study notes**
- Method: define symptom -> measure -> form hypothesis -> change ONE thing -> re-measure. No measurement, no optimisation.
- USE method (Brendan Gregg): for every resource check Utilisation, Saturation, Errors. RED for services: Rate, Errors, Duration.
- Tools in the .NET/Azure/SQL stack: Application Insights (dependency, end-to-end transaction, Profiler, Snapshot Debugger), `dotnet-counters`, `dotnet-trace`, `dotnet-dump`, PerfView, Query Store, `sys.dm_os_wait_stats`, `sys.dm_exec_query_stats`, actual execution plans, Extended Events, Azure SQL Intelligent Insights/Query Performance Insight.
- Typical order of wins: avoid work > fix the query/index > cache > parallelise > bigger hardware.
- Knuth's "premature optimisation" quote is about micro-optimising before knowing the hot spot, not about ignoring design that affects scalability (e.g. N+1, chatty calls).

```
User latency = client + network + gateway + app CPU + [dependency calls] + DB (CPU | IO | locks | memory grants)
Find the biggest slice first.
```

### 2.1 [Easy] Users say "the app is slow". What do you do first?
**What the interviewer is really testing**: Method before action.
**Typical mid-level answer**: "Check the logs / add caching."
**Strong senior answer**: I clarify: which operation, since when, for whom, how slow versus the baseline, and what changed (deploy, data growth, traffic). Then I look at the end-to-end transaction view in Application Insights to see which segment dominates: app code, a dependency, or SQL. If I have no baseline, I start by establishing one. I do not change anything until I can name the slowest slice with evidence. Then I form one hypothesis, change one thing, and re-measure under the same conditions. Risk: fixing the wrong thing and hiding the real one; also I must check whether the slowness is user-specific (data skew, one big tenant).
**Follow-up traps**: "No telemetry exists." (add minimal tracing first; use dotnet-trace/SQL DMVs for immediate evidence.)

### 2.2 [Follow-up] Walk me through the USE method on a slow SQL-backed API.
**What the interviewer is really testing**: Systematic coverage of resources.
**Typical mid-level answer**: "Check CPU and memory."
**Strong senior answer**: For each resource I ask: utilisation, saturation (queueing), errors. App tier: CPU %, thread pool queue length and thread count (`dotnet-counters`: ThreadPool Queue Length), GC pause time, connection pool wait. Network: bandwidth, SNAT port exhaustion on App Service/AKS egress. SQL: CPU, IO latency, memory grants pending, worker threads, and wait stats (PAGEIOLATCH = disk, LCK_M_* = blocking, SOS_SCHEDULER_YIELD = CPU, RESOURCE_SEMAPHORE = memory grants, WRITELOG = log). Saturation signals are the ones people skip: queue lengths, blocked sessions, throttled requests (HTTP 429 from Cosmos, 503s). The output is a table of resources vs U/S/E that points at the first saturated one. Trade-off: USE finds resource issues but not logic issues (N+1), so I pair it with request-level tracing.
**Follow-up traps**: "CPU is 30% but latency is high?" (waiting on locks, IO, thread starvation, or a downstream.)

### 2.3 [Medium] You find a 3-second query. How do you decide how to fix it?
**What the interviewer is really testing**: Priority of fixes and cost-benefit.
**Typical mid-level answer**: "Add an index."
**Strong senior answer**: First check how often it runs and its total cost (Query Store: duration x executions), since a 3 s query that runs once a day is less urgent than a 30 ms one running 10 000/min. Then the actual plan: scans vs seeks, key lookups, estimate vs actual row mismatch (stale stats, parameter sniffing), implicit conversions (nvarchar vs varchar parameter from EF), missing joins/predicates, spills. Options: rewrite the query / avoid fetching unneeded columns, add or adjust an index (checking write cost and existing indexes), update statistics, handle parameter sniffing (OPTION RECOMPILE for specific cases, Query Store plan forcing as a stopgap), or move the work (precompute, cache, read model). I choose the least invasive that meets the target and verify on production-like data volumes. Risk: an index that helps one query and slows writes or other queries.
**Follow-up traps**: "Query is fast in dev, slow in prod?" (data volume, stats, parameter sniffing, different plan.)

### 2.4 [Tricky] When is optimising early the right call?
**What the interviewer is really testing**: Nuanced reading of "premature optimisation".
**Typical mid-level answer**: "Never, measure first."
**Strong senior answer**: Micro-optimisations (struct vs class, string builders) before measurement are wasteful. But architectural decisions that are expensive to reverse deserve early thought with rough numbers: partition key choice, chatty vs chunky APIs, N+1 patterns, synchronous call chains, unbounded queries, and data model. I do a back-of-envelope estimate at design time (order of magnitude), build in observability, and defer tuning to measured hot spots. I also avoid the opposite failure: "we'll fix it later" for things like missing pagination on list endpoints. Decision rule: be deliberate where the change is a one-way door; measure for two-way doors.
**Follow-up traps**: "Example of a cheap early decision that saves a lot?" (async I/O, pagination, HttpClientFactory, indexes on foreign keys.)

### 2.5 [Hard] Profiling shows the app is CPU bound in JSON serialisation and GC. What do you consider?
**What the interviewer is really testing**: Depth on .NET performance with cost awareness.
**Typical mid-level answer**: "Switch to a faster serializer."
**Strong senior answer**: First confirm it is a meaningful share of request time and under realistic payloads. Options: reduce payload (project only needed fields, avoid serialising EF entity graphs, DTOs), use System.Text.Json with source generation, stream with `IAsyncEnumerable` / `Utf8JsonWriter` for large responses, avoid LOH allocations (>85 KB arrays), pool buffers (`ArrayPool`, `RecyclableMemoryStream`), consider Server GC settings and container CPU limits (GC heaps tied to cores). Also ask whether the work should happen at all: cache serialised responses or use response compression/ETags. I'd benchmark with BenchmarkDotNet and validate under load testing, because micro gains must show at system level. Risk: complexity and maintainability cost for small wins; only keep changes that move p95 or cost materially.
**Follow-up traps**: "Gen2 collections frequent?" (LOH churn, large object retention, caches holding data.)

**Red flags in answers**
- Jumps to a solution (cache, bigger tier) without naming the bottleneck.
- Only looks at averages and CPU.
- Uses "premature optimisation" to excuse obviously unbounded designs.

**Cheat sheet**
- Measure -> hypothesis -> one change -> re-measure.
- USE for resources, RED for services.
- Rank queries by total cost, not single-run time.
- Wait stats tell you what SQL is waiting for.
- Fix order: avoid work, query/index, cache, parallelise, hardware.
- One-way doors get early thought; two-way doors get measurement.

---

## Topic 3: Capacity planning & back-of-envelope estimation

**Study notes**
- Inputs: users, requests per user, peak-to-average ratio, payload size, retention, growth rate. Always separate average vs peak.
- Useful conversions: 1 day ~ 86 400 s (round to 10^5); 1M requests/day ~ 12 req/s average; peak often 3-10x average (state the assumption). 1 KB x 1M rows = 1 GB.
- Storage = rows x row size x (1 + index overhead ~ 0.3-1.0) x retention x replication factor.
- Headroom: plan for peak x safety margin (e.g. 30-50%), with a growth horizon (12 months).
- Validate estimates with load tests and production telemetry; revisit quarterly.
- Estimating in an interview: state assumptions, round aggressively, show units, give a range, name the biggest sensitivity, and say how you would validate.

```
DAU 200k x 30 req/day = 6M/day -> ~70 req/s avg -> peak x5 -> ~350 req/s
Payload 5 KB -> 1.75 MB/s egress at peak (~14 Mbit/s)
```

### 3.1 [Easy] Walk me through estimating request rate for a new product with 200k daily users.
**What the interviewer is really testing**: Structured estimation with explicit assumptions.
**Typical mid-level answer**: Gives one number with no assumptions.
**Strong senior answer**: I would say: assuming 200k DAU and ~30 API calls per user per day, that is 6M requests/day, about 70 req/s average (6M / 86 400). Traffic is not flat; assuming a peak factor of 5 within an hour window gives ~350 req/s, and I would design for ~500 with headroom. The biggest uncertainty is calls per user and peak factor, so I'd test sensitivity (x2 either way) and say I'd refine with real telemetry or a similar existing product. Then I'd convert to resources: how many API instances at X req/s each (from a benchmark, not guesses), and DB queries per request. Risk: false precision; I present a range.
**Follow-up traps**: "What if marketing launches a campaign?" (spike factor, pre-scale, queue/throttle.)

### 3.2 [Follow-up] Estimate storage growth for an orders system, and what it implies.
**What the interviewer is really testing**: Data growth thinking tied to decisions.
**Typical mid-level answer**: Only raw data size.
**Strong senior answer**: Assuming 100k orders/day, 5 lines each, ~300 B per line, ~1 KB per order header, that is roughly 100k x (1 KB + 1.5 KB) = 250 MB/day of data, add ~50-100% for indexes: about 400 MB/day, ~150 GB/year, plus log/backups/replicas (transaction log and backup storage multiply real footprint). Over 5 years ~ 0.75 TB: fine for one Azure SQL database, but it affects index maintenance windows, restore time (RTO), and query cost on old data. So I'd decide on archival of old orders (e.g. >2 years to cheaper storage), partitioning for sliding-window maintenance, and a separate analytics store. Risk: row-size assumptions wrong; I'd check against a real sample (`sp_spaceused`).
**Follow-up traps**: "What about restore time?" (large DB means long PITR/restore; RTO implication.)

### 3.3 [Medium] How do you plan capacity for DB connections and workers across a fleet that autoscales?
**What the interviewer is really testing**: Fleet-level multiplication effects.
**Typical mid-level answer**: "Pool size 100 default."
**Strong senior answer**: Total connections = instances x pool size. With a max pool of 100 and autoscale to 30 instances, I could demand 3 000 connections, exceeding the database limits (Azure SQL max concurrent workers/sessions depend on the service objective) and causing throttling. I'd compute needed concurrency via Little's law, set a smaller pool per instance (`Max Pool Size`), cap max instances, and use async code so fewer threads hold connections for less time. Where needed, add queueing in front (Service Bus) so the database sees a controlled concurrency. Also make sure scale-out rules consider DB saturation, not just app CPU. Risk: autoscale as a self-inflicted DDoS on the database.
**Follow-up traps**: "How do you detect it?" (pool wait times, timeout exceptions, `sys.dm_exec_requests` runnable counts.)

### 3.4 [Medium] How do you decide headroom and when to scale before demand arrives?
**What the interviewer is really testing**: Risk-based planning.
**Typical mid-level answer**: "Autoscale handles it."
**Strong senior answer**: Autoscale reacts in minutes (instance start, warm-up, JIT, cache fill), while spikes can arrive in seconds and databases do not autoscale quickly (Azure SQL provisioned needs a tier change; serverless pauses/resumes with latency). So for predictable events I pre-scale (scheduled scale, warm instances, `Always On` / minimum instance count), and for unpredictable ones I keep headroom at the stateful tier and use queue levelling and rate limits to protect it. I choose headroom from the cost of failing (revenue per minute of outage) vs the cost of idle capacity, and review with load-test breakpoints. Risk: scaling rules based on lagging metrics; prefer leading metrics like queue depth or request rate.
**Follow-up traps**: "Which metric for autoscale?" (request rate/queue length rather than only CPU.)

### 3.5 [Scenario] In an interview you're asked "how many servers do we need for 1M users?" with no other data. What do you say?
**What the interviewer is really testing**: Comfort with ambiguity, structured assumptions, no bluffing.
**Typical mid-level answer**: "About 10 servers."
**Strong senior answer**: I'd ask two or three clarifying questions: active vs registered users, what a typical session does, and latency/availability targets. If I cannot get answers, I'd say "I'll assume 10% daily active, 20 requests each, peak 5x", compute ~2M requests/day, ~25 req/s average, ~125 peak. Then "if one modest instance handles ~100-200 req/s for this workload, which I would verify by benchmark, I'd run 3 instances: 2 for load and redundancy across zones, 1 for headroom." The point I'd stress is that the web tier is the easy part; the data tier sizing and the biggest-risk assumption drive the design. I'd close with how I'd validate: load test and revise. Risk: sounding precise without a basis, so I label every number as an assumption.
**Follow-up traps**: "What if the interviewer says 'just pick one number'?" (give the number with the stated assumption and the range.)

**Red flags in answers**
- A single number with no units, assumptions, or peak factor.
- Ignoring index/log/backup overhead in storage.
- Multiplying instances without considering connection or downstream limits.

**Cheat sheet**
- 1M req/day ~ 12 req/s average.
- Peak factor is your biggest assumption; state it.
- Storage = rows x size x index overhead x retention x copies.
- Instances x pool size = DB connection demand.
- Pre-scale for known events; autoscale is reactive.
- Always finish with "here is how I'd validate".

---

## Topic 4: Caching strategy as an architecture decision

**Study notes**
- Layers: browser -> CDN (Azure Front Door / Azure CDN) -> API gateway (APIM cache) -> in-process (`IMemoryCache`) -> distributed (Azure Cache for Redis, `IDistributedCache`, HybridCache in .NET 9) -> DB buffer pool/query results.
- Patterns: cache-aside (most common), read-through, write-through, write-behind (risk of loss), refresh-ahead.
- Invalidation: TTL (simple, bounded staleness), explicit invalidation on write, event-driven invalidation, versioned keys. "Two hard things in computer science."
- Failure modes: stampede/thundering herd (many misses at expiry), penetration (requests for non-existent keys), cache as hidden dependency (cold cache outage), inconsistent invalidation, memory eviction surprises.
- Consistency: caching is choosing staleness. Decide per data type the acceptable staleness.
- Do not cache: highly personalised or per-user data at shared layers without proper keys (security leak), data requiring strict read-your-writes, rapidly changing data with low hit ratio, secrets/PII without thought.

```
Request -> CDN? -> App memory? -> Redis? -> SQL
 hit ratio at each layer drops cost/latency for everything behind it
```

### 4.1 [Easy] How do you decide whether something should be cached?
**What the interviewer is really testing**: Is caching a reasoned decision?
**Typical mid-level answer**: "Cache anything that is slow."
**Strong senior answer**: Cache when the data is read far more than written, expensive to compute or fetch, shared across users, and tolerant of some staleness. I would estimate hit ratio and the saving: if only 20% hit ratio, the complexity may not pay. I define the staleness budget with the business (price list: 5 minutes ok; account balance: no). Then choose layer by scope: static assets and public API responses at CDN, reference data in-process with short TTL, shared expensive results in Redis. Cost of caching: invalidation logic, another failure mode, memory/cost, and harder debugging. Decision: cache with a metric on hit ratio and a documented TTL, and be ready to remove it if it does not earn its keep.
**Follow-up traps**: "What if the data is slow because the query is bad?" (fix the query first; cache hides it.)

### 4.2 [Follow-up] Compare in-process cache and Redis for a multi-instance API.
**What the interviewer is really testing**: Trade-offs including consistency.
**Typical mid-level answer**: "Redis because it's shared."
**Strong senior answer**: In-process is fastest (no network, no serialisation) but each instance has its own copy, so updates are inconsistent across nodes and every new instance starts cold, multiplying DB load after scale-out or deploy. Redis gives a single shared copy and better hit ratio at the cost of ~1 ms network round trip, serialisation, a dependency to keep available and cost. Often the answer is both: a short-TTL local cache in front of Redis (L1/L2), which HybridCache in .NET 9 supports including stampede protection; for cross-node invalidation use Redis pub/sub or accept short TTL. Choose in-process for small, hot, rarely changing reference data; Redis for larger or must-be-consistent-ish data. Risk: Redis outage must degrade gracefully (fall back to DB with limits, not cascade).
**Follow-up traps**: "Redis is down; what happens?" (timeouts too long cause thread starvation; use short timeouts, circuit breaker, fallback.)

### 4.3 [Medium] Explain cache stampede and how you prevent it.
**What the interviewer is really testing**: Understanding failure modes.
**Typical mid-level answer**: "Set a TTL."
**Strong senior answer**: A stampede is many concurrent requests missing the same expired or evicted key and all hitting the database at once, which can take it down. Mitigations: single-flight (only one caller recomputes, others wait; per-key lock via `SemaphoreSlim`, or HybridCache built-in), TTL jitter so keys do not expire together, stale-while-revalidate (serve slightly stale value while one refresh happens), refresh-ahead for known hot keys, and pre-warming after deploy. For a negative-lookup attack (non-existent keys), cache negative results briefly or use a Bloom filter. The trade-off is added complexity and slightly staler data. I would test it with a load test that flushes the cache on purpose.
**Follow-up traps**: "A distributed lock in Redis?" (needs timeouts/fencing; local single-flight per instance is often sufficient.)

### 4.4 [Medium] How do you handle cache invalidation when the data is updated by multiple services?
**What the interviewer is really testing**: Consistency thinking across boundaries.
**Typical mid-level answer**: "Delete the key on update."
**Strong senior answer**: Delete-on-write has a race: reader loads old value from DB, writer updates and invalidates, reader then writes the old value into the cache. Mitigations include short TTL as a backstop, versioned keys, or writing invalidation as an event so all owners invalidate. When multiple services own writes, I'd name a single owner of the data and have others consume its change events (Service Bus topic, Event Grid, Cosmos change feed, SQL CDC) to refresh or evict. Combine with TTL so a missed event cannot cause permanent staleness. If strict consistency is required, don't cache or read from the primary. Risk: event delay means bounded staleness; I make that explicit to stakeholders.
**Follow-up traps**: "Why not write-through?" (couples write latency to cache; multi-writer still needs ownership.)

### 4.5 [Tricky] Your cache hit ratio is 95% but the DB is still overloaded. Why?
**What the interviewer is really testing**: Reasoning about miss cost and distribution.
**Typical mid-level answer**: "Cache too small."
**Strong senior answer**: 95% hit still means 5% misses; at 10 000 req/s that is 500 DB queries/s, and if misses are the expensive queries, or concentrated on a few keys at once (stampede), or the cold-start after deployment, DB load stays high. Also writes and uncached endpoints are untouched by the cache. I would break down DB load by query: which queries cause it and whether they are cacheable. Check the key distribution (long tail of unique keys has low hit ratio), eviction rate, and TTLs too short. Fixes: cache the expensive miss path, single-flight, pre-warm, widen TTL where acceptable, or fix the query. Trade-off: hit ratio alone is a vanity metric; weight by miss cost.
**Follow-up traps**: "How would you measure it?" (hit/miss by key class, DB top queries by CPU in Query Store.)

### 4.6 [Hard] Design caching for a product catalogue page used globally with personalised prices.
**What the interviewer is really testing**: Layering and cache keys with security.
**Typical mid-level answer**: "Cache the whole page in Redis."
**Strong senior answer**: Split the response into shared and personalised parts. Catalogue data (name, description, images) is identical for everyone, so cache it at CDN and in Redis with versioned URLs for images/static content and a moderate TTL plus event-based purge on edit. Personalised price depends on customer segment or contract; I'd cache price per (sku, price-list) rather than per user so the key space stays bounded, and fetch it as a separate call or edge-side composition. Never cache `Authorization`-dependent responses at a shared layer without `Vary`/private directives; a mistake leaks one customer's prices to another. Choose staleness: price changes need faster propagation than descriptions (shorter TTL or event invalidation). Risk: key cardinality explosion and leaking data; I would test with two different users and verify headers (`Cache-Control: private` where needed).
**Follow-up traps**: "How do you purge CDN for one product?" (purge by path/tag where supported, or use versioned URLs.)

**Red flags in answers**
- "Cache it in Redis" with no TTL, owner, or invalidation story.
- Caching authenticated responses at a shared layer.
- No plan for cache outage or cold start.

**Cheat sheet**
- Caching = choosing staleness; agree the budget.
- Layers: CDN, gateway, L1 memory, L2 Redis, DB.
- Cache-aside default; TTL as backstop even with events.
- Stampede: single-flight, jitter, stale-while-revalidate.
- Weight hit ratio by miss cost.
- Don't cache: strict consistency, low reuse, per-user data at shared layers.

---

## Topic 5: Read scaling & write scaling

**Study notes**
- Read scaling: cache; read replicas (Azure SQL read scale-out on Premium/Business Critical, Hyperscale named/HA replicas; geo-replicas); read models (denormalised projections); CQRS; search index for queries.
- Replica caveat: replication lag -> stale reads; route "read-your-writes" paths to primary.
- CQRS: separate write model and read model; read model updated by events/CDC/outbox; eventual consistency; more moving parts. Not needed for simple CRUD.
- Write scaling is harder (single writer): batching, reduce write amplification (indexes), queue-based levelling, partitioning/sharding, append-only/event logs, scale up, move hot counters elsewhere.
- Sharding: split data by key across databases (Elastic Database tools / shard map); cost: cross-shard queries, rebalancing, transactions, operational complexity. Partitioned tables in a single SQL Server do NOT scale out writes across servers; they help manageability.

```
Writes -> [Primary] --replication--> [Replica1][Replica2]  (reads)
Writes -> Service Bus -> workers (batch) -> DB      (levelled writes)
Sharded: key -> shard map -> DB_1 | DB_2 | DB_3
```

### 5.1 [Easy] Your read traffic is 20x write traffic and the DB is slow. Options?
**What the interviewer is really testing**: Knowing the read-scaling ladder.
**Typical mid-level answer**: "Add replicas."
**Strong senior answer**: First tune: indexes, queries, projections. Then cache the hot reads, which is usually cheapest. Next, offload to read replicas (Azure SQL read scale-out replica via `ApplicationIntent=ReadOnly`, or Hyperscale named replicas) for reporting and tolerant reads, accepting replication lag. If query shapes differ greatly from the write model, build a read model or search index. I'd pick the order by effort and risk: tune -> cache -> replica -> read model/CQRS. Risk: replicas give stale reads and don't help write load; developers must label which queries may use them.
**Follow-up traps**: "User saves then sees old data?" (read-your-writes: route to primary for a short window or use session consistency tokens.)

### 5.2 [Follow-up] When is CQRS worth it, and when is it over-engineering?
**What the interviewer is really testing**: Pragmatism.
**Typical mid-level answer**: "Always for scalability."
**Strong senior answer**: Worth it when read and write models genuinely diverge: complex write invariants vs many different read shapes, very different scaling profiles, or reads needing a different store (search, Cosmos, Redis). Not worth it for CRUD with similar shapes; the cost is eventual consistency, projection code, replay/rebuild tooling, and harder debugging. A middle path is logical CQRS in one database (separate query handlers, views, Dapper for reads, EF for writes) without separate stores. I'd start there and split physically only when a measured constraint justifies it. Risk: sync lag visible to users; mitigate via UX (optimistic UI) and outbox pattern for reliable projection updates.
**Follow-up traps**: "How do you keep the projection reliable?" (transactional outbox + idempotent consumers.)

### 5.3 [Medium] Writes are the bottleneck on one SQL database. What do you try, in order?
**What the interviewer is really testing**: Understanding write constraints.
**Typical mid-level answer**: "Shard it."
**Strong senior answer**: Before sharding, diagnose: log write latency (WRITELOG waits), blocking, too many indexes, row-by-row inserts, chatty transactions. Cheap wins: batch writes (SqlBulkCopy, TVPs, multi-row), shorter transactions, drop unused indexes, appropriate log throughput tier, in-memory OLTP or delayed durability for suitable cases (accepting loss window), scale up or move to Hyperscale for log rate. Then smooth bursts through a queue so the DB receives steady writes. Only after that consider partitioning the data across databases (sharding) if one database truly cannot hold the write rate; it adds cross-shard query, transaction, and rebalancing complexity. Risk: sharding too early locks in a key choice that may be wrong.
**Follow-up traps**: "Hot counter row (e.g. likes)?" (aggregate in memory/queue, periodic flush, or use Redis INCR and persist later.)

### 5.4 [Medium] Explain sharding trade-offs and choosing a shard key.
**What the interviewer is really testing**: Depth on distribution design.
**Typical mid-level answer**: "Shard by user id."
**Strong senior answer**: A shard key should match the dominant access pattern so most queries hit one shard, have high cardinality, and spread load evenly. For multi-tenant SaaS TenantId is natural, but one huge tenant can become a hot shard, so I'd support moving a tenant to its own shard (shard map with tenant->shard mapping, not a pure hash). Cross-shard queries and global uniqueness (use GUIDs/ULIDs or per-shard ID ranges) get harder; analytics should come from a separate store. Rebalancing and schema migration across N databases need automation. Azure options: Elastic Pools for cost sharing, Elastic Database client library/shard map, or a natively partitioned store like Cosmos DB if the model fits. Risk: a wrong key is very expensive to change; prototype with real distribution before committing.
**Follow-up traps**: "Hash vs range sharding?" (hash balances but loses range scans; range supports scans but can create hot ranges.)

### 5.5 [Hard] How do you do queue-based write levelling, and what are the risks?
**What the interviewer is really testing**: Async write patterns and correctness.
**Typical mid-level answer**: "Put a queue in front."
**Strong senior answer**: The API validates, enqueues a command to Service Bus (or Storage Queue/Event Hubs depending on needs), and returns 202 with an operation id; workers consume at a rate the DB can sustain and write in batches. Benefits: absorbs bursts, protects DB, workers scale independently with controlled concurrency. Risks: user sees eventual state (need status endpoint or notification), duplicates (at-least-once delivery -> idempotent writes with a message/idempotency key), ordering (sessions in Service Bus for per-key order), poison messages (DLQ and alerts), and unbounded queue growth hiding sustained overload (alert on queue age). It is only appropriate when the business accepts asynchronous completion. Otherwise use sync with rate limiting and scale the DB.
**Follow-up traps**: "Queue lag growing steadily?" (sustained arrival > service rate; add consumers only if DB can take it, else scale DB or shed load.)

### 5.6 [Tricky] A table partitioned by month is still slow for inserts. Why did partitioning not help?
**What the interviewer is really testing**: Partitioning misconceptions.
**Typical mid-level answer**: "Partitioning improves performance."
**Strong senior answer**: Partitioning in SQL Server mainly gives manageability (switch in/out for archival, per-partition index maintenance) and sometimes partition elimination for queries that filter on the partition key. It does not add write capacity across servers. If inserts are all going to the latest partition (time-based key), you get a hot partition and possibly last-page insert contention on an ever-increasing clustered key, plus log throughput limits. Fixes depend on the bottleneck: reduce index count, batch inserts, consider OPTIMIZE_FOR_SEQUENTIAL_KEY on SQL Server 2019+, or distribute inserts (hash key) if reads allow. Be clear that partitioning for scale-out of writes means sharding, a different thing. Risk: confusing the two causes expensive misdirected work.
**Follow-up traps**: "Does partitioning always help queries?" (only with partition-aligned predicates and aligned indexes.)

**Red flags in answers**
- Proposing replicas for a write bottleneck.
- Sharding as the first response, with no key reasoning.
- No mention of idempotency or eventual consistency with queues/CQRS.

**Cheat sheet**
- Reads: tune -> cache -> replica -> read model.
- Writes: batch, fewer indexes, queue levelling, scale up, then shard.
- Replicas = lag; route read-your-writes to primary.
- Shard key: matches access pattern, high cardinality, movable for big tenants.
- SQL partitioning != sharding.
- Queue + at-least-once => idempotent consumers, DLQ, queue-age alerts.

---

## Topic 6: SQL Server vs NoSQL vs polyglot persistence

**Study notes**
- Choose by access patterns, consistency needs, query flexibility, scale, team skills, operational cost.
- Azure SQL / SQL Server: relational integrity, ACID, ad-hoc queries/joins, mature tooling; scaling writes is vertical-first (Hyperscale extends storage/log).
- Cosmos DB: partitioned, low-latency at scale, tunable consistency (five levels), RU-based throughput pricing, multi-region writes; needs good partition key; cross-partition queries cost more; limited joins.
- Redis: in-memory cache/counters/queues/leaderboards/rate limits; not a system of record by default.
- Blob Storage: large objects, cheap; ADLS for analytics. Azure AI Search: full-text, facets, ranking. Table storage/Cosmos Table: simple key-value. Event Hubs/Kafka-style log for streams.
- Polyglot: right store per need, but each adds operations, consistency, and skill costs. Keep one system of record per fact.

| Need | Typical fit |
|---|---|
| Transactions, joins, reporting | Azure SQL / SQL Server |
| Key-based lookups at huge scale, global | Cosmos DB |
| Hot cache, counters, sessions | Redis |
| Files, media | Blob Storage |
| Text search, facets | Azure AI Search |

### 6.1 [Easy] When would you choose Cosmos DB over SQL Server?
**What the interviewer is really testing**: Reasoned selection, not fashion.
**Typical mid-level answer**: "When you need scale."
**Strong senior answer**: When access is mostly by a known key (single-partition reads/writes), needs predictable single-digit-millisecond latency, flexible schema, or multi-region writes/low-latency global reads, and the data size/throughput is beyond a comfortable single relational node. I'd check the costs: RU provisioning, cross-partition queries, lack of rich joins/transactions across partitions. If the product needs ad-hoc reporting, complex relational integrity, or the team is SQL-skilled and scale is moderate, Azure SQL is lower risk. Decision: choose on top 5 access patterns written down, plus cost estimate in RUs vs vCores. Risk: choosing Cosmos then discovering that queries span partitions and RU cost explodes.
**Follow-up traps**: "Can SQL Server store JSON?" (yes, JSON functions and native JSON type in newer versions; but not a substitute for a partitioned scale-out store.)

### 6.2 [Follow-up] Why is "NoSQL is faster" wrong as a statement?
**What the interviewer is really testing**: Avoiding cargo-cult.
**Typical mid-level answer**: "It is not always faster."
**Strong senior answer**: Speed depends on the match between access pattern and storage layout. Key-value reads by partition key are fast in Cosmos because the data is laid out for that; a join-heavy report is faster in SQL Server because the optimiser and indexes are built for it. NoSQL systems often make scale-out simpler by restricting what you can do cheaply; you pay by denormalising and managing consistency yourself. A well-indexed SQL database serves many thousands of requests per second. I'd benchmark with the real workload and check cost per operation, not trust slogans. Risk: migration cost to a store that doesn't fit the query needs.
**Follow-up traps**: "Which consistency level in Cosmos by default?" (Session.)

### 6.3 [Medium] Describe how you'd decide on polyglot persistence for a retail platform.
**What the interviewer is really testing**: Boundaries, one system of record, cost of complexity.
**Typical mid-level answer**: Lists many databases.
**Strong senior answer**: Start from bounded contexts and access patterns. Orders, payments, inventory with transactions: Azure SQL as system of record. Product catalogue read-heavy with flexible attributes: documents (Cosmos) or SQL plus a read model, with search in Azure AI Search for text/facets. Sessions, carts, rate limiting: Redis (cart may need durability, so decide). Images/media: Blob + CDN. Analytics: ADLS/Synapse/Fabric fed by CDC. Each extra store needs sync (events/outbox), monitoring, backup, security, and skills; I'd add one only when a measured requirement can't be met by the existing one. Keep one authoritative owner per fact; others are derived and rebuildable. Risk: data drift between stores, and a team that can't operate five technologies.
**Follow-up traps**: "How do you keep search in sync?" (change feed/CDC/outbox -> indexer; accept lag; rebuild capability.)

### 6.4 [Hard] Unclear access patterns for a new product. What do you pick?
**What the interviewer is really testing**: Decision-making under uncertainty; reversibility.
**Typical mid-level answer**: "Cosmos because flexible schema."
**Strong senior answer**: With unknown patterns I optimise for flexibility and reversibility: a relational store (Azure SQL) supports ad-hoc queries and changing questions, handles early scale for most products, and normalised data is easiest to reshape. Flexible schema in NoSQL helps writes but makes unknown queries expensive. I'd isolate data access behind repositories/module boundaries, keep an outbox/event stream so a second store can be derived later, and set explicit triggers to revisit (e.g. p95 > target at X rows, RU/vCore cost thresholds). Use JSON columns for genuinely variable attributes. Risk: later migration; mitigated by keeping domain boundaries clean and measuring real access patterns after launch.
**Follow-up traps**: "What if they predict 100x growth?" (still start relational unless the data model is clearly key-value; plan the sharding/partition path.)

### 6.5 [Tricky] A teammate proposes Redis as the primary store for orders because it's fast. Respond.
**What the interviewer is really testing**: Durability and consistency.
**Typical mid-level answer**: "Redis is only cache."
**Strong senior answer**: I'd ask what durability and query needs orders have: they are financial records, need transactions, auditing, ad-hoc queries, point-in-time restore. Redis persistence options (RDB/AOF; Azure managed tiers with persistence/geo-replication in certain tiers) exist but its data model and failure semantics are not designed for ACID multi-entity transactions or reporting. Use Redis where speed matters and loss is tolerable or rebuildable: caching order summaries, rate limits. Orders stay in the relational system of record. If latency is the concern, tune queries/indexes and cache reads. Risk: data loss or inconsistency in a system of record; the cost of an incident exceeds the latency gain.
**Follow-up traps**: "Redis eviction policy impact?" (keys can vanish; fine for cache, not for records.)

**Red flags in answers**
- Choosing technology by hype or one benchmark.
- Many stores with no owner/sync/rebuild story.
- Ignoring cost model (RU vs vCore) and operational skills.

**Cheat sheet**
- Write the top access patterns first.
- SQL = flexibility, integrity; Cosmos = key-based scale/global; Redis = speed/ephemeral; Blob = files; Search = text.
- Unknown patterns -> relational + clean boundaries + outbox.
- One system of record per fact; derived stores are rebuildable.
- Compare cost: RUs vs vCores vs ops effort.
- Define triggers for revisiting the decision.

---

## Topic 7: Data modelling, indexing & pagination at scale

**Study notes**
- Normalise for correctness and write simplicity; denormalise deliberately for read performance (precomputed totals, read models), owning the sync cost.
- Hot partition: one key receives disproportionate traffic (Cosmos physical partition RU cap; a single hot SQL page/row). Cure: higher-cardinality or synthetic keys (e.g. tenantId + bucket), write sharding, caching.
- Cosmos partition key: high cardinality, even distribution, appears in most queries' filter; logical partition limit 20 GB; physical partition throughput is capped, so RUs divide among partitions.
- Time-series: partition/segment by time, append-only, compress (columnstore, page compression), tier/archive old data (partition switching, Blob cool/archive, Cosmos TTL).
- Indexing: clustered key choice (narrow, ever-increasing, unique, static), covering indexes (INCLUDE), filtered indexes, columnstore for analytics. Each index costs write time, storage, maintenance. Use Query Store/missing-index DMVs as hints, not orders.
- Pagination: OFFSET/FETCH cost grows with offset (it scans and discards); keyset (seek) pagination uses `WHERE (SortCol, Id) > (@lastSort, @lastId) ORDER BY ... TOP n` and stays constant but needs a stable unique sort and does not support random page jump.

```
OFFSET 100000 ROWS FETCH 20  -> reads 100020 rows
Keyset: WHERE Id > @last ORDER BY Id  -> index seek, reads ~20
```

### 7.1 [Easy] Why would you denormalise, and what does it cost?
**What the interviewer is really testing**: Knowing denormalisation as a conscious trade-off.
**Typical mid-level answer**: "For speed."
**Strong senior answer**: Denormalise when a measured read path pays repeated join/aggregation cost and the data changes less often than it is read, e.g. storing OrderTotal on the order or CustomerName on a read model. The cost: write amplification, risk of inconsistency, and update logic in every writer. I would keep the normalised data as the source of truth and maintain the denormalised copy via the same transaction (computed/persisted columns, triggers rarely), or asynchronously via events with eventual consistency. Alternatives to try first: indexes, indexed views, caching. Risk: drift; mitigate with rebuild jobs and reconciliation checks.
**Follow-up traps**: "Indexed views downside?" (restrictions, write overhead, schema-binding.)

### 7.2 [Follow-up] How do you choose a clustered index key?
**What the interviewer is really testing**: Index design as architecture.
**Typical mid-level answer**: "The primary key."
**Strong senior answer**: Ideal clustered key is narrow (it is copied into every nonclustered index), unique, static, and ever-increasing, which avoids page splits and fragmentation. An identity/bigint or sequential GUID (NEWSEQUENTIALID) fits; random GUIDs cause page splits and bloat, though they spread inserts and avoid last-page contention, so there's a trade-off at very high insert rates (OPTIMIZE_FOR_SEQUENTIAL_KEY helps). Sometimes the best clustering is the dominant query key (e.g. TenantId, Date) when most queries are range scans on it, even if the PK is separate. I'd decide from the main access path, not by default. Risk: wide clustered keys inflate all indexes.
**Follow-up traps**: "GUID PK from client?" (fine for distributed ID generation; cluster on something else or use sequential/ULID-like.)

### 7.3 [Medium] Offset pagination is getting slow on deep pages. What do you do?
**What the interviewer is really testing**: Keyset pagination and API contract implications.
**Typical mid-level answer**: "Add an index."
**Strong senior answer**: OFFSET must read and discard all preceding rows, so cost grows with page number; an index on the sort column helps but not fully at depth. I'd switch to keyset pagination: the client sends the last seen (sort value, id) as an opaque cursor, and the query seeks with `WHERE (CreatedAt, Id) < (@c, @id) ORDER BY CreatedAt DESC, Id DESC` using a matching index; cost is constant. Trade-offs: no jump-to-page-N, total counts are costly (offer approximate counts or omit), and the sort must be deterministic with a tie-breaker. For UIs demanding page numbers, limit depth (e.g. max 1000 results) or send users to search. Risk: changing the API contract; use cursor tokens so internals can evolve.
**Follow-up traps**: "Rows inserted while paging?" (keyset is stable; offset shows duplicates/skips.)

### 7.4 [Medium] How do you pick a Cosmos DB partition key for an orders container?
**What the interviewer is really testing**: Hot partition awareness.
**Typical mid-level answer**: "OrderId."
**Strong senior answer**: Start from queries: if most reads are "orders for customer X" and writes are spread across customers, `/customerId` gives single-partition queries and good distribution. OrderId has perfect distribution but forces cross-partition queries for customer lists. A date key would make all current writes hit one partition (hot). For a very large customer, consider a synthetic key (customerId + month bucket). I'd check the 20 GB logical partition limit and RU per physical partition, and monitor with the partition-level metrics. Risk: the partition key is immutable; changing means migrating data (container copy), so validate with sample data and workload simulations.
**Follow-up traps**: "Point read vs query cost?" (point read by id+partition key is cheapest, about 1 RU per KB.)

### 7.5 [Hard] How do you design storage for time-series/event data growing 50 GB a day?
**What the interviewer is really testing**: Lifecycle and cost thinking.
**Typical mid-level answer**: "Put it in SQL and add indexes."
**Strong senior answer**: At 50 GB/day (~18 TB/year) I would ask: access pattern (recent dashboards vs rare historic queries), retention, query latency needs, and update patterns (append-only?). Hot data (last 7-30 days) in a store optimised for fast reads: SQL with a clustered columnstore or time-partitioned table, or Azure Data Explorer for telemetry-like data. Warm/cold in Parquet on ADLS/Blob with cool/archive tiers, queried via Synapse/Fabric. Partition by time so retention is a metadata operation (partition switch/drop) rather than a giant DELETE that bloats the log. Pre-aggregate rollups for dashboards. Compression and narrow types reduce cost. Risk: a single table growing forever makes backups, restores and maintenance slow; define retention with the business and legal early.
**Follow-up traps**: "DELETE old rows nightly?" (log growth, blocking, fragmentation; switch partitions instead.)

### 7.6 [Tricky] You added every index the missing-index DMV suggested and inserts got 3x slower. What went wrong?
**What the interviewer is really testing**: Index cost awareness.
**Typical mid-level answer**: "Too many indexes."
**Strong senior answer**: The missing-index DMV suggests per-query without considering overlap, existing indexes, or write cost; each nonclustered index must be updated on every insert/update/delete, increasing log, IO, and lock time. I'd review index usage (`sys.dm_db_index_usage_stats`, operational stats), merge overlapping indexes (one wider covering index instead of three), drop unused ones, and target the top costly queries only. Indexing is a budget: for write-heavy tables, keep it to what the critical read paths need and offload other reads to replicas/read models. Verify after changes with Query Store comparison. Risk: dropping an index used by a rare but critical monthly job; check over a full business cycle and script it to be recreated.
**Follow-up traps**: "How long to observe usage stats?" (they reset on restart; observe across a full cycle.)

**Red flags in answers**
- Offset pagination on large tables with no mention of keyset.
- Cosmos partition key chosen by "unique id" or by date.
- Indexing as "add whatever the DMV says".

**Cheat sheet**
- Normalise by default; denormalise with an owner and a rebuild path.
- Clustered key: narrow, unique, static, increasing.
- Keyset pagination: constant cost, no random page jump.
- Partition key: access pattern + cardinality + even load.
- Time data: partition by time, compress, tier, drop by partition.
- Every index taxes writes.

---

## Topic 8: Async processing, queues & event-driven scale

**Study notes**
- Queue-based load levelling: producers enqueue, consumers drain at a sustainable rate; absorbs spikes. Competing consumers scale out.
- Azure options: Service Bus (queues/topics, sessions, DLQ, transactions, ordering per session), Storage Queues (cheap, simple), Event Grid (reactive event routing), Event Hubs (high-throughput streaming; partitions; consumer groups).
- Delivery: at-least-once is the norm -> idempotent consumers. Exactly-once is mostly a design property (idempotency + dedup).
- Event Hubs: partition = unit of parallelism and ordering; max concurrent consumers per group = partition count; choose partition count up front (can't easily lower; increase has constraints depending on tier). Consumer groups give independent read positions.
- Autoscale: Azure Functions scale by queue length/partitions; KEDA scales AKS/Container Apps on queue depth, lag, etc.
- Batch vs stream: batch = higher throughput, higher latency, simpler; stream = low latency, complexity (windowing, late data, state).

```
Producer -> [Queue] -> Consumer x N (competing)       Event Hub: P0 P1 P2 P3 -> Consumer group A (4 readers max)
                |-> DLQ (poison)                                           -> Consumer group B (independent)
```

### 8.1 [Easy] When should work be async instead of in the request path?
**What the interviewer is really testing**: Judging user-visible vs deferrable work.
**Typical mid-level answer**: "Anything slow."
**Strong senior answer**: Work that the user does not need the result of to proceed (email, PDF generation, thumbnails, third-party sync, analytics events), work with spiky or unpredictable duration, and work depending on unreliable downstreams. Keep sync what must be consistent for the response (payment authorisation, validation). Async costs: eventual completion, need for status/notification, retries, idempotency, monitoring of queues. I would return 202 with a status resource for long tasks. Decision criterion: does the caller need the outcome to continue? Risk: hidden failure; async failures are silent unless DLQ and alerts are in place.
**Follow-up traps**: "Fire and forget in the web process (Task.Run)?" (lost on restart; use a durable queue.)

### 8.2 [Follow-up] Service Bus vs Event Hubs vs Storage Queue vs Event Grid?
**What the interviewer is really testing**: Selecting messaging primitives.
**Typical mid-level answer**: "They're all queues."
**Strong senior answer**: Service Bus for commands/business messages needing reliability features: DLQ, sessions (ordered per key), scheduled delivery, transactions, duplicate detection, topics with filters. Storage Queues for simple, cheap, huge-volume queues without advanced features. Event Hubs for high-throughput telemetry/event streams, replayable log, partitioned consumers, Kafka-compatible endpoint. Event Grid for reactive event notification (blob created, resource events, custom topics) with push delivery and filtering, not for heavy streaming. I choose by semantics (command vs event vs stream), ordering, throughput, replay, and cost. Risk: using Event Hubs as a task queue (no per-message ack/DLQ) or Service Bus for millions of telemetry events per second (cost/throughput).
**Follow-up traps**: "Need replay of last 7 days?" (Event Hubs retention, or Service Bus doesn't replay once completed.)

### 8.3 [Medium] How do you make consumers safe under at-least-once delivery?
**What the interviewer is really testing**: Idempotency design.
**Typical mid-level answer**: "Check if processed."
**Strong senior answer**: Assign each message a stable id (business key or idempotency key). The consumer does the work and records the id in the same transaction as the effect (an inbox/processed-messages table with a unique constraint), so a replay is a no-op. For naturally idempotent operations (upsert, set status) use those. For external side effects (sending email) use the provider's idempotency keys or accept at-least-once with dedup. Also handle poison messages with max delivery count and DLQ, plus replay tooling, and keep message handling quick with lock renewals for longer work. Service Bus duplicate detection helps on the send side within a window but is not a substitute. Risk: check-then-act races between competing consumers; rely on unique constraints, not just reads.
**Follow-up traps**: "Outbox?" (producer side: write event in the same DB transaction, publish asynchronously.)

### 8.4 [Medium] How do Event Hubs partitions and consumer groups affect scaling?
**What the interviewer is really testing**: Concrete understanding of parallelism limits.
**Typical mid-level answer**: "More consumers = more speed."
**Strong senior answer**: Parallelism is capped by partitions: each partition is read by one consumer per consumer group at a time (via EventProcessorClient with checkpointing in Blob), so 4 partitions means at most 4 active readers per group. Ordering is guaranteed only within a partition, so choose the partition key (e.g. deviceId) so events needing order land together, and avoid skewed keys causing hot partitions. Different downstream apps use separate consumer groups for independent progress. Throughput capacity is set by throughput units/processing units/capacity of the tier. I'd choose partition count for peak parallelism with some growth room because partition count is hard to reduce later. Risk: lag grows if processing per event is slow; scale by partitions, optimise handler, or batch.
**Follow-up traps**: "Checkpoint every event?" (costly; checkpoint per batch/interval, handle replay via idempotency.)

### 8.5 [Hard] Design autoscaling for queue workers on AKS or Functions. What signals and what limits?
**What the interviewer is really testing**: Scaling on the right signal while protecting downstream.
**Typical mid-level answer**: "Scale on CPU."
**Strong senior answer**: CPU is a poor signal for I/O-bound consumers; scale on queue depth or message age (KEDA Service Bus scaler, Functions scale controller). Target: backlog / (per-consumer rate x acceptable drain time) = replicas. Set min replicas to handle baseline and cold-start, and a max cap tied to downstream capacity (DB connection budget, third-party API rate limits), otherwise scaling floods the dependency. Tune prefetch/concurrency per instance, use graceful shutdown (finish in-flight, release locks) for scale-in, and cooldown to avoid flapping. Alert on oldest message age (the SLO view) rather than depth alone. Risk: scale-out amplifies failures (retries storm); use backoff and circuit breakers so failing downstream isn't hammered.
**Follow-up traps**: "Scale to zero?" (cheap, but cold start adds latency; fine for non-urgent queues.)

### 8.6 [Tricky] Batch vs stream: the business asks for "real-time" dashboards. How do you respond?
**What the interviewer is really testing**: Clarifying requirements; cost of low latency.
**Typical mid-level answer**: "Use streaming."
**Strong senior answer**: I'd ask what "real-time" means: seconds, minutes, or "yesterday is too old". Many needs are met by micro-batches every 1-5 minutes at a fraction of the complexity. True streaming (Event Hubs -> Stream Analytics/Spark/Flink -> store) brings windowing, late/out-of-order events, state, exactly-once semantics challenges, and higher cost and skills. I'd define a freshness SLO per dashboard, use streaming only where a decision is time-sensitive (fraud, alerts), and keep batch for the rest. Hybrid (lambda/kappa-like) can serve both but duplicates logic; prefer one pipeline where acceptable. Risk: over-building a stream platform for a need that a 1-minute refresh satisfies.
**Follow-up traps**: "Late events?" (watermarks and allowed lateness; decide correction policy.)

**Red flags in answers**
- Assumes exactly-once delivery without idempotency.
- Autoscales consumers without a downstream cap.
- Chooses streaming without a stated latency requirement.

**Cheat sheet**
- Async for work the caller doesn't need to finish.
- Service Bus = business messaging; Event Hubs = streams; Event Grid = notifications; Storage Queue = simple.
- At-least-once + idempotent consumer + DLQ.
- Event Hubs parallelism = partitions per consumer group.
- Scale on queue depth/age, cap by downstream capacity.
- "Real-time" must be quantified.

---

## Topic 9: Connection/thread management & API performance design

**Study notes**
- ADO.NET pooling: pool per connection string; default Max Pool Size 100; open late, close early (`using`); pool exhaustion shows as timeouts "max pool size was reached".
- Async all the way: `async/await` frees threads during I/O; blocking on async (`.Result`, `.Wait()`) risks thread-pool starvation (and deadlocks in some contexts). Thread pool injects threads slowly under starvation.
- HttpClient: reuse via `IHttpClientFactory`/typed clients; new HttpClient per call leads to socket exhaustion (TIME_WAIT) and SNAT port exhaustion; a static/long-lived client alone can miss DNS changes (factory rotates handlers, or `PooledConnectionLifetime`).
- Resilience: timeouts, retries with jitter, circuit breaker, bulkhead (Polly / Microsoft.Extensions.Http.Resilience).
- API design: avoid chatty (N calls) -> chunky but not bloated; return only needed fields; compression (gzip/Brotli) for text; HTTP/2 multiplexing; gRPC for internal low-latency binary; GraphQL gives flexible shapes but N+1 resolver problem (use DataLoader/batching) and query-cost limits; pagination; caching headers and ETags (`If-None-Match` -> 304).

```
Chatty:  client -> GET /order/1, /order/1/lines, /customer/9, /product/x ... (N round trips)
Chunky:  client -> GET /order/1?expand=lines,customer
```

### 9.1 [Easy] Why is `new HttpClient()` per request a problem and what is the fix?
**What the interviewer is really testing**: Practical .NET knowledge.
**Typical mid-level answer**: "It's IDisposable so dispose it."
**Strong senior answer**: Disposing HttpClient doesn't immediately free the underlying sockets; they linger in TIME_WAIT, and under load you can exhaust ports (SNAT on App Service/AKS egress) and pay TLS handshake costs for each call. Use `IHttpClientFactory` with typed/named clients, which pools and rotates handlers (respecting DNS changes), and attach resilience handlers (timeouts, retry, circuit breaker). A long-lived static client with `SocketsHttpHandler.PooledConnectionLifetime` set also works. Set explicit timeouts per call. Risk: default timeout of 100 s is far too long for most dependencies and ties up resources.
**Follow-up traps**: "Retries and non-idempotent POST?" (only with idempotency keys.)

### 9.2 [Follow-up] Users hit "timeout expired... max pool size was reached". Diagnose.
**What the interviewer is really testing**: Systematic pool-exhaustion reasoning.
**Typical mid-level answer**: "Increase Max Pool Size."
**Strong senior answer**: Increasing the pool is often treating the symptom and can overload the DB. I'd check: connection leaks (not disposed, early returns, DataReader left open, `TransactionScope` misuse), long-held connections (calling external APIs or doing CPU work while a connection is open), slow queries holding connections (Little's law: in-flight = rate x duration), and total connections across instances. Tools: `sys.dm_exec_sessions`/`dm_exec_connections` grouped by program/host, pool performance counters, App Insights dependency durations. Fix leaks and shorten hold time first; then size pool per Little's law and cap instances. Risk: more connections means more concurrent queries and blocking on the DB; the right fix is usually shorter work per connection.
**Follow-up traps**: "EF Core DbContext lifetime?" (scoped per request; don't hold across long operations; pooled contexts via AddDbContextPool.)

### 9.3 [Medium] Explain thread-pool starvation and how you'd recognise and fix it.
**What the interviewer is really testing**: Async correctness under load.
**Typical mid-level answer**: "Add more threads."
**Strong senior answer**: Starvation occurs when threads are blocked (sync-over-async `.Result`, `Thread.Sleep`, sync I/O) so queued work waits while CPU looks low; latency spikes appear under load, often in bursts because the pool grows slowly. Signs: low CPU with high latency, thread pool queue length growing, thread count climbing (`dotnet-counters`), stack dumps full of blocked waits. Fix by going async end to end (including DB calls, `ToListAsync`, HTTP, file I/O), removing blocking calls, and avoiding `Task.Run` wrapping of I/O in ASP.NET. Raising `SetMinThreads` is a stopgap. Offload CPU-heavy work to background workers. Risk: partial async still blocks at the seam, so audit the call chains and libraries.
**Follow-up traps**: "Is async faster?" (not per request; it scales better under concurrency by freeing threads.)

### 9.4 [Medium] An endpoint makes 40 DB queries per request. How do you find and fix it?
**What the interviewer is really testing**: N+1 detection and structural fixes.
**Typical mid-level answer**: "Add caching."
**Strong senior answer**: Detect via App Insights dependency count per operation, EF Core logging, or MiniProfiler. Typical cause: N+1 from lazy loading or loops calling the DB per item. Fixes: eager load with `Include` (watch cartesian explosion; use `AsSplitQuery`), projection with `Select` into DTOs, set-based queries with `WHERE Id IN (...)` or TVPs, batch data loaders, or a purpose-built read query via Dapper. Add a test that asserts query count for key endpoints. Caching comes after structure is right. Risk: over-eager Include loading huge graphs; project only the needed columns.
**Follow-up traps**: "Same issue in GraphQL?" (resolver N+1; DataLoader batches per request.)

### 9.5 [Hard] Choose between REST, gRPC and GraphQL for a mobile app plus internal services.
**What the interviewer is really testing**: Protocol selection by context.
**Typical mid-level answer**: "gRPC is faster, use it everywhere."
**Strong senior answer**: Mobile/browser clients to public API: REST (or GraphQL) because of tooling, caching via HTTP semantics (CDN, ETags), and browser support (gRPC needs gRPC-Web). GraphQL fits when many client types need different shapes and round trips matter (mobile on poor networks), but you must control query cost/depth, handle N+1, and caching is harder. Internal service-to-service: gRPC gives compact binary payloads, HTTP/2 multiplexing, streaming, strongly typed contracts, and lower latency, at the cost of debugging ergonomics and load-balancer needs (HTTP/2 aware LBs, L7). I'd measure the gain; often the network hop and DB dominate, and serialisation isn't the bottleneck. Decision: REST public edge, gRPC for hot internal paths where measured, GraphQL via BFF if client diversity warrants. Risk: protocol sprawl; keep to few standards.
**Follow-up traps**: "gRPC behind Azure Application Gateway/APIM?" (check HTTP/2/gRPC support in that service and tier.)

### 9.6 [Tricky] How do ETags and compression help, and what are the pitfalls?
**What the interviewer is really testing**: HTTP-level performance fundamentals.
**Typical mid-level answer**: "Cache headers."
**Strong senior answer**: ETags let clients revalidate: send `If-None-Match`; if unchanged the server returns 304 with no body, saving bandwidth (but still costing a server round trip, so combine with `Cache-Control: max-age` to avoid the round trip). ETags also enable optimistic concurrency on updates (`If-Match`, 412 on mismatch). Compression (Brotli/gzip) reduces text payloads substantially but costs CPU, so use it for JSON/HTML, not already-compressed media, and prefer compressing at CDN/Front Door where supported. Pitfalls: ETag computing that requires loading the full resource anyway (use a version/rowversion column), `Vary` headers for correct caching, and compression with secrets in responses (BREACH-class attacks, relevant for reflected secrets). Risk: wrong cache headers leak personalised content; mark private responses properly.
**Follow-up traps**: "Weak vs strong ETag?" (weak = semantically equivalent; strong = byte-identical.)

**Red flags in answers**
- `new HttpClient()` in a loop; `.Result` on async calls.
- Bumping pool size as the first and only fix.
- "gRPC everywhere" or "GraphQL fixes performance" with no trade-offs.

**Cheat sheet**
- Pool: open late, close early; instances x pool = DB load.
- Async end to end; sync-over-async starves the pool.
- IHttpClientFactory + timeouts + resilience.
- Count queries per request; kill N+1.
- REST edge, gRPC hot internal paths, GraphQL with cost limits.
- ETag + max-age + Vary; compress text only.

---

## Topic 10: Multi-tenant scale, rate limiting & fairness

**Study notes**
- Tenancy models: shared DB with TenantId column (cheapest, noisy-neighbour risk, row-level security possible), database per tenant (isolation, easier per-tenant restore/scale; Azure SQL Elastic Pools share resources; management overhead), schema per tenant (awkward), sharded hybrid (tenants mapped to shards; big tenants isolated), stamp/deployment-per-tenant-group.
- Noisy neighbour: one tenant consumes disproportionate resources. Mitigate with quotas, throttling, isolation tiers, resource governance (Elastic pool min/max per DB), queues per tenant class.
- Rate limiting algorithms: fixed window (burst at boundary), sliding window, token bucket (allows bursts, steady refill), leaky bucket, concurrency limiter. ASP.NET Core 7+ has built-in `RateLimiter` middleware; APIM has `rate-limit-by-key`/`quota-by-key`; Front Door/WAF has rate-limit rules.
- Return HTTP 429 with `Retry-After`; clients use exponential backoff with jitter.
- Fairness: per-tenant limits rather than only global; weighted by plan; protect critical paths; shed load (reject early) rather than queue unboundedly.

```
Pooled:  [Tenant A][Tenant B][Tenant C] -> one DB      (cheap, noisy)
Siloed:  A -> DB_A   B -> DB_B   (Elastic Pool shares compute)
Hybrid:  small tenants -> shared shards; whales -> dedicated
```

### 10.1 [Easy] What is a noisy neighbour and how do you contain it?
**What the interviewer is really testing**: Fairness awareness in shared systems.
**Typical mid-level answer**: "Rate limit them."
**Strong senior answer**: A noisy neighbour is a tenant whose load degrades others sharing a resource (DB, queue, CPU). I'd first make it visible: per-tenant telemetry (requests, DB time, RUs) tagged in App Insights, and a top-N tenants dashboard. Then contain: per-tenant rate limits and quotas at gateway/app, per-tenant concurrency caps, separate queues/priority lanes for heavy batch jobs, and isolation for the largest tenants (own DB or elastic pool with per-database max). Resource-governance in the DB layer (Elastic Pool caps, Resource Governor in SQL Server) protects against runaway queries. Trade-off: stricter isolation costs more; use tiers by plan. Risk: limits set without data cause false throttling; calibrate from measured baselines.
**Follow-up traps**: "A single tenant queries unbounded reports?" (cap, async export, read replica.)

### 10.2 [Follow-up] Compare shared-database and database-per-tenant for a SaaS with 2000 tenants of varied size.
**What the interviewer is really testing**: Trade-off reasoning on isolation, cost, operations.
**Typical mid-level answer**: "DB per tenant is more secure."
**Strong senior answer**: Shared DB with TenantId: lowest cost and simplest schema rollout, but risk of cross-tenant leaks (mitigate via global query filters and row-level security), noisy neighbours, and awkward per-tenant restore/export. DB per tenant: strong isolation, per-tenant restore, tenant-specific scaling, easy offboarding, but 2000 databases mean heavy migration orchestration, monitoring, and cost; Elastic Pools reduce cost by sharing DTU/vCores across spiky tenants, though pool limits on database count and resources apply. For varied sizes I'd propose a hybrid: pooled small tenants in shared databases/pools, a shard map (tenant -> database), and promotion path for large or regulated tenants. Schema migrations get automated (rolling, versioned, backwards compatible). Risk: operational tooling is the real cost; budget for it.
**Follow-up traps**: "Tenant doesn't fit its shard anymore?" (move tenant with online migration and shard-map flip.)

### 10.3 [Medium] Token bucket vs fixed window rate limiting?
**What the interviewer is really testing**: Algorithm trade-offs.
**Typical mid-level answer**: "Both limit requests."
**Strong senior answer**: Fixed window counts requests per interval; simple but allows 2x bursts around the boundary (end of one window, start of the next). Sliding window smooths this at more bookkeeping cost. Token bucket refills at a steady rate and allows controlled bursts up to bucket size, which matches real clients who are bursty but bounded on average. Concurrency limiting caps in-flight requests, which protects against slow-request pileups and is the right control for expensive endpoints. I'd combine: token bucket per tenant/API key for rate, concurrency limit for heavy endpoints, and a global limit as a safety net. In a multi-instance deployment, local limiters give per-instance limits; true global limits need shared state (Redis) or gateway enforcement (APIM, Front Door), trading some latency for accuracy. Risk: Redis in the hot path becomes a dependency; fail open or closed deliberately.
**Follow-up traps**: "Fail open or closed when limiter store is down?" (depends: protect backend vs availability; decide per API.)

### 10.4 [Medium] What should a throttled client see, and how do you stop retry storms?
**What the interviewer is really testing**: Cooperative load management.
**Typical mid-level answer**: "Return 429."
**Strong senior answer**: Return 429 (or 503 for overload) with a `Retry-After` header and a clear error body; document limits. Clients should back off exponentially with jitter, cap retries, and respect the header; otherwise synchronised retries create retry storms that prolong the incident. On our side, shed load early (cheap rejection before expensive work), prioritise critical traffic, and keep limits per tenant to avoid punishing everyone. Monitor throttle rates per tenant as a product signal (upsell/capacity). Risk: internal services retrying at every layer multiply load (3 layers x 3 retries = 27x); retry at one layer only and use retry budgets.
**Follow-up traps**: "Idempotency?" (retries need idempotency keys for POST.)

### 10.5 [Hard] Design per-tenant throttling for an API on APIM + ASP.NET Core + Azure SQL.
**What the interviewer is really testing**: End-to-end design with layers.
**Typical mid-level answer**: "Use APIM rate-limit policy."
**Strong senior answer**: I'd identify the tenant from the token claim/subscription key at the edge. APIM applies `rate-limit-by-key` and `quota-by-key` keyed on tenant, with plan-specific values (different products/tiers). Behind it, ASP.NET Core's RateLimiter provides a concurrency limiter per tenant on expensive endpoints, because gateway rate limits don't capture request cost. Heavy operations (exports, bulk imports) go async through queues with per-tenant concurrency caps (e.g. Service Bus sessions keyed by tenant), so one tenant cannot monopolise workers. In SQL, protect with timeouts, row limits, and for large tenants separate pools/databases. Telemetry per tenant feeds dashboards and alerts. Risk: APIM rate limits are approximate in multi-region/multi-unit deployments; treat them as protective rather than exact accounting, and bill from metered usage data.
**Follow-up traps**: "Burst allowance for premium?" (token-bucket-like semantics with larger counters.)

### 10.6 [Tricky] A big customer says "your throttling broke our nightly sync". How do you handle it?
**What the interviewer is really testing**: Customer-impact balance with system protection.
**Typical mid-level answer**: "Raise their limit."
**Strong senior answer**: First understand the load pattern: is the sync doing thousands of small calls that a bulk endpoint would serve better? Raising limits blindly shifts risk onto other tenants. Options: provide a bulk/batch API or async import with a job id, schedule sync windows (off-peak quota), provide a webhook/changes feed so they stop polling, and offer a higher plan with committed capacity if justified. Meanwhile apply a temporary, measured limit increase with monitoring. I'd also share their usage telemetry with them. Risk: special-casing creates unmaintainable exceptions; capture it as a product tier or API capability.
**Follow-up traps**: "How do you know the increase is safe?" (load test at that rate; monitor other tenants' latency.)

**Red flags in answers**
- Only global limits, no per-tenant view.
- Ignores isolation tiers and cost per tenant.
- No 429/Retry-After contract; retries at every layer.

**Cheat sheet**
- Pooled (cheap, noisy) vs siloed (isolated, costly) vs hybrid shard map.
- Tag telemetry with tenant; find top-N consumers.
- Token bucket for rate; concurrency limit for cost.
- 429 + Retry-After + jittered backoff.
- Heavy work async with per-tenant caps.
- Global limits need shared state or gateway.

---

## Topic 11: Global distribution

**Study notes**
- Goals: lower latency for distant users, resilience to region failure, data residency. Costs: complexity, consistency compromises, money (duplicated infrastructure, egress).
- Edge: Azure Front Door (global anycast entry, TLS termination, WAF, caching, routing/health probes, failover) and Azure CDN for static content. Traffic Manager is DNS-based routing.
- Patterns: active-passive (warm standby; simpler; failover RTO), active-active (stateless tier in each region; data layer is the hard part).
- Data: Azure SQL auto-failover groups (async geo-replication; single writer; RPO seconds, not zero), read-only geo-replica reads; Cosmos DB multi-region (single-write or multi-write with conflict resolution; consistency levels: Strong, Bounded Staleness, Session, Consistent Prefix, Eventual; Strong with multi-region has constraints/latency); Blob GRS/RA-GRS; Redis geo-replication (Enterprise/Premium tiers).
- CAP/PACELC in plain words: during partition choose consistency or availability; otherwise trade latency vs consistency.
- Data residency: pin tenants to a region (cell-based / geo-partitioned).

```
User -> Front Door -> Region A (app + data primary)
                   -> Region B (app + replica / multi-write)
Cross-region write = speed-of-light latency (tens-hundreds ms)
```

### 11.1 [Easy] Why would you go multi-region and what does it not solve?
**What the interviewer is really testing**: Motivation and realism.
**Typical mid-level answer**: "For high availability."
**Strong senior answer**: Three drivers: latency for global users, resilience against a regional outage (beyond zone redundancy), and legal data residency. Often zone-redundant deployment in a single region already gives high availability against datacentre failures at much lower complexity; I'd quantify the required SLA and RTO/RPO before going multi-region. Multi-region does not remove bugs, bad deployments or data corruption (replicated instantly), and it doubles cost and operational surface. Decision: start with zone-redundant single region plus tested DR (warm standby), and go active-active only when numbers justify it. Risk: untested failover is not a plan; run game days.
**Follow-up traps**: "RPO of async geo-replication?" (non-zero; data loss window possible on forced failover.)

### 11.2 [Follow-up] Active-active vs active-passive: how do you choose?
**What the interviewer is really testing**: Trade-off on complexity and data.
**Typical mid-level answer**: "Active-active is better."
**Strong senior answer**: Active-passive (primary region serves, secondary warm) is simpler: single writer for SQL via failover group, easier consistency, lower cost; the price is failover time (RTO) and unused capacity. Active-active serves users from the nearest region and survives region loss with little downtime, but writes need a strategy: partition data by user/tenant home region, use Cosmos multi-write with conflict resolution (last-writer-wins or custom), or accept cross-region writes to a primary. Anything needing global uniqueness or strong consistency (inventory, payments) gets hard. I'd choose active-passive unless the business demands near-zero RTO or low write latency globally, and even then segment: read-mostly content active-active, transactional core active-passive. Risk: split-brain and conflict handling errors.
**Follow-up traps**: "How do you test failover?" (scheduled drills, chaos experiments.)

### 11.3 [Medium] How do you choose a Cosmos DB consistency level?
**What the interviewer is really testing**: Consistency vs latency/cost trade-off with specifics.
**Typical mid-level answer**: "Strong is safest."
**Strong senior answer**: Levels from strongest to weakest: Strong, Bounded Staleness, Session, Consistent Prefix, Eventual. Session is the default and fits most apps: a client sees its own writes and monotonic reads. Strong gives linearizability but increases write latency across regions and has limitations for multi-region setups; use only where required (e.g. a ledger-like invariant). Bounded staleness gives a quantified lag bound for analytics-like reads. Weaker levels use fewer RUs for reads in some cases and give lower latency. I'd choose per workload and can relax per request. Risk: using Session with a load balancer that doesn't pass the session token causes stale reads from different instances; propagate it.
**Follow-up traps**: "Read your writes across API instances?" (pass session token via client or cache.)

### 11.4 [Medium] Users in Australia see slow pages from a West Europe deployment. Options?
**What the interviewer is really testing**: Layered latency fixes.
**Typical mid-level answer**: "Deploy to Australia."
**Strong senior answer**: Measure first: how much is network RTT (~250-300 ms per round trip to Europe) times round trips (DNS, TCP/TLS, chatty API calls), vs server time. Cheapest wins: CDN/Front Door for static assets and cacheable API responses, TLS termination at the edge with persistent connections to origin, HTTP/2/3, fewer and bigger requests (reduce chattiness), and compression. Next: API in an Australian region with reads from a local replica/Cosmos region while writes go to the primary (accepting write latency and replication lag). Full regional deployment is last because of data residency and cost. Risk: read-your-writes anomalies after writing to a distant primary; route that user's reads to primary briefly.
**Follow-up traps**: "How many round trips does a cold HTTPS request need?" (DNS + TCP + TLS before first byte; edge termination shortens the distance of each.)

### 11.5 [Hard] Design a global app where each user's data must stay in their home region (EU/US).
**What the interviewer is really testing**: Geo-partitioning and routing.
**Typical mid-level answer**: "Two databases."
**Strong senior answer**: Use a cell/stamp architecture: each region hosts a full stack and the data for its users. A global lightweight directory maps userId/tenant -> home region (small, replicated, non-sensitive), or the region is encoded in the id/subdomain/token claim. Front Door routes to the correct region by hostname/path/header; if a request arrives in the wrong region it is proxied or redirected. Cross-region features (global search, admin reporting) use aggregated, anonymised or region-local queries; analytics pipelines must respect residency too. Identity (Entra ID/B2C) location needs checking. Operational concerns: deployments per cell with staged rollouts, per-region backups and DR within the legal boundary (paired region may cross borders; verify). Risk: users who travel and teams wanting global joins; decide policies up front.
**Follow-up traps**: "User moves from EU to US?" (explicit migration workflow, not implicit replication.)

**Red flags in answers**
- Multi-region as a default, with no RTO/RPO numbers.
- "Strong consistency everywhere" in multi-region.
- Forgetting data residency, egress cost, or failover testing.

**Cheat sheet**
- Zone redundancy first; multi-region when RTO/latency/residency demand it.
- Front Door/CDN: cheapest global latency win.
- SQL failover groups: single writer, async, non-zero RPO.
- Cosmos: Session default; Strong costs latency.
- Active-active needs a write/conflict strategy.
- Test failover regularly.

---

## Topic 12: Cost-aware architecture (FinOps)

**Study notes**
- Unit economics: cost per request, per tenant, per transaction; compare to revenue; tag resources (tenant/product/env) and use Cost Management + budgets/alerts.
- Levers: right-size (Advisor, utilisation), autoscale down, reserved instances/savings plans (1-3 year commit; good for steady baseline), Azure Hybrid Benefit, spot VMs for interruptible work, dev/test scheduling (shut down), storage tiering (hot/cool/archive), serverless (Functions consumption, Azure SQL serverless auto-pause) for spiky/idle, but can be costlier at steady high load.
- Hidden costs: egress (data leaving Azure/regions/zones), cross-region replication, log ingestion (Log Analytics / App Insights per GB), high-cardinality metrics, Cosmos RU overprovision, over-provisioned DB tiers, idle non-prod, orphaned disks/IPs.
- Cost vs reliability: don't cut redundancy blindly; cut waste first.
- Log/telemetry cost controls: sampling (App Insights adaptive), log levels, Basic Logs tier / data collection rules, retention tuning, daily cap (careful: loses data).

### 12.1 [Easy] How do you think about cost when designing a system?
**What the interviewer is really testing**: Cost as an architectural quality attribute.
**Typical mid-level answer**: "Use cheaper SKUs."
**Strong senior answer**: I estimate cost per unit of business value (per request, per tenant, per order) at current and 10x scale, since linear costs can break margins. I identify the top cost drivers (usually DB, compute, egress, logging) and design them consciously: right-sized compute with autoscale, caching to cut DB cost, storage tiers, and avoiding chatty cross-region traffic. I tag resources for allocation and set budgets/alerts. Decisions are trade-offs: reserved capacity for baseline, on-demand/serverless for spikes. Risk: optimising cost at the expense of reliability or engineering time; weigh engineering cost vs savings.
**Follow-up traps**: "How do you show cost per tenant?" (tags on dedicated resources; metered usage for shared ones.)

### 12.2 [Follow-up] Reserved capacity vs serverless vs pay-as-you-go: when each?
**What the interviewer is really testing**: Matching pricing to workload shape.
**Typical mid-level answer**: "Serverless is cheaper."
**Strong senior answer**: Steady, predictable baseline (production DB, always-on API) fits reservations or savings plans with sizeable discounts, in exchange for commitment risk. Spiky/idle workloads (dev environments, rarely used APIs, event-driven jobs) fit serverless/consumption: pay for use, but cold starts, per-execution pricing, and scale limits matter; at steady high utilisation, provisioned is often cheaper. Azure SQL serverless auto-pauses and bills per second of vCore usage, great for intermittent databases, poor for always-busy ones. Strategy: reserve the floor, autoscale/serverless above it, use spot for fault-tolerant batch. Verify with actual usage data, 3-6 months of it if possible. Risk: locked-in reservations when you re-architect; use scope-flexible reservations and review quarterly.
**Follow-up traps**: "Cold start impact?" (latency for first request; premium/always-ready instances cost more.)

### 12.3 [Medium] Where do hidden costs come from in Azure bills?
**What the interviewer is really testing**: Practical experience with bills.
**Typical mid-level answer**: "Storage and VMs."
**Strong senior answer**: Frequent culprits: log ingestion (verbose debug logs, full request/response bodies, high-volume dependency telemetry billed per GB), egress and cross-region/cross-AZ traffic, NAT/Firewall data processing, over-provisioned Cosmos RUs (paying for peak 24/7), premium DB tiers sized for a past peak, idle non-prod environments, orphaned resources, Premium SSD sizes, and chatty microservices across zones. I'd analyse via Cost Management grouped by service/tag, then check top meters. For logging, sample, reduce levels, route high-volume low-value logs to cheaper tiers, and set retention by need. Risk: slashing observability harms incident response; keep errors and key traces at full fidelity.
**Follow-up traps**: "Daily cap on Log Analytics?" (stops ingestion when reached, so you lose data; use cautiously.)

### 12.4 [Hard] Reduce the Azure bill by 30% without hurting reliability. Walk me through it.
**What the interviewer is really testing**: Prioritised, safe approach.
**Typical mid-level answer**: "Downsize everything."
**Strong senior answer**: I'd begin by getting a breakdown: top 5 services and tags, trend vs usage, then categorise savings by risk: (1) zero-risk waste: unattached disks/IPs, idle environments, dev/test shutdown schedules, old snapshots, unneeded log retention; (2) low-risk commitments: reservations/savings plan and Hybrid Benefit for steady baseline; (3) right-sizing based on 2-4 weeks of utilisation, keeping headroom and checking p99 and failover capacity; (4) architectural: caching to reduce DB tier, storage lifecycle policies, telemetry sampling, Cosmos RU autoscale or reduce indexing policy, moving batch to spot. Do changes in waves with SLO monitoring and rollback, and never remove redundancy that the SLO depends on. Report savings vs risk and track in a FinOps dashboard. Risk: right-sizing before understanding peak patterns (month-end) causes outages; include business calendar.
**Follow-up traps**: "Which first?" (waste and commitments: highest saving-to-risk ratio.)

### 12.5 [Tricky] Is the cheapest architecture the one with the lowest monthly Azure bill?
**What the interviewer is really testing**: Total cost thinking.
**Typical mid-level answer**: "Yes, lower bill is better."
**Strong senior answer**: No; total cost includes engineering time, operational burden, incident cost, and opportunity cost. A managed service may cost more per month than self-hosting but saves on-call and patching; a complex multi-store design may reduce the bill but increase maintenance and risk. I'd compare options on cost per unit, risk-adjusted, over 1-3 years, with a sensitivity to growth. Also compare to revenue: a 10% bill increase to unlock 2x capacity or avoid a major outage can be the better deal. Risk: false precision; use ranges and revisit as real data arrives.
**Follow-up traps**: "Build vs buy?" (opportunity cost and differentiation.)

**Red flags in answers**
- "Turn off redundancy" or "cut logs entirely" as savings.
- No unit economics or tagging.
- Assuming serverless is always cheaper.

**Cheat sheet**
- Track cost per request/tenant; tag everything.
- Reserve the floor, scale/serverless above it, spot for batch.
- Watch egress, log ingestion, over-provisioned RUs, idle non-prod.
- Savings order: waste -> commitments -> right-size -> redesign.
- Keep SLO monitoring during every cost change.
- Compare total cost incl. people.

---

## Topic 13: Performance budgets, SLOs & load testing

**Study notes**
- SLI = measured indicator (e.g. % requests < 300 ms); SLO = target (99% < 300 ms over 30 days); SLA = contractual. Error budget = 1 - SLO; spend it on releases/experiments.
- Percentiles over averages: p50 = typical, p95/p99 = tail; tails dominate user experience and fan-out calls (if one request calls 10 services, the chance of hitting a slow one rises).
- Performance budget: allocate latency across layers (e.g. 300 ms total = 20 gateway + 100 app + 120 DB + 60 slack), plus page-weight/bundle budgets for front end.
- Test types: load (expected peak), stress/breakpoint (find the limit), spike (sudden surge), soak/endurance (hours; leaks, growth, connection exhaustion), scalability (step up), capacity/regression in CI.
- Tools: Azure Load Testing (JMeter/Locust-based), k6, JMeter, NBomber; observe with App Insights, SQL metrics.
- Realism: production-like data volume and distribution, realistic user mix and think time, open vs closed workload models (arrival-rate based is more realistic for public traffic), warm vs cold caches, test dependencies stubbed deliberately.
- Coordinated omission: load generators that wait for slow responses understate latency; use arrival-rate-based tools.

### 13.1 [Easy] Why percentiles instead of average latency?
**What the interviewer is really testing**: Statistical literacy in performance.
**Typical mid-level answer**: "Averages hide outliers."
**Strong senior answer**: Averages blend fast and slow requests so a service with 1% of requests at 5 s still shows a fine mean; users experience those slow ones, and with fan-out (a page making 20 calls) the chance of hitting at least one slow call becomes large. p95/p99 expose tails. I define SLIs on percentiles per critical user journey, and look at histograms/heatmaps rather than only a single number. Averages of percentiles across instances are invalid; aggregate histograms instead. Risk: p99 on low-volume endpoints is noisy; use adequate sample windows.
**Follow-up traps**: "Can you average p95 across servers?" (no; merge distributions.)

### 13.2 [Follow-up] How do you set an SLO for a new API?
**What the interviewer is really testing**: Tie to user need and feasibility.
**Typical mid-level answer**: "99.99% uptime."
**Strong senior answer**: Start from user journeys and business impact: what latency makes users abandon, what availability is needed. Look at baseline data (or a prototype load test) to see what's feasible; set the SLO slightly below current capability so the error budget is meaningful. Dependencies cap the achievable value: composite availability is roughly the product of the chain (three 99.9% dependencies in series ~ 99.7%). Use few SLOs (availability, latency for key operations, maybe freshness for async), alert on burn rate rather than single thresholds, and agree on what happens when the budget is exhausted (freeze features, focus on reliability). Risk: aspirational SLOs without budget or engineering response are noise.
**Follow-up traps**: "Is 100% a good target?" (no; unaffordable and blocks change.)

### 13.3 [Medium] Design a load-testing strategy for an e-commerce checkout before a major launch.
**What the interviewer is really testing**: Realistic, risk-based testing.
**Typical mid-level answer**: "Run JMeter with 1000 users."
**Strong senior answer**: Define targets from estimates: peak orders/min, concurrency, and success criteria (p95, error rate). Build the workload model from production analytics: browse/search/cart/checkout mix, think time, arrival-rate based. Use a production-like environment and data volume (empty DBs lie: plans and cache behaviour differ), seed with realistic data skew (hot products), and stub or sandbox payment providers with realistic latencies. Run: baseline, load at expected peak, step test to the breakpoint, spike test (sudden 5-10x), soak (4-12 h) to catch leaks/pool exhaustion, and failure injection (kill an instance, slow dependency, DB failover). Monitor every tier with USE/RED, record results, and fix the top bottleneck, then retest. Risk: testing against shared environments or third parties without permission; also a test that doesn't reproduce production data size gives false comfort.
**Follow-up traps**: "How do you test in production safely?" (small percentage synthetic load, feature flags, with tenant isolation and kill switch.)

### 13.4 [Medium] How do you interpret a test where throughput plateaus and latency climbs?
**What the interviewer is really testing**: Reading saturation curves.
**Typical mid-level answer**: "System is at its limit."
**Strong senior answer**: Plateau in throughput with rising latency is the saturation knee: something is a bottleneck (queueing builds). Correlate with resource metrics to find which tier saturates first: app CPU, thread pool queue, DB CPU/IO/locks, connection pool, downstream rate limit. If CPU of all tiers is low yet latency climbs, suspect locking, pool limits, or the load generator itself being the bottleneck. The usable capacity is below the knee (e.g. 70% of it) to leave headroom. Then change one thing and re-run to confirm the knee moves. Risk: attributing to the wrong tier; confirm with causal experiments, not correlation only.
**Follow-up traps**: "Errors appear before the plateau?" (check limits/timeouts, 429s, connection failures.)

### 13.5 [Hard] Latency p99 is fine in tests but bad in production. Why might that be?
**What the interviewer is really testing**: Test-vs-prod gaps.
**Typical mid-level answer**: "Production is bigger."
**Strong senior answer**: Typical gaps: data volume/skew (big tenants, hot keys), cold caches after deploys, noisy neighbours, real client behaviour (large payloads, slow clients, retries), mixed workloads (reports, batch jobs running concurrently), background tasks (index maintenance, backups, GC), network paths (WAF, egress/NAT), and third-party variability. Also closed-loop test generators understate tail latency (coordinated omission). I'd compare production traces for slow requests to find common attributes (tenant, endpoint, time of day, instance), and enrich test models with those. Add continuous low-level synthetic probes and real-user monitoring. Risk: chasing averages; investigate by slicing outliers.
**Follow-up traps**: "How do you slice outliers?" (App Insights custom dimensions, operation id, per-tenant percentiles.)

### 13.6 [Tricky] A product owner wants "fast" but won't define it. What do you do?
**What the interviewer is really testing**: Turning vague requirements into measurable targets.
**Typical mid-level answer**: "Make it as fast as possible."
**Strong senior answer**: Convert "fast" into user-journey metrics through questions: what's the most important action, what happens now, what do competitors/users tolerate? Propose a starter budget (e.g. search results < 500 ms p95, page interactive < 2.5 s on typical mobile), label as hypothesis, instrument, and review after two weeks of data. Use research like conversion vs latency from our own analytics if available. This avoids both gold-plating and neglect, and creates a shared language for trade-offs ("this feature adds 80 ms; is it worth it?"). Risk: setting a number without measurement capability; ship telemetry first.
**Follow-up traps**: "Who owns the budget?" (shared: product sets priority, engineering owns feasibility.)

**Red flags in answers**
- Reporting average latency only; no percentiles or SLOs.
- Load testing on an empty or tiny database.
- One test type only (no soak/spike/breakpoint).

**Cheat sheet**
- SLI -> SLO -> error budget; alert on burn rate.
- Percentiles; never average percentiles.
- Chain availability multiplies.
- Test types: load, stress, spike, soak, failure injection.
- Production-like data and arrival-rate workloads.
- Capacity = below the knee, not at it.

---

## Topic 14: Database scaling pains, reporting & analytics without hurting OLTP

**Study notes**
- Locking/blocking: readers block writers under default READ COMMITTED (locks); enable READ_COMMITTED_SNAPSHOT (RCSI) to reduce blocking at the cost of tempdb/version store; Azure SQL has RCSI on by default. Keep transactions short, access objects in consistent order to reduce deadlocks, use appropriate isolation.
- Large tables: maintenance (index rebuild/stats) time grows; use online/resumable operations, incremental statistics with partitioning, batch deletes, partition switching for archival.
- Migrations on big tables: expand/contract (add nullable column, backfill in batches, switch code, then drop), avoid long locks; online index operations; test on production-sized copies.
- Analytics: columnstore indexes (clustered for warehouse/facts, nonclustered for real-time operational analytics on OLTP), batch-mode processing; separate OLAP from OLTP via replicas, CDC/Change Tracking/Change Feed to a warehouse (Synapse/Fabric/ADLS), Azure Synapse Link (for Cosmos and SQL) to avoid ETL on the OLTP.
- Reporting isolation: read replica (ApplicationIntent=ReadOnly), snapshot/cache pre-aggregations, resource governor, query timeouts/limits, scheduled off-peak exports.

```
OLTP DB --CDC/Change feed--> Landing (ADLS) --> Warehouse/Lakehouse --> BI (Power BI)
   \--readable replica (light reporting)
```

### 14.1 [Easy] Why would you enable RCSI and what's the catch?
**What the interviewer is really testing**: Concurrency tuning understanding.
**Typical mid-level answer**: "Avoids locks."
**Strong senior answer**: Under default READ COMMITTED, readers take shared locks and block with writers; RCSI makes readers read the last committed row version from the version store, so readers don't block writers and vice versa, which fixes a common source of blocking in OLTP apps. Costs: version store lives in tempdb (or the PVR in the user DB with ADR), extra IO/space, long-running transactions keep versions alive; and semantics change: readers don't wait for in-flight writes, so logic relying on blocking behaviour (some queue-table patterns) may change. Test the app. Risk: version store bloat from forgotten open transactions; monitor.
**Follow-up traps**: "RCSI vs SNAPSHOT isolation?" (snapshot is transaction-level consistent and can raise update conflicts.)

### 14.2 [Follow-up] How do you run a schema change on a 2 TB table without downtime?
**What the interviewer is really testing**: Safe evolution at scale.
**Typical mid-level answer**: "Run the ALTER at night."
**Strong senior answer**: Use expand/contract. Add the new nullable column or table (metadata-only for many cases), deploy code that writes both old and new, backfill in small batches with throttling (to avoid log growth and blocking), verify, switch reads, then remove the old structure in a later release. For index changes use ONLINE = ON (and RESUMABLE where available), measuring log and tempdb impact. Test on a restored production-size copy to know duration and locks. Always have a rollback path (feature flag, still writing old column) and monitor blocking and replication lag during backfill. Risk: schema-modification locks queueing behind long transactions and blocking all traffic; set `LOCK_TIMEOUT` and run during low-activity windows.
**Follow-up traps**: "Change a column type?" (add new column, migrate, swap; direct ALTER may rewrite the table.)

### 14.3 [Medium] A report query kills the OLTP database. Short-term and long-term fixes?
**What the interviewer is really testing**: Triage plus architecture.
**Typical mid-level answer**: "Add an index / kill the query."
**Strong senior answer**: Immediate: identify and stop the offender, protect OLTP with a query timeout/limit, and move the report to a readable secondary (`ApplicationIntent=ReadOnly`) or schedule off-peak; Resource Governor (SQL Server) or Elastic Pool limits to cap impact; if it is a missing index or bad plan, a targeted fix. Medium term: pre-aggregate (summary tables, indexed views), cache results, and use nonclustered columnstore for operational analytics where light. Long term: separate analytics store (CDC or change feed -> lake/warehouse -> Power BI), so OLTP never serves heavy aggregation, with freshness agreed with business. Risk: replicas still affect primary through replication/redo and have lag; the warehouse adds pipeline ownership and data quality concerns.
**Follow-up traps**: "Replica also gets slow?" (it's still a finite resource; heavy queries still need isolation or a warehouse.)

### 14.4 [Medium] When do you use columnstore, and what are its trade-offs?
**What the interviewer is really testing**: Matching storage format to workload.
**Typical mid-level answer**: "For big tables."
**Strong senior answer**: Columnstore stores data by column with compression and batch-mode execution; it excels at scans/aggregations over many rows and few columns (fact tables), often with 5-10x compression. Clustered columnstore suits warehouse fact tables; nonclustered columnstore on an OLTP table enables real-time analytics on fresh data without a separate pipeline, at the cost of extra write overhead and delta-store/rowgroup maintenance. It's poor for point lookups and frequent single-row updates (though combined with B-tree indexes it can work). Decision: use for analytic queries on large append-heavy data; avoid for hot OLTP row access. Risk: trickle inserts producing many small rowgroups; batch loads (>=102,400 rows) help.
**Follow-up traps**: "Updates on columnstore?" (deleted bitmap + delta store; heavy updates degrade.)

### 14.5 [Hard] Design CDC-based feeding of a warehouse from OLTP. Risks?
**What the interviewer is really testing**: Data pipeline design and OLTP protection.
**Typical mid-level answer**: "Nightly ETL job."
**Strong senior answer**: Options: SQL Server CDC or Change Tracking, Azure SQL CDC, Azure Synapse Link/Fabric mirroring, or app-level events/outbox. CDC reads the transaction log asynchronously, so OLTP overhead is modest (but not zero: log retention, cleanup job, capture agent). Land changes in ADLS/Bronze as Parquet, transform to curated tables (silver/gold) with idempotent merges, handle deletes, schema evolution, late/out-of-order changes, and reconciliation counts against source. Define freshness SLO (e.g. 5-15 min) and monitoring on lag. Governance and PII handling carry over to the warehouse. Risk: CDC retention too short during pipeline outages leads to data gaps requiring re-snapshot; design replay/backfill from the start.
**Follow-up traps**: "CDC vs Change Tracking?" (CT only says which rows changed, no history of values; CDC captures column changes.)

### 14.6 [Tricky] Deadlocks appeared after scaling out the app. Why, and what do you do?
**What the interviewer is really testing**: Concurrency effects of scale.
**Typical mid-level answer**: "Retry on deadlock."
**Strong senior answer**: More concurrent writers raise the probability of conflicting lock orders and escalation, exposing latent issues (inconsistent access order across code paths, long transactions, missing indexes causing scans and wide locks, hot rows). Capture deadlock graphs (Extended Events system_health), identify the objects and statements, then fix: consistent ordering, shorter transactions, supporting indexes for narrow locks, RCSI, avoid read-then-update patterns (use `UPDLOCK` or atomic updates), and batch work with limited concurrency. Retry with backoff is a safety net (error 1205) but not a fix. Risk: increasing isolation or hints without understanding can worsen blocking.
**Follow-up traps**: "Queue table pattern?" (READPAST/UPDLOCK/ROWLOCK dequeue.)

**Red flags in answers**
- "Just do it at night" for large table changes with no online strategy.
- Reports run on the primary with no limits.
- Retry-on-deadlock as the only fix.

**Cheat sheet**
- RCSI reduces reader/writer blocking; watch version store.
- Expand/contract with batched backfill.
- Report isolation: replica -> pre-aggregate -> warehouse.
- Columnstore: scans/aggregates, not point lookups.
- CDC -> lake -> warehouse; reconcile and monitor lag.
- Deadlocks: ordering, short transactions, indexes.

---

## Topic 15: Search, autocomplete & file/media handling at scale

**Study notes**
- Search: SQL `LIKE '%x%'` can't use indexes; full-text search (SQL FTS) handles word-based queries; Azure AI Search gives relevance ranking, facets, filters, fuzzy, synonyms, vector/hybrid search; Elasticsearch is the alternative. The index is a derived store: sync via indexers (SQL, Cosmos, Blob) or push API; accept lag; support full rebuild.
- Autocomplete: prefix/suggester (AI Search suggesters/ngram analyzers), strict latency (~100 ms), debounce on client, cache popular prefixes, limit results, handle typos, protect against abuse.
- Index sizing: replicas (query throughput, availability) and partitions (storage/index size) in AI Search; plan limits per tier.
- Files: never through the app server when avoidable. Direct-to-Blob upload with SAS (short-lived, least privilege, user delegation SAS) or use of Azure Storage client with chunks; large files: block blobs with parallel block upload; resumable uploads; virus scanning (Defender for Storage) before use; metadata in SQL, bytes in Blob; CDN for delivery; image/video processing async (Functions/queues, Media services alternatives); lifecycle management tiers; avoid storing blobs in SQL for large files (FILESTREAM/varbinary(max) bloats backups and buffer pool).

```
Client -> API (get SAS URL) ; Client -> Blob (direct upload) ; Event Grid -> Function (scan/resize) -> metadata in SQL
Download: Client -> CDN -> Blob
```

### 15.1 [Easy] Why not use `LIKE '%term%'` for search at scale?
**What the interviewer is really testing**: Knowing when a database isn't a search engine.
**Typical mid-level answer**: "It's slow."
**Strong senior answer**: A leading wildcard cannot use a B-tree index seek, so it scans the table; cost grows with rows and also with concurrency, and it has no relevance ranking, typo tolerance, stemming or facets. For small tables or admin tools it's fine. For user-facing search at scale, use SQL full-text for simple word search or a dedicated engine like Azure AI Search, fed by indexers/events, with the database as system of record. Trade-off: another system to operate, lag in freshness and the need for reindex/rebuild. Risk: relevance tuning is real work; plan time for analyzers, scoring profiles and testing.
**Follow-up traps**: "Trailing wildcard `LIKE 'abc%'`?" (can seek on an index.)

### 15.2 [Follow-up] Design autocomplete for 5M products with 2000 queries/sec at peak.
**What the interviewer is really testing**: Latency-driven design with caching and client behaviour.
**Typical mid-level answer**: "Query the DB on every keystroke."
**Strong senior answer**: Requirements: ~100 ms end-to-end, high QPS, tolerance for slightly stale suggestions. Serve from a search index with a suggester or edge-ngram analyzer in Azure AI Search (replicas sized for QPS), returning few fields and max ~8 results. Client debounces (150-250 ms), cancels in-flight requests, and starts after 2-3 chars. Cache popular prefixes at CDN/Redis (short TTL), because query distribution is heavily skewed, reducing backend load substantially. Rank by popularity/recency with a precomputed signal. Rate limit per client. Keep the index updated via indexer or change events within minutes. Risk: relevance quality and typos; measure click-through on suggestions and iterate.
**Follow-up traps**: "How do you handle personalised suggestions?" (separate small per-user layer merged client/server-side, keeping shared cache for the general part.)

### 15.3 [Medium] How would you handle a 2 GB file upload feature?
**What the interviewer is really testing**: Not funnelling big data through the API.
**Typical mid-level answer**: "Upload via the API endpoint."
**Strong senior answer**: Routing 2 GB through the web tier ties up memory, threads, request limits (Kestrel/IIS/App Service/Front Door caps) and bandwidth. Instead the API authorises and issues a short-lived user-delegation SAS scoped to a specific blob path (write-only, minutes lifetime), the client uploads directly to Blob in blocks (parallel, resumable, retry per block), and the app is notified via Event Grid on blob creation. A Function/worker then scans (Defender for Storage), validates type/size, extracts metadata into SQL and moves the blob to the final container. Downloads served via SAS/CDN. Risk: SAS leakage (short TTL, narrow scope, HTTPS only), orphaned uploads (lifecycle rules to delete incomplete blobs), and content validation (don't trust client-declared type).
**Follow-up traps**: "Quota/abuse control?" (check size/user limits before issuing SAS; container-level lifecycle and monitoring.)

### 15.4 [Medium] Should images be stored in SQL Server or Blob Storage?
**What the interviewer is really testing**: Data placement judgment.
**Typical mid-level answer**: "Blob is cheaper."
**Strong senior answer**: For anything beyond small blobs, store bytes in Blob Storage and keep references and metadata in SQL. Blobs in SQL inflate the database, backups, restores, replication, and buffer pool pressure, and cost more per GB than Blob tiers. Blob also integrates with CDN, lifecycle tiering, and direct client access. Advantages of SQL: transactional consistency with metadata; but we can emulate via ordering (upload then commit metadata; cleanup job for orphans). Very small blobs (icons, a few KB) can reasonably stay inline if it simplifies things. Risk: referential gaps between DB and blob; add reconciliation jobs and soft deletes.
**Follow-up traps**: "Transactional consistency between DB row and blob?" (not atomic; use two-phase approach and idempotent cleanup.)

### 15.5 [Hard] How do you deliver and process millions of user images (thumbnails, formats) efficiently?
**What the interviewer is really testing**: Media pipeline and CDN design.
**Typical mid-level answer**: "Resize on request in the API."
**Strong senior answer**: Choose between pre-generation and on-demand. Pre-generate a fixed set of sizes asynchronously on upload (Event Grid -> queue -> workers), good for predictable sizes and fast delivery; on-demand resizing at the edge or via an image service with CDN caching handles many variants but needs protection against unbounded size permutations (allow-list sizes) and cold-miss latency. Serve through CDN with long immutable cache headers and content-hash/versioned URLs; use modern formats (WebP/AVIF) with content negotiation where supported. Store originals in cool tier when old, thumbnails hot. Control cost: egress and CDN hit ratio, transformation compute. Risk: unbounded variant explosion and CPU-heavy transformations in the web tier; isolate processing in queue-driven workers with autoscale.
**Follow-up traps**: "How to purge a changed image?" (new URL via version/hash; avoid purges.)

**Red flags in answers**
- Using SQL LIKE or storing large files in SQL for scale.
- Proxying big uploads through the app server.
- No sync/rebuild plan for the search index; no SAS expiry/scoping.

**Cheat sheet**
- Search index = derived store; sync and rebuild.
- Autocomplete: debounce, cache prefixes, small payloads, skewed queries.
- Large files: direct-to-Blob via SAS, event-driven processing.
- Bytes in Blob, metadata in SQL, reconcile orphans.
- CDN + immutable versioned URLs for media.
- Scan uploads; never trust client content-type.

---

## Topic 16: Scenario round

**Study notes**
- Scenario answer skeleton: (1) clarify goals, constraints, numbers; (2) state assumptions; (3) diagnose/measure; (4) options with trade-offs; (5) decide with phases (quick wins -> structural); (6) risks, rollback, and how to verify.
- Always separate "stop the bleeding" from "fix the cause".
- Giving estimates in interviews: state assumptions explicitly, show arithmetic, round, give ranges, identify sensitivities, name unknowns, say how you'd validate. Never pretend false precision; never refuse to estimate.

### 16.1 [Scenario] Traffic is expected to grow 10x in six months. What do you do?
**What the interviewer is really testing**: Proactive scalability planning.
**Typical mid-level answer**: "Scale up servers and add caching."
**Strong senior answer**: First get facts: current load profile (peak req/s, DB CPU/IO, p95), headroom, and what 10x means (users, data size, or both; peak shape). Then find the first thing that breaks via a load test at 2x, 5x, 10x of a production-like environment, and USE-method review of each tier. Expect the order: DB (connections, hot queries, locks), then downstream limits, then app. Build a roadmap: quick wins (index/query fixes, pagination, caching, async I/O, autoscale rules), mid-term (read replicas, queue levelling for writes, CDN), and structural (partitioning/sharding or a separate analytics store) only where tests prove need. Add observability and SLOs, capacity forecast with growth curve, and cost projection (10x traffic should not be 10x cost). Re-test after each step and keep a rollback plan. Risk: designing for 100x prematurely, or ignoring data growth, which is often the real killer.
**Follow-up traps**: "What if the DB hits its ceiling at 4x?" (scale up tier now, buy time; plan read models/sharding with the key chosen from real access patterns.)

### 16.2 [Scenario] Black Friday flash sale: 50x spike for 30 minutes on a limited-stock item.
**What the interviewer is really testing**: Spike handling, contention and fairness.
**Typical mid-level answer**: "Autoscale and add cache."
**Strong senior answer**: The hard parts are the spike speed (autoscale too slow) and contention on one inventory row. Pre-scale and warm (scheduled scale, min instances, pre-warmed caches, higher DB tier for the window), serve all static/catalogue via CDN, and put a waiting-room/virtual queue (Front Door/WAF rate limit or a queue-based admission token) in front of checkout to cap concurrent buyers at what the backend can handle. For stock: avoid per-request row contention by using atomic conditional update (`UPDATE ... SET Stock = Stock - @q WHERE Stock >= @q`), or reserve inventory in pre-allocated chunks/in Redis with periodic reconciliation, with a time-limited reservation model. Orders are accepted into a queue and processed asynchronously with idempotency; payments handled with provider limits in mind. Load test the exact scenario including breakpoints and failure injection; define a kill-switch and degrade plan (disable recommendations, reviews). Risk: overselling vs underselling policy, thundering-herd retries from clients; use jittered backoff and clear UX for waiting.
**Follow-up traps**: "Redis stock counter lost?" (reconcile with DB as source of truth; hold reservations with expiry.)

### 16.3 [Scenario] A finance report query is killing the OLTP database every morning at 9.
**What the interviewer is really testing**: Triage and long-term separation.
**Typical mid-level answer**: "Optimise the query."
**Strong senior answer**: Immediate: confirm via Query Store/wait stats it is the report (and what resource it consumes), reschedule or throttle it, give it a timeout and run it on a readable replica. Check if a quick plan/index fix drops cost drastically, but treat that as short-term. Medium: pre-aggregate overnight into summary tables and serve the report from them. Long-term: move analytical workloads off OLTP via CDC/Synapse Link to a warehouse with columnstore and Power BI, agreeing data freshness with finance (they often need "as of 8am", not live). Set guardrails (Resource Governor/elastic pool caps, report-user limits). Communicate with the finance team as stakeholders. Risk: replica lag affecting correctness of "as of" reports; define consistent snapshot time.
**Follow-up traps**: "Finance wants it real-time?" (clarify; if truly needed, nonclustered columnstore on replica or streaming, with cost trade-off.)

### 16.4 [Scenario] Choose a database for a new product with unclear access patterns.
**What the interviewer is really testing**: Decisions under uncertainty.
**Typical mid-level answer**: "Cosmos or SQL, based on preference."
**Strong senior answer**: I'd spend an hour on the top use cases, expected volumes (even rough), consistency needs, and team skills. With unclear patterns, a relational database (Azure SQL) maximises flexibility: ad-hoc queries, joins, constraints, strong tooling, and scale that's enough for most early products. I'd keep the data access behind clean module boundaries, use JSON columns for volatile attributes, emit domain events via outbox so a search/cache/Cosmos store can be derived later, and define review triggers (data size, p95, cost). I'd avoid committing to a partition key I can't justify. Revisit after 3-6 months of real access data. Risk: later migration; limited by keeping boundaries and the outbox in place.
**Follow-up traps**: "Founder insists on Cosmos for 'infinite scale'?" (ask for the access patterns and RU estimate; scale isn't the risk at this stage, query flexibility is.)

### 16.5 [Scenario] Reduce the Azure bill 30% without hurting reliability.
**What the interviewer is really testing**: Structured FinOps with risk control.
**Typical mid-level answer**: "Shrink SKUs."
**Strong senior answer**: Get the cost breakdown by service and tag, set a baseline, and rank opportunities by saving versus risk. Phase 1 (low risk): delete orphaned resources, schedule non-prod shutdowns, cut log retention and ingestion noise (sampling, levels), storage lifecycle tiering, Hybrid Benefit. Phase 2: reservations/savings plans for the steady baseline. Phase 3: right-size using utilisation over a full business cycle with headroom for failover and peaks; autoscale-down rules. Phase 4: architectural savings (cache to reduce DB tier, Cosmos RU autoscale/indexing policy, batch on spot, reduce cross-region egress). Gate each wave on SLO dashboards with rollback. Track the result as cost per unit (request/tenant) rather than just totals so growth doesn't hide savings. Risk: removing redundancy or headroom; I would explicitly protect the capacity the SLO depends on.
**Follow-up traps**: "Which saves the most fastest?" (usually commitments and non-prod/log waste.)

### 16.6 [Scenario] p99 latency spikes only at peak hours; p50 is fine. Diagnose.
**What the interviewer is really testing**: Tail-latency reasoning under load.
**Typical mid-level answer**: "Add more instances."
**Strong senior answer**: Tail spikes at load suggest queueing and contention rather than steady slowness. I'd slice slow requests (App Insights) by endpoint, tenant, instance and dependency to see if the slowness is in one dependency. Hypotheses: thread-pool starvation (queue length, sync-over-async), connection pool waits, lock/blocking or deadlock retries, parameter-sniffing plans, GC pauses (Gen2/LOH), cold autoscaled instances (JIT, cache fill), SNAT exhaustion, throttling by downstream (429/RU limits), or batch jobs overlapping peak. Use `dotnet-counters`, SQL wait stats during the spike, and Query Store regressed plans. Fix per cause: async fixes, pool sizing, indexing/RCSI, plan stabilisation, pre-warm/min instances, throttling/backoff, move batch off-peak. Verify with a load test reproducing the peak, and monitor p99 as an SLO. Risk: adding instances increases DB pressure if the DB is the bottleneck.
**Follow-up traps**: "How do you catch it live?" (alert at p99 breach, capture dumps/traces automatically via profiler triggers, Extended Events.)

### 16.7 [Medium] Give an estimate in an interview: "How much storage and bandwidth does a photo-sharing app with 1M daily uploads need?"
**What the interviewer is really testing**: Estimation technique and communication.
**Typical mid-level answer**: Blurts a number.
**Strong senior answer**: "Assuming 1M uploads/day, average 3 MB after client-side compression, and we keep an original plus ~3 thumbnails totalling ~0.5 MB: about 3.5 MB per photo, ~3.5 TB/day, ~1.3 PB/year before replication. With 3x redundancy (LRS/ZRS internal, GRS adds another copy) cost and capacity scale accordingly, so I'd use tiering: hot for 30 days, cool/archive after. Upload bandwidth is 3 TB/day / 86 400 s ~ 35 MB/s average (~280 Mbit/s), with a peak factor of 3-5 so ~1 Gbit/s. Reads are probably 20-50x uploads, mostly via CDN, so egress and CDN hit ratio drive the bill. The sensitive assumptions are average size and read ratio; I'd validate with real samples." Then I'd explicitly point out the decisions this implies: direct-to-Blob upload, CDN, lifecycle rules, and async thumbnailing. Risk: wrong peak factor; give a range.
**Follow-up traps**: "What if average is 8 MB (modern phone)?" (scale linearly: it's about 2.3x; the architecture is unchanged, the cost changes.)

**Red flags in answers**
- Jumping to technology before asking what is measured and what the goal is.
- No separation between mitigation now and root-cause fix later.
- Unqualified numbers with no assumptions.

**Cheat sheet**
- Skeleton: clarify -> assume -> measure -> options -> phased decision -> risks.
- 10x growth: test to find the first break; data growth matters too.
- Flash sale: pre-scale, waiting room, atomic stock update, async orders.
- Report vs OLTP: stop bleeding, replica, pre-aggregate, warehouse.
- p99 at peak: slice, check queues/locks/GC/cold starts/throttling.
- Estimates: assumptions, arithmetic, range, sensitivity, validation.
