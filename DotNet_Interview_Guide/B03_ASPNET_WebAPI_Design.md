# B03 — ASP.NET Core Web API Design

**How to use:** Read each topic's *Study notes* first, then try to answer every ladder question aloud before reading the **Answer**; rungs go Easy → Tricky.
Code targets C# 12 / .NET 8 (`Nullable` and `ImplicitUsings` enabled); snippets are fragments unless they show a full `Program.cs`.

## Table of Contents
1. [REST principles & HTTP methods/status codes](#topic-1-rest-principles--http-methodsstatus-codes)
2. [Controllers & [ApiController] behaviors](#topic-2-controllers--apicontroller-behaviors)
3. [Model binding sources & custom binders](#topic-3-model-binding-sources--custom-binders)
4. [Validation](#topic-4-validation)
5. [Action results: IActionResult vs ActionResult<T>](#topic-5-action-results-iactionresult-vs-actionresultt)
6. [Filters & pipeline order](#topic-6-filters--pipeline-order)
7. [Global exception handling & ProblemDetails](#topic-7-global-exception-handling--problemdetails)
8. [API versioning](#topic-8-api-versioning)
9. [Pagination, filtering, sorting](#topic-9-pagination-filtering-sorting)
10. [DTOs & mapping](#topic-10-dtos--mapping)
11. [Minimal APIs deep dive](#topic-11-minimal-apis-deep-dive)
12. [Content negotiation, formatters, System.Text.Json](#topic-12-content-negotiation-formatters-systemtextjson)
13. [File upload/download, streaming, large payloads, compression](#topic-13-file-uploaddownload-streaming-large-payloads-compression)
14. [Idempotency keys, retries, ETags & optimistic concurrency](#topic-14-idempotency-keys-retries-etags--optimistic-concurrency)
15. [Rate limiting middleware](#topic-15-rate-limiting-middleware)
16. [OpenAPI / Swagger](#topic-16-openapi--swagger)
17. [Long-running operations, bulk operations](#topic-17-long-running-operations-bulk-operations)
18. [API styles & architecture: HATEOAS, GraphQL/gRPC, Gateway/BFF, CQRS with MediatR](#topic-18-api-styles--architecture-hateoas-graphqlgrpc-gatewaybff-cqrs-with-mediatr)

---

## Topic 1: REST principles & HTTP methods/status codes

**Study notes**

REST constraints (Fielding): client-server, stateless, cacheable, uniform interface (resource identification, manipulation via representations, self-descriptive messages, HATEOAS), layered system, optional code-on-demand.

| Method | Safe | Idempotent | Typical use | Success code |
|---|---|---|---|---|
| GET | yes | yes | read | 200 |
| HEAD | yes | yes | headers only | 200 |
| OPTIONS | yes | yes | CORS preflight / capabilities | 200/204 |
| POST | no | no | create / non-idempotent action | 201 (+Location) / 200 / 202 |
| PUT | no | yes | full replace (or create at known URI) | 200 / 204 / 201 |
| PATCH | no | not guaranteed | partial update | 200 / 204 |
| DELETE | no | yes | remove | 204 (or 200/202) |

*Safe* = no observable state change requested. *Idempotent* = N identical requests have the same server-state effect as one (the response may differ, e.g. DELETE: 204 then 404).

Status classes: 2xx success, 3xx redirect/cache (304), 4xx client error, 5xx server error. Common: 400 malformed/invalid, 401 unauthenticated, 403 forbidden, 404 not found, 405, 406 (no acceptable representation), 409 conflict, 412 precondition failed, 415 unsupported media type, 422 semantically invalid, 428 precondition required, 429 too many requests.

### 1.1 [Easy] What does it mean for an HTTP method to be safe and idempotent?
**Answer**
Safe methods (GET, HEAD, OPTIONS) don't change server state as requested by the client. Idempotent methods (GET, PUT, DELETE, plus the safe ones) produce the same resulting server state whether called once or many times. POST is neither, and PATCH is not guaranteed idempotent.

**Explanation**
Idempotency is about *state effect*, not about identical responses. It matters because clients, proxies and HTTP libraries (e.g. Polly retries) can safely auto-retry idempotent requests after a timeout. Safe implies idempotent, but not vice versa (DELETE is idempotent, not safe). Note that "safe" is a contract: a GET that increments a counter or deletes data violates the contract and breaks caches/crawlers/prefetchers.

**Code**
```csharp
app.MapGet("/orders/{id:int}", (int id) => Results.Ok(new { id }));          // safe + idempotent
app.MapPut("/orders/{id:int}", (int id, OrderDto dto) => Results.NoContent()); // idempotent
app.MapPost("/orders", (OrderDto dto) => Results.Created($"/orders/42", dto)); // neither
record OrderDto(string Name);
```

### 1.2 [Follow-up] PUT vs PATCH vs POST — when do you use each?
**Answer**
POST creates a subordinate resource (server picks the URI) or triggers a non-idempotent action. PUT replaces the whole resource at a client-known URI and is idempotent. PATCH applies a partial change; its idempotency depends on the patch format (e.g. "set name=X" is idempotent, "increment by 1" is not).

**Explanation**
With PUT, omitted fields are conceptually reset/removed, so the client sends the full representation. PATCH formats: JSON Merge Patch (RFC 7396, `application/merge-patch+json`, cannot set a field to null distinctly from "remove" and replaces arrays whole) and JSON Patch (RFC 6902, `application/json-patch+json`, ordered ops like `replace`, `add`, `test`). ASP.NET Core supports JSON Patch via `Microsoft.AspNetCore.JsonPatch` + `Microsoft.AspNetCore.Mvc.NewtonsoftJson` (`JsonPatchDocument<T>`); there is no first-class System.Text.Json support in MVC, so many teams use a PATCH DTO with `Optional<T>`-style wrappers or merge-patch.

**Code**
```csharp
using Microsoft.AspNetCore.JsonPatch;
using Microsoft.AspNetCore.Mvc;

[ApiController, Route("api/products")]
public class ProductsController : ControllerBase
{
    private static readonly Product Store = new() { Id = 1, Name = "A", Price = 10 };

    [HttpPut("{id:int}")] // full replace
    public IActionResult Put(int id, ProductPutDto dto)
    {
        Store.Name = dto.Name; Store.Price = dto.Price;
        return NoContent();
    }

    [HttpPatch("{id:int}")] // needs NewtonsoftJson input formatter for JsonPatchDocument
    public IActionResult Patch(int id, [FromBody] JsonPatchDocument<Product> patch)
    {
        patch.ApplyTo(Store, ModelState);
        return ModelState.IsValid ? NoContent() : ValidationProblem(ModelState);
    }
}
public class Product { public int Id { get; set; } public string Name { get; set; } = ""; public decimal Price { get; set; } }
public record ProductPutDto(string Name, decimal Price);
```

### 1.3 [Medium] How do you choose between 400, 404, 409, 422, 401 and 403?
**Answer**
400 = request can't be parsed/bound or is structurally invalid; 422 = well-formed but semantically invalid (business validation). 401 = no/invalid credentials, 403 = authenticated but not allowed, 404 = resource missing (also used to hide existence), 409 = conflict with current state (duplicate, version clash).

**Explanation**
ASP.NET Core's `[ApiController]` returns 400 `ValidationProblemDetails` by default for model errors; many teams keep 400 for consistency and use 422 only if the contract demands it (you can change via `InvalidModelStateResponseFactory`). Use 409 for unique-constraint violations and concurrency conflicts without preconditions; use 412 when an `If-Match` precondition fails. Returning 200 with an error body is an anti-pattern: it breaks monitoring and client retry logic. For 401 the response should include a `WWW-Authenticate` header (JwtBearer does this).

**Code**
```csharp
app.MapPost("/users", (CreateUser req, IUserRepo repo) =>
{
    if (string.IsNullOrWhiteSpace(req.Email)) return Results.ValidationProblem(
        new Dictionary<string, string[]> { ["email"] = ["required"] });      // 400 w/ problem details
    if (repo.Exists(req.Email)) return Results.Conflict(new { error = "email exists" }); // 409
    return Results.Created("/users/1", req);
});
record CreateUser(string Email);
interface IUserRepo { bool Exists(string e); }
```

### 1.4 [Hard] Is DELETE really idempotent if the second call returns 404? Is PUT-create idempotent? What about POST with retry?
**Answer**
Yes: idempotency refers to server state, not the response. After the first DELETE the resource is gone, and the second leaves the same state (gone) even though it returns 404. POST is not idempotent, so a retried POST may create duplicates unless you add an idempotency key.

**Explanation**
PUT to a client-chosen URI can create (201) the first time and replace (200/204) afterward; state after N calls is the same, so it's idempotent. A subtle point: a "PUT that appends to a list" or "PUT that bumps a lastModified timestamp" is arguably still idempotent by the spec's definition (the intended effect is the same) but audit logs show differences. Some APIs make DELETE return 204 always to simplify client retry logic. Also, idempotency doesn't imply concurrency safety: two clients' PUTs can still lost-update — add ETags (Topic 14).

**Code**
```csharp
var items = new Dictionary<int, string>();
app.MapPut("/items/{id:int}", (int id, string name) =>
{
    var existed = items.ContainsKey(id);
    items[id] = name;
    return existed ? Results.NoContent() : Results.Created($"/items/{id}", name);
});
app.MapDelete("/items/{id:int}", (int id) => { items.Remove(id); return Results.NoContent(); }); // always 204
```

### 1.5 [Tricky] A client sends GET with a body, or uses POST for a search. Is that wrong? And what's the 201 contract?
**Answer**
GET with a body has undefined semantics (RFC 9110) and many proxies/clients drop it, so avoid it; use POST (e.g. `POST /orders/search`) for complex queries, or the newer `QUERY` method if supported. A 201 response should include a `Location` header to the new resource and typically the representation.

**Explanation**
Using POST for search is pragmatic but loses cacheability and the safe/idempotent guarantee, so document it as safe-in-practice. ASP.NET Core's `CreatedAtAction`/`CreatedAtRoute` build the `Location` header from route data; in minimal APIs use `Results.Created(uri, body)` or `TypedResults.CreatedAtRoute`. Also: HEAD must return the same headers as GET without a body; `[HttpGet]`/`MapGet` do not match HEAD requests, so declare `[HttpHead]` (or `MapMethods`) explicitly if you need it.

**Code**
```csharp
[ApiController, Route("api/orders")]
public class OrdersController : ControllerBase
{
    [HttpGet("{id:int}", Name = "GetOrder")]
    public ActionResult<OrderView> Get(int id) => new OrderView(id);

    [HttpPost]
    public ActionResult<OrderView> Create(NewOrder o)
    {
        var view = new OrderView(7);
        return CreatedAtRoute("GetOrder", new { id = view.Id }, view); // Location: /api/orders/7
    }

    [HttpPost("search")] // complex query via POST
    public ActionResult<OrderView[]> Search(SearchCriteria c) => Array.Empty<OrderView>();
}
public record OrderView(int Id);
public record NewOrder(string Item);
public record SearchCriteria(string? Text);
```

**Gotchas / follow-ups**
- Stateless ≠ no sessions only; it means each request carries all context (token), enabling horizontal scaling.
- Don't use verbs in URIs (`/getOrders`); actions that aren't CRUD may use sub-resources (`POST /orders/{id}/cancel`).
- Retries: only auto-retry idempotent methods unless an idempotency key exists (Topic 14).

**Interview traps**
- "PUT is for update, POST is for create" — oversimplified; the real distinction is URI ownership and idempotency.
- Saying idempotent means "same response" — it means same server state.
- Returning 200 + `{success:false}` for errors.
- Confusing 401 and 403.

**Cheat sheet**
- Safe: GET/HEAD/OPTIONS. Idempotent: those + PUT/DELETE.
- POST creates with 201 + `Location`; PUT replaces; PATCH partial.
- 400 invalid shape, 422 invalid semantics, 409 conflict, 412 failed precondition.
- 401 = who are you, 403 = not allowed.

---

## Topic 2: Controllers & [ApiController] behaviors

**Study notes**

Controllers derive from `ControllerBase` (API, no view support) rather than `Controller` (adds views). `[ApiController]` enables opinionated behaviors:

1. **Attribute routing required** (no conventional routes).
2. **Automatic HTTP 400** when `ModelState` is invalid (an implicit `ModelStateInvalidFilter`).
3. **Binding source inference**: complex types → `[FromBody]`; `IFormFile` → `[FromForm]`; route-matching parameter names → `[FromRoute]`; everything else → `[FromQuery]`.
4. **Multipart/form-data inference** for form types.
5. **ProblemDetails for error status codes** (4xx results get an RFC 7807 body via `ClientErrorMapping`).

Pipeline: Routing → endpoint selected → Middleware → filters → model binding → validation → action → result execution. Controllers are instantiated per request (transient by default, activated via DI), so don't store state in fields.

```
Request -> Kestrel -> Middleware... -> Routing(EndpointMiddleware)
        -> [Auth/Resource filters] -> Model binding + validation
        -> [Action filters] -> Action -> [Result filters] -> Formatter -> Response
```

### 2.1 [Easy] What does [ApiController] do?
**Answer**
It opts a controller into API-specific conventions: required attribute routing, automatic 400 responses for invalid models, binding-source inference, and ProblemDetails for error responses. It reduces boilerplate like `if (!ModelState.IsValid) return BadRequest(ModelState);`.

**Explanation**
It can be applied per controller, per assembly (`[assembly: ApiController]`), or via a base class. Because the 400 behavior is implemented as a filter, your action is not even invoked when binding/validation fails.

**Code**
```csharp
[ApiController]
[Route("api/[controller]")]
public class CustomersController : ControllerBase
{
    [HttpPost]
    public IActionResult Create(CreateCustomer dto) => Ok(dto); // invalid dto => automatic 400
}
public record CreateCustomer([property: System.ComponentModel.DataAnnotations.Required] string Name);
```

### 2.2 [Follow-up] How do you customize or disable the automatic 400 behavior?
**Answer**
Configure `ApiBehaviorOptions`: set `SuppressModelStateInvalidFilter = true` to handle ModelState yourself, or supply `InvalidModelStateResponseFactory` to shape the response. Other flags: `SuppressInferBindingSourcesForParameters`, `SuppressMapClientErrors`, `SuppressConsumesConstraintForFormFileParameters`.

**Explanation**
Typical reasons: the contract demands 422, a custom envelope, or you want to log validation failures. The factory receives an `ActionContext`, giving access to ModelState and HttpContext.

**Code**
```csharp
builder.Services.AddControllers().ConfigureApiBehaviorOptions(o =>
{
    o.InvalidModelStateResponseFactory = ctx =>
    {
        var pd = new Microsoft.AspNetCore.Mvc.ValidationProblemDetails(ctx.ModelState)
        {
            Status = StatusCodes.Status422UnprocessableEntity,
            Title = "Validation failed",
            Instance = ctx.HttpContext.Request.Path
        };
        return new ObjectResult(pd) { StatusCode = pd.Status };
    };
});
```

### 2.3 [Medium] How does binding source inference work, and where does it bite you?
**Answer**
Complex-type parameters are inferred `[FromBody]`, simple types come from route (if the name matches a route token) otherwise query, and `IFormFile`/`IFormFileCollection` from form. Only **one** parameter can be inferred from the body; two complex params cause an `InvalidOperationException` at startup/first request.

**Explanation**
Gotcha: a complex type you intended to bind from the query (e.g. a `PagingParams` object) needs explicit `[FromQuery]`. Also, a `CancellationToken` and services: in .NET 7+, `[FromServices]` is inferred for registered service types (`DisableImplicitFromServicesParameters` to turn off). In .NET 7+ with `[ApiController]`, a non-nullable reference-type parameter missing from the body is a 400 because of implicit `[Required]` (when `Nullable` is enabled).

**Code**
```csharp
[ApiController, Route("api/search")]
public class SearchController : ControllerBase
{
    // Without [FromQuery], PagingParams would be inferred as [FromBody] -> 415 for GET
    [HttpGet]
    public IActionResult Get([FromQuery] PagingParams p, CancellationToken ct) => Ok(p);
}
public record PagingParams(int Page = 1, int PageSize = 20);
```

### 2.4 [Hard] Controller lifetime, DI, and the base-class gotchas
**Answer**
A controller is created per request through `IControllerActivator`; constructor injection resolves scoped services from the request scope. `Controller` vs `ControllerBase`: use `ControllerBase` for APIs since `Controller` brings view-related members (`View()`, `ViewData`).

**Explanation**
Controllers are *not* registered as services by default (`AddControllersAsServices` changes that to allow DI container construction and decoration). Never inject a scoped service into a singleton, and don't capture `HttpContext` into background work: it's recycled after the request. Use `IServiceScopeFactory` for fire-and-forget work. Keep controllers thin: orchestrate, don't hold business logic or `DbContext` queries.

**Code**
```csharp
[ApiController, Route("api/[controller]")]
public class ReportsController(IReportService svc, ILogger<ReportsController> log) : ControllerBase
{
    [HttpGet("{id:guid}")]
    public async Task<ActionResult<ReportDto>> Get(Guid id, CancellationToken ct)
    {
        log.LogInformation("Fetching {Id}", id);
        var r = await svc.GetAsync(id, ct);
        return r is null ? NotFound() : r;
    }
}
public interface IReportService { Task<ReportDto?> GetAsync(Guid id, CancellationToken ct); }
public record ReportDto(Guid Id);
```

**Gotchas / follow-ups**
- Primary constructors (C# 12) work for controllers; parameters are captured, not readonly.
- Route constraint `{id:guid}` mismatch gives 404, not 400.
- `[Route("api/[controller]")]` token drops the `Controller` suffix.

**Interview traps**
- Believing `[ApiController]` validates automatically *without* model-state filter—it's the filter; `SuppressModelStateInvalidFilter` disables it.
- Two `[FromBody]` params.
- Using `Controller` for APIs and accidentally getting view-related behavior.
- Putting `[FromBody]` on a GET.

**Cheat sheet**
- `[ApiController]` = attribute routing + auto 400 + binding inference + ProblemDetails.
- Customize with `ConfigureApiBehaviorOptions`.
- One body source per action.
- Controllers are per-request; keep them thin.

---

## Topic 3: Model binding sources & custom binders

**Study notes**

Binding sources: `[FromRoute]`, `[FromQuery]`, `[FromHeader]`, `[FromBody]`, `[FromForm]`, `[FromServices]`, `[FromKeyedServices]` (.NET 8), `[AsParameters]` (minimal APIs). Order of value providers (without attributes, non-ApiController): form → route → query. Body binding is done by an input formatter (System.Text.Json by default) and reads the stream once.

Custom extension points:
- `IModelBinder` / `IModelBinderProvider` (full control)
- `TypeConverter` (simple types, also used by minimal APIs? minimal APIs use `TryParse`/`BindAsync` instead)
- `[ModelBinder(typeof(X))]` on a parameter/type
- Minimal APIs: static `TryParse(string, out T)` and `static ValueTask<T?> BindAsync(HttpContext, ParameterInfo)`.

### 3.1 [Easy] What are the main binding sources?
**Answer**
Route values, query string, headers, request body, form data and DI services. Each has an attribute: `[FromRoute]`, `[FromQuery]`, `[FromHeader]`, `[FromBody]`, `[FromForm]`, `[FromServices]`.

**Explanation**
`[FromBody]` uses input formatters and consumes the body, so only one per action. `[FromHeader(Name = "X-Correlation-Id")]` binds a named header. Binding failures are added to `ModelState`.

**Code**
```csharp
[HttpPost("{tenant}/items")]
public IActionResult Create(
    [FromRoute] string tenant,
    [FromQuery] bool dryRun,
    [FromHeader(Name = "X-Correlation-Id")] string? correlationId,
    [FromBody] ItemDto body,
    [FromServices] TimeProvider clock)
    => Ok(new { tenant, dryRun, correlationId, body, at = clock.GetUtcNow() });
public record ItemDto(string Name);
```

### 3.2 [Follow-up] How do you bind a complex object or collection from the query string?
**Answer**
Mark it `[FromQuery]`; properties bind by name, and collections bind from repeated keys (`?ids=1&ids=2`) or indexed forms. For dictionaries use `?filter[a]=1` style keys.

**Explanation**
Complex-type query binding in MVC is property-name based and case-insensitive. Minimal APIs (before .NET 8 / in general) don't bind complex query objects by default; use `[AsParameters]` to bind a record/class whose members come from various sources. Arrays in minimal APIs bind from `?ids=1&ids=2` automatically for `int[]`/`List<int>`.

**Code**
```csharp
app.MapGet("/products", ([AsParameters] ProductQuery q) => Results.Ok(q));
record ProductQuery(string? Name, int[]? Ids, int Page = 1);
// GET /products?name=x&ids=1&ids=2  -> Ids = [1,2], Page = 1
```

### 3.3 [Medium] Write a custom model binder for a comma-separated list or a custom value type.
**Answer**
Implement `IModelBinder.BindModelAsync`, read from `bindingContext.ValueProvider`, and set `bindingContext.Result`. Register it via `[ModelBinder(typeof(...))]` or an `IModelBinderProvider` inserted into `MvcOptions.ModelBinderProviders`.

**Explanation**
Always call `bindingContext.ModelState.TryAddModelError` for malformed input and `ModelBindingResult.Failed()`; call `ModelBindingResult.Success(value)` otherwise. Providers are evaluated in order, so insert yours at index 0 for precedence.

**Code**
```csharp
using Microsoft.AspNetCore.Mvc.ModelBinding;

public class CsvIntBinder : IModelBinder
{
    public Task BindModelAsync(ModelBindingContext ctx)
    {
        var vp = ctx.ValueProvider.GetValue(ctx.ModelName);
        if (vp == ValueProviderResult.None) return Task.CompletedTask;
        ctx.ModelState.SetModelValue(ctx.ModelName, vp);
        var list = new List<int>();
        foreach (var s in (vp.FirstValue ?? "").Split(',', StringSplitOptions.RemoveEmptyEntries))
        {
            if (!int.TryParse(s, out var n))
            {
                ctx.ModelState.TryAddModelError(ctx.ModelName, $"'{s}' is not an int");
                return Task.CompletedTask; // Result stays Failed
            }
            list.Add(n);
        }
        ctx.Result = ModelBindingResult.Success(list);
        return Task.CompletedTask;
    }
}

[HttpGet("by-ids")]
public IActionResult ByIds([ModelBinder(typeof(CsvIntBinder))] List<int> ids) => Ok(ids);
// GET /by-ids?ids=1,2,3 -> [1,2,3]
```

### 3.4 [Hard] How do minimal APIs customize binding, and what happens to the request body when read twice?
**Answer**
Minimal API parameter types may expose `static bool TryParse(string, out T)` for string sources or `static ValueTask<T?> BindAsync(HttpContext, ParameterInfo)` for full control. The request body is a forward-only stream, so it can only be read once unless you call `Request.EnableBuffering()` and rewind.

**Explanation**
In MVC, if a filter or middleware (e.g. logging) reads the body first, `[FromBody]` receives an empty body unless buffering is enabled and `Position` reset to 0. `EnableBuffering` spools to memory up to 30 KB then to disk temp files. Also in .NET 8 minimal APIs, `[FromBody]` is inferred only for complex types; for GET/DELETE endpoints body binding is not inferred (explicit `[FromBody]` throws unless allowed).

**Code**
```csharp
public readonly record struct Point(double X, double Y)
{
    public static bool TryParse(string? s, IFormatProvider? _, out Point p)
    {
        p = default;
        var parts = s?.Split(',');
        if (parts is { Length: 2 } && double.TryParse(parts[0], out var x) && double.TryParse(parts[1], out var y))
        { p = new(x, y); return true; }
        return false;
    }
}
// app.MapGet("/distance", (Point from, Point to) => Math.Sqrt(...));  // ?from=1,2&to=3,4

// Re-reading the body in middleware
app.Use(async (ctx, next) =>
{
    ctx.Request.EnableBuffering();
    using var reader = new StreamReader(ctx.Request.Body, leaveOpen: true);
    var raw = await reader.ReadToEndAsync();
    ctx.Request.Body.Position = 0;           // rewind for model binding
    await next();
});
```

**Gotchas / follow-ups**
- Failed `TryParse` in minimal APIs → 400 automatically.
- Over-posting: binding to entity classes lets clients set fields you didn't intend; bind to DTOs.
- `[Bind]` / `[BindNever]` exist but DTOs are better.

### 3.5 [Tricky] Why does an int property missing from JSON become 0 and pass `[Required]`? How do you detect "missing" vs "default"?
**Answer**
Value types have a default (0), so the binder can't distinguish missing from zero and `[Required]` is satisfied. Use nullable (`int?`) with `[Required]`, or C# 11 `required` members, which System.Text.Json (.NET 7+) enforces during deserialization.

**Explanation**
With `required` properties STJ throws `JsonException`, which MVC turns into a 400 model error. For PATCH semantics ("field omitted" vs "set to null") you need a tri-state wrapper or `JsonElement`/`JsonNode` handling, since plain DTOs can't differentiate.

**Code**
```csharp
public class CreateTicket
{
    public required string Title { get; init; }   // missing -> 400
    public int? Priority { get; init; }           // null = not provided
}
```

**Interview traps**
- Two `[FromBody]` parameters.
- Reading body in middleware without `EnableBuffering`.
- Binding directly to EF entities (over-posting).
- Assuming `[Required]` works on non-nullable value types.

**Cheat sheet**
- Sources: route, query, header, body, form, services.
- One body per action; body = input formatter.
- Custom: `IModelBinder` (MVC), `TryParse`/`BindAsync` (minimal).
- `[AsParameters]` groups minimal-API params.
- Use `required` or nullable to detect omission.

---

## Topic 4: Validation

**Study notes**

Layers: (1) binding errors, (2) DataAnnotations (`[Required]`, `[StringLength]`, `[Range]`, `[RegularExpression]`, `[EmailAddress]`), (3) `IValidatableObject.Validate` (cross-property; runs only if property-level validation passed), (4) FluentValidation (separate package, rule-based, DI-friendly, async rules), (5) domain/business validation in services.

MVC runs validation after binding and records errors in `ModelState`; `[ApiController]` auto-returns 400. Minimal APIs have **no built-in validation in .NET 8** (.NET 10 adds built-in `AddValidation`); use FluentValidation, an endpoint filter, or `MiniValidation`/`Validator.TryValidateObject`.

Rule of thumb: syntactic validation at the edge (DTO), business invariants in the domain, uniqueness enforced by DB constraints (validation checks are racy).

### 4.1 [Easy] How does DataAnnotations validation work in a controller?
**Answer**
Attributes on DTO properties declare rules; after model binding, MVC validates and populates `ModelState`. With `[ApiController]` an invalid model short-circuits to a 400 `ValidationProblemDetails`.

**Explanation**
Error keys are property paths (`address.city`). For records, attributes on positional parameters need the `property:` target (`[property: Required]`) — otherwise MVC still honors parameter-targeted validation attributes for records in modern versions, but the explicit target is the safe choice for serializers/OpenAPI.

**Code**
```csharp
using System.ComponentModel.DataAnnotations;
public class RegisterDto
{
    [Required, StringLength(50, MinimumLength = 2)] public string Name { get; set; } = "";
    [Required, EmailAddress] public string Email { get; set; } = "";
    [Range(18, 120)] public int Age { get; set; }
}
// POST {"name":"A","email":"x","age":5} -> 400 with errors for name, email, age
```

### 4.2 [Follow-up] When would you use IValidatableObject or FluentValidation instead?
**Answer**
Use `IValidatableObject` for simple cross-property rules on one class. Use FluentValidation for richer, testable, reusable rules (conditions, async checks, collections, DI-injected dependencies) kept separate from the DTO.

**Explanation**
Gotcha: `IValidatableObject.Validate` is only called when attribute validation passes. FluentValidation's ASP.NET auto-validation integration (`FluentValidation.AspNetCore`) is no longer recommended; call `validator.ValidateAsync` manually or via a filter, then convert to `ValidationProblem`. Async validators can't run in MVC's synchronous auto-validation pipeline — another reason for manual invocation.

**Code**
```csharp
using FluentValidation;
public record CreateBooking(DateOnly Start, DateOnly End, string Email);

public class CreateBookingValidator : AbstractValidator<CreateBooking>
{
    public CreateBookingValidator()
    {
        RuleFor(x => x.Email).NotEmpty().EmailAddress();
        RuleFor(x => x.End).GreaterThan(x => x.Start).WithMessage("End must be after Start");
    }
}

app.MapPost("/bookings", async (CreateBooking req, IValidator<CreateBooking> v) =>
{
    var r = await v.ValidateAsync(req);
    return r.IsValid ? Results.Ok() : Results.ValidationProblem(r.ToDictionary());
});
```

### 4.3 [Medium] How do you build a reusable validation endpoint filter for minimal APIs?
**Answer**
Write an `IEndpointFilter` generic on the DTO type that locates the argument, resolves `IValidator<T>`, and short-circuits with `Results.ValidationProblem` on failure. Attach with `.AddEndpointFilter<ValidationFilter<T>>()`.

**Explanation**
`EndpointFilterInvocationContext.Arguments` holds handler arguments; `GetArgument<T>(i)` or search by type. Doing this at group level (`group.AddEndpointFilter`) applies to all endpoints in the group.

**Code**
```csharp
using FluentValidation;
public class ValidationFilter<T> : IEndpointFilter where T : class
{
    public async ValueTask<object?> InvokeAsync(EndpointFilterInvocationContext ctx, EndpointFilterDelegate next)
    {
        var arg = ctx.Arguments.OfType<T>().FirstOrDefault();
        var validator = ctx.HttpContext.RequestServices.GetService<IValidator<T>>();
        if (arg is not null && validator is not null)
        {
            var result = await validator.ValidateAsync(arg, ctx.HttpContext.RequestAborted);
            if (!result.IsValid) return Results.ValidationProblem(result.ToDictionary());
        }
        return await next(ctx);
    }
}
// app.MapPost("/bookings", (CreateBooking b) => Results.Ok()).AddEndpointFilter<ValidationFilter<CreateBooking>>();
```

### 4.4 [Hard] How do you write a custom validation attribute and why is it still not enough for business rules?
**Answer**
Derive from `ValidationAttribute` and override `IsValid(object?, ValidationContext)`; you can resolve services via `validationContext.GetService`. It still can't guarantee business rules like uniqueness, because the check and the insert aren't atomic.

**Explanation**
Always back "unique email" checks with a unique DB index and translate `DbUpdateException` (SQL error 2601/2627) into 409. Validation attributes also run synchronously, so I/O in them blocks threads. Also remember that model validation depth is limited (`MaxValidationDepth` default 32) and large collections get validated element by element (cost).

**Code**
```csharp
public sealed class FutureDateAttribute : ValidationAttribute
{
    protected override ValidationResult? IsValid(object? value, ValidationContext ctx)
        => value is DateTime d && d > DateTime.UtcNow
            ? ValidationResult.Success
            : new ValidationResult("Date must be in the future", new[] { ctx.MemberName! });
}
```

### 4.5 [Tricky] Why does validation pass for a missing non-nullable string, or fail for one that's absent with Nullable enabled?
**Answer**
In MVC with nullable reference types enabled, non-nullable reference-type properties/parameters are implicitly `[Required]`, so omitted ones fail. In minimal APIs, DataAnnotations aren't evaluated at all in .NET 8, so nothing fails unless you check.

**Explanation**
Disable the implicit behavior via `MvcOptions.SuppressImplicitRequiredAttributeForNonNullableReferenceTypes = true`. Also note empty string `""` satisfies non-nullable but fails `[Required]` (`AllowEmptyStrings=false`) — the implicit required attribute is created with `AllowEmptyStrings`-like semantics only for null checks (implicit one doesn't reject empty strings). Be explicit with `[Required]` plus `[MinLength(1)]` if it matters.

**Code**
```csharp
public class Dto { public string Name { get; set; } = default!; }  // implicit required in MVC
builder.Services.AddControllers(o => o.SuppressImplicitRequiredAttributeForNonNullableReferenceTypes = true);
```

**Interview traps**
- Assuming minimal APIs validate DataAnnotations in .NET 8.
- Relying on validation instead of a unique index.
- `IValidatableObject` not running when property rules fail.
- Doing DB calls in validation attributes.

**Cheat sheet**
- DataAnnotations → ModelState → auto 400 (`[ApiController]`).
- FluentValidation: call manually/filter; async OK.
- Minimal API: endpoint filter for validation.
- DB constraints are the final truth.

---

## Topic 5: Action results: IActionResult vs ActionResult<T>

**Study notes**

| Return type | Pros | Cons |
|---|---|---|
| `T` / `Task<T>` | simplest, always 200 | can't return 404 etc. (null → 204 by default, via `HttpNoContentOutputFormatter`) |
| `IActionResult` | any result | no static type for OpenAPI (needs `[ProducesResponseType]`) |
| `ActionResult<T>` | typed body for OpenAPI + any `ActionResult` + implicit conversion from `T` | can't return `IActionResult` directly (only `ActionResult` types), not covariant |
| `IResult`/`TypedResults` (minimal) | fully typed, `Results<A,B>` union | minimal APIs only (also usable in MVC actions in .NET 7+ via `IResult`) |

Helpers on `ControllerBase`: `Ok`, `Created`, `CreatedAtAction`, `NoContent`, `BadRequest`, `NotFound`, `Conflict`, `Problem`, `ValidationProblem`, `File`, `Accepted`, `StatusCode`.

### 5.1 [Easy] IActionResult vs ActionResult<T>?
**Answer**
`IActionResult` can return any HTTP result but carries no body type info. `ActionResult<T>` lets you return either a `T` (implicit 200) or an `ActionResult` (e.g. `NotFound()`), and gives Swagger the response type automatically.

**Explanation**
With `IActionResult` you must add `[ProducesResponseType<T>(200)]` for good docs. `ActionResult<T>` is a struct wrapper with implicit conversions from `T` and from `ActionResult`.

**Code**
```csharp
[HttpGet("{id:int}")]
public async Task<ActionResult<ProductDto>> Get(int id)
{
    var p = await repo.FindAsync(id);
    if (p is null) return NotFound();
    return p;                       // implicit -> Ok(p)
}
```

### 5.2 [Follow-up] Why can't you return `Ok(x)` as ActionResult<T> if x is a derived type or an interface, and how do you document multiple responses?
**Answer**
The implicit conversion is from exactly `T`, so returning a different type needs `Ok(...)` (which is an `ActionResult`) or a cast; `ActionResult<IEnumerable<X>>` can't implicitly convert from `List<X>` due to a lack of user-defined conversion variance — wrap in `Ok(list)`. Document additional status codes with `[ProducesResponseType]`.

**Explanation**
`[ProducesResponseType(StatusCodes.Status404NotFound)]` and `[ProducesProblem(404)]` populate OpenAPI. Because of `[ApiController]`, `ProducesResponseType` with ProblemDetails is inferred by Swashbuckle/`ApiExplorer` for common codes when `ApiConventionType` is used.

**Code**
```csharp
[HttpGet]
[ProducesResponseType<IEnumerable<ProductDto>>(StatusCodes.Status200OK)]
[ProducesResponseType(StatusCodes.Status404NotFound)]
public ActionResult<IEnumerable<ProductDto>> List()
{
    List<ProductDto> list = new();
    return Ok(list);   // `return list;` would fail to compile (List<T> -> IEnumerable<T> inside ActionResult<>)
}
```

### 5.3 [Medium] What are TypedResults and Results<T1,T2> and why are they better in minimal APIs?
**Answer**
`TypedResults` return concrete result types (`Ok<T>`, `NotFound`, `Created<T>`) rather than the opaque `IResult`, and `Results<Ok<T>, NotFound>` declares a union return type. This gives compile-time checking of possible responses, automatic OpenAPI metadata, and easier unit testing.

**Explanation**
In tests you can assert `Assert.IsType<Ok<Product>>(result.Result)` without executing the pipeline. `Results.X` (static factory) returns `IResult`, losing type info.

**Code**
```csharp
using Microsoft.AspNetCore.Http.HttpResults;
app.MapGet("/products/{id:int}", async Task<Results<Ok<ProductDto>, NotFound>> (int id, IProductRepo repo) =>
    await repo.FindAsync(id) is { } p ? TypedResults.Ok(p) : TypedResults.NotFound());
```

### 5.4 [Hard] Returning null from an action / ActionResult<T>: what happens, and how do you stream large sequences?
**Answer**
Returning `T` that is null from a plain `T` action yields 204 (via `HttpNoContentOutputFormatter`), but `ActionResult<T>` with `Ok(null)` yields 200 with `null` body — behaviors differ. For large sequences, return `IAsyncEnumerable<T>` so results are streamed, not buffered.

**Explanation**
Since .NET 6 the System.Text.Json output formatter streams `IAsyncEnumerable<T>` returned from MVC actions (items are written as they're produced, no full buffering); Newtonsoft.Json formatters still buffer it. In minimal APIs it is also streamed as an incrementally-written JSON array. Keep the DbContext alive until enumeration ends, and note that an exception mid-stream arrives after the 200 status was sent.

**Code**
```csharp
app.MapGet("/export", (AppDb db) => db.Orders.AsNoTracking().AsAsyncEnumerable()); // streamed JSON array
```

### 5.5 [Tricky] Unit-testing action results: why does `result.Value` vs `result.Result` matter?
**Answer**
`ActionResult<T>` has two properties: `Value` (set when you returned a raw `T`) and `Result` (set when you returned an `ActionResult` like `NotFound()` or `Ok(x)`). Asserting the wrong one gives null.

**Explanation**
`return p;` → `Value = p, Result = null`. `return Ok(p);` → `Result = OkObjectResult` with `Value` null on the wrapper. Tests must branch accordingly; that's one argument for consistently using `Ok(...)` or `TypedResults`.

**Code**
```csharp
var r = await controller.Get(1);          // returned `return p;`
Assert.NotNull(r.Value);                  // Result is null here
var r2 = await controller.Get(999);       // returned NotFound()
Assert.IsType<NotFoundResult>(r2.Result);
```

**Interview traps**
- `return list;` for `ActionResult<IEnumerable<T>>` compile error.
- Forgetting `ProducesResponseType` with `IActionResult`.
- Different null → 204 vs 200 behavior.
- Using `Results.X` and expecting OpenAPI types.

**Cheat sheet**
- `ActionResult<T>` = typed body + any error result.
- `Results<A,B>` union for minimal APIs.
- Test `.Value` vs `.Result`.
- Stream with `IAsyncEnumerable<T>` in minimal APIs.

---

## Topic 6: Filters & pipeline order

**Study notes**

Filter types (MVC/Razor Pages), in execution order:

```
Authorization filters   (IAuthorizationFilter / [Authorize])  - short-circuit with 401/403
  Resource filters      (IResourceFilter)  - before model binding; caching, short-circuit
    Model binding + validation
      Action filters    (IActionFilter / IAsyncActionFilter) - before/after action
        ACTION
      Exception filters (IExceptionFilter) - unhandled action/filter exceptions
    Result filters      (IResultFilter) - before/after result execution
      RESULT execution (formatters)
```

Within one type, order is global → controller → action on the way in, reverse on the way out; override with `IOrderedFilter.Order`. Short-circuit by setting `context.Result`. Exception filters do not catch exceptions from result execution or from resource filters' own code. Filters are different from middleware: middleware sees every request (including static files); filters only see MVC-selected actions and have access to `ActionContext`/model binding info.

DI for filters: `[ServiceFilter(typeof(X))]` (resolved from DI, must be registered), `[TypeFilter(typeof(X))]` (activated with `ActivatorUtilities`, not registered), or `IFilterFactory`.

### 6.1 [Easy] What types of filters exist and in what order do they run?
**Answer**
Authorization → Resource → Action → (Exception when failure) → Result. Authorization runs first and can short-circuit; result filters wrap formatting of the response.

**Explanation**
`IActionFilter` has `OnActionExecuting` (before; can set `Result` to short-circuit) and `OnActionExecuted`. Async variants take a `next` delegate. Never implement both sync and async interface of the same filter type — only the async one is called.

**Code**
```csharp
public class TimingFilter(ILogger<TimingFilter> log) : IAsyncActionFilter
{
    public async Task OnActionExecutionAsync(ActionExecutingContext ctx, ActionExecutionDelegate next)
    {
        var sw = System.Diagnostics.Stopwatch.StartNew();
        var executed = await next();            // runs action (+ inner filters)
        log.LogInformation("{Action} took {Ms}ms", ctx.ActionDescriptor.DisplayName, sw.ElapsedMilliseconds);
    }
}
// builder.Services.AddScoped<TimingFilter>(); [ServiceFilter(typeof(TimingFilter))]
```

### 6.2 [Follow-up] Filters vs middleware — when to use which?
**Answer**
Middleware is for cross-cutting HTTP concerns across the entire app (logging, CORS, auth, compression). Filters are for MVC-specific concerns that need action context, model state, or action results (validation, audit of action arguments, response shaping).

**Explanation**
Middleware cannot see which action is executing or the bound model (unless it uses endpoint metadata via `GetEndpoint()`), while filters cannot affect requests that never reach MVC. Authorization middleware (`UseAuthorization`) evaluates `[Authorize]` from endpoint metadata, so it runs in middleware, not as an MVC filter.

**Code**
```csharp
// Middleware: sees everything
app.Use(async (ctx, next) => { ctx.Response.Headers["X-Server"] = "api"; await next(); });
// Filter: only MVC actions, has bound arguments
public class AuditFilter : IActionFilter
{
    public void OnActionExecuting(ActionExecutingContext c) =>
        Console.WriteLine(string.Join(",", c.ActionArguments.Keys));
    public void OnActionExecuted(ActionExecutedContext c) { }
}
```

### 6.3 [Medium] How does short-circuiting work, and what's the execution order with nested filters?
**Answer**
Setting `context.Result` in `OnActionExecuting`/authorization/resource filters short-circuits the rest, running only the outer filters' "after" parts (result filters still run for action-filter short-circuits). Ordering: by `Order` ascending then scope (global → controller → action) on entry, reverse on exit.

**Explanation**
Example with global G, controller C, action A action filters, all Order 0: `G.Executing → C.Executing → A.Executing → action → A.Executed → C.Executed → G.Executed`. If `C.Executing` sets Result, `A` and the action never run, `C.Executed` and `G.Executed` do run (Canceled = true). Resource filter short-circuit skips the action filters and model binding but runs the result.

**Code**
```csharp
public class RequireApiKeyAttribute : ActionFilterAttribute
{
    public override void OnActionExecuting(ActionExecutingContext ctx)
    {
        if (!ctx.HttpContext.Request.Headers.ContainsKey("X-Api-Key"))
            ctx.Result = new UnauthorizedResult(); // short-circuit
    }
}
```

### 6.4 [Hard] How do exception filters differ from exception-handling middleware, and what don't they catch?
**Answer**
Exception filters handle unhandled exceptions from controller creation, model binding, action filters and the action itself, but not from other middleware, result execution (e.g., formatter failures after headers sent), or resource filters. Exception-handling middleware wraps the whole pipeline and is generally preferred as the global catch-all.

**Explanation**
Exception filters can set `context.Result` and `ExceptionHandled = true`. They're useful for MVC-specific mapping (domain exception → ProblemDetails) but middleware/`IExceptionHandler` (Topic 7) covers more cases consistently, including minimal APIs.

**Code**
```csharp
public class DomainExceptionFilter : IExceptionFilter
{
    public void OnException(ExceptionContext ctx)
    {
        if (ctx.Exception is KeyNotFoundException)
        {
            ctx.Result = new NotFoundObjectResult(new ProblemDetails { Status = 404, Title = "Not found" });
            ctx.ExceptionHandled = true;
        }
    }
}
// builder.Services.AddControllers(o => o.Filters.Add<DomainExceptionFilter>());
```

### 6.5 [Tricky] Why do my filter's constructor dependencies fail, and what is the filter-as-attribute DI problem?
**Answer**
Attributes are instantiated by the CLR with constant args, so they can't take DI constructor parameters. Use `[ServiceFilter]` (service must be registered) or `[TypeFilter]` (container fills dependencies, extra args allowed), or implement `IFilterFactory`.

**Explanation**
A filter deriving from `Attribute` with constructor dependencies and applied directly will fail to compile/run. Lifetimes: `ServiceFilter` resolves per request with the registered lifetime — a singleton filter can't hold scoped deps. `TypeFilter` instances may be cached (`IsReusable`).

**Code**
```csharp
[TypeFilter(typeof(AuditFilterImpl), Arguments = new object[] { "orders" })]
public IActionResult Post() => Ok();

public class AuditFilterImpl(string area, ILogger<AuditFilterImpl> log) : IActionFilter
{
    public void OnActionExecuting(ActionExecutingContext c) => log.LogInformation("Audit {Area}", area);
    public void OnActionExecuted(ActionExecutedContext c) { }
}
```

**Gotchas / follow-ups**
- Implementing both `IActionFilter` and `IAsyncActionFilter`: only async runs.
- Result filters don't run when an authorization filter short-circuits.
- Minimal APIs have `IEndpointFilter` instead (Topic 11).

**Interview traps**
- Thinking exception filters catch everything.
- Putting DI-heavy logic in plain attribute filters.
- Confusing order: Order value first, scope second.
- `[Authorize]` ≠ always an MVC filter (endpoint metadata + middleware).

**Cheat sheet**
- Order: Authorization → Resource → Action → Exception → Result.
- Short-circuit with `context.Result`.
- DI in filters: `ServiceFilter`/`TypeFilter`.
- Prefer middleware/IExceptionHandler for global errors.

---

## Topic 7: Global exception handling & ProblemDetails

**Study notes**

RFC 7807 (obsoleted by RFC 9457) `application/problem+json`: `type`, `title`, `status`, `detail`, `instance` plus extensions (e.g. `traceId`, `errors`).

Options in ASP.NET Core:
- `app.UseExceptionHandler("/error")` or `UseExceptionHandler(builder => ...)` — re-executes the pipeline on a path; resets the response.
- **`IExceptionHandler`** (.NET 8): `TryHandleAsync(HttpContext, Exception, CancellationToken)` returns `true` if handled; registered with `AddExceptionHandler<T>()`; multiple handlers run in registration order. Requires `app.UseExceptionHandler()` (and typically `AddProblemDetails()`).
- `AddProblemDetails()` + `IProblemDetailsService` produces ProblemDetails for exceptions (via exception handler middleware), status-code pages (`UseStatusCodePages`) and developer exception page.
- `UseDeveloperExceptionPage` — dev only (leaks details).
- Exception filters (MVC only).

Never leak stack traces in production; include `traceId` for correlation. Don't use exceptions for expected flow (validation) in hot paths.

### 7.1 [Easy] How do you implement global exception handling in .NET 8?
**Answer**
Register `AddProblemDetails()`, add one or more `IExceptionHandler` implementations with `AddExceptionHandler<T>()`, and enable `app.UseExceptionHandler()` early in the pipeline. Handlers map exceptions to status codes and write a ProblemDetails response.

**Explanation**
Put the exception middleware first so it wraps everything. Returning `false` from `TryHandleAsync` passes control to the next handler, and finally to the default behavior (500 + ProblemDetails when `AddProblemDetails` is registered).

**Code**
```csharp
public sealed class GlobalExceptionHandler(ILogger<GlobalExceptionHandler> log, IProblemDetailsService pds) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(HttpContext ctx, Exception ex, CancellationToken ct)
    {
        log.LogError(ex, "Unhandled exception");
        ctx.Response.StatusCode = ex is KeyNotFoundException ? 404 : 500;
        return await pds.TryWriteAsync(new ProblemDetailsContext
        {
            HttpContext = ctx,
            Exception = ex,
            ProblemDetails = { Title = ex is KeyNotFoundException ? "Not found" : "Server error", Status = ctx.Response.StatusCode }
        });
    }
}

builder.Services.AddProblemDetails();
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
var app = builder.Build();
app.UseExceptionHandler();
```

### 7.2 [Follow-up] What is ProblemDetails and how do you add extensions like traceId?
**Answer**
`ProblemDetails` is the standardized error body (RFC 7807/9457). Add custom data via `Extensions` or via `AddProblemDetails(o => o.CustomizeProblemDetails = ctx => ...)`.

**Explanation**
ASP.NET Core automatically adds `traceId` to ProblemDetails created through the default factory (`ProblemDetailsFactory`). `ValidationProblemDetails` adds an `errors` dictionary. Content type is `application/problem+json`.

**Code**
```csharp
builder.Services.AddProblemDetails(o =>
    o.CustomizeProblemDetails = ctx =>
    {
        ctx.ProblemDetails.Extensions["requestId"] = ctx.HttpContext.TraceIdentifier;
        ctx.ProblemDetails.Instance = $"{ctx.HttpContext.Request.Method} {ctx.HttpContext.Request.Path}";
    });
```

### 7.3 [Medium] How do you map domain/validation exceptions without a giant switch, and keep the middleware order right?
**Answer**
Use chained `IExceptionHandler`s: one per exception family (e.g. `ValidationExceptionHandler`, `NotFoundExceptionHandler`) plus a catch-all last. Order of registration = order of execution. Place `UseExceptionHandler` before routing/auth so it catches errors from later middleware.

**Explanation**
Handlers are singletons (resolved once), so don't inject scoped services directly; use `IServiceScopeFactory`. Also `UseStatusCodePages()` converts bodyless 4xx/5xx (like bare 404 from routing) into ProblemDetails when `AddProblemDetails` is registered.

**Code**
```csharp
public sealed class ValidationExceptionHandler : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(HttpContext ctx, Exception ex, CancellationToken ct)
    {
        if (ex is not FluentValidation.ValidationException ve) return false;
        ctx.Response.StatusCode = 400;
        var errors = ve.Errors.GroupBy(e => e.PropertyName)
                              .ToDictionary(g => g.Key, g => g.Select(e => e.ErrorMessage).ToArray());
        await ctx.Response.WriteAsJsonAsync(new ValidationProblemDetails(errors) { Status = 400 }, cancellationToken: ct);
        return true;
    }
}
builder.Services.AddExceptionHandler<ValidationExceptionHandler>();
builder.Services.AddExceptionHandler<GlobalExceptionHandler>(); // fallback, registered last
```

### 7.4 [Hard] What happens if the response has already started when an exception occurs?
**Answer**
The exception handler middleware can't change the status code or headers; it rethrows (or logs) and the connection is typically aborted, leaving the client with a truncated body. You can't "fix" a started response.

**Explanation**
`ExceptionHandlerMiddleware` checks `Response.HasStarted`; if true it logs "The response has already started, the error handler will not be executed" and rethrows. Mitigations: buffer before writing, validate before streaming, and for streaming APIs design a trailer/terminal error frame (e.g. NDJSON error line or SSE error event). Also, `OperationCanceledException` caused by client disconnect (`HttpContext.RequestAborted`) should not be logged as an error or turned into 500 — treat as 499-style (nginx convention) / ignore.

**Code**
```csharp
public async ValueTask<bool> TryHandleAsync(HttpContext ctx, Exception ex, CancellationToken ct)
{
    if (ex is OperationCanceledException && ctx.RequestAborted.IsCancellationRequested)
        return true;                      // client left; nothing to write
    return false;
}
```

### 7.5 [Tricky] Why is returning ex.Message to the client dangerous, and why do exceptions in minimal API endpoint filters/`Task.Run` behave differently?
**Answer**
Exception messages can leak internals (SQL, paths, connection strings) — return a generic title and a correlation id, log details server-side. Exceptions on fire-and-forget threads (`Task.Run`, un-awaited tasks) never reach the middleware, so they're silently lost or crash via unobserved handling.

**Explanation**
Exceptions thrown in endpoint filters do propagate to exception-handler middleware since filters run inside the endpoint. For background work use `BackgroundService`/queue with its own try/catch and logging. In Development you may expose `Exception` details via `IHostEnvironment.IsDevelopment()` guarded branch.

**Code**
```csharp
app.MapGet("/boom", () => { throw new InvalidOperationException("secret connection string"); });
// Production response: {"title":"An error occurred while processing your request.","status":500,"traceId":"..."}
```

**Interview traps**
- Putting `UseExceptionHandler` after other middleware.
- Using exceptions for validation flow (cost, noise).
- Injecting scoped services into an `IExceptionHandler` (singleton).
- Leaking stack traces in prod.

**Cheat sheet**
- `AddProblemDetails` + `AddExceptionHandler<T>` + `UseExceptionHandler()`.
- Handler returns bool: true = handled.
- ProblemDetails = RFC 7807/9457, `application/problem+json`.
- Can't change response once started.

---

## Topic 8: API versioning

**Study notes**

Package: `Asp.Versioning.Http` / `Asp.Versioning.Mvc` / `Asp.Versioning.Mvc.ApiExplorer` (successors of Microsoft.AspNetCore.Mvc.Versioning).

| Strategy | Example | Pros | Cons |
|---|---|---|---|
| URL segment | `/api/v2/orders` | explicit, cacheable, easy to route/test | URL changes, not "pure" REST |
| Query string | `/orders?api-version=2.0` | simple | pollutes cache keys |
| Header | `X-Api-Version: 2.0` | clean URLs | hidden, harder to test in browser |
| Media type | `Accept: application/vnd.acme.v2+json` | RESTful (representation versioning) | complex |

Version only on breaking changes: removing/renaming fields, changing types/semantics. Additive changes (new optional field) are non-breaking. Deprecation: `[ApiVersion("1.0", Deprecated = true)]`, report via `api-supported-versions` / `api-deprecated-versions` headers (`ReportApiVersions = true`); communicate sunset with `Sunset` and `Deprecation` headers.

### 8.1 [Easy] Why version an API and what are the common strategies?
**Answer**
To evolve the contract without breaking existing clients. Common strategies are URL segment, query string, header and media type; URL segment is the most popular for simplicity.

**Explanation**
Prefer additive, backward-compatible changes (tolerant reader) to avoid versions entirely; version only when breaking. Each live version has maintenance cost, so define a deprecation policy.

**Code**
```csharp
using Asp.Versioning;
builder.Services.AddApiVersioning(o =>
{
    o.DefaultApiVersion = new ApiVersion(1, 0);
    o.AssumeDefaultVersionWhenUnspecified = true;
    o.ReportApiVersions = true;
    o.ApiVersionReader = ApiVersionReader.Combine(
        new UrlSegmentApiVersionReader(),
        new HeaderApiVersionReader("X-Api-Version"));
}).AddMvc();
```

### 8.2 [Follow-up] How do you declare versions on controllers?
**Answer**
Use `[ApiVersion("1.0")]` / `[ApiVersion("2.0")]` on controllers and `[MapToApiVersion]` on actions; for URL versioning use `{version:apiVersion}` in the route.

**Explanation**
One controller can serve several versions, or split into `V1`/`V2` namespaces/classes. Use `[ApiVersionNeutral]` for health endpoints. With URL segment versioning the route template needs the `apiVersion` constraint.

**Code**
```csharp
[ApiController]
[ApiVersion("1.0")]
[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/orders")]
public class OrdersController : ControllerBase
{
    [HttpGet, MapToApiVersion("1.0")] public IActionResult GetV1() => Ok(new { name = "o" });
    [HttpGet, MapToApiVersion("2.0")] public IActionResult GetV2() => Ok(new { name = "o", currency = "USD" });
}
```

### 8.3 [Medium] How does versioning work with minimal APIs?
**Answer**
Build a version set with `app.NewApiVersionSet()` and attach it to endpoints/groups via `.WithApiVersionSet(set).HasApiVersion(...)`. Use `MapToApiVersion` per endpoint when sharing a route.

**Explanation**
Requires `Asp.Versioning.Http`. `AddApiExplorer(o => o.GroupNameFormat = "'v'VVV")` for Swagger documents per version. Use `ReportApiVersions()` on the set.

**Code**
```csharp
var versions = app.NewApiVersionSet()
    .HasApiVersion(new ApiVersion(1, 0))
    .HasApiVersion(new ApiVersion(2, 0))
    .ReportApiVersions()
    .Build();

var g = app.MapGroup("/api/v{version:apiVersion}/orders").WithApiVersionSet(versions);
g.MapGet("/", () => "v1").MapToApiVersion(1, 0);
g.MapGet("/", () => "v2").MapToApiVersion(2, 0);
```

### 8.4 [Hard] How do you evolve a contract without versions, and what is a breaking change?
**Answer**
Follow Postel's/tolerant-reader rules: only add optional fields/endpoints, never remove or change meaning, and make clients ignore unknown fields. Breaking changes include removing/renaming a field, changing a type/format/enum meaning, tightening validation, changing status codes or required headers.

**Explanation**
Enum additions can break strict clients. Use contract tests (Pact/OpenAPI diff tools like `oasdiff`) in CI to detect breaks. Stash old behavior behind the same DTO via mapping layers so v1 and v2 DTOs share one domain/service layer — don't fork business logic per version.

**Code**
```csharp
public record OrderV1(int Id, string Name);
public record OrderV2(int Id, string Name, string Currency);
static class OrderMap
{
    public static OrderV1 ToV1(this Order o) => new(o.Id, o.Name);
    public static OrderV2 ToV2(this Order o) => new(o.Id, o.Name, o.Currency);
}
public record Order(int Id, string Name, string Currency);
```

### 8.5 [Tricky] An unversioned request arrives with AssumeDefaultVersionWhenUnspecified = true — what are the risks?
**Answer**
Unversioned clients get pinned to the default version, so changing the default later silently breaks them. Prefer keeping the default fixed at the oldest supported version or require explicit versions.

**Explanation**
Also with URL-segment versioning the route is required, so the assumption only helps for query/header readers. `UnsupportedApiVersion` returns 400 with an error body; unmatched version for a route returns 404/405. Combining readers makes "which one wins" ambiguous: if several readers supply different values, the request is rejected as ambiguous.

**Code**
```csharp
o.AssumeDefaultVersionWhenUnspecified = false;  // forces clients to be explicit (400 if absent)
```

**Interview traps**
- Versioning on every change.
- Duplicating business logic per version.
- Changing the default version in place.
- Claiming URL versioning isn't RESTful as a deal-breaker — it's a pragmatic tradeoff.

**Cheat sheet**
- Package: `Asp.Versioning.*`.
- Strategies: URL / query / header / media type.
- Version only for breaking changes.
- `ReportApiVersions` + Sunset header for deprecation.

---

## Topic 9: Pagination, filtering, sorting

**Study notes**

| | Offset (`page`/`pageSize`, `OFFSET/FETCH`) | Keyset / cursor (`WHERE (k) > @last`) |
|---|---|---|
| Performance | degrades with deep pages (DB scans/skips N rows) | constant, uses index seek |
| Consistency | rows shift when data inserted/deleted (duplicates/skips) | stable |
| Random page jump | yes | no (next/prev only) |
| Total count | easy (extra COUNT) | usually omitted |
| Sort | any | requires deterministic unique ordering (add tie-breaker like Id) |

Always enforce a max page size, a deterministic `ORDER BY` (otherwise results are non-deterministic in SQL Server), and whitelist sortable/filterable fields (never `OrderBy(string)` straight from input — injection/perf risk). Return pagination metadata in body or `Link` headers (RFC 8288), e.g. `X-Total-Count`.

SQL Server offset: `ORDER BY Id OFFSET @skip ROWS FETCH NEXT @take ROWS ONLY` (EF: `Skip().Take()`).

### 9.1 [Easy] How do you implement offset pagination in EF Core?
**Answer**
Order deterministically, `Skip((page-1)*size).Take(size)`, and return the items with total count and page info. Cap `pageSize`.

**Explanation**
`CountAsync` is a second query; on huge tables it's expensive — consider approximate or omit. EF translates to `OFFSET ... FETCH NEXT` on SQL Server (requires `ORDER BY`).

**Code**
```csharp
public record PagedResult<T>(IReadOnlyList<T> Items, int Page, int PageSize, int TotalCount);

public static async Task<PagedResult<T>> ToPagedAsync<T>(this IOrderedQueryable<T> q, int page, int size, CancellationToken ct)
{
    page = Math.Max(page, 1); size = Math.Clamp(size, 1, 100);
    var total = await q.CountAsync(ct);
    var items = await q.Skip((page - 1) * size).Take(size).ToListAsync(ct);
    return new(items, page, size, total);
}
// usage: db.Orders.AsNoTracking().OrderBy(o => o.Id).ToPagedAsync(page, size, ct)
```

### 9.2 [Follow-up] What's wrong with offset pagination at scale, and how does keyset pagination fix it?
**Answer**
`OFFSET N` makes the DB read and discard N rows, so deep pages are slow, and concurrent inserts/deletes cause duplicated or skipped items. Keyset uses `WHERE key > lastKey ORDER BY key` with an index seek: constant cost and stable.

**Explanation**
Fetch `size + 1` rows to know whether there's a next page. For multi-column sort (e.g. `CreatedAt DESC, Id DESC`) the predicate must be `(CreatedAt < @c) OR (CreatedAt = @c AND Id < @id)`. Encode the cursor as opaque Base64 (so clients don't depend on its shape, and you can change it); sign it if tampering matters.

**Code**
```csharp
public record Cursor(DateTime CreatedAt, int Id);

var q = db.Orders.AsNoTracking();
if (cursor is not null)
    q = q.Where(o => o.CreatedAt < cursor.CreatedAt ||
                     (o.CreatedAt == cursor.CreatedAt && o.Id < cursor.Id));
var rows = await q.OrderByDescending(o => o.CreatedAt).ThenByDescending(o => o.Id)
                  .Take(size + 1).ToListAsync(ct);
var hasMore = rows.Count > size;
var page = rows.Take(size).ToList();
string? next = hasMore ? Convert.ToBase64String(
    System.Text.Json.JsonSerializer.SerializeToUtf8Bytes(new Cursor(page[^1].CreatedAt, page[^1].Id))) : null;
```

### 9.3 [Medium] How do you safely implement dynamic sorting and filtering?
**Answer**
Whitelist the allowed fields and map each to an expression; never concatenate user input into SQL or use dynamic LINQ strings without validation. Combine filters via composable `IQueryable` `Where` clauses only for provided parameters.

**Explanation**
Raw `OrderBy(propertyName)` via reflection works but permits sorting by unindexed/unauthorized columns (perf DoS, data inference). Provide indexes matching common sort+filter combos. For rich filter grammars consider OData (`$filter`, `$orderby`, `$top`) knowing its security/perf surface (limit `MaxTop`, allowed functions).

**Code**
```csharp
static readonly Dictionary<string, Expression<Func<Order, object>>> Sortable = new(StringComparer.OrdinalIgnoreCase)
{
    ["date"] = o => o.CreatedAt,
    ["total"] = o => o.Total
};

IQueryable<Order> Apply(IQueryable<Order> q, string? sort, bool desc, string? status)
{
    if (!string.IsNullOrEmpty(status)) q = q.Where(o => o.Status == status);
    var key = sort is not null && Sortable.TryGetValue(sort, out var e) ? e : Sortable["date"];
    return desc ? q.OrderByDescending(key).ThenByDescending(o => o.Id)
                : q.OrderBy(key).ThenBy(o => o.Id);
}
```

### 9.4 [Hard] Describe how to expose pagination via headers/links and how `Skip/Take` without OrderBy bites you.
**Answer**
Return `Link: <...?page=2>; rel="next"` headers (RFC 8288) and optionally `X-Total-Count`, or embed `next`/`prev` URLs in the body. Without a unique, deterministic `ORDER BY`, SQL Server may return rows in any order and pages can overlap or miss rows.

**Explanation**
EF Core logs a warning when `Skip/Take` is used without `OrderBy`. Non-unique sort keys (e.g. `CreatedAt` only) yield unstable paging; always append a unique tie-breaker (Id). Cursor pagination across changing sort values (e.g. sorting by "score" that updates) is inherently unstable — snapshot or use a stable key.

**Code**
```csharp
[HttpGet]
public async Task<ActionResult<List<OrderDto>>> List(int page = 1, int size = 20, CancellationToken ct = default)
{
    size = Math.Clamp(size, 1, 100);
    var q = db.Orders.AsNoTracking().OrderBy(o => o.Id);
    var total = await q.CountAsync(ct);
    Response.Headers["X-Total-Count"] = total.ToString();
    if (page * size < total)
        Response.Headers.Append("Link", $"<{Url.Action(null, new { page = page + 1, size })}>; rel=\"next\"");
    return await q.Skip((page - 1) * size).Take(size).Select(o => new OrderDto(o.Id)).ToListAsync(ct);
}
```

### 9.5 [Tricky] Your keyset pagination returns duplicates when sorting by a nullable column. Why?
**Answer**
SQL comparison with NULL is unknown, so `WHERE col > @last` drops NULL rows and tie handling breaks. Make the sort key non-null (COALESCE / computed column) or handle NULLs explicitly in the predicate, and always include a unique tie-breaker.

**Explanation**
SQL Server sorts NULLs first in ascending order. A cursor with a NULL key needs `col IS NULL AND Id > @id` logic, then `col IS NOT NULL` rows. Easiest approach: don't paginate on nullable columns, or index a persisted computed non-null column.

**Code**
```csharp
// Sort on a non-null key: map NULL to a sentinel so the predicate is well-defined
var key = db.Orders.Select(o => new { o.Id, Score = o.Score ?? int.MinValue });   // Score is int?
var page = await key.Where(x => x.Score > last.Score || (x.Score == last.Score && x.Id > last.Id))
                    .OrderBy(x => x.Score).ThenBy(x => x.Id)
                    .Take(size).ToListAsync(ct);
// Better: persist a computed non-null column and index it (Score, Id).
```

**Interview traps**
- No max page size (client asks for `pageSize=1000000`).
- `Skip/Take` without `OrderBy`.
- Sorting by a non-unique column only.
- Exposing raw DB column names/dynamic `OrderBy(string)`.
- Always running `COUNT(*)` on giant tables.

**Cheat sheet**
- Offset: simple, jump pages, slow deep + unstable.
- Keyset: fast, stable, next/prev only; opaque cursor.
- Always: max size, deterministic order, unique tie-breaker.
- Whitelist sort/filter fields.

---

## Topic 10: DTOs & mapping

**Study notes**

DTOs decouple the wire contract from persistence/domain models: prevent over-posting, avoid leaking fields (password hash), avoid circular references/lazy-loading serialization, allow independent evolution/versioning, and shape payloads.

Mapping options:
- **Manual** (extension methods/constructors): explicit, compile-time safe, fastest, debuggable; boilerplate.
- **Mapster**: source-gen (`Mapster.Tool`) or runtime-compiled expression mapping; fast.
- **AutoMapper**: convention-based, `ProjectTo<T>` for EF; reflection/expression compile at config; hidden magic, runtime failures (`AssertConfigurationIsValid` in tests). AutoMapper became commercial (v15+ requires a license key) — a factor in many teams moving away.
- **Mapperly**: compile-time source generator, zero reflection.

For reads, project in the query (`Select`) so EF fetches only needed columns; avoid loading entities then mapping.

### 10.1 [Easy] Why use DTOs rather than returning EF entities?
**Answer**
They hide internal fields, prevent over-posting on input, avoid circular navigation-property serialization and lazy-loading surprises, and let the API contract evolve independently from the database schema.

**Explanation**
Binding an entity directly to the request allows a client to set `IsAdmin` or `Id`. Separate DTOs per use case (Create/Update/View) are clearer than one "God DTO".

**Code**
```csharp
public record CreateProductRequest(string Name, decimal Price);
public record ProductResponse(int Id, string Name, decimal Price);

public static class ProductMappings
{
    public static Product ToEntity(this CreateProductRequest r) => new() { Name = r.Name, Price = r.Price };
    public static ProductResponse ToResponse(this Product p) => new(p.Id, p.Name, p.Price);
}
```

### 10.2 [Follow-up] Manual mapping vs AutoMapper vs Mapster — how do you choose?
**Answer**
Prefer manual or source-generated mappers (Mapperly/Mapster tool) for compile-time safety, performance and easy debugging; use AutoMapper only where convention mapping and `ProjectTo` meaningfully cut boilerplate and you accept runtime-config risk (and its licensing).

**Explanation**
Common AutoMapper pitfalls: silently unmapped members, mapping hiding business logic, N+1 queries when mapping navigation collections after loading entities, and `ProjectTo` requirement of expressions only. Add a unit test calling `config.AssertConfigurationIsValid()`.

**Code**
```csharp
// AutoMapper (≤ v14 API shape)
var cfg = new MapperConfiguration(c => c.CreateMap<Product, ProductResponse>());
cfg.AssertConfigurationIsValid();
var list = await db.Products.ProjectTo<ProductResponse>(cfg).ToListAsync(ct); // SELECT only needed columns

// Mapster
using Mapster;
var dto = product.Adapt<ProductResponse>();
```

### 10.3 [Medium] How do projections to DTOs in EF Core improve performance, and what can't be translated?
**Answer**
`Select(p => new ProductResponse(...))` makes EF generate SQL selecting only those columns, with no change tracking. Custom C# methods or constructors with unsupported logic inside the projection can't be translated and may force client evaluation (throw in EF Core 3+ for the top-level query).

**Explanation**
Method calls are allowed only in the *final* `Select` projection, where EF evaluates them client-side after fetching. Using `ProjectTo` or an expression-bodied `Expression<Func<Product, ProductResponse>>` field keeps it reusable and translatable.

**Code**
```csharp
public static readonly Expression<Func<Product, ProductResponse>> ToDto =
    p => new ProductResponse(p.Id, p.Name, p.Price);

var items = await db.Products.AsNoTracking().Select(ToDto).ToListAsync(ct);
```

### 10.4 [Hard] How do you avoid over-posting and handle partial updates safely with DTOs?
**Answer**
Bind to a request DTO exposing only updatable fields, load the entity, apply allowed changes explicitly, and let EF's change tracker emit a minimal `UPDATE`. For PATCH, distinguish "absent" from "null" using a tri-state wrapper or JSON Merge Patch.

**Explanation**
Avoid `db.Update(entityFromClient)` (marks all columns modified, allows setting any column). Mapping onto an existing tracked entity (`mapper.Map(dto, entity)`) with null-ignoring rules wrongly makes it impossible to set a field to null. Combine with concurrency tokens (Topic 14).

**Code**
```csharp
public readonly record struct Optional<T>(bool IsSet, T? Value);
// custom JsonConverter would populate IsSet when the property is present
app.MapPatch("/products/{id:int}", async (int id, UpdateProduct dto, AppDb db) =>
{
    var p = await db.Products.FindAsync(id);
    if (p is null) return Results.NotFound();
    if (dto.Name.IsSet) p.Name = dto.Name.Value!;
    await db.SaveChangesAsync();
    return Results.NoContent();
});
record UpdateProduct(Optional<string> Name);
```

**Interview traps**
- "AutoMapper is a must" — it's optional and has real costs.
- Mapping after `ToList()` of entities with Includes instead of projecting.
- Same DTO for input and output.
- `db.Update(clientObject)`.

**Cheat sheet**
- DTO = contract; entity = persistence.
- Project with `Select` / `ProjectTo`.
- Prefer manual/Mapperly/Mapster-tool for safety.
- Separate Create/Update/Response DTOs.

---

## Topic 11: Minimal APIs deep dive

**Study notes**

Minimal APIs (`WebApplication`, `MapGet/MapPost/...`) route directly to delegates, with lower overhead than MVC (no controller activation, filters pipeline). Features by .NET 8: route groups (`MapGroup`), endpoint filters (`IEndpointFilter`), `TypedResults` + `Results<...>`, `[AsParameters]`, parameter binding (route, query, header, body, services, `HttpContext`, `CancellationToken` inferred), keyed services, `RequestDelegateGenerator`/AOT support (`WebApplication.CreateSlimBuilder`), antiforgery for forms (.NET 8), `WithOpenApi()` metadata, `.RequireAuthorization()`, `.RequireRateLimiting()`, `.WithTags()`, `.Produces<T>()`.

Not built-in (.NET 8): automatic validation, MVC-style filters, model-binding for complex query/form types w/o `[AsParameters]`, content negotiation (always JSON via `Results.Json`/`TypedResults`, unless you handle `Accept` yourself).

Endpoint filter order: filters run in the order added, wrapping the handler (like middleware); group filters run before endpoint filters.

### 11.1 [Easy] What are route groups and why use them?
**Answer**
`MapGroup` creates a common prefix and shared configuration (auth, filters, tags, rate limiting) applied to all endpoints inside. It removes repetition and lets you organize endpoints in extension methods per feature.

**Explanation**
Groups nest, and conventions added on the group apply to every endpoint (added before or after, as conventions run at build time). A group can use `.WithTags`, `.RequireAuthorization`, `.AddEndpointFilter`, `.WithOpenApi`.

**Code**
```csharp
public static class OrderEndpoints
{
    public static IEndpointRouteBuilder MapOrders(this IEndpointRouteBuilder app)
    {
        var g = app.MapGroup("/orders").RequireAuthorization().WithTags("Orders");
        g.MapGet("/{id:int}", (int id) => TypedResults.Ok(new { id }));
        g.MapPost("/", (CreateOrder o) => TypedResults.Created($"/orders/1", o));
        return app;
    }
}
record CreateOrder(string Item);
// Program.cs: app.MapOrders();
```

### 11.2 [Follow-up] How do endpoint filters work, and how do they compare to MVC action filters?
**Answer**
`IEndpointFilter.InvokeAsync(context, next)` wraps the handler: run logic before calling `next(context)`, inspect/modify arguments via `context.Arguments`, short-circuit by returning a result, and post-process the returned value. Unlike MVC filters there's a single filter type (no auth/resource/exception stages).

**Explanation**
Filters can be added as lambdas (`.AddEndpointFilter(async (ctx, next) => ...)`), as typed `IEndpointFilter` classes, or via a factory (`AddEndpointFilterFactory`) which gets `EndpointFilterFactoryContext` for per-endpoint metadata (parameter reflection) at build time — more efficient. Exceptions propagate to exception middleware.

**Code**
```csharp
app.MapGet("/hello/{name}", (string name) => $"Hello {name}")
   .AddEndpointFilter(async (ctx, next) =>
   {
       var name = ctx.GetArgument<string>(0);
       if (name.Length < 2) return Results.BadRequest("name too short");
       var result = await next(ctx);
       return result;
   });
```

### 11.3 [Medium] Explain TypedResults, `Results<T1,T2>`, and OpenAPI metadata inference.
**Answer**
`TypedResults.Ok(x)` returns `Ok<T>`; a handler declared as `Results<Ok<T>, NotFound>` lets the framework infer response metadata (200 with `T`, 404) for OpenAPI without `.Produces` calls. `Results.X` returns `IResult` and requires manual `.Produces<T>()`.

**Explanation**
`Results<...>` supports up to 6 type arguments; implicit conversion lets the handler return any one of them. Handlers returning plain types produce 200 (strings as text/plain, objects as JSON). Void-/Task-returning handlers produce 200 empty.

**Code**
```csharp
app.MapPost("/todos", async Task<Results<Created<Todo>, ValidationProblem>> (Todo t, TodoDb db) =>
{
    if (string.IsNullOrWhiteSpace(t.Title))
        return TypedResults.ValidationProblem(new Dictionary<string, string[]> { ["title"] = ["required"] });
    db.Add(t); await db.SaveChangesAsync();
    return TypedResults.Created($"/todos/{t.Id}", t);
});
```

### 11.4 [Hard] How does parameter binding resolve ambiguity, and what throws at startup?
**Answer**
Order of inference: explicit attribute → special types (`HttpContext`, `HttpRequest`, `CancellationToken`, `ClaimsPrincipal`, `IFormFile`) → `BindAsync` → `TryParse` → registered service (if the type is a service per `IServiceProviderIsService`) → body for complex types. Two inferred body parameters, or a body on GET without explicit `[FromBody]`, throw `InvalidOperationException` at startup (endpoint build).

**Explanation**
A parameter that matches a route token binds from route; otherwise simple types bind from query. Nullable/optional parameters (`int? x`, defaults) are optional; non-nullable missing query values give automatic 400 (in minimal APIs "Required parameter not provided" is a 400 logged at Debug). Failure of JSON body deserialization is also 400.

**Code**
```csharp
app.MapGet("/search", (string q, int page, ILogger<Program> log, CancellationToken ct, HttpContext ctx) =>
    Results.Ok(new { q, page }));
// GET /search?q=a            -> 400 (page is required, non-nullable)
// GET /search?q=a&page=2     -> 200
```

### 11.5 [Tricky] Minimal APIs vs controllers — what do you lose and what about large projects?
**Answer**
You lose built-in model validation, MVC filter stages, formatters/content negotiation, `[ApiController]` conventions and, to an extent, convention-based discoverability. Large apps stay maintainable with route groups, feature-folder endpoint classes, shared filters and DI — performance and AOT are the upside.

**Explanation**
Minimal API handlers are static lambdas or methods; use local functions or static methods rather than capturing lambdas to reduce allocations. Don't capture request-scoped state in closures. Handlers run with scoped DI per request, but a lambda capturing a scoped service from `Program` scope is a bug. Endpoint metadata (`.WithName`, `.WithGroupName`) is needed for link generation (`LinkGenerator`) and OpenAPI operationIds.

**Code**
```csharp
app.MapGet("/ok", Handlers.Get);
static class Handlers
{
    public static IResult Get(HttpContext ctx) => TypedResults.Ok(ctx.TraceIdentifier);
}
```

**Interview traps**
- Expecting DataAnnotations validation in .NET 8 minimal APIs.
- Returning `Results.X` and expecting OpenAPI response types.
- Body-bound GET endpoints.
- Capturing scoped services in startup closures.

**Cheat sheet**
- `MapGroup` = prefix + shared conventions.
- `IEndpointFilter` = wrap handler; group filters first.
- `TypedResults` + `Results<>` = typed + OpenAPI.
- No built-in validation/negotiation in .NET 8.

---

## Topic 12: Content negotiation, formatters, System.Text.Json

**Study notes**

MVC content negotiation: `Accept` header → output formatter chosen from `MvcOptions.OutputFormatters` (default JSON; add XML via `AddXmlSerializerFormatters()`). `Content-Type` of request → input formatter. Unsupported request media type → 415; unacceptable `Accept` → by default JSON anyway unless `MvcOptions.ReturnHttpNotAcceptable = true` → 406. `[Produces("application/json")]`/`[Consumes]` restrict. Browsers' `Accept: */*` leads to the first formatter; `RespectBrowserAcceptHeader` is off by default.

System.Text.Json (STJ) defaults in ASP.NET Core: `JsonSerializerDefaults.Web` — camelCase names, case-insensitive property matching on read, numbers can be read from strings. Configure with `AddJsonOptions` (MVC) or `ConfigureHttpJsonOptions` (minimal APIs) — these are **separate** options objects.

Common knobs: `PropertyNamingPolicy`, `DefaultIgnoreCondition = WhenWritingNull`, `JsonStringEnumConverter`, `ReferenceHandler.IgnoreCycles`, `NumberHandling`, custom `JsonConverter<T>`, source generation (`JsonSerializerContext`) for AOT/perf, `[JsonPropertyName]`, `[JsonIgnore]`, `[JsonPolymorphic]`/`[JsonDerivedType]` (.NET 7+).

### 12.1 [Easy] How does content negotiation work in ASP.NET Core?
**Answer**
The server looks at the request `Accept` header and picks the first output formatter that can write the return type to an acceptable media type; the default is JSON. Request bodies are parsed using the `Content-Type` header, or 415 if unsupported.

**Explanation**
Add formatters (XML, CSV custom). Set `ReturnHttpNotAcceptable = true` for strict 406 behavior. Minimal APIs do not negotiate — they write JSON (or whatever the `IResult` writes).

**Code**
```csharp
builder.Services.AddControllers(o => o.ReturnHttpNotAcceptable = true)
                .AddXmlSerializerFormatters();
// GET with Accept: application/xml -> XML; Accept: text/csv -> 406
```

### 12.2 [Follow-up] How do you configure System.Text.Json for controllers and minimal APIs?
**Answer**
Controllers: `AddControllers().AddJsonOptions(...)`. Minimal APIs: `builder.Services.ConfigureHttpJsonOptions(...)`. They are two separate option sets, so configure both if you use both.

**Explanation**
Typical config: string enums, ignore nulls, ignore cycles. `JsonStringEnumConverter` makes enums serialize by name (otherwise ints); consider `allowIntegerValues: false` to reject numbers.

**Code**
```csharp
using System.Text.Json; using System.Text.Json.Serialization;
builder.Services.AddControllers().AddJsonOptions(o =>
{
    o.JsonSerializerOptions.Converters.Add(new JsonStringEnumConverter());
    o.JsonSerializerOptions.DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull;
});
builder.Services.ConfigureHttpJsonOptions(o =>
{
    o.SerializerOptions.Converters.Add(new JsonStringEnumConverter());
    o.SerializerOptions.DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull;
});
```

### 12.3 [Medium] Write a custom JsonConverter and register it per type vs globally.
**Answer**
Derive `JsonConverter<T>`, implement `Read`/`Write`, then apply via `[JsonConverter(typeof(...))]` on the type/property or add to `Converters` globally.

**Explanation**
In `Read` use `Utf8JsonReader` (a ref struct, can't be captured in lambdas/async). Always honor the token type. Precedence: `[JsonConverter]` on the property, then converters in `options.Converters`, then `[JsonConverter]` on the type.

**Code**
```csharp
public sealed class DateOnlyConverter : JsonConverter<DateOnly>
{
    public override DateOnly Read(ref Utf8JsonReader r, Type t, JsonSerializerOptions o) =>
        DateOnly.ParseExact(r.GetString()!, "yyyy-MM-dd");
    public override void Write(Utf8JsonWriter w, DateOnly v, JsonSerializerOptions o) =>
        w.WriteStringValue(v.ToString("yyyy-MM-dd"));
}
// Output of JsonSerializer.Serialize(new DateOnly(2024,1,5), opts with converter): "2024-01-05"
```

### 12.4 [Hard] Polymorphism, source generation and what breaks versus Newtonsoft.
**Answer**
STJ serializes the *declared* type's properties unless you use `object` or `[JsonPolymorphic]`/`[JsonDerivedType]` (.NET 7+), which write a type discriminator. Source-generated contexts (`[JsonSerializable]`) remove reflection for speed and trimming/AOT.

**Explanation**
Behavior differences from Newtonsoft: no `JObject`/`dynamic` (use `JsonNode`/`JsonDocument`), stricter by default (no comments/trailing commas unless enabled, no single quotes), no implicit string→number except via `NumberHandling`, private setters ignored without `[JsonInclude]`, fields ignored unless `IncludeFields`, cycles throw unless `ReferenceHandler.IgnoreCycles` (.NET 6+). Property-name case: serialization uses policy; deserialization is case-insensitive in `Web` defaults.

**Code**
```csharp
[JsonPolymorphic(TypeDiscriminatorPropertyName = "kind")]
[JsonDerivedType(typeof(Circle), "circle")]
[JsonDerivedType(typeof(Square), "square")]
public abstract record Shape;
public record Circle(double R) : Shape;
public record Square(double S) : Shape;

Shape s = new Circle(2);
Console.WriteLine(JsonSerializer.Serialize(s));
// Output: {"kind":"circle","R":2}   (no naming policy applied with default options)

[JsonSerializable(typeof(Shape))]
internal partial class AppJsonContext : JsonSerializerContext { }
```

### 12.5 [Tricky] Why does the same DTO serialize differently from `Results.Ok` vs `JsonSerializer.Serialize` vs Controller?
**Answer**
Each uses different options: `JsonSerializer.Serialize` uses plain defaults (PascalCase), MVC uses `JsonOptions` from `AddJsonOptions` (camelCase), minimal APIs use `Microsoft.AspNetCore.Http.Json.JsonOptions`. Misaligned config is a classic source of mismatched casing/enums.

**Explanation**
Always pass the same `JsonSerializerOptions` instance (inject `IOptions<JsonOptions>`) if you serialize manually (e.g. caching/logging payloads). Also: STJ is case-sensitive on write names but `Web` defaults give case-insensitive read; `[JsonPropertyName]` is exact and always respected.

**Code**
```csharp
var plain = JsonSerializer.Serialize(new { FirstName = "A" });
// Output: {"FirstName":"A"}
var web = JsonSerializer.Serialize(new { FirstName = "A" }, new JsonSerializerOptions(JsonSerializerDefaults.Web));
// Output: {"firstName":"A"}
```

**Interview traps**
- Configuring only `AddJsonOptions` and expecting minimal APIs to follow.
- Expecting Newtonsoft behaviors (cycles, `dynamic`).
- Assuming minimal APIs do content negotiation.
- Forgetting enum string converter → ints on the wire.

**Cheat sheet**
- `Accept` → output formatter; `Content-Type` → input formatter (415).
- MVC: `AddJsonOptions`; minimal: `ConfigureHttpJsonOptions`.
- Web defaults: camelCase, case-insensitive read.
- Polymorphism: `[JsonPolymorphic]` + `[JsonDerivedType]`.
- Source-gen for AOT/perf.

---

## Topic 13: File upload/download, streaming, large payloads, compression

**Study notes**

Upload models:
- **Buffered** (`IFormFile`, `IFormFileCollection`): the framework reads the whole multipart body; small files in memory, larger ones spooled to a temp file (threshold 64 KB per file by `FormOptions.MemoryBufferThreshold`). Simple; resource-heavy for big files.
- **Streamed** (`MultipartReader` over `Request.Body`): process sections as they arrive; constant memory; you must disable form model binding.
- Raw body streaming (`PUT` with `Request.Body.CopyToAsync(...)`), best for large binary uploads.

Limits: Kestrel `MaxRequestBodySize` default ~30 MB (28.6 MiB); override via `[RequestSizeLimit(n)]`, `[DisableRequestSizeLimit]` or `RequireRequestSizeLimit`-style endpoint metadata; `FormOptions.MultipartBodyLengthLimit` default 128 MB; IIS has its own `maxAllowedContentLength` (~28.6 MB).

Download: `File(...)`/`TypedResults.File(...)`/`PhysicalFile` set `Content-Disposition`, support `enableRangeProcessing: true` (HTTP Range/206 for resume, video seeking), and ETag/Last-Modified for conditional GET. Stream from `FileStream` opened with `FileOptions.Asynchronous | SequentialScan`; never `ReadAllBytes` big files.

Compression: `AddResponseCompression` (Brotli/Gzip providers; HTTPS compression disabled by default due to BREACH/CRIME); typically offload to reverse proxy/CDN. Request decompression: `AddRequestDecompression` + `UseRequestDecompression` (.NET 7+).

### 13.1 [Easy] How do you accept a file upload in a minimal API/controller?
**Answer**
Bind `IFormFile` (or `IFormFileCollection`) from a `multipart/form-data` request. Validate size, extension *and* content (magic bytes), then stream it to storage using `CopyToAsync`.

**Explanation**
Never trust `FileName` or `ContentType` — generate your own stored file name to prevent path traversal (`..\`) and overwrite. In .NET 8 minimal APIs, form binding requires antiforgery validation by default; call `.DisableAntiforgery()` for pure token-based APIs or `UseAntiforgery` + tokens for browser apps.

**Code**
```csharp
app.MapPost("/upload", async (IFormFile file, CancellationToken ct) =>
{
    if (file.Length is 0 or > 5_000_000) return Results.BadRequest("Invalid size");
    var safeName = $"{Guid.NewGuid():N}{Path.GetExtension(file.FileName)}";
    var path = Path.Combine(Path.GetTempPath(), safeName);
    await using var fs = File.Create(path);
    await file.CopyToAsync(fs, ct);
    return Results.Created($"/files/{safeName}", new { safeName });
}).DisableAntiforgery();
```

### 13.2 [Follow-up] How do you serve a large download efficiently, with resume support?
**Answer**
Return a stream/`PhysicalFile` and set `enableRangeProcessing: true` so ASP.NET handles `Range` headers (206 Partial Content) and `If-Range`. Provide `lastModified`/`entityTag` so clients can use conditional requests.

**Explanation**
The file is streamed in chunks by the framework (uses `SendFile` for physical files, avoiding user-space copies where possible). With a `Stream` result, the framework disposes the stream after the response.

**Code**
```csharp
app.MapGet("/videos/{name}", (string name, IWebHostEnvironment env) =>
{
    var safe = Path.GetFileName(name);                       // strips directory parts
    var full = Path.Combine(env.ContentRootPath, "media", safe);
    return File.Exists(full)
        ? Results.File(full, "video/mp4", enableRangeProcessing: true)
        : Results.NotFound();
});
```

### 13.3 [Medium] Implement a streaming upload that doesn't buffer the file in memory.
**Answer**
Disable form binding, read `Request.Body` with `MultipartReader`, iterate sections, and copy each file section's stream straight to the destination. Memory usage stays constant regardless of file size.

**Explanation**
Raise/disable size limits on that endpoint only. Validate the boundary from `Content-Type` (`MediaTypeHeaderValue.Parse(...)`, `HeaderUtilities.RemoveQuotes`). Order of fields matters because the stream is forward-only (send metadata fields before the file).

**Code**
```csharp
using Microsoft.AspNetCore.WebUtilities;
using Microsoft.Net.Http.Headers;

app.MapPost("/upload-stream", async (HttpRequest req, CancellationToken ct) =>
{
    if (!MediaTypeHeaderValue.TryParse(req.ContentType, out var mt) || !mt.MediaType.Equals("multipart/form-data", StringComparison.OrdinalIgnoreCase))
        return Results.StatusCode(StatusCodes.Status415UnsupportedMediaType);
    var boundary = HeaderUtilities.RemoveQuotes(mt.Boundary).Value!;
    var reader = new MultipartReader(boundary, req.Body);
    MultipartSection? section;
    while ((section = await reader.ReadNextSectionAsync(ct)) is not null)
    {
        if (ContentDispositionHeaderValue.TryParse(section.ContentDisposition, out var cd) && cd.IsFileDisposition())
        {
            await using var fs = File.Create(Path.Combine(Path.GetTempPath(), Guid.NewGuid().ToString("N")));
            await section.Body.CopyToAsync(fs, ct);
        }
    }
    return Results.Ok();
}).DisableAntiforgery().WithMetadata(new DisableRequestSizeLimitAttribute());
```

### 13.4 [Hard] How do you protect against upload abuse and what are compression pitfalls?
**Answer**
Enforce size limits at the edge (Kestrel/proxy), whitelist types by content sniffing, scan for malware, store outside web root with generated names, use `Request.Body` timeouts (`MinRequestBodyDataRate`), and use pre-signed URLs to push big uploads straight to blob storage. For compression, avoid compressing secrets-bearing responses over HTTPS with attacker-controlled input (BREACH), and don't double-compress already compressed formats (jpg, zip).

**Explanation**
Zip-bombs are a decompression risk: if you enable request decompression, cap decompressed size. Large payloads should be chunked/resumable (tus protocol, Azure Blob block upload) rather than a single HTTP request. Streaming endpoints should honor `CancellationToken` (`HttpContext.RequestAborted`) so abandoned uploads free resources.

**Code**
```csharp
builder.Services.AddResponseCompression(o =>
{
    o.EnableForHttps = false;                  // default; deliberate
    o.Providers.Add<BrotliCompressionProvider>();
    o.Providers.Add<GzipCompressionProvider>();
    o.MimeTypes = ResponseCompressionDefaults.MimeTypes.Concat(new[] { "application/problem+json" });
});
builder.Services.AddRequestDecompression();
app.UseRequestDecompression();
app.UseResponseCompression();
```

### 13.5 [Tricky] Why does a 100 MB upload fail with 413 even after you set [RequestSizeLimit]? And why can a large `IFormFile` upload exhaust disk?
**Answer**
Limits stack: Kestrel `MaxRequestBodySize`, form options (`MultipartBodyLengthLimit`), IIS `maxAllowedContentLength`, and reverse proxies (nginx `client_max_body_size`) all must permit the size; the smallest wins. Buffered `IFormFile` spools to temp files, so many concurrent big uploads use temp disk.

**Explanation**
For form posts you need both `[RequestSizeLimit]` and `[RequestFormLimits(MultipartBodyLengthLimit = ...)]`. Once the body has begun being read, you can't change `MaxRequestBodySize` (it throws `InvalidOperationException`). Clean up temp files and cap concurrency.

**Code**
```csharp
[HttpPost("big")]
[RequestSizeLimit(200_000_000)]
[RequestFormLimits(MultipartBodyLengthLimit = 200_000_000)]
public async Task<IActionResult> Big(IFormFile file) { await Task.CompletedTask; return Ok(file.Length); }
```

**Interview traps**
- Using the client-supplied file name for storage.
- `ReadAllBytes`/`MemoryStream` for big files.
- Forgetting proxy/IIS limits.
- Enabling HTTPS compression blindly.
- Forgetting antiforgery on form-bound minimal endpoints in .NET 8.

**Cheat sheet**
- Small: `IFormFile`; large: `MultipartReader` / raw stream / pre-signed URL.
- Limits: Kestrel 30 MB, form 128 MB, proxy/IIS separate.
- Downloads: `enableRangeProcessing`, stream, never buffer.
- Sanitize names; store outside web root.

---

## Topic 14: Idempotency keys, retries, ETags & optimistic concurrency

**Study notes**

**Idempotency key** (Stripe-style, IETF draft `Idempotency-Key` header): client generates a unique key per logical operation; server stores key → (request fingerprint, response) and replays the stored response on retry, instead of re-executing. Must handle: in-flight duplicates (lock / 409), same key with different payload (422/409), expiry (e.g. 24h), atomicity (store key and effect in one transaction).

**Retries**: client retries on timeouts/5xx/429 with exponential backoff + jitter (Polly / `Microsoft.Extensions.Http.Resilience`), honoring `Retry-After`. Only safe for idempotent requests or those with keys.

**ETag/conditional requests**: `ETag` identifies a representation version. `If-None-Match` (GET → 304 Not Modified, saves bandwidth), `If-Match` (PUT/PATCH/DELETE → 412 if stale; prevents lost updates). Weak ETags `W/"..."`. 428 Precondition Required when server demands `If-Match`.

**Optimistic concurrency in EF Core**: `[Timestamp] byte[] RowVersion` (SQL Server `rowversion`) or `IsConcurrencyToken()`; `SaveChanges` issues `UPDATE ... WHERE Id=@id AND RowVersion=@orig` and throws `DbUpdateConcurrencyException` if 0 rows affected.

```
Client A GET -> ETag "v5"        Client B GET -> ETag "v5"
A PUT If-Match "v5" -> 204, ETag "v6"
B PUT If-Match "v5" -> 412 Precondition Failed (stale)
```

### 14.1 [Easy] What is idempotency in API design and why do POST endpoints need keys?
**Answer**
Idempotency means repeating a request yields the same effect as one. POST isn't idempotent, so a network retry could create duplicate orders/payments; an `Idempotency-Key` lets the server detect the retry and return the original result.

**Explanation**
The key identifies the *operation*, not the connection. The server persists the key with the response (status+body) and a hash of the request. Retention window should exceed the client's max retry window.

**Code**
```csharp
app.MapPost("/payments", async (PaymentRequest req, HttpContext ctx, IIdempotencyStore store, IPaymentService svc) =>
{
    if (!ctx.Request.Headers.TryGetValue("Idempotency-Key", out var key) || string.IsNullOrWhiteSpace(key))
        return Results.Problem("Idempotency-Key header required", statusCode: 400);
    if (await store.TryGetAsync(key!) is { } saved) return Results.Json(saved.Body, statusCode: saved.Status);
    var result = await svc.ChargeAsync(req);
    await store.SaveAsync(key!, 201, result);
    return Results.Json(result, statusCode: 201);
});
record PaymentRequest(decimal Amount);
record SavedResponse(int Status, object Body);
interface IIdempotencyStore { Task<SavedResponse?> TryGetAsync(string k); Task SaveAsync(string k, int s, object b); }
interface IPaymentService { Task<object> ChargeAsync(PaymentRequest r); }
```

### 14.2 [Follow-up] How do ETags and `If-Match` prevent lost updates?
**Answer**
The server returns an `ETag` with the representation. The client sends it back via `If-Match` on update; the server compares it to the current version and returns 412 if it differs. With EF Core, map the ETag to a `rowversion` concurrency token.

**Explanation**
Return 428 if the header is missing and you require preconditions. ETag value must be quoted. Using the rowversion bytes base64-encoded as the ETag is a common choice.

**Code**
```csharp
public class Product { public int Id { get; set; } public string Name { get; set; } = ""; [Timestamp] public byte[] RowVersion { get; set; } = []; }

app.MapPut("/products/{id:int}", async (int id, ProductDto dto, HttpRequest req, AppDb db) =>
{
    var p = await db.Products.FindAsync(id);
    if (p is null) return Results.NotFound();
    if (!req.Headers.TryGetValue("If-Match", out var im)) return Results.StatusCode(428);
    var tag = $"\"{Convert.ToBase64String(p.RowVersion)}\"";
    if (im != tag) return Results.StatusCode(412);
    p.Name = dto.Name;
    try { await db.SaveChangesAsync(); }
    catch (DbUpdateConcurrencyException) { return Results.StatusCode(412); }
    return Results.NoContent();
});
record ProductDto(string Name);
```

### 14.3 [Medium] Implement conditional GET with 304 Not Modified.
**Answer**
Compute the ETag, compare to `If-None-Match`; if equal return 304 with no body, otherwise return 200 with `ETag` header. Clients and caches then revalidate cheaply.

**Explanation**
304 must not include a body but should repeat `ETag`/`Cache-Control`. `Results.Ok` doesn't do this automatically; static files/`File` results with `entityTag` do. Response caching or `OutputCache` doesn't replace this for dynamic data. A `Last-Modified` with `If-Modified-Since` is the weaker date-based alternative.

**Code**
```csharp
app.MapGet("/products/{id:int}", async (int id, HttpContext ctx, AppDb db) =>
{
    var p = await db.Products.AsNoTracking().FirstOrDefaultAsync(x => x.Id == id);
    if (p is null) return Results.NotFound();
    var etag = $"\"{Convert.ToBase64String(p.RowVersion)}\"";
    ctx.Response.Headers.ETag = etag;
    if (ctx.Request.Headers.IfNoneMatch == etag) return Results.StatusCode(304);
    return Results.Ok(p);
});
```

### 14.4 [Hard] Design a robust idempotency middleware. What are the failure modes?
**Answer**
On request: atomically insert `(key, fingerprint, status=InProgress)`; if the insert fails, load the record: same fingerprint + completed → replay stored response; in progress → 409; different fingerprint → 422. On completion, persist the response. Do it in the same transaction as the business effect (or use an outbox) so a crash can't leave "effect done, key not recorded".

**Explanation**
Failure modes: (1) crash after effect before saving key → duplicate on retry (fix: same DB transaction, unique index on key); (2) crash while InProgress → stuck key (add lease timeout); (3) key reuse across users (scope the key by user/tenant); (4) storing only 2xx vs all — Stripe stores errors too, except 5xx are retried; (5) large responses; (6) distributed caches (Redis `SET NX`) lose data — fine for dedupe windows but not for financial exactness.

**Code**
```csharp
public sealed class IdempotencyRecord
{
    public string Key { get; set; } = "";       // unique index on (UserId, Key)
    public string UserId { get; set; } = "";
    public string RequestHash { get; set; } = "";
    public int? StatusCode { get; set; }
    public string? ResponseBody { get; set; }
    public DateTime CreatedUtc { get; set; }
}
// Insert first; a DbUpdateException on the unique index => duplicate/in-flight path
try { db.Idempotency.Add(rec); await db.SaveChangesAsync(ct); }
catch (DbUpdateException) { /* load existing and decide replay / 409 / 422 */ }
```

### 14.5 [Tricky] Is a retry of PUT always safe? What about optimistic concurrency with retries and `DbUpdateConcurrencyException`?
**Answer**
A PUT retry is state-safe only if the representation is identical; combined with `If-Match`, a retry after a *successful* first attempt returns 412 because the version moved on — the client can't tell if its first write succeeded. Idempotency keys or reading back and comparing resolves the ambiguity.

**Explanation**
On `DbUpdateConcurrencyException` you can: reload and retry (client-wins/store-wins/merge) for non-conflicting changes, or return 409/412 to the client. Don't blindly retry in a loop without reloading the original values. Also, EF's execution strategy (`EnableRetryOnFailure`) retries only transient connection errors, not concurrency conflicts, and wrapped user transactions must be run inside `strategy.ExecuteAsync`.

**Code**
```csharp
try { await db.SaveChangesAsync(ct); }
catch (DbUpdateConcurrencyException ex)
{
    var entry = ex.Entries.Single();
    var dbValues = await entry.GetDatabaseValuesAsync(ct);
    if (dbValues is null) return Results.NotFound();           // deleted by someone else
    entry.OriginalValues.SetValues(dbValues);                   // store-wins refresh; or merge
    return Results.StatusCode(StatusCodes.Status409Conflict);
}
```

**Interview traps**
- Treating ETag as a hash only — it's an opaque version token.
- Key not scoped per user.
- Replaying a cached response without checking the request payload hash.
- Retrying POST blindly on timeout.
- Confusing 409 and 412.

**Cheat sheet**
- Idempotency-Key: store key + hash + response; atomic with the effect.
- `If-None-Match` → 304; `If-Match` → 412/428.
- EF: `[Timestamp]` → `DbUpdateConcurrencyException`.
- Retry only idempotent or keyed; backoff + jitter + `Retry-After`.

---

## Topic 15: Rate limiting middleware

**Study notes**

Built in since .NET 7 (`Microsoft.AspNetCore.RateLimiting`): `builder.Services.AddRateLimiter(...)`, `app.UseRateLimiter()`; apply via `.RequireRateLimiting("policy")` / `[EnableRateLimiting("policy")]` / `[DisableRateLimiting]`. Rejected requests get 503 by default; set `RejectionStatusCode = 429`.

| Limiter | Behaviour | Parameters |
|---|---|---|
| Fixed window | N requests per fixed time window; resets at boundary (burst at edges up to 2N) | `PermitLimit`, `Window`, `QueueLimit` |
| Sliding window | window divided into segments; smoother, no edge burst | `PermitLimit`, `Window`, `SegmentsPerWindow` |
| Token bucket | tokens refill at a rate; allows bursts up to bucket size | `TokenLimit`, `TokensPerPeriod`, `ReplenishmentPeriod` |
| Concurrency | max concurrent in-flight requests | `PermitLimit`, `QueueLimit` |

Partitioning: `PartitionedRateLimiter` / `AddPolicy` with partition key per user/IP/API key (`RateLimitPartition.GetFixedWindowLimiter(key, ...)`). The built-in limiters are **in-process** (per instance) — for a distributed limit across many instances you need Redis/gateway-level limiting. Behind a proxy, use `ForwardedHeaders` to get the real client IP (`RemoteIpAddress`).

### 15.1 [Easy] How do you enable rate limiting in ASP.NET Core 7/8?
**Answer**
Register `AddRateLimiter` with named policies, call `UseRateLimiter()`, and attach the policy to endpoints/controllers. Set `RejectionStatusCode` to 429.

**Explanation**
`UseRateLimiter` must be placed after `UseRouting` if you use endpoint-specific policies (in .NET 7+ minimal hosting, routing is implicit but ordering relative to auth matters—put it before auth if you want to shed load early, after if keyed by user).

**Code**
```csharp
using System.Threading.RateLimiting;
builder.Services.AddRateLimiter(o =>
{
    o.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
    o.AddFixedWindowLimiter("fixed", c => { c.PermitLimit = 10; c.Window = TimeSpan.FromSeconds(10); c.QueueLimit = 0; });
});
var app = builder.Build();
app.UseRateLimiter();
app.MapGet("/ping", () => "pong").RequireRateLimiting("fixed");
```

### 15.2 [Follow-up] Fixed vs sliding window vs token bucket — differences and when to use each?
**Answer**
Fixed window is simplest but allows up to 2× burst at window boundaries. Sliding window smooths this by tracking segments. Token bucket permits controlled bursts while enforcing a steady average rate, ideal for APIs wanting burst tolerance.

**Explanation**
Concurrency limiter protects expensive endpoints (e.g. report generation) irrespective of rate. A queue (`QueueLimit` + `QueueProcessingOrder`) makes requests wait instead of failing immediately — but queued requests tie up connections/threads, so keep small.

**Code**
```csharp
o.AddSlidingWindowLimiter("sliding", c => { c.PermitLimit = 100; c.Window = TimeSpan.FromMinutes(1); c.SegmentsPerWindow = 6; });
o.AddTokenBucketLimiter("bucket", c =>
{
    c.TokenLimit = 20; c.TokensPerPeriod = 5; c.ReplenishmentPeriod = TimeSpan.FromSeconds(1);
    c.AutoReplenishment = true;
});
o.AddConcurrencyLimiter("reports", c => { c.PermitLimit = 2; c.QueueLimit = 5; });
```

### 15.3 [Medium] How do you rate limit per user/IP/API key and return Retry-After?
**Answer**
Use `o.AddPolicy("per-user", ctx => RateLimitPartition.GetFixedWindowLimiter(key, _ => options))` with a partition key from the authenticated user, API key, or client IP. In `OnRejected`, read the `RetryAfter` metadata from the lease and set the header.

**Explanation**
Fallback key for anonymous traffic (IP) with a stricter limit. Beware of memory growth with many partitions: limiters are idle-evicted automatically for `GetXLimiter` partitions. IPv6 clients can rotate within a /64 — consider partitioning by prefix.

**Code**
```csharp
o.AddPolicy("per-user", ctx =>
{
    var key = ctx.User.Identity?.IsAuthenticated == true
        ? ctx.User.Identity.Name!
        : ctx.Connection.RemoteIpAddress?.ToString() ?? "anon";
    return RateLimitPartition.GetTokenBucketLimiter(key, _ => new TokenBucketRateLimiterOptions
    {
        TokenLimit = 30, TokensPerPeriod = 10, ReplenishmentPeriod = TimeSpan.FromSeconds(1), AutoReplenishment = true
    });
});
o.OnRejected = async (ctx, ct) =>
{
    if (ctx.Lease.TryGetMetadata(MetadataName.RetryAfter, out var retry))
        ctx.HttpContext.Response.Headers.RetryAfter = ((int)retry.TotalSeconds).ToString();
    ctx.HttpContext.Response.StatusCode = StatusCodes.Status429TooManyRequests;
    await ctx.HttpContext.Response.WriteAsync("Too many requests", ct);
};
```

### 15.4 [Hard] What are the limits of the built-in limiter in a scaled-out deployment, and what do you do?
**Answer**
State is per process, so with N instances the effective limit is ~N× the configured one and varies by load-balancer routing. For accurate global limits use a shared store (Redis Lua scripts/`INCR`+expiry, or a rate-limiting library) or enforce at the API gateway/CDN (YARP, Azure API Management, Cloudflare).

**Explanation**
Also consider layered defense: gateway coarse limits + app-level per-user limits + concurrency limit on expensive operations. Return `429` with `Retry-After` and optionally `RateLimit-*` headers (IETF draft) so clients can self-throttle. Client side: honor `Retry-After` using Polly `RateLimiter`/retry strategy.

**Code**
```csharp
// Sketch of a Redis-backed fixed window (StackExchange.Redis)
var count = await db.StringIncrementAsync($"rl:{key}:{window}");
if (count == 1) await db.KeyExpireAsync($"rl:{key}:{window}", TimeSpan.FromSeconds(60));
bool allowed = count <= 100;
```

### 15.5 [Tricky] Why do rate limits not seem to apply behind a reverse proxy, and why might static files bypass them?
**Answer**
Behind a proxy `RemoteIpAddress` is the proxy's IP, so all clients share one partition unless `UseForwardedHeaders` is configured with trusted proxies. Rate limiting applies per endpoint metadata; middleware like static files that short-circuit before `UseRateLimiter` aren't limited.

**Explanation**
Don't trust `X-Forwarded-For` from the open internet: set `KnownProxies`/`KnownNetworks`, else clients can spoof IPs to evade limits (or exhaust another IP's quota). Place `UseForwardedHeaders` first.

**Code**
```csharp
builder.Services.Configure<ForwardedHeadersOptions>(o =>
{
    o.ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto;
    o.KnownProxies.Add(System.Net.IPAddress.Parse("10.0.0.4"));
});
app.UseForwardedHeaders();
app.UseRateLimiter();
```

**Interview traps**
- Default rejection is 503, not 429.
- In-memory limits aren't global in a cluster.
- Spoofable `X-Forwarded-For`.
- Large `QueueLimit` hides overload and hoards connections.

**Cheat sheet**
- `AddRateLimiter` + `UseRateLimiter` + `RequireRateLimiting`.
- Fixed (edge bursts), sliding (smooth), token bucket (bursty OK), concurrency (in-flight).
- Partition by user/IP/key.
- Per-instance only; use Redis/gateway for global.

---

## Topic 16: OpenAPI / Swagger

**Study notes**

OpenAPI Specification (OAS 3.x) = machine-readable API contract (JSON/YAML); "Swagger" = the tooling family (Swagger UI) around it. In .NET 8 templates: **Swashbuckle.AspNetCore** (`AddSwaggerGen`, `UseSwagger`, `UseSwaggerUI`); alternative **NSwag** (also generates clients); .NET 9+ adds built-in `Microsoft.AspNetCore.OpenApi` (`AddOpenApi`, `MapOpenApi`) and Swashbuckle was removed from the templates.

Generation reads `ApiExplorer` metadata: routes, parameter sources, `[ProducesResponseType]`, `ActionResult<T>`, `TypedResults`/`Results<>`, `.Produces<T>()`, `.WithOpenApi()` (minimal), XML doc comments (`<GenerateDocumentationFile>true</GenerateDocumentationFile>` + `IncludeXmlComments`). Customize with schema/operation/document filters. Contract-first vs code-first: code-first is typical; contract-first (write YAML, generate stubs/clients) suits public APIs and multi-team work.

### 16.1 [Easy] How do you add Swagger to an ASP.NET Core API?
**Answer**
Add `Swashbuckle.AspNetCore`, call `AddEndpointsApiExplorer()` and `AddSwaggerGen()`, then `UseSwagger()` and `UseSwaggerUI()`, usually only in Development. The JSON is served at `/swagger/v1/swagger.json`.

**Explanation**
`AddEndpointsApiExplorer` is needed so minimal API endpoints show up. Exposing the UI in production is a decision: it reveals your surface area, so protect it or disable it.

**Code**
```csharp
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(c => c.SwaggerDoc("v1", new() { Title = "Shop API", Version = "v1" }));
var app = builder.Build();
if (app.Environment.IsDevelopment()) { app.UseSwagger(); app.UseSwaggerUI(); }
```

### 16.2 [Follow-up] How do you document auth (JWT bearer) and response types?
**Answer**
Add a `SecurityDefinition` (type Http, scheme `bearer`) and a `SecurityRequirement` so Swagger UI offers an Authorize button. Declare responses with `[ProducesResponseType]`/`ProducesProblem` or typed returns so the spec lists them.

**Explanation**
Undocumented error responses are a frequent review complaint. `[ProducesResponseType<T>]` generic attribute is available since .NET 7. XML comments supply summaries; mark nullable/required properly (`required` keyword/`[Required]`) so clients generate correct types.

**Code**
```csharp
builder.Services.AddSwaggerGen(c =>
{
    c.AddSecurityDefinition("Bearer", new Microsoft.OpenApi.Models.OpenApiSecurityScheme
    {
        Type = Microsoft.OpenApi.Models.SecuritySchemeType.Http, Scheme = "bearer", BearerFormat = "JWT"
    });
    c.AddSecurityRequirement(new Microsoft.OpenApi.Models.OpenApiSecurityRequirement
    {
        [new() { Reference = new() { Type = Microsoft.OpenApi.Models.ReferenceType.SecurityScheme, Id = "Bearer" } }] = []
    });
});
```

### 16.3 [Medium] How do you generate typed clients from OpenAPI and keep the contract honest?
**Answer**
Use NSwag, Kiota, or OpenAPI Generator in the build to produce clients from the spec; publish the spec as a build artifact, and gate CI with a breaking-change diff (e.g. `oasdiff`) plus contract tests.

**Explanation**
Hand-written clients drift from the server. Add `operationId`s (`.WithName` in minimal APIs, `[HttpGet(Name=...)]` in controllers) since they become client method names. Swashbuckle CLI (`dotnet swagger tofile`) can emit the spec at build time.

**Code**
```csharp
app.MapGet("/products/{id:int}", (int id) => TypedResults.Ok(new Product { Id = id }))
   .WithName("GetProductById")          // operationId -> client method name
   .WithTags("Products")
   .WithOpenApi(op => { op.Summary = "Get a product"; return op; });
```

### 16.4 [Hard] How do you handle polymorphism, enums, versioned docs and custom schema issues in Swashbuckle?
**Answer**
Enable `UseOneOfForPolymorphism()`/`SelectSubTypesUsing` for inheritance, `DescribeAllEnumsAsStrings`-equivalent via `JsonStringEnumConverter` (Swashbuckle respects STJ config), and register one document per API version with `ApiExplorer` group names. Use `ISchemaFilter`/`IOperationFilter` for custom tweaks (e.g. add a header parameter to all operations).

**Explanation**
Schema ID collisions (same class name in different namespaces) throw at generation — fix with `c.CustomSchemaIds(t => t.FullName)`. Swashbuckle cannot know about runtime-only behavior (custom binders, middleware). Generated spec must match behavior: test the spec endpoint in integration tests.

**Code**
```csharp
public class CorrelationIdHeaderFilter : Swashbuckle.AspNetCore.SwaggerGen.IOperationFilter
{
    public void Apply(Microsoft.OpenApi.Models.OpenApiOperation op, Swashbuckle.AspNetCore.SwaggerGen.OperationFilterContext ctx)
    {
        op.Parameters ??= new List<Microsoft.OpenApi.Models.OpenApiParameter>();
        op.Parameters.Add(new()
        {
            Name = "X-Correlation-Id",
            In = Microsoft.OpenApi.Models.ParameterLocation.Header,
            Required = false
        });
    }
}
// c.OperationFilter<CorrelationIdHeaderFilter>(); c.CustomSchemaIds(t => t.FullName);
```

**Interview traps**
- Swagger ≠ OpenAPI (tooling vs spec).
- Leaving Swagger UI public in production.
- Missing `AddEndpointsApiExplorer` for minimal APIs.
- Spec drift between docs and behavior.

**Cheat sheet**
- OpenAPI = spec; Swagger UI/Swashbuckle/NSwag = tooling; .NET 9 built-in `AddOpenApi`.
- Document responses (`ProducesResponseType`, `TypedResults`).
- Security scheme for bearer.
- Generate clients + breaking-change checks in CI.

---

## Topic 17: Long-running operations & bulk operations

**Study notes**

**Long-running operation (async request-reply)**:

```
POST /reports           -> 202 Accepted
                           Location: /operations/123
                           Retry-After: 5
GET  /operations/123    -> 200 {"status":"running","progress":40}
GET  /operations/123    -> 200 {"status":"succeeded","resultUrl":"/reports/987"}
                           (or 303 See Other -> /reports/987)
DELETE /operations/123  -> cancel (202/204)
```

Work runs outside the request: `BackgroundService` + `Channel<T>`, a queue (Azure Service Bus/RabbitMQ), or a job framework (Hangfire, Quartz). Never run it fire-and-forget inside the controller (lost on restart; scoped services disposed). Persist operation state in DB. Alternatives to polling: **webhooks** (server calls the client's callback URL; sign payload with HMAC, retry with backoff, make receivers idempotent, verify URLs against SSRF), SignalR/SSE/WebSockets push.

**Bulk operations**: single request carrying many items. Decide: atomic (all-or-nothing, single transaction, 400/422 on any failure) vs partial success (per-item result array, HTTP **207 Multi-Status** or 200 with per-item statuses). Enforce max batch size, idempotency (per-item keys), and async processing for large batches. DB side: `AddRange`+one `SaveChanges`, `ExecuteUpdateAsync`/`ExecuteDeleteAsync` (EF Core 7+), `SqlBulkCopy`/EFCore.BulkExtensions, table-valued parameters/`MERGE`.

### 17.1 [Easy] How do you design an endpoint for a task that takes minutes?
**Answer**
Return `202 Accepted` immediately with a `Location` header pointing to a status resource, process the work in the background, and let clients poll the status resource (with `Retry-After`). When finished, the status links to the result.

**Explanation**
202 means "accepted for processing, not completed". The status resource must survive restarts, so store it in a DB/cache. Make the POST idempotent via a key so retries don't start duplicate jobs.

**Code**
```csharp
app.MapPost("/reports", async (ReportRequest req, IOperationStore ops, Channel<Guid> queue) =>
{
    var id = await ops.CreateAsync(req);          // persist as Pending
    await queue.Writer.WriteAsync(id);
    return Results.Accepted($"/operations/{id}", new { id });
});
app.MapGet("/operations/{id:guid}", async (Guid id, IOperationStore ops) =>
    await ops.GetAsync(id) is { } op ? Results.Ok(op) : Results.NotFound());
record ReportRequest(string Kind);
public interface IOperationStore { Task<Guid> CreateAsync(ReportRequest r); Task<object?> GetAsync(Guid id); }
```

### 17.2 [Follow-up] How do you run the background work reliably in-process?
**Answer**
Use a hosted `BackgroundService` consuming a `Channel<T>` (bounded, for backpressure), create a DI scope per job, handle exceptions, and honor the `stoppingToken`. For durability across restarts or multi-instance scale, use a persistent queue or Hangfire.

**Explanation**
In-memory channels lose queued items on crash — fine only when jobs are recoverable from DB state (e.g. a startup sweep re-enqueues `Pending`). Use `Channel.CreateBounded` with `FullMode` to avoid unbounded memory. Scoped services (DbContext) must be created via `IServiceScopeFactory`.

**Code**
```csharp
builder.Services.AddSingleton(Channel.CreateBounded<Guid>(new BoundedChannelOptions(100) { FullMode = BoundedChannelFullMode.Wait }));
builder.Services.AddHostedService<ReportWorker>();

public sealed class ReportWorker(Channel<Guid> queue, IServiceScopeFactory scopes, ILogger<ReportWorker> log) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stop)
    {
        await foreach (var id in queue.Reader.ReadAllAsync(stop))
        {
            try
            {
                using var scope = scopes.CreateScope();
                var svc = scope.ServiceProvider.GetRequiredService<IReportRunner>();
                await svc.RunAsync(id, stop);
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                log.LogError(ex, "Job {Id} failed", id);
            }
        }
    }
}
public interface IReportRunner { Task RunAsync(Guid id, CancellationToken ct); }
```

### 17.3 [Medium] Polling vs webhooks vs push — tradeoffs, and how to make webhooks secure and reliable?
**Answer**
Polling is simple and firewall-friendly but wasteful and delays results; webhooks give timely callbacks but require the client to expose an endpoint and handle retries/duplicates; SignalR/SSE push suits browser UIs. Secure webhooks by HMAC-signing the body with a shared secret and timestamp, verifying with constant-time compare, and retrying with exponential backoff.

**Explanation**
Delivery is at-least-once, so receivers must be idempotent (dedupe by event id). Validate callback URLs on registration (HTTPS only, block private IP ranges → SSRF). Send events via an outbox so DB state and event emission stay consistent. Provide a replay/dead-letter mechanism.

**Code**
```csharp
using System.Security.Cryptography;
static string Sign(string secret, string timestamp, string body)
{
    using var h = new HMACSHA256(Encoding.UTF8.GetBytes(secret));
    return Convert.ToHexString(h.ComputeHash(Encoding.UTF8.GetBytes($"{timestamp}.{body}")));
}
static bool Verify(string secret, string ts, string body, string sig) =>
    CryptographicOperations.FixedTimeEquals(
        Encoding.UTF8.GetBytes(Sign(secret, ts, body)), Encoding.UTF8.GetBytes(sig));
```

### 17.4 [Hard] Design a bulk-create endpoint. Atomic or partial? Which status code?
**Answer**
Choose based on the contract: atomic (one transaction, whole batch fails with 400/422 and per-item errors) for consistency-critical data; partial success returning a per-item result array with `207 Multi-Status` (or 200) for throughput-oriented imports. Always cap batch size and make the operation idempotent.

**Explanation**
Partial-success responses must let the client map results to inputs (client-provided `clientId` or index). For very large batches go asynchronous (202 + operation resource). Database: avoid per-row `SaveChanges`; batch inserts via `AddRange` (EF batches commands into a few round trips), or `SqlBulkCopy` for 10k+ rows, bypassing change tracking and validation. Disable `AutoDetectChangesEnabled` for huge `AddRange`s.

**Code**
```csharp
public record BulkItem(string ClientId, string Name);
public record BulkResult(string ClientId, int Status, int? Id, string? Error);

app.MapPost("/products/bulk", async (List<BulkItem> items, AppDb db, CancellationToken ct) =>
{
    if (items.Count is 0 or > 500) return Results.Problem("Batch must be 1..500", statusCode: 400);
    var results = new List<BulkResult>();
    var entities = new List<(BulkItem Item, Product P)>();
    foreach (var i in items)
    {
        if (string.IsNullOrWhiteSpace(i.Name)) { results.Add(new(i.ClientId, 422, null, "name required")); continue; }
        var p = new Product { Name = i.Name };
        entities.Add((i, p));
        db.Products.Add(p);
    }
    await db.SaveChangesAsync(ct);                 // batched round trips
    results.AddRange(entities.Select(e => new BulkResult(e.Item.ClientId, 201, e.P.Id, null)));
    return Results.Json(results, statusCode: StatusCodes.Status207MultiStatus);
});
```

### 17.5 [Tricky] Bulk update/delete performance: why is loading entities to change them an anti-pattern, and what about SQL parameter limits?
**Answer**
Loading N rows, modifying and saving costs N materializations plus N UPDATE statements; `ExecuteUpdateAsync`/`ExecuteDeleteAsync` issue a single set-based statement without tracking. Large `WHERE IN (...)` lists hit SQL Server's 2100 parameter limit and plan-cache bloat, so chunk or use a table-valued parameter/temp table.

**Explanation**
`ExecuteUpdate`/`ExecuteDelete` bypass the change tracker, interceptors on `SaveChanges`, concurrency tokens and cascade logic in the app (DB cascades still apply), and aren't transactional with a later `SaveChanges` unless wrapped in a transaction. EF Core 8 translates `Contains` on a list to `OPENJSON` on SQL Server, mitigating the parameter limit, but giant lists are still slow.

**Code**
```csharp
var affected = await db.Products
    .Where(p => p.Price < 1m && p.IsActive)
    .ExecuteUpdateAsync(s => s.SetProperty(p => p.IsActive, false), ct);   // single UPDATE
await db.Orders.Where(o => o.CreatedAt < DateTime.UtcNow.AddYears(-7)).ExecuteDeleteAsync(ct);
```

**Interview traps**
- Fire-and-forget `Task.Run` in the controller.
- 200 OK for work that isn't done (should be 202).
- Unbounded bulk batch size.
- Webhooks without signatures/idempotent receivers.
- Forgetting `ExecuteUpdate` skips tracker/interceptors.

**Cheat sheet**
- LRO: 202 + `Location` + status resource + `Retry-After`.
- `BackgroundService` + bounded `Channel<T>`; durable queue for reliability.
- Webhooks: HMAC, retries, idempotent, SSRF checks.
- Bulk: atomic vs partial (207), max size, per-item results.
- `ExecuteUpdate/Delete` for set-based changes.

---

## Topic 18: API styles & architecture: HATEOAS, GraphQL/gRPC, Gateway/BFF, CQRS with MediatR

**Study notes**

**Richardson Maturity Model**: L0 single endpoint/RPC over HTTP (POX), L1 resources (many URIs), L2 HTTP verbs + status codes (where most "REST" APIs stop), L3 hypermedia controls (HATEOAS: responses include links to next possible actions/state transitions, e.g. HAL, JSON:API, Siren).

| | REST | GraphQL | gRPC |
|---|---|---|---|
| Transport | HTTP/1.1–3, JSON | HTTP POST, JSON (one endpoint) | HTTP/2, Protobuf (binary) |
| Contract | OpenAPI | schema (SDL) | .proto |
| Strengths | cacheable, simple, ubiquitous | client picks fields; avoids over/under-fetch; aggregation | fast, streaming (4 modes), strong typing, codegen |
| Weaknesses | over/under-fetching, many round trips | caching harder, N+1 (use DataLoader), query cost/depth limits, complex authZ | no native browser support (needs gRPC-Web), less human-debuggable |
| Best for | public/CRUD APIs | diverse clients/BFF aggregation | internal service-to-service, low latency |

.NET: GraphQL via HotChocolate or GraphQL.NET; gRPC via `Grpc.AspNetCore`.

**API Gateway** (YARP, Ocelot, Azure APIM, Kong): single entry; routing, auth offload, rate limiting, TLS termination, aggregation, caching, observability. **BFF (Backend for Frontend)**: a gateway tailored to one client type (web, mobile), shaping/aggregating data and holding tokens server-side (cookie session for SPAs). Risks: gateway becomes a bottleneck/God service, coupling, logic creep.

**CQRS endpoints with MediatR**: commands (change state) vs queries (read) as request objects handled by `IRequestHandler<TRequest,TResponse>`; pipeline behaviors (`IPipelineBehavior`) for validation/logging/transactions; endpoints become thin: `mediator.Send(...)`.

### 18.1 [Easy] What is the Richardson Maturity Model and what is HATEOAS?
**Answer**
It grades REST-ness from Level 0 (single RPC endpoint) to Level 3 (hypermedia). HATEOAS means responses include links describing available next actions so clients navigate by following links instead of hard-coding URLs.

**Explanation**
Most real APIs are Level 2. HATEOAS gives discoverability and lets the server change URIs/workflow (e.g. include `cancel` link only when the order is cancellable) — but adds payload size, needs link-aware clients, and few teams get the payoff.

**Code**
```csharp
public record Link(string Rel, string Href, string Method);
public record OrderResource(int Id, string Status, IReadOnlyList<Link> Links);

app.MapGet("/orders/{id:int}", (int id, LinkGenerator lg, HttpContext ctx) =>
{
    var links = new List<Link> { new("self", lg.GetPathByName(ctx, "GetOrder", new { id })!, "GET") };
    // Output (shape): {"id":1,"status":"Pending","links":[{"rel":"self","href":"/orders/1","method":"GET"}, ...]}
    links.Add(new("cancel", $"/orders/{id}/cancel", "POST"));      // only when state allows
    return Results.Ok(new OrderResource(id, "Pending", links));
}).WithName("GetOrder");
```

### 18.2 [Follow-up] When would you pick GraphQL or gRPC over REST?
**Answer**
GraphQL when many different clients need flexible shapes/aggregation and over-fetching hurts (mobile + web); gRPC for internal, latency-sensitive service-to-service calls with streaming and strict contracts; REST for public, cacheable, resource-oriented APIs and broad compatibility.

**Explanation**
GraphQL caveats: HTTP caching is lost (POST), need persisted queries, depth/complexity limits, DataLoader to avoid N+1, field-level authorization, and error semantics (200 with `errors`). gRPC caveats: browsers need gRPC-Web, load balancers must support HTTP/2 (L7), harder to inspect. Many systems use gRPC internally and REST/GraphQL at the edge.

**Code**
```csharp
// gRPC service (Grpc.AspNetCore) — generated base from greet.proto
public class GreeterService : Greeter.GreeterBase
{
    public override Task<HelloReply> SayHello(HelloRequest request, ServerCallContext context)
        => Task.FromResult(new HelloReply { Message = $"Hello {request.Name}" });
}
// app.MapGrpcService<GreeterService>();
```

### 18.3 [Medium] What do an API Gateway and a BFF do, and how would you do it in .NET?
**Answer**
A gateway is the single entry point handling cross-cutting concerns (routing, authN, rate limiting, TLS, caching). A BFF is a gateway dedicated to one frontend that aggregates/reshapes backend calls and, for SPAs, keeps tokens server-side (cookie auth) — in .NET commonly built with YARP.

**Explanation**
YARP is a reverse-proxy toolkit configured via `appsettings.json` or code, with load balancing, health checks, transforms. Keep business logic out of the gateway. Avoid one BFF serving all clients — that recreates a monolithic gateway. Use `Duende.BFF` or custom cookie + token handling for OIDC.

**Code**
```csharp
builder.Services.AddReverseProxy().LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));
app.MapReverseProxy();
/* appsettings.json
"ReverseProxy": {
  "Routes": { "orders": { "ClusterId": "ordersCluster", "Match": { "Path": "/api/orders/{**rest}" } } },
  "Clusters": { "ordersCluster": { "Destinations": { "d1": { "Address": "http://orders-svc:8080/" } } } }
} */
```

### 18.4 [Hard] Show CQRS endpoints with MediatR including a pipeline behavior, then argue for and against MediatR.
**Answer**
Each endpoint sends a command/query to MediatR; handlers hold the logic and cross-cutting concerns (validation, transactions, logging) sit in `IPipelineBehavior<,>`. Pros: thin endpoints, uniform pipeline, handler-per-use-case. Cons: indirection, harder navigation ("find usages" through `Send`), runtime DI resolution, hidden coupling, overkill for simple CRUD, and recent MediatR versions moved to commercial licensing.

**Explanation**
Criticisms: it's an in-process mediator — you can get the same with plain classes/handlers injected directly (explicit dependencies, F12 navigable) and decorators (Scrutor) or endpoint filters for cross-cutting. Notifications give implicit fan-out that's hard to reason about and has no durability. Alternatives: direct handler interfaces, Wolverine, source-generated mediators (Mediator by martinothamar), or just services/vertical slices. CQRS itself ≠ MediatR; separate read models (Dapper/projections) for queries is the real CQRS value.

**Code**
```csharp
using MediatR;
public record CreateOrder(string Item) : IRequest<int>;
public record GetOrder(int Id) : IRequest<OrderDto?>;
public record OrderDto(int Id, string Item);

public class CreateOrderHandler(AppDb db) : IRequestHandler<CreateOrder, int>
{
    public async Task<int> Handle(CreateOrder cmd, CancellationToken ct)
    {
        var o = new Order { Item = cmd.Item }; db.Orders.Add(o);
        await db.SaveChangesAsync(ct); return o.Id;
    }
}

public class LoggingBehavior<TReq, TRes>(ILogger<LoggingBehavior<TReq, TRes>> log) : IPipelineBehavior<TReq, TRes>
    where TReq : notnull
{
    public async Task<TRes> Handle(TReq request, RequestHandlerDelegate<TRes> next, CancellationToken ct)
    {
        log.LogInformation("Handling {Request}", typeof(TReq).Name);
        return await next();
    }
}

builder.Services.AddMediatR(c =>
{
    c.RegisterServicesFromAssemblyContaining<Program>();
    c.AddOpenBehavior(typeof(LoggingBehavior<,>));
});
app.MapPost("/orders", async (CreateOrder cmd, ISender s) => Results.Created("/orders", await s.Send(cmd)));
app.MapGet("/orders/{id:int}", async (int id, ISender s) =>
    await s.Send(new GetOrder(id)) is { } o ? Results.Ok(o) : Results.NotFound());
```

### 18.5 [Tricky] Does CQRS with MediatR mean separate databases? And where do transactions/validation behaviors go wrong?
**Answer**
No — CQRS only separates the command and query models; one database is fine, and separate read stores are an optional scaling step bringing eventual consistency. Pipeline behaviors that wrap transactions can go wrong when applied to queries, nest unexpectedly, or run `SaveChanges` twice.

**Explanation**
Constrain behaviors with generic constraints/marker interfaces (e.g. `ICommand<T>`) so a transaction behavior applies only to commands. Behavior order = registration order. Handlers calling other handlers through `Send` creates nested pipelines (double validation/transactions). Return type design: either throw exceptions for failures or return a `Result<T>`; mixing both leads to inconsistent HTTP mapping.

**Code**
```csharp
public interface ICommand<TResponse> : IRequest<TResponse> { }

public class TransactionBehavior<TReq, TRes>(AppDb db) : IPipelineBehavior<TReq, TRes>
    where TReq : ICommand<TRes>
{
    public async Task<TRes> Handle(TReq request, RequestHandlerDelegate<TRes> next, CancellationToken ct)
    {
        await using var tx = await db.Database.BeginTransactionAsync(ct);
        var res = await next();
        await tx.CommitAsync(ct);
        return res;
    }
}
```

**Gotchas / follow-ups**
- Behavior generic constraints silently skip requests that don't match — easy to think validation runs when it doesn't.
- `Publish` notifications run in-process, in the same request, and failures can break the main flow.
- Check the current MediatR license terms before adopting it in commercial projects.

**Interview traps**
- "REST requires HATEOAS" — Level 2 is the pragmatic norm, but strictly Fielding requires hypermedia.
- Equating CQRS with MediatR or with separate databases.
- Gateway doing business logic.
- Choosing GraphQL to "avoid versioning" without addressing caching/cost limits.

**Cheat sheet**
- Richardson: L0 RPC, L1 resources, L2 verbs/status, L3 hypermedia.
- REST public/cacheable; GraphQL flexible aggregation; gRPC internal/fast/streaming.
- Gateway = cross-cutting edge; BFF = per-client; YARP in .NET.
- MediatR = in-process mediator + pipeline behaviors; indirection is the cost.
- CQRS ≠ MediatR ≠ separate DBs.
