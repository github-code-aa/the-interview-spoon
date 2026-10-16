# Z11 - Data, Messaging & Neighbouring Azure Services (Interview Guide)

**How to use:** Read each topic's Study notes first, then answer the ladder questions aloud (Easy -> Tricky) before reading the Answer. Code is illustrative; limits/pricing/SDK versions change - verify in current docs.
**Scope:** Services that sit next to Service Bus/Functions/Logic Apps/Blob/Key Vault in a typical .NET solution.

## Table of Contents
1. Azure SQL Database from .NET
2. Cosmos DB essentials
3. Event Grid
4. Event Hubs
5. Azure App Configuration
6. API Management in depth
7. Data Factory / Synapse overview
8. Container Apps, Jobs & AKS basics
9. Azure Cache, CDN & Front Door caching
10. Communication Services / SendGrid email
11. Azure SignalR Service
12. Azure OpenAI / AI Search integration points
13. Messaging pattern decision matrix (10 scenarios)
14. Data consistency patterns across services
15. Backup / DR across the stack

---

## Topic 1: Azure SQL Database from .NET

**Study notes**

```
App (SAMI/UAMI) --token--> Entra ID --> Azure SQL (contained user FROM EXTERNAL PROVIDER)
Purchase models:  DTU (bundled, simple)  |  vCore (CPU/mem/storage separate, serverless, Hyperscale, reserved capacity)
HA/DR:            zone redundancy | auto-failover groups (listener endpoints) | active geo-replication (readable secondary)
Recovery:         automated backups, PITR (default 7 days, configurable up to 35), LTR (up to 10 years), geo-restore
```

| Concept | Key point |
|---|---|
| Auth | Entra auth via Microsoft.Data.SqlClient; no password in config |
| Transient errors | 40613 (db unavailable), 40501 (throttled), 10928/10929 (resource limit), 49918/49919/49920 (busy) |
| Serverless | vCore only; auto-pause after idle delay; first connection after pause fails/slow (resume ~ tens of seconds) |
| Elastic pool | Share resources across many DBs with uneven peaks |
| Hyperscale | Up to ~100 TB, fast backups/restore, named replicas, independent scale of compute/storage (verify limits) |

### 1.1 [Easy] How does a .NET app connect to Azure SQL without a password?
**Answer:** Use a managed identity with an Entra token. In Microsoft.Data.SqlClient set `Authentication=Active Directory Default` (or `Active Directory Managed Identity`) in the connection string, and create a contained database user for the identity.
**Explanation:** The identity (system- or user-assigned) acquires a token for `https://database.windows.net/` and SqlClient presents it at login. `Active Directory Default` uses the DefaultAzureCredential-like chain (env, VS, CLI, managed identity) so the same string works locally and in Azure. `Active Directory Managed Identity` is explicit; for UAMI add `User Id=<client-id>`. The SQL server must have an Entra admin configured to create external users.
**Code / CLI / Bicep**
```csharp
// Azure: UAMI; local: dev's az login
var cs = "Server=tcp:myserver.database.windows.net,1433;Database=orders;" +
         "Authentication=Active Directory Default;Encrypt=True;";
await using var conn = new SqlConnection(cs);
await conn.OpenAsync();
```
```sql
-- run as Entra admin, connected to the target DB (not master)
CREATE USER [my-func-app] FROM EXTERNAL PROVIDER;
ALTER ROLE db_datareader ADD MEMBER [my-func-app];
ALTER ROLE db_datawriter ADD MEMBER [my-func-app];
```

### 1.2 [Follow-up] What are transient errors and how do you handle them with EF Core?
**Answer:** Azure SQL can throw transient faults during failover, scaling, or throttling (40613, 40501, 10928, etc.). EF Core's `EnableRetryOnFailure` installs a `SqlServerRetryingExecutionStrategy` that retries with exponential backoff.
**Explanation:** The default strategy already knows the transient error list; you can add extra codes. Caveat: with a retrying strategy, user-initiated transactions must be wrapped in `strategy.ExecuteAsync` so the whole unit retries. Also, retries can double-execute if the commit succeeded but the ack was lost - design idempotent writes (see Topic 14).
**Code / CLI / Bicep**
```csharp
services.AddDbContext<OrdersDb>(o => o.UseSqlServer(cs, sql =>
    sql.EnableRetryOnFailure(maxRetryCount: 5,
        maxRetryDelay: TimeSpan.FromSeconds(30),
        errorNumbersToAdd: null)));

var strategy = db.Database.CreateExecutionStrategy();
await strategy.ExecuteAsync(async () =>
{
    await using var tx = await db.Database.BeginTransactionAsync();
    // work
    await db.SaveChangesAsync();
    await tx.CommitAsync();
});
```

### 1.3 [Medium] DTU vs vCore, and when would you choose serverless or an elastic pool?
**Answer:** DTU is a bundled blend of CPU/IO/memory - simple, good for small predictable workloads. vCore gives explicit sizing, serverless, Hyperscale, and reservations. Serverless suits intermittent dev/test workloads; elastic pools suit many databases (e.g. multi-tenant) with non-overlapping peaks.
**Explanation:** Serverless bills per second for compute used between min/max vCores and pauses (compute billing stops, storage continues) after an auto-pause delay. Auto-pause is a poor fit for latency-sensitive prod because the first connection after pause fails or is slow - retry logic is mandatory. Elastic pool eDTUs/vCores are shared; one noisy DB can consume pool resources unless per-DB min/max set.
**Code / CLI / Bicep**
```bash
az sql db create -g rg -s myserver -n orders \
  --edition GeneralPurpose --compute-model Serverless --family Gen5 \
  --min-capacity 0.5 --capacity 2 --auto-pause-delay 60
az sql elastic-pool create -g rg -s myserver -n pool1 --edition GeneralPurpose --family Gen5 --capacity 4
```

### 1.4 [Medium] Explain connection pooling and common pool problems.
**Answer:** SqlClient pools physical connections per unique connection string (and identity/credential). `Open` borrows, `Dispose`/`Close` returns. Leaks cause "timeout expired ... max pool size reached".
**Explanation:** Default Max Pool Size is 100. Always `await using` connections; don't hold them across slow external calls. Different connection strings create separate pools, so avoid per-request string variations. In Functions, many instances x pool size can exceed the DB's worker/connection limit (error 10928) - cap Max Pool Size and function concurrency. After a failover, pooled dead connections are discarded on use; `ConnectRetryCount` (idle connection resiliency) helps.
**Code / CLI / Bicep**
```csharp
var cs = "Server=...;Database=orders;Authentication=Active Directory Default;" +
         "Max Pool Size=50;Connect Timeout=30;ConnectRetryCount=3;ConnectRetryInterval=10;";
```

### 1.5 [Hard] Compare failover groups and active geo-replication, and describe backups/PITR.
**Answer:** Both use async replication to a secondary region. Failover groups add read-write/read-only listener endpoints and group multiple DBs for coordinated failover, so apps need no connection string change. Geo-replication is per-database with no listener. Backups: automated full/diff/log, PITR within retention, LTR for compliance; restores create a new database.
**Explanation:** Because replication is asynchronous, RPO is small but non-zero (seconds); forced failover can lose recent commits. Geo-replication/failover groups cost a secondary DB. Restore never overwrites in place - you restore to a new name and swap. Backup storage redundancy (LRS/ZRS/GRS) is chosen at DB creation (verify options in current docs). Hyperscale uses snapshot-based backups with near-constant-time restore.
**Code / CLI / Bicep**
```bash
az sql failover-group create -n fog1 -g rg -s primarysrv --partner-server secondarysrv \
  --partner-resource-group rg2 --add-db orders --failover-policy Automatic --grace-period 1
az sql db restore -g rg -s myserver -n orders --dest-name orders-restored \
  --time "2026-10-15T10:00:00Z"
```
**Gotchas / follow-ups**
- Use the `<fog>.database.windows.net` listener name, not server names, in connection strings.
- Automatic failover grace period trades RPO for availability.
- Entra admin and logins/users: contained users replicate with the DB; server-level logins don't.

### 1.6 [Tricky] Your Function's SQL calls fail with 40613 right after a scale operation, and sometimes with "Login failed for user '<token-identified principal>'". Diagnose.
**Answer:** 40613 is transient: retry. The login failure usually means the managed identity has no contained user in *that* database (e.g. restored/copy DB, or user created in master), or the wrong identity (SAMI vs UAMI) is used.
**Explanation:** Restored or copied databases keep contained users, but a new DB created from a script may not. With UAMI and multiple identities on the host, `Active Directory Default` may pick a different one - specify `User Id=<UAMI clientId>` or `Active Directory Managed Identity`. Also check firewall/private endpoint and that the server Entra admin exists. Token caching can briefly delay role changes.
**Code / CLI / Bicep**
```csharp
var cs = $"Server=tcp:srv.database.windows.net;Database=orders;" +
         $"Authentication=Active Directory Managed Identity;User Id={uamiClientId};Encrypt=True;";
```
```sql
SELECT name, type_desc, authentication_type_desc FROM sys.database_principals WHERE name = 'my-func-app';
```
**Gotchas / follow-ups**
- `CREATE USER ... FROM EXTERNAL PROVIDER` must be run by an Entra principal, not a SQL login.
- Granting db roles to the identity's *name* - names must be unique; UAMI display name is what you use.

**Interview traps**
- "Managed identity works out of the box" - no, a contained DB user and role grants are required.
- Retrying a non-idempotent insert after a lost commit ack duplicates data.
- Serverless auto-pause is not for latency-critical prod.
- Geo-replication RPO is not zero.

**Cheat sheet**
- `Authentication=Active Directory Default|Managed Identity`; UAMI -> `User Id=clientId`.
- Transient: 40613, 40501, 10928 -> `EnableRetryOnFailure` + `CreateExecutionStrategy` for transactions.
- Pool: default max 100; dispose connections.
- FoG = listener endpoints + group; geo-rep = per-DB.
- PITR restore creates a new DB.

---

## Topic 2: Cosmos DB essentials

**Study notes**

```
Account -> Database -> Container (partition key /pk) -> Items
Logical partition (same pk value, max 20 GB) -> mapped to physical partitions (RU/s divided among them)
Throughput: Provisioned (manual/autoscale) | Serverless
Consistency: Strong > Bounded Staleness > Session (default) > Consistent Prefix > Eventual
```

| Operation | Approx cost (1 KB item) |
|---|---|
| Point read (id + pk) | ~1 RU |
| Write | ~5-10 RU (depends on indexing) |
| Cross-partition query | Much higher; fan-out |

### 2.1 [Easy] What is a partition key and why does it matter?
**Answer:** It's the property used to distribute items into logical partitions. A good key has high cardinality and even request/storage distribution, and appears in most queries' filters.
**Explanation:** RU/s is spread across physical partitions; a hot key causes 429s while other partitions idle. A logical partition is capped at 20 GB. Partition key is immutable per item; changing it means migrating data. Hierarchical partition keys (up to 3 levels) can mitigate hot/large keys (verify availability in current docs). Typical choices: `/tenantId`, `/userId`, synthetic keys.
**Code / CLI / Bicep**
```csharp
var container = (await db.CreateContainerIfNotExistsAsync(
    new ContainerProperties("orders", "/customerId"), ThroughputProperties.CreateAutoscaleThroughput(4000))).Container;
```

### 2.2 [Follow-up] What are RU/s, and how do you handle 429 errors?
**Answer:** Request Units are the normalized cost of operations. Exceeding provisioned RU/s returns HTTP 429 with `x-ms-retry-after-ms`; the SDK retries automatically (default 9 retries / 30s wait) before throwing.
**Explanation:** Tune via `CosmosClientOptions.MaxRetryAttemptsOnRateLimitedRequests` and `MaxRetryWaitTimeOnRateLimitedRequests`. Fix the cause: improve partition key, add indexing policy exclusions, use point reads, raise autoscale max. Inspect `response.RequestCharge`. Autoscale scales between 10% and 100% of max RU/s.
**Code / CLI / Bicep**
```csharp
try { var r = await container.CreateItemAsync(item, new PartitionKey(item.CustomerId)); Console.WriteLine(r.RequestCharge); }
catch (CosmosException ex) when (ex.StatusCode == (HttpStatusCode)429)
{ await Task.Delay(ex.RetryAfter ?? TimeSpan.FromSeconds(1)); }
```

### 2.3 [Medium] Explain consistency levels and the SDK singleton rule.
**Answer:** Five levels trade latency/availability vs staleness; Session (default) guarantees read-your-writes within a session token. `CosmosClient` must be a singleton per app because it manages connections, caches account/partition metadata, and is thread-safe.
**Explanation:** Strong costs 2x read RU and isn't allowed with multi-region writes. A client may relax (not strengthen) the account default. Session token is per client instance - across instances you must pass it explicitly. Creating a client per request causes socket exhaustion and cold metadata calls. Use Direct mode (default in v3) for latency; Gateway if ports are restricted.
**Code / CLI / Bicep**
```csharp
services.AddSingleton(_ => new CosmosClient(endpoint, new DefaultAzureCredential(),
    new CosmosClientOptions { ConsistencyLevel = ConsistencyLevel.Session,
        SerializerOptions = new() { PropertyNamingPolicy = CosmosPropertyNamingPolicy.CamelCase } }));
```

### 2.4 [Medium] Point reads vs queries; TTL; change feed.
**Answer:** A point read (`ReadItemAsync` with id + partition key) is the cheapest and fastest; queries cost more, especially cross-partition. TTL auto-expires items; the change feed is an ordered log of inserts/updates per partition key range.
**Explanation:** TTL must be enabled on the container (`DefaultTimeToLive`), item-level `ttl` overrides; deletions use leftover RU. Change feed (latest-version mode) doesn't emit deletes - use soft-delete flag; all-versions-and-deletes mode exists (verify status). Consume via change feed processor (leases container) or Functions Cosmos trigger.
**Code / CLI / Bicep**
```csharp
var item = await container.ReadItemAsync<Order>(id, new PartitionKey(customerId));
var cfp = container.GetChangeFeedProcessorBuilder<Order>("orders-proc",
        async (changes, ct) => { foreach (var o in changes) { /* handle */ } })
    .WithInstanceName(Environment.MachineName)
    .WithLeaseContainer(leases).Build();
await cfp.StartAsync();
```

### 2.5 [Hard] Cosmos vs Azure SQL - how do you decide?
**Answer:** Choose Cosmos for globally distributed, low-latency, schema-flexible, single-partition-access workloads at massive scale. Choose SQL for relational integrity, complex joins/aggregations, and multi-entity ACID transactions.
**Explanation:** Cosmos transactions are limited to one logical partition (transactional batch / stored procs). Cross-container joins don't exist; model by denormalizing/embedding. Cost model is RU-driven and can surprise on cross-partition queries. SQL is simpler for reporting. Hybrid is common: SQL as system of record, Cosmos for read models fed via change feed.
**Code / CLI / Bicep**
```csharp
var batch = container.CreateTransactionalBatch(new PartitionKey(customerId))
    .CreateItem(order).UpsertItem(summary);
using var resp = await batch.ExecuteAsync(); // atomic, same partition only
```
**Gotchas / follow-ups**
- Cross-partition queries cost RU per partition touched.
- Large items/arrays inflate RU; indexing policy tuning reduces write RU.

### 2.6 [Tricky] Intermittent 429s while overall RU consumption is only 30% of provisioned. Why?
**Answer:** A hot partition: RU/s is divided evenly across physical partitions, so one hot physical partition can be throttled while the container-level metric looks low.
**Explanation:** Check Metrics split by PartitionKeyRangeId and "Normalized RU Consumption" max. Causes: low-cardinality key (e.g., date, status), burst on one tenant. Fixes: better/synthetic/hierarchical key, spread writes, or rely on newer features like throughput redistribution across partitions (verify in current docs).
**Code / CLI / Bicep**
```bash
az monitor metrics list --resource <cosmos-id> --metric NormalizedRUConsumption \
  --dimension PartitionKeyRangeId --aggregation Maximum
```
**Gotchas / follow-ups**
- Autoscale is also per-partition share of max RU/s.

**Interview traps**
- "Singleton" applies to `CosmosClient`, not Container/Database objects (those are cheap).
- Change feed doesn't show deletes (latest-version mode).
- Session consistency doesn't hold across different client instances without token passing.
- Changing partition key requires data migration.

**Cheat sheet**
- Point read ~1 RU; always prefer it.
- 20 GB logical partition cap; pick high-cardinality key.
- Default consistency: Session.
- 429 -> SDK retries; fix design.
- Transactions only within a partition.

---

## Topic 3: Event Grid

**Study notes**

```
Publisher/Source --> [System topic | Custom topic | Domain | Namespace topic] --> Event subscription(filter) --> Handler
                                                                           (Function, Service Bus, Event Hubs, Storage Queue, webhook)
Push delivery, at-least-once; retry w/ backoff (up to 24h / 30 attempts by default); dead-letter to Blob
```

| Item | Note |
|---|---|
| System topic | Events from Azure services (Blob created, resource changes) |
| Custom topic | Your own events (EventGridEvent or CloudEvents v1.0 schema) |
| Filters | Event type, subject prefix/suffix, advanced (data fields) |
| Ordering | Not guaranteed |

### 3.1 [Easy] What is Event Grid and what are system vs custom topics?
**Answer:** A fully managed pub/sub event router that pushes discrete event notifications to subscribers. System topics expose events from Azure services; custom topics let your apps publish their own events.
**Explanation:** Events say "something happened" (reactive), usually small with a reference to the data, not the data itself. Subscribers are decoupled; multiple subscriptions each get a copy. Schemas: Event Grid schema, CloudEvents 1.0 (preferred for interoperability), or custom input mapping.
**Code / CLI / Bicep**
```bash
az eventgrid system-topic create -g rg -n st-storage --topic-type microsoft.storage.storageaccounts \
  --source <storage-id> -l global
az eventgrid system-topic event-subscription create -g rg --system-topic-name st-storage -n sub1 \
  --endpoint-type azurefunction --endpoint <function-resource-id> \
  --included-event-types Microsoft.Storage.BlobCreated --subject-begins-with /blobServices/default/containers/in/
```

### 3.2 [Follow-up] How do retry and dead-lettering work?
**Answer:** If the endpoint doesn't return success (2xx), Event Grid retries with exponential backoff until the event's TTL (default 24h) or max attempts (default 30). Undeliverable events can go to a dead-letter Blob container (must be configured).
**Explanation:** Some responses (400/413) are considered non-retryable and are dead-lettered immediately. Dead-lettering is off by default - events are dropped if not configured. Handlers must be idempotent since delivery is at-least-once. Dead-letter destination can use managed identity.
**Code / CLI / Bicep**
```bicep
resource sub 'Microsoft.EventGrid/systemTopics/eventSubscriptions@2022-06-15' = {
  parent: topic
  name: 'sub1'
  properties: {
    destination: { endpointType: 'ServiceBusQueue', properties: { resourceId: queueId } }
    retryPolicy: { maxDeliveryAttempts: 10, eventTimeToLiveInMinutes: 120 }
    deadLetterDestination: { endpointType: 'StorageBlob', properties: { resourceId: storageId, blobContainerName: 'deadletter' } }
  }
}
```

### 3.3 [Medium] Explain the webhook validation handshake.
**Answer:** Before delivering to a webhook, Event Grid sends a `SubscriptionValidationEvent` containing a validation code; the endpoint must echo it back as `{ "validationResponse": "<code>" }` (or complete the handshake via `validationUrl`).
**Explanation:** This proves endpoint ownership. Azure Function Event Grid triggers and Logic Apps handle it automatically; hand-written ASP.NET endpoints must implement it. CloudEvents v1.0 uses an HTTP OPTIONS request with `WebHook-Request-Origin` and expects `WebHook-Allowed-Origin`.
**Code / CLI / Bicep**
```csharp
app.MapPost("/events", async (HttpRequest req) =>
{
    var events = EventGridEvent.ParseMany(await BinaryData.FromStreamAsync(req.Body));
    foreach (var e in events)
    {
        if (e.TryGetSystemEventData(out var data) && data is SubscriptionValidationEventData v)
            return Results.Ok(new { validationResponse = v.ValidationCode });
        // handle event
    }
    return Results.Ok();
});
```

### 3.4 [Medium] Event Grid vs Service Bus - when which?
**Answer:** Event Grid: reactive, lightweight notifications with fan-out and filtering, push delivery, high scale. Service Bus: commands/transactions needing reliable processing, sessions, ordering (FIFO via sessions), DLQ, peek-lock, deferral, scheduled delivery.
**Explanation:** Event Grid has no consumer-controlled pull, no sessions, no guaranteed order. A common pattern is Event Grid -> Service Bus queue to get durable buffering and consumer pacing. Event Grid payload limit is 1 MB (verify), and publishing batches are bounded.
**Code / CLI / Bicep**
```csharp
var client = new EventGridPublisherClient(new Uri(topicEndpoint), new DefaultAzureCredential());
await client.SendEventAsync(new CloudEvent("/orders", "Order.Created", new { Id = 42 }));
```

### 3.5 [Hard] How do you secure an Event Grid -> Function/Service Bus path and filter efficiently?
**Answer:** Use managed identity on the topic/subscription to deliver to Service Bus or Event Hubs (assign "Azure Service Bus Data Sender"), and Entra-protected webhooks for HTTP targets. Filter at the subscription (event type, subject, advanced filters) to avoid paying for and processing irrelevant events.
**Explanation:** Delivery with identity requires enabling system-assigned/user-assigned identity on the topic and granting RBAC to the destination. Advanced filters support operators such as StringIn, NumberGreaterThan with a limited number of filters per subscription (verify). Subject filters are cheapest and most common (e.g., blob path prefix).
**Code / CLI / Bicep**
```bash
az eventgrid system-topic event-subscription create -g rg --system-topic-name st-storage -n toSb \
  --endpoint-type servicebusqueue --endpoint <queue-id> --delivery-identity-endpoint-type servicebusqueue \
  --delivery-identity systemassigned --delivery-identity-endpoint <queue-id> \
  --advanced-filter data.api StringIn PutBlob PutBlockList
```
**Gotchas / follow-ups**
- Grant the role before creating the subscription or delivery fails.
- Private endpoints / firewalls: Event Grid must be allowed as trusted service (verify per target).

### 3.6 [Tricky] Duplicate and out-of-order BlobCreated events cause double processing. Fix?
**Answer:** Make the handler idempotent (key on blob URL + ETag or event id) and don't rely on order. Check blob state at processing time.
**Explanation:** At-least-once delivery plus retries yields duplicates; large uploads can emit `BlobCreated` for `PutBlockList` only once, but `CopyBlob`/overwrite emits again. Use event `id` as a dedup key in a store (Redis/SQL unique index) or use blob leases/ETag conditions.
**Code / CLI / Bicep**
```csharp
[Function("OnBlob")]
public async Task Run([EventGridTrigger] CloudEvent e)
{
    if (!await _dedup.TryRegisterAsync(e.Id)) return; // unique key insert
    // process
}
```
**Gotchas / follow-ups**
- Event Grid retry may deliver after your function already succeeded but timed out.

**Interview traps**
- Event Grid is push, not pull; no ordering guarantees.
- Dead-lettering is opt-in.
- Webhook handshake is automatic only for Functions/Logic Apps.
- Don't use it for high-volume telemetry streams (that's Event Hubs).

**Cheat sheet**
- Event = notification; message = command.
- Default retry: 24h/30 attempts.
- Prefer CloudEvents v1.0.
- Event Grid -> Service Bus queue for durable buffering.
- Filter by type/subject/data.

---

## Topic 4: Event Hubs

**Study notes**

```
Producers --> Event Hub (partitions 0..N-1) --> Consumer group A (EventProcessorClient) ...
                                             --> Consumer group B (analytics)
Checkpoints stored in Blob container; Capture writes Avro to Blob/ADLS
Tiers: Basic | Standard | Premium | Dedicated ; throughput units (TU) / processing units (PU) (verify)
```

| Concept | Note |
|---|---|
| Partition | Ordered log; ordering only within partition; count fixed at creation on Basic/Standard (Premium/Dedicated can increase) |
| Consumer group | Independent view of the stream |
| Retention | 1-7 days Standard (longer on Premium/Dedicated) - verify |
| Kafka | Kafka protocol endpoint on Standard+ |

### 4.1 [Easy] What is Event Hubs and what are partitions and consumer groups?
**Answer:** A high-throughput streaming ingestion service (telemetry/logs). Partitions are ordered shards that enable parallelism; consumer groups allow multiple independent readers of the same stream.
**Explanation:** Events with the same partition key hash to the same partition, preserving order for that key. Events are not deleted when read; they expire by retention. Each consumer group tracks its own offsets. Max one reader per partition per consumer group is the useful parallelism limit.
**Code / CLI / Bicep**
```csharp
await using var producer = new EventHubProducerClient(ns, hub, new DefaultAzureCredential());
using var batch = await producer.CreateBatchAsync(new CreateBatchOptions { PartitionKey = deviceId });
batch.TryAdd(new EventData(BinaryData.FromString("{\"t\":21}")));
await producer.SendAsync(batch);
```

### 4.2 [Follow-up] How does EventProcessorClient and checkpointing work?
**Answer:** `EventProcessorClient` balances partition ownership among instances, using a Blob container for ownership and checkpoints. You call `UpdateCheckpointAsync` after processing so a restart resumes from there.
**Explanation:** Delivery is at-least-once: events after the last checkpoint are re-read on restart, so handlers must be idempotent. Checkpoint every N events rather than each event (blob write cost/latency). Needs Storage Blob Data Contributor for the identity. Functions' Event Hubs trigger does this for you.
**Code / CLI / Bicep**
```csharp
var blob = new BlobContainerClient(new Uri(containerUri), new DefaultAzureCredential());
var proc = new EventProcessorClient(blob, EventHubConsumerClient.DefaultConsumerGroupName, ns, hub, new DefaultAzureCredential());
int n = 0;
proc.ProcessEventAsync += async a =>
{
    // handle a.Data
    if (++n % 100 == 0) await a.UpdateCheckpointAsync();
};
proc.ProcessErrorAsync += a => Task.CompletedTask;
await proc.StartProcessingAsync();
```

### 4.3 [Medium] Throughput units, Capture, and Kafka endpoint.
**Answer:** A TU provides ~1 MB/s or 1000 events/s ingress and 2 MB/s egress (verify current figures); auto-inflate can scale TUs up. Capture automatically writes stream data to Blob/ADLS in Avro. The Kafka endpoint lets Kafka clients produce/consume with config changes only.
**Explanation:** Exceeding TUs causes `ServerBusy` throttling. Capture is time/size-windowed and independent of consumers. Kafka mapping: topic = event hub, group.id = consumer group; SASL PLAIN or OAUTHBEARER with Entra.
**Code / CLI / Bicep**
```bicep
resource ns 'Microsoft.EventHub/namespaces@2022-10-01-preview' = {
  name: 'ehns1'; location: resourceGroup().location
  sku: { name: 'Standard', tier: 'Standard', capacity: 2 }
  properties: { isAutoInflateEnabled: true, maximumThroughputUnits: 10 }
}
```

### 4.4 [Medium] Event Hubs vs Service Bus vs Event Grid?
**Answer:** Event Hubs: high-volume ordered-per-partition streams, replayable, consumers pull. Service Bus: transactional messaging with per-message settlement, DLQ, sessions. Event Grid: discrete event notification routing with push.
**Explanation:** Event Hubs has no per-message ack/DLQ/delay - consumers track an offset. Poison events must be handled in your code (skip/park to another store). Replay by resetting checkpoint within retention.

### 4.5 [Hard] How do you size partitions and handle a poison event or lag?
**Answer:** Choose partitions for max expected consumer parallelism (hard to change on Standard). For poison events, catch exceptions, route the event to a dead-letter store, and still checkpoint; monitor lag via incoming vs checkpoint sequence number.
**Explanation:** Throwing in the processor handler doesn't stop other partitions but doesn't advance the checkpoint; with Functions' trigger a failing batch is retried per retry policy and can block the partition. Hot keys cause a hot partition. Scale consumers up to partition count only.
**Code / CLI / Bicep**
```csharp
proc.ProcessEventAsync += async a =>
{
    try { await Handle(a.Data); }
    catch (Exception ex) { await _deadLetter.SaveAsync(a.Data, ex); }
    await a.UpdateCheckpointAsync();
};
```
**Gotchas / follow-ups**
- Lag metric: "Captured/Incoming" vs consumer sequence number; verify available metrics.
- Partition count cannot be reduced.

### 4.6 [Tricky] After redeploying, a consumer reprocesses hours of events. Why?
**Answer:** Checkpoints were not written (or use a different blob container/consumer group), so the processor started from the configured starting position (earliest/latest) instead of the checkpoint.
**Explanation:** Checks: same storage container + consumer group + namespace/hub names; identity RBAC on blob; checkpoint frequency; Functions' host storage (`AzureWebJobsStorage`) change creates new checkpoints. Mitigate with idempotent handling.
**Code / CLI / Bicep**
```bash
az storage blob list --account-name <sa> -c checkpoints --prefix "<ns>.servicebus.windows.net/<hub>/<group>/checkpoint" --auth-mode login
```

**Interview traps**
- Ordering is per partition only.
- No dead-letter queue in Event Hubs.
- Partition count on Standard cannot be changed after creation.
- Consumers > partitions are idle.

**Cheat sheet**
- Partition key -> same partition -> ordered.
- Checkpoint in Blob; at-least-once.
- Capture -> Avro in Blob/ADLS.
- Kafka endpoint on Standard+.
- Replay = reset checkpoint within retention.

---

## Topic 5: Azure App Configuration

**Study notes**

```
App --(Entra/MI or conn string)--> App Configuration store
  key-values (key + label + content type) | feature flags (.appconfig.featureflag/ prefix) | Key Vault references (pointer, not secret)
Refresh: poll sentinel key; on change reload registered keys
```

| Item | Note |
|---|---|
| Labels | Variants of a key (e.g., Dev/Prod) |
| Key Vault reference | Content type `application/vnd.microsoft.appconfig.keyvaultref+json`; app resolves secret using its own identity |
| Pricing tiers | Free / Standard (verify limits) |

### 5.1 [Easy] What is App Configuration and how does it differ from Key Vault?
**Answer:** A centralized store for non-secret settings and feature flags with labels and dynamic refresh. Key Vault stores secrets/keys/certs with stricter access control and auditing.
**Explanation:** Use App Config for settings; store secrets in Key Vault and reference them from App Config. The app needs RBAC on both: "App Configuration Data Reader" and "Key Vault Secrets User".
**Code / CLI / Bicep**
```bash
az appconfig kv set -n myappcfg --key Orders:BatchSize --value 50 --label Prod --yes
az appconfig kv set-keyvault -n myappcfg --key Orders:DbPassword --secret-identifier https://kv.vault.azure.net/secrets/dbpwd --yes
```

### 5.2 [Follow-up] How do you use it in .NET with dynamic refresh via a sentinel key?
**Answer:** Add `AddAzureAppConfiguration`, select keys/labels, register a sentinel key with `refreshAll: true`, and call `UseAzureAppConfiguration` middleware. Changing only the sentinel triggers a reload of all keys.
**Explanation:** Refresh is polling-based (default interval 30s) and triggered by incoming requests/`TryRefreshAsync`. The sentinel avoids partial updates when changing many keys. Use `IOptionsSnapshot`/`IOptionsMonitor` to see new values. Event Grid push-based refresh is also available (verify in docs).
**Code / CLI / Bicep**
```csharp
builder.Configuration.AddAzureAppConfiguration(o =>
{
    o.Connect(new Uri(endpoint), new DefaultAzureCredential())
     .Select("Orders:*", "Prod")
     .ConfigureKeyVault(kv => kv.SetCredential(new DefaultAzureCredential()))
     .ConfigureRefresh(r => r.Register("Sentinel", "Prod", refreshAll: true)
                              .SetRefreshInterval(TimeSpan.FromSeconds(30)));
});
builder.Services.AddAzureAppConfiguration();
app.UseAzureAppConfiguration();
```

### 5.3 [Medium] How do feature flags work?
**Answer:** Flags are stored as special key-values and consumed via Microsoft.FeatureManagement; `IFeatureManager.IsEnabledAsync("Beta")` evaluates them, with filters like percentage, time window, or targeting.
**Explanation:** Enable with `UseFeatureFlags()` in the provider options; flags refresh on their own interval. Targeting filter requires a targeting context (user/groups).
**Code / CLI / Bicep**
```csharp
o.UseFeatureFlags(f => f.SetRefreshInterval(TimeSpan.FromSeconds(30)));
builder.Services.AddFeatureManagement();
if (await features.IsEnabledAsync("NewCheckout")) { /* ... */ }
```

### 5.4 [Hard] Functions integration and Key Vault references - what are the pitfalls?
**Answer:** In Functions (isolated), add the provider in `ConfigureAppConfiguration`; refresh requires middleware (`UseAzureAppConfiguration`) since there's no request pipeline by default. Alternatively use App Service/Functions' own `@Microsoft.AppConfiguration(...)` / Key Vault reference app settings.
**Explanation:** Key Vault references via the provider use the *app's* credential, so grant that identity access to the vault. For Functions on Consumption, startup load adds cold-start latency; cache wisely. Rotating a secret: App Config holds the URI only; with unversioned URI, the new version is picked up on refresh of that key (configure `SetSecretRefreshInterval`, verify).
**Code / CLI / Bicep**
```csharp
var host = new HostBuilder()
  .ConfigureFunctionsWebApplication(w => w.UseAzureAppConfiguration())
  .ConfigureAppConfiguration(c => c.AddAzureAppConfiguration(o =>
      o.Connect(new Uri(endpoint), new DefaultAzureCredential())
       .ConfigureRefresh(r => r.Register("Sentinel", refreshAll: true))))
  .ConfigureServices(s => s.AddAzureAppConfiguration())
  .Build();
```
**Gotchas / follow-ups**
- Without the middleware/service, refresh never fires.
- Request limits/throttling on the store: cache and use sensible intervals.

### 5.5 [Tricky] You changed a value but the app still shows the old one. Why?
**Answer:** Typically the changed key isn't registered for refresh (only the sentinel is watched), the refresh interval hasn't elapsed, the middleware isn't wired, or the code reads `IOptions<T>` (singleton snapshot) rather than `IOptionsMonitor`/`IOptionsSnapshot`.
**Explanation:** Also check label mismatch (key saved under a different label) and that the sentinel value actually changed (refreshAll triggers only when sentinel's ETag changes).
**Code / CLI / Bicep**
```bash
az appconfig kv list -n myappcfg --key "Orders:*" --label Prod
az appconfig kv set -n myappcfg --key Sentinel --value $(date +%s) --label Prod --yes
```

**Interview traps**
- Putting secrets directly in App Config.
- Forgetting RBAC to Key Vault for the app's identity.
- `IOptions<T>` never refreshes.
- Refresh is polling; not instant.

**Cheat sheet**
- Label = environment/variant.
- Sentinel + refreshAll for atomic reload.
- KV reference = pointer; app resolves.
- Microsoft.FeatureManagement for flags.
- Role: App Configuration Data Reader.

---

## Topic 6: API Management in depth

**Study notes**

```
Client --(Ocp-Apim-Subscription-Key / JWT)--> APIM gateway
   inbound policies -> backend -> outbound policies (on-error)
Products group APIs; Subscriptions issue keys at product/API/all-APIs scope
Tiers: Consumption | Developer | Basic | Standard | Premium (multi-region, VNet, self-hosted gateway) | v2 tiers (verify)
```

### 6.1 [Easy] What are products, subscriptions, and policies?
**Answer:** A product bundles APIs with terms/quotas; a subscription provides keys to access a product/API; policies are XML rules executed in the gateway pipeline (inbound, backend, outbound, on-error).
**Explanation:** Policies can be scoped global > product > API > operation, combined using `<base />`. The developer portal lets consumers discover APIs and obtain keys.
**Code / CLI / Bicep**
```xml
<policies>
  <inbound><base />
    <rate-limit-by-key calls="100" renewal-period="60" counter-key="@(context.Subscription.Id)" />
  </inbound>
  <backend><base /></backend><outbound><base /></outbound><on-error><base /></on-error>
</policies>
```

### 6.2 [Follow-up] How does APIM authenticate to a backend with managed identity?
**Answer:** Use the `authentication-managed-identity` policy to get a token for a resource and set it as the Authorization header to the backend (e.g., a Function or Service Bus).
**Explanation:** Enable system- or user-assigned identity on APIM and grant it RBAC/app role on the backend. For UAMI specify `client-id`. Use `output-token-variable-name` to reuse the token.
**Code / CLI / Bicep**
```xml
<inbound><base />
  <authentication-managed-identity resource="api://my-backend-app-id" />
</inbound>
```

### 6.3 [Medium] Versioning, caching, and throttling policies.
**Answer:** Versioning schemes: path, query, or header with version sets; revisions are non-breaking changes. Caching via `cache-lookup`/`cache-store` (built-in or external Redis). Throttling via `rate-limit` (per subscription) and `quota`, or `rate-limit-by-key` for custom keys (e.g., IP).
**Explanation:** `rate-limit` returns 429; `quota` is a long-period cap. Caching vary-by headers/query to avoid leaking data. Built-in cache is lost on gateway restart in some tiers and not shared across regions - use external Redis for prod (verify).
**Code / CLI / Bicep**
```xml
<inbound><base />
  <cache-lookup vary-by-developer="false" vary-by-developer-groups="false" downstream-caching-type="none">
    <vary-by-query-parameter>id</vary-by-query-parameter>
  </cache-lookup>
</inbound>
<outbound><cache-store duration="60" /><base /></outbound>
```

### 6.4 [Hard] Validate JWTs and protect backends; self-hosted gateway overview.
**Answer:** Use `validate-jwt` (OpenID config URL, audiences, required claims) in inbound, and restrict the backend to APIM (managed identity, IP/VNet, or shared header). The self-hosted gateway is a containerized APIM gateway you run on-prem/other clouds, managed from the Azure control plane (Premium/Developer; verify).
**Explanation:** Subscription key identifies the *app*; JWT identifies the *user/app identity* - use both for layered security. Self-hosted gateway needs outbound connectivity to the config endpoint; it keeps working with cached config if disconnected.
**Code / CLI / Bicep**
```xml
<validate-jwt header-name="Authorization" failed-validation-httpcode="401">
  <openid-config url="https://login.microsoftonline.com/{tenant}/v2.0/.well-known/openid-configuration" />
  <audiences><audience>api://orders-api</audience></audiences>
  <required-claims><claim name="roles" match="any"><value>Orders.Read</value></claim></required-claims>
</validate-jwt>
```
**Gotchas / follow-ups**
- Don't rely on subscription keys alone as auth.
- Strip/forward headers deliberately (`set-header`).

### 6.5 [Tricky] Intermittent 429/500 from APIM while the backend is healthy. What do you check?
**Answer:** Check policy-generated 429s (rate-limit/quota), backend timeouts/retries (`retry` policy double-sends non-idempotent requests), capacity metrics on the tier, and gateway SNAT/connection exhaustion.
**Explanation:** Use `LastErrorSource/LastErrorReason` in logs and Application Insights correlation. `retry` with POST can duplicate - condition on idempotent methods. Consumption tier has cold start behavior.
**Code / CLI / Bicep**
```kusto
ApiManagementGatewayLogs
| where TimeGenerated > ago(1h) and ResponseCode >= 429
| summarize count() by ResponseCode, LastErrorReason, ApiId
```

**Interview traps**
- Subscription key != user authentication.
- Retry policy on non-idempotent calls.
- Revision vs version confusion.
- Built-in cache not for multi-region consistency.

**Cheat sheet**
- Policy order: inbound -> backend -> outbound; `<base />` inherits.
- `authentication-managed-identity` for backend tokens.
- `validate-jwt` for callers.
- `rate-limit-by-key` for custom throttles.
- Version sets for breaking changes.

---

## Topic 7: Data Factory / Synapse overview

**Study notes**

```
ADF: Pipeline -> Activities (Copy, Data Flow, Lookup, ForEach, Web, Execute Pipeline) ; Linked services; Datasets; Triggers (schedule, tumbling window, event, manual)
Integration Runtime: Azure | Self-hosted | Azure-SSIS
Synapse: ADF-like pipelines + Spark + dedicated/serverless SQL pools in one workspace
```

### 7.1 [Easy] What is ADF and what are its core building blocks?
**Answer:** A managed data integration/orchestration (ETL/ELT) service. Pipelines contain activities that use linked services (connections) and datasets (data shapes), run by triggers on an integration runtime.
**Explanation:** Copy Activity moves data at scale between 90+ connectors; mapping data flows do Spark-based transforms without code. ADF is orchestration-first and data-volume oriented, not event-per-message.
**Code / CLI / Bicep**
```bash
az datafactory pipeline create-run -g rg --factory-name adf1 --name CopyOrders
```

### 7.2 [Follow-up] ADF vs Functions vs Logic Apps - when?
**Answer:** ADF for bulk/batch data movement and transformation with many connectors and lineage; Logic Apps for SaaS/business workflows with low-code connectors and approvals; Functions for custom code, event-driven micro-processing.
**Explanation:** Don't loop a Function per row for millions of rows - use Copy/Data Flow. Don't write complex transforms in Logic Apps. They compose: ADF can call Functions (Azure Function activity) or Web activity; Logic Apps can trigger an ADF run.

### 7.3 [Medium] Tumbling window vs schedule trigger; self-hosted IR.
**Answer:** Schedule triggers fire at times with no state; tumbling window triggers are contiguous, non-overlapping, support backfill, dependencies, retries and pass window start/end. Self-hosted IR is an agent installed on-prem/VM to reach private data sources.
**Explanation:** Tumbling window passes `@trigger().outputs.windowStartTime` for incremental loads. Self-hosted IR can be scaled to multiple nodes for HA. Managed VNet IR + managed private endpoints for private connectivity from cloud.
**Code / CLI / Bicep**
```json
{ "properties": { "type": "TumblingWindowTrigger",
  "typeProperties": { "frequency": "Hour", "interval": 1, "startTime": "2026-10-01T00:00:00Z", "maxConcurrency": 2 },
  "pipeline": { "pipelineReference": { "referenceName": "Incremental", "type": "PipelineReference" },
    "parameters": { "from": "@trigger().outputs.windowStartTime", "to": "@trigger().outputs.windowEndTime" } } } }
```

### 7.4 [Hard] Idempotent incremental loads and secure credentials.
**Answer:** Use watermark (lookup last value -> copy delta -> update watermark) or window-based loads with upsert/overwrite of the target partition so reruns are safe. Store secrets in Key Vault linked service and use the factory's managed identity for sources/sinks.
**Explanation:** Reruns after failure must not duplicate. Use staging + MERGE in SQL. Grant the ADF managed identity access on SQL (contained user) and Storage (Blob Data Contributor).
**Code / CLI / Bicep**
```sql
MERGE dbo.Orders AS t USING stg.Orders AS s ON t.Id = s.Id
WHEN MATCHED THEN UPDATE SET t.Total = s.Total
WHEN NOT MATCHED THEN INSERT (Id, Total) VALUES (s.Id, s.Total);
```
**Gotchas / follow-ups**
- Cost drivers: DIU-hours, data flow cluster time, activity runs - verify pricing.
- Debug runs cost too.

**Interview traps**
- Using ADF for low-latency event handling.
- Forgetting IR networking for private sources.
- Non-idempotent reruns.

**Cheat sheet**
- Pipeline/Activity/Linked service/Dataset/Trigger/IR.
- Batch -> ADF; workflow -> Logic Apps; code -> Functions.
- Tumbling window for backfill.
- Synapse = ADF pipelines + Spark + SQL pools.

---

## Topic 8: Container Apps, Jobs & AKS basics

**Study notes**

```
Container Apps: Environment -> Apps (revisions, ingress, Dapr optional, KEDA scale rules) | Jobs (manual/schedule/event)
AKS: Cluster -> Node pools -> Pods (Deployment) -> Service/Ingress ; probes; HPA; CSI Secrets Store; Workload Identity
```

| Need | Pick |
|---|---|
| Serverless containers, scale to zero, minimal ops | Container Apps |
| Full Kubernetes control, custom operators | AKS |
| Event-driven short code | Functions |

### 8.1 [Easy] What are Container Apps and Jobs?
**Answer:** Container Apps is a serverless container platform built on Kubernetes/KEDA with scale-to-zero, revisions and ingress. Jobs run containers to completion - manual, scheduled (cron), or event-driven.
**Explanation:** No cluster management. Apps are long-running (HTTP or background workers); jobs are finite tasks (batch, migrations).
**Code / CLI / Bicep**
```bash
az containerapp create -n worker -g rg --environment env1 --image myacr.azurecr.io/worker:1 \
  --min-replicas 0 --max-replicas 10 --user-assigned <uami-id> --registry-server myacr.azurecr.io --registry-identity <uami-id>
```

### 8.2 [Follow-up] How does KEDA scale on Service Bus?
**Answer:** Define a `azure-servicebus` scale rule with queue name and `messageCount` target; KEDA scales replicas based on queue length, down to zero when empty. Auth via managed identity (`identity` on the rule) or a secret connection string.
**Explanation:** Replicas ≈ ceil(messages / messageCount), capped by maxReplicas. Each replica should use a receiver with sensible `MaxConcurrentCalls`; settle messages (complete/abandon) properly. Scale latency (polling interval) means not instant.
**Code / CLI / Bicep**
```bash
az containerapp update -n worker -g rg --min-replicas 0 --max-replicas 20 \
  --scale-rule-name sb --scale-rule-type azure-servicebus \
  --scale-rule-metadata queueName=orders messageCount=50 namespace=mysb \
  --scale-rule-identity <uami-id>
```

### 8.3 [Medium] AKS: pods, probes, HPA.
**Answer:** A pod is the smallest deployable unit (one or more containers). Liveness probe restarts hung containers; readiness gates traffic; startup probe protects slow starts. HPA scales replicas on CPU/memory/custom metrics (KEDA for event metrics).
**Explanation:** Always set resource requests/limits - HPA CPU % is relative to requests. For .NET, expose `/healthz` via `MapHealthChecks`; make liveness cheap (don't check DB, or you cause restart storms).
**Code / CLI / Bicep**
```yaml
livenessProbe:  { httpGet: { path: /healthz/live,  port: 8080 }, periodSeconds: 10 }
readinessProbe: { httpGet: { path: /healthz/ready, port: 8080 }, periodSeconds: 5 }
resources: { requests: { cpu: 250m, memory: 256Mi }, limits: { memory: 512Mi } }
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec: { scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: orders },
        minReplicas: 2, maxReplicas: 10,
        metrics: [{ type: Resource, resource: { name: cpu, target: { type: Utilization, averageUtilization: 70 } } }] }
```

### 8.4 [Hard] Secrets CSI driver and workload identity.
**Answer:** Workload identity federates a Kubernetes ServiceAccount to an Entra identity (UAMI or app) via OIDC issuer so pods get tokens without secrets. The Key Vault CSI driver (Secrets Store CSI + Azure provider) mounts KV secrets as volumes and can sync to K8s Secrets.
**Explanation:** Needs OIDC issuer + workload identity enabled on the cluster, a federated credential matching `system:serviceaccount:<ns>:<sa>`, SA annotated with client id, and pod label `azure.workload.identity/use: "true"`. In .NET, `DefaultAzureCredential` picks it up. This replaces the older pod identity (deprecated).
**Code / CLI / Bicep**
```bash
az aks update -g rg -n aks1 --enable-oidc-issuer --enable-workload-identity
az identity federated-credential create --name fc1 --identity-name uami1 -g rg \
  --issuer $(az aks show -g rg -n aks1 --query oidcIssuerProfile.issuerUrl -o tsv) \
  --subject system:serviceaccount:orders:orders-sa --audiences api://AzureADTokenExchange
```
**Gotchas / follow-ups**
- Subject string must match exactly (namespace + SA name).
- Mounted CSI secrets don't auto-reload in app unless you re-read files/enable rotation.

### 8.5 [Tricky] KEDA scales Container App to 20 replicas but the queue isn't draining and messages hit DLQ.
**Answer:** Likely lock expiry or downstream saturation: many replicas hammer a DB/API, processing exceeds lock duration, deliveries exceed MaxDeliveryCount.
**Explanation:** Limit `maxReplicas` and per-replica concurrency to what dependencies handle, tune lock duration/auto-renew, and make processing idempotent. Check DLQ reason (`MaxDeliveryCountExceeded`, `LockLost`).
**Code / CLI / Bicep**
```csharp
var processor = client.CreateProcessor("orders", new ServiceBusProcessorOptions
{ MaxConcurrentCalls = 8, MaxAutoLockRenewalDuration = TimeSpan.FromMinutes(5) });
```

**Interview traps**
- Liveness probe checking dependencies -> restart storms.
- No resource requests -> HPA misbehaves.
- Scale-to-zero cold start is not for latency-sensitive HTTP.
- Federated subject mismatch.

**Cheat sheet**
- ACA = KEDA built in; AKS = bring your own KEDA/HPA.
- Probes: startup/liveness/readiness.
- Workload identity = OIDC federation, no secrets.
- Jobs for finite work.

---

## Topic 9: Azure Cache, CDN & Front Door caching

**Study notes**

```
Browser -> Front Door/CDN (edge cache, WAF, routing) -> App Service/APIM/Functions -> Azure Cache for Redis (cache-aside) -> DB
```

| Layer | Caches | Control |
|---|---|---|
| Edge (Front Door/CDN) | Static/cacheable responses | Cache-Control, rules engine, purge |
| App cache (Redis) | Data/session/computed | TTL, eviction |
| In-proc (MemoryCache) | Per-instance | Fast, not shared |

### 9.1 [Easy] What is cache-aside with Redis?
**Answer:** The app checks the cache, on miss loads from the DB and stores with a TTL; writes update the DB and invalidate/update the cache.
**Explanation:** Always set expirations, handle cache failure gracefully (fall through to DB), and keep a single `ConnectionMultiplexer` per app. Redis supports Entra auth (verify SKU/support in current docs).
**Code / CLI / Bicep**
```csharp
var db = mux.GetDatabase();
var json = await db.StringGetAsync($"order:{id}");
if (json.IsNull) { var o = await repo.GetAsync(id); await db.StringSetAsync($"order:{id}", JsonSerializer.Serialize(o), TimeSpan.FromMinutes(10)); return o; }
return JsonSerializer.Deserialize<Order>(json!);
```

### 9.2 [Follow-up] Front Door vs CDN caching and purge?
**Answer:** Both cache at edge POPs honoring `Cache-Control`; Front Door adds global L7 routing, WAF, health probes, failover. Purge by path after content changes or use versioned URLs.
**Explanation:** Query-string caching behavior must be configured (ignore vs include). Compression and caching rules are per route/rule set. Product lineup (CDN Standard Microsoft/Verizon etc.) has been changing - verify current docs.
**Code / CLI / Bicep**
```bash
az afd endpoint purge -g rg --profile-name fd1 --endpoint-name ep1 --content-paths "/assets/*"
```

### 9.3 [Medium] Cache stampede, stale data, and eviction.
**Answer:** A stampede is many concurrent misses hitting the DB for the same hot key. Mitigate with per-key locks/single-flight, TTL jitter, and stale-while-revalidate.
**Explanation:** Redis eviction policy (default volatile-lru) matters when memory fills. Use jittered TTL so keys don't expire together. Redis is shared state: serialize carefully and version keys on schema changes.
**Code / CLI / Bicep**
```csharp
var ttl = TimeSpan.FromMinutes(10) + TimeSpan.FromSeconds(Random.Shared.Next(0, 60));
await db.StringSetAsync(key, value, ttl, When.NotExists);
```

### 9.4 [Hard] Redis in a Functions/ASP.NET app: connection issues and failover.
**Answer:** Use one multiplexer, `abortConnect=false`, retry policy, and treat timeouts as cache miss. During Redis patching/failover, connections drop and reconnect.
**Explanation:** Common errors: `RedisTimeoutException` from thread-pool starvation, large values, or sync-over-async. Keep values small, avoid `KEYS`, use `SCAN`. Functions scale-out multiplies connections; check connection limits per SKU.
**Code / CLI / Bicep**
```csharp
var opts = ConfigurationOptions.Parse(host + ":6380,ssl=true");
opts.AbortOnConnectFail = false; opts.ConnectRetry = 5; opts.ReconnectRetryPolicy = new ExponentialRetry(5000);
var mux = await ConnectionMultiplexer.ConnectAsync(opts);
```
**Gotchas / follow-ups**
- Don't use Redis as the only source of truth unless persistence tier chosen.

**Interview traps**
- No TTL -> memory growth.
- Per-request ConnectionMultiplexer.
- Caching user-specific responses at the edge.

**Cheat sheet**
- Cache-aside + TTL jitter.
- Singleton multiplexer.
- Front Door = routing + WAF + cache.
- Versioned URLs beat purges.

---

## Topic 10: Azure Communication Services / SendGrid email

**Study notes**

```
App -> ACS Email (Email Communication Service + verified domain) -> recipients ; delivery events via Event Grid
App -> SendGrid API (3rd party via Azure Marketplace) -> recipients ; webhooks for events
```

### 10.1 [Easy] How do you send email from a .NET app on Azure?
**Answer:** Options: Azure Communication Services Email (first-party, Azure.Communication.Email SDK, supports managed identity/Entra) or SendGrid (Marketplace, API key). Azure has no built-in SMTP relay service.
**Explanation:** ACS Email needs an Email Communication Service resource, a verified (or Azure-managed) domain, and a connected Communication Service. Prefer queueing sends via Service Bus so the request path isn't blocked.
**Code / CLI / Bicep**
```csharp
var client = new EmailClient(new Uri("https://myacs.communication.azure.com"), new DefaultAzureCredential());
var op = await client.SendAsync(WaitUntil.Started, "DoNotReply@mydomain.com", "user@example.com",
    "Order confirmed", "<p>Thanks</p>");
```

### 10.2 [Follow-up] ACS vs SendGrid?
**Answer:** ACS is Azure-native with RBAC/managed identity and billing; SendGrid has mature marketing/template/analytics features and API-key auth.
**Explanation:** Both need SPF/DKIM/DMARC DNS records for deliverability. Check quotas/rate limits and sender reputation; verify current limits in docs.

### 10.3 [Medium] Idempotent, reliable email sending from a queue consumer.
**Answer:** Record a send-key (e.g., orderId+template) before/after sending so retries don't email twice; handle transient failures by abandoning/retrying and permanent failures (invalid address) by dead-lettering.
**Explanation:** Email APIs are not idempotent by default; "accepted" does not mean delivered - track status via ACS delivery events (Event Grid) or SendGrid webhooks.
**Code / CLI / Bicep**
```csharp
if (!await _sent.TryMarkAsync($"{orderId}:confirmation")) return; // unique-key insert
try { await _email.SendAsync(WaitUntil.Started, from, to, subject, html); }
catch (RequestFailedException ex) when (ex.Status is 429 or >= 500) { throw; } // retry
```

### 10.4 [Hard] Bounces, throttling, and secrets.
**Answer:** Handle 429 with backoff, process bounce/complaint events to suppress bad addresses, and keep SendGrid keys in Key Vault (or use ACS with managed identity).
**Explanation:** Sending to bounced addresses harms reputation. Template rendering should avoid user-controlled HTML injection.
**Gotchas / follow-ups**
- Marketplace SendGrid billing/ownership details - verify.

**Interview traps**
- Assuming SMTP relay exists natively.
- Treating "queued" as "delivered".
- Missing DKIM/SPF.

**Cheat sheet**
- ACS Email: domain + Entra auth.
- SendGrid: API key in Key Vault.
- Send via queue; dedup key.
- Track delivery via events.

---

## Topic 11: Azure SignalR Service

**Study notes**

```
Default mode: Clients <--WebSocket--> Azure SignalR <--server conn--> ASP.NET Core hub server(s)
Serverless mode: Clients <-> SignalR <- REST/Functions output binding (no hub server)
Classic mode: legacy/mixed (verify)
```

| Concept | Note |
|---|---|
| Units | Scale by units (each ~1,000 concurrent connections; verify) |
| Transports | WebSocket > SSE > Long polling |
| Negotiate | Client calls /negotiate to get service URL + access token |

### 11.1 [Easy] What does Azure SignalR Service do?
**Answer:** It offloads persistent client connections for real-time messaging so your app servers don't hold them, allowing scale to many connections.
**Explanation:** Your app calls `AddAzureSignalR()`; clients connect to the service endpoint. Groups, users and broadcast work as with SignalR.
**Code / CLI / Bicep**
```csharp
builder.Services.AddSignalR().AddAzureSignalR(builder.Configuration["Azure:SignalR:ConnectionString"]);
app.MapHub<ChatHub>("/chat");
```

### 11.2 [Follow-up] Serverless mode with Functions bindings?
**Answer:** A negotiate HTTP function uses the `SignalRConnectionInfo` input binding; other functions push with the `SignalR` output binding (target, arguments, userId/groupName).
**Explanation:** Upstream settings can route client-to-server messages to Functions endpoints. No hub class; clients invoke via upstream/REST.
**Code / CLI / Bicep**
```csharp
[Function("negotiate")]
public static string Negotiate([HttpTrigger(AuthorizationLevel.Anonymous)] HttpRequestData req,
    [SignalRConnectionInfoInput(HubName = "notify", UserId = "{headers.x-user-id}")] string connInfo) => connInfo;

[Function("Broadcast")]
[SignalROutput(HubName = "notify")]
public static SignalRMessageAction Run([ServiceBusTrigger("events")] string msg) =>
    new("newMessage") { Arguments = new object[] { msg } };
```

### 11.3 [Medium] Authentication and scaling considerations.
**Answer:** Authenticate at negotiate (JWT/cookie) and bind userId claim so you can target users; the service issues a short-lived access token. Scale via units; use Standard/Premium tier for SLA, more units (verify).
**Explanation:** For browsers, JWT goes in query string `access_token` for WebSocket. Service supports Entra for server connections (instead of access keys) - verify. Message size and per-unit rate limits apply.
**Code / CLI / Bicep**
```csharp
builder.Services.AddAuthentication().AddJwtBearer(o => o.Events = new JwtBearerEvents {
    OnMessageReceived = c => { var t = c.Request.Query["access_token"];
        if (!string.IsNullOrEmpty(t) && c.HttpContext.Request.Path.StartsWithSegments("/chat")) c.Token = t; return Task.CompletedTask; } });
```

### 11.4 [Hard] Reconnect behaviour and multi-instance pitfalls.
**Answer:** Clients should use `withAutomaticReconnect()` and re-join groups on reconnect (group membership is lost); messages sent while disconnected are not replayed.
**Explanation:** For guaranteed delivery persist state elsewhere (e.g., store + fetch missed on reconnect). Sticky sessions aren't needed with the service. Don't assume single hub instance state.
**Gotchas / follow-ups**
- Negotiate endpoint on Functions needs CORS configured.
- Connection counts: check unit limits when load testing.

**Interview traps**
- Expecting message replay.
- In-memory hub state across instances.
- Forgetting CORS on negotiate.

**Cheat sheet**
- Default vs serverless mode.
- Negotiate -> token.
- Re-join groups after reconnect.
- Output binding for push from Functions.

---

## Topic 12: Azure OpenAI / AI Search integration points (brief)

**Study notes**

```
App (MI) -> Azure OpenAI deployment (chat/embeddings) ; App -> AI Search index (vector + keyword) -> RAG prompt
Auth: Entra (Cognitive Services OpenAI User) preferred over keys ; Limits: TPM/RPM quotas per deployment -> 429
```

### 12.1 [Easy] How does a .NET app call Azure OpenAI securely?
**Answer:** Use the Azure.AI.OpenAI SDK with `DefaultAzureCredential` and the role "Cognitive Services OpenAI User" instead of API keys; address a *deployment name*, not a model name.
**Explanation:** Keys can be disabled for local-auth hardening. SDK surface changes between versions - verify in current docs.
**Code / CLI / Bicep**
```csharp
var client = new AzureOpenAIClient(new Uri(endpoint), new DefaultAzureCredential());
var chat = client.GetChatClient("gpt-deployment");
var res = await chat.CompleteChatAsync(new UserChatMessage("Summarize this order"));
```
```bash
az role assignment create --assignee <principal-id> --role "Cognitive Services OpenAI User" --scope <aoai-id>
```

### 12.2 [Follow-up] How do you handle rate limits?
**Answer:** Quotas are tokens-per-minute/requests-per-minute per deployment; exceeding returns 429 with Retry-After. Use SDK retries, exponential backoff, request queueing (Service Bus), smaller prompts, and multiple deployments/regions behind APIM load balancing.
**Explanation:** APIM has AI gateway policies (e.g., token limit/metrics - verify current names). Provisioned throughput deployments give reserved capacity.

### 12.3 [Medium] Where does AI Search fit (RAG)?
**Answer:** Index your documents (chunks + embeddings) in AI Search; at query time retrieve top-k relevant chunks via hybrid/vector search and add them to the prompt. Authenticate with RBAC ("Search Index Data Reader").
**Explanation:** Indexers can pull from Blob/SQL/Cosmos on schedule; chunk size and security trimming (filter by user group field) matter.
**Code / CLI / Bicep**
```csharp
var search = new SearchClient(new Uri(searchEndpoint), "docs", new DefaultAzureCredential());
var results = await search.SearchAsync<SearchDocument>("return policy", new SearchOptions { Size = 5 });
```

### 12.4 [Hard] Production gotchas.
**Answer:** Treat the LLM as a flaky, slow, non-deterministic dependency: timeouts, retries, circuit breaker, content filtering handling (400 filtered), prompt-injection defense, no secrets/PII in prompts without policy, and logging token usage.
**Explanation:** Never put authorization decisions on model output. Apply security trimming at retrieval. Verify data residency/retention in docs.

**Interview traps**
- Using model name instead of deployment name.
- Ignoring 429/TPM quotas.
- Retrieval without access trimming.

**Cheat sheet**
- MI + OpenAI User role.
- Deployment name in code.
- 429 -> backoff/queue/multi-deployment.
- RAG = AI Search + prompt.

---

## Topic 13: Messaging pattern decision matrix

**Study notes**

```
Command  = "do this" (one receiver, must be processed)      -> Service Bus queue / Storage Queue
Event    = "this happened" (0..N subscribers, reactive)      -> Event Grid / Service Bus topic
Stream   = high-volume ordered telemetry, replayable         -> Event Hubs
```

| Service | Delivery | Order | DLQ | Size/scale | Best for |
|---|---|---|---|---|---|
| Service Bus | Pull (peek-lock), at-least-once/at-most-once | FIFO w/ sessions | Yes | Moderate, rich features | Commands, workflows |
| Event Grid | Push | No | Dead-letter blob | Very high fan-out | Reactive events |
| Event Hubs | Pull w/ offsets | Per partition | No | Millions/sec | Telemetry/streaming |
| Storage Queue | Pull | Best-effort | Poison queue manual | Huge, cheap, simple | Simple buffering |
| Redis pub/sub | Push, fire-and-forget | n/a | No, no persistence | Low latency | Ephemeral notifications |

### 13.1 [Easy] Command vs event vs stream?
**Answer:** A command targets one handler and expects it to be performed; an event announces a fact to unknown subscribers; a stream is a continuous ordered flow of records consumed by position.
**Explanation:** Naming helps: commands imperative (`ChargeCard`), events past tense (`OrderPlaced`). Mixing them causes coupling.

### 13.2 [Follow-up] 10 scenarios - which service?
**Answer:** (1) Order processing needing retries/DLQ -> Service Bus queue. (2) Blob uploaded triggers workflow -> Event Grid. (3) IoT telemetry 100k msg/s -> Event Hubs. (4) Ordered per-customer processing -> Service Bus sessions. (5) Cheap background job queue, >80 GB backlog, simple -> Storage Queue. (6) Broadcast cache invalidation, loss tolerable -> Redis pub/sub. (7) Fan-out to multiple services with filters, durable -> Service Bus topic. (8) Resource change notification (VM created) -> Event Grid system topic. (9) Log aggregation/analytics feed -> Event Hubs (+Capture). (10) Delayed/scheduled message, deferral -> Service Bus.
**Explanation:** Combine: Event Grid -> Service Bus for durability; Event Hubs -> Stream Analytics/Functions for analysis. Storage Queue lacks sessions/topics/duplicate detection; message TTL up to 7 days historically (verify).
**Code / CLI / Bicep**
```csharp
await sender.SendMessageAsync(new ServiceBusMessage(json) { SessionId = customerId, MessageId = orderId });
```

### 13.3 [Medium] Service Bus queue vs topic, and Storage Queue vs Service Bus?
**Answer:** Queue = competing consumers; topic = pub/sub with per-subscription filters/copies. Choose Storage Queue only for simple, cost-sensitive buffering; Service Bus when you need DLQ, sessions, dedup, transactions, scheduled/deferred messages.
**Explanation:** Service Bus Premium for isolation, larger messages, VNet/private endpoints, geo-DR; verify current limits.

### 13.4 [Hard] Pitfalls when choosing.
**Answer:** Using Event Hubs as a work queue (no per-message settle), Event Grid as durable queue (limited retention), Redis pub/sub for reliable delivery (messages lost if subscriber absent).
**Explanation:** Ordering needs partition/session design; global ordering generally unattainable at scale.

### 13.5 [Tricky] Need ordering AND high throughput AND parallelism?
**Answer:** Order by key, not globally: Service Bus sessions or Event Hubs partition keys give per-key ordering with parallelism across keys.
**Explanation:** One hot key serializes. Handler failures on a session block it unless you dead-letter/skip.
**Gotchas / follow-ups**
- Session-enabled queue requires sessionId on every message.

**Interview traps**
- "Event Grid replaces Service Bus."
- Assuming exactly-once anywhere.
- Redis pub/sub durability.

**Cheat sheet**
- Command -> SB/Storage Queue; Event -> Grid; Stream -> Hubs.
- Per-key ordering only.
- Combine services for durability.
- Idempotent consumers always.

---

## Topic 14: Data consistency patterns across services

**Study notes**

```
No distributed ACID across SQL + Service Bus + Cosmos -> use eventual consistency patterns:
Outbox:  [Tx: write entity + outbox row] -> relay publishes to Service Bus -> mark sent
Saga:    steps + compensations (Durable Functions orchestrator)
Idempotency: key per operation ; Dedup: Service Bus duplicate detection (MessageId window) + consumer-side store
```

### 14.1 [Easy] Why can't you just write to SQL and send a Service Bus message?
**Answer:** The two aren't in one atomic transaction (dual-write problem): a crash between them leaves data saved but no message, or message sent but data rolled back.
**Explanation:** Fix with the outbox pattern or by making one store the source of truth and deriving the other.

### 14.2 [Follow-up] Explain the transactional outbox with SQL + Service Bus.
**Answer:** In the same SQL transaction, insert the business row and an Outbox row; a background relay reads unsent rows, publishes them (MessageId = outbox id), then marks them sent. Delivery is at-least-once, so consumers dedupe.
**Explanation:** Relay may crash after publish before marking - republish same MessageId; Service Bus duplicate detection (enabled on the entity, window up to 7 days) discards it. Alternative: Cosmos change feed as the relay.
**Code / CLI / Bicep**
```csharp
await strategy.ExecuteAsync(async () => {
  await using var tx = await db.Database.BeginTransactionAsync();
  db.Orders.Add(order);
  db.Outbox.Add(new OutboxMessage { Id = Guid.NewGuid(), Type = "OrderPlaced", Payload = json });
  await db.SaveChangesAsync(); await tx.CommitAsync();
});
// relay
foreach (var m in await db.Outbox.Where(x => x.SentAt == null).OrderBy(x => x.CreatedAt).Take(50).ToListAsync())
{ await sender.SendMessageAsync(new ServiceBusMessage(m.Payload) { MessageId = m.Id.ToString() }); m.SentAt = DateTime.UtcNow; }
await db.SaveChangesAsync();
```

### 14.3 [Medium] Idempotency keys and deduplication.
**Answer:** Clients send an idempotency key; the server stores key + result and returns the stored result on replay. Consumers record processed MessageIds (unique constraint) to skip duplicates.
**Explanation:** Service Bus duplicate detection only covers the sender-side window and same MessageId; consumer-side dedup is still required for redeliveries after lock loss. Do the dedup insert and the business write in the same transaction.
**Code / CLI / Bicep**
```sql
CREATE TABLE ProcessedMessages (MessageId nvarchar(100) PRIMARY KEY, ProcessedAt datetime2 NOT NULL);
-- INSERT in same tx as business change; duplicate key violation => already processed
```
```bicep
resource q 'Microsoft.ServiceBus/namespaces/queues@2022-10-01-preview' = {
  parent: ns, name: 'orders'
  properties: { requiresDuplicateDetection: true, duplicateDetectionHistoryTimeWindow: 'PT10M' }
}
```

### 14.4 [Hard] Saga with Durable Functions.
**Answer:** An orchestrator calls activities (reserve stock, charge, ship); on failure it runs compensating activities in reverse (refund, release). Orchestrators must be deterministic and activities idempotent.
**Explanation:** Use `context.CurrentUtcDateTime`, not `DateTime.UtcNow`; no direct I/O in the orchestrator. Activities have at-least-once execution. Use retry options and external events/timers for human approval or timeouts.
**Code / CLI / Bicep**
```csharp
[Function("OrderSaga")]
public static async Task Run([OrchestrationTrigger] TaskOrchestrationContext ctx)
{
    var o = ctx.GetInput<Order>()!;
    var done = new Stack<string>();
    try
    {
        await ctx.CallActivityAsync("ReserveStock", o); done.Push("ReleaseStock");
        await ctx.CallActivityAsync("Charge", o);       done.Push("Refund");
        await ctx.CallActivityAsync("Ship", o);
    }
    catch (TaskFailedException)
    {
        while (done.Count > 0) await ctx.CallActivityAsync(done.Pop(), o);
    }
}
```

### 14.5 [Tricky] Outbox relay scaled to multiple instances publishes duplicates and reorders.
**Answer:** Competing relays pick the same rows. Use row claiming (`UPDATE ... OUTPUT` with `READPAST`/lock), a single leader, or accept duplicates with consumer dedup; for order use SessionId = aggregate id and sequence numbers.
**Explanation:** Ordering across relays isn't guaranteed; consumers should tolerate out-of-order via version checks.
**Code / CLI / Bicep**
```sql
WITH cte AS (SELECT TOP (50) * FROM Outbox WITH (UPDLOCK, READPAST, ROWLOCK) WHERE SentAt IS NULL ORDER BY CreatedAt)
UPDATE cte SET ClaimedAt = SYSUTCDATETIME() OUTPUT inserted.*;
```
**Gotchas / follow-ups**
- Outbox table needs cleanup/retention.
- Poison outbox row blocks FIFO relays - add attempt count.

**Interview traps**
- "Exactly-once delivery" claims.
- Non-deterministic code in orchestrators.
- Dedup window expiry.
- Dedup check outside the business transaction.

**Cheat sheet**
- Outbox = atomic write + async publish.
- Duplicate detection on SB + consumer dedup.
- Saga = compensations, idempotent steps.
- Per-key ordering via sessions.

---

## Topic 15: Backup / DR across the stack

**Study notes**

```
RPO = tolerable data loss ; RTO = tolerable downtime
Active-passive:  Front Door/Traffic Manager -> Primary region (live) | Secondary region (warm/cold)
Data tier: SQL failover group | Storage RA-GRS/RA-GZRS | Service Bus Geo-DR (metadata pairing) | Cosmos multi-region | Key Vault (auto paired replication)
```

| Service | DR mechanism | Gotcha |
|---|---|---|
| SQL | Failover group (listener) | Async: RPO > 0 |
| Storage | GRS/RA-GRS (read secondary); customer-managed failover | Failover = data loss window (Last Sync Time) |
| Service Bus | Geo-Disaster Recovery (Premium): alias, pairs namespaces | Replicates metadata only, not messages; (Geo-replication feature for data exists - verify) |
| Functions/App | Deploy to secondary region; IaC; slots | Timer/queue triggers may run in both - guard |
| Front Door | Health probes, priority/weighted origins | Global service |
| Key Vault | Replicated to paired region, read-only during failover | Plan for secrets/certs |

### 15.1 [Easy] RPO vs RTO?
**Answer:** RPO is the maximum acceptable data loss measured in time; RTO is the maximum acceptable time to restore service. They drive architecture and cost.
**Explanation:** Backup-restore (hours), warm standby (minutes), active-active (near-zero) trade cost and complexity.

### 15.2 [Follow-up] How does Service Bus Geo-DR work?
**Answer:** You pair a primary and secondary namespace (Premium) under an alias DNS name; on failover the alias points to the secondary and pairing is broken. Only metadata (entities, config) is replicated - not messages in flight.
**Explanation:** Apps use the alias connection string so no change is needed. Messages not yet consumed in the primary are not available after failover. A newer Geo-Replication feature replicates data too - verify current docs/availability.
**Code / CLI / Bicep**
```bash
az servicebus georecovery-alias set -g rg --namespace-name sb-primary --alias sb-alias --partner-namespace <secondary-ns-id>
az servicebus georecovery-alias fail-over -g rg --namespace-name sb-secondary --alias sb-alias
```

### 15.3 [Medium] Storage RA-GRS and SQL failover groups in the plan.
**Answer:** RA-GRS replicates asynchronously to the paired region with read access to the secondary; check Last Sync Time before failover. SQL failover groups provide listener endpoints for transparent redirect.
**Explanation:** Apps reading from secondary must handle stale data; with RA-GRS use a retry-to-secondary read policy. Account failover makes the secondary primary (becomes LRS until reconfigured).
**Code / CLI / Bicep**
```bash
az storage account show -n mysa --expand geoReplicationStats --query geoReplicationStats.lastSyncTime
az storage account failover -n mysa -g rg --yes
```

### 15.4 [Hard] Design multi-region active-passive for Functions + Service Bus + SQL + Blob.
**Answer:** Front Door routes to primary with health probe; secondary has identical IaC-deployed Function app (disabled/warm), Service Bus Geo-DR alias, SQL failover group listener, RA-GZRS storage. Runbook: fail over data tier, enable secondary functions, verify, then flip traffic.
**Explanation:** Use alias/listener names in config so no code change. Keep Function triggers (timers, Service Bus) disabled in the passive region to avoid double-processing; use `AzureWebJobs.<name>.Disabled`. Test failover regularly. Key Vault secrets/certs and App Config must exist in the secondary; managed identity RBAC must be deployed there too (identities are regional objects per resource).
**Code / CLI / Bicep**
```bicep
resource afdOrigin 'Microsoft.Cdn/profiles/originGroups/origins@2023-05-01' = {
  parent: originGroup, name: 'secondary'
  properties: { hostName: secondaryApp.properties.defaultHostName, priority: 2, weight: 1000, enabledState: 'Enabled' }
}
```
**Gotchas / follow-ups**
- Duplicate processing if both regions' triggers run.
- Replaying messages lost on SB failover needs upstream resend/outbox.

### 15.5 [Tricky] After SQL failover-group failover, the app still errors for minutes. Why?
**Answer:** Stale DNS/pooled connections to the old primary, apps using server names instead of the listener, DNS TTL, or missing retry logic.
**Explanation:** Use listener endpoint, `ConnectRetryCount`, `EnableRetryOnFailure`, `ClearAllPools` after known failover. Also verify secondary has firewall/VNet rules, contained users (replicated) and the app identity RBAC on other resources.
**Code / CLI / Bicep**
```csharp
SqlConnection.ClearAllPools(); // after detecting repeated failures post-failover
```

**Interview traps**
- Backups are not DR; geo-redundancy isn't tested until you fail over.
- Service Bus Geo-DR doesn't move messages (classic Geo-DR).
- Failover replication lag = data loss.
- Forgetting to disable triggers in the passive region.

**Cheat sheet**
- Define RPO/RTO first.
- Aliases/listeners in config.
- Check Last Sync Time.
- Rehearse failover; runbook + IaC.
- Disable passive triggers.

