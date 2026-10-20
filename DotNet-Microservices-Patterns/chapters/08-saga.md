# 8. Saga: a durable business process, not remote rollback

Status: initial chapter; state-machine specification, executable durable Saga
pending. The starter lab creates drafts only.

## The problem

Checkout spans Orders, Inventory, and Payments. Each owner can commit locally,
but the overall business process can stop halfway.

For the proposed capstone, a Saga tracks that process and requests compensating
actions where the business permits them. Compensation is new work with its own
failure modes, not an ACID rollback across databases. See S14.

## Choreography or orchestration

| Choice | Exercise tradeoff |
|---|---|
| Choreography: services react to events | Less central workflow code, but the end-to-end process can become harder to reconstruct |
| Orchestration: a coordinator records transitions and issues commands | Workflow is explicit, but the coordinator and its persistence become operational responsibilities |

For teaching, start with orchestration because the transition table is visible.
This is a project choice, not a rule that choreography is inferior.

## First transition table

**Design specification, not implemented code:**

| State | Accepted input | Local transition and intended command |
|---|---|---|
| Started | Checkout requested | AwaitingStock; request reservation |
| AwaitingStock | Stock reserved | AwaitingPayment; request authorization |
| AwaitingStock | Stock rejected | Rejected |
| AwaitingPayment | Payment authorized | Confirmed |
| AwaitingPayment | Payment definitively rejected | ReleasingStock; request release |
| ReleasingStock | Release confirmed | Rejected |
| ReleasingStock | Repeated release failures | Remain recoverable; escalate by policy |

A payment timeout is deliberately absent from the "definitively rejected" row.
It is uncertain, not necessarily declined. Define a reconciliation state and
query/provider contract before finalizing this machine.

## Durable transition shape

**Transaction pseudocode:**

```text
BEGIN
  deduplicate incoming message
  load Saga and verify expected state/version
  apply permitted transition
  persist next command to local outbox
COMMIT
acknowledge incoming message
```

The Saga state and outgoing command intent should share the selected local
transaction. Otherwise the state can advance while the next command is lost.

Correlation identifies the process. Separate operation/message identities
recognize duplicates of a particular step. Add optimistic concurrency so two
events cannot silently overwrite the same Saga transition.

## Compensation is business-specific

If stock was reserved, releasing that reservation can be a compensation.
If payment was authorized, voiding the authorization may be appropriate.
If money was captured, refunding may be a different process with fees, timing,
and customer-visible effects.

Do not implement compensation as `catch { undoEverything(); }`. A process can
crash without running the catch, and the compensating call can also fail.

Define reservation expiry, who owns it, how renewals work if needed, and how late
success messages interact with expiry or cancellation.

## Deadlines and late messages

Use durable deadlines or a durable scheduler. `Task.Delay` inside one request
does not survive a process restart.

Consider:

```text
Payment request sent
Saga deadline expires
Payment success arrives late
```

Decide whether to reconcile, compensate, or escalate. Blindly ignoring the late
success can leave money and order state inconsistent. Blindly confirming can
resurrect a canceled order.

## Debugging and improvements

Investigate a stuck order by querying Saga ID, current state/version, last input,
pending outbox messages, operation identities, deadline, and compensation
attempts. A single "checkout failed" log is not enough.

| Failure | Recovery requirement |
|---|---|
| Coordinator restart | Durable state and message redelivery/recovery |
| Duplicate result | Deduplication and valid state transition |
| Conflicting concurrent inputs | Version check and re-evaluation |
| Compensation service unavailable | Durable retry/escalation |
| External status uncertain | Reconciliation rather than invented rejection |

## Interview questions and answers

**Q: Why not wrap all three services in a database transaction?**

A: Their independent databases and external effects do not automatically share
one transaction. Saga models business progress and recovery instead.

**Follow-up: Is Saga equivalent to eventual consistency?**

No. A Saga is a workflow pattern; eventual consistency describes consistency
behavior. One does not automatically implement every requirement of the other.

**Q: What makes a Saga durable?**

A: Persisted state, atomic transition/outgoing-intent handling, recoverable
delivery/scheduling, concurrency control, and defined recovery paths.

**Follow-up: Is a singleton coordinator sufficient?**

No. In-memory state disappears on restart and does not establish safe concurrency.

**Q: Can compensation fail?**

A: Yes. It must be tracked and retried or escalated according to business policy.

**Follow-up: When is manual action acceptable?**

Where automation cannot safely decide and there is an explicit operational queue,
owner, evidence, and procedure. "Someone will inspect logs" is not a procedure.

## Challenge

Extend the transition table for cancellation, duplicate stock results, late
payment authorization, and expired reservations. Explain which inputs are
ignored safely and which require a new business action.

References: S13-S14.
