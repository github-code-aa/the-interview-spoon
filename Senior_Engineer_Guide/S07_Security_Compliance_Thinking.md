# S07 – Security & Compliance Thinking (Senior Engineer Interview Guide)

**How to use:** Read the study notes, then cover the answers and speak each question aloud; compare against the "Strong senior answer", and rehearse the follow-up traps.
Senior answers flow: **assets -> threats -> controls -> trade-offs -> residual risk**. Items marked *(verify in current docs)* are fast-changing Azure/.NET features.

## Table of Contents
1. Security Mindset for Architects
2. Threat Modelling
3. Zero Trust and Network Security on Azure/.NET
4. Authentication and Authorization Architecture
5. Service-to-Service Security
6. Secrets and Key Management
7. OWASP API Top 10 as a Design Review Checklist
8. Data Protection
9. PII / GDPR Design
10. Multi-Tenant Isolation
11. Audit Logging and Non-Repudiation
12. Supply Chain Security and Secure SDLC
13. API Security Controls
14. Frontend / BFF Token Handling
15. Security Incident Response
16. Compliance Basics and Security-vs-Velocity Trade-offs
17. Scenarios
18. Talking About Security Without Buzzwords

---

## Topic 1: Security Mindset for Architects

**Study notes**
- **Defence in depth**: independent layers so one failure is not fatal (edge WAF -> auth -> authorization -> input validation -> least-privilege DB login -> encryption -> monitoring).
- **Least privilege**: every identity gets the minimum permission, scope and time (managed identity with a single DB role, not `db_owner`; JIT admin access via PIM).
- **Secure by default**: the unconfigured path is the safe path (deny-by-default authorization, `[Authorize]` fallback policy, private endpoints on, public access off).
- **Shift left**: catch issues at design/PR time (threat model, analyzers, secret scanning) because cost of a fix grows with distance from the author.
- Security is risk management: you rarely get "secure", you get "risk reduced to an accepted level by someone accountable".

```
 Internet --> [WAF] --> [Gateway: coarse authn] --> [Service: authn + authz + validation]
                                                         |
                                                [DB: least-priv login, RLS, encryption]
 Each arrow is a trust boundary; each box must still defend itself.
```

### 1.1 [Easy] What is defence in depth and why not rely on one strong control?
**What the interviewer is really testing**: Whether you understand that controls fail and you design for failure.
**Typical mid-level answer**: "Multiple layers of security like firewall and authentication." Falls short: lists layers, doesn't explain independence or failure assumptions.
**Strong senior answer**: Every single control has a failure mode: a WAF is bypassed, a token validation bug ships, a developer misconfigures a storage account. Defence in depth means layers that fail independently, so an attacker needs several simultaneous failures. For an order API: WAF filters generic attacks, the API validates the JWT and enforces resource-level authorization, EF queries are parameterised, the DB login can only execute needed procedures/tables, and sensitive columns are encrypted; monitoring alerts when layers disagree (e.g. 403 spikes). The key is independence: if all layers rely on the same assumption ("the gateway authenticated this"), it is one layer wearing four hats. Trade-off is cost, latency and complexity, so I spend layers proportionally to asset value. Residual risk: a determined insider with legitimate access still passes most layers, hence audit and anomaly detection.
**Follow-up traps**: "Isn't that redundant/wasteful?" (only if layers share failure modes). "Which layer matters most?" (the one guarding the asset - authorization and data layer, not the perimeter).

### 1.2 [Follow-up] Explain least privilege with a concrete Azure + SQL example.
**What the interviewer is really testing**: Can you turn a principle into specific role assignments?
**Typical mid-level answer**: "Give only the permissions needed." Falls short: no concrete scoping or lifecycle.
**Strong senior answer**: Assets are customer data and the ability to change infrastructure. The API runs with a user-assigned managed identity that maps to a contained SQL user in a custom database role with EXECUTE on specific schemas, not `db_owner`. Storage access uses `Storage Blob Data Reader` scoped to one container, not Contributor on the subscription. Human admins get eligible (not permanent) roles via Privileged Identity Management with approval and expiry. Deployment pipelines get a separate identity with rights only on the target resource group. Migrations run under a distinct higher-privilege identity used only in the release step, so runtime compromise cannot alter schema. Trade-off: more identities and role definitions to manage; mitigate with IaC. Residual risk: privilege creep, so I schedule access reviews.
**Follow-up traps**: "App needs to run EF migrations at startup" (separate migration identity, don't give runtime DDL). "Role scope: subscription vs resource?" (narrowest that works).

### 1.3 [Medium] What does "secure by default" look like in an ASP.NET Core service?
**What the interviewer is really testing**: Designing frameworks so that forgetting a control fails closed.
**Typical mid-level answer**: "Add `[Authorize]` to controllers." Falls short: opt-in security means a forgotten attribute is a hole.
**Strong senior answer**: Make the dangerous thing explicit and the safe thing implicit. Set a `FallbackPolicy` requiring an authenticated user so any new endpoint is protected unless marked `[AllowAnonymous]`, and review those markers specifically. Bind input to DTOs (not entities) to avoid mass assignment; enable HSTS and HTTPS redirect; cookies `Secure`/`HttpOnly`/`SameSite`; Problem Details without stack traces in production; config with `ValidateOnStart` so missing security settings crash startup rather than fall back to insecure defaults. In Azure, Azure Policy denies public network access and enforces TLS 1.2+, so the platform defaults are safe too. Trade-off: new devs hit friction ("why 401?") - mitigate with templates and docs. Residual risk: policy gaps for new resource types.
```csharp
builder.Services.AddAuthorizationBuilder()
    .SetFallbackPolicy(new AuthorizationPolicyBuilder().RequireAuthenticatedUser().Build());
```
**Follow-up traps**: "What about health endpoints?" (explicit AllowAnonymous, minimal info). "Defaults in dev vs prod?" (same code path; avoid `if (dev)` security bypasses).

### 1.4 [Hard] "Shift left" is often just a slogan. How do you make it real without slowing teams?
**What the interviewer is really testing**: Pragmatism - integrating controls into the workflow with measurable signal-to-noise.
**Typical mid-level answer**: "Run SAST in the pipeline." Falls short: noisy tools get ignored; no ownership.
**Strong senior answer**: Place cheap, high-signal checks closest to the developer: Roslyn/security analyzers in the IDE, pre-commit/push secret scanning, PR-level dependency review. Place expensive ones (DAST, deep SAST) in nightly or pre-release. Tune for precision: start with a small ruleset of high-severity issues, block only on those, track the rest as backlog with SLAs. Add design-level shift-left: a short threat-model prompt in the design doc template for any new trust boundary or data class. Measure: time-to-fix, false-positive rate, percent of repos covered. Trade-off: gates slow merges; so keep PR checks under minutes and give an exception path with expiry. Residual risk: tooling finds known patterns, not logic flaws like BOLA, which need design review and tests.
**Follow-up traps**: "Developers disable the scanner" (fix noise, make it the paved road). "Who owns findings?" (the owning team, with security as advisor).

### 1.5 [Tricky] The business wants a feature next week that weakens a control. What do you do?
**What the interviewer is really testing**: Risk communication and ownership, not blocking or caving.
**Typical mid-level answer**: "Refuse" or "Just ship it." Falls short: either extreme.
**Strong senior answer**: First clarify the business goal; often a safer way meets it. Then state the risk in business terms: what asset, likelihood, impact, who is affected. Offer options: ship with compensating controls (extra logging, rate limits, scope limited to a pilot tenant), time-boxed exception with an owner and expiry, or defer. Document the decision as an accepted risk signed by the accountable owner, not by engineering alone. If the risk is regulatory or involves customer data exposure with no mitigation, I escalate rather than quietly accept. Trade-off: velocity vs exposure; the point is making it a conscious decision. Residual risk is tracked on a register with a review date.
**Follow-up traps**: "Exception never gets revisited" (expiry + ticket). "Manager overrules you" (document, escalate if legal/regulatory).

**Red flags in answers**
- Treating security as a perimeter/firewall topic or a separate team's job.
- Listing tools and acronyms with no reasoning about assets or threats.
- "We'll add security at the end" or "it's behind the VPN so it's safe".

**Cheat sheet**
- Layers must fail independently; same assumption = same layer.
- Least privilege = scope + permission + time.
- Fail closed: fallback authorization policy, deny public access by default.
- Shift left only works when signal-to-noise is high.
- Accepted risk needs an owner, expiry and compensating controls.
- Always end with residual risk.

---

## Topic 2: Threat Modelling

**Study notes**
- Four questions (Shostack): What are we building? What can go wrong? What will we do about it? Did we do a good job?
- **DFD elements**: external entities, processes, data stores, data flows, **trust boundaries** (where the level of trust changes). Threats cluster at boundary crossings.
- **STRIDE**: Spoofing (authn), Tampering (integrity), Repudiation (audit), Information disclosure (confidentiality), Denial of service (availability), Elevation of privilege (authz).
- **Abuse cases**: write "as an attacker I..." next to user stories.

```
 [Browser/Mobile] --TLS--> |B1| [API Gateway] --> |B2| [Orders svc] --> [SQL]
                                         |                  |
                                         v                  v
                                  [Identity (Entra)]   [Service Bus] --> |B3| [Payments svc] --> [PSP (3rd party)]
 |Bn| = trust boundary. Threat each flow and each store crossing a boundary.
```

### 2.1 [Easy] Walk me through how you'd start a threat model for a new service.
**What the interviewer is really testing**: A repeatable method, not ad-hoc brainstorming.
**Typical mid-level answer**: "List possible attacks like SQL injection." Falls short: starts from attacks, not from the system and assets.
**Strong senior answer**: I start with scope and assets: what data and capabilities does this service hold (PII, payment tokens, ability to approve refunds)? Then draw a data-flow diagram with trust boundaries: client to gateway, gateway to service, service to DB, service to broker, service to third parties. For each element and flow I apply STRIDE and record threats, existing mitigations, and gaps. I prioritise by impact and likelihood, not by interesting-ness, and turn gaps into backlog items with owners. I time-box it to an hour or two for a typical service and keep the artifact in the repo next to the design doc so it is updated when the architecture changes. Residual risk is listed explicitly. The model is a living document, revisited on new boundaries, data classes or integrations.
**Follow-up traps**: "Who attends?" (devs, architect, ops, security - those who know the system). "How often?" (on change of boundary/data, not calendar).

### 2.2 [Follow-up] Apply STRIDE to a microservice system: Orders -> Service Bus -> Payments.
**What the interviewer is really testing**: Can you produce specific, system-relevant threats and controls?
**Typical mid-level answer**: Generic definitions of each letter. Falls short: no concrete mapping.
**Strong senior answer**: Spoofing: a rogue service publishes `PaymentRequested` - control: managed identities with Service Bus Data Sender on that topic only, no shared SAS keys. Tampering: message altered or order total changed - control: Payments re-derives amount from authoritative order data rather than trusting the message; TLS in transit; consider signing for high-value messages. Repudiation: "I never approved that refund" - control: audit events with actor, correlation ID, immutable store. Information disclosure: card/PII in messages or logs - control: tokens/IDs instead of data, log scrubbing, private endpoints. Denial of service: poison message or flood - control: dead-letter, max delivery count, rate limits, idempotent consumers. Elevation of privilege: Orders service identity can read Payments queue - control: per-entity RBAC, separate identities per service. Residual risk: compromise of a legitimate sender within its permissions, so business-rule validation stays in the consumer.
**Follow-up traps**: "Message replay?" (idempotency keys, dedupe window). "Is the broker trusted?" (treat as a semi-trusted channel; validate at consumer).

### 2.3 [Medium] What are abuse cases and how do they change design?
**What the interviewer is really testing**: Thinking adversarially about legitimate features.
**Typical mid-level answer**: "Misuse cases for testing." Falls short: doesn't show impact on design.
**Strong senior answer**: User stories describe intended behaviour; abuse cases ask how a legitimate feature can be turned against us. "As a user I can reset my password" becomes "as an attacker I enumerate valid emails, flood a victim with reset mails, or hijack the reset token." This drives design: uniform responses, per-account and per-IP rate limits, short-lived single-use tokens, notification on change. "Export my invoices as CSV" becomes CSV/formula injection and bulk scraping: escape leading `=+-@`, cap export size, audit exports. I write two or three abuse cases per story that touches money, identity or data export, and each yields an acceptance criterion or a test. Trade-off: extra refinement time; it pays by finding logic flaws scanners cannot. Residual risk: creative attackers outrun the list, so monitoring remains important.
**Follow-up traps**: "Who writes them?" (whole team, security facilitates). "How verified?" (automated tests per abuse case).

### 2.4 [Hard] How do you rank threats and decide which to fix?
**What the interviewer is really testing**: Risk-based prioritisation instead of fix-everything.
**Typical mid-level answer**: "Use CVSS and fix highest first." Falls short: CVSS is vulnerability severity, not your context.
**Strong senior answer**: Rank by impact on specific assets times likelihood given exposure: is it internet-reachable, what privileges are needed, is there an existing compensating control, how easy is exploitation? A medium CVSS flaw on an unauthenticated internet endpoint exposing all tenants outranks a critical in an internal admin tool behind private endpoints and MFA. Use a simple scheme (e.g. DREAD is deprecated by many; a plain High/Med/Low with written rationale is fine). Options per threat: mitigate, transfer (insurance, managed service), avoid (drop the feature), accept with owner. I give each mitigated item an owner and date and keep the accepted ones on a risk register. Trade-off: judgement is subjective; calibrate by reviewing with others. Residual risk is what's left on the register.
**Follow-up traps**: "Everything's high priority" (force ranking, limit WIP). "Accept risk who signs?" (business owner).

### 2.5 [Tricky] Your threat model says "attacker is an insider with valid credentials". What changes?
**What the interviewer is really testing**: Beyond the perimeter - insider/assumed breach thinking.
**Typical mid-level answer**: "Trust employees / use firewalls." Falls short: ignores compromised-account scenarios.
**Strong senior answer**: Assets are bulk data and ability to alter records. Controls shift from preventing entry to limiting blast radius and detecting abuse: least privilege and JIT for admin access, no standing production data access, row-level authorization at the data layer, query/export rate limits, separation of duties (the person who deploys cannot approve their own change), tamper-evident audit logs written to an append-only store the actor cannot edit, and anomaly alerts (mass reads, off-hours access) via Defender/Sentinel *(verify in current docs)*. Customer-managed keys with separated key custodians limit a DBA reading plaintext. Trade-off: friction for legitimate operators and cost of monitoring. Residual risk: a collusion of privileged insiders; mitigated by dual control on the most sensitive actions.
**Follow-up traps**: "Can a DBA read encrypted data?" (Always Encrypted: no, TDE: yes). "How to detect?" (baseline + alerts, not just logs).

**Red flags in answers**
- Naming STRIDE letters without mapping them to a diagram or concrete controls.
- Threat modelling as a one-time document nobody updates.
- Ranking purely by CVSS or by what is technically interesting.

**Cheat sheet**
- 4 questions: build, go wrong, respond, verify.
- Draw trust boundaries first; threats live at crossings.
- STRIDE maps to: authn, integrity, audit, confidentiality, availability, authz.
- Abuse cases turn features into tests.
- Mitigate / transfer / avoid / accept (with owner).
- Update model on new boundary, data class or integration.

---

## Topic 3: Zero Trust and Network Security on Azure/.NET

**Study notes**
- Zero trust: **verify explicitly**, **least privilege**, **assume breach**. Network location is not identity.
- Implement via strong identity (Entra ID, MFA, Conditional Access), device health, per-request authorization, micro-segmentation, continuous monitoring.
- Network layer still matters: **Private Endpoints** (Private Link) for PaaS, disable public network access, NSGs, **egress control** with Azure Firewall/NAT, **WAF** (Front Door or Application Gateway) for L7 filtering, DDoS protection.
- VNet integration for App Service/Functions outbound; private DNS zones required for private endpoints.

```
 Internet -> [Front Door + WAF] -> [App Service (VNet-integrated)] --PE--> [SQL / KeyVault / Storage]
                                          |                                   (public access disabled)
                                          +--> [Azure Firewall / NAT] -> allow-listed FQDNs only
```

### 3.1 [Easy] What does zero trust mean in practice, beyond the buzzword?
**What the interviewer is really testing**: Translating principles into concrete designs.
**Typical mid-level answer**: "Never trust, always verify." Falls short: no specifics.
**Strong senior answer**: It means no request is trusted because of where it comes from. Concretely: every service call carries a verifiable identity (managed identity tokens, not "it's inside the VNet"); every service validates the token and authorizes for itself; users face MFA and Conditional Access based on device and risk; access is least-privilege and time-bound; and we assume breach by segmenting, limiting egress and logging for detection. For a .NET API on Azure: Entra-issued tokens validated by the API, SQL accessed via managed identity, Key Vault with RBAC, private endpoints. It is a direction not a product; I prioritise by asset value. Trade-off: more identity plumbing and operational overhead. Residual risk: misconfigured conditional access or stolen tokens, so token lifetimes and anomaly detection matter.
**Follow-up traps**: "Is a VPN zero trust?" (no, it's network-based trust). "Can you buy zero trust?" (no).

### 3.2 [Follow-up] Why use Private Endpoints if the service already requires authentication?
**What the interviewer is really testing**: Understanding layering and the exposure surface.
**Typical mid-level answer**: "Extra security." Falls short.
**Strong senior answer**: Authentication protects against unauthorised use but not against exposure to the whole internet: a leaked credential, a zero-day in the service endpoint, or brute force is reachable by anyone if the endpoint is public. A private endpoint puts a private IP in my VNet and I disable public network access, so a stolen connection string or SAS token is useless from outside. It also supports exfiltration control, since data can't be sent to arbitrary storage accounts when egress is restricted. Costs: per-endpoint charge, private DNS zone management, harder local dev and CI access (use bastion/self-hosted agents/dev VNet), cross-region/multi-tenant complexity. Residual risk: an attacker inside the VNet or the app itself compromised; hence identity-based auth remains.
**Follow-up traps**: "Service endpoints vs private endpoints?" (service endpoints keep the public IP/FQDN and allow subnet-based firewall rules; private endpoints give a private IP - verify in current docs). "DNS issues?" (private DNS zone linking).

### 3.3 [Medium] Where does a WAF help and where does it not?
**What the interviewer is really testing**: Realistic expectations of perimeter tools.
**Typical mid-level answer**: "WAF blocks SQL injection and XSS." Falls short: overstates it.
**Strong senior answer**: A WAF (Azure Front Door/App Gateway with OWASP managed rule sets) gives virtual patching, bot and rate controls, geo filtering and a central place to block active exploits like a Log4Shell-style wave while code is patched. It does not understand business logic: BOLA/IDOR, broken authorization, mass assignment and abuse of legitimate flows look like normal requests. Running in detection mode first avoids false-positive outages, then tuning exclusions for legitimate payloads (rich text, JSON). Also ensure the origin only accepts traffic from the front door (header `X-Azure-FDID` check or private link) or the WAF is bypassed. Trade-off: latency, tuning effort. Residual risk: everything application-logic, which stays in code and tests.
**Follow-up traps**: "Can I skip input validation with a WAF?" (no). "Origin exposed directly?" (bypass).

### 3.4 [Hard] How do you control egress and why does it matter?
**What the interviewer is really testing**: Assume-breach thinking: stopping exfiltration and callbacks.
**Typical mid-level answer**: "Block outbound with NSG." Falls short: ignores FQDN-level needs.
**Strong senior answer**: After compromise, attackers need to call home and exfiltrate; SSRF also abuses outbound reach (e.g. metadata endpoints, internal services). Route outbound from VNet-integrated apps through Azure Firewall (or NAT gateway with fixed IPs) using FQDN allow-lists for required destinations (identity endpoints, package feeds, partner APIs). Use private endpoints for Azure dependencies so those never traverse the internet. Log denied flows and alert. For code-level SSRF: URL allow-lists, block link-local and private ranges, don't follow redirects blindly. Trade-off: operational friction whenever a new dependency is added, and firewall cost; keep an onboarding process with quick approvals. Residual risk: allow-listed domains that host attacker content (e.g. general cloud storage), so allow by specific tenant/account where possible.
**Follow-up traps**: "Does Azure Firewall solve SSRF?" (partly; app-level controls too). "Wildcard FQDN?" (broad, avoid).

### 3.5 [Tricky] "Our services are all in one VNet, so internal calls don't need auth." Respond.
**What the interviewer is really testing**: Challenging flat-network trust.
**Typical mid-level answer**: "Add auth anyway." Falls short: doesn't argue the threat.
**Strong senior answer**: Network placement is an attribute of the attacker's foothold, not a guarantee of identity. One compromised pod, a vulnerable dependency, or an SSRF in any service gives an attacker lateral access to every internal API; with no auth they can read or write anything and we cannot even tell who did it. Each service should authenticate callers (managed identity tokens validated by the receiver) and authorize by caller identity/role, and NSGs/network policies segment so only expected paths exist. Cost is token acquisition code (cached by `DefaultAzureCredential`/MSAL) and small latency. I would pilot it on the most sensitive services first and roll out by risk. Residual risk: a compromised service still acts with its own permissions; minimal permissions per service limit that.
**Follow-up traps**: "Performance?" (token cached ~1 hour, local validation). "Service mesh?" (can provide mTLS identity, but app-level authorization still needed).

**Red flags in answers**
- "Inside the VNet = trusted."
- Treating WAF as complete protection for logic flaws.
- Private endpoints configured but public access left enabled.

**Cheat sheet**
- Verify explicitly, least privilege, assume breach.
- Identity is the new perimeter; network is still a layer.
- Private endpoint + disable public access + private DNS.
- WAF = virtual patching, not business-logic security; lock the origin.
- Egress allow-lists limit exfiltration and SSRF impact.
- Internal calls still authenticate and authorize.

---

## Topic 4: Authentication and Authorization Architecture

**Study notes**
- **Authentication** = who; **authorization** = what they may do. Centralise authentication in an IdP (Microsoft Entra ID / External ID / another OIDC provider); don't build password handling per service.
- **Token validation at every service**: signature (JWKS), `iss`, `aud`, `exp`, and required scopes/roles. Gateway-only validation makes the gateway a single point of failure and lets anything inside the network bypass it.
- **Scopes** (delegated permissions: what the *app* may do on behalf of a user) vs **roles** (what the *user/app* is assigned) vs **claims** (facts in the token) vs **policies** (ASP.NET Core rules built from claims/requirements).
- **Resource-based / ABAC**: decisions that depend on the resource instance and context (owner, tenant, state, time) - cannot be expressed in the token alone; use `IAuthorizationService` with handlers.
- Tokens: access tokens are for APIs (short-lived); ID tokens are for the client, never sent to APIs.

```
 Client --(OIDC auth code + PKCE)--> Entra ID --> access token (aud=api://orders, scp=orders.read)
 Client --Bearer--> Gateway (coarse: valid token, rate limit) --> Orders API (validates again: iss/aud/scp)
                                                                     |-> resource check: order.TenantId == caller tenant
```

### 4.1 [Easy] Authentication vs authorization, and where does each live in the architecture?
**What the interviewer is really testing**: Basic clarity plus placement of responsibility.
**Typical mid-level answer**: "AuthN is login, AuthZ is permissions." Falls short: nothing about where each is enforced.
**Strong senior answer**: Authentication establishes identity and is centralised in the IdP so credentials, MFA, and federation live in one hardened place; services only validate the resulting token. Authorization is decided by each service because only it knows the resource and business rules: coarse checks (scope/role) at the endpoint and fine-grained checks (does this user own this order, is it in a state that allows cancel) in domain/application code. Pushing authorization entirely into the IdP or gateway can't express resource-level decisions. Trade-off: authorization logic is spread out, so I centralise it into policy classes and shared libraries with tests. Residual risk: inconsistent enforcement across services; mitigated by a fallback policy and review checklist.
**Follow-up traps**: "Where do tenant checks go?" (data access layer plus policy, not only controller). "Is the ID token usable by API?" (no).

### 4.2 [Follow-up] Why is "gateway-only authentication" a pitfall?
**What the interviewer is really testing**: Spotting the confused-deputy and flat-trust antipattern.
**Typical mid-level answer**: "It's fine, gateway handles auth." Falls short.
**Strong senior answer**: If downstream services trust headers like `X-User-Id` set by the gateway, any request that reaches them another way (another compromised service, misrouted traffic, a new public endpoint, an SSRF) can forge identity. The gateway also lacks resource context, so authorization decisions are weak, and it becomes a hot, high-value single point of failure. Better: gateway does edge concerns (TLS, WAF, coarse token check, throttling, routing) while each service validates the JWT itself (cheap local signature check using cached keys) and enforces authorization. If an identity must be propagated, pass the original token or a signed internal token, never plain headers. Trade-off: duplicated validation config, solved with a shared NuGet package. Residual risk: misconfigured audience on one service; add integration tests for 401/403 paths.
**Follow-up traps**: "Performance cost of re-validating?" (negligible, local crypto). "What if gateway terminates mTLS?" (still authorize in service).

### 4.3 [Medium] Scopes, roles, claims, policies - how do you choose?
**What the interviewer is really testing**: Modelling permissions without role explosion.
**Typical mid-level answer**: "Use roles for everything." Falls short: coarse and leads to role explosion or privilege bloat.
**Strong senior answer**: Scopes describe what a client application is allowed to do on behalf of a user (`orders.read`) and bound the damage of a compromised client. App roles/groups describe the user's or service's organisational permission (`OrderApprover`). Claims are inputs (tenant ID, department). Policies combine them in code: "has scope orders.write AND role OrderApprover AND approval limit >= amount". For instance-dependent rules use resource-based authorization. I avoid putting many permissions in the token: group claims overage (users in many groups cause a claim overage - verify in current docs) and stale permissions until expiry. For fast-changing or fine-grained permissions I look them up from an authorization store with caching. Trade-off: token size and freshness vs lookup latency. Residual risk: revocation delay for long-lived tokens.
```csharp
options.AddPolicy("CanApprove", p => p
    .RequireClaim("scp", "orders.write")
    .RequireRole("OrderApprover"));
```
**Follow-up traps**: "Revoke a user's permission now?" (short token lifetime, CAE where supported, or live lookup). "Too many roles?" (move to attribute/resource rules).

### 4.4 [Hard] How do you implement resource-based / ABAC authorization in .NET?
**What the interviewer is really testing**: Practical handling of BOLA at the design level.
**Typical mid-level answer**: "Check the user ID in the controller." Falls short: scattered, easy to forget.
**Strong senior answer**: Define requirements such as `OrderOwnerOrManager` with an `AuthorizationHandler<TRequirement, Order>`, load the resource, and call `IAuthorizationService.AuthorizeAsync(user, order, policy)` in the application service so every path goes through the same logic. Better still, make the data access safe by default: repository methods take the caller context and scope queries (`Where(o => o.TenantId == ctx.TenantId)`), so an unscoped fetch isn't possible. Return 404 rather than 403 for resources outside the caller's tenant to avoid existence leaks, where that suits the API. For complex rules consider a policy engine (OPA, Cedar-like, Azure-based options - verify in current docs) but weigh the extra moving part. Test with a matrix: user A vs resource of user B for every endpoint. Trade-off: loading the resource before authorising costs a query. Residual risk: new endpoints forgetting the check; add architecture tests.
**Follow-up traps**: "List endpoints?" (filter in query, not post-filter). "Bulk operations?" (authorize each, or constrain by query).

### 4.5 [Tricky] Token validated, signature good - what else could still go wrong?
**What the interviewer is really testing**: Depth on JWT pitfalls.
**Typical mid-level answer**: "Check expiry." Falls short.
**Strong senior answer**: A valid signature only proves the issuer minted it. I also check `aud` (token meant for this API, otherwise a token for another API is replayed against mine), `iss` and tenant (multi-tenant Entra apps accept tokens from any tenant unless I restrict issuers), `exp/nbf` with small clock skew, required scopes/roles, and token type (access not ID). Reject `alg: none` and keep an algorithm allow-list; use the metadata/JWKS endpoint and handle key rollover. Don't trust claims like email for identity - use the immutable `sub`/`oid` plus `tid`. Bearer tokens are replayable if stolen, so keep lifetimes short, use TLS, don't log tokens, and consider proof-of-possession for high-risk cases *(verify in current docs)*. Residual risk: stolen valid token within its lifetime.
**Follow-up traps**: "Client app uses client credentials: which claim?" (`roles` rather than `scp`). "Mixing user and app tokens?" (separate policy for each).

**Red flags in answers**
- Trusting `X-User-Id`-style headers from a gateway.
- Authorization expressed only by role names in controllers, with no resource checks.
- Not validating audience/issuer, or using the ID token as an API token.

**Cheat sheet**
- IdP centralises authn; each service validates and authorizes.
- Validate: signature, iss, aud, exp, scope/role, tenant.
- Scope = client capability; role = principal's permission; policy = rules in code.
- Resource checks belong in app/data layer, with query scoping.
- 404 vs 403 can matter for enumeration.
- Fallback policy = deny by default.

---

## Topic 5: Service-to-Service Security

**Study notes**
- **Managed identity** (system- or user-assigned): Azure-issued identity for workloads; no secret to store; use `DefaultAzureCredential` (prefer `ManagedIdentityCredential` in production for predictability).
- **Client credentials flow**: service gets a token as itself (app roles in `roles` claim).
- **On-Behalf-Of (OBO)**: a middle-tier API exchanges the user's token for a downstream token preserving user identity (delegated). Use when the downstream must authorize the *user*.
- **mTLS**: both sides present certificates; strong transport identity, but cert lifecycle burden; often provided by a mesh (verify in current docs for AKS options).
- **Internal vs external trust**: internal doesn't mean trusted; external callers (partners) get separate auth, quotas and contract.

```
 User --token(A)--> Orders API --OBO--> token(B, user context) --> Inventory API   (user's rights apply)
 Orders API --client creds / MI--> token(C, app identity) --> Reporting API       (service's rights apply)
```

### 5.1 [Easy] Why prefer managed identity over a client secret or connection string with password?
**What the interviewer is really testing**: Eliminating secrets by design.
**Typical mid-level answer**: "It's more secure." Falls short.
**Strong senior answer**: The primary asset is the credential itself: secrets leak via repos, logs, CI variables, and developers' laptops, and need rotation. A managed identity has no secret for me to hold; Azure platform issues short-lived tokens to the workload through the local identity endpoint, and the credential is bound to the resource. I grant it a specific role (Key Vault Secrets User, SQL contained user, Service Bus Data Receiver). Benefits: no rotation, no leak path, auditable per-identity actions. Trade-offs: only works for Azure-supported targets, local dev needs developer credentials (`az login` via `DefaultAzureCredential`), identity lifecycle tied to resource (use user-assigned for sharing across slots/instances or to pre-provision roles). Residual risk: anyone who can run code in that resource can use the identity, so code-execution isolation and least privilege matter.
**Follow-up traps**: "System vs user-assigned?" (system dies with resource; user-assigned reusable and pre-provisionable). "Calling a non-Azure API?" (federated credentials/workload identity federation or Key Vault-held secret).

### 5.2 [Follow-up] When would you use On-Behalf-Of versus service identity?
**What the interviewer is really testing**: Confused deputy and who is accountable for the action.
**Typical mid-level answer**: "OBO to pass the user." Falls short: no criteria.
**Strong senior answer**: Use OBO when the downstream's authorization or audit must reflect the end user, e.g. a document service where the user may only see their files. Use the service's own identity when the operation is a system function (nightly aggregation, sending notifications) and the user's rights are irrelevant or insufficient. The risk of service identity is a confused deputy: a powerful service identity performs an action requested by an unprivileged user, so the calling service must itself authorize the user first. The risk of OBO is token chains growing (each hop adds scope surface) and latency; limit hops and scopes, and verify `aud` at each. In Microsoft identity platform OBO is supported via MSAL `AcquireTokenOnBehalfOf` and needs consent configuration - verify in current docs. Residual risk: over-broad delegated scopes.
**Follow-up traps**: "Can I just forward the user's token?" (only if the audience is valid for downstream; usually not - a token for API A shouldn't work on B). "Event-driven flow, no user?" (carry user ID as data, authorize at producer, use service identity).

### 5.3 [Medium] mTLS vs token-based auth between services: how do you decide?
**What the interviewer is really testing**: Trade-off reasoning, not tool preference.
**Typical mid-level answer**: "mTLS is more secure." Falls short.
**Strong senior answer**: They solve different problems. mTLS authenticates the transport peer and encrypts; it's strong where I control both ends and have automated cert issuance/rotation (service mesh, SPIFFE-style, internal CA). Tokens carry identity and authorization claims, support scopes/audience, and work across HTTP intermediaries (gateways and proxies that terminate TLS would break end-to-end mTLS). In Azure PaaS with managed identity, token-based is the lowest-friction and gives per-resource RBAC. I'd use token auth by default and add mTLS when regulation or threat model demands transport-level identity, or for partner B2B where certificate-based client auth is expected. Combining them (mTLS plus token) is a defence-in-depth option for high-value flows. Trade-off: certificate lifecycle burden vs token complexity. Residual risk: cert expiry outages; automate and monitor.
**Follow-up traps**: "Who rotates certs?" (automation, short lifetimes). "Does mTLS give authorization?" (no, only identity).

### 5.4 [Hard] How do you separate "internal" and "external" trust levels in an API landscape?
**What the interviewer is really testing**: Segmentation and contract design.
**Typical mid-level answer**: "Public APIs behind gateway, internal ones private." Falls short: ignores that internal is not safe.
**Strong senior answer**: Define trust tiers by what the caller is, not where it sits. External/partner callers: unique credentials, tight scopes, per-client quotas and throttling, strict schema validation, versioned contracts, no direct DB access, audit per client. Internal callers: managed identity, authorization per service, minimal roles, but still validation of inputs because a compromised internal service is a realistic threat. Separate APIs (or at least routes and policies) for external versus internal use, so internal-only operations (admin, bulk) are never exposed by routing mistakes. Deploy private endpoints for internal APIs; put external behind APIM/Front Door. Trade-off: duplicated surfaces and mapping layers. Residual risk: internal API accidentally exposed - use Azure Policy and network tests to detect.
**Follow-up traps**: "Share one API with an `isInternal` flag?" (no, separate policies and surfaces). "Partner leaks key?" (per-client revoke without affecting others).

### 5.5 [Tricky] A service's managed identity has Contributor on the resource group "because it's easier". Argue the risk and fix.
**What the interviewer is really testing**: Blast radius reasoning and pragmatic remediation.
**Typical mid-level answer**: "Reduce permissions." Falls short: no process.
**Strong senior answer**: Contributor lets the identity create, modify, and delete nearly everything in the group, including changing network rules or reading connection strings and keys (control plane lists keys). An SSRF or RCE in that service converts into full environment takeover. Fix: inventory what it actually calls (Azure Activity Logs/diagnostic logs, or access analyzer reviewing sign-ins), then replace with data-plane roles on specific resources (Blob Data Contributor on one container). Roll out in a lower environment, then in prod with a rollback; use IaC so assignments are reviewed in PRs, and Azure Policy/alerts to flag broad assignments. Trade-off: initial effort and occasional breakage when permissions are missed. Residual risk: legitimately needed roles that are still powerful; mitigated by scope and monitoring.
**Follow-up traps**: "Can't figure out what it needs?" (observe logs, iterate). "Owner role?" (even worse - can assign roles).

**Red flags in answers**
- Shared client secrets between services stored in config.
- Forwarding user tokens to any downstream without audience considerations.
- Believing mTLS replaces authorization.

**Cheat sheet**
- Managed identity first; secrets only when no alternative.
- OBO for user context; service identity for system tasks - authorize the user first.
- Tokens carry authz context; mTLS gives transport identity.
- Per-resource data-plane RBAC, not Contributor.
- External callers: quotas, per-client credentials, schema validation.
- Confused deputy = privileged service acting for unprivileged caller.

---

## Topic 6: Secrets and Key Management

**Study notes**
- Goal: **no long-lived secrets** in code, config, images, or CI logs. Prefer identity-based access (managed identity, workload identity federation for GitHub Actions/AKS) over secrets.
- **Key Vault**: secrets, keys, certificates; use RBAC authorization, soft-delete and purge protection; Managed HSM for stricter requirements *(verify tiers in current docs)*.
- **Rotation**: automated, overlap windows (two valid keys), version-aware references. Key Vault references in App Service/Functions and the configuration provider fetch secrets at runtime.
- **Break-glass**: emergency accounts excluded from Conditional Access failures, strongly protected, monitored, tested.
- Envelope encryption: data keys encrypted by a key-encryption key (KEK) in Key Vault.

```
 App (managed identity) --RBAC--> Key Vault (private endpoint, purge protection, logging)
 CI (OIDC federated credential, no stored secret) --> deploy
 Alert: any secret read by unexpected identity / outside pattern
```

### 6.1 [Easy] Where should secrets live, and where must they never live?
**What the interviewer is really testing**: Basic hygiene and the whole lifecycle view.
**Typical mid-level answer**: "In Key Vault, not in code." Falls short: ignores config files, CI, logs, images.
**Strong senior answer**: Never in source control (including history), appsettings, Docker images or layers, build logs, error messages, URLs/query strings, or tickets. Secrets belong in a managed store (Key Vault) retrieved at runtime by a managed identity; for CI use OIDC federation so pipelines hold no static cloud secret. Local dev uses `dotnet user-secrets` or developer credentials against a dev vault, never prod values. Prefer eliminating the secret: managed identity to SQL, Storage, Service Bus. Where unavoidable (third-party API keys), store in Key Vault with rotation and owner metadata. Add secret scanning (push protection) and log scrubbing as safety nets. Residual risk: a secret in application memory can be dumped by an attacker with code execution, so least privilege on what each secret unlocks.
**Follow-up traps**: "Environment variables safe?" (better than code, but visible to process and crash dumps; Key Vault references still resolve to env at runtime). "user-secrets in prod?" (no, dev only).

### 6.2 [Follow-up] How do you design secret rotation without downtime?
**What the interviewer is really testing**: Operational design, not just "rotate regularly".
**Typical mid-level answer**: "Change the password every 90 days." Falls short: causes outages and ignores automation.
**Strong senior answer**: Make rotation boring and automatic. Use two credentials (primary/secondary, e.g. storage account keys or SQL logins) or versioned secrets: create new, publish to Key Vault, apps pick it up (config reload or periodic refresh), verify usage of the new one, then revoke the old one after an overlap window. Event Grid notifications from Key Vault on near-expiry can trigger rotation functions *(verify in current docs)*. Better yet, shift to short-lived credentials (managed identity tokens, Entra auth to SQL) where rotation is implicit. Test rotation regularly in non-prod and alert if a deprecated secret is still used. Trade-off: apps must tolerate credential refresh (retry on auth failure, re-read secret); extra complexity. Residual risk: unknown consumers of a secret break; inventory via Key Vault access logs.
**Follow-up traps**: "App caches secret forever?" (refresh interval / handle 401 by reload). "Calendar rotation vs event-driven?" (do both: scheduled plus on-suspicion).

### 6.3 [Medium] Key Vault design decisions: one vault or many? Access model?
**What the interviewer is really testing**: Blast radius and operations.
**Typical mid-level answer**: "One vault for everything." Falls short.
**Strong senior answer**: Separate by environment at minimum, and by application/team or sensitivity so access boundaries match role assignments; vault-level RBAC (rather than legacy access policies) gives per-secret scoping and standard auditing. Enable soft-delete and purge protection so an attacker or mistake can't irreversibly delete keys, which also guards encrypted data availability. Restrict network (private endpoint) and send diagnostics to Log Analytics with alerts on unusual reads and on admin operations. Throttling limits matter: cache secrets in the app rather than reading per request. Use Managed HSM or HSM-backed keys when compliance demands FIPS 140-validated protection *(verify in current docs)*. Trade-off: more vaults means more management; use IaC modules. Residual risk: Key Vault admin roles are powerful; protect with PIM and separation of duties.
**Follow-up traps**: "Who can read secrets?" (data-plane roles, owners of vault can grant themselves - control with policy). "Rate limit hits?" (cache).

### 6.4 [Hard] What is break-glass access and how do you do it safely?
**What the interviewer is really testing**: Planning for IdP/outage failure without creating a permanent backdoor.
**Typical mid-level answer**: "Admin account for emergencies." Falls short.
**Strong senior answer**: Break-glass accounts exist so that if Conditional Access, MFA provider, or federation fails, someone can still administer the tenant. Typically two cloud-only accounts, excluded from the standard CA policies but protected with phishing-resistant MFA like FIDO2 keys where possible, long random credentials split between custodians in a physical safe, not tied to individuals. Any sign-in triggers a high-severity alert to multiple people; usage requires an incident record and subsequent credential rotation. Test them periodically (and confirm alerts). For app secrets, break-glass equivalents are documented, tested restore procedures for keys and vault backups. Trade-off: it's a deliberate hole; its safety comes from monitoring and procedure. Residual risk: custodian compromise or alert failure.
**Follow-up traps**: "Why exclude from CA?" (avoid lockout from CA misconfig). "Used for daily admin?" (never).

### 6.5 [Tricky] Developers want production secrets locally to debug a prod issue. What do you do?
**What the interviewer is really testing**: Holding the line pragmatically with alternatives.
**Typical mid-level answer**: "Give them to the devs." / "Say no." Falls short.
**Strong senior answer**: The real need is diagnosing prod behaviour, so offer paths that don't copy secrets: structured logs and traces in Application Insights, snapshot debugging/profiler where permitted *(verify in current docs)*, reproducible data in a masked staging environment, feature flags and diagnostic endpoints guarded by authorization. If access is truly necessary, provide time-boxed JIT access through PIM with approval, ticket and audit, using the dev's own identity rather than sharing a service credential. Production secrets should not reach laptops: they are copied, cached, synced to OneDrive, and break non-repudiation. Trade-off: slower troubleshooting initially; invest in observability to compensate. Residual risk: privileged operators with JIT access can still exfiltrate, so audit.
**Follow-up traps**: "Emergency at 3am?" (JIT path with fast approval). "Share one connection string?" (no; attribution lost).

**Red flags in answers**
- Secrets "encrypted" in appsettings or base64 in config.
- Rotation described as manual, calendar-only, with no consumer inventory.
- Key Vault without purge protection or access logging.

**Cheat sheet**
- Best secret is no secret: managed identity, OIDC federation.
- Key Vault: RBAC, soft-delete + purge protection, private endpoint, diagnostics.
- Rotate with overlap; automate; test.
- Scan commits (push protection), scrub logs.
- Break-glass: 2 accounts, FIDO2, alerted, tested.
- Prod secrets never to laptops; use JIT.

---

## Topic 7: OWASP API Top 10 as a Design Review Checklist

**Study notes**
- OWASP API Security Top 10 (2023 edition) includes: API1 Broken Object Level Authorization (BOLA), API2 Broken Authentication, API3 Broken Object Property Level Authorization (excessive data exposure + mass assignment), API4 Unrestricted Resource Consumption, API5 Broken Function Level Authorization, API6 Unrestricted Access to Sensitive Business Flows, API7 SSRF, API8 Security Misconfiguration, API9 Improper Inventory Management, API10 Unsafe Consumption of APIs. *(Verify numbering in current OWASP docs.)*
- Use it as a design-review prompt per endpoint: *who* may call, *which object*, *which properties*, *how often*, *what does it call outward*.

```
 Review per endpoint:  authn? -> function-level authz? -> object-level authz? -> property filter (in/out DTO)?
                       -> rate/size limits? -> outbound calls validated? -> logged? -> in inventory?
```

### 7.1 [Easy] What is BOLA/IDOR and why is it the top API risk?
**What the interviewer is really testing**: Understanding that authentication isn't authorization.
**Typical mid-level answer**: "Users accessing others' data by changing the ID." Falls short: no prevention strategy.
**Strong senior answer**: BOLA occurs when an endpoint takes an object identifier (`/orders/123`) and returns/changes it without checking that the caller may access that specific object. It's top-ranked because it's logic-level: scanners and WAFs can't see it, it's easy to exploit by incrementing IDs, and it appears wherever any endpoint accepts an ID. Prevention is structural: scope every query by caller (tenant/owner) in the data layer, authorize with resource-based policies, avoid relying on unguessable GUIDs as security (obscurity helps but isn't control), and write automated cross-user tests for each endpoint. Return not-found for others' objects. Trade-off: slight code overhead; benefit is removing a whole class. Residual risk: indirect references (IDs inside bodies, nested routes, batch endpoints) are forgotten; review them specifically.
**Follow-up traps**: "GUID IDs fix it?" (no, leaked/shared IDs still work). "Batch endpoint accepting IDs array?" (authorize each).

### 7.2 [Follow-up] Explain excessive data exposure and mass assignment together. What's the common fix?
**What the interviewer is really testing**: Property-level control in both directions.
**Typical mid-level answer**: "Don't return too much; validate input." Falls short.
**Strong senior answer**: Both stem from binding API shapes directly to internal entities. Returning an EF entity serialises fields like `PasswordHash`, `InternalNotes`, `IsAdmin`. Binding request JSON to the entity lets a caller set `IsAdmin=true` or `TenantId`. The fix: explicit input and output DTOs per use case (separate create/update/read), mapped deliberately; server-set fields (tenant, owner, status, audit fields) are never taken from the client. Where output varies by role, use distinct DTOs or explicit projections, not conditional nulling scattered across code. In EF, `Select` to DTOs reduces over-fetching too. Add tests asserting that forbidden properties are ignored/rejected, and use `[JsonIgnore]`-free design (explicit types). Trade-off: more types and mapping code. Residual risk: DTO growth drift; review diffs on DTOs.
**Follow-up traps**: "Use `[Bind]`/ignore attributes?" (deny-lists fail open; use allow-lists via DTOs). "PATCH endpoints?" (allow-listed fields).

### 7.3 [Medium] How do you design against unrestricted resource consumption and abusable business flows?
**What the interviewer is really testing**: Availability and cost/abuse thinking beyond simple rate limiting.
**Typical mid-level answer**: "Add rate limiting." Falls short.
**Strong senior answer**: Assets are availability, cost (cloud bills, SMS/email sends), and business integrity (ticket scalping, coupon abuse). Controls in layers: edge throttling (Front Door/APIM), per-client and per-user limits in-app (ASP.NET Core rate limiting middleware - note in-memory limiters are per-instance; use a distributed store or gateway for global limits), payload size caps, pagination with max page size, timeouts and cancellation tokens, query complexity limits (GraphQL depth), bounded concurrency, and queue-based load levelling. For sensitive flows (signup, password reset, checkout, SMS OTP) add flow-specific limits, CAPTCHA/step-up, and anomaly detection. Set budgets/alerts for cost. Trade-off: legitimate bursts may be blocked; return 429 with `Retry-After`. Residual risk: distributed low-and-slow abuse; rely on monitoring.
**Follow-up traps**: "Rate limit by IP?" (NAT/CGNAT and botnets; combine with identity). "Unbounded `pageSize`?" (cap).

### 7.4 [Hard] Use the OWASP list to review an existing API surface - how do you scale this to 200 endpoints?
**What the interviewer is really testing**: Programme-level thinking: inventory, automation, prioritisation.
**Typical mid-level answer**: "Review each endpoint manually." Falls short.
**Strong senior answer**: Start with inventory (API9): generate the endpoint list from OpenAPI/ApiExplorer and APIM, identify shadow/deprecated versions. Triage by exposure and data sensitivity. Automate detection of structural issues: reflection/architecture tests that fail any endpoint lacking an authorization attribute/policy, any controller action accepting an entity type, any query on a tenant-owned table not through the scoped repository, any `HttpClient` call with a user-supplied URL. Add integration tests in a cross-tenant/cross-user matrix generated from the inventory. Review manually only the high-risk set (payments, admin, bulk export). Track findings per category with owners and SLAs, and add the fix as a platform default (e.g. base repository) so the class is eliminated, not just instances. Trade-off: upfront investment. Residual risk: logic issues missed by automation; periodic pen-test targeted at them.
**Follow-up traps**: "Where do you start?" (internet-exposed + sensitive data). "Old v1 endpoints?" (retire or equal protection).

### 7.5 [Tricky] SSRF: what is it, and why does cloud make it worse?
**What the interviewer is really testing**: Understanding the metadata/identity angle.
**Typical mid-level answer**: "Server fetches attacker-supplied URL." Falls short: no cloud impact or controls.
**Strong senior answer**: SSRF makes the server issue requests on the attacker's behalf from a privileged network position. In cloud it can reach the instance metadata endpoint (169.254.169.254 or the App Service identity endpoint) and obtain managed identity tokens, or hit internal APIs that trust the network. Features like webhooks, URL preview, image import, and PDF generation are typical sources. Controls: avoid fetching user URLs; otherwise allow-list schemes/hosts, resolve DNS and block private/link-local/loopback ranges after resolution (and re-check after redirects to prevent DNS rebinding), disable auto-redirect, enforce timeouts and response size limits, run fetchers in an isolated low-privilege identity/network with egress firewall. Trade-off: functionality limits on arbitrary URLs. Residual risk: allow-listed hosts that redirect or host attacker content.
**Follow-up traps**: "Block 'localhost' string?" (bypass via DNS/IP encodings; validate resolved IPs). "IMDS on Azure?" (header requirement helps but not a full defence).

**Red flags in answers**
- Thinking authentication covers BOLA.
- Blocklist-based input filtering (`[Bind(Exclude=...)]`) as the sole control.
- Rate limiting only by client IP, in-memory, on multi-instance deployments.

**Cheat sheet**
- Per endpoint: authn, function-level authz, object-level authz, property filtering, limits, outbound calls.
- DTOs in and out; server-set fields never from client.
- Scope queries by caller in the data layer.
- Inventory first (shadow/old versions).
- Automate structural checks with tests and analyzers.
- SSRF: allow-list, post-DNS IP checks, no redirects, egress control.

---

## Topic 8: Data Protection

**Study notes**
- **In transit**: TLS 1.2+ everywhere (including internal hops and DB connections: `Encrypt=True`, don't blindly `TrustServerCertificate=True`); HSTS.
- **At rest**: Azure Storage and Azure SQL TDE are on by default with service-managed keys; **customer-managed keys (CMK)** in Key Vault give control over revocation and rotation.
- **Always Encrypted**: client-side column encryption; SQL Server never sees plaintext keys; deterministic (equality lookups) vs randomized; secure enclaves enable richer operations *(verify in current docs)*.
- **Dynamic Data Masking**: obfuscates query results for non-privileged users; it's a presentation control, **not** encryption or strong access control.
- **Tokenisation** replaces sensitive value with a surrogate; the real value lives in a vault/PSP.
- Hashing passwords: use slow adaptive hashes (ASP.NET Core Identity's PBKDF2 default or Argon2/bcrypt) - never reversible encryption.

```
 Client --TLS--> API --TLS--> SQL (TDE at rest; CMK in Key Vault)
                        |-- column SSN: Always Encrypted (key in KV; DB engine/DBAs see ciphertext)
 Support UI: Dynamic Data Masking on email/phone for non-privileged roles
```

### 8.1 [Easy] TDE vs Always Encrypted vs Dynamic Data Masking - what does each protect against?
**What the interviewer is really testing**: Matching control to threat.
**Typical mid-level answer**: "All of them encrypt data." Falls short: DDM doesn't encrypt.
**Strong senior answer**: TDE encrypts data files, logs, and backups at rest, protecting against stolen disks/backups; the DB engine decrypts transparently so a DBA or SQL injection still reads plaintext. Always Encrypted encrypts at the column by the client driver, so the server and its administrators see ciphertext; it protects against high-privilege insiders and cloud operators, at the cost of query limits (deterministic equality only, randomized none; enclaves extend) and application changes. DDM masks output for lower-privileged users (e.g. show `XXX-1234`) which reduces casual exposure, but users with UNMASK or clever inference queries can discover values; it's not a security boundary. Pick by threat: lost media -> TDE; privileged-user access -> Always Encrypted; accidental exposure in UIs -> DDM/app-level masking. Residual risk: key compromise, app memory.
**Follow-up traps**: "Is DDM secure against a determined user?" (no; inference via WHERE clauses). "Can I search Always Encrypted columns?" (equality on deterministic; enclaves for more).

### 8.2 [Follow-up] When do you need customer-managed keys, and what do they cost?
**What the interviewer is really testing**: Cost/benefit vs checkbox compliance.
**Typical mid-level answer**: "For better security." Falls short.
**Strong senior answer**: Service-managed keys already encrypt at rest and are fine for most workloads. CMK is justified when a regulation, contract, or threat model requires control over key lifecycle: ability to revoke (crypto-shredding by disabling the key), custody separate from the cloud operator and the DB admins, and auditable key use. Cost: availability coupling (if the key is disabled/deleted or the vault inaccessible, data becomes inaccessible - purge protection and soft delete are mandatory), key rotation process, additional permissions for the service identity to the key, and potential throttling/latency. Design: user-assigned identity with `Key Vault Crypto Service Encryption User` on a dedicated vault/HSM, backup and DR plan across regions, alerts on key state. Residual risk: key misconfiguration causing outage.
**Follow-up traps**: "Disabling the key = deletion of data?" (access loss; crypto-shred only with no backups of key). "Multi-region?" (key availability in DR region).

### 8.3 [Medium] How do you decide what to encrypt at the application/column level versus rely on platform encryption?
**What the interviewer is really testing**: Risk-based data classification.
**Typical mid-level answer**: "Encrypt everything sensitive." Falls short: ignores usability.
**Strong senior answer**: Start with classification: public, internal, confidential, regulated (PII, card, health). For regulated data where DBAs, support tools, or analytics shouldn't see plaintext, use column-level protection (Always Encrypted) or app-layer encryption with envelope keys, or better, don't store it (tokenise). For common PII, platform encryption plus tight RBAC/RLS, masking in lower roles, and audit often suffices. Account for functional needs: searchability (blind index/deterministic), sorting, reporting. Consider non-prod: copy production data only masked or synthetic. Trade-off: encrypting columns breaks ad-hoc SQL and indexes; plan query patterns first. Residual risk: key management errors and plaintext copies in logs, caches, exports.
**Follow-up traps**: "Search by email in an encrypted column?" (deterministic or HMAC blind index). "Backups/exports?" (inherit protection).

### 8.4 [Hard] Design card/PII tokenisation. What are the trade-offs?
**What the interviewer is really testing**: Scope reduction and system design.
**Typical mid-level answer**: "Replace value with random string." Falls short.
**Strong senior answer**: Tokenisation replaces the sensitive value with a surrogate that has no mathematical relation to the original; the mapping is held in a hardened token vault (or by a PSP like Stripe/Adyen), so downstream systems store only tokens and fall out of scope for the sensitive data. Decisions: format-preserving vs opaque, deterministic (same value -> same token, enabling joins/dedupe but revealing equality) vs random per-use, who can detokenise (a narrow service with strong authorization and audit), and token scoping per tenant/merchant so a token from one can't be used by another. Using a third-party provider moves the vault risk and compliance burden to them. Trade-off: dependency on the vault (availability, latency) and loss of analytics on raw values. Residual risk: detokenisation service compromise; limit callers, rate-limit, alert.
**Follow-up traps**: "Token vs encryption?" (encryption is reversible with key everywhere; tokens need the vault). "Is a token PII?" (may still be personal data if linkable; verify with legal).

### 8.5 [Tricky] TLS is terminated at the gateway; backend runs plain HTTP inside the VNet. Acceptable?
**What the interviewer is really testing**: Internal threat awareness and compliance context.
**Typical mid-level answer**: "Yes, it's internal." Falls short.
**Strong senior answer**: Depends on threat model and regulation, but the default should be TLS end-to-end. Plaintext internal traffic exposes tokens and data to anyone with a network foothold (compromised workload, misconfigured packet capture, shared network) and is often non-compliant for regulated data (PCI, HIPAA expectations). Managed platforms make it cheap: App Service/Container Apps provide HTTPS endpoints, Azure SQL enforces encryption, and service meshes can do mTLS. If I accept plaintext on a segmented, private hop temporarily, I document it as an accepted risk with compensating controls and a retirement date. Trade-off: cert management and small CPU cost (usually negligible). Residual risk: TLS interception at gateways needs protected keys.
**Follow-up traps**: "Performance concerns?" (modern TLS overhead small with session reuse). "Cert validation off in dev code?" (never ship `ServerCertificateCustomValidationCallback` returning true).

**Red flags in answers**
- Saying Dynamic Data Masking is encryption or an access control.
- `TrustServerCertificate=True` in production connection strings.
- Encrypting with keys stored beside the data in the same config.

**Cheat sheet**
- TDE = stolen disks; Always Encrypted = privileged insiders; DDM = convenience masking.
- CMK needs purge protection; key loss = data loss.
- Classify data first; encrypt by risk.
- Tokenise to reduce scope; restrict detokenisation.
- TLS everywhere, validate certificates.
- Passwords: adaptive hash, never reversible.

---

## Topic 9: PII / GDPR Design

**Study notes**
- Principles (GDPR Art. 5): lawfulness/consent, purpose limitation, **data minimisation**, accuracy, **storage limitation (retention)**, integrity/confidentiality, accountability.
- Data subject rights: access, rectification, **erasure** ("right to be forgotten", with exceptions such as legal obligations), portability, objection.
- **DPIA** (Data Protection Impact Assessment): required for high-risk processing; describes processing, necessity, risks, mitigations.
- **Residency/transfers**: where data is stored and who can access; cross-border transfer mechanisms. Not legal advice - involve DPO/legal.
- Design patterns: PII in a separate store keyed by a pseudonymous ID; crypto-shredding for immutable stores; retention jobs; consent records.

```
 Orders (pseudonymous CustomerId) ----> Orders DB / Event store / Logs   (no raw PII)
 Customer Profile service (PII vault: name, email, address, encrypted per-subject key)
 Erasure = delete profile + destroy per-subject key -> historic events become unreadable
```

### 9.1 [Easy] What does data minimisation mean for a developer, concretely?
**What the interviewer is really testing**: Turning legal principle into design decisions.
**Typical mid-level answer**: "Collect less data." Falls short: no concrete practice.
**Strong senior answer**: For every field ask: what purpose requires it, who needs it, and how long. Concretely: don't collect date of birth if age-over-18 suffices; don't log request bodies with personal data; don't copy PII into events, analytics and caches when an ID will do; store at the lowest necessary granularity (postcode area rather than address); mask or drop in non-prod. Keep PII in a bounded set of services so access control, encryption, retention and erasure apply in one place and other services use pseudonymous IDs. Document fields in a data inventory with purpose, retention, and legal basis. Trade-off: some analytics or support convenience is lost; solved with aggregated or tokenised data. Residual risk: shadow copies (logs, exports, backups) outside the inventory.
**Follow-up traps**: "Why not keep everything, storage is cheap?" (liability, breach impact, legal basis). "Logs with emails?" (scrub or hash).

### 9.2 [Follow-up] How do you handle the right to erasure with immutable logs or event sourcing?
**What the interviewer is really testing**: Reconciling legal rights with architectural patterns.
**Typical mid-level answer**: "Delete the records." Falls short: doesn't fit append-only stores.
**Strong senior answer**: First, keep PII out of immutable streams: events carry a subject ID and non-identifying facts; PII lives in a mutable profile store, so erasure is a delete there and past events become non-identifying. Where PII must be in events, use crypto-shredding: encrypt per-subject fields with a per-subject key stored in a key store; erasure destroys the key (and handles backups since ciphertext remains unreadable). Alternatively, rewrite/compact streams for erasure, which is costly and risky. Audit logs may need retention under a legal obligation exception; GDPR erasure has exceptions, so involve legal and document basis. Also propagate erasure to downstream copies (search indexes, caches, data lake) via erasure events and verify completion. Trade-off: key-management complexity and joined-view costs. Residual risk: backups restored with the old key; control key backup policy.
**Follow-up traps**: "Backups?" (retention window, restored data re-erased via erasure log). "Is pseudonymised data still personal?" (yes if re-identifiable).

### 9.3 [Medium] Design retention and deletion in a SQL Server/Azure system.
**What the interviewer is really testing**: Operationalising storage limitation.
**Typical mid-level answer**: "A cleanup job." Falls short.
**Strong senior answer**: Define retention per data category with legal/business owners (e.g. invoices 7 years, support chats 1 year). Implement with technical aids: partitioning by date to drop old partitions cheaply, soft-delete followed by hard-delete after grace, scheduled jobs with batching to avoid lock/log explosion, and lifecycle management policies on Blob storage. Include non-primary copies: indexes, Cosmos TTL, Service Bus message TTL, log workspace retention, backups (retention aligned to policy), data lake. Provide evidence: deletion run logs and metrics. Test deletes against FKs and referential integrity; keep legal-hold exceptions. Trade-off: partition design complexity; cascading deletes risk. Residual risk: forgotten stores and backups; periodic data inventory audit.
**Follow-up traps**: "Soft delete only?" (still retained, so not erased). "Archived to cheap storage?" (still in scope).

### 9.4 [Hard] How do consent, residency, and DPIA affect architecture?
**What the interviewer is really testing**: Cross-cutting regulatory awareness without overclaiming legal expertise.
**Typical mid-level answer**: "Add a consent checkbox; use EU region." Falls short.
**Strong senior answer**: Consent must be specific, recorded (who, what purpose, version of text, timestamp), and withdrawable, which implies a consent store consulted by processing paths (marketing, analytics) and events propagating changes; it's not the only legal basis, so I check with legal whether consent applies at all. Residency: choose regions deliberately, but also consider backups, replicas, telemetry (Application Insights workspace region), support access from other regions, and third-party processors; document data flows and transfer mechanisms. DPIA: for high-risk processing (large-scale sensitive data, profiling), engineers contribute data-flow diagrams, necessity/proportionality, risks and technical measures, and the DPO signs off; I treat the DPIA as input to design, early. Trade-off: multi-region complexity and cost; sometimes dedicated regional deployments. Residual risk: legal interpretation changes; involve DPO and verify in current guidance.
**Follow-up traps**: "EU region = compliant?" (no; access and transfers matter too). "Who decides lawful basis?" (legal/DPO, not engineers).

### 9.5 [Tricky] Product wants to keep all user activity events forever "for ML later". Respond.
**What the interviewer is really testing**: Purpose limitation and constructive pushback.
**Typical mid-level answer**: "GDPR doesn't allow it." Falls short: absolutist and unhelpful.
**Strong senior answer**: Unspecified future use conflicts with purpose limitation and storage limitation, and increases breach liability. Ask what models and what signals are needed, then offer alternatives: pseudonymise or aggregate, define a retention window justified by model need, separate identifiers into a store that can be erased and break linkage, or use privacy-preserving techniques (differential privacy, synthetic data) where appropriate. Record the purpose in the data inventory and get legal basis confirmed. Provide a measured plan: keep raw events for N months, derive features, drop raw. Trade-off: some ML opportunities lost; the benefit is lower risk and compliance. Residual risk: re-identification from rich behavioural data even when pseudonymised.
**Follow-up traps**: "Anonymised vs pseudonymised?" (anonymised is out of scope only if truly irreversible). "Users opt out?" (propagate to pipelines).

**Red flags in answers**
- "Soft delete satisfies erasure."
- PII spread across logs, events, caches with no inventory.
- Claiming to give definitive legal interpretations rather than involving DPO/legal.

**Cheat sheet**
- Minimise: collect, copy, keep less.
- PII in one place; others hold pseudonymous IDs.
- Immutable stores: keep PII out or crypto-shred.
- Retention covers backups, logs, indexes, lakes.
- Consent is recorded, versioned, withdrawable.
- DPIA early; legal decides basis.

---

## Topic 10: Multi-Tenant Isolation

**Study notes**
- **Isolation models**: shared DB + shared schema (`TenantId` column), shared DB + schema-per-tenant, database-per-tenant (Azure SQL elastic pools), deployment/stamp-per-tenant. More isolation = more cost and operational overhead.
- **Row-Level Security (RLS)** in SQL Server/Azure SQL: security predicate (inline TVF) + security policy with FILTER (reads) and BLOCK (writes) predicates; tenant ID passed via `SESSION_CONTEXT`.
- **EF Core global query filters** scope queries by tenant; can be bypassed with `IgnoreQueryFilters()` and don't cover raw SQL.
- Pitfalls: connection pooling with session context not reset, caches keyed without tenant, background jobs with no tenant context, blob paths, search indexes, logs.

```
 Request -> authn -> TenantContext (from validated token `tid`/claim, NOT from header/body)
   -> EF global filter (app layer)  +  SESSION_CONTEXT -> RLS policy (DB layer)   [two independent layers]
   -> cache key = {tenant}:{key};  blob path = /{tenant}/...;  search filter = tenant
```

### 10.1 [Easy] Compare multi-tenant isolation models and how you'd pick.
**What the interviewer is really testing**: Trade-off reasoning, not a favourite pattern.
**Typical mid-level answer**: "Shared DB with TenantId is cheapest." Falls short.
**Strong senior answer**: Shared schema has lowest cost and easiest ops/migrations, but the biggest leakage blast radius (one missing filter exposes everyone), noisy-neighbour risk, and hard per-tenant restore. Database-per-tenant gives stronger isolation, per-tenant backup/restore, residency and encryption keys, at higher cost and more complex migrations (elastic pools and tooling mitigate). Stamp-per-tenant suits regulated or very large customers. I choose based on regulatory and contractual demands, tenant count/size, data sensitivity and budget; often a hybrid: shared pool for small tenants and dedicated DB for large/regulated ones, using the same code with tenant-to-connection routing. Always design tenant context as first-class from the start, since retrofitting is painful. Residual risk: whichever model, application bugs can cross tenants unless controls exist at multiple layers.
**Follow-up traps**: "Thousands of tenants with DB-per-tenant?" (elastic pools, catalog/shard map; management overhead). "Customer demands own keys?" (isolation per DB with CMK per tenant).

### 10.2 [Follow-up] How do RLS and EF Core query filters work together, and what are their limits?
**What the interviewer is really testing**: Defence in depth at data layer, with awareness of bypasses.
**Typical mid-level answer**: "Use a global filter." Falls short: single layer.
**Strong senior answer**: EF global filters scope LINQ queries so developers don't repeat `Where(TenantId)`, but they're an app-layer convenience: `IgnoreQueryFilters`, raw SQL, `FromSql`, stored procedures, Dapper, or a different service reading the DB bypass them. RLS enforces in the database for all access paths: define a predicate function comparing `TenantId` to `SESSION_CONTEXT(N'TenantId')`, a security policy with FILTER and BLOCK predicates, and set the session context when opening the connection (e.g. via a connection interceptor). Key risks: forgetting to set/reset context with pooled connections (set it every time connection is opened; use `@read_only` where appropriate), privileged logins that bypass or can alter policies (db_owner can disable RLS), and performance (predicate must be inline and index-supported). Together, a bug in either layer alone doesn't leak. Residual risk: privileged DB access and side channels (error messages/inference) - review Microsoft guidance.
**Follow-up traps**: "Does RLS protect against db_owner?" (no). "Session context leaking across pooled connections?" (set on every open).
```sql
CREATE SECURITY POLICY TenantPolicy
ADD FILTER PREDICATE dbo.fn_tenant(TenantId) ON dbo.Orders,
ADD BLOCK  PREDICATE dbo.fn_tenant(TenantId) ON dbo.Orders AFTER INSERT;
```

### 10.3 [Medium] Beyond the database, where else does tenant data leak?
**What the interviewer is really testing**: Holistic view of data planes.
**Typical mid-level answer**: "Check the queries." Falls short.
**Strong senior answer**: Leaks happen wherever data is keyed or shared: in-memory and distributed caches without tenant in the key (classic: `cache["settings"]`), static/singleton state holding request data, background jobs and message handlers that don't carry/restore tenant context, blob containers/paths and SAS tokens with too-broad scope, search indexes without mandatory tenant filter, exports/reports, logs and telemetry containing tenant data visible across support teams, shared output caching, and error messages. Controls: a `ITenantContext` required by infrastructure abstractions (cache wrapper that prefixes tenant, blob service that roots paths), tenant in message envelopes and validated by consumers, tenant-aware logging scopes, analyzers banning raw `IMemoryCache` outside the wrapper. Trade-off: abstraction overhead. Residual risk: third-party libraries storing state.
**Follow-up traps**: "Where does tenant ID come from?" (validated token claim, never client-supplied header). "Output caching with auth?" (vary by tenant/user or avoid).

### 10.4 [Hard] How do you test for tenant isolation so regressions get caught?
**What the interviewer is really testing**: Verification strategy.
**Typical mid-level answer**: "Manual testing with two tenants." Falls short.
**Strong senior answer**: Make isolation a continuously verified property. Integration tests seed two tenants with distinct data and run every endpoint as tenant A requesting tenant B's IDs, lists, searches, exports, and bulk operations, asserting 404/empty; generate this matrix from the endpoint inventory so new endpoints are covered automatically. Data-layer tests assert that EF queries on tenant-owned entities include the filter (reflection over the model: every entity implementing `ITenantOwned` has a filter) and RLS policies exist for each such table (schema test querying `sys.security_predicates`). Test background jobs and cache keys with simulated tenant switch. In production, add canary checks and anomaly alerts on cross-tenant access patterns, plus audit logs recording tenant of the actor vs the object. Trade-off: test time and fixtures; keep runs in CI. Residual risk: combinatorial paths missed; supplement with periodic pen-test.
**Follow-up traps**: "New table without TenantId?" (test fails the build). "Super-admin cross-tenant?" (explicit, audited path).

### 10.5 [Scenario] A customer reports seeing another tenant's record. First hour: what do you do?
**What the interviewer is really testing**: Incident handling under uncertainty.
**Typical mid-level answer**: "Find the bug and fix." Falls short: skips containment, evidence and notification.
**Strong senior answer**: Treat as a potential data breach. Contain: acquire details (URL, time, IDs, tenant IDs) without further exposing data; if the cause is unknown and the leak is reproducible, consider feature flag or disabling the endpoint/blocking the path (availability traded for confidentiality). Preserve evidence: logs, request traces, deployment history. Scope: query audit/logs for requests where actor tenant != object tenant across the suspected window, identify all affected tenants and data. Find root cause: recent deploys, caches, a missing filter, RLS disabled, a background job, shared state. Fix and add the regression test. Engage legal/DPO early since GDPR-style breach notification clocks (72 hours to authority where applicable - verify with legal) may start at awareness. Communicate with affected customers per policy. Residual risk: undiscovered leak paths; follow with a wider review and post-incident actions.
**Follow-up traps**: "Can't tell who saw what?" (audit logging gaps become action items). "Hotfix straight to prod?" (with review and tests, but containment first).

**Red flags in answers**
- Tenant ID taken from request header/body without validation against the token.
- Relying only on a global query filter.
- Caches, blob paths and background jobs ignored.

**Cheat sheet**
- Models: shared schema -> schema/DB per tenant -> stamp; pick by risk and cost.
- Two layers: EF filter + RLS (SESSION_CONTEXT, FILTER+BLOCK).
- Tenant context derives from validated identity.
- Tenant-key every cache, path, index, job message.
- Test with a generated two-tenant matrix; schema tests for RLS presence.
- Leak = incident: contain, scope, preserve evidence, notify.

---

## Topic 11: Audit Logging and Non-Repudiation

**Study notes**
- **Audit log** = who did what, to which object, when, from where, outcome - distinct from diagnostic logs. Required for accountability, forensics and compliance.
- **Non-repudiation**: ability to prove an actor performed an action; needs strong authentication, trustworthy time, integrity protection (append-only / hash chaining / WORM storage), and restricted write/delete rights.
- Azure: Activity Log (control plane), resource diagnostic logs, Entra sign-in/audit logs, SQL Auditing to Log Analytics/Storage, Blob immutability policies (WORM), Log Analytics retention.
- Never log secrets, tokens, full card numbers, or unnecessary PII.
- Correlation IDs / `traceparent` link events across services.

```
 Service --(audit event: actor, action, resource, tenant, result, correlationId, time)--> Audit pipeline
   --> Append-only store (immutable blob / ledger table) --> SIEM (Sentinel) alerts
 Writers: service identity can ONLY append; no update/delete rights
```

### 11.1 [Easy] How is audit logging different from application logging?
**What the interviewer is really testing**: Purpose-driven design.
**Typical mid-level answer**: "Audit logs record user actions." Falls short.
**Strong senior answer**: Application logs help engineers debug; they're verbose, sampled, rotated and can be edited or lost. Audit logs are a compliance and security record: structured, complete for defined events (authn events, authorization failures, privilege changes, access to sensitive records, data exports, config changes, admin actions), with actor identity, target, outcome, timestamp and correlation ID. They need guaranteed delivery (not sampled), tamper resistance, defined retention, restricted read access, and often separate storage and ownership. I define the audit event catalogue early with security and compliance, so "what must be auditable" is not decided during an incident. Trade-off: volume and cost, and PII in audit data itself - log identifiers rather than payloads. Residual risk: events not in the catalogue are invisible.
**Follow-up traps**: "Use Serilog warnings as audit?" (no; sampling/levels). "Log request bodies?" (generally no).

### 11.2 [Follow-up] How do you make audit logs tamper-evident?
**What the interviewer is really testing**: Integrity design.
**Typical mid-level answer**: "Write to a database table." Falls short.
**Strong senior answer**: Threat: an attacker or insider covering tracks. Controls: separate the audit store from the application's trust domain; the writer identity has append-only permission; store in immutable (WORM) blob storage with time-based retention/legal hold, or SQL ledger tables (append-only with cryptographic verification - verify features in current docs); forward copies to a SIEM in a different subscription/tenant; optional hash chaining or periodic signed digests stored elsewhere to detect deletion. Protect time with trusted sources (use server time, not client). Monitor for gaps in sequence numbers or sudden volume drop, since silence can mean tampering. Access to read audit is itself audited. Trade-off: immutability prevents correction of mistakes and complicates erasure obligations - hence keep PII out. Residual risk: compromise of the audit pipeline before write; privileged cloud admin.
**Follow-up traps**: "GDPR erasure vs immutable?" (store IDs not PII; legal retention exception). "Who can delete?" (nobody within retention).

### 11.3 [Medium] What do you log for sensitive-data access, and what do you avoid logging?
**What the interviewer is really testing**: Balancing accountability and privacy.
**Typical mid-level answer**: "Log everything." Falls short: creates a second sensitive store.
**Strong senior answer**: Log who accessed which record (IDs), purpose/endpoint, tenant, result, source IP/client, and correlation; for bulk reads, record counts and filters. Avoid secrets, tokens, passwords, full PANs, health details, and free-text bodies; mask or hash identifiers where needed. Use structured logging with allow-listed fields, and a destructuring policy preventing entire objects being logged. Add log-scrubbing tests (e.g. send fake SSN in a request and assert it doesn't appear). Access to logs restricted by role; retention aligned to policy. For healthcare/financial contexts, read-access auditing may be mandatory (e.g. HIPAA access tracking) - check requirements. Trade-off: less debugging detail; compensate with trace IDs and the ability to reproduce in test. Residual risk: logs from third-party libraries or exception messages leaking data.
**Follow-up traps**: "Exceptions with SQL parameters?" (sensitive data logging off in EF). "Log retention?" (as policy requires, not infinite).

### 11.4 [Hard] Design the audit trail for an approval workflow where "I never approved that" is a risk.
**What the interviewer is really testing**: Non-repudiation chain end-to-end.
**Typical mid-level answer**: "Store user ID and timestamp." Falls short: weak evidence.
**Strong senior answer**: Non-repudiation requires proof of identity and intent. Identity: strong authentication (MFA/phishing-resistant) bound to the approval action, with step-up auth for high-value approvals; the audit event stores the immutable user ID (`oid`), authentication method/assurance, session/token ID, IP/device, and timestamp. Intent and content: record exactly what was approved - a hash of the document/version and amounts shown, so later change is detectable. For legal-grade evidence, use digital signatures (user or service-held keys in Key Vault/HSM) over the approval payload. Storage: immutable store as above, with sequence numbers. Process: no shared accounts, no impersonation without explicit audited flag (delegated approvals recorded as such). Trade-off: friction for users and key management cost. Residual risk: credential theft (account takeover) which undermines identity; mitigate with phishing-resistant MFA and anomaly checks.
**Follow-up traps**: "Admin approving on behalf?" (record both actor and delegator). "Time source?" (server-side, synchronised).

### 11.5 [Tricky] You have audit logs but nobody reads them. Is that okay?
**What the interviewer is really testing**: Detection vs recording.
**Typical mid-level answer**: "Yes, they're there for audits." Falls short.
**Strong senior answer**: Logs unread are evidence after the fact, not security. Define detections for the highest-risk behaviours: impossible-travel sign-ins, privilege assignments, mass reads/exports, failed-authz spikes, access outside normal hours, use of break-glass, secrets accessed by unexpected identities. Route to Sentinel/Log Analytics alert rules with owners and runbooks, tune to reduce noise, and measure mean time to detect. Review periodically (access reviews, sampling). Start small with 5-10 high-value alerts rather than a thousand. Trade-off: cost of ingestion and on-call load. Residual risk: attackers avoiding detection thresholds; improve via threat modelling and red-team exercises.
**Follow-up traps**: "Retention cost?" (hot/cold tiers, archive). "Who owns alerts?" (named team on-call).

**Red flags in answers**
- Audit data stored in the same writable DB with the app's full rights.
- Logging whole request bodies/tokens "for debugging".
- No defined event catalogue or alerting.

**Cheat sheet**
- Audit = who/what/which/when/where/outcome, separate from diagnostics.
- Append-only + WORM/ledger + SIEM copy.
- Log IDs not PII; never secrets/tokens.
- Non-repudiation = strong authn + integrity + trusted time (+ signatures).
- Detect, don't just record: a few good alerts.
- Silence in logs can be an indicator.

---

## Topic 12: Supply Chain Security and Secure SDLC

**Study notes**
- **NuGet risks**: typosquatting, dependency confusion (internal package names on public feed), malicious or compromised maintainers, vulnerable transitive deps, abandoned packages.
- Controls: private feed/upstream with package source mapping, `packages.lock.json` + `RestoreLockedMode`, **Central Package Management**, `dotnet list package --vulnerable --include-transitive`, NuGet Audit (built into recent SDKs - verify in current docs), Dependabot/Renovate, package signature verification.
- **SBOM** (CycloneDX/SPDX), **build provenance** (SLSA, GitHub artifact attestations - verify in current docs), reproducible/pinned builds, signed artifacts, container image scanning (Defender for Containers / Trivy), minimal base images, pin by digest.
- **Secure SDLC**: design review/threat model -> secure coding standards -> code review -> SAST (CodeQL, Roslyn analyzers) -> SCA -> secret scanning -> DAST -> pen-test -> vulnerability management with SLAs.

```
 Dev --PR(review, SAST, secret scan, dep review)--> Build (pinned deps, lock file, SBOM, sign, provenance)
   --> Registry (image scan, signed) --> Deploy gate (policy: only signed, scanned artifacts)
```

### 12.1 [Easy] What supply-chain risks exist when you add a NuGet package?
**What the interviewer is really testing**: Awareness that dependencies are code you run with your privileges.
**Typical mid-level answer**: "It might have vulnerabilities." Falls short: ignores malicious packages and transitive risk.
**Strong senior answer**: The package runs with the app's permissions at build and runtime (including MSBuild targets and install scripts in build tooling), so it extends my trusted computing base. Risks: known vulnerabilities (direct and transitive), typosquatting and dependency confusion, a compromised maintainer or release, abandonment, licence problems. Before adding, I check popularity and maintenance, ownership/signing, need versus writing 20 lines, and the size of its dependency tree. Controls: private feed with source mapping so internal names can't resolve from nuget.org, lock files, vulnerability audit in CI, a small curated approved list for sensitive services, and renovation via Dependabot with review. Trade-off: update friction vs staleness. Residual risk: zero-day in a trusted package; mitigate with least-privilege runtime and egress control.
**Follow-up traps**: "Dependency confusion fix?" (package source mapping/ID prefix reservation). "Transitive vulnerable dep?" (override via central package versions).

### 12.2 [Follow-up] What is an SBOM and what do you do with it?
**What the interviewer is really testing**: Practical use rather than checkbox.
**Typical mid-level answer**: "A list of components." Falls short.
**Strong senior answer**: A Software Bill of Materials inventories components and versions (CycloneDX or SPDX), generated in CI per build and tied to the artifact digest. Its value is speed during an incident: when a new CVE like a Log4Shell-class issue appears, I query SBOMs to find which deployed services include the affected version within minutes rather than guessing. It also supports licence compliance and customer/regulator requests. To be useful it must be accurate (include transitive and container layers), stored centrally (e.g. Dependency-Track), and linked to vulnerability feeds. SBOM doesn't tell you if the vulnerable code is reachable, so pair with VEX statements or reachability analysis for triage. Trade-off: generating/storing overhead; modest. Residual risk: unlisted vendored/native components.
**Follow-up traps**: "SBOM = secure?" (no, inventory only). "Per build or per release?" (per build, retained for deployed versions).

### 12.3 [Medium] How do you make builds trustworthy: provenance, signing, pinning?
**What the interviewer is really testing**: Understanding pipeline as attack surface (SolarWinds-style).
**Typical mid-level answer**: "Use a secure pipeline." Falls short.
**Strong senior answer**: Attackers target the build: stolen CI tokens, poisoned actions, modified dependencies. Controls: pin third-party GitHub Actions/pipeline tasks to commit SHAs, use lock files and restore in locked mode, pin container base images by digest, run builds on ephemeral isolated agents, least-privilege CI tokens with OIDC federation to Azure (no long-lived secrets), protected branches with required reviews and status checks, separate build and deploy permissions, generate provenance attestation (SLSA) and sign artifacts/images (Sigstore/cosign or Notary), and have the deployment environment verify signatures/policies *(verify tool support in current docs)*. Trade-off: slower dependency updates (offset by Dependabot PRs) and pipeline maintenance. Residual risk: compromised maintainer of a pinned action version, or insider with merge rights; require two reviewers on pipeline definition changes.
**Follow-up traps**: "Pinned version also gets stale?" (automate updates). "PR from fork running with secrets?" (don't expose secrets to untrusted PRs).

### 12.4 [Hard] SAST/DAST/secret scanning produce thousands of alerts. How do you triage and keep developers engaged?
**What the interviewer is really testing**: Programme management and judgement.
**Typical mid-level answer**: "Fix highs first." Falls short.
**Strong senior answer**: Start by baseline: accept existing findings as a backlog and block only *new* high-confidence/high-severity findings in PRs, so the signal stays meaningful. Triage by exploitability in context: internet-facing, reachable, handles sensitive data, available fix. Classify as true/false positive with suppression requiring rationale and expiry, so suppressions are reviewed. Set SLAs by severity and exposure (e.g. critical internet-facing days, others weeks), assign to owning teams, and report trends not raw counts. Tune rulesets and remove noisy rules; track false-positive rate as a quality metric of the tooling. For secrets, any real secret is revoked immediately regardless of repo age. Pen-test findings: reproduce, root-cause to a class, fix the class in the platform if possible, retest. Trade-off: effort spent on tuning rather than finding. Residual risk: logic flaws untouched by tools.
**Follow-up traps**: "Developers ignore tool?" (fix noise, integrate into PR, show value). "Wontfix?" (documented risk acceptance).

### 12.5 [Tricky] A critical CVE drops in a widely used library on Friday. Walk through your response.
**What the interviewer is really testing**: Calm, systematic vulnerability response.
**Typical mid-level answer**: "Upgrade the library." Falls short.
**Strong senior answer**: Step 1, determine exposure: use SBOMs/dependency graphs to find affected services and whether the vulnerable code path is reachable and internet-exposed. Step 2, prioritise: exploited in the wild (check CISA KEV and vendor advisories), internet-facing and unauthenticated first. Step 3, mitigate immediately where the patch isn't ready: WAF virtual patch, feature disable, network restriction, config changes. Step 4, patch via the normal pipeline with tests (an emergency path shouldn't skip tests entirely), deploy in priority order, and verify with a scan. Step 5, hunt for compromise indicators in logs during the exposure window; rotate secrets if exploitation is plausible. Communicate status to stakeholders. Afterwards, review how long detection and patching took. Residual risk: unknown copies (vendored, containers); inventory improvements.
**Follow-up traps**: "No patch available?" (compensating controls, isolate). "Is it exploitable here?" (reachability analysis, don't assume).

**Red flags in answers**
- Unpinned `*` versions and no lock files in production builds.
- "We use Dependabot" as the entire supply-chain story.
- Treating scanner count as the success metric.

**Cheat sheet**
- Packages run with your privileges: vet, pin, audit.
- Dependency confusion: package source mapping + private feed.
- SBOM = fast incident scoping; add provenance and signing.
- Pin Actions by SHA; images by digest; OIDC instead of CI secrets.
- Block new high-confidence findings; baseline the rest.
- CVE drill: exposure -> mitigate -> patch -> hunt -> learn.

---

## Topic 13: API Security Controls

**Study notes**
- **Rate limiting**: ASP.NET Core built-in rate limiting middleware (fixed/sliding window, token bucket, concurrency); per-instance by default; use gateway (APIM/Front Door) or distributed counters for global limits.
- **Input validation**: allow-list, validate at the boundary (type, length, range, format), parameterised queries, encode on output. Validation is not a substitute for authorization.
- **CORS**: a *browser* relaxation of same-origin policy, not an access control; never `AllowAnyOrigin` with credentials.
- **CSRF**: matters for cookie-authenticated apps; defences: SameSite cookies, antiforgery tokens, custom header checks. Bearer-token APIs (token in header) are not CSRF-prone by the browser sending credentials automatically.
- **File upload**: size caps, extension + content-type + magic-byte checks, store outside web root/in blob with random names, malware scan, serve with `Content-Disposition` and separate domain, no execution.
- **Webhooks**: HMAC signature over raw body with constant-time compare, timestamp tolerance, replay protection (nonce/event ID dedupe), HTTPS, secret rotation.

```
 Inbound webhook: verify HMAC(secret, timestamp + "." + rawBody) == header sig (constant-time)
                  |timestamp - now| < 5 min ; eventId not seen before (store with TTL) ; THEN process, idempotently
```

### 13.1 [Easy] Is CORS a security control for your API? Explain.
**What the interviewer is really testing**: Common misconception about CORS.
**Typical mid-level answer**: "It blocks unauthorized sites from calling the API." Falls short: it only affects browsers.
**Strong senior answer**: CORS tells browsers which origins may read cross-origin responses; it doesn't stop curl, servers, or malicious scripts outside a browser from calling the API, so authentication and authorization still enforce access. What it does protect is a user's browser from letting a malicious site read authenticated responses. Configure an explicit origin allow-list, specific methods and headers, and avoid `AllowAnyOrigin` combined with credentials (which browsers block anyway) or reflecting the `Origin` header blindly. Remember preflights are not protection for "simple" requests that can still cause state change on cookie-authenticated APIs, which is where CSRF defences matter. Trade-off: managing origins across environments; keep per-env config. Residual risk: allowed origins that suffer XSS become a bridge.
**Follow-up traps**: "Wildcard subdomain?" (risky if any subdomain is takeover-able). "CORS error means blocked by server?" (the request often reached the server).

### 13.2 [Follow-up] When do you need CSRF protection and how do you do it in .NET?
**What the interviewer is really testing**: Understanding ambient authority.
**Typical mid-level answer**: "Add antiforgery tokens everywhere." Falls short.
**Strong senior answer**: CSRF abuses credentials the browser attaches automatically (cookies, Windows auth). If the app authenticates via cookies, state-changing requests from a malicious site could succeed. Defences layered: `SameSite=Lax/Strict` cookies (Lax is default in modern browsers but don't depend on it alone), antiforgery tokens (ASP.NET Core `[ValidateAntiForgeryToken]`/auto validation, double-submit or synchronizer token pattern for SPAs), requiring a custom header (e.g. `X-Requested-With`) with CORS preflight, verifying Origin/Referer, and never doing state changes on GET. APIs that read the bearer token from an `Authorization` header set by JavaScript aren't subject to CSRF, but they have XSS-theft risks (see BFF topic). Trade-off: SPA token plumbing. Residual risk: XSS on the same origin defeats CSRF tokens.
**Follow-up traps**: "SameSite enough?" (same-site subdomain attacks; defence in depth). "JSON content type protects?" (not reliably).

### 13.3 [Medium] Secure file upload design.
**What the interviewer is really testing**: Anticipating multiple abuse paths.
**Typical mid-level answer**: "Check the extension." Falls short.
**Strong senior answer**: Threats: malware delivery, web shell upload, path traversal via file names, zip bombs, resource exhaustion, XSS via SVG/HTML, and parser exploits in image/PDF libraries. Controls: enforce size and count limits at the edge and in Kestrel, authenticate and authorise the uploader, allow-list types by validating magic bytes and not trusting the client `Content-Type`, generate server-side random names and never use user paths, store in Blob (not the web server disk) with no execute permission and private access, scan with malware scanning (e.g. Defender for Storage - verify in current docs), process files (thumbnails, conversions) in an isolated low-privilege worker, serve from a separate cookieless domain with `Content-Disposition: attachment` and `X-Content-Type-Options: nosniff`, and use short-lived SAS URLs. Trade-off: latency of scanning (async with quarantine state). Residual risk: zero-day parser flaws; isolation limits blast radius.
**Follow-up traps**: "Direct-to-blob upload via SAS?" (limit scope/size/expiry, still scan afterwards). "Allow SVG?" (sanitise or block).

### 13.4 [Hard] Design webhook receiving and sending securely, including replay protection.
**What the interviewer is really testing**: Integrity, authenticity and idempotency together.
**Typical mid-level answer**: "Use a secret token in the URL." Falls short.
**Strong senior answer**: For receiving: verify an HMAC signature computed over the raw body (read before model binding; parsing then re-serialising breaks the hash) using a per-sender secret, constant-time comparison, a timestamp inside the signed payload with a short tolerance to limit replay, and an event ID stored in a dedupe cache/table for the retention window so duplicates are ignored. Respond quickly (202) and process asynchronously via a queue with idempotent handlers since providers retry. Allow-list sender IPs only as an extra layer (they change). Secrets per integration in Key Vault with rotation supporting two active secrets. For sending webhooks: sign payloads, let customers verify, retry with backoff, SSRF-protect the destination URL (block internal ranges), limit payload data, and rate-limit per subscriber. Trade-off: complexity of dedupe storage and clock skew. Residual risk: secret leak at the sender/receiver; rotation and alerts.
**Follow-up traps**: "Why constant-time compare?" (timing side channels). "Replays within the tolerance window?" (event ID dedupe).

### 13.5 [Tricky] Input validation vs output encoding vs parameterised queries: which one stops injection?
**What the interviewer is really testing**: Context-specific defences.
**Typical mid-level answer**: "Validate inputs." Falls short.
**Strong senior answer**: Injection happens when data is interpreted as code in a particular context, so the primary control is context-appropriate separation: parameterised queries/stored procedures with parameters for SQL (EF/Dapper parameters, never string concatenation or `FromSqlRaw` with interpolated user data), output encoding for HTML (Razor encodes by default; careful with `Html.Raw`), proper escaping for shell/LDAP/CSV contexts, and avoiding command invocation with user input. Input validation (allow-list for type/length/format) reduces attack surface and enforces business rules but can't reliably cover every downstream context (a legitimate name like O'Brien must be accepted). Add CSP to mitigate XSS impact and least-privileged DB accounts to limit SQL injection impact. Trade-off: dynamic SQL needs (sorting columns) use allow-listed identifiers. Residual risk: second-order injection where stored data is later used unsafely.
**Follow-up traps**: "Dynamic ORDER BY?" (map to allow-list). "Sanitising by removing quotes?" (fragile blocklist).

**Red flags in answers**
- "CORS protects our API."
- HMAC computed on re-serialised JSON, or compared with `==`.
- Validating only on the client side.

**Cheat sheet**
- Rate limit: edge + app; per identity and per IP; 429 + Retry-After.
- CORS is a browser rule; CSRF is for cookie auth.
- Uploads: magic bytes, random names, private blob, scan, isolate processing.
- Webhooks: HMAC raw body, timestamp, event-ID dedupe, async, idempotent.
- Injection: parameterise + encode by context; validate as allow-list.
- Validate at the boundary and again at the consumer of untrusted data.

---

## Topic 14: Frontend / BFF Token Handling

**Study notes**
- **SPA with tokens in browser** (localStorage/sessionStorage): vulnerable to theft through XSS. In-memory tokens reduce persistence but not XSS.
- **BFF (Backend-for-Frontend)**: server-side component holds tokens, issues an `HttpOnly; Secure; SameSite` session cookie to the browser, proxies API calls; needs CSRF protection. Current IETF guidance for browser-based apps recommends considering BFF for high-security cases *(verify in current docs)*.
- **Authorization Code + PKCE** for public clients (SPA, mobile); implicit flow is deprecated.
- **Refresh token rotation**: each use issues a new refresh token, old becomes invalid; reuse detection revokes the family. Short-lived access tokens.
- Mobile: use system browser (AppAuth pattern), store refresh tokens in Keychain/Keystore, not embedded webviews.

```
 Browser --cookie(HttpOnly,SameSite)--> BFF (confidential client; holds access+refresh tokens) --Bearer--> APIs
 Browser never sees tokens -> XSS can ride the session but cannot exfiltrate long-lived tokens
```

### 14.1 [Easy] Where should a SPA store its tokens?
**What the interviewer is really testing**: Knowing trade-offs, no dogma.
**Typical mid-level answer**: "localStorage." or "HttpOnly cookies are always better." Falls short.
**Strong senior answer**: Any JavaScript-readable storage (localStorage, sessionStorage) is exposed to XSS exfiltration; in-memory is a bit better (no persistence, harder to steal after the fact) but still reachable by injected script. HttpOnly cookies hide the token from script but introduce CSRF concerns and ambient authority, solved with SameSite and anti-CSRF tokens. The strongest pattern for sensitive apps is a BFF: the browser holds only a session cookie; tokens stay server-side. For lower-risk apps using MSAL.js with auth code + PKCE and short-lived access tokens, accepting the XSS risk with strong CSP and dependency hygiene can be reasonable. The decision depends on data sensitivity and team capability. Residual risk: XSS can still perform actions as the user via the session even with BFF, so XSS prevention (CSP, encoding, Trusted Types) remains essential.
**Follow-up traps**: "HttpOnly solves XSS?" (no; it stops cookie theft, not request forgery by script). "Silent refresh via iframe?" (third-party cookie restrictions; verify current browser behaviour).

### 14.2 [Follow-up] Explain PKCE and why SPAs and mobile apps need it.
**What the interviewer is really testing**: Understanding the threat, not just the acronym.
**Typical mid-level answer**: "It's an extra security parameter." Falls short.
**Strong senior answer**: Public clients can't keep a client secret, so a stolen authorization code could be redeemed by an attacker (e.g. a malicious app registered for the same custom URI scheme intercepting the redirect). PKCE binds the code to the client instance: the client generates a random `code_verifier`, sends its hash (`code_challenge`) in the authorize request, and presents the verifier at the token endpoint; the IdP checks they match, so an intercepted code is useless without the verifier. Use S256, not plain, and validate `state` (CSRF on the flow) and `nonce` for ID tokens. PKCE is also recommended for confidential clients now (OAuth 2.1 direction - verify in current docs). Trade-off: none significant. Residual risk: compromise of the client itself, or phishing of credentials at the IdP.
**Follow-up traps**: "Why not implicit flow?" (tokens in URL fragment, leakage, no PKCE binding). "State vs nonce?" (state for request/response correlation and CSRF, nonce for ID token replay).

### 14.3 [Medium] How does refresh token rotation work and what's the failure mode?
**What the interviewer is really testing**: Session lifecycle design.
**Typical mid-level answer**: "Get a new access token with the refresh token." Falls short.
**Strong senior answer**: Long-lived refresh tokens are valuable targets, especially in browsers/mobile. Rotation issues a new refresh token on every use and invalidates the previous; if an old token is presented again, the IdP infers theft or duplication and revokes the whole token family, forcing re-authentication. Pair with absolute lifetime, idle timeout, binding to client/device where supported, and revocation on logout/password change. Failure mode: legitimate races (two tabs refreshing simultaneously, network retry after a lost response) trigger false revocations; mitigate with a short reuse grace window or serialising refresh in one place (BFF or a single tab-coordination). Access tokens remain short (minutes) to limit exposure. Trade-off: complexity vs reduced theft window. Residual risk: theft and use before the legitimate client's next refresh.
**Follow-up traps**: "Offline access for mobile?" (secure storage plus rotation). "Entra specifics?" (verify refresh token behaviour in current docs).

### 14.4 [Hard] You're asked to migrate a SPA from localStorage tokens to a BFF. What changes and what do you watch?
**What the interviewer is really testing**: Practical migration reasoning.
**Typical mid-level answer**: "Add a backend that proxies calls." Falls short.
**Strong senior answer**: Assets: access/refresh tokens and user session. Add an ASP.NET Core BFF (YARP or Duende BFF/similar library) configured as a confidential OIDC client using auth code + PKCE, storing tokens server-side (encrypted cookie ticket or server session store like Redis) and exposing `/login`, `/logout`, `/user` and a reverse-proxy `/api/*` that attaches the access token. Browser cookie: `HttpOnly; Secure; SameSite=Lax/Strict`, `__Host-` prefix, same-site deployment with the BFF on the same origin as the SPA. Add CSRF protection (custom header required on proxied calls, plus SameSite). Handle token refresh and expiry server-side, session scaling (distributed cache, data protection keys shared across instances), logout (front- and back-channel), and CORS no longer needed if same origin. Trade-off: new stateful component to scale and secure, plus latency. Residual risk: XSS acting through the session, BFF compromise holding tokens; keep it hardened and minimal.
**Follow-up traps**: "Scaling cookie size?" (store tokens in server-side session, cookie holds ID). "Multiple APIs?" (BFF obtains per-API tokens).

### 14.5 [Tricky] Mobile app calls the API directly. How do you stop someone from extracting a secret or abusing the API?
**What the interviewer is really testing**: Client trust boundaries.
**Typical mid-level answer**: "Obfuscate the key in the app." Falls short.
**Strong senior answer**: Anything shipped in a client binary is public; no embedded secrets provide security. Use user authentication via auth code + PKCE with the system browser, store refresh tokens in platform secure storage (Keychain/Keystore), keep access tokens short-lived, and enforce authorization server-side per user. For abuse by modified clients/bots, add layered friction: per-user rate limits, device/app attestation (Play Integrity, App Attest - verify in current docs) as a signal not as a guarantee, certificate pinning with care for rotation, anomaly detection. Never rely on client-side checks for business rules or entitlement. Trade-off: attestation and pinning add operational risk (lockouts when keys rotate). Residual risk: determined attackers on rooted devices; accept and monitor at server side.
**Follow-up traps**: "API key identifies the app?" (it's an identifier, not a secret). "Pinning risks?" (outage on cert rotation; pin to key/backup pins).

**Red flags in answers**
- Using the implicit flow or putting tokens in URLs.
- Believing HttpOnly eliminates XSS impact.
- Embedding a "client secret" in a SPA or mobile binary.

**Cheat sheet**
- SPA tokens in JS-readable storage = XSS-stealable.
- BFF: tokens server-side, `__Host-` HttpOnly SameSite cookie, CSRF header.
- Auth code + PKCE (S256), state and nonce.
- Refresh rotation + reuse detection + short access tokens.
- Mobile: system browser, Keychain/Keystore, attestation as signal only.
- XSS prevention (CSP, encoding) still essential.

---

## Topic 15: Security Incident Response

**Study notes**
- Phases (NIST-style): **Prepare -> Detect/Analyse -> Contain -> Eradicate -> Recover -> Post-incident review**.
- Containment priority: stop ongoing harm first, preserve evidence second, root-cause third; don't destroy evidence (snapshot VMs/disks, export logs before cleanup).
- Leaked key/compromised account: **revoke/rotate first**, then investigate usage during the exposure window; assume exfiltration if usable.
- Disclosure: legal/DPO decide notification; regulators (e.g. GDPR 72 hours from becoming aware where applicable - verify with legal), customers, contractual obligations.
- Blameless post-mortem produces action items with owners.

```
 Detect -> Triage severity -> Incident commander + comms lead + scribe
   -> Contain (revoke, isolate) -> Preserve evidence -> Scope (logs: what did the credential do?)
   -> Eradicate/Recover (rotate, rebuild, patch) -> Notify (legal) -> Post-mortem -> Fix class of issue
```

### 15.1 [Easy] What are the first steps when you suspect an account or key is compromised?
**What the interviewer is really testing**: Order of operations under pressure.
**Typical mid-level answer**: "Change the password." Falls short: no scope, evidence, or sessions.
**Strong senior answer**: Contain first: disable or revoke the credential, kill active sessions/refresh tokens (revoke sign-in sessions in Entra), and block suspicious IPs if useful; but snapshot relevant logs and state before changes so evidence is not lost. Then scope: what could this identity access, and what did it actually do from the earliest suspected exposure time (sign-in logs, Activity Log, storage/SQL audit, Key Vault logs)? Check for persistence: new credentials, app registrations, role assignments, forwarding rules, new keys. Rotate all secrets the identity could read. Declare an incident with roles (commander, comms), and notify legal if personal data may be involved. Trade-off: containment may interrupt service; business decides with data. Residual risk: persistence we didn't find, so monitor closely afterwards.
**Follow-up traps**: "Wipe the VM immediately?" (preserve evidence first unless spreading). "Only reset password?" (existing tokens remain valid until revoked).

### 15.2 [Follow-up] An AWS/Azure key with broad rights was in a public repo for 2 hours. What do you assume and do?
**What the interviewer is really testing**: Assume-compromise mindset.
**Typical mid-level answer**: "Delete the commit." Falls short.
**Strong senior answer**: Assume it was harvested: bots scan public commits within minutes. Deleting the commit or rewriting history doesn't help (forks, caches, clones). Revoke the key immediately, then review everything it did since the commit time: unusual resource creation (crypto-mining VMs), data access, new identities/keys, role changes, network changes. Use cloud provider logs and billing anomalies. Clean up attacker-created resources and persistence, rotate dependent secrets, and consider whether data left the environment (requires notification analysis). After, address the root cause: enable push protection, pre-commit scanning, reduce reliance on static keys (managed identity/OIDC), add alerts on key use from unknown IPs. Trade-off: speed vs complete investigation; revoke first. Residual risk: data already copied; provider-specific abuse windows.
**Follow-up traps**: "Rewrite git history?" (cosmetic after revocation; still do for hygiene if safe). "How know if used?" (provider access logs by key ID).

### 15.3 [Medium] How do you prepare so that incident response actually works?
**What the interviewer is really testing**: Operational maturity.
**Typical mid-level answer**: "Have a plan document." Falls short.
**Strong senior answer**: Preparation is what makes hour one effective. Maintain an on-call rota with defined severity levels and escalation contacts, runbooks for likely scenarios (leaked secret, compromised account, ransomware, data leak, DDoS), and access arranged in advance (break-glass, log access, ability to revoke). Ensure logging exists before the incident: audit logs centralised with adequate retention, time-synced, and queryable. Practise with tabletop exercises and game days, including comms and legal paths, and measure time to contain. Keep an asset and data-flow inventory and secret inventory so scoping is possible. Pre-agree who decides on customer notification. Trade-off: time invested vs feature work; justified by risk. Residual risk: novel incident types; keep generic skills and clear authority to act.
**Follow-up traps**: "No logs available?" (a finding; fix as action). "Who is incident commander?" (trained role, not necessarily the most senior).

### 15.4 [Hard] How do you decide whether to disclose and to whom?
**What the interviewer is really testing**: Maturity around legal/ethical obligations.
**Typical mid-level answer**: "Tell customers if data was leaked." Falls short.
**Strong senior answer**: Disclosure is a decision made with legal/DPO, security and leadership, based on facts: what data, whose, whether it was accessed vs merely exposed, whether it was encrypted with keys uncompromised, and the likelihood of harm. Obligations vary: regulators (GDPR 72-hour notification to authority when risk to individuals; notification to individuals when high risk - verify with counsel), sector rules (HIPAA breach notification, PCI card-brand rules), and contractual commitments with customers and partners. Engineers' job is to provide accurate scoping evidence and timelines, state uncertainty explicitly, and avoid speculation. Communicate promptly, factually, with actionable advice (rotate passwords). Document the decision rationale. Trade-off: early notification with incomplete facts vs delay; regulators typically allow phased reports. Residual risk: later-discovered scope expansion requires updates.
**Follow-up traps**: "Is a breach always reportable?" (no; risk assessment applies). "Engineers talking to press?" (no; comms lead).

### 15.5 [Tricky] During the incident the exec wants to "just restore service" and skip forensics. Response?
**What the interviewer is really testing**: Balancing business pressure with security needs.
**Typical mid-level answer**: "Insist on forensics." Falls short: ignores business impact.
**Strong senior answer**: Frame as a joint risk decision. Restoring without understanding the entry point risks immediate re-compromise and data loss repeating. Propose a parallel path: restore from known-good clean infrastructure (rebuilt via IaC) while preserving snapshots/logs of the compromised environment for analysis; contain the known vector (revoked credentials, blocked network paths); and agree a timeline for investigation. Provide the executive with clear options and risks (service back in X hours with residual risk Y vs Z hours with confidence). If regulatory obligations require evidence retention, make that explicit. Trade-off: speed vs assurance; the decision is theirs but informed. Residual risk: attacker persistence; heightened monitoring and credential rotation after restore.
**Follow-up traps**: "Evidence integrity?" (chain of custody, snapshots). "Restore from backup - clean?" (verify backup predates compromise).

**Red flags in answers**
- "Delete the secret from Git history" as the remedy.
- Resetting a password without revoking sessions/tokens.
- No mention of scoping (what did the credential do?) or legal/notification.

**Cheat sheet**
- Contain -> preserve evidence -> scope -> eradicate -> recover -> learn.
- Revoke first; assume harvested if exposed publicly.
- Revoke sessions/refresh tokens, not just passwords.
- Look for persistence: new keys, roles, app registrations.
- Disclosure decisions: legal/DPO with engineering facts.
- Practise: tabletops, runbooks, logs ready before incidents.

---

## Topic 16: Compliance Basics and Security-vs-Velocity Trade-offs

**Study notes**
- **SOC 2**: attestation report by an auditor on controls relating to Trust Services Criteria (Security mandatory; Availability, Confidentiality, Processing Integrity, Privacy optional). Type I = design at a point in time; Type II = operating effectiveness over a period.
- **ISO/IEC 27001**: certifiable ISMS standard - risk-based management system, Statement of Applicability, Annex A controls.
- **PCI DSS**: applies when storing/processing/transmitting cardholder data; scope reduction via hosted fields/redirect and tokenisation (SAQ A vs higher SAQs) - verify current version requirements.
- **HIPAA**: US health data (PHI): Security Rule, Privacy Rule, Breach Notification; BAAs with vendors; Azure offers HIPAA-eligible services under BAA - verify scope.
- Compliance != security, but evidence of controls operating. Engineers support audits with **evidence automation** (IaC, Azure Policy, access reviews, change records, logs).

```
 Control (e.g. "all prod changes are reviewed") -> Evidence (PR approvals export, branch protection config)
 Auditor samples evidence over period (SOC2 Type II)  => automate evidence collection, don't scramble
```

### 16.1 [Easy] How do SOC 2, ISO 27001 and PCI-DSS differ in nature?
**What the interviewer is really testing**: Basic literacy to talk with auditors and customers.
**Typical mid-level answer**: "They're security standards." Falls short.
**Strong senior answer**: SOC 2 is an auditor's attestation report against Trust Services Criteria, commonly demanded by B2B customers; Type II covers operating effectiveness across months. ISO 27001 certifies that an organisation runs an information security management system: risk assessment, treatment, continual improvement, with a Statement of Applicability. PCI DSS is a prescriptive standard mandated by card brands for anyone handling cardholder data, with specific technical requirements (segmentation, encryption, logging, vulnerability management, access control). HIPAA is a US legal framework for PHI. The overlap is large (access control, change management, logging, incident response, vendor management), so I design controls once and map them to multiple frameworks. Trade-off: audit effort; benefit from automation. Residual risk: "checkbox" compliance that doesn't match real risk.
**Follow-up traps**: "SOC 2 Type I vs II?" (point-in-time vs period). "Is Azure PCI compliant so we are?" (shared responsibility; our workload must also meet controls).

### 16.2 [Follow-up] How do engineers support an audit without it becoming a fire drill?
**What the interviewer is really testing**: Practical evidence mindset.
**Typical mid-level answer**: "Gather screenshots when asked." Falls short.
**Strong senior answer**: Make controls produce evidence as a by-product. Infrastructure as code in Git gives change history and approvals; branch protection and required reviews are exportable; Azure Policy and Defender for Cloud show configuration compliance continuously; access reviews run in Entra with recorded decisions; vulnerability and patch SLAs come from scanner reports; incidents have tickets and post-mortems; backups and DR tests are logged. Use a compliance platform or a lightweight evidence repository with owners per control. Know the key controls: least privilege, MFA, change management, logging/monitoring, incident response, backup/restore, vendor risk, onboarding/offboarding. When the auditor samples, engineers answer scoped questions and walk through the system, rather than building new documents. Trade-off: discipline from day one. Residual risk: controls that exist in policy but not in practice - find via internal checks.
**Follow-up traps**: "Emergency change bypassing review?" (documented emergency process with retrospective review). "Offboarding gaps?" (automate access removal).

### 16.3 [Medium] How does PCI scope reduction work, and why does it matter?
**What the interviewer is really testing**: Architecture to avoid risk rather than manage it.
**Typical mid-level answer**: "Use a payment provider." Falls short: no mechanism.
**Strong senior answer**: Every system that stores, processes, transmits card data, plus systems connected to them without segmentation, is in PCI scope, which means heavy controls and audit cost. Best strategy: never let the PAN touch your servers. Use a hosted payment page/redirect or embedded hosted fields/iframes from a PCI-validated provider (Stripe Elements, Adyen, Braintree, etc.), so the browser sends card data directly to the provider and your backend only gets a token. Your SAQ drops to the smallest applicable form (e.g. SAQ A, with requirements about script integrity on the pages that embed the provider - verify with current PCI DSS version guidance on payment page scripts). Store only tokens and last four digits/brand where allowed; never store CVV after authorisation. Segment any remaining systems. Trade-off: provider dependence and less UI control. Residual risk: your page being tampered (Magecart-style script injection) - CSP, SRI, script inventory.
**Follow-up traps**: "Can we log the card number briefly?" (that puts logging in scope). "Iframe vs redirect?" (iframe still needs script integrity on parent page).

### 16.4 [Hard] How do you balance security controls with developer velocity and usability, and influence teams?
**What the interviewer is really testing**: Leadership without authority; paved-road thinking.
**Typical mid-level answer**: "Security is non-negotiable." Falls short: no influence strategy.
**Strong senior answer**: Friction creates workarounds, so the goal is making the secure path the easiest. Provide paved roads: templates with auth, logging, and policy preconfigured, shared NuGet libraries (tenant filter, audit client), pre-approved patterns and Terraform/Bicep modules. Keep gates fast and high-precision, and offer exception paths with expiry. Influence with evidence: show an incident or pen-test finding relevant to their service, quantify risk in business terms, and offer to pair on the fix. Choose battles: insist on non-negotiables (no secrets in repos, authz on every endpoint, no public data stores) and flex on lower-risk items with compensating controls. Measure outcomes (time to remediate, adoption of templates) and share wins. Trade-off: central teams must invest in enablement rather than policing. Residual risk: teams bypassing the platform; make non-compliance visible.
**Follow-up traps**: "Team says deadline?" (reduce scope not controls; time-boxed exception). "Security champions?" (embedded advocates, training).

### 16.5 [Tricky] "We're SOC 2 compliant, so we're secure." Agree?
**What the interviewer is really testing**: Healthy scepticism about compliance as security.
**Typical mid-level answer**: "Yes, it's audited." Falls short.
**Strong senior answer**: Compliance shows that defined controls existed (and for Type II, operated) in the audit scope; it doesn't guarantee that the control set addresses our actual threats, that the scope covered everything, or that a determined attacker can't exploit logic flaws. A SOC 2 report can be clean while an IDOR exists in an endpoint, because audits sample controls and rarely test application logic. Compliance is a baseline and customer-trust artifact; security is risk-driven engineering: threat modelling, testing, detection and response. Use frameworks as a checklist floor, and invest additionally where our risk is highest. Trade-off: audit cost vs real improvement; prefer controls that serve both. Residual risk: false confidence; keep independent testing (pen-tests, red teams) alongside.
**Follow-up traps**: "Which is the bigger driver?" (customer requirements and risk both). "Scope exclusions?" (read the system description carefully).

**Red flags in answers**
- Equating compliance with security (or dismissing it as paperwork).
- Taking card data on your servers "temporarily".
- Hand-collected screenshots as the evidence strategy.

**Cheat sheet**
- SOC 2 = attestation (Type II over time); ISO 27001 = certified ISMS; PCI = prescriptive card rules; HIPAA = US PHI law.
- Map one control to many frameworks.
- Evidence by default: IaC, Policy, PR approvals, access reviews.
- PCI: hosted fields/redirect + tokens; no PAN in your systems; script integrity.
- Paved roads beat gates; exceptions expire.
- Compliance is the floor, not the ceiling.

---

## Topic 17: Scenarios

**Study notes**
- For scenarios, narrate: clarify context -> list assets and trust boundaries -> threats -> controls in layers -> phased rollout -> trade-offs -> residual risk.
- State assumptions aloud; ask 2-3 clarifying questions (data sensitivity, user types, compliance, scale, timeline).

```
 Scenario skeleton:  WHAT are we protecting? -> FROM whom? -> WHERE are the boundaries?
                     -> controls (prevent / detect / respond) -> rollout order -> what remains
```

### 17.1 [Scenario] Design auth for 12 microservices plus a mobile app.
**What the interviewer is really testing**: End-to-end architecture thinking and trade-offs.
**Typical mid-level answer**: "Use JWT with a gateway that validates tokens." Falls short: gateway-only, no service-to-service, no authorisation model.
**Strong senior answer**: Assets: customer data and money-moving operations; actors: mobile users, services, admins. Identity: a central OIDC provider (Entra External ID or equivalent); the mobile app uses authorization code + PKCE via the system browser, keeps refresh tokens in Keychain/Keystore with rotation, and short-lived access tokens with audience per API (or a small number of audiences). Edge: APIM/gateway does TLS, throttling, WAF and coarse token checks. Each service validates JWTs itself via a shared NuGet package (issuer, audience, signature, scopes) with a fallback deny-by-default policy and does resource-level authorization in application code with query scoping. Service-to-service: managed identities with client-credentials-style tokens and app roles; OBO only where downstream must act as the user; no forwarding of user tokens blindly. Authorization data: coarse roles/scopes in tokens, fine-grained permissions looked up and cached. Operations: key rollover handling, structured audit events, central dashboards, per-service integration tests for 401/403/cross-user. Rollout: shared library and template first, migrate highest-risk services first. Trade-offs: token acquisition latency (cached), shared library coupling (versioned). Residual risk: stolen tokens within lifetime, compromised service within its roles; mitigated by short lifetimes, least privilege, detection.
**Follow-up traps**: "How do you revoke access immediately?" (short tokens + CAE where supported + live check for high risk). "Mobile offline?" (limited scopes, re-auth for sensitive actions).

### 17.2 [Scenario] Tenant A sees tenant B's data. Investigate and prevent recurrence.
**What the interviewer is really testing**: Incident discipline plus systemic fix.
**Typical mid-level answer**: "Find the missing WHERE clause." Falls short: ignores containment, scope, and class-level prevention.
**Strong senior answer**: Treat as a confirmed or suspected breach: contain (disable the affected endpoint or feature flag if reproducible), preserve logs, and open an incident with legal/DPO involved. Scope using audit logs: requests where actor tenant differs from object tenant in the window; list impacted tenants/records. Reproduce and find root cause: common ones are caches keyed without tenant, `IgnoreQueryFilters`, raw SQL, missing authorization on a new endpoint, background job context leak, RLS context not set on a pooled connection, or a shared singleton. Fix with a regression test. Prevent the class: tenant context from the validated token, mandatory scoped repositories, EF filter plus SQL RLS (FILTER and BLOCK predicates), tenant-prefixed cache wrapper, generated two-tenant integration matrix in CI, schema tests asserting RLS coverage, and analyzers banning raw access. Add detection (cross-tenant access alerts). Notify customers per legal guidance and run a blameless review. Trade-off: some performance and developer overhead for stronger guardrails. Residual risk: privileged DB access and novel paths; periodic pen-test.
**Follow-up traps**: "How do you know nothing else leaked?" (audit-log query plus test sweep; gaps in logging noted). "Can we just add RLS tonight?" (as an extra layer after validating performance/behaviour, not a panic change).

### 17.3 [Scenario] A pen-test found IDOR in 30 endpoints. Remediation plan?
**What the interviewer is really testing**: Systemic remediation vs whack-a-mole.
**Typical mid-level answer**: "Fix each endpoint." Falls short: doesn't prevent recurrence or prioritise.
**Strong senior answer**: Triage: group the 30 into patterns (missing ownership check, unscoped list queries, nested routes, batch endpoints). Rank by exposure and data sensitivity (internet-facing financial/PII first). Immediate mitigation for the worst few: temporary restrictions, WAF/gateway rules (limited efficacy), or disabling features, while checking logs for evidence of exploitation (sequential ID enumeration, abnormal 200s across owners) to decide on notification. Permanent fix by pattern: introduce a central resource-authorization mechanism (policy handler/`IAuthorizationService`) and owner/tenant-scoped repository methods, convert endpoints in batches with tests per endpoint (user A vs user B for read/update/delete/list). Add CI structural checks that fail on unscoped queries or endpoints without a resource policy. Have the testers retest. Track in a board with owners and SLAs and report progress. Trade-off: refactor effort vs quick patches; hybrid gives safety now and a durable fix. Residual risk: logic paths not covered by the pattern; schedule a targeted retest and expand the test matrix.
**Follow-up traps**: "Were the vulnerabilities exploited?" (log analysis; if unknown, treat per risk). "Switch to GUIDs?" (reduces enumeration, not a fix).

### 17.4 [Scenario] The business wants to store card data. How do you avoid PCI scope?
**What the interviewer is really testing**: Scope-avoidance architecture and pushing back constructively.
**Typical mid-level answer**: "Encrypt the card numbers in our database." Falls short: still fully in PCI scope.
**Strong senior answer**: Clarify the actual need: recurring billing, one-click checkout, refunds? Almost all can be done with provider tokens. Design: the checkout page uses the PSP's hosted fields/payment page so card data goes browser-to-PSP; our backend receives a payment token/payment-method ID and calls the PSP for charges, refunds, and subscriptions. We store token, brand, last four, and expiry only if allowed and needed; never CVV. Webhooks from the PSP confirm payment status, verified by signature with replay protection. Secure the integration points: CSP and script integrity on the payment page, strict access to PSP API keys (Key Vault, restricted keys), logs scrubbed of card-like patterns, segmentation to keep any residual CHD environment minimal. Confirm SAQ level with a QSA or the PSP guidance since requirements vary (verify current PCI DSS version). Trade-off: PSP lock-in and fees. Residual risk: client-side script compromise and PSP outage; monitor and have fallback.
**Follow-up traps**: "We need to display the full card?" (generally no; show last four). "Moving from one PSP to another?" (PSP-to-PSP token migration supported by providers; plan it).

### 17.5 [Scenario] A secret was committed to Git. Your response?
**What the interviewer is really testing**: Correct priority order and root-cause fix.
**Typical mid-level answer**: "Remove it from the repo and force push." Falls short.
**Strong senior answer**: Assume the secret is compromised regardless of repo visibility, since clones, forks, CI logs and caches retain it. Step 1: determine what it grants and revoke/rotate immediately (new secret deployed first where zero-downtime is needed). Step 2: review the access logs for that credential during the exposure window for misuse. Step 3: purge from history only as hygiene (git filter-repo/BFG, coordinating collaborators), knowing it doesn't undo exposure. Step 4: if repo was public or scope is large, treat as an incident and follow breach analysis. Step 5: fix the process: enable secret scanning with push protection, pre-commit hooks, move to managed identity/Key Vault, `.gitignore` and config templates, and educate without blame. Trade-off: rotation causes brief disruption if consumers aren't automated. Residual risk: secret copied elsewhere (chat, tickets) and unknown consumers; check.
**Follow-up traps**: "Private repo — still rotate?" (yes; access is broad and persistent). "Secret in a PR that was never merged?" (still in the remote's reflogs/refs; rotate).

**Red flags in answers**
- Jumping to technology before stating assets, actors and threats.
- Only reactive fixes; no systemic prevention or detection.
- Skipping legal/notification thinking when personal data may be exposed.

**Cheat sheet**
- Auth: central IdP, PKCE, per-service token validation, MI for S2S, resource-level authz.
- Tenant leak: contain, scope via audit logs, root-cause, layer guardrails, tests.
- IDOR x30: cluster patterns, central mechanism, retest, CI checks.
- PCI: hosted fields + tokens, signed webhooks, CSP/SRI; confirm SAQ.
- Secret in Git: rotate first, check usage, scan/push-protect, remove secret class.
- Always say what risk remains.

---

## Topic 18: Talking About Security Without Buzzwords

**Study notes**
- Interviewers distrust vocabulary without mechanism. Replace "zero trust, shift-left, defence in depth" with **what attack it stops and what it costs**.
- A reliable answer shape: **Asset -> attacker/threat -> control -> how it fails -> cost/trade-off -> what remains**.
- Use specifics from real systems (what you implemented, what broke, what you learned). Admit uncertainty; say "I'd verify that in the current docs."
- Avoid absolutes ("unhackable", "100% secure"); prefer "reduces likelihood/impact of X".

```
 Buzzword answer:  "We use zero trust with defence in depth."
 Mechanism answer: "Each API validates the token's audience and checks object ownership, so a stolen
                    token for service A can't read service B's data; the cost is some duplicated
                    validation code, which we put in a shared package."
```

### 18.1 [Easy] "How do you think about security day to day as a developer?"
**What the interviewer is really testing**: Habits and genuine ownership.
**Typical mid-level answer**: "I follow best practices and OWASP." Falls short: vague.
**Strong senior answer**: In design reviews I ask what data we hold, who can reach it, and what happens if each component is compromised, and I look at trust boundaries before writing code. In PRs I check for authorization on new endpoints, unscoped queries, user input reaching SQL/HTTP/file system, logging of sensitive data, and new dependencies. I prefer eliminating risk (no stored secrets via managed identity, no card data via tokenisation) over adding controls. I automate repeated checks (analyzers, tests, scanning) so they don't depend on memory. When I find an issue I look for the class and fix it in shared code. I say what I don't know and ask the security team early. Trade-off: slower first iteration, faster later. Residual risk: unknown unknowns; hence monitoring and periodic testing.
**Follow-up traps**: "Example from your work?" (have a concrete one ready, with a measurable outcome). "Biggest mistake?" (honest, with lesson).

### 18.2 [Follow-up] Explain a security control to a non-technical stakeholder.
**What the interviewer is really testing**: Communication of risk in business terms.
**Typical mid-level answer**: "It's industry standard." Falls short.
**Strong senior answer**: Start with what could go wrong in their language: "Right now anyone with one stolen password could download all customer invoices." Then the control: "We'll require a second factor and limit exports, which adds about ten seconds to login." Then the cost and what remains: "It won't stop an employee misusing legitimate access; we'll log and review exports for that." Offer options with price/time impact and a recommendation. Avoid acronyms; use analogies sparingly and accurately. Link to outcomes they care about: customer trust, contract requirements, regulatory fines, downtime. Trade-off: oversimplification risk; confirm understanding. Residual risk is part of the message: risk is reduced, not eliminated.
**Follow-up traps**: "Execs ask 'are we secure?'" (answer with posture, top risks, and plan, not yes/no). "Quantify?" (ranges and scenarios, avoiding false precision).

### 18.3 [Medium] "Is this design secure?" - how do you answer honestly?
**What the interviewer is really testing**: Calibrated confidence.
**Typical mid-level answer**: "Yes, it uses encryption and authentication." Falls short.
**Strong senior answer**: "Secure against what?" Security is relative to a threat model. I'd say: against opportunistic internet attackers and a stolen user password, I believe it holds because of X, Y, Z; against a compromised internal service it's weaker because the services share a database login - here's the cheapest improvement; against a malicious privileged admin, it relies on audit and separation of duties. Then I list assumptions I haven't verified (e.g. that RLS is applied to all tables) and the tests that would confirm. This shows I know where the design is strong, where weak, and what I'd do about it. Trade-off: less comforting than "yes", more useful. Residual risk is the explicit list of known gaps and accepted risks.
**Follow-up traps**: "Give me a yes/no." ("Acceptable for the stated threat model and data, with these gaps accepted by owner."). "How do you verify?" (tests, review, targeted pen-test).

### 18.4 [Hard] Describe a time you pushed back on a risky decision, or got something wrong in security.
**What the interviewer is really testing**: Integrity, learning, influence.
**Typical mid-level answer**: Blames others or has no example. Falls short.
**Strong senior answer**: Use a structured, honest story: context (service, data), the risk I saw or the mistake I made (e.g. I assumed the gateway enforced tenant checks and a downstream service had none), the impact (found in review/test, or in prod with limited exposure), what I did (contained, fixed, communicated), and the systemic change (shared library, test matrix, review checklist). Highlight collaboration with security and product, and how I framed the risk in business terms when pushing back, including an alternative that met the deadline. Show learning without self-flagellation or blame. Trade-off narrative: what the fix cost. Residual risk: what we still monitor. Prepare two such stories: one pushback, one mistake.
**Follow-up traps**: "Would you do anything differently?" (specific). "What if the decision-maker disagreed?" (document, escalate where required, accept with owner).

### 18.5 [Tricky] "What's the most overrated and underrated security practice?"
**What the interviewer is really testing**: Judgement and prioritisation.
**Typical mid-level answer**: Names fashionable tools. Falls short: no reasoning.
**Strong senior answer**: Overrated: perimeter-focused tools or compliance checklists treated as sufficient, and frequent forced password changes or obscurity (GUIDs, hidden URLs) as access control; also scanning that produces thousands of ignored alerts. Underrated: authorization correctness (BOLA is the dominant API flaw and tools miss it), least-privilege identity hygiene (broad Contributor roles, standing admin access), inventory (knowing what you run and which data lives where), boring logging and alerting that works, tested restore and rotation procedures, and secure-by-default platform libraries. Rationale: attackers take the cheapest path, which is usually stolen credentials, over-permissioned identities, and logic flaws, not exotic exploits. Trade-off: unglamorous work gets less visibility; advocate with incident evidence. Residual risk: my view is shaped by my context; I'd adjust per threat model.
**Follow-up traps**: "Password rotation?" (current guidance such as NIST discourages forced periodic rotation without evidence of compromise - verify in current docs). "Where would you invest first?" (identity and authorization, inventory, detection).

**Red flags in answers**
- Strings of acronyms with no mechanism, cost or limit.
- Claiming something is "fully secure" or "compliant, therefore safe".
- No personal examples; or blaming others for every failure.

**Cheat sheet**
- Shape: asset -> threat -> control -> failure mode -> cost -> residual risk.
- Say what a control stops and what it doesn't.
- Prefer eliminating risk over adding controls.
- Have two real stories: a pushback and a mistake.
- Translate risk into business impact and options.
- Admit unknowns; "I'd verify in current docs" is a strength.

---
*End of S07.*
