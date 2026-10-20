# 13. Capstone: prove the design under failure

Status: assessment and exercise specification. Edition 0.1 does not implement
the complete checkout system described here.

## Assignment

Extend the initial draft-order application into a checkout workflow. Use the
patterns that solve identified requirements; justify omissions. Keep the
event-sourced aggregate, legacy migration, and sidecar as focused branches/labs
where combining them would obscure the main invariants.

Proposed business rules:

- Stock cannot become negative.
- A repeated payment operation must not cause a duplicate charge.
- Committed orders retain publication intent.
- Workflow state survives process restart.
- Rejected checkout releases its reservation under a documented recovery policy.
- A displayed summary represents freshness/uncertainty honestly.

These are exercise requirements. Define the exact payment-provider contract,
database guarantees, delivery assumptions, and failure model before claiming
that your implementation satisfies them.

## Required artifacts

1. Architecture diagram with data owners and synchronous/asynchronous boundaries.
2. Sequence diagram for success, decline, timeout, and compensation failure.
3. Design decisions comparing simpler alternatives.
4. Runnable environment with deterministic fault injection.
5. Automated assertions and captured incident evidence.
6. Recovery runbook with ownership and manual escalation.

No production traffic or credentials are needed.

## Failure matrix

| Scenario | Required observable evidence |
|---|---|
| Broker unavailable at order commit | Order and outbox intent are durable; publication recovers after restoration |
| Consumer receives duplicate reserve command | One reservation effect for the logical operation |
| Relay dies after publish before marking | Duplicate publication is tolerated without a duplicate business effect |
| Two buyers compete for the last item | Inventory invariant holds under the specified isolation/concurrency design |
| Payment response lost after authorization | Outcome is reconciled, not blindly declared declined |
| Coordinator restarts between steps | Persisted progress is recovered without losing the next command |
| Release compensation fails | Durable pending recovery and visible escalation |
| Inventory is slow under load | Configured admission bound and a measured unrelated-endpoint latency target |
| Projection falls behind | Lag observable and user contract remains honest |
| Projection deleted | Rebuild from identified retained source with validated result |
| Event schema changes | Old history still loads under the declared compatibility strategy |
| New route rolled back | Contract and data compatibility survive the rehearsed reversal |
| Sidecar/backend unavailable | Bounded resource use and defined application behavior |

Set load, duration, dataset, percentile, and thresholds BEFORE running performance
experiments. "It seems fast" is not a test result.

## Debugging playbook

Start with one order, not a wall of unfiltered logs:

```text
Order ID
  -> workflow state/version
  -> operation/message IDs
  -> local durable records
  -> outbox/inbox/checkpoint records
  -> dependency attempts and responses
  -> pending timer/reconciliation/compensation
```

For each hop ask what the acknowledgement proves. An HTTP 202, a successful
database commit, and a broker confirmation establish different facts.

## Worked interview case 1: duplicate payment

**Prompt:** a customer sees two charges after a timeout. Where do you start?

**Answer outline:** identify logical operation keys and provider operation IDs;
compare original/retry requests and payloads; distinguish two operations from
two responses; inspect local deduplication and provider idempotency; reconcile
the external ledger before issuing corrective actions. Disable unsafe retry
paths if appropriate, but do not blindly refund without identifying the charges.

**Follow-up:** would an outbox alone fix it?

No. It preserves local intent; external payment effects require their own
idempotency and reconciliation contract.

## Worked interview case 2: order saved, event missing

**Prompt:** Orders has a record, but Inventory never reserved stock.

**Answer outline:** inspect outbox intent, relay claim/attempts, broker
acknowledgement, routing/consumer availability, inbox records, and poison-message
handling. Do not assume the event was lost merely because the final effect is
absent.

**Follow-up:** can you republish manually?

Only with stable message/operation identity, known consumer behavior, and an
auditable procedure. Inventing a new identity can bypass deduplication.

## Worked interview case 3: stale CQRS screen

**Prompt:** create returned success, but the dashboard says "not found."

**Answer outline:** establish which store the response queried, inspect
projection lag/checkpoints, then verify event publication and processing.
Implement the agreed pending/read-your-writes UX rather than adding arbitrary
sleeps or pretending data is immediately synchronized.

**Follow-up:** should all reads go to the write store?

That may solve a particular requirement, but it changes the query architecture.
Choose based on consistency, load, and the screen's contract.

## Self-assessment rubric

Score each area from 0 to 3:

| Area | 0 | 1 | 2 | 3 |
|---|---|---|---|---|
| Mechanism | Cannot explain | Names components | Explains normal path | Explains crash/duplicate/concurrency paths |
| Implementation | No working code | Happy-path demo | Automated failure checks | Independent implementation with clear limits |
| Judgment | Adds every pattern | Gives generic benefits | Compares alternatives | Chooses minimal design under explicit constraints |
| Debugging | Guesses | Searches logs | Reconstructs one operation | Produces evidence and a safe recovery procedure |
| Communication | Uses slogans | Defines terms | Explains tradeoffs | Defends decisions and changes them when evidence changes |

This rubric is for learning, not a hiring or title certification. Revisit low
areas with a new failure scenario rather than memorizing more definitions.

## Where this should take an experienced .NET developer

After reading, aim to discuss the patterns accurately. After implementing the
labs, aim to build and test their mechanisms. After completing this capstone
independently, aim to defend a small distributed architecture and troubleshoot
the specified failure classes.

Real production responsibility adds incidents, migrations, workload tuning,
security review, cost constraints, and organizational ownership. No finite book
can honestly promise those experiences merely through reading.
