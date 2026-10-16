# B02 — .NET Dependency Injection, Configuration, Options, Logging & Observability

**How to use:** Read each topic's *Study notes* first, then work the ladder top to bottom, covering the answer before reading it. Rungs go Easy → Follow-up → Medium → Hard → Tricky; stop where you stop being able to answer and re-read that rung's Explanation.
**Targets:** C# 12 / .NET 8. Snippets assume `ImplicitUsings` and `Nullable` enabled, and the relevant `Microsoft.Extensions.*` packages (ASP.NET Core / Generic Host references them automatically).

## Table of Contents
1. [Topic 1: DI Concept — IoC vs DI vs DIP](#topic-1-di-concept--ioc-vs-di-vs-dip)
2. [Topic 2: Built-in Container Lifetimes](#topic-2-built-in-container-lifetimes)
3. [Topic 3: Captive Dependencies, Scope Validation & Scopes in Background Services](#topic-3-captive-dependencies-scope-validation--scopes-in-background-services)
4. [Topic 4: Constructor vs Property vs Method Injection & Service Locator](#topic-4-constructor-vs-property-vs-method-injection--service-locator)
5. [Topic 5: Registration Patterns — Open Generics, Multiple Impls, TryAdd, Replace, Decorators](#topic-5-registration-patterns--open-generics-multiple-impls-tryadd-replace-decorators)
6. [Topic 6: Keyed Services, Factory Registrations & Func\<T\>](#topic-6-keyed-services-factory-registrations--funct)
7. [Topic 7: IDisposable & Container Disposal Rules](#topic-7-idisposable--container-disposal-rules)
8. [Topic 8: Autofac / Scrutor vs Built-in; DI Without a Framework](#topic-8-autofac--scrutor-vs-built-in-di-without-a-framework)
9. [Topic 9: Configuration System](#topic-9-configuration-system)
10. [Topic 10: Options Pattern & Validation](#topic-10-options-pattern--validation)
11. [Topic 11: Environments & Secrets Best Practices](#topic-11-environments--secrets-best-practices)
12. [Topic 12: Logging Architecture](#topic-12-logging-architecture)
13. [Topic 13: Serilog, Seq/ELK & Correlation IDs](#topic-13-serilog-seqelk--correlation-ids)
14. [Topic 14: OpenTelemetry Tracing & Metrics](#topic-14-opentelemetry-tracing--metrics)
15. [Topic 15: Health Checks (Liveness / Readiness)](#topic-15-health-checks-liveness--readiness)
16. [Topic 16: Feature Management & Localization Basics](#topic-16-feature-management--localization-basics)

---

## Topic 1: DI Concept — IoC vs DI vs DIP

**Study notes**

Three related but different ideas:

| Term | What it is | Level |
|---|---|---|
| **DIP** (Dependency Inversion Principle, the "D" in SOLID) | High-level modules should not depend on low-level modules; both depend on abstractions. Abstractions shouldn't depend on details. | Design principle |
| **IoC** (Inversion of Control) | A framework/caller controls the flow or object creation instead of your code (Hollywood principle: "don't call us, we'll call you"). | Broad architectural idea |
| **DI** (Dependency Injection) | A *technique* implementing IoC for dependencies: an object receives collaborators from outside instead of `new`-ing them. | Pattern |
| **DI container** | A tool that automates DI: registrations (service type → implementation + lifetime) and a resolver that builds object graphs. | Tooling |

```
Without DI:            OrderService --new--> SqlOrderRepository   (compile-time coupling)
With DIP + DI:         OrderService --> IOrderRepository <-- SqlOrderRepository
                                   ^ composition root wires them (Program.cs)
```

- **Composition root**: the single place (Program.cs) where the object graph is assembled. Everything else just declares dependencies.
- Benefits: testability (swap fakes), replaceability, lifetime management, clearer dependency graph.
- The built-in container (`Microsoft.Extensions.DependencyInjection`) supports constructor injection, lifetimes, open generics, keyed services (.NET 8), `IEnumerable<T>`; it deliberately omits property injection, named registrations (pre-.NET 8), interception, child containers, auto-decorators.
- `IServiceCollection` = build-time list of `ServiceDescriptor`s; `IServiceProvider` = built, immutable (after `BuildServiceProvider`) resolver.

### 1.1 [Easy] What is Dependency Injection and why use it?
**Answer**
DI is a technique where a class receives the objects it depends on (usually through its constructor) rather than creating them itself. It decouples classes from concrete implementations, which makes code testable, configurable and easier to change. In .NET a container wires the object graph and manages lifetimes.

**Explanation**
Without DI, `OrderService` calls `new SqlOrderRepository()`, so you cannot unit test it without a database and cannot swap the implementation without editing it. With DI, the class declares `IOrderRepository` in its constructor; the composition root decides which implementation is supplied. Tests pass a fake directly — no container needed.

**Code**
```csharp
public interface IOrderRepository { string Get(int id); }
public sealed class SqlOrderRepository : IOrderRepository { public string Get(int id) => $"SQL order {id}"; }
public sealed class FakeOrderRepository : IOrderRepository { public string Get(int id) => $"Fake order {id}"; }

public sealed class OrderService(IOrderRepository repo) // primary constructor (C# 12)
{
    public string Describe(int id) => repo.Get(id);
}

// Composition root
var services = new ServiceCollection();
services.AddScoped<IOrderRepository, SqlOrderRepository>();
services.AddScoped<OrderService>();
using var sp = services.BuildServiceProvider();
using var scope = sp.CreateScope();
Console.WriteLine(scope.ServiceProvider.GetRequiredService<OrderService>().Describe(1));
// Output: SQL order 1

// Unit test - no container
Console.WriteLine(new OrderService(new FakeOrderRepository()).Describe(1));
// Output: Fake order 1
```

### 1.2 [Follow-up] What is the difference between IoC, DI and DIP?
**Answer**
DIP is a design principle: depend on abstractions, not concretions. IoC is the general idea that something else (framework, container) controls flow/creation. DI is one concrete technique to achieve IoC for dependencies. A container is just a tool that automates DI.

**Explanation**
You can follow DIP without a container (pass interfaces manually), and you can use a container while violating DIP (injecting concrete classes everywhere). IoC also covers things unrelated to DI: event handlers, template-method pattern, ASP.NET Core calling your middleware/controller — the framework controls the flow. A subtle point: DIP is about *ownership* of the abstraction — the interface should be defined by (live next to) the high-level consumer, not the low-level implementation.

**Code**
```csharp
// IoC without DI: framework calls you
app.Use(async (ctx, next) => { await next(); });   // you hand control to the pipeline

// DI without DIP: injecting a concrete class - still coupled to details
public sealed class ReportService(SqlOrderRepository concreteRepo) { }

// DIP + DI: consumer owns the abstraction
public interface IClock { DateTimeOffset UtcNow { get; } }   // lives with the high-level module
public sealed class SystemClock : IClock { public DateTimeOffset UtcNow => DateTimeOffset.UtcNow; }
```

### 1.3 [Medium] What is a composition root and why should the container be referenced only there?
**Answer**
The composition root is the single place near the application entry point where all dependencies are registered and the object graph is composed. Keeping the container reference confined there stops the rest of the code from depending on the container (service locator) and keeps classes plain and testable.

**Explanation**
Everything outside the root should only declare constructor dependencies. If business classes take `IServiceProvider`, they become coupled to the container and their real dependencies are hidden. In ASP.NET Core the root is `Program.cs` (plus extension methods like `AddMyFeature()` that group registrations). Organising registrations into `IServiceCollection` extension methods per module keeps `Program.cs` readable and lets each assembly own its registrations.

**Code**
```csharp
public static class OrderingModule
{
    public static IServiceCollection AddOrdering(this IServiceCollection services)
    {
        services.AddScoped<IOrderRepository, SqlOrderRepository>();
        services.AddScoped<OrderService>();
        return services;
    }
}

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddOrdering();           // composition root stays thin
var app = builder.Build();
app.MapGet("/o/{id:int}", (int id, OrderService s) => s.Describe(id)); // minimal API injects params
app.Run();
```

### 1.4 [Hard] How does the built-in container resolve a type? What happens with multiple constructors and unresolvable parameters?
**Answer**
It finds the type's public constructors and picks the one with the most parameters that it can fully satisfy from registered services (and default values). If two constructors with the same maximum count are both satisfiable but neither's parameter set is a superset of the other, it throws an ambiguity `InvalidOperationException`. If none can be satisfied, it throws saying it can't resolve a service.

**Explanation**
For `ActivatorUtilities`/container activation the rule is: constructors ordered by parameter count descending; the first fully-resolvable is chosen; ambiguity only when another resolvable ctor isn't a subset. Parameters with default values are allowed when the service is not registered. `[ActivatorUtilitiesConstructor]` can disambiguate for `ActivatorUtilities.CreateInstance`. Internally (since .NET Core 3/5) the container compiles resolution into expression trees / IL-emitted delegates after a few calls, and uses a call-site tree (`ServiceCallSite`) per service type, cached in `ServiceProviderEngine`.

**Code**
```csharp
public interface IA { } public interface IB { }
public sealed class A : IA { }
public sealed class Multi
{
    public string Used { get; }
    public Multi(IA a) { Used = "ctor(IA)"; }
    public Multi(IA a, IB b) { Used = "ctor(IA,IB)"; }   // chosen only if IB is registered
}

var s = new ServiceCollection().AddSingleton<IA, A>().AddTransient<Multi>().BuildServiceProvider();
Console.WriteLine(s.GetRequiredService<Multi>().Used);
// Output: ctor(IA)        (IB not registered, so the larger ctor is skipped)
```

**Gotchas / follow-ups**
- Registering `IB` later flips the chosen constructor silently — a source of "works on my machine" differences.
- Non-public constructors are ignored by the built-in container.
- Primitive parameters (string, int) without defaults cannot be resolved — use options or factories.

### 1.5 [Tricky] "Singleton registered with an instance" vs a factory vs a type: what differs in behaviour?
**Answer**
`AddSingleton<T>(instance)` registers an existing object that the container will *not* dispose; `AddSingleton<T, TImpl>()` makes the container construct (and dispose, if disposable) the object lazily on first resolve; a factory lambda is also container-owned and disposed but lets you run custom construction code.

**Explanation**
Ownership drives disposal: the container disposes only what it created. Instance registrations are created eagerly by you (so startup cost and exceptions occur at registration time) and you remain responsible for disposing them. Types/factories are lazy by default — a bad singleton throws on first request, not at startup (unless `ValidateOnBuild` surfaces missing dependencies, which still won't run constructors).

**Code**
```csharp
public sealed class Res : IDisposable
{
    public string Name { get; }
    public Res(string name) => Name = name;
    public void Dispose() => Console.WriteLine($"Disposed {Name}");
}

var services = new ServiceCollection();
var mine = new Res("instance");
services.AddSingleton(mine);                                  // not disposed by container
services.AddSingleton<Res>(_ => new Res("factory"));          // last wins for Res! (see Topic 5)
using (var sp = services.BuildServiceProvider())
{
    _ = sp.GetRequiredService<Res>();
}
// Output: Disposed factory   ("instance" never disposed - last registration won and it is container-owned)
mine.Dispose();
// Output: Disposed instance
```

**Interview traps**
- Saying "IoC and DI are the same thing" — DI is a *form* of IoC.
- Claiming DIP means "use interfaces everywhere" — it is about who owns the abstraction and direction of dependency.
- Injecting `IServiceProvider` into business classes "because it's DI" — that's the service locator.
- Believing the built-in container supports property injection — it does not.

**Cheat sheet**
- DIP = principle; IoC = idea; DI = technique; container = tool.
- Composition root = one place that builds the graph (Program.cs).
- Container picks the greediest resolvable public constructor.
- Container disposes only what it created; instance registrations are yours to dispose.

---

## Topic 2: Built-in Container Lifetimes

**Study notes**

| Lifetime | Instance created | Shared across | Typical use |
|---|---|---|---|
| **Transient** | Every resolve | Nobody | Lightweight, stateless services |
| **Scoped** | Once per scope (per HTTP request in ASP.NET Core) | Everything inside the scope | `DbContext`, unit-of-work, per-request state |
| **Singleton** | Once per root container (lazily, unless instance supplied) | The whole app | Caches, config, `HttpClient` factories, thread-safe stateless services |

```
Root provider ── Singleton (1 instance)
   ├─ Scope A (request 1) ── Scoped (1 instance) ── Transient (new each resolve)
   └─ Scope B (request 2) ── Scoped (1 instance) ── Transient (new each resolve)
```

- ASP.NET Core creates one scope per request (`RequestServices`). Generic Host has *no* automatic scope for `BackgroundService`/hosted services.
- Singletons **must be thread-safe**.
- A service may depend on services with the **same or longer** lifetime only (Transient → anything; Scoped → Scoped/Singleton/Transient; Singleton → Singleton or Transient (but see captive)).
- Scope objects: `IServiceScope` from `CreateScope()` / `CreateAsyncScope()` — dispose it to dispose scoped/transient services it created.

### 2.1 [Easy] Explain Transient, Scoped and Singleton.
**Answer**
Transient creates a new instance on every request to the container; Scoped creates one per scope (one per HTTP request in ASP.NET Core); Singleton creates one for the whole application lifetime. Choose the shortest lifetime that works, and make singletons thread-safe.

**Explanation**
Think of the three as "per resolve", "per scope", "per container". `DbContext` is registered scoped by `AddDbContext` because it is not thread-safe and tracks per-unit-of-work state. A stateless helper can be transient or singleton; singleton is cheaper but demands thread safety.

**Code**
```csharp
public sealed class Probe { public Guid Id { get; } = Guid.NewGuid(); }

var services = new ServiceCollection();
services.AddTransient<Probe>();
using var sp = services.BuildServiceProvider();
Console.WriteLine(sp.GetRequiredService<Probe>().Id == sp.GetRequiredService<Probe>().Id);
// Output: False  (transient: new each time)
```

### 2.2 [Follow-up] Write a demo showing the three lifetimes across two scopes.
**Answer**
Resolve each lifetime twice per scope and in two scopes, and compare instance ids. Transient differs every time, scoped matches within a scope but differs across scopes, singleton is identical everywhere.

**Explanation**
This is the canonical interview demo. Use marker classes with a `Guid` so equality is observable. Note `ReferenceEquals` on resolved instances is the cleanest check.

**Code**
```csharp
public sealed class T1 { } public sealed class S1 { } public sealed class G1 { }

var sp = new ServiceCollection()
    .AddTransient<T1>().AddScoped<S1>().AddSingleton<G1>()
    .BuildServiceProvider();

using var a = sp.CreateScope();
using var b = sp.CreateScope();
var pa = a.ServiceProvider; var pb = b.ServiceProvider;

Console.WriteLine($"Transient same in scope? {ReferenceEquals(pa.GetService<T1>(), pa.GetService<T1>())}");
Console.WriteLine($"Scoped same in scope?    {ReferenceEquals(pa.GetService<S1>(), pa.GetService<S1>())}");
Console.WriteLine($"Scoped same across?      {ReferenceEquals(pa.GetService<S1>(), pb.GetService<S1>())}");
Console.WriteLine($"Singleton same across?   {ReferenceEquals(pa.GetService<G1>(), pb.GetService<G1>())}");
// Output:
// Transient same in scope? False
// Scoped same in scope?    True
// Scoped same across?      False
// Singleton same across?   True
```

### 2.3 [Medium] How many instances exist when a scoped service is injected into two transients in the same request?
**Answer**
One. Within a scope the scoped service is cached, so both transients receive the same scoped instance, while the two transients themselves are separate objects. This is why a scoped `DbContext` is shared by all repositories in a request.

**Explanation**
Lifetime applies to the *service being resolved*, not the consumer. Each transient is new, but its scoped dependency is looked up in the scope's resolved-services cache. This sharing is intentional: a unit of work spanning several repositories uses one `DbContext` and one transaction.

**Code**
```csharp
public sealed class Ctx { public Guid Id { get; } = Guid.NewGuid(); }
public sealed class RepoA(Ctx ctx) { public Ctx Ctx { get; } = ctx; }
public sealed class RepoB(Ctx ctx) { public Ctx Ctx { get; } = ctx; }

var sp = new ServiceCollection().AddScoped<Ctx>().AddTransient<RepoA>().AddTransient<RepoB>().BuildServiceProvider();
using var scope = sp.CreateScope();
var a = scope.ServiceProvider.GetRequiredService<RepoA>();
var b = scope.ServiceProvider.GetRequiredService<RepoB>();
Console.WriteLine(ReferenceEquals(a.Ctx, b.Ctx));
// Output: True
```

### 2.4 [Hard] What happens if you resolve a scoped service from the root provider? How does `ValidateScopes` change it?
**Answer**
Without validation, a scoped service resolved from the root provider is cached in the root scope and effectively lives as a singleton — dangerous for things like `DbContext`. With `ValidateScopes = true` it throws `InvalidOperationException: Cannot resolve scoped service ... from root provider`. ASP.NET Core enables validation by default in the Development environment only.

**Explanation**
The root provider is itself the "root scope". Scoped services resolved there are tracked until the root disposes. `ValidateScopes` makes the container track which scope each resolve happens in, and also catches singletons that consume scoped services (see Topic 3). It costs a little performance, so production defaults to off — which is exactly why bugs found only in prod occur. You can force it on with `UseDefaultServiceProvider`.

**Code**
```csharp
public sealed class Db { }

var services = new ServiceCollection().AddScoped<Db>();

var lax = services.BuildServiceProvider();                         // ValidateScopes = false
Console.WriteLine(ReferenceEquals(lax.GetRequiredService<Db>(), lax.GetRequiredService<Db>()));
// Output: True   (silently behaves like a singleton)

var strict = services.BuildServiceProvider(new ServiceProviderOptions { ValidateScopes = true });
try { strict.GetRequiredService<Db>(); }
catch (InvalidOperationException ex) { Console.WriteLine(ex.Message.Contains("root provider")); }
// Output: True

// In a web app, force validation in all environments:
// builder.Host.UseDefaultServiceProvider(o => { o.ValidateScopes = true; o.ValidateOnBuild = true; });
```

**Gotchas / follow-ups**
- `ValidateOnBuild` checks that every registered service's dependencies *can be constructed*, at build time — it does not run constructors and cannot see runtime `GetService` calls or factory lambdas' internals.
- Also enabled by default in Development by the host builders.

### 2.5 [Tricky] A transient `IDisposable` is resolved many times from the root provider in a long-running app. What goes wrong?
**Answer**
The container tracks every disposable instance it creates and disposes them only when its owning scope/root is disposed. Resolving transient disposables from the root therefore keeps every instance alive until app shutdown — a memory leak. Resolve them inside short-lived scopes (or don't register them as container-created disposables).

**Explanation**
Tracking is what lets the container call `Dispose` for you. In a request scope, instances are released when the request scope ends, so it's fine. From the root, or from a singleton that repeatedly calls `GetRequiredService<T>()` on the root provider, the tracked list grows unboundedly. Fixes: create a scope per unit of work, or use a factory type you own (e.g. `IDbContextFactory<T>`) where the *caller* creates and disposes instances. Note factory-lambda registrations are still tracked; only instances you pass in are untracked.

**Code**
```csharp
public sealed class Conn : IDisposable { public static int Live; public Conn() => Live++; public void Dispose() => Live--; }

var root = new ServiceCollection().AddTransient<Conn>().BuildServiceProvider();
for (int i = 0; i < 1000; i++) _ = root.GetRequiredService<Conn>();
Console.WriteLine(Conn.Live);            // Output: 1000  (all still tracked by root)

for (int i = 0; i < 1000; i++) { using var s = root.CreateScope(); _ = s.ServiceProvider.GetRequiredService<Conn>(); }
Console.WriteLine(Conn.Live);            // Output: 1000 (the earlier 1000 remain; the scoped ones were disposed -> still 1000)
root.Dispose();
Console.WriteLine(Conn.Live);            // Output: 0
```

**Interview traps**
- "Scoped = per thread" — wrong; it's per scope object.
- Assuming a hosted service gets a request scope — it doesn't.
- Saying "singleton is always best for performance" — thread-safety and captive dependency risks.
- Forgetting that `ValidateScopes` is Development-only by default.

**Cheat sheet**
- Transient = per resolve; Scoped = per scope (request); Singleton = per root.
- Scoped from root ⇒ silent singleton unless `ValidateScopes`.
- Container tracks & disposes the disposables it creates (at scope/root end).
- Same-scope consumers share the scoped instance regardless of their own lifetime.
- `ValidateOnBuild` = missing-dependency check at startup.

---

## Topic 3: Captive Dependencies, Scope Validation & Scopes in Background Services

**Study notes**

**Captive dependency:** a longer-lived service holds a reference to a shorter-lived one, "capturing" it and extending its life.

```
Singleton Cache ──holds──> Scoped DbContext   (captured!  lives forever, shared across threads/requests)
```

Consequences: stale data in a `DbContext` change tracker, thread-safety violations (`A second operation was started on this context...`), disposed-object errors, memory growth.

Rules:
- Singleton may depend on: Singleton (and Transient, which then effectively becomes captive too).
- Scoped may depend on: Scoped, Singleton, Transient.
- Detect via `ValidateScopes` (throws "Cannot consume scoped service 'X' from singleton 'Y'").

**Using scoped services from long-lived code** (`BackgroundService`, `IHostedService`, singletons): inject `IServiceScopeFactory` (a singleton) and create a scope per unit of work. `IServiceProvider` injected into a singleton is the *root* provider, so `CreateScope()` on it also works, but `IServiceScopeFactory` states intent better.

```
BackgroundService (singleton)
  loop: using var scope = factory.CreateAsyncScope();   // new scope per iteration
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        ... work ...                                      // scope disposed => DbContext disposed
```

### 3.1 [Easy] What is a captive dependency?
**Answer**
It's when a longer-lived service (e.g. singleton) captures a shorter-lived one (scoped or transient) through its constructor, so the short-lived instance lives as long as the singleton. The classic case is a singleton holding a scoped `DbContext`.

**Explanation**
The container resolves the singleton once, passing in whatever scoped instance it can create at that time. That instance is never released or replaced, so it behaves like a singleton and is shared by all requests/threads, which breaks lifetime assumptions of the captured type.

**Code**
```csharp
public sealed class Db { }
public sealed class Cache(Db db) { public Db Db { get; } = db; }   // captive: Db is scoped

var sp = new ServiceCollection().AddScoped<Db>().AddSingleton<Cache>()
    .BuildServiceProvider(new ServiceProviderOptions { ValidateScopes = true, ValidateOnBuild = true });
// Throws InvalidOperationException (Cannot consume scoped service 'Db' from singleton 'Cache')
```

### 3.2 [Follow-up] How do you detect it, and how do you fix it?
**Answer**
Turn on `ValidateScopes` and `ValidateOnBuild` (default only in Development) so the container throws. Fix by shortening the consumer's lifetime to scoped, or by having the singleton take `IServiceScopeFactory` and create a scope for each operation.

**Explanation**
Prefer redesigning lifetimes first: if a service needs per-request data it probably should be scoped. If it truly must be a singleton (e.g. a cache refresher), pass the *scope factory* and resolve the scoped dependency inside a short scope. Alternatively pass the data the singleton needs as method arguments rather than depending on scoped services.

**Code**
```csharp
public sealed class Cache(IServiceScopeFactory factory)
{
    public int Load()
    {
        using var scope = factory.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<Db>();   // scoped, lives for this call only
        return db.GetHashCode();
    }
}

var sp = new ServiceCollection().AddScoped<Db>().AddSingleton<Cache>()
    .BuildServiceProvider(new ServiceProviderOptions { ValidateScopes = true });
Console.WriteLine(sp.GetRequiredService<Cache>().Load() != 0);
// Output: True
```

### 3.3 [Medium] How do you use a scoped `DbContext` inside a `BackgroundService`?
**Answer**
Inject `IServiceScopeFactory`, create a scope (preferably `CreateAsyncScope`) per iteration/unit of work, resolve the `DbContext` from that scope's provider, and let the scope dispose it. Don't inject the `DbContext` into the background service's constructor.

**Explanation**
Hosted services are registered as singletons by `AddHostedService`, so a scoped constructor dependency would be captive (and fail validation). A fresh scope per iteration also means a fresh change tracker, no stale entities, and bounded memory. Alternative: `IDbContextFactory<T>` (`AddDbContextFactory`) which gives short-lived contexts directly.

**Code**
```csharp
public sealed class Worker(IServiceScopeFactory scopeFactory, ILogger<Worker> log) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromSeconds(30));
        while (await timer.WaitForNextTickAsync(stoppingToken))
        {
            await using var scope = scopeFactory.CreateAsyncScope();
            var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
            var pending = await db.Orders.CountAsync(o => !o.Processed, stoppingToken);
            log.LogInformation("Pending orders: {Count}", pending);
        }
    }
}
// builder.Services.AddDbContext<AppDbContext>(...); builder.Services.AddHostedService<Worker>();
```

### 3.4 [Hard] `IServiceProvider` injected into a singleton: is it the root or the scoped provider? What if injected into a scoped service?
**Answer**
`IServiceProvider` is resolved from whichever scope is resolving the consumer. Injected into a singleton it is the root provider; injected into a scoped/transient service created in a request it is that request's scoped provider. Calling `GetService<ScopedThing>()` on the root provider yields the captive/validation problem; on the scoped provider it's correct.

**Explanation**
This is why "service locator in a singleton" is doubly wrong: it hides dependencies *and* resolves from the root. It also explains why `HttpContext.RequestServices` gives the request scope while `app.Services` is root. Note that `IServiceScopeFactory.CreateScope()` always creates a child of the root, regardless of where the factory is injected.

**Code**
```csharp
public sealed class Probe(IServiceProvider sp) { public IServiceProvider Sp { get; } = sp; }

var root = new ServiceCollection().AddSingleton<Probe>().AddScoped<Probe, Probe>().BuildServiceProvider();
// last registration (scoped) wins:
using var scope = root.CreateScope();
var fromScope = scope.ServiceProvider.GetRequiredService<Probe>();
Console.WriteLine(ReferenceEquals(fromScope.Sp, scope.ServiceProvider));
// Output: True   (scoped consumer got the scoped provider)
```

**Gotchas / follow-ups**
- Don't dispose a scope while an async operation using its services is still running (e.g. fire-and-forget after request ends) — you'll get `ObjectDisposedException`.
- `Task.Run(() => scopedService.Do())` from a controller escapes the request; create a new scope inside the task instead.

### 3.5 [Tricky] A singleton depends on a *transient* that is stateful. Is that a captive dependency, and does validation flag it?
**Answer**
Functionally yes — the transient is created once and then lives as long as the singleton, so it is effectively a singleton and may not be thread-safe. `ValidateScopes` does **not** flag it, because transient-into-singleton isn't a scope violation; only scoped-into-singleton is.

**Explanation**
The validator only checks scoped services. Disposable transients injected into singletons are tracked by the root and disposed at shutdown. Guard by code review and design: if the transient must be fresh per use, inject a factory (`Func<T>` registration or `IServiceScopeFactory`) and create instances on demand.

**Code**
```csharp
public sealed class Counter { public int N; }
public sealed class Singleton(Counter c) { public Counter C { get; } = c; }

var sp = new ServiceCollection().AddTransient<Counter>().AddSingleton<Singleton>()
    .BuildServiceProvider(new ServiceProviderOptions { ValidateScopes = true, ValidateOnBuild = true });
var s1 = sp.GetRequiredService<Singleton>();
var fresh = sp.GetRequiredService<Counter>();
Console.WriteLine(ReferenceEquals(s1.C, fresh));
// Output: False  (no exception thrown, but s1.C is stuck forever)
```

**Interview traps**
- "A singleton can inject anything" — only safe with singleton/stateless-transient deps.
- Using `IServiceProvider.GetService<DbContext>()` in a singleton and assuming it's scoped.
- Fire-and-forget tasks using request-scoped services after the response ends.
- Thinking `ValidateScopes` flags *all* lifetime mismatches.

**Cheat sheet**
- Captive = long-lived holds short-lived.
- Detect: `ValidateScopes` + `ValidateOnBuild`; fix: change lifetime or use `IServiceScopeFactory`.
- Background services: one scope per work iteration (`CreateAsyncScope`).
- `IDbContextFactory<T>` is the other clean option for singletons/Blazor.
- Validation ignores transient-in-singleton.

---

## Topic 4: Constructor vs Property vs Method Injection & Service Locator

**Study notes**

| Style | How | Built-in container | When |
|---|---|---|---|
| **Constructor** | Dependencies are ctor parameters | Yes (the only automatic style) | Required dependencies — default choice |
| **Property / setter** | Public settable property filled after construction | **No** (Autofac etc. do) | Optional deps; rare; hides requirements |
| **Method** | Dependency passed as a method argument | N/A (you pass it) — but ASP.NET Core binds action/endpoint params | Per-call data/dependency; `Invoke(HttpContext, IService)` in middleware |

- ASP.NET Core extras: `[FromServices]` on action params, minimal-API parameters inferred from DI, middleware `InvokeAsync(HttpContext ctx, IScopedService s)` (scoped deps go on `Invoke`, because middleware itself is constructed once like a singleton), `[FromKeyedServices]` (.NET 8).
- **Service locator** = asking a global/ambient container for dependencies inside your class (`sp.GetService<T>()`). It hides dependencies, causes runtime failures instead of compile/startup failures, and complicates testing.
- Acceptable locator-like uses: composition root, factories, framework infrastructure (middleware, filters), creating scopes in background work.
- Too many ctor params (>4-5) is a **smell** (SRP violation), not a reason to switch to a locator.

### 4.1 [Easy] Which injection types exist and which does the built-in container support?
**Answer**
Constructor, property (setter) and method injection exist. The built-in container supports only constructor injection automatically; ASP.NET Core additionally binds action/endpoint parameters. Constructor injection is preferred for required dependencies because it makes them explicit and guarantees a valid object.

**Explanation**
Constructor injection makes an object immutable-after-construction and impossible to create half-initialised. Property injection allows invalid states (null dependency) and hides coupling. Method injection fits dependencies/data that vary per call.

**Code**
```csharp
public interface IClock { DateTime Now { get; } }
public sealed class Clock : IClock { public DateTime Now => DateTime.UtcNow; }

public sealed class Greeter(IClock clock)                // constructor injection
{
    public string Hello() => $"Hello at {clock.Now:HH}h";
}

// method injection in a minimal API: parameter resolved from DI per request
// app.MapGet("/hi", (IClock clock) => clock.Now);
```

### 4.2 [Follow-up] What is the service locator anti-pattern and why is it bad?
**Answer**
It's pulling dependencies from the container inside a class (`_sp.GetRequiredService<T>()`) instead of declaring them. It hides what the class needs, turns missing registrations into runtime errors, makes unit tests need a container, and ties the class to the container.

**Explanation**
A reader can't tell a class's dependencies from its constructor; the DI graph can't be validated at startup; tests must build a service provider or mock `IServiceProvider`. The container type becomes a hidden global. Legitimate exceptions exist (see notes) where creating objects dynamically *is* the class's job.

**Code**
```csharp
// Anti-pattern
public sealed class BadService(IServiceProvider sp)
{
    public void Run() => sp.GetRequiredService<IEmailSender>().Send("x");   // hidden dependency
}

// Better
public interface IEmailSender { void Send(string to); }
public sealed class GoodService(IEmailSender sender)
{
    public void Run() => sender.Send("x");
}
```

### 4.3 [Medium] In middleware, why are scoped services injected into `InvokeAsync` and not the constructor?
**Answer**
Conventional middleware is instantiated once at pipeline construction, so it is effectively a singleton. A scoped service in its constructor would be captive (and fail scope validation). `InvokeAsync` parameters are resolved from the request's scoped provider on each request.

**Explanation**
The framework reflects over `Invoke`/`InvokeAsync`: first parameter must be `HttpContext`; additional parameters are resolved per call from `HttpContext.RequestServices`. Alternatively implement `IMiddleware` (factory-activated per request, registered in DI, can take scoped ctor deps).

**Code**
```csharp
public sealed class TenantMiddleware(RequestDelegate next, ILogger<TenantMiddleware> log) // singleton-ish deps only
{
    public async Task InvokeAsync(HttpContext ctx, ITenantAccessor tenant)   // scoped OK here
    {
        log.LogInformation("Tenant {Tenant}", tenant.Id);
        await next(ctx);
    }
}
public interface ITenantAccessor { string Id { get; } }
// app.UseMiddleware<TenantMiddleware>();
```

### 4.4 [Hard] Your class has 9 constructor parameters. What do you do — and when is a locator-ish approach legitimate?
**Answer**
Treat it as a design smell: split responsibilities, introduce a facade/aggregate service or mediator for cohesive groups of dependencies. A locator-style call is legitimate only where the object's purpose is dynamic creation — factories, composition root, framework plumbing, or creating scopes for background work.

**Explanation**
Refactor options: group related deps into a parameter-object/facade (`IOrderPricing` wrapping tax+discount+currency), move behaviour into smaller services, or use pipeline/handlers (MediatR-style). Hiding the same dependencies behind `IServiceProvider` just makes the problem invisible. For strategy selection, use `IEnumerable<T>` or keyed services instead of resolving by hand.

**Code**
```csharp
public interface IOrderPricing { decimal Price(int orderId); }
public sealed class OrderPricing(ITax tax, IDiscounts disc, ICurrency cur) : IOrderPricing
{
    public decimal Price(int orderId) => cur.Convert(tax.Apply(disc.Apply(100m)));
}
public interface ITax { decimal Apply(decimal x); }
public interface IDiscounts { decimal Apply(decimal x); }
public interface ICurrency { decimal Convert(decimal x); }

public sealed class CheckoutService(IOrderPricing pricing, IOrderRepository repo) { } // 2 deps instead of 9
```

### 4.5 [Tricky] How do you emulate property injection with the built-in container, and what are the risks?
**Answer**
Either use a factory lambda that sets properties after resolving constructor deps, or switch to a container like Autofac that supports property injection. The risk is half-initialised objects and hidden coupling, so use only for optional dependencies.

**Explanation**
The built-in container never sets properties. With `ActivatorUtilities.CreateInstance` you still only get constructor injection. Use `required` members (C# 11) to force initialisation at construction in your own code — but the container does not populate them either, so you'd need a factory.

**Code**
```csharp
public sealed class Reporter
{
    public ILogger<Reporter>? Log { get; set; }       // optional
}

var services = new ServiceCollection().AddLogging();
services.AddTransient(sp => new Reporter { Log = sp.GetRequiredService<ILogger<Reporter>>() });
using var provider = services.BuildServiceProvider();
Console.WriteLine(provider.GetRequiredService<Reporter>().Log is not null);
// Output: True
```

**Gotchas / follow-ups**
- Circular dependencies (A needs B needs A) can't be solved by constructor injection; the container throws `InvalidOperationException: A circular dependency was detected`. Break the cycle via redesign or lazy resolution (`Lazy<T>`-style factory), not property injection hacks.

**Interview traps**
- "`IServiceProvider` in the ctor is fine, it's still DI."
- Adding scoped ctor dependency to conventional middleware.
- Believing `[FromServices]` works in any class — only on action/handler parameters.
- "More ctor params = switch to property injection" — fix the design instead.

**Cheat sheet**
- Constructor injection = default; built-in container does nothing else automatically.
- Service locator hides deps; allowed only in factories/roots/infrastructure.
- Middleware: singleton deps in ctor, scoped deps in `InvokeAsync`.
- 5+ ctor params ⇒ SRP smell.
- Circular dependency ⇒ redesign.

---

## Topic 5: Registration Patterns — Open Generics, Multiple Impls, TryAdd, Replace, Decorators

**Study notes**

- **Last registration wins** for `GetService<T>()`; **all** registrations are returned (in registration order) by `GetServices<T>()` / `IEnumerable<T>` ctor parameter.
- **Open generics:** `services.AddScoped(typeof(IRepository<>), typeof(Repository<>))` — closed types are constructed on demand. Generic constraints are honoured (a type that doesn't satisfy the constraint can't be resolved).
- **TryAdd\*** (namespace `Microsoft.Extensions.DependencyInjection.Extensions`): `TryAddSingleton<T,TImpl>()` registers only if no registration for the service type exists. Library authors use it so apps can override defaults.
- **TryAddEnumerable(ServiceDescriptor.Singleton<IFoo, FooA>())**: adds only if that *(service, implementation)* pair isn't already there — idempotent multi-registration.
- **Replace(ServiceDescriptor)**: removes the *first* existing registration of the service type and adds the new one. **RemoveAll<T>()** removes all.
- **Decorators:** no built-in `Decorate`. Options: manual factory registration, or **Scrutor**'s `services.Decorate<IFoo, FooDecorator>()`.

```
services:  IPlugin -> A, IPlugin -> B, IPlugin -> C
GetService<IPlugin>()          => C          (last wins)
GetServices<IPlugin>()         => [A, B, C]  (registration order)
```

### 5.1 [Easy] What happens when you register the same interface twice and resolve it?
**Answer**
The last registration wins for single resolution; asking for `IEnumerable<T>` returns every registration in registration order. Earlier registrations aren't removed — they are just shadowed.

**Explanation**
This lets plug-in style registrations (validators, handlers, health checks) accumulate, while `Add*` after a library's defaults overrides them. Order of calls to `Add*` therefore matters.

**Code**
```csharp
public interface INotifier { string Name { get; } }
public sealed class Email : INotifier { public string Name => "Email"; }
public sealed class Sms : INotifier { public string Name => "Sms"; }

var sp = new ServiceCollection()
    .AddSingleton<INotifier, Email>()
    .AddSingleton<INotifier, Sms>()
    .BuildServiceProvider();

Console.WriteLine(sp.GetRequiredService<INotifier>().Name);
Console.WriteLine(string.Join(",", sp.GetServices<INotifier>().Select(n => n.Name)));
// Output:
// Sms
// Email,Sms
```

### 5.2 [Follow-up] How do you register and use open generics?
**Answer**
Register using `typeof` with unbound generic types: `AddScoped(typeof(IRepository<>), typeof(Repository<>))`. Any `IRepository<T>` request then constructs `Repository<T>`. Specific closed registrations (e.g. `IRepository<Order>`) take priority for that type only if registered *after*, since last wins — or are matched exactly first.

**Explanation**
The container resolves the closed type by closing the implementation generic over the same type arguments. Constraints must be satisfiable, otherwise resolution fails. Closed registrations are matched exactly before falling back to the open one for that closed service type, so you can specialise one entity while others use the generic.

**Code**
```csharp
public interface IRepository<T> { string Describe(); }
public class Repository<T> : IRepository<T> { public string Describe() => $"Generic repo of {typeof(T).Name}"; }
public sealed class Order { }
public sealed class OrderRepository : IRepository<Order> { public string Describe() => "Special order repo"; }

var sp = new ServiceCollection()
    .AddScoped(typeof(IRepository<>), typeof(Repository<>))
    .AddScoped<IRepository<Order>, OrderRepository>()
    .BuildServiceProvider();
using var scope = sp.CreateScope();
Console.WriteLine(scope.ServiceProvider.GetRequiredService<IRepository<string>>().Describe());
Console.WriteLine(scope.ServiceProvider.GetRequiredService<IRepository<Order>>().Describe());
// Output:
// Generic repo of String
// Special order repo
```

### 5.3 [Medium] Difference between `Add`, `TryAdd`, `TryAddEnumerable` and `Replace`?
**Answer**
`Add*` always appends a descriptor; `TryAdd*` appends only if the service type has no registration; `TryAddEnumerable` appends only if that exact service+implementation pair is absent (for multi-registration idempotency); `Replace` removes the first existing registration for the service type and adds yours.

**Explanation**
Library code should use `TryAdd*` for defaults so consumers can pre-register overrides, and `TryAddEnumerable` for extension points (`IConfigureOptions<T>`, `IHostedService`) so calling `AddMyLib()` twice doesn't duplicate. `TryAddEnumerable` requires an implementation type (not a factory lambda) in the descriptor to determine uniqueness.

**Code**
```csharp
using Microsoft.Extensions.DependencyInjection.Extensions;

public interface IPlugin { } public sealed class P1 : IPlugin { } public sealed class P2 : IPlugin { }

var services = new ServiceCollection();
services.TryAddSingleton<IPlugin, P1>();   // added
services.TryAddSingleton<IPlugin, P2>();   // ignored: IPlugin already registered
Console.WriteLine(services.Count);          // Output: 1

services.TryAddEnumerable(ServiceDescriptor.Singleton<IPlugin, P2>());   // added (pair not present)
services.TryAddEnumerable(ServiceDescriptor.Singleton<IPlugin, P2>());   // ignored (same pair)
Console.WriteLine(services.Count);          // Output: 2

services.Replace(ServiceDescriptor.Singleton<IPlugin, P2>());  // replaces the first IPlugin (P1)
Console.WriteLine(string.Join(",", services.Select(d => d.ImplementationType!.Name)));
// Output: P2,P2
```

### 5.4 [Hard] Implement a decorator with the built-in container (and with Scrutor).
**Answer**
Register the inner implementation by concrete type, then register the interface with a factory that resolves the inner and wraps it in the decorator. With Scrutor, `services.Decorate<IService, Decorator>()` does this and preserves the original lifetime.

**Explanation**
Manual: `AddScoped<RealService>(); AddScoped<IService>(sp => new LoggingDecorator(sp.GetRequiredService<RealService>(), ...))`. Cons: constructor changes in the decorator mean updating the lambda; use `ActivatorUtilities.CreateInstance<LoggingDecorator>(sp, inner)` to auto-resolve the other parameters. Scrutor's `Decorate` finds the existing registration and wraps it, supporting open generics (`Decorate(typeof(IHandler<>), typeof(LoggingHandler<>))`) and stacking multiple decorators. Order: last decorator applied is outermost.

**Code**
```csharp
public interface IGreeter { string Greet(string n); }
public sealed class Greeter : IGreeter { public string Greet(string n) => $"Hi {n}"; }
public sealed class LoudGreeter(IGreeter inner) : IGreeter
{
    public string Greet(string n) => inner.Greet(n).ToUpperInvariant() + "!";
}

var services = new ServiceCollection();
services.AddSingleton<Greeter>();
services.AddSingleton<IGreeter>(sp => ActivatorUtilities.CreateInstance<LoudGreeter>(sp, sp.GetRequiredService<Greeter>()));
var sp = services.BuildServiceProvider();
Console.WriteLine(sp.GetRequiredService<IGreeter>().Greet("ann"));
// Output: HI ANN!

// Scrutor equivalent:
// services.AddSingleton<IGreeter, Greeter>();
// services.Decorate<IGreeter, LoudGreeter>();
```

**Gotchas / follow-ups**
- Registering `IGreeter -> Greeter` and then a factory for `IGreeter` that resolves `IGreeter` causes infinite recursion/stack overflow — resolve the *concrete* type.
- Lifetime of the decorator should be ≤ the inner's lifetime.

### 5.5 [Tricky] A class takes `IEnumerable<IValidator>` but you registered nothing. What is injected? What about `IValidator` itself?
**Answer**
`IEnumerable<IValidator>` resolves to an **empty** sequence — no exception. A direct `IValidator` dependency throws because the service isn't registered (`GetService` returns null; `GetRequiredService`/ctor injection throws).

**Explanation**
The container special-cases `IEnumerable<T>` as "all registrations of T", and zero is valid. This is why optional plug-in lists work without registration. Also note the enumerable is created per resolve; for singletons the list is captured once. Order = registration order, which can be significant for pipelines (middleware-like behaviours).

**Code**
```csharp
public interface IValidator { }
public sealed class Consumer(IEnumerable<IValidator> validators)
{
    public int Count => validators.Count();
}

var sp = new ServiceCollection().AddTransient<Consumer>().BuildServiceProvider();
Console.WriteLine(sp.GetRequiredService<Consumer>().Count);   // Output: 0
Console.WriteLine(sp.GetService<IValidator>() is null);       // Output: True
```

**Interview traps**
- Believing the first registration wins.
- Using `Add*` in library defaults and breaking consumer overrides (use `TryAdd*`).
- Decorator factory that resolves its own interface (infinite recursion).
- Expecting `Replace` to replace *all* registrations — only the first.
- Assuming `AddMyLib()` called twice is safe without `TryAddEnumerable`.

**Cheat sheet**
- Single resolve = last wins; `IEnumerable<T>` = all, in order, empty is OK.
- Open generic: `AddScoped(typeof(IRepo<>), typeof(Repo<>))`.
- `TryAdd` = default-if-missing; `TryAddEnumerable` = idempotent multi-add; `Replace` = swap first.
- Decorator = factory w/ concrete inner, or Scrutor `Decorate`.

---

## Topic 6: Keyed Services, Factory Registrations & Func\<T\>

**Study notes**

**Keyed services (.NET 8):** register by `(Type, object key)`.
```csharp
services.AddKeyedSingleton<IPaymentGateway, StripeGateway>("stripe");
services.AddKeyedScoped<IPaymentGateway, PayPalGateway>("paypal");
// consume:   [FromKeyedServices("stripe")] IPaymentGateway gw      (ctor, endpoint or action parameter)
// or:        sp.GetRequiredKeyedService<IPaymentGateway>("paypal")
```
- Key can be any object (compared with `Equals`); string/enum most common. `KeyedService.AnyKey` registers/resolves for any key; `GetKeyedServices<T>(KeyedService.AnyKey)` returns all keyed registrations.
- A keyed registration is **not** returned by plain `GetService<T>()` (non-keyed lookup) — except a registration with `null` key counts as non-keyed.
- **Factory registrations:** `services.AddScoped<IFoo>(sp => new Foo(sp.GetRequiredService<Bar>(), "literal"))` — for primitive args, conditional construction, wrapping, runtime config. Instances created by factories **are** disposed by the container.
- **`Func<T>`/`Lazy<T>`:** NOT supported automatically by the built-in container (Autofac does). You register them: `services.AddSingleton<Func<IFoo>>(sp => () => sp.GetRequiredService<IFoo>())`. Beware: the func captures the provider it was built with (root if singleton) → scope issues.
- Other strategy-selection options: `IEnumerable<T>` + a discriminator property, or a factory/resolver class.

### 6.1 [Easy] What are keyed services in .NET 8 and why were they added?
**Answer**
Keyed services let you register and resolve several implementations of the same interface under a key, e.g. `AddKeyedSingleton<IGateway, Stripe>("stripe")`, and inject with `[FromKeyedServices("stripe")]`. They replace hand-written resolver/dictionary workarounds for choosing between implementations.

**Explanation**
Before .NET 8 you used `IEnumerable<T>` and filtered, `Func<string, T>` factories, or third-party containers. Keyed services make that first-class in the abstractions (`IKeyedServiceProvider`), so libraries can use them too.

**Code**
```csharp
public interface IGateway { string Name { get; } }
public sealed class Stripe : IGateway { public string Name => "stripe"; }
public sealed class PayPal : IGateway { public string Name => "paypal"; }
public sealed class Checkout([FromKeyedServices("paypal")] IGateway gateway)
{
    public string Pay() => gateway.Name;
}

var sp = new ServiceCollection()
    .AddKeyedSingleton<IGateway, Stripe>("stripe")
    .AddKeyedSingleton<IGateway, PayPal>("paypal")
    .AddTransient<Checkout>()
    .BuildServiceProvider();
Console.WriteLine(sp.GetRequiredService<Checkout>().Pay());
Console.WriteLine(sp.GetRequiredKeyedService<IGateway>("stripe").Name);
// Output:
// paypal
// stripe
```

### 6.2 [Follow-up] How do you choose an implementation at runtime (e.g. per request) with keyed services?
**Answer**
Inject `IServiceProvider` (as `IKeyedServiceProvider`) into a small resolver/factory class and call `GetRequiredKeyedService<T>(key)` with the runtime key. Keep that lookup inside a dedicated factory so the rest of the code stays locator-free.

**Explanation**
`[FromKeyedServices]` fixes the key at compile time. For dynamic keys a factory is the legitimate locator use. Missing keys throw `InvalidOperationException`; use `GetKeyedService<T>(key)` (returns null) if optional.

**Code**
```csharp
public sealed class GatewayFactory(IServiceProvider sp)
{
    public IGateway For(string key) => sp.GetRequiredKeyedService<IGateway>(key);
}

var sp = new ServiceCollection()
    .AddKeyedSingleton<IGateway, Stripe>("stripe")
    .AddKeyedSingleton<IGateway, PayPal>("paypal")
    .AddSingleton<GatewayFactory>()
    .BuildServiceProvider();
Console.WriteLine(sp.GetRequiredService<GatewayFactory>().For("stripe").Name);
// Output: stripe
```

### 6.3 [Medium] When would you use a factory registration, and what are its pitfalls?
**Answer**
Use it when construction needs runtime values, primitives, conditional logic, or wrapping (decorators). Pitfalls: you can silently bypass validation of dependencies, resolve the wrong lifetime, and create service-locator style code in the lambda; also `ValidateOnBuild` can't analyse inside the lambda.

**Explanation**
Lambda gets the *current* provider (scoped when resolving a scoped service). Disposable results are tracked and disposed. Prefer `IOptions<T>` for configuration values over reading `IConfiguration` in lambdas.

**Code**
```csharp
public sealed class ApiClient(HttpClient http, string baseUrl) { public string Url => baseUrl; }

var services = new ServiceCollection();
services.AddSingleton(new Dictionary<string, string> { ["Api"] = "https://example.test" });
services.AddHttpClient();
services.AddTransient<ApiClient>(sp =>
    new ApiClient(sp.GetRequiredService<IHttpClientFactory>().CreateClient(),
                  sp.GetRequiredService<Dictionary<string, string>>()["Api"]));
using var provider = services.BuildServiceProvider();
Console.WriteLine(provider.GetRequiredService<ApiClient>().Url);
// Output: https://example.test
```

### 6.4 [Hard] The container doesn't support `Func<T>`/`Lazy<T>`. How do you add them safely?
**Answer**
Register open-generic wrappers or specific funcs that resolve from a provider: `services.AddSingleton<Func<IFoo>>(sp => () => sp.GetRequiredService<IFoo>())`. For scoped targets, the singleton func must create a scope or be registered as scoped so it captures the scoped provider.

**Explanation**
A `Func<T>` closed over the provider remembers *that* provider. Registered as singleton, it captures the root → scoped `IFoo` would be resolved from root (validation error / captive). Registering the func as **scoped** captures the request's provider. For `Lazy<T>` there's a neat open-generic trick: `services.AddTransient(typeof(Lazy<>), typeof(LazyService<>))` where `LazyService<T>` derives from `Lazy<T>` and takes `IServiceProvider`.

**Code**
```csharp
public sealed class LazyService<T> : Lazy<T> where T : notnull
{
    public LazyService(IServiceProvider sp) : base(() => sp.GetRequiredService<T>()) { }
}
public sealed class Heavy { public Heavy() => Console.WriteLine("Heavy created"); }

var services = new ServiceCollection();
services.AddSingleton<Heavy>();
services.AddTransient(typeof(Lazy<>), typeof(LazyService<>));
var sp = services.BuildServiceProvider();
var lazy = sp.GetRequiredService<Lazy<Heavy>>();
Console.WriteLine("Resolved Lazy");
_ = lazy.Value;
// Output:
// Resolved Lazy
// Heavy created
```

**Gotchas / follow-ups**
- `Lazy<T>` hides the dependency from the graph and from `ValidateOnBuild`.
- A `Func<T>` for a transient returns a fresh instance per call — and tracks disposables in the capturing scope.

### 6.5 [Tricky] You registered `AddKeyedSingleton<IGateway, Stripe>("stripe")`. What does `GetService<IGateway>()` return, and what does a ctor `IGateway` parameter get?
**Answer**
Both return null/throw: a keyed registration isn't visible to non-keyed lookups. `GetService<IGateway>()` returns null, and a plain ctor `IGateway` parameter fails to resolve. You must ask with the key (`[FromKeyedServices("stripe")]`) — or register a non-keyed `IGateway` too.

**Explanation**
Keyed and non-keyed registrations live in separate lookup spaces. Special cases: `KeyedService.AnyKey` resolves for any key; a ctor parameter marked `[ServiceKey]` receives the key used for resolution (useful with `AnyKey` registrations). `[FromKeyedServices]` with a null key behaves like non-keyed lookup.

**Code**
```csharp
var sp = new ServiceCollection().AddKeyedSingleton<IGateway, Stripe>("stripe").BuildServiceProvider();
Console.WriteLine(sp.GetService<IGateway>() is null);                        // Output: True
Console.WriteLine(sp.GetKeyedService<IGateway>("stripe") is not null);       // Output: True
```

**Interview traps**
- Expecting `IEnumerable<IGateway>` to include keyed registrations.
- Saying `Func<T>` just works in the built-in container.
- Singleton `Func<T>` capturing the root provider for scoped services.
- Keyed lookups with a key of a different type (`"1"` vs `1`) silently miss.

**Cheat sheet**
- .NET 8: `AddKeyed*`, `[FromKeyedServices(key)]`, `GetRequiredKeyedService<T>(key)`.
- Keyed ≠ non-keyed lookups.
- Factory lambdas get the current provider; created disposables are container-tracked.
- `Func<T>`/`Lazy<T>` need manual registration.
- Dynamic key ⇒ small factory class.

---

## Topic 7: IDisposable & Container Disposal Rules

**Study notes**

Rules of the built-in container:
1. The container disposes **only instances it created** (via type or factory registration), when the owning scope / root provider is disposed. Order: **reverse of creation**.
2. Instances passed in (`AddSingleton(instance)`) are **never disposed** by the container.
3. Scoped/transient disposables are disposed at **scope** end; singletons at **root provider** end (host shutdown); transients resolved from root live until root disposal.
4. `IAsyncDisposable` is honoured by `DisposeAsync` on the scope/provider; if you call sync `Dispose` on a provider and a service implements only `IAsyncDisposable`, it throws `InvalidOperationException`.
5. **Do not dispose** injected dependencies yourself — you don't own them.
6. `HttpClient` from `IHttpClientFactory`: don't dispose handler lifetimes manually; `HttpClient` instances can be disposed harmlessly but pooling is by handler.

```
scope.Dispose()  ->  disposes tracked: [C, B, A]  (reverse creation order, so dependents die before their dependencies)
```

### 7.1 [Easy] Who disposes an `IDisposable` service registered in DI?
**Answer**
The container disposes disposable services it created, at the end of the scope for scoped/transient ones and at app shutdown for singletons. You should not call `Dispose` on injected services yourself.

**Explanation**
Ownership principle: whoever creates it disposes it. Since the container constructs the object, it owns it. A consumer that received it by injection doesn't own it and disposing it could break other consumers.

**Code**
```csharp
public sealed class Tracked(string name) : IDisposable { public void Dispose() => Console.WriteLine($"dispose {name}"); }

var sp = new ServiceCollection()
    .AddScoped<Tracked>(_ => new Tracked("scoped"))
    .BuildServiceProvider();
using (var scope = sp.CreateScope())
{
    _ = scope.ServiceProvider.GetRequiredService<Tracked>();
    Console.WriteLine("end of scope");
}
// Output:
// end of scope
// dispose scoped
```

### 7.2 [Follow-up] What is the disposal order, and why does it matter?
**Answer**
The container disposes in reverse order of creation. Since dependencies are created before dependents, dependents are disposed first, so a dependent's `Dispose` can still safely use its dependency.

**Explanation**
Consider a `UnitOfWork` depending on a `DbContext`: UoW is created after the context, so UoW disposes first and can flush/rollback using a still-live context. Order is by *creation*, not registration.

**Code**
```csharp
public sealed class Low : IDisposable { public void Dispose() => Console.WriteLine("Low disposed"); }
public sealed class High(Low low) : IDisposable { public void Dispose() => Console.WriteLine("High disposed"); }

var sp = new ServiceCollection().AddScoped<Low>().AddScoped<High>().BuildServiceProvider();
using (var s = sp.CreateScope()) { _ = s.ServiceProvider.GetRequiredService<High>(); }
// Output:
// High disposed
// Low disposed
```

### 7.3 [Medium] You register an existing instance with `AddSingleton(instance)`. Who disposes it? What about a factory lambda?
**Answer**
For an instance registration the caller must dispose it — the container won't. For a factory lambda the container owns and disposes the result. If you want container ownership of something you already built, wrap it in a factory returning it.

**Explanation**
This is a common source of leaks (`AddSingleton(new HttpClient())`, `AddSingleton(new SqlConnection(...))`) and of confusing double disposal when both the app and container dispose. Choose a single owner. Factory registrations are lazily invoked and tracked.

**Code**
```csharp
public sealed class Res(string n) : IDisposable { public void Dispose() => Console.WriteLine($"disposed {n}"); }

var sc = new ServiceCollection();
sc.AddSingleton(new Res("instance"));                      // caller owns
sc.AddSingleton<IDisposable>(_ => new Res("factory"));     // container owns
var sp = sc.BuildServiceProvider();
_ = sp.GetRequiredService<IDisposable>();
sp.Dispose();
// Output: disposed factory      (instance is never disposed by the container)
```

### 7.4 [Hard] How does `IAsyncDisposable` work with the container, and what is the sync/async mismatch pitfall?
**Answer**
`await using var scope = sp.CreateAsyncScope()` or `await provider.DisposeAsync()` calls `DisposeAsync` on `IAsyncDisposable` services (and `Dispose` on sync-only ones). If you dispose synchronously and a service implements *only* `IAsyncDisposable`, the container throws `InvalidOperationException` because it cannot block-dispose it.

**Explanation**
`ServiceProvider` implements both `IDisposable` and `IAsyncDisposable`. Services implementing both get `DisposeAsync` when disposed asynchronously and `Dispose` when synchronously. ASP.NET Core and the Generic Host dispose scopes/providers asynchronously, so this normally works; unit tests using `using var` on scopes can hit the exception. Use `CreateAsyncScope()` — it returns an `AsyncServiceScope` struct supporting `await using`.

**Code**
```csharp
public sealed class AsyncOnly : IAsyncDisposable
{
    public ValueTask DisposeAsync() { Console.WriteLine("async disposed"); return ValueTask.CompletedTask; }
}

var sp = new ServiceCollection().AddScoped<AsyncOnly>().BuildServiceProvider();
await using (var scope = sp.CreateAsyncScope())
{
    _ = scope.ServiceProvider.GetRequiredService<AsyncOnly>();
}
// Output: async disposed

using var syncScope = sp.CreateScope();
_ = syncScope.ServiceProvider.GetRequiredService<AsyncOnly>();
// leaving this scope synchronously throws InvalidOperationException (async-only disposable)
```

### 7.5 [Tricky] A scoped service is disposed while a fire-and-forget task still uses it. What happens, and what are the correct patterns?
**Answer**
The request scope ends when the response completes, disposing scoped services; the background task then hits `ObjectDisposedException` (typically a disposed `DbContext`). Instead, start the background work in its own scope created from `IServiceScopeFactory`, or enqueue to a hosted-service queue.

**Explanation**
The work must own its own scope: capture `IServiceScopeFactory` (singleton) and create/dispose a scope inside the task. Better: use a `Channel<T>` queue consumed by a `BackgroundService` that creates a scope per item. Don't capture `HttpContext`/`RequestServices` into the task.

**Code**
```csharp
app.MapPost("/send", (IServiceScopeFactory factory) =>
{
    _ = Task.Run(async () =>
    {
        await using var scope = factory.CreateAsyncScope();
        var mailer = scope.ServiceProvider.GetRequiredService<IEmailSender>();
        mailer.Send("a@b.test");
        await Task.CompletedTask;
    });
    return Results.Accepted();
});
```

**Gotchas / follow-ups**
- Unobserved exceptions in `Task.Run` are lost; log inside the task.
- Prefer a durable queue for important work (process may stop before the task ends).

**Interview traps**
- Disposing an injected service in your own `Dispose`.
- `AddSingleton(new X())` for disposables assumed to be cleaned up.
- Sync-disposing a scope containing async-only disposables.
- Resolving disposable transients from the root in a loop (leak).

**Cheat sheet**
- Container disposes what it creates, reverse creation order.
- Instance registration = you own it.
- Scope end → scoped/transient; root end → singleton.
- Use `CreateAsyncScope()`/`await using` for async disposables.
- Background work ⇒ own scope.

---

## Topic 8: Autofac / Scrutor vs Built-in; DI Without a Framework

**Study notes**

| Capability | Built-in MS.DI | Autofac | Scrutor |
|---|---|---|---|
| Constructor injection, lifetimes, open generics | Yes | Yes | (extends MS.DI) |
| Property injection | No | Yes (`PropertiesAutowired`) | No |
| Assembly scanning | No | `RegisterAssemblyTypes` | `Scan(...)` |
| Decorators | No | `RegisterDecorator` | `Decorate<,>()` |
| Named/keyed | .NET 8 keyed | Named & keyed | n/a |
| `Func<T>`, `Lazy<T>`, `IIndex<K,V>` | No | Yes (implicit relationship types) | No |
| Modules, child lifetime scopes, interception | No | Yes | No |
| Speed | Fast, minimal | Slightly slower, richer | Same as MS.DI |

- **Scrutor** is a library on top of the built-in container: `services.Scan(s => s.FromAssemblyOf<Foo>().AddClasses(c => c.AssignableTo<IHandler>()).AsImplementedInterfaces().WithScopedLifetime())`.
- **Autofac** integrates via `builder.Host.UseServiceProviderFactory(new AutofacServiceProviderFactory())` and `ConfigureContainer<ContainerBuilder>`. Still implements `IServiceProvider`, so built-in registrations continue to work.
- **Pure DI / Poor man's DI** = hand-wire the graph in the composition root with `new`. Excellent for small apps, console tools and tests; compile-time safe; no lifetimes automation.

### 8.1 [Easy] When is the built-in container enough, and when would you add Autofac or Scrutor?
**Answer**
The built-in container covers constructor injection, lifetimes, open generics, keyed services and `IEnumerable<T>`, which suits most apps. Add Scrutor for assembly scanning and decorators while staying on the built-in container; choose Autofac for property injection, `Func<T>`/`Lazy<T>` relationships, modules, interception or child lifetime scopes.

**Explanation**
Fewer moving parts is the default: the built-in container is what the framework and libraries target. Third-party containers plug in via `IServiceProviderFactory<TContainerBuilder>` and keep compatibility with `IServiceCollection` registrations.

**Code**
```csharp
// Scrutor: scan and register all IHandler implementations as scoped
builder.Services.Scan(scan => scan
    .FromAssemblyOf<Program>()
    .AddClasses(c => c.AssignableTo<IHandler>())
    .AsImplementedInterfaces()
    .WithScopedLifetime());
public interface IHandler { }
```

### 8.2 [Follow-up] How do you plug Autofac into ASP.NET Core?
**Answer**
Replace the service provider factory with `AutofacServiceProviderFactory` and register extra types in `ConfigureContainer<ContainerBuilder>`. Registrations made on `builder.Services` still flow into Autofac.

**Explanation**
Autofac builds its container from the `IServiceCollection` descriptors plus your Autofac-specific registrations, and exposes an `AutofacServiceProvider` that implements `IServiceProvider`. Autofac lifetimes: `InstancePerDependency` (transient), `InstancePerLifetimeScope`/`InstancePerRequest` (scoped), `SingleInstance` (singleton).

**Code**
```csharp
using Autofac;
using Autofac.Extensions.DependencyInjection;

var builder = WebApplication.CreateBuilder(args);
builder.Host.UseServiceProviderFactory(new AutofacServiceProviderFactory());
builder.Host.ConfigureContainer<ContainerBuilder>(cb =>
{
    cb.RegisterType<Greeter>().As<IGreeter>().InstancePerLifetimeScope();
    cb.RegisterDecorator<LoudGreeter, IGreeter>();
});
var app = builder.Build();
```

### 8.3 [Medium] What is "Pure DI"? Show how you'd compose a graph without a container.
**Answer**
Pure DI means manually constructing the object graph in the composition root with `new`, passing dependencies explicitly. It gives compile-time safety and zero magic, but you manage lifetimes yourself, so it fits small apps, tests and tools.

**Explanation**
A singleton is a single variable; per-request objects are built inside the request handler. It exposes bloat early (large constructors), and refactoring is IDE-assisted. For ASP.NET Core you can still do pure DI via custom controller activators, but most teams use the container.

**Code**
```csharp
public interface ILogSink { void Write(string m); }
public sealed class ConsoleSink : ILogSink { public void Write(string m) => Console.WriteLine(m); }
public sealed class Mailer(ILogSink sink) { public void Send(string to) => sink.Write($"mail to {to}"); }
public sealed class SignupService(Mailer mailer) { public void Register(string email) => mailer.Send(email); }

// Composition root
ILogSink sink = new ConsoleSink();            // "singleton": one variable
var mailer = new Mailer(sink);
var signup = new SignupService(mailer);
signup.Register("a@b.test");
// Output: mail to a@b.test
```

### 8.4 [Hard] Autofac child lifetime scopes and decorators vs MS.DI — what can't MS.DI do?
**Answer**
MS.DI has only root and flat scopes — no nested/tagged lifetime scopes, no registration changes after build, no interception, no implicit `Func<T>`/`Lazy<T>`/`IEnumerable<T>`-with-metadata relationships, and no property injection. Autofac supports nested scopes (`BeginLifetimeScope(builder => ...)` with scope-local registrations), tagged scopes, and AOP via Castle/interceptors.

**Explanation**
Typical use: multi-tenant apps where each tenant scope overrides some registrations; plug-in modules via `Module`; cross-cutting concerns through interceptors. Trade-offs: more power, more hidden behaviour, a bit of runtime overhead, and subtle differences (e.g. Autofac's last-registration-wins also applies but with `PreserveExistingDefaults()` to flip). Many teams keep MS.DI + Scrutor and avoid the extra container.

**Code**
```csharp
using Autofac;

var cb = new ContainerBuilder();
cb.RegisterType<Greeter>().As<IGreeter>().SingleInstance();
using var container = cb.Build();

using var tenantScope = container.BeginLifetimeScope(b =>
    b.RegisterType<LoudGreeter2>().As<IGreeter>());          // tenant-specific override
Console.WriteLine(tenantScope.Resolve<IGreeter>().Greet("x"));
// Output: X!    (assuming LoudGreeter2 uppercases and appends '!')

public sealed class LoudGreeter2 : IGreeter { public string Greet(string n) => n.ToUpperInvariant() + "!"; }
```

### 8.5 [Tricky] You switch to Autofac and a previously working registration changes behaviour. Name likely causes.
**Answer**
Differences in default lifetimes (Autofac default is per-dependency/transient, but `Populate`d MS.DI descriptors keep their lifetimes), ordering of override registrations, property auto-wiring catching unexpected props, and disposal semantics (Autofac disposes anything it resolves that is `IDisposable`, including via its own tracking, unless `ExternallyOwned()`). Also `IEnumerable<T>` ordering and `Func<T>` implicit support can mask wiring bugs.

**Explanation**
Autofac registrations made directly via `ContainerBuilder` default to `InstancePerDependency`, unlike habit-forming `AddSingleton`. Implicit relationship types mean `Func<IFoo>` resolves silently, which differs from MS.DI where it would throw. Disposal: Autofac tracks disposables per lifetime scope; `ExternallyOwned()` opts out. Test with the real container in an integration test (`WebApplicationFactory`).

**Code**
```csharp
using Autofac;
var cb = new ContainerBuilder();
cb.RegisterInstance(new Res("shared")).ExternallyOwned();   // do not dispose on container disposal
```

**Interview traps**
- "We need Autofac for decorators" — Scrutor or a factory does it on MS.DI.
- Mixing both containers' registrations and forgetting ordering rules.
- Pure DI dismissed as "unprofessional" — it's valid and often best for small apps.
- Forgetting that Autofac default lifetime ≠ MS.DI's explicit choice.

**Cheat sheet**
- MS.DI by default; Scrutor for scan+decorate; Autofac for the advanced features.
- Autofac plugs in through `IServiceProviderFactory`.
- Pure DI = hand-wired graph in the composition root.
- Autofac: `InstancePerDependency` default; `ExternallyOwned()` to prevent disposal.

---

## Topic 9: Configuration System

**Study notes**

`IConfiguration` is a flattened, hierarchical key/value view built from ordered **providers**; keys are case-insensitive, `:` is the section separator.

Default order for `WebApplication.CreateBuilder` / `Host.CreateDefaultBuilder` (later **overrides** earlier):

```
1. (host config: DOTNET_ env vars, command line for host settings)
2. appsettings.json                        (optional, reloadOnChange: true)
3. appsettings.{Environment}.json          (optional, reloadOnChange: true)
4. User Secrets                            (Development only, if the entry assembly has a UserSecretsId)
5. Environment variables                   (no prefix)
6. Command-line arguments                  (highest)
```

- Env var nesting: `Logging__LogLevel__Default=Debug` (`__` → `:`). Azure App Service: connection strings get prefixes like `SQLCONNSTR_`. Colon `:` also works on Windows/Azure but `__` works everywhere (e.g. Linux/Docker).
- **Azure Key Vault** provider: `builder.Configuration.AddAzureKeyVault(new Uri(vaultUri), new DefaultAzureCredential())` (package `Azure.Extensions.AspNetCore.Configuration.Secrets`). Secret names can't contain `:`; use `--` which maps to `:` (e.g. `ConnectionStrings--Default`).
- Read APIs: `config["Key"]`, `config.GetSection("S")`, `Get<T>()`, `Bind(obj)`, `GetValue<T>("k")`, `GetConnectionString("Name")` (= `ConnectionStrings:Name`). `Get<T>`/`Bind` need package `Microsoft.Extensions.Configuration.Binder` (included in web SDK).
- **reloadOnChange**: file providers watch for changes (file watcher / polling) and raise `IChangeToken`; `IConfiguration` values update but a consumer must be reload-aware (`IOptionsMonitor`, `IOptionsSnapshot`, `IChangeToken`). Env vars and Key Vault don't auto-reload (Key Vault supports `ReloadInterval` via `AzureKeyVaultConfigurationOptions`).
- `ConfigurationManager` (net6+) is both `IConfigurationBuilder` and `IConfigurationRoot`, so you can read config *while* still adding providers in minimal hosting.
- Arrays bind by index keys: `Urls:0`, `Urls:1`.

### 9.1 [Easy] Where does ASP.NET Core read configuration from by default, and which wins?
**Answer**
appsettings.json, appsettings.{Environment}.json, user secrets (Development), environment variables, then command-line args, in increasing precedence — the last provider added wins. So a command-line argument overrides an env var, which overrides appsettings.

**Explanation**
Each provider contributes key/value pairs; when the same key appears in several, the later provider's value is returned. That's what lets you ship defaults in JSON and override per environment through env vars (Docker/K8s/App Service) without rebuilding.

**Code**
```csharp
var builder = WebApplication.CreateBuilder(args);
// appsettings.json: { "Greeting": "from json" }
// env var:           Greeting=from env     -> wins over json
// command line:      --Greeting "from cli" -> wins over both
app.MapGet("/g", (IConfiguration cfg) => cfg["Greeting"]);
```

### 9.2 [Follow-up] How do hierarchical keys, env vars and arrays map?
**Answer**
JSON nesting becomes `:`-delimited keys (`Logging:LogLevel:Default`); env vars use `__` for the same (`Logging__LogLevel__Default`); arrays become indexed keys (`Urls:0`). Binding reads these back into POCOs.

**Explanation**
Internally all providers flatten to string key/value pairs (`ConfigurationProvider.Data`). That means everything is a string until bound. You can test providers by building a configuration from `AddInMemoryCollection`.

**Code**
```csharp
var cfg = new ConfigurationBuilder()
    .AddInMemoryCollection(new Dictionary<string, string?>
    {
        ["Db:Server"] = "sql1",
        ["Db:Port"] = "1433",
        ["Urls:0"] = "https://a",
        ["Urls:1"] = "https://b",
    })
    .Build();

Console.WriteLine(cfg["db:server"]);                       // Output: sql1   (case-insensitive)
Console.WriteLine(cfg.GetValue<int>("Db:Port") + 1);       // Output: 1434
Console.WriteLine(string.Join(",", cfg.GetSection("Urls").Get<string[]>()!));   // Output: https://a,https://b
```

### 9.3 [Medium] Explain user-secrets and why they aren't a production solution.
**Answer**
User secrets store key/values in a per-user JSON file outside the repo (`%APPDATA%\Microsoft\UserSecrets\<id>\secrets.json`) so developers don't commit credentials. They're **unencrypted**, only loaded in Development by default, and tied to the dev machine — production should use Key Vault, managed identity or platform-injected env vars.

**Explanation**
`dotnet user-secrets init` adds a `UserSecretsId` to the csproj; `dotnet user-secrets set "ConnectionStrings:Default" "..."`. The provider is added in `WebApplication.CreateBuilder` when environment is Development. They protect against accidental commits only, not against anyone with machine access.

**Code**
```csharp
// CLI:
//   dotnet user-secrets init
//   dotnet user-secrets set "Stripe:ApiKey" "sk_test_123"
var builder = WebApplication.CreateBuilder(args);              // auto-adds user secrets in Development
var key = builder.Configuration["Stripe:ApiKey"];              // available locally only
// Non-default hosts: new ConfigurationBuilder().AddUserSecrets<Program>();
```

### 9.4 [Hard] How does `reloadOnChange` work, and which consumers actually see changes?
**Answer**
JSON file providers watch the file via a `PhysicalFileProvider` change token; on change they reload their data and signal the `IConfiguration` change token. `IConfiguration` reads see new values immediately, but values already copied into objects (e.g. `IOptions<T>`, bound POCOs) don't change unless consumed through `IOptionsMonitor<T>`/`IOptionsSnapshot<T>` or by registering a change callback.

**Explanation**
`ChangeToken.OnChange(() => config.GetReloadToken(), callback)` is the underlying primitive. Pitfalls: editors that write by replace-and-rename can fire multiple reload events (debounce); in containers with mounted ConfigMaps the symlink swap may need polling (`DOTNET_USE_POLLING_FILE_WATCHER=true`); the change can be observed mid-write → transient parse errors, so prefer `optional: true, reloadOnChange: true` with validation. Env vars and command-line never reload.

**Code**
```csharp
var cfg = new ConfigurationBuilder()
    .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
    .Build();

IDisposable? sub = ChangeToken.OnChange(() => cfg.GetReloadToken(),
    () => Console.WriteLine($"Reloaded. Mode={cfg["Mode"]}"));
// edit appsettings.json -> prints: Reloaded. Mode=<new value>   (may fire more than once)
```

**Gotchas / follow-ups**
- Singleton that reads `IConfiguration["X"]` in its constructor and stores it won't refresh.
- `reloadOnChange: true` on JSON files only works if the file is under the content root/watchable path.

### 9.5 [Tricky] `appsettings.json` has `"Features": { "Beta": true }` and an env var sets `Features__Beta=false`. Then you add a *new provider* later in `Program.cs` via `builder.Configuration.AddJsonFile("extra.json")` containing `Beta: true`. What value wins?
**Answer**
`extra.json` wins: providers added after the defaults sit at the end of the list and have the highest precedence — even above environment variables and command-line args. Order of `Add*` calls, not provider "kind", determines precedence.

**Explanation**
`ConfigurationManager` appends new sources to the end, and later sources override. To keep env vars/CLI on top, re-add them after your custom provider (`AddEnvironmentVariables()`, `AddCommandLine(args)`). This ordering misunderstanding causes "my env var is ignored" bugs.

**Code**
```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Configuration.AddJsonFile("extra.json", optional: true);   // now beats env + CLI
builder.Configuration.AddEnvironmentVariables();                    // re-assert env precedence
builder.Configuration.AddCommandLine(args);                         // and CLI on top
```

**Interview traps**
- "Environment variables always override everything."
- Forgetting `__` for nested keys in env vars (and `--` in Key Vault secret names).
- Expecting `IOptions<T>` to refresh after file edits.
- Treating user-secrets as secure storage.
- Reading `config["Port"]` as int without conversion — it's a string.

**Cheat sheet**
- Default order: json → env json → user secrets (dev) → env vars → CLI; last wins.
- Nest with `:` (JSON/keys), `__` (env), `--` (Key Vault).
- Only file providers reload by default; consumers must be reload-aware.
- `ConnectionStrings:X` ⇒ `GetConnectionString("X")`.
- Custom providers added later override defaults.

---

## Topic 10: Options Pattern & Validation

**Study notes**

Bind a config section to a strongly-typed class and inject it.

| Interface | Lifetime | Reloads? | Named options | Use when |
|---|---|---|---|---|
| `IOptions<T>` | Singleton | No (value fixed after first read) | No | Static settings |
| `IOptionsSnapshot<T>` | **Scoped** | Yes, per scope/request | Yes | Web request needing latest settings; can't inject into singletons |
| `IOptionsMonitor<T>` | Singleton | Yes, immediately; `OnChange` callback | Yes (`Get(name)`) | Singletons/background services needing live updates |
| `IOptionsFactory<T>` | Singleton | — | — | Custom construction |

```csharp
builder.Services.AddOptions<MailOptions>()
    .BindConfiguration("Mail")                    // or .Bind(builder.Configuration.GetSection("Mail"))
    .ValidateDataAnnotations()                    // [Required], [Range] ...
    .Validate(o => o.Port != 25, "Port 25 is blocked")
    .ValidateOnStart();                           // fail at startup, not on first use
```

- Without `ValidateOnStart()`, validation runs lazily when `.Value`/`CurrentValue` is first accessed → `OptionsValidationException` at runtime.
- `IValidateOptions<T>` for complex/DI-aware validation. .NET 8 source-generated validation: `[OptionsValidator] partial class MyValidator : IValidateOptions<MailOptions>`.
- `services.Configure<T>(config.GetSection("X"))` is the older style (no validation chain); `PostConfigure<T>` runs after all `Configure`. `Configure` calls run in registration order.
- Named options: `Configure<T>("name", ...)`, `IOptionsMonitor<T>.Get("name")`.
- Keep options classes POCOs with settable properties (init-only works with binder in .NET 8; `required` + binder is awkward).

### 10.1 [Easy] What is the options pattern and why use it over `IConfiguration` directly?
**Answer**
It binds a configuration section to a strongly-typed class injected through `IOptions<T>`. You get type safety, validation, no magic strings scattered around, and reload support through the monitor/snapshot variants.

**Explanation**
Injecting `IConfiguration` everywhere leaks key strings and string-typed values into business code and makes testing harder. Options provide a clear contract per feature (interface segregation) and are easy to construct in tests with `Options.Create(new T { ... })`.

**Code**
```csharp
public sealed class MailOptions
{
    public string Host { get; set; } = "";
    public int Port { get; set; } = 25;
}

var cfg = new ConfigurationBuilder().AddInMemoryCollection(new Dictionary<string, string?>
{ ["Mail:Host"] = "smtp.test", ["Mail:Port"] = "587" }).Build();

var sp = new ServiceCollection().Configure<MailOptions>(cfg.GetSection("Mail")).BuildServiceProvider();
var opts = sp.GetRequiredService<IOptions<MailOptions>>().Value;
Console.WriteLine($"{opts.Host}:{opts.Port}");
// Output: smtp.test:587

// Unit test: var svc = new MailSender(Options.Create(new MailOptions { Host = "x" }));
```

### 10.2 [Follow-up] Compare `IOptions`, `IOptionsSnapshot` and `IOptionsMonitor`.
**Answer**
`IOptions<T>` is a singleton computed once and never changes; `IOptionsSnapshot<T>` is scoped and recomputed per request so it sees updated configuration at the start of each scope; `IOptionsMonitor<T>` is a singleton that always exposes the latest value and offers `OnChange`. Use monitor in singletons/background services, snapshot in per-request scoped code.

**Explanation**
Snapshot cannot be injected into singletons (scope validation fails). Monitor caches values and invalidates the cache on a configuration change token. Both support named options. `IOptionsSnapshot` rebuilds options each scope → slight cost.

**Code**
```csharp
public sealed class Poller(IOptionsMonitor<MailOptions> monitor, ILogger<Poller> log)
{
    public Poller Init()
    {
        monitor.OnChange(o => log.LogInformation("Mail host changed to {Host}", o.Host));
        return this;
    }
    public string CurrentHost => monitor.CurrentValue.Host;   // always latest
}
// IOptionsSnapshot<MailOptions> in a controller/endpoint: new value each request after file change
```

### 10.3 [Medium] How do you validate options and fail fast at startup?
**Answer**
Chain `AddOptions<T>().BindConfiguration("Section").ValidateDataAnnotations().ValidateOnStart()` (plus `.Validate(predicate, message)` for custom rules). `ValidateOnStart` makes the host validate during startup and throw `OptionsValidationException`, instead of failing on first use.

**Explanation**
`ValidateOnStart` (.NET 6+) registers the options type for validation when the host starts, forcing creation of the options (running all validators). Data annotations support `[Required]`, `[Range]`, `[Url]`; nested objects need `[ValidateObjectMembers]` (.NET 8). For DI-aware or cross-property validation implement `IValidateOptions<T>`.

**Code**
```csharp
using System.ComponentModel.DataAnnotations;

public sealed class MailOptions2
{
    [Required] public string Host { get; set; } = "";
    [Range(1, 65535)] public int Port { get; set; }
}

builder.Services.AddOptions<MailOptions2>()
    .BindConfiguration("Mail")
    .ValidateDataAnnotations()
    .Validate(o => o.Port != 25, "Port 25 is not allowed")
    .ValidateOnStart();
// Missing Mail:Host => app fails at startup:
// OptionsValidationException: DataAnnotation validation failed for 'MailOptions2' members: 'Host' ...
```

### 10.4 [Hard] Implement `IValidateOptions<T>` that needs a DI service, and explain the validation pipeline.
**Answer**
Register a class implementing `IValidateOptions<T>` (constructor-injected) with `AddSingleton<IValidateOptions<T>, MyValidator>()`. When options are created, `OptionsFactory<T>` runs all `IConfigureOptions`, then `IPostConfigureOptions`, then every `IValidateOptions<T>`; failures aggregate into `OptionsValidationException`.

**Explanation**
Order: `Configure*` (including binding) → `PostConfigure*` → validators. `ValidateOptionsResult.Fail(...)`/`Success`. For named options, `Validate(string? name, T options)` gets the name — return `Skip` if not your named instance. Because validators are singletons (registered), they can't depend on scoped services.

**Code**
```csharp
public sealed class MailValidator(IHostEnvironment env) : IValidateOptions<MailOptions>
{
    public ValidateOptionsResult Validate(string? name, MailOptions o)
    {
        if (env.IsProduction() && o.Host == "localhost")
            return ValidateOptionsResult.Fail("Host must not be localhost in Production");
        return ValidateOptionsResult.Success;
    }
}

builder.Services.AddOptions<MailOptions>().BindConfiguration("Mail").ValidateOnStart();
builder.Services.AddSingleton<IValidateOptions<MailOptions>, MailValidator>();
```

**Gotchas / follow-ups**
- `ValidateOnStart()` is called on the `OptionsBuilder<T>`; it registers a startup check that forces creation of that options type (running all its validators, including separately registered `IValidateOptions<T>`) when the host starts.
- .NET 8 `[OptionsValidator]` generates reflection-free validators (trim/AOT-friendly).

### 10.5 [Tricky] Config file changes at runtime; your singleton service injected `IOptions<T>` still shows old values, and the one using `IOptionsSnapshot<T>` fails at startup. Why and how to fix?
**Answer**
`IOptions<T>` caches its value for app lifetime, so it never updates. `IOptionsSnapshot<T>` is scoped, so injecting it into a singleton is a captive-dependency error. Fix: use `IOptionsMonitor<T>` in the singleton (reading `CurrentValue` each time or subscribing to `OnChange`).

**Explanation**
Don't store `monitor.CurrentValue` in a field in the constructor — that re-creates the staleness. Read it at the point of use. Also remember `OnChange` returns an `IDisposable` — dispose to unsubscribe. Also `OnChange` can fire multiple times for one edit.

**Code**
```csharp
public sealed class Mailer(IOptionsMonitor<MailOptions> monitor)
{
    public string Send(string to) => $"smtp {monitor.CurrentValue.Host}:{monitor.CurrentValue.Port} -> {to}";
}
builder.Services.AddSingleton<Mailer>();     // safe: monitor is singleton and reload-aware
```

**Interview traps**
- Using `IOptionsSnapshot<T>` in a singleton.
- Assuming `IOptions<T>` reloads.
- No `ValidateOnStart()` ⇒ misconfig discovered in production at first request.
- Caching `CurrentValue` in a constructor field.
- Putting secrets into options classes that get logged.

**Cheat sheet**
- `IOptions` = singleton/static; `Snapshot` = scoped/per request; `Monitor` = singleton/live.
- Validate: DataAnnotations, `.Validate(...)`, `IValidateOptions<T>`, `[OptionsValidator]` (.NET 8).
- `ValidateOnStart()` = fail fast.
- Pipeline: Configure → PostConfigure → Validate.
- Tests: `Options.Create(new T{...})`.

---

## Topic 11: Environments & Secrets Best Practices

**Study notes**

- Environment name from `ASPNETCORE_ENVIRONMENT` (web) / `DOTNET_ENVIRONMENT` (generic host); default **Production** if unset. Common values: Development, Staging, Production. Case matters only for file names on case-sensitive file systems.
- Check: `app.Environment.IsDevelopment()`, `IsStaging()`, `IsProduction()`, `IsEnvironment("QA")`; inject `IWebHostEnvironment`/`IHostEnvironment`.
- `launchSettings.json` (dev-time only, not deployed) sets env vars for `dotnet run`/VS.
- Development-only behaviours: developer exception page, user secrets provider, DI `ValidateScopes/ValidateOnBuild`, static web asset behaviours, detailed EF logging if enabled.
- **Secrets hierarchy** (best → worst): managed identity (no secret at all) → Key Vault / secret manager referenced at runtime → platform env vars/secret stores (K8s secrets, App Service settings with Key Vault references) → user-secrets (dev only) → files in repo (never).
- Avoid: secrets in `appsettings.json`, in source control, in container images, in logs (use `[LogProperties]`/redaction, never log connection strings), in client-side config.
- Rotation: use `DefaultAzureCredential`, short-lived tokens, Key Vault `ReloadInterval`, avoid long-lived static passwords.
- Config per environment: keep **structure** in appsettings.json, **overrides** in appsettings.{Env}.json, **secrets** outside files.

### 11.1 [Easy] How does ASP.NET Core determine the environment and what's the default?
**Answer**
It reads `ASPNETCORE_ENVIRONMENT` (or `DOTNET_ENVIRONMENT` for non-web hosts) and defaults to `Production` when unset. You can branch with `IsDevelopment()` etc. and it also selects which `appsettings.{Environment}.json` is loaded.

**Explanation**
Defaulting to Production is a safety feature: you won't accidentally expose the developer exception page in a deployment with a missing variable. `launchSettings.json` applies only when launching via VS/`dotnet run`.

**Code**
```csharp
var app = builder.Build();
if (app.Environment.IsDevelopment())
    app.UseDeveloperExceptionPage();
else
    app.UseExceptionHandler("/error");
Console.WriteLine(app.Environment.EnvironmentName);
// Output (nothing set): Production
```

### 11.2 [Follow-up] Where should secrets live in dev vs production?
**Answer**
Dev: `dotnet user-secrets` (outside the repo). Production: prefer managed identity so no secret exists; otherwise Azure Key Vault (via the configuration provider or App Service Key Vault references) or the platform's secret store injected as env vars. Never commit secrets or put them in appsettings.json.

**Explanation**
Each step reduces blast radius: managed identity lets the app authenticate to SQL/Storage/Key Vault with no credential to leak; Key Vault gives audit, rotation and RBAC; env vars are simple but visible to the process and to anyone with host access (`/proc`, container inspect).

**Code**
```csharp
using Azure.Identity;

var builder = WebApplication.CreateBuilder(args);
if (!builder.Environment.IsDevelopment())
{
    var vault = builder.Configuration["KeyVault:Uri"]!;       // non-secret value
    builder.Configuration.AddAzureKeyVault(new Uri(vault), new DefaultAzureCredential());
}
// secret "ConnectionStrings--Default" => ConnectionStrings:Default
```

### 11.3 [Medium] How do you avoid leaking secrets through logs, exceptions and options classes?
**Answer**
Don't log options or connection strings wholesale; mark sensitive members and exclude/redact them (e.g. `[JsonIgnore]`, `ToString` overrides, redaction libraries such as `Microsoft.Extensions.Compliance.Redaction`), prefer integrated auth/managed identity so connection strings hold no passwords, and scrub exception messages at boundaries.

**Explanation**
Structured logging serialises objects destructured with `{@Obj}` (Serilog) — a password property will be written. Exceptions from drivers sometimes include connection strings. Use `SqlConnectionStringBuilder` and remove `Password` before logging; or log only host/db name.

**Code**
```csharp
public sealed class DbOptions
{
    public string Server { get; set; } = "";
    public string Password { get; set; } = "";
    public override string ToString() => $"DbOptions {{ Server = {Server}, Password = *** }}";
}
var o = new DbOptions { Server = "sql1", Password = "p@ss" };
Console.WriteLine(o);
// Output: DbOptions { Server = sql1, Password = *** }
```

### 11.4 [Hard] Design configuration for dev / staging / prod with containers and Key Vault. What goes where?
**Answer**
Put non-secret defaults in appsettings.json, environment-specific non-secrets in appsettings.{Env}.json, secrets in Key Vault (accessed via managed identity), and deployment-specific values (URLs, flags) in env vars set by the orchestrator. The image is built once and the environment is chosen at deploy time.

**Explanation**
"Build once, deploy many": no config baked into the image except defaults. Use `ASPNETCORE_ENVIRONMENT` to select the file; map K8s secrets/ConfigMaps to env vars with `__`; use Key Vault references or CSI driver. Validate with `ValidateOnStart` so a bad environment fails fast and the readiness probe stays red. Feature flags (Topic 16) can live in App Configuration.

**Code**
```csharp
// appsettings.json                -> { "Mail": { "Host": "localhost", "Port": 25 } }       (defaults)
// appsettings.Production.json     -> { "Mail": { "Host": "smtp.prod.corp" } }              (env override)
// Key Vault secret "Mail--Password"                                                        (secret)
// Container env: Mail__Port=587                                                            (deployment override)
builder.Services.AddOptions<MailOptions>().BindConfiguration("Mail").ValidateOnStart();
```

### 11.5 [Tricky] You set `ASPNETCORE_ENVIRONMENT=Development` in production "just for a day" to debug. What can go wrong?
**Answer**
The developer exception page exposes stack traces and source, user-secrets and Development-only config load, DI validation and other dev behaviours change, Swagger/diagnostic endpoints guarded by `IsDevelopment()` become public, and `appsettings.Development.json` may point to dev resources or relax security. Use targeted logging/diagnostics (log level changes, Application Insights, dotnet-trace) instead.

**Explanation**
Many teams gate Swagger, seed data, `EnsureCreated`, CORS-allow-all and auth bypasses on `IsDevelopment()`. Flipping the environment flips all of them. Also, appsettings.Development.json is typically committed, so everyone sees its contents — no secrets there.

**Code**
```csharp
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();        // would be exposed publicly
    app.UseSwaggerUI();
}
// Safer: raise log level at runtime via config instead:  Logging__LogLevel__Default=Debug
```

**Interview traps**
- "ASPNETCORE_ENVIRONMENT must be Production/Development" — any string works.
- Secrets in `appsettings.Production.json` "because it's not committed".
- Relying on `launchSettings.json` in deployments (it isn't published).
- Logging `IConfiguration` dump at startup (includes secrets).

**Cheat sheet**
- Default env = Production; var is `ASPNETCORE_ENVIRONMENT`/`DOTNET_ENVIRONMENT`.
- Secrets: managed identity > Key Vault > env vars > user-secrets (dev).
- Build once, config per environment.
- Never flip Development in prod.
- Validate config at startup.

---

## Topic 12: Logging Architecture

**Study notes**

```
ILogger<T>  ──>  ILoggerFactory  ──>  [ILoggerProvider: Console | Debug | EventSource | EventLog | AppInsights | Serilog ...]
   (your code)       (singleton)            each provider has its own filter rules
```

- **Levels** (ascending): Trace(0), Debug(1), Information(2), Warning(3), Error(4), Critical(5), None(6).
- Inject `ILogger<T>` — category name = full type name of `T`. Filtering config: `Logging:LogLevel:Default`, per-category (`Microsoft.AspNetCore: Warning`), per-provider (`Logging:Console:LogLevel:...`). Most specific category prefix wins.
- **Structured logging / message templates:** `log.LogInformation("Order {OrderId} placed by {User}", id, user)` — named placeholders become structured properties; **positional**, not name-matched. Never use `$"..."` interpolation (loses structure, allocates, evaluates even when disabled).
- **Scopes:** `using (log.BeginScope("{CorrelationId}", id))` — state attached to all logs inside (requires provider support; console needs `IncludeScopes = true`).
- **`EventId`** identifies a log event type; **exceptions:** pass exception as the *first* argument: `log.LogError(ex, "Failed {Id}", id)`.
- **High-performance logging:** `LoggerMessage` source generator (.NET 6+): `[LoggerMessage]` on `static partial` methods generates cached delegates; avoids boxing & template parsing, checks `IsEnabled`.
- `log.IsEnabled(LogLevel.Debug)` guards expensive argument computation.
- Use `[LogProperties]`/redaction (`Microsoft.Extensions.Telemetry`/`Compliance`) for sensitive data.

### 12.1 [Easy] How do you use `ILogger` and what are the log levels?
**Answer**
Inject `ILogger<T>` and call `LogInformation`, `LogWarning`, `LogError`, etc. Levels from lowest to highest are Trace, Debug, Information, Warning, Error, Critical (and None to disable). Configured minimum levels filter what gets emitted per category/provider.

**Explanation**
Use Debug/Trace for development detail, Information for normal flow milestones, Warning for unexpected but recoverable, Error for failures of an operation, Critical for app-wide failure. Production default is typically Information (or Warning for noisy categories).

**Code**
```csharp
public sealed class OrderService(ILogger<OrderService> log)
{
    public void Place(int id, string user)
    {
        log.LogInformation("Placing order {OrderId} for {User}", id, user);
        try { if (id < 0) throw new ArgumentException("bad id"); }
        catch (Exception ex) { log.LogError(ex, "Order {OrderId} failed", id); throw; }
    }
}
// appsettings.json:
// "Logging": { "LogLevel": { "Default": "Information", "Microsoft.AspNetCore": "Warning" } }
```

### 12.2 [Follow-up] What is structured logging and why avoid string interpolation in log calls?
**Answer**
Structured logging records the template and named values separately so sinks can index/query properties (e.g. `OrderId = 5`). Interpolation bakes values into one string, losing the properties, preventing grouping by template, and doing formatting work even if the level is disabled.

**Explanation**
With templates, providers like Seq/Elastic/App Insights store `{OrderId}` as a field. Analyzer CA2254 flags non-constant templates. Placeholder *names* are for readability; arguments are matched by **position**.

**Code**
```csharp
log.LogInformation("User {User} bought {Count} items", "ann", 3);     // good: User and Count are properties
log.LogInformation($"User ann bought 3 items");                        // bad: one opaque string

// positional matching gotcha:
log.LogInformation("A {First} B {Second}", "x", "y");                  // First = x, Second = y
log.LogInformation("A {Second} B {First}", "x", "y");                  // Second = x, First = y  (by position!)
```

### 12.3 [Medium] What are logging scopes and how do you use them for context?
**Answer**
`BeginScope` attaches contextual state (like a correlation id or tenant) to every log entry emitted within the `using` block, across method calls and awaits (flowed with `AsyncLocal`). The provider must support scopes — e.g. console with `IncludeScopes`, Serilog via `LogContext`, App Insights automatically.

**Explanation**
ASP.NET Core already begins a scope per request containing `RequestId`/`RequestPath`/`SpanId`/`TraceId`/`ParentId` (when `ActivityTrackingOptions` enabled). You can pass a `Dictionary<string, object>` to add several keys. Scope value is part of provider-side state, not the message.

**Code**
```csharp
using var factory = LoggerFactory.Create(b => b.AddSimpleConsole(o => o.IncludeScopes = true));
var log = factory.CreateLogger("demo");
using (log.BeginScope(new Dictionary<string, object> { ["TenantId"] = "t42" }))
{
    log.LogInformation("Processing");
}
// Output (approx): info: demo[0]
//       => TenantId:t42
//       Processing
```

### 12.4 [Hard] Explain the `LoggerMessage` source generator and why it's faster.
**Answer**
Decorate a `static partial` method with `[LoggerMessage(EventId, Level, Message)]`; the generator emits an implementation holding a cached `Action<ILogger,...>` delegate (`LoggerMessage.Define`) with strongly-typed parameters. It avoids boxing of value types, params-array allocations and repeated template parsing, and checks `IsEnabled` before formatting.

**Explanation**
Regular `LogInformation(template, args)` allocates an `object[]`, boxes structs and parses/format-caches the template per call. Generated methods are typed (`int orderId`), so no boxing; the `FormattedLogValues` creation is skipped when the level is off. Requirements: containing class must be `partial`; logger parameter (`ILogger`) is found automatically; exception parameter is recognised; template placeholder names must match parameter names (case-insensitive) — the generator emits diagnostics otherwise. Use in hot paths and libraries.

**Code**
```csharp
public static partial class Log
{
    [LoggerMessage(EventId = 1001, Level = LogLevel.Information, Message = "Order {OrderId} shipped to {City}")]
    public static partial void OrderShipped(this ILogger logger, int orderId, string city);

    [LoggerMessage(EventId = 1002, Level = LogLevel.Error, Message = "Payment failed for {OrderId}")]
    public static partial void PaymentFailed(this ILogger logger, Exception ex, int orderId);
}

// usage
// logger.OrderShipped(5, "Pune");
// logger.PaymentFailed(ex, 5);
```

**Gotchas / follow-ups**
- Instance-method variant: class `partial` with an `ILogger` field/primary-ctor param, and the method is non-static without the logger param (supported when an `ILogger` field exists).
- Event IDs should be unique per category to enable filtering/alerting.
- Placeholder names that don't match a parameter name produce a generator diagnostic.

### 12.5 [Tricky] Predict the output / what's wrong with these logging calls?
**Answer**
(1) Passing the exception as an argument after the template doesn't attach it as an exception; use the overload with the exception first. (2) `LogDebug` args are evaluated even if Debug is disabled, so expensive calls need `IsEnabled`. (3) The `{OrderId}` name mismatch doesn't matter — matching is by position. (4) Logging inside a tight loop with string interpolation allocates heavily.

**Explanation**
`log.LogError("Failed {Ex}", ex)` logs `ex.ToString()` as a property but providers don't see `Exception` — Application Insights won't create an exception telemetry item. Right form: `log.LogError(ex, "Failed")`. Also don't catch-log-rethrow at every layer: duplicate noise; log once where handled.

**Code**
```csharp
try { throw new InvalidOperationException("boom"); }
catch (Exception ex)
{
    log.LogError("Failed {Ex}", ex);        // wrong: exception is just a formatted arg
    log.LogError(ex, "Failed");             // right: exception attached (stack trace, type)
}

if (log.IsEnabled(LogLevel.Debug))
    log.LogDebug("Snapshot {State}", ExpensiveDump());     // avoid computing when disabled
static string ExpensiveDump() => "...";
```

**Interview traps**
- `$"..."` in log messages.
- Swallowing exceptions after logging only `ex.Message` (loses stack).
- Logging PII/secrets.
- Assuming named placeholders match by name.
- Logging at Information inside tight loops (cost and cardinality).

**Cheat sheet**
- Levels: Trace < Debug < Information < Warning < Error < Critical < None.
- `ILogger<T>` category = type name; filters by category prefix.
- Templates, not interpolation; placeholders matched by position.
- Exception goes first: `LogError(ex, "...")`.
- `[LoggerMessage]` = zero-boxing, cached, fast.
- `BeginScope` for contextual properties.

---

## Topic 13: Serilog, Seq/ELK & Correlation IDs

**Study notes**

- **Serilog** = structured logging library with *sinks* (Console, File, Seq, Elasticsearch, Application Insights...), *enrichers* (`FromLogContext`, `WithMachineName`...), and *destructuring*. Plugs into `Microsoft.Extensions.Logging` via `Serilog.AspNetCore` (`UseSerilog`), so `ILogger<T>` code is unchanged.
- Message templates: `{Prop}` plain; `{@Obj}` destructure object into properties; `{$Obj}` force `ToString()`.
- `LogContext.PushProperty("K", v)` (needs `.Enrich.FromLogContext()`) is Serilog's scope mechanism; `ILogger.BeginScope` state is also forwarded as properties by the Serilog provider.
- `UseSerilogRequestLogging()` emits one summary event per request (method, path, status, elapsed) instead of many framework events.
- Use a **bootstrap logger** (`CreateBootstrapLogger`) to capture startup failures before DI/config exist; always `Log.CloseAndFlush()` on exit.
- **Seq**: dev-friendly structured log server (`Serilog.Sinks.Seq`, `WriteTo.Seq("http://localhost:5341")`) with query language. **ELK/Elastic**: ship JSON logs (Serilog Elasticsearch sink, or write JSON to stdout and let Filebeat/Fluent Bit/Logstash forward) → Elasticsearch → Kibana. In containers prefer *stdout JSON + collector* over app-side network sinks.
- **Correlation ID**: an id carried across all logs/calls of one logical request. Modern standard: W3C **Trace Context** (`traceparent` header: `00-<traceId>-<spanId>-<flags>`) handled automatically by `System.Diagnostics.Activity` (ASP.NET Core and `HttpClient` propagate it). Custom `X-Correlation-ID` is the legacy/simple alternative.

```
Client ──traceparent──> API A ──(HttpClient adds traceparent)──> API B ──> DB
        every log line in A and B carries the same TraceId  => one query finds the whole story
```

### 13.1 [Easy] How do you wire Serilog into ASP.NET Core?
**Answer**
Add `Serilog.AspNetCore`, then call `builder.Host.UseSerilog((ctx, lc) => lc.ReadFrom.Configuration(ctx.Configuration).WriteTo.Console())`. Existing `ILogger<T>` code keeps working and log output now flows through Serilog's pipeline and sinks.

**Explanation**
`UseSerilog` replaces the logging providers with a Serilog provider. Reading from configuration (`Serilog` section) lets you change sinks and levels without recompiling (needs `Serilog.Settings.Configuration`).

**Code**
```csharp
using Serilog;

Log.Logger = new LoggerConfiguration().WriteTo.Console().CreateBootstrapLogger();
try
{
    var builder = WebApplication.CreateBuilder(args);
    builder.Host.UseSerilog((ctx, services, lc) => lc
        .ReadFrom.Configuration(ctx.Configuration)
        .ReadFrom.Services(services)
        .Enrich.FromLogContext()
        .WriteTo.Console());

    var app = builder.Build();
    app.UseSerilogRequestLogging();
    app.MapGet("/", () => "ok");
    app.Run();
}
catch (Exception ex) { Log.Fatal(ex, "Host terminated unexpectedly"); }
finally { Log.CloseAndFlush(); }
```

### 13.2 [Follow-up] What is the difference between `{Obj}`, `{@Obj}` and `{$Obj}` in Serilog?
**Answer**
`{Obj}` renders the value with its default scalar formatting (usually `ToString()`); `{@Obj}` destructures — captures the object's public properties as structured data; `{$Obj}` forces stringification. Destructure carefully: it can leak sensitive properties and create huge events.

**Explanation**
Destructuring depth/size can be limited (`Destructure.ToMaximumDepth`, `ToMaximumCollectionCount`) and customised (`Destructure.ByTransforming<T>`) to drop secrets. In plain MEL, the `@` and `$` prefixes are parsed by Serilog only; other providers print them literally.

**Code**
```csharp
public sealed record Customer(int Id, string Name, string Password);

var c = new Customer(1, "Ann", "secret");
Log.Information("Plain {Customer}", c);      // Customer = "Customer { Id = 1, Name = Ann, Password = secret }" (ToString of record)
Log.Information("Destructured {@Customer}", c); // Customer = { Id: 1, Name: "Ann", Password: "secret" }  <- leaks!
Log.Information("Stringified {$Customer}", c);  // forced ToString()

// safe: Destructure.ByTransforming<Customer>(x => new { x.Id, x.Name })
```

### 13.3 [Medium] How do you implement correlation IDs end-to-end in an API?
**Answer**
Prefer W3C trace context: `Activity.Current.TraceId` is created/propagated automatically and can be added to logs. For a custom header, middleware reads `X-Correlation-ID` (or generates one), pushes it into a log scope (`LogContext.PushProperty`/`BeginScope`), echoes it on the response, and an `HttpClient` `DelegatingHandler` forwards it downstream.

**Explanation**
Scope flows via `AsyncLocal`, so every log inside the request has the property. Avoid reading `HttpContext` from deep layers — use the scope/Activity instead. Downstream calls via `IHttpClientFactory` automatically propagate `traceparent` when an `Activity` is present (DiagnosticsHandler), but not custom headers.

**Code**
```csharp
public sealed class CorrelationMiddleware(RequestDelegate next, ILogger<CorrelationMiddleware> log)
{
    private const string Header = "X-Correlation-ID";

    public async Task InvokeAsync(HttpContext ctx)
    {
        var id = ctx.Request.Headers.TryGetValue(Header, out var v) && !string.IsNullOrWhiteSpace(v)
            ? v.ToString()
            : Activity.Current?.TraceId.ToString() ?? Guid.NewGuid().ToString("N");

        ctx.Response.OnStarting(() => { ctx.Response.Headers[Header] = id; return Task.CompletedTask; });
        ctx.Items[Header] = id;

        using (log.BeginScope(new Dictionary<string, object> { ["CorrelationId"] = id }))
            await next(ctx);
    }
}

public sealed class CorrelationHandler(IHttpContextAccessor accessor) : DelegatingHandler
{
    protected override Task<HttpResponseMessage> SendAsync(HttpRequestMessage req, CancellationToken ct)
    {
        if (accessor.HttpContext?.Items["X-Correlation-ID"] is string id)
            req.Headers.TryAddWithoutValidation("X-Correlation-ID", id);
        return base.SendAsync(req, ct);
    }
}
// builder.Services.AddHttpContextAccessor();
// builder.Services.AddTransient<CorrelationHandler>();
// builder.Services.AddHttpClient("downstream").AddHttpMessageHandler<CorrelationHandler>();
// app.UseMiddleware<CorrelationMiddleware>();   // early in the pipeline
```

### 13.4 [Hard] How do logs relate to `Activity`/TraceId, and how do you get `TraceId` into every log line?
**Answer**
ASP.NET Core starts an `Activity` per request; with `ActivityTrackingOptions` set (default adds `SpanId`, `TraceId`, `ParentId` scope in the default logger config) the logging scope includes those ids. Serilog enriches via `Enrich.WithSpan()` (Serilog.Enrichers.Span) or by pushing `Activity.Current` ids; OpenTelemetry's log exporter correlates automatically.

**Explanation**
`Activity.Current` is AsyncLocal. Logs with the same `TraceId` appear in Seq/Kibana/App Insights as one distributed transaction and link to traces. `LoggerFactoryOptions.ActivityTrackingOptions` (`TraceId | SpanId | ParentId | Baggage | Tags | TraceState | TraceFlags`) controls what's attached as scope when using MEL providers.

**Code**
```csharp
builder.Logging.Configure(o =>
    o.ActivityTrackingOptions = ActivityTrackingOptions.TraceId | ActivityTrackingOptions.SpanId);

app.MapGet("/trace", (ILogger<Program> log) =>
{
    var tid = Activity.Current?.TraceId.ToString();
    log.LogInformation("Handling request");           // scope carries TraceId/SpanId
    return Results.Ok(new { traceId = tid });
});
```

**Gotchas / follow-ups**
- `Activity.Current` is null if no listener/instrumentation created an Activity; ASP.NET Core creates one only when something listens (OTel, diagnostics) — with `Microsoft.AspNetCore.Hosting` ActivityTracking enabled it generally exists.
- Return the correlation/trace id in error responses (`ProblemDetails.Extensions["traceId"]`) so support can find the logs.

### 13.5 [Tricky] Logs vanish on shutdown/crash, or the app slows under load with Serilog. Why?
**Answer**
Buffered/async sinks may not flush on abrupt termination — call `Log.CloseAndFlush()` (or dispose the logger) in `finally`. Synchronous sinks (network, file with `shared`) block request threads; wrap them in `WriteTo.Async(...)` (`Serilog.Sinks.Async`) with a bounded queue and understand the drop-vs-block policy.

**Explanation**
Also check: minimum-level overrides (`MinimumLevel.Override("Microsoft", Warning)`) to cut framework noise; sampling for high-volume debug logs; Seq API keys and batching; an unreachable sink (self-log with `SelfLog.Enable`). When using stdout JSON in containers, avoid app-side file/network sinks.

**Code**
```csharp
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .MinimumLevel.Override("Microsoft.AspNetCore", Serilog.Events.LogEventLevel.Warning)
    .WriteTo.Async(a => a.File("logs/app-.log", rollingInterval: RollingInterval.Day))
    .WriteTo.Seq("http://localhost:5341")
    .CreateLogger();

AppDomain.CurrentDomain.ProcessExit += (_, _) => Log.CloseAndFlush();
Serilog.Debugging.SelfLog.Enable(Console.Error);     // diagnose sink failures
```

**Interview traps**
- `{@Obj}` on entities (leaks data, giant payloads, lazy-loading side effects).
- Custom correlation header only — missing W3C `traceparent` interop.
- Logging to files inside containers instead of stdout.
- Forgetting `Log.CloseAndFlush()`.
- Using HttpContext in deep layers to fetch correlation id.

**Cheat sheet**
- `UseSerilog` + `ReadFrom.Configuration` + `Enrich.FromLogContext`.
- `{@x}` destructure, `{$x}` stringify.
- Correlation = W3C `traceparent` / `Activity.TraceId`; propagate custom headers with a `DelegatingHandler`.
- Seq = dev/small-team log server; ELK = ship JSON via collectors.
- Bootstrap logger + `CloseAndFlush`.

---

## Topic 14: OpenTelemetry Tracing & Metrics

**Study notes**

OpenTelemetry (OTel) = vendor-neutral standard for **traces, metrics, logs**. .NET's native APIs *are* the OTel APIs:

| Signal | .NET API | Notes |
|---|---|---|
| Traces | `System.Diagnostics.ActivitySource` / `Activity` | Activity = span; `StartActivity` returns `null` when nobody listens |
| Metrics | `System.Diagnostics.Metrics.Meter`, `Counter<T>`, `Histogram<T>`, `UpDownCounter<T>`, observable gauges | .NET 8 adds `IMeterFactory` (DI-friendly) |
| Logs | `ILogger` | Exported with `builder.Logging.AddOpenTelemetry(...)` |

Pipeline: *instrumentation* (ASP.NET Core, HttpClient, SqlClient, EF Core, your own sources) → *SDK* (sampling, processors, resource attributes like `service.name`) → *exporter* (OTLP → Collector/Jaeger/Tempo/Prometheus/Azure Monitor/Application Insights).

```csharp
builder.Services.AddOpenTelemetry()
    .ConfigureResource(r => r.AddService("orders-api"))
    .WithTracing(t => t
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddSource("Orders")                         // subscribe to custom ActivitySource
        .AddOtlpExporter())
    .WithMetrics(m => m
        .AddAspNetCoreInstrumentation()
        .AddMeter("Orders")                          // subscribe to custom Meter
        .AddOtlpExporter());
```
(Packages: `OpenTelemetry.Extensions.Hosting`, `OpenTelemetry.Instrumentation.AspNetCore`, `...Http`, `OpenTelemetry.Exporter.OpenTelemetryProtocol`.)

- Context propagation: W3C `traceparent` + `tracestate`; baggage via `Activity.AddBaggage`.
- **Sampling**: head sampling (`TraceIdRatioBasedSampler`, parent-based) vs tail sampling (done in the Collector).
- Metric tags (dimensions) must be **low cardinality** (no user ids / URLs with ids).

### 14.1 [Easy] What are traces, spans and metrics, and how does .NET expose them?
**Answer**
A trace is the end-to-end journey of a request made of spans (timed operations with attributes); metrics are numeric measurements aggregated over time (counters, histograms, gauges). In .NET, spans are `Activity` objects created from an `ActivitySource`, and metrics come from `Meter` instruments.

**Explanation**
Traces answer "where did this request spend time/fail?"; metrics answer "how many/how fast overall?"; logs give detail. OTel SDK subscribes to your sources/meters and exports them. The app code depends only on the BCL types, not on the OTel packages.

**Code**
```csharp
using System.Diagnostics;
using System.Diagnostics.Metrics;

public static class Telemetry
{
    public const string Name = "Orders";
    public static readonly ActivitySource Source = new(Name);
    public static readonly Meter Meter = new(Name);
    public static readonly Counter<long> OrdersPlaced = Meter.CreateCounter<long>("orders.placed");
}

public sealed class OrderService
{
    public void Place(int id)
    {
        using var activity = Telemetry.Source.StartActivity("PlaceOrder");   // null if nobody listens
        activity?.SetTag("order.id", id);
        Telemetry.OrdersPlaced.Add(1, new KeyValuePair<string, object?>("channel", "web"));
    }
}
```

### 14.2 [Follow-up] How do you set up OpenTelemetry in ASP.NET Core?
**Answer**
Install the OTel hosting and instrumentation packages and call `AddOpenTelemetry()` with `WithTracing`/`WithMetrics`, adding ASP.NET Core and HttpClient instrumentation, your custom `AddSource`/`AddMeter` names, and an exporter (OTLP to a collector). Set the `service.name` resource.

**Explanation**
The custom source/meter names must be registered with `AddSource`/`AddMeter`, otherwise the SDK ignores them (and `StartActivity` returns null). OTLP endpoint is usually configured via env vars (`OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_SERVICE_NAME`), which keeps code environment-agnostic.

**Code**
```csharp
builder.Services.AddOpenTelemetry()
    .ConfigureResource(r => r.AddService(serviceName: "orders-api", serviceVersion: "1.0.0"))
    .WithTracing(t => t
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddSource(Telemetry.Name)
        .SetSampler(new TraceIdRatioBasedSampler(0.25))
        .AddOtlpExporter())
    .WithMetrics(m => m
        .AddAspNetCoreInstrumentation()
        .AddRuntimeInstrumentation()                // OpenTelemetry.Instrumentation.Runtime
        .AddMeter(Telemetry.Name)
        .AddOtlpExporter());
builder.Logging.AddOpenTelemetry(o => { o.IncludeScopes = true; o.AddOtlpExporter(); });
```

### 14.3 [Medium] How does trace context propagate across services, and how do you add custom spans?
**Answer**
The caller's `Activity` ID is sent in the `traceparent` header by `HttpClient` instrumentation; the callee's ASP.NET Core instrumentation reads it and makes its server span a child, sharing the `TraceId`. Add custom spans with `ActivitySource.StartActivity`, set tags/events, and set status on errors.

**Explanation**
Child `Activity` created under `Activity.Current` automatically; `ActivityKind` (Server/Client/Internal/Producer/Consumer) describes role. For messaging (queues), propagate context by injecting `traceparent` into message headers manually and starting a Consumer activity with the parent context.

**Code**
```csharp
public async Task<decimal> PriceAsync(int id, CancellationToken ct)
{
    using var activity = Telemetry.Source.StartActivity("CalculatePrice", ActivityKind.Internal);
    activity?.SetTag("order.id", id);
    try
    {
        await Task.Delay(10, ct);
        return 42m;
    }
    catch (Exception ex)
    {
        activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
        activity?.AddEvent(new ActivityEvent("exception"));
        throw;
    }
}
```

### 14.4 [Hard] Explain metric instruments, `IMeterFactory`, and cardinality pitfalls.
**Answer**
`Counter<T>` is monotonic (requests served); `UpDownCounter<T>` goes up/down (queue depth); `Histogram<T>` records distributions (latency); observable instruments report via callbacks (gauges). In .NET 8 use `IMeterFactory` from DI to create meters scoped to the container (testable, avoids static globals). Tags create time series per unique combination, so never use unbounded values (user id, raw URL) as tags.

**Explanation**
Each unique tag set = one series stored by the backend → cost explosion. The SDK enforces a cardinality limit (default 2000 per metric) and overflows to an `otel.metric.overflow` series. `IMeterFactory` meters are disposed with the container; test them using `MetricCollector<T>` (`Microsoft.Extensions.Diagnostics.Testing`). Prefer unit-suffix naming (`http.server.duration`, units via `unit:` parameter).

**Code**
```csharp
public sealed class CheckoutMetrics
{
    private readonly Counter<long> _checkouts;
    private readonly Histogram<double> _duration;

    public CheckoutMetrics(IMeterFactory meterFactory)
    {
        var meter = meterFactory.Create("Shop.Checkout");
        _checkouts = meter.CreateCounter<long>("checkout.count");
        _duration = meter.CreateHistogram<double>("checkout.duration", unit: "ms");
    }

    public void Record(string paymentMethod, double ms)
    {
        _checkouts.Add(1, new KeyValuePair<string, object?>("payment", paymentMethod)); // low cardinality OK
        _duration.Record(ms);
    }
}
// builder.Services.AddSingleton<CheckoutMetrics>();  // IMeterFactory is registered by the host (AddMetrics)
// OTel: .WithMetrics(m => m.AddMeter("Shop.Checkout"))
```

**Gotchas / follow-ups**
- `IMeterFactory` is available through `services.AddMetrics()` (included by the default host builders).
- Observable gauges' callbacks run on the collection thread — keep them fast and thread-safe.

### 14.5 [Tricky] Your custom spans never appear in Jaeger and `StartActivity` returns null. What do you check?
**Answer**
First that the source name is registered via `AddSource("<exact name>")` (names are exact or wildcard patterns), then that tracing is enabled and a sampler isn't dropping everything, then exporter connectivity/endpoint, and that no `using` disposed the activity before child work. A null `Activity` just means no listener is subscribed.

**Explanation**
Common causes: typo in source name; `AddSource` called on metrics instead of tracing; sampler `AlwaysOff`/ratio too low or parent-based sampler inheriting an unsampled parent; exporter protocol mismatch (gRPC 4317 vs HTTP/protobuf 4318); app exits before batch export (flush on shutdown — dispose the provider); `Activity.DefaultIdFormat` not W3C in very old setups. Verify locally with a console exporter or an `ActivityListener`.

**Code**
```csharp
using var listener = new ActivityListener
{
    ShouldListenTo = s => s.Name == "Orders",
    Sample = (ref ActivityCreationOptions<ActivityContext> _) => ActivitySamplingResult.AllDataAndRecorded,
    ActivityStopped = a => Console.WriteLine($"stopped {a.OperationName}")
};
ActivitySource.AddActivityListener(listener);

using (var a = new ActivitySource("Orders").StartActivity("Demo")) { }
// Output: stopped Demo
```

**Interview traps**
- Forgetting `AddSource`/`AddMeter` for custom telemetry.
- High-cardinality metric tags.
- Believing OTel replaces logging — it complements it.
- Assuming sampling happens at the backend (head sampling occurs in the SDK).
- Not flushing on shutdown for short-lived processes.

**Cheat sheet**
- Span = `Activity` (`ActivitySource`); metric = `Meter` instrument.
- `AddSource`/`AddMeter` must match names.
- `traceparent` propagates context; `HttpClient` & ASP.NET Core instrumentation do it.
- Counter / UpDownCounter / Histogram / ObservableGauge.
- Tags must be low-cardinality; `IMeterFactory` in .NET 8.

---

## Topic 15: Health Checks (Liveness / Readiness)

**Study notes**

- Package: built into ASP.NET Core (`Microsoft.Extensions.Diagnostics.HealthChecks`). Register: `builder.Services.AddHealthChecks()`; map: `app.MapHealthChecks("/health")`.
- Results: `Healthy`, `Degraded`, `Unhealthy`. Default HTTP: Healthy/Degraded → **200**, Unhealthy → **503** (configurable via `ResultStatusCodes`).
- Implement `IHealthCheck.CheckHealthAsync(HealthCheckContext, CancellationToken)`; register with `AddCheck<T>(name, failureStatus, tags)`.
- **Tags + Predicate** let you split endpoints:

| Probe | Question | Should check | Failure action (Kubernetes) |
|---|---|---|---|
| **Liveness** | Is the process alive (not deadlocked)? | Nothing external (`Predicate = _ => false`) | Restart container |
| **Readiness** | Can it serve traffic now? | DB, cache, downstream deps (tag `ready`) | Remove from load balancer, don't restart |
| **Startup** | Has it finished starting? | Warm-up/migrations | Delay other probes |

- Packages: `Microsoft.Extensions.Diagnostics.HealthChecks.EntityFrameworkCore` (`AddDbContextCheck<T>()`), AspNetCore.HealthChecks.* community packages (SQL Server, Redis, Azure...).
- `IHealthCheckPublisher` pushes results periodically (to monitoring) instead of being polled.
- Secure/limit detailed output (`ResponseWriter`) — don't expose internals publicly.

### 15.1 [Easy] What are health checks and how do you add one?
**Answer**
They're endpoints reporting whether the app (and dependencies) are healthy, used by load balancers, orchestrators and monitors. Add with `AddHealthChecks()` and expose with `MapHealthChecks("/health")`; it returns 200 for healthy and 503 for unhealthy by default.

**Explanation**
The default endpoint runs every registered check and returns the worst status as plain text (`Healthy`/`Degraded`/`Unhealthy`). Checks run on demand per request, with the configured timeouts and in parallel.

**Code**
```csharp
builder.Services.AddHealthChecks();
var app = builder.Build();
app.MapHealthChecks("/health");
// GET /health => 200 "Healthy"
```

### 15.2 [Follow-up] Write a custom health check and register it with tags.
**Answer**
Implement `IHealthCheck`, return `HealthCheckResult.Healthy/Degraded/Unhealthy` (optionally with description/data/exception), and register with `AddCheck<T>("name", tags: new[] { "ready" })`. Constructor injection works (checks are created per execution; use scoped services carefully — the check is created from a scope).

**Explanation**
`failureStatus` applies when the check throws. Checks with `Degraded` are useful for "works but slow/partial". Add timeouts to external calls so health endpoint doesn't hang.

**Code**
```csharp
public sealed class DiskSpaceCheck : IHealthCheck
{
    public Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext context, CancellationToken ct = default)
    {
        var freeGb = new DriveInfo(Path.GetPathRoot(Environment.CurrentDirectory)!).AvailableFreeSpace / 1_000_000_000d;
        return Task.FromResult(freeGb switch
        {
            < 1 => HealthCheckResult.Unhealthy($"Only {freeGb:F1} GB free"),
            < 5 => HealthCheckResult.Degraded($"Low disk: {freeGb:F1} GB"),
            _ => HealthCheckResult.Healthy("Disk OK")
        });
    }
}

builder.Services.AddHealthChecks()
    .AddCheck<DiskSpaceCheck>("disk", failureStatus: HealthStatus.Degraded, tags: new[] { "ready" });
```

### 15.3 [Medium] How do you separate liveness and readiness endpoints, and why does it matter?
**Answer**
Tag dependency checks as `ready`, map `/health/ready` with a predicate selecting that tag, and map `/health/live` with `Predicate = _ => false` so it runs no checks and only proves the process responds. Liveness failing restarts the pod; readiness failing only removes it from rotation — mixing them causes restart storms when a dependency (e.g. DB) is down.

**Explanation**
If liveness checked the DB, a DB outage would make Kubernetes restart every pod repeatedly, worsening the outage. Readiness lets the app stay up and recover when the DB returns. Add a startup probe for slow initialisation.

**Code**
```csharp
builder.Services.AddHealthChecks()
    .AddDbContextCheck<AppDbContext>("sql", tags: new[] { "ready" })   // EF Core health check package
    .AddCheck<DiskSpaceCheck>("disk", tags: new[] { "ready" });

app.MapHealthChecks("/health/live", new HealthCheckOptions { Predicate = _ => false });
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = r => r.Tags.Contains("ready"),
    ResultStatusCodes =
    {
        [HealthStatus.Healthy] = StatusCodes.Status200OK,
        [HealthStatus.Degraded] = StatusCodes.Status200OK,
        [HealthStatus.Unhealthy] = StatusCodes.Status503ServiceUnavailable
    }
});
```

### 15.4 [Hard] Return a JSON report, and avoid health checks causing load or leaking information.
**Answer**
Set `ResponseWriter` to emit JSON (status, per-check results, durations) for internal consumers, restrict the endpoint (`RequireHost`, `RequireAuthorization`, network policy), cache or throttle expensive checks, use timeouts, and avoid exposing connection strings or exception details publicly.

**Explanation**
Checks execute on each probe: orchestrators probe every few seconds across many pods, so expensive DB checks add load — use lightweight queries (`SELECT 1`), the check's own caching, or an `IHealthCheckPublisher` that evaluates on a schedule. Add rate/size control; don't include `Exception` text in responses for external audiences.

**Code**
```csharp
using System.Text.Json;

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = r => r.Tags.Contains("ready"),
    ResponseWriter = async (ctx, report) =>
    {
        ctx.Response.ContentType = "application/json";
        var body = new
        {
            status = report.Status.ToString(),
            totalMs = report.TotalDuration.TotalMilliseconds,
            checks = report.Entries.Select(e => new
            {
                name = e.Key,
                status = e.Value.Status.ToString(),
                ms = e.Value.Duration.TotalMilliseconds
            })
        };
        await ctx.Response.WriteAsync(JsonSerializer.Serialize(body));
    }
}).RequireHost("*:5001");     // e.g. only on the internal management port
```

### 15.5 [Tricky] A scoped service is injected into your `IHealthCheck` and the app fails to start / the check misbehaves. Why?
**Answer**
`AddCheck<T>` doesn't register `T` as a service; it creates the check with `ActivatorUtilities` each time from a scope that the health check service opens per run, so scoped dependencies are valid *within the run*. If you instead register the check yourself as a singleton or run it from a `BackgroundService`, the scoped dependency becomes captive and validation fails.

**Explanation**
`DefaultHealthCheckService` creates a scope per `CheckHealthAsync` call and resolves each registration's check from it. Problems arise when people register `T` manually as singleton, cache the `HealthCheckResult`, or reuse a `DbContext` across threads (checks run in parallel, so don't share one scoped context between checks). Another gotcha: `AddCheck<T>` resolves via `ActivatorUtilities.GetServiceOrCreateInstance`, so `T` need not be registered (and if you register it yourself, that lifetime is used).

**Code**
```csharp
public sealed class DbPingCheck(AppDbContext db) : IHealthCheck       // scoped dep is fine here
{
    public async Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext context, CancellationToken ct = default)
        => await db.Database.CanConnectAsync(ct) ? HealthCheckResult.Healthy() : HealthCheckResult.Unhealthy("DB down");
}
builder.Services.AddHealthChecks().AddCheck<DbPingCheck>("db", tags: new[] { "ready" });
// builder.Services.AddSingleton<DbPingCheck>();   // would make the scoped dependency captive - don't
```

**Interview traps**
- Putting dependency checks on the liveness probe (restart storms).
- Expensive checks on every probe.
- Exposing health details publicly.
- Reporting only 200 even when a critical dependency is down (probe useless).
- Running health checks that mutate state.

**Cheat sheet**
- `AddHealthChecks()` + `MapHealthChecks(path)`.
- Healthy/Degraded → 200, Unhealthy → 503 by default.
- Liveness = process only (`Predicate = _ => false`); readiness = tagged dependencies.
- `AddDbContextCheck<T>()` for EF Core.
- Lightweight, timed, secured endpoints.

---

## Topic 16: Feature Management & Localization Basics

**Study notes**

**Feature flags — `Microsoft.FeatureManagement.AspNetCore`:**
- Config (default section `FeatureManagement`):
```json
"FeatureManagement": {
  "NewCheckout": true,
  "BetaUi": { "EnabledFor": [ { "Name": "Percentage", "Parameters": { "Value": 25 } } ] }
}
```
- Register: `builder.Services.AddFeatureManagement();` (add `.AddFeatureFilter<T>()` for custom filters; built-ins: `Percentage`, `TimeWindow`, `Targeting`, `ContextualTargeting`).
- Use: `IFeatureManager.IsEnabledAsync("NewCheckout")`, `[FeatureGate("NewCheckout")]` on controllers/actions, `<feature name="...">` tag helper, middleware branching `UseMiddlewareForFeature<T>`. `IFeatureManagerSnapshot` caches results per request for consistency.
- Dynamic refresh: store flags in **Azure App Configuration** (feature flag support + refresh middleware) rather than static files.
- Flag types/hygiene: release toggles (short-lived), ops toggles (kill switches), experiment toggles, permission toggles. Remove stale flags; test both paths; default to *off*.

**Localization (ASP.NET Core):**
- **Globalization** (formatting/culture) vs **Localization** (translated resources). `CurrentCulture` = formats numbers/dates; `CurrentUICulture` = resource lookup language.
- Setup: `AddLocalization(o => o.ResourcesPath = "Resources")`; inject `IStringLocalizer<T>`; resx files named after the type (`Resources/Controllers.HomeController.fr.resx`). `IHtmlLocalizer<T>`, `IViewLocalizer` for views.
- Middleware: `UseRequestLocalization(...)`; default providers in order: **QueryString** (`?culture=fr&ui-culture=fr`), **Cookie** (`.AspNetCore.Culture`), **Accept-Language** header.
- Fallback: `fr-CA` → `fr` → invariant/neutral resources → key name returned (`localizer["Key"].ResourceNotFound == true`).

### 16.1 [Easy] What is a feature flag and how do you use the .NET feature management library?
**Answer**
A feature flag lets you turn behaviour on/off at runtime without redeploying. With `Microsoft.FeatureManagement`, define flags in the `FeatureManagement` config section, call `AddFeatureManagement()`, and check `IFeatureManager.IsEnabledAsync("Name")` or decorate endpoints with `[FeatureGate("Name")]`.

**Explanation**
It decouples deployment from release: ship dark, enable gradually, kill switch if problems. Flags can be simple booleans or have filters (percentage, time window, targeting by user/group).

**Code**
```csharp
using Microsoft.FeatureManagement;

builder.Services.AddFeatureManagement();   // reads "FeatureManagement" config section

app.MapGet("/checkout", async (IFeatureManager features) =>
    await features.IsEnabledAsync("NewCheckout") ? "new checkout" : "old checkout");
// appsettings.json: "FeatureManagement": { "NewCheckout": true }
// Output (flag true): new checkout
```

### 16.2 [Follow-up] How do feature filters work — percentage rollout and custom filters?
**Answer**
A flag with `EnabledFor` filters is on if *any* filter evaluates true. Built-in `Percentage` enables for a configured % of evaluations; `TimeWindow` between dates; `Targeting` for specific users/groups with default rollout. Custom filters implement `IFeatureFilter` and are registered with `AddFeatureFilter<T>()`.

**Explanation**
Percentage filter is random per evaluation (not sticky); use `Targeting` with a user id for consistent assignment, or `IFeatureManagerSnapshot` for per-request consistency. Filter alias defaults to the class name minus "Filter" suffix, or set with `[FilterAlias]`.

**Code**
```csharp
[FilterAlias("Browser")]
public sealed class BrowserFilter(IHttpContextAccessor http) : IFeatureFilter
{
    public Task<bool> EvaluateAsync(FeatureFilterEvaluationContext ctx)
    {
        var allowed = ctx.Parameters.GetSection("Allowed").Get<string[]>() ?? [];
        var ua = http.HttpContext?.Request.Headers.UserAgent.ToString() ?? "";
        return Task.FromResult(allowed.Any(b => ua.Contains(b, StringComparison.OrdinalIgnoreCase)));
    }
}

builder.Services.AddHttpContextAccessor();
builder.Services.AddFeatureManagement().AddFeatureFilter<BrowserFilter>();
// "BetaUi": { "EnabledFor": [ { "Name": "Browser", "Parameters": { "Allowed": [ "Edg", "Chrome" ] } } ] }
```

### 16.3 [Medium] Using flags safely: consistency, cleanup and testing.
**Answer**
Inject `IFeatureManagerSnapshot` so a flag evaluates once per request (consistent behaviour within it), keep flags short-lived and delete them after rollout, default to off, and test both code paths. Keep flag checks at boundaries (composition/strategy) rather than scattered through domain code.

**Explanation**
Percentage flags may give different answers within one request if re-evaluated, producing inconsistent UI/logic. Stale flags create dead code and combinatorial test explosion. Pattern: choose implementation by flag at one place (e.g. DI factory/strategy) — see keyed services in Topic 6.

**Code**
```csharp
public sealed class CheckoutService(IFeatureManagerSnapshot features, INewCheckout newFlow, ILegacyCheckout oldFlow)
{
    public async Task<string> RunAsync()
        => await features.IsEnabledAsync("NewCheckout") ? newFlow.Run() : oldFlow.Run();
}
public interface INewCheckout { string Run(); }
public interface ILegacyCheckout { string Run(); }
```

### 16.4 [Medium] How does localization work in ASP.NET Core (middleware, resx, culture providers)?
**Answer**
Register `AddLocalization`, configure `UseRequestLocalization` with supported cultures, and inject `IStringLocalizer<T>` to read translated strings from `.resx` files. The middleware sets `CurrentCulture`/`CurrentUICulture` per request from query string, cookie or `Accept-Language` (in that order by default).

**Explanation**
Resource lookup is hierarchical with fallback to the neutral culture. If a key is missing, the indexer returns the key itself and `ResourceNotFound` is true — handy for spotting gaps. `CurrentCulture` affects `ToString("C")`, dates; `CurrentUICulture` affects resource files. Place `UseRequestLocalization` early (before MVC/endpoints that need culture).

**Code**
```csharp
builder.Services.AddLocalization(o => o.ResourcesPath = "Resources");
var app = builder.Build();

var supported = new[] { "en-US", "fr-FR", "de-DE" };
app.UseRequestLocalization(new RequestLocalizationOptions()
    .SetDefaultCulture("en-US")
    .AddSupportedCultures(supported)
    .AddSupportedUICultures(supported));

// Resources/Program.fr.resx has key "Hello" = "Bonjour"  (marker class Program or a SharedResource class)
app.MapGet("/hello", (IStringLocalizer<Program> l) => l["Hello"].Value);
// GET /hello?culture=fr-FR&ui-culture=fr-FR  => Bonjour
```

### 16.5 [Hard] Culture-sensitive gotchas: formatting, parsing and server code.
**Answer**
`ToString`/`Parse` use `CurrentCulture` by default, so the same code outputs `1,234.50` or `1.234,50` depending on the request. For machine-readable data (JSON, logs, DB parameters, cache keys) use `CultureInfo.InvariantCulture`; for user-facing output rely on the request culture. Also remember sorting/comparison (`StringComparison.Ordinal`) and the Turkish-I problem.

**Explanation**
Bugs: parsing `"1,5"` as decimal under `en-US` gives 15; storing culture-formatted dates in files; `ToLower()` on `"I"` in `tr-TR` yields dotless ı (use `ToLowerInvariant()` for identifiers). Also `InvariantGlobalization` mode (containers) disables ICU culture data — all cultures behave invariant; ensure it's off if you need localized formatting. Async flow: culture is stored in `ExecutionContext` so it flows across `await`, but thread-pool background jobs started outside the request get the default culture.

**Code**
```csharp
using System.Globalization;

var n = 1234.5m;
Console.WriteLine(n.ToString("N2", new CultureInfo("en-US")));   // Output: 1,234.50
Console.WriteLine(n.ToString("N2", new CultureInfo("de-DE")));   // Output: 1.234,50
Console.WriteLine(n.ToString("N2", CultureInfo.InvariantCulture)); // Output: 1,234.50

Console.WriteLine(decimal.Parse("1,5", new CultureInfo("de-DE")) == 1.5m);  // Output: True (comma is the decimal separator)
Console.WriteLine(decimal.Parse("1,5", CultureInfo.InvariantCulture)); // Output: 15  (comma is a thousands separator)
```

**Gotchas / follow-ups**
- Set `CultureInfo.DefaultThreadCurrentCulture` for non-request background work if needed.
- Time zones are separate from culture: store UTC, convert at the edge.

### 16.6 [Tricky] Your `IStringLocalizer<T>` always returns the key and the flag/culture seems ignored. What do you check?
**Answer**
Check resx file name/location versus `ResourcesPath` and the generic type's namespace/full name, that `.resx` is embedded (`EmbeddedResource`/ satellite assembly built), that the culture is in the supported list and `UseRequestLocalization` runs before the endpoint, and that the `ui-culture` (not just `culture`) is set. A returned key with `ResourceNotFound = true` confirms lookup failed.

**Explanation**
Naming rules: with `ResourcesPath = "Resources"` and type `MyApp.Controllers.HomeController`, the file is `Resources/Controllers.HomeController.fr.resx` (root namespace stripped, dots instead of folders) or `Resources/Controllers/HomeController.fr.resx`. A generic marker class (`SharedResource`) avoids per-type files. In Docker `InvariantGlobalization=true` and missing ICU also silently breaks cultures.

**Code**
```csharp
public sealed class SharedResource { }     // marker => Resources/SharedResource.fr.resx

app.MapGet("/check", (IStringLocalizer<SharedResource> l) =>
{
    var s = l["Welcome"];
    return new { s.Value, s.ResourceNotFound, SearchedLocation = s.SearchedLocation };
});
```

**Interview traps**
- Percentage flags assumed sticky per user.
- Leaving old flags in code forever.
- Setting only `culture` and not `ui-culture`.
- Formatting machine data with the current culture.
- Docker image with `InvariantGlobalization` while expecting localized dates/numbers.

**Cheat sheet**
- Flags: `AddFeatureManagement()`, `IFeatureManager`/`IFeatureManagerSnapshot`, `[FeatureGate]`, filters (Percentage/TimeWindow/Targeting).
- Dynamic flags: Azure App Configuration; remove stale flags.
- Localization: `AddLocalization` + `UseRequestLocalization` + `IStringLocalizer<T>` + resx.
- Culture providers order: QueryString → Cookie → Accept-Language.
- `InvariantCulture` for machine data; `CurrentUICulture` for resources.



