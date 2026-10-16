# Z13 – Azure Architecture Scenarios (25 Designs) for .NET Developers

## How to use this file
- Pick a design, cover the **Requirements** aloud first (interviewers reward clarifying questions), draw the diagram, then walk Services → Identity → Failure handling → Scale/Cost → Observability.
- Practice the **follow-up ladder** (Easy → Follow-up → Hard → Tricky) out loud; model answers are short on purpose.
- Limits, SKUs and preview features change. Anything marked **(verify in current docs)** must be checked at learn.microsoft.com before quoting numbers.
- Conventions: `SAMI` = system-assigned managed identity, `UAMI` = user-assigned managed identity. All code uses `DefaultAzureCredential` (local dev = your Entra login; Azure = managed identity). Snippets are illustrative, trimmed for brevity.
- Common baseline for every design: no secrets in config (use identity + RBAC), private endpoints for PaaS data planes in production, diagnostic settings to Log Analytics, Bicep/IaC for everything.

## Table of Contents
1. [Order processing: Service Bus + Functions + SQL + outbox](#design-1-order-processing-service-bus--functions--sql--outbox)
2. [File ingestion: Blob → Event Grid → Functions → SQL with dedupe](#design-2-file-ingestion-blob--event-grid--functions--sql-with-dedupe)
3. [Partner integration: Logic Apps + Service Bus + Blob](#design-3-partner-integration-logic-apps--service-bus--blob)
4. [Zero-secret microservice: UAMI + Key Vault + Service Bus + Redis](#design-4-zero-secret-microservice-uami--key-vault--service-bus--redis)
5. [Multi-region active-passive failover](#design-5-multi-region-active-passive-failover)
6. [Notification fan-out with topics/subscriptions/filters](#design-6-notification-fan-out-with-topicssubscriptionsfilters)
7. [Saga with Durable Functions (booking/payment/inventory)](#design-7-saga-with-durable-functions-bookingpaymentinventory)
8. [Claim-check pattern for large messages](#design-8-claim-check-pattern-for-large-messages)
9. [Scheduled batch/ETL: Timer + Durable fan-out/fan-in](#design-9-scheduled-batchetl-timer--durable-fan-outfan-in)
10. [Webhook receiver: Functions + Service Bus buffering + signature validation](#design-10-webhook-receiver-functions--service-bus-buffering--signature-validation)
11. [API rate-limiting + caching layer: APIM + Redis](#design-11-api-rate-limiting--caching-layer-apim--redis)
12. [Session/ordered processing per customer with Service Bus sessions](#design-12-sessionordered-processing-per-customer-with-service-bus-sessions)
13. [Dead-letter handling and replay tooling](#design-13-dead-letter-handling-and-replay-tooling)
14. [Cache-aside with Redis + stampede protection for catalog API](#design-14-cache-aside-with-redis--stampede-protection-for-catalog-api)
15. [Audit/event log pipeline to Log Analytics/Event Hubs](#design-15-auditevent-log-pipeline-to-log-analyticsevent-hubs)
16. [Document processing pipeline (Blob + Functions + queue + status API)](#design-16-document-processing-pipeline-blob--functions--queue--status-api)
17. [Email/SMS notification system with retry & throttling](#design-17-emailsms-notification-system-with-retry--throttling)
18. [Multi-tenant SaaS with per-tenant identity & data isolation](#design-18-multi-tenant-saas-with-per-tenant-identity--data-isolation)
19. [Secrets rotation automation (Key Vault + Event Grid + Functions)](#design-19-secrets-rotation-automation-key-vault--event-grid--functions)
20. [Blue-green/slot deployment with config & feature flags](#design-20-blue-greenslot-deployment-with-config--feature-flags)
21. [Real-time dashboard with SignalR Service + Functions](#design-21-real-time-dashboard-with-signalr-service--functions)
22. [Migration of on-prem WCF/SSIS/Windows Service jobs to Azure](#design-22-migration-of-on-prem-wcfssiswindows-service-jobs-to-azure)
23. [Hybrid integration with on-prem via Logic Apps gateway/Service Bus](#design-23-hybrid-integration-with-on-prem-via-logic-apps-gatewayservice-bus)
24. [Cost-optimization review of an over-provisioned system](#design-24-cost-optimization-review-of-an-over-provisioned-system)
25. [Disaster recovery runbook design with RPO/RTO](#design-25-disaster-recovery-runbook-design-with-rporto)
26. [Bonus: Monitoring & alerting design for the whole platform](#design-26-monitoring--alerting-design-for-the-whole-platform)

> Note: the brief lists 25 named designs plus the platform monitoring design; that is Design 26 here (26 total, so nothing requested is missing).

---

## Design 1: Order processing: Service Bus + Functions + SQL + outbox

### Requirements & clarifying questions
- Accept orders via API, persist, then notify downstream (inventory, billing, email). No lost orders, no double-charge.
- Ask: peak orders/sec? Is ordering needed per customer/order? Max message size? Is SQL the system of record? Exactly-once expectations (answer: at-least-once + idempotent consumers)? RPO/RTO?

### Architecture
```
Client -> APIM -> Order API (App Service/Func HTTP)
                    |  one SQL transaction
                    v
            [Azure SQL: Orders + Outbox]
                    |  Outbox Publisher (Timer/Func or worker)
                    v
        [Service Bus Topic: order-events] --sub--> Inventory Func --> SQL
                                          --sub--> Billing Func
                                          --sub--> Email Func
                    DLQ per subscription -> alert + replay tool
```

### Service choices & WHY
- **Service Bus (not Storage Queue/Event Hubs):** DLQ, duplicate detection, sessions, topics, transactions, scheduled delivery. Event Hubs is a log for telemetry streams, not command/work queues.
- **Outbox table:** the DB write and the message publish cannot share a transaction (no distributed txn across SQL and SB). Outbox gives atomic "state + intent"; publisher is at-least-once.
- **Functions Service Bus trigger:** autoscale, built-in lock renewal, settlement. Rejected: Logic Apps for the hot path (cost per action, latency).

### Identity & security
- Order API: SAMI with `Azure Service Bus Data Sender` on the topic scope only if it publishes directly; here the **publisher** gets Sender, SQL access via Entra (contained DB user for the MI: `CREATE USER [func-publisher] FROM EXTERNAL PROVIDER`).
- Each consumer Function: its own SAMI with `Azure Service Bus Data Receiver` scoped to its subscription (not the namespace).
- Service Bus Premium + private endpoint, `disableLocalAuth: true`. SQL private endpoint, public access off.
- Key Vault only for third-party keys (payment gateway), via Key Vault references.

### Data flow & failure handling
- Outbox row: `Id GUID, Type, Payload, CreatedUtc, PublishedUtc NULL`. Publisher selects unpublished with `UPDLOCK, READPAST` (competing publishers safe), sends with `MessageId = Outbox.Id`, then marks published. Crash between send and mark => duplicate; **duplicate detection** on the topic (window e.g. 10 min, verify in current docs) plus consumer idempotency absorbs it.
- Consumer idempotency: `ProcessedMessages(MessageId PK)` inserted in the same transaction as the business change; PK violation => skip.
- Retries: `MaxDeliveryCount` (default 10) then DLQ. Transient errors throw (abandon -> retry); poison (validation) errors dead-letter immediately with a reason.
- Ordering: not guaranteed on a topic; if per-order ordering is required use sessions with `SessionId = OrderId` (see Design 12).

### Scaling & cost
- Functions scale on queue depth; cap with `maxConcurrentCalls` and `WEBSITE_MAX_DYNAMIC_APPLICATION_SCALE_OUT` to protect SQL. Premium SB namespace is pricey but needed for private endpoints/large messages; Standard is OK for dev. SQL serverless for non-prod.

### Observability
- Correlation: put `Diagnostic-Id`/`traceparent` flows automatically with the SB SDK + App Insights; add `OrderId` as custom dimension.
```kusto
// Outbox lag: oldest unpublished age (custom metric emitted by publisher)
customMetrics | where name == "OutboxOldestAgeSec" | summarize max(value) by bin(timestamp, 5m)
// Failed consumer executions
requests | where success == false and cloud_RoleName startswith "func-" | summarize count() by name, resultCode, bin(timestamp,15m)
```
- Alerts: DLQ message count > 0 (metric `DeadletteredMessages`), active messages growing for 10 min, outbox age > 2 min.

### Key code
```csharp
// Outbox publisher
await using var client = new ServiceBusClient("sb-orders.servicebus.windows.net", new DefaultAzureCredential());
await using var sender = client.CreateSender("order-events");
foreach (var row in unpublished)
{
    var msg = new ServiceBusMessage(BinaryData.FromString(row.Payload))
    { MessageId = row.Id.ToString(), Subject = row.Type, ContentType = "application/json",
      CorrelationId = row.CorrelationId };
    await sender.SendMessageAsync(msg);
    await MarkPublishedAsync(row.Id);
}
// Consumer
[Function("Billing")]
public async Task Run([ServiceBusTrigger("order-events", "billing", Connection = "SbConn")]
    ServiceBusReceivedMessage m, ServiceBusMessageActions actions)
{
    try { await _handler.HandleAsync(m.MessageId, m.Body); await actions.CompleteMessageAsync(m); }
    catch (ValidationException ex) { await actions.DeadLetterMessageAsync(m, null, "Validation", ex.Message); }
}
```
```json
// host.json
{ "version": "2.0", "extensions": { "serviceBus": { "maxConcurrentCalls": 16, "autoCompleteMessages": false, "maxAutoLockRenewalDuration": "00:05:00" } } }
```
```bicep
resource topic 'Microsoft.ServiceBus/namespaces/topics@2022-10-01-preview' = {
  parent: ns, name: 'order-events'
  properties: { requiresDuplicateDetection: true, duplicateDetectionHistoryTimeWindow: 'PT10M', defaultMessageTimeToLive: 'P7D' }
}
```
Identity-based connection settings: `SbConn__fullyQualifiedNamespace = sb-orders.servicebus.windows.net` (plus `SbConn__credential`/`clientId` for UAMI).

### Trade-offs
- Outbox adds latency and a publisher component vs. dual-write simplicity (which can lose messages). CDC/Debezium-style is an alternative; Service Bus transactions only span SB entities.

### Follow-up ladder
- **Easy:** Why a queue between API and processing? *Decouples load, gives retry/DLQ, absorbs spikes.*
- **Follow-up:** What if the API saves the order but SB is down? *Outbox: order committed, publisher retries later; user still gets 202/201.*
- **Hard:** How do you guarantee no double charge? *At-least-once delivery + idempotency key (OrderId/MessageId) enforced in DB and also passed to the payment provider's idempotency header.*
- **Tricky:** Duplicate detection is on—do you still need idempotency? *Yes: window is finite, only covers same MessageId within it, and doesn't cover redelivery after lock loss where processing already succeeded.*

---

## Design 2: File ingestion: Blob → Event Grid → Functions → SQL with dedupe

### Requirements & clarifying questions
- Partners drop CSV/JSON files; load rows into SQL. Ask: file size/volume, arrival pattern, duplicates/re-sent files, schema drift, latency SLA, partial-failure policy, PII?

### Architecture
```
Partner --(SFTP-enabled storage / SAS-less RBAC upload)--> [Blob: inbound]
   BlobCreated --> [Event Grid system topic] --> [Service Bus queue: file-events]
                                                     |
                                               Func (parse, stream)
                                      +--> [SQL staging] -> MERGE -> [SQL target]
                                      +--> move blob -> processed/ or quarantine/
```

### Service choices & WHY
- **Event Grid blob trigger (not polling blob trigger):** Event Grid is push, low latency; the classic blob trigger polls and is slow/unreliable on Consumption (Functions also supports an Event Grid-based blob trigger source: verify in current docs). 
- **Queue between Event Grid and Function:** buffering, DLQ, throttle SQL load.
- **Staging table + MERGE:** set-based, idempotent. Row-by-row inserts rejected (slow, partial state).

### Identity & security
- Function SAMI: `Storage Blob Data Contributor` on the container (read + move), SQL contained user with `db_datareader/writer` on staging only plus EXEC on the merge proc.
- Event Grid system topic delivering to SB uses its SAMI with `Azure Service Bus Data Sender`.
- Storage: disable shared key (`allowSharedKeyAccess:false`), private endpoint, firewall; partners via SFTP local users or scoped user-delegation SAS.

### Data flow & failure handling
- Dedupe at 3 levels: (1) file level: `FileLedger(BlobName, ETag/ContentMD5 hash PK)`; (2) message level: SB duplicate detection on `MessageId = eventId`; (3) row level: natural/business key in MERGE.
- Event Grid delivers at-least-once; retries with backoff then dead-letters to storage (configure dead-letter destination).
- Poison file: schema invalid -> move to `quarantine/` + dead-letter message with reason; don't retry forever.
- Stream with `StreamReader`/CsvHelper and `SqlBulkCopy` in batches; never `ReadAllText` big files.

### Scaling & cost
- Event-driven, scales to zero (Consumption/Flex). Large files: split into chunks and fan-out via Durable (Design 9). Lifecycle policy moves `processed/` to Cool/Archive.

### Observability
```kusto
traces | where customDimensions.FileName != "" | summarize rows=sum(toint(customDimensions.Rows)), files=dcount(tostring(customDimensions.FileName)) by bin(timestamp,1h)
```
- Alerts: files in `quarantine/` (Event Grid on BlobCreated for that prefix), ingestion lag (file age vs processed time), DLQ > 0.

### Key code
```csharp
var blob = new BlobClient(new Uri(url), new DefaultAzureCredential());
await using var stream = await blob.OpenReadAsync();
var props = (await blob.GetPropertiesAsync()).Value;   // ETag, ContentHash
if (!await _ledger.TryRegisterAsync(blob.Name, props.ETag.ToString())) return; // duplicate file
using var bulk = new SqlBulkCopy(conn) { DestinationTableName = "stg.Orders", BatchSize = 5000 };
```
```sql
MERGE dbo.Orders AS t USING stg.Orders AS s ON t.ExternalId = s.ExternalId
WHEN MATCHED AND t.RowVer < s.RowVer THEN UPDATE SET ...
WHEN NOT MATCHED THEN INSERT (...) VALUES (...);
```
```bicep
resource sub 'Microsoft.EventGrid/systemTopics/eventSubscriptions@2022-06-15' = {
  parent: topic, name: 'blob-to-sb'
  properties: { destination: { endpointType: 'ServiceBusQueue'
      properties: { resourceId: queue.id } }
    filter: { includedEventTypes: ['Microsoft.Storage.BlobCreated'], subjectBeginsWith: '/blobServices/default/containers/inbound/' } }
}
```
(Delivery with managed identity needs `deliveryWithResourceIdentity`; verify current syntax.)

### Trade-offs
- Extra queue hop adds complexity but gives control. ETag dedupe misses same content re-uploaded under new name -> use content hash if required.

### Follow-up ladder
- **Easy:** Why Event Grid vs polling? *Push, near-real-time, cheaper.*
- **Follow-up:** Partner uploads same file twice? *File ledger by hash + idempotent MERGE.*
- **Hard:** 5 GB file times out the Function? *Stream, split into chunk messages/Durable fan-out, or use Premium/Dedicated; Consumption has execution time limit (verify).* 
- **Tricky:** Event fires before upload finishes? *BlobCreated fires on commit (`PutBlob`/`PutBlockList`); for SFTP/ADLS use `FlushWithClose` event semantics; also validate with a `.done` marker if partner writes multi-step.*

---

## Design 3: Partner integration: Logic Apps + Service Bus + Blob

### Requirements & clarifying questions
- B2B partners send orders (HTTPS/SFTP/AS2); we transform and route to internal systems; must track every transaction. Ask: protocols, formats (X12/EDIFACT/JSON), SLAs, ack requirements, partner count growth, low-code ownership?

### Architecture
```
Partner --> APIM (auth, IP filter) --> Logic App Std (HTTP trigger)
                                          |-- validate/transform (Liquid/XSLT/Inline code)
                                          |-- archive raw -> [Blob: raw/yyyy/MM/dd]
                                          |-- publish -> [SB topic: partner-orders]
                                          '-- ack/response to partner
Internal consumers (Functions) <-- SB subscriptions; failures -> DLQ -> support
```

### Service choices & WHY
- **Logic Apps Standard:** 1,000+ connectors, B2B (EDI) with integration account features, designer for integration team, VNET integration, runs on App Service plan (predictable cost). Consumption tier rejected for high-volume (per-action billing) and weaker networking.
- **Service Bus** for durable handoff; **Blob** as raw archive (legal replay).
- Heavy custom logic -> call Azure Function; keep Logic App as orchestrator.

### Identity & security
- Logic App Standard SAMI: `Azure Service Bus Data Sender`, `Storage Blob Data Contributor` (raw container), Key Vault `Key Vault Secrets User` for partner API keys. Use built-in service-provider connectors (managed-identity capable) rather than managed API connectors where possible (verify per connector).
- APIM validates JWT/partner client certs; Logic App HTTP trigger restricted to APIM IPs/private endpoint; SAS key in trigger URL is **not** sufficient alone.

### Data flow & failure handling
- Scope + "Run after" for try/catch/finally; on failure write to DLQ-like Blob `failed/` and return 4xx/5xx. Configure retry policy on actions (exponential). Idempotency: partner's `InterchangeId` as SB MessageId + dedupe table.
- Async partner contract: return `202` + status URL for long processing.

### Scaling & cost
- Standard: pay for plan (WS1-3), scale out instances; stateless workflows for lower latency/cost. Watch per-action costs in Consumption; use `Request` trigger concurrency controls.

### Observability
- Enable Log Analytics diagnostics + Application Insights on Standard (`Workflows.Runtime.*` settings). Track `clientTrackingId` = partner transaction id.
```kusto
AzureDiagnostics | where ResourceType == "WORKFLOWS/RUNS" and status_s == "Failed" | summarize count() by workflowName_s, code_s, bin(TimeGenerated,1h)
```
- Alerts: failed runs > N, run latency P95, SB DLQ.

### Key code
```bicep
resource la 'Microsoft.Web/sites@2023-01-01' = { name: 'la-partner', kind: 'functionapp,workflowapp', identity: { type: 'SystemAssigned' }
  properties: { serverFarmId: plan.id, virtualNetworkSubnetId: subnet.id, siteConfig: { appSettings: [
     { name: 'serviceBus_fullyQualifiedNamespace', value: 'sb-partner.servicebus.windows.net' } ] } } }
```
```csharp
// Custom transform in a Function the workflow calls
[Function("MapPartnerOrder")]
public IActionResult Run([HttpTrigger(AuthorizationLevel.Function, "post")] HttpRequest req) { /* map & validate */ }
```

### Trade-offs
- Low-code speed vs. testability/versioning (workflow JSON in source control; use Standard + CI). Costly at very high volume -> move hot path to Functions.

### Follow-up ladder
- **Easy:** Why Logic Apps? *Connectors, designer, B2B.*
- **Follow-up:** Standard vs Consumption? *Standard: VNET, single-tenant, fixed cost, stateless; Consumption: multi-tenant per-action.*
- **Hard:** Partner retries create duplicates? *Idempotency key from interchange ID; dedupe table; SB duplicate detection.*
- **Tricky:** Logic App connection using API connection keeps breaking after secret rotation? *Use managed identity auth/built-in connectors so no secret exists to rotate.*

---

## Design 4: Zero-secret microservice: UAMI + Key Vault + Service Bus + Redis

### Requirements & clarifying questions
- Microservice (App Service/Container Apps/AKS) with zero connection strings. Ask: how many instances/services share the same access? Need to survive re-deploys without re-granting RBAC? Redis tier (Entra auth supported on Azure Cache for Redis / Azure Managed Redis—verify in current docs)?

### Architecture
```
 [Container App / App Service]  --UAMI "id-orders-prod"-->  Entra token
        |            |             |              |
        v            v             v              v
  Service Bus    Key Vault      Redis          Storage/SQL
 (Data Sender/   (Secrets User) (Data Owner/   (Blob Data
  Receiver)      RBAC mode      Contributor    Contributor)
                 + private EP    access policy)
```

### Service choices & WHY
- **UAMI instead of SAMI** here: the identity lifecycle is independent of the app; one identity shared by all slots/instances/regions; RBAC survives resource recreation (blue-green, IaC redeploy). SAMI is simpler for 1:1 resources and auto-cleans on delete. SAMI on slots differs from production slot identity - a common gotcha.
- **Key Vault** for the few remaining secrets/certs (third-party API keys, signing certs); RBAC permission model (not access policies).
- **Redis** for cache; Entra auth via `Microsoft.Azure.StackExchangeRedis` extension.

### Identity & security
| Resource | Role | Scope |
|---|---|---|
| Service Bus queue/topic | Azure Service Bus Data Sender/Receiver | entity |
| Key Vault | Key Vault Secrets User (read) / Certificates Officer for rotation job | vault |
| Redis | Data Owner/Contributor via Redis access policy assignment | cache |
| Storage | Storage Blob Data Contributor | container |
- Least privilege: separate UAMIs per trust boundary (e.g., `id-orders-api` vs `id-orders-worker`) rather than one god identity.
- Disable local auth on SB/Storage/Redis access keys where supported; firewall + private endpoints; Key Vault `enablePurgeProtection`, soft delete.

### Data flow & failure handling
- Token acquisition failure at startup: retry with backoff; `DefaultAzureCredential` caches tokens. RBAC propagation can take minutes after assignment -> deployment pipeline should retry, not fail permanently.
- Prefer Key Vault *references* in App Service/Functions settings (`@Microsoft.KeyVault(SecretUri=...)`) with `keyVaultReferenceIdentity` set to the UAMI (references default to SAMI otherwise).

### Scaling & cost
- Managed identity is free; Key Vault is per-operation: cache secrets in memory (don't call KV per request; KV has throttling limits - verify numbers). Redis sized for hit-rate, not guess.

### Observability
- Log `AuthenticationFailedException`, 401/403 from SB/KV; KV diagnostic logs -> Log Analytics:
```kusto
AzureDiagnostics | where ResourceProvider=="MICROSOFT.KEYVAULT" and ResultSignature != "OK" | summarize count() by OperationName, CallerIPAddress, identity_claim_appid_g
```
- Alert on KV 429s and unusual secret reads.

### Key code
```csharp
var cred = new DefaultAzureCredential(new DefaultAzureCredentialOptions { ManagedIdentityClientId = cfg["UamiClientId"] });
var sb = new ServiceBusClient(cfg["Sb:Fqdn"], cred);
var kv = new SecretClient(new Uri(cfg["Kv:Uri"]), cred);
var cfgOpts = await ConfigurationOptions.Parse($"{cfg["Redis:Host"]}:6380").ConfigureForAzureWithTokenCredentialAsync(cred);
var mux = await ConnectionMultiplexer.ConnectAsync(cfgOpts);   // Microsoft.Azure.StackExchangeRedis
```
```bicep
resource uami 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = { name: 'id-orders-prod', location: loc }
resource ra 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(sbQueue.id, uami.id, 'receiver'), scope: sbQueue
  properties: { principalId: uami.properties.principalId, principalType: 'ServicePrincipal'
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions','4f6d3b9b-027b-4f4c-9142-0e5a2a2247e0') } } // Service Bus Data Receiver (verify GUID)
```
Setting `AZURE_CLIENT_ID` env var is the simplest way to pin UAMI.

### Trade-offs
- UAMI = more resources but stable; shared UAMI = blast radius. Redis Entra tokens expire (~1h) so connection must re-authenticate (the extension handles refresh).

### Follow-up ladder
- **Easy:** SAMI vs UAMI? *SAMI tied to resource lifecycle; UAMI standalone & reusable.*
- **Follow-up:** Two UAMIs assigned to one app; which is used? *Must specify clientId; otherwise ambiguous/failure.*
- **Hard:** Works locally, 403 in Azure? *Local uses your user; check MI has RBAC at right scope, propagation delay, private endpoint/firewall, `principalType`.*
- **Tricky:** Slot swap breaks Key Vault references? *SAMI is per-slot (staging slot identity needs its own role); using UAMI avoids it.*

---

## Design 5: Multi-region active-passive failover

### Requirements & clarifying questions
- Survive a regional outage. Ask: RTO/RPO targets? Data stores (SQL, Blob, Cosmos)? Is message loss acceptable? Who triggers failover (auto/manual)? Budget for warm standby? Regulatory data residency?

### Architecture
```
            Azure Front Door (health probes, WAF)  /  Traffic Manager
                 |                                   |
        [Region A: PRIMARY]                  [Region B: SECONDARY (warm)]
   App/Functions + SB Premium (geo-DR     App/Functions (stopped/idle consumers)
   primary alias) + Redis + KV            SB Premium (paired namespace)
   SQL (Failover Group primary) --async-> SQL (secondary, readable)
   Storage RA-GZRS ---------------------> secondary
   Key Vault (auto-replicated to paired region, read-only during outage)
```

### Service choices & WHY
- **Front Door** for global HTTP routing + health-probe failover; Traffic Manager (DNS) if non-HTTP.
- **SQL Auto-failover group:** listener endpoints don't change on failover; async replication => RPO seconds.
- **Service Bus Geo-Disaster Recovery** (metadata only: entities/config replicate, **messages do not**) via alias; or **Geo-Replication** (replicates data too, Premium; verify current status/limits). 
- **Storage RA-GZRS** for read from secondary; customer-managed failover (verify caveats).
- Active-active rejected: conflict resolution and 2x cost not justified by requirements.

### Identity & security
- Entra identity is global; but **managed identities and RBAC are per-resource**: deploy a UAMI + role assignments in region B via the same Bicep. Use alias FQDN for SB so apps need no change.
- Key Vault: separate vault in B or rely on paired-region replication (read-only in failover); keep same secret names.

### Data flow & failure handling
- Failover: (1) Front Door detects unhealthy origin; (2) SQL FOG failover (auto with grace period, or manual `Switch`); (3) SB alias `FailOver`; (4) start/scale Region B consumers; (5) drain/reconcile.
- Messages in primary SB not yet processed may be lost with metadata-only geo-DR → idempotent producers + outbox republish (Design 1) recover unsent events.
- After failback: re-establish pairing (geo-DR pairing is broken after failover; must re-pair).

### Scaling & cost
- Warm standby: min instances in B, SQL secondary can be lower SKU for FOG? (must be same/similar tier for failover headroom; verify). Cold standby = cheaper, longer RTO.

### Observability
- Synthetic availability tests from multiple regions; App Insights per region plus a shared Log Analytics workspace (or cross-workspace queries).
```kusto
requests | summarize fail=countif(success==false), total=count() by cloud_RoleInstance, bin(timestamp,5m)
```
- Alerts on replication lag (`geo_replication_lag` / SQL metrics), Front Door origin health, FOG state change (Activity Log).

### Key code
```csharp
// Always connect to the SB alias, never to the regional name
var client = new ServiceBusClient("sb-orders-alias.servicebus.windows.net", new DefaultAzureCredential());
```
```bicep
resource afd 'Microsoft.Cdn/profiles/originGroups@2023-05-01' = { parent: profile, name: 'api'
  properties: { healthProbeSettings: { probePath: '/health/ready', probeProtocol: 'Https', probeIntervalInSeconds: 30 }
    loadBalancingSettings: { sampleSize: 4, successfulSamplesRequired: 3 } } }
// Origins: priority 1 = region A, priority 2 = region B
```
```csharp
// Readiness probe should check dependencies, liveness should not
app.MapHealthChecks("/health/ready", new() { Predicate = c => c.Tags.Contains("ready") });
```

### Trade-offs
- Auto-failover risks false positives/split brain (grace period tuning); manual is slower but safer. Async replication = non-zero RPO.

### Follow-up ladder
- **Easy:** Active-passive vs active-active? *Passive cheaper and simpler; active-active lower RTO, needs conflict handling.*
- **Follow-up:** Do SB messages survive geo-DR? *Metadata-only: no; plan for re-publish/idempotency; geo-replication feature replicates data (verify).*
- **Hard:** How do you test failover? *Scheduled game days, FOG forced failover in non-prod, chaos, runbook timing recorded vs RTO.*
- **Tricky:** App fails over but can't reach Key Vault/DB? *Private DNS zones/endpoints and role assignments must exist in B; test dependencies not just compute.*

---

## Design 6: Notification fan-out with topics/subscriptions/filters

### Requirements & clarifying questions
- One event -> many independent consumers (email, SMS, push, analytics), each wanting different subsets. Ask: consumers count/growth, per-consumer ordering, replay needs, message rate, who owns subscriptions?

### Architecture
```
Publisher --> [SB Topic: notifications] 
        |-- sub: email   (SQL filter: channel='email' AND priority<>'low')
        |-- sub: sms     (Correlation filter: channel='sms')
        |-- sub: push    (channel='push')
        '-- sub: audit   (TrueFilter) --> Event Hubs/Storage
 each sub: own DLQ, own Function, own TTL & MaxDeliveryCount
```

### Service choices & WHY
- **Service Bus topics** (enterprise messaging, filters, DLQ per sub) vs **Event Grid** (reactive push, no ordering/DLQ semantics like SB, great for system events) vs **Event Hubs** (high-throughput streaming, consumers pull via offsets). Choose SB when each consumer needs reliable independent processing.
- **Correlation filters** preferred over SQL filters: faster, cheaper to evaluate; SQL filters for complex logic.

### Identity & security
- Publisher UAMI: Data Sender on topic. Each subscriber Function SAMI: Data Receiver on **its subscription** scope (`.../topics/notifications/subscriptions/email`).
- Management (creating subscriptions): CI/CD principal with `Azure Service Bus Data Owner` or ARM Contributor on namespace only.

### Data flow & failure handling
- Each subscription is an independent copy -> one slow consumer doesn't block others (but topic size quota is shared per entity; subscription backlog counts - monitor).
- Set `defaultMessageTimeToLive` per sub; `deadLetteringOnMessageExpiration` true for important ones. Idempotency per consumer.
- Publisher sets application properties: `channel`, `priority`, `tenantId` (filters only evaluate system and application properties, not body).

### Scaling & cost
- Filters are free-ish; many subs multiply storage/ops. Premium for high throughput & isolation. Subscription auto-forward to chain topics/queues.

### Observability
- Metrics per subscription: ActiveMessages, DeadLetteredMessages. KQL on `AzureDiagnostics` (`OperationalLogs`) for filter-rule changes.
- Alert: sub backlog age; DLQ count.

### Key code
```bicep
resource rule 'Microsoft.ServiceBus/namespaces/topics/subscriptions/rules@2022-10-01-preview' = {
  parent: subSms, name: 'sms-only'
  properties: { filterType: 'CorrelationFilter', correlationFilter: { properties: { channel: 'sms' } } } }
```
```csharp
var msg = new ServiceBusMessage(body) { Subject = "OrderShipped" };
msg.ApplicationProperties["channel"] = "email";
msg.ApplicationProperties["priority"] = "high";
await sender.SendMessageAsync(msg);
// Admin: new ServiceBusAdministrationClient(fqdn, cred).CreateRuleAsync(topic, sub, new CreateRuleOptions("r", new SqlRuleFilter("priority = 'high'")));
```

### Trade-offs
- Publisher-coupled property contract; changing property names breaks filters. Fan-out count limits per topic (verify limits: subscriptions per topic).

### Follow-up ladder
- **Easy:** Topic vs queue? *Topic = pub/sub with independent subs.*
- **Follow-up:** New consumer needs old messages? *Subscriptions only get messages after creation; need archive (Event Hubs Capture/Blob) for replay.*
- **Hard:** Event Grid or Service Bus for this? *SB if reliable per-consumer processing/DLQ/sessions; Event Grid for lightweight reactive events with many subscribers and filters on subject/type.*
- **Tricky:** Filter on message body? *Not supported; promote fields to application properties.*

---

## Design 7: Saga with Durable Functions (booking/payment/inventory)

### Requirements & clarifying questions
- Multi-step business transaction across services without distributed transactions: reserve inventory → charge payment → confirm booking; compensate on failure. Ask: step durations (seconds vs days)? Human approval? Timeouts? Is each service idempotent? Who sees status?

### Architecture
```
HTTP Start --> [Orchestrator: BookingSaga]
   1 ReserveInventory (activity) --ok--> 2 ChargePayment --ok--> 3 ConfirmBooking
        ^compensate: ReleaseInventory   ^compensate: RefundPayment
   WaitForExternalEvent("PaymentCallback") with timer (timeout race)
 State: Durable Task Hub (Storage/Netherite/MSSQL backends) ; status via query URL
```

### Service choices & WHY
- **Durable Functions** (orchestrator-as-code, replay-based, built-in retries/timers/external events). Alternatives: **Logic Apps** (visual, long waits, low code), **choreography via Service Bus** (no central coordinator, harder to follow), **MassTransit/NServiceBus saga** (library-based). Choose orchestration when flow clarity and compensation order matter.

### Identity & security
- Function SAMI/UAMI: Blob/Queue/Table Data Contributor on the task-hub storage (identity-based `AzureWebJobsStorage__accountName`); roles needed: Storage Blob Data Owner, Queue Data Contributor, Table Data Contributor (verify). Downstream APIs called with `DefaultAzureCredential` token for their app registration scope.

### Data flow & failure handling
- Orchestrator **must be deterministic**: no `DateTime.Now`, random, I/O, `Task.Delay` — use `context.CurrentUtcDateTime`, `CreateTimer`, activities.
- Activities **at-least-once** → idempotent with business keys (`BookingId`).
- Retries: `TaskOptions.FromRetryPolicy(new RetryPolicy(3, TimeSpan.FromSeconds(5), backoffCoefficient: 2))` (isolated worker).
- Compensation: on exception catch, run compensations in reverse; compensation itself retried; if compensation fails -> alert/manual queue.
- Eternal/long orchestrations: `ContinueAsNew` to avoid history bloat.

### Scaling & cost
- Storage transactions are the cost driver; Durable Task Scheduler / MSSQL / Netherite backends exist for throughput (verify current availability). Avoid huge payloads in history—use claim-check.

### Observability
```kusto
traces | where customDimensions.Category == "DurableTask.Core" or message has "orchestration"
| summarize count() by tostring(customDimensions["prop__instanceId"]), bin(timestamp,5m)
```
- Alerts: orchestrations in Failed state, runtime > SLA. Durable Functions Monitor / status query API for support.

### Key code (isolated worker)
```csharp
[Function(nameof(BookingSaga))]
public static async Task<string> BookingSaga([OrchestrationTrigger] TaskOrchestrationContext ctx)
{
    var req = ctx.GetInput<BookingRequest>()!;
    var opts = TaskOptions.FromRetryPolicy(new RetryPolicy(3, TimeSpan.FromSeconds(5), 2.0));
    var done = new Stack<Func<Task>>();
    try {
        await ctx.CallActivityAsync(nameof(Reserve), req, opts);
        done.Push(() => ctx.CallActivityAsync(nameof(Release), req, opts));
        await ctx.CallActivityAsync(nameof(Charge), req, opts);
        done.Push(() => ctx.CallActivityAsync(nameof(Refund), req, opts));
        await ctx.CallActivityAsync(nameof(Confirm), req, opts);
        return "Confirmed";
    } catch (TaskFailedException) {
        while (done.Count > 0) await done.Pop()();
        return "Compensated";
    }
}
```
```json
{ "version":"2.0", "extensions": { "durableTask": { "maxConcurrentActivityFunctions": 10, "maxConcurrentOrchestratorFunctions": 10 } } }
```

### Trade-offs
- Orchestration = central coupling and replay constraints; choreography = looser coupling but hard to reason. Saga gives eventual consistency, never isolation (handle "dirty" intermediate state, e.g., pending status).

### Follow-up ladder
- **Easy:** What is a saga? *Sequence of local transactions with compensations.*
- **Follow-up:** Why must orchestrators be deterministic? *History replay rebuilds state; non-determinism breaks replay.*
- **Hard:** Payment succeeded but activity result lost (crash)? *Activity retried; idempotency key to payment provider returns original result.*
- **Tricky:** Compensation fails (refund API down)? *Retry with backoff, then park in a manual-intervention queue + alert; compensations must be idempotent and retriable; never silently swallow.*

---

## Design 8: Claim-check pattern for large messages

### Requirements & clarifying questions
- Messages up to hundreds of MB (documents, images, batches) exceed SB limits (Standard 256 KB; Premium up to 100 MB — verify current). Ask: size distribution, retention, encryption, who reads payload, PII?

### Architecture
```
Producer --1 upload payload--> [Blob: claims/{yyyy}/{guid}] 
         --2 send small msg {claimUri, sha256, size, contentType, schemaVer}--> [SB queue]
Consumer --3 receive--> --4 download blob (stream)--> process --5 delete/lifecycle expire
```

### Service choices & WHY
- Blob for payload (cheap, streaming, lifecycle management); SB for control message (reliable). Rejected: Premium big messages for all traffic (cost, memory, batch inefficiencies); Event Hubs capture isn't a request/response work queue.

### Identity & security
- Producer MI: `Storage Blob Data Contributor`(write) on `claims`; consumer MI: `Storage Blob Data Reader` (+ Delete if it cleans up). **Do not put SAS URLs in messages** (leak via DLQ/logs); pass blob URI and let consumer use its own RBAC. If cross-tenant consumers, issue short-lived user-delegation SAS.
- Storage encryption at rest default; CMK via Key Vault if required; private endpoint.

### Data flow & failure handling
- Upload first, then send. Crash between => orphan blob: lifecycle rule deletes `claims/` older than N days. Message-before-blob is wrong.
- Integrity: verify SHA-256 on read. Idempotency: blob name = deterministic `MessageId`.
- DLQ'd message still references blob: **lifecycle retention must exceed DLQ retention + replay window**, or blob disappears (classic bug).
- Conditional upload with `BlobUploadOptions.Conditions = new BlobRequestConditions { IfNoneMatch = ETag.All }` for no-overwrite.

### Scaling & cost
- Blob transactions/egress cheap; tier to Cool for retention. Same region to avoid egress. Chunk uploads parallelized by SDK (`StorageTransferOptions`).

### Observability
- Custom dims: claimUri, size, durations; KQL percentile of payload size & download time.
```kusto
customEvents | where name=="ClaimDownloaded" | summarize p95=percentile(todouble(customMeasurements.ms),95), avg(todouble(customMeasurements.bytes)) by bin(timestamp,1h)
```
- Alert: blob 404 rate in consumers; orphan count.

### Key code
```csharp
var container = new BlobContainerClient(new Uri("https://stclaims.blob.core.windows.net/claims"), cred);
var blob = container.GetBlobClient($"{DateTime.UtcNow:yyyy/MM/dd}/{id}");
await blob.UploadAsync(payloadStream, new BlobUploadOptions { Conditions = new BlobRequestConditions { IfNoneMatch = ETag.All },
    TransferOptions = new StorageTransferOptions { MaximumConcurrency = 4 } });
var msg = new ServiceBusMessage(BinaryData.FromObjectAsJson(new { claim = blob.Uri, sha256, size })) { MessageId = id };
await sender.SendMessageAsync(msg);
```
```bicep
resource lc 'Microsoft.Storage/storageAccounts/managementPolicies@2023-01-01' = { parent: st, name: 'default'
  properties: { policy: { rules: [ { name: 'expire-claims', enabled: true, type: 'Lifecycle'
    definition: { filters: { blobTypes: ['blockBlob'], prefixMatch: ['claims/'] }
      actions: { baseBlob: { delete: { daysAfterModificationGreaterThan: 14 } } } } } ] } } }
```

### Trade-offs
- Two systems to keep consistent; extra hop latency. Gains: small fast messages, cheap storage.

### Follow-up ladder
- **Easy:** Why not put the file in the message? *Size limits, cost, throughput.*
- **Follow-up:** Who deletes the blob? *Consumer after success or lifecycle policy; keep for DLQ window.*
- **Hard:** Consumer reads blob while producer still uploading? *Send message only after upload commit.*
- **Tricky:** Lifecycle deletes blob before replay of DLQ? *Align retention; or copy to quarantine on dead-letter.*

---

## Design 9: Scheduled batch/ETL: Timer + Durable fan-out/fan-in

### Requirements & clarifying questions
- Nightly job processes millions of records from a source, transforms, loads to target; replaces SSIS/cron. Ask: data volume, window, dependencies, rerun/backfill, partial failure policy, concurrency limits on source/target?

### Architecture
```
[Timer trigger 0 0 2 * * *] -> Durable client starts Orchestrator (singleton instanceId = "etl-yyyyMMdd")
   Orchestrator:
     GetPartitions (activity) -> N partitions
     fan-out: Task.WhenAll( ProcessPartition(p) x N )   (bounded concurrency)
     fan-in: Aggregate -> Publish summary -> SB / email
   Source (SQL/Blob) --> Activity (batch read, transform) --> Target (SQL bulk)
```

### Service choices & WHY
- Durable fan-out/fan-in: retries, checkpoint, status. Alternatives: Data Factory/Synapse (better for large copy/no-code mapping, SSIS IR lift-and-shift), Azure Batch (heavy compute), Container Apps Jobs (long-running containerised, cron). Pick Functions when logic is C# and partitions are modest.

### Identity & security
- Function MI: SQL read on source / write on target, Blob roles, Storage data roles for task hub. Timer is internal—no inbound exposure; status endpoints behind APIM/Entra.

### Data flow & failure handling
- Deterministic instance ID prevents double-run if timer fires twice (check `GetInstanceAsync` status first). Timer `RunOnStartup=false`; `UseMonitor` ensures singleton across instances (blob lease).
- Partition by key range/date; each activity idempotent (MERGE or delete-insert per partition).
- Bound concurrency by batching tasks (sub-orchestrations in waves) to protect SQL; retry per partition; collect failures and continue, report "partial".
- Rerun: start with input `{ date, partitions:[failed] }`.

### Scaling & cost
- Consumption/Flex scales out; set `maxConcurrentActivityFunctions`. Activity timeouts: Consumption 5 min default/10 max (verify); use Premium/Flex for longer. Cost near zero when idle.

### Observability
```kusto
requests | where operation_Name == "ProcessPartition" | summarize p95=percentile(duration,95), fail=countif(success==false) by bin(timestamp,10m)
```
- Alert: no "EtlCompleted" custom event by 04:00 (missing-signal alert using log query with `count == 0`).

### Key code
```csharp
[Function("NightlyTimer")]
public async Task Timer([TimerTrigger("0 0 2 * * *")] TimerInfo t, [DurableClient] DurableTaskClient client)
{
    var id = $"etl-{DateTime.UtcNow:yyyyMMdd}";
    if (await client.GetInstanceAsync(id) is null) await client.ScheduleNewOrchestrationInstanceAsync("EtlOrchestrator", null, new StartOrchestrationOptions { InstanceId = id });
}
[Function("EtlOrchestrator")]
public static async Task Run([OrchestrationTrigger] TaskOrchestrationContext ctx)
{
    var parts = await ctx.CallActivityAsync<List<string>>("GetPartitions");
    foreach (var wave in parts.Chunk(20))
        await Task.WhenAll(wave.Select(p => ctx.CallActivityAsync("ProcessPartition", p, TaskOptions.FromRetryPolicy(new RetryPolicy(3, TimeSpan.FromSeconds(10))))));
}
```
Timer schedules use NCRONTAB six-field (with seconds), UTC by default (`WEBSITE_TIME_ZONE` for others).

### Trade-offs
- Code-first flexibility vs ADF's monitoring/connectors. Orchestrator history limits require `ContinueAsNew` or sub-orchestrations for huge fan-out.

### Follow-up ladder
- **Easy:** Fan-out/fan-in? *Start many activities in parallel then aggregate with WhenAll.*
- **Follow-up:** Timer fires on 3 instances? *Singleton lock + deterministic instance ID.*
- **Hard:** 1M partitions? *Hierarchical sub-orchestrators, larger batches, or move to ADF/Batch.*
- **Tricky:** Job silently doesn't run? *Missing-heartbeat alert, not error alert.*

---

## Design 10: Webhook receiver: Functions + Service Bus buffering + signature validation

### Requirements & clarifying questions
- Third party (e.g., payment/SaaS) posts webhooks; must respond fast (<~5-10 s typical), be secure, tolerate bursts, retries and duplicates. Ask: signature scheme (HMAC-SHA256?), replay window, expected rate, ordering guarantees?

### Architecture
```
Provider --HTTPS--> Front Door/APIM (WAF, rate limit) --> Func HTTP "Receive"
    1 read raw body  2 verify HMAC + timestamp  3 enqueue to [SB queue: webhooks]  4 return 202
 [SB queue] --> Func "Process" --> business logic / SQL ; DLQ for poison
```

### Service choices & WHY
- Ack-then-process: provider timeouts decoupled from our processing. SB (vs storage queue) for dedupe, DLQ, scheduled retries. Rejected: processing inline (timeouts → provider retries → duplicates).

### Identity & security
- HMAC secret in **Key Vault** (Key Vault reference, rotation-aware with two active secrets during rotation). Receive Func SAMI: Data Sender only. Process Func SAMI: Data Receiver only.
- Validate signature on **raw bytes** before parsing; constant-time compare (`CryptographicOperations.FixedTimeEquals`); reject stale timestamps (replay) e.g., >5 min. `AuthorizationLevel.Anonymous` is acceptable only because HMAC is the auth; optionally IP allowlist.

### Data flow & failure handling
- Dedupe: `MessageId = providerEventId`; SB duplicate detection + consumer idempotency table.
- If enqueue fails → return 5xx so provider retries (this is the one failure path that must be loud).
- Out-of-order events: include provider timestamp/version, apply "latest wins" in the handler; or fetch current resource state from provider API rather than trusting payload.
- Poison: DLQ + alert; replay tool (Design 13).

### Scaling & cost
- HTTP Function scales out for bursts; SB absorbs. Cap Process concurrency to protect downstream.

### Observability
```kusto
requests | where name=="Receive" | summarize total=count(), unauthorized=countif(resultCode=="401"), p95=percentile(duration,95) by bin(timestamp,5m)
```
- Alerts: spike in 401 (attack or secret rotated), enqueue failures, DLQ>0.

### Key code
```csharp
[Function("Receive")]
public async Task<IActionResult> Receive([HttpTrigger(AuthorizationLevel.Anonymous, "post", Route = "hooks/{provider}")] HttpRequest req)
{
    using var ms = new MemoryStream(); await req.Body.CopyToAsync(ms); var raw = ms.ToArray();
    var sig = Convert.FromHexString(req.Headers["X-Signature"].ToString());
    var expected = HMACSHA256.HashData(_secret, raw);
    if (!CryptographicOperations.FixedTimeEquals(sig, expected)) return new UnauthorizedResult();
    await _sender.SendMessageAsync(new ServiceBusMessage(raw) { MessageId = req.Headers["X-Event-Id"].ToString() });
    return new AcceptedResult();
}
```
```json
{ "version":"2.0", "extensions": { "http": { "maxConcurrentRequests": 200, "routePrefix": "api" } } }
```
(Signature header format is provider-specific—follow their docs.)

### Trade-offs
- 202 means "received" not "processed"; providers needing sync results can't use this. Extra latency for processing.

### Follow-up ladder
- **Easy:** Why return 202 quickly? *Avoid provider timeout/retry storms.*
- **Follow-up:** How prevent forged calls? *HMAC on raw body, timestamp tolerance, secret in KV.*
- **Hard:** Provider retries same event concurrently? *Dedupe by event ID (SB + DB unique key).*
- **Tricky:** Secret rotation without dropping events? *Accept both old and new secrets during overlap window.*

---

## Design 11: API rate-limiting + caching layer: APIM + Redis

### Requirements & clarifying questions
- Public/partner API needs per-subscription quotas, protect backend, cache GETs. Ask: tiers (free/gold)? rate per key/IP/tenant? multi-region? cache TTL/invalidation freshness? auth method?

### Architecture
```
Client -> Front Door/WAF -> APIM (Premium/Standard v2)
     inbound policies: validate-jwt -> rate-limit-by-key -> quota-by-key -> cache-lookup
     cache: built-in or external Azure Cache for Redis
     backend -> App Service/Functions (private) ; outbound: cache-store
```

### Service choices & WHY
- **APIM policies** centralise cross-cutting concerns (throttle, JWT, transform) without code changes. **External Redis cache** (vs built-in) is needed for multi-region/ shared across gateways, larger capacity, or for tiers where built-in cache is limited (verify per SKU). 
- Note: `rate-limit-by-key` counters are per gateway unit/region (approximate in multi-instance); distributed exact limits need your own Redis logic.

### Identity & security
- APIM SAMI authenticates to backend via `authentication-managed-identity resource="api://backend"`; backend validates the Entra token (`audience`). Redis connection: Redis Entra auth isn't generally available for APIM's external cache config in all cases (verify); otherwise connection string stored as **named value linked to Key Vault** (APIM SAMI: Key Vault Secrets User).
- Backend accepts traffic only from APIM (private endpoint/VNet + JWT).

### Data flow & failure handling
- Throttled -> 429 with `Retry-After`. Cache failures must not fail requests: APIM falls through to backend. Vary cache keys by user/tenant/headers when needed—**never cache personalized responses under a shared key**.
- Invalidate on change: publish event → Function calls Redis `DEL` / APIM cache purge; or short TTL.

### Scaling & cost
- APIM tiers: Consumption (serverless), Developer (no SLA), Standard/Premium, v2 tiers (verify features/pricing). Cache hit-rate cuts backend cost.

### Observability
```kusto
ApiManagementGatewayLogs | summarize total=count(), throttled=countif(ResponseCode==429), cacheHit=countif(Cache=="hit") by ApiId, bin(TimeGenerated,5m)
```
- Alerts: 429 ratio per key, backend latency, capacity metric.

### Key code (policy)
```xml
<inbound>
  <validate-jwt header-name="Authorization" failed-validation-httpcode="401">
    <openid-config url="https://login.microsoftonline.com/{tenant}/v2.0/.well-known/openid-configuration" />
    <required-claims><claim name="roles" match="any"><value>Api.Read</value></claim></required-claims>
  </validate-jwt>
  <rate-limit-by-key calls="100" renewal-period="60" counter-key="@(context.Subscription.Id)" />
  <quota-by-key calls="100000" renewal-period="2592000" counter-key="@(context.Subscription.Id)" />
  <cache-lookup vary-by-developer="false" vary-by-developer-groups="false" downstream-caching-type="none">
    <vary-by-header>Accept</vary-by-header></cache-lookup>
</inbound>
<outbound><cache-store duration="60" /></outbound>
```

### Trade-offs
- Gateway-level cache = simple but coarse; app-level Redis = fine invalidation. APIM cost significant at Premium.

### Follow-up ladder
- **Easy:** rate-limit vs quota? *Short-term burst limit vs long-term allowance.*
- **Follow-up:** Limit by IP vs key? *`rate-limit-by-key` with `counter-key` expression; careful with proxies (`X-Forwarded-For`).*
- **Hard:** Exact global limit across regions? *Built-in is approximate; use central Redis counters (INCR+EXPIRE/Lua) in a Function/policy-called service.*
- **Tricky:** Cached response leaked to another user? *Missing vary-by; cache only public or vary by identity.*

---

## Design 12: Session/ordered processing per customer with Service Bus sessions

### Requirements & clarifying questions
- Events for the same customer (account updates) must process in order, while different customers run in parallel. Ask: ordering scope (customer/order)? Hot keys? Out-of-order tolerance? Throughput per customer?

### Architecture
```
Producers --SessionId=customerId--> [SB queue (RequiresSession=true)]
   Consumer Functions (N instances) --each accepts a session lock--> process sequentially in session
   Different sessions processed concurrently across instances
```

### Service choices & WHY
- **Sessions** give FIFO per SessionId + exclusive lock + session state. Alternatives: Event Hubs partitions (ordering by partition key, but no per-message settlement/DLQ semantic as SB), Kafka. Single-consumer queue (no sessions) kills parallelism.

### Identity & security
- Same as Design 1: Sender/Receiver data roles on the entity. Session state may hold sensitive data—limit access to Receiver role holders.

### Data flow & failure handling
- A poison message in a session: dead-lettering it lets the session continue; abandoning repeatedly blocks the session (head-of-line blocking) until `MaxDeliveryCount`. Decide policy: DLQ and continue vs. stop session.
- Session lock expiry → another consumer takes over; processing must be idempotent. Use session state (`SetSessionStateAsync`) to store last processed sequence/version.
- Messages without SessionId sent to a session-enabled queue are rejected. Sender must use consistent IDs; ordering also needs a single sender or sequence numbers.
- Hot customer → a single session is single-threaded: shard sub-keys (`customerId-shard`) only if ordering rules allow.

### Scaling & cost
- Concurrency = number of active sessions × instances: Function setting `maxConcurrentSessions` and `maxConcurrentCallsPerSession` (keep 1 for strict order). Idle session timeout `sessionIdleTimeout` affects pickup latency.

### Observability
- Track per-session lag via enqueue time vs process time: `customMetrics` "SessionLagSec" dimension customerId (cardinality caution). Alert on active messages age and DLQ.

### Key code
```csharp
[Function("CustomerEvents")]
public async Task Run([ServiceBusTrigger("customer-events", Connection = "SbConn", IsSessionsEnabled = true)]
    ServiceBusReceivedMessage m, ServiceBusSessionMessageActions session)
{
    var state = await session.GetSessionStateAsync();  // BinaryData
    await _handler.ApplyAsync(m.SessionId, m, state);
    await session.CompleteMessageAsync(m);
}
// send: new ServiceBusMessage(body) { SessionId = customerId, MessageId = eventId }
```
```json
{ "version":"2.0", "extensions": { "serviceBus": { "maxConcurrentSessions": 16, "maxConcurrentCallsPerSession": 1, "sessionIdleTimeout": "00:00:30" } } }
```
```bicep
properties: { requiresSession: true, lockDuration: 'PT1M', maxDeliveryCount: 10, deadLetteringOnMessageExpiration: true }
```

### Trade-offs
- Ordering cuts parallelism per key and adds head-of-line risk; sessions can't be enabled on an existing queue (recreate—verify). Retry/backoff in-session delays everything behind it.

### Follow-up ladder
- **Easy:** How does SB guarantee order? *Sessions: FIFO within a SessionId with exclusive lock.*
- **Follow-up:** Multiple Function instances—same session on two? *No; one lock holder.*
- **Hard:** One bad message stalls a customer? *Dead-letter it (or defer) and move on; alert; ordering gap documented.*
- **Tricky:** Can you scale one customer's throughput? *No beyond single-threaded; redesign key, or relax ordering to per-entity (order ID).*

---

## Design 13: Dead-letter handling and replay tooling

### Requirements & clarifying questions
- Operators must see, diagnose, fix, and replay failed messages safely. Ask: who operates (support vs devs)? PII in messages? Replay to original or fix-first? Volume of DLQ? Audit needed?

### Architecture
```
[Queue/Sub] --fail/MaxDelivery--> [$DeadLetterQueue]
      alert (DLQ count>0) --> Triage Function/Admin API (Entra-protected, role-gated)
         peek -> classify by DeadLetterReason -> (a) resubmit (b) edit+resubmit (c) discard
         resubmit: new message copy -> original entity (+ header "replayCount", "originalMessageId")
         audit: every action logged to Log Analytics/Table
```

### Service choices & WHY
- Built-in DLQ (sub-queue) keeps context (`DeadLetterReason`, `DeadLetterErrorDescription`). Tool as Function/Container App with minimal UI or Service Bus Explorer for ad-hoc. Never auto-replay blindly: causes poison loops.

### Identity & security
- Replay tool UAMI: `Azure Service Bus Data Receiver` (DLQ) + `Data Sender` (target). Humans use Entra groups to call the admin API (App Roles `Dlq.Replay`); no human holds Data Owner in prod; PIM for break-glass. Mask PII in UI.

### Data flow & failure handling
- DLQ causes: MaxDeliveryCount exceeded, TTL expiry, explicit app dead-letter, filter eval failure, session-id issues.
- Replay: receive from DLQ with `SubQueue.DeadLetter` in PeekLock; send a **new** message (copy body/props; DLQ messages can't be "resubmitted" natively); complete DLQ message only after send succeeds. Use SB transactions (`TransactionScope` with `ServiceBusClientOptions.EnableCrossEntityTransactions`) to make receive-complete + send atomic within the same namespace (verify constraints).
- Throttle replay rate; cap `replayCount` (e.g., 3) to prevent loops; different `MessageId` or rely on idempotent consumers (dup detection would otherwise drop same MessageId within window).

### Scaling & cost
- DLQ counts toward entity size quota; set retention policy (alert + purge after N days to archive in Blob).

### Observability
- Metric alert `DeadletteredMessages` > 0 per entity (Azure Monitor metric, dimension EntityName).
```kusto
customEvents | where name in ("DlqReplayed","DlqDiscarded") | summarize count() by name, tostring(customDimensions.reason), bin(timestamp,1d)
```
- Dashboard: DLQ age, top reasons.

### Key code
```csharp
await using var dlq = client.CreateReceiver(queue, new ServiceBusReceiverOptions { SubQueue = SubQueue.DeadLetter });
await using var sender = client.CreateSender(queue);
foreach (var m in await dlq.ReceiveMessagesAsync(50, TimeSpan.FromSeconds(5)))
{
    var copy = new ServiceBusMessage(m);                 // copy constructor from received message
    copy.ApplicationProperties["replayCount"] = (int)(m.ApplicationProperties.GetValueOrDefault("replayCount") ?? 0) + 1;
    await sender.SendMessageAsync(copy);
    await dlq.CompleteMessageAsync(m);
}
```
```bicep
// Alert on any DLQ message
resource alert 'Microsoft.Insights/metricAlerts@2018-03-01' = { name:'dlq-nonzero', location:'global'
  properties: { severity: 2, evaluationFrequency:'PT5M', windowSize:'PT5M', scopes:[ns.id], criteria:{ 'odata.type':'Microsoft.Azure.Monitor.SingleResourceMultipleMetricCriteria'
    allOf:[{ name:'c1', metricName:'DeadletteredMessages', operator:'GreaterThan', threshold:0, timeAggregation:'Maximum', criterionType:'StaticThresholdCriterion' }] } } }
```

### Trade-offs
- Manual triage is safe but slow; automatic replay for known transient reasons (e.g., dependency outage) can be policy-driven. Keep a "fix then replay" path for data defects.

### Follow-up ladder
- **Easy:** What lands in DLQ? *Max deliveries, TTL expiry (if enabled), explicit dead-lettering.*
- **Follow-up:** How to replay? *Receive from DLQ sub-queue, resend copy, complete DLQ copy.*
- **Hard:** Replay creates duplicates? *Idempotent consumers; keep original MessageId in a property; note dup-detection window.*
- **Tricky:** DLQ of the DLQ? *Doesn't exist; DLQ messages have no further DLQ—so replay tool must be robust and archive before purge.*

---

## Design 14: Cache-aside with Redis + stampede protection for catalog API

### Requirements & clarifying questions
- Read-heavy catalog (products/prices) with SQL backend. Ask: staleness tolerance? QPS? data size/keys? hot keys? invalidation signal available? Multi-region?

### Architecture
```
Client -> API -> Redis GET key
                   hit -> return
                   miss -> acquire per-key lock (SET NX PX) -> load SQL -> SET w/ TTL+jitter -> release
                   others wait/serve stale
Change event (SB/Event Grid) -> Func -> DEL/refresh keys
```

### Service choices & WHY
- Redis cache-aside: app owns logic, tolerant to cache loss. Alternatives: read-through libraries, HybridCache in .NET 9 (L1 memory + L2 distributed with built-in stampede protection—verify package/version), CDN for fully public content.

### Identity & security
- App UAMI with Redis data access policy (Entra auth) via Microsoft.Azure.StackExchangeRedis; private endpoint; TLS 1.2+. Don't cache secrets/PII without encryption/TTL consideration.

### Data flow & failure handling
- **Stampede** (many concurrent misses on a hot expiring key): (1) single-flight in-process (`SemaphoreSlim`/`Lazy` per key), (2) distributed lock (`LockTakeAsync`/SET NX), (3) TTL jitter so keys don't expire together, (4) stale-while-revalidate (store soft-TTL inside value), (5) negative caching for not-found (short TTL).
- Redis down: fail open to SQL with circuit breaker (Polly) and bulkhead so DB isn't flattened; `AbortOnConnectFail=false`.
- Invalidate-on-write vs TTL: write path updates DB then deletes key (not set) to avoid race where stale write wins.

### Scaling & cost
- Choose tier by memory, throughput, HA (replicas), clustering (verify tiers / Azure Managed Redis current offering). Eviction `volatile-lru`; watch memory fragmentation; use compact serialization (MessagePack/System.Text.Json source-gen).

### Observability
- Redis metrics: Cache hits/misses, Used Memory, Server Load, Evicted Keys, Connected Clients. App custom metric hit ratio.
```kusto
customMetrics | where name in ("CacheHit","CacheMiss") | summarize v=sum(value) by name, bin(timestamp,5m) | evaluate pivot(name, sum(v))
```
- Alerts: hit ratio < target, evictions>0, server load > 80%.

### Key code (HybridCache in .NET 9 style; verify API)
```csharp
builder.Services.AddHybridCache(o => o.DefaultEntryOptions = new() { Expiration = TimeSpan.FromMinutes(10), LocalCacheExpiration = TimeSpan.FromMinutes(1) });
// usage: concurrent callers for same key share one factory execution
var product = await cache.GetOrCreateAsync($"product:{id}", async ct => await repo.GetAsync(id, ct), cancellationToken: ct);
```
Manual lock variant:
```csharp
var db = mux.GetDatabase(); var token = Guid.NewGuid().ToString();
if (await db.LockTakeAsync($"lock:{key}", token, TimeSpan.FromSeconds(10))) {
    try { var v = await LoadAsync(); await db.StringSetAsync(key, v, TimeSpan.FromMinutes(10) + TimeSpan.FromSeconds(Random.Shared.Next(0,60))); }
    finally { await db.LockReleaseAsync($"lock:{key}", token); }
}
```

### Trade-offs
- Staleness vs load; locks add latency/complexity; cache warming adds cost.

### Follow-up ladder
- **Easy:** What's cache-aside? *App checks cache, loads DB on miss, populates cache.*
- **Follow-up:** What's a stampede? *Many requests miss simultaneously and hit DB; fix with single-flight, jitter, stale-while-revalidate.*
- **Hard:** Race between update and cache fill leaves stale value? *Delete after DB commit, short TTL as safety net, versioned keys.*
- **Tricky:** Redis fails over, connection storms? *Reuse single `ConnectionMultiplexer`, retry with backoff/jitter, circuit breaker.*

---

## Design 15: Audit/event log pipeline to Log Analytics/Event Hubs

### Requirements & clarifying questions
- Tamper-evident audit trail + operational telemetry; query for investigations; long-term retention for compliance; stream to SIEM. Ask: retention years? volume GB/day? immutability needed? PII/redaction? SIEM (Sentinel/Splunk)?

### Architecture
```
Apps (ILogger/AuditEvent) --> [Event Hubs: audit] --+--> Stream Analytics/Func --> Log Analytics (custom table via DCR/Logs Ingestion API)
Azure resources (diagnostic settings) -------------+--> Log Analytics workspace (hot, 30-90d)
                                                    +--> Capture -> [Blob/ADLS immutable (WORM)] (cold, years)
                                                    +--> SIEM/Sentinel / Splunk consumer group
```

### Service choices & WHY
- **Event Hubs:** high-throughput, multiple consumer groups, Capture to storage. **Log Analytics:** KQL, alerts, Sentinel. **Immutable Blob (time-based retention/legal hold):** compliance. Rejected: writing audits only to App Insights (sampling can drop them! Audit must never be sampled) or app DB (mutable).

### Identity & security
- Producers: `Azure Event Hubs Data Sender`; consumers `Data Receiver` per consumer group; ingestion Function MI: `Monitoring Metrics Publisher` on the DCR to use Logs Ingestion API. Workspace RBAC table-level; restrict who can query audit tables. Private Link (AMPLS) for Monitor.

### Data flow & failure handling
- At-least-once; include `EventId` (GUID) + `Hash chain` (prev hash) for tamper evidence. Dedupe by EventId in queries (`summarize arg_max`).
- Event Hubs checkpointing in Blob via `EventProcessorClient`; on failure replay from checkpoint. Retention (1-90 days depending on tier; verify) as buffer.
- Producer failure: local buffer/outbox so business operation isn't lost; fail closed for critical audits (don't proceed w/o audit) as policy decision.

### Scaling & cost
- Event Hubs throughput units / processing units / partitions (partition count fixed on Standard—verify). Log Analytics cost by ingestion GB: use Basic/Auxiliary logs tiers for high-volume low-query data (verify), commitment tiers, table-level retention, archive. DCR transformations to drop noisy columns.

### Observability
```kusto
AuditEvents_CL | where TimeGenerated > ago(1d) | summarize arg_max(TimeGenerated,*) by EventId | summarize count() by Action, Actor
```
- Alerts on ingestion latency (`_TimeReceived - TimeGenerated`), EH `ThrottledRequests`, consumer lag (capture backlog), Log Analytics daily cap hit.

### Key code
```csharp
await using var producer = new EventHubProducerClient("ehns.servicebus.windows.net", "audit", new DefaultAzureCredential());
using var batch = await producer.CreateBatchAsync(new CreateBatchOptions { PartitionKey = tenantId });
batch.TryAdd(new EventData(BinaryData.FromObjectAsJson(auditEvent)));
await producer.SendAsync(batch);
// Logs Ingestion API
var logs = new LogsIngestionClient(new Uri(dceEndpoint), new DefaultAzureCredential());
await logs.UploadAsync(dcrImmutableId, "Custom-AuditEvents_CL", records);
```
```bicep
resource diag 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = { scope: sb, name:'to-law'
  properties: { workspaceId: law.id, logs:[{ categoryGroup:'allLogs', enabled:true }], metrics:[{ category:'AllMetrics', enabled:true }] } }
```

### Trade-offs
- Dual path (LA hot + blob cold) costs more but meets both needs. Event Hubs adds ops vs direct ingestion API (simpler, lower throughput resilience).

### Follow-up ladder
- **Easy:** Why Event Hubs? *High-volume ingest, multi-consumer, capture.*
- **Follow-up:** How ensure audit not sampled? *Don't route audits through App Insights sampling; separate pipeline.*
- **Hard:** Prove logs not altered? *Immutable storage (WORM), hash chaining, restricted write roles.*
- **Tricky:** Log Analytics bill explodes? *Find top tables via `Usage`, DCR transforms, Basic logs, sampling of debug logs, daily cap with care (data loss).*

---

## Design 16: Document processing pipeline (Blob + Functions + queue + status API)

### Requirements & clarifying questions
- Users upload PDFs/images; pipeline extracts text (Document Intelligence), classifies, stores results; user polls status. Ask: file size, SLA, concurrency, PII, retention, virus scanning, retries on AI quota?

### Architecture
```
Client -> API (Func HTTP): POST /documents -> returns 202 + docId + Location /documents/{id}/status
        -> upload via user-delegation SAS (direct to Blob) or API stream -> [Blob: uploads]
BlobCreated -> Event Grid -> [SB queue: doc-process] -> Func Process:
     scan -> OCR/extract (Document Intelligence) -> save result [Blob: results] -> update [Cosmos/SQL: DocStatus]
 GET /documents/{id}/status -> Queued|Processing|Succeeded|Failed (+ resultUrl)
```

### Service choices & WHY
- Async request-reply (202 + status URL) for long tasks. Queue for backpressure to rate-limited AI service. Table/Cosmos/SQL for status (cheap point reads). Durable Functions function-chaining is an alternative if multiple steps need orchestration (Design 7/9).

### Identity & security
- Function MI: Blob Data Contributor (uploads/results), `Cognitive Services User` on Document Intelligence (Entra auth, disable keys), SB Receiver, DB role. Client upload SAS: **user-delegation SAS** (signed by Entra, not account key), short expiry, write-only, scoped to blob name. Defender for Storage malware scanning (verify) before processing. Private endpoints.

### Data flow & failure handling
- Status transitions are idempotent upserts with ETag/optimistic concurrency. 429 from AI: retry with `Retry-After` (delay re-queue via `ScheduledEnqueueTime` rather than blocking locks).
- Poison document (corrupt): mark `Failed` with reason, dead-letter, user sees error. Timeout/lock: set lock duration > processing time or renew (Functions auto-renew up to `maxAutoLockRenewalDuration`).
- Cleanup: lifecycle rules for uploads; result TTL.

### Scaling & cost
- Concurrency limited by AI TPS; set `maxConcurrentCalls` accordingly. Pay-per-page OCR is dominant cost—skip reprocessing via content hash cache.

### Observability
```kusto
customEvents | where name=="DocStatus" | summarize count() by tostring(customDimensions.status), bin(timestamp,15m)
requests | where name=="Process" | summarize p95=percentile(duration,95)
```
- Alerts: Failed ratio, queue age, AI 429 rate.

### Key code
```csharp
var cred = new DefaultAzureCredential();
var svc = new BlobServiceClient(new Uri(acct), cred);
var key = await svc.GetUserDelegationKeyAsync(DateTimeOffset.UtcNow, DateTimeOffset.UtcNow.AddMinutes(15));
var sas = new BlobSasBuilder(BlobSasPermissions.Create | BlobSasPermissions.Write, DateTimeOffset.UtcNow.AddMinutes(15)) { BlobContainerName="uploads", BlobName=docId, Resource="b" };
var uri = new BlobUriBuilder(blobUri) { Sas = sas.ToSasQueryParameters(key.Value, svc.AccountName) }.ToUri();
// Status endpoint
return status switch { "Succeeded" => Results.Ok(new{status,resultUrl}), "Failed" => Results.Ok(new{status,error}), _ => Results.Accepted(null, new{status}) };
```

### Trade-offs
- Direct-to-blob upload offloads API but needs Event Grid linkage; polling vs push (SignalR/webhook callback) tradeoff.

### Follow-up ladder
- **Easy:** Why 202 + status URL? *Long job; avoid HTTP timeouts.*
- **Follow-up:** Secure client upload? *User-delegation SAS, short-lived, minimal permission.*
- **Hard:** Processing > lock duration? *Auto lock renewal, or Durable orchestration; avoid timeouts.*
- **Tricky:** AI service throttles under load? *Concurrency caps + scheduled re-enqueue with backoff, not tight retries.*

---

## Design 17: Email/SMS notification system with retry & throttling

### Requirements & clarifying questions
- Send transactional email/SMS at scale; respect provider rate limits, avoid duplicates, handle bounces. Ask: volume/burst, templates, per-user preferences/opt-out, delivery receipts, priority tiers, compliance (consent, quiet hours)?

### Architecture
```
Producers -> [SB topic: notifications] -> sub email -> Func Email -> Azure Communication Services Email
                                       -> sub sms   -> Func Sms   -> ACS SMS
 Priority: separate queues (high/low) ; Throttle: maxConcurrentCalls + token-bucket (Redis)
 Delivery events: ACS -> Event Grid -> Func -> update [Status DB]; bounces -> suppression list
```

### Service choices & WHY
- **Azure Communication Services** (Email/SMS) with Entra auth; alternatives SendGrid/Twilio (third-party; keys in KV). SB separates producers from provider limits. Separate priority queues avoid marketing blast starving OTP codes.

### Identity & security
- Sender Func MI: ACS role (`Communication and Email Service Owner` or data-plane equivalent—verify current built-in role names). SB Receiver. Templates in Blob (Reader). Never log message bodies/PII; store recipient hashes in logs. Domain auth: SPF/DKIM/DMARC configured.

### Data flow & failure handling
- Idempotency key `notificationId` stored before send (`Sent` state) → protect from duplicates on redelivery; trade-off: "at-most-once vs at-least-once" decision—emails prefer slight duplicates over loss, OTP prefers no duplicates? State explicitly.
- Retry classification: 429/5xx → exponential backoff via `ScheduledEnqueueTime` re-enqueue (avoid lock occupancy); 4xx invalid address → dead-letter/suppress, don't retry. Provider outage → circuit breaker, pause subscription (disable sub) rather than burn delivery counts.
- Throttle: Functions `maxConcurrentCalls`, plus Polly `RateLimiter`/Redis token bucket for provider TPS across instances.

### Scaling & cost
- Per-message provider cost; batch where supported. SB scales; throttle to provider quotas (verify ACS limits and raise quota requests).

### Observability
```kusto
customEvents | where name=="NotificationResult" | summarize sent=countif(tostring(customDimensions.result)=="Sent"), failed=countif(tostring(customDimensions.result)=="Failed") by channel=tostring(customDimensions.channel), bin(timestamp,15m)
```
- Alerts: failure rate, queue age for high-priority, bounce rate (reputation), DLQ.

### Key code
```csharp
var email = new EmailClient(new Uri("https://acs-contoso.communication.azure.com"), new DefaultAzureCredential());   // Azure.Communication.Email
var op = await email.SendAsync(WaitUntil.Started, senderAddress, recipient, subject, htmlContent);
// Re-enqueue with backoff
var retry = new ServiceBusMessage(m) { ScheduledEnqueueTime = DateTimeOffset.UtcNow.AddSeconds(Math.Pow(2, attempt) * 10) };
retry.ApplicationProperties["attempt"] = attempt + 1;
await sender.SendMessageAsync(retry); await actions.CompleteMessageAsync(m);
```
(Check ACS Email SDK method overloads in current docs.)

### Trade-offs
- Re-enqueue pattern gives control but changes MessageId semantics/ordering. Priority queues add ops.

### Follow-up ladder
- **Easy:** Why queue notifications? *Decouple, retry, protect provider limits.*
- **Follow-up:** Avoid duplicate SMS? *Idempotency record before sending + provider idempotency when available.*
- **Hard:** Provider 429 storms? *Backoff with jitter, rate limiter, scheduled retry.*
- **Tricky:** Bounce handling? *Consume delivery events, add to suppression list, stop sending to hard bounces.*

---

## Design 18: Multi-tenant SaaS with per-tenant identity & data isolation

### Requirements & clarifying questions
- SaaS serving many customers; each tenant has own users (possibly their own Entra tenant); data must be isolated; noisy neighbors controlled. Ask: tenant count (10 vs 10k)? compliance tiers? tenant-specific SLAs/regions? BYOK? onboarding automation?

### Architecture
```
Users (multi-tenant Entra app / External ID) -> Front Door -> APIM/API (validate tid + tenant mapping)
   Tenant resolver: claim `tid` / subdomain -> TenantContext
   Data tier options (tiered):
     Pool: shared SQL DB with TenantId + Row-Level Security / EF global filter
     Bridge: elastic pool, DB-per-tenant
     Silo: dedicated subscription/resource group for premium tenants
   Messaging: shared topic with `tenantId` property; premium tenants own queues
```

### Service choices & WHY
- **Entra multi-tenant app registration** (or Entra External ID for B2C-like) → issuer/`tid` validation. **Azure SQL elastic pools** for DB-per-tenant cost efficiency; **RLS** for pooled model. Choose by isolation need vs cost (pool cheapest, silo safest).

### Identity & security
- Validate `tid` against allowlist (multi-tenant apps otherwise accept any Entra tenant—critical check: don't just validate signature). Use `iss` check & tenant mapping table.
- Service-to-service: UAMI per tier. Per-tenant Key Vault or per-tenant keys for BYOK; tenant-specific storage containers with RBAC/ABAC conditions (role assignment conditions on container names/tags—verify) rather than shared admin identity.
- Avoid confused-deputy: tenant ID comes from validated token, never from request body.

### Data flow & failure handling
- SQL RLS: `SESSION_CONTEXT` set per connection (`sp_set_session_context 'TenantId'`)—reset on pooled connections. EF Core global query filter plus RLS as defense in depth.
- Noisy neighbor: per-tenant rate limits (APIM `rate-limit-by-key` with tenant key), per-tenant SB queue/priority, resource governance.
- Tenant onboarding: idempotent provisioning pipeline (Durable orchestration); offboarding: export + purge (GDPR).

### Scaling & cost
- Cost attribution by tags/`tenantId` custom dimension; elastic pool eDTU/vCore sizing; shard map for >N DBs (Elastic Database client library, verify status).

### Observability
```kusto
requests | extend tenant = tostring(customDimensions.tenantId) | summarize count(), p95=percentile(duration,95) by tenant, bin(timestamp,1h)
```
- Per-tenant SLIs and alerts for premium tenants; watch cardinality; audit cross-tenant access attempts (403 with tenant mismatch).

### Key code
```csharp
builder.Services.AddAuthentication().AddJwtBearer(o => {
  o.TokenValidationParameters = new() { ValidateIssuer = true, IssuerValidator = (iss, tok, p) => tenants.IsAllowedIssuer(iss) ? iss : throw new SecurityTokenInvalidIssuerException() }; });
modelBuilder.Entity<Order>().HasQueryFilter(o => o.TenantId == _tenant.Id);
```
```sql
CREATE SECURITY POLICY TenantFilter ADD FILTER PREDICATE dbo.fn_tenant(TenantId) ON dbo.Orders WITH (STATE = ON);
-- fn_tenant: WHERE TenantId = CAST(SESSION_CONTEXT(N'TenantId') AS uniqueidentifier)
```

### Trade-offs
- Pool = cheap but weakest isolation; silo = strongest but ops-heavy. Tiered model is typical.

### Follow-up ladder
- **Easy:** Where does tenant ID come from? *Validated token claims.*
- **Follow-up:** RLS vs DB-per-tenant? *RLS cheap, shared blast radius; DB-per-tenant isolation & per-tenant restore, more ops.*
- **Hard:** Multi-tenant app accepts any Entra tenant? *Validate issuer/tid against onboarded tenants.*
- **Tricky:** Pooled connection leaks previous tenant context? *Reset SESSION_CONTEXT every checkout; add DB-level RLS tests.*

---

## Design 19: Secrets rotation automation (Key Vault + Event Grid + Functions)

### Requirements & clarifying questions
- Rotate secrets/keys/certs automatically without downtime. Ask: what secrets remain (SQL passwords, third-party API keys, storage keys)? Can they be eliminated via MI? Rotation cadence? Consumers refresh behavior? Dual-credential support?

### Architecture
```
[Key Vault] --SecretNearExpiry event--> [Event Grid system topic] --> Func "RotateSecret"
   Func: 1 generate new credential on target (e.g., regenerate secondary key)
         2 set new secret VERSION in Key Vault  3 verify  4 (later) retire old
 Consumers: Key Vault refs / SecretClient w/ cache TTL pick up latest version
```

### Service choices & WHY
- Event Grid events from Key Vault (`Microsoft.KeyVault.SecretNearExpiry`, `SecretExpired`, `CertificateNearExpiry`, etc.). Key Vault native auto-rotation exists for keys (rotation policy) & integrated CA certs; secrets need custom Function. **First choice: remove the secret** (managed identity/Entra auth).

### Identity & security
- Rotation Func MI: `Key Vault Secrets Officer` on the specific vault (write new versions) — consumers only `Secrets User`. Plus rights on target (e.g., Storage `Storage Account Key Operator Service Role`, or SQL admin via Entra). Key Vault RBAC mode, soft-delete + purge protection, private endpoint (trusted services bypass for Event Grid—verify delivery with MI).

### Data flow & failure handling
- **Two-key (alternating) strategy:** key1/key2 → rotate the inactive one, publish it, retire old after consumers refreshed → zero downtime.
- Idempotent: if new version already set for the expiry cycle, skip. Event Grid retries; Function must be safe to repeat.
- Failure: alert if secret reaches expiry without rotation (EventGrid `SecretExpired` handler pages on-call).
- Consumers: Key Vault references auto-refresh (about daily; verify) unless pinned to version → **don't pin versions**; SDK clients re-fetch on 401 or periodic TTL.

### Scaling & cost
- Negligible. KV ops cost minor; avoid per-request `GetSecret` (cache).

### Observability
```kusto
AzureDiagnostics | where ResourceProvider=="MICROSOFT.KEYVAULT" and OperationName in ("SecretSet","SecretNewVersionCreated") | project TimeGenerated, id_s, CallerIPAddress, identity_claim_appid_g
```
- Alerts: rotation function failures, secrets expiring < 14 days (Azure Policy/Defender or scheduled query), unexpected `SecretGet` volume.

### Key code
```csharp
[Function("RotateSecret")]
public async Task Run([EventGridTrigger] EventGridEvent e)
{
    var data = e.Data!.ToObjectFromJson<KvEventData>();   // ObjectName, VaultName
    var sc = new SecretClient(new Uri($"https://{data.VaultName}.vault.azure.net"), new DefaultAzureCredential());
    var newValue = await _target.RegenerateSecondaryAsync();
    await sc.SetSecretAsync(new KeyVaultSecret(data.ObjectName, newValue) { Properties = { ExpiresOn = DateTimeOffset.UtcNow.AddDays(60), ContentType = "api-key" } });
}
```
```bicep
resource sub 'Microsoft.EventGrid/systemTopics/eventSubscriptions@2022-06-15' = { parent: kvTopic, name:'near-expiry'
  properties: { destination:{ endpointType:'AzureFunction', properties:{ resourceId: '${func.id}/functions/RotateSecret' } }
    filter:{ includedEventTypes:['Microsoft.KeyVault.SecretNearExpiry'] } } }
```

### Trade-offs
- Automation adds an attack surface (rotation identity is powerful) vs manual error-prone rotation; best: eliminate secrets.

### Follow-up ladder
- **Easy:** How avoid secrets? *Managed identities + RBAC.*
- **Follow-up:** How rotate without downtime? *Dual keys / overlap, consumers read latest version.*
- **Hard:** App cached old secret and now gets 401? *Retry-on-401 with refresh, TTL on cache, overlap period.*
- **Tricky:** Secret near expiry events not firing? *Needs `Expires` set on the secret and subscription at vault scope; events fire ~30 days before (verify).*

---

## Design 20: Blue-green/slot deployment with config & feature flags

### Requirements & clarifying questions
- Zero-downtime releases, fast rollback, safe config changes. Ask: stateful dependencies (DB schema)? Functions with queue triggers (both slots consuming)? traffic ramp? who toggles flags?

### Architecture
```
CI/CD (GitHub Actions/Azure DevOps) -> deploy to [staging slot] -> warm-up + smoke tests
  -> optional traffic % (slot routing) -> swap (with preview) -> prod
 App Configuration (feature flags, labels per env) + Key Vault refs ; UAMI shared by slots
 DB: expand/contract migrations (backward compatible)
```

### Service choices & WHY
- **App Service/Function slots** (swap = warm instances, instant rollback by re-swap). **Azure App Configuration** + Feature Management for runtime toggles (decouple deploy from release). Alternatives: Container Apps revisions (traffic splitting), AKS blue-green, Front Door weighted origins.

### Identity & security
- **Use UAMI** so slots share RBAC; SAMI differs per slot (staging needs its own role assignments, Key Vault refs fail otherwise). Pipeline principal (federated OIDC workload identity, no secrets): `Website Contributor` on app. App Configuration: `App Configuration Data Reader` for apps.
- Slot-specific ("sticky") settings for things that must not swap (e.g., slot name, environment-specific endpoints).

### Data flow & failure handling
- DB changes: expand → deploy → contract; both versions run simultaneously during swap.
- **Functions caveat:** non-HTTP triggers in staging slot also run → staging consumes production queue messages. Use slot-specific settings to disable functions in staging (`AzureWebJobs.<Name>.Disabled` sticky) or separate queue; for Service Bus, drain gracefully.
- Auto-swap/rollback: health-check path + `WEBSITE_SWAP_WARMUP_PING_PATH`; failed warmup aborts swap. Swap with preview applies production settings to staging for validation.

### Scaling & cost
- Slots on Standard+ share the plan's compute (cost = plan). Feature flags avoid long-lived environments.

### Observability
- Tag telemetry with `cloud_RoleInstance`/slot and `AppVersion`; compare error rate pre/post swap:
```kusto
requests | summarize fail=countif(success==false)*100.0/count() by tostring(customDimensions.AppVersion), bin(timestamp,5m)
```
- Alerts: error-rate regression within 15 min of deployment → trigger rollback automation.

### Key code
```csharp
builder.Configuration.AddAzureAppConfiguration(o => o
    .Connect(new Uri(endpoint), new DefaultAzureCredential())
    .UseFeatureFlags(ff => ff.SetRefreshInterval(TimeSpan.FromSeconds(30)))
    .ConfigureKeyVault(kv => kv.SetCredential(new DefaultAzureCredential())));
builder.Services.AddAzureAppConfiguration().AddFeatureManagement();
if (await features.IsEnabledAsync("NewCheckout")) { ... }
```
```bicep
resource slot 'Microsoft.Web/sites/slots@2023-01-01' = { parent: app, name:'staging', location: loc
  identity:{ type:'UserAssigned', userAssignedIdentities:{ '${uami.id}':{} } }
  properties:{ serverFarmId: plan.id, keyVaultReferenceIdentity: uami.id } }
```
```
az webapp deployment slot swap -g rg -n app --slot staging --target-slot production
```

### Trade-offs
- Slots are simple but share plan resources; true blue-green with separate infra costs 2x. Feature flags add flag debt—remove stale flags.

### Follow-up ladder
- **Easy:** What does a swap do? *Swaps routing after warm-up; settings marked sticky stay.*
- **Follow-up:** DB migration with swap? *Backward-compatible expand/contract.*
- **Hard:** Staging slot processing prod queue? *Disable non-HTTP triggers in staging via sticky settings.*
- **Tricky:** Key Vault references fail in new slot? *Slot SAMI lacks RBAC; use UAMI or grant per slot.*

---

## Design 21: Real-time dashboard with SignalR Service + Functions

### Requirements & clarifying questions
- Push live updates (orders, metrics) to thousands of browsers. Ask: concurrent connections, message rate, per-user/group targeting, auth, fallback, history on connect?

### Architecture
```
Producers -> [Event Hubs/SB/Change feed] -> Func (trigger) -> SignalR output binding -> [Azure SignalR Service (Serverless mode)]
Browser: POST /api/negotiate (Func, Entra auth) -> gets service URL + token -> WebSocket to SignalR Service
Initial state: GET /api/snapshot (from Redis/SQL) then deltas
```

### Service choices & WHY
- **Azure SignalR Service Serverless mode** with Functions: offloads connection management; scale to 100k+ connections (verify limits by tier). Alternatives: Web PubSub (generic WebSocket/pub-sub), self-hosted SignalR (needs sticky sessions/backplane), polling (wasteful), SSE.

### Identity & security
- Function MI to SignalR: identity-based connection `AzureSignalRConnectionString__serviceUri` (verify current binding settings); role `SignalR Service Owner`/`SignalR App Server` as per docs. Negotiate endpoint authenticates the user (EasyAuth/Entra), sets `userId` claim; group membership checked server-side (tenant dashboards). Disable local (key) auth where supported. CORS restricted.

### Data flow & failure handling
- Delivery is best-effort (no guaranteed persistence): clients on reconnect must fetch snapshot + resume from last version/sequence. Include `seq` in messages; client detects gaps and refetches.
- Backpressure/coalescing: batch updates (e.g., 1/s per entity) to avoid flooding.
- Event source with checkpoints (Event Hubs trigger) so Function failures replay.

### Scaling & cost
- Units (connection + message quotas), Free/Standard/Premium; cost scales with units/days. Message count billing → coalesce. Use groups instead of broadcasting to all.

### Observability
- SignalR metrics: Connection Count, Message Count, Connection Close Count, system errors; App Insights for negotiate failures. Alert on connection drops spike & throttling.
```kusto
requests | where name=="negotiate" | summarize count(), fail=countif(success==false) by bin(timestamp,5m)
```

### Key code
```csharp
[Function("negotiate")]
public static string Negotiate([HttpTrigger(AuthorizationLevel.Anonymous, "post")] HttpRequestData req,
    [SignalRConnectionInfoInput(HubName = "dashboard", UserId = "{headers.x-ms-client-principal-id}")] string info) => info;

[Function("Broadcast")]
[SignalROutput(HubName = "dashboard")]
public SignalRMessageAction Run([EventHubTrigger("metrics", Connection = "EhConn")] string[] events)
    => new("metricsUpdated") { GroupName = "tenant-42", Arguments = new object[] { Aggregate(events) } };
```
```ts
const conn = new signalR.HubConnectionBuilder().withUrl(info.url,{accessTokenFactory:()=>info.accessToken}).withAutomaticReconnect().build();
```
(Isolated-worker SignalR extension types: verify in current docs.)

### Trade-offs
- Serverless mode loses server-side hub logic (client→server messages go via upstream/REST). Eventual UI consistency.

### Follow-up ladder
- **Easy:** What does negotiate do? *Returns SignalR URL + access token for the client.*
- **Follow-up:** Target one tenant? *Groups/userId.*
- **Hard:** Client missed messages during reconnect? *Snapshot + sequence-gap detection.*
- **Tricky:** 100k users each updates every second? *Coalesce/aggregate; message-count billing & limits.*

---

## Design 22: Migration of on-prem WCF/SSIS/Windows Service jobs to Azure

### Requirements & clarifying questions
- Inventory legacy: WCF services (SOAP), SSIS packages, Windows Services/scheduled tasks. Move with minimal risk. Ask: which are still used? dependencies on file shares/AD/SQL Agent? SLAs, change windows? Who consumes WCF (can they change)? Licensing?

### Architecture
```
 BEFORE                                  AFTER
 WCF service (SOAP)         -->  ASP.NET Core Web API / CoreWCF on App Service/Container Apps behind APIM (SOAP passthrough/transform)
 SSIS packages              -->  (a) ADF + SSIS Integration Runtime (lift-and-shift)  (b) rewrite as ADF pipelines/Functions
 Windows Service (poller)   -->  Functions (Timer/SB trigger) or Container Apps Job / WebJob
 Scheduled tasks / SQL Agent-->  Timer Functions / ADF triggers / Elastic Jobs
 File shares                -->  Blob / Azure Files ; Hybrid: Self-hosted IR, VPN/ExpressRoute
```

### Service choices & WHY
- Strangler-fig: APIM front door routes per operation to old or new; migrate incrementally. **CoreWCF** keeps SOAP contracts for clients that can't change; REST for new. SSIS: SSIS IR in ADF for minimal change, rewrite where value. Windows Service → event-driven Function (scale to zero) unless long-running/stateful → Container Apps / AKS / VM.
- Rejected: big-bang rewrite; lift-and-shift VMs for everything (no modernization benefit, but valid first step—"rehost then refactor").

### Identity & security
- Replace Windows auth/service accounts with managed identities; AD-dependent resources: Entra Domain Services or hybrid join; connection strings/`web.config` secrets → Key Vault + MI. WCF transport security → TLS + Entra/OAuth at APIM (`validate-jwt`) or client certs. On-prem connectivity: VPN/ExpressRoute + private endpoints.

### Data flow & failure handling
- Parallel run: compare outputs old vs new (shadow traffic) before cutover. Idempotency: legacy jobs often assume single execution → make explicit (locks, instance IDs).
- Migrate data (DMS/Azure Database Migration Service) with cutover plan and rollback; keep old system read-only fallback.
- Windows Service state in memory/files → externalize (Blob/SQL/Redis).

### Scaling & cost
- Right-size: scheduled jobs → consumption plans; SSIS IR bills while running (start/stop it via ADF/automation). Compare TCO including decommission savings.

### Observability
- Replace Event Log/log files → App Insights + structured logs; carry over business KPIs; dashboards pre/post comparison; SSIS IR logs to Log Analytics; ADF pipeline failure alerts.
```kusto
requests | where cloud_RoleName in ("wcf-legacy-proxy","orders-api") | summarize count(), p95=percentile(duration,95) by cloud_RoleName
```

### Key code
```csharp
// CoreWCF hosting sample (CoreWCF.Http)
builder.Services.AddServiceModelServices();
app.UseServiceModel(sb => { sb.AddService<OrderService>(); sb.AddServiceEndpoint<OrderService, IOrderService>(new BasicHttpBinding(BasicHttpSecurityMode.Transport), "/OrderService.svc"); });
// Windows Service -> Worker/Function
[Function("PollInbox")] public Task Run([TimerTrigger("0 */5 * * * *")] TimerInfo t) => _job.RunOnceAsync();
```
```bicep
resource ir 'Microsoft.DataFactory/factories/integrationRuntimes@2018-06-01' = { parent: adf, name:'ssis-ir'
  properties:{ type:'Managed', typeProperties:{ computeProperties:{ location: loc, nodeSize:'Standard_D4_v3', numberOfNodes:1 }, ssisProperties:{ edition:'Standard', licenseType:'LicenseIncluded' } } } }
```
(Check node sizes/properties in current docs.)

### Trade-offs
- Rehost = fast/low risk, little benefit; refactor = benefits, higher cost/risk. CoreWCF reduces client changes but perpetuates SOAP.

### Follow-up ladder
- **Easy:** How migrate a Windows Service? *Timer/queue-triggered Function or Container Apps job; externalize state.*
- **Follow-up:** SSIS options? *SSIS IR lift-and-shift vs ADF/Functions rewrite.*
- **Hard:** How do SOAP clients keep working? *APIM SOAP API/passthrough, CoreWCF, versioned contracts, strangler routing.*
- **Tricky:** Job ran only on one server by design; now scales out → duplicates? *Singleton enforcement: timer singleton lock, deterministic Durable instance ID, DB lock, `WEBSITE_MAX_DYNAMIC_APPLICATION_SCALE_OUT`.*

---

## Design 23: Hybrid integration with on-prem via Logic Apps gateway/Service Bus

### Requirements & clarifying questions
- Cloud workloads must exchange data with on-prem systems (SQL, SAP, file shares, legacy APIs) that can't be exposed publicly. Ask: connectivity (ExpressRoute/VPN available)? Inbound vs outbound initiation? Latency? Firewalls allow outbound only? Message volume?

### Architecture
```
On-prem                                      Azure
[ERP/SQL/Files] <-- On-premises data gateway (outbound 443/AMQP only) <-- Logic Apps (Consumption connectors)
[ERP/SQL/Files] <-- Logic Apps Standard w/ VNET integration over VPN/ER (built-in connectors, no gateway)
[On-prem app] --outbound AMQP/HTTPS 443--> [Service Bus queue] (store & forward, no inbound firewall hole)
[Azure Relay Hybrid Connections] for synchronous request/response to on-prem listener
```

### Service choices & WHY
- **Service Bus (outbound-only):** best for async decoupling; on-prem consumer pulls. **Logic Apps + on-premises data gateway:** quick for supported connectors (SQL, file system, SAP). **Azure Relay (Hybrid Connections):** sync calls without opening inbound ports (“Service Bus relay” legacy WCF relay is Azure Relay; verify current positioning). **VPN/ExpressRoute + private endpoints** for full network integration and Logic Apps Standard.
- Gateway rejected as a hot-path: single point (cluster for HA), throughput limits.

### Identity & security
- Gateway: installed by service account, registered to a gateway resource (cluster two+ nodes for HA); Logic App connection uses Entra auth to the gateway resource. On-prem listener to SB/Relay: use Entra app/Arc/workload identity where possible (on-prem machines can use Azure Arc-enabled servers' managed identity—verify) or certificate-based service principal; avoid long-lived SAS keys. Least privilege: Sender on request queue, Receiver on reply queue.
- Network: outbound-only egress allowlists; private endpoints for SB with ExpressRoute private peering.

### Data flow & failure handling
- On-prem offline: messages accumulate in SB (TTL generous), process on recovery—watch quota & age. Request/reply: `ReplyTo` + `CorrelationId` queue (sessions for reply routing); client timeout + status.
- Idempotent on-prem writes (ERP double posting is costly). Logic App retry policies; gateway failure → alert.
- Gateway cluster for HA; upgrade cadence (monthly updates—verify) planned.

### Scaling & cost
- Gateway throughput bound by on-prem hardware; Relay billed per listener/message; SB per ops. Logic Apps Std plan cost vs Consumption actions.

### Observability
- Logic App run history/diagnostics; gateway performance counters & gateway status (Power Platform admin). Heartbeat from on-prem listener as custom metric; alert on queue age.
```kusto
customMetrics | where name=="OnPremListenerHeartbeat" | summarize last=max(timestamp) | where last < ago(10m)
```

### Key code
```csharp
// On-prem worker (outbound only)
await using var client = new ServiceBusClient("sb-hybrid.servicebus.windows.net",
    new ClientCertificateCredential(tenantId, clientId, certificate),
    new ServiceBusClientOptions { TransportType = ServiceBusTransportType.AmqpWebSockets });
await using var proc = client.CreateProcessor("to-onprem", new ServiceBusProcessorOptions { MaxConcurrentCalls = 4, AutoCompleteMessages = false });
proc.ProcessMessageAsync += async a => { await _erp.PostAsync(a.Message.Body); await a.CompleteMessageAsync(a.Message); };
proc.ProcessErrorAsync += a => { _log.LogError(a.Exception, "SB error {Src}", a.ErrorSource); return Task.CompletedTask; };
await proc.StartProcessingAsync();
```
Use AMQP over WebSockets (`ServiceBusTransportType.AmqpWebSockets`) if only 443 is open.

### Trade-offs
- Async hybrid is robust but not synchronous; relay/gateway add dependency on on-prem availability. VPN/ER enables private access but cost/ops.

### Follow-up ladder
- **Easy:** How reach on-prem without opening inbound ports? *Outbound-only: gateway, SB pull, Relay.*
- **Follow-up:** Gateway vs Service Bus? *Gateway = connector access synchronous; SB = decoupled messaging.*
- **Hard:** On-prem down for 2 days? *SB buffers (TTL/quota), idempotent catch-up, throttled replay.*
- **Tricky:** Only port 443 allowed and AMQP blocked? *AMQP over WebSockets.*

---

## Design 24: Cost-optimization review of an over-provisioned system

### Requirements & clarifying questions
- "Cut Azure spend 30% without hurting SLAs." Ask: scope (subscriptions)? SLAs/peaks? commitments already? non-prod usage patterns? Who owns each resource (tags)? Risk appetite?

### Architecture (review flow)
```
Cost Management + Advisor + Azure Resource Graph + Workbooks
 1 Measure: cost by RG/tag/service ; top 10 meters
 2 Right-size: CPU/mem/DTU < 20-30% (7-30 day P95)  3 Eliminate idle: unattached disks/IPs, empty plans, old slots
 4 Commit: Reservations / Savings Plans / Hybrid Benefit  5 Architect: serverless/consumption, tiering, caching
 6 Govern: budgets, alerts, Policy (SKU allowlist), auto-shutdown, tagging
```

### Service choices & WHY (typical findings for this stack)
- **App Service Plans**: consolidate low-use apps into shared plans; Premium → Standard if no needs; scale-in rules. **Functions**: Premium always-on → Consumption/Flex where cold-start OK. **Service Bus Premium** → Standard if no private endpoint/large messages/VNET need (verify trade); right-size messaging units (auto-scale). **SQL**: serverless/auto-pause for dev; elastic pools; reserved capacity; Hyperscale vs Business Critical needs. **Redis**: oversized tier; check Server Load & memory. **Storage**: lifecycle → Cool/Archive, delete old snapshots, reduce redundancy for non-critical (RA-GZRS → LRS). **Log Analytics**: biggest hidden cost—ingestion filtering, Basic logs, retention, sampling App Insights.
- Rejected: indiscriminate downsizing without metrics; relying on reservations before right-sizing (commit after).

### Identity & security
- Reader + Cost Management Reader for reviewers; changes through IaC/PR. Don't remove HA/security (private endpoints, WAF) for savings without risk sign-off. Policy to deny expensive SKUs in dev subscriptions.

### Data flow & failure handling (change safety)
- Stage changes: non-prod first, one change at a time, rollback plan; monitor SLIs 1-2 weeks; load test before downsizing prod. Keep headroom for peak (autoscale min/max).

### Scaling & cost
- Autoscale rules (schedule + metric); dev/test shutdown schedules; spot instances for batch (AKS/VMSS); reserved for steady baseline (1y/3y).

### Observability
```kusto
Usage | where TimeGenerated > ago(30d) | summarize GB=sum(Quantity)/1000 by DataType | top 10 by GB desc
AppRequests | summarize count() by bin(TimeGenerated,1h)   // identify true peak vs provisioned
```
```
Resource Graph: resources | where type =~ 'microsoft.compute/disks' and properties.diskState == 'Unattached'
```
- Budgets with alerts at 50/80/100%; anomaly alerts in Cost Management; track unit cost (cost per 1k orders).

### Key code
```bicep
resource budget 'Microsoft.Consumption/budgets@2023-05-01' = { name:'monthly', properties:{ category:'Cost', amount:5000, timeGrain:'Monthly'
  timePeriod:{ startDate:'2026-01-01' }, notifications:{ n80:{ enabled:true, operator:'GreaterThan', threshold:80, contactEmails:['finops@contoso.com'] } } } }
```
```json
{ "logging": { "applicationInsights": { "samplingSettings": { "isEnabled": true, "excludedTypes": "Request;Exception" } } }, "logLevel": { "default": "Warning", "Function": "Information" } }
```

### Trade-offs
- Savings vs resilience/performance; commitments reduce flexibility; consolidation increases blast radius.

### Follow-up ladder
- **Easy:** First step? *Measure and tag; find top spenders; Advisor.*
- **Follow-up:** Reservations vs savings plans? *Reservation: specific resource/region, bigger discount; savings plan: flexible compute spend.*
- **Hard:** Log Analytics is 40% of the bill? *Analyze Usage, DCR transforms, Basic logs, lower verbosity, sampling, retention tiers.*
- **Tricky:** Downsize broke at month-end peak? *Use P95/P99 over full business cycle; autoscale headroom; load test.*

---

## Design 25: Disaster recovery runbook design with RPO/RTO

### Requirements & clarifying questions
- Define and prove recovery. Ask: business-critical tiers (tier 0-3)? RPO/RTO per tier? Regulatory? Failure modes in scope (zone, region, corruption, ransomware, human error, identity/tenant compromise)? Budget? Dependencies (DNS, Entra, third parties)?

### Architecture
```
Tiering:  T0 (orders): RPO<=5m RTO<=30m  -> warm standby, SQL FOG, GZRS, SB geo-DR/replication
          T1: RPO<=1h RTO<=4h            -> geo-restore/backup, IaC redeploy
          T2: RPO 24h RTO 24-72h         -> backups only
Layers:  Zone redundancy (in-region HA) -> Region failover -> Backup/restore (corruption) -> Immutable/offline copies (ransomware)
Runbook: Detect -> Decide (authority) -> Communicate -> Execute -> Validate -> Failback -> Postmortem
```

### Service choices & WHY
- Per service: **SQL** FOG + PITR backups (corruption → PITR, not failover!); **Storage** GZRS + soft delete + versioning + immutability; **SB** geo-DR/geo-replication (verify); **Key Vault** paired-region replication + backup/restore of secrets; **Functions/App** redeploy from IaC to secondary; **Front Door** traffic switch; **Cosmos** multi-region writes/failover priority; **Azure Backup/Site Recovery** for VMs. **Entra ID** is global but have break-glass accounts.

### Identity & security
- Break-glass accounts (cloud-only, excluded from CA, monitored), PIM for DR executors, runbook automation identity (UAMI) with scoped roles; secrets/certs for secondary region pre-provisioned; backups protected from deletion (soft delete, immutable vault, separate subscription/RBAC).

### Data flow & failure handling
- RPO is determined by replication lag/backup frequency; RTO = detection + decision + execution + validation. Measure each in drills.
- Runbook steps (T0 regional outage): 1 confirm outage (Service Health + probes) 2 declare incident, name commander 3 SQL FOG failover (`Switch-AzSqlDatabaseFailoverGroup`, forced if primary unreachable: data-loss allowed) 4 SB alias failover 5 scale/start secondary apps 6 Front Door shift 7 smoke tests/synthetic 8 replay outbox/unsent messages (idempotent) 9 communicate 10 failback plan.
- Corruption scenario: PITR to new DB, diff/repair, swap names; ransomware: restore from immutable backups.

### Scaling & cost
- Warm standby cost ≈ 40-60% extra; cold ≈ 10-20%. Tier apps to match; use Pilot Light for T1. Test cost (drills) budgeted.

### Observability
- Replication lag metrics, backup success, Service Health alerts, synthetic probes from secondary region; DR dashboard with "time since last successful drill/restore test".
```kusto
AzureActivity | where OperationNameValue has "failoverGroups" | project TimeGenerated, Caller, OperationNameValue, ActivityStatusValue
```

### Key code
```powershell
# Planned failover (no data loss)
Switch-AzSqlDatabaseFailoverGroup -ResourceGroupName rg-b -ServerName sql-b -FailoverGroupName fog-orders
# Forced (possible data loss)
Switch-AzSqlDatabaseFailoverGroup -ResourceGroupName rg-b -ServerName sql-b -FailoverGroupName fog-orders -AllowDataLoss
```
```
az servicebus georecovery-alias fail-over -g rg-b --namespace-name sb-b --alias sb-orders-alias
```
```bicep
resource fog 'Microsoft.Sql/servers/failoverGroups@2023-05-01-preview' = { parent: sqlA, name:'fog-orders'
  properties:{ readWriteEndpoint:{ failoverPolicy:'Manual' }, partnerServers:[{ id: sqlB.id }], databases:[ db.id ] } }
```
(Verify CLI/cmdlet syntax in current docs.)

### Trade-offs
- Automatic vs manual failover; cost vs RTO; frequent drills disrupt vs untested plans fail. Untested DR is not DR.

### Follow-up ladder
- **Easy:** RPO vs RTO? *Data loss tolerance vs downtime tolerance.*
- **Follow-up:** HA vs DR? *HA in-region (zones); DR across regions.*
- **Hard:** Someone deleted data and it replicated? *Replication copies mistakes; use PITR/versioning/soft delete/immutable backups.*
- **Tricky:** Region outage also impacts your runbook tooling/pipeline? *Host automation/runbooks in secondary region or independent; offline copies; break-glass access.*

---

## Design 26: Monitoring & alerting design for the whole platform

### Requirements & clarifying questions
- Observability for Service Bus, Functions, Logic Apps, Blob, SQL, Redis, APIM, identity. Ask: SLOs? on-call model? noise tolerance? multi-subscription/region? retention/compliance? cost ceiling?

### Architecture
```
Apps (OpenTelemetry/App Insights SDK, workspace-based) ----\
Azure resources (Diagnostic Settings, metrics) --------------+--> [Log Analytics workspace(s)] --> KQL, Workbooks, Dashboards (Grafana)
Platform: Activity Log, Service Health, Entra sign-in/audit -/                 |
                                                           Alert rules (metric/log/activity/availability) -> Action Groups
                                                                -> Teams/PagerDuty/ITSM/Logic App auto-remediation
 SLOs: availability, latency P95, freshness (queue age), correctness (DLQ=0)
```

### Service choices & WHY
- **Workspace-based Application Insights** (classic is retired; verify) on one workspace per environment (or central with RBAC); **Azure Monitor metric alerts** for fast resource signals; **log alerts** for KQL logic; **availability tests** for synthetic; **Action Groups** with severity routing; **Alert processing rules** for maintenance suppression. **Managed Grafana/Workbooks** for dashboards. OpenTelemetry distro (`Azure.Monitor.OpenTelemetry.AspNetCore`) is the current .NET path (verify).

### Identity & security
- Monitoring Reader for engineers; `Log Analytics Reader` with table-level RBAC for sensitive tables; apps send telemetry with Entra auth (`Monitoring Metrics Publisher` on App Insights; disable local auth `DisableLocalAuth`—verify supported config) ; AMPLS private link; scrub PII via telemetry processors.

### Data flow & failure handling
- Alert on symptoms (user impact, SLO burn) over causes; page only actionable. Multi-window burn rate alerts. Missing-data alerts (heartbeats). Auto-remediation (restart, scale) via Logic App/Automation for known issues; alert fatigue reviews.
- Sampling: adaptive sampling for requests/deps; exclude exceptions/audits from sampling; telemetry itself must degrade gracefully (non-blocking).

### Scaling & cost
- Ingestion cost control: sampling, log levels, DCR transforms, daily cap (careful), commitment tier, Basic logs for verbose tables, retention policy per table.

### Observability — alert catalog (core)
| Area | Signal | Alert |
|---|---|---|
| Service Bus | ActiveMessages age, DeadletteredMessages>0, ServerErrors, ThrottledRequests | Sev2/3 |
| Functions | failures %, execution duration P95, host memory, cold starts, queue trigger lag | Sev2 |
| Logic Apps | Run failures, trigger failures, throttled actions | Sev3 |
| Storage | Availability, E2E latency, 429/503, capacity | Sev3 |
| SQL | CPU/DTU, deadlocks, DTU storage, failed connections, geo-replication lag | Sev2 |
| Redis | Server Load, memory %, evictions, connected clients | Sev3 |
| APIM | 5xx rate, backend duration, capacity | Sev2 |
| Entra/KV | Failed sign-ins spikes, KV 429s/403s, secret near expiry | Sev3 |
```kusto
// SLO burn (availability 99.9%, 1h window)
requests | where timestamp > ago(1h) | summarize bad=countif(success==false), total=count() | extend burn = (bad*1.0/total)/0.001 | where burn > 14.4
// End-to-end trace
union requests, dependencies, traces, exceptions | where operation_Id == "<id>" | order by timestamp asc
```

### Key code
```csharp
builder.Services.AddOpenTelemetry().UseAzureMonitor(o => { o.ConnectionString = cfg["APPLICATIONINSIGHTS_CONNECTION_STRING"]; o.Credential = new DefaultAzureCredential(); o.SamplingRatio = 0.2f; });
using var activity = Telemetry.Source.StartActivity("ProcessOrder"); activity?.SetTag("orderId", id);
```
```bicep
resource ag 'Microsoft.Insights/actionGroups@2023-01-01' = { name:'ag-oncall', location:'global'
  properties:{ groupShortName:'oncall', enabled:true, emailReceivers:[{ name:'team', emailAddress:'oncall@contoso.com', useCommonAlertSchema:true }] } }
resource qa 'Microsoft.Insights/scheduledQueryRules@2023-03-15-preview' = { name:'dlq-or-failures', location: loc
  properties:{ severity:2, enabled:true, evaluationFrequency:'PT5M', windowSize:'PT15M', scopes:[law.id]
    criteria:{ allOf:[{ query:'requests | where success==false', timeAggregation:'Count', operator:'GreaterThan', threshold:20 }] }
    actions:{ actionGroups:[ag.id] } } }
```
```json
{ "version":"2.0", "logging": { "applicationInsights": { "samplingSettings": { "isEnabled": true, "excludedTypes": "Request;Exception" } } } }
```

### Trade-offs
- More signal = more cost/noise; central workspace simplifies correlation but couples RBAC/retention; per-team workspaces improve isolation.

### Follow-up ladder
- **Easy:** App Insights vs Log Analytics? *App Insights = APM feature layer; data stored in a Log Analytics workspace (workspace-based).*
- **Follow-up:** Metric vs log alert? *Metric: near-real-time cheap; log: flexible KQL, more latency/cost.*
- **Hard:** Trace one request across Functions → SB → Functions? *W3C trace context propagation; `operation_Id` join; SB SDK/Functions propagate `Diagnostic-Id`.*
- **Tricky:** No alert fired when system silently stopped? *Alert on absence (heartbeat, count==0), queue age, synthetic probes.*

---

*End of Z13. Re-verify SKUs, quotas, built-in role GUIDs/names, and preview features in current Microsoft Learn docs before quoting them in an interview.*

