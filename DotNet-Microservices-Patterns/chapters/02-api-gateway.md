# 2. API Gateway: a deliberate client boundary

## The problem

Without a stable entry point, a client must know service addresses and routing
changes. For this exercise, expose one client endpoint and keep the mapping to
Orders in gateway configuration.

A gateway can route requests and host selected cross-cutting policies. That does
not make it the owner of order validation or payment workflow decisions. See S06.

## What the first lab actually builds

```text
http://127.0.0.1:5210/orders/...
                 |
             YARP Gateway
                 |
http://127.0.0.1:5211/orders/...
```

**Compiled gateway application:**

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));
var app = builder.Build();
app.MapGet("/health", () => Results.Ok(new { service = "gateway", status = "alive" }));
app.MapReverseProxy();
await app.RunAsync();
```

The full configuration is in `labs/Gateway/appsettings.json`. Its route matches
`/orders/{**catch-all}` and selects an Orders cluster with one local destination.
There is no prefix rewrite in this exercise.

YARP provides reverse-proxy infrastructure; adding it does not automatically
implement every capability people associate with an enterprise API-management
product. S07 documents its setup.

## Gateway, reverse proxy, and BFF

For the project's vocabulary:

- A reverse proxy forwards requests to upstream applications.
- A gateway is the chosen client-facing policy/routing boundary.
- A BFF is a backend tailored to one client experience.

One deployment can play more than one role. Do not multiply services merely to
put each label on a diagram. If mobile and web need genuinely different response
shapes, explore a BFF with explicit ownership rather than a universal aggregation
endpoint that accumulates every client's rules.

## What should stay out

Do not put the stock reservation state machine in the gateway. Otherwise direct
service callers could bypass business rules, gateway deployment becomes tightly
coupled to domain releases, and the gateway inherits workflow recovery work.

For this project, authorization near the domain operation is still required even
if an outer component authenticates the caller. The initial localhost lab has
neither authentication nor authorization; it must not be exposed to a network.

## Failure and latency budget

Define the client's deadline, then allocate a budget for routing and downstream
work. Do not independently choose a long timeout at every hop.

**Exercise:** suppose the user-facing objective for a particular request is
two seconds. Draw the call path, reserve time for response transit, and identify
which component owns cancellation. These numbers are exercise inputs, not
universal defaults.

If the downstream write may commit before a proxy timeout, the client still
faces an ambiguous result. The gateway does not make replay safe.

## Failure lab and improvements

| Experiment | Expected observation |
|---|---|
| Create an order through the gateway | Domain service creates the draft |
| Request a missing order | Downstream not-found is preserved |
| Stop Orders, leave gateway running | Gateway can be alive while the route cannot serve Orders |
| Request an unconfigured path | No implicit route to arbitrary services |
| Change the gateway destination to an unused local port | Routing fails; investigate configuration, not Orders SQL |

The smoke check verifies routing, but not every gateway failure or load scenario.
Add request correlation, authentication, rate limiting, deployment-specific TLS,
and meaningful readiness criteria as separate improvements.

## Debugging checklist

1. Confirm which process returned the status code.
2. Inspect the matched route and selected destination.
3. Verify path preservation/transforms, scheme, host, and port.
4. Test the destination directly from the gateway's execution environment.
5. Compare proxy logs with service logs for the same operation.

Do not read a gateway liveness response as proof that every backend is healthy.
Do not solve a certificate error by disabling certificate validation globally.

## Interview questions and answers

**Q: Is an API Gateway mandatory for microservices?**

A: No. It is useful when a stable client entry point and centralized edge policies
justify it. Some internal workloads communicate without a client gateway.

**Follow-up: Why not send all service-to-service traffic through it?**

That is another design decision, not a requirement. It can add coupling and an
extra hop without solving internal discovery or domain ownership.

**Q: Is the gateway a single point of failure?**

A: A single running instance can be. Replication and upstream routing can improve
availability, but shared bad configuration and common dependencies still matter.

**Follow-up: Does replication remove every failure?**

No. Replicas may share the same configuration defect or exhausted backend.

**Q: Should the gateway retry a failed POST?**

A: Only under an explicit retry-safe contract and policy. A lost response is not
proof that the downstream write did not occur.

**Follow-up: What evidence makes a repeated create safe?**

A stable operation key and durable downstream handling of duplicate requests,
including concurrent requests and changed payloads.

## Independent challenge

Design a mobile order-summary endpoint. Identify which fields can be omitted
when Inventory is unavailable and which must fail the response. Label partial
data visibly; do not silently turn "unknown" into zero.

References: S06-S07.
