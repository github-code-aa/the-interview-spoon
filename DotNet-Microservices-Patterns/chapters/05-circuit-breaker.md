# 5. Circuit Breaker: stop spending on a known-bad path

Status: initial chapter; state-machine exercise, not an executable breaker lab.

## The problem

Retry helps some temporary failures, but a persistently failing Inventory can
consume connections and waiting requests without useful work. A circuit breaker
observes selected outcomes and temporarily rejects calls on that path.

S11 explains the pattern. The .NET standard HTTP resilience stack in S09 includes
a breaker; adding one does not automatically define your business fallback.

## State model

**Teaching state diagram:**

```text
Closed -- selected failures cross configured criterion --> Open
Open   -- break interval elapses -----------------------> Half-open
Half-open -- permitted probe succeeds ------------------> Closed
Half-open -- probe fails -------------------------------> Open
```

This is a conceptual diagram. Actual libraries differ in sampling, minimum
throughput, probe admission, and state transition behavior.

Choose what counts as a dependency failure. A customer's invalid SKU should not
necessarily contribute to the same circuit as transport failures. Partition
policy deliberately: a shared circuit for unrelated destinations can block
healthy work.

## Use with retry and timeout

The order of strategies changes what each observes. If a breaker is inside a
retry strategy, it may observe individual attempts rather than just one final
user-operation result. This is one reason to inspect the configured stack instead
of adding handlers blindly.

For a walkthrough, assume an outage affects every attempt:

1. Closed circuit allows requests and records selected failures.
2. Policy's trip conditions are reached.
3. Open circuit rejects new attempts quickly.
4. After the configured delay, limited trial traffic is admitted.
5. Successful recovery allows ordinary traffic again under policy rules.

An open circuit does not terminate all remote work already in progress.

## Exercise configuration, not universal defaults

Invent a small lab profile with a sampling window, failure ratio, minimum
throughput, and break interval. Predict behavior under:

- One failure at very low traffic.
- Many failures during high traffic.
- Successful calls mixed with failures.
- A dependency that becomes healthy while the circuit is open.

Explain why a low-traffic service might never meet a high minimum-throughput
threshold. Test the library's real behavior rather than approximating it with
"open after five errors" unless that is the actual policy.

## Fallback must preserve meaning

For our stock endpoint, a breaker-open response means the requested live stock
is unavailable. Returning `available: 0` lies about the business state.

An explicitly labeled stale snapshot may be suitable for a display. It is not
authority to reserve inventory. Decide which endpoints may degrade and which
must reject the operation.

## Debugging and improvements

| Symptom | Evidence to collect |
|---|---|
| Circuit never opens | Selected failures, request volume, sample/minimum thresholds |
| Healthy service appears blocked | Shared policy partition and previous failure history |
| Failure remains after dependency recovery | Open duration, probe results, refresh/routing state |
| Some replicas work, others reject | Per-process breaker states and traffic distribution |
| Error dashboards look quiet | Breaker rejections counted separately from network attempts |

Observe transitions, rejections, trial results, dependency latency, and attempt
counts. A process-local breaker is not a globally coordinated service-health
registry. Restarting a process can reset local history.

## Interview questions and answers

**Q: How is a breaker different from retry?**

A: Retry repeats selected failures; a breaker temporarily stops ordinary
attempts when the path appears unhealthy. They address different costs.

**Follow-up: Should a breaker replace timeouts?**

No. Calls that are admitted still need bounded waiting.

**Q: Does an open circuit prove the service is down?**

A: No. It is a local policy decision based on observations. The service may have
recovered, or only one route may be failing.

**Follow-up: How does the caller find out about recovery?**

Through the configured trial/probe behavior or a deliberately integrated recovery
mechanism, not by assuming an unrelated health check closes the circuit.

**Q: Why can two replicas disagree about breaker state?**

A: Each may observe different requests and maintain separate local state.

**Follow-up: Should you build a global breaker?**

Only with a specific need and an understanding of coordination/staleness costs.
Local disagreement alone does not prove that shared coordination is better.

## Challenge and acceptance

Create a controlled failing dependency. Assert fewer network attempts while the
circuit is open, then restore the dependency and demonstrate recovery. Avoid a
flaky test that sleeps a guessed interval without observing state or results.

References: S09, S11.
