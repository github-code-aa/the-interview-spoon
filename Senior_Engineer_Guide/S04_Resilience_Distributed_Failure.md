# S04 - Resilience & Distributed Failure (Senior Engineer Interview Guide, .NET / Azure / SQL Server)

**How to use:** Read the study notes, then answer each ladder question aloud *before* reading the answer; the ladder escalates Easy -> Hard/Scenario.
Senior answers are reasoning-first: clarify -> options -> trade-offs -> decision -> risks, and always say "what fails when this fails?".

## Table of Contents
1. Fallacies of Distributed Computing
2. Failure Taxonomy (crash, omission, timing, partial, gray, byzantine-lite)
3. Timeouts Everywhere (budgets, deadlines, cancellation)
4. Retries Done Right
5. Circuit Breakers
6. Bulkheads & Isolation
7. Backpressure, Load Shedding, Load Leveling, Rate Limiting
8. Graceful Degradation, Fallbacks & Static Stability
9. Cascading Failure, Thundering Herd, Cache Stampede & Resilient Caching
10. Health Checks, Graceful Shutdown & Draining
11. Availability Math (SLA/SLO/SLI, nines, error budgets)
12. Disaster Recovery (RPO/RTO, failover, runbooks, game days)
13. Chaos Engineering & Testing Resilience
14. Resilient Messaging, Poison Messages, Dead-Lettering & Replay
15. Zero-Downtime Deployments, Feature Flags & Kill Switches
16. Resilient Data Access (EF Core, pooling, failover groups)
17. Autoscaling Pitfalls, Noisy Neighbours & Multi-Tenant Fairness
18. Dependency Management & Observability for Resilience
19. Scenario Drills (slow downstream, DB failover, retry storm, region outage, 99.99%)
20. Resilience Review Checklist for Architecture Reviews

---

## Topic 1: Fallacies of Distributed Computing

**Study notes**
The eight fallacies (Deutsch/Gosling): the network is reliable; latency is zero; bandwidth is infinite; the network is secure; topology doesn't change; there is one administrator; transport cost is zero; the network is homogeneous.
Practical consequences in .NET/Azure: every `HttpClient`, SQL, Service Bus, or Redis call can hang, fail, or succeed-but-lose-the-reply. A remote call is not a method call.

```
Caller --request--> Service --writes--> DB
   ^                   |
   +---- reply lost ---+   => caller sees timeout, work may HAVE happened
```
Key idea: a timeout means "I don't know", not "it failed".

### 1.1 [Easy] Name three fallacies of distributed computing and give a real incident each could cause.
**What the interviewer is really testing**
Whether you connect textbook items to production behaviour, not recite them.
**Typical mid-level answer**
"Network is reliable, latency is zero, bandwidth is infinite." Lists them without consequences; doesn't say what the code should do differently.
**Strong senior answer**
I'd pick the ones that bite most. "Network is reliable": a payment call times out, we retry, and the customer is charged twice because the first call actually succeeded; fix is idempotency keys. "Latency is zero": a chatty loop making 200 sequential calls to a service is fine in dev (1 ms) and unusable across regions (60 ms each = 12 s); fix is batching or coarser APIs. "Topology doesn't change": hard-coded IPs or a long-lived `HttpClient` that never re-resolves DNS keeps hitting a dead node after failover; fix is `PooledConnectionLifetime` on `SocketsHttpHandler` and using service names. What fails when this fails? In each case the failure is silent correctness or capacity loss rather than a clean error, which is why they are fallacies.
**Follow-up traps**
- "Is a timeout a failure?" (No - outcome unknown.)
- "Which fallacy matters for security-in-transit?" (Network is secure -> TLS/mTLS even internally.)

### 1.2 [Follow-up] How does "latency is zero" show up in a .NET codebase that looks perfectly clean?
**What the interviewer is really testing**
Spotting hidden remote calls: N+1 queries, repository abstractions, sync-over-async.
**Typical mid-level answer**
"Reduce the number of calls." Correct but doesn't identify where they hide or how to find them.
**Strong senior answer**
Clean abstractions hide remote cost: `foreach (var o in orders) await _customerClient.GetAsync(o.CustomerId)` or EF lazy loading causing N+1. Under load each hop adds tail latency, and p99 compounds - if a request fans out to 10 services each with p99 = 1% slow, ~10% of requests hit at least one slow call. Options: batch endpoints, `Include`/projection, parallel fan-out with a bounded `Task.WhenAll`, caching reference data, or moving data closer. Trade-off: parallel fan-out improves latency but multiplies load on the dependency. I'd find them with distributed tracing (many small spans in one trace) rather than code review. Risk if ignored: thread/connection pool saturation because each request now holds resources for the sum of latencies.
**Follow-up traps**
- "Why is p99 more important than average?" (Fan-out amplifies tails.)
- "Is Task.WhenAll with 1000 calls ok?" (No - unbounded concurrency.)

### 1.3 [Medium] "One administrator" and "topology doesn't change" - why do they matter to a team running on Azure?
**What the interviewer is really testing**
Operational thinking: other teams, vendors, and Azure itself change things under you.
**Typical mid-level answer**
"We should coordinate with other teams." Vague.
**Strong senior answer**
Your dependency is run by someone with a different release cadence, maintenance windows, and priorities. Azure itself does planned maintenance, VM moves, SQL failovers, and certificate rotations. So I'd design for: versioned, backward-compatible contracts (tolerant reader), explicit SLOs/contacts per dependency, no hard-coded endpoints or IPs, configuration for endpoints, health-model dashboards that include external dependencies, and subscription to Azure Service Health alerts. Trade-off: contract versioning and consumer-driven contract tests cost effort, but unannounced breaking change is a common outage cause. What fails when it fails: our service is up but returns wrong or erroring results because a dependency changed - the blast radius is bounded only if we validate and degrade gracefully.
**Follow-up traps**
- "How do you detect a breaking change before prod?" (Contract tests, canary.)
- "Who do you page when the vendor is down?" (Have runbook + escalation.)

### 1.4 [Hard] You are asked to call a synchronous HTTP API from inside a database transaction. Argue for or against.
**What the interviewer is really testing**
Understanding that a remote call inside a transaction couples lock duration to network behaviour.
**Typical mid-level answer**
"It could be slow." Doesn't explain lock holding, atomicity, or alternatives.
**Strong senior answer**
Against, in almost all cases. The transaction holds locks (and a pooled connection) for the full remote latency, so a slow dependency becomes blocking chains and pool exhaustion in SQL - the failure moves from the dependency into our database. Also there's no atomicity: if the HTTP call succeeds and the commit fails, or the reverse, state diverges, and a retry may repeat the side effect. Alternatives: do the DB work, commit, then call out; or use the transactional outbox (write intent in the same transaction, a relay performs the call with an idempotency key); or a saga with compensation if multiple steps. Trade-off: eventual consistency and more moving parts. I'd accept an in-transaction call only for a very fast, same-trust, read-only lookup, with a short timeout.
**Follow-up traps**
- "Can't we use distributed transactions (MSDTC)?" (Rarely available across cloud/HTTP services; couples availability.)
- "What does the user see during eventual consistency?" (Pending state.)

### 1.5 [Tricky] "Our network inside the VNet is reliable and secure, so we skip timeouts and TLS between services." Respond.
**What the interviewer is really testing**
Pushback with substance; defence in depth.
**Typical mid-level answer**
"Add TLS anyway." Misses timeouts and the reasoning.
**Strong senior answer**
Reliability is not a property of "inside the VNet": the dependency can be overloaded, GC-paused, deploying, or half-dead, and TCP will happily wait forever (`HttpClient.Timeout` defaults to 100 s, far too long for most calls). Without timeouts one slow service exhausts threads/connections upstream. For security, flat-network trust means one compromised workload reaches everything; I'd use TLS plus identity (Managed Identity/Entra ID tokens, or mTLS via mesh) and private endpoints. Trade-off: some cert management overhead - mitigated by managed platforms. Decision: timeouts and authn on every hop; "internal" is not a trust boundary. Risk of not doing it: one slow dependency becomes an outage for every caller.
**Follow-up traps**
- "What is the default HttpClient timeout?" (100 seconds.)
- "Does TLS cost matter?" (Negligible with connection reuse.)

**Red flags in answers**
- Treating a timeout as proof the operation failed.
- Believing "inside the VNet" removes the need for timeouts/auth.
- Reciting fallacies without a design consequence.

**Cheat sheet**
- Remote call != local call: it can fail, hang, duplicate, or lose the reply.
- Timeout = unknown outcome -> need idempotency.
- Tail latency compounds with fan-out.
- Never hold DB transactions across remote calls.
- Don't hard-code topology; handle DNS changes (`PooledConnectionLifetime`).
- Every hop: timeout + auth + TLS.

---

## Topic 2: Failure Taxonomy (crash, omission, timing, partial, gray failure, byzantine-lite)

**Study notes**
- **Crash (fail-stop):** process dies, stays dead. Easiest to detect/handle.
- **Omission:** messages lost (send or receive). Looks like silence.
- **Timing:** response arrives but too late (GC pause, noisy neighbour, throttling).
- **Partial failure:** some operations/instances/shards work, others don't.
- **Gray failure:** component appears healthy to its health checks but is degraded for real users (slow disk, 5% errors, one bad AZ). Detection differs by observer.
- **Byzantine-lite:** wrong or corrupt answers rather than no answers (bad deploy returning stale data, clock skew, schema drift, bit flips), without full malicious Byzantine model.

```
Health probe --> "OK" (tiny /health path)
Real users  --> 30% timeouts (heavy path, bad disk)   <- gray failure
```
Crash is easy because everyone agrees it is down; gray failure is hard because observers disagree.

### 2.1 [Easy] Why is a crashed service usually easier to handle than a slow one?
**What the interviewer is really testing**
Understanding that slow = resource-holding and ambiguous.
**Typical mid-level answer**
"Slow is harder to detect." True but shallow.
**Strong senior answer**
A crash fails fast: connections reset, load balancer ejects the node, clients get immediate errors and can fail over. A slow dependency fails *slowly*: callers hold threads, sockets, and pooled connections while waiting, queues grow, and the slowness propagates upstream - latency becomes unavailability. Also, with slowness the caller cannot tell whether work is happening. So I prefer to convert slow into fast failure with timeouts, circuit breakers, and bulkheads. What fails when it fails: without those, one slow node degrades the whole caller fleet, which is the classic start of a cascading failure.
**Follow-up traps**
- "Would you rather have a service crash or hang?" (Crash - fail fast.)
- "Where does the load balancer's detection fall short?" (Only sees probe path.)

### 2.2 [Follow-up] Define gray failure and give an Azure example with how you'd detect it.
**What the interviewer is really testing**
Real operational experience; knowing health checks lie.
**Typical mid-level answer**
"When it is partially down." Doesn't mention observer disagreement or detection.
**Strong senior answer**
Gray failure: the system's own failure detector says healthy while clients experience degradation. Example: an App Service instance whose `/health` returns 200 quickly, but a node-local disk problem or a poisoned connection makes 20% of real requests take 10 s; or one SQL replica lagging. Detection: measure from the client's perspective (synthetic transactions, per-instance error rate and latency from App Insights, client-side metrics), compare instances (outlier detection: instance p99 vs fleet), and make health checks exercise a representative path (carefully - see Topic 10). Mitigation: outlier ejection, hedged requests for idempotent reads, restart/replace automation. Trade-off: more aggressive ejection risks removing capacity during a global slowdown, so cap ejection (e.g. max 30% of fleet).
**Follow-up traps**
- "Why cap ejection?" (Avoid removing all capacity.)
- "Does averaging hide it?" (Yes - use percentiles per instance.)

### 2.3 [Medium] Classify: a Service Bus message is processed, DB commit succeeds, then the worker crashes before completing the message. What failure is it and what happens?
**What the interviewer is really testing**
Mapping failure modes to at-least-once semantics.
**Typical mid-level answer**
"The message is lost" or "it fails." Wrong - it's redelivered.
**Strong senior answer**
It's a crash after side effect but before acknowledgement - effectively a lost ack (omission). With peek-lock, the lock expires and the message is redelivered (up to MaxDeliveryCount, default 10, then dead-letter), so the handler runs again and duplicates the effect unless it's idempotent. Options: make the handler idempotent via natural keys/upserts; record processed MessageId in the same DB transaction (inbox/dedupe table); or Service Bus duplicate detection (helps only for duplicate *sends*). Decision: inbox table with unique constraint on MessageId inside the business transaction. Risk: dedupe table growth - purge by retention aligned with maximum redelivery window.
**Follow-up traps**
- "Can we get exactly-once?" (Effectively-once via idempotent processing.)
- "Does Service Bus duplicate detection solve it?" (No - producer side.)

### 2.4 [Hard] Explain "byzantine-lite" failures with two examples you'd actually see, and what controls you'd add.
**What the interviewer is really testing**
Awareness that components can be up but wrong.
**Typical mid-level answer**
"Hackers sending bad data." Misses benign causes.
**Strong senior answer**
I use it for a non-malicious but wrong behaviour: a bad deploy returns 200 with empty or stale data; a clock-skewed node issues tokens that look expired; a schema change makes a consumer silently drop fields; a cache serves another tenant's data due to a key bug. Retries and failover don't help because the system reports success. Controls: input/output validation and schema contracts, checksums and invariants (e.g. order total equals sum of lines), canary releases comparing business metrics (not just 5xx), reconciliation jobs, idempotent replay ability, and synthetic transactions asserting content. Trade-off: validation costs CPU and engineering effort - spend it on money/security-critical paths. What fails when it fails: corruption spreads downstream and is much costlier than an outage.
**Follow-up traps**
- "Would circuit breakers catch it?" (Not unless you classify bad results as failures.)
- "What detects it?" (Business-level SLIs, reconciliation.)

### 2.5 [Scenario] One of 10 instances returns errors for 30% of requests but passes its probe. Walk through your response.
**What the interviewer is really testing**
Incident reasoning with partial failure.
**Typical mid-level answer**
"Restart it." May fix, but without diagnosis or prevention.
**Strong senior answer**
First mitigate: remove that instance from rotation (scale-in/replace or drain) - protects users quickly and preserves evidence by capturing a dump/logs first if feasible. Confirm it's instance-specific by splitting metrics by `cloud_RoleInstance`. Diagnose: dependency connection to a bad replica, a stuck connection pool, disk/CPU throttling, bad config. Prevent: make readiness reflect the dependency state the instance actually uses, add outlier detection/ejection at the load balancer or mesh, and alert on per-instance error-rate deviation. Trade-off: automated ejection can mask a real bug that later hits all instances, so alert on ejections. Risk: if all instances degrade together, ejection logic must not remove everything.
**Follow-up traps**
- "What if all instances show 30%?" (Not gray instance - shared dependency/code.)
- "Why capture evidence before restart?" (Root cause otherwise lost.)

**Red flags in answers**
- "Healthy = /health returns 200."
- Ignoring slow failures and only handling exceptions.
- Claiming exactly-once delivery as a platform guarantee.

**Cheat sheet**
- Crash < omission < timing < partial < gray < wrong-answer in difficulty.
- Slow is worse than down: it holds resources.
- Gray failure = observers disagree; measure from the client side.
- At-least-once + idempotent handler = effectively once.
- Success responses can still be wrong: validate and reconcile.
- Outlier ejection needs a cap.

---

## Topic 3: Timeouts Everywhere (budgets, deadline propagation, cancellation)

**Study notes**
Every I/O call needs a timeout; timeouts must nest: inner < outer. A request has an overall *deadline*; each downstream call receives the *remaining* budget.

```
Client (10s total)
  -> API  (budget 9s)
       -> Service A (attempt 2s x up to 2 tries)
       -> SQL (command timeout 3s)
  Sum of worst case must fit in the budget.
```
.NET tools: `CancellationToken` (pass everywhere, incl. `HttpContext.RequestAborted`), `CancellationTokenSource.CreateLinkedTokenSource` + `CancelAfter`, `HttpClient.Timeout` (default 100 s), `SqlCommand.CommandTimeout` (default 30 s), connection timeout (default 15 s), Polly `TimeoutStrategy` (v8 `AddTimeout`), `Microsoft.Extensions.Http.Resilience` standard handler (per-attempt timeout plus total request timeout).

### 3.1 [Easy] Why must every outbound call have an explicit timeout, and what's a bad default?
**What the interviewer is really testing**
Basic discipline.
**Typical mid-level answer**
"To avoid hanging." No sizing logic.
**Strong senior answer**
Without a timeout, a stuck dependency holds a thread/socket/pool slot indefinitely, so capacity drains until the caller is down too. Defaults are usually wrong: `HttpClient.Timeout` is 100 s, `SqlCommand.CommandTimeout` 30 s - longer than most user-facing SLAs. I'd derive the timeout from observed latency (e.g. slightly above dependency p99.9, or the percentile the business tolerates) and from the caller's budget, not guess. Too short causes false failures and retry load; too long causes resource exhaustion. What fails when this fails: the caller turns slowness into unavailability.
**Follow-up traps**
- "Is `HttpClient.Timeout` enough?" (Doesn't cover your overall deadline across retries.)
- "Is infinite ever okay?" (Long-poll/streaming with heartbeats only.)

### 3.2 [Follow-up] Explain timeout budgets and deadline propagation across three services.
**What the interviewer is really testing**
Systems-level thinking about nested timeouts.
**Typical mid-level answer**
"Each service sets 30 s." Ignores that inner work continues after the caller gave up.
**Strong senior answer**
Give the edge request a total budget, e.g. 5 s. Each hop calculates remaining = deadline - now, subtracts a safety margin, and uses it as its timeout; it passes the deadline downstream (custom header like `x-deadline`, or gRPC deadlines which propagate natively). Inner timeouts must be smaller than outer ones, otherwise the outer gives up while inner work continues, wasting capacity ("zombie work"). Retries must fit within the budget too: two 2 s attempts consume 4 s of 5. Decision: a total timeout at the edge, per-attempt timeouts below it, and a no-retry rule when remaining budget is too small. Risk: clock skew if using absolute timestamps - prefer relative remaining-time values.
**Follow-up traps**
- "Why relative not absolute deadlines?" (Clock skew.)
- "What does gRPC give you?" (Deadline propagation built in.)

### 3.3 [Medium] How do cancellation tokens fit in, and what breaks when they aren't passed end-to-end?
**What the interviewer is really testing**
.NET specifics: cooperative cancellation.
**Typical mid-level answer**
"Pass the token to async methods." Doesn't explain the consequences.
**Strong senior answer**
Timeouts only free resources if cancellation actually reaches the I/O: `HttpClient.SendAsync(req, ct)`, EF `ToListAsync(ct)`, `SqlCommand.ExecuteReaderAsync(ct)`. Start from `HttpContext.RequestAborted` so a disconnected client cancels work, link with a `CancelAfter` timeout for the deadline. If a method drops the token, the work continues after the caller gave up - threads, DB cursors and locks are held, and load persists during an overload, worsening it. Also catch `OperationCanceledException` separately from real failures: cancellation shouldn't trigger retries or error alerts. Trade-off: cancelling mid-write needs care - for non-idempotent commits, don't cancel after the point of no return; let the unit of work finish or be atomic. Risk: `Task.Run` + sync-over-async hides cancellation.
```csharp
using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct);
cts.CancelAfter(TimeSpan.FromSeconds(2));
var r = await _client.GetAsync(url, cts.Token);
```
**Follow-up traps**
- "Is SQL cancelled when token fires?" (Attention is sent; server rolls back.)
- "Should OperationCanceledException be retried?" (No.)

### 3.4 [Medium] Layered timeouts: HttpClient, Polly, SQL, load balancer, Front Door. How do you reconcile them?
**What the interviewer is really testing**
Seeing the whole chain; platform timeouts exist.
**Typical mid-level answer**
"Set them all to the same value." Wrong.
**Strong senior answer**
Order them outermost to innermost: gateway/Front Door and App Service front end have fixed limits (e.g. Azure Application Gateway request timeout is configurable; App Service front end idle timeout is about 230 s - so a long request will be cut regardless), then our app's total deadline, then per-attempt timeouts, then SQL command timeout, then connection timeout. Each inner value is strictly smaller than its outer, and with retries the sum of attempts plus backoff is below the outer. If SQL command timeout (30 s) exceeds the API deadline (5 s), the API gives up while SQL keeps working. Decision: define a timeout table in the design doc and enforce it in code with the standard resilience handler plus explicit command timeouts. Risk: platform limits change - verify against current docs.
**Follow-up traps**
- "What if long operations are legitimate?" (Async pattern: 202 + polling/queue.)
- "Who wins if outer is shorter?" (Outer; inner becomes zombie work.)

### 3.5 [Scenario] After tightening a timeout from 30 s to 2 s, error rate rose and the downstream got busier. Explain.
**What the interviewer is really testing**
Second-order effects of timeouts.
**Typical mid-level answer**
"Revert it." No analysis.
**Strong senior answer**
If the dependency's real p95 is above 2 s, we now fail many requests that would have succeeded; with retries those become extra requests, while the original attempts continue on the server (no cancellation), so load rises - a self-inflicted retry storm. Steps: check latency histogram of the dependency to place the timeout properly; verify cancellation reaches the dependency; cap retries and add jitter; consider the timeout as a per-attempt value with a total budget. If the dependency is genuinely slow, fix capacity or shift to async rather than loosening timeouts blindly. Trade-off: short timeouts protect callers but increase error rate during dependency slowness - that is intended; pair with fallbacks. Risk: reverting to 30 s hides the issue and risks pool exhaustion.
**Follow-up traps**
- "How do you choose the number?" (Percentile data + budget.)
- "Do server-side requests stop on client timeout?" (Only if cancellation propagates.)

**Red flags in answers**
- Relying on default timeouts.
- Inner timeouts longer than outer.
- Retries that ignore the overall deadline.

**Cheat sheet**
- Inner timeout < outer timeout; sum of attempts < budget.
- Propagate remaining budget, not a fixed value.
- Pass `CancellationToken` from `RequestAborted` to every I/O.
- Cancellation is not failure: don't retry or alert.
- Defaults: HttpClient 100 s, SqlCommand 30 s, connect 15 s.
- Long work -> async (202 + queue), not long timeouts.

---

## Topic 4: Retries Done Right

**Study notes**
Retry only: (1) transient errors, (2) idempotent operations (or with idempotency keys), (3) with exponential backoff + jitter, (4) bounded by attempts, total deadline, and a retry budget.
```
Layer amplification:  client x3 -> gateway x3 -> service x3 -> DB
                      worst case 3*3*3 = 27 DB attempts per user click
```
Retry only at ONE layer (usually closest to the failing call, or the edge) and pass "don't retry" signals down. Honour `Retry-After` on 429/503. Retry budget: allow retries up to e.g. 10% of request volume.
Polly v8: `ResiliencePipelineBuilder` / `AddRetry(new RetryStrategyOptions { BackoffType = DelayBackoffType.Exponential, UseJitter = true, MaxRetryAttempts = 3 })`. `AddStandardResilienceHandler()` includes retry, breaker, timeouts, rate limiter.

### 4.1 [Easy] When is it safe to retry a request?
**What the interviewer is really testing**
Idempotency as the prerequisite.
**Typical mid-level answer**
"When it fails with a transient error." Ignores whether repeating the operation is harmful.
**Strong senior answer**
Two conditions: the failure is plausibly transient (timeouts, 503, 429, connection reset, SQL 40613/1205) and the operation is idempotent. GET/PUT/DELETE are idempotent by HTTP semantics (if the server honours them); POST usually isn't, so I'd add an idempotency key that the server stores with the result and returns on replay. Important: a timeout may mean the first attempt succeeded, so non-idempotent retry risks double effects (double charge, duplicate order). Never retry 4xx (except 408/429), validation, or auth errors - they won't change. What fails when this fails: duplicates, or retry load against something that is hard-down.
**Follow-up traps**
- "Is PUT always safe?" (Only if implemented idempotently, e.g. no append behaviour.)
- "Retry a 500?" (Depends; often unknown cause - limited and careful.)

### 4.2 [Follow-up] Why exponential backoff *with jitter*, and what does the formula look like?
**What the interviewer is really testing**
Understanding synchronisation of clients.
**Typical mid-level answer**
"Wait longer each time." Misses jitter's purpose.
**Strong senior answer**
Exponential backoff spreads load over time and gives the dependency room to recover. But if 1,000 clients fail at the same moment, they all retry at 1 s, 2 s, 4 s - synchronised waves that hammer the dependency (thundering herd). Jitter randomises the delay (e.g. "full jitter": random(0, base * 2^n), capped) to de-correlate clients. Add a max delay cap and total deadline. Polly: `UseJitter = true` with `DelayBackoffType.Exponential`. Trade-off: jitter makes individual latency less predictable, fine for background work; for user-facing paths keep few attempts with small delays. Risk without it: periodic retry spikes that keep a recovering service down.
**Follow-up traps**
- "Is linear backoff enough?" (Still synchronised without jitter.)
- "Where do you cap?" (Max delay + total deadline.)

### 4.3 [Medium] Explain retry amplification across layers and how you prevent it.
**What the interviewer is really testing**
System view of retries.
**Typical mid-level answer**
"Don't retry too much." No mechanism.
**Strong senior answer**
If browser, API gateway, service A (calling B), and B (calling the DB) each retry 3 times, one user action can generate 3^4 = 81 DB attempts, exactly when the DB is weakest. Prevention: designate who retries - typically only the layer directly above the failing dependency, with outer layers failing fast; propagate a "retry attempt" header so downstream layers don't retry again; use retry budgets (retries limited to a percentage of base traffic) and token-bucket style limiting; respect deadlines. Also have services return `Retry-After` or a clear non-retryable status. Trade-off: fewer retry points means some transient failures reach users - acceptable versus meltdown. Risk: library defaults (SDKs retry internally: Azure SDK, EF, SqlClient) stack on top of your Polly policy - audit them.
**Follow-up traps**
- "Azure SDK clients already retry - add Polly too?" (Usually configure the SDK's own options instead.)
- "How do you measure amplification?" (Ratio of downstream to upstream request rate.)

### 4.4 [Hard] A retry storm: describe the mechanism and design the protections.
**What the interviewer is really testing**
Metastable failure understanding.
**Typical mid-level answer**
"Add backoff." Incomplete.
**Strong senior answer**
Mechanism: dependency slows -> callers time out -> callers retry -> offered load rises above capacity -> more timeouts - a positive feedback loop that can persist even after the trigger is gone (metastable failure). Protections in layers: (1) retry budget and capped attempts, (2) backoff with jitter, (3) circuit breaker to stop calls entirely when failure ratio is high, (4) server-side load shedding / rate limiting returning fast 429/503 with `Retry-After`, (5) timeouts with cancellation so abandoned work stops, (6) client-side concurrency limits (bulkhead). Recovery: shed or pause traffic (e.g. disable a feature flag, scale up, lower inbound admission) until the queue drains, then ramp. Trade-off: shedding intentionally rejects some valid requests to save the rest. Risk: a breaker that closes all at once creates a second herd - use half-open probing.
**Follow-up traps**
- "Why doesn't it recover on its own?" (Retries sustain the overload.)
- "What do you do first in an incident?" (Reduce load.)

### 4.5 [Tricky] "We'll make a POST idempotent by checking if the record exists first." Is that enough?
**What the interviewer is really testing**
Race conditions in idempotency.
**Typical mid-level answer**
"Yes, check then insert." Check-then-act race.
**Strong senior answer**
No: two concurrent retries both see "not exists" and both insert. Idempotency must be enforced atomically: a client-supplied idempotency key stored with a unique constraint (SQL unique index on key), insert first and handle the duplicate-key error (2627/2601) by returning the stored result. Also store the response (or status) so replays return the same outcome, and treat "in-progress" explicitly (return 409 or wait). Key TTL should exceed the client retry window. Trade-off: extra table and write per request. Risk: keys scoped incorrectly (global vs per-client) or reused for different payloads - hash the payload and reject mismatches.
```sql
INSERT INTO IdempotencyKeys(Key, RequestHash, Status) VALUES (@k, @h, 'InProgress');
-- unique violation => replay/in-progress path
```
**Follow-up traps**
- "How long to keep keys?" (Longer than max retry/redelivery window.)
- "Same key, different body?" (Reject, 422.)

### 4.6 [Scenario] Your service retries calls to a payment provider that is timing out. What should it do differently?
**What the interviewer is really testing**
Money-path judgement, unknown outcome.
**Typical mid-level answer**
"Retry three times." Risks double charging.
**Strong senior answer**
Clarify: does the provider support idempotency keys? If yes, retry with the same key, bounded by deadline. If not, a timeout leaves unknown state - don't blindly resubmit; instead query the transaction status by our reference, or mark "pending/unknown" and reconcile asynchronously. Put a circuit breaker on the provider and fail fast with a clear user message ("we're confirming your payment") rather than hanging. Use the outbox so the payment intent is durable. Trade-off: UX complexity and a reconciliation job versus risk of duplicate charges - correctness wins. Risk: reconciliation lag; alert on stuck-pending count.
**Follow-up traps**
- "What if the provider has no status API?" (Reconcile through settlement files.)
- "Should the user retry by clicking again?" (Same idempotency key per cart/attempt.)

**Red flags in answers**
- Retrying non-idempotent calls blindly.
- Fixed-interval retries without jitter.
- Retries at every layer.

**Cheat sheet**
- Retry = transient + idempotent + bounded.
- Backoff + jitter + max delay + total deadline.
- One retry layer; propagate no-retry signals.
- Retry budget (~10% of traffic) stops storms.
- Idempotency via unique constraint, not check-then-insert.
- Honour `Retry-After`; never retry 4xx except 408/429.

---

## Topic 5: Circuit Breakers

**Study notes**
```
 CLOSED --failures > threshold--> OPEN --break duration elapses--> HALF-OPEN
   ^                                |                                  |
   +------ probe succeeds ----------+------ probe fails --------------+
```
Closed: calls flow, failures measured (sliding window, failure ratio + minimum throughput). Open: calls fail immediately (`BrokenCircuitException` in Polly). Half-open: limited probes decide.
Scope per dependency (and often per endpoint/host/partition), not one global breaker. Polly v8 `AddCircuitBreaker(new CircuitBreakerStrategyOptions { FailureRatio, SamplingDuration, MinimumThroughput, BreakDuration })`. The standard handler has defaults you should review.

### 5.1 [Easy] What does a circuit breaker do and why use one?
**What the interviewer is really testing**
Purpose: fail fast, protect both sides.
**Typical mid-level answer**
"Stops calls when the service is down." Omits recovery and protecting the dependency.
**Strong senior answer**
It tracks failures of calls to a dependency and, past a threshold, short-circuits subsequent calls so they fail immediately instead of waiting for timeouts. Benefits: callers free resources quickly (no pile-up of threads/connections), the struggling dependency gets breathing room to recover, and we can trigger a fallback fast. After a break duration it goes half-open, letting a few trial calls through; success closes it, failure re-opens. What fails when this fails (i.e. the breaker itself mis-trips): healthy dependency gets blocked, so tune thresholds with minimum throughput and test.
**Follow-up traps**
- "Breaker vs retry?" (Retry handles blips; breaker handles sustained failure; breaker wraps retries.)
- "What do users see when open?" (Fallback or fast error.)

### 5.2 [Follow-up] How do you choose threshold, sampling window, minimum throughput and break duration?
**What the interviewer is really testing**
Tuning from data, not defaults.
**Typical mid-level answer**
"5 failures in a row." Brittle at low or high volume.
**Strong senior answer**
Use a failure *ratio* over a sampling window, requiring a minimum throughput so 1 failure out of 2 calls at night doesn't trip it. Starting values derive from the SLO and traffic: e.g. 50% failures over 30 s with at least 20 requests; break duration long enough for recovery but short enough to limit user impact (5-30 s), then half-open with 1-few probes. Count only relevant failures: timeouts, 5xx, 429 - not 4xx caused by bad client input. Validate with fault injection and load tests. Trade-off: sensitive = false trips and lost availability; lax = slow protection. Risk: consecutive-failure counters ignore volume; static values stale as traffic changes - review.
**Follow-up traps**
- "Should 404 count?" (No.)
- "Should slow calls count?" (Yes - slow-call ratio is supported by some libraries; in Polly, timeout exceptions count if handled.)

### 5.3 [Medium] Per-dependency scope: what goes wrong with a single shared breaker?
**What the interviewer is really testing**
Blast radius thinking.
**Typical mid-level answer**
"It's better to have separate ones." No reasons.
**Strong senior answer**
A shared breaker means a failure in one dependency (or one tenant/endpoint) opens the circuit for unrelated healthy calls - turning partial failure into total failure. Conversely too fine (per-URL with high cardinality) leaves insufficient samples to trip. I'd scope per logical dependency and, where failure domains differ, per endpoint class or per region/host (e.g. a breaker per `HttpClient` named client; per partition/tenant if noisy-neighbour risk). In .NET use named/typed clients each with their own pipeline instance; with Polly v8 each `ResiliencePipeline` instance holds its own state, so don't create one per request (state lost) and don't share one for everything. Risk: pipelines created per call never trip.
**Follow-up traps**
- "Why not create a policy per request?" (State is lost; never opens.)
- "Per-instance or shared across instances?" (Usually per-instance in-memory; fleet-wide trip is emergent.)

### 5.4 [Hard] When does a circuit breaker hurt?
**What the interviewer is really testing**
Honest trade-offs.
**Typical mid-level answer**
"It never hurts." Naive.
**Strong senior answer**
(1) False positives: a few bad requests (poison input) trip it and block healthy traffic, especially on a shared scope. (2) Flapping: short break + high probe rates re-overload a recovering service. (3) Low-traffic services: too little data. (4) Cascading semantic effects: an open breaker on a mandatory dependency means 100% errors for that feature - it's only useful if paired with a fallback or accepted fail-fast. (5) Hiding diagnoses: errors become `BrokenCircuitException`, so log the original cause and alert on state changes. (6) Where dependency is a database with connection pool, breaker may help, but a breaker on a single-instance monolith call adds little. Decision: use breakers on remote dependencies with meaningful traffic and a defined fallback; emit breaker state as a metric.
**Follow-up traps**
- "Breaker on the DB?" (Possible, but failover recovery timing must match the break duration.)
- "How do you test it?" (Inject faults, assert state transitions.)

### 5.5 [Tricky] Half-open lets "a few" calls through. In a 50-instance fleet, how many probes does the dependency really see, and does it matter?
**What the interviewer is really testing**
Fleet-level effects of per-instance state.
**Typical mid-level answer**
"One probe." Per breaker, yes, but forgets the fleet.
**Strong senior answer**
Breaker state is per process, so 50 instances each send a probe at roughly the same time after the same break duration - 50 probes, and if each then closes and ramps, the dependency sees a synchronised return of full traffic. Mitigations: randomise the break duration (jitter), limit concurrency of half-open probes, gradually ramp (concurrency limiter or rate limiter after close), and combine with server-side load shedding. Alternative is a shared/coordinated state (more complexity, extra dependency that itself can fail) - rarely worth it. Trade-off: slower recovery for safety. What fails when it fails: a premature simultaneous close re-kills the dependency and creates a flapping loop.
**Follow-up traps**
- "Shared breaker in Redis?" (Adds a dependency; usually avoid.)
- "How to ramp?" (Gradual concurrency/rate increase.)

### 5.6 [Scenario] The breaker to the recommendations service opened and the product page now returns 500. What should have been designed?
**What the interviewer is really testing**
Pairing breaker with fallback and criticality classification.
**Typical mid-level answer**
"Catch the exception." Without a thought on what to show.
**Strong senior answer**
Classify dependencies: critical (cart, pricing) vs optional (recommendations). For optional ones, an open breaker should map to a fallback: cached/stale recommendations, a generic "popular items" list, or hide the widget. Handle `BrokenCircuitException` and `TimeoutRejectedException` at the composition layer, return a partial page, and log/metric the degradation. Trade-off: serving stale or generic content versus failing; business agrees in advance. Test with a fault-injection test that opens the breaker and asserts 200 with degraded content. Risk: fallback also depends on the broken service (e.g. same cache) - check that it's independent.
**Follow-up traps**
- "What if fallback is also remote?" (Then it needs its own protection.)
- "Who decides what's optional?" (Product + engineering, documented.)

**Red flags in answers**
- One global breaker for everything.
- Breaker with no fallback or alerting.
- Creating a new policy instance per request.

**Cheat sheet**
- States: closed -> open -> half-open -> closed.
- Use failure ratio + minimum throughput + sampling window.
- Scope per dependency; reuse pipeline instances.
- Jitter break duration across the fleet; ramp after close.
- Breaker without fallback = fast failure only.
- Emit breaker state as metric/alert.

---

## Topic 6: Bulkheads & Isolation

**Study notes**
Bulkhead: partition resources so one failing consumer/dependency can't drain shared capacity (ship compartments).
```
Shared pool:   [ A A A A B B B B ]   slow A takes everything
Bulkheaded:    [ A A A A ] [ B B B B ]
```
Levels: concurrency limiters per dependency (Polly v8 `AddConcurrencyLimiter`), separate `HttpClient`s/handlers, separate SQL connection pools (different connection strings / `Application Name`), separate queues/consumers per workload or tenant, separate App Service plans / AKS node pools / deployment units, separate subscriptions for quota. .NET thread pool is shared per process - blocking calls starve it.

### 6.1 [Easy] What is the bulkhead pattern and what problem does it solve?
**What the interviewer is really testing**
Isolation concept.
**Typical mid-level answer**
"Separate thread pools." Only one flavour.
**Strong senior answer**
It limits the share of a shared resource any one workload or dependency can consume so failure stays contained. Example: calls to a slow reporting service capped at 20 concurrent requests; excess is rejected immediately (`RateLimiterRejectedException`) or queued briefly, so the 200 threads/connections for checkout remain available. Same idea applies to queues (separate queue per priority/tenant), connection pools, and deployments. Cost: lower utilisation (reserved capacity unused), more config. What fails when it fails: without isolation, one slow dependency uses all shared capacity and everything - even unrelated endpoints - times out.
**Follow-up traps**
- "Bulkhead vs rate limiter?" (Bulkhead limits concurrency; rate limiter limits rate.)
- "Is queuing in a bulkhead good?" (Only a short, bounded queue.)

### 6.2 [Follow-up] How does .NET make bulkheading hard, and what do you do about it?
**What the interviewer is really testing**
Practical .NET internals: thread pool, async, pools.
**Typical mid-level answer**
"Use Task.Run for separate work." That moves work onto the same pool.
**Strong senior answer**
With async/await, in-flight calls don't hold threads, so the scarce resources are usually connections (HttpClient sockets, SQL pool, default `Max Pool Size` 100) and memory - bulkhead those with concurrency limits per dependency. Sync-over-async (`.Result`) or blocking calls hold thread pool threads; the pool injects threads slowly, causing starvation that hits every endpoint in the process. Options: remove blocking, use concurrency limiters, or physically separate (different process/App Service plan/worker role) when the workload is hostile. Decision: logical bulkheads for most cases; physical separation for batch/report or untrusted tenant workloads. Risk: limits set too low throttle healthy traffic - base on Little's Law (concurrency = rate x latency).
**Follow-up traps**
- "How compute the limit?" (rate x p99 latency plus headroom.)
- "Symptoms of thread pool starvation?" (Rising latency with low CPU.)

### 6.3 [Medium] Design isolation for a multi-purpose API where report exports are heavy and checkout is critical.
**What the interviewer is really testing**
Choosing the isolation level.
**Typical mid-level answer**
"Scale up." Doesn't isolate.
**Strong senior answer**
Options: (a) per-endpoint concurrency limiter in the same process - cheap, but shares CPU/memory/GC; (b) separate deployment (own App Service plan or AKS deployment/node pool) for exports behind its own route - strong isolation; (c) move exports to async jobs: enqueue to a Service Bus queue, workers with bounded concurrency, results by link. For checkout being critical, I'd choose (c) or (b), plus separate DB resources (read replica or a separate connection pool with lower `Max Pool Size`, resource governance/Resource Governor on SQL). Trade-off: more infra and cost versus protecting revenue paths. What fails when it fails: a report spike no longer slows checkout; reports queue and are delayed instead, which is acceptable.
**Follow-up traps**
- "Same database?" (Still shared - use replicas/governance.)
- "How do you prove it works?" (Load test report spike while measuring checkout.)

### 6.4 [Hard] Can bulkheads make things worse?
**What the interviewer is really testing**
Trade-offs and utilisation.
**Typical mid-level answer**
"No, they only protect." Overlooks costs.
**Strong senior answer**
Yes: static partitions waste capacity (idle in one compartment while another is exhausted), too-small limits cause self-inflicted rejection under legitimate bursts, and many partitions increase operational complexity and cold caches. Dynamic approaches (adaptive concurrency limits, weighted fair queuing with a guaranteed minimum and burst up to a ceiling) balance it. Also bulkheads inside one process do not isolate from process-level failure (OOM, crash loop), so critical vs non-critical may require separate processes. Decision: start with coarse partitions (critical vs non-critical), measure, then refine. Risk: misconfigured rejection leads to errors users see; make rejections observable.
**Follow-up traps**
- "Static vs adaptive limit?" (Adaptive tracks latency; more complex.)
- "Which signal shows a bulkhead is rejecting?" (Rejection counter metric.)

### 6.5 [Scenario] One tenant's bulk import saturates the Service Bus consumer and other tenants' messages are delayed. Fix it.
**What the interviewer is really testing**
Queue isolation and fairness.
**Typical mid-level answer**
"Add more consumers." Doesn't fix fairness.
**Strong senior answer**
Adding consumers just moves the bottleneck to the DB or downstream. Better: separate queues (or topics/subscriptions with filters) for bulk vs interactive traffic with dedicated consumers and independent concurrency (`MaxConcurrentCalls`), so bulk can fall behind without delaying interactive. For many tenants, per-tenant queues don't scale; use a shared queue with per-tenant throttling at the producer (admission control) and per-tenant concurrency limits in the consumer, or sessions to serialise per tenant. Trade-off: more entities and monitoring versus fairness. What fails when it fails: noisy tenant causes SLA breaches for others. Risk: dedicated bulk consumers still hit the shared DB - cap their DB concurrency.
**Follow-up traps**
- "Priority queue support?" (Service Bus has no native priorities; use separate queues.)
- "Sessions downside?" (Per-session ordering reduces parallelism.)

**Red flags in answers**
- "Just add more threads/instances."
- Using `Task.Run` as a "bulkhead".
- No plan for what rejected requests experience.

**Cheat sheet**
- Isolate by dependency, workload, tenant, criticality.
- Levels: concurrency limit < separate pools < separate deployments < separate subscriptions.
- Limit = rate x latency + headroom (Little's Law).
- Blocking calls starve the shared thread pool.
- Separate queues for bulk vs interactive.
- Make rejections visible and meaningful (fast 429/503).

---

## Topic 7: Backpressure, Load Shedding, Queue-Based Load Leveling, Rate Limiting & Admission Control

**Study notes**
```
Producers --> [ bounded queue ] --> Consumers
                 |  full?
                 +--> reject (429/503) | block producer | drop low-priority
```
- **Backpressure:** signal upstream to slow down (bounded queues, `Channel.CreateBounded`, 429 + `Retry-After`, TCP window, consumer prefetch limits).
- **Load shedding:** deliberately drop work to keep the rest healthy (prefer dropping low priority, oldest/expired, or expensive requests).
- **Queue-based load leveling:** buffer bursts in a durable queue (Service Bus/Storage Queue) so consumers process at steady rate.
- **Rate limiting:** cap per client/tenant/IP; ASP.NET Core `AddRateLimiter` (fixed window, sliding window, token bucket, concurrency). Azure API Management `rate-limit-by-key`.
- **Admission control:** decide at the edge whether to accept based on current capacity.
Unbounded queues convert overload into latency and memory growth, then failure.

### 7.1 [Easy] Why is an unbounded queue dangerous?
**What the interviewer is really testing**
Little's Law intuition.
**Typical mid-level answer**
"It can run out of memory." Only one aspect.
**Strong senior answer**
When arrival rate exceeds service rate, an unbounded queue grows without limit: memory exhaustion in-process, or in a durable queue, latency grows until responses are useless (the user already gave up) yet we still burn capacity on them. A bounded queue forces a decision: block, reject, or drop. In .NET: `Channel.CreateBounded<T>(new BoundedChannelOptions(n){ FullMode = BoundedChannelFullMode.Wait })` or reject. For durable queues, set TTL (`TimeToLive`) so stale work expires and monitor queue age, not just depth. What fails when it fails: latency SLO breached silently while everything "looks up".
**Follow-up traps**
- "Depth or age?" (Age of oldest message is the SLI.)
- "Does TTL lose data?" (Yes - only for work that's worthless late.)

### 7.2 [Follow-up] Compare backpressure, load shedding, and rate limiting. When do you use each?
**What the interviewer is really testing**
Distinguishing protective mechanisms.
**Typical mid-level answer**
"They all limit traffic." No distinction.
**Strong senior answer**
Rate limiting enforces a fairness/contract per client regardless of system health (e.g. 100 req/min per API key) - protects against abusive or buggy callers. Load shedding reacts to *system* overload (CPU, queue age, concurrency) and drops work, ideally prioritising critical traffic. Backpressure pushes the signal back so producers slow (bounded queues, 429 + Retry-After, flow control) - best when the producer can adapt. Use rate limits at the edge/per tenant, backpressure between internal components, shedding as last-resort self-protection. Trade-off: shedding sacrifices some requests to preserve the system; do it early and cheaply (reject before doing expensive work), return fast with `Retry-After`. Risk: clients that ignore 429 and retry hot; so pair with client-side backoff.
**Follow-up traps**
- "What status code for shed?" (503 or 429 with Retry-After.)
- "Where do you shed?" (As early and cheap as possible.)

### 7.3 [Medium] Explain queue-based load leveling and where it breaks.
**What the interviewer is really testing**
Trade-offs of asynchronous decoupling.
**Typical mid-level answer**
"Put a queue in front." Ignores the limits.
**Strong senior answer**
A queue decouples bursty producers from steady consumers, so a spike costs queue time instead of failures, and consumers are sized for average rather than peak load. It works when work can be async and latency-tolerant. It breaks when: the caller needs a synchronous answer; consumers can never catch up (sustained arrival > capacity - queue just hides it, so alert on age/growth); work expires; ordering or idempotency is needed; downstream (DB) limits cap consumer scaling. Design points: bounded concurrency per consumer, TTL, dead-letter, autoscale on queue length (KEDA), 202 Accepted + status endpoint for clients. What fails when it fails: if the queue is down, producers need a fallback (outbox/local buffer) or must reject.
**Follow-up traps**
- "Is the queue a SPOF?" (Zone-redundant namespaces; geo-DR for Service Bus is metadata replication - check docs.)
- "Sustained overload?" (Need capacity or shedding.)

### 7.4 [Hard] Design admission control for a multi-tenant API so one tenant can't starve others, and critical calls survive overload.
**What the interviewer is really testing**
Layered fairness and priority.
**Typical mid-level answer**
"Rate limit by IP." Misses tenant and priority.
**Strong senior answer**
Layers: (1) Edge (Front Door/WAF/APIM): per-key rate limits and quotas by tenant plan, returning 429 with Retry-After. (2) In-app: ASP.NET Core rate limiter partitioned by tenant (`PartitionedRateLimiter`) plus a concurrency limiter, so each tenant gets a guaranteed share and bursts are bounded. (3) Priority: classify requests (health, checkout, auth > search > exports) and shed lowest first when a load signal (CPU, queue wait, in-flight count) crosses a threshold. (4) Protect shared dependencies with per-tenant DB concurrency limits. Trade-off: fixed per-tenant caps waste idle capacity; weighted fair sharing with burst allowance is better but more complex. Risk: in-memory limiters are per-instance, so fleet-wide limits are approximate unless backed by shared state (Redis) - which is itself a dependency; usually accept approximation with per-instance limit = global/instances.
**Follow-up traps**
- "Distributed rate limiting trade-off?" (Accuracy vs extra dependency.)
- "How to identify tenant before auth cost?" (API key/header at the edge.)

### 7.5 [Tricky] Your autoscaler adds instances during a spike but the error rate worsens. Why might shedding be better than scaling?
**What the interviewer is really testing**
Scale doesn't fix bottleneck downstream.
**Typical mid-level answer**
"Scale takes time." Partial.
**Strong senior answer**
Two reasons: scale lag (new instances take minutes; the spike or the failure is now) and downstream limits: more instances open more DB connections and send more requests to a dependency that is the true bottleneck, making things worse (also cold instances start with empty caches and JIT warm-up). Shedding or rate limiting caps what reaches the bottleneck immediately. So: identify the constraining resource (DB DTUs/CPU, connection limit, third-party quota), cap concurrency into it, then scale only what is truly scalable. Trade-off: rejecting some traffic versus letting everything degrade. Risk: shedding triggered too late; use leading indicators (queue wait time, concurrent requests) rather than CPU alone.
**Follow-up traps**
- "Which metric triggers shedding?" (Latency/queue wait/in-flight, not just CPU.)
- "Max connections math?" (instances x pool size must fit the DB limit.)

### 7.6 [Scenario] A flash sale will send 20x normal traffic to checkout. Plan the protections.
**What the interviewer is really testing**
Practical capacity events.
**Typical mid-level answer**
"Scale out in advance." One tool only.
**Strong senior answer**
Clarify: peak rate, duration, what's the bottleneck (inventory DB row contention, payment provider limits). Plan: pre-scale (scheduled) plus load test to the target with realistic data; a virtual waiting room / queue at the edge to admit users at a rate the backend sustains; cache catalogue/read paths, with static fallback pages; reserve capacity for checkout via bulkheads, and shed browsing/recommendations first; make inventory reservation conflict-tolerant (optimistic concurrency, partitioned counters); confirm third-party quotas in advance. Runbook: kill switches for non-essential features, on-call, rollback criteria. Trade-off: waiting room frustrates some users but beats a crash. Risk: load testing against prod-like data stores and 3rd-party sandboxes not matching limits.
**Follow-up traps**
- "Hot row contention?" (Shard counters / queue serialisation.)
- "What do you turn off first?" (Non-critical features via flags.)

**Red flags in answers**
- Unbounded queues/channels.
- "Autoscale will handle it."
- Shedding after the expensive work is already done.

**Cheat sheet**
- Bounded queues + TTL; monitor oldest-message age.
- Rate limit per tenant; shed per system load; backpressure internally.
- Reject early, cheaply, with `Retry-After`.
- Queue leveling needs sustained capacity >= average load.
- Check downstream limits before scaling out.
- ASP.NET Core `AddRateLimiter` + `PartitionedRateLimiter` for tenant fairness.

---

## Topic 8: Graceful Degradation, Fallbacks & Static Stability

**Study notes**
Degradation ladder: full feature -> cached/stale -> default/generic -> feature hidden -> clear error. Classify dependencies **critical vs optional**.
Fallback options: stale cache (serve last good value with staleness metadata), default response, cached snapshot of reference data, alternate provider, queue the request for later, read-only mode.
**Static stability (AWS term, broadly applicable):** the system keeps working when the control plane/dependency is unavailable, because the data plane doesn't need to make changes. E.g. pre-provisioned capacity across zones instead of "scale up when a zone fails"; config/feature flags cached locally; tokens/JWKS cached so auth survives identity provider blips.
```
Request -> try primary (timeout) -> fallback 1 (stale cache) -> fallback 2 (default) -> degrade
```

### 8.1 [Easy] What is graceful degradation and how do you decide what to degrade?
**What the interviewer is really testing**
Product-aware engineering.
**Typical mid-level answer**
"Show an error message nicely." Not degradation.
**Strong senior answer**
Degradation means continuing to provide the core value with reduced functionality when something fails. First classify features by business criticality with product: checkout, login are core; recommendations, reviews, personalisation are optional. For each optional dependency define the fallback (stale cache, default list, hide) and the user-visible message. Implement via timeouts + breaker + fallback in a composition layer, with metrics that tell us we're degraded (so degradation isn't invisible). Trade-off: stale or generic data versus errors; some data (prices, inventory, balances) must never be stale without disclosure. What fails when it fails: an unclassified optional dependency takes down a critical page.
**Follow-up traps**
- "Can you serve stale prices?" (Only if the business accepts and the final price is revalidated at checkout.)
- "How do you test?" (Fault injection per dependency.)

### 8.2 [Follow-up] Stale-while-revalidate / serve-stale-on-error: how would you implement it in .NET, and what are the risks?
**What the interviewer is really testing**
Practical caching resilience design.
**Typical mid-level answer**
"Cache it with a TTL." No stale-on-error.
**Strong senior answer**
Keep two lifetimes: soft TTL (refresh after) and hard TTL (maximum staleness). On soft expiry, return the cached value and refresh in the background (single-flight) - or on refresh failure, keep serving until hard TTL. Libraries: FusionCache supports fail-safe and soft/hard timeouts; HybridCache (.NET 9) gives stampede protection and L1/L2; or hand-roll with `IMemoryCache` + `SemaphoreSlim`/`Lazy<Task>`. Store a `FetchedAt` so responses can expose staleness. Risks: serving wrong data beyond business tolerance, caching error responses (don't cache failures long), unbounded memory, and masking a permanent breakage - alert when serving stale beyond a threshold. Trade-off: freshness vs availability, chosen per data type.
**Follow-up traps**
- "What if first load fails (cold cache)?" (No stale; need default/fallback.)
- "Cache negative results?" (Short TTL only.)

### 8.3 [Medium] Explain static stability with a concrete Azure example.
**What the interviewer is really testing**
Control plane vs data plane reasoning.
**Typical mid-level answer**
"Stays stable." Circular.
**Strong senior answer**
A statically stable system keeps operating through a dependency failure without needing to take new actions at failure time. Example: running N+1/N+2 instances spread across availability zones in advance, so losing a zone needs no scale-out call (which might be unavailable or throttled during the incident), versus relying on autoscaling to rebuild capacity. Another: cache Entra ID signing keys/JWKS and app configuration locally, so token validation and settings work if the metadata endpoint is down; start-up with last-known-good config snapshot. Another: Front Door/Traffic Manager failover relies on health probes (data plane) not on someone running a script. Trade-off: paying for idle headroom (over-provision ~50% across 3 zones to survive losing one) versus resilience. What fails when it fails: recovery depends on control-plane operations which are exactly what is flaky during large incidents.
**Follow-up traps**
- "Why 3 zones with 50%?" (Lose one zone, remaining two carry 100%.)
- "Startup dependency on config service?" (Use cached fallback.)

### 8.4 [Hard] A feature depends on three optional services. Product wants it "always available". How do you design and what are you not allowed to promise?
**What the interviewer is really testing**
Availability composition and honest commitments.
**Typical mid-level answer**
"Add fallbacks for all." Doesn't consider combined behaviour.
**Strong senior answer**
Each optional dependency gets timeout + breaker + fallback so the feature remains available even when all three are down - at degraded quality. Parallel calls (not serial) with a collective deadline so total latency is the slowest bounded timeout, not the sum. I'd define quality tiers and expose which tier served each response in telemetry. What I won't promise: full-fidelity results at 100% availability; fallbacks themselves must be independent (not the same Redis or same region as the failed primary). Trade-off: engineering cost of N fallback paths, each of which must be tested - untested fallbacks fail when needed. Risk: fallback code paths rarely exercised; schedule game days that force them.
**Follow-up traps**
- "Serial vs parallel calls?" (Parallel with budget.)
- "Fallback depends on primary's cache?" (Shared fate - not a fallback.)

### 8.5 [Scenario] The identity provider is down. Users already logged in should keep working; new logins fail. How do you get there?
**What the interviewer is really testing**
Design choices for auth dependencies.
**Typical mid-level answer**
"Call the provider on each request." That fails entirely.
**Strong senior answer**
Validate JWTs locally using cached signing keys (ASP.NET Core JWT bearer caches metadata and refreshes; configure refresh intervals and keep last-known-good keys). Use session tokens/refresh tokens with adequate lifetimes so existing sessions survive the outage window; avoid per-request introspection calls. New logins fail with a clear message - optionally allow a read-only experience for existing sessions. Trade-off: longer token lifetimes widen the window for revoked-token abuse; use short access tokens + refresh, accept some staleness in revocation for the outage. Risk: key rotation during the outage - cached keys eventually mismatch; monitor metadata fetch failures and alert early.
**Follow-up traps**
- "Revocation during outage?" (Accept bounded staleness; document.)
- "Does Entra have regional redundancy?" (Platform-level; still design for failure.)

**Red flags in answers**
- Fallback that depends on the same failing component.
- Untested fallback paths.
- Serving stale data with no disclosure/alert.

**Cheat sheet**
- Classify dependencies: critical vs optional.
- Ladder: fresh -> stale -> default -> hide -> error.
- Soft TTL + hard TTL; single-flight refresh.
- Static stability: pre-provision, cache config/keys, avoid control-plane at failure time.
- Make degradation visible (metric, header, banner).
- Test fallbacks with injected faults.

---

## Topic 9: Cascading Failure, Thundering Herd, Cache Stampede & Resilient Caching

**Study notes**
Cascading failure: overload/failure in one component shifts load to others, which then fail. Triggers: lost capacity (instance/zone down -> survivors overloaded), retries, cache failure, GC/CPU saturation, health check removal of overloaded nodes.
```
Cache down --> 100% traffic to DB --> DB CPU 100% --> timeouts --> retries --> more DB load
```
Thundering herd: many clients act at once (retry sync, cache expiry, restart). Cache stampede: many requests miss the same key and hit the origin simultaneously.
Mitigations: request coalescing/single-flight, jittered TTLs, early refresh (probabilistic), stale-while-revalidate, L1 + L2 cache, cache warmup, rate limits to origin, DB concurrency caps, breaker around cache too.

### 9.1 [Easy] What is a cache stampede and how do you prevent it?
**What the interviewer is really testing**
Core cache hazard.
**Typical mid-level answer**
"Use a bigger cache." Not related.
**Strong senior answer**
When a hot key expires, hundreds of concurrent requests all miss and all recompute from the database - a sudden spike on the origin, potentially a cascade. Prevention: single-flight (only one request recomputes while others wait or get stale), e.g. `Lazy<Task<T>>`/`SemaphoreSlim` per key in memory, HybridCache's stampede protection (.NET 9), or a distributed lock/lease for L2 (with a short timeout - the lock holder can die). Add TTL jitter so many keys don't expire together, and refresh before expiry (soft TTL). Trade-off: waiters add latency; a stuck refresher blocks them, so bound the wait and fall back to stale. What fails when it fails: DB meltdown at moment of expiry or after a cache flush.
**Follow-up traps**
- "Distributed lock risks?" (Holder dies, lock timeouts, clock issues - use leases + TTL.)
- "Does in-memory single-flight work across instances?" (Per instance only; 50 instances = 50 refreshes - usually acceptable.)

### 9.2 [Follow-up] Redis is down. Explain what happens to your system and how you design so it doesn't melt the database.
**What the interviewer is really testing**
Cache is a dependency with its own failure mode.
**Typical mid-level answer**
"Fall back to the DB." That is the meltdown.
**Strong senior answer**
If the cache is an optimisation but the DB sized for the miss-rate (e.g. 95% hit), then losing it multiplies DB load 20x. Designs: (1) timeouts and a breaker around the cache so a hanging Redis doesn't slow every request (StackExchange.Redis `connectTimeout`/`syncTimeout`/`abortConnect=false`, reconnection handled); (2) on cache failure, go to the DB but through a concurrency limiter/bulkhead and single-flight so the DB sees bounded load; (3) L1 in-process cache (short TTL) in front of Redis so hot keys survive; (4) shed or degrade non-critical reads when over the limit; (5) capacity-plan the DB for a defined fraction of cold-load or accept degradation; (6) Redis HA (zone redundant, replicas, persistence/geo as needed). Trade-off: cost of sizing for cache-off vs degraded UX. Risk: restart of Redis returns an empty cache, a second stampede - warm gradually.
**Follow-up traps**
- "Fail open or fail closed?" (Per data: reads fail open with limits; rate limiters/locks need explicit choice.)
- "Does cold start after Redis restore matter?" (Yes - gradual warming.)

### 9.3 [Medium] Describe a cascading failure from a lost availability zone and how you'd have prevented it.
**What the interviewer is really testing**
Capacity planning under N-1.
**Typical mid-level answer**
"Autoscale." Too slow and uncertain.
**Strong senior answer**
Three zones each at 70% utilisation; one is lost, survivors must take 105% - they saturate, latency climbs, health checks fail, more instances get ejected, and load concentrates on fewer - cascade. Prevention: N-1 capacity (keep utilisation below ~60% with 3 zones, or provision accordingly), static stability (pre-provisioned), load shedding to protect survivors, health checks that don't eject on overload alone (or ejection caps), retry budgets, breaker/backoff, and connection pool limits to the DB. Trade-off: idle capacity cost. Also test with a zone-failure exercise. Risk: autoscaling reacting to CPU triggers late and cold instances worsen the load.
**Follow-up traps**
- "How much headroom with 2 zones?" (50% max util to survive losing one.)
- "Should the load balancer eject all unhealthy?" (Use panic/min-healthy thresholds.)

### 9.4 [Hard] After a deploy, the whole fleet restarted and the DB fell over though traffic was normal. Diagnose.
**What the interviewer is really testing**
Cold start herd.
**Typical mid-level answer**
"Deploy bug." Skips the mechanism.
**Strong senior answer**
Likely a cold-start herd: all instances simultaneously start with empty caches, open connection pools (e.g. each opens up to 100 connections) and warm queries, so the DB sees a surge of connections and cache-miss queries; plus Startup health checks passing early and the LB sending full traffic. Mitigations: rolling deploys with surge/maxUnavailable limits so only a fraction is cold, readiness gates that wait for warmup (cache primed, pools opened), slow-start at the LB, bounded `Max Pool Size`/`Min Pool Size` sized for fleet total, L2 cache survives restarts, jittered startup. Verify via DB connection counts and cache hit ratio at deploy time. Trade-off: slower deploys. Risk: instances x pool size beyond DB limit (e.g. 100 x 30 = 3000 connections).
**Follow-up traps**
- "Why not deploy all at once for speed?" (Herd + no rollback margin.)
- "Pre-warm?" (Warm-up requests/readiness delay.)

### 9.5 [Scenario] A cache keyed by `userId` returned another user's data after a refactor. What do you change beyond fixing the key?
**What the interviewer is really testing**
Byzantine-lite caching risk; defence in depth.
**Typical mid-level answer**
"Fix the key." Only a patch.
**Strong senior answer**
Incident first: purge affected cache entries, determine exposure window, notify per policy. Root cause: key built without tenant/user context (or lacking vary-by in output caching, `Cache-Control: public` on authenticated responses). Preventive: central cache-key builder that requires tenant+user for personalised data, tests that assert isolation, never use shared output caching for authorised content without `VaryBy`, include a schema/version in keys, store owner id in the cached value and verify on read, and alerts for anomalies. Trade-off: slightly more code per cache use. Risk: CDN or reverse-proxy caching on authenticated routes - set `Cache-Control: private/no-store`.
**Follow-up traps**
- "Is the cache a security boundary?" (No; verify ownership on read.)
- "Where else could this happen?" (CDN, output cache, HTTP caches.)

**Red flags in answers**
- "If Redis is down, just read the DB."
- No headroom for losing a zone.
- Distributed locks without lease/timeout.

**Cheat sheet**
- Cascades start with lost capacity + feedback (retries, ejection).
- Single-flight + TTL jitter + soft TTL fight stampedes.
- Cache = dependency: timeouts, breaker, bounded DB fallback.
- N-1 capacity: 3 zones <= ~66% util; 2 zones <= 50%.
- Roll deploys gradually; warm before ready.
- instances x pool size <= DB connection limit.

---

## Topic 10: Health Checks, Graceful Shutdown & Draining

**Study notes**
- **Startup:** has the app finished initialising? (Kubernetes `startupProbe`) - suppresses liveness until ready.
- **Liveness:** is the process stuck/dead and needs a restart? Should be cheap and NOT depend on external dependencies.
- **Readiness:** can this instance take traffic now? May include critical dependencies, with care.
ASP.NET Core: `AddHealthChecks()`, `MapHealthChecks("/health/live", new() { Predicate = r => r.Tags.Contains("live") })`, separate tags for ready.
```
Liveness fails  -> restart container
Readiness fails -> remove from load balancer (no restart)
Startup pending -> don't run liveness yet
```
Shutdown (K8s): SIGTERM -> readiness fails/endpoint removed -> drain in-flight -> exit; `terminationGracePeriodSeconds` default 30 s; .NET `HostOptions.ShutdownTimeout` default 30 s (in recent versions). Endpoint removal propagates asynchronously, so add a short pre-stop delay.

### 10.1 [Easy] Liveness vs readiness vs startup: what's the difference and what should each check?
**What the interviewer is really testing**
Correct semantics and consequence of each failing.
**Typical mid-level answer**
"One /health endpoint that checks everything." Conflates them.
**Strong senior answer**
Liveness answers "should I kill and restart this process?" - check only in-process health (event loop responsive, not deadlocked). Readiness answers "should it get traffic?" - check it's initialised and can serve, including truly critical local dependencies. Startup protects slow initialisation from being killed by liveness. Consequences differ: failed liveness = restart (disruptive), failed readiness = just removed from rotation. What fails when this fails: a liveness check that calls the database restarts every instance when the DB blips, turning a dependency issue into a full outage.
**Follow-up traps**
- "Should liveness check the DB?" (No.)
- "What does the startup probe do?" (Delays liveness/readiness until started.)

### 10.2 [Follow-up] Should readiness check downstream dependencies? Explain "deep health" pitfalls.
**What the interviewer is really testing**
Nuance: shared dependency failure ejects all.
**Typical mid-level answer**
"Yes, check DB, Redis, everything." Dangerous.
**Strong senior answer**
If every instance's readiness depends on a shared dependency, when that dependency blips *all* instances go unready simultaneously - load balancer has none left, so a partial degradation becomes total outage, and the unready instances can't even serve the requests that don't need that dependency. Also deep checks are expensive (probing every few seconds across a fleet adds load to the dependency) and can themselves hang. Guidance: readiness checks local readiness plus only dependencies without which the instance is useless, with short timeouts, cached results (e.g. evaluate every 10-30 s), and thresholds; keep a separate deep/diagnostic endpoint for dashboards and operators, not for orchestrators. Prefer circuit breakers and fallbacks over ejection. Trade-off: a "ready" instance may serve errors for one dependency - covered by degradation. Also protect health endpoints from the public internet.
**Follow-up traps**
- "All instances unready - what does the LB do?" (Some LBs go 'panic' routing to all; check your platform.)
- "Cost of deep probe?" (Fleet x frequency load.)

### 10.3 [Medium] Walk through a zero-error graceful shutdown of an ASP.NET Core API in Kubernetes or App Service.
**What the interviewer is really testing**
Draining mechanics.
**Typical mid-level answer**
"Handle SIGTERM." Skips endpoint propagation and in-flight work.
**Strong senior answer**
Sequence: (1) SIGTERM arrives; (2) immediately flip readiness to failing so traffic stops (Kubernetes removes the pod from endpoints, but propagation to kube-proxy/ingress is asynchronous, so a `preStop` sleep of a few seconds avoids requests arriving after SIGTERM); (3) Kestrel stops accepting new connections and waits for in-flight requests (host shutdown timeout; align with `terminationGracePeriodSeconds`, which must exceed preStop + drain + margin); (4) background services stop via their `CancellationToken` and finish or hand back the unit of work (e.g. abandon Service Bus message so it's redelivered, or complete processor with `StopProcessingAsync`); (5) flush telemetry, then exit. Risks: long-running requests exceeding the grace period get killed; non-idempotent work needs to be safe on interrupt. Trade-off: longer grace slows deploys.
**Follow-up traps**
- "Why preStop sleep?" (Endpoint removal propagation race.)
- "What about WebSockets/long polls?" (Notify clients to reconnect; cap lifetime.)

### 10.4 [Hard] The service is slow to start (60 s warmup). Liveness keeps killing it in a crash loop. Fix and prevent.
**What the interviewer is really testing**
Probe config reasoning.
**Typical mid-level answer**
"Raise the liveness timeout." Works but masks.
**Strong senior answer**
Add a `startupProbe` with failureThreshold x period covering the worst-case startup (e.g. 30 x 5 s = 150 s) - liveness only begins after it succeeds - and keep liveness tight afterwards. Investigate why startup is 60 s: eager loading, synchronous migrations, config fetch, JIT (consider ReadyToRun/AOT), cache warm. Move expensive work to background with readiness gating. Also avoid running DB migrations in each instance's startup (races and herd) - use a pipeline migration job. Trade-off: startup probe tolerance delays detection of truly broken start. Risk: a crash loop with exponential back-off (CrashLoopBackOff) plus a deploy herd takes capacity down.
**Follow-up traps**
- "Migrations at startup?" (Avoid; use separate step/job.)
- "What if startup depends on a down service?" (Start degraded; don't crash.)

### 10.5 [Scenario] A DB blip made all pods restart together and the outage lasted 12 minutes instead of 20 seconds. Explain.
**What the interviewer is really testing**
Seeing amplification caused by orchestration.
**Typical mid-level answer**
"The DB was slow." Misses the amplifier.
**Strong senior answer**
Likely: liveness/readiness checked the DB, so all pods failed and restarted at once; on restart they all hit cold caches and opened pools simultaneously against a recovering DB (herd), their startup also required the DB so they crash-looped with back-off, stretching the outage. Fixes: liveness doesn't touch dependencies; readiness uses cached/thresholded dependency checks or none; startup tolerates a down dependency (retry with backoff in the background); pool sizes and staggered restarts; PodDisruptionBudgets and rolling constraints. Prove it with a fault-injection test cutting the DB for 30 s and observing restarts. Risk: alert fatigue; alert on restart counts.
**Follow-up traps**
- "What would you check first in the timeline?" (Restart events vs DB recovery.)
- "How to prevent synchronized restarts?" (Don't tie liveness to external dependencies.)

**Red flags in answers**
- Single /health that hits the DB used for liveness.
- Ignoring endpoint-removal propagation during shutdown.
- No grace period alignment.

**Cheat sheet**
- Liveness = process only; readiness = can serve; startup = slow init.
- Deep checks go to dashboards, not orchestrators.
- Cache health results; short timeouts.
- Shutdown: fail readiness -> preStop delay -> drain -> stop workers -> flush.
- Grace period > preStop + drain + margin.
- Don't run migrations at pod startup.

---

## Topic 11: Availability Math (SLAs, nines, error budgets, SLO/SLI/SLA)

**Study notes**
Downtime per period: 99.9% = ~43.8 min/month (8.76 h/yr); 99.95% = ~21.9 min/month; 99.99% = ~4.4 min/month (52.6 min/yr); 99.999% = ~26 s/month (5.3 min/yr).
- **Serial** dependencies multiply: A x B = 0.999 x 0.999 = 99.8%.
- **Redundant/parallel** components: 1 - (1 - a)^n, assuming independent failures (rarely true).
- **SLI:** measured indicator (good requests / valid requests). **SLO:** internal target on SLIs. **SLA:** contract with consequences (credits), usually looser than SLO.
- **Error budget:** 1 - SLO; spend it on releases and risk; when exhausted, freeze features and fix reliability.
```
Chain: Front Door(99.99) x App(99.95) x SQL(99.99) x Redis(99.9) x 3rd-party(99.9) ~ 99.73%
```
Availability should also consider latency (slow = down) and the user journey, not just uptime.

### 11.1 [Easy] If service A (99.9%) calls B (99.9%) synchronously and both are required, what's the max availability?
**What the interviewer is really testing**
Serial composition.
**Typical mid-level answer**
"99.9%." Wrong.
**Strong senior answer**
About 99.8% (0.999 x 0.999 = 0.998), ~87 min/month of allowed downtime instead of 43. Each additional required synchronous dependency erodes the ceiling. Improvement levers: remove hard dependencies from the critical path (async, cache, fallback), so optional deps stop counting; or add redundancy to the weakest link. What fails when this fails: teams promise a 99.99% SLO on top of a stack whose composite math is 99.7% - the promise is unachievable regardless of code quality. Caveat: published vendor SLAs are contractual minimums and rarely equal real availability; also correlated failures break the independence assumption.
**Follow-up traps**
- "Two redundant instances at 99%?" (1 - 0.01^2 = 99.99% only if independent - they usually share failure causes.)
- "SLA vs actual availability?" (SLA is a floor with credits, not a measure.)

### 11.2 [Follow-up] Differentiate SLI, SLO, SLA, and error budget, and give an example for an API.
**What the interviewer is really testing**
Fluency and usefulness.
**Typical mid-level answer**
"SLA is uptime." Muddles terms.
**Strong senior answer**
SLI: a measured ratio like "requests answered with non-5xx in under 500 ms / valid requests", measured at the load balancer or client edge. SLO: the target, e.g. 99.9% over 28 days. SLA: the external agreement with penalties, set looser than the SLO so the team has a buffer. Error budget: 0.1% of requests (about 40 minutes of full outage equivalent per 28 days) - used to decide release velocity: if burn is high, slow down and fix reliability; if budget remains, ship. Use multi-window, multi-burn-rate alerts (e.g. fast burn over 5 min/1 h, slow burn over 6 h/3 days) instead of raw thresholds. Trade-off: too strict SLO slows delivery and is costly; too loose users suffer. Choose SLIs reflecting user journeys, excluding invalid client errors.
**Follow-up traps**
- "Should 4xx count as bad?" (Generally no, except suspected server-side causes.)
- "Why SLO tighter than SLA?" (Buffer before contractual breach.)

### 11.3 [Medium] Product wants 99.99%. What questions do you ask and what does it cost?
**What the interviewer is really testing**
Pushing back with data; understanding the cost step.
**Typical mid-level answer**
"Add redundancy." No clarifying.
**Strong senior answer**
Ask: availability of what (which journeys, measured how, what window)? what's the business impact per minute of downtime? is planned maintenance excluded? is partial degradation acceptable? 99.99% allows ~4.4 minutes of downtime a month - less than most human response times, so it requires automated detection and failover, zone redundancy everywhere, no single-region dependencies in the critical path (or active-active/multi-region), zero-downtime deploys, and graceful degradation; every dependency must be higher than 99.99% or removed from the critical path. Cost often 2-3x+ infra plus much more operational/engineering effort. Decision: propose tiers - core journey 99.99% with a reduced scope, other features 99.9%. Risk: the weakest link (third party, DNS, identity) and human-error deploys dominate.
**Follow-up traps**
- "Can you reach 99.99% on single region?" (Possible for some workloads with zone redundancy, but regional events fall outside.)
- "Excluding maintenance?" (Clarify; users don't care.)

### 11.4 [Hard] Your SQL (99.99%), App Service (99.95%), Redis (99.9%), and a vendor API (99.9%) are all on the request path. Estimate composite and propose how to raise it.
**What the interviewer is really testing**
Quantitative reasoning plus architecture moves.
**Typical mid-level answer**
"Pick the lowest, 99.9%." Not multiplicative.
**Strong senior answer**
Multiply: 0.9999 x 0.9995 x 0.999 x 0.999 ~ 0.9974 -> 99.74%, about 1.9 h/month budget vs the 4.4 minutes of a 99.99% goal. (Use actual current published SLAs.) Raise it by shrinking the critical path: make Redis optional (fallback to DB with limiter, local L1) so it's removed from the product; make the vendor call async or optional with fallback (queue, cache, alternative vendor); deploy App Service across zones, and maybe multi-region behind Front Door. Then remaining chain is dominated by infrastructure: App x SQL ~ 99.94%; multi-region active-active can lift it further but adds data-consistency complexity. Caveat: the math assumes independence; human error and shared control planes cause correlated failures. Caveat 2: measure real SLIs.
**Follow-up traps**
- "Is a fallback truly 100%?" (No; its availability counts too.)
- "What about deploy-related downtime?" (Often the largest share; zero-downtime releases.)

### 11.5 [Scenario] You've burned 80% of the monthly error budget in the first week. What do you do?
**What the interviewer is really testing**
Using error budgets as governance.
**Typical mid-level answer**
"Work harder on fixes." No policy.
**Strong senior answer**
Check what consumed it: one big incident or chronic small errors, which dependency, and whether it's in our control. Apply the error-budget policy agreed in advance: pause risky releases and feature work, prioritise reliability fixes for the top burn contributors (e.g. retry/timeout tuning, capacity, rollback automation), tighten release gates (canary, smaller batches), and communicate with product. Keep the budget conversation factual with burn-rate dashboards. Trade-off: delaying features versus customer trust; decision belongs to a pre-agreed policy not ad hoc heroics. Risk: gaming the SLI (excluding failures) - review SLI validity periodically; also if the SLO is consistently too easy/hard, revise it.
**Follow-up traps**
- "What if the cause is a vendor?" (Mitigations: fallback, or SLO adjustment with business agreement.)
- "Blameless postmortem output?" (Actions with owners, tied to budget.)

**Red flags in answers**
- "Two 99.9% services give 99.9%."
- Treating SLA, SLO, and uptime as the same.
- Assuming redundant components fail independently.

**Cheat sheet**
- Serial multiplies; parallel: 1-(1-a)^n (if independent).
- 99.9 ~ 43.8 min/mo; 99.99 ~ 4.4 min/mo.
- SLI measured, SLO target, SLA contract, budget = 1-SLO.
- Burn-rate alerts beat static threshold alerts.
- Removing a dependency from the critical path beats adding nines to it.
- 99.99% requires automation: humans are too slow.

---

## Topic 12: Disaster Recovery (RPO/RTO, backups, active-passive vs active-active, failover, runbooks)

**Study notes**
- **RPO:** max tolerable data loss (time). **RTO:** max tolerable downtime. Both drive cost.
- Backups protect against corruption/deletion/ransomware; replication protects against infrastructure loss - replication also copies mistakes. You need both.
- Azure SQL: automated backups with PITR; geo-restore; active geo-replication; auto-failover groups (read-write and read-only listener endpoints, failover grace period, async replication so possible data loss).
```
Active-passive: Region A (serving) --async repl--> Region B (standby/warm)
Active-active:  Region A <==== bidirectional / partitioned ====> Region B
Front Door / Traffic Manager route users
```
- **Pilot light / warm standby / hot standby** spectrum.
- DNS failover is limited by TTL and client caching; Front Door uses health probes and anycast edge so failover isn't DNS-TTL-bound for the origin switch.
- A backup you haven't restored is a hope, not a backup.

### 12.1 [Easy] Define RPO and RTO and how they change architecture and cost.
**What the interviewer is really testing**
Basic fluency and business linkage.
**Typical mid-level answer**
"RPO is data, RTO is time." Not tied to design.
**Strong senior answer**
RPO is how much data (measured in time) we can lose; RTO is how long until service is restored. RPO of hours allows periodic backups and geo-restore; RPO of seconds demands continuous replication (async geo-replication, or synchronous within a region/zone-redundant). RTO of days allows rebuild from IaC; RTO of minutes needs warm/hot standby with automated failover. Each tightening increases cost sharply. I'd derive them with the business per system (not one number for all), including dependencies: if the app has RTO 15 min but depends on a system with RTO 4 h, the effective RTO is 4 h. What fails when it fails: undefined targets lead to either overspending or surprise data loss.
**Follow-up traps**
- "Can RPO be 0 across regions?" (Needs synchronous commit, costs latency/availability.)
- "Who sets the targets?" (Business, informed by engineering.)

### 12.2 [Follow-up] Replication vs backup: why do you need both?
**What the interviewer is really testing**
Understanding failure classes.
**Typical mid-level answer**
"Replication is a backup." Common mistake.
**Strong senior answer**
Replication copies state quickly, so a bad `DELETE`, a corrupt deploy, or ransomware replicates to the secondary within seconds. Backups give point-in-time restore to before the error (Azure SQL PITR, retention configurable, long-term retention for compliance), and immutable/soft-delete storage protects against malicious deletion. Replication addresses hardware/zone/region loss with low RTO/RPO. Decision: both, plus tested restore procedures, isolated backup access (separate credentials/subscription, immutability), and clarity about RPO for logical errors (e.g. PITR granularity). Trade-off: storage and cost. What fails when it fails: you restore the replica and discover it contains the corruption too.
**Follow-up traps**
- "How often do you test restore?" (Regularly, automated, with timing measured against RTO.)
- "Who can delete backups?" (Separation of duties.)

### 12.3 [Medium] Active-passive vs active-active: choose for a SQL-backed line-of-business app.
**What the interviewer is really testing**
Trade-offs on data consistency and ops complexity.
**Typical mid-level answer**
"Active-active because it's more available." Ignores data.
**Strong senior answer**
Clarify RPO/RTO and write patterns. Active-passive (auto-failover group, warm app in secondary region): simple data model with a single writer, RPO of seconds (async), RTO minutes; costs a standby. Active-active needs multi-writer data: conflict resolution, or region-partitioned ownership (each tenant has a home region), or a database designed for it (Cosmos DB multi-region writes); SQL Server/Azure SQL has a single primary per database, so active-active over one SQL DB generally means reads in both and writes to one, or sharding. Decision for most LOB apps: active-passive with automated, rehearsed failover; go active-active only if the business requires near-zero RTO and I can partition data. Risks: split-brain, stale reads in secondary, config drift between regions, and untested failover paths.
**Follow-up traps**
- "Is active-passive's standby tested?" (Must be; drift is the main risk.)
- "Which is cheaper?" (Active-passive, generally.)

### 12.4 [Hard] List the pitfalls of multi-region failover with async replication.
**What the interviewer is really testing**
Depth beyond "turn on geo-replication".
**Typical mid-level answer**
"Data loss." One of many.
**Strong senior answer**
(1) Data loss equal to replication lag at failover (forced failover with `allow data loss`). (2) Split-brain if the old primary comes back accepting writes - fencing needed. (3) Failing over only the DB while apps stay in the old region adds cross-region latency; or app fails over but its secrets, Key Vault, storage, Service Bus, DNS, identity aren't replicated - every dependency needs a DR story. (4) Capacity: the secondary region may lack quota/capacity when everyone fails over - pre-provision or reserve. (5) Failback is a second risky migration, with reverse replication and reconciling writes. (6) Stale config/IaC drift between regions. (7) Failure detection ambiguity: automatic failover on a transient blip causes unnecessary data loss; choose grace periods (failover group grace period) and human-in-loop for regional decisions. (8) Messages in flight and idempotency across regions. Trade-off: automatic is quick but risks false failover.
**Follow-up traps**
- "Auto vs manual failover?" (Depends on RTO; large blast-radius -> often a decision gate.)
- "What about queue messages in the lost region?" (Lost or delayed - design for reconciliation.)

### 12.5 [Medium] How would you do DNS/Front Door failover and what are the limits?
**What the interviewer is really testing**
Traffic steering mechanics.
**Typical mid-level answer**
"Traffic Manager switches DNS." Doesn't mention TTL/caches.
**Strong senior answer**
Traffic Manager is DNS-based: failover time ~ health probe interval x failures + DNS TTL + client/resolver caching, and some clients ignore TTLs, so RTO is minutes, not seconds. Azure Front Door is a global L7 proxy: it probes origins and shifts traffic quickly with priority/weight routing and no client DNS change, but Front Door itself becomes a dependency (its own global service) - consider that in availability math and for emergency bypass runbook. Design: health probe endpoint that reflects real readiness of the stack (careful with deep checks), priority routing for active-passive or latency routing for active-active, session affinity implications, and a tested manual override. Trade-off: probe sensitivity vs flapping.
**Follow-up traps**
- "Is Front Door a SPOF?" (Mitigate with documented bypass path; consider Traffic Manager in front for DR.)
- "Sticky sessions?" (Avoid state in instances.)

### 12.6 [Scenario] Region outage playbook: the primary region is degraded; you have 30 minutes RTO. Describe it.
**What the interviewer is really testing**
Runbook thinking and decision-making under pressure.
**Typical mid-level answer**
"Fail over to the other region." No decision criteria.
**Strong senior answer**
Preconditions: a written, rehearsed runbook. Steps: declare incident and roles; confirm from independent signals (Azure Service Health, our probes, synthetic checks) that it's regional, not our deploy; decide failover using pre-agreed criteria (expected outage duration vs RTO, data-loss tolerance - RPO lag from replication metrics); freeze deploys; initiate DB failover group failover (forced if primary unreachable), verify the secondary accepted writes, scale up secondary app tier if warm standby, update routing (Front Door priority), validate with synthetic transactions, communicate on status page. Post: monitor lag, reconcile lost transactions, plan failback during a low-risk window, then postmortem. Trade-off: failing over too early loses data; too late breaches RTO. Risk: runbook not exercised, missing permissions or quotas at the secondary - verify by game days.
**Follow-up traps**
- "How do you reconcile lost writes?" (Compare last-replicated LSN/time; replay from source events/outbox if any.)
- "Who has authority to fail over?" (Predefined incident commander.)

**Red flags in answers**
- "We have geo-replication so we have backups."
- Never having done a restore or failover drill.
- Ignoring non-DB dependencies in DR.

**Cheat sheet**
- RPO = data loss window; RTO = downtime window.
- Replication != backup; need PITR + immutability.
- Async replication => possible data loss on forced failover.
- DNS failover bounded by TTL; Front Door faster but is itself a dependency.
- Failback is a project; plan it.
- Restore tests and game days or the DR plan is fiction.

---

## Topic 13: Chaos Engineering & Testing Resilience

**Study notes**
Chaos engineering = run controlled experiments on a system to build confidence in its ability to withstand turbulence. Loop: define steady state (SLIs) -> hypothesis -> inject fault -> observe -> fix -> automate. Start in non-prod, small blast radius, with abort conditions, then progress to prod with guardrails.
Azure Chaos Studio: experiments composed of targets + faults. Service-direct faults (e.g. Cosmos DB failover, Key Vault deny access, NSG rule changes, AKS faults via Chaos Mesh) and agent-based faults on VMs (CPU/memory pressure, network latency/disconnect, process kill). Experiments run under a managed identity needing least-privilege permissions on targets.
Other tools: Polly Simmy (chaos strategies for latency/fault/outcome injection in .NET), toxiproxy, load tests (Azure Load Testing), fault-injection in integration tests via `WebApplicationFactory` + `HttpMessageHandler` stubs.
```
Steady state OK? -> inject -> still OK? -> yes: confidence / no: found weakness (before customers did)
```

### 13.1 [Easy] What is chaos engineering and what is it *not*?
**What the interviewer is really testing**
Scientific, controlled mindset.
**Typical mid-level answer**
"Randomly killing servers in prod." Misses hypothesis and controls.
**Strong senior answer**
It's hypothesis-driven experimentation: define measurable steady state, state "if X fails, SLI stays within Y", inject the fault with a limited blast radius and abort conditions, compare, then fix gaps and automate regression. It is not random destruction or a substitute for unit/integration tests; it validates assumptions (timeouts, failover, fallbacks, alerts, runbooks) that tests rarely cover. Begin in staging, move to production for fidelity once guardrails (abort button, SLO monitors, business-hours, on-call aware) exist. What fails when it fails: the lesson - we find out the fallback didn't work in a safe experiment rather than at 3 am.
**Follow-up traps**
- "Why production?" (Staging rarely matches traffic/config/data - but earn it with guardrails.)
- "What is the abort condition?" (SLO burn threshold.)

### 13.2 [Follow-up] Design three first experiments for a .NET API with SQL, Redis, and a third-party API.
**What the interviewer is really testing**
Practical prioritisation.
**Typical mid-level answer**
"Kill the database." Too big and unspecific.
**Strong senior answer**
(1) Third-party latency injection (e.g. +3 s, then total failure): hypothesis: timeouts + breaker + fallback keep checkout p99 under budget and error rate < 0.5%; observe breaker state and degradation metrics. (2) Redis unavailable (block network/Chaos Studio or toxiproxy): hypothesis: DB load stays within bounded limits through single-flight and limiter, no thread starvation. (3) SQL failover (Azure SQL failover trigger or failover group test in non-prod): hypothesis: EF retry + pool recovery keep error window under 30-60 s; observe connection-pool errors and poison writes (idempotency). Each with a steady-state definition, rollback, and an owner. Trade-off: realism vs risk; start with latency/partial faults before total loss. Risk: experiments without observability only produce anecdotes - verify dashboards first.
**Follow-up traps**
- "What if the hypothesis fails?" (That's the finding; fix, re-run.)
- "Who must be notified?" (On-call, stakeholders.)

### 13.3 [Medium] How do you test retries, breakers and fallbacks in automated tests (not just chaos)?
**What the interviewer is really testing**
Testability of resilience policies.
**Typical mid-level answer**
"Mock the service to throw." Doesn't validate timing/state.
**Strong senior answer**
Layered: unit-test the pipeline with a fake `HttpMessageHandler` (or Simmy) returning 503/timeouts and assert attempts, delay behaviour (use `TimeProvider`/fake time so tests run fast), breaker state transitions, and fallback outputs. Integration tests with `WebApplicationFactory` and a stubbed dependency (WireMock, toxiproxy for latency/reset) to assert end-to-end: status code, headers (Retry-After), degraded payload. Contract tests to ensure the fallback shape matches. Load tests with fault injection to test overload behaviour (shedding, bulkhead rejection) and check that recovery occurs after the fault is removed. Trade-off: time-based tests are flaky without a controllable clock. Risk: tests that assert only "it didn't crash".
**Follow-up traps**
- "How to avoid slow tests?" (Fake `TimeProvider`.)
- "Does a passing unit test prove prod behaviour?" (No - config/platform differences; combine with experiments.)

### 13.4 [Hard] Convince a sceptical VP to allow fault injection in production.
**What the interviewer is really testing**
Risk communication.
**Typical mid-level answer**
"Netflix does it." Appeal to authority.
**Strong senior answer**
Frame it as risk reduction: incidents happen anyway; we choose between learning in a controlled, supervised window and learning during a customer outage. Proposal: phased - staging first, then production on a small slice (one instance/1% of traffic or an internal tenant), during business hours with on-call present, automatic abort on SLO burn, pre-agreed blast radius, and a rollback. Present results: expected cost of an hour of downtime vs the time cost of experiments; evidence from staging findings. Start with game days (manually simulated, e.g. disable a replica) before automated chaos. Trade-off: small chance of customer impact vs bigger unknown risk. Risk: poor guardrails; mitigate with approvals and a kill switch.
**Follow-up traps**
- "What if customers are harmed?" (Blast radius limits, abort conditions, comms plan.)
- "How do you prove value?" (Findings fixed, reduced MTTR.)

### 13.5 [Scenario] Your Chaos Studio network-latency experiment on an API VM shows no impact, but customers saw issues in the last incident. Why might the experiment be misleading?
**What the interviewer is really testing**
Experiment validity.
**Typical mid-level answer**
"We're fine then." Overconfident.
**Strong senior answer**
Possible reasons: the injected fault didn't match reality (fixed latency on one path versus real partial/gray failure on the dependency); traffic in the experiment was too low to exhaust pools/threads; the experiment targeted a non-critical hop; steady-state metric wasn't sensitive (average vs p99); caches masked effects; the incident was triggered by a combination (retry storm + deploy) rather than a single fault. Improve: replicate incident conditions (latency distribution, load level, concurrent deploy), measure p99 and saturation metrics, include dependency slowness (not just total outage), run under load. Trade-off: complex scenarios are harder to control. Risk: false assurance - treat a clean result as "not disproved", not "proved".
**Follow-up traps**
- "Is passing a chaos test proof?" (No.)
- "How do you build realism?" (Postmortem-derived scenarios.)

**Red flags in answers**
- "Chaos = random killing" without hypotheses/guardrails.
- No observability before experiments.
- Only testing the happy "total failure" case.

**Cheat sheet**
- Steady state -> hypothesis -> inject -> observe -> fix -> automate.
- Small blast radius + abort conditions.
- Chaos Studio: service-direct and agent-based faults; managed identity permissions.
- Simmy/toxiproxy/WireMock for code-level fault injection.
- Test slowness and partial failure, not just outages.
- Turn every incident into a regression experiment.

---

## Topic 14: Resilient Messaging, Poison Messages, Dead-Lettering & Replay

**Study notes**
Service Bus (peek-lock): receive -> process -> complete; failure -> abandon/lock expiry -> redelivery; after `MaxDeliveryCount` (default 10) -> dead-letter queue (DLQ, sub-queue). Lock duration default 1 min (max 5); processor `MaxAutoLockRenewalDuration` (default 5 min) renews locks. Duplicate detection (producer side, default window 10 min). Sessions for per-key ordering.
Delivery is **at-least-once** => consumers must be idempotent.
```
Outbox:  [Tx: update order + insert OutboxMessage] -> relay publishes -> mark sent   (at-least-once)
Inbox:   [Tx: insert ProcessedMessage(MessageId) unique + apply effect]            (dedupe)
```
Poison message: always fails (bad payload/bug). Transient failure: retry. Distinguish and route accordingly.

### 14.1 [Easy] What does at-least-once delivery imply for your handler design?
**What the interviewer is really testing**
Idempotency and duplicates.
**Typical mid-level answer**
"Messages might arrive twice." No design implication.
**Strong senior answer**
The handler can run multiple times for the same message (lock expiry, crash before complete, redelivery after abandon), so effects must be idempotent: use naturally idempotent operations (upsert/set), or an inbox/dedupe table with a unique constraint on MessageId committed in the same transaction as the business effect, or conditional updates with version checks. Also ordering isn't guaranteed (except sessions/FIFO), so handlers should tolerate out-of-order messages. Side effects outside the DB (emails, payments) need idempotency keys or "sent" markers. What fails when it fails: duplicate emails/charges, double-counted inventory.
**Follow-up traps**
- "Exactly-once?" (Not end-to-end; effectively-once via idempotency.)
- "Is Service Bus duplicate detection sufficient?" (Only producer duplicates within its window.)

### 14.2 [Follow-up] Explain the transactional outbox and why dual-writes fail.
**What the interviewer is really testing**
Consistency between DB and broker.
**Typical mid-level answer**
"Save then publish." Dual write.
**Strong senior answer**
Dual write: update DB, then publish; if the process crashes between, the DB changed but no event is published (lost event), or if publishing first, the event exists without state (phantom). No distributed transaction between SQL and Service Bus is practical. Outbox: in the same local transaction write the business row and an `Outbox` row; a relay (background worker/Change feed/polling) publishes and marks sent; delivery is at-least-once, so consumers dedupe by MessageId (set the broker MessageId to the outbox id to also use duplicate detection). Trade-off: extra latency, relay operations (ordering, cleaning old rows, scaling relay with `SELECT ... WITH (UPDLOCK, READPAST)` for parallel claim). Risk: outbox backlog if the broker is down - monitor lag; the DB buffers meanwhile, which is a resilience benefit.
**Follow-up traps**
- "Relay crashes after publish before marking?" (Duplicate publish; consumers dedupe.)
- "Ordering?" (Per aggregate key; sequence numbers.)

### 14.3 [Medium] A message fails processing repeatedly. How do you handle poison messages without blocking the queue or losing data?
**What the interviewer is really testing**
DLQ strategy and classification.
**Typical mid-level answer**
"Let it go to the DLQ after 10 tries." Ignores classification and retry cost.
**Strong senior answer**
Classify: transient (dependency down) -> abandon with delay/backoff (or scheduled re-enqueue) and do not burn delivery count too fast; permanent (deserialization/validation failure) -> dead-letter immediately with a reason and description (`DeadLetterAsync` with properties) - retrying cannot help. Tune `MaxDeliveryCount` and use exponential delays via re-scheduling; consider the dependency being down: 10 quick retries dead-letter all good messages - so combine with a circuit breaker that pauses the processor instead of failing messages. Operations: alert on DLQ depth/age, include the original headers and exception, tooling to inspect/resubmit, and a retention policy. Trade-off: DLQ = a growing backlog if unattended. Risk: head-of-line blocking with sessions (a poison session blocks that session) - dead-letter or defer.
**Follow-up traps**
- "Dependency outage fills the DLQ?" (Pause consumption using breaker; don't dead-letter transient failures.)
- "Who owns the DLQ?" (Named team, runbook.)

### 14.4 [Hard] Design replay/reprocessing of dead-lettered or historical messages safely.
**What the interviewer is really testing**
Operational safety of reprocessing.
**Typical mid-level answer**
"Move them back to the queue." Risks duplicates and floods.
**Strong senior answer**
Prerequisites: handlers idempotent and tolerant of old schemas/out-of-order messages. Process: investigate the cause and fix first; replay a small sample to the real handler (or a shadow) and verify; then replay throttled (rate-limited, off-peak) so it doesn't starve live traffic or overload downstream; preserve original MessageId/correlation so dedupe works, add a `replay` marker for audit; track progress and failures, and keep the DLQ copy until success is confirmed. For event-sourced/stream use cases, replay from checkpoints with idempotent projections. Trade-off: speed vs risk of re-triggering side effects (emails, payments) - gate external side effects with "replay mode" or suppress them. Risk: messages whose business context has changed (stale) - validate before applying, maybe expire.
**Follow-up traps**
- "Replay triggers customer emails?" (Suppress/flag side effects.)
- "Schema changed since?" (Versioned upcasters.)

### 14.5 [Medium] Handler takes 8 minutes occasionally. What breaks with Service Bus locks and how to handle?
**What the interviewer is really testing**
Lock renewal mechanics.
**Typical mid-level answer**
"Increase the lock duration." Limited to 5 min.
**Strong senior answer**
The peek-lock expires (default 1 min, max 5 min), the message becomes visible again and a second consumer starts it while the first still runs - duplicate concurrent work; when the first tries `CompleteAsync` it fails with `MessageLockLost`. Options: `ServiceBusProcessor` auto-renews locks up to `MaxAutoLockRenewalDuration` (set it above worst-case processing time); for very long work, don't hold the message: receive, record a job, complete quickly, and run long work via a durable job/state machine (Durable Functions, a work table + worker, or a continuation message with checkpoints). Make handler idempotent regardless. Trade-off: renewal depends on the client process being alive and network reachable; a paused process loses locks. Risk: lock renewal failures during GC pause or network blips - handle `MessageLockLost` gracefully.
**Follow-up traps**
- "What happens on MessageLockLost at completion?" (Work was done but message redelivered - idempotency again.)
- "Max lock duration?" (5 minutes; renewal extends.)

### 14.6 [Scenario] Production consumer crashes on one malformed message and the queue backs up behind it. Investigate and fix.
**What the interviewer is really testing**
Triage and long-term fixes.
**Typical mid-level answer**
"Delete the message." Data loss, no prevention.
**Strong senior answer**
Immediate: with peek-lock the message should only block if processing crashes the whole process (e.g. unhandled exception killing the host, or OOM) so the instance restarts and gets the same message repeatedly (until delivery count hits max). Check delivery counts and logs; move the offending message to the DLQ manually (or let max delivery do it - lower it temporarily); scale/resume. Fix: catch exceptions per message, dead-letter on deserialization failure with reason, protect against process-killing payload sizes (limit sizes/validate), add schema validation at the producer. Alerts: DLQ count, active message age, consumer restarts. Trade-off: strict validation at the producer can break flexible producers - define contracts. Risk: a single poison on a session blocks that session's flow.
**Follow-up traps**
- "How do you not lose the message?" (DLQ with reason.)
- "Preventing crash loops?" (Handler-level try/catch; resource limits.)

**Red flags in answers**
- Claiming exactly-once delivery from the broker.
- Publishing to the broker inside/after a DB commit with no outbox.
- Letting transient failures drain good messages into the DLQ.

**Cheat sheet**
- At-least-once => idempotent handlers + inbox dedupe.
- Outbox solves DB+broker dual-write.
- Poison -> DLQ with reason; transient -> backoff/pause.
- Replay: fix first, throttle, preserve IDs, gate side effects.
- Lock default 1 min (max 5); use auto-renew; long work -> job pattern.
- Alert on DLQ depth and oldest-message age.

---

## Topic 15: Zero-Downtime Deployments, Feature Flags & Kill Switches

**Study notes**
- **Rolling:** replace instances gradually (K8s `maxSurge`/`maxUnavailable`). Both versions run side by side -> must be compatible.
- **Blue-green:** two full environments; switch traffic (App Service slot swap, Front Door/Traffic Manager). Fast rollback, double cost during switch, DB shared still needs compat.
- **Canary:** route a small percentage to the new version, compare SLIs, then ramp (Container Apps revisions/traffic splitting, AKS with a mesh/ingress, Front Door weights).
- **DB changes: expand -> migrate -> contract.**
```
Expand:   add nullable column / new table (old code still works)
Deploy:   new code writes both (or new), reads old-or-new
Backfill: batch migrate data
Contract: remove old column once no code uses it (a later release)
```
- **Feature flags** (Azure App Configuration + Microsoft.FeatureManagement): decouple deploy from release; kill switches turn off a feature instantly without redeploy.

### 15.1 [Easy] Rolling vs blue-green vs canary: when do you pick each?
**What the interviewer is really testing**
Trade-offs awareness.
**Typical mid-level answer**
"Blue-green is best." No context.
**Strong senior answer**
Rolling: cheapest, default for stateless services, but old/new coexist and rollback is another roll. Blue-green: instant switch/rollback with a pre-validated environment (App Service slots with warm-up before swap), costs duplicate capacity and doesn't solve shared-database compatibility. Canary: limits blast radius and gives real-traffic evidence using SLI comparison (error rate, latency, business metrics) with automatic abort; needs routing control and good telemetry, and enough traffic for statistical meaning. I'd combine: canary + rolling for services; blue-green slots for web apps; always backward-compatible schemas. What fails when it fails: a bad version reaches everyone at once if no staged exposure.
**Follow-up traps**
- "Slot swap and warm-up?" (Use warm-up/swap-with-preview to avoid cold instances.)
- "Low traffic service canary?" (Not statistically meaningful; use synthetic tests.)

### 15.2 [Follow-up] Walk through renaming a column with zero downtime.
**What the interviewer is really testing**
Expand-contract discipline.
**Typical mid-level answer**
"Run a migration that renames it during deploy." Breaks old instances.
**Strong senior answer**
`sp_rename` immediately breaks running old code (and rollback). Do: (1) Expand - add the new column (nullable/default), deploy code that writes to both columns and reads the new one with fallback to old; (2) backfill existing rows in small batches (avoid long locks/log growth; throttle); (3) switch reads to new only after verification; (4) Contract - in a later release, stop writing the old column, then drop it. Each step is independently deployable and reversible. Trade-off: several releases and temporary duplication versus one outage. Risk: large-table schema changes can still take locks (check online operation support, e.g. `ONLINE = ON` for index ops, and test on prod-size data), and ORMs with auto-migration on startup racing across instances - run migrations as a controlled pipeline step.
**Follow-up traps**
- "When can you drop the old column?" (After all versions that read it are retired and rollback window passes.)
- "EF `Migrate()` at startup?" (Avoid in multi-instance prod.)

### 15.3 [Medium] How do feature flags improve resilience, and how can they hurt?
**What the interviewer is really testing**
Operational safety and flag debt.
**Typical mid-level answer**
"Turn features on and off." Misses risks.
**Strong senior answer**
They separate deploy from release: ship dark, enable for internal users, ramp by percentage, and kill instantly when metrics degrade - far faster than redeploy (seconds vs tens of minutes). Kill switches for non-critical or risky dependencies enable load shedding during incidents. Risks: the flag service is a dependency - cache flag values locally with defaults ("fail safe": define the default state if config is unreachable; App Configuration provider refreshes with caching); flag combinatorics increase untested states; stale flags become debt; permissions - who can flip prod flags (audit). Practices: owner+expiry per flag, remove after rollout, test both states, default to the safe state. Trade-off: complexity vs control.
**Follow-up traps**
- "Flag service down?" (Last-known-good cache; explicit default.)
- "How long should flags live?" (Release flags short-lived; ops kill switches may be permanent.)

### 15.4 [Hard] Your canary shows no errors but revenue per session dropped 5%. What do you do, and what does it say about your gates?
**What the interviewer is really testing**
Byzantine-lite failures; business metrics in gates.
**Typical mid-level answer**
"Ship it, no errors." Misses the signal.
**Strong senior answer**
Halt ramp and roll back (or hold) - a silent regression counts as failure. Investigate: is the sample sufficient and the comparison fair (same segments, same time)? Check functional diffs (bad pricing rule, missing recommendation, JS error, slower page). This shows the canary gate relied only on technical SLIs; add business/synthetic SLIs (conversion, order success, latency per step), automatic comparison against baseline with statistical thresholds, and soak time to catch slow leaks (memory, caches). Trade-off: more gates slow releases and risk false positives; tune thresholds. Risk: small canaries lack statistical power for 5% effects - need proper sample sizing or A/B tooling.
**Follow-up traps**
- "How long to hold canary?" (Enough for traffic cycles and sample size.)
- "Rollback when schema changed?" (Backward-compatible expand step.)

### 15.5 [Scenario] A deploy requires a data migration that locks a 500M-row table for 20 minutes. How do you ship it with no downtime?
**What the interviewer is really testing**
Practical large-table change strategies.
**Typical mid-level answer**
"Schedule a maintenance window." Not zero-downtime.
**Strong senior answer**
Clarify the change: new column, type change, index? Prefer online alternatives: add nullable column (metadata-only in most cases), backfill in batches with small transactions and throttling (watch log/replica lag, blocking), build indexes with `ONLINE = ON` and resumable options where supported (edition/version dependent), and for type changes use a shadow column plus dual-write then cut over. Run during low traffic with monitoring and a stop switch (flag). Test on a prod-sized copy to measure duration and lock behaviour. Keep the app compatible with both schemas throughout. Trade-off: weeks of staged work vs a single risky window. Risk: backfill adds load - may need to coordinate with the replica/failover group lag and transaction log growth.
**Follow-up traps**
- "How do you stop if blocking appears?" (Batch + abort flag.)
- "Replica lag?" (Throttle based on lag.)

**Red flags in answers**
- Schema changes that break the previous app version.
- Canary judged only by "no 5xx".
- Flags with no defaults for when the flag service is unreachable.

**Cheat sheet**
- Rolling = cheap; blue-green = fast rollback; canary = evidence.
- Expand -> migrate -> contract; never rename in place.
- Backward-compat across N and N-1 versions.
- Gates: technical + business SLIs; automated abort.
- Flags: safe default, owner, expiry, audit; kill switches for ops.
- Migrations: pipeline step, not app startup.

---

## Topic 16: Resilient Data Access (EF Core, pooling limits, failover groups, transient faults)

**Study notes**
- EF Core + SQL Server: `UseSqlServer(cs, o => o.EnableRetryOnFailure(maxRetryCount: 6, maxRetryDelay: TimeSpan.FromSeconds(30), errorNumbersToAdd: null))`. Default retries on known transient errors (e.g. 40613, 40501, 40197, 10928, 10929, 49918-49920, 4060, plus deadlock 1205 and timeouts in the built-in list - check the current `SqlServerTransientExceptionDetector`).
- With retry strategy, **user-initiated transactions must run inside `strategy.ExecuteAsync`** so the whole unit is re-executed; and the unit must be safe to repeat (the commit outcome might be unknown).
- ADO.NET pool: per connection string (and identity), default `Max Pool Size` = 100; exhaustion error "Timeout expired... obtaining a connection from the pool". `Connect Timeout` default 15 s. `ConnectRetryCount` default 1, `ConnectRetryInterval` 10 s (idle connection resiliency). Microsoft.Data.SqlClient also has configurable retry logic.
- Azure SQL failover: connections drop, pooled connections to the old primary are broken; use the failover group **listener** endpoint (read-write and read-only), not server names.
```
App --> fog-name.database.windows.net --> (current primary)
                         failover => listener now points to other region's server
```

### 16.1 [Easy] What does `EnableRetryOnFailure` do, and what are its limits?
**What the interviewer is really testing**
Know the tool and its hidden hazards.
**Typical mid-level answer**
"It retries failed queries." Misses transactions and idempotency.
**Strong senior answer**
It installs an execution strategy that retries operations on a list of transient SQL errors with exponential backoff (default 6 attempts, max delay 30 s). Limits: (1) it doesn't know your commands' side effects - if a commit's outcome is unknown (connection died during commit), a retry could repeat the work, so the unit must be idempotent or use a verification check (EF has commit-verification for some cases but don't rely on it for business semantics); (2) explicit transactions must be wrapped in `strategy.ExecuteAsync(...)` else EF throws; (3) total retry time may exceed your request deadline - tune count/delay to the budget; (4) it stacks with outer Polly retries - amplification. Trade-off: convenience vs hidden latency. What fails when it fails: for failovers longer than the retry window you get a burst of errors - the pool and breaker must recover cleanly.
**Follow-up traps**
- "Does it retry deadlocks?" (Deadlock 1205 is treated as transient in the default detector - verify current behaviour.)
- "Can it be used with manual transactions?" (Only via `CreateExecutionStrategy`.)

### 16.2 [Follow-up] Connection pool exhaustion: causes, symptoms, and prevention.
**What the interviewer is really testing**
Operational diagnosis.
**Typical mid-level answer**
"Increase Max Pool Size." Treats the symptom.
**Strong senior answer**
Causes: connections held too long (slow queries, a transaction spanning remote calls), leaks (undisposed `SqlConnection`/`DbContext` with manual management), too much concurrency vs pool size, sync-over-async blocking, a slow DB making each request hold longer (Little's Law). Symptoms: "timeout obtaining connection from pool" errors with low CPU on the app. Prevention: dispose via `using`/DI scopes; keep transactions short; per-dependency concurrency limits so requests queue/shed before taking connections; set `Max Pool Size` so that instances x pool <= DB max connections/worker capacity (autoscale makes this dangerous); query timeouts; monitor pool metrics (EventCounters, `sys.dm_exec_sessions`). Trade-off: larger pools shift load to a DB that may already be the bottleneck. Risk: raising the pool during an incident can worsen the DB overload.
**Follow-up traps**
- "How to find leaks?" (Count sessions per app name; dump long-open connections.)
- "Pool per app name?" (Different connection strings = separate pools - a bulkhead tool.)

### 16.3 [Medium] An Azure SQL failover causes 5 minutes of errors. Walk through why that might happen even though failover itself takes seconds.
**What the interviewer is really testing**
Client-side reasons for long recovery.
**Typical mid-level answer**
"Azure was slow." Blames platform.
**Strong senior answer**
Platform failover is usually seconds, so the remaining minutes are client-side: pooled connections to the old primary stay broken until detected, with default idle resiliency only a limited retry; retry policy too short (6 retries ~ under a minute) or absent; connection strings using the server name rather than the listener/failover group endpoint; DNS caching of the old IP; long `Connect Timeout`/command timeouts making each failed attempt slow; breaker open duration longer than recovery; liveness probes restarting pods in the middle (Topic 10); herd of reconnects after the recovery. Fixes: use the failover group listener, retry/backoff sized for tens of seconds to a couple of minutes, appropriate pool clearing on repeated failures, breaker with short half-open probing, health checks that don't restart pods, and test with a forced failover regularly. Trade-off: longer retries increase latency during failover but save requests.
**Follow-up traps**
- "How would you test?" (Planned failover in test; Chaos/failover groups.)
- "What about writes in flight?" (Unknown outcome - idempotency.)

### 16.4 [Hard] Read replicas / `ApplicationIntent=ReadOnly`: what resilience benefits and hazards?
**What the interviewer is really testing**
Replica lag & consistency.
**Typical mid-level answer**
"Offload reads, so better." Ignores staleness.
**Strong senior answer**
Benefits: relieves the primary, isolates reporting from OLTP (bulkhead), and gives a degraded read-only mode if the primary fails (partially). Hazards: replica lag causes read-your-writes violations (user saves, then page shows old data) - route post-write reads to primary or use session-sticky/LSN-aware logic; replicas share the same failure domain in some tiers; if the replica is the only source for certain endpoints, you've added a dependency; connection strings for read-only are separate pools. Decision: use for reporting/search and tolerant reads; keep critical flows on primary; define staleness tolerance. Risk: failover changes which node is secondary; design for reconnect and test read-only paths under failover.
**Follow-up traps**
- "Read-your-writes fix?" (Read from primary shortly after writes or track version.)
- "Is the readable secondary in a failover group in another region?" (Yes via read-only listener; lag applies.)

### 16.5 [Scenario] After scaling the API from 10 to 40 instances during a spike, SQL started rejecting connections and the app got worse. Explain and fix.
**What the interviewer is really testing**
Scale-out hitting downstream limits.
**Typical mid-level answer**
"Scale up the DB." Possibly, not the root fix.
**Strong senior answer**
40 instances x default 100 pool = up to 4,000 potential connections; Azure SQL has per-tier session/worker limits (error 10928/10929 resource limits), so extra connections plus parallel queries exceeded worker capacity, and rising latency increased connection hold time - a feedback loop. Fixes: cap `Max Pool Size` (e.g. 20-30) based on DB capacity/instance count, add concurrency limits and shed before DB access, tune expensive queries/indexes (the spike probably exposed them), cache reads, use read replicas for reads, scale the DB tier for the planned event, and cap autoscale max instances to what the DB supports. Trade-off: lower pool limits mean requests queue in the app earlier - better than a DB collapse. Risk: pool sizes set once and forgotten as instance counts grow.
**Follow-up traps**
- "What's the key formula?" (max instances x pool size <= DB capacity.)
- "Could a connection proxy help?" (Limited options in Azure SQL; design app-side.)

**Red flags in answers**
- Retrying unknown-outcome writes without idempotency.
- Server names instead of failover-group listener endpoints.
- Raising pool sizes as the first reaction.

**Cheat sheet**
- `EnableRetryOnFailure` + `strategy.ExecuteAsync` for transactions.
- Retry budget total must fit your deadline; avoid stacking retries.
- Pool: default 100 per connection string; instances x pool <= DB limits.
- Use the failover group listener; test failover.
- Replicas add staleness; protect read-your-writes.
- Short transactions, no remote calls inside them.

---

## Topic 17: Autoscaling Pitfalls, Noisy Neighbours & Multi-Tenant Fairness

**Study notes**
Autoscaling: reactive (metric thresholds: CPU, queue length, requests), scheduled, predictive. Pitfalls:
- **Scale lag:** metric collection + decision + provisioning + warm-up = minutes.
- **Wrong metric:** CPU doesn't capture I/O-bound or queue-bound saturation; scale on request concurrency/queue age (KEDA on Service Bus, HPA custom metrics).
- **Downstream limits:** more instances -> more DB connections/3rd-party calls.
- **Flapping:** thresholds too tight; use cooldowns.
- **Cold start:** JIT, cache empty, pool opening; Azure Functions Consumption cold start; Azure SQL serverless auto-pause resume delay.
- **Scale-in** kills in-flight work: graceful shutdown.
Noisy neighbour: one tenant/workload consumes shared resources (CPU, DB, IOPS, queue consumers, API quota). Controls: per-tenant quotas/rate limits, resource governance (Azure SQL elastic pools min/max per DB, Resource Governor in SQL Server/Managed Instance), partitioned queues, tenant-aware throttling, separate tiers for large tenants (cell-based architecture).

### 17.1 [Easy] Why can't autoscaling alone save you from a sudden traffic spike?
**What the interviewer is really testing**
Realism about reaction time.
**Typical mid-level answer**
"It scales automatically." Overconfident.
**Strong senior answer**
Autoscale reacts after metrics breach, and then instances must provision, start, warm up, and pass readiness - often several minutes. A spike lasting 2 minutes is over before capacity arrives, and during the lag requests queue or fail. Mitigations: baseline headroom (don't run hot), scheduled/predictive scaling for known events, scale on leading indicators (queue length/concurrency), faster start (smaller images, ReadyToRun, warm pools/always-ready instances on Functions Premium), and load shedding/rate limits to bridge the gap. What fails when it fails: the lag period is where outages occur, plus new instances aggravating the downstream.
**Follow-up traps**
- "Scale-in too fast?" (Cooldowns; keep scale-in conservative.)
- "Metric choice?" (Leading indicators over CPU alone.)

### 17.2 [Follow-up] Scale-out increased throughput of the API but made the downstream partner API throttle us (429s). What now?
**What the interviewer is really testing**
Bottleneck thinking, global limits.
**Typical mid-level answer**
"Retry the 429s." Worsens.
**Strong senior answer**
The partner quota is a global, shared limit - more instances don't increase it. Honour `Retry-After`, apply a client-side rate limiter / token bucket shared across the fleet (per-instance = limit/instances approximation, or a distributed limiter), queue the calls (queue-based load leveling) so we pace outbound traffic, cache/batch requests, and negotiate higher quotas. Cap autoscale max instances to what the partner supports if the instances are bottlenecked by it. Trade-off: slower processing/latency vs 429 storms. Risk: per-instance quotas drift when instance counts change; consider central pacing (single consumer or Redis token bucket) at the cost of another dependency.
**Follow-up traps**
- "Priority between calls?" (Reserve quota for critical operations.)
- "Does 429 count as breaker failure?" (Usually pause, not open for long.)

### 17.3 [Medium] Cold start: where does it hurt and how do you mitigate across Azure compute options?
**What the interviewer is really testing**
Platform-specific awareness.
**Typical mid-level answer**
"Functions are slow the first time." Narrow.
**Strong senior answer**
Cold start shows as first-request latency (JIT, DI container build, config/secret fetch, connection setup) and in the herd after deploy/scale-out. Mitigations: App Service Always On and warm-up/slot swap pre-warming; Functions Premium plan pre-warmed/always-ready instances (or Flex Consumption always-ready), keep package small, avoid heavy static init; .NET ReadyToRun/Native AOT where compatible; lazy but bounded initialisation and cached tokens; container apps with min replicas > 0; Azure SQL serverless: set auto-pause off for latency-sensitive paths or handle the resume delay with retries (connection errors while resuming - a transient 40613 style error). Trade-off: always-on instances cost money. Risk: readiness reported before warm-up completes sends users to cold nodes.
**Follow-up traps**
- "Min replicas zero acceptable?" (For batch/non-latency-critical only.)
- "SQL serverless resume?" (Retry; first connection may fail.)

### 17.4 [Hard] Design fairness for a multi-tenant SaaS where one tenant's 50x burst must not hurt others.
**What the interviewer is really testing**
Multi-layer tenant isolation design.
**Typical mid-level answer**
"Rate limit per tenant." Single layer.
**Strong senior answer**
Layers: (1) Edge/API: per-tenant quotas/rate limits by plan (APIM `rate-limit-by-key` or in-app partitioned limiter) with 429+Retry-After; (2) Concurrency: per-tenant max in-flight so one tenant can't hold all threads/connections; (3) Queues: per-tenant throttling at consumption (fair scheduling across tenants - e.g. round-robin or weighted), separate bulk/interactive lanes; (4) Data: per-tenant resource governance (elastic pool min/max, Resource Governor/workload groups where available), indexes partitioned by TenantId, avoid cross-tenant table scans; (5) Cell-based architecture for large tenants/blast radius - route heavy tenants to dedicated cells. Trade-off: static quotas underuse capacity; allow burst credits up to a ceiling, and reduce under system pressure. Observability: per-tenant dashboards (cardinality careful) to find the noisy neighbour quickly. Risk: tenant identification occurring late in pipeline wastes resources - identify early.
**Follow-up traps**
- "How to detect a noisy tenant?" (Per-tenant metrics: RU/CPU/queries.)
- "Enterprise tenant wants guarantees?" (Dedicated resources/cell.)

### 17.5 [Scenario] Autoscaling flaps: scale out at 70% CPU, scale in at 50%, repeating every few minutes, causing latency spikes. Fix.
**What the interviewer is really testing**
Control-loop thinking.
**Typical mid-level answer**
"Change thresholds." Partial.
**Strong senior answer**
Flapping arises when scale-in removes capacity that pushes the remaining instances back over the scale-out threshold, with provisioning lag in both directions. Fixes: wider hysteresis gap, longer cooldown/stabilisation windows (scale-out fast, scale-in slow), scale-in step sizes small (one at a time), evaluate over longer windows, use a metric that doesn't oscillate with instance count (requests per instance, queue length), graceful shutdown + draining so scale-in doesn't drop requests, and set minimum instances to the baseline. Trade-off: slower scale-in costs a bit more. Verify by replaying production load profiles in a load test. Risk: autoscale max too high exceeding downstream capacity.
**Follow-up traps**
- "Why is scale-in dangerous?" (Dropped in-flight work, lost warm caches.)
- "Predictive scaling?" (For known daily patterns.)

**Red flags in answers**
- "Autoscale solves traffic spikes."
- No ceiling on scale-out relative to downstream capacity.
- Treating all tenants as equal load units.

**Cheat sheet**
- Scale lag = minutes; keep headroom + shed during lag.
- Scale on leading metrics (queue length, concurrency).
- Cap scale-out by DB/partner limits; share outbound rate limiters.
- Scale out fast, scale in slow, with hysteresis.
- Mitigate cold starts: warm-up, min instances, ReadyToRun, always-ready.
- Tenant fairness: quotas + concurrency caps + resource governance + cells.

---

## Topic 18: Dependency Management & Observability for Resilience

**Study notes**
Dependencies: third-party APIs, cloud services, internal services. For each, record: criticality, SLA/SLO, rate limits, failure modes, timeout/retry/breaker config, fallback, owner/contact, and the status/health page. Vendor SLAs give credits, not uptime; read exclusions.
Observability: **golden signals** (latency, traffic, errors, saturation); RED (rate, errors, duration) for services; USE (utilisation, saturation, errors) for resources. Distributed tracing (OpenTelemetry/Application Insights), correlation IDs, dependency telemetry. **Alert on symptoms** (user-facing SLO burn) and use cause-based metrics for diagnosis; page only on actionable, urgent symptoms.
```
User symptom alert (SLO burn)  -> page
Cause metrics (CPU, queue age, breaker open, DLQ) -> dashboards / tickets
```

### 18.1 [Easy] A third-party API is your hard dependency. What do you do before it fails?
**What the interviewer is really testing**
Proactive dependency risk management.
**Typical mid-level answer**
"Add retries." Reactive.
**Strong senior answer**
Document its criticality and the user journey that depends on it; read its SLA (and exclusions) and compute composite availability honestly; measure real behaviour via synthetic probes and dependency telemetry. Engineering: timeouts, breaker, bulkhead, rate-limit compliance (client-side limiter), idempotency, response caching where safe, async decoupling (queue + outbox) if the user doesn't need an immediate answer, a defined fallback (degrade, alternative vendor, manual process), and contract tests against its schema. Process: subscribe to its status page, know its support escalation, and review post-incident. Trade-off: effort vs the cost of an outage; scale to criticality. What fails when it fails: if no fallback exists, the user journey is down - and everyone should know that in advance.
**Follow-up traps**
- "Sandbox parity?" (Behaviour/limits may differ from prod.)
- "Who talks to the vendor in an incident?" (Named contact.)

### 18.2 [Follow-up] When is a fallback vendor worth it, and what's hard about it?
**What the interviewer is really testing**
Cost-benefit and hidden complexity.
**Typical mid-level answer**
"Always have a backup." Ignores cost.
**Strong senior answer**
Worth it when the business impact of an outage is high and the vendor's observed/contracted availability is below the target - e.g. payments, SMS/email, geocoding. Difficulties: functional differences (features, data formats, limits), keeping the second integration tested (dormant code rots - run a small percentage of traffic through it or synthetic tests), data-residency/compliance, fraud/consistency concerns (payments need idempotency across vendors; tokens are vendor-specific), cost/contract minimums, and failover logic complexity (when to switch - breaker-driven, automatic vs manual). Use an abstraction (port/adapter) designed around the common denominator, and a routing config under a feature flag. Trade-off: maintenance and cost vs availability. Risk: switching causes duplicates or inconsistent state - plan reconciliation.
**Follow-up traps**
- "Payment tokens portable?" (Typically not; plan around it.)
- "Active-active vendors?" (Distribute traffic to keep both warm.)

### 18.3 [Medium] Which four signals would you put on every service dashboard, and why those?
**What the interviewer is really testing**
Golden signals and saturation.
**Typical mid-level answer**
"CPU, memory, errors." Resource-only.
**Strong senior answer**
Latency (percentiles, split success vs failure - slow errors vs fast errors), traffic (request rate), errors (rate and ratio, by endpoint/dependency, including wrong-result signals), saturation (how full: thread pool queue length, connection pool usage, queue depth/age, DB DTU/CPU, memory). For dependencies, the same signals per outbound dependency plus resilience-specific metrics: retries, breaker state, bulkhead rejections, timeouts, fallback usage. Use histograms, not averages. Link to traces for drill-down. Trade-off: cardinality and cost; choose dimensions carefully. What fails when it fails: without saturation signals you see the outage only after errors start; with them, you see it coming.
**Follow-up traps**
- "Why separate success and failure latency?" (Fast failures hide slowness.)
- "Cardinality risk?" (Avoid unbounded labels such as userId.)

### 18.4 [Hard] Design alerting for a service with a 99.9% SLO so that on-call isn't flooded but real incidents page quickly.
**What the interviewer is really testing**
Symptom-based, burn-rate alerting.
**Typical mid-level answer**
"Alert when error rate > 1%." Noisy or slow.
**Strong senior answer**
Alert on SLO burn rate across multiple windows: page when error budget is burning fast (e.g. 14.4x over 1 h with a shorter confirmation window such as 5 min, meaning ~2% of the monthly budget gone), ticket for slow burn (e.g. 6x over 6 h, or 1x over 3 days) - the commonly used multi-window pattern from the SRE workbook. Symptom alerts page; cause alerts (CPU, breaker open, DLQ growth) go to dashboards/tickets unless they predict imminent user impact. Every page needs a runbook link and clear owner; review alert quality (precision and actionability) monthly and delete noisy ones. Add synthetic probes for low-traffic journeys. Trade-off: complexity of setup and the need for SLIs measured accurately. Risk: alerts depend on the same infrastructure being monitored - a monitoring outage needs a dead-man's switch.
**Follow-up traps**
- "Low-traffic service error ratio?" (Min request counts/synthetics.)
- "Is monitor itself monitored?" (Heartbeat alerts.)

### 18.5 [Scenario] Users report slowness but all dashboards show green. What's wrong with your observability?
**What the interviewer is really testing**
Observability gaps and gray failures.
**Typical mid-level answer**
"Check logs." Reactive.
**Strong senior answer**
Likely gaps: dashboards show server-side averages or health probes rather than user-experienced latency; missing percentile/tail metrics; no client-side/synthetic monitoring (RUM, real transactions); measuring at the wrong point (past the load balancer queue or CDN); sampled traces missing slow outliers; no per-instance or per-dependency breakdown to reveal one bad node; saturation metrics absent. Investigate by selecting slow user sessions and following traces end-to-end (correlation ID), compare instance-level latency, check queue wait time before the handler, and dependency latencies. Then fix: SLIs from the user's viewpoint, percentile histograms, tail-based sampling that keeps slow/error traces, synthetic journeys. Trade-off: telemetry cost. Risk: sampling policy dropping exactly the traces you need.
**Follow-up traps**
- "Head vs tail sampling?" (Tail keeps interesting traces.)
- "What shows queueing?" (Time-in-queue/request start delay.)

**Red flags in answers**
- No fallback or ownership for a critical vendor.
- Paging on CPU alone.
- Dashboards with averages and no saturation signals.

**Cheat sheet**
- Dependency register: criticality, SLA, limits, fallback, owner.
- Vendor SLA = credits, not uptime.
- Golden signals: latency, traffic, errors, saturation.
- Track resilience metrics: retries, breaker state, rejections, fallbacks.
- Page on SLO burn rate (multi-window); tickets for causes.
- Measure from the user's perspective; keep tail traces.

---

## Topic 19: Scenario Drills

**Study notes**
Use one structure under pressure: **Stabilise** (reduce load, restore service) -> **Diagnose** (timeline, signals, hypotheses) -> **Fix** (short-term) -> **Prevent** (systemic) -> **Verify** (test/experiment). Always ask: what is the user impact, what changed, what is saturated, what is amplifying?
```
Symptoms -> Is it load, dependency, deploy, or infra? -> Mitigate first, root-cause second
```

### 19.1 [Scenario] "A downstream API is slow, not down." Walk through what happens and what you do.
**What the interviewer is really testing**
The slow-dependency cascade and layered defences.
**Typical mid-level answer**
"Add a timeout and retry." Retries worsen slowness.
**Strong senior answer**
What happens: each call holds a thread/socket/pool connection for seconds; concurrency (rate x latency) climbs until our pools or thread pool saturate; requests queue; our own latency rises and upstream callers time out and retry; unrelated endpoints on the same process degrade - slow is worse than down. Immediate actions: confirm via dependency telemetry; reduce load - flip a kill switch/feature flag for non-critical use, shed low-priority traffic; ensure the breaker is opening on timeouts/slow calls; serve fallbacks (stale cache). Fixes: per-attempt timeout set from the dependency's p99 and within the caller budget; concurrency limit (bulkhead) so only N calls are in flight and the rest fail fast; retries limited, with jitter and budgets (or none for slow-timeouts); async decoupling if the call isn't needed synchronously; hedged requests only for safe idempotent reads. Prevention: fault-injection test adding 5 s latency, verifying checkout stays under SLO. Risk: timeouts causing abandoned but continuing work on the dependency - propagate cancellation.
**Follow-up traps**
- "Why not just raise the timeout?" (Holds resources longer; capacity collapse.)
- "Should you retry timeouts?" (Rarely against an overloaded dependency; budgeted at most.)

### 19.2 [Scenario] "A DB failover causes 5 minutes of errors." Diagnose and fix.
**What the interviewer is really testing**
Systemic recovery path, not just database.
**Typical mid-level answer**
"Azure SQL failovers are slow; open a ticket." Passive.
**Strong senior answer**
Build the timeline: when the DB became unavailable vs when the app stopped erroring. Platform failover is typically seconds, so look at client behaviour: retry duration (EF `EnableRetryOnFailure` default ~6 retries may expire before recovery), pool recovery (stale connections to old primary), connection string (listener endpoint for failover groups), command and connect timeouts, DNS caching, pod restarts caused by dependency-coupled liveness probes, breaker open duration, startup herd. Fixes accordingly: retry/backoff sized for the recovery window with jitter; breaker half-open probing; liveness independent of DB; listener endpoints; idempotent writes since commit outcome may be unknown; alert on connection errors; rehearse planned failover in staging and production windows. Trade-off: longer retries increase latency but reduce errors. Risk: retry storm after recovery - jitter and ramping.
**Follow-up traps**
- "Is any data lost?" (Within-region HA: no committed data loss; geo failover async: possible.)
- "How do you validate the fix?" (Trigger failover and measure the error window.)

### 19.3 [Scenario] "A retry storm took down the dependency." Post-incident, what changes?
**What the interviewer is really testing**
Metastable failure prevention.
**Typical mid-level answer**
"Lower the retry count." Insufficient.
**Strong senior answer**
Timeline: a trigger (deploy/slowdown) pushed latency above client timeouts; clients retried, doubling or tripling load; the dependency stayed down even after the trigger vanished. Immediate mitigation: cut traffic (disable callers via flag, shed at the edge, scale the dependency temporarily), restart only after load is removed, then ramp gradually. Changes: retries limited to one layer with budgets (e.g. ≤10% extra load), exponential backoff with jitter, honour `Retry-After`, circuit breakers with jittered half-open, server-side load shedding (fast 429/503 before expensive work), timeouts with cancellation so abandoned work stops, idempotency so retries are safe, queue buffering for async callers. Verify with a load test that simulates dependency slowdown and checks that load doesn't amplify (downstream-to-upstream request ratio stays near 1). Trade-off: more user-visible fast errors versus total outage. Risk: other callers with their own retry libraries - audit all clients.
**Follow-up traps**
- "How do you know when you've amplified?" (Ratio of downstream requests to user requests.)
- "Ramp-up approach?" (Gradual admission increase.)

### 19.4 [Scenario] "Region outage playbook": Describe the first hour.
**What the interviewer is really testing**
Leadership, decision criteria, verification.
**Typical mid-level answer**
"Flip to the secondary region." Too simple.
**Strong senior answer**
0-10 min: declare incident, roles (commander, comms, ops), confirm scope using Azure Service Health, synthetic probes and independent vantage points; check if our own change caused it. 10-20: decide on failover using pre-agreed criteria: Microsoft's ETA, RTO remaining, replication lag (RPO exposure), secondary readiness; freeze deploys. 20-40: execute the runbook - DB failover group failover (forced if needed), confirm writable; scale the secondary app tier; update traffic routing (Front Door origin priority/Traffic Manager); verify secrets (Key Vault replication), queues/storage in the secondary, identity availability. 40-60: validate with synthetic transactions and business SLIs, watch for capacity throttling in the secondary region, communicate status externally, start tracking lost writes for reconciliation. After: decide failback plan, postmortem. Trade-off: speed vs data loss. Risk: unrehearsed steps or missing permissions - mitigated by game days and automation.
**Follow-up traps**
- "What if Azure control plane is impaired?" (Rely on statically stable, pre-provisioned secondary; avoid operations needing control plane.)
- "How do you handle in-flight messages?" (Reconciliation and idempotent replay.)

### 19.5 [Scenario] "Design for 99.99%." You have a monolith on App Service + Azure SQL in one region at ~99.9% today. What's your plan?
**What the interviewer is really testing**
Staged, realistic architecture and honesty about costs.
**Typical mid-level answer**
"Go multi-region active-active." Jumps to the end.
**Strong senior answer**
First clarify: which user journey, how measured, and what's the real baseline from SLI data (what actually causes downtime - deploys, dependency failures, incidents, capacity?). Plan in stages: (1) Fix the biggest causes: zero-downtime deploys (slots/canary), timeouts/retries/breakers/bulkheads, graceful degradation, autoscale headroom, better alerts and runbooks. (2) Zone redundancy: App Service zone-redundant plan with ≥3 instances, Azure SQL zone-redundant tier, zone-redundant Redis/Service Bus; Front Door in front. (3) Remove optional dependencies from the critical path. (4) Multi-region: active-passive with failover group and warm standby if the RTO allows; active-active only if needed (data partitioning). (5) Automation: automated failover decisions where safe, chaos/game days, error-budget policy. Trade-off: ~4.4 min/month allowed - human response can't meet it, so automation and design must. Cost is a multiple, so negotiate scope: 99.99% for core journeys, 99.9% elsewhere. Risk: composite math - each critical dependency must exceed target; correlated failures (config, bad deploy) remain the top threat.
**Follow-up traps**
- "What's the largest source of downtime usually?" (Changes/deploys and human error.)
- "Is multi-region required for 99.99%?" (Not always; depends on risk of regional events and service SLAs - justify with data.)

### 19.6 [Scenario] Your boss asks for "no downtime ever" for a legacy app that depends on a single SQL VM with no HA. What do you say?
**What the interviewer is really testing**
Managing expectations with options.
**Typical mid-level answer**
"Yes, we'll make it highly available." No scoping.
**Strong senior answer**
Reframe "never" as a measurable target: ask for the business impact per hour and an acceptable RTO/RPO. Present a ladder: (a) quick wins - tested backups, monitoring, runbook (hours of RTO); (b) HA - Always On availability group or move to Azure SQL/Managed Instance with zone redundancy (minutes to seconds); (c) DR region with failover group (regional events). Show cost, effort, and the legacy app's constraints (connection retries, client libs). Decision: recommend the minimum step that meets stated RTO/RPO, document residual risk, and get sign-off. Trade-off: money and migration risk vs expected loss. Risk: promising uptime the app can't deliver because the app itself (no retry logic, in-memory state) can't tolerate failover.
**Follow-up traps**
- "Is HA the same as backup?" (No.)
- "App can't reconnect after failover?" (Fix client resilience or restart automation.)

**Red flags in answers**
- Jumping to a tool before understanding impact and constraints.
- No mitigate-first mindset.
- Promising absolute guarantees.

**Cheat sheet**
- Stabilise -> diagnose -> fix -> prevent -> verify.
- Slow dependency: timeouts + bulkhead + breaker + fallback + shed.
- DB failover: client behaviour is usually the long pole.
- Retry storm: cut load first, then fix amplification.
- Region outage: criteria-based decision, rehearsed runbook.
- 99.99%: remove dependencies, automate, zone/region redundancy, scope it.

---

## Topic 20: Resilience Review Checklist for Architecture Reviews

**Study notes**
Use this list when reviewing a design (answer each with evidence, not intent):
1. **Criticality & SLOs:** user journeys, SLIs/SLOs, error budget policy, composite availability math.
2. **Dependencies:** inventory, criticality, owners, SLAs, rate limits; each has timeout/retry/breaker/bulkhead/fallback defined.
3. **Timeouts & deadlines:** budget per call chain; cancellation propagated.
4. **Retries:** one layer; idempotency; backoff + jitter; budget.
5. **Overload:** bounded queues, rate limits, load shedding, admission control, backpressure.
6. **Isolation:** bulkheads, separate pools/queues, tenant fairness, blast radius/cells.
7. **State & data:** replication, backup + restore tests, RPO/RTO, consistency, outbox/inbox, migrations (expand/contract).
8. **Failure domains:** zones/regions; N-1 capacity; static stability; DR runbook and rehearsal.
9. **Deploy & change:** canary/blue-green, flags, rollback, kill switches.
10. **Health & lifecycle:** probes, graceful shutdown, warm-up.
11. **Observability:** golden signals, tracing, burn-rate alerts, dependency/resilience metrics, runbooks.
12. **Testing:** fault injection, load tests, game days, restore drills.
13. **Security & ops:** secrets/identity availability, quota/limits, cost of resilience, on-call readiness.

### 20.1 [Easy] You're handed an architecture diagram in a review. What are the first five questions you ask about resilience?
**What the interviewer is really testing**
Prioritised, high-signal questioning.
**Typical mid-level answer**
"Is it highly available? Is there a backup?" Generic.
**Strong senior answer**
1) What are the critical user journeys and target SLOs/RPO/RTO? (everything depends on this.) 2) What's on the critical path synchronously, and what happens when each box fails or is slow? 3) Where are the single points of failure (single region/zone, single instance, shared cache/DB, DNS, identity, a vendor)? 4) How does the system behave under overload and during dependency slowness (timeouts, retries, shedding, fallbacks)? 5) How do you deploy, roll back, observe and test failure (canary, alerts, game days, restore drills)? These cover availability, load, change, and detection - the usual sources of incidents. Trade-off: depth vs time; follow the answer's weakest part. What fails when it fails: unreviewed assumptions turn into incidents.
**Follow-up traps**
- "What if the team has no SLOs?" (Start by deriving target from business impact.)
- "Which box scares you most?" (The shared stateful one.)

### 20.2 [Follow-up] Pick the top three red flags you look for in a design doc.
**What the interviewer is really testing**
Pattern recognition.
**Typical mid-level answer**
"No scaling." Too narrow.
**Strong senior answer**
(1) Synchronous chains of many services with no timeout/fallback story - composite availability collapses and failures cascade. (2) Shared state with no isolation: one DB/cache/queue for everything, so a noisy workload or its failure takes all. (3) Resilience claims without evidence: "geo-redundant" with no tested failover/restore, retries with no idempotency, "autoscale handles it". Others: unbounded queues, health checks tied to dependencies, schema changes breaking N-1, no kill switches, no owner for DLQ. For each, I'd ask for a concrete failure walk-through rather than a feature name. Trade-off: some risks are acceptable if consciously accepted and documented with an owner.
**Follow-up traps**
- "How do you push back without blocking?" (Risk register, with decision owner.)
- "Which gaps are acceptable?" (Low-criticality features with accepted impact.)

### 20.3 [Medium] How do you run a "failure walk-through" (pre-mortem) on a design?
**What the interviewer is really testing**
A repeatable technique.
**Typical mid-level answer**
"Brainstorm what could go wrong." Unstructured.
**Strong senior answer**
For each component and each edge in the diagram ask: what if it's down, slow, returns wrong data, or is throttled? What does the caller do (timeout, retry, breaker, fallback)? What does the user see? How do we detect it (alert/SLI)? How do we recover (runbook, automation) and how long (RTO)? Record answers in a table (component x failure mode x impact x mitigation x detection x owner), prioritise by impact x likelihood, and convert gaps into tickets or accepted risks. Include change-related failures (bad deploy, bad config, expired certs, quota exhaustion) since they are the common causes. Trade-off: time-box (60-90 min) and focus on critical journeys. Risk: only brainstorming known failures - add incident history and cloud provider failure modes.
**Follow-up traps**
- "How do you cover unknown unknowns?" (Chaos experiments/game days.)
- "Who attends?" (Dev, ops, product, security.)

### 20.4 [Hard] The team says "we'll add resilience later." Respond and propose what to include now.
**What the interviewer is really testing**
Practical prioritisation of resilience as design, not polish.
**Typical mid-level answer**
"We should do it now." No specifics.
**Strong senior answer**
Some resilience is cheap now and expensive later because it shapes the design: idempotent operations and unique keys, correlation IDs, API contracts supporting versioning and async responses (202), outbox hooks, expand-contract migrations, stateless services, config-driven timeouts, bounded queues, health check semantics, and telemetry. Others can be phased: multi-region DR, advanced chaos, fallback vendors. Propose a minimum resilience baseline tied to criticality: every outbound call through a standard resilience handler (`AddStandardResilienceHandler` with reviewed values), standard health endpoints, SLO dashboards, runbooks, restore test before go-live. Trade-off: delivery speed vs a risk debt; document what's deferred and the trigger for doing it (traffic, SLA commitment). Risk: retrofitting idempotency/stateful assumptions often needs rewrites.
**Follow-up traps**
- "What's non-negotiable before go-live?" (Timeouts, idempotency on write paths, backups tested, alerts, rollback.)
- "How do you measure the debt?" (Risk register + error budget.)

### 20.5 [Tricky] "Every dependency has a circuit breaker, retries, and a fallback, so we're resilient." Challenge this claim.
**What the interviewer is really testing**
Skepticism about checkbox resilience.
**Typical mid-level answer**
"That sounds good." Accepts.
**Strong senior answer**
Presence of patterns isn't evidence of behaviour. Questions: were the values derived from data (timeouts vs p99, breaker thresholds) and do they nest correctly (inner < outer, one retry layer)? Are fallbacks independent of the failed component and tested, and do they stay within business correctness? Is there idempotency for retried writes? What happens at overload (shedding), during failover (connection recovery), during cold start, and under correlated failure (shared cache, region, identity, DNS, config)? Is the human side covered (alerts, runbooks, on-call, deploy safety)? Ask for test results: fault-injection runs, load test with dependency slowness, restore/failover drills. Trade-off: some effort is theatre if not tested. Decision: ask for evidence and a short list of experiments before approving.
**Follow-up traps**
- "What evidence is sufficient?" (Passing experiment results in prod-like env + monitoring.)
- "What's most commonly missed?" (Correlated/shared dependency and change failures.)

**Red flags in answers**
- Checklists answered with "yes" and no evidence or test.
- Reviewing only the happy path and infrastructure, ignoring deploy/config risk.
- No named owner for each accepted risk.

**Cheat sheet**
- Ask: SLOs, SPOFs, slow/down/wrong for every box, overload, change, detection.
- Failure table: component x mode x impact x mitigation x detection x owner.
- Evidence > intent: experiments, drills, load tests.
- Cheap-now/costly-later: idempotency, contracts, statelessness, telemetry.
- Standard resilience handler + reviewed values as a baseline.
- Document accepted risks with owners and revisit triggers.

