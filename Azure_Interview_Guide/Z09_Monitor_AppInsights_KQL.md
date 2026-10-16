# Z09 - Azure Monitor, Application Insights & KQL (Study + Interview Guide)

**How to use:** Read each topic's *Study notes*, then walk the difficulty ladder (Easy -> Tricky); cover the Answer, say it aloud, then check the Explanation/code. Finish with *Interview traps* and the *Cheat sheet* the night before.
**Conventions:** KQL uses the **classic App Insights schema** (`requests`, `timestamp`, `customDimensions`, `cloud_RoleName`) as shown in the App Insights *Logs* blade; the **workspace schema** (`AppRequests`, `TimeGenerated`, `Properties`, `AppRoleName`) is mapped in Topic 3. Anything marked *verify in current docs* can change (SDK versions, retirements, pricing, table support).

## Table of Contents
1. Observability pillars & Azure Monitor architecture
2. Workspace-based vs classic Application Insights
3. Telemetry types & tables (classic vs workspace schema)
4. Instrumenting .NET: OpenTelemetry distro vs classic SDK vs Functions
5. ILogger integration, log-level filtering, "why are my traces missing?"
6. Custom telemetry: TelemetryClient, ActivitySource, Meter, initializers/processors
7. Sampling, cost control, daily cap, retention
8. Distributed tracing & correlation (W3C, Service Bus, Functions, Logic Apps)
9. KQL fundamentals
10. KQL advanced (joins, dynamic, let, time series, materialize, cross-resource)
11. Practical KQL query library (25 queries)
12. Diagnostic settings & resource logs routing
13. Alerts: metric vs log, action groups, processing rules, alert fatigue
14. Availability tests, Live Metrics, Profiler, Snapshot Debugger
15. Workbooks, dashboards, Grafana, Container/VM insights
16. Log Analytics cost, table plans, archive, data collection rules
17. Entra auth for ingestion (SAMI/UAMI) & PII scrubbing
18. SLO/SLI/error budgets, golden signals (RED/USE), release correlation
19. Production scenarios & troubleshooting playbook

---

## Topic 1: Observability pillars & Azure Monitor architecture

**Study notes**

Observability = being able to ask *new* questions about a running system from its outputs. Three pillars:

| Pillar | What | Cost/Cardinality | Typical question |
|---|---|---|---|
| Metrics | Numeric time series, pre-aggregated | Cheap, low-cardinality | "Is CPU / queue depth / error rate rising?" |
| Logs (events) | Discrete records with context | Expensive at volume | "What exactly failed for order 123?" |
| Traces | Causal chain of spans across services (operation_Id) | Sampled | "Where did the 4s go across 6 services?" |

```
 Sources                    Ingestion / stores                 Consume
 -------                    ------------------                 -------
 Azure resources  --platform metrics-->  Azure Monitor Metrics DB --> Metrics explorer, metric alerts
 Azure resources  --diagnostic settings-> Log Analytics workspace (Kusto) --> KQL, workbooks, log alerts
 App (SDK/OTel)   --connection string--> Application Insights (-> workspace) --> App map, Live Metrics, Search
 VMs/containers   --AMA + DCR----------> Log Analytics / Prometheus store --> Insights, Grafana
 Activity Log (subscription control plane) --> workspace / alerts
                                         Alerts -> Action groups -> email/webhook/Logic App/Function/ITSM
```

Key components: **Metrics store** (platform metrics, ~93 days, 1-min granularity, free), **Log Analytics workspace** (KQL store, tables, retention, cost), **Application Insights** (APM feature layer over a workspace), **Workbooks** (interactive reports), **Alerts** (metric, log search, activity log, Prometheus), **Action groups** (notification/automation targets), **Data Collection Rules (DCR)** (what to collect/transform/where to send).

### 1.1 [Easy] What are the three pillars of observability and how do they map to Azure Monitor?
**Answer:** Metrics, logs and traces. In Azure, platform and custom metrics live in the Azure Monitor metrics store, logs live in Log Analytics tables, and application traces/telemetry are collected by Application Insights (stored in a Log Analytics workspace).
**Explanation:** Metrics tell you *that* something is wrong (cheap, fast, alertable at 1-min). Logs tell you *why* (rich context). Traces tell you *where* in a distributed call chain. Application Insights unifies requests, dependencies, exceptions and traces correlated by `operation_Id`.
**Code / KQL / CLI / Bicep**
```kusto
// One query touching all three ideas: metric-like aggregate, log detail, trace correlation id
requests
| where timestamp > ago(1h)
| summarize reqs = sum(itemCount), failed = sumif(itemCount, success == false) by bin(timestamp, 5m)
| render timechart
```

### 1.2 [Follow-up] Where does data go: what is the difference between Azure Monitor Metrics and a Log Analytics workspace?
**Answer:** Metrics is a purpose-built time-series database for numeric, pre-aggregated values with near-real-time alerting; Log Analytics is a Kusto-based store for arbitrary records queried with KQL. Choose metrics for fast cheap threshold alerts and logs for investigation and complex conditions.
**Explanation:** Platform metrics are collected automatically and free; resource *logs* must be routed with a diagnostic setting. Metrics retention is ~93 days; workspace retention is configurable. Some resources emit the same signal both ways (e.g., Service Bus `DeadletteredMessages` metric vs operational logs).
**Code / KQL / CLI / Bicep**
```bash
# Query a platform metric (Service Bus dead-lettered messages)
az monitor metrics list --resource <sb-namespace-resource-id> \
  --metric DeadletteredMessages --interval PT5M --aggregation Maximum \
  --filter "EntityName eq 'orders'"
```

### 1.3 [Medium] How does Application Insights relate to Log Analytics today?
**Answer:** A workspace-based Application Insights resource stores its telemetry in a Log Analytics workspace (App* tables); App Insights is the APM experience (App map, Live Metrics, transaction search) layered on top. Classic (own storage) resources were retired, so everything new is workspace-based (verify retirement/migration status in current docs).
**Explanation:** Benefits: one workspace for infra + app logs, unified RBAC, cross-resource KQL joins, DCR transformations, commitment-tier pricing, CMK. In the App Insights *Logs* blade you still query with classic table names (compat layer); in the workspace blade you use `App*` names.
**Code / KQL / CLI / Bicep**
```bicep
resource law 'Microsoft.OperationalInsights/workspaces@2022-10-01' = {
  name: 'law-prod'
  location: resourceGroup().location
  properties: { sku: { name: 'PerGB2018' }, retentionInDays: 30 }
}
resource appi 'Microsoft.Insights/components@2020-02-02' = {
  name: 'appi-orders'
  location: resourceGroup().location
  kind: 'web'
  properties: {
    Application_Type: 'web'
    WorkspaceResourceId: law.id
    DisableLocalAuth: false   // true => Entra-only ingestion (see Topic 17)
  }
}
```

### 1.4 [Medium] Explain the alerting architecture end to end.
**Answer:** An alert *rule* (metric, log search, activity log) evaluates a signal and fires an *alert instance*; *alert processing rules* can suppress or route it; *action groups* execute notifications/automation (email, webhook, Logic App, Function, ITSM). Rules are decoupled from actions so one action group is reused by many rules.
**Explanation:** Metric alerts evaluate at 1-minute minimum frequency with low latency; log search alerts run a KQL query on a schedule (1 min+) against ingested data so they incur ingestion latency. Stateful alerts stay "Fired" until resolved; you can auto-resolve.
**Code / KQL / CLI / Bicep**
```bash
az monitor action-group create -g rg-mon -n ag-oncall --short-name oncall \
  --action email me me@contoso.com
az monitor metrics alert create -g rg-mon -n sb-dlq-high \
  --scopes <sb-id> --condition "max DeadletteredMessages > 0" \
  --window-size 5m --evaluation-frequency 1m --action ag-oncall
```

### 1.5 [Hard] You must design monitoring for 30 microservices across 3 subscriptions. What topology do you choose?
**Answer:** Typically one Application Insights resource per *environment/product boundary* (not per service) feeding a shared workspace, with `cloud_RoleName` distinguishing services; or one workspace per environment/region if data residency or RBAC demands it. Avoid one-App-Insights-per-microservice because it fragments the Application Map and correlation.
**Explanation:** The Application Map and end-to-end view stitch by `operation_Id` within a resource; cross-resource correlation works but is clunkier (`app("x")`). Workspace boundaries are your RBAC, retention, cost-cap and residency boundaries. Use resource-context RBAC or table-level RBAC for isolation. Consider a central workspace + per-team App Insights pointing to it if teams need separate keys/caps.
**Code / KQL / CLI / Bicep**
```kusto
// Cross-resource: look across two App Insights resources
union app('appi-orders').requests, app('appi-billing').requests
| where timestamp > ago(1h)
| summarize count() by cloud_RoleName, appId
```
**Gotchas / follow-ups**
- A daily cap is per workspace; one noisy service can starve others - consider separate workspaces or per-table transformations.
- Cross-workspace queries are slower and limited (100 resources/workspaces per query - verify).

**Interview traps**
- "Metrics and logs are the same thing" - no: different stores, latency, retention, cost.
- Saying App Insights has its own storage today - it's workspace-based.
- Forgetting resource logs are *not* collected until a diagnostic setting exists.
- One App Insights per microservice breaks the Application Map.

**Cheat sheet**
- Metrics: ~93 days, free platform, 1-min, fast alerts. Logs: KQL, per-GB cost.
- Resource logs need diagnostic settings; Activity Log is control plane.
- App Insights = APM layer over a workspace.
- Alert rule -> alert -> (processing rule) -> action group.

---

## Topic 2: Workspace-based vs classic Application Insights

**Study notes**

| Aspect | Classic (retired - verify) | Workspace-based |
|---|---|---|
| Storage | Internal AI store | Your Log Analytics workspace |
| Table names | `requests`, `traces`... | `AppRequests`, `AppTraces`... |
| Time column | `timestamp` | `TimeGenerated` |
| Retention | 90 days default, 730 max | Workspace retention (App Insights tables default 90 days free - verify) |
| Cap/pricing | Own | Workspace pricing/cap/commitment tiers |
| Entra RBAC, CMK, DCR transform | Limited | Yes |

Classic resources were scheduled for retirement (ingestion cut-off announced for Feb 2024, with migration to workspace-based; **verify current status**). Existing classic resources must be migrated via portal *Migrate to Workspace-based* or ARM `WorkspaceResourceId`.

### 2.1 [Easy] What is a workspace-based Application Insights resource?
**Answer:** An App Insights resource whose telemetry is stored in a Log Analytics workspace you choose. You get the same App Insights experiences but data lives in `App*` tables with workspace features (RBAC, retention, cost, DCR).
**Explanation:** The resource still has its own connection string/instrumentation key for ingestion; the workspace is the storage target. You can query from either blade.
**Code / KQL / CLI / Bicep**
```bash
az monitor app-insights component create -a appi-orders -g rg-mon -l westeurope \
  --workspace /subscriptions/<sub>/resourceGroups/rg-mon/providers/Microsoft.OperationalInsights/workspaces/law-prod
```

### 2.2 [Follow-up] What changes for my queries after migrating?
**Answer:** In the App Insights *Logs* blade nothing changes (classic names still work); in the workspace you use `App*` tables and PascalCase columns. Saved queries, alerts and workbooks scoped to the workspace must be rewritten.
**Explanation:** Mapping: `timestamp`->`TimeGenerated`, `duration`->`DurationMs`, `customDimensions`->`Properties`, `cloud_RoleName`->`AppRoleName`, `operation_Id`->`OperationId`, `itemCount`->`ItemCount`, `appName`->`_ResourceId`/`ResourceGUID`.
**Code / KQL / CLI / Bicep**
```kusto
// Workspace schema
AppRequests
| where TimeGenerated > ago(1h) and Success == false
| summarize Failed = sum(ItemCount) by AppRoleName, ResultCode
```

### 2.3 [Medium] Why does migration matter for cost and retention?
**Answer:** After migration, ingestion is billed at workspace rates (Pay-as-you-go or commitment tier) and retention is workspace-driven, so you can combine tiers across all data and apply table-level retention/plans.
**Explanation:** Retention for the App Insights tables beyond the free included period is billed. A daily cap is configured on the workspace (the old App Insights cap no longer applies to workspace-based data - verify). You can set per-table retention (e.g., `AppTraces` 30 days, `AppRequests` 90 days).
**Code / KQL / CLI / Bicep**
```bash
az monitor log-analytics workspace table update -g rg-mon --workspace-name law-prod \
  -n AppTraces --retention-time 30 --total-retention-time 180
```

### 2.4 [Hard] After migration an old alert on `requests` in the workspace stopped working. Why?
**Answer:** Alerts scoped to the workspace query workspace tables, which are named `AppRequests`; the classic `requests` name only resolves in the App Insights resource scope. Rescope the alert to the App Insights resource or rewrite it with App* names.
**Explanation:** Resource-scoped alerts on the App Insights component keep the compat names. Mixed scopes confuse teams; standardize one. Also `appName`-based filters must switch to `_ResourceId`.
**Code / KQL / CLI / Bicep**
```kusto
AppRequests
| where _ResourceId =~ "/subscriptions/<sub>/resourcegroups/rg-app/providers/microsoft.insights/components/appi-orders"
| summarize count() by bin(TimeGenerated, 5m)
```
**Gotchas / follow-ups**
- `_ResourceId` is lower-cased in some tables; use `=~`.
- Classic availability/`performanceCounters` names also changed (`AppPerformanceCounters`).

**Interview traps**
- Claiming classic is still a supported choice for new work.
- Forgetting migration is one-way.
- Confusing resource-scope vs workspace-scope query names.

**Cheat sheet**
- New = workspace-based always. Migration is one-way.
- `requests` (AI blade) = `AppRequests` (workspace).
- Retention/cap/pricing now at workspace.

---

## Topic 3: Telemetry types & tables (classic vs workspace schema)

**Study notes**

| Telemetry | Classic table | Workspace table | Produced by |
|---|---|---|---|
| Incoming request / function invocation | `requests` | `AppRequests` | ASP.NET Core, Functions host |
| Outgoing call (HTTP, SQL, Redis, SB, Blob) | `dependencies` | `AppDependencies` | auto-collection / TrackDependency |
| Handled/unhandled exception | `exceptions` | `AppExceptions` | TrackException / ILogger.LogError(ex) |
| Log message | `traces` | `AppTraces` | ILogger / TrackTrace |
| Custom event | `customEvents` | `AppEvents` | TrackEvent |
| Custom metric | `customMetrics` | `AppMetrics` | TrackMetric / Meter |
| Browser page view | `pageViews` | `AppPageViews` | JS SDK |
| Browser timings | `browserTimings` | `AppBrowserTimings` | JS SDK |
| Availability result | `availabilityResults` | `AppAvailabilityResults` | availability tests |
| PerfCounters | `performanceCounters` | `AppPerformanceCounters` | SDK |

Common columns (classic -> workspace): `timestamp`->`TimeGenerated`; `name`->`Name`; `duration` (ms)->`DurationMs`; `success`->`Success`; `resultCode`->`ResultCode`; `customDimensions`->`Properties`; `customMeasurements`->`Measurements`; `operation_Id`->`OperationId`; `operation_ParentId`->`ParentId`; `id`->`Id`; `cloud_RoleName`->`AppRoleName`; `cloud_RoleInstance`->`AppRoleInstance`; `itemCount`->`ItemCount`; `sdkVersion`->`SDKVersion`; `application_Version`->`AppVersion`. Dependencies: `type`->`DependencyType`, `target`->`Target`, `data`->`Data`. Exceptions: `type`->`ExceptionType`, `outerMessage`->`OuterMessage`, `innermostMessage`->`InnermostMessage`, `problemId`->`ProblemId`. Traces: `message`->`Message`, `severityLevel`->`SeverityLevel` (0 Verbose,1 Info,2 Warning,3 Error,4 Critical). Verify exact columns with `getschema`.

### 3.1 [Easy] Which table do I query to see failed incoming HTTP calls?
**Answer:** `requests` (classic naming) / `AppRequests` (workspace) filtered by `success == false` or `resultCode >= 500`.
**Explanation:** A request is the server-side span; `success` is derived from status code (<400 or 401 considered ok per defaults) but may differ for custom logic. `resultCode` is a string in classic.
**Code / KQL / CLI / Bicep**
```kusto
requests
| where timestamp > ago(1h) and success == false
| summarize failed = sum(itemCount) by name, resultCode
| order by failed desc
```

### 3.2 [Follow-up] What is the difference between `traces`, `customEvents` and `exceptions`?
**Answer:** `traces` are free-text log lines (ILogger), `customEvents` are named business/usage events with properties, and `exceptions` carry type, message and stack. Use events for analytics (funnels), traces for diagnostics, exceptions for failures.
**Explanation:** `ILogger.LogError(ex, ...)` produces an *exception* record (not a trace). `LogInformation` produces a trace with `severityLevel` 1. Logging an exception with only the message as string yields only a trace.
**Code / KQL / CLI / Bicep**
```kusto
union traces, exceptions, customEvents
| where timestamp > ago(30m) and operation_Id == "<operation id>"
| project timestamp, itemType, name, message, type, outerMessage
| order by timestamp asc
```

### 3.3 [Medium] How do I discover the schema instead of memorizing it?
**Answer:** Use `getschema`, `take 1`, and `union withsource=T *` to list which tables hold data. In workspace scope, browse the `App*` tables.
**Explanation:** Schemas evolve (new columns) so treat docs as reference and the live schema as truth.
**Code / KQL / CLI / Bicep**
```kusto
requests | getschema
union withsource=SourceTable * | where timestamp > ago(1h) | summarize count() by SourceTable
```

### 3.4 [Medium] How is a dependency represented and what are `type`/`target`/`data`?
**Answer:** A dependency is an outgoing call: `type` (HTTP, SQL, Azure Service Bus, Azure blob, Redis...), `target` (host/endpoint), `data` (URL or command text), `name` (operation), `duration`, `success`, `resultCode`.
**Explanation:** Type strings vary by SDK/instrumentation (e.g., classic: "Azure blob", "Azure Service Bus", "SQL"; OTel distro may use "HTTP", "mssql", "Redis" - verify with a `distinct type` query). Don't hardcode; discover first.
**Code / KQL / CLI / Bicep**
```kusto
dependencies | where timestamp > ago(1d) | summarize count() by type, target | top 30 by count_
```

### 3.5 [Hard] Why does `count()` on requests not match the real number of calls?
**Answer:** Because of sampling: each stored row represents `itemCount` original items. Use `sum(itemCount)` for counts of sampled item types (requests, dependencies, traces, exceptions, events), not `count()`.
**Explanation:** Adaptive/fixed-rate sampling keeps 1 in N and sets `itemCount = N` on kept items. Metrics (pre-aggregated standard metrics) are accurate regardless. `customMetrics` rows are aggregates, with `valueCount`/`valueSum`, not `itemCount`-weighted the same way; check the aggregation fields.
**Code / KQL / CLI / Bicep**
```kusto
requests
| where timestamp > ago(1h)
| summarize rows = count(), realCount = sum(itemCount)
```
**Gotchas / follow-ups**
- Percentiles over sampled data are estimates (kept rows are a random subset).
- Same sampling decision (operation_Id based) keeps related items together, so traces stay whole.

### 3.6 [Tricky] A dependency shows `success == true` but the user saw an error. Possible causes?
**Answer:** The failure was handled/swallowed (try/catch), a 404/409 treated as success by the app, retries masked it, or the failing call wasn't instrumented. Dependency `success` mirrors the call's status/exception, not business outcome.
**Explanation:** Many Azure SDKs retry internally (Service Bus/Blob/Redis), so an eventual success hides intermediate failures; check the retry spans or SDK logging. HTTP 404 on a blob GET may be flagged success depending on the instrumentation.
**Code / KQL / CLI / Bicep**
```kusto
dependencies
| where timestamp > ago(1h) and operation_Id == "<id>"
| project timestamp, name, type, target, resultCode, success, duration
| order by timestamp asc
```
**Gotchas / follow-ups**
- Set span status/Activity status manually for business failures (`activity.SetStatus(ActivityStatusCode.Error)`).

**Interview traps**
- Using `count()` instead of `sum(itemCount)`.
- `LogError(message)` without exception object -> no `exceptions` row.
- Assuming type strings are identical across SDKs.
- Using classic names in workspace scope.

**Cheat sheet**
- requests=server span, dependencies=client span, traces=logs, exceptions=errors, events/metrics=custom.
- Counts: `sum(itemCount)`.
- Workspace: `App*`, `TimeGenerated`, `Properties`, `AppRoleName`, `OperationId`.
- Learn schema with `getschema`.

---

## Topic 4: Instrumenting .NET - OpenTelemetry distro vs classic SDK vs Functions

**Study notes**

| Option | Package | Entry point | Status |
|---|---|---|---|
| Azure Monitor OpenTelemetry distro (ASP.NET Core) | `Azure.Monitor.OpenTelemetry.AspNetCore` | `builder.Services.AddOpenTelemetry().UseAzureMonitor()` | Recommended for new apps |
| Exporter only (non-ASP.NET / custom) | `Azure.Monitor.OpenTelemetry.Exporter` | `.AddAzureMonitorTraceExporter()` etc. | Low-level |
| Classic SDK (ASP.NET Core) | `Microsoft.ApplicationInsights.AspNetCore` | `AddApplicationInsightsTelemetry()` | Maintained, not the future |
| Worker/console classic | `Microsoft.ApplicationInsights.WorkerService` | `AddApplicationInsightsTelemetryWorkerService()` | Maintained |
| Functions isolated worker (classic AI) | `Microsoft.Azure.Functions.Worker.ApplicationInsights` | `ConfigureFunctionsApplicationInsights()` | Supported |
| Functions with OpenTelemetry | `Microsoft.Azure.Functions.Worker.OpenTelemetry` + exporter | `UseFunctionsWorkerDefaults()` | Verify in current docs |

Connection string (`InstrumentationKey=...;IngestionEndpoint=https://<region>.in.applicationinsights.azure.com/;...`) is preferred over the bare instrumentation key: it carries the regional ingestion endpoint (needed for sovereign clouds/regional endpoints, Entra auth, private link). Instrumentation-key-only configuration is being phased out for new features (verify). Set via `APPLICATIONINSIGHTS_CONNECTION_STRING` env var (works for both SDKs and the Functions host).

### 4.1 [Easy] How do I add Application Insights to an ASP.NET Core app today?
**Answer:** Install `Azure.Monitor.OpenTelemetry.AspNetCore` and call `UseAzureMonitor()`, supplying the connection string by config or the `APPLICATIONINSIGHTS_CONNECTION_STRING` environment variable. It auto-collects requests, dependencies, exceptions, logs and metrics.
**Explanation:** The distro bundles ASP.NET Core, HttpClient and SqlClient instrumentation, plus Azure SDK activity sources, Live Metrics and standard metrics.
**Code / KQL / CLI / Bicep**
```csharp
using Azure.Monitor.OpenTelemetry.AspNetCore;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddOpenTelemetry().UseAzureMonitor(); // reads APPLICATIONINSIGHTS_CONNECTION_STRING
var app = builder.Build();
app.MapGet("/ping", () => "ok");
app.Run();
```

### 4.2 [Follow-up] Distro vs classic SDK: what are the practical differences?
**Answer:** The distro is OpenTelemetry-based (vendor-neutral APIs: `ActivitySource`, `Meter`, `ILogger`), while the classic SDK uses `TelemetryClient`, initializers and processors. The classic SDK is richer for legacy features but OpenTelemetry is the strategic direction.
**Explanation:** Mapping differences: OTel resource attributes `service.name` -> `cloud_RoleName`, `service.instance.id` -> `cloud_RoleInstance`, `service.version` -> `application_Version`. Classic sampling is adaptive by default; the distro defaults to rate-limited sampling (`TracesPerSecond`, verify default). Classic `ITelemetryInitializer` maps to OTel processors. Some classic-only features (e.g., some auto-collected perf counters, certain TrackX semantics, Snapshot Debugger, Profiler) have differing support: check the feature matrix in current docs.
**Code / KQL / CLI / Bicep**
```csharp
using Azure.Monitor.OpenTelemetry.AspNetCore;
using OpenTelemetry.Resources;

builder.Services.AddOpenTelemetry()
    .ConfigureResource(r => r
        .AddService(serviceName: "orders-api", serviceNamespace: "shop", serviceVersion: "1.4.2"))
    .UseAzureMonitor(o =>
    {
        o.SamplingRatio = 1.0f; // verify option names in current docs
    });
```

### 4.3 [Medium] Connection string vs instrumentation key - why does it matter?
**Answer:** The connection string includes the ingestion endpoint (regional/private/sovereign) and optional live endpoint, so telemetry goes to the right place; an ikey alone defaults to the global endpoint and can't express Entra auth or private-link scopes. Microsoft recommends connection strings; ikey-only support is going away (verify).
**Explanation:** Common outage: app configured with only the ikey while network rules allow only the regional ingestion endpoint -> silent telemetry drop. Store the connection string in app settings (not secret, but not for public repos); it's an identifier rather than a credential unless local auth is the only gate.
**Code / KQL / CLI / Bicep**
```bash
az functionapp config appsettings set -g rg-app -n func-orders \
  --settings "APPLICATIONINSIGHTS_CONNECTION_STRING=InstrumentationKey=<guid>;IngestionEndpoint=https://westeurope-5.in.applicationinsights.azure.com/;LiveEndpoint=https://westeurope.livediagnostics.monitor.azure.com/"
```

### 4.4 [Medium] How do I wire Application Insights into an isolated-worker Azure Function?
**Answer:** Register the worker integration in `Program.cs` with `AddApplicationInsightsTelemetryWorkerService()` and `ConfigureFunctionsApplicationInsights()`, and set the connection string app setting. The host also sends its own telemetry via host.json settings, so the two configs work together.
**Explanation:** In-process model is retired/going out of support (verify; .NET 8 in-process support ends Nov 2026). Isolated worker splits telemetry: the **host** emits `requests` for invocations and `Host.*` traces; the **worker** emits your ILogger logs and dependencies. Correlation between them is via W3C context.
**Code / KQL / CLI / Bicep**
```csharp
var host = new HostBuilder()
    .ConfigureFunctionsWebApplication()
    .ConfigureServices(s =>
    {
        s.AddApplicationInsightsTelemetryWorkerService();
        s.ConfigureFunctionsApplicationInsights();
    })
    .Build();
host.Run();
```

### 4.5 [Hard] Telemetry is duplicated (every request appears twice). Diagnose.
**Answer:** Two pipelines are sending the same data: for example both the classic SDK and the distro/OpenTelemetry are registered, or the Functions host's integration plus a manually added exporter, or two App Insights registrations in DI. Remove one pipeline.
**Explanation:** Check `sdkVersion` values (`dotnetc:`/`dotnet8:` classic vs `dotnet:` ... prefixes like `dotnet8:otel...` - verify) and `cloud_RoleInstance`; two distinct sdkVersions for the same operation_Id proves two pipelines. Also App Service "auto-instrumentation" (codeless agent) *plus* an SDK causes duplicates; the agent usually defers but verify.
**Code / KQL / CLI / Bicep**
```kusto
requests
| where timestamp > ago(30m)
| summarize rows = count(), sdks = make_set(sdkVersion) by operation_Id, id
| where rows > 1
| take 20
```
**Gotchas / follow-ups**
- Duplicate `traces` often come from both ILogger provider and `TrackTrace` wrapper.
- Two App Insights resources in the same app also look like "missing" data in one.

**Interview traps**
- Mixing `AddApplicationInsightsTelemetry()` and `UseAzureMonitor()` in the same app.
- Treating the connection string as a secret to protect with Key Vault (it's not a credential; Entra auth is the control).
- Forgetting the Functions host and worker are separate telemetry emitters.

**Cheat sheet**
- New apps: `UseAzureMonitor()` + `ILogger`/ActivitySource/Meter.
- Config: `APPLICATIONINSIGHTS_CONNECTION_STRING`.
- Role: `service.name` -> `cloud_RoleName`; version -> `application_Version`.
- Functions isolated: `ConfigureFunctionsApplicationInsights()`.
- Duplicates = two pipelines.

---

## Topic 5: ILogger integration, log-level filtering, "why are my traces missing?"

**Study notes**

Pipeline: `ILogger` -> logging filters (per provider, per category) -> App Insights provider -> sampling -> ingestion. A log is dropped at the **first** gate that rejects it.

```
logger.LogInformation()
   |-- (1) global/provider LogLevel filter (appsettings "Logging" / host.json "logging")
   |-- (2) provider-specific rule ("ApplicationInsights": {...})
   |-- (3) sampling (adaptive/fixed)   -> itemCount
   `-- (4) ingestion (daily cap, network, auth)
```

ASP.NET Core classic SDK: default App Insights provider filter is **Warning** unless configured. Functions: `host.json` -> `logging.logLevel` governs host and (for in-process) function logs; `logging.applicationInsights.samplingSettings` governs sampling. Isolated worker: **the worker applies its own default filter for the App Insights provider (Warning+)** that you must remove in code (verify in current docs).

### 5.1 [Easy] My `LogInformation` calls don't show up in App Insights. First thing to check?
**Answer:** The minimum log level for the category/provider. By default, the Application Insights provider often only captures Warning and above, so Information logs are filtered before they leave the process.
**Explanation:** Configure `Logging:ApplicationInsights:LogLevel` (ASP.NET Core) or host.json `logging.logLevel` (Functions). Also confirm the logger category matches your filter (category = fully-qualified class name by default).
**Code / KQL / CLI / Bicep**
```json
{
  "Logging": {
    "LogLevel": { "Default": "Warning", "Microsoft.AspNetCore": "Warning" },
    "ApplicationInsights": { "LogLevel": { "Default": "Information", "MyCompany.Orders": "Debug" } }
  }
}
```

### 5.2 [Follow-up] How do I configure log levels for an Azure Function?
**Answer:** In `host.json` under `logging.logLevel` (categories like `Function`, `Host.Results`, `Host.Aggregator`) and `logging.applicationInsights.samplingSettings`. App settings override via `AzureFunctionsJobHost__logging__logLevel__default`.
**Explanation:** `Host.Results` = the request telemetry for invocations; `Host.Aggregator` = metrics aggregated every 30s; `Function.<Name>.User` = your ILogger logs in-process. Setting `Host.Results` to Warning silences successful `requests` (don't do it unless intended).
**Code / KQL / CLI / Bicep**
```json
{
  "version": "2.0",
  "logging": {
    "logLevel": {
      "default": "Warning",
      "Host.Results": "Information",
      "Function": "Information",
      "Host.Aggregator": "Trace"
    },
    "applicationInsights": {
      "samplingSettings": { "isEnabled": true, "maxTelemetryItemsPerSecond": 20, "excludedTypes": "Request;Exception" }
    }
  }
}
```

### 5.3 [Medium] Isolated worker Function: host.json says Information but worker logs are missing. Why?
**Answer:** The worker process has its own logging pipeline; the App Insights provider is added with a default rule that filters out anything below Warning, and host.json doesn't govern worker logs. Remove that default rule or set levels in worker config (appsettings/code).
**Explanation:** Documented workaround: remove the rule for `Microsoft.Extensions.Logging.ApplicationInsights.ApplicationInsightsLoggerProvider` from `LoggerFilterOptions` so `Information` passes. Verify current docs because behavior has been adjusted across package versions.
**Code / KQL / CLI / Bicep**
```csharp
.ConfigureServices(s =>
{
    s.AddApplicationInsightsTelemetryWorkerService();
    s.ConfigureFunctionsApplicationInsights();
    s.Configure<LoggerFilterOptions>(o =>
    {
        var rule = o.Rules.FirstOrDefault(r => r.ProviderName ==
            "Microsoft.Extensions.Logging.ApplicationInsights.ApplicationInsightsLoggerProvider");
        if (rule is not null) o.Rules.Remove(rule);
    });
})
```

### 5.4 [Medium] How do structured logs appear and how do I query their properties?
**Answer:** Message-template placeholders become `customDimensions` keys (prefixed `prop__`) while `{OriginalFormat}` holds the template. Query with `customDimensions["prop__OrderId"]`.
**Explanation:** Always use templates (`"Order {OrderId} failed"`), never string interpolation, so properties are indexed as structured and the template groups identical messages. Scopes (`BeginScope`) add dimensions too.
**Code / KQL / CLI / Bicep**
```csharp
using (_logger.BeginScope(new Dictionary<string, object> { ["TenantId"] = tenant }))
    _logger.LogWarning("Order {OrderId} failed after {Attempts} attempts", id, attempts);
```
```kusto
traces
| where timestamp > ago(1h) and customDimensions["prop__OrderId"] == "123"
| project timestamp, message, severityLevel, customDimensions
```

### 5.5 [Hard] Traces are present in dev but missing in prod only. Systematic checklist?
**Answer:** (1) Log level overrides in prod app settings, (2) sampling in prod (`itemCount` > 1 - or exclusion), (3) daily cap hit, (4) wrong connection string/ikey or resource, (5) network/private link/firewall blocking ingestion, (6) Entra-only ingestion (`DisableLocalAuth`) without a credential, (7) ingestion latency or wrong time range.
**Explanation:** Prove ingestion health: query `traces` of any kind; check the workspace `Operation` table for "data collection stopped" (daily cap) and `_LogOperation`. On the host, check live: Live Metrics shows data before storage, isolating pipeline vs storage issues. For local debugging, enable SDK self-diagnostics (`ApplicationInsightsEventListener`/ OTel `OTEL_LOG_LEVEL`/`APPLICATIONINSIGHTS_...` - verify names).
**Code / KQL / CLI / Bicep**
```kusto
// Daily cap hit?
_LogOperation
| where TimeGenerated > ago(7d) and Category == "Ingestion"
| project TimeGenerated, Operation, Level, Detail
```
```kusto
// Is anything arriving, and from which SDK / role?
union traces, requests, dependencies
| where timestamp > ago(30m)
| summarize cnt = sum(itemCount), last = max(timestamp) by cloud_RoleName, sdkVersion
```
**Gotchas / follow-ups**
- `_LogOperation` exists in the workspace scope (not AI scope) - run it in the workspace.
- Sampling can drop a single trace even when the rate looks fine; trace volume per operation matters.

### 5.6 [Tricky] Verbose `Debug` logging was enabled in prod and the bill spiked. How do you prevent a repeat?
**Answer:** Set per-category levels (noisy namespaces at Warning), use sampling for traces, use DCR transformations or table plans to drop/reduce, and add daily cap/budget alerts. Add `Microsoft.*`/`System.*` overrides because framework categories are the usual source.
**Explanation:** HttpClient/Azure SDK/EF Core categories emit lots at Information/Debug. Find the top contributors by `message`/category and cost per table.
**Code / KQL / CLI / Bicep**
```kusto
traces
| where timestamp > ago(1d)
| summarize rows = sum(itemCount), bytes = sum(_BilledSize) by cat = tostring(customDimensions["Category"])
| top 15 by bytes desc
```
**Gotchas / follow-ups**
- `_BilledSize` is available in the workspace (App* tables) and in the AI blade for billing analysis - verify availability in your scope.

**Interview traps**
- Believing host.json controls isolated-worker app logs.
- Using string interpolation in log messages (breaks grouping & structured props).
- `LogError(msg)` without the exception.
- Forgetting the Warning default for the AI provider.

**Cheat sheet**
- Drop points: level filter -> sampling -> cap -> network/auth.
- Functions: `host.json` `logging.logLevel`; env override with `__`.
- Structured props: `customDimensions["prop__X"]`.
- Fix cost at source: categories, sampling, DCR transform.

---

## Topic 6: Custom telemetry - TelemetryClient, ActivitySource, Meter, initializers/processors

**Study notes**

| Need | Classic SDK | OpenTelemetry |
|---|---|---|
| Custom event | `TrackEvent` | ILogger with event / `activity.AddEvent` (events -> traces; verify mapping) |
| Metric | `GetMetric("name").TrackValue(x)` | `Meter.CreateCounter/Histogram` |
| Custom dependency | `StartOperation<DependencyTelemetry>` | `ActivitySource.StartActivity(name, ActivityKind.Client)` |
| Enrich every item | `ITelemetryInitializer` | `BaseProcessor<Activity>` / `BaseProcessor<LogRecord>` |
| Filter items | `ITelemetryProcessor` | Processor that clears `ActivityTraceFlags.Recorded` or sampler |

OTel `ActivityKind`: Server -> `requests`; Client/Producer -> `dependencies`; Internal -> `dependencies` (type "InProc"); Consumer -> `requests`.

### 6.1 [Easy] How do I log a custom business event?
**Answer:** With the classic SDK use `TelemetryClient.TrackEvent("OrderPlaced", properties, metrics)`. In OpenTelemetry, log through `ILogger` or record an Activity event; custom events as a first-class `customEvents` row is a classic-SDK concept (verify distro mapping).
**Explanation:** Events feed funnels/users/sessions analysis. Properties are strings, metrics are doubles. Keep property cardinality bounded.
**Code / KQL / CLI / Bicep**
```csharp
telemetry.TrackEvent("OrderPlaced",
    new Dictionary<string,string> { ["Tenant"] = "contoso", ["Channel"] = "web" },
    new Dictionary<string,double> { ["Amount"] = 99.5 });
```
```kusto
customEvents | where name == "OrderPlaced" | summarize orders = sum(itemCount) by bin(timestamp, 1h), tostring(customDimensions.Channel)
```

### 6.2 [Follow-up] Custom metrics: `TrackMetric` vs `GetMetric` vs OpenTelemetry `Meter`?
**Answer:** `GetMetric` pre-aggregates locally and sends one aggregated point per minute (cheap, recommended); `TrackMetric` sends each value raw (expensive, sampling-affected). In OTel use `Meter` instruments, exported via the distro to `customMetrics`.
**Explanation:** Pre-aggregation avoids sampling distortion and cost. Dimensions multiply series count: limit dimension cardinality (verify custom metrics dimension caps in docs).
**Code / KQL / CLI / Bicep**
```csharp
// OpenTelemetry
static readonly Meter M = new("Shop.Orders", "1.0");
static readonly Counter<long> Placed = M.CreateCounter<long>("orders_placed");
static readonly Histogram<double> Latency = M.CreateHistogram<double>("order_latency_ms");
Placed.Add(1, new KeyValuePair<string,object?>("channel", "web"));

builder.Services.AddOpenTelemetry().UseAzureMonitor().WithMetrics(b => b.AddMeter("Shop.Orders"));
```
```kusto
customMetrics | where name == "orders_placed" | summarize total = sum(valueSum) by bin(timestamp, 5m)
```

### 6.3 [Medium] How do I add custom spans/dependencies in OpenTelemetry?
**Answer:** Create an `ActivitySource`, register it with `.WithTracing(t => t.AddSource("name"))`, and start activities around operations. They show up in the transaction view correlated under the parent.
**Explanation:** If the source isn't registered, `StartActivity` returns `null` (no listener) - a very common "my custom spans don't appear" cause.
**Code / KQL / CLI / Bicep**
```csharp
static readonly ActivitySource Src = new("Shop.Orders");

using var act = Src.StartActivity("PriceCalculation", ActivityKind.Internal);
act?.SetTag("order.id", orderId);
try { /* work */ }
catch (Exception ex) { act?.SetStatus(ActivityStatusCode.Error, ex.Message); throw; }

builder.Services.AddOpenTelemetry().UseAzureMonitor().WithTracing(t => t.AddSource("Shop.Orders"));
```

### 6.4 [Medium] How do I enrich all telemetry with tenant/user info (and set the role name)?
**Answer:** Classic: implement `ITelemetryInitializer` and register it in DI. OpenTelemetry: write a `BaseProcessor<Activity>` that sets tags in `OnEnd` (and one for `LogRecord`) and add it to the provider. Role name via resource `service.name`.
**Explanation:** Initializers run on every item at creation; keep them fast and allocation-light. Do not read HttpContext on non-request threads.
**Code / KQL / CLI / Bicep**
```csharp
// Classic
public sealed class RoleInitializer : ITelemetryInitializer
{
    public void Initialize(ITelemetry t) { t.Context.Cloud.RoleName = "orders-api"; t.Context.Component.Version = "1.4.2"; }
}
services.AddSingleton<ITelemetryInitializer, RoleInitializer>();

// OpenTelemetry
public sealed class TenantProcessor : BaseProcessor<Activity>
{
    public override void OnEnd(Activity a) => a.SetTag("tenant", Environment.GetEnvironmentVariable("TENANT"));
}
builder.Services.AddOpenTelemetry().UseAzureMonitor().WithTracing(t => t.AddProcessor(new TenantProcessor()));
```

### 6.5 [Hard] Telemetry from a background `Task`/queue handler is orphaned (no operation). Why and how to fix?
**Answer:** There is no ambient Activity/operation, so items get a new `operation_Id`. Start an operation explicitly (classic `StartOperation<RequestTelemetry>`, OTel `ActivitySource.StartActivity(..., ActivityKind.Consumer, parentContext)`) and restore parent context from the message.
**Explanation:** `Activity.Current` flows via `AsyncLocal`; `Task.Run` flows it, but Channels/queues/`Timer`/hosted-service loops don't. For messaging, extract the `traceparent` from message properties and pass an `ActivityContext` as parent.
**Code / KQL / CLI / Bicep**
```csharp
if (ActivityContext.TryParse(msg.ApplicationProperties["traceparent"]?.ToString(), null, out var parent))
{
    using var act = Src.StartActivity("Process order", ActivityKind.Consumer, parent);
    await HandleAsync(msg);
}
```
**Gotchas / follow-ups**
- Prefer links (`ActivityLink`) for batch consumers where one span handles many messages.

### 6.6 [Tricky] `TrackDependency` shows in the transaction view but not nested under the request. Why?
**Answer:** The dependency's `operation_Id`/`operation_ParentId` don't match the request (created outside the request's operation or on a different `TelemetryClient`/context), so it's a sibling or a separate trace.
**Explanation:** Use `StartOperation` so context is set automatically; manually set `Context.Operation.Id` and `ParentId` only when bridging.
**Code / KQL / CLI / Bicep**
```csharp
using var op = telemetry.StartOperation<DependencyTelemetry>("Legacy SOAP call");
op.Telemetry.Type = "SOAP"; op.Telemetry.Target = "legacy.contoso.com";
try { await CallAsync(); op.Telemetry.Success = true; }
catch { op.Telemetry.Success = false; throw; }
```

**Interview traps**
- Forgetting `AddSource` -> custom spans vanish.
- `TrackMetric` per request (cost + sampling).
- High-cardinality metric dimensions (userId).
- Setting role name only on some items.

**Cheat sheet**
- Activity kind decides requests vs dependencies.
- `GetMetric`/`Meter` pre-aggregate.
- Initializer (classic) = processor (OTel).
- Register every `ActivitySource`/`Meter` name.

---

## Topic 7: Sampling, cost control, daily cap, retention

**Study notes**

| Type | Where | Behavior |
|---|---|---|
| Adaptive sampling | SDK (classic ASP.NET Core default; Functions host default 20 items/s) | Auto-adjusts rate to a target items/sec |
| Fixed-rate sampling | SDK / OTel `SamplingRatio` | Keep N% by hashing `operation_Id` |
| Rate-limited (distro) | `TracesPerSecond` | Limits traces per second (verify default) |
| Ingestion sampling | Server side (App Insights) | Drops after arrival (you still pay network, not storage); don't use together with SDK sampling |

Sampling is per operation (same `operation_Id` kept or dropped together), applied to requests, dependencies, traces, exceptions, events; **not** applied to pre-aggregated metrics, availability results, or page-view (JS) unless configured. Kept items carry `itemCount = 1/rate`.

Cost levers: sampling -> log levels -> exclude noisy dependencies -> DCR transformations -> table plans -> commitment tiers -> daily cap. Retention: workspace-level default (30d), per-table override, archive to 12 years total; App Insights tables include 90 days free retention (verify).

### 7.1 [Easy] What is sampling and why use it?
**Answer:** Sampling keeps a representative fraction of telemetry to cut cost and volume while preserving statistics through `itemCount`. Related items of one operation are kept or dropped together.
**Explanation:** Use `sum(itemCount)` for counts. Sampling makes rare errors easier to miss: exclude exceptions/failed requests from sampling when possible.
**Code / KQL / CLI / Bicep**
```kusto
requests
| where timestamp > ago(1d)
| summarize kept = count(), estimated = sum(itemCount), avgRate = 100.0 / avg(itemCount) by bin(timestamp, 1h)
```

### 7.2 [Follow-up] How do I configure sampling for ASP.NET Core and Functions?
**Answer:** Classic SDK: `ApplicationInsightsServiceOptions.EnableAdaptiveSampling` or `AddApplicationInsightsTelemetryProcessor<>`; Functions: host.json `samplingSettings`. Distro: `SamplingRatio`/`TracesPerSecond` in `AzureMonitorOptions`.
**Explanation:** In Functions you can exclude types: `"excludedTypes": "Request;Exception"` so failures aren't dropped. For OTel, sampling is head-based at trace start, so a failing trace can still be dropped (no tail sampling in the distro).
**Code / KQL / CLI / Bicep**
```csharp
builder.Services.AddApplicationInsightsTelemetry(o => o.EnableAdaptiveSampling = false); // classic
builder.Services.AddOpenTelemetry().UseAzureMonitor(o => o.SamplingRatio = 0.2f);        // OTel (verify)
```

### 7.3 [Medium] What does a daily cap do and why is it dangerous?
**Answer:** The cap stops *all* ingestion for the workspace for the rest of the day once reached, so you lose the very telemetry (e.g., the outage) you need. Use cost alerts/budgets and sampling first; use the cap as a safety net.
**Explanation:** The cap applies to billable data; some tables (e.g., `Operation`, security solutions) may be exempt - verify. A cap-hit event is logged to `Operation`/`_LogOperation`. Set alerts at 80%.
**Code / KQL / CLI / Bicep**
```bash
az monitor log-analytics workspace update -g rg-mon -n law-prod --quota 5   # GB/day
```
```kusto
Usage | where TimeGenerated > ago(30d) | where IsBillable
| summarize GB = sum(Quantity) / 1024 by bin(TimeGenerated, 1d), DataType
| render columnchart
```

### 7.4 [Medium] How do I find what's costing the most?
**Answer:** Query the `Usage` table by `DataType` and use `_BilledSize` per table/dimension for deeper attribution.
**Explanation:** `Usage.Quantity` is MB. Typical culprits: `AppTraces`, `AppDependencies` (chatty SQL/HTTP), `AzureDiagnostics`, `ContainerLogV2`.
**Code / KQL / CLI / Bicep**
```kusto
AppDependencies
| where TimeGenerated > ago(1d)
| summarize MB = sum(_BilledSize) / 1e6, rows = sum(ItemCount) by DependencyType, Target
| top 15 by MB desc
```

### 7.5 [Hard] After enabling SDK adaptive sampling you also enabled ingestion sampling. What happens?
**Answer:** If the SDK already sampled, ingestion sampling is not applied to already-sampled data (it only samples items with no prior sampling info); but mixing complicates rate math. Pick one location, preferably the SDK.
**Explanation:** Ingestion sampling is a stopgap for apps you can't redeploy; it still counts data sent over network. Data cap, sampling and transformations all reduce cost differently: DCR transformation filters *before* billing; ingestion sampling is applied before storage (billing is on retained data).
**Code / KQL / CLI / Bicep**
```kusto
// Distribution of sampling factor = effective rate per item
requests | where timestamp > ago(1h) | summarize count() by itemCount
```
**Gotchas / follow-ups**
- Metric alerts on standard metrics are unaffected by sampling; log alerts based on `count()` are skewed.

### 7.6 [Tricky] An alert on `count()` of failed requests became unreliable after sampling was turned on. Fix?
**Answer:** Replace `count()` with `sum(itemCount)` and/or exclude requests/exceptions from sampling (`excludedTypes`), or alert on the pre-aggregated standard metric `requests/failed`.
**Explanation:** With 10% sampling, one kept failure represents 10; thresholds like `> 5` become noisy. Pre-aggregated metrics are computed before sampling and accurate.
**Code / KQL / CLI / Bicep**
```kusto
requests
| where timestamp > ago(10m) and success == false
| summarize failed = sum(itemCount)
| where failed > 5
```
**Gotchas / follow-ups**
- With low volumes even `sum(itemCount)` is quantized; consider longer windows.

**Interview traps**
- `count()` on sampled data.
- Believing sampling applies to metrics.
- Using a tight daily cap on prod.
- Layering SDK + ingestion sampling.

**Cheat sheet**
- Sampling per operation, weight via `itemCount`.
- Cost order: levels -> sampling -> DCR -> plans -> cap.
- `Usage` & `_BilledSize` find top spenders.
- Exclude Request/Exception from sampling if error visibility matters.

---

## Topic 8: Distributed tracing & correlation (W3C, Service Bus, Functions, Logic Apps)

**Study notes**

W3C Trace Context header: `traceparent: 00-<32-hex trace-id>-<16-hex parent-span-id>-<2-hex flags>` (+ `tracestate`).

| W3C | App Insights column |
|---|---|
| trace-id | `operation_Id` |
| span-id of this span | `id` |
| parent span-id | `operation_ParentId` |
| root span name | `operation_Name` |

```
Client --HTTP traceparent--> API (request id=A, parent=client)
  API dependency (id=B, parent=A) --Service Bus message: Diagnostic-Id/traceparent--> 
     Function trigger (request id=C, parent=B)   [same operation_Id]
        dependency (SQL) id=D parent=C
```
HTTP: automatic. Service Bus: the Azure SDK injects `Diagnostic-Id` (and `traceparent` in newer SDKs) into message `ApplicationProperties`; the consumer SDK/Functions extension reads it. Older Azure SDKs required an experimental switch `Azure.Experimental.EnableActivitySource` for Azure SDK tracing - verify current default. Logic Apps (Standard) can send to App Insights with correlation; Consumption relies on run history/tracking IDs and `x-ms-client-tracking-id` (verify trace-context behavior).

### 8.1 [Easy] What are `operation_Id` and `operation_ParentId`?
**Answer:** `operation_Id` identifies the whole end-to-end transaction (W3C trace id); `operation_ParentId` points to the `id` of the immediate parent span. Together they build the tree in the transaction view.
**Explanation:** All telemetry from one logical request across services shares `operation_Id`.
**Code / KQL / CLI / Bicep**
```kusto
union requests, dependencies, exceptions, traces
| where operation_Id == "<id>"
| project timestamp, itemType, cloud_RoleName, name, id, operation_ParentId, duration
| order by timestamp asc
```

### 8.2 [Follow-up] How is correlation done across HTTP calls between services?
**Answer:** The caller's HTTP instrumentation injects `traceparent`; the callee's ASP.NET Core instrumentation extracts it, so the callee's request shares `operation_Id`. Classic SDK also used `Request-Id` (hierarchical) - W3C is default in modern SDKs.
**Explanation:** Both sides must be instrumented. A reverse proxy/APIM/Front Door that strips or regenerates headers breaks it; APIM can forward `traceparent`.
**Code / KQL / CLI / Bicep**
```csharp
// HttpClient created via IHttpClientFactory is auto-instrumented; no manual code needed.
// For a custom transport you propagate yourself:
req.Headers.Add("traceparent", Activity.Current!.Id);
```

### 8.3 [Medium] How do I correlate a Service Bus send with the Function that processes it?
**Answer:** The sender creates a dependency span and the SDK stamps `Diagnostic-Id` (traceparent) into the message; the Functions Service Bus trigger uses it as the parent of the invocation, so both sit in the same `operation_Id`.
**Explanation:** Works for single messages. For batch triggers the invocation links to many messages (activity links) and has one parent. If you wrap messages yourself (custom envelopes) or use the raw REST client, propagate manually.
**Code / KQL / CLI / Bicep**
```csharp
// Manual propagation (sender)
var msg = new ServiceBusMessage(body);
msg.ApplicationProperties["traceparent"] = Activity.Current?.Id;
await sender.SendMessageAsync(msg);
```
```kusto
requests
| where timestamp > ago(1h) and operation_Name has "ProcessOrder"
| join kind=leftouter (dependencies | where type has "Service Bus") on operation_Id
| project timestamp, operation_Id, request=name, sbTarget=target1
```

### 8.4 [Medium] What does the Application Map show, and how are nodes named?
**Answer:** A topology graph built from dependencies, with node names from `cloud_RoleName` and edges from `dependencies.target`; it shows call rate, failure rate and duration. Wrong `cloud_RoleName` merges or splits nodes.
**Explanation:** Services must have distinct role names; instance count differences use `cloud_RoleInstance`. Set `service.name` (OTel) or initializer (classic). Functions default role name = function app name.
**Code / KQL / CLI / Bicep**
```kusto
union requests, dependencies
| where timestamp > ago(1h)
| summarize items = sum(itemCount) by cloud_RoleName, cloud_RoleInstance
```

### 8.5 [Hard] Correlation is broken across a queue: the consumer's requests have a new operation_Id. Causes?
**Answer:** (1) Producer not instrumented or the Activity is absent when sending, (2) properties lost via custom serialization/envelope or an intermediate service (Event Grid/ Logic App) that re-publishes the message, (3) consumer ignores `Diagnostic-Id` (own receive loop using `ServiceBusProcessor` without instrumentation, or older SDK without activity source), (4) telemetry from different pipelines using different id formats (Hierarchical vs W3C).
**Explanation:** Logic Apps and Event Grid are common breakers: they emit their own ids and don't forward `traceparent` into message properties unless mapped. Use tags such as a business correlation id (`OrderId`, `x-correlation-id`) in message properties and log it as dimension, as a fallback that survives any hop.
**Code / KQL / CLI / Bicep**
```kusto
// Find the consumer side by business key when trace correlation fails
traces
| where timestamp > ago(1d) and customDimensions["prop__OrderId"] == "123"
| project timestamp, operation_Id, cloud_RoleName, message
| order by timestamp asc
```
**Gotchas / follow-ups**
- Using `Activity.Current.Id` vs `TraceId`: `Id` is the full `traceparent`; `TraceId.ToString()` is just the 32 hex.
- Batch triggers: parent is only the first message's context, others use links.

### 8.6 [Tricky] Requests in the Function show up twice with different operation_Ids: one from host, one from worker. Why?
**Answer:** Isolated worker and host both create telemetry; if W3C propagation between them is broken (old extension/worker packages or mixed config), the worker starts a new trace instead of continuing the host's. Update to current packages and ensure `ConfigureFunctionsApplicationInsights` is used.
**Explanation:** Host-to-worker context passes via gRPC `TraceContext`. When disabled/old, you see host `requests` and worker `dependencies`/`traces` in separate traces.
**Code / KQL / CLI / Bicep**
```kusto
requests
| where timestamp > ago(1h) and cloud_RoleName == "func-orders"
| join kind=leftouter (traces | where timestamp > ago(1h)) on operation_Id
| summarize reqs = dcount(id), traces = count() by operation_Id
| where traces == 0
| take 20
```

**Interview traps**
- Saying correlation "just works" through every hop.
- Mixing up `id`, `operation_Id`, `operation_ParentId`.
- Ignoring batch triggers = links, not parents.
- Role names identical for all services.

**Cheat sheet**
- traceparent: version-traceid-spanid-flags.
- `operation_Id` = trace id; `operation_ParentId` = parent span id.
- Service Bus carries context in message `ApplicationProperties`.
- Add business correlation id as fallback.

---

## Topic 9: KQL fundamentals

**Study notes**

KQL is a read-only, pipe-based language: `Table | operator | operator`. Order matters: filter by time and selective columns first.

| Operator | Purpose |
|---|---|
| `where` | filter rows |
| `project` / `project-away` / `project-rename` | choose/rename columns |
| `extend` | add calculated column |
| `summarize ... by` | aggregate (`count, sum, avg, dcount, countif, percentiles, arg_max, make_set`) |
| `bin(x, 5m)` | bucket timestamps |
| `order by` / `top N by` | sort |
| `take N` | arbitrary N rows (non-deterministic) |
| `distinct` | unique values |
| `render` | chart (timechart, barchart, piechart, columnchart) |
| `ago(1h)`, `between`, `startofday` | time helpers |

String ops: `==` (case-sensitive), `=~` (case-insensitive), `has` (term match, indexed, fast), `contains` (substring, slower), `startswith`, `in`, `matches regex`.

### 9.1 [Easy] Write a query: number of requests per 5 minutes for the last hour.
**Answer:** Filter by time, `summarize` with `bin`, then `render timechart`.
**Explanation:** Always filter `timestamp` first (also set time range). `sum(itemCount)` handles sampling.
**Code / KQL / CLI / Bicep**
```kusto
requests
| where timestamp > ago(1h)
| summarize requests = sum(itemCount) by bin(timestamp, 5m)
| render timechart
```

### 9.2 [Follow-up] `has` vs `contains` vs `==`?
**Answer:** `==` is exact, case-sensitive; `=~` exact case-insensitive; `has` matches whole terms using the index (fast); `contains` matches any substring (slower, full scan).
**Explanation:** `has` splits on non-alphanumerics, so `"ServiceBus"` `has "Bus"` is false but `contains` is true. Prefer `has` and `in` for performance; `has_cs`/`contains_cs` for case-sensitive.
**Code / KQL / CLI / Bicep**
```kusto
exceptions
| where timestamp > ago(1d)
| where outerMessage has "timeout" or type in ("System.TimeoutException", "Azure.RequestFailedException")
```

### 9.3 [Medium] Top 10 slowest operations by p95 and the failure rate in one table.
**Answer:** Use `summarize` with `percentile` and `countif`, then `extend` the rate and `top`.
**Explanation:** `percentile(duration, 95)` is an approximation (t-digest style); `countif` with `itemCount` needs `sumif`.
**Code / KQL / CLI / Bicep**
```kusto
requests
| where timestamp > ago(24h)
| summarize total = sum(itemCount), failed = sumif(itemCount, success == false), p95 = percentile(duration, 95) by name
| extend failRate = round(100.0 * failed / total, 2)
| top 10 by p95 desc
```

### 9.4 [Medium] `take` vs `top` vs `order by | take`?
**Answer:** `take` returns arbitrary rows (fast, non-deterministic), `top N by X` returns the N largest by X, equivalent to `order by X | take N` but optimized.
**Explanation:** Use `take` for peeking; use `top` for rankings; use `top-nested` for hierarchical top-N.
**Code / KQL / CLI / Bicep**
```kusto
dependencies | where timestamp > ago(1h) | top 20 by duration desc | project timestamp, name, target, duration
```

### 9.5 [Hard] Why is this query slow, and how would you fix it?
`traces | where message contains "order" | where timestamp > ago(30d)`
**Answer:** The expensive `contains` full-text scan is applied before narrowing time; reorder to filter time first, use `has`, narrow by other selective columns and shorten the time range.
**Explanation:** The query planner usually pushes time predicates down, but be explicit. Avoid `*` search (`search "x"`), avoid `parse_json` on large sets before filtering, project only needed columns early, avoid `union *`. Check query stats for scanned data.
**Code / KQL / CLI / Bicep**
```kusto
traces
| where timestamp > ago(1d)
| where severityLevel >= 3
| where message has "order"
| project timestamp, message, operation_Id
```
**Gotchas / follow-ups**
- Default portal limit: 30,000 rows / 64 MB results; API limits differ (verify).
- `summarize` before `join` to cut the join input size.

### 9.6 [Tricky] `where success == false` returns nothing, but the portal shows failures. Why?
**Answer:** Often the wrong table/scope (e.g., `dependencies` vs `requests`), the time range selector overriding, or the data being under a different type: `success` is a `bool` in classic App Insights but in some tables (e.g., AzureDiagnostics) status is strings like `"Failed"`. Check `getschema` and `distinct`.
**Explanation:** Also `tostring(success)` is `"False"` in some contexts (customDimensions flags are strings: `customDimensions.IsRetry == "True"`). Use `isnotempty`/`isnull` carefully: empty string vs null differ.
**Code / KQL / CLI / Bicep**
```kusto
requests | where timestamp > ago(1d) | summarize count() by success, tostring(success)
```

**Interview traps**
- `contains` everywhere.
- `count()` vs `sum(itemCount)`.
- `==` is case-sensitive.
- `take` is not "latest N" (use `top ... by timestamp`).

**Cheat sheet**
- Filter time first; `has` > `contains`.
- `summarize ... by bin()` + `render`.
- `top N by x`; `take` is arbitrary.
- `percentile(x, 95)`; `countif/sumif`.

---

## Topic 10: KQL advanced (joins, dynamic, let, time series, materialize, cross-resource)

**Study notes**

| Feature | Key points |
|---|---|
| `join kind=` | **default is `innerunique`** (dedups left keys!). Use `inner`, `leftouter`, `leftanti`, `leftsemi`, or `lookup` (small right table broadcast). |
| `lookup` | Like leftouter for fact + small dimension; right side must be small. |
| dynamic | `customDimensions["k"]` is `dynamic`; cast with `tostring()`, `toint()`, `todouble()`. `parse_json()`/`todynamic()` for JSON strings; `mv-expand`, `bag_unpack`. |
| `let` | Named scalar/tabular values or user functions. |
| `materialize()` | Caches a tabular result reused multiple times in one query. |
| `union` | Concatenate tables (`withsource=`). |
| `arg_max/arg_min` | Row with max/min value per group (latest state). |
| `make-series` | Evenly-spaced time series (fills gaps) for `series_*` functions. |
| regex | `extract(regex, group, s)`, `extract_all`, `parse`, `matches regex`. |
| cross-resource | `app("name-or-id")`, `workspace("name-or-id")`, `adx()` |

### 10.1 [Easy] What do `let` and `extend` do and how do I reuse a value?
**Answer:** `let` defines a named value/table/function used later in the query; `extend` adds columns to rows. Together they make queries readable and parameterized.
**Explanation:** `let` statements end with `;`. Tabular `let` is lazily evaluated each time it's referenced unless wrapped in `materialize`.
**Code / KQL / CLI / Bicep**
```kusto
let window = 1h;
let slowMs = 2000;
requests
| where timestamp > ago(window) and duration > slowMs
| extend bucket = bin(timestamp, 5m)
| summarize slow = sum(itemCount) by bucket
```

### 10.2 [Follow-up] How do I read `customDimensions` and parse JSON inside a message?
**Answer:** Index by key and cast: `tostring(customDimensions["OrderId"])`. For JSON text use `parse_json(message)` or `todynamic()`, then access properties with dot notation.
**Explanation:** Keys with dots/spaces need bracket syntax. `parse_json` of invalid JSON returns null - guard with `isnotnull`. Use `bag_unpack` to expand a bag to columns; `mv-expand` for arrays.
**Code / KQL / CLI / Bicep**
```kusto
traces
| where timestamp > ago(1h) and message startswith "{"
| extend j = parse_json(message)
| where isnotnull(j)
| project timestamp, orderId = tostring(j.orderId), total = todouble(j.total)
```

### 10.3 [Medium] Which join do I use to attach the failing dependency to each failed request?
**Answer:** `inner` or `leftouter` on `operation_Id` (and optionally `id == operation_ParentId` for direct children). Prefer `summarize` the right side first to shrink it.
**Explanation:** Default `innerunique` would drop duplicates on the left and silently undercount. Alias columns: right-side duplicates are suffixed `1`.
**Code / KQL / CLI / Bicep**
```kusto
let failedReq = requests | where timestamp > ago(1h) and success == false | project operation_Id, reqName = name, reqId = id;
dependencies
| where timestamp > ago(1h) and success == false
| join kind=inner failedReq on operation_Id
| project timestamp, reqName, depType = type, target, resultCode, duration
```

### 10.4 [Medium] `arg_max`: get the latest event per instance/entity.
**Answer:** `summarize arg_max(timestamp, *) by key` returns the full row with the greatest timestamp per key.
**Explanation:** Useful for "latest heartbeat per instance" or "last status per order". For only specific columns: `arg_max(timestamp, status, name)`.
**Code / KQL / CLI / Bicep**
```kusto
performanceCounters
| where timestamp > ago(30m) and name == "% Processor Time"
| summarize arg_max(timestamp, value) by cloud_RoleInstance
```

### 10.5 [Hard] Detect anomalies in request rate.
**Answer:** Use `make-series` to build a gap-filled series, then `series_decompose_anomalies` which flags points outside the seasonal/trend baseline.
**Explanation:** Parameters: threshold (default 1.5), seasonality (-1 = autodetect), trend method (`linefit`/`avg`/`none`). Needs enough history (days to weeks). Render anomalies with `render anomalychart`.
**Code / KQL / CLI / Bicep**
```kusto
requests
| where timestamp > ago(14d)
| make-series reqs = sum(itemCount) default=0 on timestamp from ago(14d) to now() step 1h
| extend (flag, score, baseline) = series_decompose_anomalies(reqs, 1.5, -1, 'linefit')
| render anomalychart with (anomalycolumns=flag)
```
**Gotchas / follow-ups**
- Missing buckets without `default=0` become nulls (bad for decomposition).
- Alerts on this need the query result to be a table (not chart); use `mv-expand` to rows and filter `flag != 0`.

### 10.6 [Hard] Use `materialize`, functions and `union` together to compare two time windows.
**Answer:** Materialize the base set once, then derive two windows and join/union them to compare. Saves cost and keeps both views consistent.
**Explanation:** `materialize` caches within the query, limited to 5 GB. Save frequently used logic as a stored function (query-saved "function" in the workspace) and call it by name.
**Code / KQL / CLI / Bicep**
```kusto
let base = materialize(requests | where timestamp > ago(2h) | project timestamp, name, success, duration, itemCount);
let now_ = base | where timestamp > ago(1h) | summarize p95_now = percentile(duration, 95), n_now = sum(itemCount) by name;
let prev = base | where timestamp <= ago(1h) | summarize p95_prev = percentile(duration, 95), n_prev = sum(itemCount) by name;
now_ | join kind=leftouter prev on name
| extend delta = p95_now - p95_prev
| project name, p95_prev, p95_now, delta, n_now
| order by delta desc
```

### 10.7 [Tricky] Cross-resource queries return fewer rows than expected / are slow. Why?
**Answer:** Each `app()`/`workspace()` call fans out to a separate resource; missing permissions silently exclude resources (or fail), limits (e.g., max 100 resources - verify) apply, and one slow resource slows everything. Use consistent names (resource ID is unambiguous), and filter inside each branch.
**Explanation:** Resource names can be ambiguous if duplicated across subscriptions; use the full resource ID. Use `workspace("<id>").AppRequests` in workspace scope; `app("<name>").requests` in AI scope; schemas differ per scope.
**Code / KQL / CLI / Bicep**
```kusto
union
  (app("appi-orders").requests    | where timestamp > ago(1h) | extend src = "orders"),
  (app("appi-billing").requests   | where timestamp > ago(1h) | extend src = "billing")
| summarize failed = sumif(itemCount, success == false), total = sum(itemCount) by src
```
**Gotchas / follow-ups**
- Cross-resource queries in *alert rules* need permissions on every resource (verify scope rules).
- `join` order: put the smaller table on the right.

**Interview traps**
- Default `join` = `innerunique`.
- Casting: dynamic values compared without `tostring`.
- `parse_json` on huge sets before filtering.
- `materialize` of large results (limit).

**Cheat sheet**
- `let` + `materialize`; `join kind=` explicit.
- `arg_max(ts, *) by key` = latest row.
- `make-series ... default=0` then `series_decompose_anomalies`.
- `app()`/`workspace()` for cross-resource; use IDs.
- `extract(regex, 1, s)`, `parse`, `bag_unpack`, `mv-expand`.

---

## Topic 11: Practical KQL query library (25 queries)

**Study notes**

Queries use the classic App Insights schema; for the workspace schema map columns using Topic 3 (e.g., `requests`->`AppRequests`, `timestamp`->`TimeGenerated`, `customDimensions`->`Properties`). Resource-log queries (Key Vault, Storage, Logic Apps, Service Bus) run **in the Log Analytics workspace** that receives the diagnostic settings (Topic 12). Column names for resource logs can vary by table mode (AzureDiagnostics vs resource-specific) - verify with `getschema`.

### 11.1 [Easy] Core health queries: failed requests, exceptions by type, slowest dependencies.
**Answer:** Three staples: failures grouped by operation/result code, exceptions ranked by type/problemId, dependencies ranked by duration.
**Explanation:** Always weight with `itemCount` and look at a short window first.
**Code / KQL / CLI / Bicep**
```kusto
// Q1 Failed requests by operation and code
requests | where timestamp > ago(1h) and success == false
| summarize failed = sum(itemCount) by operation_Name, resultCode | order by failed desc

// Q2 Exceptions by type with sample message
exceptions | where timestamp > ago(24h)
| summarize count = sum(itemCount), sample = any(outerMessage), last = max(timestamp) by type, problemId
| order by count desc

// Q3 Slowest dependencies (p95) by target
dependencies | where timestamp > ago(1h)
| summarize calls = sum(itemCount), p95 = percentile(duration, 95), failed = sumif(itemCount, success == false) by type, target
| top 15 by p95 desc
```

### 11.2 [Follow-up] SQL latency, Redis timeouts, percentile trends and error rate per release.
**Answer:** Filter `dependencies` by type for SQL/Redis; for trends use `percentiles` over `bin`; for releases group by `application_Version`.
**Explanation:** `type` strings differ by instrumentation: confirm via `distinct type` (SQL may be "SQL"; Redis may be "Redis").
**Code / KQL / CLI / Bicep**
```kusto
// Q4 SQL dependency latency trend
dependencies | where timestamp > ago(24h) and type in ("SQL", "mssql")
| summarize p50=percentile(duration,50), p95=percentile(duration,95), p99=percentile(duration,99) by bin(timestamp, 15m), target
| render timechart

// Q5 Redis dependency failures and slow calls
dependencies | where timestamp > ago(24h) and type =~ "Redis"
| summarize calls = sum(itemCount), failed = sumif(itemCount, success == false), p99 = percentile(duration, 99) by bin(timestamp, 15m), target

// Q6 Redis timeouts in exceptions (StackExchange.Redis)
exceptions | where timestamp > ago(24h) and (type has "RedisTimeoutException" or outerMessage has "Timeout performing")
| summarize n = sum(itemCount) by bin(timestamp, 15m), cloud_RoleInstance

// Q7 p95/p99 latency trend for requests
requests | where timestamp > ago(7d)
| summarize p95=percentile(duration,95), p99=percentile(duration,99) by bin(timestamp, 1h) | render timechart

// Q8 Error rate per release
requests | where timestamp > ago(3d)
| summarize total = sum(itemCount), failed = sumif(itemCount, success == false) by application_Version
| extend errorRatePct = round(100.0 * failed / total, 2) | order by application_Version desc
```

### 11.3 [Medium] Service Bus lock lost / DLQ and Function failures & cold starts.
**Answer:** Lock-lost shows up as `ServiceBusException` (reason MessageLockLost) in `exceptions`; DLQ depth is a metric (`DeadletteredMessages`) in `AzureMetrics`; Function failures are `requests` with `success == false`; cold starts are inferred, not directly logged.
**Explanation:** Lock-lost means processing exceeded the lock duration (or the connection dropped) and the message will be redelivered, increasing `DeliveryCount` toward `MaxDeliveryCount` then DLQ. Cold start: first request on a new `cloud_RoleInstance` after idle shows high duration - a heuristic; confirm with host-start traces (verify exact message text).
**Code / KQL / CLI / Bicep**
```kusto
// Q9 Service Bus lock lost / messaging exceptions
exceptions | where timestamp > ago(24h)
| where type has "ServiceBusException" or outerMessage has_any ("lock supplied is invalid", "MessageLockLost", "lock has expired")
| summarize n = sum(itemCount), sample = any(outerMessage) by bin(timestamp, 1h), operation_Name

// Q10 Service Bus dependency failures (sender side)
dependencies | where timestamp > ago(24h) and (type has "Service Bus" or target has "servicebus.windows.net") and success == false
| summarize n = sum(itemCount) by target, name, resultCode

// Q11 DLQ depth from platform metrics (workspace; needs AllMetrics diagnostic setting)
AzureMetrics | where TimeGenerated > ago(24h) and ResourceProvider == "MICROSOFT.SERVICEBUS" and MetricName == "DeadletteredMessages"
| summarize dlq = max(Maximum) by bin(TimeGenerated, 15m), Resource | render timechart

// Q12 Function failures by function
requests | where timestamp > ago(24h) and cloud_RoleName == "func-orders"
| summarize total = sum(itemCount), failed = sumif(itemCount, success == false), p95 = percentile(duration, 95) by operation_Name
| extend failPct = round(100.0 * failed / total, 2) | order by failed desc

// Q13 Cold-start heuristic: first request per instance
let firstPerInstance = requests | where timestamp > ago(24h) and cloud_RoleName == "func-orders"
  | summarize first = min(timestamp) by cloud_RoleInstance;
requests | where timestamp > ago(24h) and cloud_RoleName == "func-orders"
| join kind=inner firstPerInstance on cloud_RoleInstance
| where timestamp == first
| project timestamp, cloud_RoleInstance, operation_Name, coldStartCandidateMs = duration
| order by coldStartCandidateMs desc
```

### 11.4 [Medium] Key Vault 403s and Blob 403/404.
**Answer:** Key Vault audit logs go to `AzureDiagnostics` (`Category == "AuditEvent"`) or resource-specific `AZKVAuditLogs`; blob access logs to `StorageBlobLogs`. Filter on HTTP status and group by operation and caller.
**Explanation:** KV 403 = RBAC/access-policy/firewall denial (identity lacks `Key Vault Secrets User` or network ACL). Blob 403 = auth/RBAC/SAS/firewall; 404 = missing blob/container. `AuthenticationType` (OAuth/SAS/AccountKey) pinpoints the path. Both need diagnostic settings enabled first.
**Code / KQL / CLI / Bicep**
```kusto
// Q14 Key Vault denied requests
AzureDiagnostics | where TimeGenerated > ago(24h) and ResourceProvider == "MICROSOFT.KEYVAULT" and Category == "AuditEvent"
| where httpStatusCode_d in (401, 403)
| summarize n = count() by OperationName, CallerIPAddress, ResultSignature, Resource

// Q15 Key Vault throttling (429) - also high volume of Get Secret
AzureDiagnostics | where TimeGenerated > ago(24h) and ResourceProvider == "MICROSOFT.KEYVAULT" and httpStatusCode_d == 429
| summarize n = count() by bin(TimeGenerated, 5m), OperationName

// Q16 Blob 403/404 by operation and auth type
StorageBlobLogs | where TimeGenerated > ago(24h) and StatusCode in (403, 404)
| summarize n = count() by StatusCode, StatusText, OperationName, AuthenticationType, AccountName

// Q17 Which callers get 403 on blobs
StorageBlobLogs | where TimeGenerated > ago(24h) and StatusCode == 403
| summarize n = count() by CallerIpAddress, RequesterObjectId, Uri | top 20 by n desc
```

### 11.5 [Hard] Logic App failures, user flows and failing-operation breakdown.
**Answer:** Logic Apps (Consumption) with diagnostics go to `AzureDiagnostics` (`WorkflowRuntime`) with run/action completion events (or resource-specific table - verify). User flows use `pageViews`/`customEvents` ordered by session.
**Explanation:** `status_s == "Failed"` on `workflowRunCompleted`; action-level shows which step failed. For Logic Apps Standard, telemetry also lands in App Insights `requests`/`traces`.
**Code / KQL / CLI / Bicep**
```kusto
// Q18 Failed Logic App runs
AzureDiagnostics | where TimeGenerated > ago(24h) and ResourceProvider == "MICROSOFT.LOGIC"
| where OperationName == "Microsoft.Logic/workflows/workflowRunCompleted" and status_s == "Failed"
| project TimeGenerated, workflow = resource_workflowName_s, runId = resource_runId_s, code_s
| order by TimeGenerated desc

// Q19 Failing actions inside runs
AzureDiagnostics | where TimeGenerated > ago(24h) and ResourceProvider == "MICROSOFT.LOGIC"
| where OperationName == "Microsoft.Logic/workflows/workflowActionCompleted" and status_s == "Failed"
| summarize n = count() by resource_workflowName_s, resource_actionName_s, code_s

// Q20 Next-page user flow (per session)
pageViews | where timestamp > ago(1d) and isnotempty(session_Id)
| order by session_Id asc, timestamp asc
| serialize
| extend nextPage = iff(next(session_Id) == session_Id, next(name), "(exit)")
| summarize hits = count() by name, nextPage | top 30 by hits desc

// Q21 Failing operation with the dependency responsible
requests | where timestamp > ago(1h) and success == false | project operation_Id, operation_Name
| join kind=leftouter (dependencies | where timestamp > ago(1h) and success == false | project operation_Id, depType = type, target, depCode = resultCode) on operation_Id
| summarize n = count() by operation_Name, depType, target, depCode | order by n desc
```

### 11.6 [Tricky] Production forensic queries: slow-request breakdown, retry storms, orphans, availability.
**Answer:** Attribute slow requests to dependency time vs own time, spot retry storms through repeated dependency calls per operation, find operations without a request root (broken correlation) and check availability failures by location.
**Explanation:** Dependency time is not additive when calls run in parallel; treat it as an estimate. Retry storms hide behind eventual success; count calls per `operation_Id`.
**Code / KQL / CLI / Bicep**
```kusto
// Q22 Share of slow request time spent in dependencies (approximate)
let slow = requests | where timestamp > ago(1h) and duration > 3000 | project operation_Id, reqMs = duration, operation_Name;
dependencies | where timestamp > ago(1h)
| summarize depMs = sum(duration), depCalls = sum(itemCount) by operation_Id
| join kind=inner slow on operation_Id
| extend depShare = round(depMs / reqMs, 2)
| project operation_Name, reqMs, depMs, depCalls, depShare | order by reqMs desc

// Q23 Retry storm: many calls to same target in one operation
dependencies | where timestamp > ago(1h)
| summarize calls = sum(itemCount), fails = sumif(itemCount, success == false) by operation_Id, target
| where calls >= 5 and fails > 0 | order by calls desc

// Q24 Orphaned telemetry: traces with no request in the same operation (broken correlation / sampling)
traces | where timestamp > ago(1h)
| summarize t = count() by operation_Id
| join kind=leftanti (requests | where timestamp > ago(1h) | distinct operation_Id) on operation_Id
| top 20 by t desc

// Q25 Availability failures by location
availabilityResults | where timestamp > ago(24h)
| summarize runs = sum(itemCount), failed = sumif(itemCount, success == 0) by name, location
| extend failPct = round(100.0 * failed / runs, 2) | order by failPct desc

// Q26 Exceptions that started after a release (new problemIds)
let old = exceptions | where timestamp between (ago(7d) .. ago(1d)) | distinct problemId;
exceptions | where timestamp > ago(1d) | join kind=leftanti old on problemId
| summarize n = sum(itemCount), first = min(timestamp), ver = any(application_Version) by problemId, type
```
**Gotchas / follow-ups**
- `availabilityResults.success` is an int (1/0) in classic schema - verify with `getschema`.
- Orphan traces can legitimately come from sampling of requests but not traces if sampling differs.

**Interview traps**
- Using `count()` instead of `sum(itemCount)` in these queries.
- Running resource-log queries in the AI blade (tables absent).
- Hardcoding `type` strings without checking.
- Expecting dependency time to sum to request time.

**Cheat sheet**
- App telemetry: AI blade; resource logs: workspace.
- KV: AzureDiagnostics AuditEvent; Blob: StorageBlobLogs; Logic: AzureDiagnostics WorkflowRuntime.
- Cold start = heuristic; lock lost = ServiceBusException.
- Release analysis: `application_Version`.
- `leftanti` finds "new/missing" sets.

---

## Topic 12: Diagnostic settings & resource logs routing

**Study notes**

Three log types in Azure Monitor: **Platform metrics** (automatic), **Activity Log** (subscription control-plane events; export via diagnostic setting), **Resource logs** (data-plane; *off by default*, need a diagnostic setting).

```
Resource --diagnostic setting--> Log Analytics workspace  (query, alerts)
                              |-> Storage account          (cheap long-term archive, JSON blobs)
                              |-> Event Hub                (stream to SIEM/3rd-party)
                              `-> Partner solution
```
Up to 5 diagnostic settings per resource. Select **categories** (e.g., `AuditEvent`, `StorageRead`) or **category groups** (`allLogs`, `audit`). Destination table mode for Log Analytics: **Azure diagnostics** (`AzureDiagnostics` wide table, `_s/_d/_g` suffixes) or **Resource specific** (dedicated tables like `StorageBlobLogs`, `AZKVAuditLogs`) - resource-specific is preferred (better schema, per-table retention/plan, Basic logs eligible). Enforce at scale with Azure Policy (DeployIfNotExists).

### 12.1 [Easy] Why can't I see Key Vault/Storage logs in Log Analytics?
**Answer:** Resource logs are not collected until you create a diagnostic setting that routes the chosen categories to a workspace. Platform metrics are collected automatically; logs are opt-in.
**Explanation:** Storage has per-service settings (blob/queue/table/file) on child resource `blobServices/default`. Ingestion latency is typically a few minutes.
**Code / KQL / CLI / Bicep**
```bash
az monitor diagnostic-settings create --name to-law \
  --resource <keyvault-resource-id> --workspace <law-resource-id> \
  --logs '[{"categoryGroup":"audit","enabled":true}]' \
  --metrics '[{"category":"AllMetrics","enabled":true}]'
```

### 12.2 [Follow-up] Which destination for which need?
**Answer:** Log Analytics for query/alerts, Storage for cheap compliance archive, Event Hub for streaming to a SIEM or external tool. You can use several at once.
**Explanation:** Storage destination doesn't support KQL (use ADX/external tables or Log Analytics archive instead). Event Hub requires an authorization rule/or RBAC; Event Hub & storage must be in the same region as the resource (for most resources).
**Code / KQL / CLI / Bicep**
```bicep
resource diag 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = {
  name: 'kv-diag'
  scope: kv
  properties: {
    workspaceId: law.id
    storageAccountId: sa.id
    logAnalyticsDestinationType: 'Dedicated'   // resource-specific tables
    logs: [ { categoryGroup: 'audit', enabled: true } ]
    metrics: [ { category: 'AllMetrics', enabled: true } ]
  }
}
```

### 12.3 [Medium] AzureDiagnostics vs resource-specific tables?
**Answer:** AzureDiagnostics is one shared wide table with suffixed columns (`status_s`, `httpStatusCode_d`) and a ~500 column limit; resource-specific tables have typed columns and individual retention/plans. Prefer resource-specific where the resource supports it.
**Explanation:** Changing mode changes where *new* data lands; old queries on `AzureDiagnostics` will stop returning new data - update alerts/workbooks. Not all providers support dedicated mode (e.g., Logic Apps resource-specific tables exist but verify current support).
**Code / KQL / CLI / Bicep**
```kusto
// Which resource types are writing to AzureDiagnostics?
AzureDiagnostics | where TimeGenerated > ago(1d)
| summarize rows = count(), mb = sum(_BilledSize) / 1e6 by ResourceProvider, Category | order by mb desc
```

### 12.4 [Medium] What are category groups and why use them?
**Answer:** A category group (`allLogs`, `audit`) automatically includes current and future categories of that kind, so new categories are collected without updating the setting. `audit` limits to security/audit logs, saving cost.
**Explanation:** Explicit categories are precise; groups are future-proof. Use `allLogs` carefully on chatty resources.
**Code / KQL / CLI / Bicep**
```bash
az monitor diagnostic-settings categories list --resource <resource-id>
az monitor diagnostic-settings show --name to-law --resource <resource-id>
```

### 12.5 [Hard] Diagnostic setting created but table empty. Troubleshoot.
**Answer:** Check the category enabled, region constraints, workspace permissions, a filtered/denied Policy, wait for first ingestion, correct table mode, the daily cap, and that the resource actually generated the event (e.g., no reads on blob -> no `StorageRead`).
**Explanation:** Verify using Activity Log for the setting creation and `_LogOperation`/`Usage`. For Storage, the service-level setting must be on `blobServices/default`, not the account. Confirm traffic occurs. Cross-subscription workspaces need Microsoft.Insights provider registered.
**Code / KQL / CLI / Bicep**
```kusto
Usage | where TimeGenerated > ago(1d) | summarize MB = sum(Quantity) by DataType | order by MB desc
```
**Gotchas / follow-ups**
- Firewall-enabled storage in Event Hub/Storage destinations needs "trusted Microsoft services".
- Activity Log legacy "log profiles" are deprecated; use diagnostic settings.

**Interview traps**
- "Metrics are logs" - resource logs are opt-in.
- Not recognizing `AzureDiagnostics` suffix columns.
- Forgetting Storage per-service diagnostics.

**Cheat sheet**
- Destinations: LAW / Storage / Event Hub / partner.
- Prefer resource-specific tables + category groups.
- Max 5 settings per resource; use Policy at scale.
- Activity Log is also routed via diagnostic setting.

---

## Topic 13: Alerts - metric vs log, action groups, processing rules, alert fatigue

**Study notes**

| Alert type | Signal | Latency | Notes |
|---|---|---|---|
| Metric (static) | platform/custom metric | ~1-5 min | Threshold on aggregation over window |
| Metric (dynamic) | same | ~1-5 min | ML baseline; needs history (days) to learn |
| Log search (scheduled query) | KQL on workspace/AI | + ingestion latency | Aggregation over rows or count of results; dimensions split alerts; frequency >= 1 min |
| Activity log | control-plane events | ~min | Resource health, service health |
| Smart detection | AI anomaly | - | Behavior evolved/changed - verify current status |
| Prometheus | PromQL | - | AKS/managed Prometheus |

Action groups: email/SMS/voice/push, webhook (incl. secure), Logic App, Azure Function, Event Hub, ITSM, Automation runbook. Use the **common alert schema** for uniform payloads. **Alert processing rules** (formerly action rules) add action groups, suppress notifications (maintenance windows) or apply to scope/schedule without editing rules.

### 13.1 [Easy] Metric alert vs log alert?
**Answer:** Metric alerts evaluate fast, cheap numeric metrics; log alerts run KQL and can express complex conditions but have ingestion/evaluation delay and cost per rule. Use metrics when possible.
**Explanation:** Metric: Service Bus `DeadletteredMessages > 0`. Log: "5xx rate > 2% for tier X over 10 min computed from `requests`". Standard App Insights metrics (e.g., `requests/failed`) are also usable as metric alerts.
**Code / KQL / CLI / Bicep**
```bash
az monitor metrics alert create -g rg-mon -n func-errors \
  --scopes <appinsights-id> --condition "count requests/failed > 5" \
  --window-size 5m --evaluation-frequency 1m --action ag-oncall
```

### 13.2 [Follow-up] Static vs dynamic thresholds?
**Answer:** Static uses a fixed value; dynamic learns seasonality/trend and alerts on deviation (sensitivity low/medium/high, failing periods N of M). Use dynamic for traffic-shaped metrics, static for hard limits (DLQ > 0).
**Explanation:** Dynamic needs at least ~3 days of history (verify), can mask slow degradations, and isn't suited for metrics near zero.
**Code / KQL / CLI / Bicep**
```bash
az monitor metrics alert create -g rg-mon -n req-anomaly --scopes <appi-id> \
  --condition "avg requests/duration > dynamic medium 3 of 4 since 2026-01-01T00:00:00Z" \
  --window-size 5m --evaluation-frequency 5m --action ag-oncall
```

### 13.3 [Medium] Build a log search alert for failure rate and avoid noise.
**Answer:** Compute the rate in KQL over the window, return a row only when it breaches, set evaluation frequency/window, use "number of violations" (failing periods) and auto-mitigate, split by dimension (e.g., `cloud_RoleName`).
**Explanation:** Sample-aware (`sum(itemCount)`), minimum volume guard to avoid 1/1 = 100% noise, and window >= frequency. Consider ingestion delay: don't use `now()` boundaries too tight.
**Code / KQL / CLI / Bicep**
```kusto
requests
| where timestamp > ago(15m)
| summarize total = sum(itemCount), failed = sumif(itemCount, success == false) by cloud_RoleName
| where total > 50
| extend failPct = 100.0 * failed / total
| where failPct > 5
```
```bicep
resource rule 'Microsoft.Insights/scheduledQueryRules@2023-03-15-preview' = {
  name: 'high-failure-rate'
  location: resourceGroup().location
  properties: {
    displayName: 'High failure rate'
    severity: 2
    enabled: true
    scopes: [ appi.id ]
    evaluationFrequency: 'PT5M'
    windowSize: 'PT15M'
    criteria: { allOf: [ {
      query: 'requests | where timestamp > ago(15m) | summarize total=sum(itemCount), failed=sumif(itemCount, success==false) by cloud_RoleName | where total > 50 | where 100.0*failed/total > 5'
      timeAggregation: 'Count'
      operator: 'GreaterThan'
      threshold: 0
      failingPeriods: { numberOfEvaluationPeriods: 1, minFailingPeriodsToAlert: 1 }
      dimensions: [ { name: 'cloud_RoleName', operator: 'Include', values: [ '*' ] } ]
    } ] }
    actions: { actionGroups: [ ag.id ] }
  }
}
```

### 13.4 [Medium] What are alert processing rules used for?
**Answer:** They modify alert behavior at scale without editing rules: suppress notifications during maintenance, route alerts from a scope to extra action groups, with schedules and filters (severity, resource, etc.).
**Explanation:** Suppression still creates alert instances (they're recorded) but no notification/action fires. Good for planned deployments.
**Code / KQL / CLI / Bicep**
```bash
az monitor alert-processing-rule create -g rg-mon -n maint-window \
  --scopes /subscriptions/<sub>/resourceGroups/rg-app --rule-type RemoveAllActionGroups \
  --schedule-start-datetime "2026-11-01 22:00:00" --schedule-end-datetime "2026-11-02 02:00:00" \
  --schedule-time-zone "UTC"
```

### 13.5 [Hard] Alert fatigue: how do you design alerts people trust?
**Answer:** Alert on symptoms tied to SLOs (user-facing error rate/latency burn), not on every cause; page only on actionable issues, route others to tickets/dashboards; use multi-window burn-rate alerts, dynamic/aggregated dimensions, severity tiers and processing rules; review alert volume regularly.
**Explanation:** Every page should have a runbook and require human action. Multi-window burn rate (e.g., 14.4x over 1h AND 5m for page; 6x over 6h AND 30m ticket for a 30-day 99.9% SLO - Google SRE workbook style) catches fast and slow burns with low noise.
**Code / KQL / CLI / Bicep**
```kusto
// Burn rate over 1h for 99.9% SLO (budget 0.1%)
requests | where timestamp > ago(1h)
| summarize total = sum(itemCount), bad = sumif(itemCount, success == false)
| extend errRate = todouble(bad) / total, burnRate = (todouble(bad) / total) / 0.001
| where burnRate > 14.4 and total > 100
```
**Gotchas / follow-ups**
- Log alerts evaluate one window; implement the short-window AND via two criteria/rules or a single query computing both windows.

### 13.6 [Tricky] An alert never fired during an outage. Why?
**Answer:** Typical causes: log alert missed because of ingestion latency or sampling; query returned zero rows when the service was down (no data = no breach: use "no data" or availability alerts); threshold/dimension mismatch; the action group was throttled/disabled/wrong; rule scope or processing rule suppressed notifications; alert was in "Fired" state (stateful) and didn't re-notify.
**Explanation:** Dead services emit no telemetry, so `failed` metrics are absent. Use availability tests, heartbeat absence queries, or metric alerts with "treat missing as breach" (missing data handling for metric alerts/`dynamic`), and test action groups. Check the alert history/ Alerts blade and Activity Log for action group failures.
**Code / KQL / CLI / Bicep**
```kusto
// Absence alert: no requests in 10 minutes
requests | where timestamp > ago(10m)
| summarize n = sum(itemCount)
| where n == 0
// Caveat: an empty result set (no rows) does not match; use datatable + union or evaluate with "no results" semantics - verify rule options
```
**Gotchas / follow-ups**
- Email/SMS action groups have rate limits; webhook endpoints must respond promptly.
- Alert rule "Resolved" notifications can be disabled (auto-mitigate).

**Interview traps**
- Log alert on `count()` of sampled data.
- Assuming missing data triggers a failure alert.
- Alerting on CPU instead of user symptoms.
- Editing 50 rules for maintenance instead of one processing rule.

**Cheat sheet**
- Metric = fast/cheap; log = flexible/delayed/cost per rule.
- Dynamic thresholds need history.
- Action group -> common alert schema.
- Processing rules for suppression/routing.
- SLO burn-rate multi-window alerts reduce noise.

---

## Topic 14: Availability tests, Live Metrics, Profiler, Snapshot Debugger

**Study notes**

| Tool | Purpose | Notes |
|---|---|---|
| Standard availability test | HTTP GET/HEAD/POST from multiple regions; checks status code, content, TLS cert, headers | Replaces classic URL ping tests (retirement - verify) |
| Custom `TrackAvailability` test | Your own code (e.g., Function) | For private/complex checks |
| Live Metrics | Near-real-time (~1s) view of requests, failures, CPU, logs; data not stored | Great during deployments; uses a separate live endpoint |
| Application Insights Profiler | Samples stack traces to find hot code paths in ASP.NET/ASP.NET Core | On-demand / scheduled |
| Snapshot Debugger | Captures debug snapshot on first-chance exception (.NET) | Needs runtime support; verify for your host/OS |

### 14.1 [Easy] What is an availability (standard) test?
**Answer:** A synthetic probe run from several Azure regions on a schedule that calls your endpoint and records pass/fail and duration into `availabilityResults`, with alert rules to notify on multi-location failure.
**Explanation:** Can validate status code, response content, SSL expiry (alert days before). Private endpoints aren't reachable from public test locations - use `TrackAvailability` from inside the VNet.
**Code / KQL / CLI / Bicep**
```bicep
resource avail 'Microsoft.Insights/webtests@2022-06-15' = {
  name: 'avail-orders-health'
  location: resourceGroup().location
  tags: { 'hidden-link:${appi.id}': 'Resource' }
  kind: 'standard'
  properties: {
    SyntheticMonitorId: 'avail-orders-health'
    Name: 'orders health'
    Enabled: true
    Frequency: 300
    Timeout: 30
    Kind: 'standard'
    RetryEnabled: true
    Locations: [ { Id: 'emea-nl-ams-azr' }, { Id: 'us-va-ash-azr' } ]
    Request: { RequestUrl: 'https://orders.contoso.com/health', HttpVerb: 'GET' }
    ValidationRules: { ExpectedHttpStatusCode: 200, SSLCheck: true, SSLCertRemainingLifetimeCheck: 7 }
  }
}
```

### 14.2 [Follow-up] When do I use Live Metrics?
**Answer:** During deployments/incidents for second-level feedback on request rate, failures, dependency health, live logs and per-instance CPU/memory without waiting for ingestion latency.
**Explanation:** It's not stored and not sampled the same way; with Entra auth it needs permissions (verify). Distro enables it by default (`EnableLiveMetrics`, verify).
**Code / KQL / CLI / Bicep**
```csharp
builder.Services.AddOpenTelemetry().UseAzureMonitor(o => o.EnableLiveMetrics = true); // verify option name
```

### 14.3 [Medium] Profiler vs Snapshot Debugger: what problem does each solve?
**Answer:** Profiler answers "why is this request slow" by showing sampled CPU/wall-time hot paths; Snapshot Debugger answers "why did this exception happen" by capturing locals/call stack at the throw site.
**Explanation:** Both add overhead and require supported hosting (App Service Windows/Linux, AKS with agents; Functions support limited - verify). Snapshots may contain PII; secure access via RBAC (Snapshot Debugger role).
**Code / KQL / CLI / Bicep**
```kusto
// Request traces with profiler data attached are flagged in the end-to-end view; find slow candidates
requests | where timestamp > ago(1d) and duration > 5000 | summarize n = sum(itemCount) by operation_Name
```

### 14.4 [Hard] Availability test reports failure in 1 of 5 regions only. Diagnose.
**Answer:** Likely a regional problem (CDN/Front Door POP, DNS, WAF geo-rule, rate limiting of test IPs, or a transient network path). Compare per-location results and response timings; allow-list test IPs/service tag `ApplicationInsightsAvailability`.
**Explanation:** Standard tests fail when status code/content/TLS rule fails; check the *failed test details* (DNS, connect, TLS, first byte). WAF/bot rules commonly block synthetic UA. Alert rule "failed locations >= N" avoids single-location flaps.
**Code / KQL / CLI / Bicep**
```kusto
availabilityResults | where timestamp > ago(6h)
| summarize avgMs = avg(duration), runs = sum(itemCount), failed = sumif(itemCount, success == 0) by location, bin(timestamp, 30m)
| render timechart
```
**Gotchas / follow-ups**
- Availability results appear with `operation_Id` correlation only for TrackAvailability code paths.
- App behind private endpoint -> test fails 100% from public probes.

**Interview traps**
- Believing Live Metrics data is stored/queryable.
- Pinging `/` that doesn't test dependencies (use a deep health endpoint).
- Using classic ping tests (retiring).

**Cheat sheet**
- Standard test = multi-region HTTP + SSL checks -> `availabilityResults`.
- Private apps: `TrackAvailability` from inside.
- Live Metrics = ephemeral, ~1s.
- Profiler = slow code; Snapshot = exception state.

---

## Topic 15: Workbooks, dashboards, Grafana, Container/VM insights

**Study notes**

| Tool | Best for | Notes |
|---|---|---|
| Azure Dashboard | Pinned tiles across resources | Simple, limited interactivity |
| Workbooks | Interactive reports with parameters, KQL, metrics, ARG, ARM | Shareable, templated (ARM/Bicep/Gallery) |
| Azure Managed Grafana | Multi-source dashboards (Azure Monitor, Prometheus, ADX) | Managed service; Entra auth |
| Insights (curated) | Container insights (AKS), VM insights, App Insights, Cosmos/Storage/KV insights | Prebuilt workbooks + collection config |

Container insights: Azure Monitor Agent (AMA) DaemonSet on AKS, DCR-driven, logs to `ContainerLogV2`, `KubePodInventory`, `InsightsMetrics`; pairs with **managed Prometheus** for metrics. VM insights: AMA + dependency agent for performance + Map (`VMComputer`, `VMConnection`, `InsightsMetrics`).

### 15.1 [Easy] Dashboard vs workbook?
**Answer:** Dashboards are static pinned tiles; workbooks are interactive documents with parameters, drill-down and mixed data sources. Use workbooks for investigation and reporting.
**Explanation:** Workbooks can be saved per resource/shared/ARM-deployed; support text, queries, metrics, links, conditional visibility, and RBAC through the underlying resources.
**Code / KQL / CLI / Bicep**
```kusto
// Workbook query step using a time range parameter {TimeRange}
requests | where $__timeFilter(timestamp)   // Grafana macro; in workbooks use {TimeRange}
```

### 15.2 [Follow-up] When choose Managed Grafana?
**Answer:** When you need a single pane across Azure Monitor, Prometheus and non-Azure data sources, or teams are already on Grafana. It integrates with Entra and Azure Monitor data source natively.
**Explanation:** Grafana queries Azure Monitor Logs with KQL and Metrics with metric queries; alerting can live in Grafana or Azure Monitor (prefer one source of truth).
**Code / KQL / CLI / Bicep**
```bash
az grafana create -g rg-mon -n graf-prod --location westeurope
az role assignment create --assignee <grafana-mi> --role "Monitoring Reader" --scope /subscriptions/<sub>
```

### 15.3 [Medium] What does Container insights collect and why is cost a concern?
**Answer:** Container stdout/stderr (`ContainerLogV2`), inventory/events and node/pod performance. Cost comes mostly from verbose container logs; control with DCR (namespace filters, ContainerLogV2 Basic plan, disabling stdout from noisy namespaces).
**Explanation:** Use DCR "streams" and namespace filtering; ContainerLogV2 supports the Basic logs plan. App telemetry should still go through App Insights/OTel, not console logs.
**Code / KQL / CLI / Bicep**
```kusto
ContainerLogV2 | where TimeGenerated > ago(1h)
| summarize MB = sum(_BilledSize) / 1e6 by PodNamespace, ContainerName | top 10 by MB desc
```

### 15.4 [Hard] You need a self-service "service health" view for many teams. Design?
**Answer:** A parameterized workbook (subscription/app/role/time range parameters) deployed via Bicep, standard KQL on RED metrics, linked from alerts; RBAC through resource permissions. Optionally Grafana for cross-tool.
**Explanation:** Use workbook parameters with `cloud_RoleName` multi-select, query steps share a time range, and "resource picker" parameters to scope to App Insights resources. Store source in Git (`Microsoft.Insights/workbooks` with `serializedData`).
**Code / KQL / CLI / Bicep**
```bicep
resource wb 'Microsoft.Insights/workbooks@2022-04-01' = {
  name: guid('service-health')
  location: resourceGroup().location
  kind: 'shared'
  properties: {
    displayName: 'Service Health'
    category: 'workbook'
    sourceId: appi.id
    serializedData: loadTextContent('service-health.workbook.json')
  }
}
```
**Gotchas / follow-ups**
- Workbook queries run as the *viewer*, so they need access to the data.
- Large cross-resource workbooks can be slow; pre-aggregate with summary tables if needed.

**Interview traps**
- Dashboards for interactive troubleshooting.
- Running every pod log into Analytics tier.
- Duplicate alerting in Grafana and Azure Monitor.

**Cheat sheet**
- Workbook = interactive KQL; dashboard = pinned tiles; Grafana = multi-source.
- Container insights -> `ContainerLogV2`; use DCR filters/Basic plan.
- VM insights = AMA + map.
- Deploy workbooks as code.

---

## Topic 16: Log Analytics cost, table plans, archive, data collection rules

**Study notes**

| Table plan | Query | Retention | Use |
|---|---|---|---|
| Analytics | Full KQL, alerts, all features | Interactive 30-730 days (+ archive up to 12 years total) | Operational/diagnostic data |
| Basic | Limited KQL (single table, limited operators), per-query charge, no alerts | 30 days interactive fixed (+ archive) | High-volume, low-value logs (verify supported tables) |
| Auxiliary | Very cheap, limited query, long-term | verify | Audit/verbose logs (verify) |

Pricing models: Pay-as-you-go (per GB), **commitment tiers** (discounted per day), Dedicated cluster (large scale, CMK). **Archive** keeps data cheap beyond interactive retention; access with **search jobs** or **restore**. Verify prices/limits in current docs.

**Data Collection Rules (DCR)**: define data sources (AMA, Logs Ingestion API, platform), **transformations** (KQL on `source`, applied before ingestion/billing), and destinations. A *workspace transformation DCR* applies to tables from legacy sources (diagnostic settings, App Insights).

### 16.1 [Easy] Where does Log Analytics cost come from?
**Answer:** Mainly data ingestion volume (GB) plus retention beyond the free period, with possible charges for Basic-log queries, search jobs/restores, and alert rules.
**Explanation:** Reduce volume at the source first; then pick commitment tiers when daily ingestion is stable.
**Code / KQL / CLI / Bicep**
```kusto
Usage | where TimeGenerated > ago(30d) and IsBillable
| summarize GB = sum(Quantity) / 1024 by DataType | top 10 by GB desc
```

### 16.2 [Follow-up] Set retention and archive on one table.
**Answer:** Per table: `retentionInDays` for interactive and `totalRetentionInDays` for archive total.
**Explanation:** Archived data is not directly queryable: run a search job (results to a new table) or restore a time range.
**Code / KQL / CLI / Bicep**
```bash
az monitor log-analytics workspace table update -g rg-mon --workspace-name law-prod \
  -n AppTraces --retention-time 30 --total-retention-time 365
az monitor log-analytics workspace table update -g rg-mon --workspace-name law-prod \
  -n ContainerLogV2 --plan Basic
```

### 16.3 [Medium] How do I drop or trim data before it's billed?
**Answer:** A DCR transformation: KQL on the incoming `source` stream that filters rows (`where`) or removes columns (`project-away`). For App Insights/workspace tables use the workspace default DCR transformation.
**Explanation:** Transformations run before storage; filtered data isn't billed for retention (ingestion of the transformed output is billed). Transformation language is a KQL subset (verify supported operators). Can mask PII too (Topic 17).
**Code / KQL / CLI / Bicep**
```kusto
// transformKql in the DCR dataFlow
source
| where SeverityLevel >= 2 or not(Message has "healthcheck")
| project-away Properties
```

### 16.4 [Medium] Basic logs: what do I give up?
**Answer:** Rich querying (limited operators, single table, no joins across tables except lookup to small tables), no log alerts, per-GB query charge and fixed interactive retention; benefit is much lower ingestion cost.
**Explanation:** Good for verbose container/HTTP access logs used only for ad-hoc debugging. Check which tables are eligible; the set expands over time (verify).
**Code / KQL / CLI / Bicep**
```kusto
ContainerLogV2 | where TimeGenerated > ago(1h) and LogMessage has "error" | take 50
```

### 16.5 [Hard] Your bill doubled this month. Investigate end-to-end.
**Answer:** (1) `Usage` by DataType per day to locate the table and date of the jump, (2) drill into the table (`_BilledSize` by source/resource/category), (3) correlate with a deployment/log-level change, (4) mitigate: levels/sampling/DCR/plan/diagnostic categories, (5) add budget/daily-volume alerts.
**Explanation:** Typical offenders: verbose `AppTraces`/`AppDependencies`, `AzureDiagnostics` from Storage/Key Vault `allLogs`, `ContainerLogV2`, Defender/Sentinel tables, retention increases.
**Code / KQL / CLI / Bicep**
```kusto
Usage | where TimeGenerated > ago(14d) and IsBillable
| summarize GB = sum(Quantity)/1024 by bin(TimeGenerated, 1d), DataType
| evaluate pivot(DataType, sum(GB))
```
```kusto
AzureDiagnostics | where TimeGenerated > ago(2d)
| summarize GB = sum(_BilledSize)/1e9 by ResourceProvider, Category, Resource | top 20 by GB desc
```
**Gotchas / follow-ups**
- `Quantity` in `Usage` is MB (not GB).
- Ingestion reported daily may lag by ~hours.

**Interview traps**
- Reducing retention to cut ingestion costs (different dimension).
- Daily cap as cost control only.
- Expecting Basic logs to support alerts.

**Cheat sheet**
- Cost = ingest + retention (+ query on Basic).
- DCR transformations cut before billing.
- Archive -> search job/restore.
- `Usage` and `_BilledSize` for attribution.

---

## Topic 17: Entra auth for ingestion (SAMI/UAMI) & PII scrubbing

**Study notes**

By default telemetry is accepted with just the connection string/ikey (local auth). With **Entra authentication**, the app authenticates with a managed identity (or other credential) granted **Monitoring Metrics Publisher** on the App Insights resource; set `DisableLocalAuth = true` to reject ikey-only ingestion. (Verify role names and SDK support in current docs.)

| Case | Setting |
|---|---|
| Functions/App Service with SAMI | `APPLICATIONINSIGHTS_AUTHENTICATION_STRING = Authorization=AAD` |
| With UAMI | `APPLICATIONINSIGHTS_AUTHENTICATION_STRING = Authorization=AAD;ClientId=<uami-client-id>` |
| OTel distro in code | `options.Credential = new DefaultAzureCredential()` / `ManagedIdentityCredential` |
| Classic SDK | `TelemetryConfiguration.SetAzureTokenCredential(credential)` |

PII sources: URLs with query strings/tokens, SQL parameters in `dependencies.data`, exception messages, custom dimensions, client IP (geolocation then masked by default), user ids. Controls: don't log them; telemetry processors/initializers; DCR transformations; workspace **purge** API for removal (verify); RBAC and table-level access; data residency/CMK.

### 17.1 [Easy] How can an app send telemetry without a secret?
**Answer:** Use Entra auth: give the app's managed identity the *Monitoring Metrics Publisher* role on the App Insights resource and configure the SDK/Functions to use the credential; the connection string stays for routing only.
**Explanation:** Eliminates reliance on the ikey as an authorization mechanism and lets you disable local auth.
**Code / KQL / CLI / Bicep**
```bicep
resource role 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(appi.id, principalId, 'metrics-publisher')
  scope: appi
  properties: {
    principalId: principalId
    principalType: 'ServicePrincipal'
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '3913510d-42f4-4e42-8a64-420c390055eb') // Monitoring Metrics Publisher
  }
}
```

### 17.2 [Follow-up] Configure SAMI vs UAMI in code and in Functions.
**Answer:** SAMI: `new ManagedIdentityCredential()` / `Authorization=AAD`. UAMI: pass the client id (`ManagedIdentityId.FromUserAssignedClientId`) / `Authorization=AAD;ClientId=...`.
**Explanation:** With multiple identities on a resource you must specify the client ID, or token acquisition is ambiguous. Locally use `DefaultAzureCredential` (dev login).
**Code / KQL / CLI / Bicep**
```csharp
builder.Services.AddOpenTelemetry().UseAzureMonitor(o =>
{
    o.Credential = new ManagedIdentityCredential(ManagedIdentityId.FromUserAssignedClientId(uamiClientId));
});
```
```bash
az functionapp config appsettings set -g rg-app -n func-orders \
  --settings "APPLICATIONINSIGHTS_AUTHENTICATION_STRING=Authorization=AAD;ClientId=<uami-client-id>"
```

### 17.3 [Medium] After setting `DisableLocalAuth=true`, telemetry stopped. Why?
**Answer:** Ingestion is now rejected unless the client sends a valid Entra token; the app has no credential configured, the identity lacks the role, or an older SDK doesn't support AAD. Role assignment propagation can also take minutes.
**Explanation:** Some components (older agents, Profiler/Snapshot, availability tests, Live Metrics) had differing Entra support historically - verify in docs. Rollout order: configure credential + role first, then disable local auth. Failures show as 401/403 in SDK self-diagnostics logs.
**Code / KQL / CLI / Bicep**
```bash
az resource update --ids <appi-id> --set properties.DisableLocalAuth=true
```

### 17.4 [Medium] How do I scrub PII in the SDK?
**Answer:** Add a processor/initializer that removes or hashes sensitive fields (query strings, user ids, SQL text), set `SqlClient` instrumentation not to record statements where possible, and avoid logging PII at source.
**Explanation:** For OTel remove tags in `OnEnd` of a processor; for classic implement `ITelemetryInitializer`. Complement with DCR transformations as a safety net (workspace-side).
**Code / KQL / CLI / Bicep**
```csharp
public sealed class UrlScrubber : BaseProcessor<Activity>
{
    public override void OnEnd(Activity a)
    {
        if (a.GetTagItem("url.full") is string u)
            a.SetTag("url.full", u.Split('?')[0]);
    }
}
builder.Services.AddOpenTelemetry().UseAzureMonitor().WithTracing(t => t.AddProcessor(new UrlScrubber()));
```

### 17.5 [Hard] PII was already ingested. What now?
**Answer:** Stop the leak (fix code/transform), restrict access, then purge affected records with the Log Analytics purge API (async, limited; verify), and consider lowering retention for the table. Document the incident per compliance.
**Explanation:** Purge is a per-workspace API with `filters` on columns; completion can take days; purge on App* tables available via workspace. Prevention beats cure: transformation DCR masks at ingestion.
**Code / KQL / CLI / Bicep**
```kusto
// DCR transformation: mask email addresses in Message
source
| extend Message = replace_regex(Message, @"[\w\.-]+@[\w\.-]+\.\w+", "<email>")
```
**Gotchas / follow-ups**
- Purge isn't instant and is rate-limited; backups/exports to Storage (diagnostic settings) are not purged.
- Client IP is stored as `0.0.0.0`-masked by default; check `client_IP` before assuming it's PII-free.

**Interview traps**
- Connection string treated as the secret; actual control is Entra + disable local auth.
- Forgetting UAMI ClientId.
- Disabling local auth before the apps are ready.

**Cheat sheet**
- Role: Monitoring Metrics Publisher.
- Functions: `APPLICATIONINSIGHTS_AUTHENTICATION_STRING`.
- `DisableLocalAuth=true` enforces Entra-only.
- Scrub in app, then DCR, then purge as last resort.

---

## Topic 18: SLO/SLI/error budgets, golden signals (RED/USE), release correlation

**Study notes**

- **SLI**: measured indicator (e.g., % of requests succeeding < 500 ms). **SLO**: target (99.9% / 30d). **SLA**: contractual. **Error budget** = 1 - SLO (99.9% -> 0.1% = ~43.2 min/30d).
- **Golden signals** (Google): latency, traffic, errors, saturation. **RED** (services): Rate, Errors, Duration. **USE** (resources): Utilization, Saturation, Errors.
- **Burn rate** = observed error rate / budget rate. 14.4x burns 2% of 30d budget in 1h.

Release correlation: set `service.version` (OTel) / `Component.Version` (classic) -> `application_Version`; add **release annotations** (Azure Pipelines task or ARM/REST Annotations API - verify current method) so charts show deployment markers.

### 18.1 [Easy] Define SLI, SLO and error budget.
**Answer:** An SLI is a measurable quality metric, an SLO a target for it over a window, and the error budget is the allowed unreliability (1 - SLO) that you can spend on change/risk.
**Explanation:** Example: SLI = good requests / valid requests; SLO 99.9% monthly; budget 0.1%.
**Code / KQL / CLI / Bicep**
```kusto
requests | where timestamp > ago(30d)
| summarize total = sum(itemCount), good = sumif(itemCount, success == true and duration < 500)
| extend sli = round(100.0 * good / total, 3), budgetUsedPct = round((100.0 - 100.0 * good / total) / 0.1 * 100, 1)
```

### 18.2 [Follow-up] RED vs USE vs golden signals?
**Answer:** RED (rate, errors, duration) is for request-driven services; USE (utilization, saturation, errors) is for resources like CPU, disk, queues; golden signals add saturation to the RED view. Use RED for services and USE for infra/queues.
**Explanation:** Mapping on Azure: Rate = `requests`; Errors = `success == false`; Duration = percentiles; Saturation = Service Bus active messages/queue age, CPU, thread-pool, Redis server load.
**Code / KQL / CLI / Bicep**
```kusto
requests | where timestamp > ago(1h)
| summarize rate = sum(itemCount) / 60.0, errors = sumif(itemCount, success == false), p95 = percentile(duration, 95) by bin(timestamp, 1m), cloud_RoleName
```

### 18.3 [Medium] How do you correlate an incident with a deployment?
**Answer:** Tag telemetry with the version, plot error rate/latency by `application_Version`, and show release annotations on charts; compare the windows before/after.
**Explanation:** With slot/canary deployments, version splits give direct A/B comparisons. Also use Activity Log deployment events in the same workspace.
**Code / KQL / CLI / Bicep**
```kusto
requests | where timestamp > ago(6h)
| summarize failed = sumif(itemCount, success == false), total = sum(itemCount) by bin(timestamp, 10m), application_Version
| extend errPct = 100.0 * failed / total | project timestamp, application_Version, errPct | render timechart
```
```csharp
.ConfigureResource(r => r.AddService("orders-api", serviceVersion: Environment.GetEnvironmentVariable("BUILD_ID")))
```

### 18.4 [Hard] Implement multi-window multi-burn-rate alerts for 99.9% over 30 days.
**Answer:** Page when burn rate > 14.4 over 1h and > 14.4 over 5m; ticket when > 6 over 6h and 30m (budget consumption 2% and 5%). Both windows must exceed to reduce flapping and quick reset.
**Explanation:** Fast burn catches outages; slow burn catches regressions. Short window confirms it's still burning. Implement either as two log alert criteria or one query calculating both.
**Code / KQL / CLI / Bicep**
```kusto
let budget = 0.001;
let rate = (w:timespan) { requests | where timestamp > ago(w) | summarize bad = sumif(itemCount, success == false), total = sum(itemCount) | extend r = todouble(bad) / total / budget };
union (rate(1h) | extend w = "1h"), (rate(5m) | extend w = "5m")
| summarize minBurn = min(r), samples = min(total)
| where minBurn > 14.4 and samples > 100
```
**Gotchas / follow-ups**
- Low-traffic services: minimum-volume guard, or use synthetic probes to produce steady traffic.
- Define "valid" requests (exclude health checks and client 4xx) consistently.

### 18.5 [Tricky] SLO says 99.95% but customers complain. What might the SLI miss?
**Answer:** The SLI is measured at the wrong place or excludes failures: server-side telemetry misses requests that never arrived (DNS/WAF/gateway failures), sampling skews, 4xx counted as good, client-observed latency differs, or availability measured per-instance instead of per-user journey.
**Explanation:** Measure at the edge (Front Door/APIM logs) and with synthetic tests, include real user monitoring (browser), and define SLIs per critical user journey.
**Code / KQL / CLI / Bicep**
```kusto
// Compare edge vs app view (APIM logs table: verify name)
ApiManagementGatewayLogs | where TimeGenerated > ago(1h)
| summarize edgeTotal = count(), edge5xx = countif(ResponseCode >= 500) by bin(TimeGenerated, 5m)
```

**Interview traps**
- Using SLA as SLO.
- Averages for latency (use percentiles).
- SLI counting health probes.

**Cheat sheet**
- Budget = 1 - SLO; 99.9% = 43.2 min/30d.
- RED services, USE resources.
- Multi-window burn rates: 14.4x/1h+5m, 6x/6h+30m.
- Version tags + annotations for releases.

---

## Topic 19: Production scenarios & troubleshooting playbook

**Study notes**

Triage order for "telemetry problem": **Is data produced? -> Is it sent? -> Is it accepted? -> Is it stored/visible?**

```
App code/config -> SDK filters/sampling -> network/DNS/private link -> ingestion (auth, cap) -> table -> query (scope/time/schema)
```
Tools: Live Metrics (pipeline up?), SDK self-diagnostics, `Usage`/`_LogOperation`, `sdkVersion` & `cloud_RoleInstance` breakdown, Kudu/log stream, `curl` to the ingestion endpoint (HTTP 200/206/400/401/402/429/439 semantics - verify).

### 19.1 [Easy] No telemetry appears at all. First checks?
**Answer:** Verify the connection string (correct resource/endpoint), that the SDK is registered at startup, outbound connectivity to the ingestion endpoint, daily cap, and the query time range/scope.
**Explanation:** Check Live Metrics first: if live data appears but not in Logs, it's ingestion/latency/cap/query scope; if neither, it's app config/network/auth.
**Code / KQL / CLI / Bicep**
```bash
az functionapp config appsettings list -g rg-app -n func-orders --query "[?name=='APPLICATIONINSIGHTS_CONNECTION_STRING']"
curl -s -o NUL -w "%{http_code}" https://westeurope-5.in.applicationinsights.azure.com/v2/track
```
```kusto
union requests, traces, dependencies | where timestamp > ago(1h) | summarize last = max(timestamp), n = count() by itemType
```

### 19.2 [Follow-up] Telemetry is duplicated.
**Answer:** Two pipelines (classic + OTel distro, App Service auto-instrumentation + SDK, host + worker both emitting the same logs) or the same data sent to two resources. Keep one.
**Explanation:** Differentiate by `sdkVersion`, `cloud_RoleInstance`, and duplicates sharing `operation_Id`+`id`. See Topic 4.5.
**Code / KQL / CLI / Bicep**
```kusto
requests | where timestamp > ago(1h) | summarize n = count(), sdks = make_set(sdkVersion) by id | where n > 1 | take 20
```

### 19.3 [Medium] High cost from verbose logging.
**Answer:** Identify top tables/categories with `Usage`/`_BilledSize`, raise per-category levels (`Microsoft.*`, `System.*`, Azure SDK to Warning), enable sampling, filter with DCR, move to Basic logs if appropriate, and set budget alerts.
**Explanation:** See Topics 5, 7, 16. Prevent regressions by checking log-level config in code review and alerting on daily GB per table.
**Code / KQL / CLI / Bicep**
```kusto
AppTraces | where TimeGenerated > ago(1d)
| summarize GB = sum(_BilledSize)/1e9, n = sum(ItemCount) by AppRoleName, tostring(Properties["Category"]) | top 15 by GB desc
```

### 19.4 [Medium] Dependency telemetry is missing (SQL/Redis/Service Bus).
**Answer:** The dependency isn't auto-collected: add the instrumentation (e.g., SqlClient / StackExchange.Redis OTel instrumentation or classic `Microsoft.ApplicationInsights.StackExchangeRedis`), register Azure SDK ActivitySources, or call is made outside an Activity; also sampling/filters or processors dropping it.
**Explanation:** HttpClient and SqlClient are covered by the distro; Redis often needs an extra package and `ConnectionMultiplexer` registration with the instrumentation. Verify current package names in docs.
**Code / KQL / CLI / Bicep**
```csharp
builder.Services.AddSingleton<IConnectionMultiplexer>(_ => ConnectionMultiplexer.Connect(redisConn));
builder.Services.AddOpenTelemetry().UseAzureMonitor()
    .WithTracing(t => t.AddRedisInstrumentation()); // OpenTelemetry.Instrumentation.StackExchangeRedis (verify)
```

### 19.5 [Hard] Wrong `cloud_RoleName` (all services show as one node, or GUID-like names).
**Answer:** Role name defaults to the host/app name or "unknown_service" when `service.name` isn't set. Set it explicitly per service via resource attributes (`OTEL_SERVICE_NAME` / `OTEL_RESOURCE_ATTRIBUTES`) or a classic initializer; in Functions it defaults to the function app name.
**Explanation:** In OTel, role = `service.namespace.service.name`; instance = `service.instance.id`. Changing it splits historical data (old name vs new) in the Application Map and may need alert query updates.
**Code / KQL / CLI / Bicep**
```bash
az webapp config appsettings set -g rg-app -n app-orders \
  --settings OTEL_SERVICE_NAME=orders-api "OTEL_RESOURCE_ATTRIBUTES=service.namespace=shop,service.version=1.4.2"
```
```kusto
union requests, dependencies | where timestamp > ago(1h) | summarize n = sum(itemCount) by cloud_RoleName, cloud_RoleInstance
```

### 19.6 [Hard] Correlation broken across messaging.
**Answer:** Check that the producer emits a dependency span and stamps `Diagnostic-Id`/`traceparent`, that custom envelopes preserve application properties, intermediaries (Logic Apps/Event Grid/APIM) forward them, and that consumers extract context. Fall back to a business correlation id.
**Explanation:** See Topic 8.5. Validate with a test message and look at `ApplicationProperties` in Service Bus Explorer.
**Code / KQL / CLI / Bicep**
```kusto
dependencies | where timestamp > ago(1h) and type has "Service Bus" | project operation_Id, sendId = id, timestamp
| join kind=leftouter (requests | where timestamp > ago(1h) | project operation_Id, parent = operation_ParentId, recv = id) on $left.sendId == $right.parent
| summarize sends = count(), matched = countif(isnotempty(recv))
```

### 19.7 [Tricky] Alerts didn't fire (or fire too often).
**Answer:** Didn't fire: ingestion delay, sampling skew, no-data semantics, wrong scope/dimension, action group disabled/throttled, suppression rule, stateful alert already fired. Too often: tight thresholds, no min-volume guard, single-window, flapping resolved/fired, alerting on causes.
**Explanation:** Debug with the Alerts blade (alert history + "fired but notification failed" in action group activity), run the alert query manually for the window, check `ingestion_time()` lag, and test the action group. Prefer SLO burn-rate design (Topic 13/18).
**Code / KQL / CLI / Bicep**
```kusto
// Ingestion latency to size alert windows
requests | where timestamp > ago(1h)
| extend lagSec = datetime_diff('second', ingestion_time(), timestamp)
| summarize p50 = percentile(lagSec, 50), p99 = percentile(lagSec, 99)
```
**Gotchas / follow-ups**
- Log alert query uses ingestion-based evaluation windows; late data may be missed.
- Test action groups using "Test action group" in portal.

### 19.8 [Tricky] Function app appears healthy but telemetry shows many "Host.Results" traces and the bill rose.
**Answer:** `Host.Results`/`Host.Aggregator` and Function categories log at Information for every invocation; with high throughput this dominates volume. Tune host.json levels and sampling, exclude noisy categories, and rely on metrics for aggregates.
**Explanation:** Do not silence `Host.Results` completely if you depend on `requests`; use sampling with `excludedTypes: "Request;Exception"`... note exclusion reduces savings; balance.
**Code / KQL / CLI / Bicep**
```kusto
traces | where timestamp > ago(1d) | summarize n = sum(itemCount) by tostring(customDimensions["Category"]) | top 10 by n desc
```

**Interview traps**
- Debugging queries before confirming data is produced and accepted.
- Changing role names without updating alert queries.
- Ignoring ingestion latency in alert design.

**Cheat sheet**
- Triage: produced -> sent -> accepted -> stored -> queried.
- Live Metrics separates pipeline vs storage issues.
- Duplicates = two pipelines; missing deps = missing instrumentation.
- `OTEL_SERVICE_NAME` fixes role name.
- Business correlation id is the messaging fallback.
