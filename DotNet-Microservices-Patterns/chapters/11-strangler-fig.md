# 11. Strangler Fig: migrate a capability without a big-bang rewrite

Status: initial chapter; migration design and routing sketch, executable legacy
application/cutover lab pending.

## The problem

An existing application handles ordering, billing, and reporting. A rewrite
cannot replace it in one deployment. The project needs a sequence of changes
that preserve the existing client contract while moving selected capabilities.

Strangler Fig incrementally redirects functionality to a replacement until the
chosen legacy capability can be retired. S17 is the official pattern reference.
The correct term is **Strangler Fig**, not "Stranger Fig."

## Choose a bounded slice

For the exercise, first move an order-status query, not payment capture.

Define:

- Which callers and routes belong to the slice.
- Which owner is authoritative for the underlying data.
- How old and new contracts map.
- How correctness is compared.
- How routing is switched and rolled back.

Read-only migration can simplify an initial rehearsal, but it does not solve
the later write-ownership cutover.

## Routing sketch

**Illustrative gateway configuration fragment, not the runnable lab config:**

```json
{
  "Routes": {
    "new-order-status": {
      "Order": 0,
      "ClusterId": "new-orders",
      "Match": { "Path": "/orders/{id}/status" }
    },
    "legacy-fallback": {
      "Order": 100,
      "ClusterId": "legacy",
      "Match": { "Path": "/{**catch-all}" }
    }
  }
}
```

This fragment omits the `ReverseProxy` wrapper and cluster destinations. It
illustrates prioritizing a narrow route before a fallback, not a complete
deployment. Contract/path transforms may be needed for the actual applications.

## The hardest question is often data

If both systems can write the same business record with independent rules,
routing alone does not prevent divergence.

For a possible migration:

1. Keep legacy authoritative while backfilling the new query model.
2. Capture subsequent changes with a defined synchronization mechanism.
3. Compare old/new responses under known timing differences.
4. Route selected read traffic to the new implementation.
5. Rehearse write-ownership transfer as a separate operation.

Specify the synchronization watermark and backfill/change-capture race.
Copying a table once while writes continue is not a complete migration.

## Anti-corruption layer

Translate legacy concepts at the boundary instead of copying its internal data
model everywhere. For example, a legacy numeric status might map to a new
explicit state vocabulary. Unknown values should be visible failures or
explicitly represented states, not silently mapped to "confirmed."

Keep the translation narrow and owned. A permanent compatibility layer can be
valid, but retire temporary paths once no supported client needs them.

## Rollback has a data dimension

Routing traffic back is only safe if the old system understands state created
by the new path. Ask whether new writes, new identifiers, or changed invariants
can be consumed by legacy.

A feature flag does not undo a schema change or external side effect. Define
rollback limits before cutover.

## Debugging and improvements

| Symptom | Inspect |
|---|---|
| Some clients see old behavior | Route priority, deployment/config version, client cohort |
| Old/new totals differ | Snapshot timing, mapping, missed changes |
| Rollback routes work but data breaks | Backward compatibility and write ownership |
| Shadow traffic changes state | Unsafe replay of writes or external effects |
| "Temporary" adapter lasts forever | Retirement criteria and remaining callers |

Shadow only operations proven safe to duplicate, or isolate their effects.
Do not replay a production payment command into two live implementations.

## Interview questions and answers

**Q: How is this different from deploying a rewritten service?**

A: The pattern describes controlled coexistence, incremental replacement, and
retirement rather than only the existence of a new service.

**Follow-up: Does every migration need a gateway?**

No. The interception/routing mechanism can be elsewhere if it gives the required
control and compatibility.

**Q: Why not let old and new systems dual-write?**

A: Two independent writes can partially fail or diverge. Establish authority and
a recovery/synchronization contract.

**Follow-up: Is dual-write always forbidden?**

No, but any chosen design must explain atomicity gaps, reconciliation, and
conflict resolution rather than assuming both writes succeed.

**Q: What makes a rollback plan credible?**

A: Tested routing reversal plus data and contract compatibility for states that
exist after cutover.

**Follow-up: When might rollback be impossible?**

After an irreversible or unsupported state change; then forward recovery may be
the safer plan and must be designed in advance.

## Challenge

Write a migration runbook with entry criteria, cutover steps, checks, stop
conditions, rollback limits, and retirement criteria. Execute it against test
data before a real migration.

References: S07, S17.
