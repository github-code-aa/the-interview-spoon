# S08 — Architecture Decisions & Trade-offs (Senior Engineer Interview Guide)

**How to use:** Read each topic's study notes, then answer every question aloud before reading the "Strong senior answer". Practise the shape: context -> options -> criteria -> trade-offs -> decision -> how you'd validate/reverse.
**Note:** Any "story" prompts are labelled templates; fill them with your own real experience — never memorise someone else's.

## Table of Contents
1. What "architecture" means to a senior (costly-to-change decisions, quality attributes)
2. Gathering & prioritising NFRs; ATAM-lite trade-offs
3. Architecture Decision Records (ADRs)
4. Reversible vs irreversible decisions
5. Build vs buy vs open source; technology selection; lock-in
6. Choosing .NET libraries (licensing & maintenance risk)
7. Cloud compute choices: PaaS vs containers vs serverless
8. Evolutionary architecture, fitness functions, architecture tests
9. Managing technical debt
10. Rewrite vs refactor vs strangler; modernising .NET Framework
11. Documentation that works & governance without bureaucracy
12. Architecture reviews; presenting to executives vs engineers
13. Ambiguity, "it depends" done right, over/under-engineering
14. Testability, operability & internal platform product thinking
15. Standardisation vs autonomy; monorepo vs polyrepo; shared libraries; multi-team roadmaps
16. Cost as an architecture driver; handling architecture disagreements
17. Scenarios I: messaging service choice, CTO wants a rewrite, conflicting designs
18. Scenarios II: inherited system with no docs, greenfield vs enterprise, ORM/data access strategy
19. Complete ADR walkthrough & "What is your architectural philosophy?"

---

## Topic 1: What "architecture" means to a senior

**Study notes**
- Architecture = the set of decisions that are **costly to change** plus the structures/quality attributes they produce. Not diagrams, not frameworks of the month.
- Quality attributes (NFRs) drive architecture far more than features do.

| Attribute | Typical question | .NET/Azure/SQL lever |
|---|---|---|
| Performance | p95 latency / throughput target? | caching, indexing, async I/O, batching |
| Availability | SLO? RTO/RPO? | zone redundancy, retries, failover groups |
| Security | Data class, threat model? | Entra ID, managed identity, Key Vault, private endpoints |
| Modifiability | What changes often? | module boundaries, contracts, tests |
| Cost | Unit cost per tenant/request? | SKU, autoscale, reserved capacity |
| Operability | Who is paged, can they diagnose? | App Insights, health checks, runbooks |

- Rule of thumb: architecture decisions = data stores, service boundaries, sync vs async integration, identity model, deployment topology, tenancy model.
- Code-level choices (naming, a LINQ shape) are design, usually cheap to change.

### 1.1 [Easy] How would you define software architecture to a new developer?
**What the interviewer is really testing**: Whether you see architecture as decisions/trade-offs rather than boxes and buzzwords.
**Typical mid-level answer**: "It's the high-level design: layers, microservices, databases." Falls short: describes artefacts, not why they matter or what makes a decision "architectural".
**Strong senior answer**: Architecture is the set of significant decisions — those hard to reverse or that shape quality attributes — and the reasons behind them. Context: every system has an architecture, even accidental. Options differ in which qualities they favour; e.g. a single SQL Server database favours consistency and simplicity but limits independent scaling. Criteria are the NFRs that matter for this business. I'd say "if changing it later costs weeks of multi-team work, it's architectural". The test of a good architecture is that it makes the important changes cheap and the risky ones visible. I'd validate by checking that the decisions are written down (ADRs) and traceable to a requirement.
**Follow-up traps**: "Is a folder structure architecture?" (only if it enforces a boundary). "Does an architect need to code?" (yes, to stay honest about cost).

### 1.2 [Follow-up] Which decisions in a typical .NET/Azure system are hardest to change?
**What the interviewer is really testing**: Ability to rank decisions by change cost.
**Typical mid-level answer**: "The framework and the cloud provider." Falls short: ignores data and contracts, which usually outlast frameworks.
**Strong senior answer**: Hardest are usually (1) the data model and system of record, because data outlives code and migrations need downtime or dual-writes; (2) public contracts — REST APIs, message schemas, events consumed by others; (3) identity/tenancy model; (4) service boundaries, since merging or splitting later means data migration; (5) sync vs async integration style. Easier: ORM, UI framework, hosting SKU, logging library. So I spend design effort proportionally: thorough on data ownership and contracts, light on things behind an interface. Validate by asking "what would it cost, in team-weeks, to reverse this?" and putting the expensive ones in ADRs.
**Follow-up traps**: "Isn't the cloud provider the biggest?" (depends on proprietary service usage; see Topic 5). "Where does language choice sit?" (moderately hard, team-skill coupled).

### 1.3 [Medium] A product owner says "just make it fast and reliable". What do you do?
**What the interviewer is really testing**: Turning vague quality wishes into measurable targets.
**Typical mid-level answer**: "Add caching and retries." Falls short: solutions before numbers.
**Strong senior answer**: "Fast" and "reliable" are not requirements until quantified. I'd ask: which user journeys, at what load (peak requests/sec, growth), what p95/p99 latency, what availability SLO (99.9% = ~43 min/month downtime), what RTO/RPO, and what it costs the business to miss each. Options then cost different amounts: zone-redundant App Service and Azure SQL zone redundancy cost more than single-zone; multi-region active/active costs far more and adds consistency complexity. I'd present 2-3 tiers with price and effort and let the business choose consciously. Decision recorded as SLOs; validate with load tests and Application Insights availability tests, and revisit after real production data.
**Follow-up traps**: "What if they say 100%?" (explain error budgets and cost curve). "Who owns the SLO?" (product + engineering jointly).

### 1.4 [Tricky] Two quality attributes conflict (security vs usability, performance vs consistency). How do you decide?
**What the interviewer is really testing**: Comfort with explicit trade-offs rather than "both".
**Typical mid-level answer**: "Find a balance." Falls short: no criteria, no ownership.
**Strong senior answer**: Conflict is normal, so I make it explicit. Context: e.g. read-your-writes consistency vs read scale-out using Azure SQL read replicas (replica lag). Options: route critical reads to primary, accept eventual consistency elsewhere, or cache with short TTL. Criteria: business impact of stale data for each use case, not system-wide. Trade-off: per-use-case consistency adds routing complexity. Decision: primary for money/inventory reads, replica for dashboards. I'd get the business to rank the attributes for this system, record in an ADR, and validate with metrics (replica lag, primary CPU). Reversible if the routing sits behind one repository abstraction.
**Follow-up traps**: "Who breaks the tie?" (the accountable owner after informed input). "What if the ranking changes?" (ADR supersede).

### 1.5 [Hard] Can a system have "no architecture"? What happens to quality attributes then?
**What the interviewer is really testing**: Understanding that accidental architecture emerges.
**Typical mid-level answer**: "Agile means we don't need it up front." Falls short: confuses no up-front design with no decisions.
**Strong senior answer**: Every decision made by default is still a decision — a shared database becomes de-facto integration, and static helpers become coupling. Without intent, qualities become accidents: performance degrades, deploys get risky, and onboarding slows. The senior answer is "just enough" up-front: identify the few costly decisions and architectural drivers, decide them deliberately, and leave the rest emergent. I'd keep a lightweight set of ADRs and a couple of automated architecture tests so intent survives staff turnover. Validate by whether a new engineer can find where a change belongs in under a day.
**Follow-up traps**: "Big design up front?" (no — risk-based). "Who is the architect in a self-organising team?" (the role is shared; accountability is explicit).

**Red flags in answers**
- Defining architecture only as diagrams or technology names.
- Naming "microservices" or "serverless" without naming the quality attribute it serves.
- No numbers for NFRs ("fast", "scalable").

**Cheat sheet**
- Architecture = costly-to-change decisions + the reasons.
- Data and contracts outlive frameworks.
- Quantify NFRs before choosing solutions.
- Trade-offs are made explicit and owned.
- Just enough up-front; the rest emerges.
- Write expensive decisions down.

---

## Topic 2: Gathering & prioritising NFRs; ATAM-lite

**Study notes**
- Clarifying questions: users/volume, growth, data sensitivity, compliance (GDPR, SOC2), SLOs, RTO/RPO, budget, team skills, deadlines, integration constraints, existing estate.
- Express NFRs as **quality-attribute scenarios**: *Source – Stimulus – Environment – Response – Measure*. Example: "Under peak load (5k req/s), a customer search returns in <300 ms at p95."
- **ATAM-lite** (a short version of the Architecture Tradeoff Analysis Method):
  1. List the top 3-5 quality-attribute scenarios, ranked.
  2. Sketch candidate architectures.
  3. For each scenario, mark where each candidate is strong/weak.
  4. Identify **sensitivity points** (a parameter that strongly affects a quality), **trade-off points** (one decision affecting several qualities in opposing ways), and **risks/non-risks**.
  5. Decide, record in an ADR, list mitigations.

| Scenario (ranked) | Option A: modular monolith | Option B: microservices |
|---|---|---|
| Deploy daily by 4 teams | Medium | Strong |
| Strong consistency on orders | Strong | Weak (sagas) |
| Ops with 6 engineers | Strong | Risk |

### 2.1 [Easy] What are the first questions you ask before designing anything?
**What the interviewer is really testing**: Discovery discipline.
**Typical mid-level answer**: "What features do we need?" Falls short: functional only.
**Strong senior answer**: I split into business, load, risk and constraints. Business: who uses it, what decision or revenue depends on it, what's the deadline. Load: users, requests/sec, data volume and growth over 12-24 months, peaks. Risk: data sensitivity, compliance, cost of downtime/data loss. Constraints: existing systems, team skills, budget, approved Azure services, regions. I'd also ask what is explicitly *not* required, as that reveals where to simplify. Output is a one-page list of ranked quality-attribute scenarios and an assumptions log, validated by the stakeholders before I draw boxes.
**Follow-up traps**: "What if nobody knows the numbers?" (estimate, state assumption, design to measure). "How many questions are too many?" (stop when answers wouldn't change a decision).

### 2.2 [Follow-up] How do you prioritise NFRs when everything is "critical"?
**What the interviewer is really testing**: Forcing ranking.
**Typical mid-level answer**: "Ask the business to prioritise." Falls short: passive, no technique.
**Strong senior answer**: I force-rank: show that each NFR has a price. E.g. 99.99% availability roughly means multi-region and a much larger ops burden vs 99.9% with zone redundancy. I use a simple scoring of business impact x likelihood, or ask "if you could only have two of these in the first release, which?" I also separate *must for launch* from *must within a year*. Decision criteria: regulatory > safety/data loss > revenue-impacting > convenience. I record the ranked list; validate in review with product and ops. Reversible: reprioritise per quarter using real telemetry.
**Follow-up traps**: "Isn't security always first?" (it's a gate, but depth is risk-proportional). "What if sponsors disagree?" (escalate with cost options).

### 2.3 [Medium] Walk me through an ATAM-lite for an order-processing platform.
**What the interviewer is really testing**: Can you apply a method concisely.
**Typical mid-level answer**: Lists pros/cons of microservices. Falls short: not tied to scenarios or sensitivity points.
**Strong senior answer**: Step 1: top scenarios — (a) 10x seasonal spike with checkout p95 < 500 ms, (b) no order lost on component failure, (c) six engineers on-call, (d) change tax rules without redeploying everything. Step 2: candidates — modular monolith on App Service + Azure SQL, or services on Container Apps with Service Bus. Step 3: compare per scenario. Spike favours independent scaling of checkout (Container Apps/KEDA) but a monolith can scale out horizontally too; no-loss favours transactional outbox + Service Bus either way. On-call favours fewer moving parts. Sensitivity point: SQL DTU/vCore ceiling on the shared database. Trade-off point: service split improves modifiability but hurts operability and consistency. Decision: modular monolith with outbox and a clear seam for later extraction; ADR lists the risks and triggers to split (e.g. a module needing different scale or team ownership).
**Follow-up traps**: "How long should this take?" (hours, not weeks). "Who attends?" (devs, ops, product, security).

### 2.4 [Tricky] Requirements change halfway; how do you keep NFR trade-offs from silently rotting?
**What the interviewer is really testing**: Continuous verification of NFRs.
**Typical mid-level answer**: "Update the documentation." Falls short: docs don't enforce anything.
**Strong senior answer**: Turn NFRs into executable checks: load tests in CI on a schedule (Azure Load Testing), SLO dashboards and alerts, architecture tests for structure, cost budgets/alerts in Azure Cost Management. When a requirement changes, I re-run the scenarios and see which decisions are now wrong, then supersede ADRs. This makes drift visible rather than discovered in an incident. Trade-off: effort in building checks; I'd automate only top-ranked attributes.
**Follow-up traps**: "What about attributes that can't be measured?" (proxy metrics, reviews). "Who owns the dashboards?" (the team that is on-call).

### 2.5 [Scenario] A stakeholder insists on "five nines". How do you respond?
**What the interviewer is really testing**: Pushing back with data, diplomatically.
**Typical mid-level answer**: "Sure, use multi-region." Falls short: no cost conversation.
**Strong senior answer**: 99.999% is ~5 minutes of downtime per year; you can't achieve that if dependencies (e.g. a single-region SaaS or the identity provider) are lower, because composite availability is the product of components. I'd ask what an hour of downtime costs and what recovery is acceptable, then present tiers: 99.9% zone-redundant single region (cost X), 99.95-99.99% active/passive multi-region with Azure SQL failover groups and Front Door (cost ~2x plus operations), and active/active (higher still, with conflict resolution). Compare to business impact. Usually the decision lands on a tier with practiced failover. I validate with game days/chaos tests, because untested failover isn't availability.
**Follow-up traps**: "Do SLAs equal SLOs?" (no; SLA is contractual, SLO internal target). "Planned maintenance counts?" (define it).

**Red flags in answers**
- Starting with technology choices before asking about load, risk, constraints.
- "We'll make it scalable" with no scenarios.
- Treating all NFRs as equally important.

**Cheat sheet**
- Quality-attribute scenario: source/stimulus/environment/response/measure.
- ATAM-lite: rank scenarios, compare candidates, find sensitivity and trade-off points, record risks.
- Force-rank; every NFR has a price.
- Composite availability = product of dependencies.
- Automate checks for the top NFRs.
- Keep an assumptions log.

---

## Topic 3: Architecture Decision Records (ADRs)

**Study notes**
- ADR = short, immutable record of one significant decision: context, options, decision, consequences. Stored in the repo (`/docs/adr/0007-use-service-bus.md`) so it's versioned with code and reviewed via PR.
- Write one when the decision is costly to reverse, affects multiple teams, introduces a new technology/pattern, or someone will later ask "why?".
- Don't write for trivial or easily reversible choices.
- Statuses: Proposed -> Accepted -> Deprecated / Superseded by ADR-00XX. Never edit history; supersede.

```markdown
# ADR-000N: <Short title in the form of a decision>
Status: Proposed | Accepted | Superseded by ADR-00XX
Date: YYYY-MM-DD   Deciders: <names/roles>
## Context
Forces, constraints, NFRs, what triggered this.
## Options considered
1. Option A – pros/cons
2. Option B – pros/cons
## Decision
We will ...
## Consequences
Positive, negative, risks, follow-up actions, how we'll know it's wrong (revisit trigger).
```

### 3.1 [Easy] What is an ADR and why bother?
**What the interviewer is really testing**: Understanding the purpose, not the format.
**Typical mid-level answer**: "A document that records decisions." Falls short: no value argument.
**Strong senior answer**: An ADR captures the *why* — context, options rejected, trade-offs — which code can't show. Without it, teams relitigate decisions, or reverse them blindly (Chesterton's fence). It also speeds onboarding and makes the decision reviewable before it's costly. I keep them short (one page), in the repo, with PR review so discussion is captured. Cost is low; value appears when the original decider leaves or conditions change. I'd validate adoption by seeing ADRs referenced in PRs and design discussions.
**Follow-up traps**: "Isn't it bureaucracy?" (only for significant decisions; lightweight). "Wiki vs repo?" (repo — close to code, versioned).

### 3.2 [Follow-up] When do you write one — and when not?
**What the interviewer is really testing**: Judgment on threshold.
**Typical mid-level answer**: "For every decision." Falls short: unsustainable, noise.
**Strong senior answer**: Threshold: hard to reverse (more than ~a sprint of rework), cross-team impact, new platform dependency, security/compliance implications, or a decision with real contention. Not for choosing a logging format or a NuGet helper behind an interface. A good heuristic is "would I be annoyed if a new hire undid this without knowing why?" I'd also write one when the *rejection* of an obvious option is non-obvious (e.g. "why not Cosmos DB?"). Validate by tracking: if there are more than a few per month per team, the threshold is too low.
**Follow-up traps**: "Retroactive ADRs?" (yes for inherited systems, mark as reconstructed). "Who approves?" (named deciders, not a committee).

### 3.3 [Medium] How do you review someone else's ADR?
**What the interviewer is really testing**: Review quality and constructive challenge.
**Typical mid-level answer**: "Check it reads fine." Falls short: no substantive checks.
**Strong senior answer**: I check: Is the problem stated separately from the solution? Are the forces/NFRs measurable? Were at least two real alternatives considered, including "do nothing"? Are the consequences honest, including negatives and operational cost? Is there a revisit trigger and a reversal/exit plan? Does it conflict with existing ADRs? I give feedback as questions ("what happens at 10x volume?") and flag blocking vs non-blocking comments. I time-box the review (e.g. 3-5 business days) so decisions don't stall. A decision with a weak alternatives section often signals the conclusion was reached first.
**Follow-up traps**: "What if you disagree but the team decided?" (record dissent, disagree and commit). "Who has veto?" (decider, as defined up front).

### 3.4 [Tricky] The ADR from two years ago is now wrong. What do you do?
**What the interviewer is really testing**: Handling superseding without blame.
**Typical mid-level answer**: "Edit it." Falls short: loses history.
**Strong senior answer**: Don't edit; write a new ADR that supersedes it, linking both, and mark the old as "Superseded by ADR-00XX". The new ADR explains what changed (scale, pricing, product maturity, team), which is the learning. Treat the old decision as correct for its context — blamelessness encourages honesty. Plan migration with cost/benefit; sometimes the right answer is to keep the old decision for existing parts and apply the new one only to new work. Validate by checking the supersede chain is discoverable from the index.
**Follow-up traps**: "Must we migrate everything?" (not necessarily). "What if the context hasn't changed but opinions have?" (high bar to supersede).

### 3.5 [Scenario] Your team keeps reopening settled decisions. Fix it.
**What the interviewer is really testing**: Process maturity.
**Typical mid-level answer**: "Tell them to stop." Falls short: ignores root cause.
**Strong senior answer**: First diagnose: do people not know the rationale, were they excluded, or did circumstances change? If the rationale is not visible, ADRs fix it. If people were excluded, include stakeholders earlier through a Proposed ADR with a comment window. Define reopening rules: new information (a changed NFR, incident, cost data) is a valid reason; preference is not. Include revisit triggers in the ADR so reopening is a scheduled, legitimate event. Validate by tracking how many reopenings arise with new evidence.
**Follow-up traps**: "Isn't that stifling?" (it focuses debate on evidence). "What about junior challengers?" (welcome, same rules).

**Red flags in answers**
- ADRs as post-hoc justification, written after implementation with no alternatives.
- Editing old ADRs rather than superseding.
- ADRs only in a wiki nobody finds.

**Cheat sheet**
- One decision per ADR; in repo; PR-reviewed.
- Context, options, decision, consequences, revisit trigger.
- Threshold: costly to reverse, cross-team, contentious.
- Supersede, never rewrite history.
- Review for alternatives and honest negatives.
- Time-box decisions.

---

## Topic 4: Reversible (two-way door) vs irreversible decisions

**Study notes**
- Two-way door: cheap to undo; decide fast, with the person closest to the work. One-way door: costly/impossible to undo; slow down, gather evidence, wider review.
- Technique: convert one-way doors into two-way doors by adding seams (interfaces/anti-corruption layers, feature flags, expand/contract migrations, versioned contracts).
- Costs of delay are real: over-analysing two-way doors is a failure mode too.

| Decision | Door | Making it more reversible |
|---|---|---|
| Logging library | Two-way | ILogger abstraction |
| ORM choice | Mostly two-way if behind repository | keep SQL schema independent |
| Public API shape | One-way (consumers) | versioning, deprecation policy |
| Primary datastore | One-way | outbox/CDC, export paths |
| Event schema | One-way | schema registry, additive-only changes |
| Region/tenancy model | One-way | design tenant id in from day one |

### 4.1 [Easy] What is a one-way vs two-way door decision?
**What the interviewer is really testing**: Using a mental model to calibrate decision effort.
**Typical mid-level answer**: "Some decisions are reversible." Falls short: no behavioural consequence.
**Strong senior answer**: Two-way doors can be walked back cheaply, so I decide quickly, often delegated, and learn from production. One-way doors need more evidence, wider review, and an ADR. The skill is classifying correctly and spending effort proportionally. Cost of reversal is measured in team-weeks, data migration risk, and external commitments. I'd validate classification by asking "who else is affected if we reverse?".
**Follow-up traps**: "Who coined it?" (a common framing popularised by Amazon's leadership; don't over-attribute). "Can all be two-way?" (no).

### 4.2 [Follow-up] How do you make an irreversible decision safer?
**What the interviewer is really testing**: Risk reduction techniques.
**Typical mid-level answer**: "Think carefully." Falls short: no tactics.
**Strong senior answer**: Narrow the blast radius and buy information first. Tactics: a time-boxed spike or proof of concept against the riskiest assumption; a thin vertical slice in production behind a flag; parallel run/shadow traffic; contract versioning; expand/contract schema migrations on Azure SQL; keeping an exit path (data export, abstraction at the edges). Then decide at the last responsible moment — when delay costs more than the remaining uncertainty. Record assumptions in the ADR so you can detect when they fail.
**Follow-up traps**: "Last responsible moment vs procrastination?" (it's when options would close or cost rises). "Is abstraction always the answer?" (no; leaky abstractions cost).

### 4.3 [Medium] You must pick a public API versioning approach. Which door is it?
**What the interviewer is really testing**: Spotting external commitments.
**Typical mid-level answer**: "Use v1 in the URL." Falls short: no trade-off analysis.
**Strong senior answer**: Once clients depend on it, it's one-way, because you can't force consumers to change. Options: URL segment (/v1/), header, or query param; ASP.NET API Versioning supports all. URL is most discoverable and cache-friendly; headers are cleaner but hurt debugging. More important than mechanism: policy — additive changes are non-breaking, breaking changes need a new major, deprecation window (e.g. 6-12 months), telemetry on version usage. Decision: URL segment, contract tests (Pact or OpenAPI diff in CI), published deprecation policy. Validate via OpenAPI breaking-change checks in the pipeline.
**Follow-up traps**: "How do you detect a breaking change?" (diff tooling, consumer-driven contracts). "What about internal APIs?" (still versioned if multiple deploy cadences).

### 4.4 [Tricky] Is choosing a microservices architecture a one-way door?
**What the interviewer is really testing**: Nuance on direction of reversal.
**Typical mid-level answer**: "Yes/no." Falls short: no asymmetry.
**Strong senior answer**: It's asymmetric. Splitting a well-structured monolith into services later is feasible if module boundaries and data ownership were clean; merging badly-cut services back is painful because of distributed data and contracts. Starting with a modular monolith keeps the door open and defers distribution costs (network failures, observability, deployments) until justified. Triggers to split: independent scaling, team autonomy bottleneck, different release cadence, fault isolation. I'd enforce module boundaries with architecture tests so the option stays real.
**Follow-up traps**: "Isn't a monolith a risk too?" (yes if boundaries erode). "Shared database between services?" (kills independence).

### 4.5 [Scenario] The team wants to adopt a new NoSQL database for one feature. Walk through your decision.
**What the interviewer is really testing**: Risk-weighted adoption.
**Typical mid-level answer**: "Let them try it." Falls short: ignores operations and data ownership.
**Strong senior answer**: Context: is the access pattern a poor fit for SQL Server, or is it novelty? Options: stay on Azure SQL (JSON columns, indexes, columnstore), Cosmos DB, or something else. Criteria: access pattern, consistency, query flexibility, ops skill, backup/DR, cost model (RU vs vCore), compliance. Adding a datastore adds a permanent operational surface, so it's closer to one-way. I'd require a spike with realistic data and load, an ownership/on-call plan, and an exit path (events or CDC to rebuild elsewhere). Decision comes with an ADR. If the benefit is marginal, I'd stay put.
**Follow-up traps**: "Polyglot persistence?" (justified per need, not per preference). "Who runs it?" (must be named).

**Red flags in answers**
- Applying the same heavy process to every decision.
- Claiming everything is reversible "because we're agile".
- No exit or rollback path for data-store changes.

**Cheat sheet**
- Classify the door before choosing the process.
- Create seams to turn one-way into two-way.
- Spike the riskiest assumption first.
- Data and public contracts are the one-way doors.
- Decide at the last responsible moment.
- Write assumptions down to detect when they break.

---

## Topic 5: Build vs buy vs open source; technology selection; lock-in

**Study notes**
- Build when it's a **differentiator** and you can sustain it; buy for commodity (identity, email, payments); open source when community health and your ability to maintain it are strong.
- Total cost of ownership: licence + integration + operation + upgrades + opportunity cost + exit cost. Build cost is usually underestimated (maintenance is 60-80% of lifecycle — treat as a rule of thumb, not a fact).

| Criterion | Build | Buy (SaaS/COTS) | Open source |
|---|---|---|---|
| Fit | Exact | Approximate | Variable |
| Time to value | Slow | Fast | Medium |
| Control | High | Low | High (with maintenance) |
| Ongoing cost | Team | Subscription | Team + support |
| Exit cost | n/a | Data/contract lock-in | Fork/replace |

- Technology selection framework: weighted criteria (fit to NFRs, team skills, ecosystem/community, vendor support/SLAs, security/compliance, cost, maturity, exit cost) -> shortlist -> time-boxed spike/PoC on the riskiest assumption -> decide -> ADR.
- Lock-in reality: all choices lock you in somewhere (language, cloud, database engine). Abstracting everything creates lowest-common-denominator designs and cost. Pay for portability only where the probability x cost of switching justifies it.

### 5.1 [Easy] How do you decide build vs buy?
**What the interviewer is really testing**: Strategic reasoning about differentiation.
**Typical mid-level answer**: "Buy if it exists, it's cheaper." Falls short: ignores integration, fit, and exit.
**Strong senior answer**: First ask whether the capability differentiates the business. If not (auth, e-mail, payments, document signing), default to buy — e.g. Entra ID/B2C-type services rather than a custom identity store. If it is core, build, but staff it for its lifecycle. Compare over 3-5 years TCO including integration, customisation limits, upgrade cycles, support, and data portability. Consider a hybrid: buy the platform, build the differentiating layer. Validate with a spike against the 3 riskiest requirements, and negotiate data-export terms. Reverse by retaining your domain model separate from the vendor's.
**Follow-up traps**: "What if the vendor is acquired?" (exit clauses, export). "Do you count your own salary?" (yes, opportunity cost).

### 5.2 [Follow-up] Describe your technology selection process.
**What the interviewer is really testing**: Repeatable, evidence-based method.
**Typical mid-level answer**: "Compare features and pick the most popular." Falls short: popularity isn't fit.
**Strong senior answer**: Start with the problem and ranked NFRs, then derive weighted criteria: fit, team skills, maturity, community/ecosystem, vendor support, security posture, licensing, cost, observability, exit cost. Shortlist 2-3, not 10. Run a time-boxed spike (days) that targets the riskiest unknown — throughput, a hard integration, failure behaviour — not a hello world. Score, discuss risks, and decide with an ADR including revisit trigger. Team skills matter: a 'better' tool the team can't operate is worse. Reversal: keep adoption narrow initially (one service) before standardising.
**Follow-up traps**: "How long should a spike take?" (time-boxed, e.g. 2-5 days, with a clear question). "Weighted scores subjective?" (yes; the point is to surface disagreement).

### 5.3 [Medium] Vendor lock-in: how much should we worry on Azure?
**What the interviewer is really testing**: Realism vs fear.
**Typical mid-level answer**: "Avoid it; use Kubernetes and abstractions everywhere." Falls short: pays for portability that's never used.
**Strong senior answer**: Lock-in is a cost to weigh, not a sin. Options: embrace managed services for speed and ops savings, or abstract for portability. Criteria: probability of leaving (regulatory, price, M&A), cost of switching, and benefit gained from native features. In practice the stickiest parts are data and identity, not compute. I'd use portable standards where free (containers, OpenTelemetry, SQL, AMQP, S3-like blob patterns, standard HTTP APIs), and accept native services (Service Bus, managed identity) where they deliver real value. Keep an exit path for data (export, CDC) and isolate vendor SDKs behind a thin boundary. Multi-cloud for its own sake doubles ops complexity. Validate via an exit-cost estimate in the ADR.
**Follow-up traps**: "Wouldn't Kubernetes make us portable?" (compute yes, but you still depend on managed DBs, identity, networking). "Concentration risk?" (regulated sectors may require an exit plan).

### 5.4 [Tricky] The team wants to build an in-house workflow engine. Respond.
**What the interviewer is really testing**: Spotting NIH (not-invented-here).
**Typical mid-level answer**: "Sounds fun, let's do it." Falls short: ignores total cost.
**Strong senior answer**: I'd ask what requirements existing options can't meet: Durable Functions, Logic Apps, Temporal, Elsa, or a saga via MassTransit/Wolverine. Workflow engines look simple until you need versioning of in-flight instances, timers, retries, compensation, visibility, and tooling — each is a long tail. Criteria: differentiating? Number of workflows? Who maintains? I'd propose a spike with an existing engine on the hardest workflow, and only build if gaps are demonstrated. If building, scope narrowly and plan ownership. Record as ADR.
**Follow-up traps**: "What if the licence is restrictive?" (include licence in the evaluation). "What about skills?" (hiring pool).

### 5.5 [Scenario] A SaaS vendor offers a faster path but holds your data. What do you do?
**What the interviewer is really testing**: Risk and contract thinking.
**Typical mid-level answer**: "Take it; speed matters." Falls short: no exit analysis.
**Strong senior answer**: Evaluate data classification and regulation first (residency, DPA, encryption, audit). Then exit: data export format, API access, frequency, termination assistance, and the cost to rebuild. Options: adopt with scheduled exports into our own store, adopt only for a non-critical domain, or build. I'd involve legal/security, ask for a security review (SOC2/ISO reports), and run a pilot. Decision weighs time-to-market gain against exit and compliance risk. Reversal: nightly export into a data lake keeps our option alive.
**Follow-up traps**: "What if the vendor goes down?" (SLA, degraded mode). "Pricing growth?" (model at 5x usage).

**Red flags in answers**
- Building core-irrelevant commodity capabilities.
- Choosing technology on popularity or personal preference.
- Either "never lock in" or "lock-in doesn't matter".

**Cheat sheet**
- Build differentiators, buy commodity.
- TCO includes maintenance and exit.
- Spike the riskiest assumption, time-boxed.
- Lock-in is a priced trade-off; data and identity are stickiest.
- Prefer open standards where cheap (containers, OpenTelemetry, SQL).
- Document exit path in the ADR.

---

## Topic 6: Choosing .NET libraries (licensing & maintenance risk)

**Study notes**
- Library risk factors: licence terms, maintainer bus factor, release cadence, issue responsiveness, security advisories, .NET version support, transitive dependencies, and how deeply it infects your code.
- Licensing changes happened recently in the .NET ecosystem: MediatR, AutoMapper (same author) and MassTransit moved to commercial/dual licensing models in 2025 for newer versions. Verify current terms and pricing at decision time — do not rely on memory; versions you already use under older licences generally continue under their licence.
- Alternatives (check fit and maturity): Mapperly (compile-time mapping via source generators) or hand-written mapping; plain handlers/vertical slices or other mediator libraries (e.g. Wolverine, Mediator source-generated) for MediatR; Wolverine, Brighter, NServiceBus (commercial), or raw Azure Service Bus SDK for MassTransit.
- Dependency health checks: `dotnet list package --vulnerable --include-transitive`, `--outdated`, `--deprecated`; NuGet audit, Dependabot/Renovate, SBOM, licence scanners; OpenSSF Scorecard for open-source health signals.
- Mitigate coupling: confine a library to a boundary (adapter), avoid its types in your domain.

### 6.1 [Easy] What do you check before adding a NuGet package?
**What the interviewer is really testing**: Basic supply-chain hygiene.
**Typical mid-level answer**: "Download count and it works." Falls short: ignores maintenance, licence, security.
**Strong senior answer**: Check: licence compatibility with our distribution model; last release and cadence; open issues responsiveness; maintainer count; known vulnerabilities (NuGet audit); .NET target support; transitive dependency weight; size of the API surface we'd depend on. Ask: could we write this in a day? If so, maybe don't. Also consider how contained the dependency will be. Add via an approved-package process or central package management (Directory.Packages.props), pinned, with automated update PRs. Reverse by keeping it behind an adapter.
**Follow-up traps**: "Stars = quality?" (no). "Package signing?" (verify, use trusted signers/lock files).

### 6.2 [Follow-up] MediatR, AutoMapper or MassTransit moved to commercial licensing. What do you do?
**What the interviewer is really testing**: Calm risk assessment over panic.
**Typical mid-level answer**: "Replace it immediately" or "ignore it". Falls short: both extremes lack analysis.
**Strong senior answer**: First, read the actual licence and our usage: which versions, how many services, what the price is relative to engineering time. Options: pay, pin the last open-source version (with security-patch risk), migrate to an alternative, or remove the need (e.g. replace mapping with explicit methods or Mapperly; replace mediator with direct handler interfaces). Criteria: cost of migration vs licence cost vs risk of unpatched dependency. Reasonable decision for many teams: pin now, stop new usage, and plan incremental removal opportunistically — unless the library is deeply woven in and paying is cheaper. Record in an ADR, set a review date. Validate by dependency scans showing where the library is used.
**Follow-up traps**: "Is MediatR needed at all?" (often it is indirection; judge per codebase). "Can you fork?" (maintenance burden, licence permitting).

### 6.3 [Medium] How do you decide whether to wrap a third-party library behind your own interface?
**What the interviewer is really testing**: Pragmatic abstraction.
**Typical mid-level answer**: "Always wrap everything." Falls short: leads to useless indirection.
**Strong senior answer**: Wrap when the dependency is volatile, commercial-risk, at an architectural boundary (messaging, storage, payments), or hard to fake in tests. Don't wrap stable, ubiquitous things (ILogger is already an abstraction; System.Text.Json). Criteria: probability of replacement, size of API surface, leakage of library types into domain. Trade-off: wrappers cost maintenance and may hide useful features. For a messaging library, I'd define my own `IEventPublisher` capturing what the domain needs, with the library in the infrastructure project. Validate with architecture tests that domain assemblies don't reference the package.
**Follow-up traps**: "Isn't that speculative generality?" (only wrap where change is plausible). "Wrapper for EF Core?" (debatable; repository-over-EF is contested).

### 6.4 [Tricky] A critical open-source library has one maintainer and no release in 18 months. Keep using it?
**What the interviewer is really testing**: Weighing risk vs stability.
**Typical mid-level answer**: "Replace it, it's dead." Falls short: stable libraries may need no releases.
**Strong senior answer**: Inactivity isn't automatically a problem; check whether it's feature-complete, whether issues include unpatched security bugs, whether it targets current .NET and its dependencies are current. Options: continue, contribute/sponsor, fork internally, or migrate. Criteria: criticality, replaceability, security exposure, our capacity. If it's on a security boundary (auth, crypto, parsing untrusted input), the bar is higher. Decision: for low-exposure stable code, keep with monitoring; for high-exposure, plan replacement. Add to a risk register with a trigger (e.g. a CVE with no fix in 30 days).
**Follow-up traps**: "How do you monitor?" (Dependabot, advisories). "Sponsoring?" (reasonable for critical dependencies).

### 6.5 [Scenario] Your org wants a standard approved-library list. Design it lightly.
**What the interviewer is really testing**: Governance that doesn't block.
**Typical mid-level answer**: "A committee approves each package." Falls short: bottleneck.
**Strong senior answer**: Use tiers: pre-approved defaults (Microsoft, widely-used permissive libs), "allowed with notification" (scan passes automatically), and "needs review" (copyleft/commercial licences, security-sensitive). Automate with licence/vulnerability scanning in CI, central package management, and a private feed or Azure Artifacts upstream with policy. Provide a fast path (days) and a documented rationale. The aim is guardrails and visibility, not gatekeeping. Review quarterly and publish the paved-road list. Validate via lead time for a new package and number of unapproved packages found.
**Follow-up traps**: "GPL?" (legal review). "Who owns the list?" (platform team with legal/security input).

**Red flags in answers**
- Adding packages without checking licence or maintenance.
- Assuming popular = safe.
- Wrapping every dependency "just in case".

**Cheat sheet**
- Check licence, cadence, maintainers, vulnerabilities, .NET support.
- Verify current licensing terms of MediatR/AutoMapper/MassTransit before deciding.
- Contain risky dependencies behind adapters.
- Use `dotnet list package --vulnerable --deprecated` plus Dependabot/Renovate.
- Central package management; pin versions.
- Tiered approval, automated.

---

## Topic 7: Cloud compute choices — PaaS vs containers vs serverless

**Study notes**
- Spectrum: more managed = less control, less ops. App Service (PaaS web) -> Functions (serverless/event) -> Container Apps (serverless containers, KEDA scaling, Dapr optional, scale to zero) -> AKS (full Kubernetes) -> VMs.

| Option | Best for | Strengths | Costs/limits |
|---|---|---|---|
| App Service | Standard web APIs/apps | Simple, deployment slots, autoscale | Less flexible runtime; per-plan cost |
| Azure Functions | Event-driven, glue, bursty | Triggers/bindings, scale to zero (Consumption/Flex) | Cold starts (plan-dependent), execution limits, harder local-state |
| Container Apps | Microservices/background workers without K8s ops | KEDA scaling, revisions, Dapr, any container | Fewer K8s knobs |
| AKS | Many services, custom networking/operators, platform team | Full control, ecosystem | High ops burden; need platform skills |

- Decision drivers: workload shape (steady vs bursty), team ops capacity, startup latency tolerance, networking/compliance needs, portability, and cost model (always-on vs per-execution).
- Default heuristic: pick the **most managed option that meets requirements**; move down only for a demonstrated need.
- Cost: steady high load often cheaper on reserved/dedicated plans; spiky or idle workloads cheaper on consumption.

### 7.1 [Easy] How do you choose between App Service, Functions and Container Apps?
**What the interviewer is really testing**: Workload-to-platform matching.
**Typical mid-level answer**: "Functions for everything small, AKS for big." Falls short: no criteria.
**Strong senior answer**: Start with workload shape. A standard request/response API that runs steadily fits App Service: simple, mature, slots for zero-downtime deploys. Event-triggered, short-lived, bursty work (queue consumers, timers, webhooks) fits Functions. Multiple containerised services or long-running workers with event-based scaling fit Container Apps. I'd check constraints: execution duration, cold-start tolerance, VNet needs, runtime customisation, team's container skills. Choose the most managed option satisfying them and record the revisit trigger (e.g. need for custom operators -> AKS). Validate with a load test and cost model.
**Follow-up traps**: "Isn't containers more portable?" (yes but the app is mostly portable anyway on .NET). "Functions for APIs?" (possible, but consider the trade-offs).

### 7.2 [Follow-up] When is AKS actually justified?
**What the interviewer is really testing**: Resisting resume-driven design.
**Typical mid-level answer**: "For microservices." Falls short: Container Apps also does.
**Strong senior answer**: AKS pays off when you need what the abstraction hides: custom CNI/network policies, operators/CRDs, service mesh choices, specialised node pools (GPU, spot), strict multi-tenancy controls, or you already run a platform team that treats Kubernetes as a product. Costs: upgrades, node patching, security hardening, cluster add-ons, on-call skills. Criteria: number of services, need for control, available platform expertise, compliance. If none apply, Container Apps gives most benefits with far less ops. I'd decide with an ADR and an honest staffing estimate; reverse by keeping workloads as standard containers and Helm/manifests portable.
**Follow-up traps**: "Is AKS cheaper?" (not once ops is counted). "Can we start on Container Apps and move?" (yes, containers move).

### 7.3 [Medium] Functions Consumption vs Premium/Flex vs Container Apps for a queue-processing workload with spikes.
**What the interviewer is really testing**: Cost, cold start, and scaling nuance.
**Typical mid-level answer**: "Functions scale automatically." Falls short: ignores concurrency, downstream limits.
**Strong senior answer**: Context: Service Bus queue, bursts of 50k messages, downstream Azure SQL with connection limits. Options: Functions Consumption (cheap, scale to zero, cold starts), Flex Consumption/Premium (pre-warmed instances, VNet), Container Apps with KEDA Service Bus scaler (more control over concurrency, long-running). Key criterion is not raw scale but *controlled* scale: unbounded scale-out can overwhelm SQL, so set max instances and message concurrency. Pick Functions Flex/Premium if VNet and low latency needed; Container Apps if processing is long or needs custom runtime. Validate with load test measuring DB DTU/CPU, queue age, and cost per million messages. Verify current plan features in documentation before finalising.
**Follow-up traps**: "Poison messages?" (DLQ, max delivery count). "Idempotency?" (at-least-once delivery requires it).

### 7.4 [Tricky] Serverless is "pay per use", so always cheaper. True?
**What the interviewer is really testing**: Cost modelling beyond slogans.
**Typical mid-level answer**: "Yes, you only pay when it runs." Falls short: ignores steady load and hidden costs.
**Strong senior answer**: Not always. At low/spiky usage consumption pricing wins; at sustained high utilisation, dedicated or reserved capacity is usually cheaper per unit of work. Hidden costs: premium plan always-on instances, networking (NAT, private endpoints), monitoring ingestion, downstream scaling, and debugging complexity. I'd model unit cost at current and 10x load for each option, include engineer time, then pick. Revisit when usage stabilises; moving a function app to a container or App Service plan can be an incremental change if business logic is isolated from triggers.
**Follow-up traps**: "How to keep logic portable?" (separate handlers from trigger glue). "Reserved instances?" (commit only when stable).

### 7.5 [Scenario] Greenfield internal API for ~200 requests/sec, team of five, no Kubernetes experience.
**What the interviewer is really testing**: Matching platform to team capacity.
**Typical mid-level answer**: "Microservices on AKS for future scale." Falls short: ops mismatch.
**Strong senior answer**: Load is modest; the constraint is team capacity. Choose App Service (or Container Apps if containers are already the team's standard) with Azure SQL, managed identity, App Insights, and slots/blue-green. Over-investing in AKS would consume the team's capacity in platform work. Keep the code in a modular structure and container-ready so a move is cheap. Revisit triggers: more than ~N independently deployed services, need for sidecar/mesh, or custom networking. Validate through a load test at 5x and a cost model.
**Follow-up traps**: "What if the org mandates AKS?" (use the platform team's paved road). "Autoscale rules?" (CPU plus request-based).

**Red flags in answers**
- Choosing AKS by default or for CV reasons.
- "Serverless is cheaper" with no model.
- Ignoring downstream limits when scaling out.

**Cheat sheet**
- Most managed option that meets requirements.
- Workload shape + team ops capacity decide.
- Controlled scale: cap instances and concurrency.
- Model unit cost at 1x and 10x.
- Separate business logic from trigger/hosting glue.
- Verify current plan features before committing.

---

## Topic 8: Evolutionary architecture, fitness functions, architecture tests

**Study notes**
- **Evolutionary architecture** supports guided, incremental change across multiple dimensions. **Fitness function** = an objective, automatable check that an architectural characteristic still holds.
- Types: atomic (unit-test-like, e.g. no Domain -> Infrastructure reference) vs holistic (system-level, e.g. p95 latency in load test); triggered (CI) vs continuous (production monitoring).
- Examples in .NET: **NetArchTest.Rules** or **ArchUnitNET** to assert layering/dependency rules; analyzers for banned APIs; cyclic dependency checks; load tests in pipeline; cost alerts; OpenAPI breaking-change diffs; `dotnet list package --vulnerable` as a security fitness function.

```csharp
[Fact]
public void Domain_ShouldNotDependOn_Infrastructure()
{
    var result = Types.InAssembly(typeof(Order).Assembly)
        .That().ResideInNamespace("MyApp.Domain")
        .ShouldNot().HaveDependencyOn("MyApp.Infrastructure")
        .GetResult();
    Assert.True(result.IsSuccessful);
}
```
- Principle: if an architectural rule matters, enforce it automatically; review-only rules decay.

### 8.1 [Easy] What is a fitness function?
**What the interviewer is really testing**: Can you make architecture testable.
**Typical mid-level answer**: "A test for performance." Falls short: too narrow.
**Strong senior answer**: A fitness function objectively measures how well the system meets an architectural goal, run automatically so drift is caught early. Examples: dependency-rule tests with NetArchTest, a build failing on a new circular dependency, a pipeline load test asserting p95 < 300 ms, a vulnerability scan gate, a cost budget alert. They convert intent in ADRs into guardrails. I'd choose few, tied to the top-ranked NFRs, since each costs maintenance. Validate by checking they fail when the rule is deliberately broken.
**Follow-up traps**: "Atomic vs holistic?" (single characteristic vs interacting ones). "Can all qualities be tested?" (not all; use proxies).

### 8.2 [Follow-up] How would you enforce a modular monolith's boundaries?
**What the interviewer is really testing**: Practical enforcement.
**Typical mid-level answer**: "Code reviews." Falls short: not scalable.
**Strong senior answer**: Make boundaries compile-time where possible: separate projects per module exposing a small public contract (interfaces/DTOs/events) with `internal` implementation. Add architecture tests asserting modules reference each other only through contracts, and no module reads another's tables (separate schemas per module, with DB permissions where feasible). Integrate in CI so violations fail the build. Trade-off: some ceremony, and tests need upkeep as modules evolve. This keeps the later extraction option cheap. Validate by attempting an illegal reference in a PR and confirming it fails.
**Follow-up traps**: "Cross-module queries?" (via APIs or read models). "Test false positives?" (keep rules simple and focused).

### 8.3 [Medium] Where do you draw the line on architecture tests — which rules deserve automation?
**What the interviewer is really testing**: Cost-benefit judgment.
**Typical mid-level answer**: "Automate everything." Falls short: brittle test suite.
**Strong senior answer**: Automate rules that are costly if violated, frequently violated, and objectively checkable: layering, forbidden dependencies, public API surface, naming where it drives tooling, security (no secrets in config), performance budgets. Leave subjective rules (is this cohesive?) to review. Criteria: violation cost, detectability, false-positive risk. Start with 3-5 rules from existing ADRs; each new incident adds a rule if recurring. Track test failures to see if rules help or just annoy; remove noise.
**Follow-up traps**: "Who fixes failing rules?" (the author of the change). "Exceptions?" (explicit allow-list with reason).

### 8.4 [Tricky] Does evolutionary architecture mean no up-front design?
**What the interviewer is really testing**: Balanced view.
**Typical mid-level answer**: "Yes, evolve as you go." Falls short: ignores one-way doors.
**Strong senior answer**: No. It means designing so that change is cheap and guided. Up-front effort concentrates on one-way doors (data ownership, contracts, tenancy) and on the fitness functions that protect the intended direction. Everything else evolves. Without fitness functions, "evolution" is drift. I'd define a target direction, a few guardrails, and iterate in thin slices, validating assumptions with production data.
**Follow-up traps**: "Isn't this just agile?" (it adds explicit architectural governance via automation). "Risk?" (guardrails ossify; review them).

### 8.5 [Scenario] Latency has regressed three releases in a row without anyone noticing. Fix it systemically.
**What the interviewer is really testing**: Continuous verification.
**Typical mid-level answer**: "Add monitoring." Falls short: reactive, no gate.
**Strong senior answer**: Add a holistic fitness function: scheduled load test (Azure Load Testing) with a baseline and pass/fail thresholds on p95 for key endpoints, plus SLO alerts in production via Application Insights. Add atomic checks for common causes: EF query count tests for hot paths, banned synchronous-over-async patterns via analyzers. For releases, canary/slot swap with automatic rollback if latency SLO burns. Trade-off: test environment cost and noise; use relative thresholds against a baseline. Validate by replaying the regression and confirming the gate catches it.
**Follow-up traps**: "Noisy tests?" (multiple runs, percentile ranges). "Prod vs test parity?" (limit; use canary).

**Red flags in answers**
- Architecture rules that live only in a wiki.
- Dozens of brittle tests nobody trusts.
- Equating evolutionary with "no planning".

**Cheat sheet**
- Fitness function = automated check of an architectural goal.
- NetArchTest/ArchUnitNET for dependency rules.
- Automate costly, frequent, objective rules.
- Atomic vs holistic; triggered vs continuous.
- Guardrails protect one-way doors.
- Verify rules fail when violated.

---

## Topic 9: Managing technical debt

**Study notes**
- Debt types (Fowler's quadrant): deliberate/inadvertent x prudent/reckless. Deliberate prudent debt ("ship now, fix after launch") is a legitimate tool when interest and repayment are understood.
- Quantify: interest = recurring cost (extra hours per change, incident count, onboarding delay, cloud waste); principal = cost to fix. Use DORA-style signals (lead time, change failure rate, MTTR), hotspot analysis (churn x complexity), defect clustering, and developer survey friction.
- Prioritise by (interest x blast radius x proximity to upcoming work) / principal.
- Tactics: boy-scout rule (leave code better), 15-20% capacity allocation (a rule of thumb, tune it), dedicated refactor slices attached to features, big refactors only with clear payoff and incremental delivery.
- Communicate to product in business terms: delivery speed, risk, cost, not "clean code".

### 9.1 [Easy] What is technical debt and is it always bad?
**What the interviewer is really testing**: Mature view.
**Typical mid-level answer**: "Bad code we should fix." Falls short: no concept of interest or intent.
**Strong senior answer**: Debt is a shortcut that speeds delivery now at a recurring cost later. It's like a loan: fine if deliberate, tracked, and the interest is affordable, e.g. launching with a manual reconciliation step to hit a regulatory date. Reckless debt (no one knows it exists) is the problem. I'd ensure deliberate debt is logged with its interest, owner, and a repayment trigger. Validate by reviewing the register quarterly. Debt also arises inadvertently as knowledge grows, which is normal.
**Follow-up traps**: "Is a missing test debt?" (yes, usually). "Is old code debt?" (only if it causes cost).

### 9.2 [Follow-up] How do you quantify debt so leadership will listen?
**What the interviewer is really testing**: Translate to business impact.
**Typical mid-level answer**: "Show them code smells." Falls short: not business language.
**Strong senior answer**: Convert to cost and risk. Measure change lead time in the debt-heavy area vs elsewhere, incidents attributed to it, hours lost per sprint to workarounds, and onboarding time. Use churn x complexity hotspots from git history and static analysis to find where debt actually bites. Frame: "this module costs ~X engineer-days per sprint and caused Y incidents; fixing costs Z and pays back in N months". Estimates are approximate; state confidence. Validate post-fix by re-measuring lead time and incidents.
**Follow-up traps**: "SonarQube debt minutes reliable?" (rough indicator, not business truth). "Can't measure?" (use proxies and ask engineers for time diaries).

### 9.3 [Medium] Boy-scout rule vs a big refactor — how do you choose?
**What the interviewer is really testing**: Choosing strategy by situation.
**Typical mid-level answer**: "Big refactor when it's bad." Falls short: risky and often unfunded.
**Strong senior answer**: Boy-scout suits diffuse, low-risk debt in code we touch often: small, safe improvements within feature PRs. Big refactors suit structural debt (wrong boundaries, a pervasive anti-pattern) where small steps can't fix it. Even then, I'd prefer incremental large-scale change: parallel change, branch by abstraction, strangler, with each step shippable. Criteria: interest rate, risk, test coverage (add characterization tests first), proximity to roadmap work. Never a months-long branch. Validate with continuous delivery of each step and metrics improving.
**Follow-up traps**: "No tests?" (write characterization tests first). "Who approves capacity?" (agree a standing allocation with product).

### 9.4 [Tricky] Product says "no time for debt, features first". Response?
**What the interviewer is really testing**: Influence without confrontation.
**Typical mid-level answer**: "Do it secretly" or "give up". Falls short: both erode trust.
**Strong senior answer**: Make the trade-off visible rather than argue principle. Show that slowdowns in feature X trace to debt Y, with numbers. Offer options: fix debt alongside the next feature in that area (cheapest), a time-boxed slice, or accept the risk explicitly with a revisit date. Let product decide with full information; record in the register/ADR. Agree on a baseline allocation (e.g. 10-20%) as team policy, adjusted by risk. Avoid hidden padding of estimates, which destroys trust. Validate by showing improved velocity or reduced incidents afterwards.
**Follow-up traps**: "What if a security debt?" (risk-based, may be non-negotiable). "Hidden refactor in PRs?" (OK if small, visible, and tested).

### 9.5 [Scenario] You have a backlog of 200 debt items. How do you pick?
**What the interviewer is really testing**: Prioritisation method.
**Typical mid-level answer**: "Highest severity first." Falls short: no cost/benefit.
**Strong senior answer**: Cull first: close items nobody would act on. Score the rest by interest (cost per month), blast radius, risk (security/availability), and cost to fix; boost items adjacent to planned roadmap work because the marginal cost is lower. Cluster related items into themes (e.g. "flaky deploy pipeline") for meaningful business outcomes. Pick 3-5 themes per quarter; leave the long tail. Review quarterly. Validate through outcomes (lead time, incident counts), not items closed.
**Follow-up traps**: "Do you ever delete the backlog?" (yes; stale items are noise). "Ownership?" (the team owning the area).

**Red flags in answers**
- "Debt = anything I dislike."
- Proposing a freeze on features for a cleanup.
- No measurement before or after.

**Cheat sheet**
- Debt has interest and principal; quantify both.
- Deliberate prudent debt is a legitimate tool.
- Hotspots = churn x complexity.
- Prefer incremental large-scale change; add tests first.
- Speak in delivery speed, risk and cost.
- Standing capacity allocation, reviewed quarterly.

---

## Topic 10: Rewrite vs refactor vs strangler; modernising .NET Framework

**Study notes**
- Options: **refactor in place**, **strangler fig** (route traffic gradually to new implementation, retire old), **rewrite** (replace wholesale), **replatform/lift-and-shift** (move hosting as-is).
- Rewrite risks: lost implicit behaviour, long period with no value, moving target, second-system effect (over-ambitious redesign loaded with every wish), team split between old and new.

| Criterion | Favours rewrite | Favours refactor/strangler |
|---|---|---|
| Tech is truly end-of-life/unsupportable | yes | maybe |
| System small/well-understood | yes | |
| Business behaviour poorly documented | | yes |
| Must keep shipping features | | yes |
| Architecture fundamentally wrong for NFRs | maybe | strangler piece by piece |

- Strangler in .NET: YARP or Azure API Management/Front Door as the routing façade; migrate by route/capability; anti-corruption layer; data sync via CDC/outbox; feature flags; shadow traffic comparison.
- .NET Framework -> modern .NET: assess with .NET Upgrade Assistant / API Portability analyzer (Microsoft's tooling is evolving, with newer Copilot-assisted modernization; verify current tools). Steps: move to SDK-style projects, target `netstandard2.0` for shared libs, update packages, fix incompatible APIs (WCF server, Web Forms, System.Web, AppDomains, Remoting have no direct equivalents; CoreWCF, incremental ASP.NET Core migration via System.Web adapters help), then retarget to current LTS .NET. .NET Framework 4.8 remains supported as part of Windows, so urgency is risk-based, not panic.

### 10.1 [Easy] When is a rewrite justified?
**What the interviewer is really testing**: Scepticism toward rewrites.
**Typical mid-level answer**: "When the code is messy." Falls short: messy ≠ unfixable.
**Strong senior answer**: Rarely, and only with evidence. Justified when the platform is unsupportable (unavailable skills, security-unpatchable), the architecture fundamentally cannot meet required NFRs and cannot be incrementally changed, or the system is small enough to replace quickly. The old code encodes years of business rules and edge cases that nobody remembers. Options: refactor, strangle, rewrite. I'd default to strangler/incremental with value delivered continuously. If a rewrite is chosen, time-box, define parity tests (characterization tests against old behaviour), and keep scope to like-for-like first. Validate with milestones that deliver production value early.
**Follow-up traps**: "What is the second-system effect?" (overdesigning the successor). "Can't we freeze the old system?" (rarely true).

### 10.2 [Follow-up] Explain the strangler pattern in a .NET context.
**What the interviewer is really testing**: Practical mechanics.
**Typical mid-level answer**: "Replace parts over time." Falls short: no mechanics.
**Strong senior answer**: Put a routing façade (YARP, APIM, or Front Door) in front of the legacy app. Carve out a capability (say, invoice PDF generation), build it in modern .NET, route that path to the new service, compare behaviour (shadow traffic or parity tests), then retire the legacy code. Data is the hard part: use shared DB temporarily with ownership rules, or CDC/outbox events to sync; avoid long-term dual writes. Order slices by value, risk, and seam clarity. Trade-offs: temporary complexity of two systems, and routing/auth consistency (shared cookie/token). Validate with per-slice rollback via routing flag and decommission metrics (legacy traffic share).
**Follow-up traps**: "When does it stall?" (last 20%, entangled data). "Auth across both?" (shared identity provider/token).

### 10.3 [Medium] Plan a migration of a large .NET Framework 4.8 app (WebForms + WCF + SQL) to modern .NET.
**What the interviewer is really testing**: Realistic migration approach and risk handling.
**Typical mid-level answer**: "Run the Upgrade Assistant and fix errors." Falls short: tools don't redesign WebForms/WCF.
**Strong senior answer**: Assess first: inventory dependencies, API usage (portability analyzer), third-party components, and test coverage. Increase safety: characterization tests, CI, observability. Migrate bottom-up: shared libraries to SDK-style, multi-target `netstandard2.0`/modern TFM; then host-level pieces. WCF services: CoreWCF or move to gRPC/REST with clients migrated incrementally. WebForms has no port: strangle page by page into ASP.NET Core/Blazor/SPA, using incremental ASP.NET migration with System.Web adapters and a YARP façade. Handle data via EF6 -> EF Core only where needed (EF6 works on modern .NET if time-constrained). Risks: untested behaviour, config/IIS differences, third-party licences. Validate with parity tests, canary traffic, and rollback plan; target current LTS.
**Follow-up traps**: "Big-bang weekend?" (no). "Lift-and-shift to VM first?" (a valid stepping stone if hosting is the risk).

### 10.4 [Tricky] The CTO says: "Rewrite in microservices, it'll fix everything." How do you respond?
**What the interviewer is really testing**: Challenging with data respectfully.
**Typical mid-level answer**: "Okay, start designing." or "That's a bad idea." Falls short: no diagnosis.
**Strong senior answer**: First understand the pain: slow delivery, outages, cost, scaling, hiring? Different pains have different cures. Microservices solve team-scale and independent deployment problems but add distributed-systems complexity. Present options with evidence: targeted refactor, modular monolith, strangler for the worst areas, full rewrite. For each, estimate cost, time to first value, risk, and what NFR it improves. Propose a thin slice pilot with success criteria to test the hypothesis cheaply. I'd treat it as a joint decision with the CTO, keeping respect and data central. Reversible: pilot results decide scope.
**Follow-up traps**: "What if CTO insists?" (disagree and commit, scope with checkpoints). "Metrics?" (DORA, incident rates).

### 10.5 [Scenario] Halfway through a rewrite, the old system keeps getting new requirements. What now?
**What the interviewer is really testing**: Handling the moving target.
**Typical mid-level answer**: "Implement in both." Falls short: doubles cost.
**Strong senior answer**: This is the classic rewrite trap. Options: freeze features on legacy (rarely accepted), dual-implement (expensive), or shift to strangler so new features land only in the new system's slice. Triage: new requirements touching already-migrated slices go to the new system; those in unmigrated slices are either deferred or done in legacy minimally. Reorder the migration to move the highest-churn areas first. Make the cost of dual-work visible to product to drive prioritisation. Validate by tracking the share of change landing in the new system.
**Follow-up traps**: "Second-system effect?" (resist adding wishes). "Team morale?" (rotate people across both).

**Red flags in answers**
- "Rewrite from scratch" as the default.
- Believing a tool auto-migrates WebForms/WCF.
- No parity testing or rollback.

**Cheat sheet**
- Default: strangler/incremental; rewrite needs evidence.
- Characterization tests before change.
- YARP/APIM façade; migrate by capability.
- Data sync is the hard part.
- Second-system effect: keep like-for-like first.
- Treat 4.8 urgency as risk-based; verify support status.

---

## Topic 11: Documentation that works & governance without bureaucracy

**Study notes**
- **C4 model**: Context -> Containers -> Components -> (Code). Most value from levels 1-2. Tools: Structurizr DSL, Mermaid, PlantUML — text-based, so docs-as-code (versioned, PR-reviewed, rendered in CI).
- Docs that matter: README (what/why/how to run/test), ADRs, runbooks (alerts -> diagnosis -> action), onboarding guide, API docs (OpenAPI), data ownership map, SLOs.
- Principle: document what's hard to discover from code; keep close to code; owner + review date; delete stale docs.
- **Governance without bureaucracy**: guardrails (policy-as-code: Azure Policy, analyzers, templates), **paved roads** (supported, easy defaults: project templates, pipelines, reference architectures), light review (advisory forum, decision records), **inner-source** (shared ownership via contributions), and "you build it, you run it".

### 11.1 [Easy] What documentation is worth maintaining?
**What the interviewer is really testing**: Pragmatism.
**Typical mid-level answer**: "Document everything." Falls short: becomes stale.
**Strong senior answer**: Document what can't be read from code and what is expensive to rediscover: the why (ADRs), system context and container diagrams, runbooks for operating, data ownership, onboarding steps, API contracts. Keep it in the repo, reviewed alongside changes, with an owner. Skip method-level docs that restate code. Criteria: who reads it and when (new hire, on-call at 3am). Validate by having a new engineer follow the README and runbook, and fixing where they stumble.
**Follow-up traps**: "Wiki or repo?" (repo for engineering docs; wiki for broad guides). "Staleness?" (CI link checks, review dates).

### 11.2 [Follow-up] Explain the C4 model and how you'd use it.
**What the interviewer is really testing**: Familiarity and usefulness.
**Typical mid-level answer**: "Four levels of diagrams." Falls short: no application.
**Strong senior answer**: C4 gives consistent zoom levels: System Context (people and external systems), Container (deployable units: web app, API, DB, queue), Component (internal structure of a container), Code (rarely drawn). Use abstraction consistently with clear labels, technology, and relationships; avoid mixed-level "box soup". I'd produce Context and Container for every system, Component only for complex containers, and keep them as Structurizr DSL or Mermaid in the repo so they update with PRs. Add a deployment view for Azure topology when ops matter. Validate by whether a stakeholder can explain the system from the diagram.
**Follow-up traps**: "UML?" (fine, but C4 is more approachable). "Auto-generated?" (partially; curated is clearer).

### 11.3 [Medium] How do you govern architecture across 20 teams without becoming a bottleneck?
**What the interviewer is really testing**: Scalable governance.
**Typical mid-level answer**: "Architecture review board approves everything." Falls short: bottleneck, slow.
**Strong senior answer**: Shift from approval to enablement. Guardrails in automation: Azure Policy, pipeline checks, analyzers, security baselines. Paved roads: templates, reference architectures, shared pipelines so the right way is the easy way. Review only for one-way-door or cross-team decisions, via lightweight RFC/ADR with a fixed response window; teams decide the rest. Inner-source shared components with contributor guidelines. Measure: time-to-decision, adoption of paved roads, incidents from non-compliance. Reverse/adjust by reviewing which guardrails are circumvented — a signal the road isn't good enough.
**Follow-up traps**: "What if teams ignore guidance?" (find out why; escalate only for real risk). "Who sets guardrails?" (platform + security with team input).

### 11.4 [Tricky] A team deviates from the paved road. Allow it?
**What the interviewer is really testing**: Balance of autonomy and consistency.
**Typical mid-level answer**: "No, standards are standards." Falls short: kills innovation.
**Strong senior answer**: Ask why. Perhaps there's a real requirement the road doesn't meet. Options: allow with justification, extend the road, or refuse. Criteria: risk (security/compliance), operational burden (who supports it?), reusability of the learning, and cost. Allow where the team accepts full ownership (build/run/support) and writes an ADR; refuse for non-negotiables like identity, secrets, or logging that platform-wide operations depend on. If multiple teams deviate similarly, the road needs improving. Validate by revisiting the deviation after 6 months.
**Follow-up traps**: "Who supports it at 3am?" (the deviating team). "Precedent risk?" (documented criteria).

### 11.5 [Scenario] You inherit a system with no documentation. What's the minimum useful set you produce in two weeks?
**What the interviewer is really testing**: Prioritised knowledge capture.
**Typical mid-level answer**: "Write full documentation." Falls short: unrealistic.
**Strong senior answer**: Produce: a C4 Context and Container diagram validated by running the system and reading deployment config/Azure resources; a README with build/run/test steps proven on a fresh machine; a list of external dependencies and data stores with owners; a basic runbook from alerts and past incidents; key retroactive ADRs for obvious decisions (marked reconstructed); a list of unknowns. Source of truth: code, infra-as-code, telemetry (App Insights application map), and people who know. Validate by having someone unfamiliar use them. Improve continuously as you learn.
**Follow-up traps**: "Ask whom?" (support staff, original authors, on-call history). "Automated?" (application map, dependency graphs as a start).

**Red flags in answers**
- Documentation in unversioned slides nobody updates.
- Governance equal to central approval of everything.
- No concept of paved roads or automation.

**Cheat sheet**
- C4: Context + Container are the sweet spot.
- Docs-as-code: text diagrams in the repo.
- README, ADRs, runbooks, onboarding.
- Guardrails + paved roads > gates.
- Review only one-way doors and cross-team impact.
- Deviation allowed with ownership and ADR.

---

## Topic 12: Architecture reviews; presenting to executives vs engineers

**Study notes**
- Review a design doc by checking: problem clarity, requirements/NFRs, alternatives, data ownership, failure modes, SPOFs, security/threat model, operability, cost, migration/rollout, testing, and unknowns.
- Questions to ask: What happens when X is down/slow? Who owns this data? What's the blast radius? How do we deploy/roll back? How do we know it's working? What does it cost at 10x? What's the simplest thing that could work? What did you reject and why?
- Feedback style: separate blocking from non-blocking; ask questions; propose alternatives; praise specifics; focus on the design, not the person.
- Audiences: executives want outcome, risk, cost, decision needed, timeline; engineers want mechanics and trade-offs. Lead with the decision and the "so what".

| Review area | Look for |
|---|---|
| Failure modes | timeouts, retries/backoff, circuit breakers, idempotency, poison messages |
| Data | single owner per dataset, consistency model, migration, retention/PII |
| SPOF | single region/instance, single queue, shared DB, single person |
| Security | authN/Z, secrets/managed identity, network isolation, least privilege, STRIDE |
| Cost | unit cost, scaling cliffs, egress, log ingestion |
| Operability | metrics, logs, traces, alerts, runbooks, deploy/rollback |

### 12.1 [Easy] How do you review a design document?
**What the interviewer is really testing**: Structured review.
**Typical mid-level answer**: "Read it and give comments." Falls short: unfocused.
**Strong senior answer**: I first confirm the problem and success criteria are clear, then check that NFRs are quantified and alternatives considered. Next I walk the failure paths: what happens when a dependency is slow or down, how data stays consistent, who owns each dataset, where the SPOFs are, how security (authN/Z, secrets, network) and cost are handled, and how it's deployed, observed, and rolled back. I distinguish blocking issues (correctness, safety) from suggestions, and phrase as questions. I finish with what I liked and a clear recommendation: approve, approve with changes, or revisit. Validate by following up on whether comments changed the design.
**Follow-up traps**: "How long?" (time-boxed; deep review for one-way doors). "Nitpicks?" (don't bury key issues).

### 12.2 [Follow-up] What failure-mode questions expose weak designs fastest?
**What the interviewer is really testing**: Depth on resilience.
**Typical mid-level answer**: "What if the DB goes down?" Falls short: stops at obvious.
**Strong senior answer**: I ask about partial failures: the downstream API is slow rather than down — are there timeouts, bulkheads, and backoff with jitter? A message is delivered twice — is the handler idempotent? A deployment half-completes — are schema and code compatible (expand/contract)? A retry storm after recovery? Poison messages blocking a queue? A dual-write between DB and broker — outbox? Region failure — what's RTO/RPO and have we practiced? Each question maps to a design element; if the doc has no answer, that's a gap. Validate through failure-injection tests (chaos experiments) later.
**Follow-up traps**: "Exactly-once?" (practically at-least-once + idempotency). "Cascading failure?" (circuit breakers, load shedding).

### 12.3 [Medium] How do you give critical feedback on a senior colleague's design without friction?
**What the interviewer is really testing**: Communication and respect.
**Typical mid-level answer**: "Be honest and direct." Falls short: tone and process missing.
**Strong senior answer**: Anchor on shared goals and evidence: "Against our p95 target, I'm worried about X — what happens when Y?" Ask before asserting, because I may lack context. Be specific, offer alternatives with trade-offs, and distinguish must-fix from preference. For big concerns, talk synchronously first rather than surprising in a public thread. Acknowledge strengths. If we still disagree, agree on a way to resolve: prototype, benchmark, or escalate to the named decider. Validate by the author feeling the review improved the design, and relationship intact.
**Follow-up traps**: "Public vs private feedback?" (private first for sensitive). "If they ignore it?" (record in ADR; disagree and commit unless safety risk).

### 12.4 [Tricky] How do you present the same architecture to an executive and to engineers?
**What the interviewer is really testing**: Audience calibration.
**Typical mid-level answer**: "Simplify for executives." Falls short: loses decision framing.
**Strong senior answer**: Executives: start with the decision needed and why, business outcomes, options at a high level with cost, risk and timeline, what happens if we do nothing, and what I need from them. One or two visuals (C4 context, roadmap). Engineers: start from requirements, then containers/components, data flows, failure modes, trade-offs, rollout, and open questions; invite challenge. Same facts, different framing; never contradicting. Avoid jargon with executives; avoid hand-waving with engineers. Validate by asking executives to restate the decision and engineers to identify the riskiest assumption.
**Follow-up traps**: "How long should the exec deck be?" (a few slides plus appendix). "Hard question you can't answer?" (say so, commit to follow-up date).

### 12.5 [Scenario] You're asked to approve a design in 30 minutes with no prior reading. What do you do?
**What the interviewer is really testing**: Integrity under pressure.
**Typical mid-level answer**: "Approve to keep things moving." Falls short: abdicates responsibility.
**Strong senior answer**: Be explicit about what you can assess. Use the 30 minutes on high-risk questions: data ownership, failure modes, security, irreversible parts, and rollout. Decide whether it's a two-way door: if so, "proceed with conditions and review after the first slice". If one-way, request a short written summary and a follow-up review rather than rubber-stamping. State what was and wasn't reviewed in the record. Propose changing the process: pre-reads at least 24 hours ahead. Validate by checking whether conditions were met.
**Follow-up traps**: "Pressure from a deadline?" (separate what's required for launch from later). "Approve verbally?" (record in writing).

**Red flags in answers**
- Reviews that only nitpick style, missing failure modes and data ownership.
- Delivering feedback as verdicts, not questions.
- One presentation for every audience.

**Cheat sheet**
- Review: NFRs, alternatives, failures, data, SPOFs, security, cost, operability, rollout.
- Ask about partial failures, duplicates, half-deployments.
- Mark blocking vs non-blocking.
- Executives: decision, outcome, risk, cost, timeline.
- Engineers: mechanics, trade-offs, open questions.
- Don't rubber-stamp; state review scope.

---

## Topic 13: Ambiguity, "it depends" done right, over/under-engineering

**Study notes**
- Handling ambiguity: **assumptions log** (assumption, owner, how to validate, impact if wrong), **thin vertical slices**, **tracer bullets** (production-quality end-to-end skeleton that you build on) vs **spikes** (throwaway learning, time-boxed), decide at the last responsible moment.
- "It depends" done right: 1) name the 2-3 dependencies that swing the answer, 2) state how each pushes the decision, 3) say what you'd choose given the likely values, 4) say what would change your mind.
- Over-engineering: speculative generality, premature microservices/abstractions. Under-engineering: no tests/observability/boundaries where risk is high. Heuristics: YAGNI for features, but invest early in one-way doors; **rule of three** (abstract after the third similar case); "make it work, make it right, make it fast" in order, with NFR targets guiding the last.

| Signal | Over-engineered | Under-engineered |
|---|---|---|
| Symptoms | Layers with one implementation; config for hypothetical needs | God classes; no tests; hard-coded env values |
| Cost | Slow changes, onboarding | Incidents, rework |
| Fix | Delete abstractions | Add seams at risky points |

### 13.1 [Easy] How do you handle incomplete requirements?
**What the interviewer is really testing**: Productive progress under uncertainty.
**Typical mid-level answer**: "Wait for the BA to clarify." Falls short: blocks delivery.
**Strong senior answer**: Separate what's known from what's assumed. Keep an assumptions log with owner and validation plan, and design so likely-to-change items are isolated. Deliver a thin slice end to end early to learn from real feedback, rather than wait. Ask targeted questions where an answer would change the design materially; not all gaps matter. Decide at the last responsible moment for expensive items and proceed on cheap ones. Validate assumptions in order of risk; update ADRs when they change.
**Follow-up traps**: "When do you escalate?" (when a wrong assumption is costly and owner unresponsive). "Document assumptions where?" (repo, linked to ADRs).

### 13.2 [Follow-up] Spike vs tracer bullet: when to use which?
**What the interviewer is really testing**: Precise vocabulary and judgment.
**Typical mid-level answer**: "They're the same, prototypes." Falls short: confuses throwaway and keepable code.
**Strong senior answer**: A spike answers a specific technical question quickly (can Service Bus sessions give us ordered processing at this rate?) and the code is thrown away; time-boxed with a written outcome. A tracer bullet is a thin but production-quality path through all layers (UI -> API -> DB -> deployment pipeline) that proves the architecture and is extended incrementally. Use spikes for unknowns in a technology; tracer bullets to validate the structure and the delivery pipeline early. Risk: spikes drifting into production; tracer bullets built sloppily. Validate through the spike's documented answer, and the tracer's deployment to a real environment.
**Follow-up traps**: "Spike time-box?" (days, not weeks). "Can a prototype become production?" (only if intentionally built to).

### 13.3 [Medium] Give an example of "it depends" answered well: "Should we use Redis cache?"
**What the interviewer is really testing**: Commit after conditions.
**Typical mid-level answer**: "It depends on the use case." Falls short: stops there.
**Strong senior answer**: It depends on (1) read/write ratio and read-heavy hot data, (2) staleness tolerance, (3) whether the database or its tier is the bottleneck, and (4) team ops capacity. If reads dominate, data tolerates seconds of staleness, and SQL is near limits, then yes: Azure Cache for Redis with cache-aside and TTL, plus invalidation on writes. If SQL has headroom, fix indexes/queries first, and use in-memory/HybridCache for hot items; a distributed cache adds failure modes (stampede, stale data, another dependency). My recommendation given typical symptoms: measure first, tune queries, add local caching, and introduce Redis only if scale-out instances need shared state or the data is still hot. Would change my mind if measurements show repeated identical expensive queries across instances.
**Follow-up traps**: "Cache stampede?" (locking, jitter, request coalescing). "What if Redis is down?" (degrade to DB with protection).

### 13.4 [Tricky] How do you tell over-engineering from future-proofing?
**What the interviewer is really testing**: Calibration.
**Typical mid-level answer**: "Future-proof everything." Falls short: wasteful.
**Strong senior answer**: Ask how likely and how costly the change is. Future-proof one-way doors cheaply (tenant id in the schema, versioned contracts, idempotent handlers) because adding them later is expensive. Don't build features or abstractions for needs without evidence; apply rule of three: tolerate duplication until a pattern is proven. Test: if the extension point has one implementation and no concrete second use case, delete it. Conversely, under-engineering shows when basic observability, tests, or security are skipped at the risky parts. Validate in retros: which abstractions were used, and which changes were painful?
**Follow-up traps**: "DRY vs rule of three?" (wrong abstraction costs more than duplication). "Interfaces for everything?" (not necessary).

### 13.5 [Scenario] A teammate proposes CQRS + event sourcing for a CRUD admin app. Respond.
**What the interviewer is really testing**: Resisting fashionable complexity respectfully.
**Typical mid-level answer**: "Sounds scalable, go." Falls short: no need analysis.
**Strong senior answer**: Ask what problem it solves: audit history, temporal queries, very different read/write loads, complex domain? Event sourcing adds event versioning, projections, eventual consistency, and rebuild tooling. For a CRUD admin app with modest load, a normal EF Core model with an audit table or SQL temporal tables gives history at far lower cost. Options: plain CRUD, CRUD + audit, CQRS-lite (separate read queries with Dapper), full ES. Decide by requirements; if audit is the driver, temporal tables or an outbox-based audit feed suffice. Record in an ADR, revisit if requirements grow. Validate with the team's ability to operate and evolve the choice.
**Follow-up traps**: "Isn't CQRS separate from ES?" (yes, can use alone). "Learning opportunity?" (do it in a low-risk sandbox).

**Red flags in answers**
- "It depends" with no commitment.
- Building generic frameworks before the second use case.
- Treating ambiguity as a reason to stall.

**Cheat sheet**
- Assumptions log: owner, validation, impact.
- Spike = learn and discard; tracer bullet = keep and extend.
- "It depends": name dependencies, commit, state what changes your mind.
- Invest early only in one-way doors.
- Rule of three before abstracting.
- Measure before adding infrastructure.

---

## Topic 14: Testability, operability & internal platform product thinking

**Study notes**
- First-class qualities: **testability** (seams, dependency injection, deterministic time via `TimeProvider`, test pyramid, Testcontainers, contract tests), **operability** (structured logs, metrics, traces via OpenTelemetry/Application Insights, health checks, SLOs, runbooks), **deployability** (CI/CD, IaC with Bicep/Terraform, slots/canary, feature flags, backward-compatible DB migrations), **supportability** (correlation IDs, diagnosable errors, admin tools).
- "If you can't observe it, you can't operate it": design telemetry together with features; use a definition of done including dashboards/alerts.
- **Platform as a product**: internal developers are customers. Treat the platform with a roadmap, user research, documentation, SLOs, onboarding metrics, feedback loops, versioning and deprecation policy. Success = adoption by choice, reduced lead time, and cognitive load reduction (Team Topologies: platform team provides self-service to stream-aligned teams).

### 14.1 [Easy] What does "design for operability" mean?
**What the interviewer is really testing**: Seeing ops as a design concern.
**Typical mid-level answer**: "Add logging." Falls short: partial.
**Strong senior answer**: Design so the people running it can detect, diagnose, and recover quickly. Concretely: health endpoints (liveness vs readiness), structured logs with correlation IDs, metrics for RED (rate/errors/duration) and saturation, distributed tracing across services, SLO-based alerts that are actionable, runbooks linked to alerts, safe deploy and rollback, and configuration that can change without code. Include these in the definition of done. Trade-off: telemetry cost and noise, so sample and set retention. Validate with game days: can on-call find a seeded fault in minutes?
**Follow-up traps**: "Alert fatigue?" (alert on symptoms/SLO burn, not causes). "Log volume cost?" (sampling, levels).

### 14.2 [Follow-up] How do you design for testability in a .NET service?
**What the interviewer is really testing**: Practical seams.
**Typical mid-level answer**: "Use interfaces and mocks." Falls short: over-mocking.
**Strong senior answer**: Keep domain logic pure and separate from I/O so it's unit-testable without mocks; inject volatile dependencies (time via `TimeProvider`, clock, GUIDs, external clients). Prefer integration tests for data access with real SQL Server in a container (Testcontainers) rather than in-memory providers that behave differently; use `WebApplicationFactory` for API tests; consumer-driven contract tests between services. Test pyramid proportions: many fast unit tests, fewer integration tests, few end-to-end. Avoid mocking what you don't own; wrap and test the wrapper. Validate by measuring test run time, flakiness, and escaped defects.
**Follow-up traps**: "EF in-memory provider?" (not faithful to SQL behaviour). "Coverage target?" (use as a signal, not a goal).

### 14.3 [Medium] You're asked to build an internal platform (templates, pipelines, shared services). How do you make it succeed?
**What the interviewer is really testing**: Product thinking for platforms.
**Typical mid-level answer**: "Build the tooling and mandate it." Falls short: mandates breed resentment.
**Strong senior answer**: Treat it as a product: identify customer teams and their pain via interviews, pick the highest-friction journey (e.g. "new service to production in a day"), and ship a thin paved road first. Publish docs and examples, offer a support channel, version and deprecate responsibly, and set SLOs for the platform itself. Measure adoption, onboarding time, lead time, and satisfaction, not features shipped. Keep it optional where possible so quality is proven by usage, but make non-negotiables (security, identity) default. Trade-off: platform team cost vs saved cognitive load; justify with metrics. Reverse by sunsetting unused components.
**Follow-up traps**: "Mandate or opt-in?" (default-on guardrails, opt-in for conveniences). "Platform team as bottleneck?" (self-service, contributions).

### 14.4 [Tricky] Deployability: how do you do zero-downtime deploys with SQL schema changes?
**What the interviewer is really testing**: Cross-layer compatibility.
**Typical mid-level answer**: "Run the migration then deploy." Falls short: breaks old instances during rollout.
**Strong senior answer**: Use expand/contract. Expand: add nullable columns/new tables in a backward-compatible migration; deploy code that writes both/reads new with fallback; backfill in batches; later contract: remove old columns after all instances and rollbacks no longer need them. Avoid long locking operations; consider online index operations. Use deployment slots or blue-green with health checks, and feature flags to decouple deploy from release. Trade-off: multiple releases for one change and temporary complexity. Validate in a staging environment with the previous app version running against the new schema.
**Follow-up traps**: "Rollback after contract?" (not possible; hence waiting period). "EF migrations auto-apply?" (prefer controlled migration step).

### 14.5 [Scenario] On-call is painful: noisy alerts and slow diagnosis. What architecture-level changes help?
**What the interviewer is really testing**: Systemic approach to operability.
**Typical mid-level answer**: "Add more dashboards." Falls short: adds noise.
**Strong senior answer**: Define SLOs for key journeys and alert on error-budget burn rate, deleting alerts that aren't actionable. Add distributed tracing and correlation IDs so one request is traceable across services; standardise structured logging and common dimensions. Add readiness probes and circuit breakers so failures degrade gracefully rather than cascade. Write runbooks for the top recurring alerts and automate remediations. Review incidents blamelessly and feed fixes into backlog. Validate with MTTR, pages per week, and percentage of actionable alerts.
**Follow-up traps**: "Whose job?" (team that owns the service). "Retention costs?" (tiered log retention).

**Red flags in answers**
- Observability bolted on after launch.
- Mocking everything; no integration tests against real SQL.
- A platform built without talking to its users.

**Cheat sheet**
- Operability: health, logs, metrics, traces, SLO alerts, runbooks.
- Testability: pure domain, injected volatility, Testcontainers.
- Expand/contract for zero-downtime schema changes.
- Deploy != release (feature flags).
- Platform = product: customers, roadmap, adoption metrics.
- Definition of done includes telemetry.

---

## Topic 15: Standardisation vs autonomy; monorepo vs polyrepo; shared libraries; multi-team roadmaps

**Study notes**
- Standardise where **variation has no value and coordination cost is high** (identity, logging format, CI security gates, observability, deployment); allow autonomy where teams' context differs (internal design, some tech choices).
- Monorepo: atomic cross-project changes, shared tooling, easy refactors, visibility; needs tooling (build caching, path-based CI triggers, CODEOWNERS), can slow CI/clone if unmanaged. Polyrepo: clear ownership/independent release, smaller blast radius; risks: version drift, cross-repo changes painful, duplicated tooling.
- Shared libraries: use semantic versioning, publish to a private NuGet feed, keep small and stable, avoid shared **domain** libraries across services (distributed monolith: lock-step deploys). Prefer contracts (OpenAPI/events/schemas) over shared code; allow duplication of small code.
- Dependency management across teams: explicit contracts, consumer-driven contract tests, dependency map, shared roadmap with milestones/integration points, decoupling via async events, a deprecation policy, and a single owner per capability.

| Question | Lean toward |
|---|---|
| Tightly related projects, one team | Monorepo |
| Independent teams, differing cadence | Polyrepo (or monorepo with strong ownership tooling) |
| Cross-cutting refactors frequent | Monorepo |
| Strict access separation required | Polyrepo |

### 15.1 [Easy] Where should an organisation standardise and where not?
**What the interviewer is really testing**: Principle for standardisation.
**Typical mid-level answer**: "Standardise everything for consistency." Falls short: stifles teams.
**Strong senior answer**: Standardise at the interfaces and on the undifferentiated heavy lifting: authentication, logging/telemetry formats, CI/CD security checks, API conventions, deployment mechanisms, secrets handling. Those reduce cross-team cognitive load and ops cost. Leave autonomy where it's local: internal structure, test style, sometimes the datastore if the team owns running it. Criteria: coordination cost, risk, and benefit of variation. Provide the standard as a paved road so it's the easiest option. Validate by adoption and by incident/onboarding metrics; revisit when standards hamper delivery.
**Follow-up traps**: "One language only?" (not necessarily; cost of support). "Who decides?" (guild/platform with team input).

### 15.2 [Follow-up] Monorepo or polyrepo for 8 .NET services owned by 3 teams?
**What the interviewer is really testing**: Trade-off reasoning with context.
**Typical mid-level answer**: "Monorepo is what Google does." Falls short: context mismatch.
**Strong senior answer**: Options: one monorepo, per-team repos, or per-service repos. Criteria: how often changes cross services, ownership clarity, CI capability, release independence, and access control. With 8 services and 3 teams and frequent shared contract changes, a monorepo with CODEOWNERS, path-filtered pipelines, central package management, and shared build props gives atomic changes and consistent tooling, and still lets each service deploy independently. Cost: pipeline design and repository growth. If teams are largely independent or have security separation, polyrepo with shared templates works. Decision isn't permanent; moving repos while keeping history is possible. Validate via PR cycle time, CI duration, and cross-repo change friction.
**Follow-up traps**: "Does monorepo mean monolith?" (no). "Large repo performance?" (sparse checkout, shallow clone, build caching).

### 15.3 [Medium] A shared "Common" NuGet library is slowing every team. What do you do?
**What the interviewer is really testing**: Shared-code pitfalls.
**Typical mid-level answer**: "Version it and tell teams to upgrade." Falls short: root cause untouched.
**Strong senior answer**: Diagnose the content: is it a grab-bag of domain models, utilities, and infrastructure? Domain models shared across services couple deployments. Options: split into small focused packages (logging, HTTP client defaults), move domain contracts to schemas/OpenAPI generated clients, delete or duplicate low-value shared code, and enforce semantic versioning with non-breaking minor updates. Add an owner, changelog, and deprecation policy, and automated update PRs (Renovate/Dependabot). Criteria: stability and genericity; shared code should change rarely. Validate by upgrade lead time and number of breaking releases.
**Follow-up traps**: "Inner-source?" (accept PRs with clear owners). "Multi-targeting?" (support N and N-1 TFMs).

### 15.4 [Tricky] How do you manage dependencies in a roadmap involving five teams?
**What the interviewer is really testing**: Cross-team planning.
**Typical mid-level answer**: "Weekly sync meetings." Falls short: no structural approach.
**Strong senior answer**: Make dependencies explicit and reduce them. Map them in a visible board; identify the critical path and integration milestones, not just team deliverables. Reduce coupling: agree contracts early (OpenAPI/event schemas) with stubs/mocks so teams work in parallel; deliver thin end-to-end slices to find integration issues early; use feature flags to decouple release dates. Assign a single accountable owner for each cross-team outcome. Review risks at fixed cadence and escalate slips early. Validate by integration surprises found late, which should trend to zero.
**Follow-up traps**: "Dependent team slips?" (re-plan scope or use stubs). "Big-bang integration?" (avoid).

### 15.5 [Scenario] One team wants a different language and datastore for their service. Allow?
**What the interviewer is really testing**: Autonomy with accountability.
**Typical mid-level answer**: "No, standardise on .NET and SQL." Falls short: no reasoning.
**Strong senior answer**: Ask what problem it solves and the evidence. Options: stay on standards; allow with full ownership; or partial adoption. Criteria: real fit improvement vs preference; support and hiring cost; security/compliance integration; observability compatibility; exit path. If the benefit is significant and the team commits to run it (on-call, patching, telemetry via OpenTelemetry, same CI security gates), approve with an ADR and a review in six months. Otherwise decline, explaining the cost. Avoid precedent chaos by stating criteria publicly. Validate via the review.
**Follow-up traps**: "Who covers on-call if they leave?" (risk accepted explicitly). "Bus factor?" (documented).

**Red flags in answers**
- "Everyone must use the same stack, always."
- Sharing domain models via a common NuGet across services.
- Roadmaps listing team deliverables without dependencies.

**Cheat sheet**
- Standardise interfaces and heavy lifting; free local choices.
- Monorepo vs polyrepo: depends on change coupling, ownership, tooling.
- Share contracts, not domain code.
- Semver + private feed + owners.
- Dependencies: explicit, reduced, thin slices.
- Deviations: ownership + ADR.

---

## Topic 16: Cost as an architecture driver; handling disagreements

**Study notes**
- **Unit economics**: cost per transaction/tenant/active user; compare to revenue per unit. Architecture decisions set the cost curve: scaling cliffs, chatty services, data egress, log ingestion, always-on environments.
- Levers: right-size SKUs, autoscale, reserved instances/savings plans for steady load, Azure SQL serverless for intermittent DBs, storage tiering, caching, batching, reducing log volume, dev/test auto-shutdown, FinOps tagging and budgets/alerts, architectural efficiency (fewer round trips).
- Cost vs other attributes: more redundancy = more cost; cheapest design may lose on operability.
- Disagreements: seek shared goals; make criteria explicit; use data/prototypes; time-box; timely escalation to a named decider; **disagree and commit**; record dissent in ADR.

### 16.1 [Easy] How does cost influence architecture?
**What the interviewer is really testing**: Cost awareness beyond "cloud is cheap".
**Typical mid-level answer**: "Pick cheaper services." Falls short: ignores engineering time and trade-offs.
**Strong senior answer**: Cost is a quality attribute with a curve, not a number. Each decision sets how cost scales: per-request pricing vs provisioned capacity, cross-region egress, per-GB log ingestion, DTU/vCore tiers. I compute unit cost (e.g. per tenant per month) at current and 10x scale and compare against revenue/budget. Include engineering and ops time: a cheaper design needing more operation may cost more overall. Set budgets and alerts, tag resources, and review monthly. Validate by comparing predicted and actual cost after launch.
**Follow-up traps**: "Cheapest region?" (latency/compliance matters). "Who owns cost?" (the team, with FinOps support).

### 16.2 [Follow-up] Your Azure SQL bill doubled. How do you investigate?
**What the interviewer is really testing**: Systematic cost diagnosis.
**Typical mid-level answer**: "Scale it down." Falls short: may break performance.
**Strong senior answer**: Start with Cost Management to see what changed: tier, storage, backup retention, replicas, or log analytics. Then correlate with workload: Query Store and Intelligent Insights for top resource consumers, missing indexes, regressions after a release, N+1 patterns, or unbounded scans. Options: tune queries/indexes (often the cheapest fix), right-size vCores, move to serverless or elastic pools for variable databases, add caching, archive old data, reserved capacity for stable load. Criteria: performance SLO first, then savings. Validate with before/after metrics and watch regressions.
**Follow-up traps**: "Scale down risk?" (SLO breach). "Reserved capacity?" (only for steady-state).

### 16.3 [Medium] How do you decide how much to spend on resilience?
**What the interviewer is really testing**: Cost-risk trade-off.
**Typical mid-level answer**: "As much as possible." Falls short: ignores budget.
**Strong senior answer**: Express resilience cost against business loss: cost of downtime per hour x expected downtime avoided vs added cost of redundancy. Tiers: single region zone-redundant; active/passive with backups and failover group; active/active. Each also adds operational complexity and testing cost. Pick by service criticality: tier workloads (critical, important, tolerable) instead of one level for everything. Practice failover, otherwise redundancy is theatre. Validate with RTO/RPO tests and cost reviews.
**Follow-up traps**: "Backups are DR?" (backups alone give long RTO). "Dev/test redundancy?" (not needed).

### 16.4 [Tricky] Two senior engineers disagree strongly on an architecture. You're the tie-breaker. How do you proceed?
**What the interviewer is really testing**: Facilitation and decision-making.
**Typical mid-level answer**: "Pick the better one." Falls short: no process, risks resentment.
**Strong senior answer**: First restate each position fairly and extract the underlying criteria: usually they're optimising different quality attributes. Agree on the criteria and weights together. Where facts are missing, run a time-boxed experiment (benchmark, prototype of the riskiest part). If the door is two-way, pick one and set a review point. If still tied, the accountable decider decides, records dissent in the ADR, and everyone commits. Keep it about the problem, not people. Validate with the agreed success metrics, and revisit honestly if proven wrong.
**Follow-up traps**: "What if I'm one of them?" (ask a neutral facilitator). "Disagree and commit — what if I think it's unsafe?" (safety/legal risks justify escalation).

### 16.5 [Scenario] You lose a design debate, and the decision proceeds. What do you do?
**What the interviewer is really testing**: Professional maturity.
**Typical mid-level answer**: "Implement it, but say I told you so later." Falls short: toxic.
**Strong senior answer**: Make sure my concerns and the evidence are in the ADR, ask what measurable signals would show the concern materialising, and then fully commit, implementing as if I'd chosen it. I'd propose monitoring those signals so the decision gets re-evaluated fairly. If the concern is about safety or legal compliance, escalate through the proper channel instead. Later, if the risk materialises, I focus on fixing and learning, not blame. Validate by the team seeing consistent behaviour from me: challenge before, commit after.
**Follow-up traps**: "What if the decision is clearly harming?" (present data, ask to supersede). "Passive resistance?" (unacceptable).

**Red flags in answers**
- Cost ignored until the invoice arrives.
- Optimising one component's cost at the expense of total cost of ownership.
- "Winning" arguments instead of resolving them with evidence.

**Cheat sheet**
- Unit economics: cost per tenant/transaction at 1x and 10x.
- Tune queries before buying bigger SKUs.
- Tier resilience by criticality; test failover.
- Disagreements: criteria first, then experiment, then decider.
- Disagree and commit; record dissent.
- Budgets, tags and alerts from day one.

---

## Topic 17: Scenarios I — messaging service choice, CTO wants a rewrite, conflicting designs

**Study notes**
- **Azure messaging selection**:

| Service | Model | Best for | Key traits |
|---|---|---|---|
| Service Bus | Broker: queues + topics/subscriptions | Business commands/transactions, workflows | DLQ, sessions (ordering), duplicate detection, scheduled delivery, transactions, peek-lock, at-least-once |
| Event Grid | Push-based event routing (discrete events) | Reactive notifications, Azure resource events, fan-out to webhooks/Functions | Low latency, filtering, retry/dead-letter, pay per operation; not for high-volume ordered streams |
| Event Hubs | Partitioned log/stream ingestion | Telemetry, clickstream, high-throughput streaming, replay | Partitions, consumer groups, retention, Capture to storage, Kafka-compatible endpoint; consumer tracks offset |

- Rule of thumb: **commands/work to be done -> Service Bus; "something happened" notifications -> Event Grid; high-volume telemetry/stream -> Event Hubs.** They are often combined (Event Grid for Azure-resource events, Service Bus for reliable processing).
- Verify current limits/features (tiers, message size, Event Grid namespaces/MQTT) in documentation.

### 17.1 [Scenario] Choose between Service Bus, Event Grid and Event Hubs for a new order/telemetry platform.
**What the interviewer is really testing**: Matching broker semantics to needs.
**Typical mid-level answer**: "Service Bus is the enterprise one; use that." Falls short: no workload-based reasoning.
**Strong senior answer**: I'd first classify the message flows. (1) Orders and payments: commands requiring reliable, once-processed handling, dead-lettering, maybe ordering per customer -> Service Bus queues/topics with sessions, duplicate detection, and an outbox for DB-to-broker consistency. (2) Device or app telemetry at high volume (thousands to millions events/sec) with replay and analytics -> Event Hubs with partitioning by device/tenant, Capture to storage for the data lake. (3) Lightweight reactive notifications ("blob created", "order shipped" for other teams or Functions) -> Event Grid. Criteria: throughput, ordering, delivery guarantees, replay needs, consumer model (competing vs independent), cost model, and team skills. Trade-offs: three services mean more to operate; if scope is small, Service Bus alone can cover orders and notifications via topics. Decision: Service Bus for the transactional core, Event Hubs only for the telemetry stream, Event Grid optional for integration. Validate with a load test and poison-message tests; record in an ADR with revisit trigger on volume.
**Follow-up traps**: "Exactly once?" (at-least-once; idempotent consumers). "Why not Event Hubs for everything?" (no per-message ack/DLQ semantics). "Kafka?" (Event Hubs Kafka endpoint for compatibility).

### 17.2 [Scenario] The CTO wants a full rewrite of the legacy platform in 12 months. You're the senior engineer.
**What the interviewer is really testing**: Diagnosis, influence, and risk management.
**Typical mid-level answer**: "Agree and plan it" or "Refuse." Falls short: neither is evidence-led.
**Strong senior answer**: I'd first ask for the pain behind the request and the success measure: speed, reliability, cost, hiring, or security. Then bring evidence: DORA metrics, incident data, hotspot analysis, cost of dual-running, and what is actually unfixable. Options: targeted refactoring, strangler by capability, rewrite of the worst component, or full rewrite; compare on cost, risk, time to first value, and effect on the pain. Highlight rewrite risks: lost business rules, feature freeze, second-system effect, big-bang cutover. Propose a first strangler slice as a 6-8 week pilot with explicit exit criteria, producing real data about velocity and parity. If the CTO still chooses a rewrite, I'd shape it: parity tests, incremental cutover, scoped like-for-like, milestones with go/no-go checkpoints. Commit after the decision, with risks tracked.
**Follow-up traps**: "12 months realistic?" (estimate from pilot throughput, not hope). "What if you're wrong?" (checkpoints allow course correction).

### 17.3 [Scenario] Two teams propose conflicting designs for the same capability (e.g. a shared-database approach vs an event-driven approach). How do you resolve it?
**What the interviewer is really testing**: Facilitation, criteria, evidence.
**Typical mid-level answer**: "Choose the more modern one." Falls short: preference-driven.
**Strong senior answer**: Put both proposals in the same ADR format and extract criteria: consistency needs, latency, coupling, team autonomy, delivery date, operational skill. Often each design optimises a different attribute: the shared DB gives simple strong consistency fast; events give decoupling but eventual consistency and more machinery. Weight criteria with the product owner using the ranked NFRs. Identify the riskiest assumption in each and run a short prototype or load test. Consider hybrids: start with the simple approach behind an interface, with outbox events published for future consumers. Decide with a named decider, document dissent, set a revisit trigger. Validate with the agreed metrics.
**Follow-up traps**: "Shared DB across teams?" (coupling; ownership issues). "Hybrid just a compromise?" (only if it satisfies top criteria).

### 17.4 [Medium] After choosing Service Bus, how do you guarantee that a DB write and a message publish don't diverge?
**What the interviewer is really testing**: Concrete consistency pattern.
**Typical mid-level answer**: "Use a distributed transaction." Falls short: unsupported/fragile at cloud scale.
**Strong senior answer**: Use the transactional outbox: write business data and an outbox row in the same SQL transaction; a relay (background service or CDC) publishes to Service Bus and marks sent. This gives at-least-once publishing, so consumers must be idempotent (message id dedup table or natural idempotency). Alternatives: Service Bus transactions only cover broker operations; dual writes risk loss; 2PC is unattractive. Trade-offs: an extra table and relay, and delivery latency. Libraries (e.g. MassTransit/Wolverine outbox) offer implementations, but check licensing. Validate with failure-injection: kill the process between commit and publish and confirm eventual delivery.
**Follow-up traps**: "Ordering?" (sessions or per-key partitioning). "Duplicate detection feature?" (window-limited; still design idempotent).

### 17.5 [Hard] Event schema evolution: consumers you don't control depend on your events. What's your approach?
**What the interviewer is really testing**: Contract stewardship.
**Typical mid-level answer**: "Just add a version field." Falls short: no compatibility policy.
**Strong senior answer**: Treat events as public contracts: additive-only changes within a version, tolerant readers (ignore unknown fields), explicit major versions for breaking changes running in parallel during a deprecation window, a schema registry or repository with CI compatibility checks, and consumer-driven contract tests where feasible. Prefer fat enough events to avoid chatty lookups but avoid leaking internal models. Trade-offs: slower change vs consumer stability. Document ownership and a contact. Validate by checking schema diffs in CI and monitoring consumers per version before retirement.
**Follow-up traps**: "Rename a field?" (add new, deprecate old). "Event or command?" (events past tense, facts; commands intent).

**Red flags in answers**
- One broker for everything without reasoning.
- Claiming exactly-once delivery.
- Treating a rewrite request as a technical decision only.

**Cheat sheet**
- Service Bus = commands/reliable work; Event Grid = notifications; Event Hubs = streams.
- Outbox + idempotent consumers.
- Rewrite request: diagnose pain, pilot strangler, checkpoints.
- Conflicting designs: criteria, prototype, named decider.
- Events are contracts: additive-only, versioned.
- Always state a revisit trigger.

---

## Topic 18: Scenarios II — inherited system with no docs, greenfield vs enterprise, ORM/data access

**Study notes**
- Inherited system: stabilise (observability, backups, access), learn (telemetry, code archaeology, interviews), map (C4, data flows), protect (characterization tests, CI), then improve in priority order.
- Greenfield startup vs enterprise: startup optimises time-to-learn and cost, tolerates debt, minimal ops; enterprise optimises compliance, integration, supportability, and existing standards. Same principles, different weights.
- Data access options for .NET: **EF Core** (productivity, change tracking, migrations, LINQ; use `AsNoTracking`, compiled queries, `ExecuteUpdate/ExecuteDelete`, projections), **Dapper** (micro-ORM, raw SQL, fast, minimal mapping), **ADO.NET** (max control), stored procedures. Hybrids common: EF Core for writes/domain, Dapper or raw SQL for hot reads/reports.

| Criterion | EF Core | Dapper | Raw ADO.NET |
|---|---|---|---|
| Productivity | High | Medium | Low |
| Peak throughput control | Good if tuned | Very good | Best |
| Migrations/schema tooling | Built-in | None | None |
| Query complexity | LINQ limits for complex SQL | Full SQL | Full SQL |
| Risk | N+1, tracking overhead | SQL strings, manual mapping | Boilerplate |

### 18.1 [Scenario] You inherit a production system with no documentation and the original team gone. What are your first weeks?
**What the interviewer is really testing**: Calm, structured discovery with risk-first priority.
**Typical mid-level answer**: "Read the code from the start." Falls short: slow and unprioritised.
**Strong senior answer**: Week 1, safety: confirm backups and restore ability, access/secrets inventory, monitoring and alerts, who the users and stakeholders are, and recent incident history. Then discover from outside in: Azure resource inventory, application map in Application Insights, dependency/call graphs, config and IaC (or lack), database schema and heavy queries (Query Store), and deployment process. Interview support staff, users, and any former-team contacts. Produce a C4 Context/Container diagram, README that runs on a clean machine, and a risk list (SPOFs, unsupported versions, secrets in config, no tests). Add characterization tests and CI around the riskiest paths before touching them. Prioritise improvements by risk and change frequency; write retroactive ADRs marked reconstructed. Validate by a newcomer onboarding successfully and by safe, boring deployments.
**Follow-up traps**: "Fix things immediately?" (not before you can observe and revert). "Rewrite?" (no, not without evidence).

### 18.2 [Scenario] Greenfield: a startup with six engineers vs a regulated enterprise. How does the same product architecture differ?
**What the interviewer is really testing**: Context-sensitive weighting.
**Typical mid-level answer**: "Startups use microservices to scale; enterprises are monolithic." Falls short: stereotype.
**Strong senior answer**: Principles are constant; weights differ. Startup: biggest risks are building the wrong thing and running out of money, so optimise learning speed and cost: modular monolith on App Service/Container Apps, one Azure SQL database, managed identity, CI/CD from day one, basic observability, and few moving parts; defer multi-region and heavy governance but keep one-way doors tidy (tenant id, versioned APIs, IaC). Enterprise: compliance, audit, integration with identity/network landing zones, data residency, approved services and patterns, change management, DR requirements. Design within the platform (private endpoints, Azure Policy, centralised logging), accept slower cycles, and invest in documentation and operability. Trade-offs: speed vs control. Validate with assumptions revisited at growth thresholds (users, team size, compliance).
**Follow-up traps**: "Is the startup design irresponsible?" (no; deliberate, with revisit triggers). "Enterprise too slow?" (paved roads and automation).

### 18.3 [Scenario] Select an ORM/data access strategy for a high-throughput .NET service on Azure SQL.
**What the interviewer is really testing**: Evidence-based tooling choice, not tribalism.
**Typical mid-level answer**: "Dapper is faster so use Dapper." or "EF Core always." Falls short: ignores context.
**Strong senior answer**: First characterise the workload: read/write mix, query complexity, target throughput and latency, team skills, schema change cadence. EF Core brings productivity, migrations, and safer refactoring; with `AsNoTracking`, projections, compiled queries, batching, and `ExecuteUpdate`, it is adequate for most services. Dapper/raw SQL helps on the hot read paths or complex reporting queries where precise SQL and minimal allocation matter. A common decision: EF Core for domain writes and most reads, Dapper (or SQL via `FromSql`) for a handful of measured hotspots, all behind one data-access layer. Decide after a benchmark with realistic data and concurrency, not microbenchmarks; often the bottleneck is indexing, round trips, or connection pooling, not the ORM. Also consider async everywhere, connection pool limits vs instance scale-out, retry on transient faults (`EnableRetryOnFailure`), and Query Store monitoring. Record in an ADR; reversible because the data access sits behind a boundary and the schema is owned by migrations.
**Follow-up traps**: "N+1?" (include/projection, query count tests). "Stored procedures?" (valid; consider versioning/test cost). "Bulk inserts?" (SqlBulkCopy or EF bulk libraries — check licence).

### 18.4 [Medium] How do you validate an ORM benchmark isn't misleading?
**What the interviewer is really testing**: Measurement rigor.
**Typical mid-level answer**: "Run a loop 1000 times." Falls short: unrealistic.
**Strong senior answer**: Use realistic data volume, indexes, and concurrency; test end-to-end through the service, including serialization and connection pooling. Measure p50/p95/p99, throughput, CPU/memory/allocations, DB CPU/IO, and query plans. Warm up, repeat runs, and compare on the same hardware; use BenchmarkDotNet for micro-level, Azure Load Testing or k6 for end-to-end. Ensure both implementations have good SQL; a poorly written EF query versus a tuned Dapper query is not a fair test. Look at where time is spent; if 95% is in SQL, the ORM isn't the issue. Validate by reproducing results and checking against production telemetry.
**Follow-up traps**: "Cold start?" (include warm-up). "Single run?" (insufficient).

### 18.5 [Tricky] How would you make the data-access decision reversible?
**What the interviewer is really testing**: Seams where change is likely.
**Typical mid-level answer**: "Repository pattern everywhere." Falls short: leaky and heavy.
**Strong senior answer**: Keep the schema as the stable contract, owned by migrations, so the access technology can change. Define data access around use cases (query handlers/repositories that return domain or read models) rather than exposing `IQueryable` outward. Allow different implementations per use case (EF for writes, SQL for reports). Integration tests against a real SQL container check behaviour, so swapping implementation doesn't change results. Trade-off: some duplication of mapping; avoid generic repository over EF, which adds little. Validate by swapping one hot query's implementation and confirming tests pass.
**Follow-up traps**: "Generic repository?" (usually unneeded over DbContext). "Unit of work?" (DbContext already is).

**Red flags in answers**
- Starting changes on an inherited system before backups and observability.
- Stereotyping startups/enterprises.
- ORM choice by benchmark folklore.

**Cheat sheet**
- Inherited: safety -> discover -> map -> protect -> improve.
- Same principles, different weights (startup vs enterprise).
- EF Core default; Dapper/SQL for measured hotspots.
- Bottlenecks are usually indexes, round trips, pool sizes.
- Benchmark realistically; test with real SQL.
- Keep schema contract stable.

---

## Topic 19: Complete ADR walkthrough & "What is your architectural philosophy?"

**Study notes**
- Below is a **realistic, labelled example ADR** (illustrative — not from a real company). Notice: problem separate from solution, measurable drivers, real alternatives, honest negatives, revisit triggers, reversibility.
- Philosophy answers: concise principles + examples of how they shaped decisions (use your own real examples; here shown as a template).

```markdown
# ADR-0012: Use Azure Service Bus (with transactional outbox) for order events
Status: Accepted      Date: <YYYY-MM-DD>      Deciders: Tech Lead, Product Owner, Ops Lead
(Illustrative example)

## Context
The Orders service must notify Billing, Inventory and Notifications when an order is placed.
Drivers (ranked): (1) no lost orders; (2) Billing may be down up to 1 hour without data loss;
(3) peak 300 orders/min, expected 5x in 12 months; (4) team of 6, limited ops capacity;
(5) per-customer ordering for updates. Today services call each other synchronously over HTTP;
an outage in Billing causes failed checkouts.

## Options considered
1. Keep synchronous HTTP calls with retries — simplest; couples availability; retry storms.
2. Azure Service Bus topics + subscriptions + outbox — durable, DLQ, sessions for ordering,
   decouples availability; adds an outbox table, idempotent consumers, new dependency.
3. Event Hubs — high throughput but no per-message DLQ/ack semantics; overkill at this volume.
4. Event Grid — good for notifications, weaker fit for ordered, transactional workflows.
5. Do nothing — checkout outages continue.

## Decision
We will publish order events to a Service Bus topic via a transactional outbox in the Orders SQL
database. Each subscriber uses its own subscription, peek-lock, idempotent handlers (message id
dedup), session id = customerId where ordering matters, and a DLQ alert.
Standard tier initially; revisit Premium if throughput or isolation needs grow.

## Consequences
+ Billing/Inventory outages no longer break checkout; replayable via DLQ resubmission.
+ Clear contract: versioned event schema (additive-only).
- New operational surface: topic/subscriptions, DLQ monitoring, schema governance.
- Eventual consistency between services; UI must handle "pending" states.
- Outbox relay is a new component to monitor.
Risks/mitigations: duplicate delivery (idempotency tests), relay lag (alert on outbox age).

## Revisit when
Volume > 10x current, a need for streaming/replay analytics, or cost > X per month.
Reversibility: events are consumed behind an IEventPublisher boundary; switching broker needs
adapter changes plus consumer migration (medium effort).
```

### 19.1 [Easy] Walk me through why this ADR is good (or what you'd challenge).
**What the interviewer is really testing**: Review and ADR literacy.
**Typical mid-level answer**: "It's detailed." Falls short: no evaluation criteria.
**Strong senior answer**: It states ranked drivers with numbers, so options can be judged. It includes "do nothing" and rejected options with reasons. Consequences list real negatives (operational surface, eventual consistency) and mitigations. It has a revisit trigger and reversibility note. I'd challenge: Is Standard tier enough for VNet/private endpoint requirements (Premium needed for private networking)? Are cost numbers estimated? Has a spike validated the outbox relay throughput? Has the UI team agreed to handle "pending" states? Those challenges are questions, not blockers, unless security or networking requirements demand Premium.
**Follow-up traps**: "Who signs?" (named deciders). "Length?" (one page).

### 19.2 [Follow-up] Where does the outbox pattern fit in the ADR; is it a separate decision?
**What the interviewer is really testing**: Decision granularity.
**Typical mid-level answer**: "Same thing." Falls short: ignores coupling of decisions.
**Strong senior answer**: Messaging platform and the consistency pattern are related but separable. If the outbox could apply with any broker and will be reused, a separate ADR enables independent change and reuse. If it's part of the same decision (we chose the broker because of the consistency approach), one ADR is fine. Rule: one ADR per decision that can be reversed independently. Cross-link them. Overly granular ADRs create noise; overly large ones hide decisions.
**Follow-up traps**: "How many ADRs per project?" (dozens over years, not hundreds). "Where stored?" (in repo).

### 19.3 [Medium] What is your architectural philosophy?
**What the interviewer is really testing**: Coherent, authentic principles that you apply.
**Typical mid-level answer**: "Clean architecture and SOLID." Falls short: pattern names, not philosophy.
**Strong senior answer**: Template — fill with your own examples: (1) **Start from the problem and the quality attributes that matter**, quantified, not from technology. (2) **Prefer simple, boring, managed solutions** and add complexity only when evidence demands it. (3) **Spend design effort on one-way doors** (data, contracts, boundaries) and keep the rest reversible. (4) **Make trade-offs explicit and written** (ADRs), including what we gave up. (5) **Design for operability and testability from day one**, because systems are lived in, not just built. (6) **Automate the guardrails** (fitness functions, pipelines) so intent survives people. (7) **Optimise for team flow and cognitive load**; architecture mirrors the organisation. (8) **Stay empirical**: prototype, measure, and revise; be willing to be wrong in public. Then give one concrete example from your experience for any two of these [your real example here].
**Follow-up traps**: "Give an example of changing your mind." (use your real one). "Is it different for startups?" (weights change, principles hold).

### 19.4 [Tricky] "Your philosophy sounds generic. What would you do differently from a textbook?"
**What the interviewer is really testing**: Practical judgment and honesty.
**Typical mid-level answer**: Defends dogma. Falls short: no adaptation.
**Strong senior answer**: Textbooks present patterns in isolation; reality has constraints: team size, skills, deadlines, legacy, budget. I adapt: modular monolith over microservices unless team-scale forces otherwise; a few well-chosen tests over complete coverage; documentation focused on the why; and incremental migration over tidy redesign. I treat patterns as options with costs, and when I break a rule I record why. Validate by asking whether the team can operate and change the system; if not, the design has failed regardless of elegance.
**Follow-up traps**: "When do you follow the book?" (security basics, data integrity). "Pattern you now avoid?" (use own experience).

### 19.5 [Scenario] You have 5 minutes to explain your decision-making process to a new team. What do you say?
**What the interviewer is really testing**: Communication of a repeatable method.
**Typical mid-level answer**: "I use best practices." Falls short: not transferable.
**Strong senior answer**: Five steps: (1) clarify the problem and measurable goals; (2) identify constraints and which decisions are hard to reverse; (3) generate at least two options, including "do nothing"; (4) evaluate against ranked criteria, spike the riskiest assumption; (5) decide, write a short ADR with consequences and revisit trigger, then measure outcomes. I'd add team norms: disagree openly before the decision, commit after; two-way doors are decided by the team quickly; one-way doors get wider review. Validate by the team using the ADR flow on their next real decision.
**Follow-up traps**: "What if time is short?" (shorten, not skip, steps 1 and 3). "Who decides?" (named owner).

**Red flags in answers**
- Philosophy = a list of pattern names.
- No examples of decisions that changed because of evidence.
- ADRs that only justify a decision already made.

**Cheat sheet**
- A good ADR: ranked drivers, real alternatives, honest negatives, revisit trigger.
- One ADR per independently reversible decision.
- Philosophy: problem-first, simple, one-way doors, explicit trade-offs, operability, automation, empiricism.
- Use your own real examples; never borrow.
- Decide -> record -> measure -> revisit.
- Disagree before, commit after.

---

*End of S08. Facts about product features/licensing change; verify against current Microsoft and vendor documentation before relying on them in a decision.*

