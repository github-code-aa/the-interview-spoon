# E02 — System Design & Production Scenarios (.NET / EF Core / SQL Server)

## How to use this file
- **Part 1** has 16 system designs. For each: practise the 45-minute flow — clarify → estimate → draw → schema → API → code → scale → fail → trade-offs. Say the trade-offs out loud.
- **Part 2** has 20 production incidents. Practise telling each as a STAR-style story: Symptom → Diagnosis → Root cause → Fix → Prevention.
- Numbers are back-of-envelope; interviewers care about reasoning, not exact digits.
- Code is illustrative and uses real APIs (EF Core 8, ASP.NET Core 8, SignalR, StackExchange.Redis, Polly/`Microsoft.Extensions.Resilience`). Adapt names to your project.
- Default stack assumption: ASP.NET Core + EF Core + SQL Server + Redis + a message broker (Azure Service Bus / RabbitMQ).

## Table of contents
**Part 1 — System Design**
1. [URL Shortener](#design-1-url-shortener)
2. [Rate Limiter](#design-2-rate-limiter)
3. [Order / Checkout with Inventory & Payments](#design-3-ordercheckout-system-with-inventory--payments)
4. [Notification Service](#design-4-notification-service-emailsmspush)
5. [File Upload / Processing Pipeline](#design-5-file-upload--processing-pipeline)
6. [Real-time Chat with SignalR](#design-6-real-time-chat-with-signalr)
7. [Job Scheduler](#design-7-job-scheduler)
8. [Multi-tenant SaaS](#design-8-multi-tenant-saas)
9. [Audit Log System](#design-9-audit-log-system)
10. [Search / Autocomplete](#design-10-searchautocomplete)
11. [Reporting / Analytics Dashboard on SQL Server](#design-11-reportinganalytics-dashboard-on-sql-server)
12. [Leaderboard](#design-12-leaderboard)
13. [Booking / Reservation (No Double-Booking)](#design-13-bookingreservation-system-with-double-booking-prevention)
14. [Idempotent Payment API](#design-14-idempotent-payment-api)
15. [Feature-Flag Service](#design-15-feature-flag-service)
16. [Ticketing with Flash-Sale Traffic](#design-16-ticketing-with-flash-sale-traffic)

**Part 2 — Production Scenarios**
1. API suddenly slow · 2. High CPU · 3. Memory leak · 4. Thread pool starvation · 5. SQL deadlocks · 6. N+1 in prod · 7. Connection pool exhausted · 8. Timeouts after deployment · 9. Migration locking huge table · 10. Stale cache · 11. Duplicate records from retries · 12. Message processed twice · 13. Works locally, fails in container/K8s · 14. Intermittent 500s under load · 15. Token expiry incidents · 16. DB failover errors · 17. Slow report query · 18. Disk full from logs · 19. Data corruption from race · 20. Rollback of bad release

---

# PART 1 — SYSTEM DESIGN

**A reusable opening script (say this first in every design):**
1. Restate the problem; ask functional + non-functional questions (scale, latency, consistency, availability, retention).
2. Estimate QPS, storage, bandwidth.
3. Draw the simplest design that works, then evolve it where numbers demand.
4. Go deep on 1–2 hard parts (the "interesting problem") — don't spread thin.
5. Close with failure modes, monitoring, and trade-offs.

---

## Design 1: URL Shortener

### Requirements & clarifying questions
- Functional: create short URL from long URL; redirect; optional custom alias; optional expiry; click analytics.
- Non-functional: redirect p99 < 50 ms, 99.99% availability, links must never collide, reads >> writes (100:1).
- Ask: custom aliases? expiry? same long URL → same short URL? Is analytics real-time? Abuse/malware checks? Is 301 (cacheable) or 302 (trackable) wanted?

### Capacity estimation
- 100 M new URLs/month ≈ 40 writes/s avg (peak ~200/s). Reads 100:1 ≈ 4 k/s avg, ~20 k/s peak.
- 5 years: 6 B URLs × ~500 B = ~3 TB. Key length: base62, 7 chars = 62^7 ≈ 3.5 trillion — plenty.
- Hot set small → Redis cache with 20% of daily traffic gives >90% hit rate.

### High-level architecture
```
Client ─► CDN/Edge ─► Load Balancer ─► ASP.NET Core API (stateless, N pods)
                                          │        │
                                    Redis cache    └─► SQL Server (primary + read replicas)
                                          │
                              click events ─► Service Bus/Kafka ─► Analytics worker ─► ClickStats table
```

### Data model
```sql
CREATE TABLE dbo.ShortUrl (
    Id          BIGINT IDENTITY(1,1) NOT NULL,
    Code        VARCHAR(12)   COLLATE Latin1_General_CS_AS NOT NULL, -- case-SENSITIVE
    LongUrl     NVARCHAR(2048) NOT NULL,
    LongUrlHash BINARY(32)    NOT NULL,          -- SHA-256, for dedupe lookup
    OwnerId     INT NULL,
    CreatedUtc  DATETIME2(0)  NOT NULL DEFAULT SYSUTCDATETIME(),
    ExpiresUtc  DATETIME2(0)  NULL,
    CONSTRAINT PK_ShortUrl PRIMARY KEY CLUSTERED (Id)
);
CREATE UNIQUE INDEX UX_ShortUrl_Code ON dbo.ShortUrl (Code);
CREATE INDEX IX_ShortUrl_Hash ON dbo.ShortUrl (LongUrlHash) INCLUDE (Code);
```
Note: Code column must be case-sensitive collation, otherwise `aB` and `Ab` collide.

### API design
```
POST /api/v1/urls            { "longUrl": "...", "alias": "mylink", "expiresUtc": "..." } → 201 { "shortUrl": "https://sho.rt/aZ3k9Q" }
GET  /{code}                 → 302 Location: <longUrl>   (404 if missing, 410 if expired)
GET  /api/v1/urls/{code}/stats
```

### Key .NET implementation
Code generation: take the identity `Id`, encode in base62 (no collisions, no retries). Optionally obfuscate with a salt/shuffled alphabet so IDs aren't guessable.
```csharp
public static class Base62
{
    private const string Alphabet = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";
    public static string Encode(long n)
    {
        if (n == 0) return "0";
        var sb = new System.Text.StringBuilder();
        while (n > 0) { sb.Insert(0, Alphabet[(int)(n % 62)]); n /= 62; }
        return sb.ToString();
    }
}

app.MapGet("/{code}", async (string code, IDistributedCache cache, AppDb db) =>
{
    var url = await cache.GetStringAsync(code);
    if (url is null)
    {
        var row = await db.ShortUrls.AsNoTracking()
            .Where(x => x.Code == code && (x.ExpiresUtc == null || x.ExpiresUtc > DateTime.UtcNow))
            .Select(x => x.LongUrl).FirstOrDefaultAsync();
        if (row is null) return Results.NotFound();
        url = row;
        await cache.SetStringAsync(code, url, new DistributedCacheEntryOptions { SlidingExpiration = TimeSpan.FromHours(6) });
    }
    return Results.Redirect(url, permanent: false); // 302 so we can count clicks
});
```
Insert flow: insert row → get `Id` → compute `Code` → update row (or use a SEQUENCE: `NEXT VALUE FOR` first, then insert with Code in one statement). Custom alias: rely on the unique index; catch `DbUpdateException` with `SqlException.Number` 2601/2627 → 409.

### Scaling & bottlenecks
- Reads: Redis + CDN; read replicas. Writes: one primary is fine at 200/s; shard by Code hash only at huge scale.
- Multiple writers need unique ID range: SEQUENCE with `CACHE`, or Snowflake-style IDs, or range allocation per node.
- Click analytics: never write synchronously on the redirect path — publish to queue, batch insert.

### Failure modes
- Redis down → fall back to DB (protect with circuit breaker + rate limit to avoid DB stampede).
- Cache stampede on a viral link → single-flight/`SemaphoreSlim` per key or `HybridCache` (.NET 9).
- DB failover → retry on transient errors (`EnableRetryOnFailure`).
- Abuse: rate-limit creation, block malicious domains.

### Trade-offs
- Counter/base62 (predictable, collision-free) vs random/hash (unguessable, needs collision handling).
- 301 (browser caches, fewer hits, lose analytics) vs 302 (every click hits you).
- SQL vs NoSQL: simple key→value fits both; SQL keeps constraints and is enough for this scale.

### Follow-up questions (easy → hard)
1. Why base62 instead of GUID? (short, URL-safe.)
2. How do you ensure uniqueness? (identity/sequence + unique index.)
3. How do you handle expired links cleanup? (lazy check on read + background purge job in batches.)
4. Same long URL submitted twice? (hash index lookup; policy decision.)
5. Multiple data centres generating IDs? (range allocation / Snowflake with node id.)
6. Prevent enumeration of codes? (shuffled alphabet / encrypted ID, or random codes with retry.)
7. 1 B clicks/day analytics? (stream, pre-aggregate per minute, columnstore/ClickHouse.)

---

## Design 2: Rate Limiter

### Requirements & clarifying questions
- Limit requests per client (API key/user/IP) per endpoint; return 429 + `Retry-After`.
- Ask: per-user vs per-IP vs per-tenant? Hard or soft limit? Multiple instances (distributed)? Burst allowed? Fail-open or fail-closed if limiter store is down? Tiered plans?

### Capacity estimation
- 50 k req/s at the gateway; each needs one limiter check → Redis must do ~50 k ops/s (single node handles ~100 k simple ops/s; use cluster beyond).
- Memory: 10 M active keys × ~100 B ≈ 1 GB.

### High-level architecture
```
Client ─► API Gateway / Middleware ──► Redis (atomic Lua counters/token buckets)
                  │ allowed
                  ▼
             Downstream API        (limits config in DB/appsettings, cached in memory)
```

### Data model
Redis keys, not SQL:
```
rl:{clientId}:{route}:{window}   → INCR counter, EXPIRE window   (fixed window)
rl:tb:{clientId}                 → hash {tokens, lastRefillTicks} (token bucket)
```
Config table (plans):
```sql
CREATE TABLE dbo.RateLimitPolicy (
    PolicyId INT IDENTITY PRIMARY KEY,
    PlanName VARCHAR(50) NOT NULL,
    Route VARCHAR(200) NOT NULL,
    PermitLimit INT NOT NULL,
    WindowSeconds INT NOT NULL,
    BurstSize INT NOT NULL
);
```

### API design
Response headers: `429 Too Many Requests`, `Retry-After: 12`, plus `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` (conventional, not standard).

### Key .NET implementation
Built-in (single instance): `Microsoft.AspNetCore.RateLimiting` (.NET 7+).
```csharp
builder.Services.AddRateLimiter(o =>
{
    o.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
    o.AddPolicy("per-user", ctx =>
        RateLimitPartition.GetTokenBucketLimiter(
            ctx.User.Identity?.Name ?? ctx.Connection.RemoteIpAddress?.ToString() ?? "anon",
            _ => new TokenBucketRateLimiterOptions
            {
                TokenLimit = 100,
                TokensPerPeriod = 10,
                ReplenishmentPeriod = TimeSpan.FromSeconds(1),
                QueueLimit = 0
            }));
    o.OnRejected = (ctx, ct) =>
    {
        if (ctx.Lease.TryGetMetadata(MetadataName.RetryAfter, out var ra))
            ctx.HttpContext.Response.Headers.RetryAfter = ((int)ra.TotalSeconds).ToString();
        return ValueTask.CompletedTask;
    };
});
app.UseRateLimiter();
app.MapGet("/api/items", () => "ok").RequireRateLimiting("per-user");
```
These limiters are **in-process**, so with N pods the real limit is N × limit. For distributed, use Redis with an atomic Lua script (sliding-window counter / token bucket):
```csharp
private const string Lua = @"
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local current = redis.call('INCR', key)
if current == 1 then redis.call('EXPIRE', key, window) end
if current > limit then return 0 end
return 1";

public async Task<bool> AllowAsync(string clientId, int limit, int windowSec)
{
    var db = _mux.GetDatabase();
    var bucket = DateTimeOffset.UtcNow.ToUnixTimeSeconds() / windowSec;
    var res = (int)await db.ScriptEvaluateAsync(Lua,
        new RedisKey[] { $"rl:{clientId}:{bucket}" },
        new RedisValue[] { limit, windowSec });
    return res == 1;
}
```
Lua guarantees INCR+EXPIRE atomicity (no key without TTL if crash between commands).

### Scaling & bottlenecks
- Redis is the hot spot: use Redis Cluster, key hash tags, local pre-check cache; allow small over-limit by syncing counters periodically (approximate limiter) for extreme scale.
- Enforce at edge (API Management/Front Door/YARP) so bad traffic never reaches app.

### Failure modes
- Redis down: **fail-open** for availability-critical APIs (log + alert), **fail-closed** for expensive/abuse-prone ones. Fall back to local in-memory limiter.
- Clock skew across nodes: use Redis time or windowing computed in Lua (`TIME`).
- Fixed-window boundary burst (2× limit across a boundary) → use sliding window/token bucket.
- Many users behind one NAT IP → prefer authenticated identity.

### Trade-offs
| Algorithm | Pros | Cons |
|---|---|---|
| Fixed window | trivial, cheap | boundary burst |
| Sliding window log | exact | memory heavy |
| Sliding window counter | cheap, near-exact | approximate |
| Token bucket | allows bursts, smooth | two values of state |
| Leaky bucket | smooth output | no bursts |

### Follow-up questions
1. Fixed vs sliding window? 2. Where to place the limiter (gateway vs app)? 3. What headers do you return? 4. What if Redis is down? 5. How to make INCR+EXPIRE atomic? (Lua/transactions.) 6. Per-tenant plans with changing limits live? (config cached, pub/sub invalidation.) 7. How to rate-limit across regions? (per-region quotas, async reconciliation.) 8. How to limit concurrency rather than rate? (`ConcurrencyLimiter`.)

---

## Design 3: Order/Checkout System with Inventory & Payments

### Requirements & clarifying questions
- Add to cart, place order, reserve inventory, charge payment, confirm; cancel/refund.
- Ask: can we oversell? (No.) Is payment synchronous (card) or redirect (3DS/wallet)? Single warehouse or many? Peak (sales events)? Partial shipments? Currency/tax?
- Core hard problem: **distributed consistency across Orders, Inventory, Payment** without 2PC.

### Capacity estimation
- 1 M orders/day ≈ 12/s avg; sale peak 100×≈1.2 k/s. Order + lines ≈ 3 KB → 1 M × 3 KB ≈ 3 GB/day, ~1 TB/year. Partition/archive old orders.

### High-level architecture
```
Client ─► API Gateway ─► Order API ──► SQL (Orders, Outbox) 
                              │               │ outbox relay
                              ▼               ▼
                        Service Bus  ──► Inventory Svc (reserve) ──► SQL Inventory
                              │      ──► Payment Svc (charge via PSP) ◄── PSP webhook
                              ▼
                     Order Saga / State machine (Orchestrator)
                              └─► Shipping / Notification
```
Saga: Order Created → Inventory Reserved → Payment Authorized → Order Confirmed. Failure → compensate (release stock, void payment).

### Data model
```sql
CREATE TABLE dbo.Orders (
    OrderId UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    CustomerId INT NOT NULL,
    Status TINYINT NOT NULL,  -- 0 Pending,1 Reserved,2 Paid,3 Confirmed,4 Cancelled,5 Failed
    TotalAmount DECIMAL(18,2) NOT NULL,
    Currency CHAR(3) NOT NULL,
    IdempotencyKey VARCHAR(64) NOT NULL,
    CreatedUtc DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    RowVersion ROWVERSION
);
CREATE UNIQUE INDEX UX_Orders_Idem ON dbo.Orders(CustomerId, IdempotencyKey);

CREATE TABLE dbo.OrderLines (
    OrderId UNIQUEIDENTIFIER NOT NULL REFERENCES dbo.Orders(OrderId),
    LineNo INT NOT NULL, Sku VARCHAR(40) NOT NULL, Qty INT NOT NULL, UnitPrice DECIMAL(18,2) NOT NULL,
    PRIMARY KEY (OrderId, LineNo)
);
CREATE TABLE dbo.Inventory (
    Sku VARCHAR(40) PRIMARY KEY,
    OnHand INT NOT NULL CHECK (OnHand >= 0),
    Reserved INT NOT NULL CHECK (Reserved >= 0),
    CONSTRAINT CK_Inv CHECK (Reserved <= OnHand)
);
CREATE TABLE dbo.StockReservation (
    OrderId UNIQUEIDENTIFIER, Sku VARCHAR(40), Qty INT, ExpiresUtc DATETIME2,
    PRIMARY KEY (OrderId, Sku)
);
CREATE TABLE dbo.Outbox (
    Id BIGINT IDENTITY PRIMARY KEY, Type VARCHAR(100), Payload NVARCHAR(MAX),
    CreatedUtc DATETIME2 DEFAULT SYSUTCDATETIME(), ProcessedUtc DATETIME2 NULL
);
```

### API design
```
POST /api/orders           Header: Idempotency-Key: <guid>   → 202 { orderId, status:"Pending" }
GET  /api/orders/{id}      → status
POST /api/orders/{id}/cancel
POST /webhooks/psp         (signature verified, idempotent)
```

### Key .NET implementation
Atomic reservation without oversell — conditional UPDATE (no read-then-write race):
```csharp
var rows = await db.Database.ExecuteSqlInterpolatedAsync($@"
    UPDATE dbo.Inventory
       SET Reserved = Reserved + {qty}
     WHERE Sku = {sku} AND OnHand - Reserved >= {qty}");
if (rows == 0) throw new OutOfStockException(sku);
```
Transactional outbox — order and event commit atomically:
```csharp
await using var tx = await db.Database.BeginTransactionAsync();
db.Orders.Add(order);
db.Outbox.Add(new OutboxMessage { Type = nameof(OrderCreated), Payload = JsonSerializer.Serialize(new OrderCreated(order.OrderId)) });
await db.SaveChangesAsync();
await tx.CommitAsync();
```
Outbox relay (BackgroundService) polls unprocessed rows (use `WITH (UPDLOCK, READPAST)` for multi-instance), publishes, marks processed. Consumers must be idempotent (dedupe by MessageId).

Payment: authorize first, capture on shipment (or capture straight away). Always send PSP an idempotency key = OrderId. Treat PSP webhook as source of truth for async outcomes.

### Scaling & bottlenecks
- Hot SKUs: row-level contention on Inventory → split stock into buckets (sku shards) or use Redis atomic DECR with reconciliation (see Design 16).
- Partition by CustomerId/date; read replicas for order history; CQRS read model.
- Reservation expiry job releases stock for abandoned checkouts.

### Failure modes
- Payment succeeded but order update failed → reconciliation job comparing PSP vs DB.
- Duplicate submit → unique idempotency key.
- Compensation fails → retries + dead-letter + manual queue.
- Reservation leak → TTL sweeper.
- PSP timeout (unknown outcome) → query PSP by idempotency key, don't blindly retry charge with a new key.

### Trade-offs
- Orchestrated saga (clear flow, central point) vs choreographed (loose coupling, hard to follow).
- Reserve-then-pay (no oversell, holds stock) vs pay-then-reserve (may refund).
- Strong consistency inside a service; eventual between services.
- Monolith with one DB transaction is **far simpler** if scale allows — say so.

### Follow-up questions
1. Why not oversell via `SELECT` then `UPDATE`? 2. What is the outbox pattern and why? 3. How do you make payment idempotent? 4. Why not distributed transactions/MSDTC? 5. Saga compensation example? 6. Hot SKU contention? 7. How to reconcile with PSP? 8. How do you guarantee exactly-once effect? (at-least-once + idempotent consumers.) 9. How do you migrate from monolith to this?

---

## Design 4: Notification Service (Email/SMS/Push)

### Requirements & clarifying questions
- Send transactional + marketing messages over email, SMS, push; templates; user preferences/opt-out; retries; delivery status.
- Ask: volume/peak? priority classes (OTP vs newsletter)? Scheduling? Per-user rate caps? Compliance (unsubscribe, quiet hours)? Which providers (SendGrid, Twilio, FCM/APNs)?

### Capacity estimation
- 50 M messages/day ≈ 580/s avg; campaign burst 10 k/s. Provider limits dictate throughput, so queueing is essential.

### High-level architecture
```
Producers ─► Notification API ─► validate + prefs check ─► Priority Queues (otp / transactional / bulk)
                                                              │
                                       ┌──────────────────────┼───────────────────┐
                                  Email worker           SMS worker          Push worker
                                       │                      │                   │
                                  SendGrid/SES            Twilio             FCM/APNs
                                       └── delivery webhooks ─► Status updater ─► SQL
```

### Data model
```sql
CREATE TABLE dbo.Notification (
    NotificationId BIGINT IDENTITY PRIMARY KEY,
    UserId INT NOT NULL, Channel TINYINT NOT NULL, TemplateKey VARCHAR(100) NOT NULL,
    Payload NVARCHAR(MAX) NOT NULL, DedupeKey VARCHAR(100) NULL,
    Status TINYINT NOT NULL,  -- Queued, Sent, Delivered, Failed, Suppressed
    Attempts INT NOT NULL DEFAULT 0, NextAttemptUtc DATETIME2 NULL,
    ProviderMessageId VARCHAR(100) NULL, CreatedUtc DATETIME2 DEFAULT SYSUTCDATETIME()
);
CREATE UNIQUE INDEX UX_Notif_Dedupe ON dbo.Notification(UserId, DedupeKey) WHERE DedupeKey IS NOT NULL;
CREATE TABLE dbo.UserChannelPref (UserId INT, Channel TINYINT, OptedIn BIT, QuietStart TIME NULL, QuietEnd TIME NULL, PRIMARY KEY(UserId, Channel));
CREATE TABLE dbo.Template (TemplateKey VARCHAR(100), Channel TINYINT, Locale VARCHAR(10), Body NVARCHAR(MAX), PRIMARY KEY(TemplateKey, Channel, Locale));
```

### API design
```
POST /api/notifications  { userId, channel[], templateKey, data, dedupeKey, priority, sendAfterUtc } → 202
GET  /api/notifications/{id}
POST /webhooks/sendgrid  (delivery/bounce/complaint)
```

### Key .NET implementation
```csharp
public sealed class EmailWorker : BackgroundService
{
    private readonly ServiceBusProcessor _processor;
    public EmailWorker(ServiceBusClient client) =>
        _processor = client.CreateProcessor("email", new ServiceBusProcessorOptions { MaxConcurrentCalls = 16, AutoCompleteMessages = false });

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        _processor.ProcessMessageAsync += async args =>
        {
            var msg = JsonSerializer.Deserialize<EmailJob>(args.Message.Body)!;
            if (await _store.AlreadySentAsync(msg.NotificationId)) { await args.CompleteMessageAsync(args.Message); return; }
            await _provider.SendAsync(msg, args.CancellationToken);   // wrapped in Polly retry + circuit breaker
            await _store.MarkSentAsync(msg.NotificationId);
            await args.CompleteMessageAsync(args.Message);
        };
        _processor.ProcessErrorAsync += args => { _log.LogError(args.Exception, "SB error"); return Task.CompletedTask; };
        await _processor.StartProcessingAsync(ct);
    }
}
```
Service Bus handles retries via delivery count → dead-letter queue after `MaxDeliveryCount`. Per-provider rate limiting with `System.Threading.RateLimiting`.

### Scaling & bottlenecks
- Separate queues per channel and priority so a marketing blast never delays OTP.
- Provider throttling → token bucket per provider; multiple providers for failover.
- Fan-out of campaigns: chunk recipients into batches of ~1000 messages.

### Failure modes
- Provider outage → circuit breaker, route to secondary provider.
- Duplicate sends on retry → dedupe key + status check (still at-least-once; accept rare dup for non-OTP).
- Poison message → DLQ with alert.
- Bounces/complaints hurting sender reputation → suppression list from webhooks.

### Trade-offs
- At-least-once (possible duplicates) vs at-most-once (possible loss). OTP prefers send-again; marketing prefers no-dup.
- Template rendering at enqueue time (immutable content) vs send time (fresh data).
- Build vs buy: use provider for delivery, own the orchestration/preferences.

### Follow-up questions
1. Why a queue? 2. How do you prioritize OTP? 3. How do you prevent duplicate emails? 4. How to handle provider failure? 5. How to honour unsubscribe/quiet hours? 6. How to track delivery status? 7. How to schedule a send for 9am in user's time zone at 10 M users? 8. Backpressure when provider is slow?

---

## Design 5: File Upload / Processing Pipeline

### Requirements & clarifying questions
- Upload files up to GBs, virus scan, extract metadata, generate thumbnails/transcode, notify.
- Ask: max size? allowed types? resumable? Public vs private download? Retention? Processing SLA?

### Capacity estimation
- 100 k uploads/day, avg 5 MB → 500 GB/day, ~180 TB/year → object storage (Blob), not DB.

### High-level architecture
```
Client ──(1) request SAS URL──► API ──► SQL (FileRecord: Pending)
Client ──(2) PUT blocks directly──► Blob Storage (quarantine container)
Blob event (Event Grid) ──► Queue ──► Scanner worker ──► clean? move to "processed" container
                                         └─► Processor workers (thumbnail/OCR) ──► SQL status=Ready ──► SignalR/notification
```
Bypass the API for bytes: API never streams big files.

### Data model
```sql
CREATE TABLE dbo.FileRecord (
    FileId UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    OwnerId INT NOT NULL, FileName NVARCHAR(260) NOT NULL, ContentType VARCHAR(100),
    SizeBytes BIGINT NULL, Sha256 BINARY(32) NULL, BlobPath VARCHAR(500) NOT NULL,
    Status TINYINT NOT NULL, -- Pending, Uploaded, Scanning, Infected, Processing, Ready, Failed
    CreatedUtc DATETIME2 DEFAULT SYSUTCDATETIME()
);
CREATE INDEX IX_File_Owner ON dbo.FileRecord(OwnerId, CreatedUtc DESC);
```

### API design
```
POST /api/files            { fileName, contentType, size } → 201 { fileId, uploadUrl (SAS, 15 min, write-only) }
POST /api/files/{id}/complete   → validates blob exists, size match
GET  /api/files/{id}       → status, download URL (read-only SAS, short-lived)
```

### Key .NET implementation
```csharp
// Azure.Storage.Blobs
var container = _blobService.GetBlobContainerClient("quarantine");
var blob = container.GetBlobClient($"{ownerId}/{fileId}");
var sas = blob.GenerateSasUri(
    new BlobSasBuilder(BlobSasPermissions.Create | BlobSasPermissions.Write, DateTimeOffset.UtcNow.AddMinutes(15))
    { BlobContainerName = container.Name, BlobName = blob.Name, Resource = "b" });
```
(Requires the client be created with a shared key credential; with Entra ID use a user delegation key via `GetUserDelegationKeyAsync`.)

If the API must receive the file (small files): stream, don't buffer.
```csharp
app.MapPost("/upload", async (HttpRequest req) =>
{
    var reader = new MultipartReader(MediaTypeHeaderValue.Parse(req.ContentType).Boundary.Value!, req.Body);
    MultipartSection? s;
    while ((s = await reader.ReadNextSectionAsync()) != null)
    {
        var cd = s.GetContentDispositionHeader();
        if (cd?.IsFileDisposition() == true)
            await blobClient.UploadAsync(s.Body, overwrite: false);
    }
}).DisableAntiforgery();
```
Set `[RequestSizeLimit]` / Kestrel `MaxRequestBodySize`. Validate content by magic bytes, not just extension. Generate server-side blob names — never use user file name as path (path traversal).

### Scaling & bottlenecks
- Workers autoscale on queue length (KEDA / Azure Functions).
- Large files: block upload with parallel chunks (`BlobClient.UploadAsync` with `StorageTransferOptions`).
- CDN for downloads.

### Failure modes
- Client abandons upload → FileRecord stuck Pending → cleanup job deletes after 24 h; blob lifecycle rule on quarantine.
- Worker crash mid-process → message reappears (visibility timeout) → processing must be idempotent (deterministic output path).
- Infected file → status Infected, delete, notify.
- Poison files (corrupt video) → DLQ after N tries.

### Trade-offs
- Direct-to-blob (scalable, more client logic) vs proxy through API (simple, bottleneck).
- Sync processing (simple UX) vs async (scales, needs status/notification).
- DB blob (FILESTREAM) vs object storage: object storage cheaper and scales.

### Follow-up questions
1. Why SAS URLs? 2. How to validate file type? 3. Resumable uploads? (block blobs, tus.) 4. Virus scanning flow? 5. How to prevent users reading others' files? 6. Process same file twice? 7. Processing 4 GB video — timeouts? 8. Cost control (tiering, lifecycle)?

---

## Design 6: Real-time Chat with SignalR

### Requirements & clarifying questions
- 1:1 and group chat, presence, typing indicators, message history, read receipts, offline delivery.
- Ask: group size limit? message ordering guarantee? retention? attachments? E2E encryption? Concurrent connections target?

### Capacity estimation
- 500 k concurrent connections; 50 k msgs/s peak. One Kestrel server ~ 20–50 k connections depending on activity → ~15–25 servers. Messages 200 B × 4 B/day = 800 GB/day → partition by conversation, archive.

### High-level architecture
```
Clients ◄═ WebSocket ═► LB (sticky not required with WebSockets after negotiate) ─► SignalR Hubs (N pods)
                                                          │ backplane: Redis / Azure SignalR Service
                                                          ▼
                                   Chat service ─► SQL (Messages) + Redis (presence, unread counts)
                                          └─► Notification svc (push for offline users)
```

### Data model
```sql
CREATE TABLE dbo.Conversation (ConversationId BIGINT IDENTITY PRIMARY KEY, Type TINYINT, CreatedUtc DATETIME2);
CREATE TABLE dbo.ConversationMember (ConversationId BIGINT, UserId INT, LastReadMessageId BIGINT NULL, PRIMARY KEY(ConversationId, UserId));
CREATE INDEX IX_Member_User ON dbo.ConversationMember(UserId) INCLUDE(ConversationId);
CREATE TABLE dbo.Message (
    ConversationId BIGINT NOT NULL,
    MessageId BIGINT NOT NULL,          -- monotonically increasing per conversation (or global IDENTITY)
    SenderId INT NOT NULL, Body NVARCHAR(4000) NOT NULL, ClientMsgId UNIQUEIDENTIFIER NOT NULL,
    SentUtc DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    PRIMARY KEY (ConversationId, MessageId)
);
CREATE UNIQUE INDEX UX_Msg_Client ON dbo.Message(ConversationId, ClientMsgId);  -- dedupe retries
```

### API design
Hub methods + REST for history:
```
Hub: SendMessage(conversationId, clientMsgId, text) ; Typing(conversationId) ; MarkRead(conversationId, messageId)
Server→Client: ReceiveMessage(msg), UserTyping, PresenceChanged
GET /api/conversations/{id}/messages?beforeId=123&limit=50   (keyset pagination)
```

### Key .NET implementation
```csharp
[Authorize]
public class ChatHub : Hub
{
    private readonly IChatService _chat;
    public ChatHub(IChatService chat) => _chat = chat;

    public override async Task OnConnectedAsync()
    {
        var userId = Context.UserIdentifier!;               // from IUserIdProvider / NameIdentifier claim
        foreach (var convId in await _chat.GetConversationIdsAsync(userId))
            await Groups.AddToGroupAsync(Context.ConnectionId, $"conv:{convId}");
        await base.OnConnectedAsync();
    }

    public async Task SendMessage(long conversationId, Guid clientMsgId, string text)
    {
        var msg = await _chat.SaveAsync(conversationId, Context.UserIdentifier!, clientMsgId, text); // authz + persist first
        await Clients.Group($"conv:{conversationId}").SendAsync("ReceiveMessage", msg);
    }
}

builder.Services.AddSignalR().AddStackExchangeRedis(redisConn);   // backplane for scale-out
// or: .AddAzureSignalR(conn)
app.MapHub<ChatHub>("/hubs/chat");
```
Persist before broadcast so a reconnecting client can fetch missed messages via `lastSeenMessageId`. Hub instances are transient: never keep state in hub fields. Don't use `Context.ConnectionId` as identity.

### Scaling & bottlenecks
- Connections are stateful: need Redis backplane or Azure SignalR Service (offloads connections). Backplane fan-out cost grows with message rate.
- Big groups (10 k members): broadcast cost → limit group size, paginate presence.
- Hot conversations: ordering via per-conversation sequence.
- WebSocket-only (`SkipNegotiation`) removes need for sticky sessions with LB.

### Failure modes
- Server restart → clients auto-reconnect (`withAutomaticReconnect()`) and fetch gap via REST.
- Duplicate sends on reconnect → `ClientMsgId` unique index.
- Backplane down → messages don't cross servers; degrade to local, alert.
- Slow client → SignalR buffers; configure `MaximumParallelInvocationsPerClient`, `ApplicationMaxBufferSize`.

### Trade-offs
- SignalR/WebSockets (full duplex) vs SSE (server→client only, simpler) vs long polling (fallback).
- Azure SignalR Service (managed, cost) vs self-hosted Redis backplane (control, ops).
- Messages in SQL (simple, relational) vs Cassandra/Cosmos (write-heavy scale).

### Follow-up questions
1. WebSockets vs polling? 2. How do multiple servers deliver to same group? (backplane.) 3. Message ordering? 4. Offline users? 5. How to authenticate SignalR (token in query string `access_token`)? 6. Presence with flaky connections? (heartbeat + TTL in Redis.) 7. Exactly-once delivery? (client IDs + ack.) 8. 1 M connections cost/design?

---

## Design 7: Job Scheduler

### Requirements & clarifying questions
- Run one-off and recurring (cron) jobs, retries, delays, cancellation, visibility into history; jobs must not run twice at once.
- Ask: precision (second vs minute)? Job duration? Multi-instance deployment? Priorities? Exactly-once vs at-least-once? Existing tools (Hangfire/Quartz) acceptable?

### Capacity estimation
- 1 M scheduled jobs; 2 k executions/s peak. Polling table must be indexed on `NextRunUtc`.

### High-level architecture
```
API/UI ─► SQL (Job, JobRun) ◄── Scheduler loop (leader or lock-based) ─ finds due jobs ─► Queue
                                                                                     │
                                                                  Worker pool (N) ◄──┘ executes handler, writes JobRun
```

### Data model
```sql
CREATE TABLE dbo.Job (
    JobId BIGINT IDENTITY PRIMARY KEY, Name VARCHAR(100), Type VARCHAR(100), Payload NVARCHAR(MAX),
    CronExpr VARCHAR(100) NULL, NextRunUtc DATETIME2 NULL, Status TINYINT NOT NULL, -- Scheduled, Running, Done, Failed
    LockedBy VARCHAR(100) NULL, LockedUntilUtc DATETIME2 NULL, Attempts INT NOT NULL DEFAULT 0, MaxAttempts INT NOT NULL DEFAULT 5
);
CREATE INDEX IX_Job_Due ON dbo.Job(NextRunUtc) INCLUDE (Status) WHERE Status = 0;  -- filtered index
CREATE TABLE dbo.JobRun (RunId BIGINT IDENTITY PRIMARY KEY, JobId BIGINT, StartedUtc DATETIME2, EndedUtc DATETIME2 NULL, Outcome TINYINT, Error NVARCHAR(MAX));
```

### API design
```
POST /api/jobs {type, payload, runAtUtc | cron}   GET /api/jobs/{id}   DELETE /api/jobs/{id}   POST /api/jobs/{id}/trigger
```

### Key .NET implementation
Claim jobs safely across many workers — `UPDLOCK, READPAST` queue pattern:
```sql
WITH cte AS (
    SELECT TOP (10) *
    FROM dbo.Job WITH (UPDLOCK, READPAST, ROWLOCK)
    WHERE Status = 0 AND NextRunUtc <= SYSUTCDATETIME()
    ORDER BY NextRunUtc
)
UPDATE cte SET Status = 1, LockedBy = @worker, LockedUntilUtc = DATEADD(MINUTE, 5, SYSUTCDATETIME())
OUTPUT inserted.*;
```
```csharp
public class SchedulerWorker : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromSeconds(1));
        while (await timer.WaitForNextTickAsync(ct))
        {
            var jobs = await _repo.ClaimDueJobsAsync(_workerId, 10, ct);
            await Parallel.ForEachAsync(jobs, new ParallelOptions { MaxDegreeOfParallelism = 4, CancellationToken = ct },
                async (job, token) => await RunOneAsync(job, token));
        }
    }
}
```
Recurring jobs: after run, compute next occurrence (Cronos library `CronExpression.Parse(expr).GetNextOccurrence(DateTime.UtcNow)`), set `NextRunUtc`. Lease expiry: a reaper resets jobs whose `LockedUntilUtc` passed (crashed worker). Long jobs renew the lease (heartbeat).

Alternative: **Hangfire** (SQL storage, distributed locks) or **Quartz.NET** (clustered mode with AdoJobStore) — mention that you'd use these first rather than reinventing.

### Scaling & bottlenecks
- Polling hot-spot: filtered index, small batches, jitter. Shard jobs by hash across scheduler partitions.
- Thousands of recurring jobs at same minute (thundering herd) → jitter.

### Failure modes
- Worker dies → lease expires → job re-run (so handlers idempotent).
- Clock skew → use DB time (`SYSUTCDATETIME()`), not worker clock.
- Missed runs after downtime → misfire policy (run once / skip / catch-up all).
- Poison job → MaxAttempts then Failed + alert; exponential backoff.
- DST → store UTC, evaluate cron in specified TimeZoneInfo.

### Trade-offs
- DB polling (simple, reliable, ~1 s precision) vs broker with delayed messages (scale) vs in-memory timers (fast, lost on restart).
- At-least-once + idempotency vs distributed lock "exactly once" (lock expiry still allows overlap).

### Follow-up questions
1. How do you stop two instances running the same job? 2. Crashed worker? 3. Hangfire vs Quartz vs custom? 4. Cron + time zones/DST? 5. Retries/backoff? 6. Millions of delayed jobs? 7. Fairness / priority? 8. Graceful shutdown (`IHostApplicationLifetime`, `StopAsync` waits for current jobs)?

---

## Design 8: Multi-tenant SaaS

### Requirements & clarifying questions
- Many customers (tenants) on one product; strict data isolation; per-tenant config, branding, limits; noisy-neighbour control.
- Ask: how many tenants (100 vs 100 k)? Size variance? Compliance/data residency? Per-tenant customization? Enterprise tenants wanting dedicated DB?

### Capacity estimation
- 5 k tenants, avg 10 GB, a few 1 TB whales. Total 50+ TB → single DB won't cut it long-term.

### High-level architecture
```
Request ─► Gateway ─► Tenant resolution (subdomain / header / JWT claim "tid") ─► App (scoped TenantContext)
                                                           │
        Catalog DB (tenant → connection/shard) ◄───────────┘
                    ├── Shared DB (pooled, TenantId column + RLS) for small tenants
                    └── Dedicated DBs for large tenants (DB-per-tenant)
```

### Data model
Three models: (1) shared schema + `TenantId`; (2) schema-per-tenant; (3) DB-per-tenant. Hybrid is common.
```sql
CREATE TABLE dbo.Tenant (TenantId INT PRIMARY KEY, Name NVARCHAR(200), Subdomain VARCHAR(63) UNIQUE, ConnectionString NVARCHAR(500) NULL, Plan VARCHAR(20));

CREATE TABLE dbo.Invoice (
    TenantId INT NOT NULL, InvoiceId BIGINT NOT NULL, Amount DECIMAL(18,2),
    PRIMARY KEY (TenantId, InvoiceId)       -- TenantId leading column in every key/index
);

-- Row-Level Security as defence in depth
CREATE FUNCTION Security.fn_TenantFilter(@TenantId INT) RETURNS TABLE WITH SCHEMABINDING AS
RETURN SELECT 1 AS ok WHERE @TenantId = CAST(SESSION_CONTEXT(N'TenantId') AS INT);
CREATE SECURITY POLICY Security.TenantPolicy
    ADD FILTER PREDICATE Security.fn_TenantFilter(TenantId) ON dbo.Invoice,
    ADD BLOCK PREDICATE  Security.fn_TenantFilter(TenantId) ON dbo.Invoice
WITH (STATE = ON);
```

### API design
Tenant is never taken from request body; it comes from the authenticated token/host. `GET https://acme.app.com/api/invoices`.

### Key .NET implementation
```csharp
public interface ITenantContext { int TenantId { get; } }

public class TenantMiddleware
{
    private readonly RequestDelegate _next;
    public TenantMiddleware(RequestDelegate next) => _next = next;
    public async Task InvokeAsync(HttpContext ctx, ITenantStore store, TenantContext tc)
    {
        var host = ctx.Request.Host.Host.Split('.')[0];
        var tenant = await store.FindBySubdomainAsync(host) ?? throw new BadHttpRequestException("Unknown tenant");
        tc.TenantId = tenant.TenantId;
        await _next(ctx);
    }
}

public class AppDb : DbContext
{
    private readonly ITenantContext _tenant;
    public AppDb(DbContextOptions<AppDb> o, ITenantContext tenant) : base(o) => _tenant = tenant;

    protected override void OnModelCreating(ModelBuilder mb) =>
        mb.Entity<Invoice>().HasQueryFilter(i => i.TenantId == _tenant.TenantId);   // global query filter

    public override Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        foreach (var e in ChangeTracker.Entries<ITenantOwned>().Where(e => e.State == EntityState.Added))
            e.Entity.TenantId = _tenant.TenantId;
        return base.SaveChangesAsync(ct);
    }
}
```
The filter references `_tenant` field of the context, so EF evaluates it per-instance (works with context pooling only if handled carefully — pooled contexts must not hold per-request state unless reset). For DB-per-tenant, choose connection string per request via `DbContextOptionsBuilder.UseSqlServer(...)` inside a scoped factory. Remember `IgnoreQueryFilters()` bypasses isolation — audit its use. Raw SQL does not get the filter.

For RLS: set `sp_set_session_context 'TenantId', @id` on connection open (interceptor `DbConnectionInterceptor.ConnectionOpenedAsync`).

### Scaling & bottlenecks
- Noisy neighbour: per-tenant rate limits, resource governor, queue-per-tenant fairness, move whales to dedicated DB.
- Shared DB: statistics skew (one big tenant skews plans) → parameter sniffing issues; consider `OPTIMIZE FOR UNKNOWN` or dedicated DB.
- Schema migrations across thousands of DBs → rolling, automated, backward-compatible.
- Elastic pools (Azure SQL) for many small DBs.

### Failure modes
- Missing tenant filter → data leak (the worst bug). Mitigate: global query filters + RLS + tests that assert cross-tenant access fails.
- Cache key lacking tenant id → cross-tenant cache leak. Always prefix.
- Background jobs lacking tenant context.
- One tenant's migration fails → partial rollout; track per-tenant schema version.

### Trade-offs
| Model | Isolation | Cost | Ops |
|---|---|---|---|
| Shared schema | low (logical) | lowest | easy migrations |
| Schema/tenant | medium | medium | many objects |
| DB/tenant | high | highest | migration fan-out |

### Follow-up questions
1. How do you identify the tenant? 2. How to guarantee no leaks? 3. Global query filters pitfalls? 4. Tenant-specific customization/fields (JSON column / EAV / extension tables)? 5. Migrations at 5 k DBs? 6. Noisy neighbours? 7. Per-tenant backup/restore? 8. Tenant onboarding/offboarding (data deletion, GDPR)?

---

## Design 9: Audit Log System

### Requirements & clarifying questions
- Record who did what, when, to which entity, with before/after values; tamper-evident; searchable; retained 7 years.
- Ask: what must be audited (all writes? reads?)? Retention/legal hold? Must it be immutable? Query patterns? Sync (must not lose) or async OK?

### Capacity estimation
- 20 M events/day ≈ 230/s avg, 2 k/s peak; ~1 KB each → 20 GB/day, 7 TB/year → partition + compress + archive to cold storage.

### High-level architecture
```
App (EF SaveChanges interceptor) ─► Outbox in same tx ─► relay ─► Event Hub/Service Bus ─► Audit writer ─► Audit DB (partitioned, columnstore)
                                                                                       └─► Blob (immutable WORM archive)
Query API ─► Audit DB / search index
```

### Data model
```sql
CREATE PARTITION FUNCTION pf_AuditMonth (DATETIME2) AS RANGE RIGHT FOR VALUES ('2026-01-01','2026-02-01' /* ... */);
CREATE PARTITION SCHEME ps_AuditMonth AS PARTITION pf_AuditMonth ALL TO ([PRIMARY]);

CREATE TABLE dbo.AuditLog (
    AuditId BIGINT IDENTITY NOT NULL,
    OccurredUtc DATETIME2 NOT NULL,
    TenantId INT NULL, UserId INT NULL, Action VARCHAR(50) NOT NULL,
    EntityType VARCHAR(100) NOT NULL, EntityId VARCHAR(100) NOT NULL,
    OldValues NVARCHAR(MAX) NULL, NewValues NVARCHAR(MAX) NULL,
    CorrelationId UNIQUEIDENTIFIER NULL, IpAddress VARCHAR(45) NULL,
    PrevHash BINARY(32) NULL, Hash BINARY(32) NULL,        -- hash chain for tamper evidence
    CONSTRAINT PK_Audit PRIMARY KEY (OccurredUtc, AuditId)
) ON ps_AuditMonth(OccurredUtc);
CREATE INDEX IX_Audit_Entity ON dbo.AuditLog(EntityType, EntityId, OccurredUtc) ON ps_AuditMonth(OccurredUtc);
-- App login: INSERT only; no UPDATE/DELETE permission.
```
Alternatives: SQL Server **Temporal tables** (system-versioned) give row history automatically (what, but not who), and **SQL Server Audit** for DB-level access auditing.

### API design
`GET /api/audit?entityType=Order&entityId=42&from=...&to=...&userId=...` (keyset paginated, admin only; reading audit is itself audited).

### Key .NET implementation
```csharp
public class AuditInterceptor : SaveChangesInterceptor
{
    private readonly ICurrentUser _user;
    public AuditInterceptor(ICurrentUser user) => _user = user;

    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData e, InterceptionResult<int> result, CancellationToken ct = default)
    {
        var ctx = e.Context!;
        var entries = ctx.ChangeTracker.Entries()
            .Where(x => x.Entity is not AuditLog && x.State is EntityState.Added or EntityState.Modified or EntityState.Deleted)
            .ToList();
        foreach (var en in entries)
        {
            ctx.Set<AuditLog>().Add(new AuditLog
            {
                OccurredUtc = DateTime.UtcNow, UserId = _user.Id, Action = en.State.ToString(),
                EntityType = en.Metadata.ClrType.Name,
                EntityId = string.Join(",", en.Properties.Where(p => p.Metadata.IsPrimaryKey()).Select(p => p.CurrentValue)),
                OldValues = en.State == EntityState.Added ? null :
                    JsonSerializer.Serialize(en.Properties.Where(p => p.IsModified || en.State == EntityState.Deleted)
                        .ToDictionary(p => p.Metadata.Name, p => p.OriginalValue)),
                NewValues = en.State == EntityState.Deleted ? null :
                    JsonSerializer.Serialize(en.Properties.Where(p => p.IsModified || en.State == EntityState.Added)
                        .ToDictionary(p => p.Metadata.Name, p => p.CurrentValue)),
            });
        }
        return base.SavingChangesAsync(e, result, ct);
    }
}
// registration: options.AddInterceptors(sp.GetRequiredService<AuditInterceptor>());
```
Caveat: for Added entities with DB-generated keys, `EntityId` isn't known at SavingChanges — capture in `SavedChanges` or use a temp-key strategy. Never log secrets/PII fields; mask them with an attribute.

### Scaling & bottlenecks
- Same-transaction audit writes double write cost on hot tables → use async via outbox for high volume.
- Partition by month; switch out old partitions (`ALTER TABLE ... SWITCH`) to archive instantly; clustered columnstore for old partitions (10× compression).
- JSON `OldValues/NewValues` bloat: store only changed columns.

### Failure modes
- Audit write fails → decide fail-closed (reject business op — for regulated) vs fail-open with outbox retry.
- Audit tampering by DBA → hash chain, WORM blob storage, separate DB/permissions.
- Missing context for background jobs ("system" user) → always set actor.
- Bulk updates (`ExecuteUpdate`) bypass the change tracker → no interceptor audit! Use triggers/temporal tables or manual audit for those.

### Trade-offs
- App-level interceptor (knows user/intent) vs DB triggers (catch everything, no user context) vs CDC/Temporal (low app impact, limited "who").
- Sync (guaranteed) vs async (fast, eventual).
- Row diff vs full snapshot (storage vs ease of reconstruction).

### Follow-up questions
1. Interceptor vs triggers vs temporal tables? 2. How do you get the user into the log? 3. Does `ExecuteUpdate` get audited? 4. How do you make logs immutable? 5. Retention + archive with partition switching? 6. GDPR "right to erasure" vs immutable audit? (pseudonymize/crypto-shred.) 7. Audit reads? 8. Query "show history of entity X" fast at 7 TB?

---

## Design 10: Search/Autocomplete

### Requirements & clarifying questions
- Full-text product search with filters/facets/ranking; autocomplete suggestions < 100 ms; typo tolerance.
- Ask: catalog size? update freshness (seconds vs hours)? languages? personalization? Is SQL FTS enough or do we need Elasticsearch/Azure AI Search?

### Capacity estimation
- 10 M products; 3 k search QPS peak; autocomplete 10× keystrokes → 30 k QPS (served from cache/trie).

### High-level architecture
```
Writes ─► SQL Server (source of truth) ─► Change feed (CDC/outbox) ─► Indexer ─► Elasticsearch / Azure AI Search
Search API ─► search index (query, facets)          Autocomplete API ─► Redis (prefix cache) ─► index suggester
Query logs ─► analytics job ─► popular-queries table ─► Redis
```

### Data model
SQL Server full-text option:
```sql
CREATE FULLTEXT CATALOG ftProducts;
CREATE FULLTEXT INDEX ON dbo.Product(Name LANGUAGE 1033, Description LANGUAGE 1033)
    KEY INDEX PK_Product ON ftProducts WITH CHANGE_TRACKING AUTO;

SELECT TOP 20 p.ProductId, p.Name, ft.RANK
FROM CONTAINSTABLE(dbo.Product, (Name, Description), @query) ft
JOIN dbo.Product p ON p.ProductId = ft.[KEY]
ORDER BY ft.RANK DESC;

CREATE TABLE dbo.PopularQuery (Prefix VARCHAR(50), Suggestion NVARCHAR(100), Score INT, PRIMARY KEY(Prefix, Score DESC, Suggestion));
```
Prefix autocomplete in SQL: `WHERE Name LIKE @prefix + '%'` can use an index seek (trailing wildcard only). `LIKE '%x%'` cannot (scan).

### API design
```
GET /api/search?q=wireless+mouse&category=electronics&minPrice=10&sort=relevance&page=1
GET /api/suggest?q=wire     → ["wireless mouse","wireless keyboard"]  (max 8, debounced 150 ms client-side)
```

### Key .NET implementation
Sanitize input for CONTAINS (user input must be converted to a valid FTS expression):
```csharp
var terms = q.Split(' ', StringSplitOptions.RemoveEmptyEntries)
             .Select(t => "\"" + t.Replace("\"", "") + "*\"");
var ftsQuery = string.Join(" AND ", terms);

var results = await db.Products
    .Where(p => EF.Functions.Contains(p.Name, ftsQuery))   // SqlServer provider: EF.Functions.Contains / FreeText
    .OrderBy(p => p.Name)
    .Take(20).ToListAsync();
```
Elasticsearch via `Elastic.Clients.Elasticsearch` (NEST is deprecated for new versions); keep queries in an `ISearchService` abstraction. Suggest endpoint with cache:
```csharp
app.MapGet("/api/suggest", async (string q, IDistributedCache c, ISuggestStore s) =>
{
    q = q.Trim().ToLowerInvariant();
    if (q.Length < 2) return Results.Ok(Array.Empty<string>());
    var cached = await c.GetStringAsync($"sg:{q}");
    if (cached != null) return Results.Text(cached, "application/json");
    var list = await s.TopAsync(q, 8);
    var json = JsonSerializer.Serialize(list);
    await c.SetStringAsync($"sg:{q}", json, new() { AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10) });
    return Results.Text(json, "application/json");
});
```

### Scaling & bottlenecks
- Dedicated search engine for relevance, facets, typo-tolerance (edge n-grams, fuzziness); SQL FTS lacks good ranking/facets at scale.
- Shard index by category/hash; replicas for read QPS.
- Reindex: blue/green index with alias swap.

### Failure modes
- Index lag/out-of-sync with DB → dual-write is dangerous; use outbox/CDC + periodic reconciliation.
- Search cluster down → degrade to basic SQL `LIKE` search or cached popular results.
- Deep pagination expensive → `search_after`/keyset.
- Unsanitized input breaks CONTAINS syntax → 500.

### Trade-offs
- SQL FTS (no extra infra, consistent) vs Elasticsearch/AI Search (features, scale, eventual consistency, extra ops).
- Trie in memory (fastest) vs Redis sorted sets vs index suggester.
- Personalization vs cacheability.

### Follow-up questions
1. Why `LIKE '%x%'` is slow? 2. FTS vs LIKE? 3. Autocomplete data structure (trie, prefix table)? 4. Keeping index in sync? 5. Typos? 6. Ranking (BM25 + business boosts)? 7. Zero-downtime reindex? 8. Debounce, cancel in-flight requests?

---

## Design 11: Reporting/Analytics Dashboard on SQL Server

### Requirements & clarifying questions
- Dashboards with aggregates over 500 M-row fact data; filters; export; must not slow OLTP.
- Ask: freshness (real-time vs hourly)? Concurrency? Ad hoc vs fixed reports? Row-level security? Retention?

### Capacity estimation
- Fact table 2 B rows × 60 B = 120 GB raw; columnstore compresses ~8–10× → ~15 GB. 200 concurrent dashboard users, p95 < 3 s.

### High-level architecture
```
OLTP SQL ─(CDC / Always On readable secondary / ETL)─► Reporting DB (star schema, columnstore)
                                                            │
                          Aggregation jobs ─► Summary tables / indexed views
API (read-only intent) ─► Redis cache (per filter set) ─► Reporting DB ─► Dashboard UI / Power BI
```

### Data model
```sql
CREATE TABLE dbo.FactSales (
    DateKey INT NOT NULL, ProductKey INT NOT NULL, StoreKey INT NOT NULL,
    Quantity INT NOT NULL, Revenue DECIMAL(18,2) NOT NULL
);
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactSales ON dbo.FactSales;

CREATE TABLE dbo.DailySalesSummary (
    DateKey INT, StoreKey INT, Revenue DECIMAL(18,2), Orders INT,
    PRIMARY KEY (DateKey, StoreKey)
);

-- Dashboard query hits summary first
SELECT d.[Month], SUM(s.Revenue) AS Revenue
FROM dbo.DailySalesSummary s JOIN dbo.DimDate d ON d.DateKey = s.DateKey
WHERE s.DateKey BETWEEN @from AND @to
GROUP BY d.[Month];
```
Indexed view option (needs `WITH SCHEMABINDING`, `COUNT_BIG(*)`, deterministic aggregates).

### API design
`GET /api/reports/sales?from=2026-01-01&to=2026-03-31&groupBy=month&store=12` → JSON; `POST /api/reports/export` → 202 + job id (async export to blob).

### Key .NET implementation
```csharp
// Read-only intent routes to Always On secondary
"Server=ag-listener;Database=Reporting;ApplicationIntent=ReadOnly;MultiSubnetFailover=True;..."

var data = await reportDb.DailySalesSummary
    .AsNoTracking()
    .Where(s => s.DateKey >= from && s.DateKey <= to)
    .GroupBy(s => s.StoreKey)
    .Select(g => new { Store = g.Key, Revenue = g.Sum(x => x.Revenue) })
    .ToListAsync(ct);
```
Large exports: stream with `IAsyncEnumerable`/`DbDataReader` to CSV on blob, never `ToList()` millions of rows. Use `CommandTimeout` wisely and cancellation tokens tied to `HttpContext.RequestAborted`.

### Scaling & bottlenecks
- Pre-aggregate; partition fact table by date and switch partitions for loads; batch-mode columnstore.
- Cache dashboard results keyed by (report, normalized filters) with short TTL.
- Isolation: read replica or snapshot isolation (`READ_COMMITTED_SNAPSHOT`) so reads don't block writers.
- Parameter sniffing on wide date ranges → `OPTION (RECOMPILE)` or separate plans per range.

### Failure modes
- Heavy report starves OLTP → separate server/Resource Governor workload group.
- Readable secondary lag → show "data as of" timestamp.
- ETL partial failure → idempotent load by partition/batch id; reconciliation counts.
- Timeouts for ad hoc "all time" queries → enforce max ranges, async export.

### Trade-offs
- Real-time on OLTP (fresh, risky) vs ETL warehouse (fast, delayed, more infra).
- Summary tables (fast, maintenance) vs indexed views (auto-maintained, restrictions + write overhead) vs columnstore only.
- Build in-house vs Power BI/Synapse/Fabric.

### Follow-up questions
1. Why not run reports on OLTP? 2. Rowstore vs columnstore? 3. Star schema? 4. How to keep summaries up to date (incremental by watermark)? 5. Readable secondary caveats? 6. Pagination for huge tables (keyset)? 7. Cache invalidation for dashboards? 8. Row-level security for report users?

---

## Design 12: Leaderboard

### Requirements & clarifying questions
- Show top N players, a player's rank, neighbours around a player; daily/weekly/all-time; real-time updates.
- Ask: users count? update rate? ties? rank exactness for everyone or just top? Time windows?

### Capacity estimation
- 10 M players; 5 k score updates/s; reads 20 k/s. Redis sorted set of 10 M members ≈ 1–1.5 GB. O(log N) per op.

### High-level architecture
```
Game/API ─► Score service ─► Redis ZSET (lb:global, lb:daily:2026-10-16) ─► reads (top N, rank)
                       └──► SQL (durable ScoreEvent log) ──► rebuild Redis on loss, long-term stats
```

### Data model
```sql
CREATE TABLE dbo.PlayerScore (PlayerId INT PRIMARY KEY, Score BIGINT NOT NULL, UpdatedUtc DATETIME2 NOT NULL);
CREATE INDEX IX_Score ON dbo.PlayerScore(Score DESC, UpdatedUtc ASC) INCLUDE (PlayerId);  -- tie-break by earliest
-- Rank in SQL (OK for ~1M rows, slow at 10M+):
SELECT COUNT(*) + 1 AS Rank FROM dbo.PlayerScore WHERE Score > @myScore;
```
Redis is primary for ranking; SQL for durability.

### API design
```
POST /api/leaderboard/scores {playerId, delta}      GET /api/leaderboard/top?n=100
GET  /api/leaderboard/rank/{playerId}               GET /api/leaderboard/around/{playerId}?range=5
```

### Key .NET implementation
```csharp
var db = _mux.GetDatabase();
await db.SortedSetIncrementAsync("lb:global", playerId, delta);              // ZINCRBY

var top = await db.SortedSetRangeByRankWithScoresAsync("lb:global", 0, 99, Order.Descending);  // ZREVRANGE
var rank = await db.SortedSetRankAsync("lb:global", playerId, Order.Descending);               // 0-based; add 1
var myRank = rank is null ? (long?)null : rank + 1;

var around = await db.SortedSetRangeByRankWithScoresAsync("lb:global",
    Math.Max(0, rank!.Value - 5), rank.Value + 5, Order.Descending);
```
Ties: Redis breaks ties lexicographically by member. To tie-break by time, encode score as `score * 1e6 + (maxTs - ts)` (careful with double precision: 53 bits).
Time windows: key per window (`lb:daily:yyyyMMdd`) with `KeyExpireAsync`.

### Scaling & bottlenecks
- Single ZSET is one shard/hot key: for 100 M+ users, shard by score range or user bucket, merge top-N from shards; or keep only top-K exactly and approximate rank for the rest (percentile buckets).
- Write burst: batch updates via pipeline/`CreateBatch()`.

### Failure modes
- Redis loses data (restart without persistence) → rebuild from SQL scores (script) — keep AOF on.
- Dual-write drift Redis vs SQL → write to SQL/event log first, async project to Redis; periodic reconcile.
- Hot key (global board) → read replicas + local cache of top 100 for 1–2 s.
- Cheating/duplicate score submissions → server-authoritative scores, idempotency.

### Trade-offs
- Redis ZSET (fast, memory, extra durability work) vs SQL indexed query (consistent, slow rank at scale).
- Exact rank vs approximate; real-time vs periodic snapshots.

### Follow-up questions
1. Why sorted set? complexity? 2. How do you get a user's rank in SQL and why slow? 3. Ties? 4. Weekly reset? 5. Redis crash? 6. 100 M users? 7. Friends-only leaderboard? 8. Anti-cheat?

---

## Design 13: Booking/Reservation System with Double-Booking Prevention

### Requirements & clarifying questions
- Reserve a resource (room/seat/doctor slot) for a time range; no overlaps; holds expire; cancellations.
- Ask: time ranges or fixed slots? Hold-then-pay flow? Concurrency level on same resource? Multi-resource bookings (all-or-nothing)?

### Capacity estimation
- 10 k resources, 1 M bookings/month — modest; the difficulty is **correctness under concurrency**, not scale.

### High-level architecture
```
Client ─► API ─► Booking service ─► SQL (Booking with exclusion logic, transactions)
                       │ hold (status=Held, ExpiresUtc)
                       └─► payment ─► confirm; Expiry sweeper (BackgroundService) releases stale holds
```

### Data model
Fixed slots → simplest: unique constraint kills double booking.
```sql
CREATE TABLE dbo.Slot (
    ResourceId INT NOT NULL, SlotStartUtc DATETIME2(0) NOT NULL,
    BookingId UNIQUEIDENTIFIER NULL, HoldExpiresUtc DATETIME2 NULL,
    PRIMARY KEY (ResourceId, SlotStartUtc)
);
-- Book atomically: succeeds only if free (or hold expired)
UPDATE dbo.Slot
   SET BookingId = @bookingId, HoldExpiresUtc = DATEADD(MINUTE, 10, SYSUTCDATETIME())
 WHERE ResourceId = @res AND SlotStartUtc = @start
   AND (BookingId IS NULL OR HoldExpiresUtc < SYSUTCDATETIME());
-- @@ROWCOUNT = 1 → won; 0 → someone else has it
```
Arbitrary ranges (SQL Server lacks exclusion constraints like PostgreSQL), so serialize per resource:
```sql
BEGIN TRAN;
EXEC sp_getapplock @Resource = @resKey, @LockMode = 'Exclusive', @LockOwner = 'Transaction', @LockTimeout = 5000;
IF NOT EXISTS (SELECT 1 FROM dbo.Booking WHERE ResourceId=@res AND Status IN (1,2)
               AND StartUtc < @end AND EndUtc > @start)       -- overlap test
    INSERT dbo.Booking(...) VALUES (...);
COMMIT;
```
Alternative: lock the parent `Resource` row with `UPDLOCK, HOLDLOCK`, or use `SERIALIZABLE` with range locks (deadlock-prone).
Overlap rule: `existing.Start < new.End AND existing.End > new.Start`.

### API design
```
POST /api/bookings   {resourceId, startUtc, endUtc}  Idempotency-Key → 201 | 409 Conflict (slot taken)
POST /api/bookings/{id}/confirm   DELETE /api/bookings/{id}
```

### Key .NET implementation
```csharp
public async Task<Booking> HoldAsync(int resourceId, DateTime start, DateTime end, CancellationToken ct)
{
    var strategy = _db.Database.CreateExecutionStrategy();   // required when EnableRetryOnFailure is on
    return await strategy.ExecuteAsync(async () =>
    {
        await using var tx = await _db.Database.BeginTransactionAsync(System.Data.IsolationLevel.ReadCommitted, ct);
        await _db.Database.ExecuteSqlInterpolatedAsync(
            $"EXEC sp_getapplock @Resource = {$"res:{resourceId}"}, @LockMode = 'Exclusive', @LockOwner = 'Transaction', @LockTimeout = 5000", ct);

        var clash = await _db.Bookings.AnyAsync(b => b.ResourceId == resourceId
            && (b.Status == BookingStatus.Confirmed || (b.Status == BookingStatus.Held && b.HoldExpiresUtc > DateTime.UtcNow))
            && b.StartUtc < end && b.EndUtc > start, ct);
        if (clash) throw new SlotTakenException();

        var booking = new Booking(resourceId, start, end) { Status = BookingStatus.Held, HoldExpiresUtc = DateTime.UtcNow.AddMinutes(10) };
        _db.Bookings.Add(booking);
        await _db.SaveChangesAsync(ct);
        await tx.CommitAsync(ct);
        return booking;
    });
}
```
Optimistic alternative: `RowVersion` concurrency token on a per-resource "version" row, catch `DbUpdateConcurrencyException` and retry.

### Scaling & bottlenecks
- Contention only per resource → lock granularity per resource (applock key) scales horizontally; popular resource (concert) → Design 16.
- Shard by ResourceId.

### Failure modes
- App crashes after hold → sweeper/expiry check handles it (expiry evaluated at read time, not only by job).
- Applock timeout → return 503/409 retry.
- Time zones/DST → store UTC; slot generation in local zone.
- Double click/retry → idempotency key unique index.
- Default READ COMMITTED check-then-insert race **without** lock → classic bug; say it.

### Trade-offs
- Pessimistic lock (correct, serial per resource) vs optimistic (high throughput, retries on conflict) vs unique-slot rows (simplest, needs fixed granularity) vs SERIALIZABLE (simple code, deadlock risk).

### Follow-up questions
1. How do two users booking the same slot at the same instant resolve? 2. Why not just `IF NOT EXISTS ... INSERT`? 3. Unique index vs overlap check? 4. How do holds expire? 5. Multi-resource atomic booking (lock ordering to avoid deadlock)? 6. Idempotency? 7. Overbooking intentionally (airlines)? 8. How would you test the race (parallel tasks hitting API)?

---

## Design 14: Idempotent Payment API

### Requirements & clarifying questions
- Client may retry a payment request after timeout; charge must happen at most once; response replayed.
- Ask: key supplied by client? key TTL (24 h)? What if same key different body? Concurrent duplicate requests? PSP also idempotent?

### Capacity estimation
- 500 payments/s peak; idempotency table 43 M rows/day if keys kept 24 h → purge by TTL/partition.

### High-level architecture
```
Client ─► API (Idempotency middleware/filter) ─► SQL IdempotencyKey table (claim key)
                                              └─► Payment service ─► PSP (with same key) ─► update Payment row + store response
```

### Data model
```sql
CREATE TABLE dbo.IdempotencyKey (
    KeyHash BINARY(32) NOT NULL,              -- SHA-256(clientId + ":" + key)
    ClientId INT NOT NULL,
    RequestHash BINARY(32) NOT NULL,          -- detect same key, different payload
    Status TINYINT NOT NULL,                  -- 0 InProgress, 1 Completed
    ResponseCode INT NULL, ResponseBody NVARCHAR(MAX) NULL,
    CreatedUtc DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    LockedUntilUtc DATETIME2 NULL,
    CONSTRAINT PK_Idem PRIMARY KEY (KeyHash)
);
CREATE TABLE dbo.Payment (PaymentId UNIQUEIDENTIFIER PRIMARY KEY, OrderId UNIQUEIDENTIFIER NOT NULL UNIQUE, Amount DECIMAL(18,2), Status TINYINT, PspRef VARCHAR(100));
```

### API design
```
POST /api/payments   Header: Idempotency-Key: 8f1c...   Body {orderId, amount, token}
 → 201 first time | same 201 + same body on replay | 409 if in-progress | 422 if key reused with different body
```

### Key .NET implementation
```csharp
public async Task<IResult> HandleAsync(string key, PaymentRequest req, int clientId, CancellationToken ct)
{
    var keyHash = Sha256($"{clientId}:{key}");
    var reqHash = Sha256(JsonSerializer.Serialize(req));

    try   // claim: the PK makes this atomic across all instances
    {
        _db.IdempotencyKeys.Add(new IdempotencyKey { KeyHash = keyHash, ClientId = clientId, RequestHash = reqHash, Status = 0 });
        await _db.SaveChangesAsync(ct);
    }
    catch (DbUpdateException ex) when (ex.InnerException is SqlException { Number: 2627 or 2601 })
    {
        _db.ChangeTracker.Clear();
        var existing = await _db.IdempotencyKeys.AsNoTracking().SingleAsync(x => x.KeyHash == keyHash, ct);
        if (!existing.RequestHash.SequenceEqual(reqHash)) return Results.UnprocessableEntity("Key reused with different payload");
        if (existing.Status == 0) return Results.Conflict("Request in progress");
        return Results.Text(existing.ResponseBody!, "application/json", statusCode: existing.ResponseCode);
    }

    var result = await _payments.ChargeAsync(req, idempotencyKeyForPsp: key, ct);   // PSP also gets the key
    await _db.IdempotencyKeys.Where(x => x.KeyHash == keyHash)
        .ExecuteUpdateAsync(s => s.SetProperty(x => x.Status, (byte)1)
                                  .SetProperty(x => x.ResponseCode, 201)
                                  .SetProperty(x => x.ResponseBody, JsonSerializer.Serialize(result)), ct);
    return Results.Created($"/api/payments/{result.PaymentId}", result);
}
```
Crucial: the **claim insert and unique constraint** provide the lock; an `IF EXISTS` check beforehand is a race. Also make business-level uniqueness (e.g. `Payment.OrderId UNIQUE`) as a second line of defence.

### Scaling & bottlenecks
- Key table is a hot insert point → hash-partition or purge in batches; Redis `SET NX` for fast claim with DB as record.
- Response body size small; store only essentials.

### Failure modes
- Crash after PSP charged but before key marked Completed → key stuck InProgress. Resolve: on replay after `LockedUntilUtc`, query PSP by the same idempotency key (PSP returns the original result) — never charge with a new key.
- Same key, different body → reject (422).
- Retry storms → client exponential backoff with jitter.
- Key TTL expiry then retry → duplicate; align TTL with client retry horizon; business uniqueness backs it up.

### Trade-offs
- DB claim (durable, slower) vs Redis NX (fast, durability risk) — often both.
- Store full response (exact replay) vs recompute from Payment row.
- Client-generated key (standard) vs server derived (fingerprint) — derived keys risk legit repeat purchases.

### Follow-up questions
1. What is idempotency vs safe methods? 2. Which HTTP methods are idempotent? 3. How to implement with a DB? 4. Concurrent identical requests? 5. Crash after charge? 6. Same key different payload? 7. How long to keep keys? 8. Idempotency in message consumers (inbox pattern)?

---

## Design 15: Feature-Flag Service

### Requirements & clarifying questions
- Toggle features without deploy; targeting (user/tenant/percentage); kill switch; audit; low-latency evaluation.
- Ask: client-side (SPA/mobile) flags too? Propagation delay tolerated (seconds)? Experiments (A/B)? Environments? Build vs LaunchDarkly/Azure App Configuration?

### Capacity estimation
- 100 flags, 5 k pods/clients; 100 k evaluations/s → must be local, in-memory evaluation; central service handles only config sync (~ few hundred req/s).

### High-level architecture
```
Admin UI ─► Flag API ─► SQL (Flag, Rule, AuditLog) ─► publishes "flags-changed" (Redis pub/sub / Service Bus)
SDK in each app: in-memory snapshot ◄── poll (30 s, ETag) + push invalidate ◄── Flag API/CDN
Evaluation: local, deterministic, no network
```

### Data model
```sql
CREATE TABLE dbo.FeatureFlag (
    FlagKey VARCHAR(100) PRIMARY KEY, Enabled BIT NOT NULL, DefaultValue BIT NOT NULL,
    RolloutPercent TINYINT NOT NULL DEFAULT 0, Version BIGINT NOT NULL, UpdatedUtc DATETIME2, UpdatedBy VARCHAR(100)
);
CREATE TABLE dbo.FlagRule (RuleId INT IDENTITY PRIMARY KEY, FlagKey VARCHAR(100) REFERENCES dbo.FeatureFlag, Attribute VARCHAR(50), Operator VARCHAR(10), Value NVARCHAR(200), Result BIT, Priority INT);
```

### API design
```
GET  /api/flags/snapshot?env=prod   (ETag / If-None-Match → 304)
PUT  /api/flags/{key}   (auth, audited)      POST /api/flags/{key}/kill
```

### Key .NET implementation
Production reality: use `Microsoft.FeatureManagement` with Azure App Configuration; custom sketch:
```csharp
public sealed class FlagEvaluator
{
    private volatile FlagSnapshot _snap = FlagSnapshot.Empty;
    public void Replace(FlagSnapshot s) => _snap = s;          // atomic swap

    public bool IsEnabled(string key, string userId)
    {
        if (!_snap.Flags.TryGetValue(key, out var f) || !f.Enabled) return false;
        if (f.Rules.FirstOrDefault(r => r.Matches(userId)) is { } rule) return rule.Result;
        // stable bucketing: same user always lands in same bucket
        var hash = System.IO.Hashing.XxHash32.HashToUInt32(System.Text.Encoding.UTF8.GetBytes(key + ":" + userId));
        return hash % 100 < f.RolloutPercent;
    }
}

public sealed class FlagPoller : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        using var t = new PeriodicTimer(TimeSpan.FromSeconds(30));
        do { try { var s = await _client.FetchIfChangedAsync(ct); if (s != null) _eval.Replace(s); }
             catch (Exception ex) { _log.LogWarning(ex, "flag refresh failed; keeping last snapshot"); } }
        while (await t.WaitForNextTickAsync(ct));
    }
}
```
(`System.IO.Hashing` is a NuGet package.) With Microsoft.FeatureManagement: `builder.Services.AddFeatureManagement()`, `[FeatureGate("NewCheckout")]`, `IFeatureManager.IsEnabledAsync`, percentage filter built in (`Microsoft.Percentage`).

### Scaling & bottlenecks
- Evaluate locally; snapshot small. Serve snapshot via CDN. Push for urgent kill switch; poll as fallback.
- Don't evaluate per call on hot paths with DB reads.

### Failure modes
- Flag service down → SDK keeps last-known snapshot; ship a bundled default file.
- Stale flag across pods during rollout → inconsistent behaviour; accept eventual consistency, design flag-dependent features to tolerate both states.
- Flag debt (hundreds of dead flags) → ownership + expiry date.
- Sticky bucketing broken by changing hash input → users flip.
- Dependent flags (A requires B) → avoid or validate.

### Trade-offs
- Poll (simple, delay) vs push (fast, connection mgmt) vs both.
- Server-side vs client-side evaluation (client exposes rules → privacy).
- Build vs buy: buy unless needs are unusual.

### Follow-up questions
1. Flag vs config? 2. How do percentage rollouts stay stable per user? 3. What if the flag service is down? 4. How do you remove stale flags? 5. How to test both paths? 6. Kill switch latency? 7. A/B test metrics integration? 8. Flags + DB migrations (expand/contract)?

---

## Design 16: Ticketing with Flash-Sale Traffic

### Requirements & clarifying questions
- 50 k tickets released at a fixed time; 2 M users arrive within seconds; no overselling; fairness; payment window.
- Ask: seat-selection or general admission? Is waiting-room acceptable? Bots? Hold time? Per-user limit?

### Capacity estimation
- 2 M users, each polling/clicking: 200 k req/s burst for ~60 s. Inventory writes: only 50 k successes needed, so shed 97% of traffic cheaply before DB.

### High-level architecture
```
Users ─► CDN (static) ─► Waiting Room / Virtual Queue (token with position) ─► admits N/sec
        ─► API ─► Redis (atomic stock DECR / Lua, per-user limit) ─► Order queue (Service Bus) ─► Order workers ─► SQL (source of truth)
                        └─► Hold with TTL (10 min) ─► Payment ─► confirm / expire → stock returned
```
Principle: **protect the database; absorb the spike in front of it**.

### Data model
```sql
CREATE TABLE dbo.TicketInventory (EventId INT, TierId INT, Total INT, Sold INT, PRIMARY KEY (EventId, TierId));
CREATE TABLE dbo.TicketHold (HoldId UNIQUEIDENTIFIER PRIMARY KEY, EventId INT, TierId INT, UserId INT, Qty INT, ExpiresUtc DATETIME2, Status TINYINT);
CREATE UNIQUE INDEX UX_Hold_User ON dbo.TicketHold(EventId, UserId) WHERE Status IN (0,1);   -- one active hold per user
CREATE TABLE dbo.Ticket (TicketId UNIQUEIDENTIFIER PRIMARY KEY, EventId INT, SeatId INT NULL, OrderId UNIQUEIDENTIFIER);
CREATE UNIQUE INDEX UX_Ticket_Seat ON dbo.Ticket(EventId, SeatId) WHERE SeatId IS NOT NULL;  -- DB-level seat uniqueness
```

### API design
```
POST /api/events/{id}/queue/join → { queueToken, position }     GET /queue/status (poll or SignalR)
POST /api/events/{id}/hold {tier, qty}  (requires admitted token) → 201 hold | 409 sold out | 429
POST /api/holds/{id}/checkout → payment → 200
```

### Key .NET implementation
Atomic stock claim in Redis (Lua = atomic):
```csharp
private const string Reserve = @"
local stock = tonumber(redis.call('GET', KEYS[1]) or '0')
local qty = tonumber(ARGV[1])
if redis.call('SISMEMBER', KEYS[2], ARGV[2]) == 1 then return -2 end   -- user already holds
if stock < qty then return -1 end
redis.call('DECRBY', KEYS[1], qty)
redis.call('SADD', KEYS[2], ARGV[2])
return stock - qty";

var res = (long)await db.ScriptEvaluateAsync(Reserve,
    new RedisKey[] { $"stock:{eventId}:{tier}", $"holders:{eventId}" },
    new RedisValue[] { qty, userId });
if (res == -1) return Results.Conflict("Sold out");
// then enqueue durable hold creation; SQL unique indexes are the final arbiter
```
SQL fallback for correctness (no Redis): `UPDATE TicketInventory SET Sold = Sold + @q WHERE EventId=@e AND TierId=@t AND Total - Sold >= @q` and check `@@ROWCOUNT` — works but a single hot row serializes (~ hundreds–low thousands/s); acceptable only after the waiting room limits traffic. Hold expiry: Redis key TTL + sweeper that returns stock (`INCRBY`) and marks hold expired; reconciliation job compares Redis counters to SQL.

Admission control: waiting room admits X users/sec based on measured backend capacity (use a signed token, JWT with `exp`).

### Scaling & bottlenecks
- Static assets on CDN; API autoscale pre-warmed (scale ahead of the on-sale time; autoscale lag kills flash sales).
- Hot-row contention → Redis counter or split inventory into buckets (e.g., 50 buckets, pick random/hash bucket).
- Bot defence: CAPTCHA, per-IP/user limits, signed queue tokens.
- Async order creation via queue smooths DB write rate.

### Failure modes
- Redis failover loses recent DECRs → stock may be oversold: SQL unique/conditional constraints reject extras → refund/apologise path; reconcile.
- Holds never paid → TTL expiry returns stock (users get a second chance).
- Payment gateway slow → separate payment concurrency pool; bulkhead.
- Thundering herd at T=0 → jittered client retry, queue.
- Queue token replay/sharing → bind to user/session.

### Trade-offs
- Fairness (FIFO queue) vs throughput (first-come hits servers = random luck).
- Redis speed vs SQL correctness: use Redis as gate, SQL as truth.
- Hold-then-pay (UX) vs pay-immediately (no stuck stock, more failures).
- Lottery/raffle sales eliminate the spike entirely.

### Follow-up questions
1. How do you avoid overselling? 2. Why a waiting room? 3. Hot row problem and fixes? 4. Hold expiry? 5. Redis dies mid-sale? 6. Bots/scalpers? 7. Seat maps with 50 k seats — locking granularity? 8. How would you load test it (k6/NBomber/Azure Load Testing) and what do you measure? 9. Pre-scaling vs autoscaling?

---

# PART 2 — PRODUCTION SCENARIOS

**General incident playbook (memorize):**
1. **Stabilize first** (rollback, scale out, disable flag, shed load), diagnose second.
2. Establish **what changed** (deploy, config, traffic, data volume, dependency, cert/secret expiry).
3. Use **golden signals**: latency, traffic, errors, saturation (RED/USE).
4. Follow the request: client → LB → app → dependency (DB/cache/HTTP) → find the layer with the time.
5. Capture evidence before restarting (dump, traces, query plans).
6. Blameless post-mortem with action items.

**Tooling cheat-sheet**
| Need | Tool |
|---|---|
| Live counters | `dotnet-counters monitor -p <pid> System.Runtime Microsoft.AspNetCore.Hosting` |
| CPU/trace | `dotnet-trace collect -p <pid>`, PerfView, Application Insights Profiler |
| Dumps | `dotnet-dump collect -p <pid>`; analyze: `dotnet-dump analyze` (`clrstack`, `dumpheap -stat`, `gcroot`, `threads`) |
| GC/memory | `dotnet-gcdump collect`, Visual Studio memory profiler, `dotnet-counters` gc-heap-size |
| SQL waits | `sys.dm_exec_requests`, `sys.dm_os_wait_stats`, Query Store, `sp_whoisactive` |
| Logs/traces | Application Insights (KQL), Serilog + Seq/ELK, OpenTelemetry |
| K8s | `kubectl logs/describe/top`, `kubectl get events` |

---

## Scenario 1: API suddenly slow

**Symptom:** p95 latency of `/api/orders` jumps from 200 ms to 6 s at 10:05; no error spike yet; users complain.

**First questions:** All endpoints or one? All users or some (tenant/region)? Started when — what deployed/changed at that time? Traffic up? Only reads or writes? Dependencies healthy?

**Diagnosis steps:**
1. Application Insights KQL — which operation, and where time is spent:
```kusto
requests | where timestamp > ago(1h)
| summarize p95 = percentile(duration, 95), count() by bin(timestamp, 5m), name
| order by timestamp asc
```
```kusto
dependencies | where timestamp > ago(1h)
| summarize p95 = percentile(duration, 95), count() by bin(timestamp, 5m), type, target
```
2. If dependency (SQL) time dominates → check live SQL:
```sql
SELECT r.session_id, r.status, r.wait_type, r.wait_time, r.blocking_session_id, r.cpu_time,
       r.total_elapsed_time, t.text
FROM sys.dm_exec_requests r CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
WHERE r.session_id > 50 ORDER BY r.total_elapsed_time DESC;
```
3. Query Store → "Regressed Queries" (plan change). Check `sys.dm_os_wait_stats` deltas.
4. If app time dominates (little dependency time) → `dotnet-counters` for thread pool queue length, GC pauses, CPU.
5. Check infra: CPU/memory on node, connection counts, downstream API latency.

**Root causes (typical):** plan regression after stats update/parameter sniffing; missing index with data growth; blocking by a long transaction; thread pool starvation; GC pressure; slow downstream HTTP call; cache expired/cold; noisy neighbour; N+1 introduced by deploy.

**Fix:** Mitigate: rollback/disable flag; scale out; kill blocking session; force last good plan (`sp_query_store_force_plan`) / `OPTION (RECOMPILE)`; add index; add timeout + circuit breaker on slow dependency.

**Prevention:** SLO alerts on p95, Query Store enabled with plan-regression alert, load tests pre-release, canary deployments, dependency timeouts, dashboards for RED metrics.

**Interview story:** "Latency tripled after a morning deploy. I split time by layer in App Insights and saw SQL dependency time was the whole delta. Query Store showed one query's plan flipped to a scan after a stats update on a table that had grown 10×. I forced the previous plan to stop the bleeding, then added a covering index and fixed the parameter sniffing. We added a plan-regression alert and a perf test on that endpoint."

---

## Scenario 2: High CPU in w3wp / dotnet process

**Symptom:** CPU 95%+ on app servers; requests queueing; autoscale maxed.

**First questions:** One instance or all? Started with deploy or traffic? Is it user-mode CPU in our process or something else (antivirus, other process)? GC-related (% time in GC)?

**Diagnosis steps:**
1. Identify process: `Get-Process w3wp | Sort CPU -desc` / `top -H -p <pid>` (Linux) / Task Manager.
2. Live counters: `dotnet-counters monitor -p <pid> System.Runtime` → `cpu-usage`, `gc-heap-size`, `time-in-gc` (% time in GC), `threadpool-queue-length`, `exception-count` (exception storms burn CPU).
3. Capture trace: `dotnet-trace collect -p <pid> --duration 00:00:30` → open in PerfView/VS; look at hot stacks. Or App Insights Profiler.
4. If suspect infinite loop/regex: take two dumps 10 s apart `dotnet-dump collect -p <pid>`; `dotnet-dump analyze` → `threads`, `clrstack` on hot threads (`!runaway` equivalent in WinDbg).
5. Check for exceptions flood: App Insights `exceptions | summarize count() by type, outerMessage`.

**Root causes:** catastrophic regex backtracking; exception-driven control flow; tight retry loops; serialization of huge objects; LINQ-to-Objects over big lists inside requests; GC thrash from allocations (server GC vs workstation); spinning on locks; JIT/tiered compilation storm after restart; inefficient `string +=` loops; EF client-side evaluation.

**Fix:** Mitigate: scale out, recycle, rollback. Fix hot path found in trace: regex with `RegexOptions.NonBacktracking` (.NET 7+) or match timeout; cache computed results; reduce allocations (`ArrayPool`, `Span<T>`); push filtering to SQL.

**Prevention:** Regex timeouts, benchmark critical paths (BenchmarkDotNet), CPU alerts with auto-trace capture, load tests.

**Interview story:** "CPU hit 100% after a release. `dotnet-counters` showed exceptions/sec in the thousands. A trace showed `FormatException` in a parsing loop for a malformed partner feed; each exception cost ~50 µs of stack unwinding. We replaced `Parse` with `TryParse`, rolled out the fix, and added an exception-rate alert."

---

## Scenario 3: Memory leak (growing memory)

**Symptom:** Pod memory grows steadily, OOMKilled every ~6 hours; or w3wp private bytes grow until recycle.

**First questions:** Managed or native memory? Gen2/LOH growth or flat GC heap but rising working set? Correlates with traffic or a specific endpoint? Container memory limit vs GC heap hard limit?

**Diagnosis steps:**
1. `dotnet-counters monitor -p <pid> System.Runtime` → `gc-heap-size`, `gen-2-size`, `loh-size`, `working-set`. If heap flat but working set grows → native/unmanaged leak (e.g., `HttpClient` misuse, native libs, unreleased `IDisposable`).
2. Capture two gcdumps/dumps apart in time: `dotnet-gcdump collect -p <pid> -o a.gcdump` … later `-o b.gcdump`; compare in Visual Studio / PerfView ("diff").
3. `dotnet-dump collect -p <pid>` → `dotnet-dump analyze`:
```
> dumpheap -stat          # top types by size/count
> dumpheap -type MyApp.Cache.Entry -min 85000
> gcroot <address>        # who keeps it alive
```
4. Review suspects: static collections/dictionaries that only add, event handlers never unsubscribed, `MemoryCache` without size limit, captured closures in timers, `Task`s never completing, EF `DbContext` registered singleton (change tracker grows!), `new HttpClient()` per call.

**Root causes:** unbounded static cache; singleton holding scoped `DbContext`; event subscription leaks; LOH fragmentation from big arrays/strings (`byte[]` > 85 KB); undisposed `IDisposable`s (`CancellationTokenSource` linked, `Timer`); thread-static data; `ConcurrentDictionary` keyed by unbounded user input.

**Fix:** Bound caches (`MemoryCacheOptions.SizeLimit` + `Size` on entries, expiration); fix lifetimes; dispose; use `ArrayPool<byte>`; `GCSettings.LargeObjectHeapCompactionMode` rarely; set `DOTNET_GCHeapHardLimit` sensibly in containers; stream rather than buffer.

**Prevention:** Memory trend alert (slope, not threshold), soak test before release, code-review checklist for statics/events, limits in caches.

**Interview story:** "Pods restarted every 6 h. Heap diff of two gcdumps showed millions of `UserSession` objects rooted by a static `Dictionary` in a 'quick cache' added without eviction. I replaced it with `MemoryCache` with a size limit and sliding expiration; memory graph became a sawtooth. We added a soak test."

---

## Scenario 4: Thread pool starvation

**Symptom:** Latency climbs steadily under moderate load, CPU low (!), requests time out, a restart helps briefly. Dependencies appear fine.

**First questions:** Is CPU low while requests pile up? Any sync-over-async (`.Result`, `.Wait()`, `GetAwaiter().GetResult()`)? Sudden slow dependency causing threads to block?

**Diagnosis steps:**
1. `dotnet-counters monitor -p <pid> System.Runtime` → `threadpool-thread-count` keeps rising slowly (≈1–2 threads/sec injection), `threadpool-queue-length` > 0 sustained, `cpu-usage` low.
2. `dotnet-stack report -p <pid>` (dotnet-stack tool) or `dotnet-dump` → `clrstack -all` / `threads`; look for many threads sitting in `Task.Wait`, `.Result`, `Monitor.Enter`, or sync DB/HTTP calls.
3. Check that the ASP.NET Core "ThreadPool starvation" event appears in traces (.NET 6+ diagnostic; `dotnet-trace` with `Microsoft-Windows-DotNETRuntime` ThreadPool events).
4. Reproduce with load test; observe queue length growth.

**Root causes:** sync-over-async in request path; blocking calls (`Thread.Sleep`, sync `HttpClient.Send`, `SqlCommand.ExecuteReader`) at scale; `lock` around awaited work (SemaphoreSlim should be used); `Task.Run` wrapped around sync code on request threads; fire-and-forget flood; slow downstream making async calls pile up (not starvation but similar).

**Fix:** Make code async end-to-end; replace `lock` with `SemaphoreSlim.WaitAsync`; as stopgap `ThreadPool.SetMinThreads` (hides, does not cure); add timeouts and bulkheads (limit concurrency per dependency).
```csharp
// BAD
var user = _client.GetUserAsync(id).Result;
// GOOD
var user = await _client.GetUserAsync(id, ct);
```

**Prevention:** Analyzers (`Microsoft.VisualStudio.Threading.Analyzers` / `VSTHRD002`), async-only code standard, load tests with slow-dependency fault injection, alert on `threadpool-queue-length`.

**Interview story:** "Latency rose with CPU at 15%. Counters showed thread pool queue growing and threads climbing slowly. Dump stacks showed ~200 threads blocked at `.Result` in a legacy wrapper around an HTTP client. I converted the chain to async and added timeouts. p99 dropped from 12 s to 300 ms."

---

## Scenario 5: SQL deadlocks in production

**Symptom:** Sporadic `SqlException` number 1205 "Transaction was deadlocked… chosen as the deadlock victim"; some orders fail at peak.

**First questions:** Which statements? Frequency? New code or higher concurrency? Same tables accessed in different order? Long transactions? Isolation level?

**Diagnosis steps:**
1. Capture deadlock graphs from the built-in `system_health` Extended Events session:
```sql
SELECT CAST(xed.event_data AS XML) AS deadlock_xml
FROM (SELECT CAST(target_data AS XML) AS td
      FROM sys.dm_xe_session_targets st JOIN sys.dm_xe_sessions s ON s.address = st.event_session_address
      WHERE s.name = 'system_health' AND st.target_name = 'ring_buffer') t
CROSS APPLY td.nodes('RingBufferTarget/event[@name="xml_deadlock_report"]') AS x(xed_node)
CROSS APPLY (SELECT x.xed_node.query('.') AS event_data) xed;
```
(Or read `system_health*.xel` via `sys.fn_xe_file_target_read_file`.) Enable trace flag 1222 if needed for error log output.
2. Read the graph: processes, resources (key/page locks), the two statements, indexes involved.
3. Look at the code path: transaction scope, access order, missing index causing scans (more locks).

**Root causes:** two transactions updating tables in opposite order; read-then-update patterns (shared→exclusive lock conversion; both hold S, both want X); missing index → range/table scans locking many rows; long transactions including external calls; lock escalation; foreign-key checks.

**Fix:** consistent table/row access order; add the missing index (covering); shorten transactions; use `UPDLOCK` hint on the initial read (`SELECT ... WITH (UPDLOCK, ROWLOCK)`) to avoid conversion deadlocks; enable `READ_COMMITTED_SNAPSHOT` so readers don't block/deadlock with writers; **retry on 1205** with backoff:
```csharp
services.AddDbContext<AppDb>(o => o.UseSqlServer(cs, sql => sql.EnableRetryOnFailure(
    maxRetryCount: 5, maxRetryDelay: TimeSpan.FromSeconds(5), errorNumbersToAdd: new[] { 1205 })));
```
Note: with a retrying strategy, user-initiated transactions must run inside `CreateExecutionStrategy().ExecuteAsync(...)`.

**Prevention:** deadlock alerts (XE/Monitor), standard access order doc, short transactions, load-test concurrency, idempotent operations so retries are safe.

**Interview story:** "We saw 1205s on checkout under load. The graph showed `UPDATE Inventory` then `UPDATE Orders` in one path and reverse in another. I standardized order (Inventory first), added an index on `OrderLines(OrderId)` that removed a scan, and added retry for 1205. Deadlocks dropped to zero."

---

## Scenario 6: N+1 queries discovered in prod

**Symptom:** A list endpoint with 100 items takes 4 s; DB shows thousands of tiny identical queries per minute; DB CPU moderate but round-trips high.

**First questions:** Is the query count proportional to page size? Lazy loading proxies on? Loop with `await` inside calling the DB? Which endpoint?

**Diagnosis steps:**
1. App Insights: dependency count per request:
```kusto
dependencies | where type == "SQL" | summarize calls=count() by operation_Id, operation_Name
| summarize avg(calls), max(calls) by operation_Name | order by avg_calls desc
```
2. SQL side: Query Store / DMV top by `execution_count`:
```sql
SELECT TOP 20 qs.execution_count, qs.total_elapsed_time/qs.execution_count AS avg_us, SUBSTRING(t.text,1,200) AS q
FROM sys.dm_exec_query_stats qs CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) t
ORDER BY qs.execution_count DESC;
```
3. Enable EF logging (`LogTo(Console.WriteLine, LogLevel.Information)` / `EnableSensitiveDataLogging` in non-prod) and see repeated `SELECT ... WHERE Id = @p`.

**Root causes:** lazy loading (`UseLazyLoadingProxies`) while iterating; navigation property access in a loop; mapper/serializer touching navigations; repository method called per item.

**Fix:**
```csharp
// before: N+1
var orders = await db.Orders.ToListAsync();
foreach (var o in orders) total += o.Lines.Sum(l => l.Price);   // lazy load per order

// after: projection (best), or Include
var result = await db.Orders.AsNoTracking()
    .Select(o => new OrderDto(o.Id, o.Lines.Sum(l => l.Price)))
    .ToListAsync();
// Include with .AsSplitQuery() if loading collections to avoid cartesian explosion
```
Batch lookups with `Where(x => ids.Contains(x.Id))`.

**Prevention:** disable lazy loading by default; integration tests asserting query count (EF interceptor counter); code review; dev-time MiniProfiler; alerts on dependency calls per request.

**Interview story:** "A 'recent orders' page degraded as data grew. App Insights showed 301 SQL calls per request. Lazy loading on `Lines`. I switched to a projection into a DTO, which became a single query, and added a test that fails if the endpoint exceeds 3 queries."

---

## Scenario 7: Connection pool exhausted

**Symptom:** `InvalidOperationException: Timeout expired. The timeout period elapsed prior to obtaining a connection from the pool. This may have occurred because all pooled connections were in use and max pool size was reached.`

**First questions:** DB healthy/slow? Connections leaking? Pool size vs concurrency × pods? Long-running requests holding connections?

**Diagnosis steps:**
1. Count connections per app/host:
```sql
SELECT program_name, host_name, DB_NAME(database_id) db, status, COUNT(*) conns
FROM sys.dm_exec_sessions WHERE is_user_process = 1 GROUP BY program_name, host_name, database_id, status;
```
Many `sleeping` with open transactions → leak/forgotten `Dispose`. 
```sql
SELECT s.session_id, s.host_name, s.last_request_end_time, t.text
FROM sys.dm_exec_sessions s JOIN sys.dm_exec_connections c ON c.session_id = s.session_id
CROSS APPLY sys.dm_exec_sql_text(c.most_recent_sql_handle) t
WHERE s.open_transaction_count > 0;
```
2. Counters: `Microsoft.Data.SqlClient.EventSource` pool counters (`dotnet-counters monitor ... Microsoft.Data.SqlClient.EventSource` → `active-hard-connections`, `number-of-pooled-connections`…).
3. Dump: look for threads awaiting a slow query or `.Result` blocking (starvation).
4. Capacity math: pods × `Max Pool Size` (default 100) must be < SQL max connections.

**Root causes:** connection leak (`SqlConnection` not disposed, `DbContext` singleton, `DataReader` unclosed); slow queries holding connections; thread pool starvation; transactions held across HTTP calls; too many pods × pool size; `DbContext` per-iteration in parallel loops; N parallel tasks > pool.

**Fix:** `using`/`await using` everywhere; scoped DbContext; shorten transactions; fix slow queries; tune `Max Pool Size` thoughtfully (not just raise); `AddDbContextPool`; limit parallelism (`Parallel.ForEachAsync` with `MaxDegreeOfParallelism`); use `SemaphoreSlim` to cap concurrent DB work.

**Prevention:** alert on pool usage/wait; leak detection in tests; connection string `Application Name` set for attribution; load tests.

**Interview story:** "After a feature launch we got pool timeouts. DMV showed 300 sleeping sessions with open transactions from our app. A code path opened a transaction, then called a slow third-party API before commit. I moved the HTTP call outside the transaction and capped parallel jobs. Connections fell to ~40."

---

## Scenario 8: Timeouts after deployment

**Symptom:** After release, HttpClient/SQL timeouts and 504s from gateway; before release fine.

**First questions:** Which timeout (client, gateway, SQL command, upstream)? What exactly changed (code, config, infra, dependency version, DB schema)? Cold start/warm-up? Rolling deploy mid-flight?

**Diagnosis steps:**
1. Compare before/after in telemetry by `cloud_RoleInstance`/version: errors per version.
2. Diff release: code, `appsettings`, env vars, package upgrades (e.g., `Microsoft.Data.SqlClient` encryption defaults changed to `Encrypt=True` in v4+ → connection failures), new queries/migrations.
3. Check infra: readiness probes failing → pods restarting; DNS/network policy; connection limits; SNAT port exhaustion (Azure) — `HttpClient` created per request.
4. Check cold start: JIT, EF model building, cache cold → first requests slow; DB plan cache cold after failover.
5. SQL: new query slow? new migration changed index? Query Store regressions.

**Root causes:** missing index in new query; config drift (timeout lowered, wrong connection string); new dependency call in hot path; `new HttpClient()` SNAT exhaustion; cold caches; health-check/readiness misconfig sending traffic before warm; gateway timeout < app timeout; DB stats stale after schema change.

**Fix:** rollback (see Scenario 20) if impact high; fix config; add index; use `IHttpClientFactory`; warm-up endpoints and readiness gating; align timeouts (client > server > DB hierarchy: outer must be longer than inner, to avoid orphan work).

**Prevention:** canary/blue-green with automatic rollback on SLO breach; staging with prod-size data; synthetic checks; config as code diff review; warm-up in `IHostedService` before readiness.

**Interview story:** "Post-deploy 504s hit only the new version. Diff showed a new endpoint call to a pricing service inside a loop with default 100 s timeout and no cache. I rolled back within 10 min, then batched calls, added a 2 s timeout with circuit breaker via Polly, and enabled canary releases with auto-rollback."

---

## Scenario 9: Migration locking a huge table

**Symptom:** Deploy runs EF migration `AddColumn NOT NULL` / `CREATE INDEX` on a 500 M-row table; app hangs, blocked sessions pile up, timeouts.

**First questions:** Which statement? Online operation possible (Enterprise/Azure SQL)? Peak hours? Can we kill safely (rollback time ≈ work done)?

**Diagnosis steps:**
```sql
SELECT r.session_id, r.blocking_session_id, r.wait_type, r.command, r.percent_complete, t.text
FROM sys.dm_exec_requests r CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
WHERE r.blocking_session_id <> 0 OR r.command LIKE 'ALTER%' OR r.command LIKE 'CREATE%';
-- head blocker:
SELECT session_id, blocking_session_id FROM sys.dm_exec_requests WHERE blocking_session_id <> 0;
```
Check log growth (`DBCC SQLPERF(LOGSPACE)`), `sys.dm_tran_locks` for `Sch-M` lock held/waiting (schema modification blocks everything, and queued behind a long reader it blocks all new queries too).

**Root causes:** `ALTER TABLE ... ADD col NOT NULL DEFAULT` with non-constant default or old versions → size-of-data operation; offline index build (`Sch-M`/table lock); data type change rewriting table; running migration at startup (`Database.Migrate()`) on every instance; FK add with check scans table; lock queue behind long query.

**Fix:** if just started and blocking, kill the migration session (it rolls back — can take long). Redo safely:
- Add nullable column (metadata-only) → backfill in batches → add constraint `WITH NOCHECK` then validate later / set NOT NULL after.
```sql
-- batch backfill
WHILE 1=1
BEGIN
  UPDATE TOP (5000) dbo.Orders SET NewCol = 0 WHERE NewCol IS NULL;
  IF @@ROWCOUNT = 0 BREAK;
  WAITFOR DELAY '00:00:00.200';
END
CREATE INDEX IX_X ON dbo.Orders(NewCol) WITH (ONLINE = ON, RESUMABLE = ON);  -- edition permitting
```
- Use `SET LOCK_TIMEOUT` on migration so it fails fast instead of queuing. Use expand/contract; generate idempotent script (`dotnet ef migrations script --idempotent`) and run as separate pipeline step, not `Migrate()` on app start.

**Prevention:** migration review checklist (table size, online?), test on prod-size clone, run off-peak, backward-compatible schema (old & new app versions both work), lock timeout, monitoring for blocked chains.

**Interview story:** "A migration adding an indexed column froze checkout for 8 minutes. I killed it, then redid it as expand/contract: nullable column, batched backfill with throttling, online index build, then the constraint. I moved migrations out of app startup to a gated pipeline step."

---

## Scenario 10: Wrong/stale data from cache

**Symptom:** Users see old prices/profile after update; different servers show different values; sometimes another tenant's data!

**First questions:** Which cache (in-memory per instance, Redis, CDN, HTTP response cache, EF)? TTL? How invalidated? Cache key includes user/tenant/culture/params?

**Diagnosis steps:**
1. Reproduce: hit each instance (`curl` with host/instance header) – differing answers → per-instance `IMemoryCache`.
2. Inspect keys in Redis: `redis-cli --scan --pattern 'price:*'`, `TTL key`, `GET key`.
3. Check response headers: `Cache-Control`, `Age`, `Vary`, CDN hit status (`X-Cache`).
4. Review code: where write path updates DB; is cache invalidated afterwards (and *after commit*)?

**Root causes:** TTL too long; write path doesn't invalidate; invalidate-before-commit race (reader re-caches old value); per-node memory cache with no cross-node invalidation; missing tenant/user/culture in key (data leak); caching entity with mutable shared reference; cache-aside race (reader fetches old, writer updates and invalidates, reader sets old); `ResponseCache`/CDN caching authenticated responses (`Cache-Control: public` on personalized content).

**Fix:** invalidate (delete) after commit, short TTL as safety net, versioned keys (`price:v{version}:{id}`), pub/sub invalidation or Redis for shared cache, include all variance in key, `Cache-Control: private, no-store` for personalized data, jittered TTL.
```csharp
await db.SaveChangesAsync(ct);
await cache.RemoveAsync($"tenant:{tenantId}:product:{id}", ct);   // after commit; tenant in key
```

**Prevention:** cache key conventions with tenant, tests for isolation, metrics for hit ratio/staleness, documented invalidation strategy, `Vary` headers reviewed.

**Interview story:** "Support said prices were wrong for some users. Two pods held `IMemoryCache` entries for 1 h; updates only cleared the local node. I moved to Redis with after-commit invalidation and a 5-minute TTL backstop, and namespaced keys by tenant — which we also found was missing in one key."

---

## Scenario 11: Duplicate records from retries

**Symptom:** Customers double-charged / duplicate orders; duplicates ms apart with different IDs.

**First questions:** Where are retries coming from (client, gateway, Polly, queue redelivery, user double-click)? Is there a unique business key? Was the original request actually successful (timeout after commit)?

**Diagnosis steps:**
```sql
SELECT CustomerId, Total, COUNT(*) c, MIN(CreatedUtc), MAX(CreatedUtc)
FROM dbo.Orders WHERE CreatedUtc > DATEADD(DAY,-1,SYSUTCDATETIME())
GROUP BY CustomerId, Total HAVING COUNT(*) > 1;
```
Correlate with logs: same `X-Correlation-Id`/client request ID? Polly retry logs? Gateway retry policy? Timeout counts just before duplicates.

**Root causes:** client timeout shorter than server processing, then retry while first still succeeds; retry policy on non-idempotent POST; no unique constraint; check-then-insert race; double submit button; message redelivery.

**Fix:** idempotency key (Design 14) with unique index; business unique constraints; retry only idempotent operations or those with keys; clean up duplicates (dedupe script, refund). Unique index + catch 2601/2627 → return existing.

**Prevention:** idempotent by design for all POST that create/charge, UI disable on submit, align timeouts, constraint-first thinking (the database is the last line of defence).

**Interview story:** "Double charges appeared during a slow period. The mobile client retried at 10 s while our API took 12 s. I added Idempotency-Key handling backed by a unique index and the PSP idempotency key, plus a reconcile job, and refunded affected customers."

---

## Scenario 12: Message processed twice

**Symptom:** Same email sent twice, inventory decremented twice, consumers log same MessageId twice.

**First questions:** Broker (Service Bus/RabbitMQ/Kafka) semantic? Lock/visibility timeout vs handling time? Consumer crash after side-effect before ack? Publisher retries creating dup messages?

**Diagnosis steps:**
1. Check delivery count (`DeliveryCount` on Service Bus), lock-lost exceptions (`MessageLockLostException`), handler duration vs lock duration.
2. Look at logs for `MessageId` duplicates and instance IDs (two consumers?).
3. Publisher: outbox relay publishes, crashes before marking processed → republish.
4. Kafka: offset commit after processing + rebalance.

**Root causes:** at-least-once delivery by design; handler slower than lock duration → message reappears; crash between side-effect and complete; publisher retry; non-idempotent handler.

**Fix:** idempotent consumer / inbox table:
```sql
CREATE TABLE dbo.ProcessedMessage (MessageId UNIQUEIDENTIFIER PRIMARY KEY, ProcessedUtc DATETIME2 DEFAULT SYSUTCDATETIME());
```
```csharp
await using var tx = await db.Database.BeginTransactionAsync(ct);
try { db.ProcessedMessages.Add(new(msg.MessageId)); await db.SaveChangesAsync(ct); }   // PK violation = duplicate
catch (DbUpdateException) { await args.CompleteMessageAsync(args.Message, ct); return; }
await ApplyBusinessChangeAsync(msg, ct);   // same transaction
await db.SaveChangesAsync(ct); await tx.CommitAsync(ct);
await args.CompleteMessageAsync(args.Message, ct);
```
Also: natural idempotence (set-state rather than increment), lock renewal (`MaxAutoLockRenewalDuration`), Service Bus duplicate detection (publisher side, window-limited).

**Prevention:** assume at-least-once everywhere, idempotency tests (replay same message), poison/DLQ monitoring.

**Interview story:** "Customers got duplicate emails. A handler took 90 s but the lock was 60 s, so the message was redelivered to another instance. I set automatic lock renewal, and, more importantly, added an inbox table so the handler is idempotent no matter how often it's redelivered."

---

## Scenario 13: App works locally but fails in container/Kubernetes

**Symptom:** `CrashLoopBackOff`, `ImagePullBackOff`, 502/503 from ingress, or DB connection failures only in the cluster.

**First questions:** Pod status/events? Exit code? Does the container run via `docker run` locally with same env? Config/secrets/volumes present? Network policy/DNS?

**Diagnosis steps:**
```
kubectl get pods -n app
kubectl describe pod <pod>            # events: OOMKilled, probe failed, image pull, mount errors
kubectl logs <pod> --previous         # logs of crashed container
kubectl get events --sort-by=.lastTimestamp
kubectl exec -it <pod> -- sh          # curl dependencies, env, nslookup
kubectl top pod
docker run --rm -p 8080:8080 -e ASPNETCORE_ENVIRONMENT=Production myimage
```
Checklist:
- **Port/URL binding:** .NET 8 images listen on 8080 by default (`ASPNETCORE_HTTP_PORTS`), not 80; wrong `containerPort`/service targetPort → 502/503.
- **Env config:** `ConnectionStrings__Default` with double underscore for nested keys; missing secrets; case/typo.
- **Probes:** liveness hitting dependency-checking endpoint → restarts storm; readiness before warm-up; probe timeouts.
- **Memory limits:** OOMKilled (exit 137) — container limit lower than GC needs; .NET respects cgroup limits (default heap hard limit 75%).
- **Non-root user / read-only filesystem:** writes to `/app` or temp fail; data protection keys not persisted → cookie/antiforgery failures across pods.
- **HTTPS/cert:** dev cert absent; TLS terminated at ingress → `UseForwardedHeaders` needed (`X-Forwarded-Proto`), else redirect loops.
- **Time zone / culture / ICU:** Alpine images need `InvariantGlobalization` or `icu-libs`; `System.Globalization.Invariant`. Case-sensitive file paths on Linux; `\` path separators.
- **DNS/network:** can't reach SQL (firewall, private endpoint, `Encrypt=True`/cert trust).
- **Shutdown:** SIGTERM handling & `terminationGracePeriodSeconds`.

**Root causes:** from above list; most common: env var naming, port mismatch, probes, memory limit, missing forwarded headers, Linux case sensitivity.

**Fix/Prevention:** run the same image locally with prod-like env (docker compose), health endpoints split (`/health/live` simple, `/health/ready` with deps), configure `AddDataProtection().PersistKeysToAzureBlobStorage/DbContext`, CI container smoke test, resource requests/limits tuned from load test.

**Interview story:** "Works on my machine, CrashLoop in AKS. `kubectl logs --previous` showed 'Unable to connect to SQL'. Env var was `ConnectionStrings:Default` — colon isn't valid in env names for the manifest; changed to `ConnectionStrings__Default`. Then 503s: readiness hit `/health` which checked Redis not yet reachable; split live vs ready."

---

## Scenario 14: Intermittent 500s under load

**Symptom:** 0.5–2% 500 errors at peak; not reproducible at low load; no pattern by endpoint.

**First questions:** Exception types/messages? Same instance or all? Correlated with pod restarts, GC, deploys, dependency blips? Same users?

**Diagnosis steps:**
```kusto
exceptions | where timestamp > ago(2h)
| summarize count() by type, outerMessage, cloud_RoleInstance, bin(timestamp, 5m)
| order by count_ desc
```
```kusto
requests | where resultCode startswith "5" | summarize count() by name, resultCode, bin(timestamp, 5m)
```
Look at distribution: pool timeouts? `DbUpdateConcurrencyException`? `SqlException 1205/-2`? `HttpRequestException`/`TaskCanceledException`? `ObjectDisposedException` (DbContext used after request, `async void`, shared instance)? `InvalidOperationException: A second operation was started on this context instance` (DbContext thread-safety)? Kestrel limits (max concurrent connections)? Check LB health, 502 vs 500 distinction, pod OOM/restarts.

**Root causes:** race conditions with shared state (static mutable, singleton using scoped DbContext, non-thread-safe `Dictionary`, `DateTime` formatters); parallel use of one DbContext (`Task.WhenAll` over same context); resource exhaustion (pool, threads, sockets/SNAT); transient dependency faults with no retry; deadlocks; timeouts; rare data-dependent null refs.

**Fix:** fix the dominating exception; one `DbContext` per concurrent unit (use `IDbContextFactory<T>` in parallel code); concurrent collections/locks; Polly retry+timeout+circuit breaker for transients (`AddStandardResilienceHandler()` from `Microsoft.Extensions.Http.Resilience`); global exception handler (`IExceptionHandler`) with correlation id.

**Prevention:** load/chaos testing, structured logging with correlation IDs, error budget alerts, concurrency review checklist.

**Interview story:** "At peak, 1% of requests 500'd with 'A second operation was started on this context'. A service ran `Task.WhenAll` over queries using the same scoped DbContext. I switched to `IDbContextFactory`, creating a context per task. Errors vanished."

---

## Scenario 15: Token expiry incidents

**Symptom:** Wave of 401s at exact times (e.g., every hour / midnight / on a given day); or service-to-service calls start failing; mobile users logged out unexpectedly.

**First questions:** Which token (user JWT, client secret/certificate, API key, SAS token, Entra app secret, signing key)? Pattern of timing? Clock skew? Refresh flow failing?

**Diagnosis steps:**
1. Decode the token (jwt.ms) — compare `exp`, `nbf`, `iat`, `iss`, `aud` with server time. Check server NTP/time drift.
2. Logs: `IDX10223: Lifetime validation failed. The token is expired`, `IDX10214 audience`, `IDX10501/10503 signature validation` (signing key rotated; `kid` not found in cache).
3. Check expiry of secrets/certs: Azure Entra app registration client secret expiry, Key Vault certificates, SAS expiry dates, TLS cert expiry (`openssl s_client -connect host:443 | openssl x509 -noout -dates`).
4. Check refresh logic: concurrent refreshes racing (refresh token rotation: only one use allowed → others fail), clients caching tokens past `exp`.
5. Are multiple instances using different data-protection keys / signing keys?

**Root causes:** expired client secret/cert (classic: runs fine for 1–2 years then dies); no proactive refresh; refresh token reuse race; clock skew (default `ClockSkew` 5 min); signing key rollover with stale JWKS cache (`ConfigurationManager` refresh interval); SAS/URL expiry shorter than long-running transfer; DataProtection keys not shared → cookies invalid after restart; access token cached longer than lifetime.

**Fix:** rotate secret/cert, deploy; use managed identity / certificate with automated renewal; refresh tokens proactively (e.g., at 80% lifetime) with single-flight lock; on 401 refresh-and-retry once; let `JwtBearer` auto-refresh metadata (`RefreshOnIssuerKeyNotFound = true` by default for OIDC metadata), keep `ClockSkew` reasonable.
```csharp
options.TokenValidationParameters.ClockSkew = TimeSpan.FromMinutes(2);
```

**Prevention:** expiry inventory with alerts at 30/14/7 days (Key Vault events, Azure Monitor), managed identities, runbook for rotation, dual-secret overlap during rotation, synthetic monitoring using real auth flow.

**Interview story:** "At 02:00 sharp all partner calls 401'd. The Entra client secret created two years ago expired. I added a new secret, updated Key Vault, restarted. Postmortem: moved to managed identity where possible, added expiry alerts at 30/7 days, and documented rotation."

---

## Scenario 16: DB failover causing errors

**Symptom:** Burst of connection errors for 30–120 s (`A transport-level error has occurred`, `SqlException 40613/40197/40501/10928` on Azure SQL, `-2`, `Connection forcibly closed`), then recovery; or app never recovers until restart.

**First questions:** Planned (patch) or unplanned failover? Availability Group listener or direct node? `MultiSubnetFailover` set? Did app recover by itself? Were pooled connections stale?

**Diagnosis steps:**
1. Correlate error window with failover event: AG dashboard / `sys.dm_hadr_availability_replica_states`; Azure SQL resource health/activity log; SQL error log (`EXEC xp_readerrorlog`).
2. Check connection string: using listener name? `MultiSubnetFailover=True`? `ConnectRetryCount/ConnectRetryInterval`? 
3. Check EF `EnableRetryOnFailure` enabled? User transactions wrapped in `CreateExecutionStrategy()`?
4. After failover, pooled connections are dead; first uses fail → SqlClient detects broken connection; `ClearPool` if persistent.
5. Check long timeouts (default command timeout 30 s, connect timeout 15 s) and whether the app queued requests.

**Root causes:** connecting to node name instead of listener; no transient-fault retry; connections in pool broken; non-idempotent retries; DNS TTL caching old IP (`HostNameInCertificate` / MultiSubnet); app doing writes mid-transaction → lost, uncommitted.

**Fix:**
```csharp
options.UseSqlServer(cs, sql => sql.EnableRetryOnFailure(
    maxRetryCount: 6, maxRetryDelay: TimeSpan.FromSeconds(30), errorNumbersToAdd: null));
// cs: "Server=tcp:ag-listener,1433;...;MultiSubnetFailover=True;ConnectRetryCount=3;ConnectRetryInterval=10;Connect Timeout=30"
```
For manual transactions: `await db.Database.CreateExecutionStrategy().ExecuteAsync(async () => { await using var tx = ...; ...; await tx.CommitAsync(); });`. Ensure idempotent operations (outcome of commit may be unknown). Add Polly for HTTP side. Graceful degradation: queue writes, serve cached reads.

**Prevention:** regular failover drills (chaos), listener + MultiSubnetFailover, retry/idempotency design, alert distinguishing transient vs sustained, readiness probe tolerating transient DB blips.

**Interview story:** "During Azure SQL maintenance failover we had 90 s of errors and one-pod stuck. We lacked retry on user-managed transactions: wrapped them in an execution strategy, made the operations idempotent, and ran a failover drill in staging to prove recovery without restarts."

---

## Scenario 17: Slow report query

**Symptom:** Month-end report takes 3 min or times out; fast for small date ranges; blocks others.

**First questions:** Which query/params? Always slow or only for some parameters (sniffing)? Data growth? Recently changed? Ran on OLTP?

**Diagnosis steps:**
1. Get actual execution plan (SSMS → Include Actual Plan) or from Query Store; with timing/IO:
```sql
SET STATISTICS IO, TIME ON;
```
2. Look for: table/clustered index scans, key lookups on many rows, hash spills (tempdb warnings), estimated vs actual row mismatch (stale stats / sniffing), missing index hints, implicit conversions (`CONVERT_IMPLICIT` — e.g., `NVARCHAR` param vs `VARCHAR` column), scalar UDFs, non-SARGable predicates (`WHERE YEAR(Date)=2026`).
3. Missing index DMVs:
```sql
SELECT TOP 10 migs.avg_user_impact, mid.statement, mid.equality_columns, mid.inequality_columns, mid.included_columns
FROM sys.dm_db_missing_index_group_stats migs
JOIN sys.dm_db_missing_index_groups mig ON mig.index_group_handle = migs.group_handle
JOIN sys.dm_db_missing_index_details mid ON mid.index_handle = mig.index_handle
ORDER BY migs.avg_user_impact * migs.user_seeks DESC;
```
4. Waits for that session (`PAGEIOLATCH` → IO; `CXPACKET`/`CXCONSUMER` → parallelism; `LCK_M_*` → blocking; `RESOURCE_SEMAPHORE` → memory grants).

**Root causes:** non-SARGable predicates; missing/covering index; parameter sniffing; stale statistics; big sort/aggregation on rowstore; scalar UDF/row-by-row; reading entire table to compute aggregates repeatedly; blocking from OLTP writers.

**Fix:** rewrite sargable: `WHERE Date >= '2026-01-01' AND Date < '2027-01-01'`; add covering/filtered index; update stats; `OPTION (RECOMPILE)` or `OPTIMIZE FOR`; columnstore index; pre-aggregated summary table; run on readable secondary; `READ_COMMITTED_SNAPSHOT`; paginate/async export (see Design 11).

**Prevention:** Query Store monitoring, performance tests with prod-like volume, index maintenance/stats jobs, report limits (max range), separate reporting workload.

**Interview story:** "A month-end report went from 20 s to 4 min. The plan showed a scan because `WHERE CONVERT(date, CreatedUtc) = @d` was non-sargable, plus a key lookup storm. Rewriting to a range predicate and adding a covering index cut it to 2 s; the long-term fix was a nightly summary table."

---

## Scenario 18: Disk full due to log growth

**Symptom:** Service crashes or `No space left on device`/Windows disk 100%; SQL Server error 9002 (transaction log full); app can't write files.

**First questions:** Which disk and which files (app logs, SQL `.ldf`, tempdb, dumps, docker overlay)? What grew and since when? Log level changed (Debug in prod)? DB recovery model/backup chain broken?

**Diagnosis steps:**
```powershell
Get-PSDrive -PSProvider FileSystem
Get-ChildItem D:\ -Recurse -File -ErrorAction SilentlyContinue | Sort Length -Descending | Select -First 20 FullName,@{n='GB';e={[math]::Round($_.Length/1GB,2)}}
```
Linux/K8s: `df -h`, `du -xh / | sort -rh | head`, `kubectl describe node` (DiskPressure), `docker system df`.
SQL log:
```sql
SELECT name, log_reuse_wait_desc, recovery_model_desc FROM sys.databases;   -- why log can't truncate
DBCC SQLPERF(LOGSPACE);
SELECT TOP 5 * FROM sys.dm_tran_active_transactions;   -- open long transaction
```
`log_reuse_wait_desc`: `LOG_BACKUP` (no log backups in FULL), `ACTIVE_TRANSACTION`, `REPLICATION`, `AVAILABILITY_REPLICA`.

**Root causes:** debug logging in prod; log rotation/retention missing; exception loop spamming logs; file sink without size limit; SQL FULL recovery without log backups; long-running open transaction; huge batch delete/update in single transaction; replica lag preventing truncation; crash dumps accumulating; tempdb growth from spills.

**Fix:** free space safely (delete old rotated logs/archives, move dumps, never delete live `.ldf`); take log backup (`BACKUP LOG ... `) and shrink only if needed once; end/kill runaway transaction; raise log level back to Warning; add disk. Serilog rolling:
```csharp
.WriteTo.File("logs/app-.log", rollingInterval: RollingInterval.Day,
    retainedFileCountLimit: 7, fileSizeLimitBytes: 100_000_000, rollOnFileSizeLimit: true)
```
Batch large DML (e.g., 5 000 rows per loop) with log backups in between.

**Prevention:** disk alerts at 70/85%, centralized logging (no local retention), log sampling/levels via config, log-backup job monitoring, pre-sized log file with sensible autogrowth (fixed MB, not %), limits on container logs (`max-size`).

**Interview story:** "Prod app halted: disk 100%. Cause was Debug logging left enabled plus an exception loop writing 40 GB/day. I cleared rotated logs, dropped level to Warning via config, fixed the loop, added rolling retention and a 80% disk alert, and moved logs to central store."

---

## Scenario 19: Data corruption from race condition

**Symptom:** Account balances/inventory counts incorrect (negative stock, lost updates); can't reproduce locally; appears under concurrency.

**First questions:** Which entity/field? Pattern: read-modify-write? Multiple writers (API + job + consumer)? Isolation level? Caches involved? When did it begin?

**Diagnosis steps:**
1. Find invariant violations with SQL: negative stock; `SUM(ledger) <> balance`:
```sql
SELECT a.AccountId, a.Balance, l.LedgerSum
FROM dbo.Account a CROSS APPLY (SELECT SUM(Amount) LedgerSum FROM dbo.Ledger WHERE AccountId=a.AccountId) l
WHERE a.Balance <> l.LedgerSum;
```
2. Reconstruct timeline from audit/temporal tables (`FOR SYSTEM_TIME ALL`), logs, correlation IDs; look for two updates within ms.
3. Review code: `var x = await db.Stock.FindAsync(id); x.Qty -= n; await SaveChangesAsync()` — lost update. Non-atomic multi-step logic; check-then-act; static state; cache read-modify-write.
4. Reproduce: parallel test (`Parallel.ForEachAsync` hitting endpoint 100×) or `NBomber`.

**Root causes:** lost update (read-modify-write without concurrency control); check-then-insert races; non-atomic multi-statement ops without transaction; stale cache used for decisions; two services writing same data; READ COMMITTED allows non-repeatable reads.

**Fix:**
- Atomic SQL: `UPDATE Stock SET Qty = Qty - @n WHERE Id=@id AND Qty >= @n` (check rows affected). EF Core 7+: `ExecuteUpdateAsync(s => s.SetProperty(x => x.Qty, x => x.Qty - n))` with `Where(x => x.Qty >= n)`.
- Optimistic concurrency token:
```csharp
public byte[] RowVersion { get; set; }  // [Timestamp] or .IsRowVersion()
try { await db.SaveChangesAsync(); } catch (DbUpdateConcurrencyException) { /* reload & retry or return 409 */ }
```
- Pessimistic: `SELECT ... WITH (UPDLOCK, ROWLOCK)` inside a transaction, or `sp_getapplock`.
- Constraints: `CHECK (Qty >= 0)`, unique indexes.
- Repair data from ledger/audit; communicate impact.

**Prevention:** model invariants as DB constraints, concurrency tests in CI, append-only ledger + derived balance, single-writer design, code review "read-modify-write" alarm.

**Interview story:** "Inventory went negative on hot items. `Qty -= n` on a loaded entity — two requests loaded the same row. I changed it to a conditional atomic UPDATE with a CHECK constraint, repaired counts from the movement ledger, and added a parallel test that hammers the endpoint."

---

## Scenario 20: Rollback of a bad release

**Symptom:** After deploy, error rate/latency spikes, key business metric drops (checkout conversions); alert fires within minutes.

**First questions:** Is rollback safe (DB schema/backward compat, messages already in new format, feature flags)? Can we mitigate faster (flag off, scale)? Who is incident commander? Blast radius?

**Diagnosis steps (decide quickly):**
1. Confirm correlation with deploy time: dashboards annotated with release markers; errors by `application_Version`.
2. Check rollback constraints: any **non-backward-compatible migration** (dropped/renamed column)? New message schemas published? Data already written in a new format?
3. Choose: (a) flag off, (b) rollback to previous artifact (slot swap back / `kubectl rollout undo deployment/api` / redeploy previous image tag), (c) roll-forward hotfix if rollback unsafe.

**Execution:**
```
kubectl rollout status deployment/api
kubectl rollout undo deployment/api            # or --to-revision=<n>
az webapp deployment slot swap -g rg -n app --slot staging --target-slot production   # swap back
```
Verify: error rate, latency, health endpoints, business KPIs, queue depth. Communicate status.

**Root causes of bad release:** insufficient test coverage; config drift; prod-only data shapes; missing perf test; migration incompatible with old code; dependency upgrade behaviour change.

**Fix/Prevention:** expand/contract migrations (schema compatible with N and N−1); feature flags for risky changes; canary/blue-green with automatic rollback on SLO breach; immutable, versioned artifacts retained; rollback rehearsed and documented; DB backups/point-in-time restore as last resort; post-mortem. Keep message contracts backward compatible (additive changes).

**Interview story:** "Ten minutes after release, 5xx jumped to 8%. Incident channel opened, I confirmed the spike began at deploy time, checked the migration was additive, and used a slot swap to roll back in 3 minutes. Root cause was a null reference on a data shape only in prod. We added that case to tests, enabled canary at 5% with auto-rollback, and made flags mandatory for risky features."

---

*End of E02. Practice tip: pick one design and one scenario per day; explain aloud with a whiteboard sketch, then re-read the follow-up questions and answer each in under 60 seconds.*
