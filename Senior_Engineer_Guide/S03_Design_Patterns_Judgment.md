# S03 - Design Patterns: Judgment Questions for Senior .NET / Azure / SQL Server Engineers

**How to use:** Read the study notes, then answer each ladder question aloud before reading the strong answer. Seniors are judged on *reasoning, trade-offs and when NOT to use a pattern*, not on definitions.

## Table of Contents
1. Why Patterns Exist and How Seniors Use Them
2. SOLID in Practice (and When It Hurts)
3. DI, IoC, Dependency Inversion, Service Locator
4. Repository, Unit of Work and Specification over EF Core
5. Mediator / MediatR and In-Process CQRS
6. Factory, Abstract Factory, Builder
7. Strategy and Policy-Based Design
8. Decorator, Pipeline Behaviors, Chain of Responsibility
9. Observer, Events, Pub/Sub and Domain Events
10. Adapter, Facade, Anti-Corruption Layer, Proxy
11. Singleton, Lifetimes, Options Pattern, Null Object
12. Result Pattern vs Exceptions
13. State Pattern, State Machines, Template Method
14. Cloud Resilience Patterns (Retry, Circuit Breaker, Bulkhead, Timeout, Throttling, Rate Limiting)
15. Cloud Messaging and Load Patterns
16. Cloud Data and Distributed-Consistency Patterns
17. Cloud Topology, Edge and Migration Patterns
18. DDD Tactical Patterns
19. Clean / Hexagonal / Onion, Layered vs Vertical Slice
20. Composition over Inheritance, Immutability, API Design Patterns
21. Refactoring Toward Patterns and Recognising Anti-Patterns
22. Scenario Round and the 60-Second Pattern Explanation

---

## Topic 1: Why Patterns Exist and How Seniors Use Them

**Study notes**
- A pattern is a *named, reusable solution to a recurring problem in a context*, with known consequences (GoF, POSA, Azure Architecture Center cloud patterns).
- Value = (1) shared vocabulary, (2) captured trade-offs, (3) proven starting shapes. Not a goal in itself.
- Problem-first: identify the force (change axis, coupling, failure mode, scale) -> then see if a pattern fits.
- Pattern-first smell: "we need a Factory here" with no second implementation or varying construction.
- YAGNI: abstraction has a carrying cost (indirection, navigation, onboarding). Pay it when change is *evidenced*, not imagined.
- Rule of three: tolerate duplication twice; abstract on the third real case.
- Language features replace patterns: delegates/lambdas (Strategy, Command), `IEnumerable`/LINQ (Iterator), `event` (Observer), DI container (Factory/Singleton), records (Value Object).

```
Problem/Force --> candidate pattern --> consequences OK? --> apply (smallest form)
      ^                                        |no
      +-------------- reconsider --------------+
```

### 1.1 [Easy] Why do design patterns exist, and what do they give a team?
**What the interviewer is really testing**: Whether you see patterns as communication/tradeoff tools rather than rules.
**Typical mid-level answer**: "They are reusable solutions to common problems from the Gang of Four." Correct but misses vocabulary, consequences, and context.
**Strong senior answer**: Patterns compress design knowledge into a name. Saying "this is a Decorator around the payment client adding retries" conveys structure, intent and consequences in one sentence, which speeds code review and design discussion. Each pattern also documents trade-offs: Decorator gives open/closed extension but creates many small types and order-sensitive stacks. They are *context-bound*: a pattern that is idiomatic in Java 1.4 may be a lambda in C#. I use them as a vocabulary and a checklist of consequences, not as a template to instantiate. The test is whether the team understands the code faster with the name than without it.
**Follow-up traps**
- *"Are patterns the same as best practices?"* No; they are context-dependent solutions and each has cons.
- *"Do patterns apply to cloud?"* Yes, e.g. Azure Architecture Center's cloud design patterns have the same problem/context/solution/trade-off shape.

### 1.2 [Follow-up] How do you decide between pattern-first and problem-first design?
**What the interviewer is really testing**: Discipline against resume-driven design.
**Typical mid-level answer**: "I look at the requirements and pick a pattern." It skips how you detect the force.
**Strong senior answer**: I start by naming the force: what is likely to change, what fails, what is hard to test, where do two teams collide. Only then do I ask whether a known pattern addresses that force and whether its cost is justified now. Example: payment provider might change *and* we have a signed contract for a second provider next quarter -> an `IPaymentGateway` abstraction is evidenced. If it is a vague "might switch someday", I write the concrete class behind a thin seam and refactor when it becomes real; refactoring toward a pattern is cheap in a well-tested codebase. I also prefer the smallest form of the pattern (a `Func<>` over a strategy hierarchy). When I do apply it I write down the *why* in an ADR so the next person can judge whether the force still exists.
**Follow-up traps**
- *"Isn't waiting more expensive?"* Only for hard-to-reverse decisions (public contracts, data schema, messaging formats); for internal code, reversal is cheap.
- *"How do you know the force is real?"* History of change in the code (git log), a committed roadmap item, or a failing test/incident.

### 1.3 [Medium] Where does YAGNI conflict with "design for extensibility"?
**What the interviewer is really testing**: Calibrating speculative generality vs reversibility.
**Typical mid-level answer**: "YAGNI says don't build it; but I'd add interfaces just in case." Falls short: no cost model.
**Strong senior answer**: Separate *cheap-to-change* from *expensive-to-change*. Internal class structure is cheap; published API shapes, DB schemas, message contracts and security boundaries are expensive. Extensibility belongs on the expensive side: version the contract, keep messages additive, keep a stable seam. For internal code I defer abstractions until a second use case exists. Speculative generality has real costs: every interface needs a mock, an extra file, a DI registration, and readers must follow indirection to find real behavior. A single-implementation interface that mirrors a class one-to-one (`IFooService` / `FooService`) is usually noise unless it is a seam for a test double at a process boundary. YAGNI is not "never abstract"; it is "don't pay before the benefit is evidenced".
**Follow-up traps**
- *"Interfaces for unit testing?"* Mock at I/O boundaries (HTTP, DB, queue, clock); don't mock your own pure logic.
- *"What about libraries?"* Public libraries need more upfront design because breaking changes are costly.

### 1.4 [Hard] You inherit a codebase full of patterns used badly. How do you discuss it without a blame war?
**What the interviewer is really testing**: Leadership, empathy, evidence-based persuasion.
**Typical mid-level answer**: "I'd tell them to remove unnecessary patterns." Too confrontational, no plan.
**Strong senior answer**: I first learn the original context; the author may have had a force that no longer exists. Then I gather evidence: how many lines of ceremony per feature, onboarding time, bug locations, change-failure data. I propose a measurable pain ("adding a field touches 9 files") rather than "this is over-engineered". Fix incrementally: collapse the abstraction the next time a feature touches it (boy-scout rule), backed by tests, not in a big-bang rewrite. I agree on lightweight guidelines (e.g., "new interface needs 2 implementations or a boundary reason") in an ADR. This keeps decisions team-owned and reversible.
**Follow-up traps**
- *"What if the author is senior and defends it?"* Ask which change scenario it protects; test it with a real upcoming feature.
- *"Rewrite?"* Rarely; strangle incrementally.

### 1.5 [Tricky] "Patterns are just workarounds for missing language features." Agree?
**What the interviewer is really testing**: Nuance; knowing when C# makes a pattern trivial.
**Typical mid-level answer**: "No, patterns are always useful." Dismisses a half-true statement.
**Strong senior answer**: Partly true (Norvig's observation: many GoF patterns vanish in dynamic/functional languages). In C#, delegates make Strategy/Command/Template Method a parameter; `event`/`IObservable<T>` encode Observer; `IEnumerable` + `yield` is Iterator; records give value semantics; DI containers absorb Singleton and many Factories; `using`/`IDisposable` is a resource-scope idiom. But structural/architectural patterns (Adapter at a boundary, Saga, Circuit Breaker, Outbox) address design problems independent of language. So I use the *intent* of the pattern and implement it with the lightest idiom available. Example: instead of a `IDiscountStrategy` hierarchy for two rules, pass `Func<Order, decimal>`; when rules multiply, have names, config and tests, graduate to classes.
**Follow-up traps**
- *"Is delegate Strategy harder to discover?"* Yes, less self-documenting; graduate when rules need identity/config.
- *"Which GoF patterns do you rarely see in C#?"* Iterator hand-rolled, Singleton hand-rolled, Prototype, Flyweight (outside perf code).

**Red flags in answers**
- Reciting GoF names with no example of a force or cost.
- "Always program to an interface" with no boundary justification.
- Treating pattern count as a quality measure.

**Cheat sheet**
- Problem/force first, pattern second, smallest form third.
- Abstract on evidenced change, not imagined change; rule of three.
- Expensive-to-change things (contracts, schema) deserve upfront design.
- C# idioms (delegates, records, DI) often replace classic implementations.
- Record the "why" in an ADR so the pattern can be retired later.

---

## Topic 2: SOLID in Practice (and When It Hurts)

**Study notes**
- **S**RP: one *reason to change* (one actor/stakeholder), not "one method".
- **O**CP: extend behavior without editing stable code; usually via composition/polymorphism at *known* variation points.
- **L**SP: subtypes must honor the base contract (pre/post-conditions, no stronger preconditions, no `NotSupportedException`).
- **I**SP: clients shouldn't depend on methods they don't use; prefer small role interfaces.
- **D**IP: high-level policy depends on abstractions owned by the high level; details depend on them.
- Dogmatic SOLID -> class explosion, one-method interfaces everywhere, navigation hell.

```
Without DIP:  OrderService --> SqlOrderRepo
With DIP:     OrderService --> IOrderStore (owned by domain) <-- SqlOrderStore
```

### 2.1 [Easy] Explain SRP with a real example.
**What the interviewer is really testing**: Correct definition (reason to change) rather than "does one thing".
**Typical mid-level answer**: "A class should do only one thing." Vague and leads to tiny classes.
**Strong senior answer**: SRP means a module should be responsible to one stakeholder, so changes requested by one actor don't ripple into another's code. Example: an `Invoice` class that calculates tax (finance), renders PDF (design/marketing) and persists itself (DBA/platform) has three reasons to change; a tax rule change risks the PDF layout. Splitting along those *axes of change* helps. Splitting `Invoice.Total()` into `TotalCalculator`, `TotalFormatter` when they always change together is the opposite error. I judge by looking at git history: do unrelated changes keep touching the same file, or do related changes require edits in 8 files?
**Follow-up traps**
- *"Is a 2000-line class always an SRP violation?"* Not strictly, but it usually has multiple reasons to change; check cohesion.
- *"Controller with 10 actions?"* Fine if cohesive on one resource; thin controllers delegate.

### 2.2 [Follow-up] When does OCP help, and when is it a trap?
**What the interviewer is really testing**: Knowing OCP needs *predicted* variation.
**Typical mid-level answer**: "Use interfaces so you don't modify existing code." Ignores guessing cost.
**Strong senior answer**: OCP pays off where variation is real and recurring: payment methods, tax rules per region, export formats, validation rules. There, a registry of strategies resolved by DI (`IEnumerable<IExportFormatter>`) lets you add a class without touching the caller. It is a trap when applied to everything: you can't be closed against all changes, only those you anticipated, and a wrong guess yields an abstraction that fits neither case. I apply OCP *after* the second or third variation arrives, when I can see the real axis; the first change I edit directly. A `switch` with 3 stable cases is often clearer than a strategy hierarchy.
**Follow-up traps**
- *"Is a switch an OCP violation?"* Only if it grows with each feature and is duplicated across the code.
- *"C# 8+ switch expressions?"* Great for closed sets, and the compiler warns on non-exhaustive enums.

### 2.3 [Medium] Give an LSP violation you have seen in .NET.
**What the interviewer is really testing**: Contract thinking.
**Typical mid-level answer**: "Square/Rectangle." Textbook only.
**Strong senior answer**: In .NET: `ReadOnlyCollection` or arrays implementing `IList<T>` throw `NotSupportedException` on `Add`, so code written against `IList<T>` breaks (the framework added `IReadOnlyList<T>` to fix this). In app code: a `CachedRepository : IRepository` that returns stale data or ignores `CancellationToken`, or a `NullEmailSender` that throws. A smell is `if (x is DerivedType)` in callers, or overrides that throw. Fix by splitting the interface (ISP) so the abstraction only promises what all implementers can honour, or by making the contract explicit (documented preconditions/exceptions). Contract tests run against all implementations catch LSP regressions.
**Follow-up traps**
- *"Is throwing in an override always wrong?"* Throwing a stricter exception than the contract allows is; documented failure modes are fine.
- *"How to test LSP?"* Shared abstract test fixture executed against each implementation.

### 2.4 [Hard] Where has dogmatic SOLID hurt you, and what did you do?
**What the interviewer is really testing**: Pragmatism and scars.
**Typical mid-level answer**: "SOLID is always good; I haven't seen it hurt." Red flag.
**Strong senior answer**: Typical damage: `IFooService`/`FooService` pairs for every class, 1-method interfaces mirroring concretes, a feature spread across `Controller -> Service -> Manager -> Helper -> Repository` where each layer just forwards. Symptoms: adding a field touches many files, stack traces are deep, tests mock everything and verify call order (testing implementation, not behavior). Fix: collapse forwarding layers, keep interfaces only at real seams (I/O, volatile third parties, multiple implementations), test through the public behavior with integration tests (WebApplicationFactory, Testcontainers). SOLID is a set of heuristics about managing change; when the change they protect against isn't happening, the cost dominates.
**Follow-up traps**
- *"Doesn't that break DIP?"* DIP is about direction of dependency at boundaries, not about wrapping every class.
- *"How to enforce?"* Architecture tests (NetArchTest, ArchUnitNET) for dependency direction, not interface counts.

### 2.5 [Scenario] A reviewer says "this class violates SRP" on your 300-line cohesive class. How do you respond?
**What the interviewer is really testing**: Handling dogma with evidence.
**Typical mid-level answer**: "I'd split it to keep the reviewer happy." Capitulates without analysis.
**Strong senior answer**: I ask which two reasons to change they see. If they can name actors or change drivers (e.g., pricing rules and persistence), I split. If the class is cohesive (all methods use the same state and change together), length alone isn't a violation; I might extract private methods or a nested type for readability. I'd show change history: if the file changed 20 times for one reason, it's fine. Disagreement is resolved by the concrete future change, not by the principle's name. I'd note the outcome in the PR so the debate isn't repeated.
**Follow-up traps**
- *"Metrics?"* Cyclomatic complexity and cohesion (LCOM) are hints, not verdicts.
- *"What if it's untestable?"* That's a stronger signal than length; extract pure logic.

**Red flags in answers**
- SRP defined as "one method/one function".
- Interface per class as default practice.
- Cannot name a case where SOLID went wrong.

**Cheat sheet**
- SRP = one reason/actor to change; judge by change history.
- OCP only at proven variation points.
- LSP: contracts, not inheritance diagrams; watch for throwing overrides.
- ISP: role interfaces; `IReadOnlyList<T>` is the framework example.
- DIP: direction of dependency at boundaries; not "wrap everything".
- Use architecture tests to enforce direction.

---

## Topic 3: DI, IoC, Dependency Inversion, Service Locator

**Study notes**
- **IoC**: a framework calls your code (control flipped). **DI**: a technique for supplying dependencies from outside (constructor injection preferred). **DIP**: a design principle about who owns the abstraction. They are related but distinct.
- Service Locator (`IServiceProvider.GetService` inside business code) hides dependencies, makes tests harder, and fails at runtime instead of composition time.
- Lifetimes in `Microsoft.Extensions.DependencyInjection`: Transient, Scoped (per request/scope), Singleton.
- Captive dependency: a Singleton holding a Scoped/Transient service (e.g., `DbContext`) -> stale/shared state, thread-safety bugs. Scope validation (`ValidateScopes`) is on by default in the Development environment.
- Composition root: one place (Program.cs) wires the graph.

```
Program.cs (composition root)
   |-- registers --> IClock, IOrderStore, ...
Request scope --> Controller --> OrderService --> IOrderStore
```

### 3.1 [Easy] Distinguish IoC, DI and Dependency Inversion.
**What the interviewer is really testing**: Terminology precision.
**Typical mid-level answer**: "They're all the same: injecting interfaces." Conflates three ideas.
**Strong senior answer**: IoC is the broad idea that the framework controls flow and calls you (ASP.NET Core calling your controller, event handlers). DI is one way to achieve inversion of *dependency creation*: the object receives collaborators rather than `new`-ing them. Dependency Inversion Principle is about *dependency direction*: high-level policy shouldn't depend on low-level details; both depend on an abstraction the high-level module owns. You can use a DI container and still violate DIP (the domain referencing `SqlConnection` through an injected `IDbConnection`), and you can satisfy DIP without a container (pass a delegate manually).
**Follow-up traps**
- *"Does a container need to exist?"* No; "pure DI" in Program.cs works for small apps.
- *"Who owns the interface in DIP?"* The consumer (policy) layer, not the implementer.

### 3.2 [Follow-up] Why is Service Locator considered an anti-pattern, and when is it acceptable?
**What the interviewer is really testing**: Judgment, not dogma.
**Typical mid-level answer**: "Because it's bad practice." No reasoning.
**Strong senior answer**: It hides a class's real dependencies: the constructor says nothing, so you discover requirements by reading the body or by runtime failure. It couples business code to the container, complicates unit tests, and defers missing registrations to runtime. Acceptable at the edges: factories that must resolve by key/runtime data (`IServiceProvider` inside a factory or middleware), background services creating a scope per unit of work (`IServiceScopeFactory`), or framework integration points where constructor injection isn't possible. I confine it to the composition/infrastructure layer and wrap it in a named factory. In .NET 8, keyed services (`[FromKeyedServices]`, `AddKeyedSingleton`) removes many reasons to hand-resolve.
**Follow-up traps**
- *"Hosted service needs a DbContext?"* Inject `IServiceScopeFactory`, create a scope per iteration.
- *"Is injecting `IServiceProvider` into a controller OK?"* No, it's a service-locator smell.

### 3.3 [Medium] What is a captive dependency and how do you prevent it?
**What the interviewer is really testing**: Lifetime reasoning.
**Typical mid-level answer**: "Don't inject scoped into singleton." Correct but lacks consequences and tooling.
**Strong senior answer**: A longer-lived service captures a shorter-lived one, so the short-lived instance lives (and is shared across threads/requests) for the app lifetime. With EF Core, a singleton holding a `DbContext` means a non-thread-safe context shared by concurrent requests, a growing change tracker and stale data. Prevent by: scope validation (`ValidateScopes`/`ValidateOnBuild` enabled in Development by default for scopes, and can be turned on explicitly), designing singletons to depend only on singletons or on factories (`IDbContextFactory<T>`, `IServiceScopeFactory`), and code review. Also watch transient `IDisposable` captured by singletons: they are never disposed until the container is.
**Follow-up traps**
- *"Scoped service from singleton by Func<T>?"* It still resolves from the root provider, so it's captive-like; use a scope.
- *"Is transient in singleton okay?"* Often yes if stateless and non-disposable, but it becomes effectively singleton.

### 3.4 [Hard] Constructor has 9 dependencies. What does that tell you and what do you do?
**What the interviewer is really testing**: Using DI as a design smell detector.
**Typical mid-level answer**: "Use property injection or a facade." Hides the problem.
**Strong senior answer**: Constructor injection makes SRP violations visible; 9 collaborators usually means the class has multiple responsibilities or sits at the wrong abstraction level. First I look for clusters that always travel together (e.g., email + SMS + push -> `INotifier`), responsibilities that can be extracted (validation, pricing), or logic that belongs in the domain model. Aggregate Services/Facades are valid when they represent a real concept; "ParameterObject of everything" just hides the problem. Orchestrators (an application handler coordinating several things) may legitimately have 4-6 dependencies. Avoid property injection and service-locator as escape hatches.
**Follow-up traps**
- *"Is MediatR a way out?"* It moves dependencies into handlers; it doesn't reduce responsibility.
- *"Cross-cutting like logging?"* `ILogger<T>` is cheap; consider decorators for others.

### 3.5 [Tricky] Is it OK to register everything as Singleton "for performance"?
**What the interviewer is really testing**: Understanding cost vs correctness.
**Typical mid-level answer**: "Singletons are faster, so yes." Ignores state and threading.
**Strong senior answer**: Object allocation for transient/scoped services is rarely the bottleneck; correctness is. Singletons must be thread-safe and stateless (or protect state), must not depend on scoped services, and make per-request data (user, tenant) tricky. Things that *should* be singletons: `HttpClient` via `IHttpClientFactory` handlers, caches, clients with expensive setup (`ServiceBusClient`, `CosmosClient`, `ConnectionMultiplexer`). Things that should not: `DbContext`, anything holding request/tenant identity. Choose lifetime by state and ownership, then measure before optimizing.
**Follow-up traps**
- *"HttpClient singleton vs factory?"* Factory manages handler lifetime and DNS refresh; typed clients are transient wrappers.
- *"Thread-safety of a singleton with Dictionary?"* Use `ConcurrentDictionary` or locking; better, avoid shared mutable state.

**Red flags in answers**
- Calls `IServiceProvider.GetService` in domain logic.
- Cannot explain captive dependencies or DbContext lifetime.
- Treats DI container as the DIP.

**Cheat sheet**
- IoC (framework calls you) ≠ DI (supplying deps) ≠ DIP (abstraction ownership).
- Constructor injection; composition root in Program.cs.
- Locator OK only at edges (`IServiceScopeFactory` in hosted services).
- Captive dependency: validate scopes; use `IDbContextFactory`.
- Many ctor args = design smell, not DI's fault.
- Lifetime by state/thread-safety, not by speed.

---

## Topic 4: Repository, Unit of Work and Specification over EF Core

**Study notes**
- EF Core's `DbSet<T>` already behaves like a repository, and `DbContext` like a Unit of Work (change tracking, `SaveChanges` as atomic commit).
- A wrapping repository is justified for: aggregate-oriented persistence in DDD, hiding a non-EF store, enforcing invariants (load whole aggregate), or a seam for tests of domain logic.
- Generic `IRepository<T>` with `GetAll()` returning `IQueryable<T>` leaks persistence; without it, it's a thin forwarding layer. Both are widely considered anti-patterns over EF.
- Specification: encapsulate a query predicate (and includes/order) as an object, reusable and composable (e.g., `Expression<Func<T,bool>>`; Ardalis.Specification library is a popular implementation).
- Read side: skip repositories; project directly to DTOs with `Select`.

```
Handler --> DbContext (UoW) --> DbSet<Order> (repository-like)
 vs
Handler --> IOrderRepository --> DbContext --> DB   (extra layer; justify it)
```

### 4.1 [Easy] What are Repository and Unit of Work, and how does EF Core relate?
**What the interviewer is really testing**: Knowing EF already implements them.
**Typical mid-level answer**: "Repository abstracts data access, UoW groups operations." Doesn't mention EF's built-in role.
**Strong senior answer**: Repository presents a collection-like interface to aggregates; Unit of Work tracks changes and commits them as one transaction. `DbSet<T>` is the collection-like repository, and `DbContext` tracks entities and commits via `SaveChangesAsync` in a single transaction by default. So wrapping them adds an abstraction over an abstraction. I add a repository only when it earns its place: aggregate-root loading with required includes, a store that isn't EF, or to expose domain-language queries (`GetOverdueInvoices`) so handlers don't repeat LINQ. Otherwise I use the `DbContext` directly in application handlers, which is simpler and keeps full EF power.
**Follow-up traps**
- *"Doesn't that couple to EF?"* Yes, deliberately; swapping ORMs is rare, and EF-specific features (`ExecuteUpdate`, split queries) give real value.
- *"Testing?"* Prefer real DB via Testcontainers/SQL Server container; in-memory provider hides relational behavior.

### 4.2 [Follow-up] Why is the generic repository `IRepository<T>` considered an anti-pattern over EF Core?
**What the interviewer is really testing**: Concrete reasoning on leaky abstraction.
**Typical mid-level answer**: "Because EF already does it." Not enough.
**Strong senior answer**: It lowers everything to the lowest common denominator: `GetById/Add/Remove/GetAll`. Real use cases need includes, projections, paging, filtered updates, locking; the generic interface either grows methods per use case, or exposes `IQueryable` (leaking EF translation rules and deferred execution to callers), or returns whole entities when a DTO projection would read far fewer columns. It encourages "load everything then filter in memory" and N+1. It also suggests all entities are aggregates and are persisted identically, which fails in a real domain. Better: aggregate-specific repositories (`IOrderRepository`) with intention-revealing methods, or none at all for read paths.
**Follow-up traps**
- *"What about `IQueryable` exposure?"* Callers can write non-translatable queries that fail at runtime.
- *"Is it ever fine?"* For simple CRUD admin tools or when needing swappable stores.

### 4.3 [Medium] Explain the Specification pattern and when it helps or hurts.
**What the interviewer is really testing**: Balanced view of an often over-applied pattern.
**Typical mid-level answer**: "It wraps filter criteria in a class." Omits trade-offs.
**Strong senior answer**: A specification represents a business rule as a composable object ("active customers with overdue invoices"), so rules are named, reusable, testable and combinable with And/Or/Not. It helps when the same predicate appears in many queries and in in-memory validation, or when a repository needs one `List(spec)` method rather than dozens of `GetBy...` methods. With EF, specs must expose `Expression<Func<T,bool>>` (not `Func`) so they translate to SQL. It hurts when predicates are used once (a LINQ one-liner is clearer), when specs hide includes/performance and make query tuning opaque, or when composition produces expressions that EF can't translate. I'd adopt it when duplicated predicates become a real maintenance problem.
**Follow-up traps**
- *"`Func` vs `Expression`?"* `Func` executes in memory after loading all rows.
- *"Specs and projections?"* Keep read models separate; use query objects/handlers.

### 4.4 [Hard] Unit of Work and transaction boundaries: where should `SaveChanges` and the transaction live?
**What the interviewer is really testing**: Boundary design and consistency.
**Typical mid-level answer**: "Call SaveChanges in each repository method." Breaks atomicity.
**Strong senior answer**: Transaction boundary = one business use case, owned by the application layer (a command handler), not by repositories. Repositories only stage changes; the handler (or a pipeline behavior) calls `SaveChangesAsync` once so all changes commit atomically. One aggregate per transaction is the DDD guideline; cross-aggregate consistency is eventual via domain/integration events. If a use case also publishes a message, use the Outbox pattern (write the message row in the same transaction, a relay publishes) to avoid the dual-write problem. Explicit `BeginTransaction` is needed only when mixing multiple `SaveChanges` calls or raw SQL, and when using retrying execution strategies (`EnableRetryOnFailure`) you must wrap the transaction in `strategy.ExecuteAsync`. Keep transactions short to limit locking; no HTTP calls inside them.
**Follow-up traps**
- *"Two DbContexts, one transaction?"* Share the connection/transaction or avoid; distributed transactions (MSDTC) aren't available across Azure PaaS generally.
- *"Retry strategy and manual transaction?"* Must run in the strategy's execute delegate or it throws.

### 4.5 [Scenario] Your team splits into "repository camp" vs "DbContext camp". How do you settle it?
**What the interviewer is really testing**: Decision-making under disagreement.
**Typical mid-level answer**: "I'd pick the one I prefer." No criteria.
**Strong senior answer**: Frame it as decision criteria, not taste: Is there a rich domain with aggregates and invariants? Are there multiple stores? Is the team testing with a real DB? Is the app mostly CRUD/queries? For CRUD-heavy slices, handlers using `DbContext` directly are less code and fully expressive. For core domain aggregates with invariants, an aggregate repository gives a stable persistence contract and loads full aggregates consistently. A hybrid is common: repositories (or direct DbContext) on the write side, direct projection queries on the read side. Run it as a time-boxed spike on one feature, compare lines of code, test speed and bug count, then record an ADR. Whatever is chosen, enforce it with an architecture test to avoid mixed styles.
**Follow-up traps**
- *"Hybrid inconsistent?"* Consistency by slice type (command vs query) is acceptable and documented.
- *"Mocking DbContext?"* Avoid; mocking `DbSet` is brittle and doesn't mirror SQL behavior.

**Red flags in answers**
- Generic repository returning `IQueryable` "for flexibility".
- Calling `SaveChanges` per repository method.
- Using in-memory EF provider to claim integration coverage.

**Cheat sheet**
- DbContext = UoW; DbSet = repository-like.
- Add repositories for aggregates/invariants/non-EF stores, not by default.
- Specs: `Expression<Func<>>`, composable; avoid for one-off predicates.
- Transaction = use case, owned by handler; Outbox for dual writes.
- Read side: project to DTOs, skip repositories.
- Settle debates with criteria + ADR + architecture test.

---

## Topic 5: Mediator / MediatR and In-Process CQRS

**Study notes**
- Mediator: objects communicate via a mediator rather than directly. MediatR: `IRequest<T>`, `IRequestHandler`, `INotification`, `IPipelineBehavior<,>`.
- CQRS: separate command (change state) from query (read) models. In-process CQRS doesn't require separate databases or event sourcing.
- Benefits: thin controllers, one class per use case, pipeline behaviors for cross-cutting (validation, logging, transactions).
- Criticisms: hidden control flow (no compile-time call chain, "go to definition" lands on `Send`), handlers calling handlers, runtime-resolved handlers (missing registration = runtime error), ceremony for simple CRUD. Note: MediatR moved to a commercial license model in recent versions; check licensing/alternatives (e.g., Wolverine, Mediator source generator, or plain handlers).
- Use-case classes can be invoked directly without a mediator.

```
Controller --Send(cmd)--> [Validation]->[Logging]->[Transaction]--> Handler --> DbContext
```

### 5.1 [Easy] What does MediatR give you, and what does it not?
**What the interviewer is really testing**: Separating the library from the design.
**Typical mid-level answer**: "It decouples controllers from services." Partly, but it's still coupling to a request type.
**Strong senior answer**: MediatR gives an in-process request/response and notification dispatcher plus pipeline behaviors. It encourages a handler-per-use-case structure and makes cross-cutting concerns declarative. It does not give you CQRS, domain modeling, scalability or loose coupling in the architectural sense: the controller still depends on the request type and its handler is still in the same process and deployment. The benefit is mostly structural consistency and pipeline reuse, which you can also get with plain handler classes plus DI and a decorator. I'd mention the licensing change as a factor when choosing it for new projects.
**Follow-up traps**
- *"Is it a message bus?"* No; in-memory only, no durability, retries or cross-process delivery.
- *"Notifications failing?"* Default publish is sequential; one handler throwing stops the others unless configured.

### 5.2 [Follow-up] What are the main criticisms of Mediator-based architectures?
**What the interviewer is really testing**: Critical thinking about a trendy tool.
**Typical mid-level answer**: "It adds boilerplate." Only one.
**Strong senior answer**: (1) Indirection: IDE navigation from `Send(cmd)` to the handler is by convention; call graphs are hidden. (2) Hidden coupling: handlers calling `_mediator.Send` for other handlers produce a tangled graph that's invisible in constructors, effectively a service locator for use cases. (3) Runtime failures: missing handler registration throws at runtime, not compile time. (4) Ceremony: a request + handler + validator + DTO for a trivial lookup. (5) Behavior order and generic constraints can be subtle, and exceptions in behaviors are hard to debug. (6) A "god pipeline" runs for every request when only some need it. Mitigation: keep handlers independent (shared logic in domain services), use behaviors sparingly, test with startup tests verifying all requests have handlers.
**Follow-up traps**
- *"Handler calls handler?"* Extract shared logic into a domain/application service instead.
- *"Alternatives?"* Direct injection of handler interfaces, source-generated mediators, Wolverine, minimal-API endpoint-per-use-case.

### 5.3 [Medium] Does CQRS require separate databases or Event Sourcing?
**What the interviewer is really testing**: Conceptual clarity.
**Typical mid-level answer**: "Yes, you need a read DB and write DB." Common misconception.
**Strong senior answer**: No. CQRS is a separation of command and query *models*. Levels: (1) separate command/query handlers using the same DB; (2) separate read models (projections/DTOs or Dapper reads vs EF writes) on the same DB; (3) separate read store (e.g., denormalized tables, Cosmos, search index) updated asynchronously; (4) plus Event Sourcing. Most apps benefit from 1-2. Level 3 introduces eventual consistency, projection lag, rebuild/replay concerns and operational cost; only worth it with differing read/write scale or shape. CQRS hurts in simple CRUD domains where reads and writes share the same shape: you double the models for no gain.
**Follow-up traps**
- *"Can queries use EF `AsNoTracking`?"* Yes, and projections with `Select`; that's a fine query side.
- *"Commands returning data?"* Pragmatically returning the new id is common.

### 5.4 [Hard] How do pipeline behaviors for transactions/validation go wrong?
**What the interviewer is really testing**: Operational depth of cross-cutting concerns.
**Typical mid-level answer**: "They keep code clean." Ignores failure modes.
**Strong senior answer**: Pitfalls: a transaction behavior applied to queries wastes connections and locks; so scope it with a marker interface (`ITransactionalCommand`). Behavior ordering matters: validation before transaction, logging outermost. Nested sends inherit/double-wrap transactions. Notification handlers run inside the transaction may do slow or non-transactional work (email) before commit, so a rollback leaves side effects; use domain events dispatched after commit or an outbox. With EF retry strategies the whole unit must be re-executable. Validation behaviors throwing exceptions for expected failures may be costly; return a `Result` instead. Finally, behaviors increase hidden latency, so trace them (Activity/OpenTelemetry).
**Follow-up traps**
- *"Validation: behavior or filter?"* For HTTP-only, endpoint filters/`FluentValidation` at the edge; behavior if use cases are invoked from multiple entrypoints.
- *"Idempotent commands?"* Add an idempotency key behavior.

### 5.5 [Scenario] The team wants MediatR for a 6-endpoint CRUD API. Your view?
**What the interviewer is really testing**: Proportionality.
**Typical mid-level answer**: "Sure, it's clean architecture." Pattern-first.
**Strong senior answer**: I'd ask what problem we're solving. For six CRUD endpoints, controllers/minimal APIs calling a small service or `DbContext` are less code and easier to follow. MediatR starts paying when there are many use cases, shared cross-cutting behaviors, and multiple entry points (HTTP, queue, jobs) invoking the same handlers. I'd propose handler classes injected directly (no mediator) so we can adopt a mediator later with a mechanical change. If the team still prefers it, I'd set conventions: no handler-to-handler sends, behaviors scoped by marker interface, a startup test for handler registration, and note the license position.
**Follow-up traps**
- *"Consistency for future growth?"* Vertical slices scale without a mediator.
- *"Performance cost?"* Small, but real allocations and reflection-based resolution; rarely decisive.

**Red flags in answers**
- "MediatR is CQRS."
- Handlers invoking other handlers freely.
- Transaction behavior applied to every request including queries.

**Cheat sheet**
- Mediator = in-process dispatcher + pipeline; not a bus.
- CQRS levels 1-4; most apps need 1-2.
- Costs: hidden flow, runtime errors, ceremony.
- Scope behaviors by marker interface; validate before transact.
- No side effects before commit: events after commit/outbox.
- Handlers without a mediator are a valid option.

---

## Topic 6: Factory, Abstract Factory, Builder

**Study notes**
- Factory Method / Simple Factory: centralise *which* object is created and how, when construction logic varies or is complex.
- Abstract Factory: create families of related objects (rare in app code today).
- Builder: construct complex objects step by step; in .NET, `WebApplication.CreateBuilder`, `StringBuilder`, `DbContextOptionsBuilder`, test-data builders.
- Modern C# reduces need: object initializers, `required` members, records with `with`, primary constructors, static factory methods (`Money.From(...)`), DI container (the container is a factory).
- Runtime-selected implementation: `Func<TKey,T>`, keyed services (.NET 8), or a factory class injecting `IEnumerable<T>`.

```
caller --> IPaymentProcessorFactory.Create("stripe") --> IPaymentProcessor
```

### 6.1 [Easy] When do you actually need a factory?
**What the interviewer is really testing**: Recognising real need vs `new` anxiety.
**Typical mid-level answer**: "To avoid using `new` directly." Overgeneralised.
**Strong senior answer**: A factory earns its place when creation requires a decision (which implementation by runtime data), needs dependencies the caller shouldn't know, enforces invariants (e.g., an aggregate's `Create` method validating inputs), or must be mockable/replaced. If it's just `new Foo()` with no variation, a factory is noise. In DI, I usually register implementations and select via keyed services or a small `Func<string, IProcessor>`. For invariants, a static factory method on the type (`Order.Place(customer, lines)`) with a private constructor communicates intent and prevents invalid state.
**Follow-up traps**
- *"Factory vs DI?"* DI composes the graph at startup; factories handle runtime-dependent creation.
- *"Static factory methods vs constructors?"* Named, can return cached/different types, can validate.

### 6.2 [Follow-up] Abstract Factory: ever used it?
**What the interviewer is really testing**: Honest experience; knowing it's rare.
**Typical mid-level answer**: "For creating families of objects, like UI themes." Textbook.
**Strong senior answer**: Rarely in application code. Realistic uses: multi-database providers (`DbProviderFactory` in ADO.NET is a genuine abstract factory: creates connection, command, parameter for a provider), or a multi-tenant system where each tenant gets a consistent family (storage + notifier + billing client) for a cloud/region. In most cases DI registrations per profile/environment do the job without a dedicated factory hierarchy. It hurts when each new product in the family forces changes in every factory (the classic extension problem).
**Follow-up traps**
- *"Multi-tenant with different implementations?"* Keyed registrations or a tenant-resolved scope.
- *"Alternatives?"* Configuration-driven composition in the composition root.

### 6.3 [Medium] Builder vs object initializer vs record `with`: choose.
**What the interviewer is really testing**: Modern C# fluency.
**Typical mid-level answer**: "Use Builder for complex objects." Doesn't consider language features.
**Strong senior answer**: For simple DTOs, `required` properties and initializers are enough and compile-time checked. For immutable value-like data, records with `with` copy semantics. A Builder is justified when construction is multi-step, has conditional/ordered configuration, validation at `Build()`, or when the product should be immutable but its parts are gathered incrementally (e.g., building a query, request, email message, test fixtures). Test-data builders (`OrderBuilder.Default().WithLines(3).Build()`) are the most valuable use in my experience as they keep tests readable and resilient to constructor changes. Fluent builders hurt when they allow invalid intermediate states or hide required args; use step builders or `required`.
**Follow-up traps**
- *"Telescoping constructors?"* Replace with options object/builder.
- *"Fluent API thread-safety?"* Builders are generally single-threaded, build immutable results.

### 6.4 [Hard] Factory that resolves implementations by key: how do you design it without a Service Locator?
**What the interviewer is really testing**: Applying DI properly.
**Typical mid-level answer**: "Inject IServiceProvider and call GetService." Service-locator smell.
**Strong senior answer**: Inject all implementations (`IEnumerable<IExporter>`), each exposing a `Format` key, and the factory builds a dictionary once; lookup is O(1) and unknown keys fail fast with a clear exception (or validate at startup). .NET 8's keyed services (`AddKeyedScoped<IExporter, CsvExporter>("csv")`) with `[FromKeyedServices("csv")]` or `GetRequiredKeyedService` simplifies static selection. Keep the factory in the composition/infrastructure layer. Ensure lifetimes are consistent: if implementations are scoped, the factory must be scoped too, or you'll capture them.

```csharp
public sealed class ExporterFactory(IEnumerable<IExporter> exporters)
{
    private readonly Dictionary<string, IExporter> _map =
        exporters.ToDictionary(e => e.Format, StringComparer.OrdinalIgnoreCase);
    public IExporter Get(string f) => _map.TryGetValue(f, out var e) ? e
        : throw new NotSupportedException($"Unknown format {f}");
}
```
**Follow-up traps**
- *"Add new exporter?"* Register a class; the factory doesn't change (OCP).
- *"Duplicate keys?"* `ToDictionary` throws at startup, which is desirable.

### 6.5 [Tricky] Is a Factory interface for `HttpClient` a good idea?
**What the interviewer is really testing**: Knowing platform-provided factories.
**Typical mid-level answer**: "Make my own IHttpClientFactory." Reinventing.
**Strong senior answer**: No; `IHttpClientFactory` already exists and pools/rotates `HttpMessageHandler`s, avoiding socket exhaustion from `new HttpClient()` per call and stale DNS from a never-disposed static client. Prefer typed or named clients with Polly/resilience handlers configured in one place (`AddHttpClient<T>().AddStandardResilienceHandler()` in .NET 8+). Don't wrap it again unless you add domain semantics (e.g., `IPaymentGateway` adapter). Similar platform factories: `IDbContextFactory<T>`, `ILoggerFactory`, `ObjectPool<T>`.
**Follow-up traps**
- *"Static HttpClient okay?"* Works but misses DNS refresh unless `PooledConnectionLifetime` is set.
- *"Typed client lifetime?"* Transient; handlers are pooled separately.

**Red flags in answers**
- Factory for every class "for testability".
- Passing `IServiceProvider` around as a factory.
- Using a Builder where `required` + initializer suffices.

**Cheat sheet**
- Factory when creation varies, needs deps, or enforces invariants.
- Abstract Factory is rare (`DbProviderFactory` is the classic example).
- Prefer `required`, records, `with` before builders; builders shine in test data.
- Keyed services (.NET 8) / dictionary of `IEnumerable<T>` for selection.
- Use platform factories: `IHttpClientFactory`, `IDbContextFactory`.

---

## Topic 7: Strategy and Policy-Based Design

**Study notes**
- Strategy: interchangeable algorithms behind one interface/delegate, selected at runtime.
- Policy-based design: small injectable policies (pricing, retry, authorization) composed to vary behavior.
- Replace long `if/switch` ladders when rules grow, vary by tenant/region, or need independent tests.
- In ASP.NET Core: authorization `IAuthorizationHandler`/policies, `IPasswordHasher`, `IOptions` driven strategies, `IComparer<T>`.
- Selection logic (which strategy) must live somewhere; don't just move the `switch` into a factory without gain.

```
PricingService --> IEnumerable<IDiscountRule> --> apply in order
```

### 7.1 [Easy] Strategy in one example.
**What the interviewer is really testing**: Concrete understanding.
**Typical mid-level answer**: "Interface with multiple implementations." No problem context.
**Strong senior answer**: Shipping cost calculation: carriers (`FlatRate`, `Weight-based`, `Zone-based`) change independently and per customer contract. `IShippingCostStrategy.Calculate(Order)` implementations are registered and selected by carrier code. The caller no longer has a growing `switch`, each strategy is unit-testable, and adding a carrier means adding a class. If there are only two stable cases, I'd keep the `switch`/pattern-matching or pass a `Func<Order,decimal>`. The pattern isn't free: more types, and the selection rule still needs a home.
**Follow-up traps**
- *"Strategy vs State?"* Strategy is chosen externally and rarely changes; State changes itself over the object's lifecycle.
- *"Delegate vs interface?"* Delegate for stateless one-liners; interface when it has identity, config, dependencies.

### 7.2 [Follow-up] When should a switch statement stay a switch?
**What the interviewer is really testing**: Anti-dogma.
**Typical mid-level answer**: "Never; use polymorphism." Dogmatic.
**Strong senior answer**: A `switch` over a closed set that rarely changes, in one place, is simpler than a hierarchy; modern switch expressions are concise and the compiler flags missing enum cases (warn/with `_ => throw`). Replace it when: it's duplicated across several methods (shotgun surgery on adding a case), it grows frequently, branches need dependencies or independent tests, or rules vary per tenant. Check for the same `switch(type)` appearing in 3 places: that's the signal. Parsing/mapping edges (e.g., mapping DB codes to enums) are legitimate switches.
**Follow-up traps**
- *"Enum + switch vs discriminated union?"* C# lacks DU natively; use sealed hierarchies + pattern matching.
- *"Performance?"* Switch is faster; rarely matters.

### 7.3 [Medium] Design a pricing/discount engine that business changes weekly.
**What the interviewer is really testing**: Policy composition and governance of change.
**Typical mid-level answer**: "Many strategy classes." Misses ordering, conflicts, auditing.
**Strong senior answer**: Model each discount as a rule with `AppliesTo(context)` and `Apply(context)`, injected as an ordered collection. Decide explicitly: ordering, stacking vs best-of, caps, rounding, and idempotence, because bugs hide there more than in the rules. Rules are pure functions of an immutable pricing context, so they're deterministic and unit-testable with table-driven tests. Output includes an explanation (which rules applied) for support and audit. If business changes rules weekly *without deploys*, consider data-driven rules (config/DB) or a rules engine; but a DSL brings its own testing/versioning cost, so only go there with evidence. Version rule sets with effective dates.
**Follow-up traps**
- *"Where's the ordering stored?"* Explicit priority property or composition code, not DI registration order.
- *"Floating point?"* Use `decimal`, define rounding once.

### 7.4 [Hard] Where does Strategy/policy design hurt?
**What the interviewer is really testing**: Costs and hidden complexity.
**Typical mid-level answer**: "More classes." Shallow.
**Strong senior answer**: (1) Selection logic becomes implicit (DI order, attributes), harder to trace than a switch. (2) Strategies often need the same context and end up with wide, leaky interfaces (a method taking 8 parameters) because one strategy needs data others don't (ISP violation). (3) Combinatorial interactions between policies (A then B differs from B then A) aren't captured by independent classes. (4) Over-granular strategies spread one decision over many files. (5) Configuration-driven policy tables can silently misconfigure production. Mitigate with a context object, explicit ordering, and tests of the composed behavior, not just units.
**Follow-up traps**
- *"Detect wrong config?"* Validate options at startup (`ValidateOnStart`).
- *"Strategies sharing code?"* Composition via helper, not inheritance.

### 7.5 [Scenario] Authorization rules differ per resource and role; how do you structure them?
**What the interviewer is really testing**: Using framework policy features instead of custom strategies.
**Typical mid-level answer**: "Check roles in each controller." Scattered.
**Strong senior answer**: Use ASP.NET Core's policy-based authorization: requirements + handlers (`AuthorizationHandler<TRequirement, TResource>`) for resource-based checks (`IAuthorizationService.AuthorizeAsync(user, order, policy)`), which is Strategy/Policy by design. Keep policies named and centralized, handlers testable, and avoid duplicating role strings. Claims come from Entra ID tokens; combine with data-level filters (EF global query filters for tenant) as defense in depth. Don't build a custom engine unless requirements exceed the framework (e.g., dynamic ABAC with an external policy service).
**Follow-up traps**
- *"Global query filters pitfalls?"* `IgnoreQueryFilters` bypass; ensure tenant filter tested.
- *"Resource-based vs policy only?"* Policy alone can't see the resource instance.

**Red flags in answers**
- Strategy hierarchy for two stable cases.
- Selection logic still a switch in a different file, presented as a win.
- Rules with no defined ordering/stacking semantics.

**Cheat sheet**
- Strategy = swap algorithm; delegate for trivial, interface for identity/deps.
- Replace a switch when duplicated, growing, or tenant-varying.
- Define ordering/stacking/rounding for rule engines explicitly.
- Use ASP.NET Core policies for authorization.
- Validate policy configuration at startup.

---

## Topic 8: Decorator, Pipeline Behaviors, Chain of Responsibility

**Study notes**
- Decorator: wrap an object with the same interface to add behavior (logging, caching, retry, metrics) without changing it.
- Chain of Responsibility: each handler decides to handle or pass on. ASP.NET Core middleware is CoR (+ pipeline): each `RequestDelegate` calls `next` or short-circuits.
- MediatR behaviors, `DelegatingHandler` for HttpClient, EF Core interceptors, endpoint filters are the same shape.
- Order matters; the outermost wraps everything.
- Decorating in MS DI needs manual wiring or Scrutor (`services.Decorate<>()`).

```
Request -> [ExceptionHandler] -> [Auth] -> [RateLimiter] -> [Endpoint]
Response <-            <-              <-            <-
```

### 8.1 [Easy] Decorator vs inheritance for adding caching to a service.
**What the interviewer is really testing**: Composition over inheritance reasoning.
**Typical mid-level answer**: "Subclass and override." Rigid.
**Strong senior answer**: A `CachedProductRepository : IProductRepository` wrapping the real one adds cache-aside without modifying or subclassing it, can be toggled in DI, and composes with other decorators (logging, metrics). Inheritance couples to the base implementation and can't be combined with other concerns in different orders. The cost: delegating every interface method (noisy for wide interfaces; so keep interfaces narrow), and ordering/stacking confusion. For cross-cutting concerns across many interfaces, consider generic pipeline behaviors or `DispatchProxy`/AOP, accepting some magic.
**Follow-up traps**
- *"Cache key and invalidation?"* Decorator must own invalidation on writes; otherwise stale data.
- *"Scrutor?"* Library for `Decorate`; or manual factory registrations.

### 8.2 [Follow-up] Explain ASP.NET Core middleware as Chain of Responsibility and where order bites.
**What the interviewer is really testing**: Applied understanding of the runtime pipeline.
**Typical mid-level answer**: "Middlewares run one after another." Misses short-circuit and bidirectional flow.
**Strong senior answer**: Each middleware receives `HttpContext` and `next`; it can act before, call `next` (or not, short-circuiting), then act after on the way back. Order is the contract: exception handling first (so it catches everything downstream), then HSTS/HTTPS redirection, static files, routing, CORS, authentication before authorization, rate limiting, then endpoints. Typical bugs: `UseAuthorization` before `UseAuthentication`, CORS after endpoints (no headers on preflight), writing to the response then calling `next` (headers already sent), and per-request state in singleton middleware fields. Conventional middleware is constructed once, so scoped services must be injected through `InvokeAsync` parameters.
**Follow-up traps**
- *"Middleware vs filter?"* Filters (MVC/endpoint) have access to routing/model binding context; middleware is global and earlier.
- *"Terminal middleware?"* `Run`; or endpoints.

### 8.3 [Medium] When do decorators/behaviors make code harder to understand?
**What the interviewer is really testing**: Costs of invisible composition.
**Typical mid-level answer**: "They don't." Naive.
**Strong senior answer**: Behavior is invisible at the call site: reading `OrderService` doesn't tell you it's wrapped by validation, retries, transaction and caching. Debugging deep stacks of wrappers is harder, order-dependent bugs are subtle (retry outside vs inside a transaction: retrying inside a transaction is incorrect after a failure), and duplicate application (cache decorator plus HTTP caching plus EF cache) causes inconsistent staleness. Keep the stack short, document order in one composition place, name decorators for their role, and add tracing. If one decorator holds business rules, it's no longer cross-cutting; move it into the domain/application code.
**Follow-up traps**
- *"Retry outside or inside transaction?"* Outside: re-run the whole unit of work.
- *"How to see the effective pipeline?"* Log at startup or architecture/ integration test.

### 8.4 [Hard] Design a cross-cutting pipeline for a command handler: logging, validation, authorization, transaction, idempotency. Order?
**What the interviewer is really testing**: Concrete ordering reasoning.
**Typical mid-level answer**: "Any order works." Wrong.
**Strong senior answer**: Outermost: logging/telemetry + exception translation (capture everything, including failures of inner steps). Next: authorization (reject early before spending resources), then validation (cheap, no DB), then idempotency check (return stored result for a repeated key), then the transaction wrapping the handler, so a rollback includes any idempotency record written. Post-commit: dispatch domain events/outbox relay. Put retries *around* the entire transaction unit if transient errors are expected, and only for idempotent operations. Each behavior is scoped by marker interface so queries skip transaction and idempotency.
**Follow-up traps**
- *"Idempotency record inside the transaction?"* Yes, so record and effects commit atomically.
- *"Where does exception-to-ProblemDetails mapping live?"* Edge: `IExceptionHandler`/middleware, not in handlers.

### 8.5 [Tricky] "Middleware or action filter for cross-cutting concern X?" Decide for correlation IDs, validation, response compression, per-action audit.
**What the interviewer is really testing**: Choosing level of abstraction correctly.
**Typical mid-level answer**: "Middleware for all." Loses context.
**Strong senior answer**: Correlation ID: middleware (needed early, applies to all requests, including static/failed ones, and sets logging scope). Response compression: middleware (HTTP-level). Model validation: endpoint/action filter (needs bound model and endpoint metadata). Per-action audit with attributes: filter (needs `ActionDescriptor`, result, and attribute metadata). Rule: if it needs MVC/endpoint knowledge, use a filter; if it is protocol-level and universal, use middleware.
**Follow-up traps**
- *"Minimal APIs?"* Endpoint filters (`AddEndpointFilter`).
- *"Order of filters?"* Authorization, resource, action, exception, result filters.

**Red flags in answers**
- Cannot explain why `UseAuthentication` precedes `UseAuthorization`.
- Decorators containing business rules.
- Retrying non-idempotent operations in a generic behavior.

**Cheat sheet**
- Decorator: same interface, adds behavior; Scrutor or manual DI.
- Middleware = CoR with before/after and short-circuit; order is the contract.
- Pipeline order: telemetry -> authz -> validation -> idempotency -> transaction -> events.
- Retry wraps the transaction, not the reverse.
- Filter if it needs endpoint context; middleware if protocol-wide.

---

## Topic 9: Observer, Events, Pub/Sub and Domain Events

**Study notes**
- Observer: subject notifies subscribers; C# `event`, `IObservable<T>`, `INotificationHandler` (MediatR) are variants.
- In-process events vs integration events (cross-service via bus). Domain events record something that happened in the domain, raised by aggregates, handled in-process (often before/after commit).
- C# `event` pitfalls: memory leaks (publisher holds subscriber refs), exceptions in handlers abort others, thread-safety on invoke, `async void`.
- Pub/Sub (Azure Service Bus topics, Event Grid): decoupled in time/space, at-least-once delivery -> consumers must be idempotent.
- Outbox: atomic state change + message record; Inbox for dedupe.

```
Aggregate.Raise(OrderPlaced) -> SaveChanges -> dispatch -> handlers
                                   \-> Outbox row -> relay -> Service Bus topic
```

### 9.1 [Easy] Observer vs pub/sub: difference?
**What the interviewer is really testing**: Coupling in time and space.
**Typical mid-level answer**: "Same thing." Misses the broker.
**Strong senior answer**: Observer: subject holds references to observers and calls them directly, in-process and synchronous (unless made async). Pub/Sub: a broker/event channel sits between publisher and subscribers, so neither knows the other, can run in different processes, and delivery can be asynchronous and durable (Service Bus topic/subscription). Consequences: pub/sub gives temporal decoupling and independent scaling but adds eventual consistency, duplicates (at-least-once), ordering concerns and observability needs; observer is simpler, with errors propagating to the caller.
**Follow-up traps**
- *"Event Grid vs Service Bus?"* Event Grid for reactive event notification (push, fan-out); Service Bus for reliable messaging with sessions, DLQ, transactions.
- *"Ordering?"* Service Bus sessions give FIFO per session id.

### 9.2 [Follow-up] What are common C# event pitfalls?
**What the interviewer is really testing**: Practical correctness.
**Typical mid-level answer**: "Null check before invoke." Only one.
**Strong senior answer**: (1) Leaks: a long-lived publisher keeps short-lived subscribers alive unless unsubscribed (or weak events). (2) One handler throwing prevents later ones; multicast invocation list stops at the exception. (3) `async void` handlers swallow exceptions into the sync context and can crash the process; use async-aware dispatch. (4) Re-entrancy and ordering are implicit. (5) Handlers run on the raising thread, so slow handlers block the publisher. Use `handler?.Invoke` (thread-safe copy), or for async workflows prefer explicit `Task`-returning handler interfaces or channels (`System.Threading.Channels`).
**Follow-up traps**
- *"Alternatives for async fan-out in process?"* `Channel<T>` + a background consumer.
- *"Rx?"* Good for streams/time-based composition; overkill for simple notifications.

### 9.3 [Medium] Domain events: when to dispatch, before or after commit?
**What the interviewer is really testing**: Consistency reasoning.
**Typical mid-level answer**: "After saving." No nuance.
**Strong senior answer**: Two kinds of handlers: (a) those that must change *other aggregates in the same consistency boundary* or the same transaction, dispatched *before* commit (inside the unit of work) so all succeed or fail together, and (b) side effects like email, integration events, search index updates, dispatched *after* commit, ideally through an outbox so they're not lost if the process dies between commit and publish. A typical EF Core approach: collect events from tracked aggregates in `SaveChanges` override or behavior, then dispatch. Risks: handlers in the transaction that fail or are slow, cascading events creating invisible workflows, and in-memory post-commit dispatch losing events on crash (hence Outbox).
**Follow-up traps**
- *"Dual-write problem?"* Writing DB and bus separately can't be atomic; outbox solves it.
- *"Event naming?"* Past tense facts (`OrderPlaced`), carrying minimal data.

### 9.4 [Hard] How do you make event consumers safe with at-least-once delivery?
**What the interviewer is really testing**: Idempotency design.
**Typical mid-level answer**: "Use exactly-once delivery." Not realistically available end to end.
**Strong senior answer**: Assume duplicates and reordering. Make handlers idempotent: natural idempotence (upsert by key, set-state not increment), or dedupe with an inbox table keyed by `MessageId` inserted in the same transaction as the effect (unique constraint catches duplicates). Service Bus offers duplicate detection on the sender side within a time window, helpful but not sufficient alone. Handle poison messages with max delivery count and dead-letter queue plus alerting and a replay tool. Version events additively (consumers ignore unknown fields). Where order matters, use sessions/partition keys or carry version numbers and discard stale updates.
**Follow-up traps**
- *"Exactly-once in Kafka/Service Bus?"* Limited scopes; still design idempotent consumers.
- *"Retry delay?"* Backoff and eventually DLQ to avoid poison loops.

### 9.5 [Scenario] A domain event handler sends emails and calls a third-party API inside the DB transaction. What's wrong and how to fix?
**What the interviewer is really testing**: Spotting side effects in transactions.
**Typical mid-level answer**: "It might be slow." Misses atomicity.
**Strong senior answer**: Problems: the external side effect can happen even if the transaction later rolls back (email for an order that doesn't exist), network latency holds locks, and a failure in the third-party call rolls back valid business data. Fix: record an outbox message in the same transaction; a background relay/consumer (Service Bus + worker, or Durable Functions) performs the email/API call with retries and idempotency. If the user needs the result synchronously, restructure as an explicit process with status. Monitor outbox lag and DLQ.
**Follow-up traps**
- *"Why not just publish after SaveChanges?"* Crash between commit and publish loses the event.
- *"Outbox relay concurrency?"* Use locking/claiming rows or a single leader.

**Red flags in answers**
- "Exactly-once delivery solves duplicates."
- External calls inside DB transactions.
- `async void` event handlers in business code.

**Cheat sheet**
- Observer = direct in-process; pub/sub = broker-mediated.
- Domain events: pre-commit for same boundary, post-commit/outbox for side effects.
- Assume at-least-once: idempotent consumers, inbox, DLQ.
- Service Bus topics for reliable fan-out; Event Grid for reactive notifications.
- C# events: leaks, exceptions, `async void`.

---

## Topic 10: Adapter, Facade, Anti-Corruption Layer, Proxy

**Study notes**
- Adapter: convert one interface to another (wrap a third-party SDK behind your `IPaymentGateway`).
- Facade: simplified interface over a complex subsystem.
- Anti-Corruption Layer (ACL): a translation boundary protecting your domain model from a legacy/external model (DDD); includes adapters, translators, facades.
- Proxy: stand-in controlling access (lazy loading, remote, caching, security). EF Core lazy-loading proxies (`UseLazyLoadingProxies`, requires virtual navigation properties) and change-tracking proxies are examples.
- Lazy loading pitfalls: N+1 queries, surprise queries after DbContext disposal, serialization triggering loads/cycles.

```
Domain --> IPaymentGateway (our terms) <-- StripeAdapter --> Stripe SDK (their terms)
```

### 10.1 [Easy] Adapter vs Facade vs ACL.
**What the interviewer is really testing**: Distinguishing similar-looking wrappers by intent.
**Typical mid-level answer**: "They're all wrappers." Misses intent.
**Strong senior answer**: Adapter changes *shape* to match an interface you already expect. Facade reduces *complexity*: one simple entry point over many calls. ACL protects *meaning*: it translates concepts and invariants between bounded contexts so a legacy model's vocabulary (status codes, nullable chaos) doesn't leak into your domain. An ACL often contains adapters and facades but is a strategic boundary: it has its own model translation and tests. Example: wrapping a legacy ERP SOAP service: Adapter (SOAP -> our `IInventory`), Facade (hide the 4-call sequence into one `Reserve`), ACL (map ERP status "07" to our `StockState.Backordered`).
**Follow-up traps**
- *"Where do DTOs of the external system live?"* Inside the adapter/infrastructure only.
- *"Extra cost?"* Mapping code and drift; justified for volatile or poor external models.

### 10.2 [Follow-up] When is wrapping a third-party SDK worth it?
**What the interviewer is really testing**: Cost-benefit on abstraction.
**Typical mid-level answer**: "Always, to avoid vendor lock-in." Overstated.
**Strong senior answer**: Wrap when the dependency is a volatile or replaceable capability (payments, email/SMS), when its model is poor for your domain, or when you need a testing seam, centralized resilience and telemetry. Don't wrap stable, ubiquitous libraries (`ILogger`, EF Core, Azure SDK clients) just for theoretical portability; the abstraction lowers capability to the lowest common denominator and you'll rewrite anyway on migration. Design the wrapper around *your use cases* (`ChargeCustomer`), not mirroring the vendor API; mirrored interfaces leak. Keep it thin: map, call, translate errors.
**Follow-up traps**
- *"Azure Blob abstraction?"* Wrap only for domain needs (e.g., `IDocumentStore`), not as generic storage.
- *"Vendor exceptions?"* Translate into your own failure types at the adapter.

### 10.3 [Medium] Designing an ACL while strangling a legacy system.
**What the interviewer is really testing**: Strategic boundaries during migration.
**Typical mid-level answer**: "Call legacy through a service." No translation or direction.
**Strong senior answer**: Define the new domain model first, then write translators between it and the legacy contracts, so the new code never references legacy types. Put the ACL in its own project; the new context depends on an interface it owns. Handle semantic gaps (different identifiers, state machines, time zones, null semantics), and data sync strategy (read-through vs replicate via events/CDC). Keep it removable: when the legacy capability is replaced the ACL shrinks. Test with contract/golden-master tests against recorded legacy responses. Risk: ACL becomes permanent and a hotspot; keep ownership and sunset dates.
**Follow-up traps**
- *"Shared database integration?"* Avoid; if unavoidable, use views/ACL at the data layer and treat as technical debt.
- *"Performance of translations?"* Usually negligible vs I/O.

### 10.4 [Hard] EF Core lazy loading: when does the Proxy pattern hurt?
**What the interviewer is really testing**: Knowing real pitfalls.
**Typical mid-level answer**: "Lazy loading is convenient." Ignores N+1.
**Strong senior answer**: Lazy loading proxies issue a query on first navigation access, invisibly, so loops over entities produce N+1 round trips; they require virtual navigations and proxy-compatible entities, may trigger after the context is disposed (exceptions) or during JSON serialization (loading graphs and cycles), and they obscure performance review since queries aren't visible in the repository/handler code. Prefer explicit loading: `Include`/`ThenInclude`, split queries for large collection includes (`AsSplitQuery`) to avoid cartesian explosion, or projection to DTOs with `Select`. Lazy loading is occasionally fine in desktop/small tools or rarely-accessed optional navs. Also consider `EnableDetailedErrors`/logging to spot N+1.
**Follow-up traps**
- *"Eager Include vs split query?"* Single query risks duplicated rows; split queries do multiple round trips and may be inconsistent without a transaction.
- *"Lazy loading without proxies?"* Via `ILazyLoader` injection into entities, still has same issue.

### 10.5 [Tricky] Is a Facade over your own microservices a good idea?
**What the interviewer is really testing**: Gateway/Facade distinction and coupling.
**Typical mid-level answer**: "Yes, simplifies clients." Neglects risks.
**Strong senior answer**: A facade/gateway aggregation (or BFF) can reduce chattiness and hide topology, but it centralizes coupling: every client-facing change touches it, it becomes a bottleneck/single point of failure, and logic creeping in turns it into a distributed monolith core. Keep it thin: aggregation and translation, no business rules; give each client type its own BFF if their needs diverge; use APIM/YARP for routing and cross-cutting, and put orchestration of business processes in services. Apply resilience (timeouts, partial responses) since a facade multiplies dependency failures.
**Follow-up traps**
- *"Partial failure?"* Return degraded response with explicit status per section.
- *"GraphQL?"* An alternative aggregation approach with its own operational trade-offs.

**Red flags in answers**
- Mirroring a vendor SDK 1:1 and calling it an abstraction.
- Business logic in the facade/gateway.
- Leaving lazy loading on globally without N+1 awareness.

**Cheat sheet**
- Adapter = shape; Facade = simplicity; ACL = meaning/boundary.
- Wrap volatile/poor-fit dependencies, not stable platform libraries.
- Design the wrapper around your use cases.
- EF lazy loading = N+1 and hidden I/O; prefer `Include`/projection.
- Keep gateways thin; BFF per client type.

---

## Topic 11: Singleton, Lifetimes, Options Pattern, Null Object

**Study notes**
- Classic Singleton (static `Instance`) = global state, hard to test, hidden dependency. Prefer DI singleton lifetime: one instance per container, but replaceable and testable.
- Thread safety: singletons are shared; use immutability, `ConcurrentDictionary`, `Lazy<T>`, `SemaphoreSlim`, or `Interlocked`. `lock` can't be used across `await`.
- Options pattern: strongly typed config. `IOptions<T>` (singleton, read once), `IOptionsSnapshot<T>` (scoped, reloads per request), `IOptionsMonitor<T>` (singleton, change notifications). Validate with data annotations / `ValidateOnStart`.
- Null Object: a do-nothing implementation instead of null checks (`NullLogger`, `NullEmailSender`).

```
appsettings / KeyVault / env --> IConfiguration --> Bind<T> --> IOptions<T>
```

### 11.1 [Easy] Classic Singleton vs DI singleton.
**What the interviewer is really testing**: Why DI singleton is better and what stays the same.
**Typical mid-level answer**: "Same thing, one instance." Misses testability and hidden coupling.
**Strong senior answer**: A static `Singleton.Instance` is a global accessed from anywhere: dependencies are invisible, tests can't substitute it, lifetime and initialization order are fixed, and hidden state couples tests together. A DI singleton gives one instance per container but is injected explicitly, can be faked, and its lifetime/disposal is managed by the container. The thread-safety burden remains identical: all callers share it. I'd use DI singletons for stateless services, caches, and expensive clients (`ServiceBusClient`, `CosmosClient`, Redis multiplexer) that Microsoft guidance says to reuse.
**Follow-up traps**
- *"Double-checked locking by hand?"* Use `Lazy<T>` instead.
- *"Disposal?"* Container disposes IDisposable singletons it created at shutdown, not ones you supply as instances.

### 11.2 [Follow-up] Choose among `IOptions`, `IOptionsSnapshot`, `IOptionsMonitor`.
**What the interviewer is really testing**: Lifetime-aware config handling.
**Typical mid-level answer**: "Snapshot reloads config." Doesn't handle singleton constraints.
**Strong senior answer**: `IOptions<T>` reads once at first use, doesn't see reloads; use for static config. `IOptionsSnapshot<T>` is scoped: recomputed per request, usable only in scoped/transient services (not in singletons). `IOptionsMonitor<T>` is singleton, always returns the current value and offers `OnChange`, so it's the choice for singletons and background services needing live config. Add validation (`.ValidateDataAnnotations().ValidateOnStart()`) so a bad deployment fails fast instead of at first request. Secrets come from Key Vault via the configuration providers; don't log options objects containing secrets. Reloading some options (connection strings, pool sizes) doesn't affect already constructed clients, so reload-ability can be misleading.
**Follow-up traps**
- *"Named options?"* `IOptionsMonitor<T>.Get(name)` for multiple instances, e.g. per-tenant.
- *"Azure App Configuration?"* Supports refresh with sentinel keys and feature flags.

### 11.3 [Medium] Thread-safety design for a singleton in-memory cache.
**What the interviewer is really testing**: Concurrency reasoning (cache stampede, async).
**Typical mid-level answer**: "Use a Dictionary with a lock." Blocks, can't `await` inside, stampede.
**Strong senior answer**: Prefer `IMemoryCache` (thread-safe, with expiration and size limits) over a custom dictionary. Understand `GetOrCreateAsync` doesn't guarantee single factory execution: concurrent misses can run the factory multiple times (a stampede). If the work is expensive, store a `Lazy<Task<T>>` or guard per-key with `SemaphoreSlim`, and set absolute+sliding expiry. In multi-instance Azure deployments, in-memory caches are per-instance: stale/inconsistent across nodes, so use distributed cache (Redis) or accept eventual staleness. .NET 9 introduced `HybridCache` (L1+L2 with stampede protection). Limit size to avoid memory pressure and evictions under load.
**Follow-up traps**
- *"Lock with async?"* Use `SemaphoreSlim.WaitAsync`.
- *"Cache invalidation across instances?"* Pub/sub (Redis) or short TTL.

### 11.4 [Hard] Where does the Options pattern hurt?
**What the interviewer is really testing**: Config design maturity.
**Typical mid-level answer**: "It doesn't." Dogmatic.
**Strong senior answer**: Hurts when options classes become kitchen sinks used everywhere (config coupling), when business rules live in config without validation or versioning (silent misbehavior), or when feature toggling is done through options rather than a proper feature-flag system (no targeting, no audit). Binding is by convention: renaming a property silently yields defaults unless validated (`ValidateOnStart`, required members). Options injected into domain classes couple the domain to infrastructure config; map to domain-specific parameters at the composition root. Environment drift: dozens of settings with no schema; document and validate.
**Follow-up traps**
- *"Secrets in options?"* Key Vault references/managed identity; avoid plain secrets in repos.
- *"Feature flags?"* `Microsoft.FeatureManagement` with Azure App Configuration.

### 11.5 [Tricky] Is Null Object better than returning null/throwing?
**What the interviewer is really testing**: Knowing masking risks.
**Typical mid-level answer**: "Yes, avoids null checks." Overgeneral.
**Strong senior answer**: Good where "do nothing" is a legitimate behavior: `NullLogger`, a no-op notifier in dev, a "guest user" with no permissions, an empty discount. It removes branching and keeps callers uniform. It hurts when absence is an error that should surface: a `NullPaymentGateway` that silently "succeeds" hides misconfiguration and loses money. For lookups, prefer explicit types: `T?` with nullable reference types, `TryGet`, `Option`/`Result`, or throw `NotFoundException` when absence is exceptional. Decide by whether absence is a normal domain case or a fault.
**Follow-up traps**
- *"Null Object and LSP?"* Must honor the contract; a no-op that violates postconditions breaks LSP.
- *"Empty collection vs null?"* Return empty collections.

**Red flags in answers**
- Static singleton for convenience/global state.
- `IOptionsSnapshot` injected into a singleton.
- Null Object used to silence failures.

**Cheat sheet**
- Prefer DI singleton over static; mind thread-safety.
- Options: `IOptions` static, `Snapshot` scoped, `Monitor` singleton + change.
- `ValidateOnStart` for fail-fast config.
- Cache: stampede protection, per-instance staleness; `HybridCache` in .NET 9.
- Null Object where no-op is valid, never for hidden failure.

---

## Topic 12: Result Pattern vs Exceptions

**Study notes**
- Exceptions: for *exceptional*, unexpected conditions (I/O failure, bugs); they unwind the stack and are slow-ish when thrown frequently (stack trace capture).
- Result pattern: return a value representing success or an error (`Result<T>`, `OneOf`, `ErrorOr`, FluentResults) for *expected* business failures (validation, not found, conflict).
- Hybrid is typical: Results inside the application layer; exceptions for infrastructure faults; map to ProblemDetails (RFC 7807/9457) at the API edge.
- ASP.NET Core: `IExceptionHandler` (.NET 8), `ProblemDetails`, `TypedResults`.

```csharp
public readonly record struct Result<T>(T? Value, Error? Error)
{ public bool IsSuccess => Error is null; }
```

### 12.1 [Easy] When to use exceptions and when to return results?
**What the interviewer is really testing**: Principled error modeling.
**Typical mid-level answer**: "Exceptions for errors, results for validation." Fuzzy.
**Strong senior answer**: Ask "is this failure part of normal business flow that the caller must handle?" If yes (invalid input, insufficient funds, version conflict), model it explicitly in the signature so the caller can't ignore it and it is cheap, and reflect it as 4xx. If it's unexpected or unrecoverable at that layer (DB down, null reference, bug, timeout), throw; handle at a boundary, log, return 500 or retry. Exceptions as control flow make flow invisible, slow when hot, and tangle try/catch. Conversely, Results everywhere make every method return wrappers and force mapping boilerplate.
**Follow-up traps**
- *"Perf of exceptions?"* Throwing is expensive relative to returns; negligible for rare faults, significant in hot loops.
- *"Guard clauses?"* Throw `ArgumentException` for programming errors (precondition violations).

### 12.2 [Follow-up] Critique the Result pattern in C#.
**What the interviewer is really testing**: Balanced view of a popular idea.
**Typical mid-level answer**: "It's cleaner than exceptions." One-sided.
**Strong senior answer**: Pros: honest signatures, forced handling, no hidden control flow, good performance, easy mapping to HTTP statuses. Cons: C# lacks discriminated unions and `?`-style propagation, so it's verbose (`if (r.IsFailure) return r.Error;` repeated); the ecosystem has competing libraries and no standard; it infects all layers; unchecked `.Value` access reintroduces exceptions; error taxonomy can sprawl; it doesn't compose with framework code that throws (EF, HttpClient), so you still need try/catch at boundaries. Use a small in-house type with a closed set of error kinds (Validation, NotFound, Conflict, Unauthorized) and map them centrally.
**Follow-up traps**
- *"Library or custom?"* Small custom type avoids dependency churn; libraries fine if standardized across teams.
- *"Union types in future C#?"* Proposed; don't design around it yet.

### 12.3 [Medium] Where should exceptions be translated to HTTP responses?
**What the interviewer is really testing**: Boundary design.
**Typical mid-level answer**: "try/catch in each controller." Duplicated.
**Strong senior answer**: At the edge, in one place: `IExceptionHandler`/exception middleware maps known domain/validation exceptions to `ProblemDetails` with proper status codes and trace id, and everything else to 500 without leaking internals. If using Results, an extension converts `Error` kinds to `TypedResults.Problem`. Don't catch-and-rethrow in layers, don't swallow, and log once at the boundary (avoid duplicated logs). Include correlation ids and stable error codes for clients. Treat `OperationCanceledException` from client aborts as non-errors.
**Follow-up traps**
- *"Log and rethrow?"* Produces duplicate logs; log where handled.
- *"Leaking exception messages?"* Never expose stack traces in production.

### 12.4 [Hard] A handler can fail in 5 ways. Design the error model for API consumers.
**What the interviewer is really testing**: Contract-level thinking.
**Typical mid-level answer**: "Return error strings." Unstable contract.
**Strong senior answer**: Define stable machine-readable error codes (`order.insufficient_stock`), human message separate, and HTTP status by category: 400/422 validation (with field errors), 404 not found, 409 conflict/concurrency (ETag/rowversion mismatch), 403/401 authz, 429 throttling with `Retry-After`, 503 for transient dependency failure. Use ProblemDetails with an `errors` extension. Treat the codes as part of the versioned API contract. Distinguish retryable from non-retryable so clients and retry policies behave correctly (don't retry 400/409 blindly). Document in OpenAPI.
**Follow-up traps**
- *"400 vs 422?"* Team convention; be consistent.
- *"Concurrency?"* Optimistic concurrency via `rowversion`/`IsRowVersion` -> `DbUpdateConcurrencyException` -> 409.

### 12.5 [Scenario] Reviewing code that catches `Exception` everywhere and returns null.
**What the interviewer is really testing**: Recognising failure-swallowing.
**Typical mid-level answer**: "Add logging." Doesn't fix semantics.
**Strong senior answer**: Returning null on any exception conflates "not found" with "database down", hides outages and corrupts downstream logic; callers proceed with null. Fix: catch only specific exceptions you can handle meaningfully, let others propagate to the boundary, model "not found" explicitly (nullable return/Result), and keep `catch` blocks that add value (translate, compensate, retry). Add tests for failure paths and alerts on error rates. Use filters (`when`) to narrow, and `throw;` to preserve stack trace.
**Follow-up traps**
- *"`throw ex` vs `throw`?"* `throw ex` resets the stack trace.
- *"When catch Exception OK?"* Top-level boundaries (middleware, worker loop) with logging.

**Red flags in answers**
- Exceptions for normal validation flow in hot paths.
- Swallowing exceptions or returning null on failure.
- Result type that exposes `.Value` unguarded everywhere.

**Cheat sheet**
- Expected business failure -> Result; unexpected fault -> exception.
- Translate at one boundary (`IExceptionHandler`, ProblemDetails).
- Stable error codes; map 409 for concurrency, 429 for throttling.
- Log once, at the handling point; use `throw;`.
- C# lacks DUs: Results are verbose; keep the type small.

---

## Topic 13: State Pattern, State Machines, Template Method

**Study notes**
- State pattern: object's behavior changes with its internal state, delegating to state objects. State machine: explicit states, transitions, guards, actions; libraries: Stateless (in-process), MassTransit/Saga state machines, Durable Functions/Durable Task (long-running orchestrations), Azure Logic Apps.
- Persistent workflows must store state in DB; in-memory state machines vanish on restart.
- Template Method: base class defines algorithm skeleton, subclasses override steps. Composition (strategy injection) usually replaces it.
- Enum + guard checks inside an aggregate is often enough for simple lifecycle (Order: Draft -> Placed -> Paid -> Shipped).

```
Draft --Place--> Placed --Pay--> Paid --Ship--> Shipped
            \--Cancel--> Cancelled (from Draft/Placed only)
```

### 13.1 [Easy] Model an order lifecycle: enum + methods, State pattern, or state machine library?
**What the interviewer is really testing**: Proportionality.
**Typical mid-level answer**: "State pattern with a class per state." Heavy.
**Strong senior answer**: For 4-6 states with few transitions, an enum field on the aggregate and methods that guard transitions (`Pay()` throws/returns failure unless `Placed`) is simplest and persists trivially with EF. If transitions grow, have guards, entry/exit actions, or need visualization (DOT graph), a state machine library like Stateless declares transitions in one place and is easier to audit than scattered ifs. The classic State pattern (class per state) pays off when behavior (not just allowed transitions) differs substantially per state. Always enforce transitions in the domain, not UI, and persist state plus history for audit.
**Follow-up traps**
- *"Invalid transition?"* Return a domain error/exception; don't silently ignore.
- *"Concurrency?"* Optimistic concurrency token on the aggregate.

### 13.2 [Follow-up] In-process state machine vs a durable workflow: how to choose?
**What the interviewer is really testing**: Durability and long-running process reasoning.
**Typical mid-level answer**: "Use Durable Functions." Without criteria.
**Strong senior answer**: Ask about duration, human/external waits, failure recovery, and visibility. Short, request-scoped lifecycle with DB-persisted status: an aggregate state machine. Processes spanning minutes to days with timers, external events (approvals), retries and compensation: use a durable orchestration (Durable Functions/Durable Task Framework, Logic Apps) or a saga state machine (MassTransit Automatonymous, NServiceBus sagas) with persistent saga state. Durable Functions orchestrators must be deterministic (no `DateTime.Now`, random, direct I/O; use activities and `context.CurrentUtcDateTime`) as they replay. Cost: operational model, versioning running instances, and debugging replay behavior.
**Follow-up traps**
- *"Versioning in-flight workflows?"* Side-by-side deployment or versioned orchestrations.
- *"Timers?"* Durable timers vs delayed Service Bus messages (scheduled enqueue).

### 13.3 [Medium] Template Method vs composition.
**What the interviewer is really testing**: Inheritance trade-offs.
**Typical mid-level answer**: "Template Method is fine for shared algorithm." Ignores fragile base class.
**Strong senior answer**: Template Method fixes the skeleton in a base class and lets subclasses fill hooks. Risks: fragile base class (subclasses depend on call order and protected state), hard to test, one inheritance slot used up, and combinatorial subclass growth. Composition passes the varying steps as strategies/delegates into a single concrete class, which is easier to test and recombine. Template Method is still reasonable when the framework dictates it (`BackgroundService.ExecuteAsync`, `Controller` base, `AuthorizationHandler<T>`, xUnit fixtures) or inside a closed hierarchy you own with two or three variants. I'd keep hooks few, abstract, and document invariants.
**Follow-up traps**
- *"Virtual call in constructor?"* Dangerous: derived state not initialized.
- *"`sealed` by default?"* Recommended unless designed for inheritance.

### 13.4 [Hard] Saga as a state machine: pitfalls.
**What the interviewer is really testing**: Distributed workflow rigor.
**Typical mid-level answer**: "Each step publishes the next." Choreography only.
**Strong senior answer**: Pitfalls: lost/duplicated/out-of-order messages (need idempotent transitions and correlation ids), no timeout for waiting states (add timers and a "stuck" policy), compensation not being a perfect inverse (a refund isn't an undo; model business-level compensation), saga state and message publishing not atomic (outbox), concurrency on saga instance (optimistic concurrency/row locks), and visibility (a dashboard of states). Decide choreography (simple, few services, but logic spread out) vs orchestration (central state machine, clearer, but central coupling). Define terminal states and explicit failure paths.
**Follow-up traps**
- *"Pivot step?"* After a pivot transaction, later steps retry forward rather than compensate.
- *"Testing?"* Drive the state machine with test harness (MassTransit) and property tests on transitions.

### 13.5 [Tricky] A teammate used a boolean pile (`IsPaid`, `IsShipped`, `IsCancelled`) for lifecycle. Problem?
**What the interviewer is really testing**: Illegal states representable.
**Typical mid-level answer**: "Replace with an enum." Partial.
**Strong senior answer**: Booleans allow impossible combinations (shipped and cancelled) and the validity rules sit in every reader. Replace with a single state value (enum or state type), enforce legal transitions in one method, and store timestamps per event if needed (`PaidAt`). Make illegal states unrepresentable where feasible; in SQL add a `CHECK` constraint on valid state values and optionally a transitions table or history table. Migration: map existing combos, find already-invalid rows first.
**Follow-up traps**
- *"Overlapping states (paid and shipped)?"* Then they're orthogonal; use separate state dimensions rather than one enum.
- *"Audit?"* Append-only status history table.

**Red flags in answers**
- State pattern class hierarchy for 3 trivial states.
- Non-deterministic code in Durable orchestrators.
- Boolean flags representing a lifecycle.

**Cheat sheet**
- Simple lifecycle: enum + guarded methods; complex: Stateless; long-running: durable workflow/saga.
- Orchestrators deterministic; versioning in-flight instances matters.
- Template Method = fragile base; prefer composition unless framework dictates.
- Sagas: idempotent transitions, timeouts, outbox, explicit compensation.
- Make illegal states unrepresentable; add DB CHECK.

---

## Topic 14: Cloud Resilience Patterns (Retry, Circuit Breaker, Bulkhead, Timeout, Throttling, Rate Limiting)

**Study notes**
- **Retry**: re-attempt transient failures with exponential backoff + jitter, cap attempts; only for idempotent or idempotency-keyed operations. Honor `Retry-After`.
- **Circuit Breaker**: stop calling a failing dependency (Closed -> Open -> Half-Open) to fail fast and let it recover.
- **Bulkhead**: isolate resources (separate pools/concurrency limits) so one failing dependency doesn't exhaust all threads/connections.
- **Timeout**: bound waiting; without it retries and breakers are useless. Propagate `CancellationToken`.
- **Throttling** (server protecting itself) / **Rate Limiting** (limiting caller usage): ASP.NET Core built-in rate limiting middleware (.NET 7+: fixed window, sliding window, token bucket, concurrency).
- .NET: Polly v8 / `Microsoft.Extensions.Resilience`; `AddStandardResilienceHandler()` combines rate limiter, total timeout, retry, circuit breaker, attempt timeout. Azure SDK clients have built-in retry options.
- Retry amplification: nested retries multiply load (3 x 3 x 3 = 27).

```
call -> [total timeout [retry [circuit breaker [attempt timeout -> HTTP]]]]
```

### 14.1 [Easy] When is retry appropriate, and what are the dangers?
**What the interviewer is really testing**: Judgment about idempotency and load amplification.
**Typical mid-level answer**: "Retry 3 times on failure." Dangerous.
**Strong senior answer**: Retry only transient faults (timeouts, 503, 429, deadlocks, connection resets), never validation errors or 4xx logical failures, and only where the operation is idempotent or protected by an idempotency key. Use exponential backoff with jitter to avoid synchronized retry storms, cap attempts and total time, and respect `Retry-After`. Danger: retries amplify load on an already struggling dependency and across layers multiply (each tier retrying), and non-idempotent retries cause duplicate charges/orders. Retry budgets and circuit breakers contain this. Prefer retrying at one layer only, usually closest to the call.
**Follow-up traps**
- *"Retry on SQL deadlock?"* Yes, transient (error 1205), rerun the whole transaction.
- *"EF Core?"* `EnableRetryOnFailure` handles transient SQL/Azure SQL errors.

### 14.2 [Follow-up] Circuit breaker: how does it work and where does it hurt?
**What the interviewer is really testing**: Tuning and failure semantics.
**Typical mid-level answer**: "It opens after errors." No half-open or tuning.
**Strong senior answer**: Closed: calls pass; failures tracked over a sliding window. When failure ratio exceeds threshold (with a minimum throughput), it opens: calls fail immediately for a break duration. Half-open: a probe call tests recovery; success closes, failure reopens. It protects the caller's threads and the dependency's recovery. Hurts when thresholds are too sensitive (flapping on low traffic), when one breaker is shared across unrelated endpoints/tenants (one bad route trips everything), when the fallback behavior is undefined (callers just get exceptions), or when treating business failures (404/400) as breaker failures. Per-dependency (or per-endpoint) breakers, meaningful fallbacks (cached/degraded), metrics and alerts on state changes.
**Follow-up traps**
- *"Per-instance state?"* Each app instance has its own breaker; they open independently.
- *"Breaker with retry order?"* Retry outside the breaker so open circuit stops retries (or breaker outside, depending on intent; be deliberate).

### 14.3 [Medium] Bulkhead and timeout: why are they as important as retry?
**What the interviewer is really testing**: Cascading-failure thinking.
**Typical mid-level answer**: "Timeouts avoid hanging." Incomplete.
**Strong senior answer**: Cascading failures usually start with a slow dependency, not a failed one: requests pile up holding threads, connections, memory until the whole service starves. Timeouts bound the damage per call (set below the caller's own timeout so deadlines nest: client < gateway < service). Bulkheads cap concurrency per dependency (separate `HttpClient` handlers/pools, `SemaphoreSlim`, Polly concurrency limiter, separate queues or worker pools) so a slow payment API can't consume capacity needed for catalog reads. Also avoid sync-over-async which starves the thread pool. `HttpClient.Timeout` defaults to 100 seconds, usually far too long. Pass `CancellationToken` through to DB and HTTP calls.
**Follow-up traps**
- *"Timeout too short?"* Causes retries and duplicate work; base on observed p99.
- *"Where to apply bulkheads for databases?"* Separate connection pools/connection strings or read replicas for reporting.

### 14.4 [Hard] Throttling vs rate limiting vs load shedding: design for a public API on Azure.
**What the interviewer is really testing**: System-level protection design.
**Typical mid-level answer**: "Add rate limiting." Single layer.
**Strong senior answer**: Layered: edge rate limits per client/key/plan at API Management (rate-limit-by-key, quota), app-level ASP.NET Core rate limiter (token bucket for burst, concurrency limiter for expensive endpoints) partitioned by user/tenant/IP, and back-pressure inside via bounded queues and load shedding (return 503/429 quickly when saturated rather than queuing unbounded). Return 429 with `Retry-After`. Different semantic: rate limiting enforces fairness/contract per caller; throttling/shedding protects system health regardless of caller. Distributed enforcement needs shared state (Redis) or accept per-instance approximations; the built-in ASP.NET Core limiter is per-process. Don't forget downstream: Azure SQL, Cosmos RU limits (429 `x-ms-retry-after-ms`), Service Bus quotas.
**Follow-up traps**
- *"Multiple instances?"* Per-instance limiters sum; use APIM/Redis for global limits.
- *"Fairness?"* Partition by tenant to prevent noisy neighbors.

### 14.5 [Scenario] Choose the pattern for retrying payment calls.
**What the interviewer is really testing**: Integrating idempotency, retries and failure semantics on money.
**Typical mid-level answer**: "Polly retry with exponential backoff." Dangerous without idempotency.
**Strong senior answer**: First establish idempotency: send an idempotency key (many PSPs, e.g. Stripe, support an `Idempotency-Key` header) generated once per payment intent and persisted before the first call, so retries after a timeout can't double-charge. Then: short per-attempt timeout, bounded retries with backoff and jitter for transient errors only (timeouts, 5xx, 429), no retry on declines (4xx business), circuit breaker per provider with a defined fallback (queue the payment for later, or fail clearly to the user). For ambiguous outcomes (timeout after sending) don't assume failure: reconcile by querying the provider using the key or via webhook before retrying/charging again. Persist payment state (Pending/Succeeded/Failed/Unknown), emit events through an outbox, and alert on unknown/stuck payments. Use `AddResilienceHandler` rather than hand-rolled loops.
**Follow-up traps**
- *"Retry inside a DB transaction?"* No: don't hold locks while calling a PSP.
- *"Webhook and sync result disagree?"* Make transitions idempotent and order-tolerant; the provider is source of truth.

**Red flags in answers**
- Retrying non-idempotent calls blindly.
- Retries at every layer with no budget.
- No timeouts, relying on defaults (HttpClient 100s).

**Cheat sheet**
- Retry: transient only, backoff+jitter, idempotent/keys, one layer.
- Breaker: closed/open/half-open; per dependency; define fallback.
- Bulkhead + timeout stop cascades; nest deadlines.
- Rate limit (per caller) vs throttle/shed (protect system); 429 + `Retry-After`.
- .NET: `AddStandardResilienceHandler`, ASP.NET Core rate limiter, EF `EnableRetryOnFailure`.
- Payments: idempotency key + reconcile ambiguous outcomes.

---

## Topic 15: Cloud Messaging and Load Patterns

**Study notes** (Queue-Based Load Leveling, Competing Consumers, Priority Queue, Claim Check, Publisher/Subscriber, Pipes and Filters, Cache-Aside)
- **Queue-Based Load Leveling**: queue between producer and consumer absorbs bursts; consumer processes at steady rate. Trade-off: latency, no synchronous result.
- **Competing Consumers**: multiple workers read the same queue; scale out; need idempotency, visibility timeout/lock duration, poison handling. Ordering not guaranteed unless sessions.
- **Priority Queue**: separate queues per priority (Service Bus has no native priority) with more consumers on high priority; beware starvation.
- **Claim Check**: store large payload in Blob, send a reference in the message (Service Bus message size limits: 256 KB standard, up to 100 MB premium).
- **Pub/Sub**: topics + subscriptions with filters.
- **Pipes and Filters**: decompose processing into independent stages connected by queues/streams.
- **Cache-Aside**: read from cache, on miss load from store and populate; invalidate/expire on writes. Azure Cache for Redis.

```
Producer -> [Queue] -> Consumer1
                   \-> Consumer2   (competing consumers; DLQ for poison)
```

### 15.1 [Easy] Queue-Based Load Leveling: what problem does it solve and what does it cost?
**What the interviewer is really testing**: Trade-off awareness (latency/consistency vs resilience).
**Typical mid-level answer**: "Put a queue in front to handle spikes." No costs.
**Strong senior answer**: Bursty producers (web requests, imports) can overwhelm a downstream with fixed capacity (database, third-party API). A queue buffers work and the consumer drains at a rate it can sustain; scale consumers by queue depth (KEDA/Functions scaling). It also decouples availability: producers still accept work when consumers are down. Costs: the response becomes asynchronous (202 Accepted + status endpoint/callback), added end-to-end latency, eventual consistency, need for idempotent consumers, DLQ handling and monitoring of queue age. It doesn't help if sustained arrival rate exceeds processing capacity: the queue just grows. Not for requests needing an immediate answer.
**Follow-up traps**
- *"Unbounded queue growth?"* Set TTL, alerts on age, shed load or scale out.
- *"Service Bus vs Storage Queue?"* Service Bus: sessions, DLQ, transactions, topics, duplicate detection; Storage Queue: simple, cheap, large volume.

### 15.2 [Follow-up] Competing consumers: what must your code do to be safe?
**What the interviewer is really testing**: Concurrency/delivery semantics.
**Typical mid-level answer**: "Just add more instances." Ignores correctness.
**Strong senior answer**: Messages are delivered at-least-once, so handlers must be idempotent (inbox/dedupe, upserts). Peek-lock: complete only after success; processing longer than lock duration causes redelivery, so renew lock or keep handlers short. Configure max delivery count and a DLQ, and monitor DLQ. Ordering is lost across competing consumers; if needed, use sessions (per-entity ordering) or partition keys, which limit parallelism to number of sessions. Control concurrency per instance (`MaxConcurrentCalls`) to avoid overwhelming dependencies, and make consumers stateless for scale-out. Handle poison messages without blocking the queue.
**Follow-up traps**
- *"Prefetch?"* Improves throughput; messages locked in prefetch buffer may expire if processing is slow.
- *"Ordering guarantee for one customer?"* Session id = customer id.

### 15.3 [Medium] Claim Check and large payloads.
**What the interviewer is really testing**: Practical messaging design.
**Typical mid-level answer**: "Put the file in blob and send the URL." Missing details.
**Strong senior answer**: The message carries metadata and a reference (blob URI/id), not the payload, keeping the broker fast and within size limits. Considerations: security (use managed identity or short-lived SAS/Valet Key scoped to the blob rather than long-lived keys), lifecycle (who deletes the blob: consumer after success or lifecycle management policy; orphaned blobs on failures), consistency (upload before enqueue so consumers never see a dangling reference; failure after upload leaves garbage collected by policy), and integrity (hash, content type). Costs: two systems to operate, extra latency. Skip it when payloads fit comfortably in messages.
**Follow-up traps**
- *"Upload then send fails?"* Orphan blob; lifecycle rule cleans up.
- *"Event Grid with blob created?"* Can trigger processing directly with no queue message from producer.

### 15.4 [Hard] Cache-Aside: design details and failure modes.
**What the interviewer is really testing**: Practical caching rigor.
**Typical mid-level answer**: "Check cache, otherwise read DB and store." Naive.
**Strong senior answer**: Details: key design including tenant/version, TTL with jitter to avoid synchronized expiry, invalidate or update on write (delete-then-write race: reader can repopulate stale data; short TTL bounds it), stampede protection (single-flight/lock, `HybridCache`), negative caching for known-missing keys (short TTL), serialization versioning on deployments, and size/eviction policy. Failure: Redis down should degrade to DB with a timeout/breaker, not fail the request, though that can overload the DB, so use a bulkhead. Hot keys, large values, and cache poisoning (caching per-user data under a shared key) are common bugs. Never use cache as the source of truth. Cache-aside is wrong for data requiring strong consistency or highly write-heavy keys.
**Follow-up traps**
- *"Write-through vs cache-aside?"* Write-through updates cache synchronously; cache-aside lazy-loads.
- *"Multi-instance in-memory caches?"* Inconsistent; use distributed cache or short TTL.

### 15.5 [Tricky] Priority Queue and Pipes and Filters: when do they bite?
**What the interviewer is really testing**: Seeing operational consequences.
**Typical mid-level answer**: "Process important messages first." Misses starvation.
**Strong senior answer**: Priority queues starve low-priority work under sustained high-priority load unless you reserve capacity or age priorities; implement with separate queues/subscriptions and weighted consumers. Pipes and Filters gives reusable, independently scalable stages, but each hop adds latency, serialization, queue operations and failure modes; error handling across stages (which stage's DLQ? how to replay?) and end-to-end observability (correlation ids, distributed tracing) become the real work. Often a single process with `Channel<T>` or in-process pipeline is sufficient; distribute only when stages need independent scale/deploy.
**Follow-up traps**
- *"Stage coupling by schema?"* Version message contracts and keep them additive.
- *"Per-stage retries?"* Each stage retries independently; watch amplification.

**Red flags in answers**
- "Queues give exactly-once, so no idempotency needed."
- Queue as a fix for sustained overload.
- Cache without invalidation or TTL strategy.

**Cheat sheet**
- Load leveling: absorbs bursts, adds async/latency; watch queue age.
- Competing consumers: idempotent, lock renewal, DLQ, sessions for order.
- Claim check: blob + reference; valet key; lifecycle cleanup.
- Cache-aside: TTL+jitter, stampede protection, invalidation race, never source of truth.
- Priority queue: avoid starvation. Pipes/filters: observability cost.

---

## Topic 16: Cloud Data and Distributed-Consistency Patterns

**Study notes** (Saga, Compensating Transaction, Event Sourcing, CQRS, Materialized View, Sharding, Index Table, Choreography vs Orchestration, Scheduler Agent Supervisor)
- No distributed ACID across services/PaaS in practice -> use Saga (sequence of local transactions + compensations).
- **Compensating Transaction**: business-level undo of a completed step; must be idempotent and retryable; may not be exact inverse.
- **Event Sourcing**: store events as source of truth; rebuild state by replay; snapshots; great audit; costs: schema evolution, projections, complexity, GDPR erasure difficulty.
- **Materialized View**: precomputed read-optimized data (SQL indexed views, Cosmos change feed projections, Azure Synapse).
- **Sharding**: partition data across stores by key; choose shard key for even distribution; cross-shard queries are hard.
- **Index Table**: secondary index on non-key fields for stores lacking them (e.g., Cosmos/Table Storage patterns).
- **Scheduler Agent Supervisor**: scheduler coordinates steps, agents execute, supervisor monitors and recovers failures -- implemented in practice by Durable Functions.

```
Orchestrator: Reserve -> Charge -> Ship
 failure at Ship => Refund (comp) -> Release (comp)
```

### 16.1 [Easy] Why Saga instead of a distributed transaction?
**What the interviewer is really testing**: Realistic consistency approach.
**Typical mid-level answer**: "Use MSDTC/2PC." Not available/scalable for cloud services.
**Strong senior answer**: Services own their data and use different stores; 2PC needs all participants to support it, holds locks across network calls, and reduces availability, so most PaaS/message brokers don't participate. A Saga splits the business transaction into local transactions; if a step fails, previously completed steps are compensated. It gives availability and autonomy but sacrifices isolation: other readers see intermediate states (needs semantic locks, "pending" statuses). Complexity: compensation design, idempotency, timeouts, observability. Use only when a business process truly spans services; prefer keeping consistency inside one service/aggregate when possible (don't split data to then need sagas).
**Follow-up traps**
- *"Isolation anomalies?"* Dirty reads of intermediate states; use pending states, commutative updates.
- *"Can all steps be compensated?"* No: sending an email can't be undone; order steps with pivot and retryable steps last.

### 16.2 [Follow-up] Choreography vs orchestration: how do you choose?
**What the interviewer is really testing**: Design trade-off reasoning.
**Typical mid-level answer**: "Orchestration is better." Dogmatic.
**Strong senior answer**: Choreography: services react to events; no central coordinator, loose coupling, good for 2-3 participants and simple flows. It gets hard as flows grow: the process is implicit across services, cyclic dependencies appear, and tracing "where is order 123?" is painful. Orchestration: a coordinator (saga state machine, Durable Functions) explicitly sequences steps and compensations, provides one place for process state/timeouts/visibility, at the cost of a central component that knows many services (but only commands, not logic). Rule of thumb: choreography for simple fan-out reactions, orchestration for multi-step business processes with compensation and timeouts. Mixed is common.
**Follow-up traps**
- *"Is orchestrator a god service?"* Only if it contains domain logic; it should only sequence.
- *"Single point of failure?"* Make it durable and scaled; state is persisted.

### 16.3 [Medium] Event Sourcing: when is it worth it and when is it a mistake?
**What the interviewer is really testing**: Resisting hype.
**Typical mid-level answer**: "It gives full audit history." Ignores costs.
**Strong senior answer**: Worth it when history is the domain (ledgers, trading, compliance audit, temporal queries "state as of date"), or when replay for new projections is a business value. Costs: event schema evolution/upcasting forever, projections and eventual consistency, querying requires read models (CQRS practically mandatory), snapshots for long streams, debugging replay, and tooling/ops maturity (Marten, EventStoreDB, custom on SQL). Mistakes: using it as an audit log for CRUD, events that are just row-diffs (`CustomerUpdated` with all fields) instead of intent, exposing internal events as public contract, and GDPR erasure (crypto-shredding or avoid personal data in events). If you just need audit, use temporal tables / CDC / an audit table.
**Follow-up traps**
- *"SQL Server temporal tables?"* System-versioned history is cheaper for audit/as-of queries.
- *"Event versioning?"* Upcasters, additive changes, never mutate stored events.

### 16.4 [Hard] Materialized views, sharding and index tables: pick for a hot read path.
**What the interviewer is really testing**: Data design reasoning with SQL Server/Azure.
**Typical mid-level answer**: "Shard the database." Premature.
**Strong senior answer**: Start with the cheapest fix: proper indexes (covering, filtered), query tuning (Query Store), caching, read replicas (Azure SQL read scale-out). Next a materialized view: SQL Server indexed views (with restrictions, schema binding, write overhead) or a denormalized read table maintained by events/CDC/change feed; trade-off is staleness and maintenance logic. Index Table pattern fits stores with only primary-key access (Cosmos DB, though Cosmos has secondary indexing; Table Storage), maintaining your own lookup table with consistency risks on dual writes. Sharding last: choose shard key by access pattern, avoid hot shards, plan rebalancing and cross-shard queries/transactions (elastic database tools in Azure SQL, Cosmos partition keys). It carries permanent complexity. Measure first.
**Follow-up traps**
- *"Hot partition?"* Bad key choice (e.g., date, tenant skew); use synthetic/composite keys.
- *"Indexed view downsides?"* Slow writes, constraints on query form.

### 16.5 [Scenario] A compensating transaction itself fails. What now?
**What the interviewer is really testing**: Failure-path rigor.
**Typical mid-level answer**: "Retry." Incomplete.
**Strong senior answer**: Compensations must be idempotent and retryable with backoff, since they must eventually succeed. Persist saga state so a restart resumes. After bounded retries, escalate: dead-letter, mark the saga as "needs manual intervention", alert, and give operators tooling (replay, force-complete). Model business impact: if refund fails, the customer was charged: have reconciliation jobs comparing the ledger with provider records (Scheduler Agent Supervisor-style supervisor). Design compensations at the business level (refund, credit note) and record every attempt for audit. Prefer ordering steps to minimize compensations (do uncompensatable steps last).
**Follow-up traps**
- *"Timeout in compensation?"* Treat as unknown and reconcile via query by key.
- *"Observability?"* Correlate all steps by saga/correlation id in traces.

**Red flags in answers**
- Proposing 2PC across Azure PaaS services.
- Event sourcing for audit-only needs.
- Sharding before indexing/caching/replicas.

**Cheat sheet**
- Saga = local transactions + idempotent compensations; sacrifices isolation.
- Choreography simple/few; orchestration for multi-step with timeouts.
- Event Sourcing only when history is the domain; temporal tables for audit.
- Order of fixes: index -> query -> cache -> replica -> materialized view -> shard.
- Compensation failures need reconciliation + manual escalation.

---

## Topic 17: Cloud Topology, Edge and Migration Patterns

**Study notes** (Gateway Aggregation/Offloading/Routing, BFF, Sidecar, Ambassador, Strangler Fig, Federated Identity, Valet Key, Leader Election, Health Endpoint Monitoring, Anti-Corruption Layer)
- **Gateway Offloading**: TLS, auth, rate limiting, caching at gateway (Azure API Management, Application Gateway, Front Door, YARP).
- **Gateway Routing**: one endpoint routes to many services. **Gateway Aggregation**: combine multiple calls into one.
- **BFF**: separate backend per client type (mobile/web).
- **Sidecar**: helper process alongside the app in same pod/host (Dapr, Envoy/service mesh, log shippers). **Ambassador**: sidecar acting as outbound proxy (retry, TLS, monitoring) for legacy clients.
- **Strangler Fig**: route traffic incrementally from legacy to new via a facade/proxy, replacing feature by feature.
- **Federated Identity**: delegate authentication to a trusted IdP (Microsoft Entra ID, OIDC); apps validate tokens.
- **Valet Key**: give clients limited, time-bound direct access to a resource (SAS tokens/user delegation SAS).
- **Leader Election**: one instance coordinates (blob lease, Redis lock, Kubernetes lease); handle leader loss, split-brain, fencing tokens.
- **Health Endpoint Monitoring**: `/health` endpoints (`AddHealthChecks`, liveness vs readiness) probed by load balancers/Front Door/Kubernetes.

```
Clients -> Gateway (auth, routing, throttling) -> Svc A / Svc B / Legacy
Strangler: route /orders -> new service; rest -> legacy
```

### 17.1 [Easy] Strangler Fig: how do you apply it and what can go wrong?
**What the interviewer is really testing**: Incremental migration discipline.
**Typical mid-level answer**: "Rewrite parts gradually." Vague.
**Strong senior answer**: Put a routing facade (YARP, APIM, Front Door) in front of the legacy system, then carve out vertical slices of functionality: build the new implementation, route a path/percent of traffic (feature flags, canary), verify, then retire the legacy part. Prioritize slices by value and low coupling. The hard parts are data: shared database vs sync, dual-write consistency, and ACLs for model translation. Risks: the migration stalls midway leaving two systems forever; routing facade becomes a bottleneck; behavior parity gaps (undocumented legacy behavior) discovered late: use characterization tests/shadow traffic. Set decommission milestones.
**Follow-up traps**
- *"Big-bang rewrite?"* Higher risk, delayed value, requirements drift.
- *"Data ownership during migration?"* One owner per entity at a time; CDC/events for sync.

### 17.2 [Follow-up] Gateway patterns and BFF: benefits and pitfalls.
**What the interviewer is really testing**: Avoiding gateway-as-monolith.
**Typical mid-level answer**: "A gateway gives a single entry point." Surface level.
**Strong senior answer**: Offloading cross-cutting edge concerns (TLS, JWT validation, rate limiting, WAF, caching) to a gateway keeps services simpler and consistent. Routing hides topology, enabling refactoring. Aggregation reduces client chattiness, especially mobile. BFF gives each client type its own shape and release cadence (web team owns web BFF). Pitfalls: the gateway as a central team bottleneck and single point of failure, business logic leaking into it, aggregation coupling to many services and amplifying latency/failures (timeouts, partial responses), duplication across BFFs, and services trusting the gateway only (still validate tokens/authorize in services: defense in depth). Choose APIM for policy-heavy public APIs, YARP for code-first .NET reverse proxy.
**Follow-up traps**
- *"Do services still need auth?"* Yes; zero trust, validate audience/scope.
- *"GraphQL gateway?"* Alternative for aggregation, adds complexity.

### 17.3 [Medium] Sidecar, Ambassador: when worth it?
**What the interviewer is really testing**: Platform pattern literacy without hype.
**Typical mid-level answer**: "For microservices with Kubernetes." Vague.
**Strong senior answer**: Sidecar packages cross-cutting capability (mTLS, telemetry, config, secrets refresh, pub/sub abstraction with Dapr) as a separate process sharing the app's lifecycle, language-agnostic and independently upgradable. Ambassador is a sidecar specialized for outbound calls (retry, circuit breaking, routing, auth), useful for legacy or third-party binaries that can't be changed. Costs: resource overhead per pod, extra network hop, operational complexity (mesh debugging, version skew), more moving parts for small systems. For a handful of .NET services, in-process libraries (`Microsoft.Extensions.Resilience`, OpenTelemetry SDK) are simpler. Adopt when polyglot fleet or platform team standardization justifies it.
**Follow-up traps**
- *"Service mesh necessary?"* Not for small systems.
- *"Startup order?"* App may start before sidecar is ready; readiness gating needed.

### 17.4 [Hard] Leader Election: design and failure modes.
**What the interviewer is really testing**: Distributed coordination awareness.
**Typical mid-level answer**: "Use a lock in the database." Ignores expiry and fencing.
**Strong senior answer**: Use a lease (Blob lease, Redis with expiring key, Kubernetes Lease, Service Fabric/Orleans primitives): leader renews periodically; if it stops, the lease expires and another instance acquires it. Failure modes: leader paused (GC, network) but still believes it is leader after lease expiry -> split-brain; mitigate with fencing tokens checked by the downstream resource, short critical sections, and renew-failure handling that stops work immediately. Clock skew, lease duration vs renewal interval tuning, and thundering herd on expiry. Often you don't need election: use competing consumers with idempotency, or schedulers like Azure Functions timer triggers (which use blob lease singleton internally) or Kubernetes CronJobs. Use election only for truly singleton coordination work.
**Follow-up traps**
- *"Hangfire/Quartz clustering?"* DB-based locks provide similar singleton job behavior.
- *"Exactly one execution?"* Not guaranteed; make jobs idempotent.

### 17.5 [Scenario] Design health checks for an app behind Azure Front Door/Kubernetes.
**What the interviewer is really testing**: Liveness vs readiness subtlety.
**Typical mid-level answer**: "A /health endpoint that checks the database." Can cause cascading restarts.
**Strong senior answer**: Separate liveness (process is alive, no deps; failure -> restart) from readiness (can serve traffic: critical dependencies reachable; failure -> remove from rotation, don't restart). Using DB checks in liveness means a DB outage restarts all pods, making recovery worse. Use `AddHealthChecks()` with tags, `MapHealthChecks("/health/ready", predicate)`; keep checks fast with timeouts and caching of results; protect detailed output (don't expose internals publicly); include startup probes for slow init. Distinguish critical from optional dependencies so a degraded optional one doesn't take the app out. Pair with synthetic monitoring and alerts.
**Follow-up traps**
- *"Health check spamming DB?"* Cache results for a few seconds.
- *"Deep vs shallow?"* Shallow on the LB path, deep on a protected diagnostics endpoint.

### 17.6 [Tricky] Federated Identity and Valet Key: security trade-offs.
**What the interviewer is really testing**: Secure delegation.
**Typical mid-level answer**: "Use SAS for blob uploads and Entra for login." No details.
**Strong senior answer**: Federated identity delegates authentication to Entra ID / external IdPs via OIDC/OAuth2; apps validate token signature, issuer, audience, expiry, scopes/roles and never handle passwords. Pitfalls: trusting tokens without audience validation, overly broad scopes, long token lifetimes, tenant confusion in multi-tenant apps. Valet Key (SAS): issue short-lived, least-privilege, single-blob tokens, preferably user delegation SAS (backed by Entra credentials) over account-key SAS; the app generates it server-side after authorizing the user, so client uploads bypass your API bandwidth. Risks: leaked SAS URLs are bearer tokens, so short expiry, minimal permissions, IP restrictions where possible, and stored access policies for revocation (service SAS). Prefer managed identities for service-to-service.
**Follow-up traps**
- *"Revoke a SAS?"* Ad-hoc SAS cannot be revoked except by key rotation; stored access policy or user delegation key revocation helps.
- *"Validate uploads?"* Scan/validate content after upload (event-driven), don't trust the client.

**Red flags in answers**
- Gateway containing business logic.
- Database check in a liveness probe.
- Account-key SAS with long expiry.

**Cheat sheet**
- Strangler: facade + slice-by-slice + decommission date; watch data ownership.
- Gateway: offload/route/aggregate; BFF per client; services still validate tokens.
- Sidecar/Ambassador: platform capability with overhead; don't adopt for 3 services.
- Leader election: lease + fencing; prefer idempotent competing consumers.
- Liveness != readiness.
- Valet key: short-lived, least privilege, user delegation SAS.

---

## Topic 18: DDD Tactical Patterns

**Study notes**
- **Entity**: identity + lifecycle. **Value Object**: defined by attributes, immutable, equality by value (C# `record`/`record struct`). **Aggregate**: consistency boundary with a root; external references by id only; one aggregate per transaction. **Domain Service**: logic not belonging to one entity. **Factory**: complex creation. **Repository**: per aggregate root. **Domain Event**: fact that happened.
- Rich domain model: behavior with data, invariants enforced inside. Anemic: entities are property bags, logic in services (Fowler calls it an anti-pattern, but it is sometimes pragmatic for CRUD).
- Strategic DDD (bounded contexts, ubiquitous language) matters more than tactical patterns.
- EF Core supports owned types, value converters, private setters/backing fields, complex types (EF 8).

```
Order (root) --owns--> OrderLine (entity), Address (value object)
Order --references by id--> CustomerId  (not Customer object)
```

### 18.1 [Easy] Entity vs Value Object with an example.
**What the interviewer is really testing**: Core modeling.
**Typical mid-level answer**: "Entities have IDs, value objects don't." Misses immutability/equality and usefulness.
**Strong senior answer**: An entity is tracked over time by identity (a `Customer` stays the same customer when the address changes). A value object is interchangeable if attributes are equal and is immutable (`Money(100, "USD")`, `Address`, `DateRange`). Value objects kill primitive obsession: `Money` prevents adding USD to EUR, `EmailAddress` validates once at creation. In C#, `record` gives value equality and `with`; map in EF Core as owned types/complex types or converters. Caveat: don't make everything a value object; wrapping each string in a type bloats code without invariants to protect.
**Follow-up traps**
- *"Is Address always a value object?"* Depends on the domain: for a shipping snapshot yes; for a location tracked in a registry maybe an entity.
- *"Records and EF?"* Owned types work; entities as records (reference equality needs) are usually avoided.

### 18.2 [Follow-up] How do you decide aggregate boundaries?
**What the interviewer is really testing**: Consistency vs performance reasoning.
**Typical mid-level answer**: "Group related entities." Leads to giant aggregates.
**Strong senior answer**: Boundary = set of objects that must be *transactionally consistent* for an invariant. Start small: one root and what it must protect (an Order and its lines enforce a max total; Customer isn't inside). Reference other aggregates by id. Reasons for small aggregates: less lock/concurrency contention (optimistic concurrency conflicts on big aggregates), cheaper loads, easier scaling. Cross-aggregate rules are eventually consistent through domain events. Signs of wrong boundaries: needing to modify two aggregates in one transaction constantly (merge or rethink), huge graphs loaded with many includes, or frequent concurrency exceptions.
**Follow-up traps**
- *"Query across aggregates?"* Read models, not aggregates.
- *"Rule needing both aggregates?"* Maybe it's one aggregate, or accept eventual consistency with compensation.

### 18.3 [Medium] Anemic vs rich domain: which and when?
**What the interviewer is really testing**: Pragmatism over purity.
**Typical mid-level answer**: "Rich is always better." Dogmatic.
**Strong senior answer**: Rich models shine where there are real invariants and state-dependent rules (pricing, order lifecycle, policy rules): behavior lives next to data, invariants can't be bypassed (private setters, factory/creation methods), and logic is unit-testable without infrastructure. For CRUD-heavy, data-in/data-out parts, an anemic model with services or direct queries is simpler and cheaper; forcing rich objects is ceremony. Typically a system has a core domain deserving rich modeling and supporting subdomains that don't. Pitfall in rich models: entities needing external services (pass as method parameters or use domain services, not injecting repositories into entities). Pitfall in anemic: duplicated rules across services and invalid states.
**Follow-up traps**
- *"Where does validation live?"* Input shape validation at the edge; invariants in the domain.
- *"EF friendliness?"* Private setters/backing fields work with EF Core configuration.

### 18.4 [Hard] When is DDD overkill?
**What the interviewer is really testing**: Cost-awareness.
**Typical mid-level answer**: "For small apps." Not enough.
**Strong senior answer**: DDD pays off in complex domains where the business rules are the difficulty and require deep collaboration with domain experts. Overkill for CRUD, reporting, data pipelines, simple integrations, or short-lived apps where the complexity is technical, not business. Costs: learning curve, modeling effort, more types/mapping, and slower early delivery. Adopt selectively: use ubiquitous language and bounded contexts broadly (cheap, high value), tactical patterns (aggregates, VOs) only in the core domain. A risk is "DDD-lite" cargo cult: repositories, factories and services everywhere with no domain insight, plus anemic entities. Evaluate: are the rules complex, changing and valuable? Do we have access to domain experts?
**Follow-up traps**
- *"Bounded contexts and microservices?"* A bounded context is a model boundary; microservice boundaries often align but aren't identical.
- *"Strategic vs tactical?"* Strategic (context mapping) first.

### 18.5 [Tricky] Should entities raise domain events, and how do they reach handlers with EF Core?
**What the interviewer is really testing**: Integration details.
**Typical mid-level answer**: "Call mediator from the entity." Couples domain to infrastructure.
**Strong senior answer**: Entities/aggregates add events to an internal list (`AddDomainEvent`) with no infrastructure dependency. After/before persistence, the unit of work collects events from tracked entities (`ChangeTracker.Entries<AggregateRoot>()`), clears them and dispatches them (pre-commit for in-transaction handlers; post-commit/outbox for side effects). Pitfalls: events lost on crash if dispatched in-memory after commit; handlers raising further events (loops); ordering and idempotency; serializing events for outbox needs stable contracts; domain events are internal facts, whereas integration events are a public contract and should be mapped deliberately.
**Follow-up traps**
- *"Dispatch in SaveChanges override?"* Common, but hides behavior; document or use a behavior.
- *"Same event class for domain and integration?"* Avoid; leaks internal model.

**Red flags in answers**
- One giant aggregate holding the whole customer graph.
- Repositories injected into entities.
- Applying all tactical patterns to a CRUD app.

**Cheat sheet**
- Entity (identity) vs value object (immutable, record) vs aggregate (consistency boundary).
- Small aggregates; reference by id; eventual consistency across.
- Rich in core domain, anemic OK in supporting/CRUD.
- Strategic DDD (language, contexts) > tactical patterns.
- Domain events: collect in aggregate, dispatch via UoW; outbox for side effects.

---

## Topic 19: Clean / Hexagonal / Onion Architecture, Layered vs Vertical Slice

**Study notes**
- Common idea: dependencies point inward toward the domain; infrastructure (DB, web, queues) are plugins behind ports (interfaces) with adapters.
- Classic layered (N-tier): UI -> BLL -> DAL; tends to couple everything to the DB at the bottom.
- Clean/Onion: Domain <- Application <- Infrastructure/Presentation. Typical .NET solution: `Domain`, `Application`, `Infrastructure`, `Web/Api`.
- Vertical Slice: organize by feature (command/query + handler + validator + endpoint together); each slice picks its own implementation depth.
- Folder structure debates: by technical type (Controllers/Services/Repositories) vs by feature (Orders/, Customers/).
- Enforce with architecture tests; keep it as light as the domain needs.

```
Layered:   UI -> BLL -> DAL -> DB
Clean:     Web --> Application --> Domain <-- Infrastructure
Slice:     /Features/PlaceOrder/{Endpoint, Handler, Validator, Model}
```

### 19.1 [Easy] What is the core idea of Clean/Hexagonal architecture?
**What the interviewer is really testing**: Dependency direction, not folders.
**Typical mid-level answer**: "Four projects: Domain, Application, Infrastructure, API." Structure without reason.
**Strong senior answer**: Business rules should not depend on delivery or persistence mechanisms. The domain/application core defines ports (interfaces for what it needs: store orders, send email), and infrastructure supplies adapters. Dependencies point inward, so the core is testable without a DB or web server and details (ORM, broker, UI framework) can change with less ripple. Projects are only one way to enforce this; the discipline is the point. Costs: indirection, mapping between layers, and boilerplate; benefit is greatest with rich, long-lived domain logic.
**Follow-up traps**
- *"Does Application reference EF Core?"* Purists say no; pragmatists allow `DbContext` in application layer to avoid repository ceremony; a team decision with trade-offs.
- *"Hexagonal vs Clean?"* Same family: ports/adapters vs concentric layers.

### 19.2 [Follow-up] Vertical slice vs layered: how do you choose?
**What the interviewer is really testing**: Cohesion reasoning.
**Typical mid-level answer**: "Vertical slice is modern." Fashion.
**Strong senior answer**: Layered organizes by technical concern, so a feature change touches many folders/projects, but shared rules are centralized. Vertical slices organize by use case: high cohesion, changes localized, easy to delete a feature, and each slice can choose its own approach (simple query with Dapper vs rich domain command). Weaknesses: risk of duplicated logic across slices (mitigate by extracting shared domain logic when real), inconsistent styles without guidelines, and weaker enforcement of dependency direction. They combine well: slices at the application layer around a shared domain core. For small/medium line-of-business apps I prefer slices; for a complex core domain, a protected domain model that slices call.
**Follow-up traps**
- *"Shared code between slices?"* Extract after duplication is proven; domain logic goes into the domain.
- *"Migration from layered?"* Incrementally by feature.

### 19.3 [Medium] Critique "12 layers" over-engineering: what is each layer buying?
**What the interviewer is really testing**: Ability to evaluate architecture with evidence.
**Typical mid-level answer**: "Too many layers." No analysis.
**Strong senior answer**: For each layer ask: what decision or axis of change does it isolate, and has that change happened? Forwarding layers (Controller -> Manager -> Service -> Facade -> Repository -> DAL) add mapping (DTO -> Model -> Entity -> DTO), file count and stack depth without isolating a real variation. Metrics: files touched per feature, lines of pure forwarding code, onboarding time, bugs from mapping drift, test mocks that mirror implementation. Remedy: collapse layers that only delegate, keep boundaries with real reasons (I/O seams, bounded contexts, deployment units), consolidate mappings, and test via behavior. Do it incrementally by feature, with characterization tests.
**Follow-up traps**
- *"Isn't it risky to remove layers?"* With tests and small steps, it reduces risk long term.
- *"Same DTO across layers?"* Acceptable inside one bounded context when shapes match; map at boundaries.

### 19.4 [Hard] How do you enforce architecture so it doesn't erode?
**What the interviewer is really testing**: Governance.
**Typical mid-level answer**: "Code review." Inconsistent.
**Strong senior answer**: Automate: architecture tests (NetArchTest/ArchUnitNET) asserting Domain doesn't reference Infrastructure, handlers don't reference controllers, naming/namespace rules; project references as the first line (Domain has no references); analyzers/`Directory.Build.props` for rules; dependency graphs in CI. Document decisions in ADRs and diagram with C4 sparingly. Measure drift (coupling metrics) occasionally. Rules should be few and valuable: over-constrained architecture tests become a nuisance and get ignored.
**Follow-up traps**
- *"Modular monolith?"* Enforce module boundaries with separate projects/internal visibility and tests; communicate via public contracts/events.
- *"Package-by-layer violation?"* Test namespace dependencies.

### 19.5 [Tricky] Is a modular monolith better than microservices for a new product?
**What the interviewer is really testing**: Architectural maturity.
**Typical mid-level answer**: "Microservices scale better." Ignores costs.
**Strong senior answer**: Start with a modular monolith when the domain is not yet well understood, team is small and deployment independence isn't a requirement: one deployable, strong module boundaries (own schema/tables, public API per module, no cross-module DB joins), in-process calls or events. It gives refactorability and low operational cost, and modules can be extracted later if scale/team autonomy demands. Microservices bring network failure modes, distributed data consistency, operational overhead (observability, deployment pipelines) and are justified by independent scaling/deployment, team size, and fault isolation. Premature microservices often produce a distributed monolith.
**Follow-up traps**
- *"How to keep modules honest?"* Architecture tests + separate DbContexts/schemas per module.
- *"Extraction later?"* Easier if modules already communicate via contracts/events.

**Red flags in answers**
- Architecture defined by folder names, not dependency direction.
- Defending 12 forwarding layers by "separation of concerns".
- Microservices as default for a new product.

**Cheat sheet**
- Dependencies point to the domain; ports/adapters.
- Layers are justified by axes of change that really occur.
- Vertical slices for cohesion; shared domain core for rules.
- Enforce with architecture tests, ADRs.
- Modular monolith first unless independent deploy/scale is needed.

---

## Topic 20: Composition over Inheritance, Immutability, API Design Patterns

**Study notes**
- Inheritance = tight coupling (fragile base class, LSP risks); composition = assemble behavior from collaborators. Keep inheritance for true is-a with stable base.
- Immutability: records, `init`, `readonly`, `ImmutableArray`; thread-safe, simpler reasoning, easier caching; costs: allocations, copy-on-write ceremony, ORM friction.
- Idempotency: safe to repeat. HTTP GET/PUT/DELETE idempotent by definition; POST needs `Idempotency-Key`.
- Pagination: offset (simple, unstable under changes, slow at large offsets) vs keyset/cursor (stable, efficient with index).
- Versioning: URL, header, query; additive evolution first; ASP.NET API Versioning package.
- Other: ETag/If-Match optimistic concurrency, ProblemDetails, 202 + status resource for long operations, 429 + Retry-After.

```
GET /orders?limit=50&after=cursor  ->  { items:[...], next:"opaque-cursor" }
```

### 20.1 [Easy] Why prefer composition over inheritance?
**What the interviewer is really testing**: Concrete trade-offs.
**Typical mid-level answer**: "Inheritance is bad." Absolute.
**Strong senior answer**: Inheritance binds subclass to base implementation details: changes to the base ripple (fragile base class), a single inheritance slot, and behavior combinations explode into subclasses (`LoggingCachingRepo`). Composition lets you assemble behaviors at runtime (decorators, strategies), swap them in tests and avoid LSP traps. Inheritance remains right for genuine is-a with a stable abstraction and shared invariants (exceptions hierarchy, framework base classes like `BackgroundService`, `DbContext`). Practical rules: seal classes by default, inherit from abstractions (interfaces/abstract classes designed for it), and share code via composition/helpers or extension methods.
**Follow-up traps**
- *"Default interface methods?"* Useful for evolution of interfaces, not as multiple inheritance substitute.
- *"Abstract base for DRY?"* Often a smell; extract a collaborator instead.

### 20.2 [Follow-up] Where does immutability help and hurt?
**What the interviewer is really testing**: Pragmatic functional style.
**Typical mid-level answer**: "Immutable is always better." Ignores costs.
**Strong senior answer**: Helps: thread-safety without locks, no accidental mutation, safe sharing/caching, predictable value objects/DTOs/messages/events/config. Records with `init`/`with` make it cheap in C#. Hurts: large objects copied frequently (allocation/GC pressure in hot paths), ORM entities that need tracked mutation (EF change tracking works with mutable entities; immutable entities require replacement semantics), deep object graphs, and ergonomics for builders. Use immutable for messages, value objects, options, and query results; allow controlled mutation inside aggregates through methods that enforce invariants. Beware shallow immutability: a record holding a `List<T>` is still mutable; use `IReadOnlyList`/`ImmutableArray` (and defensive copies).
**Follow-up traps**
- *"Records with EF Core?"* Works for owned types/projections; entity records problematic due to value equality and tracking.
- *"`readonly struct`?"* Avoids defensive copies, good for small value types.

### 20.3 [Medium] Idempotency in API design: how do you implement it for POST?
**What the interviewer is really testing**: Reliability patterns in APIs.
**Typical mid-level answer**: "POST isn't idempotent so can't." Gives up.
**Strong senior answer**: Client sends an `Idempotency-Key` (a UUID) per logical operation. Server stores key + request fingerprint + response status/body (in SQL with a unique index, ideally in the same transaction as the business change) and returns the stored response on a repeat; a same-key different-payload request returns 422/409. Handle concurrency: two simultaneous requests with the same key (unique constraint -> one waits/receives 409 "in progress"). Set a retention window (e.g., 24h) and clean up. Alternatively make the operation naturally idempotent (PUT with a client-supplied id; upsert). This pairs with retries: clients and gateways can safely retry after timeouts. Cost: storage, extra complexity, and defining semantics for partial failures.
**Follow-up traps**
- *"Key stored after processing?"* Race: crash between effect and key record; store atomically.
- *"PUT vs POST for create?"* Client-generated id + PUT gives natural idempotency.

### 20.4 [Hard] Offset vs keyset pagination in SQL Server-backed APIs.
**What the interviewer is really testing**: SQL performance and API contract.
**Typical mid-level answer**: "`Skip`/`Take`." Doesn't scale.
**Strong senior answer**: `OFFSET/FETCH` (EF `Skip/Take`) is simple and allows jumping to page N, but cost grows with offset (the engine reads and discards rows) and results shift (duplicates/missing rows) when data changes between requests; it also needs a deterministic `ORDER BY` including a unique tiebreaker. Keyset (seek) pagination uses `WHERE (CreatedAt, Id) < (@lastCreatedAt, @lastId) ORDER BY CreatedAt DESC, Id DESC` backed by a matching index: constant-time page cost, stable under inserts, but no random page jump and needs an opaque cursor token that encodes the last key. Choose keyset for feeds/large datasets/infinite scroll and exports; offset for small admin grids needing page numbers. Avoid `COUNT(*)` on every call for large tables; return `hasMore` or an approximate count.
**Follow-up traps**
- *"Sorting by non-unique column?"* Add Id as tiebreaker.
- *"Cursor tampering?"* Treat as opaque; validate; don't encode sensitive data.

### 20.5 [Tricky] API versioning strategy: URL vs header vs none?
**What the interviewer is really testing**: Evolution and compatibility thinking.
**Typical mid-level answer**: "Put /v1 in the URL." Without a compatibility strategy.
**Strong senior answer**: The best versioning is minimizing breaking changes: additive fields, tolerant readers, new optional parameters, expand-and-contract for removals, and deprecation headers with sunset policy. When breaking changes are unavoidable, versions are needed: URL segment (visible, cacheable, easy routing, most common), header/media type (cleaner URLs, harder to test/discover). Pick one convention org-wide and support it with the ASP.NET API Versioning library plus OpenAPI doc per version. Cost: each live version multiplies testing and maintenance, so limit concurrent versions and publish a retirement timeline. For internal APIs with controlled clients, consumer-driven contract tests (Pact) can replace versions with coordinated change.
**Follow-up traps**
- *"What counts as breaking?"* Removing/renaming fields, changing types/semantics, tightening validation, new required inputs.
- *"Event/message versioning?"* Same principle: additive, upcasters.

**Red flags in answers**
- Deep inheritance for code reuse.
- Offset pagination on millions of rows with no order tiebreaker.
- "POST can't be idempotent."

**Cheat sheet**
- Composition for behavior reuse; inherit only for stable is-a; seal by default.
- Immutability for messages/value objects; beware shallow immutability.
- Idempotency-Key stored atomically with the effect.
- Keyset pagination for scale: index + opaque cursor.
- Minimize breaking changes first; version second; limit live versions.

---

## Topic 21: Refactoring Toward Patterns and Recognising Anti-Patterns

**Study notes**
- Anti-patterns: **God class** (does everything), **Big Ball of Mud** (no structure), **Golden Hammer** (one tool for all), **Leaky Abstraction** (details bleed through; Spolsky's law: all non-trivial abstractions leak), **Primitive Obsession** (strings/ints instead of domain types), **Shotgun Surgery** (one change, many places), **Anemic Domain Model**, **Lava Flow** (dead code nobody dares remove), also Feature Envy, Divergent Change, Spaghetti code.
- Refactor under test: characterization tests, small steps, Strangler/branch by abstraction, Parallel Change (expand-contract).
- Refactoring to patterns (Kerievsky): let the pattern emerge from duplication/smell.
- Measure: churn x complexity hotspots (git history), coverage, dependency graphs.

```
Smell -> characterization tests -> small refactor steps -> introduce seam/pattern -> remove old
```

### 21.1 [Easy] Name five anti-patterns and how you'd spot them in a .NET codebase.
**What the interviewer is really testing**: Practical recognition.
**Typical mid-level answer**: Lists names without detection.
**Strong senior answer**: God class: `OrderManager` with 3000 lines, 20 constructor dependencies, touched by every PR (spot via churn and size). Primitive obsession: `string customerId, string orderId` swapped silently, `decimal amount` with currency separate; fix with strongly typed ids/Money. Shotgun surgery: adding an order status touches 12 files with the same `switch`. Leaky abstraction: `IRepository` returning `IQueryable` forcing callers to know EF translation limits. Lava flow: unused feature flags, `// TODO legacy` classes with no references or tests, but no one deletes them. Golden hammer: using MediatR/microservices/stored procedures for everything. Each is detected by smells in metrics or change patterns, not opinion.
**Follow-up traps**
- *"Is a stored-procedure-heavy design an anti-pattern?"* Not inherently; it's a trade-off; problem is when logic is untested and duplicated.
- *"Is duplication always bad?"* Wrong abstraction costs more than duplication.

### 21.2 [Follow-up] How do you refactor a God class safely?
**What the interviewer is really testing**: Method under uncertainty.
**Typical mid-level answer**: "Split it into smaller classes." No safety net.
**Strong senior answer**: First, characterization tests around current behavior (including awkward cases) at the highest feasible level (API/integration), since unit-level seams may not exist. Then identify clusters by cohesion: methods using the same fields and by reasons to change (git blame/history). Extract in small, behavior-preserving steps using the IDE (Extract Method/Class, Introduce Parameter Object), keeping the old class as a facade delegating to new collaborators to avoid breaking callers (Branch by Abstraction), migrate callers, then delete the shell. Separate pure logic from I/O to make unit tests possible. Commit frequently, ship incrementally behind flags when risk is high, and don't mix refactoring with feature changes in one PR.
**Follow-up traps**
- *"No tests and no time?"* Add tests only around the area to change; golden-master/approval testing with captured outputs.
- *"Rename-only first?"* Yes: safe wins build confidence.

### 21.3 [Medium] How do you tackle Primitive Obsession and Anemic models without a rewrite?
**What the interviewer is really testing**: Incremental modernization.
**Typical mid-level answer**: "Introduce value objects." No migration path.
**Strong senior answer**: Start at the boundaries where bugs have occurred: introduce strongly typed ids (`readonly record struct OrderId(Guid Value)`) with EF value converters and JSON converters so persistence and contracts don't change. Add `Money`/`EmailAddress` with validation at creation. Move logic next to its data one rule at a time: when touching a service method that switches on entity state, push it into the entity method and keep a thin service. Use Parallel Change: add the new method, migrate callers, remove old. Compiler errors guide the migration. Avoid wrapping everything; target types where mix-ups or duplicated validation exist.
**Follow-up traps**
- *"Perf of struct ids?"* Negligible; record structs avoid allocations.
- *"API contract impact?"* Custom `JsonConverter` keeps wire format as raw Guid.

### 21.4 [Hard] How do you detect and remove Lava Flow and dead code in a large legacy system?
**What the interviewer is really testing**: Risk management.
**Typical mid-level answer**: "Delete unused code." Risky without evidence.
**Strong senior answer**: Gather evidence: static analysis (IDE0051/CA1812 unused members, analyzers, NDepend), code coverage in production-like runs, telemetry (add logging/metrics on suspicious paths, Application Insights usage data), feature flag status, DB table/column access (SQL Server DMVs for index/usage stats since last restart, Query Store), and git history. Delete in small PRs, with tag/rollback ability (git preserves history), starting with the clearly unreachable (no references), then "probably unused" behind a log-then-throw period. For public APIs/libraries, mark `[Obsolete]` and measure callers. Communicate with stakeholders for business features (reports, quarterly jobs may only run rarely). Don't forget reflection/DI/config-driven references and scheduled jobs.
**Follow-up traps**
- *"Rare jobs?"* Observe over a full business cycle before deletion.
- *"Dead DB objects?"* DMV counters reset on restart; use extended events/audit over time.

### 21.5 [Tricky] "Don't refactor working code." Respond.
**What the interviewer is really testing**: Business-aware engineering.
**Typical mid-level answer**: "Technical debt must be paid." Not persuasive.
**Strong senior answer**: Agree partially: refactoring has cost and risk, so tie it to a purpose: upcoming change in that area, a defect hotspot, or a bottleneck to velocity. Use data: files with high churn and high complexity and frequent bugs justify investment; stable ugly code nobody touches doesn't. Refactor as part of feature work (opportunistic, scoped), with tests and in separate commits; budget larger structural refactors explicitly with measurable outcomes (lead time, change-failure rate). Communicate in business terms: reduced incident rate, faster delivery, lower onboarding cost.
**Follow-up traps**
- *"How to measure benefit?"* DORA metrics, cycle time of changes in that area, defect rates.
- *"Boy-scout rule risk?"* Keep changes small and reviewed.

**Red flags in answers**
- Big-bang rewrites as the first answer.
- Refactoring without tests or a safety net.
- Naming anti-patterns but unable to show detection or a fix path.

**Cheat sheet**
- Smells are found via churn, size, duplication, and change patterns.
- Characterization tests -> small steps -> parallel change -> delete old.
- Strongly typed ids with converters keep contracts stable.
- Dead code: evidence (analysis + telemetry), staged removal.
- Refactor where change/defects concentrate; separate PRs.

---

## Topic 22: Scenario Round and the 60-Second Pattern Explanation

**Study notes**
- Scenario answers follow: clarify context -> name forces -> options with trade-offs -> recommendation -> risks/measurement -> reversibility.
- The 60-second pattern explanation: **Problem (10s) -> Pattern idea (15s) -> Real example (20s) -> Trade-off/when not (15s)**.
- Show you'd choose the *simplest thing that works* and know how it would evolve.

```
Context -> Forces -> Options (2-3) -> Choice + why -> Cost/risk -> How we'd revisit
```

### 22.1 [Scenario] "Review this codebase with 12 layers." What do you do in the first week?
**What the interviewer is really testing**: Structured evaluation and tact.
**Typical mid-level answer**: "Remove layers." Jumping to a solution.
**Strong senior answer**: Week 1 is learning, not judging. Trace two or three real features end to end, counting files, mappings and forwarding calls. Measure: git churn per layer, files touched per change, defect clusters, test pyramid, build and onboarding time. Ask the authors what each layer was meant to isolate, then check whether that change ever happened. Classify layers: justified (I/O seam, bounded context, separate deployable), neutral, or pure forwarding. Present findings with data and one small proposal: collapse the worst forwarding layer on the next feature, with tests, and measure the outcome. Avoid a big-bang rewrite; agree on guidelines (new layer needs a stated reason) and add architecture tests.
**Follow-up traps**
- *"The team is attached to it?"* Run an experiment on a single feature and compare.
- *"What if some layers are mandated by compliance?"* Keep them, but make them thin.

### 22.2 [Scenario] "Justify adding an abstraction." What's your checklist?
**What the interviewer is really testing**: Disciplined design decision making.
**Typical mid-level answer**: "For testability and flexibility." Vague.
**Strong senior answer**: I need yes to at least one of: (1) a real second implementation exists or is committed (provider swap, per-tenant behavior), (2) it sits on an I/O or volatile boundary where a test double is needed (clock, HTTP, queue, payment), (3) it protects a stable domain from an unstable/poor external model (ACL), (4) it enforces a dependency direction between modules/teams. And the cost is acceptable: what new indirection, mapping, or config will this add? Does it leak (mirroring the vendor API)? Who owns it? If none apply, I write the concrete code and refactor later because that's cheap. I'd also express the abstraction in the consumer's vocabulary and keep it narrow (ISP).
**Follow-up traps**
- *"Isn't it cheaper to add it now?"* Not if the guess is wrong; wrong abstractions are expensive to unwind.
- *"Unit tests need mocks?"* Prefer integration tests for owned infrastructure; mock only external boundaries.

### 22.3 [Scenario] The team argues Repository vs DbContext. Walk through your facilitation.
**What the interviewer is really testing**: Leadership with technical substance.
**Typical mid-level answer**: "I'd explain why mine is right." Position-based.
**Strong senior answer**: I'd restate each camp's concern as a requirement (testability, swap-ability, query reuse, simplicity) and list the context (domain richness, stores, test strategy). Then offer a decision matrix: complex aggregates with invariants -> aggregate repositories; CRUD/query-heavy slices -> `DbContext` directly with projections; shared predicates -> specifications or named query extension methods on `IQueryable<T>`. Time-box a spike on one real feature, compare code size, test speed and clarity. Capture the outcome in an ADR with revisit criteria and enforce via architecture test. The aim is consistent, reasoned decisions, not a winner.
**Follow-up traps**
- *"Team split 50/50?"* Go with the reversible option and revisit with data.
- *"Existing code mixed?"* Don't mass-migrate; apply to new code and touched areas.

### 22.4 [Scenario] Choose the pattern(s) for retrying payment calls. Summarize in 90 seconds.
**What the interviewer is really testing**: Applying and combining resilience patterns correctly.
**Typical mid-level answer**: "Retry with exponential backoff." Missing idempotency and ambiguity.
**Strong senior answer**: Pattern combo, in order of importance: Idempotency key per payment intent (persisted before the call); timeout per attempt and overall; Retry with exponential backoff and jitter only for transient failures; Circuit Breaker per provider to fail fast; a defined fallback (queue-based deferral or clear user error); reconcile ambiguous outcomes by querying the provider/webhook before re-charging; Outbox for resulting events. Payment state machine (Pending/Succeeded/Failed/Unknown) provides visibility. Don't retry declines, don't hold DB transactions across the call, and don't stack retries in several layers. Implement with `AddResilienceHandler`/`AddStandardResilienceHandler` rather than custom loops, and alert on Unknown states.
**Follow-up traps**
- *"What if the provider lacks idempotency keys?"* Use your own dedupe via a unique reference passed in metadata and lookup before retry.
- *"User sees a spinner for 30s?"* Return 202 + status polling and process asynchronously.

### 22.5 [Medium] Explain the Circuit Breaker pattern in 60 seconds with a real example.
**What the interviewer is really testing**: Concise clarity with a concrete story.
**Typical mid-level answer**: A rambling definition ("it's like an electrical breaker...").
**Strong senior answer**: "When a downstream service is failing, continuing to call it wastes threads and makes recovery worse. A circuit breaker watches failure rates; past a threshold it opens and fails calls immediately for a cooldown, then lets a trial call through (half-open) to see if it recovered. Example: our order API called a tax service that started timing out; without a breaker, every checkout request held a thread for 30 seconds and the whole site slowed. With a breaker per dependency plus a 2-second timeout, we failed fast and fell back to a cached tax estimate, and the API stayed responsive. Trade-offs: thresholds need tuning to avoid flapping, each instance has its own breaker state, and you must define what the fallback is; it also doesn't replace timeouts or bulkheads." (Structure: problem, idea, example, trade-off.)
**Follow-up traps**
- *"How do you know it's working?"* Metrics on state transitions, rejected calls, alerts.
- *"Library?"* Polly v8 / `Microsoft.Extensions.Resilience` in .NET.

### 22.6 [Hard] Pick any pattern you used and explain when you'd *not* use it.
**What the interviewer is really testing**: Depth, honesty and judgment.
**Typical mid-level answer**: Only praises the pattern.
**Strong senior answer**: Use the structure: (1) what force led me to it, (2) the version I implemented (smallest form), (3) a measured outcome, (4) the conditions where it would have been wrong. Example: Outbox pattern for publishing order events: the force was losing events in the dual write between SQL and Service Bus; implemented a table in the same transaction plus a relay worker with row claiming, idempotent consumers, and lag alerts; result was no lost events after incidents. I wouldn't use it where the system tolerates occasional loss (telemetry), where the broker and DB can share a transaction, or for low-volume internal tools where a reconciliation job suffices. Costs: extra table, relay operations, duplicate delivery. Showing the "no" cases signals seniority.
**Follow-up traps**
- *"Outbox via CDC instead of polling?"* Possible (e.g., Debezium/Change Tracking); trades polling latency for infrastructure.
- *"Ordering in outbox?"* Order per aggregate with sequence/ordered relay; global order is costly.

**Red flags in answers**
- Reciting a pattern definition with no real example.
- Cannot state when NOT to use the pattern they just described.
- Scenario answers that jump to a solution with no clarification of context.

**Cheat sheet**
- Scenario shape: context -> forces -> options -> choice -> risks -> revisit.
- 60-second formula: Problem, Idea, Example, Trade-off.
- Abstraction checklist: second implementation, volatile boundary, ACL, dependency direction.
- Settle debates with a time-boxed spike, ADR, revisit criteria.
- Payments: idempotency + timeout + retry + breaker + reconcile + outbox.
