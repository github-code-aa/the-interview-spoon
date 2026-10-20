# 1. Database per service: ownership before storage

## The problem

Suppose Orders reads Inventory's tables directly. A renamed column, a new stock
representation, or an access-policy change can break Orders without any API
contract change. Deployment independence becomes an illusion.

The pattern assigns data ownership to one service. It does not require buying a
separate database server for every service. Separate databases or schemas on
shared infrastructure can be intermediate choices; they still share some capacity
and failure characteristics. S03 describes the ownership principle.

## Start with three boundaries

**Code:** Orders has its own persistence model and `DbContext`.

**Contract:** other services use an API or an intentional integration event.

**Access:** deployed identities must not have arbitrary access to another owner's
tables. A coding convention is weaker than permissions.

The first lab demonstrates code and file separation. Both processes run under
your Windows user, so it does not establish an OS-level security boundary.

## The local transaction

**Compiled excerpt from `labs/Orders/Program.cs`:**

```csharp
db.Orders.Add(order);
await db.SaveChangesAsync(ct);
```

In the lab this is one local persistence operation. It does not include the
Inventory service, a message broker, or a payment provider. EF Core documents
transaction behavior in S04; provider capabilities still matter.

Do not turn a local transaction into a long network operation:

```text
BEGIN TRANSACTION
  update order
  wait for remote inventory
  wait for external payment
COMMIT
```

This is an intentionally bad design sketch. It holds local resources while the
network is uncertain and does not make the remote services atomic participants.

## Sharing identifiers is not sharing ownership

Orders stores a SKU, but it does not own the product's inventory record. A local
SKU field is a reference under a contract, not a cross-database foreign key.

Sometimes an order should preserve a purchase-time description or price. Decide
whether that is a historical snapshot or a live reference. A historical snapshot
should not unexpectedly change because another service updates its current data.

**Exercise:** model a changed product description after checkout. Decide which
screen shows today's description and which shows the purchased description.
State how the snapshot is obtained and validated.

## Lost updates are a local concern too

The lab uses an application-managed `Guid` concurrency token. A request supplies
the version it read; a successful mutation changes that version.

There are two protections:

1. Reject a version already known to be stale when the entity is loaded.
2. Let EF include the original token in the write condition, catching a race
   between loading and saving.

The second protection is not redundant. Another request can commit after the
first comparison. `DbUpdateConcurrencyException` is handled as a conflict.
See S05 for EF's concurrency semantics.

**Compiled excerpt:**

```csharp
order.Note = request.Note;
order.Version = Guid.NewGuid();
await db.SaveChangesAsync(ct);
```

The token is configured with `IsConcurrencyToken()` in the model. A `Guid` token
is not a timestamp and does not establish ordering. Every relevant write path
must update it. A different production database may support a native token.

## Queries across owners

For this project, compare:

| Approach | Tradeoff to investigate |
|---|---|
| API composition | Freshness and latency depend on participating owners |
| Local read projection | Fast reads with lag and rebuild requirements |
| Direct cross-owner SQL join | Couples persistence and access boundaries |

The first lab's `/orders/{id}/stock` uses API composition. Its response explicitly
says `reserved: false`. It must never be used as proof that checkout secured stock.

## Failure lab

1. Create a draft and record its ID and version.
2. Read its informational stock.
3. Stop Inventory.
4. Retrieve the draft again: it should still succeed.
5. Request its stock: this should fail explicitly, not return zero.
6. Restart Orders using the same database and retrieve the same draft.
7. Submit two edits using the same original version; one must be rejected.

The automated smoke check covers the core assertions. It uses isolated database
files and stops only the processes it starts.

## Pitfalls, improvements, debugging

| Symptom | First evidence to inspect | Improvement |
|---|---|---|
| Data changes after service restart | Effective connection string and file path | Set explicit environment-specific storage |
| Two edits silently overwrite | Generated update condition and version management | Enforce tokens on all write paths |
| Service cannot deploy alone | Direct SQL, shared migrations, shared models | Define explicit contracts and ownership |
| Slow summary endpoint | Time spent per remote call | Measure before considering a projection |
| New column never appears | Existing database plus `EnsureCreated` | Use controlled migrations outside the teaching lab |

The lab uses `EnsureCreated` for a disposable starter database. It is not a
production schema-upgrade strategy and does not apply migrations to an existing
schema. Production permissions, backup/restore, and data lifecycle remain work.

## Interview questions and answers

**Q: Does database per service mean one server per service?**

A: No. Ownership and controlled access are the primary boundary. Shared hosting
is possible but shares infrastructure capacity and failure risks.

**Follow-up: What would you enforce besides conventions?**

Separate identities, restricted permissions, independent schema changes, and
contract tests appropriate to the deployment.

**Q: How do you enforce a foreign key to another service?**

A: You generally cannot rely on a cross-owner database foreign key as your
service contract. Choose validation, snapshots, events, and reconciliation based
on the invariant and the tolerated inconsistency.

**Follow-up: Must every check be asynchronous?**

No. Synchronous validation can be appropriate, but its failure and staleness
windows must still be understood.

**Q: Why not retry every concurrency conflict?**

A: Replaying a stale business decision can overwrite user intent. Reload and
re-evaluate, merge under a known rule, or ask the caller to resolve it.

**Follow-up: When is automatic retry reasonable?**

For a bounded operation whose decision can safely be recomputed from fresh state,
not simply because a library supports retries.

## Independent challenge

Add an address snapshot to a draft order without granting Orders database access
to a fictional Customers database. Write down update rules, privacy/lifecycle
requirements, and the behavior when Customers is unavailable.

Completion: explain ownership separately from hosting, demonstrate stale-write
rejection, and show persistence across restart.

References: S03-S05.
