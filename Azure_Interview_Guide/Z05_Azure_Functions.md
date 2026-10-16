# Z05 - Azure Functions: Study + Interview Prep Guide

**How to use:** Read the *Study notes* of each topic first, then work the difficulty ladder (Easy -> Tricky) answering out loud before reading the **Answer**.
Limits, retirement dates, preview features and SDK/package versions change often - anything marked "verify in current docs" must be re-checked on learn.microsoft.com before you quote it.

## Table of Contents
1. Topic 1: Serverless and the Azure Functions concept
2. Topic 2: In-process vs isolated worker model; .NET 8/9 isolated project structure
3. Topic 3: Triggers, bindings and binding expressions
4. Topic 4: HTTP triggers (ASP.NET Core integration, auth levels, routes)
5. Topic 5: Hosting plans compared
6. Topic 6: Scale controller, instance scaling, concurrency and host.json
7. Topic 7: Blob trigger - polling vs Event Grid
8. Topic 8: Service Bus trigger behaviour and settlement
9. Topic 9: Timer trigger, singleton and multi-instance issues
10. Topic 10: Retries, poison messages, dead-lettering, idempotency, error handling and logging
11. Topic 11: Durable Functions
12. Topic 12: Durable vs Logic Apps; Functions vs WebJobs vs Logic Apps vs Container Apps jobs
13. Topic 13: Configuration, Key Vault references, identity-based connections, SAMI vs UAMI
14. Topic 14: Deployment and local development
15. Topic 15: Testing
16. Topic 16: Monitoring (Application Insights, KQL)
17. Topic 17: Security
18. Topic 18: Performance, cost model and limits
19. Topic 19: Production scenarios (troubleshooting playbook)

---

## Topic 1: Serverless and the Azure Functions concept

**Study notes**

Serverless = you ship code, the platform handles servers, patching, scaling and (usually) bills per use. Azure Functions is event-driven FaaS: a *trigger* starts a *function*; *bindings* declaratively connect data in/out. A *function app* is the deployment/scale/config unit and hosts one or more functions sharing one runtime (Functions host) and one plan.

```
 Event source --> [Trigger listener in Functions host] --> your function --> [Output binding] --> target
 (queue, HTTP,      (scale controller watches the source       (C# method)      (queue, blob, DB)
  timer, blob..)     and adds/removes host instances)
```

| Property | Meaning |
|---|---|
| Event-driven | Executions are caused by events, not a running server loop |
| Scale to zero | Consumption / Flex Consumption can have 0 instances |
| Per-execution billing | Consumption and Flex (on-demand) bill executions + GB-s |
| Stateless by default | Use Durable Functions or external stores for state |

**Difficulty ladder**

### 1.1 [Easy] What is Azure Functions and what is a function app?
**Answer**
Azure Functions is Azure's event-driven serverless compute service: a trigger invokes your code and bindings simplify input/output. A function app is the container for one or more functions that share the same runtime, plan, scaling and configuration.
**Explanation**
Each function has exactly one trigger and zero or more input/output bindings. All functions in an app scale together as one unit (instances of the app), so mixing a heavy and a light workload in one app couples their scaling and failure domains. Settings (app settings, identity, networking) are per app.
**Code / CLI / Bicep**
```bash
az functionapp create -g rg-demo -n func-demo-001 --storage-account stdemo001 \
  --flexconsumption-location westeurope --runtime dotnet-isolated --runtime-version 8.0
```
(Flex Consumption create flags - verify in current docs for your CLI version.)

### 1.2 [Follow-up] When would you choose Functions over App Service, Container Apps or Logic Apps?
**Answer**
Choose Functions for short, event-triggered, code-first units of work with bursty load. App Service fits always-on web apps, Container Apps fits containerized microservices/long-running workers, and Logic Apps fits low-code integration orchestration with many connectors.
**Explanation**
Functions gives the richest trigger/binding catalog and scale-to-zero economics. It is a poor fit for long-running CPU jobs (timeouts), workloads needing fine-grained process control, or very chatty low-latency APIs where cold start matters (unless Premium/Flex always-ready).
**Code / CLI / Bicep**
```bash
# Decision shortcut (conceptual, not a command):
# short + event + code  -> Functions
# long/containers/KEDA  -> Container Apps (jobs)
# connector workflow    -> Logic Apps
```

### 1.3 [Medium] How does a function actually get executed - what runs where?
**Answer**
The Functions host (a .NET process) runs trigger listeners and invokes your code. In the isolated model your code runs in a separate worker process and the host communicates with it over gRPC. The scale controller (platform component, not your code) decides how many host instances exist.
**Explanation**
The host reads `function.json`-equivalent metadata (generated at build for .NET), starts listeners per trigger, deserializes the event, sends an invocation request to the worker, collects the result and runs output bindings. State such as leases, singleton locks, timer status and keys lives in the storage account referenced by `AzureWebJobsStorage`.
**Code / CLI / Bicep**
```json
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "FUNCTIONS_WORKER_RUNTIME": "dotnet-isolated"
  }
}
```

### 1.4 [Hard] Why is "stateless" not the same as "no shared state problems" in Functions?
**Answer**
Instances are ephemeral and many may run concurrently, so in-memory state is per-instance, lost on recycle, and not shared. Static fields persist only within one process lifetime and can leak across invocations on the same instance, which is both a benefit (connection reuse) and a hazard (non-thread-safe state).
**Explanation**
Within one instance, invocations can run concurrently on the thread pool, so any static mutable state must be thread-safe. Across instances, coordination needs an external store (Redis, Storage, SQL) or Durable Functions entities. Caches in static memory can serve stale data per-instance.
**Code / CLI / Bicep**
```csharp
public class Worker
{
    // BAD: shared mutable state, not thread-safe and not shared across instances
    private static int _counter;

    // OK: thread-safe, per-instance only
    private static readonly ConcurrentDictionary<string, string> _cache = new();
}
```
**Gotchas / follow-ups**
- Static `HttpClient` is good; static mutable business state is not.
- Scale-out multiplies any per-instance rate limit.

**Interview traps**
- Saying "functions scale independently" - they scale per *function app*, not per function (Flex Consumption offers per-function scaling for HTTP/Blob/Durable groups - verify in current docs).
- Claiming serverless = no cold starts.
- Forgetting the storage account dependency.

**Cheat sheet**
- Trigger (1) + bindings (0..n) = function; functions live in a function app.
- Host process + (isolated) worker process over gRPC.
- Scale unit = function app instance.
- `AzureWebJobsStorage` holds keys, leases, timer state, checkpoints for some triggers.

---

## Topic 2: In-process vs isolated worker model; .NET 8/9 isolated project structure

**Study notes**

| Aspect | In-process | Isolated worker |
|---|---|---|
| Process | Same as host | Separate process, gRPC |
| .NET versions | Tied to host's .NET | Any supported (.NET 8, 9, Framework) |
| Middleware / DI control | Limited | Full `HostBuilder`, middleware |
| ASP.NET Core integration | No | Yes (`ConfigureFunctionsWebApplication`) |
| Package | `Microsoft.NET.Sdk.Functions` | `Microsoft.Azure.Functions.Worker*` |
| Status | **Support ends 10 Nov 2026 per announcement - verify in current docs** | Recommended, actively developed |

Minimal isolated Program.cs (.NET 8/9):

```
Program.cs -> FunctionsApplication.CreateBuilder(args) / HostBuilder
           -> ConfigureFunctionsWebApplication()  (ASP.NET Core integration)
           -> services (DI), App Insights, middleware
           -> Build().Run()
```

**Difficulty ladder**

### 2.1 [Easy] What is the difference between in-process and isolated worker model?
**Answer**
In-process runs your code inside the Functions host process and is bound to the host's .NET version; isolated runs it in a separate worker process, decoupled from host. Isolated is the recommended model and in-process is being retired (verify the date in current docs).
**Explanation**
Isolated gives you control of startup, DI, middleware, .NET version choice and avoids assembly conflicts with the host. Cost: a small extra gRPC hop and some differences in binding types (e.g. different attributes and types in `Microsoft.Azure.Functions.Worker.Extensions.*`).
**Code / CLI / Bicep**
```xml
<PropertyGroup>
  <TargetFramework>net8.0</TargetFramework>
  <AzureFunctionsVersion>v4</AzureFunctionsVersion>
  <OutputType>Exe</OutputType>
</PropertyGroup>
<ItemGroup>
  <PackageReference Include="Microsoft.Azure.Functions.Worker" />
  <PackageReference Include="Microsoft.Azure.Functions.Worker.Sdk" />
  <PackageReference Include="Microsoft.Azure.Functions.Worker.Extensions.Http.AspNetCore" />
</ItemGroup>
```
(Pin versions per current NuGet.)

### 2.2 [Follow-up] Show a typical .NET 8/9 isolated Program.cs with DI and Application Insights.
**Answer**
Use `FunctionsApplication.CreateBuilder` (or `HostBuilder` with `ConfigureFunctionsWebApplication`), register services, and add the Application Insights worker telemetry. Then `Build().Run()`.
**Explanation**
`ConfigureFunctionsWebApplication` enables ASP.NET Core integration (HttpRequest/IActionResult). `ConfigureFunctionsWorkerDefaults` alone gives the built-in model (HttpRequestData/HttpResponseData). Registering App Insights requires `Microsoft.Azure.Functions.Worker.ApplicationInsights`. By default the worker adds a logging filter that can hide Information logs below Warning for App Insights - remove the default rule if you need them.
**Code / CLI / Bicep**
```csharp
using Microsoft.Azure.Functions.Worker;
using Microsoft.Azure.Functions.Worker.Builder;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

var builder = FunctionsApplication.CreateBuilder(args);
builder.ConfigureFunctionsWebApplication();

builder.Services
    .AddApplicationInsightsTelemetryWorkerService()
    .ConfigureFunctionsApplicationInsights();

builder.Services.AddHttpClient("downstream", c => c.BaseAddress = new Uri("https://api.example.com"));
builder.Services.AddSingleton<IMyService, MyService>();

// Allow Information logs through to App Insights (default rule filters them)
builder.Logging.Services.Configure<LoggerFilterOptions>(o =>
{
    var rule = o.Rules.FirstOrDefault(r =>
        r.ProviderName == "Microsoft.Extensions.Logging.ApplicationInsights.ApplicationInsightsLoggerProvider");
    if (rule is not null) o.Rules.Remove(rule);
});

builder.Build().Run();
```

### 2.3 [Medium] How do you write middleware and what is FunctionContext?
**Answer**
Implement `IFunctionsWorkerMiddleware` with `Invoke(FunctionContext, FunctionExecutionDelegate)` and register it with `UseMiddleware<T>()`. `FunctionContext` carries invocation info (function name, invocation id, bindings, `InstanceServices`).
**Explanation**
Middleware wraps every invocation (all trigger types) - good for correlation ids, global exception handling, and auth on non-HTTP concerns. For HTTP with ASP.NET Core integration you can also use standard ASP.NET Core middleware, but it only sees HTTP requests.
**Code / CLI / Bicep**
```csharp
public class ExceptionMiddleware : IFunctionsWorkerMiddleware
{
    public async Task Invoke(FunctionContext context, FunctionExecutionDelegate next)
    {
        var log = context.GetLogger<ExceptionMiddleware>();
        try { await next(context); }
        catch (Exception ex)
        {
            log.LogError(ex, "Unhandled in {Function} ({Id})",
                context.FunctionDefinition.Name, context.InvocationId);
            throw; // rethrow so retry / poison handling still works
        }
    }
}
// Program.cs: builder.UseMiddleware<ExceptionMiddleware>();
```

### 2.4 [Hard] What changes when migrating in-process to isolated?
**Answer**
Switch project to `OutputType Exe`, replace packages with `Microsoft.Azure.Functions.Worker*`, add Program.cs, change attributes (`[Function]` instead of `[FunctionName]`), use `FunctionContext` logger/`ILogger<T>` injection, and set `FUNCTIONS_WORKER_RUNTIME=dotnet-isolated`. Bindings types differ (e.g. `ServiceBusReceivedMessage` is supported; `IAsyncCollector` is replaced by return-value/output types).
**Explanation**
Output bindings are usually done via a return type with an output attribute (multiple outputs need a POCO with properties attributed). ILogger is no longer a method parameter by convention (inject `ILogger<T>` or use `context.GetLogger`). Durable uses `Microsoft.Azure.Functions.Worker.Extensions.DurableTask` with different orchestration APIs (`TaskOrchestrationContext`). Use the official .NET upgrade assistant / migration guide (verify in current docs).
**Code / CLI / Bicep**
```csharp
// In-process
[FunctionName("Hello")]
public static IActionResult Run([HttpTrigger] HttpRequest req, ILogger log) => new OkResult();

// Isolated
public class Hello(ILogger<Hello> log)
{
    [Function("Hello")]
    public IActionResult Run([HttpTrigger(AuthorizationLevel.Function, "get")] HttpRequest req)
    { log.LogInformation("hi"); return new OkResult(); }
}
```
**Gotchas / follow-ups**
- Forgetting to change `FUNCTIONS_WORKER_RUNTIME` makes the host not find your worker.
- Missing `ConfigureFunctionsWebApplication` when using `HttpRequest`/`IActionResult` yields runtime errors or 500s.
- Slot swap while changing the runtime setting requires care (mark as slot-sticky/test first).

**Interview traps**
- "Isolated is slower so avoid it" - overhead is small; in-process is retiring.
- Mixing `HttpRequestData` (built-in) with `ConfigureFunctionsWebApplication` expectations.
- Not rethrowing in middleware - swallows retries.
- Default App Insights log filter hiding Information logs.

**Cheat sheet**
- `[Function]` attr, `Program.cs`, `OutputType=Exe`, `dotnet-isolated`.
- `ConfigureFunctionsWebApplication` = ASP.NET Core integration.
- Middleware: `IFunctionsWorkerMiddleware`.
- DI by constructor injection; functions classes are instantiated per invocation (services lifetimes still apply).
- In-process retirement: verify in current docs.

---

## Topic 3: Triggers, bindings and binding expressions

**Study notes**

A trigger starts the function (exactly one). Input bindings read data; output bindings write data. In isolated .NET, bindings come from extension packages `Microsoft.Azure.Functions.Worker.Extensions.*`.

| Trigger | Package suffix | Notes |
|---|---|---|
| HTTP | `Http` / `Http.AspNetCore` | request/response |
| Timer | `Timer` | NCRONTAB schedule |
| Queue | `Storage.Queues` | poison queue `<name>-poison` |
| Service Bus | `ServiceBus` | queues/topic subscriptions, sessions |
| Blob | `Storage.Blobs` | polling or Event Grid source |
| Event Grid | `EventGrid` | push via webhook/subscription |
| Event Hubs | `EventHubs` | checkpoints in storage, partition-based |
| Cosmos DB | `CosmosDB` | change feed, lease container |

Binding expressions: `%AppSettingName%` (app setting at runtime), `{queueTrigger}`, `{name}` from trigger metadata/JSON payload properties, `{rand-guid}`, `{DateTime}`.

**Difficulty ladder**

### 3.1 [Easy] What are triggers and bindings?
**Answer**
A trigger defines what causes the function to run; bindings are declarative connections to other resources, input (read) or output (write). Every function has exactly one trigger.
**Explanation**
Bindings remove SDK boilerplate and handle connection config via app settings (connection name, not connection string). You can always skip bindings and use the SDK directly (common in production for control over retries, batching, settlement).
**Code / CLI / Bicep**
```csharp
[Function("QueueToBlob")]
[BlobOutput("out/{queueTrigger}.txt", Connection = "StorageConn")]
public string Run([QueueTrigger("jobs", Connection = "StorageConn")] string msg) => msg;
```

### 3.2 [Follow-up] How do binding expressions work, and what is `%Setting%`?
**Answer**
Binding expressions in attributes are resolved at runtime: `{prop}` pulls from trigger payload/metadata, and `%Name%` is replaced with the app setting value. This lets you avoid hard-coding queue names or schedules and vary them per environment.
**Explanation**
`%MySchedule%` works for Timer schedules and queue/topic names, enabling per-slot values. Note that `%...%` is for app settings only; `{...}` is for payload/trigger data. Attribute arguments must otherwise be compile-time constants, which is why `%` exists.
**Code / CLI / Bicep**
```csharp
[Function("Nightly")]
public void Run([TimerTrigger("%NightlySchedule%")] TimerInfo timer) { }

[Function("Copy")]
[BlobOutput("archive/{name}", Connection = "StorageConn")]
public byte[] Copy([BlobTrigger("inbox/{name}", Connection = "StorageConn")] byte[] data, string name) => data;
```

### 3.3 [Medium] How do you emit multiple outputs from an isolated function?
**Answer**
Return a POCO whose properties carry the output binding attributes; one property can also be the HTTP response marked with `[HttpResult]`. This replaces in-process `IAsyncCollector`/`out` parameters.
**Explanation**
Each attributed property is processed as an output binding after the function completes. For arbitrary dynamic number of messages, use a `string[]`/`IEnumerable<T>` property (supported for e.g. Service Bus output) or use the Service Bus SDK directly.
**Code / CLI / Bicep**
```csharp
public class MultiOut
{
    [ServiceBusOutput("outq", Connection = "SbConn")]
    public string[] Messages { get; set; } = [];

    [BlobOutput("audit/{rand-guid}.json", Connection = "StorageConn")]
    public string Audit { get; set; } = "";

    [HttpResult] public IActionResult Http { get; set; } = new OkResult();
}
```

### 3.4 [Hard] Why might you avoid output bindings in production?
**Answer**
Output bindings run only after the function returns, are not transactional with your own work, and hide batching/settlement control. For critical flows I use the SDK client (injected via `AddAzureClients`) so I can control retries, ordering, and failure handling explicitly.
**Explanation**
If the function succeeds but the output write fails, the invocation fails and retry re-runs the whole function - so the work before it must be idempotent. Also no true distributed transaction between the trigger settlement and the output; Service Bus transactions only cover Service Bus entities with the same connection (verify scope in current docs).
**Code / CLI / Bicep**
```csharp
builder.Services.AddAzureClients(c =>
{
    c.AddServiceBusClientWithNamespace("mybus.servicebus.windows.net");
    c.UseCredential(new DefaultAzureCredential());
});
// inject ServiceBusClient and create senders (cache them) in your function
```
**Gotchas / follow-ups**
- Creating a new `ServiceBusSender`/client per invocation wastes connections.
- `AddServiceBusClientWithNamespace` comes from the `Microsoft.Extensions.Azure` package (verify in current docs).

### 3.5 [Tricky] Your blob output name uses `{name}` but the function fails to start. Why?
**Answer**
The binding expression refers to a property that isn't available in the trigger's binding data (e.g. queue message not JSON, or property name case mismatch), or a `%Setting%` is missing from app settings. Binding expression resolution failures surface as startup/invocation errors.
**Explanation**
Check that the payload is valid JSON for property-based expressions, that the setting exists in the Azure app settings (not just local.settings.json), and that the trigger exposes the metadata. Also, nested properties use `{Obj.Prop}`.
**Code / CLI / Bicep**
```bash
az functionapp config appsettings list -g rg -n func --query "[?name=='NightlySchedule']"
```

**Interview traps**
- Thinking you can have two triggers on one function.
- Confusing `%Setting%` with `{expr}`.
- Using in-process binding types in isolated projects.
- Assuming output bindings are transactional.

**Cheat sheet**
- 1 trigger, N bindings; connection = *name prefix* of settings.
- `%X%` app setting, `{x}` payload/metadata.
- Multiple outputs: POCO with attributed props.
- Use SDK for control; bindings for convenience.

---

## Topic 4: HTTP triggers (ASP.NET Core integration, auth levels, routes)

**Study notes**

Two HTTP models in isolated: **ASP.NET Core integration** (`HttpRequest`, `IActionResult`, `ConfigureFunctionsWebApplication`) and the **built-in model** (`HttpRequestData`/`HttpResponseData`).

| AuthorizationLevel | Requirement |
|---|---|
| Anonymous | none |
| Function | function-specific key or host key |
| Admin | master key |

Keys are in the storage account (or Key Vault if configured) and sent via `x-functions-key` header or `?code=`. **Easy Auth** (App Service Authentication) validates Entra/other tokens before the request reaches your code - independent of function keys. Route prefix default is `api` (host.json `extensions.http.routePrefix`).

**Difficulty ladder**

### 4.1 [Easy] Create an HTTP-triggered function with a route parameter.
**Answer**
Use `[HttpTrigger(AuthorizationLevel.Function, "get", Route = "orders/{id:int}")]` and bind the route value as a method parameter.
**Explanation**
Route constraints follow ASP.NET Core syntax when ASP.NET integration is used. Default URL is `https://<app>.azurewebsites.net/api/orders/5`.
**Code / CLI / Bicep**
```csharp
[Function("GetOrder")]
public async Task<IActionResult> Run(
    [HttpTrigger(AuthorizationLevel.Function, "get", Route = "orders/{id:int}")] HttpRequest req,
    int id)
{
    return new OkObjectResult(new { id });
}
```

### 4.2 [Follow-up] What are function keys, and are they real security?
**Answer**
Function keys are shared secrets (function, host, master) that gate HTTP access. They are authorization by possession, not user identity, so for user/app identity use Easy Auth or JWT validation behind API Management.
**Explanation**
Keys can't express per-user permissions, are easily leaked in URLs/logs, and rotation breaks clients. They are acceptable for service-to-service with rotation, but best-practice is Entra ID auth (Easy Auth) with `Anonymous` function level, plus network restrictions.
**Code / CLI / Bicep**
```bash
az functionapp function keys list -g rg -n func --function-name GetOrder
curl -H "x-functions-key: <key>" https://func.azurewebsites.net/api/orders/5
```

### 4.3 [Medium] How do you use Easy Auth with Entra ID and read the caller's identity in code?
**Answer**
Enable authentication on the function app with Entra as provider and "require authentication"; the platform injects `X-MS-CLIENT-PRINCIPAL` headers. In ASP.NET Core integration you can also validate the bearer token yourself; relying on Easy Auth alone means your code trusts platform headers, which is safe only when the app can't be reached bypassing the platform.
**Explanation**
Set the function authorization level to Anonymous when using Easy Auth (otherwise clients need both key and token). Easy Auth for Functions - for direct app-to-app calls with managed identities, validate `roles` claim for app roles; `allowedApplications` / allowed client apps and audiences are configured in auth settings.
**Code / CLI / Bicep**
```csharp
var principalHeader = req.Headers["X-MS-CLIENT-PRINCIPAL"].FirstOrDefault();
if (principalHeader is null) return new UnauthorizedResult();
var json = Encoding.UTF8.GetString(Convert.FromBase64String(principalHeader));
// parse claims: roles, oid, appid ...
```
```bash
az webapp auth show -g rg -n func   # inspect auth settings (authsettingsV2 via az webapp auth)
```

### 4.4 [Hard] Why do you get 404 or wrong routes after switching to ASP.NET Core integration, and how to remove the `/api` prefix?
**Answer**
With `ConfigureFunctionsWebApplication` the route behaviour is the same `api` prefix default, but 404s arise when routes collide or `Route` isn't unique. To remove the prefix set `extensions:http:routePrefix` to an empty string in host.json.
**Explanation**
Changing the prefix is a host.json change requiring restart (app settings change/restart). Methods must be listed; omitting methods allows all. Each function must have a unique route+method combination or host indexing fails.
**Code / CLI / Bicep**
```json
{
  "version": "2.0",
  "extensions": { "http": { "routePrefix": "" } }
}
```
**Gotchas / follow-ups**
- Don't expose `/admin` or `/runtime` endpoints publicly; they require master key but still reduce attack surface via network restrictions.
- Long HTTP responses: default Azure Load Balancer idle timeout is 230 seconds regardless of function timeout - use async pattern (202 + status endpoint) / Durable.

### 4.5 [Tricky] An anonymous function behind Easy Auth returns 401 from the browser but works in curl with a key. Why?
**Answer**
Easy Auth intercepts before function-level auth: with "require authentication" on, any request without a valid token is rejected with 401 even if the function level is Anonymous or the key is valid. Keys are only evaluated after Easy Auth passes (or are excluded paths).
**Explanation**
So the two layers stack. Webhooks (Event Grid, Stripe) calling key-based endpoints must be added to excluded paths or the auth set to "allow unauthenticated" with in-code enforcement.
**Code / CLI / Bicep**
```bash
az webapp auth update -g rg -n func --excluded-paths "/api/webhook" 
```
(Verify flag availability in your az version / use authsettingsV2 in Bicep.)

**Interview traps**
- "Function key = user authentication."
- Forgetting the 230 s HTTP idle limit.
- Mixing `HttpRequest` and `HttpRequestData` in the same app without understanding the integration.
- Leaving `Admin`-level or master key in client code.

**Cheat sheet**
- ASP.NET Core integration: `ConfigureFunctionsWebApplication`.
- Levels: Anonymous / Function / Admin; key header `x-functions-key`.
- Prefer Easy Auth + managed identity/roles.
- Route prefix: host.json `extensions.http.routePrefix`.
- HTTP response hard limit ~230 s.

---

## Topic 5: Hosting plans compared

**Study notes**

| Plan | Scale to zero | Cold start | Max scale (verify) | Timeout | VNet | Billing |
|---|---|---|---|---|---|---|
| Consumption (Windows/Linux) | yes | yes | ~200 (Win) / 100 (Linux) | default 5 min, max 10 | limited (no VNet integration on classic Consumption) | executions + GB-s |
| **Flex Consumption** | yes (optional always-ready) | reduced w/ always-ready | up to ~1000 (verify) | default 30 min (verify), unbounded option | yes (VNet integration, private endpoints) | on-demand execs + GB-s, always-ready baseline |
| Premium (Elastic Premium EP1-3) | min 1 instance | none for always-ready + pre-warmed | ~100 | default 30 min, unbounded | yes | vCPU/memory per sec, min instances |
| Dedicated (App Service plan) | no (Always On needed) | none w/ Always On | plan size | 30 min default, unbounded | yes | plan VM cost |
| Container Apps (Functions on ACA) | optional | depends | KEDA-based | n/a | yes | ACA pricing |

All numbers: verify in current docs. Also note retirement notice for Linux Consumption - verify in current docs and prefer Flex Consumption for new serverless apps.

**Difficulty ladder**

### 5.1 [Easy] Name the Functions hosting plans and the main trade-off.
**Answer**
Consumption, Flex Consumption, Premium, Dedicated (App Service) and Container Apps hosting. The trade-off is cost model vs. cold start/network features: pay-per-use plans scale to zero but cold start; Premium/Dedicated keep warm instances and VNet at a baseline cost.
**Explanation**
Pick based on: need for VNet/private endpoints, latency requirements, execution duration, scale ceiling, and cost predictability.
**Code / CLI / Bicep**
```bash
az functionapp plan create -g rg -n plan-ep1 --sku EP1 --min-instances 1 --max-burst 10
```

### 5.2 [Follow-up] What is cold start and how do Premium/Flex mitigate it?
**Answer**
Cold start is the delay when a new instance must be allocated, the host started and your code loaded after zero or insufficient instances. Premium keeps always-ready instances plus pre-warmed instances to absorb scale-out; Flex Consumption provides configurable always-ready instances per trigger group.
**Explanation**
Contributors: platform allocation, runtime start, package mount, JIT/assembly loading, DI/startup work, first outbound connection. Mitigate by trimming startup, ReadyToRun compilation, run-from-package, keeping dependencies light, and warm instances. Pre-warmed instances in Premium are buffer instances already running the host.
**Code / CLI / Bicep**
```bicep
resource plan 'Microsoft.Web/serverfarms@2023-12-01' = {
  name: 'plan-ep1'
  location: resourceGroup().location
  sku: { name: 'EP1', tier: 'ElasticPremium' }
  kind: 'elastic'
  properties: { reserved: true, maximumElasticWorkerCount: 20 }
}
```
(Pre-warmed count is set on the function app site config `preWarmedInstanceCount` - verify in current docs.)

### 5.3 [Medium] Why would you choose Flex Consumption for a new app?
**Answer**
It scales to zero but supports VNet integration, faster scaling, configurable instance memory size, per-function scaling for some triggers, and always-ready instances. It is the preferred serverless plan going forward (verify current region/feature availability).
**Explanation**
Flex uses a different deployment model (blob container deployment storage, one-deploy), no slots (verify), and its own configuration for instance memory (2048/4096 MB), maximum instance count, always-ready per group, and HTTP concurrency. Check feature gaps (e.g. slots, certain triggers) before committing.
**Code / CLI / Bicep**
```bicep
resource app 'Microsoft.Web/sites@2023-12-01' = {
  name: 'func-flex'
  location: resourceGroup().location
  kind: 'functionapp,linux'
  properties: {
    serverFarmId: flexPlan.id
    functionAppConfig: {
      deployment: { storage: { type: 'blobContainer', value: '${stg.properties.primaryEndpoints.blob}deploy', authentication: { type: 'SystemAssignedIdentity' } } }
      scaleAndConcurrency: { maximumInstanceCount: 100, instanceMemoryMB: 2048 }
      runtime: { name: 'dotnet-isolated', version: '8.0' }
    }
  }
  identity: { type: 'SystemAssigned' }
}
```
(Schema details - verify in current docs.)

### 5.4 [Hard] Timer-based "keep warm" ping vs always-ready instances: which and why?
**Answer**
A timer ping is a hack: it only warms one instance, does nothing for scale-out cold starts, and on Consumption can't be guaranteed. Always-ready (Flex) or Premium minimum instances are the supported solution; Dedicated needs Always On.
**Explanation**
Always On (Dedicated) pings to prevent idle unloading; it is not available/needed on Consumption/Premium. A warmer burns executions and masks, rather than fixes, startup cost. Measure the real cold start first.
**Code / CLI / Bicep**
```bash
az functionapp config set -g rg -n func --always-on true          # Dedicated
az functionapp update -g rg -n func --set siteConfig.minimumElasticInstanceCount=1  # Premium (verify)
```
**Gotchas / follow-ups**
- Premium min instances are billed 24x7.
- Windows Consumption vs Linux differences in cold start and scaling limits.

**Interview traps**
- Believing Premium never cold starts (scale beyond pre-warmed can).
- Assuming classic Consumption supports VNet.
- Ignoring plan retirement/preview notices - always verify.
- Forgetting storage account is needed in all plans.

**Cheat sheet**
- Consumption: cheapest, cold start, limited networking.
- Flex: serverless + VNet + always-ready.
- Premium: warm, VNet, long timeouts, pay baseline.
- Dedicated: Always On, predictable, shared plan.
- ACA: container-first, KEDA scaling.

---

## Topic 6: Scale controller, instance scaling, concurrency and host.json

**Study notes**

The **scale controller** (platform service) monitors the event source (queue length, Service Bus message backlog/age, Event Hub lag, HTTP concurrency) and adds/removes host instances. Within an instance, **concurrency settings** decide how many events run in parallel. Total parallelism = instances x per-instance concurrency.

```
 backlog --> scale controller --> N instances --> each runs up to maxConcurrentCalls (SB)
                                                  / batchSize+newBatchThreshold (Queue)
                                                  / maxEventBatchSize (Event Hubs)
```

| Setting (host.json) | Applies to |
|---|---|
| `extensions.serviceBus.maxConcurrentCalls` | Service Bus (per instance); default 16 (verify) |
| `extensions.serviceBus.maxConcurrentSessions` | session-enabled entities |
| `extensions.queues.batchSize` / `newBatchThreshold` | Storage queue; default 16 / batchSize/2 |
| `extensions.eventHubs.maxEventBatchSize`, `prefetchCount` | Event Hubs |
| `concurrency.dynamicConcurrencyEnabled` | dynamic concurrency (supported triggers: Blob, Queue, Service Bus - verify) |
| `functionTimeout` | max execution time |
| app setting `WEBSITE_MAX_DYNAMIC_APPLICATION_SCALE_OUT` | cap scale-out (Consumption); Flex uses `maximumInstanceCount` |

**Difficulty ladder**

### 6.1 [Easy] What does the scale controller do?
**Answer**
It watches trigger-specific metrics and decides to add or remove function app instances. It's not your code and not the host - it's part of the platform.
**Explanation**
Each trigger type has its own scaling logic (queue length/age, Service Bus active messages, Event Hubs partition lag, HTTP concurrency). Event Hubs and Service Bus scaling is bounded by partitions/sessions and entity characteristics.
**Code / CLI / Bicep**
```bash
az functionapp config appsettings set -g rg -n func --settings WEBSITE_MAX_DYNAMIC_APPLICATION_SCALE_OUT=10
```

### 6.2 [Follow-up] How do you throttle a Service Bus triggered function that overloads a database?
**Answer**
Lower `maxConcurrentCalls`, cap instance count (`WEBSITE_MAX_DYNAMIC_APPLICATION_SCALE_OUT` or Flex max instances), and/or use dynamic concurrency. Remember total concurrency = instances x maxConcurrentCalls.
**Explanation**
Setting `maxConcurrentCalls` to 1 only limits per instance; with 10 instances you still have 10 concurrent. For strict global limits use sessions, a single instance cap, or a semaphore/rate limiter in an external store.
**Code / CLI / Bicep**
```json
{
  "version": "2.0",
  "extensions": {
    "serviceBus": { "maxConcurrentCalls": 4, "prefetchCount": 0 }
  }
}
```

### 6.3 [Medium] What is dynamic concurrency and when should you prefer it?
**Answer**
The host automatically adjusts per-instance concurrency based on CPU/memory and trigger health, instead of fixed values. It's useful when you don't know the right number; fixed values remain better when downstream limits are known.
**Explanation**
Enable with `concurrency.dynamicConcurrencyEnabled`. When enabled, static settings like `maxConcurrentCalls` are ignored for supported triggers (verify supported extension versions in current docs). Snapshot state is stored in storage.
**Code / CLI / Bicep**
```json
{ "version": "2.0", "concurrency": { "dynamicConcurrencyEnabled": true, "snapshotPersistenceEnabled": true } }
```

### 6.4 [Hard] You changed host.json but nothing changed in Azure. Why?
**Answer**
host.json is read at host start and is part of the deployed package; changing it requires redeploying and a host restart. Alternatively, you can override any host.json setting with an app setting using `__` as the separator, e.g. `AzureFunctionsJobHost__extensions__serviceBus__maxConcurrentCalls`.
**Explanation**
Common causes: deployed an older package, wrong path (host.json missing from output because `CopyToOutputDirectory` not set), slot swapped back, or run-from-package serving a stale zip. Check via Kudu/`az functionapp deployment` and the host startup logs which print effective config.
**Code / CLI / Bicep**
```bash
az functionapp config appsettings set -g rg -n func \
  --settings "AzureFunctionsJobHost__extensions__serviceBus__maxConcurrentCalls=8"
```
**Gotchas / follow-ups**
- App setting change restarts the app.
- Wrong JSON shape silently ignored in some sections - validate in startup logs.

### 6.5 [Tricky] Scaled out, the function hammers a downstream API and gets 429s. Fix?
**Answer**
Cap scale-out and concurrency, add retry with backoff honouring `Retry-After`, use a shared rate limiter, and buffer via a queue so the consumer controls the drain rate. Also check SNAT port exhaustion for outbound connections.
**Explanation**
SNAT exhaustion arises from creating `HttpClient` per call; use `IHttpClientFactory`/static client, or NAT Gateway / VNet integration with private endpoints. Rate limits per instance scale linearly with instance count.
**Code / CLI / Bicep**
```csharp
builder.Services.AddHttpClient("api").AddStandardResilienceHandler(); // Microsoft.Extensions.Http.Resilience
```

**Interview traps**
- Saying `maxConcurrentCalls` is a global limit.
- Forgetting prefetch interactions with lock duration (see Topic 8).
- Believing host.json edits apply live in Azure.
- Ignoring partition count as scale ceiling for Event Hubs.

**Cheat sheet**
- Parallelism = instances x per-instance concurrency.
- Cap instances: `WEBSITE_MAX_DYNAMIC_APPLICATION_SCALE_OUT` / Flex max count / Premium max burst.
- Override host.json via `AzureFunctionsJobHost__...`.
- Dynamic concurrency: opt-in, supported triggers only.

---

## Topic 7: Blob trigger - polling vs Event Grid

**Study notes**

| | Polling (default) | Event Grid-based (`Source = BlobTriggerSource.EventGrid`) |
|---|---|---|
| Latency | can be minutes on Consumption | near real-time |
| Mechanism | scans container + blob receipts in storage | Event Grid subscription pushes BlobCreated events |
| Scale | scan cost grows with container size | efficient for large containers |
| Setup | none | Event Grid subscription to the function's blob extension webhook |

Blob receipts (`azure-webjobs-hosts` container) prevent reprocessing. Failed blobs are retried up to 5 times, then a message goes to `webjobs-blobtrigger-poison` queue.

**Difficulty ladder**

### 7.1 [Easy] How does the blob trigger detect new blobs?
**Answer**
By default it polls the container (log scanning plus blob receipts) and runs the function for blobs without a receipt. It is not instant.
**Explanation**
Blob receipts are stored in the storage account in `azure-webjobs-hosts`; this tracks which blobs were processed by ETag so a modified blob is reprocessed.
**Code / CLI / Bicep**
```csharp
[Function("OnBlob")]
public void Run([BlobTrigger("inbox/{name}", Connection = "StorageConn")] Stream blob, string name) { }
```

### 7.2 [Follow-up] Why switch to Event Grid source and how?
**Answer**
For low latency and scalability on large containers or Consumption plans. Set `Source = BlobTriggerSource.EventGrid` and create an Event Grid subscription for `Microsoft.Storage.BlobCreated` pointing at the blob extension webhook.
**Explanation**
Requires a general-purpose v2 storage account (not blob-only legacy). The webhook endpoint is `/runtime/webhooks/blobs?functionName=Host.Functions.<FunctionName>&code=<blobs_extension key>`. Also works with identity-based Event Grid delivery (verify in current docs).
**Code / CLI / Bicep**
```csharp
[Function("OnBlobEG")]
public void Run([BlobTrigger("inbox/{name}", Source = BlobTriggerSource.EventGrid, Connection = "StorageConn")] Stream blob, string name) { }
```
```bash
az eventgrid system-topic event-subscription create -g rg -n sub-blob --system-topic-name stg-topic \
  --endpoint-type webhook --endpoint "https://func.azurewebsites.net/runtime/webhooks/blobs?functionName=Host.Functions.OnBlobEG&code=<key>" \
  --included-event-types Microsoft.Storage.BlobCreated \
  --subject-begins-with /blobServices/default/containers/inbox/
```

### 7.3 [Medium] Alternative: an Event Grid trigger function receiving the blob event. Differences?
**Answer**
An Event Grid trigger gives you the event metadata (URL, size) rather than the stream; you read the blob via the SDK. It has no blob receipts, so deduplication is on you, but you get at-least-once with Event Grid retries and full control.
**Explanation**
This decouples the work from the blob binding, supports filtering by subject/suffix at the Event Grid side, and plays well with managed identity and dead-lettering on the subscription.
**Code / CLI / Bicep**
```csharp
[Function("OnBlobEvent")]
public async Task Run([EventGridTrigger] EventGridEvent ev, [FromServices] BlobServiceClient svc) { /* parse ev.Data -> url, download */ }
```

### 7.4 [Hard] Blob trigger fires multiple times for the same blob, or never. Why?
**Answer**
Multiple: blob overwritten (new ETag), receipts lost/cleared, multiple function apps sharing the same storage `AzureWebJobsStorage` with the same function name, or Event Grid retry/duplicates. Never: wrong connection/container, Consumption polling delay, Event Grid subscription missing/unauthorized, or blob created before app existed.
**Explanation**
Always design handlers to be idempotent. After 5 failed attempts the blob is poisoned and not retried automatically. Large containers slow polling.
**Code / CLI / Bicep**
```kusto
traces
| where message has "BlobTrigger" or customDimensions.Category startswith "Function.OnBlob"
| order by timestamp desc
```
**Gotchas / follow-ups**
- Processing the blob while the uploader is still writing: use a "done" marker or upload then rename/event on `BlobCreated` after commit (events fire at commit).
- Blob trigger doesn't work with Premium blob-only accounts.

**Interview traps**
- "Blob trigger is real time" (default polling is not).
- Not knowing the 5-retry poison queue behaviour.
- Forgetting idempotency for overwritten blobs.

**Cheat sheet**
- Polling default; Event Grid source for speed/scale.
- Receipts in `azure-webjobs-hosts`.
- Poison: `webjobs-blobtrigger-poison`.
- Use Stream binding for large blobs (avoid `byte[]`).

---

## Topic 8: Service Bus trigger behaviour and settlement

**Study notes**

The trigger uses `ServiceBusProcessor` under the hood in PeekLock mode. Default `autoCompleteMessages: true`: success => Complete; exception => Abandon (delivery count +1); at `MaxDeliveryCount` the message goes to the DLQ. In isolated model you can bind `ServiceBusReceivedMessage` plus `ServiceBusMessageActions` for manual settlement (set `autoCompleteMessages` false).

```
 message --> lock (LockDuration, default 1 min) --> function --> Complete | Abandon | DeadLetter | Defer
                          \_ lock auto-renewed up to maxAutoLockRenewalDuration (default 5 min)
```

| Setting | Notes |
|---|---|
| `maxConcurrentCalls` | per instance |
| `maxAutoLockRenewalDuration` | must exceed processing time |
| `autoCompleteMessages` | default true |
| `maxMessageBatchSize` | for batch triggers (`IsBatched = true`) |
| `sessionIdleTimeout`, `maxConcurrentSessions` | sessions |

**Difficulty ladder**

### 8.1 [Easy] What happens when a Service Bus-triggered function succeeds or throws?
**Answer**
Success: the message is completed (removed). Exception: it is abandoned, becomes available again, delivery count increments, and after MaxDeliveryCount (default 10) it moves to the dead-letter queue.
**Explanation**
This is at-least-once delivery. Functions retry policies (host-level) are separate from Service Bus redelivery (see Topic 10).
**Code / CLI / Bicep**
```csharp
[Function("Consume")]
public void Run([ServiceBusTrigger("orders", Connection = "SbConn")] ServiceBusReceivedMessage msg)
{
    // throw => abandon; return => complete
}
```

### 8.2 [Follow-up] How do you dead-letter or defer manually?
**Answer**
Disable auto-complete and inject `ServiceBusMessageActions`; call `DeadLetterMessageAsync`, `CompleteMessageAsync`, `AbandonMessageAsync` or `DeferMessageAsync` yourself.
**Explanation**
Use this for poison payloads (validation failures) where retrying is pointless - dead-letter with a reason immediately rather than burning 10 deliveries.
**Code / CLI / Bicep**
```csharp
[Function("Consume")]
public async Task Run(
    [ServiceBusTrigger("orders", Connection = "SbConn", AutoCompleteMessages = false)] ServiceBusReceivedMessage msg,
    ServiceBusMessageActions actions)
{
    if (!IsValid(msg))
    {
        await actions.DeadLetterMessageAsync(msg, deadLetterReason: "InvalidPayload", deadLetterErrorDescription: "Schema");
        return;
    }
    await ProcessAsync(msg);
    await actions.CompleteMessageAsync(msg);
}
```

### 8.3 [Medium] How do batch triggers and sessions change things?
**Answer**
With `IsBatched = true` you receive an array of messages per invocation (better throughput); with sessions (`IsSessionsEnabled = true`) messages for a session are processed in order by one consumer at a time. Failure of a batch fails/abandons per your settlement logic - with auto-complete, a thrown exception abandons the whole batch.
**Explanation**
For partial failures in a batch, disable auto-complete and settle each message individually. Session trigger parallelism is `maxConcurrentSessions` x `maxConcurrentCalls` per session (default 1 per session to keep order).
**Code / CLI / Bicep**
```csharp
[Function("Batch")]
public async Task Run(
    [ServiceBusTrigger("orders", Connection = "SbConn", IsBatched = true, AutoCompleteMessages = false)] ServiceBusReceivedMessage[] msgs,
    ServiceBusMessageActions actions)
{
    foreach (var m in msgs)
    {
        try { await ProcessAsync(m); await actions.CompleteMessageAsync(m); }
        catch { await actions.AbandonMessageAsync(m); }
    }
}
```

### 8.4 [Hard] Messages are processed twice / `MessageLockLostException`. Why?
**Answer**
Processing exceeded the lock (and auto-renewal ceiling `maxAutoLockRenewalDuration`) or the instance crashed/recycled, so the broker redelivered. It also happens with prefetch holding locks while the instance is busy.
**Explanation**
Fix: raise `maxAutoLockRenewalDuration` above the max processing time, lower prefetch/concurrency, shorten work or offload long work (Durable/queue + claim-check), and make handlers idempotent (dedupe by MessageId / business key). Also, Service Bus duplicate detection only dedupes at send time.
**Code / CLI / Bicep**
```json
{ "version": "2.0", "extensions": { "serviceBus": { "maxAutoLockRenewalDuration": "00:10:00", "maxConcurrentCalls": 8, "prefetchCount": 0 } } }
```
**Gotchas / follow-ups**
- Lock renewal needs the host alive; a scale-in mid-processing causes redelivery.
- Settling after lock lost throws; catch and treat as already redelivered.

### 8.5 [Tricky] Everything is fine locally but identity-based Service Bus trigger doesn't fire in Azure.
**Answer**
Usual causes: missing app setting `SbConn__fullyQualifiedNamespace`, the identity lacks **Azure Service Bus Data Receiver** (trigger also needs Receiver; sending needs **Data Sender**; some scenarios need Data Owner only for management operations), or RBAC propagation delay. Also the scale controller needs its own access: on Consumption/Flex it uses the app's identity - verify required roles/settings in current docs.
**Explanation**
For a UAMI add `SbConn__credential=managedidentity` and `SbConn__clientId=<uami client id>`. Check Diagnostic logs / `traces` for 401/Unauthorized.
**Code / CLI / Bicep**
```bash
az functionapp config appsettings set -g rg -n func --settings \
  "SbConn__fullyQualifiedNamespace=mybus.servicebus.windows.net" \
  "SbConn__credential=managedidentity" "SbConn__clientId=<uami-client-id>"
az role assignment create --assignee <principalId> --role "Azure Service Bus Data Receiver" --scope <sbNamespaceId>
```

**Interview traps**
- Thinking retry policy and MaxDeliveryCount are the same thing.
- Letting a validation failure burn all 10 deliveries.
- Large prefetch + long processing => lock expiry.
- Using Contributor role (management plane) instead of a Data role.

**Cheat sheet**
- Default: success=complete, throw=abandon -> DLQ at MaxDeliveryCount.
- Manual settlement: `AutoCompleteMessages=false` + `ServiceBusMessageActions`.
- Lock duration vs `maxAutoLockRenewalDuration`.
- Batch/sessions available; idempotency always.

---

## Topic 9: Timer trigger, singleton and multi-instance issues

**Study notes**

Schedule uses **NCRONTAB** with six fields: `{second} {minute} {hour} {day} {month} {day-of-week}`, e.g. `0 */5 * * * *` = every 5 minutes. Default time zone is UTC; set `WEBSITE_TIME_ZONE` (Windows names on Windows, IANA on Linux) - not supported on Consumption Linux in some cases (verify). Timer uses a **singleton lock** (blob lease in the storage account) so only one instance runs a given timer at a time - per function app + storage account + function name.

`TimerInfo.IsPastDue` is true when the app was down at the scheduled time. `RunOnStartup` runs on every host start - dev only. `UseMonitor` (default true) persists schedule status in storage.

**Difficulty ladder**

### 9.1 [Easy] Write a timer function that runs every day at 02:30 UTC.
**Answer**
`[TimerTrigger("0 30 2 * * *")]` - six-field NCRONTAB, seconds first.
**Explanation**
Common error is using 5-field Linux cron. Use `%Schedule%` for per-environment values.
**Code / CLI / Bicep**
```csharp
[Function("Nightly")]
public async Task Run([TimerTrigger("0 30 2 * * *")] TimerInfo timer, FunctionContext ctx)
{
    ctx.GetLogger("Nightly").LogInformation("PastDue={Past}", timer.IsPastDue);
}
```

### 9.2 [Follow-up] If my app has 5 instances, does the timer run 5 times?
**Answer**
No. A blob lease singleton lock in the storage account ensures one instance runs each timer occurrence, even when scaled out.
**Explanation**
The lock is keyed on the function and the storage account/host ID. So multiple *different* function apps using the same storage account *and same host ID* can conflict; set a unique `AzureFunctionsWebHost__hostid`/ host ID per app (verify exact setting name in current docs).
**Code / CLI / Bicep**
```bash
az storage blob list --account-name stdemo -c azure-webjobs-hosts --auth-mode login --prefix locks
```

### 9.3 [Medium] What do `IsPastDue`, `RunOnStartup` and `UseMonitor` do?
**Answer**
`IsPastDue` tells you the execution is late (e.g. after downtime). `RunOnStartup` triggers a run at each host start (including scale-out/restarts) so use only in development. `UseMonitor` stores schedule status so missed runs are detected after restarts.
**Explanation**
Without a persisted schedule status, a restart just waits for the next occurrence. With monitor on, a missed occurrence triggers one catch-up with `IsPastDue = true`.
**Code / CLI / Bicep**
```csharp
[Function("Report")]
public void Run([TimerTrigger("0 0 * * * *", RunOnStartup = false, UseMonitor = true)] TimerInfo t)
{
    if (t.IsPastDue) { /* decide whether to run late or skip */ }
}
```

### 9.4 [Hard] The timer ran twice, or ran at the wrong hour. Why?
**Answer**
Double-run causes: two function apps/slots sharing storage and host ID, a slot (staging) also running the timer, overlapping execution because one run exceeded the interval, or `RunOnStartup` on restart. Wrong hour: UTC vs local zone and DST.
**Explanation**
Slots: both production and staging slots run triggers unless disabled - disable the timer in non-prod slots using `AzureWebJobs.<FunctionName>.Disabled=true` as a slot-sticky setting. Overlap: singleton prevents concurrent runs of the same timer only per scheduled firing; a long run may cause the next occurrence to start after completion (`IsPastDue`). For true mutual exclusion in long jobs use a lease (blob lease/Redis lock).
**Code / CLI / Bicep**
```bash
az functionapp config appsettings set -g rg -n func --slot staging \
  --slot-settings "AzureWebJobs.Nightly.Disabled=true"
```
**Gotchas / follow-ups**
- Running the app locally with the same `AzureWebJobsStorage` as prod steals/competes for the lock - use Azurite locally.
- DST: for time-zone-aware schedules set `WEBSITE_TIME_ZONE`.

### 9.5 [Tricky] How do you make the job idempotent for the same scheduled time?
**Answer**
Use the scheduled occurrence as a key (e.g. `timer.ScheduleStatus.Last` truncated to the period) and write a marker (Table/blob/DB unique key) before doing work, so a duplicate run detects completion and exits.
**Explanation**
Timer provides at-least-once-ish semantics under failure/restart; no exactly-once guarantee. Persist outcome with a unique constraint.
**Code / CLI / Bicep**
```csharp
var key = $"nightly-{DateTime.UtcNow:yyyyMMdd}";
try { await table.AddEntityAsync(new TableEntity("job", key)); }
catch (RequestFailedException e) when (e.Status == 409) { return; } // already done
```

**Interview traps**
- 5-field cron.
- "Timer runs on every instance" - singleton lock.
- Leaving `RunOnStartup=true` in production.
- Staging slot running the same timer.

**Cheat sheet**
- NCRONTAB: `s m h d M dow`.
- Singleton via blob lease in storage.
- `WEBSITE_TIME_ZONE` for local time.
- Disable via `AzureWebJobs.<Name>.Disabled`.

---

## Topic 10: Retries, poison messages, dead-lettering, idempotency, error handling and logging

**Study notes**

Layers of retry (don't confuse them):

| Layer | Applies to | Mechanism |
|---|---|---|
| Functions retry policy | Timer, Kafka, Event Hubs, Cosmos DB, (others via extension) | `fixedDelay` / `exponentialBackoff`, set in host.json or `[FixedDelayRetry]`/`[ExponentialBackoffRetry]` - verify supported triggers |
| Trigger-native redelivery | Service Bus (MaxDeliveryCount -> DLQ), Storage Queue (maxDequeueCount 5 -> `<queue>-poison`), Blob (5 tries -> poison queue) | built into source |
| Event Grid delivery retries | Event Grid trigger | subscription retry/dead-letter settings |
| In-code (Polly/resilience) | downstream calls | your handler |

Do not combine a Functions retry policy with Service Bus native redelivery on the same trigger without thought - Service Bus trigger doesn't use the Functions retry policy (it relies on the broker).

**Difficulty ladder**

### 10.1 [Easy] What is a poison message?
**Answer**
A message that repeatedly fails processing. After the max delivery attempts it is moved aside (Storage queue: `<name>-poison`; Service Bus: dead-letter queue) so it stops blocking the flow.
**Explanation**
Storage queue default `maxDequeueCount` is 5. Service Bus `MaxDeliveryCount` is configured on the entity (default 10). Someone must monitor and reprocess the poison/DLQ.
**Code / CLI / Bicep**
```json
{ "version": "2.0", "extensions": { "queues": { "maxDequeueCount": 5, "visibilityTimeout": "00:00:30" } } }
```

### 10.2 [Follow-up] How do you configure a retry policy?
**Answer**
Via attribute or host.json: fixed delay (`maxRetryCount`, `delayInterval`) or exponential backoff (`minimumInterval`, `maximumInterval`). Applies only to supported triggers; retries run in-process on the same instance.
**Explanation**
Max retry count -1 means infinite (dangerous). Retry context is available via `FunctionContext.RetryContext` (`RetryCount`, `MaxRetryCount`) for logic like "dead-letter on last attempt".
**Code / CLI / Bicep**
```csharp
[Function("Hub")]
[ExponentialBackoffRetry(5, "00:00:04", "00:15:00")]
public void Run([EventHubTrigger("hub", Connection = "EhConn")] string[] events, FunctionContext ctx)
{
    var rc = ctx.RetryContext; // RetryCount, MaxRetryCount
}
```
```json
{ "version": "2.0", "retry": { "strategy": "fixedDelay", "maxRetryCount": 3, "delayInterval": "00:00:05" } }
```

### 10.3 [Medium] Design idempotent processing for at-least-once triggers.
**Answer**
Derive a deterministic idempotency key (MessageId or business key), make writes upserts/conditional, or record processed keys in a store with a unique constraint checked in the same transaction as the side effect. Avoid relying on trigger "exactly once".
**Explanation**
Techniques: natural idempotent operations (PUT/upsert), inbox/dedup table, ETag/If-Match, outbox pattern for publishing. Side effects to external systems (email, payments) need an idempotency key supported by that system.
**Code / CLI / Bicep**
```csharp
// SQL: unique key on MessageId makes duplicates fail fast
await db.ExecuteAsync(
  "INSERT INTO Inbox(MessageId) VALUES (@id)", new { id = msg.MessageId });  // throws on duplicate -> treat as already processed
```

### 10.4 [Hard] Where should exceptions be caught, and how do you log without losing retry behaviour?
**Answer**
Catch only to add context or handle expected/permanent errors (then dead-letter/complete deliberately); rethrow transient ones so the trigger's retry/redelivery works. Swallowing an exception completes the message and loses it.
**Explanation**
Use structured logging with message templates and scopes (invocation id, message id). Use middleware for global logging (Topic 2). Log levels are controlled in host.json `logging.logLevel`, and sampling affects what reaches App Insights (Topic 16).
**Code / CLI / Bicep**
```csharp
try { await ProcessAsync(msg); }
catch (ValidationException ex)  // permanent
{
    log.LogWarning(ex, "Invalid message {MessageId}", msg.MessageId);
    await actions.DeadLetterMessageAsync(msg, deadLetterReason: "Validation", deadLetterErrorDescription: ex.Message);
}
catch (HttpRequestException ex)  // transient
{
    log.LogError(ex, "Transient failure for {MessageId}", msg.MessageId);
    throw;
}
```

### 10.5 [Tricky] Infinite poison loop with a retry policy of -1 on Event Hubs.
**Answer**
Event Hubs has no DLQ; with infinite retry a bad event blocks its partition forever (checkpoint never advances). Use a bounded retry, then catch/log and send the event to a dead-letter store yourself and continue.
**Explanation**
Event Hubs processing is checkpoint-based: if the function fails after retries, the host moves on (with finite retries), losing the event unless you captured it. Design: try/catch per event in a batch, write failures to a "dead-letter" blob/queue, and return success.
**Code / CLI / Bicep**
```csharp
foreach (var e in events)
{
    try { await ProcessAsync(e); }
    catch (Exception ex) { await deadLetterQueue.SendMessageAsync(e); log.LogError(ex, "Parked event"); }
}
```
**Gotchas / follow-ups**
- Functions retry for Service Bus/Storage queue triggers: don't set `retry` expecting it to apply (verify support matrix in current docs).
- Retry re-runs the full function; non-idempotent steps repeat.

**Interview traps**
- Mixing up retry policy vs broker redelivery.
- `catch { }` swallowing exceptions.
- Infinite retry on streams.
- No monitoring on poison/DLQ.

**Cheat sheet**
- Storage queue: 5 dequeues -> poison; Service Bus: MaxDeliveryCount -> DLQ.
- Retry policy: fixedDelay / exponentialBackoff, supported triggers only.
- Idempotency key + unique constraint.
- Rethrow transient, dead-letter permanent.

---

## Topic 11: Durable Functions

**Study notes**

Durable Functions is an extension for stateful workflows in code, built on the Durable Task Framework with event sourcing and replay.

| Function type | Role | Rules |
|---|---|---|
| Client (starter) | `[DurableClient]`, starts/queries/terminates instances | any trigger |
| Orchestrator | coordinates workflow | deterministic, no I/O, replayed |
| Activity | does real work | normal code, at-least-once |
| Entity | small stateful actor | operations processed serially per entity key |

Patterns: function chaining, fan-out/fan-in, async HTTP API, monitor, human interaction (external events + timers), aggregator (entities).

Replay: orchestrator re-executes from the start; completed steps are replayed from history without re-running. Therefore orchestrator code must be deterministic.

Storage providers (task hub): Azure Storage (default), Netherite, MSSQL, and the Durable Task Scheduler (newer managed backend) - verify availability/status in current docs.

**Difficulty ladder**

### 11.1 [Easy] What are the Durable function types?
**Answer**
Client functions start and manage orchestrations; orchestrators define the workflow; activities perform the actual work; entities hold small pieces of state accessed via operations.
**Explanation**
Orchestrators only schedule activities/sub-orchestrations/timers and `await` results. State is persisted in history (task hub tables/queues for the default provider).
**Code / CLI / Bicep**
```csharp
[Function("Start")]
public async Task<HttpResponseData> Start(
    [HttpTrigger(AuthorizationLevel.Function, "post")] HttpRequestData req,
    [DurableClient] DurableTaskClient client)
{
    string id = await client.ScheduleNewOrchestrationInstanceAsync(nameof(Orchestrate), input: "order-1");
    return await client.CreateCheckStatusResponseAsync(req, id);
}

[Function(nameof(Orchestrate))]
public async Task<string> Orchestrate([OrchestrationTrigger] TaskOrchestrationContext ctx)
{
    var a = await ctx.CallActivityAsync<string>(nameof(Step1), ctx.GetInput<string>());
    return await ctx.CallActivityAsync<string>(nameof(Step2), a);
}

[Function(nameof(Step1))]
public string Step1([ActivityTrigger] string input) => input + "-1";
[Function(nameof(Step2))]
public string Step2([ActivityTrigger] string input) => input + "-2";
```

### 11.2 [Follow-up] Show fan-out/fan-in and the async HTTP API pattern.
**Answer**
Fan-out: start many activities without awaiting, then `Task.WhenAll`. Async HTTP API: the starter returns 202 with status-query URLs (`CreateCheckStatusResponseAsync`) and clients poll.
**Explanation**
Fan-out parallelism is bounded by concurrency settings (`durableTask.maxConcurrentActivityFunctions`) and the scale of the app. Large fan-outs create big histories - consider sub-orchestrations or batching.
**Code / CLI / Bicep**
```csharp
var tasks = items.Select(i => ctx.CallActivityAsync<int>(nameof(Process), i));
int[] results = await Task.WhenAll(tasks);
return results.Sum();
```

### 11.3 [Medium] Human interaction and monitor patterns - how do timers and external events work?
**Answer**
Use `ctx.CreateTimer` and `ctx.WaitForExternalEvent<T>` and race them with `Task.WhenAny`; cancel the timer if the event wins. Monitor pattern loops with durable timers until a condition is met.
**Explanation**
Always use `ctx.CreateTimer` (durable) - never `Task.Delay`/`Thread.Sleep`. Timers cost nothing while waiting (instance unloaded). Raise events with `client.RaiseEventAsync`.
**Code / CLI / Bicep**
```csharp
using var cts = new CancellationTokenSource();
var timeout = ctx.CreateTimer(ctx.CurrentUtcDateTime.AddHours(24), cts.Token);
var approval = ctx.WaitForExternalEvent<bool>("Approval");
var winner = await Task.WhenAny(approval, timeout);
if (winner == approval) { cts.Cancel(); /* approved */ } else { /* escalate */ }
```

### 11.4 [Hard] What are the determinism constraints and why do they exist?
**Answer**
Because the orchestrator is replayed from its history, it must produce the same sequence of operations each time. So don't use `DateTime.UtcNow`, random/`Guid.NewGuid`, I/O, env vars that can change, or non-durable async (`Task.Run`, `Task.Delay`); use `ctx.CurrentUtcDateTime`, `ctx.NewGuid()`, and activities for I/O.
**Explanation**
Non-deterministic code causes `NonDeterministicOrchestrationException`. Logging in orchestrators repeats on replay - use `ctx.CreateReplaySafeLogger`. Avoid `await` on non-durable tasks. Inputs/outputs must be serializable and small (history size).
**Code / CLI / Bicep**
```csharp
var log = ctx.CreateReplaySafeLogger(nameof(Orchestrate));
log.LogInformation("Started at {Time}", ctx.CurrentUtcDateTime);   // replay-safe
var id = ctx.NewGuid();                                             // deterministic
```
**Gotchas / follow-ups**
- Activities are at-least-once: make them idempotent.
- Orchestrator should not call `HttpClient` directly (use an activity or the built-in `CallHttpAsync` in in-process/ verify support in isolated).

### 11.5 [Hard] How do you version orchestrations safely and run eternal orchestrations?
**Answer**
In-flight instances replay old history against new code, so changing orchestrator logic can break them. Options: side-by-side deployment (new task hub name or separate app), versioned orchestrator names, or built-in orchestration versioning features (verify availability in current docs). Eternal orchestrations loop via `ctx.ContinueAsNew(input)` to reset history.
**Explanation**
Safe changes: adding new activities at the end is not always safe; renaming or reordering is not. Use a distinct `taskHub` per version (`host.json: extensions.durableTask.hubName`) to let old instances drain. Eternal pattern avoids unbounded history growth.
**Code / CLI / Bicep**
```csharp
[Function(nameof(Cleaner))]
public async Task Cleaner([OrchestrationTrigger] TaskOrchestrationContext ctx)
{
    await ctx.CallActivityAsync(nameof(DoCleanup));
    await ctx.CreateTimer(ctx.CurrentUtcDateTime.AddHours(1), CancellationToken.None);
    ctx.ContinueAsNew(null);
}
```
```json
{ "version": "2.0", "extensions": { "durableTask": { "hubName": "OrdersV2" } } }
```

### 11.6 [Tricky] Two function apps share a storage account and Durable behaves strangely.
**Answer**
They share the default task hub name and compete for the same control queues/history, causing misrouted or stolen work. Give each app a unique `hubName` (or separate storage accounts).
**Explanation**
Task hub names must be alphanumeric, start with a letter, 3-45 chars. Slots too: staging and production using the same task hub and storage will both process work - use slot-sticky `hubName` settings. Choose provider by need: Netherite for high throughput, MSSQL for SQL-based/portable - verify current support status.
**Code / CLI / Bicep**
```json
{ "version": "2.0", "extensions": { "durableTask": { "hubName": "%TaskHubName%" } } }
```

**Interview traps**
- `DateTime.UtcNow` or `Task.Delay` in orchestrators.
- Calling activities as if exactly-once.
- Huge payloads in inputs/outputs (history bloat).
- Forgetting unique task hub per app/slot.

**Cheat sheet**
- Client / Orchestrator / Activity / Entity.
- Deterministic: `ctx.CurrentUtcDateTime`, `ctx.NewGuid()`, `ctx.CreateTimer`.
- `Task.WhenAll` fan-in; `WhenAny` + timer for timeouts.
- `ContinueAsNew` for eternal; version via task hub/naming.
- Providers: Azure Storage, Netherite, MSSQL, (Durable Task Scheduler - verify).

---

## Topic 12: Durable vs Logic Apps; Functions vs WebJobs vs Logic Apps vs Container Apps jobs

**Study notes**

| Need | Durable Functions | Logic Apps (Standard/Consumption) |
|---|---|---|
| Authoring | code (C#) | designer / JSON workflow |
| Connectors | write yourself (SDKs) | 1000+ managed connectors |
| Complex logic/testing | strong (unit tests, debugging) | weaker for heavy logic |
| Ops visibility | App Insights, instance APIs | run history UI per action |
| Cost model | compute + storage | per action (Consumption) or plan (Standard) |
| Best for | developer-owned, complex orchestrations | integration, B2B, SaaS glue |

| Compute option | Pick when |
|---|---|
| Functions | event-driven code snippets |
| WebJobs (SDK / App Service) | legacy continuous/triggered background tasks on an App Service plan; Functions is built on the WebJobs SDK |
| Logic Apps | low-code workflows & connectors |
| Container Apps jobs | containerized batch/scheduled/event (KEDA) jobs, long-running tasks, any language |

**Difficulty ladder**

### 12.1 [Easy] Durable Functions vs Logic Apps in one sentence each.
**Answer**
Durable Functions is code-first stateful orchestration with full control; Logic Apps is low-code workflow with a large connector library and run-history UI.
**Explanation**
They are complementary: a Logic App can call a Function for custom logic, and a Function can call connectors via SDKs. Pick by team skill, connector needs, and complexity.
**Code / CLI / Bicep**
```text
SaaS/B2B glue, approvals via Outlook/Teams, mapping -> Logic Apps
Complex branching, loops, custom retry/compensation, unit tested -> Durable
```

### 12.2 [Follow-up] When would you pick a Container Apps job over a Function?
**Answer**
For work exceeding Function timeouts, needing custom runtimes/containers, high CPU/memory, or run-to-completion batch semantics. Container Apps jobs support manual, scheduled and event-driven triggers (KEDA scalers).
**Explanation**
Functions are optimized for short events and rich bindings. Jobs give you a container run with retries (`replicaRetryLimit`) and `replicaTimeout`. Note you also can host Functions in Container Apps.
**Code / CLI / Bicep**
```bash
az containerapp job create -g rg -n job-etl --environment cae-prod \
  --trigger-type Schedule --cron-expression "0 2 * * *" \
  --image myacr.azurecr.io/etl:1 --replica-timeout 3600 --replica-retry-limit 2
```

### 12.3 [Medium] How do WebJobs relate to Functions?
**Answer**
The Azure Functions host is built on the WebJobs SDK (the same trigger/binding model). WebJobs run as background processes inside an App Service app, are tied to that plan, and lack Functions' serverless scaling and some tooling.
**Explanation**
Legacy apps often have WebJobs for background work; migrating to Functions (or Container Apps jobs) enables independent scaling and deployment. Continuous WebJobs scale with the web app instances unless singleton.
**Code / CLI / Bicep**
```text
WebJobs: App Service plan, Always On needed for continuous, shares CPU/memory with web app.
```

### 12.4 [Hard] Pick the right tool: a 3-hour nightly ETL, a 10 KB message transformation, and an approval workflow with Outlook.
**Answer**
ETL: Container Apps job (or Durable fan-out on Premium/Flex with unbounded timeout if code-friendly, or Data Factory/Batch). Message transformation: Function with Service Bus trigger. Approval via Outlook: Logic Apps (connector + built-in approval), or Durable human-interaction if code-owned.
**Explanation**
Justify with limits (timeouts, scale ceilings), cost, operability, and team skills. Mention hybrid: Logic App orchestrating, Functions for compute-heavy steps.
**Code / CLI / Bicep**
```text
Decision matrix: duration | connectors needed | ownership | cost per run | observability
```
**Gotchas / follow-ups**
- Logic Apps Consumption bills per action; high-volume loops get expensive.
- Durable history storage costs and transactions on Azure Storage provider.

**Interview traps**
- "Logic Apps is just for non-developers."
- Using Functions for multi-hour jobs on Consumption.
- Forgetting Functions-on-WebJobs-SDK lineage.

**Cheat sheet**
- Durable = code orchestration; Logic Apps = connector workflows.
- Container Apps jobs for long/containerized batch.
- WebJobs = legacy, plan-bound.
- Combine them: Logic App -> Function -> Service Bus.

---

## Topic 13: Configuration, Key Vault references, identity-based connections, SAMI vs UAMI

**Study notes**

Config sources: `local.settings.json` (local only, never deployed, gitignore it), app settings (environment variables in Azure), `host.json` (runtime behaviour, deployed). Slot settings ("deployment slot setting") stay with the slot on swap.

**Key Vault reference** as app setting value: `@Microsoft.KeyVault(SecretUri=https://kv.vault.azure.net/secrets/name/)` or `@Microsoft.KeyVault(VaultName=kv;SecretName=name)`. The app's identity needs **Key Vault Secrets User** (RBAC vault) or get-secret access policy. Unversioned references refresh periodically (verify interval in current docs); `keyVaultReferenceIdentity` selects the identity (needed for UAMI).

**Identity-based connections** (no secrets):

| Connection | Settings |
|---|---|
| Host storage | `AzureWebJobsStorage__accountName` (+ optionally `__blobServiceUri`, `__queueServiceUri`, `__tableServiceUri`) |
| Service Bus | `<Conn>__fullyQualifiedNamespace` |
| Event Hubs | `<Conn>__fullyQualifiedNamespace` |
| Storage queue/blob trigger | `<Conn>__serviceUri` (or `__queueServiceUri`/`__blobServiceUri`) - verify per extension |
| UAMI selection | `<Conn>__credential=managedidentity` + `<Conn>__clientId=<uami client id>` |

Roles for host storage: **Storage Blob Data Owner**, **Storage Queue Data Contributor**, **Storage Table Data Contributor** (Table needed for Durable/ timers? verify in current docs). Service Bus: **Data Receiver / Sender / Owner**.

**Difficulty ladder**

### 13.1 [Easy] What is local.settings.json and how does it relate to Azure app settings?
**Answer**
It holds local dev settings read by Core Tools under `Values`; it is not deployed. In Azure the same keys must be set as app settings.
**Explanation**
Nested config uses `__` or `:` separators (`A__B` in app settings maps to `A:B`). Keep secrets out of git; use Key Vault locally via `az login` credentials.
**Code / CLI / Bicep**
```json
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "FUNCTIONS_WORKER_RUNTIME": "dotnet-isolated",
    "SbConn__fullyQualifiedNamespace": "mybus.servicebus.windows.net"
  }
}
```

### 13.2 [Follow-up] How do Key Vault references work and what can go wrong?
**Answer**
The platform resolves the reference at app start using the app's identity and injects the secret as the setting value. Failures leave the raw reference string or an error status, usually due to missing RBAC, firewall/private network blocking the vault, wrong secret URI, or UAMI not specified.
**Explanation**
Check the portal's Configuration blade (green check / red X) or the `Microsoft.Web/sites/config/configreferences` API. If the vault has a firewall, the function needs VNet integration with routing set so vault traffic goes through the VNet (`vnetRouteAllEnabled`/ WEBSITE_VNET_ROUTE_ALL - verify).
**Code / CLI / Bicep**
```bash
az functionapp config appsettings set -g rg -n func --settings \
  "ApiKey=@Microsoft.KeyVault(SecretUri=https://kv-demo.vault.azure.net/secrets/ApiKey/)"
az functionapp update -g rg -n func --set keyVaultReferenceIdentity=<uami-resource-id>
az role assignment create --assignee <principalId> --role "Key Vault Secrets User" --scope <kvId>
```

### 13.3 [Medium] Configure the trigger to use identity (no connection string) for Service Bus with a UAMI.
**Answer**
Set `SbConn__fullyQualifiedNamespace`, `SbConn__credential=managedidentity` and `SbConn__clientId=<UAMI client id>`, and assign the UAMI a Service Bus Data Receiver/Sender role. For SAMI, only the namespace setting is needed.
**Explanation**
Use the *client id* (not principal/object id, not resource id) in `__clientId`. For host storage use `AzureWebJobsStorage__accountName` and remove `AzureWebJobsStorage` connection string; add `AzureWebJobsStorage__credential` and `__clientId` for UAMI.
**Code / CLI / Bicep**
```bicep
resource app 'Microsoft.Web/sites@2023-12-01' existing = { name: 'func-demo' }
resource settings 'Microsoft.Web/sites/config@2023-12-01' = {
  parent: app
  name: 'appsettings'
  properties: {
    FUNCTIONS_WORKER_RUNTIME: 'dotnet-isolated'
    FUNCTIONS_EXTENSION_VERSION: '~4'
    AzureWebJobsStorage__accountName: stg.name
    AzureWebJobsStorage__credential: 'managedidentity'
    AzureWebJobsStorage__clientId: uami.properties.clientId
    SbConn__fullyQualifiedNamespace: '${sbName}.servicebus.windows.net'
    SbConn__credential: 'managedidentity'
    SbConn__clientId: uami.properties.clientId
  }
}
```
(Merge with existing settings in real templates - this resource replaces the whole appsettings bag.)

### 13.4 [Hard] SAMI vs UAMI with Functions - pros, cons, and gotchas.
**Answer**
SAMI is tied to the app lifecycle, simple, but is recreated (new principal id) when the app is deleted and recreated, and each app/slot has its own. UAMI is reusable across apps/slots, can be pre-granted RBAC before deployment (avoids chicken-and-egg), but requires explicit `__clientId`/`keyVaultReferenceIdentity` when multiple identities exist.
**Explanation**
Slots: each slot has its own SAMI, so staging needs its own role assignments (a classic swap failure); a UAMI shared across slots avoids this. With a single UAMI and no SAMI, `DefaultAzureCredential` in code still needs `ManagedIdentityClientId` (or `AZURE_CLIENT_ID`) set when more than one identity is attached.
**Code / CLI / Bicep**
```csharp
var cred = new DefaultAzureCredential(new DefaultAzureCredentialOptions
{
    ManagedIdentityClientId = Environment.GetEnvironmentVariable("AZURE_CLIENT_ID")
});
```
**Gotchas / follow-ups**
- RBAC propagation can take minutes; transient 403 right after deploy.
- Flex Consumption deployment storage auth can use identity (see Topic 5).
- Local dev: `DefaultAzureCredential` picks Azure CLI/VS creds - ensure your user has the same roles.

### 13.5 [Tricky] Changing an app setting caused downtime / the swap carried the wrong value. Why?
**Answer**
Any app setting change restarts the app. On swap, non-sticky settings move with the code; slot (sticky) settings stay with the slot. Mark environment-specific values (connection names, task hub, disabled flags) as slot settings.
**Explanation**
Swap with preview applies target slot settings to the source slot first, lets you validate, then completes. Settings that restart the app (like `FUNCTIONS_WORKER_RUNTIME`) in a swap are a risk - test in staging.
**Code / CLI / Bicep**
```bash
az functionapp config appsettings set -g rg -n func --slot staging --slot-settings "TaskHubName=OrdersStaging"
```

**Interview traps**
- Using `__clientId` with the principal (object) id.
- Granting management-plane roles (Contributor) instead of data roles.
- Forgetting that each slot's SAMI is separate.
- Putting secrets in local.settings.json committed to git.

**Cheat sheet**
- `__` = nested config separator.
- `AzureWebJobsStorage__accountName`, `<Conn>__fullyQualifiedNamespace`, `__credential`, `__clientId`.
- KV ref: `@Microsoft.KeyVault(SecretUri=...)` + Secrets User role.
- UAMI: needs clientId; SAMI: automatic.
- Slot settings stick; others swap.

---

## Topic 14: Deployment and local development

**Study notes**

| Method | Notes |
|---|---|
| Zip deploy / `WEBSITE_RUN_FROM_PACKAGE=1` | app runs directly from read-only zip; atomic, faster cold start |
| `func azure functionapp publish <app>` | Core Tools publish |
| GitHub Actions / Azure Pipelines | `Azure/functions-action`, `AzureFunctionApp@2` |
| Containers | custom image from ACR (Premium/Dedicated/ACA) |
| Slots | Premium/Dedicated (and Consumption Windows legacy with limits); Flex slots - verify |
| Flex Consumption | one deploy to a blob container (`az functionapp deployment source config-zip`/ Core Tools / actions) |

Local: Core Tools (`func start`), **Azurite** emulates Blob/Queue/Table (`UseDevelopmentStorage=true`). Service Bus has no official free local emulator parity historically; a Service Bus emulator exists (Docker) - verify in current docs; else use a dev namespace with your Entra identity.

**Difficulty ladder**

### 14.1 [Easy] How do you create and run a function locally?
**Answer**
Use Core Tools: `func init --worker-runtime dotnet-isolated`, `func new`, then `func start` with Azurite running for storage.
**Explanation**
Core Tools host loads local.settings.json. Timer triggers need storage (Azurite is fine). HTTP triggers listen on port 7071.
**Code / CLI / Bicep**
```bash
func init MyApp --worker-runtime dotnet-isolated --target-framework net8.0
cd MyApp
func new --name Hello --template "HTTP trigger"
azurite --silent --location .azurite &
func start
```

### 14.2 [Follow-up] What does run-from-package do and what are the pitfalls?
**Answer**
With `WEBSITE_RUN_FROM_PACKAGE=1` the app mounts the deployment zip read-only, giving atomic deployments and fewer file-lock issues. The wwwroot becomes read-only, so you can't write files there.
**Explanation**
You can also set it to a URL (blob SAS or identity-accessed) - with a URL the app reads the package at startup; expired SAS breaks cold starts. Prefer deployment via zip push or identity-based URL access (verify in current docs).
**Code / CLI / Bicep**
```bash
dotnet publish -c Release -o publish
Compress-Archive -Path publish\* -DestinationPath app.zip -Force
az functionapp deployment source config-zip -g rg -n func --src app.zip
```

### 14.3 [Medium] Safe releases with slots and swaps.
**Answer**
Deploy to a staging slot, warm and smoke-test it, then swap; the old version stays in the staging slot for quick rollback. Mark slot-specific settings sticky and disable triggers in staging to avoid double processing.
**Explanation**
Triggers in staging slot run actively (queue consumers compete with production!). Use `AzureWebJobs.<fn>.Disabled` on the slot, or use a separate Service Bus subscription/queue per slot. Swap performs warm-up of the source slot via HTTP; non-HTTP functions aren't gated by it.
**Code / CLI / Bicep**
```bash
az functionapp deployment slot create -g rg -n func --slot staging
az functionapp deployment source config-zip -g rg -n func --slot staging --src app.zip
az functionapp deployment slot swap -g rg -n func --slot staging --target-slot production
```

### 14.4 [Hard] CI/CD pipeline for an isolated .NET Functions app with OIDC.
**Answer**
Build and test with `dotnet`, publish, then deploy with `Azure/functions-action` authenticated via `azure/login` using OIDC federated credentials (no stored secrets). Use staging slot + swap for production.
**Explanation**
Federated credential on an app registration/UAMI with subject `repo:org/repo:environment:prod`. The identity needs Website Contributor (or Contributor) on the function app scope.
**Code / CLI / Bicep**
```yaml
permissions: { id-token: write, contents: read }
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with: { dotnet-version: '8.0.x' }
      - run: dotnet publish -c Release -o ./out
      - uses: azure/login@v2
        with: { client-id: ${{ vars.AZURE_CLIENT_ID }}, tenant-id: ${{ vars.AZURE_TENANT_ID }}, subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }} }
      - uses: Azure/functions-action@v1
        with: { app-name: func-demo, slot-name: staging, package: ./out }
```
**Gotchas / follow-ups**
- Pin action versions per current docs.
- Container deployments: set `WEBSITES_ENABLE_APP_SERVICE_STORAGE=false` as applicable.

**Interview traps**
- Staging slot consuming production queue.
- Assuming `local.settings.json` deploys.
- Expired SAS URL for run-from-package.
- Local app sharing prod `AzureWebJobsStorage`.

**Cheat sheet**
- `func start` + Azurite.
- Zip deploy + `WEBSITE_RUN_FROM_PACKAGE`.
- Slots: sticky settings; disable triggers in staging.
- OIDC login in CI.

---

## Topic 15: Testing

**Study notes**

Layers: unit (business logic in plain classes, function is a thin adapter), integration (real emulator/test resources), end-to-end (deployed). In isolated .NET, `FunctionContext` can be mocked, but better: keep functions thin and inject services. For HTTP with ASP.NET Core integration, `HttpRequest` can be a `DefaultHttpContext().Request`.

**Difficulty ladder**

### 15.1 [Easy] How do you unit test an HTTP-triggered function?
**Answer**
Instantiate the function class with mocked dependencies, create a `DefaultHttpContext`, call the method and assert on the `IActionResult`.
**Explanation**
Keep business logic in services so tests don't need the Functions runtime.
**Code / CLI / Bicep**
```csharp
[Fact]
public async Task GetOrder_ReturnsOk()
{
    var svc = Substitute.For<IOrderService>();
    svc.GetAsync(5).Returns(new Order(5));
    var fn = new Orders(svc, NullLogger<Orders>.Instance);
    var ctx = new DefaultHttpContext();

    var result = await fn.Get(ctx.Request, 5);

    Assert.IsType<OkObjectResult>(result);
}
```

### 15.2 [Follow-up] How do you mock `FunctionContext` and `ServiceBusMessageActions`?
**Answer**
`FunctionContext` is an abstract class - mock it with a mocking library (Moq/NSubstitute) or a small fake. `ServiceBusModelFactory.ServiceBusReceivedMessage(...)` builds test messages, and `ServiceBusMessageActions` is mockable (protected/virtual members - verify mocking approach for your SDK version).
**Explanation**
Prefer testing the handler logic via a service method that accepts the message body; only mock actions when verifying settlement.
**Code / CLI / Bicep**
```csharp
var msg = ServiceBusModelFactory.ServiceBusReceivedMessage(
    body: BinaryData.FromString("{\"id\":1}"), messageId: "m1");
var ctx = Substitute.For<FunctionContext>();
```

### 15.3 [Medium] How do you integration test a function?
**Answer**
Run the real host in a test (Core Tools or a `WebApplicationFactory`-style approach is limited for Functions), with Azurite and test doubles/emulators, then trigger through HTTP/queues and assert side effects. Alternatively, test the DI composition by building the same `HostBuilder` with test overrides.
**Explanation**
Practical approach: Testcontainers for Azurite + SQL; start `func start` as a process in a fixture; send messages to a test queue in a dev subscription. There is no first-class `WebApplicationFactory` for Functions isolated workers (verify - community options exist).
**Code / CLI / Bicep**
```bash
docker run -p 10000:10000 -p 10001:10001 -p 10002:10002 mcr.microsoft.com/azure-storage/azurite
func start --port 7072
```

### 15.4 [Hard] Testing Durable orchestrators.
**Answer**
Mock `TaskOrchestrationContext`, set up `CallActivityAsync` returns, call the orchestrator method directly and assert the sequence of activity calls and results. Activities are tested as normal methods.
**Explanation**
This verifies logic but not replay/determinism; add an integration test against the real host to catch non-determinism. Replay-safe logger can be mocked via `CreateReplaySafeLogger`.
**Code / CLI / Bicep**
```csharp
var ctx = Substitute.For<TaskOrchestrationContext>();
ctx.GetInput<string>().Returns("x");
ctx.CallActivityAsync<string>("Step1", "x", Arg.Any<TaskOptions>()).Returns("x-1");
ctx.CreateReplaySafeLogger(Arg.Any<string>()).Returns(NullLogger.Instance);
```
(Overload signatures change - verify with your Durable package version.)
**Gotchas / follow-ups**
- Tests that mock `CallActivityAsync` with the wrong activity name pass vacuously; use string constants/`nameof`.
- Replay and non-determinism are only caught against a real host.

**Interview traps**
- Putting all logic in the function body.
- Assuming unit tests prove determinism.
- Tests running against shared prod storage.

**Cheat sheet**
- Thin functions + services.
- `DefaultHttpContext`, `ServiceBusModelFactory`.
- Azurite for local/integration.
- Mock `TaskOrchestrationContext`.

---

## Topic 16: Monitoring (Application Insights, KQL)

**Study notes**

Enable via `APPLICATIONINSIGHTS_CONNECTION_STRING` (the instrumentation-key-only setting is being phased out - verify). Isolated model needs worker App Insights registration (Topic 2) for worker logs; host logs flow automatically.

Sampling (host.json):
```json
{ "version": "2.0",
  "logging": {
    "applicationInsights": { "samplingSettings": { "isEnabled": true, "maxTelemetryItemsPerSecond": 20, "excludedTypes": "Request;Exception" } },
    "logLevel": { "default": "Information", "Host.Results": "Information", "Function": "Information" }
  } }
```
Key tables: `requests` (invocations), `traces` (logs), `exceptions`, `dependencies`, `customMetrics`, `performanceCounters`. `customDimensions.InvocationId`, `Category` ("Function.<Name>"), `operation_Id`.

**Difficulty ladder**

### 16.1 [Easy] How do you connect a function app to Application Insights?
**Answer**
Set `APPLICATIONINSIGHTS_CONNECTION_STRING` (ideally workspace-based AI) and, for isolated, register `ConfigureFunctionsApplicationInsights` in Program.cs.
**Explanation**
Use the Live Metrics view for real-time. Avoid also enabling the built-in Azure Monitor agent-based auto-instrumentation twice (duplicate telemetry).
**Code / CLI / Bicep**
```bash
az functionapp config appsettings set -g rg -n func \
  --settings APPLICATIONINSIGHTS_CONNECTION_STRING="<conn-string>"
```

### 16.2 [Follow-up] Write KQL to find failed invocations and their exceptions.
**Answer**
Query `requests` where `success == false`, and join to `exceptions` by `operation_Id`.
**Explanation**
Group by `operation_Name` to see failing functions; use `customDimensions.InvocationId` to correlate with logs.
**Code / CLI / Bicep**
```kusto
requests
| where timestamp > ago(24h) and success == false
| project timestamp, operation_Id, name, resultCode, duration
| join kind=leftouter (exceptions | project operation_Id, type, outerMessage) on operation_Id
| order by timestamp desc
```

### 16.3 [Medium] How does sampling affect your investigation and how do you tune it?
**Answer**
Adaptive sampling drops a percentage of telemetry to cap volume, so counts in `requests` can be under-reported; use `itemCount` to scale. Exclude `Request` and `Exception` types or raise `maxTelemetryItemsPerSecond` for full fidelity where cost allows.
**Explanation**
Sampling is per-telemetry-type and correlated by operation. In isolated model, sampling may be configured both in host.json (host telemetry) and worker SDK options - verify behaviour in current docs.
**Code / CLI / Bicep**
```kusto
requests
| where timestamp > ago(1h)
| summarize executions = sum(itemCount) by bin(timestamp, 5m), operation_Name
```

### 16.4 [Hard] Analyse cold starts and scale-out behaviour with KQL.
**Answer**
Look at `cloud_RoleInstance` first-seen times and the duration of the first request per instance; slow first requests on a new instance indicate cold start. Also inspect host start traces ("Host started", "Initializing Host").
**Explanation**
Count distinct instances over time to see scale-out; correlate with latency percentiles. There is no dedicated cold-start flag, so infer.
**Code / CLI / Bicep**
```kusto
let firstSeen = requests | summarize first = min(timestamp) by cloud_RoleInstance;
requests
| join kind=inner firstSeen on cloud_RoleInstance
| where timestamp < first + 2m
| summarize p50=percentile(duration,50), p95=percentile(duration,95), n=count() by bin(timestamp,5m)
```
```kusto
traces
| where timestamp > ago(1d) and message has_any ("Host started", "Job host started")
| summarize starts=count() by bin(timestamp,1h), cloud_RoleInstance
```
**Gotchas / follow-ups**
- Excessive Information logs inflate cost; tune `logLevel`.
- Missing logs with default isolated filter (Topic 2).
- Use `Host.Results`/`Host.Aggregator` categories to see per-function stats.

### 16.5 [Tricky] Where are my logs? Invocation succeeded but nothing in App Insights.
**Answer**
Causes: missing connection string, ingestion delay (minutes), sampling, a log-level filter rule excluding it, worker not registered for App Insights, or daily cap hit. Check Live Metrics, the Log stream, and the app's host logs.
**Explanation**
Also check `APPINSIGHTS_INSTRUMENTATIONKEY` stale/wrong resource and network (private link scope/ AMPLS blocking ingestion).
**Code / CLI / Bicep**
```bash
az monitor app-insights component show -g rg -a ai-demo --query "{ingestion:connectionString}"
az webapp log tail -g rg -n func
```

**Interview traps**
- Reading `requests` count without `itemCount` under sampling.
- Duplicate telemetry from double instrumentation.
- Logging payload PII.
- Assuming logs are instant.

**Cheat sheet**
- Setting: `APPLICATIONINSIGHTS_CONNECTION_STRING`.
- Tables: requests, traces, exceptions, dependencies.
- `operation_Id` correlation; `itemCount` for sampling.
- host.json `logging` for level and sampling.

---

## Topic 17: Security

**Study notes**

| Layer | Controls |
|---|---|
| Caller auth | Function keys (weak), Easy Auth (Entra), JWT validation in code, API Management in front |
| Network inbound | Access restrictions (IP/service tag), private endpoints (Premium/Flex/Dedicated), APIM/App Gateway |
| Network outbound | VNet integration (regional), NSG/UDR/NAT Gateway, private endpoints to Storage/Service Bus/KV |
| Secrets | Managed identity first; Key Vault references; avoid connection strings |
| Browser | CORS (platform-level setting or ASP.NET Core CORS) |
| Storage account | Host storage with identity auth, firewall + private endpoints (needs `WEBSITE_CONTENTOVERVNET` etc. on some plans - verify) |

**Difficulty ladder**

### 17.1 [Easy] How do you restrict who can call an HTTP function?
**Answer**
Combine authentication (Easy Auth / Entra token validation) with network restrictions (access restrictions or private endpoint); function keys alone are not sufficient for user identity.
**Explanation**
Access restrictions can allow only an APIM IP/service tag. Private endpoint makes the app reachable only inside the VNet.
**Code / CLI / Bicep**
```bash
az functionapp config access-restriction add -g rg -n func --rule-name apim --action Allow \
  --ip-address 10.0.1.0/24 --priority 100
```

### 17.2 [Follow-up] VNet integration vs private endpoint - what is the difference?
**Answer**
VNet integration is for *outbound* calls from the function into a VNet; a private endpoint is for *inbound* private access to the function app. You often need both.
**Explanation**
VNet integration needs a delegated subnet (`Microsoft.Web/serverFarms` on Premium/Dedicated; `Microsoft.App/environments` for Flex - verify). Classic Consumption doesn't support either for inbound private endpoints/VNet integration.
**Code / CLI / Bicep**
```bash
az functionapp vnet-integration add -g rg -n func --vnet vnet-prod --subnet snet-func
az functionapp config set -g rg -n func --vnet-route-all-enabled true
```

### 17.3 [Medium] Configure CORS correctly.
**Answer**
Set allowed origins in the function app's CORS settings (platform) - no wildcard with credentials. With ASP.NET Core integration, platform CORS is typically sufficient; custom CORS middleware may conflict (verify behaviour).
**Explanation**
CORS is a browser control only; it does not protect against non-browser callers.
**Code / CLI / Bicep**
```bash
az functionapp cors add -g rg -n func --allowed-origins https://app.contoso.com
```

### 17.4 [Hard] Fully private function app talking to private Storage, Service Bus and Key Vault.
**Answer**
Use a plan with VNet support, enable VNet integration with route-all, create private endpoints (blob/queue/table/file for storage, Service Bus, Key Vault) with private DNS zones linked to the VNet, use managed identity everywhere, and disable public access on the resources.
**Explanation**
Common failures: missing private DNS zone link (name resolves to public IP), missing file share private endpoint when the plan needs `WEBSITE_CONTENTAZUREFILECONNECTIONSTRING` (not for Flex), NSG blocking, or storage firewall without identity roles. Service Bus private endpoints require Premium tier namespace.
**Code / CLI / Bicep**
```bicep
resource pe 'Microsoft.Network/privateEndpoints@2023-09-01' = {
  name: 'pe-sb'
  location: resourceGroup().location
  properties: {
    subnet: { id: peSubnetId }
    privateLinkServiceConnections: [{
      name: 'sb'
      properties: { privateLinkServiceId: sbNamespaceId, groupIds: ['namespace'] }
    }]
  }
}
```
**Gotchas / follow-ups**
- Triggers' scale controller also needs access to the source; verify the supported network configuration for each trigger (e.g. Service Bus Premium + private endpoints) in current docs.
- Deployment from a CI runner must reach the SCM endpoint (use self-hosted runner in VNet).

**Interview traps**
- Confusing VNet integration with private endpoints.
- Believing CORS secures APIs.
- Granting `Owner`/`Contributor` to the app identity.
- Leaving master key accessible.

**Cheat sheet**
- Inbound: private endpoint / access restrictions. Outbound: VNet integration.
- Identity-first, KV references for leftovers.
- Private DNS is the usual culprit.
- CORS = browser only.

---

## Topic 18: Performance, cost model and limits

**Study notes**

Performance: reuse clients (`IHttpClientFactory`/singleton SDK clients), async all the way (no `.Result`), avoid heavy startup work, stream large blobs, keep functions short, batch where possible. SNAT port exhaustion comes from creating new connections per call.

Cost (Consumption): executions + GB-s (memory x duration) with monthly free grant (verify current rates). Premium: per-second vCPU + memory with minimum always-ready instances billed continuously. Dedicated: plan VM cost. Flex: on-demand per execution/GB-s plus always-ready baseline cost. Hidden costs: storage transactions (Durable, timers, queues), App Insights ingestion, outbound data, VNet/private endpoints, NAT.

Limits (verify in current docs - they change): Consumption default timeout 5 min, max 10; Premium/Dedicated default 30 min, unbounded possible; Flex defaults vary; HTTP request ~230 s idle limit; HTTP request body ~100 MB (Consumption) - verify; Service Bus message size 256 KB (Standard) / up to 100 MB (Premium) - verify; Storage queue message 64 KB.

**Difficulty ladder**

### 18.1 [Easy] Why use a static HttpClient or IHttpClientFactory?
**Answer**
Creating an `HttpClient` per call leaks sockets and exhausts SNAT ports under load; reusing clients pools connections and respects DNS changes (factory).
**Explanation**
In Functions the problem is amplified by scale-out and shared outbound IPs (SNAT limited per instance).
**Code / CLI / Bicep**
```csharp
builder.Services.AddHttpClient<IPartnerClient, PartnerClient>(c => c.BaseAddress = new Uri("https://partner.example"));
public class PartnerClient(HttpClient http) : IPartnerClient
{
    public Task<string> GetAsync(string p, CancellationToken ct) => http.GetStringAsync(p, ct);
}
```

### 18.2 [Follow-up] How is Consumption cost calculated and how do you estimate?
**Answer**
Billed on number of executions plus resource consumption in GB-seconds (average memory x execution time, rounded). Estimate: executions/month x avg duration x memory, minus free grant.
**Explanation**
Reduce cost by shortening execution, batching messages (fewer executions), avoiding waiting on slow I/O with sync code, and reducing log volume. Premium is cheaper at sustained high load; Consumption/Flex cheaper for sporadic.
**Code / CLI / Bicep**
```text
Example: 3M exec x 0.5 s x 0.25 GB = 375,000 GB-s ; compare with free grant & current price (verify).
```

### 18.3 [Medium] A function waits on a slow API for 8 minutes. What are the options?
**Answer**
Don't block: switch to a queue + async completion (callback/polling), Durable Functions with timers/`CallHttp`, a longer-timeout plan, or a Container Apps job. Raise `functionTimeout` only as a last resort and within plan limits.
**Explanation**
Sync HTTP responses can't exceed ~230 s anyway. Long CPU-bound work wastes GB-s on Consumption.
**Code / CLI / Bicep**
```json
{ "version": "2.0", "functionTimeout": "00:30:00" }
```

### 18.4 [Hard] Throughput tuning of a queue-triggered function that calls SQL.
**Answer**
Measure first: instance count, per-instance concurrency, SQL DTU/connection limits. Use batching (batch trigger), `async` SQL calls, connection pooling, and cap total concurrency to what SQL sustains; use bulk operations.
**Explanation**
Total concurrent connections = instances x concurrency; Azure SQL has a worker/connection limit. Excessive scale-out causes timeouts and retries (retry storm). Add backoff and circuit breakers.
**Code / CLI / Bicep**
```kusto
dependencies
| where type == "SQL" and timestamp > ago(1h)
| summarize avg(duration), p95=percentile(duration,95), failed=countif(success==false) by bin(timestamp,5m)
```
**Gotchas / follow-ups**
- Thread pool starvation from sync-over-async shows as slow, low-CPU invocations.
- Large `byte[]` blob bindings cause memory pressure - use Stream.
- Cold start impact of heavy static init and large dependency trees; consider ReadyToRun (verify with isolated).

**Interview traps**
- `new HttpClient()` in the function body.
- Raising timeout to hide slow code.
- Ignoring storage transaction & App Insights costs.
- Quoting limits from memory.

**Cheat sheet**
- Reuse clients; async only.
- Cost = executions + GB-s (+ always-ready/baseline).
- ~230 s HTTP limit; function timeout per plan.
- Concurrency x instances = downstream load.

---

## Topic 19: Production scenarios (troubleshooting playbook)

**Study notes**

General triage order: (1) Is the host running/healthy? (2) Are functions indexed/loaded and enabled? (3) Connectivity/identity to the trigger source and `AzureWebJobsStorage`? (4) Is the event actually there? (5) Logs/exceptions. Tools: Portal Diagnose and solve problems, Live Metrics, Log stream, Kudu, KQL, `az functionapp function list`.

**Difficulty ladder**

### 19.1 [Medium] Function is not triggering. What do you check?
**Answer**
Check the function appears in the portal (indexing succeeded), is not disabled (`AzureWebJobs.<name>.Disabled`), the app is running and not stopped, `AzureWebJobsStorage` is reachable, trigger connection settings/identity roles are valid, and the source actually has messages. Then host startup logs for listener errors.
**Explanation**
Typical: wrong app setting name, missing RBAC on identity connection, firewall on storage/Service Bus, scale controller can't read the trigger metric (Consumption/Flex), wrong slot, runtime version mismatch (`FUNCTIONS_WORKER_RUNTIME`, `FUNCTIONS_EXTENSION_VERSION`).
**Code / CLI / Bicep**
```bash
az functionapp function list -g rg -n func -o table
az functionapp config appsettings list -g rg -n func --query "[?contains(name,'Disabled')]"
```
```kusto
traces | where timestamp > ago(1h) and (message has "Listener" or message has "error" or severityLevel >= 3)
```

### 19.2 [Hard] Duplicate executions.
**Answer**
Expected under at-least-once delivery: lock lost/expired, retries, scale-in mid-processing, two slots/apps consuming the same source, timer double-run, Event Grid redelivery, blob overwrites. Fix by making handlers idempotent and removing accidental duplicate consumers.
**Explanation**
Diagnose via `InvocationId` and message/delivery count in logs; check Service Bus `DeliveryCount`, lock duration, `maxAutoLockRenewalDuration`; check whether a staging slot or local dev machine is consuming.
**Code / CLI / Bicep**
```kusto
traces
| where customDimensions.Category startswith "Function."
| extend inv = tostring(customDimensions.InvocationId), mid = tostring(customDimensions.prop__MessageId)
| summarize runs = dcount(inv) by mid
| where runs > 1
```
(`prop__` custom dimension names depend on your logging template.)
**Gotchas / follow-ups**
- Duplicate detection on Service Bus only prevents duplicate *sends* within the window.

### 19.3 [Hard] Poison loops and DLQ growth.
**Answer**
A bad message fails repeatedly until MaxDeliveryCount, then lands in the DLQ; if the failure is systemic (downstream outage) every message poisons. Fix root cause, add circuit breaking/backoff, alert on DLQ depth, and build a replay tool to resubmit from DLQ.
**Explanation**
Retry storms amplify with scale-out. Prefer finite retries and dead-letter with reason for permanent failures; for downstream outages, abandon with delay (schedule a new message) rather than hot-loop.
**Code / CLI / Bicep**
```bash
az monitor metrics list --resource <sbNamespaceId> --metric DeadletteredMessages --interval PT5M --filter "EntityName eq 'orders'"
```

### 19.4 [Hard] Storage account problems (host fails, triggers stop).
**Answer**
Everything depends on `AzureWebJobsStorage`: deleted/rotated key, firewall/private-endpoint DNS issues, missing identity roles, quota/throttling, or sharing one account across many heavy apps. Symptoms: host unhealthy, "Unable to connect to storage", timer/singleton errors, Durable stalls.
**Explanation**
Use identity-based host storage with proper roles, a dedicated account per app (or per few), avoid blob-only/ premium-incompatible accounts, and check storage metrics for throttling (HTTP 503/ServerBusy).
**Code / CLI / Bicep**
```bash
az storage account show -n stdemo --query "{pna:publicNetworkAccess, fw:networkRuleSet.defaultAction}"
az functionapp config appsettings list -g rg -n func --query "[?starts_with(name,'AzureWebJobsStorage')]"
```

### 19.5 [Medium] Users complain about latency on first request.
**Answer**
Confirm it's cold start using KQL (first request per instance), then reduce startup work, use run-from-package, ReadyToRun, trim dependencies, and move to Flex always-ready or Premium min instances if it must be eliminated.
**Explanation**
Don't use timer warmers as a fix; check also for scale-out cold starts during spikes and downstream warm-up (DB connection, token acquisition). Cache credentials/clients as singletons.
**Code / CLI / Bicep**
```bicep
// Flex: always-ready instances (verify property names in current docs)
properties: { functionAppConfig: { scaleAndConcurrency: { alwaysReady: [ { name: 'http', instanceCount: 1 } ] } } }
```

### 19.6 [Tricky] Timer double-run, host.json not applied, and hitting downstream limits when scaled out - quick diagnostics?
**Answer**
Timer double-run: look for another app/slot/local machine on the same storage and disable staging triggers (Topic 9). host.json not applied: confirm the deployed package contains the new file, restart, or override via `AzureFunctionsJobHost__` app settings (Topic 6). Downstream limits: cap instances and concurrency, use backoff and queue buffering (Topic 6).
**Explanation**
All three share a theme: the platform runs more instances/consumers than you mentally model. Always ask "how many instances, slots and environments are running this code right now?"
**Code / CLI / Bicep**
```kusto
requests
| where timestamp > ago(1h)
| summarize instances = dcount(cloud_RoleInstance), execs = sum(itemCount) by bin(timestamp, 1m)
```
**Gotchas / follow-ups**
- Check `cloud_RoleName` to distinguish slots/apps writing to the same App Insights.

**Interview traps**
- Jumping to code before checking indexing, disabled flags, storage and RBAC.
- Blaming the platform for duplicates that at-least-once semantics predict.
- No alerting on DLQ/poison queues.
- Fixing symptoms with higher timeouts/retries.

**Cheat sheet**
- Triage: host healthy -> indexed -> enabled -> storage -> source auth -> event present -> logs.
- Duplicates: idempotency + find extra consumers.
- Poison: finite retry, DLQ alerts, replay tool.
- Storage is a hard dependency - protect it.
- Count instances/slots/environments first.

