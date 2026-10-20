# 7. Outbox: preserve publication intent with the business change

Status: initial chapter; transaction sketches and schema design, real broker lab
pending. No broker guarantee is inferred from the first HTTP lab.

## The dual-write problem

Orders must save an order and notify other services. Consider the original
failure timeline:

```text
1. Save order in Orders database
2. Publish OrderPlaced to broker
```

Crash between 1 and 2: the order exists, but the notification was never sent.
Reverse the order: the event can escape even if the order write fails.

The relational outbox design for this book writes the business change and an
outgoing message record in the same local database transaction. A separate relay
publishes committed records. See S13 for the pattern and its platform-specific
example; our relational boundary is not a Cosmos cross-partition transaction.

## Minimal record design

**Schema sketch, not a migration:**

```text
OutboxMessage
  MessageId       unique primary key
  AggregateId
  AggregateVersion (if ordering/version rules require it)
  MessageType
  ContractVersion
  Payload
  OccurredUtc
  PublishedUtc    nullable
  AttemptCount
  NextAttemptUtc
  LeaseOwner      nullable
  LeaseUntilUtc   nullable
```

Do not blindly add every column to every system. They show the decisions a
concurrent, recoverable publisher must address.

**EF-shaped pseudocode; supporting types omitted:**

```csharp
db.Orders.Add(order);
db.OutboxMessages.Add(message);
await db.SaveChangesAsync(ct);
```

This relies on both tracked writes participating in the same local transaction
under the selected provider. Separate contexts or separate databases do not
become atomic because the method names look similar.

## Relay sequence and the unavoidable duplicate window

```text
Read/claim committed pending message
Publish using stable MessageId
Receive broker confirmation
Mark message published in the database
```

Crash after publication/confirmation but before recording `PublishedUtc`: the
relay can publish again after restart. Marking published first would instead
risk losing the message. The intended design accepts duplicate delivery and
makes consumers handle it.

A successful local send call is not always a durable broker confirmation.
Use the chosen broker client's documented acknowledgement behavior.

## Consumer inbox / deduplication

For a database-only effect, store a processed-message marker and perform the
business update in one consumer-local transaction.

**Transaction pseudocode:**

```text
BEGIN
  insert Inbox(ConsumerIdentity, MessageId) with unique constraint
  apply the stock reservation under its business invariant
COMMIT
acknowledge broker message
```

If the marker already exists for this consumer, treat it according to the
defined duplicate contract. A check-then-insert without a uniqueness constraint
is racy under concurrent delivery.

If the effect is an external payment call, this transaction cannot atomically
include the external provider. Return to provider idempotency/reconciliation.

## Multi-worker and ordering decisions

Two relay instances must not merely select the same first 100 rows and assume
only one will publish. Choose provider-appropriate claims/leases/locking and
test lease expiry and worker death.

Ordering is a separate requirement. A shared timestamp is not a reliable total
order. If per-order sequence matters, define aggregate versions, partitioning,
and consumer behavior for gaps/duplicates. Do not demand global ordering unless
the business really needs it.

## Failure lab specification

1. Broker down: commit an order and outbox row; verify both survive.
2. Broker restored: publish the pending event and record confirmation.
3. Relay crash after publish: restart and deliver the duplicate.
4. Consumer crash before commit: redelivery must be able to apply the effect.
5. Consumer crash after commit before acknowledgement: redelivery must not repeat it.
6. Two workers: verify claim behavior and stable message identity.

## Pitfalls, improvements, debugging

| Symptom | Evidence |
|---|---|
| Orders exist but no outbox intent | Transaction boundary and write paths |
| Pending messages grow | Oldest age, relay errors, broker confirmations |
| Duplicate effects | Message identity, inbox constraint, consumer transaction |
| One poison message blocks progress | Retry scheduling, quarantine/dead-letter policy |
| Disk grows indefinitely | Retention and cleanup under recovery requirements |

Track oldest pending age as well as count. A low count can still hide one
business-critical message stuck for hours. Preserve diagnostics without logging
sensitive payloads. Cleanup must respect replay and investigation needs.

## Interview questions and answers

**Q: Does Outbox provide exactly-once delivery?**

A: No. It couples local state and intent. The relay can duplicate publication.
Consumer effects require their own handling and explicit assumptions.

**Follow-up: Why not mark published before sending?**

A crash after marking but before sending can permanently suppress publication.

**Q: Can the outbox live in a different database?**

A: Not if you are relying on one ordinary local transaction to couple both writes.
A different design requires a different atomicity argument.

**Follow-up: Does the same SQL server imply the same transaction?**

No. Specify connection, transaction, database/provider capabilities, and actual
enlistment rather than assuming atomicity from hosting.

**Q: How do you handle duplicate consumer messages?**

A: Use stable identity and durable deduplication coupled to the local effect,
plus external-operation idempotency where the effect is outside the database.

**Follow-up: Can inbox rows be deleted immediately?**

Only if your replay/redelivery contract makes that safe. Otherwise a late
duplicate can be treated as new.

## Challenge

Draw every crash window from HTTP request to consumer acknowledgement. For each
window state: durable records, possible duplicate, recovery owner, and business
effect. You are done only when every window has an explicit answer.

References: S04, S13.
