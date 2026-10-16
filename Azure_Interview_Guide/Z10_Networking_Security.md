# Z10 – Azure Networking & Security: Study Notes + Interview Ladder

**How to use:** Read each topic's *Study notes* first, then work the question ladder top-down (Easy → Tricky); cover the Answer and say it aloud before reading the Explanation.
Limits, SKUs, retirements and preview features change often – anywhere marked *verify in current docs*, check learn.microsoft.com before quoting numbers.

## Table of Contents
1. VNet, Subnets, NSGs, Route Tables, Peering
2. Public Endpoints vs Service Endpoints vs Private Endpoints (Private Link)
3. Private DNS Zones, the "Resolves to Public IP" Failure, Hub-Spoke DNS
4. App Service / Functions Networking
5. Securing Service Bus, Storage, Key Vault, Redis, SQL + Logic Apps Standard
6. App Gateway vs Front Door vs Traffic Manager vs Load Balancer + WAF
7. API Management (Policies, Tiers, Identity, VNet)
8. Azure Firewall, NSG/ASG, DDoS Protection
9. TLS/Certificates, Secrets & Zero-Secret Architecture
10. Defender for Cloud, Sentinel, Azure Policy Guardrails
11. Azure Bastion & the Network Troubleshooting Toolkit
12. Hub-and-Spoke Reference Design
13. Common Production Errors (Playbook)
14. Compliance, Data Residency & End-to-End Secure Design

---

## Topic 1: VNet, Subnets, NSGs, Route Tables, Peering

**Study notes**

- A **VNet** is a regional, isolated private address space (RFC1918 recommended). **Subnets** carve it up; Azure reserves **5 IPs per subnet** (network, gateway, 2 DNS, broadcast). Plan CIDRs up front – overlapping ranges cannot be peered.
- **NSG** = stateful L3/L4 allow/deny, attached to a subnet and/or NIC. Rules evaluated by priority (100–4096, low number first). Defaults: `AllowVnetInBound` (65000), `AllowAzureLoadBalancerInBound` (65001), `DenyAllInBound` (65500); outbound: `AllowVnetOutBound`, `AllowInternetOutBound`, `DenyAllOutBound`. Use **service tags** (`Storage`, `Sql`, `AzureKeyVault`, `ServiceBus`) and **ASGs** instead of IP lists.
- **System routes** exist by default (VNet, peering, Internet 0.0.0.0/0). A **route table (UDR)** overrides them; most specific prefix wins (longest prefix match), then UDR > BGP > system on ties.
- **Peering** links two VNets over the Microsoft backbone; it is **non-transitive**. Options: allow forwarded traffic, allow gateway transit / use remote gateways. Works cross-region (global peering) and cross-subscription/tenant.

```
        Spoke A 10.1.0.0/16 <--peer--> Hub 10.0.0.0/16 <--peer--> Spoke B 10.2.0.0/16
        (A cannot reach B unless traffic is routed through a hub NVA/Firewall via UDRs)

  Subnet snet-app (10.1.1.0/24)
     NSG: 100 Allow 443 from AppGW subnet ; 4000 Deny all inbound
     UDR: 0.0.0.0/0 -> VirtualAppliance 10.0.1.4 (Azure Firewall)
```

### 1.1 [Easy] What is a VNet and what does a subnet give you?
**Answer:** A VNet is a private, regional network boundary in Azure; subnets are address-range segments inside it where you place resources and attach NSGs and route tables. Security and routing policy is applied per subnet.
**Explanation:** Resources in one VNet can talk to each other by default (via `AllowVnetInBound`). Subnets matter because NSGs, UDRs, service endpoints, delegations (e.g., `Microsoft.Web/serverFarms` for App Service integration) are all subnet-scoped. Five addresses are reserved in each subnet, so a /29 yields 3 usable IPs.
**Code / CLI / Bicep**
```bicep
resource vnet 'Microsoft.Network/virtualNetworks@2023-11-01' = {
  name: 'vnet-spoke-app'
  location: resourceGroup().location
  properties: {
    addressSpace: { addressPrefixes: ['10.1.0.0/16'] }
    subnets: [
      { name: 'snet-func',  properties: { addressPrefix: '10.1.1.0/24' } }
      { name: 'snet-pe',    properties: { addressPrefix: '10.1.2.0/24' } }
    ]
  }
}
```

### 1.2 [Follow-up] How do NSG rules get evaluated, and what happens with an NSG on both subnet and NIC?
**Answer:** Rules are processed lowest priority number first and the first match wins. With NSGs on both subnet and NIC, inbound traffic hits the subnet NSG first then the NIC NSG; outbound is NIC first, then subnet. Both must allow.
**Explanation:** NSGs are stateful: allowing inbound automatically allows the return flow. Default rules can't be deleted but can be overridden by a lower-numbered rule. Prefer subnet-level NSGs for manageability; use ASGs to group VMs by role. Enable NSG flow logs / VNet flow logs for audit (verify retirement timeline of NSG flow logs in current docs).
**Code / CLI / Bicep**
```bash
az network nsg create -g rg-net -n nsg-app
az network nsg rule create -g rg-net --nsg-name nsg-app -n AllowHttpsFromAppGw \
  --priority 100 --direction Inbound --access Allow --protocol Tcp \
  --source-address-prefixes 10.0.5.0/24 --destination-port-ranges 443
az network vnet subnet update -g rg-net --vnet-name vnet-spoke-app -n snet-func --network-security-group nsg-app
```

### 1.3 [Medium] Explain UDRs and how you force all egress through a firewall. What are the pitfalls?
**Answer:** Create a route table with `0.0.0.0/0` next hop `VirtualAppliance` pointing at the firewall private IP and associate it with workload subnets. Pitfalls: asymmetric routing, forgetting required management traffic, and breaking PaaS/VNet-integration dependencies.
**Explanation:** The firewall needs rules for everything the workload needs (Azure Monitor, Entra, storage for Functions content, etc.) or use FQDN tags/service tags. Return traffic must also traverse the same firewall (symmetric) – watch for peering with spokes that have their own routes. Some services (e.g., APIM internal mode, App Service Environment) need specific management routes/tags – read their doc. Disable BGP route propagation if you must not learn on-prem routes.
**Code / CLI / Bicep**
```bicep
resource rt 'Microsoft.Network/routeTables@2023-11-01' = {
  name: 'rt-spoke-default'
  location: resourceGroup().location
  properties: {
    disableBgpRoutePropagation: true
    routes: [ {
      name: 'default-to-fw'
      properties: { addressPrefix: '0.0.0.0/0', nextHopType: 'VirtualAppliance', nextHopIpAddress: '10.0.1.4' }
    } ]
  }
}
```

### 1.4 [Medium] Is VNet peering transitive? How do you make spoke-to-spoke work?
**Answer:** No, peering is non-transitive. Route spoke-to-spoke through a hub NVA/Azure Firewall with UDRs in each spoke (and "allow forwarded traffic" on peerings), or use Azure Virtual WAN / AVNM connectivity configs.
**Explanation:** Each peering is a pair of one-way links; both must exist (portal creates both). Gateway transit lets spokes use the hub VPN/ExpressRoute gateway: hub side "Allow gateway transit", spoke side "Use remote gateways". Address spaces cannot overlap, and changing a peered VNet's address space requires a peering resync (verify current behavior).
**Code / CLI / Bicep**
```bash
az network vnet peering create -g rg-net -n spokeA-to-hub --vnet-name vnet-spokeA \
  --remote-vnet /subscriptions/<sub>/resourceGroups/rg-hub/providers/Microsoft.Network/virtualNetworks/vnet-hub \
  --allow-vnet-access --allow-forwarded-traffic
```

### 1.5 [Tricky] A VM in a spoke can ping the hub but not another spoke, although both are peered to the hub. NSGs allow it. Why, and how do you diagnose?
**Answer:** Non-transitive peering – there is no route between spokes. Add UDRs to a hub firewall/NVA with forwarding enabled, and enable "allow forwarded traffic" on the peerings.
**Explanation:** Check the NIC's *effective routes*: the other spoke prefix will be absent (or next hop None). Network Watcher *Next hop* confirms. The NVA must have IP forwarding enabled on its NIC (and OS-level forwarding). Azure Firewall handles this natively.
**Code / CLI / Bicep**
```bash
az network nic show-effective-route-table -g rg-net -n vm1-nic -o table
az network watcher show-next-hop -g rg-net --vm vm1 --source-ip 10.1.1.4 --dest-ip 10.2.1.4
```
**Gotchas / follow-ups**
- A /16 UDR to the firewall in both spokes is needed (symmetric). Without it, one-way routing silently drops.
- If a spoke subnet UDR is more specific than the peering route, it wins – intended or not.
- Default outbound internet access for VMs without explicit egress is being retired for new VNets (private subnets) – verify in current docs; plan NAT Gateway / Firewall egress.

**Interview traps**
- "Peering is transitive" – it is not; gateway transit is the only special case.
- Forgetting 5 reserved IPs and subnet delegation exclusivity.
- NSG rules by priority: lowest number wins, not "most specific".
- Overlapping CIDRs discovered after the fact – cannot peer.

**Cheat sheet**
- 5 reserved IPs/subnet; NSG stateful; priority 100–4096.
- Longest prefix wins; UDR beats system route on same prefix.
- Peering: non-transitive, both directions, no overlap.
- Debug: effective routes, Next hop, IP flow verify.
- Use service tags + ASGs, not raw IPs.

---

## Topic 2: Public Endpoints vs Service Endpoints vs Private Endpoints (Private Link)

**Study notes**

Three ways to reach a PaaS service (Storage, Key Vault, SQL, Service Bus, Redis):

| Option | Server-side IP seen | Traffic path | Reachable from on-prem | Disables public access? |
|---|---|---|---|---|
| Public endpoint + firewall | Public IP | Internet/Microsoft backbone | Via public IP allow-list | No |
| Service endpoint | Public IP (source = VNet identity) | Backbone, subnet-scoped | No (Azure subnet only) | Allows VNet rule, keeps public endpoint |
| Private endpoint | **Private IP in your subnet** | Backbone to NIC in your VNet | Yes (VPN/ER + DNS) | Yes – can set public network access Disabled |

```
 App (snet-func) --DNS--> mystorage.blob.core.windows.net
        CNAME -> mystorage.privatelink.blob.core.windows.net -> A 10.1.2.5
        \--TCP 443--> [Private Endpoint NIC 10.1.2.5] ==Private Link==> Storage account (blob sub-resource)
```
A private endpoint targets one **sub-resource** (group ID: `blob`, `queue`, `vault`, `namespace`, `sqlServer`, `redisCache`…). Cost: per-hour + per-GB processed.

### 2.1 [Easy] What is a private endpoint?
**Answer:** A network interface with a private IP from your subnet that maps to a specific PaaS resource via Azure Private Link. Traffic stays on the Microsoft backbone and you can turn off the service's public endpoint.
**Explanation:** It is resource- and sub-resource-specific (not the whole service), so it also mitigates data exfiltration to other accounts. Needs DNS so the service FQDN resolves to the private IP (Topic 3). Approval can be automatic or manual for cross-tenant.
**Code / CLI / Bicep**
```bicep
resource pe 'Microsoft.Network/privateEndpoints@2023-11-01' = {
  name: 'pe-stg-blob'
  location: resourceGroup().location
  properties: {
    subnet: { id: peSubnetId }
    privateLinkServiceConnections: [ {
      name: 'stg-blob'
      properties: { privateLinkServiceId: storageId, groupIds: ['blob'] }
    } ]
  }
}
```

### 2.2 [Follow-up] Service endpoint vs private endpoint – which and why?
**Answer:** Service endpoints extend VNet identity to a service's public IP and are free and simple, but the service keeps a public IP and isn't reachable from on-prem. Private endpoints give a private IP, work from on-prem and peered VNets, and allow full public disable – preferred for zero-trust.
**Explanation:** Service endpoint = subnet toggle + firewall VNet rule on the resource. No DNS change. Doesn't help across peering without enabling on each subnet, and only for same-region (storage also paired region). Private endpoints cost money and need DNS work, but are the strategic option. Service endpoint policies can restrict which storage accounts a subnet can use.
**Code / CLI / Bicep**
```bash
az network vnet subnet update -g rg-net --vnet-name vnet-spoke-app -n snet-func --service-endpoints Microsoft.Storage Microsoft.KeyVault
az storage account network-rule add -g rg-app --account-name mystorage --vnet-name vnet-spoke-app --subnet snet-func
```

### 2.3 [Medium] After creating a private endpoint, what else must you configure and what does "public network access: Disabled" do?
**Answer:** You need a private DNS zone linked to the VNet(s) with an A record (via a DNS zone group), and approved connection state. Setting public network access Disabled blocks all public traffic and the firewall rules, leaving only private endpoint access.
**Explanation:** Use `privateDnsZoneGroups` so Azure auto-registers/updates the A record. Disabling public access also breaks any client not on the private path – CI/CD agents, portal data-plane browsing from your laptop, Microsoft-hosted agents. "Trusted Azure services" exceptions work only if public access is enabled with firewall (service-specific; Topic 5).
**Code / CLI / Bicep**
```bicep
resource zg 'Microsoft.Network/privateEndpoints/privateDnsZoneGroups@2023-11-01' = {
  parent: pe
  name: 'default'
  properties: { privateDnsZoneConfigs: [ { name: 'blob', properties: { privateDnsZoneId: blobZoneId } } ] }
}
```

### 2.4 [Medium] Can a private endpoint be reached from on-premises or a peered VNet? Do NSGs apply?
**Answer:** Yes, via VPN/ExpressRoute or peering, provided DNS resolves the private IP. NSG and UDR support on private endpoint subnets requires enabling network policies (`privateEndpointNetworkPolicies`) – historically disabled by default.
**Explanation:** Set `privateEndpointNetworkPolicies: Enabled` (or `RouteTableEnabled`/`NetworkSecurityGroupEnabled` in newer API – verify in current docs) on the PE subnet to apply NSG rules and UDRs to PE traffic. Without it, NSGs are ignored for the PE NIC. Putting PEs in a dedicated subnet is common.
**Code / CLI / Bicep**
```bash
az network vnet subnet update -g rg-net --vnet-name vnet-spoke-app -n snet-pe --private-endpoint-network-policies Enabled
```

### 2.5 [Hard] Explain Private Link internals: what is `groupId`, how is data exfiltration reduced, and what is a Private Link Service?
**Answer:** A group ID selects the sub-resource (e.g., `blob`, `vault`, `namespace`). The PE is a 1:1 mapping to a specific resource instance, so a workload with only that PE can't reach other tenants' accounts through it. A Private Link Service lets you expose your own Standard Load Balancer-fronted service privately to consumers' PEs.
**Explanation:** Traffic via PE is NATed/proxied by the platform to the target, the consumer never sees provider's VNet. For storage you may need separate PEs per sub-resource (blob, queue, table, file, dfs). Functions needs blob, queue, table (and file for content share) PEs for its host storage when storage public access is disabled. Providers can enable proxy protocol v2 on PLS to see client IP.
**Code / CLI / Bicep**
```bash
az network private-link-resource list --id <storageResourceId> -o table   # lists valid group IDs
```
**Gotchas / follow-ups**
- Separate PE per sub-resource -> separate DNS zone entries (`privatelink.blob...`, `privatelink.queue...`).
- Cross-region PE is allowed (PE in region A to resource in B); DNS still local.
- PEs are billed hourly plus data; many PEs add up – share via hub where design allows.

### 2.6 [Tricky] Public access disabled, PE approved, DNS looks right – but your Function app still gets 403. Name three causes.
**Answer:** (1) Wrong sub-resource PE (e.g., only blob, but the SDK calls queue); (2) the Function is not actually routing through the VNet (no VNet integration or route-all off, so DNS/traffic goes out via public path); (3) data-plane RBAC/auth is missing – networking and identity are independent.
**Explanation:** A 403 `AuthorizationFailure`/"not authorized to perform this operation" for storage usually means the request arrived via a path the firewall rejects. 403 `AuthorizationPermissionMismatch` means RBAC. Check which IP the service saw (diagnostic logs `CallerIpAddress`).
**Code / CLI / Bicep**
```kusto
StorageBlobLogs
| where StatusCode == 403
| project TimeGenerated, CallerIpAddress, AuthenticationType, StatusText, Uri
```

**Interview traps**
- "Service endpoint = private" – the service still has a public IP.
- Private endpoint alone doesn't secure anything until public access is disabled/firewalled.
- One PE per sub-resource; Functions host storage needs several.
- Forgetting DNS is the #1 cause of PE failures.

**Cheat sheet**
- PE = private IP + DNS; SE = subnet identity to public IP.
- Disable public network access to truly close the door.
- Enable PE network policies for NSG/UDR to apply.
- `az network private-link-resource list` shows group IDs.
- Trusted-services bypass only matters while public access is on.

---

## Topic 3: Private DNS Zones, the "Resolves to Public IP" Failure, Hub-Spoke DNS

**Study notes**

- Azure-provided DNS at **168.63.129.16** (virtual IP, reachable only from inside Azure VNets) resolves private DNS zones **linked** to the VNet.
- A PE-enabled service FQDN becomes a CNAME into `privatelink.*`. If the VNet resolver can't find that zone, the CNAME chain falls back to the public IP.
- Common zones: `privatelink.blob.core.windows.net`, `privatelink.queue.core.windows.net`, `privatelink.table.core.windows.net`, `privatelink.file.core.windows.net`, `privatelink.vaultcore.azure.net`, `privatelink.servicebus.windows.net`, `privatelink.database.windows.net`, `privatelink.redis.cache.windows.net`, `privatelink.azurewebsites.net` (verify the full list in current docs; zone names differ by cloud).
- On-prem / hub-spoke: custom DNS servers or **Azure DNS Private Resolver** (inbound endpoint for on-prem -> Azure, outbound endpoint + forwarding ruleset for Azure -> on-prem).

```
 Spoke VM --> VNet DNS = Hub DNS Resolver inbound IP (10.0.4.4)
                |  (resolver VNet is linked to privatelink.* zones)
                v
   mystorage.blob.core.windows.net -> CNAME mystorage.privatelink.blob.core.windows.net -> 10.1.2.5

 On-prem DNS: conditional forwarder  blob.core.windows.net  ->  10.0.4.4 (resolver inbound)
```

### 3.1 [Easy] Why does a private DNS zone matter for private endpoints?
**Answer:** The client still uses the normal service FQDN; the private DNS zone makes that name resolve to the PE's private IP inside your network. Without it the name resolves to the public IP and the traffic bypasses (or is blocked by) the private path.
**Explanation:** Create the zone with the exact `privatelink.*` name, add an A record for the resource, and link the zone to each VNet that must resolve it. A DNS zone group on the PE manages the record automatically.
**Code / CLI / Bicep**
```bicep
resource zone 'Microsoft.Network/privateDnsZones@2020-06-01' = { name: 'privatelink.blob.core.windows.net', location: 'global' }
resource link 'Microsoft.Network/privateDnsZones/virtualNetworkLinks@2020-06-01' = {
  parent: zone
  name: 'link-spoke'
  location: 'global'
  properties: { virtualNetwork: { id: vnetId }, registrationEnabled: false }
}
```

### 3.2 [Follow-up] Classic failure: after enabling a private endpoint, `nslookup` still returns a public IP. Why?
**Answer:** The resolving machine isn't using a DNS path that sees the private zone: the zone isn't linked to its VNet, the VNet uses custom DNS that doesn't forward to Azure DNS, or the client is on-prem with no conditional forwarder.
**Explanation:** Checklist: (1) is the A record present in the zone; (2) is the zone linked to the VNet where the client runs (or to the hub whose resolver is used); (3) what are the VNet's DNS servers (custom DNS must forward `privatelink` or the public zone to 168.63.129.16); (4) is the Function using VNet integration with DNS going through the VNet; (5) client-side DNS cache / DoH. Run `nslookup` from the actual compute (Kudu console), not your laptop.
**Code / CLI / Bicep**
```bash
nslookup mystorage.blob.core.windows.net
# expect: Name: mystorage.privatelink.blob.core.windows.net  Address: 10.1.2.5
az network private-dns record-set a list -g rg-dns -z privatelink.blob.core.windows.net -o table
az network private-dns link vnet list -g rg-dns -z privatelink.blob.core.windows.net -o table
```

### 3.3 [Medium] Design DNS for hub-and-spoke with on-prem. Who links which zones?
**Answer:** Centralize `privatelink.*` zones in the hub (or a connectivity subscription), link them to the hub VNet, and run Azure DNS Private Resolver there. Spokes use the resolver's inbound IP as custom DNS; on-prem forwards the public service zones to the resolver inbound endpoint.
**Explanation:** Linking zones to every spoke also works (links are cheap) and avoids a resolver dependency but doesn't solve on-prem. Resolver outbound rules forward on-prem domains (e.g., `corp.contoso.com`) to on-prem DNS. Avoid forwarding loops. Use Azure Policy (`DeployIfNotExists`) to auto-create zone group entries when a PE is created.
**Code / CLI / Bicep**
```bicep
resource resolver 'Microsoft.Network/dnsResolvers@2022-07-01' = {
  name: 'dnspr-hub'
  location: resourceGroup().location
  properties: { virtualNetwork: { id: hubVnetId } }
}
// then dnsResolvers/inboundEndpoints (own /28+ subnet) and outboundEndpoints + dnsForwardingRulesets
```

### 3.4 [Medium] What is `registrationEnabled` on a zone link and does it apply to privatelink zones?
**Answer:** Auto-registration creates A records for VMs in the linked VNet; it's for your own private zones (e.g., `contoso.internal`) and must be false for `privatelink.*` zones, which are populated by PE zone groups.
**Explanation:** Only one auto-registration zone per VNet. For privatelink zones, records are managed by the PE's DNS zone group. Manual records are fine but drift when PEs are recreated (new IP).
**Code / CLI / Bicep**
```bash
az network private-dns link vnet create -g rg-dns -z privatelink.vaultcore.azure.net -n link-hub -v vnet-hub -e false
```

### 3.5 [Hard] Split-brain: a service has both public and private clients. How does resolution behave and what about same-FQDN across tenants/subscriptions?
**Answer:** Public clients get the public CNAME chain ending in a public IP; clients in linked VNets get the private A record because the `privatelink` zone shadows the public CNAME target. For cross-subscription PEs, the zone must live somewhere linked to the consumer's VNet.
**Explanation:** The public DNS CNAMEs `acct.blob.core.windows.net` to `acct.privatelink.blob.core.windows.net` for all PE-enabled accounts, so overriding `privatelink.blob...` privately works without touching the public zone. Gotcha: if you create a zone `blob.core.windows.net` (not privatelink) you'd break every other account – never do that; use conditional forwarding for a specific FQDN only.
**Code / CLI / Bicep**
```bash
# from on-prem, verify forwarding to resolver inbound:
nslookup mystorage.blob.core.windows.net 10.0.4.4
```
**Gotchas / follow-ups**
- A VNet linked to multiple zones with the same name is not allowed; one `privatelink.blob` zone per VNet scope.
- Zones holding stale A records after PE delete/recreate -> connect to wrong IP.
- Two different Private DNS zones (one per spoke) for the same service leads to inconsistent resolution – centralize.

### 3.6 [Tricky] Function app has VNet integration and the zone is linked, but still resolves public. What Function-specific setting do you check?
**Answer:** Whether DNS is routed via the VNet: App Service uses the VNet's DNS when route-all (`WEBSITE_VNET_ROUTE_ALL=1` / `vnetRouteAllEnabled`) is on; and custom DNS can be overridden with `WEBSITE_DNS_SERVER` (and `WEBSITE_DNS_ALT_SERVER`). Verify exact behavior in current docs.
**Explanation:** Historically, private-zone resolution for integrated apps depended on route-all; newer behavior sends RFC1918 traffic via VNet regardless but non-RFC1918 and DNS traffic still depends on settings. If an app setting points `WEBSITE_DNS_SERVER` to a wrong server, resolution fails/leaks. Test from Kudu with `nameresolver <fqdn>`.
**Code / CLI / Bicep**
```bash
az functionapp config appsettings list -g rg-app -n func-orders --query "[?starts_with(name,'WEBSITE_')]" -o table
```

**Interview traps**
- "I created the private endpoint so it works" – DNS is half the job.
- Testing `nslookup` from a laptop instead of the compute.
- Creating a zone named `blob.core.windows.net`.
- Forgetting VNet links on every VNet that needs resolution.

**Cheat sheet**
- 168.63.129.16 = Azure DNS; custom DNS must forward to it.
- Link zone to VNet; zone group manages A records.
- Hub: DNS Private Resolver inbound/outbound + forwarding ruleset.
- On-prem forwards public service zone -> resolver inbound IP.
- Debug with `nslookup`/`nameresolver` from the workload itself.

---

## Topic 4: App Service / Functions Networking

**Study notes**

- **Inbound**: public endpoint + **access restrictions** (IP/service tag/VNet rules, separate rules for the SCM/Kudu site), or **private endpoint** (inbound private only). Front with App Gateway/Front Door and restrict by service tag/header `X-Azure-FDID`.
- **Outbound**: **regional VNet integration** (needs a dedicated, delegated subnet; delegation `Microsoft.Web/serverFarms` – Flex Consumption uses `Microsoft.App/environments`). It affects outbound only. Supported on Premium (EP), Dedicated (Basic+), Standard/Premium App Service plans, and Flex Consumption; **not** classic Consumption (verify in current docs).
- Routing: RFC1918 traffic goes via the VNet; everything else follows the **route-all** setting (`WEBSITE_VNET_ROUTE_ALL=1`, now a site property `vnetRouteAllEnabled` – verify). Plus `WEBSITE_CONTENTOVERVNET=1` to route content share over VNet.
- **Outbound IP**: shared set listed in `outboundIpAddresses`/`possibleOutboundIpAddresses`; for a stable single IP use **NAT Gateway** on the integration subnet (+ route-all).
- **Premium/Flex VNet triggers**: Service Bus/Event Hubs/Storage Queue triggers behind a VNet work on Premium (enable *runtime scale monitoring*) and Flex Consumption.

```
 Internet -> [Front Door/AppGW + WAF] -> (Access restriction / Private Endpoint 10.1.2.10) -> Function App
 Function App --VNet integration (snet-func, delegated)--> NAT GW (static IP) --> Internet
                                             \-> PE (Storage/SB/KV/SQL/Redis)
```

### 4.1 [Easy] VNet integration vs private endpoint on a Function app – what's the difference?
**Answer:** VNet integration controls outbound traffic from the app into a VNet; a private endpoint controls inbound traffic to the app from a VNet. They solve opposite directions and are often used together.
**Explanation:** Integration does not give the app a private inbound IP, and a PE doesn't let the app reach private resources. A fully private Function typically has both. The PE for the app uses sub-resource `sites` and zone `privatelink.azurewebsites.net`.
**Code / CLI / Bicep**
```bash
az functionapp vnet-integration add -g rg-app -n func-orders --vnet vnet-spoke-app --subnet snet-func
az functionapp config appsettings set -g rg-app -n func-orders --settings WEBSITE_VNET_ROUTE_ALL=1
```

### 4.2 [Follow-up] What are access restrictions, and how do they interact with private endpoints?
**Answer:** Access restrictions are an allow/deny list (IP, service tag, VNet subnet) for the public inbound endpoint, evaluated by priority, with an implicit deny-all once any allow rule exists. When a private endpoint is added and public access is off, the public endpoint is blocked regardless.
**Explanation:** Main site and SCM (Kudu) site have independent rules (or "use main site rules"). Service-endpoint-based rules need the service endpoint on the source subnet. For App Gateway/Front Door, use service tag `AzureFrontDoor.Backend` plus header filtering on `X-Azure-FDID`.
**Code / CLI / Bicep**
```bash
az functionapp config access-restriction add -g rg-app -n func-orders \
  --rule-name AllowFD --priority 100 --service-tag AzureFrontDoor.Backend \
  --http-header x-azure-fdid=<frontDoorId>
```

### 4.3 [Medium] Your Function must call a partner API that allow-lists a single IP. How?
**Answer:** Enable VNet integration, route all outbound traffic (route-all), and attach a NAT Gateway with a static public IP to the integration subnet. The partner allow-lists that IP.
**Explanation:** Without it, outbound IPs are a shared pool that can change on scale/plan changes. NAT Gateway also massively increases SNAT capacity (~64k ports per public IP; verify limits). If a UDR sends 0.0.0.0/0 to a firewall, egress IP becomes the firewall's instead.
**Code / CLI / Bicep**
```bicep
resource pip 'Microsoft.Network/publicIPAddresses@2023-11-01' = { name: 'pip-nat', location: loc, sku: { name: 'Standard' }, properties: { publicIPAllocationMethod: 'Static' } }
resource nat 'Microsoft.Network/natGateways@2023-11-01' = {
  name: 'nat-func', location: loc, sku: { name: 'Standard' }
  properties: { publicIpAddresses: [ { id: pip.id } ], idleTimeoutInMinutes: 4 }
}
// subnet snet-func: natGateway: { id: nat.id }
```

### 4.4 [Medium] A Premium-plan Service Bus-triggered function stops scaling/triggering once Service Bus is private. What do you check?
**Answer:** Enable runtime scale monitoring on the Function app (`functionsRuntimeScaleMonitoringEnabled`) and set pre-warmed/min instances; ensure VNet integration can resolve and reach the namespace PE and the identity has Data Receiver role.
**Explanation:** The Premium scale controller is outside your VNet and can't read queue length from a private namespace; with runtime scale monitoring the instance(s) in the VNet report metrics. Flex Consumption handles VNet triggers natively. Use identity-based connection `ServiceBusConnection__fullyQualifiedNamespace`.
**Code / CLI / Bicep**
```bash
az resource update -g rg-app -n func-orders/config/web --resource-type Microsoft.Web/sites/config \
  --set properties.functionsRuntimeScaleMonitoringEnabled=1
az functionapp config appsettings set -g rg-app -n func-orders --settings \
  ServiceBusConnection__fullyQualifiedNamespace=sb-prod.servicebus.windows.net
```

### 4.5 [Hard] Walk through making the Function host storage private. What breaks if you miss something?
**Answer:** Create PEs for blob, queue, table (and file if using Azure Files content), link DNS zones, use identity-based `AzureWebJobsStorage__accountName`, and for Premium/Dedicated set `WEBSITE_CONTENTAZUREFILECONNECTIONSTRING`/`WEBSITE_CONTENTSHARE` with `WEBSITE_CONTENTOVERVNET=1`. Missing any piece causes host startup failures or "Function runtime is unreachable".
**Explanation:** Host uses blob (leases, keys, singleton locks), queue (poison/host queues), table (diagnostics/timers) and file share (content on Windows/Premium). The Function needs roles like Storage Blob Data Owner, Storage Queue Data Contributor, Storage Table Data Contributor. Content share doesn't support managed-identity auth on Premium/Windows in some configs – verify in current docs. Flex Consumption uses blob container deployment storage, no file share.
**Code / CLI / Bicep**
```bicep
appSettings: [
  { name: 'AzureWebJobsStorage__accountName', value: stg.name }
  { name: 'AzureWebJobsStorage__credential',   value: 'managedidentity' }
  { name: 'AzureWebJobsStorage__clientId',     value: uami.properties.clientId } // when UAMI
  { name: 'WEBSITE_CONTENTOVERVNET',           value: '1' }
]
```
**Gotchas / follow-ups**
- Deleting/changing storage firewall during deployment breaks deploy (zip deploy / run-from-package needs access).
- Deployment from a pipeline needs network line-of-sight to SCM (private) – use self-hosted agent in VNet.
- Key Vault references also need the app to reach the vault privately – route-all/VNet DNS.

### 4.6 [Tricky] Function with VNet integration can reach private SQL but not Key Vault references (shows as unresolved). Cause?
**Answer:** Key Vault reference resolution is done by the platform with the app's identity (`keyVaultReferenceIdentity` for UAMI) and requires network reach to the vault; with a private vault and public access off, reach depends on VNet routing/DNS and the vault firewall – and the identity needs Key Vault Secrets User.
**Explanation:** Check app setting status in portal (Source: Key Vault Reference, status). For UAMI you must set `keyVaultReferenceIdentity` on the site; otherwise it tries SAMI. Rotation: unversioned references refresh periodically (up to ~24h; verify).
**Code / CLI / Bicep**
```bash
az functionapp update -g rg-app -n func-orders --set keyVaultReferenceIdentity=<uami-resource-id>
# setting value: @Microsoft.KeyVault(VaultName=kv-prod;SecretName=ApiKey)
```

**Interview traps**
- Thinking VNet integration gives inbound private access.
- Classic Consumption has no VNet integration.
- One subnet can't be shared by different plans' integration (one plan per integration subnet; verify).
- Outbound IPs are not static without NAT Gateway.

**Cheat sheet**
- Inbound: PE / access restrictions; Outbound: VNet integration (+ NAT GW).
- Route-all setting decides non-RFC1918 egress.
- Premium + private triggers -> runtime scale monitoring.
- Host storage needs blob+queue+table (+file) PEs.
- UAMI needs `keyVaultReferenceIdentity`.

---

## Topic 5: Securing Service Bus, Storage, Key Vault, Redis, SQL + Logic Apps Standard

**Study notes**

Layered controls per PaaS service: (1) identity/RBAC (disable keys/SAS/SQL auth), (2) network (firewall + PE, public access off), (3) exceptions ("trusted Azure services"), (4) logging.

| Service | Private Link zone | Firewall/Bypass knob | Notes |
|---|---|---|---|
| Service Bus | `privatelink.servicebus.windows.net` | `networkRuleSets` (`defaultAction`, `trustedServiceAccessEnabled`) | PE + IP/VNet rules need **Premium** (verify) |
| Storage | `privatelink.blob/queue/table/file...` | `networkAcls.defaultAction`, `bypass: AzureServices` | IP rules don't apply to same-region Azure traffic |
| Key Vault | `privatelink.vaultcore.azure.net` | `networkAcls.bypass: AzureServices` | Trusted-service list is limited |
| Redis | `privatelink.redis.cache.windows.net` | `publicNetworkAccess` | Entra auth available; verify for Azure Managed Redis zone |
| SQL | `privatelink.database.windows.net` | server firewall, `publicNetworkAccess`, "Allow Azure services" | Entra-only auth recommended |

```
 Function (VNet-int) -> PE SB / PE Blob / PE KV / PE SQL / PE Redis   (all public access Disabled)
 Logic App Std (VNet-int + inbound PE) -> built-in connectors over VNet
```

### 5.1 [Easy] What does "Allow trusted Microsoft services to bypass this firewall" mean?
**Answer:** It lets a specific list of first-party Azure services that authenticate with managed identity reach the resource even though the firewall denies by default. It is not "all Azure" and it requires public network access to remain enabled with firewall rules.
**Explanation:** For Storage the exception applies to listed resource types/instances (resource instance rules narrow it further). Key Vault supports trusted services such as Azure Backup, Disk Encryption, ARM template deployment (see list). Service Bus trusted-service access applies to e.g. Event Grid, Stream Analytics (verify list). It's very different from SQL's "Allow Azure services and resources to access this server" (0.0.0.0 rule), which allows ANY Azure subscription – avoid.
**Code / CLI / Bicep**
```bicep
resource stg 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: stgName, location: loc, kind: 'StorageV2', sku: { name: 'Standard_LRS' }
  properties: {
    allowSharedKeyAccess: false
    minimumTlsVersion: 'TLS1_2'
    publicNetworkAccess: 'Enabled'
    networkAcls: { defaultAction: 'Deny', bypass: 'AzureServices' }
  }
}
```

### 5.2 [Follow-up] How do you lock down Service Bus fully private, and with which SKU?
**Answer:** Use Premium, create a PE (group `namespace`), link `privatelink.servicebus.windows.net`, disable local (SAS) auth, and set public network access to Disabled. Clients use Entra ID with Data Sender/Receiver roles.
**Explanation:** Standard tier has no private endpoint/VNet rules (verify in current docs). Premium also adds messaging units and geo-DR. Disabling SAS requires all clients on identity-based auth (e.g., `ServiceBusClient(fqdn, DefaultAzureCredential)`; Functions `__fullyQualifiedNamespace`). AMQP over 5671/5672 and HTTPS 443; WebSockets on 443 if only 443 allowed.
**Code / CLI / Bicep**
```bicep
resource sb 'Microsoft.ServiceBus/namespaces@2022-10-01-preview' = {
  name: sbName, location: loc, sku: { name: 'Premium', capacity: 1 }
  properties: { disableLocalAuth: true, publicNetworkAccess: 'Disabled', minimumTlsVersion: '1.2' }
}
```

### 5.3 [Medium] Secure Key Vault: RBAC vs access policies, firewall, purge protection.
**Answer:** Use Azure RBAC authorization (roles like Key Vault Secrets User / Crypto User), enable soft delete + purge protection, restrict network via PE and disabled public access, and log to Log Analytics.
**Explanation:** Access policies are the legacy model; RBAC supports scoping and PIM. Apps use SAMI/UAMI. Purge protection prevents permanent deletion during retention and is irreversible once set. Vault firewall + `bypass: AzureServices` for trusted services only. Diagnostic setting `AuditEvent` for monitoring.
**Code / CLI / Bicep**
```bicep
resource kv 'Microsoft.KeyVault/vaults@2023-07-01' = {
  name: kvName, location: loc
  properties: {
    tenantId: tenant().tenantId
    sku: { family: 'A', name: 'standard' }
    enableRbacAuthorization: true
    enableSoftDelete: true
    enablePurgeProtection: true
    publicNetworkAccess: 'Disabled'
    networkAcls: { defaultAction: 'Deny', bypass: 'None' }
  }
}
```

### 5.4 [Medium] Passwordless SQL and Redis from a Function: how?
**Answer:** SQL: create an Entra user for the managed identity (`CREATE USER [name] FROM EXTERNAL PROVIDER`), grant roles, connect with `Authentication=Active Directory Managed Identity`. Redis: enable Entra authentication on the cache and use the `Microsoft.Azure.StackExchangeRedis` extension.
**Explanation:** The SQL admin must be an Entra principal to create external users. For UAMI add `User Id=<clientId>`. Both go through private endpoints with DNS. Redis access-key auth can be disabled once clients use Entra tokens (verify availability by tier).
**Code / CLI / Bicep**
```sql
CREATE USER [func-orders] FROM EXTERNAL PROVIDER;
ALTER ROLE db_datareader ADD MEMBER [func-orders];
ALTER ROLE db_datawriter ADD MEMBER [func-orders];
```
```csharp
var cs = "Server=tcp:sql-prod.database.windows.net;Database=orders;Authentication=Active Directory Managed Identity;Encrypt=True;";
// UAMI: add "User Id=<uami-client-id>;"
```

### 5.5 [Hard] Logic Apps Standard networking: how do inbound/outbound and connectors behave?
**Answer:** Inbound: private endpoint (and access restrictions) for the workflow triggers/host. Outbound: regional VNet integration; built-in (service provider) connectors like Service Bus/Blob/SQL travel through the VNet, but managed (shared) connectors call out from the multi-tenant connector service and don't use your VNet.
**Explanation:** For a private backend use built-in connectors with identity auth. Logic App Standard also uses a storage account (needs PEs, same as Functions) and shares the App Service plan WS SKUs. Consumption Logic Apps can't use VNet integration; use ISE (retired) → use Standard. Managed connectors to private resources need on-prem data gateway or alternatives.
**Code / CLI / Bicep**
```bicep
resource la 'Microsoft.Web/sites@2023-01-01' = {
  name: laName, location: loc, kind: 'functionapp,workflowapp'
  properties: { serverFarmId: planId, virtualNetworkSubnetId: subnetId, vnetRouteAllEnabled: true, publicNetworkAccess: 'Disabled' }
}
```
**Gotchas / follow-ups**
- Managed connector + private resource = failure that looks like a firewall error.
- Request trigger callback URLs go to the private DNS name; callers must resolve it.
- Storage firewall changes can brick the workflow runtime.

### 5.6 [Tricky] Storage returns 403 AuthorizationFailure from an Azure Function in the same region even though its outbound IP is in the firewall IP rules. Why?
**Answer:** Traffic from Azure services in the same region to Storage uses private Azure addresses, so IP rules on the account don't match. Use a VNet/subnet rule via service endpoint or a private endpoint instead.
**Explanation:** Documented behavior: IP network rules have no effect on requests from the same region as the account. Fix with VNet integration + PE (preferred). Also confirm the account's `networkAcls` bypass and that SAS/key auth isn't forbidden.
**Code / CLI / Bicep**
```bash
az storage account update -g rg-app -n mystorage --public-network-access Disabled
az network private-endpoint create -g rg-app -n pe-stg-blob --vnet-name vnet-spoke-app --subnet snet-pe \
  --private-connection-resource-id <stgId> --group-id blob --connection-name stg-blob
```

**Interview traps**
- "Trusted services" ≠ all Azure; SQL's "Allow Azure services" is much broader.
- Service Bus Standard can't do private endpoints.
- Managed connectors don't use your VNet integration.
- Disabling shared key on host storage breaks apps not yet on identity.

**Cheat sheet**
- Order of defense: identity -> network -> exceptions -> logs.
- SB Premium for PE; set `disableLocalAuth`.
- KV: RBAC + purge protection + PE.
- SQL: Entra-only, `FROM EXTERNAL PROVIDER`, MI auth string.
- Logic Apps Std: PE inbound, VNet-int outbound, built-in connectors.

---

## Topic 6: Application Gateway vs Front Door vs Traffic Manager vs Load Balancer + WAF

**Study notes**

| Service | Layer | Scope | Key use |
|---|---|---|---|
| Load Balancer | L4 TCP/UDP | Regional | Internal/external VM/VMSS balancing, SNAT outbound |
| Application Gateway (v2) | L7 HTTP(S) | Regional | Path routing, TLS offload, WAF, private frontend |
| Front Door (Std/Premium) | L7 HTTP(S) | Global anycast edge | CDN, global failover, WAF, Private Link to origins (Premium) |
| Traffic Manager | DNS | Global | DNS-based routing (priority/weighted/geo/perf), any endpoint, no data path |

```
 Client -> DNS -> Traffic Manager returns closest endpoint IP (no proxy)
 Client -> Front Door edge (WAF) -> [Private Link] -> regional origin (AppGW/App Service)
 Client -> AppGW (WAF) -> backend pool (App Service / VMs)
```
**WAF**: OWASP/Microsoft Default Rule Set + bot rules + custom rules; modes **Detection** vs **Prevention**; policies attach to AppGW listeners or Front Door profiles.

### 6.1 [Easy] When would you use each of these four?
**Answer:** LB for L4 inside a region, App Gateway for regional HTTP routing with WAF, Front Door for global HTTP entry/acceleration/failover, Traffic Manager for DNS-level routing of any protocol or endpoint. They can be combined.
**Explanation:** Typical: Front Door (global, WAF) -> regional App Gateway/App Service. Traffic Manager is DNS only, so failover time depends on TTL and client caching. LB can't see HTTP paths.
**Code / CLI / Bicep**
```bash
az network traffic-manager profile create -g rg-net -n tm-orders --routing-method Priority --unique-dns-name tm-orders-xyz
```

### 6.2 [Follow-up] What does the WAF actually do and how do you roll it out safely?
**Answer:** WAF inspects HTTP requests against managed rule sets (OWASP-based) and custom rules to block SQLi/XSS etc. Roll out in Detection mode, review logs, add exclusions/tune false positives, then switch to Prevention.
**Explanation:** Logs go to Log Analytics (`AzureDiagnostics`/resource-specific tables). Exclusions are scoped (per rule/field). Rate limiting and geo-filter are custom rules. WAF protects L7 only; it doesn't replace API auth.
**Code / CLI / Bicep**
```bicep
resource waf 'Microsoft.Network/ApplicationGatewayWebApplicationFirewallPolicies@2023-11-01' = {
  name: 'waf-appgw', location: loc
  properties: {
    policySettings: { state: 'Enabled', mode: 'Detection', requestBodyCheck: true }
    managedRules: { managedRuleSets: [ { ruleSetType: 'Microsoft_DefaultRuleSet', ruleSetVersion: '2.1' } ] }
  }
}
```

### 6.3 [Medium] How do you ensure traffic to a backend app only comes via Front Door?
**Answer:** Make the origin private: Front Door Premium Private Link to the origin, or restrict the origin by `AzureFrontDoor.Backend` service tag and validate the `X-Azure-FDID` header.
**Explanation:** Service tag alone lets any Front Door profile reach you; the header check ties it to yours. Private Link origins require manual approval of the private endpoint connection on the origin. For App Gateway fronting, restrict to App Gateway subnet.
**Code / CLI / Bicep**
```bash
az functionapp config access-restriction add -g rg-app -n func-orders --rule-name fd --priority 100 \
  --service-tag AzureFrontDoor.Backend --http-header x-azure-fdid=<id>
```

### 6.4 [Medium] App Gateway v2 specifics: subnet, TLS, health probes.
**Answer:** v2 needs a dedicated subnet, supports autoscale and zone redundancy, TLS termination or end-to-end TLS with certs from Key Vault via a user-assigned identity, and custom health probes. Backends not responding to probe get marked unhealthy -> 502.
**Explanation:** NSG on AppGW subnet must allow `GatewayManager` on 65200-65535 (v2) and AzureLoadBalancer. Host header override matters for App Service backends (use "pick host name from backend"). Backend cert chain needs trusted root for end-to-end TLS. v1 is retiring – verify date.
**Code / CLI / Bicep**
```bash
az network application-gateway show-backend-health -g rg-net -n agw-orders
```

### 6.5 [Hard] 502 Bad Gateway from App Gateway to an App Service backend – walk the diagnosis.
**Answer:** Check backend health and probe (path, host, status match), host-header handling, backend TLS certificate/SNI, access restrictions on the app (AppGW subnet allowed), NSG/UDR on AppGW subnet, and DNS from the gateway subnet for the backend FQDN.
**Explanation:** Most common: probe hits `/` which returns 404/redirect/401; host header sent is the AppGW's public name, so the App Service rejects or redirects; or backend restricts access to the AppGW subnet but service endpoint missing on AppGW subnet.
**Code / CLI / Bicep**
```kusto
AzureDiagnostics
| where ResourceType == "APPLICATIONGATEWAYS" and Category == "ApplicationGatewayAccessLog"
| where httpStatus_d >= 500
| project TimeGenerated, requestUri_s, httpStatus_d, serverRouted_s, serverStatus_s
```
**Gotchas / follow-ups**
- Timeout default (~20s request timeout) cuts long-running requests – verify setting.
- Redirect loops from App Service HTTPS-only with AppGW HTTP backend.
- UDR 0.0.0.0/0 on the AppGW subnet breaks v2 (not supported); verify.

### 6.6 [Tricky] Traffic Manager failover is "slow". Why, and what are alternatives?
**Answer:** It's DNS-based: clients and resolvers cache answers for the TTL and endpoint health probes run on an interval with tolerated failures, so total failover is probe interval x failures + TTL. Front Door gives faster failover since it proxies traffic.
**Explanation:** Lower TTL (e.g., 10–60s) and tune probe interval/tolerated failures. TM also cannot hide origin IPs and can't terminate TLS. For non-HTTP endpoints TM is still apt.
**Code / CLI / Bicep**
```bash
az network traffic-manager profile update -g rg-net -n tm-orders --ttl 30 --monitor-interval 10 --timeout 5 --max-failures 3
```

**Interview traps**
- Traffic Manager is not a proxy/load balancer.
- WAF in Detection mode blocks nothing.
- Service tag alone doesn't prove the request came from *your* Front Door.
- App Gateway v2 dedicated subnet; no UDR default route to firewall.

**Cheat sheet**
- LB L4 regional; AppGW L7 regional; FD L7 global; TM DNS global.
- WAF: Detection -> tune -> Prevention.
- FD Premium + Private Link for private origins.
- 502 = backend health/host header/TLS.
- TM failover = probes + TTL.

---

## Topic 7: API Management (Policies, Tiers, Identity, VNet)

**Study notes**

- APIM = gateway + management plane + developer portal. Policy pipeline: `inbound` -> `backend` -> `outbound` -> `on-error`, scoped global/product/API/operation.
- Key policies: `validate-jwt`, `rate-limit` / `rate-limit-by-key`, `quota`, `rewrite-uri`, `set-header`, `cache-lookup`/`cache-store`, `validate-client-certificate` (mTLS), `authentication-managed-identity`, `set-backend-service`, `retry`, `send-request`.
- Tiers: Consumption, Developer, Basic, Standard, Premium, and v2 tiers (Basic v2, Standard v2, Premium v2) – features (VNet, multi-region, SLA) differ; verify in current docs.
- VNet: classic Developer/Premium support *external* (public gateway, VNet-reachable backends) or *internal* (private only) injection; v2 tiers use different models (outbound integration / injection) – verify.

```
 Client --JWT--> [Front Door/AppGW WAF] -> APIM (inbound: validate-jwt, rate-limit)
                                           -> authentication-managed-identity -> Backend Function (Entra-protected)
```

### 7.1 [Easy] What is APIM and what problems does it solve?
**Answer:** APIM is a managed API gateway that fronts backends with a consistent façade: authentication, throttling, transformation, caching, versioning, and analytics without changing backend code.
**Explanation:** Policies are XML evaluated per request. Products group APIs with subscription keys. Use named values (backed by Key Vault) for config.
**Code / CLI / Bicep**
```xml
<policies>
  <inbound>
    <base />
    <rate-limit-by-key calls="100" renewal-period="60" counter-key="@(context.Request.IpAddress)" />
  </inbound>
</policies>
```

### 7.2 [Follow-up] Validate an Entra ID token at the gateway.
**Answer:** Use `validate-jwt` with the OpenID config URL, required audience and issuer, and required claims/roles; return 401 on failure.
**Explanation:** Offloads authN from backends, but backends should still validate (defense in depth). Use `v2.0` metadata for v2 tokens and match `aud` to your App ID URI/client ID.
**Code / CLI / Bicep**
```xml
<validate-jwt header-name="Authorization" failed-validation-httpcode="401" failed-validation-error-message="Unauthorized">
  <openid-config url="https://login.microsoftonline.com/{tenantId}/v2.0/.well-known/openid-configuration" />
  <audiences><audience>api://orders-api</audience></audiences>
  <required-claims><claim name="roles" match="any"><value>Orders.Read</value></claim></required-claims>
</validate-jwt>
```

### 7.3 [Medium] How does APIM authenticate to a backend using managed identity?
**Answer:** The `authentication-managed-identity` policy gets a token for a target resource/audience using the APIM system- or user-assigned identity and attaches it as a bearer token to the backend call.
**Explanation:** Grant that identity an app role or RBAC on the backend (e.g., the Function's Entra app registration role). For UAMI specify `client-id`. Set `ignore-error` as needed, and store token in a variable with `output-token-variable-name` if reused.
**Code / CLI / Bicep**
```xml
<authentication-managed-identity resource="api://orders-backend" />
```

### 7.4 [Medium] Caching, rewrite and mTLS policies in practice?
**Answer:** `cache-lookup`/`cache-store` use the built-in or external Redis cache keyed by vary-by settings; `rewrite-uri` maps public URLs to backend paths; `validate-client-certificate` enforces mTLS on the client cert (needs client-cert negotiation enabled on the gateway/hostname).
**Explanation:** External Azure Cache for Redis is recommended for production (built-in cache is per-instance/limited; verify per tier). Don't cache per-user data without `vary-by-header Authorization`. Consumption tier requires client-cert enabling at the service level.
**Code / CLI / Bicep**
```xml
<inbound>
  <cache-lookup vary-by-developer="false" vary-by-developer-groups="false" downstream-caching-type="none">
    <vary-by-header>Authorization</vary-by-header>
  </cache-lookup>
  <rewrite-uri template="/api/v2/orders/{id}" />
</inbound>
<outbound><cache-store duration="60" /></outbound>
```

### 7.5 [Hard] Put APIM in a VNet to front private Functions. What are the networking requirements?
**Answer:** Use Developer/Premium (or a v2 model, verify) injected in a dedicated subnet in internal or external mode, with NSG rules allowing APIM management (`ApiManagement` service tag, port 3443), LB probes, and outbound dependencies (Storage, SQL, Key Vault, Monitor); resolve backends through private DNS and front with App Gateway/Front Door if public.
**Explanation:** Internal mode gives no public gateway – you manage DNS for the APIM hostnames. Missing management connectivity makes the instance unhealthy and unmanageable. UDR to firewall requires correct dependencies. Verify current port/tag requirements for your tier/stv2 platform.
**Code / CLI / Bicep**
```bicep
resource apim 'Microsoft.ApiManagement/service@2023-05-01-preview' = {
  name: apimName, location: loc
  sku: { name: 'Premium', capacity: 1 }
  identity: { type: 'SystemAssigned' }
  properties: {
    publisherEmail: 'admin@contoso.com', publisherName: 'Contoso'
    virtualNetworkType: 'Internal'
    virtualNetworkConfiguration: { subnetResourceId: apimSubnetId }
  }
}
```
**Gotchas / follow-ups**
- Internal mode + no private DNS = "works on the portal, nothing resolves".
- Changing VNet mode can take long and may cause downtime.
- Always protect backends from direct access (IP restriction/JWT audience), else subscription keys are bypassed.

### 7.6 [Tricky] `rate-limit` vs `rate-limit-by-key`, and why does rate limiting seem inaccurate across instances/regions?
**Answer:** `rate-limit` is per subscription; `rate-limit-by-key` is per arbitrary key (IP, claim). Counters are synchronized across gateway instances eventually, so limits are approximate, especially across regions in multi-region Premium.
**Explanation:** Use `quota` for long-period caps. Behind a proxy, `context.Request.IpAddress` is the proxy's – use `X-Forwarded-For` carefully. Consumption tier behaves differently.
**Code / CLI / Bicep**
```xml
<rate-limit-by-key calls="20" renewal-period="60" counter-key="@(context.Request.Headers.GetValueOrDefault("X-Forwarded-For","").Split(',')[0].Trim())" />
```

**Interview traps**
- Subscription key is not authentication of a user.
- Backends reachable directly bypass every APIM policy.
- Built-in cache != production cache.
- APIM in VNet needs management traffic allowed.

**Cheat sheet**
- Pipeline: inbound/backend/outbound/on-error.
- validate-jwt + authentication-managed-identity = zero secret.
- rate-limit-by-key for per-caller throttling.
- Tier picks VNet & SLA features – verify.
- Named values from Key Vault.

---

## Topic 8: Azure Firewall, NSG/ASG, DDoS Protection

**Study notes**

- **NSG**: free, stateful L3/L4 at subnet/NIC. **ASG**: logical grouping used as source/dest in NSG rules.
- **Azure Firewall**: managed, stateful, central; network rules, application rules (FQDN), NAT rules, threat intelligence, FQDN tags. SKUs: Basic, Standard, Premium (TLS inspection, IDPS, URL filtering). Needs `AzureFirewallSubnet` (/26). Use **Firewall Policy** for rule collections.
- **DDoS**: infrastructure protection is automatic; **DDoS Network Protection** (per VNet plan) / **DDoS IP Protection** (per public IP) add adaptive tuning, telemetry, mitigation reports, rapid response (SKU features: verify in current docs). It protects L3/L4; use WAF for L7.

```
 Spoke subnets --UDR 0.0.0.0/0--> Azure Firewall (hub) --> Internet / on-prem
 Public IP on AppGW/FW  <-- DDoS Protection plan (VNet-level)
```

### 8.1 [Easy] NSG vs Azure Firewall?
**Answer:** NSG is a free distributed packet filter on subnets/NICs using IPs, ports and tags; Azure Firewall is a centralized managed firewall that adds FQDN filtering, threat intelligence, NAT, TLS inspection (Premium) and central logging. Use both: NSG for micro-segmentation, firewall for perimeter/egress.
**Explanation:** NSGs can't filter by FQDN. Firewall rules process NAT -> network -> application order by priority/collection group.
**Code / CLI / Bicep**
```bash
az network firewall policy create -g rg-hub -n fwp-hub --sku Standard
```

### 8.2 [Follow-up] What are ASGs for?
**Answer:** ASGs let you write NSG rules against roles (e.g., `asg-web`, `asg-db`) instead of IP addresses, so rules survive IP changes and scale-out.
**Explanation:** NICs join ASGs; both source and destination ASGs must be in the same VNet scope. Cleaner than CIDR lists.
**Code / CLI / Bicep**
```bash
az network asg create -g rg-net -n asg-web
az network nsg rule create -g rg-net --nsg-name nsg-app -n web-to-db --priority 200 --direction Inbound --access Allow \
  --protocol Tcp --source-asgs asg-web --destination-asgs asg-db --destination-port-ranges 1433
```

### 8.3 [Medium] Allow a Function's outbound call to `*.servicebus.windows.net` through the firewall – rule type?
**Answer:** An application rule with FQDN target (or the FQDN/service tag in a network rule) on the proper ports; preferred is a private endpoint so egress doesn't traverse the firewall at all.
**Explanation:** AMQP on 5671 isn't HTTP/S, so an application rule only handles HTTP/HTTPS/MSSQL; for AMQP use a network rule with FQDN (needs DNS proxy enabled) or service tags. Enable firewall DNS proxy for FQDN in network rules.
**Code / CLI / Bicep**
```bash
az network firewall network-rule create -g rg-hub -f fw-hub --collection-name net-allow -n sb-amqp \
  --priority 200 --action Allow --protocols TCP --source-addresses 10.1.1.0/24 --destination-fqdns sb-prod.servicebus.windows.net --destination-ports 5671
```

### 8.4 [Medium] DDoS Network Protection vs IP Protection vs basic.
**Answer:** Basic infrastructure protection is on for all Azure public IPs with platform-level thresholds; Network Protection (plan on VNets) and IP Protection (per IP) add workload-tuned mitigation, metrics/alerts, and for Network Protection cost guarantee and rapid response, with IP Protection being cheaper without some features. Verify feature matrix.
**Explanation:** Protection applies to public IPs in protected VNet. Pair with WAF and autoscale. Configure alerts on `IfUnderDDoSAttack` metric.
**Code / CLI / Bicep**
```bicep
resource plan 'Microsoft.Network/ddosProtectionPlans@2023-11-01' = { name: 'ddos-plan', location: loc }
// vnet properties: enableDdosProtection: true, ddosProtectionPlan: { id: plan.id }
```

### 8.5 [Hard] Why can Azure Firewall break a working app after you add a default UDR, and how do you fix asymmetry?
**Answer:** The default route sends all outbound via the firewall, which may block unlisted dependencies; and inbound flows via public IPs arriving directly (e.g., AppGW/Public LB) return through the firewall, creating asymmetric routing and dropped sessions. Add allow rules and use SNAT/DNAT correctly or route-specific UDRs.
**Explanation:** For public inbound to VMs with a public IP while default route to FW, replies take the firewall path with a different public IP → drop. Fix: DNAT through the firewall, or add UDR for the Internet source ranges back to Internet. Platform dependencies (App Service plan, APIM) have documented required FQDNs.
**Code / CLI / Bicep**
```kusto
AZFWNetworkRule | where Action == "Deny" | summarize count() by SourceIp, DestinationIp, DestinationPort | top 20 by count_
```
**Gotchas / follow-ups**
- Firewall SNAT port limits per public IP – add more IPs or NAT Gateway on the firewall subnet.
- Force tunneling needs a management subnet (`AzureFirewallManagementSubnet`).
- TLS inspection needs your CA trusted by clients.

### 8.6 [Tricky] An allow rule exists in NSG but traffic is still dropped – list places to look.
**Answer:** Check the NIC-level NSG (both layers), UDR next hop/firewall deny, Azure Firewall logs, the OS firewall, PE network policies, and service-level firewalls. Use IP flow verify.
**Explanation:** `az network watcher test-ip-flow` tells which NSG rule matched; it doesn't cover firewall/OS.
**Code / CLI / Bicep**
```bash
az network watcher test-ip-flow -g rg-net --vm vm1 --direction Outbound --protocol TCP \
  --local 10.1.1.4:50000 --remote 10.1.2.5:443
```

**Interview traps**
- FQDN filtering isn't an NSG feature.
- AMQP isn't HTTP → app rules don't match.
- DDoS protection ≠ WAF.
- Asymmetric routing with default UDR.

**Cheat sheet**
- NSG micro-segmentation; Firewall central egress/inspection.
- ASGs for role-based rules.
- FW Premium = TLS inspection + IDPS.
- DDoS = L3/L4; WAF = L7.
- Prefer PE over routing PaaS traffic through firewall.

---

## Topic 9: TLS/Certificates, Secrets & Zero-Secret Architecture

**Study notes**

- **TLS**: enforce min TLS 1.2 on Storage, Service Bus, SQL, App Service (`minTlsVersion`), HTTPS-only. Certificates: store in **Key Vault** (create/import, auto-renew with integrated CAs like DigiCert/GlobalSign), App Service Managed Certificates (free, for custom domains, no wildcard/private-DNS-only), App Gateway/Front Door pull certs from Key Vault with identity.
- **Zero-secret** = no connection strings/keys in code or config; use **managed identity + RBAC**.
  - **SAMI**: lifecycle tied to resource, 1:1. **UAMI**: independent resource, reusable, pre-assignable (good for IaC ordering and sharing across Function slots/apps).
  - Where a secret is unavoidable (third-party API key): Key Vault + Key Vault reference.
- Token flow: app -> IMDS/identity endpoint -> Entra token for audience -> data-plane call with RBAC.

```
 Function (UAMI) --token(aud=https://servicebus.azure.net)--> Service Bus  (Data Receiver role)
                 --token(aud=https://storage.azure.com)-----> Blob        (Blob Data Contributor)
                 --token(aud=https://database.windows.net)--> SQL         (CREATE USER FROM EXTERNAL PROVIDER)
                 --Key Vault ref--> third-party secret (Secrets User)
```

### 9.1 [Easy] SAMI vs UAMI?
**Answer:** SAMI is created with and deleted with the resource, tied to one resource. UAMI is a standalone resource you assign to multiple resources and that survives their deletion.
**Explanation:** Use UAMI when RBAC must exist before the app deploys (avoids chicken-and-egg), when many apps share permissions, or for blue/green slots. SAMI is simpler for single-resource cases. Limit: assigning many identities has cap limits – verify.
**Code / CLI / Bicep**
```bicep
resource uami 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = { name: 'id-orders', location: loc }
resource role 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(sb.id, uami.id, 'receiver')
  scope: sb
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '4f6d3b9b-027b-4f4c-9142-0e5a2a2247e0') // Azure Service Bus Data Receiver
    principalId: uami.properties.principalId
    principalType: 'ServicePrincipal'
  }
}
```

### 9.2 [Follow-up] How does a .NET app use managed identity without secrets?
**Answer:** Use `DefaultAzureCredential` (or `ManagedIdentityCredential` explicitly with the UAMI client ID); SDK clients accept the credential and the platform provides tokens.
**Explanation:** Locally `DefaultAzureCredential` falls back to Visual Studio/CLI. In production, prefer a specific credential to avoid slow probing chains and unexpected identities. Functions bindings use `Connection__fullyQualifiedNamespace` and `__credential`/`__clientId` settings.
**Code / CLI / Bicep**
```csharp
var cred = new ManagedIdentityCredential(ManagedIdentityId.FromUserAssignedClientId(clientId));
var sb   = new ServiceBusClient("sb-prod.servicebus.windows.net", cred);
var blob = new BlobServiceClient(new Uri("https://stg.blob.core.windows.net"), cred);
```

### 9.3 [Medium] Certificate lifecycle with Key Vault: auto-rotation and consumption.
**Answer:** Create a certificate with an issuance policy (CA, validity, renewal at % lifetime or days before expiry); consumers reference the unversioned secret ID to pick up new versions. Alert on near-expiry with Event Grid `CertificateNearExpiry`.
**Explanation:** App Service/App Gateway sync Key Vault certs periodically (App Service ~24h; verify). Non-integrated CAs need manual merge. Event Grid events trigger Functions/Logic Apps for automation.
**Code / CLI / Bicep**
```bash
az keyvault certificate create --vault-name kv-prod -n api-cert -p "$(az keyvault certificate get-default-policy)"
az keyvault certificate show --vault-name kv-prod -n api-cert --query "{exp:attributes.expires,sid:sid}"
```

### 9.4 [Medium] Key Vault references vs reading Key Vault in code?
**Answer:** References (`@Microsoft.KeyVault(...)`) let the platform inject secrets as app settings with no code changes but updates need refresh/restart semantics; reading in code via SDK gives control and caching but adds a dependency and token handling.
**Explanation:** References unversioned auto-update (within about a day; verify), versioned don't. Failures leave the literal reference string in the setting. Use for third-party secrets only; for Azure PaaS use identity directly.
**Code / CLI / Bicep**
```bicep
{ name: 'Partner__ApiKey', value: '@Microsoft.KeyVault(SecretUri=https://kv-prod.vault.azure.net/secrets/PartnerKey/)' }
```

### 9.5 [Hard] Design a rotation story and eliminate long-lived secrets in a pipeline too.
**Answer:** Replace service principal secrets with workload identity federation (OIDC) for GitHub Actions/Azure DevOps; apps use managed identities; remaining secrets live in Key Vault with rotation (Event Grid + Function) and short TTL; disable key/SAS auth on resources.
**Explanation:** Federated credentials trust the CI issuer token for a subject, so no secret exists. Disable `allowSharedKeyAccess`, `disableLocalAuth`, SQL SQL-auth. Audit with Azure Policy and Defender.
**Code / CLI / Bicep**
```bash
az ad app federated-credential create --id <appObjectId> --parameters '{"name":"gh-main","issuer":"https://token.actions.githubusercontent.com","subject":"repo:contoso/orders:ref:refs/heads/main","audiences":["api://AzureADTokenExchange"]}'
```
**Gotchas / follow-ups**
- RBAC propagation can take minutes -> first-call 403 after assignment.
- Token caching: identities' group membership/role changes show up after token refresh (long caches).
- Don't use `DefaultAzureCredential` with `EnvironmentCredential` leftovers in prod.

### 9.6 [Tricky] UAMI-based Function: app works locally but gets "ManagedIdentityCredential authentication unavailable" or picks the wrong identity in Azure. Causes?
**Answer:** With multiple identities, the SDK needs the UAMI's client ID (`AZURE_CLIENT_ID` or explicit); identity isn't assigned to the app; or for Functions binding connections `__credential=managedidentity` and `__clientId` are missing.
**Explanation:** With one SAMI it just works; with UAMI (and especially SAMI + UAMI both) ambiguity arises. Check the app's identity blade and role assignment scope/principalId.
**Code / CLI / Bicep**
```bash
az functionapp identity show -g rg-app -n func-orders
az role assignment list --assignee <principalId> --all -o table
```

**Interview traps**
- Managed identity doesn't remove the need for RBAC and network access.
- Role assignment lag.
- SAMI deletes with the app (and its role assignments become orphaned).
- Managed certs don't cover wildcards or private-only sites.

**Cheat sheet**
- Identity -> RBAC -> network; all three.
- UAMI for shared/pre-provisioned; SAMI for simple.
- KV: cert auto-renew + Event Grid alerts.
- OIDC federation for CI/CD.
- Disable shared key/SAS/local auth once migrated.

---

## Topic 10: Defender for Cloud, Sentinel, Azure Policy Guardrails

**Study notes**

- **Defender for Cloud**: **CSPM** (posture: Foundational CSPM free – recommendations, **secure score**; Defender CSPM paid – attack path analysis, agentless scanning, governance) and **CWPP** (workload protection plans: Servers, Storage, App Service, Key Vault, SQL, Containers, APIs…). Verify plan names/pricing.
- **Microsoft Sentinel**: cloud SIEM/SOAR on a Log Analytics workspace – data connectors, analytics rules (KQL), incidents, workbooks, playbooks (Logic Apps), UEBA, threat hunting.
- **Azure Policy**: evaluates resource properties; effects `Audit`, `Deny`, `Modify`, `DeployIfNotExists`, `AuditIfNotExists`, `Disabled` (`Append` legacy). Group into **initiatives**; assign at management group/subscription/RG; remediation tasks use managed identity.

```
 Policy (Deny public access / require PE DNS) --prevent--> Resources --posture--> Defender for Cloud (secure score)
 Resource logs/Defender alerts --> Log Analytics --> Sentinel analytics rules --> incidents --> Logic App playbooks
```

### 10.1 [Easy] What is secure score?
**Answer:** A percentage in Defender for Cloud reflecting how many security recommendations you've remediated, weighted by risk. It's a posture metric across subscriptions.
**Explanation:** Based on built-in recommendations from the Microsoft Cloud Security Benchmark initiative. Exempt/ignore items for accepted risk with justification.
**Code / CLI / Bicep**
```bash
az security secure-scores list -o table
```

### 10.2 [Follow-up] CSPM vs CWPP?
**Answer:** CSPM finds misconfigurations and measures posture (are Storage accounts public?); CWPP detects threats and protects specific workloads at runtime (malware on VMs, SQL injection alerts, anomalous Key Vault access).
**Explanation:** Enable per-plan at subscription level. Some plans (Storage malware scanning, Key Vault) add alerts into the same Security Alerts view and can stream to Sentinel.
**Code / CLI / Bicep**
```bash
az security pricing create -n KeyVaults --tier Standard
az security pricing create -n StorageAccounts --tier Standard
```

### 10.3 [Medium] Use Azure Policy to enforce "no public network access" and "PE DNS auto-config".
**Answer:** Assign built-in policies with `Deny` (or `Audit` first) for services' public network access, and `DeployIfNotExists` policies to create DNS zone group entries for private endpoints. Remediate existing resources via remediation tasks.
**Explanation:** Roll out Audit -> Deny, use exemptions for exceptions, and use `Modify` for tags. DINE needs a managed identity with sufficient role. Policy evaluation can lag (~15 min on change, 24h full scan).
**Code / CLI / Bicep**
```bash
az policy assignment create -n deny-pubnet --policy <policyDefinitionId> --scope /subscriptions/<sub> \
  --params '{"effect":{"value":"Deny"}}'
```
```json
{ "if": { "allOf": [ {"field":"type","equals":"Microsoft.Storage/storageAccounts"},
                      {"field":"Microsoft.Storage/storageAccounts/publicNetworkAccess","notEquals":"Disabled"} ] },
  "then": { "effect": "deny" } }
```

### 10.4 [Medium] Sentinel basics: how does an alert become an automated response?
**Answer:** Connectors ingest logs into the workspace; scheduled/NRT analytics rules (KQL) create alerts/incidents; automation rules and Logic App playbooks respond (e.g., disable user, post to Teams).
**Explanation:** Cost is by ingestion; use basic logs/data lake tier for high volume (verify). Entra sign-in and Azure Activity logs are core. Sentinel is moving into the Defender portal – verify current experience.
**Code / CLI / Bicep**
```kusto
SigninLogs
| where ResultType == 0 and RiskLevelDuringSignIn in ("high","medium")
| summarize count() by UserPrincipalName, IPAddress
```

### 10.5 [Hard] Policy effects: when does `Deny` fail you and how do you handle brownfield/IaC conflicts?
**Answer:** Deny blocks non-compliant creates/updates (including deployments that set a property later), which can break pipelines; use Audit first, `Modify`/DINE for auto-fix, exemptions with expiry, and make Bicep templates compliant by default.
**Explanation:** Existing noncompliant resources aren't deleted by Deny—only flagged. A two-step deployment (create then lock down) can be blocked at the first step. Use policy with `DoNotEnforce` enforcement mode to test. Test with policy aliases; not all properties have aliases.
**Code / CLI / Bicep**
```bash
az policy state list --resource-group rg-app --filter "complianceState eq 'NonCompliant'" -o table
az policy remediation create -n fix-dns --policy-assignment <assignmentId>
```
**Gotchas / follow-ups**
- DINE remediation identity lacks role -> "remediation failed".
- Policy doesn't affect data-plane.
- Conflicting policies at different scopes: Deny wins.

**Interview traps**
- Defender is detection/posture; Policy is prevention/governance.
- Secure score ≠ compliance certification.
- Sentinel needs a Log Analytics workspace and costs by ingestion.
- Enabling Deny blindly breaks deployments.

**Cheat sheet**
- Foundational CSPM free; Defender CSPM/CWPP paid.
- Policy effects: Deny/Audit/Modify/DINE/AINE.
- Initiative = policy set; assign at MG scope.
- Sentinel = SIEM+SOAR on Log Analytics.
- Always Audit first.

---

## Topic 11: Azure Bastion & the Network Troubleshooting Toolkit

**Study notes**

- **Bastion**: managed RDP/SSH over TLS (443) from the portal/native client to VMs by private IP; no public IPs on VMs. Needs subnet named `AzureBastionSubnet` (/26 or larger). SKUs: Developer, Basic, Standard, Premium (features like native client, shareable links, IP-based connection differ – verify). Alternative: jump host VM (you patch/secure it) or Just-in-Time access.
- **Toolkit**: `nslookup`/`Resolve-DnsName` (DNS), `Test-NetConnection -ComputerName x -Port 443` (TCP), `tcpping` and `nameresolver` in **Kudu console** (Windows App Service), `curl -v`, Network Watcher (**IP flow verify**, **Next hop**, **Connection troubleshoot / Connection Monitor**, **packet capture**, **effective security rules**, **NSG/VNet flow logs**), **effective routes**.

```
 Symptom -> 1 DNS? (nslookup from workload) -> 2 Route? (effective routes/next hop)
         -> 3 NSG/Firewall? (IP flow verify / FW logs) -> 4 Service firewall/PE state -> 5 Auth (RBAC/token)
```

### 11.1 [Easy] What is Azure Bastion and why use it over a jump box?
**Answer:** A PaaS service that gives secure browser/native-client RDP/SSH into VMs over TLS without exposing public IPs or opening 3389/22 to the internet. Unlike a jump VM, Microsoft patches and manages it.
**Explanation:** Deploy per VNet (or peered VNets with Standard+). Combine with NSGs allowing Bastion subnet to VM ports. Session recording/Premium options exist – verify.
**Code / CLI / Bicep**
```bash
az network bastion create -g rg-hub -n bas-hub --vnet-name vnet-hub --public-ip-address pip-bas --sku Standard
```

### 11.2 [Follow-up] How do you test connectivity to a private endpoint from an App Service/Function?
**Answer:** Open the Kudu (SCM) console/Advanced Tools and run `nameresolver <fqdn>` to check DNS and `tcpping <fqdn>:443` to check the TCP port; `curl` for Linux apps via SSH console.
**Explanation:** `tcpping`/`nameresolver` are available on Windows plans; Linux containers may need `curl`/`nc` installed or use the app's SSH. Kudu itself must be reachable (SCM access restrictions/private endpoint) – from a pipeline/laptop outside the VNet it won't open when public is blocked.
**Code / CLI / Bicep**
```text
# Kudu > Debug console > CMD
nameresolver mystorage.blob.core.windows.net
tcpping mystorage.blob.core.windows.net:443
```

### 11.3 [Medium] Use Network Watcher to prove whether NSG or routing is the culprit.
**Answer:** IP flow verify tells which NSG rule allows/denies a 5-tuple for a VM NIC; Next hop shows where packets go; effective routes/security rules show the merged view; Connection troubleshoot/Connection Monitor tests end-to-end paths.
**Explanation:** These tools only see VMs/NICs (not PaaS multi-tenant instances), so for Functions test from a VM in the same subnet/VNet or use Kudu. Check Network Watcher is enabled in the region. Connection troubleshoot features are evolving (Connection Monitor preferred) – verify.
**Code / CLI / Bicep**
```bash
az network nic show-effective-route-table -g rg-net -n vm1-nic -o table
az network nic list-effective-nsg -g rg-net -n vm1-nic
az network watcher test-ip-flow -g rg-net --vm vm1 --direction Outbound --protocol TCP --local 10.1.1.4:50000 --remote 10.1.2.5:1433
az network watcher show-next-hop -g rg-net --vm vm1 --source-ip 10.1.1.4 --dest-ip 10.1.2.5
```

### 11.4 [Medium] NSG flow logs: what do they give you and what's changing?
**Answer:** They record allowed/denied flows per NSG rule with 5-tuple, bytes and direction to a storage account, and can feed Traffic Analytics. Microsoft is moving customers to VNet flow logs; NSG flow logs are on a retirement path – verify dates in current docs.
**Explanation:** Use for forensics ("who talked to the DB at 02:00"), and to find denied traffic. Retention to storage; analytics via Log Analytics. VNet flow logs cover all VNet traffic, including some cases NSG logs missed (e.g., NSGs not attached).
**Code / CLI / Bicep**
```bash
az network watcher flow-log create -g rg-net -n fl-nsg-app --nsg nsg-app --storage-account stgflowlogs --enabled true --retention 30 --location westeurope
```

### 11.5 [Hard] Systematic triage: "Function can't connect to SQL via private endpoint (timeout)".
**Answer:** Confirm VNet integration subnet & route-all; resolve FQDN to PE private IP from Kudu; test TCP 1433; check NSG/UDR/firewall on integration subnet and PE subnet (network policies); check SQL PE connection approved and public access/firewall; check connection policy (Proxy/Redirect) and `Encrypt`; then auth.
**Explanation:** Timeout (not 40615/18456 error) means network. A "login failed" means network is fine. Via private endpoint, SQL connections go through the PE using proxy behavior on 1433; with Redirect from other paths, ports 11000-11999 are required (verify). Also check pool exhaustion vs. real timeouts (`Connect Timeout`).
**Code / CLI / Bicep**
```powershell
Resolve-DnsName sql-prod.database.windows.net
Test-NetConnection sql-prod.database.windows.net -Port 1433
```
```bash
az sql server show -g rg-data -n sql-prod --query "{pub:publicNetworkAccess,pe:privateEndpointConnections[].properties.privateLinkServiceConnectionState.status}"
```
**Gotchas / follow-ups**
- `Test-NetConnection` succeeding doesn't prove DNS from the Function (different resolver).
- Serverless SQL auto-pause causes first-connection timeouts/retries – enable retry logic.
- App-level SNAT or pool exhaustion mimics network timeouts.

### 11.6 [Tricky] `tcpping` succeeds, the app still fails TLS. What else?
**Answer:** TCP success only proves port reachability. Failures may be TLS version/cert-name mismatch (connecting to the private IP instead of FQDN), a TLS-inspecting firewall's untrusted CA, SNI, or auth.
**Explanation:** Always connect by FQDN, not the PE IP – certificates and SNI use the service name. With Azure Firewall Premium TLS inspection, the app runtime must trust the intermediate CA.
**Code / CLI / Bicep**
```bash
curl -v https://mystorage.blob.core.windows.net/?comp=list    # inspect cert, SNI, handshake
```

**Interview traps**
- Network Watcher cannot inspect PaaS internals.
- Testing from your laptop ≠ testing from the workload.
- Bastion subnet name is fixed (`AzureBastionSubnet`).
- Timeout ≠ forbidden: timeouts are network, 403/401 are policy/identity.

**Cheat sheet**
- Order: DNS -> route -> NSG/FW -> service firewall -> auth.
- Kudu: `nameresolver`, `tcpping` (Windows).
- Effective routes + IP flow verify + Next hop.
- Flow logs for history (check retirement/VNet flow logs).
- Bastion: /26 `AzureBastionSubnet`.

---

## Topic 12: Hub-and-Spoke Reference Design

**Study notes**

```
                       On-prem --VPN/ExpressRoute--> [Hub VNet 10.0.0.0/16]
                                                     - GatewaySubnet (VPN/ER GW)
                                                     - AzureFirewallSubnet (Firewall Premium)
                                                     - AzureBastionSubnet
                                                     - snet-dns (DNS Private Resolver)
                                                     - Private DNS zones linked (privatelink.*)
                       peer (gateway transit, forwarded traffic) /          \
          [Spoke-Prod 10.1.0.0/16]                                       [Spoke-Shared 10.2.0.0/16]
          snet-func (delegated) snet-pe snet-apim                        agents, tooling
          UDR 0.0.0.0/0 -> FW ; NSG per subnet
```
- Hub = shared connectivity/security services; spokes = workloads isolated by subscription/VNet. Alternatives: **Azure Virtual WAN** (managed hub) and **Azure Virtual Network Manager** (connectivity/security admin rules at scale). Landing zones (ALZ) formalize management-group hierarchy and policy.
- Decisions: central vs per-spoke private DNS zones; central vs distributed private endpoints; egress via firewall vs NAT Gateway; shared Bastion; IP plan (non-overlapping, room to grow).

### 12.1 [Easy] Why hub-and-spoke?
**Answer:** It centralizes shared services (firewall, gateway, DNS, Bastion) in a hub and isolates workloads in spokes, reducing cost and giving one inspection/egress point and clearer blast-radius boundaries.
**Explanation:** Spokes peer to the hub only. Environments (prod/non-prod) get separate spokes/subscriptions.
**Code / CLI / Bicep**
```bash
az network vnet peering create -g rg-hub -n hub-to-prod --vnet-name vnet-hub --remote-vnet <prodVnetId> --allow-vnet-access --allow-forwarded-traffic --allow-gateway-transit
```

### 12.2 [Follow-up] Hub VNet vs Virtual WAN?
**Answer:** A self-managed hub VNet gives full control (any NVA, custom DNS resolver, UDR) while Virtual WAN is a Microsoft-managed hub with automated routing, scale for branches/SD-WAN, and built-in transitivity. Choose vWAN for large multi-branch/multi-region; hub VNet for customization.
**Explanation:** vWAN hubs have routing intent and secured hub (Firewall Manager). Some constraints (DNS resolver placement, custom NVAs) differ – verify.
**Code / CLI / Bicep**
```bash
az network vwan create -g rg-hub -n vwan-corp --type Standard
```

### 12.3 [Medium] Where do you put private endpoints and zones in a hub-spoke landing zone?
**Answer:** Private DNS zones centrally in a connectivity subscription linked to the hub (and spokes or resolver); PEs live in the workload spoke next to the consumers (common) and register into the central zone via policy.
**Explanation:** Central PEs in hub add cross-VNet hops and cost; workload-local PEs fit ownership/RBAC. Zone group registration into a zone in another subscription requires permissions (Private DNS Zone Contributor) for the deploying identity/policy.
**Code / CLI / Bicep**
```bicep
resource zg 'Microsoft.Network/privateEndpoints/privateDnsZoneGroups@2023-11-01' = {
  parent: pe, name: 'default'
  properties: { privateDnsZoneConfigs: [ { name: 'kv', properties: { privateDnsZoneId: '/subscriptions/<connSub>/resourceGroups/rg-dns/providers/Microsoft.Network/privateDnsZones/privatelink.vaultcore.azure.net' } } ] }
}
```

### 12.4 [Medium] Force spoke egress through the hub firewall: pieces required.
**Answer:** UDR 0.0.0.0/0 -> firewall private IP on spoke subnets, peering with forwarded traffic allowed, firewall policy rules for needed destinations, DNS proxy as needed, and return-path symmetry.
**Explanation:** Exclude services needing direct paths via specific routes/service tags (e.g., certain platform dependencies). PE traffic inside the VNet usually bypasses (has /32 system route) unless PE network policies + UDR are enabled.
**Code / CLI / Bicep**
```bash
az network route-table route create -g rg-hub --route-table-name rt-spoke -n default --address-prefix 0.0.0.0/0 --next-hop-type VirtualAppliance --next-hop-ip-address 10.0.1.4
```

### 12.5 [Hard] Scale and governance: avoid IP overlap and enforce guardrails across 50 spokes.
**Answer:** Use an IPAM plan (central CIDR allocation, Azure Virtual Network Manager IPAM – verify), management-group policies (deny public IPs, require NSG, deny peering outside approved hub), and AVNM security admin rules that override NSGs for baseline denies.
**Explanation:** AVNM security admin rules are evaluated before NSG rules and can't be overridden by workload owners. Policy `Deny` for `Microsoft.Network/publicIPAddresses` in spoke subscriptions. Subscription vending automates spoke creation.
**Code / CLI / Bicep**
```bicep
// AVNM: Microsoft.Network/networkManagers + connectivityConfigurations (HubAndSpoke) + securityAdminConfigurations
```
**Gotchas / follow-ups**
- Hub firewall becomes a throughput/SNAT/availability bottleneck – use zones and sizing.
- Cross-region hubs: global peering charges and latency.
- Overlaps with on-prem ranges kill hybrid connectivity.

### 12.6 [Tricky] Spoke VM can't reach on-prem through the hub VPN gateway. Peering is up. Why?
**Answer:** Hub peering side lacks "Allow gateway transit" or spoke lacks "Use remote gateways", or the gateway is created after the peering without resync; also on-prem must advertise/receive spoke prefixes.
**Explanation:** A VNet can't "use remote gateways" if it has its own gateway. Check effective routes for on-prem prefixes (next hop VirtualNetworkGateway). With a firewall UDR, ensure on-prem prefixes route correctly.
**Code / CLI / Bicep**
```bash
az network vnet peering update -g rg-net -n spoke-to-hub --vnet-name vnet-spoke --set useRemoteGateways=true
```

**Interview traps**
- Spoke-to-spoke not automatic.
- Zones in the wrong subscription lacking permissions.
- Overlapping IP spaces with on-prem.
- Single firewall = single choke point if undersized.

**Cheat sheet**
- Hub: GW, FW, Bastion, DNS resolver. Spoke: workloads.
- Gateway transit (hub) + use remote gateways (spoke).
- Central private DNS zones + policy DINE.
- AVNM security admin rules beat NSGs.
- vWAN = managed hub alternative.

---

## Topic 13: Common Production Errors (Playbook)

**Study notes**

| Symptom | Likely cause | First check |
|---|---|---|
| Storage 403 `AuthorizationFailure` | Firewall denies caller path; same-region IP rule ineffective | Storage diagnostic `CallerIpAddress`, PE/DNS |
| 403 `AuthorizationPermissionMismatch` | Missing data-plane RBAC | Role assignment/scope, propagation |
| Function -> private SQL timeout | DNS/route/NSG/PE state, pool exhaustion | `nameresolver`, `tcpping :1433` |
| Works before PE, fails after | DNS now returns private IP in some networks, public blocked elsewhere | DNS from each client network |
| Intermittent outbound failures `SocketException`/timeouts | SNAT port exhaustion | App Service `SNAT Port Exhaustion` diagnostics, connections reuse |
| Pipeline can't reach Key Vault | Hosted agent on public IPs; vault private/firewalled | Self-hosted agent in VNet |

### 13.1 [Easy] 403 "This request is not authorized to perform this operation" on Blob from your Function. First move?
**Answer:** Determine whether it's network or identity: `AuthorizationFailure` points to the firewall/network path, `AuthorizationPermissionMismatch` to RBAC. Check the storage firewall, PE/DNS, and the caller IP in logs.
**Explanation:** Look at `StorageBlobLogs` for status text and caller IP: a public IP means traffic bypassed the PE; a private IP means PE path worked and RBAC is next.
**Code / CLI / Bicep**
```kusto
StorageBlobLogs
| where StatusCode in (401,403) and TimeGenerated > ago(1h)
| project TimeGenerated, OperationName, StatusText, CallerIpAddress, AuthenticationType
```

### 13.2 [Follow-up] DNS issue after enabling a private endpoint: what typically happens?
**Answer:** The service FQDN resolves differently by network: private IP in linked VNets, public IP elsewhere (laptop, build agents, on-prem without forwarder). With public access disabled, those other clients start failing with 403 or timeouts.
**Explanation:** Fix by linking zones/adding forwarders or giving clients private network access. Verify per client network with nslookup; check cached DNS (TTL) after changes.
**Code / CLI / Bicep**
```bash
nslookup stg.blob.core.windows.net          # from VM, Kudu, on-prem, and agent – compare
```

### 13.3 [Medium] SNAT port exhaustion – what is it, symptoms, and fixes?
**Answer:** Outbound connections to public IPs consume SNAT ports; each instance has a limited preallocated set, and when exhausted new connections fail/timeout intermittently. Fix with connection reuse (static `HttpClient`/pooling), NAT Gateway, private endpoints/service endpoints, and retries with backoff.
**Explanation:** Common cause in Functions: `new HttpClient()` per invocation. Idle timeouts hold ports (TIME_WAIT). NAT Gateway offers ~64k ports per public IP (verify). Use App Service diagnostics "SNAT Port Exhaustion" and metrics on NAT/LB.
**Code / CLI / Bicep**
```csharp
// register once; Functions isolated worker
services.AddHttpClient("partner", c => c.BaseAddress = new Uri("https://api.partner.com"))
        .ConfigureHttpClient(c => c.Timeout = TimeSpan.FromSeconds(30));
```

### 13.4 [Medium] Azure DevOps/GitHub-hosted agent "cannot reach Key Vault" (403 / ForbiddenByFirewall).
**Answer:** Hosted agents use dynamic public IPs not in the vault firewall; with private vault they can't resolve/reach it at all. Use self-hosted agents (or Managed DevOps Pools / VNet-injected runners) inside the VNet, or move secret retrieval to deploy-time via identity from within the network.
**Explanation:** Adding "trusted services" won't help – DevOps isn't a trusted service for data-plane. Whitelisting hosted IP ranges is brittle. ARM deployment of secrets (`getSecret`) runs from ARM, subject to Key Vault "enabled for template deployment" and firewall bypass.
**Code / CLI / Bicep**
```bash
az keyvault network-rule add --name kv-prod --ip-address <agentStaticIp>/32   # only for static self-hosted egress IPs
```

### 13.5 [Hard] Intermittent Function timeouts to Redis/SQL under load after scaling out. Diagnose.
**Answer:** Suspect connection storms (each instance opens pools), SNAT/port limits, Redis connection limits, SQL pool exhaustion/serverless pause, and cold-start DNS. Use singletons for `ConnectionMultiplexer`, bounded pool sizes, and retry policies.
**Explanation:** Correlate App Insights dependency failures with instance count; check Redis `connected_clients`/`Server Load`; use PE to reduce SNAT. Set `maxConcurrentCalls` on triggers to match backend capacity.
**Code / CLI / Bicep**
```csharp
services.AddSingleton<IConnectionMultiplexer>(_ =>
    ConnectionMultiplexer.Connect(new ConfigurationOptions { EndPoints = { "redis-prod.redis.cache.windows.net:6380" }, Ssl = true, AbortOnConnectFail = false }));
```
```kusto
dependencies | where success == false and timestamp > ago(1h) | summarize count() by target, resultCode, bin(timestamp, 5m)
```
**Gotchas / follow-ups**
- Redis Entra tokens expire; client must refresh (use the official extension).
- Retry storms amplify outages – use jitter/circuit breaker.
- Scale controller + `host.json` concurrency settings.

### 13.6 [Tricky] Everything works after deploy, breaks a day later with 401/unresolved Key Vault references. Why?
**Answer:** Likely secret/cert rotation or RBAC/identity change, an expired UAMI/role assignment, firewall changed by policy, or the Key Vault reference version pinned to a disabled version.
**Explanation:** Check app setting reference status, Key Vault `AuditEvent` logs, and Azure Policy remediation/activity logs for recent changes. Soft-deleted/disabled secret versions produce failures.
**Code / CLI / Bicep**
```kusto
AzureDiagnostics | where ResourceProvider == "MICROSOFT.KEYVAULT" and httpStatusCode_d >= 400
| project TimeGenerated, OperationName, httpStatusCode_d, CallerIPAddress, identity_claim_appid_g
```

**Interview traps**
- 403 variants differ; read the error code.
- `new HttpClient()` per call.
- Hosted agents + private Key Vault.
- Same-region storage IP rules.

**Cheat sheet**
- Timeout=network; 403 AuthorizationFailure=firewall; PermissionMismatch=RBAC.
- Reuse connections; NAT GW fixes SNAT.
- Compare DNS from every client network.
- Self-hosted/MDP agents for private vaults.
- Always check diagnostic logs for caller IP.

---

## Topic 14: Compliance, Data Residency & End-to-End Secure Design

**Study notes**

- **Data residency**: data at rest stays in the chosen region unless you geo-replicate (GRS/RA-GRS, SQL geo-replication, Service Bus geo-replication/Geo-DR, Cosmos multi-region) – replication goes to the *paired* or chosen region; check each service's residency commitments (some services store metadata/telemetry elsewhere; verify in current docs). Enforce with Azure Policy **Allowed locations**; keep Log Analytics workspace/App Insights in the compliant region.
- **Compliance tooling**: Microsoft Trust Center/Service Trust Portal, Defender for Cloud regulatory compliance dashboard (ISO 27001, PCI DSS, SOC, NIST, etc.), Azure Policy compliance, Purview. Shared responsibility: Microsoft secures the platform; you secure config, identity, data.
- **Customer-managed keys** (CMK) in Key Vault/Managed HSM for Storage, SQL TDE, Service Bus (Premium), Redis (verify) when required.

**Target architecture**

```
 Internet -> Front Door (WAF) --PrivateLink--> APIM (internal, validate-jwt, rate-limit)
                                                 | authentication-managed-identity
                                                 v
 [Spoke VNet]   Function App (Premium/Flex, VNet-int snet-func, inbound PE, UAMI)
     |-- PE -> Service Bus (Premium, local auth off)      zone privatelink.servicebus.windows.net
     |-- PE -> Blob Storage (shared key off)              zone privatelink.blob.core.windows.net
     |-- PE -> Key Vault (RBAC, purge protection)         zone privatelink.vaultcore.azure.net
     |-- PE -> Azure SQL (Entra-only auth)                zone privatelink.database.windows.net
     |-- PE -> Azure Cache for Redis (Entra auth)         zone privatelink.redis.cache.windows.net
 NAT GW (static egress)  | NSGs | UDR -> Hub Firewall | DNS -> Hub Private Resolver
 Diagnostics -> Log Analytics -> Defender for Cloud + Sentinel ; Policy: deny public access, allowed locations
 CI/CD: OIDC federation + self-hosted agents in VNet; no secrets in repos
```

### 14.1 [Easy] How do you keep data in a specific country/region?
**Answer:** Deploy resources only in approved regions (enforced by Allowed Locations policy), avoid or choose replication targets inside the boundary, and keep monitoring/log data in-region.
**Explanation:** "Region" + "pair" choice drives GRS replication. Some services are non-regional or store metadata globally (e.g., Entra ID, Front Door config/logs) – verify per service. Document in compliance evidence.
**Code / CLI / Bicep**
```bash
az policy assignment create -n allowed-locs --policy e56962a6-4747-49cd-b67b-bf8b01975c4c --scope /subscriptions/<sub> \
  --params '{"listOfAllowedLocations":{"value":["germanywestcentral","swedencentral"]}}'
```

### 14.2 [Follow-up] Which identity roles does the Function UAMI need in this design?
**Answer:** Azure Service Bus Data Receiver/Sender, Storage Blob Data Contributor (plus host storage roles: Blob Data Owner, Queue/Table Data Contributor), Key Vault Secrets User (only for third-party secrets), a SQL contained user with minimal roles, and Redis data access via Entra access policy.
**Explanation:** Scope roles to resources, not subscription. Separate UAMIs per bounded context for least privilege. Redis uses Redis data access policies (Data Owner/Contributor/Reader) – verify.
**Code / CLI / Bicep**
```bash
az role assignment create --assignee-object-id <uamiPrincipalId> --assignee-principal-type ServicePrincipal \
  --role "Azure Service Bus Data Receiver" --scope <sbNamespaceId>
az role assignment create --assignee-object-id <uamiPrincipalId> --assignee-principal-type ServicePrincipal \
  --role "Storage Blob Data Contributor" --scope <stgId>
```

### 14.3 [Medium] Bicep: assemble the Function's secure config.
**Answer:** VNet integration + inbound PE + public access off + UAMI + identity-based app settings for Service Bus and host storage, route-all on, minimum TLS 1.2, HTTPS only.
**Explanation:** App settings carry no secrets—only endpoints and client IDs. Key Vault only for external secrets.
**Code / CLI / Bicep**
```bicep
resource func 'Microsoft.Web/sites@2023-01-01' = {
  name: funcName, location: loc, kind: 'functionapp,linux'
  identity: { type: 'UserAssigned', userAssignedIdentities: { '${uami.id}': {} } }
  properties: {
    serverFarmId: planId
    httpsOnly: true
    virtualNetworkSubnetId: funcSubnetId
    vnetRouteAllEnabled: true
    publicNetworkAccess: 'Disabled'
    keyVaultReferenceIdentity: uami.id
    siteConfig: {
      minTlsVersion: '1.2'
      ftpsState: 'Disabled'
      functionsRuntimeScaleMonitoringEnabled: true
      appSettings: [
        { name: 'FUNCTIONS_WORKER_RUNTIME', value: 'dotnet-isolated' }
        { name: 'AzureWebJobsStorage__accountName', value: stg.name }
        { name: 'AzureWebJobsStorage__credential', value: 'managedidentity' }
        { name: 'AzureWebJobsStorage__clientId', value: uami.properties.clientId }
        { name: 'ServiceBusConnection__fullyQualifiedNamespace', value: '${sb.name}.servicebus.windows.net' }
        { name: 'ServiceBusConnection__credential', value: 'managedidentity' }
        { name: 'ServiceBusConnection__clientId', value: uami.properties.clientId }
      ]
    }
  }
}
```

### 14.4 [Medium] How do you prove the design is compliant and stays that way?
**Answer:** Policy initiatives (deny public network access, require TLS 1.2, allowed locations, diagnostic settings DINE), Defender for Cloud regulatory dashboard, and Sentinel analytics on activity logs for drift; IaC in pipelines with what-if and policy checks.
**Explanation:** Preventive (Policy), detective (Defender/Sentinel), and corrective (DINE/remediation) controls. Evidence exports from compliance dashboard/Service Trust Portal.
**Code / CLI / Bicep**
```bash
az deployment group what-if -g rg-app -f main.bicep -p env=prod
az policy state summarize --resource-group rg-app
```

### 14.5 [Hard] Walk through a message's path and every control it crosses; what is the blast radius if the Function is compromised?
**Answer:** Client -> Front Door WAF -> APIM (JWT, rate-limit) -> Function (identity-validated) -> Service Bus via PE -> processing -> Blob/SQL/Redis via PEs with RBAC. A compromised Function can only act with its UAMI's roles on those specific resources, from its subnet; no keys to steal, egress is firewalled, and Defender/Sentinel alert on anomalies.
**Explanation:** Limit blast radius with per-app UAMIs, least-privilege roles (receiver-only where possible), SQL roles instead of db_owner, firewall egress allow-list, PE-per-resource exfil limits, disabled key auth, and monitoring on Key Vault/Storage access anomalies.
**Code / CLI / Bicep**
```kusto
AzureDiagnostics | where ResourceProvider == "MICROSOFT.KEYVAULT" and OperationName == "SecretGet"
| summarize gets=count() by CallerIPAddress, identity_claim_appid_g, bin(TimeGenerated, 1h) | where gets > 100
```
**Gotchas / follow-ups**
- Function identity token via IDENTITY_ENDPOINT can be abused by code running in the app – scope roles tightly.
- Disaster recovery: replicate PEs/DNS in the secondary region; DNS zones are global but PEs aren't.
- Non-prod must mimic prod networking, or "works in dev" is meaningless.

### 14.6 [Tricky] Interviewer: "Everything is private but deployment is failing and monitoring has gaps." What did you forget?
**Answer:** Deployment path (CI agents with line-of-sight to SCM/storage/Key Vault), diagnostic settings/Log Analytics reachability (use Azure Monitor Private Link Scope if required), App Insights ingestion over private link, and DNS for these endpoints.
**Explanation:** App Insights/Log Analytics can use AMPLS; otherwise telemetry uses public endpoints (acceptable if policy allows, but it's data egress). Also forgot break-glass access, Bastion for admin, and emergency runbooks.
**Code / CLI / Bicep**
```bash
az monitor private-link-scope create -g rg-mon -n ampls-prod
az monitor private-link-scope scoped-resource create -g rg-mon --scope-name ampls-prod -n law --linked-resource <lawId>
```

**Interview traps**
- "Private endpoints = compliant" – identity, logging, policy, residency also matter.
- Overly broad roles on shared UAMI.
- Forgetting AMPLS/monitoring and deployment paths.
- Assuming GRS keeps data "in country" – it replicates to the pair.

**Cheat sheet**
- Layers: Edge (FD+WAF) -> Gateway (APIM) -> Compute (Function) -> Data (PE + RBAC).
- No secrets: UAMI + Entra auth everywhere; KV only for third-party.
- Policy = prevent, Defender = detect, Sentinel = respond.
- Residency: allowed locations + replication choices + log location.
- Always plan deployment, monitoring, and DR paths.

