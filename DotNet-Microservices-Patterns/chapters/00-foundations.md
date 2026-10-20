# 0. Foundations: design around what can fail

## The question changes

Inside one process, you can often follow a call stack to explain an operation.
Across processes, the caller can stop waiting while the callee keeps working.
Your first design question becomes: **what state can exist when I do not know
whether the other side succeeded?**

For this book, consider a customer buying three books. Inventory owns stock.
Payments owns payment attempts. Orders owns the customer's order and its lifecycle.
One HTTP request does not turn their databases into a single transaction.

Read this chapter before adding a broker, gateway, orchestration framework, or
container platform. Those tools do not define the business invariants for you.

## Microservices are an option, not a maturity badge

A possible benefit is independent ownership and deployment. The price in this
learning project is more failure modes, contracts, operational work, and consistency
decisions. If the application has one small team and strongly coupled workflows,
start by comparing a modular monolith rather than assuming services are superior.
See S02 for the architecture tradeoffs.

**Design exercise:** give two independent reasons to split Inventory from Orders.
"There are two tables" and "microservices scale" are not sufficient. Better
candidate requirements might be independent stock-management integrations and
different deployment ownership. These are hypothetical requirements, not facts
about every commerce system.

## Write invariants before diagrams

For the eventual capstone, choose:

- Available stock must not become negative.
- An order's payment operation must not produce duplicate external charges.
- A committed order must retain its intent to publish its integration event.
- A rejected checkout must eventually reach an explainable state.

The final invariant needs a recovery procedure and operating assumptions. It
does not mean "every failure disappears automatically."

The first executable lab implements a narrower contract: create a **Draft**
order, retrieve it, edit a note with a version check, and request an informational
stock snapshot. Creating a draft neither reserves stock nor authorizes payment.

## Vocabulary you must keep separate

| Concept | Meaning in this book |
|---|---|
| Command | A request for an owner to perform an action; it can be rejected |
| Event | A statement that something happened, published under a defined contract |
| Local transaction | A commit boundary within the selected database/resource |
| Idempotent effect | Repeating the same logical operation does not repeat its business effect |
| Eventual consistency | Different views can temporarily disagree; convergence needs working delivery/recovery assumptions |
| Correlation ID | An identifier connecting related operations for diagnosis |
| Idempotency key | An identifier used to recognize a repeated logical operation |

A correlation ID is not automatically an idempotency key. One checkout can have
many commands; a retry should retain the identity of the particular command,
not generate a new identity on every attempt.

## Learn the ambiguous outcome

```text
Orders                 Payments
  |--- Authorize(key) ---->|
  |                       | persists authorization
  |      X response ------|
  | times out             |
```

Two worlds are possible from Orders' perspective: Payments never processed the
request, or Payments processed it but the response was lost. A blind retry is
not a proof of safety. A durable operation identity and a status/reconciliation
contract make that ambiguity manageable.

**Exercise:** draw the same timeline for a database commit followed by message
publication. Place a process crash before and after each durable operation.
Keep this diagram for the Outbox chapter.

## Reference project boundaries

```text
Client -> Gateway -> Orders API -> orders.db
                        |
                        +-> Inventory API -> inventory.db
```

This is the actual first-lab topology. No broker, Saga, or payment implementation
is hidden behind it. The gateway uses configured destinations; Orders uses
service discovery for Inventory. Those are different mechanisms.

## Observation before automation

For each failure, record:

1. Which request was attempted and with what logical identity?
2. Which service saw it?
3. What durable state exists?
4. What did the caller observe?
5. Which component is responsible for recovery?

Do not start by increasing timeouts. First identify where the elapsed time went.
For the initial lab use HTTP results and per-process logs. Distributed tracing
and metrics are an explicit future improvement, not already installed.

## Interview questions and answers

**Q: Why not make every module a microservice?**

A: A process boundary adds communication and operational costs. Use the boundary
when ownership, deployment, isolation, or scale requirements justify those costs.
Compare an in-process module boundary as an alternative.

**Follow-up: What evidence would change your decision?**

Independent release requirements, measured contention, a different trust boundary,
or repeated organizational coupling could justify revisiting the design.

**Q: Does a timeout mean the transaction failed?**

A: No. It describes the caller's observation, not the callee's committed state.

**Follow-up: What do you return to the user?**

Choose an API contract that represents uncertainty honestly, for example a
tracked operation with a status endpoint. Do not label an unconfirmed payment
failure as a definitive decline.

**Q: Where should a strict stock invariant live?**

A: With the component that owns the stock mutation and can enforce the invariant
atomically. Reading stock from Orders and later subtracting elsewhere is not
one atomic decision.

**Follow-up: Does a cache fix the race?**

No. A cache may reduce reads but is not, by itself, authoritative coordination.

## Challenges and mastery check

- Run the baseline lab and explain every process/database boundary.
- Stop Inventory. Predict which Orders endpoints still work, then verify.
- Explain why a stock snapshot is not a reservation.
- Describe one case where a modular monolith is the simpler correct solution.
- Distinguish a failure to respond from a confirmed business rejection.

References: S02-S05, S09. See `labs/README.md` for runnable commands.
