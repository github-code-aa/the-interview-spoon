# Z06 - Azure Logic Apps: Study + Interview Guide

**How to use:** Read each topic's *Study notes* first, then work the difficulty ladder (Easy -> Tricky), answering aloud before reading the **Answer**.
Limits, pricing and feature availability change often - anything marked *verify in current docs* must be checked before quoting numbers in an interview.

## Table of Contents
1. [What Logic Apps is and when to use it](#topic-1-what-logic-apps-is-and-when-to-use-it)
2. [Consumption vs Standard](#topic-2-consumption-vs-standard)
3. [Workflow definition language and control flow](#topic-3-workflow-definition-language-and-control-flow)
4. [Triggers, concurrency, debatching, conditions](#topic-4-triggers-concurrency-debatching-conditions)
5. [Expressions and functions](#topic-5-expressions-and-functions)
6. [Connectors](#topic-6-connectors)
7. [Authentication and managed identity](#topic-7-authentication-and-managed-identity)
8. [Error handling, retries, idempotency](#topic-8-error-handling-retries-idempotency)
9. [Calling Functions/APIs, async patterns, limits](#topic-9-calling-functionsapis-async-patterns-limits)
10. [Service Bus integration patterns](#topic-10-service-bus-integration-patterns)
11. [Blob and Event Grid patterns](#topic-11-blob-and-event-grid-patterns)
12. [B2B/EDI, integration account, transforms](#topic-12-b2bedi-integration-account-transforms)
13. [JSON, arrays, large data, chunking](#topic-13-json-arrays-large-data-chunking)
14. [Monitoring, resubmit, rerun](#topic-14-monitoring-resubmit-rerun)
15. [CI/CD, IaC, testing](#topic-15-cicd-iac-testing)
16. [Security, cost control, governance](#topic-16-security-cost-control-governance)
17. [Scenario comparison: which tool?](#topic-17-scenario-comparison-which-tool)
18. [Production scenarios / troubleshooting](#topic-18-production-scenarios--troubleshooting)

---

## Topic 1: What Logic Apps is and when to use it

**Study notes**

Azure Logic Apps is a managed workflow/orchestration service: a **trigger** starts a run, then **actions** execute in a directed graph. Workflows are declarative JSON (Workflow Definition Language, WDL) designed visually or in code. Strength: hundreds of connectors, built-in retry/run history, no infrastructure code.

```
Tool            Model                      Sweet spot
--------------  -------------------------  ------------------------------------------
Logic Apps      Declarative workflow       SaaS/Azure integration, B2B, glue, approvals
Functions       Code, event-triggered      Custom logic, transforms, compute, short tasks
Durable Funcs   Code-first orchestration   Complex stateful code workflows, fan-out/in, human wait
Power Automate  Low-code, maker-focused    Business-user flows in M365/Dataverse (built on Logic Apps engine)
Data Factory    Data pipelines (ETL/ELT)   Bulk data movement/transformation, schedules, big copy
```

### 1.1 [Easy] What is Azure Logic Apps?
**Answer**: A managed service for building automated workflows from a trigger plus a chain of actions, using connectors to integrate services without writing infrastructure or much code. It is typically used for application/SaaS/B2B integration and orchestration.
**Explanation**: The designer produces a JSON definition. The platform handles scaling, retries, state persistence and run history. You pay per execution (Consumption) or for reserved compute (Standard).
**Code / JSON / Bicep**
```json
{
  "definition": {
    "$schema": "https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#",
    "contentVersion": "1.0.0.0",
    "triggers": { "manual": { "type": "Request", "kind": "Http" } },
    "actions": { "Response": { "type": "Response", "kind": "Http", "inputs": { "statusCode": 200 }, "runAfter": {} } }
  }
}
```

### 1.2 [Follow-up] Logic Apps vs Azure Functions - how do you choose?
**Answer**: Choose Logic Apps when the job is mostly connecting systems with existing connectors, needs visual run history and built-in retry. Choose Functions when you need custom code, tight performance control, complex transforms or unit-testable logic. They are complementary: Logic Apps orchestrates, Functions does the heavy logic.
**Explanation**: Functions give full language power, cheap per-execution cost at high volume and easy debugging/testing. Logic Apps charges per action (Consumption), so a loop over 100k items can be expensive; the same loop in code is nearly free.
**Code / JSON / Bicep**
```text
Rule of thumb:
  >10 actions of pure connector glue  -> Logic Apps
  heavy parsing / algorithms / high TPS -> Function (called from Logic App if needed)
```

### 1.3 [Medium] Logic Apps vs Durable Functions vs Power Automate vs Data Factory?
**Answer**: Durable Functions is code-first stateful orchestration (fan-out/fan-in, eternal workflows, custom retry) for developers. Power Automate is the maker/citizen-developer product on the same engine, tied to M365/Dataverse licensing and environments. Data Factory is for data movement/ETL at scale with copy activities and mapping data flows, not message-level integration.
**Explanation**: If a business user owns the flow and data lives in M365 -> Power Automate. If IT owns enterprise integration with ALM/IaC -> Logic Apps. If the orchestration needs unit tests and complex branching/loops in C# -> Durable Functions. If moving GBs of tables/files on schedule -> ADF/Fabric pipelines.
**Code / JSON / Bicep**
```text
Nightly copy of 50 GB SQL -> ADLS            : Data Factory
Service Bus msg -> enrich -> call SAP/Salesforce: Logic Apps
Saga with compensation in C#, unit-tested     : Durable Functions
Approve expense in Teams by manager           : Power Automate (or Logic Apps approvals)
```

### 1.4 [Hard] When is Logic Apps the wrong choice?
**Answer**: When throughput is very high with small messages (action-based cost and latency), logic is algorithmic/complex, you need sub-second latency, or the team needs strong automated testing. Also wrong for bulk ETL.
**Explanation**: Each action has overhead (tens of ms+) and, in Consumption, a cost. Deep expression logic becomes unreadable JSON. Debugging large foreach loops is painful. Limits on run duration, message size and concurrency apply (verify in current docs).
**Code / JSON / Bicep**
```text
Anti-pattern: foreach over 500k blob lines with 5 actions each = 2.5M action executions.
Better: Function/ADF processes the file; Logic App only starts it and handles notification.
```
**Gotchas / follow-ups**
- Interviewers like "can Logic Apps call Functions?" - yes; and Standard can host in-process custom code (verify current support).
- "Is Power Automate Logic Apps?" - same underlying engine/connectors, different product surface and licensing.

### 1.5 [Tricky] Is a Logic App a workflow engine with guaranteed exactly-once execution?
**Answer**: No. Triggers and actions are generally at-least-once; retries, redelivery and polling overlap can produce duplicates, so downstream operations must be idempotent.
**Explanation**: Retry policies re-send requests; a Service Bus lock expiry can redeliver; a polling trigger may fire twice if state was not advanced. Design with natural idempotency keys or dedupe stores.
**Code / JSON / Bicep**
```text
Idempotency key candidates: Service Bus MessageId, business order id, blob ETag, x-ms-client-tracking-id.
```
**Gotchas / follow-ups**
- Duplicate detection on Service Bus helps only for send side within window.

**Interview traps**
- Saying Logic Apps "replaces" Functions.
- Forgetting action-based billing.
- Claiming exactly-once delivery.
- Confusing Power Automate and Logic Apps licensing.

**Cheat sheet**
- Logic Apps = connectors + workflow + run history.
- Functions = code; Durable = code orchestration.
- ADF = data movement; Power Automate = maker flows.
- At-least-once; make downstream idempotent.
- Cost: per action (Consumption) vs plan (Standard).

---

## Topic 2: Consumption vs Standard

**Study notes**

```
                    Consumption                    Standard
------------------- ------------------------------ -----------------------------------
Resource            1 workflow = 1 Logic App       1 Logic App = many workflows
Hosting             Multi-tenant (shared)          Single-tenant (Functions v4 runtime), WS/ASE plans
Billing             Per trigger/action execution   Hosting plan (vCPU/memory) - built-in actions included
Workflow types      Stateful only                  Stateful + Stateless
VNet                Via ISE (retired) / limited    VNet integration + private endpoints
Local dev           Limited                        VS Code extension, run/debug locally
Connectors          Managed (shared) + built-in    Built-in service providers (in-app) + managed
Deployment          ARM/Bicep resource             Zip deploy / app-style (like Functions)
```
ISE (Integration Service Environment) is retired/retiring - verify in current docs; Standard replaces it.

### 2.1 [Easy] What is the difference between Consumption and Standard?
**Answer**: Consumption is multi-tenant pay-per-execution with one workflow per resource. Standard is single-tenant, runs on the Functions runtime, hosts multiple workflows per app, and bills by compute plan.
**Explanation**: Standard offers VNet integration, private endpoints, local development and predictable cost at high volume. Consumption suits low-volume, spiky, simple needs.
**Code / JSON / Bicep**
```bicep
resource la 'Microsoft.Logic/workflows@2019-05-01' = {   // Consumption
  name: 'la-orders'
  location: resourceGroup().location
  properties: { definition: {} }
}
resource std 'Microsoft.Web/sites@2023-01-01' = {         // Standard
  name: 'la-std-orders'
  location: resourceGroup().location
  kind: 'functionapp,workflowapp'
  properties: { serverFarmId: plan.id }
}
```

### 2.2 [Follow-up] Stateful vs stateless workflows in Standard?
**Answer**: Stateful persists inputs/outputs of every action to storage, giving run history, resubmit and long-running support. Stateless keeps state in memory only: faster and cheaper, but no durable history, short runs, and limited trigger/action support.
**Explanation**: Stateless is good for request/response low-latency orchestration under a short duration (verify current limit, historically ~5 minutes). Stateless run history can be enabled for debugging only. Stateful uses the storage account (queues/tables/blobs) - storage throttling can affect performance.
**Code / JSON / Bicep**
```json
{ "definition": { "...": "..." }, "kind": "Stateless" }
```

### 2.3 [Medium] How does pricing differ and what does it mean for design?
**Answer**: Consumption charges per action/trigger execution (built-in vs standard vs enterprise connector rates differ), so chatty loops cost more. Standard charges for the plan (WS1-3 etc.) regardless of action count, so high-volume workloads are cheaper and predictable, but idle cost exists.
**Explanation**: Rates and tiers change - verify in current docs. Break-even depends on volume; model with the pricing calculator. In Standard, built-in connectors run in-process (no per-action charge) while managed connectors may still be billed per call.
**Code / JSON / Bicep**
```text
Estimate: monthly actions = runs/day * actions/run * 30 -> compare with plan cost.
```

### 2.4 [Medium] How do you develop locally for Standard?
**Answer**: Use the Azure Logic Apps (Standard) VS Code extension with Azurite and the Functions Core Tools; workflows live as `workflow.json` files in folders, with `connections.json`, `parameters.json`, `local.settings.json`.
**Explanation**: Project structure mirrors a function app. Source control and zip deploy work naturally. Managed connectors need Azure-side API connections even locally.
**Code / JSON / Bicep**
```text
MyLogicApp/
  host.json
  local.settings.json
  connections.json
  parameters.json
  OrderWorkflow/workflow.json
```

### 2.5 [Hard] Performance and scaling differences?
**Answer**: Consumption auto-scales in the shared tier but is subject to throttling limits per workflow/connection. Standard scales by plan instances (elastic scale-out for WS plans, with always-ready/pre-warmed instances) and is typically lower latency for built-in actions.
**Explanation**: Standard stateful performance is bounded by the storage account and its throttling; use a dedicated storage account with adequate tier. Scale rules rely on trigger-driven scaling for built-in triggers (e.g., Service Bus). Verify scale-out limits in current docs.
**Code / JSON / Bicep**
```json
{ "extensions": { "workflow": { "settings": { "Runtime.Trigger.MaximumWaitingRuns": "100" } } } }
```
**Gotchas / follow-ups**
- Sharing one storage account across many Standard apps causes cross-app throttling.
- Cold start on Consumption plans for Standard (WS) is mitigated by pre-warmed instances.

**Interview traps**
- "Standard is always cheaper" - not for tiny volume.
- Stateless runs have no resubmit.
- Assuming Consumption supports VNet injection natively.

**Cheat sheet**
- Consumption: multi-tenant, per action, 1 workflow/resource.
- Standard: single-tenant, plan billing, many workflows, VNet, VS Code.
- Stateful = durable history; stateless = fast, in-memory.
- Standard = Functions runtime + extension bundle.

---

## Topic 3: Workflow definition language and control flow

**Study notes**

A definition has `parameters`, `triggers`, `actions`, `outputs`. Each action has `type`, `inputs`, `runAfter`. Order is defined only by `runAfter` (a DAG) - actions with the same `runAfter` run in parallel.

Control flow: `If` (condition), `Switch`, `Foreach`, `Until`, `Scope`.

```
Trigger -> A -> B ----\
              \-> C ---+-> D (runAfter B and C)
```

### 3.1 [Easy] What is runAfter?
**Answer**: A property that defines which previous action(s) must finish, and with which statuses, before this action runs. Default is `Succeeded`.
**Explanation**: Statuses: `Succeeded`, `Failed`, `Skipped`, `TimedOut`. Multiple predecessors act as AND; all must reach one of the listed statuses.
**Code / JSON / Bicep**
```json
"Notify": {
  "type": "Http",
  "inputs": { "method": "POST", "uri": "https://example/notify" },
  "runAfter": { "Process": [ "Failed", "TimedOut" ] }
}
```

### 3.2 [Follow-up] Condition vs Switch?
**Answer**: Condition is an if/else on a boolean expression; Switch matches one value against multiple cases plus default. Use Switch for >2 discrete values.
**Explanation**: Both are containers; actions in the untaken branch are `Skipped`. Switch cases are exact-match (strings case-insensitive comparison semantics - verify), no fall-through.
**Code / JSON / Bicep**
```json
"Route": {
  "type": "Switch",
  "expression": "@triggerBody()?['type']",
  "cases": {
    "Order": { "case": "order", "actions": {} },
    "Refund": { "case": "refund", "actions": {} }
  },
  "default": { "actions": {} },
  "runAfter": {}
}
```

### 3.3 [Medium] How does Foreach work, and what about concurrency?
**Answer**: Foreach iterates an array; by default it runs iterations in parallel (default degree 20, max 50). Set `runtimeConfiguration.concurrency.repetitions` to 1 for sequential execution.
**Explanation**: Variables modified inside a parallel foreach are not safe (race conditions); the designer warns/limits this, so set concurrency to 1 or avoid shared variables (verify current behavior). Arrays have an item limit (verify in current docs; ~100,000 for Consumption/Standard).
**Code / JSON / Bicep**
```json
"For_each_order": {
  "type": "Foreach",
  "foreach": "@body('Parse_orders')?['items']",
  "runtimeConfiguration": { "concurrency": { "repetitions": 1 } },
  "actions": { "Handle": { "type": "Compose", "inputs": "@items('For_each_order')" , "runAfter": {} } },
  "runAfter": {}
}
```

### 3.4 [Medium] Until loop and its limits?
**Answer**: Until repeats actions until a condition is true, bounded by `limit.count` (default 60) and `limit.timeout` (default PT1H). Both must be set sensibly.
**Explanation**: It checks the condition after each iteration. Used for polling an async job. Wait with a Delay action inside to avoid hammering.
**Code / JSON / Bicep**
```json
"Until_done": {
  "type": "Until",
  "expression": "@equals(body('Check')?['status'], 'Done')",
  "limit": { "count": 30, "timeout": "PT30M" },
  "actions": {
    "Wait": { "type": "Wait", "inputs": { "interval": { "count": 30, "unit": "Second" } }, "runAfter": {} },
    "Check": { "type": "Http", "inputs": { "method": "GET", "uri": "https://api/job/1" }, "runAfter": { "Wait": ["Succeeded"] } }
  },
  "runAfter": {}
}
```

### 3.5 [Hard] SplitOn vs Foreach for arrays from a trigger?
**Answer**: `splitOn` debatches the trigger array into one independent run per item; Foreach processes all items within one run. SplitOn gives per-item run history, parallel scale and isolation of failures.
**Explanation**: With Foreach, one failing item can fail the whole run and shared run limits (duration, loop size) apply. With SplitOn, each run is separate, trigger concurrency controls parallelism. Support varies by trigger type (verify in current docs).
**Code / JSON / Bicep**
```json
"triggers": {
  "manual": {
    "type": "Request", "kind": "Http",
    "splitOn": "@triggerBody()?['items']",
    "inputs": { "schema": {} }
  }
}
```
**Gotchas / follow-ups**
- With SplitOn on a Request trigger, a Response action replies per item run, so the caller contract needs care (verify behavior).
- Each split run gets its own run ID; correlate with a tracking id from the batch.

### 3.6 [Tricky] An action after a Scope/Condition shows Skipped - why?
**Answer**: Because its `runAfter` required Succeeded of a predecessor that was skipped or failed. Skip propagates down the chain.
**Explanation**: Actions in a non-taken branch are `Skipped`, and anything depending on them with default runAfter also skips. Fix by adding `Skipped` to runAfter or joining after the Condition container itself.
**Code / JSON / Bicep**
```json
"Join": { "type": "Compose", "inputs": "done", "runAfter": { "Condition": ["Succeeded","Failed","Skipped"] } }
```
**Gotchas / follow-ups**
- A scope's status is Failed if any inner action failed, even if handled inside.
- Terminate action status inside parallel branches ends the whole run.

**Interview traps**
- Believing actions run top-to-bottom in order.
- Setting a variable in a parallel Foreach.
- Forgetting Until count/timeout limits.
- Confusing SplitOn with Foreach.

**Cheat sheet**
- runAfter = DAG with statuses.
- Foreach default parallel (20), max 50.
- Until: count 60 / PT1H defaults.
- SplitOn = one run per item.
- Skipped cascades.

## Topic 4: Triggers, concurrency, debatching, conditions

**Study notes**

```
Type        How it starts                         Examples
----------  ------------------------------------  --------------------------------------
Polling     Platform checks on an interval        Recurrence, many managed connector triggers (Blob (managed), Outlook)
Push/Hook   Source calls Logic App (callback URL) Request (HTTP), Event Grid, HTTP Webhook
Built-in    Runs in the engine (Standard: in-app) Service Bus (built-in), Event Hubs, Request, Recurrence
```
A polling trigger that finds nothing counts as a trigger execution (billed in Consumption, shown as "Skipped" in trigger history).

### 4.1 [Easy] Polling vs push triggers?
**Answer**: Polling triggers check the source at a set frequency and fire when new data exists; push (webhook) triggers are invoked by the source immediately. Push has lower latency and no empty polls.
**Explanation**: Polling cost and latency depend on the interval. Request and Event Grid triggers are push. Service Bus triggers are listener/poll-based under the hood with near-real-time delivery.
**Code / JSON / Bicep**
```json
"triggers": {
  "Recurrence": { "type": "Recurrence", "recurrence": { "frequency": "Minute", "interval": 5 } }
}
```

### 4.2 [Follow-up] What does a Request (HTTP) trigger give you and how is it secured?
**Answer**: A callback URL with a SAS-like signature (`sig`) that accepts HTTP calls, optionally with a JSON schema. Combine with IP restrictions, Entra OAuth policies, and APIM in front for real security.
**Explanation**: The URL contains the secret, so treat it like a password; regenerate keys to rotate. Standard supports Easy Auth-style options (verify). With a Response action you make it synchronous.
**Code / JSON / Bicep**
```json
"manual": {
  "type": "Request", "kind": "Http",
  "inputs": {
    "method": "POST",
    "schema": { "type": "object", "properties": { "orderId": { "type": "string" } }, "required": ["orderId"] }
  }
}
```

### 4.3 [Medium] How do you control trigger concurrency and avoid overload?
**Answer**: Set `runtimeConfiguration.concurrency.runs` to cap parallel runs, and `maximumWaitingRuns` to limit queued runs. Without it, runs may be unbounded up to platform limits.
**Explanation**: Enabling concurrency control is irreversible on a workflow in the designer (cannot be undone without recreating - verify). Excess runs queue in "Waiting" state; if the waiting queue is full, new triggers are rejected/ throttled (HTTP 429 for Request).
**Code / JSON / Bicep**
```json
"runtimeConfiguration": {
  "concurrency": { "runs": 5, "maximumWaitingRuns": 20 }
}
```

### 4.4 [Medium] What are trigger conditions?
**Answer**: Boolean expressions evaluated before a run is created; if false the trigger fires no run (shown as Skipped), saving action cost.
**Explanation**: Use them to filter at the front door (e.g., only messages with a property). Trigger conditions are stateless checks on trigger output.
**Code / JSON / Bicep**
```json
"conditions": [
  { "expression": "@equals(triggerBody()?['status'], 'Approved')" }
]
```

### 4.5 [Hard] Debatching: SplitOn versus Service Bus batch triggers?
**Answer**: SplitOn creates one run per array element from a trigger that returns arrays (e.g., polling connector with multiple items). Service Bus built-in trigger can be configured for maxMessageCount/batch, returning multiple messages per run to process together.
**Explanation**: Choose per-item runs for isolation and independent retries; choose batch runs for efficiency and ordered group processing. With Standard built-in Service Bus trigger, concurrency and prefetch settings live in `host.json` / trigger settings (verify exact names in docs).
**Code / JSON / Bicep**
```json
"splitOn": "@triggerBody()?['value']"
```
**Gotchas / follow-ups**
- Large SplitOn arrays have an item limit (verify).
- Per-item runs multiply trigger-based billing in Consumption.

### 4.6 [Tricky] Why does a polling trigger fire twice or miss items?
**Answer**: Polling triggers track state ("last seen" timestamp/cursor); changing the trigger, disabling/enabling or clock skew/same-timestamp items can reset or skip state. Late-arriving data with older timestamps than the cursor is missed.
**Explanation**: E.g., "When a blob is added or modified (managed)" uses last-modified; bulk uploads of identical timestamps can be missed or reprocessed. Prefer Event Grid for blobs. Disable/enable generally preserves state, but saving a changed trigger resets it.
**Code / JSON / Bicep**
```text
Mitigation: push via Event Grid; or idempotent processing + move/rename processed blobs.
```
**Gotchas / follow-ups**
- Recurrence with start time/time zone and "schedule" drift after DST - set time zone explicitly.

**Interview traps**
- Thinking polling is free.
- Request trigger URL is not public-safe without extra controls.
- Concurrency control can't be toggled back.

**Cheat sheet**
- Poll = interval, push = callback.
- `runtimeConfiguration.concurrency.runs`.
- `conditions` saves runs.
- SplitOn = per-item runs.
- Prefer Event Grid over blob polling.

---

## Topic 5: Expressions and functions

**Study notes**

Expressions start with `@` (or `@{...}` string interpolation). Key functions: `triggerBody()`, `triggerOutputs()`, `body('Action')`, `outputs('Action')`, `items('Loop')`, `item()`, `variables('v')`, `parameters('p')`, `workflow()`, `coalesce`, `if`, `equals`, `empty`, `length`, `contains`, `union`, `concat`, `formatDateTime`, `utcNow`, `addDays`, `json`, `xml`, `xpath`, `string`, `int`, `base64`, `uriComponent`, `guid`, `createArray`, `first`, `last`, `take`, `skip`. The `?` operator is null-safe property access.

### 5.1 [Easy] How do you read the trigger body and an earlier action's output?
**Answer**: `@triggerBody()` for the trigger payload and `@body('ActionName')` for an action's body. Use `?['prop']` to avoid null-reference failures.
**Explanation**: Action names with spaces become underscores in the code view (e.g., `Get_blob_content`). `outputs('x')` includes statusCode and headers; `body('x')` is just the body.
**Code / JSON / Bicep**
```text
@triggerBody()?['orderId']
@body('Get_customer')?['address']?['city']
@outputs('Call_API')['statusCode']
```

### 5.2 [Follow-up] Variables vs Compose?
**Answer**: Compose holds an immutable value computed once; variables are mutable, must be declared at top level via Initialize variable, and are scoped to the run. Prefer Compose unless you need mutation (counters, accumulators).
**Explanation**: Variables are not thread-safe in parallel loops; Compose has no such problem and is cheaper to reason about. Initialize variable cannot be inside a loop/condition in Consumption designer (top-level only).
**Code / JSON / Bicep**
```json
"Init_total": { "type": "InitializeVariable", "inputs": { "variables": [ { "name": "total", "type": "integer", "value": 0 } ] }, "runAfter": {} },
"Add": { "type": "IncrementVariable", "inputs": { "name": "total", "value": 1 }, "runAfter": { "Init_total": ["Succeeded"] } }
```

### 5.3 [Medium] Common date/time and null handling expressions?
**Answer**: `formatDateTime(utcNow(),'yyyy-MM-dd')`, `addDays`, `convertTimeZone`, and `coalesce(a,b)` for defaults; use `if(empty(x), 'n/a', x)` for blanks.
**Explanation**: Format strings are .NET-style (`yyyy`, `MM`, `HH`, `mm`). `utcNow()` returns an ISO string, not a date object. `coalesce` returns first non-null (empty string is non-null!).
**Code / JSON / Bicep**
```text
@formatDateTime(addDays(utcNow(), -1), 'yyyy-MM-ddTHH:mm:ssZ')
@coalesce(triggerBody()?['region'], 'WEU')
@convertTimeZone(utcNow(), 'UTC', 'India Standard Time', 'yyyy-MM-dd HH:mm')
```

### 5.4 [Medium] json(), xml() and xpath()?
**Answer**: `json()` converts a string/XML to a JSON object; `xml()` converts JSON/string to XML; `xpath()` queries XML and returns an array (use `first()` or `string()`).
**Explanation**: `xpath` returns arrays even for a single node; wrap with `first(xpath(...))`. For namespaces, use `local-name()` predicates.
**Code / JSON / Bicep**
```text
@json(body('Get_blob_content'))
@xpath(xml(body('Get_xml')), 'string(/*[local-name()="Order"]/*[local-name()="Id"])')
@first(xpath(xml(triggerBody()), '/Order/Id/text()'))
```

### 5.5 [Hard] What does workflow() give and how is it used for correlation?
**Answer**: `workflow()` returns metadata: `workflow().name`, `workflow().run.name` (run ID), `workflow().location`, plus tags. Use `run.name` as a correlation ID in logs and downstream headers.
**Explanation**: Combine with Service Bus MessageId and App Insights operation IDs. `trigger().clientTrackingId` and `x-ms-client-tracking-id` header can carry upstream correlation.
**Code / JSON / Bicep**
```json
"Call_API": {
  "type": "Http",
  "inputs": {
    "method": "POST",
    "uri": "https://api.contoso/orders",
    "headers": { "x-correlation-id": "@{workflow().run.name}" },
    "body": "@triggerBody()"
  },
  "runAfter": {}
}
```
**Gotchas / follow-ups**
- `@` in literal strings must be escaped as `@@`.

### 5.6 [Tricky] Why does my expression throw "property doesn't exist" or compare wrongly?
**Answer**: Missing `?` null-safe access, string vs number comparison, case/format issues, or dynamic content from an action that was skipped (null body).
**Explanation**: `equals('1', 1)` is false (types differ). `@body('x')['prop']` throws when absent; `?['prop']` yields null. Comparing in a Condition with `greater` on strings behaves lexicographically. Boolean from JSON may arrive as string.
**Code / JSON / Bicep**
```text
@equals(int(triggerBody()?['qty']), 5)
@if(equals(triggerBody()?['flag'], true), 'yes', 'no')
```
**Gotchas / follow-ups**
- `length(null)` fails; guard with `coalesce(x, createArray())`.
- Expressions are evaluated at runtime; designer doesn't fully type-check.

**Interview traps**
- Forgetting `?` for optional properties.
- `xpath` returns array.
- `@` escaping.
- Using variables where Compose suffices.

**Cheat sheet**
- `triggerBody()`, `body()`, `outputs()`, `items()`.
- `coalesce`, `if`, `empty`, `contains`.
- `workflow().run.name` for correlation.
- `?[]` null-safe.
- Prefer Compose.

---

## Topic 6: Connectors

**Study notes**

```
Kind          Runs where                     Auth/connection
------------  -----------------------------  ---------------------------------------
Built-in      Inside Logic Apps runtime      Standard: "service provider" connections (connection string / MI)
Managed       Shared Azure-hosted connector  API connection resource (Microsoft.Web/connections)
Custom        Your own OpenAPI/SOAP/ Postman Custom connector resource
On-prem       Via on-premises data gateway   Gateway resource + install on server
```
Built-in vs managed names are often similar ("Service Bus" both exist) with different capabilities.

### 6.1 [Easy] Managed vs built-in connectors?
**Answer**: Managed connectors are hosted and run by Azure in shared infrastructure and need an API connection; built-in connectors run natively inside the Logic Apps runtime for better performance and (in Standard) VNet reach.
**Explanation**: Built-in ones include HTTP, Request, Service Bus, Event Hubs, Azure Blob, Azure Queues, Cosmos, SQL (service provider). Standard also bills built-in differently.
**Code / JSON / Bicep**
```json
// Standard connections.json
{
  "serviceProviderConnections": {
    "serviceBus": {
      "parameterValues": { "fullyQualifiedNamespace": "@appsetting('SB_NS')", "authProvider": { "Type": "ManagedServiceIdentity" } },
      "serviceProvider": { "id": "/serviceProviders/serviceBus" },
      "displayName": "sb"
    }
  }
}
```

### 6.2 [Follow-up] What is an API connection?
**Answer**: An Azure resource (`Microsoft.Web/connections`) storing the auth configuration for a managed connector; the Logic App references it via `$connections` parameter.
**Explanation**: Connections have their own status (Connected/Error) and may need re-authorization (OAuth tokens expire, MFA/password changes). In Consumption, the connection resource needs an access policy for the Logic App identity if using MI.
**Code / JSON / Bicep**
```bicep
resource conn 'Microsoft.Web/connections@2016-06-01' = {
  name: 'azureblob'
  location: location
  properties: {
    displayName: 'blob'
    api: { id: subscriptionResourceId('Microsoft.Web/locations/managedApis', location, 'azureblob') }
    parameterValueSet: { name: 'managedIdentityAuth', values: {} }
  }
}
```

### 6.3 [Medium] When do you need the on-premises data gateway?
**Answer**: To reach on-prem or private-network resources (SQL Server, file shares, SAP) from managed connectors. It's installed on a machine, registered, and linked through an Azure Data Gateway resource.
**Explanation**: Gateway should be clustered for HA, sized properly, and has throughput limits. In Standard with VNet integration, built-in connectors can reach private endpoints directly, often removing the gateway need.
**Code / JSON / Bicep**
```text
Logic App -> managed connector -> Azure gateway resource -> on-prem gateway service -> SQL Server
```

### 6.4 [Medium] When do you build a custom connector?
**Answer**: When you want to reuse a REST/SOAP API across many workflows with consistent auth, schema and designer experience. Define from OpenAPI, Postman or scratch; supports OAuth2, API key, basic, Entra.
**Explanation**: For one-off calls, the HTTP action is enough. Custom connectors (Consumption) are separate resources; in Standard you can create custom built-in connectors (extensibility) - verify current support. Versioning needs care since workflows bind to connector definition.
**Code / JSON / Bicep**
```json
{ "swagger": "2.0", "info": { "title": "Orders", "version": "1.0" }, "host": "api.contoso.com", "paths": { "/orders/{id}": { "get": { "operationId": "GetOrder", "parameters": [ { "name": "id", "in": "path", "required": true, "type": "string" } ], "responses": { "200": { "description": "ok" } } } } } }
```

### 6.5 [Hard] Why do connectors throttle, and how do you design around it?
**Answer**: Each connector/connection has per-connection rate limits and the backend has its own; excessive parallel actions yield 429. Mitigate with lower concurrency, retry with backoff, batching, and multiple connections where permitted.
**Explanation**: Managed connector throttling is per connection, so many workflows sharing one connection compete. Retry-After header is honored by default retry policy. Check connector reference docs for numbers (verify in current docs).
**Code / JSON / Bicep**
```json
"retryPolicy": { "type": "exponential", "count": 4, "interval": "PT10S", "minimumInterval": "PT5S", "maximumInterval": "PT1H" }
```
**Gotchas / follow-ups**
- Switching to built-in connector often removes the throttle layer.

### 6.6 [Tricky] A connection shows "Unauthorized/Error" after months - why?
**Answer**: Interactive OAuth connections break when the user's password changes, token refresh lifetime lapses, MFA/Conditional Access applies, or the user leaves. Use managed identity or service principal connections to avoid user-bound tokens.
**Explanation**: Fix by re-authorizing the API connection, but the durable fix is identity-based auth. Also check that deployments did not re-create the connection resource unauthorized (ARM deploy yields unauthenticated connection for OAuth).
**Code / JSON / Bicep**
```text
Symptom: 401/InvalidAuthenticationToken, connection status "Error" in Portal -> API connections.
```

**Interview traps**
- Mixing up built-in and managed Service Bus capabilities.
- Personal-account connections in production.
- Forgetting gateway HA.

**Cheat sheet**
- Built-in = in-runtime; managed = API connection.
- Custom connector for reuse.
- Gateway for on-prem (or VNet in Standard).
- Prefer MI auth.
- Throttle: lower concurrency + retry.

## Topic 7: Authentication and managed identity

**Study notes**

```
Identity           Consumption                        Standard
-----------------  ---------------------------------  ----------------------------------
SAMI               1 per Logic App resource           1 per Logic App (app-level, shared by all workflows)
UAMI               Supported; assign to the Logic App Supported; shared across apps
Used by            HTTP action, Managed connectors    HTTP action, built-in service-provider connectors, managed connectors
                   (that support MI), Function action
RBAC               Grant on target resource (data-plane roles)
```
Typical roles: Service Bus Data Receiver/Sender, Storage Blob Data Contributor/Reader, Key Vault Secrets User (RBAC vault) or access policy.

### 7.1 [Easy] How does a Logic App authenticate to Azure resources without secrets?
**Answer**: Enable a managed identity on the Logic App, grant it an RBAC role on the target, and configure the action/connection to use Managed Identity authentication.
**Explanation**: Entra issues tokens for the identity; no secrets are stored. For HTTP actions you specify the `audience` (resource), e.g. `https://vault.azure.net` or `https://storage.azure.com`.
**Code / JSON / Bicep**
```json
"Get_secret": {
  "type": "Http",
  "inputs": {
    "method": "GET",
    "uri": "https://myvault.vault.azure.net/secrets/db-pass?api-version=7.4",
    "authentication": { "type": "ManagedServiceIdentity", "audience": "https://vault.azure.net" }
  },
  "runtimeConfiguration": { "secureData": { "properties": ["outputs"] } },
  "runAfter": {}
}
```

### 7.2 [Follow-up] SAMI vs UAMI for Logic Apps?
**Answer**: SAMI is tied to the Logic App lifecycle (deleted with it) and is simplest; UAMI is a separate resource, reusable across apps and pre-provisioned with RBAC, so it survives redeployments. UAMI is preferred for IaC pipelines.
**Explanation**: With SAMI, recreating the app creates a new principal, so role assignments are lost and must be re-applied (a classic deployment break). With UAMI, specify which identity the action uses (`identity` property in the authentication block).
**Code / JSON / Bicep**
```bicep
resource uami 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = { name: 'id-la', location: location }
resource la 'Microsoft.Logic/workflows@2019-05-01' = {
  name: 'la-orders'
  location: location
  identity: { type: 'UserAssigned', userAssignedIdentities: { '${uami.id}': {} } }
  properties: { definition: {} }
}
```
```json
"authentication": { "type": "ManagedServiceIdentity", "identity": "/subscriptions/.../userAssignedIdentities/id-la", "audience": "https://storage.azure.com/" }
```

### 7.3 [Medium] How do the Service Bus, Blob and Key Vault connectors use MI?
**Answer**: Built-in (Standard) Service Bus/Blob connectors accept a managed identity auth type with the namespace/account endpoint. Managed connectors support MI via connection auth type in a connection (verify per connector). Key Vault is reached either by the managed connector or an HTTP action with MI.
**Explanation**: Grant data roles (Azure Service Bus Data Receiver, Storage Blob Data Contributor). RBAC propagation can take minutes; 403 right after deployment is common. Key Vault with RBAC needs `Key Vault Secrets User`.
**Code / JSON / Bicep**
```bicep
resource rbac 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(sb.id, uami.id, 'receiver')
  scope: sb
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '4f6d3b9b-027b-4f4c-9142-0e5a2a2247e0') // Service Bus Data Receiver
    principalId: uami.properties.principalId
    principalType: 'ServicePrincipal'
  }
}
```

### 7.4 [Medium] Secure inputs/outputs - what are they?
**Answer**: A per-action setting (`secureData`) that hides inputs and/or outputs in run history and diagnostics. Use for secrets, tokens and PII.
**Explanation**: Secure outputs are also masked for downstream action inputs in history views, but the actual value still flows at runtime. Also use secure parameters (`securestring`) for deployment-time secrets. Request trigger can secure outputs too.
**Code / JSON / Bicep**
```json
"runtimeConfiguration": { "secureData": { "properties": ["inputs", "outputs"] } }
```

### 7.5 [Hard] Why does HTTP-with-MI return 401/403 even though the role is assigned?
**Answer**: Wrong audience, role assigned at wrong scope or to the wrong principal (SAMI recreated), propagation delay, Key Vault using access policies/network firewall, or data-plane role missing (control-plane Contributor does not grant data access).
**Explanation**: Decode the token (jwt.ms) via a test: check `aud`, `oid`. For storage, the audience is `https://storage.azure.com/`; for Service Bus `https://servicebus.azure.net`; Graph `https://graph.microsoft.com`. Firewalls on the target need trusted-service exception or VNet/private endpoint.
**Code / JSON / Bicep**
```text
403 AuthorizationPermissionMismatch  -> missing data role (Blob Data *)
401 invalid audience                  -> fix "audience"
403 Forbidden by firewall            -> network rule, not RBAC
```
**Gotchas / follow-ups**
- Contributor/Owner do NOT grant blob data access.
- HTTP MI calls only work for Entra-protected endpoints.

### 7.6 [Tricky] In Standard, can each workflow have its own identity?
**Answer**: No - the identity is app-level (Logic App Standard resource); all workflows in the app share it. Isolate by splitting apps if least privilege per workflow is required.
**Explanation**: With UAMI you can attach several identities and pick per connection/action, giving some separation. Connection-level `Authentication` selects which identity.
**Code / JSON / Bicep**
```json
"authProvider": { "Type": "ManagedServiceIdentity", "Identity": "/subscriptions/.../userAssignedIdentities/id-sb" }
```
**Gotchas / follow-ups**
- Over-privileged shared identities are a security-review finding.

**Interview traps**
- Contributor does not give data-plane access.
- SAMI is lost on resource recreate.
- Missing `audience` in HTTP action.
- RBAC propagation delay.

**Cheat sheet**
- Enable MI -> RBAC -> choose MI auth in action.
- UAMI for IaC stability.
- Audience per service.
- `secureData` for secrets.
- Standard identity is app-wide.

---

## Topic 8: Error handling, retries, idempotency

**Study notes**

Patterns: (1) retry policy per action, (2) `runAfter` handlers, (3) Scope as try/catch/finally, (4) Terminate with status, (5) dead-letter/poison handling, (6) idempotency.

```
[Scope: Try] --Succeeded--> [Continue]
     |
     +--Failed/TimedOut--> [Scope: Catch] --> [Terminate Failed]
[Scope: Finally] runAfter Try+Catch with [Succeeded, Failed, Skipped, TimedOut]
```

### 8.1 [Easy] What retry policies exist?
**Answer**: Default (exponential, 4 retries), None, Fixed interval, Exponential interval. Retries apply to 408, 429 and 5xx (and connection failures) - not 4xx like 400/401/404.
**Explanation**: Configure per action via `retryPolicy`. Default exponential interval is around 7 seconds growing; verify exact defaults in docs. Retry-After header respected for 429.
**Code / JSON / Bicep**
```json
"inputs": {
  "method": "POST", "uri": "https://api/x",
  "retryPolicy": { "type": "fixed", "count": 3, "interval": "PT20S" }
}
```

### 8.2 [Follow-up] How do you implement try/catch/finally?
**Answer**: Put actions inside a Scope (Try), a second Scope (Catch) with `runAfter` Failed/TimedOut on Try, and a Finally scope running after both with all statuses. Use `result('Try')` to inspect inner action results.
**Explanation**: `result('ScopeName')` returns an array of action results; filter with Filter array where status is Failed to build an error message.
**Code / JSON / Bicep**
```json
"Catch": {
  "type": "Scope",
  "runAfter": { "Try": ["Failed", "TimedOut"] },
  "actions": {
    "Errors": {
      "type": "Query",
      "inputs": { "from": "@result('Try')", "where": "@equals(item()?['status'], 'Failed')" },
      "runAfter": {}
    },
    "Fail_run": {
      "type": "Terminate",
      "inputs": { "runStatus": "Failed", "runError": { "code": "ProcessingFailed", "message": "@{first(body('Errors'))?['error']?['message']}" } },
      "runAfter": { "Errors": ["Succeeded"] }
    }
  }
}
```

### 8.3 [Medium] Terminate action behavior?
**Answer**: Terminate ends the run immediately with status Failed, Cancelled or Succeeded, optionally with an error code/message.
**Explanation**: Without Terminate, a handled failure can still leave the run marked Failed because the failed action/scope counts toward the run result; use Terminate with Succeeded to mark handled errors intentionally. For Request/Response workflows, return a Response before terminating.
**Code / JSON / Bicep**
```json
"Done": { "type": "Terminate", "inputs": { "runStatus": "Succeeded" }, "runAfter": {} }
```

### 8.4 [Medium] Dead-lettering and poison-message patterns?
**Answer**: With Service Bus peek-lock, explicitly dead-letter bad messages (managed connector action or built-in) with a reason, or let delivery count exceed and auto DLQ. Alternatively route failures to an error queue/Blob with the payload and error context.
**Explanation**: Separate "transient" (retry) from "permanent" (DLQ). Build a DLQ reprocessing workflow (read from `<queue>/$deadletterqueue`), fix, and resubmit. Always store original message + run ID.
**Code / JSON / Bicep**
```text
Transient (429/5xx/timeouts)  -> retry policy, abandon (message redelivered)
Permanent (400, schema fail)   -> dead-letter with reason, alert
```

### 8.5 [Hard] How do you make a workflow idempotent?
**Answer**: Derive a deterministic key (MessageId, business id), check/record it in a store (Table/Cosmos/Redis/SQL unique constraint) before side effects, and use idempotent APIs (PUT, upsert, If-Match, idempotency-key headers).
**Explanation**: Retry plus at-least-once delivery produce duplicates, e.g. HTTP action timed out but the downstream succeeded. Use conditional insert: attempt insert with unique key; on conflict skip. Redis `SET key NX EX` works as a lightweight lock/dedupe.
**Code / JSON / Bicep**
```json
"Call_API": {
  "type": "Http",
  "inputs": {
    "method": "POST", "uri": "https://api/orders",
    "headers": { "Idempotency-Key": "@{triggerBody()?['messageId']}" },
    "body": "@triggerBody()"
  },
  "runAfter": {}
}
```
**Gotchas / follow-ups**
- Dedupe store itself must handle concurrent runs (use atomic ops).
- Service Bus duplicate detection only covers sender-side window.

### 8.6 [Tricky] HTTP action times out but the downstream already processed - what happens on retry?
**Answer**: The retry sends the request again, possibly creating duplicates, because timeout means "unknown outcome". Only idempotent endpoints or keys make that safe.
**Explanation**: HTTP action has a request timeout (verify current limit); `timeout` can be set via `limit.timeout`. Disable retries for non-idempotent POSTs without keys (`retryPolicy: none`) and reconcile separately.
**Code / JSON / Bicep**
```json
"inputs": { "method": "POST", "uri": "...", "retryPolicy": { "type": "none" } },
"limit": { "timeout": "PT30S" }
```
**Gotchas / follow-ups**
- `Failed` vs `TimedOut` status both need handling in runAfter.

**Interview traps**
- Assuming 4xx are retried.
- Handled failures still show run as Failed.
- No idempotency despite retries.
- Ignoring `TimedOut` in runAfter.

**Cheat sheet**
- Retry: 408/429/5xx; default 4 exponential.
- Try/Catch/Finally with Scopes + `result()`.
- Terminate sets final status.
- DLQ permanent errors.
- Idempotency key + atomic dedupe.

---

## Topic 9: Calling Functions/APIs, async patterns, limits

**Study notes**

```
Sync call      : HTTP action -> waits for response (timeout applies)
Async 202 poll : HTTP action returns 202 + Location header -> engine polls automatically (default asynchronous pattern)
Webhook        : Logic App registers callback URL (HTTP Webhook / Wait for callback), resumes when called
```
By default the HTTP action honors the 202 + Location pattern ("Asynchronous pattern" on). Limits (run duration, request timeout, message size, loop size, concurrency, throttling) differ per plan - verify in current docs.

### 9.1 [Easy] How do you call an Azure Function from a Logic App?
**Answer**: Use the Azure Functions action (select function app/function) or an HTTP action to the function URL, preferably with Entra/MI authentication instead of function keys.
**Explanation**: The built-in Functions action works with HTTP-triggered functions and keys. For Standard, HTTP with MI and Easy Auth on the function app gives key-less security.
**Code / JSON / Bicep**
```json
"Call_func": {
  "type": "Http",
  "inputs": {
    "method": "POST",
    "uri": "https://func-app.azurewebsites.net/api/Enrich",
    "body": "@triggerBody()",
    "authentication": { "type": "ManagedServiceIdentity", "audience": "api://<function-app-client-id>" }
  },
  "runAfter": {}
}
```

### 9.2 [Follow-up] How does the asynchronous 202 pattern work?
**Answer**: If an HTTP action receives 202 with a `Location` (and optional `Retry-After`), the engine polls the Location URL until a non-202 result and uses that as the action's output. Disable via `DisableAsyncPattern` option when you want the raw 202.
**Explanation**: Great for long APIs/Functions (Durable Functions' `statusQueryGetUri` pattern works here). The action stays running during polling and counts against run/action duration limits.
**Code / JSON / Bicep**
```json
"Start_job": {
  "type": "Http",
  "inputs": { "method": "POST", "uri": "https://func/api/orchestrators/Job", "body": "@triggerBody()" },
  "operationOptions": "DisableAsyncPattern",
  "runAfter": {}
}
```

### 9.3 [Medium] Webhooks/callbacks vs polling for long-running work?
**Answer**: Webhook actions (HTTP Webhook, Wait for callback with `listCallbackUrl()`) let the Logic App sleep without polling until the system calls back. Cheaper and lower latency than Until-polling, but the callee must support calling back.
**Explanation**: Include the callback URL in the request; secure it (it contains a signature). Set an action timeout so runs don't hang forever.
**Code / JSON / Bicep**
```json
"Wait_for_approval": {
  "type": "HttpWebhook",
  "inputs": {
    "subscribe": { "method": "POST", "uri": "https://api/approvals", "body": { "callback": "@{listCallbackUrl()}" } },
    "unsubscribe": { "method": "POST", "uri": "https://api/approvals/cancel" }
  },
  "limit": { "timeout": "P2D" },
  "runAfter": {}
}
```

### 9.4 [Medium] What limits matter most?
**Answer**: Run duration, HTTP request timeout, message/content size, foreach/array item limits, trigger/run concurrency, runs waiting, and connector throttling. Numbers differ for Consumption vs Standard and change - verify in current docs.
**Explanation**: Historically (verify): Consumption stateful run duration 90 days, HTTP sync request timeout ~2 min (230s through front door for Request trigger responses), message size ~100 MB with chunking for supporting connectors, foreach 100k items. Stateless runs are short. Retention of run history is configurable (default ~90 days).
**Code / JSON / Bicep**
```text
Design rule: anything near a limit -> chunk, offload to Blob, claim-check pattern, or move to Functions/ADF.
```

### 9.5 [Hard] Request-response workflow takes longer than the caller's timeout - fix?
**Answer**: Return 202 Accepted immediately with a status URL, process asynchronously, and let the client poll or receive a webhook. Do not hold the HTTP connection open for minutes.
**Explanation**: The Request trigger's synchronous response has a time limit (~2 min historically - verify). Use a Response action early (`202` with `Location`), then continue work after it. The status can be stored in Table/Cosmos.
**Code / JSON / Bicep**
```json
"Respond_202": {
  "type": "Response", "kind": "Http",
  "inputs": { "statusCode": 202, "headers": { "Location": "https://status/api/jobs/@{workflow().run.name}", "Retry-After": "10" } },
  "runAfter": {}
}
```
**Gotchas / follow-ups**
- After Response, remaining actions still run in the same run.
- APIM in front may have its own timeout.

### 9.6 [Tricky] Run is "Running" for days / stuck - why?
**Answer**: Waiting on a webhook/callback that never arrived, an Until loop without proper exit, a Delay, a long approval, or "Waiting" due to concurrency limits queueing.
**Explanation**: Check run details for the action in Running state. Add timeouts on webhook actions and `limit.timeout` on Until. Cancel the run if needed.
**Code / JSON / Bicep**
```text
Status Waiting  -> concurrency/maximumWaitingRuns queue
Status Running  -> an action (webhook/delay/until) still in progress
```

**Interview traps**
- Holding HTTP open for long jobs.
- Not setting webhook timeouts.
- Quoting stale numeric limits.

**Cheat sheet**
- 202 + Location auto-poll.
- Webhook = callback URL.
- Respond early then continue.
- Use MI + audience for Functions.
- Verify limits.

## Topic 10: Service Bus integration patterns

**Study notes**

```
Trigger options (names vary by plan - verify):
  Built-in (Standard): "When messages are available in a queue/topic subscription" (auto-complete or peek-lock, batch)
  Managed: "When a message is received in a queue (auto-complete)" / "(peek-lock)"
Peek-lock: message locked (default 30s-5min lock duration), must Complete/Abandon/Dead-letter before lock expiry.
Sessions: FIFO per session id; session-enabled entities need session-aware trigger.
```

### 10.1 [Easy] How do you trigger a workflow from a Service Bus queue?
**Answer**: Use the Service Bus trigger (built-in in Standard or managed connector), choose queue or topic subscription and a receive mode. Auto-complete removes the message when the run starts; peek-lock lets you complete it after successful processing.
**Explanation**: Built-in trigger uses a managed identity or connection string. Polling interval applies to managed connector; built-in is listener-based. 
**Code / JSON / Bicep**
```json
"triggers": {
  "When_messages_are_available_in_a_queue": {
    "type": "ServiceProvider",
    "inputs": {
      "parameters": { "queueName": "orders", "isSessionsEnabled": false },
      "serviceProviderConfiguration": { "connectionName": "serviceBus", "operationId": "receiveQueueMessages", "serviceProviderId": "/serviceProviders/serviceBus" }
    }
  }
}
```

### 10.2 [Follow-up] Peek-lock vs auto-complete - which and why?
**Answer**: Use peek-lock for reliability: complete only after the work succeeds, abandon or dead-letter on failure. Auto-complete risks losing the message if the run fails after receipt.
**Explanation**: Lock expiry before completion causes redelivery (duplicates) - long workflows must renew the lock or use a pattern that persists work then completes quickly. Managed peek-lock trigger has a lock renewal option; verify behavior per connector.
**Code / JSON / Bicep**
```text
Run succeeds   -> Complete message
Transient fail -> Abandon (DeliveryCount++), retried
Permanent fail -> Dead-letter with reason; MaxDeliveryCount reached -> auto DLQ
```

### 10.3 [Medium] Topic subscriptions and filtering?
**Answer**: One workflow (or Logic App) per subscription, with SQL/correlation filters on the subscription to route messages; keeps workflows simple and independently scalable.
**Explanation**: Filter at the broker rather than with a Condition (saves runs). Each subscription has its own DLQ.
**Code / JSON / Bicep**
```bicep
resource rule 'Microsoft.ServiceBus/namespaces/topics/subscriptions/rules@2022-10-01-preview' = {
  parent: sub
  name: 'eu-orders'
  properties: { filterType: 'SqlFilter', sqlFilter: { sqlExpression: 'region = \'EU\'' } }
}
```

### 10.4 [Medium] Sessions - how do they work with Logic Apps?
**Answer**: Enable sessions on the queue/subscription; the session-aware trigger locks a session and processes its messages in order. Run concurrency maps to number of sessions handled in parallel.
**Explanation**: Messages without a SessionId are rejected by a session-enabled entity. Ordering is preserved per session only if concurrency within a session is 1 - do not SplitOn/parallelize inside it.
**Code / JSON / Bicep**
```text
sessionId = orderId  -> all events for an order processed in order, different orders in parallel.
```

### 10.5 [Hard] Processing the dead-letter queue?
**Answer**: A separate workflow triggered on `<queue>/$deadletterqueue` (managed trigger supports DLQ option) that logs, alerts, optionally repairs and resubmits to the main queue, preserving original metadata.
**Explanation**: Read `DeadLetterReason` and `DeadLetterErrorDescription` properties. Avoid infinite loops: add a resubmission counter property. Do not auto-resubmit blindly.
**Code / JSON / Bicep**
```text
Entity path: orders/$deadletterqueue    (subscription: topic/subscriptions/sub/$deadletterqueue)
@triggerBody()?['DeadLetterReason']
```
**Gotchas / follow-ups**
- Messages dead-lettered by TTL expiry only if DeadLetteringOnMessageExpiration is enabled.

### 10.6 [Tricky] Duplicate runs / lock lost / messages stuck - causes?
**Answer**: Lock duration shorter than processing time (redelivery), run concurrency too high causing downstream throttling and failures, retries after partial success, or prefetch grabbing messages that then expire locks.
**Explanation**: Match lock duration to P99 processing; cap concurrency (e.g., 1-10); set MaxDeliveryCount sensibly (default 10); make handlers idempotent. Batching: receive N messages per run and complete them individually or as a batch; a single bad message should not fail the batch (handle per message).
**Code / JSON / Bicep**
```json
"runtimeConfiguration": { "concurrency": { "runs": 10 } }
```
**Gotchas / follow-ups**
- Standard built-in trigger scale-out can increase parallelism across instances - verify settings.
- Cost: each message = run in Consumption.

**Interview traps**
- Auto-complete in critical flows.
- Ignoring lock expiry.
- Auto-resubmitting DLQ forever.

**Cheat sheet**
- Peek-lock + complete/abandon/DLQ.
- Filters at subscription.
- Sessions = ordered per key.
- Cap concurrency; idempotent handlers.
- DLQ workflow with reason logging.

---

## Topic 11: Blob and Event Grid patterns

**Study notes**

```
Blob polling trigger (managed)  -> latency, last-modified cursor, miss/dup edge cases
Event Grid (BlobCreated)        -> push, near real time, filter by subject prefix/suffix
Built-in Blob trigger (Standard)-> polling-based in-app; verify behavior in docs
```
Event Grid delivers at-least-once with retries and its own dead-letter to Blob.

### 11.1 [Easy] How do you react to a new blob?
**Answer**: Preferred: Event Grid subscription on the storage account (BlobCreated event) to the Logic App's Event Grid trigger or a Request trigger, then Get blob content. Alternative: the Blob polling trigger.
**Explanation**: Event Grid is push and scalable; polling is simpler but slower/costly. Event contains blob URL, not content.
**Code / JSON / Bicep**
```json
{ "topic": "...", "subject": "/blobServices/default/containers/in/blobs/a.csv", "eventType": "Microsoft.Storage.BlobCreated", "data": { "url": "https://acct.blob.core.windows.net/in/a.csv", "contentLength": 1234 } }
```

### 11.2 [Follow-up] Event Grid webhook validation and filtering?
**Answer**: Event Grid performs a subscription validation handshake; the Logic App Event Grid trigger handles it automatically. Use subject-begins-with / ends-with and event type filters to cut noise.
**Explanation**: Using Request trigger as endpoint requires handling validation code response yourself (or use the Event Grid connector trigger). Filter `/containers/in/` and `.csv`.
**Code / JSON / Bicep**
```bicep
resource sub 'Microsoft.EventGrid/systemTopics/eventSubscriptions@2022-06-15' = {
  parent: topic
  name: 'blob-created'
  properties: {
    destination: { endpointType: 'WebHook', properties: { endpointUrl: callbackUrl } }
    filter: { includedEventTypes: ['Microsoft.Storage.BlobCreated'], subjectBeginsWith: '/blobServices/default/containers/in/', subjectEndsWith: '.csv' }
  }
}
```

### 11.3 [Medium] Reading blob content safely (large files)?
**Answer**: Use Get blob content with MI auth; for large files avoid loading everything into the workflow - use chunked download, or hand off to a Function/ADF and pass only the blob URL (claim check).
**Explanation**: Managed connector size limit for content (verify; ~50 MB default without chunking, up to ~1 GB with chunking on supported actions). Built-in Standard Blob connector has different limits.
**Code / JSON / Bicep**
```json
"Get_blob": {
  "type": "Http",
  "inputs": { "method": "GET", "uri": "@triggerBody()?['data']?['url']", "authentication": { "type": "ManagedServiceIdentity", "audience": "https://storage.azure.com/" }, "headers": { "x-ms-version": "2021-08-06" } },
  "runAfter": {}
}
```

### 11.4 [Hard] Avoiding reprocessing and handling partial files?
**Answer**: Blob created events can fire before the writer finishes (multi-block uploads fire on final commit, but producers writing in place can overwrite). Use a staging container + rename/copy to a "ready" container or a `.done` marker blob, and track processed ETags.
**Explanation**: Event Grid is at-least-once, so dedupe by blob URL + ETag. Move processed blobs to an archive container and delete or lifecycle-manage them.
**Code / JSON / Bicep**
```text
Pattern: upload to /staging -> producer renames/copies to /inbox -> event -> process -> move to /archive
```

### 11.5 [Tricky] Event Grid trigger not firing?
**Answer**: Check the subscription provisioning state, endpoint validation, filters, firewall on storage (event delivery needs trusted-service path/system topic), the Logic App being disabled, or delivery failing and being retried/dead-lettered.
**Explanation**: Use Event Grid metrics (Delivery failed, Dropped), enable dead-letter destination, and check Logic App trigger history. For private/firewalled Logic Apps (Standard private endpoint), Event Grid push cannot reach it - use a public path with restrictions, APIM, or an Event Grid -> Service Bus/Event Hubs relay (verify options).
**Code / JSON / Bicep**
```kusto
AzureMetrics | where ResourceProvider == "MICROSOFT.EVENTGRID" and MetricName in ("DeliveryFailedCount","DroppedEventCount")
```
**Gotchas / follow-ups**
- Deleting/recreating the Logic App changes callback URL -> subscription breaks.

**Interview traps**
- Polling blob trigger for high-volume.
- Not handling duplicate events.
- Loading huge blobs into workflow memory.

**Cheat sheet**
- Event Grid push > blob poll.
- Filter by prefix/suffix.
- Claim-check for big files.
- Dedupe by URL+ETag.
- Staging -> ready container.

---

## Topic 12: B2B/EDI, integration account, transforms

**Study notes**

```
Integration account (Consumption: required for B2B/maps/schemas; Standard: built-in artifacts can be stored in the app, B2B connectors may still use an account - verify)
Artifacts: Partners, Agreements, Schemas (XSD), Maps (XSLT; supported versions - verify in current docs, Liquid), Certificates, Assemblies
Protocols: AS2, X12, EDIFACT, flat file encode/decode
```
Tiers: Free, Basic, Standard - verify capabilities/pricing in current docs.

### 12.1 [Easy] What is an integration account?
**Answer**: A container for B2B artifacts - trading partners, agreements, schemas, maps, certificates - linked to Logic Apps that need EDI/XML transformation or validation.
**Explanation**: Consumption workflows link to one integration account; Standard can reference artifacts in-app for maps and schemas. Needed for AS2/X12/EDIFACT agreements.
**Code / JSON / Bicep**
```bicep
resource ia 'Microsoft.Logic/integrationAccounts@2019-05-01' = {
  name: 'ia-b2b'
  location: location
  sku: { name: 'Standard' }
  properties: {}
}
```

### 12.2 [Follow-up] XSLT vs Liquid transforms?
**Answer**: XSLT transforms XML to XML (or text) via a map; Liquid maps JSON to JSON/text using templates. Choose by payload format; Liquid is friendlier for JSON.
**Explanation**: The "Transform XML" action uses an XSLT map; "Transform JSON to JSON/JSON to Text/XML to JSON/etc." use Liquid templates. XSLT version support is limited; verify in current docs. Complex logic may still be best in a Function.
**Code / JSON / Bicep**
```liquid
{
  "orderId": "{{ content.id }}",
  "lines": [ {% for l in content.lines %}{ "sku": "{{ l.sku }}", "qty": {{ l.qty }} }{% unless forloop.last %},{% endunless %}{% endfor %} ]
}
```

### 12.3 [Medium] Schema validation and flat files?
**Answer**: XML Validation checks XML against an XSD schema; Flat File Decoding/Encoding use a flat-file schema. Validation failures should route to an error path/DLQ.
**Explanation**: Parse JSON validates only structure you define (and failures throw). Use XSD to enforce enums/required fields for XML-based partners.
**Code / JSON / Bicep**
```json
"Transform": {
  "type": "Xslt",
  "inputs": { "content": "@triggerBody()", "integrationAccount": { "map": { "name": "OrderToInvoice" } } },
  "runAfter": {}
}
```

### 12.4 [Hard] How do X12/EDIFACT/AS2 flows work end to end?
**Answer**: AS2 receive decodes/verifies the message and sends MDN; X12/EDIFACT decode validates against the agreement/schema, splits interchanges into transaction sets and generates acknowledgments (997/CONTRL); then transform and route internally. Outbound is the reverse.
**Explanation**: Requires partners, identities, agreements with settings (ack, validation, separators) and certificates for signing/encryption. Track with B2B tracking schema in Log Analytics.
**Code / JSON / Bicep**
```text
Partner -> AS2 decode -> X12 decode -> (997 ack) -> XSLT map -> ERP API
```
**Gotchas / follow-ups**
- Certificates expire - monitor.
- Large interchanges hit message size limits.

### 12.5 [Tricky] Map works in a test tool but fails in Logic Apps?
**Answer**: Differences in XSLT version/extensions, namespaces, encoding/BOM, or Liquid dotliquid vs Shopify Liquid syntax differences (Logic Apps uses DotLiquid, with its own filters, e.g. different naming and case behavior).
**Explanation**: Liquid in Logic Apps: property names are case-sensitive in some contexts and filters follow DotLiquid. XML with BOM or wrong content-type passes strings, not XML. Test in the action with real runs.
**Code / JSON / Bicep**
```text
Check: content-type, BOM stripped, namespaces declared, DotLiquid-supported filters only.
```

**Interview traps**
- Thinking Standard needs no artifact storage planning.
- Using Liquid for XML payloads.
- Forgetting cert expiry.

**Cheat sheet**
- Integration account = B2B artifacts.
- XSLT for XML, Liquid (DotLiquid) for JSON.
- X12/EDIFACT/AS2 need agreements.
- Validate then route errors.
- Verify tiers in docs.

---

## Topic 13: JSON, arrays, large data, chunking

**Study notes**

Data operations: Parse JSON, Compose, Select, Filter array, Join, Create CSV/HTML table, Query. Select/Filter are single actions vs a Foreach (cheaper and faster).

### 13.1 [Easy] What does Parse JSON do and why use it?
**Answer**: It validates JSON against a schema and exposes typed dynamic-content tokens. Required only for designer tokens; you can also access properties via expressions without it.
**Explanation**: Generate schema from a sample payload. A mismatch between payload and schema fails the action (e.g., null for a `string` type).
**Code / JSON / Bicep**
```json
"Parse_JSON": {
  "type": "ParseJson",
  "inputs": {
    "content": "@triggerBody()",
    "schema": { "type": "object", "properties": { "id": { "type": "string" }, "note": { "type": ["string","null"] } }, "required": ["id"] }
  },
  "runAfter": {}
}
```

### 13.2 [Follow-up] Select vs Foreach for reshaping arrays?
**Answer**: Select maps an array to a new array in one action; Foreach does per-item actions. Prefer Select/Filter for pure transformations - fewer actions, lower cost, faster.
**Explanation**: Select `from` array and `select` object map. Combine with `union`, `Join`, `Create CSV table`.
**Code / JSON / Bicep**
```json
"Select": {
  "type": "Select",
  "inputs": { "from": "@body('Parse_JSON')?['items']", "select": { "sku": "@item()?['sku']", "total": "@mul(item()?['qty'], item()?['price'])" } },
  "runAfter": {}
}
```

### 13.3 [Medium] Parse JSON pitfalls?
**Answer**: Nullable fields declared as non-null strings, missing `required` fields, numbers vs strings, schema generated from one sample being too strict, and arrays of mixed types.
**Explanation**: Allow `["string","null"]`, drop unneeded `required`, or skip Parse JSON and use `?['prop']` with defaults. Failures surface as `InvalidTemplate`/ValidationFailed.
**Code / JSON / Bicep**
```text
Error: "Invalid type. Expected String but got Null."  -> use ["string","null"]
```

### 13.4 [Hard] Large data handling strategies?
**Answer**: Claim-check: store the payload in Blob and pass references; use connector chunking where supported; stream to Functions/ADF; paginate API calls; avoid giant foreach loops. Respect message size and array limits (verify current numbers).
**Explanation**: Run history stores inputs/outputs for stateful workflows, so big payloads inflate storage and slow the portal. Pagination: enable "Pagination" setting on supporting actions with threshold, or loop with Until on a continuation token.
**Code / JSON / Bicep**
```json
"runtimeConfiguration": { "paginationPolicy": { "minimumItemCount": 5000 } }
```
**Gotchas / follow-ups**
- Pagination threshold doesn't guarantee exactly that number of items.
- Foreach over 100k items: chunk with `take`/`skip` or SplitOn.

### 13.5 [Tricky] Chunking an array into batches
**Answer**: No built-in batch function for arbitrary arrays; use `chunk()` if available (it exists in the expression language - verify in current docs) or a loop with `take()`/`skip()`.
**Explanation**: Process batches sequentially or with low concurrency to respect downstream throttles. Remove processed items by `skip` with an index variable.
**Code / JSON / Bicep**
```text
@chunk(body('Select'), 100)
@take(skip(variables('all'), mul(variables('i'), 100)), 100)
```
**Gotchas / follow-ups**
- Variable-based loops require sequential foreach/Until.

**Interview traps**
- Overusing Foreach where Select works.
- Strict schemas with nulls.
- Large payloads in run history.

**Cheat sheet**
- Select/Filter > Foreach.
- `["string","null"]` for nullable.
- Claim-check pattern.
- Pagination policy.
- chunk/take/skip.

## Topic 14: Monitoring, resubmit, rerun

**Study notes**

```
Run history (portal)   : per-action inputs/outputs, retention configurable (verify default)
Diagnostic settings    : WorkflowRuntime logs -> Log Analytics / Storage / Event Hub
App Insights (Standard): native integration for traces/telemetry
Tracked properties     : custom keys on actions, emitted to diagnostics
Alerts                 : metrics (Runs Failed, Action Failed, Throttled) + KQL log alerts
```

### 14.1 [Easy] Where do you see why a run failed?
**Answer**: Logic App > Run history > open the run, find the red action and read its inputs, outputs and error. For history across runs use Log Analytics.
**Explanation**: Secure inputs/outputs appear masked. Stateless workflows retain no history unless enabled for debugging.
**Code / JSON / Bicep**
```text
Portal: Overview > Runs history > failed run > failed action > Error details / Raw outputs
```

### 14.2 [Follow-up] How do you send logs to Log Analytics?
**Answer**: Add a diagnostic setting on the Logic App with the `WorkflowRuntime` category (and metrics) to a Log Analytics workspace, then query the `AzureDiagnostics` (or resource-specific) tables.
**Explanation**: Table naming depends on destination type (AzureDiagnostics vs resource-specific `LogicAppWorkflowRuntime`; verify). Enable before incidents; logs are not retroactive.
**Code / JSON / Bicep**
```bicep
resource diag 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = {
  name: 'to-law'
  scope: la
  properties: {
    workspaceId: law.id
    logs: [ { category: 'WorkflowRuntime', enabled: true } ]
    metrics: [ { category: 'AllMetrics', enabled: true } ]
  }
}
```

### 14.3 [Medium] KQL to find failed runs and actions?
**Answer**: Filter on `status_s == "Failed"` for run and action events, grouping by workflow and error code; join by `resource_runId_s` for a run's actions.
**Explanation**: Column names depend on table mode - verify in your workspace. Use `summarize` over time for trends.
**Code / JSON / Bicep**
```kusto
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.LOGIC" and Category == "WorkflowRuntime"
| where status_s == "Failed"
| summarize failures = count() by resource_workflowName_s, code_s, bin(TimeGenerated, 1h)
| order by failures desc
```

### 14.4 [Medium] Tracked properties and correlation?
**Answer**: Tracked properties add business keys (orderId) to an action's diagnostic record, so you can search logs by business identifier without exposing the whole payload.
**Explanation**: Set `trackedProperties` on an action; they appear in the log's `trackedProperties_*` fields. Also propagate `x-ms-client-tracking-id`.
**Code / JSON / Bicep**
```json
"Process": {
  "type": "Compose",
  "inputs": "@triggerBody()",
  "trackedProperties": { "orderId": "@{triggerBody()?['orderId']}" },
  "runAfter": {}
}
```

### 14.5 [Hard] Resubmit vs rerun - differences and pitfalls?
**Answer**: Resubmit re-executes a run using the original trigger inputs (new run ID). It needs a stateful workflow with retained history. Standard may also offer rerun-from-action options - verify in current docs.
**Explanation**: Resubmit repeats side effects, so non-idempotent steps duplicate. Webhook/Request responses are not delivered to the original caller. Time-based expressions (`utcNow`) will differ. Trigger body must still be retained (run history retention).
**Code / JSON / Bicep**
```text
Resubmit API: POST .../workflows/{name}/triggers/{trigger}/histories/{historyName}/resubmit
```
**Gotchas / follow-ups**
- Bulk resubmit scripts must throttle.
- Resubmit not possible for expired history or stateless runs.

### 14.6 [Tricky] Alerts that matter and gaps?
**Answer**: Alert on Runs Failed, Actions Failed, trigger failures, throttled events and latency, but note failed-run alerts miss "stuck waiting" and "trigger never fired" - add a heartbeat (expected runs per hour) alert.
**Explanation**: A trigger that silently stops produces zero failures. Use KQL "absence" alerts, or log alert with `count == 0` over a window; also Service Bus active message count/DLQ alerts upstream.
**Code / JSON / Bicep**
```kusto
AzureDiagnostics | where Category == "WorkflowRuntime" and OperationName has "workflow.run" | where TimeGenerated > ago(1h) | summarize runs = count()
// alert when runs == 0 during business hours
```

**Interview traps**
- Not enabling diagnostics until after an incident.
- Resubmit creating duplicates.
- Alerting only on failures.

**Cheat sheet**
- Run history for single runs; LAW for fleets.
- Diagnostic category WorkflowRuntime.
- Tracked properties for business IDs.
- Resubmit = same trigger payload, new run.
- Heartbeat alert for silence.

---

## Topic 15: CI/CD, IaC, testing

**Study notes**

```
Consumption: ARM/Bicep template (definition + parameters + $connections), deploy with az deployment
Standard   : app-style - build artifact zip (workflows, connections.json, parameters.json), deploy via az functionapp deployment / ZipDeploy / GitHub Actions / Azure DevOps
Per-env    : parameters.json + app settings (@appsetting('X')), connection resource IDs via parameters
```

### 15.1 [Easy] How do you deploy Consumption Logic Apps with IaC?
**Answer**: Author the workflow as ARM/Bicep (`Microsoft.Logic/workflows`) with the definition and parameters, plus `Microsoft.Web/connections` for API connections, deployed via pipeline.
**Explanation**: Use the VS Code/portal "Export template" or Logic Apps tooling to produce templates, then parameterize.
**Code / JSON / Bicep**
```bicep
param location string = resourceGroup().location
param sbConnId string
resource la 'Microsoft.Logic/workflows@2019-05-01' = {
  name: 'la-orders'
  location: location
  properties: {
    definition: loadJsonContent('definition.json')
    parameters: { '$connections': { value: { servicebus: { connectionId: sbConnId, connectionName: 'servicebus', id: subscriptionResourceId('Microsoft.Web/locations/managedApis', location, 'servicebus') } } } }
  }
}
```

### 15.2 [Follow-up] How do you deploy Standard?
**Answer**: Provision the Logic App Standard + plan + storage via Bicep, then zip-deploy the workflow project (build artifact) in the pipeline, and set app settings for environment values.
**Explanation**: Workflows are files; run `zip` of the project root, deploy with `az webapp deploy`/`az functionapp deployment source config-zip` or the Azure/functions-action/GitHub action. Test by starting a slot or separate env.
**Code / JSON / Bicep**
```powershell
Compress-Archive -Path .\MyLogicApp\* -DestinationPath app.zip -Force
az functionapp deployment source config-zip -g rg-app -n la-std-orders --src app.zip
```

### 15.3 [Medium] Parameters and connections per environment - avoid hardcoded IDs?
**Answer**: Parameterize connection IDs/endpoints (ARM parameters for Consumption; `parameters.json` + `@appsetting()`/`@parameters()` for Standard) and use UAMI/RBAC created by IaC. Never commit subscription IDs, connection strings or secrets.
**Explanation**: Portal-exported definitions hardcode `/subscriptions/.../connections/xyz`. Replace with template functions. Secrets via Key Vault references in app settings.
**Code / JSON / Bicep**
```json
// parameters.json (Standard)
{ "sbNamespace": { "type": "String", "value": "@appsetting('SB_NAMESPACE')" } }
```
```text
App setting: SB_NAMESPACE = sb-prod.servicebus.windows.net
Key Vault ref: @Microsoft.KeyVault(SecretUri=https://kv/secrets/x/)
```

### 15.4 [Hard] Why do deployments break connections?
**Answer**: OAuth API connections redeployed via ARM come up unauthenticated; SAMI is recreated (new principal, lost RBAC); connection IDs point at the wrong env; or the connection name changes. Use MI auth and IaC-managed role assignments (preferably UAMI).
**Explanation**: Check `Microsoft.Web/connections` status "Error" and unauthorized; for MI connections confirm access policy/RBAC. Idempotent redeploys with unchanged names avoid reauthorization.
**Code / JSON / Bicep**
```text
Symptom after deploy: 401/403, "connection not authorized". Fix: authorize connection or switch to MI + role assignment in the same deployment (dependsOn).
```
**Gotchas / follow-ups**
- Role assignment propagation lag right after deploy.
- Complete-mode deployments can delete connections.

### 15.5 [Tricky] How do you test Logic Apps?
**Answer**: Standard has a local run/debug experience and a (preview) unit-testing framework for workflows (verify current status) with mocked trigger/action outputs; otherwise do integration tests against a test environment and assert on outputs/run status via REST or Log Analytics.
**Explanation**: Keep logic in Functions to unit test in C#; keep workflows thin. Use contract tests for connectors and synthetic messages in non-prod.
**Code / JSON / Bicep**
```csharp
// Integration test: trigger via HTTP and assert run result
var resp = await http.PostAsJsonAsync(callbackUrl, new { orderId = "t-1" });
Assert.Equal(HttpStatusCode.OK, resp.StatusCode);
```

**Interview traps**
- Exporting from portal and shipping hardcoded IDs.
- Assuming connections survive redeploys.
- Believing workflows can't be tested.

**Cheat sheet**
- Consumption = Bicep; Standard = zip deploy.
- Parameters + app settings per env.
- UAMI + IaC RBAC.
- Keep logic in Functions.
- Verify the unit-test framework status.

## Topic 16: Security, cost control, governance

**Study notes**

```
Inbound  : Request trigger SAS key (sig), IP restrictions (allowed caller IPs), Entra OAuth policies, APIM front door, Standard: private endpoint
Outbound : MI/RBAC, secure inputs/outputs, VNet integration (Standard), firewall/NSG, managed connector IP ranges
Data     : secureData, Key Vault, secure parameters, encryption at rest (platform-managed)
Cost     : Consumption action count, polling frequency, loops; Standard plan size
Governance: Azure Policy, tags, RBAC roles (Logic App Contributor/Operator), naming, diagnostics required
```

### 16.1 [Easy] How do you secure a Request trigger?
**Answer**: Restrict allowed caller IPs, require OAuth (Entra) authorization policies where supported, front it with API Management, and treat the SAS URL as a secret. Rotate by regenerating access keys.
**Explanation**: Default is SAS only, which is bearer-style. Standard supports private endpoints for inbound. Don't log the callback URL.
**Code / JSON / Bicep**
```bicep
accessControl: {
  triggers: { allowedCallerIpAddresses: [ { addressRange: '10.0.0.0/24' } ] }
  contents: { allowedCallerIpAddresses: [ { addressRange: '10.0.0.0/24' } ] }
}
```
(placed in `properties` of the Consumption workflow resource)

### 16.2 [Follow-up] Private networking for Standard?
**Answer**: Use inbound private endpoints for the workflow endpoints and regional VNet integration for outbound to private resources; built-in connectors then reach private endpoints directly.
**Explanation**: Managed connectors still egress via shared Azure connector infrastructure (documented IP ranges), so firewalled targets need those ranges or a built-in alternative. Storage account for Standard also needs network design (private endpoints for blob/queue/table/file).
**Code / JSON / Bicep**
```text
Caller -> Private Endpoint (workflow inbound) -> Logic App Standard -> VNet integration -> Private Endpoint (SQL/Service Bus)
```

### 16.3 [Medium] How do you control cost?
**Answer**: Use trigger conditions and event-based triggers rather than short polling, replace foreach with Select/Filter, avoid unnecessary actions, cap concurrency to avoid retry storms, and choose Standard when volume is high. Monitor via Cost Management and metrics.
**Explanation**: In Consumption every action, including skipped/failed retries per attempt policies (verify what counts), is billable; polling triggers with no data are billed. Standard cost is plan-based; right-size plan and use autoscale.
**Code / JSON / Bicep**
```text
Polling every 1 min = 43,200 trigger executions/month per workflow even if idle.
```

### 16.4 [Medium] Governance practices?
**Answer**: Enforce naming/tagging and required diagnostic settings via Azure Policy, least-privilege RBAC, MI over user connections, IaC-only deployments, a review checklist (secure data, retry, error scope, idempotency), and ownership tags.
**Explanation**: Use Policy "deployIfNotExists" for diagnostics. Restrict who can authorize API connections. Document custom connectors and shared UAMIs.
**Code / JSON / Bicep**
```text
Checklist: MI auth, secureData, retry policy set, catch scope, diagnostics on, alerts, tags, no hardcoded IDs.
```

### 16.5 [Hard] Where can secrets leak in a workflow?
**Answer**: Run history inputs/outputs, trigger URLs in logs, plaintext parameters in definitions/source control, HTTP headers with keys, and diagnostic logs. Mitigate with secureData, MI, Key Vault, securestring parameters.
**Explanation**: Run-history access can be restricted via `accessControl.contents` IP ranges and RBAC; users with Reader on the resource may still see metadata. Secure outputs on the Key Vault GET action so the secret isn't logged.
**Code / JSON / Bicep**
```json
"runtimeConfiguration": { "secureData": { "properties": ["inputs","outputs"] } }
```
**Gotchas / follow-ups**
- Secure outputs masks downstream usage display too, but expression-level `@body()` still resolves at runtime.
- Tracked properties are NOT masked - don't put secrets there.

### 16.6 [Tricky] IP restriction set but still blocked/open?
**Answer**: Restrictions apply to calls to trigger/content endpoints, not to outbound calls; calls from Azure services (Event Grid, other Logic Apps) come from specific IPs; Logic App-to-Logic App calls need the caller's outbound IPs allowed (or use the `Microsoft.Logic` workflow action path).
**Explanation**: Misconfiguring blocks legitimate callers with 403 `AuthorizationFailed`. Verify outbound IPs from the Logic App's Properties. Empty list vs "Only other Logic Apps" setting matters (verify portal options).
**Code / JSON / Bicep**
```text
403 on trigger URL -> check caller outbound IP vs allowedCallerIpAddresses
```

**Interview traps**
- Believing SAS URL is enough.
- Managed connectors using your VNet IP.
- Secrets in tracked properties.

**Cheat sheet**
- IP restrict + OAuth + APIM + PE.
- Managed connector egress = shared IPs.
- Cost = actions/polls (Consumption).
- Policy for diagnostics/tags.
- secureData for secrets.

---

## Topic 17: Scenario comparison: which tool?

**Study notes**

| # | Scenario | Best fit | Why | Watch out |
|---|----------|----------|-----|-----------|
| 1 | Service Bus order message -> call SAP/Salesforce + send Teams notification, with retries and run history | **Logic Apps (Standard)** | Connectors, retry, visibility, low code | Idempotency; concurrency vs target throttling |
| 2 | Parse a 200 MB CSV, validate rows, upsert 1M records to SQL nightly | **Data Factory / Fabric pipeline** (or Function + bulk copy) | Bulk throughput, mapping, cost | Logic Apps per-action cost + limits would fail |
| 3 | Multi-step saga with compensation, fan-out 5,000 parallel tasks, aggregate results, unit-tested in C# | **Durable Functions** | Code-first, deterministic orchestration, fan-out/in | Determinism rules, storage cost |
| 4 | Manager approves expense in Teams/Outlook; syncs to SharePoint list | **Power Automate** (or Logic Apps approvals if IT-owned) | Maker-friendly, M365 native | Licensing, ALM/solutions, governance |
| 5 | Partner sends X12 850 via AS2; ack 997; transform to JSON; post to ERP | **Logic Apps (B2B)** | AS2/X12 connectors, agreements, tracking | Integration account tiers, cert expiry |

### 17.1 [Easy] Event-driven blob -> email notification: tool?
**Answer**: Event Grid + Logic Apps (or Event Grid + Function if custom logic). Low-code and connectors make Logic Apps the quick choice.
**Explanation**: A Function is overkill unless processing is heavy; a Logic App with Office 365/SendGrid connector is minimal.
**Code / JSON / Bicep**
```text
Blob -> Event Grid -> Logic App -> send email
```

### 17.2 [Follow-up] High-volume telemetry (10k msgs/sec) transformation?
**Answer**: Not Logic Apps - use Event Hubs + Functions or Stream Analytics. Per-action overhead and limits make Logic Apps unsuitable.
**Explanation**: Logic Apps shines in message-level integration with moderate volume; streaming needs purpose-built services.
**Code / JSON / Bicep**
```text
Event Hubs -> Functions (batched) -> storage / Cosmos
```

### 17.3 [Medium] Hybrid design: Logic App orchestrates, Function computes?
**Answer**: Use Logic Apps for triggers, connectors, retries and notifications; call a Function for algorithmic or heavy transform steps. Best of both: visibility plus testable code.
**Explanation**: Keep contract small (JSON in/out), authenticate with MI, return clear status codes for retry decisions.
**Code / JSON / Bicep**
```json
"Enrich": { "type": "Http", "inputs": { "method": "POST", "uri": "https://func/api/enrich", "body": "@triggerBody()", "authentication": { "type": "ManagedServiceIdentity", "audience": "api://func" } }, "runAfter": {} }
```

### 17.4 [Hard] Long-running human approval over 7 days with reminders?
**Answer**: Logic Apps (stateful, webhook/approval action, delay until) or Power Automate approvals; Durable Functions with external events also fits if code-first. Choose by ownership/team skills.
**Explanation**: Logic Apps keeps state without compute; Durable keeps full control. Verify duration limits for the plan.
**Code / JSON / Bicep**
```text
Send approval -> Wait for response (timeout P7D) -> Condition -> proceed/escalate
```

### 17.5 [Tricky] Team wants to "just use Logic Apps for everything" - how do you push back?
**Answer**: Frame by workload characteristics: connector-rich, low-volume, visible workflows -> Logic Apps; compute-heavy, high-throughput, or heavily tested logic -> Functions/Durable; bulk data -> ADF. Cost and testability data usually persuades.
**Explanation**: Use a decision matrix and a pilot cost estimate (actions x volume). Mixed architectures are normal.
**Code / JSON / Bicep**
```text
Decision inputs: volume, latency, payload size, connectors needed, team skill, testability, cost model.
```

**Interview traps**
- Choosing one tool for all scenarios.
- Ignoring cost model.
- Using Logic Apps for bulk ETL.

**Cheat sheet**
- Glue/B2B -> Logic Apps.
- Bulk data -> ADF.
- Code orchestration -> Durable.
- Citizen flows -> Power Automate.
- Streaming -> Event Hubs/Functions/ASA.

---

## Topic 18: Production scenarios / troubleshooting

**Study notes**

Triage order: (1) Is the trigger firing (trigger history)? (2) Is the run failing (which action/error)? (3) Is it auth/network/throttling? (4) Did a deployment or identity change happen? (5) Is the upstream/downstream healthy?

### 18.1 [Easy] Trigger not firing - checklist?
**Answer**: Confirm workflow enabled, check trigger history (Skipped = nothing found, Failed = error), validate connection/auth, trigger conditions, polling cursor, source events (Event Grid delivery), and network/firewall.
**Explanation**: A disabled Logic App, expired connection, or recreated callback URL are the frequent causes. For Service Bus, check queue active count and whether another consumer is taking messages.
**Code / JSON / Bicep**
```text
Trigger history: Succeeded(fired) / Skipped(no data or condition false) / Failed(error)
```

### 18.2 [Follow-up] Duplicate runs - causes and fixes?
**Answer**: At-least-once delivery, Service Bus lock expiry, trigger retries, polling cursor reset, resubmits, or multiple workflows on the same source. Fix with idempotency keys, correct lock duration, single consumer design.
**Explanation**: Confirm via same MessageId in multiple runs. Use dedupe store, Service Bus duplicate detection for sender duplicates, and conditional writes.
**Code / JSON / Bicep**
```kusto
AzureDiagnostics | where Category == "WorkflowRuntime" | summarize runs = dcount(resource_runId_s) by tostring(trackedProperties_orderId_s) | where runs > 1
```

### 18.3 [Medium] 429 throttling - what to do?
**Answer**: Identify which hop returns 429 (connector, API, Service Bus), lower concurrency/repetitions, rely on Retry-After with exponential retry, batch calls, and request quota increases or use built-in connectors/ more connections.
**Explanation**: Parallel foreach (up to 50) and trigger concurrency multiply load. Design backpressure: queue between systems with controlled concurrency.
**Code / JSON / Bicep**
```json
"runtimeConfiguration": { "concurrency": { "repetitions": 5 } },
"retryPolicy": { "type": "exponential", "count": 6, "interval": "PT10S", "maximumInterval": "PT10M" }
```

### 18.4 [Medium] Connection auth expired?
**Answer**: Re-authorize the API connection and then move to MI/service principal auth to remove user-token dependence; alert on Connection errors and on auth failures in Log Analytics.
**Explanation**: Look at Portal > API Connections > Status. Credential-bound connections break on password change or departed users; Conditional Access can also block non-interactive token refresh.
**Code / JSON / Bicep**
```kusto
AzureDiagnostics | where Category == "WorkflowRuntime" and (code_s in ("Unauthorized","InvalidAuthenticationToken","Forbidden")) | summarize count() by resource_workflowName_s
```

### 18.5 [Hard] Large payload failure (413 / content too large / memory)?
**Answer**: Payload exceeds action/message limit. Use claim-check with Blob, chunking-enabled connector actions, stream via Function, or split input. Verify limits for your plan in docs.
**Explanation**: Errors like `RequestEntityTooLarge` or `InvalidTemplate` for huge arrays. Also run-history storage for huge outputs; mark secure/skip content where possible.
**Code / JSON / Bicep**
```text
Before: 120 MB JSON in workflow -> fails. After: store in Blob, pass URL, process in Function/ADF.
```
**Gotchas / follow-ups**
- Chunking is per-action opt-in and not universal.
- Response bodies from HTTP also have a size limit.

### 18.6 [Hard] Run stuck in Running/Waiting?
**Answer**: Waiting = queued due to concurrency/maximumWaitingRuns; Running = a long action (webhook, Until, Delay) or a hung downstream call without timeout. Add `limit.timeout`, cancel stuck runs, and fix upstream callbacks.
**Explanation**: Review concurrency settings and queue size, check downstream health and long Delay actions. Bulk cancel with REST/PowerShell if necessary.
**Code / JSON / Bicep**
```text
az rest --method post --url ".../workflows/{name}/runs/{runName}/cancel?api-version=2016-06-01"
```

### 18.7 [Tricky] After deployment all runs fail with 401/403 - why?
**Answer**: Connections unauthorized after ARM deploy, SAMI recreated losing RBAC, wrong app settings/parameters for the environment, or private networking changes. Check connection status, role assignments, identity principalId, and app settings first.
**Explanation**: Compare previous and current deployments; confirm UAMI is attached and role assignments exist; allow RBAC propagation time; for Standard confirm zip deploy didn't overwrite `connections.json` with dev values.
**Code / JSON / Bicep**
```powershell
az role assignment list --assignee <principalId> --all -o table
az resource show --ids <connectionId> --query properties.statuses
```
**Gotchas / follow-ups**
- Add a post-deployment smoke test workflow call.
- Use deployment slots (Standard) or blue/green apps where possible.

**Interview traps**
- Jumping to code changes before checking identity/connection/network.
- Retrying blindly on 429 with high concurrency.
- Ignoring trigger history.

**Cheat sheet**
- Trigger history first, then run, then action error.
- 401/403 = identity/connection/RBAC/network.
- 429 = reduce concurrency + backoff.
- Big payload = claim-check.
- Post-deploy smoke test.

