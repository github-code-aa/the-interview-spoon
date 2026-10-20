# 12. Sidecar: colocate support without merging application code

Status: initial chapter; deployment design, runnable sidecar lab pending.

## The problem

Several application runtimes need a supporting capability, such as a local
telemetry forwarder. Implementing and upgrading it inside every application
language/library can create repeated integration work.

A sidecar is a companion process/container deployed alongside an application
with an intentional lifecycle relationship. S18 describes the pattern. A helper
service somewhere on the network is not a sidecar merely because the application
calls it.

## Proposed lab topology

```text
Application instance
  Orders process ---- local transport ---- telemetry companion
       |                                      |
  business requests                    telemetry backend
```

This diagram is conceptual. Edition 0.1 has no collector process or container
manifest. The initial Orders logs still go to ordinary process output.

For a Kubernetes deployment, colocated containers in a Pod share a network
namespace; that does not mean every sidecar arrangement on every platform shares
the same storage or networking model. Specify the actual platform.

## Library vs. sidecar vs. remote service

| Choice | Exercise consideration |
|---|---|
| In-process library | Direct integration; shares runtime and deployment |
| Sidecar process | Process boundary and colocated support; extra resources and local communication |
| Shared remote service | Independent central operation; network hop and shared dependency |

Choose based on capability and platform. Business ownership such as payment
authorization should not be hidden in a telemetry-style companion merely to
reuse the label.

## Define the failure relationship

Should Orders continue if the companion stops?

For optional telemetry export, continuing with bounded buffering or explicit
telemetry loss may be acceptable. For a required credential or proxy function,
application readiness and traffic handling may need a different rule.

Specify:

- Startup ordering and what "ready" means.
- Local transport timeouts and authentication where relevant.
- Memory/disk limits and bounded buffering.
- Shutdown/drain behavior.
- Companion restart and configuration updates.
- Whether companion failure should fail application readiness.

Do not assume a separate process creates unlimited isolation. Colocated processes
can still compete for CPU, memory, disk, and network resources.

## Contract sketch

**Pseudocode, not a supplied endpoint:**

```text
POST local-companion/telemetry
  traceId, timestamp, serviceName, eventName

accepted -> locally queued within documented capacity
overloaded -> explicit rejection/drop policy
unavailable -> application's documented telemetry policy
```

An "accepted" response is not automatically durable arrival at the final backend.
Name the acknowledgement boundary honestly. Never send secrets or sensitive
business payloads simply because the transport is local.

## Failure lab specification

1. Start application and companion separately in the chosen local topology.
2. Verify application requests and telemetry delivery.
3. Stop the remote telemetry backend.
4. Observe bounded queue/resource behavior.
5. Restart the companion and test application behavior.
6. Shut down the application while work is buffered.

For a deployment manifest, validate both configuration and real lifecycle
behavior. A syntactically valid manifest is not proof of reliable draining.

## Debugging and improvements

| Symptom | Inspect |
|---|---|
| Calls to localhost fail | Actual network namespace, bind address, and port |
| App healthy but telemetry missing | Local queue, export errors, acknowledgement boundary |
| App slows when collector is busy | Shared CPU/memory/disk and synchronous export |
| Termination loses unexpected data | Shutdown ordering, deadlines, durability assumptions |
| Upgrade breaks all applications | Companion contract/version compatibility |

Expose companion health and queue metrics without making optional telemetry a
recursive dependency of every health decision.

## Interview questions and answers

**Q: Why use a sidecar instead of a library?**

A: When independent runtime/process support and a local deployment relationship
justify extra operational/resource costs. It is not automatically faster or
simpler.

**Follow-up: What do you lose?**

Direct calls become interprocess communication; lifecycle, configuration,
resource allocation, and debugging become additional responsibilities.

**Q: Does localhost imply security?**

A: No. Consider process/container trust, accessible endpoints, shared resources,
and the platform's actual isolation.

**Follow-up: Can the companion hold secrets?**

Only under a deliberate secret-management and access design; colocating it is
not itself sufficient protection.

**Q: Should companion failure stop the application?**

A: It depends on whether the capability is required for correct/safe requests.
Choose and test readiness/degradation behavior explicitly.

**Follow-up: What if telemetry queues fill?**

Bound capacity and define drop/reject/durable-buffer behavior. Do not allow
unbounded buffering to take down the business process.

## Challenge

Compare implementing the same telemetry capability as a library and as a
companion. Record deployment work, resource use, failure behavior, and upgrade
compatibility before selecting one.

Reference: S18.
