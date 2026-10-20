# 10. Event Sourcing: history becomes the source of state

Status: initial chapter; event-fold example, event-store implementation pending.

## The problem and the cost

For a deliberately small exercise, a reservation aggregate needs to explain how
it reached its current state. Event Sourcing persists its accepted changes as an
ordered event stream and derives current state by applying those events.

It is not merely publishing `OrderPlaced` after updating an Orders table. It is
also not automatically a useful audit trail for every access or administrative
action. S16 describes the pattern and its additional complexity.

Do not convert the entire reference application. Start with one aggregate and
justify the historical/reconstruction requirement.

## A tiny state fold

**Standalone teaching fragment; not an event store or command handler:**

```csharp
record ReservationState(int Quantity, bool Released);
abstract record ReservationEvent;
record Reserved(int Quantity) : ReservationEvent;
record Released : ReservationEvent;

static class ReservationFold
{
    public static ReservationState Apply(
        ReservationState state, ReservationEvent @event) => @event switch
    {
        Reserved e => state with { Quantity = e.Quantity },
        Released => state with { Released = true },
        _ => throw new InvalidOperationException("Unsupported reservation event.")
    };
}
```

The fold illustrates applying recorded facts. It does not validate commands,
serialize events, handle multiple reservations, or persist anything.
Command validation must reject invalid state transitions before appending new
facts. Replay should not call a payment provider or send emails.

## Append with an expected version

**Storage-contract pseudocode:**

```text
Load stream "reservation-123" through version 7
Decide whether Reserve/Release command is valid
Produce proposed events
Append events only if current version is still 7
```

If another writer appended first, reject or reload and re-evaluate. Do not simply
retry an old decision without examining the new facts.

The append operation needs atomic expected-version enforcement. A normal
`SELECT MAX(version)` followed by unrelated inserts is racy unless the storage
design protects it.

## Domain events and integration events

The internal stream may contain details that external consumers should not
depend on. Define public integration contracts separately where appropriate.
Do not assume every event-store implementation atomically publishes to your
message broker. The publication/recovery problem still needs an answer.

## Replay and snapshots

Replay folds events into state. Keep it deterministic with respect to recorded
facts: do not use today's exchange rate, random value, or wall-clock decision to
reinterpret an old accepted event.

A snapshot can reduce the number of events applied when loading. It is a
performance aid with version/compatibility requirements, not a replacement for
an authoritative history under the chosen design. A corrupted snapshot should
have a defined discard/rebuild path.

## Event evolution

Suppose `ReservedV1` had only quantity, while a later contract needs a warehouse.
Do not invent a historical warehouse without a business rule.

Compare:

- New event types/versions for new facts.
- Upcasters that transform old representations at read time.
- Explicit migration with validation and rollback planning.

Each has costs. Preserve the meaning of history. Renaming a CLR class and
assuming old serialized data will still load is not a versioning strategy.

Avoid putting unnecessary personal/sensitive data into an append-oriented history.
Retention and deletion requirements need specialist review for the actual
deployment; this chapter is not legal advice.

## Failure lab specification

1. Append events with expected version.
2. Race two valid commands and verify only a permitted append wins.
3. Rebuild state from the beginning and compare it with the command result.
4. Load with/without a snapshot and compare outcomes.
5. Read a mixed old/new event stream.
6. Replay twice without triggering external effects.

## Debugging and improvement

| Symptom | Inspect |
|---|---|
| Replay differs from original state | Non-determinism, missing events, changed apply logic |
| Concurrent commands both win incorrectly | Atomic append/version enforcement |
| Old stream will not deserialize | Stable event names, schema versions, converters |
| Duplicate emails on rebuild | Side effects embedded in replay |
| Long load time | Stream length and measured snapshot strategy |

## Interview questions and answers

**Q: Event Sourcing vs. an audit table?**

A: In Event Sourcing, the events are authoritative state. An audit table commonly
records changes alongside another authoritative representation.

**Follow-up: Can an event-sourced system still have relational tables?**

Yes, for projections or other purposes. Authority and rebuild rules must remain
clear.

**Q: How do you correct a mistake?**

A: Often by appending a new corrective business fact, with explicit semantics.
Operational data repair is a separate controlled process, not casual editing.

**Follow-up: Does append-only mean no privacy concerns?**

No. Data design and lifecycle obligations still apply.

**Q: Why does a snapshot carry a stream version?**

A: So loading knows which subsequent events to apply and can reason about the
snapshot's position/compatibility.

**Follow-up: Can you discard all earlier events after snapshotting?**

Not while assuming those events remain the authoritative replayable history.
Any compaction design needs its own explicit guarantees.

## Challenge

Design the smallest reservation stream, its valid commands, expected-version
append contract, and one schema change. Explain why plain relational state plus
an audit trail might still be a better choice.

References: S15-S16.
