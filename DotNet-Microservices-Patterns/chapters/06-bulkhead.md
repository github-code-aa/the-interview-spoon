# 6. Bulkhead: reserve capacity for unrelated work

Status: initial chapter; conceptual C# excerpt, executable load lab pending.

## The problem

A slow report dependency consumes all the application's outbound concurrency.
Order reads become slow even though their own database is healthy.

For this project, a bulkhead means deliberately separating capacity: outbound
limits per dependency, distinct worker pools, bounded queues, or separate
deployments where justified. S12 describes the isolation principle.

## Start with a capacity question

Which scarce resource are you protecting?

- In-flight network operations?
- Database connections?
- CPU-heavy work?
- Background worker slots?
- Memory consumed by queued requests?

An asynchronous call does not occupy a blocked thread for its entire wait, but it
still consumes request state, memory, sockets/connections, and capacity elsewhere.
`async` is not a capacity policy.

## Minimal mechanism

**Teaching excerpt, not a registered lab service:**

```csharp
sealed class InventoryGate
{
    private readonly SemaphoreSlim slots = new(8, 8);

    public async Task<T> ExecuteAsync<T>(
        Func<CancellationToken, Task<T>> operation,
        CancellationToken cancellationToken)
    {
        if (!await slots.WaitAsync(0, cancellationToken))
            throw new InvalidOperationException("Inventory capacity exhausted.");

        try
        {
            return await operation(cancellationToken);
        }
        finally
        {
            slots.Release();
        }
    }
}
```

Eight is an arbitrary exercise value, not sizing advice. The excerpt demonstrates
fail-fast admission and release. It lacks a domain-specific rejection type,
metrics, hosting integration, lifecycle handling, and a chosen HTTP mapping.
Use established resilience/rate-limiting tooling for the later implementation.

The gate must have the intended shared lifetime. Creating it per request gives
each request eight private slots and does not cap overall concurrency.

## Queue or reject?

A queue absorbs some short bursts but increases waiting and memory use.
An unbounded queue merely moves the outage. Choose a maximum size and maximum
wait, preserve cancellation, and define the response when full.

For illustration, if a dependency sustains 20 operations/second at an average
0.25 seconds of service time under stable conditions, the average concurrency
implied by those values is 5. That does not mean a limit of 5 is automatically
optimal: bursts, tail latency, saturation, retries, and workload variation matter.
Measure the real system.

## Scope matters

A per-process limit of eight with four replicas permits up to 32 admitted
operations across those replicas, assuming all are busy. It is not a global
dependency quota. Scaling callers can overload a shared dependency unless you
revisit aggregate capacity.

Separate Orders' Inventory and Payment policies if they should not compete for
the same slots. For true process-resource isolation, a separate process or
deployment may be necessary.

## Debugging and improvement

| Symptom | Likely investigation |
|---|---|
| Concurrency cap seems ineffective | Gate lifetime and number of replicas |
| Queue grows without recovery | Arrival rate vs. completion rate; boundedness |
| Capacity disappears over time | Release paths and cancellations |
| Fail-fast responses spike after scaling | Aggregate downstream capacity |
| Unrelated requests still fail | Shared database pool, CPU, memory, or another unisolated resource |

Measure admitted, rejected, queued, active, wait duration, and execution duration
separately. A low average latency that excludes rejected requests can hide a
poor user experience.

## Interview questions and answers

**Q: Bulkhead vs. Circuit Breaker?**

A: A bulkhead limits/isolate resource consumption. A breaker rejects traffic
based on a path's observed failures. A healthy but slow service can exhaust
capacity without tripping a failure-based breaker.

**Follow-up: Can both reject a call before it reaches the network?**

Yes, but for different reasons; log and count those reasons separately.

**Q: Is `Task.Run` a bulkhead?**

A: Not by itself. It schedules work; it does not establish a dedicated bounded
capacity policy for a dependency.

**Follow-up: Is a shared `SemaphoreSlim` enough for global limits?**

No. It controls the process that owns it, not all replicas.

**Q: Should overflow queue forever?**

A: No. Define bounded admission and deadlines so overload is explicit and
recoverable rather than an ever-growing backlog.

**Follow-up: Return 429 or 503?**

Choose according to the API contract: caller quota/rate policy and temporary
service-capacity exhaustion are different conditions.

## Challenge

Run slow Inventory calls alongside fast local order reads. Define a baseline
latency threshold for the local endpoint, cap Inventory concurrency, and prove
both the cap and the unrelated-workload threshold under a specified load.

References: S09, S12.
