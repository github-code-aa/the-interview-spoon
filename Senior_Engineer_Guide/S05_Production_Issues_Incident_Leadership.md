# S05 - Production Issues & Incident Leadership (Senior .NET / Azure / SQL Server)

**How to use:** Read each topic's study notes, then answer every question aloud before reading the senior answer; aim for "stabilise -> diagnose -> fix -> prevent" structure.
Personal stories are never invented here: use the labelled STAR templates and fill them with your own real incidents and metrics.

## Table of Contents
1. Severity Levels & the First 10 Minutes
2. Incident Roles, Status Updates & Stakeholder Communication
3. Generic Triage Framework (What Changed? RED/USE/Golden Signals)
4. "The API Is Slow" - Structured Investigation
5. .NET Runtime Incidents: CPU, Memory, Thread Pool, GC
6. Intermittent 500s / Timeouts Under Load
7. Database Incidents (SQL Server / Azure SQL)
8. Messaging Incidents (Service Bus / Queues)
9. Cache Incidents (Redis)
10. Deployment-Caused Incidents & Expand-Contract Migrations
11. Third-Party / Dependency Outages
12. Data Corruption & Incorrect-Data Incidents
13. Security Incidents
14. Capacity & Cost Incidents
15. Monitoring, Alerting Quality & On-Call Sustainability
16. Blameless Postmortems & Near-Misses
17. Reproduction Strategies & Safe Production Debugging
18. Change Management, Release Safety & Post-Incident Prevention
19. Six End-to-End Incident Scenarios
20. Blame, Pressure & Telling Your Production Story (incl. "Worst Incident You Caused")

---

## Topic 1: Severity Levels & the First 10 Minutes

**Study notes**
- Severity is about **impact**, not cause: Sev1 = revenue/data/security/safety impact on many users, no workaround; Sev2 = major feature degraded or subset of customers; Sev3 = minor/workaround exists; Sev4 = cosmetic. Define it in advance with objective triggers (e.g. "checkout error rate > 5% for 5 min").
- **Stabilise before root-cause.** The goal in the first minutes is to stop the bleeding (mitigate), not to understand. Understanding comes after users are OK.
- Mitigation menu (fastest/safest first): rollback / slot swap back -> disable feature flag -> scale out / restart -> fail over / shed load -> traffic shift/block bad client -> fix-forward (last, riskiest).
- Blast radius: who (tenants, regions, % users), what (read vs write, data at risk), since when (start time = correlates with a change?).

```
Alert/Report
   |
   v
Acknowledge (1 min) -> Is it real? (dashboard, synthetic, App Insights failures blade)
   |
   v
Assign severity + name IC (2-3 min) -> open bridge/channel, start timeline log
   |
   v
Blast radius (who/what/since when) + "what changed?" (deploys, flags, config, infra, Azure Service Health)
   |
   +-- change in last hour & reversible?  --YES--> ROLLBACK / flag off
   |                                      --NO--> scale/failover/shed load, then diagnose
   v
First comms (<=10 min): impact, status, next update time
```

**Rollback vs fix-forward:** rollback if the change is the likely cause, rollback is tested and has no data-schema incompatibility; fix-forward if rollback is unsafe (irreversible migration, already-written data in new format) and the fix is small, reviewed and tested. Never fix-forward "because rollback feels like failure".

### 1.1 [Easy] What do you do in the first 10 minutes of a Sev1?
**What the interviewer is really testing:** Do you prioritise mitigation and coordination over heroic debugging?
**Typical mid-level answer:** "I'd check the logs and find the root cause." Falls short: dives deep alone, no comms, no roles, no mitigation-first mindset.
**Strong senior answer:** First I acknowledge and confirm the issue is real using an independent signal (availability test, App Insights failures, not just one customer email). I declare severity from agreed impact criteria and make sure there's a single incident commander; if nobody is, I take it until relieved. I open a bridge/channel and start a timestamped timeline. Then I size the blast radius (which users/regions/operations, since when) and ask "what changed?" - deployments, feature flags, config, certs/secrets, infra, and Azure Service Health. If a recent reversible change correlates, I roll back or flip the flag immediately; otherwise I use generic mitigations (scale out, restart, fail over, shed load). I send first comms within ~10 minutes with impact, what we know, and the next update time. Only once users are protected do we dig into root cause, preserving evidence (dumps, logs) before restarting where feasible.
**Follow-up traps:** "What if you restart and lose the evidence?" (capture dump/logs first if cheap, or take one instance out of rotation). "Who decides rollback?" (IC, with the owning engineer's input).

### 1.2 [Follow-up] Rollback or fix-forward - how do you decide?
**What the interviewer is really testing:** Risk reasoning, awareness of irreversible changes.
**Typical mid-level answer:** "Always roll back." Or "just push a quick fix." Both are dogma.
**Strong senior answer:** I compare time-to-mitigate and risk. Rollback is preferred when the deploy is the prime suspect, the previous version is known-good, and the rollback path is rehearsed (slot swap back, previous container tag, pipeline redeploy). It's unsafe when a migration or data write is not backward-compatible - then old code may fail against the new schema or mis-read new data. That is exactly why we use expand-contract so rollback stays possible. Fix-forward is justified when the fix is a one-liner with a clear causal link, can go through an expedited pipeline with a second reviewer, and rollback has its own risk. I also consider that a rollback that doesn't resolve the problem is itself diagnostic: it proves the deploy wasn't the cause. I'd time-box: "if no improvement 15 minutes after rollback, we revert to other hypotheses".
**Follow-up traps:** "Rollback didn't help - now what?" (change wasn't the cause or there were two causes; check dependencies/data). "Hotfix bypassing the pipeline?" (only with explicit IC approval and retrospective review).

### 1.3 [Medium] How do you determine blast radius quickly?
**What the interviewer is really testing:** Using telemetry dimensions rather than guessing.
**Typical mid-level answer:** "Ask support how many users complain." Anecdotal, slow, biased.
**Strong senior answer:** I slice telemetry by dimensions: region, tenant/customer id, app version, endpoint, instance, dependency target, client type. In App Insights I compare failure rate per `cloud_RoleInstance`, `appVersion` and `resultCode`; if only one instance is bad it's a node problem, if one version it's a deploy, if one tenant it's data-specific, if one dependency it's downstream. I also determine the *type* of damage: read-only errors, failed writes, silently wrong data (the worst), or security exposure. Start time matters: I look for the inflection point and correlate it to the change log. This shapes the mitigation (e.g. route away one region) and the comms (who's affected).
```kusto
requests
| where timestamp > ago(1h)
| summarize total=count(), failed=countif(success==false) by bin(timestamp,5m), cloud_RoleInstance, tostring(customDimensions.appVersion)
| extend failRate = todouble(failed)/total
| order by timestamp asc
```
**Follow-up traps:** "Telemetry itself is sampled/down?" (use platform metrics, load balancer/Front Door logs). "Silent data errors?" (need invariants/reconciliation queries).

### 1.4 [Hard] Customers are reporting errors but your dashboards are all green. What now?
**What the interviewer is really testing:** Healthy skepticism of monitoring; knowing blind spots.
**Typical mid-level answer:** "Dashboards are green so it's probably the customer." Dismissive.
**Strong senior answer:** Customer reports are signal; green dashboards mean my monitoring has a blind spot, which is itself a finding. I reproduce from the customer's perspective: their tenant, region, client, path (CDN/Front Door/WAF in front?). I check things dashboards often miss: requests blocked before reaching the app (WAF/Front Door logs), 4xx that are counted as "success", errors swallowed by catch blocks returning 200, sampling hiding rare failures, a single tenant or feature path with low volume, client-side errors (browser console, mobile). I query raw logs for the customer's correlation id or user id. Once found, I mitigate, and afterwards add a symptom-based alert or synthetic test for that gap.
**Follow-up traps:** "How do you correlate a customer complaint to a trace?" (return/ log operation id, X-Request-Id on error pages). "Is sampling to blame?" (check sampling percentage; exclude failures from sampling).

### 1.5 [Tricky] The CEO joins the bridge and demands the root cause now. What do you do?
**What the interviewer is really testing:** Composure and prioritisation under hierarchy pressure.
**Typical mid-level answer:** Starts speculating a cause to satisfy them. Falls short: premature, may be wrong, erodes trust.
**Strong senior answer:** The commander (or I, if IC) acknowledges the concern and redirects to what leadership can actually use: impact, current mitigation status, ETA confidence, and the time of the next update. I state clearly what we know, what we don't, and that root cause analysis follows mitigation. I never guess a cause publicly - a wrong guess sends people down the wrong path and creates a false story that sticks. I offer a single liaison so executives don't interrupt engineers (they get updates in the comms channel). After resolution, the postmortem will provide the cause with evidence.
**Follow-up traps:** "They insist on a cause?" (offer 'leading hypotheses, unconfirmed' clearly labelled). "Engineers get pulled to brief execs?" (comms lead absorbs it).

### 1.6 [Scenario] Alerts fire at 02:10: error rate 30%, deploy finished at 02:00. Walk through your first minutes.
**What the interviewer is really testing:** Applying the decision tree concretely.
**Typical mid-level answer:** "Look at the logs for the exception." No mitigation decision.
**Strong senior answer:** The 10-minute gap between deploy and alert strongly suggests the deploy, so I confirm by splitting failures by app version/slot instance. If the new version carries all the failures, and the deploy had no irreversible migration, I roll back immediately (slot swap back) and watch the error rate drop - this is both mitigation and confirmation. I page the author of the change for the diagnosis phase but don't wait for them. I post a first update stating impact, that we suspect the 02:00 release, and that rollback is in progress. After recovery, I pull the exception from the new version's traces (`exceptions | where appVersion==...`), reproduce in staging, and write a fix with a test, then redeploy through the normal pipeline. Prevent: canary/slot warm-up with health gating, automatic rollback on error rate.
**Follow-up traps:** "Rollback fails because migration ran?" (forward-compatible schema design; else fix-forward under IC control). "Why didn't staging catch it?" (parity gap - becomes an action item).

**Red flags in answers**
- Jumping to root-cause hunting while users are still impacted.
- No mention of who is coordinating or how stakeholders are updated.
- Guessing causes publicly or restarting everything without capturing evidence.

**Cheat sheet**
- Mitigate first, understand second.
- Severity = impact, defined in advance.
- "What changed?" is the highest-yield question; check deploys, flags, config, certs, Service Health.
- Rollback needs backward-compatible schema; design for it (expand-contract).
- First comms within ~10 min with a next-update time.
- Slice telemetry by version/instance/tenant/dependency to size blast radius.

---

## Topic 2: Incident Roles, Status Updates & Stakeholder Communication

**Study notes**
- Roles (borrowed from ICS/SRE): **Incident Commander (IC)** coordinates and decides, does *not* debug; **Ops/Tech lead** drives investigation and executes changes; **Communications lead** writes updates to stakeholders/customers; **Scribe** keeps the timeline; **Subject-matter experts** pulled in as needed. On small teams one person may hold two roles - but never IC + hands-on debugger on a Sev1.
- Update cadence: Sev1 every 15-30 min even if "no change"; each update has: **Impact - Status - What we're doing - Next update at**.
- Separate channels: war-room (engineers) vs status channel (stakeholders) vs customer status page.
- Handoffs: explicit ("You are now IC") with state summary; incidents longer than ~4h need rotating relief.

```
Update template
[HH:MM] SEV1 - <short title>
Impact:   who/what is affected, % if known
Status:   Investigating | Identified | Mitigating | Monitoring | Resolved
Actions:  what is being done now
Next update: HH:MM (even if no news)
```

### 2.1 [Easy] What does an incident commander do, and why shouldn't they debug?
**What the interviewer is really testing:** Understanding of coordination vs execution.
**Typical mid-level answer:** "The most senior person who fixes it." Falls short: confuses authority with hands-on work.
**Strong senior answer:** The IC maintains the big picture: assigns roles, keeps hypotheses parallelised without duplication, makes the rollback/escalate decisions, controls the comms cadence, and decides when to declare resolved. If the IC is deep in a debugger they lose situational awareness - tunnel vision is the classic failure. So I delegate investigation to an ops lead and ask for periodic summaries ("what do we know, what are we trying, what do you need"). The IC also protects the engineers from interruptions by executives. I'd also call out people who've been awake too long and arrange relief.
**Follow-up traps:** "Small team of three?" (IC is the one who doesn't touch the keyboard; scribe/comms can be combined). "Who's the IC if the manager is absent?" (anyone trained; the role is a function, not rank).

### 2.2 [Follow-up] Write the first customer-facing update for a payment outage.
**What the interviewer is really testing:** Clear, honest, non-speculative communication.
**Typical mid-level answer:** "We are experiencing technical difficulties." Vague, no impact or next update.
**Strong senior answer:** Customer comms should be plain-language, factual, and time-stamped: "We're investigating an issue where some card payments are failing (started about 14:05 UTC). Orders already placed are not affected; you may retry in a few minutes. We'll update by 14:45 UTC." I avoid internal jargon, blame of vendors without confirmation, root-cause speculation, and promises of ETAs I can't back. For internal stakeholders I add specifics: affected percentage, mitigation underway, support talking points. If money may have been double-charged or data exposed, legal/compliance get looped in before wording goes out. I commit to the next-update time and meet it even with "no change".
**Follow-up traps:** "Should you give an ETA?" (only with confidence and a range; say how confidence will be updated). "Who approves external wording?" (comms lead + IC; legal for data/security).

### 2.3 [Medium] An exec asks every 5 minutes for status; engineers are getting distracted. How do you handle it?
**What the interviewer is really testing:** Protecting focus while keeping stakeholders informed.
**Typical mid-level answer:** Answers each ping personally. Falls short: kills diagnostic throughput.
**Strong senior answer:** I fix the cause of the pings: lack of predictable information. I set a communication contract - a named comms lead, a status channel, and a next-update time that I keep. I invite the exec to the status channel rather than the war-room, and brief them one-on-one at the start so they know the structure. I explain that pings slow the fix, and show that the cadence will give them what they need. If the exec needs a decision (budget, customer call), the IC presents options with a recommendation rather than raw technical detail.
**Follow-up traps:** "They insist on joining the call?" (let them listen on mute / give them a liaison). "Status says 'monitoring' too long?" (define exit criteria).

### 2.4 [Hard] How do you decide an incident is resolved, and how do you communicate recovery?
**What the interviewer is really testing:** Avoiding premature "all clear".
**Typical mid-level answer:** "When the error goes away." Falls short: no verification, no monitoring window.
**Strong senior answer:** Resolved requires objective exit criteria: key SLIs back to baseline for a defined window (e.g. 30 min), backlog drained, no residual customer-visible symptoms, and mitigations (feature flags off, extra capacity) tracked as temporary debt. For data-affecting incidents "resolved" also needs a repair/reconciliation plan with an owner. I move through states Mitigated -> Monitoring -> Resolved, and the recovery message states impact window, what's fixed, residual actions and that a postmortem will follow. I keep a pager owner for the post-incident window since recurrences are common.
**Follow-up traps:** "Backlog still draining?" (not resolved for customers waiting; communicate delayed processing). "What if root cause unknown?" (Resolved-mitigated; keep a problem ticket open).

### 2.5 [Tricky] Two senior engineers on the bridge disagree strongly about the fix. What do you do?
**What the interviewer is really testing:** Decision-making under disagreement.
**Typical mid-level answer:** Lets the louder one win. Falls short.
**Strong senior answer:** As IC I turn opinions into testable hypotheses with cheap discriminating checks, and parallelise: assign each engineer to prove or disprove their own theory within a time box. When both fixes are safe and independent, try the less risky/more reversible first. If they are mutually exclusive, I choose by risk and reversibility, record the decision and rationale in the timeline, and ask the dissenter to flag evidence that would change the decision. Afterwards the disagreement goes into the postmortem as a learning, not a grievance.
**Follow-up traps:** "You aren't the most technical in the room?" (IC doesn't need to be; they need to structure the decision). "Both are wrong?" (time-box helps avoid sunk cost).

### 2.6 [Scenario] You're the on-call engineer and no IC has been named for a clearly Sev1 incident, with ten people chatting. What do you do?
**What the interviewer is really testing:** Leadership without title.
**Typical mid-level answer:** Waits for a manager. Falls short: leaderless chaos persists.
**Strong senior answer:** I say in the channel: "I'm taking IC until someone more suitable relieves me." Then I assign: ops lead, comms lead, scribe, and ask everyone else to stay quiet unless they have evidence or are assigned. I state the current known facts and the first three actions with owners and time-boxes. I post the first stakeholder update with next update time. The structure costs two minutes and prevents duplicated work, conflicting changes (two people deploying at once) and silent stakeholders. A "change freeze except via IC" rule is declared.
**Follow-up traps:** "Someone senior shows up and takes over?" (hand over with a crisp state summary). "People ignore you?" (anchor to the on-call policy that gives IC authority).

**Red flags in answers**
- IC who is also the one deep in debugging a Sev1.
- Vague updates without impact, status or next-update time.
- No exit criteria for declaring resolved.

**Cheat sheet**
- Roles: IC, Ops, Comms, Scribe.
- Update = Impact / Status / Actions / Next update.
- Keep war-room and stakeholder channels separate.
- IC doesn't debug; escalates, decides, protects focus.
- Resolved needs criteria + monitoring window.
- Hand over explicitly; relieve tired people.

---

## Topic 3: Generic Triage Framework (What Changed? RED/USE/Golden Signals)

**Study notes**
- **Four golden signals** (Google SRE): latency, traffic, errors, saturation. **RED** (services): Rate, Errors, Duration. **USE** (resources): Utilization, Saturation, Errors per CPU/memory/disk/network/pool.
- Triage loop: Observe symptom -> Form 2-3 hypotheses ranked by likelihood x cheapness-to-test -> Pick discriminating test -> Act/mitigate -> Re-observe.
- "What changed?": code deploys, config/app settings, feature flags, infra (scale, SKU, networking, NSG, DNS), data (volume, new tenant, bad record), traffic pattern, dependency changes, **time-based** (cert/secret expiry, cron job, month-end, DST), Azure platform events.
- Differential: *all* instances vs one; *all* endpoints vs one; *all* users vs a cohort; sudden vs gradual (step = change; ramp = leak/growth; sawtooth = periodic job/GC).

```
Symptom shape        -> likely class
Step change          -> deploy/config/flag/expiry/dependency
Gradual ramp         -> leak, growth, fragmentation, queue backlog
Periodic spikes      -> cron/batch, GC, cache TTL expiry, autoscale flapping
Only under load      -> saturation: pool, threads, locks, DTU/CPU
One instance only    -> bad node, noisy neighbor, poisoned warm state
```

### 3.1 [Easy] What is your first question in any production issue, and why?
**What the interviewer is really testing:** Instinct for highest-yield diagnostics.
**Typical mid-level answer:** "Check the logs." Falls short: unfocused.
**Strong senior answer:** "What changed?" - most incidents are caused by a change in code, config, data, traffic, infrastructure, or time (expiries, cron). I check the deployment history, feature flag audit log, Azure Activity Log (resource changes), recent config changes, Azure Service Health, and then traffic/data shape changes (a new large tenant, a marketing campaign). Correlating the incident start to a change gives a ranked hypothesis list in minutes. If nothing changed, that itself tells me to suspect gradual accumulation (leak, data growth, log full) or an external dependency.
**Follow-up traps:** "Nothing changed according to the team?" (check platform: Azure maintenance, cert auto-renewals, upstream vendor). "Who has the change log?" (should be a single place - fix if not).

### 3.2 [Follow-up] Explain RED vs USE and when each is used.
**What the interviewer is really testing:** Conceptual clarity of monitoring methods.
**Typical mid-level answer:** Names them but can't apply.
**Strong senior answer:** RED looks at a request-driven service from the user's perspective: rate, error rate, duration (percentiles, not averages). USE looks at each resource: utilization (% busy), saturation (queued work, e.g. run-queue, thread pool queue, pending connections), errors. I start with RED to confirm user pain and find the affected service/endpoint, then use USE to find which resource is the bottleneck. Golden signals add saturation explicitly. For example, p99 latency up (RED) with CPU 40% but thread pool queue length growing and pool wait times (USE/saturation) points at thread starvation rather than compute.
**Follow-up traps:** "Why not average latency?" (hides tails; use p95/p99 and histograms). "Where does queue depth fit?" (saturation).

### 3.3 [Medium] How do you generate and rank hypotheses without guessing?
**What the interviewer is really testing:** Scientific method under pressure.
**Typical mid-level answer:** Follows whichever idea comes first.
**Strong senior answer:** I write down symptoms precisely ("p99 on POST /orders 8s since 14:03, only westeurope, CPU flat, dependency SQL duration up"), then list candidate causes per layer: client/edge, app, runtime, dependency, data, infra. Rank by prior likelihood (recent change?), evidence consistency, and cost to test. I pick the test that *discriminates* between hypotheses, not one that confirms my favourite: e.g. compare dependency durations vs request durations to separate "app slow" from "database slow". I record eliminated hypotheses in the timeline so the team doesn't revisit them. I avoid confirmation bias by asking "what would I see if I'm wrong?".
**Follow-up traps:** "Multiple concurrent causes?" (yes, possible; remove one and re-measure). "Time-box?" (re-evaluate every 15 minutes).

### 3.4 [Hard] How do you separate correlation from causation during an incident?
**What the interviewer is really testing:** Rigor.
**Typical mid-level answer:** "The graphs moved at the same time so that's the cause."
**Strong senior answer:** Coincident timing is a lead, not proof. I look for a mechanism (how would X cause Y), a dose-response (more load -> more errors), specificity (only affected endpoints that use X), and reversibility (undoing X fixes Y). Where safe, I run the intervention: flag off, rollback, scale up - and watch the metric. I also check for a common cause: a traffic spike might cause both CPU rise and error rise. Controlled experiments (canary instance with the change reverted) beat inference.
**Follow-up traps:** "Metric moved after rollback but also after traffic drop?" (confounded; look at per-version split). "Intervention unsafe?" (use read-only evidence like dumps and traces).

### 3.5 [Tricky] A teammate says "it's the database" because DB CPU is high. Is that enough?
**What the interviewer is really testing:** Avoiding blaming the first hot resource; cause vs effect.
**Typical mid-level answer:** Agrees and starts tuning SQL.
**Strong senior answer:** High DB CPU may be the *effect*: a retry storm, a new query pattern from a deploy, a cache stampede after a Redis flush, or a plan regression. I ask what is consuming the CPU (Query Store top resource consumers, `sys.dm_exec_requests`), whether the workload changed (request counts per endpoint) and whether that began before or after other symptoms. If the app is hammering the DB due to cache misses, scaling the DB is a costly band-aid; fixing the cache or backoff is the cure. Sequence in time matters: the first domino, not the loudest.
**Follow-up traps:** "Scale up DB as mitigation?" (acceptable as temporary relief with a cost note; investigate in parallel).

### 3.6 [Scenario] You're paged: "latency up". You know nothing else. Give the first five queries/looks you'd run.
**What the interviewer is really testing:** Concrete, ordered tooling.
**Typical mid-level answer:** Vague "check metrics".
**Strong senior answer:** (1) App Insights Performance/Failures blade or KQL for p50/p95/p99 and failure rate by operation over the last 3 hours to find the step time and affected operation. (2) Split by version/instance/region to see if localised. (3) Dependencies table: duration and failure by target to see if a downstream slowed. (4) Platform saturation: App Service/AKS CPU, memory, thread pool and connection counts; SQL CPU/DTU, waits, blocking. (5) Change log and Service Health around the step time. Based on the result I choose a mitigation (rollback, scale, failover) before deeper tracing.
```kusto
requests
| where timestamp > ago(3h)
| summarize p50=percentile(duration,50), p95=percentile(duration,95), p99=percentile(duration,99), n=count() by bin(timestamp,5m), name
| order by timestamp asc
```
**Follow-up traps:** "Why percentiles?" (tails). "Dependencies table empty?" (no dependency tracking - fix instrumentation).

**Red flags in answers**
- Anchoring on the first hot graph without checking cause vs effect.
- Using averages for latency analysis.
- Never asking what changed.

**Cheat sheet**
- Ask "what changed?" first (code, config, flag, infra, data, traffic, time).
- RED for services, USE for resources, golden signals = latency/traffic/errors/saturation.
- Shape of the graph hints at cause (step/ramp/periodic).
- Discriminating tests beat confirming tests.
- Use interventions to prove causation.
- Record eliminated hypotheses.

---

## Topic 4: "The API Is Slow" - Structured Investigation

**Study notes**
- First define slow: which endpoint, what percentile, since when, for whom, compared to what baseline/SLO.
- Decompose latency: client -> network/CDN/WAF -> LB -> app queueing -> app CPU -> dependencies (SQL, Redis, HTTP, Service Bus) -> serialization -> response. Use end-to-end transaction view (App Insights, W3C trace context) to see where time goes.
- Common .NET causes: N+1 queries/EF lazy loading; missing index/plan regression; sync-over-async blocking (thread pool starvation); HttpClient misuse (new per request -> socket exhaustion, or no timeouts); large payloads/serialization; GC pauses; cold starts/scale-out; lock contention; chatty dependency calls; cache misses; slow external API.
- Distinguish **latency** (single request slow) from **throughput/queuing** (requests wait to start). If time-to-first-byte is long but dependency spans are short, suspect queuing/threads.

```
Slow? -> which endpoint/percentile/since
  |-> only that endpoint? -> its dependencies in trace (SQL? HTTP? cache?)
  |-> all endpoints?      -> shared resource: thread pool, CPU, GC, DB, network, scale
  |-> only under load?    -> saturation/queuing/locks
  |-> only first calls?   -> cold start, JIT, cache warm-up, connection setup
```

### 4.1 [Easy] A user says "the API is slow". What do you ask first?
**What the interviewer is really testing:** Clarifying before diagnosing.
**Typical mid-level answer:** Starts profiling the code.
**Strong senior answer:** I turn the vague report into measurable facts: which operation, how slow (seconds vs baseline), since when, how often (always/intermittent), who (one user/tenant/region/all), client (browser, mobile, partner system), and a sample request id/timestamp. Then I check what telemetry says for that operation versus its baseline and SLO. This frames whether it's a real regression or perceived (e.g. client-side render, user's network). It also lets me locate a specific trace to analyse rather than hunting generic averages.
**Follow-up traps:** "No baseline?" (use history in App Insights; also an action item to define SLOs). "Customer's on a bad network?" (compare server-side duration to client-observed).

### 4.2 [Follow-up] How do you find where the time goes in one slow request?
**What the interviewer is really testing:** Distributed tracing fluency.
**Typical mid-level answer:** Adds Stopwatch logging.
**Strong senior answer:** I open the end-to-end transaction in App Insights (or the trace for that operation_Id), which shows the request span and child dependency spans on a timeline. Gaps between spans with no dependency indicates app CPU, blocking, or queueing before the handler started. Long SQL spans -> capture the query and check its plan/Query Store. Many tiny spans -> N+1 or chatty calls. A long gap before the first span with a normal handler time -> thread pool queuing or startup. I compare a slow trace with a fast trace of the same operation: the diff is the answer.
```kusto
union requests, dependencies
| where operation_Id == "<id>"
| project timestamp, itemType, name, target, duration, success
| order by timestamp asc
```
**Follow-up traps:** "Sampling dropped the slow trace?" (adaptive sampling; search by id, lower sampling for failures). "No dependency spans for SQL?" (SDK/collector config).

### 4.3 [Medium] p50 is fine but p99 is terrible. What kind of problems cause that?
**What the interviewer is really testing:** Tail-latency understanding.
**Typical mid-level answer:** "Some requests are just slow."
**Strong senior answer:** Tails are typically from intermittent contention or pauses: GC (gen2/LOH collections), thread pool starvation bursts, lock contention, connection pool waits, retries with timeouts, cache misses on cold keys, noisy neighbour/one slow instance, cold starts, parameter-sniffing-affected inputs (certain parameter values pick a bad plan), and large result sets for big tenants. I segment p99 by customer/parameter size/instance/time to see if the slow requests share an attribute. For fan-out calls, tail amplification matters: calling N dependencies in parallel makes the overall latency the max. Mitigation depends on the finding - hedged requests, timeouts, caches, indexing, or pagination.
**Follow-up traps:** "Average is flat but customers complain?" (averages hide it). "How to confirm GC?" (dotnet-counters gc pause time / ETW events).

### 4.4 [Hard] Latency rose after a release but code diff looks harmless. What could be hiding?
**What the interviewer is really testing:** Non-obvious regressions.
**Typical mid-level answer:** "Revert it and see."
**Strong senior answer:** I'd mitigate with rollback if available, then diagnose, because harmless-looking diffs hide things: a library or framework upgrade (EF Core query translation change, serializer behaviour), a changed config (connection string pool size, timeouts, logging level raising I/O), new logging with sync I/O, a new middleware, EF query changed (client-side evaluation, Include producing cartesian explosion), different DB parameter types causing implicit conversions that kill index seeks, a feature flag combination, or a changed dependency version. I diff the deployed artifact (dependency lockfile), compare the generated SQL for the hot endpoint before/after, and compare Query Store plans. A statistical approach: run old and new versions side by side under replay load.
**Follow-up traps:** "Implicit conversion?" (nvarchar vs varchar parameter -> index scan; check actual plan warnings). "Replay in prod?" (use shadow traffic or staging with prod-like data).

### 4.5 [Tricky] CPU and memory are low but the API is slow and requests queue. What do you suspect?
**What the interviewer is really testing:** Understanding of waiting, not working.
**Typical mid-level answer:** "Scale up the VM."
**Strong senior answer:** Low CPU with high latency means threads are waiting, not computing. Suspects: thread pool starvation (sync-over-async `.Result`/`.Wait()` blocking), exhausted connection pool to SQL or HttpClient limits, lock contention, slow downstream with no timeout, or rate limiting/throttling (429, SQL resource governance). Evidence: `dotnet-counters` shows ThreadPool Queue Length growing and ThreadPool Thread Count climbing slowly, `dotnet-stack`/dump shows many threads parked in `Task.Wait`/`GetResult`; SQL `sys.dm_exec_requests` waits show where; App Insights shows long gaps before dependency spans. Scaling up cores wouldn't help; fixing the blocking calls, adding timeouts, and tuning pools would.
```
dotnet-counters monitor -p <pid> System.Runtime Microsoft.AspNetCore.Hosting
# watch: threadpool-queue-length, threadpool-thread-count, monitor-lock-contention-count
```
**Follow-up traps:** "Raise ThreadPool.SetMinThreads?" (band-aid that can help; real fix is async all the way). "Why does thread count grow slowly?" (hill-climbing injects ~1-2 threads per 500ms).

### 4.6 [Scenario] The /search endpoint went from 200ms to 6s for one large customer only. Walk through it.
**What the interviewer is really testing:** Data-skew awareness.
**Typical mid-level answer:** "Add an index."
**Strong senior answer:** Single-tenant slowness suggests data-dependent behaviour: plan chosen for typical cardinalities is bad for a large tenant (parameter sniffing), missing selective index, unbounded result set/pagination absent, or an N+1 that scales with entity count. I confirm the slow trace belongs only to that tenant and see which span dominates. If SQL: Query Store for that query - compare plans, check regressed plan and parameter values; mitigation could be forcing the good plan in Query Store or `OPTION (RECOMPILE)` for that query pending a code fix. If it's N+1, a hotfix to batch queries; longer-term, proper pagination and index. Prevent with load tests that include skewed data and an alert on per-tenant p95.
```sql
SELECT TOP 10 qt.query_sql_text, rs.avg_duration, rs.count_executions, p.plan_id
FROM sys.query_store_query q
JOIN sys.query_store_query_text qt ON q.query_text_id=qt.query_text_id
JOIN sys.query_store_plan p ON q.query_id=p.query_id
JOIN sys.query_store_runtime_stats rs ON p.plan_id=rs.plan_id
ORDER BY rs.avg_duration DESC;
```
**Follow-up traps:** "Forcing a plan is risky?" (monitor for forced-plan failures; temporary). "Don't want to touch SQL?" (cache/pagination on app side).

**Red flags in answers**
- Optimising before locating where the time is spent.
- Reaching for "scale up" regardless of the signals.
- Looking only at averages.

**Cheat sheet**
- Quantify "slow": endpoint, percentile, baseline, scope.
- Use end-to-end trace; compare slow vs fast trace.
- Gaps with no spans = queuing/blocking/CPU.
- Low CPU + slow = waiting (threads, pools, locks, throttling).
- Single-tenant slow = data skew/plan/N+1.
- Percentiles segmented by attribute reveal tails.

---

## Topic 5: .NET Runtime Incidents: CPU, Memory, Thread Pool, GC

**Study notes**
- Tools: `dotnet-counters` (live metrics), `dotnet-trace` (CPU sampling/events -> speedscope/PerfView), `dotnet-dump` (collect/analyze heap, threads), `dotnet-gcdump` (heap object graph, lightweight), Azure App Service **Diagnose and solve problems** (CPU/memory profiler, auto-heal, memory dump), Application Insights **Profiler / Snapshot Debugger**, Kudu/Process Explorer; in AKS `kubectl debug` ephemeral container or sidecar.
- **High CPU:** collect `dotnet-trace` (or Profiler) -> find hot methods: regex backtracking, tight retry loops, serialization, LINQ in loops, GC (high % time in GC), spin locks. Compare with request rate: more work or same work costing more?
- **Memory growing:** managed vs native; check `gc-heap-size`, gen0/1/2 counts, LOH size, `working-set`. Managed leak patterns: static collections/caches without eviction, event handler subscriptions not unsubscribed, `HttpClient`/timer/Task roots, captured closures, Dictionary growth by unbounded keys, `IMemoryCache` without size limit, finalizer queue backup. Take two `dotnet-gcdump`s minutes apart and diff.
- **Thread pool starvation:** symptoms - low CPU, high latency, `ThreadPool Queue Length` growing, thread count rising slowly. Cause: sync-over-async blocking, long-running sync work on pool threads. Fix: async all the way, no `.Result`; don't `Task.Run` in ASP.NET to "make it async".
- **GC pauses:** Server vs Workstation GC, gen2 blocking collections, LOH allocations (>=85KB objects) causing fragmentation; containers: memory limits affect heap hard limit (`GCHeapHardLimit`), `GCConserveMemory`. Metrics: `% time in GC`, gen2 collection count, allocation rate, pause durations (`GC.GetGCMemoryInfo()`).
- Containers: OOMKilled (exit 137) is a cgroup limit - not necessarily managed leak.

```
CPU high? -> dotnet-trace/profiler -> hot method?
                \-> GC % high? -> allocation rate/LOH -> reduce allocations, pooling
Memory ramp? -> counters: gc-heap-size vs working-set
   gc-heap grows  -> managed leak -> gcdump diff -> retention path -> fix root
   only working-set grows -> native/unmanaged/fragmentation/ GC settings
Latency w/ low CPU? -> threadpool queue length + stacks (dotnet-stack/dump) -> blocked on .Result?
```

### 5.1 [Easy] The app's CPU is pegged at 100% in production. What do you do?
**What the interviewer is really testing:** Mitigate, then profile with proper tools.
**Typical mid-level answer:** "Scale up and look at the code."
**Strong senior answer:** Stabilise: scale out or take a degraded instance out of rotation if the load balancer allows, after capturing evidence. Diagnose: first compare request rate to CPU - did traffic rise or did cost per request rise (recent deploy)? Then take a CPU trace: App Insights Profiler, or `dotnet-trace collect -p <pid> --duration 00:00:30` and open the nettrace in PerfView/speedscope to see hot call stacks. Typical culprits: regex catastrophic backtracking, serialization of huge objects, an infinite or tight retry loop, excessive logging, LINQ over big collections, or GC churn (check `% time in GC`). Fix the hotspot, add a regex timeout or caching as relevant, and add a CPU/saturation alert plus a perf test for the path to prevent regression.
```
dotnet-counters monitor -p <pid> System.Runtime --counters cpu-usage,gc-heap-size,alloc-rate,threadpool-queue-length
dotnet-trace collect -p <pid> --duration 00:00:30
```
**Follow-up traps:** "Profiler adds overhead in prod?" (sampling traces are low overhead; avoid heavy instrumentation). "Can't install tools on App Service?" (use Diagnose & solve / Kudu profiler, or remote-collect with diagnostics port).

### 5.2 [Follow-up] Memory keeps growing until the container is OOMKilled. How do you find the leak?
**What the interviewer is really testing:** Memory diagnostics method.
**Typical mid-level answer:** "Increase the memory limit."
**Strong senior answer:** Raise the limit only as a stopgap and set a restart schedule so users aren't impacted. Then determine whether the growth is managed or native: compare `gc-heap-size` with `working-set`. If managed heap grows (and gen2/LOH are growing), I capture a `dotnet-gcdump` at two points and diff types and counts - a growing type such as a `byte[]`, `string`, or a custom entity points to the leak; I then use `dotnet-dump` + `dumpheap -stat` and `gcroot` to find who's holding the reference (static dictionary, event handler, unbounded `MemoryCache`, a long-lived singleton capturing scoped objects). If only working set grows, suspect native memory (e.g. unmanaged libraries, image processing), fragmentation, or the GC not returning memory under a container limit. Fix the root retention, add bounded caches (SizeLimit/expiry), and add a memory-growth alert with a soak test.
```
dotnet-gcdump collect -p <pid> -o before.gcdump
# ...10 minutes later
dotnet-gcdump collect -p <pid> -o after.gcdump
dotnet-dump collect -p <pid>
> dumpheap -stat ; dumpheap -mt <MT> -min 85000 ; gcroot <addr>
```
**Follow-up traps:** "Memory increases but plateaus - leak?" (could be normal GC budget/caches; check after gen2 GCs). "Server GC with many cores?" (larger heaps expected; tune with limits).

### 5.3 [Medium] Explain thread pool starvation: symptoms, confirmation, fix.
**What the interviewer is really testing:** Practical async knowledge.
**Typical mid-level answer:** "Too many threads - increase pool size."
**Strong senior answer:** Starvation occurs when pool threads are blocked (e.g. `.Result`, `.Wait()`, `Task.Run` waiting synchronously, sync I/O) so new work items queue while the pool injects threads slowly. Symptoms: latency and timeouts rising with low CPU, queue length growing, timeouts on unrelated endpoints and health checks. Confirm with `dotnet-counters` (ThreadPool Queue Length, Thread Count), and stacks: `dotnet-stack report` or a dump with `clrstack` across threads showing many in `Monitor.Wait`/`GetResult`. Immediate mitigation: restart/scale out and possibly raise `ThreadPool.SetMinThreads` as a temporary relief. Real fix: remove sync-over-async, use async APIs end to end, add timeouts and cancellation tokens, limit concurrency to slow dependencies (SemaphoreSlim/bulkhead) so a slow dependency can't consume all threads.
**Follow-up traps:** "ConfigureAwait(false) fixes it?" (in ASP.NET Core there's no sync context; blocking is still the problem). "Why does it start suddenly?" (a downstream slowdown makes blocked calls last longer, tipping the pool).

### 5.4 [Hard] GC pauses are causing latency spikes. How do you analyse and reduce them?
**What the interviewer is really testing:** GC mechanics knowledge.
**Typical mid-level answer:** "Call GC.Collect() periodically." (harmful)
**Strong senior answer:** I measure rather than assume: `% time in GC`, gen2 collection count, pause durations (`GC.GetTotalPauseDuration()` in .NET 7+, or EventPipe GC events), allocation rate and LOH size. Frequent gen2 or LOH collections signal large-object churn (big `byte[]`, string concatenation, large lists, JSON of large payloads). Fixes: reduce allocations (`ArrayPool<T>`, `Span`, streaming serialization, avoid materialising large collections), reuse buffers/`RecyclableMemoryStream`, avoid LOH churn, choose Server vs Workstation GC and concurrent GC appropriately, set container memory limits realistically so the GC isn't constantly under memory pressure. Never sprinkle `GC.Collect()`. Validate by before/after load tests with the same counters.
**Follow-up traps:** "Server vs Workstation?" (Server: throughput, multiple heaps; Workstation: lower footprint; test per workload). "Container with 1 core?" (Server GC provides little; check config).

### 5.5 [Tricky] After adding an `IMemoryCache`, memory exploded. Why, and what's the fix?
**What the interviewer is really testing:** Cache design pitfalls.
**Typical mid-level answer:** "Too much data was cached."
**Strong senior answer:** `MemoryCache` has no default size limit - without `SizeLimit` and per-entry `Size` it grows until memory pressure (it does react to GC pressure, but late and unpredictably). Unbounded key cardinality (caching per user/query string/long-tail keys) means near-zero hit rate and unlimited growth. Also, entries holding references to large graphs (e.g. EF entities with change tracker) retain far more than expected, and sliding expirations without absolute expiration keep hot entries forever. Fix: set `SizeLimit`, assign sizes, use absolute expirations, cache DTOs not entities, restrict cacheable key space, track hit ratio and eviction metrics, and consider distributed cache for large/shared data.
**Follow-up traps:** "How to prove the leak is the cache?" (gcdump shows entries retained by MemoryCache's entries dictionary). "Cache stampede on expiry?" (topic 9).

### 5.6 [Scenario] A single pod's memory jumps on each large export request and the pod is OOMKilled. Walk through it.
**What the interviewer is really testing:** Peak memory vs leak distinction.
**Typical mid-level answer:** "Increase memory."
**Strong senior answer:** Memory that spikes with specific requests and recovers (or kills the pod) suggests peak allocation, not a leak: the export materialises the entire dataset (`ToList`, big `MemoryStream`, string building, loading entities then serialising). Confirm by correlating the pod's memory with the export request in telemetry and capturing a gcdump during the request. Fix: stream results (`IAsyncEnumerable`, `AsNoTracking`, write directly to response/Blob stream in chunks), paginate, move heavy exports to a background job with its own resources, and cap concurrency of exports. Mitigate immediately: limit export size/concurrency via flag, route exports to a larger worker. Prevent: memory-based load test on large tenants; alert on OOMKills and restarts.
**Follow-up traps:** "Increase limit?" (hides the issue, hurts bin-packing, may still fail with larger data). "Many concurrent exports?" (bulkhead/queue them).

**Red flags in answers**
- Calling `GC.Collect()` or just raising memory/instances as the fix.
- Blaming "the cloud" without capturing counters, dumps, or traces.
- Confusing low-CPU latency with a CPU problem (missing thread pool starvation).

**Cheat sheet**
- CPU: dotnet-trace/Profiler -> hot stack; check GC %.
- Memory: gc-heap-size vs working-set; two gcdumps diff; `gcroot`.
- Starvation: low CPU + queue length up + blocked stacks; async all the way.
- GC: reduce allocations & LOH churn; measure pause and gen2 count.
- Containers: OOMKilled (137) = cgroup limit; set GC limits sensibly.
- Bound every cache and queue.

---

## Topic 6: Intermittent 500s / Timeouts Under Load

**Study notes**
- Intermittent + load-correlated = **saturation or contention** until proven otherwise. Intermittent + instance-specific = bad node/state. Intermittent + periodic = scheduled job/TTL/GC/autoscale.
- Classic causes: SNAT port exhaustion (Azure outbound connections from App Service/AKS NAT), `HttpClient` created per request, SQL connection pool exhaustion (`Timeout expired... max pool size reached`), thread pool starvation, downstream throttling (429/503, SQL 40501/10928/10929), deadlocks (error 1205), transient network faults, health-probe-induced restarts, autoscale cold instances, request timeout mismatches (client 30s, gateway 230s on App Service / 60s on App Gateway defaults), keep-alive/idle-timeout mismatches (Azure LB idle 4 min -> connection resets).
- Evidence: exception **type & message** clusters (group by `outerMessage`/`type`), correlate with time/instance/load, `resultCode` distribution (500 vs 502/503/504 tells where it failed: app vs gateway).
- Resilience patterns: timeouts everywhere, retries with exponential backoff + jitter **only for idempotent/transient**, circuit breaker, bulkhead, load shedding, `HttpClientFactory`/`SocketsHttpHandler` pooled connections (`PooledConnectionLifetime`), SQL retry-on-failure (`EnableRetryOnFailure`).

```
500/timeouts intermittent
 |- group exceptions by type/message (KQL)       -> dominant signature?
 |- vs load (RPS) & per-instance                 -> saturation or bad node?
 |- 502/504 from gateway vs 500 from app         -> where does it fail?
 |- dependency failures/duration spikes          -> downstream throttling/timeouts
 |- pools: SQL connections, SNAT ports, threads  -> exhaustion
```

### 6.1 [Easy] ~2% of requests return 500, randomly. Where do you start?
**What the interviewer is really testing:** Grouping errors before theorising.
**Typical mid-level answer:** "Look at a single exception in the logs." Falls short: one sample may be unrepresentative.
**Strong senior answer:** I aggregate: group failed requests/exceptions by exception type, message pattern, operation, instance, and time; a 2% rate usually has a dominant signature. Then I compare against load and deploy time. I look for the first occurrence time to correlate with changes. I pick a few exemplar traces from each cluster to read the stack and the dependency spans. Only then do I form hypotheses (e.g. SQL timeouts cluster at peak minutes -> pool or DB saturation; `TaskCanceledException` against one HTTP dependency -> its timeouts).
```kusto
exceptions
| where timestamp > ago(2h)
| summarize n=count() by type, outerMessage=substring(outerMessage,0,80), cloud_RoleInstance
| top 20 by n desc
```
**Follow-up traps:** "Exceptions not recorded because they're caught?" (log at the catch site with correlation). "Sampling?" (aggregate with `itemCount`).

### 6.2 [Follow-up] You see "Timeout expired. The timeout period elapsed prior to obtaining a connection from the pool." What's happening and what do you do?
**What the interviewer is really testing:** Connection pool exhaustion understanding.
**Typical mid-level answer:** "Increase Max Pool Size." Falls short: treats the symptom, can overload the DB.
**Strong senior answer:** The pool (default max 100 per unique connection string per process) is exhausted: either connections are held too long (slow queries, long transactions, a slow dependency called while holding a connection), leaked (not disposed - missing `using`, returning an open `IQueryable`/reader), or demand exceeds capacity (bursty parallelism, many instances x pool size > DB limit). Diagnose: SQL `sys.dm_exec_sessions`/`sys.dm_exec_connections` grouped by host/program, `sys.dm_exec_requests` for long runners and blocking; app-side counters for pool usage (`Microsoft.Data.SqlClient` event counters: `active-hard-connections`, `number-of-pooled-connections`). Mitigate: restart leaking instance, kill head blockers, scale out carefully (more instances means more connections). Fix: dispose correctly, shorten transactions, no remote calls inside a DB transaction, async I/O, and size pool x instances below DB worker limits; add an alert on pool wait/time.
**Follow-up traps:** "Pool per connection string variance?" (different strings = different pools; app name differences fragment). "Does async fix it?" (reduces threads but not held connections).

### 6.3 [Medium] Timeouts appear only when outbound calls to a third-party API surge on App Service. What do you suspect?
**What the interviewer is really testing:** SNAT/port exhaustion and HttpClient hygiene.
**Typical mid-level answer:** "The third party is slow."
**Strong senior answer:** Possible SNAT port exhaustion: each outbound connection to the same destination IP:port uses a SNAT port and App Service gives limited ports per instance (default 128 preallocated, dynamic after); creating a new `HttpClient` per request leaves sockets in TIME_WAIT and burns ports. Evidence: App Service "SNAT Port Exhaustion" detector in Diagnose and solve problems, `TCP Connections` metrics, intermittent connect timeouts with low CPU. Fix: single shared `HttpClient`/`IHttpClientFactory`, connection reuse (keep-alive), `PooledConnectionLifetime` for DNS refresh, use private endpoints/service endpoints for Azure dependencies, or add NAT Gateway / more outbound IPs for external calls. Also add per-call timeouts and a circuit breaker so slow vendors don't hold connections.
**Follow-up traps:** "Singleton HttpClient and DNS changes?" (set PooledConnectionLifetime). "Can retries make it worse?" (yes, retry storms).

### 6.4 [Hard] Retries were added to fix the timeouts and the outage got worse. Explain.
**What the interviewer is really testing:** Retry amplification.
**Typical mid-level answer:** "Retries should have helped."
**Strong senior answer:** Retries multiply load on a struggling dependency: with 3 retries per layer and three layers, one failure becomes up to 27 calls (retry amplification), and synchronised retries create thundering herds. Retries also keep threads/connections held longer, causing pool exhaustion upstream, and retrying non-idempotent operations creates duplicates. Safe design: retry only transient errors and idempotent operations, exponential backoff with jitter, retry budgets (cap total retries as a % of traffic), a single retry layer, per-attempt and overall deadlines, circuit breaker to fail fast, and idempotency keys for writes. Respect `Retry-After` on 429/503.
**Follow-up traps:** "Polly vs built-in?" (either; Microsoft.Extensions.Http.Resilience in .NET 8). "Retry on 500?" (only if the operation is idempotent and error is transient).

### 6.5 [Tricky] Errors happen on one of five instances only. What do you check?
**What the interviewer is really testing:** Instance-level diagnostics.
**Typical mid-level answer:** "Restart that instance."
**Strong senior answer:** Restart restores service but loses evidence, so first I pull the instance out of rotation or capture dumps/logs. Hypotheses: poisoned in-memory state (corrupt cache, stuck circuit breaker), a failed warm-up/config load (secret/Key Vault fetch failed at startup), disk full or temp storage exhausted, noisy neighbour on the host, a stuck deployment (partial rollout, old file lock), DNS or network issue specific to the node, or a leaked-thread/deadlocked process. I compare that instance's metrics with peers (CPU, memory, threads, handles, disk), its startup logs and `cloud_RoleInstance` exception profile. I'd add health checks that detect it and auto-heal rules (e.g. App Service Auto-Heal, k8s liveness probes) while still alerting.
**Follow-up traps:** "Liveness probe too aggressive?" (can restart-loop under load; use readiness vs liveness correctly). "Sticky sessions?" (affinity may hide the problem).

### 6.6 [Scenario] The load test is fine but real production traffic at 10 AM gives 504s. What's different?
**What the interviewer is really testing:** Gap between test and prod.
**Typical mid-level answer:** "Load test wasn't realistic."
**Strong senior answer:** 504s come from the gateway (Front Door/App Gateway/APIM) waiting on origin beyond timeout, so I check the gateway logs to find origin response time and which backend. Differences from test: real data skew (big tenants), realistic request mix and payload sizes, cache cold vs warm, auth/token refresh bursts at start of day (everybody logging in at 10), batch jobs scheduled at 10 sharing the database, third-party dependencies throttling, and slow clients. I correlate 10 AM with cron schedules (Hangfire, Azure Functions timers, SQL Agent) and login spikes. Mitigation: pause/stagger the batch jobs, scale out ahead of the spike (scheduled autoscale), enable caching. Prevent by modelling production traffic shape in load tests (replayed or synthetic with the same mix) and scheduling heavy jobs off-peak.
**Follow-up traps:** "How to detect thundering herd at token expiry?" (look at request bursts aligned to expiry). "504 vs 502?" (timeout vs invalid response/connection failure).

**Red flags in answers**
- Adding retries or larger pools without considering amplification and downstream limits.
- Treating one exception sample as the whole truth.
- Restarting instances with no evidence capture or root cause.

**Cheat sheet**
- Group errors by signature, instance, time, load.
- Pool exhaustion = held too long / leaked / too much demand; fix holders first.
- SNAT exhaustion: shared HttpClient, private endpoints, NAT gateway.
- Retries: backoff+jitter, idempotent only, single layer, budgets, breaker.
- 502/504 = gateway view; check origin timing.
- Compare test traffic vs prod shape (data skew, schedules, bursts).

---

## Topic 7: Database Incidents (SQL Server / Azure SQL)

**Study notes**
- **Blocking:** one session holds locks, others wait. Find head blocker: `sys.dm_exec_requests` (`blocking_session_id`), `sys.dm_tran_locks`, `sys.dm_os_waiting_tasks`; `sp_whoisactive` if available. Causes: long transactions, missing indexes causing scans/lock escalation, user interaction inside transactions. Mitigations: kill head blocker (after assessing), RCSI (`READ_COMMITTED_SNAPSHOT`) to reduce reader/writer blocking, shorter transactions, proper indexes.
- **Deadlocks:** cycle of lock waits; victim gets error 1205. Capture via `system_health` extended event / Azure SQL deadlock graphs (Query Store doesn't, use XE). Fixes: consistent access order, indexes, shorter transactions, retry logic on 1205, RCSI, avoid read-then-write upgrade patterns (UPDLOCK hint).
- **Parameter sniffing / plan regression:** plan compiled for atypical parameter is reused. Evidence: Query Store shows plan change with duration jump. Remedies: force last good plan (`sp_query_store_force_plan`), `OPTION (RECOMPILE)` / `OPTIMIZE FOR`, update stats, Parameter Sensitive Plan optimisation (compat level 160), better indexing. Statistic updates or restarts/failovers clear plan cache and can trigger it.
- **Log full / log growth:** `log_reuse_wait_desc` in `sys.databases` (ACTIVE_TRANSACTION, LOG_BACKUP, AVAILABILITY_REPLICA, REPLICATION). Azure SQL: log rate governance (`LOG_RATE_GOVERNOR` waits). Fix: end the long transaction, log backups (full/bulk-logged), batch big deletes/updates.
- **tempdb:** contention (PAGELATCH on allocation pages), spills from bad estimates, version store growth (long snapshot transactions), large sorts/hashes. Check `sys.dm_db_session_space_usage`, version store size.
- **Failover:** Azure SQL failover causes brief connection drops (seconds-30s); apps must retry transient errors (40613, 40197, 40501, 49918). Connection pools hold dead connections -> `ClearPool`/retry.
- **Resource limits:** DTU/vCore CPU 100%, workers exhausted (error 10928/10929), IO governance (`dm_db_resource_stats`, `sys.dm_user_db_resource_governance`).

```sql
-- Who is blocked and by whom
SELECT r.session_id, r.blocking_session_id, r.wait_type, r.wait_time, t.text
FROM sys.dm_exec_requests r CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
WHERE r.blocking_session_id <> 0;
-- Why is log not truncating
SELECT name, log_reuse_wait_desc FROM sys.databases;
-- Recent resource use (Azure SQL)
SELECT TOP 30 end_time, avg_cpu_percent, avg_data_io_percent, avg_log_write_percent
FROM sys.dm_db_resource_stats ORDER BY end_time DESC;
```

### 7.1 [Easy] Many requests hang and SQL shows a long chain of blocked sessions. What do you do?
**What the interviewer is really testing:** Head-blocker identification and safe resolution.
**Typical mid-level answer:** "Restart the database/app." Falls short: heavy-handed; the blocker returns.
**Strong senior answer:** Find the head blocker: the session with `blocking_session_id = 0` that others (transitively) wait on, using `sys.dm_exec_requests` and `sys.dm_tran_locks`; check what it's running, its open transaction (`sys.dm_tran_active_transactions`), duration, and login/host. If it's a forgotten transaction (an idle session with open tran) or a rogue ad-hoc query, kill it after weighing rollback time (a large transaction's rollback can take as long as it ran). Then determine the root cause: a long transaction in app code, escalation to table lock due to scans, a migration script, or an app that awaits an external call inside a transaction. Fix with shorter transactions, index to avoid scans, RCSI, and alerting on blocked-process threshold.
**Follow-up traps:** "Is KILL always safe?" (rollback may be long; monitor with `KILL <spid> WITH STATUSONLY`). "Restarting app?" (leaves the transaction open until connection drops).

### 7.2 [Follow-up] Deadlock errors (1205) spiked after release. How do you handle them?
**What the interviewer is really testing:** Reading deadlock graphs and structural fixes.
**Typical mid-level answer:** "Add retry." Partially right but incomplete.
**Strong senior answer:** Retry on 1205 is a necessary safety net but not the cure. I pull deadlock graphs from the `system_health` Extended Events session, identify the two statements and resources (key/page locks on which index), then see what the release changed: new access order across tables, a new query that scans and takes more locks, or a changed isolation level. Fixes: access objects in a consistent order, add/adjust indexes so queries seek and lock fewer rows, shorten transactions, avoid read-modify-write without `UPDLOCK`, enable RCSI to remove reader-writer deadlocks, and batch updates. Mitigate immediately by rolling back the change or serialising the hot path (queue/lock by key) if the storm is severe.
```sql
SELECT CAST(xed.event_data AS xml) AS deadlock
FROM (SELECT CAST(target_data AS xml) AS td FROM sys.dm_xe_session_targets t
      JOIN sys.dm_xe_sessions s ON s.address=t.event_session_address WHERE s.name='system_health') x
CROSS APPLY td.nodes('//RingBufferTarget/event[@name="xml_deadlock_report"]') AS q(xed);
```
(Azure SQL DB: use the `xml_deadlock_report` via the `system_health` ring buffer/event file or Query Performance Insight diagnostics as available.)
**Follow-up traps:** "Deadlock vs blocking?" (deadlock is a cycle resolved by victim kill; blocking is one-directional). "Retry with transactions?" (retry the whole unit of work; must be idempotent).

### 7.3 [Medium] A query that took 50ms now takes 30s, no deploy. What happened?
**What the interviewer is really testing:** Plan regression/statistics knowledge.
**Typical mid-level answer:** "Maybe the DB is overloaded."
**Strong senior answer:** Likely a plan change: statistics auto-updated, plan cache evicted by failover/restart/memory pressure, or a first-compile with an atypical parameter (parameter sniffing). Query Store shows the same `query_id` with multiple `plan_id`s and the duration jump at the time. Mitigate by forcing the previous good plan in Query Store (or `sp_recompile` / `OPTION (RECOMPILE)` as a quick unblock), updating statistics with a full scan if stats are skewed, then verify. Longer term: fix with better indexes, rewriting to avoid parameter-sensitive predicates, PSP optimisation (compat 160+), or `OPTIMIZE FOR UNKNOWN` with care; monitor forced plans. Also check data growth: maybe the table crossed a threshold making the old plan's assumption wrong.
```sql
EXEC sp_query_store_force_plan @query_id = 123, @plan_id = 456;
-- check later
SELECT * FROM sys.query_store_plan WHERE is_forced_plan = 1;
```
**Follow-up traps:** "Forced plan fails after schema change?" (`force_failure_count`). "Clearing the entire plan cache?" (dangerous - recompile storm).

### 7.4 [Hard] The transaction log is full and writes fail. Walk through response.
**What the interviewer is really testing:** Knowing why logs don't truncate.
**Typical mid-level answer:** "Shrink the log file."
**Strong senior answer:** Writes fail (error 9002) because log space can't be reused or grown. First find why: `log_reuse_wait_desc` - ACTIVE_TRANSACTION (long-running transaction), LOG_BACKUP (log backups not running in full recovery), AVAILABILITY_REPLICA (secondary lagging), REPLICATION. Mitigate: for an active transaction, assess and end it (rollback needs log space too); for LOG_BACKUP take a log backup; if disk-bound, add space/raise the max size. Avoid shrinking as routine (it fragments and doesn't fix cause) and never switch recovery model blindly because it breaks the backup chain. In Azure SQL, the log is managed, but long transactions/log rate governance still bite; scale up temporarily if governed. Prevent: batch large operations in chunks, monitor log used %, alert on long-running transactions and backup/replica lag.
**Follow-up traps:** "Why can rollback make it worse?" (needs log). "Bulk delete of millions of rows?" (batch with small transactions; consider partition switching).

### 7.5 [Tricky] After an Azure SQL failover the app stayed broken for 20 minutes though the DB was fine. Why?
**What the interviewer is really testing:** Client resilience & pooled connections.
**Typical mid-level answer:** "Azure SQL had an outage."
**Strong senior answer:** Failover is expected and brief; prolonged app errors implicate the client: no retry logic for transient errors (40613/40197/40501/10928/10929 etc.), pools holding broken connections that are not validated, threads blocked on long default command timeouts, or a circuit breaker stuck open or configured too long, or a DNS cache/pinned IP. Fixes: `EnableRetryOnFailure` in EF Core or SqlClient configurable retry logic, sensible connect and command timeouts, circuit breaker with half-open probing, health checks that revalidate DB connectivity, and testing failover with a game day (Azure SQL supports manual failover via portal/PowerShell/CLI).
**Follow-up traps:** "Redirect vs proxy connection policy?" (redirect requires ports 11000-11999 open). "Retry on any exception?" (only transient).

### 7.6 [Scenario] Production CPU on Azure SQL hits 100% at 9:05 every morning. How do you investigate and fix?
**What the interviewer is really testing:** Query Store and workload analysis.
**Typical mid-level answer:** "Scale up the database."
**Strong senior answer:** Scaling up is a valid first mitigation (cost-noted) while I analyse. The recurring time suggests a scheduled job or traffic pattern. I use Query Store (Top Resource Consumers by CPU within the time window) and `dm_db_resource_stats` to find which queries dominate, how many executions, and whether plans changed. Check for batch jobs (reports, ETL, index maintenance, stats updates) starting at 9:05, plus app-level cache expiries synchronised on the hour. Fixes: tune top queries (indexes, rewrite), move reporting to a readable secondary/replica, stagger or throttle jobs, add caching, and schedule maintenance off-peak with resumable index operations. Then add alerts on CPU > threshold with context and a regression test for the offender.
**Follow-up traps:** "Read scale-out?" (Premium/BC/Hyperscale replicas via `ApplicationIntent=ReadOnly`). "Auto-tuning?" (automatic plan correction is available; review its actions).

**Red flags in answers**
- Shrinking logs, clearing the entire plan cache, or killing sessions without understanding consequences.
- Fixing deadlocks only by retry.
- Treating failover as a prolonged DB outage instead of a client-resilience gap.

**Cheat sheet**
- Blocking: find head blocker; fix long transactions/indexes; consider RCSI.
- Deadlock: graph from system_health; consistent order + indexes + retry.
- Plan regression: Query Store compare plans; force good plan; fix root.
- Log full: check `log_reuse_wait_desc`.
- Failover: transient retry + validated pools + sane timeouts.
- Azure SQL: `dm_db_resource_stats`, Query Store, Intelligent Insights.

---

## Topic 8: Messaging Incidents (Service Bus / Queues)

**Study notes**
- Service Bus semantics: **PeekLock** (default; at-least-once) vs ReceiveAndDelete (at-most-once). Message locked for `LockDuration` (default 1 min, max 5); if not completed in time the lock is lost -> redelivery (`DeliveryCount++`). After `MaxDeliveryCount` (default 10) -> dead-letter queue (DLQ). Sessions provide ordering/FIFO per session id. Duplicate detection (time window) uses `MessageId`.
- **Backlog:** consumers slower than producers: check active message count, oldest message age, consumer count/concurrency, processing duration, downstream slowness, throttling (Standard vs Premium), prefetch settings.
- **DLQ growth:** poison messages (schema mismatch, null refs, validation), transient failures exhausting delivery count, expired TTL, explicit dead-lettering. Inspect `DeadLetterReason`/`DeadLetterErrorDescription`.
- **Duplicates:** at-least-once => consumers must be **idempotent** (inbox/dedup table with unique key on MessageId, idempotent upserts, conditional updates). Duplicates arise from lock loss, crashes after side-effect before complete, producer retries.
- **Poison message loop cost:** a message that always fails burns compute/ DB / dependency calls up to MaxDeliveryCount; pair with backoff, DLQ, alert on DLQ > 0.
- Patterns: outbox (atomic DB write + publish), competing consumers, scheduled retry (`ScheduledEnqueueTime`), DLQ reprocessor tooling with safeguards.

```
Backlog growing?
 |- consumers running/healthy? (scale, crashes, auth)
 |- processing time per msg up? -> dependency slow / DB / lock loss
 |- inflow spike? -> autoscale on queue length (KEDA / Functions)
 |- poison loops? -> DeliveryCount high, same MessageId repeating
DLQ growing? -> read reason; fix cause; replay safely (idempotent)
```

### 8.1 [Easy] The queue depth is climbing steadily. What do you check?
**What the interviewer is really testing:** Throughput vs inflow reasoning.
**Typical mid-level answer:** "Add more consumers."
**Strong senior answer:** Backlog = inflow > drain rate. I check inflow (producer spike or a loop republishing?), consumers (alive, scaled, authenticating, not crash-looping), per-message processing time (a downstream slowed?), lock-lost/abandon rates, and concurrency settings (`MaxConcurrentCalls`, prefetch). Then I decide: scale out consumers if the downstream can take it (otherwise I just move the bottleneck), throttle producers, or fix the slow dependency. Oldest-message age is the user-impact signal, so I alert on that rather than raw count. If the backlog is time-critical, I prioritise (separate queues/topic filters) so important messages aren't stuck behind bulk work.
**Follow-up traps:** "Scale consumers x10 and DB melts?" (backpressure, concurrency limits). "Ordering required?" (sessions limit parallelism per session).

### 8.2 [Follow-up] Messages are processed twice, causing duplicate side-effects. Why and how do you fix it?
**What the interviewer is really testing:** Idempotency design.
**Typical mid-level answer:** "Turn on duplicate detection."
**Strong senior answer:** With PeekLock delivery is at-least-once: lock expiry because processing exceeded `LockDuration`, a crash after the side effect but before `CompleteMessageAsync`, or producer retry can all produce duplicates. Service Bus duplicate detection only covers producer-side duplicates within a window by `MessageId`, not redelivery to consumers. The real fix is idempotent consumers: record processed MessageIds (or business keys) in a table with a unique constraint written in the **same transaction** as the side effect, use idempotent upserts / conditional updates, and pass idempotency keys to external APIs (e.g. payment provider). Also renew locks for long work (auto lock renewal) or break up the work, and set LockDuration beyond p99 processing time. Diagnose by checking `DeliveryCount` and lock-lost exceptions (`MessageLockLostException`).
**Follow-up traps:** "Exactly-once possible?" (effectively-once via idempotency + transactions). "Outbox?" (for producer-side atomicity of DB change and message).

### 8.3 [Medium] The DLQ has 40,000 messages. What do you do?
**What the interviewer is really testing:** Methodical triage and safe replay.
**Typical mid-level answer:** "Resubmit them all."
**Strong senior answer:** First stop new bleed: understand why they dead-lettered by sampling and grouping by `DeadLetterReason` and error description - deserialization failures after a schema change, validation errors, a transient outage that exhausted the delivery count, or TTL expiry. Fix the cause *before* replay (otherwise they dead-letter again). Then replay in a controlled way: copy to the main queue (or a repair queue) in batches with rate limiting, idempotent consumers, monitoring, and preserving original ids/metadata. Some messages may need transformation or manual disposition (discard with audit). Check business impact: do customers have orders stuck? Provide a list. Prevent: alert on DLQ > 0, schema versioning with backward compatibility, and a documented DLQ-handling runbook/tool.
**Follow-up traps:** "Messages already compensated manually?" (replay could double-process; check state). "Order of replay?" (may matter; sessions/timestamps).

### 8.4 [Hard] One poison message is crashing consumers and blocking the queue/session. How do you handle it?
**What the interviewer is really testing:** Poison-message design.
**Typical mid-level answer:** "Delete the message."
**Strong senior answer:** Under PeekLock the message returns and is retried until `MaxDeliveryCount` then dead-letters; if the consumer *crashes* the process (unhandled exception in a background thread, OOM from the payload), the retry loop can take whole instances down and the delivery-count race blocks others. For session queues, the session is blocked until it moves on. Mitigation: lower MaxDeliveryCount temporarily or manually dead-letter the message by sequence number, inspect and keep the payload for analysis. Fix: catch exceptions per message, classify as transient vs permanent, dead-letter permanent failures immediately with reason, bound payload size, and use exponential backoff via scheduled redelivery for transient ones. Add alerting on high delivery counts and crash-loops. Never delete without recording what was lost.
**Follow-up traps:** "Azure Functions trigger?" (host retries then DLQs; check `maxDeliveryCount`/host.json `maxAutoLockRenewalDuration`). "Session blocked?" (the session lock; abandon/dead-letter message).

### 8.5 [Tricky] "MessageLockLostException" appears under load. Why does it occur and what's the fix?
**What the interviewer is really testing:** Lock mechanics.
**Typical mid-level answer:** "Network flakiness."
**Strong senior answer:** The lock expired before completion: processing slower than `LockDuration`, concurrency/prefetch too high so locks tick while messages wait in the client's prefetch buffer, or the host was CPU/thread-starved so renewals didn't run. Result: message redelivered while the first handler is still working -> duplicates, and `Complete` fails. Fixes: reduce prefetch/concurrency to what the process can handle, enable auto lock renewal (`MaxAutoLockRenewalDuration`), increase LockDuration modestly, speed up handlers or split long work into steps, ensure idempotency, and fix thread starvation. Monitor lock-lost counts and delivery counts.
**Follow-up traps:** "Max lock duration?" (5 minutes per lock; renew for more). "Why prefetch hurts?" (locks start at prefetch time).

### 8.6 [Scenario] After a deploy, queue consumers silently stopped processing, queue grows, no errors. Walk through it.
**What the interviewer is really testing:** Silent failure diagnosis.
**Typical mid-level answer:** "Check logs for exceptions."
**Strong senior answer:** Silent = no errors logged: consumer not receiving. Hypotheses: processor never started (startup exception swallowed, host.json/trigger binding changed), wrong queue/subscription name or connection setting after the config change, managed identity lost `Azure Service Bus Data Receiver` role, subscription rule/filter changed, consumer disabled flag, or app scaled to zero with the scaler misconfigured. Check queue metrics (active messages, consumer/ incoming vs outgoing), Activity log for role/config changes, app startup logs, function disabled setting (`AzureWebJobs.<name>.Disabled`), and trace for `Microsoft.Azure.ServiceBus`/`Azure.Messaging.ServiceBus` errors. Mitigate by rolling back the config/deploy. Prevent: alert on oldest-message-age and "no messages processed" heartbeat metrics plus a post-deploy smoke test that sends a test message end to end.
**Follow-up traps:** "How to detect absence of events?" (alerts on missing data/`count == 0` with expected traffic). "Why no logs?" (nothing was running to log).

**Red flags in answers**
- Assuming exactly-once delivery.
- Replaying a DLQ blindly without fixing cause or idempotency.
- Scaling consumers without considering downstream capacity.

**Cheat sheet**
- PeekLock = at-least-once -> idempotent consumers (inbox table/unique key).
- Alert on oldest message age and DLQ > 0, not just depth.
- Lock lost: reduce prefetch/concurrency, auto-renew, speed up handler.
- Fix cause before DLQ replay; replay rate-limited.
- Classify failures transient vs permanent; DLQ permanent ones.
- Outbox for atomic DB-change + publish.

---

## Topic 9: Cache Incidents (Redis)

**Study notes**
- Azure Cache for Redis: single-threaded command execution per shard; big keys and slow commands (`KEYS`, `SMEMBERS` on huge sets, large `HGETALL`) block everyone. Metrics: server load, CPU, memory fragmentation, evicted keys, connected clients, cache hit/miss, network bandwidth, operations/sec, timeouts.
- **Stampede (thundering herd):** a hot key expires and many requests recompute concurrently -> DB overload. Mitigations: request coalescing/single-flight (lock per key), TTL jitter, stale-while-revalidate, background refresh, pre-warming, `HybridCache` (.NET 9) with stampede protection.
- **Eviction:** `maxmemory-policy` (volatile-lru, allkeys-lru, noeviction); evictions cause misses -> DB load. Check memory use and key sizes.
- **Stale data:** missing invalidation, race between write and cache set, TTL too long, multi-layer caches (in-process + Redis + CDN). Patterns: cache-aside, write-through, event-based invalidation, versioned keys.
- **Redis down:** the cache is an optimisation, not a dependency: app must degrade gracefully (timeouts short, circuit breaker, fallback to DB with load protection). StackExchange.Redis: shared `ConnectionMultiplexer` singleton, `abortConnect=false`, retry/reconnect policy, sensible `syncTimeout`/`asyncTimeout`. Failover/patching causes brief connection loss.
- Cache is not a source of truth; never store the only copy of data in it unless designed for it.

```
Cache miss storm -> DB CPU up -> latency up -> timeouts -> more retries -> collapse
Defences: single-flight, jitter TTL, stale-while-revalidate, circuit breaker + DB protection (bulkhead/rate-limit)
```

### 9.1 [Easy] What is a cache stampede and how do you prevent it?
**What the interviewer is really testing:** Core cache failure mode awareness.
**Typical mid-level answer:** "Increase cache TTL." Falls short: delays but doesn't prevent.
**Strong senior answer:** A stampede occurs when many callers miss the same key at once (expiry, eviction or cold start) and all hit the database to rebuild it, potentially collapsing the DB. Prevention: single-flight per key (only one caller recomputes; others wait or get stale data), randomised TTL jitter so keys don't expire together, stale-while-revalidate/background refresh, pre-warming after deploy or restart, and protecting the DB with concurrency limits. In .NET, `HybridCache` offers built-in stampede protection; otherwise use a keyed `SemaphoreSlim` or Lazy/async lock for in-process, and a Redis lock for distributed with care for lock expiry.
**Follow-up traps:** "Distributed lock correctness?" (use short expiry, tokens; accept occasional duplicates). "Negative caching?" (cache 'not found' briefly to avoid penetration).

### 9.2 [Follow-up] Redis went down for 10 minutes and the whole app went down with it. What should the design have been?
**What the interviewer is really testing:** Graceful degradation.
**Typical mid-level answer:** "Make Redis highly available." Necessary but incomplete.
**Strong senior answer:** HA reduces probability; the app must still tolerate failure. Cache calls need short timeouts (hundreds of ms), a circuit breaker so we stop trying a dead cache (avoiding thread pile-up on timeouts), and a fallback to the source of truth with DB protection (concurrency limit, rate limit, load shedding) because suddenly every request becomes a DB read. Non-critical features should degrade (e.g. skip personalisation). Configure StackExchange.Redis with `abortConnect=false`, a singleton multiplexer and reconnect logic. Test it: chaos/game day that kills Redis, verify DB headroom for 100% miss rate. Capacity-plan the DB for cache-cold operation or define degraded-mode behaviour.
**Follow-up traps:** "Using Redis for sessions/locks?" (then it's a dependency; design accordingly). "Cache warm-up after recovery?" (staggered to avoid stampede).

### 9.3 [Medium] Users see stale prices after an update. How do you investigate?
**What the interviewer is really testing:** Cache invalidation reasoning.
**Typical mid-level answer:** "Flush the cache."
**Strong senior answer:** Flushing is the quick mitigation (targeted key deletion, not FLUSHALL, which causes a stampede) but doesn't explain it. Possible causes: invalidation missing on one write path (admin tool or batch import bypassing the service), a race (read old value -> write DB -> invalidate -> reader sets stale value after invalidation), TTL too long, multiple layers (in-process MemoryCache per instance not invalidated across instances, CDN caching API responses), replicas lagging, or event-based invalidation messages lost. I trace the key lifecycle: where it's set/deleted, TTLs, and layers. Fixes: invalidate on every write path via one component, use versioned keys or short TTL + event-driven invalidation (with pub/sub for in-process caches), and set `Cache-Control` appropriately. Define staleness tolerance explicitly per data type.
**Follow-up traps:** "Write-through vs cache-aside race?" (set with version/ETag). "CDN?" (purge or surrogate keys).

### 9.4 [Hard] Redis server load is 100% and clients are timing out, but memory is fine. What do you check?
**What the interviewer is really testing:** Redis internals awareness.
**Typical mid-level answer:** "Scale up the cache."
**Strong senior answer:** Single-threaded command processing means one slow command blocks everyone. I check `SLOWLOG GET`, `INFO commandstats`, and big keys (`--bigkeys`, `MEMORY USAGE`), commands like `KEYS *`, large `SMEMBERS/HGETALL`, Lua scripts, or huge payloads saturating bandwidth. Also connection counts and client-side issues: thread pool starvation looks like Redis timeouts (StackExchange.Redis timeouts include thread pool and client stats in the message), network bandwidth limits of the SKU, and CPU/IO of the client VM. Fixes: replace O(N) commands with `SCAN`/smaller structures, break up big keys, use pipelining/batching sensibly, scale to a bigger SKU or clustering/shards, and use more appropriate data modelling. Distinguish server-side saturation from client-side starvation using the timeout message's `IOCP/WORKER` pool info.
**Follow-up traps:** "Timeouts but server idle?" (client thread starvation or network). "Clustering?" (multi-key ops require same hash slot).

### 9.5 [Tricky] After a deploy, hit ratio dropped to 10% and DB CPU doubled. Cause?
**What the interviewer is really testing:** Cache key discipline.
**Typical mid-level answer:** "Cache cleared on deploy."
**Strong senior answer:** Candidates: key format/versioning changed (new prefix -> everything cold), serialization format changed (old entries fail to deserialize and are treated as misses), TTL config changed, new unbounded key cardinality (including a timestamp or user-specific parameter in the key), or eviction due to larger values. Check hit/miss and evicted-keys metrics, sample keys, and the diff for key-building code. Mitigation: revert the key change or warm the cache gradually; protect the DB with single-flight. Prevention: key-versioning policy with dual-read migration, and a dashboard with hit ratio and DB load tied to deploy markers.
**Follow-up traps:** "Rolling deployment with mixed versions?" (old and new write incompatible formats - use versioned keys). "Warm-up on deploy?" (slot warm-up before swap).

### 9.6 [Scenario] Design the on-call response when Redis memory hits 95% and evictions begin during peak.
**What the interviewer is really testing:** Capacity-aware mitigation.
**Typical mid-level answer:** "Add memory."
**Strong senior answer:** Evictions raise misses and DB load, so I watch DB CPU and latency alongside the cache. Stabilise: scale the cache SKU or shard (note that scaling can cause brief connection disruption; plan for it) or lower memory by purging low-value/oversized keys (targeted), after confirming the eviction policy (`allkeys-lru` vs `volatile-*` only evicting TTL keys; `noeviction` would produce write errors). Diagnose which keys grew: big-key scan, new feature caching entire payloads, missing TTLs. Fix: set TTLs on all keys, compress or slim values, cap cardinality, separate caches by criticality (sessions vs content). Prevent: alerts at 75-80% memory and on eviction rate, with capacity planning tied to traffic growth.
**Follow-up traps:** "Fragmentation ratio >1.5?" (restart/defrag settings). "Persistence/patching impact?" (reconnect behaviour).

**Red flags in answers**
- Treating the cache as a source of truth or an always-available dependency.
- `FLUSHALL` in production as a first resort.
- No fallback or DB protection when the cache fails.

**Cheat sheet**
- Stampede: single-flight, TTL jitter, stale-while-revalidate, warm-up.
- Cache failure must degrade, with timeouts + breaker + DB protection.
- Stale: find the write path without invalidation; layered caches; versioned keys.
- Redis slowness: SLOWLOG, big keys, O(N) commands, client thread starvation.
- Watch hit ratio, evictions, server load, memory, DB load together.
- Plan DB for cold-cache scenario.

---

## Topic 10: Deployment-Caused Incidents & Expand-Contract Migrations

**Study notes**
- Most incidents are change-induced. Categories: bad code; **schema migration** (locking, long-running, incompatible with old/new version); **config drift** (prod differs from staging; app settings, connection strings, slot-sticky settings); **secret/cert expiry**; **slot swap** surprises (sticky vs swapped settings, warm-up failures, DB schema mismatch); **feature flag** misconfiguration; infra-as-code changes (NSG, DNS, SKU); dependency version bumps.
- **Expand-contract (parallel change):** make schema changes backward compatible across deploys.
  1. **Expand:** add new column/table (nullable or defaulted), deploy code that writes both / reads old.
  2. **Migrate:** backfill in batches; dual-write; verify.
  3. **Switch:** read from new.
  4. **Contract:** after all versions use new and rollback window passed, drop old.
  Never rename/drop in the same release as the code that stops using it. Rolling deploys and slot swaps mean old and new code run simultaneously against the same DB.
- Risky DDL on SQL Server: adding NOT NULL column with default (metadata-only in modern versions for constants), index creation (use `ONLINE=ON, RESUMABLE=ON` where edition supports), large `ALTER COLUMN` (size-of-data operation), FK additions (validation scan, `WITH NOCHECK` caveats), long locks `Sch-M` blocking all queries. Use `LOCK_TIMEOUT`, run in low traffic, batch backfills.
- Safe release: small changes, CI gates, deploy markers on dashboards, health-gated progressive rollout, automatic rollback, post-deploy smoke tests, and slot warm-up (`applicationInitialization`).

```
Release N   (expand):   schema +col (nullable), code writes old+new, reads old
Release N+1 (switch):   backfill done+verified, code reads new, still writes both
Release N+2 (contract): code stops writing old; later drop old column
Rollback possible at each step because adjacent versions are schema-compatible.
```

### 10.1 [Easy] A migration locks a big table in prod and the site hangs. What do you do now, and how do you prevent it?
**What the interviewer is really testing:** Migration safety awareness.
**Typical mid-level answer:** "Wait for it to finish." Falls short: long schema locks queue all other requests (lock convoy).
**Strong senior answer:** Immediately identify the DDL session (`sys.dm_exec_requests`, blocking chain) and decide whether to kill it: DDL rollback may also take time, but waiting with all traffic blocked is usually worse if completion is far. I'd also check whether it is blocked behind a long transaction and itself blocking everyone (Sch-M lock waiting makes new readers queue). After recovery, prevent: test migrations on a prod-sized copy, set `LOCK_TIMEOUT` so DDL fails fast rather than queueing, use online/resumable operations, batch data backfills, run off-peak, separate schema changes from code deploys, and use expand-contract so no step needs a big exclusive operation. Review migrations like code with a checklist.
**Follow-up traps:** "Adding a nullable column safe?" (metadata-only, but still needs brief Sch-M). "EF migrations auto-run at startup?" (risky with multiple instances; run as a controlled pipeline step).

### 10.2 [Follow-up] Explain expand-contract with a column rename.
**What the interviewer is really testing:** Backward-compatible design.
**Typical mid-level answer:** "Use sp_rename and deploy." Breaks old instances during rollout and prevents rollback.
**Strong senior answer:** Renaming `Phone` to `Mobile`: (1) Expand - add `Mobile` nullable; deploy code writing both columns and reading `Phone`(or coalesce). (2) Backfill `Mobile` from `Phone` in batches and verify counts/ checksums; keep triggers or dual-write for new data. (3) Switch reads to `Mobile`, still dual-writing so rollback remains safe. (4) Once no running version uses `Phone` and the rollback window passed, stop writing it and finally drop it in a later release. Each deploy is compatible with the previous one, so a rollback or a mixed fleet during rolling deployment never breaks. The cost is more releases and temporary duplication - worth it for zero-downtime and reversible changes.
**Follow-up traps:** "Dual-write consistency?" (do in one transaction; verify with reconciliation query). "Large backfill?" (small batches with `WAITFOR`, avoid log growth/blocking).

### 10.3 [Medium] After a slot swap, errors begin. What could differ between slots and how do you respond?
**What the interviewer is really testing:** App Service deployment mechanics.
**Typical mid-level answer:** "Staging worked, so swap again."
**Strong senior answer:** Swap again (swap back) is a valid first mitigation - it's fast. Then diagnose differences: slot-sticky vs swapped app settings/connection strings (settings marked "deployment slot setting" stay with the slot; others move), managed identity differences (slot has its own identity and RBAC/Key Vault access policy), missing warm-up so swapped instances were cold, different environment variables (e.g. `ASPNETCORE_ENVIRONMENT`), DB migrations applied to the shared DB but incompatible with the old code now back in production, and app settings changing triggers restarts. Use swap with preview (multi-phase) to validate with production settings before routing traffic. Add `applicationInitialization` warm-up paths and health checks. Maintain config parity via IaC, not portal edits.
**Follow-up traps:** "Swap back after migration?" (old code vs new schema). "Slot identity?" (grant staging slot identity access too).

### 10.4 [Hard] Production broke because a secret/certificate expired. Walk through the response and prevention.
**What the interviewer is really testing:** Time-bomb awareness.
**Typical mid-level answer:** "Renew it." Falls short on diagnosing and prevention.
**Strong senior answer:** Symptoms: sudden auth failures (401/403 to a dependency), TLS handshake failures, `AuthenticationException`, "The remote certificate is invalid", `AADSTS7000222` (expired client secret). Fast diagnosis: the failure starts at a precise timestamp matching an expiry; check dependency exceptions, Key Vault secret/cert `Expires` attribute, SP credential expiry (Entra ID app registration), and TLS certificate validity. Mitigate: issue/rotate the new secret or cert, update Key Vault, and refresh consumers (app may cache secrets until restart; Key Vault references in App Service update periodically, new versions picked up automatically only for versionless references - restart if needed). Prevent: prefer managed identities (no secrets), automated rotation/auto-renewed certs (App Service Managed Certificates / Key Vault with CA integration), Key Vault Event Grid `SecretNearExpiry`/`CertificateNearExpiry` events -> alerts at 30/14/7 days, an inventory of expiry dates, and synthetic checks that validate cert expiry. Document the rotation runbook and rehearse it.
```bash
az keyvault secret list --vault-name <kv> --query "[].{n:name, exp:attributes.expires}" -o table
az keyvault certificate list --vault-name <kv> --query "[].{n:name, exp:attributes.expires}" -o table
```
**Follow-up traps:** "Rotation causes downtime for dual-credential systems?" (overlap period with two valid credentials). "Who owns third-party certs?" (inventory with owner).

### 10.5 [Tricky] A feature flag was flipped on in prod and caused an outage. How do you think about flags and what do you change?
**What the interviewer is really testing:** Flags as change-management and risk.
**Typical mid-level answer:** "Flags are safe because we can turn them off." Partly true, but flags are production changes.
**Strong senior answer:** Flipping a flag is a deploy of behaviour, so it needs the same discipline: audit log, who/when, change visible on dashboards as annotation, gradual rollout (percentage/ring/tenant) and a kill switch tested for the "off" path. Mitigation is to flip it off (fastest rollback), but check whether turning it off is safe: data written under the new behaviour must still be readable. Problems to examine: flag combination explosion, stale flags with untested code paths, flags evaluated with inconsistent state across instances, dependency on flag service availability (default values when it's down - fail to safe default). Prevent: expiry/cleanup policy for flags, owners, testing both states in CI, and progressive exposure with automated metric checks (Azure App Configuration feature flags / targeting filters).
**Follow-up traps:** "Flag service outage?" (cached last known value/safe default). "Long-lived flags?" (tech debt; remove).

### 10.6 [Scenario] Release 5.4 deploys fine, health checks pass, but 20 minutes later orders fail for one payment method. Why didn't the pipeline catch it, and what do you change?
**What the interviewer is really testing:** Test and gate gaps.
**Typical mid-level answer:** "QA missed it."
**Strong senior answer:** Health checks that only confirm the process is up don't validate business paths; the issue appears for one payment method, perhaps due to a config value (provider credential or endpoint) missing in prod, or a data-dependent branch. Mitigate by rolling back or disabling that method via flag with a customer message. Diagnose via failures grouped by payment method and dependency, plus config diff between environments. Improvements: deep health check/smoke test for each payment method (with a sandbox account or test transactions), canary release with business-metric gating (payment success rate per method), config validation at startup (fail fast if required settings missing), and config managed in IaC. I'd also check why staging passed - parity or untested path - and add that test.
**Follow-up traps:** "Synthetic transactions in prod?" (use test accounts, isolate from reporting). "Why fail-fast at startup?" (surfaces misconfig in slot warm-up rather than live).

**Red flags in answers**
- Renaming or dropping columns in the same release as the code change.
- Treating feature flags and config changes as "not deployments".
- Health checks that only test process liveness.

**Cheat sheet**
- Expand -> migrate -> switch -> contract; each step backward compatible.
- Migration: `LOCK_TIMEOUT`, online/resumable, batch backfills, rehearse on prod-sized data.
- Slot swap: sticky settings, identities, warm-up, swap with preview.
- Expiry: managed identity first; Event Grid near-expiry alerts; rotation runbook.
- Flags: audited, gradual, kill switch, safe defaults, cleanup.
- Gate releases on business metrics, not only HTTP 200.

---

## Topic 11: Third-Party / Dependency Outages

**Study notes**
- Know your dependency map and which are **critical path** vs degradable. Verify via vendor status page, Azure Service Health / Resource Health (for Azure), your own dependency telemetry (App Insights `dependencies` failure/duration by target).
- Patterns: timeouts (every outbound call), circuit breaker, bulkhead (isolate pools/threads per dependency), fallback/degrade (cached data, queue for later), retry with backoff+jitter (idempotent), load shedding, graceful messaging to users, **queue-and-retry** for async-tolerant operations (e.g. send emails later), idempotency keys for payments.
- Contract: SLAs are not your SLO. Composite availability = product of dependency availabilities (3 x 99.9% ~ 99.7%).
- Communication: don't blame the vendor publicly before proof; keep your status page honest; track vendor incident id; escalate through support contracts/TAM.
- Multi-region/ failover for Azure dependencies: paired regions, Geo-DR for Service Bus, zone redundancy, Front Door/Traffic Manager failover.

```
Dependency failing?
 confirm (status page, Service Health, own telemetry) -> contain (breaker/bulkhead, shed non-critical)
  -> degrade (cache/queue/disable feature) -> communicate -> recover (half-open, drain queued work, reconcile)
```

### 11.1 [Easy] A third-party SMS provider is down. What happens to your system and what should happen?
**What the interviewer is really testing:** Resilience design.
**Typical mid-level answer:** "Nothing we can do; wait for them."
**Strong senior answer:** Confirm the outage with vendor status plus our own dependency telemetry. Our system should isolate it: timeouts and circuit breaker so threads/connections aren't consumed, and checkout or login shouldn't fail because an SMS call failed unless the SMS is essential (e.g. OTP). For non-critical notifications, queue them for retry with backoff and TTL; for critical ones, fall back to an alternate channel/provider (email, secondary SMS vendor). Communicate status to support so they can tell customers. After recovery, drain the queue at a controlled rate and check duplicates. Post-incident: classify the dependency's criticality, add a failover provider if the business needs it, and review timeouts.
**Follow-up traps:** "Queued messages are stale by the time vendor returns?" (TTL, drop or reprioritise). "OTP while SMS down?" (alternative factor).

### 11.2 [Follow-up] Explain circuit breaker, bulkhead, timeout, and how they combine.
**What the interviewer is really testing:** Resilience vocabulary and interplay.
**Typical mid-level answer:** Lists names only.
**Strong senior answer:** Timeouts bound how long we wait so resources are released; without them nothing else works. A circuit breaker watches failure rate and opens to fail fast, giving the dependency room to recover, then half-opens to probe. A bulkhead limits concurrency/resources per dependency so one slow service can't exhaust shared threads/connections. Retries handle transient blips with backoff and jitter. Combined order (outer to inner): overall deadline/timeout -> bulkhead -> circuit breaker -> retry -> per-attempt timeout. Plus a fallback for degraded behaviour. In .NET: `Microsoft.Extensions.Http.Resilience` / Polly v8 pipelines on the `HttpClient`.
**Follow-up traps:** "Breaker threshold tuning?" (min throughput, failure ratio, break duration). "Breaker per instance or global?" (per instance by default; shared state is complex).

### 11.3 [Medium] An Azure region has a service degradation affecting your App Service/SQL. What do you do?
**What the interviewer is really testing:** Platform-outage handling and DR judgement.
**Typical mid-level answer:** "Wait for Microsoft."
**Strong senior answer:** Confirm via Azure Service Health (personalised alerts for subscription/region) and Resource Health, and see whether it's our resource or regional. Decide on failover vs wait using the RTO/RPO and the vendor ETA: if multi-region is set up (Front Door/Traffic Manager, SQL failover group, geo-replicated storage), failover per runbook - noting data loss exposure (async geo-replication RPO) and that failover groups can auto-failover with grace period. If no DR exists, communicate honestly, protect data (pause writes that could be lost), and queue work. After: test failover regularly, because untested DR does not work. Set up Service Health alerts to action groups so we learn from Azure not customers.
**Follow-up traps:** "Failover and failback data divergence?" (plan reconciliation). "Zone vs region outage?" (zone redundancy covers the former).

### 11.4 [Hard] Your payment provider times out; some charges succeed at the provider but your system recorded failure. How do you handle it?
**What the interviewer is really testing:** Distributed uncertainty and idempotency.
**Typical mid-level answer:** "Retry the charge." Dangerous: double charge.
**Strong senior answer:** A timeout means unknown outcome, not failure. Treat payment as a state machine: record `Pending` with an **idempotency key** before calling; on timeout, mark `Unknown` and resolve by querying the provider by idempotency key/reference or via webhook/reconciliation job, never blind-retry without idempotency. Retried requests with the same key return the original result. Reconcile periodically with provider settlement reports to catch mismatches. Surface a "processing" status to the user rather than "failed". For the incident: find transactions in `Unknown`/`Failed` within the window, compare with provider records, fix state or refund duplicates, and notify affected customers if charged twice.
**Follow-up traps:** "Webhooks lost?" (poll/reconcile). "Exactly-once?" (idempotency + reconciliation = effectively once).

### 11.5 [Tricky] Your vendor's SLA is 99.9% and your SLO is 99.95%. Is that feasible?
**What the interviewer is really testing:** Reliability math.
**Typical mid-level answer:** "Yes, with good code."
**Strong senior answer:** If the vendor is on the critical path with no degradation path, your availability cannot exceed theirs (and multiplies with others). So a 99.95% SLO requires making the dependency non-critical: caching, async queueing, fallback provider, or degraded modes so failures don't count against user-visible availability. Also SLA credits don't compensate lost revenue; SLAs are contractual minimums and often exclude things. Be explicit about which user journeys depend on the vendor and compute the composite availability. Otherwise, revise the SLO to reality.
**Follow-up traps:** "99.9% means how much downtime?" (~43.8 min/month; 99.95% ~21.9 min/month). "Soft vs hard dependency?" (design-defining).

### 11.6 [Scenario] Your app calls a partner API with no timeout configured. During the partner's outage your own app dies. Walk through diagnosis and the fix.
**What the interviewer is really testing:** Cascading failure.
**Typical mid-level answer:** "Add a timeout."
**Strong senior answer:** Symptom: our latency and errors rise with low CPU; threads/connections pile up waiting on the partner (HttpClient default timeout is 100s), exhausting thread pool/SNAT/connection pool, causing health probes to fail and instances restart (cascade). Evidence: dependency duration for that target climbing to timeout, thread pool queue length rising, dumps showing blocked threads in the HTTP call. Mitigate: feature flag to disable the partner call or return fallback; scale out only as temporary relief. Fix: explicit short timeouts and overall deadline, circuit breaker, bulkhead (SemaphoreSlim/ rate limiter) per partner, async calls, and queue-based decoupling when the business permits. Add synthetic dependency monitoring and a runbook "partner down: flip flag X".
**Follow-up traps:** "Cascade through health checks?" (health probes shouldn't depend on non-critical partners). "Timeout value choice?" (p99 of normal + margin, shorter than caller's deadline).

**Red flags in answers**
- "Wait for the vendor" with no containment or degradation plan.
- Blind retries of payments/charges.
- Health checks depending on non-critical external services.

**Cheat sheet**
- Confirm: vendor page + Azure Service Health + own dependency telemetry.
- Timeout everything; breaker + bulkhead + retry (idempotent only) + fallback.
- Timeout = unknown outcome; use idempotency keys + reconciliation.
- Composite availability < weakest critical dependency; make dependencies soft.
- Test DR; Service Health alerts to action groups.
- Drain queued work gradually after recovery.

---

## Topic 12: Data Corruption & Incorrect-Data Incidents

**Study notes**
- Incorrect data is worse than downtime: it's silent, spreads (downstream systems, reports, customer invoices) and may be hard to reverse. Priority: **stop the bleeding** (halt the writer / feature-flag off / pause jobs and consumers), **preserve evidence** (snapshot, don't overwrite), **scope** (which rows, which time window, which tenants), **repair**, **reconcile/verify**, **communicate** (legal/finance if money or regulated data).
- Scoping: use temporal data and audit tables, `OUTPUT` logs, Change Tracking/CDC, temporal tables (`FOR SYSTEM_TIME AS OF`), event/outbox history, App Insights logs, and **Point-in-Time Restore (PITR)** to a *side-by-side* database (never overwrite prod directly) to compare "before" vs "now".
- Repair strategies: (1) restore from backup/PITR into a new DB and surgically copy back affected rows; (2) compensating transactions/corrective events (preserve audit trail; preferred for event-sourced/financial); (3) replay events/messages through fixed code; (4) scripted correction in batches with dry-run, row count checks, transaction, peer-review, and rollback script; (5) full restore (heavy: loses legitimate new data).
- **Reconciliation:** compare two sources of truth (ledger vs provider settlement, orders vs payments, source vs warehouse). Automate as scheduled invariant checks ("every paid order has exactly one capture").
- Causes: race conditions, missing transactions, bad migration/backfill, timezone/rounding errors, idempotency failure, ETL bug, concurrent updates (lost updates - use rowversion/optimistic concurrency), manual SQL.
- Backups: Azure SQL PITR (7-35 days), LTR; test restores; know RPO/RTO.

```sql
-- Temporal table: what did the row look like before?
SELECT * FROM dbo.Orders FOR SYSTEM_TIME BETWEEN '2025-01-01T10:00' AND '2025-01-01T12:00' WHERE OrderId = 42;
-- Dry-run repair pattern
BEGIN TRAN;
UPDATE o SET o.Total = r.CorrectTotal OUTPUT inserted.OrderId, deleted.Total, inserted.Total INTO dbo.RepairLog
FROM dbo.Orders o JOIN dbo.RepairPlan r ON r.OrderId=o.OrderId;
-- verify @@ROWCOUNT, spot check
ROLLBACK; -- COMMIT only after review
```

### 12.1 [Easy] You discover that a bug has been writing wrong totals for 3 days. What are your first steps?
**What the interviewer is really testing:** Containment and scoping before repair.
**Typical mid-level answer:** "Fix the bug and update the rows." Falls short: no containment, scope or verification.
**Strong senior answer:** First stop further damage: deploy the fix or disable the code path/flag, pause jobs that propagate the bad data (exports, invoices, emails). Then preserve evidence: take a snapshot/PITR copy to a separate database. Scope precisely: define the affected window from the deploy time, find affected rows by a deterministic predicate, and check downstream consumers (reports, finance, third parties). Decide repair strategy per data type (compensating entries for financial data, in-place correction for derived data). Review the repair script with a peer, dry-run on the copy, log before/after values, apply in batches, and reconcile afterwards. Communicate to stakeholders (finance, support, legal if customers were charged wrong amounts), and add an invariant check so it can't go unnoticed again.
**Follow-up traps:** "Restore whole DB from backup?" (loses valid changes since; prefer side-by-side surgical). "Downstream copies?" (data lake, caches, search index).

### 12.2 [Follow-up] How do you repair millions of affected rows safely?
**What the interviewer is really testing:** Operational discipline on bulk fixes.
**Typical mid-level answer:** "Run an UPDATE script."
**Strong senior answer:** Build a repair plan table of (key, old value, new value, reason) computed from source-of-truth data and reviewed/sampled by a second person. Rehearse on a restored copy with realistic volume. Apply in small batches (e.g. 5-10k rows) with short transactions to avoid lock escalation and log growth, throttle for replicas/log rate, include an idempotent predicate (`WHERE value = old`) so reruns are safe and concurrent legitimate updates aren't clobbered, and write an audit/undo log. Monitor blocking and replication lag during the run. After: reconciliation queries comparing expected vs actual and a sign-off. Notify consumers of changed data (events/CDC) so downstream copies re-sync.
**Follow-up traps:** "Triggers and CDC will fire?" (consider impact). "Optimistic concurrency?" (rowversion check to avoid overwriting new edits).

### 12.3 [Medium] How would you design reconciliation to catch data issues early?
**What the interviewer is really testing:** Prevention through invariants.
**Typical mid-level answer:** "Unit tests."
**Strong senior answer:** Define business invariants and compare independent sources: orders vs payments, ledger debit = credit, inventory movements sum to stock, internal totals vs provider settlement files, source DB vs data warehouse counts/checksums. Run scheduled checks (daily or hourly) that output mismatches to a table and alert when above threshold; include aging (how long a mismatch has existed). Make them idempotent, read-only and cheap (use replicas). Include "canary records"/synthetic transactions. Reconciliation results feed an ops dashboard, with an owner and a runbook for each type of mismatch. This turns silent corruption into a visible alert within hours, not weeks.
**Follow-up traps:** "Eventual consistency noise?" (grace window/aging). "Cost?" (incremental windows by modified date).

### 12.4 [Hard] A bad backfill migration overwrote customer emails in production. Restore options and trade-offs?
**What the interviewer is really testing:** Backup/restore judgement.
**Typical mid-level answer:** "Restore last night's backup."
**Strong senior answer:** Restoring in place discards all valid changes since the backup time. Better: PITR to just before the migration into a new database, then join on key and restore only the affected column for rows not modified since (compare `ModifiedAt`/rowversion or temporal history to avoid reverting legitimate updates). For conflicts (customer changed email after the bug) decide policy - latest legitimate wins; consult audit logs. Pause email-sending jobs meanwhile so mail doesn't go to wrong addresses (privacy issue - may be a reportable breach if sent to wrong recipients; involve privacy/legal). Also verify that PITR retention covers the incident, and later add: backfills run on rehearsed scripts with row-level audit and a dry-run, plus a rollback script.
**Follow-up traps:** "Retention expired?" (other sources: logs, events, CDC, warehouse). "Is PITR instant?" (duration scales with size; plan time).

### 12.5 [Tricky] Lost updates are occurring: two users edit the same record and one change vanishes. Why and what's the fix?
**What the interviewer is really testing:** Concurrency control knowledge.
**Typical mid-level answer:** "Use transactions."
**Strong senior answer:** Transactions alone don't prevent the read-modify-write race across separate requests: both read version A, both write, last write wins. Fix with optimistic concurrency: a `rowversion` column and `WHERE Id=@id AND RowVersion=@rv` (EF Core `[Timestamp]`/ concurrency token); on `DbUpdateConcurrencyException` reload and merge or prompt the user. For hot counters/balances use atomic updates (`UPDATE ... SET Balance = Balance - @x WHERE Balance >= @x`) or pessimistic locks (`UPDLOCK`) when contention is high. For incident repair, use audit history to reconstruct lost edits where possible.
**Follow-up traps:** "Serializable isolation?" (heavy; deadlock risk). "ETag in APIs?" (If-Match for HTTP-level concurrency).

### 12.6 [Scenario] Finance says revenue reports are off by 2% since last Tuesday; nobody changed the reporting code. How do you investigate?
**What the interviewer is really testing:** Investigative approach on data issues.
**Typical mid-level answer:** "Rerun the report."
**Strong senior answer:** Treat the report as the symptom. Define "off" against which source of truth; reconcile report vs raw transactions vs payment-provider settlement to find where divergence begins. Check what changed Tuesday outside the reporting code: upstream deploys, ETL schedule or failure (partial loads, duplicated loads), timezone/DST shifts, currency rate source, new product types or a discount code path that stores amounts differently, schema changes, deduplication keys, late-arriving data, and permissions that silently filter rows. Compare row counts/sums by day and by dimension (currency, region, payment method) to isolate the slice causing the 2%. Once localised, trace lineage to the writer. Fix the root cause, re-run/backfill the affected pipelines idempotently, and add the reconciliation check with alerting.
**Follow-up traps:** "Duplicate loads?" (idempotent loads with merge keys). "Timezone?" (store UTC, convert at the edges).

**Red flags in answers**
- Running a production UPDATE without dry-run, audit trail, or peer review.
- Restoring a full backup in place as the default fix.
- No containment of downstream propagation.

**Cheat sheet**
- Stop writer -> preserve -> scope -> repair -> reconcile -> communicate.
- Prefer side-by-side PITR + surgical repair; compensating entries for ledgers.
- Idempotent, batched, audited repair scripts with guards (`WHERE value = old`).
- Reconciliation = scheduled invariants across independent sources.
- Optimistic concurrency (rowversion) for lost updates.
- Check downstream copies (cache, search, warehouse, exports).

---

## Topic 13: Security Incidents

**Study notes**
- Phases (NIST-like): **Detect -> Contain -> Eradicate -> Recover -> Learn**, with legal/privacy/compliance engagement (GDPR 72h breach notification to authority where applicable; contractual obligations). Involve your security team/CISO process; preserve evidence and chain of custody.
- **Leaked secret/token (e.g. in Git, logs):** assume compromised. **Rotate/revoke first** (secret, key, SAS token, PAT, connection string, service principal credential), *then* audit usage during exposure window, then clean up (removing from Git history doesn't un-leak; it's already cloned/scraped). Check cloud audit logs: Entra ID sign-in/audit logs, Azure Activity Log, Key Vault diagnostics (`AuditEvent`), Storage logs, SQL auditing, Defender for Cloud alerts. Add secret scanning/push protection (GitHub secret scanning).
- **Suspicious access:** disable/ reset the account, revoke sessions/refresh tokens, require MFA, review conditional access, check for persistence (new app registrations, role assignments, API keys, forwarding rules), lateral movement, data exfiltration (egress volume, large downloads). Network isolate compromised resources rather than deleting (preserve forensics).
- Prevention: managed identities, least privilege RBAC, Key Vault with RBAC and soft-delete/purge protection, private endpoints, short-lived tokens, WAF, Defender for Cloud, Sentinel alerts, PIM for privileged roles, dependency scanning (SCA), input validation (OWASP), SQL injection prevention (parameterised).
- Comms: need-to-know, separate secure channel, don't discuss on open channels that might be compromised.

```
Suspected compromise
 1. Contain: revoke/rotate, disable account, isolate (don't destroy)
 2. Preserve: logs, snapshots, timeline
 3. Scope: what could the credential do? what did it do? (audit logs, exposure window)
 4. Eradicate: remove persistence, patch root cause
 5. Recover & monitor: rotate dependents, heightened monitoring
 6. Notify (legal/privacy/customers/regulators) and learn
```

### 13.1 [Easy] A developer committed a connection string with a password to a public repo. What do you do?
**What the interviewer is really testing:** Correct ordering: rotate first.
**Typical mid-level answer:** "Remove the commit/force-push." Falls short: the secret is already exposed.
**Strong senior answer:** Treat it as compromised from the moment of push: rotate the credential immediately (and revoke the old), because bots scrape public repos within minutes. Then audit what that credential could access and review logs for use during the exposure window (SQL auditing, firewall/connection logs, sign-ins from unexpected IPs). Remove the secret from the repo and history (and cache/forks) as hygiene, not remediation. Notify security and, if data was accessed, follow the breach process. Prevent: secrets in Key Vault, managed identity so no password exists, GitHub push protection/secret scanning, pre-commit hooks, and least-privilege DB accounts.
**Follow-up traps:** "Repo was private?" (still rotate; lower urgency, same process). "How to know if it was used?" (audit logs and anomaly alerts).

### 13.2 [Follow-up] Suspicious sign-ins to an admin account from another country. What's your plan?
**What the interviewer is really testing:** Contain-audit-eradicate structure.
**Typical mid-level answer:** "Reset the password."
**Strong senior answer:** Contain: block sign-in/ disable the account, revoke all sessions and refresh tokens, reset credentials and MFA methods, and conditional-access block the source. Investigate with Entra sign-in and audit logs: what the actor accessed after sign-in, new app registrations/service principal secrets, role assignments, mailbox rules, API consents, resource changes in Activity Log, Key Vault secret reads, data egress. Look for persistence and lateral movement: those survive a password reset. Eradicate by removing malicious grants and rotating any secrets the account could read. Engage the security team and legal; determine notification obligations. Afterwards enforce phishing-resistant MFA, PIM for just-in-time admin, and alerting for impossible travel/risky sign-ins.
**Follow-up traps:** "Is a password reset enough?" (no: tokens, persistence). "Delete the VM?" (preserve for forensics, isolate).

### 13.3 [Medium] How do you rotate a database credential used by many services with zero downtime?
**What the interviewer is really testing:** Practical rotation design.
**Typical mid-level answer:** "Change the password and restart services."
**Strong senior answer:** Prefer eliminating the secret: Entra ID authentication with managed identities to Azure SQL. If a credential must exist: use the **dual-credential** pattern - create a second login/user, update the secret in Key Vault, roll consumers to the new one (apps reading the secret via Key Vault reference/refresh), verify, then disable the old one after a grace period. Track consumers via sign-in/connection logs to ensure none still use the old credential before revocation. For emergencies (known leak), accept brief disruption: revoke first, fix consumers after. Automate with Key Vault rotation functions and Event Grid, and rehearse.
**Follow-up traps:** "Apps cache connection string?" (restart or refresh logic). "Pooled connections with old creds?" (they die on credential disable; retry/reconnect).

### 13.4 [Hard] You suspect data exfiltration from your storage account. How do you scope it?
**What the interviewer is really testing:** Forensic data sources.
**Typical mid-level answer:** "Check logs." Not specific.
**Strong senior answer:** Ensure diagnostic logging exists (Storage resource logs to Log Analytics, if enabled; if not, that's a finding and limits scoping). Query `StorageBlobLogs` for reads/list operations by caller IP, auth type (SAS vs key vs OAuth), user agent, and bytes out, in the suspected window; compare with the baseline. Check which SAS tokens/keys could have been used and rotate account keys/revoke stored access policies; review network rules/public access, Entra role assignments, and Defender for Storage alerts. Estimate what data was touched (container/blob listing) and classify (PII?). Contain by restricting network access/disabling shared key/ SAS, then follow the breach-notification process with privacy/legal. Prevention: disable shared-key auth where possible, use private endpoints, short-lived user delegation SAS, immutable logging.
```kusto
StorageBlobLogs
| where TimeGenerated between (datetime(2025-01-10) .. datetime(2025-01-12))
| where OperationName in ("GetBlob","ListBlobs")
| summarize bytes=sum(ResponseBodySize), ops=count() by CallerIpAddress, AuthenticationType
| order by bytes desc
```
**Follow-up traps:** "Logs weren't enabled?" (state limits honestly; assume worst-case scope). "Rotating account keys breaks apps?" (inventory; use key1/key2 rotation).

### 13.5 [Tricky] A vulnerability (e.g. critical CVE in a NuGet package) is announced; leadership asks "are we affected?" within the hour. How?
**What the interviewer is really testing:** Inventory and response readiness.
**Typical mid-level answer:** "Search the repos manually."
**Strong senior answer:** This requires an SBOM/dependency inventory: `dotnet list package --vulnerable --include-transitive`, GitHub Dependabot/ Advanced Security dependency graph, Defender for Cloud/container scanning for images. I determine presence, version, and exploitability (is the vulnerable code path reachable; is it internet-facing; compensating controls like WAF). Prioritise internet-facing and sensitive systems; patch via the normal pipeline with expedited review, with temporary mitigations (WAF rule, disabling feature) if patching is slow. Report to leadership: affected services, exposure, mitigation ETA. Afterwards, shorten dependency update cycles and automate scanning in CI.
**Follow-up traps:** "Transitive dependency?" (use `--include-transitive`; override/pin). "Base image vulnerabilities?" (rebuild images).

### 13.6 [Scenario] Alerts show a service principal making hundreds of Key Vault secret reads at 3 AM from an unknown IP. Walk through it.
**What the interviewer is really testing:** Security incident reflex.
**Typical mid-level answer:** "Investigate the IP."
**Strong senior answer:** Declare a security incident and engage security. Contain: disable or rotate the service principal's credential (or remove its Key Vault access via RBAC), apply Key Vault firewall/private network restrictions, and then rotate all secrets that principal could read, since they're assumed compromised. Investigate: Key Vault `AuditEvent` logs for which secrets were read, source IP, user agent, time range; Entra sign-in logs for the service principal; how the credential leaked (repo, pipeline variables, logs, developer machine). Check for follow-on use of the stolen secrets (databases, APIs, storage) and persistence. Recover with managed identities/federated credentials instead of client secrets, and restore access with least privilege. Document timeline and notify as required.
**Follow-up traps:** "Rotating hundreds of secrets?" (prioritise by sensitivity; automation). "Preserve evidence?" (export logs before retention rolls).

**Red flags in answers**
- Deleting a leaked secret from Git without rotating it.
- Resetting a password but not revoking sessions/tokens or checking persistence.
- Discussing the incident in open channels / no legal-privacy involvement.

**Cheat sheet**
- Rotate/revoke first, audit usage second, clean history third.
- Contain without destroying evidence; isolate, snapshot, export logs.
- Sources: Entra sign-in/audit, Activity Log, Key Vault AuditEvent, storage logs, SQL audit, Defender.
- Dual-credential rotation; prefer managed identity.
- Know breach notification duties (e.g. GDPR 72h).
- Keep an SBOM/dependency scan ready for CVE questions.

---

## Topic 14: Capacity & Cost Incidents

**Study notes**
- Cost incidents are production incidents: runaway autoscale (scale-out loop on a bad metric, no max cap), **log/telemetry ingestion bill spike** (verbose logging, debug level left on, high-cardinality custom metrics, dependency/trace noise, retry storm generating exceptions), **poison-message loops** (messages retried repeatedly burning compute, DB and egress), unbounded Functions consumption scale, forgotten test resources, egress charges, over-provisioned SKUs, SQL DTU/vCore overscale, storage transactions explosion.
- Guardrails: Azure Budgets + cost anomaly alerts (Cost Management), autoscale **max instance** caps and sensible cool-down, App Insights daily cap + sampling + log-level config + Basic Logs/data collection rules for filtering, Log Analytics workspace usage queries, tags/ownership, Azure Policy for allowed SKUs, quotas.
- Capacity planning: know the saturating resource and headroom; load-test to find limits; scale on leading metrics (queue length, RPS, CPU) not lagging; scale-in protections; pre-scale for known events (scheduled autoscale); reserved capacity for steady baseline.
- Autoscale pitfalls: flapping (thresholds too close, short cool-down), scaling on a metric that doesn't relieve the bottleneck (scale web tier while DB is bottleneck -> more load on DB), warm-up time longer than spike, scale-out raising DB connection count past limit.

```kusto
// What is driving ingestion cost?
Usage
| where TimeGenerated > ago(7d) and IsBillable == true
| summarize GB=sum(Quantity)/1024 by DataType, bin(TimeGenerated,1d)
| order by GB desc
```
```kusto
// App Insights: which telemetry type is large
union traces, requests, dependencies, exceptions, customEvents
| where timestamp > ago(1d)
| summarize n=count(), bytes=sum(_BilledSize) by itemType
```

### 14.1 [Easy] The App Insights/Log Analytics bill tripled this month. How do you find and fix it?
**What the interviewer is really testing:** Systematic cost forensics.
**Typical mid-level answer:** "Turn down logging." Falls short: unclear what's causing it.
**Strong senior answer:** Identify the contributor first: query `Usage`/`_BilledSize` by data type, table, operation/cloud role, and day to find the inflection and match it to a deploy or incident. Typical culprits: Debug/Trace log level in prod, a new chatty log line in a hot loop, exceptions from a retry storm, dependency tracking of a high-frequency call, high-cardinality custom dimensions, or health-check noise. Fix at the source (log levels via configuration, remove per-request verbose logs), apply sampling/telemetry processors to filter noise (health probes), move high-volume diagnostic logs to cheaper tiers (Basic Logs) or storage, set a daily cap with alerting as a safety net (understanding a cap loses data), and add a budget/anomaly alert. Keep error and key business telemetry unsampled.
**Follow-up traps:** "Daily cap risk?" (you lose telemetry during an incident - cap high and alert at 80%). "Sampling skews metrics?" (use pre-aggregated metrics and `itemCount`).

### 14.2 [Follow-up] Autoscale scaled out to 50 instances overnight and cost spiked, but traffic was flat. Diagnose.
**What the interviewer is really testing:** Autoscale correctness.
**Typical mid-level answer:** "Autoscale misbehaved."
**Strong senior answer:** Check the autoscale activity log to see which rule fired and the metric value. Hypotheses: scaling on CPU driven by something not traffic (a background job, GC storm, a retry storm from a failing dependency), memory metric that rises monotonically (leak) so never scales in, queue-length rule with a poison backlog that never drains, a missing max cap, scale-in rule absent/cool-down too long, or flapping. I look at what caused metric growth - scaling treated the symptom. Fix: cap max instances, correct rules (separate scale-out/in with margin), scale on leading indicators, address the underlying runaway, set budget alerts and review. Test autoscale behaviour under load; consider scheduled profiles.
**Follow-up traps:** "Scale-out didn't help latency?" (bottleneck is elsewhere, e.g. DB). "Scale-in kills in-flight work?" (graceful shutdown/drain).

### 14.3 [Medium] A poison message loop is burning money. How does it happen and how do you stop it?
**What the interviewer is really testing:** Cost awareness of retries.
**Typical mid-level answer:** "Delete the message."
**Strong senior answer:** A message that always fails keeps being redelivered (up to MaxDeliveryCount per message, but multiplied by number of messages and any republishing logic - some code re-enqueues failures itself, creating an infinite loop with no cap). Each attempt costs compute, DB calls, external API calls (possibly billed), and log volume. Stop it: identify the messages by repeating MessageId/DeliveryCount, dead-letter or quarantine them, pause the consumer if needed. Fix: classify permanent vs transient errors, DLQ permanent ones immediately, cap retries with backoff, never self-requeue without a counter, and alert on delivery-count and DLQ growth plus cost anomaly alerts. Include cost in the failure analysis (e.g. a Functions-on-Consumption bill).
**Follow-up traps:** "Retry via re-send?" (new message id loses delivery count - dangerous). "Cost of DLQ'd messages?" (negligible; storage only).

### 14.4 [Hard] How would you prepare for a Black Friday 10x traffic peak from a capacity standpoint?
**What the interviewer is really testing:** Systematic capacity planning.
**Typical mid-level answer:** "Scale everything up."
**Strong senior answer:** Start from forecast (peak RPS per journey, with margin) and find the bottleneck chain by load testing a production-like environment with realistic data skew; fix the first bottleneck, re-test until the target is met with headroom (e.g. 2x). Review each tier: app instances/pre-scale (don't depend on reactive autoscale for a spike that arrives in seconds), DB (DTU/vCore, read replicas, indexing, connection counts across instances), cache sizing, queue/consumer throughput, third-party rate limits (ask vendors to raise limits), CDN offload, and quotas (subscription core quota, SNAT, Service Bus throttling). Plan degradation: load shedding, disable non-essential features via flags, queue-based order intake, waiting-room. Freeze non-essential changes, set up a war-room with named on-call, dashboards, a go/no-go checklist, runbooks, and a rollback-ready release. After the event: capture cost and scale down safely.
**Follow-up traps:** "Autoscale enough?" (warm-up lag; scheduled pre-scale). "Third-party limits?" (the hidden ceiling).

### 14.5 [Tricky] A "cost optimisation" project cut the SKU and the app is now slower and flaky. How do you balance cost vs reliability?
**What the interviewer is really testing:** Engineering economics.
**Typical mid-level answer:** "Revert; reliability matters more."
**Strong senior answer:** Frame it with data: cost per transaction, SLO attainment, and the revenue risk of degradation. Cost cuts should come with a measurable hypothesis ("CPU p95 < 40%, we can halve capacity"), staged rollout with SLO guardrails and rollback. Here, I'd revert or partially restore capacity to bring SLO back, then investigate with load tests what size meets SLO at lowest cost, and pursue efficiency (code, caching, query tuning, reserved instances/savings plans, scheduled scale-down for non-prod) instead of blind downsizing. Include headroom for failover (N+1) and spikes. Make the trade-off explicit to stakeholders: error budget burn vs saved dollars.
**Follow-up traps:** "Who decides?" (product + engineering with SLO data). "Non-prod savings?" (auto-shutdown).

### 14.6 [Scenario] A dev environment left a Premium SQL elastic pool and several Standard-tier VMs running for two months. How do you fix it as a process, not a one-off?
**What the interviewer is really testing:** Governance mindset.
**Typical mid-level answer:** "Delete them and remind people."
**Strong senior answer:** First verify nobody needs them (owner tag, activity metrics) and snapshot before deleting, then remove. The systemic fix: mandatory tags (owner, environment, expiry) enforced by Azure Policy, budgets and anomaly alerts per subscription/resource group with owner notification, auto-shutdown schedules for dev/test, IaC-provisioned ephemeral environments with TTL destroy, regular orphan resource reports (Azure Advisor, Resource Graph queries), and FinOps visibility. Cost showback to teams drives behaviour. Keep it blameless: fix the system so the default is cheap.
**Follow-up traps:** "Delete risks?" (confirm owner; soft-delete/backups). "Policy too strict?" (exemptions with expiry).

**Red flags in answers**
- Scaling out as the only response to everything.
- No upper bounds (max instances, daily caps, budgets).
- Ignoring cost as an operational signal.

**Cheat sheet**
- Find the cost driver by dimension (type/table/role/day), match to change.
- Autoscale: caps, cool-downs, leading metrics, check the true bottleneck.
- Poison loops: classify errors, cap retries, DLQ, alert on delivery count.
- Pre-scale for known peaks; load-test with skew; plan degradation.
- Budgets + anomaly alerts + tags + policy.
- Right-size with SLO data, not guesses.

---

## Topic 15: Monitoring, Alerting Quality & On-Call Sustainability

**Study notes**
- **Alert on symptoms (user pain), investigate causes.** Page for: SLO burn, error rate, latency, failed synthetic checks, queue age. Cause metrics (CPU, memory) go on dashboards or low-priority tickets unless they predict imminent failure (disk filling, cert expiring).
- **Every page must be urgent, actionable, and need a human now.** Otherwise it's a ticket/log.
- **SLI/SLO/error budget:** SLI = measurable (good events / valid events); SLO = target (99.9% over 30 days); error budget = 1 - SLO (43.2 min/30 days at 99.9%).
- **Multi-window multi-burn-rate alerts** (Google SRE Workbook): e.g. page when burn rate > 14.4 over 1h AND 5m (2% budget in 1h), also > 6 over 6h and 30m (5% in 6h); ticket when > 1 over 3 days (10%) and 6h. Short window makes alert reset quickly; long window prevents noise.
- Azure tooling: Azure Monitor metric/log/ smart detection alerts, action groups (Teams, PagerDuty, SMS), availability tests (multi-location), Application Insights, Workbooks/Grafana dashboards, Service Health alerts, alert processing rules (maintenance suppression), dynamic thresholds.
- **Dashboard design:** answer questions in order: Is it broken (RED/SLO)? Where (by service/region/version)? Why (dependencies, saturation)? What changed (deploy annotations)? Fewer, purposeful dashboards; link alert -> dashboard -> runbook.
- **Runbooks:** trigger, impact, quick checks, mitigations (with commands), escalation, owner, last-reviewed date. Executable and tested; reduce toil with automation.
- **On-call sustainability:** rotation size (>= 6-8 ideal for primary/secondary), follow-the-sun if possible, compensation/time-off in lieu, page budget (e.g. <= 2 incidents per 12h shift), handoffs, shadowing for new people, review alert load each rotation, fix top noisy alerts, protect after-hours sleep, escalation policy, "no hero" culture.

```
Burn rate = (error rate observed) / (error rate allowed by SLO)
SLO 99.9% => allowed 0.1%.  Observed 1.44% errors => burn 14.4 => budget gone in ~2 days (30d/14.4)
```

### 15.1 [Easy] What makes a good alert, and what makes a bad one?
**What the interviewer is really testing:** Alert design principles.
**Typical mid-level answer:** "Alert when CPU > 80%." Cause-based, noisy, rarely user-impacting.
**Strong senior answer:** A good alert is symptom-based, actionable, urgent, has a clear owner, a linked runbook/dashboard, and a threshold tied to user impact (SLO burn, error rate, latency, failed synthetic, queue oldest-message age). A bad alert fires without needing action, flaps, can't be explained, duplicates others, or reflects a cause that doesn't hurt users (CPU 80% while latency is fine). Cause metrics are useful for diagnosis dashboards, and for *predictive* conditions (disk will fill in 4h, cert expires in 7 days) as ticket-level alerts. I review alerts regularly: if an alert was ignored or auto-resolved more than a couple of times, change or delete it.
**Follow-up traps:** "No alert for CPU at all?" (maybe a warning-level/ticket; page only if tied to impact). "Missing-data alerts?" (needed to detect silent failure).

### 15.2 [Follow-up] Explain SLO burn-rate alerting and why it beats static thresholds.
**What the interviewer is really testing:** Modern alerting knowledge.
**Typical mid-level answer:** "Alert if error rate above 1%."
**Strong senior answer:** A static threshold ignores how much budget is being consumed: 1% errors for 2 minutes is trivial, 0.2% for 10 days exhausts a 99.9% budget silently. Burn rate = observed error rate / allowed error rate. Multi-window multi-burn-rate alerts page on fast burns (e.g. 14.4x over 1h with a 5m confirmation) and ticket on slow burns (e.g. 1x over 3 days), balancing detection time, precision, and reset time. They tie alerts directly to the business promise (SLO) and make release/freeze decisions based on remaining error budget. Implement with KQL scheduled queries over request success ratios or Azure Monitor/Prometheus recording rules.
```kusto
requests
| where timestamp > ago(1h)
| summarize bad=countif(success==false and resultCode startswith "5"), total=count()
| extend errRate = todouble(bad)/total, burn = (todouble(bad)/total)/0.001
```
**Follow-up traps:** "Low-traffic services?" (burn alerts noisy with few requests; use longer windows/synthetics). "Which errors count?" (server-side faults, not 4xx client errors; define SLI carefully).

### 15.3 [Medium] On-call engineers are getting 40 alerts per night and ignoring them. What do you do?
**What the interviewer is really testing:** Alert fatigue remediation.
**Typical mid-level answer:** "Tell people to respond faster."
**Strong senior answer:** Treat alert load as a defect backlog. Export a month of alerts, group by alert rule and outcome (actionable, self-resolved, ignored, duplicates). Delete or downgrade those that never needed action; merge duplicates; convert cause alerts to symptom alerts; add dampening (for-duration, multi-window), maintenance suppression, and routing by severity (page vs ticket vs dashboard). Fix the top 5 recurring offenders at the root (automate the toil, fix the flapping dependency). Set a goal (e.g. > 50% of pages actionable, <= 2 pages/shift) and review it in the weekly ops meeting. Show leadership the on-call cost (sleep loss, attrition) to prioritise reliability work over features.
**Follow-up traps:** "Silencing risks missing real issues?" (pair with SLO alerts as the safety net). "Who owns alert quality?" (service team).

### 15.4 [Hard] Design monitoring for a new order service from scratch.
**What the interviewer is really testing:** Observability design.
**Typical mid-level answer:** Lists tools without structure.
**Strong senior answer:** Start from user journeys and define SLIs/SLOs: availability (successful order submissions / valid attempts), latency (p95 < X ms), freshness/processing (order processed within N min - queue age). Instrument: structured logging with correlation id (W3C traceparent), distributed tracing (OpenTelemetry/Application Insights), RED metrics, dependency telemetry, business metrics (orders/min, payment success rate), and deploy annotations. Alerts: burn-rate on availability/latency SLOs, oldest message age, DLQ > 0, synthetic transaction, Service Health, cert expiry (predictive). Dashboards: one overview (golden signals + SLO + recent deploys) and drill-downs (dependencies, DB, queue). Runbooks per alert; ownership and escalation; test the alerts (fire drill/chaos). Review quarterly.
**Follow-up traps:** "Where do logs vs metrics vs traces fit?" (metrics detect, traces localise, logs explain). "Cardinality?" (avoid unbounded labels).

### 15.5 [Tricky] Your team's runbooks are out of date and on-call people ignore them. Fix?
**What the interviewer is really testing:** Making runbooks live.
**Typical mid-level answer:** "Ask the team to update the wiki."
**Strong senior answer:** Make runbooks part of the alert definition: the alert links directly to the runbook, and the incident template includes "was the runbook accurate? update it" as a closing step. Keep runbooks short, command-ready, and verified through game days and during onboarding (new joiner runs each one). Prefer automation: convert repeated steps into scripts, Logic Apps/automation runbooks, or auto-remediation (with guardrails). Assign owners and last-reviewed dates; stale ones surface in a monthly review. Measure usage and time-to-mitigate to prove value.
**Follow-up traps:** "Auto-remediation risks?" (blast radius; log actions; rate-limit). "Tribal knowledge?" (capture in postmortems).

### 15.6 [Scenario] You're asked to improve on-call for a team of 4 that is burning out. What do you propose?
**What the interviewer is really testing:** Sustainable operations and leadership.
**Typical mid-level answer:** "Hire more people." Slow and not enough.
**Strong senior answer:** Measure first: pages per shift, after-hours pages, time-to-resolve, top sources, and who carries the load. Reduce load: kill noisy alerts, automate toil, fix the top recurring incidents (reliability work prioritised via error budget policy). Change the structure: with only four people, 1-in-4 weekly primary is heavy; options include business-hours-only on-call for non-critical services, a secondary escalation tier, a combined rotation with adjacent teams who share the platform, or follow-the-sun if a distributed team exists. Add support: compensation/time off after night pages, no-meeting recovery day, structured handoffs, shadowing to spread knowledge, clear escalation so nobody is alone. Report burnout risk to management with data and ask for reliability capacity in the roadmap. Revisit after two rotations.
**Follow-up traps:** "Management refuses time for reliability?" (use error-budget policy and incident cost). "Single expert dependency?" (pair/knowledge transfer).

**Red flags in answers**
- Paging on causes (CPU, memory) rather than user-visible symptoms.
- Treating alert fatigue as a people problem instead of a system problem.
- Runbooks that are wiki pages nobody tests.

**Cheat sheet**
- Page = urgent + actionable + symptom-based + owned + runbook linked.
- SLI/SLO/error budget; 99.9% = ~43 min/month budget.
- Multi-window multi-burn-rate: 14.4x/1h page, 1x/3d ticket.
- Dashboards: broken? where? why? what changed?
- Audit alerts regularly; delete noise; automate toil.
- On-call: rotation size, page budget, recovery time, shadowing, no heroes.

---

## Topic 16: Blameless Postmortems & Near-Misses

**Study notes**
- **Blameless** = assume people acted rationally with the information they had; focus on how the system allowed the failure. Not "no accountability" - accountability is for improving the system and follow-through. Individuals report openly only if safe.
- Structure: Summary; Impact (users, duration, revenue, SLO budget burn); Timeline (UTC, detection, escalation, mitigation, resolution); Root cause(s)/**contributing factors**; What went well/poorly/where we got lucky; Detection (how we found out; time-to-detect, time-to-mitigate); Action items (owner, priority, due date, tracked in the backlog); Lessons learned; Supporting data.
- **5 Whys** is simple but linear, tends toward a single root cause and ends at "human error"; **systems thinking** (contributing factors, Swiss-cheese model, safeguards that failed or were missing, normal-work vs work-as-imagined) yields multiple factors and better actions. Use 5 whys as a prompt, not the model. "Human error" is a starting point of the investigation, not the root cause.
- **Action items that get done:** few, specific, owned, with due dates, prioritised against roadmap; classify as prevent / detect / mitigate; prefer system-level fixes (automation, guardrails, tests) over "be more careful" or "add documentation"; track completion in a review; an exec sponsor; closure verification.
- **Repeat incidents** signal weak actions or deprioritisation: link incidents by theme, review the recurring ones, escalate unfinished high-priority items.
- **Near-misses** (caught by luck/good catch) are free lessons: record them with the same rigour (lightweight), share widely, celebrate reporting.
- Facilitation: written draft before the meeting, timeline from data not memory, neutral facilitator, ask "how" not "who/why did you", hindsight/outcome bias awareness, publish internally and share learnings.

```
Postmortem skeleton
1 Summary  2 Impact  3 Timeline  4 Detection  5 Contributing factors
6 What went well / badly / lucky  7 Actions (owner/due/priority/type)  8 Lessons
```

### 16.1 [Easy] What is a blameless postmortem and why is it valuable?
**What the interviewer is really testing:** Cultural understanding.
**Typical mid-level answer:** "A meeting where we don't blame anyone." Surface-level.
**Strong senior answer:** It's a structured review that treats an incident as a system failure: the person who made the change was acting reasonably given tooling, information and incentives, so asking "who" yields hiding and fear, while asking "how did our system make this easy to do and hard to detect" yields fixes. It's still accountable: actions have owners and dates. Value: honest timelines (people share what really happened), organisational learning, and prevention of repeats. The outputs are contributing factors and prioritised actions across prevention, detection and mitigation, plus what went well to preserve.
**Follow-up traps:** "What about negligent behaviour?" (distinct from honest mistakes; handled through management, not the postmortem). "Blameless but names appear?" (roles/ actions, not blame).

### 16.2 [Follow-up] 5 Whys vs systems thinking - which do you use?
**What the interviewer is really testing:** Depth of analysis.
**Typical mid-level answer:** "5 Whys always."
**Strong senior answer:** 5 Whys is a useful prompt but assumes a single causal chain and tends to stop at a convenient answer such as "engineer forgot to run the test". Real incidents have multiple contributing factors: a latent bug, a missing guardrail, weak detection, an unclear runbook, time pressure. I use a contributing-factors approach (e.g. layers of defence / Swiss cheese): which safeguards existed and why they failed or were absent - code review, tests, canary, alert, rollback, capacity - and what conditions made the failure likely. Questions are "how" rather than "why did person X". Actions then target several layers (prevent, detect, mitigate) instead of one.
**Follow-up traps:** "Where do you stop?" (when actions are actionable and within your control). "Root cause singular?" (there is rarely one).

### 16.3 [Medium] How do you ensure postmortem action items actually get done?
**What the interviewer is really testing:** Follow-through.
**Typical mid-level answer:** "Create Jira tickets."
**Strong senior answer:** Limit to the few actions with the best risk reduction and make each SMART: specific, owner, due date, success criteria, type (prevent/detect/mitigate). Put them in the same backlog as feature work, prioritised against it with explicit product agreement (e.g. P0 actions within 30 days), and review completion weekly or in a monthly ops review with leadership visibility. Prefer automation and guardrails to documentation. Track metrics: action-item closure rate and age, repeat-incident rate. If actions repeatedly slip, escalate and treat it as an organisational risk - an unfinished action is an accepted risk that needs an owner's explicit signature.
**Follow-up traps:** "Too many actions?" (filter; 'nice-to-have' separated). "Product refuses time?" (error-budget policy).

### 16.4 [Hard] The same type of incident recurs for the third time. What's wrong and what do you do?
**What the interviewer is really testing:** Systemic analysis of repeats.
**Typical mid-level answer:** "Be more careful."
**Strong senior answer:** Repeats mean prior actions were weak (training/documentation instead of automation), not completed, or addressed symptoms. I audit the previous postmortems: which actions existed, were they done, did they work? Then look for the underlying class: e.g. three "config missing in prod" incidents point to absent config validation and IaC parity, not to three careless engineers. Propose a class-level fix (startup config validation, policy-as-code, pre-deploy checks) with a sponsor and a measurable result. Present the pattern with cost data to leadership to justify prioritisation. Add detection and mitigation improvements in the meantime.
**Follow-up traps:** "Teams blame each other?" (shared ownership of the class). "How do you know the fix worked?" (leading indicators + no recurrence over N months).

### 16.5 [Tricky] Management wants a named person responsible in the postmortem. How do you respond?
**What the interviewer is really testing:** Maintaining blameless principles with leadership.
**Typical mid-level answer:** Complies or refuses flatly.
**Strong senior answer:** Explain the goal behind the request: they want accountability and assurance it won't recur. Offer accountability for *actions* and for system improvement (named owners with deadlines) and for the incident process (who was commander). Explain that naming a person as the cause teaches people to hide mistakes and doesn't prevent recurrence since the next engineer faces the same conditions. If there was genuine negligence or policy violation, that is a separate management conversation outside the postmortem. Give leadership a clear narrative: what happened, why defences failed, what we change, and by when.
**Follow-up traps:** "Customer demands someone be held accountable?" (organisational accountability, formal RCA). "Individuals still learn?" (private coaching, not public blame).

### 16.6 [Scenario] A deployment was stopped by a reviewer at the last minute, who spotted a missing index that would have caused a production outage. Nobody noticed. What do you do?
**What the interviewer is really testing:** Learning from near-misses.
**Typical mid-level answer:** "Good catch; move on."
**Strong senior answer:** This is a near-miss and its value equals an incident's. I record it with a short write-up: what would have happened, why the safeguards (tests, perf checks) didn't catch it, and the fact that detection relied on one person's luck/expertise. Ask what systemic guard would catch this next time: automated query plan checks in CI, a performance test on prod-sized data, a migration/index checklist, or static analysis. Recognise the reviewer publicly to encourage reporting, share the story in the team/engineering forum, and track the action item. Measure near-miss reporting rate as a healthy-culture indicator.
**Follow-up traps:** "Is it worth a full postmortem?" (lightweight version). "How to encourage reports?" (reward, no punishment).

**Red flags in answers**
- "Human error" as the final root cause.
- Action items such as "be more careful" / "add a wiki page".
- No owner/due date or follow-up on actions.

**Cheat sheet**
- Blameless = system focus + clear accountability for actions.
- Contributing factors > single root cause; 5 Whys only as a prompt.
- Timeline from data; metrics: TTD, TTM, TTR, budget burned.
- Actions: few, owned, dated, automated, tracked with leadership visibility.
- Repeats = weak actions; fix the class of problem.
- Treat near-misses as free incidents.

---

## Topic 17: Reproduction Strategies & Safe Production Debugging

**Study notes**
- "Works on my machine / only prod" = an **environment, data, scale, timing, or config difference**. Enumerate differences systematically: versions (runtime, OS, dependencies), config/secrets, data volume/shape/skew, concurrency & load, network topology (private endpoints, proxies, TLS), identities/permissions (managed identity vs developer login), time (timezone, clocks, expiry), hardware (cores, memory limits), feature flags, caches warm/cold, third-party sandbox vs live.
- Reproduction ladder: replay the exact failing request from telemetry (inputs, headers, tenant) in a lower env -> restore anonymised prod snapshot -> run with prod config in an isolated slot -> record & replay traffic/shadow traffic -> load/concurrency tests (race conditions) -> chaos/fault injection (timeouts, throttling) -> time travel (fake clock).
- Hard-to-reproduce classes: race conditions, timing, memory pressure, data-dependent, rare retry paths, DST/leap, culture/locale, large payload, TLS/protocol quirks.
- **Debugging in production safely:** least invasive first. Order: existing telemetry -> add logging/tracing (dynamic log level, sampling, targeted by user/tenant via filter/flag) -> metrics -> **dumps/traces** (`dotnet-dump`, `dotnet-trace`, Profiler - low overhead; capture and analyse offline) -> canary/one instance with extra diagnostics -> Snapshot Debugger (non-breaking snapshots) -> **never** attach an interactive debugger that halts a production process on a live instance; remote debugging only for isolated instances.
- Data safety: PII in logs/dumps (dumps contain memory incl. secrets/PII - secure storage, access control, retention, delete after); read-only DB access (use a read replica/ Query Store); don't "just run an UPDATE".
- Controls: change tickets even for diagnostics, time-boxed log-level raise with automatic revert, sampling to control cost, correlation ids, targeted diagnostic flag per request (e.g. header-based `X-Debug` for authorised users).

```
Can't reproduce?
 1 Get the exact inputs/context from telemetry (operation_Id)
 2 Diff environment: version/config/data/scale/identity/time
 3 Make dev look like prod (prod config in isolated slot; sanitised data copy)
 4 Force the timing: concurrency harness, delay injection, fake clock
 5 If still no: instrument prod minimally (targeted logging/trace/dump)
```

### 17.1 [Easy] A bug can't be reproduced locally. What is your approach?
**What the interviewer is really testing:** Systematic environment-difference thinking.
**Typical mid-level answer:** "Add more logging and wait."
**Strong senior answer:** I start by extracting everything the production telemetry knows about the failing instance: request inputs, user/tenant, headers, timing, exception stacks, dependency results, app version, and config. Then I list differences between that context and local: data shape (large tenant, nulls, Unicode), config/feature flags, identity/permissions, concurrency, versions, network/TLS, time zone/culture. I reproduce progressively closer to prod: replay the request in a staging slot against a sanitised copy of prod data; use a test harness to force concurrency or inject delays if timing is suspected. If still elusive, I add targeted low-overhead instrumentation (structured log for that path, activity tags) behind a flag and examine the next occurrence rather than guessing.
**Follow-up traps:** "Data privacy for copying prod data?" (anonymise/mask; restricted access). "Heisenbug?" (observation changes timing; use trace-based evidence).

### 17.2 [Follow-up] How do you debug a suspected race condition that only appears in production?
**What the interviewer is really testing:** Concurrency debugging.
**Typical mid-level answer:** "Add locks."
**Strong senior answer:** Evidence first: look for symptoms of lost updates, duplicates or intermittent null states correlated with concurrency or specific instances. Reconstruct the interleaving from traces/logs with high-resolution timestamps and correlation ids. Reproduce by amplifying: run N parallel requests against the same entity in a test harness, add controlled delays (`Task.Delay`) at suspected points, use a stress test. Review shared mutable state (static fields, singleton with state, non-thread-safe collections, `DateTime.Now`, check-then-act), database read-modify-write without concurrency token, caches populated concurrently. Fix with the appropriate mechanism: immutability, `ConcurrentDictionary` with correct semantics (`GetOrAdd` with factory may run multiple times), optimistic concurrency / unique constraints / atomic SQL; locks only where the scope is single-process. Add a regression test using the stress harness.
**Follow-up traps:** "Locks across multiple instances?" (don't; use DB constraints or distributed lock carefully). "Singleton holding per-request state?" (classic bug).

### 17.3 [Medium] What can you safely do in production to debug a live issue, and what must you avoid?
**What the interviewer is really testing:** Safe-ops judgement.
**Typical mid-level answer:** "Attach the debugger."
**Strong senior answer:** Safe, increasing in invasiveness: read existing telemetry; dynamic log-level increase (time-boxed, scoped to a category or tenant, with sampling and cost awareness); distributed traces; `dotnet-counters` for live metrics; sample-based `dotnet-trace` or Profiler (low overhead); a memory dump of a single instance taken after removing it from rotation (a dump freezes the process for seconds-minutes proportional to heap size - plan for it); Snapshot Debugger. Avoid: breakpoint-halting debuggers on live instances, ad-hoc UPDATE/DELETE, shipping unreviewed "debug" builds, logging PII/secrets, leaving elevated logging on, running heavy queries against the primary at peak. Treat dumps as sensitive data: encrypted storage, restricted access, deletion after analysis. Record what you did in the incident timeline.
**Follow-up traps:** "Dump on a container?" (ephemeral container with tools sharing process namespace; must save to persistent volume before the pod dies). "Taking a dump pauses the app?" (yes, briefly).

### 17.4 [Hard] How do you use feature flags and canaries to debug or validate a fix in production?
**What the interviewer is really testing:** Progressive verification.
**Typical mid-level answer:** "Deploy to all and watch."
**Strong senior answer:** Ship the fix dark behind a flag, enable first for internal users, then a small percentage or a single tenant/ring (canary instance or slot), and compare SLIs against the control group (error rate, latency, business metrics) before expanding. For diagnosis, flags can enable verbose diagnostics only for a specific tenant or request cohort. Prerequisites: the ability to segment telemetry by flag state/version, a tested kill switch, and automated analysis/rollback thresholds. Danger areas: data written by the new path must be compatible with the old; flags increase combinatorial states and require cleanup. Roll out progressively (1% -> 5% -> 25% -> 100%) with hold periods covering peak.
**Follow-up traps:** "Canary at low traffic gives no signal?" (synthetic traffic or longer hold). "Sticky users?" (consistent hashing so a user stays in a cohort).

### 17.5 [Tricky] Logs are insufficient and there's no budget for verbose logging in production. How do you improve observability without huge costs?
**What the interviewer is really testing:** Smart telemetry design.
**Typical mid-level answer:** "Turn on debug logs."
**Strong senior answer:** Use structured, meaningful logs at the right level (errors/warnings always; info for business events), traces for flow, and metrics for counts. Employ adaptive/fixed-rate sampling for high-volume success telemetry while retaining 100% of failures and slow requests; use tail-based sampling where available (OpenTelemetry collector) so error traces are kept. Add on-demand diagnostics: dynamic log level by category/tenant for limited time, "debug header" for authorised testers, ring buffer logging that flushes detail only when an error occurs, and exemplars linking metrics to traces. Use data collection rules/transformations to drop noise and Basic Logs for cheap, rarely-queried data. Define what questions we need to answer and instrument for those rather than log everything.
**Follow-up traps:** "Sampling and rare bugs?" (keep all failures and tag by feature). "Correlation across services?" (W3C trace context).

### 17.6 [Scenario] A bug appears only in production for customers in a particular timezone around midnight. Walk through finding and proving the cause.
**What the interviewer is really testing:** Time/data-dependent bug reasoning.
**Typical mid-level answer:** "Probably timezone handling."
**Strong senior answer:** Pull failing request examples to confirm they cluster around local midnight/UTC offsets and what operation they hit. Hypotheses: `DateTime.Now` vs `UtcNow`/`DateTimeOffset` mixing, date-only comparisons using server time zone (App Service is UTC, dev machine isn't), DST transitions, daily batch cut-off boundaries, SQL `GETDATE()` vs `SYSUTCDATETIME()`, midnight rollover of "today" computed once at startup. Reproduce deterministically by injecting a clock (`TimeProvider` in .NET 8) and running tests with the failing timestamps and timezones, including DST edges. Fix by storing UTC, converting at the edges using IANA/Windows tz ids, and passing time via `TimeProvider`. Add regression tests across time zones and boundary instants; check historical data for already-incorrect values.
**Follow-up traps:** "Server timezone different locally?" (that's why it works on dev machine). "Existing wrong data?" (reconciliation/repair).

**Red flags in answers**
- Attaching a breaking debugger to production or running ad-hoc data modifications.
- Not treating dumps/logs as sensitive data.
- Guessing instead of listing environmental differences.

**Cheat sheet**
- List differences: version, config, data, scale, identity, time, network.
- Replay exact inputs from telemetry; use sanitised prod-like data.
- Force timing with harnesses, delay injection, `TimeProvider`.
- Least-invasive-first: telemetry -> targeted logs -> counters -> trace/dump on one instance.
- Flags/canary + segmented telemetry to validate fixes.
- Sample success, keep 100% of failures.

---

## Topic 18: Change Management, Release Safety & Post-Incident Prevention

**Study notes**
- Most outages follow changes -> make changes **small, frequent, reversible, observable**. DORA metrics: deployment frequency, lead time, change failure rate, MTTR - elite performance has both high speed and stability.
- **Progressive delivery:** feature flags, canary releases (1-5% then ramp), blue/green & slot swaps, ring-based rollout (internal -> early adopters -> all), region-by-region (safe deployment practices: one region at a time, bake time between rings), automated health gates and auto-rollback, deployment markers.
- **Error-budget policy:** if budget is healthy, ship fast; if exhausted, freeze risky feature releases and focus on reliability until recovered. This makes the reliability-vs-velocity debate data-driven and pre-agreed with product.
- **Change freezes** (holiday peaks) are blunt: reduce risk but batch up changes (bigger risk later); prefer lower-risk changes allowed with extra approvals; emergency change path.
- **Change management proportional to risk:** automated pipelines and peer review for standard changes; CAB-style review only for high-risk/irreversible (schema contract, region migration); pre-approved "standard changes"; avoid heavyweight approval that slows without reducing risk (research suggests external approvals correlate with worse outcomes than peer review + automation).
- **Post-incident prevention toolkit:** test pyramid + contract tests; integration tests with real dependencies (Testcontainers); load/soak tests with realistic data; chaos/fault-injection (Azure Chaos Studio); **game days** (practise incidents: dependency failure, region failover, secret expiry, DB failover); DR drills; pre-production parity via IaC; policy-as-code; static analysis; performance budgets in CI; production readiness reviews (PRR) before launching a service; checklists for risky operations.

```
Safe release flow
 PR review + CI tests -> deploy to staging (smoke) -> prod canary/ring 0 (bake) -> health gate
 -> ring 1 -> ... -> 100%, with auto-rollback + flags + deploy markers
 Reliability gate: error budget remaining? yes -> proceed; no -> reliability work first
```

### 18.1 [Easy] How do you make releases safer without slowing the team down?
**What the interviewer is really testing:** Balance of speed and safety.
**Typical mid-level answer:** "More approvals and manual testing."
**Strong senior answer:** Reduce batch size and automate: frequent small releases are easier to reason about and roll back. Add automated gates - unit/integration/contract tests, static analysis, schema-migration checks - then deploy progressively (canary/slot warm-up/rings) with automated health checks on SLIs and fast rollback. Decouple deploy from release with feature flags. Make changes observable through deployment markers and per-version telemetry. Peer review for code, lightweight risk-based review for high-risk changes only. Measure DORA metrics (change failure rate, MTTR) to see whether safety actually improves. More manual approvals tend to slow delivery without reducing failures.
**Follow-up traps:** "Compliance requires approvals?" (automate evidence, approvals in pipeline). "Hotfix path?" (still goes through pipeline).

### 18.2 [Follow-up] What is an error-budget policy and how do you use it for release decisions?
**What the interviewer is really testing:** SRE practice.
**Typical mid-level answer:** "Don't release when there are bugs."
**Strong senior answer:** The SLO defines acceptable unreliability; the error budget is the remainder. Policy (agreed with product in advance): while the budget is healthy, ship normally; when burn is high or the budget is exhausted, pause risky feature releases and prioritise reliability work until recovery, while allowing fixes and security patches. It turns arguments into data: both product and engineering share the objective. Practically: show the budget on the team dashboard, trigger a review when >50% consumed mid-period, and treat dependency-caused burn properly (counted if user-visible). Beware of gaming - SLIs must reflect user experience.
**Follow-up traps:** "Budget exhausted but critical feature deadline?" (explicit exception signed by a VP; risk accepted). "Too perfect SLO?" (100% is wrong target).

### 18.3 [Medium] Leadership wants a code freeze for the holiday season. Your view?
**What the interviewer is really testing:** Pragmatic risk reasoning.
**Typical mid-level answer:** "Agree - freeze everything." Or "Freezes are bad."
**Strong senior answer:** A freeze reduces change-induced risk in a critical window but it's blunt: changes pile up and release later becomes riskier, and needed fixes may be blocked. I'd propose a **targeted freeze**: no risky/feature/infra/schema changes during peak days; allowed: bug fixes for production, security patches and config changes via an expedited path with extra review and a named approver. Pre-peak, complete load testing, pre-scale, runbooks, and rollback verification; ensure on-call coverage and a war-room. After the window, release incrementally rather than as a big batch. Let the error budget and recent change failure rate inform how strict to be.
**Follow-up traps:** "Config changes during freeze?" (they're changes too; treat them as such). "Emergency approval?" (defined path, post-review).

### 18.4 [Hard] Describe a game day you would run for an order platform, and what you'd expect to learn.
**What the interviewer is really testing:** Practising resilience.
**Typical mid-level answer:** "Simulate an outage."
**Strong senior answer:** Define hypotheses and blast-radius controls first: "If Redis is unavailable, checkout stays above 99% success with higher latency." Choose scenarios from real risk: kill Redis, inject latency/throttling into the payment provider (Azure Chaos Studio or a proxy), fail over Azure SQL, expire a test secret, drop a Service Bus consumer, region failover. Run in staging first, then production with small scope and a stop button, with named roles (commander, observer, scribe) and on-call participating with no advance warning of details (but with approval). Observe: did alerts fire, was the runbook correct, did the circuit breakers and fallbacks behave, time-to-detect and mitigate, and was the DB protected. Output: action items, updated runbooks, and automation of detection gaps. Repeat regularly.
**Follow-up traps:** "Risk of production chaos?" (start small, abort conditions, business-hours, stakeholders informed). "Success measure?" (hypotheses validated and gaps found).

### 18.5 [Tricky] Production incidents keep slipping through even with 85% unit test coverage. Where are the testing gaps?
**What the interviewer is really testing:** Beyond coverage.
**Typical mid-level answer:** "Increase coverage to 95%."
**Strong senior answer:** Coverage measures executed lines, not behaviour under production conditions. Typical gaps: integration with real SQL (plans, locking, isolation) and message brokers, contract tests between services, configuration/environment validation, data-skew and volume performance tests, concurrency and idempotency tests, failure-mode tests (timeouts, retries, partial failure), migration tests on realistic data with old+new code, security tests, and tests of operational behaviour (health checks, alerts, rollback). I analyse the last N incidents and classify why tests didn't catch them, then invest where escapes concentrate. Add production safeguards (canary, flags) because no test suite covers everything.
**Follow-up traps:** "Mutation testing?" (measures test strength). "Testing in production?" (synthetic transactions, canaries).

### 18.6 [Scenario] You're asked to define a "production readiness review" for new services. What goes in it?
**What the interviewer is really testing:** Systematic prevention mindset.
**Typical mid-level answer:** A generic checklist of "tests, docs".
**Strong senior answer:** A short, risk-tiered checklist reviewed before launch: SLIs/SLOs defined with burn-rate alerts; dashboards and runbooks; structured logging + tracing with correlation; on-call ownership and escalation; capacity/load-test results with headroom; dependency map with timeouts, retries, circuit breakers and defined degradation; data: backup/PITR tested restore, migrations backward compatible, RPO/RTO documented; security: managed identity, least privilege, secrets in Key Vault, private networking, threat model; deployment: IaC, progressive rollout, rollback tested, feature flags; cost: budgets and caps; DR plan and a game day; support/escalation contacts. The PRR is a conversation with action items and a go/no-go, not a gate used to blame.
**Follow-up traps:** "Too heavy for small services?" (tiering by criticality). "After launch?" (periodic re-review).

**Red flags in answers**
- Relying on code coverage numbers as proof of safety.
- Freezes or approvals as the only risk control.
- No plan for rollback or progressive exposure.

**Cheat sheet**
- Small, frequent, reversible, observable changes.
- Progressive delivery: flags, canary, rings, bake time, auto-rollback.
- Error-budget policy agreed with product beforehand.
- Targeted freezes + expedited path for fixes.
- Game days & chaos with hypotheses and abort controls.
- Analyse incident escapes to find test gaps; PRR for new services.

---

## Topic 19: Six End-to-End Incident Scenarios

**Study notes**
- Use one skeleton for every scenario: **Stabilise -> Scope -> Hypotheses & signals -> Mitigate -> Fix -> Verify -> Prevent -> Communicate.** State assumptions out loud; the interviewer wants the reasoning order, not the "right" cause.
- All data in the scenarios is hypothetical; they're practice prompts, not claims about real events.

```
Skeleton (say it in this order)
1 Impact & severity   2 What changed?   3 Slice telemetry (version/instance/tenant/dependency)
4 Ranked hypotheses + discriminating test each   5 Mitigate (rollback/flag/scale/shed)
6 Root-cause fix with test   7 Verify with SLIs + backlog drained   8 Prevent (detect/mitigate/prevent) + postmortem
```

### 19.1 [Scenario] Black Friday: at 10:02 checkout latency climbs from 400ms to 12s and 8% of orders time out. Walk through it.
**What the interviewer is really testing:** Structured thinking under peak load and pressure.
**Typical mid-level answer:** "Scale out the web tier and check logs." Might shift load to the DB and ignore the real bottleneck.
**Strong senior answer:** *Stabilise/Scope:* Sev1 (revenue). Name an IC, open a bridge, post the first update. Check the last-hour change log (anything deployed? flags? campaign start?) and App Insights: latency and failure by operation, version, instance, region, plus the dependencies table. *Hypotheses and signals:* (a) DB saturation - Azure SQL CPU/log IO/workers (`dm_db_resource_stats`), Query Store top consumers, blocking chains; (b) connection pool exhaustion - "timeout obtaining connection" exceptions, connection counts across scaled-out instances; (c) thread pool starvation - low CPU, queue length rising; (d) third-party limits - payment/tax API throttling (429, duration up); (e) cache stampede/evictions - hit ratio drop, Redis server load; (f) autoscale lag - instance count vs demand. *Mitigate:* shed non-essential load via flags (recommendations, personalisation, heavy search), pre-scale or scale up the saturated tier (database first if it's the bottleneck - scaling web instances would add connections and worsen it), enable a waiting room/queue for checkout if needed, route static content to CDN, increase vendor limits. If one query is the hog, force the known-good plan or add the missing index under change control. *Fix/Verify:* verify p95 and success rate return to SLO and the backlog of pending orders is processed; compare before/after. *Prevent:* load test at 2x forecast with realistic skew, pre-scale schedules, vendor limit agreements, degradation switches, capacity dashboards, postmortem.
**Follow-up traps:** "Scaling out made it worse?" (DB connection limits/bottleneck shifted). "Order placed twice due to timeouts?" (idempotency on checkout). "What would you not do?" (deploy a risky fix during peak without rollback).

### 19.2 [Scenario] Support reports customers were charged twice for the same order. Walk through it.
**What the interviewer is really testing:** Money-related incident handling: containment, idempotency, reconciliation, customer remediation.
**Typical mid-level answer:** "Find the bug and refund them." Misses containment and scope.
**Strong senior answer:** *Stabilise:* this is Sev1 even at low volume - financial harm and trust. First stop further double charges: identify whether it's ongoing; if so disable the payment retry path/ the offending feature or put checkout in a safe mode (or pause the consumer/job). *Scope:* query payments for the same order with more than one successful capture; compare with provider records (settlement/ transaction API) as source of truth; determine start time (deploy/flag/provider incident) and affected segments. *Hypotheses:* client double-submit (no idempotency key; user clicks twice or mobile retries after timeout), server retry after timeout when the first attempt actually succeeded, message redelivery (lock lost/at-least-once) causing the capture handler to run twice, webhook processed twice, race between two instances, a deploy changed retry policy. Evidence: for duplicate payments check timestamps, instance ids, idempotency keys, Service Bus `DeliveryCount`, and `MessageLockLost` exceptions. *Fix:* idempotency key per order/attempt on provider calls (store before calling), unique constraint on (OrderId, successful capture), inbox/dedup on message id, handle timeout as Unknown with reconcile; fix the specific defect. *Repair:* refunds via the provider with finance approval and an audit log, customer communication (apology, expected refund time) coordinated with legal/finance. *Prevent:* daily reconciliation of internal ledger vs provider settlement, alert on multiple captures per order, test for duplicate delivery and concurrent requests.
```sql
SELECT OrderId, COUNT(*) AS captures, SUM(Amount) AS total
FROM dbo.Payments WHERE Status = 'Captured' AND CreatedAt >= @since
GROUP BY OrderId HAVING COUNT(*) > 1;
```
**Follow-up traps:** "Refund automatically?" (batch with review and idempotency, don't double refund). "Provider also says it's their bug?" (still own detection/reconciliation).

### 19.3 [Scenario] After a deploy, orders are stuck in "Processing" - no errors, queue length is rising. Walk through it.
**What the interviewer is really testing:** Silent failure and async pipeline reasoning.
**Typical mid-level answer:** "Restart the workers."
**Strong senior answer:** *Stabilise/scope:* confirm via oldest-message age and count of orders stuck by status and age; identify the start time (aligned with the deploy). Notify support that orders are delayed, not lost. *Hypotheses:* new consumer version isn't receiving (queue/subscription name or connection setting changed, managed identity role missing, filter rule changed); consumer receives but fails silently (exception swallowed, messages abandoned and re-delivered); consumer is slow (new dependency call, lock loss); message contract changed so older/newer versions mis-parse and dead-letter or ignore; a new state transition guard rejects orders; a migration locked or changed a table the worker uses; messages are going to DLQ. *Signals:* Service Bus metrics (active vs dead-letter, incoming/outgoing), consumer logs for startup and receive loop, App Insights `traces` for the handler, DLQ reasons, blocked sessions in SQL, Activity Log for permission/config changes. *Mitigate:* roll back the worker (messages stay safe in the queue) or fix the config; because messages are durable, no data loss as long as TTL hasn't expired - check TTL. *Recover:* let the backlog drain at a controlled rate (watch DB), reprocess DLQ after fix with idempotency; verify stuck orders reached final states; reconcile orders vs payments/shipping. *Prevent:* end-to-end smoke test posting a test message after deploy, alert on oldest-message-age and "processed count == 0", message contract versioning (tolerant reader), and DLQ alerts.
**Follow-up traps:** "TTL expiry caused message loss?" (check dead-letter on expiration setting). "Old version and new version both consuming?" (competing consumers with incompatible contract).

### 19.4 [Scenario] After release 7.2, memory climbs steadily and pods get OOMKilled every ~6 hours. Walk through it.
**What the interviewer is really testing:** Memory leak method with tools.
**Typical mid-level answer:** "Increase pod memory."
**Strong senior answer:** *Stabilise:* sawtooth of restarts causes intermittent errors during restarts; first mitigation is rollback to 7.1 if memory returns to flat (also confirms the release as the cause); if rollback is not possible, schedule rolling restarts every 4h and raise limits temporarily, and ensure readiness/graceful shutdown so restarts don't drop requests. *Diagnose:* split memory by version, correlate the ramp rate with request rate (leak per request) or with a specific endpoint/job. In a canary pod: `dotnet-counters` for `gc-heap-size` vs `working-set` and gen2/LOH size to see managed vs native; capture two `dotnet-gcdump`s 30-60 minutes apart and diff for growing types; `dotnet-dump` with `dumpheap -stat` and `gcroot` for the retention path. Typical release culprits: new static dictionary/ in-memory cache without eviction, event subscription never unsubscribed, `HttpClient` created per call, new logging scope/telemetry accumulating, closure capturing large objects, a `MemoryCache` without `SizeLimit`, EF context kept alive by a singleton, unbounded channel/queue in a background service. Look at the release diff for those patterns. *Fix:* correct retention (bounded cache with expiry, unsubscribe, scope correctly), test with a soak test and verify memory plateaus. *Prevent:* soak/stability test pre-release with memory growth check, memory-growth and OOM alerts, per-version dashboards, canary bake time covering at least one leak cycle.
```
dotnet-counters monitor -p <pid> System.Runtime --counters gc-heap-size,gen-2-gc-count,loh-size,working-set
dotnet-gcdump collect -p <pid> -o t0.gcdump   # repeat later, diff in VS/PerfView
```
**Follow-up traps:** "Working set grows but GC heap flat?" (native memory/fragmentation). "Limit and GC behaviour in containers?" (GC heap hard limit ~75% of container limit by default).

### 19.5 [Scenario] A deadlock storm: error 1205 jumps to hundreds per minute after a release; orders fail. Walk through it.
**What the interviewer is really testing:** Database concurrency diagnostics and mitigation.
**Typical mid-level answer:** "Add retries."
**Strong senior answer:** *Stabilise:* retries may mask but increase load; the quickest relief is rollback if the release correlates with the first 1205s. If not possible, throttle concurrency on the affected path (queue/semaphore by order id) or disable the feature flag that triggers it. *Diagnose:* capture deadlock graphs from `system_health` `xml_deadlock_report` (or Azure SQL equivalents): identify the two statements, objects and indexes, lock modes (e.g. key lock U vs S, range locks under serializable), and the victim. Compare with release diff: a new query touching tables in a different order, a missing index causing scans (more locks), larger transaction scope (an external call inside a transaction), an isolation-level change (`SERIALIZABLE`/TransactionScope default is Serializable in .NET Framework unless configured - `TransactionScope` defaults to Serializable; set ReadCommitted), or a changed batch size. *Fix:* access resources in a consistent order, add a covering index, shorten the transaction, use `UPDLOCK` for read-then-update, enable RCSI where appropriate, split batches; keep a retry with backoff on 1205 for the whole unit of work as defence in depth. *Verify:* deadlock count per minute, blocked-process report, p95 latency. *Prevent:* concurrency test with parallel requests in CI/perf env, alert on 1205 rate, code review checklist for transaction scope and lock order.
**Follow-up traps:** "TransactionScope default isolation?" (Serializable; specify `IsolationLevel.ReadCommitted`). "Retry safe?" (only if the operation is idempotent and retried as a whole).

### 19.6 [Scenario] At 06:00 every API call to a partner starts failing with 401, and a few minutes later the web app can't read configuration - Key Vault/certificate expiry suspected. Walk through it.
**What the interviewer is really testing:** Time-bomb failures and dependency chains.
**Typical mid-level answer:** "Renew the cert."
**Strong senior answer:** *Stabilise:* Sev1; confirm "what changed": nothing deployed, but failure begins at an exact time - the signature of expiry. *Scope:* identify what's failing: partner 401 (expired client secret/cert used for client-credential auth), app failing to resolve Key Vault references (managed identity or vault access/firewall problem, or secret/cert version disabled/expired), TLS errors on inbound (expired server cert). Check Key Vault `AuditEvent`/diagnostics, secret/cert `expires` attributes, Entra app registration credential expiry (`AADSTS7000222`), App Service config blade showing Key Vault reference status, and exceptions in App Insights (`AuthenticationException`, "remote certificate is invalid"). *Mitigate:* create/renew the credential or certificate (new version in Key Vault, update partner with the new public cert/secret if needed), then refresh consumers: restart the app or the Key Vault reference refresh (App Service re-resolves references periodically and when settings change), re-issue tokens, purge caches of failed tokens. If the partner needs to register a new cert (lead time), use an emergency contact path - an action item to avoid next time. *Verify:* auth success rate, dependency success, synthetic test; confirm no other expiries in the next 30 days (audit all certs/secrets). *Prevent:* managed identity/federated credentials instead of secrets; Event Grid near-expiry events for Key Vault (30/14/7 days) routed to the owning team; certificate and secret inventory with owners; auto-rotation; synthetic check on cert validity; runbook rehearsal in a game day; calendar/ticket for partner-owned credentials.
```bash
az keyvault secret list --vault-name <kv> --query "[?attributes.expires!=null].{n:name,exp:attributes.expires}" -o table
az keyvault certificate list --vault-name <kv> --query "[].{n:name,exp:attributes.expires}" -o table
az webapp config appsettings list -g <rg> -n <app> --query "[?contains(value,'@Microsoft.KeyVault')]"
```
**Follow-up traps:** "Key Vault reference shows unresolved?" (identity, access policy/RBAC, firewall/private endpoint, disabled version). "Auto-renew didn't happen?" (CA integration/ policy not configured).

**Red flags in answers**
- Scaling or restarting before understanding which tier is the bottleneck.
- Treating money/data incidents as ordinary bugs without containment and reconciliation.
- No verification step or prevention step in the walk-through.

**Cheat sheet**
- Same skeleton every time: stabilise -> scope -> hypotheses -> mitigate -> fix -> verify -> prevent -> communicate.
- Peak load: shed non-essential, scale the real bottleneck, protect DB.
- Double charge: idempotency key + unique constraint + reconcile with provider + refund process.
- Stuck orders: oldest message age, DLQ reasons, contract/config/identity changes.
- Leak: rollback to confirm, two gcdumps diff, gcroot.
- Deadlocks: graph, lock order, indexes, short transactions, isolation level, retry.
- Expiry: exact-timestamp failure; inventory + near-expiry alerts + managed identity.

---

## Topic 20: Blame, Pressure & Telling Your Production Story (incl. "Worst Incident You Caused")

**Study notes**
- Interviewers use incident stories to evaluate **ownership, judgement under pressure, communication, learning and scope of influence**. Seniors show *systemic* impact (what changed for the team) and honest reflection - not hero narratives.
- **Never invent a story.** Use only real ones; if you lack an exact number, say "approximately" or describe the measurement method. Pick incidents where you had a meaningful role and can explain the reasoning in depth. Avoid confidential details: anonymise customer names and numbers if required.
- **STAR+ template for production stories** (fill-in; keep to ~2-3 minutes):
  - **Situation:** system, scale, your role, why it mattered (users/revenue/SLO). _[Fill in: service, customers affected, your role]_
  - **Task:** what you were accountable for in that moment. _[Fill in: IC? ops lead? on-call?]_
  - **Action (in order):** detection -> stabilise -> diagnose (hypotheses, tools, signals) -> fix -> comms -> prevent. _[Fill in: the 3 key decisions and why you made them, including what you ruled out]_
  - **Result:** metrics - time to detect, time to mitigate, impact (users/orders/$), recurrence. _[Fill in: TTD / TTM / TTR, % users, $ or SLO budget]_
  - **Reflection:** what you'd do differently; what changed in the system/team afterwards. _[Fill in: the action items completed and their measurable effect]_
- What makes it **senior:** you prioritised mitigation over curiosity; you organised others (roles, comms) not only fixed code; you used data to decide; you considered trade-offs and risks; you drove **system-level fixes** and verified them; you shared learnings (postmortem, docs, game day); you protected people (blameless); you can say what you got wrong.
- **Blame and pressure:** separate people from problem; offer facts + options; keep calm and transparent; communicate uncertainty; don't over-promise; escalate appropriately; push for postmortem actions rather than scapegoats; protect the team from executive churn; accept your own part.
- **"Worst incident you caused" answer shape:** own it plainly (no deflection or minimising), explain the context and why it was a reasonable decision at the time *without excusing*, how you detected and responded (you flagged it immediately), impact honestly sized, what you changed (personal habit + system guardrail), and how it made you better. Interviewers penalise "I never caused one" or blaming others.

```
Story arc (2-3 min):  Situation(15s) -> Task(10s) -> Actions with decisions(60-75s) -> Result with numbers(20s) -> Reflection(30s)
Prepare 3 stories: (1) high-severity incident you led, (2) a diagnosis puzzle you solved, (3) one you caused or got wrong
```

### 20.1 [Easy] Tell me about a production incident you handled. (STAR template)
**What the interviewer is really testing:** Real experience, structure, ownership, learning.
**Typical mid-level answer:** A chronological tale about the bug's technical detail with no decision reasoning, no metrics, and no learning beyond "we fixed it".
**Strong senior answer:** Use the template: *Situation:* "[service] handling [scale] had [symptom] affecting [who]; I was [role]." *Task:* "I was responsible for [mitigation / coordination / diagnosis]." *Action:* give 3 decisions with reasoning - e.g. "[I chose to rollback rather than fix forward because ...]", "[I ruled out X by checking signal Y]", "[I set up roles/comms cadence of N minutes]". *Result:* "[TTD, TTM, impact]". *Reflection:* "[what we changed: alerting, test, guardrail; its measured effect]". Say what you'd do differently. Keep the technical depth available for follow-ups rather than leading with it. Fill each bracket with real facts from your own history - if you cannot recall a number, describe how it was measured or give an honest range.
**Follow-up traps:** "What was your specific contribution vs the team's?" (be precise; use 'I' for your actions). "What would you do differently?" (have a genuine answer).

### 20.2 [Follow-up] What makes an incident story "senior" rather than "mid-level"?
**What the interviewer is really testing:** Self-awareness about seniority.
**Typical mid-level answer:** "I fixed a very hard bug."
**Strong senior answer:** A mid-level story centres on solving a hard technical problem. A senior story shows: judgement about priorities (stabilise before root-cause), leadership of others (roles, communication, relieving tired colleagues), use of data and hypothesis elimination, risk trade-offs (rollback vs fix-forward, what was explicitly *not* done), communication to stakeholders in impact terms, and, crucially, **systemic follow-through**: postmortem with contributing factors, actions completed, and measurable reduction in recurrence (e.g. fewer pages, faster detection). It also includes influence beyond your team (shared runbook, template, practice game day) and honest reflection on mistakes.
**Follow-up traps:** "No big incident experience?" (use a near-miss or a smaller incident, show the same behaviours). "Heroics?" (heroics are a red flag; prevention is the point).

### 20.3 [Medium] Leadership is pressing for a fix ETA you can't give. What do you say?
**What the interviewer is really testing:** Honest communication under pressure.
**Typical mid-level answer:** Gives an optimistic ETA to calm things down.
**Strong senior answer:** I'd explain what we know, what we don't, what we're doing, and when the next update is - ETA given as a confidence range tied to what we're verifying ("if the rollback restores service, within 20 minutes; if not, we move to X and I'll reassess at 15:30"). I'd avoid false precision because a missed ETA damages trust more than uncertainty. I'd ask what decisions they need to make (customer comms, support staffing, pausing campaigns) and give options with trade-offs so they can act. I keep the engineers insulated: updates go through one channel and one person.
**Follow-up traps:** "They insist on a single number?" (give a range with the conditions). "Customer threatens to leave?" (executive-to-executive contact, sincere updates, credits after).

### 20.4 [Hard] Tell me about the worst production incident you caused. (STAR template, no fabricated content)
**What the interviewer is really testing:** Accountability, humility, learning, and systems orientation.
**Typical mid-level answer:** Minimises ("it wasn't really my fault"), blames process/others, or claims "I've never caused an outage".
**Strong senior answer:** Choose a real incident where you were genuinely involved and that was moderate-severity (not so catastrophic it raises questions of judgement, not so trivial it seems evasive). Structure: *Situation:* "[the change I made and its intent]". *What went wrong:* "[the technical mechanism; own it directly: 'I did X; I missed Y']". *Detection/response:* "[how quickly you flagged it yourself, how you stabilised (rollback/flag), how you communicated]". *Impact:* "[honest numbers: users, duration, $ if known]". *Why it was possible:* "[the missing safeguard: test, review, alert, canary, migration practice]" - this is explanation not excuse. *What changed:* "[personal habit + system guardrail you built or drove, and how you verified it works]". *Learning:* "[what you now do automatically]". Fill every bracket with your real story; do not copy examples. Typical honest categories people draw from: a migration without lock timeout, a config change applied to the wrong environment, a retry loop that amplified load, a missing idempotency check, a secret that expired. Choose only one that is truly yours.
**Follow-up traps:** "Who found out first?" (best answer: you, and you raised it immediately). "Was anyone blamed?" (describe blameless handling or what you'd do). "Would you do it differently today?" (specific behaviour/guardrail).

### 20.5 [Tricky] A junior engineer's change caused an outage and your manager wants to know "who did it". What do you do?
**What the interviewer is really testing:** Protecting people while ensuring accountability.
**Typical mid-level answer:** Names the junior or stays silent.
**Strong senior answer:** I take shared ownership: the change passed our review, tests and deployment process, so the system allowed it. I give the manager facts about what happened and what fix is in progress; I explain that the lasting question is "why could this reach production" and present contributing factors and actions. If they need a name for the timeline, it's factual and neutral ("a change merged by the platform team at 14:02"), and I frame the learning at the team level. Privately I support the junior: walk through what happened without shaming, thank them for fast disclosure, and pair on the guardrail fix (e.g. adding a test or review checklist) - turning the incident into growth. I'd also speak up if leadership pushes for punishment, since fear reduces reporting.
**Follow-up traps:** "Manager insists on discipline?" (separate performance management from incident review; share data). "Senior reviewers also approved?" (shared accountability).

### 20.6 [Scenario] A major customer's executive is angry after an outage and wants a call with the engineering lead. How do you prepare and run the call?
**What the interviewer is really testing:** Customer-facing incident leadership.
**Typical mid-level answer:** Defensive technical explanation.
**Strong senior answer:** Preparation: gather facts with account/support leads: timeline, impact on *their* tenant (not just global), what we've done, what we will do. Agree an owner for follow-ups and what can and cannot be committed (credits/SLA terms are for account management/legal). On the call: start with acknowledgement and apology for the impact, then concise facts (what happened, how long, what's fixed), then specifics on prevention with dates, then listen - ask what mattered most to them and what they need now (reports, data repair, escalation contact). Avoid blaming vendors/individuals, speculation or technical overload. Follow up in writing within a day, deliver the postmortem summary (customer-appropriate), and report progress on actions until closed.
**Follow-up traps:** "They demand root cause on the call but analysis isn't done?" (state what's confirmed, and when the written RCA arrives). "Legal concerns?" (consult before committing).

**Red flags in answers**
- "I've never caused an incident" or blaming others for the one you caused.
- Stories with no metrics, no decisions explained, and no system-level follow-up.
- Naming and shaming individuals or giving false ETAs under pressure.

**Cheat sheet**
- Prepare 3 real stories: led-incident, hard-diagnosis, caused-by-me; never fabricate.
- STAR+: Situation, Task, Actions (decisions + reasoning), Result (TTD/TTM/impact), Reflection (what changed).
- Senior = priorities + leadership + data + trade-offs + systemic fixes + sharing learnings.
- Under pressure: facts, unknowns, actions, next-update time; ranges not false ETAs.
- Worst-incident answer: own it, size it honestly, explain the missing safeguard, show the change.
- Protect people, keep accountability for actions.
