# Z12 - DevOps, IaC & Deployment (Bicep, Pipelines, CD, Troubleshooting)

**How to use:** Read each topic's study notes first, then work the ladder top to bottom (Easy -> Tricky); cover the Answer and say it aloud before reading the Explanation.
**Versions:** Task/action versions and API versions change often - items marked "verify in current docs" must be checked before you quote them.

## Table of Contents
1. IaC concepts & Bicep fundamentals
2. Bicep for the stack: identity, Key Vault, Service Bus, Storage
3. Bicep for the stack: Functions, Logic Apps, Redis, monitoring, private endpoints
4. Azure DevOps pipelines vs GitHub Actions
5. CI for .NET
6. CD strategies (slots, blue-green, canary, flags, DB migrations)
7. Deploying Azure Functions
8. Deploying Logic Apps (Standard & Consumption)
9. Secrets in pipelines & least-privilege deployment identities
10. Environment promotion & per-environment config
11. GitOps, branching, PR validation, policy as code, drift
12. ACR, Docker for .NET, Container Apps/AKS overview
13. Testing in pipelines
14. Release health & rollback
15. Cost governance & tagging in IaC
16. Pipeline troubleshooting scenarios

---

## Topic 1: IaC concepts & Bicep fundamentals

**Study notes**

IaC = infrastructure described in version-controlled files, deployed repeatably and idempotently. Bicep is a DSL that transpiles to ARM JSON; ARM (the Azure Resource Manager service) is the actual deployment engine. Bicep has **no state file** - ARM itself is the source of truth and a deployment compares the template to live resources. Terraform keeps a **state file** (remote backend, locking) and supports many clouds.

```
main.bicep + main.bicepparam
      |  (az bicep build / implicit)
      v
  ARM JSON  --->  ARM (what-if / validate / deploy)  --->  Resources
      ^                                   |
   modules (.bicep / registry br:)        +--> deployment record (history, outputs)
```

Building blocks: `param`, `var`, `resource`, `module`, `output`, `existing` resources, `for` loops, `if` conditions, `targetScope` (resourceGroup default, subscription, managementGroup, tenant), decorators (`@secure()`, `@allowed`, `@minLength`, `@description`).

Deployment modes: **Incremental** (default) - leaves resources not in the template alone. **Complete** - deletes resources in the RG not in the template (dangerous; not all types are deleted - verify in current docs). Deployment stacks are the newer way to manage lifecycle/deny settings (verify in current docs).

### 1.1 [Easy] What is Bicep and how does it relate to ARM templates?
**Answer**: Bicep is a declarative, more readable language that compiles to ARM JSON. It has first-class tooling, type safety and modules, but deploys through the same ARM engine and supports all resource types on day one.
**Explanation**: Because it compiles to ARM, there is no extra runtime and no separate state. You can decompile ARM to Bicep (`az bicep decompile`) as a starting point, but output usually needs cleanup.
**Code / YAML / Bicep / CLI**
```bicep
@description('Environment short name')
@allowed(['dev','test','prod'])
param env string

param location string = resourceGroup().location
var stName = 'st${uniqueString(resourceGroup().id)}${env}'

resource st 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: take(toLower(stName), 24)
  location: location
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
  properties: { minimumTlsVersion: 'TLS1_2', allowBlobPublicAccess: false }
}
output stId string = st.id
```
```bash
az bicep build --file main.bicep
az bicep lint --file main.bicep
```

### 1.2 [Follow-up] How do modules, outputs, loops and conditions work?
**Answer**: A `module` references another Bicep file (local path or registry `br:`) and gets its own deployment; inputs are params and results are outputs. `for` creates arrays of resources/modules and `if` conditionally deploys one.
**Explanation**: Each module becomes a nested deployment, so outputs of secrets should be avoided (they are stored in deployment history). Use `@batchSize` to serialize loops. Conditions on resources make referencing them tricky - use the safe-access operator `?.`.
**Code / YAML / Bicep / CLI**
```bicep
param queues array = ['orders','payments']
param deployDlqAlert bool = false

module sb 'modules/servicebus.bicep' = {
  name: 'sb-${uniqueString(resourceGroup().id)}'
  params: { queueNames: queues }
}

resource q 'Microsoft.ServiceBus/namespaces/queues@2022-10-01-preview' = [for n in queues: {
  name: 'sbns/${n}'
  properties: { maxDeliveryCount: 10 }
}]

resource alert 'Microsoft.Insights/metricAlerts@2018-03-01' = if (deployDlqAlert) {
  name: 'dlq-alert'
  location: 'global'
  properties: { /* ... */ }
}
```
(Illustrative; in real code give the namespace as `parent:` instead of the slash name.)

### 1.3 [Medium] How do you reference existing resources and deploy across scopes?
**Answer**: Use the `existing` keyword with a `scope:` for resources in another RG/subscription, and `targetScope` or module `scope:` to deploy at subscription level. This lets you assign roles on a resource owned by another team without redeploying it.
**Explanation**: `existing` does not create/modify anything; it only gives a typed symbol to read properties or be a `parent`/`scope`. Cross-RG: `scope: resourceGroup(subId, rgName)`.
**Code / YAML / Bicep / CLI**
```bicep
targetScope = 'subscription'
resource rg 'Microsoft.Resources/resourceGroups@2024-03-01' = {
  name: 'rg-app-prod'
  location: 'westeurope'
}
module app 'app.bicep' = { name: 'app', scope: rg, params: {} }
```
```bicep
resource kv 'Microsoft.KeyVault/vaults@2023-07-01' existing = {
  name: 'kv-shared'
  scope: resourceGroup('sub-guid', 'rg-shared')
}
```
```bash
az deployment sub create -l westeurope -f main.bicep -p main.bicepparam
```

### 1.4 [Medium] What does `what-if` do, and how do you use it in a pipeline?
**Answer**: What-if previews Create/Modify/Delete/NoChange/Ignore for a deployment without applying it. It is used as a PR gate or pre-approval step so reviewers see the impact.
**Explanation**: What-if is best-effort: it can show noise (read-only properties, API-version defaults) and cannot always evaluate things like `reference()` of not-yet-created resources. Treat it as a review aid, not a guarantee. `az deployment group validate` checks template validity only.
**Code / YAML / Bicep / CLI**
```bash
az deployment group what-if -g rg-app-dev -f main.bicep -p main.dev.bicepparam
az deployment group create -g rg-app-dev -f main.bicep -p main.dev.bicepparam --mode Incremental
```
```bicep
using 'main.bicep'
param env = 'dev'
param queues = ['orders']
```

### 1.5 [Hard] Incremental vs Complete mode, and Bicep vs Terraform state - what are the production consequences?
**Answer**: Incremental only adds/updates; Complete deletes anything in the target RG not in the template, so a partial template can wipe resources. Terraform tracks resources in a state file (and can detect drift against it); Bicep relies on ARM and has no state to lose or lock, but also no native plan/destroy of removed resources.
**Explanation**: In Bicep, removing a resource from the file does nothing in Incremental mode - orphaned resources persist (use Complete carefully, or deployment stacks with `actionOnUnmanage`, verify in current docs). Terraform's state needs secure remote storage, locking, and can contain secrets; import/moved blocks manage refactors. Bicep refactors (rename) cause delete+create unless names stay the same.
**Code / YAML / Bicep / CLI**
```bash
# Always what-if before Complete
az deployment group what-if -g rg-app-dev -f main.bicep --mode Complete
```
**Gotchas / follow-ups**
- Complete mode on a shared RG is a classic outage.
- Renaming a symbolic name is safe; changing the `name:` property is not.
- Deployment history limit is 800 per RG - old ones are auto-deleted but can cause errors if disabled (verify in current docs).
- Linter rules are configured in `bicepconfig.json`.

**Interview traps**
- "Bicep has state" - no; ARM is the source of truth.
- Putting secrets in outputs - visible in deployment history.
- Using Complete mode casually.
- Treating what-if as exact truth.

**Cheat sheet**
- Bicep -> ARM JSON -> ARM; no state file.
- `existing` + `scope` for cross-RG references.
- `.bicepparam` with `using`.
- `az deployment {group|sub|mg|tenant} {create|what-if|validate}`.
- Lint via `az bicep lint` / `bicepconfig.json`.

---

## Topic 2: Bicep for the stack - identity, Key Vault, Service Bus, Storage

**Study notes**

Typical order: UAMI -> Key Vault (RBAC mode) -> Service Bus / Storage -> role assignments -> compute that uses the identity.

```
UAMI (created first, principalId known at deploy time)
  |--> role: Key Vault Secrets User   @ Key Vault
  |--> role: Service Bus Data Sender  @ namespace/queue
  |--> role: Storage Blob Data Contrib@ storage account
Function App identity: UserAssigned [UAMI]  (no secrets in app settings)
```

Role assignment names must be a GUID that is **deterministic** and unique per (scope, principal, role): `guid(scope.id, principalId, roleDefinitionId)`. Set `principalType: 'ServicePrincipal'` to avoid replication-delay failures ("PrincipalNotFound") when the identity was just created. Role definition IDs are GUIDs under `/providers/Microsoft.Authorization/roleDefinitions/` - verify IDs in current docs.

### 2.1 [Easy] How do you create a UAMI and grant it a role in Bicep?
**Answer**: Declare `Microsoft.ManagedIdentity/userAssignedIdentities`, then a `Microsoft.Authorization/roleAssignments` scoped to the target resource, named with `guid()`. Use `principalType: 'ServicePrincipal'`.
**Explanation**: The deploying identity needs permission to write role assignments (User Access Administrator, Owner, or RBAC Administrator with conditions) at that scope. A SAMI is created with the resource (`identity: { type: 'SystemAssigned' }`) and dies with it; a UAMI is a separate lifecycle resource, shareable and pre-authorizable.
**Code / YAML / Bicep / CLI**
```bicep
resource uami 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = {
  name: 'id-app-${env}'
  location: location
}

var kvSecretsUser = '4633458b-17de-408a-b874-0f2f5f2b2e9b'

resource kv 'Microsoft.KeyVault/vaults@2023-07-01' existing = { name: kvName }

resource ra 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  scope: kv
  name: guid(kv.id, uami.id, kvSecretsUser)
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', kvSecretsUser)
    principalId: uami.properties.principalId
    principalType: 'ServicePrincipal'
  }
}
```

### 2.2 [Follow-up] Key Vault with RBAC - what properties matter?
**Answer**: Set `enableRbacAuthorization: true`, enable soft delete (default) and purge protection, and restrict network access. Secrets are read via the "Key Vault Secrets User" role rather than access policies.
**Explanation**: Purge protection cannot be disabled once on. Re-deploying a vault name that is soft-deleted fails unless recovered (`createMode: 'recover'`) or purged. Mixing access policies and RBAC is confusing; pick RBAC.
**Code / YAML / Bicep / CLI**
```bicep
resource kv 'Microsoft.KeyVault/vaults@2023-07-01' = {
  name: kvName
  location: location
  properties: {
    tenantId: subscription().tenantId
    sku: { family: 'A', name: 'standard' }
    enableRbacAuthorization: true
    enablePurgeProtection: true
    softDeleteRetentionInDays: 90
    publicNetworkAccess: 'Disabled'
  }
}
```

### 2.3 [Medium] Service Bus namespace, queues, topics and subscriptions in Bicep
**Answer**: Child resources use `parent:`; queues, topics, and subscriptions are separate resources. Use Standard/Premium for topics; Premium for private endpoints and zone redundancy.
**Explanation**: Disable local (SAS) auth with `disableLocalAuth: true` to force Entra auth. Set `maxDeliveryCount`, `lockDuration`, `deadLetteringOnMessageExpiration` explicitly. Changing certain properties (e.g. enablePartitioning) forces recreation - verify in current docs.
**Code / YAML / Bicep / CLI**
```bicep
resource ns 'Microsoft.ServiceBus/namespaces@2022-10-01-preview' = {
  name: 'sbns-${env}'
  location: location
  sku: { name: 'Standard', tier: 'Standard' }
  properties: { disableLocalAuth: true, minimumTlsVersion: '1.2' }
}
resource topic 'Microsoft.ServiceBus/namespaces/topics@2022-10-01-preview' = {
  parent: ns
  name: 'orders'
  properties: { defaultMessageTimeToLive: 'P14D' }
}
resource sub 'Microsoft.ServiceBus/namespaces/topics/subscriptions@2022-10-01-preview' = {
  parent: topic
  name: 'billing'
  properties: {
    maxDeliveryCount: 10
    lockDuration: 'PT1M'
    deadLetteringOnMessageExpiration: true
  }
}
resource q 'Microsoft.ServiceBus/namespaces/queues@2022-10-01-preview' = {
  parent: ns
  name: 'payments'
  properties: { requiresSession: false, maxDeliveryCount: 5 }
}
```

### 2.4 [Medium] Storage account and containers
**Answer**: Create the account, then a `blobServices/default` child and `containers` under it. Harden with TLS 1.2, no public blob access, and optionally disable shared keys.
**Explanation**: Functions host storage historically needs a connection string/shared key; with `allowSharedKeyAccess: false` you must use identity-based host storage connections (`AzureWebJobsStorage__accountName`, verify in current docs for your hosting plan).
**Code / YAML / Bicep / CLI**
```bicep
resource blob 'Microsoft.Storage/storageAccounts/blobServices@2023-05-01' = {
  parent: st
  name: 'default'
}
resource c 'Microsoft.Storage/storageAccounts/blobServices/containers@2023-05-01' = {
  parent: blob
  name: 'uploads'
  properties: { publicAccess: 'None' }
}
```

### 2.5 [Hard] Why do role assignments fail intermittently, and how do you make them idempotent?
**Answer**: Failures come from non-deterministic names (RoleAssignmentExists with a different GUID), replication lag for new principals (fix with `principalType`), or the deployer lacking write permission. A `guid(scope, principal, role)` name makes re-runs idempotent.
**Explanation**: Using `newGuid()` or a random name creates duplicates and conflicts. If you change the GUID seed formula later, ARM tries to create a second assignment for the same principal/role/scope and fails with `RoleAssignmentExists`; delete the old one first. Use values known at start of deployment in `guid()` where possible - using `uami.properties.principalId` in the name causes "must be calculable at start" errors; use `uami.id` instead.
**Code / YAML / Bicep / CLI**
```bicep
// GOOD: seed from resource ids (known at compile time)
name: guid(kv.id, uami.id, kvSecretsUser)
// BAD: seed from runtime-only value
// name: guid(kv.id, uami.properties.principalId, kvSecretsUser)
```
**Gotchas / follow-ups**
- Assigning roles to a user/group object ID: use `principalType: 'User'`/`'Group'`.
- Deployer needs Microsoft.Authorization/roleAssignments/write.
- Propagation can take minutes; app may 403 right after deploy - retry, don't redeploy.

**Interview traps**
- Random GUID for role assignment names.
- Forgetting `principalType` -> PrincipalNotFound.
- Soft-deleted Key Vault name collision.
- Secrets passed as module outputs.

**Cheat sheet**
- `guid(scope.id, principal.id, roleId)`.
- KV: RBAC + purge protection.
- SB: `disableLocalAuth: true`, children use `parent:`.
- UAMI = separate lifecycle; SAMI = tied to resource.

---

## Topic 3: Bicep for the stack - Functions, Logic Apps, Redis, monitoring, private endpoints

**Study notes**

```
Log Analytics <-- App Insights (workspace-based)
       ^                ^
       |                | APPLICATIONINSIGHTS_CONNECTION_STRING
Diagnostic settings   Function App --(UAMI)--> KV (@Microsoft.KeyVault refs)
 (all PaaS)               |
                     App Service Plan (Y1/EP1/Flex)
Private endpoint (NIC in subnet) + privateDnsZoneGroup -> Private DNS zone
```

App Insights should be workspace-based (`WorkspaceResourceId`). Key Vault references in app settings use `@Microsoft.KeyVault(SecretUri=...)` or `VaultName=;SecretName=`; for a UAMI you must set `keyVaultReferenceIdentity` on the site.

### 3.1 [Easy] How do you define Log Analytics + App Insights + diagnostic settings?
**Answer**: Create the workspace, a workspace-based App Insights component, and a `Microsoft.Insights/diagnosticSettings` extension resource per PaaS resource sending logs/metrics to the workspace.
**Explanation**: Diagnostic settings are extension resources: set `scope:` to the target. Use category groups (`allLogs`) to avoid listing categories.
**Code / YAML / Bicep / CLI**
```bicep
resource law 'Microsoft.OperationalInsights/workspaces@2023-09-01' = {
  name: 'law-${env}'
  location: location
  properties: { sku: { name: 'PerGB2018' }, retentionInDays: 30 }
}
resource ai 'Microsoft.Insights/components@2020-02-02' = {
  name: 'appi-${env}'
  location: location
  kind: 'web'
  properties: { Application_Type: 'web', WorkspaceResourceId: law.id }
}
resource diag 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = {
  name: 'to-law'
  scope: ns   // Service Bus namespace
  properties: {
    workspaceId: law.id
    logs: [ { categoryGroup: 'allLogs', enabled: true } ]
    metrics: [ { category: 'AllMetrics', enabled: true } ]
  }
}
```

### 3.2 [Follow-up] Function app with UAMI and Key Vault references
**Answer**: Create a plan and a `Microsoft.Web/sites` with kind `functionapp`, `identity` of type UserAssigned, `keyVaultReferenceIdentity` set to the UAMI id, and app settings using Key Vault references. Grant the UAMI "Key Vault Secrets User" first.
**Explanation**: Without `keyVaultReferenceIdentity`, references resolve with the system identity (and fail if none). Reference status appears in the portal ("Key Vault Reference" green check). Versionless URIs pick up rotated secrets (refresh may lag up to ~24h - verify in current docs).
**Code / YAML / Bicep / CLI**
```bicep
resource plan 'Microsoft.Web/serverfarms@2023-12-01' = {
  name: 'asp-${env}'
  location: location
  sku: { name: 'EP1', tier: 'ElasticPremium' }
  kind: 'elastic'
}
resource func 'Microsoft.Web/sites@2023-12-01' = {
  name: 'func-orders-${env}'
  location: location
  kind: 'functionapp'
  identity: {
    type: 'UserAssigned'
    userAssignedIdentities: { '${uami.id}': {} }
  }
  properties: {
    serverFarmId: plan.id
    httpsOnly: true
    keyVaultReferenceIdentity: uami.id
    siteConfig: {
      appSettings: [
        { name: 'FUNCTIONS_EXTENSION_VERSION', value: '~4' }
        { name: 'FUNCTIONS_WORKER_RUNTIME', value: 'dotnet-isolated' }
        { name: 'APPLICATIONINSIGHTS_CONNECTION_STRING', value: ai.properties.ConnectionString }
        { name: 'AzureWebJobsStorage__accountName', value: st.name }
        { name: 'ServiceBusConnection__fullyQualifiedNamespace', value: '${ns.name}.servicebus.windows.net' }
        { name: 'ServiceBusConnection__credential', value: 'managedidentity' }
        { name: 'ServiceBusConnection__clientId', value: uami.properties.clientId }
        { name: 'ExternalApiKey', value: '@Microsoft.KeyVault(VaultName=${kvName};SecretName=external-api-key)' }
      ]
    }
  }
}
```

### 3.3 [Medium] Logic App Standard and Redis in Bicep
**Answer**: Logic App Standard is a `Microsoft.Web/sites` with `kind: 'functionapp,workflowapp'` on a Workflow Standard plan (WS1-3); it also needs a storage account. Redis uses `Microsoft.Cache/redis` (or Azure Managed Redis - verify current resource type/docs).
**Explanation**: Workflows themselves are deployed as files (zip), not by Bicep. For Redis, disable the non-SSL port, set TLS 1.2, and prefer Entra auth where supported (verify in current docs for your SKU).
**Code / YAML / Bicep / CLI**
```bicep
resource wfPlan 'Microsoft.Web/serverfarms@2023-12-01' = {
  name: 'asp-wf-${env}'
  location: location
  sku: { name: 'WS1', tier: 'WorkflowStandard' }
}
resource la 'Microsoft.Web/sites@2023-12-01' = {
  name: 'la-${env}'
  location: location
  kind: 'functionapp,workflowapp'
  identity: { type: 'SystemAssigned' }
  properties: { serverFarmId: wfPlan.id, httpsOnly: true }
}
resource redis 'Microsoft.Cache/redis@2023-08-01' = {
  name: 'redis-${env}'
  location: location
  properties: {
    sku: { name: 'Standard', family: 'C', capacity: 1 }
    enableNonSslPort: false
    minimumTlsVersion: '1.2'
  }
}
```

### 3.4 [Medium] Private endpoints - what are the moving parts?
**Answer**: A private endpoint NIC in your subnet, a `privateLinkServiceConnection` with the right group ID, and a private DNS zone (e.g. `privatelink.vaultcore.azure.net`) linked to the VNet via a `privateDnsZoneGroup`. DNS is where it usually breaks.
**Explanation**: Group IDs: `vault`, `blob`, `namespace` (Service Bus), `sites` (Web). Public access should then be disabled. Function apps need VNet integration for outbound as well as a PE for inbound. Pipelines on public hosted agents can no longer reach a private-only data plane - use self-hosted/VNet agents.
**Code / YAML / Bicep / CLI**
```bicep
resource pe 'Microsoft.Network/privateEndpoints@2023-11-01' = {
  name: 'pe-kv'
  location: location
  properties: {
    subnet: { id: subnetId }
    privateLinkServiceConnections: [{
      name: 'kv'
      properties: { privateLinkServiceId: kv.id, groupIds: ['vault'] }
    }]
  }
}
resource dns 'Microsoft.Network/privateEndpoints/privateDnsZoneGroups@2023-11-01' = {
  parent: pe
  name: 'default'
  properties: {
    privateDnsZoneConfigs: [{ name: 'kv', properties: { privateDnsZoneId: kvZoneId } }]
  }
}
```

### 3.5 [Hard] Circular dependencies and ordering in this stack
**Answer**: Cycles arise e.g. when the Function needs the KV reference (needs role), the role needs the identity principal (from the function's SAMI), and KV network rules need the function's outbound IPs. Break them with a UAMI created first, or by splitting into modules/separate deployments.
**Explanation**: With SAMI, the role assignment depends on the site and the site's setting resolves the secret at runtime - ARM deploys fine but the reference fails until the role propagates; restart the app or use UAMI. Bicep infers dependencies from symbolic references; use `dependsOn` sparingly for implicit ordering (e.g. DNS before app).
**Code / YAML / Bicep / CLI**
```bicep
// UAMI first -> role -> function depends on role
resource func 'Microsoft.Web/sites@2023-12-01' = {
  // ...
  dependsOn: [ ra ]   // ensure role exists before the app resolves KV references
}
```
**Gotchas / follow-ups**
- Bicep reports BCP080/BCP-cycle at compile time; ARM-level cycles surface as "Circular dependency detected".
- Do not read `.properties` of a resource into its own name/dependencies.
- Outputs of secrets (`listKeys()`) in outputs leak into history.

**Interview traps**
- SAMI + KV reference at first deploy -> unresolved until propagation.
- Missing `keyVaultReferenceIdentity` for UAMI.
- Diagnostic settings need `scope`, not `parent`.
- Public agents cannot reach private-only resources.

**Cheat sheet**
- Workspace-based App Insights.
- `AzureWebJobsStorage__accountName` + MI for identity-based host storage.
- Logic App Std: `functionapp,workflowapp`, WS plan.
- PE = NIC + DNS zone group.

---

## Topic 4: Azure DevOps pipelines vs GitHub Actions

**Study notes**

```
ADO:  trigger -> stages -> jobs (agent pool) -> steps (task/script)
        |             \-> deployment job -> environment (approvals/checks)
GHA:  on: -> jobs (runs-on) -> steps (uses:/run:)
        \-> job.environment (protection rules, required reviewers)
Auth: service connection (ADO) / azure/login (GHA)  --OIDC federated cred--> Entra app/UAMI
```

| Aspect | Azure DevOps | GitHub Actions |
|---|---|---|
| Unit of reuse | YAML templates (`template:`), extends | Reusable workflows, composite actions |
| Approvals | Environments + checks | Environments + protection rules |
| Secrets | Variable groups (KV-linked), secret vars | Repo/env secrets, OIDC |
| Azure auth | Service connection (ARM) | `azure/login` |

Task/action versions change - verify in current docs: `AzureCLI@2`, `AzureFunctionApp@2`, `AzureWebApp@1`, `DotNetCoreCLI@2`, `AzureKeyVault@2`, `azure/login@v2`.

### 4.1 [Easy] Describe the structure of an Azure DevOps YAML pipeline.
**Answer**: A pipeline has triggers, variables, and `stages` containing `jobs` containing `steps` (tasks or scripts). Deployment jobs target an `environment` and use a deployment strategy (`runOnce`, `rolling`, `canary`).
**Explanation**: Stages run sequentially by default and can depend on each other; jobs in a stage run in parallel unless `dependsOn`. Templates (`- template: steps/build.yml`) give reuse; `extends` templates can enforce required steps.
**Code / YAML / Bicep / CLI**
```yaml
trigger:
  branches: { include: [main] }
pool: { vmImage: ubuntu-latest }
stages:
- stage: Build
  jobs:
  - job: build
    steps:
    - task: UseDotNet@2
      inputs: { packageType: sdk, version: '8.x' }
    - script: dotnet build -c Release
- stage: DeployDev
  dependsOn: Build
  jobs:
  - deployment: deploy
    environment: dev
    strategy:
      runOnce:
        deploy:
          steps:
          - script: echo deploying
```

### 4.2 [Follow-up] Equivalent in GitHub Actions with OIDC (no secrets)?
**Answer**: Use `permissions: id-token: write` and `azure/login` with client-id, tenant-id and subscription-id; a federated credential on the Entra app/UAMI trusts the repo/environment/branch subject. No client secret exists to leak or rotate.
**Explanation**: Federated credential subject must match exactly, e.g. `repo:org/repo:environment:prod` or `repo:org/repo:ref:refs/heads/main`. Mismatch gives AADSTS70021 "no matching federated identity record". The client-id etc. are not secrets but are often stored as secrets/variables.
**Code / YAML / Bicep / CLI**
```yaml
name: deploy
on: { push: { branches: [main] } }
permissions: { id-token: write, contents: read }
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: prod
    steps:
    - uses: actions/checkout@v4
    - uses: azure/login@v2
      with:
        client-id: ${{ vars.AZURE_CLIENT_ID }}
        tenant-id: ${{ vars.AZURE_TENANT_ID }}
        subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
    - run: az deployment group create -g rg-app-prod -f main.bicep -p main.prod.bicepparam
```
```bash
az identity federated-credential create --name gh-prod --identity-name id-deploy -g rg-ci \
  --issuer https://token.actions.githubusercontent.com \
  --subject repo:myorg/myrepo:environment:prod --audiences api://AzureADTokenExchange
```

### 4.3 [Medium] Service connections: secret vs workload identity federation?
**Answer**: ADO ARM service connections can use app registration with a secret/certificate, or workload identity federation (OIDC) which needs no stored secret. WIF is preferred; secrets expire and must be rotated.
**Explanation**: With WIF the connection exchanges the pipeline's OIDC token for an Entra token. It can be created automatically or manually (issuer/subject shown in the connection). Existing secret-based connections can be converted (verify in current docs). Some older tasks/extensions may lag in WIF support.

### 4.4 [Medium] When do you use `AzureCLI@2` vs `AzureFunctionApp@2` / `AzureWebApp@1`?
**Answer**: Use the dedicated deploy tasks for packages (`AzureFunctionApp@2`, `AzureWebApp@1`) and `AzureCLI@2` for Bicep/az scripting or anything without a task. `AzureCLI@2` logs in with the service connection for the script's duration.
**Explanation**: `addSpnToEnvironment: true` exposes service principal id and `idToken` to the script (useful for tools needing OIDC). Prefer `scriptType: bash` on Linux agents.
**Code / YAML / Bicep / CLI**
```yaml
- task: AzureCLI@2
  displayName: Deploy infra
  inputs:
    azureSubscription: sc-prod
    scriptType: bash
    scriptLocation: inlineScript
    inlineScript: |
      az deployment group create -g $(rg) -f infra/main.bicep -p infra/main.prod.bicepparam
- task: AzureFunctionApp@2
  inputs:
    azureSubscription: sc-prod
    appType: functionAppLinux
    appName: func-orders-prod
    package: $(Pipeline.Workspace)/drop/*.zip
```

### 4.5 [Hard] Variable groups linked to Key Vault - how do they work and what can go wrong?
**Answer**: A variable group can link to a Key Vault via a service connection and expose selected secrets as masked pipeline variables fetched at run start. The connection identity needs get/list secret permission on the vault.
**Explanation**: Secrets are masked in logs but masking is string-matching: transformed values (base64, substring) can leak. Values are fetched once at job start; rotation mid-run is not seen. If the vault has a firewall, hosted agents are blocked - use the `AzureKeyVault@2` task from a trusted network/agent instead. Secret variables are not auto-passed to scripts as env vars; map explicitly via `env:`.
**Code / YAML / Bicep / CLI**
```yaml
- task: AzureKeyVault@2
  inputs:
    azureSubscription: sc-prod
    KeyVaultName: kv-shared
    SecretsFilter: 'db-password,api-key'
    RunAsPreJob: false
- script: ./migrate.sh
  env:
    DB_PASSWORD: $(db-password)
```
**Gotchas / follow-ups**
- Fork PRs: secrets are not exposed by default - intentional.
- Prefer managed identity/OIDC at runtime over passing secrets at all.
- Secrets in `-p password=$(x)` args may appear in process lists/logs.

**Interview traps**
- "Secret variables can't leak" - derived values can.
- OIDC subject mismatch (branch vs environment).
- Using one all-powerful service connection for every stage.
- Quoting task versions from memory.

**Cheat sheet**
- ADO: stages > jobs > steps; deployment jobs + environments.
- GHA: `id-token: write` + `azure/login`.
- WIF/OIDC > client secrets.
- `AzureCLI@2` for Bicep, dedicated tasks for packages.

---

## Topic 5: CI for .NET

**Study notes**

```
checkout -> restore (cached) -> build -> test (+coverage) -> publish -> pack artifact -> upload
                                           |-> trx / cobertura reports
```

Build once, deploy the same artifact everywhere (immutable artifact). Use `--no-restore` / `--no-build` to avoid redoing work; lock files (`packages.lock.json`) make caching keys deterministic.

### 5.1 [Easy] What does a basic .NET CI pipeline do?
**Answer**: Restore, build in Release, run tests, publish the app, and upload the output as an artifact. Nothing environment-specific is baked in.
**Explanation**: Keep versioning in the build (e.g. `-p:Version=`), fail on warnings where practical, and store test results for the test tab.
**Code / YAML / Bicep / CLI**
```yaml
steps:
- task: UseDotNet@2
  inputs: { packageType: sdk, version: '8.x' }
- task: DotNetCoreCLI@2
  inputs: { command: restore, projects: '**/*.csproj' }
- task: DotNetCoreCLI@2
  inputs: { command: build, arguments: '-c Release --no-restore' }
- task: DotNetCoreCLI@2
  inputs:
    command: test
    arguments: '-c Release --no-build --collect:"XPlat Code Coverage"'
- task: DotNetCoreCLI@2
  inputs: { command: publish, publishWebProjects: false, projects: 'src/Func/Func.csproj', arguments: '-c Release -o $(Build.ArtifactStagingDirectory)', zipAfterPublish: true }
- publish: $(Build.ArtifactStagingDirectory)
  artifact: drop
```

### 5.2 [Follow-up] Code coverage and test reporting?
**Answer**: Use coverlet via `XPlat Code Coverage` (Cobertura output) and publish with `PublishCodeCoverageResults` (verify task version in current docs); test results are published automatically by `DotNetCoreCLI@2 test`. Gate on thresholds sparingly.
**Explanation**: Coverage percent is a weak quality signal; use it to find untested areas, not as a target to game.
**Code / YAML / Bicep / CLI**
```bash
dotnet test --collect:"XPlat Code Coverage" --results-directory ./TestResults --logger trx
```

### 5.3 [Medium] How do you cache NuGet packages?
**Answer**: Key the cache on a hash of lock files/`*.csproj`/`Directory.Packages.props` and cache `~/.nuget/packages`. ADO: `Cache@2`; GHA: `actions/setup-dotnet` `cache: true` or `actions/cache`.
**Explanation**: Without lock files, the key must include csproj files. Enable `RestorePackagesWithLockFile` for reproducibility.
**Code / YAML / Bicep / CLI**
```yaml
variables:
  NUGET_PACKAGES: $(Pipeline.Workspace)/.nuget/packages
steps:
- task: Cache@2
  inputs:
    key: 'nuget | "$(Agent.OS)" | **/packages.lock.json,!**/bin/**,!**/obj/**'
    restoreKeys: 'nuget | "$(Agent.OS)"'
    path: $(NUGET_PACKAGES)
```

### 5.4 [Hard] What makes CI builds non-reproducible or flaky?
**Answer**: Floating package versions, unpinned SDKs, tests relying on time/network/order, shared state, and rebuilding per environment. Pin the SDK with `global.json`, use lock files, and build once.
**Explanation**: Use `--locked-mode` in restore to fail on drift. Deterministic builds (`<Deterministic>`, `ContinuousIntegrationBuild=true`) matter for source-link/symbols. Quarantine flaky tests rather than rerunning blindly.
**Code / YAML / Bicep / CLI**
```bash
dotnet restore --locked-mode
dotnet build -c Release -p:ContinuousIntegrationBuild=true
```
**Gotchas / follow-ups**
- Functions isolated worker publish output must be zipped with the correct folder layout (host.json at root).
- Do not publish `appsettings.Development.json` secrets.

**Interview traps**
- Rebuilding for each environment.
- Cache key not including lock/csproj hash.
- Coverage as a goal.

**Cheat sheet**
- restore -> build -> test -> publish -> artifact.
- `global.json` + lock files.
- `--no-restore`/`--no-build`.
- Immutable artifact.

---

## Topic 6: CD strategies - slots, blue-green, canary, flags, DB migrations

**Study notes**

```
Slot swap:   deploy -> staging slot -> warm-up (applicationInitialization) -> swap -> prod
                                         ^ swap-with-preview optional; rollback = swap back
Canary:      traffic % (slot routing / Front Door / Container Apps revisions) -> observe -> promote
Blue-green:  two full environments, switch at router (Front Door/Traffic Manager/DNS)
Rolling:     replace instances in batches (AKS, VMSS)
```

Slots (Standard+ plans; Premium for Functions slots as well) share the plan's compute. **Slot-sticky** settings stay with the slot during a swap; non-sticky move with the code.

### 6.1 [Easy] What is a deployment slot and what happens on swap?
**Answer**: A slot is a live app instance with its own hostname. On swap, App Service applies the target's settings to the source slot, warms it, then switches routing so there is no cold start and no downtime; swapping back is the rollback.
**Explanation**: Non-sticky settings (most app settings, connection strings) swap with the app; slot-sticky settings (marked "deployment slot setting") do not. Swap can be done with preview (multi-phase).
**Code / YAML / Bicep / CLI**
```bash
az webapp deployment slot create -g rg -n app --slot staging
az webapp deployment slot swap -g rg -n app --slot staging --target-slot production
```
```bicep
resource slot 'Microsoft.Web/sites/slots@2023-12-01' = {
  parent: func
  name: 'staging'
  location: location
  properties: { serverFarmId: plan.id }
}
```

### 6.2 [Follow-up] How do you ensure warm-up before swap?
**Answer**: Configure `applicationInitialization` (web.config for Windows) or the `WEBSITE_SWAP_WARMUP_PING_PATH` / `WEBSITE_SWAP_WARMUP_PING_STATUSES` settings, and hit a health endpoint that touches dependencies. Swap fails if warm-up pings don't return an accepted status.
**Explanation**: Warm-up should exercise caches, DB connections and Service Bus clients. Don't make the health check so deep it fails the swap on an optional dependency.
**Code / YAML / Bicep / CLI**
```bash
az webapp config appsettings set -g rg -n app --slot staging --settings \
  WEBSITE_SWAP_WARMUP_PING_PATH=/health WEBSITE_SWAP_WARMUP_PING_STATUSES=200
```

### 6.3 [Medium] Canary and feature flags - how do they differ from slots?
**Answer**: Slots switch whole-instance code; canary sends a percentage of traffic to the new version (App Service "traffic %" routing, Front Door weights, Container Apps revisions). Feature flags decouple deploy from release so code ships dark and is toggled at runtime.
**Explanation**: Use Azure App Configuration feature management (`Microsoft.FeatureManagement`) with targeting filters for gradual rollout. Slot traffic routing uses the `x-ms-routing-name` cookie for sticky sessions.
**Code / YAML / Bicep / CLI**
```bash
az webapp traffic-routing set -g rg -n app --distribution staging=10
az webapp traffic-routing clear -g rg -n app
```
```csharp
if (await _features.IsEnabledAsync("NewPricing"))
    return NewPricing(order);
return LegacyPricing(order);
```

### 6.4 [Medium] Database migrations in the pipeline
**Answer**: Use backward-compatible (expand/contract) migrations, applied by a controlled step - preferably an EF Core migration bundle or idempotent SQL script - before routing traffic to the new code. Avoid `Database.Migrate()` at app startup in multi-instance apps.
**Explanation**: Old and new code run simultaneously during swap/canary, so schema must support both: add nullable columns first, backfill, then remove old columns in a later release. A bundle is a self-contained executable needing no SDK on the agent.
**Code / YAML / Bicep / CLI**
```bash
dotnet ef migrations bundle --self-contained -r linux-x64 -o efbundle
./efbundle --connection "$CONN"
dotnet ef migrations script --idempotent -o migrate.sql
```

### 6.5 [Hard] Swap failures and slot-sticky gotchas in production
**Answer**: Swaps fail on warm-up non-200s, missing Key Vault access for slot identity, or config differences; and sticky vs non-sticky mistakes can point staging at prod resources. Slots have their own identity (SAMI) and need their own role assignments.
**Explanation**: A slot with SAMI has a different principalId than production - Key Vault/Service Bus roles must be granted to both, or use a shared UAMI. Triggers (Service Bus/queue) in the staging slot can steal messages from the prod queue because both consume the same queue - mark trigger-related settings (e.g. `AzureWebJobs.<Func>.Disabled`) slot-sticky and disabled in staging. Timer triggers likewise double-fire.
**Code / YAML / Bicep / CLI**
```bicep
resource stickyNames 'Microsoft.Web/sites/config@2023-12-01' = {
  parent: func
  name: 'slotConfigNames'
  properties: { appSettingNames: ['AzureWebJobs.ProcessOrder.Disabled', 'ENVIRONMENT_NAME'] }
}
```
**Gotchas / follow-ups**
- Functions in a slot: keep the staging copy disabled until after swap, then enable.
- `WEBSITE_RUN_FROM_PACKAGE` URL settings swap with code unless sticky.
- Swap with preview lets you validate with prod settings before completing.

**Interview traps**
- Both slots consuming the same Service Bus queue.
- Forgetting RBAC for the slot's identity.
- `Migrate()` at startup race across instances.
- Destructive schema change in same release as code.

**Cheat sheet**
- Swap = warm, then flip; rollback = swap back.
- Sticky = stays with slot.
- Expand/contract migrations; EF bundle.
- Flags decouple deploy from release.

---

## Topic 7: Deploying Azure Functions

**Study notes**

```
zip -> (az functionapp deployment source config-zip | AzureFunctionApp@2 | func publish)
    -> run-from-package (WEBSITE_RUN_FROM_PACKAGE=1 | blob URL) -> read-only wwwroot
Plans: Consumption | Premium (EP) | Dedicated | Flex Consumption | Container (ACA/ACR)
```

Run-from-package mounts the zip read-only (atomic, faster cold start). With a blob URL the host needs read access (SAS or managed identity). Flex Consumption deploys via a blob deployment container configured in `functionAppConfig` (verify in current docs); some app settings like `WEBSITE_RUN_FROM_PACKAGE` are not used there.

### 7.1 [Easy] How do you deploy a .NET isolated Function?
**Answer**: Publish, zip, and deploy with `AzureFunctionApp@2`, `azure/functions-action`, `func azure functionapp publish`, or `az functionapp deployment source config-zip`. Prefer a pipeline-built artifact over publishing from a laptop.
**Explanation**: The deploying identity needs Contributor (or Website Contributor) on the app; with Kudu basic auth disabled, use Entra-based deployment (verify in current docs).
**Code / YAML / Bicep / CLI**
```bash
az functionapp deployment source config-zip -g rg -n func-orders-prod --src drop/app.zip
```
```yaml
- task: AzureFunctionApp@2
  inputs:
    azureSubscription: sc-prod
    appType: functionApp
    appName: func-orders-prod
    package: $(Pipeline.Workspace)/drop/app.zip
    deployToSlotOrASE: true
    resourceGroupName: rg-app-prod
    slotName: staging
```

### 7.2 [Follow-up] What is run-from-package and why use it?
**Answer**: `WEBSITE_RUN_FROM_PACKAGE=1` runs the app directly from the deployed zip, mounted read-only. Deployments are atomic, files can't drift, and cold starts improve.
**Explanation**: You can't write to wwwroot. Changing the package triggers a restart. Using a blob URL decouples package and app and works with managed-identity access.

### 7.3 [Medium] Slot settings and sticky settings for Functions
**Answer**: Mark environment-specific settings sticky (e.g. environment name, disabled-function flags) and keep shared ones non-sticky so they follow the code. Use `slotConfigNames` in Bicep or `--slot-settings` in CLI.
**Explanation**: Be careful with `AzureWebJobsStorage`: sharing storage across slots/apps is OK but the same function IDs and lease blobs coordinate; use `AzureFunctionsWebHost__hostid` or separate task hub names for Durable Functions.
**Code / YAML / Bicep / CLI**
```bash
az functionapp config appsettings set -g rg -n func --slot staging \
  --slot-settings AzureWebJobs.ProcessOrder.Disabled=1
```

### 7.4 [Medium] Containers and Flex Consumption - when?
**Answer**: Containerize when you need custom runtimes/dependencies or to run on Container Apps/AKS (KEDA scaling); Flex Consumption gives per-instance concurrency, VNet and fast scale with pay-per-use. Both change how you deploy (image push vs. zip to a deployment container).
**Explanation**: Verify Flex feature/region support and the current `functionAppConfig` schema in current docs; slots are not supported on Flex at time of writing - verify in current docs.

### 7.5 [Hard] Deployment succeeded but the Function does not appear / old code runs
**Answer**: Common causes are wrong package layout (host.json not at zip root), `FUNCTIONS_WORKER_RUNTIME` mismatch, stale `WEBSITE_RUN_FROM_PACKAGE` URL pointing to an old blob, host failing on startup due to unresolved Key Vault references, or triggers not syncing.
**Explanation**: Check Log stream/App Insights `traces`, the Functions "Diagnose and solve problems", and call `az functionapp function list`. After infra changes that alter triggers, a sync-triggers may be needed (ARM/zip deploys generally handle it; for external package URL, restart).
**Code / YAML / Bicep / CLI**
```bash
az functionapp function list -g rg -n func-orders-prod -o table
az functionapp restart -g rg -n func-orders-prod
```
**Gotchas / follow-ups**
- Mixed in-process/isolated settings after migration.
- Storage firewall blocks the host (AzureWebJobsStorage unreachable).
- Out-of-date SAS in package URL expires -> app breaks on restart.

**Interview traps**
- Deploying straight to production slot with no warm-up.
- SAS-URL packages that expire.
- Kudu/SCM basic auth disabled breaking old pipelines.

**Cheat sheet**
- zip + run-from-package; host.json at root.
- Sticky for env-specific settings.
- Flex/containers have different deployment models.
- Verify slot support per plan.

---

## Topic 8: Deploying Logic Apps (Standard & Consumption)

**Study notes**

```
Consumption: workflow = ARM resource (Microsoft.Logic/workflows) -> Bicep/ARM; connections = Microsoft.Web/connections
Standard:    Logic App site (Microsoft.Web/sites) + files: workflow.json, connections.json, parameters.json
             deploy = zip deploy (like Functions) ; infra = Bicep
```

Parameterize everything environment-specific. In Standard, `connections.json` holds service-provider (built-in) and managed API connections; `parameters.json` + app settings (`@appsetting('X')`) make workflows portable. Managed identity auth: built-in connectors use `ManagedServiceIdentity` authentication; managed API connections use `parameterValueSet`/`authentication` with MSI (verify per connector in current docs).

### 8.1 [Easy] How does deployment differ between Consumption and Standard?
**Answer**: Consumption workflows are ARM resources deployed with templates; Standard workflows are files in a Logic App site deployed via zip, like a Function app. Infra is still Bicep in both cases.
**Explanation**: Standard supports VS Code project structure, local run, slots, and VNet. Consumption is multi-tenant, per-execution billing and simpler.
**Code / YAML / Bicep / CLI**
```bash
cd la-project && zip -r ../la.zip . -x "local.settings.json" ".vscode/*"
az logicapp deployment source config-zip -g rg -n la-prod --src ../la.zip
```
(`az functionapp deployment source config-zip` also works for Standard - verify in current docs.)

### 8.2 [Follow-up] How do you parameterize per environment?
**Answer**: In Standard, reference app settings with `@appsetting('Name')` in `connections.json`/workflows and set those app settings through Bicep per environment. In Consumption, use ARM/Bicep parameters passed into the `definition` parameters.
**Explanation**: Keep `parameters.json` for non-secret values; secrets come from Key Vault references in app settings.
**Code / YAML / Bicep / CLI**
```json
{
  "serviceProviderConnections": {
    "serviceBus": {
      "parameterValues": {
        "fullyQualifiedNamespace": "@appsetting('sb_fqdn')",
        "authProvider": { "Type": "ManagedServiceIdentity" }
      },
      "serviceProvider": { "id": "/serviceProviders/serviceBus" },
      "displayName": "serviceBus"
    }
  }
}
```

### 8.3 [Medium] Managed identity connections - permissions to set up
**Answer**: Give the Logic App's identity the data-plane RBAC role on the target (e.g. Service Bus Data Receiver) and choose the identity in the connection. For managed (shared) connectors, Consumption uses `Microsoft.Web/connections` with MSI auth where supported.
**Explanation**: Standard: SAMI or UAMI (specify identity in the connection). Role assignment must exist before first run, else 401/403 at runtime, not at deploy.
**Code / YAML / Bicep / CLI**
```bicep
resource laRole 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  scope: ns
  name: guid(ns.id, la.id, sbDataReceiverRoleId)
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', sbDataReceiverRoleId)
    principalId: la.identity.principalId
    principalType: 'ServicePrincipal'
  }
}
```

### 8.4 [Hard] Pitfalls moving a Logic App between environments
**Answer**: Hard-coded resource IDs/URLs in workflows, API connections requiring interactive OAuth consent (e.g. Office 365), callback URLs changing per environment, and connection authorization state not deploying. Managed API connections with OAuth need manual authorization after deployment, or use service principals/MSI.
**Explanation**: Treat workflow trigger URLs (SAS signature) as secrets and re-fetch after deployment (`listCallbackUrl`). Use a dedicated service account, not personal credentials, for OAuth connections. Slot swaps on Standard behave like Functions (triggers duplicate in staging).
**Gotchas / follow-ups**
- Consumption redeploy with changed trigger resets run state; check "enabled" state.
- Source-control workflow JSON designer artifacts are stable; avoid hand edits that the designer can't parse.

**Interview traps**
- Believing a template deploy authorizes OAuth connections.
- Hard-coded subscription IDs in workflow JSON.
- No role assignment for the MI.

**Cheat sheet**
- Standard = zip; Consumption = ARM.
- `@appsetting()` for config.
- MI + RBAC before first run.
- OAuth connectors need authorization step.

---

## Topic 9: Secrets in pipelines & least-privilege deployment identities

**Study notes**

```
Pipeline --OIDC--> Entra app / UAMI (deployer) --RBAC--> scope (RG, not subscription)
   |                                   \-> Contributor (resources)  + RBAC Administrator (restricted) if it must assign roles
   \-> runtime secrets: Key Vault refs / managed identity, not pipeline variables
```

Separate identities for infra (Contributor on RG + role-assignment rights) and app deploy (Website Contributor). Separate per environment, prod with approvals. Contributor **cannot** create role assignments.

### 9.1 [Easy] Why avoid client secrets in pipelines?
**Answer**: Secrets expire, leak through logs/forks, and need rotation. OIDC federation issues short-lived tokens bound to repo/branch/environment so nothing long-lived is stored.
**Explanation**: Where a secret is unavoidable, store in Key Vault, scope tightly, rotate, and use `@secure()` params.

### 9.2 [Follow-up] Which RBAC does a deployment identity need?
**Answer**: Contributor at resource-group scope for resources, plus a way to write role assignments if your Bicep does so - either User Access Administrator / Owner at that scope, or better, Role Based Access Control Administrator with a condition limiting which roles/principals can be assigned.
**Explanation**: Avoid subscription-wide Owner. Subscription-level Bicep (creating RGs) needs rights at subscription scope; pre-create RGs to avoid that.
**Code / YAML / Bicep / CLI**
```bash
az role assignment create --assignee-object-id $SP_OID --assignee-principal-type ServicePrincipal \
  --role "Contributor" --scope /subscriptions/$SUB/resourceGroups/rg-app-prod
az role assignment create --assignee-object-id $SP_OID --assignee-principal-type ServicePrincipal \
  --role "User Access Administrator" --scope /subscriptions/$SUB/resourceGroups/rg-app-prod
```

### 9.3 [Medium] How do you pass secrets to Bicep safely?
**Answer**: Mark params `@secure()`, source values from Key Vault (`getSecret` in `.bicepparam`, or Key Vault reference in parameter files), and never output them. Better: don't pass secrets - use managed identity or generate and store them in Key Vault inside the template.
**Explanation**: `getSecret` in `.bicepparam` requires the deployer to have secret get access and (for ARM-side reference) the vault's `enabledForTemplateDeployment` if using classic references - verify in current docs.
**Code / YAML / Bicep / CLI**
```bicep
using 'main.bicep'
param sqlPassword = az.getSecret('<subId>', 'rg-shared', 'kv-shared', 'sql-admin-password')
```
```bicep
@secure()
param sqlPassword string
```

### 9.4 [Hard] Secrets leaking in logs - where and how?
**Answer**: Leaks occur from echoing variables, `set -x`, non-secret params, outputs, error messages printing connection strings, derived/transformed values escaping masking, and `az ... --debug`. Masking is exact-match only.
**Explanation**: In ADO, mark values secret and pass via `env:`; in GHA use `::add-mask::` for dynamically obtained values. Never log `listKeys` results. Review deployment history in portal - non-secure params and outputs are visible there.
**Code / YAML / Bicep / CLI**
```bash
TOKEN=$(az account get-access-token --query accessToken -o tsv)
echo "::add-mask::$TOKEN"   # GitHub Actions
echo "##vso[task.setvariable variable=token;issecret=true]$TOKEN"  # Azure DevOps
```
**Gotchas / follow-ups**
- Fork/PR pipelines from untrusted code can exfiltrate secrets - restrict triggers and use environments.
- Rotate anything that ever hit a log.

**Interview traps**
- Contributor can assign roles - it cannot.
- `@secure()` makes outputs safe - outputs should not carry secrets at all.
- Subscription Owner for the pipeline "to make it work".

**Cheat sheet**
- OIDC > secret; per-env identities.
- RG-scope Contributor + UAA/conditioned RBAC Admin.
- `@secure()`, KV `getSecret`.
- Mask dynamic secrets.

---

## Topic 10: Environment promotion & per-environment config

**Study notes**

```
build once -> artifact v1.2.3 -> dev (auto) -> test (auto+tests) -> staging (approval) -> prod (approval, window)
config: per-env .bicepparam (infra)  |  App Configuration (settings, flags)  |  Key Vault (secrets)
```

Rule: same artifact + same template, different parameters. Environment-specific values never in code. App Configuration holds non-secret config and feature flags (with labels per environment) and can hold Key Vault references.

### 10.1 [Easy] What is environment promotion?
**Answer**: Moving the same immutable build through dev -> test -> staging -> prod, gated by automated tests and approvals. Only configuration changes between stages.
**Explanation**: This guarantees what you tested is what you ship. Use pipeline environments with approvals/checks (ADO) or protection rules (GHA).
**Code / YAML / Bicep / CLI**
```yaml
- stage: Prod
  dependsOn: Staging
  jobs:
  - deployment: prod
    environment: prod     # approvals/checks configured on the environment
    strategy:
      runOnce:
        deploy:
          steps:
          - template: steps/deploy.yml
            parameters: { env: prod }
```

### 10.2 [Follow-up] Where does each kind of config live?
**Answer**: Topology/SKU in `.bicepparam` per environment; app settings in App Service/App Configuration; secrets in Key Vault; feature flags in App Configuration. The app reads config via the .NET configuration providers.
**Explanation**: Environment parameter files keep diffs reviewable; avoid duplicating Bicep per environment.
**Code / YAML / Bicep / CLI**
```csharp
builder.Configuration.AddAzureAppConfiguration(o =>
    o.Connect(new Uri(endpoint), new DefaultAzureCredential())
     .Select(KeyFilter.Any, labelFilter: env)
     .ConfigureKeyVault(kv => kv.SetCredential(new DefaultAzureCredential()))
     .UseFeatureFlags());
```

### 10.3 [Medium] App Configuration vs Key Vault vs app settings?
**Answer**: App settings are simple and require restart on change; App Configuration offers central, labeled, dynamically refreshable settings and feature flags; Key Vault stores secrets with audit/rotation. App Configuration should reference, not store, secrets.
**Explanation**: Dynamic refresh requires a sentinel key and the refresh middleware; adds a dependency and its own RBAC (App Configuration Data Reader).

### 10.4 [Hard] Promotion gotchas
**Answer**: Config drift between environments, prod-only config never tested, manual portal changes, shared resources (e.g. one Service Bus namespace used by two environments), and approvals that rubber-stamp. Detect with what-if and policy; keep non-prod topology as close to prod as budget allows.
**Explanation**: Use the same Bicep with SKU parameters; test infra changes in lower environments first. Version infra and app together or ensure compatibility (infra first, backward-compatible).
**Gotchas / follow-ups**
- Hotfix path: branch -> same pipeline with expedited approval, never bypass the pipeline.
- Approvals should be from different people than the author; use required reviewers/ branch protection.

**Interview traps**
- Rebuilding per environment.
- Separate Bicep copies per environment.
- Secrets in App Configuration.

**Cheat sheet**
- Build once, promote many.
- `.bicepparam` per env.
- AppConfig labels + sentinel refresh.
- Approvals at environment level.

---

## Topic 11: GitOps, branching, PR validation, policy as code, drift

**Study notes**

```
PR -> build + unit tests + bicep lint/build + what-if (read-only identity) -> review -> merge main
main -> CD pipeline -> envs        (Git = desired state; pipeline = reconciler)
Azure Policy (deny/audit/modify/deployIfNotExists)  guards what pipelines can create
```

Trunk-based: short-lived branches, merge to main frequently, flags for incomplete work, release from main. GitFlow: long-lived develop/release branches - heavier, suits versioned/boxed releases. For cloud services trunk-based is generally preferred.

### 11.1 [Easy] Trunk-based vs GitFlow?
**Answer**: Trunk-based uses one main branch with short-lived feature branches and flags; GitFlow adds develop, release and hotfix branches. Trunk-based reduces merge pain and fits continuous delivery.
**Explanation**: Branch policies (required reviewers, build validation, no direct push) enforce quality in either model.
**Code / YAML / Bicep / CLI**
```bash
git switch -c feature/add-dlq-alert
git push -u origin feature/add-dlq-alert
```

### 11.2 [Follow-up] What belongs in PR validation?
**Answer**: Build, unit tests, Bicep build/lint, and a what-if posted for reviewers, using a read-only identity. No deploy and no prod secrets on PR builds.
**Explanation**: what-if needs Reader plus deployment-validate permissions; ARM what-if requires `Microsoft.Resources/deployments/whatIf/action` (included in Contributor; a custom role can scope it).
**Code / YAML / Bicep / CLI**
```yaml
pr:
  branches: { include: [main] }
steps:
- script: az bicep build --file infra/main.bicep
- task: AzureCLI@2
  inputs:
    azureSubscription: sc-whatif-ro
    scriptType: bash
    scriptLocation: inlineScript
    inlineScript: az deployment group what-if -g rg-app-dev -f infra/main.bicep -p infra/main.dev.bicepparam
```

### 11.3 [Medium] Policy as code
**Answer**: Define Azure Policy/initiatives as code (Bicep/JSON) and assign at management group/subscription to enforce tagging, allowed regions/SKUs, private endpoints, and diagnostics. Effects include Audit, Deny, Modify, DeployIfNotExists.
**Explanation**: Policy denial surfaces as `RequestDisallowedByPolicy` in deployment errors. Roll out new policies in Audit first. Complement with `bicep` linter and tools like PSRule for Azure (optional).
**Code / YAML / Bicep / CLI**
```bicep
targetScope = 'subscription'
resource requireTag 'Microsoft.Authorization/policyAssignments@2022-06-01' = {
  name: 'require-owner-tag'
  properties: {
    displayName: 'Require owner tag on RGs'
    policyDefinitionId: '/providers/Microsoft.Authorization/policyDefinitions/<built-in-id>' // verify built-in ID in current docs
    parameters: { tagName: { value: 'owner' } }
  }
}
```

### 11.4 [Hard] Drift detection and remediation
**Answer**: Drift is live state diverging from code (portal edits). Detect via scheduled `what-if` runs, Azure Resource Graph/Activity Log queries, and policy compliance; remediate by re-deploying or codifying the change. Deployment stacks with deny settings can prevent out-of-band edits (verify in current docs).
**Explanation**: In Terraform, `plan` against state shows drift natively; for Bicep what-if is noisy so triage. Lock critical resources (`CanNotDelete`) - note locks can break some operations/deployments.
**Code / YAML / Bicep / CLI**
```yaml
schedules:
- cron: '0 5 * * 1-5'
  branches: { include: [main] }
  always: true
```
```bash
az graph query -q "Resources | where tags['managedBy'] != 'bicep' | project name, type, resourceGroup"
```
**Gotchas / follow-ups**
- Scheduled what-if with exit code: parse JSON `changes` - non-zero changes != failure by itself.
- Autoscale/platform-managed properties produce false positives.

**Interview traps**
- Long-lived branches in "trunk-based".
- Deploying from PR builds with prod credentials.
- Enabling Deny policy without Audit first.

**Cheat sheet**
- PR: build/test/lint/what-if only.
- Policy: Audit -> Deny; DINE for diagnostics.
- Drift: scheduled what-if + Resource Graph.
- Lock != governance replacement.

---

## Topic 12: ACR, Docker for .NET, Container Apps / AKS overview

**Study notes**

```
Dockerfile (multi-stage: sdk -> build/publish -> aspnet runtime) -> docker build -> ACR (az acr build / push)
   -> Container Apps (revisions, KEDA)  or  AKS (Deployment, HPA, ingress)
Pull auth: managed identity with AcrPull (no admin user)
```

### 12.1 [Easy] Multi-stage Dockerfile for .NET
**Answer**: Build with the SDK image, publish, then copy output to a smaller `aspnet`/runtime image. This keeps images small and without build tools.
**Explanation**: Order COPY of csproj before source for layer caching. Run as non-root (Microsoft images provide a non-root `app` user in recent .NET versions - verify for your version).
**Code / YAML / Bicep / CLI**
```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY *.csproj .
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /out --no-restore

FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app
COPY --from=build /out .
USER $APP_UID
ENTRYPOINT ["dotnet", "Api.dll"]
```

### 12.2 [Follow-up] Pushing to ACR and pulling securely
**Answer**: Build with `az acr build` (cloud build) or docker push; grant the runtime's managed identity `AcrPull` and disable the admin user. Tag with immutable version/commit SHA, not only `latest`.
**Explanation**: Registry tasks need no local Docker daemon on the agent.
**Code / YAML / Bicep / CLI**
```bash
az acr build -r myacr -t orders-api:$(git rev-parse --short HEAD) .
az role assignment create --assignee-object-id $UAMI_OID --assignee-principal-type ServicePrincipal \
  --role AcrPull --scope $(az acr show -n myacr --query id -o tsv)
```

### 12.3 [Medium] Deploying to Container Apps
**Answer**: Update the container app with a new image; each change creates a revision. Traffic can be split between revisions for canary/blue-green; scale rules use KEDA (e.g. Service Bus queue length).
**Explanation**: Use `revisionsMode: Multiple` to split traffic. Identity-based registry pull via `registries` with `identity`.
**Code / YAML / Bicep / CLI**
```bash
az containerapp update -n orders-api -g rg --image myacr.azurecr.io/orders-api:abc123
az containerapp ingress traffic set -n orders-api -g rg --revision-weight orders-api--v2=10 orders-api--v1=90
```

### 12.4 [Hard] AKS deployment overview and tradeoffs
**Answer**: AKS deploys via manifests/Helm/GitOps (Flux extension) with rolling updates by default; you manage nodes, networking, ingress and upgrades. Choose Container Apps for simpler microservices, AKS when you need full Kubernetes control.
**Explanation**: Use workload identity (federated credential for a service account) for pod access to Key Vault/Service Bus. Readiness/liveness probes drive rolling update safety; set `maxUnavailable`/`maxSurge`.
**Code / YAML / Bicep / CLI**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: orders-api }
spec:
  replicas: 3
  strategy: { type: RollingUpdate, rollingUpdate: { maxSurge: 1, maxUnavailable: 0 } }
  selector: { matchLabels: { app: orders-api } }
  template:
    metadata: { labels: { app: orders-api } }
    spec:
      containers:
      - name: api
        image: myacr.azurecr.io/orders-api:abc123
        readinessProbe: { httpGet: { path: /health/ready, port: 8080 } }
```
**Gotchas / follow-ups**
- `latest` tag + cached images -> stale pods.
- ACR admin user left enabled.
- Default .NET 8 container port is 8080 (not 80) - verify for your image version.

**Interview traps**
- Using `latest` in production.
- Image pull failures from missing AcrPull.
- Choosing AKS when Container Apps suffices.

**Cheat sheet**
- Multi-stage, non-root.
- `az acr build`; AcrPull via MI.
- ACA revisions = canary.
- AKS: probes + workload identity.

---

## Topic 13: Testing in pipelines

**Study notes**

```
unit (fast, no I/O) -> integration (Testcontainers / Azurite / emulators) -> deploy dev
   -> smoke tests post-deploy -> (optional) load tests -> promote
```

Azurite emulates Blob/Queue/Table. Service Bus has an official emulator (verify current availability/limits in docs); Testcontainers runs Docker dependencies from test code (needs Docker on the agent - Microsoft-hosted Ubuntu agents have it).

### 13.1 [Easy] Test pyramid in CI
**Answer**: Many fast unit tests, fewer integration tests, few end-to-end/smoke tests. Unit tests gate every PR; slower tests run post-merge or per stage.
**Explanation**: Tag tests with categories/traits to select them in different stages.
**Code / YAML / Bicep / CLI**
```bash
dotnet test --filter "Category=Unit"
dotnet test --filter "Category=Integration"
```

### 13.2 [Follow-up] Integration tests with Testcontainers and Azurite
**Answer**: Spin up containers from the test fixture (SQL, Redis, Azurite), run tests against them, dispose after. Results are hermetic and reproducible.
**Explanation**: Wait strategies matter to avoid flakiness. Emulators differ from real services (e.g. no Entra auth, limited features), so keep a few real-cloud tests.
**Code / YAML / Bicep / CLI**
```csharp
public class BlobFixture : IAsyncLifetime
{
    private readonly IContainer _azurite = new ContainerBuilder()
        .WithImage("mcr.microsoft.com/azure-storage/azurite")
        .WithPortBinding(10000, true)
        .WithWaitStrategy(Wait.ForUnixContainer().UntilPortIsAvailable(10000))
        .Build();
    public string ConnectionString => $"DefaultEndpointsProtocol=http;AccountName=devstoreaccount1;AccountKey=Eby8vdM02xNOcqFlqUwJPLlmEtlCDXJ1OUzFT50uSRZ6IFsuFq2UVErCz4I6tq/K1SZFPTOtr/KBHBeksoGMGw==;BlobEndpoint=http://127.0.0.1:{_azurite.GetMappedPublicPort(10000)}/devstoreaccount1;";
    public Task InitializeAsync() => _azurite.StartAsync();
    public Task DisposeAsync() => _azurite.DisposeAsync().AsTask();
}
```
(The key above is Azurite's public well-known dev key, not a secret.)

### 13.3 [Medium] Post-deploy smoke tests
**Answer**: After deploy, call health and a few critical read-only endpoints on the staging slot/new revision before swapping or shifting traffic. Failure stops promotion/auto-rolls back.
**Explanation**: Make smoke tests idempotent and safe in prod (synthetic data, read-only). Retry with backoff to allow warm-up.
**Code / YAML / Bicep / CLI**
```bash
for i in 1 2 3 4 5; do
  code=$(curl -s -o /dev/null -w "%{http_code}" https://func-orders-prod-staging.azurewebsites.net/api/health)
  [ "$code" = "200" ] && exit 0; sleep 10
done; exit 1
```

### 13.4 [Hard] Load tests and flakiness
**Answer**: Use Azure Load Testing (JMeter/Locust based; verify current features) or k6 against a prod-like environment, with pass/fail criteria (p95 latency, error rate). Run against dedicated environment, not shared prod.
**Explanation**: Scale limits and downstream throttling (Service Bus, Storage) shape results; include dependency metrics. Flaky tests: isolate shared state, avoid fixed sleeps, use unique names per run.
**Gotchas / follow-ups**
- Testcontainers requires Docker; fails on agents without it.
- Real Service Bus in tests: use per-run queue names and clean up.
- Load tests billing/quotas.

**Interview traps**
- Treating emulator parity as production parity.
- Smoke tests that mutate prod data.
- Fixed `Thread.Sleep` waits.

**Cheat sheet**
- Pyramid + categories.
- Testcontainers/Azurite for hermetic tests.
- Smoke before swap.
- Load tests with explicit thresholds.

---

## Topic 14: Release health & rollback

**Study notes**

```
deploy -> health checks + App Insights (failures, latency, exceptions, DLQ depth) -> gate
   ok  -> promote / swap        bad -> auto rollback (swap back / previous revision / flag off / redeploy last good artifact)
```

Rollback options, fastest first: flag off, swap back, shift traffic to previous revision, redeploy previous artifact. Database changes are the hard part - hence expand/contract.

### 14.1 [Easy] How do you know a release is healthy?
**Answer**: Watch failed requests, response time, exceptions, dependency failures and business/queue metrics (e.g. dead-letter count) against baseline, plus health endpoint checks. Alerts and release gates tie these to the pipeline.
**Explanation**: Mark deployments in App Insights (annotations) to correlate regressions with releases.
**Code / YAML / Bicep / CLI**
```kusto
requests
| where timestamp > ago(30m)
| summarize failed = countif(success == false), total = count(), p95 = percentile(duration, 95) by bin(timestamp, 5m), cloud_RoleName
```

### 14.2 [Follow-up] What are the rollback mechanisms for each hosting model?
**Answer**: App Service/Functions: swap back. Container Apps: re-activate/route to previous revision. AKS: `kubectl rollout undo`. Otherwise redeploy the previous artifact through the same pipeline.
**Explanation**: Keep N previous artifacts retained; rollback should be rehearsed and use the same pipeline path.
**Code / YAML / Bicep / CLI**
```bash
az webapp deployment slot swap -g rg -n app --slot staging --target-slot production   # after a swap, staging holds the previous version
kubectl rollout undo deployment/orders-api
az containerapp ingress traffic set -n orders-api -g rg --revision-weight orders-api--v1=100
```

### 14.3 [Medium] Automated rollback in a pipeline
**Answer**: Add a post-deploy verification stage; on failure run a rollback step (`condition: failed()`), e.g. swap back or traffic reset. Keep it automated and idempotent.
**Explanation**: Define explicit SLO thresholds and a bake time; Azure DevOps "gates" can query Azure Monitor alerts (verify current gate names in docs).
**Code / YAML / Bicep / CLI**
```yaml
- stage: Verify
  jobs:
  - job: smoke
    steps: [ { script: ./smoke.sh } ]
- stage: Rollback
  dependsOn: Verify
  condition: failed()
  jobs:
  - job: rollback
    steps:
    - task: AzureCLI@2
      inputs:
        azureSubscription: sc-prod
        scriptType: bash
        scriptLocation: inlineScript
        inlineScript: az webapp deployment slot swap -g rg -n app --slot staging --target-slot production
```

### 14.4 [Hard] Rollback gotchas: data and messages
**Answer**: Code can roll back but data cannot - messages already processed or schema changes persist. New-format messages on Service Bus may break old consumers, so keep message contracts backward compatible and versioned.
**Explanation**: After a bad release, poison messages may be in DLQ; replay after fix. Non-reversible migrations need roll-forward plans. Swapping back also swaps non-sticky settings back.
**Gotchas / follow-ups**
- "Roll forward" is often faster than rollback if the fix is small and pipeline is quick.
- Rolled-back slot still has the bad code warm - redeploy a good build there.

**Interview traps**
- Assuming DB rolls back with code.
- Rollback never rehearsed.
- Ignoring in-flight messages.

**Cheat sheet**
- Flag off < swap back < redeploy.
- Gate on SLOs; annotate releases.
- Backward-compatible contracts.
- Plan roll-forward.

---

## Topic 15: Cost governance & tagging in IaC

**Study notes**

```
Tags (env, owner, costCenter, app) set in Bicep + Policy (require/inherit via Modify) -> Cost Management grouping
Budgets + alerts | right-size SKUs per env | autoshutdown / scale-to-zero | retention limits (Log Analytics)
```

### 15.1 [Easy] Why tag resources, and how in Bicep?
**Answer**: Tags enable cost allocation, ownership and automation. Define a common tag object and apply it to every resource.
**Explanation**: Tags are not inherited from RGs by default; enforce via Policy (Modify effect / inherit tag built-ins).
**Code / YAML / Bicep / CLI**
```bicep
param tags object = { env: env, owner: 'team-orders', costCenter: '1234', managedBy: 'bicep' }
resource st 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: stName
  location: location
  tags: tags
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
}
```

### 15.2 [Follow-up] Budgets and alerts as code
**Answer**: Define `Microsoft.Consumption/budgets` with thresholds and contact emails/action groups at subscription or RG scope.
**Code / YAML / Bicep / CLI**
```bicep
targetScope = 'subscription'
resource budget 'Microsoft.Consumption/budgets@2023-11-01' = {
  name: 'monthly-app'
  properties: {
    category: 'Cost'
    amount: 500
    timeGrain: 'Monthly'
    timePeriod: { startDate: '2026-01-01' }
    notifications: {
      actual80: { enabled: true, operator: 'GreaterThan', threshold: 80, contactEmails: ['team@contoso.com'] }
    }
  }
}
```
**Explanation**: Budgets alert; they do not stop spend. Verify API versions in current docs.

### 15.3 [Medium] Cost levers in this stack
**Answer**: Service Bus tier (Standard vs Premium), Function plan (Consumption/Flex vs Premium always-warm), Log Analytics ingestion and retention (largest silent cost - use sampling, table plans, daily cap), Redis tier, and non-prod shutdown or lower SKUs via parameters.
**Explanation**: App Insights sampling and diagnostic setting category selection cut ingestion. Parameterize SKUs per environment.
**Code / YAML / Bicep / CLI**
```bicep
param planSku object = env == 'prod' ? { name: 'EP1', tier: 'ElasticPremium' } : { name: 'Y1', tier: 'Dynamic' }
```

### 15.4 [Hard] Governance pitfalls
**Answer**: Tag-based reports break when tags are inconsistent or case-different; policy `Modify` needs a managed identity with role rights; Complete mode and locks interact badly with cleanup; orphaned resources (disks, public IPs, PEs) accumulate when removed from Bicep in Incremental mode.
**Explanation**: Use scheduled Resource Graph queries to find untagged/orphaned resources and Azure Advisor for rightsizing.
**Gotchas / follow-ups**
- Some resource types don't support tags - verify.
- Tag value limits (256 chars) and name limits.

**Interview traps**
- Believing budgets cap spending.
- Ignoring log ingestion cost.
- Same prod SKUs in dev.

**Cheat sheet**
- Common `tags` param on every resource.
- Policy for enforcement/inheritance.
- Budgets = alerts only.
- Parameterize SKUs.

---

## Topic 16: Pipeline troubleshooting scenarios

**Study notes**

```
Symptom -> layer: auth (service connection/OIDC) | RBAC (deployer) | ARM (template/policy/quota) | platform (swap/warm-up) | app (startup/config)
Tools: deployment operations (az deployment group show / operation list), Activity Log, Kudu/log stream, App Insights
```

### 16.1 [Easy] Service connection authorization failure
**Answer**: Typical errors: AADSTS70021 (no matching federated credential - subject mismatch), AADSTS7000222 (expired secret), or "does not have authorization to perform action ... over scope" (RBAC missing). Fix the credential subject/secret or grant the role at the right scope.
**Explanation**: Check the service connection's issuer/subject versus the Entra federated credential. Role assignments can take minutes to propagate.
**Code / YAML / Bicep / CLI**
```bash
az role assignment list --assignee $CLIENT_ID --all -o table
az ad app federated-credential list --id $APP_OBJECT_ID
```

### 16.2 [Follow-up] "AuthorizationFailed ... roleAssignments/write"
**Answer**: Contributor cannot create role assignments. Grant the deployment identity User Access Administrator, Owner, or a conditioned RBAC Administrator at the narrowest scope that contains the assignment target.
**Explanation**: The scope must contain the resource where the assignment is made - a role on RG A doesn't help when assigning on a Key Vault in RG B.
**Code / YAML / Bicep / CLI**
```bash
az role assignment create --assignee-object-id $SP_OID --assignee-principal-type ServicePrincipal \
  --role "Role Based Access Control Administrator" --scope $KV_ID
```

### 16.3 [Medium] Deployment stuck or timing out
**Answer**: Inspect deployment operations to see which resource is stuck. Common causes: long-running provisioning (Premium Service Bus, App Gateway), dependent resource waiting, soft-delete name conflicts, quota limits, or a concurrent deployment/lock on the same RG.
**Explanation**: Cancel with `az deployment group cancel`; re-run is idempotent. A "DeploymentActive"/"another operation in progress" error means wait or cancel. Pipeline timeouts (job timeout default is limited - set `timeoutInMinutes`) can kill the client while ARM continues.
**Code / YAML / Bicep / CLI**
```bash
az deployment operation group list -g rg-app-prod -n main --query "[?properties.provisioningState!='Succeeded'].{r:properties.targetResource.resourceName,s:properties.provisioningState,m:properties.statusMessage}" -o json
az deployment group cancel -g rg-app-prod -n main
```

### 16.4 [Medium] Slot swap failures
**Answer**: Causes: warm-up ping not returning accepted status, app failing on startup in the slot (unresolved Key Vault refs because the slot identity lacks a role), storage/network restrictions, or a conflicting sticky setting. Review the swap error and slot log stream.
**Explanation**: Test with swap-with-preview; ensure the slot has its own role assignments (SAMI) or share a UAMI. Check `WEBSITE_SWAP_WARMUP_PING_PATH`. For Functions, ensure host starts.
**Code / YAML / Bicep / CLI**
```bash
az webapp log tail -g rg -n app --slot staging
az webapp config appsettings list -g rg -n app --slot staging -o table
```

### 16.5 [Hard] Bicep circular dependency & what-if surprises
**Answer**: A cycle means two resources reference each other's runtime properties; break it with a UAMI created first, move one side into a module, reference `.id`/names (compile-time known) rather than `.properties`, or split deployments. What-if surprises (phantom modifications, unresolved expressions) are tolerated noise; verify by reading the property diff and checking API version defaults.
**Explanation**: Example: Function SAMI principalId -> role assignment on KV -> KV firewall rule needs function outbound IPs from the same site. Use `ignore`/`noEffect` categories mentally: what-if can classify as "Ignore" and sometimes shows Modify for unchanged values because the template omits properties ARM defaults. Pin API versions and set defaults explicitly to reduce noise.
**Code / YAML / Bicep / CLI**
```bash
az deployment group what-if -g rg -f main.bicep -p main.dev.bicepparam --result-format FullResourcePayloads
az deployment group what-if -g rg -f main.bicep -p main.dev.bicepparam --no-pretty-print > whatif.json
```
**Gotchas / follow-ups**
- Parameter secrets leak: non-`@secure()` params and outputs appear in deployment history and logs; `--debug` prints request bodies.
- Rotate any leaked secret, then enforce `@secure()` and masking.
- Quota/policy errors (`RequestDisallowedByPolicy`, `QuotaExceeded`) are ARM-side, not pipeline bugs.

**Interview traps**
- Contributor can write role assignments - no.
- Re-running a cancelled deployment is unsafe - ARM is idempotent.
- What-if is authoritative.
- Fixing a leak by only deleting the log.

**Cheat sheet**
- Auth error -> check federated subject / secret expiry.
- Role write -> UAA/RBAC Admin.
- `az deployment operation group list` for stuck deploys.
- Swap failure -> warm-up + slot identity RBAC.
- Cycles: UAMI first, use `.id`, modules.
