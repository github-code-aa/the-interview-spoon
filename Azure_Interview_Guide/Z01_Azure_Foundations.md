# Z01 - Azure Foundations (Study + Interview Prep)

**How to use:** Read the *Study notes* of each topic first, then try to answer each ladder question aloud before reading the **Answer**; the ladder goes Easy -> Tricky.
Limits, prices, SDK versions and preview features change - anything marked "verify in current docs" must be checked before quoting it in an interview.

## Table of Contents
1. Azure hierarchy (tenant, management groups, subscriptions, RGs, resources)
2. ARM vs Bicep vs Terraform
3. Regions, availability zones, paired regions, SLAs
4. RBAC vs Azure Policy vs resource locks
5. Control plane vs data plane
6. Tags, naming, cost management and budgets
7. Well-Architected Framework pillars
8. Choosing compute (App Service / Functions / Container Apps / AKS / VMs)
9. PaaS vs IaaS vs serverless
10. Azure Resource Graph and Azure CLI / PowerShell basics
11. Azure SDK for .NET fundamentals
12. Resiliency and transient faults in Azure SDKs
13. Azure Landing Zone concepts
14. Environment strategy (dev/test/prod)
15. Quotas and throttling (429)
16. Azure DevOps vs GitHub (brief)
17. "Which service would you choose?" questions

---

## Topic 1: Azure hierarchy (tenant, management groups, subscriptions, resource groups, resources)

**Study notes**

```
Microsoft Entra tenant (identity boundary; one directory)
 └── Tenant Root Management Group
      └── Management Groups (up to 6 levels deep excluding root - verify in current docs)
           └── Subscriptions (billing + quota + access boundary)
                └── Resource Groups (lifecycle container; has a location for metadata)
                     └── Resources (Service Bus namespace, Function app, ...)
```

- RBAC assignments and Azure Policy assignments **inherit downward** through this tree.
- A subscription trusts exactly one tenant; it can be moved to another tenant (RBAC is lost).
- A resource belongs to exactly one resource group; resources in an RG can be in different regions.
- Subscription = unit of billing, quota (e.g. vCPU per region), and many service limits.

### 1.1 [Easy] What is the Azure resource hierarchy?
**Answer**: Tenant (Entra ID) at the top, then management groups, subscriptions, resource groups and finally resources. Access (RBAC) and governance (Policy) assigned at a higher scope inherit to everything below. Subscriptions are billing/quota boundaries; resource groups are lifecycle containers.

**Explanation**: Scope in Azure always means one of: management group, subscription, resource group, resource. Role assignments and policy assignments are scope-based and inherit. Management groups only contain subscriptions and other MGs, never RGs directly.

**Code / CLI / Bicep**
```bash
az account list -o table
az account management-group list -o table
az group list --query "[].{name:name,loc:location}" -o table
az resource list -g rg-orders-prod -o table
```

### 1.2 [Follow-up] Difference between a tenant and a subscription?
**Answer**: A tenant is an Entra ID directory - the identity boundary holding users, groups, app registrations and service principals. A subscription is a billing and resource boundary that trusts one tenant for authentication/authorization. One tenant can have many subscriptions.

**Explanation**: Managed identities (SAMI/UAMI) live as service principals in the tenant of the subscription that hosts the resource. "Directory" and "tenant" are used interchangeably. Cross-tenant access needs guest users, multi-tenant apps or Azure Lighthouse.

**Code / CLI / Bicep**
```bash
az account show --query "{sub:id,tenant:tenantId}" -o json
az account set --subscription "<sub-id-or-name>"
```

### 1.3 [Medium] When would you create a new subscription vs a new resource group?
**Answer**: New RG for a separate lifecycle (deploy/delete together) within the same workload and access model. New subscription for hard boundaries: prod vs non-prod, separate billing/chargeback, quota isolation, different policy/RBAC posture, or hitting subscription limits.

**Explanation**: Subscriptions are the main scale unit in landing zones. Quotas (vCPU, Service Bus namespaces per sub, etc.) and throttling of ARM (per-subscription/tenant request limits) are subscription-scoped, so noisy workloads should not share one. RG-level design: group by lifecycle and ownership, not by resource type.

**Code / CLI / Bicep**
```bicep
targetScope = 'subscription'
param location string = 'westeurope'
resource rg 'Microsoft.Resources/resourceGroups@2022-09-01' = {
  name: 'rg-orders-prod-weu'
  location: location
  tags: { workload: 'orders', env: 'prod' }
}
```

### 1.4 [Hard] What happens when you delete a resource group, and what are the pitfalls of moving resources between RGs/subscriptions?
**Answer**: Deleting an RG deletes every resource in it (asynchronously, unless locked with a delete lock, which blocks it). Moving resources changes the resource ID but not the location; not all resource types support move, and role assignments/dependencies may break.

**Explanation**: During a move both source and target RGs are locked for writes; the resource ID changes, so anything keyed by ID (RBAC assignments scoped to the resource, diagnostic settings targets, ARM templates, UAMI references) must be re-checked. Role assignments for SAMI are tied to the principal ID and are lost if the resource is deleted and recreated (new principalId). Key Vault soft-delete and purge protection keep deleted vault names reserved. Verify supported move operations in current docs ("Move support for resources").

**Code / CLI / Bicep**
```bash
az resource invoke-action --action validateMoveResources \
  --ids /subscriptions/<sub>/resourceGroups/rg-src \
  --request-body '{"resources":["<resource-id>"],"targetResourceGroup":"/subscriptions/<sub>/resourceGroups/rg-dst"}'
az resource move --destination-group rg-dst --ids <resource-id>
az lock create --name nodelete --lock-type CanNotDelete -g rg-orders-prod
```

**Gotchas / follow-ups**
- Soft-deleted Key Vaults/APIM/App Configuration keep names reserved - redeploy can fail with a "name in use/soft-deleted" error.
- Delete lock on an RG also blocks deleting resources inside, including autoscale/cleanup tasks that need delete.
- Follow-up: "How do you recover an accidentally deleted RG?" - you generally can't; rely on IaC redeploy + soft-delete/backups.

**Interview traps**
- Saying a subscription "belongs to" a resource group (reverse).
- Thinking RGs are a security boundary on their own - it's RBAC at that scope.
- Forgetting a resource's location can differ from its RG's location.
- Forgetting tenant != subscription.

**Cheat sheet**
- Scopes: MG > Subscription > RG > Resource; inheritance downward.
- Subscription = billing, quota, throttling boundary.
- RG = lifecycle container; one RG per resource.
- Tenant = Entra identity boundary.
- Deleting RG deletes contents; use CanNotDelete locks on prod.

---

## Topic 2: ARM vs Bicep vs Terraform

**Study notes**

| Aspect | ARM JSON | Bicep | Terraform |
|---|---|---|---|
| Language | JSON | DSL, transpiles to ARM | HCL |
| State | None (Azure is source of truth) | None | State file (remote backend) |
| Day-0 new resource support | Immediate | Immediate (same RPs) | Via azurerm/azapi provider; may lag |
| Preview | `what-if` | `what-if` | `plan` |
| Multi-cloud | No | No | Yes |
| Modules | Linked/nested templates | Modules (+ registry) | Modules |

- Deployment modes: **Incremental** (default) leaves unlisted resources; **Complete** deletes resources in the RG not in the template.
- Deployment scopes: RG, subscription, management group, tenant.

### 2.1 [Easy] What are ARM templates and how does Bicep relate?
**Answer**: ARM templates are declarative JSON that Azure Resource Manager executes to create resources idempotently. Bicep is a cleaner DSL that compiles to ARM JSON; it has first-class support for all resource types and API versions.

**Explanation**: ARM is the single control-plane API all tools (portal, CLI, SDK, Terraform azurerm) ultimately call. Bicep has no state and no separate runtime - `az deployment` compiles and submits it.

**Code / CLI / Bicep**
```bicep
param location string = resourceGroup().location
param name string

resource sa 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: name
  location: location
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
  properties: {
    minimumTlsVersion: 'TLS1_2'
    allowBlobPublicAccess: false
  }
}
output id string = sa.id
```
```bash
az deployment group create -g rg-demo -f main.bicep -p name=stdemo001
```

### 2.2 [Follow-up] What does `what-if` do and what are deployment modes?
**Answer**: `what-if` previews which resources would be created, modified or deleted without deploying. Incremental mode only adds/updates; Complete mode deletes anything in the RG not in the template.

**Explanation**: what-if is a best-effort diff with some noise (readonly properties) - not a guarantee. Complete mode is dangerous with shared RGs. Use `--mode Complete` only with isolated RGs.

**Code / CLI / Bicep**
```bash
az deployment group what-if -g rg-demo -f main.bicep -p name=stdemo001
az deployment group create -g rg-demo -f main.bicep --mode Incremental
```

### 2.3 [Medium] Bicep or Terraform - how do you choose?
**Answer**: Bicep for Azure-only shops wanting day-0 resource support, no state management and native tooling. Terraform when you need multi-cloud/multi-provider (e.g. Azure + Datadog + GitHub), an existing Terraform skill base, or its plan/state workflow.

**Explanation**: Terraform needs secure remote state (e.g. Azure Storage backend with locking) and handles drift via refresh/plan. Bicep relies on ARM's idempotent PUT; there is no drift detection beyond what-if. The azapi provider gives Terraform day-0 access to new Azure APIs. Bicep also has deployment stacks (verify GA status/features in current docs) for lifecycle management.

**Code / CLI / Bicep**
```bash
terraform init && terraform plan -out tfplan && terraform apply tfplan
```
```bash
az bicep install && az bicep build -f main.bicep
az bicep decompile -f template.json
```

### 2.4 [Hard] How do you make Bicep deployments safe and repeatable in CI/CD, including secrets and cross-scope deployments?
**Answer**: Use parameter files per environment, never secrets as plain parameters (use `@secure()` or Key Vault references), run what-if on PRs, deploy with a federated workload identity, and use modules with explicit `scope`. Pin API versions and module versions.

**Explanation**: Key Vault references in `.bicepparam`/parameter JSON resolve at deploy time (needs the deployer permission on the vault and "enabled for template deployment" for the legacy method). Cross-scope: a subscription-scope file deploys modules to RG scope via `scope: resourceGroup(...)`. Role assignment names must be deterministic GUIDs (`guid(...)`) for idempotency.

**Code / CLI / Bicep**
```bicep
targetScope = 'subscription'
resource rg 'Microsoft.Resources/resourceGroups@2022-09-01' existing = { name: 'rg-app' }

module app 'app.bicep' = {
  name: 'appDeploy'
  scope: rg
  params: { sqlPassword: kv.getSecret('sqlPassword') }
}
resource kvRg 'Microsoft.Resources/resourceGroups@2022-09-01' existing = { name: 'rg-sec' }
resource kv 'Microsoft.KeyVault/vaults@2023-07-01' existing = {
  name: 'kv-shared'
  scope: kvRg
}

// in app.bicep, role assignment with deterministic name
// name: guid(sa.id, principalId, roleDefinitionId)
```

**Gotchas / follow-ups**
- Random `newGuid()` for role assignments creates duplicates / conflicts on redeploy; use `guid()` of scope+principal+role.
- Role assignment from a freshly created identity may fail due to AAD replication: set `principalType: 'ServicePrincipal'`.
- Output values containing secrets are logged in deployment history - never output secrets.

**Interview traps**
- "Bicep has state" - no.
- Using Complete mode on shared RGs.
- Believing what-if is exact.
- Hard-coded secrets in parameter files.

**Cheat sheet**
- Bicep -> ARM JSON; no state; day-0 support.
- Terraform: state + plan; multi-provider.
- Incremental default; Complete deletes extras.
- Always `what-if` before prod.
- Deterministic `guid()` names for role assignments.

---

## Topic 3: Regions, availability zones, paired regions, SLAs

**Study notes**

```
Geography (data residency)
 └── Region (set of datacenters, e.g. westeurope)
      └── Availability Zones (>=3 physically separate zones in zonal regions)
           └── Datacenters (independent power/cooling/network)
Paired region = Microsoft-defined partner region in same geography
```

- **Zonal** services pin to a zone; **zone-redundant** services replicate across zones automatically (e.g. ZRS storage, zone-redundant Service Bus Premium/standard namespaces in supported regions - verify in current docs).
- Composite SLA = product of component SLAs in a serial chain.
- Paired regions: sequential platform updates, prioritized recovery; GRS storage replicates to the pair. Not all regions have pairs, and DR design should not rely solely on pairing.

### 3.1 [Easy] What is an availability zone?
**Answer**: A physically separate location within an Azure region with independent power, cooling and networking. Deploying across zones protects against a datacenter-level failure inside the region.

**Explanation**: Zone numbers (1,2,3) are logical per subscription and map to different physical zones per subscription. Not all regions or SKUs support zones.

**Code / CLI / Bicep**
```bash
az vm list-skus -l westeurope --zone --query "[?name=='Standard_D2s_v5'].locationInfo[0].zones" -o json
```

### 3.2 [Follow-up] Zonal vs zone-redundant vs regional resources?
**Answer**: Zonal = you pick a zone (VM, disk); you build redundancy yourself. Zone-redundant = platform spreads replicas across zones (ZRS storage, zone-redundant App Service plan, Azure SQL zone-redundant). Regional/non-zonal = no zone guarantee.

**Explanation**: Zone-redundant plans need minimum instance counts (e.g. App Service Plan needs >=2 instances - verify in current docs). Cross-zone traffic can carry small latency and, in some cases, cost.

**Code / CLI / Bicep**
```bicep
resource plan 'Microsoft.Web/serverfarms@2023-01-01' = {
  name: 'asp-prod'
  location: location
  sku: { name: 'P1v3', capacity: 3 }
  properties: { zoneRedundant: true }
}
```

### 3.3 [Medium] How do SLAs compose, and what is a composite SLA?
**Answer**: Serial dependencies multiply: 99.95% x 99.9% = about 99.85%. Redundant (parallel) paths improve availability: 1 - (1-A)(1-B). SLA is a financial commitment, not an uptime guarantee, and has conditions.

**Explanation**: Example: Function app (99.95) -> Service Bus (99.9) -> SQL (99.99) ≈ 99.84%. Adding a second region with Front Door/Traffic Manager lifts availability but raises complexity and data-consistency problems. Free/Basic tiers often have no SLA. Always verify the current SLA numbers at the Azure SLA summary.

**Code / CLI / Bicep**
```csharp
double serial = 0.9995 * 0.999 * 0.9999;       // ~0.9984
double parallel = 1 - (1 - 0.999) * (1 - 0.999); // two independent regions
Console.WriteLine($"{serial:P3} {parallel:P4}");
```

### 3.4 [Hard] Design DR for a Service Bus + Functions workload across regions. What does paired-region really give you?
**Answer**: Use Service Bus geo-replication or geo-disaster recovery (alias) for the namespace, deploy Functions in both regions (secondary idle or active per design), and put an alias/Front Door in front. Pairing only gives platform-level benefits like GRS target and update sequencing - failover is still your design.

**Explanation**: Service Bus Geo-DR (metadata only, active-passive, alias connection string) replicates entity metadata, not messages; the newer Geo-Replication feature (Premium) can replicate data too - verify exact capabilities and failover semantics in current docs. Consumers must tolerate duplicates (at-least-once) after failover. Define RPO/RTO first.

**Code / CLI / Bicep**
```bash
az servicebus georecovery-alias set -g rg-sb --namespace-name sb-primary \
  --alias sb-orders-alias --partner-namespace <secondary-namespace-resource-id>
az servicebus georecovery-alias fail-over -g rg-sb --namespace-name sb-secondary --alias sb-orders-alias
```

**Gotchas / follow-ups**
- Geo-DR failover breaks the pairing; you must re-pair after.
- Messages in flight / unreplicated may be lost or duplicated depending on feature; design idempotent consumers.
- Functions triggers with SB connection must use alias FQDN, and identity-based role assignments must exist on both namespaces.

**Interview traps**
- "Paired region means automatic failover" - false for most services.
- Multiplying SLAs but ignoring that SLA != your achieved availability.
- Assuming all regions have zones.
- Confusing zone-redundant (HA) with geo-redundant (DR).

**Cheat sheet**
- AZ = intra-region HA; Region pair/geo = DR.
- ZRS vs GRS vs GZRS (storage).
- Serial SLA = multiply; parallel = 1-(1-a)(1-b).
- Define RPO/RTO before choosing.
- Verify SLA numbers in current docs.

---

## Topic 4: RBAC vs Azure Policy vs resource locks

**Study notes**

| Tool | Question it answers | Effect |
|---|---|---|
| Azure RBAC | *Who* can do *what* at which scope? | Allow (additive); deny only via deny assignments / `NotActions` subtraction |
| Azure Policy | *What state/config* must resources have? | Audit, Deny, Modify, DeployIfNotExists, Append, etc. |
| Resource locks | *Prevent accidental* delete/modify | `CanNotDelete`, `ReadOnly` - apply to everyone, including Owners |

- RBAC role assignment = security principal + role definition + scope.
- Policy evaluates the resource request/state regardless of who made it; assignments inherit; exemptions exist.
- Locks apply to control-plane operations only; ReadOnly lock can break things that "read" but write behind the scenes (e.g. listing keys on a storage account is a POST).

### 4.1 [Easy] What's the difference between RBAC and Azure Policy?
**Answer**: RBAC controls what an identity is allowed to do; Policy controls what resource configurations are allowed to exist. A Contributor can still be blocked by a Deny policy (e.g. disallowed region or public network access).

**Explanation**: RBAC is about identities and actions; Policy is about resource properties and compliance. They are complementary and evaluated independently.

**Code / CLI / Bicep**
```bash
az role assignment create --assignee <objectId> --role "Storage Blob Data Contributor" \
  --scope /subscriptions/<sub>/resourceGroups/rg-app/providers/Microsoft.Storage/storageAccounts/stapp
az policy assignment list --query "[].{n:displayName,s:scope}" -o table
```

### 4.2 [Follow-up] Explain Policy effects and what DeployIfNotExists / Modify do.
**Answer**: Audit flags non-compliance; Deny blocks the request; Append/Modify change properties on create/update (Modify can add tags using a managed identity); DeployIfNotExists deploys a related resource (e.g. diagnostic settings) if missing. DINE/Modify need a managed identity on the assignment with proper roles, and remediation tasks for existing resources.

**Explanation**: Policy effects on existing resources only mark them non-compliant until a remediation task runs. Start in Audit mode, then move to Deny. Initiatives group multiple policies.

**Code / CLI / Bicep**
```bicep
targetScope = 'subscription'
resource pa 'Microsoft.Authorization/policyAssignments@2022-06-01' = {
  name: 'require-tag-env'
  properties: {
    displayName: 'Require env tag'
    policyDefinitionId: tenantResourceId('Microsoft.Authorization/policyDefinitions','871b6d14-10aa-478d-b590-94f262ecfa99')
    parameters: { tagName: { value: 'env' } }
  }
}
```
(Built-in definition GUID shown is "Require a tag on resources" - verify the ID in current docs.)

### 4.3 [Medium] How does RBAC evaluation work with multiple assignments, and what are deny assignments?
**Answer**: Effective permissions are the union of all role assignments at the scope and parents, minus `NotActions` per role. Deny assignments (created by Blueprints/managed apps, not by users directly) override allows.

**Explanation**: Roles have `Actions`/`NotActions` (control plane) and `DataActions`/`NotDataActions` (data plane). `NotActions` is not a deny; another role can re-grant the action. Limits exist on role assignments per subscription (verify in current docs). Prefer groups over individual assignments, and assign at least privilege scope. Assignments can take minutes to propagate; tokens cache group/role claims for up to ~hour-ish.

**Code / CLI / Bicep**
```bash
az role assignment list --assignee <objectId> --all -o table
az role definition list --name "Contributor" -o json
```

### 4.4 [Hard] A deployment pipeline's identity is Owner of the RG but fails to create a role assignment/delete a resource. Why?
**Answer**: Likely causes: Contributor lacks `Microsoft.Authorization/roleAssignments/write` (Owner or User Access Administrator needed), a Deny policy blocks the action, or a resource lock (CanNotDelete/ReadOnly) is in place. Read the error code: `AuthorizationFailed` vs `RequestDisallowedByPolicy` vs `ScopeLocked`.

**Explanation**: Three layers must all permit. Locks beat Owner. Policy errors name the policy assignment. For least privilege role-assignment delegation use ABAC conditions on role assignments (e.g. restrict which roles can be assigned) - verify in current docs.

**Code / CLI / Bicep**
```bash
az lock list -g rg-app -o table
az policy state list -g rg-app --filter "complianceState eq 'NonCompliant'" -o table
az monitor activity-log list -g rg-app --offset 1h --query "[?status.value=='Failed'].{op:operationName.value,err:properties.statusMessage}" -o json
```

**Gotchas / follow-ups**
- ReadOnly lock on a Function app/storage can block key listing and slot swaps.
- A ReadOnly lock on an RG can break autoscale or Logic App runs that write via ARM.
- Owner on subscription can remove locks - locks are not security against admins, just safeguards.

**Interview traps**
- "Owner can do anything" - not past policy Deny or locks (well, locks removable by Owner, but they block until removed).
- Policy is not authorization; it never grants access.
- Mixing up Actions and DataActions.

**Cheat sheet**
- RBAC = who; Policy = what; Lock = accident guard.
- Error names: AuthorizationFailed / RequestDisallowedByPolicy / ScopeLocked.
- DINE/Modify need managed identity + remediation.
- Assign roles to groups, at narrowest scope.
- Role assignment names = deterministic GUID.

---

## Topic 5: Control plane vs data plane

**Study notes**

```
Control plane (management.azure.com, ARM)        Data plane (service endpoint)
 create/update/delete resources                   read/write the data inside
 RBAC: Actions                                    RBAC: DataActions (or keys/SAS/ACLs)
 e.g. create storage account, namespace           e.g. upload blob, send SB message
 Activity Log                                     Diagnostic logs / service logs
```

- Same identity may have control-plane access but no data-plane access (Owner on a storage account cannot read blobs without a data role - unless it uses shared key access, which Owner can list).
- Key Vault: RBAC permission model vs legacy access policies; secrets access is data plane.
- Service Bus: `Azure Service Bus Data Sender/Receiver/Owner` are data-plane roles.

### 5.1 [Easy] Give examples of control-plane vs data-plane operations.
**Answer**: Control plane: creating a Service Bus namespace or queue via ARM. Data plane: sending/receiving a message, reading a blob, getting a Key Vault secret. They use different endpoints and different permission sets.

**Explanation**: ARM endpoint is management.azure.com; data endpoints are like `<ns>.servicebus.windows.net`, `<acct>.blob.core.windows.net`, `<vault>.vault.azure.net`. Some services (Service Bus queues, Event Hubs) also expose management ops via ARM and data ops over AMQP/HTTPS.

**Code / CLI / Bicep**
```bash
az servicebus queue create -g rg --namespace-name sb --name orders   # control plane
az storage blob upload --account-name stapp -c docs -f a.txt -n a.txt --auth-mode login  # data plane
```

### 5.2 [Follow-up] Why does `Owner` on a storage account give "403 AuthorizationPermissionMismatch" when reading a blob?
**Answer**: Owner only grants control-plane actions. Blob read needs a data-plane role like Storage Blob Data Reader (or key/SAS auth). Assign the data role to the identity.

**Explanation**: With `--auth-mode login`/Entra auth the data role is required. With account keys (if `allowSharedKeyAccess` is true) a principal that can `listKeys` (control plane) effectively gets data access - a privilege escalation path; disable shared key where possible.

**Code / CLI / Bicep**
```bicep
resource blobReader 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(sa.id, principalId, '2a2b9908-6ea1-4ae2-8e65-a410df84e7d1')
  scope: sa
  properties: {
    principalId: principalId
    principalType: 'ServicePrincipal'
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions','2a2b9908-6ea1-4ae2-8e65-a410df84e7d1')
  }
}
```
(GUID is Storage Blob Data Reader - verify in current docs.)

### 5.3 [Medium] How does this affect Managed Identity design for a Function that reads Key Vault secrets and sends to Service Bus?
**Answer**: Give the identity data-plane roles at the narrowest scope: Key Vault Secrets User on the vault (RBAC model) and Azure Service Bus Data Sender on the queue/namespace. Control-plane roles are not needed at runtime.

**Explanation**: Separating planes keeps the runtime identity unable to modify infrastructure. Deployer identity (pipeline) gets control-plane roles; app identity gets data-plane roles. In Functions, identity-based triggers use settings like `ServiceBusConnection__fullyQualifiedNamespace`.

**Code / CLI / Bicep**
```json
{
  "ServiceBusConnection__fullyQualifiedNamespace": "sb-orders.servicebus.windows.net",
  "KeyVaultUri": "https://kv-orders.vault.azure.net/"
}
```
```csharp
var client = new ServiceBusClient("sb-orders.servicebus.windows.net", new DefaultAzureCredential());
var secretClient = new SecretClient(new Uri(kvUri), new DefaultAzureCredential());
```

### 5.4 [Hard] Why can a resource be visible in the portal but its contents are not (and vice versa) - and how does network firewall interplay?
**Answer**: Portal listing uses ARM (control plane), but viewing contents (blobs, secrets, messages) calls the data plane, which needs data-plane RBAC and network access (firewall/private endpoint). Your browser IP may be blocked even with proper roles.

**Explanation**: Private endpoint/firewall restrictions apply to the data plane only; ARM management is still reachable. Data-plane denies show as 403 with network-specific substatus (e.g. `AuthorizationFailure` for storage firewall). Also, RBAC propagation delay for data roles can be several minutes.

**Code / CLI / Bicep**
```bash
az storage account network-rule add -g rg --account-name stapp --ip-address <myip>
az keyvault network-rule add -n kv-orders --ip-address <myip>/32
```

**Gotchas / follow-ups**
- Trusted Microsoft services bypass is limited to specific services/roles - verify per service.
- Key Vault access policy model vs RBAC: assigning RBAC does nothing if vault is on access-policy model.
- Diagnostic logs: control plane = Activity Log; data plane = resource logs via diagnostic settings (must be enabled).

**Interview traps**
- Thinking Contributor can read secrets/blobs.
- Forgetting data-plane logging is opt-in.
- Confusing `Actions` and `DataActions`.

**Cheat sheet**
- Control = ARM; data = service endpoint.
- Data roles: Blob Data *, SB Data *, KV Secrets User.
- Firewall/private endpoint = data plane.
- Activity Log vs resource logs.
- Runtime identity: data roles only.

---

## Topic 6: Tags, naming, cost management and budgets

**Study notes**

- **Tags**: key/value metadata on resources/RGs/subscriptions (limits on count/length - verify in current docs). Not inherited by default; use Policy (Modify / "inherit tag from resource group") to enforce. Used for cost allocation and automation.
- **Naming**: follow CAF convention `<type>-<workload>-<env>-<region>-<instance>`, e.g. `sb-orders-prod-weu-001`. Storage accounts: 3-24 lowercase alphanumeric, globally unique; Key Vault 3-24 chars; per-resource-type rules differ.
- **Cost Management**: cost analysis, budgets (alerts at thresholds via action groups), exports, Advisor recommendations, Reservations/Savings Plans, Azure Hybrid Benefit, dev/test pricing.

### 6.1 [Easy] Why use tags?
**Answer**: Tags label resources for cost allocation, ownership, environment and automation. Typical tags: `env`, `owner`, `costCenter`, `workload`.

**Explanation**: Cost analysis can group/filter by tag. Tags are metadata, not security. Some resources don't emit tags in billing data (verify in current docs), and tags added later apply only to future usage records.

**Code / CLI / Bicep**
```bash
az tag update --resource-id <id> --operation merge --tags env=prod owner=team-orders
az resource list --tag env=prod -o table
```

### 6.2 [Follow-up] How do you enforce tagging and inherit tags?
**Answer**: Azure Policy: Deny for required tags, and Modify (e.g. "Inherit a tag from the resource group if missing") with remediation for existing resources.

**Explanation**: Resource tags aren't inherited from RG automatically. Modify needs a managed identity with Tag Contributor-capable role. Beware Deny on tags blocking ARM deployments that lack tags - roll out in Audit first.

**Code / CLI / Bicep**
```bicep
param tags object = { env: 'prod', owner: 'team-orders', costCenter: 'CC123' }
resource sb 'Microsoft.ServiceBus/namespaces@2022-10-01-preview' = {
  name: 'sb-orders-prod-weu'
  location: location
  sku: { name: 'Standard', tier: 'Standard' }
  tags: tags
}
```

### 6.3 [Medium] How would you set a budget and alert on a spike?
**Answer**: Create a budget at subscription/RG scope with threshold notifications (actual and forecasted) wired to an action group; review Cost Analysis and Advisor, and enable anomaly alerts. Budgets only alert - they do not stop spend unless you automate a response.

**Explanation**: Cost data lags (hours). To stop spend, an action group can trigger an Automation/Logic App/Function that scales down or disables resources. Reservations (1/3-year) reduce steady-state compute; savings plans for flexible compute - verify current offerings.

**Code / CLI / Bicep**
```bicep
resource budget 'Microsoft.Consumption/budgets@2023-05-01' = {
  name: 'budget-orders-monthly'
  properties: {
    category: 'Cost'
    amount: 1000
    timeGrain: 'Monthly'
    timePeriod: { startDate: '2026-11-01' }
    notifications: {
      actual80: {
        enabled: true
        operator: 'GreaterThan'
        threshold: 80
        contactEmails: [ 'team@contoso.com' ]
      }
    }
  }
}
```

### 6.4 [Tricky] Your Service Bus/Functions bill spiked. How do you investigate?
**Answer**: Cost Analysis grouped by resource and meter, then cross-check with metrics: for Functions, execution count/GB-s and retries; for Service Bus, tier (Premium messaging units are fixed cost) and operations; also Log Analytics ingestion and App Insights sampling, which are common hidden costs.

**Explanation**: Typical causes: poison-message retry storms (delivery count loops re-triggering Functions), infinite loops between queue and function, verbose logging ingested into Log Analytics, Premium plan always-ready instances, missing sampling, Redis tier oversizing. Find culprits via cost analysis + metrics + KQL on `traces`/`requests` counts.

**Code / CLI / Bicep**
```kusto
requests
| where timestamp > ago(7d)
| summarize count() by bin(timestamp, 1h), cloud_RoleName
| render timechart
```
```kusto
Usage
| where TimeGenerated > ago(30d)
| summarize GB=sum(Quantity)/1000 by DataType
| order by GB desc
```

**Gotchas / follow-ups**
- Log Analytics `Usage` table Quantity is in MB.
- Daily cap on workspace stops ingestion (loses telemetry) - weigh trade-off.
- Orphaned disks/public IPs/idle App Service plans cost money.

**Interview traps**
- Believing budgets cap spending.
- Assuming tags inherit from RG.
- Forgetting global-uniqueness of storage/Key Vault names.
- Ignoring monitoring ingestion as cost driver.

**Cheat sheet**
- Tags: Policy-enforced, not inherited.
- Naming: CAF pattern; mind length/charset limits.
- Budgets alert; automation acts.
- Reservations/Savings Plans/Hybrid Benefit for savings.
- Check Log Analytics ingestion first on surprise bills.

---

## Topic 7: Well-Architected Framework pillars

**Study notes**

| Pillar | Focus | Example in your stack |
|---|---|---|
| Reliability | Resilience, recovery | Retries, DLQ, zones, idempotent consumers |
| Security | Confidentiality, integrity | Managed identity, Key Vault, private endpoints |
| Cost Optimization | Value for spend | Right-size, consumption plans, reservations |
| Operational Excellence | DevOps, monitoring | IaC, CI/CD, App Insights, runbooks |
| Performance Efficiency | Scale, meet demand | Autoscale, caching with Redis, partitioning |

Tools: Azure Well-Architected Review, Azure Advisor, design principles/trade-offs per workload.

### 7.1 [Easy] Name the five pillars.
**Answer**: Reliability, Security, Cost Optimization, Operational Excellence, Performance Efficiency. They are used to evaluate trade-offs in a workload design.

**Explanation**: Pillars conflict by design: more redundancy raises cost; more security can add latency. Good architecture documents the trade-offs.

**Code / CLI / Bicep**
```bash
az advisor recommendation list --category Cost -o table   # categories: Cost, Security, HighAvailability, Performance, OperationalExcellence
```

### 7.2 [Follow-up] Map a queue-based Functions pipeline to the pillars.
**Answer**: Reliability: Service Bus with DLQ, retries, idempotency. Security: managed identity, no secrets in config. Cost: Consumption/Flex plan scaling to zero. Ops: App Insights + alerts + IaC. Performance: concurrency tuning, prefetch, Redis cache.

**Explanation**: Cite concrete settings: `maxConcurrentCalls`, `maxDeliveryCount`, lock duration, auto-complete behavior. Show that each pillar has a measurable signal.

**Code / CLI / Bicep**
```json
{
  "version": "2.0",
  "extensions": {
    "serviceBus": {
      "maxConcurrentCalls": 16,
      "maxAutoLockRenewalDuration": "00:05:00"
    }
  }
}
```

### 7.3 [Medium] How do you decide between reliability and cost for a workload?
**Answer**: Start from business requirements: RTO/RPO, SLO, and cost of downtime. Tier workloads (critical vs non-critical), apply zone redundancy/multi-region only where justified, and document accepted risks.

**Explanation**: Define SLIs/SLOs and error budgets. Non-critical dev environments use cheaper SKUs without zones. Use failure mode analysis (FMA) for each dependency.

**Code / CLI / Bicep**
```kusto
requests
| where timestamp > ago(30d)
| summarize availability = 100.0 * countif(success == true) / count() by bin(timestamp, 1d)
```

### 7.4 [Tricky] "Your app is Reliable but the audit says insecure" - give concrete examples of pillar tension and resolution.
**Answer**: Example: retries improve reliability but can amplify load and cause throttling (429); public endpoints ease access but violate security; fix with bounded exponential backoff with jitter and private endpoints plus VNet integration. Resolution is explicit trade-off documentation, not maximizing one pillar.

**Explanation**: Private endpoints add DNS complexity and cost; zone redundancy raises price; aggressive logging aids operations but leaks PII and increases cost. A mature answer states the decision, owner, and review trigger.

**Code / CLI / Bicep**
```bicep
resource pe 'Microsoft.Network/privateEndpoints@2023-05-01' = {
  name: 'pe-sb'
  location: location
  properties: {
    subnet: { id: subnetId }
    privateLinkServiceConnections: [{
      name: 'sb'
      properties: { privateLinkServiceId: sbNamespaceId, groupIds: [ 'namespace' ] }
    }]
  }
}
```

**Gotchas / follow-ups**
- Service Bus private endpoints require Premium SKU (verify in current docs).
- Need Private DNS zone `privatelink.servicebus.windows.net` linked to VNet.

**Interview traps**
- Reciting pillars without any trade-off examples.
- Not mentioning measurable targets.
- Treating WAF as a one-time checklist.

**Cheat sheet**
- 5 pillars: Rel, Sec, Cost, OpEx, Perf.
- Trade-offs must be explicit.
- Use Advisor + WAF Review.
- SLO/RTO/RPO drive design.
- Apply tiering: critical vs non-critical.

---

## Topic 8: Choosing compute (App Service vs Functions vs Container Apps vs AKS vs VMs)

**Study notes**

| Option | Model | Scale | Best for | Ops burden |
|---|---|---|---|---|
| App Service | PaaS web hosting | Plan-based autoscale | Web apps/APIs, always-on | Low |
| Functions | Serverless/event-driven | Event-driven (scale to zero on Consumption/Flex) | Triggers: SB, Blob, Timer, HTTP | Low |
| Container Apps | Serverless containers (Kubernetes under the hood, KEDA, Dapr) | HTTP/event/KEDA, scale to zero | Microservices, background workers | Low-Med |
| AKS | Managed Kubernetes | Pods/nodes, full control | Complex platforms, custom networking/operators | High |
| VMs | IaaS | VMSS autoscale | Legacy, custom OS, licensing | Highest |

Decision criteria: control needed, event-driven vs steady, startup/cold-start tolerance, execution time limits, networking (VNet), team Kubernetes skill, cost model, portability (containers), compliance.

### 8.1 [Easy] When would you pick Functions over App Service?
**Answer**: Functions for event-driven, short-lived, bursty work with bindings/triggers (Service Bus, Blob, Timer) and pay-per-use. App Service for always-on web APIs with steady traffic and predictable cost.

**Explanation**: Functions can also run on App Service plans or Premium/Flex plans to avoid cold start and gain VNet integration. Consumption plan has execution timeout limits (verify in current docs; Premium/Dedicated allow longer).

**Code / CLI / Bicep**
```csharp
public class OrderFunction(ILogger<OrderFunction> log)
{
    [Function("ProcessOrder")]
    public async Task Run(
        [ServiceBusTrigger("orders", Connection = "ServiceBusConnection")] ServiceBusReceivedMessage msg,
        ServiceBusMessageActions actions)
    {
        log.LogInformation("Msg {Id}", msg.MessageId);
        await actions.CompleteMessageAsync(msg);
    }
}
```
(Isolated worker model, `Microsoft.Azure.Functions.Worker.Extensions.ServiceBus`.)

### 8.2 [Follow-up] Container Apps vs AKS - how to choose?
**Answer**: Container Apps when you want containers, scale-to-zero, KEDA scaling and Dapr without managing Kubernetes. AKS when you need Kubernetes API access, custom operators/CRDs, node-level control or advanced networking/multi-tenant platform.

**Explanation**: Container Apps hides the cluster; you cannot use kubectl on it. Container Apps supports jobs for batch work. AKS means managing upgrades, node pools, ingress, policy and security posture (managed to varying degrees).

**Code / CLI / Bicep**
```bash
az containerapp create -n orders-worker -g rg --environment cae-prod \
  --image myacr.azurecr.io/orders:1.0 \
  --min-replicas 0 --max-replicas 10 \
  --scale-rule-name sb --scale-rule-type azure-servicebus \
  --scale-rule-metadata queueName=orders messageCount=50 namespace=sb-orders \
  --scale-rule-auth connection=sb-conn-secret
```
(Verify exact scale-rule flags in current docs; identity-based scale rules also supported.)

### 8.3 [Medium] Functions hosting plans: how do they differ for a Service Bus workload?
**Answer**: Consumption: scale to zero, cold starts, limited VNet. Flex Consumption: scale-to-zero with VNet and per-instance concurrency (verify current availability). Premium: pre-warmed instances, VNet, no cold start, longer runtimes. Dedicated: runs on App Service plan.

**Explanation**: Service Bus-triggered functions scale based on queue length (scale controller); on Consumption/Flex it is runtime-driven, while on Premium it uses runtime scale monitoring if enabled for non-HTTP triggers with VNet-restricted resources. Each instance processes `maxConcurrentCalls` messages concurrently - total concurrency = instances x concurrency; protect downstream.

**Code / CLI / Bicep**
```bash
az functionapp plan create -g rg -n plan-prem --sku EP1 --min-instances 1
az functionapp config set -g rg -n func-orders --generic-configurations '{"functionAppScaleLimit": 10}'
```
(`functionAppScaleLimit` - verify exact switch for your plan in current docs.)

### 8.4 [Hard] You must run a long-running, stateful, GPU or Windows-legacy workload plus event-driven glue. What's your architecture?
**Answer**: Run the heavy/legacy part on VMs/VMSS (or AKS with GPU node pools if containerized), and keep event glue on Functions/Container Apps consuming Service Bus, using managed identities. Match each component to the least-operational option that meets constraints.

**Explanation**: Constraints (OS dependency, GPU, license, long-running) push to IaaS/AKS; everything else stays serverless. Use queues for decoupling and load levelling; use Durable Functions or Logic Apps for orchestration where appropriate.

**Code / CLI / Bicep**
```bicep
resource vmss 'Microsoft.Compute/virtualMachineScaleSets@2023-09-01' = {
  name: 'vmss-legacy'
  location: location
  sku: { name: 'Standard_D4s_v5', capacity: 2 }
  zones: [ '1', '2', '3' ]
  identity: { type: 'SystemAssigned' }
  properties: { orchestrationMode: 'Flexible', platformFaultDomainCount: 1 }
}
```

**Gotchas / follow-ups**
- Consumption function timeouts and memory caps (verify in current docs).
- AKS adds operational cost even when idle (node pools).
- Container Apps limits on resource sizes/replicas (verify).

**Interview traps**
- Picking AKS "because containers".
- Forgetting Functions cold start/VNet limits per plan.
- Ignoring downstream throttling when scaling out consumers.

**Cheat sheet**
- Web API steady: App Service. Event glue: Functions. Containers w/o K8s: Container Apps. K8s control: AKS. Full control/legacy: VMs.
- Premium/Flex for VNet + no cold start.
- Scale-out concurrency = instances x per-instance concurrency.
- Choose least operational overhead that meets constraints.

---

## Topic 9: PaaS vs IaaS vs serverless

**Study notes**

```
You manage ─────────────────────────────────────────────►  Provider manages
IaaS (VM):        app | runtime | OS | patching | scaling | network   (HW by Azure)
PaaS (App Svc):   app | config                      (OS, runtime patching, platform by Azure)
Serverless (Func):code only; scale-to-zero, per-execution billing
SaaS:             just use it (M365)
```

- Shared responsibility shifts toward Azure as you go up the stack; you always own data, identities, and access.
- Serverless traits: event-driven, auto scale, consumption billing, stateless, execution limits.

### 9.1 [Easy] Difference between IaaS, PaaS and serverless?
**Answer**: IaaS gives you virtual infrastructure (VMs) where you manage OS and up; PaaS manages OS/runtime so you deploy code; serverless abstracts servers entirely with event-driven, per-use scaling and billing. Each moves operational responsibility to Azure.

**Explanation**: Examples: VM (IaaS), App Service/SQL Database (PaaS), Functions/Logic Apps consumption (serverless). The boundaries are fuzzy - Container Apps and Functions Premium blur them.

**Code / CLI / Bicep**
```bash
az vm create -g rg -n vm1 --image Ubuntu2204 --assign-identity
az webapp create -g rg -p plan -n app-orders --runtime "DOTNETCORE:8.0"
az functionapp create -g rg -n func-orders --storage-account stfunc --flexconsumption-location westeurope --runtime dotnet-isolated --runtime-version 8.0
```
(Flex Consumption creation flags: verify in current docs.)

### 9.2 [Follow-up] Who is responsible for what under shared responsibility?
**Answer**: Customer always owns data, identities/access and endpoints config; Azure owns physical, host and (in PaaS) OS patching. IaaS pushes OS patching, firewall and runtime to the customer.

**Explanation**: Example: With Key Vault (PaaS) you own access policies/RBAC and secret rotation; Azure owns HSM hardware and patching. Misconfigured public access to Blob is customer's failure.

**Code / CLI / Bicep**
```bicep
resource sa 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: saName
  location: location
  sku: { name: 'Standard_ZRS' }
  kind: 'StorageV2'
  properties: {
    allowBlobPublicAccess: false
    allowSharedKeyAccess: false
    supportsHttpsTrafficOnly: true
  }
}
```

### 9.3 [Medium] What are the trade-offs of serverless?
**Answer**: Pros: elastic scale, no idle cost, low ops. Cons: cold starts, execution limits, less control over networking/runtime, cost unpredictability at high sustained volume, debugging distributed complexity, and downstream overload from rapid scale-out.

**Explanation**: At steady high throughput a dedicated plan/containers can be cheaper than consumption. Statelessness requires external state (Storage, Redis, Cosmos); Durable Functions provides orchestration state.

**Code / CLI / Bicep**
```csharp
// Idempotent consumer: serverless instances may process a message more than once
if (await cache.StringSetAsync($"done:{msg.MessageId}", "1", TimeSpan.FromHours(24), When.NotExists))
{
    await HandleAsync(msg);
}
```
(StackExchange.Redis `IDatabase.StringSetAsync` with `When.NotExists`.)

### 9.4 [Tricky] Is Logic Apps Consumption "serverless" and how does it differ from Standard?
**Answer**: Consumption is multi-tenant, pay-per-action serverless. Standard runs on the Logic Apps runtime hosted like App Service/Functions (workflow service plan), supports VNet integration, local development, multiple workflows per app and fixed-cost hosting.

**Explanation**: Choose by networking/isolation needs, cost model at volume (per-action vs plan), and tooling (VS Code, CI/CD for Standard). Connector pricing and behaviors differ - verify in current docs.

**Code / CLI / Bicep**
```bicep
resource logic 'Microsoft.Logic/workflows@2019-05-01' = {
  name: 'logic-orders'
  location: location
  identity: { type: 'SystemAssigned' }
  properties: { definition: { '$schema': 'https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#', contentVersion: '1.0.0.0', triggers: {}, actions: {} } }
}
```

**Gotchas / follow-ups**
- At-least-once delivery means idempotency is needed in every serverless consumer.
- Scale-out can overwhelm SQL/Redis - cap concurrency.

**Interview traps**
- Saying serverless has "no servers".
- Equating PaaS with serverless.
- Ignoring that data/identity remain the customer's responsibility.

**Cheat sheet**
- IaaS: you patch OS. PaaS: you deploy code. Serverless: you deploy functions; billed per execution.
- Customer always owns data + identity.
- Serverless needs idempotency and concurrency caps.
- Cost crossover: steady high load favors dedicated.

---

## Topic 10: Azure Resource Graph and Azure CLI / PowerShell basics

**Study notes**

- **Azure Resource Graph (ARG)**: fast, cross-subscription query over resource metadata using KQL subset. Tables: `resources`, `resourcecontainers`, `advisorresources`, `policyresources`, etc. Eventually consistent (seconds-minutes lag).
- **Azure CLI** (`az`): cross-platform, JSON default, `--query` uses JMESPath, `-o table|json|tsv`. **Az PowerShell**: object pipeline cmdlets (`Get-AzResource`).
- Auth: `az login` (user), `az login --service-principal`, `az login --identity` (managed identity), `--use-device-code`.

### 10.1 [Easy] What is Azure Resource Graph used for?
**Answer**: Querying resources at scale across subscriptions and management groups quickly, e.g. inventory, compliance and tag audits. It is read-only and uses Kusto-like syntax.

**Explanation**: Unlike `az resource list` which pages ARM per subscription, ARG indexes resources. It holds resource properties, not data-plane content.

**Code / CLI / Bicep**
```kusto
resources
| where type =~ 'microsoft.servicebus/namespaces'
| project name, resourceGroup, location, sku=tostring(sku.name), tags
| order by name asc
```
```bash
az graph query -q "resources | summarize count() by type | top 10 by count_" 
```
(Requires `az extension add --name resource-graph` on older CLI versions.)

### 10.2 [Follow-up] JMESPath in CLI vs KQL in ARG - give examples.
**Answer**: CLI `--query` filters/reshapes the JSON client-side (JMESPath); ARG executes KQL server-side. Use ARG for tenant-wide questions, JMESPath for shaping a single command's output.

**Explanation**: JMESPath strings with `==` need backtick or quotes for literals. In PowerShell, quoting differs from bash - a classic gotcha.

**Code / CLI / Bicep**
```bash
az functionapp list --query "[?state=='Running'].{name:name,rg:resourceGroup}" -o table
```
```powershell
az functionapp list --query "[?state=='Running'].{name:name,rg:resourceGroup}" -o table
Get-AzFunctionApp | Where-Object Status -eq 'Running' | Select-Object Name, ResourceGroupName
```

### 10.3 [Medium] Find resources missing a tag or with public network access, across all subscriptions.
**Answer**: Use ARG with `isnull(tags['env'])` or property filters like `properties.publicNetworkAccess`. Results can be exported for remediation or used in dashboards/workbooks.

**Explanation**: ARG returns max rows per query (default 100, max 1000 per page; use `--first`/skip tokens - verify in current docs). Property availability depends on the resource type's API version surfaced.

**Code / CLI / Bicep**
```kusto
resources
| where isnull(tags['env']) or tags['env'] == ''
| project id, type, resourceGroup, subscriptionId
```
```kusto
resources
| where type =~ 'microsoft.storage/storageaccounts'
| where properties.publicNetworkAccess =~ 'Enabled' or properties.allowBlobPublicAccess == true
| project name, resourceGroup, subscriptionId
```

### 10.4 [Tricky] Your script works locally but fails in a pipeline/managed-identity context with 'Please run az login'. Why, and how do you script robustly?
**Answer**: Pipelines don't have your interactive session. Use a service connection/federated credential (AzureCLI task), or `az login --identity` on hosts with a managed identity. In code, use `DefaultAzureCredential` rather than shelling out to CLI.

**Explanation**: `DefaultAzureCredential` tries environment, workload identity, managed identity, Visual Studio/CLI/PowerShell etc. in order - convenient locally but ambiguous in prod; prefer a specific credential (`ManagedIdentityCredential`) in production. Always set `--subscription` explicitly in scripts to avoid wrong-subscription mistakes.

**Code / CLI / Bicep**
```bash
az login --identity                      # SAMI
az login --identity --username <uami-client-id>   # UAMI (verify flag in current docs)
az account set --subscription <sub-id>
az account get-access-token --resource https://servicebus.azure.net
```

**Gotchas / follow-ups**
- ARG results lag newly created resources.
- ARG is not for data-plane info (blob counts, messages).
- CLI output includes secrets for some commands - avoid logging.

**Interview traps**
- Using `az resource list` for huge estates (slow).
- Relying on the CLI's default subscription.
- Quote handling in PowerShell vs bash.

**Cheat sheet**
- ARG: `resources | where type =~ '...'`.
- `az graph query`, `Search-AzGraph`.
- JMESPath client-side, KQL server-side.
- `az login --identity` for MI.
- `-o table|tsv|json`; always `az account set`.

---

## Topic 11: Azure SDK for .NET fundamentals

**Study notes**

- Packages `Azure.*` (modern, e.g. `Azure.Messaging.ServiceBus`, `Azure.Storage.Blobs`, `Azure.Security.KeyVault.Secrets`, `Azure.Identity`) built on `Azure.Core`. Legacy `Microsoft.Azure.*` / `WindowsAzure.*` are older; prefer `Azure.*`.
- **Client pattern**: service clients are thread-safe, designed as singletons; options classes (`ClientOptions`) configure retry, diagnostics, transport. Operations return `Response<T>`/`Pageable<T>`/`AsyncPageable<T>`/`Operation<T>` (LRO).
- **Pipeline**: request -> policies (telemetry, retry, auth, logging) -> transport (HttpClient) -> response. Custom `HttpPipelinePolicy` can be added.
- Errors: `RequestFailedException` (Status, ErrorCode); Service Bus uses `ServiceBusException` with `Reason`.
- Management SDK: `Azure.ResourceManager.*` (ArmClient).

### 11.1 [Easy] How do you create and authenticate a client?
**Answer**: Instantiate the client with an endpoint and a `TokenCredential`, typically `DefaultAzureCredential`. Create it once and reuse (singleton), since clients are thread-safe and hold connections.

**Explanation**: Avoid connection strings/keys where managed identity works. `DefaultAzureCredential` caches tokens and chains credential sources.

**Code / CLI / Bicep**
```csharp
using Azure.Identity;
using Azure.Storage.Blobs;

var blobService = new BlobServiceClient(
    new Uri("https://stapp.blob.core.windows.net"),
    new DefaultAzureCredential());

var container = blobService.GetBlobContainerClient("docs");
await container.CreateIfNotExistsAsync();
```

### 11.2 [Follow-up] How do you configure retries and timeouts?
**Answer**: Through `ClientOptions.Retry` (`RetryOptions`): `Mode` (Exponential/Fixed), `MaxRetries`, `Delay`, `MaxDelay`, `NetworkTimeout`. Service Bus has its own `ServiceBusRetryOptions` on `ServiceBusClientOptions.RetryOptions`.

**Explanation**: Defaults: 3 retries, exponential, ~0.8s delay (verify in current docs). Retries apply to transient status codes (408, 429, 5xx etc.) and honor `Retry-After`. Don't stack your own retries (e.g. Polly) on top blindly - multiplicative retries.

**Code / CLI / Bicep**
```csharp
var options = new BlobClientOptions
{
    Retry =
    {
        Mode = Azure.Core.RetryMode.Exponential,
        MaxRetries = 5,
        Delay = TimeSpan.FromSeconds(0.5),
        MaxDelay = TimeSpan.FromSeconds(30),
        NetworkTimeout = TimeSpan.FromSeconds(100)
    }
};
var svc = new BlobServiceClient(new Uri(url), new DefaultAzureCredential(), options);

var sb = new ServiceBusClient(fqdn, new DefaultAzureCredential(), new ServiceBusClientOptions
{
    RetryOptions = new ServiceBusRetryOptions { MaxRetries = 5, Mode = ServiceBusRetryMode.Exponential }
});
```

### 11.3 [Medium] How do you register clients with DI (AddAzureClients) and mock them in tests?
**Answer**: `Microsoft.Extensions.Azure`'s `AddAzureClients` registers clients as singletons, shares one credential, and binds config; for tests, mock via the protected parameterless constructor and virtual methods (Moq/NSubstitute), or use `*Model` factories (e.g. `BlobsModelFactory`) to build response models.

**Explanation**: SDK clients have public virtual methods and protected constructors for mocking. `Response.FromValue(value, response)` builds responses. For `Pageable`, use `AsyncPageable<T>.FromPages`.  Prefer wrapping in your own interface for domain logic.

**Code / CLI / Bicep**
```csharp
builder.Services.AddAzureClients(b =>
{
    b.AddServiceBusClientWithNamespace("sb-orders.servicebus.windows.net");
    b.AddBlobServiceClient(new Uri("https://stapp.blob.core.windows.net"));
    b.AddSecretClient(new Uri("https://kv-orders.vault.azure.net/"));
    b.UseCredential(new DefaultAzureCredential());
    b.ConfigureDefaults(o => o.Retry.MaxRetries = 4);
    b.AddClient<ServiceBusSender, ServiceBusClientOptions>((_, _, sp) =>
        sp.GetRequiredService<ServiceBusClient>().CreateSender("orders"))
     .WithName("orders");
});
```
```csharp
// Mock test
var mock = new Mock<BlobClient>();
mock.Setup(c => c.ExistsAsync(It.IsAny<CancellationToken>()))
    .ReturnsAsync(Response.FromValue(true, Mock.Of<Response>()));
```
(`AddServiceBusClientWithNamespace` and `UseCredential` exist in recent Microsoft.Extensions.Azure/Azure.Messaging.ServiceBus.Extensions - verify in current package docs.)

### 11.4 [Hard] How does the pipeline work, how do you add custom policies, and how do you handle LROs, paging and cancellation?
**Answer**: Requests pass through policies (per-call, then retry, then per-retry) before transport; add `HttpPipelinePolicy` via `options.AddPolicy(policy, HttpPipelinePosition.PerCall|PerRetry)`. LROs return `Operation<T>` with `WaitForCompletionAsync`; paging via `await foreach` over `AsyncPageable<T>`; always pass `CancellationToken`.

**Explanation**: PerRetry policies run on every attempt (good for logging each try or adding headers); PerCall once. Diagnostics: `AzureEventSourceListener` logs requests/responses; ActivitySource enables distributed tracing with OpenTelemetry/App Insights (`Azure.Experimental.EnableActivitySource` switch in older SDKs - verify in current docs).

**Code / CLI / Bicep**
```csharp
public sealed class CorrelationPolicy : HttpPipelinePolicy
{
    public override void Process(HttpMessage m, ReadOnlyMemory<HttpPipelinePolicy> p) { Add(m); ProcessNext(m, p); }
    public override async ValueTask ProcessAsync(HttpMessage m, ReadOnlyMemory<HttpPipelinePolicy> p) { Add(m); await ProcessNextAsync(m, p); }
    static void Add(HttpMessage m) => m.Request.Headers.SetValue("x-ms-client-request-id", Guid.NewGuid().ToString());
}
options.AddPolicy(new CorrelationPolicy(), HttpPipelinePosition.PerCall);

using var listener = AzureEventSourceListener.CreateConsoleLogger();

await foreach (var item in container.GetBlobsAsync(cancellationToken: ct)) { /* ... */ }
```

**Gotchas / follow-ups**
- Creating a new client per request leaks sockets/credentials and loses token cache.
- `DefaultAzureCredential` in prod can be slow due to probing chain - restrict with `ExcludeXxxCredential` or use `ManagedIdentityCredential`.
- UAMI: pass `ManagedIdentityId`/`ClientId` (`new DefaultAzureCredential(new DefaultAzureCredentialOptions { ManagedIdentityClientId = id })`).
- `ServiceBusClient` should be singleton; senders/processors long-lived; dispose on shutdown.

**Interview traps**
- New client per call.
- Double retry layers.
- Mocking non-virtual members.
- Using legacy `Microsoft.Azure.ServiceBus`.

**Cheat sheet**
- `Azure.*` + `Azure.Identity` + `DefaultAzureCredential`/`ManagedIdentityCredential`.
- Clients: singleton, thread-safe.
- `ClientOptions.Retry` / `ServiceBusRetryOptions`.
- `AddAzureClients` for DI + shared credential.
- Mock via protected ctor + `*ModelFactory`.
- `RequestFailedException.Status/ErrorCode`.

---

## Topic 12: Resiliency and transient faults in Azure SDKs

**Study notes**

- **Transient faults**: timeouts, 408, 429, 500/503/504, connection resets, `ServiceBusException.IsTransient`, throttling.
- Pattern set: retry with exponential backoff + jitter, circuit breaker, timeout, bulkhead, idempotency, dead-letter, fallback.
- Azure SDK clients already retry; layer additional resilience (Polly / `Microsoft.Extensions.Resilience`) around *your* HTTP calls or business operations, not blindly around SDK calls.
- Service Bus: at-least-once delivery -> consumers must be idempotent; lock expiry causes redelivery; `MaxDeliveryCount` -> DLQ.

### 12.1 [Easy] What is a transient fault and how should you handle it?
**Answer**: A temporary failure that succeeds if retried (throttling, brief network/service blip). Handle with bounded retries using exponential backoff and jitter; permanent errors (400, 401, 403, 404) should not be retried.

**Explanation**: Azure SDK retry policy classifies transient responses automatically. Idempotent operations are safe to retry; non-idempotent ones need idempotency keys or dedup.

**Code / CLI / Bicep**
```csharp
try { await sender.SendMessageAsync(msg, ct); }
catch (ServiceBusException ex) when (ex.IsTransient)
{
    // SDK already retried per ServiceBusRetryOptions; reaching here means retries exhausted
    log.LogWarning(ex, "Transient failure persisted: {Reason}", ex.Reason);
    throw;
}
```

### 12.2 [Follow-up] Where does SDK retry end and Polly begin?
**Answer**: SDK retry covers individual HTTP/AMQP operations. Use Polly/resilience pipelines for composed business operations, non-Azure HTTP dependencies, circuit breaking and fallbacks; avoid wrapping SDK calls in additional retry without lowering SDK `MaxRetries`.

**Explanation**: Nested retries multiply: 3 SDK x 3 Polly = 9 attempts plus delays - can worsen throttling. Set one layer authoritative.

**Code / CLI / Bicep**
```csharp
builder.Services.AddHttpClient("partner")
    .AddStandardResilienceHandler(); // Microsoft.Extensions.Http.Resilience - verify package/options in current docs
```
```csharp
var pipeline = new ResiliencePipelineBuilder()
    .AddRetry(new RetryStrategyOptions { MaxRetryAttempts = 3, BackoffType = DelayBackoffType.Exponential, UseJitter = true })
    .AddTimeout(TimeSpan.FromSeconds(10))
    .Build();   // Polly v8
```

### 12.3 [Medium] How do you build a resilient Service Bus consumer?
**Answer**: Use `ServiceBusProcessor` (or Functions trigger) with proper `MaxConcurrentCalls`, lock renewal, explicit settle (Complete/Abandon/DeadLetter), idempotent handlers, and DLQ monitoring. Distinguish transient errors (abandon -> retry) from poison (dead-letter with reason).

**Explanation**: If processing exceeds lock duration the message is redelivered and `CompleteMessageAsync` fails with `MessageLockLost`. `MaxAutoLockRenewalDuration` extends the lock. Delivery count increments on each abandon/lock-loss; at `MaxDeliveryCount` it goes to DLQ.

**Code / CLI / Bicep**
```csharp
var processor = client.CreateProcessor("orders", new ServiceBusProcessorOptions
{
    MaxConcurrentCalls = 8,
    AutoCompleteMessages = false,
    MaxAutoLockRenewalDuration = TimeSpan.FromMinutes(5)
});
processor.ProcessMessageAsync += async a =>
{
    try { await HandleAsync(a.Message, a.CancellationToken); await a.CompleteMessageAsync(a.Message); }
    catch (BusinessValidationException ex)
    { await a.DeadLetterMessageAsync(a.Message, "ValidationFailed", ex.Message); }
};
processor.ProcessErrorAsync += e => { log.LogError(e.Exception, "SB error {Source}", e.ErrorSource); return Task.CompletedTask; };
await processor.StartProcessingAsync();
```

### 12.4 [Hard] Retry storms: how do you prevent a failing dependency from being hammered and exhausting downstream capacity?
**Answer**: Combine exponential backoff with jitter, a circuit breaker, bounded concurrency, and queue-based load levelling; on repeated failure, stop consuming (pause processor) rather than abandon-loop. Honor `Retry-After` for 429.

**Explanation**: Abandon-loops with Service Bus burn delivery count and Functions executions fast. Better: schedule redelivery (`ScheduleMessageAsync` clone) or defer. Functions Service Bus extension settings cap concurrency; dynamic concurrency may be available (verify in current docs).

**Code / CLI / Bicep**
```csharp
var breaker = new ResiliencePipelineBuilder()
    .AddCircuitBreaker(new CircuitBreakerStrategyOptions
    {
        FailureRatio = 0.5, MinimumThroughput = 10,
        SamplingDuration = TimeSpan.FromSeconds(30), BreakDuration = TimeSpan.FromSeconds(30)
    }).Build();

// When open: pause consumption
catch (BrokenCircuitException) { await processor.StopProcessingAsync(); /* restart after BreakDuration */ }
```

**Gotchas / follow-ups**
- Retrying non-idempotent POST can duplicate side effects.
- Timeouts shorter than the downstream p99 produce self-inflicted failures.
- Lock-lost + duplicate processing: idempotency store (Redis `SET NX` with TTL, or DB unique key).
- Duplicate detection on Service Bus (`MessageId` + time window) helps on send side only.

**Interview traps**
- "Retry everything."
- Retrying 4xx.
- Forgetting jitter (thundering herd).
- Treating DLQ as a bin nobody reads.

**Cheat sheet**
- Transient = retry; permanent = fail/DLQ.
- Backoff + jitter + max attempts.
- One authoritative retry layer.
- Idempotent consumers; at-least-once.
- Circuit breaker + bounded concurrency + DLQ alerts.

---

## Topic 13: Azure Landing Zone concepts

**Study notes**

```
Tenant Root Group
 └── Org MG
      ├── Platform MG
      │    ├── Identity MG     (identity subs)
      │    ├── Management MG   (monitoring, automation)
      │    └── Connectivity MG (hub VNet, firewall, DNS)
      ├── Landing Zones MG
      │    ├── Corp MG   (private, connected workloads)
      │    └── Online MG (internet-facing workloads)
      ├── Sandbox MG
      └── Decommissioned MG
```

- Cloud Adoption Framework (CAF) **Azure landing zone** = pre-built environment: management groups, policies, networking (hub-spoke or Virtual WAN), identity, management/monitoring, subscription vending.
- Platform landing zone (shared services) vs application landing zone (workload subscription).
- Accelerators: ALZ Bicep / Terraform modules / portal accelerator (verify current repos in docs).

### 13.1 [Easy] What is a landing zone?
**Answer**: A pre-configured, governed environment (subscriptions, networking, identity, policies, monitoring) into which workloads are deployed. It ensures security and governance are in place before apps arrive.

**Explanation**: Design areas: billing/tenant, identity/access, network topology, resource organization, security, management, governance, platform automation/DevOps.

**Code / CLI / Bicep**
```bash
az account management-group create --name mg-landingzones --display-name "Landing Zones" --parent mg-org
az account management-group subscription add --name mg-corp --subscription <sub-id>
```

### 13.2 [Follow-up] Hub-and-spoke networking and why it matters for PaaS like Service Bus/Key Vault?
**Answer**: Hub VNet hosts shared services (firewall, DNS, gateway); spokes host workloads peered to the hub. PaaS services are reached via private endpoints in spokes with Private DNS zones linked centrally, giving private access and centralised egress/inspection.

**Explanation**: DNS is the usual failure point: private DNS zones (`privatelink.vaultcore.azure.net`, `privatelink.servicebus.windows.net`, `privatelink.blob.core.windows.net`) must resolve from workloads and on-prem. Functions need VNet integration to reach private endpoints.

**Code / CLI / Bicep**
```bicep
resource peering 'Microsoft.Network/virtualNetworks/virtualNetworkPeerings@2023-05-01' = {
  parent: spokeVnet
  name: 'spoke-to-hub'
  properties: {
    remoteVirtualNetwork: { id: hubVnetId }
    allowForwardedTraffic: true
    useRemoteGateways: false
  }
}
```

### 13.3 [Medium] What is subscription vending and policy-driven governance at scale?
**Answer**: Subscription vending automates creating subscriptions with baseline config (MG placement, RBAC, budgets, networking, tags) via IaC and pipelines. Governance is enforced by policy assigned at management groups so new subscriptions inherit guardrails.

**Explanation**: Reduces ad hoc subscriptions and drift. Guardrails: allowed locations, deny public IPs in Corp, require diagnostic settings (DINE), enforce tags, defender plans.

**Code / CLI / Bicep**
```bicep
targetScope = 'managementGroup'
resource assign 'Microsoft.Authorization/policyAssignments@2022-06-01' = {
  name: 'allowed-locations'
  properties: {
    displayName: 'Allowed locations'
    policyDefinitionId: tenantResourceId('Microsoft.Authorization/policyDefinitions','e56962a6-4747-49cd-b67b-bf8b01975c4c')
    parameters: { listOfAllowedLocations: { value: [ 'westeurope', 'northeurope' ] } }
  }
}
```
(Built-in "Allowed locations" ID - verify in current docs.)

### 13.4 [Tricky] A new team complains a Function can't reach Key Vault in the Corp landing zone though RBAC is right. Why?
**Answer**: Likely network: Key Vault firewall/private endpoint with no VNet integration on the Function, or private DNS not resolving to the private IP (resolving to public IP and being blocked). Check DNS resolution, route/NSG/firewall, and vault network rules.

**Explanation**: Debug with Kudu console/`nslookup`/`tcpping` (Functions with VNet integration), check `WEBSITE_VNET_ROUTE_ALL`/route-all setting (verify current naming), DNS server settings (custom DNS must forward `privatelink` zones to Azure DNS 168.63.129.16).

**Code / CLI / Bicep**
```bash
nslookup kv-orders.vault.azure.net
az functionapp vnet-integration add -g rg -n func-orders --vnet vnet-spoke --subnet snet-func
az network private-dns link vnet list -g rg-dns -z privatelink.vaultcore.azure.net -o table
```

**Gotchas / follow-ups**
- Landing-zone policies may Deny public network access - dev experiments fail by design.
- Centralised firewall must allow Azure AD/management endpoints for managed identity token acquisition (IMDS is local, fine).
- Subnet delegation required for some integrations (e.g. `Microsoft.Web/serverFarms`).

**Interview traps**
- Treating landing zone as a single template, not an operating model.
- Forgetting DNS for private endpoints.
- Putting prod and dev in one subscription.

**Cheat sheet**
- Platform vs application landing zones.
- Corp (private) vs Online (public).
- Policy at MG level; inherited.
- Hub-spoke, private endpoints, central Private DNS.
- Subscription vending = automation of onboarding.

---

## Topic 14: Environment strategy (dev/test/prod)

**Study notes**

- Common models: separate subscription per environment (recommended for prod isolation), or per-environment RG within shared non-prod subscription.
- Same IaC, different parameter files; promote artifacts, not rebuilt code. Configuration per environment via App Configuration/Key Vault/app settings; no secrets in repos.
- Identity: separate managed identities and RBAC per environment; pipeline uses distinct federated credentials per environment, with approvals/gates on prod.
- Cost levers: lower SKUs in dev (Basic/Standard instead of Premium), auto-shutdown, dev/test pricing (verify eligibility).

### 14.1 [Easy] Why separate prod from non-prod subscriptions?
**Answer**: Isolates blast radius, RBAC, policy, quotas and billing. Developers can have broad rights in dev while prod is locked down.

**Explanation**: Also avoids dev load consuming prod quota (vCPU, throttling) and simplifies cost reporting.

**Code / CLI / Bicep**
```bash
az account set --subscription sub-orders-prod
az deployment group create -g rg-orders-prod -f main.bicep -p @prod.bicepparam
```

### 14.2 [Follow-up] How do you parameterize environments in Bicep and in a Function app?
**Answer**: `.bicepparam` per environment controlling SKUs/names/scale; app settings and Key Vault references for runtime config, with `env` in resource names. The same build artifact deploys everywhere.

**Explanation**: Key Vault references in app settings use `@Microsoft.KeyVault(SecretUri=...)` and the app's identity must have Secrets User. Avoid environment `if` logic in code; read config.

**Code / CLI / Bicep**
```bicep
using './main.bicep'
param env = 'prod'
param sbSku = 'Premium'
param functionSku = 'EP1'
```
```json
{ "SqlConnection": "@Microsoft.KeyVault(SecretUri=https://kv-orders-prod.vault.azure.net/secrets/sql/)" }
```

### 14.3 [Medium] CI/CD promotion with approvals and workload identity federation.
**Answer**: Build once, deploy per environment via pipeline stages/environments with approvals; authenticate using OIDC federated credentials (no stored secrets) with an identity scoped per environment subscription.

**Explanation**: GitHub Actions `azure/login` with `client-id`, `tenant-id`, `subscription-id` and `id-token: write` permission; federated credential subject restricts to repo/environment. Azure DevOps uses workload identity federation service connections.

**Code / CLI / Bicep**
```yaml
permissions:
  id-token: write
  contents: read
jobs:
  deploy-prod:
    environment: prod
    runs-on: ubuntu-latest
    steps:
      - uses: azure/login@v2
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
      - run: az deployment group create -g rg-orders-prod -f main.bicep -p @prod.bicepparam
```

### 14.4 [Tricky] Config drift and 'works in dev, fails in prod'. Common causes?
**Answer**: Differences in networking (private endpoints/firewalls), RBAC propagation or missing role assignments, SKUs/limits (e.g. Premium vs Standard Service Bus features), policies in prod, and manual portal changes. Fix by IaC-only changes and drift checks (what-if).

**Explanation**: Parity matters for networking and identity more than size. Use prod-like pre-prod for integration tests. Verify service tier feature differences (e.g. message size limits, private endpoints) in current docs.

**Code / CLI / Bicep**
```bash
az deployment group what-if -g rg-orders-prod -f main.bicep -p @prod.bicepparam
```

**Gotchas / follow-ups**
- Shared Service Bus namespace across envs risks cross-env message consumption - separate namespaces/queues.
- Same UAMI reused across envs breaks least privilege.
- Slot swaps keep slot-sticky settings; know which are sticky.

**Interview traps**
- Rebuilding per environment instead of promoting artifacts.
- Secrets in pipeline variables rather than Key Vault/federation.
- Dev with Owner on prod.

**Cheat sheet**
- Sub per env (at least prod separate).
- One artifact, many configs.
- OIDC federation; approvals on prod.
- IaC only; what-if for drift.
- Separate identities per env.

---

## Topic 15: Quotas, throttling (429) across Azure

**Study notes**

- **Quotas/limits**: per-subscription/region (vCPU, public IPs, namespaces, etc.); soft limits can be raised by support request; hard limits cannot. Check `az vm list-usage`, portal Quotas blade.
- **Throttling**: services return **429 Too Many Requests** (or 503 for some e.g. Service Bus `ServerBusy`) and often a `Retry-After` header. ARM throttles per subscription/tenant with remaining-request headers (e.g. `x-ms-ratelimit-remaining-subscription-reads`) - verify in current docs.
- Service-specific: Storage (account/partition targets), Cosmos DB (RU, 429 with `x-ms-retry-after-ms`), Key Vault (requests per vault per 10 s), Service Bus (credit/messaging unit limits, namespace throttling), Entra token endpoints, Redis (CPU/connection limits).

### 15.1 [Easy] What does a 429 mean and what do you do?
**Answer**: The service is throttling you for exceeding a rate limit. Back off and retry after the `Retry-After` interval using exponential backoff with jitter; reduce request rate and cache where possible.

**Explanation**: Azure SDK retry policy honors Retry-After for supporting services. Persistent 429s mean you need caching, batching, scale-up, or a quota increase.

**Code / CLI / Bicep**
```csharp
try { await secretClient.GetSecretAsync("db-pass", cancellationToken: ct); }
catch (RequestFailedException ex) when (ex.Status == 429)
{
    log.LogWarning("Throttled: {Code}", ex.ErrorCode);
    throw;
}
```

### 15.2 [Follow-up] How do you reduce Key Vault throttling in an app with many instances?
**Answer**: Fetch secrets once at startup and cache in memory with a TTL (or use Key Vault references/App Configuration), reuse the `SecretClient` singleton, and avoid per-request secret reads. Use separate vaults per app/environment to avoid shared limits.

**Explanation**: Key Vault limits are per vault and per region per subscription (verify numbers in current docs). Scale-out functions each reading at cold start can spike - add jitter/caching.

**Code / CLI / Bicep**
```csharp
public sealed class SecretCache(SecretClient client, IMemoryCache cache)
{
    public Task<string> GetAsync(string name) =>
        cache.GetOrCreateAsync(name, async e =>
        {
            e.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(15);
            return (await client.GetSecretAsync(name)).Value.Value;
        })!;
}
```

### 15.3 [Medium] Check and request quota increases; and ARM throttling in pipelines.
**Answer**: Check usage via CLI/portal Quotas, request increases via support or the Quotas API; for ARM throttling, reduce parallel deployments/reads, use ARG for bulk reads, and respect `Retry-After`.

**Explanation**: Large Terraform/Bicep runs with high parallelism or tools polling `az resource list` often hit ARM read limits. Pre-check quotas before regional expansions/DR failovers - DR region may have insufficient vCPU quota!

**Code / CLI / Bicep**
```bash
az vm list-usage -l westeurope -o table
az quota list --scope /subscriptions/<sub>/providers/Microsoft.Compute/locations/westeurope   # quota extension; verify
terraform apply -parallelism=5
```

### 15.4 [Hard] Service Bus / Functions scale-out is throttled (ServerBusy / 429) or overloads a downstream SQL/Redis. Diagnose and fix.
**Answer**: Check metrics (Throttled Requests, server errors, CPU/memory for Premium) and downstream saturation; limit concurrency (`maxConcurrentCalls`, `functionAppScaleLimit`), use batching, upgrade tier/MUs, or partition across namespaces. Add backpressure rather than retries only.

**Explanation**: Standard tier shares capacity and throttles; Premium offers dedicated messaging units. Redis throttles on CPU/bandwidth/connection counts - use a singleton `ConnectionMultiplexer`. Functions scale-out multiplies connections to SQL; use connection pooling limits.

**Code / CLI / Bicep**
```kusto
AzureMetrics
| where ResourceProvider == "MICROSOFT.SERVICEBUS" and MetricName in ("ThrottledRequests","ServerErrors")
| summarize sum(Total) by MetricName, bin(TimeGenerated, 5m)
```
```json
{ "extensions": { "serviceBus": { "maxConcurrentCalls": 8, "prefetchCount": 0 } } }
```

**Gotchas / follow-ups**
- Retries add to the load causing the throttle.
- Throttling headers differ by service; don't assume uniform behaviour.
- Moving to Premium solves noisy-neighbour issues but costs fixed money.

**Interview traps**
- Retrying without backoff.
- Believing quotas are unlimited / requests instant.
- Forgetting DR region quota.
- Per-request secret fetches.

**Cheat sheet**
- 429/503 + Retry-After -> backoff.
- Cache secrets/tokens.
- Check `az vm list-usage` before scaling/DR.
- Cap concurrency to protect downstream.
- Limits change - verify in current docs.

---

## Topic 16: Azure DevOps vs GitHub (brief)

**Study notes**

| Aspect | Azure DevOps | GitHub |
|---|---|---|
| Source | Azure Repos (Git/TFVC) | GitHub repos |
| CI/CD | Azure Pipelines (YAML + classic) | GitHub Actions |
| Boards | Azure Boards (strong) | Issues/Projects |
| Artifacts | Azure Artifacts | GitHub Packages |
| Security | Basic/Defender for DevOps | Advanced Security (Dependabot, code scanning, secret scanning) |
| Azure auth | Service connections (WIF) | OIDC federated credentials |

Both support YAML pipelines, environments with approvals, and self-hosted agents/runners. Microsoft's direction leans toward GitHub for new development; verify roadmap in current docs.

### 16.1 [Easy] Main difference between Azure DevOps and GitHub?
**Answer**: Azure DevOps is an integrated suite (Boards, Repos, Pipelines, Test Plans, Artifacts); GitHub centres on repos/collaboration/open source with Actions for CI/CD. Both can deploy to Azure.

**Explanation**: Many orgs mix: GitHub repos + Azure Pipelines, or Azure Boards linked to GitHub.

**Code / CLI / Bicep**
```yaml
# azure-pipelines.yml
trigger: [ main ]
pool: { vmImage: ubuntu-latest }
steps:
  - task: AzureCLI@2
    inputs:
      azureSubscription: 'sc-orders-prod'
      scriptType: bash
      scriptLocation: inlineScript
      inlineScript: az deployment group create -g rg-orders-prod -f main.bicep
```

### 16.2 [Follow-up] How do pipelines authenticate to Azure without secrets?
**Answer**: Workload identity federation: an Entra app/UAMI with a federated credential trusting the pipeline's OIDC token (repo/environment or service connection). No client secret to rotate.

**Explanation**: Subject claim format restricts which repo/branch/environment can obtain tokens. Grant the identity the minimum RBAC at the target scope.

**Code / CLI / Bicep**
```bash
az identity federated-credential create --name gh-prod --identity-name uami-deploy -g rg-id \
  --issuer https://token.actions.githubusercontent.com \
  --subject repo:contoso/orders:environment:prod --audiences api://AzureADTokenExchange
```

### 16.3 [Medium] Pick one for a new .NET + Bicep project and justify.
**Answer**: GitHub + Actions if the team values code-centric collaboration, built-in security scanning and open-source style PRs; Azure DevOps if you need mature work-item tracking, test management, or have existing enterprise pipelines/processes. The choice is rarely technical - decide on governance, tooling and existing investments.

**Explanation**: Both handle build-once/deploy-many with environments, approvals, templates (reusable workflows / pipeline templates) and caching.

**Code / CLI / Bicep**
```yaml
- uses: actions/setup-dotnet@v4
  with: { dotnet-version: '8.0.x' }
- run: dotnet test --configuration Release
- run: dotnet publish -c Release -o out
```

### 16.4 [Tricky] Pipeline succeeds in Bicep what-if but fails on deploy with AuthorizationFailed on role assignment. Why?
**Answer**: The pipeline identity likely has Contributor but not role-assignment write permission (needs Owner/User Access Administrator or a constrained delegated role); what-if may not fully surface it.

**Explanation**: Grant a narrowly scoped role (e.g. Role Based Access Control Administrator with conditions) rather than Owner. Verify role names in current docs.

**Code / CLI / Bicep**
```bash
az role assignment create --assignee <deployer-objectId> --role "Role Based Access Control Administrator" --scope /subscriptions/<sub>/resourceGroups/rg-orders-prod
```

**Gotchas / follow-ups**
- Secrets in pipeline logs - mask and avoid echo.
- Self-hosted agents in VNet are needed to reach private endpoints (e.g. Key Vault/ACR) during deploy.
- Federated credential subject mismatch yields AADSTS70021/AADSTS700213-style errors (verify exact codes).

**Interview traps**
- Storing service principal secrets in variables.
- "Azure DevOps is deprecated" - it is not; check current roadmap.
- Forgetting environment-protected approvals.

**Cheat sheet**
- ADO = suite; GitHub = repo-centric + Actions.
- OIDC/WIF over secrets.
- Build once, promote.
- Self-hosted agents for private networks.
- Least-privilege deploy identity.

---

## Topic 17: Common "which service would you choose" interview questions

**Study notes**

| Need | Choose | Why / Notes |
|---|---|---|
| Reliable async work queue, DLQ, sessions, transactions | **Service Bus queue** | Enterprise messaging, FIFO via sessions, dedup |
| Pub/sub with filters | **Service Bus topic/subscription** | Rules/filters per subscription |
| High-throughput event/telemetry stream | **Event Hubs** | Partitioned log, replay, Kafka endpoint |
| Reactive event routing (resource events, blob created) | **Event Grid** | Push, low-latency, filtering |
| Simple storage queue, huge, cheap | **Storage Queue** | Basic, no DLQ/sessions |
| Short event-driven code | **Functions** | Triggers/bindings |
| Low-code integration/orchestration with connectors | **Logic Apps** | Connectors, designer |
| Long-running code workflow with state | **Durable Functions** | Code-first orchestration |
| Cache, session store, rate limit counters | **Azure Cache for Redis** (or Azure Managed Redis; verify) | In-memory; not source of truth |
| Secrets/keys/certs | **Key Vault** | RBAC, soft delete |
| Unstructured files | **Blob Storage** | Tiers, lifecycle |
| Relational | **Azure SQL** | Managed SQL |
| Global multi-model NoSQL | **Cosmos DB** | RU-based, multi-region |
| Identity for workloads | **Managed identity** | No secrets |
| Telemetry/logs/traces | **Azure Monitor + App Insights + Log Analytics** | KQL |

### 17.1 [Easy] Service Bus vs Storage Queue vs Event Hubs vs Event Grid?
**Answer**: Service Bus for transactional, ordered, reliable business messages with DLQ and sessions; Storage Queue for simple, cheap, large-scale queues; Event Hubs for high-volume event streaming; Event Grid for reactive event notifications/routing. Choose by semantics: command/message vs event/stream vs notification.

**Explanation**: Messages (commands, one consumer owns processing) vs events (facts, many subscribers). Event Hubs consumers track their own offset (checkpoint); Service Bus broker tracks state per message.

**Code / CLI / Bicep**
```bash
az servicebus namespace create -g rg -n sb-orders --sku Standard
az servicebus queue create -g rg --namespace-name sb-orders -n orders --max-delivery-count 10 --enable-dead-lettering-on-message-expiration true
```

### 17.2 [Follow-up] Functions vs Logic Apps vs Durable Functions?
**Answer**: Logic Apps for connector-heavy, low-code workflows with built-in monitoring; Functions for custom code and high-performance triggers; Durable Functions for code-based stateful orchestrations (fan-out/fan-in, human approval, retries). They are often combined: Logic App calls Function, Function emits to Service Bus.

**Explanation**: Consider team skills, testing/CI-CD story (code vs designer), cost model (per action vs per execution), latency, and versioning.

**Code / CLI / Bicep**
```csharp
[Function("Orchestrator")]
public static async Task<string> Run([OrchestrationTrigger] TaskOrchestrationContext ctx)
{
    var a = await ctx.CallActivityAsync<string>("StepA", "in");
    return await ctx.CallActivityAsync<string>("StepB", a);
}
```

### 17.3 [Medium] SAMI vs UAMI - which and why? Plus where Redis and Key Vault fit.
**Answer**: SAMI is tied to one resource's lifecycle (deleted with it, simple); UAMI is a standalone resource reusable across resources, survives redeploys, and lets you pre-assign RBAC before the app exists. Use UAMI for shared access patterns/IaC ordering and slots; SAMI for simple one-to-one. Key Vault stores secrets/keys; Redis is a cache, not a secret store.

**Explanation**: UAMI role assignments persist across app recreation (stable principal). SAMI recreated -> new principalId -> RBAC must be re-granted. A resource can have multiple UAMIs, but picking one in code requires specifying client ID (`ManagedIdentityClientId`). Redis with Entra auth is available - verify current support per tier.

**Code / CLI / Bicep**
```bicep
resource uami 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = {
  name: 'uami-orders'
  location: location
}
resource func 'Microsoft.Web/sites@2023-01-01' = {
  name: 'func-orders'
  location: location
  kind: 'functionapp'
  identity: { type: 'UserAssigned', userAssignedIdentities: { '${uami.id}': {} } }
  properties: { serverFarmId: planId }
}
```
```csharp
var cred = new DefaultAzureCredential(new DefaultAzureCredentialOptions { ManagedIdentityClientId = uamiClientId });
```

### 17.4 [Hard] Design: ingest orders via API, process reliably, notify partners, and observe everything. Which services and why?
**Answer**: API (App Service/Functions HTTP or APIM) -> Service Bus queue/topic -> Functions consumers (idempotent, DLQ) -> Blob for payloads/claim-check -> Logic Apps for partner connectors -> App Insights/Log Analytics with correlation IDs; Key Vault for secrets; managed identity everywhere; Redis for dedup/cache.

**Explanation**: Justify each choice, cite trade-offs: Service Bus for guaranteed delivery and DLQ; Event Grid only for notifications (e.g. blob created); Premium tier for private networking/larger messages (verify limits). Observability: distributed tracing via W3C `traceparent` in message `ApplicationProperties`; alerts on DLQ depth, failures, and throttling.

**Code / CLI / Bicep**
```csharp
var msg = new ServiceBusMessage(BinaryData.FromObjectAsJson(order))
{
    MessageId = order.Id,                       // enables duplicate detection when enabled on the entity
    CorrelationId = Activity.Current?.TraceId.ToString(),
    ContentType = "application/json"
};
await sender.SendMessageAsync(msg, ct);
```
```kusto
traces
| where timestamp > ago(1h) and severityLevel >= 3
| summarize count() by cloud_RoleName, operation_Name
```

**Gotchas / follow-ups**
- Message size limits differ by tier (verify in current docs) - use claim-check for large payloads.
- Duplicate detection window is bounded; idempotent handling still needed.
- Event Grid is at-least-once and unordered; don't use it for strict order.
- Sessions required for FIFO per key; limits parallelism per session.

**Interview traps**
- Choosing Event Hubs for command-style work queues.
- Using Redis as the only dedup store without durability thinking.
- Saying "Logic Apps for everything" or "Functions for everything".
- No answer for failure/DLQ handling.

**Cheat sheet**
- Command/work -> Service Bus; stream -> Event Hubs; notification -> Event Grid; trivial queue -> Storage Queue.
- Code orchestration -> Durable; connectors -> Logic Apps.
- UAMI for reuse/stable RBAC; SAMI for simple lifecycle-bound.
- Always state trade-offs and failure handling.
- Verify tier limits/prices in current docs.

---
