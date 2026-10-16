# E03 – Coding Exercises & Puzzles (C#, LINQ, SQL)

## How to Use This File

1. **Try before you peek.** Read the **Problem**, write your answer (or code) on paper/IDE, only then open **Hint** and finally **Solution**.
2. **Run everything.** Code targets **.NET 8 / C# 12** with `ImplicitUsings` enabled (`System`, `System.Linq`, `System.Collections.Generic`, `System.Threading`, `System.Threading.Tasks`, `System.IO`, `System.Net.Http`). Extra `using`s are shown where needed. Snippets marked *top-level* can be pasted into a `Program.cs`; others are full types. C# requires top-level statements to come *before* type declarations, so where a puzzle lists types first and then a `// top-level` section, put that section at the top of `Program.cs` (or inside `Main`).
3. **Difficulty tags**: `[Easy]` ≈ 5 min, `[Medium]` ≈ 15 min, `[Hard]` ≈ 30+ min / senior-level.
4. **Explain out loud.** In interviews the reasoning matters more than the answer: state complexity, edge cases, and thread-safety assumptions.
5. **Follow-up variations** are what interviewers ask next – practise them.
6. SQL targets **SQL Server (T-SQL)** unless noted; the schema is defined once at the top of Part 4.

## Table of Contents

- [Part 1 – C# Output-Prediction Puzzles](#part-1--c-output-prediction-puzzles) (1.1 – 1.37)
- [Part 2 – C# Coding Tasks](#part-2--c-coding-tasks) (2.1 – 2.28)
- [Part 3 – LINQ Challenges](#part-3--linq-challenges) (3.1 – 3.18)
- [Part 4 – SQL Challenges](#part-4--sql-challenges) (4.1 – 4.28)

---

# Part 1 – C# Output-Prediction Puzzles

> Predict the output *before* reading the solution. Each puzzle lists code, output, and the reason.

### 1.1 [Easy] Static constructor order

**Problem**
```csharp
class A { static A() => Console.WriteLine("A static"); public A() => Console.WriteLine("A inst"); }
class B : A { static B() => Console.WriteLine("B static"); public B() => Console.WriteLine("B inst"); }

// top-level
new B();
new B();
```
**Hint**: When is a static constructor triggered, and what runs first when you `new` a derived type?

**Solution**
```
B static
A static
A inst
B inst
A inst
B inst
```
**Explanation / complexity**: A static constructor runs once, just before the first instance creation or first static member access of *that* type. `new B()` triggers `B`'s static ctor first; then B's instance ctor starts, which calls `A`'s ctor – that triggers `A`'s static ctor. Static ctors never run twice.

**Follow-up variations**: What if static fields with initializers but no explicit static ctor (`beforefieldinit`)? What if a static ctor throws (`TypeInitializationException`, type unusable)?

### 1.2 [Easy] Constructor chain & field initializer order

**Problem**
```csharp
class Base
{
    int x = Log("Base field");
    public Base() { Log("Base ctor"); }
    protected static int Log(string s) { Console.WriteLine(s); return 0; }
}
class Derived : Base
{
    int y = Log("Derived field");
    public Derived() { Log("Derived ctor"); }
}
// top-level
new Derived();
```
**Hint**: Field initializers are compiled into the constructor *before* the base-constructor call.

**Solution**
```
Derived field
Base field
Base ctor
Derived ctor
```
**Explanation / complexity**: Order: derived field initializers → base field initializers → base ctor body → derived ctor body. (VB.NET differs – initializers run after the base ctor.)

**Follow-up variations**: Add `: this(...)` chaining – which initializers run once? (Only in the ctor that calls `base`.)

### 1.3 [Medium] Virtual call in constructor

**Problem**
```csharp
class Base
{
    public Base() { Show(); }
    public virtual void Show() => Console.WriteLine("Base");
}
class Derived : Base
{
    string s;
    public Derived() { s = "set"; }
    public override void Show() => Console.WriteLine(s ?? "null");
}
// top-level
new Derived();
```
**Hint**: Which constructor body has run when `Show()` is invoked?

**Solution**
```
null
```
**Explanation / complexity**: Virtual dispatch uses the *runtime* type (`Derived`) even during the base ctor, but `Derived`'s ctor body hasn't run, so `s` is still `null`. (Had `s` a field initializer, it would already be assigned.) Avoid virtual calls in constructors.

**Follow-up variations**: Make `s` initialized inline (`string s = "init";`) → prints `init`. Analyzer CA2214.

### 1.4 [Easy] Method hiding vs overriding

**Problem**
```csharp
class A { public virtual void F() => Console.WriteLine("A.F"); public void G() => Console.WriteLine("A.G"); }
class B : A { public override void F() => Console.WriteLine("B.F"); public new void G() => Console.WriteLine("B.G"); }

// top-level
A a = new B();
a.F();
a.G();
((B)a).G();
```
**Hint**: `override` = runtime binding; `new` = compile-time binding.

**Solution**
```
B.F
A.G
B.G
```
**Explanation / complexity**: `F` is virtual → dispatched on runtime type. `G` is hidden, not overridden → chosen by the *static* type of the expression.

**Follow-up variations**: What if `B.G` omits `new`? (Warning CS0108, still hides.) What about `sealed override`?

### 1.5 [Medium] Closure captured in a `for` loop vs `foreach`

**Problem**
```csharp
var actions = new List<Action>();
for (int i = 0; i < 3; i++) actions.Add(() => Console.Write(i + " "));
foreach (var a in actions) a();
Console.WriteLine();

actions.Clear();
foreach (var n in new[] { 0, 1, 2 }) actions.Add(() => Console.Write(n + " "));
foreach (var a in actions) a();
```
**Hint**: How many distinct variables does each loop create?

**Solution**
```
3 3 3 
0 1 2 
```
**Explanation / complexity**: A `for` loop has a single `i` shared by all closures; after the loop it is 3. Since C# 5, `foreach` declares a fresh iteration variable per iteration. Fix for `for`: copy to a local inside the body (`var j = i;`).

**Follow-up variations**: `Task.Run(() => Console.Write(i))` inside a `for` loop – same bug, nondeterministic output.

### 1.6 [Medium] String immutability & interning

**Problem**
```csharp
string a = "hello";
string b = "hel" + "lo";
string c = new string("hello".ToCharArray());
string d = string.Intern(c);

Console.WriteLine(ReferenceEquals(a, b));
Console.WriteLine(ReferenceEquals(a, c));
Console.WriteLine(a == c);
Console.WriteLine(ReferenceEquals(a, d));

string s = "abc";
s.ToUpper();
Console.WriteLine(s);
```
**Hint**: Compile-time constants are interned; `==` on strings compares values.

**Solution**
```
True
False
True
True
abc
```
**Explanation / complexity**: `"hel" + "lo"` is folded at compile time into the same literal, interned in the intern pool. `new string(...)` allocates a new object. `string.Intern` returns the pooled reference. Strings are immutable – `ToUpper()` returns a new string that was discarded.

**Follow-up variations**: Why use `StringBuilder` for loops? Cost of `+=` in a loop is O(n²).

### 1.7 [Medium] Boxing and equality

**Problem**
```csharp
int i = 5;
object o1 = i, o2 = i;
long l = 5;

Console.WriteLine(o1 == o2);
Console.WriteLine(o1.Equals(o2));
Console.WriteLine((int)o1 == (int)o2);
Console.WriteLine(o1.Equals(l));
```
**Hint**: How many boxes were created? What does `object.Equals` check first?

**Solution**
```
False
True
True
False
```
**Explanation / complexity**: `o1` and `o2` are two separate boxes; `object == object` is reference equality. `Int32.Equals(object)` compares values *and* requires the argument to be an `int`. Boxing `l` yields a boxed `long`, so `Equals` returns false.

**Follow-up variations**: `object.ReferenceEquals(1, 1)` (False – two boxes). Cached boxes? (None for ints in .NET.)

### 1.8 [Medium] Struct copy semantics

**Problem**
```csharp
struct P { public int X; public void Inc() => X++; }
class H { public readonly P pt = new P(); }

// top-level
var p = new P(); p.Inc();
Console.WriteLine(p.X);

var list = new List<P> { new P() };
list[0].Inc();
Console.WriteLine(list[0].X);

var h = new H();
h.pt.Inc();
Console.WriteLine(h.pt.X);
```
**Hint**: Where does a struct get copied?

**Solution**
```
1
0
0
```
**Explanation / complexity**: A local variable is mutated in place. The `List<T>` indexer returns a *copy*, so `Inc` mutates a temporary. A `readonly` field of struct type is accessed through a defensive copy for non-readonly members. Mutable structs are a code smell; prefer `readonly struct`.

**Follow-up variations**: With an array `arr[0].Inc()` the element *is* a variable → mutates (prints 1).

### 1.9 [Easy] Deferred LINQ execution

**Problem**
```csharp
var list = new List<int> { 1, 2, 3 };
var q = list.Where(x => x > 1);
var q2 = list.Where(x => x > 1).ToList();
list.Add(4);
Console.WriteLine(string.Join(",", q));
Console.WriteLine(q2.Count);
```
**Hint**: When does `Where` actually read the source?

**Solution**
```
2,3,4
2
```
**Explanation / complexity**: `q` is just a query definition and is evaluated at enumeration (after `Add(4)`). `ToList()` materialises immediately, so `q2` holds `{2,3}` → count 2.

**Follow-up variations**: Modify the list *while* enumerating → `InvalidOperationException`. Which operators are streaming vs buffering (`OrderBy`, `GroupBy`)?

### 1.10 [Medium] Multiple enumeration side effects

**Problem**
```csharp
var q = new[] { 1, 2 }.Select(x => { Console.Write($"S{x} "); return x; });
foreach (var x in q) { }
foreach (var x in q) { }
```
**Hint**: Each `foreach` calls `GetEnumerator()` again.

**Solution**
```
S1 S2 S1 S2 
```
**Explanation / complexity**: The projection runs for every enumeration. Call `ToList()`/`ToArray()` once if the source is expensive (database query, HTTP). Resharper warns "possible multiple enumeration".

**Follow-up variations**: What does `q.Count()` followed by `q.ToList()` print? How does EF Core treat `IQueryable` multiple enumeration (2 SQL roundtrips)?

### 1.11 [Medium] `yield return` laziness

**Problem**
```csharp
static IEnumerable<int> Gen()
{
    Console.WriteLine("start");
    yield return 1;
    Console.WriteLine("mid");
    yield return 2;
    Console.WriteLine("end");
}

// top-level
var g = Gen();
Console.WriteLine("created");
foreach (var x in g) Console.WriteLine(x);
```
**Hint**: Calling an iterator method does not run its body.

**Solution**
```
created
start
1
mid
2
end
```
**Explanation / complexity**: The compiler turns `Gen` into a state machine; code runs on each `MoveNext()`. Nothing runs at the call site, which is why `Gen()` printed nothing before `created`.

**Follow-up variations**: What does `Gen().First()` print? (`start`, then 1 – the `finally`/dispose happens but `mid` is never reached.)

### 1.12 [Medium] Argument validation in iterators

**Problem**
```csharp
static IEnumerable<int> Take(int n)
{
    if (n < 0) throw new ArgumentException("n");
    yield return n;
}

// top-level
var e = Take(-1);
Console.WriteLine("ok");
try { foreach (var x in e) { } }
catch (ArgumentException) { Console.WriteLine("caught"); }
```
**Hint**: When does the guard clause execute?

**Solution**
```
ok
caught
```
**Explanation / complexity**: The exception is thrown on first `MoveNext()`, not at call time. Standard fix: a non-iterator public method validates, then returns a private iterator (this is how `Enumerable.Take` behaves). C# also allows a `static` local iterator function inside the validating method.

**Follow-up variations**: Same issue with `async` methods? (There the exception is captured in the returned `Task`.)

### 1.13 [Medium] Async ordering

**Problem**
```csharp
static class Program
{
    static async Task Main()
    {
        Console.WriteLine("1");
        var t = Work();
        Console.WriteLine("3");
        await t;
        Console.WriteLine("5");
    }
    static async Task Work()
    {
        Console.WriteLine("2");
        await Task.Delay(100);
        Console.WriteLine("4");
    }
}
```
**Hint**: An async method runs synchronously until its first incomplete `await`.

**Solution**
```
1
2
3
4
5
```
**Explanation / complexity**: `Work()` executes synchronously up to `await Task.Delay`, printing `2`, then returns an incomplete task to `Main`, which prints `3`. After the delay the continuation prints `4`; `Main` resumes and prints `5`.

**Follow-up variations**: Replace `Task.Delay(100)` with `Task.CompletedTask` – order becomes 1,2,4,3,5 (the await completes synchronously).

### 1.14 [Hard] `.Result` deadlock and `ConfigureAwait(false)`

**Problem**
```csharp
// Imagine this runs on a UI thread / classic ASP.NET (a SynchronizationContext with ONE thread)
string GetData()
{
    return GetDataAsync().Result;            // (A)
}
async Task<string> GetDataAsync()
{
    await Task.Delay(100);                   // (B)
    return "done";
}
```
What happens at (A)? How do you fix it?

**Hint**: Who must run the continuation after (B)? Who is blocking that thread?

**Solution**
```csharp
// Fix 1 (best): async all the way
async Task<string> GetDataAsyncAllTheWay() => await GetDataAsync();

// Fix 2 (library code): don't capture the context
async Task<string> GetDataAsync()
{
    await Task.Delay(100).ConfigureAwait(false);
    return "done";
}
```
Result: with the original code on a single-threaded context the call **hangs forever (deadlock)**. In a console app or ASP.NET Core (no `SynchronizationContext`) it simply returns `"done"`.

**Explanation / complexity**: `await` captures the current `SynchronizationContext`; the continuation is posted back to it. The UI thread is blocked in `.Result`, so the continuation never runs → deadlock. `ConfigureAwait(false)` lets the continuation run on a thread-pool thread. Rule: library code uses `ConfigureAwait(false)`; app-level code (UI) doesn't; never block on async (`.Result`, `.Wait()`, `.GetAwaiter().GetResult()`).

**Follow-up variations**: Does ASP.NET Core need `ConfigureAwait(false)`? (No context, so no deadlock, but harmless.) What does `ConfigureAwait(ConfigureAwaitOptions.SuppressThrowing)` do in .NET 8?

### 1.15 [Medium] `finally` and `return`

**Problem**
```csharp
static int F()
{
    try { return 1; }
    finally { Console.WriteLine("finally"); }
}
static int G()
{
    int x = 1;
    try { return x; }
    finally { x = 2; }
}
// top-level
Console.WriteLine(F());
Console.WriteLine(G());
```
**Hint**: When is the return value evaluated relative to `finally`?

**Solution**
```
finally
1
1
```
**Explanation / complexity**: The return value is evaluated and stored *before* `finally` runs; `finally` can mutate the local but not the already-saved value (for value types / immutable refs). C# forbids `return` inside `finally` (CS0157). Note `F()` runs (and prints `finally`) before `Console.WriteLine` prints its result.

**Follow-up variations**: What if `G` returned a `List<int>` and `finally` added an item? (The caller sees the item – same reference.)

### 1.16 [Medium] `throw;` vs `throw ex;`

**Problem**
```csharp
static void Thrower() => throw new InvalidOperationException("boom");
static void A() { try { Thrower(); } catch (Exception ex) { throw ex; } }
static void B() { try { Thrower(); } catch { throw; } }

// top-level
try { A(); } catch (Exception e) { Console.WriteLine(e.StackTrace!.Contains("Thrower")); }
try { B(); } catch (Exception e) { Console.WriteLine(e.StackTrace!.Contains("Thrower")); }
```
**Hint**: Which one resets the stack trace?

**Solution**
```
False
True
```
**Explanation / complexity**: `throw ex;` restarts the stack trace at the rethrow point, losing the origin. `throw;` preserves it. To wrap, use `throw new X("msg", ex)`; to rethrow later use `ExceptionDispatchInfo.Capture(ex).Throw()`.

**Follow-up variations**: How does `await` rethrow preserve stack traces? (`ExceptionDispatchInfo`.)

### 1.17 [Medium] Exception filter runs before inner `finally`

**Problem**
```csharp
static bool Log(string s) { Console.WriteLine(s); return true; }
static void Run()
{
    try
    {
        try { throw new Exception("x"); }
        finally { Console.WriteLine("inner finally"); }
    }
    catch (Exception) when (Log("filter"))
    {
        Console.WriteLine("catch");
    }
}
// top-level
Run();
```
**Hint**: Exception handling has two passes.

**Solution**
```
filter
inner finally
catch
```
**Explanation / complexity**: Pass 1 walks up the stack evaluating filters to find a handler (no unwinding yet); pass 2 unwinds, running `finally` blocks, then enters the `catch`. A filter that returns `false` doesn't unwind – great for logging without catching.

**Follow-up variations**: Security implication: filters run before finally, so don't rely on `finally` to revert privilege state.

### 1.18 [Medium] Missing `GetHashCode`/`Equals` as Dictionary key

**Problem**
```csharp
class K { public int Id; public K(int id) => Id = id; }
record R(int Id);

// top-level
var d1 = new Dictionary<K, string> { [new K(1)] = "a" };
Console.WriteLine(d1.ContainsKey(new K(1)));

var d2 = new Dictionary<R, string> { [new R(1)] = "a" };
Console.WriteLine(d2.ContainsKey(new R(1)));
```
**Hint**: Default equality for classes is reference equality; records generate value equality.

**Solution**
```
False
True
```
**Explanation / complexity**: Classes inherit `object.Equals`/`GetHashCode` (identity). Records synthesise both over their properties. Contract: equal objects must have equal hash codes. Lookup is O(1) average.

**Follow-up variations**: Overriding only `Equals` (no `GetHashCode`) → compiler warning CS0659 and lookups usually fail. Use `HashCode.Combine(...)`.

### 1.19 [Hard] Mutating a dictionary key

**Problem**
```csharp
class MK
{
    public int Id;
    public override int GetHashCode() => Id;
    public override bool Equals(object? o) => o is MK m && m.Id == Id;
}
// top-level
var k = new MK { Id = 1 };
var d = new Dictionary<MK, int> { [k] = 1 };
k.Id = 2;
Console.WriteLine(d.ContainsKey(k));
Console.WriteLine(d.ContainsKey(new MK { Id = 1 }));
Console.WriteLine(d.Count);
```
**Hint**: The dictionary stored the *old* hash code in its bucket.

**Solution**
```
False
False
1
```
**Explanation / complexity**: `ContainsKey(k)` hashes to 2 → wrong bucket → miss. `ContainsKey(new MK{Id=1})` finds the entry by hash 1, but `Equals` compares against the stored (now Id=2) key → false. The entry is orphaned but still counted. Keys must be immutable w.r.t. equality.

**Follow-up variations**: Same issue for `HashSet<T>`. Why are `string`/`record struct` good keys?

### 1.20 [Medium] Nullable lifted operators

**Problem**
```csharp
static string S<T>(T? v) where T : struct => v?.ToString() ?? "null";

int? a = null, b = 5;
bool? t = true, n = null;

Console.WriteLine(S(a + b));
Console.WriteLine(a == null);
Console.WriteLine(a < b);
Console.WriteLine(a > b);
Console.WriteLine(a != b);
Console.WriteLine(S(t | n));
Console.WriteLine(S(t & n));
```
**Hint**: Arithmetic with null yields null; relational comparisons with null yield `false`; `bool?` uses three-valued logic.

**Solution**
```
null
True
False
False
True
True
null
```
**Explanation / complexity**: Lifted arithmetic returns `null` if any operand is null. `<`, `>`, `<=`, `>=` return `false` when either operand is null; `==` is true only if both null or both equal, `!=` the inverse. `true | null` = `true` (short-circuit logic); `true & null` = `null`.

**Follow-up variations**: `bool? ` in an `if` needs explicit `== true`. Compare with SQL `NULL` semantics (`NULL = NULL` is unknown).

### 1.21 [Easy] Integer overflow

**Problem**
```csharp
int max = int.MaxValue;
Console.WriteLine(max + 1);

try { checked { int r = max + 1; Console.WriteLine(r); } }
catch (OverflowException) { Console.WriteLine("overflow"); }

int big = 300;
Console.WriteLine((byte)big);
try { Console.WriteLine(Math.Abs(int.MinValue)); }
catch (OverflowException) { Console.WriteLine("abs overflow"); }
```
**Hint**: The default context is `unchecked` (unless the project sets `<CheckForOverflowUnderflow>`).

**Solution**
```
-2147483648
overflow
44
abs overflow
```
**Explanation / complexity**: Two's-complement wraparound by default. `checked` throws. `300 mod 256 = 44`. `Math.Abs(int.MinValue)` cannot be represented → throws. (`int.MaxValue + 1` written as a *constant* expression is a compile error CS0220, which is why a variable is used.)

**Follow-up variations**: Average of two ints `(a+b)/2` overflow bug in binary search – use `lo + (hi - lo) / 2`.

### 1.22 [Easy] Floating-point comparison

**Problem**
```csharp
double x = 0.1, y = 0.2, z = 0.3;
Console.WriteLine(x + y == z);
Console.WriteLine(Math.Abs(x + y - z) < 1e-9);
Console.WriteLine(0.1m + 0.2m == 0.3m);
Console.WriteLine(double.NaN == double.NaN);
Console.WriteLine(double.NaN.Equals(double.NaN));
```
**Hint**: Binary floating point can't represent 0.1 exactly.

**Solution**
```
False
True
True
False
True
```
**Explanation / complexity**: `0.1 + 0.2 = 0.30000000000000004`. Use tolerance or `decimal` (base-10, for money). `NaN == NaN` is false by IEEE-754, but `Equals` is reflexive so that collections work.

**Follow-up variations**: `float.Epsilon` is *not* a good tolerance. Why `decimal` is slower but exact for decimal fractions.

### 1.23 [Easy] `ref` vs value parameter

**Problem**
```csharp
static void Change(List<int> l) { l.Add(1); l = new List<int> { 9, 9 }; }
static void ChangeRef(ref List<int> l) { l = new List<int> { 9, 9 }; }

// top-level
var list = new List<int>();
Change(list);
Console.WriteLine(list.Count);
ChangeRef(ref list);
Console.WriteLine(list.Count);
```
**Hint**: A reference is passed *by value* by default.

**Solution**
```
1
2
```
**Explanation / complexity**: `Change` mutates the shared object (Add visible) but reassigning its local copy of the reference doesn't affect the caller. With `ref`, the parameter aliases the caller's variable.

**Follow-up variations**: `out` vs `ref` vs `in`; `ref` locals/returns; passing a struct by `in` to avoid a copy.

### 1.24 [Medium] `params` ambiguity

**Problem**
```csharp
static class Program
{
    static void F(params int[] a) => Console.WriteLine("params");
    static void F(int a, int b) => Console.WriteLine("two");
    static void P(params object[] a) => Console.WriteLine(a?.Length.ToString() ?? "null");

    static void Main()
    {
        F(1, 2);
        F(1);
        F();
        F(new[] { 1, 2 });
        P(null);
        P((object)null!);
        P(new int[] { 1, 2 });
        P(new string[] { "a", "b" });
    }
}
```
**Hint**: Normal form beats expanded form; array covariance applies to `string[]` → `object[]`.

**Solution**
```
two
params
params
params
null
1
1
2
```
**Explanation / complexity**: `F(1,2)` matches `(int,int)` in normal form – better than the expanded `params`. `P(null)` passes `null` as the array itself. `(object)null` is wrapped into a 1-element array. `int[]` is not an `object[]` (no covariance for value types) so it's wrapped → length 1; `string[]` converts to `object[]` → length 2.

**Follow-up variations**: In .NET 9 / C# 13, `params ReadOnlySpan<T>` & `params IEnumerable<T>` – overload ranking changes.

### 1.25 [Medium] Overload resolution – numeric conversions

**Problem**
```csharp
static class Program
{
    static void M(long x)   => Console.WriteLine("long");
    static void M(double x) => Console.WriteLine("double");
    static void M(object x) => Console.WriteLine("object");
    static void Main()
    {
        byte b = 1; int i = 1; float f = 1f;
        M(b);
        M(i);
        M(f);
        M("s");
    }
}
```
**Hint**: Pick the "better conversion": if one target type converts implicitly to the other (and not vice-versa), it's better.

**Solution**
```
long
long
double
object
```
**Explanation / complexity**: `byte`/`int` → `long` is better than → `double` (since `long` converts implicitly to `double`, not the reverse) and better than boxing to `object`. `float` can't convert to `long`, so `double` wins over `object`.

**Follow-up variations**: What if `M(int)` is added? Generic vs non-generic overloads – the non-generic wins on a tie.

### 1.26 [Hard] Derived-class overloads win over base overrides

**Problem**
```csharp
class B { public virtual void F(int x) => Console.WriteLine("B.F(int)"); }
class D : B
{
    public override void F(int x) => Console.WriteLine("D.F(int)");
    public void F(object x) => Console.WriteLine("D.F(object)");
}
// top-level
new D().F(1);
```
**Hint**: Overrides don't "count" as declared in the derived class during lookup.

**Solution**
```
D.F(object)
```
**Explanation / complexity**: Member lookup starts at the most-derived class and considers only methods *declared* there (overrides excluded). `F(object)` is applicable (via boxing), so the search stops before reaching `F(int)`.

**Follow-up variations**: Cast `((B)new D()).F(1)` → `D.F(int)`. Why interface default methods behave differently.

### 1.27 [Medium] Extension vs instance method

**Problem**
```csharp
class C { public void Hi() => Console.WriteLine("instance"); }
static class E
{
    public static void Hi(this C c) => Console.WriteLine("extension");
    public static void Hi(this C c, int x) => Console.WriteLine("ext int");
}
// top-level
new C().Hi();
new C().Hi(1);
```
**Hint**: Extension methods are only considered when no applicable instance method exists.

**Solution**
```
instance
ext int
```
**Explanation / complexity**: Instance methods always win if applicable. For `Hi(1)` no instance method is applicable, so extension methods are considered.

**Follow-up variations**: What if the instance method is added later to a class you don't own? (Silent behaviour change – a versioning hazard.)

### 1.28 [Medium] Event unsubscribe with lambdas

**Problem**
```csharp
class Pub { public event Action? E; public void Fire() => E?.Invoke(); }

// top-level
var p = new Pub();
p.E += () => Console.WriteLine("A");
p.E -= () => Console.WriteLine("A");
p.Fire();
Console.WriteLine("--");

Action h = () => Console.WriteLine("B");
p.E += h; p.E += h;
p.E -= h;
p.Fire();
```
**Hint**: Delegate equality = same method + same target.

**Solution**
```
A
--
A
B
```
**Explanation / complexity**: The two lambdas compile to different methods, so `-=` removes nothing and `A` still fires. `h` is added twice and removed once (removal takes the *last* matching occurrence), so one `B` remains. Lesson: keep a delegate reference (or named method) to unsubscribe; unremoved handlers cause memory leaks.

**Follow-up variations**: Weak events; why static events are leak-prone.

### 1.29 [Easy] `using` / `IDisposable` order

**Problem**
```csharp
class R : IDisposable
{
    readonly string n;
    public R(string n) { this.n = n; Console.WriteLine("open " + n); }
    public void Dispose() => Console.WriteLine("close " + n);
}
static class Program
{
    static void Main()
    {
        using var a = new R("A");
        using var b = new R("B");
        Console.WriteLine("body");
    }
}
```
**Hint**: Disposal is LIFO.

**Solution**
```
open A
open B
body
close B
close A
```
**Explanation / complexity**: `using` declarations dispose at end of the enclosing scope in reverse order of declaration. If `new R("B")` threw, only `A` would be disposed.

**Follow-up variations**: `await using` / `IAsyncDisposable`; Dispose pattern with finalizer and `GC.SuppressFinalize`.

### 1.30 [Medium] `Lazy<T>`

**Problem**
```csharp
var lz = new Lazy<int>(() => { Console.WriteLine("init"); return 42; });
Console.WriteLine("created");
Console.WriteLine(lz.IsValueCreated);
Console.WriteLine(lz.Value);
Console.WriteLine(lz.Value);
Console.WriteLine(lz.IsValueCreated);
```
**Hint**: Factory runs on first `.Value` only.

**Solution**
```
created
False
init
42
42
True
```
**Explanation / complexity**: Default mode `ExecutionAndPublication` is thread-safe: one factory execution, result cached. If the factory throws, that exception is also cached and rethrown on every access (use `LazyThreadSafetyMode.PublicationOnly` to retry).

**Follow-up variations**: `Lazy<T>` vs `static readonly` initialisation vs `LazyInitializer`.

### 1.31 [Easy] Captured variable is shared, not copied

**Problem**
```csharp
int x = 1;
Func<int> f = () => x;
x = 2;
Console.WriteLine(f());
```
**Hint**: The closure captures the variable, not the value.

**Solution**
```
2
```
**Explanation / complexity**: The compiler hoists `x` into a closure class; both the method and lambda use the same field.

**Follow-up variations**: Static lambdas (`static () => x` is a compile error) prevent accidental capture.

### 1.32 [Easy] `char`/`string` arithmetic

**Problem**
```csharp
Console.WriteLine('a' + 1);
Console.WriteLine((char)('a' + 1));
Console.WriteLine("" + 'a' + 1);
Console.WriteLine(1 + 2 + "3" + 4 + 5);
```
**Hint**: Evaluation is left to right; `char + int` is integer arithmetic.

**Solution**
```
98
b
a1
3345
```
**Explanation / complexity**: `'a'` is 97. `1 + 2` = 3 before string concatenation starts; afterwards `"3"+4` = `"34"`, then `"345"`→ `"3345"`.

**Follow-up variations**: `"" + (1 + 2)`; interpolated string `$"{1}{2}"`.

### 1.33 [Easy] Integer division, modulo, rounding

**Problem**
```csharp
int a = -7, b = 2, c = 3;
Console.WriteLine(a / b);
Console.WriteLine(a % c);
Console.WriteLine(7 / 2.0);
Console.WriteLine(Math.Round(2.5));
Console.WriteLine(Math.Round(3.5));
Console.WriteLine(Math.Round(2.5, MidpointRounding.AwayFromZero));
```
**Hint**: Integer division truncates toward zero; default rounding is banker's.

**Solution**
```
-3
-1
3.5
2
4
3
```
**Explanation / complexity**: Remainder takes the sign of the dividend. `Math.Round` defaults to `ToEven`, halving bias in aggregates.

**Follow-up variations**: Positive modulo `((a % n) + n) % n`; `Math.Floor` vs `Math.Truncate` for negatives.

### 1.34 [Medium] Statics in generic classes

**Problem**
```csharp
class Counter<T> { public static int N; }

// top-level
Counter<int>.N++;
Counter<string>.N += 2;
Counter<int>.N++;
Console.WriteLine(Counter<int>.N);
Console.WriteLine(Counter<string>.N);
Console.WriteLine(Counter<long>.N);
```
**Hint**: `Counter<int>` and `Counter<string>` are different runtime types.

**Solution**
```
2
2
0
```
**Explanation / complexity**: Each closed constructed type has its own static fields (and static constructor run). Handy for per-type caches (`EqualityComparer<T>.Default`).

**Follow-up variations**: Reference-type generic arguments share *code* but still have separate statics.

### 1.35 [Hard] `Task.WhenAll` and multiple exceptions

**Problem**
```csharp
static class Program
{
    static async Task Fail(int n) { await Task.Yield(); throw new Exception("fail" + n); }

    static async Task Main()
    {
        var t = Task.WhenAll(Fail(1), Fail(2));
        try { await t; }
        catch (Exception e) { Console.WriteLine(e.Message); }
        Console.WriteLine(t.Exception!.InnerExceptions.Count);
    }
}
```
**Hint**: `await` unwraps an `AggregateException` to its first inner exception.

**Solution**
```
fail1
2
```
**Explanation / complexity**: `WhenAll` waits for *all* tasks and aggregates all exceptions (in input order). `await` rethrows only the first; the full set is on `task.Exception`.

**Follow-up variations**: `Task.WaitAll` throws `AggregateException`. `Task.WhenAny` doesn't observe the losers' exceptions.

### 1.36 [Medium] Records – equality and `with`

**Problem**
```csharp
record Person(string Name, int Age);

// top-level
var a = new Person("Ann", 30);
var b = new Person("Ann", 30);
var c = a with { Age = 31 };
Console.WriteLine(a == b);
Console.WriteLine(ReferenceEquals(a, b));
Console.WriteLine(a == c);
Console.WriteLine(c);
```
**Hint**: Records compare by value; `with` creates a copy.

**Solution**
```
True
False
False
Person { Name = Ann, Age = 31 }
```
**Explanation / complexity**: Compiler-generated `Equals`, `==`, `GetHashCode`, `ToString` and a clone method for `with`. Positional records are immutable (`init`) by default; `record struct` is a value type.

**Follow-up variations**: Records with a `List<T>` property – equality uses the list's reference equality.

### 1.37 [Hard] `static` init order across fields

**Problem**
```csharp
static class Cfg
{
    public static readonly int A = B + 1;
    public static readonly int B = 10;
}
// top-level
Console.WriteLine(Cfg.A);
Console.WriteLine(Cfg.B);
```
**Hint**: Static field initialisers run in textual order.

**Solution**
```
1
10
```
**Explanation / complexity**: `A` is initialised first when `B` is still `0` → `A = 1`. (With `const` instead of `static readonly`, the compiler would inline values and give 11.) Reordering fields changes behaviour – a classic bug.

**Follow-up variations**: Circular static dependencies across classes; `const` vs `static readonly` and assembly-versioning.

---

# Part 2 – C# Coding Tasks

> Each solution is a complete, compilable snippet (.NET 8, implicit usings). *Top-level* snippets go in `Program.cs`; types declared after top-level statements are allowed. State complexity and edge cases out loud.

### 2.1 [Easy] Reverse a string

**Problem**: Reverse `"hello"` without using `Enumerable.Reverse` on a string.

**Hint**: `char[]` + `Array.Reverse`, or two-pointer swap.

**Solution**
```csharp
static string Reverse(string s)
{
    var a = s.ToCharArray();
    Array.Reverse(a);
    return new string(a);
}
Console.WriteLine(Reverse("hello"));
```
Output: `olleh`

**Explanation / complexity**: O(n) time, O(n) extra space (strings are immutable). Two-pointer swap on the `char[]` is the same complexity.

**Follow-up variations**: Handle surrogate pairs / combining characters (`StringInfo`/`TextElementEnumerator`). Reverse in place with `string.Create` or `Span<char>`.

### 2.2 [Easy] Reverse words in a sentence

**Problem**: `"the sky is blue"` → `"blue is sky the"`; collapse extra spaces.

**Hint**: Split with `RemoveEmptyEntries`, reverse the array, join.

**Solution**
```csharp
static string ReverseWords(string s)
{
    var w = s.Split(' ', StringSplitOptions.RemoveEmptyEntries);
    Array.Reverse(w);
    return string.Join(" ", w);
}
Console.WriteLine(ReverseWords("  the sky   is blue "));
```
Output: `blue is sky the`

**Explanation / complexity**: O(n) time/space.

**Follow-up variations**: Reverse the letters of each word but keep word order. In-place on `char[]` (reverse whole, then each word).

### 2.3 [Easy] Palindrome check

**Problem**: Ignore case and non-alphanumerics: `"A man, a plan, a canal: Panama"` is a palindrome.

**Hint**: Two pointers moving inward, skipping non-letters/digits.

**Solution**
```csharp
static bool IsPalindrome(string s)
{
    int i = 0, j = s.Length - 1;
    while (i < j)
    {
        if (!char.IsLetterOrDigit(s[i])) { i++; continue; }
        if (!char.IsLetterOrDigit(s[j])) { j--; continue; }
        if (char.ToLowerInvariant(s[i]) != char.ToLowerInvariant(s[j])) return false;
        i++; j--;
    }
    return true;
}
Console.WriteLine(IsPalindrome("A man, a plan, a canal: Panama"));
Console.WriteLine(IsPalindrome("race a car"));
```
Output:
```
True
False
```
**Explanation / complexity**: O(n) time, O(1) space (no allocation).

**Follow-up variations**: Longest palindromic substring (expand around centre, O(n²)). Palindrome number without converting to string.

### 2.4 [Easy] Anagram check

**Problem**: Are `"listen"` and `"silent"` anagrams?

**Hint**: Compare character frequency counts (O(n)) instead of sorting (O(n log n)).

**Solution**
```csharp
static bool IsAnagram(string a, string b)
{
    if (a.Length != b.Length) return false;
    var counts = new Dictionary<char, int>();
    foreach (var c in a) counts[c] = counts.GetValueOrDefault(c) + 1;
    foreach (var c in b)
    {
        if (!counts.TryGetValue(c, out var n) || n == 0) return false;
        counts[c] = n - 1;
    }
    return true;
}
Console.WriteLine(IsAnagram("listen", "silent"));
Console.WriteLine(IsAnagram("rat", "car"));
```
Output:
```
True
False
```
**Explanation / complexity**: O(n) time, O(k) space (k = distinct chars). For lowercase a–z use an `int[26]`.

**Follow-up variations**: Case-insensitive, ignore spaces. Check if `b` contains an anagram of `a` (sliding window).

### 2.5 [Easy] FizzBuzz and a configurable variant

**Problem**: Print 1..15 with Fizz/Buzz; then generalise to arbitrary rules (add 7 → `Bazz`).

**Hint**: Concatenate matching words; fall back to the number.

**Solution**
```csharp
static string Fb(int n, (int Div, string Word)[] rules)
{
    var s = string.Concat(rules.Where(r => n % r.Div == 0).Select(r => r.Word));
    return s.Length == 0 ? n.ToString() : s;
}
var classic = new[] { (3, "Fizz"), (5, "Buzz") };
var ext = new[] { (3, "Fizz"), (5, "Buzz"), (7, "Bazz") };
Console.WriteLine(string.Join(" ", Enumerable.Range(1, 15).Select(n => Fb(n, classic))));
Console.WriteLine(string.Join(" ", Enumerable.Range(1, 15).Select(n => Fb(n, ext))));
```
Output:
```
1 2 Fizz 4 Buzz Fizz 7 8 Fizz Buzz 11 Fizz 13 14 FizzBuzz
1 2 Fizz 4 Buzz Fizz Bazz 8 Fizz Buzz 11 Fizz 13 Bazz FizzBuzz
```
**Explanation / complexity**: O(n·r) for r rules. No `if (n%15==0)` special-casing, so new rules are just data (Open/Closed).

**Follow-up variations**: Lazy infinite sequence with `yield`; no modulo operator; print without `if`/`switch`.

### 2.6 [Easy] Find duplicates

**Problem**: Return each duplicated value once from `{1,2,3,2,4,3,5,2}`.

**Hint**: `HashSet.Add` returns `false` for an existing element.

**Solution**
```csharp
static IEnumerable<int> Dups(IEnumerable<int> xs)
{
    var seen = new HashSet<int>();
    var reported = new HashSet<int>();
    foreach (var x in xs)
        if (!seen.Add(x) && reported.Add(x)) yield return x;
}
Console.WriteLine(string.Join(" ", Dups(new[] { 1, 2, 3, 2, 4, 3, 5, 2 })));
```
Output: `2 3`

**Explanation / complexity**: O(n) time, O(n) space. LINQ alternative: `xs.GroupBy(x => x).Where(g => g.Count() > 1).Select(g => g.Key)`.

**Follow-up variations**: O(1) extra space when values are 1..n (negate-index marking). Count occurrences. Duplicates in a stream that doesn't fit in memory.

### 2.7 [Easy] First non-repeating character

**Problem**: `"swiss"` → `w`; `"aabb"` → none.

**Hint**: Two passes – count, then scan in original order.

**Solution**
```csharp
static char? FirstUnique(string s)
{
    var counts = new Dictionary<char, int>();
    foreach (var c in s) counts[c] = counts.GetValueOrDefault(c) + 1;
    foreach (var c in s) if (counts[c] == 1) return c;
    return null;
}
Console.WriteLine(FirstUnique("swiss"));
Console.WriteLine(FirstUnique("aabb")?.ToString() ?? "none");
```
Output:
```
w
none
```
**Explanation / complexity**: O(n) time, O(k) space.

**Follow-up variations**: Single pass for a stream (ordered dictionary / linked list of candidates). Return the index.

### 2.8 [Easy] Two-sum

**Problem**: Return indices of two numbers adding to `target`: `{2,7,11,15}`, 9.

**Hint**: Store each seen value → index; look up the complement.

**Solution**
```csharp
static (int, int)? TwoSum(int[] a, int target)
{
    var seen = new Dictionary<int, int>();
    for (int i = 0; i < a.Length; i++)
    {
        if (seen.TryGetValue(target - a[i], out var j)) return (j, i);
        seen[a[i]] = i;
    }
    return null;
}
Console.WriteLine(TwoSum(new[] { 2, 7, 11, 15 }, 9));
Console.WriteLine(TwoSum(new[] { 3, 3 }, 6));
```
Output:
```
(0, 1)
(0, 1)
```
**Explanation / complexity**: O(n) time, O(n) space vs. O(n²) brute force. Looking up *before* inserting handles duplicates and prevents using one element twice.

**Follow-up variations**: Sorted input → two pointers O(1) space. 3-sum. Return all pairs.

### 2.9 [Hard] LRU cache

**Problem**: Implement `Get`/`Put` in O(1) with a fixed capacity; evict the least recently used. Make it thread-safe.

**Hint**: Dictionary for lookup + doubly linked list for recency order.

**Solution**
```csharp
var cache = new LruCache<int, string>(2);
cache.Put(1, "a");
cache.Put(2, "b");
cache.TryGet(1, out var v1);
cache.Put(3, "c");                       // evicts key 2
Console.WriteLine(v1);
Console.WriteLine(cache.TryGet(2, out _));
Console.WriteLine(cache.TryGet(3, out var v3) ? v3 : "-");

public sealed class LruCache<TKey, TValue> where TKey : notnull
{
    private readonly int _capacity;
    private readonly Dictionary<TKey, LinkedListNode<(TKey Key, TValue Value)>> _map = new();
    private readonly LinkedList<(TKey Key, TValue Value)> _order = new();
    private readonly object _lock = new();

    public LruCache(int capacity)
    {
        if (capacity <= 0) throw new ArgumentOutOfRangeException(nameof(capacity));
        _capacity = capacity;
    }

    public bool TryGet(TKey key, out TValue value)
    {
        lock (_lock)
        {
            if (_map.TryGetValue(key, out var node))
            {
                _order.Remove(node);
                _order.AddFirst(node);
                value = node.Value.Value;
                return true;
            }
            value = default!;
            return false;
        }
    }

    public void Put(TKey key, TValue value)
    {
        lock (_lock)
        {
            if (_map.TryGetValue(key, out var node))
            {
                node.Value = (key, value);
                _order.Remove(node);
                _order.AddFirst(node);
                return;
            }
            if (_map.Count >= _capacity)
            {
                var last = _order.Last!;
                _order.RemoveLast();
                _map.Remove(last.Value.Key);
            }
            _map[key] = _order.AddFirst((key, value));
        }
    }
}
```
Output:
```
a
False
c
```
**Explanation / complexity**: All operations O(1). The linked list head = most recent, tail = least recent. A single lock keeps map and list consistent (even reads mutate order, so a reader-writer lock doesn't help).

**Follow-up variations**: Add TTL expiry; use `ConcurrentDictionary` + sampled eviction for scalability; `MemoryCache` in production; LFU cache.

### 2.10 [Medium] Thread-safe singleton

**Problem**: Provide a lazily created singleton that is guaranteed to be constructed once under contention.

**Hint**: Prefer `Lazy<T>` or a static readonly field over hand-written double-checked locking.

**Solution**
```csharp
Parallel.For(0, 100, i => { var inst = AppConfig.Instance; });
Console.WriteLine(AppConfig.Created);
Console.WriteLine(ReferenceEquals(AppConfig.Instance, AppConfig.Instance));

public sealed class AppConfig
{
    private static int _created;
    private static readonly Lazy<AppConfig> _lazy = new(() => new AppConfig());   // thread-safe by default
    public static AppConfig Instance => _lazy.Value;
    public static int Created => _created;
    private AppConfig() => Interlocked.Increment(ref _created);
}
```
Output:
```
1
True
```
**Explanation / complexity**: `Lazy<T>` default mode `ExecutionAndPublication` runs the factory once under a lock. Even simpler: `public static readonly AppConfig Instance = new();` – the CLR guarantees type-initialiser thread safety (lazy as long as no explicit static ctor... *almost* lazy with `beforefieldinit`). Classic double-checked locking needs a `volatile` field to be correct.

**Follow-up variations**: Why singletons hurt testability – prefer DI `AddSingleton`. Per-thread singleton (`ThreadLocal<T>`).

### 2.11 [Easy] Thread-safe counter

**Problem**: Increment a shared counter 1,000,000 times from many threads and get exactly 1,000,000.

**Hint**: `count++` is read-modify-write (not atomic).

**Solution**
```csharp
int safe = 0;
Parallel.For(0, 1_000_000, _ => { Interlocked.Increment(ref safe); });
Console.WriteLine(safe);

int locked = 0;
var gate = new object();
Parallel.For(0, 1_000_000, _ => { lock (gate) { locked++; } });
Console.WriteLine(locked);
```
Output:
```
1000000
1000000
```
**Explanation / complexity**: `Interlocked` uses a single atomic CPU instruction (fastest). `lock` is fine for compound operations. A plain `safe++` would typically print a smaller number due to lost updates.

**Follow-up variations**: `LongAdder`-style striped counters (`ThreadLocal` + sum) for heavy contention; `volatile` doesn't make `++` atomic.

### 2.12 [Hard] Producer–consumer with `Channel<T>`

**Problem**: One producer emits 1..10 into a bounded channel (capacity 3); two consumers sum the items. Apply back-pressure and shut down cleanly.

**Hint**: `Channel.CreateBounded`, `Writer.Complete()`, `Reader.ReadAllAsync()`.

**Solution**
```csharp
using System.Threading.Channels;

var ch = Channel.CreateBounded<int>(new BoundedChannelOptions(3)
{
    SingleWriter = true,
    FullMode = BoundedChannelFullMode.Wait      // back-pressure
});

var producer = Task.Run(async () =>
{
    for (int i = 1; i <= 10; i++) await ch.Writer.WriteAsync(i);
    ch.Writer.Complete();
});

long total = 0;
var consumers = Enumerable.Range(0, 2).Select(_ => Task.Run(async () =>
{
    await foreach (var x in ch.Reader.ReadAllAsync())
        Interlocked.Add(ref total, x);
})).ToArray();

await producer;
await Task.WhenAll(consumers);
Console.WriteLine(total);
```
Output: `55`

**Explanation / complexity**: `WriteAsync` awaits when the buffer is full (no unbounded memory growth). `Complete()` ends `ReadAllAsync` loops once the buffer is drained, so consumers exit naturally. Channels are async-friendly and allocation-light compared with `BlockingCollection<T>`.

**Follow-up variations**: Propagate failure with `Complete(exception)`. Cancellation tokens. `DropOldest` mode for telemetry.

### 2.13 [Hard] Token-bucket rate limiter

**Problem**: Allow bursts up to `capacity` and sustained `rate` per second. Make time injectable so it can be tested.

**Hint**: Refill lazily on each call: `tokens += elapsed * rate`, capped at capacity.

**Solution**
```csharp
var now = new DateTime(2024, 1, 1, 0, 0, 0, DateTimeKind.Utc);
var bucket = new TokenBucket(capacity: 3, ratePerSecond: 1, () => now);

Console.WriteLine(string.Join(" ", Enumerable.Range(0, 4).Select(_ => bucket.TryAcquire())));
now = now.AddSeconds(2);
Console.WriteLine(string.Join(" ", Enumerable.Range(0, 3).Select(_ => bucket.TryAcquire())));

public sealed class TokenBucket
{
    private readonly double _capacity, _rate;
    private readonly Func<DateTime> _now;
    private readonly object _lock = new();
    private double _tokens;
    private DateTime _last;

    public TokenBucket(double capacity, double ratePerSecond, Func<DateTime> now)
    {
        _capacity = capacity; _rate = ratePerSecond; _now = now;
        _tokens = capacity; _last = now();
    }

    public bool TryAcquire()
    {
        lock (_lock)
        {
            var n = _now();
            _tokens = Math.Min(_capacity, _tokens + (n - _last).TotalSeconds * _rate);
            _last = n;
            if (_tokens < 1) return false;
            _tokens -= 1;
            return true;
        }
    }
}
```
Output:
```
True True True False
True True False
```
**Explanation / complexity**: O(1) per call, O(1) memory per bucket. The first burst uses the 3 tokens; after 2 s, 2 tokens are refilled.

**Follow-up variations**: Per-key buckets in a `ConcurrentDictionary`; sliding-window log/counter algorithms; built-in `System.Threading.RateLimiting` (`TokenBucketRateLimiter`, `AddRateLimiter` in ASP.NET Core); distributed limiter with Redis (Lua script).

### 2.14 [Medium] Retry with exponential backoff and jitter

**Problem**: Retry an async operation up to N attempts, doubling the delay each time with random jitter, only for transient errors.

**Hint**: Use an exception filter (`when`) so the final failure propagates with its original stack trace.

**Solution**
```csharp
static async Task<T> RetryAsync<T>(
    Func<int, Task<T>> op,
    int maxAttempts = 5,
    TimeSpan? baseDelay = null,
    Func<Exception, bool>? isTransient = null,
    CancellationToken ct = default)
{
    var delay = baseDelay ?? TimeSpan.FromMilliseconds(200);
    for (int attempt = 1; ; attempt++)
    {
        try { return await op(attempt); }
        catch (Exception ex) when (attempt < maxAttempts
                                   && ex is not OperationCanceledException
                                   && (isTransient?.Invoke(ex) ?? true))
        {
            var backoff = TimeSpan.FromTicks(delay.Ticks * (1L << (attempt - 1)));
            var jitter = TimeSpan.FromMilliseconds(Random.Shared.NextDouble() * backoff.TotalMilliseconds * 0.2);
            await Task.Delay(backoff + jitter, ct);
        }
    }
}

var result = await RetryAsync(async attempt =>
{
    Console.WriteLine($"attempt {attempt}");
    await Task.Yield();
    if (attempt < 3) throw new HttpRequestException("transient");
    return "ok";
}, baseDelay: TimeSpan.FromMilliseconds(10));
Console.WriteLine(result);
```
Output:
```
attempt 1
attempt 2
attempt 3
ok
```
**Explanation / complexity**: Delays are ≈ 10, 20 ms (+ up to 20 % jitter). Jitter avoids thundering-herd synchronised retries. On the last attempt the filter is false, so the exception escapes unchanged.

**Follow-up variations**: Cap the max delay; honour `Retry-After`; circuit breaker; Polly / `Microsoft.Extensions.Resilience`. Idempotency of retried operations.

### 2.15 [Hard] Debounce

**Problem**: Many calls in quick succession should trigger the action only once, after a quiet period (e.g. search-as-you-type).

**Hint**: Each call cancels the previous pending delay.

**Solution**
```csharp
var fired = new List<int>();
var deb = new Debouncer(TimeSpan.FromMilliseconds(100));
for (int i = 1; i <= 5; i++)
{
    int v = i;
    deb.Debounce(() => { lock (fired) fired.Add(v); return Task.CompletedTask; });
    await Task.Delay(10);
}
await Task.Delay(400);
Console.WriteLine("fired: " + string.Join(",", fired));

public sealed class Debouncer
{
    private readonly TimeSpan _delay;
    private CancellationTokenSource? _cts;
    private readonly object _lock = new();

    public Debouncer(TimeSpan delay) => _delay = delay;

    public void Debounce(Func<Task> action)
    {
        CancellationToken token;
        lock (_lock)
        {
            _cts?.Cancel();
            _cts = new CancellationTokenSource();
            token = _cts.Token;
        }
        _ = RunAsync(action, token);
    }

    private async Task RunAsync(Func<Task> action, CancellationToken token)
    {
        try
        {
            await Task.Delay(_delay, token);
            await action();
        }
        catch (OperationCanceledException) { }
    }
}
```
Output: `fired: 5`

**Explanation / complexity**: Only the last call survives the 100 ms quiet window. O(1) per call. (Old CTS instances are cancelled and become garbage; dispose them in production code.)

**Follow-up variations**: Throttle (at most once per interval, leading/trailing edge); Rx `Throttle`/`Sample`; fire-and-forget exception handling.

### 2.16 [Hard] Simple DI container

**Problem**: Build a container supporting transient/singleton registrations, factory registration, and constructor injection (greediest constructor).

**Hint**: Reflection over `GetConstructors()`, recursive `Resolve` for each parameter type.

**Solution**
```csharp
using System.Collections.Concurrent;

var c = new Container()
    .Register<IClock, FixedClock>(Lifetime.Singleton)
    .Register<Greeter, Greeter>();

var g1 = c.Resolve<Greeter>();
var g2 = c.Resolve<Greeter>();
Console.WriteLine(g1.Greet("Ann"));
Console.WriteLine(ReferenceEquals(c.Resolve<IClock>(), c.Resolve<IClock>()));
Console.WriteLine(ReferenceEquals(g1, g2));

public interface IClock { string Now { get; } }
public sealed class FixedClock : IClock { public string Now => "12:00"; }
public sealed class Greeter(IClock clock)
{
    public string Greet(string name) => $"Hello {name} at {clock.Now}";
}

public enum Lifetime { Transient, Singleton }

public sealed class Container
{
    private sealed record Reg(Type Impl, Lifetime Life, Func<Container, object>? Factory);
    private readonly Dictionary<Type, Reg> _regs = new();
    private readonly ConcurrentDictionary<Type, object> _singletons = new();

    public Container Register<TService, TImpl>(Lifetime life = Lifetime.Transient) where TImpl : TService
    {
        _regs[typeof(TService)] = new Reg(typeof(TImpl), life, null);
        return this;
    }

    public Container Register<TService>(Func<Container, TService> factory, Lifetime life = Lifetime.Transient)
        where TService : class
    {
        _regs[typeof(TService)] = new Reg(typeof(TService), life, c => factory(c));
        return this;
    }

    public T Resolve<T>() => (T)Resolve(typeof(T));

    public object Resolve(Type type)
    {
        if (!_regs.TryGetValue(type, out var reg))
            throw new InvalidOperationException($"Not registered: {type.Name}");
        return reg.Life == Lifetime.Singleton
            ? _singletons.GetOrAdd(type, _ => Create(reg))
            : Create(reg);
    }

    private object Create(Reg reg)
    {
        if (reg.Factory is not null) return reg.Factory(this);
        var ctor = reg.Impl.GetConstructors().OrderByDescending(c => c.GetParameters().Length).First();
        var args = ctor.GetParameters().Select(p => Resolve(p.ParameterType)).ToArray();
        return ctor.Invoke(args);
    }
}
```
Output:
```
Hello Ann at 12:00
True
False
```
**Explanation / complexity**: Resolution is O(depth of graph) reflection calls. `GetOrAdd` may run the factory twice under a race (only one result is kept) – use `Lazy<object>` values for strict once-only creation.

**Follow-up variations**: Scoped lifetime (per-request child container); circular dependency detection; open generics; compile expression trees for speed; captive-dependency validation (singleton depending on scoped).

### 2.17 [Medium] Generic repository

**Problem**: Define `IRepository<T, TKey>` and an in-memory implementation.

**Hint**: Pass a key selector; use `ConcurrentDictionary` for safety.

**Solution**
```csharp
using System.Collections.Concurrent;

var repo = new InMemoryRepository<Product, int>(p => p.Id);
repo.Add(new Product(1, "Pen", 2.5m));
repo.Add(new Product(2, "Book", 12m));
repo.Add(new Product(3, "Lamp", 30m));

Console.WriteLine(string.Join(",", repo.Find(p => p.Price > 10).Select(p => p.Name)));
Console.WriteLine(repo.Get(2)?.Name);
Console.WriteLine(repo.Remove(1));
Console.WriteLine(repo.All().Count);

public record Product(int Id, string Name, decimal Price);

public interface IRepository<T, TKey> where T : class where TKey : notnull
{
    T? Get(TKey id);
    IEnumerable<T> Find(Func<T, bool> predicate);
    void Add(T entity);
    bool Remove(TKey id);
    IReadOnlyCollection<T> All();
}

public sealed class InMemoryRepository<T, TKey> : IRepository<T, TKey>
    where T : class where TKey : notnull
{
    private readonly ConcurrentDictionary<TKey, T> _store = new();
    private readonly Func<T, TKey> _key;
    public InMemoryRepository(Func<T, TKey> key) => _key = key;

    public T? Get(TKey id) => _store.TryGetValue(id, out var e) ? e : null;
    public IEnumerable<T> Find(Func<T, bool> predicate) => _store.Values.Where(predicate);
    public void Add(T entity)
    {
        if (!_store.TryAdd(_key(entity), entity))
            throw new InvalidOperationException("Duplicate key");
    }
    public bool Remove(TKey id) => _store.TryRemove(id, out _);
    public IReadOnlyCollection<T> All() => _store.Values.ToArray();
}
```
Output:
```
Book,Lamp
Book
True
2
```
(`Find` order follows `ConcurrentDictionary` enumeration, which is insertion order for a small unmodified dictionary but not guaranteed – sort if order matters.)

**Explanation / complexity**: O(1) Get/Add/Remove, O(n) Find. Real implementations take `Expression<Func<T,bool>>` for EF Core so filters translate to SQL.

**Follow-up variations**: Unit of Work; is the generic repository an anti-pattern over `DbContext`? Async API with `CancellationToken`; specification pattern.

### 2.18 [Medium] Custom LINQ operators

**Problem**: Implement `Batch`, `Pairwise`, and `TakeEvery` as lazy extension methods.

**Hint**: Validate arguments eagerly, then return an iterator (see puzzle 1.12).

**Solution**
```csharp
Console.WriteLine(string.Join(" ", Enumerable.Range(1, 7).Batch(3).Select(b => "[" + string.Join(",", b) + "]")));
Console.WriteLine(string.Join(" ", Enumerable.Range(1, 4).Pairwise()));
Console.WriteLine(string.Join(" ", Enumerable.Range(1, 10).TakeEvery(3)));

public static class MyLinq
{
    public static IEnumerable<T[]> Batch<T>(this IEnumerable<T> source, int size)
    {
        ArgumentNullException.ThrowIfNull(source);
        if (size <= 0) throw new ArgumentOutOfRangeException(nameof(size));
        return Iterate();

        IEnumerable<T[]> Iterate()
        {
            var buffer = new List<T>(size);
            foreach (var x in source)
            {
                buffer.Add(x);
                if (buffer.Count == size) { yield return buffer.ToArray(); buffer.Clear(); }
            }
            if (buffer.Count > 0) yield return buffer.ToArray();
        }
    }

    public static IEnumerable<(T Prev, T Curr)> Pairwise<T>(this IEnumerable<T> source)
    {
        ArgumentNullException.ThrowIfNull(source);
        return Iterate();

        IEnumerable<(T, T)> Iterate()
        {
            using var e = source.GetEnumerator();
            if (!e.MoveNext()) yield break;
            var prev = e.Current;
            while (e.MoveNext()) { yield return (prev, e.Current); prev = e.Current; }
        }
    }

    public static IEnumerable<T> TakeEvery<T>(this IEnumerable<T> source, int n)
    {
        ArgumentNullException.ThrowIfNull(source);
        if (n <= 0) throw new ArgumentOutOfRangeException(nameof(n));
        return source.Where((x, i) => i % n == 0);
    }
}
```
Output:
```
[1,2,3] [4,5,6] [7]
(1, 2) (2, 3) (3, 4)
1 4 7 10
```
**Explanation / complexity**: All are O(n) time, streaming, O(size)/O(1) memory. Built-ins since .NET 6: `Chunk`; `DistinctBy`, `MaxBy`, `Zip` with tuples.

**Follow-up variations**: `Scan` (running aggregate), `Window(k)`, `MinBy` without a built-in, async `IAsyncEnumerable` versions.

### 2.19 [Medium] Flatten nested lists

**Problem**: Flatten `{1, {2, {3, 4}}, 5}` into `1 2 3 4 5`.

**Hint**: Recursion with `yield return`; or an explicit stack of enumerators to avoid deep recursion.

**Solution**
```csharp
using System.Collections;

static IEnumerable<int> Flatten(IEnumerable items)
{
    foreach (var item in items)
    {
        if (item is int i) yield return i;
        else if (item is IEnumerable nested)
            foreach (var x in Flatten(nested)) yield return x;
    }
}

static IEnumerable<int> FlattenIterative(IEnumerable items)
{
    var stack = new Stack<IEnumerator>();
    stack.Push(items.GetEnumerator());
    while (stack.Count > 0)
    {
        var e = stack.Peek();
        if (!e.MoveNext()) { stack.Pop(); continue; }
        if (e.Current is int i) yield return i;
        else if (e.Current is IEnumerable nested) stack.Push(nested.GetEnumerator());
    }
}

var data = new object[] { 1, new object[] { 2, new object[] { 3, 4 } }, 5 };
Console.WriteLine(string.Join(" ", Flatten(data)));
Console.WriteLine(string.Join(" ", FlattenIterative(data)));
```
Output:
```
1 2 3 4 5
1 2 3 4 5
```
**Explanation / complexity**: O(n) elements visited. The recursive `yield` version costs O(depth) per element (nested iterators); the stack version is O(1) per element and safe for very deep nesting.

**Follow-up variations**: Generic `Flatten<T>` over `IEnumerable<IEnumerable<T>>` = `SelectMany`. Don't treat `string` as nested (it's `IEnumerable<char>`). Tree flatten (DFS/BFS).

### 2.20 [Medium] Merge intervals

**Problem**: Merge overlapping intervals: `[1,3],[2,6],[8,10],[15,18]`.

**Hint**: Sort by start; extend the last merged interval when the next one starts inside it.

**Solution**
```csharp
static List<(int Start, int End)> Merge(IEnumerable<(int Start, int End)> xs)
{
    var res = new List<(int Start, int End)>();
    foreach (var cur in xs.OrderBy(x => x.Start))
    {
        if (res.Count > 0 && cur.Start <= res[^1].End)
            res[^1] = (res[^1].Start, Math.Max(res[^1].End, cur.End));
        else
            res.Add(cur);
    }
    return res;
}
var merged = Merge(new[] { (1, 3), (2, 6), (8, 10), (15, 18) });
Console.WriteLine(string.Join(" ", merged.Select(i => $"[{i.Start},{i.End}]")));
Console.WriteLine(string.Join(" ", Merge(new[] { (1, 4), (4, 5) }).Select(i => $"[{i.Start},{i.End}]")));
```
Output:
```
[1,6] [8,10] [15,18]
[1,5]
```
**Explanation / complexity**: O(n log n) for the sort, O(n) merge. `<=` treats touching intervals as overlapping – state that decision explicitly.

**Follow-up variations**: Insert a new interval into a sorted list; find the minimum number of meeting rooms; free time slots.

### 2.21 [Easy] Binary search

**Problem**: Find the index of a target in a sorted array; also implement lower bound.

**Hint**: Avoid `(lo + hi) / 2` overflow; decide loop invariants (`<=` vs `<`).

**Solution**
```csharp
static int BinarySearch(int[] a, int t)
{
    int lo = 0, hi = a.Length - 1;
    while (lo <= hi)
    {
        int mid = lo + (hi - lo) / 2;
        if (a[mid] == t) return mid;
        if (a[mid] < t) lo = mid + 1; else hi = mid - 1;
    }
    return -1;
}
static int LowerBound(int[] a, int t)   // first index with a[i] >= t
{
    int lo = 0, hi = a.Length;
    while (lo < hi)
    {
        int mid = lo + (hi - lo) / 2;
        if (a[mid] < t) lo = mid + 1; else hi = mid;
    }
    return lo;
}
var arr = new[] { 1, 3, 5, 7, 9 };
Console.WriteLine(BinarySearch(arr, 7));
Console.WriteLine(BinarySearch(arr, 4));
Console.WriteLine(LowerBound(arr, 4));
Console.WriteLine(LowerBound(new[] { 1, 2, 2, 2, 3 }, 2));
```
Output:
```
3
-1
2
1
```
**Explanation / complexity**: O(log n) time, O(1) space. Built-in: `Array.BinarySearch` returns `~insertionIndex` when not found.

**Follow-up variations**: Search in a rotated sorted array; binary search on the answer (e.g. minimum capacity); first/last occurrence.

### 2.22 [Medium] Group words by anagram

**Problem**: `["eat","tea","tan","ate","nat","bat"]` → groups of anagrams.

**Hint**: Use the sorted letters as the grouping key.

**Solution**
```csharp
var words = new[] { "eat", "tea", "tan", "ate", "nat", "bat" };
var groups = words.GroupBy(w => new string(w.OrderBy(c => c).ToArray()));
foreach (var g in groups) Console.WriteLine(string.Join(",", g));
```
Output:
```
eat,tea,ate
tan,nat
bat
```
**Explanation / complexity**: O(n · k log k) for n words of length k. `GroupBy` returns groups in order of first appearance and keeps in-group order. For lowercase-only words use a 26-count key for O(n·k).

**Follow-up variations**: Unicode normalisation; return only groups with size > 1; streaming/very large dictionaries.

### 2.23 [Hard] Parse a CSV line (RFC 4180 style)

**Problem**: Parse `a,"b,c","d ""q"" e",,f` handling quoted commas, escaped quotes and empty fields.

**Hint**: A small state machine with an `inQuotes` flag.

**Solution**
```csharp
using System.Text;

static List<string> ParseCsvLine(string line)
{
    var fields = new List<string>();
    var sb = new StringBuilder();
    bool inQuotes = false;
    for (int i = 0; i < line.Length; i++)
    {
        char c = line[i];
        if (inQuotes)
        {
            if (c == '"')
            {
                if (i + 1 < line.Length && line[i + 1] == '"') { sb.Append('"'); i++; }
                else inQuotes = false;
            }
            else sb.Append(c);
        }
        else if (c == '"') inQuotes = true;
        else if (c == ',') { fields.Add(sb.ToString()); sb.Clear(); }
        else sb.Append(c);
    }
    fields.Add(sb.ToString());
    return fields;
}
var f = ParseCsvLine("a,\"b,c\",\"d \"\"q\"\" e\",,f");
Console.WriteLine(string.Join(" | ", f.Select(x => "[" + x + "]")));
```
Output: `[a] | [b,c] | [d "q" e] | [] | [f]`

**Explanation / complexity**: O(n) single pass. Real CSV also allows newlines inside quotes, so a production parser must work on a stream, not line by line.

**Follow-up variations**: Header mapping to a record; `Span<char>` zero-alloc parser; use CsvHelper / `TextFieldParser`; configurable delimiter.

### 2.24 [Hard] Bank account with concurrency

**Problem**: Thread-safe deposit/withdraw/transfer that never overdraws, never loses money, and cannot deadlock on A→B vs B→A transfers.

**Hint**: Lock both accounts in a consistent global order (by `Id`).

**Solution**
```csharp
var a = new Account(5000);
var b = new Account(5000);
Parallel.For(0, 10_000, i =>
{
    if (i % 2 == 0) Account.TryTransfer(a, b, 1);
    else Account.TryTransfer(b, a, 1);
});
Console.WriteLine(a.Balance + b.Balance);
Console.WriteLine(a.TryWithdraw(1_000_000));

public sealed class Account
{
    private static int _nextId;
    private readonly object _lock = new();
    private decimal _balance;
    public int Id { get; } = Interlocked.Increment(ref _nextId);

    public Account(decimal initial) => _balance = initial;

    public decimal Balance { get { lock (_lock) return _balance; } }

    public void Deposit(decimal amount)
    {
        if (amount <= 0) throw new ArgumentOutOfRangeException(nameof(amount));
        lock (_lock) _balance += amount;
    }

    public bool TryWithdraw(decimal amount)
    {
        if (amount <= 0) throw new ArgumentOutOfRangeException(nameof(amount));
        lock (_lock)
        {
            if (_balance < amount) return false;
            _balance -= amount;
            return true;
        }
    }

    public static bool TryTransfer(Account from, Account to, decimal amount)
    {
        if (from == to) throw new ArgumentException("Same account");
        var first = from.Id < to.Id ? from : to;
        var second = ReferenceEquals(first, from) ? to : from;
        lock (first._lock)
        lock (second._lock)
        {
            if (from._balance < amount) return false;
            from._balance -= amount;
            to._balance += amount;
            return true;
        }
    }
}
```
Output:
```
10000
False
```
**Explanation / complexity**: Ordered lock acquisition removes the circular-wait condition (deadlock). Money is `decimal`, never `double`. Check-then-act (`balance >= amount` then subtract) must be inside the same lock.

**Follow-up variations**: Lock-free with `Interlocked.CompareExchange` on a `long` of cents; optimistic concurrency with a version column in a database; idempotent transfers/outbox.

### 2.25 [Medium] Event aggregator

**Problem**: Decouple publishers from subscribers: `Subscribe<T>(handler)` returns an `IDisposable`; `Publish<T>(message)` invokes handlers of that type.

**Hint**: `Dictionary<Type, List<Delegate>>` guarded by a lock; copy a snapshot before invoking.

**Solution**
```csharp
var bus = new EventAggregator();
var sub1 = bus.Subscribe<UserCreated>(e => Console.WriteLine("Welcome " + e.Name));
bus.Subscribe<UserCreated>(e => Console.WriteLine("Audit: " + e.Name));
bus.Publish(new UserCreated("Ann"));
sub1.Dispose();
bus.Publish(new UserCreated("Bob"));

public record UserCreated(string Name);

public sealed class EventAggregator
{
    private readonly Dictionary<Type, List<Delegate>> _handlers = new();
    private readonly object _lock = new();

    public IDisposable Subscribe<T>(Action<T> handler)
    {
        lock (_lock)
        {
            if (!_handlers.TryGetValue(typeof(T), out var list))
                _handlers[typeof(T)] = list = new List<Delegate>();
            list.Add(handler);
        }
        return new Unsubscriber(() => { lock (_lock) _handlers[typeof(T)].Remove(handler); });
    }

    public void Publish<T>(T message)
    {
        Delegate[] snapshot;
        lock (_lock)
            snapshot = _handlers.TryGetValue(typeof(T), out var list) ? list.ToArray() : Array.Empty<Delegate>();
        foreach (var h in snapshot) ((Action<T>)h)(message);
    }

    private sealed class Unsubscriber(Action onDispose) : IDisposable
    {
        public void Dispose() => onDispose();
    }
}
```
Output:
```
Welcome Ann
Audit: Ann
Audit: Bob
```
**Explanation / complexity**: Publish is O(h) for h handlers. Handlers are invoked *outside* the lock (avoids deadlocks/re-entrancy issues, allows handlers to subscribe/unsubscribe).

**Follow-up variations**: Weak references; async handlers (`Func<T, Task>`); handler exceptions isolation; publish to base types/interfaces.

### 2.26 [Medium] Memoize a function

**Problem**: Wrap a pure function so each argument is computed once (even under concurrency). Use it for Fibonacci.

**Hint**: `ConcurrentDictionary<TArg, Lazy<TResult>>` so the factory runs at most once per key.

**Solution**
```csharp
using System.Collections.Concurrent;

static Func<TArg, TResult> Memoize<TArg, TResult>(Func<TArg, TResult> f) where TArg : notnull
{
    var cache = new ConcurrentDictionary<TArg, Lazy<TResult>>();
    return arg => cache.GetOrAdd(arg, a => new Lazy<TResult>(() => f(a))).Value;
}

int calls = 0;
Func<int, long>? fib = null;
fib = Memoize<int, long>(n =>
{
    Interlocked.Increment(ref calls);
    return n < 2 ? n : fib!(n - 1) + fib!(n - 2);
});
Console.WriteLine(fib(50));
Console.WriteLine(calls);
```
Output:
```
12586269025
51
```
**Explanation / complexity**: Naive recursion is O(φⁿ); memoised is O(n) time/space – 51 computations for n = 0..50. `GetOrAdd` alone can run the factory twice on a race; wrapping in `Lazy<T>` makes evaluation once-only. The recursive call happens inside `.Value`, outside any dictionary lock, so recursion is safe.

**Follow-up variations**: Cache eviction / TTL; async memoize (`Lazy<Task<T>>`); multi-argument via tuple keys; bottom-up DP with O(1) space.

### 2.27 [Hard] Async throttling with `SemaphoreSlim`

**Problem**: Process 10 items with an async operation but at most 3 at a time; return results in input order.

**Hint**: `await sem.WaitAsync()` before the work, `Release()` in `finally`.

**Solution**
```csharp
static class Program
{
    static async Task<IReadOnlyList<TResult>> ForEachThrottledAsync<T, TResult>(
        IEnumerable<T> items, int maxDegree, Func<T, Task<TResult>> work)
    {
        using var sem = new SemaphoreSlim(maxDegree);
        var tasks = items.Select(async item =>
        {
            await sem.WaitAsync();
            try { return await work(item); }
            finally { sem.Release(); }
        }).ToList();
        return await Task.WhenAll(tasks);
    }

    static async Task Main()
    {
        int current = 0, max = 0;
        var gate = new object();
        var results = await ForEachThrottledAsync(Enumerable.Range(1, 10), 3, async x =>
        {
            int now = Interlocked.Increment(ref current);
            lock (gate) max = Math.Max(max, now);
            await Task.Delay(50);
            Interlocked.Decrement(ref current);
            return x * x;
        });
        Console.WriteLine(results.Sum());
        Console.WriteLine(max <= 3);
    }
}
```
Output:
```
385
True
```
**Explanation / complexity**: All 10 tasks start, but only 3 pass the semaphore at a time; `Task.WhenAll` preserves input order. Memory O(n) for the task list. In .NET 6+ prefer `Parallel.ForEachAsync(items, new ParallelOptions { MaxDegreeOfParallelism = 3 }, ...)`.

**Follow-up variations**: Cancellation (`WaitAsync(ct)`); fail-fast on first error; throttle by rate rather than concurrency; per-host limits in `HttpClient` (`SocketsHttpHandler.MaxConnectionsPerServer`).

### 2.28 [Hard] Simple topic-based pub/sub

**Problem**: Build an in-memory broker with topics, async subscribers, per-subscriber ordering, and no slow-subscriber blocking the publisher.

**Hint**: Give each subscriber its own unbounded `Channel<T>` and a reader loop.

**Solution**
```csharp
using System.Collections.Concurrent;
using System.Threading.Channels;

var s1 = new List<string>(); var s2 = new List<string>(); var s3 = new List<string>();
await using (var broker = new PubSub())
{
    broker.Subscribe("news", async m => { await Task.Delay(20); s1.Add(m); });
    broker.Subscribe("news", m => { s2.Add(m); return Task.CompletedTask; });
    broker.Subscribe("sports", m => { s3.Add(m); return Task.CompletedTask; });
    broker.Publish("news", "a");
    broker.Publish("news", "b");
    broker.Publish("sports", "x");
}   // DisposeAsync drains queues
Console.WriteLine("s1: " + string.Join(",", s1));
Console.WriteLine("s2: " + string.Join(",", s2));
Console.WriteLine("s3: " + string.Join(",", s3));

public sealed class PubSub : IAsyncDisposable
{
    private sealed class Sub
    {
        public readonly Channel<string> Queue = Channel.CreateUnbounded<string>();
        public Task Loop = Task.CompletedTask;
    }
    private sealed class Unsubscriber(Action a) : IDisposable { public void Dispose() => a(); }

    private readonly ConcurrentDictionary<string, ConcurrentDictionary<Guid, Sub>> _topics = new();

    public IDisposable Subscribe(string topic, Func<string, Task> handler)
    {
        var sub = new Sub();
        var id = Guid.NewGuid();
        sub.Loop = Task.Run(async () =>
        {
            await foreach (var m in sub.Queue.Reader.ReadAllAsync())
                await handler(m);
        });
        _topics.GetOrAdd(topic, _ => new())[id] = sub;
        return new Unsubscriber(() =>
        {
            if (_topics.TryGetValue(topic, out var d) && d.TryRemove(id, out var s))
                s.Queue.Writer.TryComplete();
        });
    }

    public void Publish(string topic, string message)
    {
        if (_topics.TryGetValue(topic, out var subs))
            foreach (var s in subs.Values) s.Queue.Writer.TryWrite(message);
    }

    public async ValueTask DisposeAsync()
    {
        var all = _topics.Values.SelectMany(d => d.Values).ToList();
        foreach (var s in all) s.Queue.Writer.TryComplete();
        await Task.WhenAll(all.Select(s => s.Loop));
    }
}
```
Output:
```
s1: a,b
s2: a,b
s3: x
```
**Explanation / complexity**: Publish is O(subscribers) and non-blocking (`TryWrite` on an unbounded channel). Each subscriber processes messages sequentially, so ordering holds per subscriber. Completing the writer lets the loop drain and exit. A throwing handler would fault that subscriber's loop – wrap handler calls in try/catch in production.

**Follow-up variations**: Bounded channels with back-pressure; wildcard topics (`news.*`); at-least-once delivery with acks; replace with Redis/Service Bus/RabbitMQ.

---

# Part 3 – LINQ Challenges

> **Shared dataset.** Put the block below in `Data.cs` (global namespace) of a .NET 8 console project and start every solution's `Program.cs` with `using static Data;`. All outputs were produced from this exact data.

```csharp
public record Employee(int Id, string Name, int DeptId, decimal Salary, int? ManagerId);
public record Department(int Id, string Name);
public record Customer(int Id, string Name, string City);
public record Order(int Id, int CustomerId, DateTime Date, decimal Amount);
public record OrderLine(int OrderId, string Product, int Qty, decimal Price);

public static class Data
{
    public static readonly List<Department> Departments = new()
    {
        new(1, "Engineering"), new(2, "Sales"), new(3, "HR"), new(4, "Legal")
    };

    public static readonly List<Employee> Employees = new()
    {
        new(1, "Alice", 1, 120000m, null),
        new(2, "Bob",   1, 100000m, 1),
        new(3, "Carol", 1, 100000m, 1),
        new(4, "Dave",  2,  80000m, 1),
        new(5, "Eve",   2,  90000m, 4),
        new(6, "Frank", 3,  60000m, 1),
    };

    public static readonly List<Customer> Customers = new()
    {
        new(1, "Ann", "Pune"), new(2, "Raj", "Delhi"), new(3, "Mia", "Pune"), new(4, "Zed", "Mumbai")
    };

    public static readonly List<Order> Orders = new()
    {
        new(1, 1, new DateTime(2024, 1, 5),  100m),
        new(2, 1, new DateTime(2024, 1, 20), 250m),
        new(3, 2, new DateTime(2024, 2, 3),   75m),
        new(4, 3, new DateTime(2024, 2, 14), 300m),
        new(5, 1, new DateTime(2024, 3, 1),   50m),
        new(6, 2, new DateTime(2024, 3, 9),  125m),
        new(7, 3, new DateTime(2024, 3, 15), 200m),
    };

    public static readonly List<OrderLine> OrderLines = new()
    {
        new(1, "Pen", 10, 2.5m), new(1, "Book", 1, 75m), new(2, "Lamp", 5, 50m),
        new(3, "Pen", 30, 2.5m), new(4, "Lamp", 6, 50m), new(5, "Pen", 20, 2.5m), new(6, "Book", 1, 125m),
    };
}
```

### 3.1 [Easy] Group and aggregate

**Problem**: For each department show the employee count and average salary (rounded), ordered by department name.

**Hint**: `GroupBy(DeptId)` then `Join` to get the department name.

**Solution**
```csharp
using static Data;

var stats = Employees
    .GroupBy(e => e.DeptId)
    .Join(Departments, g => g.Key, d => d.Id,
          (g, d) => new { d.Name, Count = g.Count(), Avg = Math.Round(g.Average(e => e.Salary)) })
    .OrderBy(x => x.Name);
foreach (var s in stats) Console.WriteLine($"{s.Name}: {s.Count} employees, avg {s.Avg}");
```
Output:
```
Engineering: 3 employees, avg 106667
HR: 1 employees, avg 60000
Sales: 2 employees, avg 85000
```
**Explanation / complexity**: `GroupBy` is O(n) (hash based); `Join` builds a hash lookup on the inner side → O(n + m). Legal has no employees, so the inner join drops it.

**Follow-up variations**: Include departments with zero employees (see 3.4). Only groups with `Count() > 1`. Max/min salary per department with `MaxBy`.

### 3.2 [Easy] Inner join (query and method syntax)

**Problem**: List each employee with their department name.

**Hint**: `join … on … equals …`.

**Solution**
```csharp
using static Data;

var joined = from e in Employees
             join d in Departments on e.DeptId equals d.Id
             select $"{e.Name}-{d.Name}";
Console.WriteLine(string.Join(", ", joined));
```
Output: `Alice-Engineering, Bob-Engineering, Carol-Engineering, Dave-Sales, Eve-Sales, Frank-HR`

**Explanation / complexity**: LINQ-to-Objects `Join` hashes the *inner* sequence (`Departments`), then streams the outer – O(n + m). The result preserves outer order.

**Follow-up variations**: Join on a composite key (`new { a.X, a.Y } equals new { b.X, b.Y }`). Why `Join` beats nested `Where` (O(n·m)).

### 3.3 [Medium] Left join

**Problem**: Show every department and each of its employees; departments without employees appear once with `(none)`.

**Hint**: `join … into g` + `g.DefaultIfEmpty()`.

**Solution**
```csharp
using static Data;

var left = from d in Departments
           join e in Employees on d.Id equals e.DeptId into g
           from e in g.DefaultIfEmpty()
           select $"{d.Name}: {e?.Name ?? "(none)"}";
foreach (var l in left) Console.WriteLine(l);
```
Output:
```
Engineering: Alice
Engineering: Bob
Engineering: Carol
Sales: Dave
Sales: Eve
HR: Frank
Legal: (none)
```
**Explanation / complexity**: `join … into` is a *group join*; `DefaultIfEmpty()` yields a single `null` for empty groups, which converts it into a left outer join. O(n + m).

**Follow-up variations**: Right join = swap the sequences. Full outer join = left ∪ right (`Union`) of both. EF Core translates this to `LEFT JOIN`.

### 3.4 [Easy] Group join – counts including zero

**Problem**: Employee count per department, including empty ones.

**Hint**: `GroupJoin` already keeps outer rows with empty inner groups.

**Solution**
```csharp
using static Data;

var counts = Departments.GroupJoin(Employees, d => d.Id, e => e.DeptId,
    (d, es) => $"{d.Name}={es.Count()}");
Console.WriteLine(string.Join(", ", counts));
```
Output: `Engineering=3, Sales=2, HR=1, Legal=0`

**Explanation / complexity**: O(n + m). Unlike 3.3, results are one row per department (hierarchical shape).

**Follow-up variations**: Order by count desc then name; departments with no employees only (`!es.Any()`).

### 3.5 [Medium] Running total

**Problem**: Order orders by date and compute the cumulative amount.

**Hint**: A captured accumulator inside `Select`, *materialised immediately*; or `Aggregate` for a side-effect-free version.

**Solution**
```csharp
using static Data;

decimal run = 0;
var rt = Orders.OrderBy(o => o.Date)
    .Select(o => new { o.Id, o.Amount, Running = run += o.Amount })
    .ToList();                                    // materialise: the query has side effects
Console.WriteLine(string.Join(" ", rt.Select(x => $"{x.Id}:{x.Running}")));
```
Output: `1:100 2:350 3:425 4:725 5:775 6:900 7:1100`

**Explanation / complexity**: O(n log n) for the sort + O(n). Because the lambda mutates `run`, enumerating the query twice would give wrong totals – hence `ToList()`. A pure alternative: `Aggregate` building a `List<(int, decimal)>`, or a custom `Scan` operator. (EF Core cannot translate this; use raw SQL window functions, see Part 4.)

**Follow-up variations**: Running total per customer (group first). Moving average (3.18). Running total with `Zip`.

### 3.6 [Medium] Top-N per group

**Problem**: The two highest-paid employees in each department (ties broken by name).

**Hint**: `GroupBy` then `OrderByDescending(...).ThenBy(...).Take(n)` inside each group.

**Solution**
```csharp
using static Data;

var deptNames = Departments.ToDictionary(d => d.Id, d => d.Name);
foreach (var g in Employees.GroupBy(e => e.DeptId))
{
    var top = g.OrderByDescending(e => e.Salary).ThenBy(e => e.Name).Take(2).Select(e => e.Name);
    Console.WriteLine($"{deptNames[g.Key]}: {string.Join(", ", top)}");
}
```
Output:
```
Engineering: Alice, Bob
Sales: Eve, Dave
HR: Frank
```
**Explanation / complexity**: O(n log n) overall (sorting within groups). Use `.SelectMany(...)` to flatten to one sequence. Bob and Carol tie at 100 000 – `ThenBy(Name)` makes the result deterministic.

**Follow-up variations**: Include ties (dense-rank approach, see 3.15). EF Core translates this to `ROW_NUMBER()` in newer versions.

### 3.7 [Medium] Pivot (rows → columns)

**Problem**: Total order amount per customer per month (Jan–Mar) as a table.

**Hint**: Pick the column values up-front, then compute one cell per (row, column).

**Solution**
```csharp
using static Data;

var months = new[] { 1, 2, 3 };
Console.WriteLine("Name  Jan  Feb  Mar");
foreach (var c in Customers)
{
    var row = months.Select(m => Orders.Where(o => o.CustomerId == c.Id && o.Date.Month == m).Sum(o => o.Amount));
    Console.WriteLine($"{c.Name,-4} " + string.Join(" ", row.Select(x => x.ToString().PadLeft(4))));
}
```
Output:
```
Name  Jan  Feb  Mar
Ann   350    0   50
Raj     0   75  125
Mia     0  300  200
Zed     0    0    0
```
**Explanation / complexity**: O(customers × months × orders) as written – fine for small data. For large data, group once: `Orders.ToLookup(o => (o.CustomerId, o.Date.Month))` then look up cells in O(1).

**Follow-up variations**: Dynamic columns from the data (`Select(o => o.Date.Month).Distinct().Order()`); totals row/column; SQL `PIVOT` (Part 4).

### 3.8 [Easy] Aggregate over flattened child collections

**Problem**: Total quantity sold per product, highest first.

**Hint**: Group the `OrderLine`s directly by product.

**Solution**
```csharp
using static Data;

var byProduct = OrderLines.GroupBy(l => l.Product)
    .Select(g => new { Product = g.Key, Qty = g.Sum(l => l.Qty) })
    .OrderByDescending(x => x.Qty);
Console.WriteLine(string.Join(", ", byProduct.Select(x => $"{x.Product} {x.Qty}")));
```
Output: `Pen 60, Lamp 11, Book 2`

**Explanation / complexity**: O(n log k) for k products. If lines were nested in `Order.Lines`, you would first flatten with `orders.SelectMany(o => o.Lines)`.

**Follow-up variations**: `SelectMany` with a result selector to keep the parent (`SelectMany(o => o.Lines, (o, l) => new { o.Id, l.Product })`); revenue per product (`Qty * Price`).

### 3.9 [Medium] `DistinctBy` / `MaxBy` – first and biggest order per customer

**Problem**: Each customer's first order, and each customer's largest order.

**Hint**: Sort then `DistinctBy` keeps the first element of each key; `MaxBy` per group.

**Solution**
```csharp
using static Data;

var first = Orders.OrderBy(o => o.Date).DistinctBy(o => o.CustomerId);
Console.WriteLine(string.Join(" ", first.Select(o => $"cust{o.CustomerId}->order{o.Id}")));
var biggest = Orders.GroupBy(o => o.CustomerId).Select(g => g.MaxBy(o => o.Amount)!);
Console.WriteLine(string.Join(" ", biggest.Select(o => $"cust{o.CustomerId}->order{o.Id}")));
```
Output:
```
cust1->order1 cust2->order3 cust3->order4
cust1->order2 cust2->order6 cust3->order4
```
**Explanation / complexity**: `DistinctBy` (.NET 6+) is O(n) after the O(n log n) sort and relies on sort stability (`OrderBy` is stable). Before .NET 6: `GroupBy(k).Select(g => g.First())`.

**Follow-up variations**: Distinct by a composite key; custom `IEqualityComparer<T>`; latest order per customer (`OrderByDescending` first).

### 3.10 [Medium] Self-join – employees earning more than their manager

**Problem**: Find employees whose salary exceeds their manager's.

**Hint**: Join `Employees` with itself on `ManagerId = Id`.

**Solution**
```csharp
using static Data;

var rich = from e in Employees
           join m in Employees on e.ManagerId equals m.Id
           where e.Salary > m.Salary
           select $"{e.Name} > {m.Name}";
Console.WriteLine(string.Join(", ", rich));
```
Output: `Eve > Dave`

**Explanation / complexity**: O(n). Alice (no manager, `ManagerId = null`) never matches, as with SQL's `NULL` join semantics.

**Follow-up variations**: Include top-level employees using a left join; employees whose salary is above their department average.

### 3.11 [Medium] Group by composite key with a "having" filter

**Problem**: Customers who placed more than one order in the same month.

**Hint**: Anonymous types have structural equality, so they work as composite keys.

**Solution**
```csharp
using static Data;

var multi = Orders.GroupBy(o => new { o.CustomerId, o.Date.Month })
    .Where(g => g.Count() > 1)
    .Select(g => $"customer {g.Key.CustomerId}, month {g.Key.Month}: {g.Count()} orders");
foreach (var m in multi) Console.WriteLine(m);
```
Output: `customer 1, month 1: 2 orders`

**Explanation / complexity**: O(n). `Where` after `GroupBy` = SQL `HAVING`. Include the year in the key for real data.

**Follow-up variations**: Use a `(int, int)` tuple key; months with the maximum order count.

### 3.12 [Easy] Anti-join – customers without orders

**Problem**: List customers who never ordered.

**Hint**: `Any` inside `Where` (O(n·m)) or `ExceptBy` with a key set (O(n + m)).

**Solution**
```csharp
using static Data;

Console.WriteLine(string.Join(", ", Customers.Where(c => !Orders.Any(o => o.CustomerId == c.Id)).Select(c => c.Name)));
Console.WriteLine(string.Join(", ", Customers.ExceptBy(Orders.Select(o => o.CustomerId), c => c.Id).Select(c => c.Name)));
```
Output:
```
Zed
Zed
```
**Explanation / complexity**: The first form is O(n·m) in LINQ-to-Objects (EF translates it to `NOT EXISTS`). `ExceptBy` builds a hash set of the keys once.

**Follow-up variations**: Customers who ordered in Jan but not in Feb (set operations on key sets).

### 3.13 [Medium] Month-over-month change with `Zip`

**Problem**: Compute monthly totals, then the change from the previous month.

**Hint**: Zip a sequence with itself skipped by one.

**Solution**
```csharp
using static Data;

var monthly = Orders.GroupBy(o => o.Date.Month).OrderBy(g => g.Key)
    .Select(g => (Month: g.Key, Total: g.Sum(o => o.Amount))).ToList();
var changes = monthly.Zip(monthly.Skip(1), (a, b) => $"{b.Month}: {b.Total - a.Total:+0;-0;0}");
Console.WriteLine(string.Join(", ", changes));
```
Output: `2: +25, 3: 0`

**Explanation / complexity**: Totals are Jan 350, Feb 375, Mar 375. O(n). The custom format `+0;-0;0` prints a sign for positive, negative, and zero values respectively.

**Follow-up variations**: Percentage growth; `Pairwise` operator from Part 2 (2.18); handle missing months by joining to a calendar.

### 3.14 [Medium] Median

**Problem**: Median salary of all employees.

**Hint**: Sort; average the two middle values when the count is even.

**Solution**
```csharp
using static Data;

var sal = Employees.Select(e => e.Salary).OrderBy(s => s).ToArray();
var median = sal.Length % 2 == 1
    ? sal[sal.Length / 2]
    : (sal[sal.Length / 2 - 1] + sal[sal.Length / 2]) / 2;
Console.WriteLine(median);
```
Output: `95000`

**Explanation / complexity**: Sorted salaries 60k, 80k, 90k, 100k, 100k, 120k → (90k + 100k)/2. O(n log n); quickselect gives O(n) average.

**Follow-up variations**: Median per department; streaming median with two heaps (`PriorityQueue<T,TPriority>`).

### 3.15 [Hard] Dense rank with ties

**Problem**: Rank employees by salary (highest = 1); equal salaries share the rank and ranks have no gaps.

**Hint**: Group by salary, order the groups, and use the *group index* as the rank.

**Solution**
```csharp
using static Data;

var ranked = Employees.GroupBy(e => e.Salary).OrderByDescending(g => g.Key)
    .SelectMany((g, i) => g.OrderBy(e => e.Name).Select(e => (e.Name, Rank: i + 1)));
Console.WriteLine(string.Join(", ", ranked.Select(r => $"{r.Name} {r.Rank}")));
```
Output: `Alice 1, Bob 2, Carol 2, Eve 3, Dave 4, Frank 5`

**Explanation / complexity**: O(n log n). This is `DENSE_RANK()`. For standard `RANK()` (gaps after ties) track a running count of items before each group.

**Follow-up variations**: Top-3 *distinct* salaries (Nth highest) = `Select(Salary).Distinct().OrderByDescending().Skip(2).FirstOrDefault()`.

### 3.16 [Hard] Hierarchy traversal (org chart)

**Problem**: Print the management tree with indentation.

**Hint**: Recursive local function: children of `managerId`, depth-first.

**Solution**
```csharp
using static Data;

void Print(int? managerId, int depth)
{
    foreach (var e in Employees.Where(e => e.ManagerId == managerId).OrderBy(e => e.Id))
    {
        Console.WriteLine(new string(' ', depth * 2) + e.Name);
        Print(e.Id, depth + 1);
    }
}
Print(null, 0);
```
Output:
```
Alice
  Bob
  Carol
  Dave
    Eve
  Frank
```
**Explanation / complexity**: As written O(n · depth-levels) because each call rescans the list; build `ToLookup(e => e.ManagerId)` first for O(n) total. Guard against cycles with a visited set.

**Follow-up variations**: Compute each employee's depth or full reporting chain; SQL recursive CTE (Part 4).

### 3.17 [Easy] `Chunk` and `ToLookup`

**Problem**: Sum orders in pages of 3; build a lookup of orders by customer and query a missing key.

**Hint**: `Chunk(size)` (.NET 6) and `ToLookup` returns an empty group for missing keys.

**Solution**
```csharp
using static Data;

Console.WriteLine(string.Join(" ", Orders.Chunk(3).Select(p => p.Sum(o => o.Amount))));
var lookup = Orders.ToLookup(o => o.CustomerId);
Console.WriteLine(lookup[1].Count() + " " + lookup[99].Count());
```
Output:
```
425 475 200
3 0
```
**Explanation / complexity**: O(n). Unlike `Dictionary`, a `Lookup` never throws for a missing key and allows many values per key; it is immutable and evaluated eagerly.

**Follow-up variations**: Paging with `Skip((page-1)*size).Take(size)`; `GroupBy` (lazy) vs `ToLookup` (eager).

### 3.18 [Medium] Moving average (sliding window)

**Problem**: 3-order moving average of amounts ordered by date, rounded to 1 decimal.

**Hint**: `Enumerable.Range` over window start indexes + `Skip/Take`.

**Solution**
```csharp
using static Data;

var amounts = Orders.OrderBy(o => o.Date).Select(o => o.Amount).ToArray();
var ma = Enumerable.Range(0, amounts.Length - 2)
    .Select(i => Math.Round(amounts.Skip(i).Take(3).Average(), 1));
Console.WriteLine(string.Join(" ", ma));
```
Output: `141.7 208.3 141.7 158.3 125`

**Explanation / complexity**: O(n·w) for window size w. A running sum (add the new item, subtract the one leaving) makes it O(n). `Skip` on an array is O(1) in modern .NET.

**Follow-up variations**: Handle fewer than 3 items; exponential moving average; `Window` custom operator.

---

# Part 4 – SQL Challenges

> Dialect: **T-SQL (SQL Server)**. Every query and result below was executed on SQL Server 2025 (LocalDB) against exactly the data shown; `GENERATE_SERIES` needs SQL Server 2022+ (compat level 160). Portable alternatives are noted (PostgreSQL/MySQL differences in the follow-ups). Decimal columns are displayed trimmed (`0.00` shown as `0.00`).

## Sample schema and data (used by all challenges)

```sql
CREATE TABLE Departments (
    DeptId   INT          NOT NULL PRIMARY KEY,
    DeptName VARCHAR(50)  NOT NULL
);
CREATE TABLE Employees (
    EmpId     INT           NOT NULL PRIMARY KEY,
    Name      VARCHAR(50)   NOT NULL,
    DeptId    INT           NOT NULL REFERENCES Departments(DeptId),
    Salary    DECIMAL(10,2) NOT NULL,
    ManagerId INT           NULL REFERENCES Employees(EmpId),
    HireDate  DATE          NOT NULL
);
CREATE TABLE Customers (
    CustomerId INT         NOT NULL PRIMARY KEY,
    Name       VARCHAR(50) NOT NULL,
    City       VARCHAR(50) NOT NULL,
    SignupDate DATE        NOT NULL
);
CREATE TABLE Products (
    ProductId   INT           NOT NULL PRIMARY KEY,
    ProductName VARCHAR(50)   NOT NULL,
    Category    VARCHAR(50)   NOT NULL,
    Price       DECIMAL(10,2) NOT NULL
);
CREATE TABLE Orders (
    OrderId     INT           NOT NULL PRIMARY KEY,
    CustomerId  INT           NOT NULL REFERENCES Customers(CustomerId),
    OrderDate   DATE          NOT NULL,
    TotalAmount DECIMAL(10,2) NOT NULL
);
CREATE TABLE OrderItems (
    OrderId   INT           NOT NULL REFERENCES Orders(OrderId),
    ProductId INT           NOT NULL REFERENCES Products(ProductId),
    Qty       INT           NOT NULL,
    UnitPrice DECIMAL(10,2) NOT NULL,
    PRIMARY KEY (OrderId, ProductId)
);
-- helper tables for specific puzzles
CREATE TABLE Contacts (ContactId INT NOT NULL PRIMARY KEY, Email VARCHAR(100) NOT NULL);
CREATE TABLE Tickets  (TicketId  INT NOT NULL PRIMARY KEY);
CREATE TABLE Logins   (CustomerId INT NOT NULL REFERENCES Customers(CustomerId), LoginDate DATE NOT NULL,
                       PRIMARY KEY (CustomerId, LoginDate));
CREATE TABLE Events   (EventId INT IDENTITY PRIMARY KEY, UserId INT NOT NULL, EventTime DATETIME2(0) NOT NULL);

INSERT INTO Departments VALUES (1,'Engineering'),(2,'Sales'),(3,'HR'),(4,'Legal');

INSERT INTO Employees VALUES
 (1,'Alice',1,120000,NULL,'2018-03-01'),
 (2,'Bob',  1,100000,1,   '2019-06-15'),
 (3,'Carol',1,100000,1,   '2020-01-10'),
 (4,'Dave', 2, 80000,1,   '2019-09-01'),
 (5,'Eve',  2, 90000,4,   '2021-02-20'),
 (6,'Frank',3, 60000,1,   '2022-07-05'),
 (7,'Grace',1, 90000,2,   '2023-01-09');

INSERT INTO Customers VALUES
 (1,'Ann','Pune','2022-01-10'),(2,'Raj','Delhi','2022-01-25'),(3,'Mia','Pune','2022-02-05'),
 (4,'Zed','Mumbai','2023-03-12'),(5,'Lee','Delhi','2023-03-20');

INSERT INTO Products VALUES
 (1,'Pen','Stationery',2.50),(2,'Notebook','Stationery',5.00),(3,'Lamp','Home',50.00),
 (4,'Chair','Home',120.00),(5,'Desk','Home',300.00),(6,'Mug','Home',8.00);

INSERT INTO Orders VALUES
 (1, 1,'2023-01-12',100),(2, 1,'2023-01-20',250),(3, 2,'2023-02-03', 75),(4, 3,'2023-02-14',300),
 (5, 1,'2023-03-01', 50),(6, 2,'2023-03-09',125),(7, 3,'2023-03-15',200),
 (8, 1,'2024-01-12',400),(9, 3,'2024-02-02',150),(10,2,'2024-02-20',100),
 (11,1,'2022-03-05',200),(12,2,'2022-06-10',300);

INSERT INTO OrderItems VALUES
 (1,3,2,50),
 (2,4,1,120),(2,3,2,50),(2,2,6,5),
 (3,1,10,2.5),(3,3,1,50),
 (4,5,1,300),
 (5,3,1,50),
 (6,1,10,2.5),(6,3,2,50),
 (7,4,1,120),(7,3,1,50),(7,1,12,2.5),
 (8,5,1,300),(8,3,2,50),
 (9,3,3,50),
 (10,2,20,5),
 (11,3,4,50),
 (12,5,1,300);

INSERT INTO Contacts VALUES (1,'a@x.com'),(2,'b@x.com'),(3,'a@x.com'),(4,'c@x.com'),(5,'b@x.com'),(6,'a@x.com');

INSERT INTO Tickets VALUES (1),(2),(3),(5),(6),(9),(10);

INSERT INTO Logins VALUES
 (1,'2024-01-15'),(1,'2024-02-10'),(1,'2024-03-01'),(1,'2024-03-02'),(1,'2024-03-03'),(1,'2024-03-05'),
 (1,'2024-03-06'),(1,'2024-03-07'),(1,'2024-03-08'),
 (2,'2024-01-20'),(2,'2024-03-01'),(2,'2024-03-03'),(2,'2024-03-04'),(2,'2024-03-05'),
 (3,'2024-02-05'),(3,'2024-02-25'),(3,'2024-03-02'),(3,'2024-03-10'),
 (4,'2024-02-12'),(5,'2024-03-15');

INSERT INTO Events (UserId, EventTime) VALUES
 (1,'2024-03-01 10:00'),(1,'2024-03-01 10:10'),(1,'2024-03-01 10:25'),
 (1,'2024-03-01 11:30'),(1,'2024-03-01 11:40'),(1,'2024-03-01 15:00'),
 (2,'2024-03-01 09:00'),(2,'2024-03-01 09:50');
```

Facts to keep in mind: salaries sorted are 60k, 80k, 90k, 90k, 100k, 100k, 120k (ties!); the company has 7 employees; Zed and Lee never ordered; product *Mug* never sold; yearly revenue is 2022 = 500, 2023 = 1100, 2024 = 650.

### 4.1 [Easy] Nth highest salary

**Problem**: Return the 3rd highest *distinct* salary.

**Hint**: `DENSE_RANK()` handles ties; `OFFSET … FETCH` is the alternative.

**Solution**
```sql
SELECT DISTINCT Salary AS ThirdHighest
FROM (SELECT Salary, DENSE_RANK() OVER (ORDER BY Salary DESC) AS rnk FROM Employees) x
WHERE rnk = 3;

-- returns NULL (not an empty set) when there is no 3rd value:
SELECT (SELECT DISTINCT Salary FROM Employees ORDER BY Salary DESC OFFSET 2 ROWS FETCH NEXT 1 ROWS ONLY) AS ThirdHighest;
```
Result (both): `ThirdHighest = 90000.00`

**Explanation / complexity**: Distinct salaries ranked: 120k=1, 100k=2, 90k=3. `ROW_NUMBER()` would wrongly return 100000 (ties get distinct numbers); `RANK()` assigns 1,2,2,4,4,6,7 to the raw rows, so it also gives the wrong answer for the 3rd *distinct* value. Sort is O(n log n); an index on `Salary` makes it an index scan.

**Follow-up variations**: Nth highest per department; the classic correlated subquery `WHERE (SELECT COUNT(DISTINCT Salary) FROM Employees e2 WHERE e2.Salary > e1.Salary) = n-1`; MySQL `LIMIT 1 OFFSET n-1`.

### 4.2 [Easy] Highest-paid employee(s) per department

**Problem**: Show the top earner(s) in each department, keeping ties.

**Hint**: Correlated `MAX` subquery (or `RANK() = 1`).

**Solution**
```sql
SELECT d.DeptName, e.Name, e.Salary
FROM Employees e JOIN Departments d ON d.DeptId = e.DeptId
WHERE e.Salary = (SELECT MAX(Salary) FROM Employees WHERE DeptId = e.DeptId)
ORDER BY d.DeptName, e.Name;
```
Result:

| DeptName | Name | Salary |
|---|---|---|
| Engineering | Alice | 120000.00 |
| HR | Frank | 60000.00 |
| Sales | Eve | 90000.00 |

**Explanation / complexity**: Inner join drops Legal (no employees). Correlated subquery is evaluated per department (optimiser usually turns it into a join/aggregate): O(n) with an index on `(DeptId, Salary)`.

**Follow-up variations**: Lowest paid; employees paid above their dept average (4.26); `FIRST_VALUE`.

### 4.3 [Easy] Find duplicates

**Problem**: Which emails appear more than once in `Contacts`?

**Hint**: `GROUP BY` + `HAVING COUNT(*) > 1`.

**Solution**
```sql
SELECT Email, COUNT(*) AS Cnt
FROM Contacts
GROUP BY Email
HAVING COUNT(*) > 1
ORDER BY Email;
```
Result:

| Email | Cnt |
|---|---|
| a@x.com | 3 |
| b@x.com | 2 |

**Explanation / complexity**: `HAVING` filters *after* aggregation (`WHERE` filters rows before). O(n) hash/stream aggregate; a unique index would have prevented this.

**Follow-up variations**: Show the duplicate rows themselves (`JOIN` back or `COUNT(*) OVER (PARTITION BY Email)`); duplicates across multiple columns; case-insensitive duplicates (`LOWER(Email)`).

### 4.4 [Medium] Delete duplicates, keep the lowest id

**Problem**: Remove duplicate emails, keeping the row with the smallest `ContactId`.

**Hint**: Number rows within each email group; delete rn > 1. SQL Server lets you delete through a CTE.

**Solution**
```sql
BEGIN TRAN;   -- demo only: roll back to keep the data for the other exercises

WITH d AS (
    SELECT ROW_NUMBER() OVER (PARTITION BY Email ORDER BY ContactId) AS rn
    FROM Contacts)
DELETE FROM d WHERE rn > 1;

SELECT ContactId, Email FROM Contacts ORDER BY ContactId;
ROLLBACK;
```
Result (before rollback):

| ContactId | Email |
|---|---|
| 1 | a@x.com |
| 2 | b@x.com |
| 4 | c@x.com |

**Explanation / complexity**: One sort/partition pass, O(n log n). Portable form: `DELETE FROM Contacts WHERE ContactId NOT IN (SELECT MIN(ContactId) FROM Contacts GROUP BY Email)`. Always `SELECT` first, run in a transaction, then add a unique constraint.

**Follow-up variations**: Keep the *latest* row (`ORDER BY ContactId DESC`); delete duplicates on a heap table without any id (use the CTE approach with `ORDER BY (SELECT NULL)`); MySQL self-join delete.

### 4.5 [Medium] Running total

**Problem**: Cumulative revenue ordered by date for 2023 orders.

**Hint**: `SUM() OVER (ORDER BY … ROWS UNBOUNDED PRECEDING)`; add a tiebreaker column.

**Solution**
```sql
SELECT OrderId, OrderDate, TotalAmount,
       SUM(TotalAmount) OVER (ORDER BY OrderDate, OrderId
                              ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS RunningTotal
FROM Orders
WHERE YEAR(OrderDate) = 2023
ORDER BY OrderDate, OrderId;
```
Result:

| OrderId | OrderDate | TotalAmount | RunningTotal |
|---|---|---|---|
| 1 | 2023-01-12 | 100.00 | 100.00 |
| 2 | 2023-01-20 | 250.00 | 350.00 |
| 3 | 2023-02-03 | 75.00 | 425.00 |
| 4 | 2023-02-14 | 300.00 | 725.00 |
| 5 | 2023-03-01 | 50.00 | 775.00 |
| 6 | 2023-03-09 | 125.00 | 900.00 |
| 7 | 2023-03-15 | 200.00 | 1100.00 |

**Explanation / complexity**: Explicit `ROWS` frames are much faster than the default `RANGE` frame (which spools to disk) and avoid "peer" rows with equal order keys being summed together. O(n log n) for the sort. Prefer `YEAR(col) = 2023` → `OrderDate >= '2023-01-01' AND OrderDate < '2024-01-01'` in production so an index is usable (sargable).

**Follow-up variations**: Reset per month (`PARTITION BY YEAR, MONTH`); pre-2012 alternative with a self-join or `CROSS APPLY` (O(n²)).

### 4.6 [Medium] Running total per customer

**Problem**: Cumulative spend per customer (customers 1 and 2 only).

**Hint**: Add `PARTITION BY CustomerId`.

**Solution**
```sql
SELECT CustomerId, OrderId, OrderDate, TotalAmount,
       SUM(TotalAmount) OVER (PARTITION BY CustomerId ORDER BY OrderDate, OrderId) AS RunningTotal
FROM Orders
WHERE CustomerId IN (1, 2)
ORDER BY CustomerId, OrderDate, OrderId;
```
Result:

| CustomerId | OrderId | OrderDate | TotalAmount | RunningTotal |
|---|---|---|---|---|
| 1 | 11 | 2022-03-05 | 200.00 | 200.00 |
| 1 | 1 | 2023-01-12 | 100.00 | 300.00 |
| 1 | 2 | 2023-01-20 | 250.00 | 550.00 |
| 1 | 5 | 2023-03-01 | 50.00 | 600.00 |
| 1 | 8 | 2024-01-12 | 400.00 | 1000.00 |
| 2 | 12 | 2022-06-10 | 300.00 | 300.00 |
| 2 | 3 | 2023-02-03 | 75.00 | 375.00 |
| 2 | 6 | 2023-03-09 | 125.00 | 500.00 |
| 2 | 10 | 2024-02-20 | 100.00 | 600.00 |

**Explanation / complexity**: The window restarts at each partition. Because `OrderDate, OrderId` is unique here the default `RANGE` frame happens to behave like `ROWS`, but state `ROWS` explicitly in real code.

**Follow-up variations**: Order number per customer (`ROW_NUMBER`); days since previous order (`DATEDIFF` + `LAG`); cumulative % of customer total.

### 4.7 [Medium] Find missing IDs

**Problem**: Which ticket numbers between 1 and the maximum are missing?

**Hint**: Generate a number series and anti-join.

**Solution**
```sql
SELECT g.value AS MissingId
FROM GENERATE_SERIES(1, (SELECT MAX(TicketId) FROM Tickets)) AS g
WHERE NOT EXISTS (SELECT 1 FROM Tickets t WHERE t.TicketId = g.value);
```
Result: `MissingId` = **4, 7, 8**

**Explanation / complexity**: O(max id). Before SQL Server 2022 use a recursive CTE (`OPTION (MAXRECURSION 0)`), a tally/numbers table, or `master..spt_values`. For a large sparse table prefer gap *ranges* (4.8).

**Follow-up variations**: Missing dates in a date range; the smallest missing positive id (`MIN(TicketId + 1) … NOT EXISTS`).

### 4.8 [Hard] Gaps and islands

**Problem**: Show the ranges of consecutive ticket ids (islands) and the ranges that are missing (gaps).

**Hint**: Islands: `value − ROW_NUMBER()` is constant within a consecutive run. Gaps: `LEAD` and compare to the current value.

**Solution**
```sql
-- islands
SELECT MIN(TicketId) AS StartId, MAX(TicketId) AS EndId
FROM (SELECT TicketId, TicketId - ROW_NUMBER() OVER (ORDER BY TicketId) AS grp FROM Tickets) x
GROUP BY grp
ORDER BY StartId;

-- gaps
SELECT TicketId + 1 AS GapStart, NextId - 1 AS GapEnd
FROM (SELECT TicketId, LEAD(TicketId) OVER (ORDER BY TicketId) AS NextId FROM Tickets) x
WHERE NextId - TicketId > 1;
```
Result – islands:

| StartId | EndId |
|---|---|
| 1 | 3 |
| 5 | 6 |
| 9 | 10 |

Result – gaps:

| GapStart | GapEnd |
|---|---|
| 4 | 4 |
| 7 | 8 |

**Explanation / complexity**: Ids 1,2,3,5,6,9,10 minus row numbers 1..7 gives 0,0,0,1,1,3,3 – the constant `grp` identifies each island. One sort, O(n log n) (O(n) with a clustered index on the key).

**Follow-up variations**: Islands over dates (`DATEADD(DAY, -ROW_NUMBER(), date)`, see 4.11); islands of equal status values (difference of two row numbers).

### 4.9 [Medium] Top N per group

**Problem**: The top 2 salary levels in each department (ties included).

**Hint**: Rank within `PARTITION BY DeptId`, filter in an outer query (window functions can't be used in `WHERE`).

**Solution**
```sql
SELECT DeptName, Name, Salary, rnk
FROM (SELECT d.DeptName, e.Name, e.Salary,
             DENSE_RANK() OVER (PARTITION BY e.DeptId ORDER BY e.Salary DESC) AS rnk
      FROM Employees e JOIN Departments d ON d.DeptId = e.DeptId) x
WHERE rnk <= 2
ORDER BY DeptName, rnk, Name;
```
Result:

| DeptName | Name | Salary | rnk |
|---|---|---|---|
| Engineering | Alice | 120000.00 | 1 |
| Engineering | Bob | 100000.00 | 2 |
| Engineering | Carol | 100000.00 | 2 |
| HR | Frank | 60000.00 | 1 |
| Sales | Eve | 90000.00 | 1 |
| Sales | Dave | 80000.00 | 2 |

**Explanation / complexity**: Grace (90k) is rank 3 in Engineering and is excluded. Use `ROW_NUMBER()` when you want *exactly* N rows per group; `DENSE_RANK`/`RANK` keep ties. Supporting index: `(DeptId, Salary DESC)`. Alternative: `CROSS APPLY (SELECT TOP (2) … WHERE DeptId = d.DeptId ORDER BY Salary DESC)`.

**Follow-up variations**: Top 3 products per category by revenue (see 4.23); `QUALIFY` in Snowflake/BigQuery instead of the wrapper query.

### 4.10 [Easy] Employees earning more than their manager

**Problem**: Find employees whose salary is higher than their manager's.

**Hint**: Self-join `Employees` on `ManagerId = EmpId`.

**Solution**
```sql
SELECT e.Name AS Employee, e.Salary, m.Name AS Manager, m.Salary AS ManagerSalary
FROM Employees e
JOIN Employees m ON m.EmpId = e.ManagerId
WHERE e.Salary > m.Salary;
```
Result:

| Employee | Salary | Manager | ManagerSalary |
|---|---|---|---|
| Eve | 90000.00 | Dave | 80000.00 |

**Explanation / complexity**: The inner join also excludes Alice (NULL manager). O(n) with a PK lookup. Grace (90k) reports to Bob (100k), so she isn't included.

**Follow-up variations**: Employees with no manager (`LEFT JOIN … IS NULL`); managers with more than N reports; earning more than the *department* average.

### 4.11 [Hard] Consecutive days – longest login streak

**Problem**: For each customer, find the longest streak of consecutive login days.

**Hint**: `date − ROW_NUMBER()` is constant for consecutive dates; group by it.

**Solution**
```sql
WITH g AS (
    SELECT CustomerId, LoginDate,
           DATEADD(DAY, -ROW_NUMBER() OVER (PARTITION BY CustomerId ORDER BY LoginDate), LoginDate) AS grp
    FROM Logins),
s AS (
    SELECT CustomerId, MIN(LoginDate) AS StartDate, MAX(LoginDate) AS EndDate, COUNT(*) AS Days
    FROM g GROUP BY CustomerId, grp)
SELECT CustomerId, StartDate, EndDate, Days
FROM (SELECT *, ROW_NUMBER() OVER (PARTITION BY CustomerId ORDER BY Days DESC, StartDate) AS rn FROM s) t
WHERE rn = 1
ORDER BY CustomerId;
```
Result:

| CustomerId | StartDate | EndDate | Days |
|---|---|---|---|
| 1 | 2024-03-05 | 2024-03-08 | 4 |
| 2 | 2024-03-03 | 2024-03-05 | 3 |
| 3 | 2024-02-05 | 2024-02-05 | 1 |
| 4 | 2024-02-12 | 2024-02-12 | 1 |
| 5 | 2024-03-15 | 2024-03-15 | 1 |

**Explanation / complexity**: Customer 1 has streaks 03-01..03-03 (3 days) and 03-05..03-08 (4 days). The `(CustomerId, LoginDate)` primary key guarantees no duplicate dates; with duplicates apply `DISTINCT` first or `ROW_NUMBER` would break the arithmetic. O(n log n).

**Follow-up variations**: Customers with a streak ≥ 3 days; current streak as of today; allow a 1-day gap.

### 4.12 [Medium] Pivot rows to columns

**Problem**: Total 2023 order amount per customer for Jan, Feb, Mar as columns (customers with no orders show zeros).

**Hint**: Conditional aggregation (portable) or `PIVOT` (T-SQL).

**Solution**
```sql
-- conditional aggregation
SELECT c.Name,
       SUM(CASE WHEN MONTH(o.OrderDate) = 1 THEN o.TotalAmount ELSE 0 END) AS Jan,
       SUM(CASE WHEN MONTH(o.OrderDate) = 2 THEN o.TotalAmount ELSE 0 END) AS Feb,
       SUM(CASE WHEN MONTH(o.OrderDate) = 3 THEN o.TotalAmount ELSE 0 END) AS Mar
FROM Customers c
LEFT JOIN Orders o ON o.CustomerId = c.CustomerId AND YEAR(o.OrderDate) = 2023
GROUP BY c.CustomerId, c.Name
ORDER BY c.CustomerId;

-- PIVOT
SELECT Name, ISNULL([1], 0) AS Jan, ISNULL([2], 0) AS Feb, ISNULL([3], 0) AS Mar
FROM (SELECT c.CustomerId, c.Name, MONTH(o.OrderDate) AS M, o.TotalAmount
      FROM Customers c
      LEFT JOIN Orders o ON o.CustomerId = c.CustomerId AND YEAR(o.OrderDate) = 2023) s
PIVOT (SUM(TotalAmount) FOR M IN ([1], [2], [3])) p
ORDER BY CustomerId;
```
Result (identical for both):

| Name | Jan | Feb | Mar |
|---|---|---|---|
| Ann | 350.00 | 0.00 | 50.00 |
| Raj | 0.00 | 75.00 | 125.00 |
| Mia | 0.00 | 300.00 | 200.00 |
| Zed | 0.00 | 0.00 | 0.00 |
| Lee | 0.00 | 0.00 | 0.00 |

**Explanation / complexity**: The year filter lives in the `ON` clause so customers without matching orders are preserved (putting it in `WHERE` would turn the left join into an inner join). Columns of `PIVOT` must be known at compile time; for dynamic columns build the `IN` list with `STRING_AGG`/`QUOTENAME` and `sp_executesql`. O(n).

**Follow-up variations**: Unpivot (`UNPIVOT` / `CROSS APPLY (VALUES …)`); counts instead of sums; add a total column.

### 4.13 [Hard] Recursive hierarchy (org chart)

**Problem**: List every employee with their level and the management path from the top.

**Hint**: Recursive CTE: anchor = employees without a manager; recursive member joins on `ManagerId`.

**Solution**
```sql
WITH org AS (
    SELECT EmpId, Name, 0 AS Lvl, CAST(Name AS VARCHAR(500)) AS Path
    FROM Employees WHERE ManagerId IS NULL
    UNION ALL
    SELECT e.EmpId, e.Name, o.Lvl + 1, CAST(o.Path + ' > ' + e.Name AS VARCHAR(500))
    FROM Employees e JOIN org o ON e.ManagerId = o.EmpId)
SELECT REPLICATE('  ', Lvl) + Name AS OrgChart, Lvl, Path
FROM org
ORDER BY Path;
```
Result:

| OrgChart | Lvl | Path |
|---|---|---|
| Alice | 0 | Alice |
| &nbsp;&nbsp;Bob | 1 | Alice > Bob |
| &nbsp;&nbsp;&nbsp;&nbsp;Grace | 2 | Alice > Bob > Grace |
| &nbsp;&nbsp;Carol | 1 | Alice > Carol |
| &nbsp;&nbsp;Dave | 1 | Alice > Dave |
| &nbsp;&nbsp;&nbsp;&nbsp;Eve | 2 | Alice > Dave > Eve |
| &nbsp;&nbsp;Frank | 1 | Alice > Frank |

**Explanation / complexity**: Each recursion step expands one level; total work O(n) for a tree. The default recursion limit is 100 (`OPTION (MAXRECURSION n)`); cycles in bad data cause an error, so add a depth guard. Both `Path` columns must have identical types/lengths in anchor and recursive part (hence the `CAST`). Sorting by `Path` yields depth-first order.

**Follow-up variations**: All subordinates of a given manager; total salary of each subtree; `hierarchyid`; PostgreSQL needs `WITH RECURSIVE`.

### 4.14 [Hard] Median

**Problem**: Median salary of all employees.

**Hint**: `PERCENTILE_CONT(0.5)`, or number the rows and average the middle one/two.

**Solution**
```sql
SELECT DISTINCT PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY Salary) OVER () AS MedianSalary
FROM Employees;

-- portable version
SELECT AVG(Salary) AS MedianSalary
FROM (SELECT Salary, ROW_NUMBER() OVER (ORDER BY Salary) AS rn, COUNT(*) OVER () AS cnt FROM Employees) x
WHERE rn IN ((cnt + 1) / 2, (cnt + 2) / 2);
```
Result (both): `MedianSalary = 90000` (displayed as 90000.0 / 90000.000000 depending on the type)

**Explanation / complexity**: Seven rows → the 4th value (sorted: 60, 80, 90, **90**, 100, 100, 120k). For an even count the two middle rows are averaged (integer division makes `(cnt+1)/2` and `(cnt+2)/2` equal for odd counts, different for even). `PERCENTILE_CONT` is an analytic function, so `DISTINCT` (or `TOP 1`) collapses the per-row repeats. `PERCENTILE_DISC` returns an actual value instead of interpolating. O(n log n).

**Follow-up variations**: Median per department (`PARTITION BY DeptId` in the `OVER`); mode; 90th percentile.

### 4.15 [Hard] Retention / cohort analysis

**Problem**: Group customers by the month of their first login (cohort); for each following month show how many were active and the retention %.

**Hint**: First-login month per customer; distinct active months; `DATEDIFF(MONTH, cohort, activeMonth)` is the cohort age.

**Solution**
```sql
WITH first_login AS (
    SELECT CustomerId, DATEFROMPARTS(YEAR(MIN(LoginDate)), MONTH(MIN(LoginDate)), 1) AS Cohort
    FROM Logins GROUP BY CustomerId),
activity AS (
    SELECT DISTINCT CustomerId, DATEFROMPARTS(YEAR(LoginDate), MONTH(LoginDate), 1) AS ActiveMonth
    FROM Logins),
cohort_size AS (
    SELECT Cohort, COUNT(*) AS Size FROM first_login GROUP BY Cohort)
SELECT f.Cohort, DATEDIFF(MONTH, f.Cohort, a.ActiveMonth) AS MonthN, COUNT(*) AS Users, cs.Size,
       CAST(100.0 * COUNT(*) / cs.Size AS DECIMAL(5,1)) AS RetentionPct
FROM first_login f
JOIN activity a ON a.CustomerId = f.CustomerId
JOIN cohort_size cs ON cs.Cohort = f.Cohort
GROUP BY f.Cohort, DATEDIFF(MONTH, f.Cohort, a.ActiveMonth), cs.Size
ORDER BY f.Cohort, MonthN;
```
Result:

| Cohort | MonthN | Users | Size | RetentionPct |
|---|---|---|---|---|
| 2024-01-01 | 0 | 2 | 2 | 100.0 |
| 2024-01-01 | 1 | 1 | 2 | 50.0 |
| 2024-01-01 | 2 | 2 | 2 | 100.0 |
| 2024-02-01 | 0 | 2 | 2 | 100.0 |
| 2024-02-01 | 1 | 1 | 2 | 50.0 |
| 2024-03-01 | 0 | 1 | 1 | 100.0 |

**Explanation / complexity**: January cohort = customers 1, 2. In February only customer 1 logged in (50 %), in March both did (100 %) – "returning" retention, not "rolling" retention. The `DISTINCT` in `activity` prevents counting a customer several times per month. O(n).

**Follow-up variations**: Pivot into the classic triangle (`MonthN` as columns); rolling retention (active in month N *or later*); weekly cohorts; churn.

### 4.16 [Easy] Customers without orders

**Problem**: Which customers have never placed an order?

**Hint**: `NOT EXISTS` or `LEFT JOIN … IS NULL` (avoid `NOT IN` with nullable columns).

**Solution**
```sql
SELECT c.CustomerId, c.Name
FROM Customers c
WHERE NOT EXISTS (SELECT 1 FROM Orders o WHERE o.CustomerId = c.CustomerId);

SELECT c.CustomerId, c.Name
FROM Customers c LEFT JOIN Orders o ON o.CustomerId = c.CustomerId
WHERE o.OrderId IS NULL;
```
Result (both):

| CustomerId | Name |
|---|---|
| 4 | Zed |
| 5 | Lee |

**Explanation / complexity**: Both compile to an anti-semi-join (hash or merge) – O(n + m). `NOT IN (SELECT CustomerId …)` returns **no rows at all** if the subquery yields a single `NULL`.

**Follow-up variations**: Customers who ordered in 2023 but not 2024; products never sold (4.22).

### 4.17 [Medium] Year-over-year growth

**Problem**: Revenue per year with the previous year's revenue and % growth.

**Hint**: `LAG(Revenue) OVER (ORDER BY Yr)`; guard the division with `NULLIF`.

**Solution**
```sql
WITH y AS (
    SELECT YEAR(OrderDate) AS Yr, SUM(TotalAmount) AS Revenue
    FROM Orders GROUP BY YEAR(OrderDate))
SELECT Yr, Revenue,
       LAG(Revenue) OVER (ORDER BY Yr) AS PrevRevenue,
       CAST(100.0 * (Revenue - LAG(Revenue) OVER (ORDER BY Yr))
            / NULLIF(LAG(Revenue) OVER (ORDER BY Yr), 0) AS DECIMAL(7,1)) AS YoYPct
FROM y
ORDER BY Yr;
```
Result:

| Yr | Revenue | PrevRevenue | YoYPct |
|---|---|---|---|
| 2022 | 500.00 | NULL | NULL |
| 2023 | 1100.00 | 500.00 | 120.0 |
| 2024 | 650.00 | 1100.00 | -40.9 |

**Explanation / complexity**: (1100−500)/500 = +120 %; (650−1100)/1100 = −40.9 %. First year has no predecessor → `NULL`. Missing years would make `LAG` compare non-adjacent years – join to a calendar table if gaps are possible. O(n log n).

**Follow-up variations**: Month-over-month and same-month-last-year (`LAG(Revenue, 12)`); self-join alternative `ON y2.Yr = y1.Yr - 1`; per-customer YoY.

### 4.18 [Hard] Sessionization with window functions

**Problem**: Split each user's events into sessions: a gap of more than 30 minutes starts a new session. Report start, end, and count.

**Hint**: Flag session starts with `LAG`, then a running `SUM` of the flags is the session number.

**Solution**
```sql
WITH flagged AS (
    SELECT UserId, EventTime,
           CASE WHEN LAG(EventTime) OVER (PARTITION BY UserId ORDER BY EventTime) IS NULL
                  OR DATEDIFF(MINUTE, LAG(EventTime) OVER (PARTITION BY UserId ORDER BY EventTime), EventTime) > 30
                THEN 1 ELSE 0 END AS NewSession
    FROM Events),
numbered AS (
    SELECT UserId, EventTime,
           SUM(NewSession) OVER (PARTITION BY UserId ORDER BY EventTime ROWS UNBOUNDED PRECEDING) AS SessionNo
    FROM flagged)
SELECT UserId, SessionNo, MIN(EventTime) AS StartTime, MAX(EventTime) AS EndTime, COUNT(*) AS Events
FROM numbered
GROUP BY UserId, SessionNo
ORDER BY UserId, SessionNo;
```
Result:

| UserId | SessionNo | StartTime | EndTime | Events |
|---|---|---|---|---|
| 1 | 1 | 2024-03-01 10:00:00 | 2024-03-01 10:25:00 | 3 |
| 1 | 2 | 2024-03-01 11:30:00 | 2024-03-01 11:40:00 | 2 |
| 1 | 3 | 2024-03-01 15:00:00 | 2024-03-01 15:00:00 | 1 |
| 2 | 1 | 2024-03-01 09:00:00 | 2024-03-01 09:00:00 | 1 |
| 2 | 2 | 2024-03-01 09:50:00 | 2024-03-01 09:50:00 | 1 |

**Explanation / complexity**: Gaps: user 1 → 65 and 200 minutes (new sessions); user 2 → 50 minutes (new session). This is the "gaps and islands" pattern with a time threshold. `DATEDIFF(MINUTE)` counts minute *boundaries*; use `SECOND` for exact thresholds. One sort per partition, O(n log n); index on `(UserId, EventTime)`.

**Follow-up variations**: Session duration/avg events per session; `LAG` with a default value to drop the `IS NULL` check; 30-minute *inactivity* vs 24-hour maximum session length.

### 4.19 [Medium] Departments with above-average pay

**Problem**: Departments whose average salary exceeds the company average.

**Hint**: `HAVING` can compare an aggregate to a scalar subquery.

**Solution**
```sql
SELECT d.DeptName, AVG(e.Salary) AS AvgSalary
FROM Employees e JOIN Departments d ON d.DeptId = e.DeptId
GROUP BY d.DeptName
HAVING AVG(e.Salary) > (SELECT AVG(Salary) FROM Employees);
```
Result:

| DeptName | AvgSalary |
|---|---|
| Engineering | 102500.000000 |

**Explanation / complexity**: Company average = 640 000 / 7 ≈ 91 428.57. Engineering (102 500) passes; Sales (85 000) and HR (60 000) do not. The subquery is uncorrelated → computed once. O(n).

**Follow-up variations**: Employees above their own department average (4.26); departments with ≥ 2 employees.

### 4.20 [Easy] Counts including empty groups

**Problem**: Headcount per department, including departments with no employees.

**Hint**: `LEFT JOIN` and `COUNT(e.EmpId)` (not `COUNT(*)`).

**Solution**
```sql
SELECT d.DeptName, COUNT(e.EmpId) AS Headcount
FROM Departments d LEFT JOIN Employees e ON e.DeptId = d.DeptId
GROUP BY d.DeptId, d.DeptName
ORDER BY d.DeptName;
```
Result:

| DeptName | Headcount |
|---|---|
| Engineering | 4 |
| HR | 1 |
| Legal | 0 |
| Sales | 2 |

**Explanation / complexity**: `COUNT(*)` would report 1 for Legal because the null-extended row still counts; `COUNT(column)` ignores NULLs. O(n + m).

**Follow-up variations**: Only departments with zero employees; headcount percentage per department.

### 4.21 [Medium] Latest row per group

**Problem**: Each customer's most recent order.

**Hint**: `ROW_NUMBER() OVER (PARTITION BY … ORDER BY date DESC)` and filter `rn = 1`.

**Solution**
```sql
SELECT CustomerId, OrderId, OrderDate, TotalAmount
FROM (SELECT *, ROW_NUMBER() OVER (PARTITION BY CustomerId ORDER BY OrderDate DESC, OrderId DESC) AS rn
      FROM Orders) x
WHERE rn = 1
ORDER BY CustomerId;
```
Result:

| CustomerId | OrderId | OrderDate | TotalAmount |
|---|---|---|---|
| 1 | 8 | 2024-01-12 | 400.00 |
| 2 | 10 | 2024-02-20 | 100.00 |
| 3 | 9 | 2024-02-02 | 150.00 |

**Explanation / complexity**: A deterministic tiebreaker (`OrderId DESC`) matters when two orders share a date. O(n log n); index `(CustomerId, OrderDate DESC)` allows a streaming plan. Alternatives: `CROSS APPLY (SELECT TOP 1 …)` or `WHERE OrderDate = (SELECT MAX…)`.

**Follow-up variations**: First order per customer; customers whose latest order is older than 90 days; latest order *with* customer name (join).

### 4.22 [Easy] Products never sold

**Problem**: Find products that appear in no order.

**Hint**: Anti-join on `OrderItems`.

**Solution**
```sql
SELECT p.ProductId, p.ProductName
FROM Products p
LEFT JOIN OrderItems oi ON oi.ProductId = p.ProductId
WHERE oi.ProductId IS NULL;
```
Result:

| ProductId | ProductName |
|---|---|
| 6 | Mug |

**Explanation / complexity**: O(n + m). Test the join key column (or the PK of the right table) for `NULL`, never a column that can legitimately be NULL.

**Follow-up variations**: Products sold only once; products never sold in 2024; categories where *no* product sold.

### 4.23 [Medium] Revenue ranking within a category (ties)

**Problem**: Revenue per product (`Qty × UnitPrice`) with `RANK` inside each category.

**Hint**: Aggregate first, then rank over the aggregate: `RANK() OVER (… ORDER BY SUM(...))`.

**Solution**
```sql
SELECT p.Category, p.ProductName, SUM(oi.Qty * oi.UnitPrice) AS Revenue,
       RANK() OVER (PARTITION BY p.Category ORDER BY SUM(oi.Qty * oi.UnitPrice) DESC) AS RankInCategory
FROM Products p
JOIN OrderItems oi ON oi.ProductId = p.ProductId
GROUP BY p.Category, p.ProductName
ORDER BY p.Category, RankInCategory, p.ProductName;
```
Result:

| Category | ProductName | Revenue | RankInCategory |
|---|---|---|---|
| Home | Desk | 900.00 | 1 |
| Home | Lamp | 900.00 | 1 |
| Home | Chair | 240.00 | 3 |
| Stationery | Notebook | 130.00 | 1 |
| Stationery | Pen | 80.00 | 2 |

**Explanation / complexity**: Windows are evaluated *after* `GROUP BY`, so they can reference aggregates. Desk and Lamp tie at 900, so `RANK` leaves a gap (next rank is 3; `DENSE_RANK` would give 2). O(n log n).

**Follow-up variations**: Top 1 per category with ties; share of category revenue (`SUM(...) OVER (PARTITION BY Category)`); ABC classification with `NTILE`.

### 4.24 [Medium] Moving average

**Problem**: 3-order moving average of 2023 order totals (current + two previous).

**Hint**: `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW`.

**Solution**
```sql
SELECT OrderId, OrderDate, TotalAmount,
       CAST(AVG(TotalAmount) OVER (ORDER BY OrderDate, OrderId
                                   ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS DECIMAL(10,1)) AS MovingAvg3
FROM Orders
WHERE YEAR(OrderDate) = 2023
ORDER BY OrderDate, OrderId;
```
Result:

| OrderId | OrderDate | TotalAmount | MovingAvg3 |
|---|---|---|---|
| 1 | 2023-01-12 | 100.00 | 100.0 |
| 2 | 2023-01-20 | 250.00 | 175.0 |
| 3 | 2023-02-03 | 75.00 | 141.7 |
| 4 | 2023-02-14 | 300.00 | 208.3 |
| 5 | 2023-03-01 | 50.00 | 141.7 |
| 6 | 2023-03-09 | 125.00 | 158.3 |
| 7 | 2023-03-15 | 200.00 | 125.0 |

**Explanation / complexity**: The first two rows average a partial window (1 and 2 rows). To output only full windows add `COUNT(*) OVER (same frame) = 3` in a wrapper. O(n).

**Follow-up variations**: Time-based window (`RANGE` with intervals is not in T-SQL → use `CROSS APPLY` or a date table); centred window `1 PRECEDING AND 1 FOLLOWING`.

### 4.25 [Medium] Percent of total

**Problem**: Each customer's share of total revenue.

**Hint**: A window over the aggregate: `SUM(SUM(x)) OVER ()`.

**Solution**
```sql
SELECT c.Name, SUM(o.TotalAmount) AS Total,
       CAST(100.0 * SUM(o.TotalAmount) / SUM(SUM(o.TotalAmount)) OVER () AS DECIMAL(5,1)) AS PctOfTotal
FROM Customers c
JOIN Orders o ON o.CustomerId = c.CustomerId
GROUP BY c.Name
ORDER BY Total DESC;
```
Result:

| Name | Total | PctOfTotal |
|---|---|---|
| Ann | 1000.00 | 44.4 |
| Mia | 650.00 | 28.9 |
| Raj | 600.00 | 26.7 |

**Explanation / complexity**: Grand total = 2250. The inner `SUM` is the group aggregate; the outer `SUM … OVER ()` totals it across all groups – no self-join or subquery needed. O(n).

**Follow-up variations**: Cumulative percent (Pareto/80-20 analysis); percent within city (`PARTITION BY City`).

### 4.26 [Medium] Compare each row to its group average

**Problem**: Show every employee's difference from their department's average salary.

**Hint**: `AVG(Salary) OVER (PARTITION BY DeptId)` keeps row-level detail (no `GROUP BY`).

**Solution**
```sql
SELECT d.DeptName, e.Name, e.Salary,
       AVG(e.Salary) OVER (PARTITION BY e.DeptId) AS DeptAvg,
       e.Salary - AVG(e.Salary) OVER (PARTITION BY e.DeptId) AS DiffFromAvg
FROM Employees e JOIN Departments d ON d.DeptId = e.DeptId
ORDER BY d.DeptName, e.Salary DESC, e.Name;
```
Result:

| DeptName | Name | Salary | DeptAvg | DiffFromAvg |
|---|---|---|---|---|
| Engineering | Alice | 120000.00 | 102500.00 | 17500.00 |
| Engineering | Bob | 100000.00 | 102500.00 | -2500.00 |
| Engineering | Carol | 100000.00 | 102500.00 | -2500.00 |
| Engineering | Grace | 90000.00 | 102500.00 | -12500.00 |
| HR | Frank | 60000.00 | 60000.00 | 0.00 |
| Sales | Eve | 90000.00 | 85000.00 | 5000.00 |
| Sales | Dave | 80000.00 | 85000.00 | -5000.00 |

(SQL Server returns `AVG` of `DECIMAL(10,2)` with scale 6, e.g. `102500.000000`; shown here at 2 decimals.)

**Explanation / complexity**: One pass with a window aggregate instead of a join to a grouped subquery. O(n log n) for partitioning.

**Follow-up variations**: Only employees above the department average (wrap and filter); z-score (`STDEV() OVER`).

### 4.27 [Hard] Relational division – "bought all of …"

**Problem**: Customers who have bought *every* product in the `Stationery` category.

**Hint**: Count the distinct matching products per customer and compare with the category size.

**Solution**
```sql
SELECT c.Name
FROM Customers c
JOIN Orders o       ON o.CustomerId = c.CustomerId
JOIN OrderItems oi  ON oi.OrderId = o.OrderId
JOIN Products p     ON p.ProductId = oi.ProductId
WHERE p.Category = 'Stationery'
GROUP BY c.CustomerId, c.Name
HAVING COUNT(DISTINCT p.ProductId) = (SELECT COUNT(*) FROM Products WHERE Category = 'Stationery');
```
Result:

| Name |
|---|
| Raj |

**Explanation / complexity**: Raj bought Pen (orders 3, 6) and Notebook (order 10). Ann bought only Notebook, Mia only Pen. `COUNT(DISTINCT …)` is essential because the same product can appear in several orders. O(n) joins + aggregate.

**Follow-up variations**: "Not exists not exists" formulation (double negation); customers who bought *at least N* different categories; customers who bought Pen but never Notebook.

### 4.28 [Medium] String aggregation per group

**Problem**: One row per department listing its employees alphabetically (empty departments included).

**Hint**: `STRING_AGG(...) WITHIN GROUP (ORDER BY ...)` (SQL Server 2017+).

**Solution**
```sql
SELECT d.DeptName,
       STRING_AGG(e.Name, ', ') WITHIN GROUP (ORDER BY e.Name) AS Employees
FROM Departments d LEFT JOIN Employees e ON e.DeptId = d.DeptId
GROUP BY d.DeptName
ORDER BY d.DeptName;
```
Result:

| DeptName | Employees |
|---|---|
| Engineering | Alice, Bob, Carol, Grace |
| HR | Frank |
| Legal | NULL |
| Sales | Dave, Eve |

**Explanation / complexity**: `STRING_AGG` skips `NULL` inputs and yields `NULL` for a group with no non-null values (Legal). The result type is `VARCHAR(8000)` unless the input is `MAX` – cast to `VARCHAR(MAX)` for large groups. Pre-2017: `FOR XML PATH('')` trick. MySQL `GROUP_CONCAT`, PostgreSQL `string_agg`.

**Follow-up variations**: Distinct values only; comma-separated list → rows (`STRING_SPLIT`); limit the list length.

---

*End of E03.*
