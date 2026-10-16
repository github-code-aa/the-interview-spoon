# Z04 – Azure Service Bus: Study + Interview Guide

**How to use:** Read each topic's *Study notes*, then walk the difficulty ladder (Easy -> Tricky) answering aloud before reading the **Answer**. Code targets `Azure.Messaging.ServiceBus` (v7.x) and isolated-worker Functions; limits/versions change, so anything marked *verify in current docs* should be re-checked.

## Table of Contents
1. Messaging concepts and choosing Service Bus vs Event Grid / Event Hubs / Storage Queues
2. Tiers, limits, messaging units, partitioning
3. Message anatomy
4. Sending (ServiceBusClient / Sender, batching, scheduling)
5. Receiving: Processor vs Receiver, receive modes
6. Settlement, locks, delivery count
7. Dead-letter queue (DLQ)
8. Sessions
9. Duplicate detection and transactions
10. Topics, subscriptions, rules/filters, auto-forwarding
11. Idempotency, delivery guarantees, ordering
12. Retries, backoff, deferral
13. Request/response, sagas, outbox
14. Service Bus + Azure Functions
15. Service Bus + Logic Apps
16. Security (RBAC, SAS, private endpoints)
17. Geo-DR, geo-replication, availability zones
18. Monitoring and alerting
19. Testing (emulator, mocking)
20. Scaling consumers, claim-check, production scenarios

---

## Topic 1: Messaging concepts and choosing the right service

**Study notes**

- **Queue** = point-to-point: each message is processed by exactly one consumer (competing consumers share the load).
- **Topic + subscriptions** = pub/sub: each subscription gets its own copy of every matching message; within a subscription consumers compete.
- Service Bus is an *enterprise message broker* (commands/workflows, transactions, sessions, DLQ). Event Grid is *reactive event routing* (discrete "something happened", push/HTTP). Event Hubs is a *streaming log* (high-throughput telemetry, consumer-managed offsets, replay). Storage Queues are *simple, cheap, huge queues* (no ordering guarantee, no DLQ/sessions/topics).

```
Producer -> [Queue] -> Consumer A | Consumer B   (one gets each msg)
Producer -> [Topic] -> Sub1 -> Consumers (copy 1)
                    -> Sub2 -> Consumers (copy 2)
```

| Need | Pick |
|---|---|
| Business commands, ordering (sessions), DLQ, transactions | Service Bus |
| React to Azure resource events (blob created), fan-out via HTTP/webhook | Event Grid |
| Millions of events/sec, replay, analytics | Event Hubs |
| Simple work queue, >80 GB backlog, lowest cost | Storage Queue |

### 1.1 [Easy] What is the difference between a queue and a topic?
**Answer**
A queue delivers each message to one receiver (point-to-point). A topic fans out each message to every subscription (pub/sub), and each subscription behaves like its own queue.

**Explanation**
Subscriptions hold independent copies, so one slow subscriber never blocks another. A queue with N consumers is competing consumers; a topic with N subscriptions is N independent workloads.

**Code / CLI / Bicep**
```bash
az servicebus queue create -g rg -n orders --namespace-name sb-demo
az servicebus topic create -g rg -n order-events --namespace-name sb-demo
az servicebus topic subscription create -g rg -n billing --topic-name order-events --namespace-name sb-demo
```

### 1.2 [Follow-up] When would you choose Service Bus over Storage Queues?
**Answer**
When you need FIFO via sessions, DLQ, duplicate detection, transactions, topics/filters, scheduled delivery, or AAD-based RBAC at the entity level. Choose Storage Queues when you only need a cheap, simple, very large queue.

**Explanation**
Storage Queues: 64 KB messages, no strict ordering, no built-in DLQ (you implement poison handling via dequeue count), at-least-once. Service Bus has richer semantics but cost and limits (e.g., entity size) differ — verify current limits.

**Code / CLI / Bicep**
```csharp
// Service Bus: broker-managed DLQ
await receiver.DeadLetterMessageAsync(msg, "BadPayload", "Missing OrderId");
```

### 1.3 [Medium] Event Grid vs Event Hubs vs Service Bus: how do you decide?
**Answer**
Event Grid = discrete notifications, push with retry, low-latency fan-out. Event Hubs = high-volume ordered-per-partition stream you read at your own offset. Service Bus = reliable command/workflow messaging where each message must be processed and settled.

**Explanation**
Event semantic ("OrderCreated happened") vs message semantic ("ProcessOrder please"). Event Grid can deliver to Service Bus queues/topics as a handler, a common combo: Blob event -> Event Grid -> Service Bus -> Function with retries/DLQ.

**Code / CLI / Bicep**
```bash
az eventgrid event-subscription create --name blob-to-sb \
  --source-resource-id <storageAccountId> \
  --endpoint-type servicebusqueue \
  --endpoint <serviceBusQueueResourceId>
```

### 1.4 [Tricky] "Service Bus is just a queue, so we can use it for telemetry ingestion." What is wrong?
**Answer**
Telemetry is high volume, order-insensitive, replay-oriented: that is Event Hubs. Service Bus has per-message broker overhead, tier-based throughput limits, and no offset replay.

**Explanation**
Each Service Bus message is individually locked and settled; this is expensive at millions/sec. Events can't be "re-read" after completion in Service Bus (only before settlement or via DLQ), whereas Event Hubs retains the log for the retention window.

**Code / CLI / Bicep**
```csharp
// Event Hubs: consumer controls checkpoint/replay; Service Bus: broker deletes on Complete.
await args.CompleteMessageAsync(args.Message); // message is gone
```

**Gotchas / follow-ups**
- Event Grid delivery is at-least-once with limited retry window; Service Bus keeps messages up to TTL.
- Mixed designs are normal; say so in the interview.

**Interview traps**
- Calling Event Grid "a queue" — it is push-based routing, not a pull queue.
- Claiming Service Bus has replay — it does not.
- Forgetting Storage Queues exist as a cheaper option.

**Cheat sheet**
- Queue = 1 consumer per message; Topic = copy per subscription.
- SB = commands/workflows; EG = reactive events; EH = streams; SQ = cheap simple.
- Event Grid -> Service Bus is a common bridge.

---

## Topic 2: Tiers, limits, messaging units, partitioning

**Study notes**

| | Basic | Standard | Premium |
|---|---|---|---|
| Queues | yes | yes | yes |
| Topics/subscriptions | no | yes | yes |
| Pricing | per-operation | per-operation + base | per Messaging Unit (MU), dedicated |
| Max message size | 256 KB | 256 KB | up to 100 MB (verify in current docs) |
| Private endpoints / IP firewall / VNet | no | no | yes |
| Geo-DR, zone redundancy, customer-managed keys | no | no | yes |
| Resource isolation | shared | shared (throttling) | dedicated |

- Premium capacity = Messaging Units (1, 2, 4, 8, 16). Standard has no MUs (shared, can be throttled). "Throughput units" are an Event Hubs concept, not Service Bus.
- **Partitioning** splits an entity across multiple message brokers/stores to raise throughput/availability; ordering is only per partition key (SessionId/PartitionKey). Cannot be toggled after entity creation. Premium partitioning model differs from Standard — verify in current docs.

### 2.1 [Easy] What are the Service Bus tiers?
**Answer**
Basic (queues only), Standard (queues + topics, shared infrastructure), Premium (dedicated MUs, large messages, network isolation, Geo-DR, zones).

**Explanation**
Production enterprise workloads usually need Premium for predictable latency and private networking.

**Code / CLI / Bicep**
```bash
az servicebus namespace create -g rg -n sb-prem --location westeurope --sku Premium --capacity 1
```

### 2.2 [Follow-up] What is the max message size and what happens if exceeded?
**Answer**
256 KB on Basic/Standard; Premium supports up to 100 MB over AMQP (verify in current docs). Exceeding throws `ServiceBusException` with `Reason == MessageSizeExceeded`.

**Explanation**
The limit includes headers and properties, not only body. For larger payloads use claim-check (Topic 20).

**Code / CLI / Bicep**
```csharp
try { await sender.SendMessageAsync(msg); }
catch (ServiceBusException ex) when (ex.Reason == ServiceBusFailureReason.MessageSizeExceeded)
{ /* upload to blob, send pointer */ }
```

### 2.3 [Medium] Standard vs Premium: what actually differs operationally?
**Answer**
Premium gives dedicated CPU/memory (MUs), no noisy-neighbor throttling, VNet/private endpoint, Geo-DR, zone redundancy, bigger messages, and JMS 2.0. Standard is cheaper and shared and can return server-busy under load.

**Explanation**
Premium scale-out is by changing MUs (some control-plane time; verify). Monitor `NamespaceCpuUsage` / `NamespaceMemoryUsage` in Premium to decide.

**Code / CLI / Bicep**
```bicep
resource ns 'Microsoft.ServiceBus/namespaces@2022-10-01-preview' = {
  name: 'sb-prem'
  location: resourceGroup().location
  sku: { name: 'Premium', tier: 'Premium', capacity: 1 }
  properties: { zoneRedundant: true, disableLocalAuth: true, minimumTlsVersion: '1.2' }
}
```

### 2.4 [Hard] How does partitioning affect ordering, sessions, and transactions?
**Answer**
Messages with the same partition key / SessionId go to the same partition, preserving order there; there is no global order across partitions. Transactions across partitions have restrictions, and you cannot change partitioning after creation.

**Explanation**
Partitioned entities improve availability (a failed partition doesn't take the entity down) and throughput. Duplicate detection with partitioning needs MessageId + PartitionKey consistency. Many teams don't need partitioning on Premium; verify the current guidance.

**Code / CLI / Bicep**
```bicep
resource q 'Microsoft.ServiceBus/namespaces/queues@2022-10-01-preview' = {
  parent: ns
  name: 'orders'
  properties: { enablePartitioning: false, requiresSession: true }
}
```

**Gotchas / follow-ups**
- `enablePartitioning` is immutable; recreate the entity to change.
- Setting PartitionKey and SessionId to different values on a message causes an error; they must match when both are set.

### 2.5 [Tricky] Your Standard namespace returns ServiceBusy / throttling. What do you do?
**Answer**
Confirm via `ThrottledRequests` metric, make clients retry with exponential backoff, batch sends, reuse clients, and if sustained, move to Premium or spread load across namespaces.

**Explanation**
Standard is shared and throttles per-namespace credit-based limits. Premium throttles only on CPU/memory of your MUs. Also check connection count and unnecessary receiver creation per message.

**Code / CLI / Bicep**
```bash
az monitor metrics list --resource <nsId> --metric ThrottledRequests --interval PT5M --aggregation Total
```

**Gotchas / follow-ups**
- `ServiceBusFailureReason.ServiceBusy` is retriable; SDK retry handles it within limits.

**Interview traps**
- "Throughput units" are Event Hubs; SB Premium uses Messaging Units.
- Saying Standard supports private endpoints (Premium only).
- Assuming partitioning can be enabled later.

**Cheat sheet**
- Basic: no topics. Standard: shared. Premium: dedicated MUs + network + Geo-DR.
- 256 KB (Std), up to 100 MB (Premium) - verify.
- Partitioning immutable; order per partition/session only.

---

## Topic 3: Message anatomy

**Study notes**

```
ServiceBusMessage
 |- Body (BinaryData)             - opaque payload
 |- ContentType                   - e.g. application/json
 |- Subject (Label)               - app-level routing label
 |- MessageId                     - dedup key
 |- CorrelationId                 - trace/correlate
 |- SessionId / PartitionKey      - ordering / grouping
 |- ReplyTo / ReplyToSessionId    - request-reply
 |- TimeToLive                    - per-message TTL (<= entity default)
 |- ScheduledEnqueueTime          - delayed visibility
 |- ApplicationProperties         - custom key/values (usable in filters)
Broker-set (ServiceBusReceivedMessage): SequenceNumber, EnqueuedTime, DeliveryCount, LockToken, LockedUntil, DeadLetterReason...
```

### 3.1 [Easy] What are the main parts of a Service Bus message?
**Answer**
Body, system/broker properties (MessageId, CorrelationId, SessionId, TTL, ContentType, Subject), and custom `ApplicationProperties`.

**Explanation**
Body is opaque binary; set `ContentType` so consumers can deserialize. `ApplicationProperties` are the only custom fields usable in subscription filters.

**Code / CLI / Bicep**
```csharp
var msg = new ServiceBusMessage(BinaryData.FromObjectAsJson(order))
{
    MessageId = order.Id.ToString(),
    ContentType = "application/json",
    Subject = "OrderCreated",
    CorrelationId = Activity.Current?.TraceId.ToString()
};
msg.ApplicationProperties["tenant"] = "contoso";
```

### 3.2 [Follow-up] What is the difference between MessageId and CorrelationId?
**Answer**
MessageId identifies one message (used for duplicate detection). CorrelationId links related messages (e.g., reply to request, trace ID) and is not unique.

**Explanation**
For request/reply, the reply sets `CorrelationId = request.MessageId`. Don't use random GUID MessageIds if you want duplicate detection to work for logical retries; use a business key.

**Code / CLI / Bicep**
```csharp
var reply = new ServiceBusMessage(body) { CorrelationId = request.MessageId, SessionId = request.ReplyToSessionId };
```

### 3.3 [Medium] How does TimeToLive work, and what happens on expiry?
**Answer**
Message TTL is set per message but capped by the entity's `DefaultMessageTimeToLive`. On expiry the message is removed, or moved to the DLQ if `DeadLetteringOnMessageExpiration` is enabled.

**Explanation**
Expiry is evaluated lazily by the broker; expired messages may linger visually in counts briefly. Scheduled messages' TTL starts at enqueue time (verify). Messages with expiry in DLQ get reason `TTLExpiredException`.

**Code / CLI / Bicep**
```bicep
properties: { defaultMessageTimeToLive: 'P7D', deadLetteringOnMessageExpiration: true }
```

### 3.4 [Hard] What is stored in ApplicationProperties vs the body, and how does it impact filters and size?
**Answer**
Put routing/metadata in `ApplicationProperties` (filterable, readable without deserializing) and keep the domain payload in the body. Properties count against the size limit and support primitive types only.

**Explanation**
SQL/correlation filters can't see the body. Property values should be simple types (string, numeric, bool, DateTime, Guid...) — complex objects aren't supported (verify supported types).

**Code / CLI / Bicep**
```csharp
msg.ApplicationProperties["priority"] = 1;
msg.ApplicationProperties["region"] = "EU";
```

**Gotchas / follow-ups**
- Putting PII in properties exposes it to logs/filters.
- Property key names are case-sensitive in filters? Treat as exact; verify.

### 3.5 [Tricky] A consumer reads `msg.Body.ToString()` and gets garbage/XML-escaped characters from a legacy producer. Why?
**Answer**
Legacy SDKs (WindowsAzure.ServiceBus/Microsoft.Azure.ServiceBus) serialized with DataContractSerializer or a different AMQP body type; the new SDK exposes raw bytes. Fix by checking `ContentType`/body type and decoding accordingly or by using `msg.GetRawAmqpMessage()`.

**Explanation**
Mixed-SDK systems need an agreed envelope (JSON + ContentType). Don't assume UTF-8 text.

**Code / CLI / Bicep**
```csharp
var amqp = msg.GetRawAmqpMessage();
var bodyType = amqp.Body.BodyType; // Data / Value / Sequence
```

**Gotchas / follow-ups**
- Old SDK `GetBody<T>()` no longer exists; use `Body.ToObjectFromJson<T>()`.

**Interview traps**
- Assuming CorrelationId is unique.
- Trying to filter on body content.
- Forgetting TTL cannot exceed entity default.

**Cheat sheet**
- MessageId = dedup; CorrelationId = link; SessionId = group/order.
- Filters see properties only, not body.
- TTL expiry -> DLQ only if configured.
- Set ContentType always.

---

## Topic 4: Sending (ServiceBusClient / ServiceBusSender)

**Study notes**

- `ServiceBusClient` owns the AMQP connection; `ServiceBusSender` is a lightweight link per entity. Both are thread-safe and intended to be **long-lived singletons** (register in DI, e.g. `AddAzureClients`). Creating a client per message exhausts connections/sockets and is slow.
- Transport: AMQP over TCP (ports 5671/5672) by default; `AmqpWebSockets` uses 443 (useful behind restrictive firewalls/proxies).
- Prefer **batches** (`CreateMessageBatchAsync` + `TryAddMessage`) to reduce round trips; batch size is capped by the max message size.
- Scheduling: `ScheduleMessageAsync` returns a sequence number; `CancelScheduledMessageAsync` cancels before enqueue.

### 4.1 [Easy] How do you send a message with the .NET SDK?
**Answer**
Create one `ServiceBusClient` (with `DefaultAzureCredential`), get a `ServiceBusSender` for the queue/topic, and call `SendMessageAsync`.

**Explanation**
Use fully qualified namespace + credential rather than a connection string, so no secrets are stored.

**Code / CLI / Bicep**
```csharp
await using var client = new ServiceBusClient("sb-demo.servicebus.windows.net", new DefaultAzureCredential());
ServiceBusSender sender = client.CreateSender("orders");
await sender.SendMessageAsync(new ServiceBusMessage("hello"));
```

### 4.2 [Follow-up] How do you send many messages efficiently?
**Answer**
Use `ServiceBusMessageBatch`: create it, `TryAddMessage` until it returns false, send, then start a new batch.

**Explanation**
`SendMessagesAsync(IEnumerable)` throws if the set exceeds the size limit; the batch API enforces it for you. Batches are atomic from a single-send standpoint (all or nothing per send call).

**Code / CLI / Bicep**
```csharp
var batch = await sender.CreateMessageBatchAsync();
foreach (var m in messages)
{
    if (!batch.TryAddMessage(m))
    {
        await sender.SendMessagesAsync(batch);
        batch.Dispose();
        batch = await sender.CreateMessageBatchAsync();
        if (!batch.TryAddMessage(m)) throw new InvalidOperationException("Message too large");
    }
}
if (batch.Count > 0) await sender.SendMessagesAsync(batch);
batch.Dispose();
```

### 4.3 [Medium] How do scheduled messages work and how do you cancel one?
**Answer**
Set `ScheduledEnqueueTime` or call `ScheduleMessageAsync(msg, time)`; the message becomes visible at that time. Keep the returned sequence number and call `CancelScheduledMessageAsync` to cancel.

**Explanation**
Scheduled messages count in the `ScheduledMessages` metric. Cancel needs the sequence number, so persist it (e.g., in your DB) if you may cancel later. Typical uses: timeouts in sagas, delayed retries.

**Code / CLI / Bicep**
```csharp
long seq = await sender.ScheduleMessageAsync(msg, DateTimeOffset.UtcNow.AddMinutes(30));
await sender.CancelScheduledMessageAsync(seq);
```

### 4.4 [Hard] Why must the client be a singleton, and what about transport choices?
**Answer**
The client manages a pooled AMQP connection and token refresh; per-call creation causes connection churn, port exhaustion, throttling and latency. Use TCP by default and WebSockets only where 5671 is blocked.

**Explanation**
Dispose on app shutdown (`await using`/DI lifetime). Retries are built in via `ServiceBusRetryOptions`. A sender whose connection drops recovers transparently on the next operation.

**Code / CLI / Bicep**
```csharp
builder.Services.AddAzureClients(b =>
{
    b.AddServiceBusClientWithNamespace("sb-demo.servicebus.windows.net")
     .ConfigureOptions(o => o.TransportType = ServiceBusTransportType.AmqpWebSockets);
    b.UseCredential(new DefaultAzureCredential());
});
```

**Gotchas / follow-ups**
- Registering client as Transient in DI is a classic bug.
- In Functions, use the output binding or inject a singleton client rather than newing up per invocation.

### 4.5 [Tricky] A send throws after a timeout. Was the message sent?
**Answer**
Unknown: a timeout can occur after the broker accepted the message. Retrying can create duplicates, so either enable duplicate detection with a stable MessageId or make consumers idempotent.

**Explanation**
This is why at-least-once is the realistic contract. A stable business MessageId plus the duplicate detection window collapses retried sends.

**Code / CLI / Bicep**
```csharp
var msg = new ServiceBusMessage(body) { MessageId = $"order-{orderId}-created" };
```

**Gotchas / follow-ups**
- Duplicate detection only works within the configured window and the entity must have it enabled at creation.

**Interview traps**
- "Client per request is fine."
- Forgetting to dispose batches.
- Thinking scheduled messages can be cancelled without the sequence number.

**Cheat sheet**
- Singleton `ServiceBusClient`; `DefaultAzureCredential`.
- Batch with `TryAddMessage`.
- `ScheduleMessageAsync` -> sequence number -> `CancelScheduledMessageAsync`.
- WebSockets = port 443.

---

## Topic 5: Receiving – ServiceBusProcessor vs ServiceBusReceiver; receive modes

**Study notes**

- **ServiceBusProcessor**: event-driven, push-style. Handles concurrency, lock renewal, auto-complete, reconnects. Use for long-running services.
- **ServiceBusReceiver**: pull-style (`ReceiveMessagesAsync`, `PeekMessagesAsync`). Use for on-demand reads, DLQ tooling, deferred messages, batch pulls.
- Key processor options: `MaxConcurrentCalls` (parallel handlers), `AutoCompleteMessages` (default true), `PrefetchCount`, `MaxAutoLockRenewalDuration` (default 5 min), `ReceiveMode`.
- **PeekLock** (default): message locked, you settle it; at-least-once. **ReceiveAndDelete**: message deleted on receive; at-most-once, fastest, data loss on crash.

### 5.1 [Easy] How do you consume messages continuously?
**Answer**
Create a `ServiceBusProcessor`, register `ProcessMessageAsync` and `ProcessErrorAsync`, and call `StartProcessingAsync`.

**Explanation**
Error handler is mandatory; it receives SDK/transport errors and exceptions thrown by your handler (the latter also abandon the message).

**Code / CLI / Bicep**
```csharp
var processor = client.CreateProcessor("orders", new ServiceBusProcessorOptions
{
    MaxConcurrentCalls = 8,
    AutoCompleteMessages = true,
    PrefetchCount = 20,
    MaxAutoLockRenewalDuration = TimeSpan.FromMinutes(5)
});
processor.ProcessMessageAsync += async args =>
{
    var order = args.Message.Body.ToObjectFromJson<Order>();
    await Handle(order, args.CancellationToken);
};
processor.ProcessErrorAsync += args =>
{
    logger.LogError(args.Exception, "SB error {Source} {Entity}", args.ErrorSource, args.EntityPath);
    return Task.CompletedTask;
};
await processor.StartProcessingAsync();
```

### 5.2 [Follow-up] When use ServiceBusReceiver instead?
**Answer**
For ad-hoc/batch pulls, peeking, reading the DLQ, receiving deferred messages by sequence number, or when you need precise control of the loop.

**Explanation**
Receiver does not renew locks automatically; you call `RenewMessageLockAsync` yourself.

**Code / CLI / Bicep**
```csharp
var receiver = client.CreateReceiver("orders");
var msgs = await receiver.ReceiveMessagesAsync(maxMessages: 10, maxWaitTime: TimeSpan.FromSeconds(5));
foreach (var m in msgs) await receiver.CompleteMessageAsync(m);
```

### 5.3 [Medium] What do MaxConcurrentCalls and PrefetchCount do and how do they interact?
**Answer**
`MaxConcurrentCalls` = how many messages your handler processes in parallel. `PrefetchCount` = how many extra messages the client pulls into its local buffer to cut latency.

**Explanation**
Prefetched messages are *already locked*: their lock clock runs while they wait in the buffer. If handling is slow and prefetch is high, locks expire before you process (lock lost, DeliveryCount increments). Rule of thumb: keep prefetch modest relative to concurrency and processing time (start with 0 for slow handlers; verify guidance in current docs).

**Code / CLI / Bicep**
```csharp
new ServiceBusProcessorOptions { MaxConcurrentCalls = 4, PrefetchCount = 0 } // slow handlers
```

### 5.4 [Hard] PeekLock vs ReceiveAndDelete: trade-offs and when each is correct?
**Answer**
PeekLock gives at-least-once with explicit settlement; ReceiveAndDelete gives at-most-once with lower overhead but loses messages if the consumer crashes. Use ReceiveAndDelete only for loss-tolerant data.

**Explanation**
ReceiveAndDelete messages cannot be abandoned, dead-lettered, deferred, or have locks renewed. Sessions and transactions require PeekLock.

**Code / CLI / Bicep**
```csharp
var r = client.CreateReceiver("telemetry-q", new ServiceBusReceiverOptions { ReceiveMode = ServiceBusReceiveMode.ReceiveAndDelete });
```

**Gotchas / follow-ups**
- `PeekMessagesAsync` doesn't lock or change delivery count, and may show locked/deferred/scheduled messages.
- Peek is not a way to "read without consuming" for processing logic.

### 5.5 [Tricky] Handler throws; what happens with AutoCompleteMessages true vs false?
**Answer**
With true: success -> auto-complete; exception -> auto-abandon (delivery count +1, redelivered). With false: you must settle yourself; if you don't, the lock expires and it is redelivered.

**Explanation**
If you already settled manually in the handler while AutoComplete is true, the SDK skips auto-complete (it checks the settled state). Throwing after Complete doesn't undo it.

**Code / CLI / Bicep**
```csharp
processor.ProcessMessageAsync += async args =>
{
    try { await Handle(args.Message); await args.CompleteMessageAsync(args.Message); }
    catch (TransientException) { await args.AbandonMessageAsync(args.Message); }
    catch (Exception ex) { await args.DeadLetterMessageAsync(args.Message, "Unhandled", ex.Message); }
};
// with options.AutoCompleteMessages = false
```

**Gotchas / follow-ups**
- Unhandled exceptions loop until MaxDeliveryCount then DLQ.
- Always stop processor gracefully (`StopProcessingAsync`) to let in-flight handlers finish.

**Interview traps**
- Assuming prefetch is free; it burns lock time.
- Using ReceiveAndDelete "for speed" on business data.
- Omitting the error handler (SDK throws on Start).

**Cheat sheet**
- Processor = push + concurrency + renew; Receiver = pull + control.
- PeekLock default; RnD = at-most-once.
- AutoComplete true => success completes, exception abandons.
- Prefetch -> locked in buffer.

---

## Topic 6: Message settlement, locks, delivery count

**Study notes**

```
Received (PeekLock, lock timer starts; default LockDuration 1 min, max 5 min)
  |-- Complete   -> removed
  |-- Abandon    -> unlocked, DeliveryCount++ , redelivered
  |-- DeadLetter -> moved to DLQ
  |-- Defer      -> stays, only retrievable by SequenceNumber
  `-- lock expires -> same as Abandon (DeliveryCount++)
DeliveryCount > MaxDeliveryCount (default 10) -> DLQ (MaxDeliveryCountExceeded)
```

- Lock renewal: processor auto-renews up to `MaxAutoLockRenewalDuration`; receiver: `RenewMessageLockAsync`.
- The new SDK reports lock problems as `ServiceBusException` with `Reason == MessageLockLost` (old SDK: `MessageLockLostException`).

### 6.1 [Easy] What are the four settlement actions?
**Answer**
Complete (done), Abandon (release for retry), DeadLetter (park as poison), Defer (postpone and fetch later by sequence number).

**Explanation**
Only valid in PeekLock mode and only while holding the lock.

**Code / CLI / Bicep**
```csharp
await args.CompleteMessageAsync(args.Message);
await args.AbandonMessageAsync(args.Message, propertiesToModify: new Dictionary<string, object> { ["lastError"] = "timeout" });
await args.DeadLetterMessageAsync(args.Message, "ValidationFailed", "Missing field X");
await args.DeferMessageAsync(args.Message);
```

### 6.2 [Follow-up] What is lock duration and what happens if processing exceeds it?
**Answer**
Lock duration (default 1 min, max 5 min) is how long a message is invisible to others. If it expires, the message is redelivered and your settle call fails with `MessageLockLost`.

**Explanation**
Result: duplicate processing. Fix with auto lock renewal, shorter handlers, or splitting work. Configure per entity.

**Code / CLI / Bicep**
```bash
az servicebus queue create -g rg --namespace-name sb-demo -n orders --lock-duration PT2M --max-delivery-count 5
```

### 6.3 [Medium] How does MaxAutoLockRenewalDuration work?
**Answer**
The processor renews the lock in the background until `MaxAutoLockRenewalDuration` (default 5 min) elapses or the handler returns. Beyond that duration it stops renewing and the lock can expire.

**Explanation**
Set it to your worst-case processing time. Setting `TimeSpan.Zero` disables renewal. Renewal is best-effort: network failure can still lose the lock. In Functions the equivalent is `maxAutoLockRenewalDuration` in host.json.

**Code / CLI / Bicep**
```csharp
new ServiceBusProcessorOptions { MaxAutoLockRenewalDuration = TimeSpan.FromMinutes(20) }
```

### 6.4 [Hard] Delivery count and MaxDeliveryCount: what increments it, and what are the surprises?
**Answer**
Each delivery attempt (abandon, lock expiry, handler exception) increments `DeliveryCount`; exceeding `MaxDeliveryCount` auto-dead-letters. Peeking does not increment it.

**Explanation**
Prefetched-but-not-processed messages whose lock expires also count. Failures from dependency outages burn attempts, so a long outage can dead-letter healthy messages: pair with circuit breaker/backoff or set MaxDeliveryCount sensibly.

**Code / CLI / Bicep**
```csharp
if (args.Message.DeliveryCount >= 5)
    await args.DeadLetterMessageAsync(args.Message, "TooManyRetries", lastError);
```

**Gotchas / follow-ups**
- Changing MaxDeliveryCount affects only entity config; existing messages retain counts.
- Don't use DeliveryCount alone as a business retry counter across DLQ resubmits (a resubmitted copy starts at 1).

### 6.5 [Tricky] You get `MessageLockLost` on Complete after a successful handler. What now?
**Answer**
The work was done but the message will be redelivered; this is expected at-least-once behavior. Make the handler idempotent and treat the lock-lost on settle as a log-and-continue case.

**Explanation**
Root causes: handler > lock duration without renewal, high prefetch, CPU starvation, connection reset, or the processor being stopped. Fix the cause and add idempotency; don't retry Complete on a lost lock because it will never succeed.

**Code / CLI / Bicep**
```csharp
try { await args.CompleteMessageAsync(args.Message); }
catch (ServiceBusException ex) when (ex.Reason == ServiceBusFailureReason.MessageLockLost)
{
    logger.LogWarning("Lock lost for {Id}; will be redelivered", args.Message.MessageId);
}
```

**Gotchas / follow-ups**
- Settling via a different receiver than the one that received it fails.
- After a connection drop, locks are lost; messages return.

**Interview traps**
- "Abandon doesn't count as a delivery."
- Believing renewal is unbounded.
- Using `Defer` as a retry mechanism without storing sequence numbers.

**Cheat sheet**
- Lock default 1 min (max 5); MaxDeliveryCount default 10.
- Lock expiry == abandon.
- `MessageLockLost` => redelivery; be idempotent.
- Renew via `MaxAutoLockRenewalDuration`.

---

## Topic 7: Dead-letter queue (DLQ)

**Study notes**

- Every queue and subscription has a built-in sub-queue: `<entity>/$deadletterqueue`. It is not auto-purged and has no DLQ of its own.
- Reasons messages land there: `MaxDeliveryCountExceeded`, `TTLExpiredException` (if dead-lettering on expiration enabled), explicit `DeadLetterMessageAsync`, subscription filter evaluation exceptions, header size exceeded (verify list in current docs).
- Properties set: `DeadLetterReason`, `DeadLetterErrorDescription`.
- Read via a receiver with `SubQueue = SubQueue.DeadLetter`.

### 7.1 [Easy] What is a dead-letter queue and why is it useful?
**Answer**
A secondary sub-queue that holds messages that couldn't be delivered or processed, so poison messages don't block the main flow and can be inspected later.

**Explanation**
It is a parking lot, not a retry mechanism. Someone must monitor and act on it.

**Code / CLI / Bicep**
```csharp
await args.DeadLetterMessageAsync(args.Message, "InvalidSchema", "Field 'amount' missing");
```

### 7.2 [Follow-up] How do you read messages from the DLQ?
**Answer**
Create a receiver with `SubQueue.DeadLetter` (for subscriptions, pass topic and subscription), then receive/peek and inspect `DeadLetterReason`.

**Explanation**
You can also use Service Bus Explorer in the portal for manual inspection.

**Code / CLI / Bicep**
```csharp
var dlq = client.CreateReceiver("orders", new ServiceBusReceiverOptions { SubQueue = SubQueue.DeadLetter });
var msgs = await dlq.ReceiveMessagesAsync(10, TimeSpan.FromSeconds(5));
foreach (var m in msgs)
    Console.WriteLine($"{m.MessageId}: {m.DeadLetterReason} - {m.DeadLetterErrorDescription}");

// Subscription DLQ
var subDlq = client.CreateReceiver("order-events", "billing", new ServiceBusReceiverOptions { SubQueue = SubQueue.DeadLetter });
```

### 7.3 [Medium] How do you resubmit dead-lettered messages?
**Answer**
Receive from the DLQ, create a new `ServiceBusMessage` from it (copy body/properties), send to the original entity, then complete the DLQ message.

**Explanation**
Fix the root cause first or you re-poison the queue. A resubmitted message is new: DeliveryCount resets, and a new SequenceNumber is assigned. Optionally add an `ApplicationProperties["resubmitCount"]` to cap loops. For topics, resubmitting to the topic reaches *all* subscriptions; to target one subscription, send to that subscription's entity-specific path via a dedicated mechanism (e.g., a property used by a filter) - design carefully.

**Code / CLI / Bicep**
```csharp
var sender = client.CreateSender("orders");
await foreach (var dead in dlq.ReceiveMessagesAsync())
{
    var copy = new ServiceBusMessage(dead);          // copies body & app properties
    copy.ApplicationProperties["resubmitted"] = true;
    await sender.SendMessageAsync(copy);
    await dlq.CompleteMessageAsync(dead);
}
```

### 7.4 [Hard] Design poison-message handling and DLQ monitoring end to end.
**Answer**
Classify errors: transient -> abandon/backoff; permanent (validation) -> dead-letter immediately with reason; cap with MaxDeliveryCount. Alert on `DeadletteredMessages` metric and run a triage/resubmit tool.

**Explanation**
Don't rely on delivery-count exhaustion for permanent errors; it wastes capacity. Record correlation ID and error in `DeadLetterErrorDescription`. Add runbooks: inspect -> fix -> resubmit or discard. Avoid auto-resubmit loops.

**Code / CLI / Bicep**
```bicep
resource dlqAlert 'Microsoft.Insights/metricAlerts@2018-03-01' = {
  name: 'dlq-not-empty'
  location: 'global'
  properties: {
    severity: 2
    enabled: true
    scopes: [ ns.id ]
    evaluationFrequency: 'PT5M'
    windowSize: 'PT5M'
    criteria: {
      'odata.type': 'Microsoft.Azure.Monitor.SingleResourceMultipleMetricCriteria'
      allOf: [ {
        name: 'dlq'
        metricName: 'DeadletteredMessages'
        operator: 'GreaterThan'
        threshold: 0
        timeAggregation: 'Maximum'
        criterionType: 'StaticThresholdCriterion'
      } ]
    }
  }
}
```

**Gotchas / follow-ups**
- Dead-lettered messages do not expire automatically; treat the DLQ as unbounded growth (verify current TTL behavior in docs).
- Entity size quota includes DLQ; a growing DLQ can cause `QuotaExceeded` on sends.

### 7.5 [Tricky] Does the DLQ itself have a DLQ, and can you auto-forward the DLQ?
**Answer**
No DLQ for DLQ. You can auto-forward dead-lettered messages to another entity with `ForwardDeadLetteredMessagesTo`.

**Explanation**
Useful to centralize poison handling in one "triage" queue. The forwarded messages keep DeadLetter properties (verify).

**Code / CLI / Bicep**
```bicep
properties: { forwardDeadLetteredMessagesTo: 'triage-queue' }
```

**Gotchas / follow-ups**
- Target must be in the same namespace.
- Receiving from DLQ in PeekLock; abandoning there just returns it to DLQ.

**Interview traps**
- "DLQ messages are retried automatically."
- Ignoring that DLQ consumes entity quota.
- Not monitoring: silent data loss.

**Cheat sheet**
- Path: `entity/$deadletterqueue`; `SubQueue.DeadLetter`.
- Reasons: MaxDeliveryCountExceeded, TTL expiry, explicit, filter errors.
- Resubmit = send new copy + complete DLQ copy.
- Alert on `DeadletteredMessages` > 0.

---

## Topic 8: Sessions

**Study notes**

- A **session** groups messages by `SessionId`. A receiver accepts a session and gets an exclusive **session lock**; only it receives that session's messages, in order (FIFO within a session).
- Requires `RequiresSession = true` on the queue/subscription at creation; every message must then have a SessionId.
- **Session state** (up to ~256 KB Standard / ~1 MB Premium; verify) stores per-session data on the broker, handy for workflows.
- Use `ServiceBusSessionProcessor` with `MaxConcurrentSessions`, `MaxConcurrentCallsPerSession` (use 1 to preserve order), `SessionIdleTimeout`.

```
orders (RequiresSession)
  session A: a1 a2 a3 -> consumer 1 (locked)
  session B: b1 b2    -> consumer 2 (locked)
```

### 8.1 [Easy] What problem do sessions solve?
**Answer**
Ordered, grouped processing: all messages with the same SessionId go to one consumer sequentially, while different sessions run in parallel.

**Explanation**
Ordering per entity key (customer, order) without a single-consumer bottleneck.

**Code / CLI / Bicep**
```csharp
await sender.SendMessageAsync(new ServiceBusMessage(body) { SessionId = customerId });
```

### 8.2 [Follow-up] How do you receive from a session-enabled queue?
**Answer**
Use `CreateSessionProcessor` (or `AcceptNextSessionAsync` with a receiver) and handle messages through `ProcessSessionMessageAsync`.

**Explanation**
`MaxConcurrentCallsPerSession` = 1 keeps order; higher values break per-session ordering.

**Code / CLI / Bicep**
```csharp
var sp = client.CreateSessionProcessor("orders", new ServiceBusSessionProcessorOptions
{
    MaxConcurrentSessions = 8,
    MaxConcurrentCallsPerSession = 1,
    SessionIdleTimeout = TimeSpan.FromSeconds(30)
});
sp.ProcessMessageAsync += async args =>
{
    var stateBytes = await args.GetSessionStateAsync();
    // ... process args.Message in order ...
    await args.SetSessionStateAsync(BinaryData.FromString("step-2"));
};
sp.ProcessErrorAsync += a => Task.CompletedTask;
await sp.StartProcessingAsync();
```

### 8.3 [Medium] What is session state and when would you use it?
**Answer**
A small opaque blob stored on the broker per session, read/written through the session receiver. Use it for workflow progress or aggregation state without an external store.

**Explanation**
State persists until you clear it (set it to null/empty); clean it up on workflow completion. Size is limited; don't store large data.

**Code / CLI / Bicep**
```csharp
var receiver = await client.AcceptNextSessionAsync("orders");
var state = await receiver.GetSessionStateAsync();
await receiver.SetSessionStateAsync(BinaryData.FromObjectAsJson(new { step = 3 }));
```

### 8.4 [Hard] How do session locks behave, and what goes wrong?
**Answer**
The lock is on the session (renewed automatically by the session processor; `RenewSessionLockAsync` for manual receivers). If it is lost (`SessionLockLost`), another consumer can take the session and ordering/duplicates can occur.

**Explanation**
A session with no messages is released after `SessionIdleTimeout`. With few sessions and many consumers, some consumers idle: parallelism = number of active sessions. A huge number of sessions with a few consumers cycles through them.

**Code / CLI / Bicep**
```bash
az servicebus queue create -g rg --namespace-name sb-demo -n orders --enable-session true
```

**Gotchas / follow-ups**
- `RequiresSession` cannot be changed later.
- A message without SessionId sent to a session-enabled entity is rejected.
- Hot session (one huge customer) limits throughput to one consumer.

### 8.5 [Tricky] Functions with sessions: what must you configure?
**Answer**
Set `IsSessionsEnabled = true` on the trigger and tune `maxConcurrentSessions` in host.json. The function scales by sessions, and messages within a session are processed sequentially.

**Explanation**
In the isolated worker, you can bind `ServiceBusSessionMessageActions` to get/set session state (verify in current extension docs). Mixing sessions with high `maxConcurrentCalls` is wasteful; concurrency is per-session.

**Code / CLI / Bicep**
```csharp
[Function("ProcessOrder")]
public async Task Run(
    [ServiceBusTrigger("orders", Connection = "ServiceBusConnection", IsSessionsEnabled = true)]
    ServiceBusReceivedMessage message,
    ServiceBusMessageActions actions)
{
    await Handle(message);
    await actions.CompleteMessageAsync(message);
}
```

**Gotchas / follow-ups**
- Trigger `IsSessionsEnabled` against a non-session queue (or vice versa) fails at runtime.

**Interview traps**
- "Sessions guarantee order across the whole queue."
- Raising MaxConcurrentCallsPerSession and expecting order.
- Forgetting sessions must be enabled at entity creation.

**Cheat sheet**
- SessionId = ordering + affinity; one consumer holds session lock.
- `RequiresSession` is create-time.
- MaxConcurrentCallsPerSession = 1 to keep order.
- Session state = small broker-side blob.

---

## Topic 9: Duplicate detection and transactions

**Study notes**

- **Duplicate detection** (create-time flag `RequiresDuplicateDetection`): broker drops a message whose `MessageId` was already seen within the window (default 10 min, max 7 days). Applies to *sends*, not redelivery to consumers.
- **Transactions**: group operations (e.g., complete one message + send another) into one atomic unit using `TransactionScope` with `TransactionScopeAsyncFlowOption.Enabled`. Operations in one transaction must be on a single connection; to span entities, create the client with `EnableCrossEntityTransactions = true`.
- Transactions are scoped to Service Bus only; they don't include SQL (no distributed transaction) - use outbox for that.

### 9.1 [Easy] What does duplicate detection do?
**Answer**
It discards a send if a message with the same MessageId arrived within the detection window.

**Explanation**
Protects against producer retries. It doesn't make consumers exactly-once.

**Code / CLI / Bicep**
```bicep
properties: { requiresDuplicateDetection: true, duplicateDetectionHistoryTimeWindow: 'PT30M' }
```

### 9.2 [Follow-up] Can you enable duplicate detection on an existing queue?
**Answer**
No: `RequiresDuplicateDetection` is set at creation; you can change the window afterwards but not toggle the feature (verify in current docs).

**Explanation**
Plan ahead; recreate the entity otherwise.

**Code / CLI / Bicep**
```bash
az servicebus queue create -g rg --namespace-name sb-demo -n orders \
  --enable-duplicate-detection true --duplicate-detection-history-time-window PT30M
```

### 9.3 [Medium] How do you complete a message and send a new one atomically?
**Answer**
Wrap both in a `TransactionScope` (async flow enabled), using a client created with `EnableCrossEntityTransactions` when entities differ. Both succeed or both roll back.

**Explanation**
Typical pipeline stage: receive from `in`, send to `out`, complete from `in`. Settlement and sends must use the same client/connection. Receive itself isn't part of the transaction.

**Code / CLI / Bicep**
```csharp
var client = new ServiceBusClient(fqns, new DefaultAzureCredential(),
    new ServiceBusClientOptions { EnableCrossEntityTransactions = true });
var receiver = client.CreateReceiver("in");
var sender = client.CreateSender("out");

var msg = await receiver.ReceiveMessageAsync();
using (var ts = new TransactionScope(TransactionScopeAsyncFlowOption.Enabled))
{
    await sender.SendMessageAsync(new ServiceBusMessage(msg.Body));
    await receiver.CompleteMessageAsync(msg);
    ts.Complete();
}
```

### 9.4 [Hard] Why can't you make "update SQL + send message" atomic with Service Bus transactions, and what do you do?
**Answer**
Service Bus transactions don't enlist in DB distributed transactions (no MSDTC/2PC support). Use the outbox pattern (Topic 13) plus idempotent consumers.

**Explanation**
Dual-write problem: DB commit then send may crash in between. Outbox writes the message to a table in the same DB transaction, and a relay publishes it with a stable MessageId.

**Code / CLI / Bicep**
```csharp
// WRONG: not atomic
await db.SaveChangesAsync();
await sender.SendMessageAsync(msg); // crash here => lost event
```

**Gotchas / follow-ups**
- TransactionScope without `AsyncFlowOption.Enabled` throws or silently loses context across awaits.
- Transactions have a timeout and a max operations count (verify).

### 9.5 [Tricky] Duplicate detection is on, yet the consumer still processes duplicates. Why?
**Answer**
Detection only dedups *sends* within the window; redelivery from lock expiry/abandon, resubmits from DLQ, distinct MessageIds for the same business event, or sends outside the window all produce repeats.

**Explanation**
Also, with random GUID MessageIds, retried sends differ and aren't caught. Use business-key MessageIds and idempotent handlers.

**Code / CLI / Bicep**
```csharp
MessageId = $"{eventType}:{aggregateId}:{version}"
```

**Gotchas / follow-ups**
- Window tradeoff: longer windows increase history storage.
- Scheduled messages and duplicates: id is checked at send time.

**Interview traps**
- "Duplicate detection = exactly-once."
- Using TransactionScope across SQL and SB.
- Forgetting AsyncFlow option.

**Cheat sheet**
- Dedup: create-time flag, MessageId, window 10 min default.
- Transactions: single connection; `EnableCrossEntityTransactions`.
- No SQL+SB atomicity -> outbox.

---

## Topic 10: Topics, subscriptions, rules/filters, auto-forwarding

**Study notes**

- Each subscription has one or more **rules**: filter + optional action. A message is copied to a subscription if *any* rule matches.
- Default rule `$Default` = `TrueFilter` (everything). Remove it when you add specific rules or you'll get all messages.
- Filter types: `SqlRuleFilter` (SQL92-like over system and user properties), `CorrelationRuleFilter` (equality on properties: efficient), `TrueFilter`/`FalseFilter`.
- Actions (`SqlRuleAction`) can modify properties on the copy.
- **Auto-forward** (`ForwardTo`) chains an entity to a queue/topic in the same namespace; chains are limited in depth (verify; commonly 4 hops).

### 10.1 [Easy] How does a subscription filter work?
**Answer**
Each subscription's rules decide which topic messages it gets copied. Non-matching messages are not delivered to that subscription.

**Explanation**
Filters evaluate on properties, not body.

**Code / CLI / Bicep**
```bash
az servicebus topic subscription rule create -g rg --namespace-name sb-demo \
  --topic-name order-events --subscription-name eu-orders -n eu-only \
  --filter-sql-expression "region = 'EU'"
```

### 10.2 [Follow-up] SQL vs correlation filters?
**Answer**
Correlation filters do exact-match comparisons on specific properties and are faster; SQL filters allow expressions (`>`, `LIKE`, `IN`, AND/OR).

**Explanation**
Prefer correlation for high-volume routing. System properties in SQL use `sys.` prefix (e.g., `sys.Label`, `sys.To`).

**Code / CLI / Bicep**
```csharp
var admin = new ServiceBusAdministrationClient(fqns, new DefaultAzureCredential());
await admin.DeleteRuleAsync("order-events", "eu-orders", RuleProperties.DefaultRuleName);
await admin.CreateRuleAsync("order-events", "eu-orders", new CreateRuleOptions(
    "eu-only", new CorrelationRuleFilter { Subject = "OrderCreated", ApplicationProperties = { ["region"] = "EU" } }));
await admin.CreateRuleAsync("order-events", "vip", new CreateRuleOptions(
    "big", new SqlRuleFilter("amount > 1000 AND tier IN ('gold','platinum')"))
    { Action = new SqlRuleAction("SET priority = 1") });
```

### 10.3 [Medium] A new subscription receives everything even though you added a filter. Why?
**Answer**
The `$Default` TrueFilter rule is still present; any matching rule delivers, so it matches all. Delete it.

**Explanation**
Creating a subscription via SDK/CLI adds `$Default` unless you provide rules at creation. Bicep resource `rules` child can define them.

**Code / CLI / Bicep**
```bash
az servicebus topic subscription rule delete -g rg --namespace-name sb-demo \
  --topic-name order-events --subscription-name eu-orders -n '$Default'
```

### 10.4 [Hard] How does forwarding and chaining work, and what are the pitfalls?
**Answer**
`ForwardTo` makes the broker move messages from a subscription/queue to a destination entity automatically, enabling fan-in/regroup. Chains have depth limits, and the identity creating the forward needs send rights on the destination.

**Explanation**
Fan-in: many subscriptions -> one processing queue. Failed forwards (destination full/disabled) can pile messages in the source or its DLQ. Use `ForwardDeadLetteredMessagesTo` for DLQ aggregation.

**Code / CLI / Bicep**
```bicep
resource sub 'Microsoft.ServiceBus/namespaces/topics/subscriptions@2022-10-01-preview' = {
  parent: topic
  name: 'to-processing'
  properties: { forwardTo: 'processing-queue' }
}
```

**Gotchas / follow-ups**
- Forwarding preserves most properties; destination duplicate detection can apply.
- Destination must exist in the same namespace; don't create cycles.

### 10.5 [Tricky] Number of rules and subscriptions: any limits that bite?
**Answer**
Yes: per-topic subscription count and per-subscription rule count are limited (verify current numbers), and every message is evaluated against all rules, so many complex SQL rules cost throughput.

**Explanation**
Design with correlation filters, or a single "router" subscription feeding specialized queues.

**Code / CLI / Bicep**
```csharp
// Prefer: CorrelationRuleFilter { Subject = "OrderCreated" }
```

**Gotchas / follow-ups**
- A filter evaluation error (e.g., type mismatch) can dead-letter the message in the subscription's DLQ (verify reason).
- Missing property in SQL filter -> evaluates false/unknown, not an error.

**Interview traps**
- Not removing `$Default`.
- Thinking filters inspect the body.
- Assuming unlimited forward chain depth.

**Cheat sheet**
- Rule = filter (+ action); any-match delivers.
- Correlation > SQL for performance.
- `$Default` must be removed.
- `ForwardTo` same namespace, limited depth.

---

## Topic 11: Idempotency, delivery guarantees, ordering

**Study notes**

- Service Bus PeekLock = **at-least-once**. "Exactly-once" end to end is a myth; you get *effectively-once* via idempotent consumers (+ dedup on send).
- Idempotency techniques: natural idempotent ops (upsert), processed-message table keyed by MessageId (unique constraint), optimistic concurrency/version checks, idempotency keys passed downstream.
- **Ordering**: guaranteed FIFO only within a session (or a single non-partitioned queue with one consumer and no failures). Lost by: competing consumers, abandon/redelivery, retries, deferral, scheduling, multiple partitions, DLQ resubmits.

### 11.1 [Easy] Is Service Bus exactly-once?
**Answer**
No. PeekLock is at-least-once; duplicates occur on lock expiry, retries, and crashes between work and Complete. Make consumers idempotent.

**Explanation**
ReceiveAndDelete is at-most-once. Duplicate detection narrows send duplicates only.

**Code / CLI / Bicep**
```csharp
// at-least-once => handler may run twice for same MessageId
```

### 11.2 [Follow-up] How do you implement an idempotent consumer?
**Answer**
Record the MessageId (or business key) in the same DB transaction as the side effect, guarded by a unique constraint; skip if already processed.

**Explanation**
Check-then-act isn't enough without the constraint (race between concurrent duplicates). Add retention cleanup for the table.

**Code / CLI / Bicep**
```csharp
await using var tx = await db.Database.BeginTransactionAsync();
try
{
    db.Processed.Add(new Processed { MessageId = msg.MessageId });
    await ApplyBusinessChange(order);
    await db.SaveChangesAsync();
    await tx.CommitAsync();
}
catch (DbUpdateException ex) when (IsUniqueViolation(ex))
{
    // duplicate: already processed -> safe to complete
}
await args.CompleteMessageAsync(msg);
```

### 11.3 [Medium] When is ordering preserved and when is it lost?
**Answer**
Preserved within a session processed by one consumer sequentially. Lost with competing consumers on non-session queues, redelivery after abandon, deferral, multi-partition entities, and DLQ resubmit.

**Explanation**
Even in a session, a failed message that is abandoned is retried before the next (session processing is sequential), but if you dead-letter it, later messages proceed, so decide: stop the session or skip.

**Code / CLI / Bicep**
```csharp
new ServiceBusSessionProcessorOptions { MaxConcurrentCallsPerSession = 1 }
```

### 11.4 [Hard] Design ordered, idempotent processing of per-customer events.
**Answer**
Use SessionId = customerId, one call per session, a sequence/version number in the payload, and reject/park out-of-order or stale versions; handler is idempotent on (customerId, version).

**Explanation**
Sequence numbers from the broker aren't business versions. Use payload version to detect gaps, defer (with stored sequence numbers) or fail the session for re-sync.

**Code / CLI / Bicep**
```csharp
if (evt.Version <= lastAppliedVersion) { await args.CompleteMessageAsync(msg); return; } // dup/stale
if (evt.Version > lastAppliedVersion + 1) { await args.DeferMessageAsync(msg); return; }  // gap
```

**Gotchas / follow-ups**
- Deferred messages are invisible to normal receive; track sequence numbers (session state is a good place).
- Hot sessions serialize everything for that key.

### 11.5 [Tricky] A side effect (email) is not transactional. How do you avoid duplicate emails?
**Answer**
Pass an idempotency key to the provider if supported, or record "sent" state before/after with a state machine and accept a small residual risk; true exactly-once isn't possible across non-transactional systems.

**Explanation**
Order matters: record intent (pending) -> send -> mark sent; on retry, check state. Some duplicates may remain; document the guarantee.

**Code / CLI / Bicep**
```csharp
await emailClient.SendAsync(mail, idempotencyKey: msg.MessageId); // only if the provider supports it
```

**Gotchas / follow-ups**
- Retry of Complete after success is harmless only if the handler is idempotent.

**Interview traps**
- Claiming exactly-once.
- Dedup via in-memory cache (lost on restart/scale-out).
- Believing sessions fix ordering across retries with Abandon+concurrency>1.

**Cheat sheet**
- At-least-once + idempotency = effectively once.
- Unique constraint on MessageId.
- Ordering: sessions + 1 call/session.
- Ordering broken by competing consumers, defer, resubmit.

---

## Topic 12: Retries, backoff, deferral

**Study notes**

Two distinct retry layers:
1. **Client (transport) retry**: `ServiceBusRetryOptions` retries broker *operations* (send/receive/settle) on transient errors.
2. **Redelivery**: broker redelivers a message after Abandon/lock expiry up to MaxDeliveryCount. Your *business* retry logic lives here (or in app-level policies like Polly).

`ServiceBusRetryOptions`: `Mode` (Exponential/Fixed), `MaxRetries` (default 3), `Delay` (0.8 s default), `MaxDelay` (60 s default), `TryTimeout` (60 s default) - verify defaults.

Service Bus has no built-in delayed-redelivery on abandon (redelivery is immediate). Options: in-handler Polly delay (holds lock), re-send as a scheduled message with incremented retry property, or defer.

### 12.1 [Easy] What does ServiceBusRetryOptions control?
**Answer**
How the SDK retries transient failures of its own operations (send/receive/complete), with fixed or exponential backoff.

**Explanation**
It doesn't control message redelivery after your handler fails.

**Code / CLI / Bicep**
```csharp
var client = new ServiceBusClient(fqns, new DefaultAzureCredential(), new ServiceBusClientOptions
{
    RetryOptions = new ServiceBusRetryOptions
    {
        Mode = ServiceBusRetryMode.Exponential,
        MaxRetries = 5,
        Delay = TimeSpan.FromSeconds(1),
        MaxDelay = TimeSpan.FromSeconds(30),
        TryTimeout = TimeSpan.FromSeconds(60)
    }
});
```

### 12.2 [Follow-up] How do you get a delay between business retries?
**Answer**
Complete the failed message and re-send a copy as a scheduled message (with a retry counter and backoff), dead-lettering after N attempts.

**Explanation**
Abandon redelivers right away, causing hot-loop retries against a failing dependency. Scheduled re-send gives real backoff but changes SequenceNumber/order and requires atomicity (use a transaction for complete+schedule).

**Code / CLI / Bicep**
```csharp
int retry = args.Message.ApplicationProperties.TryGetValue("retry", out var r) ? (int)r : 0;
if (retry >= 5) { await args.DeadLetterMessageAsync(args.Message, "RetriesExhausted"); return; }
var next = new ServiceBusMessage(args.Message) { };
next.ApplicationProperties["retry"] = retry + 1;
await sender.ScheduleMessageAsync(next, DateTimeOffset.UtcNow.AddSeconds(Math.Pow(2, retry) * 10));
await args.CompleteMessageAsync(args.Message);
```

### 12.3 [Medium] What is deferral and when to use it?
**Answer**
Defer sets a message aside; it stays in the entity but can only be fetched by its SequenceNumber via `ReceiveDeferredMessageAsync`. Use it for out-of-order arrival (process later when its prerequisite arrives).

**Explanation**
You must remember the sequence number (DB or session state); deferred messages aren't returned by normal receive and aren't auto-retried. They still obey TTL and count toward size.

**Code / CLI / Bicep**
```csharp
long seq = args.Message.SequenceNumber;
await args.DeferMessageAsync(args.Message);
// later
var m = await receiver.ReceiveDeferredMessageAsync(seq);
await receiver.CompleteMessageAsync(m);
```

### 12.4 [Hard] Dependency is down for 20 minutes and MaxDeliveryCount=10: what happens and how do you protect messages?
**Answer**
Messages are retried immediately and exhaust their deliveries within seconds, landing in the DLQ. Use a circuit breaker that stops the processor (or abandons without hammering), scheduled backoff re-sends, or a larger MaxDeliveryCount with delays.

**Explanation**
A circuit breaker can call `StopProcessingAsync`, wait, then `StartProcessingAsync`, leaving messages in the queue without burning delivery counts. Alerting on DLQ shows the damage; resubmit afterward.

**Code / CLI / Bicep**
```csharp
if (breaker.IsOpen)
{
    await processor.StopProcessingAsync();
    _ = Task.Delay(TimeSpan.FromMinutes(2)).ContinueWith(_ => processor.StartProcessingAsync());
}
```

**Gotchas / follow-ups**
- Holding the lock while sleeping risks lock loss.
- Throttling (429-like `ServiceBusy`) differs from app failures.

### 12.5 [Tricky] You added Polly retries inside the handler plus SDK retries plus MaxDeliveryCount=10. Why is it a problem?
**Answer**
Retries multiply (3 Polly x N SDK ops x 10 deliveries) and can exceed lock duration, causing lock loss, duplicate processing, and load amplification on the failing dependency.

**Explanation**
Budget total time per delivery below lock duration/renewal, and use one layer for business retry.

**Code / CLI / Bicep**
```csharp
// Keep inner retries short; total time << MaxAutoLockRenewalDuration
```

**Gotchas / follow-ups**
- Defer isn't retry; deferred messages need explicit pickup.

**Interview traps**
- Confusing SDK retry with redelivery.
- Expecting abandon to apply a delay.
- Forgetting to persist deferred sequence numbers.

**Cheat sheet**
- Two layers: SDK op retry vs broker redelivery.
- Backoff = scheduled re-send or circuit breaker.
- Defer -> `ReceiveDeferredMessageAsync(seq)`.

---

## Topic 13: Request/response, sagas, outbox

**Study notes**

- **Request/reply**: request has `ReplyTo` (queue) and optional `ReplyToSessionId`; reply carries `CorrelationId = request.MessageId`. Use a session-enabled reply queue to route replies to the right requester instance.
- **Saga**: long-running workflow across services. *Choreography* (events) or *orchestration* (coordinator with state + compensation). Libraries: MassTransit, NServiceBus, Durable Functions.
- **Outbox**: write business data + outgoing message in one local DB transaction; a relay publishes and marks sent. Result: at-least-once publish; pair with idempotent consumers.

```
App TX: [orders row] + [outbox row]  -> COMMIT
Relay: poll outbox -> send to SB (MessageId = outbox.Id) -> mark sent
```

### 13.1 [Easy] How do you do request/response over Service Bus?
**Answer**
Send with `ReplyTo`, consumer replies to that queue with `CorrelationId` set to the request's MessageId, and the requester matches on it.

**Explanation**
Use `ReplyToSessionId` + session-enabled reply queue so each requester instance only gets its replies.

**Code / CLI / Bicep**
```csharp
var req = new ServiceBusMessage(body) { MessageId = Guid.NewGuid().ToString(), ReplyTo = "replies", ReplyToSessionId = instanceId };
await sender.SendMessageAsync(req);
var rx = await client.AcceptSessionAsync("replies", instanceId);
var reply = await rx.ReceiveMessageAsync(TimeSpan.FromSeconds(30));
```

### 13.2 [Follow-up] What is a saga and how do you handle failure?
**Answer**
A saga coordinates multi-step business transactions without distributed locks; each step has a compensating action executed on failure.

**Explanation**
Orchestration keeps flow in one place (state machine stored durably); choreography reduces coupling but can be hard to trace. Use timeouts via scheduled messages and always make steps idempotent.

**Code / CLI / Bicep**
```csharp
// Timeout for a saga step
await sender.ScheduleMessageAsync(new ServiceBusMessage($"timeout:{sagaId}"), DateTimeOffset.UtcNow.AddMinutes(15));
```

### 13.3 [Medium] Implement the outbox pattern with SQL + Service Bus.
**Answer**
Insert an outbox row in the same transaction as the entity change; a background relay reads unsent rows, publishes with MessageId = row id, then marks them sent.

**Explanation**
Relay crash after send but before marking -> republish; duplicate detection (same MessageId within window) and idempotent consumers absorb it. Use row locking (`UPDLOCK, READPAST`) for multiple relay instances.

**Code / CLI / Bicep**
```csharp
// In the business transaction
db.Orders.Add(order);
db.Outbox.Add(new OutboxMessage { Id = Guid.NewGuid(), Type = "OrderCreated", Payload = json });
await db.SaveChangesAsync();

// Relay
var batch = await db.Outbox.Where(o => o.SentAt == null).OrderBy(o => o.CreatedAt).Take(50).ToListAsync();
foreach (var o in batch)
{
    await sender.SendMessageAsync(new ServiceBusMessage(o.Payload) { MessageId = o.Id.ToString(), Subject = o.Type });
    o.SentAt = DateTime.UtcNow;
}
await db.SaveChangesAsync();
```

### 13.4 [Hard] Outbox relay scaling and ordering?
**Answer**
Multiple relays need exclusive claiming (SQL `UPDLOCK, READPAST` or a lease), and per-aggregate ordering requires keeping order per key (e.g., SessionId = aggregate id, ordered by sequence).

**Explanation**
Alternatives: CDC/Change feed (Cosmos), MassTransit/NServiceBus transactional outbox. Cleanup old sent rows to keep the table small.

**Code / CLI / Bicep**
```sql
SELECT TOP (50) * FROM Outbox WITH (UPDLOCK, READPAST, ROWLOCK)
WHERE SentAt IS NULL ORDER BY CreatedAt;
```

**Gotchas / follow-ups**
- Polling latency vs DB load.
- Poison outbox rows (bad payload) must not block the relay: track attempt count.

### 13.5 [Tricky] Reply arrives but the requester restarted. What breaks?
**Answer**
A requester that used non-session reply queues or in-memory correlation loses/mismatches replies. Persist correlation state or use a durable workflow with TTL on replies.

**Explanation**
Don't use temporary per-request queues in production at scale (cost/cleanup); use a shared reply queue with sessions or an async callback model.

**Code / CLI / Bicep**
```csharp
// Persist pending requests: (MessageId, instanceId, deadline) so a restarted instance can resume.
```

**Gotchas / follow-ups**
- Set TTL on reply messages to avoid orphaned replies filling the queue.

**Interview traps**
- "Use a distributed transaction for DB + SB."
- Forgetting idempotency in sagas.
- Relay doesn't handle poison rows.

**Cheat sheet**
- Reply: ReplyTo + CorrelationId (+ session).
- Saga: compensation + timeouts (scheduled msgs).
- Outbox: same-TX insert, relay publishes with stable MessageId.

---

## Topic 14: Service Bus + Azure Functions (isolated worker)

**Study notes**

- Trigger `[ServiceBusTrigger(queue or topic, subscription, Connection = "Name")]`. Identity-based connection uses app settings `Name__fullyQualifiedNamespace` (plus optional `Name__credential`, `Name__clientId` for UAMI).
- Isolated worker binds to `ServiceBusReceivedMessage` (or string/byte[]/batch `ServiceBusReceivedMessage[]`) and `ServiceBusMessageActions` for manual settlement (set `autoCompleteMessages: false` in host.json).
- host.json `extensions.serviceBus` (Functions Service Bus extension 5.x): `maxConcurrentCalls`, `maxConcurrentSessions`, `maxMessageBatchSize`, `prefetchCount`, `maxAutoLockRenewalDuration`, `autoCompleteMessages`, `sessionIdleTimeout`, `enableCrossEntityTransactions` (verify names/defaults for your extension version).
- Success -> auto-complete; exception -> abandon -> redelivery -> DLQ at MaxDeliveryCount.
- Scaling: Functions scale controller scales instances by queue length (Consumption/Premium); Dedicated plans don't autoscale by queue unless configured; Flex Consumption scales per its own rules (verify).

### 14.1 [Easy] Write a Service Bus triggered function (isolated).
**Answer**
Use `[ServiceBusTrigger]` with a queue name and a `Connection` setting, taking a message parameter.

**Explanation**
The function host handles receive, lock renewal, settlement.

**Code / CLI / Bicep**
```csharp
public class OrderFn(ILogger<OrderFn> log)
{
    [Function("OrderFn")]
    public void Run([ServiceBusTrigger("orders", Connection = "ServiceBusConnection")] ServiceBusReceivedMessage msg)
        => log.LogInformation("Got {Id}: {Body}", msg.MessageId, msg.Body);
}
```

### 14.2 [Follow-up] How do you configure identity-based connection?
**Answer**
Replace the connection string with `ServiceBusConnection__fullyQualifiedNamespace`, and grant the Function's identity *Azure Service Bus Data Receiver* (and *Data Sender* for output).

**Explanation**
For SAMI nothing else; for UAMI add `__credential=managedidentity` and `__clientId=<uami client id>`.

**Code / CLI / Bicep**
```bash
az functionapp config appsettings set -g rg -n func-demo --settings \
  "ServiceBusConnection__fullyQualifiedNamespace=sb-demo.servicebus.windows.net" \
  "ServiceBusConnection__credential=managedidentity" \
  "ServiceBusConnection__clientId=<uami-client-id>"
```

### 14.3 [Medium] Manual settlement and batch triggers?
**Answer**
Set `autoCompleteMessages` false, inject `ServiceBusMessageActions`, and settle explicitly. For batches use `IsBatched = true` and a `ServiceBusReceivedMessage[]` parameter.

**Explanation**
Batch mode: if the function throws, the whole batch is abandoned unless you settled individually; settle each message to avoid reprocessing successes.

**Code / CLI / Bicep**
```csharp
[Function("Batch")]
public async Task Run(
    [ServiceBusTrigger("orders", Connection = "ServiceBusConnection", IsBatched = true)] ServiceBusReceivedMessage[] msgs,
    ServiceBusMessageActions actions)
{
    foreach (var m in msgs)
    {
        try { await Handle(m); await actions.CompleteMessageAsync(m); }
        catch (Exception ex) { await actions.DeadLetterMessageAsync(m, deadLetterReason: "HandlerFailed", deadLetterErrorDescription: ex.Message); }
    }
}
```
```json
{ "version": "2.0", "extensions": { "serviceBus": {
    "autoCompleteMessages": false,
    "maxConcurrentCalls": 16,
    "maxAutoLockRenewalDuration": "00:10:00",
    "prefetchCount": 0,
    "maxMessageBatchSize": 100 } } }
```

### 14.4 [Hard] Tune concurrency and avoid downstream overload.
**Answer**
Total concurrency = instances x `maxConcurrentCalls`. Lower `maxConcurrentCalls`, cap instances (`WEBSITE_MAX_DYNAMIC_APPLICATION_SCALE_OUT` / function app scale limit / Premium max burst), and match to downstream capacity.

**Explanation**
Defaults are high; a backlog can fan out and flood a database. Slow handlers need `maxAutoLockRenewalDuration` >= worst case, else lock loss -> duplicates. Sessions: concurrency is `maxConcurrentSessions`.

**Code / CLI / Bicep**
```bash
az functionapp config appsettings set -g rg -n func-demo --settings WEBSITE_MAX_DYNAMIC_APPLICATION_SCALE_OUT=5
```

**Gotchas / follow-ups**
- Scale-out limit setting differs by plan (verify in current docs).
- Lock lost errors in logs signal timing misconfiguration.

### 14.5 [Tricky] Function succeeded but message processed twice. Why?
**Answer**
Typically lock expiry during long runs, host shutdown/scale-in mid-processing, or manual-settlement paths that skip Complete; at-least-once applies.

**Explanation**
Check `DeliveryCount`, `maxAutoLockRenewalDuration`, and whether your code swallows exceptions (swallowing = auto-complete = silent loss!). Idempotency is the fix.

**Code / CLI / Bicep**
```csharp
catch (Exception ex) { log.LogError(ex, "fail"); /* swallowed => message completed, data lost */ throw; }
```

**Gotchas / follow-ups**
- Catch-and-log without rethrow hides failures from retry/DLQ.
- Poison messages in Functions go to the DLQ automatically after MaxDeliveryCount.
- Output binding `[ServiceBusOutput]` sends after function completion, not transactionally with Complete.

**Interview traps**
- Swallowing exceptions.
- Forgetting the Data Receiver role.
- Assuming unlimited scale protects the database.

**Cheat sheet**
- Identity: `Conn__fullyQualifiedNamespace` (+ `__clientId` for UAMI).
- `ServiceBusMessageActions` + `autoCompleteMessages:false`.
- host.json: maxConcurrentCalls, maxAutoLockRenewalDuration, prefetchCount.
- Exceptions -> abandon -> DLQ.

---

## Topic 15: Service Bus + Logic Apps

**Study notes**

- Two connector families: **managed (shared) Service Bus connector** (Consumption and Standard) and **built-in connector** (Logic Apps Standard, runs in-process in the runtime; typically lower latency and supports managed identity/VNet). Verify capability differences in current docs.
- Triggers: "When a message is received in a queue/topic subscription" (auto-complete) vs (peek-lock). Peek-lock exposes a lock token: you then call **Complete / Abandon / Dead-letter** actions.
- Trigger polls; scale is governed by concurrency control and polling interval.
- Auth: connection string, or Managed Identity (grant Data Receiver/Sender).

### 15.1 [Easy] What triggers can Logic Apps use for Service Bus?
**Answer**
Queue and topic-subscription triggers, in auto-complete or peek-lock variants, plus batch variants.

**Explanation**
Auto-complete completes upon successful trigger firing (before your workflow finishes), so later workflow failures lose the message; peek-lock lets you settle at the end.

**Code / CLI / Bicep**
```json
{ "type": "ServiceProvider", "kind": "Trigger", "inputs": { "serviceProviderConfiguration": { "operationId": "receiveQueueMessages", "serviceProviderId": "/serviceProviders/serviceBus" }, "parameters": { "queueName": "orders" } } }
```
(Illustrative shape; verify the exact operationId in the designer's code view.)

### 15.2 [Follow-up] Auto-complete vs peek-lock: which to choose?
**Answer**
Peek-lock when the workflow must succeed before the message is considered done; auto-complete only for loss-tolerant or very fast flows.

**Explanation**
Peek-lock requires explicit Complete; lock duration limits workflow time unless you renew. Failures -> Abandon/Dead-letter action in a failure scope.

**Code / CLI / Bicep**
```text
Trigger(peek-lock) -> Scope(try: process) -> [success] Complete message
                                          -> [failure, runAfter Failed] Dead-letter message
```

### 15.3 [Medium] How do you control throughput and ordering in Logic Apps?
**Answer**
Use trigger concurrency (degree of parallelism; 1 for sequential) and session-enabled triggers for per-session order.

**Explanation**
Concurrency control on a trigger can't be turned off once enabled, and changes some behaviors (verify). Per-run debatching and run limits apply; long workflows with peek-lock risk lock expiry.

**Code / CLI / Bicep**
```json
"runtimeConfiguration": { "concurrency": { "runs": 1 } }
```

### 15.4 [Hard] Message processed twice in a Logic App; where do you look?
**Answer**
Check peek-lock lock expiry versus run duration, trigger retries, and whether auto-complete masks failures; add idempotency using MessageId.

**Explanation**
Run history shows trigger messages and `DeliveryCount`. For long processes, hand off to a durable workflow, or complete early and persist state.

**Code / CLI / Bicep**
```text
Condition: @equals(triggerBody()?['DeliveryCount'], 1)  -- use only for diagnostics, not for correctness
```

**Gotchas / follow-ups**
- Managed identity on managed connector needs role assignment at namespace/entity scope.
- Content is base64 in some trigger outputs; decode via `base64ToString(triggerBody()?['ContentData'])`.

**Interview traps**
- Auto-complete = at-most-once-ish loss on workflow failure.
- Forgetting Data Receiver role with MI.
- Using Logic Apps for very high throughput.

**Cheat sheet**
- Peek-lock + explicit settlement for reliability.
- Concurrency 1 = sequential.
- MI + RBAC preferred over connection string.

---

## Topic 16: Security (RBAC, SAS, private endpoints)

**Study notes**

- **Entra ID (RBAC)** roles: *Azure Service Bus Data Owner* (manage + send + receive), *Data Sender*, *Data Receiver*. Assign at namespace, topic/queue, or subscription scope (least privilege).
- **SAS**: shared access policies (Send/Listen/Manage) with keys or tokens; secrets that can leak; `RootManageSharedAccessKey` is all-powerful.
- Disable SAS with `disableLocalAuth: true` (then only Entra auth works).
- Network: Premium supports private endpoints, IP firewall rules, virtual network rules (Standard/Basic: public endpoint, no private link). Min TLS 1.2 setting. Customer-managed keys: Premium.
- Control plane roles (Contributor) don't grant data-plane send/receive.

### 16.1 [Easy] Entra ID vs SAS?
**Answer**
Entra auth uses identities and RBAC (no secrets, auditable, revocable); SAS uses shared keys that can leak and are hard to rotate. Prefer managed identity + RBAC and disable SAS.

**Explanation**
SAS is still useful for third parties with limited, expiring tokens.

**Code / CLI / Bicep**
```csharp
var client = new ServiceBusClient("sb-demo.servicebus.windows.net", new DefaultAzureCredential());
```

### 16.2 [Follow-up] Which role does a producer/consumer need?
**Answer**
Producer: Data Sender. Consumer: Data Receiver. Administration client (create rules, queues): Data Owner or custom role.

**Explanation**
Scope to the entity when possible. For SAMI/UAMI, assign to the principal ID.

**Code / CLI / Bicep**
```bicep
resource sender 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(ns.id, principalId, 'sender')
  scope: ns
  properties: {
    principalId: principalId
    principalType: 'ServicePrincipal'
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '69a216fc-b8fb-44d8-bc22-1f3c2cd27a39') // Data Sender
  }
}
// Data Receiver: 4f6d3b9b-027b-4f4c-9142-0e5a2a2247e0 ; Data Owner: 090c5cfd-751d-490a-894a-3ce6f1109419
```

### 16.3 [Medium] How do you disable SAS and lock down networking?
**Answer**
Set `disableLocalAuth = true`, then use Premium with private endpoint and disable public network access.

**Explanation**
Verify all clients (Functions, Logic Apps, KEDA, Explorer tools) use identity before disabling SAS; otherwise they fail with 401.

**Code / CLI / Bicep**
```bicep
properties: { disableLocalAuth: true, publicNetworkAccess: 'Disabled', minimumTlsVersion: '1.2' }
```
```bash
az servicebus namespace update -g rg -n sb-prem --disable-local-auth true  # verify flag in current az version
```

### 16.4 [Hard] Role assigned but you still get 401/403/"Unauthorized".
**Answer**
Common causes: propagation delay (minutes), wrong scope, wrong identity (UAMI clientId not specified so DefaultAzureCredential picks another), control-plane role only, or private endpoint/firewall blocking (shows as connectivity error).

**Explanation**
Decode with `az role assignment list --assignee`. For UAMI set `ManagedIdentityClientId` in `DefaultAzureCredentialOptions`.

**Code / CLI / Bicep**
```csharp
var cred = new DefaultAzureCredential(new DefaultAzureCredentialOptions { ManagedIdentityClientId = uamiClientId });
```
```bash
az role assignment list --assignee <principalId> --scope <nsId> -o table
```

**Gotchas / follow-ups**
- Local dev: your user needs the data role too (Owner of subscription != data access).
- Private DNS zone `privatelink.servicebus.windows.net` must resolve; AMQP needs ports 5671/5672 (or 443 for WebSockets).

### 16.5 [Tricky] Private endpoint is set up; Standard-tier namespace?
**Answer**
Not possible: private endpoints and IP firewall are Premium features. Upgrade tier (migration via new namespace; verify in-place options) or accept public endpoint with Entra auth.

**Explanation**
Tier cannot always be changed in place Standard <-> Premium; plan migration (create Premium namespace, move entities, cut over producers/consumers).

**Code / CLI / Bicep**
```bash
az network private-endpoint create -g rg -n pe-sb --vnet-name vnet --subnet snet \
  --private-connection-resource-id <premiumNsId> --group-id namespace --connection-name sb-conn
```

**Interview traps**
- Believing Contributor can send/receive.
- Disabling local auth before migrating clients.
- Forgetting private DNS.

**Cheat sheet**
- Sender/Receiver/Owner roles; scope to entity.
- `disableLocalAuth` kills SAS.
- Private endpoint = Premium only.
- UAMI needs explicit clientId in credential.

---

## Topic 17: Geo-DR, geo-replication, availability zones

**Study notes**

- **Availability zones**: Premium namespaces can be zone-redundant (spread across zones in region); protects against zonal failure; no client change.
- **Geo-Disaster Recovery** (Premium): pairs a primary and secondary namespace under an **alias** (DNS CNAME). Replicates *metadata only* (entities, config), **not messages**. Failover is manual, breaks the pairing, and the alias then points to secondary; in-flight messages in the primary are not carried over.
- **Geo-replication** (Premium; newer feature, verify status/availability in current docs) replicates message data and state to a secondary region, with promotion.
- Use the alias connection endpoint in clients, so failover needs no config change.

```
clients -> alias.servicebus.windows.net -> primary ns (region A)
                                        -> [failover] secondary ns (region B)
```

### 17.1 [Easy] How do you protect Service Bus from a zone outage?
**Answer**
Use Premium with zone redundancy enabled.

**Explanation**
Must be set when creating the namespace in supported regions (verify whether it can change later).

**Code / CLI / Bicep**
```bicep
sku: { name: 'Premium', tier: 'Premium', capacity: 2 }
properties: { zoneRedundant: true }
```

### 17.2 [Follow-up] What does Geo-DR replicate?
**Answer**
Entity metadata and configuration only, not messages or session state.

**Explanation**
After failover, the secondary has the entities but not undelivered messages; data in the primary is unavailable until it recovers. Design producers to re-send and consumers to be idempotent.

**Code / CLI / Bicep**
```bash
az servicebus georecovery-alias set -g rg --namespace-name sb-primary --alias sb-alias --partner-namespace <secondaryNsId>
```

### 17.3 [Medium] How do you perform a failover and what happens afterward?
**Answer**
Run failover on the *secondary*; the pairing breaks and the alias points to the secondary. To return, re-pair (in the other direction) or create a new pairing.

**Explanation**
Failover is manual and meant for real disasters; test with drills. Clients using SAS strings must keep alias-based strings; identity-based clients use the alias FQDN and need RBAC on both namespaces.

**Code / CLI / Bicep**
```bash
az servicebus georecovery-alias fail-over -g rg --namespace-name sb-secondary --alias sb-alias
```

### 17.4 [Hard] Design for RTO/RPO with Service Bus.
**Answer**
Geo-DR gives low RTO for metadata but RPO of unprocessed messages is the entire backlog. For tighter RPO use geo-replication (verify) or dual-publish/active-passive with idempotent consumers and replay from source of truth (outbox).

**Explanation**
Outbox + idempotent consumers let you re-publish after failover. Keep RBAC and private endpoints/DNS replicated in secondary. Zone redundancy handles zonal issues; Geo-DR for regional.

**Code / CLI / Bicep**
```csharp
// Clients use alias endpoint; on failover no config change
new ServiceBusClient("sb-alias.servicebus.windows.net", cred);
```

**Gotchas / follow-ups**
- Geo-DR and geo-replication are alternative approaches; check whether they can coexist (verify).
- Secondary namespace capacity must match primary.
- Private-endpoint setups need an endpoint in the secondary region too.

### 17.5 [Tricky] After failover the secondary has queues but old messages are "missing". Bug?
**Answer**
Not a bug: Geo-DR doesn't copy messages. Recover by re-publishing from the outbox/source of truth, and by draining the old primary if it comes back.

**Explanation**
Document this expectation in the DR runbook; test regularly.

**Code / CLI / Bicep**
```bash
az servicebus georecovery-alias show -g rg --namespace-name sb-secondary --alias sb-alias
```

**Interview traps**
- "Geo-DR replicates messages."
- Using the namespace FQDN instead of the alias.
- Forgetting RBAC/PE in the secondary.

**Cheat sheet**
- AZ = zonal; Geo-DR = regional metadata; geo-replication = data (verify).
- Failover manual, breaks pairing.
- Always use alias FQDN.

---

## Topic 18: Monitoring and alerting

**Study notes**

Key Azure Monitor metrics: `ActiveMessages`, `DeadletteredMessages`, `ScheduledMessages`, `Messages`, `Size`, `IncomingMessages`, `OutgoingMessages`, `CompleteMessage`, `AbandonMessage`, `ThrottledRequests`, `ServerErrors`, `UserErrors`, `ActiveConnections`; Premium: `NamespaceCpuUsage`, `NamespaceMemoryUsage` (verify names in portal).
Dimension `EntityName` lets you split per queue/topic.
Diagnostic logs (send to Log Analytics): `OperationalLogs`, `RuntimeAuditLogs`, `VNetAndIPFilteringLogs`, `ApplicationMetricsLogs` (verify categories). Resource-specific tables e.g. `AZMSOperationalLogs`, `AZMSRuntimeAuditLogs`.
There's no native "oldest message age" metric; derive it by peeking the head and computing `EnqueuedTime` age, or via app telemetry (verify in current docs).

### 18.1 [Easy] Which metrics matter most?
**Answer**
Active messages (backlog), dead-lettered messages, scheduled messages, throttled requests, server errors, and size vs quota.

**Explanation**
Backlog trending up with steady incoming = consumers too slow. DLQ >0 = failures.

**Code / CLI / Bicep**
```bash
az monitor metrics list --resource <nsId> --metric ActiveMessages DeadletteredMessages \
  --filter "EntityName eq 'orders'" --interval PT1M --aggregation Maximum
```

### 18.2 [Follow-up] How do you alert on DLQ growth?
**Answer**
Metric alert on `DeadletteredMessages` > 0 (or growth), scoped by EntityName, routed to an action group.

**Explanation**
Use dynamic thresholds for noisy entities. Also alert on ActiveMessages high and ThrottledRequests > 0.

**Code / CLI / Bicep**
```bash
az monitor metrics alert create -g rg -n dlq-orders --scopes <nsId> \
  --condition "max DeadletteredMessages > 0" --window-size 5m --evaluation-frequency 5m \
  --action <actionGroupId>
```

### 18.3 [Medium] Enable diagnostic logs and query them.
**Answer**
Create a diagnostic setting to Log Analytics with the Service Bus log categories, then query `AZMS*` tables with KQL.

**Explanation**
Use resource-specific destination tables (`AZMSOperationalLogs`). Verify the exact column names in your workspace (e.g., `OperationName`, `Status`).

**Code / CLI / Bicep**
```bicep
resource diag 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = {
  name: 'sb-diag'
  scope: ns
  properties: {
    workspaceId: law.id
    logs: [ { categoryGroup: 'allLogs', enabled: true } ]
    metrics: [ { category: 'AllMetrics', enabled: true } ]
  }
}
```
```kusto
AZMSOperationalLogs
| where TimeGenerated > ago(1h)
| summarize count() by OperationName, Status
| order by count_ desc
```

### 18.4 [Hard] Correlate producer-to-consumer in App Insights.
**Answer**
The SDK propagates W3C trace context via message `Diagnostic-Id` application property; with OpenTelemetry/App Insights the processor creates linked activities, so end-to-end traces appear.

**Explanation**
Enable the Azure SDK ActivitySource (`Azure.Messaging.ServiceBus`) experimental switch if needed (verify current flags). Also log MessageId/CorrelationId consistently.

**Code / CLI / Bicep**
```csharp
AppContext.SetSwitch("Azure.Experimental.EnableActivitySource", true); // verify in current docs
builder.Services.AddOpenTelemetry().WithTracing(t => t.AddSource("Azure.Messaging.ServiceBus*")).UseAzureMonitor();
```

**Gotchas / follow-ups**
- Metric latency of 1-5 minutes; don't alert on single points.
- Log retention costs: filter noisy categories.

### 18.5 [Tricky] ActiveMessages is zero but messages seem delayed. Why?
**Answer**
They may be scheduled (`ScheduledMessages`), deferred, locked in-flight, or in the DLQ; Active only counts ready messages.

**Explanation**
`Messages` total includes all states. Check each metric and Service Bus Explorer.

**Code / CLI / Bicep**
```kusto
AzureMetrics
| where ResourceProvider == "MICROSOFT.SERVICEBUS" and MetricName in ("ActiveMessages","DeadletteredMessages","ScheduledMessages")
| summarize max(Maximum) by MetricName, bin(TimeGenerated, 5m)
```

**Interview traps**
- Expecting an "oldest message age" metric out of the box.
- Alerting only on active count.
- Forgetting EntityName splitting.

**Cheat sheet**
- Active, DLQ, Scheduled, Throttled, Size.
- Diagnostics -> Log Analytics, `AZMS*` tables.
- DLQ alert threshold >0.
- Trace context flows via Diagnostic-Id.

---

## Topic 19: Testing (emulator, mocking)

**Study notes**

- **Service Bus emulator**: local Docker-based emulator (with SQL Server dependency) for dev/CI. Entities are declared in a config JSON at startup; supports queues/topics/subscriptions and most data-plane ops; some features (e.g., management plane/dynamic creation, some filters, size limits) differ - verify in current docs.
- Connection string pattern for emulator includes `UseDevelopmentEmulator=true`.
- Unit-test with mocks: `ServiceBusSender`, `ServiceBusReceiver`, `ServiceBusClient` have protected constructors and virtual members, so Moq/NSubstitute can mock them. Build received messages with `ServiceBusModelFactory`.
- Integration tests: real namespace (isolated, cleaned up) is the most reliable.

### 19.1 [Easy] How do you create a `ServiceBusReceivedMessage` in a unit test?
**Answer**
Use `ServiceBusModelFactory.ServiceBusReceivedMessage(...)`, since the class has no public constructor.

**Explanation**
Specify body, messageId, deliveryCount, properties, etc.

**Code / CLI / Bicep**
```csharp
var msg = ServiceBusModelFactory.ServiceBusReceivedMessage(
    body: BinaryData.FromObjectAsJson(new Order { Id = 1 }),
    messageId: "m-1",
    deliveryCount: 1,
    properties: new Dictionary<string, object> { ["tenant"] = "contoso" });
```

### 19.2 [Follow-up] How do you mock a sender?
**Answer**
Mock `ServiceBusSender` (virtual methods) and verify `SendMessageAsync` calls; inject via an interface or the sender type.

**Explanation**
Wrap in your own interface for cleaner tests. Mocks don't reproduce broker behavior (locks, DLQ, duplicates).

**Code / CLI / Bicep**
```csharp
var sender = new Mock<ServiceBusSender>();
sender.Setup(s => s.SendMessageAsync(It.IsAny<ServiceBusMessage>(), It.IsAny<CancellationToken>()))
      .Returns(Task.CompletedTask);
await new Publisher(sender.Object).Publish(order);
sender.Verify(s => s.SendMessageAsync(It.Is<ServiceBusMessage>(m => m.MessageId == "order-1"), It.IsAny<CancellationToken>()), Times.Once);
```

### 19.3 [Medium] How do you unit test an Azure Function handler with settlement?
**Answer**
Mock `ServiceBusMessageActions` and pass a model-factory message; verify Complete/DeadLetter calls.

**Explanation**
`ServiceBusMessageActions` is mockable (protected ctor, virtual methods).

**Code / CLI / Bicep**
```csharp
var actions = new Mock<ServiceBusMessageActions>();
await fn.Run(msg, actions.Object);
actions.Verify(a => a.CompleteMessageAsync(msg, It.IsAny<CancellationToken>()), Times.Once);
```

### 19.4 [Hard] Use the emulator in CI; what are the gotchas?
**Answer**
Run the emulator container with its config and SQL dependency, point tests at the emulator connection string, and design tests knowing feature gaps versus the cloud.

**Explanation**
Entities must exist in the config file; tests needing runtime entity creation or advanced features need a real namespace. Wait for readiness before running tests.

**Code / CLI / Bicep**
```csharp
var cs = "Endpoint=sb://localhost;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=SAS_KEY_VALUE;UseDevelopmentEmulator=true;";
await using var client = new ServiceBusClient(cs);
```

**Gotchas / follow-ups**
- Behavior (lock timing, limits) can differ; never certify production behavior with the emulator alone.
- Don't commit real connection strings; emulator key is a well-known placeholder.

### 19.5 [Tricky] Tests pass with mocks but production loses messages. Why?
**Answer**
Mocks don't model lock expiry, delivery counts, duplicate detection, or ordering; those failure modes need integration/chaos tests.

**Explanation**
Add tests for idempotency (feed the same message twice), poison handling, and handler timeouts.

**Code / CLI / Bicep**
```csharp
await handler.Handle(msg); await handler.Handle(msg); // same MessageId twice
Assert.Equal(1, db.Orders.Count());
```

**Interview traps**
- Trying to `new ServiceBusReceivedMessage()`.
- Trusting emulator parity.
- No idempotency tests.

**Cheat sheet**
- `ServiceBusModelFactory` for messages.
- Mock Sender/Receiver/Actions.
- Emulator = local dev/CI; verify gaps.
- Test duplicates explicitly.

---

## Topic 20: Scaling consumers, claim-check, production scenarios

**Study notes**

- **Competing consumers**: scale by adding instances reading one queue. Throughput = instances x concurrency; ordering lost unless sessions.
- **KEDA** `azure-servicebus` scaler scales on `messageCount` (target per replica) for AKS/Container Apps/Functions on Kubernetes; use workload identity/pod identity for auth (verify TriggerAuthentication options).
- **Claim-check**: store large payload in Blob; message carries blob URI/id + metadata. Receiver downloads (via identity), processes, optionally deletes blob.

```
Producer -> Blob (payload) -> SB message { blobUri, contentType, size }
Consumer <- SB message -> Blob download -> process -> complete (+ lifecycle delete)
```

### 20.1 [Easy] How do you scale consumers?
**Answer**
Add more instances (competing consumers) and/or raise `MaxConcurrentCalls`; scale automatically on backlog with Functions or KEDA.

**Explanation**
Downstream limits (DB, API) cap useful concurrency. Sessions limit parallelism to number of active sessions.

**Code / CLI / Bicep**
```yaml
# KEDA ScaledObject (AKS)
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata: { name: orders-worker }
spec:
  scaleTargetRef: { name: orders-worker }
  minReplicaCount: 0
  maxReplicaCount: 20
  triggers:
  - type: azure-servicebus
    metadata:
      queueName: orders
      messageCount: "50"
      namespace: sb-demo
    authenticationRef: { name: sb-auth }
```

### 20.2 [Follow-up] Implement claim-check with Blob.
**Answer**
Upload payload to Blob, send a small message with the blob reference; consumer fetches the blob. Use identity auth for both.

**Explanation**
Needed for payloads above limits (256 KB Standard) or to keep messages small/fast. Handle orphan blobs with lifecycle policies and atomicity: upload first, then send (orphans are cheaper than dangling references).

**Code / CLI / Bicep**
```csharp
var blob = blobContainer.GetBlobClient($"{Guid.NewGuid()}.json");
await blob.UploadAsync(BinaryData.FromObjectAsJson(bigPayload));
var msg = new ServiceBusMessage { ContentType = "application/json" };
msg.ApplicationProperties["blobUri"] = blob.Uri.ToString();
await sender.SendMessageAsync(msg);

// consumer
var uri = new Uri((string)args.Message.ApplicationProperties["blobUri"]);
var payload = (await new BlobClient(uri, new DefaultAzureCredential()).DownloadContentAsync()).Value.Content;
```

### 20.3 [Medium] Production: messages stuck / backlog growing. Triage?
**Answer**
Check consumers are running and authorized, ActiveMessages vs incoming rate, handler latency, dependency throttling, lock-loss/abandon rates, and whether messages are scheduled/deferred/locked rather than active.

**Explanation**
Order: (1) is anything receiving (`OutgoingMessages`, host logs)? (2) auth/network errors? (3) handler slow -> tune concurrency/scale. (4) `AbandonMessage` high -> handler failing. (5) session starvation (few sessions). (6) disabled entity or subscription filter mismatch.

**Code / CLI / Bicep**
```bash
az servicebus queue show -g rg --namespace-name sb-demo -n orders \
  --query "{active:countDetails.activeMessageCount,dlq:countDetails.deadLetterMessageCount,sched:countDetails.scheduledMessageCount}"
```

### 20.4 [Hard] Production scenarios cheat table: symptoms -> causes -> fixes.
**Answer**
Map each symptom to the likely layer (producer, broker config, consumer, downstream) and fix the cause, not the symptom.

**Explanation**

| Symptom | Likely causes | Fix |
|---|---|---|
| Messages stuck / backlog | Consumers down, auth (401), slow handler, session starvation | Check processor logs/roles, scale, tune concurrency |
| Duplicate processing | Lock expiry, retries, resubmit, at-least-once | Idempotency table, renew locks, shorter handler |
| Lock lost | Handler > lock, high prefetch, CPU starvation | `MaxAutoLockRenewalDuration`, lower prefetch, offload work |
| DLQ growing | Permanent failures, dependency outage, TTL expiry, filter errors | Read `DeadLetterReason`, fix, resubmit; circuit breaker |
| Consumer slow | Low concurrency, downstream latency, sync-over-async | Parallelism, async I/O, scale out, batch |
| Ordering broken | Competing consumers, defer/abandon, no sessions, partitions | Sessions + 1 call/session, version checks |
| Quota exceeded | Entity size full (incl. DLQ/scheduled), namespace limits, connection limits | Drain/purge DLQ, raise max size, add consumers, review TTL |
| ServiceBusy/throttling | Standard tier shared limits, chatty clients | Batching, singleton clients, Premium |

**Code / CLI / Bicep**
```kusto
AZMSOperationalLogs
| where TimeGenerated > ago(1h) and Status != "Succeeded"
| summarize count() by OperationName, Status, bin(TimeGenerated, 5m)
```

**Gotchas / follow-ups**
- `QuotaExceeded` on send: sender gets `ServiceBusException` Reason `QuotaExceeded`; messages aren't dropped but producers stall (use the outbox).
- Purging: receive-and-delete loop in DLQ is the usual way (no native purge; verify).

### 20.5 [Tricky] Backlog of 1M messages: scaling out makes the database fall over. What now?
**Answer**
The queue is the buffer; cap concurrency to downstream capacity (max replicas, `maxConcurrentCalls`), batch writes, and let the backlog drain gradually rather than maximizing consumers.

**Explanation**
Autoscaling on queue length ignores downstream health; add rate limiting/circuit breaker and use KEDA max replicas.

**Code / CLI / Bicep**
```json
{ "extensions": { "serviceBus": { "maxConcurrentCalls": 4 } } }
```

**Gotchas / follow-ups**
- Messages with short TTL expire during a long drain (and may be dead-lettered).
- Aged backlog may violate ordering/business freshness; consider skip logic.

**Interview traps**
- "Just add more instances."
- Ignoring DLQ/scheduled counts toward quota.
- Claim-check without cleanup or auth.

**Cheat sheet**
- Competing consumers + KEDA `messageCount`.
- Claim-check: upload first, send pointer.
- Triage: consumers -> auth -> latency -> abandon rate -> messages states.
- Cap concurrency to downstream.
