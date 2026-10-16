# B04 — ASP.NET Core Security: Study + Interview Guide

**How to use:** Read each topic's *Study notes* first, then climb the difficulty ladder (Easy → Tricky) and try to answer aloud before reading the **Answer**.
Code targets C# 12 / .NET 8. Snippets are focused fragments; assume `using` directives for the namespaces named in the text.

## Table of Contents
1. Authentication vs Authorization
2. Cookie Auth vs JWT Bearer
3. Access vs Refresh Tokens
4. OAuth 2.0 and OpenID Connect
5. ASP.NET Core Identity and Password Hashing
6. Authorization Models: Role, Claims, Policy, Resource
7. Custom Handlers, Attributes and Fallback Policy
8. CORS
9. CSRF and Antiforgery
10. XSS, SQL Injection and Mass Assignment
11. Data Protection API, HTTPS/HSTS and Security Headers
12. Secrets Management, API Keys and HMAC Signing
13. Rate Limiting and Brute-Force Protection
14. Entra ID, SSO, B2C and Multi-Tenant Authorization
15. Audit Logging, PII/GDPR and OWASP Top 10 Mapping
16. Security Interview Scenarios (Stolen Token, IDOR)

---

## Topic 1: Authentication vs Authorization

**Study notes**
- **Authentication (AuthN)** = who are you? Produces a `ClaimsPrincipal` (one or more `ClaimsIdentity`, each with `Claim`s). **Authorization (AuthZ)** = what may you do? Evaluated against the principal.
- Pipeline order matters: `UseAuthentication()` populates `HttpContext.User`; `UseAuthorization()` enforces policies. Authorization must come after authentication and (with endpoint routing) after `UseRouting()`.
- Auth **schemes** are named pairs of (handler, options). Each handler implements `IAuthenticationHandler` (usually derives `AuthenticationHandler<TOptions>`). Operations: Authenticate, Challenge (401 / redirect to login), Forbid (403 / access denied), SignIn, SignOut.
- 401 = not authenticated (or bad credentials); 403 = authenticated but not allowed.

```
Request -> UseAuthentication -> [scheme handler reads cookie/header -> ClaimsPrincipal]
        -> UseAuthorization  -> [policy evaluates principal/resource]
              fail & anonymous  -> Challenge (401/302)
              fail & logged in  -> Forbid (403)
```

### 1.1 [Easy] What is the difference between authentication and authorization?
**Answer**
Authentication verifies identity (who the caller is); authorization decides whether that identity may perform an action. In ASP.NET Core, authentication builds a `ClaimsPrincipal` and authorization evaluates policies against it. Failing the first yields 401, failing the second yields 403.

**Explanation**
They are separate middleware and separate services (`IAuthenticationService` vs `IAuthorizationService`). You can authenticate with cookies, JWT, etc. and authorize with roles, claims, or custom policies independently.

**Code**
```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddAuthentication("Cookies").AddCookie();
builder.Services.AddAuthorization();
var app = builder.Build();

app.UseAuthentication(); // who are you?
app.UseAuthorization();  // may you?

app.MapGet("/me", (HttpContext ctx) => ctx.User.Identity?.Name ?? "anonymous");
app.MapGet("/admin", () => "secret").RequireAuthorization();
app.Run();
```

### 1.2 [Follow-up] When does ASP.NET Core return 401 vs 403, and what do Challenge/Forbid do?
**Answer**
An unauthenticated user failing authorization triggers `ChallengeAsync` (401 for APIs/JWT, 302 redirect to login for cookies). An authenticated user failing authorization triggers `ForbidAsync` (403, or redirect to access-denied for cookies). The scheme handler decides the concrete response.

**Explanation**
`PolicyEvaluator` returns `PolicyAuthorizationResult` (Challenged / Forbidden / Succeeded). `AuthorizationMiddleware` then calls the matching method on the scheme. Cookie handler redirects by default; for APIs you override `OnRedirectToLogin` to return 401.

**Code**
```csharp
builder.Services.AddAuthentication("Cookies").AddCookie("Cookies", o =>
{
    o.Events.OnRedirectToLogin = ctx => { ctx.Response.StatusCode = 401; return Task.CompletedTask; };
    o.Events.OnRedirectToAccessDenied = ctx => { ctx.Response.StatusCode = 403; return Task.CompletedTask; };
});
```

### 1.3 [Medium] What is a ClaimsPrincipal and how can one user have multiple identities?
**Answer**
`ClaimsPrincipal` holds one or more `ClaimsIdentity` objects, each carrying claims and an `AuthenticationType`. Multiple identities occur when several schemes authenticate or when you add identities via claims transformation. `Identity.IsAuthenticated` is true only if `AuthenticationType` is non-empty.

**Explanation**
`HttpContext.User` is produced by the default authenticate scheme only (unless a policy lists several schemes, in which case the evaluator merges principals). `IClaimsTransformation` runs on every `AuthenticateAsync` call, so make it idempotent.

**Code**
```csharp
using System.Security.Claims;

var anon = new ClaimsIdentity();                         // no authenticationType
var auth = new ClaimsIdentity(new[] { new Claim(ClaimTypes.Name, "ann") }, "Cookies");
Console.WriteLine(anon.IsAuthenticated); // Output: False
Console.WriteLine(auth.IsAuthenticated); // Output: True
var principal = new ClaimsPrincipal(new[] { anon, auth });
Console.WriteLine(principal.Identities.Count()); // Output: 2
```

### 1.4 [Hard] Why can `UseAuthorization()` placed before `UseAuthentication()` leave endpoints "authorized" wrongly or always 401?
**Answer**
Authorization middleware reads `HttpContext.User`; if authentication has not run yet, `User` is anonymous, so protected endpoints always challenge (policies with no explicit schemes just inspect `context.User`). Always order: `UseRouting` → `UseAuthentication` → `UseAuthorization` → endpoints.

**Explanation**
In .NET 7+, `WebApplication` auto-adds `UseAuthentication` (when auth services are registered) early in the pipeline, which hides ordering mistakes in minimal hosting. Policies with explicit `AuthenticationSchemes` authenticate those schemes themselves, but any custom middleware placed before `UseAuthentication` that inspects `User` sees anonymous.

**Gotchas / follow-ups**
- Custom middleware needing the user must be after `UseAuthentication`.
- A `[AllowAnonymous]` endpoint still runs authentication (if registered) but doesn't require success.

**Code**
```csharp
app.UseRouting();
app.UseAuthentication();
app.UseAuthorization();
app.Use(async (ctx, next) =>
{
    // Safe: User is populated here
    Console.WriteLine(ctx.User.Identity?.IsAuthenticated);
    await next();
});
```

### 1.5 [Tricky] Does `[Authorize]` on a controller protect an action marked `[AllowAnonymous]`? What about `[Authorize(Roles="Admin")]` on the action under `[AllowAnonymous]` controller?
**Answer**
`[AllowAnonymous]` anywhere on the endpoint's metadata short-circuits authorization, so the action is open. An action-level `[AllowAnonymous]` overrides controller `[Authorize]`; and a controller-level `[AllowAnonymous]` also wins over action-level `[Authorize]` because the authorization middleware just checks for the presence of `IAllowAnonymous` metadata.

**Explanation**
Unlike filters, endpoint metadata is aggregated: `IAllowAnonymous` present anywhere → bypass. This is a classic gotcha: adding `[AllowAnonymous]` on a base controller silently opens every derived action's `[Authorize]`.

**Gotchas / follow-ups**
- With MVC filters (`IAllowAnonymousFilter`), behaviour is the same in recent versions.
- Fallback policy doesn't apply to endpoints having `[AllowAnonymous]`.

**Code**
```csharp
[AllowAnonymous]
public class BaseController : ControllerBase { }

public class OrdersController : BaseController
{
    [Authorize(Roles = "Admin")] // STILL anonymous-accessible because of AllowAnonymous on the base class
    [HttpGet("orders")] public IActionResult Get() => Ok();
}
```

**Interview traps**
- 401 vs 403 confusion; many say "403 for not logged in".
- Ordering of middleware.
- `[AllowAnonymous]` always wins.
- `IsAuthenticated` depends on `AuthenticationType`.

**Cheat sheet**
- AuthN → ClaimsPrincipal; AuthZ → policies.
- 401 = challenge, 403 = forbid.
- Order: Routing → AuthN → AuthZ → endpoints.
- `AllowAnonymous` beats `Authorize`.
- Claims transformation runs on every authenticate: keep idempotent.

---

## Topic 2: Cookie Auth vs JWT Bearer

**Study notes**
- **Cookie auth**: server issues an encrypted+signed (Data Protection) cookie containing the serialized principal (`AuthenticationTicket`). Browser sends it automatically. Stateful-ish (sliding expiration, can use `ITicketStore` for server-side sessions). Needs CSRF protection. Best for server-rendered apps / same-site BFF.
- **JWT bearer**: client sends `Authorization: Bearer <jwt>`. Token is self-contained and signed (JWS); stateless validation. Not auto-sent by browser → not CSRF-prone, but token theft via XSS if stored in JS-accessible storage.
- JWT = `base64url(header).base64url(payload).base64url(signature)`. Payload is **encoded, not encrypted** — never put secrets in it.
- Registered claims: `iss`, `sub`, `aud`, `exp`, `nbf`, `iat`, `jti`.
- Signing: **HS256** = HMAC-SHA256, shared symmetric secret (issuer and validator both can mint tokens). **RS256** = RSA-SHA256, private key signs, public key verifies (validators can't mint; keys distributed via JWKS). ES256 is another asymmetric option.
- Validation (`TokenValidationParameters`): signature, issuer, audience, lifetime (`exp`/`nbf`), `ClockSkew` (default **5 minutes**).

```
JWT:  eyJhbGciOi...(header).eyJzdWIiOi...(payload).SflKxw...(signature)
sig = HMAC_SHA256(secret, header + "." + payload)    // HS256
sig = RSA_Sign(privKey, SHA256(header + "." + payload)) // RS256
```

### 2.1 [Easy] What is a JWT and what are its three parts?
**Answer**
A JSON Web Token is a compact, URL-safe string of header, payload (claims) and signature, separated by dots. The signature proves integrity and authenticity of header+payload. The payload is only base64url-encoded, so anyone can read it.

**Explanation**
Header holds `alg` and `typ` (and `kid`). Payload holds claims. A signed JWT is a JWS; an encrypted one is a JWE (rarely used in APIs).

**Code**
```csharp
using System.IdentityModel.Tokens.Jwt;
using Microsoft.IdentityModel.Tokens;
using System.Text;
using System.Security.Claims;

var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes("a-very-long-secret-key-at-least-32-bytes!!"));
var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);
var token = new JwtSecurityToken(
    issuer: "https://api.example", audience: "web",
    claims: new[] { new Claim(JwtRegisteredClaimNames.Sub, "42") },
    expires: DateTime.UtcNow.AddMinutes(15), signingCredentials: creds);
string jwt = new JwtSecurityTokenHandler().WriteToken(token);
Console.WriteLine(jwt.Split('.').Length); // Output: 3
```

### 2.2 [Follow-up] How do you configure JWT bearer validation in ASP.NET Core?
**Answer**
Call `AddAuthentication(JwtBearerDefaults.AuthenticationScheme).AddJwtBearer(...)` and set `TokenValidationParameters` with issuer, audience, signing key and lifetime validation. Prefer `Authority` (OIDC metadata/JWKS) for third-party IdPs so keys rotate automatically.

**Explanation**
With `Authority`, the handler downloads discovery metadata and signing keys, and refreshes them. Never disable `ValidateIssuer/Audience/Lifetime/IssuerSigningKey` in production.

**Code**
```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(o =>
    {
        o.Authority = "https://login.example.com";
        o.Audience = "my-api";
        o.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true, ValidateAudience = true,
            ValidateLifetime = true, ValidateIssuerSigningKey = true,
            ClockSkew = TimeSpan.FromSeconds(30)
        };
    });
```

### 2.3 [Medium] HS256 vs RS256: when do you choose which?
**Answer**
HS256 uses one shared secret, so every validator could forge tokens; it suits a single service that both issues and validates. RS256 uses a private key to sign and a public key to verify, so many APIs can validate via JWKS without being able to issue tokens. Use RS256/ES256 for multi-service or third-party scenarios.

**Explanation**
Asymmetric keys enable rotation via `kid` and published JWKS. Symmetric keys need secure distribution to each validator. RSA signing is slower, verification is fast.

**Code**
```csharp
using System.Security.Cryptography;

using var rsa = RSA.Create(2048);
var rsaKey = new RsaSecurityKey(rsa) { KeyId = "key-2024-01" };
var creds = new SigningCredentials(rsaKey, SecurityAlgorithms.RsaSha256);
// validators only need the public part:
var publicKey = new RsaSecurityKey(rsa.ExportParameters(false)) { KeyId = "key-2024-01" };
```

### 2.4 [Hard] Explain ClockSkew. What's the effective lifetime of a token with `exp` = 10:00 and defaults?
**Answer**
`ClockSkew` tolerates clock differences between issuer and validator; the default is 5 minutes and it's added to expiry, so the token is accepted until about 10:05. For short-lived tokens (e.g. 5-minute) the default effectively doubles lifetime, so set it lower (30 s or `TimeSpan.Zero`) if you need strictness.

**Explanation**
Validation checks `notBefore - skew <= now <= expires + skew`. Too small a skew on machines with drifted clocks produces spurious 401s `IDX10223: Lifetime validation failed`. Keep servers NTP-synced.

**Gotchas / follow-ups**
- `ClockSkew` also applies to `nbf`.
- A custom `LifetimeValidator` can replace the logic.
- Revocation is not solved by expiry; see Topic 3.

**Code**
```csharp
var p = new TokenValidationParameters { ClockSkew = TimeSpan.Zero };
Console.WriteLine(new TokenValidationParameters().ClockSkew); // Output: 00:05:00
```

### 2.5 [Tricky] What is the "alg: none" / algorithm confusion attack and how does .NET defend against it?
**Answer**
An attacker alters the header to `alg: none` (no signature) or switches RS256 to HS256 and signs with the public key as the HMAC secret. Defence: validate with a fixed key and restrict accepted algorithms via `ValidAlgorithms`; Microsoft.IdentityModel rejects unsigned tokens by default (`RequireSignedTokens = true`) and chooses the key by configured key, not by the attacker's header.

**Explanation**
The vulnerability arises in libraries that trust the header's `alg` to pick the verification method. In .NET, a key type (RSA vs symmetric) binds to algorithm families, but pinning `ValidAlgorithms` is cheap defence-in-depth.

**Gotchas / follow-ups**
- Also validate `kid` against trusted keys only; don't fetch keys from a `jku`/`x5u` URL in the token.
- Keep `RequireSignedTokens=true`.

**Code**
```csharp
var tvp = new TokenValidationParameters
{
    ValidAlgorithms = new[] { SecurityAlgorithms.RsaSha256 },
    RequireSignedTokens = true,
    ValidateIssuerSigningKey = true
};
```

**Interview traps**
- "JWT is encrypted" — no, only signed; payload is readable.
- Using HS256 with the secret shared among many services.
- Leaving default 5-minute ClockSkew with short expiries.
- Storing JWT in `localStorage` (XSS-readable).
- Cookies need CSRF protection; bearer tokens don't (not auto-sent).

**Cheat sheet**
- JWT = header.payload.signature (base64url).
- HS256 symmetric; RS256 asymmetric + JWKS.
- Validate iss, aud, exp, signature; pin algorithms.
- ClockSkew default 5 min.
- Cookie: auto-sent, CSRF risk; JWT: manual, XSS risk if in JS storage.

---

## Topic 3: Access vs Refresh Tokens, Rotation, Revocation, Storage

**Study notes**
- **Access token**: short-lived (5–15 min), sent to the API on each call. **Refresh token**: long-lived, opaque, sent only to the token endpoint to get new access tokens.
- Why two? Limits blast radius of a leaked access token while keeping users signed in.
- **Rotation**: every refresh returns a new refresh token and invalidates the old. **Reuse detection**: if an already-used refresh token is presented, revoke the whole token family (theft signal).
- **Revocation**: JWTs are stateless, so revoke by short expiry + revoking refresh tokens; optionally a deny-list keyed on `jti`/session id, or introspection (RFC 7662).
- **Storage**: browser SPA — prefer BFF with HttpOnly+Secure+SameSite cookie; avoid `localStorage`. Server: store refresh tokens **hashed** (like passwords, but SHA-256 suffices for high-entropy random values). Mobile: platform secure storage.

```
Client --access(expired)--> API 401
Client --refresh_token----> /token  -> new access + new refresh (old one marked used)
Reuse of old refresh --> revoke family, force re-login
```

### 3.1 [Easy] Why use a refresh token instead of a long-lived access token?
**Answer**
A short-lived access token limits damage if stolen, while the refresh token lets the client silently get new ones without prompting the user. The refresh token is only sent to the authorization server, reducing its exposure.

**Explanation**
Refresh tokens can be bound to a client/device, revoked server-side, and rotated. Access tokens are validated offline by APIs, so you can't revoke them instantly.

**Code**
```csharp
public sealed record TokenResponse(string AccessToken, string RefreshToken, int ExpiresIn);
// Typical issuance: access 15 min, refresh 7-30 days
```

### 3.2 [Follow-up] How do you generate and store a refresh token securely?
**Answer**
Generate with a cryptographically secure RNG (at least 32 bytes), return the raw value once, and store only its hash plus metadata (user, expiry, device, family, used/revoked flags). Never use `Guid` or `Random`.

**Explanation**
Since a refresh token is high-entropy random, a fast hash (SHA-256) is fine; slow password hashes aren't needed. Hashing means a DB leak doesn't leak usable tokens.

**Code**
```csharp
using System.Security.Cryptography;

static (string raw, string hash) NewRefreshToken()
{
    byte[] bytes = RandomNumberGenerator.GetBytes(32);
    string raw = Convert.ToBase64String(bytes);
    string hash = Convert.ToHexString(SHA256.HashData(Encoding.UTF8.GetBytes(raw)));
    return (raw, hash);
}
```

### 3.3 [Medium] Describe refresh token rotation with reuse detection.
**Answer**
On each refresh, mark the presented token as used and issue a new one in the same family. If a used token is presented again, assume theft: revoke every token in that family and force re-authentication. This turns a stolen-and-replayed token into a detectable event.

**Explanation**
Race condition: legit client may retry after a network failure and send the old token twice. Mitigate with a short grace window or idempotent response. Persist with a transaction and a concurrency token so two parallel refreshes can't both succeed.

**Code**
```csharp
public class RefreshToken
{
    public int Id { get; set; }
    public string TokenHash { get; set; } = "";
    public Guid FamilyId { get; set; }
    public DateTime ExpiresUtc { get; set; }
    public DateTime? UsedUtc { get; set; }
    public DateTime? RevokedUtc { get; set; }
    public byte[] RowVersion { get; set; } = []; // configure as concurrency token
}

async Task<TokenResponse?> RefreshAsync(AppDb db, string raw)
{
    var hash = Convert.ToHexString(SHA256.HashData(Encoding.UTF8.GetBytes(raw)));
    var rt = await db.RefreshTokens.SingleOrDefaultAsync(t => t.TokenHash == hash);
    if (rt is null || rt.ExpiresUtc < DateTime.UtcNow || rt.RevokedUtc != null) return null;
    if (rt.UsedUtc != null)
    {   // reuse detected: revoke whole family
        await db.RefreshTokens.Where(t => t.FamilyId == rt.FamilyId)
            .ExecuteUpdateAsync(s => s.SetProperty(t => t.RevokedUtc, DateTime.UtcNow));
        return null;
    }
    rt.UsedUtc = DateTime.UtcNow;
    // issue new token in same family, save atomically ...
    await db.SaveChangesAsync();
    return null; // build and return the new pair here
}
```

### 3.4 [Hard] How can you revoke a JWT access token immediately?
**Answer**
You can't with pure stateless validation. Options: keep access tokens very short; maintain a deny-list of `jti` (or a per-user "tokens valid after" timestamp/security stamp) checked in `OnTokenValidated`; or use opaque tokens with introspection. Each option reintroduces a state lookup, trading scalability for control.

**Explanation**
A cheap pattern: store `SecurityStamp` or `TokenVersion` in the token and compare to a cached value on each request; bump it on logout/password change. Cache with a short TTL to limit DB hits.

**Gotchas / follow-ups**
- Distributed cache needed in multi-instance deployments.
- Logout in a stateless world only deletes client copy unless you do the above.

**Code**
```csharp
.AddJwtBearer(o => o.Events = new JwtBearerEvents
{
    OnTokenValidated = async ctx =>
    {
        var cache = ctx.HttpContext.RequestServices.GetRequiredService<IDistributedCache>();
        var jti = ctx.Principal!.FindFirstValue(JwtRegisteredClaimNames.Jti);
        if (jti != null && await cache.GetStringAsync($"revoked:{jti}") != null)
            ctx.Fail("Token revoked");
    }
});
```

### 3.5 [Tricky] Where should a SPA store tokens, and what's the trade-off between localStorage and HttpOnly cookies?
**Answer**
`localStorage`/`sessionStorage` is readable by any injected script (XSS steals tokens). HttpOnly cookies are invisible to JS but are auto-sent, so you need CSRF defences (SameSite, antiforgery). The recommended pattern is a Backend-for-Frontend (BFF) that holds tokens server-side and gives the browser only a session cookie.

**Explanation**
Even with HttpOnly, XSS can still make authenticated requests as the user (riding the session), but can't exfiltrate the token for use elsewhere. BFF also keeps refresh tokens off the browser.

**Gotchas / follow-ups**
- SameSite=Lax/Strict plus custom header checks mitigate CSRF.
- In-memory tokens + silent refresh via cookie is a hybrid.

**Code**
```csharp
builder.Services.AddAuthentication("Cookies").AddCookie("Cookies", o =>
{
    o.Cookie.HttpOnly = true;
    o.Cookie.SecurePolicy = CookieSecurePolicy.Always;
    o.Cookie.SameSite = SameSiteMode.Strict;
    o.SlidingExpiration = true;
    o.ExpireTimeSpan = TimeSpan.FromMinutes(30);
});
```

**Interview traps**
- Claiming JWTs can be "logged out" server-side without any state.
- Storing refresh tokens in plaintext in the DB.
- Ignoring the concurrency race in rotation.
- Using `Guid.NewGuid()` as a token (not a CSPRNG guarantee).

**Cheat sheet**
- Access short (minutes), refresh long, opaque, hashed at rest.
- Rotate on use; reuse ⇒ revoke family.
- Revoke JWT = deny-list/version check/short life.
- SPA: BFF + HttpOnly cookie > localStorage.
- Use `RandomNumberGenerator`.

---

## Topic 4: OAuth 2.0 and OpenID Connect

**Study notes**
- **OAuth 2.0** = delegated *authorization* framework (access tokens for APIs). **OIDC** = identity layer on top: adds **ID token** (a JWT about the user), `openid` scope, UserInfo endpoint, discovery (`/.well-known/openid-configuration`).
- Roles: Resource Owner, Client, Authorization Server, Resource Server.
- **Authorization Code + PKCE**: for user-facing apps (SPA, mobile, and recommended for confidential web apps too). **Client Credentials**: machine-to-machine, no user. Implicit and ROPC (password) flows are deprecated.
- **PKCE**: client creates random `code_verifier`, sends `code_challenge = BASE64URL(SHA256(verifier))` with the auth request, then sends `code_verifier` at token exchange. Stops intercepted-code attacks.
- Other params: `state` (CSRF on the redirect), `nonce` (OIDC replay protection of ID token), exact `redirect_uri` matching.
- Access token is for the API (audience = API); ID token is for the client (audience = client id). Don't send ID tokens to APIs.

```
User -> Client -> /authorize?response_type=code&code_challenge=H(v)&state&nonce
User logs in at IdP -> redirect back ?code=XYZ&state
Client -> /token (code + code_verifier [+client_secret]) -> access, id, refresh tokens
```

### 4.1 [Easy] What's the difference between OAuth 2.0 and OpenID Connect?
**Answer**
OAuth 2.0 grants access to resources via access tokens and says nothing standard about user identity. OIDC adds authentication: an ID token with user claims, standard scopes (`openid`, `profile`, `email`) and discovery. "OAuth for authorization, OIDC for login."

**Explanation**
Using an access token to determine "who logged in" is a known anti-pattern because access tokens are meant for the API, may be opaque, and aren't audience-bound to the client.

**Code**
```csharp
builder.Services.AddAuthentication(o =>
{
    o.DefaultScheme = "Cookies";
    o.DefaultChallengeScheme = "oidc";
})
.AddCookie("Cookies")
.AddOpenIdConnect("oidc", o =>
{
    o.Authority = "https://login.example.com";
    o.ClientId = "web";
    o.ClientSecret = builder.Configuration["Oidc:Secret"];
    o.ResponseType = "code"; // auth code flow; PKCE is on by default in ASP.NET Core 3.0+ handler
    o.UsePkce = true;
    o.SaveTokens = true;
    o.Scope.Add("openid"); o.Scope.Add("profile");
});
```

### 4.2 [Follow-up] Which flow for which client type? Explain client credentials.
**Answer**
SPA/mobile → Authorization Code + PKCE (public clients, no secret). Server-side web app → Authorization Code (with secret and PKCE). Service-to-service with no user → Client Credentials, where the client authenticates with its secret/certificate and gets an access token for itself.

**Explanation**
Client credentials has no user context, so no refresh token or ID token; tokens carry app identity/roles (app roles in Entra ID). Prefer certificates or managed identity over secrets.

**Code**
```csharp
using var http = new HttpClient();
var res = await http.PostAsync("https://login.example.com/connect/token",
    new FormUrlEncodedContent(new Dictionary<string, string>
    {
        ["grant_type"] = "client_credentials",
        ["client_id"] = "svc-a",
        ["client_secret"] = Environment.GetEnvironmentVariable("SVC_A_SECRET")!,
        ["scope"] = "api.read"
    }));
```

### 4.3 [Medium] How does PKCE work and what attack does it stop?
**Answer**
The client generates a random verifier, sends its SHA-256 hash (challenge) in the authorize request, and presents the raw verifier when redeeming the code. If an attacker steals the authorization code (e.g., via a malicious app on a custom URI scheme), they can't redeem it without the verifier.

**Explanation**
This binds the code to the instance that started the flow, so public clients without secrets are still protected. Method should be `S256`, not `plain`.

**Code**
```csharp
using System.Security.Cryptography;
using System.Text;

static string B64Url(byte[] b) => Convert.ToBase64String(b).TrimEnd('=').Replace('+', '-').Replace('/', '_');
string verifier = B64Url(RandomNumberGenerator.GetBytes(32));
string challenge = B64Url(SHA256.HashData(Encoding.ASCII.GetBytes(verifier)));
Console.WriteLine(challenge.Length); // Output: 43
```

### 4.4 [Hard] What are `state` and `nonce` for, and what happens if you skip them?
**Answer**
`state` is an opaque value the client round-trips in the redirect and checks to prevent CSRF/login-CSRF on the callback. `nonce` is embedded in the ID token and checked by the client to prevent ID-token replay/injection. Skipping them enables an attacker to bind the victim's session to the attacker's account or replay tokens.

**Explanation**
ASP.NET Core's OIDC handler generates and validates both automatically, storing them in correlation/nonce cookies. This is why `SameSite` and cookie issues (e.g. behind proxies, http vs https) commonly cause "Correlation failed" errors.

**Gotchas / follow-ups**
- "Correlation failed" = correlation cookie missing: check SameSite=None;Secure, HTTPS redirect, forwarded headers.
- Always register exact redirect URIs; wildcard redirect = token leakage.

**Code**
```csharp
builder.Services.Configure<OpenIdConnectOptions>("oidc", o =>
{
    o.NonceCookie.SameSite = SameSiteMode.None;
    o.CorrelationCookie.SameSite = SameSiteMode.None;
});
```

### 4.5 [Tricky] Why validate the `aud` and why is accepting an ID token at your API dangerous?
**Answer**
`aud` says which recipient a token was minted for. If your API accepts any validly-signed token from the IdP without checking audience, a token issued for another app can be replayed against your API (confused deputy). An ID token's audience is the client app, so it must never be accepted as an API credential.

**Explanation**
Also check `scp`/`scope` or `roles` claims to enforce permissions, and `azp`/`appid` if needed. In multi-tenant Entra apps also validate issuer per tenant.

**Gotchas / follow-ups**
- Set `ValidateAudience = true` and a specific `ValidAudience`.
- With `Authority` and `Audience` set on JwtBearerOptions, audience validation is on.

**Code**
```csharp
builder.Services.AddAuthorization(o => o.AddPolicy("ReadScope", p =>
    p.RequireAssertion(ctx => ctx.User.FindFirst("scope")?.Value.Split(' ').Contains("api.read") == true)));
```

**Interview traps**
- "OAuth is for authentication" — it's authorization; OIDC authenticates.
- Implicit flow / ROPC are deprecated.
- Wildcard redirect URIs.
- Sending ID token to APIs.
- Believing PKCE is only for public clients (it's recommended for all).

**Cheat sheet**
- Code+PKCE for users; client credentials for services.
- ID token → client; access token → API.
- PKCE = verifier/S256 challenge.
- `state` anti-CSRF; `nonce` anti-replay.
- Always validate iss, aud, exp, signature.

---

## Topic 5: ASP.NET Core Identity and Password Hashing

**Study notes**
- ASP.NET Core Identity = membership system: `UserManager<TUser>`, `SignInManager<TUser>`, `RoleManager<TRole>`, EF Core stores (`IdentityDbContext`). Tables: AspNetUsers, AspNetRoles, AspNetUserRoles, AspNetUserClaims, AspNetUserLogins, AspNetUserTokens, AspNetRoleClaims.
- Default hasher `PasswordHasher<TUser>`: **PBKDF2 / HMAC-SHA512, 100,000 iterations** (Identity v3 format, since .NET 7; earlier HMAC-SHA256 with 10,000), 128-bit random salt, 256-bit subkey; stored as base64 with a format marker so it can be upgraded (`PasswordVerificationResult.SuccessRehashNeeded`).
- Salt = random per-password value; defeats rainbow tables and makes equal passwords hash differently. Pepper = secret app-wide value stored outside DB (optional).
- Algorithms: **PBKDF2** (FIPS-friendly, built-in), **bcrypt** (72-byte input limit, cost factor), **Argon2id** (memory-hard, current OWASP first choice; needs third-party package). Never MD5/SHA-1/plain SHA-256.
- **Lockout**: `LockoutOptions` (default 5 failed attempts, 5-minute lockout) – only counted when `lockoutOnFailure: true` in `PasswordSignInAsync`.
- **2FA**: TOTP authenticator apps via `UserManager.GetAuthenticatorKeyAsync` / `VerifyTwoFactorTokenAsync`; recovery codes; SMS/email providers.
- **SecurityStamp** changes on password/role change and invalidates existing cookies (validated periodically, default every 30 min).

### 5.1 [Easy] How does ASP.NET Core Identity store passwords?
**Answer**
It never stores the password; it stores a salted, iterated hash produced by `IPasswordHasher<TUser>`. The default is PBKDF2 with HMAC-SHA512 and 100,000 iterations (in .NET 7+), with a random salt embedded in the stored string.

**Explanation**
The hash string includes a version byte, PRF, iteration count and salt size, so parameters can be raised later and old hashes verified, then rehashed on next login.

**Code**
```csharp
using Microsoft.AspNetCore.Identity;

var hasher = new PasswordHasher<IdentityUser>();
var user = new IdentityUser("ann");
string h1 = hasher.HashPassword(user, "P@ssw0rd!");
string h2 = hasher.HashPassword(user, "P@ssw0rd!");
Console.WriteLine(h1 == h2); // Output: False (random salt)
Console.WriteLine(hasher.VerifyHashedPassword(user, h1, "P@ssw0rd!")); // Output: Success
```

### 5.2 [Follow-up] Configure password rules, lockout and set up Identity with EF Core.
**Answer**
Call `AddIdentity`/`AddIdentityCore` (or `AddDefaultIdentity`) with `AddEntityFrameworkStores<TContext>` and configure `IdentityOptions`. Lockout applies only when you pass `lockoutOnFailure: true` when signing in.

**Explanation**
Modern guidance (NIST 800-63B) favours length and breach-list checks over composition rules. You can plug custom `IPasswordValidator<TUser>` to check breached passwords.

**Code**
```csharp
builder.Services.AddDbContext<AppDb>(o => o.UseSqlServer(builder.Configuration.GetConnectionString("Default")));
builder.Services.AddIdentity<IdentityUser, IdentityRole>(o =>
{
    o.Password.RequiredLength = 12;
    o.Password.RequireNonAlphanumeric = false;
    o.Lockout.MaxFailedAccessAttempts = 5;
    o.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromMinutes(15);
    o.User.RequireUniqueEmail = true;
    o.SignIn.RequireConfirmedEmail = true;
}).AddEntityFrameworkStores<AppDb>().AddDefaultTokenProviders();

// AppDb : IdentityDbContext<IdentityUser>
var result = await signInManager.PasswordSignInAsync(email, pwd, isPersistent: false, lockoutOnFailure: true);
if (result.IsLockedOut) { /* show lockout message */ }
```

### 5.3 [Medium] Compare PBKDF2, bcrypt and Argon2. Which would you pick?
**Answer**
All three are deliberately slow, salted KDFs. PBKDF2 is CPU-bound only (GPU-friendly attackers), built into .NET and FIPS-approved. bcrypt has a tunable cost but a 72-byte limit and little memory hardness. Argon2id is memory-hard, resisting GPU/ASIC attacks, and is the current OWASP first choice; choose it when you can take a dependency, otherwise PBKDF2 with high iterations is acceptable.

**Explanation**
Tune cost so one hash takes ~100–500 ms on your hardware. Use constant-time comparison (`CryptographicOperations.FixedTimeEquals`). Plan rehash-on-login when raising parameters.

**Code**
```csharp
using System.Security.Cryptography;

byte[] salt = RandomNumberGenerator.GetBytes(16);
byte[] hash = Rfc2898DeriveBytes.Pbkdf2("secret", salt, 600_000, HashAlgorithmName.SHA256, 32);
byte[] again = Rfc2898DeriveBytes.Pbkdf2("secret", salt, 600_000, HashAlgorithmName.SHA256, 32);
Console.WriteLine(CryptographicOperations.FixedTimeEquals(hash, again)); // Output: True
```

### 5.4 [Hard] How would you upgrade password hashing without forcing every user to reset?
**Answer**
Keep verifying the old hash format, and on successful login rehash with the new algorithm/parameters and save. Identity supports this via `PasswordVerificationResult.SuccessRehashNeeded`; implement a custom `IPasswordHasher<TUser>` that recognizes both formats.

**Explanation**
For inactive users whose hashes remain old, optionally wrap: `newHash = Argon2(oldHash)` and mark the format, then unwrap on login. `SignInManager` automatically rehashes when it gets `SuccessRehashNeeded`.

**Gotchas / follow-ups**
- Mark the hash version in the stored string.
- Don't log passwords or hashes.

**Code**
```csharp
public class UpgradingHasher : IPasswordHasher<IdentityUser>
{
    private readonly PasswordHasher<IdentityUser> _inner = new();
    public string HashPassword(IdentityUser u, string p) => "v2:" + _inner.HashPassword(u, p); // new format
    public PasswordVerificationResult VerifyHashedPassword(IdentityUser u, string stored, string p)
    {
        if (stored.StartsWith("v2:"))
            return _inner.VerifyHashedPassword(u, stored[3..], p);
        var r = LegacyVerify(stored, p);
        return r ? PasswordVerificationResult.SuccessRehashNeeded : PasswordVerificationResult.Failed;
    }
    private static bool LegacyVerify(string stored, string p) => false; // legacy scheme check here
}
```

### 5.5 [Tricky] User changed password but old cookie still works — why, and what is SecurityStamp?
**Answer**
Cookies are self-contained; they remain valid until validated. `SecurityStamp` is a random value on the user stored in the cookie; `SecurityStampValidator` compares it to the DB periodically (default every 30 minutes) and rejects mismatches. So there can be up to a 30-minute window unless you lower `ValidationInterval`.

**Explanation**
`UpdateSecurityStampAsync` is called on password change, role change, external login removal. Setting `ValidationInterval = TimeSpan.Zero` validates on every request at the cost of a DB hit per request.

**Gotchas / follow-ups**
- JWT bearer doesn't use this unless you check the stamp yourself.
- Data Protection keys must be shared across instances or cookies fail.

**Code**
```csharp
builder.Services.Configure<SecurityStampValidatorOptions>(o =>
    o.ValidationInterval = TimeSpan.FromMinutes(1));
await userManager.UpdateSecurityStampAsync(user); // invalidates other sessions soon
```

**Interview traps**
- Saying Identity uses bcrypt (it's PBKDF2).
- Lockout doesn't work unless `lockoutOnFailure: true`.
- Thinking a hash and an encryption are the same (hash is one-way).
- Comparing hashes with `==` (timing) rather than fixed-time.
- Complexity rules vs length/breach lists.

**Cheat sheet**
- Identity default: PBKDF2-HMAC-SHA512, 100k iterations (.NET 7+), per-password salt.
- Argon2id > bcrypt/PBKDF2 by OWASP, but PBKDF2 is built in.
- Lockout default 5 attempts / 5 min.
- SecurityStamp invalidates old cookies (30 min validation).
- `SuccessRehashNeeded` enables silent upgrades.

---

## Topic 6: Authorization Models — Role, Claims, Policy, Resource

**Study notes**
- **Role-based (RBAC)**: `[Authorize(Roles = "Admin,Manager")]` (comma = OR; stacked attributes = AND). Roles are just `ClaimTypes.Role` claims.
- **Claims-based**: check arbitrary claims (`department=HR`). Implemented as policy `RequireClaim`.
- **Policy-based**: named policy = set of **requirements** evaluated by **handlers**. The recommended abstraction; roles and claims are policy shorthand.
- **Resource-based**: decision depends on the object (e.g. "only the author can edit this document"). Cannot be done with attributes because the resource is loaded in the action — use `IAuthorizationService.AuthorizeAsync(User, resource, policyOrRequirement)`.
- Multiple requirements in a policy = AND. Multiple handlers for one requirement = OR (any success suffices, unless one calls `Fail()`).

```
Policy "CanEdit" = [ AuthenticatedRequirement, OwnerOrAdminRequirement ]
   each requirement -> handlers -> context.Succeed(req) | context.Fail()
   Result succeeds iff ALL requirements succeeded and no handler Fail()ed
```

### 6.1 [Easy] How do you restrict an endpoint by role and by claim?
**Answer**
Use `[Authorize(Roles = "Admin")]` for roles, or define a policy with `RequireClaim` and use `[Authorize(Policy = "...")]`. Roles are claims of type `ClaimTypes.Role`, so both work the same underneath.

**Explanation**
Role names in JWTs often arrive under the `roles` claim; map via `TokenValidationParameters.RoleClaimType`.

**Code**
```csharp
builder.Services.AddAuthorization(o =>
{
    o.AddPolicy("HrOnly", p => p.RequireClaim("department", "HR"));
    o.AddPolicy("AdminOrManager", p => p.RequireRole("Admin", "Manager"));
});

app.MapGet("/hr", () => "ok").RequireAuthorization("HrOnly");

[Authorize(Roles = "Admin")]
[Authorize(Roles = "Auditor")] // AND with the line above: needs both roles
public class ReportsController : ControllerBase { }
```

### 6.2 [Follow-up] Why prefer policies over roles?
**Answer**
Policies express business rules (age, tenant, subscription) and can combine requirements and run custom logic, while roles are coarse. They also decouple code from role names: change what satisfies a policy centrally without touching every controller.

**Explanation**
Controllers reference intent ("CanApproveInvoice"); the policy definition maps that to roles/claims/assertions. Easier to test and evolve.

**Code**
```csharp
builder.Services.AddAuthorizationBuilder()
    .AddPolicy("CanApproveInvoice", p => p
        .RequireAuthenticatedUser()
        .RequireRole("Finance")
        .RequireAssertion(ctx => ctx.User.HasClaim(c => c.Type == "approval_limit")));
```

### 6.3 [Medium] How does resource-based authorization work and why can't it be an attribute?
**Answer**
Attributes run before the action, when the resource isn't loaded. In resource-based authorization you load the resource, then call `IAuthorizationService.AuthorizeAsync(user, resource, requirement/policy)` and return `Forbid()` if it fails. A handler (`AuthorizationHandler<TRequirement, TResource>`) contains the rule.

**Explanation**
In minimal APIs inject `IAuthorizationService` as a parameter. Return 404 instead of 403 when the existence of the resource should be hidden.

**Code**
```csharp
public record Document(int Id, string OwnerId);

app.MapGet("/docs/{id:int}", async (int id, ClaimsPrincipal user, IAuthorizationService authz, AppDb db) =>
{
    var doc = await db.Documents.FindAsync(id);
    if (doc is null) return Results.NotFound();
    var result = await authz.AuthorizeAsync(user, doc, "DocumentOwner");
    return result.Succeeded ? Results.Ok(doc) : Results.Forbid();
}).RequireAuthorization();
```

### 6.4 [Hard] If a policy has two requirements and one requirement has two handlers, how is the final result computed?
**Answer**
All requirements must succeed (AND). For each requirement, it's satisfied if at least one handler calls `context.Succeed(requirement)`; a handler that calls `context.Fail()` forces overall failure regardless of other handlers. A handler that does nothing neither succeeds nor fails.

**Explanation**
`AuthorizationHandlerContext` tracks `PendingRequirements`. Default `InvokeHandlersAfterFailure = true` means all handlers run even after a failure (useful for logging); set false to short-circuit.

**Gotchas / follow-ups**
- Forgetting to call `Succeed` returns failure (no handler = fail).
- `Fail()` is rarely needed; mostly just don't call `Succeed`.

**Code**
```csharp
builder.Services.AddAuthorization(o => o.InvokeHandlersAfterFailure = false);
// Handler A succeeds, Handler B calls Fail() => policy fails.
```

### 6.5 [Tricky] `[Authorize(Roles="Admin")]` doesn't work with my JWT that has `"roles": ["Admin"]`. Why?
**Answer**
The role check looks for the claim type configured as `RoleClaimType` (default `ClaimTypes.Role`, the long URI). The default inbound claim map renames `role` (singular) to that URI but not `roles` (plural, as Entra ID emits), so the claim keeps its short name and role checks fail. Set `RoleClaimType = "roles"`.

**Explanation**
Arrays become multiple claims. Check what you actually received by dumping `User.Claims`. `MapInboundClaims = false` keeps raw JWT names for predictability, after which you must set `RoleClaimType`/`NameClaimType` explicitly.

**Gotchas / follow-ups**
- Inbound mapping behaviour has differed between `JwtSecurityTokenHandler` and `JsonWebTokenHandler`; verify by dumping claims.

**Code**
```csharp
.AddJwtBearer(o =>
{
    o.MapInboundClaims = false;
    o.TokenValidationParameters.RoleClaimType = "roles";
    o.TokenValidationParameters.NameClaimType = "name";
});
```

**Interview traps**
- Comma in `Roles` = OR; separate attributes = AND.
- Resource-based auth can't be an attribute.
- No handler succeeded ⇒ failure (deny by default).
- Claim-type mapping mismatches with JWT roles.

**Cheat sheet**
- Roles/claims are policy shorthand.
- Requirements AND; handlers per requirement OR.
- Resource auth: `IAuthorizationService.AuthorizeAsync(user, resource, policy)`.
- Default deny.
- Set `RoleClaimType` for JWTs.

---

## Topic 7: Custom Handlers, Attributes and Fallback Policy

**Study notes**
- **Requirement**: marker class implementing `IAuthorizationRequirement` carrying parameters. **Handler**: `AuthorizationHandler<TRequirement>` (or `<TRequirement, TResource>`); override `HandleRequirementAsync`. Register as `IAuthorizationHandler` (usually singleton/scoped — scoped needed if it uses DbContext).
- **Minimal API**: `.RequireAuthorization(...)`, `.AllowAnonymous()`, `.RequireAuthorization(policy => ...)`; groups: `MapGroup("/api").RequireAuthorization()`.
- **Fallback policy**: applied to endpoints with *no* authorization metadata; **Default policy**: used by bare `[Authorize]`. Setting fallback to `RequireAuthenticatedUser()` gives "secure by default".
- Custom policy providers (`IAuthorizationPolicyProvider`) allow dynamic policies like `[MinimumAge(21)]`.

### 7.1 [Easy] How do you write a custom authorization requirement and handler?
**Answer**
Define a requirement class, a handler deriving from `AuthorizationHandler<T>` that calls `context.Succeed(requirement)` when the rule holds, register the handler in DI, and add a policy that includes the requirement.

**Explanation**
The handler reads claims from `context.User`. If the rule isn't met, simply return without succeeding.

**Code**
```csharp
public class MinimumAgeRequirement(int minAge) : IAuthorizationRequirement
{
    public int MinAge { get; } = minAge;
}

public class MinimumAgeHandler : AuthorizationHandler<MinimumAgeRequirement>
{
    protected override Task HandleRequirementAsync(AuthorizationHandlerContext ctx, MinimumAgeRequirement req)
    {
        var dob = ctx.User.FindFirst("birthdate")?.Value;
        if (DateTime.TryParse(dob, out var d) && d.AddYears(req.MinAge) <= DateTime.Today)
            ctx.Succeed(req);
        return Task.CompletedTask;
    }
}

builder.Services.AddSingleton<IAuthorizationHandler, MinimumAgeHandler>();
builder.Services.AddAuthorizationBuilder().AddPolicy("Over18", p => p.AddRequirements(new MinimumAgeRequirement(18)));
```

### 7.2 [Follow-up] What are the fallback and default policies?
**Answer**
The default policy applies to a plain `[Authorize]` (default: authenticated user). The fallback policy applies to endpoints that have no authorization metadata at all (default: none, i.e. anonymous allowed). Setting `FallbackPolicy` to require an authenticated user makes every endpoint protected unless it opts out with `[AllowAnonymous]`.

**Explanation**
Secure-by-default prevents forgotten attributes. Remember to `AllowAnonymous` login, health and static endpoints.

**Code**
```csharp
builder.Services.AddAuthorizationBuilder()
    .SetFallbackPolicy(new AuthorizationPolicyBuilder().RequireAuthenticatedUser().Build())
    .SetDefaultPolicy(new AuthorizationPolicyBuilder().RequireAuthenticatedUser().RequireClaim("tenant").Build());

app.MapGet("/health", () => "ok").AllowAnonymous();
app.MapGet("/data", () => "needs auth"); // protected by fallback
```

### 7.3 [Medium] Resource-based handler example with an operation requirement.
**Answer**
Use `OperationAuthorizationRequirement` (or a custom one) with `AuthorizationHandler<TReq, TResource>`. The handler compares the resource's owner id with the user's id claim. Call it via `AuthorizeAsync(user, resource, Operations.Edit)`.

**Explanation**
Keeps the access rule in one testable class instead of scattered `if` statements across controllers.

**Code**
```csharp
using Microsoft.AspNetCore.Authorization.Infrastructure;

public static class Operations
{
    public static readonly OperationAuthorizationRequirement Edit = new() { Name = nameof(Edit) };
}

public class DocumentOwnerHandler : AuthorizationHandler<OperationAuthorizationRequirement, Document>
{
    protected override Task HandleRequirementAsync(AuthorizationHandlerContext ctx,
        OperationAuthorizationRequirement req, Document doc)
    {
        if (req.Name == nameof(Operations.Edit) &&
            doc.OwnerId == ctx.User.FindFirstValue(ClaimTypes.NameIdentifier))
            ctx.Succeed(req);
        return Task.CompletedTask;
    }
}
// await authz.AuthorizeAsync(User, doc, Operations.Edit);
```

### 7.4 [Hard] Handler lifetime pitfalls: using a scoped DbContext inside a handler.
**Answer**
Authorization handlers registered as singletons can't take a scoped `DbContext` (captive dependency / validation error). Register the handler as scoped (`AddScoped<IAuthorizationHandler, ...>`). Handlers are resolved per-request, so scoped works.

**Explanation**
With `ValidateScopes` on in Development, a singleton consuming scoped fails at startup/first use. Alternatively inject `IServiceScopeFactory`. Also avoid blocking I/O: override the async method and `await`.

**Gotchas / follow-ups**
- A handler that calls the DB on every request needs caching.
- Handlers shouldn't write response/redirect; they only decide.

**Code**
```csharp
public class SubscriptionHandler(AppDb db) : AuthorizationHandler<ActiveSubscriptionRequirement>
{
    protected override async Task HandleRequirementAsync(AuthorizationHandlerContext ctx, ActiveSubscriptionRequirement req)
    {
        var id = ctx.User.FindFirstValue(ClaimTypes.NameIdentifier);
        if (id != null && await db.Subscriptions.AnyAsync(s => s.UserId == id && s.ExpiresUtc > DateTime.UtcNow))
            ctx.Succeed(req);
    }
}
public class ActiveSubscriptionRequirement : IAuthorizationRequirement { }
builder.Services.AddScoped<IAuthorizationHandler, SubscriptionHandler>();
```

### 7.5 [Tricky] Minimal API groups: how do `RequireAuthorization` and `AllowAnonymous` interact on a group and endpoint?
**Answer**
Group metadata is added to every endpoint inside. `AllowAnonymous` on an individual endpoint overrides the group's `RequireAuthorization`, since `IAllowAnonymous` metadata short-circuits. Policies add up: group policy plus endpoint policy both must be satisfied.

**Explanation**
Because multiple `IAuthorizeData` metadata items are combined into one effective policy (AND), an endpoint cannot "relax" a group's policy with a different `RequireAuthorization` — only `AllowAnonymous` can bypass.

**Gotchas / follow-ups**
- Order of `.RequireAuthorization()` vs `.AllowAnonymous()` doesn't matter.

**Code**
```csharp
var api = app.MapGroup("/api").RequireAuthorization("AdminOnly");
api.MapGet("/stats", () => "admins only");
api.MapGet("/ping", () => "open").AllowAnonymous();        // bypasses group policy
api.MapGet("/extra", () => "both").RequireAuthorization("Over18"); // needs AdminOnly AND Over18
```

**Interview traps**
- Singleton handler with scoped DbContext.
- Fallback policy blocks health checks/login pages unless `AllowAnonymous`.
- Failing to register the handler ⇒ the policy always fails.
- Assuming a second `[Authorize]` loosens the first.

**Cheat sheet**
- Requirement = data; handler = logic; register handler in DI.
- Fallback = endpoints without metadata; default = bare `[Authorize]`.
- Resource handlers: `AuthorizationHandler<TReq,TRes>`.
- Group `RequireAuthorization` stacks (AND); `AllowAnonymous` overrides.

---

## Topic 8: CORS

**Study notes**
- **Same-Origin Policy**: browser blocks JS from reading cross-origin responses. Origin = scheme + host + port. **CORS** is a browser-enforced opt-in via response headers; it is **not** server-side access control (curl/Postman ignore it).
- **Simple requests** (GET/HEAD/POST with safe headers and content types `text/plain`, `form-urlencoded`, `multipart`) go directly; the browser checks `Access-Control-Allow-Origin` on the response.
- **Preflight**: for other methods (PUT/DELETE/PATCH), custom headers (`Authorization`, `Content-Type: application/json`), the browser first sends `OPTIONS` with `Origin`, `Access-Control-Request-Method`, `Access-Control-Request-Headers`. Server replies `Access-Control-Allow-Origin/Methods/Headers` (+ `Max-Age` cache).
- Credentials (cookies/auth) need `Access-Control-Allow-Credentials: true` and a **specific** origin (not `*`).
- ASP.NET Core: `AddCors` + `UseCors` (after `UseRouting`, before `UseAuthentication/UseAuthorization`).

```
Browser (https://app.com)                 API (https://api.com)
  OPTIONS /orders  Origin: app.com   -->
        <-- 204 ACAO: app.com, ACAM: PUT, ACAH: authorization
  PUT /orders  Authorization: ...    -->
        <-- 200 ACAO: app.com
```

### 8.1 [Easy] What is CORS and why does it exist?
**Answer**
CORS is a browser mechanism that lets a server declare which other origins may read its responses, relaxing the same-origin policy. It exists because browsers attach cookies automatically and would otherwise let any site read your authenticated data.

**Explanation**
CORS doesn't stop requests from being sent (a simple request still reaches the server); it stops the page's JS from reading the response.

**Code**
```csharp
builder.Services.AddCors(o => o.AddPolicy("Spa", p => p
    .WithOrigins("https://app.example.com")
    .WithMethods("GET", "POST", "PUT")
    .WithHeaders("Content-Type", "Authorization")));

var app = builder.Build();
app.UseRouting();
app.UseCors("Spa");
app.UseAuthentication();
app.UseAuthorization();
```

### 8.2 [Follow-up] What is a preflight request and when does it happen?
**Answer**
A preflight is an automatic `OPTIONS` request the browser sends before a "non-simple" cross-origin request to ask whether the method and headers are allowed. It happens for methods like PUT/DELETE or headers such as `Authorization` and `Content-Type: application/json`. The result can be cached with `SetPreflightMaxAge`.

**Explanation**
ASP.NET Core's CORS middleware answers preflights itself when a matching policy exists. An authorization-protected endpoint must not reject OPTIONS before CORS runs, hence `UseCors` before auth.

**Code**
```csharp
p.WithOrigins("https://app.example.com")
 .AllowAnyHeader().AllowAnyMethod()
 .SetPreflightMaxAge(TimeSpan.FromMinutes(10));
```

### 8.3 [Medium] Why is `AllowAnyOrigin()` + `AllowCredentials()` rejected, and what is the secure way to allow dynamic origins?
**Answer**
The CORS spec forbids `Access-Control-Allow-Origin: *` with credentials, and ASP.NET Core throws an `InvalidOperationException` for that combination. Use an explicit origin list; for subdomains use `SetIsOriginAllowedToAllowWildcardSubdomains()` with a strict pattern, or `SetIsOriginAllowed` with an allow-list check.

**Explanation**
Blindly reflecting the `Origin` header (`SetIsOriginAllowed(_ => true)` + credentials) is equivalent to allowing every site to read authenticated responses — a serious misconfiguration.

**Code**
```csharp
builder.Services.AddCors(o => o.AddPolicy("Tenants", p => p
    .WithOrigins("https://*.example.com")
    .SetIsOriginAllowedToAllowWildcardSubdomains()
    .AllowCredentials()
    .WithHeaders("Content-Type")));
```

### 8.4 [Hard] Common CORS misconfigurations and why "CORS error" can be a server crash or redirect.
**Answer**
Mistakes: reflecting any origin with credentials, trusting `null` origin, loose regex (`example.com` matching `evilexample.com`), `UseCors` after auth/after endpoints, or missing the policy on errors. A 500 exception or an auth redirect on the preflight response lacks CORS headers, so the browser reports a CORS error even though the real cause is elsewhere.

**Explanation**
Check Network tab: look at the OPTIONS response status and headers. Exception-handler middleware should run so that CORS headers are still added (place `UseCors` early; exceptions thrown after CORS ran keep headers on most responses).

**Gotchas / follow-ups**
- `WithExposedHeaders` is needed for JS to read custom response headers.
- CORS ≠ CSRF protection: simple POST forms still reach the server.

**Code**
```csharp
p.SetIsOriginAllowed(origin =>
{
    var host = new Uri(origin).Host;
    return host == "example.com" || host.EndsWith(".example.com", StringComparison.Ordinal);
});
```

### 8.5 [Tricky] If CORS blocks the browser, is the API protected? A malicious site POSTs a form to your API — what happens?
**Answer**
No. CORS only controls whether JS can read responses; the request is still sent and processed (for simple requests), and HTML form posts aren't subject to CORS at all. State-changing endpoints still need CSRF protection and real authorization.

**Explanation**
Non-browser clients ignore CORS. Therefore never rely on CORS as an authentication/authorization mechanism.

**Gotchas / follow-ups**
- Requiring `Content-Type: application/json` or a custom header forces a preflight, which implicitly blocks cross-site forms — but verify rather than rely on it.

**Code**
```csharp
// Default [ApiController] binding rejects form-encoded bodies for [FromBody] JSON endpoints (415),
// which helps but is not a CSRF defence by itself.
app.MapPost("/transfer", ([FromBody] TransferDto dto) => Results.Ok()).RequireAuthorization();
public record TransferDto(decimal Amount, string To);
```

**Interview traps**
- "CORS secures my API" — no.
- `*` with credentials is illegal.
- `UseCors` order relative to auth.
- Preflight returning 401 because auth ran first.
- Regex origin checks not anchored.

**Cheat sheet**
- Origin = scheme+host+port.
- CORS = browser read-permission, not access control.
- Preflight = OPTIONS for non-simple requests.
- Credentials ⇒ explicit origin.
- `UseCors` after `UseRouting`, before auth.

---

## Topic 9: CSRF and Antiforgery

**Study notes**
- **CSRF**: attacker site makes the victim's browser send an authenticated request (cookies auto-attached) to your site. Relevant when auth uses **ambient credentials** (cookies, Windows auth). Not relevant to `Authorization: Bearer` headers added by JS.
- Defences: **antiforgery tokens** (synchronizer / double-submit with signed tokens), **SameSite cookies** (Lax is the browser default; Strict/Lax reduce risk), custom header checks, requiring re-auth for critical actions.
- ASP.NET Core: `AddAntiforgery`, `[ValidateAntiForgeryToken]`, `[AutoValidateAntiforgeryToken]` (all unsafe methods), `[IgnoreAntiforgeryToken]`. Razor forms/Tag helpers emit hidden `__RequestVerificationToken` and a cookie; Razor Pages enable it by default. For SPAs: `IAntiforgery.GetAndStoreTokens` and send in header `X-CSRF-TOKEN`.
- Minimal APIs: antiforgery validation automatically applied for form-bound endpoints (.NET 8) when `AddAntiforgery` and `UseAntiforgery()` are used.
- Tokens: a cookie token + a request token; both bound to the user identity; protected by Data Protection.

### 9.1 [Easy] What is CSRF and when does it apply?
**Answer**
Cross-Site Request Forgery tricks a logged-in user's browser into sending an unwanted request to a site that trusts its cookies. It applies to cookie-based (ambient) authentication; header-based bearer tokens aren't auto-sent so aren't vulnerable.

**Explanation**
Typical attack: a hidden form on evil.com POSTs to bank.com/transfer. The browser attaches bank.com cookies.

**Code**
```csharp
[AutoValidateAntiforgeryToken] // validates POST/PUT/DELETE/PATCH automatically
public class AccountController : Controller
{
    [HttpPost] public IActionResult Transfer(decimal amount) => Ok();
}
```

### 9.2 [Follow-up] How do antiforgery tokens work in a Razor form and in an SPA?
**Answer**
Razor's form tag helper adds a hidden request token field and the framework sets a cookie token; on POST both must match and belong to the current user. In an SPA, an endpoint returns a request token via `IAntiforgery.GetAndStoreTokens`, and JS sends it in the `X-CSRF-TOKEN` header while the cookie goes automatically.

**Explanation**
An attacker site can trigger the cookie being sent but can't read the page/token due to same-origin policy, so cannot supply the matching request token.

**Code**
```csharp
builder.Services.AddAntiforgery(o => o.HeaderName = "X-CSRF-TOKEN");
app.UseAntiforgery(); // .NET 8, after UseAuthentication/UseAuthorization

app.MapGet("/csrf", (IAntiforgery af, HttpContext ctx) =>
{
    var t = af.GetAndStoreTokens(ctx);
    return Results.Ok(new { token = t.RequestToken });
});

app.MapPost("/submit", ([FromForm] string note) => Results.Ok(note)); // validated automatically in .NET 8
```

### 9.3 [Medium] Is SameSite=Lax enough? What does it block?
**Answer**
`Lax` blocks cookies on cross-site subrequests and POST navigations but allows them on top-level GET navigations, so it mitigates most CSRF but not state-changing GETs, and doesn't cover same-site (sibling subdomain) attacks. `Strict` blocks all cross-site sending but harms UX on inbound links. Combine SameSite with antiforgery tokens for defence in depth.

**Explanation**
ASP.NET Core cookie auth default is `SameSite=Lax`. `None` (required for cross-site embedded/OIDC flows) must also be `Secure`.

**Code**
```csharp
builder.Services.ConfigureApplicationCookie(o =>
{
    o.Cookie.SameSite = SameSiteMode.Lax;
    o.Cookie.SecurePolicy = CookieSecurePolicy.Always;
});
```

### 9.4 [Hard] Why is "never use GET for state changes" a CSRF rule, and how do antiforgery tokens bind to a user?
**Answer**
GET requests are not validated by default and are triggered by `<img>` tags or links, and Lax cookies are sent on top-level GET navigations. The antiforgery token embeds the user's identity (name claim/`sub`); a token generated for user A fails for user B, and anonymous tokens fail after login.

**Explanation**
After login the identity changes, so the antiforgery token must be regenerated: a common bug — "antiforgery token not valid" on the login page → submit loop. Tokens also depend on Data Protection keys, so multi-instance apps need shared keys.

**Gotchas / follow-ups**
- Check `IAntiforgeryAdditionalDataProvider` for extra binding.
- Behind web farms: shared key ring (Topic 11).

**Code**
```csharp
[HttpGet("delete/{id}")]          // WRONG: state change via GET
public IActionResult DeleteBad(int id) => Ok();

[HttpPost("delete/{id}")]         // RIGHT
[ValidateAntiForgeryToken]
public IActionResult Delete(int id) => Ok();
```

### 9.5 [Tricky] Does a JSON API with JWT in `Authorization` header need antiforgery? What about a JSON API using cookies?
**Answer**
JWT in a header: no, because the browser doesn't attach it automatically. Cookie-authenticated JSON API: yes (or equivalent), since cross-site requests can still carry the cookie; relying on `Content-Type: application/json` forcing a preflight is fragile, so use antiforgery header tokens and SameSite.

**Explanation**
Beware: if a cookie-auth app also accepts bearer tokens on the same endpoints, requests authenticated by cookie remain CSRF-able.

**Gotchas / follow-ups**
- Mixed schemes: apply antiforgery to cookie-authenticated paths only.
- `[IgnoreAntiforgeryToken]` on JWT-only endpoints is fine.

**Code**
```csharp
app.MapPost("/api/pay", (PayDto d) => Results.Ok())
   .RequireAuthorization(new AuthorizeAttribute { AuthenticationSchemes = "Bearer" })
   .DisableAntiforgery(); // .NET 8 minimal API: opt out for bearer-only endpoint
public record PayDto(decimal Amount);
```

**Interview traps**
- Thinking CORS prevents CSRF.
- Bearer tokens being "CSRF vulnerable" (they aren't, but XSS is the risk).
- Login-flow token mismatch after identity change.
- Forgetting shared Data Protection keys breaks tokens across servers.

**Cheat sheet**
- CSRF = ambient credentials abused.
- Antiforgery = cookie token + request token (header/hidden field).
- SameSite Lax default; defence in depth.
- GET must be safe.
- Bearer-header APIs: no CSRF.

---

## Topic 10: XSS, SQL Injection and Mass Assignment

**Study notes**
- **XSS**: injected script runs in victim's browser. Types: stored, reflected, DOM-based. Defence: **context-sensitive output encoding** (HTML, attribute, JS, URL), CSP, validate input, avoid `Html.Raw`/`innerHTML`, sanitize rich HTML with an allow-list sanitizer (e.g., HtmlSanitizer package).
- Razor auto-encodes `@value` (HTML-encode). `@Html.Raw`, `MvcHtmlString`/`HtmlString` bypass encoding. Encoders: `HtmlEncoder`, `JavaScriptEncoder`, `UrlEncoder` (System.Text.Encodings.Web).
- **SQL injection**: untrusted input concatenated into SQL. Defence: **parameterized queries**; EF Core LINQ is parameterized; `FromSqlInterpolated`/`FromSql` (interpolated → parameters) vs `FromSqlRaw` with concatenation (dangerous). Least-privilege DB account; never build identifiers from input — allow-list them.
- **Mass assignment / over-posting**: model binder populates properties the client shouldn't control (`IsAdmin`). Defence: use DTOs/view models, `[BindNever]`, `[Bind]`, or `required`/explicit mapping.

### 10.1 [Easy] How do you prevent SQL injection in ASP.NET Core with EF Core / ADO.NET?
**Answer**
Use parameterized queries: LINQ, `FromSqlInterpolated`, or `SqlParameter`s. Never concatenate user input into SQL strings. Also grant the DB login least privilege.

**Explanation**
Parameters are sent separately from the SQL text so input is treated as data, never parsed as SQL.

**Code**
```csharp
// Safe: interpolated string becomes a parameter
var users = await db.Users.FromSql($"SELECT * FROM Users WHERE Name = {name}").ToListAsync();

// ADO.NET
using var cmd = new SqlCommand("SELECT * FROM Users WHERE Name = @name", conn);
cmd.Parameters.Add("@name", SqlDbType.NVarChar, 100).Value = name;
```

### 10.2 [Follow-up] What is the difference between `FromSqlRaw` and `FromSqlInterpolated`/`FromSql`?
**Answer**
`FromSqlInterpolated` and `FromSql` (EF Core 7+) take an interpolated string and turn each hole into a DB parameter. `FromSqlRaw` takes a plain string and you must supply parameters yourself; passing an interpolated/concatenated string to it is injectable.

**Explanation**
`FromSql($"...")` with a `FormattableString` is safe, but `FromSql(someStringVariable)` (non-interpolated string) won't compile for `FromSql` — that's a deliberate design. Analyzer EF1002 warns about risky `FromSqlRaw` use.

**Code**
```csharp
string name = "x'; DROP TABLE Users;--";
// UNSAFE: injection
var bad = db.Users.FromSqlRaw($"SELECT * FROM Users WHERE Name = '{name}'");
// SAFE
var ok1 = db.Users.FromSqlRaw("SELECT * FROM Users WHERE Name = {0}", name);
var ok2 = db.Users.FromSqlInterpolated($"SELECT * FROM Users WHERE Name = {name}");
```

### 10.3 [Medium] Does Razor protect me from XSS? Where does it not?
**Answer**
Razor HTML-encodes `@expression` output by default, so reflected/stored text in HTML body and quoted attributes is safe. It doesn't protect when you use `Html.Raw`, write into `<script>` blocks, put user data in unquoted/`href`/`javascript:` URLs, or build HTML in JS (`innerHTML`). Use the context-appropriate encoder.

**Explanation**
Encoding is context-specific: HTML-encoding inside a JS string isn't enough. Validate URL schemes (`http/https` only) before emitting in `href`.

**Code**
```csharp
using System.Text.Encodings.Web;

var enc = HtmlEncoder.Default;
Console.WriteLine(enc.Encode("<script>alert(1)</script>"));
// Output: &lt;script&gt;alert(1)&lt;/script&gt;
Console.WriteLine(JavaScriptEncoder.Default.Encode("</script>"));
// Output: \u003C/script\u003E
```

### 10.4 [Hard] How do you defend against injection when the column/table name or sort order comes from the user?
**Answer**
Parameters can't bind identifiers, so use an allow-list that maps user input to known column names (or a `switch`/dictionary), never splice raw input. With EF Core, prefer building the `OrderBy` from a whitelist of expressions.

**Explanation**
For dynamic LINQ, map strings to `Expression<Func<T,object>>`. Reject unknown values with 400.

**Gotchas / follow-ups**
- Stored procedures built with dynamic SQL inside are still injectable.
- Second-order injection: data stored safely but later concatenated into SQL.
- `LIKE` wildcards in parameters: escape `%`/`_` if semantics matter.

**Code**
```csharp
static readonly Dictionary<string, Expression<Func<Product, object>>> Sorts = new(StringComparer.OrdinalIgnoreCase)
{
    ["name"] = p => p.Name,
    ["price"] = p => p.Price
};

IQueryable<Product> Sort(IQueryable<Product> q, string? col)
    => Sorts.TryGetValue(col ?? "name", out var key) ? q.OrderBy(key)
       : throw new ArgumentException("Invalid sort column");
public record Product(int Id, string Name, decimal Price);
```

### 10.5 [Tricky] What is over-posting/mass assignment and how do you prevent it?
**Answer**
If you bind a request directly to an entity, a client can add extra JSON/form fields (`"isAdmin": true`) that the binder sets. Prevent by binding to purpose-built input DTOs/view models containing only allowed fields, mapping explicitly, or marking sensitive properties `[BindNever]`.

**Explanation**
Also applies to EF `Update(entity)` with attached full graphs, which marks all properties as modified. Load the entity and copy only allowed fields.

**Gotchas / follow-ups**
- `System.Text.Json` ignores unknown properties by default; `[JsonIgnore]` on server-only fields prevents deserialization into them.
- Don't expose entities as API models; also avoids data over-exposure in responses.

**Code**
```csharp
public class User { public int Id { get; set; } public string Name { get; set; } = ""; public bool IsAdmin { get; set; } }
public record UpdateProfileDto(string Name);

app.MapPut("/profile/{id:int}", async (int id, UpdateProfileDto dto, AppDb db) =>
{
    var u = await db.Users.FindAsync(id);
    if (u is null) return Results.NotFound();
    u.Name = dto.Name;              // IsAdmin cannot be altered
    await db.SaveChangesAsync();
    return Results.NoContent();
});
```

**Interview traps**
- "EF Core is immune to SQL injection" — not with `FromSqlRaw`+concatenation.
- `Html.Raw` and JS contexts.
- Input validation alone ≠ output encoding.
- Binding entities directly to requests.
- Identifiers can't be parameterized.

**Cheat sheet**
- Encode output per context; Razor encodes by default.
- Parameterize everything; allow-list identifiers.
- `FromSql($"")` safe; `FromSqlRaw` + concat unsafe.
- DTOs prevent over-posting.
- CSP is defence in depth for XSS.

---

## Topic 11: Data Protection API, HTTPS/HSTS and Security Headers

**Study notes**
- **Data Protection API (DPAPI-like, `Microsoft.AspNetCore.DataProtection`)**: symmetric encryption + MAC for short-lived app data: auth cookies, antiforgery tokens, TempData, `Protect()` payloads. Not for long-term storage of data you must decrypt years later.
- **Key ring**: keys auto-generated, default 90-day lifetime, auto-rotated; old keys retained for decrypting. Default algorithm AES-256-CBC + HMACSHA256 (`AuthenticatedEncryptorConfiguration`).
- Default key storage: on Windows `%LOCALAPPDATA%\ASP.NET\DataProtection-Keys` (IIS: registry); in containers/web farms they're lost/not shared → must persist and share: `PersistKeysToFileSystem`, `PersistKeysToAzureBlobStorage`, `PersistKeysToDbContext<T>` (EF Core), `PersistKeysToStackExchangeRedis`; protect keys at rest with `ProtectKeysWithAzureKeyVault` / certificate / DPAPI. Set `SetApplicationName` consistently across apps sharing keys.
- **Purpose strings** isolate protectors: `provider.CreateProtector("Orders.Tokens.v1")`; data protected under one purpose can't be unprotected under another.
- **HTTPS**: `UseHttpsRedirection()` (redirects HTTP→HTTPS, 307/308), **HSTS**: `UseHsts()` (non-Development; sends `Strict-Transport-Security: max-age=...; includeSubDomains; preload`). TLS versions: TLS 1.2+ (prefer 1.3). Behind a reverse proxy use `UseForwardedHeaders`.
- **Security headers**: `Content-Security-Policy`, `X-Content-Type-Options: nosniff`, `X-Frame-Options`/CSP `frame-ancestors`, `Referrer-Policy`, `Permissions-Policy`. ASP.NET Core doesn't add these by default — add middleware.

### 11.1 [Easy] What is the Data Protection API used for in ASP.NET Core?
**Answer**
It's a built-in crypto service that encrypts and signs data such as auth cookies, antiforgery tokens and TempData. You can also use `IDataProtector.Protect/Unprotect` directly for tamper-proof tokens. It manages key creation and rotation for you.

**Explanation**
Don't confuse with password hashing — it's reversible encryption designed for transient data.

**Code**
```csharp
using Microsoft.AspNetCore.DataProtection;

var provider = DataProtectionProvider.Create("MyApp");
var protector = provider.CreateProtector("Orders.v1");
string cipher = protector.Protect("order-123");
Console.WriteLine(protector.Unprotect(cipher)); // Output: order-123
```

### 11.2 [Follow-up] Why do users get logged out / "antiforgery token invalid" after deploying to multiple instances or containers?
**Answer**
Each instance generated its own key ring, so cookies encrypted by one can't be decrypted by another (or keys vanish on container restart). Persist the key ring to shared storage and use the same application name; protect keys at rest.

**Explanation**
Slot swaps in Azure App Service share keys; Docker/Kubernetes pods don't unless you configure external storage.

**Code**
```csharp
builder.Services.AddDataProtection()
    .SetApplicationName("MyApp")
    .PersistKeysToAzureBlobStorage(new Uri(builder.Configuration["Dp:BlobUri"]!), new DefaultAzureCredential())
    .ProtectKeysWithAzureKeyVault(new Uri(builder.Configuration["Dp:KeyUri"]!), new DefaultAzureCredential());
// Packages: Azure.Extensions.AspNetCore.DataProtection.Blobs / .Keys, Azure.Identity
```

### 11.3 [Medium] Explain purpose strings and time-limited protection.
**Answer**
A purpose string creates an isolated protector, so a payload from one feature can't be replayed into another. `ToTimeLimitedDataProtector()` embeds an expiry and fails to unprotect after it, useful for reset links.

**Explanation**
Use hierarchical purposes (`App.Feature.Version`). Time-limited uses the `ITimeLimitedDataProtector` extension.

**Code**
```csharp
using Microsoft.AspNetCore.DataProtection;

var dp = DataProtectionProvider.Create("MyApp");
var timed = dp.CreateProtector("Reset.v1").ToTimeLimitedDataProtector();
string token = timed.Protect("user-7", TimeSpan.FromMinutes(15));
Console.WriteLine(timed.Unprotect(token)); // Output: user-7
// After 15 minutes Unprotect throws CryptographicException
```

### 11.4 [Hard] How do HSTS and HTTPS redirection differ, and what are the pitfalls?
**Answer**
Redirection sends 307/308 from HTTP to HTTPS on each request, leaving the first request exposed to downgrade/MITM. HSTS tells the browser (after one secure visit) to always use HTTPS for the host for `max-age`, preventing that. `UseHsts` is intentionally not active in Development, and `preload` + long max-age is hard to undo.

**Explanation**
Browsers ignore HSTS headers sent over HTTP and for localhost-ish hosts. Behind a TLS-terminating proxy, forwarded headers must be configured or redirect loops occur.

**Gotchas / follow-ups**
- Start with a short `MaxAge`, then increase.
- Include subdomains only if all subdomains support HTTPS.

**Code**
```csharp
builder.Services.AddHsts(o => { o.MaxAge = TimeSpan.FromDays(365); o.IncludeSubDomains = true; o.Preload = true; });
builder.Services.Configure<ForwardedHeadersOptions>(o =>
    o.ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto);

var app = builder.Build();
app.UseForwardedHeaders();
if (!app.Environment.IsDevelopment()) app.UseHsts();
app.UseHttpsRedirection();
```

### 11.5 [Tricky] Add security headers and a CSP — what are the XSS-relevant pieces, and what breaks?
**Answer**
Add middleware that sets `Content-Security-Policy` (e.g. `default-src 'self'; script-src 'self'; object-src 'none'; frame-ancestors 'none'`), `X-Content-Type-Options: nosniff`, and `Referrer-Policy`. CSP blocks inline scripts unless you use nonces/hashes, which breaks apps that rely on inline `<script>` or `onclick`.

**Explanation**
Roll out with `Content-Security-Policy-Report-Only` first. Use per-request nonces for needed inline scripts. Avoid `'unsafe-inline'`/`'unsafe-eval'`, which neuter protection.

**Gotchas / follow-ups**
- Don't forget `Server` header removal: `AddServerHeader = false` on Kestrel.
- Headers must be added before the response starts.

**Code**
```csharp
app.Use(async (ctx, next) =>
{
    ctx.Response.Headers.Append("Content-Security-Policy",
        "default-src 'self'; script-src 'self'; object-src 'none'; frame-ancestors 'none'");
    ctx.Response.Headers.Append("X-Content-Type-Options", "nosniff");
    ctx.Response.Headers.Append("Referrer-Policy", "strict-origin-when-cross-origin");
    await next();
});
builder.WebHost.ConfigureKestrel(k => k.AddServerHeader = false);
```

**Interview traps**
- Not sharing the key ring in multi-instance deployments.
- Using Data Protection for long-term archival encryption.
- HSTS only works after a first HTTPS visit.
- CSP with `'unsafe-inline'` gives false confidence.
- Redirect loops behind proxies without forwarded headers.

**Cheat sheet**
- Data Protection: auto-rotating key ring (90 days), purpose strings.
- Persist + protect keys in web farms; set ApplicationName.
- HSTS = browser memory of HTTPS-only; redirect = server nudge.
- Headers: CSP, nosniff, frame-ancestors, Referrer-Policy.
- TLS 1.2+ minimum.

---

## Topic 12: Secrets Management, API Keys and HMAC Request Signing

**Study notes**
- Never commit secrets. Hierarchy: **User Secrets** (`dotnet user-secrets`, dev only, stored in `%APPDATA%\Microsoft\UserSecrets\<id>\secrets.json` — *not encrypted*), **environment variables**, **Azure Key Vault** (`Azure.Extensions.AspNetCore.Configuration.Secrets` → `AddAzureKeyVault`), and **Managed Identity** + `DefaultAzureCredential` to avoid credentials altogether.
- Config precedence (later wins): appsettings.json → appsettings.{Env}.json → user secrets (Development) → env vars → command line.
- Key Vault specifics: secret name `--` maps to `:` in config; caching/refresh via `ReloadInterval`; RBAC roles ("Key Vault Secrets User").
- **API keys**: identify/authenticate a *client app*, not a user; send in header (not query string — logged); store only hash; support rotation (two active keys) and scopes.
- **HMAC request signing**: client signs canonical string (method, path, body hash, timestamp, nonce) with a shared secret; server recomputes and compares in constant time; reject stale timestamps and reused nonces to prevent replay. The secret itself never travels.

### 12.1 [Easy] How do you keep secrets out of source control in .NET?
**Answer**
In development use `dotnet user-secrets` or environment variables; in production use Azure Key Vault with Managed Identity (or platform secret stores). `appsettings.json` should hold only non-secret settings.

**Explanation**
User secrets are just a plain JSON file outside the repo, a dev convenience, not a security boundary. If a secret was ever committed, rotate it — removing it from history isn't enough.

**Code**
```csharp
// CLI: dotnet user-secrets init; dotnet user-secrets set "Jwt:Key" "..."
var key = builder.Configuration["Jwt:Key"]
    ?? throw new InvalidOperationException("Jwt:Key not configured");

builder.Configuration.AddAzureKeyVault(
    new Uri("https://my-vault.vault.azure.net/"), new DefaultAzureCredential());
```

### 12.2 [Follow-up] Why Managed Identity + `DefaultAzureCredential`?
**Answer**
The Azure resource gets an identity automatically, so the app authenticates to Key Vault/SQL/Storage without any stored credential. `DefaultAzureCredential` tries a chain (environment, workload/managed identity, Visual Studio/CLI login), so the same code works locally and in Azure.

**Explanation**
Eliminates the "bootstrap secret" problem. Grant least-privilege RBAC to the identity. For SQL, use `Authentication=Active Directory Default` in the connection string.

**Code**
```csharp
var client = new SecretClient(new Uri("https://my-vault.vault.azure.net/"), new DefaultAzureCredential());
KeyVaultSecret secret = await client.GetSecretAsync("DbPassword");
```

### 12.3 [Medium] Implement API-key authentication properly.
**Answer**
Use a custom authentication handler (or middleware/endpoint filter) that reads `X-Api-Key`, hashes it, looks up the hash, and produces a `ClaimsPrincipal` with client identity/scopes. Compare in constant time, store only hashes, allow rotation, and never accept keys in query strings.

**Explanation**
Using a proper `AuthenticationHandler` lets `[Authorize]`/policies work uniformly and integrates with 401 challenge handling.

**Code**
```csharp
public class ApiKeyHandler(IOptionsMonitor<AuthenticationSchemeOptions> o, ILoggerFactory l,
    UrlEncoder e, IApiKeyStore store) : AuthenticationHandler<AuthenticationSchemeOptions>(o, l, e)
{
    protected override async Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        if (!Request.Headers.TryGetValue("X-Api-Key", out var key)) return AuthenticateResult.NoResult();
        var hash = Convert.ToHexString(SHA256.HashData(Encoding.UTF8.GetBytes(key.ToString())));
        var client = await store.FindByHashAsync(hash);
        if (client is null) return AuthenticateResult.Fail("Invalid API key");
        var id = new ClaimsIdentity(new[] { new Claim(ClaimTypes.Name, client.Name) }, Scheme.Name);
        return AuthenticateResult.Success(new AuthenticationTicket(new ClaimsPrincipal(id), Scheme.Name));
    }
}
public interface IApiKeyStore { Task<ApiClient?> FindByHashAsync(string hash); }
public record ApiClient(string Name);
// builder.Services.AddAuthentication("ApiKey").AddScheme<AuthenticationSchemeOptions, ApiKeyHandler>("ApiKey", null);
```

### 12.4 [Hard] Design HMAC request signing. How do you prevent replay and timing attacks?
**Answer**
Client sends `X-Timestamp`, `X-Nonce`, `X-Signature = HMAC_SHA256(secret, method\npath\ntimestamp\nnonce\nSHA256(body))`. Server rebuilds the string, compares with `CryptographicOperations.FixedTimeEquals`, rejects timestamps outside a small window (e.g. ±5 min) and remembers nonces within that window to stop replay. Hash the body so modification invalidates the signature.

**Explanation**
Body must be buffered (`EnableBuffering`) to be hashed and still read by model binding. Use TLS as well: HMAC gives integrity/authenticity, not confidentiality.

**Gotchas / follow-ups**
- Canonicalization ambiguity (query param order, encoding) causes false failures or bypass.
- Per-client secrets; rotate; store server-side encrypted (the server needs the raw secret, unlike API-key hashes).

**Code**
```csharp
static string Sign(string secret, string method, string path, string ts, string nonce, byte[] body)
{
    string bodyHash = Convert.ToHexString(SHA256.HashData(body));
    string canonical = $"{method}\n{path}\n{ts}\n{nonce}\n{bodyHash}";
    using var h = new HMACSHA256(Encoding.UTF8.GetBytes(secret));
    return Convert.ToHexString(h.ComputeHash(Encoding.UTF8.GetBytes(canonical)));
}

static bool Verify(string expectedHex, string providedHex) =>
    CryptographicOperations.FixedTimeEquals(
        Encoding.ASCII.GetBytes(expectedHex), Encoding.ASCII.GetBytes(providedHex));
```

### 12.5 [Tricky] What is wrong with comparing signatures using `==` and with storing a secret in `appsettings.json` of a deployed container image?
**Answer**
`==` short-circuits at the first differing byte, leaking timing information that can allow byte-by-byte signature guessing; use `FixedTimeEquals`. A secret baked into an image is readable by anyone with image access and lives in every layer/history; inject at runtime via env vars/secret stores/Key Vault.

**Explanation**
Over a network the timing signal is small but practical attacks exist; it's a free fix. Also note `FixedTimeEquals` requires equal-length spans, else returns false immediately (length isn't secret).

**Gotchas / follow-ups**
- Don't log signatures, secrets or full Authorization headers.
- Configuration binding of secrets into logged option objects leaks them.

**Code**
```csharp
byte[] a = [1, 2, 3];
byte[] b = [1, 2, 3];
Console.WriteLine(CryptographicOperations.FixedTimeEquals(a, b)); // Output: True
```

**Interview traps**
- User secrets are not encrypted and not for production.
- Query-string API keys land in logs/referrers.
- Storing API keys in plaintext (store hashes).
- HMAC without timestamp/nonce is replayable.
- Key Vault `--` ↔ `:` naming.

**Cheat sheet**
- Dev: user-secrets; Prod: Key Vault + Managed Identity.
- `DefaultAzureCredential` = credential chain.
- API key = app identity; hash at rest; header only.
- HMAC: canonical string + timestamp + nonce + constant-time compare.
- Rotate; never commit.

---

## Topic 13: Rate Limiting and Brute-Force Protection

**Study notes**
- .NET 7+ built-in `Microsoft.AspNetCore.RateLimiting`: `AddRateLimiter`, `UseRateLimiter`, `[EnableRateLimiting("policy")]` / `.RequireRateLimiting("policy")`, `[DisableRateLimiting]`.
- Algorithms: **Fixed window**, **Sliding window**, **Token bucket**, **Concurrency**. Partition by IP/user/API key via `PartitionedRateLimiter` / `RateLimitPartition.GetFixedWindowLimiter(key, ...)`.
- `RejectionStatusCode` default 503 — set 429. `OnRejected` callback can set `Retry-After`.
- Built-in limiters are **in-process, per instance**; for multi-instance you need a distributed limiter (Redis, API gateway, Azure Front Door/APIM).
- Brute-force defence layers: Identity lockout, per-IP and per-account throttling, CAPTCHA, MFA, uniform error messages (no user enumeration), delay/backoff, breached-password checks, monitoring.
- Client IP behind proxy requires `UseForwardedHeaders` (+ `KnownProxies`), else everyone shares the proxy's IP.

### 13.1 [Easy] How do you add rate limiting in ASP.NET Core 8?
**Answer**
Register `AddRateLimiter` with a named policy (e.g. fixed window), call `UseRateLimiter()`, and apply the policy to endpoints. Rejected requests get the configured status code, preferably 429.

**Explanation**
Each limiter defines permit limit, window, queue limit. Queueing delays requests rather than rejecting.

**Code**
```csharp
using System.Threading.RateLimiting;

builder.Services.AddRateLimiter(o =>
{
    o.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
    o.AddFixedWindowLimiter("api", l => { l.PermitLimit = 100; l.Window = TimeSpan.FromMinutes(1); l.QueueLimit = 0; });
});
var app = builder.Build();
app.UseRateLimiter();
app.MapGet("/items", () => "ok").RequireRateLimiting("api");
```

### 13.2 [Follow-up] Compare fixed window, sliding window, token bucket and concurrency limiters.
**Answer**
Fixed window counts per discrete window (simple; allows 2× bursts at the boundary). Sliding window divides the window into segments for smoother enforcement. Token bucket refills tokens at a steady rate allowing controlled bursts. Concurrency limits simultaneous in-flight requests rather than rate.

**Explanation**
Pick token bucket for API clients with burst tolerance; concurrency for expensive endpoints (reports, uploads).

**Code**
```csharp
builder.Services.AddRateLimiter(o =>
{
    o.AddTokenBucketLimiter("bucket", l =>
    {
        l.TokenLimit = 20; l.TokensPerPeriod = 5; l.ReplenishmentPeriod = TimeSpan.FromSeconds(1);
        l.QueueLimit = 0; l.AutoReplenishment = true;
    });
    o.AddConcurrencyLimiter("heavy", l => { l.PermitLimit = 2; l.QueueLimit = 5; });
});
```

### 13.3 [Medium] Partition limits per user/IP/API key.
**Answer**
Use `GlobalLimiter` or a policy with `RateLimitPartition.Get*Limiter(partitionKey, factory)`. Choose the key from the authenticated user id, else the client IP. Each partition gets its own independent limiter.

**Explanation**
Behind a proxy, `RemoteIpAddress` must come from forwarded headers. Don't trust arbitrary `X-Forwarded-For` from the internet.

**Code**
```csharp
builder.Services.AddRateLimiter(o =>
{
    o.RejectionStatusCode = 429;
    o.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(ctx =>
    {
        var key = ctx.User.Identity?.IsAuthenticated == true
            ? ctx.User.Identity.Name!
            : ctx.Connection.RemoteIpAddress?.ToString() ?? "unknown";
        return RateLimitPartition.GetSlidingWindowLimiter(key, _ => new SlidingWindowRateLimiterOptions
        {
            PermitLimit = 60, Window = TimeSpan.FromMinutes(1), SegmentsPerWindow = 6
        });
    });
    o.OnRejected = (ctx, ct) => { ctx.HttpContext.Response.Headers.RetryAfter = "60"; return ValueTask.CompletedTask; };
});
```

### 13.4 [Hard] The built-in limiter isn't enough for 5 pods. What are the issues and options?
**Answer**
Counters are in-memory per pod, so the effective limit is N× the configured value and resets on restart. Options: enforce at the edge (API Management, Front Door/WAF, NGINX), use a Redis-based distributed limiter (Lua script/token bucket) or a custom `PartitionedRateLimiter` backed by Redis.

**Explanation**
Distributed limiting adds latency and a failure mode: decide fail-open vs fail-closed. Sticky sessions are not a reliable fix.

**Gotchas / follow-ups**
- Attackers rotate IPs; combine with per-account throttling.
- Login throttling by IP only enables a lockout-DoS if you lock by username alone — mix both signals.

**Code**
```csharp
// Edge enforcement is preferred; in-app, a per-pod limit can be divided: limit / podCount
int perPod = 100 / Math.Max(1, int.Parse(Environment.GetEnvironmentVariable("POD_COUNT") ?? "1"));
Console.WriteLine(perPod);
```

### 13.5 [Tricky] How do you defend a login endpoint from credential stuffing without enabling user enumeration or lockout DoS?
**Answer**
Return identical messages/timing for unknown user vs wrong password, rate limit by IP and by account with exponential backoff, add MFA/CAPTCHA after suspicious attempts, check passwords against breach lists, and alert on anomalies. Permanent lockout by username lets attackers lock out victims, so use temporary lockout.

**Explanation**
Identity's `PasswordSignInAsync(lockoutOnFailure: true)` gives temporary lockout (default 5 min after 5 failures). Registration/forgot-password endpoints also leak existence — respond generically ("if the email exists, we sent a link").

**Gotchas / follow-ups**
- Timing: hash a dummy password for unknown users to equalize response time.
- Log failures with IP/user for detection (without logging the password).

**Code**
```csharp
var user = await userManager.FindByEmailAsync(email);
if (user is null)
{
    new PasswordHasher<IdentityUser>().HashPassword(new IdentityUser(), password); // equalize timing
    return Results.Problem("Invalid credentials", statusCode: 401);
}
var r = await signInManager.CheckPasswordSignInAsync(user, password, lockoutOnFailure: true);
return r.Succeeded ? Results.Ok() : Results.Problem("Invalid credentials", statusCode: 401);
```

**Interview traps**
- Default rejection status is 503, not 429.
- In-memory limiter is per instance.
- Behind a proxy all clients look like one IP.
- Distinct "user not found" message enables enumeration.
- Permanent lockout = DoS vector.

**Cheat sheet**
- Four limiters: fixed, sliding, token bucket, concurrency.
- `RequireRateLimiting` / `[EnableRateLimiting]`.
- Partition by user/IP/key.
- Set 429 + Retry-After.
- Distributed limiting at the edge or Redis.

---

## Topic 14: SSO, Entra ID (Azure AD), B2C and Multi-Tenant Authorization

**Study notes**
- **SSO**: log in once at an IdP, access many apps via protocols OIDC / SAML 2.0 / WS-Fed. The IdP session cookie enables silent sign-in to other apps.
- **Microsoft Entra ID** (formerly Azure AD): workforce identity. App registration gives client ID, redirect URIs, API scopes (delegated `scp`) and **app roles** (`roles` claim, for both users and apps). Endpoints: `https://login.microsoftonline.com/{tenant}/v2.0`.
- **Microsoft.Identity.Web** library: `AddMicrosoftIdentityWebApp` (web app sign-in) and `AddMicrosoftIdentityWebApi` (protect API), `RequiredScope`/`RequiredScopeOrAppPermission`, token acquisition, downstream API calls (OBO flow).
- **Tenancy types**: single-tenant (own directory), multi-tenant (`organizations`/`common` authority; `iss` varies by tenant `tid`, so validate issuer against allowed tenants).
- **Azure AD B2C / Entra External ID**: customer identity (social logins, custom user flows/policies). B2C authority includes policy name; Microsoft's newer offering for customers is Entra External ID.
- **Multi-tenant authorization (SaaS data isolation)**: tenant id from a validated claim (`tid` or custom `tenant_id`), never from request body/header unchecked; enforce with EF Core **global query filters**, and optionally SQL **Row-Level Security** (`SESSION_CONTEXT`).
- **Groups overage**: if the user is in >200 groups (JWT), the token omits groups and gives a `_claim_names` pointer → call Microsoft Graph.

### 14.1 [Easy] What is SSO and how does OIDC with Entra ID provide it?
**Answer**
SSO lets a user authenticate once with an identity provider and access multiple apps without re-entering credentials. Each app redirects to Entra ID via OIDC; if the IdP session cookie exists, it returns tokens immediately.

**Explanation**
Apps trust the IdP's signed tokens (issuer/keys from discovery). Logout needs front/back-channel logout to end sessions across apps.

**Code**
```csharp
using Microsoft.Identity.Web;

builder.Services.AddMicrosoftIdentityWebApp(builder.Configuration.GetSection("AzureAd"));
// appsettings: "AzureAd": { "Instance": "https://login.microsoftonline.com/", "TenantId": "...", "ClientId": "...", "CallbackPath": "/signin-oidc" }
```

### 14.2 [Follow-up] Protect a Web API with Entra ID, enforcing scopes and app roles.
**Answer**
Use `AddMicrosoftIdentityWebApi` and require scopes with `[RequiredScope("access_as_user")]` for delegated calls, or require app roles (`roles` claim) for daemon/app permissions. Validate `aud` is your API's app ID URI/client ID.

**Explanation**
Delegated permission = user + app; application permission = app alone (client credentials). Check the right claim for each token type, otherwise an app-only token may pass user checks or vice versa.

**Code**
```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApi(builder.Configuration.GetSection("AzureAd"));

[Authorize]
[RequiredScope("access_as_user")]
[ApiController, Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    [HttpGet] public IActionResult Get() => Ok();
}
```

### 14.3 [Medium] Entra ID vs B2C — when and why?
**Answer**
Entra ID (workforce) is for employees/partners in an organization's directory, with conditional access and app roles. B2C (or Entra External ID) is for customer-facing apps with self-sign-up, social providers and customizable user flows. B2C is configured with policy (user flow) names in the authority and supports different claim sets.

**Explanation**
Microsoft Identity Web supports B2C via `AddMicrosoftIdentityWebApp` with an `AzureAdB2C` section (`Domain`, `SignUpSignInPolicyId`). In B2C, the user object id claim is `oid`/`sub`; B2C custom policies are XML (Identity Experience Framework).

**Code**
```csharp
builder.Services.AddAuthentication(OpenIdConnectDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApp(builder.Configuration.GetSection("AzureAdB2C"));
// "AzureAdB2C": { "Instance": "https://contoso.b2clogin.com", "Domain": "contoso.onmicrosoft.com",
//   "ClientId": "...", "SignUpSignInPolicyId": "B2C_1_susi" }
```

### 14.4 [Hard] Design tenant isolation in a multi-tenant SaaS API with EF Core and MSSQL.
**Answer**
Resolve the tenant from a validated token claim, expose it via a scoped `ITenantProvider`, add an EF Core global query filter on every tenant-owned entity, and set `TenantId` automatically on save. For defence in depth, add SQL Server Row-Level Security using `SESSION_CONTEXT` set per connection. Test that cross-tenant reads/writes fail.

**Explanation**
Global filters can be bypassed with `IgnoreQueryFilters()` or raw SQL, so RLS covers those gaps. Filter must read the tenant from the DbContext instance (property), not a captured static, so EF evaluates it per context instance.

**Gotchas / follow-ups**
- Caches and background jobs must be tenant-scoped, otherwise data leaks.
- Don't trust a `X-Tenant-Id` header without verifying the user belongs to that tenant.
- `ExecuteUpdate/Delete` also apply filters, but raw SQL doesn't.

**Code**
```csharp
public interface ITenantOwned { Guid TenantId { get; set; } }
public class Invoice : ITenantOwned { public int Id { get; set; } public Guid TenantId { get; set; } public decimal Total { get; set; } }
public interface ITenantProvider { Guid TenantId { get; } }

public class AppDb(DbContextOptions<AppDb> o, ITenantProvider tenant) : DbContext(o)
{
    public DbSet<Invoice> Invoices => Set<Invoice>();
    private Guid CurrentTenant => tenant.TenantId; // evaluated per context instance

    protected override void OnModelCreating(ModelBuilder b)
        => b.Entity<Invoice>().HasQueryFilter(i => i.TenantId == CurrentTenant);

    public override Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        foreach (var e in ChangeTracker.Entries<ITenantOwned>().Where(e => e.State == EntityState.Added))
            e.Entity.TenantId = CurrentTenant;
        return base.SaveChangesAsync(ct);
    }
}
// SQL RLS (T-SQL): CREATE SECURITY POLICY TenantFilter ADD FILTER PREDICATE dbo.fn_tenant(TenantId) ON dbo.Invoices;
```

### 14.5 [Tricky] Multi-tenant Entra app: what happens if you set `ValidateIssuer = false`, and how do you validate correctly?
**Answer**
Any tenant's tokens (or a malicious tenant's) would be accepted, enabling cross-tenant impersonation. For multi-tenant apps the issuer differs per tenant (`https://login.microsoftonline.com/{tid}/v2.0`), so supply a custom `IssuerValidator` that checks the `tid` against an allow-list of onboarded tenants (and the issuer format).

**Explanation**
Also key users by `tid` + `oid` together: `oid` is the user's object id within a directory, and the same person can appear as a guest in many tenants with different object ids. Don't use email/UPN as an identifier (mutable).

**Gotchas / follow-ups**
- `sub` is pairwise per application; `oid` is stable across apps in the tenant.
- Check `idtyp`/`azp` claims if you must distinguish app vs user tokens.

**Code**
```csharp
builder.Services.Configure<JwtBearerOptions>(JwtBearerDefaults.AuthenticationScheme, o =>
{
    var allowed = new HashSet<string>(builder.Configuration.GetSection("AllowedTenants").Get<string[]>() ?? []);
    o.TokenValidationParameters.IssuerValidator = (issuer, token, p) =>
    {
        var tid = ((Microsoft.IdentityModel.JsonWebTokens.JsonWebToken)token).Claims.FirstOrDefault(c => c.Type == "tid")?.Value;
        if (tid != null && allowed.Contains(tid) && issuer == $"https://login.microsoftonline.com/{tid}/v2.0")
            return issuer;
        throw new SecurityTokenInvalidIssuerException("Tenant not allowed");
    };
});
```

**Interview traps**
- Reading tenant from a request header/body.
- Using email as user key.
- Global query filters bypassed by `IgnoreQueryFilters`/raw SQL.
- Mixing delegated scopes (`scp`) and app roles (`roles`).
- Group overage claim at >200 groups.

**Cheat sheet**
- Entra ID = workforce; B2C/External ID = customers.
- Microsoft.Identity.Web does the plumbing.
- Delegated → `scp`; application → `roles`.
- Identity key = `tid` + `oid`.
- Tenant isolation: claim → query filter → RLS.

---

## Topic 15: Audit Logging, PII/GDPR and OWASP Top 10 Mapping

**Study notes**
- **Audit log**: who did what, when, to which resource, from where, outcome. Append-only, tamper-evident, separate from app logs, retained per policy. Capture auth events (login, failure, lockout, role change), data changes (before/after), admin actions.
- Implementation options: EF Core `SaveChanges` interceptor (`SaveChangesInterceptor`), SQL Server **temporal tables**, SQL Server Audit, middleware/filters for API calls.
- **Never log** passwords, tokens, full card numbers, or PII unnecessarily; use structured logging with redaction; avoid log forging by not concatenating raw input (structured templates handle it; also sanitize CR/LF).
- **GDPR** essentials: lawful basis & consent, data minimization, purpose limitation, right of access, rectification, **erasure ("right to be forgotten")**, portability, breach notification (72h), privacy by design. Techniques: encryption at rest (TDE, Always Encrypted), pseudonymization, soft-delete + purge jobs, retention policies, data masking (SQL Dynamic Data Masking).
- **OWASP Top 10 (2021) mapping**: A01 Broken Access Control → policies/resource auth/IDOR checks; A02 Cryptographic Failures → TLS/HSTS, Data Protection, strong hashing; A03 Injection → parameterized SQL/encoding; A04 Insecure Design → threat modeling; A05 Security Misconfiguration → CORS, headers, dev exception page off, defaults; A06 Vulnerable Components → NuGet audit/Dependabot; A07 Identification & Auth Failures → Identity lockout/MFA; A08 Software & Data Integrity Failures → signed packages, safe deserialization (avoid `BinaryFormatter`); A09 Logging & Monitoring Failures → audit logging/alerts; A10 SSRF → validate outbound URLs.

### 15.1 [Easy] What should an audit log contain and what must it never contain?
**Answer**
It should record who (user id), what (action/entity/id), when (UTC), where (IP/client), and the result, ideally with old/new values for changes. It must never contain secrets (passwords, tokens, keys) or unnecessary PII.

**Explanation**
Audit logs must be tamper-resistant: write-only store, restricted access, perhaps hash-chained or sent to a SIEM.

**Code**
```csharp
public class AuditEntry
{
    public long Id { get; set; }
    public string UserId { get; set; } = "";
    public string Action { get; set; } = "";
    public string Entity { get; set; } = "";
    public string EntityId { get; set; } = "";
    public string? Changes { get; set; }
    public string? Ip { get; set; }
    public DateTime TimestampUtc { get; set; } = DateTime.UtcNow;
}
```

### 15.2 [Follow-up] Implement automatic data-change auditing with EF Core.
**Answer**
Use a `SaveChangesInterceptor` that inspects `ChangeTracker.Entries()` before saving and adds `AuditEntry` rows for Added/Modified/Deleted entities within the same transaction. Inject the current user from `IHttpContextAccessor`.

**Explanation**
Doing it in the same `SaveChanges` guarantees atomicity. Generated keys for Added entities aren't available before save; capture them in a second pass if needed.

**Code**
```csharp
public class AuditInterceptor(IHttpContextAccessor http) : SaveChangesInterceptor
{
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData e, InterceptionResult<int> result, CancellationToken ct = default)
    {
        var ctx = e.Context!;
        var user = http.HttpContext?.User.FindFirstValue(ClaimTypes.NameIdentifier) ?? "system";
        var audits = ctx.ChangeTracker.Entries()
            .Where(x => x.Entity is not AuditEntry && x.State is EntityState.Added or EntityState.Modified or EntityState.Deleted)
            .Select(x => new AuditEntry
            {
                UserId = user, Action = x.State.ToString(), Entity = x.Metadata.ClrType.Name,
                EntityId = string.Join(",", x.Properties.Where(p => p.Metadata.IsPrimaryKey()).Select(p => p.CurrentValue)),
            }).ToList();
        ctx.Set<AuditEntry>().AddRange(audits);
        return base.SavingChangesAsync(e, result, ct);
    }
}
// optionsBuilder.AddInterceptors(new AuditInterceptor(accessor));
```

### 15.3 [Medium] Map the OWASP Top 10 to concrete ASP.NET Core controls.
**Answer**
A01 → policies/resource-based authz and fallback policy; A02 → HTTPS/HSTS, Data Protection, PBKDF2/Argon2; A03 → parameterized queries and encoding; A05 → disable developer exception page in prod, tight CORS, security headers; A06 → NuGet audit and Dependabot; A07 → Identity lockout/MFA; A08 → avoid insecure deserialization; A09 → audit logging; A10 → allow-list outbound URLs.

**Explanation**
Broken Access Control is #1: most real bugs are missing ownership checks (IDOR), not auth failures. Interviewers like concrete examples per item.

**Code**
```csharp
if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/error"); // no stack traces to clients
    app.UseHsts();
}
// A06: dotnet list package --vulnerable --include-transitive
// A10: validate outbound URL host against an allow-list before HttpClient call
```

### 15.4 [Hard] Implement GDPR erasure when data is spread across tables and backups. What are the trade-offs?
**Answer**
Delete or anonymize PII across all stores (database, logs, caches, search indexes), keep non-personal/legally required records (invoices) by anonymizing the identifying fields, and handle backups by letting them age out under a documented retention while ensuring restored data re-applies erasure requests. Keep an erasure ledger to replay after restores.

**Explanation**
Pseudonymization (replace identifiers with tokens, keep the mapping separately and delete it on erasure) allows referential integrity and analytics. EF Core: use `ExecuteUpdate` for bulk anonymization; avoid PII in logs so it's not a problem.

**Gotchas / follow-ups**
- Soft delete is not erasure.
- Data subject access requests need export of all PII (portability).
- Encrypt PII columns (Always Encrypted) to limit exposure.

**Code**
```csharp
await db.Customers.Where(c => c.Id == id).ExecuteUpdateAsync(s => s
    .SetProperty(c => c.Name, "REDACTED")
    .SetProperty(c => c.Email, c => "deleted-" + c.Id + "@invalid.local")
    .SetProperty(c => c.Phone, (string?)null)
    .SetProperty(c => c.ErasedUtc, DateTime.UtcNow));
```

### 15.5 [Tricky] How can logging itself become a vulnerability?
**Answer**
Logging secrets/PII leaks them to anyone with log access; unsanitized user input enables log forging (injected newlines) and, in some viewers, XSS. Verbose request logging can capture Authorization headers and bodies. Use structured logging templates, redact sensitive fields, and limit log access/retention.

**Explanation**
Serilog `Destructure` policies or `[LogMasked]`-style attributes help. `HttpLogging` redacts headers by default except an allow-list — configure `RequestHeaders` and `LoggingFields` deliberately.

**Gotchas / follow-ups**
- Logging whole exception objects with connection strings in messages.
- Correlation IDs help forensics without PII.

**Code**
```csharp
builder.Services.AddHttpLogging(o =>
{
    o.LoggingFields = HttpLoggingFields.RequestPropertiesAndHeaders | HttpLoggingFields.ResponsePropertiesAndHeaders;
    // Only headers in these sets are logged unredacted; others show as [Redacted]
    o.RequestHeaders.Add("X-Correlation-Id");
});
logger.LogInformation("User {UserId} updated order {OrderId}", userId, orderId); // structured, not concatenated
```

**Interview traps**
- Soft delete ≠ GDPR erasure.
- Putting PII/tokens in logs.
- Audit log stored in same writable table the app/admin can edit.
- Naming OWASP items without examples.
- Thinking encryption at rest alone meets GDPR.

**Cheat sheet**
- Audit: who/what/when/where/result; append-only.
- SaveChangesInterceptor / temporal tables for data audit.
- OWASP A01 Broken Access Control is #1.
- GDPR: minimize, erase, export, 72h breach notice.
- Never log secrets; structured logs.

---

## Topic 16: Security Interview Scenarios (Stolen Token, IDOR)

**Study notes**
- **IDOR / BOLA** (Broken Object Level Authorization, OWASP API #1): user changes `/orders/123` → `/orders/124` and sees someone else's data. Fix: always authorize against the object (ownership/tenant), scope queries by user, use resource-based auth; unguessable GUIDs are *not* a control.
- **Mass assignment** (BOPLA): see Topic 10.
- **Stolen token** response: revoke refresh family, rotate keys if signing key leaked, shorten lifetime, bind tokens (DPoP/mTLS), detect anomalies (IP/user-agent changes), force re-auth for sensitive actions, notify user.
- **Other classics**: SSRF, open redirect (validate `returnUrl` with `Url.IsLocalUrl`), insecure deserialization, path traversal on file download, excessive data exposure (return DTOs), verbose errors, missing function-level authz (admin endpoints reachable), JWT `alg` confusion.
- Structure for scenario answers: **Detect → Contain → Eradicate → Recover → Prevent**.

### 16.1 [Easy] What is IDOR and how do you fix it in an API?
**Answer**
IDOR is when an endpoint trusts a client-supplied identifier without checking the caller may access that object. Fix by scoping every query by the authenticated user/tenant or performing a resource-based authorization check, returning 404/403 on failure.

**Explanation**
Authentication ≠ authorization: being logged in doesn't entitle you to every record.

**Code**
```csharp
app.MapGet("/orders/{id:int}", async (int id, ClaimsPrincipal user, AppDb db) =>
{
    var uid = user.FindFirstValue(ClaimTypes.NameIdentifier);
    var order = await db.Orders.SingleOrDefaultAsync(o => o.Id == id && o.UserId == uid); // ownership in the query
    return order is null ? Results.NotFound() : Results.Ok(order);
}).RequireAuthorization();
```

### 16.2 [Follow-up] Are GUID identifiers a fix for IDOR?
**Answer**
No. Unguessable IDs reduce enumeration but leak through logs, URLs, referrers, shared links, or other endpoints, and are security by obscurity. You still need an explicit authorization check on every access.

**Explanation**
GUIDs are fine as identifiers (and avoid sequential scraping), just not as an access control mechanism.

**Code**
```csharp
var result = await authz.AuthorizeAsync(user, order, "OrderOwnerOrAdmin");
if (!result.Succeeded) return Results.NotFound(); // 404 avoids confirming existence
```

### 16.3 [Medium] A user's JWT was stolen. Walk through your response.
**Answer**
Contain: revoke the user's refresh-token family and bump their security stamp/token version so the access token fails the deny-list check; shorten access-token lifetime going forward. Investigate: find how it leaked (XSS, logs, malware) via audit logs. Recover: force password reset/MFA re-enrolment if needed, notify. Prevent: store tokens in HttpOnly cookies/BFF, CSP, sender-constrained tokens, anomaly detection.

**Explanation**
If the *signing key* leaked, rotate keys immediately (publish new `kid` in JWKS, remove the old) — all tokens signed with the old key are then invalid, which logs everyone out.

**Code**
```csharp
await userManager.UpdateSecurityStampAsync(user);
await db.RefreshTokens.Where(t => t.UserId == user.Id && t.RevokedUtc == null)
    .ExecuteUpdateAsync(s => s.SetProperty(t => t.RevokedUtc, DateTime.UtcNow));
```

### 16.4 [Hard] Open redirect after login: how does it become a security issue and how do you prevent it?
**Answer**
An unvalidated `returnUrl` lets attackers craft a login link that redirects to a phishing site after authentication. Validate it is local with `Url.IsLocalUrl(returnUrl)` (or `LocalRedirect`), or allow-list hosts; for OAuth, require exact `redirect_uri` matches.

**Explanation**
Tricky payloads: `//evil.com`, `/\evil.com`, `https:evil.com`. `IsLocalUrl` handles these; hand-rolled `StartsWith("/")` checks do not (`//evil.com` starts with `/`).

**Gotchas / follow-ups**
- `LocalRedirect` throws for non-local URLs.
- Open redirects chain into OAuth token theft when `redirect_uri` validation is loose.

**Code**
```csharp
public IActionResult Login(string? returnUrl)
{
    if (!string.IsNullOrEmpty(returnUrl) && Url.IsLocalUrl(returnUrl))
        return LocalRedirect(returnUrl);
    return RedirectToAction("Index", "Home");
}
```

### 16.5 [Tricky] An endpoint `/admin/users` is hidden in the UI but not protected server-side. What is this class of bug and how do you systematically prevent it?
**Answer**
It's Broken Function Level Authorization (OWASP API5, under A01): relying on UI hiding. Prevent by default-deny: fallback policy requiring authentication, admin routes in a group with `RequireAuthorization("AdminOnly")`, and automated tests that enumerate all endpoints and assert each has authorization metadata unless explicitly anonymous.

**Explanation**
Security must be enforced server-side on every request. A test over `EndpointDataSource` catches newly added unprotected endpoints.

**Gotchas / follow-ups**
- Also check HTTP-method variance (GET protected, PUT not).
- Versioned/legacy routes (`/v1/...`) forgotten.

**Code**
```csharp
app.MapGroup("/admin").RequireAuthorization("AdminOnly").MapGet("/users", () => "users");

// Test-ish audit: list endpoints without authorization metadata
var open = app.Services.GetRequiredService<EndpointDataSource>().Endpoints
    .OfType<RouteEndpoint>()
    .Where(e => e.Metadata.GetMetadata<IAuthorizeData>() is null && e.Metadata.GetMetadata<IAllowAnonymous>() is null)
    .Select(e => e.RoutePattern.RawText);
foreach (var r in open) Console.WriteLine($"UNPROTECTED: {r}");
```

**Interview traps**
- "UUIDs prevent IDOR."
- Hiding buttons ≠ authorization.
- `StartsWith("/")` as open redirect check.
- Forgetting to rotate signing keys when they leak.
- Responding to an incident without containment-first thinking.

**Cheat sheet**
- IDOR/BOLA: scope by owner/tenant on every query.
- 404 vs 403 to avoid existence leak.
- Stolen token: revoke refresh, bump stamp, rotate keys if needed.
- `Url.IsLocalUrl` for redirects.
- Default-deny + endpoint audit tests.

