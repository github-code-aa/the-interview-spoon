# Z03 - Azure Key Vault: Study + Interview Guide

**How to use:** Read each topic's *Study notes* first, then work the difficulty ladder (Easy -> Tricky) by answering aloud before reading the **Answer**.
Limits, SDK versions and SKUs change; anything marked "verify in current docs" must be re-checked before quoting numbers in an interview.

## Table of Contents
1. What Key Vault is: secrets, keys, certificates; vault vs Managed HSM; Standard vs Premium
2. Access model: access policies vs Azure RBAC; data plane vs control plane
3. Authenticating from .NET: SecretClient, DefaultAzureCredential, SAMI vs UAMI
4. Configuration provider: AddAzureKeyVault
5. Key Vault references in App Service / Functions
6. Versions, rotation, expiry and Event Grid
7. Soft delete, purge protection, recovery
8. Certificates
9. Keys and crypto operations, CMK, BYOK
10. Throttling and caching secrets in the app
11. Networking: firewall, private endpoint, trusted services, DNS
12. Diagnostics, auditing, alerts
13. Key Vault in CI/CD and Bicep
14. Secrets in local development
15. Common errors and troubleshooting
16. Disaster recovery, multi-env design, what NOT to store

---

## Topic 1: What Key Vault is; vault vs Managed HSM; Standard vs Premium

**Study notes**

Key Vault is a managed service for storing and controlling access to three object types:

| Object | What it is | Typical use |
|---|---|---|
| Secret | Opaque string/bytes up to 25 KB (verify in current docs) | Connection strings, API keys, passwords |
| Key | RSA / EC (and oct-HSM in HSM tiers) cryptographic key; private material never leaves the vault | Sign, verify, wrap/unwrap, encrypt, CMK |
| Certificate | X.509 cert + policy; backed by a secret (PFX/PEM with private key) and a key | TLS certs, client auth certs, auto-renewal |

```
 Vault (https://<name>.vault.azure.net)
   |-- Secrets      /secrets/<name>/<version>
   |-- Keys         /keys/<name>/<version>
   `-- Certificates /certificates/<name>/<version>
        (a cert also creates a same-named secret + key)
```

- **Vault (Standard)**: software-protected keys. **Vault (Premium)**: adds HSM-protected keys (FIPS 140-2/140-3 level per docs - verify). Secrets and certs behave the same in both.
- **Managed HSM**: separate resource type (`Microsoft.KeyVault/managedHSMs`), single-tenant, dedicated FIPS 140 Level 3 HSMs, keys only (no secrets/certs), local RBAC only, hourly pricing, security domain you must back up.

**Ladder**

### 1.1 [Easy] What is Azure Key Vault and what three object types does it hold?
**Answer**
Key Vault is a managed service for securely storing and tightly controlling access to secrets, cryptographic keys and certificates. Apps fetch values at runtime using Entra identity instead of keeping them in code or config. Access is audited and objects are versioned.

**Explanation**
Secrets are opaque blobs. Keys are crypto material where the private part stays in the vault and you ask the vault to perform operations. Certificates are a managed lifecycle wrapper (policy, issuer, renewal) that also surface as a key and a secret (the secret carries the exportable PFX/PEM).

**Code / CLI / Bicep**
```bash
az keyvault create -n kv-demo-001 -g rg-demo -l westeurope --enable-rbac-authorization true
az keyvault secret set --vault-name kv-demo-001 -n SqlPassword --value "S3cret!"
az keyvault secret show --vault-name kv-demo-001 -n SqlPassword --query value -o tsv
```

### 1.2 [Follow-up] Standard vs Premium vault: what actually differs?
**Answer**
Premium allows HSM-backed keys; Standard only software-protected keys. Secrets and certificates are the same on both. Choose Premium when compliance requires keys in an HSM boundary.

**Explanation**
The tier is the vault `sku` (`standard`/`premium`). With Premium you create keys of type `RSA-HSM` / `EC-HSM`. You cannot change tier by editing in place in some flows - verify in current docs. Pricing for operations differs.

**Code / CLI / Bicep**
```bicep
resource kv 'Microsoft.KeyVault/vaults@2023-07-01' = {
  name: 'kv-demo-001'
  location: resourceGroup().location
  properties: {
    tenantId: subscription().tenantId
    sku: { family: 'A', name: 'premium' }
    enableRbacAuthorization: true
  }
}
```
```bash
az keyvault key create --vault-name kv-demo-001 -n sign-key --kty RSA-HSM --size 2048
```

### 1.3 [Medium] When would you choose Managed HSM over a Premium vault?
**Answer**
When you need single-tenant dedicated HSM hardware at FIPS 140 Level 3, customer-controlled security domain, or high key-operation throughput for keys only. It does not store secrets or certificates, and uses local RBAC exclusively.

**Explanation**
Managed HSM is a different resource and data-plane endpoint (`*.managedhsm.azure.net`). On creation you must activate it by downloading the security domain (needs quorum of RSA keys) - lose it and you cannot recover the HSM after a disaster. Use for regulated BYOK/CMK at scale. A Premium vault shares multi-tenant HSM pools.

**Code / CLI / Bicep**
```bash
az keyvault create --hsm-name hsm-demo-001 -g rg-demo -l westeurope \
  --administrators <objectId> --retention-days 7
```

### 1.4 [Hard] Why does a certificate in Key Vault also appear as a secret and a key, and what does it imply for security?
**Answer**
Key Vault stores the cert's private key as a key object and exposes the PFX/PEM as a same-named secret so that App Service, App Gateway and apps can retrieve it. Anyone with secret `get` on that name can export the private key, even if you only intended "certificate" access.

**Explanation**
Policy `exportable: true/false` controls whether the private key can leave the vault. With `exportable=false` the secret is not retrievable with the private key and you must use the key for crypto operations in the vault. Under RBAC, `Key Vault Certificates Officer` manages certs but reading the private key needs secret get permission (`Key Vault Secrets User`) - grant carefully.

**Code / CLI / Bicep**
```bash
az keyvault certificate show --vault-name kv-demo-001 -n web-cert --query "{sid:sid,kid:kid,cid:id}"
az keyvault secret download --vault-name kv-demo-001 -n web-cert -f web-cert.pfx --encoding base64
```

**Gotchas / follow-ups**
- Secret content type tells format: `application/x-pkcs12` or `application/x-pem-file`.
- Non-exportable cert: App Service cannot import it.

**Interview traps**
- "Key Vault stores certificates" - also state they are backed by a key + secret.
- Managed HSM does not hold secrets or certificates.
- Premium vs Managed HSM are not the same thing.
- Secret size limit is small (25 KB) - not a blob store.

**Cheat sheet**
- Secrets/keys/certs; names URL-addressable with versions.
- Standard = software keys; Premium = HSM keys; Managed HSM = dedicated, keys only.
- Cert -> key + secret + policy.
- Always create new vaults with RBAC enabled.

---

## Topic 2: Access model: access policies vs Azure RBAC; data plane vs control plane

**Study notes**

| Plane | What | Authorized by |
|---|---|---|
| Control plane (ARM, `management.azure.com`) | Create/delete vault, network rules, access policies, diagnostic settings | Azure RBAC on the vault resource (Contributor, Owner, Key Vault Contributor) |
| Data plane (`*.vault.azure.net`) | Get/set secrets, keys ops, cert ops | Access policies OR Azure RBAC data-role (depends on vault permission model) |

Key RBAC built-in roles (names are exact): `Key Vault Administrator`, `Key Vault Secrets User` (read secret values), `Key Vault Secrets Officer` (manage secrets), `Key Vault Crypto User` (perform crypto ops), `Key Vault Crypto Officer`, `Key Vault Certificates Officer`, `Key Vault Reader` (metadata only, not values), `Key Vault Crypto Service Encryption User` (wrap/unwrap for CMK).

**Ladder**

### 2.1 [Easy] What are the two permission models for a vault?
**Answer**
Vault access policies (legacy, per-principal permission lists on the vault) and Azure RBAC (role assignments at vault, resource group, subscription or even single-object scope). Microsoft recommends RBAC. A vault uses one model at a time.

**Explanation**
Property `enableRbacAuthorization` selects it. When true, access policies are ignored. RBAC gives consistent governance, PIM, conditions, Activity Log, and per-secret scope.

**Code / CLI / Bicep**
```bash
az role assignment create --assignee <principalId> --role "Key Vault Secrets User" \
  --scope $(az keyvault show -n kv-demo-001 --query id -o tsv)
```

### 2.2 [Follow-up] Control plane vs data plane: a user is Owner of the vault but gets 403 reading a secret. Why?
**Answer**
Owner/Contributor are control-plane roles; with RBAC model they carry no data actions (`NotDataActions` / no `dataActions`). Under RBAC the user needs a data role like Key Vault Secrets User. Under the access-policy model, Contributor could even grant themselves access by editing policies.

**Explanation**
This is a known security difference: with access policies, `Microsoft.KeyVault/vaults/write` allows self-granting data access. With RBAC, writing role assignments requires `Microsoft.Authorization/roleAssignments/write` (Owner / User Access Administrator), a tighter privilege.

**Code / CLI / Bicep**
```bicep
param principalId string
resource kv 'Microsoft.KeyVault/vaults@2023-07-01' existing = { name: 'kv-demo-001' }
// Key Vault Secrets User
var secretsUser = '4633458b-17de-408a-b874-0445c86b69e6'
resource ra 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(kv.id, principalId, secretsUser)
  scope: kv
  properties: {
    principalId: principalId
    principalType: 'ServicePrincipal'
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', secretsUser)
  }
}
```

### 2.3 [Medium] How do you migrate from access policies to RBAC safely?
**Answer**
Inventory existing policies, map each to equivalent RBAC roles at the narrowest scope, create the assignments first, then flip `enableRbacAuthorization` to true, and validate. Do it in a maintenance window since RBAC propagation can take minutes.

**Explanation**
Mapping examples: secrets get/list -> Secrets User; set/delete -> Secrets Officer; keys wrap/unwrap/sign -> Crypto User; cert management -> Certificates Officer. Once flipped, access policies are retained but not enforced (you can flip back). Key Vault-references and Functions use identities that also need roles. Check dependents (Disk Encryption, Storage CMK identities, App Gateway) before switching. Microsoft publishes a migration guide - verify in current docs.

**Code / CLI / Bicep**
```bash
az keyvault show -n kv-demo-001 --query properties.accessPolicies
# after assignments exist:
az keyvault update -n kv-demo-001 --enable-rbac-authorization true
```

### 2.4 [Hard] Why can RBAC be better and what limits does it have?
**Answer**
RBAC supports object-level scope, PIM, Azure Policy, conditions, and centralized audit; access policies are capped (about 1024 per vault - verify) and vault-wide only. Downsides: role assignment propagation delay, ARM write limits on assignments per subscription, and the need for role-assign rights.

**Explanation**
Object-level scope: scope string `/.../vaults/kv/secrets/mysecret`. Good for one secret per app in a shared vault, though one vault per app/env is still the simpler boundary. Data-plane RBAC caches in Key Vault for a few minutes, so a freshly granted role can still return 403.

**Code / CLI / Bicep**
```bash
az role assignment create --assignee <oid> --role "Key Vault Secrets User" \
  --scope "/subscriptions/<sub>/resourceGroups/rg-demo/providers/Microsoft.KeyVault/vaults/kv-demo-001/secrets/SqlPassword"
```

**Gotchas / follow-ups**
- `Key Vault Reader` cannot read values.
- A role granted on the resource group applies to every vault under it.
- Terraform/Bicep must wait for the role then retry; add `dependsOn`.

**Interview traps**
- Owner != can read secrets under RBAC.
- Access policies and RBAC are mutually exclusive per vault.
- Role assignment propagation is not instant.
- Contributor can escalate under access policy model.

**Cheat sheet**
- Control plane = ARM; data plane = vault endpoint.
- Use RBAC; Secrets User for apps, Officer for admins.
- Crypto User for sign/wrap; Crypto Service Encryption User for CMK.
- Least privilege + narrow scope.

---

## Topic 3: Authenticating from .NET: SecretClient, DefaultAzureCredential, SAMI vs UAMI

**Study notes**

Packages: `Azure.Security.KeyVault.Secrets`, `Azure.Security.KeyVault.Keys`, `Azure.Security.KeyVault.Certificates`, `Azure.Identity`. Token audience is `https://vault.azure.net`.

```
App (Function/App Service/AKS)
  -> DefaultAzureCredential -> Managed Identity endpoint (IMDS / App Service MSI) -> Entra token
  -> SecretClient -> https://<vault>.vault.azure.net/secrets/name -> RBAC check -> value
```

| | SAMI | UAMI |
|---|---|---|
| Lifecycle | Tied to the resource | Standalone resource |
| Sharing | One resource only | Many resources |
| Pre-provisioned RBAC | No (identity exists after deploy) | Yes - good for IaC ordering |
| Selecting in code | default | `ManagedIdentityClientId` / `AZURE_CLIENT_ID` |

**Ladder**

### 3.1 [Easy] How do you read a secret in .NET with managed identity?
**Answer**
Create a `SecretClient` with the vault URI and `DefaultAzureCredential`, then call `GetSecretAsync`. The identity must have a data role such as Key Vault Secrets User.

**Explanation**
`DefaultAzureCredential` tries a chain (environment, workload identity, managed identity, Visual Studio, Azure CLI, etc.; order and members vary by version - verify in current docs). Locally it uses your developer login, in Azure the managed identity.

**Code / CLI / Bicep**
```csharp
using Azure.Identity;
using Azure.Security.KeyVault.Secrets;

var client = new SecretClient(new Uri("https://kv-demo-001.vault.azure.net/"),
                              new DefaultAzureCredential());
KeyVaultSecret secret = await client.GetSecretAsync("SqlPassword");
string value = secret.Value;
```

### 3.2 [Follow-up] SAMI vs UAMI: which and why?
**Answer**
SAMI is simplest and dies with the resource; UAMI is reusable and can be given RBAC before the app exists, avoiding chicken-and-egg ordering and secret-reference failures at first deploy. Prefer UAMI for multiple apps/slots sharing a vault or for IaC pipelines.

**Explanation**
Slots: a SAMI is per-slot (each slot has its own principal) so you need to grant each; a UAMI assigned to all slots avoids that. With multiple identities on a resource, DefaultAzureCredential doesn't know which one: set the client id.

**Code / CLI / Bicep**
```csharp
var cred = new DefaultAzureCredential(new DefaultAzureCredentialOptions
{
    ManagedIdentityClientId = config["UamiClientId"]
});
```
```bicep
resource uami 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = {
  name: 'id-app'
  location: resourceGroup().location
}
resource func 'Microsoft.Web/sites@2023-01-01' = {
  name: 'func-demo'
  location: resourceGroup().location
  kind: 'functionapp'
  identity: {
    type: 'UserAssigned'
    userAssignedIdentities: { '${uami.id}': {} }
  }
  properties: {}
}
```

### 3.3 [Medium] Why is DefaultAzureCredential discouraged in production by some teams, and what is the alternative?
**Answer**
The chain probes multiple sources, adding latency and potential surprise (e.g., an environment variable overriding managed identity), and failures are noisy. In production use `ManagedIdentityCredential` explicitly, or `ChainedTokenCredential` with only the sources you want.

**Explanation**
Newer Azure.Identity versions also allow `DefaultAzureCredential` to be narrowed via `AZURE_TOKEN_CREDENTIALS` (verify in current docs for exact values/version). A common pattern: environment-based selection - `ManagedIdentityCredential` in Azure, `AzureCliCredential` locally.

**Code / CLI / Bicep**
```csharp
TokenCredential cred = env.IsDevelopment()
    ? new AzureCliCredential()
    : new ManagedIdentityCredential(ManagedIdentityId.FromUserAssignedClientId(clientId));
var client = new SecretClient(vaultUri, cred);
```

### 3.4 [Tricky] Managed identity works in one slot but 403s after swap; another app works but only after restart. Explain.
**Answer**
SAMI principals are per slot, so the staging slot's identity may lack the role the production slot had. Also, MI tokens are cached by the platform and Azure RBAC changes take minutes, so restarts/waiting are often what "fixed" it.

**Explanation**
After a swap, app settings that are not slot-sticky move, but the identity does not. Token caching: the MSI endpoint caches tokens (roughly 24h life; role changes appear through the vault's own authorization cache, not the token - verify specifics in current docs). Group-based RBAC assignments, however, can be baked into token claims and cause staleness for managed identities.

**Code / CLI / Bicep**
```bash
az webapp identity show -g rg -n app --slot staging --query principalId -o tsv
az role assignment list --assignee <principalId> --scope <vaultId> -o table
```

**Gotchas / follow-ups**
- Assign roles to a UAMI to avoid per-slot grants.
- Don't use security-group membership for MI access if you need fast changes (token claim caching).
- Create the `SecretClient` once (singleton) - it reuses tokens.

**Interview traps**
- "403 means wrong code" - it's usually RBAC/firewall.
- Forgetting `ManagedIdentityClientId` with multiple UAMIs.
- Assuming secrets cached by the client library; they aren't.
- Creating a new `DefaultAzureCredential` per call.

**Cheat sheet**
- `new SecretClient(uri, cred)` singleton.
- UAMI for shared/slots/IaC; SAMI for simple 1:1.
- Prefer explicit credential in prod.
- Role: Key Vault Secrets User.

---

## Topic 4: Configuration provider: AddAzureKeyVault

**Study notes**

Package `Azure.Extensions.AspNetCore.Configuration.Secrets`. It loads all enabled secrets of a vault into `IConfiguration`.

| Concept | Behavior |
|---|---|
| Name mapping | Secret names cannot contain `:`; use `--` (e.g. `ConnectionStrings--Sql` -> `ConnectionStrings:Sql`) |
| Default manager | `KeyVaultSecretManager` loads every enabled secret; custom subclass can filter/map |
| Reload | `ReloadInterval` (TimeSpan?) - null by default = load once at startup |
| Load-time failure | Exception at startup (host fails) unless you handle it |

```
Startup -> list secrets (GetPropertiesOfSecrets) -> Get each enabled secret -> IConfiguration
                                                      (N+1 calls; startup cost grows with vault size)
```

**Ladder**

### 4.1 [Easy] How do you add Key Vault to .NET configuration?
**Answer**
Call `builder.Configuration.AddAzureKeyVault(vaultUri, credential)`. Secrets then bind like any other config key, with `--` standing in for `:`.

**Explanation**
Later providers override earlier ones, so add it after appsettings/env vars if you want vault to win.

**Code / CLI / Bicep**
```csharp
builder.Configuration.AddAzureKeyVault(
    new Uri("https://kv-demo-001.vault.azure.net/"),
    new DefaultAzureCredential());

var cs = builder.Configuration["ConnectionStrings:Sql"]; // secret: ConnectionStrings--Sql
```

### 4.2 [Follow-up] How do you get secrets to refresh without restarting?
**Answer**
Set `AzureKeyVaultConfigurationOptions.ReloadInterval`. The provider polls and reloads; consumers must use `IOptionsMonitor`/`IOptionsSnapshot` (not `IOptions`) to see changes.

**Explanation**
Reload re-reads the whole vault each interval, which costs many transactions and can hit throttling in large vaults. Choose a long interval (minutes+) and a small, filtered vault.

**Code / CLI / Bicep**
```csharp
builder.Configuration.AddAzureKeyVault(vaultUri, credential,
    new AzureKeyVaultConfigurationOptions
    {
        ReloadInterval = TimeSpan.FromMinutes(10)
    });
builder.Services.Configure<SqlOptions>(builder.Configuration.GetSection("Sql"));
// inject IOptionsMonitor<SqlOptions>
```

### 4.3 [Medium] Startup fails with 403 or the vault is unreachable. What happens, and how do you design for it?
**Answer**
By default the load throws and the app fails to start (fail fast). For optional vaults you can catch and continue, but then required settings are missing, so validate with `ValidateOnStart`.

**Explanation**
During reloads, failures: the provider by default ignores/handles per version behavior - verify in current docs. Use a custom `KeyVaultSecretManager` to load only the prefix you need (reduces calls, avoids loading unrelated secrets, and avoids needing list permission over everything).

**Code / CLI / Bicep**
```csharp
public sealed class PrefixSecretManager(string prefix) : KeyVaultSecretManager
{
    public override bool Load(SecretProperties secret) => secret.Name.StartsWith(prefix + "-");
    public override string GetKey(KeyVaultSecret secret) =>
        secret.Name[(prefix.Length + 1)..].Replace("--", ConfigurationPath.KeyDelimiter);
}
// options: new AzureKeyVaultConfigurationOptions { Manager = new PrefixSecretManager("orders") }
builder.Services.AddOptions<SqlOptions>().BindConfiguration("Sql").ValidateOnStart();
```

### 4.4 [Hard] The provider needs which permissions, and why is startup slow with a big vault?
**Answer**
It needs list and get on secrets - Key Vault Secrets User covers both. It lists all secret properties then gets each enabled one, so time and transactions scale with secret count and versions.

**Explanation**
Disabled and expired secrets are skipped by the default manager (it checks `Enabled`). Cold starts of Functions suffer: prefer Key Vault references (platform-resolved) in Functions, or a filtered manager.

**Code / CLI / Bicep**
```bash
az keyvault secret list --vault-name kv-demo-001 --query "length(@)"
```

**Gotchas / follow-ups**
- Secret names allow only letters, digits, `-`; no `_` or `:`.
- Section binding of arrays: `Items--0`, `Items--1`.
- Two secrets mapping to the same key - last wins.

**Interview traps**
- Using `:` in a secret name (invalid).
- `IOptions<T>` never reloads.
- Config provider + per-request `SecretClient` double-fetching.
- Assuming provider order doesn't matter.

**Cheat sheet**
- `AddAzureKeyVault(uri, cred)`; `--` = `:`.
- `ReloadInterval` null = no reload.
- Custom `KeyVaultSecretManager` to filter.
- Needs list + get.

---

## Topic 5: Key Vault references in App Service / Functions

**Study notes**

App setting value syntax (platform resolves it; code sees the plain value):
```
@Microsoft.KeyVault(SecretUri=https://kv.vault.azure.net/secrets/Name/<version>)
@Microsoft.KeyVault(SecretUri=https://kv.vault.azure.net/secrets/Name/)     <- latest
@Microsoft.KeyVault(VaultName=kv;SecretName=Name;SecretVersion=<version>)
```
| Aspect | Behavior |
|---|---|
| Identity | SAMI by default; for UAMI set `keyVaultReferenceIdentity` on the site |
| Version omitted | Resolves latest; re-fetch roughly daily (verify in current docs) |
| Version pinned | Never auto-updates until you change the setting |
| Failure | Reference shown as unresolved in portal "Source: Key Vault reference" with an error status; app receives the literal reference text |

**Ladder**

### 5.1 [Easy] What is a Key Vault reference?
**Answer**
It is an app setting whose value points to a vault secret using `@Microsoft.KeyVault(...)`. App Service or Functions resolves it with the app's managed identity, so code reads an ordinary setting and no SDK is needed.

**Explanation**
Works for app settings and connection strings. In Functions, it also works for bindings' connection settings (e.g., `ServiceBusConnection`), but some trigger settings have restrictions - verify in current docs.

**Code / CLI / Bicep**
```bash
az webapp config appsettings set -g rg -n app --settings \
 "SqlPassword=@Microsoft.KeyVault(VaultName=kv-demo-001;SecretName=SqlPassword)"
```

### 5.2 [Follow-up] Which identity is used, and how do you use a UAMI?
**Answer**
The system-assigned identity by default. For a user-assigned identity, set the site property `keyVaultReferenceIdentity` to the UAMI resource ID, and grant it Key Vault Secrets User.

**Explanation**
Without this property, a UAMI-only app can't resolve references. The identity needs network access to the vault too (VNet integration + private endpoint/firewall rules).

**Code / CLI / Bicep**
```bicep
resource site 'Microsoft.Web/sites@2023-01-01' = {
  name: 'func-demo'
  location: resourceGroup().location
  kind: 'functionapp'
  identity: { type: 'UserAssigned', userAssignedIdentities: { '${uami.id}': {} } }
  properties: {
    keyVaultReferenceIdentity: uami.id
    siteConfig: {
      appSettings: [
        { name: 'SqlPassword', value: '@Microsoft.KeyVault(VaultName=kv-demo-001;SecretName=SqlPassword)' }
      ]
    }
  }
}
```

### 5.3 [Medium] Pinned version vs latest: tradeoffs and how fast does rotation propagate?
**Answer**
Unversioned references auto-pick up new secret versions but with a delay (platform refresh, about a day by default - verify), so rotation isn't immediate. Pinned versions are deterministic but require an app-setting update per rotation, which restarts the app.

**Explanation**
To force a refresh of an unversioned reference you can call the ARM/CLI "refresh" operation on the setting, or restart. Many teams use unversioned references plus a rotation process that overlaps old and new credentials (dual-secret) so a day of lag is harmless.

**Code / CLI / Bicep**
```bash
# After rotation, force immediate re-resolve by touching settings or restarting
az webapp restart -g rg -n app
```

### 5.4 [Hard] The app receives the literal string "@Microsoft.KeyVault(...)". Diagnose.
**Answer**
The reference failed to resolve. Check the portal's app settings status for the error: typical causes are missing role/403, vault firewall blocking the app, wrong identity (UAMI without `keyVaultReferenceIdentity`), disabled/expired/deleted secret, or bad syntax.

**Explanation**
Use "Diagnose and solve problems" -> "Key Vault Application Settings Diagnostics". Network: with the vault locked down, the app needs regional VNet integration and `WEBSITE_VNET_ROUTE_ALL`/route settings so outbound reaches the private endpoint (verify in current docs); App Service outbound IPs can alternatively be allow-listed, but they change. Cross-tenant vault references are not supported.

**Code / CLI / Bicep**
```bash
az webapp config appsettings list -g rg -n app --query "[?name=='SqlPassword']"
az resource show --ids <siteId> --api-version 2022-03-01 \
  --query properties.keyVaultReferenceIdentity
```

**Gotchas / follow-ups**
- Consumption plan: referencing through a vault with firewall needs plan support (verify).
- Slot-specific settings must be marked sticky if environments differ.
- Config changes made in the vault update to the app lazily, not instantly.

**Interview traps**
- Believing code needs the SDK.
- Not realizing UAMI requires the extra property.
- Expecting instant rotation for unversioned references.
- Putting a reference in `local.settings.json` (not resolved locally).

**Cheat sheet**
- `@Microsoft.KeyVault(SecretUri=...)` or `(VaultName=;SecretName=)`.
- UAMI -> `keyVaultReferenceIdentity`.
- Unversioned = latest with delay; pinned = static.
- Failure -> literal string + portal diagnostics.

---

## Topic 6: Versions, rotation, expiry and Event Grid

**Study notes**

Every write creates a new immutable version (GUID). `GetSecret(name)` returns the latest *enabled* version. Attributes: `enabled`, `nbf` (activation), `exp` (expiry). Expiry is *informational* for most consumers - Key Vault does not delete or block reads automatically for secrets; the data plane still returns expired secrets (clients must check). Verify nuance in current docs.

```
Near-expiry -> Event Grid (Microsoft.KeyVault.SecretNearExpiry / SecretExpired)
            -> Azure Function -> create new credential at target -> set new secret version
```

Event types: `Microsoft.KeyVault.SecretNewVersionCreated`, `SecretNearExpiry`, `SecretExpired` (same for keys/certs).

**Ladder**

### 6.1 [Easy] What happens when you set a secret that already exists?
**Answer**
A new version is created and becomes the latest; old versions remain until deleted or disabled. Readers without a version get the newest one.

**Explanation**
Max versions are not capped by default but each is retained. Disable old versions after consumers move.

**Code / CLI / Bicep**
```bash
az keyvault secret set --vault-name kv-demo-001 -n ApiKey --value v2 --expires 2027-01-01T00:00:00Z
az keyvault secret list-versions --vault-name kv-demo-001 -n ApiKey -o table
az keyvault secret set-attributes --vault-name kv-demo-001 -n ApiKey --version <oldVer> --enabled false
```

### 6.2 [Follow-up] How do you detect secrets nearing expiry and automate rotation?
**Answer**
Subscribe an Event Grid system topic on the vault to `SecretNearExpiry` and trigger an Azure Function that rotates the credential at the target service and writes a new secret version. Near-expiry fires 30 days before expiry by default (verify in current docs).

**Explanation**
The event contains vault name and object name. The Function's identity needs rights on both the target (e.g., SQL admin, storage key regenerate) and Secrets Officer on the vault. Make rotation idempotent: the event may be delivered more than once.

**Code / CLI / Bicep**
```csharp
[Function("RotateOnNearExpiry")]
public async Task Run([EventGridTrigger] EventGridEvent evt)
{
    var data = evt.Data!.ToObjectFromJson<Dictionary<string, string>>()!;
    var name = data["ObjectName"];
    var newValue = await _target.RegenerateKeyAsync(name);
    await _secretClient.SetSecretAsync(new KeyVaultSecret(name, newValue)
    {
        Properties = { ExpiresOn = DateTimeOffset.UtcNow.AddDays(90) }
    });
}
```
```bicep
resource topic 'Microsoft.EventGrid/systemTopics@2022-06-15' = {
  name: 'kv-events'
  location: resourceGroup().location
  properties: { source: kv.id, topicType: 'Microsoft.KeyVault.vaults' }
}
```

### 6.3 [Medium] Explain the dual-secret (two-key) rotation pattern.
**Answer**
Keep two valid credentials (primary/secondary). Rotate the inactive one, publish it to Key Vault, let consumers pick it up, then later rotate the other. At no moment is there a single valid credential, so cached clients never break.

**Explanation**
Works for services with two keys (Storage, Service Bus SAS, Cosmos). For single-credential systems such as SQL password, use alternating users (user A / user B) behind one secret. Overlap window must exceed the maximum cache/reload delay (e.g., Key Vault reference lag of up to a day).

**Code / CLI / Bicep**
```bash
az storage account keys renew -g rg -n stdemo --key key2
az keyvault secret set --vault-name kv-demo-001 -n StorageKey \
  --value "$(az storage account keys list -g rg -n stdemo --query '[1].value' -o tsv)"
```

### 6.4 [Hard] Does an expired secret stop working? What about `nbf`?
**Answer**
Key Vault still returns expired or not-yet-active secrets in many cases via get; `exp`/`nbf` are metadata and your code or tooling must honor them. By contrast, keys and certificate operations do enforce validity for crypto operations.

**Explanation**
Services that consume Key Vault (e.g., App Service referencing a secret) may treat expiry as a failure - behavior varies; verify in current docs. Use Azure Policy ("secrets should have expiration date") to force an expiry on every secret, and alerts on near-expiry.

**Code / CLI / Bicep**
```csharp
KeyVaultSecret s = await client.GetSecretAsync("ApiKey");
if (s.Properties.ExpiresOn is { } exp && exp < DateTimeOffset.UtcNow)
    throw new InvalidOperationException("Secret expired");
```

**Gotchas / follow-ups**
- Disabled latest version -> `GetSecret` (no version) returns 403/Forbidden-style "disabled" error, not the previous version.
- Event Grid delivery is at-least-once.
- Near-expiry threshold is determined by Key Vault, not configurable per secret for secrets (cert policy has its own lifetime action).

**Interview traps**
- "Latest" = latest enabled? No - if latest is disabled, unversioned get fails (verify).
- Rotating without overlap.
- Not updating `exp` on the new version.

**Cheat sheet**
- New value = new version.
- Event Grid: NearExpiry -> Function -> rotate.
- Dual-secret for zero downtime.
- Policy to require expiry.

---

## Topic 7: Soft delete, purge protection, recovery

**Study notes**

- **Soft delete**: always on for new vaults (cannot be disabled). Deleted vaults/objects are retained 7-90 days (default 90) and can be recovered; name remains reserved.
- **Purge protection**: optional but irreversible once enabled; prevents purge (permanent delete) until retention ends, even by owners. Required for CMK scenarios (Storage, SQL TDE, disk encryption).

```
delete -> Deleted state (recoverable) -> retention elapses -> permanently gone
                |-> recover (restores)
                |-> purge (only if purge protection OFF, needs purge permission)
```

**Ladder**

### 7.1 [Easy] What are soft delete and purge protection?
**Answer**
Soft delete keeps deleted vaults and objects recoverable for a retention period. Purge protection additionally forbids permanent deletion during that period. Together they defend against accidental or malicious deletion.

**Explanation**
Soft delete is mandatory now; purge protection is opt-in and cannot be turned off once on.

**Code / CLI / Bicep**
```bash
az keyvault create -n kv-demo-001 -g rg-demo --enable-purge-protection true --retention-days 90
az keyvault secret delete --vault-name kv-demo-001 -n ApiKey
az keyvault secret recover --vault-name kv-demo-001 -n ApiKey
```

### 7.2 [Follow-up] Redeploying a vault by Bicep fails because the name exists. Why?
**Answer**
The previous vault was soft-deleted and its name is reserved globally until purged or retention ends. Recover it, purge it (if no purge protection), or choose another name; Bicep can also `createMode: 'recover'`.

**Explanation**
Vault names are globally unique DNS labels, so a soft-deleted vault in any subscription blocks reuse.

**Code / CLI / Bicep**
```bash
az keyvault list-deleted -o table
az keyvault recover -n kv-demo-001
az keyvault purge -n kv-demo-001 --location westeurope   # fails if purge protection on
```
```bicep
resource kv 'Microsoft.KeyVault/vaults@2023-07-01' = {
  name: 'kv-demo-001'
  location: resourceGroup().location
  properties: {
    createMode: 'recover'
    tenantId: subscription().tenantId
    sku: { family: 'A', name: 'standard' }
  }
}
```

### 7.3 [Medium] How do you recover a deleted secret in code, and what permissions are needed?
**Answer**
Use `StartDeleteSecretAsync`/`StartRecoverDeletedSecretAsync`, which return long-running operations to await. Recover/purge need the right permission: under RBAC, Secrets Officer covers delete/recover but purge needs a role with the purge action (e.g., Key Vault Administrator or a custom role).

**Explanation**
After delete the secret returns 404 `SecretNotFound` for get until recovered; `GetDeletedSecretAsync` finds it. Verify exact role action names in current docs.

**Code / CLI / Bicep**
```csharp
DeleteSecretOperation del = await client.StartDeleteSecretAsync("ApiKey");
await del.WaitForCompletionAsync();
RecoverDeletedSecretOperation rec = await client.StartRecoverDeletedSecretAsync("ApiKey");
await rec.WaitForCompletionAsync();
```

### 7.4 [Tricky] Test pipelines create and delete vaults daily. What breaks, and what's the production consequence of enabling purge protection?
**Answer**
Names stay reserved for the retention period so recreating the same name fails; use unique suffixes or short retention in non-prod and purge protection only where needed. In production purge protection is irreversible and means you pay for and cannot remove the vault/keys until retention ends.

**Explanation**
Purge protection means even subscription deletion leaves the vault in soft-deleted state. Retention days cannot be changed after creation. CMK needs it, so plan names (include a unique string) and don't set it on throwaway vaults.

**Code / CLI / Bicep**
```bicep
param suffix string = uniqueString(resourceGroup().id)
var kvName = 'kv-orders-${suffix}'
```

**Gotchas / follow-ups**
- Retention is set at creation; immutable afterwards.
- Soft-deleted vault still counts towards name collisions and quotas.

**Interview traps**
- "Can I disable soft delete?" - no on new vaults.
- Purge protection cannot be undone.
- Assuming delete then create same name works instantly.

**Cheat sheet**
- Soft delete mandatory, 7-90 days.
- Purge protection = irreversible, required for CMK.
- `recover`, `purge`, `list-deleted`.
- Make names unique in IaC.

---

## Topic 8: Certificates

**Study notes**

A certificate object = policy + versions. Policy defines subject, SANs, key type/size, `exportable`, `reuse key`, content type (PKCS12/PEM), issuer, and lifetime actions (auto-renew at % lifetime or N days before expiry, or email-only).

| Issuer | Notes |
|---|---|
| Self | Self-signed (dev/test) |
| Integrated CAs (DigiCert, GlobalSign) | Auto-issue and renew (needs issuer config) |
| Unknown / non-integrated | KV creates CSR; you complete with `merge` |
| Import | Upload existing PFX/PEM; renewal only if policy and issuer set |

Consumers: App Service (import from vault; auto-syncs new version), Application Gateway (secretId reference with UAMI), Front Door, API Management.

**Ladder**

### 8.1 [Easy] How do you create a certificate and what does auto-renewal do?
**Answer**
Create it with a policy; a lifetime action can auto-renew at a percentage of lifetime or days before expiry by creating a new version. Works fully only with integrated CAs or self-signed; others need manual merge.

**Explanation**
Renewal creates a new version (and a new secret/key version if not `reuse key`). Consumers must pick up the new version.

**Code / CLI / Bicep**
```bash
az keyvault certificate get-default-policy > policy.json
az keyvault certificate create --vault-name kv-demo-001 -n web-cert -p @policy.json
az keyvault certificate import --vault-name kv-demo-001 -n imported --file cert.pfx --password "pw"
```

### 8.2 [Follow-up] How do you read/use a certificate from .NET?
**Answer**
Use `CertificateClient` for metadata/policy and `SecretClient` (or the `DownloadCertificate` helper) for the private key. `CertificateClient.DownloadCertificateAsync` returns an `X509Certificate2` with the private key when exportable.

**Explanation**
`GetCertificateAsync` returns only the public part (cer). The helper `DownloadCertificate` (in Azure.Security.KeyVault.Certificates) needs secret get permission to obtain the private key - verify exact method names in current docs.

**Code / CLI / Bicep**
```csharp
var certClient = new CertificateClient(vaultUri, new DefaultAzureCredential());
X509Certificate2 cert = await certClient.DownloadCertificateAsync("web-cert");
// private key present if exportable and caller has secrets/get
```

### 8.3 [Medium] How does App Service / Application Gateway bind to a Key Vault certificate and pick up renewals?
**Answer**
App Service imports the cert from the vault into its own store and syncs new versions automatically (roughly within a day - verify). App Gateway references the versionless secret ID with a user-assigned identity and polls for new versions (about every 4 hours - verify).

**Explanation**
Use the versionless secret URI so renewals flow through. App Service's resource provider identity (not your app's) needs secrets `get` on the vault for import. App Gateway needs a UAMI with Secrets User and network access (trusted services bypass applicability - verify).

**Code / CLI / Bicep**
```bicep
resource appgw 'Microsoft.Network/applicationGateways@2023-09-01' = {
  name: 'agw'
  location: resourceGroup().location
  identity: { type: 'UserAssigned', userAssignedIdentities: { '${uami.id}': {} } }
  properties: {
    sslCertificates: [
      { name: 'web', properties: { keyVaultSecretId: 'https://kv-demo-001.vault.azure.net/secrets/web-cert' } }
    ]
    // ...rest omitted
  }
}
```

### 8.4 [Hard] A renewed certificate isn't being served. Why?
**Answer**
Typical causes: the consumer pinned a versioned ID, sync interval hasn't elapsed, the identity lost access, the new version used a different CN/SANs, or the lifetime action only emails. Check cert versions and consumer's last-sync status.

**Explanation**
Also the app might cache `X509Certificate2` loaded at startup (Kestrel loads once) - you must restart or reload. For private-key export: `exportable=false` can't be undone; create a new cert version with new policy.

**Code / CLI / Bicep**
```bash
az keyvault certificate list-versions --vault-name kv-demo-001 -n web-cert -o table
az keyvault certificate show --vault-name kv-demo-001 -n web-cert --query "policy.lifetimeActions"
```

**Gotchas / follow-ups**
- CA-issued cert validation can be pending; check `certificate pending show`.
- Merge for unknown CAs requires full chain in the PEM.
- Importing PFX with non-exportable key flag not possible later.

**Interview traps**
- Thinking auto-renew works with any CA.
- Pinned version => no renewal flow.
- Cert read permission != private key permission (secrets get).
- App code caching old cert.

**Cheat sheet**
- Cert = policy + key + secret.
- Versionless secret URI for consumers.
- `CertificateClient` for management, secret for private key.
- Lifetime action: AutoRenew vs EmailContacts.

---

## Topic 9: Keys and crypto operations, CMK, BYOK

**Study notes**

Key types: RSA (2048/3072/4096), EC (P-256/384/521/P-256K), `-HSM` variants (Premium/Managed HSM). Operations: `sign/verify`, `encrypt/decrypt`, `wrapKey/unwrapKey`, plus key `get/rotate`. The private key never leaves the vault, so the app calls the service.

Envelope encryption:
```
data --AES-256 DEK (local)--> ciphertext
DEK --wrapKey (RSA KEK in Key Vault)--> wrapped DEK   (store next to ciphertext)
Decrypt: unwrapKey(wrapped DEK) -> DEK -> decrypt data
```
Do not send bulk data to the vault: RSA encrypt only handles small payloads and costs a network call.

**Ladder**

### 9.1 [Easy] Why use a Key Vault key rather than a secret?
**Answer**
With keys, the private material never leaves the vault: you send data/digests in and receive results. That limits blast radius and enables HSM protection and audit of every operation.

**Explanation**
Secrets are readable values; if an app is compromised the secret is exfiltrated. A key can only be used while the identity has access.

**Code / CLI / Bicep**
```bash
az keyvault key create --vault-name kv-demo-001 -n app-kek --kty RSA --size 3072 \
  --ops wrapKey unwrapKey
```

### 9.2 [Follow-up] Show sign and verify with CryptographyClient.
**Answer**
Hash the data locally, then call `SignAsync` with the algorithm and digest. Verification can occur locally with the public key or in the vault.

**Explanation**
`CryptographyClient` is created from the key ID (`KeyClient.GetCryptographyClient` or key URI). Role: Key Vault Crypto User.

**Code / CLI / Bicep**
```csharp
var keyClient = new KeyClient(vaultUri, new DefaultAzureCredential());
CryptographyClient crypto = keyClient.GetCryptographyClient("sign-key");
byte[] digest = SHA256.HashData(payload);
SignResult sig = await crypto.SignAsync(SignatureAlgorithm.RS256, digest);
VerifyResult ok = await crypto.VerifyAsync(SignatureAlgorithm.RS256, digest, sig.Signature);
```

### 9.3 [Medium] Implement envelope encryption: why wrap a DEK instead of encrypting data directly?
**Answer**
Generate a random AES key locally, encrypt data with it, and wrap that DEK with a Key Vault key. This is fast, supports large data, and costs one vault call per DEK; rotating the KEK means re-wrapping DEKs, not re-encrypting data.

**Explanation**
Store wrapped DEK, algorithm and key version with the ciphertext. Cache unwrapped DEKs briefly in memory to limit vault calls (watch throttling).

**Code / CLI / Bicep**
```csharp
byte[] dek = RandomNumberGenerator.GetBytes(32);
WrapResult wrapped = await crypto.WrapKeyAsync(KeyWrapAlgorithm.RsaOaep256, dek);
// store wrapped.EncryptedKey + wrapped.KeyId
UnwrapResult unwrapped = await crypto.UnwrapKeyAsync(KeyWrapAlgorithm.RsaOaep256, wrapped.EncryptedKey);
using var aes = new AesGcm(unwrapped.Key, 16);
```

### 9.4 [Hard] How do customer-managed keys (CMK) work for Storage/SQL, and what are the failure modes?
**Answer**
The service's identity (system or user-assigned) wraps/unwraps its own data encryption key with your Key Vault key. It needs `Key Vault Crypto Service Encryption User` (or wrap/unwrap permissions), and the vault needs soft delete + purge protection. If access is revoked or the key is disabled/deleted, the service loses access to data.

**Explanation**
Auto-rotation: use a versionless key URI so the service picks up new key versions (Storage checks about daily - verify). Key rotation policy on the key can rotate automatically. Firewall: trusted Microsoft services bypass is needed for some services.

**Code / CLI / Bicep**
```bicep
resource st 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: 'stdemocmk001'
  location: resourceGroup().location
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
  identity: { type: 'UserAssigned', userAssignedIdentities: { '${uami.id}': {} } }
  properties: {
    encryption: {
      keySource: 'Microsoft.Keyvault'
      identity: { userAssignedIdentity: uami.id }
      keyvaultproperties: { keyvaulturi: kv.properties.vaultUri, keyname: 'cmk' }
    }
  }
}
```

**Gotchas / follow-ups**
- BYOK: generate key on-prem HSM, wrap with a KEK exported from the vault (`az keyvault key import` / `key import --byok-file`; verify flags in current docs), then import - key never leaves HSM boundary in clear.
- Pinned version means manual rotation.
- Deleting CMK = data inaccessible; purge protection prevents permanent loss.

**Interview traps**
- Encrypting bulk data with the vault.
- Forgetting soft delete + purge protection for CMK.
- Using Crypto User instead of Crypto Service Encryption User for services.
- Thinking BYOK = CMK (BYOK is about key origin).

**Cheat sheet**
- KEK in vault, DEK local.
- `WrapKeyAsync` / `UnwrapKeyAsync`, `SignAsync`.
- CMK needs identity + role + purge protection.
- Versionless key URI for auto-rotation.

---

## Topic 10: Throttling and caching secrets in the app

**Study notes**

Key Vault enforces per-vault and per-region limits on transactions (for example, on the order of 4,000 GET secret transactions per 10 seconds on Standard vaults - verify in current docs). Excess requests return **HTTP 429** with a `Retry-After` header. Azure SDK retries 429 automatically (exponential back-off) but sustained overload still fails.

```
Bad : each request -> SecretClient.GetSecretAsync -> vault (latency + 429 risk)
Good: startup / interval -> load once -> in-memory cache -> request uses cache
```

**Ladder**

### 10.1 [Easy] Should you call Key Vault on every request?
**Answer**
No. Read secrets at startup or on a refresh interval and cache them in memory. Per-request calls add latency, cost, and risk 429 throttling.

**Explanation**
Also a Function scaled out to many instances multiplies calls. Use config provider or Key Vault references so the platform handles resolution.

**Code / CLI / Bicep**
```csharp
builder.Services.AddSingleton(_ => new SecretClient(vaultUri, new DefaultAzureCredential()));
```

### 10.2 [Follow-up] How would you implement a cached secret provider?
**Answer**
Register a singleton that wraps `SecretClient` and a memory cache with an absolute expiry, using a lock/lazy so concurrent misses do a single fetch.

**Explanation**
Keep the TTL below your rotation overlap window. On refresh failure, serve the stale value rather than failing.

**Code / CLI / Bicep**
```csharp
public sealed class CachedSecrets(SecretClient client, IMemoryCache cache)
{
    public Task<string> GetAsync(string name) =>
        cache.GetOrCreateAsync(name, async e =>
        {
            e.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(15);
            return (await client.GetSecretAsync(name)).Value.Value;
        })!;
}
```
Note: `GetOrCreateAsync` doesn't prevent concurrent factory runs; use `Lazy<Task<T>>` or `SemaphoreSlim` per key if that matters.

### 10.3 [Medium] How does 429 handling work and what do you do beyond SDK retries?
**Answer**
The SDK pipeline retries 429/5xx honoring `Retry-After`. Beyond that: cache, reduce secrets read per instance, split hot workloads across vaults, and avoid stampedes with jittered refresh.

**Explanation**
Tune via client options. Limits apply per vault and also per subscription+region across vaults for some operations - verify in current docs.

**Code / CLI / Bicep**
```csharp
var options = new SecretClientOptions
{
    Retry = { Mode = RetryMode.Exponential, MaxRetries = 6, Delay = TimeSpan.FromSeconds(1) }
};
var client = new SecretClient(vaultUri, cred, options);
```

### 10.4 [Tricky] A Function app with 200 scaled instances starts and many get 429 / timeouts. Why and what's the fix?
**Answer**
Every cold-started instance loads all secrets at once (thundering herd), consuming vault transaction quota. Fix with Key Vault references (platform resolves), load only needed secrets via filtered manager, jitter/stagger, and use a static client with cache.

**Explanation**
Another cause: creating `DefaultAzureCredential` per invocation, hammering the token endpoint (IMDS also throttles). Make clients static/singleton.

**Code / CLI / Bicep**
```csharp
// Isolated worker: register once
services.AddAzureClients(b =>
{
    b.AddSecretClient(vaultUri);
    b.UseCredential(new DefaultAzureCredential());
});
```

**Gotchas / follow-ups**
- `Microsoft.Extensions.Azure` `AddAzureClients` registers clients as singletons.
- Cached secrets vs rotation: add refresh-on-401/403 from the downstream service.

**Interview traps**
- New client per request.
- Cache without rotation awareness.
- Forgetting IMDS token throttling.
- Treating 429 as fatal.

**Cheat sheet**
- Singleton client + cache + TTL.
- 429 => back off with `Retry-After`.
- Filter config provider.
- Use references in Functions.

---

## Topic 11: Networking: firewall, private endpoint, trusted services, DNS

**Study notes**

| Option | Effect |
|---|---|
| Public access, no restrictions (default) | Reachable from internet, still needs Entra auth |
| Firewall (IP rules / VNet service endpoints) | Only listed IPs/subnets, `defaultAction: Deny` |
| Private endpoint | Private IP in your VNet; `privatelink.vaultcore.azure.net` DNS zone |
| Trusted services bypass (`bypass: AzureServices`) | Allows listed first-party services (e.g., Storage CMK, Disk Encryption, Backup) through the firewall |
| `publicNetworkAccess: Disabled` | Only private endpoint works |

```
App (VNet integrated) -> DNS kv.vault.azure.net -> CNAME kv.privatelink.vaultcore.azure.net -> 10.0.x.x
```

**Ladder**

### 11.1 [Easy] How do you restrict network access to a vault?
**Answer**
Set the firewall default action to Deny and allow specific IPs or VNet subnets, or disable public access and use a private endpoint. Authentication and RBAC still apply on top.

**Explanation**
Network rules are a second layer; they don't replace identity.

**Code / CLI / Bicep**
```bicep
resource kv 'Microsoft.KeyVault/vaults@2023-07-01' = {
  name: 'kv-demo-001'
  location: resourceGroup().location
  properties: {
    tenantId: subscription().tenantId
    sku: { family: 'A', name: 'standard' }
    enableRbacAuthorization: true
    publicNetworkAccess: 'Disabled'
    networkAcls: { defaultAction: 'Deny', bypass: 'AzureServices' }
  }
}
```

### 11.2 [Follow-up] What is "Allow trusted Microsoft services" and what does it NOT cover?
**Answer**
It lets a fixed list of Microsoft first-party services bypass the firewall (e.g., Storage CMK, Azure Backup, Disk Encryption, certain ARM deployments). It does not cover your own App Service, Functions or Logic Apps calling the vault, nor Key Vault references from App Service in general - verify the current trusted list in docs.

**Explanation**
Your own compute must come from an allowed VNet/subnet (service endpoint), allowed IP, or private endpoint. Bypass also requires that the service uses a managed identity.

**Code / CLI / Bicep**
```bash
az keyvault network-rule add -n kv-demo-001 --subnet <subnetId>
az keyvault update -n kv-demo-001 --default-action Deny --bypass AzureServices
```

### 11.3 [Medium] Set up a private endpoint properly.
**Answer**
Create a private endpoint with group ID `vault`, link a `privatelink.vaultcore.azure.net` private DNS zone to the VNet, and ensure the A record exists. Clients keep using the normal `*.vault.azure.net` name.

**Explanation**
Functions/App Service need regional VNet integration; outbound traffic to private IPs must route via the VNet (verify route-all settings). Hybrid/on-prem clients need DNS forwarders to Azure DNS.

**Code / CLI / Bicep**
```bicep
resource pe 'Microsoft.Network/privateEndpoints@2023-09-01' = {
  name: 'pe-kv'
  location: resourceGroup().location
  properties: {
    subnet: { id: subnetId }
    privateLinkServiceConnections: [{
      name: 'kv'
      properties: { privateLinkServiceId: kv.id, groupIds: ['vault'] }
    }]
  }
}
```

### 11.4 [Tricky] Private endpoint exists, yet the app gets "Client address is not authorized / Forbidden by firewall". Why?
**Answer**
Usually DNS: the name resolves to the public IP (private zone not linked to the VNet, custom DNS not forwarding), so traffic arrives via public path and gets blocked. Also check VNet integration and route settings.

**Explanation**
Test from inside the app with `nslookup`/`tcpping`. Error body mentions `ForbiddenByFirewall` and shows the caller IP and "vnet: false" style details. Another gotcha: with public access disabled, ARM/portal secret browsing from your laptop fails by design.

**Code / CLI / Bicep**
```bash
nslookup kv-demo-001.vault.azure.net
# expect 10.x.x.x via privatelink CNAME
az network private-dns link vnet list -g rg -z privatelink.vaultcore.azure.net -o table
```

**Gotchas / follow-ups**
- Cross-VNet peering needs the zone linked to every VNet that resolves.
- Pipeline agents (Microsoft-hosted) can't reach a private vault; use self-hosted agents.
- Soft-deleted vault + private endpoint recreation conflicts.

**Interview traps**
- Believing trusted services covers your apps.
- Firewall without VNet integration for Functions.
- Missing DNS zone link.
- Forgetting service-endpoint subnet must have `Microsoft.KeyVault` endpoint.

**Cheat sheet**
- Deny + IP/VNet rules or private endpoint.
- group ID `vault`; zone `privatelink.vaultcore.azure.net`.
- Trusted services = fixed 1st-party list.
- Debug with nslookup.

---

## Topic 12: Diagnostics, auditing, alerts

**Study notes**

Enable a **diagnostic setting** on the vault: category `AuditEvent` (all data-plane requests) and optionally `AzurePolicyEvaluationDetails`, plus `AllMetrics`. Send to Log Analytics (resource-specific or `AzureDiagnostics` table), Storage, or Event Hub. Logs are not on by default. Control-plane changes go to the Activity Log.

Key `AzureDiagnostics` columns: `OperationName` (SecretGet, SecretSet, KeyUnwrap, ...), `CallerIPAddress`, `identity_claim_*` (oid, appid, upn), `ResultSignature` (HTTP status), `requestUri_s`, `id_s`.

**Ladder**

### 12.1 [Easy] How do you audit Key Vault access?
**Answer**
Create a diagnostic setting that sends `AuditEvent` logs to a Log Analytics workspace, then query with KQL. Activity Log covers management operations.

**Explanation**
Logs are for the vault's own requests, including failures (403/429).

**Code / CLI / Bicep**
```bicep
resource diag 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = {
  name: 'to-law'
  scope: kv
  properties: {
    workspaceId: lawId
    logs: [{ category: 'AuditEvent', enabled: true }]
    metrics: [{ category: 'AllMetrics', enabled: true }]
  }
}
```

### 12.2 [Follow-up] KQL: who read a given secret in the last 7 days?
**Answer**
Filter `OperationName == "SecretGet"` and the secret in `requestUri_s`/`id_s`, then project caller identity and IP.

**Explanation**
Column names differ in resource-specific mode (`AZKVAuditLogs`); verify in current docs.

**Code / CLI / Bicep**
```kusto
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.KEYVAULT" and Category == "AuditEvent"
| where OperationName == "SecretGet" and requestUri_s contains "/secrets/SqlPassword"
| where TimeGenerated > ago(7d)
| project TimeGenerated, CallerIPAddress, identity_claim_oid_g, identity_claim_appid_g, ResultSignature
| order by TimeGenerated desc
```

### 12.3 [Medium] Which alerts would you configure?
**Answer**
Alert on spikes in 403/401 results, 429 throttling (ServiceApiResult metric or log query), unexpected `SecretSet`/`VaultDelete`/`SecretPurge`, access from unexpected IPs, and near-expiry events. Use Azure Monitor alert rules, Event Grid for near-expiry.

**Explanation**
Metrics available include `ServiceApiHit`, `ServiceApiLatency`, `ServiceApiResult`, `Availability`, `SaturationShoebox` (verify names in current docs). Microsoft Defender for Key Vault detects anomalous access.

**Code / CLI / Bicep**
```kusto
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.KEYVAULT" and ResultSignature in ("Forbidden","Unauthorized")
| summarize fails=count() by CallerIPAddress, identity_claim_oid_g, bin(TimeGenerated, 5m)
| where fails > 20
```

### 12.4 [Hard] Logs are on but you see no SecretGet for an app that works. Why?
**Answer**
Possibilities: the app uses Key Vault references (calls appear with the platform/App Service identity or its resolver, not your code's call pattern), cached values aren't hitting the vault, diagnostic setting points to a different workspace, ingestion delay (minutes), or the table name differs (resource-specific).

**Explanation**
Audit proves vault requests only; it doesn't show what happened to the value afterwards. Retention costs: AuditEvent can be high volume with large reload intervals.

**Code / CLI / Bicep**
```bash
az monitor diagnostic-settings list --resource $(az keyvault show -n kv-demo-001 --query id -o tsv)
```

**Gotchas / follow-ups**
- `ResultSignature` for success is `OK`.
- List operations (`SecretList`) don't reveal values.
- Log immutability: send to Storage with immutability for compliance.

**Interview traps**
- Assuming logs are enabled by default.
- Mixing up Activity Log and AuditEvent.
- Querying wrong table for resource-specific mode.
- Ignoring ingestion latency.

**Cheat sheet**
- Diagnostic setting -> `AuditEvent` -> Log Analytics.
- Fields: OperationName, CallerIPAddress, identity_claim_*.
- Alert: 403 spikes, purge, 429.
- Defender for Key Vault.

---

## Topic 13: Key Vault in CI/CD and Bicep

**Study notes**

- **Pipelines**: Azure Pipelines `AzureKeyVault@2` task (fetch secrets as pipeline variables) or variable groups linked to a vault; GitHub Actions uses OIDC federated login (`azure/login`) then `az keyvault secret show`. Prefer workload identity federation over stored client secrets.
- **Bicep**: `getSecret()` on an `existing` vault resource can only be passed to a module parameter marked `@secure()` (or a secure resource property). Requires the deployment principal to have `Microsoft.KeyVault/vaults/deploy/action` and `enabledForTemplateDeployment: true` on the vault.

**Ladder**

### 13.1 [Easy] How do you use a secret in a Bicep deployment without exposing it?
**Answer**
Reference the vault as `existing` and call `kv.getSecret('name')`, passing the result to a `@secure()` module parameter. The value is resolved by ARM and never appears in the template or deployment output.

**Explanation**
You cannot use `getSecret` directly in a resource in the same file as a property unless it's a secure one; the standard pattern is via a module.

**Code / CLI / Bicep**
```bicep
resource kv 'Microsoft.KeyVault/vaults@2023-07-01' existing = {
  name: 'kv-demo-001'
  scope: resourceGroup('rg-shared')
}

module sql 'sql.bicep' = {
  name: 'sql'
  params: {
    adminPassword: kv.getSecret('SqlAdminPassword')
  }
}
```
```bicep
// sql.bicep
@secure()
param adminPassword string
```

### 13.2 [Follow-up] What vault settings are needed for ARM to read secrets?
**Answer**
`enabledForTemplateDeployment: true`, and the deploying identity needs `Microsoft.KeyVault/vaults/deploy/action` plus data access (secret get via RBAC) as applicable. A firewall-restricted vault also needs ARM through the trusted services bypass.

**Explanation**
Role "Key Vault Secrets User" suffices for the value; the `deploy/action` is in roles like Contributor/Owner or a custom role. Verify current requirements in docs.

**Code / CLI / Bicep**
```bicep
resource kv 'Microsoft.KeyVault/vaults@2023-07-01' = {
  name: 'kv-demo-001'
  location: resourceGroup().location
  properties: {
    tenantId: subscription().tenantId
    sku: { family: 'A', name: 'standard' }
    enableRbacAuthorization: true
    enabledForTemplateDeployment: true
  }
}
```

### 13.3 [Medium] Pipeline: fetching secrets securely in Azure DevOps and GitHub Actions.
**Answer**
Use a service connection with workload identity federation (or OIDC in GitHub) granted Key Vault Secrets User on the vault, then fetch just the needed secrets. Secrets get masked in logs, but avoid echoing them.

**Explanation**
Better still: don't pull secrets into the pipeline; let the app resolve them at runtime via identity and only pipeline-deploy references.

**Code / CLI / Bicep**
```yaml
# Azure Pipelines
- task: AzureKeyVault@2
  inputs:
    azureSubscription: 'sc-prod'
    KeyVaultName: 'kv-demo-001'
    SecretsFilter: 'SqlAdminPassword,ApiKey'
    RunAsPreJob: false
```

### 13.4 [Hard] Create secrets from Bicep: what is the risk?
**Answer**
Creating `Microsoft.KeyVault/vaults/secrets` from Bicep with a plain value puts that value in deployment history/state and parameter files. Generate it at deploy time from another resource (e.g., `listKeys`) or write it out-of-band so it never lives in source control.

**Explanation**
Deployment history stores template parameters (secure ones are redacted) but `listKeys()` results used as secret value are fine when used inside the same deployment. Rotation-by-redeploy overwrites versions each run; use conditions to avoid creating a new version every pipeline run.

**Code / CLI / Bicep**
```bicep
resource st 'Microsoft.Storage/storageAccounts@2023-01-01' existing = { name: 'stdemo' }
resource secret 'Microsoft.KeyVault/vaults/secrets@2023-07-01' = {
  parent: kv
  name: 'StorageKey'
  properties: {
    value: st.listKeys().keys[0].value
  }
}
```

**Gotchas / follow-ups**
- Redeploys re-set the secret (new version) each time.
- Deploying principal needs Secrets Officer (control-plane secret writes also need rights).
- Prefer identity-based access to Storage rather than keys at all.

**Interview traps**
- Putting a secret value in a `.bicepparam` file.
- Missing `@secure()`.
- Forgetting `enabledForTemplateDeployment`.
- Hosted agents can't reach private vaults.

**Cheat sheet**
- `existing` + `getSecret()` -> `@secure()` param via module.
- OIDC / workload identity federation.
- Prefer runtime resolution.
- `enabledForTemplateDeployment: true`.

---

## Topic 14: Secrets in local development

**Study notes**

Order of preference for dev:
1. `az login` / Visual Studio sign-in + `DefaultAzureCredential` against a **dev vault** (no secrets on disk).
2. `dotnet user-secrets` for local-only values (stored outside repo, in user profile; not encrypted).
3. `local.settings.json` for Functions (git-ignored) with plain values.
4. Never commit secrets; add secret scanning (e.g., GitHub push protection).

**Ladder**

### 14.1 [Easy] How do you avoid committing secrets locally?
**Answer**
Use `dotnet user-secrets` or log in with `az login` and read from a dev Key Vault via `DefaultAzureCredential`. Keep `local.settings.json` in `.gitignore`.

**Explanation**
User-secrets is a convenience, not security: plain JSON under `%APPDATA%\Microsoft\UserSecrets`.

**Code / CLI / Bicep**
```bash
dotnet user-secrets init
dotnet user-secrets set "ConnectionStrings:Sql" "Server=...;"
az login
```

### 14.2 [Follow-up] Make the same code work locally and in Azure.
**Answer**
Use the config provider conditionally and rely on `DefaultAzureCredential`, which picks up the developer identity locally and the managed identity in Azure. Local fallback to user-secrets when vault URI is not configured.

**Explanation**
The developer needs RBAC on the dev vault (Secrets User). Developers shouldn't have access to prod vaults.

**Code / CLI / Bicep**
```csharp
var vaultUri = builder.Configuration["KeyVault:Uri"];
if (!string.IsNullOrEmpty(vaultUri))
    builder.Configuration.AddAzureKeyVault(new Uri(vaultUri), new DefaultAzureCredential());
if (builder.Environment.IsDevelopment())
    builder.Configuration.AddUserSecrets<Program>(); // added last so it overrides
```

### 14.3 [Medium] Functions locally: how do Key Vault references behave?
**Answer**
`@Microsoft.KeyVault(...)` in `local.settings.json` is not resolved by the Functions Core Tools; you get the literal text. Put real values locally or fetch from the vault in code.

**Explanation**
Some versions of tooling may differ - verify in current docs. Common approach: local.settings.json holds dev values; Azure holds references.

**Code / CLI / Bicep**
```json
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "FUNCTIONS_WORKER_RUNTIME": "dotnet-isolated",
    "ServiceBusConnection__fullyQualifiedNamespace": "sb-dev.servicebus.windows.net"
  }
}
```

### 14.4 [Tricky] `DefaultAzureCredential` works in Visual Studio but fails in the container / CI. Why?
**Answer**
The developer credentials (VS, CLI) aren't available in a container or agent. Provide `AZURE_CLIENT_ID`/`AZURE_TENANT_ID` plus workload identity or federated credentials, or mount a token source; do not bake a client secret into the image.

**Explanation**
In AKS use workload identity; in GitHub Actions OIDC. The wrong tenant (guest account) in VS/CLI also causes failures: set `TenantId` in credential options.

**Code / CLI / Bicep**
```csharp
var cred = new DefaultAzureCredential(new DefaultAzureCredentialOptions
{
    TenantId = "<tenant-guid>",
    ExcludeVisualStudioCredential = false
});
```

**Gotchas / follow-ups**
- Multiple accounts in VS: set `VisualStudioTenantId`.
- Conditional access/MFA breaking CLI tokens - `az login` again.

**Interview traps**
- Treating user-secrets as encrypted.
- Expecting references to resolve locally.
- Giving developers prod vault access.
- Committing `local.settings.json`.

**Cheat sheet**
- `az login` + dev vault.
- `user-secrets` for local only.
- Same code path via DefaultAzureCredential.
- Separate dev/prod vaults.

---

## Topic 15: Common errors and troubleshooting

**Study notes**

| Symptom | Likely cause |
|---|---|
| 401 Unauthorized | No/invalid token: wrong audience/tenant, expired token, no credential available |
| 403 Forbidden `ForbiddenByRbac` / "Caller is not authorized to perform action on resource" | Authenticated, but missing data-plane role (RBAC) or access policy |
| 403 `ForbiddenByFirewall` / "Client address is not authorized" | Network rule/private endpoint/DNS |
| 404 `SecretNotFound` | Wrong name/version, **disabled**-or-deleted secret, soft-deleted, wrong vault |
| 409 Conflict | Object currently being deleted/recovered or name in deleted state |
| 429 | Throttled; honor `Retry-After` |

**Ladder**

### 15.1 [Easy] 401 vs 403: what's the difference?
**Answer**
401 means the caller could not be authenticated (bad or missing token); 403 means they are authenticated but not authorized (RBAC, access policy or firewall). Look at the error code in the body to tell which authorization layer.

**Explanation**
Key Vault first challenges unauthenticated calls with 401 + `WWW-Authenticate` (the SDK handles this challenge flow - it's normal to see one 401 in traces).

**Code / CLI / Bicep**
```csharp
try { await client.GetSecretAsync("x"); }
catch (RequestFailedException ex) when (ex.Status == 403)
{
    logger.LogError("Forbidden {Code}: {Msg}", ex.ErrorCode, ex.Message);
}
```

### 15.2 [Follow-up] Role assigned but still "Caller is not authorized" - why?
**Answer**
RBAC propagation can take several minutes; also check scope (assigned to a different vault/resource group), principal (SAMI of a different slot, wrong UAMI), the vault's permission model (access policy mode ignores RBAC), and group-claims token caching.

**Explanation**
The message includes the caller's object ID (`oid`) and action - compare it to the identity you actually granted.

**Code / CLI / Bicep**
```bash
az keyvault show -n kv-demo-001 --query properties.enableRbacAuthorization
az role assignment list --all --assignee <oid> -o table
```

### 15.3 [Medium] SecretNotFound but you can see it in the portal.
**Answer**
Typical causes: the secret is disabled or deleted (soft-deleted), you are hitting a different vault/environment, the name has different casing/underscore mapping, or you requested a version that doesn't exist.

**Explanation**
Check "Manage deleted secrets". Also the configuration provider skips disabled secrets, so the key silently doesn't appear in config rather than throwing.

**Code / CLI / Bicep**
```bash
az keyvault secret list-deleted --vault-name kv-demo-001 -o table
az keyvault secret show --vault-name kv-demo-001 -n ApiKey --query attributes
```

### 15.4 [Hard] Intermittent 403 or timeouts only in production under load. Approach?
**Answer**
Correlate by time with AuditEvent `ResultSignature`/`ResultType`, metrics for 429/saturation, DNS/private endpoint health, and token acquisition errors. Check vault request volume per instance and caching.

**Explanation**
Possible causes: throttling beyond retries, SNAT port exhaustion on App Service reaching the public endpoint, IMDS throttling for token calls, firewall IP changes, failover DNS flipping.

**Code / CLI / Bicep**
```kusto
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.KEYVAULT" and TimeGenerated > ago(1h)
| summarize count() by ResultSignature, OperationName, bin(TimeGenerated, 1m)
| render timechart
```

**Gotchas / follow-ups**
- 409 "Conflict" when purge protection and recreating a deleted name.
- `AuthenticationFailedException` from Azure.Identity lists each credential that failed - read it fully.

**Interview traps**
- Treating 403 as 401.
- Not reading the error code/`oid`.
- Forgetting the vault's permission model.
- Not caching -> self-induced 429.

**Cheat sheet**
- 401 authN, 403 authZ/network, 404 missing/disabled/deleted, 429 throttle.
- Compare `oid` to granted principal.
- Wait for propagation.
- Use AuditEvent logs.

---

## Topic 16: Disaster recovery, multi-env design, what NOT to store

**Study notes**

- **Redundancy**: Vaults are replicated within the region and, for most regions, to the paired region (read-only failover during a regional outage; writes unavailable until failover - verify in current docs). Managed HSM offers multi-region replication.
- **Backup/restore**: per-object only (`secret backup`, `key backup`, `certificate backup`) - encrypted blob restorable only into a vault in the same subscription and geography. No whole-vault backup for standard vaults.
- **Multi-env**: one vault per app per environment (dev/test/prod); never share prod vaults with lower envs.
- **What not to store**: non-secret configuration (use **Azure App Configuration**; it can hold Key Vault references), large blobs, high-frequency runtime data.

```
App Configuration (non-secret settings, feature flags)
      |-- key: Sql:Server        = value
      `-- key: Sql:Password      = KeyVault reference -> Key Vault secret
```

**Ladder**

### 16.1 [Easy] How do you back up and restore Key Vault objects?
**Answer**
Use per-object backup commands to get an encrypted blob and restore it into another vault in the same subscription and geography. There is no one-click full vault backup for standard vaults.

**Explanation**
Backup includes all versions. Restores fail if the target has an existing/deleted object with that name. Better DR approach: treat vault content as reproducible via IaC + a secret-generation process.

**Code / CLI / Bicep**
```bash
az keyvault secret backup --vault-name kv-a -n ApiKey --file ApiKey.bak
az keyvault secret restore --vault-name kv-b --file ApiKey.bak
```

### 16.2 [Follow-up] What happens in a regional outage?
**Answer**
Key Vault fails over to the paired region in read-only mode: gets and crypto operations work, while writes (set, create, delete) are unavailable until recovery. Apps must therefore cache secrets and not need writes during an outage.

**Explanation**
Design: startup-time load plus cache; for hard regional independence, deploy a second vault per region and sync via a pipeline/Function, and have apps use the regional one. Network DNS/private endpoint must also be present in the failover region.

**Code / CLI / Bicep**
```bash
# Sync secrets to a secondary vault (illustrative)
for n in $(az keyvault secret list --vault-name kv-primary --query "[].name" -o tsv); do
  v=$(az keyvault secret show --vault-name kv-primary -n "$n" --query value -o tsv)
  az keyvault secret set --vault-name kv-secondary -n "$n" --value "$v" >/dev/null
done
```

### 16.3 [Medium] Design secret management for dev/test/prod across many services.
**Answer**
One vault per environment (and per workload/team where blast-radius matters), RBAC with UAMI per app granted Secrets User, secrets named identically across envs so code is env-agnostic, IaC-provisioned vaults, soft delete + purge protection in prod, diagnostic logs to central workspace, and policy for expiry/networking.

**Explanation**
Same secret name across vaults (`Sql--Password`) lets only the vault URI vary by env. Humans get no standing prod access (PIM, break glass). Prefer identity-based auth (managed identity to SQL/Storage/Service Bus) so fewer secrets exist.

**Code / CLI / Bicep**
```bicep
param env string
param workload string
var kvName = take('kv-${workload}-${env}-${uniqueString(resourceGroup().id)}', 24)
```

### 16.4 [Tricky] A teammate stores feature flags, URLs and connection strings of a prod DB all in Key Vault. Critique and fix.
**Answer**
Only the secret parts belong in Key Vault. URLs, flags and toggles are non-secret configuration and belong in App Configuration (with feature flags, labels per env, and Key Vault references for the secret entries). This reduces transaction load, simplifies changes, and keeps audit logs focused on real secrets.

**Explanation**
Also reconsider the DB connection string: use Entra auth with managed identity so no password exists. Key Vault vault name length limit: 3-24 chars; secret names 1-127 chars alphanumeric and dashes.

**Code / CLI / Bicep**
```csharp
builder.Configuration.AddAzureAppConfiguration(o =>
    o.Connect(new Uri(appConfigEndpoint), new DefaultAzureCredential())
     .ConfigureKeyVault(kv => kv.SetCredential(new DefaultAzureCredential()))
     .UseFeatureFlags());
```

**Gotchas / follow-ups**
- App Configuration KV references resolve client-side through the SDK, so the app identity needs vault access too.
- Backups can't cross subscriptions/geographies.
- Read-only failover: writes (e.g., rotation functions) fail during outage.

**Interview traps**
- Claiming a full-vault backup exists.
- One shared vault for all environments.
- Storing config and flags as secrets.
- Not planning for read-only failover mode.

**Cheat sheet**
- Backup per object; same sub + geo.
- Paired-region failover = read-only.
- Vault per env/workload; same names across envs.
- Non-secrets -> App Configuration; prefer managed identity auth.

