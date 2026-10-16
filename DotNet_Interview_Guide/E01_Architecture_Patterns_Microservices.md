# E01 - Architecture, Patterns & Microservices (.NET 8)

**How to use:** Read the *Study notes* of a topic first, then walk the difficulty ladder top to bottom; stop at the rung where you can no longer answer without peeking.
Re-read *Interview traps* and *Cheat sheet* the night before an interview. Code targets C# 12 / .NET 8 unless stated.

## Table of Contents
1. [Layered vs Clean vs Onion vs Hexagonal](#topic-1-layered-vs-clean-vs-onion-vs-hexagonal)
2. [Domain-Driven Design](#topic-2-domain-driven-design)
3. [CQRS and Vertical Slice](#topic-3-cqrs-and-vertical-slice-architecture)
4. [Event Sourcing and Projections](#topic-4-event-sourcing-and-projections)
5. [Monolith vs Modular Monolith vs Microservices](#topic-5-monolith-vs-modular-monolith-vs-microservices)
6. [Service Communication and Message Brokers](#topic-6-service-communication-and-message-brokers)
7. [Outbox, Inbox and Idempotency](#topic-7-outbox-inbox-and-idempotency)
8. [Saga, Distributed Transactions, Eventual Consistency](#topic-8-saga-distributed-transactions-eventual-consistency)
9. [CAP Theorem and Consistency Models](#topic-9-cap-theorem-and-consistency-models)
10. [API Gateway, BFF, YARP, Discovery, Resilience](#topic-10-api-gateway-bff-yarp-discovery-resilience)
11. [Data Ownership and Strangler Fig Migration](#topic-11-data-ownership-and-strangler-fig-migration)
12. [GoF Patterns in .NET](#topic-12-gof-patterns-in-net)
13. [Enterprise Patterns: Specification, UoW, Repository, Result, Options, Null Object, DIP](#topic-13-enterprise-patterns)
14. [Cross-Cutting Concerns and Domain vs Integration Events](#topic-14-cross-cutting-concerns-and-domain-vs-integration-events)
15. [Multi-Tenancy and Scalability Patterns](#topic-15-multi-tenancy-and-scalability-patterns)
16. [Versioning, Observability, Feature Flags, Deployment, ADRs](#topic-16-versioning-observability-feature-flags-deployment-adrs)

---

## Topic 1: Layered vs Clean vs Onion vs Hexagonal

**Study notes**

All four aim at separating business rules from infrastructure. They differ in dependency direction.

```
Classic N-layer (dependencies point DOWN)      Clean / Onion / Hexagonal (dependencies point IN)

  Presentation                                     +-----------------------------+
       |                                           |  Frameworks: Web, EF, MQ    |
  Business (BLL)                                   |  +-----------------------+  |
       |                                           |  | Adapters/Controllers  |  |
  Data Access (DAL) --> DB                         |  |  +-----------------+  |  |
                                                   |  |  | Application     |  |  |
 BLL depends on DAL => domain knows EF             |  |  |  +-----------+  |  |  |
                                                   |  |  |  |  Domain   |  |  |  |
                                                   |  |  |  +-----------+  |  |  |
                                                   |  |  +-----------------+  |  |
                                                   |  +-----------------------+  |
                                                   +-----------------------------+
                                                   Source-code deps always point to the center.
```

- **Layered**: simple, each layer uses the one below. Risk: domain coupled to persistence.
- **Onion** (Palermo): domain model at the center, application services around it, infrastructure outside.
- **Hexagonal / Ports & Adapters** (Cockburn): the app exposes *ports* (interfaces); *adapters* (HTTP, DB, queue) plug into them. Driving (primary) vs driven (secondary) ports.
- **Clean** (Uncle Bob): Entities -> Use Cases -> Interface Adapters -> Frameworks. Essentially the same idea with named rings.
- Typical .NET solution:

```
src/
  Shop.Domain/           (no project refs)
  Shop.Application/      (-> Domain)  use cases, interfaces (ports), DTOs
  Shop.Infrastructure/   (-> Application, Domain) EF Core, email, brokers
  Shop.Api/              (-> Application, Infrastructure for DI composition root)
tests/
  Shop.Domain.Tests, Shop.Application.Tests, Shop.Api.IntegrationTests
```

### 1.1 [Easy] What is the main difference between N-layer and Clean Architecture?
**Answer**
In N-layer, dependencies flow top to bottom so the business layer depends on data access. In Clean Architecture dependencies point inward: infrastructure depends on the domain/application, never the reverse. This is achieved with Dependency Inversion: the application defines interfaces, infrastructure implements them.

**Explanation**
The practical payoff is that the domain can be unit tested without a database, and you can swap EF Core, a message broker, or an external API without touching business rules. The cost is more projects, more interfaces and mapping code, which is overkill for simple CRUD.

**Code**
```csharp
// Application project: the port
public interface IOrderRepository
{
    Task<Order?> GetAsync(Guid id, CancellationToken ct);
    Task AddAsync(Order order, CancellationToken ct);
}

// Infrastructure project: the adapter
public sealed class EfOrderRepository(ShopDbContext db) : IOrderRepository
{
    public Task<Order?> GetAsync(Guid id, CancellationToken ct) =>
        db.Orders.Include(o => o.Lines).FirstOrDefaultAsync(o => o.Id == id, ct);

    public async Task AddAsync(Order order, CancellationToken ct) =>
        await db.Orders.AddAsync(order, ct);
}
```

### 1.2 [Follow-up] How are Onion, Hexagonal and Clean related? Which would you pick?
**Answer**
They are variations of one idea: a framework-independent core with dependencies pointing inward. Hexagonal emphasises ports/adapters at the boundary, Onion emphasises concentric layers, Clean names the rings (entities, use cases, adapters). I would pick whichever vocabulary the team already shares and enforce the dependency rule, since the rule matters more than the name.

**Explanation**
Hexagonal is useful when explaining multiple entry points (HTTP, CLI, queue consumer all drive the same use case) and multiple outputs. Clean gives a concrete checklist. In .NET all three typically map onto the Domain / Application / Infrastructure / Api project split.

**Code**
```csharp
// Driving adapter (HTTP) and driving adapter (queue consumer) call the same use case
public interface IPlaceOrder { Task<Guid> HandleAsync(PlaceOrderCommand cmd, CancellationToken ct); }

app.MapPost("/orders", async (PlaceOrderCommand cmd, IPlaceOrder uc, CancellationToken ct) =>
    Results.Created("/orders", await uc.HandleAsync(cmd, ct)));

public sealed class PlaceOrderConsumer(IPlaceOrder uc) // e.g. invoked by a broker adapter
{
    public Task OnMessage(PlaceOrderCommand cmd, CancellationToken ct) => uc.HandleAsync(cmd, ct);
}
public sealed record PlaceOrderCommand(Guid CustomerId, List<Guid> ProductIds);
```

### 1.3 [Medium] How do you enforce the dependency rule in a .NET solution?
**Answer**
Use project references so Domain references nothing and Application references only Domain; the composition root (Api) wires implementations. Add architecture tests (NetArchTest or ArchUnitNET) in CI to fail the build on violations.

**Explanation**
Project references are the first line of defence since the compiler rejects cycles. But namespaces inside one project can still be violated, which is why architecture tests exist. Also keep EF attributes/types out of domain (use Fluent API in Infrastructure with `IEntityTypeConfiguration<T>`).

**Code**
```csharp
// NetArchTest.Rules
using NetArchTest.Rules;
using Xunit;

public class ArchitectureTests
{
    [Fact]
    public void Domain_should_not_depend_on_infrastructure()
    {
        var result = Types.InAssembly(typeof(Order).Assembly)
            .ShouldNot().HaveDependencyOnAny("Shop.Infrastructure", "Microsoft.EntityFrameworkCore")
            .GetResult();

        Assert.True(result.IsSuccessful);
    }
}

// Infrastructure/Configurations
public sealed class OrderConfig : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> b) { b.HasKey(o => o.Id); b.ToTable("Orders"); }
}
```

### 1.4 [Hard] Isn't Clean Architecture over-engineering? Where does it break down, and what about the "Application depends on EF" debate?
**Answer**
It is over-engineering for CRUD-heavy apps with little domain logic; the layers become pass-through. A pragmatic variant lets Application use `DbContext` directly (DbSet is already a repository and DbContext a unit of work) while keeping the domain pure. Pick the strictness by domain complexity.

**Explanation**
Wrapping EF Core in a generic repository hides IQueryable features, leaks anyway, and duplicates what EF does. Many teams use repositories only for aggregate roots (write side) and query EF directly in read handlers (CQRS). Mapping between layers (entity <-> DTO) is a cost: avoid mapping the same shape three times.

**Code**
```csharp
// Pragmatic: handler uses DbContext through an interface owned by Application
public interface IAppDbContext
{
    DbSet<Order> Orders { get; }
    Task<int> SaveChangesAsync(CancellationToken ct);
}

public sealed class GetOrderTotalHandler(IAppDbContext db)
{
    public Task<decimal> HandleAsync(Guid id, CancellationToken ct) =>
        db.Orders.Where(o => o.Id == id)
                 .Select(o => o.Lines.Sum(l => l.Quantity * l.UnitPrice))
                 .SingleAsync(ct);
}
```

**Gotchas / follow-ups**
- Exposing `DbSet<T>` from Application makes Application depend on `Microsoft.EntityFrameworkCore` (the interface lives in Application, so the package reference does too). Acknowledge the trade-off explicitly.
- Follow-up: "How do you test handlers?" Use SQLite in-memory or Testcontainers rather than the EF InMemory provider (it does not enforce relational behaviour).

### 1.5 [Tricky] Where do validation rules, DTOs and "business logic" live, and what is wrong with putting interfaces in the Infrastructure project?
**Answer**
Invariants live in the domain (entities/value objects), request-shape validation in Application (e.g. FluentValidation) or API, and DTOs at the boundary. Interfaces in Infrastructure invert nothing: Application would have to reference Infrastructure, breaking the dependency rule.

**Explanation**
Dependency *inversion* means the high-level module owns the abstraction. If `IOrderRepository` sits beside its EF implementation, the consumer references the concrete assembly (even transitively). Another trap: "anemic services" in Application that just shuffle data while entities are bags of setters.

**Code**
```csharp
// Domain: invariant enforced inside the entity
public sealed class Order
{
    private readonly List<OrderLine> _lines = [];
    public Guid Id { get; private set; } = Guid.NewGuid();
    public IReadOnlyList<OrderLine> Lines => _lines;

    public void AddLine(Guid productId, int qty, decimal price)
    {
        if (qty <= 0) throw new ArgumentOutOfRangeException(nameof(qty));
        _lines.Add(new OrderLine(productId, qty, price));
    }
}
public sealed record OrderLine(Guid ProductId, int Quantity, decimal UnitPrice);
```

**Gotchas / follow-ups**
- Shared "Common"/"Core" projects with everything in them become a dumping ground; prefer per-layer, per-feature organisation.
- Domain should not reference `Microsoft.Extensions.DependencyInjection`; register services in each layer's `AddXxx` extension method instead.

**Interview traps**
- Saying "Clean Architecture means 4 projects" - it is about dependency direction, not project count.
- Putting the repository interface in Infrastructure.
- Claiming the EF InMemory provider is a faithful test database.
- Returning EF entities directly from controllers.

**Cheat sheet**
- Dependencies point inward; domain has zero references.
- Application owns ports; Infrastructure implements adapters.
- Composition root (Api) is the only place that knows everything.
- Hexagonal = ports/adapters; Onion = rings; Clean = named rings; same rule.
- Enforce with project refs + NetArchTest.

---

## Topic 2: Domain-Driven Design

**Study notes**

```
Bounded Context: Sales                  Bounded Context: Shipping
+--------------------------+            +---------------------------+
| Customer (name, credit)  |  <-- ACL -> | Recipient (address)       |
| Order (aggregate root)   |            | Shipment (aggregate root) |
|   OrderLine (entity)     |            |   Parcel                  |
|   Money (value object)   |            +---------------------------+
+--------------------------+     same word, different model per context
```

- **Entity**: identity-based equality, lifecycle. **Value object**: equality by value, immutable (`Money`, `Address`) - C# `record`/`record struct` fits.
- **Aggregate**: cluster of objects with one **root**; transaction/consistency boundary; external references only to the root, by ID. Keep aggregates small.
- **Domain event**: something that happened (`OrderPlaced`), past tense, raised by the aggregate.
- **Repository**: collection-like persistence abstraction per aggregate root.
- **Bounded context**: explicit boundary where a model and its ubiquitous language are valid; integrate through contracts / anti-corruption layers.
- **Anemic model**: entities with only getters/setters, logic in services. **Rich model**: behaviour inside entities.
- Strategic DDD (contexts, context maps) matters more than tactical patterns.

### 2.1 [Easy] Entity vs value object?
**Answer**
An entity has a stable identity and its attributes may change; two entities with the same data but different IDs are different. A value object has no identity, is immutable, and is equal when all its values are equal. `Money`, `Address`, `DateRange` are classic value objects.

**Explanation**
C# `record` gives value equality and `with` expressions for free. In EF Core, map value objects as owned types (`OwnsOne`) or, in EF Core 8, complex types (`ComplexProperty`). Replace a value object instead of mutating it.

**Code**
```csharp
public readonly record struct Money(decimal Amount, string Currency)
{
    public static Money operator +(Money a, Money b) =>
        a.Currency == b.Currency
            ? new(a.Amount + b.Amount, a.Currency)
            : throw new InvalidOperationException("Currency mismatch");
}

// EF Core 8 complex type
modelBuilder.Entity<Order>().ComplexProperty(o => o.Total);
// or owned type:
// modelBuilder.Entity<Order>().OwnsOne(o => o.Total);
```

### 2.2 [Follow-up] What is an aggregate and how do you size it?
**Answer**
An aggregate is a consistency boundary with a root that guards invariants; it is loaded and saved as a unit and one transaction modifies one aggregate. Size it by true invariants that must be immediately consistent: smaller is better, and other aggregates are referenced by ID.

**Explanation**
Big aggregates cause lock contention, concurrency conflicts and heavy loads. Cross-aggregate rules are enforced with eventual consistency through domain events. Use optimistic concurrency (rowversion) on the root.

**Code**
```csharp
public sealed class Order
{
    private readonly List<OrderLine> _lines = [];
    public Guid Id { get; private set; }
    public Guid CustomerId { get; private set; }          // reference by ID, not Customer object
    public OrderStatus Status { get; private set; }
    public byte[] RowVersion { get; private set; } = [];   // concurrency token
    public IReadOnlyCollection<OrderLine> Lines => _lines;

    private Order() { }  // EF
    public Order(Guid customerId) { Id = Guid.NewGuid(); CustomerId = customerId; }

    public void AddLine(Guid productId, int qty, Money price)
    {
        if (Status != OrderStatus.Draft) throw new InvalidOperationException("Order is locked");
        _lines.Add(new OrderLine(productId, qty, price));
    }
}
public enum OrderStatus { Draft, Placed, Shipped }
public sealed record OrderLine(Guid ProductId, int Quantity, Money Price);

// config
builder.Property(o => o.RowVersion).IsRowVersion();
```

### 2.3 [Medium] How are domain events raised and dispatched with EF Core?
**Answer**
The aggregate collects events in an internal list; a `SaveChanges` override (or interceptor) gathers them from tracked entities, saves, then publishes them. Dispatch before commit gives same-transaction handlers; after commit is safer for side effects.

**Explanation**
Dispatching in-process before save lets handlers modify other state in one transaction; dispatching after save means handlers can fail without rolling back the aggregate, hence the outbox for reliable external publishing (Topic 7). Clear the events after dispatch to avoid duplicates.

**Code**
```csharp
public interface IDomainEvent { }
public abstract class AggregateRoot
{
    private readonly List<IDomainEvent> _events = [];
    public IReadOnlyCollection<IDomainEvent> DomainEvents => _events;
    protected void Raise(IDomainEvent e) => _events.Add(e);
    public void ClearEvents() => _events.Clear();
}
public sealed record OrderPlaced(Guid OrderId) : IDomainEvent;

public interface IDomainEventDispatcher { Task DispatchAsync(IEnumerable<IDomainEvent> events, CancellationToken ct); }

public sealed class ShopDbContext(DbContextOptions<ShopDbContext> o, IDomainEventDispatcher dispatcher) : DbContext(o)
{
    public override async Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        var aggregates = ChangeTracker.Entries<AggregateRoot>()
            .Select(e => e.Entity).Where(a => a.DomainEvents.Count > 0).ToList();
        var events = aggregates.SelectMany(a => a.DomainEvents).ToList();
        aggregates.ForEach(a => a.ClearEvents());

        var result = await base.SaveChangesAsync(ct);
        await dispatcher.DispatchAsync(events, ct); // after commit
        return result;
    }
}
```

### 2.4 [Hard] Anemic vs rich domain model - when is anemic acceptable, and how do you persist a rich model with private setters in EF Core?
**Answer**
Anemic is acceptable for simple CRUD or data-driven reporting where there are no real invariants; rich is for complex rules. EF Core can materialise rich models via private parameterless constructors, private setters and backing fields, so persistence does not force an anemic design.

**Explanation**
Fowler calls anemic models an anti-pattern because it gives the cost of an object model without the benefits - but it's a tradeoff, not a sin. EF Core uses backing-field conventions (`_lines` for `Lines`) and can bind to private constructors. Use `Navigation(...).UsePropertyAccessMode` if needed.

**Code**
```csharp
public sealed class Account
{
    public Guid Id { get; private set; }
    public decimal Balance { get; private set; }
    private Account() { }
    public Account(Guid id) => Id = id;

    public void Withdraw(decimal amount)
    {
        if (amount <= 0) throw new ArgumentOutOfRangeException(nameof(amount));
        if (amount > Balance) throw new InvalidOperationException("Insufficient funds");
        Balance -= amount;
    }
}

// Backing field for a collection exposed read-only
modelBuilder.Entity<Order>()
    .Navigation(o => o.Lines)
    .UsePropertyAccessMode(PropertyAccessMode.Field);
```

**Gotchas / follow-ups**
- Anemic smell: a `OrderService` with 800 lines of if-statements mutating public setters.
- Concurrency: two withdrawals loading the same `Balance` - need a row version or `ExecuteUpdate` with a guard.
- Don't let UI/DTO shape drive the domain model.

### 2.5 [Tricky] How do you handle a rule spanning two aggregates, and how do bounded contexts integrate without sharing a model?
**Answer**
Cross-aggregate rules use eventual consistency via domain/integration events, or a domain service if the data is in the same context and consistency must be immediate. Across contexts, use published contracts and an anti-corruption layer (ACL) that translates the foreign model to yours.

**Explanation**
Example: "customer's credit limit is checked on order" - either read a replicated credit snapshot (stale risk) or reserve credit through a saga. Avoid Customer-owns-Orders in a single giant aggregate. An ACL is a translator class so the legacy/external vocabulary doesn't leak.

**Code**
```csharp
// Anti-corruption layer in Sales context for a legacy CRM
public interface ICustomerDirectory { Task<CustomerInfo?> FindAsync(Guid id, CancellationToken ct); }
public sealed record CustomerInfo(Guid Id, bool IsActive, Money CreditLimit);

public sealed class LegacyCrmCustomerDirectory(HttpClient http) : ICustomerDirectory
{
    private sealed record CrmCustomerDto(string CUST_ID, string STATUS_CD, decimal CRD_LMT);

    public async Task<CustomerInfo?> FindAsync(Guid id, CancellationToken ct)
    {
        var dto = await http.GetFromJsonAsync<CrmCustomerDto>($"customers/{id}", ct);
        return dto is null ? null
            : new CustomerInfo(Guid.Parse(dto.CUST_ID), dto.STATUS_CD == "A", new Money(dto.CRD_LMT, "USD"));
    }
}
```

**Gotchas / follow-ups**
- "One transaction = one aggregate" is a guideline; breaking it knowingly in a modular monolith is sometimes pragmatic.
- Shared kernel vs customer/supplier vs conformist vs ACL - know the context-map names.

**Interview traps**
- Treating DDD as only entities + repositories and ignoring bounded contexts.
- Making every table an aggregate; huge aggregates with navigation to everything.
- Repository per table instead of per aggregate root.
- Saying value objects must be structs (records/classes are fine).

**Cheat sheet**
- Entity = identity; value object = immutable, value equality.
- Aggregate = consistency boundary; reference others by ID.
- Domain events: past tense, raised in aggregate, dispatched on save.
- Bounded context = own model + language; integrate via ACL/contracts.
- Rich model if invariants matter; anemic OK for CRUD.

---

## Topic 3: CQRS and Vertical Slice Architecture

**Study notes**

```
            Commands (write)                         Queries (read)
Client --> [Command Handler] --> Domain --> Write DB     Client --> [Query Handler] --> Read model/DB
                              \-> events --> projection --------------------------^

Levels: (1) same DB, separate models  (2) separate read store, synced by events  (3) + event sourcing
Vertical slice: organise by feature, not by technical layer

Features/Orders/PlaceOrder/  Command + Validator + Handler + Endpoint (all in one folder)
```

- **CQRS**: separate the model that changes state (commands) from the model that reads it (queries). Not the same as event sourcing, and not necessarily two databases.
- **MediatR** (third-party, commercial licence for newer versions) is a mediator in-process: `IRequest<T>`, `IRequestHandler<,>`, `IPipelineBehavior<,>`, notifications. CQRS does not require it - plain handler classes work.
- **Vertical Slice** (Jimmy Bogard): each feature owns everything it needs; slices may use different data-access styles.

### 3.1 [Easy] What is CQRS and why use it?
**Answer**
CQRS splits operations that change state (commands) from those that return data (queries), each with its own model. It lets reads be optimised (projections, denormalised views) independently from the write model that enforces invariants. It adds complexity, so use it where read and write needs diverge.

**Explanation**
Commands express intent (`PlaceOrder`), return little (ID or nothing), and may fail validation. Queries never change state. With one database it's simply separate handlers: write side uses aggregates via EF, read side uses `AsNoTracking` projections or Dapper.

**Code**
```csharp
public sealed record PlaceOrder(Guid CustomerId);
public sealed record OrderSummary(Guid Id, int LineCount, decimal Total);

public sealed class PlaceOrderHandler(ShopDbContext db)
{
    public async Task<Guid> HandleAsync(PlaceOrder cmd, CancellationToken ct)
    {
        var order = new Order(cmd.CustomerId);
        db.Orders.Add(order);
        await db.SaveChangesAsync(ct);
        return order.Id;
    }
}

public sealed class GetOrderSummaryHandler(ShopDbContext db)
{
    public Task<OrderSummary?> HandleAsync(Guid id, CancellationToken ct) =>
        db.Orders.AsNoTracking().Where(o => o.Id == id)
          .Select(o => new OrderSummary(o.Id, o.Lines.Count, o.Lines.Sum(l => l.Price.Amount * l.Quantity)))
          .FirstOrDefaultAsync(ct);
}
```

### 3.2 [Follow-up] How do you implement CQRS with MediatR, and without it?
**Answer**
With MediatR: define `IRequest<T>` commands/queries and `IRequestHandler`, register via `AddMediatR`, and send through `IMediator`. Without it, register handler classes in DI and inject them directly or use a thin `IDispatcher`; minimal APIs make this easy.

**Explanation**
MediatR's value is pipeline behaviors (cross-cutting) and decoupling endpoints from handlers; its costs are indirection, harder navigation, and licensing changes in recent versions. Direct injection of a handler is explicit and trivially debuggable.

**Code**
```csharp
// MediatR (packages: MediatR)
public sealed record GetOrder(Guid Id) : IRequest<OrderSummary?>;

public sealed class GetOrderHandler(ShopDbContext db) : IRequestHandler<GetOrder, OrderSummary?>
{
    public Task<OrderSummary?> Handle(GetOrder q, CancellationToken ct) =>
        db.Orders.AsNoTracking().Where(o => o.Id == q.Id)
          .Select(o => new OrderSummary(o.Id, o.Lines.Count, 0m)).FirstOrDefaultAsync(ct);
}
// builder.Services.AddMediatR(cfg => cfg.RegisterServicesFromAssemblyContaining<GetOrderHandler>());
// app.MapGet("/orders/{id}", (Guid id, IMediator m, CancellationToken ct) => m.Send(new GetOrder(id), ct));

// Without MediatR
// builder.Services.AddScoped<GetOrderSummaryHandler>();
// app.MapGet("/v2/orders/{id}", (Guid id, GetOrderSummaryHandler h, CancellationToken ct) => h.HandleAsync(id, ct));
```

### 3.3 [Medium] How does Vertical Slice differ from layered, and how do you structure it in .NET?
**Answer**
Layered groups code by technical role (Controllers, Services, Repositories) so one feature touches many folders; vertical slice groups by feature so a change touches one folder. Each slice picks its own implementation style, sharing only a thin kernel (domain, infrastructure).

**Explanation**
Minimizes coupling between features and maximizes coupling inside a feature ("minimize coupling between slices, maximize within"). Good with minimal APIs. Risk: duplicated logic - extract only when a real shared concept appears (rule of three), typically into the domain.

**Code**
```csharp
// Features/Orders/CancelOrder.cs - the whole slice in one file
public static class CancelOrder
{
    public sealed record Command(Guid OrderId);

    public static void MapEndpoint(IEndpointRouteBuilder app) =>
        app.MapPost("/orders/{id:guid}/cancel", Handle);

    private static async Task<IResult> Handle(Guid id, ShopDbContext db, CancellationToken ct)
    {
        var order = await db.Orders.FindAsync([id], ct);
        if (order is null) return Results.NotFound();
        order.Cancel();                       // domain behaviour
        await db.SaveChangesAsync(ct);
        return Results.NoContent();
    }
}
```

### 3.4 [Hard] Describe CQRS with separate read store and the consistency problems it introduces.
**Answer**
Writes go to the primary DB, and events update a denormalised read store asynchronously, so reads are eventually consistent. Problems: stale reads after write (read-your-writes), projection failures/duplicates, rebuild strategy, and operational complexity.

**Explanation**
Mitigations: return the written data/version from the command, have the UI poll or use optimistic update, make projections idempotent (store last processed version/event id), and track projection lag. Use the outbox so events are not lost.

**Code**
```csharp
// Idempotent projection updater using a version check
public sealed class OrderReadModel { public Guid Id { get; set; } public int Total { get; set; } public long Version { get; set; } }

public sealed class OrderProjection(ReadDbContext read)
{
    public async Task ApplyAsync(Guid orderId, long eventVersion, int total, CancellationToken ct)
    {
        var m = await read.OrderReadModels.FindAsync([orderId], ct);
        if (m is null) read.OrderReadModels.Add(m = new OrderReadModel { Id = orderId });
        if (eventVersion <= m.Version) return;   // duplicate / out-of-order, ignore
        m.Total = total; m.Version = eventVersion;
        await read.SaveChangesAsync(ct);
    }
}
```

**Gotchas / follow-ups**
- Out-of-order delivery breaks naive "last write wins" - version checks needed.
- "Do all queries need to go through the read store?" No - use the write DB for cases needing strong consistency.

### 3.5 [Tricky] Can a command return data? Is CQRS the same as event sourcing? When is CQRS a bad idea?
**Answer**
Strict CQS says commands return nothing, but pragmatic CQRS lets commands return an ID or result. CQRS is independent from event sourcing - ES usually pairs with CQRS but not vice versa. It's a bad idea for simple CRUD, small teams, or when strong consistency everywhere is needed.

**Explanation**
Returning a created ID is the usual accepted exception (client-generated GUIDs avoid it). Applying CQRS system-wide rather than per bounded context is a common error: apply it only where the model diverges.

**Code**
```csharp
// Client-generated id lets the command return nothing while staying idempotent-friendly
public sealed record CreateProduct(Guid Id, string Name);

public sealed class CreateProductHandler(ShopDbContext db)
{
    public async Task HandleAsync(CreateProduct cmd, CancellationToken ct)
    {
        if (await db.Products.AnyAsync(p => p.Id == cmd.Id, ct)) return; // idempotent retry
        db.Products.Add(new Product { Id = cmd.Id, Name = cmd.Name });
        await db.SaveChangesAsync(ct);
    }
}
```

**Gotchas / follow-ups**
- MediatR notifications are in-process and not durable: not a message broker.
- Handler-calls-handler chains recreate the service-layer tangle.

**Interview traps**
- "CQRS requires two databases / event sourcing / MediatR."
- Putting business logic in MediatR behaviors.
- Forgetting read-your-writes on a read replica/read store.
- Query handlers that track entities (use `AsNoTracking`).

**Cheat sheet**
- CQRS = separate write model and read model; levels of separation.
- MediatR optional; its value = pipeline behaviors.
- Vertical slice = organise by feature.
- Separate read store => eventual consistency, idempotent projections.
- Don't apply CQRS to simple CRUD.

---

## Topic 4: Event Sourcing and Projections

**Study notes**

```
Command --> load stream (events) --> fold to state --> decide --> append new event(s)
                                                                  |
  Event store (append-only)                                       v
  Stream "order-42": [OrderCreated v1][LineAdded v2][OrderPlaced v3] --> projections --> read models
```

- State is not stored; the **sequence of events** is. Current state = fold of events (`Apply`).
- **Optimistic concurrency** on append: expected stream version.
- **Snapshots** speed up loading long streams. **Projections** build read models (rebuildable by replaying).
- Benefits: full audit, temporal queries, new projections retroactively. Costs: event schema evolution (upcasting), eventual consistency of read models, learning curve, GDPR deletion difficulty (crypto-shredding).
- .NET options: Marten (PostgreSQL), EventStoreDB, or hand-rolled on SQL Server.

### 4.1 [Easy] What is event sourcing?
**Answer**
Instead of storing current state, you persist every state-changing event in an append-only log and rebuild state by replaying. The log is the source of truth, giving an audit trail and the ability to derive new views later.

**Explanation**
An event is an immutable fact in the past tense. Aggregates expose behaviour methods that validate and emit events, and `Apply` methods mutate state from events (without validation, since events already happened).

**Code**
```csharp
public abstract record DomainEvent;
public sealed record AccountOpened(Guid AccountId) : DomainEvent;
public sealed record MoneyDeposited(decimal Amount) : DomainEvent;
public sealed record MoneyWithdrawn(decimal Amount) : DomainEvent;

public sealed class Account
{
    public Guid Id { get; private set; }
    public decimal Balance { get; private set; }
    public int Version { get; private set; }
    private readonly List<DomainEvent> _uncommitted = [];
    public IReadOnlyList<DomainEvent> Uncommitted => _uncommitted;

    public static Account Rehydrate(IEnumerable<DomainEvent> history)
    {
        var a = new Account();
        foreach (var e in history) a.Apply(e);
        return a;
    }

    public void Withdraw(decimal amount)
    {
        if (amount > Balance) throw new InvalidOperationException("Insufficient funds");
        Raise(new MoneyWithdrawn(amount));
    }

    private void Raise(DomainEvent e) { Apply(e); _uncommitted.Add(e); }

    private void Apply(DomainEvent e)
    {
        switch (e)
        {
            case AccountOpened o: Id = o.AccountId; break;
            case MoneyDeposited d: Balance += d.Amount; break;
            case MoneyWithdrawn w: Balance -= w.Amount; break;
        }
        Version++;
    }
}
```

### 4.2 [Follow-up] How do you append events safely and what do snapshots do?
**Answer**
Append with an expected version; if another writer already appended, the append fails and the command retries. Snapshots store state at version N so rehydration loads the snapshot plus later events instead of the whole stream.

**Explanation**
In SQL Server, a unique index on (StreamId, Version) turns concurrent appends into a constraint violation - a simple, correct optimistic-concurrency mechanism.

**Code**
```sql
CREATE TABLE Events (
    StreamId   UNIQUEIDENTIFIER NOT NULL,
    Version    INT              NOT NULL,
    Type       NVARCHAR(200)    NOT NULL,
    Data       NVARCHAR(MAX)    NOT NULL,
    OccurredAt DATETIME2        NOT NULL DEFAULT SYSUTCDATETIME(),
    CONSTRAINT PK_Events PRIMARY KEY (StreamId, Version)   -- duplicate version => concurrency conflict
);
```
```csharp
public async Task AppendAsync(Guid streamId, int expectedVersion, IReadOnlyList<DomainEvent> events, CancellationToken ct)
{
    var v = expectedVersion;
    foreach (var e in events)
        db.Set<EventRow>().Add(new EventRow(streamId, ++v, e.GetType().Name, JsonSerializer.Serialize(e, e.GetType())));
    try { await db.SaveChangesAsync(ct); }
    catch (DbUpdateException) { throw new InvalidOperationException("Concurrency conflict, retry"); }
}
public sealed record EventRow(Guid StreamId, int Version, string Type, string Data);
```

### 4.3 [Medium] What are projections and how do you rebuild them?
**Answer**
A projection consumes the event stream and maintains a read-optimised model (table, document, search index). To rebuild, truncate the read model, reset the checkpoint, and replay all events.

**Explanation**
Track a checkpoint (last processed global position) with the read-model update in the same transaction so replays are idempotent. Inline projections are consistent with the write, async projections are eventually consistent and scale separately.

**Code**
```csharp
public sealed class BalanceProjector(ReadDbContext db)
{
    public async Task ProjectAsync(IReadOnlyList<(long Position, Guid AccountId, DomainEvent Event)> batch, CancellationToken ct)
    {
        await using var tx = await db.Database.BeginTransactionAsync(ct);
        var cp = await db.Checkpoints.SingleAsync(c => c.Name == "balances", ct);
        foreach (var (pos, id, e) in batch.Where(b => b.Position > cp.Position))
        {
            var row = await db.Balances.FindAsync([id], ct) ?? db.Balances.Add(new BalanceRow { AccountId = id }).Entity;
            row.Amount += e switch { MoneyDeposited d => d.Amount, MoneyWithdrawn w => -w.Amount, _ => 0m };
            cp.Position = pos;
        }
        await db.SaveChangesAsync(ct);
        await tx.CommitAsync(ct);
    }
}
public sealed class BalanceRow { public Guid AccountId { get; set; } public decimal Amount { get; set; } }
public sealed class Checkpoint { public string Name { get; set; } = ""; public long Position { get; set; } }
```

### 4.4 [Hard] How do you handle event schema evolution?
**Answer**
Events are immutable, so never edit stored ones; version event types and use upcasters that convert old versions to the current shape on read. Prefer additive changes (new optional fields), and for breaking changes publish a new event type.

**Explanation**
Options: weak schema/tolerant reader (JSON with defaults), upcasting pipeline, or copy-and-transform migration of the stream to a new store (last resort). Keep event names stable independent of CLR type names via an explicit type-name map; renaming a class must not break deserialisation.

**Code**
```csharp
public sealed record CustomerRegisteredV1(Guid Id, string FullName);
public sealed record CustomerRegisteredV2(Guid Id, string FirstName, string LastName);

public static class Upcaster
{
    public static object Upcast(object e) => e switch
    {
        CustomerRegisteredV1 v1 => new CustomerRegisteredV2(v1.Id,
            v1.FullName.Split(' ', 2)[0],
            v1.FullName.Contains(' ') ? v1.FullName.Split(' ', 2)[1] : ""),
        _ => e
    };
}
```

**Gotchas / follow-ups**
- Storing CLR assembly-qualified type names in the store couples you to code layout.
- Personal data in immutable events: crypto-shredding (encrypt per-subject key, delete key).

### 4.5 [Tricky] When should you NOT use event sourcing, and what is the difference from an audit log or "event-driven"?
**Answer**
Avoid it for CRUD-style data, simple reporting systems, or teams unready for eventual consistency and versioning. Event sourcing makes the events the *source of truth*; an audit table is a side record, and event-driven architecture just means components communicate via events - none of which require event sourcing.

**Explanation**
Querying current state across many aggregates needs projections, which many teams underestimate. Also, deletions, corrections (compensating events), and long streams (snapshots) all need design. Integration events published to other services are not the same as the domain events in your store - don't expose internal events as a public contract.

**Code**
```csharp
// Correction via compensating event rather than mutation
public sealed record DepositReversed(Guid DepositId, string Reason) : DomainEvent;
```

**Gotchas / follow-ups**
- "Replay the whole store in production" can take hours: design projection rebuild into a new table and swap.
- Side effects (emails) must not fire during replay.

**Interview traps**
- Confusing event sourcing with event-driven messaging.
- Thinking you can "update" or "delete" an old event.
- Letting projection handlers send emails (replay duplicates).
- Ignoring versioning until after production.

**Cheat sheet**
- State = fold(events); log is source of truth.
- Append with expected version (unique (stream, version)).
- Projections: checkpoint + idempotent, rebuildable.
- Snapshots optimise reads; upcasters handle schema change.
- Not for plain CRUD.

---

## Topic 5: Monolith vs Modular Monolith vs Microservices

**Study notes**

```
Big-ball monolith        Modular monolith                    Microservices
+----------------+      +-----------------------------+      +-----+  +-----+  +-----+
| everything     |      | [Orders]  [Billing] [Stock] |      | Ord |  | Bil |  | Stk |
| calls          |      |  public API only between    |      | DB1 |  | DB2 |  | DB3 |
| everything     |      |  modules, own schema/tables |      +--+--+  +--+--+  +--+--+
| 1 DB           |      | 1 deployable, 1 DB (schemas)|         network calls / messages
+----------------+      +-----------------------------+
```

- **Monolith**: one deployable. Simple deploy, debug, transactions, refactor.
- **Modular monolith**: one deployable, strong module boundaries (internal types, public contracts, separate schemas, no cross-module table joins). Can later split modules into services.
- **Microservices**: independently deployable services owning their data; buy independent scaling/deploy/team autonomy, pay with distributed-system complexity (network failures, observability, data consistency, versioning, ops).
- Prerequisites: mature CI/CD, monitoring, DevOps culture, clear domain boundaries.

### 5.1 [Easy] What are the benefits and costs of microservices?
**Answer**
Benefits: independent deployment and scaling, team autonomy, technology and fault isolation. Costs: network latency and failure, distributed data consistency, operational overhead (deployment, monitoring, tracing), and harder testing and debugging.

**Explanation**
The gain is organisational as much as technical (Conway's law). A small team with one product usually pays the cost without receiving the benefit.

**Code**
```csharp
// What was an in-process call becomes a remote call that can fail: must add timeouts/retries
builder.Services.AddHttpClient<IInventoryClient, InventoryClient>(c =>
{
    c.BaseAddress = new Uri("https://inventory");
    c.Timeout = TimeSpan.FromSeconds(3);
});
public interface IInventoryClient { Task<int> GetStockAsync(Guid productId, CancellationToken ct); }
public sealed class InventoryClient(HttpClient http) : IInventoryClient
{
    public async Task<int> GetStockAsync(Guid id, CancellationToken ct) =>
        await http.GetFromJsonAsync<int>($"stock/{id}", ct);
}
```

### 5.2 [Follow-up] What is a modular monolith and how do you enforce module boundaries in .NET?
**Answer**
A single deployable split into cohesive modules that interact only through public contracts, each owning its data. Enforce with separate projects per module (internal implementation, small public contracts project), separate DB schemas, and architecture tests.

**Explanation**
Modules communicate via an interface/facade or in-process events - never by touching another module's tables. Use `internal` types, one `DbContext` per module mapped to its own schema, and no foreign keys across schemas. This makes later extraction cheap.

**Code**
```csharp
// Orders.Contracts (public)
public interface IOrdersModule { Task<OrderDto?> GetAsync(Guid id, CancellationToken ct); }
public sealed record OrderDto(Guid Id, decimal Total);

// Orders (internal implementation)
internal sealed class OrdersModule(OrdersDbContext db) : IOrdersModule
{
    public Task<OrderDto?> GetAsync(Guid id, CancellationToken ct) =>
        db.Orders.Where(o => o.Id == id).Select(o => new OrderDto(o.Id, o.Total)).FirstOrDefaultAsync(ct);
}
public static class OrdersModuleExtensions
{
    public static IServiceCollection AddOrdersModule(this IServiceCollection s, string cs) =>
        s.AddDbContext<OrdersDbContext>(o => o.UseSqlServer(cs, x => x.MigrationsHistoryTable("__EFMigrationsHistory", "orders")))
         .AddScoped<IOrdersModule, OrdersModule>();
}
// OrdersDbContext: modelBuilder.HasDefaultSchema("orders");
```

### 5.3 [Medium] When should you NOT use microservices?
**Answer**
Don't use them for small teams, early-stage products with unclear domain boundaries, systems needing strong cross-entity transactions, or without CI/CD and observability maturity. Start with a modular monolith and extract services when there is a concrete scaling or team-autonomy need.

**Explanation**
Wrong boundaries are far more expensive to fix across services than across modules. Signs you built a distributed monolith: services must deploy together, share a DB, chain synchronous calls, or one change touches many repos. Real extraction reasons: different scaling profile, different release cadence, regulatory isolation, a different technology need.

**Code**
```csharp
// Same host, module boundary preserved -> extraction later = swap the implementation of the interface
builder.Services.AddOrdersModule(cs);          // in-process today
// later: builder.Services.AddHttpClient<IOrdersModule, OrdersHttpClient>(...) // remote tomorrow
```

### 5.4 [Hard] How do you decide service boundaries?
**Answer**
Align them to bounded contexts and business capabilities, not to nouns or database tables. A good boundary has high cohesion, low coupling, owns its data, and can change independently of others; look at team structure and rate of change as well.

**Explanation**
Techniques: event storming to find contexts and events, analysing which data changes together, checking transactions that cross the boundary (a smell), and counting chatty calls between candidates. Avoid "entity services" (CustomerService, OrderService as CRUD wrappers) and one-service-per-function nano-services.

**Code**
```csharp
// Smell test in code: if placing an order requires synchronous calls to 4 services, the boundary is probably wrong
public async Task<Guid> PlaceAsync(PlaceOrder cmd, CancellationToken ct)
{
    var customer  = await customers.GetAsync(cmd.CustomerId, ct); // call 1
    var stock     = await inventory.ReserveAsync(cmd.Items, ct);   // call 2
    var price     = await pricing.QuoteAsync(cmd.Items, ct);       // call 3
    var risk      = await fraud.CheckAsync(cmd.CustomerId, ct);    // call 4: availability = product of all
    return Guid.NewGuid();
}
```

**Gotchas / follow-ups**
- Availability of a synchronous chain = product of availabilities (0.99^4 ~ 0.96).
- Shared libraries containing domain models re-couple services.

### 5.5 [Tricky] "Netflix uses microservices, so should we." Respond. Also: what is the distributed monolith and the "microservice premium"?
**Answer**
Netflix has hundreds of teams and scale problems that justify the cost; most companies do not. A distributed monolith has the coupling of a monolith with the failure modes of a distributed system - the worst of both worlds. The microservice premium is the fixed overhead you pay before benefits appear.

**Explanation**
Show judgment: ask what problem you're solving (deploy speed? scaling? team conflicts?) and whether a modular monolith, better CI, or scaling the monolith horizontally solves it more cheaply. Mention the real industry trend of consolidating back to monoliths in some cases.

**Code**
```csharp
// A tell-tale: lock-step releases enforced by a shared contract package and a shared DB
// <PackageReference Include="Shop.SharedModels" Version="*" />   // every service rebuilds on any model change
```

**Gotchas / follow-ups**
- Horizontal scaling of a stateless monolith behind a load balancer is often enough.
- Premature extraction: you also need to extract data, not only code.

**Interview traps**
- Presenting microservices as strictly better.
- Services sharing one database "temporarily".
- Splitting by technical layer or by entity.
- No mention of operational prerequisites.

**Cheat sheet**
- Modular monolith = default starting point.
- Microservice = independent deploy + own data.
- Boundary = bounded context/capability, not table.
- Distributed monolith = worst of both.
- Sync call chains multiply failure probability.

---

## Topic 6: Service Communication and Message Brokers

**Study notes**

```
Sync (request/response)               Async (messaging)
 A --HTTP/gRPC--> B  (A waits)         A --> [Broker: queue/topic] --> B  (temporal decoupling)

Queue (point-to-point, competing consumers)     Topic (pub/sub)
  P -> [ q ] -> C1                                P -> [topic] -> sub1 -> C1
              -> C2 (each msg to ONE)                         -> sub2 -> C2 (each sub gets a copy)
```

- **REST/JSON**: ubiquitous, cacheable, human-friendly. **gRPC**: HTTP/2, protobuf, strongly typed contracts, streaming, smaller/faster; poor browser support (gRPC-Web needed).
- **Messaging**: decoupling in time and availability, load levelling, retries; adds eventual consistency and broker ops.
- Message kinds: **command** (one receiver, intent), **event** (many receivers, fact), **query/reply**.
- Brokers: RabbitMQ, Azure Service Bus, Kafka (log, partitions, consumer groups, replay), SQS/SNS. Libraries: **MassTransit**, **NServiceBus** (commercial), Wolverine, Rebus.
- Delivery: usually **at-least-once** -> consumers must be idempotent.

### 6.1 [Easy] Sync vs async communication - when to choose which?
**Answer**
Use synchronous calls when the caller needs the answer now (queries, validations); use asynchronous messaging for workflows where the work can be done later, for fan-out, or to decouple availability. Async improves resilience and load levelling but adds eventual consistency.

**Explanation**
Prefer async for service-to-service commands/events within business processes, sync for user-facing reads. Avoid long synchronous call chains. For sync inside .NET use `HttpClientFactory` with resilience handlers.

**Code**
```csharp
// Sync: typed client + resilience (Microsoft.Extensions.Http.Resilience)
builder.Services.AddHttpClient<IInventoryClient, InventoryClient>(c => c.BaseAddress = new Uri("https://inventory"))
    .AddStandardResilienceHandler();
```

### 6.2 [Follow-up] REST vs gRPC?
**Answer**
gRPC uses HTTP/2 and Protocol Buffers: compact, fast, contract-first with code generation and streaming. REST/JSON is simpler, browser-friendly and cache-friendly. gRPC suits internal service-to-service calls; REST suits public APIs.

**Explanation**
In ASP.NET Core, gRPC services derive from the generated base class; clients are registered with `AddGrpcClient<T>`. Protobuf evolution rule: add fields with new numbers, never reuse/renumber.

**Code**
```csharp
// stock.proto: service Stock { rpc Get (StockRequest) returns (StockReply); } (compiled by Grpc.Tools)
public sealed class StockService : Stock.StockBase
{
    public override Task<StockReply> Get(StockRequest request, ServerCallContext context) =>
        Task.FromResult(new StockReply { Quantity = 5 });
}
// Server: builder.Services.AddGrpc(); app.MapGrpcService<StockService>();
// Client: builder.Services.AddGrpcClient<Stock.StockClient>(o => o.Address = new Uri("https://inventory"));
```

### 6.3 [Medium] Explain pub/sub vs competing consumers and give a MassTransit example.
**Answer**
Pub/sub delivers each published event to every subscribed consumer group; competing consumers means several instances of the same consumer share one queue, and each message is processed by exactly one of them, scaling throughput. MassTransit creates a queue per consumer type, so scaling out instances gives competing consumers while different consumer types each receive the event.

**Explanation**
MassTransit abstracts transports (RabbitMQ, Azure Service Bus, in-memory, etc.), providing consumers, retries, outbox, sagas and request/response. Message contracts should be small immutable records in a shared contracts assembly.

**Code**
```csharp
public sealed record OrderSubmitted(Guid OrderId, decimal Total);   // event contract

public sealed class SendReceiptConsumer : IConsumer<OrderSubmitted>
{
    public Task Consume(ConsumeContext<OrderSubmitted> context)
    {
        Console.WriteLine($"Receipt for {context.Message.OrderId}");
        return Task.CompletedTask;
    }
}

// Program.cs  (MassTransit + MassTransit.RabbitMQ)
builder.Services.AddMassTransit(x =>
{
    x.AddConsumer<SendReceiptConsumer>();
    x.UsingRabbitMq((ctx, cfg) =>
    {
        cfg.Host("localhost", "/", h => { h.Username("guest"); h.Password("guest"); });
        cfg.UseMessageRetry(r => r.Exponential(5, TimeSpan.FromSeconds(1), TimeSpan.FromMinutes(1), TimeSpan.FromSeconds(5)));
        cfg.ConfigureEndpoints(ctx);
    });
});
// publish: await publishEndpoint.Publish(new OrderSubmitted(id, 10m));
```

### 6.4 [Hard] Kafka vs RabbitMQ/Service Bus - what are the model differences and ordering guarantees?
**Answer**
Kafka is a partitioned, retained log: consumers track offsets, can replay, and ordering holds only within a partition, so you pick the partition key (e.g. aggregate ID). RabbitMQ/Service Bus are brokers with queues where messages are removed after ack; ordering is per queue, and competing consumers or retries can reorder. Service Bus offers sessions for ordered, grouped processing.

**Explanation**
Use Kafka for high-throughput streams, event retention, replay, analytics; use queues for task distribution, per-message routing, dead-lettering, delayed delivery. Scaling Kafka consumers is capped by partition count. In both cases at-least-once is the realistic default.

**Code**
```csharp
// Ordering key = aggregate id (Confluent.Kafka)
using Confluent.Kafka;
var cfg = new ProducerConfig { BootstrapServers = "localhost:9092", EnableIdempotence = true, Acks = Acks.All };
using var producer = new ProducerBuilder<string, string>(cfg).Build();
await producer.ProduceAsync("orders", new Message<string, string> { Key = orderId.ToString(), Value = json });
```

**Gotchas / follow-ups**
- Kafka `EnableIdempotence` dedups producer retries per partition session, not end-to-end business idempotency.
- Poison messages: need dead-letter queue/topic and retry topology; otherwise one bad message blocks a partition.

### 6.5 [Tricky] A consumer crashes after processing but before ack. What happens, and how do you design for it?
**Answer**
The broker redelivers the message, so it is processed twice; delivery is at-least-once. The consumer must be idempotent (dedupe by message ID or make the operation naturally idempotent), and acknowledge only after the work is durably done.

**Explanation**
Acking before processing gives at-most-once and risks loss. Exactly-once processing end to end is effectively "at-least-once delivery + idempotent processing". See Topic 7 for the inbox pattern.

**Code**
```csharp
public sealed class ReserveStockConsumer(StockDbContext db) : IConsumer<ReserveStock>
{
    public async Task Consume(ConsumeContext<ReserveStock> ctx)
    {
        var id = ctx.MessageId ?? throw new InvalidOperationException("No MessageId");
        if (await db.Processed.AnyAsync(p => p.MessageId == id, ctx.CancellationToken)) return;
        // ... do the work ...
        db.Processed.Add(new ProcessedMessage { MessageId = id });
        await db.SaveChangesAsync(ctx.CancellationToken);   // work + dedupe record in one transaction
    }
}
public sealed record ReserveStock(Guid ProductId, int Qty);
public sealed class ProcessedMessage { public Guid MessageId { get; set; } }
```

**Gotchas / follow-ups**
- Check-then-insert races under competing consumers: protect with a unique key on MessageId.
- Poison message + infinite retry storms: bounded retries + dead-letter.

**Interview traps**
- Claiming brokers provide exactly-once delivery.
- Using HTTP chains for everything, or messaging for everything (request/response over queues needs care).
- Assuming global ordering in Kafka or Service Bus without sessions/partition keys.
- Putting domain entities into message contracts.

**Cheat sheet**
- Command = one receiver; event = fact, many receivers.
- Queue = competing consumers; topic = pub/sub copies.
- gRPC internal/perf; REST public/simple.
- Kafka: log + offsets + partition ordering; queues: ack + DLQ.
- At-least-once + idempotent consumer.

---

## Topic 7: Outbox, Inbox and Idempotency

**Study notes**

```
Dual-write problem:  SaveChanges(db) OK --- crash --- Publish(broker) never happens

Transactional Outbox:
  [ Tx: business rows + OutboxMessages row ]  --commit-->  Relay/Dispatcher polls --> Broker
                                                                (at-least-once, may duplicate)
Inbox (consumer side):
  Broker --> [ Tx: check InboxMessages(MessageId) ; do work ; insert MessageId ]  --> ack
```

- You cannot atomically commit to a DB and a broker without distributed transactions; the outbox writes the message in the same local transaction as the state change, and a background process publishes it.
- Delivery from outbox is at-least-once, so consumers need an **inbox / idempotency key**.
- **Idempotency**: applying an operation many times has the same effect as once (PUT, delete-by-id, upsert; `POST` needs an Idempotency-Key).
- **"Exactly-once" is a myth** at the network level; what exists is effectively-once = at-least-once + idempotent handling (Kafka transactions give exactly-once only within Kafka read-process-write).
- MassTransit and NServiceBus ship an EF Core transactional outbox; CDC (Debezium) is an alternative relay.

### 7.1 [Easy] What problem does the outbox pattern solve?
**Answer**
It solves the dual-write problem: saving state and publishing an event can't be atomic across a DB and a broker. The outbox stores the event in the same DB transaction as the state, and a separate relay publishes it later, guaranteeing no lost events.

**Explanation**
Without it, a crash between `SaveChanges` and `Publish` loses the event; reversing the order can publish an event for a rolled-back change.

**Code**
```csharp
public sealed class OutboxMessage
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public string Type { get; set; } = "";
    public string Payload { get; set; } = "";
    public DateTime OccurredUtc { get; set; } = DateTime.UtcNow;
    public DateTime? ProcessedUtc { get; set; }
}

public async Task PlaceAsync(Order order, CancellationToken ct)
{
    db.Orders.Add(order);
    db.Outbox.Add(new OutboxMessage
    {
        Type = nameof(OrderPlaced),
        Payload = JsonSerializer.Serialize(new OrderPlaced(order.Id))
    });
    await db.SaveChangesAsync(ct);    // one local transaction for both
}
```

### 7.2 [Follow-up] How do you build the relay (dispatcher)?
**Answer**
A `BackgroundService` polls unprocessed outbox rows in order, publishes them to the broker, then marks them processed. Because publish and mark are not atomic, a crash can cause a re-publish, so it is at-least-once.

**Explanation**
For multiple instances, avoid double dispatch with row locks (`UPDLOCK, READPAST` in SQL Server) so instances skip rows others are holding. Add batching, back-off and cleanup of old rows.

**Code**
```sql
SELECT TOP (50) Id, Type, Payload
FROM OutboxMessages WITH (UPDLOCK, READPAST, ROWLOCK)
WHERE ProcessedUtc IS NULL
ORDER BY OccurredUtc;
```
```csharp
public sealed class OutboxRelay(IServiceScopeFactory scopes) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromSeconds(2));
        while (await timer.WaitForNextTickAsync(ct))
        {
            using var scope = scopes.CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<ShopDbContext>();
            var bus = scope.ServiceProvider.GetRequiredService<IPublishEndpoint>(); // scoped service
            var batch = await db.Outbox.Where(m => m.ProcessedUtc == null)
                                       .OrderBy(m => m.OccurredUtc).Take(50).ToListAsync(ct);
            foreach (var m in batch)
            {
                await bus.Publish(new OrderPlaced(JsonSerializer.Deserialize<OrderPlaced>(m.Payload)!.OrderId), ct);
                m.ProcessedUtc = DateTime.UtcNow;
            }
            await db.SaveChangesAsync(ct);
        }
    }
}
```

### 7.3 [Medium] What is the inbox pattern and how does it differ from the outbox?
**Answer**
The outbox guarantees reliable *sending*; the inbox guarantees idempotent *receiving* by recording processed message IDs in the same transaction as the handler's effects. A redelivered message is detected by its ID and skipped.

**Explanation**
Insert-first with a unique constraint is the race-safe way: attempt to insert the MessageId, and if a unique violation occurs, it's a duplicate. Do it in the same transaction as the business change so you can't record without doing, or do without recording.

**Code**
```sql
CREATE TABLE InboxMessages (
    MessageId   UNIQUEIDENTIFIER NOT NULL PRIMARY KEY,
    Consumer    NVARCHAR(200)    NOT NULL,
    ProcessedAt DATETIME2        NOT NULL DEFAULT SYSUTCDATETIME()
);
```
```csharp
public async Task<bool> TryHandleOnceAsync(Guid messageId, Func<Task> work, CancellationToken ct)
{
    await using var tx = await db.Database.BeginTransactionAsync(ct);
    try
    {
        db.Inbox.Add(new InboxMessage { MessageId = messageId });
        await db.SaveChangesAsync(ct);          // unique violation => duplicate
    }
    catch (DbUpdateException)
    {
        return false;                           // already processed
    }
    await work();
    await db.SaveChangesAsync(ct);
    await tx.CommitAsync(ct);
    return true;
}
public sealed class InboxMessage { public Guid MessageId { get; set; } }
```

### 7.4 [Hard] How do you make an HTTP POST idempotent (e.g. payments)?
**Answer**
The client sends an `Idempotency-Key` header; the server stores the key with the request fingerprint and result, and on a repeat returns the stored response instead of re-executing. Key insertion must be atomic (unique constraint) to handle concurrent duplicates.

**Explanation**
Store key + status (InProgress/Completed) + response. A concurrent duplicate while InProgress should return 409 or wait. Validate that the same key isn't reused with a different payload. Expire keys after a window (e.g. 24h).

**Code**
```csharp
app.MapPost("/payments", async (HttpContext http, PaymentRequest req, PaymentsDb db, CancellationToken ct) =>
{
    if (!http.Request.Headers.TryGetValue("Idempotency-Key", out var key) || string.IsNullOrWhiteSpace(key))
        return Results.BadRequest("Idempotency-Key required");

    var existing = await db.IdempotencyKeys.FindAsync([key.ToString()], ct);
    if (existing is not null) return Results.Json(existing.ResponseBody, statusCode: existing.StatusCode);

    var payment = new Payment { Id = Guid.NewGuid(), Amount = req.Amount };
    db.Payments.Add(payment);
    db.IdempotencyKeys.Add(new IdempotencyKey { Key = key!, StatusCode = 201, ResponseBody = payment.Id.ToString() });
    try { await db.SaveChangesAsync(ct); }              // PK on Key => concurrent duplicate fails
    catch (DbUpdateException)
    {
        return Results.Conflict("Duplicate request in progress");
    }
    return Results.Created($"/payments/{payment.Id}", payment.Id);
});
public sealed record PaymentRequest(decimal Amount);
public sealed class Payment { public Guid Id { get; set; } public decimal Amount { get; set; } }
public sealed class IdempotencyKey { public string Key { get; set; } = ""; public int StatusCode { get; set; } public string ResponseBody { get; set; } = ""; }
```

**Gotchas / follow-ups**
- A third-party call (card charge) happening outside your DB transaction still needs its own idempotency key passed downstream.
- Don't key on a timestamp; the client must generate and reuse the key on retry.

### 7.5 [Tricky] "Can't we just use Kafka/Service Bus exactly-once?" What is really guaranteed?
**Answer**
Exactly-once delivery across independent systems is impossible in the general case (two generals problem). Features marketed as exactly-once (Kafka transactions, Service Bus duplicate detection) cover narrow scopes: they dedupe producer retries or make read-process-write atomic within the same system. End-to-end you still need idempotent consumers.

**Explanation**
Service Bus duplicate detection dedupes by MessageId within a time window at the broker, which doesn't protect against a consumer crashing after side effects. Kafka exactly-once semantics holds only when the output is also Kafka and offsets commit in the same transaction.

**Code**
```csharp
// Natural idempotency: upsert rather than insert/increment
await db.Database.ExecuteSqlInterpolatedAsync($@"
MERGE StockLevels AS t
USING (SELECT {productId} AS ProductId) AS s ON t.ProductId = s.ProductId
WHEN MATCHED THEN UPDATE SET Quantity = {absoluteQuantity}   -- absolute set, not 'Quantity += n'
WHEN NOT MATCHED THEN INSERT (ProductId, Quantity) VALUES ({productId}, {absoluteQuantity});", ct);
```

**Gotchas / follow-ups**
- "Increment" operations are not idempotent; "set to X (with version)" are.
- Inbox table growth: purge old entries beyond redelivery window.

**Interview traps**
- Publishing to the broker inside the DB transaction and calling it atomic.
- Dedupe by check-then-insert without unique constraint.
- Believing "exactly-once" settings remove the need for idempotency.
- Outbox relay with multiple instances and no locking.

**Cheat sheet**
- Outbox = reliable publish; inbox = idempotent consume.
- Relay is at-least-once; use `READPAST` for multi-instance.
- Idempotency key + unique constraint.
- Exactly-once = at-least-once + idempotent.
- Prefer set/upsert over increment.

---

## Topic 8: Saga, Distributed Transactions, Eventual Consistency

**Study notes**

```
Orchestration                               Choreography
            +-------------+                  Order --OrderPlaced--> Payment --PaymentTaken--> Stock
 Order <--> | Orchestrator| <--> Payment                \                                         |
            |  (saga)     | <--> Stock               <--StockFailed-- ... compensation events <---+
            +-------------+
 central state machine, explicit flow           no coordinator, implicit flow, harder to trace
```

- 2PC/MSDTC is rarely used in cloud/microservices (locks, coordinator availability, unsupported by brokers/HTTP/most cloud DBs).
- **Saga**: a sequence of local transactions; on failure, run **compensating transactions** (semantic undo, e.g. refund, release stock) for completed steps.
- **Orchestration**: a central saga (MassTransit state machine, NServiceBus saga, Temporal-like workflows) tells participants what to do. **Choreography**: services react to each other's events.
- Sagas give **ACD without I** (no isolation): intermediate states are visible; use semantic locks (e.g. status `Pending`), commutative updates, or reread.
- Eventual consistency: all replicas/services converge if no new updates; design UX and invariants for it.

### 8.1 [Easy] What is a saga?
**Answer**
A saga is a long-running business transaction split into local transactions across services, each publishing the trigger for the next. If a step fails, previous steps are undone with compensating actions rather than a rollback.

**Explanation**
Example: place order -> reserve stock -> charge payment. If payment fails, release stock and cancel the order. Compensations must be idempotent and may not perfectly restore the prior state (an email sent can't be unsent).

**Code**
```csharp
public sealed record OrderSubmitted(Guid OrderId);
public sealed record StockReserved(Guid OrderId);
public sealed record StockReservationFailed(Guid OrderId);
public sealed record PaymentCompleted(Guid OrderId);
public sealed record PaymentFailed(Guid OrderId);
public sealed record ReleaseStock(Guid OrderId);          // compensation
```

### 8.2 [Follow-up] Orchestration vs choreography - trade-offs?
**Answer**
Choreography is decentralised and loosely coupled but the overall flow is implicit, cyclic dependencies appear, and it is hard to debug as steps grow. Orchestration centralises the flow and state in one place making it visible and testable, at the cost of a coordinator that knows all participants. Use choreography for 2-3 simple steps, orchestration for complex or branching processes.

**Explanation**
The orchestrator is still not a distributed lock: it persists state, correlates messages and handles timeouts. Ensure the orchestrator itself is durable (saga state in DB with optimistic concurrency).

**Code**
```csharp
// Choreography: Payment service reacts to an event and emits its own result
public sealed class StockReservedConsumer(IPublishEndpoint bus, IPaymentGateway gw) : IConsumer<StockReserved>
{
    public async Task Consume(ConsumeContext<StockReserved> ctx)
    {
        var ok = await gw.ChargeAsync(ctx.Message.OrderId, ctx.CancellationToken);
        if (ok) await bus.Publish(new PaymentCompleted(ctx.Message.OrderId));
        else    await bus.Publish(new PaymentFailed(ctx.Message.OrderId));
    }
}
public interface IPaymentGateway { Task<bool> ChargeAsync(Guid orderId, CancellationToken ct); }
```

### 8.3 [Medium] Sketch an orchestrated saga with MassTransit.
**Answer**
Define a saga state class (with `CorrelationId` and `CurrentState`) and a `MassTransitStateMachine<TState>` declaring events, states and transitions, with compensation on failure. Persist it with EF Core and an optimistic-concurrency repository.

**Explanation**
Correlate events by `OrderId`. Handle timeouts with schedules. All saga data is stored per instance so the saga is resumable after restarts.

**Code**
```csharp
using MassTransit;

public sealed class OrderState : SagaStateMachineInstance
{
    public Guid CorrelationId { get; set; }
    public string CurrentState { get; set; } = "";
    public int Version { get; set; }          // optimistic concurrency for EF repository
}

public sealed record ReserveStockCmd(Guid OrderId);
public sealed record ChargePaymentCmd(Guid OrderId);
public sealed record CancelOrderCmd(Guid OrderId);

public sealed class OrderStateMachine : MassTransitStateMachine<OrderState>
{
    public State Reserving { get; private set; } = null!;
    public State Charging { get; private set; } = null!;
    public State Completed { get; private set; } = null!;
    public State Failed { get; private set; } = null!;

    public Event<OrderSubmitted> Submitted { get; private set; } = null!;
    public Event<StockReserved> Reserved { get; private set; } = null!;
    public Event<StockReservationFailed> ReservationFailed { get; private set; } = null!;
    public Event<PaymentCompleted> Paid { get; private set; } = null!;
    public Event<PaymentFailed> PaymentFailedEvt { get; private set; } = null!;

    public OrderStateMachine()
    {
        InstanceState(x => x.CurrentState);
        Event(() => Submitted, e => e.CorrelateById(m => m.Message.OrderId));
        Event(() => Reserved, e => e.CorrelateById(m => m.Message.OrderId));
        Event(() => ReservationFailed, e => e.CorrelateById(m => m.Message.OrderId));
        Event(() => Paid, e => e.CorrelateById(m => m.Message.OrderId));
        Event(() => PaymentFailedEvt, e => e.CorrelateById(m => m.Message.OrderId));

        Initially(When(Submitted)
            .Publish(ctx => new ReserveStockCmd(ctx.Message.OrderId))
            .TransitionTo(Reserving));

        During(Reserving,
            When(Reserved).Publish(ctx => new ChargePaymentCmd(ctx.Message.OrderId)).TransitionTo(Charging),
            When(ReservationFailed).Publish(ctx => new CancelOrderCmd(ctx.Message.OrderId)).TransitionTo(Failed));

        During(Charging,
            When(Paid).TransitionTo(Completed).Finalize(),
            When(PaymentFailedEvt)
                .Publish(ctx => new ReleaseStock(ctx.Message.OrderId))   // compensate
                .Publish(ctx => new CancelOrderCmd(ctx.Message.OrderId))
                .TransitionTo(Failed));

        SetCompletedWhenFinalized();
    }
}
// Registration:
// x.AddSagaStateMachine<OrderStateMachine, OrderState>().EntityFrameworkRepository(r =>
// { r.ConcurrencyMode = ConcurrencyMode.Optimistic; r.ExistingDbContext<ShopDbContext>(); });
```

### 8.4 [Hard] What about isolation in sagas, failed compensations and timeouts?
**Answer**
Sagas lack isolation, so other transactions can see partial state; mitigate with semantic locks (`Pending` status), commutative operations, pessimistic ordering (do hard-to-undo steps last) and re-reading values. Compensations are retried until they succeed (they must be idempotent), and timeouts turn silence into a failure path.

**Explanation**
Order steps so *compensatable* steps run first, then a *pivot* step, then *retriable* steps that must eventually succeed. If a compensation can't ever succeed, escalate to manual intervention (dead-letter, alerting, human task).

**Code**
```csharp
// Semantic lock: other flows see the order is in flux
public enum OrderStatus { Pending, Confirmed, Rejected, Cancelled }
// MassTransit timeout via schedule (inside state machine):
// public Schedule<OrderState, PaymentTimeout> PaymentTimeoutSchedule { get; private set; } = null!;
// Schedule(() => PaymentTimeoutSchedule, s => s.TimeoutToken, c => { c.Delay = TimeSpan.FromMinutes(15); c.Received = r => r.CorrelateById(m => m.Message.OrderId); });
public sealed record PaymentTimeout(Guid OrderId);
```

**Gotchas / follow-ups**
- Late arrival: a `PaymentCompleted` arriving after the saga already cancelled -> need refund path.
- Compensations are business actions (refund), not technical rollbacks.

### 8.5 [Tricky] Why not just use TransactionScope/MSDTC across services? And how do you explain eventual consistency to a product owner?
**Answer**
Distributed transactions hold locks across services, couple availability, need XA/DTC support that brokers, HTTP and most cloud stores lack, and don't scale. Eventual consistency means the system will be consistent after a short delay, so the UX shows states like "Processing" and invariants are enforced via reservations, not instant global checks.

**Explanation**
`TransactionScope` is fine for multiple resources on one SQL Server connection scope, but .NET 8 on Linux doesn't support promotion to distributed transactions, and using it across HTTP calls does nothing. Business tolerance is key: overbooking with compensation may be better than blocking sales.

**Code**
```csharp
// Wrong: this does NOT make the HTTP call transactional
using var scope = new TransactionScope(TransactionScopeAsyncFlowOption.Enabled);
await db.SaveChangesAsync(ct);
await http.PostAsJsonAsync("https://billing/charge", dto, ct);   // not enlisted
scope.Complete();
```

**Gotchas / follow-ups**
- Promotion to a distributed transaction (two connections / two DBs) is supported in .NET 7+ on Windows only.
- Saga steps must tolerate duplicates (inbox).

**Interview traps**
- Describing a saga as "rollback across services".
- Compensations that are not idempotent.
- Choreography with 8 services and no way to see the flow.
- Forgetting timeouts and late messages.

**Cheat sheet**
- Saga = local txs + compensations; no isolation.
- Orchestration = central, visible; choreography = decentralised, implicit.
- Order: compensatable -> pivot -> retriable.
- 2PC avoided in microservices.
- Pending/semantic locks for in-flight state.

---

## Topic 9: CAP Theorem and Consistency Models

**Study notes**

```
During a network Partition you must choose:
   C (consistency: every read sees latest write)   OR   A (availability: every request gets a non-error reply)

 CP: reject/wait when cannot reach quorum   (e.g. etcd, ZooKeeper, HBase, Cosmos DB strong)
 AP: answer with possibly stale data        (e.g. Cassandra defaults, DynamoDB eventual, Cosmos eventual)

 PACELC: if Partition -> A or C; Else (normal) -> Latency or Consistency.
```

- Partitions are unavoidable in distributed systems, so the real choice is C vs A *when partitioned*. "CA" distributed systems don't exist.
- CAP's "C" is linearizability, different from ACID's "C" (invariants).
- Consistency spectrum: **strong/linearizable** > sequential > **causal** > **read-your-writes / monotonic reads (session)** > **eventual**.
- Azure Cosmos DB exposes 5 levels: Strong, Bounded Staleness, Session, Consistent Prefix, Eventual.
- SQL Server: single node ACID; Always On readable secondaries are asynchronously updated (stale reads possible).

### 9.1 [Easy] State the CAP theorem.
**Answer**
A distributed data store can provide at most two of: consistency, availability and partition tolerance - but since network partitions happen, you really choose between consistency and availability when a partition occurs. CP systems refuse some requests; AP systems return possibly stale data.

**Explanation**
It describes behaviour *during* partitions only; in normal operation you can have both. It is often over-simplified: consistency and availability are spectrums with latency tradeoffs (PACELC).

**Code**
```csharp
// AP-style choice in code: serve stale cache when the source is unreachable
public async Task<Product?> GetAsync(Guid id, CancellationToken ct)
{
    try
    {
        var p = await remote.GetProductAsync(id, ct);
        cache.Set(id, p, TimeSpan.FromMinutes(5));
        return p;
    }
    catch (HttpRequestException)
    {
        return cache.TryGetValue(id, out Product? stale) ? stale : null;  // availability over consistency
    }
}
```

### 9.2 [Follow-up] Is ACID "C" the same as CAP "C"? Where do SQL Server and Cosmos DB fall?
**Answer**
No. ACID consistency means a transaction takes the database between valid states respecting constraints; CAP consistency means linearizable reads across replicas. A single SQL Server instance is not a distributed system for CAP purposes; Cosmos DB lets you tune the trade-off per account/request.

**Explanation**
For Cosmos, Session consistency (default) gives read-your-writes within a session token and is the commonly right default. Strong consistency costs latency and is limited in multi-region write setups.

**Code**
```csharp
// Microsoft.Azure.Cosmos: relax consistency per request (can only weaken the account default)
var options = new ItemRequestOptions { ConsistencyLevel = ConsistencyLevel.Eventual };
var resp = await container.ReadItemAsync<Product>(id, new PartitionKey(pk), options);
```

### 9.3 [Medium] Explain read-your-writes, monotonic reads and causal consistency with a practical example.
**Answer**
Read-your-writes: after I update my profile, I always see my update. Monotonic reads: I never see data go backwards in time across reads. Causal: if event B depends on A, everyone sees A before B. These session guarantees are cheaper than strong consistency and fix most user-visible anomalies.

**Explanation**
Problem scenario: write to primary, read from lagging replica -> user doesn't see own write. Fixes: route that user's reads to primary for a short window, pass a version/LSN token, or return the result directly from the command.

**Code**
```csharp
// Route to primary for a window after a write (simple "sticky primary" approach)
public sealed class ReadRouter(IMemoryCache cache)
{
    public void MarkWrite(string userId) => cache.Set($"w:{userId}", true, TimeSpan.FromSeconds(5));
    public bool UsePrimary(string userId) => cache.TryGetValue($"w:{userId}", out _);
}
```

### 9.4 [Hard] How do quorums work, and how do you reason about N, R, W?
**Answer**
With N replicas, writing to W and reading from R replicas guarantees that a read sees the latest write if R + W > N because the sets overlap. Lower W or R reduces latency and improves availability but may give stale reads; W = N hurts write availability.

**Explanation**
Common config N=3, W=2, R=2 tolerates one node failure with strong-ish reads (still subject to edge cases like sloppy quorums, clock issues, concurrent writes needing conflict resolution via vector clocks/LWW/CRDTs).

**Code**
```csharp
public static class Quorum
{
    public static bool ReadSeesLatestWrite(int n, int r, int w) => r + w > n;
}
// Quorum.ReadSeesLatestWrite(3, 2, 2) == true ; (3, 1, 1) == false
```

**Gotchas / follow-ups**
- Last-write-wins relies on clocks; skew silently loses writes.
- Quorum overlap alone doesn't give linearizability in all implementations.

### 9.5 [Tricky] Your e-commerce app: which operations require strong consistency and which can be eventual?
**Answer**
Payment capture, inventory decrement for scarce items, and unique constraints (username, order number) need strong consistency or a single authority. Product catalog views, search, recommendations, counters and analytics can be eventual. Decide per use case based on the business cost of staleness versus the cost of unavailability.

**Explanation**
Unique-username across partitions requires a single owner/partition-by-key or a reservation step. For stock, a conditional update (`WHERE Quantity >= @n`) on the authoritative store prevents oversell without distributed locks.

**Code**
```sql
UPDATE Stock
SET Quantity = Quantity - @qty
WHERE ProductId = @pid AND Quantity >= @qty;   -- @@ROWCOUNT = 0 => insufficient stock
```
```csharp
var rows = await db.Database.ExecuteSqlInterpolatedAsync(
    $"UPDATE Stock SET Quantity = Quantity - {qty} WHERE ProductId = {pid} AND Quantity >= {qty}", ct);
if (rows == 0) throw new InvalidOperationException("Insufficient stock");
```

**Gotchas / follow-ups**
- Caches in front of the authoritative check must never be the oversell guard.
- Business can accept overselling + apology; document the decision (ADR).

**Interview traps**
- "Pick any two of three" without noting partitions are not optional.
- Treating CAP's C as ACID's C.
- Calling a single-node RDBMS "CA" as if it's a distributed design.
- Assuming async SQL replicas are read-your-writes safe.

**Cheat sheet**
- Partition => choose C or A.
- PACELC adds latency vs consistency in normal ops.
- Spectrum: linearizable > causal > session > eventual.
- R + W > N => overlapping quorums.
- Use conditional updates for scarce invariants.

---

## Topic 10: API Gateway, BFF, YARP, Discovery, Resilience

**Study notes**

```
 Web app --> [Web BFF]    \                      +--> Orders svc
 Mobile  --> [Mobile BFF]  >--> [API Gateway] --+--> Catalog svc     (gateway: routing, auth, rate-limit,
 Partner --> ----------- /      (YARP/Ocelot/APIM)  +--> Billing svc    TLS termination, aggregation)

 Resilience around each outbound call:  Timeout -> Retry(+jitter) -> Circuit Breaker -> Bulkhead/Rate limiter
```

- **API Gateway**: single entry point: routing, authN/Z offload, rate limiting, caching, TLS, request aggregation. Keep it thin; no business logic.
- **BFF** (Backend for Frontend): one gateway/backend per client type, shaped to that UI's needs.
- **YARP** (Yet Another Reverse Proxy, Microsoft): .NET library for building reverse proxies/gateways, config-driven routes and clusters, load balancing, health checks, transforms. Alternatives: Ocelot, Azure API Management, Envoy.
- **Service discovery**: clients find instances dynamically (Kubernetes DNS/Services, Consul, .NET Aspire/`Microsoft.Extensions.ServiceDiscovery`). **Config**: Azure App Configuration, Kubernetes ConfigMaps/Secrets, Key Vault.
- **Resilience**: Polly v8 / `Microsoft.Extensions.Http.Resilience` (retry, circuit breaker, timeout, hedging, rate limiter).

### 10.1 [Easy] What does an API gateway do, and what is a BFF?
**Answer**
A gateway is the single entry point that routes requests to services and handles cross-cutting concerns like authentication, rate limiting and TLS. A BFF is a gateway dedicated to one client type (web, mobile) that tailors and aggregates responses for it.

**Explanation**
Without a gateway, clients know every service address and each service re-implements auth/limits. Risk: gateway becomes a bottleneck/monolith if it accumulates business logic; BFFs per client team avoid a one-size-fits-all API.

**Code**
```csharp
// A BFF endpoint aggregating two downstream services
app.MapGet("/bff/order-page/{id:guid}", async (Guid id, IOrdersClient orders, ICatalogClient catalog, CancellationToken ct) =>
{
    var order = await orders.GetAsync(id, ct);
    if (order is null) return Results.NotFound();
    var products = await Task.WhenAll(order.ProductIds.Select(p => catalog.GetAsync(p, ct)));
    return Results.Ok(new { order, products });
});
public interface IOrdersClient { Task<OrderView?> GetAsync(Guid id, CancellationToken ct); }
public interface ICatalogClient { Task<ProductView> GetAsync(Guid id, CancellationToken ct); }
public sealed record OrderView(Guid Id, List<Guid> ProductIds);
public sealed record ProductView(Guid Id, string Name);
```

### 10.2 [Follow-up] How do you set up YARP?
**Answer**
Add the `Yarp.ReverseProxy` package, call `AddReverseProxy().LoadFromConfig(...)` and `MapReverseProxy()`. Routes (match paths) map to clusters (destinations) defined in configuration, which can be reloaded at runtime.

**Explanation**
You can attach ASP.NET Core auth policies and rate-limiter policies to routes, add request/response transforms, active/passive health checks and load-balancing policies (RoundRobin, LeastRequests, PowerOfTwoChoices, etc.).

**Code**
```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddReverseProxy().LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));
builder.Services.AddAuthentication().AddJwtBearer();
builder.Services.AddAuthorization();
var app = builder.Build();
app.UseAuthentication(); app.UseAuthorization();
app.MapReverseProxy();
app.Run();
```
```json
{
  "ReverseProxy": {
    "Routes": {
      "orders": { "ClusterId": "orders", "AuthorizationPolicy": "default", "Match": { "Path": "/orders/{**catch-all}" } }
    },
    "Clusters": {
      "orders": {
        "LoadBalancingPolicy": "RoundRobin",
        "Destinations": {
          "d1": { "Address": "https://orders-1:8080/" },
          "d2": { "Address": "https://orders-2:8080/" }
        }
      }
    }
  }
}
```

### 10.3 [Medium] Configure retry + circuit breaker for an HttpClient. How do they interact?
**Answer**
Use `AddResilienceHandler` (or `AddStandardResilienceHandler`) to compose a pipeline. Retry handles transient faults; the circuit breaker opens after a failure ratio threshold to stop hammering a failing dependency and fails fast, then half-opens to probe. Order matters: retry wraps the breaker, so retries count toward breaker statistics and an open circuit fails fast.

**Explanation**
Retry only idempotent operations (or those with idempotency keys), add jitter/exponential backoff to avoid thundering herd, set an overall timeout plus per-attempt timeout, and never retry non-transient errors (4xx).

**Code**
```csharp
// Microsoft.Extensions.Http.Resilience
builder.Services.AddHttpClient("billing", c => c.BaseAddress = new Uri("https://billing"))
    .AddResilienceHandler("billing-pipeline", b =>
    {
        b.AddTimeout(TimeSpan.FromSeconds(10));                       // total
        b.AddRetry(new HttpRetryStrategyOptions
        {
            MaxRetryAttempts = 3,
            BackoffType = DelayBackoffType.Exponential,
            UseJitter = true,
            Delay = TimeSpan.FromMilliseconds(200)
        });
        b.AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions
        {
            SamplingDuration = TimeSpan.FromSeconds(30),
            FailureRatio = 0.5,
            MinimumThroughput = 10,
            BreakDuration = TimeSpan.FromSeconds(15)
        });
        b.AddTimeout(TimeSpan.FromSeconds(2));                        // per attempt
    });
```

### 10.4 [Hard] How does service discovery work in Kubernetes and in .NET, and what about client-side vs server-side load balancing?
**Answer**
In Kubernetes, a `Service` gets a stable DNS name and virtual IP, and kube-proxy load-balances across ready pods (server-side). Client-side discovery has the client query a registry and pick an instance. .NET provides `Microsoft.Extensions.ServiceDiscovery` so `http://orders` can resolve via configuration, DNS SRV, or Aspire/Consul providers.

**Explanation**
gRPC over HTTP/2 uses long-lived connections, so L4 balancing pins all calls from a client to one pod; use client-side balancing (headless service + gRPC load balancing) or an L7 proxy. Health/readiness probes ensure only ready pods receive traffic.

**Code**
```csharp
// Microsoft.Extensions.ServiceDiscovery
builder.Services.AddServiceDiscovery();
builder.Services.ConfigureHttpClientDefaults(http => http.AddServiceDiscovery());
builder.Services.AddHttpClient<IOrdersClient, OrdersHttpClient>(c => c.BaseAddress = new Uri("https+http://orders"));

// health endpoints for orchestrator probes
builder.Services.AddHealthChecks().AddSqlServer(builder.Configuration.GetConnectionString("Db")!);
app.MapHealthChecks("/health/ready");
app.MapHealthChecks("/health/live", new HealthCheckOptions { Predicate = _ => false });
```

**Gotchas / follow-ups**
- Retry storms: retries at every layer multiply (3 x 3 x 3 = 27 calls). Retry at one layer only.
- Liveness probes must not check dependencies or a DB outage restarts every pod.

### 10.5 [Tricky] Your circuit breaker is open - what should the caller do? And can the gateway be a single point of failure?
**Answer**
Fail fast and degrade gracefully: return cached/default data, a partial response, or a clear 503 with Retry-After, depending on the feature. The gateway is a SPOF unless you run multiple stateless instances behind a load balancer, keep it thin and monitored.

**Explanation**
Distinguish `BrokenCircuitException` from timeouts in logging. Fallbacks should be intentional business decisions (e.g. show recommendations empty, but never fake a payment success). Rate limiting at the gateway protects services from overload (ASP.NET Core `AddRateLimiter`).

**Code**
```csharp
try
{
    return await billing.GetInvoiceAsync(id, ct);
}
catch (Polly.CircuitBreaker.BrokenCircuitException)
{
    logger.LogWarning("Billing circuit open, serving cached invoice for {Id}", id);
    return cache.TryGetValue(id, out Invoice? inv) ? inv : throw new ServiceUnavailableException();
}
public sealed class ServiceUnavailableException : Exception { }
```

**Gotchas / follow-ups**
- Resilience handlers in Polly v8 surface `BrokenCircuitException` from `Polly.CircuitBreaker`; verify the exception type for your handler.
- Gateway doing JSON aggregation of 10 services causes latency amplification (slowest wins).

**Interview traps**
- Retrying non-idempotent POSTs blindly.
- Business logic in the gateway.
- Liveness probe depending on external services.
- Retries stacked across multiple layers.

**Cheat sheet**
- Gateway = edge cross-cutting; BFF = per client.
- YARP: routes -> clusters, config-driven.
- Pipeline order: total timeout > retry > breaker > attempt timeout.
- Retry needs idempotency + jitter.
- Liveness != readiness.

---

## Topic 11: Data Ownership and Strangler Fig Migration

**Study notes**

```
Database-per-service                        Shared DB anti-pattern
 [Orders]-->OrdersDB                         [Orders]--\
 [Billing]-->BillingDB                       [Billing]--+--> ONE DB (joins, shared tables)
 data only via API/events                    schema change breaks everyone; hidden coupling

Strangler Fig
  Client --> [Facade/YARP] --route "/orders/*"--> New Orders service
                      \--everything else-------> Legacy monolith       (move routes one at a time)
```

- Each service is the single owner (writer) of its data; others use API or replicated read copies (events).
- Reporting across services: data warehouse, CQRS read models, or API composition.
- **Strangler fig** (Fowler): incrementally replace a legacy system by routing slices of traffic to new components behind a facade until the old one is gone. Needs a seam, anti-corruption layer and data migration strategy (e.g. CDC, dual-run, branch by abstraction).

### 11.1 [Easy] Why database-per-service?
**Answer**
It ensures loose coupling: each service can change its schema, choose its storage technology and scale independently, and no one else writes its data. A shared database turns the schema into a hidden global contract and couples release cycles.

**Explanation**
"Database" can mean separate schema + separate credentials in early stages (logical separation) before physical separation. The key is that no other service reads/writes those tables.

**Code**
```sql
-- Logical separation inside one SQL Server instance (step toward physical split)
CREATE SCHEMA orders;  CREATE SCHEMA billing;
CREATE USER orders_svc  WITHOUT LOGIN WITH DEFAULT_SCHEMA = orders;
CREATE USER billing_svc WITHOUT LOGIN WITH DEFAULT_SCHEMA = billing;
GRANT CONTROL ON SCHEMA::orders  TO orders_svc;
GRANT CONTROL ON SCHEMA::billing TO billing_svc;   -- no cross-schema grants
```

### 11.2 [Follow-up] How do you query data across services without joins?
**Answer**
Options: API composition (a gateway/BFF calls services and merges), CQRS read model fed by events (denormalised copy), or a reporting store/data lake. Choose based on freshness and volume: composition for small lookups, replicated read models for heavy queries.

**Explanation**
Replicating only the fields you need (e.g. product name and price in Orders) accepts staleness. Keep a clear owner and update via events with idempotent handlers.

**Code**
```csharp
// Orders service keeps a local replica of product names, updated by integration events
public sealed record ProductRenamed(Guid ProductId, string NewName, long Version);

public sealed class ProductRenamedConsumer(OrdersDbContext db) : IConsumer<ProductRenamed>
{
    public async Task Consume(ConsumeContext<ProductRenamed> ctx)
    {
        var m = ctx.Message;
        var row = await db.ProductReplicas.FindAsync([m.ProductId], ctx.CancellationToken);
        if (row is null) db.ProductReplicas.Add(new ProductReplica { Id = m.ProductId, Name = m.NewName, Version = m.Version });
        else if (m.Version > row.Version) { row.Name = m.NewName; row.Version = m.Version; }
        await db.SaveChangesAsync(ctx.CancellationToken);
    }
}
public sealed class ProductReplica { public Guid Id { get; set; } public string Name { get; set; } = ""; public long Version { get; set; } }
```

### 11.3 [Medium] Walk through a strangler fig migration of a legacy ASP.NET app.
**Answer**
Put a routing facade (YARP) in front of the legacy app, then peel off one capability at a time: build it as a new service, route its URLs to it, migrate/replicate data, and retire the old code. Start with low-risk, well-bounded slices, and keep the legacy fallback available until verified.

**Explanation**
Steps: (1) identify seams/bounded contexts, (2) introduce the facade with 100% passthrough, (3) implement slice, (4) shadow/compare traffic, (5) switch routing (canary by percentage/header), (6) migrate data (CDC or dual-write with reconciliation), (7) delete legacy code. Use an ACL so the legacy model doesn't leak into new code.

**Code**
```json
{
  "ReverseProxy": {
    "Routes": {
      "new-invoices": { "ClusterId": "invoices-new", "Order": 1, "Match": { "Path": "/invoices/{**rest}" } },
      "legacy":       { "ClusterId": "legacy",       "Order": 100, "Match": { "Path": "/{**rest}" } }
    },
    "Clusters": {
      "invoices-new": { "Destinations": { "d": { "Address": "https://invoices.internal/" } } },
      "legacy":       { "Destinations": { "d": { "Address": "https://legacy.internal/" } } }
    }
  }
}
```

### 11.4 [Hard] How do you split a shared database when extracting a service?
**Answer**
Refactor incrementally: first separate the schema/tables by ownership and remove cross-table joins from code, replace foreign keys and joins with API calls or replicated data, then move the tables to a new database. Use CDC or the outbox to keep data in sync during the transition.

**Explanation**
Hard problems: shared tables (decide an owner), foreign keys across the boundary (replace with ID references plus eventual validation), transactions spanning both (convert to saga), and reporting queries (move to read store). Use the "synchronise data in application" or "database view" interim patterns carefully, and keep a rollback path.

**Code**
```sql
-- Interim: temporary cross-schema view for the old code while it migrates, owned by the extraction team
CREATE VIEW legacy.CustomerNames AS
SELECT Id, Name FROM customers.Customer;     -- remove once callers use the Customers API

-- Replace a cross-boundary foreign key with a soft reference
ALTER TABLE orders.[Order] DROP CONSTRAINT FK_Order_Customer;
-- CustomerId stays as a plain column; validity checked via API/event, not by the DB
```

**Gotchas / follow-ups**
- Dual writes drift: run reconciliation jobs and treat one side as source of truth.
- Big-bang rewrites fail; strangler keeps shipping value.

### 11.5 [Tricky] Business insists on a shared DB "for reporting and joins". What do you say?
**Answer**
Separate the transactional ownership from reporting: services own writes, and a replicated analytics store (data warehouse, read replicas via CDC/events) serves reporting. Reporting directly against operational tables couples schemas and loads production.

**Explanation**
A shared database can be pragmatic inside a modular monolith (schemas per module, no cross-module writes). In a true microservices setup, the shared DB defeats independent deployment. Offer a trade: read-only reporting views over replicas.

**Code**
```sql
-- Read-only reporting login scoped to a replica/reporting database, never the OLTP write DB
CREATE LOGIN report_reader WITH PASSWORD = '<strong-password>';
CREATE USER report_reader FOR LOGIN report_reader;
ALTER ROLE db_datareader ADD MEMBER report_reader;
```

**Gotchas / follow-ups**
- Reporting users hitting OLTP with heavy queries cause lock/CPU contention.
- Schema ownership: who gets to change a column the report depends on?

**Interview traps**
- "Shared DB, separate services" is fine.
- Cross-service foreign keys.
- Big-bang rewrite rather than strangler.
- Forgetting data migration in a strangler plan.

**Cheat sheet**
- One writer per dataset.
- Cross-service queries: composition or replicated read models.
- Strangler = facade + route slice by slice + retire.
- Replace cross-boundary FKs with ID refs.
- Reporting from a replica/warehouse.

---

## Topic 12: GoF Patterns in .NET

**Study notes**

```
Creational: Singleton, Factory Method, Abstract Factory, Builder, Prototype
Structural: Adapter, Decorator, Facade, Proxy      (also Composite, Bridge, Flyweight)
Behavioral: Strategy, Observer, Command, Chain of Responsibility, Template Method, Mediator

Decorator:  Client -> [Logging] -> [Caching] -> [Real service]   same interface, wraps & adds behaviour
Chain:      Request -> [Auth] -> [Validate] -> [Handle]          each may short-circuit (ASP.NET middleware)
```

Where .NET already gives you the pattern:
- Singleton: DI `AddSingleton`. Factory: `IHttpClientFactory`, `ILoggerFactory`. Builder: `WebApplication.CreateBuilder`, `StringBuilder`, `HostBuilder`.
- Prototype: `record` + `with`, `ICloneable` (discouraged). Adapter: wrapping 3rd-party SDKs. Decorator: `Stream` wrappers (`GZipStream`), Scrutor `Decorate`.
- Proxy: `DispatchProxy`, EF lazy-loading proxies, gRPC clients. Facade: simplified service over subsystems.
- Strategy: injected interface / `IComparer<T>` / `Func<>`. Observer: events, `IObservable<T>`, `Channel<T>`. Command: MediatR requests, ICommand in WPF.
- Chain of Responsibility: ASP.NET Core middleware, `DelegatingHandler`. Template Method: base class with abstract steps (`BackgroundService.ExecuteAsync`). Mediator: MediatR.

### 12.1 [Easy] Singleton and Factory in modern .NET - what is the right way?
**Answer**
Don't hand-roll static singletons: register the type as a singleton in DI so lifetime is managed and it stays testable. Use a factory (an interface or `Func<T>`) when you need to choose or create instances at runtime, e.g. by a runtime key.

**Explanation**
A classic `static Instance` hides dependencies and blocks mocking. Singleton services must be thread-safe. If you must hand-roll lazily, use `Lazy<T>` (thread-safe by default).

**Code**
```csharp
public interface INotifier { Task SendAsync(string to, string msg); }
public sealed class EmailNotifier : INotifier { public Task SendAsync(string to, string msg) => Task.CompletedTask; }
public sealed class SmsNotifier : INotifier { public Task SendAsync(string to, string msg) => Task.CompletedTask; }

public interface INotifierFactory { INotifier Create(string channel); }
public sealed class NotifierFactory(IServiceProvider sp) : INotifierFactory
{
    public INotifier Create(string channel) => channel switch
    {
        "email" => sp.GetRequiredService<EmailNotifier>(),
        "sms"   => sp.GetRequiredService<SmsNotifier>(),
        _ => throw new ArgumentException($"Unknown channel {channel}")
    };
}
// builder.Services.AddSingleton<EmailNotifier>(); builder.Services.AddSingleton<SmsNotifier>();
// builder.Services.AddSingleton<INotifierFactory, NotifierFactory>();

// Classic lazy thread-safe singleton (if DI is unavailable)
public sealed class Clock
{
    private static readonly Lazy<Clock> _instance = new(() => new Clock());
    public static Clock Instance => _instance.Value;
    private Clock() { }
}
```

### 12.2 [Follow-up] Abstract Factory, Builder, Prototype, Adapter - give a short .NET example each.
**Answer**
Abstract Factory creates families of related objects behind one interface (e.g. a SQL vs Postgres provider set). Builder constructs complex objects step by step with a fluent API. Prototype copies an existing object (records' `with`). Adapter converts one interface to another so incompatible classes can collaborate.

**Explanation**
Prefer immutable records and `with` for prototype-like cloning. Beware shallow copy: `with` copies references of reference-type members.

**Code**
```csharp
// Abstract Factory
public interface IDbFactory { IDbConnection CreateConnection(); IDbCommand CreateCommand(string sql); }

// Builder
public sealed class EmailBuilder
{
    private string _to = "", _subject = "", _body = "";
    public EmailBuilder To(string to) { _to = to; return this; }
    public EmailBuilder Subject(string s) { _subject = s; return this; }
    public EmailBuilder Body(string b) { _body = b; return this; }
    public Email Build() => string.IsNullOrWhiteSpace(_to) ? throw new InvalidOperationException("To required") : new Email(_to, _subject, _body);
}
public sealed record Email(string To, string Subject, string Body);

// Prototype via record
var original = new Email("a@x.com", "Hi", "Body");
var copy = original with { To = "b@x.com" };

// Adapter: legacy SDK -> our port
public sealed class LegacySmsSdk { public int Push(string number, string text) => 0; }
public interface ISmsSender { bool Send(string to, string text); }
public sealed class LegacySmsAdapter(LegacySmsSdk sdk) : ISmsSender
{
    public bool Send(string to, string text) => sdk.Push(to, text) == 0;
}
```

### 12.3 [Medium] Decorator, Proxy, Facade - how do they differ? Show a Decorator registered via DI.
**Answer**
Decorator adds behaviour to an object while keeping the same interface (caching, logging). Proxy controls access to an object (lazy loading, remote, security) with the same interface. Facade provides a new, simpler interface over a complex subsystem. The difference is intent: add behaviour vs control access vs simplify.

**Explanation**
Decorators are the idiomatic way to add cross-cutting concerns without touching the real service. In plain MS DI you register the decorator via a factory; Scrutor offers `services.Decorate<TService, TDecorator>()`.

**Code**
```csharp
public interface IProductReader { Task<Product?> GetAsync(Guid id, CancellationToken ct); }

public sealed class SqlProductReader(ShopDbContext db) : IProductReader
{
    public Task<Product?> GetAsync(Guid id, CancellationToken ct) => db.Products.FirstOrDefaultAsync(p => p.Id == id, ct);
}

public sealed class CachedProductReader(IProductReader inner, IMemoryCache cache) : IProductReader
{
    public async Task<Product?> GetAsync(Guid id, CancellationToken ct) =>
        await cache.GetOrCreateAsync(id, e =>
        {
            e.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5);
            return inner.GetAsync(id, ct);
        });
}

// Registration without libraries
builder.Services.AddScoped<SqlProductReader>();
builder.Services.AddScoped<IProductReader>(sp =>
    new CachedProductReader(sp.GetRequiredService<SqlProductReader>(), sp.GetRequiredService<IMemoryCache>()));

// Facade
public sealed class CheckoutFacade(IInventory inv, IPayments pay, IShipping ship)
{
    public async Task CheckoutAsync(Guid orderId, CancellationToken ct)
    {
        await inv.ReserveAsync(orderId, ct); await pay.ChargeAsync(orderId, ct); await ship.ScheduleAsync(orderId, ct);
    }
}
public interface IInventory { Task ReserveAsync(Guid id, CancellationToken ct); }
public interface IPayments { Task ChargeAsync(Guid id, CancellationToken ct); }
public interface IShipping { Task ScheduleAsync(Guid id, CancellationToken ct); }
```

### 12.4 [Medium] Strategy, Observer, Command in .NET.
**Answer**
Strategy encapsulates interchangeable algorithms behind an interface chosen at runtime (pricing rules). Observer notifies dependents of changes (C# events, `IObservable<T>`, channels). Command encapsulates a request as an object, enabling queuing, undo and logging.

**Explanation**
Strategy replaces large `switch` statements on type; with DI, inject `IEnumerable<IStrategy>` and pick by a key. C# events have leak risks: subscribers not unsubscribed keep the publisher holding references (memory leak); use weak events or dispose subscriptions.

**Code**
```csharp
// Strategy
public interface IDiscount { string Code { get; } decimal Apply(decimal total); }
public sealed class PercentDiscount(decimal pct) : IDiscount { public string Code => "PCT"; public decimal Apply(decimal t) => t * (1 - pct); }
public sealed class NoDiscount : IDiscount { public string Code => "NONE"; public decimal Apply(decimal t) => t; }
public sealed class PricingService(IEnumerable<IDiscount> strategies)
{
    public decimal Price(string code, decimal total) =>
        (strategies.FirstOrDefault(s => s.Code == code) ?? new NoDiscount()).Apply(total);
}

// Observer (event)
public sealed class Stock
{
    public event EventHandler<int>? QuantityChanged;
    private int _qty;
    public int Quantity { get => _qty; set { _qty = value; QuantityChanged?.Invoke(this, value); } }
}

// Command with undo
public interface ICommand { void Execute(); void Undo(); }
public sealed class AddItemCommand(List<string> list, string item) : ICommand
{
    public void Execute() => list.Add(item);
    public void Undo() => list.Remove(item);
}
```

### 12.5 [Hard] Chain of Responsibility, Template Method, Mediator - and how ASP.NET Core implements them.
**Answer**
Chain of Responsibility passes a request along handlers, each of which can process or forward it - ASP.NET Core middleware and `DelegatingHandler` for HttpClient are exactly this. Template Method defines an algorithm skeleton in a base class with overridable steps (`BackgroundService.ExecuteAsync`). Mediator centralises communication so objects don't reference each other directly (MediatR).

**Explanation**
Middleware order matters (auth before authorization, exception handling first). Mediator trades direct coupling for indirection; overusing it hides the call graph. Prefer composition (delegates/strategy) to Template Method inheritance when steps vary independently.

**Code**
```csharp
// Chain: DelegatingHandler adds a correlation header then forwards
public sealed class CorrelationHandler : DelegatingHandler
{
    protected override Task<HttpResponseMessage> SendAsync(HttpRequestMessage req, CancellationToken ct)
    {
        req.Headers.TryAddWithoutValidation("X-Correlation-Id", Guid.NewGuid().ToString());
        return base.SendAsync(req, ct);   // next handler in the chain
    }
}
// builder.Services.AddTransient<CorrelationHandler>();
// builder.Services.AddHttpClient("x").AddHttpMessageHandler<CorrelationHandler>();

// Middleware chain
app.Use(async (ctx, next) =>
{
    if (!ctx.Request.Headers.ContainsKey("X-Api-Key")) { ctx.Response.StatusCode = 401; return; } // short-circuit
    await next(ctx);
});

// Template Method
public abstract class ReportJob
{
    public async Task RunAsync(CancellationToken ct) { var d = await LoadAsync(ct); var r = Transform(d); await PublishAsync(r, ct); }
    protected abstract Task<string> LoadAsync(CancellationToken ct);
    protected virtual string Transform(string data) => data;
    protected abstract Task PublishAsync(string report, CancellationToken ct);
}
```

**Gotchas / follow-ups**
- `DelegatingHandler` instances must be transient; handler lifetime is managed by HttpClientFactory (pooled ~2 min by default).
- Mediator with a pipeline: handler ordering is registration order.

### 12.6 [Tricky] What goes wrong with singletons in DI (captive dependency), and what is a safe Proxy/lazy pattern for a scoped dependency in a singleton?
**Answer**
A singleton that captures a scoped service (like `DbContext`) keeps it alive for the app lifetime, causing stale data and thread-safety bugs; ASP.NET Core's scope validation throws in Development. Inject `IServiceScopeFactory` (or a factory) and create a scope per operation instead.

**Explanation**
`DbContext` is not thread-safe, so a singleton using one across requests will fail under concurrency. For background workers, create a scope per unit of work. `IDbContextFactory<T>` is the EF Core-provided factory for this exact use.

**Code**
```csharp
public sealed class PriceRefresher(IDbContextFactory<ShopDbContext> factory) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            await using var db = await factory.CreateDbContextAsync(ct);   // short-lived context
            await db.Database.ExecuteSqlRawAsync("UPDATE Products SET Price = Price", ct);
            await Task.Delay(TimeSpan.FromMinutes(1), ct);
        }
    }
}
// builder.Services.AddDbContextFactory<ShopDbContext>(o => o.UseSqlServer(cs));
// builder.Services.AddHostedService<PriceRefresher>();
```

**Gotchas / follow-ups**
- Scope validation runs by default only in the Development environment.
- Singletons holding mutable state need locking or immutable snapshots.
- Static caches grow unbounded: use `MemoryCache` with size/expiry limits.

**Interview traps**
- Static `Instance` singleton in an app that already has DI.
- Confusing Decorator with Proxy/Facade (intent differs).
- Event handlers never unsubscribed -> leaks.
- Template Method inheritance chains that are untestable.

**Cheat sheet**
- Singleton = DI lifetime; stateless or thread-safe.
- Factory when type chosen at runtime; `IHttpClientFactory`.
- Decorator = same interface + extra behaviour; Proxy = access control; Facade = simplify.
- Chain = middleware/DelegatingHandler.
- Strategy replaces switch; inject `IEnumerable<T>`.
- Never capture scoped in singleton.

---

## Topic 13: Enterprise Patterns

**Study notes**

```
Specification:  Spec<T> -> Expression<Func<T,bool>>  -> composed with And/Or -> translated to SQL by EF
Repository + Unit of Work:  Handler -> IRepo -> DbContext (DbSet = repo, DbContext = UoW)
Result:  Result<T> = Success(value) | Failure(error)   (no exceptions for expected failures)
Options:  appsettings section -> IOptions<T> / IOptionsSnapshot<T> / IOptionsMonitor<T>
Null Object:  NullLogger / NoDiscount instead of null checks
DIP:  High-level policy owns the interface; low-level detail implements it
```

- **Repository**: collection-like access per aggregate root. **Unit of Work**: tracks changes and commits atomically - EF Core's `DbContext` already is one.
- **Specification**: encapsulates a query criterion as a reusable object.
- **Result pattern**: model expected failures (validation, not found) as values, reserving exceptions for exceptional conditions.
- **Options pattern**: strongly typed config binding; `IOptions<T>` (singleton, read once), `IOptionsSnapshot<T>` (scoped, reloads per request), `IOptionsMonitor<T>` (singleton, change notifications).
- **Dependency Inversion Principle** (the D in SOLID): depend on abstractions; high-level modules own them. DI container and Inversion of Control are *mechanisms*; DIP is the design principle.

### 13.1 [Easy] Is a Repository + Unit of Work over EF Core redundant?
**Answer**
Largely yes for generic wrappers, because `DbSet<T>` already behaves like a repository and `DbContext` like a unit of work. A repository still pays off when it expresses aggregate-specific, domain-language operations and hides persistence from the domain, or to ease testing.

**Explanation**
Generic `IRepository<T>` with `GetAll() : IQueryable<T>` leaks persistence and offers little. A better design is one narrow repository per aggregate root, with `SaveChanges` handled by a UoW (or a pipeline behavior committing after each command).

**Code**
```csharp
public interface IUnitOfWork { Task<int> SaveChangesAsync(CancellationToken ct = default); }
public interface IOrderRepository
{
    Task<Order?> GetWithLinesAsync(Guid id, CancellationToken ct);
    void Add(Order order);
}
public sealed class OrderRepository(ShopDbContext db) : IOrderRepository
{
    public Task<Order?> GetWithLinesAsync(Guid id, CancellationToken ct) =>
        db.Orders.Include(o => o.Lines).SingleOrDefaultAsync(o => o.Id == id, ct);
    public void Add(Order order) => db.Orders.Add(order);
}
// DbContext implements IUnitOfWork: public sealed class ShopDbContext : DbContext, IUnitOfWork { ... }
```

### 13.2 [Follow-up] Implement the Specification pattern with EF Core.
**Answer**
A specification exposes an `Expression<Func<T,bool>>` (and optionally includes/ordering), so EF translates it into SQL. Specifications can be combined, and reused across queries and domain checks.

**Explanation**
Use expressions (not `Func`) otherwise EF would pull all rows to memory and filter client-side. Combining expressions requires rewriting parameters; a small helper or a library (Ardalis.Specification) handles this.

**Code**
```csharp
public abstract class Spec<T>
{
    public abstract Expression<Func<T, bool>> Criteria { get; }
}
public sealed class ActiveCustomersSpec : Spec<Customer>
{
    public override Expression<Func<Customer, bool>> Criteria => c => c.IsActive;
}
public sealed class CustomerInCitySpec(string city) : Spec<Customer>
{
    public override Expression<Func<Customer, bool>> Criteria => c => c.City == city;
}

public static class SpecExtensions
{
    public static IQueryable<T> Where<T>(this IQueryable<T> q, Spec<T> spec) => q.Where(spec.Criteria);

    public static Expression<Func<T, bool>> And<T>(this Expression<Func<T, bool>> a, Expression<Func<T, bool>> b)
    {
        var p = Expression.Parameter(typeof(T));
        var body = Expression.AndAlso(
            new ReplaceVisitor(a.Parameters[0], p).Visit(a.Body),
            new ReplaceVisitor(b.Parameters[0], p).Visit(b.Body));
        return Expression.Lambda<Func<T, bool>>(body, p);
    }
    private sealed class ReplaceVisitor(Expression from, Expression to) : ExpressionVisitor
    {
        public override Expression? Visit(Expression? node) => node == from ? to : base.Visit(node);
    }
}
public sealed class Customer { public Guid Id { get; set; } public bool IsActive { get; set; } public string City { get; set; } = ""; }

// usage: db.Customers.Where(new ActiveCustomersSpec().Criteria.And(new CustomerInCitySpec("Pune").Criteria))
```

### 13.3 [Medium] Result pattern vs exceptions - when and how?
**Answer**
Use exceptions for unexpected, exceptional faults (infrastructure errors, bugs) and Result for expected business outcomes (validation failed, not found, conflict), which are part of normal flow. It makes failure explicit in signatures, avoids exception cost and control-flow-by-exception, and maps cleanly to HTTP responses.

**Explanation**
Make `Result` a small immutable type with an `Error` (code + message). Map to `ProblemDetails`/status codes at the API edge. Don't swallow genuine exceptions into Results.

**Code**
```csharp
public sealed record Error(string Code, string Message)
{
    public static readonly Error None = new("", "");
}

public readonly struct Result<T>
{
    private readonly T? _value;
    public Error Error { get; }
    public bool IsSuccess => Error == Error.None;
    public T Value => IsSuccess ? _value! : throw new InvalidOperationException("No value on failure");
    private Result(T? v, Error e) { _value = v; Error = e; }
    public static Result<T> Success(T v) => new(v, Error.None);
    public static Result<T> Failure(Error e) => new(default, e);
    public TOut Match<TOut>(Func<T, TOut> ok, Func<Error, TOut> fail) => IsSuccess ? ok(_value!) : fail(Error);
}

public static Result<Order> Cancel(Order o) =>
    o.Status == OrderStatus.Shipped
        ? Result<Order>.Failure(new Error("Order.AlreadyShipped", "Cannot cancel a shipped order"))
        : Result<Order>.Success(o);

// endpoint
app.MapPost("/orders/{id}/cancel", async (Guid id, IOrderRepository r, CancellationToken ct) =>
{
    var order = await r.GetWithLinesAsync(id, ct);
    if (order is null) return Results.NotFound();
    return Cancel(order).Match(_ => Results.NoContent(), e => Results.Problem(e.Message, statusCode: 409));
});
```

### 13.4 [Medium] Explain IOptions, IOptionsSnapshot and IOptionsMonitor, with validation.
**Answer**
`IOptions<T>` is a singleton snapshot read once at first use; `IOptionsSnapshot<T>` is scoped and recomputed per request (so config reloads apply per request); `IOptionsMonitor<T>` is a singleton that always returns the current value and raises change notifications. Validate at startup with `ValidateDataAnnotations().ValidateOnStart()`.

**Explanation**
Use `IOptionsMonitor` inside singletons/background services needing live updates. `IOptionsSnapshot` can't be injected into singletons (it's scoped). Named options support multiple instances of the same type.

**Code**
```csharp
public sealed class SmtpOptions
{
    public const string Section = "Smtp";
    [Required] public string Host { get; init; } = "";
    [Range(1, 65535)] public int Port { get; init; } = 25;
}

builder.Services.AddOptions<SmtpOptions>()
    .Bind(builder.Configuration.GetSection(SmtpOptions.Section))
    .ValidateDataAnnotations()
    .ValidateOnStart();

public sealed class Mailer(IOptionsMonitor<SmtpOptions> opts)
{
    public string Endpoint => $"{opts.CurrentValue.Host}:{opts.CurrentValue.Port}";
}
```

### 13.5 [Hard] DIP vs DI vs IoC - and where does DIP get violated in "clean" codebases?
**Answer**
DIP is the principle that high-level policy shouldn't depend on low-level details; both depend on abstractions owned by the high-level side. IoC is the broader idea of the framework controlling flow, and DI is a technique to supply dependencies. Violations: interfaces defined next to implementations, abstractions that leak details (`IRepository` exposing `IQueryable` or `DbContext`), and service locator usage.

**Explanation**
An interface like `ISqlOrderStore.ExecuteStoredProc(...)` is a thin alias of the implementation, not an abstraction. Good abstractions speak the domain language (`IOrderRepository.GetOpenOrdersFor(customer)`). Prefer constructor injection; avoid injecting `IServiceProvider` except for factories.

**Code**
```csharp
// Application (high-level) owns this abstraction
public interface IClock { DateTimeOffset UtcNow { get; } }
public interface IEmailSender { Task SendAsync(string to, string subject, string body, CancellationToken ct); }

public sealed class WelcomeService(IEmailSender email, IClock clock)
{
    public Task WelcomeAsync(string to, CancellationToken ct) =>
        email.SendAsync(to, "Welcome", $"Joined at {clock.UtcNow:u}", ct);
}
// .NET 8 also ships TimeProvider: prefer it over a custom IClock in new code
public sealed class Expiry(TimeProvider time)
{
    public bool IsExpired(DateTimeOffset at) => time.GetUtcNow() >= at;
}
```

**Gotchas / follow-ups**
- Service locator (`sp.GetService<T>()` inside business code) hides dependencies and makes tests brittle.
- Over-abstraction: don't wrap `List<T>` or `DateTime` unless it buys testability.

### 13.6 [Tricky] Null Object pattern and EF Core nullable pitfalls
**Answer**
Null Object provides a do-nothing implementation so callers don't null-check (`NullLogger<T>.Instance`, `NoDiscount`). It's best for behavioural dependencies, not for data: returning a "null customer" object may hide a real not-found condition.

**Explanation**
For lookups, prefer `Result`/`Option` or an exception; for optional collaborators (loggers, notifiers, discounts) the null object is ideal. Records with `with` can express "empty" instances (`Address.Empty`) but beware conflating "unknown" with "empty".

**Code**
```csharp
public interface IAuditSink { void Record(string what); }
public sealed class NullAuditSink : IAuditSink
{
    public static readonly NullAuditSink Instance = new();
    public void Record(string what) { }
}
public sealed class TransferService(IAuditSink? audit = null)
{
    private readonly IAuditSink _audit = audit ?? NullAuditSink.Instance;
    public void Transfer() { /* ... */ _audit.Record("transfer"); }
}
// Built-in: Microsoft.Extensions.Logging.Abstractions.NullLogger<T>.Instance
```

**Gotchas / follow-ups**
- Silent no-ops can mask misconfiguration (auditing silently disabled): log at startup.

**Interview traps**
- Wrapping EF in a generic repository exposing `IQueryable`.
- Passing `Func<T,bool>` to EF queries (client evaluation).
- Using exceptions for "user not found" in hot paths.
- Injecting `IOptionsSnapshot` into a singleton.
- Interface in the same assembly as implementation = DIP violated.

**Cheat sheet**
- DbContext = UoW; DbSet = repository.
- Spec = `Expression<Func<T,bool>>`, composable.
- Result for expected failures; exceptions for faults.
- IOptions (once) / Snapshot (per request) / Monitor (live).
- DIP: high-level owns the abstraction.
- Null Object for optional behaviour.

---

## Topic 14: Cross-Cutting Concerns and Domain vs Integration Events

**Study notes**

```
Request --> [Logging] --> [Validation] --> [Caching] --> [Transaction/UoW] --> Handler
            (MediatR IPipelineBehavior  |  Decorators  |  ASP.NET middleware/filters)

Domain event:      in-process, inside a bounded context, may carry domain objects/IDs, same transaction
Integration event: cross-service contract, versioned, serialised, async via broker (via outbox)
   Aggregate --raises--> DomainEvent --handler--> maps --> IntegrationEvent --> outbox --> broker
```

- Cross-cutting concerns implemented once at a pipeline seam: middleware (HTTP-level), filters (MVC), MediatR behaviors, or decorators over handlers.
- **Domain events** express things that happened in the domain; handlers are in the same process/bounded context.
- **Integration events** are public contracts to other services: stable, versioned, minimal, no internal types. Don't publish domain events directly.

### 14.1 [Easy] How do you add logging/validation to every handler without repeating code?
**Answer**
Put the concern in a pipeline: a MediatR `IPipelineBehavior<TRequest,TResponse>`, or a decorator around the handler interface. The behavior runs before/after each handler, so the handlers stay focused on business logic.

**Explanation**
Order matters: logging outermost, then validation, then transaction. Behaviors are registered in the order they are added; the first registered is the outermost.

**Code**
```csharp
public sealed class LoggingBehavior<TRequest, TResponse>(ILogger<LoggingBehavior<TRequest, TResponse>> log)
    : IPipelineBehavior<TRequest, TResponse> where TRequest : notnull
{
    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        var name = typeof(TRequest).Name;
        var sw = Stopwatch.StartNew();
        log.LogInformation("Handling {Request}", name);
        try { return await next(); }
        finally { log.LogInformation("Handled {Request} in {Ms} ms", name, sw.ElapsedMilliseconds); }
    }
}
// builder.Services.AddMediatR(c => {
//   c.RegisterServicesFromAssemblyContaining<Program>();
//   c.AddOpenBehavior(typeof(LoggingBehavior<,>));
//   c.AddOpenBehavior(typeof(ValidationBehavior<,>)); });
```

### 14.2 [Follow-up] Implement a validation behavior with FluentValidation and a caching behavior.
**Answer**
The validation behavior resolves all `IValidator<TRequest>`, runs them and throws (or returns a Result) on failures before calling the handler. The caching behavior applies only to queries marked with a cache key interface and serves from `IMemoryCache`/`IDistributedCache`.

**Explanation**
Cache only queries, never commands; include the user/tenant in the key if results differ; invalidate on write or use short TTL. Keep exception-based validation mapped to HTTP 400 `ValidationProblemDetails` in a global handler.

**Code**
```csharp
public sealed class ValidationBehavior<TRequest, TResponse>(IEnumerable<IValidator<TRequest>> validators)
    : IPipelineBehavior<TRequest, TResponse> where TRequest : notnull
{
    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        if (validators.Any())
        {
            var ctx = new ValidationContext<TRequest>(request);
            var failures = (await Task.WhenAll(validators.Select(v => v.ValidateAsync(ctx, ct))))
                .SelectMany(r => r.Errors).Where(f => f is not null).ToList();
            if (failures.Count > 0) throw new ValidationException(failures);
        }
        return await next();
    }
}

public interface ICachedQuery { string CacheKey { get; } TimeSpan Ttl { get; } }
public sealed class CachingBehavior<TRequest, TResponse>(IMemoryCache cache)
    : IPipelineBehavior<TRequest, TResponse> where TRequest : notnull, ICachedQuery
{
    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        if (cache.TryGetValue(request.CacheKey, out TResponse? hit) && hit is not null) return hit;
        var result = await next();
        cache.Set(request.CacheKey, result, request.Ttl);
        return result;
    }
}
```

### 14.3 [Medium] Domain event vs integration event - what is the difference and how do you convert between them?
**Answer**
A domain event is internal to a bounded context, raised by an aggregate and handled in-process, possibly within the same transaction. An integration event is a versioned, serialisable contract published to other services over a broker. A domain event handler maps the former to the latter and writes it to the outbox.

**Explanation**
Reasons not to publish domain events directly: they evolve with your internal model, may contain internal details, and become a hard-to-change public API. Integration events carry the minimum data needed, with IDs and timestamps, and are additive-only.

**Code**
```csharp
// Domain event (internal)
public sealed record OrderPlacedDomainEvent(Guid OrderId, Guid CustomerId, Money Total) : INotification;

// Integration event (public contract in a Contracts assembly)
public sealed record OrderPlacedIntegrationEvent(Guid EventId, Guid OrderId, Guid CustomerId, decimal Total, string Currency, DateTimeOffset OccurredAt);

public sealed class OrderPlacedHandler(ShopDbContext db) : INotificationHandler<OrderPlacedDomainEvent>
{
    public Task Handle(OrderPlacedDomainEvent e, CancellationToken ct)
    {
        var ie = new OrderPlacedIntegrationEvent(Guid.NewGuid(), e.OrderId, e.CustomerId, e.Total.Amount, e.Total.Currency, DateTimeOffset.UtcNow);
        db.Outbox.Add(new OutboxMessage { Type = nameof(OrderPlacedIntegrationEvent), Payload = JsonSerializer.Serialize(ie) });
        return Task.CompletedTask;   // persisted by the surrounding SaveChanges (same transaction)
    }
}
```

### 14.4 [Hard] Where should the transaction boundary and SaveChanges live in a pipeline, and what's the danger with domain event handlers?
**Answer**
Open one transaction per command in a behavior (or call `SaveChanges` once at the end of the command), dispatching domain events before the commit so handler changes and outbox rows join the same transaction. The danger is handlers that call external systems, or that raise more events and create hidden cascades/ordering problems.

**Explanation**
Two models: (a) dispatch before save: atomic, but handlers must stay local; (b) dispatch after save: handlers can fail independently - need outbox for reliability. Keep domain event handlers fast and side-effect-free externally; external effects go through integration events + outbox.

**Code**
```csharp
public sealed class TransactionBehavior<TRequest, TResponse>(ShopDbContext db)
    : IPipelineBehavior<TRequest, TResponse> where TRequest : notnull, ICommandMarker
{
    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        var strategy = db.Database.CreateExecutionStrategy();   // needed when SqlServer retry-on-failure is enabled
        return await strategy.ExecuteAsync(async () =>
        {
            await using var tx = await db.Database.BeginTransactionAsync(ct);
            var response = await next();
            await db.SaveChangesAsync(ct);
            await tx.CommitAsync(ct);
            return response;
        });
    }
}
public interface ICommandMarker { }
```

**Gotchas / follow-ups**
- User-initiated transactions with `EnableRetryOnFailure` require wrapping in an execution strategy (shown above) or EF throws.
- Re-raising events inside handlers risks infinite loops.

### 14.5 [Tricky] A MediatR behavior vs ASP.NET middleware vs MVC filter vs decorator - how do you choose?
**Answer**
Middleware works on the HTTP pipeline for all requests (correlation IDs, exception handling, auth); filters apply to MVC/minimal endpoint execution with access to action context (model-state, result shaping); behaviors/decorators operate on application use cases regardless of the entry point (HTTP, queue, scheduler). Choose the narrowest layer that covers every entry point you need.

**Explanation**
Validation in middleware only protects HTTP; consumers from a queue would bypass it. Put business-level cross-cutting concerns (transaction, validation, audit) at the use-case layer and transport-level ones (CORS, compression, auth) at HTTP.

**Code**
```csharp
// Global exception -> ProblemDetails mapping (transport-level concern) in .NET 8
public sealed class GlobalExceptionHandler(ILogger<GlobalExceptionHandler> log) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(HttpContext ctx, Exception ex, CancellationToken ct)
    {
        log.LogError(ex, "Unhandled");
        var status = ex is ValidationException ? 400 : 500;
        ctx.Response.StatusCode = status;
        await ctx.Response.WriteAsJsonAsync(new ProblemDetails { Status = status, Title = "Request failed" }, ct);
        return true;
    }
}
// builder.Services.AddExceptionHandler<GlobalExceptionHandler>(); builder.Services.AddProblemDetails();
// app.UseExceptionHandler();
```

**Gotchas / follow-ups**
- Behaviors are not applied to `INotification` handlers by default (only `IRequest`).
- Don't cache in a behavior on queries that depend on the user without the user in the key.

**Interview traps**
- Putting business rules into behaviors.
- Publishing domain events straight to the broker.
- Calling external APIs inside domain event handlers in the transaction.
- Forgetting `CreateExecutionStrategy` with manual transactions and retries.

**Cheat sheet**
- Behaviors/decorators = use-case-level cross-cutting.
- Middleware = HTTP-level.
- Domain event = internal; integration event = public contract.
- Map domain -> integration, write to outbox.
- One transaction per command.

---

## Topic 15: Multi-Tenancy and Scalability Patterns

**Study notes**

```
Multi-tenancy models                                   Scalability levers
 1. DB per tenant      (strong isolation, costly)       Vertical: bigger box
 2. Schema per tenant  (middle)                         Horizontal: stateless instances + LB
 3. Shared DB, TenantId column (cheap, risk of leaks)   Cache: in-memory -> distributed (Redis) -> CDN
                                                        Read replicas (CQRS), partitioning/sharding
 Tenant resolution: host / header / route / JWT claim   Async: queues for load levelling
```

- **Tenant resolution** in middleware -> scoped `ITenantContext`. Enforce isolation with EF **global query filters** (shared DB) plus Row-Level Security in SQL Server as defence in depth.
- Noisy-neighbour control: per-tenant rate limits and quotas.
- **Sharding**: split data horizontally by a shard key (tenant, customer id); hot partitions, cross-shard queries and rebalancing are the pain. SQL Server options: partitioned tables (same DB), elastic database tools/Azure SQL elastic pools, or app-level routing.
- **Caching**: cache-aside most common; invalidation and stampede protection are the hard parts. CDN for static/public cacheable content.

### 15.1 [Easy] What multi-tenancy models exist and how do you choose?
**Answer**
Database-per-tenant gives the strongest isolation and per-tenant backup/restore but costs more and is harder to operate at scale. Schema-per-tenant is a middle ground, and a shared database with a TenantId column is cheapest and scales to many tenants but needs rigorous query isolation. Choose by compliance needs, tenant count and size variance.

**Explanation**
Hybrid is common: small tenants share a database, large/regulated tenants get dedicated databases. Azure SQL elastic pools support many tenant databases cost-effectively.

**Code**
```csharp
public interface ITenantContext { string TenantId { get; } }
public sealed class TenantContext : ITenantContext { public string TenantId { get; set; } = ""; }

app.Use(async (ctx, next) =>
{
    var tenant = ctx.User.FindFirst("tid")?.Value ?? ctx.Request.Headers["X-Tenant"].FirstOrDefault();
    if (string.IsNullOrEmpty(tenant)) { ctx.Response.StatusCode = 400; return; }
    ctx.RequestServices.GetRequiredService<TenantContext>().TenantId = tenant;
    await next();
});
// builder.Services.AddScoped<TenantContext>(); builder.Services.AddScoped<ITenantContext>(sp => sp.GetRequiredService<TenantContext>());
```

### 15.2 [Follow-up] Isolate tenants in a shared database with EF Core.
**Answer**
Add a `TenantId` to tenant-owned entities and a global query filter that references the current tenant from the DbContext. Also stamp `TenantId` on insert, and consider SQL Server Row-Level Security so a bug in the app can't read across tenants.

**Explanation**
The filter's tenant value must be a DbContext member (EF re-evaluates per context instance), not a captured constant - this is the well-known pattern for per-request filters. Use `IgnoreQueryFilters()` deliberately for admin tasks. Raw SQL and `FromSql` still need the filter applied manually (they compose with filters only when composed with LINQ).

**Code**
```csharp
public sealed class TenantDbContext(DbContextOptions<TenantDbContext> o, ITenantContext tenant) : DbContext(o)
{
    public DbSet<Invoice> Invoices => Set<Invoice>();
    private string CurrentTenant => tenant.TenantId;   // evaluated per context instance

    protected override void OnModelCreating(ModelBuilder b) =>
        b.Entity<Invoice>().HasQueryFilter(i => i.TenantId == CurrentTenant);

    public override Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        foreach (var e in ChangeTracker.Entries<Invoice>().Where(e => e.State == EntityState.Added))
            e.Entity.TenantId = CurrentTenant;
        return base.SaveChangesAsync(ct);
    }
}
public sealed class Invoice { public Guid Id { get; set; } public string TenantId { get; set; } = ""; public decimal Total { get; set; } }
```
```sql
-- SQL Server Row-Level Security as defence in depth
CREATE FUNCTION dbo.fn_TenantFilter(@TenantId NVARCHAR(50))
RETURNS TABLE WITH SCHEMABINDING AS
RETURN SELECT 1 AS ok WHERE @TenantId = CAST(SESSION_CONTEXT(N'TenantId') AS NVARCHAR(50));
GO
CREATE SECURITY POLICY dbo.TenantPolicy
ADD FILTER PREDICATE dbo.fn_TenantFilter(TenantId) ON dbo.Invoices,
ADD BLOCK  PREDICATE dbo.fn_TenantFilter(TenantId) ON dbo.Invoices;
-- app sets: EXEC sp_set_session_context 'TenantId', @tenant;
```

### 15.3 [Medium] How do you implement caching correctly (cache-aside, stampede, invalidation)?
**Answer**
Cache-aside: read from cache; on miss load from the source and populate it; on write update the DB then invalidate the key. Protect against stampedes (many concurrent misses) with per-key locking or `HybridCache` (.NET 9) / `GetOrCreateAsync` with single-flight; always set TTLs.

**Explanation**
Hard truths: invalidation is the difficulty; caches can serve stale data, so decide what's acceptable. `IMemoryCache.GetOrCreateAsync` doesn't guarantee a single factory execution per key under concurrency; combine with `SemaphoreSlim` per key or use `Lazy<Task<T>>`. In multi-instance deployments use `IDistributedCache` (Redis), and include tenant in keys.

**Code**
```csharp
public sealed class ProductCache(IMemoryCache cache, IProductReader reader)
{
    private static readonly SemaphoreSlim Gate = new(1, 1);

    public async Task<Product?> GetAsync(Guid id, CancellationToken ct)
    {
        if (cache.TryGetValue(id, out Product? p)) return p;
        await Gate.WaitAsync(ct);
        try
        {
            if (cache.TryGetValue(id, out p)) return p;          // double-check
            p = await reader.GetAsync(id, ct);
            cache.Set(id, p, new MemoryCacheEntryOptions { AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5) });
            return p;
        }
        finally { Gate.Release(); }
    }
    public void Invalidate(Guid id) => cache.Remove(id);
}
```

### 15.4 [Hard] Describe sharding, its shard-key choice and the problems it introduces.
**Answer**
Sharding splits data across multiple databases by a shard key so each holds a subset, scaling writes and storage beyond a single node. Choose a key with high cardinality and even distribution that matches your dominant access path (e.g. TenantId), because cross-shard queries, joins and transactions are expensive and re-sharding is painful.

**Explanation**
Strategies: range, hash, directory/lookup-based. Hot shards (celebrity tenants) need splitting; auto-increment IDs complicate uniqueness, so use GUIDs/ULIDs/Snowflake-like IDs. Often first try: read replicas, caching, partitioned tables, archiving - sharding is a last resort.

**Code**
```csharp
public sealed class ShardRouter(IReadOnlyList<string> connectionStrings)
{
    // Stable hash; do not use string.GetHashCode (randomised per process)
    public string For(Guid tenantId)
    {
        var bytes = tenantId.ToByteArray();
        var h = BitConverter.ToUInt32(System.IO.Hashing.XxHash32.Hash(bytes));
        return connectionStrings[(int)(h % (uint)connectionStrings.Count)];
    }
}
// Note: modulo hashing remaps most keys when the shard count changes -> use consistent hashing or a directory table.
```

**Gotchas / follow-ups**
- `string.GetHashCode()` differs between processes in .NET Core: never use it for routing.
- `System.IO.Hashing` is a separate NuGet package.

### 15.5 [Tricky] Read replicas + CQRS in SQL Server: what are the hazards?
**Answer**
Readable secondaries (Always On availability groups) are replicated asynchronously by default, so reads can be stale and a user may not see their own write. Route read-only queries with `ApplicationIntent=ReadOnly` for reads that tolerate staleness and keep read-after-write paths on the primary.

**Explanation**
Also snapshot-isolation overhead on secondaries, and queries on a secondary during failover. Replica lag monitoring and a fallback to primary when lag exceeds a threshold keep UX sane.

**Code**
```csharp
// Separate contexts: writes to primary, reads to the readable secondary
builder.Services.AddDbContext<WriteDbContext>(o => o.UseSqlServer(cfg["Sql:Primary"]));
builder.Services.AddDbContext<ReadDbContext>(o => o.UseSqlServer(cfg["Sql:Secondary"]) // Server=listener;...;ApplicationIntent=ReadOnly;MultiSubnetFailover=True
    .UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking));
```

**Gotchas / follow-ups**
- Read replicas scale reads, not writes.
- Caching hides replica lag differently - layering caches and replicas multiplies staleness.

**Interview traps**
- Global query filter capturing a constant tenant instead of a context member.
- Forgetting tenant in cache keys (cross-tenant data leak).
- Using shared DB multi-tenancy without a second line of defence (RLS).
- Using `string.GetHashCode` for sharding.

**Cheat sheet**
- Tenancy: DB-per / schema-per / shared-column; hybrid is common.
- Filter + RLS = defence in depth.
- Cache-aside + TTL + single-flight.
- Shard key = high cardinality, matches access path.
- Replicas scale reads; expect lag.

---

## Topic 16: Versioning, Observability, Feature Flags, Deployment, ADRs

**Study notes**

```
API versioning: URL (/v2/orders) | header (api-version) | query (?api-version=2.0) | media type
Backward compatible (safe): add optional fields/endpoints.  Breaking: remove/rename/retype, change semantics.

Observability = Logs + Metrics + Traces (OpenTelemetry), correlated by trace id
  Svc A --traceparent--> Svc B --traceparent--> Svc C          (W3C Trace Context)

Deploy: Blue/Green (swap whole env)   Canary (1% -> 10% -> 100%)   Feature flags decouple deploy from release
ADR: Title / Status / Context / Decision / Consequences    (short, immutable, numbered)
```

- **Tolerant reader** + **expand/contract** (parallel change) allow evolving schemas and APIs without breaking consumers. Consumer-driven contract tests (Pact) catch breakage.
- **OpenTelemetry** is the vendor-neutral standard: `System.Diagnostics.Activity` = span, `Meter` = metrics, `ILogger` = logs; export via OTLP.
- **Feature flags**: `Microsoft.FeatureManagement` with Azure App Configuration. Release toggles are temporary - remove them.

### 16.1 [Easy] How do you version a REST API in ASP.NET Core?
**Answer**
Use the Asp.Versioning packages to version by URL segment, query string or header, and mark old versions deprecated. Prefer additive, backward-compatible changes and only cut a new version for breaking changes.

**Explanation**
URL versioning is the most discoverable and cache-friendly; header versioning keeps URLs clean. Report supported/deprecated versions in response headers so consumers can plan.

**Code**
```csharp
// Asp.Versioning.Http (minimal APIs)
builder.Services.AddApiVersioning(o =>
{
    o.DefaultApiVersion = new ApiVersion(1, 0);
    o.AssumeDefaultVersionWhenUnspecified = true;
    o.ReportApiVersions = true;
    o.ApiVersionReader = new UrlSegmentApiVersionReader();
});
var versionSet = app.NewApiVersionSet().HasApiVersion(new ApiVersion(1, 0)).HasApiVersion(new ApiVersion(2, 0)).Build();
app.MapGet("/api/v{version:apiVersion}/orders/{id:guid}", (Guid id) => Results.Ok(new { id }))
   .WithApiVersionSet(versionSet).MapToApiVersion(1, 0);
```

### 16.2 [Follow-up] What counts as a breaking change, and how do message contracts evolve safely?
**Answer**
Breaking: removing/renaming fields, changing types or meaning, making optional fields required, changing status codes or enum semantics. Safe: adding optional fields, new endpoints, new enum values only if consumers tolerate unknowns. For messages, deploy consumers first (tolerant of new fields), then producers.

**Explanation**
Expand/contract: add the new field alongside the old (expand), migrate consumers, remove the old (contract). System.Text.Json ignores unknown properties by default, which supports tolerant reading; avoid `JsonUnmappedMemberHandling.Disallow` for external contracts. Protobuf: never reuse field numbers.

**Code**
```csharp
// v1 of the event
public sealed record CustomerRegistered(Guid Id, string FullName);
// v1.1: additive and optional -> old consumers unaffected
public sealed record CustomerRegisteredV1_1(Guid Id, string FullName, string? PreferredLanguage = null);

var json = """{ "id": "7f6f...", "fullName": "Asha", "newUnknownField": 1 }""";
var opts = new JsonSerializerOptions(JsonSerializerDefaults.Web);   // unknown members ignored by default
```

### 16.3 [Medium] Set up observability across services with OpenTelemetry.
**Answer**
Register OpenTelemetry tracing and metrics with ASP.NET Core, HttpClient and EF/SQL instrumentation, and export via OTLP to a backend (Jaeger, Application Insights, Grafana). Trace context (W3C `traceparent`) propagates automatically over HttpClient; for message brokers, instrumentation or manual propagation in headers is needed.

**Explanation**
Use structured logging with the trace id included so logs correlate with traces. Define RED metrics (rate, errors, duration) per endpoint and USE for resources. Add custom `ActivitySource` spans for business operations. Beware high-cardinality metric labels (user id) that explode costs.

**Code**
```csharp
// Packages: OpenTelemetry.Extensions.Hosting, OpenTelemetry.Instrumentation.AspNetCore, .Http, .SqlClient, OpenTelemetry.Exporter.OpenTelemetryProtocol
builder.Services.AddOpenTelemetry()
    .ConfigureResource(r => r.AddService("orders-api"))
    .WithTracing(t => t
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddSqlClientInstrumentation()
        .AddSource(Telemetry.SourceName)
        .AddOtlpExporter())
    .WithMetrics(m => m
        .AddAspNetCoreInstrumentation()
        .AddMeter(Telemetry.SourceName)
        .AddOtlpExporter());

public static class Telemetry
{
    public const string SourceName = "Shop.Orders";
    public static readonly ActivitySource Source = new(SourceName);
    public static readonly Meter Meter = new(SourceName);
    public static readonly Counter<long> OrdersPlaced = Meter.CreateCounter<long>("orders_placed");
}

using var activity = Telemetry.Source.StartActivity("PlaceOrder");
activity?.SetTag("order.id", orderId);
Telemetry.OrdersPlaced.Add(1);
```

### 16.4 [Hard] Feature flags with blue/green and canary: how do they differ and combine?
**Answer**
Blue/green runs two full environments and switches traffic at once, giving instant rollback but requiring double capacity and careful DB compatibility. Canary shifts a small percentage of traffic to the new version and increases gradually while watching metrics. Feature flags decouple deploy from release, letting code ship dark and be enabled per user/percentage without redeploying.

**Explanation**
Database changes must be backward compatible with both versions (expand/contract) so rollback is possible. Flags are temporary technical debt; clean up. Flags can be evaluated per tenant/user group using filters in `Microsoft.FeatureManagement`.

**Code**
```csharp
// Microsoft.FeatureManagement.AspNetCore
builder.Services.AddFeatureManagement(builder.Configuration.GetSection("FeatureManagement"));

app.MapGet("/checkout", async (IFeatureManager fm) =>
    await fm.IsEnabledAsync("NewCheckout") ? Results.Ok("v2 checkout") : Results.Ok("v1 checkout"));
```
```json
{
  "FeatureManagement": {
    "NewCheckout": {
      "EnabledFor": [ { "Name": "Microsoft.Percentage", "Parameters": { "Value": 10 } } ]
    }
  }
}
```

**Gotchas / follow-ups**
- Percentage filter is per evaluation unless you use `Microsoft.Targeting` for consistent per-user assignment.
- Migrations that drop columns break the old version during blue/green/canary overlap.

### 16.5 [Tricky] How do you write and use an Architecture Decision Record, and what makes a decision hard to reverse?
**Answer**
An ADR is a short, numbered document capturing one significant decision with Status, Context, Decision and Consequences (including alternatives rejected), stored in the repo next to the code. It is immutable: if the decision changes, write a new ADR that supersedes the old. Decisions that are expensive to reverse (database engine, messaging backbone, service boundaries, public API contracts) deserve ADRs.

**Explanation**
ADRs prevent "why did we do this?" amnesia and re-litigation, and give new joiners context. Keep them lightweight (one page), include trade-offs and review conditions ("revisit if load exceeds X"). Pair with fitness functions (architecture tests) to keep the decision enforced.

**Code**
```csharp
// A decision made executable: ADR-0007 "Domain must not reference infrastructure"
[Fact]
public void ADR_0007_Domain_has_no_infrastructure_dependency()
{
    var result = Types.InAssembly(typeof(Order).Assembly)
        .ShouldNot().HaveDependencyOn("Microsoft.EntityFrameworkCore")
        .GetResult();
    Assert.True(result.IsSuccessful, string.Join(", ", result.FailingTypeNames ?? []));
}
```
```
docs/adr/0007-domain-independent-of-infrastructure.md
Status: Accepted (supersedes 0003)
Context: ...   Decision: ...   Consequences: + testable domain, - extra mapping code
```

**Gotchas / follow-ups**
- An ADR without alternatives and consequences is just an announcement.
- Don't edit accepted ADRs; supersede them.

**Interview traps**
- Bumping the API version for every additive change.
- Dropping a DB column in the same release as the code that stops using it.
- Treating feature flags as permanent configuration.
- Logging without correlation/trace ids across services.
- High-cardinality labels in metrics.

**Cheat sheet**
- Additive = compatible; rename/remove/retype = breaking.
- Expand/contract for schema and contracts; consumers first.
- Observability = logs + metrics + traces via OpenTelemetry, W3C traceparent.
- Blue/green = swap; canary = gradual; flags = decouple release.
- ADR = context, decision, consequences; supersede, don't edit.

<!-- end of file -->
