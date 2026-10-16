# S10 — Senior Interview Playbook (.NET / Azure / SQL Server)

## How to use this file
- This is the **capstone**. Read it after the technical guides (`DotNet_Interview_Guide`, `Azure_Interview_Guide`, S01–S09 in this folder). Those teach *what is true*; this teaches *how to present it at senior level*.
- Practise **out loud**. Every "Strong answer outline" is a skeleton: replace `[placeholders]` with **your own real** projects, numbers and lessons. Never invent metrics or stories.
- Use sections 1–3 for the first week (mindset + frameworks + traps), section 4–5 for loop strategy, section 6 for the final revision, section 7 for mocks with a partner.

## Table of Contents
1. [What separates mid vs senior vs staff](#1-what-separates-mid-vs-senior-vs-staff)
2. [Answer frameworks](#2-answer-frameworks)
3. [60 senior "trap" questions](#3-60-senior-trap-questions)
4. [Questions YOU should ask the interviewer](#4-questions-you-should-ask-the-interviewer)
5. [Interview-loop map & story preparation](#5-interview-loop-map--story-preparation)
6. [Rapid-revision sheet & 2-week schedule](#6-rapid-revision-sheet--2-week-schedule)
7. [Mock interview scripts](#7-mock-interview-scripts)

---

# 1. What separates mid vs senior vs staff

## 1.1 Dimension table

| Dimension | Mid | Senior | Staff |
|---|---|---|---|
| **Scope** | A feature / ticket / component | A service or a multi-sprint project, incl. its operations | Multiple teams / a domain / org-wide platform direction |
| **Depth** | Knows *how* to use the tool | Knows *why* it works that way and how it fails (internals, limits) | Knows when the tool itself is the wrong choice; shapes standards |
| **Trade-offs** | Gives "the best" way | Names 2–3 options, states costs, picks with reasons | Frames trade-offs in business terms (cost, time-to-market, risk, org structure) |
| **Failure thinking** | Happy path; handles exceptions locally | Designs for partial failure, retries, idempotency, timeouts, observability | Designs for blast-radius, recovery, organisational failure modes, game days |
| **Communication** | Answers the question asked | Structures answer, states assumptions, checks understanding | Tailors to audience (exec vs engineer), writes decision docs/RFCs |
| **Ownership** | Finishes assigned work | Owns outcomes: monitoring, on-call, tech debt, post-mortems | Owns problems no one assigned; sets up systems so others succeed |
| **Ambiguity** | Waits for clear requirements | Asks clarifying questions, proposes a plan with assumptions | Defines the problem itself; chooses what *not* to build |
| **Mentoring** | Learns from reviews | Reviews with teaching intent, unblocks juniors, raises team bar | Grows seniors, builds culture, sponsors people |

> Interview reality: the same question is scored on *signals*. Senior signals = **clarify, trade-offs, failure modes, operability, measurable validation**.

## 1.2 Twelve "mid answer vs senior answer" examples

### E1. "What is Dependency Injection?"
- **Mid:** "It's passing dependencies through the constructor instead of creating them with `new`. .NET has a built-in container with Singleton, Scoped, Transient."
- **Senior:** "DI is inversion of control for object creation: a class declares what it needs, and a composition root wires it. It gives testability and swappable implementations. In ASP.NET Core I register in one composition root, choose lifetimes deliberately — e.g. a `DbContext` is scoped; a singleton must never capture a scoped service (captive dependency; scope validation catches it in Development). I avoid service-locator (`IServiceProvider` injected everywhere) and constructor over-injection, which is a design smell signalling too many responsibilities. Trade-off: container indirection makes flow harder to trace, so I keep graphs shallow. [Example from my project: ____]."
- **Why senior wins:** connects concept → lifetimes pitfalls → smell → trade-off → real example.

### E2. "How do you handle retries?"
- **Mid:** "Wrap the call in try/catch and retry 3 times."
- **Senior:** "Retry only **transient** failures (timeouts, 429, 503, SQL transient error codes) and only when the operation is **idempotent** or protected by an idempotency key. Use exponential backoff **with jitter**, a max attempt/total time budget, honour `Retry-After`, and combine with timeouts and a circuit breaker so retries don't amplify an outage (retry storms). Layer retries in one place only (avoid retries-of-retries across SDK + Polly + gateway). Emit metrics on retry counts. In .NET I'd use `Microsoft.Extensions.Http.Resilience`/Polly; Azure SDKs have built-in retry policies to configure rather than wrap."
- **Why:** identifies what, when, how, and what can go wrong.

### E3. "How do you design an API?"
- **Mid:** "REST with controllers, GET/POST/PUT/DELETE, return JSON, add Swagger."
- **Senior:** "Start with consumers and use-cases, then resources and contracts. Decide: versioning strategy (and backward-compat rules: additive changes only), pagination (cursor vs offset), idempotency for POST (Idempotency-Key), consistent error model (`ProblemDetails`, RFC 7807/9457), authN/Z (OAuth2/Entra ID, scopes), rate limiting, validation, async operations for long-running work (202 + status resource), observability (correlation IDs, OpenTelemetry), and OpenAPI as the contract with contract tests. Then I'd explain trade-offs, e.g. REST vs gRPC vs GraphQL for this consumer set."
- **Why:** treats API as a long-lived product contract, not endpoints.

### E4. "How do you debug slow SQL?"
- **Mid:** "Look at the query and add an index."
- **Senior:** "Measure first: is it slow *always* or only sometimes? Capture the actual execution plan, `SET STATISTICS IO/TIME`, Query Store for regressions and wait stats for blocking/IO/CPU. Look for scans vs seeks, key lookups, estimate-vs-actual row mismatches (stale stats, parameter sniffing), implicit conversions, non-sargable predicates, missing/duplicated indexes. Fix the cause (rewrite, index with included columns, update stats, `OPTION (RECOMPILE)`/Query Store hint for sniffing), then validate with before/after metrics and check write-cost of new indexes. Also rule out the app side: N+1 from EF, chatty calls, connection pool starvation."
- **Why:** hypothesis-driven, measures, considers trade-offs of the fix.

### E5. "Monolith or microservices?"
- **Mid:** "Microservices are more scalable and modern."
- **Senior:** "It depends on team size, domain clarity and operational maturity. I default to a **modular monolith** with enforced boundaries and extract a service only when there's a concrete driver: independent scaling, independent deploy cadence, different tech/compliance needs, or team autonomy. Microservices cost: distributed data, network failures, observability, versioning, platform effort. If the team is [small] and the domain still shifting, a modular monolith wins; revisit with metrics."
- **Why:** refuses dogma, states drivers and costs.

### E6. "How do you ensure message processing is reliable?"
- **Mid:** "Use Service Bus so messages aren't lost."
- **Senior:** "Assume **at-least-once** delivery, so consumers must be **idempotent**. Use peek-lock with explicit complete after the work is durable; set lock duration/renewal, `MaxDeliveryCount` and a dead-letter queue with alerting and a replay process. Producer side: **transactional outbox** so DB write and message publish can't diverge. Consumer side: an inbox/dedup table keyed by MessageId. Handle poison messages, ordering needs (sessions), and monitor queue depth/age/DLQ count."
- **Why:** covers both ends of the pipe and the failure semantics.

### E7. "Explain async/await."
- **Mid:** "It makes code run asynchronously and faster."
- **Senior:** "Async frees the thread during I/O waits so the server handles more concurrent requests — it improves **scalability/throughput**, not single-request latency, and adds state-machine overhead. Avoid `.Result`/`.Wait()` (deadlocks, thread-pool starvation), use `ConfigureAwait(false)` in libraries, pass `CancellationToken`, and don't use `async void`. CPU-bound work needs `Task.Run`/parallelism, not async."
- **Why:** correct mental model and common production pitfalls.

### E7b. "How do you scale a web app on Azure?"
- **Mid:** "Scale up the App Service plan or add instances."
- **Senior:** "Find the bottleneck first (CPU, memory, DB, downstream). Make the app stateless (externalise sessions to Redis), scale out with autoscale rules on a meaningful metric, cache reads, offload work to queues, put a CDN/Front Door in front for static/global traffic, and scale the data tier (read replicas, partitioning, right tier). Load-test to validate and set budgets/alerts for cost."
- **Why:** bottleneck-first, system-wide.

### E8. "What do you do for a production outage?"
- **Mid:** "Find the bug and fix it."
- **Senior:** "**Mitigate first, diagnose second**: declare incident, assign roles, stop the bleeding (rollback, feature flag off, scale, failover), communicate status. Then root-cause using dashboards/traces/logs. Afterwards a blameless post-mortem with action items (detection gap, prevention, runbook). Track MTTD/MTTR."
- **Why:** prioritises customer impact and learning.

### E9. "How do you do transactions across services?"
- **Mid:** "Use distributed transactions."
- **Senior:** "2PC/MSDTC doesn't fit cloud/HTTP/messaging well (blocking, coupling, availability). I use **sagas** (orchestrated or choreographed) with compensating actions, outbox for atomic state+event, idempotent steps, and design the business flow to tolerate intermediate states. Where strong consistency is mandatory, I keep those entities in one service/DB boundary."
- **Why:** rejects the naive tool with reasons and offers an alternative with its costs.

### E10. "How do you secure your API?"
- **Mid:** "Add JWT authentication."
- **Senior:** "Defence in depth: validate tokens (issuer, audience, signature, expiry) from Entra ID, authorise via scopes/roles/policies (not just 'authenticated'), least privilege, managed identity + Key Vault instead of secrets, TLS everywhere, input validation, rate limiting/WAF, private endpoints, audit logging, dependency scanning, and threat-modelling (OWASP API Top 10: BOLA is most common — check object-level authorisation)."
- **Why:** layers, not a single control; names the real common vuln.

### E11. "How would you improve code quality in a team?"
- **Mid:** "Do code reviews and write unit tests."
- **Senior:** "Make the right thing easy: CI gates (build, tests, analysers, formatting), small PRs, review checklists focused on design/risk not style, testing pyramid with contract/integration tests, ADRs for decisions, tech-debt budget in each sprint, and metrics (change-failure rate, escaped defects). I lead by example and pair rather than police."
- **Why:** systemic and people-aware.

### E12. "Tell me about caching."
- **Mid:** "Use Redis to make things faster."
- **Senior:** "Cache what's expensive, read-heavy and tolerant of staleness. Decide the pattern (cache-aside typical), TTL and invalidation strategy, key design, stampede protection, serialisation cost, eviction, and failure behaviour (cache down ≠ outage). Measure hit ratio and verify the gain. Remember the two hard problems: invalidation and consistency."
- **Why:** treats caching as a consistency trade-off, not a free speed-up.

---

# 2. Answer frameworks

## 2.1 The core framework: Clarify → Options → Trade-offs → Decision → Risks → Validate
| Step | What you say | Example (design a notification service) |
|---|---|---|
| **Clarify** | Ask about scale, users, SLAs, constraints, existing stack | "Volume? Channels? Delivery guarantee? Latency target?" |
| **Options** | 2–3 viable approaches | "Direct send in-request; queue + worker; managed service (e.g. Event Grid/Notification Hubs/ACS)" |
| **Trade-offs** | Compare on 3–4 criteria that matter here | "Complexity, reliability, cost, time-to-deliver" |
| **Decision** | Pick one, say why, state assumptions | "Queue + worker because bursts and retries needed" |
| **Risks** | What could go wrong, what you'd monitor | "Duplicates, poison messages, provider throttling" |
| **Validate** | How you prove it works | "Load test, canary, DLQ alerts, SLO dashboards" |

## 2.2 The 60-second answer: Define → Example → Trade-off
1. **Define** (15 s): one crisp sentence, precise.
2. **Example** (25 s): real usage — "In [project] we …" (use your own).
3. **Trade-off** (20 s): when not to use it / cost / alternative.
4. Stop. Offer: "Happy to go deeper on X or Y."

## 2.3 Handling "it depends"
Never stop at "it depends". Say **on what**, then resolve:
> "It depends on three things: [consistency need], [scale], [team skill]. If [A], I'd choose X; if [B], Y. Given what you've told me, I'd go with X."

## 2.4 Thinking aloud
- Narrate: "I'm considering X because …; rejecting Y because …".
- Write assumptions on the board; number them.
- Pause deliberately (5–10 s) rather than rambling; say "Let me think about failure cases."
- Check-in every ~5–7 min: "Does this direction match what you're looking for?"

## 2.5 System design round: 45-minute flow
| Minutes | Phase | Output |
|---|---|---|
| 0–5 | Requirements & scope (functional, non-functional, scale) | Written list + numbers (rough QPS, data size) |
| 5–10 | Capacity estimates & API/data model sketch | Entities, key APIs |
| 10–20 | High-level architecture | Boxes/arrows, data flow, tech choices |
| 20–35 | Deep dive on 1–2 hardest areas (interviewer-led or your pick) | Consistency, scaling, failure handling |
| 35–42 | Reliability, security, observability, cost, deployment | Checklist coverage |
| 42–45 | Summary, trade-offs revisited, what you'd do next | Closing statement |

Tips: state non-goals; start simple then evolve; explicitly cover **failure modes**, **idempotency**, **data ownership**, **monitoring**.

## 2.6 Questions you don't know
1. Be honest: "I haven't used that directly."
2. Bridge to adjacent knowledge: "The closest I've worked with is X; it works by …".
3. Reason from first principles aloud: "I'd expect it to … because …".
4. Say how you'd find out: docs, experiment, who you'd ask.
5. Never bluff on specifics (numbers, API names). Bluffing is scored worse than not knowing.

## 2.7 Recovering from a mistake mid-interview
- Stop as soon as you notice: "I need to correct something I said."
- State the correct version, briefly why the earlier was wrong, and continue. No long apology.
- If the interviewer corrects you: acknowledge, adjust the design, and show the impact. Responding well is a **senior signal**.

## 2.8 Hostile or ambiguous interviewers
- **Pushback/challenging:** treat as a collaboration; "Good point — here's the trade-off I accepted…". Don't get defensive; don't cave without reason either. If they're right, concede and adapt.
- **Vague questions:** ask 2–3 clarifying questions, then state assumptions and proceed.
- **Silent/neutral interviewer:** self-check-in; ask "Would you like more depth here?"
- **Interrupting:** let them redirect; it usually means you're off-target.
- **Trick/gotcha:** answer the stated question, add the nuance once.

## 2.9 STAR+ (with metrics & lessons)
| Part | Content | Tips |
|---|---|---|
| **S**ituation | Context, scale, stakes (1–2 sentences) | `[team size, system, users, deadline]` |
| **T**ask | Your specific responsibility | Make "I" role clear |
| **A**ction | What *you* did — decisions, trade-offs, how you influenced | 60% of time here |
| **R**esult | Quantified outcome | `[metric before → after]` only real numbers |
| **+ Lessons** | What you learned / would do differently | Shows reflection |
| **+ Scale-up** | How you applied the lesson later | Shows growth |

If you have no exact metric, say "roughly [X]" or describe qualitative impact honestly.

---

# 3. 60 senior "trap" questions

Each entry: what the interviewer wants → trap answer → strong outline. Outlines use `[placeholders]` for your own experience.

### 1. Is microservices always better?
**What the interviewer wants**: Judgement over fashion; awareness of distributed-systems cost.
**Trap answer**: "Yes, it scales better and is the modern standard."
**Strong answer outline**:
- No. Benefits: independent deploy/scale, team autonomy, fault isolation, tech flexibility.
- Costs: network failures, distributed data/transactions, observability, versioning, ops/platform overhead, latency.
- Prefer modular monolith until a driver exists; extract by bounded context.
- Example: [project where I chose / avoided microservices and result].

### 2. Is async always faster?
**What the interviewer wants**: Understanding of throughput vs latency.
**Trap answer**: "Yes, async runs in parallel."
**Strong answer outline**:
- Async releases threads during I/O → higher concurrency/scalability.
- Single-request latency often slightly worse (state machine overhead).
- No benefit for CPU-bound work; can harm via thread-pool starvation if mixed with blocking.
- Sync-over-async deadlocks; use cancellation tokens; measure.

### 3. Is caching always good?
**What the interviewer wants**: Consistency and failure awareness.
**Trap answer**: "Caching speeds everything up."
**Strong answer outline**:
- Good for read-heavy, expensive, staleness-tolerant data.
- Costs: invalidation, stale data bugs, memory cost, stampede, cold-start, added failure mode.
- Define TTL/invalidation, stampede protection (locking/jitter), fallback if cache down.
- Measure hit ratio; don't cache before measuring a bottleneck.

### 4. Why not use 2PC (distributed transactions)?
**What the interviewer wants**: Understanding of coupling/availability.
**Trap answer**: "It's old/doesn't work."
**Strong answer outline**:
- 2PC needs all participants available, holds locks (blocking), coordinator is a failure point, many cloud services/HTTP/Service Bus-to-SQL combos don't support it.
- Reduces availability, couples services, poor latency.
- Alternatives: saga, outbox + idempotent consumers, redesign boundaries.
- Where it still fits: single trusted environment, few resources, low throughput.

### 5. "Singleton is bad" — agree?
**What the interviewer wants**: Nuance between pattern and DI lifetime.
**Trap answer**: "Singletons are an anti-pattern, never use them."
**Strong answer outline**:
- Hand-rolled static singleton = hidden global state, hard to test.
- DI singleton lifetime is fine for stateless/thread-safe shared services (`HttpClient` factory handlers, caches, configuration).
- Pitfalls: thread-safety, captive dependencies, memory leaks.
- Decide by statefulness and concurrency.

### 6. Do you need the Repository pattern over EF Core?
**What the interviewer wants**: Pragmatism, no cargo-culting.
**Trap answer**: "Always — for testability/abstraction" or "Never."
**Strong answer outline**:
- `DbContext` already is Unit of Work and `DbSet` a repository.
- Generic repository on top leaks `IQueryable` or hides EF features; adds ceremony.
- Use repositories/query objects for aggregate-focused domain (DDD), complex query encapsulation, or to isolate persistence.
- Testing: prefer integration tests with real DB (Testcontainers/SQL) over mocking.
- Decision based on domain complexity.

### 7. NoSQL vs SQL?
**What the interviewer wants**: Workload-driven choice.
**Trap answer**: "NoSQL scales, SQL doesn't."
**Strong answer outline**:
- Clarify access patterns, consistency, relationships, scale, query flexibility.
- SQL: relational integrity, ACID, ad-hoc queries, mature tooling; scales further than people think.
- NoSQL (e.g. Cosmos DB): known access patterns, horizontal scale, global distribution, flexible schema, partition-key design critical; cross-partition queries/transactions cost.
- Polyglot where justified; cost of operating two.

### 7b. How do you guarantee exactly-once processing?
**What the interviewer wants**: Honesty that it's effectively achieved, not magical.
**Trap answer**: "Service Bus guarantees exactly-once."
**Strong answer outline**:
- True exactly-once delivery is impossible in distributed systems generally; aim for **at-least-once delivery + idempotent processing = effectively-once**.
- Techniques: dedup key (MessageId/business key) in an inbox table with unique constraint, atomic with the state change; idempotent operations (upsert/set rather than increment); Service Bus duplicate detection (limited window) for producers.
- Outbox for the producing side.
- Test with redelivery/chaos.

### 8. What happens when a Service Bus message lock expires?
**What the interviewer wants**: Peek-lock mechanics.
**Trap answer**: "The message is lost / nothing happens."
**Strong answer outline**:
- Message becomes available again; delivery count increments; another (or same) consumer may process it → **duplicates**.
- Your original completion call fails with `MessageLockLostException`.
- Mitigate: right lock duration, auto lock renewal (processor `MaxAutoLockRenewalDuration`), shorter handlers, idempotency.
- Eventually exceeds `MaxDeliveryCount` → dead-letter. Monitor.

### 9. How do you handle backward compatibility?
**What the interviewer wants**: Evolution discipline.
**Trap answer**: "We just version the API v2."
**Strong answer outline**:
- Prefer additive, tolerant-reader changes; never remove/rename/repurpose fields without deprecation.
- Expand/contract (parallel change) for DB and contracts: add → dual-write/read → migrate → remove.
- Versioning (URL/header) when break unavoidable; deprecation policy with telemetry on who uses what.
- Messages: schema versioning, consumers deployed before producers.
- Consumer-driven contract tests.

### 10. How do you test in production?
**What the interviewer wants**: Safe-release practices, not recklessness.
**Trap answer**: "We don't" or "We just deploy and watch."
**Strong answer outline**:
- Feature flags, canary/blue-green/slot swaps, gradual rollout with automated rollback on SLO regression.
- Synthetic monitoring/health probes, shadow traffic, dark launches, A/B.
- Observability first; test data isolation; protect PII.
- Complements, not replaces, pre-prod tests.

### 11. What would you do on day 1 in a new codebase?
**What the interviewer wants**: Learning approach, humility, delivery.
**Trap answer**: "Start refactoring / rewrite it."
**Strong answer outline**:
- Get it running locally and in CI; read README/architecture/ADRs; know deploy path.
- Look at production: dashboards, alerts, incident history, on-call.
- Ship a small change early (fix a bug) to learn the pipeline.
- Meet people; ask what hurts; map hotspots (git churn, bug density).
- Hold opinions until evidence.

### 12. How do you decide to refactor?
**What the interviewer wants**: Business-justified engineering.
**Trap answer**: "When the code is ugly."
**Strong answer outline**:
- Trigger: code in high-churn area slowing delivery, causing incidents, blocking a feature.
- Ensure tests (characterisation) first; small incremental steps (strangler/branch by abstraction).
- Tie to measurable outcome; time-box; piggyback on feature work.
- Don't refactor stable code nobody touches.

### 13. What would you NOT do?
**What the interviewer wants**: Judgement, restraint, principles.
**Trap answer**: "I'd do everything well."
**Strong answer outline**:
- Examples to choose from (use real convictions): not rewrite from scratch; not adopt tech without an operational owner; not add microservices/Kafka for hypothetical scale; not optimise before measuring; not skip rollback plans; not take on work with no success metric.
- Explain reasoning and the exception where you'd reverse.
- Link to a lesson [story].

### 14. Tell me about your biggest architectural mistake.
**What the interviewer wants**: Self-awareness, learning, ownership.
**Trap answer**: A fake/trivial mistake or blaming others.
**Strong answer outline**:
- Real decision `[context, what I chose, why it seemed right]`.
- What went wrong `[symptom + impact, honest]`.
- How I detected and corrected it; what it cost.
- Lesson + a changed practice since (ADRs, spikes, load testing).
- Don't invent; choose a true one.

### 15. How do you keep a system simple?
**What the interviewer wants**: Restraint and long-term thinking.
**Trap answer**: "Use clean architecture and patterns."
**Strong answer outline**:
- Solve today's requirements with room to evolve (YAGNI); boring technology.
- Minimise moving parts, deployables, and dependencies; one way to do common things.
- Delete code; challenge new components with "what do we remove/own?".
- Explicit boundaries, small interfaces, ADRs; measure complexity via onboarding time and incident causes.

### 16. A DB deadlock is happening in production. What do you do?
**What the interviewer wants**: Systematic diagnosis & prevention.
**Trap answer**: "Add retry" or "increase timeout".
**Strong answer outline**:
- Capture deadlock graph (Extended Events `xml_deadlock_report` / system_health).
- Identify resources/queries, lock order, isolation level, missing indexes causing wide locks.
- Fixes: consistent access order, shorter transactions, proper indexes, RCSI/snapshot, avoid user interaction in transactions, reduce lock escalation.
- Retry (error 1205) as a safety net, idempotent. Validate via monitoring.

### 17. What is idempotency and how do you implement it?
**What the interviewer wants**: Practical mechanics.
**Trap answer**: "Calling something twice gives the same result" and stop.
**Strong answer outline**:
- Same request repeated has same effect as once — critical under retries/redelivery.
- Naturally idempotent: PUT, delete, upsert, set-state.
- Make POST idempotent: client `Idempotency-Key`, store key + response with unique constraint, return stored result; TTL on keys.
- Consumers: inbox table/dedup by MessageId in same transaction.
- Edge: concurrent duplicates → unique constraint/lock.

### 18. Explain eventual consistency to a business user.
**What the interviewer wants**: Communication to non-engineers.
**Trap answer**: Jargon (CAP, replication lag).
**Strong answer outline**:
- Analogy: bank cheque / email — "your update is accepted now, but other screens may take a few seconds to show it".
- State the business impact: what user may see, how long (e.g. `[N seconds]` SLO), and what we do to avoid harm (confirmation states, "processing" status).
- Why we accept: availability, scale, cost.
- Where we *don't* accept: payments balance checks.

### 19. How do you secure service-to-service communication?
**What the interviewer wants**: Identity-based, zero-trust approach.
**Trap answer**: "API keys / network firewall."
**Strong answer outline**:
- Managed identities + Entra ID tokens (client-credentials/OBO), validate audience/roles.
- Network: private endpoints/VNet integration, NSGs; mTLS in service meshes where needed.
- No secrets in config; Key Vault; rotation.
- Least privilege RBAC, audit logs; assume network compromise.

### 20. How do you handle a team that ignores standards?
**What the interviewer wants**: Influence without authority.
**Trap answer**: "Escalate to the manager" or "enforce rigidly."
**Strong answer outline**:
- Understand why: are standards unclear, outdated, or costly?
- Co-create/revisit them with the team; document in an ADR/guide.
- Automate (analysers, CI gates, templates) so compliance is free.
- Coach in reviews; celebrate wins; escalate only for persistent risk.
- Example: [my story].

### 21. When would you choose a queue vs. Event Grid vs. Event Hubs?
**What the interviewer wants**: Messaging fit.
**Trap answer**: "They're all messaging, any works."
**Strong answer outline**:
- Service Bus: reliable commands/work, transactions, sessions, DLQ.
- Event Grid: reactive event notifications (discrete events), push, fan-out.
- Event Hubs: high-throughput telemetry streams, partitions, replay by offset.
- Decide on semantics (command vs event), volume, ordering, retention.

### 22. Why not just scale up the database?
**What the interviewer wants**: Cost/limits thinking.
**Trap answer**: "Scale-up is bad."
**Strong answer outline**:
- Scale-up is simplest and often right first; has ceiling and cost cliffs; single point.
- First fix queries/indexes/caching; then read replicas, partitioning, sharding, CQRS read stores.
- Each step adds complexity; measure to justify.

### 23. Should every service have its own database?
**What the interviewer wants**: Data ownership vs pragmatism.
**Trap answer**: "Yes always" / "No, shared DB is fine."
**Strong answer outline**:
- Own data → no hidden coupling, independent evolution.
- Shared DB couples schema, deployments, locking.
- Pragmatic: separate schemas/permissions first; split physically when needed; cross-service reads via APIs/events/replicated read models.

### 24. How do you do zero-downtime deployments and DB migrations?
**What the interviewer wants**: Operational rigor.
**Trap answer**: "Deploy at night."
**Strong answer outline**:
- Slots/blue-green/rolling; health checks and readiness.
- Expand/contract migrations; backward-compatible schema; avoid long locks; online index ops.
- Feature flags; rollback plan (roll forward vs back); test migrations on prod-sized data.

### 25. How do you choose between REST, gRPC, and messaging?
**What the interviewer wants**: Fit-for-purpose.
**Trap answer**: "REST for everything."
**Strong answer outline**:
- REST: public/external, broad tooling, cacheable.
- gRPC: internal low-latency, strong contracts, streaming.
- Messaging: async decoupling, buffering, reliability.
- Consider consumers, latency, coupling, team skill.

### 26. What is the CAP theorem really telling you?
**What the interviewer wants**: Practical understanding.
**Trap answer**: "Pick any two of three."
**Strong answer outline**:
- Under a network **partition**, choose consistency or availability; otherwise trade latency vs consistency (PACELC).
- It's per-operation, not per-system; tunable (Cosmos DB consistency levels).
- Business decides which operations need which.

### 27. How do you design for high availability on Azure?
**What the interviewer wants**: Concrete, cost-aware thinking.
**Trap answer**: "Deploy to two regions."
**Strong answer outline**:
- Define SLO/RTO/RPO first; compute composite SLA.
- Availability zones, redundancy for each dependency, stateless compute, health probes, Front Door/Traffic Manager.
- Data replication (geo-replication/failover groups), tested failover.
- Cost trade-off; DR drills.

### 28. What is the outbox pattern and why?
**What the interviewer wants**: Dual-write problem knowledge.
**Trap answer**: "Save to DB then send message."
**Strong answer outline**:
- Dual write can fail between steps → inconsistency.
- Write business data + outbox row in one local transaction; a relay/poller (or CDC) publishes and marks sent.
- Gives at-least-once → consumers idempotent. Monitor outbox lag; clean up.

### 29. How do you handle a poison message?
**What the interviewer wants**: Resilience plumbing.
**Trap answer**: "Retry until it works."
**Strong answer outline**:
- Limit deliveries → DLQ; capture reason; alert.
- Distinguish transient vs permanent errors (don't retry validation failures).
- Triage/replay tool after fix; keep ordering implications in mind; avoid blocking the queue.

### 30. How do you design rate limiting / protect from a noisy neighbour?
**What the interviewer wants**: Multi-tenant protection.
**Trap answer**: "Add more servers."
**Strong answer outline**:
- Limits per client/tenant (token bucket/sliding window) at gateway (APIM) and in-app (`RateLimiter` middleware).
- Return 429 + `Retry-After`; bulkhead isolation, queues for load leveling, quotas.
- Observability per tenant.

### 31. A service is slow in production — how do you find why?
**What the interviewer wants**: Structured investigation.
**Trap answer**: "Check the logs / restart."
**Strong answer outline**:
- Define "slow" (p95/p99, since when, who). Check recent changes.
- Distributed traces to find the slow span; RED/USE metrics; Application Insights, dependency timings.
- Common: DB, downstream, thread-pool starvation, GC, cold start, connection pool exhaustion.
- Mitigate then fix and add alert.

### 32. How would you handle a memory leak in a .NET service?
**What the interviewer wants**: Tooling and root-cause depth.
**Trap answer**: "Restart the pod periodically."
**Strong answer outline**:
- Confirm via metrics (GC heap, working set); capture dumps (`dotnet-dump`, `dotnet-gcdump`, counters).
- Typical causes: static collections, event handlers not unsubscribed, `HttpClient` misuse, caches without bounds, LOH fragmentation.
- Fix, add bounds, regression test/alert; restart only as temporary mitigation.

### 33. How do you manage configuration and secrets?
**What the interviewer wants**: Security hygiene.
**Trap answer**: "appsettings.json and environment variables."
**Strong answer outline**:
- Non-secret config in App Configuration; secrets in Key Vault accessed through managed identity; no secrets in repo/pipelines logs.
- Rotation, feature flags, per-environment overrides; secret scanning in CI.

### 34. How do you decide build vs buy?
**What the interviewer wants**: Business thinking.
**Trap answer**: "Engineers can build anything."
**Strong answer outline**:
- Is it core differentiator? Total cost of ownership (maintenance, on-call), time-to-market, vendor lock-in, compliance, integration cost.
- Buy/manage commodity; build differentiators; revisit with exit strategy.

### 35. How do you estimate work under uncertainty?
**What the interviewer wants**: Honest planning.
**Trap answer**: "Give a single date confidently."
**Strong answer outline**:
- Break down, identify unknowns, spike to reduce risk, give ranges with confidence, compare to past similar work `[my reference]`.
- Re-forecast regularly, communicate early; negotiate scope not quality.

### 36. A stakeholder demands a risky shortcut for a deadline. What do you do?
**What the interviewer wants**: Pushback with pragmatism.
**Trap answer**: "Refuse" or "Just do it."
**Strong answer outline**:
- Understand the real goal and date's firmness; quantify the risk in business terms.
- Offer options: reduce scope, phased delivery, feature flag, shortcut with explicit tracked debt and a payback plan.
- Document the decision; ensure safeguards (monitoring, rollback).

### 37. How do you do code reviews as a senior?
**What the interviewer wants**: Leadership via review.
**Trap answer**: "Check for bugs and style."
**Strong answer outline**:
- Prioritise: correctness, design/boundaries, security, failure handling, test quality, readability; leave style to analysers.
- Ask questions, explain why, praise good work, label nits; fast turnaround.
- Keep PRs small; review in terms of outcomes.

### 38. What's wrong with `new HttpClient()` per request?
**What the interviewer wants**: Real .NET depth.
**Trap answer**: "It's fine, it's disposable."
**Strong answer outline**:
- Socket exhaustion (TIME_WAIT) and lost connection reuse; opposite problem: static client ignores DNS changes.
- Use `IHttpClientFactory` (pooled handlers with lifetimes) or `SocketsHttpHandler` with `PooledConnectionLifetime`.
- Add resilience handler, timeouts.

### 39. EF Core: when is it the wrong tool?
**What the interviewer wants**: Balanced ORM view.
**Trap answer**: "EF is slow, use Dapper" / "EF does everything."
**Strong answer outline**:
- Strong for CRUD/aggregates, migrations, change tracking.
- Pitfalls: N+1, over-fetching, tracking overhead, cartesian explosion, generated SQL surprises.
- Use `AsNoTracking`, projections, split queries; Dapper/raw SQL/stored procs for bulk/reporting/hot paths. Mix is fine.
- Check generated SQL & measure.

### 40. How do you handle distributed tracing and correlation?
**What the interviewer wants**: Observability practicality.
**Trap answer**: "Log everything."
**Strong answer outline**:
- W3C trace-context propagation through HTTP and message properties; OpenTelemetry; structured logs with trace IDs.
- Three pillars + SLO-based alerts; sampling strategy; cost control; PII redaction.

### 41. What are your SLIs/SLOs for a service you own?
**What the interviewer wants**: Reliability maturity.
**Trap answer**: "99.99% uptime."
**Strong answer outline**:
- SLIs from user perspective: availability, latency p95/p99, correctness/freshness.
- SLO derived from user needs `[my numbers]`, error budget guiding release pace; alert on burn rate not raw errors.

### 42. How do you prioritise tech debt?
**What the interviewer wants**: Business alignment.
**Trap answer**: "Fix it all someday."
**Strong answer outline**:
- Catalogue by impact × interest (cost of delay): incidents, velocity drag, security risk.
- Fixed capacity budget (e.g. `[X]%`) per sprint; attach to roadmap items; make debt visible in business terms.

### 43. How do you design multi-tenancy?
**What the interviewer wants**: Isolation trade-offs.
**Trap answer**: "Add a TenantId column."
**Strong answer outline**:
- Models: shared DB shared schema / schema-per-tenant / DB-per-tenant; trade isolation, cost, ops, noisy neighbours, compliance.
- Enforce tenant filters (EF global filter + SQL Row-Level Security), tenant-aware caching/limits, per-tenant observability, onboarding automation.

### 44. How do you handle long-running operations in an API?
**What the interviewer wants**: Async API patterns.
**Trap answer**: "Increase timeout."
**Strong answer outline**:
- Return 202 + operation URL; process via queue/worker (Durable Functions/background service); status polling or webhook/SignalR; idempotency; cancellation; result retention.

### 45. What's the problem with shared libraries across microservices?
**What the interviewer wants**: Coupling awareness.
**Trap answer**: "None, reuse is good."
**Strong answer outline**:
- Shared domain/model libs couple release cycles (distributed monolith), diamond versioning.
- OK for cross-cutting technical concerns (logging, auth) with semver and backward compatibility; contracts via schemas not shared DTO assemblies.
- Duplication is sometimes cheaper than coupling.

### 46. How do you ensure ordering of messages?
**What the interviewer wants**: Knowing the cost of ordering.
**Trap answer**: "Queues are FIFO so fine."
**Strong answer outline**:
- Ordering only guaranteed within a session/partition key; competing consumers break order.
- Use sessions (Service Bus) / partition key (Event Hubs); design idempotent/commutative handlers with version numbers; accept reduced parallelism; or avoid needing order.

### 47. How do you manage SQL Server index strategy?
**What the interviewer wants**: Beyond "add an index".
**Trap answer**: "Index every filtered column."
**Strong answer outline**:
- Base on workload (Query Store, DMVs for missing/unused indexes); clustered key choice (narrow, ever-increasing, unique); covering with INCLUDE; filtered indexes.
- Write overhead, storage, fragmentation/maintenance; remove duplicates; test impact.

### 48. Isolation levels: which do you use and why?
**What the interviewer wants**: Concurrency understanding.
**Trap answer**: "Default, never thought about it."
**Strong answer outline**:
- Default READ COMMITTED (locking) causes readers-block-writers; **RCSI** reduces blocking with row versioning (tempdb/PVR cost).
- Snapshot for consistent reads; Serializable for strict invariants with deadlock/blocking risk.
- Optimistic concurrency (rowversion) at app level; NOLOCK is dirty-read risk, not a fix.

### 49. How do you migrate a legacy system incrementally?
**What the interviewer wants**: Strangler fig/risk management.
**Trap answer**: "Big-bang rewrite."
**Strong answer outline**:
- Strangler fig: facade/proxy routes slices to new implementation; migrate by capability, highest value/lowest risk first.
- Characterisation tests, data sync strategy, parallel run/shadow comparison, rollback per slice, measurable milestones.

### 50. What would you do if you disagree with the architect/tech lead?
**What the interviewer wants**: Disagree-and-commit maturity.
**Trap answer**: "Do it my way" or "Stay silent."
**Strong answer outline**:
- Understand their constraints; present data/prototype/trade-offs privately first; propose a time-boxed experiment.
- If decision stands: disagree and commit, document concerns, set revisit criteria.
- Story: [my example].

### 51. How do you onboard and mentor a junior developer?
**What the interviewer wants**: Multiplier behaviour.
**Trap answer**: "Give them tickets and answer questions."
**Strong answer outline**:
- Structured first-week plan, starter tasks with increasing scope, pairing, review as teaching, explicit feedback cadence, delegating with safety nets; measure growth `[example]`.

### 52. How do you decide what to monitor and alert on?
**What the interviewer wants**: Alert quality.
**Trap answer**: "CPU > 80%."
**Strong answer outline**:
- Alert on symptoms (user impact: error rate, latency, queue age) not causes; every alert actionable with runbook; page vs ticket; reduce noise; review alerts post-incident.

### 53. What are the pitfalls of using Azure Functions for everything?
**What the interviewer wants**: Platform fit.
**Trap answer**: "Serverless solves scaling."
**Strong answer outline**:
- Cold starts, execution limits (plan-dependent), connection limits to downstream, concurrency/scale surprises hitting DB, harder local/test/observability, cost at sustained high load, function sprawl.
- Great for event-driven glue/bursty workloads; consider Container Apps/App Service for steady APIs.

### 54. How do you protect against SQL injection and over-posting?
**What the interviewer wants**: Security basics at depth.
**Trap answer**: "We use EF so we're safe."
**Strong answer outline**:
- Parameterise always (EF/Dapper params; careful with `FromSqlRaw` interpolation vs `FromSqlInterpolated`, dynamic SQL via `sp_executesql`).
- Over-posting: bind to DTOs, not entities; allow-list fields; least-privilege DB accounts; validation.

### 55. How do you approach performance tuning generally?
**What the interviewer wants**: Scientific method.
**Trap answer**: "Optimise the obvious hot code."
**Strong answer outline**:
- Define target; baseline; profile (traces, dotnet-trace, PerfView, SQL plans); fix biggest bottleneck; re-measure; stop when target met. Avoid premature optimisation; load test realistic data.

### 56. Tell me about a time you influenced without authority.
**What the interviewer wants**: Leadership signal.
**Trap answer**: Generic "I convinced them."
**Strong answer outline** (STAR+):
- `[Situation: cross-team problem]`, how I built the case with data/prototype, found allies, addressed their concerns, `[result with metric]`, lesson.

### 57. How do you handle a production incident you caused?
**What the interviewer wants**: Accountability under pressure.
**Trap answer**: Deflect blame or minimise.
**Strong answer outline**:
- Own it immediately, mitigate (rollback), communicate transparently, post-mortem focusing on system gaps (missing test, no canary), concrete fixes; `[my real story, what changed after]`.

### 58. How do you ensure data consistency between SQL and a cache/search index?
**What the interviewer wants**: Dual-store consistency.
**Trap answer**: "Update both in code."
**Strong answer outline**:
- Single source of truth; derive others via events/CDC/outbox; accept eventual consistency with defined lag; reconciliation/rebuild jobs; versioned updates to prevent stale overwrites; monitor drift.

### 59. How do you evaluate a new technology before adopting it?
**What the interviewer wants**: Disciplined adoption.
**Trap answer**: "It's popular / I want to learn it."
**Strong answer outline**:
- Problem first; criteria (fit, maturity, community, ops cost, security, skills, exit cost); time-boxed spike/PoC with success metrics; pilot in non-critical path; ADR; owner for operations.

### 60. Where do you see the limits of your own knowledge, and how do you grow?
**What the interviewer wants**: Honest self-assessment, learning loops.
**Trap answer**: "I have no weaknesses" or a disguised strength.
**Strong answer outline**:
- Name a real gap `[e.g. area]`, evidence of how I'm closing it (projects, reading, teaching), how I compensate (peers, reviews), and how I decide what to learn next (impact on team/product).

---

# 4. Questions YOU should ask the interviewer

Ask 3–5 per round, tailored. Listen for **specifics** (concrete examples, numbers, named practices) versus **vagueness** (buzzwords, "we're like a family", "it depends" with no example).

## 4.1 Engineering culture
| # | Question | Good answer reveals | Bad answer reveals |
|---|---|---|---|
| 1 | "How does a change get from commit to production? How long does it take?" | Automated CI/CD, small batches, hours/days | Manual gates, weeks, fear of release |
| 2 | "How are technical decisions made and documented?" | ADRs/RFCs, input from engineers | Top-down, tribal knowledge |
| 3 | "Tell me about a recent technical disagreement and how it resolved." | Healthy debate, data-driven | Avoidance or hierarchy wins |
| 4 | "What does your code review process look like?" | Fast turnaround, learning-oriented | Rubber-stamp or bottleneck |
| 5 | "How much time goes to new features vs maintenance/tech debt?" | Explicit budget | "No time for debt" |

## 4.2 On-call and incidents
| # | Question | Good | Bad |
|---|---|---|---|
| 6 | "What's the on-call rotation, and how many pages per week on average?" | Fair rotation, low noise, compensation/time off | Constant pages, same few heroes |
| 7 | "Walk me through your last significant incident." | Blameless, clear learnings and fixes | Blame, can't recall, repeating incidents |
| 8 | "Do you write post-mortems and track the action items?" | Yes, tracked to closure | Action items never done |
| 9 | "What are your SLOs, and who defines them?" | Defined, used for decisions | None or "100% uptime" |

## 4.3 Architecture
| # | Question | Good | Bad |
|---|---|---|---|
| 10 | "What's the current architecture, and what's the biggest architectural pain?" | Honest, specific | "It's all great" |
| 11 | "Where are you on monolith/microservices and why?" | Reasoned, evolving | Fashion-driven or accidental |
| 12 | "How do you handle data ownership and schema changes across teams?" | Clear boundaries, versioned contracts | Shared DB chaos |
| 13 | "What's the Azure footprint and how is infrastructure managed (IaC)?" | Bicep/Terraform, environments consistent | Click-ops, snowflake envs |
| 14 | "What is the legacy you're most worried about?" | Plan exists | Denial |

## 4.4 Quality
| # | Question | Good | Bad |
|---|---|---|---|
| 15 | "What's the test strategy (unit/integration/contract/e2e) and test confidence?" | Layered, trusted tests | Manual QA only / flaky suite ignored |
| 16 | "How do you measure quality (change failure rate, escaped defects)?" | Uses data | No measurement |
| 17 | "How do you do releases—feature flags, canaries, rollback?" | Safe progressive delivery | Big-bang, rollbacks painful |
| 18 | "What's your definition of done?" | Includes observability/docs/tests | "It works on my machine" |

## 4.5 Growth
| # | Question | Good | Bad |
|---|---|---|---|
| 19 | "What does the senior→staff path look like here? Examples of people who made it?" | Clear criteria, real examples | Vague, politics |
| 20 | "How do engineers learn—budget, time, mentoring?" | Concrete support | "Learn on your own time" |
| 21 | "What would success look like at 30/60/90 days for this role?" | Specific outcomes | Unclear expectations |
| 22 | "How is feedback given and how often?" | Regular, two-way | Annual only |

## 4.6 Team health
| # | Question | Good | Bad |
|---|---|---|---|
| 23 | "Why is this role open? What happened to the previous person?" | Growth/new headcount | High turnover, burnout |
| 24 | "What's the team's attrition over the past year?" | Low or explained | Evasive |
| 25 | "How does the team handle overload or conflicting priorities?" | Says no, negotiates scope | Heroics expected |
| 26 | "How is remote/hybrid collaboration handled?" | Documented norms | Inconsistent |

## 4.7 Product
| # | Question | Good | Bad |
|---|---|---|---|
| 27 | "How do engineers interact with product/customers?" | Direct, involved in problem shaping | Ticket-takers |
| 28 | "How are priorities set and how often do they change?" | Stable, transparent | Constant thrash |
| 29 | "What's the biggest risk to the product in the next year?" | Candid | Cannot answer |

## 4.8 Compensation & logistics
(Ask the recruiter/HM, not the technical peers.)
| # | Question | Note |
|---|---|---|
| 30 | "What's the level and band for this role, and how is leveling decided?" | Levels drive scope & pay |
| 31 | "What's the total compensation structure (base, bonus, equity/stock, benefits)?" | Compare total, not just base |
| 32 | "What are the review/promotion cycles?" | Timing matters |
| 33 | "What are the expectations on working hours, travel, and remote policy?" | Clarify in writing |
| 34 | "What's the process and timeline for the remaining steps?" | Plan your negotiation |

## 4.9 Red flags in a company/team
- Cannot describe how code reaches production, or releases are scary/manual.
- No on-call info, or on-call is "everyone, always, unpaid".
- Blame-oriented incident stories; repeated identical incidents.
- Interviewers disagree on what the role is; vague success criteria.
- High turnover; "we need a rockstar/ninja"; hero culture.
- No tests/observability; "we don't have time for quality."
- Unwillingness to discuss compensation range or leveling.
- Disrespect or disengagement during the interview itself — it's a preview.

---

# 5. Interview-loop map & story preparation

## 5.1 Loop map for senior .NET roles
| Stage | Purpose | What is assessed | Prep tactics |
|---|---|---|---|
| **Recruiter screen** (20–30 min) | Fit, logistics, motivation | Level fit, tech stack match, comp expectations, communication | 60-second pitch ("[years] in .NET/Azure/SQL; recently [domain]; looking for [scope]"); know your comp range; have 2 reasons for this company |
| **Coding** (45–60 min) | Problem solving & code quality | Clean C#, data structures, testing mindset, communication | Practise LINQ/collections/string/graph basics, async; clarify → brute force → optimise; write tests/edge cases; talk through complexity |
| **System design** (45–60 min) | Architecture judgement | Requirements, trade-offs, failure modes, scale | Use §2.1 and §2.5; practise designs: order processing, notification service, URL shortener, file upload pipeline, rate limiter |
| **Resume deep-dive** (45–60 min) | Authenticity, depth, ownership | Your actual decisions, mistakes, metrics | Prepare project one-pagers (§5.2); expect "why?" five levels deep; know every technology on your CV |
| **Behavioural** (45 min) | Leadership & collaboration | Conflict, failure, mentoring, influence | Story bank (§5.3); STAR+ |
| **Hiring manager** (30–60 min) | Team and expectations fit | Motivation, growth, approach to ownership | Research team/product; ask §4 questions; clarify 30/60/90 |
| **Bar-raiser / cross-team** (45–60 min) | Independent high-bar check | Consistency, depth, culture/value alignment, judgement | Expect hardest "trap" questions (§3) and ambiguous scenarios; be consistent with earlier answers; show learning |

General tactics: keep a notes file of questions asked per round; send thank-you notes; ask for the loop format beforehand.

## 5.2 Resume deep-dive: project one-pager template
Create one per major project (3–5 total). **Fill with real facts only.**

```
PROJECT: [name]            ROLE: [my role]            DURATION: [dates]   TEAM: [size, roles]
1. Problem / business context
   - Who were the users? What pain? Why now?  [ ... ]
2. Scale & constraints
   - Users/requests/data volume: [real numbers or "approx"]
   - SLAs/compliance/budget/deadline: [ ... ]
3. Architecture (draw it in 2 minutes)
   - Components, data stores, integrations, Azure services: [ ... ]
4. Key decisions (3–5)  — for each: options considered | choice | why | trade-off accepted
   - D1: [ ... ]
5. My specific contribution (use "I", be precise)
   - [ ... ]
6. Failures / incidents / things that went wrong
   - [what, impact, how detected, fix, lesson]
7. Results (metrics before → after)
   - [latency, cost, defects, delivery time, adoption]
8. What I'd do differently
   - [ ... ]
9. Likely drill-down questions & my answers
   - "Why not X?" [ ... ]   "How did you test/deploy/monitor?" [ ... ]   "How did it scale?" [ ... ]
10. People angle
   - Conflict handled, mentoring, stakeholders: [ ... ]
```
Rehearsal checks: Can you explain it in 2 minutes and in 15? Can you justify each tech choice? Can you name a real mistake?

## 5.3 Story bank matrix
Fill each cell with the **project/story ID** that supports it (one story can cover several themes; aim for ≥2 stories per theme so you don't repeat).

| Theme → / Story ↓ | Conflict | Failure | Impact | Ambiguity | Leadership | Technical depth | Production incident |
|---|---|---|---|---|---|---|---|
| Story A: [title] | | | | | | | |
| Story B: [title] | | | | | | | |
| Story C: [title] | | | | | | | |
| Story D: [title] | | | | | | | |
| Story E: [title] | | | | | | | |
| Story F: [title] | | | | | | | |

For each story record: **S/T** (1 line) · **Actions** (3 bullets, your decisions) · **Result** `[metric]` · **Lesson** · **Questions it answers** (e.g. "disagree with manager", "tight deadline", "mentored someone", "biggest failure", "improved performance").

---

# 6. Rapid-revision sheet & 2-week schedule

## 6.1 One-page senior concepts checklist
Tick when you can explain each in 60 seconds with an example and a trade-off.

**Microservices**
- [ ] Bounded contexts, service boundaries, data ownership
- [ ] Modular monolith vs microservices; strangler fig
- [ ] Sync (REST/gRPC) vs async (messaging) communication; API gateway/BFF
- [ ] Service discovery, config, versioning, contract testing
- [ ] Distributed monolith anti-pattern

**Distributed transactions & data**
- [ ] Why 2PC is avoided; saga (orchestration vs choreography); compensation
- [ ] Outbox/inbox; idempotency keys; at-least-once vs effectively-once
- [ ] Eventual consistency; CAP/PACELC; Cosmos DB consistency levels
- [ ] CQRS and event sourcing — when and when not
- [ ] Isolation levels, RCSI, deadlocks, optimistic concurrency

**Patterns**
- [ ] DI lifetimes & captive dependency; Options pattern; Mediator/pipeline behaviours
- [ ] Repository/UoW vs EF; specification; Factory/Strategy/Decorator in .NET
- [ ] Clean/Onion/Hexagonal; DDD aggregates, value objects

**Resilience**
- [ ] Timeouts, retries with jitter, circuit breaker, bulkhead, fallback; Polly/resilience pipelines
- [ ] Rate limiting, load shedding, backpressure, queue-based load leveling
- [ ] Service Bus: peek-lock, lock renewal, DLQ, sessions, duplicate detection
- [ ] Health checks, graceful shutdown, chaos testing

**Production**
- [ ] Observability: logs/metrics/traces, OpenTelemetry, App Insights, correlation
- [ ] SLI/SLO/error budget, alerting on symptoms, runbooks
- [ ] Incident response: mitigate → diagnose → communicate → post-mortem
- [ ] Deployments: slots, blue-green, canary, feature flags, expand/contract migrations
- [ ] .NET diagnostics: dotnet-counters/dump/trace, thread-pool starvation, memory leaks, GC

**Scalability & performance**
- [ ] Stateless services, horizontal scaling, autoscale signals
- [ ] Caching (cache-aside, invalidation, stampede), CDN, Redis
- [ ] SQL: execution plans, indexes, statistics, parameter sniffing, Query Store, partitioning
- [ ] Async/await correctness; N+1; connection pooling; batch/bulk operations

**Security**
- [ ] Entra ID/OAuth2/OIDC flows, JWT validation, scopes/roles/policies
- [ ] Managed identity, Key Vault, private endpoints, WAF
- [ ] OWASP API Top 10 (BOLA, mass assignment), SQL injection, secrets hygiene
- [ ] Multi-tenant isolation, RLS, auditing

**Leadership**
- [ ] Decision records, trade-off communication, estimation under uncertainty
- [ ] Mentoring, code review philosophy, standards via automation
- [ ] Influence without authority; disagree-and-commit; managing tech debt
- [ ] Blameless culture; stakeholder pushback

## 6.2 Two-week prep schedule
*Sibling guides referenced by folder/name; S01–S09 are the other files in this `Senior_Engineer_Guide` folder — open them by their file names (topics vary; use this plan with whichever file covers the listed theme).*

| Day | Focus | Material | Output |
|---|---|---|---|
| 1 | Baseline & plan | This file §1–2; self-assess with §6.1 checklist | Gap list |
| 2 | .NET internals refresh | `DotNet_Interview_Guide` (async, DI, memory/GC, EF Core) | 10 flashcards |
| 3 | Azure refresh | `Azure_Interview_Guide` (App Service, Functions, Service Bus, Storage, Entra ID) | Service-choice cheat-sheet |
| 4 | SQL Server depth | SQL-focused S0x file(s) in this folder; plans, indexes, isolation | Explain 5 plan scenarios aloud |
| 5 | Architecture & patterns | Microservices/patterns S0x files | 3 design sketches |
| 6 | Messaging & distributed data | Messaging/resilience S0x files; §3 Q4, 7b, 8, 17, 28, 29, 46 | Saga/outbox diagram from memory |
| 7 | **Mock #1 (system design)** — §7.1 | Partner or recorded solo | Rubric score + fixes |
| 8 | Production & observability | Ops/security S0x files; §3 Q16, 31, 32, 41, 52 | Incident story drafted |
| 9 | Security & scalability | Security S0x file; §3 Q19, 33, 43, 54 | Checklist cleared |
| 10 | Resume deep-dives | §5.2: finish 3 one-pagers | Rehearse each aloud (2 + 15 min) |
| 11 | Behavioural story bank | §5.3, STAR+ | ≥8 stories, matrix filled |
| 12 | **Mock #2 (incident)** + **Mock #3 (behavioural)** — §7.2, §7.3 | Partner | Rubric scores |
| 13 | Trap questions sweep | §3 — answer all 60 aloud (timed 60–90 s); redo weakest 15 | Weak list down to <5 |
| 14 | Light revision & logistics | §6.1, §4 questions chosen per company; rest | Calm, rested |

Daily habit: 30 min coding practice (C#/LINQ/data structures) in the first 10 days.

---

# 7. Mock interview scripts

Use a partner (or record yourself). Interviewer reads **Prompts**; candidate answers; interviewer checks **Expected points**; then scores with §7.4.

## 7.1 Mock 1 — System design: Order Processing (45 min)

**Setup prompt (interviewer):** "Design the order processing backend for an e-commerce platform on Azure. Customers place orders; the system must reserve inventory, take payment, and trigger fulfilment and notifications."

| Time | Interviewer prompt | Expected answer points |
|---|---|---|
| 0–5 | (Wait for clarifying questions. If none, note it.) When asked: "Peak ~[N] orders/min, global? Payment via third-party gateway. Inventory in a separate system. Orders must never be lost or double-charged." | Asks about scale, consistency, SLAs, payment provider, refunds, order states; states assumptions and non-goals |
| 5–10 | "Walk me through the API and the order lifecycle." | `POST /orders` with Idempotency-Key; 202/201 + order ID; states (Created → InventoryReserved → Paid → Fulfilling → Completed / Cancelled/Failed); status endpoint; ProblemDetails; auth |
| 10–20 | "Draw the high-level architecture." | API (App Service/Container Apps), Order service with own SQL DB, Service Bus topics/queues, Inventory/Payment/Fulfilment/Notification services (or modules), Redis cache for catalogue, API gateway (APIM/Front Door), observability |
| 20–30 | "How do you keep inventory, payment and order consistent without distributed transactions?" | Saga (orchestrated by order process manager, e.g. Durable Functions/state machine), compensations (release stock, refund), outbox for atomic state + event publish, idempotent consumers/inbox, timeouts for stuck steps |
| 25–30 | "Payment gateway times out — what happens?" | Don't assume failure; idempotency key at provider, query status/reconcile, retry with backoff + circuit breaker, pending state, reconciliation job |
| 30–35 | "A message is redelivered or the consumer crashes mid-process?" | Peek-lock, lock renewal, MaxDeliveryCount → DLQ, dedup by MessageId, alerts, replay tooling |
| 35–40 | "Flash sale: 20× traffic. What breaks first?" | Bottleneck-first: DB writes, inventory hot rows (contention → reservation queue/partitioning), queue-based load leveling, autoscale, rate limiting, cache, load tests, graceful degradation |
| 40–43 | "Security, observability, deployment?" | Entra ID/managed identity, Key Vault, PCI scope minimisation (tokenised payments), trace ID across messages, SLO dashboards, blue-green/canary, expand/contract migrations |
| 43–45 | "Summarise and tell me the top 3 risks." | Clear recap, honest risks (double charge, stuck sagas, inventory oversell), validation plan |

**Follow-up probes:** "Why not 2PC?" "Orchestration vs choreography?" "How do you handle partial refunds?" "How would you evolve the order schema without downtime?"

**Common mistakes to watch:** jumping to boxes before requirements; no idempotency; ignoring failure; over-engineering with Kafka/Kubernetes without justification; no data model.

## 7.2 Mock 2 — Production-incident deep dive (45 min)

**Setup prompt:** "Tell me about the most serious production incident you handled. Alternatively, work through this scenario: *Tuesday 14:05 — the checkout API's p99 latency jumps from [0.5 s] to [12 s] and error rate hits [8%]. SQL CPU is moderate. No deployment in the last 24 hours.*"

| Phase | Interviewer prompt | Expected answer points |
|---|---|---|
| Triage | "What do you do in the first 5 minutes?" | Acknowledge, declare incident/roles (incident commander, comms), check scope & impact (who/what/since when), check dashboards, recent changes/flags/config/cert/dependency status; **mitigate first** (rollback/scale/failover/disable feature) |
| Hypotheses | "What could cause this without a deploy?" | Downstream dependency slowdown, thread-pool starvation, connection pool exhaustion, DB blocking/plan regression (statistics/parameter sniffing), cache outage/stampede, traffic spike/bot, expired secret/cert, noisy neighbour, Azure platform incident |
| Investigation | "Traces show time spent in a DB call; CPU is moderate. Next?" | Check blocking (`sys.dm_exec_requests`, wait stats), Query Store plan change, long transaction/deadlocks, connection pool metrics, `sys.dm_tran_locks`; app-side: pool starvation/N+1; find head blocker |
| Mitigation | "You find a long-running report query blocking checkout writes. What now?" | Kill/offload the blocker cautiously, communicate, stop the report job, longer-term: RCSI/readable secondary for reports, query fix, resource governor |
| Communication | "How do you update stakeholders?" | Regular cadence, plain language, impact/ETA/next update, single source status page |
| Recovery | "How do you confirm it's resolved?" | SLIs back to baseline, error budget impact, watch period, close incident |
| Post-incident | "What goes in the post-mortem?" | Timeline, impact, root + contributing causes, what went well/poorly, detection gap, action items with owners/dates (alert on blocking, report isolation, runbook), blameless |
| Personal | "What would you do differently?" | Honest reflection `[my real story]` |

**Probes:** "How do you avoid alert fatigue?" "How did you decide rollback vs roll-forward?" "How did you know the fix worked?"

**Watch for:** random guessing instead of hypothesis testing; fixing before mitigating; no communication; blame.

## 7.3 Mock 3 — Leadership / behavioural (45 min)

| # | Interviewer prompt | Expected answer points |
|---|---|---|
| 1 | "Tell me about yourself and what you're looking for." | Concise narrative (≤2 min), scope growth, fit for role |
| 2 | "Describe a time you led a technical initiative across teams." | Clear goal, stakeholder mapping, influence, trade-offs, `[measured result]`, lessons |
| 3 | "Tell me about a significant disagreement with a peer or lead." | Understood other view, data-driven, outcome, relationship preserved, disagree-and-commit if lost |
| 4 | "Tell me about a failure." | Real failure, ownership, impact, corrective action, behavioural change |
| 5 | "Describe working with ambiguous requirements." | Clarified, assumptions, incremental delivery, validated with stakeholders |
| 6 | "How have you mentored or raised the bar of your team?" | Concrete mentee growth, process improvements, scalable (docs, tools, standards) |
| 7 | "Tell me about a time you said no / pushed back on a deadline." | Quantified risk, offered alternatives, outcome |
| 8 | "A time you improved a process or system with measurable impact." | Baseline → result, how you measured |
| 9 | "How do you handle underperformance in a teammate?" | Early, private, specific feedback, support, escalation path |
| 10 | "Why this company/role? Any questions for us?" | Researched, thoughtful questions from §4 |

**Follow-up probes (use on any story):** "What was *your* specific contribution?" "What would you do differently?" "What did the other person say?" "What metrics?" "What happened next?"

**Watch for:** "we" without "I"; no result; blame; rehearsed/generic stories; no reflection.

## 7.4 Scoring rubric (1–4 scale)
Score each dimension after each mock. 1 = Weak, 2 = Mid-level, 3 = Senior (hire bar), 4 = Exceptional/Staff-level.

| Dimension | 1 – Weak | 2 – Mid | 3 – Senior | 4 – Staff+ |
|---|---|---|---|---|
| **Problem framing / clarification** | Starts solving immediately | Asks a few questions | Clarifies requirements, states assumptions & non-goals | Reframes the problem, challenges requirements with business insight |
| **Technical depth** | Surface/incorrect | Correct but usage-level | Explains internals & pitfalls accurately | Teaches nuances, cross-domain connections |
| **Trade-off reasoning** | Single "best" solution | Mentions options | Compares options with explicit costs and decides | Frames in cost/risk/org terms; knows when to change course |
| **Failure & resilience thinking** | Ignores failure | Basic error handling | Idempotency, retries, partial failure, monitoring | Blast-radius, recovery, game days, SLO-driven |
| **Communication & structure** | Disorganised | Understandable | Structured, thinks aloud, checks in | Concise, adapts to audience, drives the conversation |
| **Ownership & leadership** | Task-focused | Owns own work | Owns outcomes, mentors, influences | Shapes direction across teams |
| **Handling ambiguity & feedback** | Defensive/stuck | Needs prompting | Proceeds with assumptions, adapts to hints | Uses pushback to improve design; leads resolution |
| **Evidence & honesty** | Vague/bluffing | Some examples | Specific real examples with metrics, admits gaps | Insightful lessons, changed practices |

**Interpreting totals (8 dimensions, max 32):**
- **≤ 16:** significant gaps — repeat the relevant sections and redo the mock.
- **17–23:** mid-level signal — focus on trade-offs and failure thinking.
- **24–28:** senior bar — polish stories and depth.
- **≥ 29:** strong senior / staff signal.

Record scores per mock in a table (Mock, Date, Dimension scores, Top 3 improvements) and retest the weakest dimension in the next mock.

---
*End of S10. Remember: use only your own real experiences; replace every `[placeholder]` before the interview.*