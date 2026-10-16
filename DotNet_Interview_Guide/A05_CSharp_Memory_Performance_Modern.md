# A05 - C# Memory, Performance & Modern Language Features

**How to use:** Read each topic's *Study notes* first, then climb the question ladder (Easy -> Tricky); cover the Answer and try to say it aloud before reading.
Code targets C# 12 / .NET 8; snippets are focused fragments (in a real top-level-statements file, move type declarations below the statements; add the `using`s shown).
Finish each topic by re-reading *Interview traps* and the *Cheat sheet*.

## Table of Contents
1. [Garbage Collection](#topic-1-garbage-collection)
2. [IDisposable, Finalizers, SafeHandle, using / await using](#topic-2-idisposable-finalizers-safehandle-using--await-using)
3. [Memory Leaks in Managed Code](#topic-3-memory-leaks-in-managed-code)
4. [Span<T>, Memory<T>, ReadOnlySpan<T>, stackalloc](#topic-4-spant-memoryt-readonlyspant-stackalloc)
5. [ArrayPool and ObjectPool](#topic-5-arraypool-and-objectpool)
6. [Struct vs Class Performance & Defensive Copies](#topic-6-struct-vs-class-performance--defensive-copies)
7. [String Performance](#topic-7-string-performance)
8. [BenchmarkDotNet Basics](#topic-8-benchmarkdotnet-basics)
9. [Exception Handling](#topic-9-exception-handling)
10. [Pattern Matching](#topic-10-pattern-matching)
11. [Records and with-expressions](#topic-11-records-and-with-expressions)
12. [Nullable Reference Types](#topic-12-nullable-reference-types)
13. [C# 9-13 Feature Tour](#topic-13-c-9-13-feature-tour)
14. [Reflection, Attributes, dynamic / DLR](#topic-14-reflection-attributes-dynamic--dlr)
15. [Source Generators & Roslyn Analyzers](#topic-15-source-generators--roslyn-analyzers)
16. [Unsafe Code, Pointers, P/Invoke](#topic-16-unsafe-code-pointers-pinvoke)
17. [IL, JIT, Tiered Compilation, ReadyToRun, Native AOT](#topic-17-il-jit-tiered-compilation-readytorun-native-aot)
18. [Weak References & Lazy<T>](#topic-18-weak-references--lazyt)
19. [Serialization: System.Text.Json vs Newtonsoft](#topic-19-serialization-systemtextjson-vs-newtonsoft)

---

## Topic 1: Garbage Collection

**Study notes**

- The CLR GC is a **tracing, generational, mark-compact** collector. Allocation is a pointer bump in the *allocation context* of the current thread (very cheap); the cost is paid at collection time.
- **Generations**: gen0 (new, small, collected often), gen1 (buffer between short and long lived), gen2 (long-lived; collecting it = *full GC*). Collecting gen N also collects all younger generations. Rationale: *generational hypothesis* - most objects die young.
- **LOH** (Large Object Heap): objects >= 85,000 bytes. Logically part of gen2 (`GC.GetGeneration` returns 2). Historically **not compacted** (swept, free-list) -> fragmentation; compaction can be requested once via `GCSettings.LargeObjectHeapCompactionMode`.
- **POH** (Pinned Object Heap, .NET 5+): for long-lived pinned arrays, allocated with `GC.AllocateArray<T>(n, pinned: true)` / `GC.AllocateUninitializedArray<T>`. Keeps pinned objects from fragmenting gen0/1.
- **Phases**: *mark* (from roots: stacks, statics, handles, finalization queue, card-table for old->young refs) -> *plan* (decide compact or sweep) -> *relocate* -> *compact*. Gen0/gen1 collections are "ephemeral GCs".
- **Flavours**: Workstation (default for desktop/console; non-concurrent or concurrent) vs Server (one heap + dedicated GC thread per logical core, bigger budgets, higher throughput, more memory). **Concurrent / background GC** lets gen2 marking run concurrently with app threads; gen0/1 can still occur during it. `DOTNET_gcServer=1`, `<ServerGarbageCollector>true</ServerGarbageCollector>`, `<ConcurrentGarbageCollection>false</ConcurrentGarbageCollection>`. Also `GCHeapCount`, `GCHeapAffinitizeMask`, `DOTNET_GCgen0size`, and DATAS (dynamic adaptation, default for Server GC in .NET 9).
- GC suspends managed threads at safe points (a "stop the world" for the non-concurrent portions).

```
 Roots (stack, statics, GC handles)
      |
      v
 +-------+  survive  +-------+  survive  +--------------+
 | gen0  | --------> | gen1  | --------> |    gen2      |
 +-------+           +-------+           +--------------+
                                         |  LOH (>=85K) |  (logically gen2)
                                         |  POH (pinned)|
```

**Ladder**

### 1.1 [Easy] What are GC generations and why do they exist?
**Answer**
The managed heap is split into gen0, gen1 and gen2 so the GC can collect only the young part most of the time. New objects start in gen0; survivors are promoted. Because most objects die young, ephemeral collections are fast and full GCs are rare.

**Explanation**
A gen0 GC only needs to mark objects reachable from roots plus old-to-young references recorded in the *card table* (set by the write barrier). It never touches gen2, so pause time is proportional to survivors, not heap size. Promotion: gen0 survivor -> gen1, gen1 survivor -> gen2. Gen2 collections are expensive because they scan the entire heap, and they also include the LOH.

**Code**
```csharp
var o = new object();
Console.WriteLine(GC.GetGeneration(o)); // Output: 0
GC.Collect();                           // demo only
Console.WriteLine(GC.GetGeneration(o)); // Output: 1
GC.Collect();
Console.WriteLine(GC.GetGeneration(o)); // Output: 2
Console.WriteLine(GC.CollectionCount(0) >= GC.CollectionCount(2)); // Output: True
```

### 1.2 [Follow-up] What is the LOH and what problems does it cause? What is the POH?
**Answer**
The LOH holds objects of 85,000 bytes or more and is collected only with gen2. It is normally swept not compacted, so it can fragment and cause `OutOfMemoryException` or frequent gen2 GCs. The POH (.NET 5+) is a separate region for pinned arrays so pinning does not fragment the normal heaps.

**Explanation**
Typical offenders: big `byte[]` buffers, `List<T>` that grows past ~10.6K references (85,000/8) , large strings from `ReadToEnd`. Remedies: pool buffers (`ArrayPool<T>`), stream instead of buffering, reuse `StringBuilder`/`MemoryStream` (RecyclableMemoryStream), or set `GCSettings.LargeObjectHeapCompactionMode = GCLargeObjectHeapCompactionMode.CompactOnce` before one blocking gen2. The threshold is configurable (`GCLOHThreshold`).

**Code**
```csharp
var small = new byte[1000];
var big = new byte[100_000];
Console.WriteLine(GC.GetGeneration(small)); // Output: 0
Console.WriteLine(GC.GetGeneration(big));   // Output: 2  (LOH is logically gen2)

byte[] pinned = GC.AllocateArray<byte>(4096, pinned: true); // lives on the POH
System.Runtime.GCSettings.LargeObjectHeapCompactionMode =
    System.Runtime.GCLargeObjectHeapCompactionMode.CompactOnce;
GC.Collect(); // one-off compacting gen2
```

### 1.3 [Medium] Workstation vs Server GC, and what is concurrent/background GC?
**Answer**
Workstation GC uses a single heap and runs on the allocating thread (or a background thread), optimised for low latency on client apps. Server GC has a heap and dedicated high-priority GC thread per core, giving higher throughput at the cost of memory and longer individual pauses. Concurrent (background) GC lets gen2 marking run while the app keeps executing, shortening gen2 pauses.

**Explanation**
Server GC is the right choice for throughput-bound services on multi-core machines (don't assume which flavour a host uses; check `GCSettings.IsServerGC`). In containers, heap hard limit follows the cgroup limit (default 75%). Background GC only applies to gen2; gen0/gen1 are always blocking. Turning concurrent GC off saves a thread and some memory but increases gen2 pauses. `GCSettings.LatencyMode` (`SustainedLowLatency`, `LowLatency` workstation only) and `GC.TryStartNoGCRegion(totalSize)` give temporary control.

**Code**
```csharp
using System.Runtime;
Console.WriteLine(GCSettings.IsServerGC);   // depends on config
Console.WriteLine(GCSettings.LatencyMode);  // Interactive (concurrent) by default
if (GC.TryStartNoGCRegion(10_000_000))      // no GC while allocating <= 10MB
{
    try { /* latency-critical work */ }
    finally { if (GCSettings.LatencyMode == GCLatencyMode.NoGCRegion) GC.EndNoGCRegion(); }
}
```

### 1.4 [Hard] Should you call GC.Collect()? Debunk the common myths.
**Answer**
Almost never. The GC tunes itself from allocation budgets and survival rates; forcing a collection promotes live objects early, wastes time and can *increase* later pauses. Legit uses: tests/benchmarks, after a one-off huge release (e.g. unloading a big dataset), or around `TryStartNoGCRegion`.

**Explanation**
Myths: (1) "Set to null to free memory" - the JIT already treats locals as dead after last use in optimized code; nulling matters only for long-lived fields. (2) "Dispose frees managed memory" - no, `Dispose` releases resources; the GC frees memory. (3) "GC.Collect fixes leaks" - a leak means something still references the object. (4) "Finalizers run promptly" - they run on the finalizer thread at an unspecified time, if ever at process exit (.NET Core doesn't run them on exit). (5) "Memory in Task Manager = live managed heap" - the GC retains committed memory for reuse; use `GC.GetTotalMemory(false)` / `GC.GetGCMemoryInfo()` / dotnet-counters. `GC.Collect(2, GCCollectionMode.Optimized)` lets the GC decide; `Aggressive` (.NET 7+) also compacts LOH.

**Code**
```csharp
Console.WriteLine(GC.GetTotalMemory(forceFullCollection: false) > 0); // Output: True
GCMemoryInfo info = GC.GetGCMemoryInfo();
Console.WriteLine(info.HeapSizeBytes >= 0);                          // Output: True
GC.Collect(2, GCCollectionMode.Optimized, blocking: false);          // hint, not command
Console.WriteLine(GC.GetAllocatedBytesForCurrentThread() > 0);       // Output: True
```

**Gotchas / follow-ups**
- `GC.Collect()` in a hot path (e.g. per request) is a classic perf bug. 
- `GC.AddMemoryPressure` is for unmanaged memory owned by managed wrappers; prefer `SafeHandle`/`NativeMemory` + `Dispose`.
- Follow-up: how would you diagnose high GC time? `% Time in GC` counter, `dotnet-counters`, `dotnet-gcdump`, PerfView, allocation profiling.

### 1.5 [Tricky] An object is still being used by a method, yet it gets collected. How? What do finalizable objects cost?
**Answer**
In Release builds the JIT considers a local dead after its last use, so an object whose only reference is a local can be collected (and finalized) while its method is still running, e.g. while native code uses its raw handle. `GC.KeepAlive(obj)` extends the lifetime. Finalizable objects also survive one extra GC because they go onto the f-reachable queue.

**Explanation**
Lifetime = reachability, not scope. A finalizable object allocated in gen0 is registered on the finalization list; when found unreachable it moves to the f-reachable queue (resurrected until the finalizer thread runs it), so it is promoted to gen1/2 and freed only on a later GC. Hence finalizers are expensive; use `SafeHandle` and `GC.SuppressFinalize` in `Dispose`. In Debug builds locals live to end of scope, so the bug appears only in Release.

**Code**
```csharp
class Holder { ~Holder() => Console.WriteLine("finalized"); public IntPtr Raw = 1; }

static void Use()
{
    var h = new Holder();
    IntPtr raw = h.Raw;
    // h is dead here in Release; a GC could finalize it while "raw" is still in use.
    Native(raw);
    GC.KeepAlive(h); // keeps h reachable up to this point
}
static void Native(IntPtr p) { }
```

**Interview traps**
- LOH threshold is 85,000 bytes (not 85 KB = 87,040) and applies to the object size, e.g. `double[10_624]` crosses it.
- "Gen2 = LOH" is wrong; LOH is *logically collected with* gen2 but is a separate region.
- `GC.Collect()` does not run finalizers synchronously; pair with `GC.WaitForPendingFinalizers()` only in tests.
- Don't assume Server vs Workstation GC for a given host; check config / `GCSettings.IsServerGC`.
- Value types are not "on the stack" - struct fields of a class live on the heap.

**Cheat sheet**
- Gen0 -> gen1 -> gen2; LOH >= 85,000 B (collected with gen2); POH for pinned.
- Mark -> plan -> relocate -> compact; card table tracks old->young refs.
- Workstation = latency/low memory; Server = throughput, heap per core.
- Background GC = concurrent gen2 only.
- Don't call GC.Collect; use pooling & streaming to cut LOH churn.
- Finalizers delay freeing by one extra GC; prefer SafeHandle.

---

## Topic 2: IDisposable, Finalizers, SafeHandle, using / await using

**Study notes**

- `IDisposable.Dispose()` deterministically releases **unmanaged or scarce resources** (files, sockets, DB connections, native memory, locks). GC reclaims memory only.
- **Canonical pattern** (needed only when the class owns unmanaged resources directly *or* is unsealed and may be subclassed):

```
Dispose()  -> Dispose(true);  GC.SuppressFinalize(this);
~Finalizer -> Dispose(false);
Dispose(bool disposing):
   if (disposed) return;
   if (disposing) { dispose managed members }   // not safe from finalizer
   free unmanaged resources; disposed = true;
```
- **Finalizer** (`~T()`): runs on the finalizer thread, order between objects is undefined, can run on a half-constructed object, last-resort safety net. In .NET Core not run at process exit.
- **SafeHandle** (`Microsoft.Win32.SafeHandles`): critical-finalizable wrapper over an `IntPtr`; ref-counted during P/Invoke so it can't be closed in use. Prefer it to raw `IntPtr` + finalizer.
- `using` statement/declaration compiles to `try/finally { Dispose() }`. `await using` calls `IAsyncDisposable.DisposeAsync()` (returns `ValueTask`). `using` works on `ref struct` with a pattern-based `Dispose()` too.
- Disposable **should be idempotent**; throwing `ObjectDisposedException` on use-after-dispose is conventional.

**Ladder**

### 2.1 [Easy] What does `using` do and when do you need IDisposable?
**Answer**
`using` guarantees `Dispose()` is called when the scope ends, even on exceptions, by compiling to try/finally. Implement `IDisposable` when your type owns resources the GC cannot release deterministically - handles, connections, streams or other disposables.

**Explanation**
A `using` declaration (`using var x = ...;`, C# 8) disposes at the end of the enclosing block, avoiding nesting. The compiler expands to a try/finally with a null check for reference types. Disposal order is reverse of declaration.

**Code**
```csharp
using var a = new Res("A");
using var b = new Res("B");
Console.WriteLine("work");
// Output: work
//         dispose B
//         dispose A

class Res(string name) : IDisposable
{
    public void Dispose() => Console.WriteLine($"dispose {name}");
}
```

### 2.2 [Follow-up] Show the full Dispose pattern. When is it overkill?
**Answer**
Use `Dispose(bool disposing)` plus a finalizer only when the type directly holds unmanaged resources. If you merely hold other `IDisposable` fields, implement a simple `Dispose()` and seal the class (or provide `protected virtual Dispose(bool)` for inheritance without a finalizer).

**Explanation**
`disposing == true` means called from `Dispose()` - safe to touch managed members. `false` means called from the finalizer - other managed objects may already be finalized so touch only unmanaged state. `GC.SuppressFinalize(this)` removes the object from the finalization queue, avoiding the extra-GC cost. Today the recommendation is: wrap native resources in a `SafeHandle` so *your* type needs no finalizer.

**Code**
```csharp
public class NativeBuffer : IDisposable
{
    private IntPtr _ptr = System.Runtime.InteropServices.Marshal.AllocHGlobal(1024);
    private bool _disposed;

    public void Dispose() { Dispose(true); GC.SuppressFinalize(this); }

    protected virtual void Dispose(bool disposing)
    {
        if (_disposed) return;
        if (disposing) { /* dispose managed fields here */ }
        System.Runtime.InteropServices.Marshal.FreeHGlobal(_ptr);
        _ptr = IntPtr.Zero;
        _disposed = true;
    }
    ~NativeBuffer() => Dispose(false);
}
```

### 2.3 [Medium] IAsyncDisposable and `await using`: how do they work, and how do you implement both?
**Answer**
`IAsyncDisposable.DisposeAsync()` returns a `ValueTask` so cleanup can await I/O (flushing a stream, closing a connection). `await using` calls it at scope end. A type that supports both sync and async cleanup implements both and often shares logic via a `DisposeAsyncCore()` pattern.

**Explanation**
`await using var x` -> try/finally with `await x.DisposeAsync()`. `ConfigureAwait` is available via `await using var x = ...; ` -> `await using (x.ConfigureAwait(false))`. DI containers (`IServiceProvider` scopes) dispose services implementing `IAsyncDisposable` via `DisposeAsync` if you use `await using` on the scope; disposing the container synchronously when a service only implements `IAsyncDisposable` throws `InvalidOperationException`.

**Code**
```csharp
public sealed class Writer : IAsyncDisposable, IDisposable
{
    private readonly FileStream _fs = new("out.txt", FileMode.Create);
    public async ValueTask DisposeAsync()
    {
        await _fs.DisposeAsync().ConfigureAwait(false);
        GC.SuppressFinalize(this);
    }
    public void Dispose() => _fs.Dispose();
}
// usage
await using var w = new Writer();
```

### 2.4 [Hard] Why prefer SafeHandle over IntPtr + finalizer? What does it guarantee?
**Answer**
`SafeHandle` is a `CriticalFinalizerObject` whose finalizer is guaranteed to run (the runtime pre-prepares it), and the marshaller ref-counts it during P/Invoke calls so the handle can't be released while native code uses it. It also protects against handle recycling attacks and races between `Dispose` and use.

**Explanation**
You derive from `SafeHandleZeroOrMinusOneIsInvalid` and override `ReleaseHandle()`. P/Invoke signatures take the `SafeHandle` type directly; the marshaller calls `DangerousAddRef`/`DangerousRelease` around the call. `Dispose()` on an in-use handle defers the actual release until the count drops to zero. With raw `IntPtr`, a GC during the native call may finalize your wrapper and close the handle under you (see 1.5).

**Code**
```csharp
using Microsoft.Win32.SafeHandles;
using System.Runtime.InteropServices;

sealed class MemHandle : SafeHandleZeroOrMinusOneIsInvalid
{
    public MemHandle() : base(ownsHandle: true) =>
        SetHandle(Marshal.AllocHGlobal(256));
    protected override bool ReleaseHandle()
    {
        Marshal.FreeHGlobal(handle);
        return true;
    }
}
// usage
using var h = new MemHandle();
Console.WriteLine(h.IsInvalid); // Output: False
```

**Gotchas / follow-ups**
- `ReleaseHandle` must not throw/allocate/take locks - it runs in a constrained context.
- Follow-up: `SafeFileHandle` backs `FileStream`; `File.OpenHandle` returns one for `RandomAccess` APIs.

### 2.5 [Tricky] Edge cases: disposing twice, exceptions in Dispose, HttpClient and DI-owned objects, `using` on a struct.
**Answer**
Dispose must be idempotent and should not throw. An exception in `Dispose` inside `finally` replaces the original exception. Don't dispose things you don't own (DI-injected services, shared `HttpClient`). Using a mutable struct in `using` is risky because the variable is readonly, so `Dispose` mutations act on a copy.

**Explanation**
1. Exception masking: if the body throws A and `Dispose` throws B, caller sees B. 2. `HttpClient` should be long-lived or from `IHttpClientFactory`; `using (new HttpClient())` per call exhausts sockets (TIME_WAIT). 3. The container owns objects it created; disposing them yourself leads to `ObjectDisposedException` for the next consumer. Objects you *new up* and register as instances are not disposed by the container. 4. `using` on a `struct` variable: compiler works on the readonly resource variable; for non-readonly struct, a defensive copy semantic can make state changes invisible - prefer `readonly struct` or `ref struct` disposables.

**Code**
```csharp
try
{
    using var r = new Bad();
    throw new InvalidOperationException("body");
}
catch (Exception ex)
{
    Console.WriteLine(ex.Message); // Output: dispose  (original "body" is lost)
}

class Bad : IDisposable
{
    public void Dispose() => throw new Exception("dispose");
}
```

**Interview traps**
- "Dispose frees memory" - no; it frees resources. "Finalizer = destructor" - it is not deterministic.
- Accessing managed fields in `Dispose(false)` is a bug.
- A finalizer on a class with no unmanaged fields is pure cost.
- `using (var x = ...)` with `async` code: use `await using` for `IAsyncDisposable`, else you may block or skip async cleanup.
- DI: transient `IDisposable`s resolved from the root provider are held until app end (leak).

**Cheat sheet**
- `using` = try/finally Dispose; `await using` = DisposeAsync.
- Full pattern only for unmanaged owners/unsealed bases; otherwise seal + simple Dispose.
- `SuppressFinalize` in Dispose; finalizer runs `Dispose(false)`.
- Use SafeHandle instead of IntPtr + finalizer.
- Dispose must be idempotent and non-throwing.

---

## Topic 3: Memory Leaks in Managed Code

**Study notes**

- A managed "leak" = objects that are **still reachable** (so GC can't free them) but no longer needed. GC can't tell intent.
- **Top causes**
  1. **Event handlers**: publisher holds a delegate -> subscriber. Long-lived publisher (static event, singleton) keeps subscribers alive. Fix: unsubscribe, weak events, or short-lived publisher.
  2. **Static fields / static collections / caches** with no eviction (`static Dictionary`, `MemoryCache` without size limit).
  3. **Closures / lambdas** capturing `this` or large locals; the closure object lives as long as the delegate.
  4. **Timers**: `System.Threading.Timer` / `System.Timers.Timer` callbacks root their target while running; forgetting `Dispose`.
  5. **Undisposed IDisposable** (native memory, `CancellationTokenSource` linked tokens, `HttpClient` per request).
  6. **Unbounded** queues/channels/`List` growth; `Task`s never completed (`TaskCompletionSource` never set); thread-static/`ThreadLocal`; `ConditionalWeakTable` misuse; DI captive dependencies.
  7. **Finalizer thread blocked** -> finalizable objects accumulate.
- **Diagnose**: `dotnet-counters` (gc-heap-size, gen2 size), `dotnet-gcdump collect` + compare, `dotnet-dump analyze` (`dumpheap -stat`, `gcroot`), Visual Studio memory profiler, PerfView. Look for a type growing between snapshots, then trace the *GC root path*.

```
 static Publisher --event--> delegate --Target--> Subscriber (never freed)
                                                   \--> big graph
```

**Ladder**

### 3.1 [Easy] Can managed code leak memory? Give examples.
**Answer**
Yes - logically. If objects remain reachable from a root (static, long-lived object, running thread, timer), GC keeps them. Typical: event subscriptions never removed, static caches that only grow, and undisposed resources.

**Explanation**
The GC only frees unreachable objects. "Root" categories: stack locals, statics, GC handles (pinned/strong), the finalization queue, and active timers/threads. Memory growth with a stable workload that persists across gen2 GCs indicates a leak (the gen2 size trend is the test).

**Code**
```csharp
static class Cache
{
    public static readonly Dictionary<string, byte[]> Items = new();
}
// Every call adds 1MB that is never removed:
Cache.Items[Guid.NewGuid().ToString()] = new byte[1_000_000];
```

### 3.2 [Follow-up] How do events cause leaks and how do you fix them?
**Answer**
`publisher.Event += subscriber.Handler` stores a delegate that references the subscriber. If the publisher outlives the subscriber and you never `-=`, the subscriber stays reachable. Fix: unsubscribe (often in `Dispose`), use weak-event patterns, or make the publisher's lifetime shorter.

**Explanation**
The reverse is not a problem: subscriber referencing a short-lived publisher is fine. In UI frameworks, static `Messenger`/`EventAggregator` are common culprits; use weak references (`WeakReferenceMessenger` in CommunityToolkit.Mvvm). Anonymous lambdas can't be unsubscribed unless stored in a variable.

**Code**
```csharp
class Publisher { public event EventHandler? Changed; public void Raise() => Changed?.Invoke(this, EventArgs.Empty); }

sealed class Subscriber : IDisposable
{
    private readonly Publisher _p;
    public Subscriber(Publisher p) { _p = p; _p.Changed += OnChanged; }
    private void OnChanged(object? s, EventArgs e) => Console.WriteLine("changed");
    public void Dispose() => _p.Changed -= OnChanged; // breaks the reference
}
```

### 3.3 [Medium] How can closures, timers and `static` lambdas leak or help?
**Answer**
A lambda that uses instance members or locals compiles into a closure object that captures them; if the delegate is stored somewhere long-lived, everything captured lives on. Timers root their callback target until disposed. Marking a lambda `static` prevents accidental capture and avoids allocations.

**Explanation**
Compiler generates a display class for captured locals (shared by all lambdas in that scope - capturing one small variable can keep a big one alive if in the same scope). `System.Threading.Timer` is kept alive by the runtime's timer queue while active even if you drop the reference; a callback targeting `this` pins `this`. Use `await using`/`Dispose` on timers, or `PeriodicTimer` in async loops. A `static` lambda (C# 9) can't capture, and non-capturing lambdas are cached so allocate once.

**Code**
```csharp
class Poller : IDisposable
{
    private readonly Timer _timer;
    public Poller() => _timer = new Timer(_ => Tick(), null, 0, 1000); // captures this
    void Tick() { }
    public void Dispose() => _timer.Dispose(); // without this, Poller is never collected
}

Func<int, int> f = static x => x * 2; // cannot capture; zero closure
Console.WriteLine(f(21)); // Output: 42
```

### 3.4 [Hard] Walk through diagnosing a suspected leak in production.
**Answer**
Confirm with counters that gen2/LOH size keeps rising after full GCs, then take two `dotnet-gcdump` (or dump) snapshots under load, diff by type, and find a GC root path for the growing type with `gcroot`/VS/PerfView. Fix the root, redeploy, and verify the trend flattens.

**Explanation**
Steps: 1) `dotnet-counters monitor -p <pid> System.Runtime` - watch `gc-heap-size`, `gen-2-size`, `loh-size`, `gc-fragmentation`. 2) `dotnet-gcdump collect -p <pid>` twice, compare in VS or PerfView (type count/size deltas). 3) For deep inspection: `dotnet-dump collect` -> `dotnet-dump analyze` -> `dumpheap -stat`, `dumpheap -type X`, `gcroot <addr>`. 4) Distinguish from *fragmentation*, *native memory* (check `Process.PrivateMemorySize64` vs `GC.GetTotalMemory`) and *container limit with Server GC* growth. Native leaks need different tools (dotnet-trace allocation, WinDbg `!address`, heap profilers).

**Code**
```csharp
// Quick in-app diagnostic probe
long managed = GC.GetTotalMemory(forceFullCollection: true);
long working = System.Diagnostics.Process.GetCurrentProcess().WorkingSet64;
Console.WriteLine($"managed={managed / 1024 / 1024}MB working={working / 1024 / 1024}MB");
Console.WriteLine($"gen2 collections={GC.CollectionCount(2)}");
```

**Gotchas / follow-ups**
- High memory != leak: the GC deliberately keeps spare commit; check *after gen2*.
- `forceFullCollection: true` is for diagnostics only.

### 3.5 [Tricky] A class unsubscribes correctly, yet instances still aren't collected. Name hidden roots.
**Answer**
Look beyond events: `Task`s that never complete (awaiting a `TaskCompletionSource` nobody sets), `CancellationToken.Register` callbacks not disposed, `ThreadStatic`/`AsyncLocal` values, `ConditionalWeakTable`, DI singletons capturing scoped services, `static` caches such as `MemoryCache` without limits, and ETW/`DiagnosticListener` subscriptions.

**Explanation**
`CancellationTokenSource` with long life: each `token.Register(...)` adds to its callback list; not disposing the `CancellationTokenRegistration` leaks per call. `Task` continuations referencing a state machine keep locals alive until the awaited task completes. The "captive dependency": singleton -> scoped DbContext keeps a change tracker growing (EF Core: tracked entities grow memory unless `AsNoTracking` or short-lived contexts). `MemoryCache` has no size limit by default - set `SizeLimit` and entry `Size`.

**Code**
```csharp
var cts = new CancellationTokenSource(); // long-lived
void Work()
{
    // Each call leaks a registration unless disposed:
    using var reg = cts.Token.Register(() => Console.WriteLine("cancelled"));
    // ... work ...
} // registration removed here
Work();
```

**Interview traps**
- "Static = leak" is too broad; unbounded static *collections* leak.
- Calling `GC.Collect()` to "prove" no leak - objects still rooted survive.
- Weak event handlers need the target to be rooted elsewhere, else the handler silently stops firing.
- A `Timer` can keep an object alive even with no variable referencing it.
- EF `DbContext` registered as singleton = tracked entity leak + thread-safety bug.

**Cheat sheet**
- Leak = still reachable; find the root path.
- Events: always `-=` or use weak events.
- Dispose timers, CTS registrations, linked CTS.
- Tools: dotnet-counters, dotnet-gcdump, dotnet-dump (`gcroot`), PerfView.
- Compare after gen2, not working set.
- Bound every cache/queue.

---
## Topic 4: Span<T>, Memory<T>, ReadOnlySpan<T>, stackalloc

**Study notes**

- `Span<T>` is a **`ref struct`**: a (managed reference/pointer, length) pair giving a safe, bounds-checked window over contiguous memory: arrays, slices, `stackalloc`, native memory, strings (`ReadOnlySpan<char>`). Slicing is O(1) and allocation-free.
- Because it may point to the stack, a `ref struct` **cannot**: be a field of a class or non-ref struct, be boxed, implement interfaces (pre-C# 13 `allows ref struct` generics), be captured in lambdas/local functions, or live across `await`/`yield` boundaries.
- `Memory<T>` / `ReadOnlyMemory<T>` are normal structs (heap-friendly) that wrap an array/string/`MemoryManager<T>`; get a `Span` via `.Span` when you need to work on it. Use them for async APIs (`Stream.ReadAsync(Memory<byte>)`).
- `stackalloc` allocates on the stack frame, freed on method return; safe when assigned to `Span<T>` (`Span<byte> b = stackalloc byte[256];`). Keep small (< ~1 KB typical; stack is ~1 MB main thread/ 8MB Linux) and **never in loops without bound**.
- Compare: `ReadOnlySpan<char>` APIs: `Trim`, `IndexOf`, `Split` (.NET 8 `MemoryExtensions.Split`/`EnumerateLines`), `int.Parse(ReadOnlySpan<char>)`, `MemoryExtensions.SequenceEqual`.

```
 string/array/stack/native memory
        [ 0 1 2 3 4 5 6 7 ]
              ^------^        <- span.Slice(2,4): pointer+len, no copy
```

**Ladder**

### 4.1 [Easy] What is `Span<T>` and what problem does it solve?
**Answer**
It's a stack-only view over contiguous memory that lets you slice and process arrays, strings or stack memory without copying or allocating. It replaces patterns like `Substring`/`Array.Copy` in hot paths.

**Explanation**
`s.Substring(3, 4)` allocates a new string; `s.AsSpan(3, 4)` is just pointer arithmetic. Because it is a ref struct, the compiler/GC enforce that it can't outlive the memory it points to (escape analysis, "ref safety rules").

**Code**
```csharp
string text = "id=42;name=bob";
ReadOnlySpan<char> span = text.AsSpan();
int eq = span.IndexOf('=');
int semi = span.IndexOf(';');
int id = int.Parse(span.Slice(eq + 1, semi - eq - 1)); // no substring allocation
Console.WriteLine(id); // Output: 42

int[] arr = { 1, 2, 3, 4, 5 };
Span<int> mid = arr.AsSpan(1, 3);
mid[0] = 99;
Console.WriteLine(arr[1]); // Output: 99  (span is a view, not a copy)
```

### 4.2 [Follow-up] Span vs Memory: when do you use each?
**Answer**
Use `Span<T>` for synchronous, stack-bound processing. Use `Memory<T>` when the buffer must be stored in a field, passed across `await`, or captured, then call `.Span` inside the synchronous section. `Memory<T>` is a regular struct so it can live on the heap.

**Explanation**
Async methods compile to state machines stored on the heap, which can't contain ref structs across suspension. Typical pattern: `async` method takes `Memory<byte>`, awaits I/O, then inside a sync helper uses `memory.Span`. `Memory<T>` can come from an array, `string.AsMemory()`, or `IMemoryOwner<T>` (`MemoryPool<T>.Shared.Rent`) which is `IDisposable`.

**Code**
```csharp
static async Task<int> ReadSomeAsync(Stream s, Memory<byte> buffer)
{
    int n = await s.ReadAsync(buffer); // Memory<byte> overload, no byte[] needed
    Parse(buffer.Span[..n]);           // back to Span for sync work
    return n;
}
static void Parse(ReadOnlySpan<byte> data) { /* ... */ }

using var ms = new MemoryStream(new byte[] { 1, 2, 3 });
Console.WriteLine(await ReadSomeAsync(ms, new byte[16])); // Output: 3
```

### 4.3 [Medium] How does stackalloc work and what's the safe pattern for variable sizes?
**Answer**
`stackalloc` reserves memory in the current stack frame (zero-initialised by default unless `[SkipLocalsInit]` is applied) and frees it automatically at return. The safe idiom is a size threshold: stackalloc for small, `ArrayPool` for large, avoiding stack overflow, which is unrecoverable.

**Explanation**
Stack overflow kills the process (no catch). Never `stackalloc` inside a loop (memory isn't released until method exit) or with user-controlled sizes unchecked. The result can be assigned to `Span<T>` without `unsafe`. In C# 12 you can use inline arrays `[InlineArray(N)]` as a safe alternative for fixed small buffers.

**Code**
```csharp
using System.Buffers;

static string Hex(ReadOnlySpan<byte> data)
{
    int len = data.Length * 2;
    char[]? rented = null;
    Span<char> buf = len <= 256
        ? stackalloc char[256]
        : (rented = ArrayPool<char>.Shared.Rent(len));
    try
    {
        for (int i = 0; i < data.Length; i++)
            data[i].TryFormat(buf.Slice(i * 2, 2), out _, "x2");
        return new string(buf[..len]);
    }
    finally { if (rented is not null) ArrayPool<char>.Shared.Return(rented); }
}
Console.WriteLine(Hex(new byte[] { 0xCA, 0xFE })); // Output: cafe
```

### 4.4 [Hard] Why can't Span be used in async methods or lambdas, and what changed in C# 13?
**Answer**
Spans may reference the stack, but async state machines and closures are heap objects that outlive the frame; allowing them would create dangling references. C# 13 relaxes this: async methods/iterators can *declare* ref-struct locals as long as they are not live across an `await`/`yield`; and `allows ref struct` anti-constraint lets generics accept ref structs.

**Explanation**
Workarounds pre-13: split into a sync helper method taking the span, or use `Memory<T>`. `allows ref struct` (C# 13/.NET 9) enables `Span<T>` as a type argument for generic APIs, and ref structs can implement interfaces (though they can't be boxed to them). Also new in C# 13: `params ReadOnlySpan<T>` (see Topic 13).

**Code**
```csharp
static async Task DemoAsync()
{
    await Task.Yield();
    Process(new byte[] { 1, 2, 3 });   // fine: span created inside sync method

    // Span<byte> s = stackalloc byte[4];
    // await Task.Delay(1);            // error CS4007 (before C# 13 any such local); C# 13 allows
    //                                 // declaring it if not used after the await
    // Console.WriteLine(s.Length);    // using it after await is still an error
}
static void Process(ReadOnlySpan<byte> b) => Console.WriteLine(b.Length);
await DemoAsync(); // Output: 3
```

**Gotchas / follow-ups**
- `foreach` over a span is fine (ref struct enumerator); LINQ is not available on Span.
- Cannot `return` a span that points to a local `stackalloc` (compile error CS8352).

### 4.5 [Tricky] Edge cases: array covariance, empty/default spans, string mutation.
**Answer**
Creating a `Span<T>` over an array with covariance mismatch throws `ArrayTypeMismatchException` at construction. `default(Span<T>)` is a valid empty span (null reference, length 0). Strings are immutable but `MemoryMarshal.AsMemory`/unsafe tricks can mutate them - never do it.

**Explanation**
`string[] a = ...; object[] o = a; Span<object> s = o;` throws because writes through `Span<object>` could store a non-string. `ReadOnlySpan<object>` has no such check since it can't write. `ReadOnlySpan<byte> x = new byte[] {1,2,3};` is compiled to static data - no allocation (supported for byte-sized primitives and, in newer compilers, other primitives). `Span.Equals` is not content equality; use `SequenceEqual`.

**Code**
```csharp
string[] strings = { "a", "b" };
object[] objs = strings;
try { Span<object> s = objs; }
catch (ArrayTypeMismatchException) { Console.WriteLine("mismatch"); } // Output: mismatch

Span<int> empty = default;
Console.WriteLine(empty.IsEmpty); // Output: True

ReadOnlySpan<byte> table = new byte[] { 1, 2, 3 }; // no heap allocation
Console.WriteLine(table.SequenceEqual(new byte[] { 1, 2, 3 })); // Output: True
```

**Interview traps**
- "Span is on the stack so it's always faster" - it avoids allocation/copy, but bounds checks still apply.
- `Memory<T>.Span` is cheap but not free; hoist it out of loops.
- Spans over pooled arrays: after `Return`, the span is a use-after-free bug.
- `stackalloc` + `Span` in a loop leaks stack until method end.
- A span over a `string` is read-only; `Span<char>` over a string requires unsafe hacks.

**Cheat sheet**
- Span = ref struct view; Memory = heap-safe view; `.Span` to process.
- No async/lambda/field storage (C# 13 relaxes locals).
- stackalloc only small + bounded; else ArrayPool.
- `AsSpan` / `Slice` / `[a..b]` = no copy.
- Use `SequenceEqual`, not `==`, to compare contents.

---

## Topic 5: ArrayPool and ObjectPool

**Study notes**

- **`ArrayPool<T>.Shared`** (System.Buffers): thread-safe pool of arrays bucketed by power-of-two sizes. `Rent(minLen)` returns an array of **at least** that length (often larger), with **undefined contents**; `Return(arr, clearArray: false)` gives it back. Rent beyond max pooled size (1 MB-ish; `ArrayPool<T>.Create(max, perBucket)` to customise) -> allocates normally and is dropped on return.
- **`ObjectPool<T>`** (`Microsoft.Extensions.ObjectPool` NuGet; included in ASP.NET Core): `Get()`/`Return(T)` with an `IPooledObjectPolicy<T>` (`Create()` and `Return(T) -> bool` to reset & decide whether to keep). Default pool retains `Environment.ProcessorCount * 2` items. Built-in `StringBuilder` pool policy.
- `MemoryPool<T>.Shared` returns `IMemoryOwner<T>` (disposable wrapper around ArrayPool).
- Why pool: avoid **LOH allocations & gen2 GCs** for big/short-lived buffers; reduce allocation rate. Trade-offs: bugs from use-after-return, stale data, retention (memory held), complexity. Measure first.

**Ladder**

### 5.1 [Easy] What does ArrayPool do and how do you use it correctly?
**Answer**
It reuses arrays to avoid repeatedly allocating large buffers. You `Rent`, use only the length you asked for (not `array.Length`), and `Return` in a `finally`, never touching the array afterwards.

**Explanation**
Rented arrays can be bigger than requested and contain old data. Track your own logical length. Returning the same array twice or using it after return can corrupt another consumer's data.

**Code**
```csharp
using System.Buffers;
byte[] buf = ArrayPool<byte>.Shared.Rent(1000);
try
{
    Console.WriteLine(buf.Length >= 1000); // Output: True  (probably 1024)
    // use buf.AsSpan(0, 1000) only
}
finally
{
    ArrayPool<byte>.Shared.Return(buf, clearArray: true); // clear if it held secrets
}
```

### 5.2 [Follow-up] When should `clearArray` be true? What happens if you forget to Return?
**Answer**
Set `clearArray: true` when the buffer held sensitive data or reference types you want released (otherwise stale data/objects stay reachable). Forgetting to `Return` isn't a leak - the array is simply GC'd normally, you just lose the pooling benefit.

**Explanation**
For `T` containing references (e.g. `string[]`), unreturned-clear means pooled array keeps references alive -> hidden retention. Also, the pool doesn't validate the array is yours: returning an array not rented is accepted (and could be re-rented - another reason to be careful with slices/`ToArray`).

**Code**
```csharp
string[] names = ArrayPool<string>.Shared.Rent(8);
names[0] = "secret";
ArrayPool<string>.Shared.Return(names, clearArray: true); // drops the "secret" reference
```

### 5.3 [Medium] How do you use ObjectPool<T> with a custom policy, and how does it differ from ArrayPool?
**Answer**
`ObjectPool<T>` pools arbitrary reusable objects (StringBuilders, parsers, buffers wrapped in classes) via a policy that creates and resets them. Unlike ArrayPool it has no size buckets; you decide object identity and reset behaviour in `Return`.

**Explanation**
Create through `DefaultObjectPoolProvider`. `Return(T)` in the policy should reset state and return `false` to discard (e.g. too large). In ASP.NET Core inject `ObjectPoolProvider` from DI (`services.AddSingleton<ObjectPoolProvider, DefaultObjectPoolProvider>()` is registered by default). Don't pool objects that are cheap to allocate - pooling adds contention and prevents gen0 free-lunch.

**Code**
```csharp
// dotnet add package Microsoft.Extensions.ObjectPool
using Microsoft.Extensions.ObjectPool;
using System.Text;

var provider = new DefaultObjectPoolProvider { MaximumRetained = 16 };
ObjectPool<StringBuilder> pool = provider.CreateStringBuilderPool();

StringBuilder sb = pool.Get();
try { sb.Append("hello ").Append("pool"); Console.WriteLine(sb); } // Output: hello pool
finally { pool.Return(sb); } // policy clears it (and drops it if capacity is too large)
```

### 5.4 [Hard] What are the failure modes of pooling and how would you decide whether to pool?
**Answer**
Failure modes: use-after-return (data corruption across requests), double return, state leakage from incomplete reset, memory retention, and sometimes slower code than plain allocation. Decide by measuring allocation rate/GC pause with BenchmarkDotNet and a profiler; pool only large (LOH) or hot-path objects.

**Explanation**
Gen0 allocation is extremely cheap; pooled objects may be promoted to gen2 and then write-barriers/card marking make them *more* expensive to reference young objects (old->young). `ArrayPool<T>.Shared` per-core buckets reduce contention. Security: pooled buffers hold previous request data - clear them in auth/crypto paths. Alternatives: `stackalloc` for small, `ArrayBufferWriter<T>`/`IBufferWriter<T>`, `System.IO.Pipelines`, `RecyclableMemoryStream` (Microsoft.IO).

**Code**
```csharp
using System.Buffers;
// Bug: slice escapes after Return
byte[] a = ArrayPool<byte>.Shared.Rent(16);
Memory<byte> view = a.AsMemory(0, 4);
ArrayPool<byte>.Shared.Return(a);
byte[] b = ArrayPool<byte>.Shared.Rent(16); // likely the same array
view.Span[0] = 7;                           // silently corrupts 'b' users!
Console.WriteLine(ReferenceEquals(a, b));   // typically True
```

**Gotchas / follow-ups**
- `IMemoryOwner<T>.Dispose()` returns to the pool; owner semantics should be documented (who disposes?).
- Follow-up: why not pool every `List<T>`? Capacity retention and cleaning cost.

**Interview traps**
- `Rent(n).Length == n` is false in general.
- Not clearing a pooled array of references retains objects.
- Pools are not free: measure; a pooled object promoted to gen2 can hurt GC.
- ArrayPool never zeroes on Rent (security).

**Cheat sheet**
- Rent >= size, track your own length, Return in finally.
- `clearArray: true` for secrets / reference arrays.
- ObjectPool: policy Create + Return(reset, bool).
- Pool only where profiling shows big/hot allocations.
- Never touch a buffer (or slice of it) after Return.

---

## Topic 6: Struct vs Class Performance & Defensive Copies

**Study notes**

- **class**: reference type, heap allocated, GC tracked, identity semantics, 8/16-byte object header + MethodTable pointer overhead. **struct**: value type, copied on assignment/pass, stored inline (stack, inside arrays, inside objects), no header.
- Structs avoid allocation & improve cache locality (`Point[]` is contiguous) but large structs (> ~16 bytes guideline) cost copy time. Guidelines: small, immutable, logically a single value, short-lived, not frequently boxed.
- **Boxing**: converting struct -> `object`/interface allocates; unboxing copies. Generics with constraints (`where T : struct, IEquatable<T>`) avoid boxing.
- **Defensive copy**: when calling a member on a struct reached via a `readonly` context (readonly field, `in` parameter, `ref readonly`, readonly property result), the compiler copies the struct first to guarantee that the method can't mutate it. If the struct is `readonly struct`, the compiler knows no mutation is possible and skips the copy.
- `readonly struct`, `readonly` members (C# 8), `ref struct`, `record struct` (C# 10).
- Equality: `ValueType.Equals` default can use reflection/box; always implement `IEquatable<T>` + `GetHashCode` (or use `record struct`).

**Ladder**

### 6.1 [Easy] Struct vs class: key differences and when to choose which?
**Answer**
Classes are reference types allocated on the heap and shared by reference; structs are value types copied on assignment, stored inline. Use a struct for small, immutable, short-lived value-like data (points, money, ids); otherwise a class.

**Explanation**
"Structs live on the stack" is simplified: they live wherever their container does - stack for locals, heap when fields of a class or elements of an array. Structs can't inherit (only implement interfaces), have no finalizers, and `default` is always valid (zeroed) - design so zero is a sane state.

**Code**
```csharp
var p1 = new PointS { X = 1 };
var p2 = p1; p2.X = 99;
Console.WriteLine(p1.X); // Output: 1  (copied)

var c1 = new PointC { X = 1 };
var c2 = c1; c2.X = 99;
Console.WriteLine(c1.X); // Output: 99 (same object)

struct PointS { public int X; }
class PointC { public int X; }
```

### 6.2 [Follow-up] What is boxing and how do you avoid it?
**Answer**
Boxing wraps a value type in a heap object when treated as `object` or an interface; it allocates and copying occurs on unbox. Avoid by using generics (`List<int>`, constrained generics), `IEquatable<T>` overloads, and avoiding non-generic APIs like `ArrayList` or `string.Format(object)`.

**Explanation**
Common hidden boxing: `string.Format("{0}", 5)` (before interpolation handlers), calling `Equals(object)`, using a struct as `Dictionary` key without `IEquatable<T>` (comparer falls back), enum `.ToString()`/ `HasFlag` in old frameworks, passing a struct to `IComparable` param un-generically, capturing in lambdas. Calling an interface method on a *constrained generic* `T` does not box (constrained call).

**Code**
```csharp
int i = 5;
object o = i;           // box
i = 6;
Console.WriteLine((int)o); // Output: 5  (box holds a copy)

static bool Same<T>(T a, T b) where T : IEquatable<T> => a.Equals(b); // no boxing
Console.WriteLine(Same(3, 3)); // Output: True
```

### 6.3 [Medium] What is a defensive copy? Show how it silently changes behaviour.
**Answer**
When you call a method on a struct held in a readonly location, the compiler operates on a temporary copy so the readonly guarantee holds. Mutating methods therefore appear to do nothing, and non-readonly-struct calls cost an extra copy.

**Explanation**
Readonly locations: `readonly` fields, `in` parameters, `foreach` iteration variables, `using` variables, results of readonly properties, `this` in readonly structs. Fix: make the struct `readonly struct` (then members can't mutate; compiler elides copies), mark non-mutating members `readonly`, or avoid mutable structs. `readonly` members help the same way.

**Code**
```csharp
struct Counter { public int N; public void Inc() => N++; }
class Holder { public readonly Counter C = new(); }

var h = new Holder();
h.C.Inc();
Console.WriteLine(h.C.N); // Output: 0  (Inc ran on a defensive copy)

var list = new List<Counter> { new() };
list[0].Inc();            // indexer returns a copy
Console.WriteLine(list[0].N); // Output: 0
```

### 6.4 [Hard] `in` parameters, `readonly struct`, `ref readonly`: how do they interplay for performance?
**Answer**
Passing large structs by value copies them; `in` passes by readonly reference (no copy) but if the struct isn't `readonly struct` calls on it make defensive copies, erasing the gain. So pair `in` with `readonly struct` for large structs; for small (<= 16 B) structs pass by value.

**Explanation**
`in` args at the call site can be implicit (compiler may create a temp if you pass an rvalue/ property). `ref readonly` returns/locals expose references without copying (e.g. `ref readonly T this[int]` in `ImmutableArray`/ `Span`'s indexer). Struct layout: `[StructLayout]` default Sequential for structs; field ordering affects padding - 8-byte long after byte pads 7 bytes. `Unsafe.SizeOf<T>()` reports size. Also `Equals` override recommended; `record struct` generates `IEquatable<T>`, `==`, `GetHashCode`, `ToString`.

**Code**
```csharp
readonly struct Matrix4
{
    public readonly double A, B, C, D, E, F, G, H;
    public Matrix4(double a) { A = B = C = D = E = F = G = H = a; }
    public double Sum() => A + B + C + D + E + F + G + H; // no defensive copy when called via 'in'
}
static double Use(in Matrix4 m) => m.Sum();
Console.WriteLine(Use(new Matrix4(1))); // Output: 8
```

**Gotchas / follow-ups**
- Without `readonly` on the struct, `m.Sum()` on an `in` parameter copies 64 bytes each call.
- Follow-up: why avoid `ref` returns to list elements - `List<T>` can reallocate; use `CollectionsMarshal.AsSpan(list)`.

### 6.5 [Tricky] Struct equality & hashing pitfalls, and mutable-struct-in-collection bug.
**Answer**
Default `ValueType.Equals` compares fields bitwise only if the struct has no reference fields/padding concerns; otherwise it uses reflection - slow - and `GetHashCode` of the default may use only the first field, producing collisions. Always implement `IEquatable<T>` and `GetHashCode`. Mutating a struct stored in a `Dictionary` value requires reassigning it.

**Explanation**
`dict[key].Count++` fails to compile (CS1612) because the indexer returns a copy; use `CollectionsMarshal.GetValueRefOrNullRef(dict, key)` (.NET 6+) to mutate in place. Mutable struct as dictionary key: mutating after insertion breaks lookup. Floating-point struct equality: `double.NaN` handling differs between `==` and `Equals`.

**Code**
```csharp
using System.Runtime.InteropServices;

struct Stat { public int Count; }
var dict = new Dictionary<string, Stat> { ["a"] = new Stat() };

// dict["a"].Count++;   // error CS1612
ref Stat s = ref CollectionsMarshal.GetValueRefOrNullRef(dict, "a");
s.Count++;
Console.WriteLine(dict["a"].Count); // Output: 1
```

**Interview traps**
- "Structs are always faster" - big/boxed/mutable structs are slower.
- Struct fields of a class live on the heap; not "always the stack".
- `readonly` field + mutable struct = silent no-op mutations.
- `in` without `readonly struct` can be slower than by-value.
- Not overriding Equals/GetHashCode on struct keys hurts dictionary perf.

**Cheat sheet**
- Struct for <=16 B immutable values; class otherwise.
- Boxing allocates; use generics/constrained calls.
- Defensive copy: readonly context + non-readonly struct.
- `readonly struct` + `in` for large value types.
- Implement `IEquatable<T>` on structs (or `record struct`).

---

## Topic 7: String Performance

**Study notes**

- `string` is **immutable**, UTF-16, length-prefixed, heap-allocated. Every "modification" allocates.
- `a + b + c` in a single expression compiles to one `string.Concat(a,b,c)` call (efficient). `s += x` **in a loop** is O(n^2) in copied chars.
- **`StringBuilder`**: chunked mutable buffer; best for unknown/large number of appends; set `Capacity`. For few concatenations, `+`/interpolation is faster.
- **Interpolated strings (C# 10 / .NET 6)**: `$"..."` targeting `string` uses `DefaultInterpolatedStringHandler` (pooled char buffer, no boxing of value types, `ISpanFormattable`). Custom handlers via `[InterpolatedStringHandler]` let APIs (logging, `Debug.Assert`, `StringBuilder.Append($"...")`) skip formatting when disabled.
- **`string.Create<TState>(len, state, SpanAction)`**: build the final string in-place with exactly one allocation.
- **Comparison**: `StringComparison.Ordinal`/`OrdinalIgnoreCase` is fastest and culture-safe; culture-sensitive (default for `Compare`, `StartsWith(string)`, `ToUpper()`) is slower and can surprise (Turkish i).
- **Interning**: literals interned; `string.Intern` for rarely used dedupe. `CompositeFormat` (.NET 8) caches parsed format strings. `SearchValues<T>` (.NET 8) speeds multi-value search. `string.Join`, `string.Concat(IEnumerable)`, `Split` allocates - prefer span-based `EnumerateLines`/`Split`.

**Ladder**

### 7.1 [Easy] Why is string concatenation in a loop slow, and what do you use instead?
**Answer**
Strings are immutable, so each `+=` creates a new string and copies all previous characters, giving quadratic time and lots of garbage. Use `StringBuilder` (or `string.Join`/`string.Concat` over a collection).

**Explanation**
For 3-4 known pieces, `a + b + c` is one `Concat` call and fine. The compiler does not turn loop `+=` into a StringBuilder. `string.Join(",", items)` computes total length first and allocates once.

**Code**
```csharp
using System.Text;
var sb = new StringBuilder(capacity: 64);
for (int i = 0; i < 5; i++) sb.Append(i).Append(',');
sb.Length--;                       // remove trailing comma
Console.WriteLine(sb);             // Output: 0,1,2,3,4
Console.WriteLine(string.Join(",", Enumerable.Range(0, 5))); // Output: 0,1,2,3,4
```

### 7.2 [Follow-up] How are interpolated strings compiled now? Do they allocate?
**Answer**
Since C# 10, `$"..."` assigned to `string` lowers to a `DefaultInterpolatedStringHandler` that appends pieces into a pooled buffer and produces the final string with one allocation; value types are formatted directly without boxing. Before, it was `string.Format` with `object[]` boxing.

**Explanation**
If all parts are constants, the string is a compile-time constant (`const string s = $"{A}-{B}"` allowed C# 10 when interpolation holes are constant strings). When the target is `FormattableString` or `IFormattable`, old behaviour. With handler-aware APIs (`Debug.Assert`, `ILogger` via source-gen, `StringBuilder.Append($"")`) the handler writes straight into the target.

**Code**
```csharp
using System.Text;
int n = 42; double d = 3.5;
string s = $"n={n}, d={d:F1}";
Console.WriteLine(s); // Output: n=42, d=3.5

var sb = new StringBuilder();
sb.Append($"x={n}");  // uses StringBuilder.AppendInterpolatedStringHandler, no temp string
Console.WriteLine(sb); // Output: x=42
```

### 7.3 [Medium] How does `string.Create` work and when would you use it?
**Answer**
`string.Create(length, state, (span, state) => ...)` allocates the final string once and lets you fill its characters through a `Span<char>` before it is published, avoiding intermediate buffers/strings. Use for hot-path formatting, hex/base conversions and custom joins where length is known up front.

**Explanation**
The delegate should be `static` to avoid closures; pass data through `state` (use a tuple/struct). The string must be fully written (unwritten chars are `'\0'`). This is how many BCL methods (e.g. `Convert.ToHexString`-like code) are implemented.

**Code**
```csharp
static string Repeat(char c, int count) =>
    string.Create(count, c, static (span, ch) => span.Fill(ch));

static string Join3(string a, string b, string c) =>
    string.Create(a.Length + b.Length + c.Length + 2, (a, b, c), static (span, t) =>
    {
        t.a.CopyTo(span); span = span[t.a.Length..];
        span[0] = '-'; span = span[1..];
        t.b.CopyTo(span); span = span[t.b.Length..];
        span[0] = '-'; span = span[1..];
        t.c.CopyTo(span);
    });
Console.WriteLine(Repeat('x', 3));        // Output: xxx
Console.WriteLine(Join3("a", "bb", "c")); // Output: a-bb-c
```

### 7.4 [Hard] Write a custom interpolated string handler that skips formatting when disabled.
**Answer**
Define a `ref struct` marked `[InterpolatedStringHandler]` with a constructor `(int literalLength, int formattedCount, bool enabled, out bool isEnabled)` plus `AppendLiteral`/`AppendFormatted<T>`. Mark the API parameter with `[InterpolatedStringHandlerArgument("enabled")]`; when `isEnabled` is false the compiler skips evaluating the Append calls.

**Explanation**
Holes are only evaluated if the constructor reports `isEnabled == true`: the compiler skips the Append calls (and the hole expressions) otherwise. This is how `ILogger` high-perf patterns and `Debug.Assert($"...")` avoid cost.

**Code**
```csharp
using System.Runtime.CompilerServices;
using System.Text;

[InterpolatedStringHandler]
public ref struct LogHandler
{
    private readonly StringBuilder? _sb;
    public LogHandler(int literalLength, int formattedCount, bool enabled, out bool isEnabled)
    {
        isEnabled = enabled;
        _sb = enabled ? new StringBuilder(literalLength) : null;
    }
    public void AppendLiteral(string s) => _sb!.Append(s);
    public void AppendFormatted<T>(T value) => _sb!.Append(value);
    public override string ToString() => _sb?.ToString() ?? "";
}

public static class Log
{
    public static void Info(bool enabled, [InterpolatedStringHandlerArgument("enabled")] LogHandler msg)
    {
        if (enabled) Console.WriteLine(msg.ToString());
    }
}
// Log.Info(true,  $"user={42}");               // Output: user=42
// Log.Info(false, $"user={Expensive()}");      // Expensive() never called
```

**Gotchas / follow-ups**
- Handlers are `ref struct`s; can't be used in async lambdas.
- Always handle `null`/`!` carefully: `_sb` is null when disabled.

### 7.5 [Tricky] Reference equality, interning and culture bugs.
**Answer**
Equal-content strings aren't necessarily the same reference: literals are interned but runtime-created strings aren't unless `string.Intern` is called. Culture-sensitive operations can produce surprising results (Turkish `"I".ToLower()`), so use ordinal comparison for identifiers, file paths and protocol data.

**Explanation**
`==` on strings compares content (operator overload); `ReferenceEquals` compares identity. `string.Intern` keeps strings for the lifetime of the process (the intern pool is not collected in practice) - misuse leaks memory. For case-insensitive dictionaries use `StringComparer.OrdinalIgnoreCase`. `ToUpperInvariant` for normalization, not for security-sensitive comparisons - prefer `OrdinalIgnoreCase`.

**Code**
```csharp
string a = "hello";
string b = new string(new[] { 'h', 'e', 'l', 'l', 'o' });
Console.WriteLine(a == b);                                  // Output: True
Console.WriteLine(ReferenceEquals(a, b));                   // Output: False
Console.WriteLine(ReferenceEquals(a, string.Intern(b)));    // Output: True

var tr = new System.Globalization.CultureInfo("tr-TR");
Console.WriteLine("TITLE".ToLower(tr));                     // Output: tıtle  (dotless i)
Console.WriteLine("TITLE".Equals("title", StringComparison.OrdinalIgnoreCase)); // Output: True
```

**Interview traps**
- "StringBuilder is always faster" - for 2-3 concatenations it's slower.
- `string.Intern` on user data = unbounded memory.
- Default string comparisons (`Compare`, `IndexOf(string)`) are culture-sensitive; `Equals`/`==` are ordinal.
- `Substring` allocates; `AsSpan` doesn't.
- `Split` + LINQ in hot paths allocates heavily.

**Cheat sheet**
- Loop concat -> StringBuilder / Join; few known pieces -> `+` / interpolation.
- `$""` -> DefaultInterpolatedStringHandler (pooled, no boxing).
- `string.Create` = single allocation, write via span.
- Use `StringComparison.Ordinal*` unless linguistic.
- `AsSpan` instead of `Substring` in hot paths.

---
## Topic 8: BenchmarkDotNet Basics

**Study notes**

- **BenchmarkDotNet** (NuGet `BenchmarkDotNet`) is the standard micro-benchmark library: it generates a separate project per benchmark, runs it **out-of-process**, performs *pilot -> warm-up -> actual* iterations, applies statistics (mean, error, stddev, median) and reports allocations.
- Essentials: `[Benchmark]` methods in a **public, non-sealed class**; run `BenchmarkRunner.Run<T>()` in a **Release build, without a debugger** (`dotnet run -c Release`). `[MemoryDiagnoser]` adds Gen0/1/2 collections and `Allocated` columns. `[Baseline = true]` on a benchmark gives a `Ratio` column. `[Params(...)]` for input sizes; `[GlobalSetup]`/`[IterationSetup]`/`[GlobalCleanup]` for setup outside the measured code. `[SimpleJob(RuntimeMoniker.Net80)]`, `[ShortRun]`, `[DisassemblyDiagnoser]`, `[Orderer]`.
- Pitfalls: dead-code elimination (always **return** the result or use `Consumer`), constant folding (don't use compile-time constants as inputs), setup inside the benchmark method, measuring Debug builds, tiny noise differences, allocation in setup vs. measured code, differing JIT tiers (BDN handles by warm-up).
- Read the output: **Mean** is a typical time; look at **Allocated** and **Gen0** to see GC pressure; **Ratio** relative to baseline.

**Ladder**

### 8.1 [Easy] How do you write a basic benchmark?
**Answer**
Create a public class with `[Benchmark]` methods, annotate with `[MemoryDiagnoser]`, and call `BenchmarkRunner.Run<T>()` from `Main` in a Release build. BenchmarkDotNet handles warm-up, iteration counts and statistics.

**Explanation**
Run from the command line: `dotnet run -c Release`. If you run in Debug or with a debugger attached, BDN warns/refuses (validators). Output includes a Markdown table you can paste into PRs.

**Code**
```csharp
// dotnet add package BenchmarkDotNet
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;
using System.Text;

BenchmarkRunner.Run<ConcatBench>();

[MemoryDiagnoser]
public class ConcatBench
{
    [Params(10, 1000)] public int N;

    [Benchmark(Baseline = true)]
    public string PlusEquals()
    {
        string s = "";
        for (int i = 0; i < N; i++) s += "x";
        return s;                       // returning prevents dead-code elimination
    }

    [Benchmark]
    public string Builder()
    {
        var sb = new StringBuilder();
        for (int i = 0; i < N; i++) sb.Append('x');
        return sb.ToString();
    }
}
```

### 8.2 [Follow-up] Why are naive `Stopwatch` loops unreliable compared to BenchmarkDotNet?
**Answer**
A Stopwatch loop ignores JIT tiering, warm-up, GC noise, CPU frequency effects, optimiser dead-code elimination and statistical variance. BDN isolates the process, warms up, repeats until stable and gives confidence intervals.

**Explanation**
Tiered compilation means the first calls run unoptimised tier-0 code; hot methods are re-JITted after 30 calls (+ backedge counting/OSR). A one-off Stopwatch measurement may mostly measure tier-0 code and JIT time. Also the JIT can remove a computation whose result is unused, giving "0 ns" results.

**Code**
```csharp
// Misleading approach:
var sw = System.Diagnostics.Stopwatch.StartNew();
for (int i = 0; i < 1_000_000; i++) { _ = Math.Sqrt(i); } // result unused: may be eliminated
sw.Stop();
Console.WriteLine(sw.ElapsedMilliseconds); // meaningless without warm-up and use of result
```

### 8.3 [Medium] How do you use `[Params]`, `[GlobalSetup]`, and `[IterationSetup]`, and when is each right?
**Answer**
`[Params]` runs each benchmark for each value (e.g. input sizes). `[GlobalSetup]` runs once per parameter combination - use for building data. `[IterationSetup]` runs before every iteration but is imprecise for fast (<100 ms) benchmarks, so use only when the benchmark mutates its input.

**Explanation**
BDN docs warn: with `IterationSetup` it can't batch invocations, so micro-benchmarks need `[InvocationCount]`/`UnrollFactor` tuning; results for very fast methods become unreliable. Prefer making each benchmark non-destructive or copying the data inside the benchmark (and including that cost in the baseline).

**Code**
```csharp
using BenchmarkDotNet.Attributes;

[MemoryDiagnoser]
public class SortBench
{
    private int[] _data = Array.Empty<int>();
    [Params(100, 10_000)] public int Size;

    [GlobalSetup]
    public void Setup() => _data = Enumerable.Range(0, Size).Reverse().ToArray();

    [Benchmark(Baseline = true)]
    public int[] ArraySort() { var c = (int[])_data.Clone(); Array.Sort(c); return c; }

    [Benchmark]
    public int[] LinqOrder() => _data.OrderBy(x => x).ToArray();
}
```

### 8.4 [Hard] What mistakes make a benchmark lie, and how do you check for them?
**Answer**
Dead-code elimination, constant folding, measuring setup, shared mutable state across invocations, wrong build config, CPU throttling/noisy machine, and comparing different amounts of work. Check with the disassembly diagnoser, `[MemoryDiagnoser]`, run to run variance (StdDev, Error), and reviewing BDN's warnings.

**Explanation**
Return values or use `Consumer.Consume`. Make inputs fields, not `const`/literals. `[DisassemblyDiagnoser]` shows JIT output to confirm the loop wasn't eliminated. Check "MultimodalDistribution" and "Outliers" hints. Also consider hardware intrinsics / PGO: .NET 8 uses dynamic PGO, so BDN default tiered PGO may change the result vs. AOT. Benchmarks measure *micro*-behaviour; confirm with a load test.

**Code**
```csharp
using BenchmarkDotNet.Attributes;

public class DceBench
{
    private int _x = 5;                 // field, not const -> no constant folding

    [Benchmark] public int Good() => _x * _x;   // returned: result used
    [Benchmark] public void Bad() { var _ = _x * _x; } // may be optimised to nothing
}
```

**Gotchas / follow-ups**
- Benchmark classes must be `public`, non-`sealed`, with public benchmark methods (non-static allowed).
- Follow-up: How do you compare .NET 6 vs .NET 8? Multiple `[SimpleJob(RuntimeMoniker.Net60)]` / `Net80` attributes.

**Interview traps**
- Running in Debug or from Visual Studio with F5 - invalid.
- Returning nothing from a benchmark -> dead-code elimination.
- `Mean` alone is not enough; check `Allocated`, `StdDev`, `Ratio`.
- Micro-benchmark wins don't always translate to app-level improvements.

**Cheat sheet**
- `[Benchmark]`, `[MemoryDiagnoser]`, `BenchmarkRunner.Run<T>()`, Release.
- `[Params]` inputs; `[GlobalSetup]` setup; `Baseline = true` for ratio.
- Return results to avoid dead-code elimination.
- Watch Mean/Error/StdDev/Allocated/Gen0.
- Use `DisassemblyDiagnoser` to verify what was measured.

---

## Topic 9: Exception Handling

**Study notes**

- `try` / `catch` (first matching handler, ordered most-specific first) / `finally` (always runs on normal exit, exception, `return`; not on process termination/`Environment.FailFast`/stack overflow).
- **Two-pass model**: pass 1 searches for a handler, evaluating **exception filters** (`catch (E e) when (cond)`) *before* the stack unwinds; pass 2 unwinds, running `finally` blocks, then the catch. Filters run with the original stack intact -> better diagnostics, and a false filter doesn't catch (no rethrow cost).
- **Rethrow**: `throw;` preserves original stack trace; `throw ex;` resets stack trace to the current line (bad). To rethrow elsewhere preserving the stack: `ExceptionDispatchInfo.Capture(ex).Throw()`. Wrapping: `throw new X("msg", ex)` keeps `InnerException`.
- **Custom exceptions**: derive from `Exception` (not `ApplicationException`), name ends in `Exception`, provide `()`, `(string)`, `(string, Exception)` constructors; add data as read-only properties. `[Serializable]` + serialization ctor isn't required in modern .NET.
- **Cost**: creating + throwing is expensive (stack trace capture, two-pass unwinding; tens of microseconds, vs nanoseconds for return codes; .NET 9 made managed EH ~2-4x faster). Don't use exceptions for normal control flow: use `TryParse`, `TryGetValue`, `Result` patterns.
- **AggregateException**: holds multiple `InnerExceptions`; produced by `Task.Wait()`, `.Result`, `Parallel`, `Task.WhenAll().Wait()`. `await` unwraps and rethrows the **first** inner exception only.
- Uncatchable: `StackOverflowException`, `ExecutionEngineException`; `OutOfMemoryException` is catchable but dangerous. `catch (Exception)` swallowing = anti-pattern.

**Ladder**

### 9.1 [Easy] In what order do try/catch/finally execute? Does finally run after `return`?
**Answer**
The try block runs; if it throws, the first matching catch runs; `finally` runs afterwards regardless of outcome, even when the try or catch returns. The return value is computed first, then finally executes, then control leaves.

**Explanation**
`finally` is for cleanup (prefer `using`). Don't `return`/`throw` inside `finally`: C# forbids `return` there, and throwing replaces the in-flight exception. Value of a returned *value type* is captured before finally; mutating it in finally doesn't change what is returned.

**Code**
```csharp
static int F()
{
    int x = 1;
    try { return x; }
    finally { x = 2; Console.WriteLine("finally"); }
}
Console.WriteLine(F());
// Output: finally
//         1
```

### 9.2 [Follow-up] `throw;` vs `throw ex;` vs wrapping - what's the difference?
**Answer**
`throw;` rethrows the current exception unchanged, preserving the original stack trace. `throw ex;` throws the same object but resets the trace to the rethrow site, hiding the real origin. Wrapping (`throw new X("...", ex)`) adds context while retaining the original as `InnerException`.

**Explanation**
Only `throw;` inside a catch block is legal as a bare rethrow. To rethrow later or on another thread without losing the trace, use `ExceptionDispatchInfo.Capture(ex).Throw()`; `Task` and `await` use this internally. Prefer a filter or `catch { log; throw; }`; don't catch just to rethrow.

**Code**
```csharp
using System.Runtime.ExceptionServices;

static void Thrower() => throw new InvalidOperationException("boom");

try { try { Thrower(); } catch (Exception ex) { throw ex; } }          // trace now points here
catch (Exception e) { Console.WriteLine(e.StackTrace!.Contains("Thrower")); } // Output: False

try { try { Thrower(); } catch { throw; } }                              // preserved
catch (Exception e) { Console.WriteLine(e.StackTrace!.Contains("Thrower")); } // Output: True

try { try { Thrower(); } catch (Exception ex) { ExceptionDispatchInfo.Capture(ex).Throw(); } }
catch (Exception e) { Console.WriteLine(e.StackTrace!.Contains("Thrower")); } // Output: True
```

### 9.3 [Medium] What are exception filters and why are they better than catch-and-rethrow? How do you design custom exceptions?
**Answer**
`catch (E e) when (predicate)` evaluates the predicate during the first pass, so the stack isn't unwound if it's false and the full original stack/locals are available for logging or dumps. Custom exceptions should derive from `Exception`, expose context as properties, and provide the standard constructors.

**Explanation**
Filters run *before* inner `finally` blocks execute, which can surprise: state changed in a finally isn't yet visible to the filter. Filters that throw are swallowed and treated as false. A common trick: `when (Log(e))` where `Log` returns false to log without catching.

**Code**
```csharp
static bool Log(string s) { Console.WriteLine(s); return false; }

try
{
    try { throw new Exception("x"); }
    catch (Exception) when (Log("filter")) { }
    finally { Console.WriteLine("inner finally"); }
}
catch { Console.WriteLine("outer catch"); }
// Output: filter
//         inner finally
//         outer catch

public class OrderNotFoundException : Exception
{
    public int OrderId { get; }
    public OrderNotFoundException() { }
    public OrderNotFoundException(string message) : base(message) { }
    public OrderNotFoundException(string message, Exception inner) : base(message, inner) { }
    public OrderNotFoundException(int orderId) : base($"Order {orderId} not found") => OrderId = orderId;
}
```

### 9.4 [Hard] How does AggregateException interact with Task, Wait/Result, await and WhenAll?
**Answer**
`Wait()`/`Result` wrap failures in `AggregateException`; `await` unwraps and throws the first inner exception only. For `Task.WhenAll`, `await` throws just the first, but `task.Exception.InnerExceptions` holds all of them - inspect the task if you need every failure.

**Explanation**
Never block with `.Result`/`.Wait()` in async code (deadlocks, aggregate wrapping); use `GetAwaiter().GetResult()` if you must block (rethrows the original, no wrapper). `Flatten()` collapses nested aggregates; `Handle(predicate)` handles selected ones and rethrows the rest. `Parallel.ForEach` aggregates exceptions; `Parallel.ForEachAsync` rethrows first.

**Code**
```csharp
var t1 = Task.FromException(new InvalidOperationException("a"));
var t2 = Task.FromException(new ArgumentException("b"));
var all = Task.WhenAll(t1, t2);

try { await all; }
catch (Exception ex)
{
    Console.WriteLine(ex.GetType().Name);                  // Output: InvalidOperationException
    Console.WriteLine(all.Exception!.InnerExceptions.Count); // Output: 2
}

try { t1.Wait(); }
catch (AggregateException ae) { Console.WriteLine(ae.InnerException!.Message); } // Output: a
```

**Gotchas / follow-ups**
- Unobserved task exceptions don't crash the process in .NET Core (the `UnobservedTaskException` event still fires) .
- `async void` exceptions go to the `SynchronizationContext` / crash the process.

### 9.5 [Tricky] Performance cost, exceptions in finally, and catch ordering gotchas.
**Answer**
Throwing is thousands of times slower than a failed `TryParse`, so never use exceptions for expected flow. An exception thrown inside `finally` (or `Dispose`) replaces the in-flight one, losing it. Catch clauses are checked top to bottom; a general catch before a specific one is a compile error (CS0160).

**Explanation**
Where it hurts: parsing user input in loops, `int.Parse` in hot paths, `Convert` failures, `KeyNotFoundException` for missing keys. The first throw in a process also pays for loading resources/stack trace machinery. Also: `throw` in a static constructor produces `TypeInitializationException` permanently for that type; exceptions inside `Task`-returning methods that aren't `async` are thrown synchronously, while in `async` methods they're stored in the task.

**Code**
```csharp
Console.WriteLine(int.TryParse("abc", out _)); // Output: False  (no exception)

static async Task A() { throw new Exception("in async"); }       // stored in task
static Task B() { throw new Exception("sync"); }                  // thrown at call

var t = A();                                  // no exception here
Console.WriteLine(t.IsFaulted);               // Output: True
try { B(); } catch (Exception e) { Console.WriteLine(e.Message); } // Output: sync
```

**Interview traps**
- `throw ex;` loses the original stack trace.
- `catch (Exception) { }` hides bugs; log or let it propagate.
- Exceptions in `finally` mask the original.
- `await` shows only the first exception from `WhenAll`.
- `return` is illegal in `finally`; using `ApplicationException` is outdated guidance.

**Cheat sheet**
- `throw;` = preserve; `throw ex;` = reset; `ExceptionDispatchInfo` = rethrow later.
- Filters run in pass 1 before unwinding.
- Custom: derive `Exception`, 3 standard ctors, extra props.
- Exceptions are expensive -> use `Try*` APIs for expected failures.
- `Wait/Result` -> AggregateException; `await` -> first inner only.

---

## Topic 10: Pattern Matching

**Study notes**

- **Pattern kinds**: constant (`1`, `null`, `"x"`), type (`string s`), declaration, `var`, discard `_`, **relational** (`< 5`, `>= 18`), **logical** (`and`, `or`, `not`, parenthesised), **property** (`{ Length: > 0 }`, extended `{ Address.City: "X" }` C# 10), **positional** (uses `Deconstruct`: `(0, 0)`), **list** (C# 11: `[1, 2, ..]`, `[var first, .., var last]`, slice `.. var rest`), `is` expressions, `switch` statements/expressions, `when` guards.
- **Switch expression** (C# 8): `x switch { pattern => value, ... }` is an expression, must be exhaustive (compiler warns CS8509; runtime throws `SwitchExpressionException`). Arms are evaluated top to bottom; a subsumed arm is a compile error (CS8510).
- `is not null` is preferred to `!= null` (can't be fooled by overloaded `==`). `is { }` means non-null.
- List patterns work on types with `Length`/`Count` and an `int` indexer (arrays, `List<T>`, `string`, `Span<T>`), not on plain `IEnumerable<T>`.

**Ladder**

### 10.1 [Easy] What is pattern matching and what does a switch expression look like?
**Answer**
Pattern matching tests a value's shape/type/value and optionally extracts parts. A switch expression returns a value from the first matching arm and replaces long if/else or switch statements.

**Explanation**
Common patterns: type (`obj is string s`), constant, relational and logical. The discard `_` arm is the default. Switch expressions are expressions, so they can be expression-bodied members.

**Code**
```csharp
static string Describe(object? o) => o switch
{
    null            => "null",
    int n when n < 0 => "negative int",
    int             => "int",
    string { Length: 0 } => "empty string",
    string s        => $"string({s.Length})",
    _               => "other"
};
Console.WriteLine(Describe(-5));     // Output: negative int
Console.WriteLine(Describe(""));     // Output: empty string
Console.WriteLine(Describe("abc"));  // Output: string(3)
Console.WriteLine(Describe(null));   // Output: null
```

### 10.2 [Follow-up] Explain relational, logical, property (including extended) and positional patterns.
**Answer**
Relational patterns compare to constants (`>= 18`), logical patterns combine patterns (`and`/`or`/`not`), property patterns match members (`{ City: "Pune" }`, nested via `{ Address.City: "Pune" }`), and positional patterns deconstruct (`(0, _)`) via `Deconstruct`. They compose freely.

**Explanation**
`and` binds tighter than `or`; `not` binds tighter than both. Property patterns include an implicit non-null check. Tuples support positional patterns natively; use them to switch on several values at once.

**Code**
```csharp
record Address(string City);
record Person(string Name, int Age, Address Address);

static string Bucket(int age) => age switch
{
    < 0 => "invalid",
    >= 0 and < 18 => "minor",
    >= 18 and < 65 => "adult",
    _ => "senior"
};

static string Where(Person p) => p switch
{
    { Address.City: "Pune", Age: >= 18 } => "adult in Pune",
    { Address: null } => "no address",
    _ => "elsewhere"
};
Console.WriteLine(Bucket(30)); // Output: adult
Console.WriteLine(Where(new Person("A", 20, new Address("Pune")))); // Output: adult in Pune

static string Quad(int x, int y) => (x, y) switch
{
    (0, 0) => "origin",
    (> 0, > 0) => "Q1",
    (_, 0) or (0, _) => "axis",
    _ => "other"
};
Console.WriteLine(Quad(1, 2)); // Output: Q1
```

### 10.3 [Medium] How do list patterns work?
**Answer**
List patterns (C# 11) match sequences by element: `[1, 2, 3]` exact, `[1, ..]` starts with 1, `[.., 9]` ends with 9, `[var first, .., var last]` captures, and `.. var rest` captures a slice. The slice `..` can appear once.

**Explanation**
The compiler requires a countable type (`Length` or `Count`) and an `int` indexer; slice captures need `Slice(int,int)` or a range indexer (arrays give a copied array, `string` a substring, `Span<T>` a span). `[]` matches empty. Works in `is` and `switch`.

**Code**
```csharp
static string Show(int[] a) => a switch
{
    [] => "empty",
    [var x] => $"one:{x}",
    [var f, .., var l] => $"{f}..{l}"
};
Console.WriteLine(Show(new[] { 1, 2, 3, 4 })); // Output: 1..4
Console.WriteLine(Show(new[] { 7 }));          // Output: one:7

int[] nums = { 1, 2, 3 };
if (nums is [1, .. var rest]) Console.WriteLine(rest.Length); // Output: 2
```

### 10.4 [Hard] How does the compiler check exhaustiveness and subsumption, and what happens at runtime when nothing matches?
**Answer**
It reports CS8509 (non-exhaustive switch expression) as a warning and CS8510 (arm already handled) as an error when an earlier arm covers a later one. If no arm matches at runtime a `SwitchExpressionException` is thrown (or `InvalidOperationException` for older cases). Enums are only considered exhaustive when all members *and* the unnamed values are handled, hence the warning with `_`-less enum switches.

**Explanation**
Because an enum can hold any underlying value, `Color c => c switch { Red=>..., Green=>... }` warns even if all members are listed; add a `_ => throw new ArgumentOutOfRangeException()` arm. Compiler generates decision DAG; arms are reordered internally for efficiency but semantics follow source order. Guards (`when`) make arms non-total for exhaustiveness analysis.

**Code**
```csharp
enum Color { Red, Green }
static string Name(Color c) => c switch
{
    Color.Red => "red",
    Color.Green => "green",
    _ => throw new ArgumentOutOfRangeException(nameof(c))
};
Console.WriteLine(Name(Color.Green)); // Output: green
try { Name((Color)7); }
catch (ArgumentOutOfRangeException) { Console.WriteLine("bad enum"); } // Output: bad enum
```

**Gotchas / follow-ups**
- Without the `_` arm, `(Color)7` throws `SwitchExpressionException`.
- Follow-up: a `when` guard on an arm that covers all cases doesn't count for exhaustiveness.

### 10.5 [Tricky] Operator precedence and null gotchas: `is not A or B`, `is T t` with null, `var` pattern.
**Answer**
`x is not A or B` parses as `(not A) or B`, not `not (A or B)`. Type patterns never match `null` (`null is string s` is false), but the `var` pattern does (`var v` matches null). Use parentheses to be explicit.

**Explanation**
`is not null` is the idiomatic non-null check. Definite assignment: variables declared in a pattern are only definitely assigned when the match succeeds - `if (o is not string s) return; use(s);` works because of flow analysis. A `case` guard's side effects run only for arms reached.

**Code**
```csharp
object o = 5;
Console.WriteLine(o is not string or int);   // Output: True   ((not string) or int)
Console.WriteLine(o is not (string or int)); // Output: False

string? str = null;
Console.WriteLine(str is string s);          // Output: False
Console.WriteLine(str is var v);             // Output: True   (var matches null)

object x = "hi";
if (x is not string t) return;
Console.WriteLine(t.Length);                 // Output: 2
```

**Interview traps**
- Switch expression without discard -> warning + runtime exception.
- Enum "all cases covered" still needs a default arm.
- `and`/`or`/`not` aren't `&&`/`||`/`!` - they combine *patterns*.
- List patterns don't work on `IEnumerable<T>`.
- `is T x` fails for null; `is var x` doesn't.

**Cheat sheet**
- `x switch { pattern => expr, _ => ... }`.
- Relational: `< > <= >=`; logical: `and or not`.
- Property: `{ A.B: pat }`; positional: `(p1, p2)`.
- List: `[a, .., b]`, slice capture `.. var r`.
- Prefer `is null` / `is not null`.

---

## Topic 11: Records and with-expressions

**Study notes**

- `record` (alias `record class`, C# 9) = reference type with **compiler-generated value-based equality**, `ToString()` (`Person { Name = Ann, Age = 30 }`), `GetHashCode`, `==`/`!=`, **copy constructor** + hidden `<Clone>$` method, `EqualityContract` (type is part of equality) and, for **positional** records, a primary constructor with init-only properties and `Deconstruct`.
- `record struct` (C# 10) = value type with the same features; positional properties are *mutable* unless declared `readonly record struct`.
- **`with` expression** copies the instance (calls the synthesized clone, so it preserves the **runtime type**) and then applies object-initializer assignments to `init` (or settable) properties. Shallow copy.
- Records can inherit only from other records (or `object`); interfaces fine. Positional parameters of a derived record are passed to the base.
- Records are *immutable by convention* only: you can declare `{ get; set; }` members.
- EF Core notes: value-like equality conflicts with change-tracking identity; records work fine as DTOs, value objects, owned types/ value converters; avoid for tracked entities.

**Ladder**

### 11.1 [Easy] What is a record and how is it different from a class?
**Answer**
A record is a type optimised for immutable data with automatic value equality, `ToString`, deconstruction and non-destructive mutation via `with`. A class uses reference equality by default.

**Explanation**
`record Person(string Name, int Age);` generates init-only properties, constructor, `Equals`, `GetHashCode`, `ToString`, `Deconstruct`, and `with` support. Two records are equal if they are the same type and all instance fields are equal (using `EqualityComparer<T>.Default` for each field).

**Code**
```csharp
var p = new Person("Ann", 30);
var q = p with { Age = 31 };
Console.WriteLine(p);                       // Output: Person { Name = Ann, Age = 30 }
Console.WriteLine(q);                       // Output: Person { Name = Ann, Age = 31 }
Console.WriteLine(p == new Person("Ann", 30)); // Output: True
var (name, age) = p;
Console.WriteLine($"{name} {age}");         // Output: Ann 30

record Person(string Name, int Age);
```

### 11.2 [Follow-up] `record` vs `record struct` vs `readonly record struct`?
**Answer**
`record` (class) is a heap object with value equality; `record struct` is a value type with value equality and *mutable* positional properties; `readonly record struct` makes it immutable and lets the compiler elide defensive copies. Choose structs for small immutable value types to avoid allocation.

**Explanation**
Class records have `EqualityContract` and support inheritance; struct records don't (structs can't inherit). Struct records get `IEquatable<T>`, `==`, `!=`, `ToString`, and `with` but `with` on a struct is just a copy + member assignment. Equality for `record struct` is generated on fields, avoiding the slow reflection-based `ValueType.Equals`.

**Code**
```csharp
readonly record struct Point(int X, int Y);
record struct MutablePoint(int X, int Y);

var a = new Point(1, 2);
var b = a with { X = 5 };
Console.WriteLine(b);       // Output: Point { X = 5, Y = 2 }

var m = new MutablePoint(1, 2);
m.X = 10;                   // allowed: positional props of record struct are mutable
Console.WriteLine(m.X);     // Output: 10
```

### 11.3 [Medium] What exactly does `with` do under the hood? How does it behave with inheritance?
**Answer**
`a with { X = 1 }` calls the compiler-generated virtual clone method (a copy constructor call) to copy the object and then sets the specified init properties. Because clone is virtual, the result has the *runtime* type of `a`, even if the static type is a base record.

**Explanation**
The copy constructor is `protected R(R original)` and copies all fields (shallow). Derived records override `<Clone>$`. So `Person p = new Student(...); var p2 = p with { Age = 2 };` yields a `Student`. You can write your own copy constructor to customise cloning (e.g. deep copy a list). Since it's shallow, reference-type members are shared between copies.

**Code**
```csharp
record Person(string Name, int Age);
record Student(string Name, int Age, string School) : Person(Name, Age);

Person p = new Student("Ann", 20, "MIT");
Person p2 = p with { Age = 21 };
Console.WriteLine(p2.GetType().Name); // Output: Student
Console.WriteLine(p2);                // Output: Student { Name = Ann, Age = 21, School = MIT }
```

### 11.4 [Hard] Equality in records: what's compared, and what surprises exist with collections and inheritance?
**Answer**
Generated equality compares `EqualityContract` (the exact runtime type) and every instance field with the default comparer. Collection-typed members compare by reference, so two records holding different-but-equal lists are *not* equal. A base-typed record isn't equal to a derived-typed one even if the shared fields match.

**Explanation**
For value semantics over collections, hold `ImmutableArray<T>` with a custom comparer, or override `Equals`/`GetHashCode` yourself (when you declare `Equals(R? other)` the compiler doesn't synthesize it). Mutating a record used as a dictionary key (if it has settable props) breaks the dictionary. `GetHashCode` for fields that are mutable collections is by reference hash.

**Code**
```csharp
record Basket(List<int> Items);
var l1 = new List<int> { 1 };
Console.WriteLine(new Basket(l1) == new Basket(new List<int> { 1 })); // Output: False
Console.WriteLine(new Basket(l1) == new Basket(l1));                 // Output: True

record Person(string Name, int Age);
record Student(string Name, int Age, string School) : Person(Name, Age);
Person a = new Person("A", 1);
Person b = new Student("A", 1, "X");
Console.WriteLine(a == b); // Output: False   (different EqualityContract)
```

**Gotchas / follow-ups**
- Custom `ToString` can be sealed in a record (`public sealed override string ToString()`) to stop derived records regenerating it.
- Follow-up: can a record have a parameterless constructor? Yes, declare it; positional ctor still available.

### 11.5 [Tricky] `init` properties, validation, and records as EF Core entities.
**Answer**
Positional records generate `init` properties, so validation must be put in an explicit property or constructor; `with` can bypass constructor validation because it uses the copy constructor and property setters. For EF Core, records' value equality makes tracked entities with the same data "equal" and can confuse identity resolution/collections, so prefer classes for entities and records for DTOs/value objects.

**Explanation**
To validate on `with`, declare the property with a full `init` accessor that checks the value. EF Core materialises via constructor/property binding; it works with records, but `Equals`/`GetHashCode` on mutable tracked state changes as properties change, breaking `HashSet`/`Dictionary` usage. Keep entity identity on key only or use classes.

**Code**
```csharp
public record Money
{
    private readonly decimal _amount;
    public decimal Amount
    {
        get => _amount;
        init => _amount = value >= 0 ? value : throw new ArgumentOutOfRangeException(nameof(value));
    }
    public string Currency { get; init; } = "INR";
}

var m = new Money { Amount = 5m };
try { var bad = m with { Amount = -1m }; }   // the init accessor runs on 'with' too
catch (ArgumentOutOfRangeException) { Console.WriteLine("blocked"); } // Output: blocked
```
*Note:* validation placed only in a positional primary-constructor initialiser is **not** re-run by `with`; put it in the `init` accessor as above.

**Interview traps**
- "Records are immutable" - only if you keep `init`/readonly members.
- `with` is shallow; nested lists are shared.
- Record `==` includes runtime type; base != derived.
- A positional `record struct` is mutable by default.
- Don't use records with mutable state as dictionary keys or EF-tracked entities.

**Cheat sheet**
- `record` = class + value equality/ToString/with/Deconstruct.
- `record struct` mutable; `readonly record struct` immutable.
- `with` = virtual clone + init props; preserves runtime type.
- Collections inside records compare by reference.
- Validate in accessors, not just the primary constructor.

---
## Topic 12: Nullable Reference Types

**Study notes**

- NRT (C# 8) is a **compile-time-only** feature: `string` = non-null *intent*, `string?` = may be null. Enable with `<Nullable>enable</Nullable>` (project) or `#nullable enable/disable/restore` (file/region). Nothing changes at runtime - no checks, same IL (just attributes like `[Nullable]` metadata).
- The compiler does **flow analysis**, tracking each expression's *null-state* (`maybe-null` / `not-null`) and emits warnings (CS8600-CS8605, CS8618 for uninitialised non-null fields, CS8625, ...). Warnings are about *possible* null deref, not guarantees.
- **Null-forgiving `!`**: suppress a warning ("I know it isn't null") - no runtime effect; `null!`/`default!` for late-initialised members (DI, EF navigation, serializer-populated).
- **Annotation attributes** (`System.Diagnostics.CodeAnalysis`) describe contracts: `[NotNullWhen(true)]`, `[MaybeNullWhen(false)]`, `[NotNull]`, `[MaybeNull]`, `[AllowNull]`, `[DisallowNull]`, `[NotNullIfNotNull("p")]`, `[MemberNotNull(nameof(f))]`, `[MemberNotNullWhen]`, `[DoesNotReturn]`.
- **Generics**: `T?` on an unconstrained `T` = "T or null/default"; with `where T : struct` it is `Nullable<T>`; with `where T : class` it's an annotated reference. Constraints `notnull`, `class?`.
- `required` members (C# 11) are the modern answer to "non-null property set via initializer".

**Ladder**

### 12.1 [Easy] What do `string` and `string?` mean with NRT enabled, and does it prevent NullReferenceException?
**Answer**
`string` declares non-null intent, `string?` allows null. The compiler warns when you dereference a possibly-null value or assign null to a non-null one, but nothing is enforced at runtime, so a `NullReferenceException` can still happen when null sneaks through (interop, reflection, deserialization, `!`).

**Explanation**
Enable it per project. Migration strategy: turn on `enable` with warnings as warnings, fix by file using `#nullable enable`, then treat warnings as errors (`<WarningsAsErrors>nullable</WarningsAsErrors>`). Value types aren't affected (`int?` is `Nullable<int>`, a real type change).

**Code**
```csharp
#nullable enable
string name = "Ann";
string? maybe = GetOrNull(name);

// Console.WriteLine(maybe.Length);        // warning CS8602: possible null dereference
if (maybe is not null) Console.WriteLine(maybe.Length);  // fine, flow analysis knows
Console.WriteLine(maybe?.Length ?? 0);

static string? GetOrNull(string s) => s.Length > 5 ? s : null;
// Output: 0
```

### 12.2 [Follow-up] How do `[NotNullWhen]`, `[MaybeNullWhen]`, `[NotNullIfNotNull]` and `[MemberNotNull]` help the compiler?
**Answer**
They let you describe nullability contracts that signatures can't express. E.g. `bool TryGet(..., [NotNullWhen(true)] out string? value)` tells callers that `value` is non-null when the method returns true; `[MemberNotNull(nameof(_f))]` says calling a method initialises a field, which silences CS8618 in constructors that call it.

**Explanation**
BCL uses them extensively (`string.IsNullOrEmpty([NotNullWhen(false)] string?)`, `Dictionary.TryGetValue([MaybeNullWhen(false)] out TValue)`). `[NotNullIfNotNull("input")]` on a return value: result is non-null if the named parameter is non-null (e.g. `Trim(string? s)` overloads). `[DoesNotReturn]` helps for throw-helper methods.

**Code**
```csharp
#nullable enable
using System.Diagnostics.CodeAnalysis;

static bool TryGetName(int id, [NotNullWhen(true)] out string? name)
{
    name = id == 1 ? "Ann" : null;
    return name is not null;
}
if (TryGetName(1, out var n)) Console.WriteLine(n.Length); // Output: 3  (no warning: n is non-null)

class Service
{
    private string _conn;                        // no CS8618 thanks to MemberNotNull
    public Service() => Init();
    [MemberNotNull(nameof(_conn))]
    private void Init() => _conn = "server=.";
    public string Conn => _conn;
}
```

### 12.3 [Medium] How does `T?` behave in generics, and what about `where T : notnull`?
**Answer**
Unconstrained `T?` means "T, but may be null/default" and doesn't turn value types into `Nullable<T>`. If you want `Nullable<T>`, constrain `where T : struct`; for annotated reference types use `where T : class`. `notnull` forbids nullable type arguments.

**Explanation**
You can't have one method returning "nullable T" for both structs and classes with the same meaning - provide two overloads, or use unconstrained `T?` and accept that for `int` it's just `int` (default 0). Dictionary keys: `TKey : notnull`.

**Code**
```csharp
#nullable enable
static T? FirstS<T>(IList<T> l) where T : struct => l.Count > 0 ? l[0] : null;   // Nullable<T>
static T? FirstC<T>(IList<T> l) where T : class  => l.Count > 0 ? l[0] : null;   // string? etc.
static T? FirstAny<T>(IList<T> l) => l.Count > 0 ? l[0] : default;               // no Nullable<int>

Console.WriteLine(FirstS(new int[0]) is null);        // Output: True   (int?)
Console.WriteLine(FirstC(new[] { "a" }));             // Output: a
Console.WriteLine(FirstAny(new int[0]));              // Output: 0      (just int)
```

### 12.4 [Hard] How does the compiler's flow analysis treat fields, properties, constructors and loops - and where does it give up?
**Answer**
It tracks null-state of locals precisely, and of fields/properties only until something could change them (method calls, other threads), after which state resets to the declared annotation for properties (but not always for fields). Non-nullable fields must be definitely non-null at the end of every constructor (CS8618) unless `required`, `[MemberNotNull]`, or `null!`.

**Explanation**
Property access is assumed stable within a straight-line check (`if (p.Name != null) use(p.Name)` works) even though it's technically a call. Lambdas and local functions are analysed conservatively. Arrays: `new string[3]` creates elements that are null despite `string[]` - the compiler can't track that, a known hole. Generic code: `default(T)` is `maybe-null`.

**Code**
```csharp
#nullable enable
class Person
{
    public required string Name { get; init; }         // must be set by initializer
    public string? Nick { get; set; }
    public int NickLength => Nick is null ? 0 : Nick.Length;   // flow state: safe
}
var p = new Person { Name = "Ann" };
Console.WriteLine(p.NickLength); // Output: 0

string[] arr = new string[2];     // elements are null but typed non-null: no warning
Console.WriteLine(arr[0] is null); // Output: True
```

**Gotchas / follow-ups**
- `Nick` null-check then a method call on another object may invalidate assumptions in concurrent code - copy to a local.
- Follow-up: how to make a project-wide strategy? `Directory.Build.props` with Nullable enable + analyzers.

### 12.5 [Tricky] Where do nulls still leak in with NRT on: deserialization, EF Core, `!` and reflection?
**Answer**
JSON/XML deserializers, EF Core materialisation, Reflection, and `default` bypass constructors and set properties to null regardless of annotation. `!` and `= null!` just silence the compiler. EF Core uses nullability annotations to decide column nullability, so `string` -> NOT NULL, `string?` -> NULL.

**Explanation**
System.Text.Json: missing property for `string Name` stays null unless `required` (JSON deserialization then throws `JsonException`) or a default value. EF navigation: `public Customer Customer { get; set; } = null!;` is the standard idiom for required navigation set by EF. Validate at boundaries (`ArgumentNullException.ThrowIfNull(x)`), since NRT is not a runtime guarantee.

**Code**
```csharp
#nullable enable
using System.Text.Json;

record Dto { public string Name { get; set; } = default!; }   // 'default!' silences warning

var d = JsonSerializer.Deserialize<Dto>("{}")!;
Console.WriteLine(d.Name is null);   // Output: True  despite 'string' (non-null) annotation

static void Save(string? s)
{
    ArgumentNullException.ThrowIfNull(s);   // runtime guard + flow analysis: s is non-null after
    Console.WriteLine(s.Length);
}
```

**Interview traps**
- NRT doesn't add runtime null checks.
- `!` doesn't "fix" anything, it hides a warning.
- `T?` unconstrained is not `Nullable<T>`.
- Non-nullable arrays/serialized objects can contain nulls.
- Enabling NRT changes EF Core column nullability on next migration.

**Cheat sheet**
- Compile-time only; `?` = may be null; `!` = suppress.
- Contracts: NotNullWhen / MaybeNullWhen / MemberNotNull / NotNullIfNotNull.
- `required` for must-set members; `= null!` for framework-populated.
- Validate at boundaries: `ArgumentNullException.ThrowIfNull`.
- `where T : struct` -> Nullable<T>; `where T : class` -> annotated ref.

---

## Topic 13: C# 9-13 Feature Tour

**Study notes**

| Version (.NET) | Headline features |
|---|---|
| C# 9 (.NET 5) | records, `init` accessors, top-level statements, target-typed `new()`, `and/or/not` patterns, relational patterns, `static` lambdas, module initializers, covariant returns, native ints |
| C# 10 (.NET 6) | file-scoped namespaces, `global using`, record structs, extended property patterns, const interpolated strings, lambda natural types/attributes, `CallerArgumentExpression`, `with` on structs |
| C# 11 (.NET 7) | raw string literals `"""`, list patterns, `required` members, generic math (static abstract interface members), UTF-8 literals `"..."u8`, `file` types, auto-default structs, `ref` fields/`scoped`, newlines in interpolation holes |
| C# 12 (.NET 8) | primary constructors (all classes/structs), collection expressions `[..]`, inline arrays, `ref readonly` params, default lambda params, alias any type (`using P = (int X, int Y);`) |
| C# 13 (.NET 9) | `params` collections (`params ReadOnlySpan<T>` etc.), `System.Threading.Lock`, `\e` escape, `allows ref struct`, ref locals in async (non-spanning), partial properties/indexers, implicit `^` index in initializers |
| C# 14 (.NET 10) | `field` keyword in properties, extension members, null-conditional assignment (`a?.b = x`) |

**Ladder**

### 13.1 [Easy] What are top-level statements, file-scoped namespaces and global usings?
**Answer**
Top-level statements (C# 9) let `Program.cs` contain code without a `Main` class; the compiler synthesises `Program.Main`. File-scoped namespaces (C# 10) `namespace X;` remove one indentation level for the whole file; `global using` (C# 10) applies a using to the whole project (and `ImplicitUsings` generates common ones).

**Explanation**
Only one file per project may have top-level statements; they can use `await`, args are available as `args`, and return an exit code via `return 1;`. Types declared in that file must come after the statements. These are purely compile-time conveniences.

**Code**
```csharp
// Program.cs
global using System.Text;          // must precede other usings/statements in a file
namespace Demo;                    // file-scoped (for type declarations in a normal file)

public static class Greeter
{
    public static string Hi(string n) => $"Hi {n}";
}
// In a separate Program.cs with top-level statements:
//   Console.WriteLine(Demo.Greeter.Hi(args.Length > 0 ? args[0] : "world")); // Output: Hi world
```

### 13.2 [Follow-up] Explain `required`, `init`, raw string literals and UTF-8 literals.
**Answer**
`init` allows setting a property only during object initialisation (immutability after). `required` (C# 11) forces callers to set the member in an object initializer (CS9035 otherwise). Raw string literals `"""..."""` hold arbitrary text without escaping; UTF-8 literals `"abc"u8` produce a `ReadOnlySpan<byte>` with no allocation.

**Explanation**
`required` can be bypassed by a constructor marked `[SetsRequiredMembers]`. For raw strings, the closing `"""` indentation is stripped from every line; use `$$"""` with `{{expr}}` when the text contains `{ }` (JSON). The delimiter quotes count must exceed any run of quotes inside.

**Code**
```csharp
public class User
{
    public required string Name { get; init; }
    public int Age { get; init; }
}
var u = new User { Name = "Ann", Age = 30 };   // new User { Age = 1 } -> error CS9035

int id = 7;
string json = $$"""
    {"id": {{id}}, "tags": ["a","b"]}
    """;
Console.WriteLine(json);        // Output: {"id": 7, "tags": ["a","b"]}

ReadOnlySpan<byte> bytes = "abc"u8;
Console.WriteLine(bytes.Length); // Output: 3
```

### 13.3 [Medium] What are primary constructors (C# 12) and how do they differ from records' primary constructors?
**Answer**
On classes/structs, `class C(int x)` declares constructor parameters in scope for the whole type; the compiler *captures* them into hidden mutable fields if used in members. Unlike records, no public properties are generated. Records still synthesise `init` properties from their primary-constructor parameters.

**Explanation**
Parameters are not `readonly` and are not members: `this.x` doesn't exist and they can be reassigned, which is a common bug. If you use a parameter to initialise a field/property only, no capture occurs. Compiler warns (CS9124) if a parameter is both captured and passed to the base constructor. Good for DI: `class Svc(IRepo repo)`. Add validation in field initializers: `private readonly IRepo _repo = repo ?? throw ...`.

**Code**
```csharp
class Counter(int start)
{
    private int _n = start;                  // initialiser use: not captured
    public int Next() => _n++;
    public int Start => start;               // captured -> hidden field
    public void Reset() => start = 0;        // legal! parameter is mutable
}
var c = new Counter(5);
Console.WriteLine(c.Next()); // Output: 5
Console.WriteLine(c.Next()); // Output: 6
c.Reset();
Console.WriteLine(c.Start);  // Output: 0
```

### 13.4 [Hard] How do collection expressions work (target typing, spread, builder)?
**Answer**
`[1, 2, 3]` is target-typed: the compiler builds whatever the target is - array, `List<T>`, `Span<T>`, `ReadOnlySpan<T>`, `IEnumerable<T>`/`IReadOnlyList<T>`/`ICollection<T>`/`IList<T>`, `ImmutableArray<T>` (via `[CollectionBuilder]`), or any type with a collection initializer pattern. `..x` spreads another collection into it. It has no natural type in C# 12/13, so `var x = [1];` is an error.

**Explanation**
The compiler picks efficient strategies (stack/inline buffers for `Span` targets, exact-size array when it knows the count, `CollectionsMarshal.SetCount` for `List<T>`). `[]` is an empty value (for arrays, `Array.Empty<T>()`; for interfaces, possibly a shared empty instance). Custom collections opt in with `[CollectionBuilder(typeof(Builder), nameof(Builder.Create))]` whose method takes `ReadOnlySpan<T>`. Dictionaries are not supported by collection expressions in C# 12/13.

**Code**
```csharp
int[] a = [1, 2, 3];
List<int> l = [.. a, 4, 5];
ReadOnlySpan<int> s = [10, 20];
IEnumerable<string> e = ["x", "y"];
int[][] jag = [[1], [2, 3]];
Console.WriteLine(string.Join(",", l));   // Output: 1,2,3,4,5
Console.WriteLine(s.Length + jag[1].Length); // Output: 4
// var bad = [1, 2];   // error: no natural type in C# 12/13
```

### 13.5 [Tricky] `params` collections and the new `Lock` type (C# 13) - what changes and what gotchas exist?
**Answer**
`params` can now be applied to `Span<T>`, `ReadOnlySpan<T>`, `IEnumerable<T>`, `List<T>` etc., not just arrays - `params ReadOnlySpan<int>` lets callers pass loose arguments with no heap allocation. `System.Threading.Lock` (.NET 9) is a dedicated lock type: `lock(obj)` on a `Lock`-typed expression compiles to `EnterScope()` instead of `Monitor`.

**Explanation**
Overload resolution prefers span-based `params` over array-based. Gotchas: C# 13/.NET 9 required; if you cast a `Lock` to `object` (or lock on it via a variable typed `object`), you silently get `Monitor` semantics; `Lock` isn't re-entrant across different threads and `lock` inside `async` code still can't contain `await`. Use `Lock.EnterScope()` in a `using` for scoped locking.

**Code**
```csharp
// Requires C# 13 / .NET 9
static int Sum(params ReadOnlySpan<int> xs)
{
    int t = 0; foreach (var x in xs) t += x; return t;
}
Console.WriteLine(Sum(1, 2, 3)); // Output: 6   (no array allocation)

class Counter
{
    private readonly Lock _gate = new();
    private int _n;
    public void Inc() { lock (_gate) { _n++; } }   // lowers to _gate.EnterScope()
}
```

**Interview traps**
- Primary-constructor parameters are mutable captured fields, not properties.
- `required` doesn't stop `default`/reflection/deserializer bypass (STJ honours it from .NET 7).
- Raw-string indentation rules: closing `"""` column defines stripped whitespace.
- Collection expressions lack a natural type (before C# 14 improvements) - `var` fails.
- A `Lock` typed as `object` is just a Monitor.

**Cheat sheet**
- C#9 records/init/top-level; 10 file-scoped ns/global using; 11 raw strings/list patterns/required; 12 primary ctor/collection expr; 13 params collections/Lock.
- `required` + `init` = enforced immutable construction.
- `[..a, x]` spread; `u8` literals = ReadOnlySpan<byte>.
- Primary ctor on class: capture, no properties.
- `params ReadOnlySpan<T>` = alloc-free varargs.

---

## Topic 14: Reflection, Attributes, dynamic / DLR

**Study notes**

- **Reflection** reads metadata at runtime: `Type` (`typeof(T)`, `obj.GetType()`), `Assembly`, `MethodInfo`, `PropertyInfo`, `FieldInfo`, `ConstructorInfo`, `Activator.CreateInstance`. `BindingFlags` controls what is returned: default `GetMethod/GetProperty` = **public instance + static** only; private needs `NonPublic | Instance`.
- Cost: `Invoke`/`GetValue` allocate (boxing/`object[]`) and are slow (tens-hundreds of ns, improved in .NET 7+ via emit-based invoker). Cache `MethodInfo`; for repeated use convert to a **delegate** (`CreateDelegate`), a **compiled expression tree**, or `DynamicMethod`/source generators.
- **Attributes**: metadata annotations. Derive from `System.Attribute`; constrain with `[AttributeUsage(AttributeTargets..., AllowMultiple, Inherited)]`; constructor args and named properties must be compile-time constants (primitives, string, `Type`, enums, arrays thereof). Instances are created when retrieved (`GetCustomAttribute<T>()`), not when the type loads. Naming `XAttribute` used as `[X]`.
- **dynamic**: static type `dynamic` defers member binding to runtime through the **DLR** (Dynamic Language Runtime; `Microsoft.CSharp.RuntimeBinder`, call-site caching). Failures -> `RuntimeBinderException`. `ExpandoObject` (add members at runtime; implements `IDictionary<string, object?>`), `DynamicObject` (override `TryGetMember`, `TrySetMember`, `TryInvokeMember`).
- **Trimming/Native AOT**: reflection over unknown types is trimmer-unsafe; use `[DynamicallyAccessedMembers]`, `[RequiresUnreferencedCode]`, or source generators.

**Ladder**

### 14.1 [Easy] How do you inspect and invoke members with reflection?
**Answer**
Get a `Type`, then find members with `GetProperty/GetMethod/GetField` and use `GetValue/SetValue/Invoke`. Use `BindingFlags` to reach non-public or static members. `Activator.CreateInstance` creates objects from a `Type`.

**Explanation**
Reflection results are objects, so value types get boxed and return types need casts. Missing members return `null` - handle it. Reflection ignores accessibility only for non-public if you ask via `BindingFlags.NonPublic` (full-trust only; fine in normal apps, but breaks encapsulation and trimming).

**Code**
```csharp
using System.Reflection;

var t = typeof(Person);
object p = Activator.CreateInstance(t)!;
t.GetProperty("Name")!.SetValue(p, "Ann");
Console.WriteLine(t.GetMethod("Greet")!.Invoke(p, new object[] { "Bob" })); // Output: Hi Bob, I'm Ann
var f = t.GetField("_secret", BindingFlags.NonPublic | BindingFlags.Instance)!;
Console.WriteLine(f.GetValue(p));                                            // Output: 7

class Person
{
    private int _secret = 7;
    public string Name { get; set; } = "";
    public string Greet(string who) => $"Hi {who}, I'm {Name}";
}
```

### 14.2 [Follow-up] How do you define and read a custom attribute (e.g., a validation attribute)?
**Answer**
Derive from `Attribute`, decorate with `[AttributeUsage]`, add constructor/properties, apply it with `[MaxLen(5)]`, then read via `GetCustomAttribute<T>()` on a `MemberInfo`. The attribute does nothing by itself - some code (framework, your reflection or a source generator) must act on it.

**Explanation**
`Inherited = true` (default) means derived classes' members inherit it (for members, only when overriding). `AllowMultiple` controls repeating. Retrieval constructs a new attribute instance each call - cache the results. ASP.NET model validation, EF Core configuration (`[Key]`, `[MaxLength]`), and test frameworks are all attribute-driven.

**Code**
```csharp
using System.Reflection;

[AttributeUsage(AttributeTargets.Property, AllowMultiple = false, Inherited = true)]
public sealed class MaxLenAttribute(int max) : Attribute
{
    public int Max { get; } = max;
}

public class Dto { [MaxLen(3)] public string Name { get; set; } = "Annabelle"; }

var d = new Dto();
foreach (var prop in typeof(Dto).GetProperties())
{
    var a = prop.GetCustomAttribute<MaxLenAttribute>();
    if (a is not null && prop.GetValue(d) is string s && s.Length > a.Max)
        Console.WriteLine($"{prop.Name} too long");   // Output: Name too long
}
```

### 14.3 [Medium] Why is reflection slow, and how do you make repeated calls fast?
**Answer**
Each `Invoke` does argument validation, boxing, array allocation and security/binding checks. Cache the `MethodInfo`/`PropertyInfo`, then create a strongly typed delegate (`Delegate.CreateDelegate`, compiled `Expression`, or `DynamicMethod`/IL emit) so repeated calls run near direct-call speed; or avoid reflection with a source generator.

**Explanation**
Order of speed (fast to slow): direct call / generated code > cached typed delegate/compiled expression > `MethodInfo.Invoke` (with .NET 7+ faster IL-emitted invokers after warm-up) > repeated `GetMethod` + `Invoke`. Lookups (`GetMethod`) are the most expensive part, so never do them in a loop. `Expression.Compile` has a one-off compile cost (ms) so it pays off only on reuse.

**Code**
```csharp
using System.Linq.Expressions;
using System.Reflection;

MethodInfo mi = typeof(Person).GetMethod("Greet")!;

// Open-instance delegate: first arg is 'this'
var del = (Func<Person, string, string>)Delegate.CreateDelegate(typeof(Func<Person, string, string>), mi);
Console.WriteLine(del(new Person { Name = "Ann" }, "Bob")); // Output: Hi Bob, I'm Ann

// Compiled getter via expression tree
var param = Expression.Parameter(typeof(Person));
Func<Person, string> getName =
    Expression.Lambda<Func<Person, string>>(Expression.Property(param, "Name"), param).Compile();
Console.WriteLine(getName(new Person { Name = "Zed" }));    // Output: Zed

class Person { public string Name { get; set; } = ""; public string Greet(string who) => $"Hi {who}, I'm {Name}"; }
```

### 14.4 [Hard] How does `dynamic` work? ExpandoObject vs DynamicObject, and what are the costs?
**Answer**
A `dynamic` expression compiles to a *call site*; at first run the C# runtime binder resolves the member using the runtime type and caches a rule, so later calls with the same type are fast-ish. `ExpandoObject` is a ready-made dynamic bag; `DynamicObject` lets you implement custom dynamic behaviour. Costs: no compile-time safety, binder startup (hundreds of ms first use), and incompatibility with Native AOT/trimming.

**Explanation**
The DLR also hosts IronPython etc. Extension methods and lambdas can't be called on `dynamic` directly. Result of any dynamic operation is `dynamic` (viral). Use cases: COM interop (Excel), JSON-ish loose objects, interop with dynamic languages, generic-math workarounds in old code. Prefer typed models/`System.Text.Json.Nodes.JsonObject` otherwise.

**Code**
```csharp
using System.Dynamic;
using Microsoft.CSharp.RuntimeBinder;

dynamic e = new ExpandoObject();
e.Name = "Ann";
e.Add = (Func<int, int, int>)((a, b) => a + b);
Console.WriteLine(e.Add(2, 3));                                   // Output: 5
Console.WriteLine(((IDictionary<string, object?>)e).ContainsKey("Name")); // Output: True

dynamic x = 5;
try { x.Foo(); }
catch (RuntimeBinderException) { Console.WriteLine("binder error"); } // Output: binder error

class Bag : DynamicObject
{
    private readonly Dictionary<string, object?> _d = new();
    public override bool TryGetMember(GetMemberBinder b, out object? r) => _d.TryGetValue(b.Name, out r);
    public override bool TrySetMember(SetMemberBinder b, object? v) { _d[b.Name] = v; return true; }
}
```

**Gotchas / follow-ups**
- `dynamic` and `var` differ: `var` is static-typed inference; `dynamic` is late-bound.
- Overload resolution happens at runtime based on runtime types - can pick a different overload than static typing would.

### 14.5 [Tricky] Reflection gotchas: BindingFlags defaults, inheritance, nullable/generic types, trimming/AOT.
**Answer**
`GetMethod("x")` with no flags finds public members only (null if not found) and throws `AmbiguousMatchException` when overloads make the name ambiguous; private fields of a base class aren't returned by `GetField` on the derived type; `typeof(int?)` is `Nullable<int>` while a boxed `int?` becomes a boxed `int` (or null); and trimmed/AOT apps may lose members reflection expects unless annotated.

**Explanation**
`GetProperties()` returns public instance + inherited public; for private base members walk `BaseType`. `Attribute.GetCustomAttribute` honours `Inherited`, while `MemberInfo.GetCustomAttributes(false)` ignores inheritance. Open generics need `MakeGenericType`/`MakeGenericMethod`; with AOT, instantiations unseen at compile time may not exist. Use `[DynamicallyAccessedMembers(DynamicallyAccessedMemberTypes.PublicProperties)]` on the `Type` parameter so the trimmer keeps what you reflect on.

**Code**
```csharp
using System.Reflection;

int? maybe = 5;
object boxed = maybe;
Console.WriteLine(boxed.GetType());                    // Output: System.Int32  (not Nullable<int>)
Console.WriteLine(typeof(int?).Name);                  // Output: Nullable`1

class Base { private int _hidden = 1; }
class Derived : Base { }
var f = typeof(Derived).GetField("_hidden", BindingFlags.NonPublic | BindingFlags.Instance);
Console.WriteLine(f is null);                          // Output: True  (private base field not visible)
Console.WriteLine(typeof(Base).GetField("_hidden", BindingFlags.NonPublic | BindingFlags.Instance) is null); // Output: False
```

**Interview traps**
- `GetMethod` defaults to public only; ambiguous names throw.
- Attribute instances are created on retrieval; constructor logic runs each time.
- Repeated `GetMethod`/`Invoke` in hot paths is a perf bug.
- `dynamic` defeats compile-time checks and AOT.
- Reflection on private members breaks with trimming/AOT.

**Cheat sheet**
- `typeof`/`GetType` -> members -> `Invoke/GetValue`.
- Cache `MethodInfo`; convert to delegate/expression for speed.
- Attribute = metadata; needs code to act on it.
- `dynamic` = DLR runtime binding; Expando/DynamicObject for custom bags.
- Prefer source generators for AOT/trim friendliness.

---

## Topic 15: Source Generators & Roslyn Analyzers

**Study notes**

- **Roslyn** exposes the compiler as an API (syntax trees, semantic model, symbols, `Compilation`). Two extensibility points run *during compilation* (and in the IDE):
  - **Analyzers** (`DiagnosticAnalyzer`): inspect code and report **diagnostics** (warnings/errors, IDs like `CA2007`, `IDE0051`); optional **code fixes** (`CodeFixProvider`). Severity controlled via `.editorconfig` / `<TreatWarningsAsErrors>`.
  - **Source generators**: produce **additional C# source** added to the compilation (can't modify existing code - hence `partial` classes). Use **`IIncrementalGenerator`** (Roslyn 4.x) rather than legacy `ISourceGenerator`.
- Both ship as **netstandard2.0** analyzer assemblies, referenced with `OutputItemType="Analyzer" ReferenceOutputAssembly="false"` or packed under `analyzers/dotnet/cs` in a NuGet package.
- **Incremental pipeline**: `context.SyntaxProvider` (`CreateSyntaxProvider`, `ForAttributeWithMetadataName`) -> `Select/Where/Combine/Collect` -> `RegisterSourceOutput`. Each step is cached by equality of its output - keep models **value-equatable** (records, immutable arrays wrappers) and never store `ISymbol`/`SyntaxNode`/`Compilation` in them.
- **In-box generators**: `[GeneratedRegex]`, `[LoggerMessage]`, `[JsonSerializable]` (System.Text.Json), `[LibraryImport]`, configuration binder generator, `[GeneratedComInterface]`. Benefits: no reflection, AOT/trim-friendly, faster startup.

**Ladder**

### 15.1 [Easy] What is a source generator and why use one instead of reflection?
**Answer**
A source generator runs at compile time, inspects your code and emits extra C# files that are compiled into your assembly. It replaces runtime reflection/emit with generated, strongly typed code: faster startup, no runtime overhead, and compatible with trimming and Native AOT.

**Explanation**
Typical uses: serialisers, DI registration, mapping (Mapperly), logging, regex compilation, INotifyPropertyChanged boilerplate. The generated code is visible (`EmitCompilerGeneratedFiles`, or Analyzers node in VS). Generators can only *add* code; types that need generated members are declared `partial`.

**Code**
```csharp
using System.Text.RegularExpressions;

public static partial class Rx
{
    [GeneratedRegex(@"\d+", RegexOptions.CultureInvariant)]
    public static partial Regex Digits();      // implementation generated at compile time
}
Console.WriteLine(Rx.Digits().Match("ab12cd").Value); // Output: 12
```

### 15.2 [Follow-up] Source generator vs analyzer vs code fix - how do they differ?
**Answer**
An analyzer *reports* diagnostics about existing code, a code fix *edits* the code to resolve a diagnostic (when the user applies it), and a generator *adds* new source to the compilation. Generators never change user-written code; analyzers/fixes never add compile-time code on their own.

**Explanation**
Analyzer skeleton: derive from `DiagnosticAnalyzer`, declare `SupportedDiagnostics` (a `DiagnosticDescriptor` with ID, message, category, default severity), and in `Initialize` call `ConfigureGeneratedCodeAnalysis`, `EnableConcurrentExecution`, then `RegisterSyntaxNodeAction`/`RegisterOperationAction`/`RegisterSymbolAction`. Analyzer packages are the way to enforce team rules (e.g. "no `DateTime.Now`"). Built-in rule sets: .NET analyzers (CAxxxx), IDE rules (IDExxxx), StyleCop.

**Code**
```csharp
// Analyzer project (netstandard2.0, Microsoft.CodeAnalysis.CSharp)
using System.Collections.Immutable;
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp;
using Microsoft.CodeAnalysis.CSharp.Syntax;
using Microsoft.CodeAnalysis.Diagnostics;

[DiagnosticAnalyzer(LanguageNames.CSharp)]
public sealed class NoDateTimeNowAnalyzer : DiagnosticAnalyzer
{
    private static readonly DiagnosticDescriptor Rule = new(
        "DEMO001", "Avoid DateTime.Now", "Use TimeProvider instead of DateTime.Now",
        "Usage", DiagnosticSeverity.Warning, isEnabledByDefault: true);

    public override ImmutableArray<DiagnosticDescriptor> SupportedDiagnostics => [Rule];

    public override void Initialize(AnalysisContext context)
    {
        context.ConfigureGeneratedCodeAnalysis(GeneratedCodeAnalysisFlags.None);
        context.EnableConcurrentExecution();
        context.RegisterSyntaxNodeAction(ctx =>
        {
            var access = (MemberAccessExpressionSyntax)ctx.Node;
            if (access.ToString() == "DateTime.Now")
                ctx.ReportDiagnostic(Diagnostic.Create(Rule, access.GetLocation()));
        }, SyntaxKind.SimpleMemberAccessExpression);
    }
}
```

### 15.3 [Medium] Sketch an incremental generator that targets an attribute.
**Answer**
Implement `IIncrementalGenerator.Initialize`, use `context.SyntaxProvider.ForAttributeWithMetadataName("Ns.MyAttribute", predicate, transform)` to find annotated types cheaply, transform to a small value-equatable model, then `RegisterSourceOutput` to emit source with `AddSource`.

**Explanation**
`ForAttributeWithMetadataName` is far more efficient than scanning every syntax node. The predicate runs on syntax only (cheap, must be fast), the transform uses the semantic model. Emit hint names that are unique (`Name.g.cs`). For the attribute itself, emit it via `RegisterPostInitializationOutput` so consumers don't need a reference. Handle global namespace (`ContainingNamespace.IsGlobalNamespace`).

**Code**
```csharp
// Generator project: netstandard2.0, <IsRoslynComponent>true</IsRoslynComponent>
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp.Syntax;

[Generator]
public sealed class AutoToStringGenerator : IIncrementalGenerator
{
    private sealed record Model(string Ns, string Name);   // value-equatable -> cacheable

    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        context.RegisterPostInitializationOutput(static ctx => ctx.AddSource("AutoToStringAttribute.g.cs",
            "namespace Gen { [System.AttributeUsage(System.AttributeTargets.Class)] internal sealed class AutoToStringAttribute : System.Attribute { } }"));

        var models = context.SyntaxProvider.ForAttributeWithMetadataName(
            "Gen.AutoToStringAttribute",
            static (node, _) => node is ClassDeclarationSyntax,
            static (ctx, _) => new Model(ctx.TargetSymbol.ContainingNamespace.ToDisplayString(), ctx.TargetSymbol.Name));

        context.RegisterSourceOutput(models, static (spc, m) =>
            spc.AddSource($"{m.Name}.g.cs",
                $"namespace {m.Ns} {{ partial class {m.Name} {{ public override string ToString() => \"{m.Name}\"; }} }}"));
    }
}
// Consumer: [Gen.AutoToString] public partial class Order { }   // new Order().ToString() -> "Order"
```

### 15.4 [Hard] Why do incremental generators break caching, and how do you keep the IDE responsive?
**Answer**
The pipeline reuses a step's cached output only if the new output is *equal* to the old. If your model holds `SyntaxNode`, `ISymbol`, `Compilation` or mutable/reference-equal collections, equality always fails, so the generator re-runs on every keystroke and the IDE lags. Use small immutable records, and wrap lists in a value-equatable `EquatableArray<T>`.

**Explanation**
Also avoid `Collect()` of large inputs early, avoid `compilation.GetSemanticModel` in predicates, keep predicates syntactic, and don't do I/O or reflection. Test caching with `GeneratorDriver` and `TrackIncrementalGeneratorSteps` (verify `IncrementalStepRunReason.Cached/Unchanged`). Snapshot testing (Verify) is common. Generators must be deterministic and not depend on file system/time.

**Code**
```csharp
using System.Collections.Immutable;

// A cache-friendly model: record + value-equality over a collection
public readonly record struct PropModel(string Name, string Type);

public sealed record ClassModel(string Name, EquatableArray<PropModel> Props);

// Minimal value-equatable array wrapper (what real generators vendor in)
public readonly struct EquatableArray<T>(ImmutableArray<T> items) : IEquatable<EquatableArray<T>>
    where T : IEquatable<T>
{
    private readonly ImmutableArray<T> _items = items;
    public bool Equals(EquatableArray<T> other) =>
        _items.AsSpan().SequenceEqual(other._items.AsSpan());
    public override bool Equals(object? obj) => obj is EquatableArray<T> o && Equals(o);
    public override int GetHashCode()
    {
        var h = new HashCode();
        foreach (var i in _items) h.Add(i);
        return h.ToHashCode();
    }
}
```

**Gotchas / follow-ups**
- `ISourceGenerator` (v1) is legacy - it reruns on every edit.
- Follow-up: debugging - `Debugger.Launch()` in generator, or run under a unit test with `CSharpGeneratorDriver`.

### 15.5 [Tricky] Limitations and gotchas of generators and analyzers.
**Answer**
Generators can't modify existing source, can't see other generators' output (they all run against the original compilation), must target `netstandard2.0`, and pin you to a specific Roslyn version range. A crashing analyzer/generator produces a warning (AD0001 / CS8785) and silently disables itself - tests must assert it works.

**Explanation**
Everything partial: the target class must be `partial` or the generated half fails (CS0260/duplicate). Hint names must be unique per generator or you get an exception. Generated files appear under `obj/` with `EmitCompilerGeneratedFiles=true`. Analyzers don't flow transitively to consumers unless packed as an analyzer package (set `PrivateAssets` appropriately; `DevelopmentDependency`). Source generators and interceptors (experimental) can change behaviour of calls but are opt-in and not stable.

**Code**
```xml
<!-- Consumer .csproj: reference a generator project locally -->
<ItemGroup>
  <ProjectReference Include="..\MyGenerator\MyGenerator.csproj"
                    OutputItemType="Analyzer" ReferenceOutputAssembly="false" />
</ItemGroup>
<PropertyGroup>
  <EmitCompilerGeneratedFiles>true</EmitCompilerGeneratedFiles>
</PropertyGroup>
```

**Interview traps**
- Generators add code; they don't rewrite yours.
- Storing symbols/syntax nodes in the model kills incremental caching.
- Analyzer/generator projects must target `netstandard2.0`.
- Forgetting `partial` on the target type.
- A generator exception disables it (look for CS8785/AD0001).

**Cheat sheet**
- Analyzer = report; CodeFix = edit; Generator = add source.
- Use `IIncrementalGenerator` + `ForAttributeWithMetadataName`.
- Models: value-equatable records only.
- In-box: GeneratedRegex, LoggerMessage, JsonSerializable, LibraryImport.
- Ship as netstandard2.0 analyzer in `analyzers/dotnet/cs`.

---
## Topic 16: Unsafe Code, Pointers, P/Invoke

**Study notes**

- `unsafe` context (needs `<AllowUnsafeBlocks>true</AllowUnsafeBlocks>`) allows pointers (`int*`, `&x`, `*p`, `p->field`, pointer arithmetic), `stackalloc` to pointers, and function pointers (`delegate* managed<int,int>`, `delegate* unmanaged<...>`). Pointers to managed (reference-containing) types are disallowed; only *unmanaged* types (primitives, enums, pointers, structs of those).
- **`fixed`** pins a managed object (array, string, field) so the GC can't move it while you hold a raw pointer. `GCHandle.Alloc(obj, GCHandleType.Pinned)` for longer pins; prefer the POH (`GC.AllocateArray(pinned: true)`).
- Safer modern alternatives: `Span<T>`, `ref` locals/returns, `System.Runtime.CompilerServices.Unsafe` (`Unsafe.As<T>`, `Unsafe.Add`, `Unsafe.SizeOf<T>`, `Unsafe.AsRef`), `MemoryMarshal`, `NativeMemory.Alloc/Free` (.NET 6), `nint/nuint`.
- **P/Invoke** = call native (unmanaged) exports. `[DllImport]` (runtime-generated marshalling stub) vs **`[LibraryImport]`** (.NET 7+, source-generated marshalling, AOT/trim friendly; method must be `static partial`, requires AllowUnsafeBlocks). Key settings: `EntryPoint`, `StringMarshalling` / `CharSet`, `SetLastError = true` (then `Marshal.GetLastPInvokeError()`), calling convention.
- **Blittable** types (same layout managed/native: ints, floats, `IntPtr`, structs of blittables) need no copying; `bool`, `char`, `string`, arrays of those need marshalling. Wrap handles in `SafeHandle`. Callbacks: pass a delegate (keep it alive!) or `delegate* unmanaged` with `[UnmanagedCallersOnly]`.

**Ladder**

### 16.1 [Easy] What does `unsafe` enable and when would you use it?
**Answer**
It allows raw pointers and pointer arithmetic, bypassing bounds checks and type safety. Use it for interop, high-performance parsing or serialization where `Span<T>` is insufficient - but first try `Span<T>`/`Unsafe`, which are safer and usually as fast.

**Explanation**
Unsafe code is not "unverified" in .NET Core (no code access security), but memory-safety errors (buffer overrun, dangling pointers) become possible. It requires the compiler flag, and can't be used in iterators/async methods (pointers can't be locals across suspension).

**Code**
```csharp
// <AllowUnsafeBlocks>true</AllowUnsafeBlocks>
unsafe
{
    int x = 10;
    int* p = &x;
    *p = 20;
    Console.WriteLine(x);   // Output: 20

    int* arr = stackalloc int[3] { 1, 2, 3 };
    Console.WriteLine(*(arr + 2)); // Output: 3   (pointer arithmetic scales by sizeof(int))
}
```

### 16.2 [Follow-up] What does `fixed` do and why is it needed?
**Answer**
Managed objects can be moved by the compacting GC, so a raw pointer into one could dangle. `fixed` pins the object for the duration of the block and gives you a stable pointer, releasing the pin at the end of the block.

**Explanation**
Pinning has a cost: it fragments the gen0/1 heap and blocks compaction around that object, so keep scopes short or use the POH for long-lived pins. `fixed` on an empty array or null yields a null pointer - check length first. For strings: `fixed (char* c = str)` points to the first char (null-terminated).

**Code**
```csharp
int[] data = { 1, 2, 3 };
unsafe
{
    fixed (int* p = data)
    {
        for (int i = 0; i < data.Length; i++) p[i] *= 2;
    }
}
Console.WriteLine(string.Join(",", data)); // Output: 2,4,6

int[] empty = Array.Empty<int>();
unsafe { fixed (int* q = empty) Console.WriteLine(q == null); } // Output: True
```

### 16.3 [Medium] `[DllImport]` vs `[LibraryImport]`: what's the difference and how do you call a Windows API?
**Answer**
`DllImport` generates the marshalling stub at runtime via IL emit; `LibraryImport` is a source generator that produces the stub at compile time, so it is faster to start, trimmable and Native-AOT-compatible. You declare a `partial` method and the generator fills in the body.

**Explanation**
Use `StringMarshalling.Utf16` for `...W` APIs (`StringMarshalling.Utf8` for Unix). Set `SetLastError = true` and read `Marshal.GetLastPInvokeError()` instead of `GetLastWin32Error` in new code. Resolve library names portably with `NativeLibrary`/`DllImportResolver`. Windows APIs are not portable - guard with `OperatingSystem.IsWindows()`.

**Code**
```csharp
using System.Runtime.InteropServices;

static partial class Native
{
    [LibraryImport("kernel32.dll", EntryPoint = "GetCurrentProcessId")]
    public static partial uint GetCurrentProcessId();

    [LibraryImport("kernel32.dll", EntryPoint = "GetModuleHandleW",
        StringMarshalling = StringMarshalling.Utf16, SetLastError = true)]
    public static partial nint GetModuleHandle(string? moduleName);
}

if (OperatingSystem.IsWindows())
{
    Console.WriteLine(Native.GetCurrentProcessId() == (uint)Environment.ProcessId); // Output: True
    Console.WriteLine(Native.GetModuleHandle("kernel32.dll") != 0);                // Output: True
}
```

### 16.4 [Hard] Marshalling pitfalls: blittable vs non-blittable, struct layout, callbacks and delegate lifetime.
**Answer**
Non-blittable types (`bool`, `char`, `string`, arrays of them) are copied/converted; getting sizes or `bool` width wrong corrupts memory. Structs must match native layout (`[StructLayout(LayoutKind.Sequential)]`, `Pack`, field order, `nint` for pointers). A managed delegate passed as a callback must be kept alive (static field) until native code stops calling it, otherwise the GC collects the thunk and the app crashes.

**Explanation**
`bool` marshals as 4-byte BOOL by default; use `[MarshalAs(UnmanagedType.U1)]` for C++ `bool`. `long` is 32-bit in C on Windows (`LONG`) but 64-bit on Linux - use fixed-size types (`int`, `long`) deliberately. Prefer `delegate* unmanaged` + `[UnmanagedCallersOnly]` which has no delegate lifetime issue. Buffers: use `byte[]`/`Span<byte>` pinned automatically for the call duration; pass `StringBuilder` carefully (or `char[]`). Always free native-allocated memory using the matching allocator.

**Code**
```csharp
using System.Runtime.InteropServices;

[StructLayout(LayoutKind.Sequential, Pack = 1)]
struct Header { public byte Kind; public int Length; }   // Pack=1 -> size 5, not 8

Console.WriteLine(Marshal.SizeOf<Header>()); // Output: 5

delegate bool EnumProc(nint hwnd, nint lParam);
static class Win
{
    [DllImport("user32.dll")] public static extern bool EnumWindows(EnumProc cb, nint lParam);
    public static readonly EnumProc Keep = (h, l) => true;   // static field keeps the delegate alive
}
```

**Gotchas / follow-ups**
- Passing a lambda directly (`EnumWindows((h,l)=>true, 0)`) is safe for the duration of a *synchronous* call, not for callbacks registered for later.
- Follow-up: `Marshal.AllocHGlobal` must be paired with `FreeHGlobal`; prefer `NativeMemory.Free` for `NativeMemory.Alloc`.

### 16.5 [Tricky] What can go wrong with pinning, pointers and `Unsafe` APIs?
**Answer**
A pointer used after leaving `fixed`/after the GC handle is freed is a dangling pointer; `Unsafe.As` to an incompatible type gives type confusion; `Unsafe.Add` past the end reads other memory; long-lived pins fragment the heap. None of these throw reliably - they corrupt data or crash.

**Explanation**
Never return a pointer obtained inside `fixed`. `Unsafe.AsRef`/`MemoryMarshal.GetReference` skip bounds checks - fine in vetted hot loops, never with unvalidated input. Struct padding and endianness differ across platforms when casting `byte*` to structs; use `BinaryPrimitives` for portable parsing. Pointer comparisons between different allocations are meaningless. `sizeof(T)` for managed types needs `unsafe` and `Unsafe.SizeOf<T>()` gives managed size.

**Code**
```csharp
using System.Buffers.Binary;
using System.Runtime.CompilerServices;

ReadOnlySpan<byte> raw = new byte[] { 0x01, 0x00, 0x00, 0x00 };
Console.WriteLine(BinaryPrimitives.ReadInt32LittleEndian(raw)); // Output: 1  (portable)

int[] a = { 10, 20, 30 };
ref int first = ref a[0];
Console.WriteLine(Unsafe.Add(ref first, 2));   // Output: 30
// Unsafe.Add(ref first, 3) would read past the array: undefined behaviour, no exception
```

**Interview traps**
- `fixed` pins only for the block; the pointer is invalid after.
- Forgetting to keep a callback delegate alive -> intermittent crash.
- `bool`/`char`/`string` are not blittable.
- `LibraryImport` needs `partial` + AllowUnsafeBlocks.
- Pinned objects hurt GC compaction; keep pins short or use the POH.

**Cheat sheet**
- `unsafe` + AllowUnsafeBlocks; pointers only to unmanaged types.
- `fixed` = temporary pin; POH = long-lived pin.
- Prefer Span/Unsafe/MemoryMarshal over raw pointers.
- P/Invoke: `LibraryImport` (source-gen) > `DllImport`; use SafeHandle.
- Mind struct layout, bool width, delegate lifetime.

---

## Topic 17: IL, JIT, Tiered Compilation, ReadyToRun, Native AOT

**Study notes**

```
 C# --Roslyn--> IL + metadata (.dll) --(runtime)--> JIT --> machine code
                                  \--(crossgen)--> ReadyToRun (precompiled native + IL)
                                  \--(ILCompiler)-> Native AOT (self-contained native exe, no JIT)
```
- **IL** (CIL): stack-based, CPU-independent bytecode; inspect with ILSpy, `dotnet ildasm`, SharpLab. Verified by the JIT; metadata describes types.
- **JIT** compiles a method to machine code on its *first call* (via a stub that gets patched). Warm code stays in memory per process.
- **Tiered compilation** (default on): **Tier-0** = fast, minimally optimised (quick start); methods called >= **30 times** (and after a ~100 ms quiet delay) are re-JITted at **Tier-1** (fully optimised) on a background thread. **OSR** (on-stack replacement, .NET 7+) upgrades long-running loops in tier-0 methods. **Dynamic PGO** (default in .NET 8): tier-0 instrumented code collects profile (hot/cold, types seen) to guide tier-1 (guarded devirtualization, inlining, layout).
- **ReadyToRun (R2R)**: `<PublishReadyToRun>true</PublishReadyToRun>` (+ RuntimeIdentifier) precompiles to native code in the assembly -> faster startup; code is slightly lower quality, still re-JITted to tier-1 when hot. Framework assemblies are R2R already.
- **Native AOT**: `<PublishAot>true</PublishAot>` -> self-contained native binary: fastest startup, smallest working set, no JIT needed (can run where JIT is banned, e.g. iOS), but **no runtime code generation** (`Reflection.Emit`, `dynamic`, `Expression.Compile` interpreted), limited reflection, trimming applied, platform-specific build, peak throughput can be lower than JIT with PGO. Compat analyzers: `<IsAotCompatible>true</IsAotCompatible>` / `EnableTrimAnalyzer`.
- Knobs: `TieredCompilation`, `TieredPGO`, `TC_QuickJitForLoops`, `ReadyToRun`; per method `[MethodImpl(MethodImplOptions.AggressiveInlining | NoInlining | AggressiveOptimization)]`.

**Ladder**

### 17.1 [Easy] What happens between `dotnet build` and your code running?
**Answer**
The C# compiler produces IL plus metadata in an assembly. At run time the CLR loads the assembly, and the JIT compiles each method to native machine code the first time it's called. So startup includes JIT time, which tiered compilation and R2R reduce.

**Explanation**
Benefits of IL + JIT: portability across CPUs/OSes, runtime-specific optimisations (the JIT knows the actual CPU: AVX2, etc. and static readonly values), and generics specialisation over value types. Costs: warm-up time and memory for the JIT'd code.

**Code**
```csharp
// C#:
static int Add(int a, int b) => a + b;
// IL (as shown by ILSpy / SharpLab):
//   ldarg.0
//   ldarg.1
//   add
//   ret
Console.WriteLine(Add(2, 3)); // Output: 5
```

### 17.2 [Follow-up] Explain tiered compilation and why it exists.
**Answer**
Tiered compilation trades peak performance for fast startup: methods are first compiled quickly at Tier-0 (unoptimised); once a method is hot (30+ calls) it's recompiled in the background at Tier-1 with full optimisation, and callers are redirected. Loops in cold methods use OSR so they don't stay slow.

**Explanation**
Call counting begins after a 100 ms "tiering delay" that resets whenever new tier-0 activity occurs, so startup bursts aren't distracted by background re-JIT. Consequence for benchmarks/ latency: first requests are slower; warm up services before health checks if latency matters. `TieredCompilation=false` makes everything JIT fully optimised at first call (slower startup, no PGO).

**Code**
```xml
<PropertyGroup>
  <TieredPGO>true</TieredPGO>                         <!-- default in .NET 8 -->
  <TieredCompilationQuickJitForLoops>true</TieredCompilationQuickJitForLoops>
  <!-- <TieredCompilation>false</TieredCompilation> -->
</PropertyGroup>
```
```csharp
using System.Runtime.CompilerServices;

[MethodImpl(MethodImplOptions.AggressiveOptimization)]   // skips tier-0, jitted fully optimised (and without PGO)
static int Hot(int x) => x * 2;
Console.WriteLine(Hot(21)); // Output: 42
```

### 17.3 [Medium] ReadyToRun vs Native AOT vs JIT - what are the trade-offs and when to pick which?
**Answer**
JIT: best peak performance and full dynamic features, slowest startup. R2R: JIT app with precompiled code - better startup, bigger files, still needs the runtime. Native AOT: no runtime/JIT, fastest startup and lowest memory, small self-contained binary - but restricted reflection/dynamic code and platform-specific builds. Pick AOT for CLIs, serverless/containers with cold starts, and microservices that are AOT-compatible.

**Explanation**
Table:

| | JIT | R2R | Native AOT |
|---|---|---|---|
| Startup | slowest | faster | fastest |
| Peak perf | best (PGO) | good (re-JIT hot) | good, no dynamic PGO |
| Reflection/Emit/dynamic | full | full | limited / none |
| Needs runtime installed | yes | yes (or self-contained) | no |
| Size | small IL | larger | small-ish single file |

ASP.NET Core supports AOT mainly with Minimal APIs (+ `CreateSlimBuilder`, JSON source generation); MVC, Razor runtime compilation and EF Core's full feature set are not AOT-ready (as of .NET 8).

**Code**
```xml
<PropertyGroup>
  <PublishReadyToRun>true</PublishReadyToRun>
  <RuntimeIdentifier>win-x64</RuntimeIdentifier>
  <SelfContained>true</SelfContained>
</PropertyGroup>
<!-- or, for Native AOT: -->
<!-- <PublishAot>true</PublishAot>  then: dotnet publish -c Release -r win-x64 -->
```
```csharp
using System.Runtime.CompilerServices;
Console.WriteLine(RuntimeFeature.IsDynamicCodeSupported); // Output: True on JIT, False under Native AOT
```

### 17.4 [Hard] How do dynamic PGO, OSR and `static readonly` interact with tiering?
**Answer**
Tier-0 code is instrumented to record branch counts and receiver types; at tier-1 the JIT uses that to inline hot paths, do guarded devirtualization of interface/virtual calls and lay out hot blocks. OSR lets a long-running loop in a tier-0 method transition to optimised code mid-execution. At tier-1, `static readonly` fields are treated as JIT-time constants (since the class is already initialised), enabling dead-branch elimination.

**Explanation**
That's why feature-flag checks (`static readonly bool Enabled`) can be free at tier-1. `AggressiveOptimization` bypasses tiering *and* PGO, so it can be slower than default for polymorphic call sites. R2R code doesn't have instrumentation unless rejitted via tier-1 instr (`TieredPGO_InstrumentOnlyHotCode`). Methods with `stackalloc`, explicit tail calls or loops may follow different tiering paths. Profile before tuning flags.

**Code**
```csharp
static class Features
{
    public static readonly bool Verbose = Environment.GetEnvironmentVariable("VERBOSE") == "1";
}
static void Log(string m)
{
    if (Features.Verbose)           // at tier-1 this check can be folded away if Verbose == false
        Console.WriteLine(m);
}
Log("hello"); // prints nothing unless VERBOSE=1
```

**Gotchas / follow-ups**
- A `static readonly` initialised *after* the method is tier-1 compiled isn't constant-folded; initialisation order matters.
- Follow-up: interface dispatch cost - virtual stub dispatch; PGO guarded devirtualization removes it for monomorphic sites.

### 17.5 [Tricky] Native AOT / trimming gotchas that compile fine but fail at runtime.
**Answer**
Reflection over types the trimmer can't see, `Type.GetType(string)`, `Activator.CreateInstance` for unknown types, `MakeGenericType` over new value-type instantiations, `Reflection.Emit`, `Assembly.LoadFrom`, `dynamic`, Newtonsoft.Json reflection and EF Core runtime model building can all fail or lose members. Fix with source generators, `[DynamicallyAccessedMembers]`, or rd.xml/`TrimmerRootAssembly` hints.

**Explanation**
Symptoms are runtime `MissingMethodException`/`NotSupportedException`/null from reflection, often only in Release publish. Warnings IL2026/IL2067/IL3050 from the trim/AOT analyzers tell you about unsafe patterns - treat them as errors in AOT projects. Use `System.Text.Json` source generation, `GeneratedRegex`, `LoggerMessage`, configuration binder generator, and `LibraryImport`.

**Code**
```csharp
using System.Diagnostics.CodeAnalysis;

static object? Make([DynamicallyAccessedMembers(DynamicallyAccessedMemberTypes.PublicParameterlessConstructor)] Type t)
    => Activator.CreateInstance(t);      // trimmer keeps the ctor of the type passed in

Console.WriteLine(Make(typeof(List<int>))?.GetType().Name); // Output: List`1
```

**Interview traps**
- "JIT happens once at startup" - it's per method, first call, plus tier-1 re-JIT.
- Tier-0 code is slow; benchmark only after warm-up (BenchmarkDotNet does).
- R2R != AOT: R2R still ships IL and needs the runtime.
- Native AOT can't use `dynamic`/Emit; not every NuGet package is AOT-safe.
- `AggressiveOptimization` disables PGO.

**Cheat sheet**
- C# -> IL -> JIT (per method, first call).
- Tier-0 -> (30 calls) -> Tier-1; OSR for loops; PGO on by default (.NET 8).
- R2R = precompiled start, still JIT hot code.
- Native AOT = no JIT/runtime, restricted reflection, platform-specific.
- Fix trim/AOT issues with source generators & annotations.

---

## Topic 18: Weak References & Lazy<T>

**Study notes**

- **`WeakReference` / `WeakReference<T>`**: refers to an object without keeping it alive. `TryGetTarget(out T)` returns false after collection. *Short* weak refs (default) are cleared when the object becomes unreachable (before finalization); *long* weak refs (`trackResurrection: true`) survive finalization. Use for caches/ canonical-instance maps/ event listeners that shouldn't pin objects. Always promote to a strong reference before use.
- **`ConditionalWeakTable<TKey,TValue>`**: associates data with an object by *reference identity*; entry lives as long as the key is alive (value is not rooted by the table), solving "attach metadata to someone else's object" leaks. API: `Add`, `TryGetValue`, `GetOrCreateValue`, `AddOrUpdate`, `Remove`.
- **`Lazy<T>`**: defer creation until `.Value`; `IsValueCreated`. Modes (`LazyThreadSafetyMode`): **ExecutionAndPublication** (default; locks, factory runs once, exceptions are **cached**), **PublicationOnly** (multiple threads may run the factory, first result wins, exceptions **not** cached), **None** (not thread-safe, fastest). `LazyInitializer.EnsureInitialized` for lazy fields without allocating a `Lazy`. `Lazy<Task<T>>` for async-once initialisation.

**Ladder**

### 18.1 [Easy] What does `Lazy<T>` do and when is it useful?
**Answer**
It delays creating an expensive object until first access of `.Value`, and caches it. Use for costly initialisation that may never be needed (large config, connections, singletons), keeping startup fast.

**Explanation**
By default it's thread-safe and the factory runs exactly once. `IsValueCreated` lets you check without triggering creation. A `Lazy<T>` field plus `=> _lazy.Value` property is the standard thread-safe lazy singleton.

**Code**
```csharp
var lazy = new Lazy<string>(() => { Console.WriteLine("init"); return "value"; });
Console.WriteLine(lazy.IsValueCreated); // Output: False
Console.WriteLine(lazy.Value);          // Output: init
                                        //         value
Console.WriteLine(lazy.Value);          // Output: value   (no second init)
```

### 18.2 [Follow-up] Compare the three `LazyThreadSafetyMode` values.
**Answer**
`ExecutionAndPublication` uses a lock so only one thread runs the factory and everyone gets that instance; `PublicationOnly` lets racing threads each run the factory but publishes only the first result (others are discarded) and doesn't cache exceptions; `None` has no synchronisation and is only safe single-threaded.

**Explanation**
Choose `PublicationOnly` when the factory is cheap/idempotent and you want to avoid lock contention or allow retry after failure. With `ExecutionAndPublication` the factory runs under a lock - if the factory itself blocks on another thread needing the same Lazy -> deadlock; and if the factory throws, that exception is cached and rethrown on every access forever.

**Code**
```csharp
int calls = 0;
var cached = new Lazy<int>(() => { calls++; throw new InvalidOperationException("fail"); });
for (int i = 0; i < 2; i++) { try { _ = cached.Value; } catch (InvalidOperationException) { } }
Console.WriteLine(calls); // Output: 1   (exception cached)

calls = 0;
var retry = new Lazy<int>(() => { calls++; throw new InvalidOperationException("fail"); },
                          LazyThreadSafetyMode.PublicationOnly);
for (int i = 0; i < 2; i++) { try { _ = retry.Value; } catch (InvalidOperationException) { } }
Console.WriteLine(calls); // Output: 2   (not cached)
```

### 18.3 [Medium] How do weak references work, and how would you build a simple weak cache?
**Answer**
A `WeakReference<T>` holds the object without rooting it; if no strong refs remain, a GC can clear it. A weak cache stores `WeakReference<T>` values: on lookup, `TryGetTarget`; if it fails, recreate and store again. Collections of weak refs themselves need pruning of dead entries.

**Explanation**
Objects are collected at an unpredictable time, so a weak cache is *opportunistic*: hit rate depends on GC activity (gen0 GCs clear young objects fast). For real caches prefer `MemoryCache` with size/expiry; weak refs suit "canonical instance" maps or avoiding event-handler leaks. In .NET Core you can use `GC.RegisterForFullGCNotification`/ `Gen2GcCallback` to prune.

**Code**
```csharp
using System.Runtime.CompilerServices;

static class Demo
{
    [MethodImpl(MethodImplOptions.NoInlining)]
    public static WeakReference<object> Make() => new(new object());
}

var weak = Demo.Make();
GC.Collect(); GC.WaitForPendingFinalizers();
Console.WriteLine(weak.TryGetTarget(out _)); // Output: False  (typically)

var strong = new object();
var weak2 = new WeakReference<object>(strong);
GC.Collect();
Console.WriteLine(weak2.TryGetTarget(out var t) && ReferenceEquals(t, strong)); // Output: True
GC.KeepAlive(strong);
```

### 18.4 [Hard] How does `ConditionalWeakTable` solve "attached property" leaks, and what are `Lazy` recursion/async pitfalls?
**Answer**
A `ConditionalWeakTable` keeps the value alive only while the key is alive (it uses dependent handles), so you can attach extra data to objects you don't own without leaking them - a plain `Dictionary<object,X>` would root keys forever. For `Lazy<T>`, a factory that reads its own `.Value` throws `InvalidOperationException`, and `Lazy<Task<T>>` caches a faulted task.

**Explanation**
CWT compares keys by reference, not `Equals`, and values may reference the key without creating a cycle problem (ephemeron semantics). For async lazy init use `Lazy<Task<T>>` (or `AsyncLazy<T>`); if the first attempt faults, the faulted task is cached - build a retry wrapper (reset the Lazy) if desired. `Lazy<T>` also isn't automatically supported by Microsoft.Extensions.DI - register `services.AddTransient(typeof(Lazy<>), typeof(LazyResolver<>))` yourself.

**Code**
```csharp
using System.Runtime.CompilerServices;

var meta = new ConditionalWeakTable<object, string>();
var key = new object();
meta.Add(key, "tag");
Console.WriteLine(meta.TryGetValue(key, out var v) ? v : "none"); // Output: tag

Lazy<int>? self = null;
self = new Lazy<int>(() => self!.Value + 1);     // reads itself
try { _ = self.Value; }
catch (InvalidOperationException) { Console.WriteLine("recursive Lazy"); } // Output: recursive Lazy
```

**Gotchas / follow-ups**
- `GCHandleType.Weak` / `WeakTrackResurrection` are the low-level equivalents.
- Follow-up: `LazyInitializer.EnsureInitialized(ref field, factory)` avoids allocating a `Lazy` wrapper.

### 18.5 [Tricky] Why can a weak reference's target disappear immediately - or never?
**Answer**
Targets can be collected as soon as no strong reference exists, including mid-method in Release builds after the last use. Conversely, string literals/interned strings, statics and objects reachable from any root are never cleared, so tests with literals always show "alive". Also a gen2 object isn't cleared by a gen0 GC.

**Explanation**
Test determinism: create the object in a `NoInlining` helper, call `GC.Collect()`; for gen2/LOH objects it needs a full collection. `GC.Collect()` in Debug can keep locals alive until method end, so results differ between Debug and Release. Long weak refs let you observe resurrected objects (finalizer resurrection) - rarely needed. Never do `if (wr.IsAlive) use(wr.Target)` (race): `TryGetTarget` once and keep the strong ref.

**Code**
```csharp
var wr = new WeakReference<string>("literal");   // literal is interned -> rooted
GC.Collect();
Console.WriteLine(wr.TryGetTarget(out _));        // Output: True  (never collected)

var wr2 = new WeakReference("made at runtime".ToUpper());
// if (wr2.IsAlive) Use(wr2.Target);   // racy: Target may be null by then
if (wr2.Target is string s) Console.WriteLine(s.Length); // single read, then strong ref
```

**Interview traps**
- `IsAlive` then `Target` is a race; get the target once.
- Weak refs to literals/statics never clear.
- Default `Lazy<T>` caches exceptions forever.
- `Lazy<T>` self-reference -> InvalidOperationException; blocking in factory can deadlock.
- A weak cache's hit rate is GC-dependent; use `MemoryCache` for real caching.

**Cheat sheet**
- WeakReference<T>: `TryGetTarget`; short vs long (resurrection).
- ConditionalWeakTable: attach data by identity, lifetime tied to key.
- Lazy modes: ExecutionAndPublication (default, caches exceptions), PublicationOnly, None.
- `Lazy<Task<T>>` for async once; faulted task is cached.
- `LazyInitializer.EnsureInitialized` for lightweight lazy fields.

---

## Topic 19: Serialization: System.Text.Json vs Newtonsoft

**Study notes**

- **System.Text.Json (STJ)**: built-in (since .NET Core 3.0), UTF-8 native, low allocation, **strict by default** (case-sensitive names, no comments/trailing commas, no private setters, fields ignored, enums as numbers, numbers not read from strings). APIs: `JsonSerializer`, `JsonSerializerOptions`, `JsonDocument` (read-only DOM), `JsonNode`/`JsonObject` (mutable DOM, .NET 6), `Utf8JsonReader/Writer`.
- **Newtonsoft.Json (Json.NET)**: older, very flexible/lenient: `JObject`/`JToken`, `dynamic`, `[JsonProperty]`, `TypeNameHandling`, `PreserveReferencesHandling`, `ReferenceLoopHandling`, `JsonConverter`, `ISerializationBinder`, DataTable/ISerializable support, private setters, case-insensitive. Slower and allocates more. Still used for legacy code/features STJ lacks (some LINQ-to-JSON conveniences, `JsonPath` queries (STJ has none built in), `DataSet`).
- **Web defaults**: `new JsonSerializerOptions(JsonSerializerDefaults.Web)` -> camelCase, case-insensitive, numbers from strings allowed. ASP.NET Core uses them.
- **Source generation**: `[JsonSerializable(typeof(T))]` on a `partial class : JsonSerializerContext` generates serialization metadata at build -> no reflection, faster startup, trimming/AOT-safe. Use `AppJsonContext.Default.T` or `options.TypeInfoResolver`.
- **Custom converters**: derive `JsonConverter<T>`, override `Read(ref Utf8JsonReader, Type, JsonSerializerOptions)` and `Write(Utf8JsonWriter, T, JsonSerializerOptions)`; attach via `[JsonConverter]` or `options.Converters`. `JsonConverterFactory` for open generics/families.
- **Cycles**: STJ throws when depth > `MaxDepth` (default 64) or a cycle is detected; options `ReferenceHandler.Preserve` (`$id/$ref`) or `ReferenceHandler.IgnoreCycles` (.NET 6). Newtonsoft: `ReferenceLoopHandling.Ignore/Serialize` and `PreserveReferencesHandling`.
- **Reuse `JsonSerializerOptions`** instances (cached metadata) - creating one per call is a classic perf bug.
- Polymorphism (.NET 7+): `[JsonPolymorphic]`, `[JsonDerivedType(typeof(Dog), "dog")]`.

**Ladder**

### 19.1 [Easy] How do you serialize/deserialize with System.Text.Json, and what are the default naming/case rules?
**Answer**
`JsonSerializer.Serialize(obj)` and `Deserialize<T>(json)`. By default property names are written as-is (PascalCase) and matching on read is case-*sensitive*; use `JsonSerializerDefaults.Web` (camelCase, case-insensitive) or set `PropertyNamingPolicy`/`PropertyNameCaseInsensitive`.

**Explanation**
Only public properties are included (fields need `IncludeFields = true` or `[JsonInclude]`). Use `[JsonPropertyName]`, `[JsonIgnore]`, `[JsonIgnore(Condition = ...)]`. Keep a static `JsonSerializerOptions` to reuse.

**Code**
```csharp
using System.Text.Json;

var person = new Person("Ann", 30);
Console.WriteLine(JsonSerializer.Serialize(person));          // Output: {"Name":"Ann","Age":30}

var web = new JsonSerializerOptions(JsonSerializerDefaults.Web);
Console.WriteLine(JsonSerializer.Serialize(person, web));     // Output: {"name":"Ann","age":30}
var back = JsonSerializer.Deserialize<Person>("{\"name\":\"Bob\",\"age\":5}", web);
Console.WriteLine(back!.Name);                                // Output: Bob

record Person(string Name, int Age);
```

### 19.2 [Follow-up] STJ vs Newtonsoft: when would you still pick Newtonsoft and what behaviours differ?
**Answer**
Pick STJ by default for performance, security and AOT support. Keep Newtonsoft for legacy code or features STJ lacks or does differently: `JObject`/`JsonPath` querying, `dynamic`, very lenient parsing, `TypeNameHandling`, `DataTable/DataSet`, extensive converters. Behaviour differences matter during migration.

**Explanation**
Differences: STJ is case-sensitive, ignores fields, throws on unknown enum strings unless `JsonStringEnumConverter`, rejects comments/trailing commas (enable via `ReadCommentHandling`/`AllowTrailingCommas`), doesn't quote-parse numbers by default (`NumberHandling = AllowReadingFromString`), serialises `object`-typed values by runtime type but *deserialises* them as `JsonElement`, no `[JsonProperty]` (use `[JsonPropertyName]`), no `JObject` (use `JsonNode`), null handling via `DefaultIgnoreCondition`. In ASP.NET Core you can still plug Newtonsoft via `AddNewtonsoftJson()`.

**Code**
```csharp
using System.Text.Json;
using System.Text.Json.Nodes;

JsonNode node = JsonNode.Parse("""{"a":{"b":5},"tags":["x","y"]}""")!;
Console.WriteLine((int)node["a"]!["b"]!);          // Output: 5
node["a"]!["c"] = "new";                           // mutable DOM
Console.WriteLine(node.ToJsonString());            // Output: {"a":{"b":5,"c":"new"},"tags":["x","y"]}

object o = JsonSerializer.Deserialize<object>("5")!;
Console.WriteLine(o.GetType().Name);               // Output: JsonElement
```

### 19.3 [Medium] How do you write a custom converter?
**Answer**
Derive from `JsonConverter<T>`, implement `Read` (parse from `Utf8JsonReader`) and `Write` (emit with `Utf8JsonWriter`), then register it with `[JsonConverter(typeof(X))]` on the type/property or via `options.Converters.Add`. Use `JsonConverterFactory` when one converter must handle many generic types.

**Explanation**
In `Read`, the reader is positioned on the first token of the value; for objects, you must read until the matching `EndObject`. Always honour `options` for nested values (`JsonSerializer.Deserialize<X>(ref reader, options)`). Override `HandleNull`/`CanConvert` for special cases. Keep converters stateless and thread-safe.

**Code**
```csharp
using System.Text.Json;
using System.Text.Json.Serialization;

public sealed class UnixTimeConverter : JsonConverter<DateTime>
{
    public override DateTime Read(ref Utf8JsonReader reader, Type typeToConvert, JsonSerializerOptions options)
        => DateTimeOffset.FromUnixTimeSeconds(reader.GetInt64()).UtcDateTime;

    public override void Write(Utf8JsonWriter writer, DateTime value, JsonSerializerOptions options)
        => writer.WriteNumberValue(new DateTimeOffset(value).ToUnixTimeSeconds());
}

public class Evt { [JsonConverter(typeof(UnixTimeConverter))] public DateTime At { get; set; } }

var e = new Evt { At = new DateTime(2020, 1, 1, 0, 0, 0, DateTimeKind.Utc) };
string json = JsonSerializer.Serialize(e);
Console.WriteLine(json);                                          // Output: {"At":1577836800}
Console.WriteLine(JsonSerializer.Deserialize<Evt>(json)!.At.Year); // Output: 2020
```

### 19.4 [Hard] How does System.Text.Json source generation work and why does it matter for AOT?
**Answer**
A source generator emits serialization metadata and (in the fast path) hand-written read/write code for the types you list in a `JsonSerializerContext`, so no runtime reflection or IL emit is needed. That gives faster startup and lower memory, and it makes serialization trimmer- and Native-AOT-safe, where reflection-based STJ is not.

**Explanation**
Declare `[JsonSerializable(typeof(T))]` for each root type (and collections like `List<T>`); nested types are discovered. Options like naming policy go in `[JsonSourceGenerationOptions]`. Pass `AppJson.Default.T` (a `JsonTypeInfo<T>`) to Serialize/Deserialize, or chain contexts via `JsonSerializerOptions.TypeInfoResolver = JsonTypeInfoResolver.Combine(...)`. In ASP.NET Core minimal APIs: `options.SerializerOptions.TypeInfoResolverChain.Insert(0, AppJson.Default)`. Not supported in source-gen mode: some features like `JsonConverter` options configured at runtime differ; polymorphism/`object` members need registration. Setting `JsonSerializerIsReflectionEnabledByDefault=false` makes missing metadata fail loudly.

**Code**
```csharp
using System.Text.Json;
using System.Text.Json.Serialization;

[JsonSourceGenerationOptions(PropertyNamingPolicy = JsonKnownNamingPolicy.CamelCase)]
[JsonSerializable(typeof(Person))]
[JsonSerializable(typeof(List<Person>))]
internal partial class AppJson : JsonSerializerContext { }

string json = JsonSerializer.Serialize(new Person("Ann", 30), AppJson.Default.Person);
Console.WriteLine(json);                                               // Output: {"name":"Ann","age":30}
var list = JsonSerializer.Deserialize("[{\"name\":\"Z\",\"age\":1}]", AppJson.Default.ListPerson);
Console.WriteLine(list![0].Name);                                      // Output: Z

record Person(string Name, int Age);
```

**Gotchas / follow-ups**
- `JsonSerializerContext` must be `partial`; types must be accessible to the generated code.
- Follow-up: metadata mode vs serialization mode (`GenerationMode`) - fast-path needs compatible options.

### 19.5 [Tricky] Circular references and other serialization gotchas.
**Answer**
By default STJ throws `JsonException` for object cycles (e.g. EF navigation properties `Parent.Children[i].Parent`). Fix by breaking the cycle (DTOs, `[JsonIgnore]`), `ReferenceHandler.IgnoreCycles` (drops the repeated reference as null) or `ReferenceHandler.Preserve` (emits `$id`/`$ref`, needs the consumer to understand it). Newtonsoft equivalents: `ReferenceLoopHandling.Ignore` and `PreserveReferencesHandling`.

**Explanation**
Preserve changes the JSON shape (clients must resolve `$ref`), so prefer DTOs/projection for APIs. Other gotchas: creating `JsonSerializerOptions` per call (slow, leaks cached metadata); `long` > 2^53 loses precision in JS - serialise as string; `DateTime` kinds/offsets; `decimal`/`double.NaN` need `NumberHandling.AllowNamedFloatingPointLiterals`; missing properties on classes leave defaults, so use `required`; polymorphic payloads need `[JsonDerivedType]` or a type discriminator; deserialising untrusted data with Newtonsoft `TypeNameHandling` != None is a known remote-code-execution vector.

**Code**
```csharp
using System.Text.Json;
using System.Text.Json.Serialization;

var a = new Node { Name = "a" };
a.Next = a;                                         // cycle

try { JsonSerializer.Serialize(a); }
catch (JsonException) { Console.WriteLine("cycle detected"); }  // Output: cycle detected

var ignore = new JsonSerializerOptions { ReferenceHandler = ReferenceHandler.IgnoreCycles };
Console.WriteLine(JsonSerializer.Serialize(a, ignore));        // Output: {"Name":"a","Next":null}

var preserve = new JsonSerializerOptions { ReferenceHandler = ReferenceHandler.Preserve };
Console.WriteLine(JsonSerializer.Serialize(a, preserve));      // Output: {"$id":"1","Name":"a","Next":{"$ref":"1"}}

class Node { public string Name { get; set; } = ""; public Node? Next { get; set; } }
```
```csharp
// Newtonsoft equivalent (dotnet add package Newtonsoft.Json)
// JsonConvert.SerializeObject(a, new JsonSerializerSettings { ReferenceLoopHandling = ReferenceLoopHandling.Ignore });
// -> {"Name":"a"}   (the looping property is omitted)
```

**Interview traps**
- STJ is case-sensitive and ignores fields by default.
- `Deserialize<object>` returns `JsonElement`, not a dictionary/primitive.
- `new JsonSerializerOptions()` per call = perf bug; cache it.
- `ReferenceHandler.Preserve` changes the wire format.
- Newtonsoft `TypeNameHandling` on untrusted input = security hole.

**Cheat sheet**
- STJ: fast, strict, UTF-8, AOT-friendly with source gen; Newtonsoft: flexible, legacy.
- Web defaults = camelCase + case-insensitive + numbers from strings.
- Custom converter = `JsonConverter<T>` Read/Write; factory for generics.
- Source gen: `[JsonSerializable]` + `JsonSerializerContext`.
- Cycles: DTOs > IgnoreCycles > Preserve.
