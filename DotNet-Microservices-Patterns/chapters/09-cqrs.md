# 9. CQRS: separate decisions from presentation

Status: initial chapter; baseline read/write endpoints exist, but a dedicated
CQRS implementation and asynchronous projection lab are pending.

## The problem

An order's write model must protect lifecycle rules. A dashboard wants customer
display data, payment summary, and a compact stock status. Forcing both concerns
into the same shape can create unnecessary complexity.

CQRS separates the command model from the query model. Separate databases,
brokers, and Event Sourcing are optional design choices, not the definition.
S15 describes these distinctions.

## Start with the smallest useful split

For the first extension of our project:

```text
Command: ChangeOrderNote -> validate -> update -> commit
Query:   GetOrderDetails -> project -> return DTO
                  same Orders database
```

The existing lab uses different HTTP operations and a DTO, but we do not claim
that naming endpoints alone constitutes a full CQRS architecture.

**EF-shaped query excerpt; not a standalone runnable program:**

```csharp
var result = await db.Orders
    .AsNoTracking()
    .Where(order => order.Id == id)
    .Select(order => new OrderDetails(
        order.Id, order.Sku, order.Quantity, order.Note, order.Version))
    .SingleOrDefaultAsync(ct);
```

`OrderDetails` is a proposed query DTO, not an extra entity. This excerpt avoids
loading a domain object merely to serialize every tracked property.

## When to add a separate read store

Only after a requirement justifies it: a denormalized screen, a different query
workload, independent read scaling, or data from several owners under explicit
contracts.

For this book, propose an `OrderSummary` projection fed by integration events:

```text
OrderId | OrderStatus | PaymentStatus | ReservationStatus | LastUpdatedUtc
```

Its value is presentation, not authority to mutate Payments or Inventory.
Projection freshness and data ownership still matter.

## A projection is an operational component

Specify:

- Stable event identity and consumer deduplication.
- Per-aggregate ordering/version rules where needed.
- Checkpoint ownership and atomic update boundaries.
- Poison-message handling and lag visibility.
- A rebuild source and retention sufficient to use it.

An outbox is not automatically a permanent replay log. If published rows are
deleted and no retained source can reproduce history, you cannot assume a
deleted projection can be rebuilt.

## Read your writes

The user creates an order and immediately opens its summary. An asynchronous
projection may not have processed it yet.

For the exercise, compare:

| Option | Decision to examine |
|---|---|
| Return a command result with current local state | Useful immediate response without pretending the projection caught up |
| Return operation ID and show pending status | Honest asynchronous UX |
| Read authoritative write state for that screen | Additional query path and consistency semantics |
| Wait for a known projection version with a deadline | Extra coordination and bounded waiting |

Do not sleep an arbitrary 500 ms and call that a consistency guarantee.

## Failure lab specification

1. Pause the projection consumer.
2. Create and update an order.
3. Observe the old summary with an explicit freshness signal.
4. Resume processing and assert convergence under the test's delivery assumptions.
5. Deliver duplicate and out-of-order events.
6. Rebuild into a new projection version, validate it, and switch readers.

Define how version gaps are repaired. Silently dropping an earlier event can be
wrong if the later event is a delta rather than a full-state snapshot.

## Pitfalls and debugging

| Symptom | Inspect |
|---|---|
| Write succeeds, read appears missing | Projection lag/checkpoint, not just write transaction |
| Summary regresses | Event ordering and version handling |
| Rebuild cannot reproduce state | Replay-source retention and event contract |
| UI shows impossible "confirmed + declined" | Cross-owner timing and summary semantics |
| Too many abstractions, little benefit | Whether simple CRUD/query DTOs would suffice |

Track processed version/checkpoint and lag separately from HTTP success rate.
Projection rebuilds need compatibility rules and safe cutover, not merely
"delete the table and replay."

## Interview questions and answers

**Q: Does CQRS require two databases?**

A: No. You can separate read/write models using one database. Separate stores
introduce synchronization and consistency work.

**Follow-up: Does it require MediatR?**

No. A dispatch library can organize code but does not define the architecture.

**Q: How is CQRS different from Event Sourcing?**

A: CQRS concerns read/write models. Event Sourcing concerns the authoritative
representation of state as events. Either can be used without the other.

**Follow-up: Can an ordinary relational write model feed a projection?**

Yes, using an appropriate change/publication contract such as an outbox.

**Q: How do you rebuild a projection?**

A: From an identified retained source, using versioned transformation logic,
checkpoints, validation, and a safe reader cutover.

**Follow-up: Is the broker always that source?**

No. Broker retention and replay semantics must actually support the requirement.

## Challenge

Implement same-database query DTOs first. Write down the requirement that would
justify a separate read store, including the maximum acceptable staleness and
how the user sees pending state.

References: S13, S15.
