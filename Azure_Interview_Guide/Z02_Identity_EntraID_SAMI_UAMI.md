# Z02 - Identity: Entra ID, Managed Identity (SAMI/UAMI), RBAC, Tokens

**How to use:** Read each topic's Study notes, then work the ladder top-down (Easy -> Tricky); cover the Answer and say it aloud before reading the Explanation.
Where limits/SDK versions/preview features are mentioned, "verify in current docs" - Azure changes these often.

## Table of Contents
1. Entra ID basics: tenant, users, groups, app registrations vs enterprise apps
2. AuthN vs AuthZ and the managed identity concept
3. SAMI vs UAMI
4. Service principal (secret/cert) vs managed identity
5. RBAC (roles, scope, propagation, deny) and control plane vs data plane
6. DefaultAzureCredential and credential chains
7. Token acquisition internals (IMDS, App Service endpoint, caching)
8. Using identity with Azure services (SB, Blob, KV, SQL, Redis, EH, AppConfig, Cosmos)
9. Functions / Logic Apps identity-based connections
10. OAuth 2.0 / OIDC flows in Entra
11. Securing your ASP.NET Core API (Microsoft.Identity.Web)
12. MSAL and the token cache
13. Workload identity federation
14. Conditional Access, MFA, PIM, access reviews
15. Cross-tenant, B2B, B2C, External ID
16. Least privilege and auditing
17. Common errors and fixes
18. Zero-secret architecture end-to-end (Bicep)

---

## Topic 1: Entra ID basics: tenant, users, groups, app registrations vs enterprise apps

**Study notes**

- **Tenant** = a dedicated Entra ID directory instance (a security boundary), identified by a Tenant ID (GUID) and a domain like `contoso.onmicrosoft.com`.
- **Subscription** trusts exactly one tenant; one tenant can have many subscriptions. Entra roles (Global Admin) are different from Azure RBAC roles.
- **Objects:** users, groups (Security / Microsoft 365), devices, applications, service principals, managed identities.

```
App registration (Application object)   -- the global "definition", lives in HOME tenant
        |  1 : N
        v
Enterprise application (Service principal) -- the local "instance" in EACH tenant that uses the app
        ^
        |  RBAC / consent / assignments are granted to the SERVICE PRINCIPAL
```

| Thing | Where you see it | Holds |
|---|---|---|
| App registration | Entra > App registrations | client ID (appId), redirect URIs, secrets/certs, exposed scopes, app roles, API permissions requested |
| Enterprise app / service principal | Entra > Enterprise applications | object ID, user/group assignment, granted consent, Conditional Access targets, role assignments |

Identifiers: **Application (client) ID** = appId (same across tenants); **Object ID** = differs per object (app object vs SP have different object IDs).

### 1.1 [Easy] What is a tenant and how does it relate to a subscription?
**Answer**
A tenant is a dedicated Entra ID directory that holds identities and app objects and acts as a security boundary. Each Azure subscription trusts one tenant for authentication, while one tenant can back many subscriptions.

**Explanation**
Moving a subscription to another tenant is possible but breaks RBAC assignments and managed identities (they are tied to the old directory). Entra directory roles (e.g., Global Administrator) govern the directory; Azure RBAC governs resources. Don't confuse the two.

**Code / CLI / Bicep**
```bash
az account show --query "{sub:id, tenant:tenantId, user:user.name}" -o table
az account tenant list -o table
```

### 1.2 [Follow-up] App registration vs enterprise application - what's the difference?
**Answer**
The app registration is the global application object (definition) in the home tenant; the enterprise application is the service principal, the local instance in a tenant that actually receives permissions, consent, and role assignments. Registering an app via the portal creates both in the home tenant.

**Explanation**
For multi-tenant apps, each customer tenant gets its own service principal when an admin/user consents. Role assignments always target the service principal's object ID, never the app registration. Managed identities have a service principal but no app registration.

**Code / CLI / Bicep**
```bash
az ad app create --display-name my-api
az ad sp create --id <appId>            # create the SP for an existing app registration
az ad sp show --id <appId> --query id -o tsv   # SP object ID (used for role assignments)
```

### 1.3 [Medium] Which ID do you use where: client ID, object ID, tenant ID?
**Answer**
Client (application) ID is used by apps to authenticate and in token requests; object ID of the service principal is used for RBAC role assignment (`principalId`); tenant ID identifies the directory/authority. Mixing the app object ID with the SP object ID is a classic mistake.

**Explanation**
Three GUIDs for one app: appId, application object ID, service principal object ID. Bicep `roleAssignment.properties.principalId` needs the SP object ID. CLI `az role assignment create --assignee` accepts appId or SP object ID and resolves it.

**Code / CLI / Bicep**
```bash
az ad sp list --filter "appId eq '<appId>'" --query "[0].{appId:appId, spObjectId:id}" -o json
```

### 1.4 [Medium] Groups: why assign roles to groups, and what are the limits?
**Answer**
Assigning roles to groups keeps RBAC manageable (membership changes instead of new assignments). Group claims in tokens can overflow (about 200 groups in a JWT, 150 for SAML), after which a `_claim_names` overage pointer appears and you must call Microsoft Graph.

**Explanation**
Security groups can be role-assignable (for Entra roles) only if created that way. Managed identities can be members of security groups, which is a handy pattern for granting many identities the same access, though group membership is cached in the token so changes take until token refresh. Verify overage limits in current docs.

**Code / CLI / Bicep**
```bash
az ad group create --display-name app-readers --mail-nickname app-readers
az ad group member add --group app-readers --member-id <principalId-of-MI>
az role assignment create --assignee-object-id <groupObjectId> --assignee-principal-type Group \
  --role "Storage Blob Data Reader" --scope <storageAccountId>
```

### 1.5 [Tricky] An app works in your tenant but a customer's tenant says "application not found". Why?
**Answer**
The multi-tenant app's service principal doesn't exist in the customer's tenant until consent is granted (or the SP is provisioned). The app registration is only in your home tenant.

**Explanation**
Error is typically AADSTS700016 (application not found in directory) or AADSTS650051/65001 for consent. Fix: have a customer admin consent via the admin consent URL, or provision the SP with `az ad sp create`. Also check the app's `signInAudience` is multi-tenant (`AzureADMultipleOrgs`).

**Code / CLI / Bicep**
```bash
# admin consent URL pattern
# https://login.microsoftonline.com/{customerTenant}/adminconsent?client_id={appId}
az ad app update --id <appId> --sign-in-audience AzureADMultipleOrgs
```

**Gotchas / follow-ups**
- Deleting an enterprise app but not the registration leaves the registration; recreating SP gives a NEW object ID, so RBAC assignments are orphaned.
- "Assignment required" on the enterprise app blocks users not assigned.

**Interview traps**
- Saying "app registration gets the role" - roles go to the service principal.
- Confusing Entra directory roles with Azure RBAC roles.
- Believing a subscription can trust multiple tenants.

**Cheat sheet**
- Tenant = directory; subscription trusts one tenant.
- App registration = definition (home tenant); enterprise app = SP (per tenant).
- RBAC `principalId` = SP/MI object ID.
- Group assignments > individual assignments.

---

## Topic 2: AuthN vs AuthZ and the managed identity concept

**Study notes**

- **Authentication (AuthN):** proving who you are -> Entra issues a token (JWT).
- **Authorization (AuthZ):** what you may do -> Azure RBAC (resource), app roles/scopes (your API), service-native ACLs (SQL permissions).
- **401** = unauthenticated/invalid token; **403** = authenticated but not permitted.
- **Managed identity (MI):** an Entra identity Azure manages for a resource; credentials are rotated by the platform and never visible to you. No secrets in code/config.

```
App code --> (identity endpoint, local) --> Entra ID --> access token (audience = target resource)
App code --> target service with  Authorization: Bearer <token>
Target service validates signature/audience/issuer, then checks RBAC
```

### 2.1 [Easy] Difference between authentication and authorization?
**Answer**
Authentication verifies identity; authorization decides what that identity can do. In Azure, Entra ID authenticates and issues tokens, and RBAC (or app-level roles/scopes) authorizes.

**Explanation**
A valid token does not imply access. Typical failure: token acquired fine (AuthN OK) but 403 from Service Bus because no data role assigned (AuthZ missing).

**Code / CLI / Bicep**
```bash
az account get-access-token --resource https://servicebus.azure.net --query accessToken -o tsv
```

### 2.2 [Follow-up] What is a managed identity and why use it?
**Answer**
A managed identity is an Entra service principal automatically managed by Azure for a resource, letting code get tokens without storing secrets. It removes secret rotation, leakage risk, and Key Vault bootstrap problems.

**Explanation**
The "secret zero" problem: you need a credential to reach Key Vault to get credentials. MI solves it because the platform proves the resource identity. Works only for code running on Azure compute (VM, App Service, Functions, AKS nodes, Container Apps, Logic Apps, etc.); verify supported services in current docs.

**Code / CLI / Bicep**
```csharp
using Azure.Identity;
using Azure.Security.KeyVault.Secrets;

var client = new SecretClient(new Uri("https://myvault.vault.azure.net/"), new DefaultAzureCredential());
KeyVaultSecret s = await client.GetSecretAsync("db-conn");
```

### 2.3 [Medium] Where does authorization happen when a managed identity calls Blob Storage?
**Answer**
The MI gets a token with audience `https://storage.azure.com/`; Blob validates it and Azure RBAC data actions on the storage scope decide access. The token itself carries identity, not permissions (for ARM/data-plane RBAC).

**Explanation**
Entra tokens for Azure resources carry `oid`, `tid`, `appid`/`azp`, `aud` but not RBAC roles. The service checks role assignments server-side at request time (subject to caching/propagation delay). In contrast, your own API typically relies on `scp`/`roles` claims in the token.

**Code / CLI / Bicep**
```bash
az role assignment create --assignee-object-id <miPrincipalId> --assignee-principal-type ServicePrincipal \
  --role "Storage Blob Data Contributor" --scope <storageId>
```

### 2.4 [Tricky] Can an MI be used from your laptop?
**Answer**
No. The identity endpoint is only reachable from the Azure resource it's bound to. Locally you use developer credentials (VS, VS Code, Azure CLI) or a service principal, typically through DefaultAzureCredential.

**Explanation**
Common "works in Azure, fails locally" is the reverse: the dev account lacks the RBAC role the MI has. Give the developer group the same data role on dev resources (not prod). Use Azure Arc if you need MI on non-Azure servers.

**Code / CLI / Bicep**
```bash
az login
az role assignment create --assignee <devUpnOrObjectId> --role "Azure Service Bus Data Owner" --scope <dev-namespace-id>
```

**Gotchas / follow-ups**
- Dev user's MFA/Conditional Access may block CLI token acquisition.
- Don't grant dev users prod roles just to make local work.

**Interview traps**
- "Managed identity means no authorization needed" - you still need RBAC.
- Thinking tokens contain RBAC role claims for ARM/data plane.

**Cheat sheet**
- 401 = who are you?; 403 = you're not allowed.
- MI = platform-managed SP, no secrets.
- Locally: use developer credential, same code via DefaultAzureCredential.

---

## Topic 3: SAMI vs UAMI

**Study notes**

| Aspect | System-assigned (SAMI) | User-assigned (UAMI) |
|---|---|---|
| Created | Enabled on the resource | Standalone ARM resource `Microsoft.ManagedIdentity/userAssignedIdentities` |
| Lifecycle | Tied to resource; deleted with it | Independent; deleted explicitly |
| Sharing | 1 resource only | Attach to many resources |
| Per resource | Max 1 | Many UAMIs per resource (verify limits in docs) |
| Role assignments | Lost when resource deleted (orphaned) / new principal on recreate | Survive resource deletion/recreation |
| Principal known | Only after resource deployment | Before resource exists (good for pre-provisioning RBAC) |

**Three IDs for a UAMI**
- `clientId` - what you pass to SDKs (`ManagedIdentityCredential(clientId)`, `AZURE_CLIENT_ID`).
- `principalId` (= object ID of its service principal) - what you use in role assignments.
- `id` (resource ID) - what you attach to resources (`identity.userAssignedIdentities` key) and can be used in ARM-style identity requests.

### 3.1 [Easy] What is the difference between SAMI and UAMI?
**Answer**
SAMI is enabled on a single resource and shares its lifecycle; UAMI is a separate Azure resource you create once and attach to many resources. UAMIs survive deletion of the resources using them.

**Explanation**
SAMI is simplest for one-to-one workloads. UAMI shines for shared access, pre-provisioned RBAC, and redeployable/ephemeral compute.

**Code / CLI / Bicep**
```bash
az functionapp identity assign -g rg -n myfunc                      # SAMI
az identity create -g rg -n id-app-prod                            # create UAMI
az functionapp identity assign -g rg -n myfunc --identities <uamiResourceId>
```

### 3.2 [Follow-up] When do you choose UAMI over SAMI?
**Answer**
Choose UAMI when many resources need the same access (e.g., 10 function apps -> one Service Bus role assignment), when you deploy with IaC and need RBAC before the compute exists, or when slots/recreation would otherwise rotate the principal. Choose SAMI when one resource needs isolated, auto-cleaned identity.

**Explanation**
Trade-off: UAMI blast radius is wider (anything attached gets all permissions), so least privilege suffers. SAMI gives per-resource auditing granularity (the sign-in log shows which resource). With UAMI, audit relies on the resource-level logs. Also note deployment-order race: with SAMI role assignment in the same Bicep template, you reference `app.identity.principalId`, fine, but propagation delays can cause first-run failures.

**Code / CLI / Bicep**
```bicep
resource uami 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = {
  name: 'id-orders-prod'
  location: resourceGroup().location
}
output clientId string = uami.properties.clientId
output principalId string = uami.properties.principalId
```

### 3.3 [Medium] What happens to role assignments when you delete and recreate a resource with SAMI?
**Answer**
Deleting the resource deletes the SAMI service principal; the recreated resource gets a new principalId, so old role assignments are orphaned (shown as "Identity not found") and the new identity has no access. A UAMI keeps its principalId so access persists.

**Explanation**
Common outage after "delete and redeploy" of a Function App. Fix: re-run the template that assigns roles (use deterministic `guid()` names), and clean orphaned assignments. Orphans count against the role assignment limit per subscription (verify the current limit, commonly 4000).

**Code / CLI / Bicep**
```bash
az role assignment list --all --query "[?principalName==''].{id:id, role:roleDefinitionName, scope:scope}" -o table
az role assignment delete --ids <orphanedAssignmentId>
```

### 3.4 [Medium] A UAMI is attached but the app still uses the wrong identity. Why?
**Answer**
With a UAMI (or with both SAMI and UAMI present) you must tell the SDK which identity to use - the clientId - otherwise DefaultAzureCredential tries the system-assigned identity and fails if none exists. Set `AZURE_CLIENT_ID` or pass `ManagedIdentityClientId`.

**Explanation**
If a resource has multiple UAMIs and no client ID is given, the IMDS call is ambiguous and fails (error like "Multiple user assigned identities exist"). Using the wrong ID type (principalId instead of clientId) gives "identity not found".

**Code / CLI / Bicep**
```csharp
var cred = new DefaultAzureCredential(new DefaultAzureCredentialOptions
{
    ManagedIdentityClientId = Environment.GetEnvironmentVariable("AZURE_CLIENT_ID")
});
// or explicitly:
var mi = new ManagedIdentityCredential(ManagedIdentityId.FromUserAssignedClientId("<clientId>"));
```
(`ManagedIdentityId` exists in recent Azure.Identity versions; older ones take a clientId string - verify in current docs.)

### 3.5 [Hard] Role assignment from Bicep to SAMI fails intermittently with PrincipalNotFound. Why and how to fix?
**Answer**
The service principal is created in Entra and may not have replicated yet when the role assignment runs. Setting `principalType: 'ServicePrincipal'` on the role assignment lets ARM skip the Graph lookup and avoids the replication race.

**Explanation**
Without `principalType`, ARM validates the principal via Graph, which can lag behind a just-created identity. Also use a deterministic name with `guid(scope, principalId, roleDefId)` so redeploys are idempotent; changing principalId with the same name causes `RoleAssignmentUpdateNotPermitted`.

**Code / CLI / Bicep**
```bicep
resource ra 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  scope: storage
  name: guid(storage.id, func.identity.principalId, 'ba92f5b4-2d11-453d-a403-e96b0029c9fe')
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', 'ba92f5b4-2d11-453d-a403-e96b0029c9fe') // Storage Blob Data Contributor
    principalId: func.identity.principalId
    principalType: 'ServicePrincipal'
  }
}
```

**Gotchas / follow-ups**
- Function deployment slots: slots can have their own SAMI; slot swap doesn't move identity -> grant roles to slot identity too (or use UAMI).
- Soft/hard limits on UAMIs per subscription and per-resource attachments: verify in current docs.
- Identity-token caching: after attach/remove, the old identity may linger in tokens up to ~24h (backend caches per resource URI); verify in docs.

**Interview traps**
- Using principalId where clientId is needed (or vice-versa).
- Claiming SAMI role assignments survive resource recreation.
- "UAMI is always better" - it widens blast radius.

**Cheat sheet**
- SDK wants clientId; RBAC wants principalId; attach uses resource ID.
- SAMI: dies with resource. UAMI: independent.
- Set `principalType: 'ServicePrincipal'` and `guid()` names in Bicep.
- Many functions sharing one access set -> UAMI.

---

## Topic 4: Service principal (secret/cert) vs managed identity

**Study notes**

| | Service principal + secret/cert | Managed identity |
|---|---|---|
| Credential | Client secret (max typically 24 months) or certificate | None visible; platform-managed |
| Rotation | You (expiry outages!) | Automatic |
| Works from | Anywhere | Azure compute only |
| Leak risk | High (config, repos, logs) | Very low |
| Use for | CI/CD (prefer federated), on-prem, external systems | Azure-hosted workloads |

### 4.1 [Easy] When would you still use a service principal instead of a managed identity?
**Answer**
When the workload runs outside Azure (on-prem, other clouds, local automation) or in a CI/CD system that can't use managed identity. Even then, prefer federated credentials (OIDC) over secrets.

**Explanation**
Order of preference: managed identity > workload identity federation > certificate credential > client secret.

**Code / CLI / Bicep**
```bash
az ad sp create-for-rbac --name sp-ci --role Reader --scopes /subscriptions/<sub>   # returns a secret: store safely
```

### 4.2 [Follow-up] Secret vs certificate credentials?
**Answer**
Certificates are stronger (private key never sent, can be in HSM/Key Vault, assertion-based auth) while secrets are bearer strings that can be leaked. Both expire and need rotation unless you federate.

**Explanation**
Secret expiry is a top production-outage cause (`AADSTS7000222: provided client secret keys are expired`). Monitor expiry with alerts or scripts.

**Code / CLI / Bicep**
```csharp
using Azure.Identity;
var cred = new ClientCertificateCredential(tenantId, clientId, new X509Certificate2("cert.pfx", pwd));
var secretCred = new ClientSecretCredential(tenantId, clientId, clientSecret); // avoid in prod
```

### 4.3 [Medium] How does the SDK pick up a service principal without code changes?
**Answer**
DefaultAzureCredential's EnvironmentCredential reads `AZURE_TENANT_ID`, `AZURE_CLIENT_ID`, and `AZURE_CLIENT_SECRET` (or `AZURE_CLIENT_CERTIFICATE_PATH`). That's the usual way CI or local test setups plug in an SP.

**Explanation**
Danger: leaving `AZURE_CLIENT_SECRET` set in an Azure host app setting makes EnvironmentCredential win over ManagedIdentityCredential (it comes first in the chain), silently using the SP.

**Code / CLI / Bicep**
```bash
export AZURE_TENANT_ID=<t> AZURE_CLIENT_ID=<c> AZURE_CLIENT_SECRET=<s>
dotnet run
```

### 4.4 [Tricky] An app suddenly fails at 2 AM with AADSTS7000222. What happened and how to prevent it?
**Answer**
The SP client secret expired. Prevent by moving to managed identity/federation, or by setting expiry alerts, using certificates in Key Vault with auto-rotation, and overlapping two credentials during rotation.

**Explanation**
Entra supports multiple credentials on one app, so add a new secret, roll out, then delete the old. Key Vault can store and (for certs) auto-renew, but the app must reload.

**Code / CLI / Bicep**
```bash
az ad app credential list --id <appId> --query "[].{id:keyId, end:endDateTime}" -o table
az ad app credential reset --id <appId> --append --years 1   # --append keeps old credential
```

**Gotchas / follow-ups**
- `reset` without `--append` replaces all credentials -> instant outage for other consumers.

**Interview traps**
- Putting SP secrets in app settings "temporarily".
- Forgetting EnvironmentCredential precedence.

**Cheat sheet**
- Prefer MI > federation > cert > secret.
- Rotate with overlap (`--append`).
- Alert on credential expiry.

---

## Topic 5: RBAC, scope inheritance, and control plane vs data plane

**Study notes**

- Role assignment = **principal + role definition + scope**.
- Scopes inherit downward: Management group > Subscription > Resource group > Resource.
- Role definition has `Actions` / `NotActions` (control plane) and `DataActions` / `NotDataActions` (data plane).
- **Control plane** = ARM (`management.azure.com`): create/delete/configure resources. **Data plane** = service endpoints: read blobs, send messages, get secrets.

| Role | Plane | Notes |
|---|---|---|
| Owner / Contributor / Reader | Control | Contributor can't assign roles; Owner can. Do NOT give data access (mostly; Contributor can list storage keys -> indirect access!) |
| Storage Blob Data Contributor/Reader/Owner | Data | Read/write blobs |
| Azure Service Bus Data Sender / Receiver / Owner | Data | Send / receive+complete / all |
| Key Vault Secrets User / Officer | Data (RBAC vault) | Get secrets / manage secrets |
| Azure Event Hubs Data Sender / Receiver / Owner | Data | |
| App Configuration Data Reader / Owner | Data | |
| Cosmos DB | Own RBAC system | Cosmos SQL role definitions (not ARM roles); verify |
| Azure SQL | T-SQL contained users | Not Azure RBAC for data |

### 5.1 [Easy] What are the components of a role assignment?
**Answer**
A security principal (user, group, SP, MI), a role definition, and a scope. Permissions apply to that scope and everything below it.

**Explanation**
Assign at the narrowest scope that works. Assignment names are GUIDs.

**Code / CLI / Bicep**
```bash
az role assignment create --assignee-object-id <oid> --assignee-principal-type ServicePrincipal \
  --role "Azure Service Bus Data Receiver" \
  --scope /subscriptions/<sub>/resourceGroups/rg/providers/Microsoft.ServiceBus/namespaces/ns/queues/orders
```

### 5.2 [Follow-up] I'm Owner on the storage account but get 403 reading blobs with Entra auth. Why?
**Answer**
Owner/Contributor are control-plane roles and carry no data actions. You need a data-plane role such as Storage Blob Data Reader. (With shared key auth it would work because Owner can fetch keys.)

**Explanation**
Classic trap. Portal may fall back to access keys, masking the issue; CLI with `--auth-mode login` exposes it. Same for Key Vault with RBAC permission model and Service Bus.

**Code / CLI / Bicep**
```bash
az storage blob list --account-name acct -c data --auth-mode login   # uses Entra; fails without data role
```

### 5.3 [Medium] Where can you assign a role and what is the minimal scope for Service Bus?
**Answer**
At management group, subscription, resource group, or resource scope; for Service Bus you can go down to the namespace, queue, topic, or subscription. Prefer entity-level scope for least privilege.

**Explanation**
Entity-level assignments (queue) avoid giving access to the whole namespace. Receiver role includes Receive + complete/abandon; Sender role only Send. Peek/lock needs Receiver.

**Code / CLI / Bicep**
```bicep
resource queue 'Microsoft.ServiceBus/namespaces/queues@2022-10-01-preview' existing = { name: '${ns.name}/orders' }
resource send 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  scope: queue
  name: guid(queue.id, principalId, '69a216fc-b8fb-44d8-bc22-1f3c2cd27a39')
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '69a216fc-b8fb-44d8-bc22-1f3c2cd27a39') // Azure Service Bus Data Sender
    principalId: principalId
    principalType: 'ServicePrincipal'
  }
}
```
(Verify role GUIDs in current docs / `az role definition list`.)

### 5.4 [Medium] I just assigned a role and still get 403. What now?
**Answer**
RBAC propagation can take a few minutes (sometimes longer), and tokens/caches may hold old authorization. Wait, retry, and if it persists re-acquire the token (restart the app) and verify scope and principal.

**Explanation**
ARM caches role assignments; MI tokens are cached ~24h by the platform and carry group membership, so group-based changes may need a new token. Verify the scope is the right resource and that you assigned to the correct identity (UAMI principalId, not clientId).

**Code / CLI / Bicep**
```bash
az role assignment list --assignee <principalId> --all -o table
az role assignment list --scope <resourceId> --include-inherited -o table
```

### 5.5 [Hard] What are deny assignments and how do they interact with role assignments?
**Answer**
Deny assignments explicitly block actions and take precedence over role assignments. You can't create them directly; they come from Azure Blueprints / deployment stacks / managed apps, and they can block even Owners.

**Explanation**
Evaluation: deny assignments first, then allow via role assignments (union of all applicable roles). `NotActions` is not a deny - it merely subtracts from that role's allowed set, while another role can still grant it. Azure ABAC (conditions on role assignments, e.g., blob tag/container conditions) can narrow data-plane roles further.

**Code / CLI / Bicep**
```bash
az role assignment list --scope <rid> --include-inherited -o table
az rest --method get --url "https://management.azure.com<scope>/providers/Microsoft.Authorization/denyAssignments?api-version=2022-04-01"
```

### 5.6 [Tricky] Key Vault: Access policies vs RBAC - what breaks when switching?
**Answer**
Vaults use either access policies or Azure RBAC as permission model; switching to RBAC makes existing access policies ineffective, so you must create equivalent role assignments (e.g., Key Vault Secrets User) first. RBAC is the recommended model.

**Explanation**
Under RBAC, `Key Vault Secrets User` reads secret values; `Key Vault Reader` only reads metadata; `Key Vault Contributor` manages the vault (control plane) with no secret data access. Key Vault references in Functions/App Service need the app's identity to have secret-read access (and `keyVaultReferenceIdentity` for UAMI).

**Code / CLI / Bicep**
```bash
az keyvault update -n myvault --enable-rbac-authorization true
az role assignment create --assignee-object-id <oid> --assignee-principal-type ServicePrincipal \
  --role "Key Vault Secrets User" --scope <vaultId>
```

**Gotchas / follow-ups**
- Contributor can read Storage keys/SB SAS keys -> sidesteps "no data role"; disable local/shared key auth (`disableLocalAuth`, `allowSharedKeyAccess`) to enforce Entra only.
- Role assignment count limits and orphaned assignments: verify in docs.
- Custom roles can supply exact `DataActions`.

**Interview traps**
- Owner != data access.
- NotActions is not a deny.
- Assigning at subscription "to be safe".

**Cheat sheet**
- Principal + role + scope; inheritance downward.
- Control plane roles (Owner/Contributor/Reader) vs data roles (`... Data ...`).
- Wait for propagation; restart to get fresh token.
- Deny > allow; use `principalType` and `guid()` in IaC.
- Disable key/SAS auth to force Entra.

---

## Topic 6: DefaultAzureCredential and credential chains

**Study notes**

`DefaultAzureCredential` (DAC) tries credentials in order and uses the first that returns a token. Typical .NET order (verify in current Azure.Identity docs; it changed across versions):

```
1 EnvironmentCredential          (AZURE_TENANT_ID / AZURE_CLIENT_ID / AZURE_CLIENT_SECRET|CERT)
2 WorkloadIdentityCredential     (AKS / federated; AZURE_FEDERATED_TOKEN_FILE)
3 ManagedIdentityCredential      (Azure compute; AZURE_CLIENT_ID selects UAMI)
4 VisualStudioCredential
5 VisualStudioCodeCredential     (may be excluded by default in newer versions - verify)
6 AzureCliCredential
7 AzurePowerShellCredential
8 AzureDeveloperCliCredential
9 InteractiveBrowserCredential   (excluded by default)
```

Control: `ExcludeXxxCredential` options, `ManagedIdentityClientId`, `TenantId`; newer versions support an `AZURE_TOKEN_CREDENTIALS` env var (e.g., `prod` / `dev` or a specific credential) - verify availability in your version.

### 6.1 [Easy] What is DefaultAzureCredential?
**Answer**
A credential that tries a sequence of authentication methods (env vars, workload identity, managed identity, then developer tools) so the same code works locally and in Azure. It's convenient but not deterministic.

**Explanation**
Create one instance and reuse it (token caching is per instance). It is fine for most apps, but production-hardened services often use a specific credential for predictability.

**Code / CLI / Bicep**
```csharp
builder.Services.AddAzureClients(c =>
{
    c.AddServiceBusClientWithNamespace("myns.servicebus.windows.net");
    c.AddBlobServiceClient(new Uri("https://acct.blob.core.windows.net"));
    c.UseCredential(new DefaultAzureCredential());
});
```
(Package `Microsoft.Extensions.Azure`.)

### 6.2 [Follow-up] How do you target a user-assigned identity with DAC?
**Answer**
Set the `AZURE_CLIENT_ID` environment variable to the UAMI's client ID, or set `ManagedIdentityClientId` in `DefaultAzureCredentialOptions`. Without it DAC uses the system-assigned identity.

**Explanation**
On Functions/App Service add it as an app setting. Note `AZURE_CLIENT_ID` is also read by EnvironmentCredential (needs secret too) and WorkloadIdentityCredential, so keep it consistent.

**Code / CLI / Bicep**
```bash
az functionapp config appsettings set -g rg -n myfunc --settings AZURE_CLIENT_ID=<uamiClientId>
```

### 6.3 [Medium] Why does it work locally but fail in Azure with "ManagedIdentityCredential authentication unavailable"?
**Answer**
Locally DAC falls through to your CLI/VS identity; in Azure it needs a managed identity actually enabled/attached to that host and the correct client ID. The error means no identity endpoint responded or the identity isn't assigned.

**Explanation**
Check: identity enabled; UAMI attached to the right resource/slot; `AZURE_CLIENT_ID` correct; the app isn't running somewhere without MI (e.g., a container with blocked IMDS, or Docker locally). The aggregated `CredentialUnavailableException` message lists each credential's failure; read it all.

**Code / CLI / Bicep**
```bash
az webapp identity show -g rg -n myapp
az functionapp identity show -g rg -n myfunc
```

### 6.4 [Medium] How do you make it deterministic and fast in production?
**Answer**
Use `ManagedIdentityCredential` directly in prod and a developer credential locally, or exclude unneeded DAC credentials. This avoids probing/timeouts, stray env-var credentials, and surprising identities.

**Explanation**
Each failed credential costs time (CLI/PowerShell credentials spawn processes; failures may take seconds). Probing can also cause slow cold starts. Pick by environment.

**Code / CLI / Bicep**
```csharp
TokenCredential credential = builder.Environment.IsDevelopment()
    ? new ChainedTokenCredential(new AzureCliCredential(), new VisualStudioCredential())
    : new ManagedIdentityCredential(ManagedIdentityId.FromUserAssignedClientId(clientId));

// or trim DAC
var dac = new DefaultAzureCredential(new DefaultAzureCredentialOptions
{
    ExcludeVisualStudioCodeCredential = true,
    ExcludeInteractiveBrowserCredential = true,
    ExcludeAzurePowerShellCredential = true,
    ManagedIdentityClientId = clientId
});
```

### 6.5 [Hard] A stale AZURE_CLIENT_SECRET in prod app settings caused access as a different principal. Explain.
**Answer**
EnvironmentCredential is first in the chain; if all three env vars (tenant, client id, secret) are present it authenticates as that service principal and never reaches ManagedIdentityCredential. Remove the vars or use an explicit credential.

**Explanation**
Symptom: 403s that don't match the MI's role assignments, or `AADSTS7000222` after expiry. Diagnose by decoding the token's `oid`/`appid` claim, or enabling Azure.Identity logging.

**Code / CLI / Bicep**
```csharp
using var listener = Azure.Core.Diagnostics.AzureEventSourceListener.CreateConsoleLogger();
var tok = await cred.GetTokenAsync(new TokenRequestContext(new[] { "https://storage.azure.com/.default" }), default);
// decode tok.Token at a trusted offline tool/jwt library and check oid/appid
```

### 6.6 [Tricky] Local dev with multiple tenants/accounts picks the wrong one. Fix?
**Answer**
Specify the tenant (`TenantId` / `VisualStudioTenantId`/`SharedTokenCacheTenantId`-style options, per credential) or sign in with `az login --tenant`. Also pin the credential you want via exclusions.

**Explanation**
VS and VS Code signed into several accounts; DAC can choose a different one than you expect. For CLI: `az account set`. Conditional Access/MFA may require `az login --tenant <id> --scope ...`.

**Code / CLI / Bicep**
```csharp
var cred = new DefaultAzureCredential(new DefaultAzureCredentialOptions { TenantId = "<tenantId>" });
```
```bash
az login --tenant <tenantId>
```

**Gotchas / follow-ups**
- `ChainedTokenCredential` caches the successful credential per instance in newer versions - verify; creating a new instance per request defeats token caching and hammers IMDS (throttling, HTTP 429).
- Use a singleton credential (and clients).
- Docker on dev machine: Azure CLI creds not visible in container; mount or use env/SP.

**Interview traps**
- "DAC uses managed identity first" - no; Environment first.
- New `DefaultAzureCredential()` per call.
- Assuming the chain order is stable across SDK versions.

**Cheat sheet**
- Env -> Workload Identity -> MI -> VS/VSCode/CLI/PS/azd.
- `AZURE_CLIENT_ID` = UAMI selector.
- Prod: explicit MI credential or trimmed chain.
- Singleton credential.

---

## Topic 7: Token acquisition internals (IMDS, App Service endpoint, caching)

**Study notes**

```
VM/AKS node/VMSS/Container*:  GET http://169.254.169.254/metadata/identity/oauth2/token
                                  ?api-version=2018-02-01&resource=https://vault.azure.net
                                  [&client_id=<uamiClientId> | &object_id= | &msi_res_id=]
                              Header: Metadata: true
App Service / Functions:      GET $IDENTITY_ENDPOINT?api-version=2019-08-01&resource=...[&client_id=...]
                              Header: X-IDENTITY-HEADER: $IDENTITY_HEADER
```
(Verify api-versions and parameter names such as `mi_res_id` in current docs.)

- Env vars `IDENTITY_ENDPOINT`/`IDENTITY_HEADER` are injected by the platform on App Service/Functions (older: `MSI_ENDPOINT`/`MSI_SECRET`).
- The platform's token service caches MI tokens (roughly 24h validity; expiry ~86400s) per resource URI; SDKs cache in memory and refresh shortly before expiry.
- Audience (`aud`) is derived from `resource` / `scope` (`https://x/.default`).

### 7.1 [Easy] How does code obtain a managed-identity token?
**Answer**
It calls a local, non-routable identity endpoint on the host (IMDS on VMs, a platform-injected endpoint on App Service/Functions), passing the target resource; Azure returns a JWT access token. SDKs hide this.

**Explanation**
No credentials are sent: the endpoint trusts that the request originates from the resource. App Service requires a secret header (`X-IDENTITY-HEADER`) to mitigate SSRF.

**Code / CLI / Bicep**
```bash
# inside an App Service Kudu console
curl -s -H "X-IDENTITY-HEADER: $IDENTITY_HEADER" \
 "$IDENTITY_ENDPOINT?api-version=2019-08-01&resource=https://vault.azure.net"
# inside a VM
curl -s -H "Metadata: true" "http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://storage.azure.com/"
```

### 7.2 [Follow-up] How is the token cached and when does it refresh?
**Answer**
Azure.Identity caches tokens in memory per credential instance and refreshes before expiry; the MI backend additionally caches tokens for about 24 hours. So per-request credential creation or manual calls are wasteful.

**Explanation**
Implication: role/group changes can take up to token lifetime to reflect for claims baked into the token, though RBAC itself is evaluated server-side. Continuous Access Evaluation (CAE) applies to some user flows; MI tokens aren't revoked on demand.

**Code / CLI / Bicep**
```csharp
// Singleton: do this once
services.AddSingleton<TokenCredential>(_ => new ManagedIdentityCredential());
var token = await cred.GetTokenAsync(new TokenRequestContext(new[] { "https://servicebus.azure.net/.default" }), ct);
Console.WriteLine(token.ExpiresOn);
```

### 7.3 [Medium] What are the resource/scope values for common services?
**Answer**
Use `<resource>/.default` scopes: Storage `https://storage.azure.com/.default`, Key Vault `https://vault.azure.net/.default`, Service Bus/Event Hubs `https://servicebus.azure.net/.default` / `https://eventhubs.azure.net/.default`, SQL `https://database.windows.net/.default`, ARM `https://management.azure.com/.default`, Redis `https://redis.azure.com/.default`. Verify in current docs (sovereign clouds differ).

**Explanation**
Wrong audience -> 401 `InvalidAudience`. SDKs set the scope for you; you only choose it with raw `HttpClient` calls.

**Code / CLI / Bicep**
```csharp
var cred = new DefaultAzureCredential();
var t = await cred.GetTokenAsync(new TokenRequestContext(new[] { "https://management.azure.com/.default" }));
using var http = new HttpClient();
http.DefaultRequestHeaders.Authorization = new("Bearer", t.Token);
```

### 7.4 [Hard] You see HTTP 429 / timeouts from IMDS under load. Why and what to do?
**Answer**
Too many token requests (new credential per call, or cold-start storms) get throttled by IMDS/identity endpoint. Reuse a singleton credential so SDK caching works, and rely on retries with backoff.

**Explanation**
Another cause: DAC probing in containers with blocked IMDS access. In AKS, pod network policies or the node's IMDS restrictions can block 169.254.169.254; use workload identity instead. The MI backend retry guidance: retry 429/5xx with exponential backoff (verify in docs).

**Code / CLI / Bicep**
```csharp
var options = new ManagedIdentityCredentialOptions(ManagedIdentityId.SystemAssigned);
options.Retry.MaxRetries = 5;
options.Retry.Mode = Azure.Core.RetryMode.Exponential;
var cred = new ManagedIdentityCredential(options);   // verify type/shape in your Azure.Identity version
```

### 7.5 [Tricky] Decode a token: what do you check to debug 401/403?
**Answer**
Check `aud` (target resource), `iss`/`tid` (tenant), `oid` (principal), `appid`/`azp`, `ver` (v1/v2), `exp`, and for your own APIs `scp`/`roles`. A mismatched `aud` or unexpected `oid` explains most failures.

**Explanation**
Never paste production tokens into untrusted online decoders; use local tools or `System.IdentityModel.Tokens.Jwt`.

**Code / CLI / Bicep**
```csharp
var jwt = new System.IdentityModel.Tokens.Jwt.JwtSecurityTokenHandler().ReadJwtToken(token);
foreach (var c in jwt.Claims.Where(c => new[]{"aud","iss","oid","tid","appid","azp","scp","roles","ver"}.Contains(c.Type)))
    Console.WriteLine($"{c.Type}={c.Value}");
```

**Gotchas / follow-ups**
- Don't call IMDS directly in code unless needed; the SDK handles it.
- Changing identity assignment on a resource may not take effect until old cached token expires (verify).

**Interview traps**
- "The token contains my RBAC roles".
- Newing up credentials per request.
- Using `resource` (v1) vs `scope` (v2) incorrectly.

**Cheat sheet**
- IMDS 169.254.169.254 + `Metadata: true`; App Service `IDENTITY_ENDPOINT` + `X-IDENTITY-HEADER`.
- Token ~24h backend cache; SDK caches in memory.
- Audience mismatch -> 401.
- Decode `aud`, `oid`, `tid`, `ver`.

---

## Topic 8: Using identity with Azure services (Service Bus, Blob, Key Vault, SQL, Redis, Event Hubs, App Config, Cosmos)

**Study notes**

| Service | Constructor with identity | Role / permission |
|---|---|---|
| Service Bus | `new ServiceBusClient("ns.servicebus.windows.net", cred)` | Azure Service Bus Data Sender / Receiver / Owner |
| Blob | `new BlobServiceClient(uri, cred)` | Storage Blob Data Reader/Contributor/Owner |
| Key Vault | `new SecretClient(uri, cred)` | Key Vault Secrets User (RBAC) |
| Event Hubs | `new EventHubProducerClient(fqns, hub, cred)` | Azure Event Hubs Data Sender/Receiver/Owner |
| App Configuration | `options.Connect(new Uri(endpoint), cred)` | App Configuration Data Reader |
| Cosmos DB | `new CosmosClient(endpoint, cred)` | Cosmos SQL data-plane roles (not ARM RBAC) |
| Azure SQL | connection string `Authentication=Active Directory Default/Managed Identity` | Contained DB user + DB roles |
| Redis | Entra auth with data access policy | Redis data access policies |

### 8.1 [Easy] How do you connect to Service Bus without a connection string?
**Answer**
Use the fully qualified namespace plus a TokenCredential, and assign the identity Azure Service Bus Data Sender/Receiver. No SAS key needed.

**Explanation**
Optionally disable SAS (`disableLocalAuth: true`) so only Entra works.

**Code / CLI / Bicep**
```csharp
await using var client = new ServiceBusClient("myns.servicebus.windows.net", new DefaultAzureCredential());
var sender = client.CreateSender("orders");
await sender.SendMessageAsync(new ServiceBusMessage("hello"));
var receiver = client.CreateReceiver("orders");
var msg = await receiver.ReceiveMessageAsync();
await receiver.CompleteMessageAsync(msg);
```

### 8.2 [Follow-up] Blob and Key Vault with identity?
**Answer**
Pass a credential to the client with the account/vault URL; ensure data-plane roles (Blob Data Contributor, Key Vault Secrets User). For storage you can disable shared-key access.

**Explanation**
Generating user-delegation SAS requires `Storage Blob Delegator` action (included in Blob Data roles at account/ container-wide scope per docs; verify) and a user delegation key.

**Code / CLI / Bicep**
```csharp
var blob = new BlobServiceClient(new Uri("https://acct.blob.core.windows.net"), cred);
var container = blob.GetBlobContainerClient("data");
await container.GetBlobClient("a.txt").UploadAsync(BinaryData.FromString("hi"), overwrite: true);

var kv = new SecretClient(new Uri("https://myvault.vault.azure.net"), cred);
var secret = (await kv.GetSecretAsync("name")).Value.Value;
```

### 8.3 [Medium] Azure SQL with managed identity - what are the steps?
**Answer**
Set an Entra admin on the SQL server, connect as an Entra user, create a contained user for the identity (`CREATE USER [name] FROM EXTERNAL PROVIDER`), grant DB roles, and use `Authentication=Active Directory Managed Identity` (or `Active Directory Default`) in the connection string. Authorization inside SQL is T-SQL, not Azure RBAC.

**Explanation**
The user name is the SAMI (resource name) or the UAMI name. For UAMI add `User Id=<clientId>` in the connection string. `Active Directory Default` uses a DAC-like chain (Microsoft.Data.SqlClient).

**Code / CLI / Bicep**
```sql
CREATE USER [id-orders-prod] FROM EXTERNAL PROVIDER;
ALTER ROLE db_datareader ADD MEMBER [id-orders-prod];
ALTER ROLE db_datawriter ADD MEMBER [id-orders-prod];
```
```csharp
var cs = "Server=tcp:srv.database.windows.net;Database=db;Authentication=Active Directory Managed Identity;User Id=<uamiClientId>;Encrypt=True";
using var conn = new Microsoft.Data.SqlClient.SqlConnection(cs);
await conn.OpenAsync();
```

### 8.4 [Medium] Cosmos DB and Redis with Entra - what's different?
**Answer**
Both use their own data-plane authorization: Cosmos uses Cosmos RBAC role assignments (created with `az cosmosdb sql role assignment`), and Redis uses data access policies (Data Owner/Contributor/Reader) with the principal as the user. Azure ARM roles like Contributor don't grant Cosmos data access.

**Explanation**
Cosmos built-in role IDs: `00000000-0000-0000-0000-000000000001` Data Reader, `...0002` Data Contributor (verify). Redis: Entra auth must be enabled on the cache; the username is the principal's object ID and the "password" is the token, re-authenticated before expiry (the `Microsoft.Azure.StackExchangeRedis` library handles this).

**Code / CLI / Bicep**
```bash
az cosmosdb sql role assignment create -g rg -a mycosmos \
  -s "/" -p <principalId> -d 00000000-0000-0000-0000-000000000002
```
```csharp
var cfg = ConfigurationOptions.Parse("mycache.redis.cache.windows.net:6380");
await cfg.ConfigureForAzureWithTokenCredentialAsync(new DefaultAzureCredential()); // Microsoft.Azure.StackExchangeRedis
var mux = await ConnectionMultiplexer.ConnectAsync(cfg);
```
(Verify API names and policy assignment resource types in current docs.)

### 8.5 [Tricky] App Configuration + Key Vault references with identity
**Answer**
The app connects to App Configuration via credential and resolves Key Vault references using the same (or specified) credential, so the identity needs App Configuration Data Reader plus Key Vault Secrets User.

**Explanation**
Common bug: App Config role assigned but no Key Vault role -> refs fail at startup with 403. In Functions/App Service the platform-level `@Microsoft.KeyVault(...)` reference uses the app's identity (`keyVaultReferenceIdentity` for UAMI).

**Code / CLI / Bicep**
```csharp
builder.Configuration.AddAzureAppConfiguration(o =>
    o.Connect(new Uri("https://myappconfig.azconfig.io"), cred)
     .ConfigureKeyVault(kv => kv.SetCredential(cred)));
```

**Gotchas / follow-ups**
- SQL: forgetting contained user -> "Login failed for user '<token-identified principal>'".
- SQL: AD admin must create users; MI can't create other external users without Graph permissions (Directory Readers equivalents; verify).
- Event Hubs checkpoint store (Blob) also needs Blob data roles.
- Disable local auth where possible.

**Interview traps**
- Thinking Azure RBAC governs SQL/Cosmos/Redis data.
- Forgetting separate Blob role for Event Hubs/Functions checkpoints.

**Cheat sheet**
- FQDN + credential everywhere; no keys.
- SQL: `CREATE USER ... FROM EXTERNAL PROVIDER`.
- Cosmos: own SQL role assignments; Redis: data access policies.
- KV refs need KV role.

---

## Topic 9: Functions / Logic Apps identity-based connections

**Study notes**

Azure Functions binds triggers/bindings to a *connection name* prefix. Instead of a connection string, define settings with the prefix plus suffixes:

```
<Conn>__fullyQualifiedNamespace = myns.servicebus.windows.net    (Service Bus / Event Hubs)
<Conn>__credential              = managedidentity                  (use MI; omit => default identity behaviour)
<Conn>__clientId                = <UAMI clientId>                  (selects a UAMI; omit => system-assigned)
```
Storage host (`AzureWebJobsStorage`): `AzureWebJobsStorage__accountName` (or `__blobServiceUri`/`__queueServiceUri`/`__tableServiceUri`) plus `__credential`/`__clientId`. Not every extension/plan supports identity connections; verify per-extension in current docs.

Required roles (typical): Service Bus trigger -> Azure Service Bus Data Receiver; output -> Data Sender; host storage -> Storage Blob Data Owner + Storage Queue Data Contributor + Storage Table Data Contributor (verify exact set for your triggers).

### 9.1 [Easy] How does a Service Bus-triggered Function use managed identity?
**Answer**
Replace the connection string app setting with `<Conn>__fullyQualifiedNamespace` and give the Function's identity the Service Bus Data Receiver role. The binding's `Connection = "<Conn>"` stays the same.

**Explanation**
The Functions host builds the client with a managed identity credential. For UAMI add `<Conn>__clientId` and `<Conn>__credential=managedidentity`.

**Code / CLI / Bicep**
```csharp
[Function("ProcessOrder")]
public async Task Run(
    [ServiceBusTrigger("orders", Connection = "SbConn")] ServiceBusReceivedMessage msg,
    ServiceBusMessageActions actions)
{
    await actions.CompleteMessageAsync(msg);
}
```
```bash
az functionapp config appsettings set -g rg -n myfunc --settings \
  SbConn__fullyQualifiedNamespace=myns.servicebus.windows.net \
  SbConn__credential=managedidentity \
  SbConn__clientId=<uamiClientId>
```

### 9.2 [Follow-up] How do you move AzureWebJobsStorage to identity?
**Answer**
Delete the `AzureWebJobsStorage` connection string, set `AzureWebJobsStorage__accountName` (and `__credential`/`__clientId` for UAMI), and assign the identity the blob/queue/table data roles. Some hosting modes (e.g., classic Consumption with Azure Files content share) still need a connection string for `WEBSITE_CONTENTAZUREFILECONNECTIONSTRING` - verify current support.

**Explanation**
Missing a role typically surfaces as host startup errors or triggers not firing ("This request is not authorized to perform this operation using this permission"). Durable Functions / Event Hubs triggers need their own role sets.

**Code / CLI / Bicep**
```bicep
resource app 'Microsoft.Web/sites@2023-12-01' = {
  name: 'myfunc'
  location: location
  kind: 'functionapp'
  identity: { type: 'UserAssigned', userAssignedIdentities: { '${uami.id}': {} } }
  properties: {
    serverFarmId: plan.id
    siteConfig: {
      appSettings: [
        { name: 'AzureWebJobsStorage__accountName', value: storage.name }
        { name: 'AzureWebJobsStorage__credential', value: 'managedidentity' }
        { name: 'AzureWebJobsStorage__clientId', value: uami.properties.clientId }
        { name: 'SbConn__fullyQualifiedNamespace', value: '${sb.name}.servicebus.windows.net' }
        { name: 'SbConn__credential', value: 'managedidentity' }
        { name: 'SbConn__clientId', value: uami.properties.clientId }
        { name: 'FUNCTIONS_WORKER_RUNTIME', value: 'dotnet-isolated' }
        { name: 'FUNCTIONS_EXTENSION_VERSION', value: '~4' }
      ]
    }
  }
}
```

### 9.3 [Medium] Logic Apps: how do you use managed identity?
**Answer**
Enable SAMI/UAMI on the Logic App, then choose "Managed identity" authentication on connectors or HTTP actions, and grant that identity the needed roles. Consumption and Standard differ in where connections are configured.

**Explanation**
HTTP action example: auth type `ManagedServiceIdentity` with an `audience`. Standard built-in (service provider) connectors configure auth in `connections.json`; managed (API) connections have their own auth type setting. For UAMI you pick which identity in the connection/identity block. Verify JSON shapes in current docs.

**Code / CLI / Bicep**
```json
{
  "type": "Http",
  "inputs": {
    "method": "GET",
    "uri": "https://myvault.vault.azure.net/secrets/name?api-version=7.4",
    "authentication": {
      "type": "ManagedServiceIdentity",
      "audience": "https://vault.azure.net"
    }
  }
}
```
For a UAMI add `"identity": "<uamiResourceId>"` inside `authentication` (verify).

### 9.4 [Medium] Key Vault references in app settings
**Answer**
Use `@Microsoft.KeyVault(SecretUri=...)` or `@Microsoft.KeyVault(VaultName=...;SecretName=...)`; the platform resolves them with the app's identity (needs Secrets User). For UAMI set the site property `keyVaultReferenceIdentity`.

**Explanation**
Unresolved references show as the literal string and flag a status in the portal. Soft-delete/network firewall on vault can block. Prefer identity-based connections over KV-referenced connection strings where supported.

**Code / CLI / Bicep**
```bash
az resource update --ids <siteResourceId> --set properties.keyVaultReferenceIdentity=<uamiResourceId>
az functionapp config appsettings set -g rg -n myfunc --settings "DbPwd=@Microsoft.KeyVault(VaultName=myvault;SecretName=dbpwd)"
```

### 9.5 [Tricky] Function with SAMI works in prod but not on the staging slot.
**Answer**
Each deployment slot has its own system-assigned identity (and app-setting stickiness), so roles assigned to the production slot's identity don't apply to the slot. Assign roles to the slot identity or use a shared UAMI.

**Explanation**
After swap, identity does NOT move with the code; the settings marked "slot setting" stay with the slot. Mark `__clientId`/`__fullyQualifiedNamespace` slot-specific as needed so test slots don't read production queues.

**Code / CLI / Bicep**
```bash
az functionapp identity assign -g rg -n myfunc --slot staging
az functionapp config appsettings set -g rg -n myfunc --slot staging --slot-settings SbConn__fullyQualifiedNamespace=stg.servicebus.windows.net
```

**Gotchas / follow-ups**
- Service Bus scale-controller on Consumption/Premium needs the identity to have rights too (Receiver role) - scaling issues = missing role.
- Changing app settings restarts the app.
- Local dev: use `local.settings.json` with the same keys; DAC resolves your CLI identity.

**Interview traps**
- Using `__connectionString` suffix by habit.
- Forgetting `__clientId` with UAMI (looks for SAMI).
- Not granting host-storage roles.

**Cheat sheet**
- `<Conn>__fullyQualifiedNamespace` + `__credential` + `__clientId`.
- Storage: `AzureWebJobsStorage__accountName`.
- KV refs: `keyVaultReferenceIdentity` for UAMI.
- Slots have own SAMI.

---

## Topic 10: OAuth 2.0 / OIDC flows in Entra

**Study notes**

OAuth 2.0 = authorization (access tokens); OIDC = authentication layer on top (ID token). Endpoints (v2): `https://login.microsoftonline.com/{tenant}/oauth2/v2.0/authorize` and `/token`.

| Flow | Use | Who is the subject |
|---|---|---|
| Authorization code + PKCE | Web apps, SPAs, mobile/desktop (public clients: must use PKCE) | User (delegated) |
| Client credentials | Daemon/service-to-service | App (app-only; `roles`) |
| On-behalf-of (OBO) | API calls downstream API as the calling user | User (delegated) |
| Device code | Input-constrained devices/CLI | User |
| Implicit / ROPC | Legacy; avoid | |

Tokens: **ID token** (who signed in; for the client), **access token** (for the API; client shouldn't parse), **refresh token** (long-lived, public/confidential clients).

### 10.1 [Easy] Which flow for a daemon/background service calling an API?
**Answer**
Client credentials: the app authenticates with its own credential and gets an app-only token whose permissions are app roles (`roles` claim). Use the `/.default` scope.

**Explanation**
No user, no consent screen; an admin grants application permissions. On Azure prefer managed identity over client secret for this.

**Code / CLI / Bicep**
```csharp
var app = ConfidentialClientApplicationBuilder.Create(clientId)
    .WithClientSecret(secret)   // or WithCertificate(cert)
    .WithAuthority($"https://login.microsoftonline.com/{tenantId}")
    .Build();
var r = await app.AcquireTokenForClient(new[] { "api://my-api/.default" }).ExecuteAsync();
```

### 10.2 [Follow-up] Why PKCE and what does it protect?
**Answer**
PKCE binds the authorization code to the client that started the request (code_verifier/code_challenge), preventing stolen-code interception for public clients that can't hold a secret. It is recommended for all clients, now.

**Explanation**
SPAs use code + PKCE with CORS to the token endpoint (not implicit). The redirect URI must be exactly registered (type SPA vs Web matters).

**Code / CLI / Bicep**
```bash
# authorize request (conceptual)
GET /authorize?client_id=...&response_type=code&redirect_uri=...&scope=openid%20api://my-api/access_as_user
    &code_challenge=<sha256-b64url>&code_challenge_method=S256&state=...
```

### 10.3 [Medium] Explain on-behalf-of (OBO).
**Answer**
API A receives a user's access token (audience = A) and exchanges it with Entra for a new token for API B, preserving the user identity. It requires A to be a confidential client and consent for B's scope.

**Explanation**
Delegated permissions only. Failure `AADSTS65001` means consent missing (use `knownClientApplications` or pre-authorize). Chains > 1 hop need care. Conditional Access claims challenges may be returned (`claims` challenge) and must be propagated to the client.

**Code / CLI / Bicep**
```csharp
var result = await app.AcquireTokenOnBehalfOf(new[] { "api://downstream/.default" },
    new UserAssertion(incomingAccessToken)).ExecuteAsync();
```
(With Microsoft.Identity.Web use `ITokenAcquisition.GetAccessTokenForUserAsync`/`IDownstreamApi`.)

### 10.4 [Medium] Delegated vs application permissions?
**Answer**
Delegated permissions (scopes, `scp`) act on behalf of a signed-in user and are limited by both the app and the user's rights; application permissions (app roles, `roles`) let the app act without a user and always need admin consent. Application permissions are usually broader and riskier.

**Explanation**
Least privilege: choose delegated where a user exists. For Graph, application permissions like `User.Read.All` apply tenant-wide.

**Code / CLI / Bicep**
```bash
az ad app permission add --id <appId> --api 00000003-0000-0000-c000-000000000000 \
  --api-permissions <permissionGuid>=Role
az ad app permission admin-consent --id <appId>
```

### 10.5 [Hard] Device code flow: when and what are the risks?
**Answer**
For devices/CLIs without a browser: user visits a URL and enters a code. It's phishable (device-code phishing), so many tenants block it via Conditional Access.

**Explanation**
Don't use it for general apps. CA policy "authentication flows" can block device code flow. Prefer interactive browser/broker (WAM) auth.

**Code / CLI / Bicep**
```csharp
var pca = PublicClientApplicationBuilder.Create(clientId).WithAuthority(AzureCloudInstance.AzurePublic, tenantId).Build();
var r = await pca.AcquireTokenWithDeviceCode(new[] { "User.Read" }, dc => { Console.WriteLine(dc.Message); return Task.CompletedTask; }).ExecuteAsync();
```

### 10.6 [Tricky] ID token vs access token - can the client send the ID token to the API?
**Answer**
No. ID token is for the client (audience = client app) to learn the user's identity; the API must receive an access token with audience = API. APIs validating an ID token is an anti-pattern.

**Explanation**
Access tokens for Graph are opaque-ish to clients; don't parse. Audience, issuer, scope are validated by the API.

**Code / CLI / Bicep**
```bash
curl -H "Authorization: Bearer $ACCESS_TOKEN" https://api.contoso.com/orders
```

**Gotchas / follow-ups**
- Refresh-token lifetimes and CA sign-in frequency affect long-lived sessions.
- `/common`, `/organizations`, `/consumers` authorities differ; confidential single-tenant apps should use the tenant ID.
- Redirect URI mismatch -> AADSTS50011.

**Interview traps**
- Implicit flow recommended for SPAs (no).
- Sending ID token to an API.
- Using client credentials when a user context is needed.

**Cheat sheet**
- Code+PKCE = user sign-in; client credentials = daemon; OBO = API->API with user; device = no-browser.
- `scp` delegated, `roles` application.
- App-only scope = `<resource>/.default`.

---

## Topic 11: Securing your own ASP.NET Core API (Microsoft.Identity.Web)

**Study notes**

```
Client --Bearer JWT--> API (AddMicrosoftIdentityWebApi)
                         validates: signature (Entra keys), issuer, audience, lifetime
                         then authorization: scope (scp) for delegated, app role (roles) for app-only
```

- **Scopes** (`scp`): delegated permissions defined under "Expose an API".
- **App roles** (`roles`): defined in app manifest; assigned to users/groups/apps; used for app-only tokens or role-based user access.
- **Audience:** v2 tokens: `aud` = API client ID (GUID); v1 tokens: `aud` = Application ID URI (e.g., `api://guid`). Set `requestedAccessTokenVersion` (manifest: `accessTokenAcceptedVersion` / newer `requestedAccessTokenVersion` in API settings - verify) to 2.

### 11.1 [Easy] Minimal setup to protect an API with Entra.
**Answer**
Register the API, expose a scope/app role, add `Microsoft.Identity.Web`, call `AddMicrosoftIdentityWebApi` with the `AzureAd` config section, and add `[Authorize]`. Clients get tokens for the API's scope.

**Explanation**
Config includes Instance, TenantId, ClientId, and Audience (when it differs from ClientId, e.g., `api://...`).

**Code / CLI / Bicep**
```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApi(builder.Configuration.GetSection("AzureAd"));
builder.Services.AddAuthorization();
var app = builder.Build();
app.UseAuthentication(); app.UseAuthorization();
app.MapControllers();
```
```json
{ "AzureAd": { "Instance": "https://login.microsoftonline.com/", "TenantId": "<tenant>", "ClientId": "<api-client-id>", "Audience": "api://<api-client-id>" } }
```

### 11.2 [Follow-up] Scopes vs app roles in the API code?
**Answer**
Check scopes for delegated user tokens with `[RequiredScope]` and app roles for app-only (and role-based) tokens with `[Authorize(Roles=...)]` or policies. If both caller types exist, use `[RequiredScopeOrAppPermission]`.

**Explanation**
A client-credentials token has no `scp`, only `roles`; a user token has `scp`. Using only `RequiredScope` rejects daemon callers.

**Code / CLI / Bicep**
```csharp
[Authorize]
[ApiController, Route("orders")]
public class OrdersController : ControllerBase
{
    [HttpGet, RequiredScope("Orders.Read")]
    public IActionResult Get() => Ok();

    [HttpPost, Authorize(Roles = "Orders.Write")]  // app role
    public IActionResult Post() => Accepted();
}
```

### 11.3 [Medium] v1 vs v2 tokens and why audience validation fails
**Answer**
v1 tokens have `iss` `https://sts.windows.net/{tid}/` and `aud` = App ID URI; v2 tokens have `iss` `https://login.microsoftonline.com/{tid}/v2.0` and `aud` = client ID GUID. If the API accepts the other version/audience than configured you get 401 IDX10206/IDX10214 audience/issuer errors.

**Explanation**
The version is decided by the API's `requestedAccessTokenVersion` setting (default 1 for single-tenant registrations created in older flows; verify), not by the endpoint called by the client. Configure `ValidAudiences` if needed.

**Code / CLI / Bicep**
```csharp
builder.Services.Configure<JwtBearerOptions>(JwtBearerDefaults.AuthenticationScheme, o =>
{
    o.TokenValidationParameters.ValidAudiences = new[] { "api://<id>", "<id>" };
});
```

### 11.4 [Medium] How do you accept a managed identity calling your API?
**Answer**
Define an app role on the API, then assign it to the MI's service principal (an app role assignment via Graph); the MI requests a token for the API's App ID URI and the API checks the `roles` claim.

**Explanation**
You can't assign app roles to MIs in the portal UI historically; use Graph/CLI/PowerShell. Validate `roles` and also optionally the caller `oid`.

**Code / CLI / Bicep**
```bash
az rest --method POST \
  --url "https://graph.microsoft.com/v1.0/servicePrincipals/<miPrincipalId>/appRoleAssignedTo" \
  --body '{"principalId":"<miPrincipalId>","resourceId":"<apiSpObjectId>","appRoleId":"<appRoleGuid>"}'
```
```csharp
var token = await new ManagedIdentityCredential().GetTokenAsync(new TokenRequestContext(new[] { "api://<api-client-id>/.default" }));
```

### 11.5 [Hard] Why is "any valid token from my tenant" a vulnerability?
**Answer**
If the API validates signature and issuer but not that the caller is permitted (roles/scopes or assignment), any app/user in the tenant that can get a token for the API can call it. Enforce `scp`/`roles` checks or require assignment on the enterprise app ("Assignment required") and, for app-only, check app roles.

**Explanation**
Also multi-tenant issuer validation must restrict allowed tenants; accepting `/common` issuers without an allow-list lets foreign tenants in. Tokens for app-only flows without `roles` are accepted if you rely only on `[Authorize]` - check `roles` is present.

**Code / CLI / Bicep**
```csharp
builder.Services.AddAuthorization(o =>
    o.AddPolicy("AppOnlyWrite", p => p.RequireClaim("roles", "Orders.Write")));
```
```bash
az ad sp update --id <apiAppId> --set appRoleAssignmentRequired=true
```

### 11.6 [Tricky] User token has groups claim missing / overage. What do you do?
**Answer**
When a user is in too many groups the token omits `groups` and includes `_claim_names`; call Graph (`/me/getMemberObjects` or `checkMemberGroups`) or, better, use app roles instead of group IDs.

**Explanation**
Microsoft.Identity.Web can handle overage with `AddMicrosoftGraph` and a claims transformation. Alternatively configure groups claim to "groups assigned to the application".

**Code / CLI / Bicep**
```csharp
var groups = User.FindAll("groups").Select(c => c.Value).ToList();
if (!groups.Any() && User.HasClaim(c => c.Type == "_claim_names")) { /* call Graph checkMemberGroups */ }
```

**Gotchas / follow-ups**
- `MapInboundClaims` renames claims (`roles` stays; `scp` -> `http://schemas.microsoft.com/identity/claims/scope`); Microsoft.Identity.Web sets this up for you.
- Clock skew defaults to 5 minutes.
- `AllowedToAcceptTokenFromTenants`-style allow-lists for multi-tenant (verify names).

**Interview traps**
- Treating `[Authorize]` as sufficient.
- Mixing v1/v2 audience.
- Role claims for users vs apps confusion.

**Cheat sheet**
- `AddMicrosoftIdentityWebApi`, `[RequiredScope]`, `[Authorize(Roles)]`.
- Delegated -> `scp`; app-only -> `roles`.
- v2 aud = GUID; v1 aud = App ID URI.
- Require assignment/roles; don't trust any token.

---

## Topic 12: MSAL and the token cache

**Study notes**

- **MSAL.NET** (`Microsoft.Identity.Client`) obtains tokens for your own app flows; **Azure.Identity** is for Azure SDK credentials (built on MSAL).
- `PublicClientApplication` (desktop/mobile) vs `ConfidentialClientApplication` (web/API/daemon).
- Always call `AcquireTokenSilent` first for user flows; fall back to interactive on `MsalUiRequiredException`.
- `AcquireTokenForClient` uses an internal app token cache automatically - don't build your own cache layer on top (verify in docs).
- Token cache: in-memory default; for web apps use Microsoft.Identity.Web distributed token caches (Redis/SQL/Cosmos) so multi-instance apps share user tokens.

### 12.1 [Easy] What does MSAL do for you?
**Answer**
It handles protocol flows, token caching, refresh, and silent renewal for Entra tokens. You request scopes and MSAL returns a cached or fresh token.

**Explanation**
It also supports WAM broker, PoP, claims challenges and CAE.

**Code / CLI / Bicep**
```csharp
var accounts = await pca.GetAccountsAsync();
try { r = await pca.AcquireTokenSilent(scopes, accounts.FirstOrDefault()).ExecuteAsync(); }
catch (MsalUiRequiredException) { r = await pca.AcquireTokenInteractive(scopes).ExecuteAsync(); }
```

### 12.2 [Follow-up] Where is the token cache stored and how do you scale web apps?
**Answer**
By default in memory per app instance, lost on restart and not shared across scale-out. Use a distributed cache (Redis, SQL, Cosmos) via Microsoft.Identity.Web token cache providers.

**Explanation**
Web apps signing in users store per-user tokens keyed by account; without a shared cache, users bounce to re-login after restarts/scale. Secure the cache (it holds refresh tokens).

**Code / CLI / Bicep**
```csharp
builder.Services.AddMicrosoftIdentityWebApiAuthentication(builder.Configuration)
    .EnableTokenAcquisitionToCallDownstreamApi()
    .AddDistributedTokenCaches();
builder.Services.AddStackExchangeRedisCache(o => o.Configuration = "mycache.redis.cache.windows.net:6380,ssl=true,...");
```
(For Redis with Entra auth, see Topic 8; verify provider packages.)

### 12.3 [Medium] Managed identity in MSAL?
**Answer**
MSAL.NET has `ManagedIdentityApplicationBuilder` so you can acquire MI tokens with the same caching behaviour, though most Azure workloads just use Azure.Identity.

**Explanation**
Useful for calling your own API with MI. It handles IMDS/App Service endpoints. Verify the MSAL version required.

**Code / CLI / Bicep**
```csharp
var mi = ManagedIdentityApplicationBuilder.Create(ManagedIdentityId.SystemAssigned).Build();
var r = await mi.AcquireTokenForManagedIdentity("api://<api>").ExecuteAsync();   // resource, not .default
```

### 12.4 [Hard] Client credential token requested on every call -> throttling. Why?
**Answer**
A new `ConfidentialClientApplication` per request means an empty cache so each call hits Entra. Create the app once (singleton) and call AcquireTokenForClient; MSAL returns cached tokens until near expiry.

**Explanation**
Entra throttles heavy token endpoint use (`429`, `AADSTS` throttle messages). Same applies to creating `DefaultAzureCredential` per call.

**Code / CLI / Bicep**
```csharp
services.AddSingleton<IConfidentialClientApplication>(_ =>
    ConfidentialClientApplicationBuilder.Create(clientId).WithCertificate(cert).WithAuthority(authority).Build());
```

### 12.5 [Tricky] MsalUiRequiredException / interaction_required in an API OBO call
**Answer**
Usually MFA/CA, consent, or token revocation needs user interaction; an API can't show UI, so return a 401/403 with a claims challenge so the client re-acquires a token interactively.

**Explanation**
Microsoft.Identity.Web has `MicrosoftIdentityWebChallengeUserException` handling and `[AuthorizeForScopes]` for web apps. For APIs, return `WWW-Authenticate` with the `claims` parameter.

**Code / CLI / Bicep**
```csharp
catch (MsalUiRequiredException ex)
{
    return Unauthorized(new { error = ex.ErrorCode, claims = ex.Claims });
}
```

**Gotchas / follow-ups**
- Token cache serialization must be thread-safe.
- Don't log tokens.
- ADAL is deprecated; migrate to MSAL.

**Interview traps**
- Building a new MSAL client per request.
- Not calling silent first.
- Confusing Azure.Identity vs MSAL roles.

**Cheat sheet**
- Singleton MSAL apps; silent first.
- Distributed cache for scale-out.
- `AcquireTokenForClient` auto-caches.
- UI-required in API -> claims challenge.

---

## Topic 13: Workload identity federation (GitHub Actions OIDC, AKS workload identity) vs secrets

**Study notes**

Federation lets an external IdP's token (GitHub, Kubernetes service account, Google...) be exchanged for an Entra access token, so no secret is stored.

```
GitHub Actions job --OIDC JWT (iss=token.actions.githubusercontent.com, sub=repo:org/repo:ref:refs/heads/main)-->
Entra: federated credential on app reg or UAMI matches issuer+subject+audience --> access token
```
- Federated identity credential (FIC) on an **app registration** or a **UAMI**: `issuer`, `subject`, `audiences` (`api://AzureADTokenExchange`).
- Subject must match EXACTLY (no wildcards in the classic model; flexible matching exists for app registrations in newer features - verify).
- AKS workload identity: pod uses a projected service-account token (OIDC issuer of the cluster); SA annotated with the client ID; `WorkloadIdentityCredential` in Azure.Identity (included in DAC).

### 13.1 [Easy] Why use federation for CI/CD?
**Answer**
There's no client secret to store, leak, or rotate - GitHub proves the job's identity via a short-lived OIDC token that Entra trusts through a federated credential. Access is scoped by repo/branch/environment.

**Explanation**
Replaces `AZURE_CREDENTIALS` secrets. The workflow needs `permissions: id-token: write`.

**Code / CLI / Bicep**
```yaml
permissions:
  id-token: write
  contents: read
steps:
  - uses: azure/login@v2
    with:
      client-id: ${{ vars.AZURE_CLIENT_ID }}
      tenant-id: ${{ vars.AZURE_TENANT_ID }}
      subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
```

### 13.2 [Follow-up] How do you create the federated credential?
**Answer**
Add a federated credential to an app registration or UAMI with the GitHub issuer and a subject like `repo:org/repo:environment:prod`. Then give that identity RBAC on the target scope.

**Explanation**
Subject formats: `repo:ORG/REPO:ref:refs/heads/main`, `...:environment:NAME`, `...:pull_request`, `...:ref:refs/tags/x`.

**Code / CLI / Bicep**
```bicep
resource uami 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' existing = { name: 'id-deploy' }
resource fic 'Microsoft.ManagedIdentity/userAssignedIdentities/federatedIdentityCredentials@2023-01-31' = {
  parent: uami
  name: 'github-prod'
  properties: {
    issuer: 'https://token.actions.githubusercontent.com'
    subject: 'repo:contoso/orders:environment:prod'
    audiences: [ 'api://AzureADTokenExchange' ]
  }
}
```

### 13.3 [Medium] AKS workload identity - how do the pieces connect?
**Answer**
Enable the cluster OIDC issuer and workload identity, create a UAMI with a federated credential whose subject is `system:serviceaccount:<ns>:<sa>`, annotate the ServiceAccount with the UAMI client ID, and label the pod `azure.workload.identity/use: "true"`. Code then uses DAC/WorkloadIdentityCredential.

**Explanation**
Replaces the deprecated AAD pod identity. Env vars `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_FEDERATED_TOKEN_FILE`, `AZURE_AUTHORITY_HOST` are injected by the webhook.

**Code / CLI / Bicep**
```bash
az aks update -g rg -n aks --enable-oidc-issuer --enable-workload-identity
az aks show -g rg -n aks --query oidcIssuerProfile.issuerUrl -o tsv
az identity federated-credential create --name fic-orders --identity-name id-orders -g rg \
  --issuer <issuerUrl> --subject system:serviceaccount:orders:orders-sa --audiences api://AzureADTokenExchange
```
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: orders-sa
  namespace: orders
  annotations:
    azure.workload.identity/client-id: <uamiClientId>
---
# pod template metadata.labels:
#   azure.workload.identity/use: "true"
```

### 13.4 [Hard] Federation succeeds locally in the pipeline but fails with AADSTS70021 "No matching federated identity record found". Why?
**Answer**
The token's issuer/subject/audience doesn't exactly match any federated credential - typically because the job runs on a different branch/environment/PR than the configured subject. Inspect the actual `sub` claim and add a credential for it.

**Explanation**
Environment-based subjects replace branch-based when a job uses `environment:`. Reusable workflows and forks alter subjects. Max federated credentials per identity is limited (20 at time of writing; verify).

**Code / CLI / Bicep**
```bash
az identity federated-credential list --identity-name id-deploy -g rg -o table
```

### 13.5 [Tricky] Can an app registration's secret-less access use a UAMI as federated credential (MI as FIC)?
**Answer**
Yes - a managed identity can act as a federated credential for an app registration (including cross-tenant scenarios), letting Azure workloads authenticate as that app without secrets. Verify current docs for setup and limits.

**Explanation**
Used for multi-tenant SaaS apps where the app registration (not MI) is what customer tenants consent to. The workload gets an MI token for audience `api://AzureADTokenExchange` and uses it as a client assertion.

**Code / CLI / Bicep**
```csharp
var mi = new ManagedIdentityCredential(ManagedIdentityId.FromUserAssignedClientId(miClientId));
var app = ConfidentialClientApplicationBuilder.Create(appClientId)
    .WithAuthority($"https://login.microsoftonline.com/{customerTenantId}")
    .WithClientAssertion(async ct =>
        (await mi.GetTokenAsync(new TokenRequestContext(new[] { "api://AzureADTokenExchange/.default" }), ct)).Token)
    .Build();
```

**Gotchas / follow-ups**
- Overly broad subject (e.g., any branch) lets any workflow in the repo deploy; use environments with approvals.
- Federation requires outbound access to Entra from the runner.

**Interview traps**
- Thinking federation still needs a stored secret.
- Putting wildcard expectations in `subject`.
- Forgetting `id-token: write`.

**Cheat sheet**
- FIC = issuer + subject + audience on app reg/UAMI.
- GitHub: `permissions: id-token: write`, `azure/login`.
- AKS: SA annotation + pod label + FIC subject `system:serviceaccount:ns:sa`.
- AADSTS70021 -> subject mismatch.

---

## Topic 14: Conditional Access, MFA, PIM, access reviews (overview)

**Study notes**

- **Conditional Access (CA):** if-then policies (signals: user/group, app, device, location, risk; controls: require MFA, compliant device, block, session limits). Needs Entra ID P1+.
- **MFA:** second factor; enforce via CA or Security Defaults.
- **PIM (Privileged Identity Management, P2/Governance):** *eligible* assignments activated just-in-time with time limit, approval, justification, MFA; works for Entra roles, Azure RBAC roles, and groups.
- **Access reviews:** periodic attestation of group/app/role memberships.
- Workload identities (SPs/MIs) are **not** subject to user MFA; CA for workload identities exists for SPs (licensing; MI not supported - verify).

### 14.1 [Easy] What is Conditional Access?
**Answer**
A policy engine that evaluates sign-in signals and applies controls like MFA, compliant device, or block before issuing tokens. It enforces zero-trust access to apps.

**Explanation**
Evaluated after first-factor auth. Use report-only mode to test. Always exclude emergency break-glass accounts.

**Code / CLI / Bicep**
```bash
az rest --method get --url https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies --query "value[].{n:displayName,s:state}"
```

### 14.2 [Follow-up] What does PIM give you for Azure RBAC?
**Answer**
Users are *eligible* for Owner/Contributor and activate for a limited window with justification/approval/MFA instead of holding standing access. It shrinks the privilege exposure window and produces audit trails.

**Explanation**
Eligible vs active assignments; time-bound. Apply to subscription-level roles and break-glass processes. Service principals/MIs generally have permanent active assignments (design them with least privilege).

**Code / CLI / Bicep**
```bash
# list eligible assignments (verify command availability/version)
az rest --method get --url "https://management.azure.com/subscriptions/<sub>/providers/Microsoft.Authorization/roleEligibilityScheduleInstances?api-version=2020-10-01"
```

### 14.3 [Medium] A pipeline/dev's az cli token fails after CA policy rollout.
**Answer**
CA (MFA, compliant device, location) applies to user sign-ins via CLI/VS too, giving errors like AADSTS50076/53003 or claims challenges. Use `az login --tenant` interactively with MFA, or switch automation to federated workload identity, which isn't subject to user CA.

**Explanation**
Service accounts that were regular user accounts with passwords are a smell - convert to service principals/MI.

**Code / CLI / Bicep**
```bash
az login --tenant <tenantId> --scope https://management.core.windows.net//.default
```

### 14.4 [Hard] Continuous Access Evaluation and token lifetime
**Answer**
CAE lets supporting resources (Graph, Exchange, etc.) revoke tokens near-real-time on events like user disable or location change, extending lifetimes to up to 28 hours for CAE-capable clients. Clients must handle claims challenges (401 with `claims`).

**Explanation**
Without CAE, access tokens live ~60-90 minutes and survive revocation until expiry. MI tokens are cached ~24h so removing roles is the effective control, not token revocation. Verify details in current docs.

**Code / CLI / Bicep**
```csharp
var app = ConfidentialClientApplicationBuilder.Create(id).WithClientCapabilities(new[] { "CP1" }).Build(); // declare CAE capable
```

### 14.5 [Tricky] Why do access reviews matter for managed identities and groups?
**Answer**
Groups that grant roles accumulate members; reviews force owners to re-attest. MIs themselves aren't reviewed like users, so role-assignment audits (Topic 17) must cover them.

**Explanation**
Review scopes: group membership, Entra role assignments, Azure resource roles, app assignments. Needs P2/Governance licensing.

**Code / CLI / Bicep**
```bash
az rest --method get --url "https://graph.microsoft.com/v1.0/identityGovernance/accessReviews/definitions"
```

**Gotchas / follow-ups**
- Licensing (P1 vs P2) varies - verify.
- Lock-out risk: always keep two break-glass accounts excluded from CA and monitored.

**Interview traps**
- "MFA applies to managed identities" (no).
- PIM only for Entra roles (it also covers Azure resource roles and groups).
- Standing Owner access in prod.

**Cheat sheet**
- CA = signals + controls; report-only first.
- PIM = eligible -> activate JIT.
- CAE = near-real-time revocation, claims challenges.
- Reviews = periodic re-attestation.

---

## Topic 15: Cross-tenant, B2B, B2C, External ID (overview)

**Study notes**

| Scenario | Solution |
|---|---|
| Partner employees access your apps | **B2B collaboration** - guest users (`#EXT#`) in your tenant, authenticate with their home IdP |
| Customers sign in to your consumer app | **Entra External ID** (external tenant; successor to Azure AD B2C for new projects) / **Azure AD B2C** (legacy; verify retirement/availability for new customers) |
| Your SaaS app used by many orgs | Multi-tenant app registration + admin consent |
| Azure resources in another tenant | Azure Lighthouse (delegated resource mgmt) or guest/SP in that tenant |
| Workloads across tenants | Multi-tenant SP/app consent or MI-as-FIC |

### 15.1 [Easy] B2B vs B2C in one line each?
**Answer**
B2B invites external organizational users as guests into your workforce tenant; B2C (now External ID for new work) handles customer identity with local/social accounts in a separate directory.

**Explanation**
Different tenants, branding, user flows, and licensing (MAU-based for external identities).

**Code / CLI / Bicep**
```bash
az rest --method post --url https://graph.microsoft.com/v1.0/invitations \
  --body '{"invitedUserEmailAddress":"partner@fabrikam.com","inviteRedirectUrl":"https://myapp.contoso.com"}'
```

### 15.2 [Medium] How do you make an API accept tokens from multiple tenants?
**Answer**
Register it as multi-tenant (`AzureADMultipleOrgs`), have customers consent so the SP exists in their tenant, and validate the `tid`/issuer against an allow-list instead of accepting any issuer. Client uses the customer's tenant authority or `/organizations`.

**Explanation**
Issuer is tenant-specific, so Microsoft.Identity.Web uses a multi-tenant issuer validator. Without allow-listing, any Entra tenant could call you.

**Code / CLI / Bicep**
```csharp
builder.Services.Configure<JwtBearerOptions>(JwtBearerDefaults.AuthenticationScheme, o =>
{
    var allowed = new HashSet<string> { "<tenantA>", "<tenantB>" };
    o.TokenValidationParameters.IssuerValidator = (issuer, token, p) =>
    {
        var tid = ((System.IdentityModel.Tokens.Jwt.JwtSecurityToken)token).Claims.First(c => c.Type == "tid").Value;
        return allowed.Contains(tid) ? issuer : throw new SecurityTokenInvalidIssuerException();
    };
});
```
(Prefer the built-in multi-tenant validation patterns of Microsoft.Identity.Web; verify.)

### 15.3 [Medium] Managed identity access to a resource in another tenant?
**Answer**
An MI is bound to its own tenant and can't directly be granted roles in another tenant. Use a multi-tenant app registration with MI as federated credential, or a service principal in the target tenant (ideally federated).

**Explanation**
Subscriptions in a different tenant trust that tenant's directory; Lighthouse is for management delegation (ARM), not data-plane identity.

**Code / CLI / Bicep**
```bash
az role assignment create --assignee <spObjectIdInTargetTenant> --role Reader --scope <resourceInTargetTenant>
```

### 15.4 [Tricky] Guest user can sign in but gets 403 on an Azure resource.
**Answer**
B2B guests need Azure RBAC assignments like any principal, and CA/MFA trust settings (cross-tenant access) may require MFA in your tenant. Assign roles to the guest's object ID in your tenant and check cross-tenant access settings.

**Explanation**
Guest default directory permissions are limited; some portal browsing fails for guests. Cross-tenant access settings control inbound/outbound trust and MFA claims acceptance.

**Code / CLI / Bicep**
```bash
az ad user list --filter "userType eq 'Guest'" --query "[].{upn:userPrincipalName,id:id}" -o table
```

**Gotchas / follow-ups**
- B2C/External ID product direction changes: verify current availability.
- Consent in the customer's tenant can be blocked by user-consent settings.

**Interview traps**
- Confusing guest users with service principals.
- Accepting any tenant's tokens.

**Cheat sheet**
- B2B = guests; External ID/B2C = customers.
- Multi-tenant app = consent + SP per tenant.
- Allow-list `tid`.
- MI is single-tenant; use MI-as-FIC for cross-tenant.

---

## Topic 16: Least privilege design and auditing role assignments

**Study notes**

Principles: narrowest scope, narrowest role, separate identities per workload/environment, prefer data-plane roles to Contributor, no keys/SAS (disable local auth), custom roles when built-ins are too broad, PIM for humans, review regularly.

Audit sources:
- `az role assignment list` (point in time)
- **Activity Log** / `AzureActivity` (who created/deleted assignments)
- **Entra sign-in logs**: interactive, non-interactive, **service principal**, and **managed identity** sign-ins (`SigninLogs`, `AADNonInteractiveUserSignInLogs`, `AADServicePrincipalSignInLogs`, `AADManagedIdentitySignInLogs`) - need diagnostic settings export to Log Analytics.
- **Azure Policy** to deny/flag `disableLocalAuth=false`, etc.

### 16.1 [Easy] What does least privilege look like for a Function that receives from one queue and writes one blob container?
**Answer**
Service Bus Data Receiver scoped to that queue and Storage Blob Data Contributor scoped to that container, on a dedicated identity. No namespace-level or subscription-level roles.

**Explanation**
Plus host-storage roles on the function's own storage account. Separate dev/test/prod identities.

**Code / CLI / Bicep**
```bash
az role assignment create --assignee-object-id <pid> --assignee-principal-type ServicePrincipal --role "Azure Service Bus Data Receiver" --scope <queueId>
az role assignment create --assignee-object-id <pid> --assignee-principal-type ServicePrincipal --role "Storage Blob Data Contributor" --scope <storageId>/blobServices/default/containers/out
```

### 16.2 [Follow-up] How do you list who has what on a resource, including inherited?
**Answer**
Use `az role assignment list --scope <id> --include-inherited` (or the portal IAM blade) and resolve principal types; also list by assignee with `--all`.

**Explanation**
Check for orphaned ("Identity not found") assignments and overly broad scopes (subscription Contributor for apps).

**Code / CLI / Bicep**
```bash
az role assignment list --scope <rid> --include-inherited --query "[].{p:principalName,t:principalType,r:roleDefinitionName,s:scope}" -o table
az role assignment list --all --query "[?roleDefinitionName=='Owner'].{p:principalName,s:scope}" -o table
```

### 16.3 [Medium] Who created that role assignment? (Activity Log KQL)
**Answer**
Query `AzureActivity` for `MICROSOFT.AUTHORIZATION/ROLEASSIGNMENTS/WRITE` events to find the caller, time and scope. The Activity Log must be routed to Log Analytics for long retention (portal keeps 90 days).

**Explanation**
Role assignments are subscription-scoped operations visible in the Activity Log of that subscription. Correlate with the caller's IP/principal.

**Code / CLI / Bicep**
```kusto
AzureActivity
| where OperationNameValue =~ "MICROSOFT.AUTHORIZATION/ROLEASSIGNMENTS/WRITE"
| where ActivityStatusValue == "Success"
| project TimeGenerated, Caller, CallerIpAddress, ResourceId, Properties
| order by TimeGenerated desc
```

### 16.4 [Medium] Prove which managed identity accessed Key Vault / failed sign-in.
**Answer**
Use MI sign-in logs for token issuance, and the service's resource logs (Key Vault `AuditEvent`, Service Bus operational logs, Storage logs) for data-plane calls with caller identity. Sign-in logs show the token request, not what the token was used for.

**Explanation**
Enable diagnostic settings for each. Key Vault AuditEvent includes `identity_claim_oid_g`/appid in `identity_claim`.

**Code / CLI / Bicep**
```kusto
AADManagedIdentitySignInLogs
| where TimeGenerated > ago(1d)
| summarize count() by ServicePrincipalName, ResourceDisplayName, ResultType
| order by count_ desc
```
```kusto
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.KEYVAULT" and Category == "AuditEvent"
| where ResultSignature != "OK"
| project TimeGenerated, OperationName, ResultSignature, CallerIPAddress, identity_claim_oid_g
```
(Column names vary between legacy AzureDiagnostics and resource-specific tables such as `AZKVAuditLogs` - verify.)

### 16.5 [Hard] Enforce zero-key access via policy and spot drift.
**Answer**
Use Azure Policy (built-in "Local authentication methods should be disabled" definitions per service, and storage `allowSharedKeyAccess` auditing) to audit/deny key-based auth, then monitor compliance. Combine with deployment pipelines setting `disableLocalAuth: true`.

**Explanation**
Disabling local auth breaks clients still using connection strings; roll out audit -> fix -> deny. Verify exact built-in policy names in current docs.

**Code / CLI / Bicep**
```bicep
resource sb 'Microsoft.ServiceBus/namespaces@2022-10-01-preview' = {
  name: sbName
  location: location
  sku: { name: 'Standard' }
  properties: { disableLocalAuth: true }
}
resource st 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: stName
  location: location
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
  properties: { allowSharedKeyAccess: false, defaultToOAuthAuthentication: true }
}
```

### 16.6 [Tricky] Custom role needed - how to craft safely?
**Answer**
Start from a built-in role, list the exact `Actions`/`DataActions` needed, scope with `AssignableScopes`, and test. Avoid wildcards; remember data actions go in `DataActions`.

**Explanation**
Custom roles count against limits (verify). Use `az provider operation show` to discover operation names.

**Code / CLI / Bicep**
```bash
az role definition create --role-definition '{
 "Name":"Orders Queue Peek-only",
 "Description":"Peek messages",
 "Actions":[],
 "DataActions":["Microsoft.ServiceBus/namespaces/messages/receive/action"],
 "AssignableScopes":["/subscriptions/<sub>"]}'
```
(The DataAction name above is illustrative - confirm operation names via `az provider operation show --namespace Microsoft.ServiceBus`.)

**Gotchas / follow-ups**
- Contributor on a resource can read keys even if local auth is "unused" - disable it.
- Reader + Key Vault Secrets User on subscription scope = every vault's secrets readable.
- MI sign-in log latency can be minutes.

**Interview traps**
- Granting subscription Contributor to "make deployments work".
- Auditing only users, not SP/MI.
- Assuming sign-in logs show data access.

**Cheat sheet**
- Narrow scope + narrow role + dedicated identity.
- `az role assignment list --all --include-inherited`.
- Activity Log for assignment writes; MI sign-in logs for token issuance; resource logs for data access.
- Disable local/key auth + Policy.

---

## Topic 17: Common errors and fixes

**Study notes**

| Symptom | Likely cause | Fix |
|---|---|---|
| ARM `AuthorizationFailed` ("does not have authorization to perform action ... over scope") | Missing control-plane role at that scope | Assign needed role; check scope/inheritance/deny |
| Blob 403 `AuthorizationPermissionMismatch` / "This request is not authorized to perform this operation using this permission" | Entra auth OK but no Blob Data role | Assign Storage Blob Data *; wait for propagation |
| Blob 403 `AuthorizationFailure` | Storage firewall/VNet/private endpoint blocks | Allow network path / trusted services |
| Service Bus 401 `Unauthorized access. 'Send' claim(s) are required` / `UnauthorizedAccessException` | No SB data role or wrong audience/namespace | Assign Data Sender/Receiver at right scope |
| Key Vault 403 `Forbidden ... ForbiddenByRbac/ForbiddenByPolicy`, `Caller is not authorized` | No KV data role / access-policy vs RBAC mismatch / firewall | Assign Secrets User; check permission model and network |
| `ManagedIdentityCredential authentication unavailable` | No MI on host / wrong client ID / IMDS blocked | Enable/attach identity; set AZURE_CLIENT_ID |
| `CredentialUnavailableException` all credentials failed (local) | Not logged in | `az login`, set tenant |
| `AADSTS700016` | App not found in tenant | Consent/provision SP; right tenant |
| `AADSTS7000215` / `7000222` | Bad / expired secret | New secret or use federation/MI |
| `AADSTS50011` | Redirect URI mismatch | Register exact URI |
| `AADSTS65001` | Consent missing | Admin consent |
| `AADSTS50076` / `50079` / `53003` | MFA required / CA blocked | Complete MFA, adjust CA, use claims challenge |
| `AADSTS70021` | No matching federated credential | Fix subject/issuer |
| `AADSTS500011` | Resource principal not found | Wrong audience/resource or SP missing in tenant |
| `AADSTS70011` | Invalid scope | Use `<resource>/.default` for app-only |
| SQL "Login failed for user '<token-identified principal>'" | No contained user | `CREATE USER ... FROM EXTERNAL PROVIDER` |

### 17.1 [Easy] 401 vs 403 - how do you reason about them?
**Answer**
401 = the service couldn't authenticate you (missing/invalid/expired/wrong-audience token); 403 = authenticated but not authorized (RBAC, firewall, policy). Not all Azure services follow it strictly (Service Bus often returns 401 for missing claims).

**Explanation**
First decode the token (`aud`, `oid`), then check roles at the exact scope, then network.

**Code / CLI / Bicep**
```bash
az role assignment list --assignee <oid> --all -o table
```

### 17.2 [Follow-up] "This request is not authorized to perform this operation using this permission" - steps?
**Answer**
It's Blob/Queue `AuthorizationPermissionMismatch`: your identity lacks the data-plane role for that operation. Assign the right Storage Blob/Queue/Table Data role at account or container scope and retry after propagation.

**Explanation**
Owner/Contributor don't suffice. Distinguish from `AuthorizationFailure` (network). Remember user-delegation SAS and queue vs blob roles (queue triggers need Queue Data roles).

**Code / CLI / Bicep**
```bash
az role assignment create --assignee-object-id <oid> --assignee-principal-type ServicePrincipal \
  --role "Storage Queue Data Contributor" --scope <storageId>
```

### 17.3 [Medium] AuthorizationFailed when deploying Bicep role assignments.
**Answer**
Creating role assignments requires `Microsoft.Authorization/roleAssignments/write` (Owner or User Access Administrator, or a role with that action). Contributor can't do it; give the pipeline identity User Access Administrator (ideally constrained via ABAC conditions) or Role Based Access Control Administrator.

**Explanation**
"Role Based Access Control Administrator" can be limited with conditions on which roles it may assign - good for least privilege.

**Code / CLI / Bicep**
```bash
az role assignment create --assignee <pipelineSpOid> --role "Role Based Access Control Administrator" --scope <rgId>
```

### 17.4 [Hard] Intermittent 401 right after deployment, then fine.
**Answer**
New identity/role assignment replication and cached tokens cause transient failures. Retry with backoff, avoid failing fast at startup, and for UAMI recreation ensure the same principal is used.

**Explanation**
Orchestrate: create identity -> role assignment (principalType set) -> deploy app; add a delay or retry in health probes. MI token cache (~24h) may keep old claims if identities changed.

**Code / CLI / Bicep**
```csharp
var client = new ServiceBusClient(fqns, cred, new ServiceBusClientOptions {
    RetryOptions = new ServiceBusRetryOptions { Mode = ServiceBusRetryMode.Exponential, MaxRetries = 8, MaxDelay = TimeSpan.FromSeconds(30) } });
```

### 17.5 [Tricky] Works with connection string, fails with identity - only on the private endpoint environment.
**Answer**
Identity auth also needs the token endpoint reachable (login.microsoftonline.com or IMDS) and the data endpoint's DNS/firewall correct; connection-string path may mask a firewall/DNS issue differently. Check outbound access to Entra (service tags `AzureActiveDirectory`) and private DNS for the resource.

**Explanation**
Also verify `fullyQualifiedNamespace` uses the public FQDN that resolves privately (don't use IPs). Time skew can cause token validation failures on VMs.

**Code / CLI / Bicep**
```bash
nslookup myns.servicebus.windows.net
curl -s -o /dev/null -w "%{http_code}" https://login.microsoftonline.com/<tenant>/v2.0/.well-known/openid-configuration
```

**Gotchas / follow-ups**
- Read the full AADSTS message; it includes trace/correlation IDs - capture them for support.
- `AADSTS` error lookup: `https://login.microsoftonline.com/error?code=<n>`.

**Interview traps**
- Treating all 403s as RBAC (could be firewall).
- Restarting instead of reading the exception chain.

**Cheat sheet**
- Decode token -> check role at scope -> check network.
- `AuthorizationPermissionMismatch` = missing data role.
- Role assignment write needs RBAC Admin/UAA/Owner.
- Retry with exponential backoff post-deploy.

---

## Topic 18: Zero-secret architecture end-to-end (Bicep)

**Study notes**

Target: Function App (Service Bus trigger) -> Blob output + Key Vault read, all via one UAMI, no keys anywhere.

```
GitHub Actions --(OIDC FIC)--> deploy identity --> Bicep deploy
Function App --(UAMI)--> Service Bus (Receiver)  Storage (Blob Data Contributor, host roles)  Key Vault (Secrets User)
Local auth disabled on SB + Storage; KV uses RBAC
```
Role IDs below are well-known built-in IDs; verify with `az role definition list --name "<role>" --query [0].name`.

### 18.1 [Easy] What are the building blocks?
**Answer**
One UAMI, resources with local auth disabled, role assignments to the UAMI's principalId, and app settings that contain only endpoints and the UAMI clientId. No secrets exist to store.

**Explanation**
Order: UAMI -> resources -> role assignments -> app. Creating the UAMI first lets the role assignments exist before the app starts.

**Code / CLI / Bicep**
```bicep
param location string = resourceGroup().location
param prefix string = 'orders'
param planId string   // existing Elastic Premium or Flex plan resource ID

var roles = {
  sbReceiver: '4f6d3b9b-027b-4f4c-9142-0e5a2a2247e0'
  blobOwner: 'b7e6dc6d-f1e8-4753-8033-0f276bb0955b'
  blobContributor: 'ba92f5b4-2d11-453d-a403-e96b0029c9fe'
  queueContributor: '974c5e8b-45b9-4653-ba55-5f855dd0fb88'
  tableContributor: '0a9a7e1f-b9d0-4cc4-a60d-0319b160aaa3'
  kvSecretsUser: '4633458b-17de-408a-b874-0445c86b69e6'
}

resource uami 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = {
  name: 'id-${prefix}'
  location: location
}

resource sb 'Microsoft.ServiceBus/namespaces@2022-10-01-preview' = {
  name: 'sb-${prefix}-${uniqueString(resourceGroup().id)}'
  location: location
  sku: { name: 'Standard' }
  properties: { disableLocalAuth: true }
}
resource queue 'Microsoft.ServiceBus/namespaces/queues@2022-10-01-preview' = {
  parent: sb
  name: 'orders'
}

resource st 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: 'st${prefix}${uniqueString(resourceGroup().id)}'
  location: location
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
  properties: { allowSharedKeyAccess: false, defaultToOAuthAuthentication: true, minimumTlsVersion: 'TLS1_2' }
}

resource kv 'Microsoft.KeyVault/vaults@2023-07-01' = {
  name: 'kv-${prefix}-${uniqueString(resourceGroup().id)}'
  location: location
  properties: {
    sku: { family: 'A', name: 'standard' }
    tenantId: subscription().tenantId
    enableRbacAuthorization: true
    enableSoftDelete: true
  }
}
```

### 18.2 [Follow-up] Role assignments at minimal scope
**Answer**
Assign Receiver on the queue, storage roles on the account (host needs blob/queue/table), and Secrets User on the vault, all with deterministic GUID names and `principalType: 'ServicePrincipal'`.

**Explanation**
A reusable module keeps this tidy. Scope can be narrowed to a secret for Key Vault where appropriate.

**Code / CLI / Bicep**
```bicep
resource raSb 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  scope: queue
  name: guid(queue.id, uami.id, roles.sbReceiver)
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', roles.sbReceiver)
    principalId: uami.properties.principalId
    principalType: 'ServicePrincipal'
  }
}

resource raStorage 'Microsoft.Authorization/roleAssignments@2022-04-01' = [for r in [roles.blobOwner, roles.queueContributor, roles.tableContributor]: {
  scope: st
  name: guid(st.id, uami.id, r)
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', r)
    principalId: uami.properties.principalId
    principalType: 'ServicePrincipal'
  }
}]

resource raKv 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  scope: kv
  name: guid(kv.id, uami.id, roles.kvSecretsUser)
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', roles.kvSecretsUser)
    principalId: uami.properties.principalId
    principalType: 'ServicePrincipal'
  }
}
```

### 18.3 [Medium] The Function App wiring
**Answer**
Attach the UAMI to the app, set `AZURE_CLIENT_ID`, identity-based connection settings (`__fullyQualifiedNamespace`, `__credential`, `__clientId`) and `keyVaultReferenceIdentity`; leave no connection strings.

**Explanation**
Plan type and storage settings for content share vary (Premium/Flex have different requirements; verify). `dependsOn` role assignments so the app starts with access.

**Code / CLI / Bicep**
```bicep
resource func 'Microsoft.Web/sites@2023-12-01' = {
  name: 'func-${prefix}-${uniqueString(resourceGroup().id)}'
  location: location
  kind: 'functionapp'
  identity: { type: 'UserAssigned', userAssignedIdentities: { '${uami.id}': {} } }
  dependsOn: [ raSb, raKv, raStorage ]
  properties: {
    serverFarmId: planId
    httpsOnly: true
    keyVaultReferenceIdentity: uami.id
    siteConfig: {
      minTlsVersion: '1.2'
      appSettings: [
        { name: 'FUNCTIONS_EXTENSION_VERSION', value: '~4' }
        { name: 'FUNCTIONS_WORKER_RUNTIME', value: 'dotnet-isolated' }
        { name: 'AZURE_CLIENT_ID', value: uami.properties.clientId }
        { name: 'AzureWebJobsStorage__accountName', value: st.name }
        { name: 'AzureWebJobsStorage__credential', value: 'managedidentity' }
        { name: 'AzureWebJobsStorage__clientId', value: uami.properties.clientId }
        { name: 'SbConn__fullyQualifiedNamespace', value: '${sb.name}.servicebus.windows.net' }
        { name: 'SbConn__credential', value: 'managedidentity' }
        { name: 'SbConn__clientId', value: uami.properties.clientId }
        { name: 'OutputBlobUri', value: st.properties.primaryEndpoints.blob }
        { name: 'ApiKey', value: '@Microsoft.KeyVault(VaultName=${kv.name};SecretName=partner-api-key)' }
      ]
    }
  }
}
```

### 18.4 [Hard] How does the code look and how do you make CI/CD secretless too?
**Answer**
The code uses one singleton credential bound to the UAMI client ID; the pipeline uses GitHub OIDC federated credentials on a deploy identity with RBAC Administrator-style rights scoped to the resource group.

**Explanation**
End state: no client secrets, no connection strings, no SAS, no keys. Only IDs (non-secret) in configuration.

**Code / CLI / Bicep**
```csharp
var clientId = Environment.GetEnvironmentVariable("AZURE_CLIENT_ID");
var cred = new ManagedIdentityCredential(ManagedIdentityId.FromUserAssignedClientId(clientId!));

var blob = new BlobServiceClient(new Uri(Environment.GetEnvironmentVariable("OutputBlobUri")!), cred);
await blob.GetBlobContainerClient("out").GetBlobClient($"{Guid.NewGuid()}.json")
    .UploadAsync(BinaryData.FromString("{}"));
```
```bash
az deployment group create -g rg-orders -f main.bicep -p planId=<id>
```

### 18.5 [Tricky] Validation checklist and failure modes
**Answer**
Verify no secrets/keys remain, roles exist at expected scopes, and the app can acquire tokens; common failures are missing host-storage roles, missing `__clientId`, wrong principal IDs, propagation delay, and firewall/private endpoint rules.

**Explanation**
Test by checking app logs for `CredentialUnavailable`/403, and Service Bus trigger scale behaviour.

**Code / CLI / Bicep**
```bash
az role assignment list --assignee <uamiPrincipalId> --all -o table
az functionapp config appsettings list -g rg -n <func> --query "[?contains(name,'Key') || contains(name,'ConnectionString')]"
az servicebus namespace show -g rg -n <sb> --query disableLocalAuth
az storage account show -g rg -n <st> --query allowSharedKeyAccess
```

**Gotchas / follow-ups**
- Role assignment `name` must be stable; changing principal with same GUID name fails.
- Disable shared key on the host storage only after verifying all consumers (Durable, content share) support identity - verify.
- Add diagnostic settings so you can audit access.

**Interview traps**
- Leaving `AzureWebJobsStorage` as a connection string "temporarily".
- Role on the app's SAMI instead of the UAMI actually attached.
- Forgetting `dependsOn`/propagation delay.

**Cheat sheet**
- UAMI first, roles second, app third.
- `disableLocalAuth`, `allowSharedKeyAccess:false`, KV RBAC.
- App settings = endpoints + clientId only.
- Federate CI/CD; audit with logs.
