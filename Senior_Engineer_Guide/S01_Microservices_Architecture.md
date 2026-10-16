# S01 - Microservices Architecture (Senior Engineer Interview Guide)

**How to use:** Read the study notes, then answer each ladder question aloud *before* reading the strong answer. Seniors are judged on reasoning (clarify -> options -> trade-offs -> decision -> risks), not on buzzwords.
Stories marked **[Template]** are fill-in-the-blank frames: replace them with your own real experience; never present a template as fact.

## Table of Contents
1. When NOT to use microservices; monolith vs modular monolith vs microservices
2. Defining service boundaries (DDD, event storming, coupling/cohesion)
3. Distributed monolith: symptoms, fixes, and inheriting 40 services on a shared DB
4. Data ownership, database-per-service, shared DB, reporting
5. Sync vs async communication
6. API gateway, BFF, YARP, gateway anti-patterns
7. API and event contract versioning, compatibility, Pact
8. Service sizing and team topology (Conway)
9. Inter-service auth and trust boundaries
10. Service discovery, configuration, service mesh
11. Cross-service queries (composition, CQRS, replication)
12. Testing strategy in microservices
13. Deployment independence, CI/CD, shared libraries
14. Observability across services
15. Platform engineering and the cost of microservices
16. Migration from a monolith (strangler fig, ACL, branch by abstraction)
17. Multi-tenancy in microservices
18. .NET-specific stack decisions (Aspire, Dapr, MassTransit, YARP, gRPC, Container Apps/AKS)

---

## Topic 1: When NOT to use microservices; monolith vs modular monolith vs microservices

**Study notes**
- Microservices trade *in-process complexity* for *distributed-systems complexity*. You buy independent deployability and team autonomy; you pay with network failure, eventual consistency, ops overhead, and harder debugging.
- Modular monolith: one deployable, strict module boundaries (separate projects, internal APIs, per-module schema/ownership, enforced by architecture tests). Often the best default.
- Real drivers for microservices: independent scaling profiles, independent release cadence for many teams, fault isolation, differing tech/compliance needs, org size.
- Weak drivers: "Netflix does it", resume-driven design, "cleaner code" (that is a modularity problem, not a deployment one).

```
Team size / org complexity ->   small          medium            large, many teams
Domain clarity           ->     unclear        clearer           stable boundaries
Best starting point      ->     Monolith       Modular monolith  Microservices (selectively)
```

| Factor | Monolith | Modular monolith | Microservices |
|---|---|---|---|
| Deploy | 1 unit | 1 unit | N units |
| Boundaries enforced by | convention | compiler/tests | network |
| Transactions | ACID | ACID (per module discipline) | sagas / eventual |
| Ops cost | low | low | high |
| Team autonomy | low | medium | high |

### 1.1 [Easy] What are the main costs of microservices compared with a monolith?
**What the interviewer is really testing**
Whether you see microservices as a trade-off rather than an upgrade.

**Typical mid-level answer**
"Complexity and network latency." Correct but vague; it omits data consistency, operational burden, and team-cognitive cost.

**Strong senior answer**
The costs fall in four groups. (1) Distributed-systems failure modes: partial failure, timeouts, retries, duplicate messages, need for idempotency. (2) Data: no cross-service ACID transactions, so you need sagas/outbox and accept eventual consistency; joins become API calls or replicated read models. (3) Operations: per-service pipelines, monitoring, versioning, security, on-call, and often a platform team (AKS/Container Apps, APIM, Service Bus, tracing). (4) Human: debugging a request across 8 services, onboarding, and the need for contract discipline. The benefit - independent deploy and scale - only pays off if you have enough teams and change-rate to need it. I would phrase it to stakeholders as "we move complexity from code structure to infrastructure and process".

**Follow-up traps**
- *"Isn't latency the main issue?"* No; in-region calls are cheap. Failure handling and consistency are the real costs.
- *"Can't a monolith scale?"* Yes, horizontally behind a load balancer; scaling parts independently is the limitation.

**Red flags in answers**
- Treating microservices as "modern" and monoliths as "legacy".
- Listing only benefits.
- Not mentioning data consistency.

**Cheat sheet**
- Microservices = organizational scaling tool first, technical second.
- You pay: failure, consistency, ops, cognitive load.
- You get: independent deploy/scale/fault isolation.
- Default: modular monolith until a specific pain justifies extraction.

### 1.2 [Follow-up] Describe a modular monolith. How do you keep it from decaying into a ball of mud?
**What the interviewer is really testing**
Whether "modular" is more than folders.

**Typical mid-level answer**
"Separate projects per feature." Falls short: projects alone do not stop cross-module coupling to tables or internals.

**Strong senior answer**
A modular monolith has modules aligned to bounded contexts, each exposing a small public contract (interfaces/DTOs, or in-process events) and hiding everything else (`internal` types). Each module owns its data: separate schema (or at minimum tables no other module queries), with no cross-module joins or FKs. Communication is through the public API or domain events, ideally via an in-process bus so it can later become Service Bus without redesign. Enforcement matters: architecture tests (NetArchTest / ArchUnitNET), analyzers, and CI rules that fail on forbidden references; code owners per module. Trade-off: you still share a deployment, runtime, and one bad deploy can take everything, and scaling is coarse. The decision I would make: start here, keep modules extractable, and extract only when scaling/release/team pressure is measurable. Risk: discipline erodes under deadline pressure, so the checks must be automated.

**Follow-up traps**
- *"Same DB for all modules?"* Acceptable physically, but separate schemas and no cross-schema queries; it keeps extraction cheap.
- *"How do modules transact together?"* Prefer one transaction per module; use domain events + outbox to avoid coupling, accept in-process eventual consistency.

**Red flags in answers**
- "Modules" that all reference one shared `Entities` project.
- No automated enforcement.
- Cross-module SQL joins.

**Cheat sheet**
- Modules = bounded contexts with public contracts, hidden internals.
- Own data per module (schema level).
- Enforce with architecture tests in CI.
- Design for extractability, do not pre-extract.

```csharp
// NetArchTest sketch
var result = Types.InAssembly(typeof(OrdersModule).Assembly)
    .That().ResideInNamespace("Orders")
    .ShouldNot().HaveDependencyOn("Billing.Internal")
    .GetResult();
Assert.True(result.IsSuccessful);
```

### 1.3 [Medium] Give me a decision framework: how do you decide between the three styles for a new product?
**What the interviewer is really testing**
Structured, criteria-based thinking, not opinion.

**Typical mid-level answer**
"If it's big, use microservices." Falls short: size of code is not the criterion.

**Strong senior answer**
I clarify first: number of teams and expected growth, domain maturity (are boundaries known or guessed?), scaling profiles, compliance/isolation needs, deploy frequency requirements, and the ops capability we actually have. Then I score against criteria: independent deployability need (many teams blocked by each other?), heterogeneous scaling (one hot component?), fault isolation, tech heterogeneity, and platform maturity. If boundaries are uncertain, microservices are risky because wrong boundaries are expensive to fix across the network; a modular monolith lets us refactor with compiler support. If we have 1-2 teams, no platform, and fast iteration needs, I choose monolith or modular monolith. If we have 5+ teams colliding in one release train, stable subdomains, and a platform, I pick microservices for those subdomains only. Risk of deciding wrong is asymmetric: extracting later from a good modular monolith is cheaper than merging bad services. I document the decision in an ADR with the revisit triggers (e.g. "extract Billing if deploy contention > X per week or its scale diverges").

**Follow-up traps**
- *"What if you are sure you'll need microservices?"* Still start modular; "sure" is usually a guess about boundaries.
- *"Hybrid?"* Yes - a modular monolith core plus a few extracted services (e.g. image processing, notifications) is common and valid.

**Red flags in answers**
- No mention of team structure or domain clarity.
- Decision made on technology fashion.
- No revisit triggers.

**Cheat sheet**
- Criteria: teams, domain clarity, scaling, isolation, platform capability.
- Unclear boundaries -> stay in one deployable.
- Hybrid is normal.
- Record in ADR with triggers.

### 1.4 [Hard] When is a monolith the right *long-term* choice, and what would make you leave it?
**What the interviewer is really testing**
Willingness to defend simplicity and name concrete exit criteria.

**Typical mid-level answer**
"When the app is small." Falls short: no measurable triggers.

**Strong senior answer**
A monolith is right long-term when one team (or a few cooperating teams) owns the product, the load is manageable with horizontal scaling, and the domain is cohesive, e.g. a line-of-business app on Azure App Service + Azure SQL. Many successful systems stay there for years; the real risk is internal coupling, which a modular structure addresses. I would leave when I can show measurable pain: deployments blocked by cross-team coordination, a component whose resource profile forces overprovisioning the whole app (e.g. CPU-heavy report generation), blast-radius incidents from a low-value feature, or a regulatory need to isolate data. Even then I extract the *one* component with a clear seam, not everything. Trade-off: extraction adds ops cost, so the benefit must exceed it. I would measure first (deploy lead time, change failure rate, per-component resource use) rather than rely on intuition.

**Follow-up traps**
- *"Slow builds?"* Fix with build caching, test splitting, modularization before service splitting.
- *"Large codebase morale issue?"* Real but addressable with module ownership.

**Red flags in answers**
- "Monoliths can't scale."
- Extract everything at once.
- No metrics.

**Cheat sheet**
- Stay while one team/cohesive domain/simple scaling.
- Leave on measured pain, not aesthetics.
- Extract the smallest seam first.
- Use DORA metrics as evidence.

### 1.5 [Tricky] "Our modular monolith already has clean modules, so extracting to microservices is just changing the call from in-proc to HTTP." Agree?
**What the interviewer is really testing**
Awareness of fallacies of distributed computing.

**Typical mid-level answer**
"Yes, that's the benefit." Falls short: ignores semantics changes.

**Strong senior answer**
Partly agree: clean boundaries make extraction feasible, but the call changing from in-proc to network changes semantics. Latency and failure become possible on every call; you need timeouts, retries with backoff, circuit breakers (Polly / `Microsoft.Extensions.Http.Resilience`), and idempotency. Chatty module interactions that were free become a performance and availability problem, so the API granularity often needs redesign. Transactions spanning modules no longer work; you need sagas/outbox. Data: modules sharing a DB must be split, with data migration. Also deployment, auth, observability, and versioning all appear as new concerns. So extraction is a project, not a refactor. I would verify readiness with a checklist: no cross-module joins, coarse-grained contracts, async-friendly workflows, and idempotent handlers.

**Follow-up traps**
- *"What about gRPC to reduce cost?"* It reduces serialization/latency overhead, not failure semantics.
- *"Use distributed transactions (MSDTC)?"* Avoid; poor fit for cloud and couples availability.

**Red flags in answers**
- "Just swap the interface implementation."
- Ignoring partial failure.
- Keeping the shared DB.

**Cheat sheet**
- Network call != method call (8 fallacies).
- Redesign granularity and consistency.
- Add resilience + idempotency.
- Split the data, don't share it.

### 1.6 [Scenario] Leadership wants microservices "because competitors do". How do you respond?
**What the interviewer is really testing**
Influence without blocking; turning fashion into a business-outcome conversation.

**Typical mid-level answer**
"I'd explain why it's a bad idea" or "I'd just do it." Falls short: either confrontational or compliant, with no evidence.

**Strong senior answer**
I start by asking what outcome they want: faster releases, scaling, reliability, hiring, or competitive features? Then I translate: microservices are a means, and I would show whether our actual bottlenecks (lead time, incident causes, scaling costs) are solved by them. I present options with costs: (A) status quo + fixes, (B) modular monolith with CI/CD improvements, (C) selective extraction of one or two services, (D) full microservices - each with estimated effort, run cost, risk, and time to first value. I propose a pilot: extract one high-pain, well-bounded capability with success metrics (deploy frequency, lead time, incident rate), and a decision gate after e.g. a quarter. That respects their direction while limiting irreversible spend. I would be explicit about prerequisites: observability, CI/CD, ownership model, on-call. Risk: if they insist on a big-bang rewrite, I name the risks (parallel run, feature freeze, second-system effect) in writing and push for strangler-style incremental delivery.
**[Template story]** "In [situation], leadership asked for [X]; I measured [metric], proposed [pilot], and the outcome was [result]." Use only if real.

**Follow-up traps**
- *"They overrule you."* Disagree and commit, but keep the ADR and metrics so the decision can be revisited.
- *"Competitors really are faster."* Check whether the cause is architecture or process/product focus.

**Red flags in answers**
- Dismissing leadership as clueless.
- Agreeing without risk analysis.
- No measurable pilot.

**Cheat sheet**
- Ask for the outcome, not the technology.
- Offer options with cost/risk.
- Pilot with metrics and a gate.
- List prerequisites explicitly.
- Prefer incremental over big-bang.

---

## Topic 2: Defining service boundaries (DDD, event storming, coupling/cohesion)

**Study notes**
- Bounded context: a boundary within which a model and its language are consistent ("Customer" means different things in Sales vs Support).
- Subdomains: core (competitive advantage - invest), supporting, generic (buy/off-the-shelf).
- Event storming: workshop with domain experts; orange events -> commands -> aggregates -> policies -> contexts emerge.
- Good boundary: high cohesion inside, loose coupling outside, owned by one team, changes together.
- Smells: chatty services, shared DB, lock-step releases, "entity services" (CustomerService with CRUD only), nanoservices.

```
Sales context          Billing context       Shipping context
 Customer(name,prefs)   Payer(taxId,terms)    Recipient(address)
        \__ events: OrderPlaced -> InvoiceRequested -> ShipmentPlanned __/
```

### 2.1 [Easy] What is a bounded context and why is it a better unit than "entity per service"?
**What the interviewer is really testing**
Whether you understand boundaries follow business capability, not data tables.

**Typical mid-level answer**
"A bounded context is a part of the domain; services map to entities like User or Order." Falls short: entity services become CRUD wrappers requiring many calls for any use case.

**Strong senior answer**
A bounded context defines where a model and its ubiquitous language apply. The same word can mean different things in different contexts, so forcing one canonical `Customer` model creates coupling and endless negotiation. Services aligned to contexts own behavior and data for a business capability (Pricing, Ordering, Fulfilment), so most use cases complete within one service. Entity-per-service forces each use case to orchestrate several services, producing chatty calls and distributed transactions. I would test a boundary by asking: can a typical change be made by one team in one service? Do two services always change together? If yes, they are probably one context. Risk: contexts found wrongly at the start; mitigate by starting coarse and splitting later.

**Follow-up traps**
- *"Is a bounded context = a microservice?"* Not necessarily; a context may contain several deployables, or live in a modular monolith as one module.
- *"Aggregate vs context?"* Aggregate is a consistency boundary inside a context.

**Red flags in answers**
- Services named after nouns with only CRUD.
- One shared canonical data model.
- Ignoring language differences.

**Cheat sheet**
- Context = model + language boundary.
- Services follow capabilities, not tables.
- Context != necessarily one service.
- Start coarse.

### 2.2 [Follow-up] How would you actually discover boundaries in a legacy domain you don't know?
**What the interviewer is really testing**
A practical, collaborative method.

**Typical mid-level answer**
"Look at the database tables and group them." Falls short: legacy schemas reflect history, not domain.

**Strong senior answer**
I combine domain input with evidence from the code. Run event storming with domain experts and developers: list domain events chronologically, add commands, actors, aggregates, and policies; clusters where vocabulary shifts or ownership differs hint at boundaries. In parallel, analyze the codebase: git history for files that change together (temporal coupling), team ownership, call graphs and table access patterns (which modules write which tables), and incident hotspots. Compare both views; disagreement highlights hidden coupling. Classify subdomains (core/supporting/generic) to decide where to invest and what to buy. The output is a context map showing relationships (customer-supplier, conformist, ACL, shared kernel). I would validate with a thin vertical slice before committing. Risk: workshops with only developers produce technical boundaries; domain experts must be present.

**Follow-up traps**
- *"Not enough time for workshops?"* A half-day big-picture event storm pays for itself versus mis-splitting.
- *"Experts disagree on terms?"* That is a signal of two contexts.

**Red flags in answers**
- Derives boundaries from DB tables only.
- No domain experts involved.
- No context map.

**Cheat sheet**
- Event storming + code/history analysis.
- Language changes = boundary hint.
- Change-together files = same context.
- Produce a context map.

### 2.3 [Medium] You find Order and Inventory services call each other 12 times per checkout. What does that tell you and what do you do?
**What the interviewer is really testing**
Recognizing chatty-service and wrong-boundary smells.

**Typical mid-level answer**
"Add caching or use gRPC." Falls short: treats the symptom, not the boundary.

**Strong senior answer**
High-frequency bidirectional calls signal strong coupling: either the boundary is wrong, or the interaction is too fine-grained. First I clarify why each call is needed: reading stock levels, reserving, price checks? Options: (1) merge them if they change together and share invariants, (2) make the API coarse-grained (one `ReserveItems` command instead of per-item calls), (3) replicate the needed read data via events (Order keeps a local availability projection) if staleness is acceptable, (4) move the invariant to one owner and make the other react asynchronously. The decision depends on the invariant: if "never oversell" must be strict, the reservation belongs inside one consistency boundary. Trade-off between consistency and autonomy. Risk: replicated data goes stale; define acceptable staleness and compensation. I would measure p95 latency and call counts using distributed traces to confirm the diagnosis.

**Follow-up traps**
- *"Why not just batch the calls?"* Batching helps granularity but not bidirectional dependency.
- *"Is merging admitting failure?"* No; merging wrongly split services is correct course.

**Red flags in answers**
- Only performance tuning.
- Never considers merging.
- Ignores invariants.

**Cheat sheet**
- Chatty = coupling smell.
- Fix: merge, coarser API, replicate reads, move invariant.
- Strict invariants need one owner.
- Use traces to prove it.

### 2.4 [Hard] How do you handle a concept needed by many contexts, like Customer?
**What the interviewer is really testing**
Handling shared concepts without a canonical model or shared DB.

**Typical mid-level answer**
"A CustomerService that everyone calls." Falls short: becomes a god service and runtime dependency.

**Strong senior answer**
I split the concept by meaning. Each context keeps its own representation holding only what it needs (Billing: tax id and payment terms; Shipping: addresses) keyed by a shared identifier. One context is the system of record for the identity/master data (e.g. a Customer Profile context), and it publishes events (`CustomerRegistered`, `CustomerAddressChanged`) that others consume to maintain local copies. This avoids runtime calls on the hot path, so those services keep working when Profile is down. Trade-off: eventual consistency and duplicated storage; define ownership per attribute and what happens on stale data. For rarely-needed fields, a synchronous lookup with caching may be simpler. Avoid a shared `Customer` NuGet type used everywhere; that rebuilds a canonical model. Risk: privacy/GDPR - replicated PII must be deletable via events.

**Follow-up traps**
- *"Which service is source of truth for email?"* Exactly one; others are copies with defined refresh.
- *"Event carries full data or only ID?"* Carry what consumers need (event-carried state transfer) vs notification + callback; trade coupling vs payload size.

**Red flags in answers**
- One canonical entity shared via NuGet.
- A mandatory sync call on every request.
- Ignoring PII deletion.

**Cheat sheet**
- Same ID, different models per context.
- One owner per attribute.
- Replicate via events for hot paths.
- Plan PII erasure.

### 2.5 [Tricky] Is smaller always better? Someone proposes 60 "nano" services for a 6-person team.
**What the interviewer is really testing**
Understanding sizing as a function of team and cohesion.

**Typical mid-level answer**
"Smaller services are easier to maintain." Falls short: ignores per-service overhead.

**Strong senior answer**
Size should be driven by cohesion and team cognitive load, not line counts. Each service carries fixed overhead: pipeline, deployment config, monitoring, alerts, secrets, versioning, on-call surface. For 6 people and 60 services, each person owns 10 services; the overhead dominates and cross-service changes become the norm. I'd challenge with questions: how many deploy independently in practice? How many changes touch several services? I would consolidate toward services of around one team-ownable capability (perhaps 3-6 services for that team), or move the nano pieces into modules of a modular monolith. Rule of thumb: "a service small enough for one team to understand fully, large enough to own a business capability end to end." Trade-off: bigger services have more internal coupling, which modular design contains.

**Follow-up traps**
- *"What about serverless functions?"* Functions can be fine-grained deployables, but still need grouping and ownership; same overhead concern in a different form.
- *"The two-pizza team?"* A heuristic for team size, not for service count.

**Red flags in answers**
- Number of services as a success metric.
- No consideration of operational overhead.
- Size by LOC.

**Cheat sheet**
- Size by cohesion + team cognitive load.
- Per-service fixed overhead is real.
- Consolidate nano services.
- Services per team should be small single digits.

---

## Topic 3: Distributed monolith: symptoms, fixes, and inheriting 40 services on a shared DB

**Study notes**
- Distributed monolith = microservices cost without microservices benefit: services must be deployed/changed together.
- Symptoms: lock-step releases, shared database/tables, synchronous call chains (A->B->C->D), shared "common" library carrying domain types, one change touching many repos, a single outage cascading everywhere, integration-test-only confidence.
- Fixes: align boundaries to change, own data, async where possible, contracts + versioning, resilience patterns, or merge services back.

```
Healthy:     [A] -(event)-> [B]      each has own DB, deploys alone
Distributed: [A]-sync->[B]-sync->[C]  all share [ One DB ], release together
```

### 3.1 [Easy] What are the symptoms of a distributed monolith?
**What the interviewer is really testing**
Pattern recognition from experience.

**Typical mid-level answer**
"Services that depend on each other." Falls short: every system has dependencies; what matters is deployment and change coupling.

**Strong senior answer**
The defining test is independent deployability: can I release service A today without coordinating with B? If not, it is a distributed monolith. Symptoms I look for: releases scheduled as a "train"; a shared database with services reading each other's tables; a common library with domain models forcing simultaneous upgrades; long synchronous chains where availability is the product of every service's availability; one feature needing PRs in five repos; tests that require the whole environment to run; and breaking API changes made with no versioning. I would gather evidence: deploy history, cross-repo PR frequency, trace graphs showing call depth, DB access logs by service identity. The diagnosis then drives the fix instead of generic "we need better microservices".

**Follow-up traps**
- *"Is a shared library always bad?"* No - infrastructure/utility libs are fine; domain types and business rules in shared libs are the problem.
- *"Chain A->B->C is always wrong?"* Not always, but each hop lowers availability and adds latency; justify per case.

**Red flags in answers**
- No mention of deployability.
- Considers any dependency a symptom.
- Proposes more services as the fix.

**Cheat sheet**
- Test: deploy one service alone?
- Shared DB + shared domain lib + sync chains = red.
- Measure with deploy/PR/trace data.
- Availability multiplies down a chain.

### 3.2 [Follow-up] What are your options to fix it?
**What the interviewer is really testing**
Ability to choose between "fix" and "retreat".

**Typical mid-level answer**
"Refactor and add message queues." Falls short: async layered onto wrong boundaries just hides the coupling.

**Strong senior answer**
First decide per cluster whether the split was right. Where services always change together, merge them (consolidation is legitimate) into one service or module. Where boundaries are right but coupling is technical, fix the mechanisms: give each service its own data and replace table access with APIs or events; introduce versioned contracts and the expand-contract pattern so releases decouple; replace shared domain libraries with contracts or duplicated small models; convert non-essential synchronous calls to events; add timeouts, circuit breakers and bulkheads to stop cascades. Sequence by pain and risk: start with the most painful release-coupling pair, deliver one fix, measure deployment independence, then continue. Trade-off: merging reduces autonomy for some teams but cuts operational waste. Risk: attempting everything at once, creating a long-lived half-migrated state; keep each step shippable.

**Follow-up traps**
- *"Won't merging look like a failure?"* Judge by outcomes (lead time, incident rate).
- *"How to know it worked?"* Fraction of releases that go out alone; cross-service change count falls.

**Red flags in answers**
- Big-bang rewrite.
- Only technical fixes, no merge option.
- No success metric.

**Cheat sheet**
- Merge what changes together.
- Split data, version contracts, add resilience.
- Sync -> async only where it fits.
- Measure independent releases.

### 3.3 [Medium] Explain the availability math of a synchronous chain and what you'd do about it.
**What the interviewer is really testing**
Quantitative reasoning about coupling.

**Typical mid-level answer**
"More calls means slower." Falls short: misses availability and tail latency.

**Strong senior answer**
If a request needs services A, B, C, D in series, each 99.9% available, the composite is roughly 0.999^4 ~ 99.6%, so about 3-4x more downtime than a single service (assuming independent failures, which is optimistic). Tail latency compounds too: p99 of the chain is dominated by the slowest hop. Mitigations: remove hops (merge or coarsen), make non-critical dependencies optional with fallbacks (serve cached/default data), replicate needed reference data locally via events, use asynchronous workflows for steps that do not need an immediate answer, and apply timeouts, budgets, and circuit breakers so a failing dependency degrades quickly rather than exhausting threads. Also set per-dependency SLOs so we know which hop is the weak link. Trade-off: local replicas and fallbacks add complexity and staleness. I would choose per dependency: is it on the critical path for the user's goal?

**Follow-up traps**
- *"Retries fix it?"* Retries can amplify load (retry storms); use backoff, jitter, budgets.
- *"Parallel calls instead?"* Helps latency (max not sum) but availability still multiplies.

**Red flags in answers**
- Believes availability is additive.
- Unlimited retries.
- No fallback thinking.

**Cheat sheet**
- Serial availability = product.
- Cut hops, make deps optional.
- Timeouts + breakers + bulkheads.
- Retries need backoff/jitter/budget.

### 3.4 [Hard] How do you demonstrate to a skeptical manager that the architecture is hurting the business?
**What the interviewer is really testing**
Tying architecture to measurable outcomes.

**Typical mid-level answer**
"I'd show the diagram and say it's bad." Falls short: no evidence.

**Strong senior answer**
I use delivery and reliability metrics: DORA (lead time for change, deployment frequency, change failure rate, time to restore), plus counts of cross-team coordination per release, incident postmortems attributable to cascading failures, and engineering time spent on release coordination. I convert to money: hours per release x people x frequency, plus incident cost. I then show a before/after projection for a specific proposed change (e.g. decoupling two services shortens release from 2 weeks to daily). I propose a small experiment with a measurable target so the manager risks little. Trade-off: gathering data takes time; I use existing sources (pipeline history, Azure DevOps/GitHub PR data, incident tickets). Risk: metrics gaming; I use them for diagnosis, not individual evaluation.

**Follow-up traps**
- *"No data exists."* Start capturing now; meanwhile use a sample of recent releases.
- *"DORA is not everything."* Agreed; combine with developer-experience feedback.

**Red flags in answers**
- Opinion-only argument.
- Using metrics to blame teams.
- No cost translation.

**Cheat sheet**
- DORA metrics + incident data.
- Translate to hours/money.
- Propose a small, measurable experiment.
- Metrics for diagnosis, not blame.

### 3.5 [Scenario] You inherit 40 services that all use one shared SQL Server database. What do you do in the first 90 days?
**What the interviewer is really testing**
Prioritization, risk management, and pragmatism in a brownfield system.

**Typical mid-level answer**
"Split the database per service." Falls short: a big-bang split of 40 services is unrealistic and dangerous; no discovery, no sequencing.

**Strong senior answer**
I would not touch anything structural in the first weeks; I would learn. Discovery: inventory services, owners, deploy frequency, incident history, and critically *who reads/writes which tables* using Query Store, Extended Events, `sys.dm_exec_*` and login/identity per service (give each service its own SQL principal or managed identity if they share one, so access becomes visible). Build a table-to-service ownership matrix and find shared hot tables, cross-service writes, and cross-service joins. Then classify: services that are really one cluster (merge candidates), clear owners with a few foreign reads, and genuinely shared reference data. Quick wins with low risk: per-service schemas and permissions (deny cross-schema access progressively), views as a stable contract for foreign reads, a single writer per table. Medium term: for the highest pain cluster, move reads to API/events, then split the data, using expand-contract and dual-write avoidance (outbox/CDC). I would also stabilize operations first: backups, monitoring, ownership of the DB, performance baselines. Success metric: number of cross-owner table accesses falling, and independent releases rising. Risk: DB is a single point of failure and contention; keep that on the risk register. Not everything should be split; low-change, tightly related services may be consolidated.

**Follow-up traps**
- *"Can't do it without downtime?"* Use expand-contract, schema-per-service, views/synonyms and phased cutover.
- *"Management wants 'done' in 3 months."* Offer a staged roadmap with visible milestones, not a date for completion.

**Red flags in answers**
- Immediate big-bang split.
- No discovery of access patterns.
- Ignoring day-to-day operational stability.

**Cheat sheet**
- Learn first: table-to-service matrix.
- Separate identities and schemas to make coupling visible.
- Single writer per table; views as contracts.
- Split the most painful cluster first; consolidate others.
- Metrics: cross-owner accesses, independent releases.

**Red flags in answers (topic)**
- Treating distributed monolith as a tooling problem.
- Recommending more services as the fix.
- Not distinguishing "coupled and should merge" from "coupled badly".

**Cheat sheet (topic)**
- Independent deployability is the test.
- Merge, split data, version contracts, add resilience.
- Evidence beats opinion.
- Incremental, shippable steps.

---

## Topic 4: Data ownership, database-per-service, shared DB, reporting

**Study notes**
- Each service owns its data and exposes it only through its API/events. "Database per service" can be a separate database, or a separate schema with strict permissions (logical isolation) - the key is exclusive access.
- Shared DB anti-pattern: hidden coupling through schema, locks, and migrations; can't change tables safely; no independent deploy.
- Consequences of ownership: no cross-service transactions -> sagas, outbox, idempotency; cross-service reporting needs a different mechanism.
- Reporting options: analytical store fed by events/CDC (Azure Synapse/Fabric/Data Lake), read replicas for owner-approved views, API composition for small cases.

```
Svc A -> DB A --\
Svc B -> DB B ---+--(CDC / events)--> Lake / Warehouse --> Power BI
Svc C -> DB C --/
```

### 4.1 [Easy] Why database-per-service, and does it mean separate SQL Server instances?
**What the interviewer is really testing**
Understanding the principle vs the physical implementation.

**Typical mid-level answer**
"Each service has its own database for isolation." Doesn't explain the real reason or the cost spectrum.

**Strong senior answer**
The principle is exclusive ownership: only the owning service reads/writes its data, so the schema is an internal detail that can change without coordinating releases. Physical options form a spectrum: separate tables with permissions in one database (cheap, weakest isolation), separate schemas per service with distinct logins/identities (good pragmatic start), separate databases on a shared server or Azure SQL elastic pool (stronger isolation, still cost-efficient), or separate servers/different engines (strongest, most costly). The right choice depends on noisy-neighbor risk, compliance, scaling, and ops budget. For example, I might use one Azure SQL elastic pool with a database per service, and Cosmos DB for a service with document-shaped, high-scale access. Polyglot persistence is permitted, not required. The non-negotiable part is the access rule, enforced by permissions, not by convention.

**Follow-up traps**
- *"Same server, so is it 'really' separate?"* Logically yes if enforced; blast radius and resource contention still shared.
- *"Cost?"* Elastic pools keep per-database cost manageable.

**Red flags in answers**
- "Separate DB" but cross-database joins/linked servers.
- Convention-only enforcement.
- Always insisting on polyglot.

**Cheat sheet**
- Principle: exclusive access.
- Spectrum: schema -> database -> server.
- Enforce with identities/permissions.
- Choose isolation by risk and cost.

### 4.2 [Follow-up] Why is the shared database an anti-pattern, and when is sharing tolerable?
**What the interviewer is really testing**
Nuanced, not dogmatic.

**Typical mid-level answer**
"Because services get coupled." Right but shallow.

**Strong senior answer**
Shared tables turn the schema into a public API nobody versioned: any column change can break unknown consumers, so teams stop evolving it or must coordinate releases. It also couples runtime behavior (locks, deadlocks, plan regressions, resource contention) and hides invariants because any service can bypass another's business rules by writing directly. It breaks independent deployability. Tolerable cases: read-only shared reference data owned by one team and exposed as a stable contract (views, or a published dataset); a modular monolith where module boundaries are enforced; a transitional state during migration with an explicit exit plan. In those cases I use views as an interface layer, a single writer per table, and monitoring for foreign access. Trade-off is speed now vs flexibility later. I record tolerated sharing as tech debt with an owner and a date to revisit.

**Follow-up traps**
- *"Views as contract?"* They help, but become the API; version them and test consumers.
- *"Reference data like country codes?"* Fine as a shared read-only dataset/package or replicated; low change rate.

**Red flags in answers**
- "Never share anything, ever."
- Unaware of runtime coupling (locks, contention).
- No exit plan for tolerated sharing.

**Cheat sheet**
- Schema = unversioned API.
- Runtime coupling: locks, load, plans.
- Tolerable: read-only reference data, transition.
- Views + single writer + monitoring.

### 4.3 [Medium] An order must reserve stock and charge payment across three services. How do you keep data consistent?
**What the interviewer is really testing**
Saga, outbox, idempotency competence.

**Typical mid-level answer**
"Use distributed transactions or two-phase commit." Falls short: not viable across services/cloud and kills availability.

**Strong senior answer**
I accept eventual consistency and model the process as a saga: a sequence of local transactions with compensating actions (reserve stock -> charge -> confirm; on payment failure, release stock). Choose orchestration (a coordinator such as a MassTransit state machine, Durable Functions, or Dapr workflow; clearer flow, central place to see state) or choreography (services react to events; looser, but harder to understand as steps grow). For up to ~3-4 steps with simple logic I may use choreography; for complex or branching flows I prefer orchestration. Reliability requires the transactional outbox (write state change and outgoing message in the same local transaction; a dispatcher publishes) and idempotent consumers (inbox/dedup by message id), since Service Bus gives at-least-once delivery. Compensation must be designed as business actions (refund, release), and states like "pending" must be visible to the user. Risks: compensation can fail (needs retry and manual intervention path), and semantic locks/pending states leak to the UI. Monitor stuck sagas with timeouts and alerts.

**Follow-up traps**
- *"Exactly-once?"* Not realistically; at-least-once + idempotency yields effectively-once.
- *"Why not DTC?"* Couples availability, poor fit with PaaS and messaging brokers.

**Red flags in answers**
- 2PC across services.
- No outbox -> dual-write bug.
- No compensation failure handling.

**Cheat sheet**
- Saga = local txns + compensations.
- Orchestration for complex flows, choreography for simple.
- Outbox + idempotent consumer.
- Monitor stuck/timeout sagas.

```csharp
// Outbox idea: same transaction
await using var tx = await db.Database.BeginTransactionAsync();
db.Orders.Add(order);
db.Outbox.Add(new OutboxMessage(new OrderPlaced(order.Id)));
await db.SaveChangesAsync();
await tx.CommitAsync(); // background dispatcher publishes later
```

### 4.4 [Hard] Finance wants cross-service reports joining Orders, Billing and Customers. Services own their data. How?
**What the interviewer is really testing**
Separating operational from analytical workloads.

**Typical mid-level answer**
"Give the reporting tool read access to all databases." Falls short: recreates the shared-DB coupling and loads OLTP databases.

**Strong senior answer**
Reporting is a different workload with different consistency and shape needs, so I give it its own store. Options: (1) event/CDC-fed analytical store: services publish domain events, or change feeds (Cosmos change feed, SQL CDC/Change Tracking) land in a lake/warehouse (Microsoft Fabric, Synapse, or Azure SQL reporting DB), then Power BI sits on top. (2) Owner-published data products: each service exposes a versioned dataset/view for analytics, with contracts and ownership, the "data mesh"-flavored approach. (3) For tiny scale, API composition or a read replica per service, but only as an interim. I favor (1)/(2): decoupled, no load on OLTP, history preserved. Trade-offs: latency (minutes to hours), pipeline ownership, and schema drift. Governance matters: contracts for published data, data quality checks, PII handling and access control (Purview). Risk: the pipeline reaching into service databases directly creates hidden coupling, so go through published interfaces or CDC with agreed schemas.

**Follow-up traps**
- *"Real-time dashboards?"* Stream the events (Event Hubs + Stream Analytics/Fabric) for operational dashboards.
- *"Who owns the pipeline?"* The data/platform team owns infrastructure; service teams own their published contracts.

**Red flags in answers**
- Direct cross-DB queries/linked servers.
- Reports on OLTP primary.
- No ownership of contracts.

**Cheat sheet**
- Reporting gets its own store.
- Feed by events/CDC/published datasets.
- Accept latency; govern contracts and PII.
- Never reach into private tables.

### 4.5 [Tricky] Two services need to update the "same" record - e.g. Customer address in Sales and Shipping. Who wins?
**What the interviewer is really testing**
Ownership clarity and conflict handling.

**Typical mid-level answer**
"Last write wins." Falls short: silent data loss.

**Strong senior answer**
The scenario means ownership is unclear. I first resolve semantics: is it one fact or two? A shipping address on an order is a *snapshot* belonging to Shipping at the time of order; the profile address is master data owned by Customer Profile. Two facts, two owners, no conflict. If it is truly one fact, assign a single owner and have others submit change requests (commands) to that owner, receiving updates via events. If multiple writers are unavoidable (offline/multi-master), choose an explicit conflict strategy: optimistic concurrency with ETags/rowversion and reject-or-merge, versioned events, or CRDT-like merging for specific fields; "last write wins" only for data where loss is acceptable. Also record provenance and timestamps for audits. Trade-off: single owner adds a dependency; multi-writer adds complexity. I default to single ownership.

**Follow-up traps**
- *"Clock skew with LWW?"* Timestamps across services are unreliable; use version vectors or server-side sequence.
- *"What if owner is down?"* Queue the command; show pending state.

**Red flags in answers**
- "Last write wins" unconditionally.
- Never questions whether they are the same fact.
- Distributed locks as the first answer.

**Cheat sheet**
- Same fact -> one owner.
- Snapshot vs master data.
- Concurrency tokens (ETag/rowversion).
- LWW only if loss is OK.

**Red flags in answers (topic)**
- Cross-database joins presented as "separate DBs".
- Reporting directly against OLTP.
- 2PC across services.

**Cheat sheet (topic)**
- Own your data exclusively; enforce via identities.
- Sagas + outbox + idempotency.
- Analytics via events/CDC to a separate store.
- Resolve ownership before concurrency.

---

## Topic 5: Sync vs async communication (REST/gRPC vs messaging)

**Study notes**
- Synchronous (HTTP/REST, gRPC): caller waits; simple, immediate response; creates temporal coupling (callee must be up now).
- Asynchronous (Azure Service Bus queues/topics, Event Hubs, Event Grid): decoupled in time; resilience and buffering; adds eventual consistency, ordering/duplicate handling, harder tracing.
- Commands (one receiver, "do this") vs events (facts, many subscribers, "this happened").
- Choose per interaction, not per system.

| Need | Prefer |
|---|---|
| Query needing an immediate answer to render UI | Sync (REST/gRPC) |
| Trigger work that can complete later | Async command via queue |
| Notify many parties of a fact | Pub/sub event (Service Bus topic / Event Grid) |
| High-volume telemetry/streams | Event Hubs |
| Low-latency internal service calls | gRPC |

### 5.1 [Easy] Explain temporal coupling and why it matters.
**What the interviewer is really testing**
Core concept understanding.

**Typical mid-level answer**
"Services depend on each other at the same time." Needs the availability consequence.

**Strong senior answer**
Temporal coupling means caller and callee must both be available at the same moment for the operation to succeed. With synchronous calls, the caller's availability is bounded by the callee's, and load spikes pass straight through. A queue breaks it: the producer succeeds once the message is durably accepted, and the consumer processes when it can, which absorbs spikes (load leveling) and outages. The price is that the caller no longer gets an immediate result, so the user experience and consistency model must handle "accepted, processing". Not all coupling is bad: if a user needs the answer now (price quote), the dependency is inherent and I handle it with timeouts, caching, and fallbacks. So the design question is whether the business process truly requires immediacy.

**Follow-up traps**
- *"Does a queue remove all coupling?"* No - message schema coupling remains; also coupling on semantics and ordering.
- *"Does async mean faster?"* Not per request; it improves resilience and throughput, not latency.

**Red flags in answers**
- "Async is always better."
- Ignoring UX for eventual results.
- Confusing temporal with schema coupling.

**Cheat sheet**
- Temporal coupling = both must be up now.
- Queues buffer and decouple in time.
- Schema coupling remains.
- Async improves resilience, not latency.

### 5.2 [Follow-up] REST vs gRPC for internal calls - how do you choose?
**What the interviewer is really testing**
Practical tech trade-offs.

**Typical mid-level answer**
"gRPC is faster, so use it." Overlooks tooling, browser support, and ecosystem.

**Strong senior answer**
gRPC (HTTP/2, Protobuf) offers compact payloads, strongly typed contracts generated from `.proto`, streaming, and good performance, which suits internal service-to-service calls with high volume or latency sensitivity, and polyglot teams. REST/JSON suits public APIs, browser clients, simple debugging (curl), caching with HTTP semantics, and broad tooling like APIM. gRPC cons: browsers need gRPC-Web, harder ad hoc inspection, load balancing on HTTP/2 needs care (long-lived connections pin to one pod, so use client-side balancing or a proxy aware of it), and contract discipline for proto evolution (field numbers never reused). My default: REST for externally-facing and low-volume internal; gRPC where measured performance, streaming, or strict typed contracts justify it. I would benchmark before choosing for performance reasons; JSON over HTTP in .NET is fast enough for most.

**Follow-up traps**
- *"gRPC in Kubernetes/AKS balancing?"* Use headless services with client-side LB, or a mesh/ingress that understands HTTP/2.
- *"Protobuf evolution?"* Add fields, never renumber/reuse; mark removed fields `reserved`.

**Red flags in answers**
- Choosing on hype/benchmarks not measured.
- Exposing gRPC directly to browsers.
- Unaware of LB pitfalls.

**Cheat sheet**
- gRPC: internal, typed, streaming, performance.
- REST: public, browser, tooling, caching.
- Watch HTTP/2 load balancing.
- Never reuse proto field numbers.

### 5.3 [Medium] Describe how you choose between a command, an event and a request/response for a new interaction.
**What the interviewer is really testing**
Per-interaction decision making.

**Typical mid-level answer**
"Use events for everything to be decoupled." Falls short: event-only systems become untraceable.

**Strong senior answer**
I ask: does the sender need a result now? Does it know who must act? Who owns the outcome? If the caller needs data immediately to proceed, request/response. If the caller wants something done by a specific service but not necessarily now, a command on a queue (one consumer, sender knows the target - some coupling, clear intent). If something happened and others may care (OrderPlaced), an event on a topic; the publisher does not know subscribers, which gives extensibility but makes flows implicit. Events should be named in the past tense, carry stable business facts, and be owned by the publisher. For multi-step business processes I consider an orchestrator for visibility, rather than long choreographed event chains. Risks of events: hidden dependencies, version management, and debugging; mitigate with an event catalog (AsyncAPI), correlation IDs and tracing.

**Follow-up traps**
- *"Event vs command naming?"* Command: imperative `ReserveStock`; event: past `StockReserved`.
- *"Request/reply over queues?"* Possible (MassTransit request client) but use sparingly.

**Red flags in answers**
- Events that are secretly commands ("PleaseCreateInvoice").
- No catalog of events.
- Events with no owner.

**Cheat sheet**
- Need answer now -> request/response.
- Do-this -> command (one consumer).
- It-happened -> event (many).
- Catalog events; trace with correlation IDs.

### 5.4 [Hard] A queue consumer processes the same message twice and charges a customer twice. Prevent it.
**What the interviewer is really testing**
Idempotency and delivery semantics.

**Typical mid-level answer**
"Use exactly-once delivery." Falls short: not available end-to-end.

**Strong senior answer**
Service Bus is at-least-once (peek-lock can redeliver after lock expiry or failure), so the consumer must be idempotent. Techniques: an idempotency key (message id or business key such as OrderId + attempt) with an inbox table recording processed keys inside the same local transaction as the side effect; unique constraints that make duplicates fail harmlessly; naturally idempotent operations (set status = Paid rather than increment); and passing the idempotency key to the payment provider (Stripe-style idempotency keys) so retries do not double charge. Service Bus duplicate detection on send helps for producer retries but only within a window and is not enough alone. Also set lock duration/renewal appropriately for long processing and use dead-letter queues with an alert and replay process for poison messages. Trade-offs: inbox table growth (TTL cleanup) and extra write per message. Test by deliberately delivering duplicates in component tests.

**Follow-up traps**
- *"Ordering?"* Use sessions for per-key ordering; global ordering is rarely worth it.
- *"Poison messages?"* Max delivery count -> DLQ -> alert -> fix -> replay.

**Red flags in answers**
- Believes in exactly-once.
- Idempotency only via in-memory cache.
- No DLQ handling.

**Cheat sheet**
- At-least-once + idempotent handler.
- Inbox table / unique constraint / natural idempotency.
- Pass idempotency keys to downstream APIs.
- DLQ with alert and replay.

### 5.5 [Tricky] Your team converted all REST calls to Service Bus messages to "be resilient", and now support complains they can't tell whether orders succeeded. What went wrong?
**What the interviewer is really testing**
Recognizing over-application of async and missing observability.

**Typical mid-level answer**
"Add logging." Falls short: the design lacks visible state.

**Strong senior answer**
They applied async uniformly rather than per interaction. Where users need an immediate outcome, converting to fire-and-forget pushes the answer into the void; there is no status, no correlation between request and result, and failures disappear into DLQs. Fixes: reintroduce synchronous APIs where the user needs a response (or return `202 Accepted` with a status resource/`Location` header and optionally push updates via SignalR/webhook); model process state explicitly (Order status: Pending/Confirmed/Failed) so support can query it; propagate correlation IDs through messages and use distributed tracing; monitor queue depth, age of oldest message, and DLQ counts with alerts; provide an operations view. Decision going forward: use async for work that tolerates delay and for events, sync for queries and interactive decisions. Trade-off: some temporal coupling returns in return for clarity. Lesson: resilience mechanisms should not remove observability.

**Follow-up traps**
- *"202 Accepted pattern?"* Return a status URL; client polls or subscribes.
- *"Timeouts for async?"* Define process deadlines and escalation.

**Red flags in answers**
- Uniform rule for all calls.
- No process-state model.
- Alerts only on errors, not on stuck messages.

**Cheat sheet**
- Choose per interaction.
- 202 + status resource for long ops.
- Explicit process states.
- Alert on queue age and DLQ.

**Red flags in answers (topic)**
- "Async for everything" or "REST for everything".
- Assuming exactly-once delivery.
- No correlation across messages.

**Cheat sheet (topic)**
- Sync for immediate answers; async for deferrable work and facts.
- Commands one receiver; events many.
- Idempotent consumers, DLQ discipline.
- Trace and expose process state.

---

## Topic 6: API gateway, BFF, YARP, gateway anti-patterns

**Study notes**
- API gateway: single entry point for routing, TLS termination, authN, rate limiting, caching, request aggregation (limited). Azure options: API Management (APIM), Application Gateway/Front Door (L7 routing/WAF), YARP (.NET reverse proxy library), Ocelot.
- BFF (Backend for Frontend): one backend per client type (web, mobile) shaped to that client's needs, owned by the front-end team.
- Gateway anti-patterns: business logic in the gateway, one giant shared gateway owned by a central team (bottleneck), orchestrating long workflows, becoming a single point of failure, gateway-to-service chatty coupling.

```
Web ---> BFF-Web ---\
Mobile -> BFF-Mobile --> [Services]
Partners -> APIM (public API, quotas, keys) --> [Services]
```

### 6.1 [Easy] What does an API gateway do, and what should it NOT do?
**What the interviewer is really testing**
Knowing the boundary between cross-cutting concerns and business logic.

**Typical mid-level answer**
"Routes requests to services." Incomplete; no warning on logic creep.

**Strong senior answer**
A gateway handles cross-cutting edge concerns: routing, TLS termination, authentication/token validation, rate limiting and quotas, WAF, request/response transformation, caching, and basic aggregation. It should not hold business rules, domain orchestration, or data ownership; that makes it a coupled, high-change component every team must touch. Also avoid making it the only protection: services still authorize their own operations (defense in depth). I pick the product by need: APIM for external/partner APIs needing developer portal, subscriptions, policies; Front Door/App Gateway for global/L7 load balancing and WAF; YARP when I want a code-first, customizable proxy in .NET inside our own platform. Risk: gateway as a single point of failure and a team bottleneck; mitigate with HA deployment, config-as-code, and delegated route ownership.

**Follow-up traps**
- *"Can the gateway aggregate calls?"* Light composition yes; anything with logic belongs in a BFF or service.
- *"Gateway authorizes?"* Coarse checks only; fine-grained authorization in services.

**Red flags in answers**
- Business rules in APIM policies.
- Gateway as only security layer.
- Single gateway owned by one overloaded team.

**Cheat sheet**
- Edge concerns only.
- Defense in depth: services still authorize.
- APIM external, YARP custom internal, Front Door global.
- Config as code, HA.

### 6.2 [Follow-up] When do you introduce a BFF instead of a single gateway?
**What the interviewer is really testing**
Understanding client-driven API shaping.

**Typical mid-level answer**
"When there are mobile and web clients." Surface-level.

**Strong senior answer**
A BFF is worthwhile when different clients need materially different data shapes, call patterns, or release cadences, and one generic API forces over-fetching, many round trips, or compromise changes. Each BFF is owned by the client team, so it evolves at the client's pace and aggregates/reshapes data from downstream services, keeping domain services free of UI concerns. It also isolates failure and security concerns, e.g. mobile gets compact payloads; web gets session-cookie handling (a BFF can keep tokens server-side, a recommended pattern for SPAs). Costs: more deployables, duplicated aggregation logic across BFFs, and risk of logic creeping from services into BFFs. I would not add a BFF for a single client with a simple API; GraphQL or a thin gateway can suffice. Decision rule: add one when the client team is repeatedly blocked or contorting by a shared API.

**Follow-up traps**
- *"GraphQL instead?"* Valid alternative (client picks fields); adds query-complexity and caching concerns.
- *"Where does shared logic go?"* In downstream services, not copied between BFFs.

**Red flags in answers**
- BFF per screen.
- Domain rules inside the BFF.
- BFF owned by backend team who ignores the client's needs.

**Cheat sheet**
- BFF per client type, owned by that client's team.
- Reshape/aggregate; no domain rules.
- Optional for single client.
- SPA BFF can hold tokens server-side.

### 6.3 [Medium] You are asked to choose between APIM and YARP for the platform edge. Walk through it.
**What the interviewer is really testing**
Build vs buy reasoning with Azure/.NET specifics.

**Typical mid-level answer**
"APIM because it's Azure's gateway." Falls short: no criteria.

**Strong senior answer**
I clarify consumers: external partners/public developers, or internal clients only? Needs: subscriptions/keys, quotas, developer portal, analytics, policy language, versioning, SLA, compliance. APIM provides these as managed PaaS with tiers (consumption/standard/premium; cost and latency differ, and VNet integration needs higher tiers); policies are XML, which is awkward to test and tempting for logic creep. YARP is a library: full code control in C#, custom routing, transforms, load balancing, easy local dev and unit tests, and cheap, but you own HA, scaling, security patching, and build the missing features (portal, quotas). For internal edge with a strong .NET team, YARP is attractive; for externally monetized/partner APIs, APIM. They also combine: Front Door/WAF -> APIM -> services, or APIM for external and YARP as an internal BFF. I decide by who must operate it and the feature set required, not by preference. Risk with APIM: cost and policy sprawl; with YARP: homegrown platform maintenance.

**Follow-up traps**
- *"Cost?"* Compare APIM tier cost vs engineer time maintaining YARP; neither is free.
- *"Versioning in the gateway?"* APIM supports version sets; YARP via routes/headers.

**Red flags in answers**
- Picks by familiarity only.
- Ignores operating cost.
- Puts logic in XML policies.

**Cheat sheet**
- External/partner -> APIM.
- Internal/custom -> YARP.
- Combine layers if needed.
- Count operating cost, not just license.

### 6.4 [Hard] Our gateway team is a bottleneck: every new route needs their ticket. Fix it.
**What the interviewer is really testing**
Organizational as well as technical solutions.

**Typical mid-level answer**
"Hire more people for the gateway team." Falls short.

**Strong senior answer**
The central gateway has become a coordination bottleneck, which conflicts with team autonomy. Options: self-service via config-as-code where each service team owns its route definition in its own repo (e.g. APIM products/APIs via Bicep/APIOPS, YARP config fragments aggregated at deploy time) and a CI pipeline validates against guardrails (naming, auth required, rate limits); the platform team owns the gateway runtime and policies, not individual routes. Another option is to split gateways by domain/BFF so ownership aligns with teams. I would define a golden path with default policies and automated checks so most routes need no review. Trade-off: less central control; mitigate with policy-as-code (Azure Policy/linting) and audits. Measure time-to-expose-a-route before and after.

**Follow-up traps**
- *"Security risk of self-service?"* Guardrails: mandatory auth policy, WAF, schema validation, automated review.
- *"Many gateways = chaos?"* Standardize via shared templates.

**Red flags in answers**
- Manual approval as the permanent solution.
- No guardrails.
- Not measuring.

**Cheat sheet**
- Platform owns runtime; teams own routes.
- Config-as-code + CI guardrails.
- Golden path defaults.
- Measure lead time to expose an API.

### 6.5 [Tricky] "Our gateway aggregates data from 6 services for the home screen; it's getting slow and fragile." What do you do?
**What the interviewer is really testing**
Handling composition problems.

**Typical mid-level answer**
"Add caching." Treats only the symptom.

**Strong senior answer**
A synchronous 6-way fan-out inherits the weakest service's latency and availability. Steps: call in parallel with per-call timeouts and a global deadline; make non-essential widgets optional so the page renders partially (degrade gracefully, using Polly fallbacks); cache stable pieces; move aggregation to a BFF owned by the client team rather than the shared gateway. If this screen is critical and frequently read, build a read model (denormalized view populated by events) so one call serves it, trading freshness for speed (see Topic 11). Also check whether the UI really needs all data upfront (lazy-load below the fold). I would measure per-dependency latency from traces to target the slowest. Risk: caching staleness and read-model complexity; choose based on traffic and freshness requirements.

**Follow-up traps**
- *"Partial failure UX?"* Show skeletons/cached data and an error for the failed widget only.
- *"Hedged requests?"* Possible for tail latency, at cost of extra load.

**Red flags in answers**
- Sequential calls.
- No timeouts/partial rendering.
- Business logic in aggregation.

**Cheat sheet**
- Parallel + timeouts + optional widgets.
- Move aggregation to BFF.
- Read model for hot screens.
- Use traces to find slow hop.

**Red flags in answers (topic)**
- Gateway as a business-logic layer.
- One central gateway team approving every route.
- Gateway as the only auth layer.

**Cheat sheet (topic)**
- Gateway = edge concerns.
- BFF per client type, owned by client team.
- APIM external, YARP custom.
- Self-service route ownership with guardrails.

---

## Topic 7: API & event contract versioning, backward/forward compatibility, Pact

**Study notes**
- Backward compatible: new provider works with old consumers (adding optional fields). Forward compatible: old provider output readable by new consumers (consumers ignore unknown fields: "tolerant reader").
- Breaking: removing/renaming fields, changing types or semantics, making optional required, changing enum meaning.
- Strategies: additive evolution, URL/header/media-type versioning, expand-contract, deprecation policy with telemetry.
- Events are harder: consumers are unknown and messages persist in queues/logs; use schema registry (Event Hubs Schema Registry with Avro), upcasters, and versioned types.
- Consumer-driven contract testing (Pact): consumers publish expectations; providers verify them in CI (`can-i-deploy`).

| Change | Safe? |
|---|---|
| Add optional response field | Yes |
| Add required request field | No |
| Rename field | No (add new, deprecate old) |
| Change meaning of a value | No (even if type same) |
| Remove enum value | Risky |

### 7.1 [Easy] What counts as a breaking change in an API or event?
**What the interviewer is really testing**
Practical compatibility knowledge.

**Typical mid-level answer**
"Removing a field." Misses semantic and behavioral changes.

**Strong senior answer**
Breaking changes include removing or renaming fields, changing types, making an optional input required, tightening validation, changing status codes or error shapes, altering semantics (a `status` value now meaning something else), changing ordering/pagination behavior, and removing enum values consumers switch on. Adding fields is usually safe *if* consumers are tolerant readers (strict deserializers that fail on unknown properties break). Also adding enum values can break consumers with exhaustive switches. For .NET, I recommend consumers use default `System.Text.Json` behavior (ignore unknowns) and handle unknown enum values defensively. Compatibility is defined by consumer behavior, not just the schema, hence contract tests.

**Follow-up traps**
- *"Adding an enum value?"* Potentially breaking to strict consumers; document and test.
- *"Behavior changes?"* A latency or idempotency change may also break consumers.

**Red flags in answers**
- Only considers schema, not semantics.
- Strict deserialization everywhere.
- No consumer awareness.

**Cheat sheet**
- Breaking: remove, rename, retype, tighten, change meaning.
- Additive is safe for tolerant readers.
- Enums are a trap.
- Compatibility = consumer behavior.

### 7.2 [Follow-up] Compare URL, header, and media-type versioning, and say what you'd choose.
**What the interviewer is really testing**
Opinionated yet balanced.

**Typical mid-level answer**
"Use /v1/ in the URL." Acceptable but no reasoning.

**Strong senior answer**
URL versioning (`/v2/orders`) is explicit, cache/route/gateway-friendly, easy to test and document; downside is it implies whole-API versions and can proliferate. Header or query versioning keeps URLs stable and supports finer control, but is less discoverable and harder for caching and simple clients. Media-type versioning is the most REST-pure but heavy in practice. For public/partner APIs I choose URL major versions (breaking changes only) with additive evolution inside a major, plus a published deprecation policy (e.g. sunset header, notice period, usage telemetry per consumer). In ASP.NET Core, `Asp.Versioning` supports all schemes. The goal is to avoid breaking changes in the first place: prefer additive changes and keep majors rare. Trade-off: running v1 and v2 side by side adds maintenance; set sunset dates and track who still calls v1 via gateway analytics.

**Follow-up traps**
- *"Internal APIs also need versions?"* Often compatibility discipline + contract tests suffice; formal versions when consumers cannot be coordinated.
- *"How to retire v1?"* Telemetry, outreach, sunset header, brownouts, then removal.

**Red flags in answers**
- Versioning every change.
- No deprecation process.
- No telemetry on consumers.

**Cheat sheet**
- URL major versions for public APIs.
- Evolve additively within a major.
- `Asp.Versioning` library.
- Sunset with telemetry and notice.

### 7.3 [Medium] How do consumer-driven contract tests (Pact) work, and where do they fit?
**What the interviewer is really testing**
Practical test strategy, not tool trivia.

**Typical mid-level answer**
"Pact tests integration between services." Vague.

**Strong senior answer**
The consumer writes a test describing the requests it makes and the response parts it relies on; Pact records this as a contract file and publishes it to a broker. The provider's CI replays those requests against the real provider (with stubbed state via provider states) and verifies responses. Because contracts contain only what consumers actually use, providers learn which fields are safe to change. The Pact Broker `can-i-deploy` check gates deployment on verified compatibility, enabling independent releases without a shared integration environment. Limits: Pact tests shape/protocol, not business correctness; it works best for HTTP and messages (message pacts for events); it needs discipline (consumers must keep contracts minimal, providers must run verification on every build). Alternatives: OpenAPI-based compatibility checks (e.g. oasdiff) when consumers are unknown (public APIs); schema registry compatibility modes for events. Pact fits internal service meshes where consumers are known.

**Follow-up traps**
- *"Public API with unknown consumers?"* Use OpenAPI diff in CI and strict additive rules.
- *"Pact vs e2e?"* Pact replaces most inter-service e2e checks, not user-journey smoke tests.

**Red flags in answers**
- Treating Pact as end-to-end test.
- Providers don't run verification in CI.
- Contracts that mirror full schemas.

**Cheat sheet**
- Consumer records expectations -> provider verifies.
- `can-i-deploy` gates release.
- Minimal contracts, provider states.
- Unknown consumers -> OpenAPI diff.

### 7.4 [Hard] Evolve an event schema (`OrderPlaced`) that has 6 consumers and is stored in a long-retention log. How?
**What the interviewer is really testing**
Schema evolution under persistence and unknown consumers.

**Typical mid-level answer**
"Bump the version and update consumers." Falls short: can't coordinate 6 consumers or rewrite history.

**Strong senior answer**
Use compatibility-aware evolution. Prefer additive, optional fields so old and new consumers both work. For a breaking change, publish a new event type/version (`OrderPlaced.v2`) alongside the old, possibly dual-publishing for a deprecation window while consumers migrate, then retire v1 using consumer lag/usage telemetry. For stored events (event sourcing or long retention in Event Hubs/Kafka), consumers must read old versions forever, so use upcasters (transform old to new on read) or schema registry with compatibility rules (backward/forward/full) enforced in CI. A schema registry with Avro/JSON Schema/Protobuf provides explicit compatibility checks. Include a version/type header in the envelope. Keep events as facts with stable semantics; avoid putting internal DB shapes in events. Trade-off: dual publishing temporarily doubles traffic and complexity. Risk: consumers that deserialize strictly; mitigate via Pact message tests and a catalog of subscribers (AsyncAPI).

**Follow-up traps**
- *"Who migrates consumers?"* Publisher announces; consumer teams migrate by sunset date; platform tracks lag.
- *"Rewrite old events?"* Generally don't; upcast on read.

**Red flags in answers**
- Big-bang coordinated upgrade.
- Mutating published event meaning.
- No subscriber inventory.

**Cheat sheet**
- Additive first; new type for breaking changes.
- Dual-publish within deprecation window.
- Schema registry compatibility checks.
- Upcasters for stored history.

### 7.5 [Scenario] A producer team ships a breaking change on Friday and three consumers break. What's your response and prevention?
**What the interviewer is really testing**
Incident handling and systemic prevention.

**Typical mid-level answer**
"Roll back and tell them to be careful." No systemic fix.

**Strong senior answer**
Immediate: restore service - roll back or feature-flag the change, since fast recovery beats diagnosis; communicate status to affected consumers; check for poisoned messages/DLQs and replay after fix. Then a blameless post-mortem seeking system causes: no contract tests gating deploy, no consumer inventory, no breaking-change detection, Friday release without canary. Preventions: Pact verification and `can-i-deploy` (or OpenAPI/schema diff) as required pipeline checks; expand-contract discipline; canary/slot deployment with consumer-error monitoring; a registry of consumers (from gateway/telemetry) so the producer knows who to notify; a change-management norm (no risky releases before weekends is a heuristic, but automated safeguards matter more). **[Template]** "When [incident] happened, we [action]; the root cause was [cause]; we changed [control] and measured [result]."

**Follow-up traps**
- *"Is it the consumers' fault for not being tolerant?"* Both; the producer owns not breaking contracts, consumers own tolerant reading.
- *"Rollback with DB migration?"* Use backward-compatible migrations so rollback is safe.

**Red flags in answers**
- Blame-oriented.
- No pipeline gate.
- Only a process reminder.

**Cheat sheet**
- Recover first, analyze after.
- Automate compatibility checks in CI.
- Maintain consumer inventory.
- Canary + backward-compatible DB changes.

**Red flags in answers (topic)**
- Versions bumped for every change.
- No consumer inventory.
- Assuming consumers can upgrade simultaneously.

**Cheat sheet (topic)**
- Additive, tolerant reader.
- Major versions rare, with sunset policy.
- Contract tests + `can-i-deploy`.
- Schema registry/upcasters for events.

---

## Topic 8: Service sizing and team topology (Conway)

**Study notes**
- Conway's law: system structure mirrors communication structure of the organization. Inverse Conway: shape teams to get the architecture you want.
- Team Topologies: stream-aligned teams (own a flow of value), platform, enabling, complicated-subsystem teams; limit cognitive load.
- "You build it, you run it": the owning team is on-call and accountable.
- Ownership signals: one team per service; services per team limited; shared services = no owner = decay.

```
Stream-aligned team A -> Services A1,A2   \
Stream-aligned team B -> Services B1       > use -> Platform team (golden paths)
Enabling team -> coaches    Complicated-subsystem team -> e.g. pricing engine
```

### 8.1 [Easy] What is Conway's law and how does it affect microservices?
**What the interviewer is really testing**
Linking org and architecture.

**Typical mid-level answer**
"Software reflects the org chart." Needs the practical consequence.

**Strong senior answer**
Conway's law says systems mirror the communication structures of the organization that builds them. If three teams must coordinate to ship one feature, the architecture will show tight coupling at those seams; if a team has a clear, end-to-end remit, its service boundaries are crisp. So I use the inverse Conway maneuver: design teams around business capabilities (stream-aligned) so the natural architecture is decoupled and each team can deploy independently. Practically, I would map current services to teams, find services with multiple owners or teams with too many, and adjust either. Trade-off: reorganizing is costly and disruptive, so I do it when architectural coupling is an organizational problem. Splitting services without changing team structure just moves the coordination from code to meetings.

**Follow-up traps**
- *"Can architecture fix org problems?"* Not alone; structure and communication must change together.
- *"Is a vertical team (full-stack) required?"* Preferred for autonomy, but depends on size.

**Red flags in answers**
- Treats it as trivia.
- Splits services without team alignment.
- Ignores organizational change cost.

**Cheat sheet**
- Org structure shapes architecture.
- Inverse Conway: shape teams first.
- Stream-aligned, capability-based teams.
- Don't split code without ownership.

### 8.2 [Follow-up] How many services should a team own? What if nobody owns one?
**What the interviewer is really testing**
Practical ownership policy.

**Typical mid-level answer**
"As many as needed." No limit reasoning.

**Strong senior answer**
The constraint is cognitive load: a team must understand, operate, secure, and evolve every service it owns, including on-call. As a heuristic, a team of 5-8 can responsibly own a handful of services (say 2-5) tied to one capability; beyond that quality of ownership drops. I would not state a number as law; I'd watch signals: unmaintained dependencies, slow incident response, missing runbooks. Every service needs exactly one owning team recorded in a catalog (Backstage, or a repo `CODEOWNERS` plus service catalog) with on-call contact, SLO, and runbook. For orphaned services: decide to retire, merge into an owner's service, or formally assign with capacity; "owned by everyone" means no one. Trade-off: reassigning ownership takes team capacity, so prioritize by criticality. Regularly review the catalog.

**Follow-up traps**
- *"Shared libraries and platform services?"* Owned by the platform team with a service-level commitment.
- *"Inner source?"* Allowed contributions, but an owner approves.

**Red flags in answers**
- Orphan services tolerated.
- Ownership by role not team.
- No catalog.

**Cheat sheet**
- Cognitive load caps services per team.
- One owner per service in a catalog.
- Orphan -> retire/merge/assign.
- Runbook, SLO, on-call per service.

### 8.3 [Medium] Developers resist "you build it, you run it" because on-call is hard. How do you introduce it?
**What the interviewer is really testing**
Change management and empathy with engineering realism.

**Typical mid-level answer**
"Make everyone do on-call." Ignores readiness and fairness.

**Strong senior answer**
The principle gives teams incentive to build reliable systems because they feel the pain, and shortens feedback loops. But on-call without preparation causes burnout and attrition. I introduce it incrementally: first make services operable (alerts tied to SLOs, runbooks, dashboards, safe rollback, feature flags), then do shadowing and paired rotations with a support/SRE team in a transitional "you build it, we help run it" phase. Set limits: reasonable rotation size (enough people to avoid constant duty), compensation or time-off in lieu per company policy, alert-quality targets (few actionable pages), and a post-incident process that fixes causes. Measure pages per shift and toil. If the team is too small for sustainable on-call, that signals too many services or too little platform support. Trade-off: slower initial velocity while investing in operability. Risk: alert fatigue; track and prune noisy alerts.

**Follow-up traps**
- *"Why not a dedicated ops team?"* It decouples pain from cause; but SRE/platform teams can support.
- *"Small team of 3?"* Pool on-call across related teams or reduce service count.

**Red flags in answers**
- Forcing on-call with no tooling.
- Ignoring burnout.
- Paging on non-actionable alerts.

**Cheat sheet**
- Make operable first: SLO alerts, runbooks, rollback.
- Transition with shadowing.
- Measure pages and toil.
- Small teams: pool or consolidate.

### 8.4 [Hard] Two teams share one service and constantly step on each other. Options?
**What the interviewer is really testing**
Resolving shared ownership structurally.

**Typical mid-level answer**
"Use better branching and code reviews." Treats a structural problem tactically.

**Strong senior answer**
Shared mutable ownership of one service produces merge conflicts, unclear priorities, and diffused accountability. I diagnose why: are there two capabilities inside it (split along the seam), or one capability that both teams touch (merge the teams or assign one owner)? Options: (1) split the service by capability aligned with each team (needs data split); (2) one team owns it as a platform/service with a clear API and backlog intake, while the other consumes it (customer-supplier); (3) inner-source model: one owning team approves contributions from others with clear guidelines; (4) merge the teams if the work is one stream. I would choose by change patterns (git history by team/area) and strategic importance. Trade-off: splitting costs effort; the service-with-intake model adds dependency waiting but is simple. Interim measures: `CODEOWNERS`, modular internal structure, and a working agreement.

**Follow-up traps**
- *"Feature teams vs component teams?"* Prefer stream-aligned; component teams create handoffs unless the subsystem is truly complicated.
- *"Priority conflicts?"* Explicit intake process and shared roadmap.

**Red flags in answers**
- Only process fixes.
- No data on change patterns.
- Component teams for everything.

**Cheat sheet**
- Split along the seam, or one owner + intake, or inner source, or merge teams.
- Decide with git history data.
- Interim: CODEOWNERS, modular code.
- Prefer stream-aligned teams.

### 8.5 [Tricky] "Each team picks any language/stack they like; it's microservices!" Your view?
**What the interviewer is really testing**
Balance of autonomy and standardization.

**Typical mid-level answer**
"Yes, autonomy is the point." Ignores operational sprawl.

**Strong senior answer**
Polyglot freedom is a benefit but has real costs: hiring and mobility across teams, duplicated tooling, security patching across runtimes, and platform support multiplied. I favor "paved road": a recommended default stack (.NET + standard templates, logging, tracing, auth, CI/CD) that is cheap to follow, with room to deviate when there is a justified need (e.g. Python for ML) and the team accepts the extra operational burden. Standardize interfaces, not implementations: contracts, telemetry formats (OpenTelemetry), security baselines, and deployment mechanisms. Decision: constrain choices where cost is shared (security, observability, deployment), free where value is local. Risk: too strict a platform recreates a central bottleneck; too loose creates unsupportable sprawl. Review deviations via lightweight ADRs.

**Follow-up traps**
- *"Who supports the odd stack?"* The deviating team, entirely.
- *"Does it hurt hiring?"* Standard stack widens mobility.

**Red flags in answers**
- Unlimited freedom or rigid mandate.
- No cost accounting.
- No standard telemetry.

**Cheat sheet**
- Paved road + justified exceptions.
- Standardize interfaces, telemetry, security.
- Deviating team bears support cost.
- Lightweight ADRs for exceptions.

**Red flags in answers (topic)**
- Splitting services without team alignment.
- Orphaned or "everyone owns it" services.
- On-call without operability investment.

**Cheat sheet (topic)**
- Inverse Conway: capability-aligned teams.
- One owning team per service, in a catalog.
- Operability before on-call.
- Paved road for tech choices.

---

## Topic 9: Inter-service auth and trust boundaries

**Study notes**
- Trust boundaries: internet -> edge, edge -> internal, service -> service, service -> data. Do not assume "inside the network = trusted" (zero trust).
- Identity types: end-user identity (delegated) and workload identity (service itself).
- Azure: Microsoft Entra ID, managed identities, OAuth2 client credentials, on-behalf-of (OBO) flow, mTLS (mesh/ingress), Key Vault, private endpoints.
- Patterns: token validation at each service; propagate user context via JWT; downscope tokens; least privilege roles/scopes.

```
User -JWT(user)-> Gateway -JWT(user)-> Orders svc
                                   \-(MI token, scope: billing.read)-> Billing svc
```

### 9.1 [Easy] Is it OK for internal services to trust all calls from inside the VNet?
**What the interviewer is really testing**
Zero-trust mindset.

**Typical mid-level answer**
"Yes, the network is private." Falls short: lateral movement risk.

**Strong senior answer**
No; network location is not identity. If one service is compromised (dependency vulnerability, SSRF), an attacker inside the VNet could call anything that trusts the network. Zero-trust requires every service to authenticate callers and authorize each request. In Azure, workloads use managed identities to obtain Entra ID tokens; the callee validates the token (issuer, audience, signature, expiry) and checks roles/scopes. Network controls (private endpoints, NSGs, network policies) remain as defense in depth, not as the only control. Cost: token handling overhead and setup complexity, mitigated by shared platform libraries/templates (`Microsoft.Identity.Web`). Risk: misconfigured audience validation or overly broad roles; use least privilege per service pair.

**Follow-up traps**
- *"Isn't it overkill for 3 services?"* Managed identity + token validation is small effort; avoid building habits of trust-by-network.
- *"mTLS enough?"* It proves workload identity; still need authorization.

**Red flags in answers**
- "Firewall is enough."
- Shared API keys in config.
- No authorization, only authentication.

**Cheat sheet**
- Network location != identity.
- Managed identity + token validation.
- Authorize per request; least privilege.
- Network controls are defense in depth.

### 9.2 [Follow-up] User identity vs service identity: when do you pass the user's token downstream and when do you use the service's own identity?
**What the interviewer is really testing**
Delegation and confused-deputy awareness.

**Typical mid-level answer**
"Always pass the user's token." Can leak privileges and breaks background jobs.

**Strong senior answer**
Use the user's context when the downstream decision depends on *who the user is* (e.g. "show my orders"): propagate the token or exchange it using OAuth2 on-behalf-of for a downstream-audience token with reduced scope. Use the service's own identity (managed identity / client credentials) for system actions not tied to a user (background processing, event handlers, batch), and where the downstream service should not trust the user context. Passing a user's token raw to many services widens exposure (token replay, wrong audience), so tokens should be audience-restricted. When a service uses its own identity on behalf of a user, it must itself enforce the user's permissions to avoid the confused-deputy problem. Sometimes a pattern combines: service identity for authentication plus signed user-context claims (via token exchange) for authorization. Trade-off: OBO adds latency and complexity. Events are a special case: avoid carrying tokens in messages; carry user id and re-authorize or run as the service.

**Follow-up traps**
- *"Token in message body?"* No; tokens expire and leak into logs/DLQs.
- *"Confused deputy?"* A privileged service doing something for a lower-privileged caller without checking their rights.

**Red flags in answers**
- Forwarding same token everywhere.
- Service identity used without user permission checks.
- Tokens stored in queue messages.

**Cheat sheet**
- User-dependent decisions -> OBO/user token (audience-scoped).
- System work -> managed identity.
- Beware confused deputy.
- Never put tokens in messages.

### 9.3 [Medium] Where do authorization decisions live: gateway, each service, or a central policy service?
**What the interviewer is really testing**
Authorization architecture trade-offs.

**Typical mid-level answer**
"At the gateway." Single-layer and brittle.

**Strong senior answer**
Layered approach. The gateway does coarse checks: authentication, token validity, maybe scope presence, rate limiting. Each service enforces its own authorization because only it knows resource-level rules ("can this user edit *this* order?"); this prevents bypass if the gateway is circumvented. For consistent complex policies across services, a policy engine (OPA, Cedar-style, or ASP.NET Core policy-based authorization with shared handlers) avoids copy-pasted logic; a central decision service is possible but adds a latency and availability dependency, so evaluate locally with distributed policies/claims where possible. Claims in tokens are fast but stale until expiry; for highly dynamic permissions, query on demand with caching. Trade-off between consistency of policy and autonomy/latency. I would start with in-service policy-based authorization and shared libraries, and adopt a policy engine when rules become complex and cross-cutting.

**Follow-up traps**
- *"Stale roles in JWT?"* Keep tokens short-lived; use continuous access evaluation or fetch permissions for sensitive ops.
- *"Central authz service as SPOF?"* Cache decisions, local fallback, or sidecar evaluation.

**Red flags in answers**
- Authorization only at gateway.
- Duplicated divergent logic in each service.
- Central service with no availability plan.

**Cheat sheet**
- Gateway coarse; service fine-grained.
- Shared policy libraries or engine.
- Short-lived tokens for freshness.
- Beware SPOF in central authz.

### 9.4 [Hard] How do you handle secrets, certificates, and rotation across 30 services?
**What the interviewer is really testing**
Operational security at scale.

**Typical mid-level answer**
"Store secrets in Key Vault." Right start but incomplete.

**Strong senior answer**
Primary goal: eliminate secrets where possible. Use managed identities to reach Azure resources (SQL with Entra auth, Storage, Service Bus, Key Vault), so no connection-string passwords exist. For unavoidable secrets (third-party API keys), store them in Key Vault with RBAC per service identity, referenced via Key Vault references (App Service/Container Apps) or the configuration provider, with short caching and refresh. Plan rotation: dual-credential overlap (two valid keys), automated rotation (Key Vault rotation policies/Event Grid events), and apps that reload without redeploy. Certificates: managed certificates or automated issuance and renewal; monitor expiry with alerts. Provide this through the platform template so teams get it by default. Audit access through Key Vault logs. Risks: secrets in env vars leaking via logs/crash dumps, shared secrets across services (use per-service secrets so revocation is scoped), and break-glass procedures.

**Follow-up traps**
- *"Why not env vars?"* They leak into diagnostics; acceptable only as Key Vault references resolved by the platform.
- *"Rotation downtime?"* Overlap two keys, rotate consumers, then retire old.

**Red flags in answers**
- Secrets in repos/pipelines variables unscoped.
- One shared secret for all services.
- No rotation plan.

**Cheat sheet**
- Prefer managed identity: no secret.
- Key Vault + RBAC per service.
- Dual-key rotation, hot reload.
- Monitor cert expiry; scope secrets per service.

### 9.5 [Scenario] A pentest shows any internal service can call any other service's admin endpoints. Remediate.
**What the interviewer is really testing**
Prioritized, practical hardening.

**Typical mid-level answer**
"Add API keys." Weak, shared, hard to rotate.

**Strong senior answer**
First contain: restrict network exposure of admin endpoints (separate port/ingress, private only, network policies/NSGs) so quick risk reduction does not need code changes. Then fix properly: require Entra ID authentication on all internal endpoints with audience validation; define app roles per caller-service pair (least privilege), with admin endpoints requiring a distinct role or even a separate admin surface not reachable from other services. Use managed identities for callers and authorization policies in ASP.NET Core (`[Authorize(Roles = "Orders.Admin")]` or policy). Add audit logging of admin calls and alerts on unusual access. Roll out service by service with report-only logging first to find legitimate callers, then enforce, to avoid outages. Verify with a retest and automated tests that unauthenticated and unauthorized calls return 401/403. Trade-off: rollout effort vs. risk; prioritize by endpoint sensitivity.

**Follow-up traps**
- *"Mesh mTLS solves it?"* Authenticates callers; you still need authorization policy per caller.
- *"How avoid breaking callers?"* Report-only mode, then enforce.

**Red flags in answers**
- Shared static keys.
- Enforce everywhere at once without discovery.
- Only perimeter fixes.

**Cheat sheet**
- Contain via network first.
- Entra auth + roles per caller pair.
- Separate admin surface.
- Report-only -> enforce; automated authZ tests.

**Red flags in answers (topic)**
- "Internal = trusted."
- Same token/secret everywhere.
- Authorization only at the edge.

**Cheat sheet (topic)**
- Zero trust; managed identities.
- OBO for user context; service identity for system work.
- Layered authorization.
- Key Vault, rotation, per-service secrets.

---

## Topic 10: Service discovery, configuration, service mesh (needed or not?)

**Study notes**
- Discovery: on Kubernetes use DNS/Services; on Azure Container Apps use built-in internal DNS/ingress; Aspire/Dapr provide name-based resolution. Avoid hand-rolled registries (Consul/Eureka) unless needed.
- Config: externalize (Azure App Configuration, Key Vault, env vars), feature flags, per-environment, refreshable.
- Service mesh (Istio, Linkerd; Istio-based add-on for AKS; Dapr is not a mesh): sidecar/ambient data plane giving mTLS, retries, traffic splitting, telemetry, policy.
- Mesh costs: resource overhead, operational complexity, learning curve, debugging layers.

| Need | Without mesh | With mesh |
|---|---|---|
| mTLS | app-level or ingress | automatic |
| Retries/timeouts | Polly/Resilience in code | policy in mesh |
| Canary traffic split | ingress/Container Apps revisions | native |
| Telemetry | OpenTelemetry in app | automatic L7 metrics |

### 10.1 [Easy] How do services find each other on AKS or Container Apps?
**What the interviewer is really testing**
Platform awareness; avoiding unnecessary components.

**Typical mid-level answer**
"Use Consul or Eureka." Falls short on managed platforms.

**Strong senior answer**
The platform already solves it. On Kubernetes, a `Service` gives a stable virtual IP and DNS name (`orders.namespace.svc.cluster.local`) and load balances across ready pods via readiness probes. On Azure Container Apps, apps in the same environment reach each other via internal FQDN/app name, with revisions and traffic splitting. .NET Aspire adds service discovery abstractions for local dev and deployment manifests, and Dapr offers app-id invocation. So I do not add a separate registry unless I span heterogeneous environments. Key care points: correct readiness/liveness probes (so traffic goes only to ready instances), client-side connection handling (HttpClientFactory with DNS refresh; HTTP/2 gRPC long-lived connections), and config for endpoints through environment variables rather than hardcoded URLs.

**Follow-up traps**
- *"Stale DNS with HttpClient?"* Use `IHttpClientFactory`/`SocketsHttpHandler` with `PooledConnectionLifetime`.
- *"Cross-cluster?"* Then consider mesh/multicluster or gateway-based routing.

**Red flags in answers**
- Deploying Consul on AKS by default.
- Hardcoded IPs.
- No readiness probes.

**Cheat sheet**
- Use platform DNS/Service/internal ingress.
- Readiness probes gate traffic.
- Set `PooledConnectionLifetime`.
- Registry only for heterogeneous/multicluster.

### 10.2 [Follow-up] Centralized configuration: how do you manage config and feature flags across 30 services safely?
**What the interviewer is really testing**
Config discipline and blast-radius control.

**Typical mid-level answer**
"Put everything in appsettings." No governance.

**Strong senior answer**
Separate config from secrets: Azure App Configuration for non-secret settings and feature flags (with labels per environment, versioned snapshots), Key Vault for secrets referenced through App Configuration. In .NET, use the App Configuration provider with sentinel-based refresh and `Microsoft.FeatureManagement` for flags. Treat config as code: stored in git, reviewed, promoted through environments by pipeline, with validation (typed options with `ValidateOnStart`). Limit blast radius: scope settings per service/prefix and RBAC, avoid a global "shared" config many services read (which becomes hidden coupling), and roll out risky flags gradually with percentage/targeting filters. Flag hygiene: owner, expiry date, remove after rollout; stale flags are tech debt. Risk: a bad config pushed to all services at once; use staged rollouts and fast rollback via snapshot.

**Follow-up traps**
- *"Dynamic refresh danger?"* Not all settings should hot-reload; mark safe ones.
- *"Flag service outage?"* Cache last known good values and define default behavior.

**Red flags in answers**
- Secrets in App Configuration plain values.
- Giant shared config.
- Permanent flags.

**Cheat sheet**
- App Config + Key Vault separation.
- Config as code, validated on start.
- Scope per service; staged rollout.
- Flags have owners and expiry.

### 10.3 [Medium] Do we need a service mesh? Make the call.
**What the interviewer is really testing**
Resisting complexity; reasoning about benefits.

**Typical mid-level answer**
"Yes, meshes are standard for microservices." Falls short.

**Strong senior answer**
I ask what problems we have that a mesh solves: mandatory mTLS between all workloads, consistent retry/timeout/traffic policy across polyglot services, fine-grained traffic shifting, uniform L7 telemetry without code changes, and zero-trust network policy. If we are mostly .NET with a common platform library (resilience handlers, OpenTelemetry), a handful of services, and a managed environment like Container Apps (which offers mTLS options, ingress, revisions, Dapr), a mesh is likely unnecessary overhead. Mesh costs: sidecar CPU/memory per pod (ambient/sidecarless modes reduce this), upgrade complexity, a new failure domain, and harder debugging (double retries: app and mesh both retrying multiplies load). I would adopt a mesh when requirements are real and widespread (compliance mandating mTLS everywhere, many languages, many teams) and we have operations capability; AKS Istio add-on or Linkerd are options. Otherwise, start with ingress + platform libraries and revisit with specific triggers.

**Follow-up traps**
- *"Retries in both layers?"* Pick one layer to own retries; otherwise retry amplification.
- *"Sidecar overhead?"* Measure; ambient mode reduces it.

**Red flags in answers**
- Mesh as default.
- No owner for the mesh.
- Duplicated retry policies.

**Cheat sheet**
- Mesh solves: mTLS everywhere, polyglot policy, traffic shifting, uniform telemetry.
- Skip if small, .NET-only, on managed platform.
- One layer owns retries.
- Needs operational ownership.

### 10.4 [Hard] Compare Dapr, a service mesh, and plain libraries for cross-cutting concerns.
**What the interviewer is really testing**
Understanding overlapping tools and their abstraction levels.

**Typical mid-level answer**
"Dapr is like a mesh." Conflates them.

**Strong senior answer**
Mesh operates at the network layer (L4/L7): mTLS, traffic policy, telemetry, mostly transparent to apps. Dapr is an application runtime exposing building-block APIs (service invocation, pub/sub, state, bindings, secrets, workflows, actors) via sidecar HTTP/gRPC; it abstracts brokers/state stores so you can swap Service Bus for another broker, and offers its own mTLS and resiliency policies, so it overlaps partially with meshes. Plain libraries (Polly/Resilience, MassTransit, OpenTelemetry SDK, Azure SDK) give explicit, debuggable, in-process control with strong .NET integration, at the cost of per-language implementation. Trade-offs: Dapr adds a sidecar and an abstraction lowest-common-denominator risk, helpful for polyglot or portability but less valuable for a .NET-only Azure shop that is fine binding to Service Bus; meshes help with uniform infra policy. My default for .NET/Azure: libraries + platform features, adding Dapr only for specific building blocks (workflows, bindings) if they bring real value.

**Follow-up traps**
- *"Dapr plus mesh?"* Possible but duplicate mTLS/retry layers; design deliberately.
- *"Portability claim?"* Real but rarely exercised; do not pay for it speculatively.

**Red flags in answers**
- Saying Dapr and mesh are the same.
- Choosing for portability never needed.
- Ignoring sidecar operational cost.

**Cheat sheet**
- Mesh = network policy; Dapr = app building blocks; libs = in-process.
- Dapr helps polyglot/portability.
- .NET-only Azure: libs first.
- Avoid overlapping layers.

### 10.5 [Tricky] After adding a mesh, p99 latency doubled and intermittent 503s appeared. Approach?
**What the interviewer is really testing**
Systematic debugging in layered infrastructure.

**Typical mid-level answer**
"Remove the mesh." Possibly premature, no diagnosis.

**Strong senior answer**
Form hypotheses and test them with data. Compare traces/metrics before vs after for added hops (client sidecar + server sidecar). Check sidecar resource limits (CPU throttling causes latency), connection pool and keepalive settings, HTTP/2 vs HTTP/1.1 mismatches, protocol detection issues, and timeouts/retries configured in both app and mesh (retry amplification leads to load spikes and 503s). Check readiness during pod startup/shutdown (sidecar not ready before app, or app terminating before sidecar drains: use proper `preStop` and termination grace) and outlier detection/circuit-breaker settings ejecting healthy endpoints. Use mesh telemetry and `istioctl`/proxy logs to find where time is spent. Fix specific misconfigs; if overhead remains unacceptable for the value delivered, consider ambient mode or dropping the mesh for latency-critical paths. Decision based on evidence.

**Follow-up traps**
- *"503 UC/UF flags?"* Envoy response flags point to upstream connection failures/resets.
- *"Rollback plan?"* Gradual namespace-by-namespace injection allows quick opt-out.

**Red flags in answers**
- Blame or remove without data.
- Not checking duplicate retries.
- Ignoring lifecycle ordering.

**Cheat sheet**
- Compare traces before/after.
- Check sidecar limits, timeouts, retries, lifecycle ordering.
- Roll out mesh incrementally.
- Fix or scope; don't guess.

**Red flags in answers (topic)**
- Mesh as a default.
- Custom service registry on a managed platform.
- Duplicate retry layers.

**Cheat sheet (topic)**
- Platform DNS first.
- App Config + Key Vault + flags with hygiene.
- Mesh only for real needs.
- Dapr vs mesh vs libs: different layers.

---

## Topic 11: Cross-service queries (API composition, CQRS read models, data replication via events)

**Study notes**
- API composition: a composer calls several services and joins in memory. Simple; bad for large joins, pagination/sorting across services, and availability.
- CQRS read model: denormalized view built from events, optimized for queries (Azure SQL/Cosmos/Azure AI Search). Fast; eventual consistency; added pipeline.
- Data replication: keep local copies of data you need from other services (event-carried state transfer).
- Choose by volume, freshness, and query complexity.

```
Orders svc --events--\
Customers svc -events--> [Projection worker] -> Read DB (OrderSummary) <- UI query
Products svc --events-/
```

### 11.1 [Easy] How do you do a "join" across services?
**What the interviewer is really testing**
Knowing the options.

**Typical mid-level answer**
"Call both services and merge." Only one of three options.

**Strong senior answer**
Three options. (1) API composition: a BFF or composer service fetches from each owner and merges; simplest and always fresh, but latency/availability multiply, and sorting/filtering/pagination across sources is hard. (2) CQRS read model: build a denormalized projection fed by events from each service; fast single-query reads, supports complex filtering/search, but is eventually consistent and requires projection maintenance and rebuild capability. (3) Local replication of specific reference data via events so the service can join locally; good for stable, small data. I choose by query pattern: small result sets by key -> composition; list/search with filters across entities -> read model; stable reference attributes needed in hot paths -> replication. Risk in all: ownership confusion; the replicated or projected data is not a source of truth and must not be written back.

**Follow-up traps**
- *"Pagination across services?"* Hard in composition; use a read model.
- *"Rebuilding projections?"* Keep replay ability (event log/outbox history/snapshot).

**Red flags in answers**
- Direct cross-DB query.
- Not mentioning staleness.
- Writing to read models as a source of truth.

**Cheat sheet**
- Composition: simple, fresh, fragile.
- Read model: fast, flexible, eventual.
- Replication: local reference data.
- Projections are disposable and rebuildable.

### 11.2 [Follow-up] When is API composition good enough, and what makes it fail?
**What the interviewer is really testing**
Right-sizing solutions.

**Typical mid-level answer**
"It's fine for small stuff." Vague.

**Strong senior answer**
Composition works when the query is key-based (an order with its customer and items), the fan-out is small (2-3 calls), data volumes are small, and freshness must be high. It fails for list/search endpoints needing filtering and sorting across services (you'd need to over-fetch and join in memory), large result sets, high-traffic screens (amplified load on downstream), and when many dependencies make availability poor. Mitigations: parallel calls, batch endpoints (`GET /customers?ids=...`) to avoid N+1 across services, timeouts, caching, and partial responses. Beware the N+1 problem: 50 orders each calling Customers individually. When composition starts needing in-memory joins and paging hacks, that is the trigger to build a read model.

**Follow-up traps**
- *"GraphQL federation?"* Helps shape composition, but doesn't remove runtime dependency or N+1; use data loaders.
- *"Cache composition results?"* Yes with TTL and defined staleness.

**Red flags in answers**
- N+1 across network.
- Sorting across services in memory for big sets.
- No timeouts.

**Cheat sheet**
- Good: key lookups, few hops.
- Bad: search/list/paging, big data, hot paths.
- Batch endpoints defeat N+1.
- Trigger for read model: in-memory joins.

### 11.3 [Medium] Design a read model for an "Order search" screen across Orders, Customers and Shipping.
**What the interviewer is really testing**
CQRS projection design with practical details.

**Typical mid-level answer**
"Create a table with all the fields from events." Lacks reliability, ordering, rebuild details.

**Strong senior answer**
Define the query needs first: filters (customer name, status, date range), sort, paging, and acceptable staleness (e.g. a few seconds). Build a projection worker subscribing to `OrderPlaced`, `OrderStatusChanged`, `CustomerRenamed`, `ShipmentDispatched` and updating a denormalized `OrderSearch` table (Azure SQL with indexes, or Azure AI Search/Cosmos for text-heavy search). Design for at-least-once and out-of-order delivery: handlers are idempotent and use version/sequence numbers (only apply if event version > stored version); missing pieces (a shipment event arriving before the order) are handled by upsert with partial rows. Support rebuild: replay from the event log/outbox or a snapshot API from owners. Expose staleness via a `lastUpdated` and monitor projection lag. Ownership: the Search/Reporting team (or the Orders team) owns the read model; it is read-only to others. Trade-offs: duplicated data and eventual consistency, extra component to run. Risk: user edits and immediately searches and does not see their change; mitigate with read-your-writes tricks (return the new item locally, or poll).

**Follow-up traps**
- *"Out-of-order events?"* Version check and upsert; for ordering per aggregate use Service Bus sessions keyed by order id.
- *"Customer rename touches millions of rows?"* Accept batch updates or store customer name lookup join in the read store; decide per scale.

**Red flags in answers**
- Non-idempotent projections.
- No rebuild strategy.
- Ignoring lag monitoring.

**Cheat sheet**
- Start from query + freshness needs.
- Idempotent, version-checked upserts.
- Replay/rebuild capability.
- Monitor projection lag; read-your-writes workaround.

### 11.4 [Hard] Event-carried state transfer vs "thin event + callback": how do you decide?
**What the interviewer is really testing**
Coupling and data-duplication trade-offs.

**Typical mid-level answer**
"Always include all the data." or "Just IDs." No reasoning.

**Strong senior answer**
Thin events (just IDs) keep payloads small and the source of truth centralized, but consumers call back, reintroducing runtime coupling and load on the publisher, and the data may have changed by the time they call (the event describes the past while the callback returns the present). Fat events (event-carried state transfer) let consumers build local copies and work independently of the publisher's availability; downsides: bigger payloads, wider contract (more breaking-change surface), duplicated PII, and consumers might misuse data. My rule: include what consumers legitimately need and what is a stable part of the business fact; keep sensitive or volatile data out; for large or rarely used details provide a link/API. If consumers need autonomy and the data is reference-like, go fat; if consumers rarely need details or the data is sensitive, go thin. Document in the event catalog and version it. Risk: accidentally publishing internal model; design event schema deliberately, separate from the DB entity.

**Follow-up traps**
- *"Claim-check pattern?"* Store big payload in Blob Storage, put reference in message.
- *"PII in events?"* Minimize; encrypt or omit; plan erasure.

**Red flags in answers**
- Publishing DB entities as events.
- Always thin, creating chatty callbacks.
- Ignoring PII.

**Cheat sheet**
- Fat: autonomy, bigger contract.
- Thin: small, but callback coupling.
- Include stable facts consumers need.
- Claim-check for large payloads; minimize PII.

### 11.5 [Scenario] The read model is 20 minutes behind during peak and users complain. What do you check and change?
**What the interviewer is really testing**
Operational diagnosis of async pipelines.

**Typical mid-level answer**
"Scale out the workers." Might not help (ordering, DB bottleneck).

**Strong senior answer**
Measure the pipeline stage by stage: publish rate, queue/topic depth and oldest message age, consumer throughput, handler duration, and the read-store write latency/DTU/CPU or throttling (Cosmos RU 429s, Azure SQL log rate/locks). Find the bottleneck. Common causes and fixes: single-threaded ordered processing (sessions limit concurrency per session; partition by aggregate so many sessions process in parallel); slow handlers doing one-row-at-a-time writes (batch/bulk upserts, prefetch, increase concurrency); lock contention on hot rows; hot partitions; downstream throttling (scale the store); retries/poison messages blocking sessions. Also protect users: show "data updating" indicators, and for the user's own recent actions provide read-your-writes. Set an SLO on projection lag with an alert before users complain. Trade-off: costs of scaling vs relaxing the freshness expectation; align with product on acceptable lag.

**Follow-up traps**
- *"Autoscale on lag?"* Yes: KEDA scaling on Service Bus queue length.
- *"Catching up after outage?"* Separate catch-up path/bulk replay with lower priority.

**Red flags in answers**
- Scale blindly.
- No lag metric/SLO.
- Ignoring ordering constraints.

**Cheat sheet**
- Measure queue age, throughput, store latency.
- Batch writes, partition by key, KEDA scaling.
- Watch throttling and hot rows.
- SLO on lag + UX for staleness.

**Red flags in answers (topic)**
- Cross-service SQL joins.
- Non-idempotent projections.
- N+1 across services.

**Cheat sheet (topic)**
- Composition for small key lookups.
- Read models for search/list.
- Replicate reference data.
- Idempotent, versioned, rebuildable projections.

---

## Topic 12: Testing strategy in microservices (contract, component, e2e, test in prod)

**Study notes**
- Pyramid per service: unit -> integration (real DB via Testcontainers) -> component (service in isolation, external deps stubbed) -> contract (Pact/OpenAPI) -> few e2e journeys -> production verification (synthetics, canaries, feature flags, observability).
- Avoid a big shared integration environment as the main safety net: slow, flaky, ownership unclear.
- Test doubles for external dependencies: WireMock, in-memory/emulators (Azurite, Service Bus emulator, Cosmos emulator).
- Resilience/chaos testing: fault injection, latency, dependency down.

```
        /\   few e2e / synthetic journeys
       /--\  contract tests (Pact)
      /----\ component tests (service + real DB, stubs for others)
     /------\ unit tests (domain logic)
```

### 12.1 [Easy] Why is relying on full end-to-end tests a poor strategy for microservices?
**What the interviewer is really testing**
Test economics.

**Typical mid-level answer**
"They are slow." True but incomplete.

**Strong senior answer**
E2E tests across many services are slow, flaky (any service, network, or test data problem fails them), hard to debug (which service failed?), and require a fully assembled environment, which creates a release bottleneck and defeats independent deployment. They also tell you late. I use them sparingly for a few critical user journeys (login, checkout) as a smoke net. Most confidence comes from faster, more focused tests: unit tests for domain logic, component tests exercising one service through its real API with real infrastructure like Azure SQL in a container via Testcontainers and stubs for neighbors, and contract tests proving compatibility between teams. Then production safeguards (canary, synthetic monitoring, rollback) catch what remains. Trade-off: contract tests need discipline, but they scale linearly with the number of integrations rather than the number of journeys.

**Follow-up traps**
- *"Flaky e2e?"* Quarantine, fix or delete; never ignore.
- *"Where do e2e tests live?"* Owned by a journey owner; run post-deploy, non-blocking when possible.

**Red flags in answers**
- E2E as the primary gate.
- Shared staging with no owner.
- Ignoring flakiness.

**Cheat sheet**
- E2E: few, critical journeys.
- Component + contract carry the weight.
- Testcontainers for real infra.
- Production safeguards complete the net.

### 12.2 [Follow-up] What is a component test and how do you build one for an ASP.NET Core service?
**What the interviewer is really testing**
Hands-on practicality.

**Typical mid-level answer**
"A bigger unit test." Vague.

**Strong senior answer**
A component test runs the whole service in-process or in a container through its public interface, with real internals but externally replaced collaborators. In .NET: `WebApplicationFactory<Program>` hosts the app, Testcontainers spins up SQL Server, configuration points at it, and downstream HTTP services are stubbed with WireMock.Net; the message broker is replaced by the Service Bus emulator or MassTransit's in-memory harness. Tests assert behavior: HTTP request -> DB state -> published messages (outbox). They cover serialization, middleware, auth, EF mappings, migrations, and idempotency, which unit tests miss, and they run in a minute or so on CI. Trade-off: heavier than unit tests; keep the number focused on key behaviors and error paths (timeouts, duplicate messages, downstream 500s).

**Follow-up traps**
- *"In-memory EF provider?"* Misleading; behaves differently from SQL Server. Use a real SQL engine.
- *"Test auth?"* Use test token issuer or override auth handler with explicit roles.

**Red flags in answers**
- EF InMemory as proof of correctness.
- Stubbing the service's own internals.
- Not testing failure paths.

**Cheat sheet**
- `WebApplicationFactory` + Testcontainers + WireMock.
- Assert HTTP, DB, and emitted messages.
- Include duplicate/failure scenarios.
- Avoid EF InMemory for confidence.

```csharp
public class OrdersApi : IClassFixture<OrdersFactory> // WebApplicationFactory + MsSqlContainer
{
    [Fact] public async Task Duplicate_message_does_not_double_charge() { /* send twice, assert one charge */ }
}
```

### 12.3 [Medium] How do you test event-driven flows across services without a full environment?
**What the interviewer is really testing**
Contract thinking for async.

**Typical mid-level answer**
"Run everything in a test environment and wait." Flaky and slow.

**Strong senior answer**
Break the flow into verifiable pieces. For each producer, a test asserts that the action emits the expected event (via the outbox or test harness) and that its schema matches the contract; for each consumer, tests feed the contract-conforming message and assert resulting state and idempotency/out-of-order handling. Message contract tests (Pact message pacts, or schema registry compatibility checks) tie the two sides together. MassTransit's test harness (`AddMassTransitTestHarness`) allows asserting published/consumed messages in-memory. For orchestrations/sagas, test state machines with the harness for each path, including timeouts and compensation. Finally a small number of integration smoke tests with real Service Bus in a pre-production environment verify wiring, permissions, and topology. Trade-off: not testing the entire chain at once may miss emergent behavior, mitigated by production tracing and synthetic transactions.

**Follow-up traps**
- *"Timing in async tests?"* Use harness awaiting, not sleeps.
- *"Verify topology and permissions?"* Deployment-time smoke test in a real environment.

**Red flags in answers**
- `Thread.Sleep` assertions.
- No contract on message schema.
- Never testing compensations.

**Cheat sheet**
- Producer asserts events; consumer asserts handling.
- Message contracts bind them.
- MassTransit harness for sagas.
- Real-broker smoke test for wiring only.

### 12.4 [Hard] Is "testing in production" responsible? How do you do it safely?
**What the interviewer is really testing**
Mature risk management.

**Typical mid-level answer**
"No, always test before prod." Ignores that pre-prod can't mimic prod.

**Strong senior answer**
Pre-production cannot replicate real traffic, data, and failure patterns, so some verification must happen in production; the responsible approach is to limit blast radius and detect quickly. Techniques: feature flags to expose changes to internal users or a small percentage; canary/blue-green or Container Apps revisions with traffic splitting and automated rollback on SLO regression; synthetic transactions probing key journeys continuously; shadow traffic/mirroring for read paths; dark launches for new consumers; observability with per-version metrics. Safeguards: backward-compatible DB changes, kill switches, test-tenant data separated from real data, no destructive synthetic actions. This complements, not replaces, automated pre-merge testing. Trade-off: needs good observability and discipline; without them it is just YOLO. I'd define rollback criteria before the release.

**Follow-up traps**
- *"Test data in production?"* Use dedicated test tenants/accounts and exclude from analytics and billing.
- *"Chaos engineering?"* Only with guardrails, starting in lower environments, then controlled prod experiments (Azure Chaos Studio).

**Red flags in answers**
- Prod testing without rollback or observability.
- Synthetic tests mutating real customer data.
- Treating it as a replacement for earlier tests.

**Cheat sheet**
- Limit blast radius: flags, canary, revisions.
- Auto-rollback on SLO regression.
- Synthetics with test tenants.
- Needs observability first.

### 12.5 [Scenario] The team has 400 flaky e2e tests blocking releases weekly. What do you do?
**What the interviewer is really testing**
Pragmatic repair of a broken test strategy.

**Typical mid-level answer**
"Rewrite them all" or "add retries". Both poor.

**Strong senior answer**
First stop the bleeding: make e2e non-blocking for deployment while a smaller trusted set remains blocking, and quarantine flaky tests with ownership and a deadline. Classify the 400: which journeys are truly critical (maybe 10-20) versus tests that duplicate what component/contract tests could cover. Gather data on failure causes (environment, data, timing, shared state); fix root causes such as shared mutable test data (use isolated data per test), `sleep`-based waits (use polling with timeouts), unstable dependencies. Push logic checks down the pyramid: write contract and component tests for the integrations those e2e tests were covering, then delete redundant e2e tests. Track flake rate and mean time to green as metrics. Retries only as a temporary measure with reporting. Risk: loss of coverage while pruning; mitigate by mapping each deleted test to a lower-level test or production monitor.

**Follow-up traps**
- *"Who owns e2e?"* A named owner per journey, not 'QA'.
- *"Is deleting tests OK?"* Yes, when coverage moves elsewhere.

**Red flags in answers**
- Auto-retry until green.
- Rewrite everything at once.
- No ownership.

**Cheat sheet**
- Quarantine flakes; shrink blocking set.
- Fix root causes: data isolation, no sleeps.
- Move coverage down the pyramid.
- Track flake rate; assign owners.

**Red flags in answers (topic)**
- E2E as the main safety net.
- EF InMemory trust.
- No production verification.

**Cheat sheet (topic)**
- Unit + component + contract carry confidence.
- Few e2e; owned and non-flaky.
- Test async via harness + message contracts.
- Safe prod testing: flags, canary, synthetics.

---

## Topic 13: Deployment independence, CI/CD per service, versioning of shared libraries

**Study notes**
- Independent deployability: build, test, release one service without others; per-service pipeline, versioned artifacts (container images), immutable deployments.
- Repo strategies: polyrepo (strong autonomy, harder cross-cutting change), monorepo (easier refactor/atomic change, needs path-filtered pipelines and tooling).
- Release strategies: rolling, blue-green, canary, feature flags; DB expand-contract migrations.
- Shared libraries: infrastructure/utility OK; shared domain models/business logic = distributed monolith (shared kernel pitfalls).

```
commit -> build+unit -> component+contract -> image(tag=sha) -> deploy dev -> canary prod -> promote
(only this service)      can-i-deploy gate       immutable
```

### 13.1 [Easy] What does "independently deployable" require in practice?
**What the interviewer is really testing**
Concrete prerequisites.

**Typical mid-level answer**
"Each service has its own pipeline." Necessary, not sufficient.

**Strong senior answer**
A separate pipeline is the visible part; the real requirements are: no shared database schema with other services; backward-compatible API and event contracts (verified by contract tests); no lock-step dependency on shared domain libraries; configuration and secrets per service; backward-compatible DB migrations (expand-contract) so old and new versions run side by side during rollout; health/readiness probes and safe rollback; and tests that can prove safety without a full environment. Culturally, the owning team has the permission and confidence to ship alone. I verify with a metric: share of deployments done without coordination with another team, plus change failure rate. If a service cannot be released alone, it is not independent regardless of its pipeline.

**Follow-up traps**
- *"Coordinated releases sometimes unavoidable?"* Treat as a smell; use expand-contract in steps rather than simultaneous.
- *"Rollback with DB changes?"* Only safe if migrations are backward-compatible.

**Red flags in answers**
- Pipeline count as proof.
- DB migrations that break the previous version.
- Release trains.

**Cheat sheet**
- Own data, compatible contracts, no lock-step libs.
- Expand-contract migrations.
- Probes and rollback.
- Measure uncoordinated deploy share.

### 13.2 [Follow-up] Explain expand-contract for a column rename with zero downtime across rolling deployments.
**What the interviewer is really testing**
Practical migration safety.

**Typical mid-level answer**
"Rename the column in a migration then deploy." Breaks the running old version.

**Strong senior answer**
Rolling and canary deployments mean old and new code run concurrently against one database, so each step must be compatible with both. Expand: add the new column (nullable) and deploy code that writes both columns and reads from the old (or new with fallback); backfill existing rows in batches (avoid long locks/log growth on SQL Server); then switch reads to the new column, deploy; once all instances and rollbacks are safe, stop writing the old column; contract: drop the old column in a later release. Each step is independently deployable and reversible. Trade-off: more releases and temporary duplication, in exchange for no downtime and safe rollback. Tools: EF Core migrations generated but reviewed, idempotent scripts, a migration job run before app rollout (not at app startup across many instances), and testing the migration against a production-sized copy.

**Follow-up traps**
- *"Large backfill?"* Batch with throttling; consider online operations; monitor log usage.
- *"Run migrations at startup?"* Risky with many instances; use a dedicated job.

**Red flags in answers**
- Destructive migration in one step.
- Migrations at app startup with N replicas.
- No rollback story.

**Cheat sheet**
- Add -> dual-write -> backfill -> switch reads -> stop old -> drop.
- Each step compatible with N-1 version.
- Dedicated migration job.
- Test on prod-like data volume.

### 13.3 [Medium] Shared library policy: your org has `Company.Common` with models, helpers and DB access. What are the problems and what do you do?
**What the interviewer is really testing**
Shared kernel pitfalls.

**Typical mid-level answer**
"DRY is good; keep it shared." Ignores coupling costs.

**Strong senior answer**
A fat shared library creates version coupling: a change forces upgrades across services, drifting versions cause runtime mismatch, and shared domain types entangle bounded contexts so that one team's change forces others to rebuild/redeploy, the classic distributed-monolith driver. I would split it by nature. Keep: truly cross-cutting infrastructure with stable APIs (logging/telemetry setup, auth wiring, resilience defaults) as small, versioned NuGet packages following semantic versioning, with backward compatibility and a deprecation policy; Dependabot/Renovate to propagate updates. Remove: domain models, business rules, and persistence entities; let each service own its model and translate at the boundary. Contracts (DTOs, events) should be published as versioned contract packages by their owner or generated from schemas (OpenAPI/proto), not as a monolithic common DLL. Accept some duplication where DRY would create coupling ("duplication is cheaper than the wrong abstraction"). Governance: an owner team, small surface, and measured adoption. Migration: deprecate gradually, extract the platform bits into new packages, and shrink the old one.

**Follow-up traps**
- *"Contracts as NuGet?"* Fine if owner-versioned and additive; consumers must not be forced to upgrade in lockstep; schema-first generation is safer.
- *"Who upgrades dependencies?"* Automated PRs plus a support window.

**Red flags in answers**
- Domain entities shared across services.
- Unversioned "latest" dependency.
- DRY above all.

**Cheat sheet**
- Share infrastructure; don't share domain.
- SemVer, small surface, owner.
- Contracts via owner-published packages or schemas.
- Duplication can be the cheaper trade.

### 13.4 [Hard] Monorepo vs polyrepo for 30 services: decide.
**What the interviewer is really testing**
Trade-off analysis around autonomy vs consistency.

**Typical mid-level answer**
"Polyrepo is the microservices way." Dogmatic.

**Strong senior answer**
Polyrepo gives clear ownership, isolated permissions and pipelines, and fits autonomous teams; cons: cross-cutting changes (upgrading a library, API changes across producer/consumer) need many PRs, dependency drift, and inconsistent tooling. Monorepo gives atomic cross-service changes, easier refactoring and discoverability, shared tooling and consistent standards; cons: needs build tooling for path-filtered/incremental builds (Nx, Bazel, or well-built Azure Pipelines/GitHub Actions path filters), codeowners for protection, repository scale/performance, and a risk that teams treat it as one system and couple deploys. Both are workable; the real requirement is independent pipelines and deployable artifacts regardless of repo layout. I decide by team structure and tooling maturity: few teams close together with strong shared standards may prefer monorepo; many autonomous teams in different orgs prefer polyrepo with templates and automated dependency updates. Reversibility is moderate; avoid ideological moves.

**Follow-up traps**
- *"Monorepo means monolith?"* No; deployables can still be independent.
- *"Polyrepo consistency?"* Repo templates, shared pipeline templates, Renovate.

**Red flags in answers**
- Dogma either way.
- Monorepo without path-filtered builds.
- No governance for polyrepo drift.

**Cheat sheet**
- Poly: autonomy; Mono: atomic change.
- Independent pipelines in both.
- Tooling cost for monorepo; template cost for polyrepo.
- Decide by team structure.

### 13.5 [Tricky] "We deploy 30 services every Friday together because it's safer." Respond.
**What the interviewer is really testing**
Challenging risk myths.

**Typical mid-level answer**
"That's fine if it works." Misses the batch-size risk.

**Strong senior answer**
Large batch releases increase risk: more changes per release, harder to find the culprit, bigger blast radius, and slower recovery. It also confirms lock-step coupling. The instinct for safety is valid, so I'd address the underlying fear: why is it unsafe to release a single service? Likely missing contract tests, weak monitoring, non-backward-compatible changes, or manual deployment steps. A path: introduce per-service pipelines with automated tests, canary/automatic rollback, and feature flags to decouple deploy from release; start with one or two low-risk services deploying on demand; track change failure rate and MTTR to show smaller releases are safer. Be careful with 'Friday' rule - I'd focus on rollback capability and monitoring, not the weekday. Trade-off: investment in pipeline and testing up front. Risk: moving to independent releases while contracts are not compatible-safe creates breakage; sequence the safety nets first.

**Follow-up traps**
- *"Compliance requires change windows?"* Keep approvals automated and evidence-based; windows can coexist with smaller batches.
- *"Feature flags vs deploy?"* Deploy dark, release by flag.

**Red flags in answers**
- Batch size as safety.
- No rollback automation.
- Ignoring the root fear.

**Cheat sheet**
- Small, frequent releases reduce risk.
- Fix causes of fear: tests, monitoring, rollback.
- Decouple deploy from release via flags.
- Pilot with low-risk services; show metrics.

**Red flags in answers (topic)**
- Shared domain library.
- Destructive one-step migrations.
- Lock-step release trains.

**Cheat sheet (topic)**
- Independence = own data + compatible contracts + safe rollout.
- Expand-contract always.
- Share infra, not domain.
- Repo choice secondary to independent pipelines.

---

## Topic 14: Observability across services (correlation, tracing, SLOs)

**Study notes**
- Three signals: logs, metrics, traces (+ profiles, events). OpenTelemetry (OTel) is the vendor-neutral standard; Azure Monitor / Application Insights consumes OTel in .NET (`Azure.Monitor.OpenTelemetry.AspNetCore`).
- Correlation: W3C Trace Context (`traceparent`) propagates trace id across HTTP and messages; log with the trace id.
- SLI (measurement) -> SLO (target) -> error budget; alert on symptoms/burn rate, not every cause.
- Cardinality, sampling, and cost matter.

```
Client -> Gateway -> Orders -> (Service Bus msg: traceparent in properties) -> Billing
         one trace id across all spans; logs carry TraceId; metrics carry service/route
```

### 14.1 [Easy] How do you trace a request across 6 services, including through a queue?
**What the interviewer is really testing**
Context propagation, including async.

**Typical mid-level answer**
"Log a correlation id." Manual and inconsistent.

**Strong senior answer**
Use distributed tracing with W3C Trace Context. In .NET, `Activity` and OpenTelemetry instrument ASP.NET Core, `HttpClient`, SQL, and Azure SDK/Service Bus automatically, propagating `traceparent` on HTTP headers and in message application properties, so spans for producer and consumer join one trace (or are linked, as is common for message processing since one message may be processed much later; use span links for batch consumers). Logs include `TraceId`/`SpanId` (enabled through `ILogger` scopes/OTel logging) so I can pivot from a trace to logs. Export to Application Insights/Azure Monitor or another OTel backend through the Collector. Add business attributes (order id, tenant id) as span attributes carefully (no PII). Standardize via a shared platform package so every service is instrumented the same way. Risk: missing propagation on custom transports; include a test that verifies the trace id crosses a message hop.

**Follow-up traps**
- *"Sampling?"* Head-based sampling saves cost but can drop interesting traces; tail-based sampling (in a Collector) keeps errors and slow traces.
- *"Trace vs correlation id?"* Trace id is a standardized correlation id with span structure.

**Red flags in answers**
- Custom header per team.
- No propagation through messaging.
- PII in span attributes.

**Cheat sheet**
- OTel + W3C Trace Context.
- Propagate through message properties.
- Logs carry TraceId.
- Tail sampling keeps errors/slow traces.

### 14.2 [Follow-up] How do you define SLOs for a service, and what do you alert on?
**What the interviewer is really testing**
Reliability engineering fundamentals.

**Typical mid-level answer**
"Alert when CPU is high or errors occur." Cause-based, noisy.

**Strong senior answer**
Start from user-visible behavior: pick SLIs such as availability (good requests / total), latency (percent under a threshold, e.g. 95% < 300 ms), and for async pipelines freshness/lag. Agree SLO targets with product based on user need and cost (99.9% allows about 43 minutes of downtime per 30 days), leaving an error budget that guides risk-taking: if the budget is burnt, prioritize reliability over features. Alert on symptoms and burn rate (multi-window: fast burn pages, slow burn tickets) rather than individual causes; keep cause metrics (CPU, queue depth) on dashboards for diagnosis. Each page must be actionable and linked to a runbook. Dependencies matter: the service SLO cannot exceed its critical dependencies' without fallbacks. Trade-off: higher SLO costs more; not every service needs three nines. Review SLOs quarterly.

**Follow-up traps**
- *"100% SLO?"* Unrealistic and blocks change.
- *"SLO for internal services?"* Yes, define between teams; they are the basis for dependency expectations.

**Red flags in answers**
- Alerts on every metric.
- SLOs without business input.
- No error-budget policy.

**Cheat sheet**
- SLI -> SLO -> error budget.
- Alert on burn rate/symptoms.
- Actionable pages with runbooks.
- Dependency SLOs bound yours.

### 14.3 [Medium] Users report "checkout is slow sometimes". Walk through diagnosing it with your telemetry.
**What the interviewer is really testing**
Systematic debugging using observability data.

**Typical mid-level answer**
"Check logs of each service." Slow and unstructured.

**Strong senior answer**
First scope: when, which users/tenants/regions, what percentile (p50 fine, p99 bad?). Start at metrics for the checkout route: latency histograms by version, region, and dependency to see whether it correlates with a deployment, time of day, or tenant. Then pull sample slow traces (filter by duration) and read the critical path in the trace waterfall: which span dominates? Typical culprits: a downstream service's tail latency, SQL waits/blocking/parameter sniffing (check Query Store), retries stacking up, cold starts, connection pool exhaustion, thread pool starvation from sync-over-async, noisy neighbor or throttling (HTTP 429/Cosmos RU). Correlate with logs by trace id and with resource metrics. Form a hypothesis, validate by comparing fast vs slow traces, then fix and verify against the SLI. If observability is missing the needed span, add it and record the learning in a post-incident action. Trade-off: richer telemetry costs money; choose sampling policy wisely.

**Follow-up traps**
- *"Only p99 is bad?"* Look at tail causes: GC, lock waits, retries, cold start.
- *"No trace for slow ones due to sampling?"* Tail-based sampling to keep slow traces.

**Red flags in answers**
- Starts from random logs.
- Looks only at averages.
- Doesn't verify the fix against the SLI.

**Cheat sheet**
- Scope -> metrics -> traces -> logs.
- Percentiles, not averages.
- Compare fast vs slow traces.
- Verify fix on SLI; add missing telemetry.

### 14.4 [Hard] Observability bill is exploding. How do you cut cost without going blind?
**What the interviewer is really testing**
Cost-awareness and data strategy.

**Typical mid-level answer**
"Turn off logging." Dangerous.

**Strong senior answer**
First find where the volume comes from (Log Analytics usage by table/source; usually verbose logs, dependency telemetry, high-cardinality metrics, or health-check noise). Then apply levers in order of least information loss: drop noise at the source (health probes, debug logs in production, chatty dependency calls), fix log levels and structured logging so fewer, richer events are emitted; apply sampling (adaptive or tail-based, keeping 100% of errors and slow requests); reduce metric cardinality (do not label by user id); use cheaper log tiers (Basic logs/Auxiliary or archive in storage for compliance data), set retention per table, and use the OTel Collector to filter/route before ingestion. Establish budgets per team with dashboards so costs are visible (cost allocation). Keep SLO-driving signals untouched. Trade-off: less raw data for forensic investigation; compensate with good traces and the ability to temporarily raise verbosity via dynamic log level. Risk: sampling distorting rate metrics; compute metrics pre-sampling.

**Follow-up traps**
- *"Sampling breaks counts?"* Use metrics (pre-sampled) for counts; traces for examples.
- *"Compliance retention?"* Archive cheaply rather than keeping hot.

**Red flags in answers**
- Blanket cut with no analysis.
- High-cardinality labels.
- No per-team visibility.

**Cheat sheet**
- Find top volume sources first.
- Filter noise, sample, keep errors 100%.
- Control metric cardinality.
- Tiers/retention; team cost visibility.

### 14.5 [Scenario] A message was "lost": the customer was charged but no order exists. How do you investigate and prevent it?
**What the interviewer is really testing**
Forensics across async boundaries and reliability design.

**Typical mid-level answer**
"Check the dead-letter queue." One place only.

**Strong senior answer**
Start with the identifiers: payment id/order id and time window; find the trace or correlation across services. Walk the chain: was the command accepted (HTTP 202) and persisted, was the outbox row written, was the message published (outbox dispatcher status/logs), did Service Bus receive it (metrics, DLQ, scheduled/deferred messages), was it consumed (handler logs by message id), did the consumer fail after charging but before saving (non-atomic side effects), or was it deduplicated incorrectly? Common root causes: dual write (DB commit succeeded, publish failed) without an outbox, handler acknowledging (completing) the message before the work committed, swallowed exceptions, poison message ignored in DLQ, or idempotency key collision. Prevention: transactional outbox, complete messages only after durable processing, idempotent handlers, DLQ alerts and replay tooling, and a reconciliation job (payments vs orders) as a safety net that detects drift. Customer remediation: complete the order or refund per business policy. Document in a post-mortem.

**Follow-up traps**
- *"Reconciliation worthwhile?"* Yes for money flows; detects bugs and external failures.
- *"Charge-then-save ordering?"* Prefer saga with pending state; compensation if later step fails.

**Red flags in answers**
- Assuming broker lost it without checking the app.
- No reconciliation.
- Dual writes.

**Cheat sheet**
- Follow ids across each hop.
- Suspect dual write and early completion.
- Outbox, idempotency, DLQ alerts.
- Reconciliation as safety net.

**Red flags in answers (topic)**
- Correlation ids by convention only.
- Cause-based alert floods.
- Cost cuts with no analysis.

**Cheat sheet (topic)**
- OTel + W3C context everywhere, including messages.
- SLOs with burn-rate alerts.
- Diagnose: scope, metrics, traces, logs.
- Manage telemetry cost deliberately.

---

## Topic 15: Platform engineering and the cost of microservices (cognitive load, infra cost)

**Study notes**
- Platform team provides an internal developer platform (IDP): golden paths/templates, CI/CD, hosting (AKS/Container Apps), observability, secrets, service catalog (Backstage), policies. Product mindset; developers are customers.
- Costs of microservices: compute baseline per service (min replicas), networking, observability ingestion, per-service databases, licenses, engineering time (pipelines, on-call), cognitive load.
- FinOps: tag/allocate cost per team/service; right-size; autoscale to zero where suitable.

| Cost type | Examples |
|---|---|
| Infra | idle replicas, DBs per service, APIM tier, Service Bus premium, log ingestion |
| People | platform team, on-call, upgrades, security patching |
| Cognitive | tracing across services, many repos/pipelines, contract management |

### 15.1 [Easy] What does a platform team provide and why not let every team do it themselves?
**What the interviewer is really testing**
Understanding platform as an enabler.

**Typical mid-level answer**
"Manages infrastructure." Too narrow, ops-flavored.

**Strong senior answer**
A platform team builds self-service capabilities that reduce each stream-aligned team's cognitive load: paved-road service templates (a `dotnet new` template with OTel, health checks, auth, resilience preconfigured), standard pipelines, infrastructure modules (Bicep/Terraform), observability defaults, secrets and identity wiring, a service catalog, and documentation. If every team builds these themselves, there is duplicated effort, inconsistent security/observability, and slow onboarding. But the platform must be treated as a product: adoption is voluntary-ish, measured by developer satisfaction and time to first deploy; mandated and unusable platforms become bottlenecks. The platform should offer thin, composable capabilities, not a ticket queue. Trade-off: central investment and risk of building a platform nobody uses; start with the biggest pain and iterate with user feedback.

**Follow-up traps**
- *"Is a platform needed with 3 teams?"* Probably a template and shared pipeline are enough; scale up as teams grow.
- *"Metrics?"* Lead time to first deploy, adoption, DORA metrics, developer survey.

**Red flags in answers**
- Platform = ops ticket queue.
- Mandated tools nobody likes.
- No success metrics.

**Cheat sheet**
- Self-service paved roads.
- Treat platform as product.
- Measure time-to-first-deploy and satisfaction.
- Right-size platform to org size.

### 15.2 [Follow-up] Estimate the cost of splitting one service into three. What do you include?
**What the interviewer is really testing**
Total cost of ownership thinking.

**Typical mid-level answer**
"Three times the servers." Incomplete.

**Strong senior answer**
Include recurring and one-off costs. Infra: baseline replicas per service (minimum two for HA across zones), possibly a database each, ingress/APIM routes, Service Bus entities, observability ingestion, and non-prod environments (dev/test/staging multiply it). Engineering: three pipelines, deployment configs, dashboards and alerts, three on-call surfaces (or a bigger one), versioned contracts and contract tests, dependency/vulnerability patching x3, local dev complexity (Aspire/docker-compose). Redesign: data separation, sagas/outbox for former transactions, retries/idempotency. Opportunity cost: time not spent on features. Then I compare with benefits quantified: avoided contention, independent scaling savings, reduced incident blast radius. Present as ranges with assumptions and uncertainty, not false precision. Decision: split only if benefit clearly outweighs a conservative cost estimate; otherwise modularize internally.

**Follow-up traps**
- *"Scale-to-zero reduces cost?"* Helps for bursty workloads (Container Apps/KEDA) but cold starts and always-on needs remain.
- *"Hidden cost?"* Cross-service debugging time and coordination overhead.

**Red flags in answers**
- Infra-only estimate.
- Ignoring non-prod environments.
- False precision.

**Cheat sheet**
- Include infra, non-prod, people, redesign.
- Baseline replicas x HA.
- Quantify benefits on same basis.
- Present ranges and assumptions.

### 15.3 [Medium] Container Apps or AKS for a team of 8 running 12 services? Decide.
**What the interviewer is really testing**
Platform choice reasoning rather than defaults.

**Typical mid-level answer**
"AKS because Kubernetes is the standard." Ignores the operational burden.

**Strong senior answer**
Clarify requirements: need for custom operators/CRDs, advanced networking, GPU, strict control of node pools, service mesh, existing Kubernetes skills, multi-cloud portability, compliance. Azure Container Apps (built on Kubernetes, with managed control plane, KEDA autoscaling, Dapr integration, revisions with traffic splitting, scale to zero, ingress) removes cluster management: good for HTTP APIs, event-driven workers, and small-to-mid teams without dedicated platform engineers. AKS provides full Kubernetes control and ecosystem (Helm, operators, Istio add-on, fine-grained scheduling) at the cost of cluster lifecycle: upgrades, node pools, security patching, networking design, and needing platform expertise. For 8 developers and 12 services with no stated special needs, I choose Container Apps and revisit if hitting limits (e.g. custom networking or operators). Also consider App Service for simple web apps/APIs. Risks: Container Apps feature limits and lock-in; mitigate by keeping workloads containerized with standard config so migrating to AKS stays feasible.

**Follow-up traps**
- *"Isn't AKS more flexible?"* Yes, and it costs engineer time; flexibility you don't use is waste.
- *"Migration later?"* Containers + IaC make it achievable.

**Red flags in answers**
- Kubernetes by default.
- No consideration of team skills.
- Ignoring upgrade burden.

**Cheat sheet**
- Container Apps: managed, small/mid teams, KEDA, revisions.
- AKS: control, operators, mesh, platform team required.
- Choose on requirements and skills.
- Keep workloads portable.

### 15.4 [Hard] Developers complain about cognitive load: 20 repos, 20 pipelines, local setup takes 2 days. What do you do?
**What the interviewer is really testing**
Developer experience improvements.

**Typical mid-level answer**
"Write better documentation." Helpful but insufficient.

**Strong senior answer**
Quantify and prioritize pain via developer survey and metrics (time to first PR, time to run locally, pipeline duration). Quick wins: one-command local environment (Aspire AppHost or docker-compose with dependencies, seeded data), where developers run only the service under test plus stubs/emulators of neighbors instead of the whole system. Standardize via service templates and shared pipeline templates (YAML templates / reusable workflows) so a pipeline is a few lines; automate dependency updates; a service catalog (Backstage) with ownership, docs, and runbooks. Reduce the problem at the root: if people regularly need to run many services together, boundaries may be wrong, so consider consolidating. Establish a platform backlog with a product owner and measure improvements. Trade-off: platform investment vs feature delivery; justify by lead time and onboarding savings. Risk: building a heavy abstraction layer; keep it thin and optional.

**Follow-up traps**
- *"Local vs cloud dev environments?"* Dev containers or remote dev for heavy stacks; ephemeral environments per PR for integration.
- *"Docs vs automation?"* Automate what is repeated; document the rest.

**Red flags in answers**
- Docs only.
- Whole-system local runs as norm.
- No metrics.

**Cheat sheet**
- Measure DX: onboarding time, pipeline time.
- One-command local (Aspire/compose) + stubs.
- Templates and reusable pipelines.
- Wrong boundaries show up as DX pain.

### 15.5 [Tricky] "Cloud is elastic, so cost isn't a concern for architecture." Respond.
**What the interviewer is really testing**
Cost-conscious architecture mindset.

**Typical mid-level answer**
"True, we can scale as needed." Ignores waste and unit economics.

**Strong senior answer**
Elasticity controls capacity, but it also makes spend open-ended and architecture decisions have recurring cost: always-on replicas, per-service premium resources, chatty cross-AZ/region traffic with egress charges, log ingestion, and over-provisioned databases. Architecture is a cost lever: for instance, a database per service on provisioned tiers multiplies cost, solved by elastic pools or serverless tiers; chatty microservices increase network and compute; a monolith vs services decision changes baseline cost. I'd track unit economics (cost per transaction/tenant), tag resources to teams/services for showback, set budgets/alerts, and review right-sizing (Azure Advisor), reserved capacity/savings plans for steady workloads, and autoscaling with sensible min/max. But balance: engineer time is usually the costliest resource; do not over-optimize pennies at the expense of velocity. Make cost a visible non-functional requirement in design reviews.

**Follow-up traps**
- *"Reservations?"* Commit for steady baseline; keep burst on-demand.
- *"Cheapest architecture?"* The one meeting SLOs at lowest total cost, including people.

**Red flags in answers**
- "Cost is finance's problem."
- Optimizing infra pennies, ignoring people cost.
- No tagging.

**Cheat sheet**
- Elastic != free; track unit economics.
- Tag and allocate cost per team.
- Right-size, reserve baseline, autoscale.
- Include people cost in decisions.

**Red flags in answers (topic)**
- Kubernetes by default.
- Infra-only cost estimates.
- Platform without product thinking.

**Cheat sheet (topic)**
- Platform = paved road product.
- TCO includes non-prod and people.
- Container Apps vs AKS by needs/skills.
- Cost and cognitive load are design inputs.

---

## Topic 16: Migration from a monolith (strangler fig, ACL, branch by abstraction, data migration)

**Study notes**
- Strangler fig: route traffic through a facade/proxy; replace functionality piece by piece with new services; retire old code gradually.
- Anti-corruption layer (ACL): translation layer protecting new model from legacy model.
- Branch by abstraction: introduce an abstraction in the monolith over a component, build the new implementation behind it, switch by flag, remove old.
- Data migration: separate data last or with sync (CDC/outbox), dual-run and compare, expand-contract.
- Avoid big-bang rewrites.

```
Clients -> [Facade: YARP / APIM]
              |-- /invoices/* -> New Billing service (own DB)
              |-- everything else -> Legacy monolith
 ACL translates legacy <-> new model for shared interactions
```

### 16.1 [Easy] Describe the strangler fig pattern and its first steps.
**What the interviewer is really testing**
Practical incremental migration.

**Typical mid-level answer**
"Gradually replace the monolith with services." Lacks mechanics.

**Strong senior answer**
Place a routing facade (YARP, APIM, or Front Door) in front of the monolith so that all traffic passes through one point under our control. Then pick a capability, build the replacement, and route a subset of requests (by path, tenant, or percentage) to it while everything else still goes to the legacy system. Verify with shadow traffic or comparison, ramp up, then remove the legacy implementation. First steps: introduce the facade with pass-through (no behavior change) to validate it, choose the first slice by value/risk (a seam with few dependencies, high change rate, or clear pain), define the contract and data strategy, add observability to compare old and new, and plan rollback via routing. Prefer slices at the edge first (read-heavy, low coupling) before the core. Trade-off: temporary duplicated systems and two operational surfaces; the benefit is continuous delivery and reversible steps. Risk: stalling halfway, leaving a hybrid forever; keep a visible roadmap and retire legacy pieces promptly.

**Follow-up traps**
- *"Facade a SPOF?"* HA deployment; it is a thin router.
- *"How choose the first slice?"* Low-coupling, valuable, small data footprint.

**Red flags in answers**
- Starts with the hardest core module.
- No rollback via routing.
- Never decommissions legacy.

**Cheat sheet**
- Facade first (pass-through).
- Slice by path/tenant/percentage.
- Compare old vs new; ramp gradually.
- Decommission to finish.

### 16.2 [Follow-up] What is an anti-corruption layer and when do you need it?
**What the interviewer is really testing**
Model protection.

**Typical mid-level answer**
"A mapper between systems." Doesn't say why.

**Strong senior answer**
An ACL is a boundary component that translates between the legacy model/protocol and the new bounded context's model, so legacy quirks (cryptic codes, god tables, SOAP, flag-driven semantics) do not leak into new code. It is needed when the new service must interact with the legacy system whose model is poor or unstable, and we want to avoid conforming to it. Implementation can be an adapter in the new service, a separate translator service, or events published from legacy via CDC and translated into clean domain events. Translations should be explicit and tested (golden-file tests against real legacy data). Trade-off: extra code and a maintenance point, but it localizes coupling and makes the eventual removal of legacy easy: delete the ACL. Don't build one if you are replacing the legacy entirely in the same slice with no integration, or the legacy model is acceptable (conformist relationship).

**Follow-up traps**
- *"Where does ACL live?"* Owned by the new context's team.
- *"Performance?"* Cache and batch; ACL calls are still network/db calls.

**Red flags in answers**
- Leaking legacy DTOs into new domain.
- Putting ACL logic in legacy code.
- ACL with no tests.

**Cheat sheet**
- ACL isolates new model from legacy.
- Adapter, translator service, or CDC-to-events.
- Owned by new context team; tested.
- Delete ACL when legacy is gone.

### 16.3 [Medium] How does branch by abstraction help when the code you want to replace is deep inside the monolith?
**What the interviewer is really testing**
Refactoring-based migration technique.

**Typical mid-level answer**
"Create an interface." Misses the stepwise switch.

**Strong senior answer**
When a component has many internal callers and can't be intercepted at the HTTP edge, introduce an abstraction (interface) over it and route all callers through it, still using the old implementation; this is a safe, incremental refactor on main. Then build the new implementation (which might call a new external service) behind the same interface; use a feature flag (Microsoft.FeatureManagement) to switch by tenant/percentage, possibly running both and comparing results (a "parallel run" with the old as the source of truth). Once confidence is high, flip permanently, delete the old implementation and the flag. It avoids long-lived branches and keeps the system releasable. Limits: the abstraction must be at the right level (coarse enough to cover behavior including side effects); data ownership still needs a plan. Risk: forgotten flags; track cleanup.

**Follow-up traps**
- *"Comparison mode side effects?"* Run the new path read-only/dry-run, or in shadow with no side effects.
- *"vs strangler?"* Strangler works at request edge; branch by abstraction works internally; they combine.

**Red flags in answers**
- Long-lived feature branches.
- Abstraction leaking implementation details.
- No flag cleanup.

**Cheat sheet**
- Abstract -> new impl -> flag switch -> remove old.
- Always releasable.
- Parallel run with comparison (no side effects).
- Combine with strangler.

### 16.4 [Hard] How do you migrate the data for the extracted service without downtime or data loss?
**What the interviewer is really testing**
Data migration rigor.

**Typical mid-level answer**
"Export and import over the weekend." Downtime, no verification.

**Strong senior answer**
Phased approach. (1) Make ownership explicit: define which tables belong to the new service. (2) Initial load: copy the data to the new database (bulk copy, ADF, or snapshot). (3) Keep in sync during transition: CDC / Change Tracking from the legacy database or events from the monolith (outbox) feed the new store; avoid app-level dual writes since one write can fail, leaving divergence. (4) Run in shadow: new service reads its own store while the legacy remains the writer; compare results (reconciliation queries, checksums, sampled diffs) until they match. (5) Cutover the write path: briefly pause or use a controlled switch (flag) so the new service becomes the single writer; reverse the sync direction for a rollback window if needed. (6) Remove legacy access to those tables (permissions first, then drop). Handle identity/sequence collisions, referential links from the monolith (replace FKs with ids and API/event lookups), and data quality problems discovered. Risk: long dual-run increases complexity; set an exit date. Rehearse in a prod-like environment and define rollback criteria.

**Follow-up traps**
- *"Dual write?"* Avoid; use CDC/outbox for one-directional sync, switch writer once.
- *"Foreign keys from legacy tables?"* Replace with ids + lookups, or maintain view/API shim.

**Red flags in answers**
- Application dual writes.
- No reconciliation.
- Big-bang data cutover.

**Cheat sheet**
- Load -> CDC sync -> shadow read -> switch single writer -> retire legacy access.
- Reconcile with checksums/samples.
- Rollback window with reverse sync.
- Replace FKs with ids.

### 16.5 [Tricky] The CTO proposes a two-year full rewrite as microservices with a feature freeze on the old system. Argue the alternative.
**What the interviewer is really testing**
Risk assessment of rewrites and constructive influence.

**Typical mid-level answer**
"Rewrites always fail." Dogmatic and unhelpful.

**Strong senior answer**
Acknowledge the goals (maintainability, speed, scale) and then examine risks. Rewrites delay value by years, a moving target arises as the business evolves while the old system is frozen, the legacy hides undocumented behavior that is expensive to rediscover, the team maintains two systems, and there's a second-system effect; and the date is uncertain. Propose incremental replacement via strangler: deliver value every few weeks, prioritize slices by business value, and keep the freedom to stop when benefits flatten. Compare options explicitly in cost, time to first value, risk, and reversibility; present with a pilot on one slice to calibrate estimates with real data. Where the legacy is truly unsalvageable (end-of-life platform, security), a bounded rewrite of that portion may be justified with a parallel-run safety net. I would ask what constraints force a freeze; usually they can be relaxed by routing. Decision framed for the CTO in business outcomes, not engineering taste.

**Follow-up traps**
- *"Rewrite justified ever?"* When the platform is dead/unmaintainable and the scope is small enough to replace quickly, with parallel run.
- *"Feature freeze?"* Costs revenue/market response; avoid.

**Red flags in answers**
- Absolutes.
- No alternative with metrics.
- Ignoring undocumented legacy behavior.

**Cheat sheet**
- Rewrite risks: delay, moving target, hidden behavior, two systems.
- Strangler gives early, reversible value.
- Pilot to calibrate estimates.
- Argue in business outcomes.

**Red flags in answers (topic)**
- Big-bang rewrite/cutover.
- Application dual writes.
- Starting with the hardest core.

**Cheat sheet (topic)**
- Facade + incremental slices.
- ACL shields the new model.
- Branch by abstraction for internal seams.
- Data: CDC sync, reconcile, single-writer switch.

---

## Topic 17: Multi-tenancy in microservices

**Study notes**
- Tenancy models: silo (dedicated resources per tenant), pool (shared), bridge/hybrid. Database models: DB per tenant, schema per tenant, shared schema with `TenantId` column.
- Concerns: tenant identification (token claim, host, header), isolation (data, performance, security), noisy neighbor, per-tenant customization/limits, onboarding automation, cost allocation, compliance/data residency.
- Azure: elastic pools, Azure SQL Row-Level Security (RLS), Cosmos partition key per tenant, Entra ID multi-tenant apps (do not confuse with SaaS tenancy).

| Model | Isolation | Cost | Ops |
|---|---|---|---|
| DB per tenant | high | higher | many DBs to manage/migrate |
| Shared schema + TenantId | low (logical) | lowest | simple, risk of leaks |
| Hybrid | tiered | medium | complexity of both |

### 17.1 [Easy] How do you identify the tenant on each request and carry it through services?
**What the interviewer is really testing**
Tenant context propagation and trust.

**Typical mid-level answer**
"A tenantId header." Easily spoofed.

**Strong senior answer**
Derive the tenant from a trusted source: a claim in the validated token (issued by our identity provider, e.g. `tid`/custom `tenant_id` claim), or the host name mapped by the gateway; never trust a client-supplied header without validating that the authenticated principal belongs to that tenant. The gateway/service resolves it once into a request-scoped `ITenantContext`, used by the data layer (EF Core global query filters, RLS session context) and logs/traces (as attribute). For service-to-service calls, propagate the tenant via the token or a signed context header, and for messages include `TenantId` in the envelope so consumers can set context before processing; background jobs run per tenant explicitly. Risks: code paths that forget the tenant filter, leaking data across tenants; mitigate with defense in depth - global filters, RLS in SQL, and automated tests that attempt cross-tenant access.

**Follow-up traps**
- *"Spoofed header?"* Validate it against token claims.
- *"Async consumers?"* Tenant in message metadata; set context before handling, reject if missing.

**Red flags in answers**
- Trusting client header.
- Tenant filter by convention in each query.
- Missing tenant in messages.

**Cheat sheet**
- Tenant from validated token/host.
- Resolve once into scoped context.
- Propagate in tokens and message envelopes.
- Global filters + RLS + cross-tenant tests.

### 17.2 [Follow-up] Choose a data model for 500 small tenants and 5 large enterprise tenants.
**What the interviewer is really testing**
Tiered isolation reasoning.

**Typical mid-level answer**
"Database per tenant for isolation." Ignores operational and cost impact for 500.

**Strong senior answer**
Segment by needs. For 500 small tenants, a shared database with a `TenantId` column (or sharded pools of such databases) is the most cost-efficient, protected by RLS and global query filters; or databases per tenant in elastic pools if per-tenant backup/restore and isolation are valued and the count is manageable with automation (hundreds is feasible with tooling; thousands needs a catalog, scripted migrations, and sharding patterns). For the 5 enterprise tenants with contractual isolation, residency, or heavy loads, dedicated databases (or even dedicated stacks/regions) as a premium tier. A hybrid means the code must support both through a tenant catalog mapping tenant to connection/stamp. Considerations: schema migrations across many databases (staged rollouts, tooling), per-tenant restore, noisy neighbor protection (rate limits, resource governance), and cost allocation. Risk: complexity of supporting both models; limit it to two tiers and automate onboarding.

**Follow-up traps**
- *"Cross-tenant reporting?"* Separate analytics store fed from all tenants.
- *"Tenant moves between tiers?"* Build and test a tenant migration tool.

**Red flags in answers**
- One-size-fits-all.
- No catalog or migration tooling.
- Ignoring noisy neighbors.

**Cheat sheet**
- Pool for small, silo for large/regulated.
- Tenant catalog maps to resources.
- Automate migrations and onboarding.
- Limit to two tiers.

### 17.3 [Medium] How do you stop one tenant from degrading everyone else (noisy neighbor)?
**What the interviewer is really testing**
Resource governance in shared systems.

**Typical mid-level answer**
"Scale up." Costly and doesn't stop abuse.

**Strong senior answer**
Combine limits at several layers. Edge: per-tenant rate limiting and quotas (APIM policies or ASP.NET Core rate limiter partitioned by tenant) with fair behavior (429 + `Retry-After`). Compute: bulkheads such as per-tenant concurrency limits, separate queues or priority lanes for heavy tenants' batch work, and worker pools so one tenant's backlog does not starve others (Service Bus sessions or per-tenant queues for big ones). Data: indexed tenant-aware queries, query governors, resource-governed pools, or moving heavy tenants to dedicated databases. Observability: per-tenant metrics (carefully, cardinality) to spot offenders, and alert on skew. Policy: tier-based limits aligned to plans. Trade-off: complexity and perhaps customer friction when throttling; communicate limits clearly. Risk: hidden shared bottlenecks (a global lock, shared cache); load test with a noisy tenant scenario.

**Follow-up traps**
- *"Per-tenant metrics cardinality?"* Use aggregates for all, and top-N/tenant-level only for large tenants or via logs.
- *"Fairness in queues?"* Per-tenant queues or weighted consumption.

**Red flags in answers**
- Only scaling up.
- No per-tenant limits.
- Never load-tested noisy tenant.

**Cheat sheet**
- Rate limit per tenant at the edge.
- Bulkheads and separate lanes for heavy work.
- Dedicated resources for outliers.
- Load test a noisy tenant.

### 17.4 [Hard] How do you do schema migrations and releases across 300 tenant databases?
**What the interviewer is really testing**
Operational design for siloed tenancy.

**Typical mid-level answer**
"Run the migration script on each database." Manual and risky.

**Strong senior answer**
Make it an automated, observable, staged process. A tenant catalog records each tenant database, schema version, and status. A migration runner (job/pipeline) applies versioned idempotent scripts (EF migration bundles or DbUp/Flyway-style) with concurrency limits, per-tenant timeouts, and retry; results are recorded in the catalog. Roll out in waves: internal/test tenants, a small canary group, then the rest, pausing on failure thresholds. Because tenants will be on different schema versions temporarily, application code must tolerate N and N-1 schemas (expand-contract), so deployment and migration are decoupled. Handle failure: a tenant stuck on an old schema continues to work with compatible code; alert and fix individually. Long operations use online strategies. Elastic Jobs in Azure SQL can run scripts across a pool. Trade-off: complexity vs benefit of isolation. Risk: version skew accumulating; enforce SLA for convergence.

**Follow-up traps**
- *"Rollback?"* Forward-fix preferred; ensure backward compatible changes allow app rollback.
- *"Backups per tenant?"* Point-in-time restore per database is a siloed-model benefit.

**Red flags in answers**
- Manual scripts.
- Requires all tenants in lockstep.
- No catalog of versions.

**Cheat sheet**
- Catalog + automated runner + waves.
- Support N and N-1 schemas.
- Alert on failures; converge by SLA.
- Elastic Jobs or migration bundles.

### 17.5 [Scenario] A bug exposed one tenant's data to another. What now, and how to prevent recurrence?
**What the interviewer is really testing**
Incident response, accountability, and defense in depth.

**Typical mid-level answer**
"Fix the query and deploy." Misses containment and disclosure.

**Strong senior answer**
Contain first: disable the affected feature or deploy a kill switch/rollback, and stop ongoing exposure. Scope the blast radius using audit/access logs and traces: which tenants, which records, what time window. Engage security/privacy/legal immediately as this is likely a reportable incident (e.g. GDPR breach-notification timelines exist; follow the company's incident process rather than deciding alone). Communicate to affected customers per policy. Root cause with a blameless review, probably a missing tenant filter in a code path (raw SQL, cache key without tenant, background job). Prevention: defense in depth - EF global query filters plus SQL Row-Level Security with `SESSION_CONTEXT` so the database enforces tenancy even if code forgets; tenant-qualified cache keys; automated cross-tenant tests in CI (attempt reads/writes as tenant A against tenant B data); static checks/code review rules banning unfiltered raw queries; and for highest-sensitivity tenants, siloed databases. **[Template]** "Detected via [signal], contained by [action], root cause [x], now enforced by [control]."

**Follow-up traps**
- *"Cache leaks?"* Include tenant in cache keys; separate caches per tenant for the sensitive.
- *"How detect earlier?"* Anomaly alerts, canary tenants, and tests with seeded data.

**Red flags in answers**
- Fix without containment/disclosure.
- Relying only on developer discipline.
- Blame individuals.

**Cheat sheet**
- Contain -> scope -> notify (security/legal) -> fix -> prevent.
- RLS + global filters + tenant-aware caches.
- Cross-tenant tests in CI.
- Siloing for highest-risk tenants.

**Red flags in answers (topic)**
- Trusting a tenant header.
- One tenancy model for all.
- No automated cross-tenant tests.

**Cheat sheet (topic)**
- Tenant from validated identity, propagated everywhere.
- Pool vs silo by size/compliance; hybrid with catalog.
- Rate limits and bulkheads vs noisy neighbors.
- Defense-in-depth isolation: filters + RLS + tests.

---

## Topic 18: .NET-specific stack decisions (Aspire, Dapr, MassTransit, YARP, gRPC, Container Apps/AKS)

**Study notes**
- .NET Aspire: opinionated app model for composing distributed apps (AppHost orchestrates projects/containers/resources), service discovery, ServiceDefaults (OTel, health checks, resilience), local dev dashboard; deployment manifests to Container Apps/AKS (via `azd`). Not a runtime mesh.
- MassTransit: messaging abstraction over Service Bus/RabbitMQ/etc.: consumers, sagas/state machines, outbox, retries, test harness. Check current licensing terms for the version you adopt (v9 moves to commercial licensing; v8 remains open source) - verify before deciding.
- Alternatives: Azure SDK direct, NServiceBus (commercial), Wolverine, Brighter, CAP.
- Dapr: building blocks via sidecar; YARP: reverse proxy; gRPC: typed RPC; Durable Functions/Dapr workflow for orchestration.

| Concern | Typical .NET/Azure choice |
|---|---|
| Local orchestration | Aspire AppHost |
| Messaging | Service Bus + MassTransit / Wolverine / NServiceBus / Azure SDK |
| Edge/proxy | YARP or APIM |
| Hosting | Container Apps (default), AKS (when needed), App Service (simple) |
| Resilience | `Microsoft.Extensions.Http.Resilience` (Polly v8) |
| Telemetry | OpenTelemetry + Azure Monitor |

### 18.1 [Easy] What problem does .NET Aspire solve, and what doesn't it solve?
**What the interviewer is really testing**
Accurate understanding of a trendy tool.

**Typical mid-level answer**
"It's a framework for microservices." Inaccurate.

**Strong senior answer**
Aspire addresses developer experience and consistency in building distributed apps: the AppHost declares projects, containers, and dependencies (SQL, Redis, Service Bus emulator) so one command (F5) runs everything locally with service discovery and configuration injected; ServiceDefaults wires OpenTelemetry, health checks, and resilience defaults consistently; the dashboard shows logs, traces, and metrics locally; and deployment tooling (`azd`, manifests) maps the model to Azure Container Apps (or Kubernetes). It does not replace production infrastructure design, does not make boundaries right, and is not a service mesh or message framework. It is also fairly new and evolving, so check version/feature maturity and keep your IaC and deployment path reviewed. I would adopt it for multi-project solutions to cut onboarding time, while ensuring production concerns (networking, identity, scaling) are explicitly designed.

**Follow-up traps**
- *"Use in production?"* The AppHost is mainly dev/composition; production runs the same services with generated/explicit infra.
- *"Lock-in?"* Services remain normal .NET apps; ServiceDefaults are optional conventions.

**Red flags in answers**
- Calling it a runtime orchestrator or mesh.
- Treating it as architecture.
- Ignoring production deployment design.

**Cheat sheet**
- Aspire = app model + DX + defaults + dashboard.
- Not a mesh, not a broker, not architecture.
- Maps to Container Apps via `azd`.
- Services stay ordinary .NET apps.

### 18.2 [Follow-up] MassTransit vs Azure Service Bus SDK directly vs NServiceBus vs Wolverine: how do you choose?
**What the interviewer is really testing**
Build vs adopt with awareness of licensing and risk.

**Typical mid-level answer**
"MassTransit is the most popular." Popularity isn't a decision.

**Strong senior answer**
Clarify needs: simple queue consumers (a few handlers) versus sagas, outbox/inbox, retry/redelivery policies, request/response, scheduling, and testing support. For a few simple message flows, the Azure Service Bus SDK (or `ServiceBusProcessor`) is lightweight, transparent, and has no extra dependency, but you build outbox, retries, idempotency, and serialization conventions yourself. A framework (MassTransit, NServiceBus, Wolverine, Brighter) provides these patterns tested and consistent, at the cost of an abstraction and framework coupling (handlers, conventions), plus learning curve and upgrade tracking. Licensing and support must be examined: NServiceBus is commercial with strong support and tooling; MassTransit's licensing model has changed across versions (verify current terms for the version you'd standardize on); Wolverine and others are open-source with different maturity and community size. My process: shortlist two, run a spike on our hardest scenario (saga + outbox + testing), consider team skills, support needs, and exit cost (abstract at the handler/contract boundary so a swap is possible), and record an ADR. Risk: a framework becoming unsupported or costly; mitigate by keeping business logic framework-free.

**Follow-up traps**
- *"Why not just abstract the bus yourself?"* A home-grown bus abstraction usually ends up as a worse framework.
- *"Outbox with EF?"* Supported in frameworks (e.g. EF outbox integrations) so state and messages commit atomically.

**Red flags in answers**
- Choice by popularity.
- Ignoring licensing/support.
- Business logic coupled to the framework.

**Cheat sheet**
- Few simple flows -> SDK; patterns-rich -> framework.
- Spike on the hardest scenario.
- Check licensing/support for chosen version.
- Keep handlers thin, domain framework-free.

### 18.3 [Medium] When would you choose Dapr in a .NET/Azure shop?
**What the interviewer is really testing**
Balanced, non-hyped judgment.

**Typical mid-level answer**
"For portability across clouds." Rarely the real driver.

**Strong senior answer**
Dapr is attractive when: the system is polyglot and you want a uniform way to do pub/sub, state, bindings, secrets and service invocation across languages; you need specific building blocks that save effort (workflows, actors, input/output bindings, configuration, cron/jobs); or you want broker/state-store swapping per environment (local Redis, cloud Service Bus). It is a first-class option on Azure Container Apps and AKS. Costs: sidecar per instance (resources and one more thing to debug), a lowest-common-denominator API surface that can lag native features of Service Bus/Cosmos, versioning of the runtime and components, and extra learning for the team. For a .NET-only team already comfortable with Azure SDKs, MassTransit/NServiceBus and native SDKs are often simpler and more debuggable. Decision: adopt Dapr for concrete building-block needs or polyglot uniformity, not for speculative portability; pilot in one service first.

**Follow-up traps**
- *"Dapr plus Aspire?"* Aspire can host Dapr sidecars locally; they are complementary.
- *"Production support?"* Container Apps manages the Dapr runtime; still monitor versions and component config.

**Red flags in answers**
- Portability as the sole justification.
- Not mentioning sidecar and abstraction cost.
- Adopting without a pilot.

**Cheat sheet**
- Good: polyglot, specific building blocks, swap per environment.
- Cost: sidecar, abstraction ceiling, runtime versioning.
- .NET-only: native SDK/framework often simpler.
- Pilot before standardizing.

### 18.4 [Hard] Design the platform stack for a new .NET product with 5 teams on Azure. Justify each choice and what you'd defer.
**What the interviewer is really testing**
Synthesizing decisions with restraint.

**Typical mid-level answer**
"AKS, Istio, Kafka, Dapr, everything." Overbuilt.

**Strong senior answer**
Start from requirements: team skills (.NET), expected scale, compliance, release cadence. Proposal: begin with a modular monolith core plus a small number of services where independence is justified, hosted on Azure Container Apps (managed, revisions, KEDA, Dapr optional) with Azure SQL (elastic pool) for relational, Cosmos DB only where its access pattern fits, Service Bus for messaging with MassTransit or similar framework for outbox/sagas (after spike/licensing check), YARP or APIM at the edge depending on external API needs, OpenTelemetry to Application Insights, Entra ID + managed identities, Key Vault, Bicep for IaC, GitHub Actions/Azure Pipelines with per-service pipelines and contract tests, and Aspire AppHost for local dev. Defer: service mesh, AKS, Dapr (unless needed), Kafka/Event Hubs unless streaming workloads appear, multi-region active-active, and polyglot. Record ADRs with revisit triggers (e.g. move to AKS if we need custom operators/networking). Trade-off: start simple and accept some migration later versus predicting future needs. Risk: platform inconsistencies across teams; mitigate with templates and a small platform group.

**Follow-up traps**
- *"Why not AKS from day one?"* No requirement justifies the operations cost; reversible later.
- *"Why SQL first?"* Strong consistency, familiar, handles most workloads; add polyglot when justified.

**Red flags in answers**
- Everything at once.
- No deferrals or triggers.
- Tool list without requirements.

**Cheat sheet**
- Start managed and simple: Container Apps, Azure SQL, Service Bus.
- Frameworks for patterns; native Azure auth/identity.
- Defer mesh, AKS, Dapr, Kafka, multi-region.
- ADRs with revisit triggers.

### 18.5 [Tricky] gRPC calls to a service on AKS all hit one pod while others idle. Why, and what do you do?
**What the interviewer is really testing**
HTTP/2 and load balancing depth.

**Typical mid-level answer**
"Kubernetes load balancing is broken." Misdiagnosis.

**Strong senior answer**
gRPC uses long-lived HTTP/2 connections that multiplex many calls. A standard Kubernetes ClusterIP Service balances at L4 per connection (kube-proxy), so once a client opens a connection it sticks to one pod and all calls go there; scaling out adds pods that receive no traffic from existing clients. Options: client-side load balancing via a headless Service (DNS returns pod IPs) with the .NET gRPC client's `DnsResolver`/round-robin balancer and periodic DNS refresh (watch for stale endpoints); an L7 proxy/ingress that balances per request (Envoy-based ingress, Linkerd/Istio, NGINX with gRPC support); or Container Apps' ingress which handles HTTP/2 per-request balancing (verify behavior in your environment). Complement with connection lifetime limits (`MaxConnectionAge` server-side or `PooledConnectionLifetime` client-side) so clients periodically reconnect and rebalance. Choose by operational preference: mesh/ingress if already present, otherwise headless + client-side. Verify with per-pod request metrics.

**Follow-up traps**
- *"Multiple channels per client?"* Can spread load but not a real fix; use proper balancing.
- *"Health-aware balancing?"* Mesh/proxy outlier detection or gRPC health checking.

**Red flags in answers**
- Blaming Kubernetes without understanding L4 vs L7.
- Over-provisioning pods as the fix.
- No verification.

**Cheat sheet**
- gRPC = long-lived HTTP/2 -> L4 balancing pins connections.
- Fix: L7 proxy/mesh or headless + client-side LB.
- Limit connection age to rebalance.
- Verify via per-pod metrics.

### 18.6 [Scenario] You inherit a team that standardized on Dapr + Istio + MassTransit + YARP + APIM for 6 services. Rationalize it.
**What the interviewer is really testing**
Simplifying overlapping tools with minimal disruption.

**Typical mid-level answer**
"Rip it out and start clean." Disruptive and risky.

**Strong senior answer**
Map each tool to the capability it provides and find overlaps: Dapr and Istio both provide mTLS and retries; Dapr pub/sub and MassTransit both handle messaging; YARP and APIM both sit at the edge. Overlap causes duplicate retries, confusing failures, and several upgrade/ownership burdens for just 6 services. Gather facts: who uses what, which features are actually used, incidents attributed to the stack, cost, and team skills. Then decide per capability with the principle "one tool per concern, justified by need": e.g. keep MassTransit for messaging/sagas, drop Dapr pub/sub if unused; drop Istio unless mTLS/traffic policy is required (Container Apps or AKS network policies + managed identity might suffice); keep APIM only if external consumers need its features, else YARP/ingress alone. Sequence removal one capability at a time, behind tests and observability, starting with the least-used, with rollback paths. Document in ADRs and communicate to the team; the goal is lower cognitive load and operational surface, not purity. Risk: removing something someone depends on silently; mitigate by discovery and gradual rollout.

**Follow-up traps**
- *"Sunk cost?"* Ignore it; compare future cost and benefit.
- *"What if some are required by compliance?"* Keep those and simplify the rest.

**Red flags in answers**
- Wholesale rewrite.
- Keeping overlaps for sunk cost.
- Removing without usage data.

**Cheat sheet**
- Map tools to capabilities; find overlaps.
- One tool per concern.
- Remove incrementally with tests and observability.
- Decide on future value, not sunk cost.

**Red flags in answers (topic)**
- Tool lists without requirements.
- Treating Aspire as architecture or a mesh.
- Ignoring licensing/support when choosing frameworks.

**Cheat sheet (topic)**
- Start managed and simple; defer mesh/AKS/Dapr until justified.
- Aspire for DX; MassTransit/others for messaging patterns (check licensing).
- gRPC needs L7/client-side balancing on Kubernetes.
- Record ADRs with revisit triggers.

---
*End of S01.*

