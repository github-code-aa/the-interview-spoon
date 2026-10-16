# Z14 — Azure Production Troubleshooting Playbook (43 Incidents)

## How to use this guide
- **Study mode:** read one incident, cover the Diagnosis section, and try to recall the steps.
- **Incident mode:** jump to the symptom, run "First questions", then follow Diagnosis in order.
- **Interview mode:** each incident has a STAR sketch plus a 3-rung follow-up ladder (Follow-up -> Hard -> Tricky). Replace the sketch with your own real story and numbers.
- Table names: App Insights **workspace-based** tables (Log Analytics) are `AppRequests`, `AppDependencies`, `AppExceptions`, `AppTraces`, `AppEvents`, `AppMetrics`, `AppPerformanceCounters`. **Classic** (Application Insights Logs blade on classic resources) names are `requests`, `dependencies`, `exceptions`, `traces`, `customEvents`, `customMetrics`, `performanceCounters`. Column names also differ (workspace: `TimeGenerated`, `Name`, `DurationMs`, `Success`, `ResultCode`, `OperationId`, `AppRoleName`; classic: `timestamp`, `name`, `duration`, `success`, `resultCode`, `operation_Id`, `cloud_RoleName`). Queries below show the workspace-based form with the classic form noted where it matters.
- Anything about limits, quotas, defaults, or SKU behavior: **verify in current docs** — they change.
- Placeholders: `<rg>`, `<kv>`, `<sa>`, `<sbns>`, `<func>`, `<principalId>`, `<clientId>`.

## Table of contents
1. Key Vault 403 / AuthorizationFailed after deployment
2. DefaultAzureCredential works locally, fails in Azure (and the reverse)
3. UAMI not used because AZURE_CLIENT_ID missing
4. SAMI role assignment lost after resource recreate
5. Blob "not authorized" with managed identity (control vs data plane)
6. Service Bus MessageLockLostException
7. Service Bus messages stuck / not consumed
8. Dead-letter queue growing
9. Duplicate processing
10. Message ordering broken
11. Service Bus quota / throttling
12. Service Bus trigger scaling overwhelms downstream
13. Function not triggering after deployment
14. Timer function running twice
15. Function cold start latency
16. "Host not running" / AzureWebJobsStorage problems
17. Durable Functions stuck or non-deterministic replay
18. Function timeout
19. Logic App trigger not firing
20. Logic App 429 / throttling
21. Logic App connection auth expired / MI connector failing
22. Logic App large payload failure
23. Blob 403 AuthorizationFailure from network rules
24. Blob 409 / 412 conflicts and lease problems
25. Blob trigger missing files
26. SAS token expired / leaked
27. Redis timeouts under load
28. Redis connection storm after failover
29. Redis stale / serialization-broken cache after deploy
30. Redis memory full / evictions
31. Missing telemetry in App Insights
32. Double telemetry
33. High App Insights cost
34. Broken correlation across messaging
35. Alerts not firing / alert storm
36. Private endpoint DNS resolves public IP
37. SNAT port exhaustion
38. Azure SQL transient errors / failover
39. Slot swap caused outage
40. Pipeline deployment fails with role-assignment permission error
41. Secret rotation broke the app
42. Cost spike investigation
43. Cross-region failover drill gone wrong
44. Appendix A: Azure triage checklist (one page)
45. Appendix B: KQL cheat sheet

---

## Incident 1: Key Vault 403 / AuthorizationFailed after deployment
**Symptom:** App returns 500 at startup or on secret read. Exception: `Azure.RequestFailedException: Caller is not authorized to perform action on resource ... Code: Forbidden` / `(Forbidden) The user, group or application 'appid=...;oid=...;iss=...' does not have secrets get permission on key vault '<kv>'` (access-policy vaults) or `ForbiddenByRbac` / `Forbidden ... Action: 'Microsoft.KeyVault/vaults/secrets/getSecret/action'` (RBAC vaults). Firewall variant: `ForbiddenByFirewall` / "Client address is not authorized and caller is not a trusted service".

**First questions to ask**
- Did anything change: new deployment, new identity, vault recreated, role assignment in the same template?
- Is the vault on **RBAC** or **access policies** (permission model)?
- Which identity is the app really using (oid in the error)?
- Is the vault firewall / private endpoint enabled? Is the caller in an allowed network?

**Diagnosis steps**
1. Read the `oid=` in the error and compare to the app's principal:
   `az webapp identity show -g <rg> -n <app> --query principalId` (Functions: `az functionapp identity show`).
2. Check permission model: `az keyvault show -n <kv> --query "{rbac:properties.enableRbacAuthorization,fw:properties.networkAcls}"`.
3. Check assignments: `az role assignment list --assignee <principalId> --scope $(az keyvault show -n <kv> --query id -o tsv) -o table`. Need `Key Vault Secrets User` (get/list secret) for RBAC.
4. Vault diagnostics (enable `AuditEvent` to Log Analytics) then:
```kql
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.KEYVAULT" and Category == "AuditEvent"
| where ResultSignature has "403" or ResultType == "Forbidden"
| project TimeGenerated, OperationName, CallerIPAddress, identity_claim_oid_g, requestUri_s, ResultSignature, httpStatusCode_d
| order by TimeGenerated desc
```
   (If the resource-specific table is enabled for your vault, use `AZKVAuditLogs`; verify columns in current docs.)
5. Portal: Key Vault -> Networking (firewall, "Allow trusted Microsoft services"), Access control (IAM) -> Check access.
6. Network path: from Kudu console `nameresolver <kv>.vault.azure.net` and `tcpping`; outbound IP vs vault firewall.

**Root causes (ranked)**
1. Role assignment missing/not yet propagated (assignment created in same deployment; Entra/ARM caching can take minutes, and tokens cache the old roles for up to ~hours — verify in current docs).
2. Wrong identity: SAMI recreated (new principalId) or app uses UAMI but role was given to SAMI.
3. Vault uses access policies but you granted RBAC (or vice versa).
4. Firewall/private endpoint blocks the app (App Service outbound IPs not allowed, no VNet integration).
5. Role at wrong scope or wrong role (Reader/Contributor are control plane only; they do not read secrets).

**Fix:** assign `Key Vault Secrets User` at vault (or secret) scope to the correct principal; restart the app to refresh the token cache; allow the VNet subnet / enable VNet integration + private endpoint; align permission model.

**Prevention:** declare the role assignment in Bicep with `principalType: 'ServicePrincipal'` and dependsOn the identity; deploy RBAC before the app that needs it at startup; use Key Vault references with retries; add startup health check on secret access; keep one permission model per vault via policy.

**STAR sketch:** *Situation:* new environment rollout, API 500s at startup. *Task:* restore in the release window. *Action:* matched the oid in the 403 to the app's SAMI, found role assignment raced the app deployment, assigned Secrets User, restarted. *Result:* healthy in minutes; added Bicep ordering and a smoke test that reads a secret.

**Follow-up ladder**
- *Follow-up:* Why still 403 right after assigning the role? — RBAC propagation delay plus token caching; wait, restart/refresh token.
- *Hard:* Role is correct but only some instances fail. — Different outbound IPs/regions vs firewall, or instances holding stale tokens; check `CallerIPAddress` in AuditEvent.
- *Tricky:* "Allow trusted Microsoft services" is on and still blocked. — Trusted-service bypass applies to specific first-party services, not App Service/Functions code calling the vault; you need VNet/private endpoint or allow IPs.

## Incident 2: DefaultAzureCredential works locally, fails in Azure (and the reverse)
**Symptom:** `Azure.Identity.CredentialUnavailableException: DefaultAzureCredential failed to retrieve a token from the included credentials` listing EnvironmentCredential, ManagedIdentityCredential, VisualStudioCredential, AzureCliCredential... Or the reverse: works in Azure, fails on dev box with `ManagedIdentityCredential authentication unavailable`/`AzureCliCredential: Please run 'az login'`.

**First questions to ask**
- Where does it fail (App Service, Function, AKS, container, CI agent)? Is managed identity enabled?
- SAMI or UAMI? Is `AZURE_CLIENT_ID` set?
- Locally: which tool is signed in (VS, `az`, VS Code), which tenant/account?

**Diagnosis steps**
1. Enable Azure.Identity logging: `using var listener = AzureEventSourceListener.CreateConsoleLogger();` or logging category `Azure.Identity` at Information in App Insights.
2. Azure side: `az webapp identity show -g <rg> -n <app>`; Kudu -> Environment shows `IDENTITY_ENDPOINT`/`IDENTITY_HEADER` (App Service MI endpoint) — absent means MI not enabled.
3. Locally: `az account show`, `az account get-access-token --resource https://vault.azure.net`.
4. Check env vars: `AZURE_TENANT_ID`, `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET` stray values in the environment make EnvironmentCredential win and fail with `AADSTS7000215`/`AADSTS700016`.
5. Different permissions: your user has the role locally; the MI does not. Compare `az role assignment list --assignee`.
6. In traces query the exception text:
```kql
AppExceptions
| where TimeGenerated > ago(1h) and (OuterMessage has "DefaultAzureCredential" or ExceptionType has "CredentialUnavailable")
| project TimeGenerated, AppRoleName, OuterMessage
```
(classic: `exceptions`, `outerMessage`, `cloud_RoleName`)

**Root causes (ranked)**
1. Identity has no RBAC in Azure while the developer's own account has it locally (authentication OK, authorization fails — often misreported as credential error).
2. Managed identity not enabled, or UAMI not specified (see Incident 3).
3. Stale env vars (service principal) shadow the chain.
4. Local: not signed in, wrong tenant, VS account differs from `az` account, MFA/CA policy.
5. Chain timeout: ManagedIdentity probe slow in containers; IMDS unreachable.

**Fix:** enable MI and grant roles; set `AZURE_CLIENT_ID` for UAMI; pin credential explicitly (`new ManagedIdentityCredential(clientId)` in Azure, `AzureCliCredential` locally) or use `ChainedTokenCredential`; sign in with the right tenant (`az login --tenant`).

**Prevention:** avoid DefaultAzureCredential in production hot paths (explicit credential by environment); integration test identity in a deployed slot; document local-dev role assignments to a dev security group.

**STAR sketch:** *S:* feature passed in dev, failed in staging. *T:* find why in an hour. *A:* turned on Azure.Identity logging, saw ManagedIdentityCredential succeeded but Blob returned 403 — roles only on the dev's user. *R:* assigned the role to the MI; added a deploy-time identity test.

**Follow-up ladder**
- *Follow-up:* Order of DefaultAzureCredential? — Environment, Workload Identity, Managed Identity, (VS, VS Code, CLI, PowerShell, azd depending on SDK version); verify in current docs.
- *Hard:* How to prevent dev-only credentials from being used in prod? — `ExcludeVisualStudioCredential` etc., or choose credential by `IHostEnvironment`.
- *Tricky:* It works in Azure only after a few seconds delay each cold start? — Chain probes credentials sequentially; failed earlier probes add latency; pin the credential.

## Incident 3: UAMI not used because AZURE_CLIENT_ID missing
**Symptom:** App with only a user-assigned identity: `ManagedIdentityCredential authentication failed: ... Identity not found` or `Multiple user assigned identities exist, please specify the clientId / resourceId of the identity in the token request` (when several UAMIs are attached). If the app has SAMI+UAMI, the SAMI is silently used and you get 403.

**First questions to ask**
- How many identities are attached? Which one holds the role?
- Is client ID passed in code or via `AZURE_CLIENT_ID`?
- Same code on Functions/App Service/AKS?

**Diagnosis steps**
1. `az webapp identity show -g <rg> -n <app>` -> `userAssignedIdentities` keys and `clientId`s.
2. `az identity show -g <rg> -n <uami> --query "{c:clientId,p:principalId}"`.
3. `az webapp config appsettings list -g <rg> -n <app> --query "[?name=='AZURE_CLIENT_ID']"`.
4. Look at the `oid` in resource-side logs (Key Vault AuditEvent `identity_claim_oid_g`, Storage `StorageBlobLogs.RequesterObjectId`) — if it equals SAMI principalId, UAMI isn't used.
5. Key Vault references: with UAMI you must set `keyVaultReferenceIdentity` on the site.

**Root causes (ranked)**
1. `AZURE_CLIENT_ID` (or `ManagedIdentityClientId` in `DefaultAzureCredentialOptions`) not set.
2. Set to the **principalId/objectId** instead of **clientId**.
3. Role assigned to the wrong identity.
4. Identity not attached to that slot/app (slots have their own identity config).
5. Extension-based bindings (Service Bus) need `__clientId` and `__credential` settings (see below).

**Fix:** set `AZURE_CLIENT_ID=<clientId>`; or code `new DefaultAzureCredential(new DefaultAzureCredentialOptions{ ManagedIdentityClientId = clientId })`. For Functions identity-based connections: `ServiceBusConnection__fullyQualifiedNamespace`, `ServiceBusConnection__credential=managedidentity`, `ServiceBusConnection__clientId=<clientId>`. Key Vault references: `az webapp update --set keyVaultReferenceIdentity=<uami resourceId>` (verify current CLI form).

**Prevention:** one config module that requires the client ID; Bicep outputs clientId into app settings; startup log of which identity is resolved.

**STAR sketch:** *S:* migration from SAMI to UAMI so identity survives recreation. *T:* no downtime. *A:* pre-granted roles to the UAMI, added `AZURE_CLIENT_ID`, verified oid in audit logs per slot. *R:* swap succeeded without auth errors.

**Follow-up ladder**
- *Follow-up:* clientId vs principalId vs resourceId? — clientId (appId) used by SDK; principalId (objectId) used in role assignments; resourceId used in ARM/key vault reference config.
- *Hard:* Why do Functions need `__clientId` separately from `AZURE_CLIENT_ID`? — Identity-based connection config is per connection name and takes precedence for that binding; verify in current docs.
- *Tricky:* The UAMI is in another subscription — works? — Yes if attached properly and roles granted on the target; ensure the identity is in the same tenant.

## Incident 4: SAMI role assignment lost after resource recreate
**Symptom:** After redeploying / delete-and-recreate of an App Service/Function/Logic App, everything returns 401/403. Role assignments list shows `Identity not found` (orphaned) entries.

**First questions to ask**
- Was the resource deleted and recreated (not just updated)? Pipeline using `--mode Complete` or rename?
- Are role assignments defined in IaC or created manually?

**Diagnosis steps**
1. `az webapp identity show -g <rg> -n <app> --query principalId` — compare to previous principalId in IaC state/logs.
2. `az role assignment list --all --query "[?principalName==''||principalName==null]" -o table` (orphans show with unresolved principals), or in portal IAM -> Role assignments -> "Identity not found".
3. Activity Log: filter Operation `Delete Web App`, `Create or Update Role Assignment`.
```kql
AzureActivity
| where TimeGenerated > ago(7d) and OperationNameValue has_any ("MICROSOFT.WEB/SITES/DELETE","MICROSOFT.AUTHORIZATION/ROLEASSIGNMENTS")
| project TimeGenerated, Caller, OperationNameValue, ActivityStatusValue, ResourceGroup, _ResourceId
```
4. Confirm resource-side `oid` differs from the one assigned.

**Root causes (ranked)**
1. SAMI lifecycle is tied to the resource; recreate = new principalId, old assignments orphaned.
2. Role assignments were manual, not in IaC.
3. Assignment in IaC ran before the identity existed / deterministic GUID name collision (`RoleAssignmentExists`).

**Fix:** reassign roles to the new principalId; delete orphans; for Bicep use `guid(scope.id, principalId, roleDefId)` names.

**Prevention:** use **UAMI** for resources that get recreated or for blue/green; keep all role assignments in IaC; avoid delete-and-recreate; add a post-deploy RBAC verification script.

**STAR sketch:** *S:* nightly environment rebuild nuked access. *T:* make it repeatable. *A:* moved to a UAMI with pre-provisioned roles and codified assignments. *R:* rebuilds no longer cause 403s.

**Follow-up ladder**
- *Follow-up:* SAMI vs UAMI trade-off? — SAMI simple/lifecycle-bound; UAMI reusable, pre-authorizable, shared (larger blast radius).
- *Hard:* Bicep `RoleAssignmentExists` on redeploy? — Name (GUID) must be deterministic from scope+principal+role; if principal changes the name changes but the old stays; clean orphans.
- *Tricky:* Does stop/start or slot swap change SAMI? — No; deletion does. Slots have their own SAMI (distinct principal).

## Incident 5: Blob "not authorized" with managed identity (control vs data plane)
**Symptom:** `Azure.RequestFailedException: This request is not authorized to perform this operation using this permission. Status: 403 (AuthorizationPermissionMismatch)`. Owner/Contributor on the storage account yet the app cannot read blobs.

**First questions to ask**
- Which roles does the identity hold, at what scope?
- Is shared key access disabled? Is the app using a connection string or a token?
- Same container/account? Cross-tenant?

**Diagnosis steps**
1. `az role assignment list --assignee <principalId> --scope <storage account id> -o table`.
2. Look for data roles: `Storage Blob Data Reader/Contributor/Owner`. Owner/Contributor/Reader are management-plane roles (`Microsoft.Storage/*`) and grant no data access via Entra (they can list keys, which is different).
3. Diagnostics:
```kql
StorageBlobLogs
| where TimeGenerated > ago(1h) and StatusCode == 403
| project TimeGenerated, OperationName, AuthenticationType, StatusText, RequesterObjectId, Uri, CallerIpAddress
```
`StatusText` shows `AuthorizationPermissionMismatch` vs `AuthorizationFailure` (network) — key distinction.
4. Check ABAC conditions on the role assignment (conditions can restrict containers/prefixes).

**Root causes (ranked)**
1. Management role only; missing a Blob Data role.
2. Role at wrong scope or for the wrong identity (SAMI/UAMI).
3. Propagation delay.
4. ABAC condition not satisfied.
5. For queue/table: different data roles (Queue Data..., Table Data...).

**Fix:** assign `Storage Blob Data Contributor` (or Reader) at the narrowest scope (container) to the correct principal; wait/restart.

**Prevention:** least-privilege data roles in IaC; policy to disable shared key; integration test touching blob in each environment.

**STAR sketch:** *S:* migration to identity-based storage access. *T:* remove keys. *A:* diagnosed `AuthorizationPermissionMismatch`, explained control vs data plane to the team, applied data roles via Bicep. *R:* zero key usage, audit-clean.

**Follow-up ladder**
- *Follow-up:* Why do Owners still get blocked? — Owner is control-plane (`actions`), data access needs `dataActions`.
- *Hard:* Owner can browse blobs in portal though? — Portal may use account key if allowed ("Authentication method: Access key").
- *Tricky:* Anything where a control-plane role does grant data access? — `Microsoft.Storage/storageAccounts/listKeys/action` lets you get keys and use Shared Key, a bypass unless shared key is disabled.

## Incident 6: Service Bus lock lost / MessageLockLostException
**Symptom:** `Azure.Messaging.ServiceBus.ServiceBusException: The lock supplied is invalid. Either the lock expired, or the message has already been removed from the queue. (MessageLockLost)`. Messages reappear with `DeliveryCount` incrementing; eventually dead-lettered with `MaxDeliveryCountExceeded`.

**First questions to ask**
- Receive mode (PeekLock)? Lock duration on the entity? Processing time p95/p99?
- Using Function trigger or `ServiceBusProcessor`? `AutoCompleteMessages`, `MaxAutoLockRenewalDuration` settings?
- Was there a pod/instance restart, or a connection drop?

**Diagnosis steps**
1. `az servicebus queue show -g <rg> --namespace-name <sbns> -n <q> --query "{lock:lockDuration,max:maxDeliveryCount,dl:deadLetteringOnMessageExpiration}"`.
2. Compare processing duration with lock duration:
```kql
AppDependencies
| where TimeGenerated > ago(6h) and Type == "Azure Service Bus"
| summarize p50=percentile(DurationMs,50), p95=percentile(DurationMs,95), p99=percentile(DurationMs,99) by Name
```
(classic: `dependencies`, `duration`, `type`)
3. Find the exceptions: `AppExceptions | where ExceptionType has "ServiceBusException" and OuterMessage has "lock"`.
4. Function: check `host.json` -> `extensions.serviceBus.messageHandlerOptions.maxAutoLockRenewalDuration` (v4 ext: `maxAutoLockRenewalDuration`) — verify exact setting names per extension version.
5. Look for CPU/thread starvation (sync-over-async) that delays renewals: `AppPerformanceCounters`/Live Metrics.

**Root causes (ranked)**
1. Processing exceeds lock duration (default 1 min, max 5 min; verify) and auto-renew is off or capped lower than the work.
2. Instance restart/scale-in/connection loss -> lock invalid.
3. Thread/CPU starvation prevented renewal.
4. Settling the message on a different receiver/link than received it (e.g., after reconnect).
5. Message already settled twice in code (manual `CompleteMessageAsync` with `AutoCompleteMessages=true`).

**Fix:** set `MaxAutoLockRenewalDuration` >= worst-case processing; reduce per-message work (offload long jobs, use Durable); make handler idempotent; remove double-settle; fix async blocking.

**Prevention:** alert on `DeliveryCount>1` ratio; load test with realistic duration; keep lock duration ≥ p99 plus buffer; idempotent consumers.

**STAR sketch:** *S:* spike in redelivered orders. *T:* stop duplicates. *A:* correlated lock exceptions to a slow downstream (p99 90s vs 60s lock), enabled renewal to 5 min and added timeouts to the downstream call. *R:* lock-lost errors went to zero.

**Follow-up ladder**
- *Follow-up:* Does lock renewal have a ceiling? — Renewal works until you stop; SDK auto-renew bounded by `MaxAutoLockRenewalDuration`; broker still enforces message TTL/delivery rules.
- *Hard:* Is increasing lock duration enough? — No; if the instance dies lock expires anyway; need idempotency.
- *Tricky:* Lock lost on a session queue? — Session lock (`SessionLockLostException`) applies to the session; renew via `SessionLockedUntil`/`MaxAutoLockRenewalDuration` on the session processor.

## Incident 7: Service Bus messages stuck / not consumed
**Symptom:** Active message count rising; consumers idle or error-looping; or messages invisible (scheduled/locked).

**First questions to ask**
- Active vs scheduled vs dead-letter vs transfer counts? Any consumers connected?
- Subscription filters correct? Session-enabled entity?
- Consumer disabled, or throttled?

**Diagnosis steps**
1. `az servicebus queue show -g <rg> --namespace-name <sbns> -n <q> --query countDetails`. For topics: `az servicebus topic subscription show ... --query countDetails`.
2. Portal: Service Bus -> entity -> Service Bus Explorer (peek), check Status (Active/Disabled/ReceiveDisabled).
3. Metrics: `ActiveMessages`, `DeadletteredMessages`, `IncomingMessages`, `CompleteMessage`, `ServerErrors`, `ThrottledRequests`.
4. Sessions: if `requiresSession=true` and the consumer is not session-aware, messages never delivered. Check `az servicebus queue show --query requiresSession`.
5. Function consumer: Live Metrics, `AppTraces | where Message has "Listener"`, check function not disabled (`AzureWebJobs.<name>.Disabled`).
6. Subscription rules: `az servicebus topic subscription rule list ...` — a `$Default` rule removed or filter mismatch makes the subscription empty (the opposite: messages missing).
7. Operational logs (if enabled): `AzureDiagnostics | where ResourceProvider=="MICROSOFT.SERVICEBUS"` / `ASBOperationalLogs`? Verify table names by diagnostic setting destination (resource-specific vs AzureDiagnostics).

**Root causes (ranked)**
1. Consumer down/disabled/misconfigured connection or identity (no Receiver role: `Azure Service Bus Data Receiver`).
2. Poison message loop consuming capacity (retries).
3. Session-required queue with non-session consumer, or one session locked by a stuck receiver.
4. Messages scheduled for future (`ScheduledEnqueueTimeUtc`).
5. Throttling (Premium CU saturation) or networking (firewall/PE).

**Fix:** restore consumer/identity/roles; move poison to DLQ; release stuck session by restarting the holder; fix filters.

**Prevention:** alert on ActiveMessages age (oldest message), consumer heartbeat, DLQ count; synthetic message round trip.

**STAR sketch:** *S:* queue depth growing overnight. *T:* drain and root-cause. *A:* found function app disabled by a config swap setting `AzureWebJobs.X.Disabled=true` slot-sticky. *R:* re-enabled, drained backlog, added a depth alert.

**Follow-up ladder**
- *Follow-up:* Which metric best shows backlog health? — Active messages plus oldest-message age (not directly a metric; derive via peek/consumer logs; verify).
- *Hard:* Count is 0 but messages "missing"? — Check DLQ, scheduled, deferred, transfer DLQ, and auto-delete-on-idle.
- *Tricky:* Receiver connected but nothing arrives? — Prefetch holding locks, session lock held, or filter mismatch.

## Incident 8: Dead-letter queue growing
**Symptom:** `DeadletteredMessages` metric climbing; downstream data missing. Messages carry `DeadLetterReason` and `DeadLetterErrorDescription`.

**First questions to ask**
- Which reasons? `MaxDeliveryCountExceeded`, `TTLExpiredException`, `HeaderSizeExceeded`, app-explicit, or filter evaluation exceptions?
- Started after a deploy/schema change?

**Diagnosis steps**
1. Peek DLQ: Service Bus Explorer -> Dead-letter tab; read `DeadLetterReason`.
2. SDK: `receiver = client.CreateReceiver(queue, new ServiceBusReceiverOptions{ SubQueue = SubQueue.DeadLetter })`; peek and group by `DeadLetterReason`.
3. `az servicebus queue show ... --query "{max:maxDeliveryCount,ttl:defaultMessageTimeToLive,dlOnExp:deadLetteringOnMessageExpiration}"`.
4. Correlate with exceptions at the dead-letter time:
```kql
AppExceptions
| where TimeGenerated > ago(24h)
| summarize n=count() by ProblemId, bin(TimeGenerated,15m)
| order by TimeGenerated desc
```
(classic: `exceptions`, `problemId`, `timestamp`)
5. Compare message body version/schema with consumer's expected contract.

**Root causes (ranked)**
1. Poison/invalid message that always throws -> exceeds max delivery count.
2. Transient downstream outage shorter than retry budget (delivery count 10 burnt in seconds).
3. Message TTL expiry with dead-lettering enabled.
4. Contract/schema change after deploy.
5. Lock lost loops (Incident 6).

**Fix:** fix code or data; resubmit corrected messages from DLQ (clone to main queue with new message; set resubmit marker); consider longer backoff via scheduled re-enqueue.

**Prevention:** retry with backoff before abandon; explicit dead-lettering with reason for validation failures; DLQ alert and runbook/replay tool; schema versioning.

**STAR sketch:** *S:* DLQ grew 5k after deploy. *T:* recover without loss. *A:* grouped by reason (deserialization), rolled back contract change, built replay tool. *R:* all messages reprocessed, added contract tests.

**Follow-up ladder**
- *Follow-up:* Does the DLQ have its own DLQ? — No; messages in DLQ aren't auto-moved; transfer DLQ is separate.
- *Hard:* DLQ messages disappeared? — DLQ messages don't expire by TTL; check for a consumer draining them or deletion.
- *Tricky:* Max delivery count hit with no exception? — Lock expiry or abandon; each lock expiry counts as a delivery.

## Incident 9: Duplicate processing
**Symptom:** Same order charged twice, duplicate rows/emails; same `MessageId` handled more than once.

**First questions to ask**
- Same MessageId or different (producer duplicates)? DeliveryCount > 1?
- Is duplicate detection enabled? Are there multiple subscriptions/consumers?
- Retries on HTTP/outbox publishing?

**Diagnosis steps**
1. Query logs by MessageId:
```kql
AppTraces
| where TimeGenerated > ago(24h) and Message has "<messageId>"
| project TimeGenerated, OperationId, Message, AppRoleName
```
2. Check `DeliveryCount` in logs; if >1 -> redelivery (lock lost, exception after side effect, abandon).
3. `az servicebus queue show --query requiresDuplicateDetection,duplicateDetectionHistoryTimeWindow`.
4. Same payload, different MessageIds -> producer retried and sent twice (client timeout after send succeeded).
5. Check for two function apps/slots (staging slot also consuming the same queue!).

**Root causes (ranked)**
1. At-least-once delivery + non-idempotent handler (lock lost, crash after side effect before Complete).
2. Producer retries without stable MessageId.
3. Staging/other environment consuming the same entity.
4. Duplicate detection window expired or disabled.
5. Fan-out via multiple subscriptions intended-or-not.

**Fix:** idempotency key (MessageId/business key) in a store with unique constraint; inbox/outbox pattern; set deterministic `MessageId` and enable duplicate detection; separate staging entities.

**Prevention:** design for at-least-once; idempotent DB writes (upsert / `INSERT ... WHERE NOT EXISTS`); transactional outbox.

**STAR sketch:** *S:* double emails to customers. *T:* stop duplicates permanently. *A:* traced DeliveryCount=2 after timeouts; added idempotency table keyed by MessageId, raised lock renewal. *R:* duplicates eliminated.

**Follow-up ladder**
- *Follow-up:* Exactly-once in Service Bus? — Not end-to-end; duplicate detection dedupes sends within a window; consumers still need idempotency.
- *Hard:* Duplicate-detection window sizing? — Cover your max producer retry horizon; larger windows cost storage/memory; verify limits (Premium up to 7 days).
- *Tricky:* Transactions? — Complete + send within a transaction scope on same connection gives atomic settle+send, but not external DB side effects.

## Incident 10: Message ordering broken
**Symptom:** Updates applied out of order (e.g., "shipped" before "created"); stale state overwrites newer.

**First questions to ask**
- Is ordering required per entity key? Sessions enabled? Single or multiple consumers/concurrency?
- Partitioned entity? Retries/abandons?

**Diagnosis steps**
1. `az servicebus queue show --query "{sess:requiresSession,part:enablePartitioning}"`.
2. Check the `SessionId` set by producer and `SequenceNumber`/`EnqueuedSequenceNumber` vs processing order in logs.
3. Check `MaxConcurrentCalls` / Function `maxConcurrentCalls` (>1 on non-session = parallel, unordered).
4. Look for retries: message A abandoned and redelivered after B processed.
5. Check multiple publishers with different clocks and no ordering key.

**Root causes (ranked)**
1. No sessions with concurrent consumers (competing consumers destroy order).
2. Retry/abandon/redelivery re-queues earlier message later.
3. Producer sends via multiple paths/instances without a sequence.
4. Sessions used but different SessionId per related message.
5. Dead-letter/resubmit replays old message later.

**Fix:** use sessions keyed by entity ID (FIFO per session), `ServiceBusSessionProcessor`; or version numbers/ETags in the payload and discard stale updates (optimistic concurrency).

**Prevention:** always include entity version; design handlers commutative/idempotent; avoid mixing replays with live traffic.

**STAR sketch:** *S:* order status flapping. *T:* guarantee per-order ordering. *A:* introduced SessionId=OrderId and session-enabled queue plus version check. *R:* ordering consistent; throughput scaled by sessions.

**Follow-up ladder**
- *Follow-up:* Does a regular queue guarantee FIFO? — Only with a single receiver and no abandon/retry; not a safe assumption under concurrency.
- *Hard:* Throughput limit with sessions? — Parallelism = number of active sessions; one hot session is serialized.
- *Tricky:* Can you enable sessions on an existing queue? — No, must be set at creation; need a new entity and migration.

## Incident 11: Service Bus quota / throttling
**Symptom:** `ServiceBusException: The request was terminated because the entity is being throttled. Error code: 50009` / `ServerBusyException`; `QuotaExceededException` / `The maximum entity size has been reached or exceeded` (`MessagingEntityFull`); 429 from REST.

**First questions to ask**
- Tier (Basic/Standard/Premium) and messaging units? Which metric limit hit?
- Send burst or receive polling? Too many connections?

**Diagnosis steps**
1. Metrics: `ThrottledRequests`, `Size`, `CPU` and `Memory` (Premium namespace %), `ActiveConnections`, `ServerErrors`, `UserErrors` split by `MessagingErrorSubCode`.
2. `az servicebus namespace show -g <rg> -n <sbns> --query "{sku:sku,zone:zoneRedundant}"`.
3. `az servicebus queue show ... --query "{size:sizeInBytes,max:maxSizeInMegabytes}"`.
4. KQL on client side:
```kql
AppExceptions
| where TimeGenerated > ago(6h) and (OuterMessage has "throttl" or OuterMessage has "50009" or OuterMessage has "quota")
| summarize count() by bin(TimeGenerated,5m), AppRoleName
```
5. Check number of senders/receivers and whether each message creates a new `ServiceBusClient` (connection churn).

**Root causes (ranked)**
1. Standard tier shared-capacity throttling during bursts (no guarantee) vs Premium CU saturation.
2. Entity full because no consumer (Incident 7) -> `MessagingEntityFull`.
3. Client creating clients per call; excess connections.
4. Large batches/messages and tiny messages individually sent.
5. Namespace-level limits (verify in current docs).

**Fix:** use batching (`CreateMessageBatchAsync`), singleton client; rely on SDK retry with exponential backoff; scale Premium MUs (autoscale), partition load across namespaces; drain backlog.

**Prevention:** alerts on Throttled requests and namespace CPU; load tests; capacity planning; quotas documented.

**STAR sketch:** *S:* batch import caused 50009 errors. *T:* absorb bursts. *A:* batched sends, singleton client, moved to Premium with autoscale. *R:* no throttling at 5x load.

**Follow-up ladder**
- *Follow-up:* Why does Premium throttle less? — Dedicated resources (messaging units) rather than shared multi-tenant capacity.
- *Hard:* SDK retry vs your own retry? — SDK `ServiceBusRetryOptions` handles transient; don't wrap with unbounded retries; honor backoff.
- *Tricky:* Throttled while CPU is low? — Could be per-entity or connection limits, or sender-side bandwidth; check sub-code.

## Incident 12: Service Bus Function trigger scaling overwhelms downstream
**Symptom:** Backlog spike -> Function scales out -> downstream API/SQL returns 429/timeouts, connection pool exhausted, `Too many concurrent calls`/`ThreadPool starvation`; DB DTU/CPU 100%.

**First questions to ask**
- Plan (Consumption/Premium/Flex/Dedicated)? Instances during spike? `maxConcurrentCalls`?
- Downstream capacity limit? Is `WEBSITE_MAX_DYNAMIC_APPLICATION_SCALE_OUT` set?

**Diagnosis steps**
1. `host.json` -> `extensions.serviceBus.maxConcurrentCalls` (v5 ext; `maxConcurrentCalls` default often 16 per instance; verify) and `prefetchCount`.
2. Instance count:
```kql
AppRequests
| where TimeGenerated > ago(2h)
| summarize instances=dcount(CloudRoleInstance), reqs=count(), fail=countif(Success==false) by bin(TimeGenerated,1m)
```
(classic: `cloud_RoleInstance`)
3. Dependency failures: `AppDependencies | summarize count() by Target, ResultCode, bin(TimeGenerated,1m)`.
4. Total concurrency = instances × maxConcurrentCalls.

**Root causes (ranked)**
1. Unbounded scale-out × per-instance concurrency exceeds downstream capacity.
2. Missing downstream throttling/circuit breaker/backoff.
3. Prefetch too large -> locks held while queued -> lock lost.
4. `HttpClient`/connection pool misuse (new per call).

**Fix:** cap scale-out (`functionAppScaleLimit` / `WEBSITE_MAX_DYNAMIC_APPLICATION_SCALE_OUT`; Premium maximum burst); lower `maxConcurrentCalls`; Polly circuit breaker/bulkhead; for downstream limits use a semaphore or queue-based rate limiter; singleton clients.

**Prevention:** load test at backlog-drain rates; define downstream max RPS and derive limits; alerts on dependency 429.

**STAR sketch:** *S:* nightly backlog drain took SQL down. *T:* drain fast but safely. *A:* capped scale to 5 instances, concurrency 8, added retry/backoff. *R:* drained within SLA with DB at 60%.

**Follow-up ladder**
- *Follow-up:* Where do you set max scale on Consumption? — `functionAppScaleLimit` site config / `WEBSITE_MAX_DYNAMIC_APPLICATION_SCALE_OUT`.
- *Hard:* Does lowering concurrency increase lock-lost risk? — Less queue wait, so less; but check prefetch.
- *Tricky:* Per-queue vs per-app concurrency? — Concurrency is per function/trigger per instance; multiple functions add up.

## Incident 13: Azure Function not triggering after deployment
**Symptom:** Deployment "succeeds" but functions list is empty in portal or events not processed; portal says "No functions found" / "Function host is not running"; logs: `0 functions loaded`.

**First questions to ask**
- Runtime model (in-process vs isolated), `FUNCTIONS_WORKER_RUNTIME`, `FUNCTIONS_EXTENSION_VERSION`?
- Deployment method (zip/run-from-package, container)? Slots?
- Was a setting changed (e.g., `AzureWebJobs.<fn>.Disabled`, connection names)?

**Diagnosis steps**
1. `az functionapp function list -g <rg> -n <func> -o table`.
2. `az functionapp config appsettings list -g <rg> -n <func>` — check `FUNCTIONS_WORKER_RUNTIME`, `WEBSITE_RUN_FROM_PACKAGE`, `AzureWebJobsStorage`, connection settings, `AzureWebJobs.*.Disabled`.
3. Log stream / Diagnose and solve problems -> "Function App Down or Reporting Errors".
4. Startup errors:
```kql
AppTraces
| where TimeGenerated > ago(1h) and (Message has "Host" or Message has "listener" or Message has "Error indexing")
| order by TimeGenerated desc
```
5. Kudu (`https://<func>.scm.azurewebsites.net`) -> Debug console -> `site\wwwroot` contents (`host.json`, binaries, `functions.metadata`); for run-from-package check `/home/data/SitePackages`.
6. Trigger config: connection string/identity names exist; queue/entity exists; for slots the **staging slot setting names** vs sticky.

**Root causes (ranked)**
1. Missing/wrong app setting (connection name, `FUNCTIONS_WORKER_RUNTIME`) — indexing failure.
2. Runtime/extension version mismatch (e.g., isolated model deployed without worker settings; extension bundle incompatibility).
3. Function disabled via setting or slot-sticky setting.
4. Package didn't deploy (run-from-package URL SAS expired, zip structure wrong, build output missing).
5. Trigger identity lacks permission (listener fails to start).
6. Consumption plan: trigger sync didn't occur (needs "sync triggers"; for ARM deployments call syncfunctiontriggers).

**Fix:** correct settings; redeploy; `az functionapp restart`; sync triggers (`az rest --method post --url https://management.azure.com/<siteId>/syncfunctiontriggers?api-version=2022-03-01`, verify API version).

**Prevention:** post-deploy smoke test that checks function list + a synthetic trigger; infra-as-code for settings; deployment slots with warmup validation.

**STAR sketch:** *S:* release shipped, queue stopped draining. *T:* restore. *A:* trace showed indexing error on missing `ServiceBusConnection__fullyQualifiedNamespace` in prod; restored setting. *R:* resumed in 10 min; added config validation to the pipeline.

**Follow-up ladder**
- *Follow-up:* In-process vs isolated difference in troubleshooting? — Isolated has separate worker process; host and worker logs, `FUNCTIONS_WORKER_RUNTIME=dotnet-isolated`.
- *Hard:* Works in staging slot, fails after swap? — Slot-sticky settings/identities differ; swap applies production settings.
- *Tricky:* Functions list is fine but trigger silent? — Listener failed (permissions/firewall), or `Disabled` setting; check host logs for listener errors.

## Incident 14: Timer function running twice
**Symptom:** Scheduled job executes two times at the same minute; duplicate reports/emails.

**First questions to ask**
- Multiple instances / slots? Same storage account shared by prod and staging slots or other apps with same `AzureWebJobsStorage` and host ID?
- `RunOnStartup` set? Any manual swaps/restarts?

**Diagnosis steps**
1. Query invocations:
```kql
AppRequests
| where TimeGenerated > ago(24h) and Name == "<TimerFunctionName>"
| project TimeGenerated, OperationId, CloudRoleInstance, Success, DurationMs
| order by TimeGenerated desc
```
(classic: `requests`, `cloud_RoleInstance`)
2. Different `CloudRoleInstance` values at the same time -> singleton lock failing; same instance twice -> `RunOnStartup` or schedule overlap.
3. Check storage account `azure-webjobs-hosts` container -> `locks` blobs for the timer singleton; `host.json` `hostId`/`id`.
4. Check whether staging slot and prod slot share storage and have the same function app name/host ID (slots normally are distinct; check custom `AzureFunctionsWebHost__hostid`).
5. Check `[TimerTrigger("...", RunOnStartup = true)]` and `IsPastDue` handling.

**Root causes (ranked)**
1. Two apps/environments share the same host ID/storage so singleton lock isn't separate (or the opposite, a swap leaves both running during overlap).
2. `RunOnStartup=true` + restarts/scale events.
3. Non-singleton timer on multiple instances with lock blob contention/outage (storage issue).
4. Catch-up after downtime (`IsPastDue`, schedule monitor).
5. Two separate functions with the same schedule.

**Fix:** remove `RunOnStartup`; make job idempotent (execution key = schedule time); separate storage per environment; set `useMonitor`/`UseMonitor=false` if catch-up unwanted; distributed lock (blob lease) for critical work.

**Prevention:** idempotent job keyed by scheduled time; one storage account per app/env; avoid sharing host IDs.

**STAR sketch:** *S:* invoices generated twice monthly. *T:* fix root cause. *A:* saw two instances starting at same second, found staging sharing prod's storage and same hostId after cloning settings. *R:* separated storage, added idempotency by billing-period key.

**Follow-up ladder**
- *Follow-up:* How does a timer stay single across instances? — Blob lease-based singleton lock in `AzureWebJobsStorage`.
- *Hard:* Slot swap overlapping? — During swap both may fire briefly; idempotency covers it.
- *Tricky:* UTC vs local time? — `WEBSITE_TIME_ZONE`, DST double/skip; use UTC.

## Incident 15: Function cold start latency
**Symptom:** Sporadic 5–30s first request latency after idle; HTTP clients time out (`504` from gateway at ~230s limit is separate); p99 spikes after deploy/scale-out.

**First questions to ask**
- Hosting plan? Always Ready/prewarmed instances? Isolated vs in-process? Package size? DI startup work?
- Region/VNet integration? Cold only on first call after idle?

**Diagnosis steps**
1. Identify cold instances: first request per `CloudRoleInstance`:
```kql
AppRequests
| where TimeGenerated > ago(24h)
| summarize first=min(TimeGenerated), firstDuration=arg_min(TimeGenerated, DurationMs) by CloudRoleInstance
| order by first desc
```
2. Compare `AppTraces` host start messages ("Host started", "Initializing Warmup") with first request.
3. Check `WEBSITE_RUN_FROM_PACKAGE=1`, package size, startup code (Key Vault calls, EF migrations, heavy DI).
4. Plan: Consumption (cold likely), Premium (prewarmed), Flex (always-ready instances configurable), Dedicated (Always On).
5. Dependencies slow at startup: look at `AppDependencies` with `OperationId` of first requests.

**Root causes (ranked)**
1. Consumption scale-to-zero / new instance.
2. Heavy startup initialization (sync Key Vault, large assemblies, JIT, EF model building).
3. Deployment restarts / slot swap without warmup.
4. VNet integration or storage mount slowness.
5. Dedicated plan with Always On disabled.

**Fix:** Premium/Flex always-ready instances; Always On for dedicated; lazy init + async; ReadyToRun compilation; trim dependencies; warmup trigger; keep package small; timer ping only as last resort.

**Prevention:** measure p95 incl. cold; SLO by plan; load test scale-out.

**STAR sketch:** *S:* user-facing API on Consumption had 15s first hits. *T:* reduce to <2s. *A:* moved to Premium with 1 always-ready instance, ReadyToRun, moved KV load to cached singleton. *R:* p99 from 14s to 1.2s.

**Follow-up ladder**
- *Follow-up:* Premium vs Flex Consumption? — Both reduce cold start; Flex offers per-function scale and always-ready config; verify current features.
- *Hard:* Still slow on Premium on scale-out? — New instances beyond pre-warmed still cold; set prewarmed count and burst limits.
- *Tricky:* Timer pinging to "keep warm" on Consumption? — Unreliable; no guarantee of instance retention.

## Incident 16: "Host not running" / AzureWebJobsStorage connectivity
**Symptom:** Portal banner "Your app is currently not in a running state" / "Function host is not running"; logs: `Microsoft.Azure.WebJobs.Script: Error creating a Blob container reference... ` , `Unable to access storage account`, `The remote server returned an error: (403) Forbidden`, `Host initialization failed`, `Storage account connection string 'AzureWebJobsStorage' does not exist or is invalid`.

**First questions to ask**
- Was the storage account changed (key rotated, firewall enabled, shared-key disabled, deleted, region)?
- Identity-based `AzureWebJobsStorage__accountName`? VNet integration + private endpoints (blob, queue, table, file)?

**Diagnosis steps**
1. `az functionapp config appsettings list -g <rg> -n <func> --query "[?starts_with(name,'AzureWebJobsStorage')||starts_with(name,'WEBSITE_CONTENT')]"`.
2. `az storage account show -n <sa> --query "{fw:networkRuleSet,key:allowSharedKeyAccess,pe:privateEndpointConnections}"`.
3. Kudu console: `nameresolver <sa>.blob.core.windows.net`, `tcpping <sa>.blob.core.windows.net:443`.
4. Storage logs:
```kql
StorageBlobLogs | where TimeGenerated > ago(1h) and StatusCode >= 400
| summarize count() by StatusText, AuthenticationType, OperationName
```
5. Diagnose and solve problems -> "Function App Down or Reporting Errors"; check `WEBSITE_CONTENTAZUREFILECONNECTIONSTRING`/`WEBSITE_CONTENTSHARE` (Consumption/Premium file share; requires file private endpoint, and key-based auth).
6. Roles for identity-based host storage: `Storage Blob Data Owner`, `Storage Queue Data Contributor`, `Storage Table Data Contributor` (verify current docs).

**Root causes (ranked)**
1. Storage key rotated / connection string stale.
2. Storage firewall/private endpoint blocks the function (missing file/queue/table endpoints, DNS).
3. `allowSharedKeyAccess=false` while app uses keys (content share needs key).
4. Storage account deleted/region outage/throttling.
5. Missing roles for identity-based host storage.

**Fix:** update setting (prefer Key Vault reference / identity); allow VNet subnet; create private endpoints for blob/queue/table/file with private DNS zones; restore roles; restart.

**Prevention:** dedicated storage per app; no key rotation without updating settings; alerts on storage 4xx from the app; IaC for networking.

**STAR sketch:** *S:* all functions down after security hardening. *T:* restore quickly without weakening. *A:* identified firewall on host storage; added subnet rule + queue/table private endpoints. *R:* host up; documented required endpoints.

**Follow-up ladder**
- *Follow-up:* What does the host use storage for? — Triggers' state (blob receipts, singleton locks, timer status), keys, Durable state, content share.
- *Hard:* Can host storage use managed identity? — Yes for AzureWebJobsStorage (`__accountName`); content share still needs key/connection on some plans; verify.
- *Tricky:* Everything up but timers stop? — Lock/lease writes failing; check `azure-webjobs-hosts`.

## Incident 17: Durable Functions orchestration stuck or non-deterministic replay
**Symptom:** Instances remain `Running`/`Pending` for hours; error `Non-Deterministic workflow detected: A previous execution of this orchestration scheduled an activity task with sequence ID 3 and name 'X' ... but the current replay execution scheduled 'Y'` (`NonDeterministicOrchestrationException`).

**First questions to ask**
- Code changed while instances in flight? Which backend (Azure Storage, Netherite, MSSQL)?
- Any awaiting of external events/timers? Sub-orchestrators?

**Diagnosis steps**
1. Status: `GET /runtime/webhooks/durabletask/instances/<id>?code=<system key>` or Core Tools/`DurableTaskClient`.
2. Query orchestration logs (Durable emits to `AppTraces` customDimensions `prop__...`; verify):
```kql
AppTraces
| where TimeGenerated > ago(24h) and Message has "<instanceId>"
| order by TimeGenerated asc
```
3. Inspect storage: History/Instances tables (`<hubname>History`, `<hubname>Instances`), control-queues; or Durable Functions Monitor.
4. Check for orchestrator violations: `DateTime.Now`, `Guid.NewGuid()`, I/O, `Task.Run`, `Task.Delay`, non-durable await, env var reads, changed logic order.
5. Check poison control-queue messages / host not running (Incident 16), `maxConcurrentOrchestratorFunctions`.

**Root causes (ranked)**
1. Breaking code change with in-flight instances (non-determinism).
2. Orchestrator uses non-deterministic APIs.
3. Waiting on external event never raised / timer without timeout.
4. Storage/host issues; same task hub name shared between apps/slots (messages stolen by wrong version!).
5. Activity failing forever without retry policy bound.

**Fix:** terminate and restart affected instances (`az rest`/Core Tools `func durable terminate-instance`); use `context.CurrentUtcDateTime`, `context.NewGuid()`; version orchestrations (new function name / `OrchestrationVersion` / side-by-side deployment); give each slot its own task hub name.

**Prevention:** deploy with drain strategy; versioning; analyzer (`Microsoft.Azure.Functions.Worker.Extensions.DurableTask.Analyzers`); timeouts via durable timers + `Task.WhenAny`.

**STAR sketch:** *S:* 300 stuck order workflows post-release. *T:* recover and prevent. *A:* read non-determinism exception; reordered activity calls was the cause; terminated and replayed from source with new version; separated task hub per slot. *R:* all completed; adopted versioning policy.

**Follow-up ladder**
- *Follow-up:* Why must orchestrators be deterministic? — History is replayed to rebuild state; replay must schedule the same actions.
- *Hard:* Safe deployment pattern? — Side-by-side versioning with distinct task hub or function names; wait for old instances to drain.
- *Tricky:* Can you await `Task.Delay` in an orchestrator? — No; use `context.CreateTimer`.

## Incident 18: Function hitting timeout
**Symptom:** `Microsoft.Azure.WebJobs.Host.FunctionTimeoutException: Timeout value of 00:05:00 was exceeded by function: X`; HTTP 504/`502` after ~230 s (Azure Load Balancer idle timeout for HTTP triggers regardless of functionTimeout).

**First questions to ask**
- Plan and configured `functionTimeout` (Consumption default 5 min, max 10; Premium/Dedicated default 30 min and unbounded possible; verify)? Trigger type?
- Which dependency is slow?

**Diagnosis steps**
1. `host.json` `functionTimeout`.
2. KQL:
```kql
AppRequests
| where TimeGenerated > ago(24h) and DurationMs > 240000
| summarize count(), avg(DurationMs) by Name
```
3. Waterfall by `OperationId` in AppDependencies to find the slow call:
```kql
AppDependencies | where OperationId == "<id>" | order by TimeGenerated asc | project TimeGenerated, Type, Target, Name, DurationMs, ResultCode, Success
```
4. Check SNAT (Incident 37), thread-pool starvation, missing `CancellationToken`/HttpClient timeouts, large data loops.

**Root causes (ranked)**
1. Slow/hung dependency without client timeout.
2. Work genuinely too long for the plan limit.
3. HTTP-trigger long work (230 s front-end limit).
4. Thread starvation / sync-over-async.
5. Retry loops inside the function.

**Fix:** set explicit dependency timeouts and cancellation; split the work (queue + Durable fan-out/fan-in); use 202 + status endpoint (async HTTP API pattern); move to Premium/dedicated or Container Apps jobs for long runs; adjust `functionTimeout` within limits.

**Prevention:** SLO budget per call; load tests; alert on duration p95.

**STAR sketch:** *S:* nightly export hit 10-min limit. *T:* make reliable. *A:* chunked into queue messages processed by parallel functions, aggregated with Durable. *R:* finished in 4 min, no timeouts.

**Follow-up ladder**
- *Follow-up:* Can an HTTP function run 10 min? — Client gets cut at ~230 s; use async pattern.
- *Hard:* What happens to the message when timeout hits? — Invocation fails, lock lost/abandoned -> retry -> eventual DLQ.
- *Tricky:* Timeout raised but still failing at 5 min? — Plan maximum or the trigger/gateway limit, or host.json in the wrong place/not deployed.

## Incident 19: Logic App trigger not firing
**Symptom:** No runs in Run history; Trigger history shows `Skipped`/`Failed`/nothing; source event happened.

**First questions to ask**
- Consumption or Standard? Is the workflow Enabled? Trigger type (polling, webhook, HTTP, Service Bus, recurrence)?
- Recent changes: connection, identity, firewall/IP restrictions, trigger conditions, concurrency?

**Diagnosis steps**
1. Portal: Logic App -> Overview -> Trigger history (status `Succeeded` with no runs = no new data / condition not met; `Failed` -> error).
2. `az logic workflow show -g <rg> -n <name> --query state` (Consumption; needs `logic` extension). Standard: workflow Overview -> status.
3. Diagnostic logs (Consumption: `AzureDiagnostics` where `ResourceProvider=="MICROSOFT.LOGIC"`, `Category=="WorkflowRuntime"`, `OperationName has "microsoft.logic/workflows/workflowTriggerCompleted"`; verify names for your diagnostic destination):
```kql
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.LOGIC" and OperationName == "microsoft.logic/workflows/workflowTriggerCompleted"
| project TimeGenerated, resource_workflowName_s, status_s, code_s, error_message_s
| order by TimeGenerated desc
```
Standard logs go to resource-specific tables when enabled (e.g., `LogicAppWorkflowRuntime`) — verify in current docs.
4. Check trigger conditions/`splitOn`, polling interval, "trigger fires only for new items since last poll" semantics.
5. Connection status in API connections blade (Standard: connections.json; managed identity roles).
6. Network: IP access restrictions on HTTP trigger; for Standard, storage/VNet access.

**Root causes (ranked)**
1. Workflow disabled (auto-disabled after prolonged failures or by a deployment).
2. Connection/auth failure on trigger (Incident 21).
3. Trigger condition false / polling sees no new data / wrong filter or folder.
4. Caller blocked by access control (IP restrictions, SAS `sig` regenerated after redeploy so callback URL changed).
5. Standard: backing storage/ASP down; app settings missing after deploy.

**Fix:** enable; repair connection; update callback URL (listCallbackUrl) in callers; correct conditions; resubmit missed events using replay (trigger history "Resubmit" for failed/skipped).

**Prevention:** alert on trigger failures; heartbeat workflow; manage callback URLs via APIM front door to avoid SAS churn.

**STAR sketch:** *S:* nightly file-intake workflow stopped. *T:* find why. *A:* trigger history showed Failed 401 on SFTP/Blob connector after secret change; rebound to managed identity. *R:* resumed, resubmitted missed items.

**Follow-up ladder**
- *Follow-up:* Succeeded trigger, no run? — Polling found nothing new or condition false.
- *Hard:* Callback URL changed after redeploy? — SAS signature depends on workflow keys/version; regenerate and distribute or front with APIM.
- *Tricky:* Recurrence trigger drift or missed? — Platform best-effort scheduling; set start time/timezone, use concurrency.

## Incident 20: Logic App 429 / throttling
**Symptom:** Action fails `429 TooManyRequests` / `Rate limit is exceeded. Try again in N seconds`; runs slow; connector (SharePoint, Outlook, Service Bus) throttles; "You have exceeded the throttle limit" from platform.

**First questions to ask**
- Which action/connector? Parallelism in for-each? Trigger concurrency?
- Limits documented for that connector (per connection/time window)?

**Diagnosis steps**
1. Run history -> failed action -> outputs show status 429 and `Retry-After`.
2. Metrics: `Action Throttled Events`, `Trigger Throttled Events`, `Runs Started`.
```kql
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.LOGIC" and status_s == "Failed" and code_s has "429" or error_message_s has "throttl"
| summarize count() by resource_actionName_s, bin(TimeGenerated,5m)
```
(column names for Consumption diagnostic logs; verify in your workspace.)
3. Inspect for-each concurrency setting (default 20, max 50; verify) and trigger concurrency/`maximumWaitingRuns`.
4. Check action retry policy (default 4 exponential retries).

**Root causes (ranked)**
1. High concurrency fan-out against one connection.
2. Shared connection/service account used by many workflows.
3. Backend (e.g., SQL, API) itself returning 429.
4. Burst after backlog/catch-up.

**Fix:** reduce for-each concurrency; add delays; spread over multiple connections; configure retry policy with exponential/`Retry-After`; batch operations; use native Standard built-in connectors (service provider) with higher limits; request quota if applicable.

**Prevention:** capacity model per connector; trigger concurrency limits; backoff.

**STAR sketch:** *S:* SharePoint updates throttled during bulk run. *T:* complete without failures. *A:* set concurrency 5, used batch API, retry with backoff. *R:* 99.9% success.

**Follow-up ladder**
- *Follow-up:* Where is throttling enforced? — Connector/backend per connection and the Logic Apps runtime.
- *Hard:* Consumption vs Standard limits? — Standard has configurable host/limits and built-in connectors; verify limits docs.
- *Tricky:* Retry policy makes it worse? — Aggressive retries amplify; use exponential with jitter and honor `Retry-After`.

## Incident 21: Logic App connection auth expired / managed identity connector failing
**Symptom:** Actions fail `401 Unauthorized` / `InvalidAuthenticationTokenInTenant` / `The connection is not authorized`, "Token has expired", `ManagedIdentityAuthFailed`, `AADSTS50173`/`AADSTS700082` (refresh token expired). Connection status "Error"/"Unauthenticated" in API Connections.

**First questions to ask**
- Connector type: OAuth user-based, service principal, or managed identity? Who owns the connection (user account)?
- Was the user's password changed/MFA/Conditional Access/ account disabled? Recent role changes for the identity?

**Diagnosis steps**
1. Portal: Resource group -> API Connection -> Edit API connection / Status; Logic App -> API connections.
2. `az resource show --ids <apiConnectionId> --query properties.statuses`.
3. For managed identity: Logic App -> Identity (enabled?), target IAM (role), connection `authentication.type=ManagedServiceIdentity` in the workflow JSON; Standard built-in connectors use `Authentication: Managed identity` with identity audience.
4. Sign-in logs (Entra): service principal / user sign-ins with failures:
```kql
AADServicePrincipalSignInLogs
| where TimeGenerated > ago(24h) and ResultType != "0"
| project TimeGenerated, ServicePrincipalName, ResultType, ResultDescription, IPAddress
```
(table exists when exported to Log Analytics; managed identity sign-ins are in `AADManagedIdentitySignInLogs`.)
5. Check Conditional Access policy blocking non-interactive refresh.

**Root causes (ranked)**
1. OAuth user-based connection token expired/revoked (password change, MFA/CA, 90-day inactivity, leaver).
2. Managed identity missing role/not enabled/wrong audience.
3. Service principal secret expired.
4. Tenant policy change.

**Fix:** re-authorize connection; switch to managed identity or service principal with certificate; grant data role; update secret and Key Vault.

**Prevention:** use managed identity wherever supported; avoid personal accounts; expiry monitoring; alert on connection errors.

**STAR sketch:** *S:* many workflows failed Monday morning after an employee left. *T:* restore and de-risk. *A:* found connections owned by a disabled user; re-created with managed identity and roles. *R:* no further expiry incidents.

**Follow-up ladder**
- *Follow-up:* Why avoid user-owned connections? — Lifecycle tied to person, MFA/CA, token expiry.
- *Hard:* MI works in designer test but fails at runtime? — Different identity (SAMI vs UAMI) or role propagation; verify which identity the connection uses.
- *Tricky:* Standard vs Consumption identity? — Standard app-level identity with per-connection settings; Consumption workflow-level.

## Incident 22: Logic App large payload failure
**Symptom:** `BadRequest ... The content size exceeds the limit` / `RequestEntityTooLarge` (413) / `InvalidTemplate` / `Message size exceeded` / action output "Content was truncated"; connector-specific message-size cap.

**First questions to ask**
- Payload size and source? Which action fails? Consumption or Standard?
- Chunking supported by connector?

**Diagnosis steps**
1. Run history -> failing action -> Inputs/Outputs sizes.
2. Check limits (message size ~100 MB for chunking-enabled actions; default ~100 MB per HTTP action in Consumption, 210 MB Standard stateless? verify in current docs — these differ by connector).
3. Look for string/array operations (`json()`, `union`, `join`) on large data; expression/variable size limits.
4. Look at diagnostic `AzureDiagnostics` for `code_s` values like `RequestEntityTooLarge`.

**Root causes (ranked)**
1. Payload exceeds connector/HTTP action limit and chunking not enabled.
2. In-memory processing of big arrays/strings in expressions.
3. Service Bus message > size limit (256 KB Standard; Premium up to 100 MB; verify).
4. Trigger input too large for stateful workflow storage.

**Fix:** Claim-check pattern — store payload in Blob, pass reference; enable chunking ("Allow chunking") on supporting actions; stream copy via Blob connector; split into pages; use Functions for heavy transforms.

**Prevention:** design contracts with size caps; reject early; telemetry on payload size.

**STAR sketch:** *S:* nightly file passing through Logic App failed on bigger files. *T:* handle 500 MB. *A:* switched to claim-check with Blob SAS/MI and Function-based processing. *R:* flow reliable regardless of size.

**Follow-up ladder**
- *Follow-up:* What's the claim-check pattern? — Store large body externally, send pointer message.
- *Hard:* Why does a large run slow all runs? — Memory/storage throttling in the host; per-run overhead.
- *Tricky:* Chunking works on one action but not another? — Only certain connectors/actions support it.

## Incident 23: Blob 403 AuthorizationFailure from network rules
**Symptom:** `This request is not authorized to perform this operation. Status: 403 (AuthorizationFailure)` — often right after enabling firewall/private endpoint; role assignments are correct.

**First questions to ask**
- Is "Public network access" selected networks / disabled? Caller IP / subnet?
- Is the caller in the same region? (Storage VNet service endpoints don't work cross-region as IP rules; private endpoints do.)

**Diagnosis steps**
1. `az storage account show -n <sa> --query "{pna:publicNetworkAccess,def:networkRuleSet.defaultAction,vnet:networkRuleSet.virtualNetworkRules,ip:networkRuleSet.ipRules,bypass:networkRuleSet.bypass}"`.
2. Logs:
```kql
StorageBlobLogs
| where TimeGenerated > ago(1h) and StatusText == "AuthorizationFailure"
| project TimeGenerated, CallerIpAddress, Uri, OperationName, AuthenticationType
```
`CallerIpAddress` showing a public IP while expecting private endpoint -> DNS issue (Incident 36); `AuthorizationPermissionMismatch` instead -> RBAC (Incident 5).
3. App Service: `az webapp show --query outboundIpAddresses`; check VNet integration and route-all (`WEBSITE_VNET_ROUTE_ALL`/vnetRouteAllEnabled).
4. Kudu `nameresolver <sa>.blob.core.windows.net`, `tcpping`.
5. App Service/Function in same region as storage -> IP rules don't apply to same-region Azure services; need VNet rule/PE.

**Root causes (ranked)**
1. Caller not in allowed VNet/IP; no VNet integration.
2. Private endpoint DNS not resolved (public IP used, blocked).
3. Cross-region/same-region IP rule quirk.
4. Resource instance rule/trusted services exception missing for service (e.g., Event Grid, Backup).
5. Corporate proxy/NAT changes caller IP.

**Fix:** add subnet rule + service endpoint, or private endpoint + DNS zone link (`privatelink.blob.core.windows.net`); add resource instance rule; set route-all; allow trusted services where valid.

**Prevention:** network IaC with DNS; connectivity tests in pipeline; Azure Policy for storage network defaults.

**STAR sketch:** *S:* uploads failed after security enabled firewall. *T:* fix keeping lockdown. *A:* read StorageBlobLogs caller IPs, saw public path; fixed private DNS link. *R:* traffic over private endpoint, 0 failures.

**Follow-up ladder**
- *Follow-up:* AuthorizationFailure vs AuthorizationPermissionMismatch? — Network/auth layer vs RBAC data role.
- *Hard:* Public access disabled and portal fails? — Your browser isn't on the private network.
- *Tricky:* Works from one function, not another in same VNet? — Different subnet not delegated/allowed or route-all/DNS differs.

## Incident 24: Blob 409 / 412 conflicts and lease problems
**Symptom:** `409 LeaseAlreadyPresent` / `There is already a lease present`; `412 ConditionNotMet` (ETag/If-Match/If-None-Match); `409 BlobAlreadyExists`; `412 LeaseIdMissing`/`There is currently a lease on the blob and no lease ID was specified`; `409 ContainerBeingDeleted`.

**First questions to ask**
- Optimistic concurrency (ETag) or leases? Multiple writers? Retries after timeouts?
- Soft delete/versioning/immutability involved?

**Diagnosis steps**
1. Show blob lease: `az storage blob show --account-name <sa> --container-name <c> --name <b> --auth-mode login --query properties.lease`.
2. Logs:
```kql
StorageBlobLogs
| where TimeGenerated > ago(6h) and StatusCode in (409,412)
| summarize count() by StatusText, OperationName, Uri
```
3. Inspect code: lease acquire without release in `finally`, lease duration (15-60 s or infinite), `IfNoneMatch="*"` create semantics.
4. Check immutable policy / legal hold blocking overwrite (typically 409 `BlobImmutableDueToPolicy`).

**Root causes (ranked)**
1. Two writers racing; ETag changed (412 is expected and correct).
2. Lease not released after crash; fixed-duration lease not expired.
3. Retry resent a request that succeeded first time (BlobAlreadyExists).
4. Immutability/soft-delete/versioning semantics.
5. Container being deleted/recreated quickly.

**Fix:** treat 412 as retry-with-reread; release lease in `finally`; use short lease and renewals; `break lease` (`az storage blob lease break`) when orphaned; stable blob names with `If-None-Match: *` for idempotent create.

**Prevention:** idempotent writes; unique names with correlation id; lease for leader election with renewal loop; avoid infinite leases.

**STAR sketch:** *S:* counter file updates failing under load. *T:* make concurrency safe. *A:* moved to ETag-conditional writes with retry-reread, then to table/SQL for counters. *R:* correctness, 0 lost updates.

**Follow-up ladder**
- *Follow-up:* ETag vs lease? — Optimistic vs pessimistic.
- *Hard:* 412 on a lease'd blob after your own write? — Lease ID missing on the operation.
- *Tricky:* Is blob lease usable as distributed lock? — Yes (used for singleton), but needs renewal and handling of clock/timeouts.

## Incident 25: Blob trigger missing files
**Symptom:** Files uploaded but function didn't fire (or very late); some blobs processed twice; "blob trigger delay minutes".

**First questions to ask**
- Which trigger source: polling (default) or Event Grid-based (`Source = BlobTriggerSource.EventGrid`)? Plan (Consumption polling can lag)? Container large (many blobs)?
- Container/path pattern correct? Storage firewall?

**Diagnosis steps**
1. Check trigger definition and path pattern (case-sensitive) and connection.
2. Check blob receipts: `azure-webjobs-hosts` -> `blobreceipts/<hostid>/<function>/...` (a receipt means processed) and poison queue `webjobs-blobtrigger-poison`.
3. Logs:
```kql
AppTraces | where TimeGenerated > ago(6h) and Message has "<blobname>"
```
4. Confirm blob has been written fully (Put Block List commit time) and storage events flow if Event Grid-based (Event Subscription Metrics -> delivery failures).
5. Identity/network (Incident 16/23).

**Root causes (ranked)**
1. Polling latency/log-scan limitations (Consumption); large containers -> slow scan.
2. Event Grid subscription missing/broken delivery (endpoint validation, filters).
3. Receipts present so blob skipped (rewrites with same ETag), or blob modified in place doesn't retrigger.
4. Wrong path pattern/container.
5. Function disabled / listener failing.

**Fix:** use Event Grid-based blob trigger (or Event Grid + Service Bus); delete stale receipts when reprocessing; fix pattern; scale to Premium; reduce container size via date partitions.

**Prevention:** use Event Grid; monitor missing-file reconciliation job; idempotency.

**STAR sketch:** *S:* 10-minute latency on ingest. *T:* near-real-time. *A:* switched to Event Grid-based trigger with filtered subscription. *R:* latency < 5 s.

**Follow-up ladder**
- *Follow-up:* How does polling detect new blobs? — Scans storage analytics logs/blob listing with receipts; not instantaneous.
- *Hard:* Reprocess a blob? — Remove receipt or change ETag; or re-send event.
- *Tricky:* Why double-processing? — Multiple apps same container with different host IDs or receipts lost.

## Incident 26: SAS token expired / leaked
**Symptom:** `403 AuthenticationFailed ... Server failed to authenticate the request ... Signature not valid` / `Signed expiry time [..] must be after signed start time` / `SAS token expired`; or storage activity from unknown IPs, unexpected egress/cost, security alert.

**First questions to ask**
- Service SAS, account SAS, or user-delegation SAS? Stored access policy? Expiry and start time? Clock skew?
- Who holds the token? Where was it logged/committed?

**Diagnosis steps**
1. Decode SAS parameters: `se` (expiry), `st`, `sp`, `sr`, `sig`, `sv`, `skoid` (present = user delegation).
2. Logs:
```kql
StorageBlobLogs
| where TimeGenerated > ago(24h) and AuthenticationType == "SAS"
| summarize reqs=count(), ips=dcount(CallerIpAddress), bytes=sum(ResponseBodySize) by Uri, bin(TimeGenerated,1h)
```
(`AuthenticationType` values include `AccountKey`, `SAS`, `OAuth`, `Anonymous`; verify.)
3. Leak response: if service SAS tied to a stored access policy -> delete/modify the policy; account SAS/ad-hoc -> rotate the account key (invalidates all SAS signed by it); user delegation SAS -> revoke user delegation keys (`az storage account revoke-delegation-keys`).
4. Check clock skew (start time in future); use `st` slightly in the past.

**Root causes (ranked)**
1. Expired token (cached in client/UI beyond `se`).
2. Clock skew/start time in future; wrong permissions `sp`/resource `sr`.
3. Key rotated, invalidating SAS.
4. Leak: SAS in URL logs/Git/emails; long expiry.
5. Network rule blocking (SAS bypasses RBAC, not firewall).

**Fix:** reissue short-lived user-delegation SAS from backend with MI; for leaks, revoke via policy/key/delegation-key revoke; investigate access logs.

**Prevention:** prefer user-delegation SAS, short TTL (minutes), least permission, IP restrict (`sip`), HTTPS only, stored access policies, disable shared key access, Defender for Storage, secret scanning.

**STAR sketch:** *S:* leaked download link found in a ticket. *T:* contain. *A:* identified account SAS via logs; rotated key; migrated to user-delegation SAS with 5-min TTL. *R:* no further exposure.

**Follow-up ladder**
- *Follow-up:* Can you revoke a single ad-hoc SAS? — Not directly; only by key rotation (or stored policy/delegation key revoke).
- *Hard:* User delegation SAS max lifetime? — Delegation key up to 7 days (verify).
- *Tricky:* SAS still works after you disabled shared key access? — Account/service SAS stop working (they use key); user delegation SAS still works.

## Incident 27: Redis timeouts under load
**Symptom:** `StackExchange.Redis.RedisTimeoutException: Timeout performing GET (5000ms), next: GET key, inst: 1, qu: 0, qs: 12, aw: False, rs: ReadAsync, ws: Idle, in: 0, serverEndpoint: ..., mc: 1/1/0, mgr: 10 of 10 available, clientName: ..., IOCP: (Busy=0,Free=1000,Min=4,Max=1000), WORKER: (Busy=50,Free=32717,Min=4,Max=32767)`.

**First questions to ask**
- Which tier/size? Server load / CPU / network at the time? Big keys, `KEYS *`, Lua, large values?
- Client thread pool settings; many sync-over-async calls? Single multiplexer?

**Diagnosis steps**
1. Read the exception: `qs` (queued), `in` (bytes waiting), `IOCP/WORKER Busy > Min` -> thread pool starvation. 
2. Azure Cache metrics: `Server Load`, `CPU`, `Used Memory`, `Connected Clients`, `Network Bandwidth`, `Operations Per Second`, `Cache Latency`; `az redis show -n <r> -g <rg> --query "{sku:sku,ver:redisVersion}"`.
3. Redis commands: `SLOWLOG GET 20`, `INFO commandstats`, `MEMORY USAGE <key>`, `--bigkeys` (use carefully on prod).
4. Client logs:
```kql
AppExceptions
| where TimeGenerated > ago(6h) and ExceptionType has "RedisTimeoutException"
| summarize count() by bin(TimeGenerated,5m), AppRoleName
```
5. Check for maintenance/patching events and Azure Cache "Diagnose and solve problems".

**Root causes (ranked)**
1. Client thread-pool starvation (sync-over-async, low min threads).
2. Large values/big keys or expensive commands blocking the single-threaded server.
3. Server CPU/network bandwidth limit for the tier (verify limits).
4. Too many connections / non-singleton multiplexer.
5. Patching/failover.

**Fix:** singleton `ConnectionMultiplexer`; async APIs; `ThreadPool.SetMinThreads`; shard/compress big values; avoid O(N) commands; scale up/out (cluster) ; tune `syncTimeout/asyncTimeout`, retry with backoff.

**Prevention:** load tests; alerts on Server Load >70%; cache-aside with fallback; avoid huge payloads.

**STAR sketch:** *S:* peak traffic timeouts. *T:* stabilize. *A:* exception telemetry showed WORKER busy > min and 1 MB values; set min threads, compressed payloads, scaled P-tier. *R:* timeouts gone, p99 -40%.

**Follow-up ladder**
- *Follow-up:* What does `qs` mean? — Requests sent, waiting for response.
- *Hard:* CPU low but timeouts? — Client-side starvation or network; check IOCP/worker.
- *Tricky:* Timeout only during patch windows? — Failover reconnect; need retry policy and `abortConnect=false`.

## Incident 28: Redis connection storm after failover
**Symptom:** After patch/failover, app instances hammer Redis: `No connection is available to service this operation`, `UnableToConnect`, `Connected Clients` spike, server load high, cascade failures even after Redis healthy.

**First questions to ask**
- Is the multiplexer a singleton and resilient (`AbortOnConnectFail=false`)? Does code recreate multiplexer on errors?
- Retry policy (exponential/jitter)? Number of app instances?

**Diagnosis steps**
1. Metrics: `Connected Clients`, `Total Operations`, `Errors` (Failover), `Server Load` timeline vs patch time.
2. Check code for `Connect()` per request or aggressive reconnect loop.
3. KQL:
```kql
AppExceptions | where TimeGenerated > ago(2h) and ExceptionType has "RedisConnectionException"
| summarize count() by bin(TimeGenerated,1m)
AppDependencies | where Type == "Redis" and Success == false | summarize count() by bin(TimeGenerated,1m)
```
(Dependency type name may appear as `Redis`; verify for your instrumentation.)
4. Check cache-miss thundering herd on the DB (SQL DTU spike after Redis failover).

**Root causes (ranked)**
1. Recreating multiplexers per failure or per request.
2. Reconnect without backoff/jitter.
3. Cache stampede: all clients hit DB when keys unavailable.
4. `AbortOnConnectFail=true` (default in older clients) causing startup failure.

**Fix:** singleton lazy multiplexer with `ConnectRetry`, `AbortOnConnectFail=false`, `ReconnectRetryPolicy=ExponentialRetry`; handle `ForceReconnect` with throttle (min interval); stampede protection (locking/single-flight, jittered TTL); circuit breaker to DB.

**Prevention:** failover drills; load test patch scenario; alert on client count spike.

**STAR sketch:** *S:* patch night took down checkout. *T:* resilience. *A:* replaced reconnect-per-call with a throttled ForceReconnect and single-flight cache loads. *R:* subsequent patch = no customer impact.

**Follow-up ladder**
- *Follow-up:* Why singleton multiplexer? — Connections expensive; it multiplexes.
- *Hard:* How to avoid stampede? — Single-flight locks, stale-while-revalidate, jitter, request coalescing.
- *Tricky:* Does failover drop connections? — Yes; clients must reconnect, ensure DNS re-resolution (don't pin IP).

## Incident 29: Redis stale / serialization-broken cache after deploy
**Symptom:** After release, exceptions deserializing cached objects (`JsonSerializationException`, `SerializationException`, `InvalidCastException`), or users see old data/wrong shapes; rollback doesn't fix.

**First questions to ask**
- Did DTO/class shape/serializer change (BinaryFormatter, JSON, MessagePack)? Is the cache shared between old/new versions (blue-green, slots)?
- TTL on keys? Key naming versioned?

**Diagnosis steps**
1. Exceptions:
```kql
AppExceptions | where TimeGenerated > ago(6h) and (ExceptionType has "Serializ" or OuterMessage has "Redis")
| summarize count() by ExceptionType, OuterMessage
```
2. `redis-cli GET <key>` (or `--scan --pattern`) to inspect stored payload; `TTL <key>`.
3. Compare versions deployed per slot and release time.
4. Check if both versions write same key with different shapes.

**Root causes (ranked)**
1. Non-versioned schema change; old entries incompatible.
2. Mixed versions (slot/canary) sharing keys.
3. Long TTL / no invalidation on deploy.
4. Serializer/type-name handling (`TypeNameHandling`) differences.
5. Time zone/culture formatting.

**Fix:** treat deserialization failure as cache miss (delete + reload); version key prefix (`v2:`); flush affected keyspace (targeted); tolerant readers.

**Prevention:** cache-key schema version in key; backward-compatible DTOs; separate cache per slot or version; tests with previous-version payloads.

**STAR sketch:** *S:* post-release 500s on product pages. *T:* fix w/o downtime. *A:* identified old-shape JSON; catch-and-evict logic, version prefix. *R:* error rate 0 in minutes; added contract test.

**Follow-up ladder**
- *Follow-up:* Why not FLUSHALL? — Causes stampede and affects other data; targeted/versioned keys safer.
- *Hard:* Canary with shared cache? — Namespace by version or ensure two-way compatibility.
- *Tricky:* BinaryFormatter? — Obsolete/unsafe; avoid; use JSON/MessagePack with explicit contracts.

## Incident 30: Redis memory full / evictions
**Symptom:** `OOM command not allowed when used memory > 'maxmemory'` (noeviction) or rising `Evicted Keys`, cache hit ratio drop, latency, high `Used Memory Percentage`.

**First questions to ask**
- Maxmemory policy? Keys without TTL? Growth after feature? Persistence enabled (reserves memory)? Fragmentation?

**Diagnosis steps**
1. Metrics: `Used Memory`, `Used Memory RSS`, `Evicted Keys`, `Cache Hits/Misses`, `Expired Keys`; `INFO memory`, `INFO keyspace`.
2. Config: `az redis show --query redisConfiguration` (`maxmemory-policy`, `maxmemory-reserved`, `maxfragmentationmemory-reserved`).
3. Find big/no-TTL keys: `SCAN` with `MEMORY USAGE`, `redis-cli --bigkeys`, `OBJECT FREQ/IDLETIME`.
4. Correlate with deploy of new caching code.

**Root causes (ranked)**
1. Keys without TTL / unbounded key growth (per-user/per-request keys).
2. Big values or large collections.
3. Undersized tier/reserved memory misconfigured; fragmentation.
4. Wrong eviction policy (`noeviction` for a cache; default volatile-lru only evicts keys with TTL).

**Fix:** add TTLs; `allkeys-lru` for pure cache; compress; scale up/out; increase reserved memory for fragmentation; delete runaway key patterns.

**Prevention:** TTL standard in cache wrapper; memory alert at 80%; key-cardinality review.

**STAR sketch:** *S:* sessions vanished and DB load spiked. *T:* find cause. *A:* evictions + missing TTLs on per-request keys; added TTLs, changed policy, scaled tier. *R:* hit ratio recovered 70% -> 95%.

**Follow-up ladder**
- *Follow-up:* Eviction policies? — noeviction, allkeys-lru/lfu/random, volatile-*; verify Azure defaults.
- *Hard:* Memory fine but evictions? — maxmemory excludes reserved/fragmentation; check percentages.
- *Tricky:* Used memory drops after scale but latency higher? — Failover/rewarm.

## Incident 31: Missing telemetry in App Insights
**Symptom:** No requests/traces in portal, gaps in data, dashboards empty after deploy, Live Metrics works but logs don't (or reverse).

**First questions to ask**
- Classic or workspace-based? Connection string vs instrumentation key (ikey retired for ingestion in favor of connection strings — verify)? Sampling? Daily cap?
- Network: AMPLS/private link, firewall? Ingestion delay?

**Diagnosis steps**
1. Check settings: `APPLICATIONINSIGHTS_CONNECTION_STRING` present/correct; Functions: `host.json` `logging.applicationInsights.samplingSettings`, log levels (`logLevel.default`).
2. Workspace: Usage and estimated costs -> Daily cap reached? `Usage` table:
```kql
Usage | where TimeGenerated > ago(7d) | where DataType startswith "App" | summarize GB=sum(Quantity)/1000 by DataType, bin(TimeGenerated,1d)
```
3. Heartbeat: `AppTraces | summarize max(TimeGenerated) by AppRoleName` to see last data per role.
4. Check ingestion sampling (`AppRequests | summarize count() by ItemCount` -> `ItemCount>1` indicates sampling).
5. Network: from Kudu `curl -v https://<region>.in.applicationinsights.azure.com/v2/track`; with AMPLS ensure ingestion access mode.
6. Check `TelemetryProcessor`/filters excluding; Application Insights "Diagnose" blade.

**Root causes (ranked)**
1. Wrong/missing connection string or setting lost on deploy/slot.
2. Daily cap reached / sampling too aggressive / log level filters.
3. Network (private link, firewall, DNS).
4. Ingestion delay or resource moved/deleted workspace.
5. Instrumentation not initialized (e.g., isolated worker missing `AddApplicationInsightsTelemetryWorkerService`).

**Fix:** correct connection string; adjust cap/sampling; AMPLS config; add worker service registration and `ConfigureFunctionsApplicationInsights`.

**Prevention:** deploy-time check; alert on "no data" (log alert on absence); canary traces.

**STAR sketch:** *S:* dashboards blank after migration to isolated model. *T:* restore. *A:* found missing App Insights worker registration and default Warning-level filter rule. *R:* telemetry back; added no-data alert.

**Follow-up ladder**
- *Follow-up:* Why is `ItemCount` useful? — Shows sampling; use `sum(ItemCount)` for true counts.
- *Hard:* Live Metrics shows but logs absent? — Live metrics bypasses sampling/ingestion; check cap/sampling/log level.
- *Tricky:* Data in workspace but not in AI blade? — Table-level RBAC or different resource/workspace.

## Incident 32: Double telemetry
**Symptom:** Every request/trace appears twice; costs double; duplicate exceptions.

**First questions to ask**
- Both auto-instrumentation (agent/extension) and SDK enabled? Both Function host logger and `AddApplicationInsightsTelemetry`? Two exporters (OpenTelemetry + classic)?

**Diagnosis steps**
1. Duplicate check:
```kql
AppRequests | where TimeGenerated > ago(1h)
| summarize n=count() by OperationId, Name, TimeGenerated
| where n > 1
```
Compare `SDKVersion` column (`sdkVersion` classic) values per duplicate: e.g., `azurefunctions:` vs `dotnetc:`/`dotnet8:` vs `otel`.
2. App settings `ApplicationInsightsAgent_EXTENSION_VERSION=~3` (App Service auto-instrumentation) while the SDK is in code.
3. Check logger registration duplicates (`AddApplicationInsightsTelemetry` twice; TelemetryClient from both).

**Root causes (ranked)**
1. Agent-based + SDK instrumentation both on.
2. Two exporters (OpenTelemetry Azure Monitor + AI SDK).
3. Functions isolated: host and worker each emit when mis-configured.
4. Two AI resources with same data (ILogger + Serilog sink).

**Fix:** pick one; disable agent (`ApplicationInsightsAgent_EXTENSION_VERSION=disabled`) or remove SDK; deduplicate sinks.

**Prevention:** instrumentation standard; check in CI for `SDKVersion` diversity.

**STAR sketch:** *S:* AI bill doubled. *T:* find cause. *A:* `SDKVersion` showed two stacks; removed Serilog AI sink. *R:* cost halved.

**Follow-up ladder**
- *Follow-up:* Which column identifies source SDK? — `SDKVersion` (classic `sdkVersion`).
- *Hard:* Isolated Functions pitfalls? — Need `ConfigureFunctionsApplicationInsights()` and log filter rules removed default Warning.
- *Tricky:* Duplicates only for exceptions? — Logged and re-thrown/tracked twice.

## Incident 33: High App Insights cost
**Symptom:** Log Analytics/App Insights charges spike; "Daily cap" hit; ingestion >> expectation.

**First questions to ask**
- Which table/role drives volume? Sampling configured? Debug logs enabled? Dependency tracking of high-frequency calls?

**Diagnosis steps**
1. Biggest data types:
```kql
Usage | where TimeGenerated > ago(30d) and IsBillable == true
| summarize GB=sum(Quantity)/1000 by DataType | order by GB desc
```
2. Drill: `AppTraces | where TimeGenerated > ago(1d) | summarize GB=sum(_BilledSize)/1e9 by AppRoleName, SeverityLevel`; same for `AppDependencies`, `AppRequests`.
3. Top messages: `AppTraces | summarize n=count(), mb=sum(_BilledSize)/1e6 by Message | top 20 by mb`.
4. Review sampling, log levels, `host.json` `excludedTypes`, daily cap, retention, Basic Logs/commitment tier options.

**Root causes (ranked)**
1. Verbose logging (Information/Debug) in production, esp. in loops.
2. No sampling; chatty dependencies (Redis/SQL per call).
3. Duplicate telemetry (Incident 32).
4. Health-probe/availability noise.
5. Long retention/too many custom metrics dimensions.

**Fix:** adaptive/fixed sampling (exclude Request, Exception from sampling in Functions `excludedTypes: "Request;Exception"`), raise log level, filter noisy dependencies, commitment tier, shorter retention, route noisy tables to Basic Logs.

**Prevention:** budgets/alerts on workspace ingestion; log level review in PRs; sampling standards.

**STAR sketch:** *S:* AI spend tripled. *T:* cut cost without losing signals. *A:* found one loop logging at Information; sampling + level fix. *R:* -70% cost; alerts retained.

**Follow-up ladder**
- *Follow-up:* `_BilledSize` ? — Billed bytes per record in Log Analytics.
- *Hard:* Sampling effect on metrics? — Use `itemCount` aware aggregation; pre-aggregated metrics unaffected.
- *Tricky:* Daily cap as cost control? — Stops ingestion (blind spots); use alerts and cap cautiously.

## Incident 34: Broken correlation across messaging
**Symptom:** End-to-end trace stops at Service Bus; consumer operation has a different `OperationId`; application map shows disconnected nodes.

**First questions to ask**
- SDK versions (Azure.Messaging.ServiceBus emits `Diagnostic-Id`/W3C `traceparent` properties)? Custom producers (REST)? Logic App in between?

**Diagnosis steps**
1. Check message application properties in Service Bus Explorer: `Diagnostic-Id` (format `00-<traceid>-<spanid>-01`).
2. Enable ActivitySource: `AppContext.SetSwitch("Azure.Experimental.EnableActivitySource", true)` (older SDKs; verify), ensure `Activity.DefaultIdFormat = W3C`.
3. Query:
```kql
union AppRequests, AppDependencies, AppTraces
| where OperationId == "<traceId>"
| project TimeGenerated, Type=itemType, Name, AppRoleName, ParentId, Id
| order by TimeGenerated asc
```
(classic: `operation_Id`, `operation_ParentId`, `itemType`)
4. Check if consumer link is `Link`-based (batch triggers link rather than parent).

**Root causes (ranked)**
1. Producer doesn't inject trace context (custom client/old SDK/REST).
2. Consumer doesn't extract (manual receiver not creating Activity).
3. Intermediary (Logic App, APIM, Event Grid) drops headers.
4. Mixed ID formats (hierarchical vs W3C).
5. Batch triggers produce links not parent-child.

**Fix:** set `Diagnostic-Id`/`traceparent` on `ServiceBusMessage.ApplicationProperties`; start Activity from properties in consumer; add business correlation ID property and log it everywhere.

**Prevention:** correlation ID standard; contract test verifying propagation; use OpenTelemetry.

**STAR sketch:** *S:* couldn't trace an order across 4 services. *T:* get E2E visibility. *A:* added traceparent + OrderId custom dimension, fixed consumer Activity. *R:* single trace per order.

**Follow-up ladder**
- *Follow-up:* operation_Id vs custom correlation? — Trace ID is technical; business ID for support searches.
- *Hard:* Batch consumer? — Use span links.
- *Tricky:* Why Logic App breaks it? — Logic Apps have own `x-ms-workflow-run-id` tracking; needs header forwarding.

## Incident 35: Alerts not firing / alert storm
**Symptom:** Outage without alerts; or hundreds of notifications for one fault.

**First questions to ask**
- Alert type (metric/log/activity log/Service Health)? Condition & evaluation frequency/lookback? Action group configured and not suppressed? Alert processing rules?

**Diagnosis steps**
1. Portal: Monitor -> Alerts -> Alert rules (enabled?), Alert processing rules (suppression), action group test.
2. `az monitor scheduled-query show`, `az monitor metrics alert show -n <n> -g <rg>`.
3. Log alerts: run the query in the window; check ingestion latency (log alerts evaluate on already ingested data; use lookback > latency).
4. Missing data handling: "no data" doesn't fire a threshold rule by default; use absence queries.
5. Fired alerts history:
```kql
AlertsManagementResources | where type == "microsoft.alertsmanagement/alerts" | project name, properties.essentials.severity, properties.essentials.startDateTime
```
(Azure Resource Graph query; table `alertsmanagementresources`.)
6. Action group: email rate limits, SMS limits, webhook failures (Activity Log / action group "Test").

**Root causes (ranked)**
1. Query/threshold/lookback mismatch (ingestion delay, wrong dimension split).
2. Alert suppressed/disabled/wrong scope/action group target.
3. Stateful alerts: remains Fired, no re-notify (auto-resolve).
4. Storm: per-instance/per-request dimensions, no aggregation, flapping threshold, noisy dependencies.
5. Rate limits on notifications.

**Fix:** correct rule; use aggregation (count over 5 min), dimensions carefully, `autoMitigate`, alert processing rules for grouping/suppression, action group re-check; use smart groups.

**Prevention:** SLO-based alerts, runbooks, alert-testing during game days, dead-man's-switch heartbeat alert.

**STAR sketch:** *S:* a silent 40-min outage. *T:* explain and fix. *A:* alert used 1-min lookback with 5-min ingestion latency; changed to 15 min and added absence alert. *R:* MTTD <5 min next incident.

**Follow-up ladder**
- *Follow-up:* Metric vs log alert latency? — Metric faster; log has ingestion delay.
- *Hard:* Alert storm control? — Aggregate, dimension limits, processing rules grouping, severity routing.
- *Tricky:* Stateless vs stateful? — Stateful fires once until resolved; stateless each evaluation.

## Incident 36: Private endpoint DNS resolves public IP
**Symptom:** After adding a private endpoint, app still connects over public IP and gets 403 (`AuthorizationFailure`, `ForbiddenByFirewall`, SQL `Client with IP address ... is not allowed to access the server`) or timeouts; or from some networks it works and others not.

**First questions to ask**
- Is the private DNS zone linked to the VNet the caller uses? Custom DNS servers / forwarders? On-prem resolving via conditional forwarder to 168.63.129.16?

**Diagnosis steps**
1. From the app (Kudu console): `nameresolver <sa>.blob.core.windows.net` — expect `10.x` via `privatelink` CNAME chain; public IP = wrong.
2. `az network private-endpoint show -g <rg> -n <pe> --query "{state:privateLinkServiceConnections[0].privateLinkServiceConnectionState.status,nic:networkInterfaces[0].id}"` (must be `Approved`).
3. `az network private-dns zone list -g <rg>` ; `az network private-dns link vnet list -g <rg> -z privatelink.blob.core.windows.net` (link to the correct VNet, A record present: `az network private-dns record-set a list`).
4. Check VNet DNS servers: `az network vnet show --query dhcpOptions`; custom DNS must forward to Azure DNS (168.63.129.16) or use DNS Private Resolver.
5. App Service: `WEBSITE_DNS_SERVER`, `WEBSITE_VNET_ROUTE_ALL`, VNet integration subnet.
6. Zone names: blob `privatelink.blob.core.windows.net`, vault `privatelink.vaultcore.azure.net`, SQL `privatelink.database.windows.net`, Service Bus `privatelink.servicebus.windows.net`, Redis `privatelink.redis.cache.windows.net` (verify per service in docs).

**Root causes (ranked)**
1. Private DNS zone not linked to caller VNet (or in hub only without link/forwarding).
2. Custom DNS not forwarding the privatelink zones.
3. Missing A record / PE approval pending.
4. App not VNet-integrated or route-all off.
5. Client caching DNS / hosts file override.

**Fix:** link zone; add forwarders/Private Resolver; fix record; enable VNet integration; flush DNS/restart app.

**Prevention:** DNS zone group on PE in IaC; central DNS policy; connectivity test (`nameresolver`) in post-deploy.

**STAR sketch:** *S:* storage worked from VM but not Function. *T:* resolve. *A:* `nameresolver` returned public IP; Function VNet lacked zone link. *R:* link added; resolved to private IP.

**Follow-up ladder**
- *Follow-up:* Why `privatelink` CNAME? — Public name CNAMEs to privatelink name, which private zone overrides.
- *Hard:* Hub-and-spoke DNS? — Link zones to hub or use resolver; spokes use hub DNS.
- *Tricky:* Disable public access, then on-prem fails? — On-prem needs conditional forwarder to Azure.

## Incident 37: SNAT port exhaustion
**Symptom:** Intermittent outbound failures: `An attempt was made to access a socket in a way forbidden...`, `SocketException: Only one usage of each socket address`, `HttpRequestException ... A connection attempt failed`, timeouts at load to the same external endpoint; App Service diagnostics detector "SNAT Port Exhaustion".

**First questions to ask**
- Many outbound calls to same destination IP:port? HttpClient created per request? Plan (App Service has ~128 preallocated per instance; verify)? NAT Gateway/VNet integration?

**Diagnosis steps**
1. Portal: App Service -> Diagnose and solve problems -> "SNAT Port Exhaustion"; metrics `SNAT Connection Count` (Azure Load Balancer / NAT GW).
2. Kudu `tcpping`, `netstat -an` (TCP TIME_WAIT counts).
3. KQL on failed dependencies:
```kql
AppDependencies | where TimeGenerated > ago(2h) and Success == false
| summarize count() by Target, ResultCode, bin(TimeGenerated,5m)
```
4. Code review: `new HttpClient()` per call, `using` disposal; missing keep-alive; no connection pooling.
5. Check private endpoints/service endpoints (avoid SNAT for Azure PaaS).

**Root causes (ranked)**
1. HttpClient anti-pattern / no connection reuse.
2. High fan-out to a single destination.
3. Long-lived idle connections timed out at 4 min.
4. Missing NAT Gateway for the VNet-integrated app.

**Fix:** `IHttpClientFactory`/static HttpClient with `SocketsHttpHandler` pooling, `PooledConnectionLifetime`; private endpoints/service endpoints; NAT Gateway (more ports); reduce parallelism; retry with backoff.

**Prevention:** connection-pooling standards; alerts on SNAT metrics; load tests.

**STAR sketch:** *S:* Function intermittent timeouts to partner API at scale. *T:* fix. *A:* detector confirmed SNAT; replaced per-call HttpClient with factory; added NAT gateway. *R:* timeouts vanished.

**Follow-up ladder**
- *Follow-up:* What is SNAT? — Source NAT maps private outbound flows to a limited set of public IP ports.
- *Hard:* Ports are per what? — Per destination IP:port tuple per public IP.
- *Tricky:* Private endpoint reduces SNAT? — Yes for PaaS traffic, since it doesn't traverse public SNAT.

## Incident 38: Azure SQL transient errors / failover
**Symptom:** Errors 40613 (`Database ... is not currently available`), 40197, 40501 (service busy), 49918/49919/49920, 4060, 10928/10929 (resource limits), `Timeout expired`, connection resets during maintenance/failover.

**First questions to ask**
- Tier (DTU/vCore, serverless auto-pause)? Retry policy configured? Frequency/time pattern (maintenance)? Connection resiliency?

**Diagnosis steps**
1. Resource health + Service Health; portal -> SQL DB -> Metrics: `cpu_percent`, `dtu_consumption_percent`, `workers_percent`, `sessions_percent`, `connection_failed`, `deadlock`.
2. Activity/logs: `AzureDiagnostics | where ResourceProvider=="MICROSOFT.SQL" and Category in ("Errors","Timeouts","Blocks","Deadlocks")` (or `SQLInsights`/resource-specific tables; verify).
3. Query Store / DMV: `SELECT * FROM sys.dm_db_resource_stats ORDER BY end_time DESC;` `sys.event_log` in master (`SELECT * FROM sys.event_log WHERE database_name='<db>'`).
4. Client: EF Core `EnableRetryOnFailure()`, `ConnectRetryCount/Interval` in connection string; app KQL:
```kql
AppExceptions | where TimeGenerated > ago(24h) and OuterMessage has_any ("40613","40197","transient","SqlException")
| summarize count() by bin(TimeGenerated,10m)
```
5. Serverless: auto-pause resume latency (first connection error ~60 s).

**Root causes (ranked)**
1. Normal platform reconfiguration/failover without retry logic.
2. Resource limits reached (workers/CPU) -> 10928/10929/40501.
3. Serverless auto-pause/resume.
4. Network/firewall/pooling issues; long transactions blocking.
5. Region incident.

**Fix:** retry with exponential backoff (SqlClient configurable retry/EF execution strategy), idempotent operations, scale up, query tuning, avoid long transactions, keep-alive for serverless or disable auto-pause.

**Prevention:** resilience tested via failover (`az sql db failover`), alerts on failed connections, connection pooling.

**STAR sketch:** *S:* brief errors every few weeks. *T:* eliminate customer impact. *A:* matched to maintenance events; enabled EF retry + idempotent writes. *R:* failovers invisible to users.

**Follow-up ladder**
- *Follow-up:* Which errors are transient? — 40613, 40197, 40501, 49918–49920, 10928/10929, 4060 etc. (verify list).
- *Hard:* Retry and non-idempotent writes? — Use idempotency keys/transactions; commit-unknown risk.
- *Tricky:* Test failover? — `Invoke-AzSqlDatabaseFailover` / `az sql db failover` (Premium/BC, Hyperscale variations).

## Incident 39: Slot swap caused outage
**Symptom:** After swap, 500s/503s, app can't reach dependencies, wrong config, connections refused; slot warm-up looked fine.

**First questions to ask**
- Which settings are slot-sticky? Does the staging slot use a different identity/connection? Did warm-up request hit the right path? Are timers/queue consumers running in staging?

**Diagnosis steps**
1. Compare settings: `az webapp config appsettings list -g <rg> -n <app> --slot staging` vs production; look at `slotSetting` flags.
2. Activity Log -> `Swap Web App Slots`; swap steps in Kudu/Diagnostics "Swap" detector.
3. After swap, new production (old staging code) uses production's sticky settings; managed identity differs per slot (role assignments missing for slot SAMI!).
4. KQL around swap time:
```kql
AppRequests | where TimeGenerated between (datetime(<t>)-15m .. datetime(<t>)+30m)
| summarize fail=countif(Success==false), n=count() by bin(TimeGenerated,1m), CloudRoleInstance
```
5. Check health check path configuration; `WEBSITE_SWAP_WARMUP_PING_PATH`/`WEBSITE_SWAP_WARMUP_PING_STATUSES`; Auto-swap.

**Root causes (ranked)**
1. Config drift/non-sticky vs sticky mismatches (connection strings, feature flags).
2. Slot SAMI missing roles (different principal from production slot).
3. Warm-up inadequate -> cold instances take traffic; no health check.
4. Background consumers (Service Bus/Timer) in staging slot competing/duplicating; DB migrations incompatible with rollback.
5. Networking: slot not VNet-integrated / access restrictions differ.

**Fix:** swap back (instant rollback); fix settings; grant roles to slot identity (or use UAMI shared); implement health-check and warmup; backward compatible DB migrations.

**Prevention:** swap with preview (multi-phase); consistent IaC for slots; smoke test staging pre-swap; disable non-prod triggers (`AzureWebJobs.X.Disabled` slot setting).

**STAR sketch:** *S:* swap broke prod auth. *T:* rollback and prevent. *A:* swapped back; found slot SAMI without Key Vault role; moved to UAMI and added swap-with-preview gating. *R:* clean swaps for months.

**Follow-up ladder**
- *Follow-up:* Slot-sticky settings? — Stay with the slot, not swapped.
- *Hard:* Rollback with DB changes? — Use expand/contract migrations.
- *Tricky:* Slots and Functions timers? — Both slots may run triggers; disable in staging.

## Incident 40: Pipeline deployment fails with role-assignment permission error
**Symptom:** Bicep/ARM/Terraform step fails: `AuthorizationFailed: The client '<spn>' ... does not have authorization to perform action 'Microsoft.Authorization/roleAssignments/write' over scope ...`; `PrincipalNotFound` (replication delay); `RoleAssignmentExists`; `InvalidPrincipalType`.

**First questions to ask**
- Which service connection/SPN? What role does it hold at which scope (Contributor can't write role assignments)? Is it creating assignments in the same template? Is it a new MI just created?

**Diagnosis steps**
1. `az role assignment list --assignee <spnObjectId> --all -o table`.
2. Activity Log for the failure: Monitor -> Activity log -> filter `Create role assignment`.
```kql
AzureActivity | where TimeGenerated > ago(1d) and OperationNameValue =~ "MICROSOFT.AUTHORIZATION/ROLEASSIGNMENTS/WRITE" and ActivityStatusValue == "Failure"
| project TimeGenerated, Caller, ResourceGroup, Properties
```
3. Check whether pipeline identity holds `User Access Administrator`, `Role Based Access Control Administrator` (with ABAC conditions limiting assignable roles), or `Owner`.
4. `PrincipalNotFound`: set `principalType` ("ServicePrincipal") to avoid directory replication lag.
5. Policy/management-group denies; PIM-eligible roles not activated.

**Root causes (ranked)**
1. Pipeline identity only Contributor (can't write role assignments).
2. Wrong scope (assignment target beyond granted scope).
3. Replication delay for new principal (set principalType).
4. Duplicate assignment names (non-deterministic GUID).
5. Conditional/PIM/policy blocks.

**Fix:** grant least-privileged `Role Based Access Control Administrator` constrained by condition (only specific roles) at the required scope; set `principalType`; deterministic `guid()` names; separate "identity & RBAC" stage run by a privileged identity.

**Prevention:** split pipelines (platform vs app); document pipeline permissions; avoid Owner.

**STAR sketch:** *S:* new service onboarding failed at RBAC step. *T:* unblock safely. *A:* gave pipeline SPN constrained RBAC Administrator limited to Data roles; set principalType. *R:* automated onboarding with least privilege.

**Follow-up ladder**
- *Follow-up:* Why can Contributor not assign roles? — `Microsoft.Authorization/*/write` is excluded from Contributor.
- *Hard:* Limiting what the pipeline can assign? — ABAC conditions on RBAC Administrator.
- *Tricky:* `PrincipalNotFound` intermittently? — Replication lag; set principalType or retry.

## Incident 41: Secret rotation broke the app
**Symptom:** After secret/certificate/key rotation: `401`, `AADSTS7000215: Invalid client secret provided`, `AuthenticationFailed`, SQL `Login failed for user`, Service Bus `UnauthorizedAccess`, storage `AuthenticationFailed (Server failed to authenticate the request)`; app still using old value.

**First questions to ask**
- Which secret rotated, who consumes it? Key Vault reference or copied value? Key Vault reference refresh cadence? Did app restart?

**Diagnosis steps**
1. Key Vault reference status: portal -> app -> Configuration -> shows green check/"Key vault Reference" error; `az webapp config appsettings list` shows `@Microsoft.KeyVault(...)`; Diagnose and solve problems -> "Key Vault Application Settings Diagnostics".
2. Versionless references refresh about daily (verify; in current docs) and aren't instant; use restart or event-driven refresh.
3. KV audit: `AuditEvent` for `SecretGet`/`SecretNew`/version change times.
4. Identify consumers via search for the secret name across apps/pipelines/Logic App connections/Function connection strings.
5. App Insights exceptions correlated with rotation time.

**Root causes (ranked)**
1. Consumer caches old value (app setting not refreshed, in-memory config).
2. Secret copied to multiple places; not all updated.
3. Both old/new not valid concurrently (no overlap window) — e.g., single key regenerated.
4. Key Vault reference to a pinned version.
5. Identity lacks access to new version/secret.

**Fix:** restart/refresh; update missing consumers; roll back to previous version for quick recovery; for dual-key resources (storage/Service Bus keys primary/secondary) rotate secondary first.

**Prevention:** move to managed identity (no secrets); versionless KV references + Event Grid `SecretNearExpiry`/`SecretNewVersionCreated` to trigger rotation/restart; dual-credential rotation; inventory of consumers.

**STAR sketch:** *S:* quarterly rotation caused outage. *T:* design safe rotation. *A:* implemented two-key rotation + Event Grid-triggered refresh; replaced SP secrets by MI where possible. *R:* rotation with zero downtime.

**Follow-up ladder**
- *Follow-up:* Best fix for secrets? — Eliminate them with managed identities.
- *Hard:* Versioned vs versionless KV references? — Versionless auto-updates (periodic), versioned needs explicit change.
- *Tricky:* App didn't pick up the new secret after refresh? — Process cached it; reload config or restart.

## Incident 42: Cost spike investigation
**Symptom:** Budget alert or invoice shows sudden increase; Cost Analysis shows a daily jump.

**First questions to ask**
- Which subscription/RG/service/meter? When did it start? Any deploys, scale changes, new environments, data egress?

**Diagnosis steps**
1. Portal: Cost Management -> Cost analysis -> Group by Service name / Resource / Meter, daily granularity, compare periods; use Anomaly alerts.
2. CLI: `az costmanagement query` (verify syntax) or export to storage; `az consumption usage list --start-date ... --end-date ... ` (verify).
3. Correlate with Activity Log (`AzureActivity` create/scale operations):
```kql
AzureActivity | where TimeGenerated > ago(14d) and OperationNameValue has_any ("WRITE","ACTION") and ActivityStatusValue == "Success"
| summarize count() by Caller, OperationNameValue | top 20 by count_
```
4. Check likely suspects: Log Analytics ingestion (`Usage`), Premium/Always-ready instances, Service Bus MU, Redis tier, storage transactions/egress, Functions Consumption executions loop (retry storm), Logic Apps Standard actions, orphan disks/public IPs, Defender plans.
5. Metrics: executions/GB-s, `Transactions`, `Egress`.

**Root causes (ranked)**
1. Log ingestion explosion.
2. Retry loop/poison messages causing runaway executions.
3. Scaled-up/out SKU left on, dev environments running.
4. Data egress/cross-region traffic or storage transactions.
5. New resources/Defender plans.

**Fix:** stop the loop; right-size/autoscale; sampling; delete orphans; budgets & alerts; reserved instances/savings plans for steady workloads.

**Prevention:** tagging, budgets per RG, anomaly alerts, cost reviews, IaC with SKU guardrails, Azure Policy.

**STAR sketch:** *S:* cost doubled mid-month. *T:* identify. *A:* Cost Analysis by meter pointed to Log Analytics; `_BilledSize` revealed a debug-level loop. *R:* fixed with sampling; savings $X/month.

**Follow-up ladder**
- *Follow-up:* First view? — Cost analysis grouped by meter daily.
- *Hard:* Attribute shared costs? — Tags, allocation rules, resource groups.
- *Tricky:* Cost spike but no resource change? — Usage-driven meters (egress, transactions, executions).

## Incident 43: Cross-region failover drill gone wrong
**Symptom:** During DR drill, secondary region serves errors: missing secrets/roles, DNS not switching, data stale/diverged, queues empty or duplicated, Redis cold, Front Door/Traffic Manager still routes to primary.

**First questions to ask**
- RTO/RPO targets? Active-passive or active-active? Which components have geo-replication (SQL failover group, storage RA-GRS/GZRS, Service Bus Geo-DR, Key Vault auto-replication)? Last drill date?

**Diagnosis steps**
1. Verify routing: Front Door/Traffic Manager endpoint health (`az network traffic-manager endpoint list`), DNS TTL, health probe path.
2. Data: SQL failover group role (`az sql failover-group show`), storage account failover status (`az storage account show --query "{lastSync:geoReplicationStats.lastSyncTime,status:statusOfSecondary}" --expand geoReplicationStats`), Service Bus Geo-DR pairing (`az servicebus georecovery-alias show`): metadata only — messages are not replicated in Geo-DR (verify; Premium Geo-Replication differs).
3. Secondary app: settings, identity roles, Key Vault in secondary region (vault replicated read-only; use paired vault), private endpoints/DNS zones in the secondary VNet, certificates, app registration redirect URIs.
4. KQL cross-region:
```kql
AppRequests | where TimeGenerated > ago(2h) | summarize n=count(), fail=countif(Success==false) by cloud_RoleName=AppRoleName, ClientCountryOrRegion, bin(TimeGenerated,5m)
```
5. Config differences via `az deployment` what-if between regions.

**Root causes (ranked)**
1. Secondary environment drift (settings, RBAC, DNS/private endpoints, secrets).
2. Data replication gaps/assumptions (async RPO, Service Bus messages not replicated, Redis not replicated unless geo-replication).
3. Connection strings pinned to regional endpoints rather than failover group/alias listener.
4. DNS TTL/health probe thresholds; client caching.
5. Runbook not exercised/unknown dependency order (failback).

**Fix:** repair drift via IaC; use alias/listener endpoints; rehearse sequence (data -> app -> traffic); document failback; accept RPO.

**Prevention:** regular game days, IaC parity, automated DR tests, dependency inventory, chaos testing, runbooks with owners.

**STAR sketch:** *S:* DR drill left secondary returning 500s. *T:* make DR credible. *A:* identified RBAC/DNS drift and Service Bus alias unused; codified secondary via IaC, switched to failover-group endpoints, scripted traffic cutover. *R:* next drill met 30-min RTO.

**Follow-up ladder**
- *Follow-up:* RTO vs RPO? — Time to recover vs max data loss.
- *Hard:* Service Bus Geo-DR behavior? — Pairs namespaces; replicates metadata, not messages (Geo-DR); in-flight messages need handling — verify current docs and Premium geo-replication.
- *Tricky:* Active-active data pitfalls? — Conflict resolution, ordering, idempotency, duplicate processing across regions.

---

# Appendix A: Azure triage checklist (one page)
Work top to bottom; stop when you find a change that explains the timeline.

1. **Scope & timeline:** What changed and when? Who/what is affected (all users, one region, one instance)? Compare first-failure time with deployments (Activity Log, pipeline), config changes, certificate/secret expiry.
2. **Identity:** Which principal is calling (SAMI/UAMI/SP/user)? Find `oid` in the error/resource logs. Role at right scope, plane (control vs data)? Propagation delay? `AZURE_CLIENT_ID` set? Secret/cert expired? Conditional Access? (`az role assignment list --assignee`, Entra sign-in logs.)
3. **Network:** Firewall/NSG/UDR, private endpoint approval, DNS (`nameresolver`), VNet integration/route-all, SNAT, TLS/cert, outbound IP vs allow-list. (`tcpping`, Kudu, Network Watcher connection troubleshoot.)
4. **Config:** App settings/slot-sticky, connection names, identity-based settings, Key Vault reference status, host.json, feature flags, runtime/extension versions, env var shadowing.
5. **Quota / throttle:** 429/50009/`ServerBusy`, subscription quotas, Service Bus/Redis/SQL limits, Logic App connector limits, Function scale limits, storage throttling (`ServerBusy`, 503), daily caps.
6. **Dependency health:** SQL/Redis/Storage/Service Bus metrics, p95 dependency duration (`AppDependencies`), downstream API status, retries & circuit breakers, poison messages/DLQ.
7. **Deployment change:** Slot swap, package contents, IaC what-if diff, schema/contract change, rollback readiness. Roll back first when impact is high; investigate after.
8. **Platform health:** Azure Service Health (portal -> Service Health, Resource health), status.azure.com, region incidents, planned maintenance; Diagnose and solve problems blade.
9. **Mitigate, then root-cause:** restart/scale/roll back/failover; communicate; capture evidence (queries, timelines) for post-incident review; add detection + prevention.

**Golden rules:** read the *exact* error text and code; find the *real* identity; distinguish `AuthorizationFailure` (network) / `AuthorizationPermissionMismatch` (RBAC data role) / 401 (authentication); assume at-least-once delivery; verify limits in current docs.

# Appendix B: KQL cheat sheet
Workspace-based names first (classic in parentheses).

```kql
// Failures by operation, last 24h
AppRequests | where TimeGenerated > ago(24h)
| summarize total=sum(ItemCount), failed=sumif(ItemCount, Success==false) by Name
| extend failRate = round(100.0*failed/total,2) | order by failed desc
// classic: requests, timestamp, itemCount, success

// Latency percentiles
AppRequests | where TimeGenerated > ago(6h)
| summarize p50=percentile(DurationMs,50), p95=percentile(DurationMs,95), p99=percentile(DurationMs,99) by bin(TimeGenerated,5m)
// classic: duration (ms)

// Exceptions top problems
AppExceptions | where TimeGenerated > ago(24h)
| summarize n=count() by ProblemId, ExceptionType, OuterMessage | top 20 by n
// classic: exceptions, problemId, type, outerMessage

// Dependency failures by target
AppDependencies | where TimeGenerated > ago(6h) and Success == false
| summarize n=count() by Type, Target, ResultCode | order by n desc
// classic: dependencies, type, target, resultCode

// End-to-end trace by operation id
union AppRequests, AppDependencies, AppTraces, AppExceptions
| where OperationId == "<id>" | project TimeGenerated, Type, Name, Message, DurationMs, Success, ParentId, Id | order by TimeGenerated asc
// classic: operation_Id, itemType, operation_ParentId

// Role instances and restarts
AppTraces | where Message has "Host started" | summarize count() by AppRoleName, CloudRoleInstance, bin(TimeGenerated,1h)

// Function invocation outcome (if function logs forwarded)
AppRequests | where OperationName != "" | summarize count() by OperationName, Success, ResultCode

// Sampling detection
AppRequests | summarize count(), sum(ItemCount) by bin(TimeGenerated,1h)

// Ingestion volume / cost
Usage | where TimeGenerated > ago(30d) and IsBillable == true | summarize GB=sum(Quantity)/1000 by DataType | order by GB desc
AppTraces | summarize MB=sum(_BilledSize)/1e6 by AppRoleName, SeverityLevel

// Key Vault audit (AzureDiagnostics destination)
AzureDiagnostics | where ResourceProvider=="MICROSOFT.KEYVAULT" and Category=="AuditEvent" and httpStatusCode_d >= 400
| project TimeGenerated, OperationName, CallerIPAddress, identity_claim_oid_g, ResultSignature

// Storage blob failures
StorageBlobLogs | where TimeGenerated > ago(6h) and StatusCode >= 400
| summarize n=count() by StatusCode, StatusText, AuthenticationType, OperationName

// Service Bus metrics via Azure Monitor alert/metric (use Metrics blade): ActiveMessages, DeadletteredMessages, ThrottledRequests, ServerErrors

// Activity log: who changed what
AzureActivity | where TimeGenerated > ago(7d) and CategoryValue == "Administrative" and ActivityStatusValue == "Success"
| project TimeGenerated, Caller, OperationNameValue, ResourceGroup, _ResourceId

// Entra sign-ins for service principals / managed identities
AADServicePrincipalSignInLogs | where ResultType != "0" | summarize count() by ServicePrincipalName, ResultType
AADManagedIdentitySignInLogs | where ResultType != "0" | summarize count() by ServicePrincipalName, ResultType

// Heartbeat / no-data detection
AppTraces | summarize last=max(TimeGenerated) by AppRoleName | where last < ago(15m)

// Alert history (Azure Resource Graph)
alertsmanagementresources | where type == "microsoft.alertsmanagement/alerts" | project name, properties.essentials.severity, properties.essentials.monitorCondition, properties.essentials.startDateTime
```

**Handy operators:** `bin()`, `summarize`, `arg_max/arg_min`, `percentile`, `dcount`, `make-series` + `series_decompose_anomalies` for anomalies, `join kind=leftanti` to find missing items, `parse`/`extract` for message fields, `has` (term, fast) vs `contains` (substring, slow).

**Disclaimer:** Table names, columns, defaults, limits, and CLI syntax evolve — verify in current docs before relying on any value in a production runbook.

