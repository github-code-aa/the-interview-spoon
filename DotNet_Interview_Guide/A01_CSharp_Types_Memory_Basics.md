# A01 — C# Types & Memory Basics

**How to use:** Read each topic's *Study notes* first, then work down the difficulty ladder (Easy → Tricky). Cover the **Answer**, say it aloud, then check the **Explanation** and run the **Code**.
Before an interview, skim only the **Interview traps** and **Cheat sheet** of every topic. Code targets C# 12 / .NET 8 (top-level statements or a `Main` wrapper; implicit usings on).

## Table of Contents
1. [Value vs reference types & stack/heap myths](#topic-1-value-vs-reference-types--stackheap-myths)
2. [Boxing and unboxing](#topic-2-boxing-and-unboxing)
3. [string: immutability, interning, StringBuilder](#topic-3-string-immutability-interning-stringbuilder)
4. [const vs readonly vs static readonly](#topic-4-const-vs-readonly-vs-static-readonly)
5. [Static members & static constructors](#topic-5-static-members--static-constructors)
6. [ref / out / in / ref returns / ref structs](#topic-6-ref--out--in--ref-returns--ref-structs)
7. [Nullable value types & nullable reference types](#topic-7-nullable-value-types--nullable-reference-types)
8. [var vs dynamic vs object](#topic-8-var-vs-dynamic-vs-object)
9. [struct vs class vs record vs record struct](#topic-9-struct-vs-class-vs-record-vs-record-struct)
10. [Enums & [Flags]](#topic-10-enums--flags)
11. [Equality & Object members](#topic-11-equality--object-members)
12. [Parameters: optional, named, params](#topic-12-parameters-optional-named-params)
13. [Properties vs fields, auto-props, init/required](#topic-13-properties-vs-fields-auto-props-initrequired)
14. [Operators & implicit/explicit conversions](#topic-14-operators--implicitexplicit-conversions)
15. [checked/unchecked, integer/decimal/double precision](#topic-15-checkedunchecked-integerdecimaldouble-precision)
16. [DateTime vs DateTimeOffset](#topic-16-datetime-vs-datetimeoffset)
17. [default / default literal, tuples & deconstruction](#topic-17-default--default-literal-tuples--deconstruction)

---

## Topic 1: Value vs reference types & stack/heap myths

**Study notes**

- **Value types** (`struct`, `enum`, primitives, `DateTime`, `Nullable<T>`, `ValueTuple`) derive from `System.ValueType`. Assignment/argument passing **copies the value**. Variables *contain* the data.
- **Reference types** (`class`, `interface`, `delegate`, `string`, arrays, `object`, `record`) — the variable holds a **reference**; assignment copies the reference, both point to the same object. Default value is `null`.
- **Where do they live?** The real rule: *"a value type lives wherever its container lives."*

```
local int x            -> stack frame (or a CPU register after JIT)
field int in a class   -> inside the heap object
int[] elements         -> inside the heap array object
captured local (lambda)-> heap closure class
local in async method  -> heap state machine (if it lives across an await)
boxed int              -> own heap object
reference-type local   -> the *reference* on stack/register; the object on heap
```

- Heap objects are managed by the GC (generational: Gen0/1/2, LOH for ≥85,000 bytes). Stack memory is reclaimed when the frame pops, no GC.
- Reference-type *object* location is an implementation detail; newer runtimes (.NET 9+) can stack-allocate some non-escaping objects. Don't claim "classes are always on the heap" as a language guarantee.

### 1.1 [Easy] What is the difference between a value type and a reference type?

**Answer**
A value type variable holds its data directly and is copied on assignment; a reference type variable holds a reference to an object, so assignment copies the reference and both variables see the same object. Structs/enums/primitives are value types; classes/interfaces/delegates/strings/arrays are reference types.

**Explanation**
Copy semantics are the observable difference. Mutating a copy of a struct never affects the original, while mutating an object through any reference is visible via all references. Value types cannot be `null` unless wrapped in `Nullable<T>`.

**Code**
```csharp
struct PointS { public int X; }
class PointC { public int X; }

class Demo
{
    static void Main()
    {
        var s1 = new PointS { X = 1 };
        var s2 = s1;          // copy
        s2.X = 99;
        Console.WriteLine(s1.X); // Output: 1

        var c1 = new PointC { X = 1 };
        var c2 = c1;          // same object
        c2.X = 99;
        Console.WriteLine(c1.X); // Output: 99
    }
}
```

### 1.2 [Follow-up] "Value types live on the stack, reference types on the heap" — true?

**Answer**
No, it's a half-truth. A value type lives wherever its container lives: a local struct may be on the stack (or in a register), but a struct field of a class, an element of an array, a captured variable, or a boxed value lives on the heap. Reference-type *variables* hold a reference; the object itself is normally on the heap.

**Explanation**
The stack-vs-heap distinction is an implementation detail, not part of the C# spec. The correct mental model is semantic (copy vs reference). The spec itself talks about "storage locations". Also, JIT may enregister locals, and newer runtimes may stack-allocate non-escaping objects.

**Code**
```csharp
class Holder
{
    public int Field = 5;           // int stored INSIDE the heap object Holder
}

class Demo
{
    static void Main()
    {
        int local = 1;              // stack/register
        var h = new Holder();       // reference on stack, object on heap
        int[] arr = new int[3];     // array object on heap; ints live inside it

        Func<int> f = () => local;  // 'local' is hoisted into a heap closure object
        local++;
        Console.WriteLine(f());     // Output: 2  (the lambda sees the hoisted variable)
    }
}
```

### 1.3 [Medium] You pass a `List<int>` to a method that does `list = new List<int>()`. Does the caller see it? What about `list.Add(1)`?

**Answer**
Reference types are passed by value *of the reference*. `Add` mutates the shared object, so the caller sees it. Reassigning the parameter only changes the local copy of the reference, so the caller doesn't see it — unless the parameter is `ref`.

**Explanation**
"Reference types are passed by reference" is a common wrong answer. By default *everything* is passed by value; for reference types the value is the reference. `ref` passes the variable itself (an alias).

**Code**
```csharp
static void Mutate(List<int> l)   { l.Add(1); }
static void Reassign(List<int> l) { l = new List<int> { 100 }; }
static void ReassignRef(ref List<int> l) { l = new List<int> { 100 }; }

var list = new List<int>();
Mutate(list);
Console.WriteLine(list.Count);   // Output: 1
Reassign(list);
Console.WriteLine(list.Count);   // Output: 1  (caller's reference unchanged)
ReassignRef(ref list);
Console.WriteLine(list[0]);      // Output: 100
```

### 1.4 [Hard] Why does mutating a struct stored in a `readonly` field or `List<T>` indexer "not work"?

**Answer**
Calling a mutating method on a struct in a `readonly` field operates on a hidden *defensive copy*, so the original isn't changed. A `List<T>` indexer returns a copy of the struct (not a reference), so `list[0].X = 5` is a compile error (CS1612); you must read, modify, and write back.

**Explanation**
The compiler can't know whether a method mutates `this`, so for `readonly` fields (and `in` parameters) it copies before invoking a non-`readonly` member. Marking the struct `readonly struct` (or the member `readonly`) lets the compiler skip the copy. Arrays are different: `arr[0].X = 5` works because array indexing yields a variable reference.

**Code**
```csharp
struct Counter
{
    public int N;
    public void Inc() => N++;
}

class Owner
{
    private readonly Counter _c = new();
    public int Run() { _c.Inc(); return _c.N; }   // Inc runs on a defensive copy
}

class Demo
{
    static void Main()
    {
        Console.WriteLine(new Owner().Run());      // Output: 0

        var list = new List<Counter> { new() };
        // list[0].N = 5;                          // error CS1612
        var tmp = list[0]; tmp.N = 5; list[0] = tmp; // read-modify-write
        Console.WriteLine(list[0].N);              // Output: 5

        var arr = new Counter[1];
        arr[0].Inc();                              // fine: array element is a variable
        Console.WriteLine(arr[0].N);               // Output: 1
    }
}
```

**Gotchas / follow-ups**
- Prefer immutable (`readonly struct`) value types; defensive copies then disappear.
- `foreach` iteration variables are readonly: assigning a struct member (`c.N = 1`) is a compile error (CS1654), and calling a mutating method (`c.Inc()`) silently runs on a copy.
- Follow-up: "How would you avoid the copy for big structs?" → `in` parameters / `ref readonly` returns, with `readonly struct`.

### 1.5 [Tricky] Is `string` a value type? What does this print?

**Answer**
`string` is a reference type, but it's immutable and overrides `==`, so it *behaves* like a value type for comparisons and "modifications". The code below prints `a` then `ab`, because `+=` creates a new string and rebinds only `t`.

**Explanation**
Two references to the same string can never see each other's changes because no change is possible. That immutability is why strings are safe to share, intern, and use as dictionary keys.

**Code**
```csharp
string s = "a";
string t = s;
t += "b";                // new string "ab"; t now references it
Console.WriteLine(s);    // Output: a
Console.WriteLine(t);    // Output: ab
```

**Gotchas / follow-ups**
- Don't concatenate in loops (O(n²) allocations) → `StringBuilder` (Topic 3).
- `object.ReferenceEquals(s, "a")` is true here due to interning of literals.

**Interview traps**
- "Reference types are passed by reference" — wrong; the reference is passed by value.
- "Structs are always on the stack" — wrong (fields, arrays, boxing, closures, async).
- Mutable structs + `readonly`/indexers/foreach cause silent no-op bugs.
- `string` is a reference type with value-like behavior.
- Value-type `default` is zeroed memory; reference-type default is `null`.

**Cheat sheet**
- Value type: copy on assign, derives `ValueType`, no null (without `?`).
- Reference type: variable = pointer to heap object; default `null`.
- Location = wherever the container lives; stack-vs-heap is an implementation detail.
- Everything is passed by value unless `ref`/`out`/`in`.
- Prefer immutable (`readonly`) structs.

---

## Topic 2: Boxing and unboxing

**Study notes**

- **Boxing** = converting a value type to `object`, `System.ValueType`, or an interface it implements. The runtime allocates a heap object (header + method table pointer + copy of the value) and copies the value in.
- **Unboxing** = explicit cast back; runtime checks the boxed object's *exact* type and copies the value out.

```
int i = 42;            stack:  [ 42 ]
object o = i;          stack:  [ ref ] ---> heap: [hdr|MT(int)|42]    (boxing: alloc + copy)
int j = (int)o;        type check == int? then copy 42 out            (unboxing)
```

- Cost: allocation + GC pressure + copy + type check. Hot path boxing is a classic performance bug.
- Common hidden boxing: non-generic collections (`ArrayList`), `string.Format`/interpolation with value-type args into `object` params (older APIs; interpolated string handlers in C# 10+ avoid many), calling an interface method on a struct via interface variable, `enum.ToString()`-like APIs taking `Enum`, using struct as `object` key, lambdas/delegates bound to struct instance methods.
- Generics (`List<int>`) avoid boxing: the JIT specializes code for each value type.
- `Nullable<T>`: boxing `int?` with value gives a boxed `int`; with no value gives `null`.

### 2.1 [Easy] What is boxing and unboxing?

**Answer**
Boxing wraps a value type in a heap-allocated object (implicit conversion to `object` or an interface); unboxing is the explicit cast that extracts the value back. Both have a cost: allocation on box, type check and copy on unbox.

**Explanation**
Boxing creates a *copy*; the box and the original are independent after that.

**Code**
```csharp
int i = 10;
object o = i;       // boxing
i = 20;             // does not affect the box
int j = (int)o;     // unboxing
Console.WriteLine($"{i} {j}");   // Output: 20 10
```

### 2.2 [Follow-up] Where does boxing happen accidentally, and how do you avoid it?

**Answer**
`ArrayList`/non-generic APIs, passing value types to `object` parameters, casting a struct to an interface, and using a struct as a key through non-generic comparers. Avoid by using generics, generic constraints (`where T : struct, IFoo`), and strongly typed overloads.

**Explanation**
A *constrained* generic call (`T : IFoo` with `T` a struct) does not box: the JIT emits a `constrained.` call that invokes the method directly on the struct. Casting to the interface type first would box.

**Code**
```csharp
using System.Collections;

interface IShape { double Area(); }
struct Square : IShape
{
    public double Side;
    public double Area() => Side * Side;
}

class Demo
{
    static double Boxes(IShape s) => s.Area();                // call site boxes a Square
    static double NoBox<T>(T s) where T : IShape => s.Area(); // no boxing for struct T

    static void Main()
    {
        var sq = new Square { Side = 2 };
        Console.WriteLine(Boxes(sq));   // Output: 4  (boxed)
        Console.WriteLine(NoBox(sq));   // Output: 4  (not boxed)

        var al = new ArrayList();
        al.Add(1);                      // boxes
        int x = (int)al[0]!;            // unboxes
        Console.WriteLine(x);           // Output: 1
    }
}
```

### 2.3 [Medium] Why does `(long)o` fail when `o` holds a boxed `int`?

**Answer**
Unboxing requires the *exact* boxed type; it is not a conversion. An `InvalidCastException` is thrown. Unbox to `int` first, then convert: `(long)(int)o`.

**Explanation**
Unboxing is a type-identity check plus copy, whereas `(long)intValue` is a numeric conversion. Exception: an enum can be unboxed as its underlying type and vice versa, e.g. a boxed `DayOfWeek` can be unboxed as `int`.

**Code**
```csharp
object o = 5;
try { long l = (long)o; }
catch (InvalidCastException) { Console.WriteLine("InvalidCast"); }  // Output: InvalidCast

long ok = (long)(int)o;
Console.WriteLine(ok);                    // Output: 5

object e = DayOfWeek.Monday;
Console.WriteLine((int)e);                // Output: 1  (enum <-> underlying type allowed)

object n = null!;
try { int bad = (int)n; }                 // unboxing null into non-nullable value type
catch (NullReferenceException) { Console.WriteLine("NRE"); } // Output: NRE
```

### 2.4 [Hard] A struct implements an interface with a mutating method. What happens when you mutate it through the interface variable?

**Answer**
Casting to the interface boxes a *copy*; mutations through the interface affect that boxed copy only. The original struct variable remains unchanged.

**Explanation**
Each assignment to an interface variable creates a new box. Further calls on the same interface reference mutate the same box, which is why such mutable structs produce confusing behavior. This is a main reason to keep structs immutable.

**Code**
```csharp
interface ICounter { void Inc(); int N { get; } }

struct Counter : ICounter
{
    public int N { get; private set; }
    public void Inc() => N++;
}

class Demo
{
    static void Main()
    {
        Counter c = default;
        ICounter boxed = c;      // box (copy)
        boxed.Inc();
        boxed.Inc();
        Console.WriteLine(c.N);       // Output: 0
        Console.WriteLine(boxed.N);   // Output: 2

        ICounter other = boxed;  // copies the reference, same box
        other.Inc();
        Console.WriteLine(boxed.N);   // Output: 3
    }
}
```

**Gotchas / follow-ups**
- Follow-up: "How would you avoid boxing here?" → generic method with `where T : struct, ICounter` and `ref T`.
- `Dictionary<TKey,...>` with a struct key that doesn't implement `IEquatable<T>` boxes on every comparison via `Equals(object)`.

### 2.5 [Tricky] Boxing `Nullable<T>`, and `==` on boxed values

**Answer**
Boxing `int?` that has a value produces a boxed `int` (there is no boxed `Nullable<int>`); with no value it produces `null`. And `==` on two `object` variables compares references, so two separate boxes of the same int are not `==` but are `Equals`.

**Explanation**
`GetType()` on a boxed nullable returns `typeof(int)`. Comparing boxed values with `==` is a classic bug; use `Equals` or unbox first.

**Code**
```csharp
int? a = 5;
object oa = a;
Console.WriteLine(oa.GetType());   // Output: System.Int32

int? none = null;
object on = none;
Console.WriteLine(on is null);     // Output: True

object x = 1, y = 1;
Console.WriteLine(x == y);         // Output: False  (reference comparison, two boxes)
Console.WriteLine(x.Equals(y));    // Output: True
Console.WriteLine((int)x == (int)y); // Output: True

int? back = (int?)oa;              // unbox int box into int?
Console.WriteLine(back);           // Output: 5
```

**Gotchas / follow-ups**
- `oa is int` is true; `oa is int?` is also true (pattern for non-null).
- `ReferenceEquals(1, 1)` is false: each argument is boxed separately.

**Interview traps**
- "Unboxing converts" — no; it needs the exact type (`(long)obj` on boxed int throws).
- `object == object` is reference equality even when both hold ints.
- Mutable struct via interface mutates a box copy.
- Interface-typed parameters on structs box at the call site; generics with constraints don't.
- Boxing allocates → measure with a profiler/`BenchmarkDotNet` memory diagnoser.

**Cheat sheet**
- Box: value → `object`/interface (alloc+copy). Unbox: cast (exact type check + copy).
- `List<int>` no box; `ArrayList` box.
- Boxed `int?` → `int` or `null`.
- Use `where T : struct, IFoo` to avoid boxing.
- `object==object` → reference comparison.

---

## Topic 3: string: immutability, interning, StringBuilder

**Study notes**

- `string` = immutable sequence of UTF-16 code units (`char`). Reference type, sealed, overrides `==`/`Equals` (ordinal value equality) and `GetHashCode`.
- Any "modification" (`Replace`, `+`, `Substring`, `ToUpper`) returns a **new** string.
- **Interning:** the CLR keeps an *intern pool*. All compile-time literals (and constant expressions like `"a" + "b"`) are interned automatically. Runtime-created strings are not, unless `string.Intern(s)` is called. `string.IsInterned(s)` checks.
- **StringBuilder:** a mutable buffer (chunk list) for many appends; `ToString()` produces the final string. Use when concatenating in loops; for ≤ ~4 concatenations in one expression, `+`/interpolation is fine (compiler uses `String.Concat`/`DefaultInterpolatedStringHandler`).
- Comparison: `==`/`Equals` = ordinal, case-sensitive. Use `StringComparison.OrdinalIgnoreCase` for identifiers; `CurrentCulture`/`InvariantCulture` for linguistic compare. `string.Compare`, `StartsWith(string)` etc. without a comparison parameter can be culture-sensitive (`StartsWith`, `IndexOf(string)` are culture-sensitive; `Equals`, `Contains`, `Replace` are ordinal).
- Useful: `string.IsNullOrEmpty`, `IsNullOrWhiteSpace`, `string.Join`, `string.Create`, `Span<char>`/`ReadOnlySpan<char>` slicing without allocation, raw string literals `"""..."""`, verbatim `@""`.

### 3.1 [Easy] Why are strings immutable, and what are the benefits?

**Answer**
Once created, a string's characters can't change; operations return new strings. That makes strings inherently thread-safe, safe as dictionary keys (stable hash code), and allows interning and sharing.

**Explanation**
Hash codes can be cached/stable, security-sensitive values (paths, URLs) can't be altered after validation, and the runtime can share a single literal instance everywhere.

**Code**
```csharp
string s = "hello";
string upper = s.ToUpper();
Console.WriteLine(s);      // Output: hello  (unchanged)
Console.WriteLine(upper);  // Output: HELLO
```

### 3.2 [Follow-up] When should you use `StringBuilder` instead of `+`?

**Answer**
When building a string over many iterations or an unknown number of appends. Each `+` in a loop allocates a new string and copies all previous characters (O(n²)); `StringBuilder` appends into a growing buffer (amortized O(n)). For a handful of concatenations in a single expression, `+` or interpolation is fine and often faster.

**Explanation**
The compiler turns `a + b + c + d` in one expression into a single `String.Concat(a, b, c, d)`, so no intermediate garbage there. The problem is repeated concatenation across statements/loop iterations. `StringBuilder(capacity)` avoids resizing when you know the size.

**Code**
```csharp
using System.Text;

var sb = new StringBuilder(capacity: 64);
for (int i = 0; i < 5; i++)
    sb.Append(i).Append(',');
sb.Length--;                       // remove trailing comma
Console.WriteLine(sb.ToString());  // Output: 0,1,2,3,4

// Often simpler for joins:
Console.WriteLine(string.Join(",", Enumerable.Range(0, 5))); // Output: 0,1,2,3,4
```

### 3.3 [Medium] What is string interning? What does this print?

**Answer**
Interning stores one canonical instance per distinct string value in a runtime pool. Literals are interned automatically, so two equal literals are the same reference. Runtime-built strings (e.g., from `StringBuilder` or concatenation of non-constants) are separate instances unless you call `string.Intern`.

**Explanation**
Constant folding means `"a" + "b"` is the literal `"ab"` at compile time → interned. `string.Intern` has a cost: interned strings live for the lifetime of the process in the pool (may not be collectible), so don't intern arbitrary user data.

**Code**
```csharp
using System.Text;

string a = "hello";
string b = "hel" + "lo";                          // compile-time constant => interned literal
string part = "hel";
string c = part + "lo";                           // runtime concat => new object
string d = new StringBuilder("hel").Append("lo").ToString();

Console.WriteLine(ReferenceEquals(a, b));         // Output: True
Console.WriteLine(ReferenceEquals(a, c));         // Output: False
Console.WriteLine(a == c);                        // Output: True  (value equality)
Console.WriteLine(ReferenceEquals(a, string.Intern(d))); // Output: True
```

### 3.4 [Hard] Culture and comparison pitfalls — which of these are culture-sensitive?

**Answer**
`string.Equals`/`==`, `Contains(string)`, `Replace(string,string)` are ordinal. `StartsWith(string)`, `EndsWith(string)`, `IndexOf(string)`, `CompareTo`, `string.Compare(a,b)` and `ToUpper()/ToLower()` (culture casing) use the current culture. Always pass an explicit `StringComparison` for non-user-facing logic (typically `Ordinal`/`OrdinalIgnoreCase`).

**Explanation**
Culture-sensitive comparison treats e.g. ignorable characters/ligatures specially. The classic Turkish-I bug: `"I".ToLower()` under `tr-TR` gives dotless `ı`, breaking case-insensitive key checks. Use `ToLowerInvariant()` or `OrdinalIgnoreCase`. In .NET 5+ with ICU on Linux, culture comparison results can differ across OSes.

**Code**
```csharp
using System.Globalization;

string file = "FILE";
Console.WriteLine(file.Equals("file", StringComparison.OrdinalIgnoreCase)); // Output: True

var tr = new CultureInfo("tr-TR");
Console.WriteLine("TITLE".ToLower(tr));          // Output: tıtle  (dotless i)
Console.WriteLine("TITLE".ToLowerInvariant());   // Output: title

// Explicit ordinal avoids culture surprises
bool starts = "http://x".StartsWith("http", StringComparison.Ordinal);
Console.WriteLine(starts);                       // Output: True
```

**Gotchas / follow-ups**
- Dictionary keys: `new Dictionary<string,T>(StringComparer.OrdinalIgnoreCase)`.
- `InvariantGlobalization` mode changes behavior (no ICU).

### 3.5 [Tricky] Does `null + "a"` throw? What about `string.Concat`, `Substring` edge cases, and `char` arithmetic?

**Answer**
`null + "a"` yields `"a"` — null is treated as an empty string in concatenation. But calling any instance method on a null string throws `NullReferenceException`. `'a' + 1` is an `int` (98), not a string, while `"" + 'a' + 1` is `"a1"`.

**Explanation**
`+` with `string` uses `String.Concat`, which maps null to `""`. `char` + `int` is numeric promotion to `int`. `Substring(start)` with `start == Length` returns `""`; `start > Length` throws `ArgumentOutOfRangeException`.

**Code**
```csharp
string? n = null;
Console.WriteLine((n + "a") == "a");   // Output: True
// n.Length;                           // would throw NullReferenceException

Console.WriteLine('a' + 1);            // Output: 98
Console.WriteLine("" + 'a' + 1);       // Output: a1
Console.WriteLine("abc".Substring(3).Length); // Output: 0
Console.WriteLine(string.IsNullOrWhiteSpace("  ")); // Output: True
```

**Gotchas / follow-ups**
- `$"{n}"` also renders null as empty.
- `string.Empty` vs `""`: the same interned instance in practice.

**Interview traps**
- "Strings are value types" — they're immutable reference types.
- Loop concatenation = O(n²); fix with `StringBuilder`/`string.Join`.
- `ReferenceEquals` on strings depends on interning; never rely on it for equality.
- Culture-sensitive default overloads (`StartsWith`, `IndexOf`, `ToLower`) cause locale bugs.
- Interning arbitrary data bloats memory.

**Cheat sheet**
- Immutable, UTF-16, ordinal `==`.
- Literals/constants are interned; runtime strings aren't.
- `StringBuilder` for loops; `string.Join` for sequences.
- Use `StringComparison.Ordinal[IgnoreCase]` for non-linguistic compare.
- Null in `+`/interpolation = empty string.

---

## Topic 4: const vs readonly vs static readonly

**Study notes**

| | `const` | `readonly` (instance) | `static readonly` |
|---|---|---|---|
| Value known at | compile time | runtime (ctor/initializer) | runtime (static init) |
| Implicitly static | yes | no (per instance) | yes |
| Allowed types | primitives, `string`, enums, `null` for refs | any | any |
| Assignment | declaration only | declaration or instance ctor | declaration or static ctor |
| Inlined into callers' IL | **yes** | no | no |
| Usable in `case`, attribute args, default params | yes | no | no |

- `readonly` is **shallow**: the reference can't be reassigned, but the object it points to can be mutated.
- `const` values are copied into every consumer assembly at compile time → changing a `const` in a library requires recompiling consumers (binary-compat trap). `static readonly` is read at runtime.
- `const` string interpolation is allowed when all parts are `const string` (C# 10).

### 4.1 [Easy] What's the difference between `const` and `readonly`?

**Answer**
`const` is a compile-time constant, implicitly static, and its value is baked into the compiled code. `readonly` is a field that can only be assigned in its declaration or in a constructor, so its value is determined at runtime and can differ per instance.

**Explanation**
Use `const` for true universal constants (e.g., `Pi`, number of days in a week); use `readonly` for values that are computed at runtime (e.g., injected dependency, `DateTime.UtcNow` at construction).

**Code**
```csharp
class Config
{
    public const int MaxRetries = 3;                  // compile-time
    public readonly Guid Id;                          // per instance, runtime
    public readonly DateTime Created = DateTime.UtcNow;

    public Config(Guid id) { Id = id; }               // allowed: assign in ctor
    // public void Change() { Id = Guid.NewGuid(); }  // error CS0191
}
```

### 4.2 [Follow-up] When do you need `static readonly` instead of `const`?

**Answer**
When the type isn't allowed as a `const` (e.g., `DateTime`, `TimeSpan`, arrays, custom classes/structs, `Regex`) or when the value is computed at runtime or must be changeable without recompiling consumers. `static readonly` is initialized once per type.

**Explanation**
`decimal` and `string` can be `const`; `DateTime`, `TimeSpan`, and `Guid` cannot. Note that `static readonly` arrays still have mutable elements — expose `IReadOnlyList<T>`/`ImmutableArray<T>` instead.

**Code**
```csharp
static class Limits
{
    public const decimal TaxRate = 0.18m;                       // decimal ok as const
    public static readonly TimeSpan Timeout = TimeSpan.FromSeconds(30);
    public static readonly int[] Primes = { 2, 3, 5 };           // elements still mutable!
}

class Demo
{
    static void Main()
    {
        Limits.Primes[0] = 99;                 // legal
        Console.WriteLine(Limits.Primes[0]);   // Output: 99
        const int k = 3;
        switch (3) { case k: Console.WriteLine("const in case"); break; } // Output: const in case
    }
}
```

### 4.3 [Hard] Why is changing a public `const` in a library a breaking change even if you redeploy only the library?

**Answer**
The compiler copies the literal value of a `const` into the IL of every consumer at compile time. If only the library DLL is updated, consumers still use the old value until recompiled. A `static readonly` field is read from the library at runtime, so a redeploy takes effect.

**Explanation**
This is why framework-style versioned values (e.g., default timeouts, API versions) should be `static readonly` (or properties), while math/protocol constants that will never change can be `const`. The same inlining applies to enum members.

**Code**
```csharp
// Library v1
public static class Lib
{
    public const int Version = 1;                 // inlined into callers
    public static readonly int RuntimeVersion = 1; // looked up at runtime
}

// Consumer compiled against v1:
//   int a = Lib.Version;         // IL contains literal 1
//   int b = Lib.RuntimeVersion;  // IL contains ldsfld
// After replacing Lib.dll with v2 (both set to 2) and NOT recompiling consumer:
//   a == 1 (stale), b == 2
```

**Gotchas / follow-ups**
- Same issue for optional parameter default values (they're baked into the call site).
- Follow-up: "Is `const` faster?" Marginally; JIT can treat `static readonly` primitives as constants after the type is initialized (tiered JIT), so don't choose `const` for perf.

### 4.4 [Tricky] What does this print? (`static readonly` initialization order and shallow readonly)

**Answer**
`X` is `1`, not `11`, because static field initializers run in textual order: when `X` is initialized, `Y` is still `0`. And a `readonly` list can still be mutated.

**Explanation**
Textual order matters inside a class. Cross-class static initialization can also observe default values when there are cycles. `readonly` prevents reassigning the field, not mutating the object.

**Code**
```csharp
class A
{
    public static readonly int X = Y + 1;
    public static readonly int Y = 10;
}

class Demo
{
    private static readonly List<int> Items = new();

    static void Main()
    {
        Console.WriteLine(A.X);   // Output: 1
        Console.WriteLine(A.Y);   // Output: 10

        Items.Add(1);             // legal: object is mutable
        // Items = new();         // error CS0198 outside static ctor
        Console.WriteLine(Items.Count); // Output: 1
    }
}
```

**Interview traps**
- `readonly` ≠ immutable (shallow).
- `const` is baked into consumers → breaking-change risk.
- Static initializers run in textual order; forward references see defaults.
- `const` can't hold `DateTime`/`TimeSpan`/arrays/objects (except `null`/`string`).
- `readonly` fields may be assigned multiple times inside a constructor.

**Cheat sheet**
- `const`: compile-time, static, inlined, primitives/string/enum.
- `readonly`: assigned in declaration/ctor, per-instance.
- `static readonly`: once per type, runtime-evaluated.
- Prefer `static readonly` for public API values that may change.
- Shallow readonly → use immutable collections.

---

## Topic 5: Static members & static constructors

**Study notes**

- A `static` member belongs to the **type**, not instances: fields, methods, properties, events, constructors, nested types. `static class` = `abstract sealed`: no instances, only static members (extension-method hosts, utilities).
- **Static constructor**: parameterless, no access modifier, can't be called manually. Runs **at most once per closed type (per AppDomain/ALC)**, automatically, before the first instance is created or any static member is accessed. The runtime holds a type-initializer lock so it's thread-safe.
- **`beforefieldinit`:** if a class has static field initializers but *no explicit static constructor*, the C# compiler marks it `beforefieldinit` → the runtime may run initialization **any time before the first static field access** (lazier/earlier at JIT's discretion, e.g. at JIT time of the calling method). With an *explicit* static constructor, initialization is "precise": exactly on first access/instantiation.
- **Order**: static field initializers (textual order) → static ctor body. Instance: field initializers (derived class first, then base) → base ctor → derived ctor body.
- **Failure**: if a static constructor/initializer throws, the runtime wraps it in `TypeInitializationException`; the type is then permanently unusable (subsequent accesses throw again).
- Static fields in **generic** types exist **per closed constructed type** (`Cache<int>` ≠ `Cache<string>`).
- Static state = shared global state: thread-safety, testability, memory-lifetime concerns.

### 5.1 [Easy] What does `static` mean on a member, and what is a static class?

**Answer**
A static member belongs to the type itself and is shared by all instances; it's accessed via the type name. A static class can't be instantiated or inherited and can only contain static members — used for helpers and extension methods.

**Explanation**
Static methods have no `this`, so they can't access instance members directly. Static fields live once per type (per closed generic type).

**Code**
```csharp
static class MathUtil
{
    public static int Square(int x) => x * x;
}

class Counter
{
    public static int Instances;
    public Counter() => Instances++;
}

class Demo
{
    static void Main()
    {
        _ = new Counter(); _ = new Counter();
        Console.WriteLine(Counter.Instances);  // Output: 2
        Console.WriteLine(MathUtil.Square(4)); // Output: 16
    }
}
```

### 5.2 [Follow-up] What is a static constructor and when is it called?

**Answer**
It initializes static state; it's parameterless with no access modifier and can't be called explicitly. The CLR runs it once, automatically, before the first instance is created or the first static member is accessed (precisely, because it's explicit).

**Explanation**
Static field initializers run first (in textual order), then the static constructor body. Execution is thread-safe via a type-initialization lock.

**Code**
```csharp
static class Log
{
    public static void W(string s) => Console.WriteLine(s);
}

class P
{
    public static int V = Init("P.V initializer");
    static P() { Log.W("P static ctor"); }
    public static void Touch() { }
    static int Init(string s) { Log.W(s); return 1; }
}

class Demo
{
    static void Main()
    {
        Console.WriteLine("Start");
        P.Touch();
        // Output:
        // Start
        // P.V initializer
        // P static ctor
    }
}
```

### 5.3 [Medium] What is `beforefieldinit` and why does adding an empty static constructor change behavior?

**Answer**
A type with static field initializers but no explicit static constructor gets the `beforefieldinit` flag, letting the runtime initialize it lazily-or-earlier (any time before first static *field* access). Adding an explicit static constructor removes the flag, so initialization happens precisely on first access to any member or instance creation — which is "stricter" and can prevent some JIT optimizations (e.g., beforefieldinit statics can be checked less often).

**Explanation**
Consequences: with `beforefieldinit`, side effects in static initializers may occur at unpredictable times (e.g., when the method touching the field is JIT-compiled) and calling a static *method* doesn't necessarily trigger initialization. If you need deterministic timing (e.g., logging, registration), add a static ctor. For lazy singletons, `Lazy<T>`/static holder idioms are standard.

**Code**
```csharp
class Lazy1               // beforefieldinit: init timing not guaranteed to be at first method call
{
    public static readonly int V = Compute();
    static int Compute() { Console.WriteLine("Lazy1 init"); return 1; }
    public static void M() { }                 // may NOT trigger init
}

class Precise             // explicit static ctor: precise
{
    public static readonly int V = Compute();
    static Precise() { }                        // removes beforefieldinit
    static int Compute() { Console.WriteLine("Precise init"); return 1; }
    public static void M() { }                  // triggers init on first call
}
```

**Gotchas / follow-ups**
- Don't rely on exact timing of `beforefieldinit` types; output can differ between Debug/Release and runtimes.
- Idiom for thread-safe lazy singleton: `private static readonly Lazy<S> _i = new(() => new S());`.

### 5.4 [Hard] Predict the initialization order with static and instance initializers, constructors, and a virtual call from the base constructor.

**Answer**
Static field initializers → static ctor (on first use); then for `new Derived()`: derived instance field initializers → base ctor → derived ctor body. Because derived field initializers run *before* the base ctor, a virtual call from the base ctor sees initialized initializer-assigned fields — but not fields assigned in the derived ctor body.

**Explanation**
This C# ordering differs from C++/Java. Calling virtual methods from constructors is still risky because the derived constructor body hasn't executed yet.

**Code**
```csharp
class Base
{
    public Base()
    {
        Console.WriteLine("Base ctor");
        Init();
    }
    protected virtual void Init() { }
}

class Derived : Base
{
    static int s = L("Derived static field");
    int a = L("Derived field init");
    int b;

    static Derived() { Console.WriteLine("Derived static ctor"); }
    public Derived() { Console.WriteLine("Derived ctor"); b = 7; }

    protected override void Init() => Console.WriteLine($"Init a={a} b={b}");
    static int L(string m) { Console.WriteLine(m); return 1; }
}

class Demo
{
    static void Main() => new Derived();
    // Output:
    // Derived static field
    // Derived static ctor
    // Derived field init
    // Base ctor
    // Init a=1 b=0
    // Derived ctor
}
```

**Gotchas / follow-ups**
- Instance field initializers run before the base ctor call; constructor-body assignments after it.
- Follow-up: "Are initializers of `readonly` fields different?" No, same order.

### 5.5 [Tricky] What happens if a static constructor/initializer throws? And do generic classes share static fields?

**Answer**
The exception is wrapped in `TypeInitializationException` (original in `InnerException`); the type is then dead for the rest of that AppDomain/load context — every later access throws `TypeInitializationException` again, without re-running the initializer. Static fields in a generic class are per closed type, so `Box<int>.Count` and `Box<string>.Count` are separate.

**Explanation**
You can't "retry" static initialization. This commonly appears as a mysterious startup failure (config load in a static ctor) that happens only on first access, with the real error hidden in `InnerException`. Per-closed-type statics are handy for caches (e.g., `EqualityComparer<T>.Default`) but surprise people expecting one shared counter.

**Code**
```csharp
class Bad
{
    public static readonly int V = int.Parse("x");   // throws FormatException
}

class Box<T> { public static int Count; }

class Demo
{
    static void Main()
    {
        for (int i = 0; i < 2; i++)
        {
            try { _ = Bad.V; }
            catch (TypeInitializationException e)
            {
                Console.WriteLine(e.GetType().Name);
            }
        }
        // Output:
        // TypeInitializationException
        // TypeInitializationException   <- type stays broken; initializer is not re-run

        Box<int>.Count = 5;
        Console.WriteLine(Box<string>.Count);   // Output: 0
    }
}
```

**Gotchas / follow-ups**
- Exact message text of the repeated exception can vary; rely on the type, and log the *first* exception's `InnerException` (the real cause).
- Cyclic static initialization across two types can deadlock if each is initialized on a different thread.

**Interview traps**
- "Static ctor runs at app start" — no, lazily on first use (or per `beforefieldinit` rules).
- Static ctor can't have parameters/modifiers or be called manually.
- Exception in static init → `TypeInitializationException`, permanent.
- Generic static fields are per closed type.
- Static mutable state → thread-safety and test isolation issues.

**Cheat sheet**
- Static = belongs to type; `static class` = sealed abstract.
- Order: static fields (textual) → static ctor; instance: derived fields → base ctor → derived ctor.
- Explicit static ctor ⇒ precise timing; none ⇒ `beforefieldinit`.
- Failure ⇒ `TypeInitializationException`, type unusable.
- Generic static ⇒ one copy per closed type.

---

## Topic 6: ref / out / in / ref returns / ref structs

**Study notes**

| Modifier | Caller must init? | Callee must assign? | Callee may write? | Notes |
|---|---|---|---|---|
| (none) | yes | – | local copy | pass by value |
| `ref` | yes | no | yes | alias to caller's variable |
| `out` | no | **yes** (before return) | yes | for extra return values (`TryParse`) |
| `in` | yes | no | **no** (readonly) | pass large structs without copy; `in` at call site optional |
| `ref readonly` | – | – | no | readonly alias param (C# 12) |

- **ref locals / ref returns**: `ref int r = ref arr[2];` `ref int Find(int[] a, int v)`. Return alias must be to storage that outlives the call (array element, field, `ref` param) — not to a local.
- **`ref struct`** (e.g., `Span<T>`, `ReadOnlySpan<T>`, `Utf8JsonReader`): lives only on the stack. Can't be boxed, can't be a field of a class/normal struct, can't be captured by lambdas, can't be used in `async` methods across awaits or iterators, can't be a generic type argument (in C# 12; C# 13 adds `allows ref struct`). C# 11 added `ref` fields in ref structs and `scoped`.
- Can't use `ref`/`out`/`in` parameters in `async` methods or iterators or lambdas capturing them.
- Inline `out var x` and discards `out _` are C# 7+ conveniences.

### 6.1 [Easy] Explain `ref`, `out`, and `in`.

**Answer**
All three pass arguments by reference instead of copying. `ref` requires the caller to initialize and allows read/write; `out` doesn't require initialization but the method must assign before returning; `in` passes by reference as read-only, avoiding copies of large structs.

**Explanation**
The keyword is required at the call site for `ref`/`out`; optional for `in`. Overload resolution differentiates `ref/out/in` from by-value but not `ref` from `out`.

**Code**
```csharp
static void AddOne(ref int x) => x++;
static void Split(string s, out string first, out string rest)
{
    int i = s.IndexOf(' ');
    first = i < 0 ? s : s[..i];
    rest  = i < 0 ? "" : s[(i + 1)..];
}
static double Len(in (double X, double Y) p) => Math.Sqrt(p.X * p.X + p.Y * p.Y);

int n = 1;
AddOne(ref n);
Console.WriteLine(n);                          // Output: 2
Split("hello big world", out var f, out var r);
Console.WriteLine($"{f}|{r}");                 // Output: hello|big world
Console.WriteLine(Len((3, 4)));                // Output: 5
```

### 6.2 [Follow-up] How does the `TryParse` pattern work, and what are `out var` and discards?

**Answer**
`TryXxx` returns a `bool` for success and the result through an `out` parameter, avoiding exceptions for expected failures. `out var x` declares the variable inline; `out _` discards a value you don't need.

**Explanation**
The `out` variable is in scope in the enclosing block, so it's usable after an `if`. On failure `out` gets `default`. Prefer `TryParse` over `Parse` + try/catch for user input (exceptions are expensive).

**Code**
```csharp
if (int.TryParse("123", out var v))
    Console.WriteLine(v + 1);            // Output: 124

bool ok = int.TryParse("abc", out _);    // discard
Console.WriteLine(ok);                   // Output: False

Console.WriteLine(int.TryParse("abc", out int bad) ? "ok" : $"failed, bad={bad}");
// Output: failed, bad=0
```

### 6.3 [Medium] What are ref returns and ref locals? Give a use case and a restriction.

**Answer**
A method can return a reference (alias) to existing storage with `ref` return, and the caller can hold it in a `ref` local to read or write the original location without copying. The returned reference must point to storage that outlives the method (array element, field, ref parameter), never a local variable.

**Explanation**
Use cases: in-place updates in large arrays/structs, avoiding copies in high-performance code (`Span<T>` indexer, `CollectionsMarshal.GetValueRefOrNullRef`). If the caller assigns without `ref`, it gets a copy.

**Code**
```csharp
static ref int MaxElement(int[] a)
{
    int idx = 0;
    for (int i = 1; i < a.Length; i++) if (a[i] > a[idx]) idx = i;
    return ref a[idx];                  // alias to array element
}

var nums = new[] { 3, 9, 4 };
ref int m = ref MaxElement(nums);
m = 100;                                // writes into the array
Console.WriteLine(string.Join(",", nums)); // Output: 3,100,4

int copy = MaxElement(nums);            // no 'ref' => copy
copy = 0;
Console.WriteLine(nums[1]);             // Output: 100
```

### 6.4 [Hard] When does `in` still copy the struct, and how do you prevent it?

**Answer**
If the struct isn't `readonly`, the compiler can't prove that calling its members won't mutate it, so it makes a defensive copy of the `in` argument before calling non-readonly members. Declare the struct `readonly struct` (or members `readonly`) so no copy is needed.

**Explanation**
Also, when you pass an rvalue/non-addressable expression (e.g., a property result) to an `in` parameter, the compiler creates a temporary. `in` only pays off for structs larger than roughly 16 bytes (two machine words); for small structs, by-value is as fast or faster.

**Code**
```csharp
struct BigMutable
{
    public long A, B, C, D;
    public void Bump() => A++;
    public long Sum() => A + B + C + D;     // non-readonly member => defensive copy via 'in'
}

readonly struct BigReadonly
{
    public readonly long A, B, C, D;
    public BigReadonly(long a, long b, long c, long d) => (A, B, C, D) = (a, b, c, d);
    public long Sum() => A + B + C + D;     // no defensive copy
}

class Demo
{
    static long Use(in BigMutable m)   => m.Sum();
    static long Use(in BigReadonly r)  => r.Sum();

    static void Main()
    {
        Console.WriteLine(Use(new BigReadonly(1, 2, 3, 4)));  // Output: 10
        var bm = new BigMutable { A = 1, B = 2, C = 3, D = 4 };
        Console.WriteLine(Use(in bm));                         // Output: 10
    }
}
```

**Gotchas / follow-ups**
- `in` parameters cannot be reassigned; they alias the caller's variable, so mutation by another path is visible in the callee (aliasing).
- Follow-up: "`readonly` member vs readonly struct?" Member-level `readonly` (C# 8) marks only that member as non-mutating.

### 6.5 [Tricky] What is a `ref struct` and what are its restrictions? Why can't you use `Span<T>` in an `async` method across an `await`?

**Answer**
A `ref struct` is a struct that must live on the stack (e.g., `Span<T>`), enforced by the compiler. It can't be boxed, be a field of a class or regular struct, be captured by a lambda, or live across `await`/`yield`, because the async/iterator state machine would hoist locals into a heap object.

**Explanation**
`Span<T>` can point at stack memory (`stackalloc`); if it could escape to the heap, it'd become a dangling pointer. Compiler "safe-context" rules also stop you returning a span over a local. C# 12 prohibits `ref struct` as a generic argument and (before C# 13) from implementing interfaces; C# 13/.NET 9 relaxes this with `allows ref struct`. Use `Memory<T>` when you need to store/await across async code.

**Code**
```csharp
ref struct Window
{
    private readonly Span<int> _data;
    public Window(Span<int> data) => _data = data;
    public int Sum() { int s = 0; foreach (var x in _data) s += x; return s; }
}

class Demo
{
    static int SumStack()
    {
        Span<int> buf = stackalloc int[4] { 1, 2, 3, 4 };   // no heap allocation
        var w = new Window(buf);
        return w.Sum();
    }

    // static Span<int> Bad() { Span<int> s = stackalloc int[2]; return s; } // error CS8352
    // static async Task M() { Span<int> s = default; await Task.Yield(); _ = s.Length; } // error
    static async Task<int> Fine(Memory<int> m) { await Task.Yield(); return m.Span[0]; }

    static void Main() => Console.WriteLine(SumStack());    // Output: 10
}
```

**Gotchas / follow-ups**
- `object o = span;` → error (can't box).
- `Span<T>` slicing is O(1) and allocation-free (`arr.AsSpan(1, 3)`).

**Interview traps**
- `ref` vs `out`: init requirement differs; both need the keyword at the call site.
- Passing a reference type "by reference" (`ref`) is only needed to *reassign* it for the caller.
- `in` on non-readonly structs can still copy (defensive copies).
- Returning `ref` to a local is illegal.
- `ref struct`s can't be boxed/captured/used across `await`.

**Cheat sheet**
- `ref`: init'd in, read/write. `out`: must assign. `in`: readonly alias.
- `out var x` / `out _`.
- `ref return` → alias to long-lived storage; caller needs `ref` local to keep alias.
- `readonly struct` → no defensive copies.
- `ref struct` = stack-only (`Span<T>`); use `Memory<T>` for async.

---

## Topic 7: Nullable value types & nullable reference types

**Study notes**

- **Nullable value types**: `int?` = `Nullable<int>`, a struct `{ bool HasValue; T Value }`. A real *type change* with runtime representation. `.Value` on empty throws `InvalidOperationException`; `GetValueOrDefault()` doesn't.
- **Lifted operators**: arithmetic with a `null` operand → `null`; comparison `<,>,<=,>=` → `false`; `==` is `true` only if both null or both equal; `!=` is its negation. `bool?` has three-valued logic for `&`, `|`.
- Operators: `??` (null-coalescing), `??=`, `?.`/`?[]` (null-conditional), `!` (null-forgiving, compile-time only).
- **Nullable reference types (NRT, C# 8)**: *compile-time-only* annotations/flow analysis. `string` = "not supposed to be null", `string?` = "may be null". Enable via `<Nullable>enable</Nullable>` or `#nullable enable`. Violations are **warnings** (CS8600-series), not errors, and there is **no runtime check**; `string?` and `string` are the same runtime type.
- Attributes refine flow analysis: `[NotNullWhen(true)]`, `[MaybeNullWhen(false)]`, `[NotNull]`, `[MemberNotNull]`, `[AllowNull]`, `[DisallowNull]`.
- EF Core reads NRT annotations: `string` property → `NOT NULL` column; `string?` → nullable column.

### 7.1 [Easy] What is `int?` and how do you use it?

**Answer**
`int?` is shorthand for `Nullable<int>`, a value type that can hold an `int` or `null`. Check with `HasValue`/`is not null`, get the value with `.Value`, or supply a default with `??` or `GetValueOrDefault()`.

**Explanation**
Typical use: optional database columns, optional form fields. Accessing `.Value` when empty throws `InvalidOperationException`.

**Code**
```csharp
int? age = null;
Console.WriteLine(age.HasValue);          // Output: False
Console.WriteLine(age ?? -1);             // Output: -1
Console.WriteLine(age.GetValueOrDefault()); // Output: 0
age = 30;
if (age is int a) Console.WriteLine(a + 1); // Output: 31

try { int? e = null; _ = e.Value; }
catch (InvalidOperationException) { Console.WriteLine("no value"); } // Output: no value
```

### 7.2 [Follow-up] How do `??`, `??=`, and `?.` work together?

**Answer**
`a ?? b` returns `a` if non-null else `b`; `a ??= b` assigns `b` only if `a` is null; `a?.M()` calls `M` only if `a` is non-null, otherwise the whole expression is `null`. Combined, they provide concise null handling.

**Explanation**
`?.` short-circuits the entire member-access chain. When the result type is a value type, the result becomes nullable (`s?.Length` is `int?`). `??` is right-associative.

**Code**
```csharp
string? name = null;
int? len = name?.Length;                 // null (int?)
Console.WriteLine(len is null);          // Output: True

name ??= "default";
Console.WriteLine(name);                 // Output: default

List<int>? list = null;
Console.WriteLine(list?.Count ?? 0);     // Output: 0
Console.WriteLine(list?[0]);             // Output: (empty line, null)
```

### 7.3 [Medium] How do nullable value types differ from nullable reference types?

**Answer**
`int?` is a distinct runtime type (`Nullable<int>`) with different layout and behavior. `string?` is just an annotation: same runtime type as `string`, enforced only by compiler warnings, and can still be null at runtime when the annotation is bypassed (reflection, serialization, `!`, legacy code).

**Explanation**
Consequences: you can't write `typeof(string?)`; you can't `where T : class?`-check at runtime. NRT adds no overhead and no exceptions; a "non-nullable" property may be null after JSON deserialization or EF materialization unless `required`/initialized. `Nullable<T>` allocates nothing but costs an extra byte of padding/alignment.

**Code**
```csharp
#nullable enable
string? maybe = null;
string definitely = "x";

// definitely = maybe;                 // warning CS8600: converting null literal or possible null
definitely = maybe ?? "fallback";      // OK
Console.WriteLine(definitely);         // Output: fallback

if (maybe is not null)
    Console.WriteLine(maybe.Length);   // flow analysis: non-null here

string forced = maybe!;                // suppress warning; still null at runtime
Console.WriteLine(forced is null);     // Output: True

Console.WriteLine(typeof(int?));       // Output: System.Nullable`1[System.Int32]
```

### 7.4 [Hard] Predict lifted-operator results, including `bool?` logic.

**Answer**
Arithmetic with `null` gives `null`; relational `<`/`>` with `null` give `false`; `null == null` is `true`. For `bool?`, `null | true` = `true`, `null & false` = `false`, otherwise `null` (three-valued logic). `if (nb)` doesn't compile — need `== true`.

**Explanation**
Note `a != b` is `true` when exactly one is null, even though `a < b` and `a > b` are both false, so "not less and not greater" doesn't imply "equal". `&&`/`||` aren't defined for `bool?` (only `&`/`|`).

**Code**
```csharp
int? a = null, b = 5;
Console.WriteLine(a + b);          // Output: (blank: null)
Console.WriteLine(a < b);          // Output: False
Console.WriteLine(a > b);          // Output: False
Console.WriteLine(a == b);         // Output: False
Console.WriteLine(a != b);         // Output: True
Console.WriteLine(a == null);      // Output: True

bool? t = true, f = false, n = null;
Console.WriteLine(n | t);          // Output: True
Console.WriteLine(n & f);          // Output: False
Console.WriteLine(n & t);          // Output: (blank: null)
Console.WriteLine(n == true);      // Output: False
```

**Gotchas / follow-ups**
- Sorting with `Comparer<int?>.Default`: `null` sorts before any value.
- `Nullable.Compare` / `Nullable.Equals` follow that convention.

### 7.5 [Tricky] If NRT is enabled, can a non-nullable `string` property still be null at runtime? How do you handle it?

**Answer**
Yes. NRT is only compile-time analysis, so deserializers, ORMs, reflection, `default!`, and arrays (`new string[3]` has null elements) can produce nulls. Mitigate with `required` members, constructors that validate, `ArgumentNullException.ThrowIfNull`, and nullable attributes.

**Explanation**
Common patterns: `public string Name { get; set; } = null!;` (tell compiler "someone else initializes this", e.g., EF Core), or `required string Name { get; init; }` (C# 11) so object initializers must set it. EF Core navigation properties are often declared `= null!` or `Customer? Customer`. `[NotNullWhen(true)]` documents `TryGet` patterns.

**Code**
```csharp
#nullable enable
using System.Diagnostics.CodeAnalysis;

class Person
{
    public required string Name { get; init; }
    public string Nick { get; set; } = null!;     // promise to set later; null at runtime if not
}

static class Lookup
{
    public static bool TryGet(string key, [NotNullWhen(true)] out string? value)
    {
        value = key == "a" ? "alpha" : null;
        return value is not null;
    }
}

class Demo
{
    static void Main()
    {
        var p = new Person { Name = "Ann" };
        Console.WriteLine(p.Nick is null);          // Output: True  (despite 'string')

        if (Lookup.TryGet("a", out var v))
            Console.WriteLine(v.Length);            // Output: 5  (no warning thanks to attribute)

        string[] arr = new string[2];
        Console.WriteLine(arr[0] is null);          // Output: True
        ArgumentNullException.ThrowIfNull(p);       // runtime guard for public APIs
    }
}
```

**Interview traps**
- NRT gives warnings, not errors, and nothing at runtime.
- `int? == null` vs `.Value`: `.Value` throws `InvalidOperationException` (not `NullReferenceException`).
- `a < b` false doesn't imply `a >= b` true with nulls (both false).
- `null!` hides bugs; use `required` or constructors.
- Boxed `int?` becomes `int` or `null` (Topic 2).

**Cheat sheet**
- `T?` for structs = `Nullable<T>` (runtime type); for classes = annotation only.
- `??`, `??=`, `?.`, `!`.
- Lifted ops: arithmetic → null; relational → false; `==` null-aware.
- Enable `<Nullable>enable</Nullable>` project-wide; treat warnings as errors for new code.
- Use `[NotNullWhen]`, `required`, `ThrowIfNull`.

---

## Topic 8: var vs dynamic vs object

**Study notes**

| | `var` | `object` | `dynamic` |
|---|---|---|---|
| Typing | static, inferred at compile time | static type `object` | static type `dynamic` (erased to `object` in IL) |
| Member lookup | compile time (on inferred type) | compile time (only `object` members) | **runtime** (DLR) |
| Errors | compile errors | compile errors (cast needed) | `RuntimeBinderException` at runtime |
| Value types | no boxing | boxing | boxing |
| Perf | native | cast/unbox | binder + call-site caching, slower |

- `var` is **not** a type; it requires an initializer; local variables only (not fields, parameters, return types). Required for anonymous types. `var x = new List<int>()` = `List<int>`.
- Since C# 10 lambdas/method groups have a *natural type*: `var f = (int x) => x * 2;` is `Func<int,int>`.
- `dynamic` uses the Dynamic Language Runtime: `Microsoft.CSharp.RuntimeBinder`. Operations on a `dynamic` receiver return `dynamic`. Useful for COM interop (Office), `ExpandoObject`, JSON/IronPython interop; avoid in normal code. Extension methods and lambdas conversions are not resolved on dynamic receivers.

### 8.1 [Easy] Is `var` the same as `dynamic`?

**Answer**
No. `var` is compile-time type inference — the variable is strongly typed to whatever the initializer's type is. `dynamic` defers member binding to runtime, so type errors surface at runtime.

**Explanation**
`var` is just syntactic convenience: the IL is identical to writing the type. `var x = 5; x = "a";` is a compile error. With `dynamic d = 5; d = "a";` it compiles.

**Code**
```csharp
var v = 5;                // int
// v = "text";            // compile error CS0029
dynamic d = 5;
d = "text";               // fine
Console.WriteLine(v.GetType());  // Output: System.Int32
Console.WriteLine(d.GetType());  // Output: System.String
```

### 8.2 [Follow-up] When can't you use `var`, and when is it required?

**Answer**
You can't use it for fields, parameters, return types, or without an initializer (or initialized only with `null`). It's required for anonymous types, and handy with long generic types or when the type is obvious from the right-hand side.

**Explanation**
Style guideline: use `var` when the type is apparent (`new`, cast, `as`) and an explicit type otherwise, per team convention. Target-typed `new()` (C# 9) is the alternative for fields: `private readonly List<int> _x = new();`.

**Code**
```csharp
var anon = new { Id = 1, Name = "A" };      // anonymous type: var required
Console.WriteLine(anon.Name);                // Output: A

var fn = (int x) => x * 2;                   // natural type: Func<int,int>
Console.WriteLine(fn(4));                    // Output: 8

// var nothing = null;                       // error CS0815
// var unset;                                // error CS0818
List<string> names = new();                  // target-typed new
```

### 8.3 [Medium] How does `object` differ from `dynamic` when calling a method that doesn't exist?

**Answer**
With `object`, the compiler rejects calls to members `object` doesn't have; you'd have to cast. With `dynamic`, the call compiles and is resolved at runtime; if the method doesn't exist you get a `RuntimeBinderException`.

**Explanation**
`dynamic` trades compile-time safety for flexibility. It also costs: call-site binder creation, then cached delegates (cache makes later calls faster but still slower than static calls). It also infects: any expression involving a dynamic operand is dynamic.

**Code**
```csharp
using Microsoft.CSharp.RuntimeBinder;

object o = "hello";
// o.Length;                              // compile error: object has no Length
Console.WriteLine(((string)o).Length);    // Output: 5

dynamic d = "hello";
Console.WriteLine(d.Length);              // Output: 5   (resolved at runtime)
try { d.NoSuchMethod(); }
catch (RuntimeBinderException) { Console.WriteLine("binder error"); } // Output: binder error
```

### 8.4 [Hard] How does overload resolution differ for `object` vs `dynamic` arguments?

**Answer**
Overload resolution is done at compile time using the *static* type for `object`, but at runtime using the *actual runtime type* for `dynamic`. So the same string value picks the `object` overload via an `object` variable, and the `string` overload via a `dynamic` variable.

**Explanation**
This is runtime multi-dispatch ("poor man's double dispatch/visitor"). Extension methods are not found for dynamic receivers, because extension resolution needs compile-time context; and any dynamic argument makes the invocation dynamic, with the return type `dynamic` (except for certain cases where the compiler can determine the type).

**Code**
```csharp
class Demo
{
    static string Print(object x) => "object";
    static string Print(string x) => "string";
    static string Print(int x)    => "int";

    static void Main()
    {
        object o = "hi";
        dynamic d = "hi";
        Console.WriteLine(Print(o));   // Output: object
        Console.WriteLine(Print(d));   // Output: string
        d = 5;
        Console.WriteLine(Print(d));   // Output: int

        dynamic d2 = new List<int> { 1, 2, 3 };
        // d2.Select(x => x);          // runtime error: extension methods unsupported on dynamic
        Console.WriteLine(System.Linq.Enumerable.Count(d2)); // Output: 3 (static-method call, dynamic arg)
    }
}
```

**Gotchas / follow-ups**
- Lambdas can't be passed to dynamic-dispatched calls without a cast to a delegate type.
- Local functions can't be invoked with dynamic arguments (compile error) — use methods.
- Follow-up: "Where would you really use dynamic?" COM interop, `ExpandoObject`, and `DynamicObject` subclasses, scripting hosts.

### 8.5 [Tricky] What are the static types of `r1` and `r2`? What if you `var` a dynamic expression?

**Answer**
`r1 = d + 1` is `dynamic` (any operation with a dynamic operand is dynamic), even though at runtime it holds an `int`. `var r2 = (object)5` has the static type `object`. `var` on a dynamic expression infers `dynamic`.

**Explanation**
Static type vs runtime type is the core idea. `GetType()` shows the runtime type; the compiler's static type governs overload resolution and what members you can call without a cast. Mixing dynamic into a codebase "infects" downstream code and loses IntelliSense and refactoring safety.

**Code**
```csharp
dynamic d = 5;
var r1 = d + 1;            // static type: dynamic
var r2 = (object)5;        // static type: object
Console.WriteLine(r1.GetType()); // Output: System.Int32
Console.WriteLine(r2.GetType()); // Output: System.Int32

// r2 + 1;                  // compile error: object doesn't support +
Console.WriteLine(r1 + 1);   // Output: 7

int typed = d;               // implicit runtime conversion from dynamic
Console.WriteLine(typed);    // Output: 5
```

**Interview traps**
- "`var` is a variant/late-bound type" — wrong; it's fully static.
- Errors with `dynamic` appear only at runtime (and only on the executed path).
- Extension methods and lambdas don't work on dynamic receivers.
- `dynamic` is `object` in IL plus attributes — boxed value types.
- Overload resolution result can differ between `object` and `dynamic` (compile time vs runtime).

**Cheat sheet**
- `var` = compile-time inferred, strong typing.
- `object` = universal base, needs casts, boxes value types.
- `dynamic` = runtime-bound via DLR; `RuntimeBinderException`.
- Anonymous types need `var`.
- Avoid `dynamic` except interop/dynamic data.

---

## Topic 9: struct vs class vs record vs record struct

**Study notes**

| | `class` | `struct` | `record` (class) | `record struct` |
|---|---|---|---|---|
| Kind | reference | value | reference | value |
| Default equality | reference | field-wise via reflection-ish `ValueType.Equals` (slow, boxes) | **value-based** (generated) | **value-based** (generated) |
| Inheritance | yes | no (interfaces only) | yes (from records/`object`) | no |
| `with` expression | no | struct: yes (C# 10) | yes (copy ctor) | yes |
| Positional ctor/Deconstruct | manual | manual | generated | generated |
| Properties from positional | – | – | `init` (immutable) | **mutable** `{get;set;}` (unless `readonly record struct`) |
| `ToString` | type name | type name | `Name { P = v, ... }` | `Name { P = v, ... }` |
| `null` possible | yes | no | yes | no |

- Struct guidelines (MS design): small (≈ ≤16 bytes), logically a single value, immutable, short-lived, not frequently boxed.
- Records generate: `Equals`, `GetHashCode`, `==`/`!=`, `ToString`, `PrintMembers`, `Deconstruct` (positional), `<Clone>$` copy method, and `EqualityContract` (for class records; ensures `Base` ≠ `Derived` instances even if fields match).
- `with` makes a **shallow** copy then applies `init`s.
- `struct` always has a "zero" default (`default(T)`), and constructors can be bypassed (arrays, `default`).

### 9.1 [Easy] What's the core difference between `struct` and `class`?

**Answer**
A struct is a value type (copied on assignment, no inheritance, can't be null) and a class is a reference type (shared reference, supports inheritance, can be null). Use structs for small, immutable value-like data; classes for most other things.

**Explanation**
Struct arrays are stored inline (good cache locality, no per-element allocation); class arrays store references to separately allocated objects.

**Code**
```csharp
Vec a = new() { X = 1, Y = 2 };
Vec b = a; b.X = 9;
Console.WriteLine(a.X);          // Output: 1

Node n1 = new() { V = 1 };
Node n2 = n1; n2.V = 9;
Console.WriteLine(n1.V);         // Output: 9

struct Vec { public double X, Y; }
class Node { public int V; public Node? Next; }
```

### 9.2 [Follow-up] When should you choose a struct over a class?

**Answer**
When it's small (around 16 bytes or less), represents a single logical value (like `Point`, `Money`, `DateTime`), is immutable, and is created in large numbers or short-lived so avoiding heap allocation pays off. Avoid structs that are large, mutable, or often boxed/passed as interfaces.

**Explanation**
Large structs are costly to copy; frequently boxed structs lose the allocation benefit. Structs can't be used as polymorphic base types. Measure with BenchmarkDotNet; don't assume.

**Code**
```csharp
public readonly struct Money : IEquatable<Money>
{
    public decimal Amount { get; }
    public string Currency { get; }
    public Money(decimal amount, string currency) => (Amount, Currency) = (amount, currency);

    public bool Equals(Money other) => Amount == other.Amount && Currency == other.Currency;
    public override bool Equals(object? obj) => obj is Money m && Equals(m);
    public override int GetHashCode() => HashCode.Combine(Amount, Currency);
    public static bool operator ==(Money l, Money r) => l.Equals(r);
    public static bool operator !=(Money l, Money r) => !l.Equals(r);
}
```

### 9.3 [Medium] What does a `record` give you? Show `with`, equality, `ToString`, deconstruction.

**Answer**
A `record` (class) is a reference type with compiler-generated value-based equality, `ToString`, `GetHashCode`, a copy constructor/`with` support, and for positional records, `init` properties and `Deconstruct`. It's ideal for immutable DTOs and value objects.

**Explanation**
`==` compares members, not references. `with` creates a *shallow* copy with changed properties — original is untouched. Positional properties are `init`-only by default.

**Code**
```csharp
public record Person(string Name, int Age);

class Demo
{
    static void Main()
    {
        var p1 = new Person("Ann", 30);
        var p2 = p1 with { Age = 31 };
        Console.WriteLine(p1);                         // Output: Person { Name = Ann, Age = 30 }
        Console.WriteLine(p2);                         // Output: Person { Name = Ann, Age = 31 }
        Console.WriteLine(p1 == new Person("Ann", 30)); // Output: True
        Console.WriteLine(ReferenceEquals(p1, p2));    // Output: False

        var (name, age) = p1;
        Console.WriteLine($"{name} {age}");            // Output: Ann 30
    }
}
```

### 9.4 [Hard] How does record equality behave with inheritance and with collection members?

**Answer**
Record equality includes the runtime type via `EqualityContract`, so a `Base` record and a `Derived` record with the same members are *not* equal. Members are compared with `EqualityComparer<T>.Default`, so collection-typed members (e.g., `List<int>`) are compared by **reference**, not by content — two records with equal-content but different list instances are unequal.

**Explanation**
To get content equality for collections, override `Equals`/`GetHashCode` manually or wrap the collection in a type with value equality. The hash code similarly depends on member hash codes, so mutating a member after inserting the record into a `HashSet` breaks lookups.

**Code**
```csharp
record Animal(string Name);
record Dog(string Name) : Animal(Name);
record Bag(List<int> Items);

class Demo
{
    static void Main()
    {
        Animal a = new Animal("x");
        Animal d = new Dog("x");
        Console.WriteLine(a == d);                 // Output: False  (different EqualityContract)
        Console.WriteLine(a.Equals(new Animal("x"))); // Output: True

        var b1 = new Bag(new List<int> { 1 });
        var b2 = new Bag(new List<int> { 1 });
        Console.WriteLine(b1 == b2);               // Output: False  (list compared by reference)
        var b3 = b1 with { };                      // shallow copy: same list instance
        Console.WriteLine(b1 == b3);               // Output: True
    }
}
```

**Gotchas / follow-ups**
- Records are only "immutable" shallowly; `init` doesn't protect nested mutable objects.
- Follow-up: "Can a class record inherit from a class?" No — only from `object` or another record.

### 9.5 [Tricky] Why are `record struct` positional properties mutable, and how can a struct's constructor validation be bypassed?

**Answer**
`record struct` generates read/write properties for positional parameters by default (unlike `record class`, which generates `init`). Use `readonly record struct` for immutability. And any struct can be created without running a constructor — `default(T)`, `new T[n]`, or fields of a containing type — so validation in constructors can't guarantee valid state.

**Explanation**
This is why "invalid default" must be a valid or detectable state for structs (e.g., treat `default` as "empty"). Since C# 10 you can declare a parameterless struct constructor, but it still isn't run for `default` or array allocation.

**Code**
```csharp
record struct Pt(int X, int Y);
readonly record struct RPt(int X, int Y);

readonly struct PositiveAmount
{
    public decimal Value { get; }
    public PositiveAmount(decimal v)
    {
        if (v <= 0) throw new ArgumentOutOfRangeException(nameof(v));
        Value = v;
    }
}

class Demo
{
    static void Main()
    {
        var p = new Pt(1, 2);
        p.X = 5;                                  // legal: mutable
        Console.WriteLine(p);                     // Output: Pt { X = 5, Y = 2 }
        // var r = new RPt(1, 2); r.X = 5;        // error: init-only/readonly

        var bad = default(PositiveAmount);        // ctor never runs
        Console.WriteLine(bad.Value);             // Output: 0  (invalid state escapes)
        var arr = new PositiveAmount[2];
        Console.WriteLine(arr[0].Value);          // Output: 0
    }
}
```

**Interview traps**
- Record equality: collections by reference; `Base != Derived` by contract.
- `with` is a shallow copy.
- `struct` default equality is slow/boxing unless `IEquatable<T>` is implemented (record struct generates it).
- Struct constructor validation is bypassable via `default`.
- "Records are always immutable" — only with init-only/readonly and shallowly.

**Cheat sheet**
- class=ref/mutable by default; struct=value; record=class with value equality; record struct=struct with value equality.
- `record` positional → `init` props; `record struct` positional → mutable props.
- `readonly record struct` for immutable value objects.
- `with` = shallow copy + init.
- Struct: small, immutable, `IEquatable<T>`.

---

## Topic 10: Enums & [Flags]

**Study notes**

- An `enum` is a value type deriving from `System.Enum`, backed by an integral type (default `int`; can be `byte`, `short`, `long`, `uint`, etc.). Members are named constants (inlined at compile time like `const`).
- `default(MyEnum)` is `0`, **even if no member has value 0**. An enum variable can hold *any* value of its underlying type — casting isn't validated. Validate with `Enum.IsDefined`.
- `Enum.Parse`/`TryParse` accept names *and numeric strings* (e.g., `"99"` parses successfully even if undefined). `Enum.GetValues<T>()` (generic, .NET 5+), `Enum.GetName`, `Enum.GetNames<T>()`.
- **`[Flags]`**: a bit-field enum. Use powers of two (`1, 2, 4, 8…`), include `None = 0`, combine with `|`, test with `&`/`HasFlag`, remove with `&= ~flag`, toggle with `^=`. The attribute changes `ToString()`/`Parse` (comma-separated names) but **doesn't** enforce anything.

```
Read=0001  Write=0010  Exec=0100
Read|Write = 0011
(p & Write) != 0  -> has Write
p &= ~Read        -> 0010
```

### 10.1 [Easy] What is an enum and what's its default underlying type?

**Answer**
An enum is a named set of integral constants — a value type whose underlying type defaults to `int` (changeable, e.g., `enum Level : byte`). It improves readability and type safety over magic numbers.

**Explanation**
Values auto-increment from 0 unless specified. Enum values can be cast to/from their underlying type explicitly.

**Code**
```csharp
Console.WriteLine((int)Status.Closed);               // Output: 6
Console.WriteLine(Status.Active);                    // Output: Active
Console.WriteLine(Enum.GetUnderlyingType(typeof(Status))); // Output: System.Byte
foreach (var s in Enum.GetValues<Status>()) Console.Write(s + " "); // Output: Pending Active Closed

enum Status : byte { Pending, Active = 5, Closed }   // 0, 5, 6
```

### 10.2 [Follow-up] Can an enum variable hold a value that isn't a defined member? How do you validate?

**Answer**
Yes. Casting any integer to an enum type compiles and succeeds at runtime, producing an undefined value. Validate with `Enum.IsDefined`, or a `switch` with a default that throws; for external input, never trust a raw cast.

**Explanation**
`Enum.TryParse` also accepts numeric strings, so `"99"` parses to `(Status)99` unless you check `IsDefined` afterwards. `IsDefined` doesn't work for combined `[Flags]` values.

**Code**
```csharp
var c = (Color)42;                                   // allowed
Console.WriteLine(c);                                // Output: 42
Console.WriteLine(Enum.IsDefined(typeof(Color), c)); // Output: False

Console.WriteLine(Enum.TryParse<Color>("99", out var parsed)); // Output: True  (!)
Console.WriteLine(Enum.IsDefined(parsed));           // Output: False  (generic overload, .NET 5+)
Console.WriteLine(default(Color));                   // Output: 0  (no member equals 0)

enum Color { Red = 1, Green = 2 }
```

### 10.3 [Medium] How do you use a `[Flags]` enum (set, test, clear, toggle)?

**Answer**
Declare members as powers of two, combine with `|`, test with `(value & flag) == flag` or `HasFlag`, clear with `&= ~flag`, toggle with `^= flag`. The `[Flags]` attribute makes `ToString` print `Read, Write`.

**Explanation**
Define composite members (`All = Read | Write | Exec`) for convenience. Always define `None = 0`. `HasFlag` is optimized (JIT intrinsic) in modern .NET, but `(v & f) == f` is universally fast and clear.

**Code**
```csharp
Perm p = Perm.Read | Perm.Write;
Console.WriteLine(p);                         // Output: Read, Write
Console.WriteLine(p.HasFlag(Perm.Write));     // Output: True
p &= ~Perm.Read;                              // clear
Console.WriteLine(p);                         // Output: Write
p ^= Perm.Exec;                               // toggle on
Console.WriteLine(p);                         // Output: Write, Exec
Console.WriteLine(Perm.All);                  // Output: All

[Flags]
enum Perm { None = 0, Read = 1, Write = 2, Exec = 4, All = Read | Write | Exec }
```

### 10.4 [Hard] What does `ToString` give with and without `[Flags]`, and what's the `HasFlag(None)` gotcha?

**Answer**
Without `[Flags]`, a combined or unknown value prints as its number (`3`); with `[Flags]`, it prints matched names joined by commas, but if any bits remain unmatched it falls back to the number. `x.HasFlag(None)` (value 0) is always `true`, so use `x == None` to test emptiness.

**Explanation**
Typical bug: `if (perms.HasFlag(Perm.None))` is always true. Another: non-power-of-two flag values (e.g., `Read = 1, Write = 2, Both = 3, Admin = 3`) create ambiguity; `ToString` prefers a defined name for the whole value, then decomposes into the largest matching flags.

**Code**
```csharp
Console.WriteLine((Plain)3);          // Output: 3
Console.WriteLine((Fl)3);             // Output: A, B
Console.WriteLine((Fl)8);             // Output: 8
Console.WriteLine((Fl)9);             // Output: 9   (bit 8 unmatched -> whole number)
Console.WriteLine(Fl.A.HasFlag(Fl.None)); // Output: True  (zero flag is always "present")
Console.WriteLine(Fl.A == Fl.None);   // Output: False

enum Plain { A = 1, B = 2 }
[Flags] enum Fl { None = 0, A = 1, B = 2 }
```

**Gotchas / follow-ups**
- Enum `switch` isn't exhaustive by the compiler for undefined values — add a `default` guard.
- Follow-up: "How would you store flags in SQL?" As an integer column (bitwise ops in queries) — EF Core maps enums to their underlying int by default.

### 10.5 [Tricky] What are the risks of persisting or serializing enums?

**Answer**
Stored as integers, reordering/inserting enum members silently changes meaning of existing data; stored as strings, renaming a member breaks old data. Fix by assigning explicit numeric values and never reusing/renumbering them, and by choosing one representation deliberately (EF Core `HasConversion<string>()`, `JsonStringEnumConverter`).

**Explanation**
`System.Text.Json` serializes enums as numbers by default; use `JsonStringEnumConverter` for names. Consumers (APIs, DB, other services) should treat unknown values gracefully (e.g., `Unknown = 0` default sentinel because `default` of an enum is 0).

**Code**
```csharp
using System.Text.Json;
using System.Text.Json.Serialization;

enum OrderState { Unknown = 0, Placed = 1, Shipped = 2 }   // explicit, never renumber

class Order { public OrderState State { get; set; } }

class Demo
{
    static void Main()
    {
        var o = new Order { State = OrderState.Shipped };
        Console.WriteLine(JsonSerializer.Serialize(o));   // Output: {"State":2}

        var opts = new JsonSerializerOptions { Converters = { new JsonStringEnumConverter() } };
        Console.WriteLine(JsonSerializer.Serialize(o, opts)); // Output: {"State":"Shipped"}

        Console.WriteLine(JsonSerializer.Deserialize<Order>("{}")!.State); // Output: Unknown
    }
}
```

**Interview traps**
- Enum casts are unchecked; any int is "valid".
- `HasFlag(None)` is always true.
- `[Flags]` doesn't enforce powers of two or define `None`.
- `Enum.TryParse` accepts numeric strings.
- Renumbering enum members breaks persisted data.

**Cheat sheet**
- Underlying type default `int`; `default` = 0.
- Validate: `Enum.IsDefined`.
- Flags: powers of two, `|`, `&`, `&= ~f`, `^=`.
- `None = 0`; test emptiness with `== None`.
- Use `Unknown = 0` sentinel; explicit values for persisted enums.

---

## Topic 11: Equality & Object members

**Study notes**

- **`==`**: operator, resolved at **compile time** by static types. For reference types default = reference equality (unless overloaded, as in `string`, records). For primitives/structs with overloaded `==`, value comparison.
- **`Equals(object)`**: virtual. `object` default = reference equality; `ValueType.Equals` = field-by-field (reflection-based/slow if not overridden; boxes); `string`, records, tuples override with value equality.
- **`ReferenceEquals(a, b)`**: always identity; never overridable.
- **`GetHashCode()` contract**: equal objects ⇒ equal hash codes (reverse not required); hash must be stable for the object's lifetime while stored in a hash-based collection; should be fast and well distributed. Override `Equals` and `GetHashCode` **together**.
- **`IEquatable<T>`**: strongly typed `Equals(T)`; avoids boxing for structs; `EqualityComparer<T>.Default` uses it, so `Dictionary`/`HashSet`/`List.Contains` benefit.
- Use `HashCode.Combine(a, b, …)` (.NET Core 2.1+) for hashes. Custom comparers: `IEqualityComparer<T>`.
- **`System.Object` members**: `ToString()`, `Equals(object)`, `GetHashCode()`, `GetType()`, static `Equals(object,object)`, static `ReferenceEquals`, protected `MemberwiseClone()` (shallow copy), protected `Finalize()`.
- `double.NaN == double.NaN` is `false` but `double.NaN.Equals(double.NaN)` is `true` (so collections work).

### 11.1 [Easy] What's the difference between `==` and `.Equals()`?

**Answer**
`==` is an operator resolved at compile time on the static types (reference equality by default for classes, but overloaded for `string`, records, etc.); `.Equals` is a virtual method that dispatches on the runtime type and can be overridden for value equality. For `string` both compare content.

**Explanation**
`Equals` on a null reference throws `NullReferenceException`; `==` and `object.Equals(a, b)` are null-safe. `ReferenceEquals` always checks identity.

**Code**
```csharp
string a = "abc";
string b = new string(new[] { 'a', 'b', 'c' });

Console.WriteLine(a == b);                    // Output: True   (string overloads ==)
Console.WriteLine(a.Equals(b));               // Output: True
Console.WriteLine(ReferenceEquals(a, b));     // Output: False

var l1 = new List<int>(); var l2 = new List<int>();
Console.WriteLine(l1 == l2);                  // Output: False  (reference)
Console.WriteLine(l1.Equals(l2));             // Output: False
```

### 11.2 [Follow-up] Which members does every .NET object inherit from `System.Object`, and which should you override?

**Answer**
`ToString`, `Equals`, `GetHashCode`, `GetType`, plus protected `MemberwiseClone` and `Finalize` (and static `ReferenceEquals`/`Equals`). Commonly override `ToString` (diagnostics), and `Equals` + `GetHashCode` together for value-like types; `GetType` is non-virtual.

**Explanation**
`MemberwiseClone` makes a shallow copy (reference fields are shared). `Finalize` is the destructor `~T()`; avoid unless holding unmanaged resources (prefer `IDisposable`/`SafeHandle`). `GetType()` returns the runtime type; `typeof(T)` is compile-time.

**Code**
```csharp
Animal a = new Dog();
Console.WriteLine(a.GetType().Name);          // Output: Dog
Console.WriteLine(typeof(Animal).Name);       // Output: Animal
Console.WriteLine(a is Animal);               // Output: True
Console.WriteLine(a.GetType() == typeof(Animal)); // Output: False (exact type check)
Console.WriteLine(new object().ToString());   // Output: System.Object

class Animal { }
class Dog : Animal { }

class Shallow
{
    public List<int> Items = new();
    public Shallow Clone() => (Shallow)MemberwiseClone();   // shares Items
}
```

### 11.3 [Medium] How do you correctly implement value equality for a class (Equals, GetHashCode, IEquatable<T>, operators)?

**Answer**
Implement `IEquatable<T>.Equals(T)` with the real logic, override `Equals(object)` to delegate to it, override `GetHashCode` using the same fields (`HashCode.Combine`), and optionally add `==`/`!=` operators that call `Equals`. Make the type `sealed` or handle inheritance deliberately.

**Explanation**
Rules: reflexive, symmetric, transitive, consistent, `x.Equals(null)` is false. Equal objects must have equal hashes. Using only immutable fields in the hash avoids the lost-key bug (next question). Or just use a `record`.

**Code**
```csharp
public sealed class Point : IEquatable<Point>
{
    public int X { get; }
    public int Y { get; }
    public Point(int x, int y) => (X, Y) = (x, y);

    public bool Equals(Point? other) => other is not null && X == other.X && Y == other.Y;
    public override bool Equals(object? obj) => Equals(obj as Point);
    public override int GetHashCode() => HashCode.Combine(X, Y);

    public static bool operator ==(Point? l, Point? r) => l is null ? r is null : l.Equals(r);
    public static bool operator !=(Point? l, Point? r) => !(l == r);
}

class Demo
{
    static void Main()
    {
        var set = new HashSet<Point> { new(1, 2) };
        Console.WriteLine(set.Contains(new Point(1, 2)));          // Output: True
        Console.WriteLine(new Point(1, 2) == new Point(1, 2));      // Output: True
        Console.WriteLine(new Point(1, 2).Equals(null));            // Output: False
    }
}
```

### 11.4 [Hard] Why can an object "disappear" from a `HashSet`/`Dictionary` after you mutate it?

**Answer**
Hash-based collections store the item in a bucket chosen by its hash code at insertion time. If a field used in `GetHashCode` changes afterward, the object's hash changes, so lookups go to a different bucket (or the stored hash mismatches) and the item is no longer found — though it's still in the collection.

**Explanation**
Fix: use immutable keys, or don't include mutable fields in the hash. Same issue with mutable `record` members used as keys. Also: overriding `Equals` without `GetHashCode` makes dictionaries misbehave (compiler warning CS0659).

**Code**
```csharp
class Key
{
    public int Id;
    public override bool Equals(object? o) => o is Key k && k.Id == Id;
    public override int GetHashCode() => Id;
}

class Demo
{
    static void Main()
    {
        var set = new HashSet<Key>();
        var k = new Key { Id = 1 };
        set.Add(k);
        k.Id = 2;                                      // mutated while inside the set

        Console.WriteLine(set.Count);                  // Output: 1
        Console.WriteLine(set.Contains(k));            // Output: False  (looks in the wrong bucket)
        Console.WriteLine(set.Contains(new Key { Id = 1 })); // Output: False (hash matches, Equals fails)
    }
}
```

**Gotchas / follow-ups**
- Same hazard applies to `Dictionary` keys and `SortedSet` ordering keys.
- `string.GetHashCode()` is randomized per process in .NET Core — never persist hash codes.

### 11.5 [Tricky] Predict the output: static-type `==`, boxed numeric `Equals`, and `NaN`.

**Answer**
`object == string` uses reference equality (compile-time static types are `object`), so it's `false` even with equal content. `((object)1).Equals(1L)` is `false` because `Int32.Equals(object)` requires the argument to be an `int`. `NaN == NaN` is `false` but `NaN.Equals(NaN)` is `true`.

**Explanation**
Operator overloads aren't virtual: `a == b` picks `operator ==` from the static types. Numeric `Equals(object)` checks exact type, whereas `1 == 1L` converts. `double.Equals` treats NaN as equal to itself so sorting/hashing is consistent.

**Code**
```csharp
object o = "abc";
string s = new string(new[] { 'a', 'b', 'c' });
Console.WriteLine(o == s);                    // Output: False  (reference ==; compiler warns CS0252/CS0253)
Console.WriteLine(o.Equals(s));               // Output: True   (virtual -> string.Equals)
Console.WriteLine((string)o == s);            // Output: True

Console.WriteLine(1 == 1L);                   // Output: True   (1 promoted to long)
Console.WriteLine(1.Equals(1L));              // Output: False  (argument boxed long != int)
Console.WriteLine(((object)1).Equals(1));     // Output: True

Console.WriteLine(double.NaN == double.NaN);       // Output: False
Console.WriteLine(double.NaN.Equals(double.NaN));  // Output: True
Console.WriteLine(0.0 == -0.0);                    // Output: True
```

**Gotchas / follow-ups**
- With unconstrained generics, `a == b` on `T` is reference equality for classes; use `EqualityComparer<T>.Default.Equals(a, b)`.
- `Tuple`/`ValueTuple`, `Nullable<T>`, and records have value-based equality built in.

**Interview traps**
- Overriding `Equals` without `GetHashCode` (or vice versa).
- Mutable fields in `GetHashCode` of dictionary keys.
- `==` is not polymorphic; `Equals` is.
- `struct` without `IEquatable<T>` boxes and uses slow reflection-based `Equals` (unless fields are simple; still slow).
- `NaN` equality differs between `==` and `Equals`.

**Cheat sheet**
- `==` = compile-time operator; `Equals` = virtual; `ReferenceEquals` = identity.
- `Equals` + `GetHashCode` + `IEquatable<T>` together.
- Equal ⇒ same hash (not vice versa); keep hash stable.
- `HashCode.Combine(...)`.
- Object members: `ToString/Equals/GetHashCode/GetType/MemberwiseClone/Finalize`.

---

## Topic 12: Parameters: optional, named, params

**Study notes**

- **Optional parameters**: have a default value (`int x = 5`); must follow required ones; default must be a compile-time constant, `default`, or `new()` of a value type. The default is **copied into the call site at compile time** (like `const`).
- **Named arguments**: `Foo(y: 2, x: 1)`; improve readability, allow skipping optionals; arguments are *evaluated in the order written at the call site*, not in parameter order. Positional arguments can follow named ones only when the named ones are in their correct position (C# 7.2).
- **`params`**: last parameter, one per method, a single-dimensional array (C# 13/.NET 9 adds `Span<T>`, `IEnumerable<T>` etc.). Callable with zero or more args, or with an actual array (normal form).
- **Overload resolution preferences**: candidates not requiring default-value substitution, and normal form over expanded `params` form, win ties.
- `[CallerMemberName]`, `[CallerFilePath]`, `[CallerLineNumber]`, `[CallerArgumentExpression]` fill optional parameters at compile time.

### 12.1 [Easy] How do optional and named parameters work?

**Answer**
Optional parameters have defaults so callers can omit them; named arguments let callers specify arguments by parameter name in any order, which also lets them skip optional parameters in the middle.

**Explanation**
Optional parameters must come after required ones. They reduce the need for many overloads, but have a binary-compat caveat (see 12.4).

**Code**
```csharp
static string Connect(string host, int port = 1433, bool encrypt = true, int timeout = 30)
    => $"{host}:{port} enc={encrypt} t={timeout}";

Console.WriteLine(Connect("db"));                        // Output: db:1433 enc=True t=30
Console.WriteLine(Connect("db", timeout: 5));            // Output: db:1433 enc=True t=5
Console.WriteLine(Connect(port: 5432, host: "pg"));      // Output: pg:5432 enc=True t=30
```

### 12.2 [Follow-up] How does `params` work, and what are its rules?

**Answer**
`params` lets callers pass a variable number of arguments, which the compiler packs into an array. It must be the last parameter, only one per method, and it must be a (single-dimensional) array in C# 12. You can also pass an existing array directly.

**Explanation**
Each expanded call allocates a new array (unless the compiler can optimize; C# 13 collection `params` such as `ReadOnlySpan<T>` avoid this). Provide fixed-arity overloads for hot paths (that's why `string.Format` and `Path.Combine` have several).

**Code**
```csharp
static int Sum(params int[] nums) => nums.Sum();

Console.WriteLine(Sum());                    // Output: 0
Console.WriteLine(Sum(1, 2, 3));             // Output: 6
Console.WriteLine(Sum(new[] { 4, 5 }));      // Output: 9   (array passed directly)
```

### 12.3 [Medium] In what order are named arguments evaluated?

**Answer**
In the order they're written at the call site (left to right), not the parameter declaration order. The values are then bound to parameters by name.

**Explanation**
This matters if the argument expressions have side effects. The compiler uses temporaries to preserve the written order when it differs from the parameter order.

**Code**
```csharp
class Demo
{
    static int A() { Console.Write("A "); return 1; }
    static int B() { Console.Write("B "); return 2; }
    static void F(int x, int y) => Console.WriteLine($"x={x} y={y}");

    static void Main()
    {
        F(y: A(), x: B());
        // Output: A B x=2 y=1
    }
}
```

### 12.4 [Hard] What are the versioning and overload-resolution pitfalls of optional parameters?

**Answer**
Defaults are baked into the call site at compile time, so changing a default in a library has no effect on already-compiled callers until they're recompiled; adding an optional parameter to a public method is a binary-breaking change (signature changed) even though it's source-compatible. In overload resolution, a candidate that needs no default substitution beats one that does, and normal-form beats `params` expansion.

**Explanation**
For public APIs, prefer explicit overloads (`M(x)` and `M(x, y)`) over optional parameters, to preserve binary compatibility. Don't mix optional parameters and overloads that can create ambiguity. Also, interface implementations can declare different defaults than the interface: which default applies depends on the *static type* used at the call.

**Code**
```csharp
class Demo
{
    static void M(int x)             => Console.WriteLine("M(int)");
    static void M(int x, int y = 0)  => Console.WriteLine("M(int, int=0)");

    static void N(int x, int y)      => Console.WriteLine("N(int,int)");
    static void N(int x, params int[] rest) => Console.WriteLine("N(int, params)");

    interface IGreeter { string Greet(string n = "interface"); }
    class G : IGreeter { public string Greet(string n = "class") => n; }

    static void Main()
    {
        M(1);                                   // Output: M(int)
        N(1, 2);                                // Output: N(int,int)   (normal form beats params expansion)
        N(1, 2, 3);                             // Output: N(int, params)

        var g = new G();
        Console.WriteLine(g.Greet());           // Output: class
        Console.WriteLine(((IGreeter)g).Greet()); // Output: interface   (default chosen by static type)
    }
}
```

**Gotchas / follow-ups**
- Optional params can't be used with `ref`/`out`; attribute constructor arguments use them as constants.
- Reflection/`Expression` trees don't allow omitting optional arguments (expression trees can't contain calls with omitted optionals).

### 12.5 [Tricky] What do these `params object[]` calls print?

**Answer**
`Show()` → 0 items; `Show(null)` → null array (the `null` literal converts to `object[]` in normal form); `Show(1, 2)` → 2; `Show(new object[]{1,2,3})` → 3 (passed as the array); `Show((object)new object[]{1,2,3})` → 1 (single element); `Show(new int[]{1,2})` → 1, because `int[]` isn't an `object[]`.

**Explanation**
`params object[]` creates ambiguity between "the array itself" and "a single element". Normal form applies when the argument is implicitly convertible to `object[]` (including `null` and covariant `string[]`). Casting to `object` forces the expanded form.

**Code**
```csharp
class Demo
{
    static string Show(params object[]? a) => a is null ? "null array" : $"{a.Length} items";

    static void Main()
    {
        Console.WriteLine(Show());                                  // Output: 0 items
        Console.WriteLine(Show(null));                              // Output: null array
        Console.WriteLine(Show(1, 2));                              // Output: 2 items
        Console.WriteLine(Show(new object[] { 1, 2, 3 }));          // Output: 3 items
        Console.WriteLine(Show((object)new object[] { 1, 2, 3 })); // Output: 1 items
        Console.WriteLine(Show(new int[] { 1, 2 }));                // Output: 1 items
        Console.WriteLine(Show(new string[] { "a", "b" }));         // Output: 2 items (array covariance)
    }
}
```

**Interview traps**
- Optional parameter defaults are compile-time constants baked into callers.
- Named args are evaluated in written order.
- `params` array allocated on each expanded call; only one `params`, must be last.
- Passing `null` to `params object[]` gives a null array, not `[null]`.
- Overriding methods/interface impls can have different defaults; the static type decides.

**Cheat sheet**
- Optional: constant default, after required, baked into call site.
- Named: any order, evaluated left-to-right as written.
- `params`: last, one, array (C# 13: also span/IEnumerable).
- Prefer overloads for public/binary-stable APIs.
- Normal form beats expanded `params`; no-default-substitution beats defaults.

---

## Topic 13: Properties vs fields, auto-props, init/required

**Study notes**

- **Field** = raw storage. **Property** = one or two accessor *methods* (`get_X`/`set_X`) with field-like syntax: can validate, compute, be `virtual`/`abstract`, appear in interfaces, have different accessor accessibility (`public get; private set;`), raise change notifications.
- **Auto-property** `public int Id { get; set; }`: compiler generates a hidden backing field (`<Id>k__BackingField`). Get-only auto-properties (`{ get; }`) can be set in the declaration initializer or constructor only.
- **`init`** accessor (C# 9): settable during object initialization (initializer, constructor, `with`), read-only afterwards. **`required`** (C# 11): callers *must* set the member in an object initializer (compile-time check; bypass with `[SetsRequiredMembers]` on a constructor).
- Expression-bodied: `public string Full => $"{First} {Last}";` (computed on **every** access).
- Properties can't be passed by `ref`/`out` and aren't variables; reading a struct-typed property returns a **copy**.
- Serializers, data-binding, and most frameworks work with properties (public ones), not fields.

### 13.1 [Easy] What's the difference between a field and a property? Why prefer properties for public members?

**Answer**
A field is plain storage; a property is a pair of accessor methods with field-like syntax. Public data should be exposed via properties because they allow validation, lazy computation, change notification, different get/set accessibility, and can be changed later without breaking binary compatibility; frameworks like serializers and data binding also expect properties.

**Explanation**
Changing a public field to a property is a *breaking* change (binary and for `ref` usage), so start with a property.

**Code**
```csharp
class Account
{
    private decimal _balance;                  // field
    public decimal Balance                     // property with validation
    {
        get => _balance;
        private set
        {
            if (value < 0) throw new ArgumentOutOfRangeException(nameof(value));
            _balance = value;
        }
    }
    public void Deposit(decimal amount) => Balance += amount;
}
```

### 13.2 [Follow-up] What is an auto-property, and what are the forms (get-only, private set, init)?

**Answer**
An auto-property lets the compiler generate the backing field. `{ get; set; }` is mutable, `{ get; }` is get-only (assignable in declaration/constructor), `{ get; private set; }` is mutable only inside the class, and `{ get; init; }` is settable only during initialization.

**Explanation**
Get-only auto-properties give you true immutability of the field (the backing field is `readonly`). Initializers like `= new()` run before the constructor body.

**Code**
```csharp
class Order
{
    public int Id { get; }                         // get-only
    public string Status { get; private set; } = "New";
    public DateTime Created { get; init; } = DateTime.UtcNow;
    public List<string> Lines { get; } = new();    // get-only reference; list is still mutable

    public Order(int id) => Id = id;
    public void Ship() => Status = "Shipped";
}
```

### 13.3 [Medium] How do `init` and `required` work together?

**Answer**
`init` makes a property assignable only in an object initializer, constructor, or `with` expression. `required` forces callers to assign it in the initializer (or via a constructor marked `[SetsRequiredMembers]`), giving immutable-after-creation objects with a compile-time guarantee that mandatory values are set.

**Explanation**
Before `required`, `init` alone couldn't force assignment. Both are compile-time features (`required` is enforced by the compiler through attributes; the runtime doesn't check, so reflection/deserializers may bypass — System.Text.Json (.NET 7+) does honor `required`).

**Code**
```csharp
class Person
{
    public required string Name { get; init; }
    public int Age { get; init; }
}

class Demo
{
    static void Main()
    {
        var p = new Person { Name = "Ann", Age = 30 };
        // p.Name = "Bob";                    // error CS8852: init-only property
        // var q = new Person { Age = 5 };    // error CS9035: required member 'Name' must be set
        Console.WriteLine($"{p.Name} {p.Age}"); // Output: Ann 30
    }
}
```

### 13.4 [Hard] Why can't you write `obj.Location.X = 5` when `Location` is a struct property, and why can't you pass a property by `ref`?

**Answer**
A property getter returns a *copy* of a struct value, so assigning to a field of that temporary would be lost; the compiler rejects it (CS1612). Properties aren't variables (storage locations) but method pairs, so they can't be passed by `ref`/`out`. Read the struct, change it, and assign it back — or expose a `ref` return or a field.

**Explanation**
Contrast with arrays and fields of struct type, which are variables (`arr[0].X = 1` and `obj.field.X = 1` work). A `ref` property (ref-returning get-only property) returns an alias and does allow in-place modification.

**Code**
```csharp
struct Pt { public int X, Y; }

class Shape
{
    public Pt Location { get; set; }               // property returns a copy
    private Pt _loc;
    public ref Pt LocationRef => ref _loc;         // ref-returning property
    public int Read() => _loc.X;
}

class Demo
{
    static void Main()
    {
        var s = new Shape();
        // s.Location.X = 5;                       // error CS1612
        var tmp = s.Location; tmp.X = 5; s.Location = tmp;   // read-modify-write
        Console.WriteLine(s.Location.X);           // Output: 5

        s.LocationRef.X = 7;                       // OK: alias to the field
        Console.WriteLine(s.Read());               // Output: 7

        // int.TryParse("1", out s.Location.X);    // error: property is not a variable
    }
}
```

**Gotchas / follow-ups**
- Hence mutable structs + properties = bugs. Prefer immutable structs.
- Follow-up: "Is there a performance cost for properties?" Trivial getters/setters are inlined by the JIT.

### 13.5 [Tricky] What's the difference between `Items { get; } = new();` and `Items => new();`?

**Answer**
`{ get; } = new()` creates one list when the object is constructed and returns the same instance every time. The expression-bodied `=> new()` is a getter that creates a *new* list on every access, so adds are lost. This is a classic silent bug.

**Explanation**
Expression-bodied properties are computed on each access; use them for derived values, not for storage. Lazy initialization is a third form: `_items ??= new();` in the getter (not thread-safe without synchronization; use `Lazy<T>`).

**Code**
```csharp
class Bag
{
    public List<int> Stored { get; } = new();      // one instance
    public List<int> Fresh => new();                // new instance per call (bug!)

    private List<int>? _lazy;
    public List<int> Lazy => _lazy ??= new();       // created on first use
}

class Demo
{
    static void Main()
    {
        var b = new Bag();
        b.Stored.Add(1);  b.Fresh.Add(1);  b.Lazy.Add(1);
        Console.WriteLine(b.Stored.Count);  // Output: 1
        Console.WriteLine(b.Fresh.Count);   // Output: 0
        Console.WriteLine(b.Lazy.Count);    // Output: 1
    }
}
```

**Interview traps**
- `=>` vs `{ get; } =` initializer difference.
- Struct-typed property: modifying returned copy (CS1612).
- `init` is not "readonly" for reflection; `required` is compile-time.
- Public field → property later is a breaking change.
- Expensive or side-effecting getters surprise callers (debugger evaluates them).

**Cheat sheet**
- Property = accessor methods; auto-property = generated backing field.
- `{ get; }` set in ctor/initializer only; `init` in initializers/ctor/`with`.
- `required` = must be set at construction (C# 11).
- `=>` re-evaluates on every access.
- Property ≠ variable (no `ref`, struct copy).

---

## Topic 14: Operators & implicit/explicit conversions

**Study notes**

- **User-defined operators** are `public static` methods: `public static T operator +(T a, T b)`. Overloadable: unary `+ - ! ~ ++ -- true false`; binary `+ - * / % & | ^ << >> == != < > <= >=` (relational/equality must be overloaded in pairs). Not overloadable: `=`, `&&`/`||` (derived from `&`/`|` + `true`/`false`), `?:`, `.`, `new`, `is`, `as`, `=>`. Compound `+=` comes from `+`.
- **Implicit conversion**: must never throw or lose information. **Explicit conversion** (cast): may throw or lose data. Declared with `implicit operator`/`explicit operator` inside either the source or target type. Can't define conversions to/from interfaces or base/derived types.
- **Built-in numeric conversions**: widening is implicit (`int→long→float/double/decimal`), narrowing explicit. `(int)3.9` truncates toward zero → 3.
- **`as`** returns null on failure (reference/nullable types only), **cast** throws `InvalidCastException`, **`is`** tests (and with patterns, binds). User-defined conversions are **not** considered by `is`/`as`.
- **`Convert.ToInt32(double)`** rounds (banker's: midpoint to even); `Math.Round` also defaults to `ToEven`.
- C# 11: `static abstract` interface members enable generic math (`INumber<T>`), and `checked` user-defined operators.

### 14.1 [Easy] What's the difference between an implicit and an explicit conversion?

**Answer**
An implicit conversion happens automatically and must be safe (no data loss, no exceptions), like `int` → `long`. An explicit conversion requires a cast because it may lose data or fail, like `double` → `int` or `object` → `string`.

**Explanation**
Narrowing is explicit because the compiler can't guarantee the value fits. User-defined conversions follow the same convention: use `implicit` only when truly lossless.

**Code**
```csharp
int i = 42;
long l = i;               // implicit widening
double d = 3.99;
int t = (int)d;           // explicit narrowing: truncation
Console.WriteLine(t);     // Output: 3
Console.WriteLine((int)-3.99); // Output: -3  (toward zero)

object o = "text";
string s = (string)o;     // explicit reference conversion
```

### 14.2 [Follow-up] How do you overload operators? What are the rules?

**Answer**
Declare `public static` methods with the `operator` keyword in one of the operand types. Comparison operators must be overloaded in pairs (`==`/`!=`, `<`/`>`, `<=`/`>=`), and if you overload `==` you should override `Equals`/`GetHashCode`. `+=` is automatically derived from `+`.

**Explanation**
Overloading should be intuitive (math-like types: vectors, money, time spans). Compiler warns if you overload `==` without `Equals`/`GetHashCode`.

**Code**
```csharp
public readonly struct Vec2 : IEquatable<Vec2>
{
    public double X { get; }
    public double Y { get; }
    public Vec2(double x, double y) => (X, Y) = (x, y);

    public static Vec2 operator +(Vec2 a, Vec2 b) => new(a.X + b.X, a.Y + b.Y);
    public static Vec2 operator -(Vec2 v) => new(-v.X, -v.Y);
    public static bool operator ==(Vec2 a, Vec2 b) => a.Equals(b);
    public static bool operator !=(Vec2 a, Vec2 b) => !a.Equals(b);

    public bool Equals(Vec2 o) => X == o.X && Y == o.Y;
    public override bool Equals(object? obj) => obj is Vec2 v && Equals(v);
    public override int GetHashCode() => HashCode.Combine(X, Y);
    public override string ToString() => $"({X}, {Y})";
}

class Demo
{
    static void Main()
    {
        var v = new Vec2(1, 2);
        v += new Vec2(3, 4);                 // uses operator +
        Console.WriteLine(v);                // Output: (4, 6)
        Console.WriteLine(-v);               // Output: (-4, -6)
        Console.WriteLine(v == new Vec2(4, 6)); // Output: True
    }
}
```

### 14.3 [Medium] Compare cast, `as`, `is`, `Convert`, and `Parse`; what does rounding look like?

**Answer**
A cast throws on failure and truncates numerics; `as` returns null on failure (only for reference/nullable types); `is` tests the type (and can bind with patterns); `Convert.ToXxx` converts between many types and *rounds* doubles to nearest (midpoint to even); `Parse`/`TryParse` convert from strings.

**Explanation**
Use `TryParse` for untrusted strings and `is`/`as` for type checks that can legitimately fail. `Convert.ToInt32(null string)` returns 0 while `int.Parse(null)` throws.

**Code**
```csharp
object o = 42;
Console.WriteLine(o as string is null);       // Output: True  (no exception)
if (o is int n && n > 40) Console.WriteLine("big int"); // Output: big int
try { _ = (string)o; } catch (InvalidCastException) { Console.WriteLine("bad cast"); } // Output: bad cast

Console.WriteLine((int)2.9);                  // Output: 2
Console.WriteLine(Convert.ToInt32(2.5));      // Output: 2   (banker's rounding)
Console.WriteLine(Convert.ToInt32(3.5));      // Output: 4
Console.WriteLine(Math.Round(2.5));           // Output: 2
Console.WriteLine(Math.Round(2.5, MidpointRounding.AwayFromZero)); // Output: 3
```

### 14.4 [Hard] Are user-defined conversions applied at runtime by `is`/`as`/`object`? Show how static types decide.

**Answer**
No. User-defined conversions are resolved at compile time from static types and are only applied by casts/assignments; `is`, `as`, and unboxing consider only built-in reference/boxing/identity conversions. So an `object` holding a `Celsius` is never "a `double`", even if an implicit `Celsius → double` conversion exists.

**Explanation**
This also means a conversion operator isn't polymorphic: `Base b = new Derived()` uses `Base`'s operators. You also can't declare a user-defined conversion from/to an interface or between a class and its base. Don't use conversions to change identity, only representation.

**Code**
```csharp
readonly struct Celsius
{
    public double Degrees { get; }
    public Celsius(double d) => Degrees = d;
    public static implicit operator double(Celsius c) => c.Degrees;       // lossless
    public static explicit operator Celsius(double d) => new(d);          // validation could go here
}

class Demo
{
    static void Main()
    {
        var c = new Celsius(21.5);
        double d = c;                          // implicit user-defined conversion
        Celsius back = (Celsius)d;             // explicit
        Console.WriteLine(d);                  // Output: 21.5

        object o = c;
        Console.WriteLine(o is double);        // Output: False  (user-defined conversion not considered)
        Console.WriteLine(o is Celsius);       // Output: True
        // double bad = (double)o;             // InvalidCastException at runtime (unbox to Celsius required)
    }
}
```

**Gotchas / follow-ups**
- Implicit conversions can make overload resolution surprising; keep them rare.
- `string` ↔ custom type implicit conversions hide allocations/failures: prefer explicit factory methods.

### 14.5 [Tricky] Predict: `+` with strings, `char`, compound assignment, and implicit numeric narrowing.

**Answer**
`+` is evaluated left to right: `1 + 2 + "a"` is `"3a"`, but `"a" + 1 + 2` is `"a12"`. `short s = 1; s = s + 1;` doesn't compile because `s + 1` is `int`, but `s += 1` compiles (compound assignment includes an implicit cast). Constant expressions that fit the target type convert implicitly (`byte b = 10;`).

**Explanation**
Arithmetic on types smaller than `int` promotes to `int`. Compound assignment `x op= y` is `x = (T)(x op y)` when the operator result is explicitly convertible to `T`, and may silently truncate.

**Code**
```csharp
Console.WriteLine(1 + 2 + "a");        // Output: 3a
Console.WriteLine("a" + 1 + 2);        // Output: a12
Console.WriteLine('a' + 'b');          // Output: 195  (char promoted to int)
Console.WriteLine((char)('a' + 1));    // Output: b

short s = 1;
// s = s + 1;                          // error CS0266: int -> short needs cast
s += 1;                                // OK
Console.WriteLine(s);                  // Output: 2

byte b = 250;
b += 10;                               // compiles; unchecked wrap
Console.WriteLine(b);                  // Output: 4
```

**Interview traps**
- `implicit` conversions that can throw or lose data (violates convention).
- `==` overloaded without `Equals`/`GetHashCode`.
- `is`/`as` ignore user-defined conversions.
- `Convert.ToInt32(2.5)` is 2 (banker's), `(int)2.5` also 2, but `Convert.ToInt32(3.5)` is 4 while `(int)3.5` is 3.
- `&&`/`||` can't be overloaded directly.

**Cheat sheet**
- Operators = `public static`; pairs for comparisons.
- `implicit` = safe; `explicit` = may fail/lose.
- Cast throws; `as` → null; `is` tests/binds.
- Cast truncates; `Convert` rounds (to even); `Math.Round` default to even.
- Small integer arithmetic promotes to `int`.

---

## Topic 15: checked/unchecked, integer/decimal/double precision

**Study notes**

- **Integer overflow** is silent wraparound by default (`unchecked`) for non-constant expressions; **constant** expressions that overflow are compile errors (CS0220). Use `checked(expr)`, `checked { }`, or project setting `<CheckForOverflowUnderflow>true</CheckForOverflowUnderflow>` to throw `OverflowException`. `unchecked` locally opts out.
- Integer division truncates toward zero; `x / 0` throws `DivideByZeroException` for integers; for floating point it yields `Infinity`/`NaN`. `%` keeps the dividend's sign.
- `float` (32-bit, ~6–9 digits), `double` (64-bit IEEE-754 binary, ~15–17 digits), `decimal` (128-bit **base-10** floating point, 28–29 significant digits). Binary floating point can't represent 0.1 exactly → use `decimal` for money.
- `decimal` is slower (software), has a larger range of precision but smaller *range* than double, keeps trailing zeros (`1.0m` vs `1.00m` — equal, print differently), and throws `OverflowException` on overflow.
- Mixing `double` and `decimal` requires an explicit cast.
- `NaN` is unequal to everything (including itself); compare doubles with a tolerance (`Math.Abs(a - b) < eps`).

### 15.1 [Easy] What happens when an `int` overflows in C#?

**Answer**
By default it silently wraps around (`int.MaxValue + 1` becomes `int.MinValue`) in unchecked context; no exception is thrown. If the overflow is detectable at compile time with constants, it's a compile error. In a `checked` context it throws `OverflowException`.

**Explanation**
Unchecked is the default for performance. Wrapping bugs are silent and nasty (e.g., size calculations, IDs).

**Code**
```csharp
int max = int.MaxValue;
int wrapped = max + 1;                 // non-constant: wraps silently
Console.WriteLine(wrapped);            // Output: -2147483648
// int bad = int.MaxValue + 1;         // compile error CS0220 (constant overflow)
```

### 15.2 [Follow-up] How do you enable overflow checking?

**Answer**
Wrap specific expressions or blocks in `checked(...)`/`checked { }`, or enable `<CheckForOverflowUnderflow>` project-wide; use `unchecked` to opt specific hot code out (e.g., hash computations).

**Explanation**
Checked arithmetic has a small cost. For security-sensitive arithmetic (sizes, money in integers), prefer checked. Hashing code intentionally relies on wrapping and should be marked `unchecked`.

**Code**
```csharp
int max = int.MaxValue;

try
{
    int r = checked(max + 1);
    Console.WriteLine(r);
}
catch (OverflowException) { Console.WriteLine("overflow"); }   // Output: overflow

checked
{
    try { int x = max; x++; }
    catch (OverflowException) { Console.WriteLine("block overflow"); } // Output: block overflow
}

int doubled = unchecked(max * 2);       // intentional wrap
Console.WriteLine(doubled);             // Output: -2
```

### 15.3 [Medium] Why does `0.1 + 0.2 == 0.3` fail with `double`, and when should you use `decimal`?

**Answer**
`double` is binary floating point; 0.1 and 0.2 have no exact binary representation, so the sum is `0.30000000000000004`. `decimal` is base-10, so these values are exact; use `decimal` for money and anything requiring exact decimal fractions, and `double` for scientific/graphics work where speed matters and tiny errors are acceptable.

**Explanation**
Compare doubles using a tolerance. `decimal` is not magic: `1m / 3m * 3m` is `0.9999999999999999999999999999`. Store money in SQL as `decimal(18,2)`/`money` (not `float`).

**Code**
```csharp
Console.WriteLine(0.1 + 0.2 == 0.3);        // Output: False
Console.WriteLine(0.1 + 0.2);               // Output: 0.30000000000000004
Console.WriteLine(0.1m + 0.2m == 0.3m);     // Output: True
Console.WriteLine(1m / 3m * 3m);            // Output: 0.9999999999999999999999999999

double a = 0.1 + 0.2, b = 0.3;
Console.WriteLine(Math.Abs(a - b) < 1e-9);  // Output: True
```

### 15.4 [Hard] Name integer edge cases: `int.MinValue` negation/`Math.Abs`/division, and overflow before widening.

**Answer**
`-int.MinValue` wraps to `int.MinValue` (unchecked); `Math.Abs(int.MinValue)` throws `OverflowException` (the result doesn't fit); `int.MinValue / -1` throws `OverflowException` on .NET. And `long r = a * b` with `int` operands multiplies in 32-bit first, overflowing *before* widening; cast an operand to `long` first.

**Explanation**
Arithmetic type is determined by operand types, not by the assignment target. This is a perennial production bug (e.g., computing byte sizes or milliseconds from ints).

**Code**
```csharp
int m = int.MinValue;
Console.WriteLine(-m);                      // Output: -2147483648
try { Math.Abs(m); }
catch (OverflowException) { Console.WriteLine("abs overflow"); }  // Output: abs overflow

int minus1 = -1;
try { Console.WriteLine(m / minus1); }
catch (OverflowException) { Console.WriteLine("div overflow"); }  // Output: div overflow

int a = 2_000_000_000, b = 2;
long wrong = a * b;                         // 32-bit multiply wraps first
long right = (long)a * b;
Console.WriteLine(wrong);                   // Output: -294967296
Console.WriteLine(right);                   // Output: 4000000000
```

**Gotchas / follow-ups**
- `-7 / 2` is `-3` and `-7 % 2` is `-1` (truncate toward zero; remainder has the dividend's sign).
- Casting an out-of-range `double` to `int` is unspecified/platform-dependent in unchecked code; validate range first.

### 15.5 [Tricky] Special floating-point values and `decimal` quirks

**Answer**
`1.0 / 0` is `Infinity` (no exception) while `1 / 0` on ints throws `DivideByZeroException`; `0.0 / 0` is `NaN` and `NaN != NaN`. `decimal` equality ignores scale (`1.0m == 1.00m`) but `ToString` shows the scale; and `decimal` division by zero throws.

**Explanation**
Use `double.IsNaN`, `double.IsInfinity`, `double.IsFinite`. `float` adds even more error: `0.1f + 0.2f` prints `0.3` (rounded display) but `(double)0.1f` is `0.10000000149011612`. `decimal.Round` defaults to banker's rounding too (`MidpointRounding.ToEven`).

**Code**
```csharp
Console.WriteLine(1.0 / 0);                 // Output: Infinity
Console.WriteLine(double.NaN == double.NaN);// Output: False
Console.WriteLine(double.IsNaN(0.0 / 0));   // Output: True

int zero = 0;
try { Console.WriteLine(1 / zero); }
catch (DivideByZeroException) { Console.WriteLine("div by zero"); }  // Output: div by zero

Console.WriteLine(1.0m == 1.00m);           // Output: True
Console.WriteLine(1.00m);                   // Output: 1.00
Console.WriteLine(decimal.Round(2.5m));     // Output: 2
Console.WriteLine(decimal.Round(2.5m, MidpointRounding.AwayFromZero)); // Output: 3
Console.WriteLine((double)0.1f);            // Output: 0.10000000149011612
```

**Interview traps**
- Overflow is silent by default; constants fail at compile time.
- `long x = intA * intB` overflows before widening.
- `double` for money; `decimal` for exact decimals.
- `Math.Abs(int.MinValue)` throws.
- `decimal` equality ignores trailing zeros; `NaN != NaN`.

**Cheat sheet**
- `checked(...)`/`unchecked(...)`; project flag `CheckForOverflowUnderflow`.
- Int `/0` throws; double `/0` = ∞/NaN.
- double = binary (~15–17 digits); decimal = base-10 (28–29 digits).
- Cast operands before multiplying to avoid overflow.
- Compare doubles with tolerance; `IsNaN` for NaN.

---

## Topic 16: DateTime vs DateTimeOffset

**Study notes**

- **`DateTime`**: a tick count (100 ns units since 0001-01-01) plus a `Kind` flag: `Utc`, `Local`, or `Unspecified`. It carries **no offset**, so an `Unspecified` value is ambiguous. `DateTime.Now` = local, `DateTime.UtcNow` = UTC.
- **`DateTimeOffset`**: a `DateTime` + the **UTC offset** (e.g., `+05:30`). It identifies a single unambiguous instant. It stores the *offset*, **not** the time zone (no DST rules, no zone name) — use `TimeZoneInfo` for zone logic.
- **Equality/comparison**: `DateTime` compares ticks and **ignores `Kind`** (12:00 Utc == 12:00 Local). `DateTimeOffset` compares the underlying UTC instant (`12:00Z == 17:30+05:30`). `EqualsExact` compares offset too.
- `ToUniversalTime()` on `Unspecified` **assumes Local**; `ToLocalTime()` on `Unspecified` also assumes UTC. Use `DateTime.SpecifyKind` to label (doesn't convert).
- Subtracting gives a `TimeSpan`. `DateOnly`/`TimeOnly` (.NET 6) for date-only/time-only values.
- **SQL Server mapping**: `datetime2` ↔ `DateTime`, `datetimeoffset` ↔ `DateTimeOffset`; legacy `datetime` has ~3.33 ms precision and a limited range (1753+). EF Core maps `DateTime` to `datetime2` by default (SQL Server) and doesn't preserve `Kind` on reads (returns `Unspecified`).
- **Best practice**: store/transfer instants in UTC (or `DateTimeOffset`); convert to local only for display; inject a clock (`TimeProvider`, .NET 8) for testing.

### 16.1 [Easy] What's the difference between `DateTime` and `DateTimeOffset`?

**Answer**
`DateTime` is a date and time with only a loose `Kind` (UTC/Local/Unspecified), while `DateTimeOffset` also stores the offset from UTC, so it represents an exact, unambiguous point in time. Prefer `DateTimeOffset` (or UTC `DateTime`) for timestamps crossing machine/time-zone boundaries.

**Explanation**
Neither stores a time-zone *name*; an offset is a snapshot (e.g., `-05:00`), not rules for DST.

**Code**
```csharp
DateTime dt = DateTime.UtcNow;
DateTimeOffset dto = DateTimeOffset.UtcNow;

Console.WriteLine(dt.Kind);          // Output: Utc
Console.WriteLine(dto.Offset);       // Output: 00:00:00

var ist = new DateTimeOffset(2024, 6, 1, 18, 30, 0, TimeSpan.FromHours(5.5));
Console.WriteLine(ist.UtcDateTime);  // Output: 6/1/2024 1:00:00 PM  (culture-dependent format)
```

### 16.2 [Follow-up] What is `DateTime.Kind` and why does it matter?

**Answer**
`Kind` labels the value as `Utc`, `Local`, or `Unspecified`. Conversion methods (`ToUniversalTime`, `ToLocalTime`) use it: `Utc` isn't converted again by `ToUniversalTime`, `Local` converts using the machine time zone, and `Unspecified` is *assumed* to be Local (for `ToUniversalTime`) or UTC (for `ToLocalTime`), which causes subtle bugs.

**Explanation**
Values parsed from strings without a zone, or read from SQL `datetime2` via EF/ADO.NET, are `Unspecified`. Call `DateTime.SpecifyKind(value, DateTimeKind.Utc)` when you know it's UTC. `Kind` isn't persisted by SQL Server.

**Code**
```csharp
var raw = new DateTime(2024, 1, 1, 12, 0, 0);               // Unspecified
Console.WriteLine(raw.Kind);                                 // Output: Unspecified

var asUtc = DateTime.SpecifyKind(raw, DateTimeKind.Utc);     // label only, no shift
Console.WriteLine(asUtc.Hour);                               // Output: 12
Console.WriteLine(asUtc.ToUniversalTime().Hour);             // Output: 12 (already UTC)

var parsed = DateTime.Parse("2024-01-01T12:00:00Z");
Console.WriteLine(parsed.Kind);                              // Output: Local  (Parse converts 'Z' to local time!)
var exact = DateTime.Parse("2024-01-01T12:00:00Z", null, System.Globalization.DateTimeStyles.AdjustToUniversal);
Console.WriteLine(exact.Kind);                               // Output: Utc
```

### 16.3 [Medium] When would you use each, and how do they map to SQL Server and JSON?

**Answer**
Use UTC `DateTime` or `DateTimeOffset` for event timestamps; `DateTimeOffset` when the original offset matters (e.g., user-local appointment time, audit) and `DateOnly`/`TimeOnly` for dates/times without the other part. SQL Server: `datetime2` ↔ `DateTime`, `datetimeoffset` ↔ `DateTimeOffset`, `date` ↔ `DateOnly`, `time` ↔ `TimeOnly`. `System.Text.Json` emits ISO 8601 and keeps the offset/`Z`.

**Explanation**
Don't use `DateTime.Now` for stored data; it depends on the server's zone. A future scheduled local event (e.g., "9:00 Berlin time next March") should store local time + zone ID (IANA/Windows), since the offset may change with DST rules before the event occurs.

**Code**
```csharp
using System.Text.Json;

var dto = new DateTimeOffset(2024, 6, 1, 18, 30, 0, TimeSpan.FromHours(5.5));
Console.WriteLine(JsonSerializer.Serialize(dto));                 // Output: "2024-06-01T18:30:00+05:30"

var utc = new DateTime(2024, 6, 1, 13, 0, 0, DateTimeKind.Utc);
Console.WriteLine(JsonSerializer.Serialize(utc));                 // Output: "2024-06-01T13:00:00Z"

DateOnly d = new(2024, 6, 1);
Console.WriteLine(d.ToString("O"));                               // Output: 2024-06-01
```

### 16.4 [Hard] Does `DateTime` equality consider `Kind`? How do `DateTimeOffset` comparisons work?

**Answer**
`DateTime ==`/`Equals`/`CompareTo` compare only the tick values and ignore `Kind`, so `12:00 Utc` equals `12:00 Local` even though they're different instants (unless the machine is in UTC). `DateTimeOffset` compares the UTC instants, so `12:00+00:00` equals `17:30+05:30`; use `EqualsExact` to also compare offsets.

**Explanation**
Always normalize to UTC before comparing `DateTime` values from different sources. `DateTimeOffset` makes the comparison safe by construction; its `Offset` is informational for display.

**Code**
```csharp
var utc   = new DateTime(2024, 1, 1, 12, 0, 0, DateTimeKind.Utc);
var local = new DateTime(2024, 1, 1, 12, 0, 0, DateTimeKind.Local);
Console.WriteLine(utc == local);                       // Output: True  (Kind ignored!)

var a = new DateTimeOffset(2024, 1, 1, 12, 0, 0, TimeSpan.Zero);
var b = new DateTimeOffset(2024, 1, 1, 17, 30, 0, TimeSpan.FromHours(5.5));
Console.WriteLine(a == b);                             // Output: True  (same instant)
Console.WriteLine(a.EqualsExact(b));                   // Output: False (different offset)
Console.WriteLine(a.Offset == b.Offset);               // Output: False
```

**Gotchas / follow-ups**
- Implicit conversion `DateTime` → `DateTimeOffset` uses the `Kind` (Local/Unspecified use the local offset): can silently pick the wrong offset.
- `DateTimeOffset` as a dictionary key: two equal instants with different offsets are equal and hash the same.

### 16.5 [Tricky] DST edge cases: invalid and ambiguous local times, and testing with a clock abstraction

**Answer**
A local time can be *invalid* (skipped in spring-forward) or *ambiguous* (repeated in fall-back), which `DateTime` can't represent unambiguously and `TimeZoneInfo.IsInvalidTime`/`IsAmbiguousTime` detect. `DateTimeOffset` resolves it by carrying the offset. For testable code, depend on `TimeProvider` (.NET 8) instead of calling `DateTime.UtcNow` directly.

**Explanation**
Adding `TimeSpan.FromHours(24)` isn't "same wall-clock time tomorrow" across DST. Convert with `TimeZoneInfo.ConvertTimeFromUtc`/`ConvertTime` instead. Time zone IDs differ by OS (Windows IDs vs IANA); .NET 6+ can map between them with ICU, but be explicit in cross-platform code.

**Code**
```csharp
// Windows time zone ID (IANA "America/New_York" also works on current .NET with ICU).
var tz = TimeZoneInfo.FindSystemTimeZoneById("Eastern Standard Time");

var skipped   = new DateTime(2024, 3, 10, 2, 30, 0);     // spring forward in US 2024
var repeated  = new DateTime(2024, 11, 3, 1, 30, 0);     // fall back in US 2024
Console.WriteLine(tz.IsInvalidTime(skipped));            // Output: True
Console.WriteLine(tz.IsAmbiguousTime(repeated));         // Output: True

var instant = new DateTimeOffset(2024, 3, 10, 12, 0, 0, TimeSpan.Zero);
Console.WriteLine(TimeZoneInfo.ConvertTime(instant, tz).Offset); // Output: -04:00 (DST in effect)

class FixedTime(DateTimeOffset now) : TimeProvider
{
    public override DateTimeOffset GetUtcNow() => now;
}

// Usage: code takes TimeProvider and calls provider.GetUtcNow(); tests pass FixedTime.
```

**Interview traps**
- `DateTime ==` ignores `Kind`.
- `Unspecified` is guessed as Local/UTC depending on the method.
- `DateTimeOffset` stores an offset, not a time zone.
- `DateTime.Now` in server code/storage → bugs across servers/DST.
- EF Core/SQL `datetime2` doesn't preserve `Kind`.

**Cheat sheet**
- `DateTime` = ticks + Kind; `DateTimeOffset` = instant + offset.
- Store UTC / `DateTimeOffset`; display local.
- `SpecifyKind` labels; `ToUniversalTime` converts.
- `datetime2` ↔ `DateTime`, `datetimeoffset` ↔ `DateTimeOffset`.
- Inject `TimeProvider` for testable time.

---

## Topic 17: default / default literal, tuples & deconstruction

**Study notes**

- **`default(T)`** = the all-zero-bits value: `null` for references and `Nullable<T>`, `0`/`0.0`/`false`/`'\0'` for primitives, enums = 0, structs = every field `default`. **Constructors are not run.** `default` literal (C# 7.1) is target-typed: `int x = default;`, `return default;`, parameter `= default`.
- `default!` suppresses NRT warnings for reference types (still null at runtime). `new T()` for structs calls the (possibly custom) parameterless ctor; `default(T)` never does.
- **Tuples** (C# 7): `ValueTuple<T1,…>` — mutable value types with fields `Item1…`. Element names (`(int Id, string Name)`) are **compile-time only** (stored as `[TupleElementNames]` attributes on signatures; erased on locals). Tuple type identity ignores names. `System.Tuple<…>` is the older immutable reference type.
- **Tuple equality**: `==`/`!=` compare element-wise (C# 7.3); `Equals` is value-based; `ValueTuple` implements `IEquatable<>` so it's safe as a dictionary key (no boxing).
- **Deconstruction**: `var (a, b) = tuple;`, discards `_`, swap `(a, b) = (b, a);`. Any type with an accessible `Deconstruct(out …)` (instance or extension) can be deconstructed; records with positional parameters get one automatically.
- Tuples > 7 elements nest via `Rest`. Use records/classes when the data has meaning beyond a quick return value.

### 17.1 [Easy] What does `default` produce for different types?

**Answer**
`default(T)` yields the zero-initialized value: `null` for reference types and nullable value types, `0` for numerics, `false` for `bool`, and a struct with all fields defaulted. The `default` literal infers `T` from context, so `int x = default;` is `0`.

**Explanation**
Fields and array elements are automatically initialized to `default`. Enums default to `0`, even if no member is `0` (Topic 10).

**Code**
```csharp
Console.WriteLine(default(int));            // Output: 0
Console.WriteLine(default(bool));           // Output: False
Console.WriteLine(default(string) is null); // Output: True
Console.WriteLine(default(int?) is null);   // Output: True
Console.WriteLine(default(DateTime));       // Output: 1/1/0001 12:00:00 AM  (culture-dependent format)

string s = default;                         // default literal (null)
Pt p = default;
Console.WriteLine($"{p.X},{p.Y}");          // Output: 0,0

struct Pt { public int X, Y; }
```

### 17.2 [Follow-up] `default` vs `new T()` for a struct — what's the difference? Where is `default(T)` used in generics?

**Answer**
`default(T)` never runs constructors; `new T()` runs the struct's parameterless constructor if you declared one (C# 10+), otherwise it equals `default`. In generic code `default(T)` (or `default`) gives the type's zero value — `null` for reference types, zero for value types — e.g., when a lookup fails.

**Explanation**
This difference matters for structs with custom parameterless constructors: arrays and `default` still produce the zeroed form, so invariants set in the constructor aren't guaranteed. `default(T)` in a `T?` returning method for unconstrained `T` is `null` only for reference types.

**Code**
```csharp
struct Counter
{
    public int Value;
    public Counter() { Value = 10; }             // allowed since C# 10
}

class Demo
{
    static T? FirstOrNone<T>(IEnumerable<T> src)
    {
        foreach (var x in src) return x;
        return default;                          // null for ref types, 0 for int, etc.
    }

    static void Main()
    {
        Console.WriteLine(new Counter().Value);       // Output: 10
        Console.WriteLine(default(Counter).Value);    // Output: 0
        Console.WriteLine((new Counter[1])[0].Value);  // Output: 0  (arrays don't run the ctor)
        Console.WriteLine(FirstOrNone(new List<int>()));        // Output: 0
        Console.WriteLine(FirstOrNone(new List<string>()) is null); // Output: True
    }
}
```

### 17.3 [Medium] Show tuple usage: returning multiple values, naming, deconstruction, discards.

**Answer**
Return a tuple to hand back several values without a custom type: `(int Min, int Max) MinMax(...)`. Callers can use names (`r.Min`) or deconstruct (`var (min, max) = ...`), and discard unneeded parts with `_`.

**Explanation**
Tuples are value types (no allocation), good for private helpers and internal APIs. For public APIs with long-lived meaning, prefer a record so names are part of a real type. Inferred element names from variables (C# 7.1): `(x, y)` gets `x`, `y`.

**Code**
```csharp
static (int Min, int Max) MinMax(IEnumerable<int> src) => (src.Min(), src.Max());

var r = MinMax(new[] { 4, 2, 9 });
Console.WriteLine($"{r.Min}-{r.Max}");     // Output: 2-9

var (min, max) = r;                         // deconstruction
var (lo, _) = r;                            // discard
Console.WriteLine($"{min} {max} {lo}");     // Output: 2 9 2

int a = 1, b = 2;
(a, b) = (b, a);                            // swap without temp
Console.WriteLine($"{a} {b}");              // Output: 2 1

Console.WriteLine(r);                       // Output: (2, 9)
```

### 17.4 [Hard] Are tuple element names part of the type? What do tuple `==` and `GetType` show?

**Answer**
No — names are compile-time sugar. `(int a, int b)` and `(int x, int y)` are the same runtime type, `ValueTuple<int,int>`, and are implicitly convertible. At runtime the fields are `Item1`, `Item2`. `==`/`!=` compare element-wise, and `ValueTuple` implements value equality/hash, so tuples work as dictionary keys.

**Explanation**
Names survive in method signatures through `[TupleElementNames]`, so IDEs/compilers see them across assemblies, but reflection on locals or `object`-typed tuples doesn't. Since `ValueTuple` is a mutable struct, `readonly` fields and properties holding tuples give the same defensive-copy/copy-on-read problems as other mutable structs.

**Code**
```csharp
(int a, int b) t1 = (1, 2);
(int x, int y) t2 = t1;                       // names differ: fine
Console.WriteLine(t2.x);                      // Output: 1
Console.WriteLine(t1 == t2);                  // Output: True
Console.WriteLine(t1.GetType());              // Output: System.ValueTuple`2[System.Int32,System.Int32]

object o = (First: 1, Second: 2);
Console.WriteLine(o.GetType().GetField("Item1")!.GetValue(o)); // Output: 1
Console.WriteLine(o.GetType().GetField("First") is null);      // Output: True

var map = new Dictionary<(int, string), string> { [(1, "a")] = "one-a" };
Console.WriteLine(map[(1, "a")]);             // Output: one-a
```

**Gotchas / follow-ups**
- `System.Text.Json` serializes `ValueTuple` fields poorly by default (no `Item1` properties unless `IncludeFields`); use records/DTOs for serialization.
- Tuples have no `ReferenceEquals` semantics (they're values).

### 17.5 [Tricky] How do custom `Deconstruct` methods work, and what pitfalls exist with default structs and tuple mutation?

**Answer**
Any type can support `var (a, b) = obj;` by providing an accessible `Deconstruct(out T1, out T2)` instance or extension method (overloads by arity). Mutating a `ValueTuple` stored in a list or `readonly` field changes only a copy, and a `default` struct bypasses constructor logic, so `default((int, string))` has `Item2 == null`.

**Explanation**
Deconstruction is compile-time pattern-based, not an interface. Positional records generate `Deconstruct` for you. Tuple elements are public mutable fields, so `list[0].Item1 = 5` is a compile error (CS1612) for the same reason as for any struct.

**Code**
```csharp
class Person
{
    public string First { get; }
    public string Last  { get; }
    public int Age      { get; }
    public Person(string f, string l, int a) => (First, Last, Age) = (f, l, a);

    public void Deconstruct(out string first, out string last) => (first, last) = (First, Last);
    public void Deconstruct(out string first, out string last, out int age)
        => (first, last, age) = (First, Last, Age);
}

class Demo
{
    static void Main()
    {
        var p = new Person("Ada", "Lovelace", 36);
        var (f, l) = p;
        var (f3, l3, age) = p;
        Console.WriteLine($"{f} {l} {age}");          // Output: Ada Lovelace 36

        var list = new List<(int Id, string Name)> { (1, "a") };
        // list[0].Id = 5;                             // error CS1612
        var item = list[0]; item.Id = 5; list[0] = item;
        Console.WriteLine(list[0]);                    // Output: (5, a)

        var d = default((int Id, string? Name));
        Console.WriteLine(d.Name is null);             // Output: True
    }
}
```

**Interview traps**
- `default(T)` skips constructors (invariants not guaranteed for structs).
- Tuple names don't exist at runtime; `Item1`/`Item2` do.
- `ValueTuple` is a mutable struct → copy-semantics bugs in collections.
- `default` for generic `T` is `null` only for reference types.
- Deconstruction needs a `Deconstruct` method (not an interface); arity must match.

**Cheat sheet**
- `default` = zero bits; no constructor runs.
- `default` literal is target-typed; `default!` silences NRT.
- Tuples = `ValueTuple`, names compile-time only, `==` element-wise.
- `var (a, b) = x;`, `_` discards, `(a, b) = (b, a)` swap.
- Prefer records for public/serialized data; tuples for private helpers.
