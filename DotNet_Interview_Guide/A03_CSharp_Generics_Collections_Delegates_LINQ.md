# A03 - C# Generics, Collections, Delegates, Events, Closures & LINQ

> **How to use:** Read the *Study notes* of a topic first, then work the difficulty ladder top to bottom (Easy -> Tricky); cover the **Answer** and say it aloud before reading it.
> Code targets C# 12 / .NET 8 (implicit usings on); each snippet is a standalone top-level-statements file, so where a snippet declares types before statements, move those type declarations below the statements (compiler rule CS8803). Predict each `// Output:` before checking it.

## Table of Contents

1. [Topic 1: Generics basics & constraints](#topic-1-generics-basics--constraints)
2. [Topic 2: Generics at runtime (reified, JIT sharing)](#topic-2-generics-at-runtime-reified-jit-sharing)
3. [Topic 3: Covariance & contravariance (in/out)](#topic-3-covariance--contravariance-inout)
4. [Topic 4: Array vs List<T>, growth, Span/Memory intro](#topic-4-array-vs-listt-growth-spanmemory-intro)
5. [Topic 5: Dictionary, HashSet, Sorted*, Stack/Queue, LinkedList](#topic-5-dictionary-hashset-sorted-stackqueue-linkedlist)
6. [Topic 6: IEnumerable vs ICollection vs IList vs IReadOnlyList](#topic-6-ienumerable-vs-icollection-vs-ilist-vs-ireadonlylist)
7. [Topic 7: IEnumerable vs IQueryable vs IAsyncEnumerable](#topic-7-ienumerable-vs-iqueryable-vs-iasyncenumerable)
8. [Topic 8: Iterators, yield, IEnumerator & foreach internals](#topic-8-iterators-yield-ienumerator--foreach-internals)
9. [Topic 9: Delegates (Func/Action/Predicate, multicast)](#topic-9-delegates-funcactionpredicate-multicast)
10. [Topic 10: Events (event vs delegate, leaks, weak events)](#topic-10-events-event-vs-delegate-leaks-weak-events)
11. [Topic 11: Lambdas & closures](#topic-11-lambdas--closures)
12. [Topic 12: Anonymous types & tuples](#topic-12-anonymous-types--tuples)
13. [Topic 13: LINQ - deferred execution & core operators](#topic-13-linq---deferred-execution--core-operators)
14. [Topic 14: LINQ performance pitfalls & expression trees](#topic-14-linq-performance-pitfalls--expression-trees)
15. [Topic 15: Immutable & concurrent collections; Dictionary vs ConcurrentDictionary](#topic-15-immutable--concurrent-collections-dictionary-vs-concurrentdictionary)
16. [Topic 16: Comparer / IComparable / sorting stability](#topic-16-comparer--icomparable--sorting-stability)

---

## Topic 1: Generics basics & constraints

**Study notes**

- Generics let you write a type or method once with a **type parameter** (`T`) and get compile-time type safety, no boxing for value types, and no casts.
- Applies to classes, structs, interfaces, delegates, methods (not properties, events, operators, or constructors directly - those use the enclosing type's `T`).
- **Constraints** (`where T : ...`) tell the compiler what members you may use on `T`:

| Constraint | Meaning |
|---|---|
| `where T : class` / `class?` | reference type (non-nullable / nullable-aware) |
| `where T : struct` | non-nullable value type (excludes `Nullable<U>`) |
| `where T : notnull` | non-nullable (ref or value) |
| `where T : unmanaged` | struct with no reference-type fields (usable with pointers/`stackalloc`) |
| `where T : new()` | public parameterless ctor (must be listed **last**) |
| `where T : BaseClass` / `IFoo` | derives from / implements |
| `where T : U` | naked type-constraint: T derives from another type parameter |
| `where T : Enum` / `Delegate` | allowed since C# 7.3 |
| `where T : default` | disambiguates overrides/explicit interface impls with unconstrained T |

- Multiple constraints are comma-separated; `class`/`struct` first, `new()` last.
- Without constraints `T` is treated as `object`: you can call `ToString`, `Equals`, `GetHashCode`, `GetType`, compare to `null`, and use `default(T)`.
- Since C# 11 / .NET 7, **static abstract interface members** + "generic math" (`INumber<T>`, `IAdditionOperators<,,>`) let you constrain `T` to types that support operators like `+`.

---

### 1.1 [Easy] What is a generic type and why use it instead of `object`?

**Answer**
A generic type takes type parameters so one definition works for many types while staying type-safe. Compared to `object`-based collections (`ArrayList`) it removes casts, catches type errors at compile time, and avoids boxing value types.

**Explanation**
`ArrayList.Add(5)` boxes the `int` onto the heap and you must unbox/cast on read - a wrong cast throws `InvalidCastException` at runtime. `List<int>` stores ints directly in an `int[]`, no boxing, and `list.Add("x")` fails to compile.

**Code**
```csharp
using System.Collections;

var old = new ArrayList { 1, "two" };      // compiles; boxing; mixed types
int a = (int)old[0]!;                      // cast required
// int b = (int)old[1]!;                   // InvalidCastException at runtime

var typed = new List<int> { 1, 2 };
// typed.Add("two");                       // compile error
int c = typed[0];                          // no cast, no unboxing
Console.WriteLine(a + c);                  // Output: 2
```

### 1.2 [Follow-up] What constraints can you put on a type parameter, and why are they needed?

**Answer**
`class`, `struct`, `notnull`, `unmanaged`, `new()`, a base class, interfaces, another type parameter, `Enum`, `Delegate`. They are needed because the compiler only lets you use members it can prove exist on `T`.

**Explanation**
An unconstrained `T` is effectively `object`. If you want `a.CompareTo(b)` you need `where T : IComparable<T>`; if you want `new T()` you need `new()`. Constraints also document intent and are checked at every call site, not at runtime.

**Code**
```csharp
static T Max<T>(T a, T b) where T : IComparable<T>
    => a.CompareTo(b) >= 0 ? a : b;

static T Create<T>() where T : class, new() => new T();

Console.WriteLine(Max(3, 7));          // Output: 7
Console.WriteLine(Max("pear", "apple")); // Output: pear
var sb = Create<System.Text.StringBuilder>();
Console.WriteLine(sb.Length);          // Output: 0
```

### 1.3 [Medium] Why can't you write `a + b` for a generic `T`, and how does C# 11 fix that?

**Answer**
Operators are static members, and before C# 11 interfaces could not declare static members, so no constraint could promise `+` exists. C# 11 added static abstract interface members; `INumber<T>` / `IAdditionOperators<T,T,T>` can now be used as constraints.

**Explanation**
Pre-.NET 7 workarounds were `dynamic` (slow), `Expression` trees compiled once, or passing a `Func<T,T,T>`. With generic math the JIT specializes the code per value type, so it is as fast as hand-written `int` code.

**Code**
```csharp
using System.Numerics;

static T Sum<T>(IEnumerable<T> items) where T : INumber<T>
{
    T total = T.Zero;
    foreach (var i in items) total += i;
    return total;
}

Console.WriteLine(Sum(new[] { 1, 2, 3 }));        // Output: 6
Console.WriteLine(Sum(new[] { 1.5, 2.5 }));       // Output: 4
```

### 1.4 [Hard] How does `where T : struct` differ from `where T : notnull` and `where T : unmanaged`? What about `T?`?

**Answer**
`struct` = non-nullable value type; `notnull` = any non-nullable type (reference types get a *nullable-analysis* warning only); `unmanaged` = a struct whose whole tree has no reference fields. `T?` means `Nullable<T>` when `T : struct`, but only an annotation (no runtime type) when `T : class`.

**Explanation**
`notnull` is enforced by flow analysis (warnings), not the runtime. `unmanaged` allows `sizeof`, `T*` and `stackalloc T[]`. An unconstrained `T?` is legal in C# 9+ but means "default-able/maybe null" annotation only - it is *not* `Nullable<T>` when T is a value type; the code must handle both cases.

**Code**
```csharp
static int SizeOf<T>() where T : unmanaged => System.Runtime.CompilerServices.Unsafe.SizeOf<T>();

static T? FirstOrNull<T>(IEnumerable<T> src) where T : struct
{
    foreach (var x in src) return x;       // returns Nullable<T>
    return null;
}

Console.WriteLine(SizeOf<long>());                  // Output: 8
Console.WriteLine(FirstOrNull(new int[0]) is null); // Output: True
// SizeOf<string>();  // compile error: string is not unmanaged
```

**Gotchas / follow-ups**
- Overloads differing only by `class` vs `struct` constraint are **not** allowed (constraints are not part of the signature), so `Foo<T>(T) where T:class` and `Foo<T>(T) where T:struct` conflict.
- `T : struct` with `T?` return needs `Nullable<T>`; with `T : class` the same source means a different thing - they cannot be mixed in one unconstrained method.

### 1.5 [Tricky] What does `x == null` do when `T` is an unconstrained generic and is instantiated with `int`? What does `default(T)` give?

**Answer**
It compiles and evaluates to `false` for non-nullable value types (the JIT removes the check). `default(T)` is `null` for reference types and zero-initialised for value types.

**Explanation**
The C# compiler allows comparing an unconstrained `T` with `null`. For a value-type instantiation the JIT knows the value can never be null and folds the branch. But `x == y` between two `T`s using `==` is **not** allowed without a constraint (reference equality is only available with `where T : class`); use `EqualityComparer<T>.Default.Equals`.

**Code**
```csharp
static bool IsNull<T>(T x) => x == null;
static bool Same<T>(T a, T b) => EqualityComparer<T>.Default.Equals(a, b);

Console.WriteLine(IsNull(5));          // Output: False
Console.WriteLine(IsNull<string>(null)); // Output: True
Console.WriteLine(Same("a", "a"));     // Output: True
Console.WriteLine(default(int));       // Output: 0
```

**Gotchas / follow-ups**
- `static bool Eq<T>(T a, T b) => a == b;` fails to compile unless `T : class` (and then it is *reference* equality, ignoring `Equals` overloads).
- Generic `new T()` compiles to `Activator.CreateInstance<T>()`, which is slower than a direct `new` (and swallows constructor exceptions inside `TargetInvocationException` in older runtimes).

**Interview traps**
- Constraints are not part of the method signature -> cannot overload on them.
- `where T : class` does not give you `==` value semantics; it is reference equality.
- `new()` constraint = only *public parameterless* ctor; you cannot constrain ctor with arguments.
- `T?` means different things for `struct` and `class` constraints.
- Static members are **per closed generic type**, not shared across `T`.

**Cheat sheet**
- Generics = type safety + no boxing + one definition.
- Constraint order: `class`/`struct` first, `new()` last.
- Need operators? `INumber<T>` (C# 11 generic math).
- Equality in generics: `EqualityComparer<T>.Default`.
- `default(T)`: null/0.

---

## Topic 2: Generics at runtime (reified, JIT sharing)

**Study notes**

- .NET generics are **reified**: the CLR keeps the type arguments at runtime (unlike Java's erasure). `List<int>` and `List<string>` are distinct runtime types; `typeof(T)` works; `obj is List<int>` works.
- The IL contains the **open** generic definition (`List<T>`); the CLR constructs **closed** types (`List<int>`) at load/JIT time.
- JIT strategy:

```
 Instantiation                     Native code
 ---------------------------------------------------------------
 List<int>, List<double>, List<MyStruct>   -> separate specialised code per value type
 List<string>, List<Foo>, List<object>     -> ONE shared code body (canonical form "__Canon")
                                              + runtime "generic dictionary" for type handles
```
- Each closed generic type has its **own static fields** and static constructor run.
- Value-type specialisation avoids boxing and enables inlining; shared reference-type code is possible because all references are pointer-sized.
- Open types: `typeof(List<>)`; closed: `typeof(List<int>)`. Reflection: `MakeGenericType`, `MakeGenericMethod`.
- Generic type + AOT (NativeAOT/iOS) may need pre-instantiation; `MakeGenericType` over value types can fail under AOT.

---

### 2.1 [Easy] What does it mean that .NET generics are "reified"? How does it differ from Java?

**Answer**
The type arguments exist at runtime, so `List<int>` and `List<string>` are different types and you can use `typeof(T)`, `is T`, `new T[]`. Java erases generics at compile time to `Object`, so that information is lost.

**Explanation**
Consequences in .NET: no boxing for `List<int>`, `typeof(List<int>) != typeof(List<string>)`, reflection can see `T`, and you can create `T[]`. Java needs boxed `Integer` and cannot do `new T[]` or `instanceof List<String>`.

**Code**
```csharp
Console.WriteLine(typeof(List<int>) == typeof(List<string>)); // Output: False
Console.WriteLine(typeof(List<int>).GetGenericTypeDefinition() == typeof(List<>)); // Output: True

static string Describe<T>() => typeof(T).Name;
Console.WriteLine(Describe<Dictionary<string, int>>()); // Output: Dictionary`2
object o = new List<int>();
Console.WriteLine(o is List<int>);   // Output: True
Console.WriteLine(o is List<long>);  // Output: False
```

### 2.2 [Follow-up] Is a static field in `Foo<T>` shared between `Foo<int>` and `Foo<string>`?

**Answer**
No. Each closed constructed type has its own static fields and its own static constructor execution.

**Explanation**
`Foo<int>` and `Foo<string>` are different runtime types, so statics are separate. This is often used for per-type caches (e.g. `EqualityComparer<T>.Default`, serializers). It is a trap when someone expects a global counter.

**Code**
```csharp
class Counter<T> { public static int Count; }

Counter<int>.Count++;
Counter<int>.Count++;
Counter<string>.Count++;
Console.WriteLine(Counter<int>.Count);    // Output: 2
Console.WriteLine(Counter<string>.Count); // Output: 1
```

### 2.3 [Medium] How does the JIT compile generics - is code duplicated for every `T`?

**Answer**
For value-type arguments the JIT generates specialised native code per instantiation. For all reference-type arguments it generates a single shared body (`__Canon`) that fetches type info from a runtime generic dictionary.

**Explanation**
Value types differ in size/layout, so each needs its own code (that is why `List<int>` is fast and has no boxing). References are all pointer-sized so one body serves `List<string>`, `List<Foo>`. The trade-off: shared code may need an extra dictionary lookup for `typeof(T)`, `new T()`, casts - slightly slower than specialised code. Many specialised instantiations of large generic methods increase code size/startup (code bloat).

**Code**
```csharp
// Observable consequence: no boxing allocation for value-type generics
long before = GC.GetAllocatedBytesForCurrentThread();
var list = new List<int>(1000);
for (int i = 0; i < 1000; i++) list.Add(i);
long after = GC.GetAllocatedBytesForCurrentThread();
Console.WriteLine(after - before < 5000); // Output: True (one int[1000] ~4 KB, no per-item boxes)
```

### 2.4 [Hard] Why is a generic constraint to an interface on a struct type not boxing, while passing it as the interface type is?

**Answer**
When `T : IFoo` is instantiated with a struct, the JIT specialises the code and makes a *constrained call* directly on the struct, so no boxing occurs. Assigning the struct to a variable of type `IFoo` boxes it.

**Explanation**
This is why `List<T>.Sort` with `IComparable<T>` on structs, `EqualityComparer<T>.Default`, and "struct-generic" tricks (zero-cost abstractions via `where TComparer : struct, IComparer<T>`) are fast: the JIT sees the concrete struct type and can inline the call. IL uses the `constrained.` prefix.

**Code**
```csharp
interface IShape { double Area(); }
struct Square : IShape { public double S; public double Area() => S * S; }

static double ViaGeneric<T>(T shape) where T : IShape => shape.Area(); // no boxing
static double ViaInterface(IShape shape) => shape.Area();               // boxes the struct

var sq = new Square { S = 3 };
Console.WriteLine(ViaGeneric(sq));   // Output: 9
Console.WriteLine(ViaInterface(sq)); // Output: 9 (but allocated a box)
```

**Gotchas / follow-ups**
- Mutating a struct through the interface mutates the **box copy**, not the original.
- Reference-type instantiations share code, so the "no virtual call" benefit is only for value types.

### 2.5 [Tricky] Can you create `List<T>` for a `T` known only at runtime? What is the cost/risk?

**Answer**
Yes via `typeof(List<>).MakeGenericType(type)` and `Activator.CreateInstance`, or `MakeGenericMethod` for methods. Cost is reflection overhead and the JIT must build a new instantiation; the result is only usable through non-generic interfaces (`IList`) unless you call back into generic code via reflection.

**Explanation**
Typical use: serializers, DI containers (open-generic registrations like `IRepository<>`), mappers. Cache the constructed delegates/types. Under NativeAOT/trimming, instantiations over value types that were not statically seen may be missing and fail at runtime.

**Code**
```csharp
using System.Collections;

Type itemType = typeof(int);
Type listType = typeof(List<>).MakeGenericType(itemType);
var list = (IList)Activator.CreateInstance(listType)!;
list.Add(42);
Console.WriteLine(list.GetType());   // Output: System.Collections.Generic.List`1[System.Int32]
Console.WriteLine(list[0]);          // Output: 42
// list.Add("x"); // ArgumentException at runtime - type safety only enforced dynamically here
```

**Gotchas / follow-ups**
- Generic *methods* via `MethodInfo.MakeGenericMethod(...).Invoke(...)` wrap exceptions in `TargetInvocationException`.
- Constraints are verified at `MakeGenericType` time (throws `ArgumentException` if violated).

**Interview traps**
- "Generics are erased" is Java, not .NET.
- Statics are per closed type.
- Reference-type instantiations share code; value-type instantiations do not.
- Casting a struct to an interface boxes; generic constraint calls do not.

**Cheat sheet**
- Reified: type args live at runtime.
- Value types -> specialised JIT code; refs -> shared `__Canon`.
- `typeof(List<>)` = open, `typeof(List<int>)` = closed.
- Static field per closed type.
- `MakeGenericType` for runtime-known T.

---

## Topic 3: Covariance & contravariance (in/out)

**Study notes**

- **Variance** = whether `Generic<Derived>` can be used where `Generic<Base>` is expected (or vice-versa).
- Declared only on **interfaces and delegates**, via `out` (covariant) and `in` (contravariant). Classes/structs are always invariant (`List<T>` is invariant).
- Works **only for reference types** as arguments (identity-preserving conversion). `IEnumerable<int>` is *not* `IEnumerable<object>`.

```
 out T  (producer) : IEnumerable<Dog>  ->  IEnumerable<Animal>   OK   (T only in output positions)
 in  T  (consumer) : Action<Animal>    ->  Action<Dog>           OK   (T only in input positions)
 invariant         : List<Dog>         ->  List<Animal>          NO
```
- Built-ins: `IEnumerable<out T>`, `IEnumerator<out T>`, `IReadOnlyList<out T>`, `IReadOnlyCollection<out T>`, `IQueryable<out T>`, `Func<..., out TResult>`, `Action<in T>`, `Predicate<in T>`, `IComparer<in T>`, `IComparable<in T>`, `IEqualityComparer<in T>`, `IObservable<out T>`, `IObserver<in T>`. `IList<T>`, `ICollection<T>`, `IDictionary` are invariant (T is both in and out).
- **Arrays are covariant** (legacy of CLR design) but unsafe: writes are type-checked at runtime -> `ArrayTypeMismatchException`.

---

### 3.1 [Easy] What is covariance? Give an example.

**Answer**
Covariance lets you use a more derived type argument where a less derived one is expected, e.g. `IEnumerable<string>` assigned to `IEnumerable<object>`. It is safe because `T` is only returned (`out`), never accepted.

**Explanation**
Since `IEnumerable<out T>` only produces `T`, any consumer that reads `object` can happily read `string`.

**Code**
```csharp
IEnumerable<string> strings = new List<string> { "a", "b" };
IEnumerable<object> objects = strings;          // covariance
Console.WriteLine(objects.Count());             // Output: 2

// List<object> bad = new List<string>();       // compile error: List<T> is invariant
```

### 3.2 [Follow-up] What is contravariance? Which BCL types use it?

**Answer**
Contravariance reverses the direction: a handler of a base type can be used where a handler of a derived type is needed (`Action<object>` as `Action<string>`). It applies to `in` positions: `Action<T>`, `IComparer<T>`, `IEqualityComparer<T>`, `Predicate<T>`.

**Explanation**
A comparer that can compare any `Animal` can certainly compare two `Dog`s. That is why you can pass `IComparer<Animal>` to `List<Dog>.Sort(IComparer<Dog>)`.

**Code**
```csharp
class Animal { public string Name = ""; }
class Dog : Animal { }

IComparer<Animal> byName = Comparer<Animal>.Create((a, b) => string.Compare(a.Name, b.Name, StringComparison.Ordinal));
IComparer<Dog> dogComparer = byName;     // contravariance
var dogs = new List<Dog> { new() { Name = "Rex" }, new() { Name = "Ace" } };
dogs.Sort(dogComparer);
Console.WriteLine(dogs[0].Name);         // Output: Ace

Action<object> printAny = o => Console.WriteLine(o);
Action<string> printStr = printAny;      // contravariance
printStr("hi");                          // Output: hi
```

### 3.3 [Medium] Why is `IList<T>` invariant but `IReadOnlyList<T>` covariant? Why doesn't variance work with `int`?

**Answer**
`IList<T>` has both input (`Add(T)`, setter) and output (`this[i]` getter) positions, so neither variance is type-safe. `IReadOnlyList<T>` only outputs `T`. Variance requires a reference conversion that preserves the representation; `int` -> `object` needs boxing, which is a representation change, so it's not allowed.

**Explanation**
If `IList<Dog>` converted to `IList<Animal>`, you could `Add(new Cat())` into a list of dogs. The compiler checks variance annotations at declaration: using `out T` in a parameter position is a compile error (CS1961).

**Code**
```csharp
IReadOnlyList<string> ro = new List<string> { "x" };
IReadOnlyList<object> ro2 = ro;                     // OK
Console.WriteLine(ro2[0]);                          // Output: x

IEnumerable<int> ints = new[] { 1, 2 };
// IEnumerable<object> o = ints;                    // compile error (value type)
IEnumerable<object> o = ints.Cast<object>();        // works via boxing each element
Console.WriteLine(o.Count());                       // Output: 2
```

### 3.4 [Hard] Why are arrays covariant and what's the runtime danger?

**Answer**
Arrays were made covariant in CLR 1.0 before generics existed (to allow `Array.Sort(object[])` on `string[]`). The compiler lets `object[] a = new string[1]`, but each store performs a runtime type check and throws `ArrayTypeMismatchException` if the element isn't compatible.

**Explanation**
Safe reading, unsafe writing. The check also costs performance on reference-type array stores (covariance check, partially elided for sealed types like `string`). `readonly struct` arrays and value-type arrays are not covariant.

**Code**
```csharp
object[] arr = new string[2];
arr[0] = "ok";
try { arr[1] = 42; }
catch (ArrayTypeMismatchException) { Console.WriteLine("boom"); } // Output: boom
```

**Gotchas / follow-ups**
- `int[]` is **not** an `object[]`, but `int[]` -> `uint[]` casts work at CLR level (quirk) - don't rely on it.
- `ImmutableArray<T>`/`ReadOnlySpan<T>` avoid this issue.

### 3.5 [Tricky] Can a class be variant? Can `out T` appear in a method parameter?

**Answer**
Classes cannot declare variance. `out T` cannot appear as a plain method parameter type, but it can appear inside a parameter of type `Action<T>` because contravariance applied twice yields covariance. Variance also only applies to reference-type arguments.

**Explanation**
Variance is checked on the whole interface: `interface IProducer<out T> { void Visit(Action<T> a); }` is legal because `T` is in a contravariant-of-contravariant (i.e. covariant) position. `ref`/`out` parameters make a type parameter invariant. Variant conversions with value-type arguments never exist.

**Code**
```csharp
interface IProducer<out T>
{
    T Produce();
    void Visit(Action<T> visitor);       // legal: T flips twice
}
// interface IBad<out T> { void Set(T t); }   // error CS1961: invalid variance

class StrProducer : IProducer<string>
{
    public string Produce() => "p";
    public void Visit(Action<string> v) => v("visited");
}

IProducer<object> p = new StrProducer();
Console.WriteLine(p.Produce());          // Output: p
p.Visit(o => Console.WriteLine(o));      // Output: visited
```

**Gotchas / follow-ups**
- Overload resolution with variant types can become ambiguous when a class implements both `IFoo<Dog>` and `IFoo<Cat>` and you convert to `IFoo<Animal>`.
- `in`/`out` on a generic *method* type parameter does not exist - only on interface/delegate declarations.

**Interview traps**
- "Is `List<Dog>` a `List<Animal>`?" -> No (invariant class).
- Array covariance compiles but throws at runtime on write.
- No variance for value types (`IEnumerable<int>` -> `IEnumerable<object>` fails).
- `IList<T>` invariant, `IReadOnlyList<T>` covariant.

**Cheat sheet**
- `out` = producer = covariant; `in` = consumer = contravariant.
- Only interfaces & delegates; only reference types.
- `Func<out R>`, `Action<in T>`, `IEnumerable<out T>`, `IComparer<in T>`.
- Arrays: covariant but runtime-checked.
- Both in & out usage -> invariant.

---

## Topic 4: Array vs List<T>, growth, Span/Memory intro

**Study notes**

- **Array** `T[]`: fixed length, contiguous, CLR-native, covariant, fastest indexing, `Length` property, can be multi-dimensional/jagged.
- **`List<T>`**: wrapper around a `T[] _items` + `int _size` + `int _version`. `Capacity` = `_items.Length`; `Count` = `_size`.
- **Growth**: default ctor -> empty shared array (capacity 0). First `Add` -> capacity **4**; afterwards capacity **doubles** (4, 8, 16, ...) by allocating a new array and copying (`Array.Copy`). Amortised O(1) `Add`; worst-case O(n) on the resize. `Insert`/`RemoveAt` are O(n) (shifting).
- Pre-size with `new List<T>(n)` or `EnsureCapacity(n)` (.NET 6+); `TrimExcess()` shrinks; `CollectionsMarshal.AsSpan(list)` gives a span over the internal array (.NET 5+).
- Large arrays (>= 85,000 bytes) go on the **Large Object Heap**; repeated doubling of big lists causes LOH churn.
- **`Span<T>`**: `ref struct` view over contiguous memory (array, stackalloc, native). No allocation when slicing. Cannot be a class field, boxed, captured in lambdas, or used across `await`. **`Memory<T>`** is the heap-storable counterpart (`.Span` to access).

```
List<int> growth:   cap 0 -> 4 -> 8 -> 16 -> 32 ...
 _items: [1][2][3][4]   Add(5) => allocate int[8], copy 4, add -> [1][2][3][4][5][ ][ ][ ]
```

---

### 4.1 [Easy] Array vs `List<T>` - when would you choose each?

**Answer**
Arrays are fixed-size and slightly faster, good for known-size data, interop, and hot paths. `List<T>` resizes automatically and has rich APIs (`Add`, `Remove`, `Sort`, `Find`), so it's the default for general collections.

**Explanation**
`List<T>` is backed by an array so reading is nearly as fast; the indexer adds a bounds + version-free check. Arrays implement `IList<T>` explicitly, so `Add` throws `NotSupportedException` (fixed size).

**Code**
```csharp
int[] arr = new int[3] { 1, 2, 3 };
var list = new List<int>(arr);
list.Add(4);
Console.WriteLine(arr.Length);   // Output: 3
Console.WriteLine(list.Count);   // Output: 4

IList<int> asList = arr;
try { asList.Add(9); }
catch (NotSupportedException) { Console.WriteLine("fixed size"); } // Output: fixed size
```

### 4.2 [Follow-up] How does `List<T>` grow? What are `Count` vs `Capacity`?

**Answer**
`Count` is the number of items; `Capacity` is the length of the internal array. When `Count == Capacity` on `Add`, a new array of double size (4 if currently 0) is allocated and elements copied.

**Explanation**
Doubling gives amortised O(1) appends. If you know the size, `new List<T>(n)` avoids all intermediate copies and garbage. `Clear()` does not reduce capacity; `TrimExcess()` only trims if it saves >10%.

**Code**
```csharp
var l = new List<int>();
Console.WriteLine(l.Capacity);   // Output: 0
l.Add(1);
Console.WriteLine(l.Capacity);   // Output: 4
for (int i = 0; i < 4; i++) l.Add(i);   // Count 5 -> resize
Console.WriteLine(l.Capacity);   // Output: 8
l.Clear();
Console.WriteLine($"{l.Count} {l.Capacity}"); // Output: 0 8
```

### 4.3 [Medium] Why is `Insert(0, x)` / `Remove(x)` slow and what alternatives exist?

**Answer**
They shift all subsequent elements in the backing array: O(n). Alternatives: `Queue<T>`/`Deque`-like structure (`LinkedList<T>`, circular buffer), `HashSet<T>` for membership removal, or swap-with-last removal when order doesn't matter.

**Explanation**
`RemoveAll(pred)` is O(n) total (one compaction pass) and is far better than a loop of `Remove`. `Contains`/`IndexOf` are O(n) linear searches; use `HashSet<T>` or `Dictionary` for lookups.

**Code**
```csharp
var items = new List<int> { 1, 2, 3, 4, 5, 6 };
items.RemoveAll(x => x % 2 == 0);          // single pass
Console.WriteLine(string.Join(",", items)); // Output: 1,3,5

// O(1) unordered remove: swap with last
static void RemoveUnordered<T>(List<T> l, int i) { l[i] = l[^1]; l.RemoveAt(l.Count - 1); }
RemoveUnordered(items, 0);
Console.WriteLine(string.Join(",", items)); // Output: 5,3
```

### 4.4 [Hard] What is `Span<T>`, how is it different from `Memory<T>`, and why can't it be used in async methods?

**Answer**
`Span<T>` is a stack-only `ref struct` that points to a contiguous region without copying (arrays, `stackalloc`, native memory, string chars via `ReadOnlySpan<char>`). `Memory<T>` is a normal struct that can live on the heap and be stored in fields. Spans can't cross `await` because async state machines are heap-allocated classes/structs that would have to store the ref struct.

**Explanation**
`ref struct` rules: cannot be boxed, cannot be a field of a class/non-ref struct, cannot be captured by lambda/local function used as closure, cannot implement interfaces (until C# 13 allows it with `allows ref struct`), cannot be a generic argument (before C# 13). Slicing `span[1..3]` creates a new view in O(1) with no allocation, which is why parsers (`int.Parse(ReadOnlySpan<char>)`) are allocation-free.

**Code**
```csharp
int[] data = { 1, 2, 3, 4, 5 };
Span<int> slice = data.AsSpan(1, 3);   // view of {2,3,4}
slice[0] = 99;                          // writes through to the array
Console.WriteLine(string.Join(",", data)); // Output: 1,99,3,4,5

ReadOnlySpan<char> text = "key=value";
int eq = text.IndexOf('=');
Console.WriteLine(text[..eq].ToString());      // Output: key
Console.WriteLine(text[(eq + 1)..].ToString()); // Output: value

Span<byte> stack = stackalloc byte[16];        // no heap allocation
Console.WriteLine(stack.Length);               // Output: 16
```

**Gotchas / follow-ups**
- `stackalloc` of large sizes can overflow the stack; keep it small (e.g. <= 1 KB) or fall back to `ArrayPool<T>`.
- In an `async` method use `Memory<T>`; call `.Span` inside a synchronous section.

### 4.5 [Tricky] What happens to a `Span` obtained from `CollectionsMarshal.AsSpan(list)` if the list grows?

**Answer**
The span keeps pointing at the **old** backing array, which the list no longer uses, so writes through it are silently lost (and the data is stale). It's valid only until the list is resized/modified.

**Explanation**
`CollectionsMarshal.AsSpan` exposes the internal buffer for zero-copy access (e.g. bulk fill). There is no version check, so structural changes make the span a dangling-but-valid reference to the previous array (memory-safe, but logically wrong).

**Code**
```csharp
using System.Runtime.InteropServices;

var list = new List<int> { 1, 2, 3, 4 };       // capacity 4
Span<int> span = CollectionsMarshal.AsSpan(list);
list.Add(5);                                    // resize -> new array
span[0] = 100;                                  // writes to the OLD array
Console.WriteLine(list[0]);                     // Output: 1
```

**Gotchas / follow-ups**
- `CollectionsMarshal.SetCount(list, n)` (.NET 8) changes `Count` without initialising - use with care.
- Modifying list while enumerating still throws; spans bypass the check.

**Interview traps**
- `new List<int>()` has capacity **0**, first add gives **4**; doubling afterwards.
- `Capacity` isn't `Count`; `Clear()` keeps capacity.
- Arrays are covariant and mutable; `Array.Resize` creates a *new* array.
- `Span<T>` cannot be used in async methods, iterators, or captured by lambdas.

**Cheat sheet**
- `List<T>` = array + count + version.
- Growth: 0 -> 4 -> x2; pre-size with capacity ctor.
- `Insert`/`RemoveAt` O(n); `Add` amortised O(1).
- `Span<T>` = stack-only view; `Memory<T>` = heap-safe view.
- Arrays >= 85,000 bytes -> LOH.

---

## Topic 5: Dictionary, HashSet, Sorted*, Stack/Queue, LinkedList

**Study notes**

- **`Dictionary<TKey,TValue>`** - hash table with **separate chaining by index** (not linked node objects):

```
 buckets : int[]   (size = prime)   bucket[h % size] = (index of first entry) + 1   (0 = empty)
 entries : Entry[] { int hashCode; int next; TKey key; TValue value }   next = index of next entry in chain (-1 end)
 freeList: removed entries are chained for reuse

 Add(k): h = comparer.GetHashCode(k); b = h % buckets.Length
         walk chain: compare hashCode && Equals(key) -> duplicate => throw / overwrite
         else put in entries[count++] (or freeList), entry.next = old bucket head, bucket = new index
 Resize: when count == entries.Length -> new capacity = next prime >= 2*old; all entries re-bucketed (O(n))
```
- Lookup O(1) average; O(n) worst case if many collisions (all keys same hash). Keys must have stable `GetHashCode`/`Equals` while in the dictionary. `Dictionary` is **not** thread-safe for writes; enumeration order is unspecified (in practice insertion order if no removals, but never rely on it).
- `HashSet<T>`: same bucket/entries design, only keys; `Add` returns `bool`; set ops (`UnionWith`, `IntersectWith`, `ExceptWith`, `IsSubsetOf`).
- **`SortedDictionary<K,V>`**: red-black tree, O(log n) add/remove/lookup, keys enumerated in order. **`SortedList<K,V>`**: two sorted arrays; O(log n) lookup, O(n) insert, less memory, indexable by position. `SortedSet<T>`: tree-based set.
- **`Stack<T>`** (LIFO, array-backed, `Push/Pop/Peek`), **`Queue<T>`** (FIFO, circular array, `Enqueue/Dequeue/Peek`), both O(1) amortised.
- **`LinkedList<T>`**: doubly linked, node objects (`LinkedListNode<T>`); O(1) insert/remove *given a node*; O(n) find; no index; poor cache locality, one allocation per node.
- `PriorityQueue<TElement,TPriority>` (.NET 6): binary heap, min-priority first, **not stable**, no update-priority.

---

### 5.1 [Easy] How does a `Dictionary` find a value by key?

**Answer**
It calls `GetHashCode()` on the key, maps it to a bucket (hash modulo bucket count), then walks the short chain of entries in that bucket comparing hash codes and `Equals`. Average O(1).

**Explanation**
The bucket holds an index into the entries array, and each entry stores `next`, so collisions form an index-linked chain with no per-entry allocation. A good hash function spreads keys across buckets.

**Code**
```csharp
var ages = new Dictionary<string, int> { ["Ann"] = 30, ["Bob"] = 25 };
Console.WriteLine(ages["Ann"]);                 // Output: 30
Console.WriteLine(ages.TryGetValue("Zed", out var v)); // Output: False
Console.WriteLine(ages.GetValueOrDefault("Zed", -1));  // Output: -1
try { _ = ages["Zed"]; } catch (KeyNotFoundException) { Console.WriteLine("missing"); } // Output: missing
```

### 5.2 [Follow-up] What happens on a hash collision, and when/how does it resize?

**Answer**
Colliding keys share a bucket and form a chain through `entries[i].next`; lookups compare the full hash then `Equals` along the chain. When the entries array is full, the dictionary allocates a larger array (about double, rounded to the next prime) and re-buckets every entry.

**Explanation**
Resize is O(n), amortised O(1) per add. Pre-size with `new Dictionary<K,V>(capacity)` to avoid it. With adversarial/poor hashes everything lands in one bucket -> O(n) lookups. For `string` keys with the default comparer .NET starts with a fast non-randomised hash and switches to randomised hashing when it detects too many collisions (hash-flooding defence).

**Code**
```csharp
class BadKey(int id)
{
    public int Id = id;
    public override int GetHashCode() => 1;                 // everything collides
    public override bool Equals(object? o) => o is BadKey b && b.Id == Id;
}

var d = new Dictionary<BadKey, int>();
for (int i = 0; i < 1000; i++) d[new BadKey(i)] = i;        // each add walks a growing chain: O(n^2)
Console.WriteLine(d[new BadKey(500)]);                      // Output: 500 (but O(n) lookup)
```

### 5.3 [Medium] What are the rules for a type used as a Dictionary key? What if the key is mutated?

**Answer**
`Equals` and `GetHashCode` must be consistent (equal objects -> equal hash) and the hash must not change while the key is in the dictionary. If a key's hash-affecting fields are mutated, lookups look in the wrong bucket and the entry becomes effectively lost.

**Explanation**
Prefer immutable keys: `string`, `int`, `Guid`, `record`, `readonly record struct`, `ValueTuple`. For custom classes override both members (use `HashCode.Combine`). For case-insensitive strings pass `StringComparer.OrdinalIgnoreCase`. Value-type keys without overriding `Equals`/`GetHashCode` use reflection-based/slow default `ValueType.Equals` in some cases - implement `IEquatable<T>`.

**Code**
```csharp
class Mutable { public int X; public override int GetHashCode() => X; public override bool Equals(object? o) => o is Mutable m && m.X == X; }

var k = new Mutable { X = 1 };
var d = new Dictionary<Mutable, string> { [k] = "v" };
k.X = 2;                                                     // hash changed after insertion
Console.WriteLine(d.ContainsKey(k));                         // Output: False
Console.WriteLine(d.ContainsKey(new Mutable { X = 1 }));     // Output: False (stored key now has X=2 but sits in the X=1 bucket)
Console.WriteLine(d.Count);                                  // Output: 1 (orphaned entry)

var ci = new Dictionary<string, int>(StringComparer.OrdinalIgnoreCase) { ["Key"] = 1 };
Console.WriteLine(ci["KEY"]);                                // Output: 1
```

### 5.4 [Hard] Compare `Dictionary`, `SortedDictionary`, `SortedList`, and `HashSet`/`SortedSet`. When use which?

**Answer**
`Dictionary` for fastest O(1) lookups with no order. `SortedDictionary` for ordered iteration with frequent inserts/removals (O(log n), tree). `SortedList` for mostly-static sorted data, lowest memory and index access, but O(n) inserts. `HashSet` for fast membership/uniqueness; `SortedSet` when you also need order/range views.

**Explanation**

| Type | Lookup | Insert | Memory | Ordered | Notes |
|---|---|---|---|---|---|
| Dictionary | O(1) avg | O(1) amortised | medium | no | resize cost |
| SortedDictionary | O(log n) | O(log n) | high (nodes) | yes | red-black tree |
| SortedList | O(log n) | O(n) | low | yes | arrays, `Keys[i]` |
| HashSet | O(1) | O(1) | medium | no | set ops |
| SortedSet | O(log n) | O(log n) | high | yes | `GetViewBetween` |
| LinkedList | O(n) | O(1) w/ node | high | no | no index |

**Code**
```csharp
var sd = new SortedDictionary<int, string> { [3] = "c", [1] = "a", [2] = "b" };
Console.WriteLine(string.Join(",", sd.Keys));    // Output: 1,2,3

var set = new HashSet<int> { 1, 2, 3 };
Console.WriteLine(set.Add(3));                   // Output: False
set.IntersectWith(new[] { 2, 3, 4 });
Console.WriteLine(string.Join(",", set.Order())); // Output: 2,3

var sorted = new SortedSet<int> { 5, 1, 9, 3 };
Console.WriteLine(string.Join(",", sorted.GetViewBetween(2, 6))); // Output: 3,5
```

**Gotchas / follow-ups**
- `Order()` is a LINQ method new in .NET 7.
- Neither `HashSet` nor `Dictionary` guarantees order even if it looks insertion-ordered; removals reuse slots.

### 5.5 [Tricky] Stack, Queue, LinkedList and PriorityQueue - internals and edge cases?

**Answer**
`Stack<T>` is an array (push/pop at the end), `Queue<T>` is a circular array with head/tail indices, `LinkedList<T>` is a doubly linked list of node objects, and `PriorityQueue` is a binary min-heap. Peeking/popping an empty `Stack`/`Queue` throws `InvalidOperationException` - use `TryPop`/`TryDequeue`.

**Explanation**
`Queue<T>` grows by doubling (min growth 4) and linearises on resize. `LinkedList<T>` is rarely faster in practice because of allocations and cache misses, even for "insert in the middle" unless you already hold the node. `PriorityQueue` dequeues the *smallest* priority; equal-priority order is unspecified.

**Code**
```csharp
var q = new Queue<int>(); q.Enqueue(1); q.Enqueue(2);
Console.WriteLine(q.Dequeue());              // Output: 1 (FIFO)

var s = new Stack<int>(); s.Push(1); s.Push(2);
Console.WriteLine(s.Pop());                  // Output: 2 (LIFO)
Console.WriteLine(new Stack<int>().TryPop(out _)); // Output: False

var pq = new PriorityQueue<string, int>();
pq.Enqueue("low", 10); pq.Enqueue("high", 1); pq.Enqueue("mid", 5);
Console.WriteLine(pq.Dequeue());             // Output: high

var ll = new LinkedList<int>(new[] { 1, 3 });
ll.AddAfter(ll.First!, 2);
Console.WriteLine(string.Join(",", ll));     // Output: 1,2,3
```

**Gotchas / follow-ups**
- Enumerating a `Stack<T>` yields LIFO order (top first).
- `Queue<T>.Remove` doesn't exist; use `Queue.Contains` (O(n)) or another structure.

**Interview traps**
- Mutating a key's hash-relevant state after insertion breaks the dictionary.
- Overriding `Equals` without `GetHashCode` -> broken lookups (and a compiler warning).
- `Dictionary` enumeration order is not guaranteed.
- `SortedList` inserts are O(n), `SortedDictionary` O(log n).
- Reading `dict[key]` throws; use `TryGetValue`.

**Cheat sheet**
- Dictionary = `int[] buckets` + `Entry[] entries` chained via `next`.
- Resize at full -> ~2x, prime-sized, rehash all.
- `Equals`/`GetHashCode` contract; immutable keys.
- Sorted: tree (log n) vs SortedList (array, O(n) insert).
- Stack/Queue O(1); LinkedList O(1) only with a node.

---

## Topic 6: IEnumerable vs ICollection vs IList vs IReadOnlyList

**Study notes**

```
IEnumerable<T>            : GetEnumerator()                     (forward-only iteration)
  |-- IReadOnlyCollection<T>  : + Count
  |     |-- IReadOnlyList<T>      : + this[int] (get)
  |-- ICollection<T>          : + Count, IsReadOnly, Add, Remove, Contains, Clear, CopyTo
        |-- IList<T>              : + this[int] get/set, IndexOf, Insert, RemoveAt
```
- `ICollection<T>` does **not** derive from `IReadOnlyCollection<T>`; `List<T>`, `T[]`, `HashSet<T>`, `Dictionary` implement both families separately.
- `IReadOnly*` means *read-only view*, **not immutable** - the underlying collection may still change. `IReadOnlyList<out T>` is covariant; `IList<T>` is not.
- `IEnumerable<T>` has no `Count` - LINQ `Count()` tries `ICollection<T>`/`IReadOnlyCollection<T>` first (O(1)) otherwise enumerates (O(n)).
- Guidance: **accept** the most general type you actually need (`IEnumerable<T>`/`IReadOnlyCollection<T>`); **return** the most useful read-only abstraction (`IReadOnlyList<T>`) or a concrete type for internal code.
- Truly immutable: `ImmutableArray<T>`, `ImmutableList<T>`, `FrozenSet<T>` (.NET 8). `List<T>.AsReadOnly()` returns `ReadOnlyCollection<T>` (a wrapper, live view).

---

### 6.1 [Easy] What's the difference between `IEnumerable<T>` and `List<T>`?

**Answer**
`IEnumerable<T>` is a minimal contract for forward-only iteration; `List<T>` is a concrete in-memory collection with indexing, add/remove, sorting. Any `List<T>` is an `IEnumerable<T>` but not vice-versa.

**Explanation**
An `IEnumerable<T>` can be lazily generated (e.g. a `yield` method, a database cursor) and may even be infinite, so you can't assume it has a `Count` or can be enumerated twice cheaply.

**Code**
```csharp
IEnumerable<int> Naturals() { int i = 0; while (true) yield return i++; }

Console.WriteLine(string.Join(",", Naturals().Take(3))); // Output: 0,1,2
List<int> l = Naturals().Take(3).ToList();               // materialised
Console.WriteLine(l.Count);                              // Output: 3
```

### 6.2 [Follow-up] ICollection vs IList vs IReadOnlyList - what does each add?

**Answer**
`ICollection<T>` adds `Count`, `Add`, `Remove`, `Contains`, `Clear`. `IList<T>` adds positional access (`[i]`, `Insert`, `RemoveAt`, `IndexOf`). `IReadOnlyList<T>` offers only `Count` and the indexer getter.

**Explanation**
Use `IReadOnlyList<T>` for public return values when callers need count/index but must not mutate. `HashSet<T>` is an `ICollection<T>` but not an `IList<T>` (no order/index). Dictionary is `ICollection<KeyValuePair<K,V>>`.

**Code**
```csharp
IReadOnlyList<int> Evens(int n) => Enumerable.Range(0, n).Where(x => x % 2 == 0).ToList();

var r = Evens(10);
Console.WriteLine($"{r.Count} {r[1]}");        // Output: 5 2
Console.WriteLine(new HashSet<int>() is IList<int>); // Output: False
Console.WriteLine(new int[0] is IList<int>);         // Output: True
```

### 6.3 [Medium] Is `IReadOnlyList<T>` immutable? How do callers still mutate it?

**Answer**
No, it's only a read-only *interface*. If the underlying object is a `List<T>`, a caller can downcast and mutate it, and the owner can still change it, which the "read-only" view reflects.

**Explanation**
To truly protect internal state: return `ReadOnlyCollection<T>` (wrapper; downcast gives the wrapper, not the list), return a copy (`ToArray()`), or use immutable collections (`ImmutableArray<T>`, `FrozenSet<T>`). Beware: defensive copying costs allocations.

**Code**
```csharp
class Bag
{
    private readonly List<int> _items = new() { 1, 2 };
    public IReadOnlyList<int> Leaky => _items;                 // downcast possible
    public IReadOnlyList<int> Safe => _items.AsReadOnly();     // wrapper
    public void Add(int x) => _items.Add(x);
}

var b = new Bag();
((List<int>)b.Leaky).Add(99);                                  // mutated the internal list!
Console.WriteLine(b.Leaky.Count);                              // Output: 3
Console.WriteLine(b.Safe is List<int>);                        // Output: False
b.Add(4);
Console.WriteLine(b.Safe.Count);                               // Output: 4 (live view)
```

### 6.4 [Hard] How does LINQ `Count()` / `ToList()` use these interfaces, and why does it matter for API design?

**Answer**
`Count()` checks if the source implements `ICollection<T>` (or `IReadOnlyCollection<T>`/internal iterators with known size) and returns the count in O(1); otherwise it enumerates fully. `ToList()`/`ToArray()` also use `ICollection<T>.CopyTo` to pre-size and copy fast.

**Explanation**
If you expose `IEnumerable<T>` backed by a lazy query, every `Count()` re-executes it. Exposing `IReadOnlyCollection<T>` documents that count is cheap and the data is already materialised. `Enumerable.TryGetNonEnumeratedCount` (.NET 6) lets you ask without enumerating.

**Code**
```csharp
IEnumerable<int> lazy = Enumerable.Range(1, 5).Where(x => x > 2);
Console.WriteLine(lazy.TryGetNonEnumeratedCount(out _)); // Output: False

IEnumerable<int> list = new List<int> { 1, 2, 3 };
Console.WriteLine(list.TryGetNonEnumeratedCount(out int n)); // Output: True
Console.WriteLine(n);                                        // Output: 3
```

**Gotchas / follow-ups**
- `Enumerable.Range(...)` alone reports a count without enumeration in .NET (internal optimisation), but `Where` on it does not.
- Never call `.Count()` in a loop condition on a lazy `IEnumerable`.

### 6.5 [Tricky] An API takes `List<T>` and you pass `T[]`; what about the reverse for `IEnumerable<T>`? How do arrays satisfy `IList<T>`?

**Answer**
`T[]` is not a `List<T>`, so that doesn't compile. Arrays implement `IList<T>`, `ICollection<T>`, `IReadOnlyList<T>` through the runtime (`SZArray` helper), with mutation methods (`Add`/`Remove`) throwing `NotSupportedException`. So accepting `IReadOnlyList<T>` or `IEnumerable<T>` works for both arrays and lists.

**Explanation**
This is a good reason to design parameters as interfaces. Note that `IList<T>.IsReadOnly` returns `true` for arrays, and `Array.IsFixedSize` is true. Setting an element via `IList<T>[i] = x` on an array *does* work.

**Code**
```csharp
static int Sum(IReadOnlyList<int> xs) { int s = 0; for (int i = 0; i < xs.Count; i++) s += xs[i]; return s; }

Console.WriteLine(Sum(new[] { 1, 2, 3 }));          // Output: 6
Console.WriteLine(Sum(new List<int> { 4, 5 }));     // Output: 9

IList<int> a = new[] { 1, 2 };
a[0] = 10;                                          // allowed
Console.WriteLine(a.IsReadOnly);                    // Output: True
try { a.Add(3); } catch (NotSupportedException) { Console.WriteLine("nope"); } // Output: nope
```

**Interview traps**
- `IReadOnlyList<T>` != immutable.
- `IEnumerable<T>.Count()` may be O(n) and may re-execute a query.
- `ICollection<T>` doesn't inherit `IReadOnlyCollection<T>`; types needing both implement both.
- Accepting `List<T>` in public APIs forces callers to convert.

**Cheat sheet**
- Param: `IEnumerable<T>`; return: `IReadOnlyList<T>`.
- `IList<T>` = index + mutation; `IReadOnlyList<T>` = index only, covariant.
- Arrays implement `IList<T>` (fixed size).
- Immutable truly: `ImmutableArray<T>`, `FrozenSet<T>`.
- `TryGetNonEnumeratedCount` avoids hidden enumeration.

---

## Topic 7: IEnumerable vs IQueryable vs IAsyncEnumerable

**Study notes**

```
IEnumerable<T>   : in-memory, LINQ-to-Objects, operators take Func<>       -> delegates run in your process
IQueryable<T>    : provider-based (EF Core), operators take Expression<Func<>> -> tree translated (e.g. to SQL)
IAsyncEnumerable<T> : async pull stream  (await foreach)  -> items produced with await between them
```
- `IQueryable<T>` : `IEnumerable<T>` and exposes `Expression`, `ElementType`, `Provider`. Calling `Where(x => ...)` on it records a `MethodCallExpression` into the tree; **nothing executes until enumerated** (`foreach`, `ToList`, `First`, `Count`...).
- Overload resolution decides: `IQueryable<Order>.Where(o => o.Total > 5)` binds to `Queryable.Where(Expression<Func<>>)` -> SQL `WHERE`. Calling `AsEnumerable()` (or `ToList()`) first switches to `Enumerable.Where(Func<>)` -> filtering happens in memory after fetching **all** rows.
- Things not translatable to SQL (custom C# methods) throw at runtime in EF Core (or, for older client-eval, silently pulled data).
- `IAsyncEnumerable<T>` (C# 8 / .NET Core 3.0): produced by `async` iterator methods (`async IAsyncEnumerable<T> M() { ... await ...; yield return x; }`) and consumed by `await foreach`. EF Core: `AsAsyncEnumerable()`, and `ToListAsync`/`FirstAsync` extension methods exist for `IQueryable`. LINQ for async streams: `System.Linq.Async` package (in .NET 10 is built-in; not in 8).
- Cancellation: `[EnumeratorCancellation] CancellationToken` parameter + `.WithCancellation(token)`; `ConfigureAwait(false)` on `await foreach` through `.ConfigureAwait`.

---

### 7.1 [Easy] What is the difference between `IEnumerable<T>` and `IQueryable<T>`?

**Answer**
`IEnumerable<T>` runs LINQ in memory using delegates. `IQueryable<T>` builds an expression tree that a provider (EF Core) translates to another language such as SQL, so filtering/sorting happens at the source.

**Explanation**
With `IEnumerable`, `db.Orders.AsEnumerable().Where(...)` pulls every row, then filters in .NET. With `IQueryable`, the `WHERE` goes into the SQL and only matching rows travel over the network.

**Code**
```csharp
// Pseudocode with EF Core DbContext `db`:
// IQueryable<Order> q = db.Orders.Where(o => o.Total > 100);   // SQL: WHERE Total > 100
// IEnumerable<Order> e = db.Orders.AsEnumerable().Where(o => o.Total > 100); // SELECT * then filter in memory

// Runnable in-memory demo of the type difference:
IQueryable<int> queryable = new[] { 1, 2, 3 }.AsQueryable().Where(x => x > 1);
Console.WriteLine(queryable.Expression);   // Output: System.Int32[].Where(x => (x > 1))
Console.WriteLine(string.Join(",", queryable)); // Output: 2,3
```

### 7.2 [Follow-up] When does an `IQueryable` actually execute? What is the "deferred execution" risk with a DbContext?

**Answer**
On enumeration or a materialising operator (`ToList`, `First`, `Count`, `Any`, `foreach`). The risk: returning an `IQueryable` out of a method whose `DbContext` was disposed, or enumerating it several times (running the SQL each time).

**Explanation**
Returning `IQueryable` from repositories lets callers compose further filters (good for flexibility, leaks persistence concerns). Returning materialised `List<T>`/`IReadOnlyList<T>` after `await ToListAsync()` bounds the DB work and context lifetime.

**Code**
```csharp
// Pseudocode:
// IQueryable<User> Users() { using var db = new AppDb(); return db.Users; }  // BUG: disposed on use
// var list = Users().ToList();   // ObjectDisposedException

// Safe pattern
// async Task<IReadOnlyList<User>> GetAdultsAsync(AppDb db, CancellationToken ct)
//     => await db.Users.Where(u => u.Age >= 18).ToListAsync(ct);
Console.WriteLine("see comments");   // Output: see comments
```

### 7.3 [Medium] Why does `Where(x => MyMethod(x))` behave differently on `IEnumerable` vs `IQueryable`?

**Answer**
On `IEnumerable` the lambda is compiled to a delegate and can call anything. On `IQueryable` it becomes an expression tree that the provider must translate; a custom C# method has no SQL equivalent, so EF Core throws `InvalidOperationException` at execution (unless it is the final projection, where client evaluation is allowed).

**Explanation**
EF Core 3.0+ removed silent client evaluation in `Where`; only the top-level `Select` projection may run client-side. Fixes: use translatable members (`EF.Functions.Like`, string methods EF maps), map custom functions with `HasDbFunction`, or call `AsEnumerable()` deliberately after filtering as much as possible in SQL.

**Code**
```csharp
static bool IsGood(int x) => x % 2 == 0;

// In memory: fine.
Console.WriteLine(string.Join(",", new[] { 1, 2, 3, 4 }.Where(IsGood)));   // Output: 2,4

// Method-group on AsQueryable: LINQ-to-Objects provider can execute it, SQL providers cannot.
var q = new[] { 1, 2, 3, 4 }.AsQueryable().Where(x => IsGood(x));
Console.WriteLine(q.Count());                                             // Output: 2
```

### 7.4 [Hard] How do `yield`-based `IAsyncEnumerable<T>` streams work and how do you cancel them?

**Answer**
An `async` method returning `IAsyncEnumerable<T>` with `yield return` is compiled to a state machine implementing `IAsyncEnumerable<T>`/`IAsyncEnumerator<T>`. `await foreach` calls `MoveNextAsync()` (a `ValueTask<bool>`) per item and `DisposeAsync()` at the end. Cancellation flows via a `[EnumeratorCancellation] CancellationToken` parameter supplied by `WithCancellation`.

**Explanation**
Streaming avoids buffering the whole result (good for large result sets or paged APIs, e.g. EF Core `AsAsyncEnumerable()`, ASP.NET Core minimal APIs/controllers serialising an `IAsyncEnumerable` incrementally). It holds the connection/resources open while consumers are slow, so don't mix with long processing per item without care.

**Code**
```csharp
static async IAsyncEnumerable<int> Numbers(
    int count,
    [System.Runtime.CompilerServices.EnumeratorCancellation] CancellationToken ct = default)
{
    for (int i = 0; i < count; i++)
    {
        await Task.Delay(10, ct);
        yield return i;
    }
}

using var cts = new CancellationTokenSource();
await foreach (var n in Numbers(100).WithCancellation(cts.Token))
{
    Console.WriteLine(n);              // Output: 0 1 2 (each on its own line)
    if (n == 2) cts.Cancel();          // next MoveNextAsync's Task.Delay throws OperationCanceledException
    if (n == 2) break;
}
```

**Gotchas / follow-ups**
- Without `[EnumeratorCancellation]`, the token you pass to `WithCancellation` never reaches the method body.
- `IAsyncEnumerable<T>` has no built-in LINQ in .NET 8; use `System.Linq.Async` (NuGet) or write loops.

### 7.5 [Tricky] What's wrong with: `IEnumerable<Order> orders = db.Orders; var recent = orders.Where(o => o.Date > cutoff).ToList();`?

**Answer**
Assigning `db.Orders` (a `DbSet<Order>`, an `IQueryable`) to an `IEnumerable<Order>` variable makes the later `Where` bind to `Enumerable.Where`, so EF loads the whole table and filters in memory. The *static type* picks the extension method.

**Explanation**
This is one of the most common silent EF performance bugs: nothing fails, you just transfer far too much data. Keep the variable typed as `IQueryable<Order>` (or use `var`) until the final materialisation.

**Code**
```csharp
IQueryable<int> source = new[] { 1, 2, 3 }.AsQueryable();

IEnumerable<int> asEnum = source;                       // static type now IEnumerable
var a = asEnum.Where(x => x > 1);                       // Enumerable.Where (in-memory delegate)
var b = source.Where(x => x > 1);                       // Queryable.Where (expression tree)

Console.WriteLine(b is IQueryable<int>);                 // Output: True
Console.WriteLine(a is IQueryable<int>);                 // Output: False (in-memory iterator)
```

**Gotchas / follow-ups**
- Same trap with `Func<T,bool>` vs `Expression<Func<T,bool>>` parameters in repository methods: a `Func` parameter forces in-memory filtering.
- `AsEnumerable()` is the *intentional* switch; `ToList()` is a materialisation + switch.

**Interview traps**
- Static type of the variable chooses `Enumerable` vs `Queryable`.
- Returning `IQueryable` from a disposed-context scope.
- Re-enumerating an `IQueryable` re-runs SQL.
- `[EnumeratorCancellation]` needed for `WithCancellation`.

**Cheat sheet**
- IEnumerable = delegates, in memory.
- IQueryable = expression tree -> provider (SQL).
- `AsEnumerable()` = switch to client-side.
- `IAsyncEnumerable` + `await foreach` = async streaming.
- Materialise with `ToListAsync` within context lifetime.

---

## Topic 8: Iterators, yield, IEnumerator & foreach internals

**Study notes**

- `yield return` / `yield break` make the compiler generate a **state-machine class** that implements `IEnumerable<T>` and `IEnumerator<T>` (and `IDisposable`). Locals become fields; a `<>1__state` int tracks where `MoveNext()` resumes.

```
 IEnumerable<int> Gen() { yield return 1; yield return 2; }
 compiles to (conceptually):
   class <Gen>d__0 : IEnumerable<int>, IEnumerator<int> {
      int <>1__state; int <>2__current;
      bool MoveNext() { switch(state){ case 0: current=1; state=1; return true;
                                       case 1: current=2; state=2; return true;
                                       default: return false; } }
   }
```
- Execution is **lazy**: the method body doesn't start until the first `MoveNext()`; argument validation inside an iterator is therefore deferred - split into a public checker + private iterator (or local function).
- Restrictions: no `ref`/`out` params, no `yield` inside `try` with a `catch` or inside a `finally`, no `unsafe`. `try/finally` is allowed (finally runs on `Dispose`, e.g. `foreach` early exit).
- **`foreach` is pattern-based**: the compiler looks for a public `GetEnumerator()` returning something with `MoveNext()` and `Current` (does not require `IEnumerable`). It expands to:

```csharp
var e = coll.GetEnumerator();
try { while (e.MoveNext()) { var x = e.Current; ... } }
finally { (e as IDisposable)?.Dispose(); }
```
- `List<T>.GetEnumerator()` returns a **struct** `List<T>.Enumerator` (no allocation when used via `foreach` on `List<T>`; boxed if accessed as `IEnumerable<T>`). Arrays in `foreach` compile to an indexed `for` loop.
- `List<T>`/`Dictionary` track a `_version`; structural modification during enumeration -> next `MoveNext()` throws `InvalidOperationException` ("Collection was modified").

---

### 8.1 [Easy] What does `yield return` do?

**Answer**
It lets a method produce a sequence lazily, returning one element at a time and remembering its position between calls. The compiler turns the method into an iterator state machine, so you don't write an `IEnumerator` by hand.

**Explanation**
Each `MoveNext()` runs the body until the next `yield return`. `yield break` ends the sequence early. No collection is built in memory, so it can model very large or infinite sequences.

**Code**
```csharp
static IEnumerable<int> Evens(int max)
{
    for (int i = 0; i <= max; i += 2)
    {
        Console.WriteLine($"producing {i}");
        yield return i;
    }
}

foreach (var e in Evens(4))
    Console.WriteLine($"consumed {e}");
// Output:
// producing 0
// consumed 0
// producing 2
// consumed 2
// producing 4
// consumed 4
```

### 8.2 [Follow-up] When does the code of an iterator method start running? How do you validate arguments eagerly?

**Answer**
Not when you call the method, but on the first `MoveNext()`. To validate eagerly, put the checks in a normal wrapper method and delegate to a private iterator (or a local function).

**Explanation**
A call like `var s = Gen(null);` returns immediately without throwing; the `ArgumentNullException` appears only on enumeration, far from the bug. LINQ operators use exactly this wrapper pattern.

**Code**
```csharp
static IEnumerable<int> Bad(string? s)
{
    ArgumentNullException.ThrowIfNull(s);        // deferred!
    yield return s.Length;
}

static IEnumerable<int> Good(string? s)
{
    ArgumentNullException.ThrowIfNull(s);        // runs immediately
    return Impl();
    IEnumerable<int> Impl() { yield return s.Length; }
}

var bad = Bad(null);                              // no exception yet
Console.WriteLine("Bad created");                 // Output: Bad created
try { Good(null); } catch (ArgumentNullException) { Console.WriteLine("Good threw immediately"); }
// Output: Good threw immediately
```

### 8.3 [Medium] What does `foreach` compile to, and why is it fine to `foreach` over a type that isn't `IEnumerable`?

**Answer**
It compiles to `GetEnumerator()`, a `while (MoveNext())` loop reading `Current`, wrapped in `try/finally` calling `Dispose()` if available. The compiler uses duck typing - any type with those public members works, which lets `List<T>` return a struct enumerator without allocations.

**Explanation**
Duck typing also enables `foreach` on `Span<T>` (a ref struct can't implement `IEnumerable`) and on custom types with an extension `GetEnumerator()` (C# 9). `IDisposable` handling matters for iterators with `finally` blocks.

**Code**
```csharp
class Countdown
{
    private readonly int _n;
    public Countdown(int n) => _n = n;
    public Enumerator GetEnumerator() => new(_n);
    public struct Enumerator
    {
        private int _cur;
        public Enumerator(int n) => _cur = n + 1;
        public int Current => _cur;
        public bool MoveNext() => --_cur > 0;
    }
}

foreach (var i in new Countdown(3)) Console.Write(i + " ");   // Output: 3 2 1
```

### 8.4 [Hard] Why does modifying a `List<T>` inside `foreach` throw, and what are safe alternatives? What about `Dictionary`?

**Answer**
`List<T>` increments `_version` on structural changes and the enumerator compares it on each `MoveNext`, throwing `InvalidOperationException` if it differs. Safe options: iterate a copy (`ToList()`), iterate backwards with `for`, use `RemoveAll`, or collect changes then apply. `Dictionary<,>` and `HashSet<T>` since .NET Core 3.0 permit `Remove`/`Clear` during enumeration but still throw on `Add`.

**Explanation**
The version check is best-effort bug detection, not thread safety. Across threads you need locks or concurrent collections. In older .NET Framework, any dictionary modification threw.

**Code**
```csharp
var list = new List<int> { 1, 2, 3, 4 };
try { foreach (var x in list) if (x == 2) list.Remove(x); }
catch (InvalidOperationException) { Console.WriteLine("modified"); }   // Output: modified

list = new List<int> { 1, 2, 3, 4 };
foreach (var x in list.ToList()) if (x % 2 == 0) list.Remove(x);       // iterate a snapshot
Console.WriteLine(string.Join(",", list));                             // Output: 1,3

var dict = new Dictionary<int, string> { [1] = "a", [2] = "b", [3] = "c" };
foreach (var kv in dict) if (kv.Key == 2) dict.Remove(kv.Key);         // OK on .NET Core 3.0+
Console.WriteLine(dict.Count);                                         // Output: 2
```

**Gotchas / follow-ups**
- `list[i] = x` on a `List<T>` also bumps `_version` and makes the next `MoveNext` throw; use a `for` loop if you need to overwrite elements while walking.
- `foreach` variable is read-only: `x++` doesn't compile.

### 8.5 [Tricky] Predict the output: an iterator with `try/finally` where the consumer stops early; also, are two enumerations independent?

**Answer**
`finally` runs when the enumerator is disposed - `foreach` calls `Dispose` on `break`. Each `GetEnumerator()` on a typical `yield` method produces an independent state machine (the first call on the creating thread reuses the same object, later calls allocate new ones), so two enumerations re-run the body from the start.

**Explanation**
If you call `MoveNext()` manually and never dispose, `finally` never runs. Resource cleanup inside iterators is therefore tied to disposing the enumerator - always use `foreach`/`using`.

**Code**
```csharp
static IEnumerable<int> Source()
{
    try
    {
        Console.WriteLine("start");
        yield return 1;
        yield return 2;
    }
    finally { Console.WriteLine("cleanup"); }
}

foreach (var x in Source()) { Console.WriteLine(x); break; }
// Output:
// start
// 1
// cleanup

var seq = Source();
Console.WriteLine(seq.First());   // Output: start / cleanup / 1  (First disposes the enumerator)
Console.WriteLine(seq.Count());   // Output: start / cleanup / 2  (second run; body runs again)
```

**Gotchas / follow-ups**
- Output ordering for `Console.WriteLine(seq.First())`: "start", "cleanup" are printed before "1" because the argument is evaluated first.
- `yield return` inside `try { } catch { }` is a compile error (CS1626); `try/finally` is OK.
- Iterators cannot have `ref` locals/`Span` locals across a `yield`.

**Interview traps**
- Lazy: exceptions/validation surface at first `MoveNext`.
- Multiple enumeration re-runs the iterator body (and its side effects).
- Modifying `List<T>` during `foreach` throws on the *next* `MoveNext`.
- Boxing: struct enumerators get boxed if you hold them as `IEnumerator<T>`.
- Mutable struct enumerators in `readonly` fields silently copy.

**Cheat sheet**
- `yield` = compiler-generated state machine, lazy.
- Wrapper + local function for eager validation.
- `foreach` = `GetEnumerator`/`MoveNext`/`Current` + `Dispose` in `finally`.
- Version counter -> "Collection was modified".
- No `yield` in `catch`/`finally`, no `ref` params.

---

## Topic 9: Delegates (Func/Action/Predicate, multicast)

**Study notes**

- A **delegate** is a type-safe, object-oriented function pointer: a reference type (class deriving from `MulticastDelegate`) holding a **target** object (null for static) + **method pointer**. Delegates are **immutable**; `+=`/`-=` create new instances via `Delegate.Combine`/`Remove`.
- Built-ins: `Action`/`Action<T1..T16>` (void), `Func<T1..T16,TResult>` (returns), `Predicate<T>` (`bool Predicate<in T>(T)`), `Comparison<T>`, `EventHandler`/`EventHandler<TEventArgs>`, `Converter<TIn,TOut>`.
- **Multicast**: a delegate holds an invocation list. `Invoke` calls each in order; the **return value is that of the last** handler; if one throws, later ones do **not** run. `GetInvocationList()` lets you call each individually.
- Equality: two delegates are equal if same method and same target (and same list).
- Conversions: method group -> delegate (`Func<int,int> f = Square;`); lambda -> delegate or `Expression<>`. Since C# 10 lambdas have a *natural type* (`var f = (int x) => x * 2;` is `Func<int,int>`); C# 12 allows default parameter values in lambdas.
- `BeginInvoke/EndInvoke` are not supported on .NET Core/5+ (throw `PlatformNotSupportedException`) - use `Task`.
- Delegate invocation is a bit slower than a direct call (indirect call, possible closure object); `static` lambdas avoid captures; `delegate*` function pointers are the unmanaged, allocation-free alternative.

---

### 9.1 [Easy] What is a delegate and how do `Func`, `Action`, `Predicate` differ?

**Answer**
A delegate is a type-safe reference to a method. `Action` returns `void`, `Func<..., TResult>` returns a value (last type parameter), and `Predicate<T>` is a `Func<T,bool>` specialised for tests (used by `List<T>.Find`, `RemoveAll`).

**Explanation**
Before generics you declared a custom `delegate` type for every signature; `Func`/`Action` made that mostly unnecessary. Custom delegates are still used for readability, `ref`/`out` parameters, and events.

**Code**
```csharp
Func<int, int, int> add = (a, b) => a + b;
Action<string> log = s => Console.WriteLine($"log: {s}");
Predicate<int> isEven = n => n % 2 == 0;

log($"{add(2, 3)}");                       // Output: log: 5
Console.WriteLine(isEven(4));              // Output: True
Console.WriteLine(new List<int> { 1, 2, 3, 4 }.FindAll(isEven).Count); // Output: 2
```

### 9.2 [Follow-up] What is a multicast delegate? What does it return and what happens if a handler throws?

**Answer**
A delegate whose invocation list has multiple methods, built with `+=`. Invoking calls them in order; the return value is that of the last one; an exception in one handler aborts the remaining handlers (the exception propagates to the caller).

**Explanation**
`GetInvocationList()` returns the individual delegates so you can invoke each in its own try/catch or collect all return values. Removing with `-=` removes the **last** matching occurrence.

**Code**
```csharp
Func<int> f = () => 1;
f += () => 2;
f += () => 3;
Console.WriteLine(f());                                     // Output: 3 (last wins)

Action a = () => Console.WriteLine("A");
a += () => throw new InvalidOperationException("B failed");
a += () => Console.WriteLine("C");
try { a(); } catch (InvalidOperationException e) { Console.WriteLine(e.Message); }
// Output: A
// Output: B failed        (C never runs)

foreach (Action handler in a.GetInvocationList())
{
    try { handler(); } catch { Console.WriteLine("handled"); }
}
// Output: A / handled / C
```

### 9.3 [Medium] How is a delegate represented in memory, and what's the cost of delegates vs. interface calls?

**Answer**
A delegate object stores `_target` (the instance, or the closure/null) and `_methodPtr` (plus `_methodPtrAux` for static/shuffle thunks). Calling goes through `Invoke`, an indirect call. Instance-method delegates keep the target alive. Creating a delegate allocates (unless cached), so lambdas without captures are cached by the compiler in a static field; capturing lambdas allocate a closure + delegate each time.

**Explanation**
Hot-path advice: use `static` lambdas (C# 9) to guarantee no capture, pass state through generic parameters/`TState`-style overloads (e.g. `ConcurrentDictionary.GetOrAdd(key, factory, arg)`), and avoid delegates in tight loops when an interface/generic struct constraint will do.

**Code**
```csharp
Func<int, int> noCapture = static x => x * 2;     // cannot capture: compile error if it tries
int factor = 3;
Func<int, int> capture = x => x * factor;         // closure class allocated

Console.WriteLine(noCapture(4));                  // Output: 8
Console.WriteLine(capture(4));                    // Output: 12
Console.WriteLine(capture.Target is null);        // Output: False (closure instance)
```

**Gotchas / follow-ups**
- `Delegate.Target` for non-capturing lambdas is an implementation detail; don't depend on it.
- A delegate to an instance method captures `this` -> keeps the whole object alive (common leak cause, see Topic 10).

### 9.4 [Hard] How do delegate variance and method-group conversion work? Can you assign `Func<string>` to `Func<object>`?

**Answer**
Yes: `Func<out TResult>` is covariant, so `Func<string>` converts to `Func<object>`. `Action<in T>` is contravariant. Method-group conversion is also flexible: the method may return a more derived type and accept less derived parameters than the delegate signature (`string M(object)` can bind to `Func<string,string>`; `object M(string)` cannot).

**Explanation**
This works for reference types only. The flexibility mirrors the interface-variance rules in Topic 3.

**Code**
```csharp
Func<string> fs = () => "hello";
Func<object> fo = fs;                         // covariance
Console.WriteLine(fo());                      // Output: hello

static string Describe(object o) => o.ToString() ?? "";
Func<string, string> g = Describe;            // method group: parameter contravariance
Console.WriteLine(g("x"));                    // Output: x

Action<object> ao = o => Console.WriteLine(o);
Action<string> astr = ao;                     // contravariance
astr("s");                                    // Output: s
```

### 9.5 [Tricky] What is the output? Delegates are immutable and compared by value.

**Answer**
`+=` creates a new delegate each time; a copy of the variable taken earlier is unaffected. Two delegates for the same method and target are equal, so `-=` with a *new but equal* lambda variable works only if it's the same method+target (lambdas create separate instances/methods, so removal with a fresh lambda fails).

**Explanation**
Removing a handler requires an equal delegate. Method groups (`h -= Handler`) work; an inline lambda (`h -= () => ...`) is a different compiled method, so it silently does nothing - a classic reason events leak when subscribers use lambdas.

**Code**
```csharp
Action a = () => Console.Write("1 ");
Action snapshot = a;
a += () => Console.Write("2 ");
snapshot();                                   // Output: 1
a();                                          // Output: 1 2
Console.WriteLine();

static void Hello() => Console.Write("H ");
Action h = Hello;
h += Hello;
h -= Hello;                                   // removes ONE occurrence
h();                                          // Output: H
Console.WriteLine();

Action l = null!;
l += () => Console.Write("L ");
l -= () => Console.Write("L ");               // different lambda: not removed
l();                                          // Output: L
```

**Gotchas / follow-ups**
- Invoking a null delegate throws `NullReferenceException`; use `handler?.Invoke()`.
- `a -= x` where the list becomes empty makes `a` `null`.

**Interview traps**
- Multicast returns the last handler's result; first exception stops the chain.
- Lambdas can't be unsubscribed with a new identical lambda.
- Instance-method delegates root the target object.
- `BeginInvoke` not supported in .NET Core+.
- `var f = x => x` doesn't compile without types (lambda parameter types needed, C# 10 natural type needs explicit param types).

**Cheat sheet**
- `Action` = void, `Func` = returns last type arg, `Predicate<T>` = bool test.
- Delegate = target + method pointer, immutable.
- Multicast: ordered, last return wins.
- `static` lambda = no capture allocation.
- Invoke safely: `d?.Invoke()`.

---

## Topic 10: Events (event vs delegate, leaks, weak events)

**Study notes**

- An **event** is a delegate field with restricted access: outside the declaring class you can only `+=` and `-=`; only the declaring class can invoke it or assign it. It is an encapsulation wrapper, not a different runtime type.
- A *field-like* event `public event EventHandler<T>? Changed;` makes the compiler generate a private delegate field plus `add`/`remove` accessors (thread-safe via `Interlocked.CompareExchange` loops). You can write custom accessors (e.g. to forward to another event or store handlers in a dictionary).
- Convention: `delegate void EventHandler(object? sender, EventArgs e)` / `EventHandler<TEventArgs>` (TEventArgs no longer needs to derive from `EventArgs`), raise via `protected virtual void OnX(...)` using `Changed?.Invoke(this, args)`.
- **Memory leak**: the publisher's delegate list holds a strong reference to every subscriber's *target object*. A long-lived publisher (static event, singleton service, `Application` events) keeps short-lived subscribers alive until they `-=`.

```
 Publisher (long-lived) --Changed--> [delegate: target=SubscriberA, method=OnChanged]
                                     [delegate: target=SubscriberB, method=OnChanged]   <- B cannot be GC'd
```
- Mitigations: always unsubscribe (`IDisposable`/`Dispose`), subscribe with method groups (so you can unsubscribe), **weak event pattern** (`WeakEventManager<TSource,TEventArgs>` in WPF; or a `WeakReference`-based wrapper handler), or use `IObservable`/messenger abstractions with weak refs.
- Handlers run **synchronously on the raising thread**; an exception in one handler stops the rest. `async void` handlers are fire-and-forget; their exceptions go to the `SynchronizationContext`/crash the process.

---

### 10.1 [Easy] What is the difference between an `event` and a plain public delegate field?

**Answer**
An `event` restricts outsiders to subscribing/unsubscribing; they cannot invoke it or replace the whole invocation list. A public delegate field allows anyone to call it, assign `= null` (wiping other subscribers), or overwrite handlers.

**Explanation**
This protects the publisher's invariants - only the owner decides *when* the event fires.

**Code**
```csharp
class Button
{
    public event EventHandler? Clicked;
    public Action? RawDelegate;                       // unprotected
    public void Click() => Clicked?.Invoke(this, EventArgs.Empty);
}

var b = new Button();
b.Clicked += (s, e) => Console.WriteLine("clicked");
b.RawDelegate += () => Console.WriteLine("raw");
b.Click();                         // Output: clicked
b.RawDelegate = null;              // allowed: wipes all subscribers!
// b.Clicked = null;               // compile error
// b.Clicked(b, EventArgs.Empty);  // compile error outside the class
```

### 10.2 [Follow-up] What is the standard pattern for raising an event safely?

**Answer**
Use `handler?.Invoke(this, args)` from a protected virtual `OnXxx` method. The null-conditional operator reads the field once, so another thread unsubscribing the last handler between the null check and the call can't cause a `NullReferenceException`.

**Explanation**
The old idiom was `var h = Changed; if (h != null) h(this, e);` - same idea: copy to a local, because delegates are immutable. Note this makes the raise race-free from NRE, but a handler might still be called right after it unsubscribed.

**Code**
```csharp
class OrderEventArgs(int id) : EventArgs { public int Id { get; } = id; }

class OrderService
{
    public event EventHandler<OrderEventArgs>? OrderPlaced;
    protected virtual void OnOrderPlaced(OrderEventArgs e) => OrderPlaced?.Invoke(this, e);
    public void Place(int id) => OnOrderPlaced(new OrderEventArgs(id));
}

var svc = new OrderService();
svc.OrderPlaced += (_, e) => Console.WriteLine($"order {e.Id}");
svc.Place(7);                                     // Output: order 7
```

### 10.3 [Medium] How do events cause memory leaks? Demonstrate and fix.

**Answer**
The publisher's delegate holds a strong reference to the subscriber instance. If the publisher outlives the subscriber and the subscriber never unsubscribes, the subscriber (and everything it references) can't be garbage collected.

**Explanation**
Typical culprits: static events, singleton/event-aggregator services, UI controls subscribing to long-lived view-models. Fix: unsubscribe in `Dispose`/`Unloaded`; keep the handler as a method (not an anonymous lambda you can't remove); or use weak events.

**Code**
```csharp
class Publisher { public event EventHandler? Tick; public void Raise() => Tick?.Invoke(this, EventArgs.Empty); }

class Subscriber : IDisposable
{
    private readonly Publisher _p;
    public Subscriber(Publisher p) { _p = p; _p.Tick += OnTick; }
    private void OnTick(object? s, EventArgs e) => Console.WriteLine("tick");
    public void Dispose() => _p.Tick -= OnTick;     // breaks the reference
}

var pub = new Publisher();
WeakReference wr;
{
    var sub = new Subscriber(pub);
    wr = new WeakReference(sub);
    sub.Dispose();                // comment this line out and the subscriber stays alive
}
GC.Collect(); GC.WaitForPendingFinalizers(); GC.Collect();
Console.WriteLine(wr.IsAlive);    // Output: False in a Release build (JIT may keep locals alive in Debug)
```

**Gotchas / follow-ups**
- In Debug builds the JIT extends local lifetimes; test leaks in Release or move the code into a separate non-inlined method.

### 10.4 [Hard] How would you implement a weak event?

**Answer**
The publisher holds a small forwarding object that references the real subscriber only through a `WeakReference`; when the subscriber has been collected the forwarder unsubscribes itself. In WPF use `WeakEventManager<TSource,TEventArgs>.AddHandler`.

**Explanation**
Weak events trade determinism for leak safety: a dead-but-not-yet-collected subscriber may still receive events until the GC runs, so explicit unsubscription is still preferred. The forwarder must reference the subscriber weakly and must not be a lambda capturing the subscriber strongly.

**Code**
```csharp
class Publisher
{
    public event EventHandler? Tick;
    public void Raise() => Tick?.Invoke(this, EventArgs.Empty);
}

class Listener { public void OnTick(object? s, EventArgs e) => Console.WriteLine("listener got tick"); }

class WeakTickForwarder
{
    private readonly WeakReference<Listener> _weak;
    private readonly Publisher _pub;

    public WeakTickForwarder(Publisher pub, Listener target)
    {
        _pub = pub;
        _weak = new WeakReference<Listener>(target);
        _pub.Tick += OnTick;                       // publisher references only the forwarder
    }

    private void OnTick(object? s, EventArgs e)
    {
        if (_weak.TryGetTarget(out var l)) l.OnTick(s, e);
        else _pub.Tick -= OnTick;                  // target collected: clean up
    }
}

var pub = new Publisher();
var listener = new Listener();
var fwd = new WeakTickForwarder(pub, listener);
pub.Raise();                                       // Output: listener got tick
GC.KeepAlive(listener);
```

**Gotchas / follow-ups**
- Don't use a `WeakReference` to the *delegate* itself: nothing else references it, so it is collected immediately.
- Weak events don't help when a lambda captures `this`; the closure object is what the delegate references.

### 10.5 [Tricky] Predict the behaviour: an event with several handlers where one throws; and an `async void` handler.

**Answer**
The exception from the second handler propagates to the raiser and the third handler never runs (unless you iterate `GetInvocationList()`). An `async void` handler returns at its first `await`; an exception thrown after that is posted to the captured `SynchronizationContext` (or crashes the process on the thread pool), never to the raiser.

**Explanation**
Raise events defensively when handlers are third-party: loop over `GetInvocationList()` with try/catch per handler. For async handlers prefer a pattern that returns `Task` (e.g. an `Func<TArgs,Task>` delegate or `Channel<T>`) and `await` each.

**Code**
```csharp
class Source
{
    public event Action? Fired;
    public void Raise()
    {
        if (Fired is null) return;
        foreach (Action h in Fired.GetInvocationList())
        {
            try { h(); }
            catch (Exception ex) { Console.WriteLine($"handler failed: {ex.Message}"); }
        }
    }
}

var s = new Source();
s.Fired += () => Console.WriteLine("one");
s.Fired += () => throw new InvalidOperationException("two broke");
s.Fired += () => Console.WriteLine("three");
s.Raise();
// Output: one
// Output: handler failed: two broke
// Output: three
```

**Gotchas / follow-ups**
- Unlike delegates, events can be declared in interfaces; `static` events are legal and are the classic leak source.
- Custom `add/remove` accessors must be thread-safe yourself.

**Interview traps**
- `event` is not a type; it's access restriction around a delegate.
- `handler?.Invoke` not `handler.Invoke` (null when no subscribers).
- Anonymous lambda subscriptions can't be unsubscribed -> leaks.
- `async void` handlers swallow/redirect exceptions.
- Handlers run synchronously, on the raising thread.

**Cheat sheet**
- Event = encapsulated multicast delegate (`+=`/`-=` only outside).
- Raise: `Changed?.Invoke(this, e)` in `OnChanged`.
- Leak = long-lived publisher + forgotten `-=`.
- Weak events: `WeakEventManager` / `WeakReference` forwarder.
- Per-handler try/catch via `GetInvocationList()`.

---

## Topic 11: Lambdas & closures

**Study notes**

- A **lambda** is an anonymous function convertible to a delegate (`Func<>`, `Action<>`...) or an `Expression<TDelegate>` (expression tree). C# 10: natural type + return-type inference; C# 12: default parameter values/`params` in lambdas.
- A **closure** is a lambda (or local function) that *captures* variables from its enclosing scope. The compiler hoists captured variables into a generated **display class** (`<>c__DisplayClass0_0`) so they live as long as the delegate does. You capture the **variable**, not its value at that moment.

```
 int x = 5; Func<int> f = () => x;          class <>c__DisplayClass { public int x; int <M>b__0() => x; }
 x = 10; f();  // 10                         the local 'x' is replaced by display.x everywhere
```
- Loop variables:
  - `for (int i...)`: **one** `i` for the entire loop -> all lambdas see the final value.
  - `foreach (var x in ...)`: since **C# 5**, a fresh `x` per iteration -> each lambda sees its own element.
- Costs: capturing allocates a display-class object and a delegate each time the delegate is created; non-capturing lambdas are cached in a static field. `static` lambdas (C# 9) forbid capture.
- Closures extend object lifetimes: any variable captured by *any* lambda in a scope shares the same display class (lifetime of the longest-living lambda), so unrelated large objects can be kept alive.
- Local functions can capture too and avoid delegate allocation when not converted to a delegate (compiler uses a struct closure).

---

### 11.1 [Easy] What is a lambda expression and how does it relate to delegates?

**Answer**
A lambda is an inline anonymous function written with `=>`. It converts to a delegate type (or an expression tree) whose signature matches; the compiler infers parameter types from the target.

**Explanation**
`x => x * 2` can become `Func<int,int>`, `Converter<int,int>` or `Expression<Func<int,int>>` depending on context. Statement lambdas use braces and `return`.

**Code**
```csharp
Func<int, int> twice = x => x * 2;
Func<int, int, int> add = (a, b) => { var s = a + b; return s; };
var square = (int x) => x * x;                 // natural type Func<int,int> (C# 10)
var withDefault = (int x, int y = 10) => x + y; // C# 12 default parameter

Console.WriteLine(twice(4));          // Output: 8
Console.WriteLine(add(2, 3));         // Output: 5
Console.WriteLine(square(5));         // Output: 25
Console.WriteLine(withDefault(1));    // Output: 11
```

### 11.2 [Follow-up] What is a closure? Does a lambda capture a value or a variable?

**Answer**
A closure is a lambda together with the captured variables of its enclosing scope. It captures the *variable* (by reference through a hidden class), so later changes to the variable are visible when the lambda runs.

**Explanation**
That is why the output reflects the value at *invocation* time, not at lambda-creation time.

**Code**
```csharp
int counter = 0;
Action inc = () => counter++;
inc(); inc();
Console.WriteLine(counter);          // Output: 2 (the lambda mutated the outer variable)

int x = 5;
Func<int> f = () => x;
x = 10;
Console.WriteLine(f());              // Output: 10
```

### 11.3 [Medium] Classic: lambdas created in a `for` loop vs a `foreach` loop - what's printed?

**Answer**
In a `for` loop all lambdas share the single loop variable, so after the loop they all print the final value (3). In a `foreach` (C# 5+) each iteration has its own variable, so they print 0 1 2. Copy the `for` variable into a local inside the body to fix it.

**Explanation**
The `for` loop variable `i` is declared once outside the body; the `foreach` iteration variable is declared inside each iteration. The fix `int copy = i;` creates a fresh variable (and display class instance) per iteration.

**Code**
```csharp
var fors = new List<Action>();
for (int i = 0; i < 3; i++) fors.Add(() => Console.Write(i + " "));
foreach (var a in fors) a();                 // Output: 3 3 3
Console.WriteLine();

var fixedFor = new List<Action>();
for (int i = 0; i < 3; i++) { int copy = i; fixedFor.Add(() => Console.Write(copy + " ")); }
foreach (var a in fixedFor) a();             // Output: 0 1 2
Console.WriteLine();

var each = new List<Action>();
foreach (var n in new[] { 0, 1, 2 }) each.Add(() => Console.Write(n + " "));
foreach (var a in each) a();                 // Output: 0 1 2
```

### 11.4 [Hard] What does the compiler generate for a closure, and what are the performance/memory implications?

**Answer**
A private nested "display class" holding the captured variables, plus a method for the lambda body; the enclosing method instantiates it at the start of the scope where the variable is declared and then allocates a delegate pointing at the display-class method. Each delegate creation allocates; all lambdas in the same scope share one display class.

**Explanation**
Implications: (1) allocations in hot paths (use `static` lambdas or state-passing overloads); (2) captured variables are heap-allocated for the entire lifetime of the delegate (a large buffer captured by one lambda stays alive while any lambda from that scope lives); (3) capturing `this` captures the whole object; (4) captured variables in `async` methods and iterators are hoisted into the state machine anyway. Tools: SharpLab or ILSpy show `<>c__DisplayClass`.

**Code**
```csharp
static Func<int> MakeCounter()
{
    int count = 0;                       // hoisted into a display class
    return () => ++count;
}

var c1 = MakeCounter();
var c2 = MakeCounter();                  // separate display-class instance
Console.WriteLine(c1());                 // Output: 1
Console.WriteLine(c1());                 // Output: 2
Console.WriteLine(c2());                 // Output: 1

// static lambda: compile-time guarantee of no capture, so no per-call closure allocation
Func<int, int> sq = static n => n * n;
Console.WriteLine(sq(6));                // Output: 36
```

**Gotchas / follow-ups**
- Two lambdas in one method, one capturing a 100 MB array and another capturing a small `int`: both delegates share the same display class when variables share a scope, keeping the array alive.
- `ref`/`out`/`in` parameters and `ref struct` locals (`Span<T>`) cannot be captured.

### 11.5 [Tricky] What is printed? Closure over a modified variable plus deferred LINQ.

**Answer**
The query is lazy and captures the `threshold` *variable*, so changing it before enumeration changes the result: it prints `4` (items greater than 3), not `2,3,4`. A materialised copy taken earlier keeps the old result.

**Explanation**
Combines closure capture with deferred execution (Topic 13). To fix a snapshot, copy to a new variable or materialise with `ToList()` before changing.

**Code**
```csharp
int threshold = 1;
var query = new[] { 1, 2, 3, 4 }.Where(x => x > threshold);   // captures 'threshold' variable
threshold = 3;
Console.WriteLine(string.Join(",", query));                    // Output: 4

int snap = 1;
var materialised = new[] { 1, 2, 3, 4 }.Where(x => x > snap).ToList();
snap = 3;
Console.WriteLine(string.Join(",", materialised));             // Output: 2,3,4
```

**Gotchas / follow-ups**
- Capturing a mutable variable in lambdas run on multiple threads is a data race.
- A `foreach` variable of a *struct* captured in a lambda is a copy per iteration, OK; but `for` loops remain the trap.

**Interview traps**
- Captures variables, not values.
- `for` shares the loop variable; `foreach` (C# 5+) does not.
- A closure's lifetime extends captured objects (leaks via `this`).
- Lambdas that capture allocate per creation; non-capturing ones are cached.
- Can't capture `ref`/`out`/`in` params or `Span<T>`.

**Cheat sheet**
- Lambda -> delegate or expression tree.
- Closure = lambda + display class of captured variables.
- Loop fix: copy to a local.
- `static` lambda = no capture.
- Query lambdas see the variable's value at enumeration time.

---

## Topic 12: Anonymous types & tuples

**Study notes**

- **Anonymous type**: `new { Id = 1, Name = "A" }` - compiler-generated, **immutable** (read-only properties), sealed **reference type** deriving from `object`. Has structural `Equals`/`GetHashCode`/`ToString` (`{ Id = 1, Name = A }`). Same property names/types/order in the same assembly -> same type. Can't be a return type/parameter except as `object`/`dynamic`/generic inference. Supports non-destructive mutation with `with` (C# 10). Common in LINQ projections and composite `GroupBy`/`Join` keys.
- **Tuples** (`System.ValueTuple<...>`, C# 7): **mutable value types** with fields `Item1..ItemN`; element names (`(int Id, string Name)`) are compile-time only (stored in attributes), they do not exist at runtime. Equality `==`/`!=` (C# 7.3) is element-wise; `Equals`, `GetHashCode` structural -> usable as dictionary keys. Deconstruction: `var (id, name) = t;`, discards `_`.
- Old `System.Tuple<...>` is an immutable **class** (`Item1` read-only) - avoid in new code.

| | Anonymous type | ValueTuple | `record` |
|---|---|---|---|
| Kind | sealed class | struct | class/struct |
| Mutability | immutable | mutable fields | immutable (init) or positional |
| Nameable in signatures | no | yes | yes |
| Structural equality | yes | yes | yes |
| Best for | local LINQ projections | multiple returns, private helpers | DTOs/domain values |

---

### 12.1 [Easy] What is an anonymous type and when do you use it?

**Answer**
A compiler-generated immutable class created with `new { ... }` to bundle a few read-only properties without declaring a type. You mostly use it for LINQ projections within a method.

**Explanation**
You can't name it, so you use `var`; it can't leave the method in a strongly typed way (use a record/tuple/DTO for that).

**Code**
```csharp
var person = new { Id = 1, Name = "Ann" };
Console.WriteLine(person.Name);           // Output: Ann
Console.WriteLine(person);                // Output: { Id = 1, Name = Ann }
// person.Name = "Bob";                    // compile error: read-only

var names = new[] { "ann", "bob" }.Select(n => new { n, Len = n.Length });
Console.WriteLine(string.Join(";", names)); // Output: { n = ann, Len = 3 };{ n = bob, Len = 3 }
```

### 12.2 [Follow-up] How do tuples differ from anonymous types? How do you return multiple values?

**Answer**
Tuples are `ValueTuple` structs, can be named in method signatures, and are mutable; anonymous types are immutable reference types you cannot name in signatures. For multiple return values prefer tuples (or records for public APIs).

**Explanation**
Tuple element names are only metadata on the signature (`TupleElementNamesAttribute`); reflection and `ToString()` show `Item1`... e.g. `(1, Ann)`.

**Code**
```csharp
static (int Min, int Max) Range(IEnumerable<int> xs) => (xs.Min(), xs.Max());

var (lo, hi) = Range(new[] { 4, 2, 9 });
Console.WriteLine($"{lo}-{hi}");          // Output: 2-9

var t = (Id: 1, Name: "Ann");
Console.WriteLine(t);                      // Output: (1, Ann)
Console.WriteLine(t.Item2 == t.Name);      // Output: True
Console.WriteLine((1, "a") == (1, "a"));   // Output: True (element-wise ==)
```

### 12.3 [Medium] Anonymous types as composite keys in `GroupBy`/`Join` - why does it work?

**Answer**
The compiler gives every anonymous type value-based `Equals` and `GetHashCode` over all its properties, so two instances with equal property values are considered the same key. That makes them convenient composite keys.

**Explanation**
Value tuples work the same way and are allocation-free (struct), so `GroupBy(x => (x.City, x.Year))` is typically cheaper in-memory. In EF Core both translate to `GROUP BY` on multiple columns.

**Code**
```csharp
var sales = new[]
{
    new { City = "Pune", Year = 2023, Amt = 10 },
    new { City = "Pune", Year = 2023, Amt = 5 },
    new { City = "Pune", Year = 2024, Amt = 7 },
};

var byKey = sales.GroupBy(s => new { s.City, s.Year })
                 .Select(g => new { g.Key.City, g.Key.Year, Total = g.Sum(x => x.Amt) });
foreach (var r in byKey) Console.WriteLine($"{r.City} {r.Year} {r.Total}");
// Output: Pune 2023 15
// Output: Pune 2024 7

Console.WriteLine(new { A = 1 }.Equals(new { A = 1 })); // Output: True
```

### 12.4 [Hard] Why can't you modify a tuple inside a `List<(int,int)>` via indexer, and what are other tuple pitfalls?

**Answer**
`list[0]` returns a **copy** of the struct, so assigning to `list[0].Item1` modifies a temporary; the compiler rejects it (CS1612). You must replace the whole element, or use `CollectionsMarshal.AsSpan(list)[0].Item1 = ...`. Other pitfalls: names aren't preserved at runtime/in serialisation, mutable struct copies, and large tuples are bulky to copy (8+ elements nest as `Rest`).

**Explanation**
Value-type semantics: copy on read. Arrays are the exception: `arr[0].Item1 = 5` works because array element access yields a reference (a variable) to the element. For public APIs prefer records because tuple element names disappear in JSON/reflection (`Item1`, `Item2`).

**Code**
```csharp
var list = new List<(int X, int Y)> { (1, 2) };
// list[0].X = 10;                       // error CS1612: cannot modify the return value
list[0] = (10, list[0].Y);               // replace whole element
Console.WriteLine(list[0]);              // Output: (10, 2)

var arr = new (int X, int Y)[] { (1, 2) };
arr[0].X = 10;                           // OK: array element is a variable
Console.WriteLine(arr[0]);               // Output: (10, 2)

(int a, int b) = (1, 2);
(a, b) = (b, a);                         // swap without temp
Console.WriteLine($"{a} {b}");           // Output: 2 1
```

**Gotchas / follow-ups**
- Equality of tuples with different element *names* is still true; names are ignored.
- `Tuple.Create(1,2)` makes the old class `Tuple<int,int>`; `(1,2)` makes `ValueTuple<int,int>` - they are different types.

### 12.5 [Tricky] Can two anonymous types in different places be the same type? Can you return one from a method?

**Answer**
Yes - within one assembly, anonymous types with identical property names, types and order are the same type. You can't declare a method returning one by name; you can return `object`/`dynamic` (loses static typing) or flow it through generic inference (e.g. a lambda). Prefer a tuple or record.

**Explanation**
Across assemblies they are different types (internal to each), which is why returning them from a public library method is a bug source, and why passing them via `dynamic` can fail with `RuntimeBinderException` (anonymous types are `internal`).

**Code**
```csharp
var a = new { X = 1, Y = "s" };
var b = new { X = 2, Y = "t" };
Console.WriteLine(a.GetType() == b.GetType());       // Output: True

var c = new { Y = "s", X = 1 };                       // different order
Console.WriteLine(a.GetType() == c.GetType());       // Output: False

var d = a with { X = 99 };                           // C# 10 non-destructive mutation
Console.WriteLine(d);                                // Output: { X = 99, Y = s }
```

**Interview traps**
- Anonymous types are reference types; tuples are value types.
- Tuple names don't exist at runtime.
- Modifying a tuple returned from `List<T>`/property getter doesn't compile (copy).
- Anonymous types can't be returned strongly typed; `dynamic` consumption across assemblies fails.

**Cheat sheet**
- Anonymous type: immutable sealed class, structural equality, local use.
- `ValueTuple`: mutable struct, named elements (compile-time), `var (a,b) = t`.
- Composite key: tuple or anonymous type.
- Public API: use `record`, not tuple/anon.
- `with` works on anonymous types and records.

---

## Topic 13: LINQ - deferred execution & core operators

**Study notes**

- LINQ-to-Objects = extension methods on `IEnumerable<T>` in `System.Linq.Enumerable`. Query operators build a **pipeline of iterators**; nothing runs until you enumerate.

| Execution | Operators |
|---|---|
| **Deferred, streaming** (yield one by one) | `Where`, `Select`, `SelectMany`, `Take`, `Skip`, `TakeWhile`, `Concat`, `Zip`, `Cast`, `OfType`, `Distinct`, `Union` |
| **Deferred, buffering** (reads all source on first `MoveNext`) | `OrderBy/ThenBy`, `GroupBy`, `Reverse`, `Join/GroupJoin` (buffers the *inner* sequence), `Except/Intersect` (buffer the second) |
| **Immediate** | `ToList`, `ToArray`, `ToDictionary`, `ToHashSet`, `ToLookup`, `Count`, `Sum`, `Min`, `Max`, `Average`, `Aggregate`, `First`, `Single`, `Any`, `All`, `Contains`, `ElementAt`, `SequenceEqual` |

- **Multiple enumeration**: a lazy query re-executes (and repeats side effects/DB calls) on every enumeration. Materialise once (`ToList`/`ToArray`) if reused.
- `First` throws on empty; `FirstOrDefault` returns default; `Single` throws if 0 **or** >1 items; `SingleOrDefault` throws if >1. `Sum` on `int` throws `OverflowException` (checked); `Average/Max/Min` on an empty non-nullable sequence throw `InvalidOperationException`.
- Useful .NET 6+: `DistinctBy`, `MinBy`/`MaxBy`, `UnionBy`, `ExceptBy`, `Chunk`, `Zip` (3-way), `TryGetNonEnumeratedCount`, `Take(Range)`, `Order()`/`OrderDescending()` (.NET 7).
- Query syntax (`from x in xs where ... select ...`) compiles to the same method calls.

---

### 13.1 [Easy] What is deferred execution in LINQ? Which operators execute immediately?

**Answer**
A query like `Where`/`Select` only describes the work; it runs when you enumerate the result (`foreach`, `ToList`, `Count`...). Operators that return a single value or a materialised collection (`ToList`, `Count`, `First`, `Sum`, `Any`) execute immediately.

**Explanation**
Consequence: the query sees the data and captured variables as they are **at enumeration time**, not definition time.

**Code**
```csharp
var numbers = new List<int> { 1, 2, 3 };
var query = numbers.Where(n => n > 1);      // nothing executed yet
numbers.Add(4);
Console.WriteLine(string.Join(",", query)); // Output: 2,3,4  (sees the later Add)

var snapshot = numbers.Where(n => n > 1).ToList();   // immediate
numbers.Add(5);
Console.WriteLine(snapshot.Count);          // Output: 3
```

### 13.2 [Follow-up] What is "multiple enumeration" and why is it a problem?

**Answer**
Enumerating the same lazy query more than once re-runs the whole pipeline each time - duplicate work, repeated side effects, repeated database/HTTP calls, and possibly different results if the source changed.

**Explanation**
Tools like ReSharper flag "possible multiple enumeration of IEnumerable". Fix by materialising once with `ToList()`/`ToArray()`, or by accepting `IReadOnlyCollection<T>` in APIs that need a count plus iteration.

**Code**
```csharp
int calls = 0;
var query = Enumerable.Range(1, 3).Select(x => { calls++; return x * 2; });

Console.WriteLine(query.Max());              // Output: 6
Console.WriteLine(query.Sum());              // Output: 12
Console.WriteLine(calls);                    // Output: 6 (selector ran twice per item)

var list = query.ToList();                   // calls -> 9 (one more full pass)
Console.WriteLine(list.Count + list.Sum());  // reuse without re-running: 3 + 12 = 15  // Output: 15
```

### 13.3 [Medium] Explain `SelectMany`, `Join`, `GroupJoin` (left join) and `Zip` with examples.

**Answer**
`SelectMany` flattens nested sequences. `Join` is an inner join on keys; a left outer join is `GroupJoin` + `DefaultIfEmpty` (or `SelectMany` over the group). `Zip` pairs elements positionally and stops at the shortest sequence.

**Explanation**
`Join` builds a lookup over the inner sequence and streams the outer one: O(n + m). Left joins need `GroupJoin` because `Join` drops outer items without matches.

**Code**
```csharp
var customers = new[] { new { Id = 1, Name = "Ann" }, new { Id = 2, Name = "Bob" } };
var orders    = new[] { new { CustId = 1, Item = "Pen" }, new { CustId = 1, Item = "Ink" } };

var inner = customers.Join(orders, c => c.Id, o => o.CustId, (c, o) => $"{c.Name}:{o.Item}");
Console.WriteLine(string.Join(",", inner));   // Output: Ann:Pen,Ann:Ink

var left = customers.GroupJoin(orders, c => c.Id, o => o.CustId, (c, os) => new { c, os })
                    .SelectMany(x => x.os.DefaultIfEmpty(), (x, o) => $"{x.c.Name}:{o?.Item ?? "-"}");
Console.WriteLine(string.Join(",", left));    // Output: Ann:Pen,Ann:Ink,Bob:-

var flat = new[] { new[] { 1, 2 }, new[] { 3 } }.SelectMany(a => a);
Console.WriteLine(string.Join(",", flat));    // Output: 1,2,3

var zipped = new[] { 1, 2, 3 }.Zip(new[] { "a", "b" });   // stops at shortest (.NET 6 tuple overload)
Console.WriteLine(string.Join(",", zipped));  // Output: (1, a),(2, b)
```

### 13.4 [Hard] How do `GroupBy`, `ToLookup` and `Aggregate` work? When is `GroupBy` streaming?

**Answer**
`GroupBy` returns `IEnumerable<IGrouping<TKey,TElement>>`; on the first `MoveNext` it reads the **entire** source into groups (order of first appearance preserved), so it's deferred but not streaming. `ToLookup` does the same eagerly and returns an immutable `ILookup` you can index by key (missing key -> empty sequence). `Aggregate` folds a sequence with an accumulator, with an optional seed and result selector.

**Explanation**
In EF Core a `GroupBy` followed by aggregate projection (`g.Key`, `g.Count()`, `g.Sum`) translates to SQL `GROUP BY`; projecting the group's elements may not translate / triggers client evaluation. `Aggregate` without a seed throws on empty input; with a seed it returns the seed.

**Code**
```csharp
var words = new[] { "apple", "avocado", "banana", "blueberry", "cherry" };

var groups = words.GroupBy(w => w[0]);
foreach (var g in groups) Console.WriteLine($"{g.Key}: {string.Join("/", g)}");
// Output: a: apple/avocado
// Output: b: banana/blueberry
// Output: c: cherry

var lookup = words.ToLookup(w => w.Length);
Console.WriteLine(lookup[6].Count());          // Output: 2 (banana, cherry)
Console.WriteLine(lookup[99].Count());         // Output: 0 (no exception)

int product = new[] { 1, 2, 3, 4 }.Aggregate(1, (acc, x) => acc * x);
Console.WriteLine(product);                    // Output: 24
string csv = new[] { "a", "b", "c" }.Aggregate((acc, x) => acc + "," + x);
Console.WriteLine(csv);                        // Output: a,b,c
```

**Gotchas / follow-ups**
- Using `Aggregate` to concatenate strings is O(n^2); use `string.Join`.
- `new int[0].Aggregate((a, b) => a + b)` throws `InvalidOperationException`.

### 13.5 [Tricky] Predict the output: lazy `Select` with side effects, `Take`, and short-circuiting; and the `First/Single` differences.

**Answer**
Iterator pipelines pull one item at a time through every stage, so `Take(2)` stops the source after two items and side effects interleave (`select 1, where 1, select 2, ...`). `Single` throws if the sequence has zero or more than one match; `First` takes the first, `SingleOrDefault` still throws on >1.

**Explanation**
Streaming lets you short-circuit infinite sequences: `Enumerable.Range`/`yield` infinite generator + `Take`. But `OrderBy` or `GroupBy` before `Take` forces a full read first.

**Code**
```csharp
var q = Enumerable.Range(1, 10)
    .Select(x => { Console.Write($"S{x} "); return x; })
    .Where(x => { Console.Write($"W{x} "); return x % 2 == 0; })
    .Take(2);
foreach (var x in q) Console.Write($"=>{x} ");
// Output: S1 W1 S2 W2 =>2 S3 W3 S4 W4 =>4

var items = new[] { 5, 8, 12 };
Console.WriteLine(items.First(x => x > 6));            // Output: 8
try { items.Single(x => x > 6); }
catch (InvalidOperationException) { Console.WriteLine("more than one"); }  // Output: more than one
Console.WriteLine(items.SingleOrDefault(x => x > 100)); // Output: 0
```

**Gotchas / follow-ups**
- `Take(2)` stops after the second yielded item - it does not evaluate item 5.
- `OrderBy` is a *stable* sort and buffers everything before yielding the first item.

**Interview traps**
- Query reflects data changes made after definition (deferred).
- `Count()` then `foreach` = two enumerations.
- `Single` vs `First` semantics; `SingleOrDefault` throws on duplicates.
- `GroupBy`/`OrderBy`/`Join` buffer even though they're deferred.
- `Join` is inner; left join = `GroupJoin` + `DefaultIfEmpty`.

**Cheat sheet**
- Deferred = runs on enumeration; `To*`/aggregates/`First` are immediate.
- Streaming vs buffering: `Where/Select/Take` vs `OrderBy/GroupBy/Reverse`.
- `SelectMany` flatten; `Zip` shortest; `ToLookup` eager group.
- `Aggregate(seed, func)`; seedless throws on empty.
- Materialise once to avoid re-execution.

---

## Topic 14: LINQ performance pitfalls & expression trees

**Study notes**

*Performance pitfalls*
- **Multiple enumeration** of an expensive lazy query (DB, I/O, heavy projection).
- `Count() > 0` instead of `Any()` on non-collection sources (full pass vs first-item check). On `List<T>`/arrays `Count()` is O(1) via `ICollection<T>`; `Count` *property* is always best.
- `list.Where(x => other.Contains(x))` with `other` a `List<T>` -> O(n*m). Use a `HashSet<T>`.
- `OrderBy(...).First()` sorts everything: O(n log n). Use `MinBy`/`MaxBy` (.NET 6) for O(n).
- Chained `ToList()` between steps -> extra allocations/copies; materialise once at the end.
- Allocation per query: iterator objects, closures, delegates, boxing of struct enumerators; LINQ in hot loops is slower than a plain `for`/`foreach` (no LINQ on `Span<T>`).
- `Contains`/`Distinct`/`GroupBy` on custom types without good `GetHashCode`/`Equals` -> degenerate hashing.
- EF Core: LINQ that can't translate -> exception / client eval; N+1 queries from lazy loading in loops; `IEnumerable` instead of `IQueryable` (Topic 7); `Count()` on big sets vs `Any()`.
- Prefer `ElementAt`/`Last` on `IList<T>` (O(1) optimised) but not on lazy sequences.

*Expression trees*
- `Expression<Func<T,TResult>>` stores code **as data**: a tree of `Expression` nodes (`ParameterExpression`, `ConstantExpression`, `MemberExpression`, `BinaryExpression`, `MethodCallExpression`, `LambdaExpression`...). Compiler builds it when a lambda converts to `Expression<TDelegate>`.
- Uses: LINQ providers (EF Core translates to SQL), mappers, dynamic predicate building, validation/`nameof`-style property selectors, ASP.NET model binding helpers.
- `.Compile()` produces a delegate (JIT cost; **cache** it). Trees are immutable; rewrite with an `ExpressionVisitor`.
- Limits: only expression lambdas (no statement bodies), no assignment/`++`, no `?.`/`??=`/`dynamic`/optional-arg calls (in a lambda -> expression tree).

```
 x => x.Age > 18 && x.Name != null
 LambdaExpression
  +- Parameter x
  +- AndAlso
      +- GreaterThan( MemberAccess(x.Age), Constant(18) )
      +- NotEqual   ( MemberAccess(x.Name), Constant(null) )
```

---

### 14.1 [Easy] Name three common LINQ performance mistakes.

**Answer**
(1) Enumerating a lazy query multiple times; (2) using `Count() > 0` rather than `Any()` (or `Contains` on a `List` inside a loop instead of a `HashSet`); (3) sorting an entire sequence just to get the min/max/first. Also dropping to `foreach` in genuinely hot paths.

**Explanation**
Each costs an extra full pass or an O(n log n) sort. Profile first - LINQ is readable and fine outside hot paths.

**Code**
```csharp
var data = Enumerable.Range(1, 1000).Select(x => x * 3).ToList();

bool hasAny = data.Any(x => x > 2000);                 // stops at first match
var largest = data.MinBy(x => -x);                     // O(n); .NET 6+
// vs data.OrderBy(x => -x).First()                    // O(n log n)

var allowed = new HashSet<int> { 3, 6, 9 };
var hits = data.Where(allowed.Contains).ToList();      // O(n) with O(1) lookups
Console.WriteLine($"{hasAny} {largest} {hits.Count}"); // Output: True 3000 3
```

### 14.2 [Follow-up] `Any()` vs `Count() > 0`, and `First()` vs `FirstOrDefault()` / `Single()` - which and why?

**Answer**
`Any()` stops at the first element; `Count()` may scan the whole sequence (O(1) only if the source is an `ICollection<T>`/known-size). In EF Core `Any()` translates to `EXISTS`, which is much cheaper than `COUNT(*)`. `Single` costs a second element probe (EF adds `TOP 2`), so use `First` when you don't need uniqueness validation.

**Explanation**
Use the collection's own `Count`/`Length` property when you have a `List`/array; use `Any()` for lazy/query sources.

**Code**
```csharp
IEnumerable<int> lazy = Enumerable.Range(1, 1_000_000).Where(x => x % 2 == 0);
Console.WriteLine(lazy.Any());            // Output: True (touches only a couple of items)

var list = new List<int> { 1, 2, 3 };
Console.WriteLine(list.Count);            // Output: 3 (property, O(1))
Console.WriteLine(list.Count() > 0);      // True, but goes via ICollection<T>; prefer the property
```

### 14.3 [Medium] What is an expression tree and how is `Expression<Func<T,bool>>` different from `Func<T,bool>`?

**Answer**
`Func<T,bool>` is compiled executable code; `Expression<Func<T,bool>>` is a data structure describing the lambda's code that you can inspect, translate (to SQL) or compile later. The compiler decides based on the target type the lambda is assigned to.

**Explanation**
You can go expression -> delegate with `.Compile()`, but not delegate -> expression. The expression tree captures *captured variables* as member accesses on a constant closure object, which EF Core turns into SQL parameters.

**Code**
```csharp
using System.Linq.Expressions;

Expression<Func<int, bool>> expr = x => x > 5;
Console.WriteLine(expr);                           // Output: x => (x > 5)

var body = (BinaryExpression)expr.Body;
Console.WriteLine(body.NodeType);                  // Output: GreaterThan
Console.WriteLine(body.Right);                     // Output: 5

Func<int, bool> compiled = expr.Compile();
Console.WriteLine(compiled(10));                   // Output: True

int limit = 3;
Expression<Func<int, bool>> withCapture = x => x > limit;
Console.WriteLine(((BinaryExpression)withCapture.Body).Right is MemberExpression); // Output: True (closure field access)
```

### 14.4 [Hard] How do you build an expression tree dynamically and rewrite one with `ExpressionVisitor`?

**Answer**
Use the factory methods on `Expression` (`Parameter`, `Property`, `Constant`, `Equal`, `AndAlso`, `Lambda`) to assemble nodes, then `Expression.Lambda<TDelegate>(body, params)`. To transform an existing tree, subclass `ExpressionVisitor` and override `VisitXxx`; the base class rebuilds immutable nodes.

**Explanation**
Typical use: user-driven filters ("column X contains Y") that must still translate to SQL through EF Core; or combining two predicates (need to replace parameters with a visitor so both bodies use the same `ParameterExpression`). Compiling a tree each call is slow (~ms) - cache the delegate.

**Code**
```csharp
using System.Linq.Expressions;

// Build: p => p.Length > 3
var p = Expression.Parameter(typeof(string), "p");
var body = Expression.GreaterThan(Expression.Property(p, nameof(string.Length)), Expression.Constant(3));
var lambda = Expression.Lambda<Func<string, bool>>(body, p);
Console.WriteLine(lambda);                                   // Output: p => (p.Length > 3)
Console.WriteLine(lambda.Compile()("hello"));                // Output: True

// Rewrite: replace the constant 3 with 10
class ConstReplacer : ExpressionVisitor
{
    protected override Expression VisitConstant(ConstantExpression node)
        => node.Value is 3 ? Expression.Constant(10) : base.VisitConstant(node);
}

var rewritten = (Expression<Func<string, bool>>)new ConstReplacer().Visit(lambda);
Console.WriteLine(rewritten);                                // Output: p => (p.Length > 10)
```

**Gotchas / follow-ups**
- `Expression.AndAlso(e1.Body, e2.Body)` fails (parameters differ) unless you rewrite `e2` to use `e1`'s parameter via a visitor - the classic "PredicateBuilder" pitfall.
- Nodes are immutable: `Visit` returns a new tree.
- Type mismatches (`Expression.Equal(int, long)`) throw `InvalidOperationException` at build time - no implicit conversions; add `Expression.Convert`.

### 14.5 [Tricky] Why can't some lambdas convert to `Expression<Func<>>`, and what performance trap lurks in `Compile()`?

**Answer**
Expression trees only model expressions: statement bodies (`{ ... }`), assignments, `?.`, `dynamic`, `ref` locals, etc. aren't supported, so those lambdas can only become delegates (compiler errors CS0834, CS8072...). And `Compile()` invokes the compiler/JIT each time (hundreds of microseconds to ms), so compiling inside a loop is a big performance bug - cache the delegate.

**Explanation**
Likewise, passing `Expression<Func<>>` where a `Func<>` suffices adds overhead. In EF Core, a predicate parameter typed `Func<T,bool>` forces in-memory filtering (Topic 7). `Expression.Lambda(...).Compile(preferInterpretation: true)` can avoid JIT for one-off use.

**Code**
```csharp
using System.Linq.Expressions;

// Expression<Func<int,int>> bad = x => { var y = x + 1; return y; };   // error CS0834: statement body
Expression<Func<int, int>> ok = x => x + 1;

// Cache compiled delegates per expression (key by expression text for illustration)
Console.WriteLine(Cache<int, int>.Get(ok)(41));   // Output: 42

static class Cache<T, R>
{
    private static readonly Dictionary<string, Func<T, R>> Map = new();
    public static Func<T, R> Get(Expression<Func<T, R>> e)
    {
        var key = e.ToString();
        if (!Map.TryGetValue(key, out var f)) Map[key] = f = e.Compile();
        return f;
    }
}
```

**Gotchas / follow-ups**
- Keying by `ToString()` conflates different captured variables; real caches use structural comparers or explicit keys.
- The static `Dictionary` above isn't thread-safe - use `ConcurrentDictionary` in real code.

**Interview traps**
- `Count() > 0` vs `Any()`; `OrderBy().First()` vs `MinBy`.
- `List.Contains` inside `Where` -> O(n*m).
- `Func` where `Expression<Func>` is expected silently moves work client-side.
- `Compile()` in a loop; combining predicates without parameter replacement.
- LINQ allocates; benchmark before replacing with loops.

**Cheat sheet**
- Expression tree = code as data (`Expression<Func<>>`), `.Compile()` -> delegate.
- Providers translate trees; delegates run in-process.
- `ExpressionVisitor` to rewrite; trees are immutable.
- Cache compiled delegates.
- Prefer `Any`, `HashSet`, `MinBy`, single materialisation.

---

## Topic 15: Immutable & concurrent collections; Dictionary vs ConcurrentDictionary

**Study notes**

- **Immutable collections** (`System.Collections.Immutable`): every "mutation" returns a **new** collection and leaves the original unchanged; unchanged parts are shared (**structural sharing**), so it is cheaper than a full copy for tree-based types. Inherently thread-safe to read/share.

| Type | Internals | Index/lookup | Add/modify |
|---|---|---|---|
| `ImmutableArray<T>` | struct wrapping `T[]` | O(1) | O(n) copy |
| `ImmutableList<T>` | balanced (AVL) tree | O(log n) | O(log n) |
| `ImmutableDictionary<K,V>` / `ImmutableHashSet<T>` | hash-ordered tree (AVL of buckets) | O(log n) | O(log n) |
| `ImmutableSortedDictionary` / `Set` | AVL tree | O(log n) | O(log n) |
| `ImmutableQueue<T>`/`Stack<T>` | linked structures | O(1) ends | O(1) |

- Use **builders** (`ImmutableArray.CreateBuilder<T>()`, `list.ToBuilder()`) for bulk construction, then `ToImmutable()`.
- **.NET 8 Frozen collections** (`System.Collections.Frozen`): `FrozenDictionary<K,V>`/`FrozenSet<T>` via `ToFrozenDictionary()`/`ToFrozenSet()` - slow to create, **fastest to read**; ideal for long-lived lookup tables built at startup.
- `ReadOnlyCollection<T>` / `IReadOnlyList<T>` are read-only *views*, not immutable.
- **Concurrent collections** (`System.Collections.Concurrent`): `ConcurrentDictionary`, `ConcurrentQueue`, `ConcurrentStack`, `ConcurrentBag` (unordered, per-thread lists), `BlockingCollection<T>` (bounded blocking producer/consumer), and `System.Threading.Channels.Channel<T>` (modern async producer/consumer).
- `ConcurrentDictionary` internals: array of buckets + a set of **locks (lock striping, default concurrency level ~ processor count)**; reads are lock-free; writes lock only the relevant stripe; whole-collection operations (`Count`, `ToArray`, `Keys`, `Values`, `IsEmpty`... ) acquire all locks.
- Atomic ops: `TryAdd`, `TryUpdate`, `TryRemove`, `AddOrUpdate`, `GetOrAdd`. The **factory/update delegates may run more than once and outside the lock**; only one result is stored.

---

### 15.1 [Easy] What is an immutable collection and why use it?

**Answer**
One that can never change after creation; operations like `Add` return a new collection. Benefits: thread-safety without locks, safe sharing/caching, no defensive copies, easy reasoning and snapshots.

**Explanation**
Because internal nodes are shared, `a.Add(x)` is much cheaper than copying the whole list for tree-based immutables (but `ImmutableArray<T>` still copies). The cost is slower reads than `List<T>` for tree-based ones and more allocation churn.

**Code**
```csharp
using System.Collections.Immutable;

var a = ImmutableList.Create(1, 2, 3);
var b = a.Add(4);                         // new list; 'a' untouched
Console.WriteLine($"{a.Count} {b.Count}"); // Output: 3 4

var arr = ImmutableArray.Create("x", "y");
var arr2 = arr.Add("z");
Console.WriteLine(string.Join(",", arr2)); // Output: x,y,z
Console.WriteLine(arr.Length);             // Output: 2
```

### 15.2 [Follow-up] How do `ImmutableList<T>`, `ImmutableArray<T>`, builders and `FrozenDictionary` differ and when do you choose each?

**Answer**
`ImmutableArray<T>` - fastest reads, rarely changed (cheap struct wrapper). `ImmutableList<T>` - frequent "modify" producing new versions (tree, O(log n)). Builders - efficient batch construction. `FrozenDictionary/FrozenSet` (.NET 8) - build once, read many times with best lookup speed.

**Explanation**
Don't use `ImmutableList<T>` just for iteration/indexing speed - `List<T>`/`ImmutableArray<T>` are faster. `ImmutableArray<T>` default value (`default`) is an uninitialised struct (`IsDefault == true`) that throws on `Length` - use `IsDefaultOrEmpty`.

**Code**
```csharp
using System.Collections.Frozen;
using System.Collections.Immutable;

var builder = ImmutableArray.CreateBuilder<int>();
for (int i = 0; i < 5; i++) builder.Add(i);
ImmutableArray<int> built = builder.ToImmutable();
Console.WriteLine(built.Length);                      // Output: 5

FrozenDictionary<string, int> codes =
    new Dictionary<string, int> { ["IN"] = 91, ["US"] = 1 }.ToFrozenDictionary();
Console.WriteLine(codes["IN"]);                       // Output: 91

ImmutableArray<int> uninit = default;
Console.WriteLine(uninit.IsDefault);                  // Output: True
```

### 15.3 [Medium] Dictionary vs ConcurrentDictionary - when is plain `Dictionary` unsafe and what does `ConcurrentDictionary` guarantee?

**Answer**
`Dictionary` supports concurrent *reads* only when nothing writes; any concurrent write (or write during read) can corrupt internal state, lose entries or throw. `ConcurrentDictionary` makes individual operations atomic and thread-safe, but compound "check-then-act" sequences across multiple calls are still not atomic.

**Explanation**
Use `ConcurrentDictionary` for shared caches/counters with many threads. For read-mostly data built once, an ordinary `Dictionary` (or `FrozenDictionary`) shared after construction is fine and faster. A single-threaded dictionary protected by `lock` can outperform `ConcurrentDictionary` under heavy write contention with tiny critical sections, so measure.

**Code**
```csharp
using System.Collections.Concurrent;

var counts = new ConcurrentDictionary<string, int>();
Parallel.For(0, 1000, i => counts.AddOrUpdate("hits", 1, (_, old) => old + 1));
Console.WriteLine(counts["hits"]);               // Output: 1000

// NOT atomic even on ConcurrentDictionary: read-modify-write in two calls
// counts["hits"] = counts["hits"] + 1;          // lost updates under concurrency
```

### 15.4 [Hard] What are the pitfalls of `GetOrAdd`/`AddOrUpdate` factories? How do you guarantee the factory runs once?

**Answer**
The value factory runs **outside the lock** and may be executed several times concurrently for the same key; only one produced value wins, the others are discarded (side effects/costly work duplicated, disposable results leaked). Wrap the value in `Lazy<T>` (or `Lazy<Task<T>>`) so the expensive work happens once on first `.Value`.

**Explanation**
`AddOrUpdate`'s update delegate may also be retried if the value changed concurrently - it must be side-effect free and idempotent. Overloads with an extra `TArg factoryArgument` avoid closure allocation (`GetOrAdd(key, static (k, arg) => ..., arg)`).

**Code**
```csharp
using System.Collections.Concurrent;

var cache = new ConcurrentDictionary<string, Lazy<int>>();
int calls = 0;

Parallel.For(0, 100, _ =>
{
    int v = cache.GetOrAdd("k", key => new Lazy<int>(() =>
    {
        Interlocked.Increment(ref calls);   // expensive init
        return 42;
    })).Value;
});

Console.WriteLine(calls);                   // Output: 1  (only the stored Lazy is ever evaluated)
Console.WriteLine(cache["k"].Value);        // Output: 42
```

**Gotchas / follow-ups**
- `Lazy<T>` default mode `ExecutionAndPublication` caches exceptions; a failed init stays failed - consider removing the entry on failure.
- For async init use `ConcurrentDictionary<K, Lazy<Task<V>>>`, or `AsyncLazy`; never block on `.Result` inside the factory.

### 15.5 [Tricky] Is enumerating a `ConcurrentDictionary` a snapshot? What do `Count`, `Keys` and `ToArray` cost? When prefer Channels/BlockingCollection?

**Answer**
Enumeration never throws on concurrent modification, but it is **not** a point-in-time snapshot: you may or may not see concurrent changes. `Count`, `Keys`, `Values`, `ToArray`, `IsEmpty` acquire **all** the internal locks (expensive, block writers) and return a consistent snapshot. For producer/consumer pipelines use `Channel<T>` (async, bounded, backpressure) or `BlockingCollection<T>` (blocking threads), not polling a concurrent queue.

**Explanation**
Avoid calling `Count` in hot paths; track your own counter with `Interlocked` if needed. `ConcurrentBag<T>` has no ordering and `TryTake` can return items from other threads' lists; it's only advantageous when the same threads both produce and consume.

**Code**
```csharp
using System.Threading.Channels;

var channel = Channel.CreateBounded<int>(capacity: 2);

var producer = Task.Run(async () =>
{
    for (int i = 1; i <= 5; i++) await channel.Writer.WriteAsync(i);   // waits when full (backpressure)
    channel.Writer.Complete();
});

int sum = 0;
await foreach (var item in channel.Reader.ReadAllAsync()) sum += item;
await producer;
Console.WriteLine(sum);                                                // Output: 15
```

**Gotchas / follow-ups**
- `ConcurrentQueue<T>.Count` is O(1)-ish but `IsEmpty` is preferred for emptiness; `ConcurrentStack.Count` is O(n).
- `ImmutableInterlocked.Update(ref field, f)` gives lock-free updates of immutable collections held in a field (retry loop on CAS).

**Interview traps**
- Immutable != `IReadOnlyList`; `ReadOnlyCollection` reflects later changes.
- Check-then-act on `ConcurrentDictionary` (`ContainsKey` then `Add`) is a race; use `TryAdd`/`GetOrAdd`.
- `GetOrAdd` factory can run multiple times.
- `Count`/`ToArray`/`Keys` lock everything.
- `default(ImmutableArray<T>)` is uninitialised.

**Cheat sheet**
- Immutable: new instance per change, structural sharing, thread-safe.
- `ImmutableArray` = fast reads; `ImmutableList` = log n edits; Frozen = build once read fast.
- `Dictionary` safe only for read-only sharing.
- `ConcurrentDictionary`: atomic single ops, striped locks, `Lazy<T>` for once-only init.
- Pipelines: `Channel<T>`.

---

## Topic 16: Comparer / IComparable / sorting stability

**Study notes**

- `IComparable<T>.CompareTo(T other)` - the type's *natural order* (used by `Comparer<T>.Default`, `SortedSet`, `List.Sort()`, `Max`). `IComparer<T>.Compare(x, y)` / `Comparison<T>` delegate - an *external* ordering. Contract: negative = less, 0 = equal, positive = greater; must be **consistent** (antisymmetric, transitive, `Compare(x,x)==0`).
- `Comparer<T>.Default` uses `IComparable<T>`, else non-generic `IComparable`, else throws `InvalidOperationException` ("Failed to compare two elements in the array"). `Comparer<T>.Create(Comparison<T>)` makes a comparer from a lambda. `StringComparer` offers `Ordinal`, `OrdinalIgnoreCase`, `CurrentCulture`, `InvariantCulture`.
- **Stability**: a stable sort keeps the relative order of equal elements.
  - `List<T>.Sort` / `Array.Sort`: **introsort** (quicksort + heapsort fallback + insertion sort for small partitions) - O(n log n), in-place, **unstable**.
  - LINQ `OrderBy`/`ThenBy`/`OrderByDescending`: **stable** (buffers + stable index-tie-breaking sort).
  - `PriorityQueue` is not stable either.
- Don't implement comparers via subtraction (`a - b`) - overflow gives wrong sign.
- `string.CompareTo`/default comparison is **culture-sensitive**; for keys/identifiers use `StringComparer.Ordinal`.
- `SortedSet/SortedDictionary/SortedList` treat `Compare == 0` as "same key"; hash collections use `Equals`/`GetHashCode`. Keep both consistent.
- Struct types: implement `IComparable<T>` (generic) to avoid boxing in sorts.

---

### 16.1 [Easy] How do you sort a list by a property, and what's the difference between `IComparable<T>` and `IComparer<T>`?

**Answer**
Use `OrderBy`/`ThenBy` (LINQ, returns a new sequence) or `list.Sort(comparison)` (in place). `IComparable<T>` is implemented by the type itself and defines its default order; `IComparer<T>` is a separate object that defines an alternative order, so one type can have many orderings.

**Explanation**
Implement `IComparable<T>` when there's one obvious natural order (e.g. version numbers, money); use comparers/lambdas for context-specific sorts.

**Code**
```csharp
record Person(string Name, int Age) : IComparable<Person>
{
    public int CompareTo(Person? other) => other is null ? 1 : string.CompareOrdinal(Name, other.Name);
}

var people = new List<Person> { new("Cid", 30), new("Ann", 25), new("Bob", 30) };

people.Sort();                                          // natural order (Name)
Console.WriteLine(string.Join(",", people.Select(p => p.Name))); // Output: Ann,Bob,Cid

people.Sort((a, b) => b.Age.CompareTo(a.Age));          // external order, in place
Console.WriteLine(people[^1].Name);                     // Output: Ann (youngest last)

var sorted = people.OrderBy(p => p.Age).ThenBy(p => p.Name).Select(p => p.Name);
Console.WriteLine(string.Join(",", sorted));            // Output: Ann,Bob,Cid
```

### 16.2 [Follow-up] Is `List<T>.Sort` stable? Is `OrderBy`? Why does it matter?

**Answer**
`List<T>.Sort`/`Array.Sort` are **not** stable (introsort); `OrderBy`/`ThenBy` **are** stable. It matters when you sort by one key after another or when equal items carry other data whose original order (e.g. insertion/time order) must be preserved.

**Explanation**
To get a stable in-place sort use `OrderBy(...).ToList()`, or sort by a composite comparer that adds a tie-breaker (index or secondary key). Do multi-key sorts with `ThenBy` rather than two successive `Sort` calls (the second would scramble the first).

**Code**
```csharp
var items = new[] { ("Ann", 30), ("Bob", 25), ("Cid", 30), ("Dan", 25) };

var stable = items.OrderBy(x => x.Item2).Select(x => x.Item1);
Console.WriteLine(string.Join(",", stable));            // Output: Bob,Dan,Ann,Cid (ties keep input order)

var desc = items.OrderByDescending(x => x.Item2).Select(x => x.Item1);
Console.WriteLine(string.Join(",", desc));              // Output: Ann,Cid,Bob,Dan (ties STILL keep input order)

// Array.Sort / List.Sort on equal keys: order of ties is unspecified (do not rely on it)
```

### 16.3 [Medium] Why is `Compare = (a, b) => a - b` a bug? And what's wrong with culture-sensitive string sorting?

**Answer**
Subtraction overflows for large magnitudes (`int.MaxValue - (-1)`), producing the wrong sign and an inconsistent comparer; use `a.CompareTo(b)`. Default `string.CompareTo` is culture-sensitive: ordering depends on the current culture and can differ between machines; use `StringComparer.Ordinal` for identifiers/keys and `OrdinalIgnoreCase` for case-insensitive keys.

**Explanation**
Culture-sensitive comparison (ICU on modern .NET) may sort "a" before "B" while ordinal sorts "B" before "a" (by UTF-16 code unit). The Turkish-I problem is the classic bug. Note: with `InvariantGlobalization=true` culture comparisons degrade to ordinal behaviour.

**Code**
```csharp
Comparison<int> bad  = (a, b) => a - b;
Comparison<int> good = (a, b) => a.CompareTo(b);
Console.WriteLine(bad(int.MaxValue, -1));      // Output: -2147483648 (wrong sign: overflow)
Console.WriteLine(good(int.MaxValue, -1));     // Output: 1

var words = new[] { "b", "A", "a", "B" };
Console.WriteLine(string.Join(",", words.OrderBy(w => w, StringComparer.Ordinal)));
// Output: A,B,a,b
Console.WriteLine(string.Join(",", words.OrderBy(w => w, StringComparer.OrdinalIgnoreCase)));
// Output: A,a,b,B   (ties 'A'/'a' and 'b'/'B' keep input order: stable)
```

### 16.4 [Hard] How does `Comparer<T>.Default` choose a comparer, and what happens with an inconsistent comparer?

**Answer**
It picks `GenericComparer<T>` if `T : IComparable<T>`, `NullableComparer` for `Nullable<T>`, an enum comparer for enums, `ObjectComparer` for non-generic `IComparable`, and otherwise throws `InvalidOperationException` (wrapping an `ArgumentException`: "At least one object must implement IComparable") when a comparison is actually needed. An inconsistent comparer (e.g. returns random values, or `Compare(a,b)>0` and `Compare(b,a)>0`) can make `Sort` throw `InvalidOperationException` ("IComparer.Compare() method returns inconsistent results"), produce an unsorted result, or - in older frameworks - loop/crash.

**Explanation**
Null handling: `Comparer<T>.Default` treats null as less than non-null for reference types. A comparer must also be consistent with `SortedSet`'s notion of duplicates: elements with `Compare == 0` are the same element and the second add is ignored.

**Code**
```csharp
class NoOrder { }

try { new List<NoOrder> { new(), new() }.Sort(); }
catch (InvalidOperationException) { Console.WriteLine("not comparable"); }  // Output: not comparable

var set = new SortedSet<string>(Comparer<string>.Create((a, b) => a.Length.CompareTo(b.Length)));
set.Add("aa"); set.Add("bb"); set.Add("c");
Console.WriteLine(string.Join(",", set));      // Output: c,aa   ("bb" ignored: Compare("aa","bb") == 0)

Console.WriteLine(Comparer<string?>.Default.Compare(null, "x")); // Output: -1 (null is smallest)
```

**Gotchas / follow-ups**
- Comparing `double.NaN`: `double.CompareTo` treats NaN as smaller than everything (total order), while `<`/`>` operators with NaN are always false.
- Sorting a `List<T>` of structs through `IComparable` (non-generic) boxes every comparison.

### 16.5 [Tricky] Multi-key sorting, descending ties, and "sort then Reverse" - what's the difference?

**Answer**
`OrderByDescending` keeps ties in original order, while `OrderBy(...).Reverse()` also reverses the tie order. Multi-key sorts must use `ThenBy` (or one composite comparer); calling `OrderBy` twice makes the *second* key the primary key (the first only breaks ties because of stability).

**Explanation**
Because LINQ sorting is stable, `OrderBy(x => A).OrderBy(x => B)` yields B-primary, A-secondary - works but is confusing and does two sorts; `ThenBy` is clearer and one sort. `List<T>.Reverse()` is in-place; `Enumerable.Reverse()` is lazy (buffers all).

**Code**
```csharp
var data = new[] { (N: "a", K: 1), (N: "b", K: 1), (N: "c", K: 0) };

Console.WriteLine(string.Join(",", data.OrderByDescending(x => x.K).Select(x => x.N)));
// Output: a,b,c   (ties a,b keep input order)
Console.WriteLine(string.Join(",", data.OrderBy(x => x.K).Reverse().Select(x => x.N)));
// Output: b,a,c   (ties reversed)

var byBoth = data.OrderBy(x => x.K).ThenByDescending(x => x.N).Select(x => x.N);
Console.WriteLine(string.Join(",", byBoth));  // Output: c,b,a
```

**Interview traps**
- `List.Sort` is unstable; `OrderBy` is stable.
- Subtraction comparers overflow; use `CompareTo`.
- Culture-sensitive string sort differs by machine; use `Ordinal` for keys.
- `Compare == 0` means "duplicate" in sorted sets, regardless of `Equals`.
- Two successive sorts: the last sort's key becomes primary.

**Cheat sheet**
- `IComparable<T>` = natural order; `IComparer<T>`/`Comparison<T>` = custom order.
- `List.Sort`/`Array.Sort` = introsort, unstable, in place.
- `OrderBy/ThenBy` = stable, lazy, buffered.
- Use `Comparer<T>.Create`, `StringComparer.Ordinal`.
- Comparer must be consistent (transitive, antisymmetric).

---

*End of A03.*

