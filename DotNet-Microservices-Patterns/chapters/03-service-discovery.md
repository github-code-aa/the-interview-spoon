# 3. Service Discovery: names are not addresses

## The problem

An address is deployment information. If every caller embeds Inventory's IP and
port, changing the deployment changes application code.

For the lab, Orders calls a logical name, `inventory`. Configuration maps that
name to a local address. This is a deliberately simple discovery setup, not a
dynamic registration platform. See S08.

## Executable setup

**Compiled excerpt from Orders:**

```csharp
builder.Services.AddServiceDiscovery();
builder.Services.AddHttpClient<InventoryClient>(client =>
{
    client.BaseAddress = new Uri("http://inventory");
    client.Timeout = TimeSpan.FromSeconds(2);
}).AddServiceDiscovery();
```

**Actual configuration excerpt:**

```json
{
  "Services": {
    "inventory": {
      "http": ["http://127.0.0.1:5212"]
    }
  }
}
```

`InventoryClient` requests `stock/{escapedSku}` relative to the base address.
The smoke check supplies an alternate Inventory endpoint via configuration,
demonstrating that the service address is not fixed in client code.

The package does not infer the name of your database or establish trust in the
remote service. Authentication, TLS, and endpoint permissions are separate work.

## Discovery is not the whole connectivity story

Separate four questions:

| Question | Concern |
|---|---|
| Where can this service be reached? | Discovery |
| Which instance receives this request? | Selection/load balancing |
| Can it serve useful traffic now? | Readiness/health |
| What happens if this request fails? | Resilience and business recovery |

A selected address can become unavailable immediately after it is resolved.
No amount of naming removes that race.

The initial client has one configured endpoint and a timeout. It has no retry,
circuit breaker, active probing, failover policy, or measured load distribution.
The gateway uses its own static destination configuration; it is not wired to
the Orders discovery client.

## Environment names and transport

`localhost` inside a container refers to that container, not another container
or your host by default. When moving this lab to containers, change the mapping
to the actual reachable service address.

An HTTPS configuration also needs valid certificates and the expected service
identity. Do not "fix discovery" by ignoring TLS validation.

## Failure lab

1. Run the smoke check: it chooses unused loopback ports and injects mappings.
2. In a manual run, set Orders' Inventory mapping to a different port.
3. Request `/orders/{id}/stock`.
4. Confirm the explicit dependency error and the Orders log.
5. Restore the mapping and repeat.

**Challenge:** run Inventory on another loopback port without modifying
`InventoryClient`. Explain which configuration provider wins if the file and
environment define different values.

The environment key for the first address is:

```powershell
$env:Services__inventory__http__0 = 'http://127.0.0.1:5312'
```

Use it only in the Orders process/session where intended. Remove it afterwards
with `Remove-Item Env:Services__inventory__http__0`.

## Debugging and improvement

| Symptom | Investigation |
|---|---|
| Name resolves locally, not in container | Network namespace and effective configuration |
| Correct service, wrong port | Scheme-specific endpoint mapping |
| Works once, fails after deployment | Stale configuration, endpoint change, in-flight connections |
| Liveness succeeds, business call fails | Database/queue/downstream readiness and route |
| Unexpected destination | Configuration-provider precedence |

The full lab logs transport errors rather than returning a fabricated stock
value. A not-found SKU is a business/data response; a failed connection is a
dependency failure. Do not collapse them.

## Interview questions and answers

**Q: How is service discovery different from an API Gateway?**

A: Discovery locates a service; a gateway handles a chosen traffic boundary and
routing/policies. They can interact but are not substitutes.

**Follow-up: Can DNS be enough?**

Yes, where the platform's DNS behavior and addressing meet the application's
requirements. A dedicated registry is not inherently required.

**Q: Does discovery guarantee that an instance is healthy?**

A: No. Even health-aware mechanisms have a gap between observation and use.
The caller still needs deadlines and failure handling.

**Follow-up: Should a failed lookup return an empty result?**

No. That would turn an infrastructure failure into apparently valid business
data. Return an explicit error or a deliberately defined degraded response.

**Q: How would you change a service address safely?**

A: Use the deployment's supported configuration/discovery mechanism, understand
refresh and connection behavior, and observe traffic during the change.

**Follow-up: Is editing configuration always immediate?**

No. Refresh behavior depends on the provider and client. Test it rather than
assuming every component reloads in the same way.

## Mastery check

Explain the logical name, resolved address, selected endpoint, and timeout as
four separate things. Move Inventory without editing client code. Diagnose a
wrong-port mapping from evidence.

Reference: S08.
