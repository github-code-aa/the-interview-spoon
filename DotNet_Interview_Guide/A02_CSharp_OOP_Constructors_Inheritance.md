# A02 – C# OOP, Constructors & Inheritance

**How to use:** Read the *Study notes* of a topic first, then climb the ladder (Easy → Tricky). Cover the Answer, say it aloud, then check Explanation/Code.
**Tip:** Run every Code block in a console project (.NET 8, C# 12) and confirm the `// Output:` comments yourself.

## Table of Contents
1. Topic 1: Instance Constructors (parameterless, private, copy, chaining)
2. Topic 2: Static Constructors in Depth
3. Topic 3: Object Initialization Order
4. Topic 4: The Four OOP Pillars
5. Topic 5: Inheritance & `base`
6. Topic 6: virtual / override / new / sealed
7. Topic 7: Abstract Class vs Interface (DIM, static abstract)
8. Topic 8: Polymorphism & Dynamic Dispatch
9. Topic 9: Access Modifiers
10. Topic 10: Partial Classes & Methods
11. Topic 11: Explicit Interface Implementation
12. Topic 12: Composition vs Inheritance
13. Topic 13: SOLID with C# Examples
14. Topic 14: Indexers & Operator Overloading
15. Topic 15: Object Initializers, init-only, Primary Constructors
16. Topic 16: Static Classes & Extension Methods
17. Topic 17: Nested Classes
18. Topic 18: Common Design Patterns

---

## Topic 1: Instance Constructors (parameterless, private, copy, chaining)

**Study notes**
- A constructor has the class name, no return type, and runs once per `new`. It initializes the object after the CLR has zeroed the memory.
- If you declare **no** constructor, the compiler generates a public parameterless one (protected for abstract classes). Declare any constructor and the default disappears.
- Constructor chaining: `: this(...)` calls another ctor in the same class; `: base(...)` calls a base ctor. Without an explicit initializer the compiler inserts `: base()`.
- Private constructor: blocks outside instantiation (Singleton, static factory methods, static-only helpers).
- C# has no built-in copy constructor; by convention you write `Foo(Foo other)`. Records get a compiler-generated protected copy ctor (used by `with`).
- Structs: C# 10+ allows explicit parameterless ctors; all fields must be assigned before ctor returns.

```
new Derived(1)
  └─ Derived(int) : base(x)  → Base(int) : base() → object()
        runs: Base body first, then Derived body
```

### 1.1 [Easy] What is a constructor and when does the compiler give you a default one?
**Answer**
A constructor is a special member that initializes a new instance. If a class declares no constructors, the compiler supplies a public parameterless one; as soon as you declare any constructor, that default is no longer generated.

**Explanation**
This is why adding a parameterized ctor to a class can break code (or serializers/ORMs) that relied on `new Foo()`. Abstract classes get a `protected` default ctor. Static classes get none.

**Code**
```csharp
public class A { }                       // compiler adds public A() { }
public class B { public B(int x) { } }   // no default ctor any more

var a = new A();
// var b = new B();   // compile error CS7036
var b = new B(1);
```

### 1.2 [Follow-up] How does constructor chaining with `this(...)` work and why use it?
**Answer**
`this(...)` forwards to another constructor in the same class, so shared initialization lives in one place. The chained ctor body runs first, then the body of the calling ctor.

**Explanation**
It avoids duplicated logic and keeps invariants centralized. You cannot chain in a cycle (compile error), and a ctor cannot use both `this(...)` and `base(...)`.

**Code**
```csharp
public class Person
{
    public string Name { get; }
    public int Age { get; }

    public Person() : this("Unknown") { Console.WriteLine("Person()"); }
    public Person(string name) : this(name, 0) { Console.WriteLine("Person(name)"); }
    public Person(string name, int age) { Name = name; Age = age; Console.WriteLine("Person(name, age)"); }
}

new Person();
// Output:
// Person(name, age)
// Person(name)
// Person()
```

### 1.3 [Medium] When would you use a private constructor? Give two uses.
**Answer**
To prevent outside instantiation: Singleton (one instance exposed through a static property) and factory-method/named-constructor designs where validation happens in a static `Create`. Also used for static-only holders (though a static class is better).

**Explanation**
A private ctor also stops inheritance from outside the class (derived classes cannot call it) — except nested classes, which can. A private ctor can be used by static members of the same class.

**Code**
```csharp
public sealed class Money
{
    public decimal Amount { get; }
    public string Currency { get; }
    private Money(decimal amount, string currency) { Amount = amount; Currency = currency; }

    public static Money Create(decimal amount, string currency)
    {
        if (amount < 0) throw new ArgumentOutOfRangeException(nameof(amount));
        return new Money(amount, currency);
    }
}
```

### 1.4 [Medium] How do you implement a copy constructor in C#, and how does it differ from `MemberwiseClone` and record `with`?
**Answer**
Write a ctor that takes the same type and copies each field, deep-copying reference members as needed. `MemberwiseClone` makes a shallow copy bypassing constructors; a record's `with` calls the compiler-generated protected copy ctor and then applies the initializers.

**Explanation**
Shallow copy shares nested references. A hand-written copy ctor lets you control depth and run validation. `MemberwiseClone` does not run any constructor, so invariants set there are not re-run.

**Code**
```csharp
public class Order
{
    public int Id { get; set; }
    public List<string> Items { get; set; } = new();
    public Order() { }
    public Order(Order other)            // deep copy of the list
    {
        Id = other.Id;
        Items = new List<string>(other.Items);
    }
}

public record Point(int X, int Y);
var p = new Point(1, 2);
var q = p with { Y = 5 };   // Point(1, 5)
```

### 1.5 [Tricky] A base class has only a parameterized constructor. What happens with a derived class that declares no constructor?
**Answer**
Compile error CS7036: the implicit derived default ctor tries to call `base()` which doesn't exist. The derived class must declare a ctor that calls `: base(args)` explicitly.

**Explanation**
Every constructor calls a base constructor; if you omit the initializer the compiler inserts `base()`. The same applies when the base's parameterless ctor is private or inaccessible.

**Code**
```csharp
public class Base { public Base(int x) { } }

// public class Bad : Base { }                 // error CS7036
public class Good : Base { public Good() : base(42) { } }
```

**Gotchas / follow-ups**
- Primary ctor on derived: `class D(int x) : Base(x)`.
- Struct ctor must assign all fields (or call `this()`), before C# 11 auto-default made this lenient.
- ctor can't be `virtual`; calling virtual methods inside is dangerous (see Topic 3).

**Interview traps**
- "Default ctor always exists" – false once you add any ctor.
- `MemberwiseClone` doesn't call a ctor and is shallow.
- Chained `this(...)` runs *before* the body of the caller.
- Constructors are not inherited.

**Cheat sheet**
- No ctor declared → compiler adds public parameterless.
- `: this()` same class, `: base()` parent; implicit `base()` otherwise.
- Private ctor → singleton / factory.
- Copy ctor is a convention; records auto-generate one.
- Constructors can't be virtual or inherited.

---

## Topic 2: Static Constructors in Depth

**Study notes**
- `static Foo() { }` – no access modifier, no parameters, one per type. Initializes static state / runs one-time setup.
- Called automatically by the CLR **at most once per type per AppDomain/ALC**, before the first instance is created or any static member is accessed (precise timing depends on `beforefieldinit`).
- **beforefieldinit**: a type with static field initializers but **no explicit static ctor** is marked `beforefieldinit`; the runtime may run the initializer any time before first static *field* access (often lazily at JIT time or earlier). A type **with** an explicit static ctor is *precise*: it runs exactly at first access to the type (instance creation or any static member).
- Thread safety: the CLR guarantees the type initializer runs once, with a lock; other threads block until done. (Beware deadlocks with cyclic static init across threads.)
- If it throws, the type is poisoned: a `TypeInitializationException` is thrown on first access and again (cached) on every later access.
- Order: static field initializers (textual order) → static ctor body. Derived static ctor does **not** trigger base static ctor first by rule; base static init runs when the base type is first touched (which creating a derived instance does, via the base instance ctor).

```
first use of Derived
  static field initializers (Derived, textual order)
  static Derived() body
  -- then instance creation:
  instance field inits (Derived) → Base static init (if not yet run) → Base ctor → Derived ctor body
```

### 2.1 [Easy] What is a static constructor and when is it called?
**Answer**
A static constructor initializes static data for a type; it has no parameters or access modifier and cannot be called directly. The runtime invokes it once, automatically, before the first instance is created or any static member is referenced.

**Explanation**
Use it when static initialization needs more than a field initializer (e.g., loading config, complex setup). You can't control exactly when; only that it's before first use.

**Code**
```csharp
public class Config
{
    public static readonly string Env;
    static Config() { Env = "Prod"; Console.WriteLine("static ctor"); }
    public static void Hello() => Console.WriteLine("Hello");
}

Console.WriteLine("start");
Config.Hello();
Config.Hello();
// Output:
// start
// static ctor
// Hello
// Hello
```

### 2.2 [Follow-up] Is a static constructor thread-safe? Can it run twice?
**Answer**
The CLR runs it exactly once per type (per load context) and blocks other threads that need the type until it finishes. So it's thread-safe for initialization, but code inside shouldn't wait on other threads that need the same type.

**Explanation**
This is why `static readonly Lazy`-less singletons via static field initializers are thread-safe. Deadlock risk: thread A is initializing type X and waits on thread B, which needs type X.

**Code**
```csharp
public sealed class Singleton
{
    public static readonly Singleton Instance = new();   // CLR guarantees once
    private Singleton() { }
}
```

### 2.3 [Medium] What is `beforefieldinit` and how does it change timing?
**Answer**
It's an IL flag the C# compiler puts on types that have static field initializers but no explicit static constructor. It lets the runtime run initialization earlier/lazier at its convenience, any time before first static field access. Adding an empty `static Foo() {}` removes the flag and makes timing precise.

**Explanation**
With `beforefieldinit` you can observe initialization happening earlier than your first call (JIT-dependent, especially in optimized builds). Precise semantics matter when initializers have side effects (logging, ordering). That's why the "Jon Skeet singleton" adds an empty static ctor.

**Code**
```csharp
public class Lazyish   // beforefieldinit: initializer timing unspecified
{
    public static int V = Init("Lazyish");
    public static void M() { }
    static int Init(string n) { Console.WriteLine("init " + n); return 1; }
}

public class Precise   // explicit static ctor: runs exactly at first access
{
    public static int V = Init("Precise");
    static Precise() { }
    public static void M() { }
    static int Init(string n) { Console.WriteLine("init " + n); return 1; }
}
// Precise.M() triggers "init Precise" right then.
// Lazyish.M() might not trigger init at all until Lazyish.V is read.
```

### 2.4 [Hard] What happens if a static constructor throws?
**Answer**
The exception is wrapped in `TypeInitializationException` (original in `InnerException`). The type is unusable for the rest of the process: every later access rethrows `TypeInitializationException` without rerunning the ctor.

**Explanation**
Static field initializer exceptions behave identically. This is a classic production bug: a missing config in a static ctor makes the whole type permanently broken until restart. Note the first throw also wraps; later throws carry the same cached failure.

**Code**
```csharp
public class Bad
{
    public static readonly int X = int.Parse("abc");   // FormatException
    public static void Use() { }
}

for (int i = 0; i < 2; i++)
{
    try { Bad.Use(); }
    catch (TypeInitializationException ex)
    {
        Console.WriteLine($"{ex.TypeName} -> {ex.InnerException!.GetType().Name}");
    }
}
// Output:
// Bad -> FormatException
// Bad -> FormatException
```

**Gotchas / follow-ups**
- `beforefieldinit` types may throw at an unexpected place.
- Don't catch-and-retry; redesign with `Lazy<T>` with retry semantics if recovery is needed.

### 2.5 [Tricky] Predict the output: static initializers, static ctors and inheritance order.
**Answer**
Static field initializers run before the static ctor body in the same type; the derived type's static init runs first when you touch it, while the base's static init runs when base is first needed — which, for creating a derived instance, is before the derived instance ctor body (but after derived instance field initializers).

**Explanation**
Two distinct sequences: static (once per type) and instance (per `new`). Instance field initializers of derived run *before* the base ctor; base ctor body runs before derived ctor body.

**Code**
```csharp
class Base
{
    static int sb = Log("Base static field");
    int ib = Log("Base instance field");
    static Base() => Log("Base static ctor");
    public Base() => Log("Base ctor");
    public static int Log(string s) { Console.WriteLine(s); return 0; }
}
class Derived : Base
{
    static int sd = Log("Derived static field");
    int id = Log("Derived instance field");
    static Derived() => Log("Derived static ctor");
    public Derived() => Log("Derived ctor");
}

new Derived();
// Output:
// Derived static field
// Derived static ctor
// Derived instance field
// Base static field
// Base static ctor
// Base instance field
// Base ctor
// Derived ctor
```

**Gotchas / follow-ups**
- Static init timing for base may vary slightly with `beforefieldinit` (no explicit static ctor); here both have explicit ones so it's precise.
- Static fields declared textually later are still default (0/null) while earlier initializers run.

**Interview traps**
- "Static ctor runs when the app starts" – no, on first use.
- Static ctor can't take params or be called manually; no access modifier.
- Failure is permanent (`TypeInitializationException`), not retried.
- Initializers run in textual order; a field used before its declaration sees default.
- Generic types: one static ctor run **per closed generic type** (`Foo<int>` and `Foo<string>` each init).

**Cheat sheet**
- Static fields init → static ctor body.
- Once per type, thread-safe, CLR-locked.
- Explicit static ctor ⇒ precise; none ⇒ `beforefieldinit` (lazy-ish).
- Exception ⇒ `TypeInitializationException`, type poisoned.
- Derived instance fields → base ctor → derived ctor body.

---

## Topic 3: Object Initialization Order

**Study notes**
Order for `new Derived()` (instance side):
1. Memory allocated, all fields zeroed.
2. Derived field initializers (textual order).
3. Derived ctor's `: base(...)` call → Base field initializers → Base ctor body (recursively up to `object`).
4. Derived ctor body.

C# runs field initializers **derived-first** (unlike C++/Java, where base fields init first). This lets derived field initializers run before the base ctor, but they can't reference `this`/instance members (error CS0236), except via ctor args.

```
Derived field inits → Base field inits → Base ctor body → Derived ctor body
```
A virtual method called from the Base ctor dispatches to the **most-derived override** even though Derived's ctor body hasn't run (derived fields set in the ctor body are still default; initializer-assigned ones are already set).

### 3.1 [Easy] In what order do constructors run in an inheritance chain?
**Answer**
Base class constructor bodies run first (from the root `object` down), then derived constructor bodies. Each derived ctor implicitly calls the base ctor before its own body.

**Explanation**
That ensures the base part of the object is ready before derived code uses it.

**Code**
```csharp
class A { public A() => Console.WriteLine("A"); }
class B : A { public B() => Console.WriteLine("B"); }
class C : B { public C() => Console.WriteLine("C"); }
new C();
// Output:
// A
// B
// C
```

### 3.2 [Follow-up] Where do field initializers run relative to the base constructor?
**Answer**
In C#, field initializers of the derived class run before the base constructor is invoked (derived first), then the base's initializers and ctor, then the derived ctor body.

**Explanation**
Instance field initializers are effectively copied to the start of each ctor that doesn't chain `this(...)`, before the base call. That's why they can't use other instance members.

**Code**
```csharp
class P { public P() => Console.WriteLine("P ctor"); }
class Q : P
{
    int f = Log("Q field");
    public Q() => Console.WriteLine("Q ctor");
    static int Log(string s) { Console.WriteLine(s); return 1; }
}
new Q();
// Output:
// Q field
// P ctor
// Q ctor
```

### 3.3 [Medium] What happens when a base constructor calls a virtual method?
**Answer**
The call dispatches to the derived override, even though the derived ctor hasn't run yet. The override sees fields assigned in the derived ctor body as default values.

**Explanation**
Unlike C++, the object's runtime type is already `Derived` during base construction. It's a design smell; avoid virtual calls in ctors, or document and keep overrides free of state dependence.

**Code**
```csharp
class Base
{
    public Base() { Console.WriteLine("Base ctor"); Show(); }
    public virtual void Show() => Console.WriteLine("Base.Show");
}
class Derived : Base
{
    string _name = "init";   // field initializer: already set
    string? _set;
    public Derived() { _set = "ctor"; Console.WriteLine("Derived ctor"); }
    public override void Show() => Console.WriteLine($"Derived.Show name={_name} set={_set ?? "null"}");
}
new Derived();
// Output:
// Base ctor
// Derived.Show name=init set=null
// Derived ctor
```

### 3.4 [Hard] How do `this(...)` chaining and field initializers interact?
**Answer**
Field initializers are emitted only in constructors that call `base(...)` (explicitly or implicitly), not in ones that chain with `this(...)`. So they run once, in the "terminal" ctor.

**Explanation**
Otherwise they'd execute multiple times in a chain. With primary constructors, initializers run as part of the primary ctor.

**Code**
```csharp
class T
{
    int x = Log("init");
    public T() : this(1) => Log("T()");
    public T(int v) => Log("T(int)");
    static int Log(string s) { Console.WriteLine(s); return 0; }
}
new T();
// Output:
// init
// T(int)
// T()
```

### 3.5 [Tricky] An exception is thrown in a constructor — what's the state of the object, and what about `readonly` and disposal?
**Answer**
`new` never returns the reference, so the caller gets nothing; the partially built object is garbage. Its finalizer (if any) still runs, but `Dispose` is never called because the caller has no reference.

**Explanation**
Therefore a ctor that acquires an unmanaged resource then throws leaks it unless it cleans up in a try/catch inside the ctor. Prefer `SafeHandle`.

**Code**
```csharp
class Res : IDisposable
{
    Stream _s;
    public Res(string path)
    {
        _s = File.OpenRead(path);
        try { Validate(); }
        catch { _s.Dispose(); throw; }   // clean up before the ctor fails
    }
    void Validate() => throw new InvalidOperationException();
    public void Dispose() => _s.Dispose();
}
```

**Gotchas / follow-ups**
- Derived-first field init is a C#-specific ordering; Java/C++ differ.
- In a struct, `new S()` always yields a value; no exception leak issue of the same kind.

**Interview traps**
- Saying "base field initializers run first" – wrong in C#.
- Virtual call in ctor sees uninitialized derived state.
- `this(...)` chain: initializers run once.
- Ctor exception ⇒ no `Dispose` call.

**Cheat sheet**
- Derived inits → base inits → base ctor → derived ctor.
- Virtual calls in ctor hit the most-derived override.
- Initializers can't reference `this` members.
- Partial object after throw is unreachable.

---

## Topic 4: The Four OOP Pillars

**Study notes**
| Pillar | Meaning | C# tools |
|---|---|---|
| Encapsulation | Hide state, expose behavior with controlled access | access modifiers, properties, `init`, `private set` |
| Abstraction | Expose *what*, hide *how* | interfaces, abstract classes |
| Inheritance | Reuse/specialize via is-a | `:` base class, `base`, `virtual/override` |
| Polymorphism | One interface, many behaviors | overriding, interfaces, generics (parametric), overloads (compile-time) |

Polymorphism kinds: *subtype* (runtime, virtual dispatch), *ad hoc* (overloading, compile-time), *parametric* (generics).

### 4.1 [Easy] Name and define the four pillars of OOP.
**Answer**
Encapsulation bundles data with the code that guards it; abstraction exposes only essential behavior; inheritance lets a type reuse and extend another; polymorphism lets different types respond to the same call differently.

**Explanation**
Give one C# example each: private fields + properties; interface `IRepository`; `class Dog : Animal`; `virtual Speak()` overridden.

**Code**
```csharp
public abstract class Animal                 // abstraction
{
    private string _name;                    // encapsulation
    protected Animal(string name) => _name = name;
    public string Name => _name;
    public abstract string Speak();          // polymorphism hook
}
public class Dog : Animal                    // inheritance
{
    public Dog(string n) : base(n) { }
    public override string Speak() => "Woof";
}
```

### 4.2 [Follow-up] How is encapsulation different from abstraction?
**Answer**
Encapsulation is about protecting internal state and enforcing invariants (implementation level). Abstraction is about presenting a simplified contract that hides complexity (design level). Encapsulation is *how* you hide; abstraction is *what you choose to show*.

**Explanation**
A `BankAccount` that exposes `Deposit()` but keeps `_balance` private is encapsulation. The `IAccount` interface callers depend on is abstraction.

**Code**
```csharp
public class BankAccount
{
    private decimal _balance;                      // hidden state
    public decimal Balance => _balance;
    public void Deposit(decimal amt)
    {
        if (amt <= 0) throw new ArgumentException("Positive only");
        _balance += amt;                           // invariant enforced
    }
}
```

### 4.3 [Medium] What are the different kinds of polymorphism in C#?
**Answer**
Compile-time (overloading, operators, generics) and runtime (virtual methods / interface dispatch). Runtime polymorphism is resolved via the object's actual type; compile-time via the declared argument types.

**Explanation**
Overload resolution picks a method from static types — a common trap with `object` params.

**Code**
```csharp
static class P
{
    public static void Print(object o) => Console.WriteLine("object");
    public static void Print(string s) => Console.WriteLine("string");
}

object x = "hi";
P.Print(x);          // Output: object   (static type decides)
P.Print("hi");       // Output: string
P.Print((dynamic)x); // Output: string  (dynamic binds at runtime)
```

### 4.4 [Hard] Why is "inheritance breaks encapsulation" a known criticism?
**Answer**
A subclass depends on base-class implementation details (fragile base class problem): changing a base's internal call pattern can silently break derived classes. Prefer composition or sealed/designed-for-inheritance classes.

**Explanation**
Classic example: overriding both `Add` and `AddAll` in a counting collection double counts when `AddAll` internally calls `Add`.

**Code**
```csharp
public class Bag
{
    protected readonly List<int> Items = new();
    public virtual void Add(int x) => Items.Add(x);
    public virtual void AddAll(IEnumerable<int> xs) { foreach (var x in xs) Add(x); }
}
public class CountingBag : Bag
{
    public int Count { get; private set; }
    public override void Add(int x) { Count++; base.Add(x); }
    public override void AddAll(IEnumerable<int> xs) { Count += xs.Count(); base.AddAll(xs); }
}
var b = new CountingBag(); b.AddAll(new[] { 1, 2, 3 });
Console.WriteLine(b.Count);   // Output: 6 (expected 3)
```

**Gotchas / follow-ups**
- Solve with composition (wrap a `Bag`) or don't override `AddAll`.

**Interview traps**
- Confusing abstraction with abstract classes.
- Overload resolution is static; override is dynamic.
- Getters/setters alone aren't encapsulation if they expose mutable internals (returning a `List<T>` field).

**Cheat sheet**
- Encapsulation = protect state; Abstraction = hide complexity.
- Inheritance = is-a reuse; Polymorphism = same call, different behavior.
- Runtime poly: virtual/interface. Compile-time: overload/generics.
- Fragile base class → favor composition.

---

## Topic 5: Inheritance & `base`

**Study notes**
- Single class inheritance, multiple interface implementation. All types derive from `System.Object`.
- Structs cannot inherit classes (only implement interfaces); `sealed` stops inheritance; static classes can't be inherited.
- `base.Member` calls the base implementation (non-virtual call, statically bound). `base(...)` in ctors.
- Derived inherits accessible members (public/protected/internal-in-assembly), **not** constructors or private members (private members exist in the object but are inaccessible).
- Conversions: upcast implicit; downcast needs cast, `as`, or `is`.

```
object
 └─ Animal
     ├─ Dog
     └─ Cat
Animal a = new Dog();   // static type Animal, runtime type Dog
```

### 5.1 [Easy] How do you declare inheritance, and what can't a derived class inherit?
**Answer**
Use `class Derived : Base`. It inherits accessible members but not constructors, and it can't access private members of the base.

**Explanation**
Private fields still exist in the object's memory layout; they are just not accessible by name.

**Code**
```csharp
class Vehicle { public int Wheels = 4; private int _secret = 1; protected int Speed; }
class Car : Vehicle
{
    public void Go() { Speed = 10; Console.WriteLine(Wheels); }   // _secret not accessible
}
```

### 5.2 [Follow-up] What does the `base` keyword do?
**Answer**
It accesses members of the immediate base class from the derived class — to call an overridden method's base version or to invoke a specific base constructor.

**Explanation**
`base.M()` is a non-virtual call, so it doesn't recurse into the override. It can only reference the immediate base (no `base.base`).

**Code**
```csharp
class Logger { public virtual string Format(string m) => $"[LOG] {m}"; }
class TimeLogger : Logger
{
    public override string Format(string m) => $"{DateTime.UtcNow:HH:mm} " + base.Format(m);
}
```

### 5.3 [Medium] What do `is`, `as`, and a cast do for downcasting? Differences?
**Answer**
A cast throws `InvalidCastException` on failure; `as` returns null; `is` tests (and with a pattern, declares a typed variable). Use `is Type t` for safe check-and-use.

**Explanation**
`as` works only with reference/nullable types. Pattern matching avoids a double check.

**Code**
```csharp
Animal a = new Dog("Rex");
if (a is Dog d) Console.WriteLine(d.Name);
Cat? c = a as Cat;            // null
// Cat c2 = (Cat)a;           // throws InvalidCastException
Console.WriteLine(c is null); // Output: True
```

### 5.4 [Hard] Can a derived class change the accessibility or return type of an inherited member?
**Answer**
An override must keep the same accessibility (except `protected internal` across assemblies → `protected`). Return types must match, except covariant returns (C# 9+) allow a more-derived return type on an override.

**Explanation**
Covariant returns are only for reference-type return types on overridden methods/read-only properties, in .NET 5+.

**Code**
```csharp
class Animal { public virtual Animal Create() => new Animal(); }
class Dog : Animal { public override Dog Create() => new Dog(); }   // covariant return
```

### 5.5 [Tricky] Can you inherit from a `struct`, `string`, or `Nullable<T>`? What does a sealed class buy you?
**Answer**
No: structs can't be base types, and `string` is sealed. `sealed` prevents inheritance, enables JIT devirtualization/faster casts, and protects invariants (immutable types).

**Explanation**
Many BCL types (`string`, `Uri`) are sealed. Records are not sealed by default; `sealed record` is common.

**Code**
```csharp
public sealed class Token { public string Value { get; } public Token(string v) => Value = v; }
// class Evil : Token { }  // error CS0509
```

**Gotchas / follow-ups**
- Interfaces can inherit multiple interfaces; classes only one class.
- Diamond problem avoided for classes; with default interface methods, ambiguity must be resolved explicitly.

**Interview traps**
- "Private members aren't inherited" – they're in the object, just inaccessible.
- `base` call is non-virtual; no `base.base`.
- Constructors are never inherited.
- Overrides can't narrow/widen accessibility.

**Cheat sheet**
- One base class, many interfaces; everything → `object`.
- `base.M()` / `: base(args)`.
- Upcast implicit; downcast via `is`/`as`/cast.
- `sealed` stops inheritance; covariant returns C# 9.

---

## Topic 6: virtual / override / new / sealed

**Study notes**
| Keyword | Effect | Binding |
|---|---|---|
| `virtual` | Allows override | runtime (vtable) |
| `override` | Replaces base virtual impl | runtime |
| `new` | Hides base member (new slot) | compile-time (static type of reference) |
| `sealed override` | Final override, further overriding blocked | runtime |
| `abstract` | No body; must override | runtime |

Without `override` or `new`, a same-name member hides the base one and the compiler warns CS0108. Hiding creates a *new* vtable slot; calls via a base-typed reference reach the base method.

### 6.1 [Easy] What's the difference between `virtual` and `override`?
**Answer**
`virtual` marks a base method as overridable; `override` in a derived class provides a replacement. At runtime the actual object type's override is invoked.

**Explanation**
Methods are non-virtual by default in C# (unlike Java), so you opt in.

**Code**
```csharp
class Shape { public virtual double Area() => 0; }
class Circle : Shape
{
    public double R;
    public override double Area() => Math.PI * R * R;
}
Shape s = new Circle { R = 1 };
Console.WriteLine(s.Area());   // Output: 3.141592653589793
```

### 6.2 [Follow-up] What does the `new` modifier do on a method?
**Answer**
It hides an inherited member: the derived version is a separate method. Which one runs depends on the *compile-time type* of the variable, not the object.

**Explanation**
Use it when you can't make the base virtual (third-party base) and want to suppress warning CS0108. It's usually a design smell.

**Code**
```csharp
class A { public void Hi() => Console.WriteLine("A.Hi"); }
class B : A { public new void Hi() => Console.WriteLine("B.Hi"); }
A a = new B();
a.Hi();          // Output: A.Hi
((B)a).Hi();     // Output: B.Hi
```

### 6.3 [Medium] Predict the output: override vs new mixed.
**Answer**
Override resolves by runtime type; `new` resolves by variable type. So `Base b = new Derived()` calls the override of `Virt` but the base version of the hidden `Hid`.

**Explanation**
Walk through each of the three variable types explicitly.

**Code**
```csharp
class Base
{
    public virtual void Virt() => Console.WriteLine("Base.Virt");
    public void Hid() => Console.WriteLine("Base.Hid");
}
class Derived : Base
{
    public override void Virt() => Console.WriteLine("Derived.Virt");
    public new void Hid() => Console.WriteLine("Derived.Hid");
}
Base b = new Derived();
Derived d = new Derived();
b.Virt(); b.Hid(); d.Virt(); d.Hid();
// Output:
// Derived.Virt
// Base.Hid
// Derived.Virt
// Derived.Hid
```

### 6.4 [Hard] What does `sealed override` do and what happens in a three-level chain with `new virtual`?
**Answer**
`sealed override` ends the override chain: further subclasses can't override that method. `new virtual` starts a *fresh* virtual chain; calls through the original base type still reach the old chain.

**Explanation**
Each virtual slot is independent; hiding with `new virtual` doesn't override the base slot.

**Code**
```csharp
class A { public virtual void M() => Console.WriteLine("A.M"); }
class B : A { public sealed override void M() => Console.WriteLine("B.M"); }
class C : B { public new virtual void M() => Console.WriteLine("C.M"); }
class D : C { public override void M() => Console.WriteLine("D.M"); }

A a = new D(); C c = new D();
a.M();   // Output: B.M   (A's chain: sealed at B)
c.M();   // Output: D.M   (C's new chain)
```

### 6.5 [Tricky] Which overload runs when a derived class adds a non-override overload alongside an override?
**Answer**
Overload resolution prefers methods declared in the *most derived* class first if applicable, even over a better-matching base overload — and overrides don't count as declared there.

**Explanation**
C# rule: candidate methods that are overrides are excluded from the derived class set; the search starts at the most-derived type that has an applicable non-override method.

**Code**
```csharp
class B { public virtual void F(int x) => Console.WriteLine("B.F(int)"); }
class D : B
{
    public override void F(int x) => Console.WriteLine("D.F(int)");
    public void F(object o) => Console.WriteLine("D.F(object)");
}
new D().F(5);
// Output: D.F(object)   (D's own non-override F(object) is applicable and wins; override F(int) ignored)
```

**Gotchas / follow-ups**
- Struct methods can't be `virtual` (they can override `ToString`, `Equals`).
- Properties and events can be virtual too; `abstract` implies virtual.
- Interface members are implicitly virtual/abstract.

**Interview traps**
- `new` ≠ `override`; behavior depends on reference type.
- Methods are non-virtual by default in C#.
- Can't override non-virtual, static, or sealed methods.
- Overload resolution subtleties with overrides in derived classes.

**Cheat sheet**
- virtual → override: runtime type decides.
- new: hides; static type decides.
- sealed override: stop the chain.
- abstract: must override, implicitly virtual.

---

## Topic 7: Abstract Class vs Interface (incl. default interface methods & static abstract members)

**Study notes**
| | Abstract class | Interface |
|---|---|---|
| Instantiate | No | No |
| State (fields) | Yes | No instance fields (static allowed) |
| Constructors | Yes | No |
| Inheritance | Single | Multiple |
| Members | any access, impl + abstract | public by default; since C# 8 default impl, static, private helpers |
| Use | shared base behavior/state, "is-a" | capability/contract, "can-do" |

- **Default interface methods (C# 8)**: interface can provide a body; implementers inherit it only **through the interface type** (`((IFoo)obj).M()`), not as a member of the class.
- **Static abstract/virtual members (C# 11)**: interfaces can declare `static abstract` members (operators, factories) enabling generic math (`INumber<T>`). Used via constrained generics `where T : IFoo`.

### 7.1 [Easy] Abstract class vs interface — when to use which?
**Answer**
Use an abstract class when related types share state or implementation; use an interface to define a capability that unrelated types can implement, since a class can implement many interfaces but inherit only one class.

**Explanation**
Interfaces suit DI/mocking and versioning-by-composition; abstract classes suit template-method patterns.

**Code**
```csharp
public abstract class Shape { public abstract double Area(); public void Print() => Console.WriteLine(Area()); }
public interface IDisposableThing { void Close(); }
```

### 7.2 [Follow-up] Can an abstract class have a constructor, fields, and non-abstract methods?
**Answer**
Yes to all. The constructor runs as part of derived construction (it's protected in effect), which is how it initializes shared state.

**Explanation**
Abstract members implicitly virtual; derived non-abstract classes must override all of them.

**Code**
```csharp
abstract class Repo
{
    protected readonly string ConnStr;
    protected Repo(string cs) => ConnStr = cs;
    public abstract void Save(object o);
}
class SqlRepo : Repo
{
    public SqlRepo(string cs) : base(cs) { }
    public override void Save(object o) => Console.WriteLine($"save to {ConnStr}");
}
```

### 7.3 [Medium] What are default interface methods and why were they added?
**Answer**
They let an interface supply a method body, so you can add members to a published interface without breaking existing implementers. The default is only callable through an interface-typed reference.

**Explanation**
Not a replacement for abstract classes—no state. Beware diamond ambiguity if two interfaces provide different defaults for the same member; the class must implement it explicitly.

**Code**
```csharp
interface ILogger
{
    void Log(string m);
    void Warn(string m) => Log("WARN: " + m);   // default
}
class ConsoleLogger : ILogger { public void Log(string m) => Console.WriteLine(m); }

var c = new ConsoleLogger();
// c.Warn("x");           // error: not a member of the class
((ILogger)c).Warn("x");   // Output: WARN: x
ILogger l = c; l.Warn("y"); // Output: WARN: y
```

### 7.4 [Hard] What are `static abstract` interface members and what do they enable?
**Answer**
They let an interface require implementers to provide static members/operators, callable generically as `T.Member`. This powers generic math (`INumber<T>`) and static factory contracts without reflection.

**Explanation**
The interface can't be used as a plain type for those members; you call through a type parameter constrained to it. In .NET 7+ BCL types like `int` implement `IAdditionOperators<,,>`.

**Code**
```csharp
public interface IParsable2<TSelf> where TSelf : IParsable2<TSelf>
{
    static abstract TSelf Create(string s);
}
public readonly record struct Id(int V) : IParsable2<Id>
{
    public static Id Create(string s) => new(int.Parse(s));
}
static T Make<T>(string s) where T : IParsable2<T> => T.Create(s);

Console.WriteLine(Make<Id>("7"));   // Output: Id { V = 7 }

static T Sum<T>(IEnumerable<T> xs) where T : System.Numerics.INumber<T>
{
    T acc = T.Zero;
    foreach (var x in xs) acc += x;
    return acc;
}
Console.WriteLine(Sum(new[] { 1, 2, 3 }));       // Output: 6
Console.WriteLine(Sum(new[] { 1.5, 2.5 }));      // Output: 4
```

### 7.5 [Tricky] Two interfaces both provide a default for `Hello()`. What happens when a class implements both?
**Answer**
If the class doesn't implement `Hello`, calling via either interface is fine, but the class itself has no unique most-specific implementation, so converting/calling through the class is a compile error; you must provide an implementation in the class (typically explicit).

**Explanation**
This is the "diamond" case. Resolution rule: the most specific override wins; with unrelated interfaces, there's none. Providing `void IA.Hello()` / `void IB.Hello()` or a class-level public method resolves it.

**Code**
```csharp
interface IA { void Hello() => Console.WriteLine("A"); }
interface IB { void Hello() => Console.WriteLine("B"); }
class C : IA, IB
{
    void IA.Hello() => Console.WriteLine("C via A");
    void IB.Hello() => Console.WriteLine("C via B");
}
((IA)new C()).Hello();   // Output: C via A
((IB)new C()).Hello();   // Output: C via B
```

**Gotchas / follow-ups**
- Without the explicit implementations in `C`, compile error CS8705 (no most specific implementation).
- Interfaces still can't have instance fields or constructors.

**Interview traps**
- "Interfaces can't have code" – outdated since C# 8.
- DIM not callable on the class reference.
- Adding an abstract member to an interface is a breaking change; adding a DIM isn't.
- `static abstract` members: call through generic `T`, not through the interface type.

**Cheat sheet**
- Abstract class: state + ctor + single inheritance.
- Interface: contract, multi-implement, DIM since C# 8.
- `static abstract` since C# 11 → generic math.
- Diamond DIM → implement explicitly.

---

## Topic 8: Polymorphism & Dynamic Dispatch

**Study notes**
- Every object has a method table pointer (MT). Each class's MT contains a **vtable**: slots for virtual methods. `callvirt` loads the MT from the object, indexes the slot, and jumps — O(1).
- Override replaces the slot's pointer in the derived MT; `new` adds a different slot.
- Interface calls use **virtual stub dispatch** (interface map lookup, cached with monomorphic/polymorphic caches); slightly slower than a vtable call. Tiered PGO in .NET 8 can *guard-devirtualize* hot call sites.
- Non-virtual instance methods are still emitted with `callvirt` by C# for the null check, but bound statically.

```
Animal a = new Dog();  a.Speak()
 a → [obj header | MT* → Dog MT {Speak→Dog.Speak, ToString→...}]
 callvirt: MT->slot[Speak] → Dog.Speak
```

### 8.1 [Easy] What is runtime polymorphism? Give an example.
**Answer**
It's invoking an overridden method through a base reference and having the derived implementation run, determined by the actual object type at runtime.

**Explanation**
Enables extensible code: a loop over `List<Shape>` calling `Area()` without type checks.

**Code**
```csharp
abstract class Shape { public abstract double Area(); }
class Sq(double s) : Shape { public override double Area() => s * s; }
class Circ(double r) : Shape { public override double Area() => Math.PI * r * r; }

foreach (Shape sh in new Shape[] { new Sq(2), new Circ(1) })
    Console.WriteLine(Math.Round(sh.Area(), 2));
// Output:
// 4
// 3.14
```

### 8.2 [Follow-up] How does the CLR implement virtual dispatch?
**Answer**
Each type has a method table with a vtable of method pointers; objects hold a pointer to their type's MT. A virtual call reads the MT from the object and calls the pointer at the method's fixed slot index.

**Explanation**
Slot indices are fixed along the inheritance chain, so derived classes keep base slots in the same positions and override by replacing the pointer. That's why a base-typed reference works.

**Code**
```csharp
object o = new List<int>();
Console.WriteLine(o.ToString());      // virtual: List<int>'s MT slot → Object.ToString inherited
Console.WriteLine(o.GetType().Name);  // List`1
```

### 8.3 [Medium] How are interface method calls dispatched, and is it slower?
**Answer**
Through virtual stub dispatch: the runtime looks up the implementing method in the type's interface map and caches the result at the call site. It's slightly slower than a vtable call, though the JIT can devirtualize/guard with PGO.

**Explanation**
Practical advice: don't micro-optimize prematurely; sealed classes and exact-type knowledge let the JIT inline.

**Code**
```csharp
interface IGreeter { string Greet(); }
sealed class En : IGreeter { public string Greet() => "Hi"; }

IGreeter g = new En();
Console.WriteLine(g.Greet());   // Output: Hi  (stub dispatch, may be devirtualized)
```

### 8.4 [Hard] Overloading vs overriding with `dynamic` and double dispatch — what does the visitor-style call resolve to?
**Answer**
Overloads bind on static types, so passing a base-typed reference picks the base overload. To get runtime-based selection use `dynamic`, pattern matching, or the Visitor pattern (double dispatch).

**Explanation**
`dynamic` resolves overloads at runtime using the actual argument type (slower, uses DLR). Visitor's `Accept(v) => v.Visit(this)` fixes `this` as the concrete type.

**Code**
```csharp
class Animal { }
class Dog : Animal { }
static class Dispatch   // must be real methods: dynamic calls can't bind to local functions
{
    public static string Who(Animal a) => "animal";
    public static string Who(Dog d) => "dog";
}

Animal x = new Dog();
Console.WriteLine(Dispatch.Who(x));            // Output: animal
Console.WriteLine(Dispatch.Who((dynamic)x));   // Output: dog
```

### 8.5 [Tricky] What does calling a virtual method on a `null` reference or on a struct via interface do?
**Answer**
A null reference throws `NullReferenceException` at the call (callvirt null check, even for non-virtual instance methods). Calling an interface method on a struct boxes it, so mutations affect the box copy.

**Explanation**
Boxing yields a copy; mutating via the interface changes the boxed instance only, a classic mutable-struct bug.

**Code**
```csharp
interface ICounter { void Inc(); }
struct Counter : ICounter { public int N; public void Inc() => N++; }

var c = new Counter();
ICounter i = c;        // boxed copy
i.Inc();
Console.WriteLine(c.N);            // Output: 0
Console.WriteLine(((Counter)i).N); // Output: 1
```

**Gotchas / follow-ups**
- Generic constraint `where T : struct, ICounter` calls without boxing (constrained call).

**Interview traps**
- Overloads aren't polymorphic at runtime.
- Interface call on struct causes boxing.
- Non-virtual method on null instance still throws (callvirt).
- `GetType()` returns runtime type, `typeof` compile-time.

**Cheat sheet**
- vtable slot per virtual method; override swaps pointer.
- Interface dispatch → stub with cache.
- Overload = compile-time; override = runtime.
- `dynamic` / Visitor give double dispatch.

---

## Topic 9: Access Modifiers

**Study notes**
| Modifier | Visible to |
|---|---|
| `public` | everywhere |
| `private` | same type (default for members) |
| `protected` | same type + derived types |
| `internal` | same assembly (default for top-level types) |
| `protected internal` | same assembly **OR** derived types (union) |
| `private protected` (C# 7.2) | derived types **in the same assembly** (intersection) |
| `file` (C# 11) | same source file only (types) |

Notes: `protected` access via an instance requires the instance be of the derived type (or deeper). Nested types default to `private`. Interface members default to `public`. `InternalsVisibleTo` extends internal to named friend assemblies.

### 9.1 [Easy] List C# access modifiers and the defaults.
**Answer**
public, private, protected, internal, protected internal, private protected, and file. Class members default to private; top-level types default to internal.

**Explanation**
Always be explicit in code; defaults differ for interface members (public) and enum members (public).

**Code**
```csharp
class Foo                // internal
{
    int x;               // private
    public int Y;
}
```

### 9.2 [Follow-up] `protected internal` vs `private protected`?
**Answer**
`protected internal` = protected OR internal (more permissive: any code in the assembly plus derived classes anywhere). `private protected` = protected AND internal (only derived classes inside the same assembly).

**Explanation**
Mnemonic: the first word is the strong one for the intersection form.

**Code**
```csharp
public class Base
{
    protected internal void A() { }   // same assembly OR subclasses elsewhere
    private protected void B() { }    // subclasses in this assembly only
}
```

### 9.3 [Medium] What is the `file` modifier and why use it?
**Answer**
A `file`-scoped type is visible only within its source file, avoiding name collisions and hiding helpers — handy with source generators.

**Explanation**
Two files can each declare `file class Helper` without conflict.

**Code**
```csharp
file class Helper { public static string Tag() => "internal-to-file"; }
public class Api { public string Get() => Helper.Tag(); }
```

### 9.4 [Hard] How does `protected` work when accessed through an instance of the base type?
**Answer**
A derived class can access a protected member only through an instance of its own type (or a more derived type), not through a base-typed reference.

**Explanation**
This prevents a `Dog` from poking protected state of a `Cat` sibling via an `Animal` reference.

**Code**
```csharp
class Animal { protected int Age; }
class Dog : Animal
{
    void Test(Dog d, Animal a)
    {
        Age = 1;      // OK (this)
        d.Age = 2;    // OK
        // a.Age = 3; // error CS1540
    }
}
```

### 9.5 [Tricky] Can a public method return a less accessible type? What about a derived class widening accessibility?
**Answer**
No — "inconsistent accessibility" (CS0050/CS0051): a public member's signature types must be at least as accessible. Overrides can't change accessibility, but a `new` hiding member can.

**Explanation**
Test your assembly boundary reasoning: an internal type in a public method signature breaks callers outside the assembly.

**Code**
```csharp
internal class Secret { }
public class Api
{
    // public Secret Get() => new();    // error CS0050
    internal Secret Get() => new();     // fine
}
class B { protected void M() { } }
class D : B { public new void M() { } } // hiding can widen; override cannot
```

**Interview traps**
- `private protected` ≠ `protected internal`.
- Default for top-level class is `internal`, not public.
- `protected` access via base-typed reference doesn't compile.
- Reflection bypasses access modifiers (not a security boundary).

**Cheat sheet**
- Member default: private; type default: internal.
- `protected internal` = OR; `private protected` = AND.
- `file` = file-local type (C# 11).
- Signature types can't be less accessible than the member.

---

## Topic 10: Partial Classes & Methods

**Study notes**
- `partial` splits a type's definition across files in the same assembly/namespace; the compiler merges them. All parts need `partial` and compatible accessibility; modifiers like `sealed`, `abstract`, base class and interfaces are the union across parts.
- Uses: designer/generated code (EF scaffolds, WinForms), source generators (`[GeneratedRegex]`, `LoggerMessage`), separating concerns.
- **Partial methods**: classic form (declaration without body, `void`, implicitly private, removed at compile time if no implementation, no `out`). Since C# 9, with an access modifier / non-void / `out` params, an implementation is **required**.
- C# 13 adds partial properties/indexers (not C# 12).

### 10.1 [Easy] What is a partial class and why use it?
**Answer**
A class whose definition is split across multiple files, merged at compile time into one type. It's mainly for separating generated code from hand-written code.

**Explanation**
Both parts must be in the same assembly and namespace and carry the `partial` keyword.

**Code**
```csharp
// Customer.cs
public partial class Customer { public string Name { get; set; } = ""; }
// Customer.Validation.cs
public partial class Customer { public bool IsValid() => Name.Length > 0; }
```

### 10.2 [Follow-up] What are partial methods and how do they differ from regular methods?
**Answer**
A partial method has a declaration in one part and an optional implementation in another. In the classic form (void, private, no `out`), if there's no implementation, the compiler removes all calls.

**Explanation**
Gives generated code extension hooks with zero cost when unused.

**Code**
```csharp
public partial class Gen
{
    partial void OnCreated();           // hook, optional
    public Gen() { OnCreated(); }
}
public partial class Gen
{
    partial void OnCreated() => Console.WriteLine("created");
}
```

### 10.3 [Medium] What changed for partial methods in C# 9, and how do source generators use them?
**Answer**
Partial methods can now have accessibility modifiers, non-void return types, and `out` parameters, but then an implementation is mandatory. Source generators supply the implementation, e.g. `[GeneratedRegex]` on a `partial` method.

**Explanation**
The generator emits the body into another partial part at compile time, so you get AOT-friendly code instead of reflection.

**Code**
```csharp
using System.Text.RegularExpressions;

public static partial class Validators
{
    [GeneratedRegex(@"^\d{5}$")]
    public static partial Regex ZipRegex();
}
Console.WriteLine(Validators.ZipRegex().IsMatch("12345"));   // Output: True
```

### 10.4 [Hard] How are attributes, base types and members merged across partial parts? What can go wrong?
**Answer**
Attributes and interface lists are combined; if parts specify different base classes it's an error; members can't be defined twice. Access modifiers must agree if specified on multiple parts.

**Explanation**
Typical pitfalls: duplicate member names across parts, partial in different namespaces/assemblies (produces two distinct types), and generated-file overwrite — never edit generated parts; extend via another partial.

**Code**
```csharp
[Serializable] public partial class Doc : IDisposable { }
[Obsolete("x")] public partial class Doc : ICloneable
{
    public void Dispose() { }
    public object Clone() => new Doc();
}
// Doc has both attributes and both interfaces.
```

**Interview traps**
- Partial classes aren't multiple inheritance or cross-assembly.
- Classic partial methods are private and disappear if unimplemented.
- Edits to generated files are lost.
- Not a substitute for good class design (god class split in files).

**Cheat sheet**
- `partial` on every part, same assembly + namespace.
- Union of interfaces/attributes; single base class.
- Classic partial methods optional; accessible ones required.
- Source generators rely on partial.

---

## Topic 11: Explicit Interface Implementation

**Study notes**
- `void IFoo.Bar() { }` – no access modifier; member is accessible **only** through an interface-typed reference.
- Uses: resolve name collisions between interfaces; hide noisy members from the public API (e.g., `ICollection<T>.IsReadOnly`, `IDisposable` patterns); implement different behavior per interface.
- Implicit impl (public method) satisfies the interface *and* is part of the class API.
- Explicit members can't be `virtual`/`abstract`/`public`; a derived class can't call them via `base` (workaround: protected virtual helper).

### 11.1 [Easy] What is explicit interface implementation?
**Answer**
Implementing an interface member by qualifying it with the interface name so it's only reachable through that interface reference, not the class.

**Explanation**
Helps hide members and avoid ambiguity.

**Code**
```csharp
interface IPlayer { void Play(); }
class Media : IPlayer { void IPlayer.Play() => Console.WriteLine("playing"); }

var m = new Media();
// m.Play();            // error
((IPlayer)m).Play();    // Output: playing
```

### 11.2 [Follow-up] When do you *have* to use it?
**Answer**
When two interfaces declare a member with the same signature and you need different implementations, or when the member would conflict with a class member of the same signature.

**Explanation**
Also used to implement a get-only property of an interface as a different-typed public property.

**Code**
```csharp
interface IA { string Name(); }
interface IB { string Name(); }
class Both : IA, IB
{
    string IA.Name() => "A";
    string IB.Name() => "B";
}
IA a = new Both(); IB b = new Both();
Console.WriteLine(a.Name() + b.Name());   // Output: AB
```

### 11.3 [Medium] How do you allow derived classes to customize an explicitly-implemented member?
**Answer**
Explicit implementations can't be virtual, so delegate to a `protected virtual` method that derived classes override.

**Explanation**
This is the pattern used by many framework classes (e.g., `Dispose(bool)`).

**Code**
```csharp
interface IRunner { void Run(); }
class BaseRunner : IRunner
{
    void IRunner.Run() => RunCore();
    protected virtual void RunCore() => Console.WriteLine("base");
}
class MyRunner : BaseRunner { protected override void RunCore() => Console.WriteLine("mine"); }
((IRunner)new MyRunner()).Run();   // Output: mine
```

### 11.4 [Hard] What are the implicit/explicit mixing rules and the struct boxing impact?
**Answer**
A class can implement some members implicitly and others explicitly. Calling an explicit member on a struct requires casting to the interface, which boxes the struct.

**Explanation**
Explicit implementation on a struct can still avoid boxing in generic code via constrained calls `where T : IFoo`.

**Code**
```csharp
interface IVal { int Get(); }
struct V : IVal { int IVal.Get() => 42; }
static int Read<T>(T t) where T : IVal => t.Get();   // constrained call, no boxing
Console.WriteLine(Read(new V()));   // Output: 42
```

**Interview traps**
- Explicit members have no access modifier and aren't visible on the class.
- Can't be virtual; use protected virtual helper.
- Implicit implementation can satisfy multiple interfaces with the same signature at once.
- Casting a struct to an interface boxes.

**Cheat sheet**
- `void IFoo.M()` – reachable only via `IFoo`.
- Solves collisions & hides API noise.
- Not virtual → delegate to protected virtual.
- Struct + interface cast = box.

---

## Topic 12: Composition vs Inheritance

**Study notes**
- Inheritance = "is-a", compile-time fixed, white-box reuse (derived sees base internals), strong coupling.
- Composition = "has-a", objects collaborate through interfaces, runtime swappable, black-box reuse.
- Guidance (GoF): *favor object composition over class inheritance*. Inherit when there is a true substitutable subtype (LSP) and the base is designed for it.
- Delegation + DI is the usual C# way; decorator is composition applied to an interface.

### 12.1 [Easy] What's the difference between inheritance and composition?
**Answer**
Inheritance models "is-a" by deriving from a class; composition models "has-a" by holding another object and delegating to it. Composition is looser and more flexible.

**Explanation**
Test: "A Car *has an* Engine" (composition), "a Dog *is an* Animal" (inheritance).

**Code**
```csharp
class Engine { public void Start() => Console.WriteLine("vroom"); }
class Car
{
    private readonly Engine _engine = new();
    public void Start() => _engine.Start();   // delegation
}
```

### 12.2 [Follow-up] Why is composition generally preferred?
**Answer**
It reduces coupling, allows swapping behavior at runtime and via DI, avoids fragile base class issues, and is easier to unit test with fakes.

**Explanation**
Inheritance hierarchies explode with combinations (`LoggingCachingRepo`...); composition solves via wrappers.

**Code**
```csharp
interface IClock { DateTime Now { get; } }
class Service(IClock clock)
{
    public bool IsMorning() => clock.Now.Hour < 12;   // testable via fake IClock
}
```

### 12.3 [Medium] Square-extends-Rectangle: why does inheritance fail here?
**Answer**
It violates the Liskov Substitution Principle: setting `Width` on a Square must also change `Height`, breaking code that assumes independent setters.

**Explanation**
Behavioral subtyping matters beyond structural compatibility. Prefer immutable shapes or a common `IShape`.

**Code**
```csharp
class Rect { public virtual int W { get; set; } public virtual int H { get; set; } public int Area => W * H; }
class Square : Rect
{
    public override int W { get => base.W; set { base.W = value; base.H = value; } }
    public override int H { get => base.H; set { base.W = value; base.H = value; } }
}
Rect r = new Square(); r.W = 5; r.H = 2;
Console.WriteLine(r.Area);   // Output: 4 (client expected 10)
```

### 12.4 [Hard] How do you replace inheritance with composition while keeping polymorphism?
**Answer**
Extract an interface, hold a field of that interface, and delegate; wrap with decorators for cross-cutting behavior instead of subclassing.

**Explanation**
Decorators let you stack caching/logging in any order at composition root.

**Code**
```csharp
interface IStore { string Get(int id); }
class DbStore : IStore { public string Get(int id) => $"row{id}"; }
class LoggingStore(IStore inner) : IStore
{
    public string Get(int id) { Console.WriteLine($"get {id}"); return inner.Get(id); }
}
IStore s = new LoggingStore(new DbStore());
Console.WriteLine(s.Get(1));
// Output:
// get 1
// row1
```

**Interview traps**
- "Never use inheritance" – overstated; use it for real subtypes.
- LSP is about behavior, not just compile-compatibility.
- Delegation without an interface still couples to the concrete type.

**Cheat sheet**
- is-a → inherit, has-a → compose.
- Favor composition + interfaces + DI.
- LSP violations signal wrong inheritance.
- Decorator = composition-based extension.

---

## Topic 13: SOLID with C# Examples

**Study notes**
| Letter | Principle | One-liner |
|---|---|---|
| S | Single Responsibility | One reason to change |
| O | Open/Closed | Open to extension, closed to modification |
| L | Liskov Substitution | Subtypes must be usable wherever base is expected |
| I | Interface Segregation | Many small client-specific interfaces |
| D | Dependency Inversion | Depend on abstractions; high-level doesn't depend on low-level |

DI containers (`Microsoft.Extensions.DependencyInjection`) are an implementation tool for DIP; DIP ≠ DI ≠ IoC (principle vs technique vs general concept).

### 13.1 [Easy] Explain the Single Responsibility Principle with an example.
**Answer**
A class should have one reason to change. Don't mix business logic, persistence, and formatting in one class.

**Explanation**
Split `InvoiceService` into calculation, repository, and printing.

**Code**
```csharp
class InvoiceCalculator { public decimal Total(IEnumerable<decimal> lines) => lines.Sum(); }
class InvoiceRepository { public void Save(decimal total) { /* db */ } }
class InvoicePrinter { public string Print(decimal total) => $"Total: {total}"; }
```

### 13.2 [Follow-up] Open/Closed: how do you add a new discount without editing existing code?
**Answer**
Depend on an abstraction (`IDiscount`) and add new implementations; the calculator iterates strategies instead of using `switch` on type.

**Explanation**
Replaces `if/else` chains; new behavior = new class registered in DI.

**Code**
```csharp
interface IDiscount { decimal Apply(decimal price); }
class NoDiscount : IDiscount { public decimal Apply(decimal p) => p; }
class Holiday : IDiscount { public decimal Apply(decimal p) => p * 0.9m; }
class Checkout(IDiscount d) { public decimal Total(decimal p) => d.Apply(p); }
Console.WriteLine(new Checkout(new Holiday()).Total(100));   // Output: 90.0
```

### 13.3 [Medium] Interface Segregation vs Liskov — give violation examples.
**Answer**
ISP: a fat `IWorker { Work(); Eat(); }` forces a `Robot` to implement `Eat`; split into `IWorkable`/`IFeedable`. LSP: a `ReadOnlyFile : File` whose `Write` throws `NotSupportedException` breaks callers of base.

**Explanation**
Both signal wrong abstraction; fix by smaller roles.

**Code**
```csharp
interface IWorkable { void Work(); }
interface IFeedable { void Eat(); }
class Human : IWorkable, IFeedable { public void Work() { } public void Eat() { } }
class Robot : IWorkable { public void Work() { } }
```

### 13.4 [Hard] Dependency Inversion: show a violation and its fix using constructor injection.
**Answer**
Violation: `OrderService` does `new SqlRepository()` internally. Fix: depend on `IOrderRepository`, inject via ctor, and wire in the DI container so high- and low-level modules both depend on the abstraction.

**Explanation**
Enables testing with a fake and swapping storage; the interface is owned by the high-level module.

**Code**
```csharp
using Microsoft.Extensions.DependencyInjection;

interface IOrderRepository { void Save(string order); }
class SqlOrderRepository : IOrderRepository { public void Save(string o) => Console.WriteLine("sql: " + o); }
class OrderService(IOrderRepository repo) { public void Place(string o) => repo.Save(o); }

var sp = new ServiceCollection()
    .AddScoped<IOrderRepository, SqlOrderRepository>()
    .AddScoped<OrderService>()
    .BuildServiceProvider();
sp.GetRequiredService<OrderService>().Place("A1");   // Output: sql: A1
```

### 13.5 [Tricky] Is every `if/else` or `switch` a violation of OCP? When is over-applying SOLID harmful?
**Answer**
No. Apply OCP where variation actually occurs; premature abstraction adds indirection (YAGNI). A switch over a closed enum that rarely changes is fine.

**Explanation**
Interviewers like nuance: SOLID are heuristics. A one-implementation interface "for DI" without a seam need can be noise.

**Code**
```csharp
static string Label(DayOfWeek d) => d switch
{
    DayOfWeek.Saturday or DayOfWeek.Sunday => "Weekend",
    _ => "Weekday"
};   // closed, stable domain: a strategy hierarchy would be overkill
```

**Interview traps**
- DIP ≠ DI ≠ IoC.
- SRP isn't "one method per class"; it's one reason to change.
- LSP is behavioral, including preconditions/postconditions and exceptions.
- OCP via inheritance only is brittle; prefer strategy/composition.

**Cheat sheet**
- S: one reason to change.
- O: extend via new types, not edits.
- L: subtype honors base contract.
- I: small client-focused interfaces.
- D: depend on abstractions, inject them.

---

## Topic 14: Indexers & Operator Overloading

**Study notes**
- **Indexer**: `public T this[int i] { get; set; }` – property-like, parameters, can be overloaded by parameter type, can be get-only, can be `virtual`, declared in interfaces. Name is `Item` in IL (`[IndexerName]` can change). C# 8 adds `Index`/`Range` support (`^1`, `1..3`) automatically if the type has `Length`/`Count` and an int indexer (and `Slice` for ranges).
- **Operators**: `public static` methods; at least one parameter must be the containing type. Overloadable: unary `+ - ! ~ ++ -- true false`, binary `+ - * / % & | ^ << >> == != < > <= >=`. Comparison operators must be overloaded in pairs (`==`/`!=`, `<`/`>`, `<=`/`>=`). `&&`/`||` are derived from `&`/`|` plus `true`/`false`. Not overloadable: `=`, `.`, `?:`, `new`, `is`, `as`, `[]` (use indexer). Compound `+=` comes from `+`.
- User-defined conversions: `implicit`/`explicit` operators. Override `Equals`/`GetHashCode` together with `==`.

### 14.1 [Easy] What is an indexer and how do you declare one?
**Answer**
An indexer lets an object be accessed with array syntax. It is declared like a property named `this` with parameters.

**Explanation**
Useful for collection-like types (matrix, dictionary wrapper).

**Code**
```csharp
class Playlist
{
    private readonly List<string> _songs = new() { "A", "B", "C" };
    public string this[int i] { get => _songs[i]; set => _songs[i] = value; }
    public int this[string name] => _songs.IndexOf(name);   // overload by param type
}
var p = new Playlist();
Console.WriteLine(p[1]);     // Output: B
Console.WriteLine(p["C"]);   // Output: 2
```

### 14.2 [Follow-up] How do you overload `+` and `==` for a value type like `Money`?
**Answer**
Declare `public static` operator methods; for `==` also define `!=`, and override `Equals` and `GetHashCode` so they're consistent.

**Explanation**
The compiler warns CS0660/CS0661 if you overload `==` without them. Records/record structs generate these for you.

**Code**
```csharp
public readonly struct Money : IEquatable<Money>
{
    public decimal Amount { get; }
    public Money(decimal a) => Amount = a;
    public static Money operator +(Money a, Money b) => new(a.Amount + b.Amount);
    public static bool operator ==(Money a, Money b) => a.Equals(b);
    public static bool operator !=(Money a, Money b) => !a.Equals(b);
    public bool Equals(Money o) => Amount == o.Amount;
    public override bool Equals(object? o) => o is Money m && Equals(m);
    public override int GetHashCode() => Amount.GetHashCode();
}
Console.WriteLine(new Money(1) + new Money(2) == new Money(3));   // Output: True
```

### 14.3 [Medium] Implicit vs explicit conversion operators — when to use which?
**Answer**
Use `implicit` only when conversion is lossless and can't throw; use `explicit` for lossy/throwing conversions so callers must cast.

**Explanation**
Implicit conversions can cause surprising overload resolution; use sparingly. Conversions to/from interfaces or base types aren't allowed.

**Code**
```csharp
readonly record struct Celsius(double Value)
{
    public static implicit operator double(Celsius c) => c.Value;
    public static explicit operator Celsius(double d) =>
        d < -273.15 ? throw new ArgumentOutOfRangeException(nameof(d)) : new Celsius(d);
}
double d = new Celsius(20);          // implicit
var c = (Celsius)25.0;               // explicit
Console.WriteLine($"{d} {c.Value}"); // Output: 20 25
```

### 14.4 [Hard] How do `^` and `..` work with your custom indexer? And C# 11 `checked` / generic math operators?
**Answer**
If a type is countable (`Count`/`Length`) and has an `int` indexer, `x[^1]` compiles as `x[x.Count - 1]` (implicit Index support); ranges need a `Slice(int,int)` method. C# 11 lets you define `checked` operator variants (`operator checked +`).

**Explanation**
Or define an indexer taking `Index`/`Range` explicitly.

**Code**
```csharp
class Bag
{
    private readonly int[] _a = { 10, 20, 30 };
    public int Count => _a.Length;
    public int this[int i] => _a[i];
}
var b = new Bag();
Console.WriteLine(b[^1]);   // Output: 30  (compiler rewrites to b[b.Count - 1])
```

### 14.5 [Tricky] What happens with `==` overloading and `null` checks / reference comparison?
**Answer**
Inside the overloaded `==`, `a == null` recurses infinitely; use `is null` or `ReferenceEquals`. Also `object o1 == o2` with both typed `object` does reference comparison regardless of overloads.

**Explanation**
Overloads are static-bound by compile-time types (like `string == object`).

**Code**
```csharp
class Tag
{
    public string V = "";
    public static bool operator ==(Tag? a, Tag? b) => a is null ? b is null : b is not null && a.V == b.V;
    public static bool operator !=(Tag? a, Tag? b) => !(a == b);
    public override bool Equals(object? o) => o is Tag t && this == t;
    public override int GetHashCode() => V.GetHashCode();
}
object x = new Tag { V = "a" }, y = new Tag { V = "a" };
Console.WriteLine(x == y);          // Output: False (object == object is reference equality)
Console.WriteLine((Tag)x == (Tag)y);// Output: True
```

**Interview traps**
- `==` without `Equals`/`GetHashCode` consistency.
- `a == null` inside `operator ==` → stack overflow.
- Operators resolved statically.
- Indexers can't be `static`.

**Cheat sheet**
- Indexer = `this[...]` property; overloadable.
- Operators: public static, pair them.
- implicit = safe, explicit = may lose/throw.
- Countable + int indexer ⇒ `^` works.

---

## Topic 15: Object Initializers, init-only, Primary Constructors (C# 12)

**Study notes**
- **Object initializer**: `new T { A = 1 }` calls the ctor then sets accessible members. Collection initializers call `Add`.
- **`init` accessor (C# 9)**: settable only during construction/initializer/`with`; compile-time enforced (reflection can still set). **`required` (C# 11)**: caller must set the member. `[SetsRequiredMembers]` on a ctor.
- **Records**: reference-type value semantics (`record`), `record struct`; positional ones generate `init` props, `Deconstruct`, `with`.
- **Primary constructors (C# 12)** for classes & structs: `class Svc(IRepo repo)`. Parameters are in scope in the type body but are **not members**; the compiler captures them as hidden mutable fields when used in members. In records, positional parameters become properties. Other ctors must chain `: this(...)`.

### 15.1 [Easy] What is an object initializer?
**Answer**
Syntax to set properties/fields right after construction in one expression: `new Person { Name = "A" }`. It runs the constructor first, then assigns members in order.

**Explanation**
Not the same as a ctor: members must be accessible and settable.

**Code**
```csharp
class Person { public string Name { get; set; } = ""; public int Age { get; set; } }
var p = new Person { Name = "Ann", Age = 30 };
var list = new List<int> { 1, 2, 3 };    // collection initializer → Add
```

### 15.2 [Follow-up] What is `init` and how is it different from `private set` and `readonly`?
**Answer**
`init` can be set in object initializers, ctors, and `with`, then is immutable. `private set` allows mutation by the class at any time; `readonly` fields are only settable in ctors/initializers and are for fields, not properties.

**Explanation**
Enables immutable types with initializer syntax.

**Code**
```csharp
class Point { public int X { get; init; } public int Y { get; init; } }
var p = new Point { X = 1, Y = 2 };
// p.X = 5;   // error CS8852
```

### 15.3 [Medium] What does `required` add?
**Answer**
It forces callers to set the member in an object initializer (or via a ctor marked `[SetsRequiredMembers]`), giving compile-time safety without a big constructor.

**Explanation**
Works on fields and properties; interacts well with `init`. Serializers like System.Text.Json (.NET 7+) honor it.

**Code**
```csharp
class User { public required string Email { get; init; } public string? Nick { get; init; } }
var u = new User { Email = "a@b.com" };
// var bad = new User();   // error CS9035
```

### 15.4 [Hard] How do primary constructors in classes capture parameters and how do they differ from records?
**Answer**
In a class/struct, primary ctor parameters are plain parameters whose scope is the whole type; if used in a member they're captured into compiler-generated mutable fields — not properties. Records' positional parameters instead generate public `init` properties (and equality/`with`/`Deconstruct`).

**Explanation**
Consequences: no `Name` member unless you declare one; parameters are not `readonly` (can be reassigned); capturing a parameter in a member while also using it to initialize a property triggers warning CS9124 (stored twice).

**Code**
```csharp
public class Counter(int start)
{
    public int Current { get; private set; } = start;   // initialize from param
    public void Inc() => Current++;
}
public record Pt(int X, int Y);          // X, Y are properties
public class Svc(IDisposable d)          // captured only if used in a member
{
    public void Close() => d.Dispose();  // d becomes a hidden field
}
Console.WriteLine(new Pt(1, 2));         // Output: Pt { X = 1, Y = 2 }
```

### 15.5 [Tricky] What's the order of evaluation: field initializers, ctor body, and object initializer? Can `init` be set in the ctor of a derived class?
**Answer**
Field/property initializers → ctor body → object-initializer assignments (after ctor returns). `init` accessors are callable in ctors of the same type and derived types (via `this`/`base`), and in object initializers, but not afterward.

**Explanation**
Therefore ctor logic can't see values from the object initializer, which matters for validation — prefer `required`/ctor params for invariants.

**Code**
```csharp
class A
{
    public int X { get; init; } = Log("field init");
    public A() => Log("ctor");
    static int Log(string s) { Console.WriteLine(s); return 0; }
}
var a = new A { X = Log("initializer") };
// Output:
// field init
// ctor
// initializer
```

**Gotchas / follow-ups**
- Object initializer with `init` + `with` expressions on classes requires records (or implement a clone).
- Primary ctor + explicit ctor: extras must call `this(...)`.

**Interview traps**
- Primary-ctor params aren't properties (unlike records).
- `init` isn't deep immutability (list can still mutate).
- Initializer assignments run *after* the ctor.
- `required` isn't runtime enforcement for reflection/deserialization unless supported.

**Cheat sheet**
- Initializer = ctor then member sets.
- `init` = set at creation only; `required` = must set.
- Record positional → properties; class primary ctor → captured params.
- `with` copies via compiler copy ctor.

---

## Topic 16: Static Classes & Extension Methods

**Study notes**
- `static class`: implicitly `abstract sealed`; only static members; no instances, no inheritance; may have a static ctor.
- **Extension method**: static method in a non-generic, non-nested static class, first parameter `this T x`. Compiles to a plain static call: `x.Foo()` → `Ext.Foo(x)`. Resolved at **compile time** via `using` namespaces; instance methods always win over extensions.
- Works on null receivers (no NRE until you dereference). Works on interfaces (LINQ). C# 14 introduces extension blocks (not covered here).
- Static classes with mutable state are global state → thread-safety and testability issues.

### 16.1 [Easy] What is a static class and what can it contain?
**Answer**
A class that can't be instantiated or inherited and contains only static members; used for utilities (e.g., `Math`, `Console`).

**Explanation**
Compiler marks it abstract + sealed. It can have a static constructor but no instance members.

**Code**
```csharp
public static class MathX
{
    public static int Square(int x) => x * x;
}
Console.WriteLine(MathX.Square(4));   // Output: 16
```

### 16.2 [Follow-up] How do you write an extension method, and how is it resolved?
**Answer**
Declare it as a static method in a static class with `this` on the first parameter; callers must import the namespace. The compiler rewrites it to a static call, so no runtime dispatch occurs.

**Explanation**
Doesn't give access to private members; can't override; instance methods with the same signature take priority.

**Code**
```csharp
public static class StringExt
{
    public static bool IsNullOrBlank(this string? s) => string.IsNullOrWhiteSpace(s);
}
string? s = null;
Console.WriteLine(s.IsNullOrBlank());   // Output: True (no NRE: it's a static call)
```

### 16.3 [Medium] What happens when an extension method conflicts with an instance method? Can extensions extend interfaces?
**Answer**
The instance method always wins, even if the extension is a better match; extensions on interfaces apply to every implementer (that's how LINQ's `IEnumerable<T>` methods work).

**Explanation**
Adding a method later to the type can silently change which method is called.

**Code**
```csharp
class Foo { public void Hi(object o) => Console.WriteLine("instance"); }
static class FooExt { public static void Hi(this Foo f, int i) => Console.WriteLine("extension"); }
new Foo().Hi(5);   // Output: instance   (instance applicable → wins)

public static class EnumerableExt
{
    public static IEnumerable<T> Evens<T>(this IEnumerable<T> src) => src.Where((_, i) => i % 2 == 0);
}
```

### 16.4 [Hard] Static class pitfalls: state, testing, and static constructor interplay?
**Answer**
Static mutable state is shared process-wide (not thread-safe by default), hard to mock, and lives for the app's lifetime. A static constructor failure poisons the class (`TypeInitializationException`). Prefer DI with interfaces for anything with dependencies.

**Explanation**
Stateless pure helpers are fine as static. Don't use static classes as service locators.

**Code**
```csharp
public static class Counter
{
    private static int _n;
    public static int Next() => Interlocked.Increment(ref _n);   // thread-safe
}
```

### 16.5 [Tricky] Can extension methods be `virtual`, access private members, or be used with `ref this` on structs? Do they work with generics constraints?
**Answer**
Not virtual and no private access. For structs, `this ref T` / `this in T` allows mutating or avoiding copies (`ref` extension on value types only). Generic constraints work normally.

**Explanation**
Without `ref`, an extension on a struct receives a copy so mutations are lost.

**Code**
```csharp
public struct Pt { public int X; }
public static class PtExt
{
    public static void IncNoRef(this Pt p) => p.X++;        // modifies a copy
    public static void IncRef(this ref Pt p) => p.X++;      // modifies original
}
var pt = new Pt();
pt.IncNoRef(); Console.WriteLine(pt.X);   // Output: 0
pt.IncRef();   Console.WriteLine(pt.X);   // Output: 1
```

**Interview traps**
- Instance method beats extension.
- Extension on null receiver works.
- Not true "adding" methods: no reflection visibility on the type.
- Static classes are not DI-friendly.

**Cheat sheet**
- Static class = abstract sealed, static members only.
- Extension = static method with `this` first param, compile-time sugar.
- Namespace import needed.
- `this ref` for struct mutation.

---

## Topic 17: Nested Classes

**Study notes**
- A type declared inside another type. Nested default accessibility: `private`; can be any modifier.
- Nested type can access **private** members of the containing type (given an instance); the outer can't see the nested's privates.
- Unlike Java inner classes, C# nested classes have **no implicit reference** to an outer instance; pass it explicitly.
- Use cases: implementation details (enumerators, builders), private helpers, state machines, strongly-coupled types (e.g. `Builder` inside its product), singleton holder.
- Referenced as `Outer.Inner`; generics: nested type shares outer's type parameters.

### 17.1 [Easy] What is a nested class and why use it?
**Answer**
A class declared inside another class, used for tightly-coupled helpers that should be hidden from the outside. Its default accessibility is private.

**Explanation**
Improves encapsulation and namespace organization.

**Code**
```csharp
public class Order
{
    private class Line { public string Sku = ""; }    // implementation detail
    private readonly List<Line> _lines = new();
    public void Add(string sku) => _lines.Add(new Line { Sku = sku });
}
```

### 17.2 [Follow-up] Can a nested class access private members of the outer class? And vice versa?
**Answer**
Yes, the nested class can access outer private members (through an instance reference), but the outer class can't access the nested class's private members.

**Explanation**
Unlike Java there's no automatic outer instance.

**Code**
```csharp
class Outer
{
    private int _secret = 42;
    public class Inner
    {
        public int Read(Outer o) => o._secret;   // allowed
    }
}
Console.WriteLine(new Outer.Inner().Read(new Outer()));   // Output: 42
```

### 17.3 [Medium] Nested builder / private implementation pattern — show it.
**Answer**
Put the builder inside the product, so the product's ctor can be private yet only the nested builder can call it.

**Explanation**
Guarantees construction only via the builder; common for immutable types.

**Code**
```csharp
public sealed class Email
{
    public string To { get; }
    public string Subject { get; }
    private Email(string to, string subject) { To = to; Subject = subject; }

    public sealed class Builder
    {
        private string _to = "", _subject = "";
        public Builder To(string t) { _to = t; return this; }
        public Builder Subject(string s) { _subject = s; return this; }
        public Email Build() => new(_to, _subject);   // can call private ctor
    }
}
var e = new Email.Builder().To("a@x.com").Subject("Hi").Build();
Console.WriteLine(e.Subject);   // Output: Hi
```

### 17.4 [Hard] How do nested types interact with generics and inheritance?
**Answer**
A nested type in a generic outer is itself generic over the outer's parameters: `Outer<int>.Inner` and `Outer<string>.Inner` are distinct types with separate statics. A nested type can inherit its outer class.

**Explanation**
Gotcha: static fields in nested types are per closed outer type, and a static ctor runs per closed instantiation (see Topic 2).

**Code**
```csharp
class Outer<T>
{
    public class Inner { public static int Count; }
}
Outer<int>.Inner.Count = 5;
Console.WriteLine(Outer<string>.Inner.Count);   // Output: 0
```

**Interview traps**
- No implicit outer reference in C#.
- Default accessibility is private.
- Nested in generic → also generic.
- Overuse hurts readability; prefer separate files for public nested types.

**Cheat sheet**
- Nested default = private.
- Can read outer privates; reverse not.
- Explicit outer instance needed.
- Perfect for builders/enumerators/helpers.

---

## Topic 18: Common Design Patterns (Singleton, Factory, Strategy, Decorator, Observer, Repository)

**Study notes**
| Pattern | Intent | .NET real-world |
|---|---|---|
| Singleton | One instance, global access | DI `AddSingleton` (preferred), `Lazy<T>` |
| Factory | Hide creation logic | `ILoggerFactory`, `HttpClientFactory` |
| Strategy | Swap algorithms at runtime | `IComparer<T>`, `IPasswordHasher` |
| Decorator | Add behavior by wrapping same interface | `Stream` wrappers (`GZipStream`), middleware-ish, Scrutor |
| Observer | Notify subscribers of changes | `event`, `IObservable<T>`, `INotifyPropertyChanged` |
| Repository | Abstract data access | custom over EF Core (DbContext is already UoW/repository-like) |

### 18.1 [Easy] Implement a thread-safe Singleton in C#.
**Answer**
Use `Lazy<T>` (thread-safe by default) or a static readonly field initialized by the CLR. Hide the ctor and make the class sealed. In ASP.NET Core prefer registering it as a DI singleton.

**Explanation**
Avoid double-checked locking unless needed: needs `volatile`. `Lazy<T>` default mode `ExecutionAndPublication` ensures a single factory execution.

**Code**
```csharp
public sealed class AppSettings
{
    private static readonly Lazy<AppSettings> _lazy = new(() => new AppSettings());
    public static AppSettings Instance => _lazy.Value;
    private AppSettings() { }
}
Console.WriteLine(ReferenceEquals(AppSettings.Instance, AppSettings.Instance));   // Output: True
```

### 18.2 [Follow-up] What are the alternatives and pitfalls of Singleton?
**Answer**
Static-field singleton (eager, CLR-guaranteed), `Lazy<T>`, double-checked locking with `volatile`, and DI singleton lifetime. Pitfalls: hidden global state, hard to test, concurrency on mutable state, and a captive-dependency problem if a singleton depends on a scoped service.

**Explanation**
Singleton-ness through DI is testable since consumers depend on an interface.

**Code**
```csharp
public sealed class Eager
{
    public static readonly Eager Instance = new();
    static Eager() { }          // precise init (no beforefieldinit)
    private Eager() { }
}
public sealed class Dcl
{
    private static volatile Dcl? _i;
    private static readonly object _gate = new();
    public static Dcl Instance
    {
        get
        {
            if (_i is null) lock (_gate) { _i ??= new Dcl(); }
            return _i;
        }
    }
    private Dcl() { }
}
```

### 18.3 [Medium] Factory + Strategy: select a payment processor at runtime.
**Answer**
Strategy defines a family of interchangeable algorithms behind an interface; a Factory chooses the right one. In DI, you can inject `IEnumerable<IPaymentStrategy>` or a keyed service (.NET 8).

**Explanation**
Adding a new strategy needs no change to callers (OCP).

**Code**
```csharp
interface IPayment { string Name { get; } decimal Pay(decimal amt); }
class Card : IPayment { public string Name => "card"; public decimal Pay(decimal a) => a * 1.02m; }
class Upi  : IPayment { public string Name => "upi";  public decimal Pay(decimal a) => a; }

class PaymentFactory(IEnumerable<IPayment> all)
{
    public IPayment Create(string name) =>
        all.FirstOrDefault(p => p.Name == name) ?? throw new NotSupportedException(name);
}
var f = new PaymentFactory(new IPayment[] { new Card(), new Upi() });
Console.WriteLine(f.Create("card").Pay(100));   // Output: 102.00
```

### 18.4 [Medium] Implement the Decorator pattern (caching around a repository).
**Answer**
A decorator implements the same interface, holds an inner instance, and adds behavior before/after delegating. Decorators stack in any order.

**Explanation**
Differs from inheritance by being runtime-composed; contrasts with Proxy mainly in intent.

**Code**
```csharp
interface IProductRepo { string Get(int id); }
class SqlProductRepo : IProductRepo { public string Get(int id) { Console.WriteLine("DB hit"); return $"P{id}"; } }
class CachedProductRepo(IProductRepo inner) : IProductRepo
{
    private readonly Dictionary<int, string> _cache = new();
    public string Get(int id)
    {
        if (!_cache.TryGetValue(id, out var v)) _cache[id] = v = inner.Get(id);
        return v;
    }
}
IProductRepo r = new CachedProductRepo(new SqlProductRepo());
r.Get(1); r.Get(1);
// Output:
// DB hit
```

### 18.5 [Hard] Observer: events vs `IObservable<T>`, and the memory-leak gotcha.
**Answer**
C# `event` is a built-in observer (delegate list); `IObservable<T>/IObserver<T>` is the push-based, composable (Rx) form. A publisher holds strong references to subscribers, so forgetting to unsubscribe leaks them (the "lapsed listener" problem).

**Explanation**
Raise events safely with `?.Invoke` (copies delegate reference, avoiding a race with unsubscribe). Long-lived publisher + short-lived subscriber → unsubscribe or use weak events.

**Code**
```csharp
class Stock
{
    public event Action<decimal>? PriceChanged;
    private decimal _price;
    public decimal Price
    {
        get => _price;
        set { if (_price == value) return; _price = value; PriceChanged?.Invoke(value); }
    }
}
var s = new Stock();
Action<decimal> h = p => Console.WriteLine($"New price {p}");
s.PriceChanged += h;
s.Price = 10;            // Output: New price 10
s.PriceChanged -= h;     // unsubscribe to avoid leaks
s.Price = 11;            // no output
```

### 18.6 [Tricky] Repository pattern over EF Core: is it still worthwhile? Show a minimal, correct version.
**Answer**
`DbContext` already implements Unit of Work and `DbSet` repository-like behavior, so a generic repository on top is often redundant and leaks `IQueryable`. A repository still pays off to express domain-specific queries, isolate persistence for testing, or support multiple stores.

**Explanation**
Keep it specific (aggregate-oriented), return materialized results, and let a single scoped `DbContext` act as the unit of work (`SaveChangesAsync` once). Avoid exposing `IQueryable` (leaky abstraction, deferred execution outside the scope).

**Code**
```csharp
using Microsoft.EntityFrameworkCore;

public class Product { public int Id { get; set; } public string Name { get; set; } = ""; }
public class ShopContext(DbContextOptions<ShopContext> o) : DbContext(o)
{
    public DbSet<Product> Products => Set<Product>();
}
public interface IProductRepository
{
    Task<Product?> GetAsync(int id, CancellationToken ct = default);
    Task<IReadOnlyList<Product>> SearchAsync(string term, CancellationToken ct = default);
    void Add(Product p);
}
public class ProductRepository(ShopContext db) : IProductRepository
{
    public Task<Product?> GetAsync(int id, CancellationToken ct = default) =>
        db.Products.FirstOrDefaultAsync(p => p.Id == id, ct);

    public async Task<IReadOnlyList<Product>> SearchAsync(string term, CancellationToken ct = default) =>
        await db.Products.AsNoTracking().Where(p => p.Name.Contains(term)).ToListAsync(ct);

    public void Add(Product p) => db.Products.Add(p);   // caller commits via db.SaveChangesAsync
}
```

**Gotchas / follow-ups**
- Register scoped: `AddScoped<IProductRepository, ProductRepository>()`.
- Repositories per aggregate root, not per table.
- Singleton + scoped `DbContext` dependency = captive dependency bug.

**Interview traps**
- Double-checked locking without `volatile` is broken in theory.
- Singleton by `static` vs by DI lifetime – prefer DI.
- Events hold strong refs → leaks; raise with `?.Invoke`.
- Generic `IRepository<T>` exposing `IQueryable<T>` leaks persistence concerns.
- Strategy vs State: strategy chosen by client, state transitions internally.

**Cheat sheet**
- Singleton: `Lazy<T>` / static readonly / DI singleton.
- Factory: encapsulate `new` + selection logic.
- Strategy: interface + swappable implementations.
- Decorator: same interface, wraps inner, adds behavior.
- Observer: `event` / `IObservable<T>`; unsubscribe!
- Repository: aggregate-focused persistence abstraction.

---
*End of A02.*
