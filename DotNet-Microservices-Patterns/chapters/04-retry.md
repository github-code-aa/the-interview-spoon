# 4. Retry: repeat safely, not hopefully

Status: initial chapter. API excerpt below is instructional; the executable
resilience/fault-injection lab is not included in edition 0.1.

## Start with a failed stock read

Imagine Inventory rejects one request during a brief restart. Repeating an
informational read may recover. Now imagine Payments authorized a charge but
its response was lost. Repeating without an operation identity may charge twice.

The first decision is not "three retries or five?" It is "can I safely repeat
this logical operation after an ambiguous outcome?"

Official guidance: S09-S10. Treat retry classification and budgets as part of
the operation's contract, not a universal HTTP-client default.

## A decision process

1. Determine whether the failure is plausibly transient.
2. Determine whether the operation can be repeated safely.
3. Check the remaining end-to-end deadline.
4. Apply bounded backoff, using jitter where appropriate.
5. Preserve cancellation and the logical operation identity.
6. Stop and surface the final failure when the policy is exhausted.

Authentication errors and invalid input normally need a changed request or
credentials, not repeated copies of the same operation. A 404 may be a legitimate
missing record. A timeout may hide a successful write. Rate-limit responses may
carry useful `Retry-After` guidance; consider it within your total deadline.

## .NET API direction

**Teaching excerpt; package `Microsoft.Extensions.Http.Resilience` is required
but intentionally not added to the first lab:**

```csharp
builder.Services
    .AddHttpClient("inventory-read", client =>
        client.BaseAddress = new Uri("http://inventory"))
    .AddStandardResilienceHandler(options =>
    {
        options.Retry.DisableForUnsafeHttpMethods();
    });
```

This shows API composition, not a fully tuned production policy. The standard
handler contains more strategies than retry. Learn and inspect those strategies
before combining it with additional handlers. Do not accidentally stack retry
policies from several libraries.

Disabling retries for unsafe HTTP methods is a useful starting restriction. It
is not proof that every GET endpoint is safe or that no POST can ever be retried.
The actual business contract is decisive. A safe write can be retried under a
proper idempotency contract; that requires implementation, not just a header.

## Budget arithmetic

**Original exercise values:**

```text
3 attempts, each allowed 400 ms
backoff delays: 100 ms, 200 ms
maximum configured attempt time + delays = 1,500 ms
```

This is not the full observed latency: connection establishment, scheduling,
policy overhead, serialization, and cancellation behavior also matter. A total
deadline provides a stronger outer bound than adding optimistic averages.

If three layers each make up to three attempts, one user request can fan into
as many as 27 lowest-level attempts in the worst nesting case. Count original
attempts and retries consistently when discussing this.

## Idempotency design exercise

For `POST /payments/authorizations`, propose:

- A caller-supplied operation key scoped to the correct customer/operation.
- A durable unique constraint enforcing that scope.
- A payload fingerprint to reject key reuse with different input.
- A persisted operation state/result.
- A concurrency rule for two requests arriving with the same key.
- Recovery for a crash after the external provider acts but before local recording.

The final bullet often requires the provider's own idempotency or reconciliation
contract. A local dictionary cannot provide this guarantee across replicas,
restarts, or external side effects.

## Failure lab specification

Build a deterministic Inventory fault endpoint in a later lab, restricted to
development/testing. Use a known sequence: fail first attempt, succeed second.
Assert the attempt count and final response. Then make every attempt fail and
assert the bounded total behavior. Cancel during backoff and assert no new
attempt starts.

For writes, repeat the same operation key concurrently and verify exactly one
business effect under the stated persistence assumptions. Repeat with changed
input and verify a visible conflict.

## Pitfalls and debugging

| Symptom | Suspect |
|---|---|
| Dependency traffic jumps during an outage | Nested/unbounded retries or no jitter |
| Duplicate order/payment | Replayed write without durable deduplication |
| User cancels, work continues | Cancellation token lost across a layer |
| Requests take much longer than expected | Conflicting attempt/total/client timeouts |
| Every bad request is repeated | Failure predicate too broad |

Log the attempt number, operation identity, reason, delay, and final result.
Avoid logging payment details or sensitive request bodies.

## Interview questions and answers

**Q: When is retry harmful?**

A: When it amplifies load, repeats a permanent failure, exceeds the user's
deadline, or duplicates an unsafe side effect.

**Follow-up: Does exponential backoff make a write safe?**

No. Timing controls load; idempotency controls repeated effects.

**Q: Why add jitter?**

A: To spread attempts that would otherwise synchronize after a shared failure.
It is not a replacement for bounded attempts or capacity protection.

**Follow-up: How do you test randomized delays?**

Test configured ranges/invariants and use controllable time/randomness for
deterministic tests rather than asserting exact wall-clock millisecond values.

**Q: Is an idempotency header sufficient?**

A: No. The service must persist and enforce its contract across duplicates,
concurrency, restarts, and ambiguous external results.

**Follow-up: How long do you keep keys?**

Choose a retention window consistent with allowed retries, replay, and business
requirements; document behavior after expiry.

## Challenge

Design separate policies for stock reads and payment authorization. Give the
failure predicate, deadline, operation key rules, observability, and a test that
would prove your design wrong.

References: S09-S10.
