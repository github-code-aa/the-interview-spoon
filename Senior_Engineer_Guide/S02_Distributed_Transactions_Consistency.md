# S02 - Distributed Transactions & Consistency (Senior Engineer Interview Guide)

How to use: read each topic's study notes first, then answer every question aloud before reading the "Strong senior answer". Practise the reasoning order: clarify -> options -> trade-offs -> decision -> risks, and always say "what fails when this fails?".

## Table of Contents
1. ACID vs BASE and why local transactions stop working across services
2. Two-phase commit, MSDTC, XA
3. The dual-write problem, Transactional Outbox and Inbox
4. Idempotency and the exactly-once myth
5. Sagas: orchestration vs choreography
6. Compensation, pivot transactions, timeouts, isolation anomalies, observability
7. Scenario family: "payment succeeded but order failed"
8. CAP, PACELC and consistency models
9. Designing UX and CQRS read models for eventual consistency
10. Message ordering, partitioning, out-of-order events, event versioning
11. Event sourcing
12. Distributed locks and lock-free designs
13. Optimistic concurrency: rowversion and ETags across services
14. Retries, timeouts, poison messages and dead-letter strategy
15. Distributed IDs, clock skew, why timestamps lie
16. Data replication and reference-data caching between services
17. Reconciliation, audit/repair, and testing distributed failure
18. Azure specifics (Service Bus, Durable Functions, Cosmos, Event Grid)
19. Inventory reservation, double-booking, multi-region write conflicts
20. Decision tree: which consistency mechanism for which problem

---

## Topic 1: ACID vs BASE and why local transactions stop working across services

**Study notes**
- ACID: Atomicity, Consistency (invariants), Isolation, Durability. A SQL Server transaction gives this for ONE database (one log, one lock manager).
- BASE: Basically Available, Soft state, Eventually consistent. It is a design posture, not a product: accept temporary inconsistency in exchange for availability and independent scaling.
- Database-per-service means no shared log/lock manager. `BEGIN TRAN` cannot span a SQL write and a Service Bus send, or two services' databases.
- You trade atomic commit for: local transactions + messages + retries + idempotency + compensation + reconciliation.

```
Monolith:   [Order tbl][Payment tbl][Stock tbl]  --one BEGIN TRAN--> all or nothing
Services:   Orders DB   Payments DB   Inventory DB   --no shared tx--> each commits alone;
            consistency is reached over time via messages
```

### 1.1 [Easy] What is the difference between ACID and BASE?
**What the interviewer is really testing**
Whether you see BASE as a trade-off with consequences, not a buzzword.

**Typical mid-level answer**
"ACID is for SQL, BASE is for NoSQL and is eventually consistent." Falls short: ties it to products, not to scope of the transaction or business invariants.

**Strong senior answer**
ACID is a guarantee within a single transactional boundary; BASE describes what you accept once you leave that boundary. I would say the question is really "where do my invariants live?" Invariants that must hold at every instant (account balance not negative) stay inside one aggregate/database under ACID. Cross-service flows become eventually consistent with explicit intermediate states (Pending, Reserved, Confirmed). SQL Server can sit inside a BASE architecture, and Cosmos DB can offer strong consistency in a single region, so it's about boundaries not products. The risk of BASE is that the intermediate states are visible to users and other services, so I design for them explicitly.

**Follow-up traps**
- "Is eventual consistency the same as inconsistent?" No: it converges if all messages are eventually processed; you need bounded lag and reconciliation to prove it.
- "Can BASE systems have transactions?" Yes, local ones, per partition/aggregate.

### 1.2 [Follow-up] Why exactly can't a local transaction span an Order service and a Payment service?
**What the interviewer is really testing**
Depth of understanding of what a transaction manager needs.

**Typical mid-level answer**
"They're different databases." True but shallow.

**Strong senior answer**
Atomic commit requires a single coordinator that can make all participants durable-or-abort together, plus locks held until the decision. Separate services have separate stores, separate failure domains and often different technologies (SQL, Cosmos, Service Bus, a third-party payment API that has no enlistment). Even if we had a coordinator (2PC), we'd couple availability: the transaction commits only if all services are up, and locks are held across network calls. The payment provider is the clearest example: Stripe-like APIs cannot enlist in your transaction at all. So the realistic options are 2PC where every participant supports it (rare), or local commits plus compensations. What fails when this fails: if Payment commits and Order crashes before committing, money moved with no order, so the design must detect and repair.

**Follow-up traps**
- "What about one shared database?" It restores ACID but couples schema and deploys; acceptable for a tightly coupled bounded context, an anti-pattern for independent services.
- "What about TransactionScope?" It escalates to MSDTC only for participants that support it; HTTP calls don't.

### 1.3 [Medium] A teammate proposes "just wrap the DB save and the Service Bus send in a try/catch and retry". What's wrong?
**What the interviewer is really testing**
Recognising the dual-write problem early.

**Typical mid-level answer**
"Retry until both succeed." Falls short: retrying after a crash doesn't exist because the process died between the two calls.

**Strong senior answer**
Clarify the failure windows first. If we save then send and the process dies between them, the DB has the order but no event was published and nobody retries because the in-memory retry died with the process. If we send then save, we publish an event about a state that never committed. try/catch handles exceptions but not crashes, timeouts with unknown outcome, or deploys killing the pod. Options: transactional outbox (my default), change data capture, or event sourcing where the event is the state. I'd choose the outbox with SQL Server because it needs no new infrastructure: write business row and outbox row in one local transaction, a relay publishes with at-least-once semantics, and consumers are idempotent. Risk: duplicates and relay lag, which I monitor.

**Follow-up traps**
- "Can you reorder to send first and make it safe?" Only if consumers tolerate phantom events; usually not.
- "Does Service Bus transaction fix it?" It only makes multiple Service Bus operations atomic, not SQL plus Service Bus.

### 1.4 [Hard] How do you preserve a business invariant like "order total must not exceed customer credit limit" when Orders and Credit are separate services?
**What the interviewer is really testing**
Whether you can redesign boundaries and use reservation patterns rather than hand-wave.

**Typical mid-level answer**
"Call the Credit service synchronously before saving." Falls short: race between check and save, and the Credit call can succeed while Order save fails.

**Strong senior answer**
First ask: how strict must this be? If a breach is merely costly (can be reversed), eventual with compensation is fine; if legally prohibited, the invariant must live in one place. Option A: make Credit the sole owner and have it perform an atomic "reserve credit" (conditional update `UPDATE ... SET Reserved = Reserved + @amt WHERE Limit - Reserved >= @amt`) keyed by order id, idempotently. Option B: move the check into the same aggregate by merging the boundary. Option C: optimistic acceptance with later cancellation. I'd choose A: reservation with expiry, Order moves Pending -> Confirmed on reservation event, and a timeout releases reservations. Risk: orphan reservations if Order dies, handled by expiry and reconciliation.

**Follow-up traps**
- "What if the reserve call times out?" Retry with same reservation id; idempotent on order id.
- "Why not read credit and write locally?" Stale read allows double spend.

### 1.5 [Tricky] "Eventual consistency means we can ignore consistency problems." Respond.
**What the interviewer is really testing**
Maturity: eventual consistency requires more engineering, not less.

**Typical mid-level answer**
Agrees or says "it depends" without specifics.

**Strong senior answer**
The opposite: eventual consistency moves work from the database to my application. I must define the convergence guarantee (every event processed at least once), idempotency, ordering assumptions, compensation, monitoring of lag, and reconciliation to detect permanent divergence. I'd define SLOs such as "99% of order events reflected in the read model within 5 seconds" and alert on outbox age and DLQ depth. The cost of getting it wrong is silent data drift that is found by customers. I'd also be explicit with the business about which screens may show stale data. Failure mode: a dropped event with no reconciliation never converges; "eventual" becomes "never".

**Follow-up traps**
- "How do you prove convergence?" A reconciliation job comparing source of truth to projections.
- "Bounded staleness?" Metrics on lag; for Cosmos, bounded staleness consistency level offers it natively.

**Red flags in answers**
- Equating BASE with "NoSQL" and not mentioning invariants or compensation.
- Proposing try/catch-and-retry as the cross-service atomicity mechanism.
- Saying eventual consistency is "free" or needs no monitoring.

**Cheat sheet**
- ACID is per-boundary; across services use local tx + messages + idempotency + compensation.
- Keep hard invariants inside one aggregate/database.
- Dual write cannot be made atomic by retry logic.
- Name intermediate states (Pending/Reserved/Confirmed) explicitly.
- Define lag SLOs and reconcile.

---

## Topic 2: Two-phase commit, MSDTC, XA

**Study notes**
- 2PC: coordinator asks all participants to PREPARE (write to log, hold locks, vote yes/no); if all yes, coordinator logs COMMIT and tells all; otherwise ABORT.
- MSDTC is Windows' coordinator (OLE-Tx/ XA support through bridges); .NET `TransactionScope` escalates a lightweight transaction to a distributed one when a second durable resource enlists. XA is the X/Open standard used by Java/JMS and many databases.
- Blocking: if the coordinator dies after participants voted yes, they are "in doubt" and hold locks until it recovers.

```
Coordinator        Participant A         Participant B
   |--PREPARE------->|                      |
   |--PREPARE---------------------------->  |
   |<--VOTE YES------|                      |
   |<--VOTE YES-----------------------------|
   |--log COMMIT (decision point)           |
   |--COMMIT-------->|                      |
   |--COMMIT------------------------------->|
```

### 2.1 [Easy] Explain two-phase commit.
**What the interviewer is really testing**
Clear, correct description including the decision point and what is durable.

**Typical mid-level answer**
"Prepare then commit." Misses the logs, locks held, and in-doubt state.

**Strong senior answer**
In phase one each participant durably logs enough to commit later, acquires and holds its locks, and votes. After it votes yes it cannot unilaterally abort. In phase two the coordinator logs the global decision and broadcasts it. The guarantee is atomicity across participants. The cost is latency (multiple round trips and forced log writes), lock duration extended across network calls, and blocking if the coordinator is lost between phases. What fails when this fails: coordinator crash after votes leaves participants in doubt, holding locks; recovery depends on the coordinator's log being available.

**Follow-up traps**
- "Is 2PC fault tolerant?" Not for coordinator failure; 3PC/Paxos-commit address it but aren't common.
- "Who decides if a participant crashes in prepare?" Coordinator times out and aborts.

### 2.2 [Follow-up] Why is 2PC generally avoided in cloud-native systems?
**What the interviewer is really testing**
Ability to give reasons beyond "it's slow".

**Typical mid-level answer**
"It doesn't scale and it's slow." Not specific.

**Strong senior answer**
Five reasons. (1) Availability: the transaction needs all participants up, so overall availability is the product of each one's. (2) Locks held across network hops cause contention and deadlocks under load. (3) Participant support: Service Bus, Cosmos DB, HTTP APIs, SaaS payment providers and most PaaS services don't expose an enlistable resource manager. (4) Coordinator is an operational single point and needs a stable recoverable log (MSDTC is Windows-bound, awkward in containers/Kubernetes). (5) It couples teams' deploy and failure domains. Azure SQL Database does offer elastic database transactions and Managed Instance supports distributed transactions across instances, but these are limited to SQL participants. I'd choose messaging + outbox + saga, accepting eventual consistency.

**Follow-up traps**
- "Does that mean never use it?" No, see 2.3.
- "Does .NET TransactionScope work in cloud?" Escalation needs MSDTC availability; avoid accidental escalation by opening one connection and not mixing resources.

### 2.3 [Medium] Where is 2PC still acceptable?
**What the interviewer is really testing**
Pragmatism rather than dogma.

**Typical mid-level answer**
"Never." Overly rigid.

**Strong senior answer**
When all participants are in one trust and failure domain, support XA/MSDTC, transactions are short, volume is moderate, and the alternative's complexity is higher. Examples: an on-prem line-of-business app updating two SQL Server databases on the same instance or linked servers; a legacy migration where wrapping in a saga is not worth it; a SQL Server + MSMQ system. Cross-database transactions within one SQL Server instance actually don't need MSDTC in many cases (they commit via the same log manager semantics), and within Azure SQL Database elastic transactions can be used for small scale. I'd still document the limits and set a plan to remove it. Risk: a team adds a third non-enlisting participant and silently loses atomicity.

**Follow-up traps**
- "Can you mix a SQL write and an HTTP call under TransactionScope?" The HTTP call doesn't enlist; the rollback won't undo it.
- "Does Service Bus participate?" Not in MSDTC with SQL; it has its own local transactions.

### 2.4 [Hard] Your legacy system uses MSDTC between SQL Server and a message queue. You're moving it to Azure. Plan the migration.
**What the interviewer is really testing**
Practical migration planning with risk management.

**Typical mid-level answer**
"Replace with Service Bus and sagas." Misses incremental rollout and in-flight transactions.

**Strong senior answer**
Inventory first: which transactions rely on DTC, their volume, and what invariants they protect. Then for each, choose: collapse into one database if the data belongs together, or introduce outbox + idempotent consumer. Migrate incrementally: add the outbox table in the same DB, publish through a relay to Service Bus in parallel with the old path, compare outputs (shadow mode), then cut over. Keep message contracts stable and add dedupe on consumers since at-least-once replaces exactly-once-within-DTC. Plan for in-flight transactions to drain. Failure: duplicates the legacy path never produced; consumers must be made idempotent before cutover. Roll back by flipping a feature flag to the old path.

**Follow-up traps**
- "Why shadow mode?" Detect divergence without risk.
- "What did DTC give that you now lose?" Atomic commit and effectively-once delivery; replaced by idempotency.

### 2.5 [Tricky] A participant voted YES then the coordinator crashed. Describe what happens and how you'd operate it.
**What the interviewer is really testing**
Understanding of in-doubt transactions and ops.

**Typical mid-level answer**
"It rolls back." Wrong: after a yes vote it cannot abort unilaterally.

**Strong senior answer**
The participant is in doubt: it holds locks and waits for the coordinator's decision. Other sessions touching those rows block. On recovery the coordinator reads its log and resends the decision. If the coordinator's log is lost, an operator must manually resolve (in SQL Server `KILL` with the UOW GUID and MSDTC admin commit/abort) knowing that a wrong guess breaks atomicity. Operationally: monitor in-doubt transactions, keep short timeouts, and place the coordinator on durable HA storage. This operational risk is a main reason cloud-native designs avoid 2PC.

**Follow-up traps**
- "Heuristic decisions?" Participant-forced outcomes risk heuristic mixed results.
- "How reduce window?" Fewer participants, shorter transactions.

**Red flags in answers**
- "2PC is just slow" with no mention of blocking/in-doubt state.
- Believing Service Bus/Cosmos/HTTP can enlist in MSDTC with SQL.
- Dismissing 2PC entirely without any valid use.

**Cheat sheet**
- Prepare (vote, hold locks) -> decision logged -> commit.
- Blocking in-doubt state if coordinator dies.
- Cloud PaaS mostly lacks enlistment; availability multiplies.
- OK: few, homogeneous, short, in one failure domain.
- Replacement: outbox + idempotent consumers + saga.

---

## Topic 3: The dual-write problem, Transactional Outbox and Inbox

**Study notes**
- Dual write: updating two systems (DB + broker) without a shared transaction; any crash between them leaves divergence.
- Outbox: in the SAME local transaction, write the business change and an outbox row. A relay reads unsent rows and publishes, then marks them sent. Delivery is at-least-once.
- Relay options: polling publisher (simple, adds latency/DB load) vs CDC/log tailing (SQL Server CDC / Change Tracking, Debezium; lower latency, more infrastructure).
- Inbox: the consumer records processed MessageIds in the same local transaction as its side effects, to dedupe.

```
App tx:  [UPDATE Orders] + [INSERT Outbox(Id, Type, Payload, CreatedAt)] --COMMIT-->
Relay:   SELECT unsent -> publish to Service Bus (MessageId = Outbox.Id) -> mark sent
Consumer: BEGIN; INSERT Inbox(MessageId) (PK; dup => skip); do work; COMMIT; Complete()
```

```sql
CREATE TABLE dbo.Outbox (
  Id UNIQUEIDENTIFIER NOT NULL PRIMARY KEY,
  Seq BIGINT IDENTITY(1,1) NOT NULL,
  AggregateId NVARCHAR(64) NOT NULL,
  Type NVARCHAR(200) NOT NULL,
  Payload NVARCHAR(MAX) NOT NULL,
  CreatedUtc DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
  SentUtc DATETIME2 NULL
);
CREATE INDEX IX_Outbox_Unsent ON dbo.Outbox(Seq) WHERE SentUtc IS NULL;
```

### 3.1 [Easy] What is the dual-write problem?
**What the interviewer is really testing**
Can you articulate the failure windows precisely.

**Typical mid-level answer**
"DB and queue can get out of sync." Doesn't enumerate failure points.

**Strong senior answer**
Two orders of operation, both broken. DB first then publish: crash/timeout in between means a committed change with no event, and downstream never learns. Publish first then DB: the event announces a change that rolled back. Even "success" can be ambiguous: a send that times out may or may not have been accepted. Fixes: outbox, CDC, or event sourcing, all of which reduce it to a single write to one system and derive the other. What fails when this fails: silent divergence, found later by users. So I also add reconciliation.

**Follow-up traps**
- "What about publish inside the DB transaction before commit?" Commit can still fail after publish.
- "Idempotent retry fix it?" Only if the retry survives the crash, which it doesn't.

### 3.2 [Follow-up] Walk through implementing the transactional outbox on SQL Server + Service Bus in .NET.
**What the interviewer is really testing**
Concrete implementation judgement: concurrency of relays, marking, MessageId.

**Typical mid-level answer**
"Insert into an outbox table and have a background job send." Misses concurrency, ordering and dup handling.

**Strong senior answer**
In the handler: one `DbContext` transaction saves the aggregate and adds the outbox row (SaveChanges commits both atomically). A `BackgroundService` relay claims rows with `SELECT TOP (n) ... WITH (UPDLOCK, READPAST, ROWLOCK)` ordered by Seq so multiple relay instances don't fight, sends to Service Bus with `MessageId = Outbox.Id` (so Service Bus duplicate detection can drop relay-retry duplicates within the window), then sets SentUtc. If the relay crashes after send before marking, the row is resent: duplicate, hence consumers must be idempotent. Alternatively MassTransit's EF Core bus outbox does this for you. Risks: outbox growth (cleanup), relay lag (alert on oldest unsent age), poison payloads (park after N failures).

```csharp
await using var tx = await db.Database.BeginTransactionAsync(ct);
order.Confirm();
db.Outbox.Add(new OutboxMessage(Guid.NewGuid(), "OrderConfirmed", Json(order.ToEvent())));
await db.SaveChangesAsync(ct);
await tx.CommitAsync(ct);
```

**Follow-up traps**
- "Why READPAST?" Skip rows locked by another relay instead of blocking.
- "Sent flag vs delete?" Delete/archive keeps table small; flag aids auditing; purge on a schedule.

### 3.3 [Medium] Polling publisher or CDC? How do you choose?
**What the interviewer is really testing**
Trade-off reasoning under real constraints.

**Typical mid-level answer**
"CDC is better because it's real time." Ignores operational cost.

**Strong senior answer**
Clarify throughput, latency target, team skills and infrastructure. Polling every 100-500 ms is simple, works anywhere, easy to debug, but adds read load, has latency floor, and ordering requires care with multiple instances. CDC reads the transaction log: low latency, no polling load, preserves commit order per table, but needs infrastructure (Debezium + Kafka Connect, or Azure-native options), SQL Server CDC/Change Tracking enabled, schema-change handling and more operational expertise. For a typical line-of-business app with a few hundred msgs/sec, I'd start with polling and measure. For very high volume or sub-100 ms requirements, CDC. Risk with polling: long-running transactions make identity (Seq) values visible out of commit order, so a relay may skip a row committed later with a lower Seq.

**Follow-up traps**
- "Why can Seq order mislead?" Identity values are allocated before commit; a lower Seq may commit after a higher one. Use a re-scan of unsent rows rather than a high-water mark.
- "CDC delivers exactly-once?" No, at-least-once on restart.

### 3.4 [Hard] How do you handle ordering, duplicates and cleanup with the outbox?
**What the interviewer is really testing**
Edge-case thinking.

**Typical mid-level answer**
"Use timestamps for ordering, delete after sending." Timestamps lie; deleting loses audit.

**Strong senior answer**
Ordering: global order is rarely needed; per-aggregate order is. The relay processes rows per AggregateId in Seq order and sets Service Bus `SessionId = AggregateId` (session-enabled queue) or partition key so the broker keeps that order; consumers also carry a version number so they can detect gaps. Duplicates: relay at-least-once, so use MessageId = outbox Id for Service Bus duplicate detection (limited window) and an inbox/dedupe table in consumers for the real guarantee. Cleanup: purge sent rows older than N days in batches (`DELETE TOP (5000)`), partition by date for cheap switch-out, and keep a monitored "oldest unsent" metric. Failure: a relay that stops silently; alert on age.

**Follow-up traps**
- "Is Service Bus duplicate detection enough?" No, bounded window and only on MessageId.
- "Big purge blocks?" Batch with small transactions or partition switching.

### 3.5 [Tricky] Explain Inbox and when outbox alone is not enough.
**What the interviewer is really testing**
That the consumer side also has a dual write (side effect + ack).

**Typical mid-level answer**
"Consumer just retries." Ignores that processing and `Complete()` are two actions.

**Strong senior answer**
The consumer does side effect then acknowledges. Crash between them causes redelivery and a repeated side effect. Inbox: store MessageId in the same transaction as the side effect; the unique key makes the second attempt a no-op, and then we complete the message. This yields effectively-once processing for database effects. For non-transactional effects (send email, call provider) the inbox cannot make them atomic: use the provider's idempotency key and record status before/after. MassTransit offers an EF inbox. Risk: inbox growth, handled by retention longer than max redelivery/DLQ-replay period.

```sql
BEGIN TRAN;
INSERT INTO dbo.Inbox(MessageId, ReceivedUtc) VALUES (@MessageId, SYSUTCDATETIME()); -- PK violation => already processed
-- apply business change
COMMIT;
```

**Follow-up traps**
- "Retention?" At least longer than any possible redelivery, including DLQ resubmit.
- "Catch PK violation as success?" Yes, then complete the message.

**Red flags in answers**
- Publishing to the broker inside the DB transaction and calling it atomic.
- Using DateTime ordering or a high-water mark without addressing uncommitted-gap issue.
- No idempotent consumer because "the outbox guarantees once".

**Cheat sheet**
- Business row + outbox row in one local transaction.
- Relay is at-least-once; MessageId = outbox Id.
- Order per aggregate (session/partition key), not globally.
- Inbox/dedupe table in consumer's transaction.
- Monitor oldest-unsent age; purge in batches.

---

## Topic 4: Idempotency and the exactly-once myth

**Study notes**
- Idempotent: applying an operation N times has the same effect as once. (`SET balance=5` is; `balance += 5` is not.)
- Exactly-once delivery across a network is not achievable in general; practical goal is at-least-once delivery + idempotent processing = effectively-once effects.
- Keys: natural (business identity, e.g. OrderId + step) vs surrogate (client-generated GUID per attempt/command). Same logical command must reuse the same key across retries.
- Dedupe store: unique constraint in the same transaction as the effect; TTL longer than retry horizon.
- HTTP: `Idempotency-Key` header (IETF draft; used by Stripe-style APIs); store key, request fingerprint and response.

```
Client --POST (Key=K)--> API: INSERT IdemKeys(K,status=InProgress) ok -> do work -> store response
Client --POST (Key=K) retry--> API: K exists: Completed => return stored response; InProgress => 409/wait; hash differs => 422
```

### 4.1 [Easy] What is idempotency and why does it matter in messaging?
**What the interviewer is really testing**
Basics plus link to at-least-once delivery.

**Typical mid-level answer**
"Same request twice gives the same result." Doesn't link to why duplicates occur.

**Strong senior answer**
Brokers like Service Bus deliver at-least-once: if my handler finishes but the lock expires or the process dies before `Complete`, the message is redelivered. Clients also retry on timeouts without knowing if the first attempt succeeded. So duplicates are normal, not exceptional. Making operations idempotent turns "retry safely" into the universal answer to uncertainty. Preferred approaches: naturally idempotent operations (upsert/set), else a dedupe key stored atomically with the effect. What fails when this fails: double charges, duplicate emails, doubled stock decrements.

**Follow-up traps**
- "Is DELETE idempotent?" Effect-wise yes, response may differ (404 on second).
- "Is POST idempotent?" Not by default; add keys.

### 4.2 [Follow-up] Natural key vs surrogate/idempotency key: how do you choose?
**What the interviewer is really testing**
Practical key design.

**Typical mid-level answer**
"Use a GUID." Doesn't say who generates it nor what scope.

**Strong senior answer**
A natural key (OrderId + operation, or provider transaction id) is deterministic: two independent attempts to do the same business thing collide, which is what I want, and it needs no extra storage if a unique constraint already exists (e.g. one payment per OrderId). A surrogate key is generated by the caller per intended action, and is needed when the same business entity can legitimately have several actions (two refunds on one order). It must be created once per user intent and persisted before the first send, otherwise every retry makes a new one and defeats the purpose. Scope the key by tenant/user and operation, store a request hash to detect misuse. Risk: key reuse across different payloads; reject with 422.

**Follow-up traps**
- "Generate key in the server on retry?" Defeats the purpose; client/producer must persist it.
- "Key collisions?" Scope by tenant+operation; GUIDs make accidental collision negligible.

### 4.3 [Medium] Design idempotency for an HTTP POST /payments endpoint in ASP.NET Core.
**What the interviewer is really testing**
Handling concurrency, in-progress state, response replay, and partial failure.

**Typical mid-level answer**
"Check a table for the key first." Race between check and insert; no in-progress state.

**Strong senior answer**
Require an `Idempotency-Key` header. Insert a row with unique (TenantId, Key) as the very first step: the unique constraint, not a prior SELECT, arbitrates concurrent duplicates. States: InProgress, Completed(with stored status and body). A duplicate that finds Completed returns the stored response; InProgress returns 409 with Retry-After; a different request hash returns 422. Do the business effect and mark Completed in the same transaction where possible. When the effect is an external call (card provider), pass our key downstream and persist the outcome; on crash with InProgress, a recovery path re-queries the provider by key. Expire keys after 24h-7d per business retry horizon. Risk: stuck InProgress rows, so add a lease/timeout.

```sql
INSERT INTO dbo.IdempotencyKey(TenantId,[Key],RequestHash,Status,CreatedUtc)
VALUES(@t,@k,@h,'InProgress',SYSUTCDATETIME());  -- unique(TenantId,[Key]); dup => lookup existing
```

**Follow-up traps**
- "Where to implement?" Middleware/filter, but business-level dedupe still needed for message paths.
- "What if provider doesn't support keys?" Query by our reference first, or use natural unique reference.

### 4.4 [Hard] "Our broker offers exactly-once, so we don't need idempotent consumers." Respond.
**What the interviewer is really testing**
Seeing through marketing claims and understanding scope.

**Typical mid-level answer**
Accepts the claim or just says it's impossible.

**Strong senior answer**
Clarify scope: such claims (Kafka transactions, Service Bus duplicate detection) are exactly-once within a boundary: e.g. producer-to-topic dedupe or consume-process-produce within Kafka. Service Bus duplicate detection removes duplicates of sent messages by MessageId within a configured time window; it does not prevent redelivery to a consumer after a lock timeout, and it doesn't cover side effects in my SQL database or in third-party APIs. The moment my handler touches anything outside the broker, a crash between effect and ack gives repeats. So I'd treat broker features as an optimisation and keep an inbox/natural idempotency for the real guarantee. Failure: the dedupe window expires and a late retry produces a duplicate.

**Follow-up traps**
- "Service Bus duplicate detection window?" Configurable; default 10 minutes, up to 7 days; set at queue/topic level.
- "Then what's 'effectively once'?" At-least-once delivery + idempotent effects.

### 4.5 [Scenario] A consumer decrements stock on `OrderPlaced`. Duplicates caused stock to go negative. Fix it.
**What the interviewer is really testing**
Turning a non-idempotent operation into an idempotent one and repairing data.

**Typical mid-level answer**
"Add a check that stock is not below zero." Treats symptom only.

**Strong senior answer**
`stock -= qty` is not idempotent. Make it so: record a reservation row keyed by (OrderId, Sku) with unique constraint, and insert it in the same transaction as the decrement; a duplicate violates the key and is skipped. Better, derive stock from reservations rather than mutating a counter blindly. For repair, reconcile: recompute expected stock from the ledger of unique reservations versus the counter, correct, and audit. Also add a guard `WHERE Stock >= @qty` for oversell protection, but I'd emphasise it is a separate concern from idempotency. Risk: backfilling dedupe for historical messages; run the reconciliation once and then enforce the key going forward.

```sql
BEGIN TRAN;
INSERT INTO dbo.StockReservation(OrderId,Sku,Qty) VALUES(@o,@s,@q); -- PK(OrderId,Sku)
UPDATE dbo.Stock SET Qty=Qty-@q WHERE Sku=@s AND Qty>=@q;
IF @@ROWCOUNT=0 THROW 50001,'Insufficient stock',1;
COMMIT;
```

**Follow-up traps**
- "Why both checks?" Idempotency stops repeats; the guard enforces the business invariant.
- "How would you find affected SKUs?" Compare ledger sum to counter.

**Red flags in answers**
- Claiming the broker gives exactly-once end to end.
- Check-then-insert dedupe without a unique constraint.
- Generating a fresh idempotency key on each retry.

**Cheat sheet**
- At-least-once + idempotent processing = effectively-once.
- Unique constraint arbitrates concurrent duplicates.
- Persist the key before the first attempt; scope it; hash the request.
- Dedupe state in the same transaction as the effect.
- Retention longer than the maximum redelivery window.

---

## Topic 5: Sagas: orchestration vs choreography

**Study notes**
- Saga: a sequence of local transactions; each publishes an event/command to trigger the next; failures are handled by compensating transactions (semantic undo), not rollback.
- Choreography: services react to each other's events; no central brain. Orchestration: a coordinator (state machine) sends commands and handles replies.
- .NET tooling: MassTransit saga state machines (persisted in EF/SQL, Mongo, etc.), Durable Functions orchestrators (deterministic, event-sourced replay), NServiceBus sagas.

```
Orchestrated:  Orchestrator --ReserveStock--> Inventory --StockReserved--> Orchestrator
               Orchestrator --ChargeCard---> Payment   --PaymentFailed--> Orchestrator
               Orchestrator --ReleaseStock-> Inventory (compensation)
Choreographed: OrderPlaced -> Inventory -> StockReserved -> Payment -> PaymentFailed -> Inventory(release)
```

```csharp
// MassTransit-style state (shape only)
public class OrderState : SagaStateMachineInstance {
  public Guid CorrelationId { get; set; }
  public string CurrentState { get; set; }
  public Guid? PaymentId { get; set; }
  public DateTime? ExpiresUtc { get; set; }
  public byte[] RowVersion { get; set; }   // optimistic concurrency for saga instance
}
```

### 5.1 [Easy] What is a saga and why use one?
**What the interviewer is really testing**
Clear definition including compensation and lack of isolation.

**Typical mid-level answer**
"A chain of steps with rollback." Rollback is wrong: compensations are new business actions.

**Strong senior answer**
A saga replaces one distributed ACID transaction with a series of local transactions, each with a defined compensating action for steps already done. There's no rollback: a refund is a new transaction, visible to the customer, and may itself fail. Sagas are atomic only in the eventual sense (all steps complete or compensated) and lack isolation: other transactions see intermediate states. I use them when a business process spans services and I can express "undo" in business terms. What fails when this fails: a failing compensation must be retried until done and escalated to a human if it can't.

**Follow-up traps**
- "Is the saga atomic?" Eventually; not isolated.
- "What if compensation fails?" Retry with backoff, then alert and manual queue.

### 5.2 [Follow-up] Orchestration vs choreography: trade-offs and your default?
**What the interviewer is really testing**
Decision-making, not preference.

**Typical mid-level answer**
"Choreography is more decoupled, orchestration is centralized." Doesn't say when each bites.

**Strong senior answer**
Choreography has no central component and is good for 2-3 step flows with stable ownership, but the workflow is implicit across services: hard to see, easy to create cyclic dependencies, and adding a step touches multiple services. Orchestration makes the process explicit, testable, observable, with timeouts and compensation logic in one place, at the cost of a coordinator that can become a "god service" if business logic leaks into it. My default is orchestration once a flow has more than ~3 steps, compensations, or timeouts; choreography for simple fan-out notifications. Risk with orchestration: the orchestrator must be highly available and idempotent; it only coordinates, domain rules stay in services.

**Follow-up traps**
- "Single point of failure?" State persisted; run multiple instances with optimistic concurrency on saga rows.
- "Can you mix?" Yes: orchestrate the core, choreograph side effects like notifications.

### 5.3 [Medium] Design the saga state machine for place-order: reserve stock, charge payment, ship.
**What the interviewer is really testing**
States, events, transitions, timeouts and compensations.

**Typical mid-level answer**
Lists steps only, no failure paths or timeouts.

**Strong senior answer**
States: Submitted -> StockReserving -> PaymentPending -> Confirmed -> Shipping -> Completed, plus Compensating and Failed/Cancelled. Events/commands: ReserveStock, StockReserved/Failed, ChargePayment, PaymentSucceeded/Failed, ReleaseStock, ShipOrder. On PaymentFailed: send ReleaseStock, then go Cancelled. Each state has a timeout (schedule a message or Durable timer) so a lost reply doesn't hang forever. Every command carries the saga CorrelationId as the idempotency key; every transition is idempotent (receiving StockReserved twice in PaymentPending is ignored). Saga instance persisted with a version column; MassTransit retries on concurrency conflicts. Risk: ordering of ShipOrder vs compensation; define the pivot (after payment succeeds we go forward only).

**Follow-up traps**
- "Duplicate reply events?" State machine ignores events not valid in the current state, or handles idempotently.
- "Timeout fires after reply arrives?" Check state before acting; timeouts are hints.

### 5.4 [Hard] When would you pick Durable Functions over MassTransit sagas, and what are the pitfalls?
**What the interviewer is really testing**
Knowledge of the tools' models and constraints.

**Typical mid-level answer**
"Durable Functions is serverless so simpler." Ignores determinism and replay.

**Strong senior answer**
Durable Functions writes the workflow as code (`await context.CallActivityAsync`), persists history in Azure Storage (or Netherite/MSSQL backends) and replays the orchestrator from history to rebuild state. That is great for sequential/fan-out flows, human approval with `WaitForExternalEvent`, and timers, with low infrastructure effort on Azure. The cost: orchestrator code must be deterministic (no DateTime.Now, Guid.NewGuid, I/O; use `context.CurrentUtcDateTime`), versioning in-flight instances when code changes is tricky, and history growth requires `ContinueAsNew` for long-lived loops. MassTransit state machines are event-driven, run in any host (AKS, App Service), store state in SQL with optimistic concurrency and fit systems already built on messages. I'd choose Durable Functions for Azure-native, activity-style workflows; MassTransit for message-centric domain sagas needing correlation of many events. Failure: both need idempotent activities/consumers.

**Follow-up traps**
- "Activity retried?" At-least-once; activities must be idempotent.
- "Deploy while instances running?" Breaking orchestrator changes cause non-determinism errors; version or drain.

### 5.5 [Tricky] Your orchestrator crashes after sending ChargePayment but before persisting the new state. What happens?
**What the interviewer is really testing**
Dual write inside the orchestrator itself.

**Typical mid-level answer**
"It will retry." Doesn't realise the command may already be sent.

**Strong senior answer**
The orchestrator has its own dual-write problem: state change and command send. If it persists state then sends and crashes, the command is never sent, so the saga hangs: fixed by timeouts or by using an outbox for the orchestrator (MassTransit's transactional outbox stores the state and outgoing messages together). If it sends then crashes before persisting, on redelivery of the triggering event it sends the command again, so the Payment service must be idempotent on the saga CorrelationId. In Durable Functions the framework checkpoints history and replays, and activities may be executed more than once. So the answer is: outbox in the saga + idempotent receivers + timeouts. Risk: no timeout means permanent hang; I'd also have a stuck-saga query alert.

**Follow-up traps**
- "How find stuck sagas?" Query instances in non-terminal state older than SLA.
- "Does outbox remove duplicates?" No, it removes lost messages; duplicates still possible.

**Red flags in answers**
- Calling compensation "rollback" and assuming it can't fail.
- Putting domain rules inside the orchestrator.
- No timeout or stuck-saga handling.

**Cheat sheet**
- Saga = local transactions + compensations; no isolation.
- Orchestrate when >3 steps, timeouts, compensations; choreograph simple fan-out.
- Every command/event idempotent, correlated by saga id.
- Persist saga with optimistic concurrency; use outbox for state+messages.
- Durable Functions: deterministic orchestrators; MassTransit: event-driven state machines.

---

## Topic 6: Compensation, pivot transactions, timeouts, isolation anomalies, observability

**Study notes**
- Step types (Richardson): compensatable (can undo), pivot (point of no return: go forward only), retriable (must eventually succeed after pivot).
- Compensation is semantic: cancel booking, refund, credit note. Must be idempotent and retryable; it can't "fail permanently" without human fallback.
- Non-compensatable: emails sent, goods shipped, third-party irreversible calls. Put them AFTER the pivot or replace with a tentative state (hold, authorise-not-capture).
- Isolation anomalies: lost updates, dirty reads (seeing uncommitted saga state), non-repeatable reads. Countermeasures: semantic locks (status flags like PENDING), commutative updates, reread value, version checks, pessimistic view.

```
Order of steps:  [compensatable] -> [compensatable] -> PIVOT -> [retriable] -> [retriable]
Example:         Reserve stock -> Authorise card -> Capture(pivot) -> Ship -> Email
```

### 6.1 [Easy] What is a compensating action and what makes a good one?
**What the interviewer is really testing**
Semantic undo, idempotency and business awareness.

**Typical mid-level answer**
"The reverse operation." Ignores that reversal may not exist or be exact.

**Strong senior answer**
A compensation is a business action that semantically offsets a prior committed step: release the reservation, void the authorisation, issue a refund. It must be idempotent (may run multiple times), commutative with late arrivals (compensation could arrive before the original action: record it so the late original becomes a no-op), and retried until success. It may not restore the exact prior state (fees, notifications already sent), so I'd agree with the business on what "undone" means. What fails when this fails: stuck money or stock; route to a manual-intervention queue with full context.

**Follow-up traps**
- "Compensation arrives before the action?" Store a tombstone keyed by the saga/action id so the late action is ignored.
- "Can you compensate a compensation?" Avoid; model forward-only recovery.

### 6.2 [Follow-up] What are pivot transactions and how do they shape saga ordering?
**What the interviewer is really testing**
Designing step order to minimise irreversible risk.

**Typical mid-level answer**
Unfamiliar or says "the last step".

**Strong senior answer**
The pivot is the step after which the saga can no longer be undone and is committed to complete: it either is the last compensatable step's successor or the first irreversible one. Steps before it must be compensatable; steps after it must be retriable (guaranteed eventually to succeed). Design implication: put risky, failure-prone and easily reversible steps first (validate, reserve, authorise), put the pivot (capture payment) when everything that could reasonably fail is done, and put irreversible actions (ship, email) after it. After the pivot we only move forward: retry forever, then escalate to a human, never "cancel". Risk: a step I thought was retriable (shipping partner rejects the address) isn't; so I move validation before the pivot.

**Follow-up traps**
- "Authorise vs capture?" Authorise is compensatable (void); capture is the pivot.
- "What if post-pivot step fails permanently?" Manual remediation and customer communication, perhaps a refund as a new business decision.

### 6.3 [Medium] How do you design timeouts in a saga?
**What the interviewer is really testing**
Handling silence, the commonest distributed failure.

**Typical mid-level answer**
"Set HTTP timeouts." That's per-call, not per-saga.

**Strong senior answer**
Each waiting state has a deadline: PaymentPending for 15 minutes, say. Implementation: schedule a timeout message (Service Bus `ScheduledEnqueueTime` or MassTransit schedule) or Durable Functions `CreateTimer` raced with `WaitForExternalEvent` using `Task.WhenAny`. When it fires, the handler re-checks current state: if still waiting, compensate or escalate; if already moved on, ignore. Distinguish "no reply" from "failure reply": a timeout means unknown outcome, so query the downstream by business key before compensating (don't refund a payment that actually succeeded). Cancel the Durable timer when the event wins to avoid lingering instances. Risk: very short timeouts cause false compensation, very long ones hold reservations; pick them from measured p99 plus business tolerance.

**Follow-up traps**
- "Timeout and success reply race?" State check plus idempotent handling; last guard is reconciliation.
- "Clock reliance?" Timers fire at-least-roughly; don't rely on exactness.

### 6.4 [Hard] How do you deal with the lack of isolation in sagas, i.e. other transactions seeing half-finished state?
**What the interviewer is really testing**
Knowing semantic countermeasures.

**Typical mid-level answer**
"Eventual consistency, so it's fine." Misses concrete anomalies.

**Strong senior answer**
Identify anomalies: a lost update when two sagas modify the same order; a dirty read when a UI shows Reserved stock that may be released. Countermeasures: semantic lock (an `OrderStatus = PendingApproval` flag that other operations respect, e.g. blocking edit/cancel while a saga is in flight), commutative updates (add/subtract deltas rather than set), pessimistic view (reorder steps so risky ones come first so that visible data is less likely to be retracted), reread value before overwrite, version checks (rowversion/ETag), and by-value decisions (high-risk operations use a stricter mechanism). Semantic locks must have expiry to avoid permanent locks if the saga dies. Trade-off: each adds complexity, so I apply them only on the contested aggregates.

**Follow-up traps**
- "Semantic lock deadlocks?" Possible; use timeouts and ordering.
- "Why not DB locks?" Can't span services and would hold across async waits.

### 6.5 [Scenario] A saga stuck in "Compensating" for 3 days. How do you investigate and prevent it?
**What the interviewer is really testing**
Observability and operations.

**Typical mid-level answer**
"Check logs." No systematic approach.

**Strong senior answer**
Start from correlation: every message carries CorrelationId/saga id and W3C trace context, so I can find the instance and its message trail in Application Insights. Check saga row state, last transition time, retries, and DLQ for the compensation command. Typical causes: compensation consumer poison message, downstream permanent 4xx, schema mismatch, or an expired lock. Fix the cause, then replay from DLQ (idempotent consumers make that safe). Prevention: saga state dashboard (count by state and age), alert on instances older than SLA, DLQ alerts, structured events for each transition (queryable timeline), runbook with manual resolve command, and an audit trail of manual actions. Risk: manual fix without recording leaves reconciliation confused.

**Follow-up traps**
- "What metrics?" Time in state, completion rate, compensation rate, DLQ depth.
- "Replay safe?" Only if consumers are idempotent.

**Red flags in answers**
- Putting irreversible steps (email, shipping) before the pivot.
- Treating timeout as failure without checking actual downstream state.
- No visibility plan for in-flight sagas.

**Cheat sheet**
- Order: compensatable -> pivot -> retriable.
- Compensations: idempotent, retried, tombstone for late arrivals.
- Timeout = unknown outcome; query before compensating.
- Semantic locks, commutative updates, versions for isolation.
- Observe: correlation id, state/age dashboards, DLQ alerts.

---

## Topic 7: Scenario family: "payment succeeded but order failed"

**Study notes**
- Root pattern: two systems, two commits, a failure between. Payment provider is authoritative for money; your DB is authoritative for the order.
- Ordering choices: (a) order first as Pending, then pay (safer: no money without an order record); (b) pay then create order (risk of orphan payment).
- Recovery tools: payment webhooks, idempotent order creation by payment id, reconciliation against provider reports, refunds as compensation.

```
Client -> Orders: create Order(Pending, id=O1) [committed]
Orders -> Provider: Charge(ref=O1, idempotency=O1)
Provider -> Orders: webhook PaymentSucceeded(O1)  (at-least-once)
Orders: Pending -> Paid (idempotent)  | crash here => webhook retry / reconciliation repairs
```

### 7.1 [Scenario] The card was charged but your Order DB write failed. Customer sees an error. What now?
**What the interviewer is really testing**
Systematic reasoning about unknown/partial outcomes and customer impact.

**Typical mid-level answer**
"Refund the payment." Reasonable but premature and not systematic.

**Strong senior answer**
First clarify: why did the write fail (transient, constraint, outage)? Is it retriable? Prefer forward recovery: if the failure is transient, complete the order using the payment as the source (payment reference is a natural idempotency key) rather than refund and annoy the customer. Design to avoid the problem: create Order as Pending before charging, so a payment always maps to an order id; the provider's webhook (not the synchronous response) drives Pending -> Paid. If the order truly can't be fulfilled (item gone), refund automatically via compensation with an idempotency key and notify the customer. Add a reconciliation job comparing provider charges to orders; unmatched charges go to a queue. Risk: refunding while the order is simultaneously completed by a retry; guard with state checks and the refund idempotency key.

**Follow-up traps**
- "Customer retries the checkout?" Idempotency key prevents a second charge.
- "Can you roll back the charge?" Only void if authorised, not captured; else refund.

### 7.2 [Follow-up] Which order is better: charge then create order, or create order then charge?
**What the interviewer is really testing**
Trade-off analysis with failure modes.

**Typical mid-level answer**
"Charge first so we're sure of money." Ignores orphan payments.

**Strong senior answer**
Order-first (Pending) gives a durable record before any money moves; failure modes are benign: abandoned Pending orders expire. Charge-first risks money taken with no record if the next step fails. A hybrid is best: create Pending order, authorise (not capture) payment, reserve stock, then capture when fulfilment is certain, using capture as the pivot. Authorisations can be voided at no cost to the customer (though authorisation holds exist and expire in days), which keeps compensation simple. Cost: Pending orders clutter and need expiry. What fails when this fails: the webhook never arrives, so a poller/reconciler queries the provider for Pending orders older than N minutes.

**Follow-up traps**
- "Authorisation expiry?" Holds lapse after provider-specific days; capture before.
- "Is webhook reliable?" At-least-once and may be late/out of order; verify signature and be idempotent.

### 7.3 [Medium] The payment webhook arrives before your Order commit is visible, or twice. How do you handle it?
**What the interviewer is really testing**
Race conditions and idempotent receivers.

**Typical mid-level answer**
"Ignore if order not found." Loses the event.

**Strong senior answer**
Treat webhooks as hints, not truth. Persist the raw event to an inbox keyed by provider event id (dedupe), then process: if the order isn't found yet, don't drop; leave the event for retry (return 5xx so the provider retries, or park in a delayed queue). Handle duplicates via the unique event id and a state-guarded transition (`UPDATE Orders SET Status='Paid' WHERE Id=@id AND Status='Pending'`). Verify signature and, for critical events, fetch the payment from the provider API rather than trusting the payload. Out-of-order (refund before charge) is handled by state machine rules and timestamps from the provider, not arrival order. Risk: provider retry window ends; reconciliation catches leftovers.

**Follow-up traps**
- "Return 200 immediately?" Yes after durable inbox write, process asynchronously.
- "Replay attacks?" Signature plus timestamp tolerance.

### 7.4 [Hard] After the pivot (payment captured) the shipping service rejects the order permanently. What does the design do?
**What the interviewer is really testing**
Post-pivot recovery and business involvement.

**Typical mid-level answer**
"Roll everything back." Pivot means no rollback in the saga sense.

**Strong senior answer**
Ideally validation moved before capture so this is rare. When it happens, it becomes a business exception, not a technical retry: emit a ShipmentRejected event, move the saga to ManualReview or an AwaitingCustomerAction state, and open a task (fix address, choose alternative carrier). If the business decides to cancel, it's an explicit new flow: refund and restock, each idempotent and audited. Automatic refund can be allowed for well-understood reasons (unshippable region) with policy. I'd record the decision and who/what made it. Risk: auto-refund loops and customer confusion; keep state machine transitions explicit and notify the customer.

**Follow-up traps**
- "Retry forever?" Retries for transient errors only; permanent errors need decision.
- "Who owns the refund?" Payment service, triggered by command with idempotency key.

### 7.5 [Tricky] How do you prove that no customer was charged without an order, ever?
**What the interviewer is really testing**
Reconciliation as an invariant checker, honesty about limits.

**Typical mid-level answer**
"Our code handles it." Unverifiable.

**Strong senior answer**
You can't prove it by code review; you detect it continuously. Daily (or hourly) reconciliation: pull provider settlement/charge list, join to Payments and Orders by provider reference, and report orphans in each direction (charge without order, order marked paid without charge, amount mismatches). Orphans open tickets and auto-heal where safe (complete order or refund). Track the orphan rate as an SLI. Also alert on Pending orders older than the timeout. The residual risk is the window between failure and detection, which the business accepts as a stated bound (e.g. 1 hour).

**Follow-up traps**
- "Reconciliation changes data?" Report first, auto-repair only for safe classes, always audit.
- "Provider report delays?" Account for settlement lag in matching windows.

**Red flags in answers**
- Jumping to "refund" without root cause or idempotency.
- Trusting the synchronous payment response as the only source of truth.
- No reconciliation or ownership of orphan cases.

**Cheat sheet**
- Create Pending order first; payment reference as natural key.
- Webhooks: dedupe by event id, verify, state-guarded transition.
- Authorise early, capture as pivot.
- Forward recovery preferred; refund as explicit idempotent compensation.
- Reconcile provider vs your data; measure orphan rate.

---

## Topic 8: CAP, PACELC and consistency models

**Study notes**
- CAP: under a network Partition you must choose Consistency (linearizable) or Availability. It says nothing about normal operation. "CA" is not a real choice for a distributed system: partitions happen.
- PACELC: if Partition, choose A or C; Else (normal), choose Latency or Consistency. This is the everyday trade-off (replication latency vs freshness).
- Models, strongest to weakest: linearizable/strong; sequential; causal; session guarantees (read-your-writes, monotonic reads, monotonic writes, writes-follow-reads); eventual.
- Cosmos DB's five levels: Strong, Bounded Staleness, Session, Consistent Prefix, Eventual. Azure SQL geo-replica reads are asynchronous (stale); SQL primary is strongly consistent.

```
Write W at primary --async--> replica (lag Δ)
Read at replica within Δ: stale.  Fix: read-your-writes via sticky session / LSN token / read primary
```

### 8.1 [Easy] Explain CAP in plain language.
**What the interviewer is really testing**
Correct, non-cliche understanding.

**Typical mid-level answer**
"Pick two of consistency, availability, partition tolerance." Misleading: P isn't optional.

**Strong senior answer**
When the network splits the system into parts that can't talk, each side must either refuse requests that could return stale or conflicting data (consistent but unavailable) or keep serving and risk divergence (available but inconsistent). Because partitions can't be ruled out, it's really a choice made at partition time. Banking transfer ledger: choose consistency, return an error. Shopping cart or "like" counter: choose availability and merge later. And CAP's "consistency" means linearizability, different from ACID's C. What fails when this fails: choosing wrongly gives either lost sales (needless unavailability) or double-spent resources.

**Follow-up traps**
- "Is SQL Server CA?" A single instance isn't distributed; with Always On across sites you face the trade-off in config (sync vs async commit).
- "Does CAP apply when no partition?" No; that's PACELC's ELC part.

### 8.2 [Follow-up] What does PACELC add and how does it show up in Azure choices?
**What the interviewer is really testing**
Mapping theory to real knobs.

**Typical mid-level answer**
"It's CAP plus latency." Vague.

**Strong senior answer**
Most of the time there is no partition, and the real trade-off is latency vs consistency: waiting for replicas to acknowledge costs round trips. Examples: Azure SQL / Always On synchronous commit within a region (consistent, small latency) vs asynchronous cross-region replicas (low latency, stale reads, possible data loss on failover: RPO > 0). Cosmos DB consistency levels explicitly dial this: Strong requires cross-region quorum (higher write latency, and multi-region write isn't available with Strong), Session gives read-your-writes within a session at lower cost, Eventual is cheapest. I'd default to Session for user-facing apps, Strong only where the invariant demands it, and measure RU and latency. Risk: assuming replicas are fresh in reporting or authorisation checks.

**Follow-up traps**
- "Cosmos default level?" Session.
- "Bounded staleness?" Reads lag by at most K versions or T time.

### 8.3 [Medium] Explain read-your-writes, monotonic reads and causal consistency, and how you'd provide them.
**What the interviewer is really testing**
Session guarantees knowledge and implementation.

**Typical mid-level answer**
Confuses them with eventual consistency.

**Strong senior answer**
Read-your-writes: after I write, my reads reflect it. Monotonic reads: I never see data go backwards in time (reading replica A then a staler replica B). Causal: if event B was caused by A, everyone sees A before B. Provision: route a user's reads to the primary for a short window after a write; carry a version/LSN/session token with the response and have the reader wait or retry until the replica has caught up (Cosmos Session token does exactly that); sticky routing to a replica for monotonic reads; carry causality via message headers or version vectors. Trade-off: complexity and reduced load distribution. Decision: provide the weakest guarantee that keeps users from noticing; read-your-writes covers most UX complaints.

**Follow-up traps**
- "Is Session consistency global?" Scoped to a session token; other clients may see stale data.
- "Why not read primary always?" Load and latency, cross-region cost.

### 8.4 [Hard] Pick consistency levels for: bank balance, product catalog, shopping cart, "likes" counter, seat booking.
**What the interviewer is really testing**
Applying business risk to the choice.

**Typical mid-level answer**
"Strong for everything important." Overpays, and not reasoned.

**Strong senior answer**
Ask for each: what is the cost of a stale or conflicting read, and what's the invariant? Bank balance/ledger: strong within the account aggregate (single-writer, serializable/ conditional update), reads for transfers on primary. Product catalog: eventual/cached is fine, minutes of staleness acceptable. Cart: session/read-your-writes, merge on conflict (union of items). Likes: eventual, commutative counters. Seat booking: the final claim needs strong, atomic conditional write on that seat (unique constraint/ETag), while seat map display can be stale and the claim reports conflict. Pattern: weak reads for browsing, strong conditional writes for committing. Risk: showing stale availability causes failed bookings, handled by clear UX.

**Follow-up traps**
- "Why not strong for the seat map?" Cost; the commit check is the guard.
- "Counter under concurrency?" CRDT counter or atomic increment per partition.

### 8.5 [Tricky] "We have Cosmos Strong consistency in 3 regions, so no distributed consistency issues." Challenge it.
**What the interviewer is really testing**
Scope awareness: consistency of one store isn't cross-service consistency.

**Typical mid-level answer**
Agrees.

**Strong senior answer**
Strong consistency covers reads and writes of that one account's data (linearizable per item/partition). It doesn't link Cosmos to my SQL database, the broker, or a payment provider, so the dual-write and saga problems remain. It also costs availability and latency: writes need a cross-region quorum, and during a partition the affected region may be unavailable. Multi-region writes aren't supported with Strong. And transactions in Cosmos are limited to a logical partition (transactional batch/stored procedures). So I'd use Strong only for invariants that demand it and still design cross-service flows with outbox/saga.

**Follow-up traps**
- "Cross-partition transaction in Cosmos?" Not supported; model aggregates within a partition key.
- "Strong's cost?" Higher latency; RU cost of reads is higher.

**Red flags in answers**
- "Pick two of three" with P treated as optional.
- Using "strong" for everything or "eventual" for money.
- Confusing ACID's C with CAP's C.

**Cheat sheet**
- CAP matters during partitions; PACELC covers normal-time latency vs consistency.
- Session guarantees (RYW, monotonic reads) solve most user complaints.
- Cosmos: Strong, Bounded, Session(default), Consistent Prefix, Eventual.
- Weak reads to browse, strong conditional writes to commit.
- Store-level consistency doesn't solve cross-service atomicity.

---

## Topic 9: Designing UX and CQRS read models for eventual consistency

**Study notes**
- CQRS: separate write model (commands, invariants) from read model(s) (projections tuned for queries). Projections are updated from events, hence lag.
- UX patterns: optimistic UI (assume success, reconcile), pending/processing states, 202 Accepted + status resource, correlation id polling or SignalR push, "read your own write" shortcuts (return the written resource/state in the response), disabling duplicate actions, clear messaging ("we're processing your order").
- Measure lag: event timestamp vs projection watermark.

```
POST /orders -> 202 Accepted + Location: /orders/O1/status
UI shows "Processing"  ... projection catches up ...  GET status -> Confirmed (or push via SignalR)
```

### 9.1 [Easy] A user saves a change then refreshes and doesn't see it. Why, and what do you do?
**What the interviewer is really testing**
Diagnosing read-model lag and picking pragmatic fixes.

**Typical mid-level answer**
"Add a delay/refresh." Hacky.

**Strong senior answer**
The write committed but the read model/replica hasn't caught up. Options: return the updated resource in the command response and have the UI update local state (optimistic/confirmed); route the user's next read to the write model/primary for a short period; use a version token so the read model query waits until it reaches at least that version; or show a "pending" indicator. I'd pick returning the written state plus version token, because it fixes the most common case without extra infrastructure. Risks: token waiting needs timeout; fall back to "processing".

**Follow-up traps**
- "Spinner until consistent?" Only with a bounded wait.
- "Two tabs?" Second tab may be stale; versions and refresh.

### 9.2 [Follow-up] How do you design the API for a long-running, eventually-consistent command?
**What the interviewer is really testing**
HTTP semantics for async work.

**Typical mid-level answer**
"Return 200 when done," holding the connection.

**Strong senior answer**
Accept the command, validate synchronously what can be validated, persist intent (with an idempotency key), return 202 Accepted with a status URL. The status resource exposes Pending/Succeeded/Failed with reason and links. Notify via SignalR/webhook/polling with Retry-After. Make commands idempotent so client retries are safe. For failures after acceptance, surface a clear failed state and next action. Risk: unbounded Pending; saga timeouts guarantee a terminal state.

**Follow-up traps**
- "Why not wait synchronously?" Timeouts, thread/connection use, and failures remain ambiguous.
- "Polling cost?" Backoff, or push.

### 9.3 [Medium] How do you decide how much lag a read model may have, and how do you monitor it?
**What the interviewer is really testing**
SLO-driven thinking.

**Typical mid-level answer**
"As low as possible." No numbers.

**Strong senior answer**
Per screen/use case ask the business: what's the harm of a stale view? Dashboards: minutes; order status after checkout: seconds; inventory at pick time: must check the source. Set an SLO (e.g. p99 < 5 s), measure by stamping events with source commit time and tracking each projection's last processed event time or sequence; the delta is lag. Alert on breach and on projector failure. Use per-projection checkpointing so rebuilds are possible. Risk: a poison event stalling a projection blocks all subsequent events in the partition; dead-letter and continue where safe.

**Follow-up traps**
- "Lag by clock?" Prefer sequence/position over wall clocks.
- "Rebuild projection?" Replay from events/snapshot into a new version, then swap.

### 9.4 [Hard] The business insists on "instant consistency" between the order page and inventory shown on the product page. Respond.
**What the interviewer is really testing**
Negotiation: converting requirements to measurable ones, offering options.

**Typical mid-level answer**
"We can't do that, it's eventually consistent." Unhelpful.

**Strong senior answer**
Clarify what is actually harmful: showing "In stock" when it's gone leads to failed purchases and bad experience; showing exact counts isn't needed. Options: (1) shared read path with the source of truth for the critical check at add-to-cart/checkout (the commit is strongly consistent anyway); (2) show coarse availability (In stock / Low / Out) with a TTL of seconds; (3) push invalidation via events to cache; (4) if truly needed, co-locate inventory and order in one transactional boundary, accepting coupling. I'd propose 1+2: browsing is approximate, checkout is authoritative with clear messaging on conflict. Cost-benefit shown to the business.

**Follow-up traps**
- "Why not always query inventory?" Coupling/availability: product page would fail when inventory is down.
- "Cache invalidation?" Event-driven plus short TTL as safety net.

### 9.5 [Scenario] Users double-click "Pay" because the UI shows nothing while processing. Design the fix end to end.
**What the interviewer is really testing**
Combining UX with idempotency.

**Typical mid-level answer**
"Disable the button." Necessary, not sufficient.

**Strong senior answer**
Layers: UI disables the button and shows progress; client generates an idempotency key when the checkout form is created (persisted in page state) and sends it with the request; the API dedupes by key (unique constraint) and returns the stored response for repeats; the payment provider call carries the same key; the order status page is the landing target so refresh is safe. Edge: user opens a second tab: the key is per cart/checkout session, so the second attempt maps to the same order. Failure: if the response is lost, retry with the same key returns the original result. UI-only fixes fail on retries from proxies/mobile networks.

**Follow-up traps**
- "Key per click?" No, per intent.
- "Browser back button?" Server-side state is the guard.

**Red flags in answers**
- Fixing double submits only in the UI.
- Telling users to "refresh" as the design.
- No staleness SLO or lag metric for projections.

**Cheat sheet**
- 202 + status resource + terminal state guaranteed by timeouts.
- Return written state and version token for read-your-writes.
- Set lag SLO per screen; measure by position/time.
- Weak reads to browse, authoritative checks at commit.
- Idempotency key per user intent, end to end.

---

## Topic 10: Message ordering, partitioning, out-of-order events, event versioning

**Study notes**
- Global ordering is expensive and rarely needed; per-entity ordering is the usual requirement.
- Service Bus: queues/topics are FIFO-ish but competing consumers, retries and DLQ break processing order. Sessions (SessionId) give strict FIFO per session with a single receiver holding the session lock. Partitioned entities: messages with the same PartitionKey/SessionId go to the same partition. Kafka/Event Hubs: order only within a partition; partition key = entity id.
- Out-of-order handling: version/sequence number per aggregate; apply if version == current+1; buffer or park if gap; ignore if <= current (duplicate/old).
- Event versioning: additive changes are backward compatible; use explicit schema versions, upcasters, tolerant readers; never mutate published event meaning.

```
Aggregate A5 events: v1 v2 v3        Consumer state: version=1
Receive v3 -> gap (need v2): park/delay.  Receive v2 -> apply.  Receive v3 -> apply.  Receive v2 again -> ignore.
```

### 10.1 [Easy] Does Service Bus guarantee message order?
**What the interviewer is really testing**
Accuracy about guarantees.

**Typical mid-level answer**
"Yes, it's a FIFO queue." Only partially true.

**Strong senior answer**
Service Bus queues deliver in order to a single receiver when nothing fails, but with competing consumers, concurrent handlers, lock expiry and redelivery, or a dead-lettered message, processing order is not guaranteed. For guaranteed FIFO you enable sessions and set a SessionId (e.g. the order id): messages in a session are delivered in order to one receiver at a time. The trade-off: parallelism is per session, and a poison message in a session blocks that session. What fails when this fails: a handler error on message 2 could allow 3 to be processed if not using sessions, or blocks the session if using them.

**Follow-up traps**
- "Concurrency inside a session?" One receiver per session; process serially.
- "Topics?" Subscriptions can also be session-enabled.

### 10.2 [Follow-up] How do you handle out-of-order events at a consumer?
**What the interviewer is really testing**
Practical algorithms.

**Typical mid-level answer**
"Sort by timestamp." Timestamps lie.

**Strong senior answer**
Carry a per-aggregate monotonically increasing version from the producer (the aggregate's own version, assigned inside its transaction). Consumer rule: expected = stored+1; apply if equal; ignore if lower (duplicate/stale); if higher, there is a gap, so defer (requeue with delay, or store in a holding area) until the missing version arrives or a timeout triggers a fetch from the source (query API for current state). For state-snapshot events ("OrderUpdated" carrying full state) a simpler rule works: apply only if version > stored (last-writer-by-version), making it idempotent and order-tolerant. Choose based on whether events are deltas (need ordering) or snapshots (don't). Risk: gap that never fills; the timeout/refetch is the escape hatch.

```sql
UPDATE dbo.OrderView SET Status=@s, Version=@v
WHERE OrderId=@id AND Version < @v;   -- ignore stale/duplicate
```

**Follow-up traps**
- "Why not timestamps?" Clock skew, equal timestamps, retries.
- "Gap forever?" Refetch from source of truth.

### 10.3 [Medium] How do you choose a partition key / session id?
**What the interviewer is really testing**
Balancing ordering needs against hot partitions.

**Typical mid-level answer**
"Customer id." Without analysing the access pattern.

**Strong senior answer**
Pick the smallest unit that needs ordering: the aggregate id (order id), not the tenant, unless cross-aggregate order matters. High cardinality spreads load; low cardinality (country, tenant with one giant customer) creates hot partitions and serialises throughput. If I need ordering for related entities (order and its lines), use the root aggregate id for all. Consider future needs: re-keying later is costly. In Cosmos the partition key also defines the transaction scope and change feed ordering (within a logical partition). Risk: a single hot key (one huge tenant) throttles; mitigate by sub-partitioning (tenant+bucket) if ordering allows.

**Follow-up traps**
- "Many sessions cost?" Sessions are lightweight but each needs a receiver slot; use session processors with max concurrent sessions.
- "Need global order?" Rethink; single partition becomes bottleneck.

### 10.4 [Hard] You must change the shape of a published event consumed by 6 teams. How?
**What the interviewer is really testing**
Contract evolution without breaking consumers.

**Typical mid-level answer**
"Version the API and deploy together." Coordinated release is brittle.

**Strong senior answer**
Prefer backward-compatible changes: add optional fields, never remove/rename/repurpose; consumers use tolerant readers (ignore unknown fields). For breaking changes publish a new event type/version (`OrderPlaced.v2`) alongside v1 for a deprecation period (dual publish), or use schema registry compatibility rules. Include `schemaVersion` and type in message headers so consumers can route. Track which consumers still read v1 (subscriptions/metrics) before retiring. For stored events (event sourcing) use upcasters. Contract tests (consumer-driven) in CI catch breaks. Risk: dual publishing doubles messages and may skew ordering; plan sunset date.

**Follow-up traps**
- "Rename a field?" Add new, populate both, migrate, remove later.
- "Messages already in flight/DLQ?" Old-shape messages persist; consumers must still read them.

### 10.5 [Tricky] A session-enabled queue gets stuck: one session isn't progressing. Why and what do you do?
**What the interviewer is really testing**
Understanding session semantics and poison handling.

**Typical mid-level answer**
"Restart the consumer."

**Strong senior answer**
Within a session, messages are processed in order by one receiver; if message N keeps failing and exceeds MaxDeliveryCount it is dead-lettered and the session moves on (or, if the handler abandons indefinitely or holds the lock, the session stalls). Check lock renewal, handler exceptions, delivery counts and the DLQ. If a message is permanently bad, dead-letter it explicitly and continue, accepting a gap, and ensure the consumer's version-gap logic refetches state. Alternatively a session state can store progress. Also check that the session lock didn't get lost due to long processing. Prevention: bounded processing time, poison classification, metrics per session age.

**Follow-up traps**
- "Skipping a message breaks order?" Yes: gap handling and reconciliation required.
- "Does DLQ preserve session?" Messages go to DLQ individually; replay ordering is your job.

**Red flags in answers**
- Ordering by timestamps/DateTime.
- Using a low-cardinality partition key for throughput-heavy flows.
- Planning breaking event changes as a "big bang" deployment.

**Cheat sheet**
- Order matters per aggregate; use sessions/partition key = aggregate id.
- Version numbers beat timestamps; apply only if version is next/greater.
- Snapshot-style events are order-tolerant; delta events need gap handling.
- Evolve events additively; version type for breaking changes.
- A poison message blocks its session/partition: plan DLQ and gap recovery.

---

## Topic 11: Event sourcing

**Study notes**
- State is stored as an append-only sequence of events per aggregate; current state = fold of events. Reads via projections.
- Benefits: full audit, temporal queries, natural outbox (events are the record), rebuildable read models, debugging.
- Costs: eventual-consistent reads, schema evolution of stored events, learning curve, query complexity, rebuild time, GDPR deletion difficulty.
- Snapshots: periodic state checkpoints to avoid replaying thousands of events.
- Concurrency: append with expected version (optimistic).
- GDPR: crypto-shredding (encrypt PII per subject, delete key), or keep PII outside events (reference ids), or rewrite streams (rare).

```
Stream Order-O1:  [1 Placed][2 PaymentTaken][3 Shipped]   snapshot@100 ...
Append(expectedVersion=3, event 4) -> conflict if someone else appended 4
```

```sql
-- append with optimistic concurrency
INSERT INTO dbo.Events(StreamId, Version, Type, Data, CreatedUtc) VALUES(@s,@v,@t,@d,SYSUTCDATETIME());
-- PK(StreamId, Version): duplicate version => concurrency conflict
```

### 11.1 [Easy] What is event sourcing and what problem does it solve?
**What the interviewer is really testing**
Balanced understanding.

**Typical mid-level answer**
"Store events instead of state; you get an audit log." Omits costs.

**Strong senior answer**
Instead of overwriting rows, I append immutable domain events and derive state by replay. It solves: complete history (audit, "why is it in this state?"), temporal queries, and the dual-write issue because the event store is the single write that feeds everything else. It suits domains where history is a first-class concern (finance, bookings, logistics). Cost: you must build projections for reads, handle schema evolution of events for years, and think in eventual consistency. I'd adopt it per bounded context, not system-wide. What fails: broken projections are rebuildable, but a bad event in the store is permanent, so corrections are new compensating events.

**Follow-up traps**
- "Same as audit table?" No: the audit table is a side-record; here events are the source of truth.
- "Is it required for CQRS?" No, independent.

### 11.2 [Follow-up] Snapshots and replay: how and when?
**What the interviewer is really testing**
Practical performance and rebuild knowledge.

**Typical mid-level answer**
"Snapshot every N events."

**Strong senior answer**
Replaying a stream is O(events); with small streams (dozens) it's fine and snapshots are premature. For long-lived aggregates, store snapshot (state + version) every N events or time; load snapshot then events after its version. Snapshots are a cache, never truth, and versioned: if the aggregate code changes, discard and rebuild. For projections, replay the full store into a new projection version in parallel, switch traffic, and drop the old one (blue/green projections). Replay must be idempotent and side-effect free: don't send emails during replay (separate "process managers" from "projections"). Risk: rebuild duration and load on the store; throttle and checkpoint.

**Follow-up traps**
- "Replay triggers emails?" Must be disabled; reactors keep their own checkpoint.
- "Snapshot corruption?" Fallback to full replay.

### 11.3 [Medium] How do you evolve event schemas over years?
**What the interviewer is really testing**
Handling immutability with change.

**Typical mid-level answer**
"Migrate the events." Dangerous.

**Strong senior answer**
Events are facts, so avoid rewriting. Strategies: weak/tolerant schema (additive optional fields); upcasting at read time (v1 -> v2 transformation chain in the deserialiser); new event types for new meaning; copy-and-transform migration into a new stream/store for large breaking changes (carefully, offline, verified). Name events after business facts (`ShippingAddressChanged`), not CRUD, to reduce churn. Keep type name + version in metadata. Test with a corpus of historical events in CI. Risk: upcaster chain grows; periodically do a controlled migration.

**Follow-up traps**
- "Missing field default?" Upcaster supplies explicit default.
- "Rename event class?" Use stable type name mapping, not CLR type name.

### 11.4 [Hard] How do you satisfy GDPR erasure in an event-sourced system?
**What the interviewer is really testing**
Knowing the conflict and options.

**Typical mid-level answer**
"Delete the events." Breaks integrity.

**Strong senior answer**
Immutable store versus right to erasure: design for it up front. Preferred: keep personal data out of events (store a customer reference; PII lives in a mutable store that can be deleted). If PII must be in events: crypto-shredding: encrypt the PII fields with a per-subject key stored in Key Vault/ a key table; erasure = delete the key, rendering data unreadable while event structure stays. Rewriting streams is a last resort (breaks hashes/ordering, complex backups). Also address projections, snapshots, backups and downstream copies: they must be purged or rebuilt. Legal review decides retention exceptions. Risk: key deletion must cover backups (keys backed up separately with their own retention) and analytics copies.

**Follow-up traps**
- "Backups?" Keys' lifecycle controls readability; document policy.
- "Search by deleted person?" Return nothing; replay yields pseudonymised values.

### 11.5 [Tricky] When would you NOT use event sourcing?
**What the interviewer is really testing**
Judgment, avoiding resume-driven design.

**Typical mid-level answer**
"When it's simple CRUD." Correct but shallow.

**Strong senior answer**
Don't use it for CRUD-dominant data, reference data, reporting-heavy domains needing ad-hoc relational queries across everything, teams without experience and with tight deadlines, domains with heavy PII and frequent erasure, or where audit needs are met with temporal tables / CDC. SQL Server temporal tables or an audit table give history cheaply. Event sourcing also complicates simple questions like "list all orders by status" without projections. If I only need reliable event publishing, an outbox is much cheaper. Decision: use it where history and replay are business value, and keep it contained to specific aggregates.

**Follow-up traps**
- "Temporal tables vs ES?" Temporal tracks state history, not intent.
- "Hybrid?" State-stored aggregates with outbox events is common.

**Red flags in answers**
- Event sourcing for everything "for the audit trail".
- Updating or deleting past events to fix bugs.
- No plan for schema evolution or PII.

**Cheat sheet**
- Events are source of truth; state = fold(events); read via projections.
- Append with expected version for concurrency.
- Snapshots are a cache; projections rebuildable and side-effect free.
- Evolve by additive changes and upcasters.
- GDPR: keep PII out or crypto-shred; use sparingly.

---

## Topic 12: Distributed locks and lock-free designs

**Study notes**
- Need: mutual exclusion across processes for a critical section (e.g. run a job once, protect a shared resource).
- Options: SQL `sp_getapplock` (session/transaction-scoped, uses the DB you already have), Azure Blob lease (15-60 s or infinite, renewable), Redis `SET key value NX PX ttl` (with unique token, release via compare-and-delete script), Redlock (multiple Redis nodes).
- Any lease/TTL lock can expire while the holder is paused (GC, VM pause, network): two holders at once. Fencing token: monotonically increasing number issued with the lock; the protected resource rejects writes with a lower token.
- Lock-free: idempotency, optimistic concurrency, single-writer partitions (session per aggregate), queues serialising work, unique constraints, conditional updates.

```
Client1 gets lock (token 33) -> long pause -> lease expires
Client2 gets lock (token 34) -> writes to storage (accepts 34)
Client1 resumes -> writes with token 33 -> storage REJECTS (33 < 34)
```

```sql
EXEC @r = sp_getapplock @Resource='nightly-job', @LockMode='Exclusive',
     @LockOwner='Session', @LockTimeout=0;   -- @r >= 0 => acquired
```

### 12.1 [Easy] Why would you need a distributed lock and what's the simplest safe option on Azure + SQL Server?
**What the interviewer is really testing**
Choosing the simplest tool.

**Typical mid-level answer**
"Redis lock."

**Strong senior answer**
First ask whether I need a lock at all: perhaps a unique constraint or conditional update does it. If I need mutual exclusion for a job (only one instance runs), the simplest is a lock on infrastructure already in the path: `sp_getapplock` in SQL Server (released on session end/connection loss, so crash releases it) or an Azure Blob lease for singleton background work (holder renews; expires if it dies). Redis adds another dependency and failure mode. Trade-off: SQL locks tie to DB availability and pooled connections need care; blob leases are coarse and time-based. Risk: lease expiry during a pause: protect the resource with fencing or make the job idempotent.

**Follow-up traps**
- "Applock with connection pooling?" Use transaction owner or hold a dedicated connection; reset on return to pool can release it.
- "Blob lease duration?" 15-60 seconds or infinite; renew periodically.

### 12.2 [Follow-up] What's wrong with a Redis lock using SET NX PX?
**What the interviewer is really testing**
Understanding lease expiry and release safety.

**Typical mid-level answer**
"Nothing, it's atomic."

**Strong senior answer**
Atomic acquire is fine, but: (1) the TTL is a guess; if work exceeds TTL the lock expires and a second holder enters. (2) Releasing with plain DEL could delete someone else's lock, so store a unique token and delete only if it matches (Lua script). (3) A failover of a single primary with async replication can lose the key, granting the lock to two clients. (4) Pauses (GC, network) mean the holder can still act after expiry. So it's an efficiency lock (avoid duplicate work) not a correctness lock unless paired with fencing at the resource. Decision: use it for best-effort deduplication; for correctness use DB constraints/fencing.

**Follow-up traps**
- "Extend TTL (watchdog)?" Helps but doesn't remove pause risk.
- "Redis Cluster?" Failover can still lose locks.

### 12.3 [Medium] Summarise the Redlock debate and your position.
**What the interviewer is really testing**
Awareness of known critiques without cargo culting.

**Typical mid-level answer**
"Redlock is the safe way to use Redis locks."

**Strong senior answer**
Redlock acquires the lock on a majority of N independent Redis nodes within a validity time. Martin Kleppmann's critique: it relies on bounded clock drift and bounded pauses, and with no fencing token a paused client can still write after expiry, so it's unsafe for correctness-critical locks; for efficiency-only locks it's overkill (single node is enough). Antirez (Redis author) responded that the algorithm's timing assumptions are acceptable in practice. My position: if a lock violation corrupts data, don't depend on timing: use a fencing token or conditional write at the resource (rowversion/ETag/unique key). If it's only to avoid duplicate work, a simple lease suffices. Risk: treating Redlock as a safety guarantee.

**Follow-up traps**
- "Does Redlock issue fencing tokens?" No, it doesn't provide monotonic tokens.
- "Alternative with consensus?" ZooKeeper/etcd offer sequential tokens; on Azure, blob lease + conditional writes or SQL.

### 12.4 [Hard] Explain fencing tokens and how to implement one with SQL Server.
**What the interviewer is really testing**
Making a lease lock correct.

**Typical mid-level answer**
Hasn't heard of them.

**Strong senior answer**
A fencing token is a number that increases every time the lock is granted; the resource checks it on each write and rejects any token lower than the highest seen. Implementation: a `Locks` table row per resource with `Token BIGINT`, holder, expiry. Acquire: `UPDATE ... SET Token = Token+1, Holder=@me, ExpiresUtc=... WHERE Resource=@r AND (ExpiresUtc < SYSUTCDATETIME() OR Holder=@me)` and read the new token. The protected table stores `LastToken`; writes use `WHERE LastToken <= @token` and set it. A stale holder's write affects zero rows. Often simpler: skip the lock and use optimistic concurrency (rowversion) on the resource, which gives the same safety. The resource must support the check, which is the limitation with third-party systems.

**Follow-up traps**
- "Resource doesn't support tokens?" Then lock only gives best-effort; make operation idempotent.
- "Clock for expiry?" Use DB time (SYSUTCDATETIME) consistently, not app clocks.

### 12.5 [Scenario] A scheduled job runs on 5 instances of an App Service. Ensure it runs once without introducing new infrastructure.
**What the interviewer is really testing**
Pragmatic singleton designs.

**Typical mid-level answer**
"Use Redis lock."

**Strong senior answer**
Options: (1) run the job in exactly one place: an Azure Function timer trigger on a single plan or an App Service WebJob singleton: let the platform do it (Functions timer uses a blob lease singleton). (2) Claim work in SQL: `UPDATE Jobs SET LockedBy=@me, LockedUntil=... WHERE Id=@id AND (LockedUntil IS NULL OR LockedUntil < SYSUTCDATETIME())` and check rows affected = 1. (3) `sp_getapplock` with timeout 0. I'd choose (1) or (2) depending on whether the job is a simple cron or a work-queue claim; and make the job idempotent so an overlap (pause, expired claim) is harmless. Failure: an instance dies mid-run, so claim expiry must let another instance resume.

**Follow-up traps**
- "Overlap after expiry?" Idempotent + fencing/version.
- "Scale-out Functions?" Timer trigger singleton prevents multiple concurrent runs of the same timer across instances.

**Red flags in answers**
- Treating TTL locks as a correctness guarantee.
- Releasing a Redis lock with plain DEL, no token.
- Reaching for a distributed lock when a unique constraint would do.

**Cheat sheet**
- Prefer no lock: unique constraints, conditional updates, single writer per key.
- Leases expire; pauses cause double holders: use fencing or idempotency.
- Redis NX PX: efficiency lock; token-checked release.
- Redlock: timing assumptions; no fencing token.
- SQL applock / blob lease are the low-infrastructure options.

---

## Topic 13: Optimistic concurrency: rowversion and ETags across services

**Study notes**
- Optimistic: read a version, write only if the version is unchanged; on conflict, reload/merge/retry or report. Good when conflicts are rare. Pessimistic: lock up front; good when conflicts are frequent, but doesn't span services/HTTP requests.
- SQL Server `rowversion` (formerly timestamp): 8-byte, database-wide increasing value auto-updated on each row change; unrelated to time. EF Core: `[Timestamp] byte[] RowVersion` or `IsRowVersion()`, throws `DbUpdateConcurrencyException`.
- HTTP: `ETag` response header + `If-Match` on PUT/PATCH; mismatch => `412 Precondition Failed`; `If-None-Match: *` for create-only; missing precondition can return `428`.
- Cosmos DB: `_etag` with `IfMatchEtag` request option.

```
GET /orders/1 -> 200, ETag: "AAAAAAAB7Zs="
PUT /orders/1  If-Match: "AAAAAAAB7Zs="  -> 204 (new ETag)   or 412 if changed meanwhile
```

```sql
UPDATE dbo.Orders SET Status=@s WHERE Id=@id AND RowVer=@expected;
IF @@ROWCOUNT=0 -- conflict: someone else changed it
```

### 13.1 [Easy] What is optimistic concurrency and when do you use it?
**What the interviewer is really testing**
Basics with selection criteria.

**Typical mid-level answer**
"Check a version column on update."

**Strong senior answer**
Assume conflicts are rare: read without locking, and make the update conditional on the version read. If zero rows are updated, someone else won; the application decides: retry (for commutative/idempotent operations), merge, or tell the user. It prevents lost updates without holding locks across user think-time or across service calls. I pick it for web/API workloads and cross-service flows; pessimistic only for short, high-contention critical sections inside one database. What fails when this fails: without it, last write silently overwrites, e.g. two support agents edit an order.

**Follow-up traps**
- "Retry on conflict always?" Only if the operation is recomputed from fresh state.
- "High contention?" Optimistic retries thrash; use atomic update statements or queue/serialise.

### 13.2 [Follow-up] Rowversion vs a manual Version int vs datetime column?
**What the interviewer is really testing**
Technical correctness.

**Typical mid-level answer**
"Datetime LastModified works."

**Strong senior answer**
`rowversion` is automatically bumped by the engine on any row change including changes by other code paths/scripts, so it can't be forgotten. A manual int version works but every writer must increment it. Datetime is unreliable: resolution collisions (two updates within the same tick), clock skew across app servers, and updates that forget to set it. Rowversion values are binary, per-database ordering, and unsuitable as a cross-database event version (not contiguous, changes on every update including no-op-ish ones). For event versions I use an explicit aggregate version I control. Risk: exposing rowversion as an ETag is fine; using it for cross-service ordering is not.

**Follow-up traps**
- "Is rowversion a timestamp?" No, despite older name `timestamp`.
- "Contiguous?" No, gaps exist.

### 13.3 [Medium] How does optimistic concurrency work across services (e.g. UI -> API -> another service)?
**What the interviewer is really testing**
Propagating version tokens.

**Typical mid-level answer**
Version only at the DB layer.

**Strong senior answer**
The token must travel with the data. API returns an ETag derived from rowversion/aggregate version; client sends it back in `If-Match`; the service maps it to the conditional update, and returns 412 on mismatch. When service A calls service B on behalf of the user, pass B's version/ETag in the call (or have B's operation be defined as a command with an expected version). In messaging, include `ExpectedVersion` in commands. For event-sourced aggregates the expected version is checked on append. Trade-off: clients must handle 412 (refetch, merge, retry). Risk: services that ignore the precondition reintroduce lost updates, so enforce in API filters and tests.

**Follow-up traps**
- "Weak vs strong ETags?" Strong implies byte-identical representation; `W/` weak means semantically equivalent.
- "PATCH?" Still use If-Match.

### 13.4 [Hard] Two users edit the same order in different services (Orders: shipping address; Billing: payment method). Do you need concurrency control?
**What the interviewer is really testing**
Granularity: concurrency at aggregate boundary and conflicts between fields.

**Typical mid-level answer**
"Yes, version the order."

**Strong senior answer**
Clarify aggregate boundaries. If the fields are different aggregates in different services, they don't conflict by construction; each has its own version. If they're the same aggregate, a single version makes unrelated edits conflict unnecessarily (false conflicts); options: finer-grained version columns, field-level merge, or splitting the aggregate. Conflicts that remain semantic (an address change after "Shipped") are better enforced by state rules than version checks. My decision: version at aggregate granularity, keep aggregates small, and encode business rules for state-dependent changes. Risk: cross-aggregate invariants (address must match billing country) need a saga/validation, not a version.

**Follow-up traps**
- "Merge on conflict?" Possible for disjoint fields; ask user for overlapping.
- "Big aggregate problem?" More conflicts and contention.

### 13.5 [Tricky] You get frequent DbUpdateConcurrencyException on a hot row (inventory counter). Fix it.
**What the interviewer is really testing**
Knowing when optimistic concurrency is the wrong tool.

**Typical mid-level answer**
"Retry more times."

**Strong senior answer**
A hot counter has high contention, so optimistic retries waste work and add latency. Use an atomic set-based update: `UPDATE Stock SET Qty = Qty - @q WHERE Sku=@s AND Qty >= @q` (row lock held briefly; check rows affected), which is both race-free and invariant-enforcing. If still hot, shard the counter (N sub-rows summed on read), or serialise by routing commands for a SKU through a session/partition so one consumer processes them, or reserve in batches. Keep the transaction short and consistent index to avoid deadlocks. Risk: sharded counters complicate exact reads and reconciliation.

**Follow-up traps**
- "Read-modify-write in app?" Source of lost updates; avoid.
- "Isolation level?" Atomic update works under READ COMMITTED; snapshot isolation would raise update conflicts.

**Red flags in answers**
- Using datetime LastModified as the concurrency token.
- Ignoring If-Match on PUT/PATCH.
- Retrying optimistic failures blindly on hot rows.

**Cheat sheet**
- Conditional update on a version; zero rows = conflict.
- rowversion is automatic but DB-local; ETag/If-Match/412 over HTTP.
- Carry tokens through API and commands (ExpectedVersion).
- Small aggregates reduce false conflicts.
- Hot rows: atomic UPDATE with guard, sharding, or serialisation.

---

## Topic 14: Retries, timeouts, poison messages and dead-letter strategy

**Study notes**
- Timeout = unknown outcome. The call may have succeeded. Retrying a non-idempotent call can duplicate effects.
- Retry policy: only transient errors; exponential backoff with jitter; cap attempts; per-attempt timeouts under an overall deadline; circuit breaker; avoid retry multiplication across layers (retry storm).
- Service Bus: peek-lock, `MaxDeliveryCount` (default 10) then dead-letter; lock duration (default 1 min on queues, max 5 min) with renewal; DLQ is a sub-queue `$DeadLetterQueue`; TTL expiry can dead-letter if enabled.
- Poison: message that always fails (bad payload, bug, unmet dependency). Strategy: classify errors (transient -> retry; permanent -> dead-letter immediately with reason), alert, inspect, fix, resubmit.
- Polly / `Microsoft.Extensions.Http.Resilience` for HTTP; MassTransit retry/redelivery/`_error` queues.

```
Consumer: transient (503, timeout) -> Abandon (redeliver, backoff)
          permanent (validation/schema) -> DeadLetter(reason, description)
          success -> Complete
```

### 14.1 [Easy] A call to another service times out. Is it safe to retry?
**What the interviewer is really testing**
Unknown outcome and idempotency.

**Typical mid-level answer**
"Yes, retry 3 times."

**Strong senior answer**
A timeout only says I didn't get an answer; the server may have processed the request. Retrying is safe if the operation is idempotent (GET, PUT, DELETE by nature, or POST with an idempotency key). For non-idempotent operations I either add a key or first query the outcome. Retry only transient failures (timeouts, 503, 429 honoring Retry-After), with exponential backoff and jitter and a limit. What fails when this fails: duplicate charges/orders, or retry storms amplifying an outage.

**Follow-up traps**
- "Retry 400s?" No, permanent.
- "Retry budget?" Cap total retries across layers.

### 14.2 [Follow-up] How do you avoid retry storms in a multi-tier system?
**What the interviewer is really testing**
Systemic thinking.

**Typical mid-level answer**
"Use Polly."

**Strong senior answer**
Retries multiply: 3 retries at each of 3 layers is 27 calls to the bottom. Fix: retry at one layer (usually closest to the failing dependency or at the edge), propagate deadlines so downstream doesn't work past the caller's timeout, add jitter, use circuit breakers to fail fast when the dependency is clearly down, respect 429/Retry-After, and add bulkheads to isolate pools. Load shedding at the dependency protects it. Metrics: retry rate and breaker state. Risk: a breaker that opens too eagerly causes needless failures; tune with real traffic.

**Follow-up traps**
- "Jitter why?" Avoid synchronised retry waves.
- "Retry in the message consumer too?" Prefer broker redelivery with delay over in-process loops when long.

### 14.3 [Medium] Design the dead-letter strategy for a Service Bus consumer.
**What the interviewer is really testing**
Operational design.

**Typical mid-level answer**
"Messages go to the DLQ after 10 tries."

**Strong senior answer**
Classify: transient errors abandon with backoff (or use scheduled redelivery); permanent errors dead-letter immediately with `DeadLetterReason`/description to avoid 10 pointless attempts. Set MaxDeliveryCount and lock duration appropriately; renew locks for long handlers. Monitor DLQ depth and age with alerts, and have a runbook: inspect, fix cause (code/data), then resubmit (a tool that re-sends a copy to the main queue; consumers idempotent). Record DLQ'd message metadata for audit. For ordered flows (sessions) think about what a skipped message does to downstream versions. Retain messages beyond TTL expectations. Risk: DLQ becomes a graveyard; assign an owner and SLA.

**Follow-up traps**
- "DLQ of DLQ?" Not automatic; resubmit tooling must avoid infinite loops: track resubmit count.
- "Subscription DLQ?" Each subscription has its own DLQ.

### 14.4 [Hard] One poison message is blocking throughput for a partition/session. What do you do right now and long term?
**What the interviewer is really testing**
Incident response plus design.

**Typical mid-level answer**
"Delete the message."

**Strong senior answer**
Now: identify it (delivery count, exception logs), dead-letter it to unblock, preserve the payload for analysis, and note the gap for any ordering/version logic and reconciliation. Don't delete without a record. Then root cause: bad data, schema change, bug. Fix, test with the captured message, resubmit. Long term: validate at the edge (schema validation at producer/consumer), classify errors, set sane MaxDeliveryCount, isolate poison via the DLQ quickly, add per-partition lag alerting, and contract tests. Failure: skipping may break an invariant; follow with reconciliation.

**Follow-up traps**
- "Skip vs stop-the-world?" For financially critical ordering you may deliberately stop and page rather than skip.
- "Replay after fix?" Might arrive out of order; consumers use versions.

### 14.5 [Scenario] A downstream API is down for 2 hours. Your consumer keeps retrying and fills the DLQ. Design a better approach.
**What the interviewer is really testing**
Backpressure and circuit breaking in async flows.

**Typical mid-level answer**
"Increase the delivery count."

**Strong senior answer**
Messages dead-lettered because a dependency is down aren't poison; they're casualties. Add a circuit breaker in the consumer: when the breaker opens, stop receiving (pause the processor or let locks expire without incrementing wasted work) or abandon with a long scheduled delay; resume when half-open probes succeed. Use delayed redelivery tiers (1 min, 5 min, 30 min). Keep messages in the queue (queue depth grows safely) rather than the DLQ; alert on queue age. Make sure TTL > expected outage. When the dependency recovers, the backlog drains at a controlled rate (limit concurrency) to avoid overwhelming it. Risk: an unbounded backlog; set alerts and capacity assumptions.

**Follow-up traps**
- "Abandon increments delivery count." Yes, so repeated abandons lead to DLQ: stop receiving instead.
- "Rate limit drain?" MaxConcurrentCalls and prefetch.

**Red flags in answers**
- Retrying all errors including 4xx validation failures.
- Infinite immediate retries without backoff/jitter.
- Deleting poison messages without capturing them.

**Cheat sheet**
- Timeout = unknown; retry only idempotent or keyed operations.
- Backoff + jitter + cap + breaker; one retry layer.
- Classify transient vs permanent; dead-letter with reason.
- DLQ needs owner, alert, runbook, resubmit tool.
- Dependency outage: pause consumption, don't burn delivery counts.

---

## Topic 15: Distributed IDs, clock skew, why timestamps lie

**Study notes**
- ID options: GUID v4 (random; fragmenting as clustered index), sequential GUID (`NEWSEQUENTIALID()` in SQL, or `Guid.CreateVersion7()` in .NET 9), ULID (48-bit time + 80-bit random, lexicographically sortable), Snowflake-style 64-bit (timestamp + machine id + sequence; needs worker id management), DB identity/sequence (central, simple, not client-generated).
- Client-generated IDs enable idempotent creates and avoid round trips but need uniqueness without coordination.
- Clocks: wall clocks drift and skew between machines; NTP can step backwards; leap seconds; VMs pause. `DateTime.UtcNow` is not monotonic. Use `Stopwatch` for durations. Ordering across machines by wall-clock is unreliable.
- Logical clocks: Lamport timestamps, vector clocks; hybrid logical clocks (HLC) combine physical and logical; versions per aggregate are simplest.

### 15.1 [Easy] GUID vs int identity vs ULID: how do you pick a primary key?
**What the interviewer is really testing**
Practical SQL Server effects and distributed needs.

**Typical mid-level answer**
"GUIDs are unique, so use them."

**Strong senior answer**
Need client/service-generated IDs (idempotent creates, offline, merging data from services)? Then GUID/ULID. Random GUIDs as a clustered key cause page splits and fragmentation in SQL Server; mitigate with sequential GUIDs (`NEWSEQUENTIALID()`, `Guid.CreateVersion7()` in .NET 9, or app-side sequential algorithms), or use a narrow clustered key (bigint identity) and keep the GUID as a non-clustered unique key. If IDs are only DB-generated and not exposed, identity is simplest and smallest. Exposing sequential ints leaks counts and allows enumeration; public IDs should be non-guessable. Risk: mixing generators inconsistently.

**Follow-up traps**
- "ULID in SQL Server?" Store as binary(16) or uniqueidentifier with byte-order caveats (SQL Server sorts GUIDs unusually).
- "GUID collisions?" Negligible for v4.

### 15.2 [Follow-up] How does a Snowflake-style ID work and what are its pitfalls?
**What the interviewer is really testing**
Understanding structure and operational risks.

**Typical mid-level answer**
"Twitter's unique ID generator."

**Strong senior answer**
A 64-bit ID composed of a timestamp (ms since custom epoch), a worker/machine id, and a per-ms sequence. Time-sortable, compact, no central coordination at generate-time. Pitfalls: worker id assignment must be unique (reuse after pod restart or autoscale causes collisions), clock moving backwards produces duplicates or must block/refuse, sequence overflow within a ms, and ordering is only approximate across machines. In Kubernetes/App Service, assigning worker ids is the hard part (lease from a table/blob). If I don't need 64-bit or strict time ordering, GUID v7 or ULID avoid worker-id management. Failure: collisions produce primary key violations or silent overwrites with upserts.

**Follow-up traps**
- "Clock goes backwards?" Wait/refuse to generate until time catches up.
- "Sortable means ordered?" Roughly, not strictly across nodes.

### 15.3 [Medium] Why shouldn't you order events by timestamp across services?
**What the interviewer is really testing**
Clock realism.

**Typical mid-level answer**
"Timestamps may be wrong."

**Strong senior answer**
Machines' clocks differ by milliseconds to seconds (NTP corrections can step time backwards), and events with the same timestamp tie; VMs can pause and resume. So "A happened before B" by wall-clock can be false, and "last write wins" by timestamp can silently discard the newer real write. Causality is better captured by: per-aggregate version numbers, sequence from a single writer, logical/hybrid logical clocks, or broker-assigned order within a partition/session. Wall-clock timestamps remain useful for display, TTLs, and coarse windows. Use DB time (`SYSUTCDATETIME()`) from the single authority when a consistent clock is needed, e.g. lock expiry. Risk: LWW conflict resolution with skewed clocks loses data.

**Follow-up traps**
- "Use Azure time sync?" Reduces skew, doesn't eliminate it.
- "Measure durations?" Monotonic Stopwatch, not DateTime differences.

### 15.4 [Hard] Design IDs for an order system with services in 3 regions that create orders offline and sync later.
**What the interviewer is really testing**
Uniqueness without coordination, merge, and human-friendly numbers.

**Typical mid-level answer**
"Use GUIDs."

**Strong senior answer**
Internal identity: client-generated GUID/ULID (v7 for index locality), created once and persisted before send, doubling as an idempotency key: replays from sync don't duplicate. Human-facing order number: a separate business identifier allocated at sync/confirmation by a central sequence (gap-tolerant) or region-prefixed sequence (`EU-000123`), because users need short numbers and finance wants sequences; if legal demands gapless numbers, allocate only at the final confirmation step in one place. Keep both: stable internal id and display number. Risk: assuming the display number exists at creation time; UX shows "pending number".

**Follow-up traps**
- "Gapless invoice numbers?" Requires single sequence allocation at issuing, with its own failure handling; identity gaps happen on rollback.
- "Leak info?" Prefix/regions reveal structure; acceptable?

### 15.5 [Tricky] Your "last write wins by UpdatedAt" sync is losing updates. Diagnose.
**What the interviewer is really testing**
Linking clock skew to data loss and alternatives.

**Typical mid-level answer**
"Add more precise timestamps."

**Strong senior answer**
Likely causes: client or server clock skew so an older edit has a later timestamp; same-timestamp ties; timezone/DST bugs; and conceptual: LWW ignores concurrent edits so one is lost by design. Fixes: server-assigned time from one clock (or better, a version), compare versions with If-Match, detect concurrent edits (vector/version comparison) and merge or flag conflicts, field-level merge for disjoint fields. For counters/sets use mergeable types (CRDTs). Add audit logging of discarded writes so loss is visible. Risk: choosing LWW for data where a lost edit is costly.

**Follow-up traps**
- "Server timestamp is enough?" Still LWW; loses concurrent edits.
- "How detect concurrency?" Version vectors or base-version in the update.

**Red flags in answers**
- Ordering distributed events by DateTime.UtcNow.
- Random GUID clustered key with no fragmentation awareness.
- Snowflake IDs with no plan for worker id uniqueness.

**Cheat sheet**
- Client-generated IDs: GUID v7/sequential/ULID; keep clustered key narrow if needed.
- Snowflake needs unique worker ids and clock-backwards handling.
- Wall clocks skew and step; use versions/logical clocks for causality.
- Display numbers are separate from internal IDs.
- LWW by timestamp can silently lose data.

---

## Topic 16: Data replication and reference-data caching between services

**Study notes**
- Services need other services' data (customer name, price, tax rules). Options: synchronous call per use (coupling, latency, availability), replicate a local copy via events (autonomy, staleness), cache with TTL, or share via a read API/CDN.
- Event-carried state transfer: events include enough data to update local copies; local copy is owned by source, read-only here.
- Staleness is a business decision: tax rate by effective date vs product description.
- Cache invalidation: TTL, event-driven invalidation, versioned keys, stampede protection (single-flight), negative caching, stale-while-revalidate.
- Bootstrap/backfill: new consumer needs initial load (snapshot + stream from position) and periodic reconciliation.

```
Catalog svc --ProductChanged(v7, name, price)--> Orders svc local ProductRef(Id, Version, Name, Price)
Apply only if v > stored. Backfill via snapshot API/event replay when new.
```

### 16.1 [Easy] Should Orders call Customer service on every request, or keep a copy?
**What the interviewer is really testing**
Coupling vs autonomy trade-off.

**Typical mid-level answer**
"Call it; one source of truth."

**Strong senior answer**
Clarify: how fresh must it be, how often is it read vs changed, and what happens if Customer is down? Synchronous calls on the hot path couple availability (Orders is down when Customer is) and add latency. A local read-only replica fed by events keeps Orders autonomous, at the cost of staleness and sync machinery. For slowly changing reference data, replicate/cache; for data where staleness is dangerous (credit status at charge time), call the owner or verify at commit. Capture snapshots of needed values into the order at placement (e.g. shipping address, price) because the order should reflect what was agreed, not later changes. Risk: copies drifting; reconcile.

**Follow-up traps**
- "Who owns the copy?" Source service; consumers never write back.
- "Price changes after order?" Snapshot into the order.

### 16.2 [Follow-up] How do you keep a local replica in sync and bootstrap a new consumer?
**What the interviewer is really testing**
Operational completeness.

**Typical mid-level answer**
"Subscribe to events."

**Strong senior answer**
Event subscription alone misses history before the subscription and any lost events. Bootstrap: take a consistent snapshot from the owner (API export or table copy at a known version/position), load it, then apply events after that position (deduped by version). Updates: each event carries entity id + version; apply if newer; deletes as tombstone events. Safety net: periodic reconciliation comparing counts/checksums or version lists with the source and repair by refetching differing rows. Monitor lag and DLQ for this feed. Trade-off: event-carried state transfer duplicates data and PII (privacy: replicate only needed fields and honour erasure events).

**Follow-up traps**
- "Events during snapshot?" Buffer events from before snapshot completes; version checks dedupe.
- "GDPR?" Propagate deletion events; purge copies.

### 16.3 [Medium] Cache design for reference data in a scaled-out App Service: what could go wrong?
**What the interviewer is really testing**
Cache pitfalls.

**Typical mid-level answer**
"Use IMemoryCache with a TTL."

**Strong senior answer**
Per-instance memory caches diverge: each instance refreshes at different times, so users can see different values per request. Options: distributed cache (Azure Cache for Redis) for shared view, or in-memory with short TTL plus event-driven invalidation (Service Bus topic subscription per instance, or Redis pub/sub) when staleness must be bounded. Stampede: on expiry many requests hit the source; use single-flight (lock per key) and jittered TTLs, plus stale-while-revalidate. Failure of cache: fall back to source with limits; don't let a cold cache take down the owner (rate limiting). Negative caching for not-found with short TTL. Risk: caching authorisation or price data with long TTLs.

**Follow-up traps**
- "HybridCache in .NET 9?" Provides L1+L2 and stampede protection.
- "Cache the null?" Yes, briefly.

### 16.4 [Hard] The price shown to the customer was cached and is now wrong at checkout. How do you design for it?
**What the interviewer is really testing**
Authoritative check at commit with good UX.

**Typical mid-level answer**
"Reduce the TTL."

**Strong senior answer**
Separate display from commitment. The product page may use cache/replica, but checkout re-validates the price from the authoritative source (or a recent versioned value) and compares with the price the customer saw (client sends price version). If different, tell the user and ask to confirm; business policy might honour the displayed price within a short window (price-lock/quote with an expiry). Shorter TTL reduces frequency but doesn't remove the race. Also record the price version in the order for audit. Risk: too strict rejections annoy customers; too lax loses margin. Make this a business rule, not an accident.

**Follow-up traps**
- "Price lock?" Quote with id and expiry.
- "Promotion end at midnight?" Effective-dated pricing, not cache timing.

### 16.5 [Scenario] A consumer's local copy of customers is missing 2% of records after a broker outage. Fix and prevent.
**What the interviewer is really testing**
Repair design.

**Typical mid-level answer**
"Re-run the import."

**Strong senior answer**
Diagnose: dead-lettered or expired messages (TTL), consumer errors, or events published before outbox caught up. Immediate repair: identify differences by comparing id/version lists against the source (or request a full snapshot), refetch and upsert missing/older rows; version checks make it idempotent. Check DLQ and resubmit. Prevent: monitor oldest outbox message and DLQ, set TTL beyond expected outage, schedule periodic reconciliation (e.g. nightly checksum per range), and make "resync entity X" an admin operation (request-reply or republish command). Failure: reconciliation hitting the owner hard; throttle and run off-peak.

**Follow-up traps**
- "Why not rely on events?" Any pipeline can lose or skip; reconciliation is the safety net.
- "Hash comparison?" Range hashes localise differences cheaply.

**Red flags in answers**
- Per-instance in-memory cache with no plan for divergence.
- Replicating data with no bootstrap or reconciliation.
- Re-reading mutable reference data for a historical order instead of snapshotting.

**Cheat sheet**
- Call for fresh-critical data; replicate for autonomy; snapshot into transactions.
- Replica = snapshot + events after position; apply if version newer.
- Reconcile periodically; provide a resync operation.
- Cache: jittered TTL, single-flight, invalidation events, negative caching.
- Display may be stale; commit re-validates.

---

## Topic 17: Reconciliation, audit/repair, and testing distributed failure

**Study notes**
- Reconciliation: periodic comparison of two systems' views of the same facts (counts, sums, per-key versions) to detect divergence; classify, repair or escalate. It's the safety net behind every eventually consistent design.
- Audit trail: append-only record of who/what/why changed state, including automated repairs; essential for trust and debugging.
- Repair safety: idempotent repair actions, dry-run mode, approval for risky classes, rate limits, never delete first.
- Testing failure: unit-test state machines/idempotency; integration tests with duplicated, reordered, delayed, lost messages; crash-injection between steps; chaos experiments (Azure Chaos Studio), fault injection proxies (Toxiproxy), Polly chaos strategies (Simmy), Testcontainers with SQL Server/Service Bus emulator; consumer-driven contract tests.

```
Source of truth S ----extract(range, version/hash)----> Reconciler <----extract---- Replica R
Reconciler: missing in R | extra in R | version mismatch -> classify -> auto-repair | ticket | alert
```

### 17.1 [Easy] Why do eventually consistent systems need reconciliation?
**What the interviewer is really testing**
Systems humility.

**Typical mid-level answer**
"To fix bugs."

**Strong senior answer**
Because every pipeline has residual failure modes: dropped events beyond retention, bugs, manual DB edits, bad deploys, partially applied sagas, expired dedupe windows. Reconciliation turns "we believe it converges" into "we measure that it does". It compares the owner's data with downstream copies (or our payments with the provider's settlement report), reports differences, and repairs the safe categories. It gives a bounded detection time (e.g. daily) rather than relying on customers. What fails without it: silent drift accumulates and trust erodes.

**Follow-up traps**
- "Frequency?" Based on business harm and cost: hourly for money, daily for catalog.
- "Cost?" Use incremental windows and hashes.

### 17.2 [Follow-up] Design a safe automated repair process.
**What the interviewer is really testing**
Risk management of auto-fix.

**Typical mid-level answer**
"A script that fixes the data."

**Strong senior answer**
Start read-only: produce a diff report and metrics. Classify discrepancies and define a repair for each with an owner. Auto-repair only classes that are deterministic and reversible (re-publish a missing event, refetch a replica row); everything else generates a ticket with context. Repairs go through the normal path (re-emit events/commands with idempotency keys) rather than direct DB edits, so invariants and downstream consumers stay consistent. Add dry-run mode, rate limits, a kill switch, and audit entries (who/what/when/before/after). Re-run reconciliation to verify convergence. Risk: a buggy repair at scale amplifies damage; roll out to a small range first.

**Follow-up traps**
- "Direct SQL fix?" Last resort, scripted, reviewed, audited.
- "Repair loops?" Guard with max attempts and alert.

### 17.3 [Medium] How would you test that your outbox + consumer is correct under failure?
**What the interviewer is really testing**
Practical failure-testing skill.

**Typical mid-level answer**
"Unit tests with mocks."

**Strong senior answer**
Layers: unit tests for the idempotency check and state transitions; integration tests against real SQL Server (Testcontainers or LocalDB) and a Service Bus emulator/real namespace, injecting faults at each step: crash after commit before publish (relay must publish later), crash after publish before marking sent (duplicate arrives; consumer dedupes), crash after side effect before Complete (redelivery, no double effect), out-of-order and duplicate delivery. Property-style test: random delivery order/duplicates of a message set must produce the same final state. Assert metrics/DLQ behavior too. For concurrency, run many parallel consumers on the same key. Risk: tests that mock the broker can't reveal lock-expiry behavior; keep a few tests on the real thing.

**Follow-up traps**
- "How inject crash?" Test hooks/fault points, kill process in test harness, or throw at defined step.
- "Test time-based behavior?" Fake clock/TimeProvider.

### 17.4 [Hard] How do you introduce chaos/failure injection safely in production-like environments?
**What the interviewer is really testing**
Mature resilience practice.

**Typical mid-level answer**
"Randomly kill servers."

**Strong senior answer**
Start with a hypothesis tied to an SLO ("if Payment is slow 2 s, checkout p95 stays under 3 s and no duplicate charges"). Define steady-state metrics, limit blast radius (one environment, percentage of traffic, time-boxed), have abort criteria and a kill switch, and run in staging first, then production during staffed windows. Tools: Azure Chaos Studio (service faults, e.g. stop VM/AKS faults, network latency), Toxiproxy/Polly chaos (latency, errors), Service Bus tests like pausing consumers or forcing lock expiry. After each experiment: document findings, fix gaps (missing timeouts, no idempotency), automate as regression. Risk: experimenting without observability yields noise.

**Follow-up traps**
- "What first?" Dependency latency and failure; they expose retry and timeout bugs.
- "Money flows?" Use sandbox provider and synthetic accounts.

### 17.5 [Scenario] After a deploy, 300 orders show as Paid but have no shipment. How do you handle the incident and learn from it?
**What the interviewer is really testing**
Incident handling combining reconciliation, replay and prevention.

**Typical mid-level answer**
"Roll back and fix manually."

**Strong senior answer**
Stabilise: stop further damage (rollback or feature flag), quantify via query (Paid orders without Shipment event older than SLA) and capture scope. Find the cause: DLQ for the shipping command, schema mismatch from the deploy, consumer crash. Repair by replaying the affected events/commands (idempotent) from the outbox/DLQ or re-emitting via a repair job; verify with the reconciliation query; communicate with support/customers. Postmortem: why did tests not catch it (contract test missing), why detection took X (no alert on Paid-without-shipment age), add alert and automatic reconciliation, add a deploy check on DLQ growth. Risk: replaying blindly duplicates shipments; check idempotency by order id.

**Follow-up traps**
- "Replay vs recreate?" Replay preserves original intent and ids.
- "Blameless?" Focus on system gaps.

**Red flags in answers**
- No reconciliation or detection for cross-system invariants.
- Repairing data with ad-hoc SQL that bypasses events and audit.
- Testing only the happy path with mocked brokers.

**Cheat sheet**
- Reconcile: compare, classify, auto-fix safe classes, ticket the rest.
- Repair through normal paths with idempotency keys; audit everything.
- Test each crash point: after commit, after publish, before ack.
- Duplicate/reorder/delay/drop messages in tests.
- Chaos: hypothesis, blast radius, abort criteria, observability.

---

## Topic 18: Azure specifics (Service Bus, Durable Functions, Cosmos, Event Grid)

**Study notes**
- Service Bus: peek-lock (at-least-once) vs receive-and-delete (at-most-once); MaxDeliveryCount; lock duration; DLQ; scheduled messages; **duplicate detection** (by MessageId within a history window, default 10 min, max 7 days; must be enabled when creating the entity); **sessions** (FIFO per SessionId, session state, session-enabled entity required); **transactions** (`ServiceBusClient` with `CreateSender`, `TransactionScope(TransactionScopeAsyncFlowOption.Enabled)`: atomic across operations on Service Bus entities in one namespace, e.g. complete incoming + send outgoing; "send via" for cross-entity). Transactions do NOT include SQL Server operations.
- Durable Functions: orchestrator (deterministic, replayed), activities (at-least-once), entities, timers, external events, sub-orchestrations, fan-out/fan-in, `ContinueAsNew`.
- Cosmos DB change feed: ordered log of inserts/updates per logical partition key (latest version mode does not capture deletes: use soft-delete + TTL); at-least-once; consumed via Change Feed Processor / Azure Functions trigger with leases; transactional batch within one partition key.
- Event Grid: push, at-least-once, no ordering guarantee, retries with exponential backoff (default up to 30 attempts / 24 h event TTL), optional dead-lettering to blob storage. Good for reactive notifications; use Service Bus for commands/workflows needing sessions, DLQ in queue, transactions.

```
SQL + Service Bus outbox:  [Orders tx: row + Outbox row] -> Relay -> SB topic (MessageId=OutboxId; dup detection)
Consumer: receive (peek-lock) -> [SQL tx: Inbox + effect] -> Complete
Alternative on SB-only hop: TransactionScope{ Complete(in) + Send(out) } atomic within namespace
```

### 18.1 [Easy] Service Bus duplicate detection: what does it do and not do?
**What the interviewer is really testing**
Accurate feature scope.

**Typical mid-level answer**
"It makes processing exactly-once."

**Strong senior answer**
It drops a message sent with a MessageId already seen on that entity within the configured time window (default 10 minutes, up to 7 days), which protects against producer retries (e.g. the outbox relay resending after a crash). It does not stop redelivery of a message to a consumer, doesn't cover duplicates outside the window, different MessageIds for the same logical event, or duplicates produced upstream. So it complements but doesn't replace consumer idempotency. Setting it needs the feature enabled on the queue/topic at creation. Failure: a relay outage longer than the window resends and duplicates pass.

**Follow-up traps**
- "MessageId choice?" Deterministic (outbox id/business key), not random per send.
- "Cost?" Slight throughput impact and storage of ids.

### 18.2 [Follow-up] What can a Service Bus transaction cover, and what can't it?
**What the interviewer is really testing**
Avoid mistaken atomicity claims.

**Typical mid-level answer**
"Atomic send and database write."

**Strong senior answer**
A Service Bus transaction groups operations (send, complete, abandon, dead-letter, defer) against Service Bus entities in the same namespace so they commit together, e.g. complete the input message and send the output message atomically, avoiding the duplicate/loss window between them (with .NET `TransactionScope` and `TransactionScopeAsyncFlowOption.Enabled`). It cannot include SQL Server, Cosmos, or HTTP calls. Receive operations in a transaction are limited (one entity; with send via to other entities). Therefore: consume from queue A -> transform -> publish to B is solved with the SB transaction; DB + SB still needs outbox/inbox. Risk: relying on TransactionScope to escalate to MSDTC for SQL; don't.

**Follow-up traps**
- "Cross-namespace?" Not covered.
- "Partitioned entity?" Transaction operations need the same partition key/session.

### 18.3 [Medium] Choose between Service Bus and Event Grid for domain events.
**What the interviewer is really testing**
Picking the right Azure messaging service.

**Typical mid-level answer**
"Event Grid is newer and serverless."

**Strong senior answer**
Event Grid: reactive, push-based fan-out, filtering, great for Azure resource events and lightweight notifications; at-least-once with retries over up to 24 h then drop or dead-letter to blob, no ordering, consumer must be an HTTP endpoint (or other handlers) that responds fast. Service Bus: enterprise messaging: pull with peek-lock, sessions for ordering, DLQ, duplicate detection, transactions, scheduled delivery, larger controls for backpressure. For business workflows (orders, payments), commands and sagas I choose Service Bus; for "something happened, whoever cares react" notifications Event Grid is fine. Could combine: Event Grid routes to a Service Bus queue to gain buffering. Failure: consumer down for > retry window with no dead-letter loses events.

**Follow-up traps**
- "Ordering in Event Grid?" Not guaranteed.
- "Event Hubs?" For high-throughput streaming/partitioned log, not queue semantics.

### 18.4 [Hard] Design a Cosmos DB change feed based projection/outbox and discuss pitfalls.
**What the interviewer is really testing**
Understanding change feed semantics.

**Typical mid-level answer**
"Trigger a Function on changes."

**Strong senior answer**
Cosmos has no cross-service transaction, but the change feed gives a durable, ordered (per logical partition key) log of changes to a container, so writing the business document is itself the "outbox": consumers (Change Feed Processor or Functions Cosmos trigger) get at-least-once delivery with checkpointing in a lease container. Pitfalls: ordering only within a partition key; deletes not seen in latest-version mode (use soft delete + TTL); only the latest state per change (intermediate updates may be coalesced), so don't rely on every intermediate version; duplicates on restart so handlers must be idempotent; processing lag and RU on the lease container; poison documents block a lease range unless handled (catch, log, send to a dead-letter container). If every individual state transition matters, write explicit event documents (event store pattern) instead of relying on state snapshots.

**Follow-up traps**
- "Multi-document atomicity?" Transactional batch within one partition key only.
- "Replay?" Start from beginning or a time for new consumers.

### 18.5 [Scenario] Build "place order" on Azure with SQL Server, Service Bus, payment API and Durable Functions. Walk through the consistency design.
**What the interviewer is really testing**
Integrated end-to-end design with failure analysis.

**Typical mid-level answer**
"API writes DB, sends to Service Bus, Function calls payment."

**Strong senior answer**
API: validate, take an Idempotency-Key, in one SQL transaction insert Order(Pending) + Outbox(OrderPlaced); return 202 with status URL. Relay publishes to Service Bus (MessageId = outbox id; duplicate detection). A Durable Functions orchestrator starts per order (instance id = OrderId makes starting idempotent), calls activities: ReserveStock (idempotent by OrderId), AuthorizePayment (provider idempotency key = OrderId), then CapturePayment as the pivot with retry policy, then ShipOrder; compensation activities (ReleaseStock, VoidAuthorization) on failures; `WaitForExternalEvent` with timeout for async provider confirmation. Activities write via local SQL transactions with inbox/dedupe. Orchestrator history is the saga state; custom status exposes progress for the status URL. Observability: correlation id across logs, alerts on orchestrations running longer than SLA and DLQ depth. Reconciliation compares provider charges to orders. Failure analysis: activity crash -> retried (idempotent); orchestrator host crash -> replay; relay crash -> duplicate publish deduped; payment unknown -> query provider before compensating.

**Follow-up traps**
- "Orchestrator deterministic?" Yes; no DateTime.Now/random/direct I/O.
- "Where's the outbox needed?" Between the SQL write and starting the workflow.

**Red flags in answers**
- Claiming Service Bus transactions include SQL operations.
- Treating Event Grid as ordered/guaranteed forever.
- Non-deterministic code in orchestrators.

**Cheat sheet**
- SB duplicate detection: MessageId within window; producer-side only.
- SB transactions: SB entities within namespace; not SQL.
- Sessions for per-key FIFO; DLQ per entity/subscription.
- Cosmos change feed: ordered per partition key, at-least-once, deletes need soft-delete.
- Event Grid: push, retries, no ordering; Service Bus for workflows.
- Durable Functions: deterministic orchestrators, idempotent activities.

---

## Topic 19: Inventory reservation, double-booking, multi-region write conflicts

**Study notes**
- Double-booking arises from check-then-act races (read availability, then write). Fix by making the claim a single atomic conditional write: unique constraint, `UPDATE ... WHERE Available >= @q`, ETag/rowversion check, or single-writer serialisation (session per resource).
- Reservation pattern: hold with expiry (TTL), then confirm or release; sweepers release expired holds; reservation ids are idempotency keys.
- Overselling policy is a business choice: strict (never oversell) vs allow with backorder/compensation.
- Multi-region writes: single-writer (home region per entity) avoids conflicts; multi-writer needs conflict resolution: last-write-wins (clock/version based, loses data), custom merge, or CRDTs (data types that merge deterministically: G-Counter, PN-Counter, OR-Set, LWW-Register). Cosmos DB multi-region writes support LWW by a numeric path (default `_ts`) or custom conflict resolution procedure/conflict feed.

```sql
-- seat booking: unique constraint is the arbiter
CREATE UNIQUE INDEX UX_Booking_Seat ON dbo.Booking(EventId, SeatId) WHERE Status IN ('Held','Confirmed');
-- stock: atomic conditional decrement
UPDATE dbo.Stock SET Available = Available - @q WHERE Sku=@s AND Available >= @q;
```

### 19.1 [Easy] Two customers book the last seat at the same time. How do you prevent a double booking?
**What the interviewer is really testing**
Atomic claim vs check-then-act.

**Typical mid-level answer**
"Check availability first, then book." Racy.

**Strong senior answer**
Make the database arbitrate: a unique index on (Event, Seat) for active bookings, or a conditional update on the seat row (`WHERE Status='Free'`) and check rows affected. The loser gets a clear conflict and is offered alternatives. Add a short hold (e.g. 10 minutes) during checkout, so a seat isn't contested while the user pays, with expiry releasing it. Under high contention (ticket drop) put requests through a queue/session per event so they are serialised, protecting the database. What fails when this fails: without atomic claim, double booking; with holds but no expiry sweeper, ghost reservations block inventory.

**Follow-up traps**
- "SERIALIZABLE isolation?" Works but costs concurrency; constraints are cheaper.
- "Hold expiry race with payment?" Confirm checks hold still valid and extends or re-claims.

### 19.2 [Follow-up] Design the inventory reservation lifecycle across Orders, Inventory and Payment.
**What the interviewer is really testing**
Reservation states, idempotency and expiry.

**Typical mid-level answer**
"Decrement stock when ordering, add back if cancelled."

**Strong senior answer**
Inventory owns stock and exposes Reserve(orderId, sku, qty), Confirm(orderId), Release(orderId). Reserve is idempotent by (orderId, sku): inserts a reservation row with status Held and expiry and decrements Available atomically in one local transaction. Confirm moves Held to Committed when payment is captured; Release returns quantity, on payment failure, cancellation or timeout via sweeper. Available = OnHand - Held - Committed is derived from the reservation ledger for reconcilability. Messages carry orderId; duplicates are no-ops; late Release after Confirm is rejected by state rules. Risks: expiry shorter than payment duration releases stock the customer is paying for; confirm should handle "expired but still available -> re-reserve else compensate with refund".

**Follow-up traps**
- "Partial availability?" Business policy: partial ship, backorder or fail.
- "Multi-warehouse?" Reserve per warehouse; allocation strategy is a separate decision.

### 19.3 [Medium] Overselling is acceptable up to 2% in a retail business. How does that change the design?
**What the interviewer is really testing**
Using business tolerance to simplify.

**Typical mid-level answer**
Ignores it; designs for strict.

**Strong senior answer**
Tolerance allows an optimistic design: accept orders against an approximate, cached or replicated stock level, then fulfil from a periodic authoritative count, compensating (backorder, substitute, apology/discount, cancel+refund) for the oversold few. This removes the cross-service synchronous reservation from the hot path, improving availability and latency. I'd still bound the risk: stop selling when estimated stock hits a safety buffer, monitor oversell rate against the 2% budget, and have the compensation workflow ready. Cost: customer experience for oversold orders; limits for high-value or limited items (use strict mode for those). Failure: staleness during a partition may exceed the budget; the alerting triggers fallback to strict.

**Follow-up traps**
- "Which items strict?" Limited editions, tickets, regulated goods.
- "Dynamic switching?" Feature flag by SKU class.

### 19.4 [Hard] Active-active multi-region writes: what conflict resolution strategies exist and how do you choose?
**What the interviewer is really testing**
Conflict resolution trade-offs, CRDT awareness.

**Typical mid-level answer**
"Last write wins."

**Strong senior answer**
First ask if multi-writer is necessary: single-writer per entity (home region by customer or tenant, with failover) avoids conflicts entirely and is my default for business entities. If true active-active is needed: (1) LWW: simple, deterministic, but depends on clocks/version path and silently discards concurrent writes: acceptable for idempotent state like "last seen" or profile preferences. (2) Custom merge: application-level logic (field-level merge, domain rules such as "cancelled beats shipped" ), possibly via Cosmos custom conflict resolution or conflicts feed. (3) CRDTs: data structures that merge without coordination (counters, sets, registers): good for likes, carts (OR-set), presence; not for invariants like "balance >= 0" or "one winner for a seat". For invariants needing uniqueness, route those writes to a single region/leader. Risks: unbounded conflict complexity; test with partition simulations.

**Follow-up traps**
- "Can CRDT enforce non-negative stock?" No, needs coordination (escrow/reservation rights per region).
- "Cosmos default conflict policy?" LWW on `_ts`, configurable to custom path or manual.

### 19.5 [Scenario] A ticketing site must handle 50k users hitting a 5k-seat event at 10:00. Design for correctness and survival.
**What the interviewer is really testing**
Combining queueing, atomic claims, and UX under load.

**Typical mid-level answer**
"Scale out the web servers and the database."

**Strong senior answer**
Protect the invariant and the database at the same time. Admission control: a virtual waiting room/queue admitting a controlled rate of users to checkout. Seat claim: atomic conditional write/unique index per seat, with holds (TTL ~8 min), and per-event serialisation via Service Bus session or partition to cut lock contention if needed. Display: seat map from a cache/replica with short TTL and push updates, expecting stale entries, with graceful "seat just taken" handling. Payment: after hold, idempotent payment; confirm hold on success, release on failure/timeout via sweeper. Rate-limit bots, idempotency keys for retries. Capacity: test with load and failure injection. Failure analysis: payment provider slow -> holds expire; queue longer expiry vs fairness; DB throttling -> waiting room reduces admission rate dynamically.

**Follow-up traps**
- "Fairness?" Queue ordering/lottery is a product decision.
- "Hold storms?" Limit holds per user/session.

**Red flags in answers**
- Check-then-insert without a unique constraint or conditional update.
- Holds with no expiry or sweeper.
- Using LWW or CRDT for invariants like "only one winner".

**Cheat sheet**
- Atomic claim: unique index / conditional UPDATE / version check.
- Reservations: Held -> Committed/Released, idempotent, with expiry.
- Business tolerance can justify optimistic oversell + compensation.
- Multi-region: prefer single-writer per entity; LWW loses data; CRDTs for commutative data only.
- Protect hot resources with admission control and per-key serialisation.

---

## Topic 20: Decision tree: which consistency mechanism for which problem

**Study notes**
- Start from the invariant and its blast radius, not from the technology.
- Use the weakest mechanism that preserves the invariant at acceptable business risk; add reconciliation always.

```
Is the invariant within one aggregate/database?
 +-- yes -> local ACID transaction (unique constraints, conditional updates, rowversion)
 +-- no: must it hold at every instant across services?
      +-- yes -> reconsider boundaries (merge into one service/DB) or single-owner reservation/lock-by-ownership;
      |         2PC only if all participants support it and are in one failure domain
      +-- no (can be eventually true):
           +-- need to publish an event after a DB change?  -> Transactional Outbox (+ idempotent consumer / Inbox)
           +-- multi-step business process with undo?       -> Saga (orchestrate if >3 steps/timeouts; choreograph if simple)
           +-- duplicates possible (retries/redelivery)?    -> Idempotency key + unique constraint / dedupe store
           +-- concurrent edits of same data?               -> Optimistic concurrency (ETag/rowversion); atomic UPDATE for hot rows
           +-- need per-entity order?                       -> Sessions / partition key + version numbers
           +-- need exclusive job/resource access?          -> Avoid lock; else lease + fencing token / idempotent job
           +-- copy of other service's data?                -> Event-carried state + snapshot bootstrap + reconciliation
           +-- multi-region writers?                        -> Single-writer per entity; else merge/CRDT (non-invariant data)
 Always: timeouts, DLQ + alerts, reconciliation, observability (correlation id)
```

### 20.1 [Medium] Walk me through how you decide which mechanism to use for a new cross-service requirement.
**What the interviewer is really testing**
Structured decision-making.

**Typical mid-level answer**
"Use a saga" (default hammer).

**Strong senior answer**
I ask five questions. What is the invariant and the cost of violating it (money, legal, UX)? Does it fit in one aggregate/service (if so, local transaction, done)? How fresh must the data be (seconds, minutes)? What are failure modes: duplicates, loss, reordering, partial completion? What can the team operate? From there: events after DB change -> outbox; multi-step flow with undo -> saga; duplicate risk -> idempotency; concurrent edits -> versions; ordering needed -> sessions plus versions. Then I add observability, timeouts, DLQ and reconciliation. Finally I write down the chosen consistency promise (e.g. "order status within 5 s, eventually consistent, reconciled daily") so product and engineering agree. Risk: over-engineering; I prefer the simplest mechanism that meets the stated risk.

**Follow-up traps**
- "When merge services?" If two services always change together and need atomicity, the boundary is wrong.
- "Document where?" ADR with failure analysis.

### 20.2 [Scenario] Match mechanisms: (a) email after signup, (b) transfer between accounts in different services, (c) profile edit from two tabs, (d) nightly report job on 4 instances, (e) price list copied to 5 services.
**What the interviewer is really testing**
Fast, correct mapping with rationale.

**Typical mid-level answer**
Uses sagas or locks for all of them.

**Strong senior answer**
(a) Outbox publishes UserRegistered; email consumer idempotent by event id (duplicate email is low harm, so dedupe cheaply); retries and DLQ. (b) Money: single owner per account with local transaction; across services a saga with reserve/debit then credit and compensation, ledger-based entries for idempotency and reconciliation, or better, keep both accounts in one ledger service so it's one ACID transaction. (c) Optimistic concurrency via ETag/If-Match; 412 -> reload/merge. (d) Singleton via timer trigger or SQL claim row/applock; idempotent job output. (e) Event-carried state transfer with versions and snapshot bootstrap, TTL as safety net, nightly reconciliation. Failure view: (b) is the only one where a partial failure can cost money, so it gets the most rigorous mechanism.

**Follow-up traps**
- "Why not saga for (b) everywhere?" Better to avoid the distributed problem via boundary design.
- "(a) exactly once?" Not needed; harm is minimal.

### 20.3 [Tricky] An architect says "we'll use 2PC across 12 services for simplicity". Respond using the decision tree.
**What the interviewer is really testing**
Respectful but principled pushback.

**Typical mid-level answer**
"No, 2PC is bad."

**Strong senior answer**
I'd start by understanding the underlying need: probably "I don't want to think about partial failure". I'd show the availability math (12 dependencies: the transaction is available only when all are), lock durations across network hops, lack of enlistment support for Service Bus/Cosmos/payment APIs, and in-doubt blocking. Then propose alternatives per flow: merge services whose invariants need atomicity, outbox + saga for the rest, with a defined consistency promise and reconciliation, and a prototype of the hardest flow demonstrating failure behaviour. If specific pairs of SQL databases do need atomicity, scoped 2PC (elastic transactions/MSDTC) may be acceptable. Frame as a risk trade-off, with a proof-of-concept and failure test results rather than opinion.

**Follow-up traps**
- "Is saga simpler?" Not simpler; it's more explicit; the complexity exists anyway.
- "Cost of change?" Prototype the critical path first.

### 20.4 [Hard] What does "done" look like for a distributed workflow before you ship it to production?
**What the interviewer is really testing**
Operational readiness mindset.

**Typical mid-level answer**
"Tests pass."

**Strong senior answer**
Checklist: invariants and consistency promise documented; every step idempotent and keyed; outbox/inbox in place; timeouts and terminal states for every wait; compensations tested including failing compensations; pivot identified; ordering/versioning rules stated; DLQ with alerts, owner and replay tool; correlation id and traces across services; dashboards for lag, saga state age, DLQ depth, outbox age; reconciliation job with a repair runbook; failure-injection tests at each crash point; load test including dependency slowness; rollout with feature flag and ability to drain in-flight instances; capacity/retention (dedupe windows > retry horizon). Risk: shipping without a drain/versioning plan for in-flight sagas on the next deploy.

**Follow-up traps**
- "In-flight sagas on deploy?" Version state machines; keep old handlers until drained.
- "Who's on call?" Named owners for DLQ and reconciliation.

### 20.5 [Tricky] Everything above is "eventually consistent". How do you know the system is actually correct, and what do you tell the business?
**What the interviewer is really testing**
Evidence-based assurance and honest communication.

**Typical mid-level answer**
"We trust the framework."

**Strong senior answer**
Define correctness as measurable properties: no lost events (outbox age and unsent count), no duplicate effects (idempotency hit rates and reconciliation duplicates), convergence time (lag SLO), and cross-system invariants (e.g. charges equal orders) via reconciliation with an orphan-rate SLI. Provide dashboards and an agreed error budget. To the business: state the promise plainly ("order status updates within seconds; a small fraction may need minutes; money discrepancies are detected within an hour and corrected automatically or by support within a day"), list the visible intermediate states and what the user sees, and describe the manual fallback. Honest uncertainty with bounded detection time beats a false claim of perfection.

**Follow-up traps**
- "Zero discrepancies?" Not provable; bound detection and repair time.
- "Who owns an orphan?" Named team via routing in the reconciler.

**Red flags in answers**
- Picking a pattern before stating the invariant and failure modes.
- Promising "exactly once" or "never inconsistent" to the business.
- No operational plan (alerts, replay, reconciliation).

**Cheat sheet**
- Invariant first; keep atomic needs in one boundary.
- Event after DB change: outbox; multi-step with undo: saga; duplicates: idempotency.
- Concurrent edits: optimistic concurrency; ordering: sessions + versions.
- Locks only with fencing or idempotency; prefer constraints.
- Always add timeouts, DLQ alerts, correlation ids, reconciliation.
- State the consistency promise in business language.
