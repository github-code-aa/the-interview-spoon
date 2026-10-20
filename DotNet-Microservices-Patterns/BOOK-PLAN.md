# Editorial and implementation plan

## Audience and scope

Experienced .NET developer moving into distributed application design. Skip long
C# introductions; explain database and network assumptions in depth.

The twelve patterns remain in the agreed order. Foundations precede them and a
capstone follows. Event Sourcing, migration, and sidecar experiments stay separate
where combining them would obscure the lesson.

## Delivery sequence

1. **First installment:** initial text for all chapters; runnable ownership,
   gateway, discovery, persistence, and optimistic-concurrency baseline.
2. **Resilience installment:** isolated retry, breaker, and bulkhead labs with
   deterministic fault injection, assertions, attempt counts, and latency bounds.
3. **Messaging installment:** a real broker, durable outbox/inbox, duplicate
   delivery, concurrent workers, and crash-window exercises.
4. **Workflow installment:** persisted Saga transitions, deadlines, late messages,
   compensation failure, recovery, and operational queries.
5. **Modeling installment:** same-database CQRS, asynchronous read projections,
   event-sourced aggregate, expected-version appends, replay, and event evolution.
6. **Migration/deployment installment:** legacy/new routing, data cutover and
   rollback rehearsal, then sidecar process lifecycle and failure isolation.
7. **Editorial pass:** expand diagrams, worked answers, independent challenges,
   measured outcomes, cross-references, terminology, and a single reading edition.

These are work packages, not time estimates or automatically scheduled work.

## Chapter quality gate

- Business requirement and observable invariant are explicit.
- Happy path, ambiguous outcome, duplicate, timeout, and restart are considered.
- Every code block is labeled runnable, excerpt, pseudocode, or exercise.
- Runnable examples have exact commands and expected outputs.
- Unsupported claims of exactly-once, automatic rollback, or guaranteed recovery
  are removed or bounded by their actual assumptions.
- Every interview question has an answer or a separate solution.
- Implementation status is not confused with architectural intent.
- Documentation references are checked; package API usage is compiled.
- Production limitations and deferred work are visible to the reader.

## Still to expand in this edition

The later chapters are substantive starting drafts, not full-length finished
chapters. They need their executable labs and more worked examples. Initial
interview sections contain three answered question/follow-up pairs per pattern;
the expanded edition targets six to ten plus incident scenarios.
