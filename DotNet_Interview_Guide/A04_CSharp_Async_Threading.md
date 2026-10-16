# A04 — C# Async, Tasks & Threading (Study + Interview Prep)

**How to use:** Read each topic's *Study notes* first, then work the difficulty ladder top-down (Easy → Tricky); cover the Answer and try to say it aloud before reading.
Run the **Code** blocks (C# 12 / .NET 8, console app with implicit usings) and predict `// Output:` before checking. Finish each topic with the *Interview traps* and *Cheat sheet*.

## Table of Contents
1. [Process vs Thread vs Task](#topic-1-process-vs-thread-vs-task)
2. [Thread vs ThreadPool vs Task](#topic-2-thread-vs-threadpool-vs-task)
3. [async/await basics — and what it does NOT do](#topic-3-asyncawait-basics--and-what-it-does-not-do)
4. [The compiler-generated async state machine](#topic-4-the-compiler-generated-async-state-machine)
5. [SynchronizationContext & ConfigureAwait(false)](#topic-5-synchronizationcontext--configureawaitfalse)
6. [Deadlocks with .Result / .Wait()](#topic-6-deadlocks-with-result--wait)
7. [Task vs ValueTask](#topic-7-task-vs-valuetask)
8. [async void & exception handling in async](#topic-8-async-void--exception-handling-in-async)
9. [Task.WhenAll / WhenAny / Run / FromResult / Delay](#topic-9-taskwhenall--whenany--run--fromresult--delay)
10. [CancellationToken, linked tokens, timeouts](#topic-10-cancellationtoken-linked-tokens-timeouts)
11. [IAsyncEnumerable & await foreach](#topic-11-iasyncenumerable--await-foreach)
12. [Parallel.For/ForEach/ForEachAsync & PLINQ](#topic-12-parallelforforeachforeachasync--plinq)
13. [Locks: lock/Monitor, SemaphoreSlim, Mutex, ReaderWriterLockSlim, SpinLock](#topic-13-locks-lockmonitor-semaphoreslim-mutex-readerwriterlockslim-spinlock)
14. [Interlocked, volatile & memory model basics](#topic-14-interlocked-volatile--memory-model-basics)
15. [Race conditions & thread-safety demos](#topic-15-race-conditions--thread-safety-demos)
16. [Concurrent collections & BlockingCollection](#topic-16-concurrent-collections--blockingcollection)
17. [System.Threading.Channels producer/consumer](#topic-17-systemthreadingchannels-producerconsumer)
18. [Task.Run: CPU-bound vs I/O-bound](#topic-18-taskrun-cpu-bound-vs-io-bound)
19. [Thread starvation & pool tuning](#topic-19-thread-starvation--pool-tuning)
20. [Async locks & why `lock` can't await](#topic-20-async-locks--why-lock-cant-await)
21. [TaskCompletionSource](#topic-21-taskcompletionsource)
22. [AsyncLocal / ThreadLocal](#topic-22-asynclocal--threadlocal)
23. [Timers](#topic-23-timers)
24. [Fire-and-forget pitfalls](#topic-24-fire-and-forget-pitfalls)
25. [Async output-prediction puzzles](#topic-25-async-output-prediction-puzzles)

---

## Topic 1: Process vs Thread vs Task

**Study notes**

- **Process**: an OS-isolated container with its own virtual address space, handle table, security context. Crash in one process does not corrupt another. Heavy to create; communication needs IPC (pipes, sockets, memory-mapped files).
- **Thread**: the OS unit of *scheduling/execution* inside a process. Threads share the heap and static data but each has its own stack (default 1 MB on Windows for .NET), registers, and thread-local storage. Context switching costs microseconds.
- **Task** (`System.Threading.Tasks.Task`): a .NET *abstraction of a unit of asynchronous work / a promise of a future result*. A Task is **not** a thread. It may run on a thread-pool thread (delegate task), or may represent I/O completion that uses **no thread at all** while waiting.

```
Process (own address space)
 ├─ Heap / statics  (shared by all threads)
 ├─ Thread 1  [own stack]
 ├─ Thread 2  [own stack]
 └─ ThreadPool threads ──► run work items / Task delegates
Task = object with state (Created, WaitingForActivation, Running, RanToCompletion, Faulted, Canceled)
```

| Concept | Isolation | Cost | Shares memory? |
|---|---|---|---|
| Process | Strong | Highest | No (needs IPC) |
| Thread | None (within process) | ~1 MB stack + kernel object | Yes |
| Task | None | Small object (~100 bytes) | Yes |

### 1.1 [Easy] What is the difference between a process and a thread?
**Answer**
A process is an isolated program instance with its own virtual memory; a thread is an execution path inside a process. Threads in the same process share the heap and statics but have separate stacks. Processes are isolated, so need IPC to communicate.

**Explanation**
The OS schedules *threads*, not processes. A process always has at least one thread (the main thread). Sharing memory makes thread communication cheap but introduces races; isolation makes processes safer but communication costlier. In .NET, `AppDomain`s no longer provide isolation in .NET Core+, so use processes (or `AssemblyLoadContext` for unloading, not isolation).

**Code**
```csharp
using System.Diagnostics;

var p = Process.GetCurrentProcess();
Console.WriteLine($"PID={p.Id}, Threads={p.Threads.Count}");
Console.WriteLine($"Main thread id={Environment.CurrentManagedThreadId}");
// Output (varies): PID=12345, Threads=~10   Main thread id=1
```

### 1.2 [Follow-up] Is a Task a thread?
**Answer**
No. A Task represents an operation that will complete in the future. A delegate-based Task (`Task.Run`) is *queued* to a thread-pool thread, but a Task from an I/O call (`HttpClient.GetAsync`) occupies no thread while waiting.

**Explanation**
Task = state machine of completion + result/exception + continuations. The scheduler (default `TaskScheduler.Default` → ThreadPool) decides which thread runs a delegate task. Promise-style tasks (created by `TaskCompletionSource` or async methods) are never "run" by a thread themselves; they are completed by whoever calls `SetResult`/the I/O completion port callback.

**Code**
```csharp
var t1 = Task.Run(() => Console.WriteLine($"delegate task on pool thread: {Thread.CurrentThread.IsThreadPoolThread}"));
await t1;
// Output: delegate task on pool thread: True

var tcs = new TaskCompletionSource<int>();
Console.WriteLine(tcs.Task.Status); // Output: WaitingForActivation  (no thread involved)
tcs.SetResult(1);
Console.WriteLine(tcs.Task.Status); // Output: RanToCompletion
```

### 1.3 [Medium] Why is creating many threads expensive, and how do Tasks help?
**Answer**
Each thread reserves a stack (1 MB virtual), a kernel object, and incurs context-switch costs; thousands of threads thrash the scheduler. Tasks are scheduled onto a small, reused pool of threads, and async I/O tasks use no thread while waiting, so far more concurrent operations are possible.

**Explanation**
Context-switching saves/restores registers and can flush CPU caches. With 10,000 concurrent HTTP calls, 10,000 blocked threads is impractical, whereas 10,000 pending tasks cost a few MB total. The pool also uses hill-climbing to adjust thread count to throughput.

**Code**
```csharp
using System.Diagnostics;
var sw = Stopwatch.StartNew();
var tasks = Enumerable.Range(0, 10_000).Select(_ => Task.Delay(1000));
await Task.WhenAll(tasks);
Console.WriteLine($"10k concurrent delays took ~{sw.ElapsedMilliseconds} ms with a handful of threads");
// Output: ~1000-1100 ms. 10k dedicated threads would be very costly/fail.
```

### 1.4 [Hard] Concurrency vs parallelism vs asynchrony — distinguish them.
**Answer**
Concurrency is *dealing with* many things at once (interleaving), parallelism is *doing* many things simultaneously on multiple cores, asynchrony is not blocking the caller while waiting for an operation. Async I/O gives concurrency without parallelism or extra threads; `Parallel.For` gives parallelism for CPU work.

**Explanation**
A single-threaded UI app using `await` is concurrent but not parallel. A CPU-bound matrix multiply benefits from parallelism but not from async. Mixing them up leads to wrong tooling: `Task.Run` around I/O wastes a thread; `async` around CPU loops gives no speedup.

**Code**
```csharp
// Concurrency w/o parallelism (single thread via simulated context would show same thread);
// Parallelism for CPU-bound:
var data = Enumerable.Range(1, 8).ToArray();
Parallel.ForEach(data, i => Console.WriteLine($"{i} on thread {Environment.CurrentManagedThreadId}"));
// Output: 8 lines, order and thread ids vary

// Asynchrony for I/O:
await Task.Delay(100); // no thread blocked during delay
```

**Gotchas / follow-ups**
- "Is async faster?" — Not per-operation; it improves *scalability* (throughput per thread), not latency.
- Follow-up: "Can async code be single-threaded?" Yes (UI thread, `SynchronizationContext`).

### 1.5 [Tricky] How many threads does `await Task.Delay(1000)` use while waiting?
**Answer**
Zero. `Task.Delay` is backed by a timer, not a sleeping thread. When the timer fires, the continuation is queued to the pool (or captured context).

**Explanation**
Contrast with `Thread.Sleep(1000)`, which blocks the OS thread for the duration. In ASP.NET Core, using `Thread.Sleep` in request handlers burns pool threads and can lead to starvation (Topic 19). Timers in .NET are managed by a shared timer queue (a single dedicated timer thread dispatches to the pool).

**Code**
```csharp
Console.WriteLine($"Before: {Environment.CurrentManagedThreadId}");
await Task.Delay(100);
Console.WriteLine($"After : {Environment.CurrentManagedThreadId}");
// Output (console app, no SynchronizationContext): Before: 1 / After: some pool thread id (e.g. 4)
```

**Gotchas / follow-ups**
- Console apps have no SynchronizationContext, so continuation resumes on a pool thread; UI apps resume on the UI thread.

**Interview traps**
- "Task = thread" is wrong; say "Task = promise of work/result."
- Async ≠ parallel ≠ multi-threaded.
- Threads share heap → races; processes don't.
- `Thread.Sleep` blocks a thread; `Task.Delay` doesn't.

**Cheat sheet**
- Process = isolation; thread = execution; Task = abstraction.
- Threads: own stack, shared heap.
- I/O-bound async uses no thread while waiting.
- Parallelism = CPU-bound; async = I/O-bound.

---

## Topic 2: Thread vs ThreadPool vs Task

**Study notes**

- `new Thread(...)`: dedicated thread, foreground by default (keeps process alive), configurable stack size/priority/name. Expensive; use for long-running dedicated work or special apartment state (STA).
- **ThreadPool**: shared pool of worker threads (+ I/O completion threads on Windows). Work items go to a global queue plus per-thread local queues (work stealing). Threads are background threads. Min threads default = processor count; pool injects extra threads slowly (hill-climbing, ~1–2 per second beyond min) when starved.
- **Task / Task.Run**: scheduled via `TaskScheduler.Default` → ThreadPool. Adds continuations, result, exception propagation, cancellation, composition (`WhenAll`, `ContinueWith`).
- `Task.Factory.StartNew(..., TaskCreationOptions.LongRunning)` hints the scheduler to use a dedicated thread.

```
           ┌─────────── Global queue ───────────┐
Task.Run ─►│ work item                          │
           └───────┬────────────────────────────┘
   ┌───────────────▼─────────┐   ┌─────────────────┐
   │ Worker 1 [local queue]  │◄─►│ Worker 2 (steal)│
   └─────────────────────────┘   └─────────────────┘
```

| | Thread | ThreadPool.QueueUserWorkItem | Task |
|---|---|---|---|
| Return value | No (manual) | No | Yes (`Task<T>`) |
| Exceptions | Crash process if unhandled | Crash process | Captured in Task |
| Cancellation | Manual | Manual | CancellationToken |
| Composition | Join only | None | Rich |
| Cost | High | Low | Low |

### 2.1 [Easy] When would you create a raw `Thread` instead of using the pool?
**Answer**
For long-running or blocking loops that would otherwise hog a pool thread, when you need a foreground thread, custom priority/stack size/name, or STA apartment state (e.g., COM/legacy UI interop). Otherwise prefer Tasks.

**Explanation**
Pool threads are a shared resource; parking one forever starves other work. Dedicated threads also let you set `IsBackground`, `Name` (helps debugging), and `SetApartmentState` before start.

**Code**
```csharp
var t = new Thread(() =>
{
    Console.WriteLine($"Name={Thread.CurrentThread.Name}, Pool={Thread.CurrentThread.IsThreadPoolThread}, BG={Thread.CurrentThread.IsBackground}");
}) { Name = "worker-1" };
t.Start();
t.Join();
// Output: Name=worker-1, Pool=False, BG=False
```

### 2.2 [Follow-up] What is the difference between foreground and background threads?
**Answer**
The process stays alive until all foreground threads finish; background threads are killed when the last foreground thread exits. `new Thread` is foreground by default; pool threads are background.

**Explanation**
A forgotten foreground thread prevents app exit. Background threads get no chance to finish cleanup (`finally` may not run), so don't rely on them for critical flushes.

**Code**
```csharp
var bg = new Thread(() => { Thread.Sleep(5000); Console.WriteLine("never printed"); }) { IsBackground = true };
bg.Start();
Console.WriteLine("Main exiting");
// Output: Main exiting   (process exits immediately; background thread is abandoned)
```

### 2.3 [Medium] How are exceptions handled differently in Thread vs Task?
**Answer**
An unhandled exception on a `Thread` or `QueueUserWorkItem` crashes the process. In a Task the exception is captured in the Task and rethrown when you `await`/`Wait`/`.Result` it (as `AggregateException` for Wait/Result).

**Explanation**
Unobserved Task exceptions *used to* crash the process on finalization (.NET 4.0); since 4.5 and in .NET Core+, they are swallowed (raise `TaskScheduler.UnobservedTaskException` only) unless configured otherwise. `await` unwraps and throws the first inner exception.

**Code**
```csharp
var task = Task.Run(() => throw new InvalidOperationException("boom"));
try { await task; }
catch (InvalidOperationException ex) { Console.WriteLine($"Caught: {ex.Message}"); }
// Output: Caught: boom

try { task.Wait(); }
catch (AggregateException ae) { Console.WriteLine(ae.InnerException!.Message); }
// Output: boom
```

### 2.4 [Hard] Explain ThreadPool work-stealing and the global vs local queues.
**Answer**
Tasks queued from *outside* the pool go to the global (FIFO) queue; tasks queued from *within* a pool thread (e.g., child work or `Task.Run` inside a task) go to that thread's local LIFO deque. An idle worker first checks its own local queue, then global, then steals from other workers' local queues (from the opposite end).

**Explanation**
Local LIFO improves cache locality and reduces contention (no lock on owner push/pop). Stealing from the tail keeps balance. `TaskCreationOptions.PreferFairness` (or `ThreadPool.QueueUserWorkItem(..., preferLocal: false)`) pushes to global queue for FIFO-ish fairness. Consequence: ordering of Tasks is not guaranteed.

**Code**
```csharp
await Task.Run(() =>
{
    // These two are queued to *this* worker's local queue
    var a = Task.Run(() => Console.WriteLine("A"));
    var b = Task.Run(() => Console.WriteLine("B"));
    Task.WaitAll(a, b);
});
// Output: order of A/B is not guaranteed.
```

**Gotchas / follow-ups**
- Don't assume FIFO execution of tasks.
- `Task.WaitAll` inside a pool thread blocks it (but the pool may inline tasks onto the waiting thread — "task inlining").

### 2.5 [Tricky] What happens if you set `ThreadPool.SetMinThreads` too low / too high, and what is the default behavior under blocking?
**Answer**
Below min threads, the pool creates threads immediately on demand; above min, it throttles injection to roughly one thread per 0.5–1 s using hill-climbing. Raising min threads can help short bursts of blocking work but masks the real problem (blocking on async); too high wastes memory and context switches.

**Explanation**
Default min = `Environment.ProcessorCount` for workers. A burst of 100 blocking requests on an 8-core box means ~92 requests wait for slow injection → latency spikes ("thread pool starvation"). The right fix is to remove sync-over-async, not to raise limits (Topic 19).

**Code**
```csharp
ThreadPool.GetMinThreads(out int w, out int io);
Console.WriteLine($"Min worker={w}, io={io}"); // Output: Min worker=<cores>, io=<cores>
ThreadPool.GetMaxThreads(out int mw, out int mio);
Console.WriteLine($"Max worker={mw}, io={mio}"); // Output: large numbers (e.g. 32767, 1000)
```

**Interview traps**
- `Task.Run` does not create a new thread — it queues to the pool.
- Unhandled exceptions on a raw Thread kill the app.
- Pool threads are background: process can exit mid-work.
- `LongRunning` is a *hint* for dedicated thread, only via `StartNew`/`TaskCreationOptions`.

**Cheat sheet**
- Thread = dedicated, costly, foreground by default.
- Pool thread = shared, background, reused.
- Task = pool + result + exceptions + continuations.
- Min threads default = core count; slow injection beyond.
- Local queues LIFO, steal from others, global FIFO.

---

## Topic 3: async/await basics — and what it does NOT do

**Study notes**

- `async` marks a method as able to use `await`; it does *not* make it asynchronous or run on another thread. The compiler rewrites it into a state machine.
- `await expr`: if `expr` (an *awaitable*) is already complete, continue **synchronously**; else register a continuation, **return to the caller** (with an incomplete Task), and resume when it completes.
- Return types: `Task`, `Task<T>`, `ValueTask`, `ValueTask<T>`, `void` (event handlers only), or custom task-like (`[AsyncMethodBuilder]`). C# 7+ allows any task-like type.
- Everything before the first *incomplete* await runs synchronously on the caller's thread.
- Naming convention: `Async` suffix.
- Exceptions thrown in async method are stored in the returned Task.

```
Caller ──► AsyncMethod() runs synchronously ─► await (incomplete) ─► returns Task to caller
                                                     │ (I/O in flight, no thread)
                                          completion ▼
                                  continuation runs (captured context or pool)
```

### 3.1 [Easy] What does the `async` keyword do?
**Answer**
It enables `await` inside the method and tells the compiler to generate a state machine and wrap the return value/exception in a Task. It does not start a thread or make the method run in the background.

**Explanation**
An `async` method with no `await` runs entirely synchronously (compiler warning CS1998). Only awaiting a genuinely incomplete operation yields control.

**Code**
```csharp
static async Task<int> NoAwaitAsync()   // warning CS1998
{
    Console.WriteLine($"inside: {Environment.CurrentManagedThreadId}");
    return 42;
}
Console.WriteLine($"main:   {Environment.CurrentManagedThreadId}");
var r = await NoAwaitAsync();
// Output: main: 1 / inside: 1  (same thread — fully synchronous)
```

### 3.2 [Follow-up] Does `await` create a new thread?
**Answer**
No. `await` never creates a thread. It checks if the awaited task is complete; if not, it hooks a continuation and returns. The continuation runs later on whichever thread completes the operation/the captured context.

**Explanation**
For real I/O (sockets, files opened with `FileOptions.Asynchronous`), the OS signals completion via IOCP; a pool I/O thread briefly runs the completion and queues your continuation. While waiting, no thread is dedicated to you.

**Code**
```csharp
using var http = new HttpClient();
Console.WriteLine($"before: {Environment.CurrentManagedThreadId}");
var s = await http.GetStringAsync("https://example.com");
Console.WriteLine($"after : {Environment.CurrentManagedThreadId}"); // may differ in console app
```

### 3.3 [Medium] Where does the code before the first `await` run, and what is returned to the caller?
**Answer**
It runs synchronously on the caller's thread. At the first incomplete await, the method returns an incomplete `Task` to the caller, which can continue and later await it.

**Explanation**
This is why a "slow" async method with CPU work before its first await still blocks the caller. It also explains why argument validation in async methods throws lazily (inside the Task) rather than at call time.

**Code**
```csharp
static async Task DoAsync()
{
    Console.WriteLine("1: sync part");
    await Task.Delay(100);
    Console.WriteLine("3: continuation");
}
var t = DoAsync();
Console.WriteLine("2: caller continues");
await t;
// Output:
// 1: sync part
// 2: caller continues
// 3: continuation
```

### 3.4 [Medium] Explain "async all the way" and why.
**Answer**
Once you call an async API, you should await it and make your own method async, propagating up to the entry point (controller, `Main`, event handler). Blocking with `.Result`/`.Wait()` mid-chain risks deadlocks and thread starvation.

**Explanation**
`Main` can be `static async Task Main`. ASP.NET Core controllers/minimal APIs support async natively. Mixing sync and async ("sync-over-async" or "async-over-sync") defeats the scalability purpose.

**Code**
```csharp
public static class Program
{
    public static async Task Main()
    {
        var x = await GetAsync();
        Console.WriteLine(x); // Output: 7
    }
    static async Task<int> GetAsync() { await Task.Delay(10); return 7; }
}
```

### 3.5 [Hard] What are the rules for what can be awaited?
**Answer**
Anything with a `GetAwaiter()` method (instance or extension) that returns a type with `IsCompleted`, `OnCompleted` (via `INotifyCompletion`) and `GetResult()`. `Task`, `ValueTask`, `YieldAwaitable` (`Task.Yield()`), `ConfiguredTaskAwaitable` all qualify; you can write your own.

**Explanation**
That's duck typing. This is how `await TimeSpan` extension tricks or `await process.WaitForExitAsync` wrappers work. `Task.Yield()` forces a continuation to be scheduled instead of completing synchronously.

**Code**
```csharp
using System.Runtime.CompilerServices;

public static class TimeSpanAwaiterExt
{
    public static TaskAwaiter GetAwaiter(this TimeSpan ts) => Task.Delay(ts).GetAwaiter();
}
// usage: await TimeSpan.FromMilliseconds(50);
```

**Gotchas / follow-ups**
- Extension `GetAwaiter` is legal but unusual; use sparingly.
- `await Task.Yield()` in a console app queues to the pool; in UI posts to the UI loop.

### 3.6 [Tricky] What does this print? (sync part vs async part)
**Answer**
`B-before`, `A`, `D`, `C`. Everything before the await runs synchronously; the caller prints `D` before the continuation prints `C`.

**Explanation**
`Task.Delay(0)` returns an already-completed task, so with `Delay(0)` the await would complete synchronously! `Delay(10)` is effectively always incomplete at the await.

**Code**
```csharp
static async Task FooAsync()
{
    Console.WriteLine("A");
    await Task.Delay(10);
    Console.WriteLine("C");
}
Console.WriteLine("B-before");
var t = FooAsync();         // prints A
Console.WriteLine("D");     // caller continues
await t;                    // prints C
// Output: B-before, A, D, C

// With Task.Delay(0) the await is synchronous: output would be A, C, D
```

**Gotchas / follow-ups**
- `await Task.CompletedTask` / already-completed tasks don't yield.

**Interview traps**
- "async makes it run on a background thread" — false.
- `async` method w/o await runs synchronously.
- Argument validation inside async methods is deferred to the returned Task.
- Already-completed awaits continue synchronously.

**Cheat sheet**
- async = compiler transform; await = "continue when done, don't block".
- Code before first incomplete await is sync.
- No thread held during I/O waits.
- Async all the way; avoid `.Result`.
- Awaitable = has `GetAwaiter()`.

---

## Topic 4: The compiler-generated async state machine

**Study notes**

For each `async` method the C# compiler generates a struct (Release) or class (Debug) implementing `IAsyncStateMachine` with:
- `<>1__state` — int: `-1` = running/not started, `0..n` = suspended at the nth await, `-2` = finished.
- Hoisted locals and parameters (become fields).
- `<>t__builder` — `AsyncTaskMethodBuilder<T>` which owns/creates the returned Task.
- `MoveNext()` — the original body as a `switch(state)` with `try/catch`.
- Awaiters stored as fields (e.g. `TaskAwaiter<int>`).

Flow at each `await`:
```
MoveNext():
  switch (state) { case 0: goto resume0; ... }
  awaiter = expr.GetAwaiter();
  if (!awaiter.IsCompleted) {
      state = 0; this.awaiter = awaiter;
      builder.AwaitUnsafeOnCompleted(ref awaiter, ref this);   // registers continuation (MoveNext)
      return;                                                  // unwinds to caller
  }
resume0:
  result = awaiter.GetResult();   // throws if faulted
  ...
  builder.SetResult(value) / SetException(ex)   // completes the Task
```
- The struct state machine is **boxed** to the heap the first time an await is incomplete (so the fast synchronous path allocates less).
- Builder captures `ExecutionContext` (flows AsyncLocal/culture/security) and, unless `ConfigureAwait(false)`, the `SynchronizationContext`/`TaskScheduler`.
- `try/finally` and `using` are preserved across awaits via state.

### 4.1 [Easy] What does the compiler turn an async method into?
**Answer**
A state machine class/struct implementing `IAsyncStateMachine`, with a `MoveNext` method containing the original code split at each `await`, and fields for hoisted locals. The original method becomes a stub that creates the state machine and starts it.

**Explanation**
You can see this by decompiling (ILSpy, sharplab.io) — look for `<MethodName>d__0`. It is not threads or callbacks you wrote; it's a compiler-generated resumable function.

**Code**
```csharp
// Source
static async Task<int> AddAsync(int a, int b)
{
    await Task.Delay(10);
    return a + b;
}
// Roughly generated:
// struct <AddAsync>d__0 : IAsyncStateMachine { int <>1__state; AsyncTaskMethodBuilder<int> <>t__builder; int a, b; TaskAwaiter <>u__1; void MoveNext() {...} }
```

### 4.2 [Follow-up] Why are local variables preserved across awaits?
**Answer**
The compiler "hoists" locals that live across an await into fields of the state machine, so they survive the method returning and resuming later.

**Explanation**
Locals not used across awaits may stay as true locals of `MoveNext`. This also means large locals live as long as the state machine does → can extend object lifetimes (memory retention). Ref locals and `Span<T>` cannot live across awaits (CS4012/CS4007) because they can't be hoisted to the heap.

**Code**
```csharp
static async Task Demo()
{
    int counter = 0;               // hoisted
    await Task.Delay(10);
    counter++;
    Console.WriteLine(counter);    // Output: 1
}
// Span<int> s = stackalloc int[4]; await ...; // compile error: cannot use Span across await
```

### 4.3 [Medium] Where do allocations happen in an async method?
**Answer**
On the synchronous-completion path a `Task<T>`-returning method may allocate only the Task (or nothing for cached results like `Task<bool>` true/false, small ints, `Task.CompletedTask`). When it truly suspends, the struct state machine is boxed and a Task object is allocated, plus delegate for continuation.

**Explanation**
.NET 6+ also pools state machine boxes optionally via `[AsyncMethodBuilder(typeof(PoolingAsyncValueTaskMethodBuilder))]` for `ValueTask`. `DOTNET_SYSTEM_THREADING_POOLASYNCVALUETASKS=1` enables it globally for ValueTask-returning methods. This is why hot paths prefer `ValueTask` (Topic 7).

**Code**
```csharp
Task<bool> a = Task.FromResult(true);
Task<bool> b = Task.FromResult(true);
Console.WriteLine(ReferenceEquals(a, b));
// Output: True  (cached completed Task<bool>, no allocation) in .NET Core 3+/5+
```

### 4.4 [Hard] How does the exception flow through the state machine?
**Answer**
`MoveNext` wraps the body in try/catch. An exception calls `builder.SetException(ex)`, which faults the Task; the exception is rethrown at the `await` site via `awaiter.GetResult()` in the caller's own state machine. The `await` rethrows the *original* exception (not wrapped in `AggregateException`), preserving stack via `ExceptionDispatchInfo`.

**Explanation**
Only the first exception is thrown by `await` for a Task faulted with several (e.g., `WhenAll`); all are in `task.Exception.InnerExceptions`. The stack trace shows "--- End of stack trace from previous location ---" lines at await boundaries.

**Code**
```csharp
static async Task Thrower() { await Task.Yield(); throw new InvalidOperationException("x"); }
var t = Thrower();                    // no throw here
Console.WriteLine(t.IsFaulted || !t.IsCompleted); // Output: True
try { await t; } catch (InvalidOperationException e) { Console.WriteLine(e.Message); } // Output: x
```

**Gotchas / follow-ups**
- Exceptions thrown *before* the first await still end up inside the returned Task, not thrown synchronously (unlike iterator/sync methods).

### 4.5 [Tricky] When does an awaited continuation run "inline" (synchronously) vs scheduled? What is `RunContinuationsAsynchronously`?
**Answer**
By default a continuation may run synchronously on the thread that completes the antecedent (inline) when no context requires otherwise. This can cause the completer (`SetResult` caller) to execute arbitrary user code, causing re-entrancy and deadlocks. `TaskCreationOptions.RunContinuationsAsynchronously` on a `TaskCompletionSource` forces continuations onto the pool.

**Explanation**
Inline continuations are an optimization, but when you hold a lock while calling `tcs.SetResult` the awaiting code may run under your lock. Always create TCS with `RunContinuationsAsynchronously` unless you've proven otherwise (Topic 21).

**Code**
```csharp
var tcs = new TaskCompletionSource<int>(TaskCreationOptions.RunContinuationsAsynchronously);
var waiter = Task.Run(async () =>
{
    await tcs.Task;
    Console.WriteLine($"continuation on {Environment.CurrentManagedThreadId}");
});
await Task.Delay(50);
tcs.SetResult(1); // returns immediately; continuation runs on pool
await waiter;
```

**Interview traps**
- State machine is a struct in Release, class in Debug.
- Locals across awaits are hoisted → retain memory.
- `Span<T>`/ref locals can't cross await.
- Exceptions in async methods never throw synchronously (stored in Task).

**Cheat sheet**
- `IAsyncStateMachine.MoveNext` + state int.
- Builder creates Task, awaiters register continuations.
- Boxing happens on first real suspension.
- `await` unwraps exception (first inner) via `GetResult()`.
- Use `RunContinuationsAsynchronously` with TCS.

---

## Topic 5: SynchronizationContext & ConfigureAwait(false)

**Study notes**

- `SynchronizationContext` (SC) abstracts "where should continuations run". `SynchronizationContext.Current` is set per thread.
  - WinForms/WPF/MAUI: SC posts to the UI thread (single thread).
  - Classic ASP.NET (System.Web): SC ties to the request context (one-at-a-time per request).
  - **ASP.NET Core: no SC** (null). Console apps: null. Continuations go to `TaskScheduler.Current` (usually the pool).
- On `await`, the awaiter captures `SynchronizationContext.Current` (if not null) else `TaskScheduler.Current` (if not Default). On completion, the continuation is `Post`ed there.
- `ConfigureAwait(false)` → don't capture; continue on whatever thread completes (usually pool).
- `ConfigureAwait(true)` is the default. `ConfigureAwaitOptions` (.NET 8): `None`, `ContinueOnCapturedContext`, `SuppressThrowing`, `ForceYielding` for `Task` (non-generic with options overload).
- `ExecutionContext` (AsyncLocal, culture, etc.) flows regardless of ConfigureAwait.

```
UI thread: await X ──► captures UI SC
   ... X completes on pool thread ...
   SC.Post(continuation) ──► runs on UI thread   (default)
ConfigureAwait(false): continuation runs inline/pool thread, UI not touched
```

### 5.1 [Easy] What does `ConfigureAwait(false)` do?
**Answer**
It tells the awaiter not to marshal the continuation back to the captured `SynchronizationContext`/`TaskScheduler`; the code after the await runs on whichever thread completed the awaited task (typically the pool).

**Explanation**
Benefits: avoids deadlocks when callers block, and avoids needless context hops. Cost: after it, you can't touch UI elements or `HttpContext` (classic ASP.NET). Library code should use it; app-level code (UI handlers) shouldn't.

**Code**
```csharp
static async Task<string> LoadAsync(HttpClient http)
{
    var s = await http.GetStringAsync("https://example.com").ConfigureAwait(false);
    return s.Length.ToString(); // may run on a pool thread, not on the original context
}
```

### 5.2 [Follow-up] Do I need ConfigureAwait(false) in ASP.NET Core?
**Answer**
Not for correctness: ASP.NET Core has no SynchronizationContext, so there is nothing to capture. It is still recommended in reusable libraries that might be consumed from UI/legacy ASP.NET apps.

**Explanation**
In ASP.NET Core app code, adding it everywhere is noise (and analyzers like CA2007 are typically disabled for apps). `TaskScheduler.Current` is Default, so no hop either.

**Code**
```csharp
Console.WriteLine(SynchronizationContext.Current is null); // Output (console / ASP.NET Core): True
```

### 5.3 [Medium] Does ConfigureAwait(false) guarantee a thread switch? Does it affect only code after the await?
**Answer**
No guarantee of switching: if the awaited task is already complete, the code continues synchronously on the same thread. It only affects the continuation of *that specific await*; each await needs its own `ConfigureAwait`, and it doesn't change the context of the caller.

**Explanation**
A common bug: `ConfigureAwait(false)` on the first await but later awaits (or an inner helper without it) re-capture. Another: after the first ConfigureAwait(false) the SC is gone on that thread (it's null on a pool thread), so subsequent awaits there capture nothing anyway—but if the first await completed synchronously you may still be on the UI thread and later awaits capture the UI context.

**Code**
```csharp
static async Task Demo()
{
    await Task.CompletedTask.ConfigureAwait(false); // completes synchronously: no switch
    // still on caller thread / context here
    await Task.Delay(10).ConfigureAwait(false);
    // now on a pool thread (no context)
}
```

### 5.4 [Hard] How does a custom SynchronizationContext work? Show a single-threaded one.
**Answer**
Derive from `SynchronizationContext`, override `Post` (async queue) and `Send` (sync). The awaiter calls `Post(callback, state)` for continuations. A "message pump" thread then executes queued callbacks one at a time, which is how UI and test frameworks (xUnit's `AsyncTestSyncContext`) serialize work.

**Explanation**
This explains the classic deadlock: a single-threaded SC + blocking wait on that thread means the posted continuation can never run (Topic 6).

**Code**
```csharp
using System.Collections.Concurrent;

sealed class SingleThreadSyncContext : SynchronizationContext
{
    private readonly BlockingCollection<(SendOrPostCallback cb, object? st)> _queue = new();
    public override void Post(SendOrPostCallback d, object? state) => _queue.Add((d, state));

    public void RunOnCurrentThread()
    {
        SetSynchronizationContext(this);
        foreach (var (cb, st) in _queue.GetConsumingEnumerable()) cb(st);
    }
    public void Complete() => _queue.CompleteAdding();
}

static class Demo
{
    public static void Run()
    {
        var ctx = new SingleThreadSyncContext();
        ctx.Post(_ =>
        {
            _ = Work(ctx);
        }, null);
        ctx.RunOnCurrentThread();
    }
    static async Task Work(SingleThreadSyncContext ctx)
    {
        var tid = Environment.CurrentManagedThreadId;
        await Task.Delay(50);
        Console.WriteLine($"same thread after await: {tid == Environment.CurrentManagedThreadId}");
        ctx.Complete();
    }
}
// Demo.Run(); // Output: same thread after await: True
```

**Gotchas / follow-ups**
- Without `Complete()` the pump never exits.
- `Send` default implementation calls the delegate on the current thread.

### 5.5 [Tricky] What does ConfigureAwait do in these cases: `await using`, `await foreach`, `Task.Run`?
**Answer**
`await using` and `await foreach` need explicit `.ConfigureAwait(false)` on the disposable/enumerable (`await using var x = y.ConfigureAwait(false);` — via `ConfiguredAsyncDisposable`; `await foreach (var i in src.ConfigureAwait(false))`). `Task.Run(async () => ...)` runs the delegate on the pool where no SC exists, so inner awaits don't capture the UI context even without ConfigureAwait.

**Explanation**
Forgetting it on async disposal is a frequent library bug. `ConfigureAwait(false)` on an `IAsyncEnumerable` returns `ConfiguredCancelableAsyncEnumerable<T>`; `WithCancellation(ct)` can be chained.

**Code**
```csharp
static async IAsyncEnumerable<int> Gen()
{
    for (int i = 0; i < 3; i++) { await Task.Delay(1); yield return i; }
}
await foreach (var i in Gen().ConfigureAwait(false))
    Console.WriteLine(i);

await using var d = new MyRes().ConfigureAwait(false);   // ConfiguredAsyncDisposable usage
// Output: 0 1 2

sealed class MyRes : IAsyncDisposable
{
    public ValueTask DisposeAsync() => ValueTask.CompletedTask;
}
```

**Gotchas / follow-ups**
- `ConfigureAwait` is an extension on awaitables, not a global setting.
- In .NET 8: `await task.ConfigureAwait(ConfigureAwaitOptions.SuppressThrowing)` swallows exceptions — use carefully.

**Interview traps**
- "ConfigureAwait(false) makes it run on a background thread" — no; it only skips context capture.
- Not needed in ASP.NET Core app code, but is in libraries.
- It does not affect AsyncLocal/ExecutionContext flow.
- Needed on each await; mixing leaves residual context captures.

**Cheat sheet**
- Default: capture SC (or non-default TaskScheduler).
- `false`: skip capture → no deadlock from blocked context.
- UI code after await touching controls → keep default.
- ASP.NET Core / console: SC is null.
- `await foreach`/`await using` need `.ConfigureAwait(false)` too.

---

## Topic 6: Deadlocks with .Result / .Wait()

**Study notes**

Classic sync-over-async deadlock requires:
1. A `SynchronizationContext` that allows only one thread at a time (UI or legacy ASP.NET).
2. Caller **blocks** that thread on `Task.Result` / `.Wait()` / `.GetAwaiter().GetResult()`.
3. The awaited async method captured that same context and needs it to resume.

```
UI thread: calls GetAsync().Result  ──► blocked waiting for Task
GetAsync: await Delay ──► done ──► Post(continuation to UI SC)
UI SC: thread busy (blocked) ─── never runs continuation ─── Task never completes ─► DEADLOCK
```
Fixes (best → worst): (1) async all the way; (2) `ConfigureAwait(false)` in the callee (library); (3) offload with `Task.Run(() => GetAsync()).GetAwaiter().GetResult()` (hack). Also: `.Result`/`.Wait()` wrap exceptions in `AggregateException`; `.GetAwaiter().GetResult()` does not.

ASP.NET Core has no SC so it won't classic-deadlock, but blocking still causes thread-pool starvation (Topic 19).

### 6.1 [Easy] Why does `.Result` deadlock in a UI app?
**Answer**
The UI thread blocks waiting for the Task, but the Task's continuation is posted to the UI thread's SynchronizationContext, which can't run because the thread is blocked. Each waits for the other.

**Explanation**
Two circular dependencies: Task completion depends on continuation running on the UI thread; UI thread depends on Task completion.

**Code**
```csharp
// WinForms/WPF button handler (conceptual)
void Button_Click(object s, EventArgs e)
{
    var text = GetTextAsync().Result;   // DEADLOCK
}
static async Task<string> GetTextAsync()
{
    await Task.Delay(100);              // captures UI SC
    return "hi";                        // needs the UI thread
}
```

### 6.2 [Follow-up] How do you fix it?
**Answer**
Make the caller async (`await GetTextAsync()`) — the real fix. If you can't (e.g., library called by sync code), use `ConfigureAwait(false)` inside the async method so it doesn't need the original context.

**Explanation**
Fixing only inside the callee protects against that deadlock but all downstream awaits must also use it; any one missing await without ConfigureAwait(false) can reintroduce it. Blocking still wastes a thread.

**Code**
```csharp
async void Button_Click(object s, EventArgs e)    // event handler: async void is ok
{
    var text = await GetTextAsync();               // no deadlock
}
static async Task<string> GetTextAsync()
{
    await Task.Delay(100).ConfigureAwait(false);
    return "hi";
}
```

### 6.3 [Medium] Reproduce the deadlock in a console app so it can be tested.
**Answer**
Install a single-threaded SynchronizationContext (like the one in Topic 5.4), then block on `.Result` on that thread. Alternatively use xUnit's, or `Nito.AsyncEx.AsyncContext`.

**Explanation**
Plain console apps don't deadlock because SC is null. To demo it, set `SynchronizationContext.SetSynchronizationContext(new SingleThreadSyncContext())` and block from within a posted work item (the pump thread).

**Code**
```csharp
// Uses SingleThreadSyncContext from Topic 5.4
var ctx = new SingleThreadSyncContext();
ctx.Post(_ =>
{
    // await-based method needing this context:
    var r = Compute().Wait(2000);  // times out -> false (deadlock avoided by timeout)
    Console.WriteLine($"Completed within 2s? {r}");
    ctx.Complete();
}, null);
ctx.RunOnCurrentThread();

static async Task Compute() { await Task.Delay(100); }  // continuation posted to ctx, which is blocked
// Output: Completed within 2s? False
```

### 6.4 [Hard] Why does ASP.NET Core not classic-deadlock, yet `.Result` is still harmful there?
**Answer**
There's no SC, so continuations run on the pool, so the blocked thread isn't needed for completion. But each `.Result` pins a pool thread; under load all pool threads block waiting for continuations that need pool threads → starvation, latency spikes, and possible effective deadlock when the pool can't inject threads fast enough.

**Explanation**
Pool threads injected ≈ 1–2/s beyond min. With 100 concurrent requests each blocking, the continuations queue behind the blocked ones. Symptoms: high latency, low CPU, many threads waiting.

**Code**
```csharp
// Bad in ASP.NET Core:
app.MapGet("/bad", () => DoWorkAsync().Result);
// Good:
app.MapGet("/good", async () => await DoWorkAsync());
static async Task<string> DoWorkAsync() { await Task.Delay(100); return "ok"; }
```

**Gotchas / follow-ups**
- Use `dotnet-counters` (`ThreadPool Queue Length`, `ThreadPool Thread Count`) to diagnose.

### 6.5 [Tricky] Is `.GetAwaiter().GetResult()` safe? What about `Task.Run(...).Result`?
**Answer**
`GetAwaiter().GetResult()` avoids `AggregateException` wrapping but still blocks and can deadlock. `Task.Run(() => AsyncMethod()).GetAwaiter().GetResult()` avoids the SC deadlock (the async method runs on the pool without SC) but still blocks a thread and is a last-resort hack.

**Explanation**
`Task.Run` with an async lambda unwraps to the inner task. Because the lambda runs with `SynchronizationContext.Current == null`, no capture happens. Remaining cost: blocked thread, potential starvation.

**Code**
```csharp
static string Sync() => Task.Run(() => GetAsync()).GetAwaiter().GetResult(); // no SC deadlock
static async Task<string> GetAsync() { await Task.Delay(10); return "x"; }

try { Task.Run(() => throw new ArgumentException("a")).Wait(); }
catch (AggregateException ex) { Console.WriteLine(ex.InnerException!.GetType().Name); } // Output: ArgumentException
try { Task.Run(() => throw new ArgumentException("a")).GetAwaiter().GetResult(); }
catch (ArgumentException ex) { Console.WriteLine("direct: " + ex.GetType().Name); }     // Output: direct: ArgumentException
```

**Gotchas / follow-ups**
- Async `Main` and `IHostedService` remove most reasons to block.
- Constructors/properties can't be async → use async factory methods (`static async Task<T> CreateAsync()`) or lazy init (`AsyncLazy<T>`).

**Interview traps**
- Deadlock needs a single-thread SC + blocking; ASP.NET Core variant is starvation, not SC deadlock.
- `ConfigureAwait(false)` is a mitigation, not a fix.
- `.Wait()` / `.Result` → AggregateException; `GetAwaiter().GetResult()` → raw exception.
- Console apps don't reproduce it without a custom SC.

**Cheat sheet**
- Don't block on async: `await`.
- Classic deadlock = UI/legacy ASP.NET + `.Result`.
- Mitigation: `ConfigureAwait(false)` in libraries.
- Last resort: `Task.Run(...).GetAwaiter().GetResult()`.
- Blocking in ASP.NET Core = starvation.

---

## Topic 7: Task vs ValueTask

**Study notes**

- `Task<T>` is a **reference type** (heap alloc per call unless cached). `ValueTask<T>` is a **struct** wrapping either a result `T` or a `Task<T>` or an `IValueTaskSource<T>` — avoiding allocation when the operation completes synchronously (hot path) or when backed by a pooled source.
- Use `ValueTask` when: the method often completes synchronously (cache hit, buffered read) *and* is on a hot path. Otherwise `Task`.
- **Rules of ValueTask** (consume each instance only **once**):
  - Await it once; don't await it multiple times or concurrently.
  - Don't call `.Result`/`.GetAwaiter().GetResult()` unless completed (`IsCompleted`).
  - If you need to do multiple things, call `.AsTask()` (once) and use the Task.
  - Don't `WhenAll` on ValueTasks directly (convert via `AsTask()`).
- Struct size: `ValueTask<T>` has an object ref + T + short token + bool → bigger than a reference, costly to copy for large T.
- Non-generic `ValueTask` and `ValueTask<T>` exist (.NET Core 2.1+/Standard 2.1); `Task.CompletedTask`, `ValueTask.CompletedTask` (.NET 5+), `ValueTask.FromResult`.

### 7.1 [Easy] What is a ValueTask and why does it exist?
**Answer**
A struct awaitable that can represent a completed result without allocating a Task. It reduces allocations when an async method frequently completes synchronously, such as a cached read.

**Explanation**
`Task<T>` costs an object (~72+ bytes) per non-cached completion; at millions of calls/s this creates GC pressure. `ValueTask<T>` stores the result inline in the struct.

**Code**
```csharp
class Cache
{
    private readonly Dictionary<int, string> _d = new();
    public ValueTask<string> GetAsync(int key)
    {
        if (_d.TryGetValue(key, out var v)) return new ValueTask<string>(v); // no allocation
        return new ValueTask<string>(LoadAsync(key));
    }
    private async Task<string> LoadAsync(int key)
    {
        await Task.Delay(10);
        return _d[key] = $"v{key}";
    }
}
```

### 7.2 [Follow-up] When should you NOT use ValueTask?
**Answer**
When the result is usually asynchronous, when callers need to await multiple times, `WhenAll/WhenAny`, store it, or when the benefit isn't measured. Default to `Task`; use `ValueTask` only after profiling shows allocation hot spots.

**Explanation**
ValueTask has a misuse risk (multiple awaits, concurrent awaits, use after completion with pooled sources → undefined behavior/data corruption). It's also larger, so passing it around costs more. It's mainly for library/framework hot paths (`Stream.ReadAsync`, `Channel.Reader.ReadAsync`, `IAsyncEnumerator.MoveNextAsync`).

**Code**
```csharp
ValueTask<int> vt = new ValueTask<int>(5);
var a = await vt;
// var b = await vt; // allowed here since it holds a value, but is undefined behaviour with pooled sources. Never rely on it.
```

### 7.3 [Medium] How do you correctly consume a ValueTask when you need it more than once, or with WhenAll?
**Answer**
Call `.AsTask()` once and use the resulting Task for multiple awaits/combinators. `Task.WhenAll` takes `Task`s, not `ValueTask`s.

**Explanation**
`AsTask()` on a ValueTask wrapping a `Task` returns that same Task; wrapping a result creates `Task.FromResult`; wrapping a pooled source allocates a Task and the source is consumed.

**Code**
```csharp
static ValueTask<int> A() => new(1);
static ValueTask<int> B() => new(Task.Run(() => 2));
var results = await Task.WhenAll(A().AsTask(), B().AsTask());
Console.WriteLine(string.Join(",", results)); // Output: 1,2
```

### 7.4 [Hard] How does `IValueTaskSource<T>` allow zero-allocation async?
**Answer**
You implement `IValueTaskSource<T>` (often via `ManualResetValueTaskSourceCore<T>`) in a reusable (poolable) object; the ValueTask holds a reference to the source + a *token* (version). The source's `GetStatus/OnCompleted/GetResult` are invoked by the awaiter, and the token detects stale usage after the source is reused.

**Explanation**
Used by `Socket`, `Channel`, `PipeReader`. The version token is why awaiting the same ValueTask twice may throw `InvalidOperationException` or silently read another operation's result if the source was reused.

**Code**
```csharp
using System.Threading.Tasks.Sources;

sealed class SimpleSource : IValueTaskSource<int>
{
    private ManualResetValueTaskSourceCore<int> _core = new() { RunContinuationsAsynchronously = true };
    public ValueTask<int> WaitAsync() => new(this, _core.Version);
    public void Complete(int v) => _core.SetResult(v);
    public int GetResult(short token) { var r = _core.GetResult(token); _core.Reset(); return r; }
    public ValueTaskSourceStatus GetStatus(short token) => _core.GetStatus(token);
    public void OnCompleted(Action<object?> c, object? s, short token, ValueTaskSourceOnCompletedFlags f)
        => _core.OnCompleted(c, s, token, f);
}
// var src = new SimpleSource(); var vt = src.WaitAsync(); src.Complete(5); Console.WriteLine(await vt); // Output: 5
```

**Gotchas / follow-ups**
- Reset in GetResult means the token is invalidated: second await fails.
- Never share across concurrent operations.

### 7.5 [Tricky] What happens if you await the same ValueTask twice?
**Answer**
It's undefined. If it wraps a plain result or a normal `Task`, it appears to work; if backed by an `IValueTaskSource` that has been reset/reused, it may throw or return wrong data. Treat as single-use.

**Explanation**
Analyzers CA2012 ("Use ValueTasks correctly") flag misuse. Also don't ignore a ValueTask without awaiting (can leak pooled sources / swallow exceptions).

**Code**
```csharp
async ValueTask<int> F() { await Task.Yield(); return 1; }
var v = F();
Console.WriteLine(await v);        // Output: 1
// await v again -> unsupported; use v.AsTask() once if you need multiple consumers
var t = F().AsTask();
Console.WriteLine(await t + await t); // Output: 2   (Task can be awaited many times)
```

**Interview traps**
- "ValueTask is always faster" — no; misuse and size overheads.
- No `WhenAll` on raw ValueTasks.
- Never `.Result` an incomplete ValueTask.
- `Task.FromResult` caches only some values (e.g., `Task<bool>`, small ints; behavior is an implementation detail).

**Cheat sheet**
- Task = ref type, reusable many awaits.
- ValueTask = struct, await once.
- Use when sync-completion common + hot path.
- `AsTask()` to convert (once).
- CA2012 analyzer enforces rules.

---

## Topic 8: async void & exception handling in async

**Study notes**

- `async void`: no Task returned → caller can't await, can't catch exceptions. An exception thrown is **posted to the SynchronizationContext** (or rethrown on a pool thread if none) → typically **crashes the process**. Only use for event handlers.
- `async Task` captures exceptions in the Task; they surface at `await` (first exception unwrapped) or `.Wait()/.Result` (`AggregateException`).
- `try/catch` around an un-awaited call catches nothing from the async part.
- `Task.WhenAll` faulted with several exceptions: `await` throws the first; `task.Exception.InnerExceptions` holds all.
- Unobserved faulted Tasks: `TaskScheduler.UnobservedTaskException` is raised on finalization, but process doesn't crash by default (.NET 4.5+).
- Async lambdas assigned to `Action` become `async void` silently! (e.g., `list.ForEach(async x => ...)`).

### 8.1 [Easy] Why is `async void` discouraged?
**Answer**
Callers can't await it or observe its exceptions, and an exception thrown inside can crash the app. It is only appropriate for event handlers whose signature must return void.

**Explanation**
It also makes unit testing hard (can't await completion) and the caller can't know when it finishes.

**Code**
```csharp
static async void Bad() { await Task.Delay(10); throw new Exception("boom"); }
static async Task Good() { await Task.Delay(10); throw new Exception("boom"); }

try { Bad(); }          // does NOT catch; exception later crashes process (console: unhandled)
catch { Console.WriteLine("never"); }

try { await Good(); }   // caught
catch (Exception e) { Console.WriteLine($"caught {e.Message}"); } // Output: caught boom
```

### 8.2 [Follow-up] Where does the exception from `async Task` appear if I forget to await?
**Answer**
It sits in the Task and nobody sees it (silently lost, "fire-and-forget"). It's raised via `UnobservedTaskException` when the Task is GC'd but won't crash by default.

**Explanation**
Always await, store, or attach a continuation/handler. Compiler warns CS4014 for un-awaited calls within async methods.

**Code**
```csharp
static async Task Fail() { await Task.Yield(); throw new InvalidOperationException("lost"); }
_ = Fail();                    // exception silently lost
await Task.Delay(100);
Console.WriteLine("still running"); // Output: still running
```

### 8.3 [Medium] What exception do I get from await vs .Wait vs WhenAll?
**Answer**
`await` rethrows the first inner exception, not an `AggregateException`. `.Wait()`/`.Result` throw `AggregateException`. For `WhenAll` with multiple failures, `await` throws only the first; read `task.Exception!.InnerExceptions` for all.

**Explanation**
This mirrors synchronous semantics for single failures. For all errors, keep a reference to the WhenAll Task.

**Code**
```csharp
async Task F(string m) { await Task.Yield(); throw new Exception(m); }
var all = Task.WhenAll(F("a"), F("b"));
try { await all; }
catch (Exception e)
{
    Console.WriteLine(e.Message);                                   // Output: a (or b: first in argument order -> a)
    Console.WriteLine(all.Exception!.InnerExceptions.Count);        // Output: 2
}
```

### 8.4 [Hard] How do exceptions escape from `async void` and how can you handle them globally?
**Answer**
They are rethrown via `SynchronizationContext.Post` (UI: `Application.ThreadException`/`DispatcherUnhandledException`), or on the pool if no SC, ending in `AppDomain.UnhandledException` and process termination. Hook those events to log, but can't recover reliably.

**Explanation**
`AsyncVoidMethodBuilder.SetException` captures the SC at call time (`AsyncVoidMethodBuilder.Create`), increments the pending-operation count (`OnUndelayedOperationStarted`) so test SCs can await completion. In ASP.NET Core, an unhandled `async void` exception takes the host down.

**Code**
```csharp
AppDomain.CurrentDomain.UnhandledException += (_, e) =>
    Console.WriteLine($"Unhandled: {((Exception)e.ExceptionObject).Message}");

TaskScheduler.UnobservedTaskException += (_, e) =>
{
    Console.WriteLine($"Unobserved: {e.Exception.InnerException?.Message}");
    e.SetObserved();
};
// async void Bad() { throw ... } would reach UnhandledException then terminate.
```

**Gotchas / follow-ups**
- `UnobservedTaskException` only fires after GC finalizes the Task — non-deterministic.

### 8.5 [Tricky] What's the bug in `list.ForEach(async x => await ProcessAsync(x))`?
**Answer**
`List<T>.ForEach` takes `Action<T>`, so the async lambda compiles as `async void`; ForEach returns immediately without waiting and exceptions are unobservable. Use `await Task.WhenAll(list.Select(ProcessAsync))` or a `foreach` with await (or `Parallel.ForEachAsync`).

**Explanation**
Same trap with `Parallel.ForEach(items, async x => ...)`, `Array.ForEach`, `Timer` callbacks, `Task.Run(Action)`-style overloads (though `Task.Run(async () => ...)` correctly picks `Func<Task>`).

**Code**
```csharp
var items = new List<int> { 1, 2, 3 };
items.ForEach(async i => { await Task.Delay(100); Console.WriteLine($"done {i}"); });
Console.WriteLine("ForEach returned");
// Output: ForEach returned  (the 'done' lines appear later or not at all if process ends)

await Task.WhenAll(items.Select(async i => { await Task.Delay(100); Console.WriteLine($"ok {i}"); }));
// Output: ok 1, ok 2, ok 3 (order may vary) before continuing
```

**Gotchas / follow-ups**
- `Task.Run(async () => ...)` is fine (overload resolves to `Func<Task>`).

**Interview traps**
- `try/catch` around un-awaited async call catches nothing.
- Async lambdas to `Action` = hidden async void.
- `await` unwraps; `.Result` wraps in AggregateException.
- Fire-and-forget loses exceptions.

**Cheat sheet**
- async void only for event handlers.
- Return `Task`, always await.
- `WhenAll` → keep task to read all exceptions.
- Hook `UnhandledException` / `UnobservedTaskException` for logging only.
- CS4014 warns of un-awaited call.

---

## Topic 9: Task.WhenAll / WhenAny / Run / FromResult / Delay

**Study notes**

| API | Purpose | Notes |
|---|---|---|
| `Task.Run(Func)` | Queue CPU work to pool | Unwraps async lambdas (`Func<Task>`) |
| `Task.FromResult(x)` | Completed Task with value | No async machinery; use for sync implementations of async interfaces |
| `Task.CompletedTask` | Cached completed Task | |
| `Task.FromException/FromCanceled` | Faulted/cancelled completed tasks | |
| `Task.Delay(ms/ts[, ct])` | Timer-based async wait | Cancelable; `Delay(0)` completes sync |
| `Task.WhenAll(tasks)` | Completes when all complete | Result array in input order; aggregates exceptions; if any canceled & none faulted → canceled |
| `Task.WhenAny(tasks)` | Completes when first completes | Returns the winning Task; others keep running |
| `Task.WaitAll/WaitAny` | **Blocking** versions | Avoid in async code |
| `Task.WhenEach` | **.NET 9+** stream tasks as they complete | Not in .NET 8 |
| `Task.WaitAsync(timeout/ct)` | .NET 6+ add timeout/cancel to any Task | Underlying work not canceled |

### 9.1 [Easy] How do you run multiple async calls concurrently and wait for all?
**Answer**
Start them without awaiting each, collect the Tasks, then `await Task.WhenAll(tasks)`. Awaiting inside a loop runs them sequentially.

**Explanation**
Tasks begin running (hot) when the async method is called; `WhenAll` only awaits them. The result array preserves the order of the input tasks, not completion order.

**Code**
```csharp
static async Task<int> Work(int i) { await Task.Delay(100 * (4 - i)); return i; }

var sw = System.Diagnostics.Stopwatch.StartNew();
int[] r = await Task.WhenAll(Work(1), Work(2), Work(3));
Console.WriteLine($"{string.Join(",", r)} in ~{sw.ElapsedMilliseconds}ms");
// Output: 1,2,3 in ~300ms   (not 600ms sequential)
```

### 9.2 [Follow-up] How does WhenAny work, and how do you implement a timeout with it?
**Answer**
`WhenAny` returns the first Task to complete (success, fault, or cancel). For a timeout, race the work against `Task.Delay`; but the loser isn't canceled — pass a `CancellationToken`. In .NET 6+ prefer `task.WaitAsync(TimeSpan)`, which throws `TimeoutException`.

**Explanation**
WhenAny doesn't observe exceptions of the others, nor cancel them. If the delay wins, the original work continues running in background — leaking it unless canceled.

**Code**
```csharp
static async Task<int> Slow(CancellationToken ct) { await Task.Delay(5000, ct); return 1; }

using var cts = new CancellationTokenSource();
var work = Slow(cts.Token);
var winner = await Task.WhenAny(work, Task.Delay(200));
if (winner != work) { cts.Cancel(); Console.WriteLine("timed out"); } // Output: timed out

try { await Slow(CancellationToken.None).WaitAsync(TimeSpan.FromMilliseconds(100)); }
catch (TimeoutException) { Console.WriteLine("WaitAsync timeout"); } // Output: WaitAsync timeout
```

### 9.3 [Medium] Task.Run vs Task.Factory.StartNew vs `new Task().Start()` — and what happens with an async lambda?
**Answer**
`Task.Run` = `StartNew` with safe defaults (`TaskScheduler.Default`, `DenyChildAttach`) and unwraps `Func<Task>`. `Task.Factory.StartNew(async () => ...)` returns `Task<Task>` (needs `.Unwrap()`), and uses `TaskScheduler.Current` (could be a surprising scheduler). Prefer `Task.Run`.

**Explanation**
Forgetting `Unwrap` means awaiting only the *start* of the inner work; exceptions from inner Task unobserved.

**Code**
```csharp
Task<Task> outer = Task.Factory.StartNew(async () => { await Task.Delay(100); Console.WriteLine("inner done"); });
await outer;                 // completes immediately-ish, inner not finished
Console.WriteLine("after outer");
await outer.Unwrap();        // now waits
// Output: after outer / inner done   (order of first two lines may vary)

await Task.Run(async () => { await Task.Delay(100); Console.WriteLine("Run: inner done"); }); // properly awaits inner
```

### 9.4 [Hard] Behavior of WhenAll with faulted/canceled tasks and empty input?
**Answer**
It waits for **all** tasks even if one faults early. Result: Faulted if any faulted (all exceptions aggregated), else Canceled if any canceled, else RanToCompletion. Empty input returns an already-completed Task (result empty array).

**Explanation**
This is not fail-fast. For fail-fast semantic, use `WhenAny` loop or cancellation tokens so peers stop on first failure.

**Code**
```csharp
var ok = Task.Delay(300);
var bad = Task.FromException(new InvalidOperationException("bad"));
var sw = System.Diagnostics.Stopwatch.StartNew();
try { await Task.WhenAll(ok, bad); } catch { }
Console.WriteLine($"waited ~{sw.ElapsedMilliseconds}ms"); // Output: waited ~300ms (waits for slow one even though 'bad' already failed)

var empty = await Task.WhenAll(Array.Empty<Task<int>>());
Console.WriteLine(empty.Length); // Output: 0
```

**Gotchas / follow-ups**
- Materialize the enumerable (`ToList()`) before `WhenAll` to avoid multiple enumeration starting tasks twice.
- Unbounded `WhenAll` over 10k items can overwhelm resources: throttle with `SemaphoreSlim` or `Parallel.ForEachAsync`.

### 9.5 [Tricky] What does this print? Delay vs Thread.Sleep inside tasks.
**Answer**
Sequential total ≈ 2 s for the awaited version in a loop; ≈ 1 s with `WhenAll`. `Thread.Sleep` inside `Task.Run` also takes ~1 s total in parallel but burns two pool threads.

**Explanation**
Lazy LINQ deferral: `Select` creates tasks only when enumerated; enumerating twice starts the work twice.

**Code**
```csharp
var sw = System.Diagnostics.Stopwatch.StartNew();
var q = Enumerable.Range(0, 2).Select(async i => { await Task.Delay(1000); return i; }); // lazy
foreach (var t in q) await t;                      // each enumerated + awaited in turn
Console.WriteLine($"sequential: {sw.ElapsedMilliseconds / 1000}s");   // Output: sequential: 2s

sw.Restart();
await Task.WhenAll(q);                              // enumerates now -> starts both
Console.WriteLine($"parallel: {sw.ElapsedMilliseconds / 1000}s");     // Output: parallel: 1s
```

**Interview traps**
- WhenAll isn't fail-fast.
- `StartNew(async...)` returns `Task<Task>`.
- Lazy `Select` + multiple enumeration = duplicate work.
- `WhenAny` doesn't cancel losers.
- `Task.Delay(0)` completes synchronously.

**Cheat sheet**
- `WhenAll` = parallel await; keep result order.
- `WhenAny` + `Delay` or `WaitAsync` = timeout.
- Prefer `Task.Run` over `StartNew`.
- `FromResult` for sync implementations.
- `WaitAll/WaitAny` block — avoid.

---

## Topic 10: CancellationToken, linked tokens, timeouts

**Study notes**

- **Cooperative cancellation**: `CancellationTokenSource` (CTS) produces a `CancellationToken` (struct); consumers observe it; nothing is forcibly aborted (`Thread.Abort` unsupported in .NET Core).
- Consumers: `ct.ThrowIfCancellationRequested()` → throws `OperationCanceledException` (OCE); `ct.IsCancellationRequested` polling; `ct.Register(callback)`; pass to APIs (`HttpClient`, `Task.Delay`, EF Core `ToListAsync(ct)`).
- Task becomes **Canceled** (not Faulted) when an OCE whose token equals the task's token propagates out (for `Task.Run(..., ct)` or async methods that throw an OCE).
- `CancelAfter(TimeSpan)` / `new CancellationTokenSource(TimeSpan)` for timeouts. `CancellationTokenSource.CreateLinkedTokenSource(a, b)` cancels when any source cancels.
- **Dispose** CTS when done (especially timers/linked sources) to release resources/registrations.
- .NET 8: `cts.CancelAsync()`, `Task.WaitAsync`, `CancellationTokenSource.TryReset()` (.NET 6+), `TimeProvider` overloads.

### 10.1 [Easy] How does cancellation work in .NET?
**Answer**
It's cooperative: a CancellationTokenSource signals via its token, and the running code checks the token (or passes it to cancellable APIs) and stops by throwing `OperationCanceledException`. Nothing is killed forcibly.

**Explanation**
Producer/consumer separation: the CTS owner can cancel; token holders can only observe. Always propagate tokens through the call chain, including ASP.NET Core's `HttpContext.RequestAborted` (can be bound as a `CancellationToken` param in controller/minimal API actions).

**Code**
```csharp
using var cts = new CancellationTokenSource();
var t = Task.Run(async () =>
{
    for (int i = 0; ; i++)
    {
        cts.Token.ThrowIfCancellationRequested();
        Console.WriteLine($"tick {i}");
        await Task.Delay(100, cts.Token);
    }
}, cts.Token);
await Task.Delay(250);
cts.Cancel();
try { await t; } catch (OperationCanceledException) { Console.WriteLine($"Status: {t.Status}"); }
// Output: tick 0, tick 1, tick 2, Status: Canceled
```

### 10.2 [Follow-up] How do you add a timeout, and how do you combine a timeout with a caller token?
**Answer**
Use `CancelAfter` on a CTS; to also honor the caller's token, create a linked CTS: `CancellationTokenSource.CreateLinkedTokenSource(callerToken)` then `.CancelAfter(timeout)`.

**Explanation**
Linked CTS registers with the parent; dispose it to unregister. To distinguish timeout from caller cancel, check `callerToken.IsCancellationRequested` in the catch.

**Code**
```csharp
static async Task<string> CallAsync(CancellationToken callerCt)
{
    using var linked = CancellationTokenSource.CreateLinkedTokenSource(callerCt);
    linked.CancelAfter(TimeSpan.FromMilliseconds(200));
    try
    {
        await Task.Delay(1000, linked.Token);
        return "done";
    }
    catch (OperationCanceledException) when (!callerCt.IsCancellationRequested)
    {
        return "timeout";
    }
}
Console.WriteLine(await CallAsync(CancellationToken.None)); // Output: timeout
```

### 10.3 [Medium] Canceled vs Faulted: how does a Task end up Canceled?
**Answer**
If the async method/delegate throws an `OperationCanceledException` (for async methods any OCE; for `Task.Run(action, token)` the OCE's token must match the token passed to Run and be signaled), the Task is Canceled; other exceptions → Faulted. `await` on a canceled Task throws `OperationCanceledException` (`TaskCanceledException` derives from it).

**Explanation**
Catch `OperationCanceledException` to handle both. Don't log cancellation as an error. `Task.Delay(…, ct)` throws `TaskCanceledException`.

**Code**
```csharp
using var cts = new CancellationTokenSource();
cts.Cancel();
var t1 = Task.Run(() => { cts.Token.ThrowIfCancellationRequested(); }, cts.Token);
var t2 = Task.Run(() => throw new InvalidOperationException());
try { await t1; } catch (OperationCanceledException) { }
try { await t2; } catch (InvalidOperationException) { }
Console.WriteLine($"{t1.Status}, {t2.Status}"); // Output: Canceled, Faulted
```

### 10.4 [Hard] How do you make blocking/legacy code cancellable, and use `Register`?
**Answer**
Use `ct.Register(() => ...)` to invoke a callback on cancellation (e.g., close a socket, `Dispose` a resource, or `SetCanceled` a TCS). Dispose the returned `CancellationTokenRegistration` to unregister. Callbacks run synchronously on the thread calling `Cancel()` — keep them quick.

**Explanation**
If a callback throws, `Cancel()` throws `AggregateException` (after running all callbacks). `Cancel()` blocks until callbacks run (use `CancelAsync` in .NET 8 to avoid blocking the caller on callbacks).

**Code**
```csharp
static Task<int> LegacyAsync(CancellationToken ct)
{
    var tcs = new TaskCompletionSource<int>(TaskCreationOptions.RunContinuationsAsynchronously);
    var reg = ct.Register(() => tcs.TrySetCanceled(ct));
    // ... start legacy op that calls tcs.TrySetResult(x) later ...
    tcs.Task.ContinueWith(_ => reg.Dispose(), TaskScheduler.Default);
    return tcs.Task;
}
using var cts = new CancellationTokenSource(100);
try { await LegacyAsync(cts.Token); } catch (OperationCanceledException) { Console.WriteLine("canceled"); } // Output: canceled
```

**Gotchas / follow-ups**
- Registration with `UnsafeRegister` skips ExecutionContext capture (perf).
- `Cancel()` then dispose: avoid disposing while a `Cancel` is running concurrently.

### 10.5 [Tricky] Why isn't my timeout working / what are common cancellation bugs?
**Answer**
Typical causes: the token wasn't passed down to the actual I/O call; CPU loops don't poll the token; swallowing OCE in a generic `catch (Exception)`; CTS disposed (ObjectDisposedException on `.Token` access) or garbage-collected; using `Task.WhenAny(work, Delay)` without cancelling work.

**Explanation**
A CTS with `CancelAfter` has an internal timer; if you don't keep the CTS reachable/dispose properly, behaviors can surprise. Always `using` the CTS for the duration of the operation (not before it ends). Also `catch (Exception)` must exclude OCE or rethrow.

**Code**
```csharp
static async Task Bad(CancellationToken ct)
{
    // token never forwarded -> not cancellable
    await Task.Delay(5000);
}
static async Task Good(CancellationToken ct)
{
    await Task.Delay(5000, ct);
}
using var cts = new CancellationTokenSource(200);
try { await Good(cts.Token); } catch (OperationCanceledException) { Console.WriteLine("Good canceled"); }
// Output: Good canceled  (Bad would run the full 5 s)
```

**Interview traps**
- Cancellation is cooperative, not preemptive.
- `OperationCanceledException` ≠ error; don't log as such.
- Dispose linked/timeout CTS.
- A token isn't cancelled by `Dispose`.
- Pass `HttpContext.RequestAborted` / `CancellationToken` params through to EF Core and HttpClient.

**Cheat sheet**
- CTS owns; Token observes.
- `CancelAfter` for timeouts; linked CTS to combine.
- `ThrowIfCancellationRequested` in loops.
- `Register` to bridge legacy APIs.
- Canceled status needs OCE with matching token.

---

## Topic 11: IAsyncEnumerable & await foreach

**Study notes**

- C# 8 / .NET Core 3.0+: `IAsyncEnumerable<T>` with `GetAsyncEnumerator(CancellationToken)` returning `IAsyncEnumerator<T>` (`ValueTask<bool> MoveNextAsync()`, `T Current`, `ValueTask DisposeAsync()`).
- Produce with an `async` iterator: `async IAsyncEnumerable<T> M() { ... await ...; yield return x; }`.
- Consume with `await foreach (var x in source)`. Add `.WithCancellation(ct)` and `.ConfigureAwait(false)`.
- `[EnumeratorCancellation] CancellationToken ct` parameter attribute lets `WithCancellation` flow into the iterator.
- Lazy / streaming: items produced on demand, low memory. EF Core: `AsAsyncEnumerable()` streams results; ASP.NET Core can stream `IAsyncEnumerable<T>` returns as JSON (buffered in older versions, streamed in .NET 6+ for System.Text.Json).
- LINQ over async streams: `System.Linq.Async` package (.NET 10 adds built-in; not in .NET 8 BCL).

### 11.1 [Easy] What is IAsyncEnumerable and when to use it?
**Answer**
It's the asynchronous counterpart of `IEnumerable<T>` where retrieving each next element can await. Use it for streaming data (paged APIs, DB rows, file lines) without loading everything in memory.

**Explanation**
Contrast: `Task<List<T>>` returns when everything is loaded; `IAsyncEnumerable<T>` lets consumers start processing the first item immediately.

**Code**
```csharp
static async IAsyncEnumerable<int> GenerateAsync(int count)
{
    for (int i = 1; i <= count; i++)
    {
        await Task.Delay(100);
        yield return i;
    }
}
await foreach (var n in GenerateAsync(3))
    Console.WriteLine(n);
// Output: 1 2 3 (one every ~100ms)
```

### 11.2 [Follow-up] How do you support cancellation in an async iterator?
**Answer**
Add a `[EnumeratorCancellation] CancellationToken ct = default` parameter to the iterator; consumers call `.WithCancellation(token)` and the token flows into it. Then pass it to inner awaits.

**Explanation**
Without the attribute, `WithCancellation` has no effect on your iterator's own token parameter (compiler warning CS8425). If you pass the token directly as an argument too, they can be combined with linked tokens.

**Code**
```csharp
using System.Runtime.CompilerServices;

static async IAsyncEnumerable<int> CountAsync([EnumeratorCancellation] CancellationToken ct = default)
{
    for (int i = 0; ; i++)
    {
        await Task.Delay(100, ct);
        yield return i;
    }
}
using var cts = new CancellationTokenSource(350);
try
{
    await foreach (var i in CountAsync().WithCancellation(cts.Token))
        Console.WriteLine(i);
}
catch (OperationCanceledException) { Console.WriteLine("cancelled"); }
// Output: 0 1 2 cancelled
```

### 11.3 [Medium] How does an async iterator compile, and how does `await foreach` dispose?
**Answer**
The compiler generates a state machine implementing `IAsyncEnumerable<T>`, `IAsyncEnumerator<T>`, and `IValueTaskSource` for `MoveNextAsync`; `yield return` stores `Current` and completes the `MoveNextAsync`. `await foreach` wraps the loop in `try/finally { await enumerator.DisposeAsync(); }`, so `finally` blocks and `await using` inside the iterator run even when you `break`.

**Explanation**
Breaking early disposes the enumerator, running the iterator's `finally` — good for releasing DB readers.

**Code**
```csharp
static async IAsyncEnumerable<int> Src()
{
    try
    {
        for (int i = 0; i < 5; i++) { await Task.Yield(); yield return i; }
    }
    finally { Console.WriteLine("cleanup"); }
}
await foreach (var i in Src())
{
    if (i == 1) break;
    Console.WriteLine(i);
}
// Output: 0 / cleanup
```

### 11.4 [Hard] Streaming vs buffering: sync `IEnumerable` over async source, and EF Core usage.
**Answer**
`IEnumerable<T>` can't await, so converting async data to it forces blocking (`.Result`). Use `IAsyncEnumerable<T>`. In EF Core, `await foreach (var x in db.Orders.AsAsyncEnumerable())` streams rows with the connection held open while you process; `ToListAsync()` buffers all rows then releases it.

**Explanation**
Streaming keeps a DataReader/connection open during consumer processing — slow consumers hold connections (pool exhaustion) and (without MARS) block other queries on the same DbContext/connection. Choose per use-case; for big exports stream, for typical APIs buffer with paging.

**Code**
```csharp
// EF Core (conceptual, assumes DbContext db)
// await foreach (var o in db.Orders.Where(o => o.Total > 100).AsAsyncEnumerable().WithCancellation(ct))
//     await WriteRowAsync(o);   // connection held until loop ends

static async IAsyncEnumerable<T> ToAsync<T>(IEnumerable<T> src)
{
    foreach (var x in src) { await Task.Yield(); yield return x; }
}
```

**Gotchas / follow-ups**
- Don't run another query on the same DbContext inside the loop (unless MARS enabled; SQL Server provider can't run two active readers by default).
- `ToListAsync` inside the producer then `yield` — not real streaming.

### 11.5 [Tricky] Can you use `yield return` inside try/catch or with `await` in a non-async method? What does `await foreach` over `IEnumerable` do?
**Answer**
`yield return` cannot appear inside a `try` block that has a `catch` clause (it can inside `try/finally`), same as sync iterators. You can't `await foreach` an `IEnumerable<T>` (compile error) — only types with `GetAsyncEnumerator`. You can `foreach` an `IAsyncEnumerable` only via `await foreach`.

**Explanation**
Workaround: catch exceptions around the `MoveNextAsync`-driven loop in the consumer, or manually wrap by catching in a helper that returns results/errors as values.

**Code**
```csharp
static async IAsyncEnumerable<int> Ok()
{
    try { await Task.Yield(); yield return 1; }   // OK: try/finally only
    finally { Console.WriteLine("fin"); }
}
// Not allowed: try { yield return 1; } catch { }  // CS1626
try
{
    await foreach (var x in Ok()) Console.WriteLine(x);
}
catch (Exception e) { Console.WriteLine(e.Message); }
// Output: 1 / fin
```

**Interview traps**
- Missing `[EnumeratorCancellation]` makes `WithCancellation` ineffective.
- `await foreach` early exit → `DisposeAsync` runs `finally`.
- Streaming EF queries hold connection open.
- No `yield return` in try-with-catch.

**Cheat sheet**
- `async IAsyncEnumerable<T>` + `yield return` + `await`.
- Consume with `await foreach`.
- `.WithCancellation(ct)`, `.ConfigureAwait(false)`.
- Lazy streaming = low memory.
- LINQ-async needs `System.Linq.Async` (pre-.NET 10).

---

## Topic 12: Parallel.For/ForEach/ForEachAsync & PLINQ

**Study notes**

- `Parallel.For/ForEach/Invoke` (TPL): data parallelism for **CPU-bound** work; the calling thread participates; partitions the range across pool threads; **blocks** until done. Options: `ParallelOptions { MaxDegreeOfParallelism, CancellationToken, TaskScheduler }`.
- Exceptions aggregate into `AggregateException`. `ParallelLoopState.Stop()/Break()`.
- `Parallel.ForEach` with async lambda = `async void` trap. Use **`Parallel.ForEachAsync`** (.NET 6+): `await Parallel.ForEachAsync(source, options, async (item, ct) => {...})` — async-friendly, bounded concurrency (default MaxDOP = processor count), supports `IAsyncEnumerable`.
- **PLINQ**: `source.AsParallel()....` ; `AsOrdered()` preserves order (cost), `WithDegreeOfParallelism`, `WithCancellation`, `ForAll` (unordered side-effect sink), `AsSequential()`. Merge buffering via `WithMergeOptions`.
- Overhead: partitioning, delegate calls, sync. Only helps when per-item work is substantial; small loops are slower than sequential.
- Shared-state: use thread-local accumulators (`Parallel.For` overload with `localInit/localFinally`) rather than locking per iteration.

### 12.1 [Easy] When to use Parallel.For vs async/await?
**Answer**
`Parallel.For` is for CPU-bound work to use multiple cores and it blocks the calling thread; async/await is for I/O-bound waiting, using no threads. Don't use Parallel.For for I/O (wastes threads) and don't use async for pure CPU (no speed-up).

**Explanation**
For I/O-bound fan-out use `Parallel.ForEachAsync` or `WhenAll` with throttling.

**Code**
```csharp
long total = 0;
Parallel.For(0, 1_000_000, () => 0L,
    (i, state, local) => local + i,
    local => Interlocked.Add(ref total, local));
Console.WriteLine(total); // Output: 499999500000
```

### 12.2 [Follow-up] How do you limit parallelism and handle async work per item?
**Answer**
Use `Parallel.ForEachAsync` with `ParallelOptions.MaxDegreeOfParallelism`; it awaits async bodies and passes a token. This replaces ad-hoc `SemaphoreSlim` throttling.

**Explanation**
The body is `Func<TSource, CancellationToken, ValueTask>`. Exceptions: the first one cancels the loop and is thrown (not AggregateException) when awaited.

**Code**
```csharp
var urls = Enumerable.Range(1, 10);
var opts = new ParallelOptions { MaxDegreeOfParallelism = 3 };
int running = 0, maxSeen = 0;
await Parallel.ForEachAsync(urls, opts, async (u, ct) =>
{
    int now = Interlocked.Increment(ref running);
    InterlockedMax(ref maxSeen, now);
    await Task.Delay(100, ct);
    Interlocked.Decrement(ref running);
});
Console.WriteLine($"max concurrent = {maxSeen}"); // Output: max concurrent = 3

static void InterlockedMax(ref int target, int v)
{
    int cur;
    while ((cur = Volatile.Read(ref target)) < v &&
           Interlocked.CompareExchange(ref target, v, cur) != cur) { }
}
```

### 12.3 [Medium] PLINQ: ordering, side effects and when it hurts.
**Answer**
`AsParallel()` doesn't preserve order unless `AsOrdered()`. Use `ForAll` for side effects to avoid merge overhead. PLINQ can be slower for cheap per-item work or when the source can't be range-partitioned (e.g., `IEnumerable` from a linked list → chunk partitioning with locking).

**Explanation**
PLINQ chooses partitioning (range for arrays/IList, chunk for generic enumerables, hash for GroupBy/Join). Exceptions are AggregateException. Avoid shared mutable state in lambdas.

**Code**
```csharp
var nums = Enumerable.Range(1, 20);
var unordered = nums.AsParallel().Where(n => n % 2 == 0).Select(n => n * n).ToList();
var ordered   = nums.AsParallel().AsOrdered().Where(n => n % 2 == 0).Select(n => n * n).ToList();
Console.WriteLine(string.Join(",", ordered)); // Output: 4,16,36,64,100,144,196,256,324,400
// 'unordered' has same elements, order not guaranteed
```

### 12.4 [Hard] How does Parallel.ForEach partition work and what's the "async lambda" bug?
**Answer**
Default partitioner chunks the source dynamically (chunks grow) so threads pull work as needed; it uses the calling thread plus pool threads, which may be injected slowly. If you pass an `async` lambda to `Parallel.ForEach`, it binds to `Action<T>` (async void): the loop finishes immediately after kicking off work, losing exceptions.

**Explanation**
Use `Parallel.ForEachAsync` or `Task.WhenAll`. Also `Parallel.ForEach` inside ASP.NET requests blocks a request thread and competes for pool threads — avoid on busy servers.

**Code**
```csharp
// BUG:
Parallel.ForEach(new[] { 1, 2, 3 }, async i => { await Task.Delay(500); Console.WriteLine(i); });
Console.WriteLine("loop done");   // Output: loop done  (before any number)

// FIX:
await Parallel.ForEachAsync(new[] { 1, 2, 3 }, async (i, ct) => { await Task.Delay(500, ct); Console.WriteLine(i); });
Console.WriteLine("really done"); // after all numbers
```

**Gotchas / follow-ups**
- Don't write to `List<T>` from parallel body: use `ConcurrentBag`, or `PLINQ` `.ToList()`.
- `Parallel.For` isn't guaranteed to use N threads.

### 12.5 [Tricky] Why can Parallel.For be slower than a plain for loop?
**Answer**
Delegate invocation per iteration, partition/sync overhead, false sharing, shared-state contention (locks/Interlocked per iteration), and memory-bound workloads can erase gains. Tiny bodies need chunking (range partitioner) or a sequential fallback.

**Explanation**
Use `Partitioner.Create(0, n)` to process ranges per delegate call. Measure with BenchmarkDotNet.

**Code**
```csharp
using System.Collections.Concurrent;
long sum = 0;
Parallel.ForEach(Partitioner.Create(0, 10_000_000), () => 0L,
    (range, _, local) => { for (int i = range.Item1; i < range.Item2; i++) local += i; return local; },
    local => Interlocked.Add(ref sum, local));
Console.WriteLine(sum); // Output: 49999995000000
```

**Interview traps**
- Parallel.ForEach + async lambda = async void.
- PLINQ not ordered by default.
- Shared counters inside parallel body without sync → wrong results.
- Don't use for I/O-bound work or on busy web servers.

**Cheat sheet**
- CPU-bound → Parallel/PLINQ; I/O-bound → async + `ForEachAsync`.
- `MaxDegreeOfParallelism` to throttle.
- `localInit/localFinally` for aggregation.
- `AsOrdered` costs; `ForAll` for sinks.
- Range partitioner for tiny bodies.

---

## Topic 13: Locks: lock/Monitor, SemaphoreSlim, Mutex, ReaderWriterLockSlim, SpinLock

**Study notes**

| Primitive | Scope | Reentrant | Async-capable | Use |
|---|---|---|---|---|
| `lock` / `Monitor` | In-process | Yes | No (can't `await` inside) | Short critical sections |
| `System.Threading.Lock` (.NET 9, C# 13) | In-process | Yes | No | Faster `lock` on dedicated type (**not in .NET 8**) |
| `SemaphoreSlim` | In-process | No | **Yes** (`WaitAsync`) | Throttling, async mutual exclusion |
| `Semaphore` | Cross-process (named) | No | No | Rare |
| `Mutex` | **Cross-process** (named) | Yes (thread-affine) | No | Single-instance app, machine-wide lock |
| `ReaderWriterLockSlim` | In-process | Optional | No | Many readers / few writers |
| `SpinLock` | In-process (struct) | No (by default) | No | Ultra-short sections; never copy it |
| `ManualResetEventSlim`/`AutoResetEvent` | signalling | — | — | Signal threads |

- `lock(obj)` compiles to `Monitor.Enter(obj, ref lockTaken)` + `try/finally Monitor.Exit`. Lock on a **private readonly object**; never on `this`, `typeof(X)`, strings or value types (boxing → different object each time → no mutual exclusion; compiler errors on value types).
- Monitor has `Wait/Pulse/PulseAll` for condition signalling.
- Locks use a thin lock (object header) → spin → fat lock (sync block) → kernel wait.

### 13.1 [Easy] How does `lock` work?
**Answer**
`lock(obj)` acquires a mutual-exclusion lock on the object's monitor so only one thread runs the block at a time; others block until released. It compiles to `Monitor.Enter` in a `try/finally` with `Monitor.Exit`, so it's released even on exceptions.

**Explanation**
Locks are reentrant: the same thread may re-acquire. Keep critical sections short and don't call unknown code inside.

**Code**
```csharp
object gate = new();
int counter = 0;
Parallel.For(0, 100_000, _ => { lock (gate) { counter++; } });
Console.WriteLine(counter); // Output: 100000
```

### 13.2 [Follow-up] Why shouldn't you lock on `this`, `typeof`, or a string?
**Answer**
Those objects are publicly reachable, so unrelated code can lock the same object → unexpected contention or deadlocks. Strings are interned (shared across the AppDomain). Use a private readonly `object`.

**Explanation**
Locking a boxed value type would box separately each time – different objects – no exclusion (the compiler rejects `lock` on value types directly, but passing a boxed `object` variable that changes is a bug). Also never lock on an object you reassign.

**Code**
```csharp
class Account
{
    private readonly object _sync = new();
    private decimal _balance;
    public void Deposit(decimal x) { lock (_sync) _balance += x; }
}
```

### 13.3 [Medium] When to use SemaphoreSlim vs Mutex vs ReaderWriterLockSlim?
**Answer**
`SemaphoreSlim` for in-process throttling (N concurrent) and the only one with `WaitAsync` for async code. `Mutex` for cross-process exclusion (named) with thread affinity. `ReaderWriterLockSlim` when reads vastly outnumber writes and sections are long enough to justify overhead.

**Explanation**
`Mutex` must be released by the owning thread — it can't be used across `await` where the thread changes (ApplicationException/`ApplicationException: Object synchronization method was called from an unsynchronized block of code`). RWLS has upgradeable read locks (only one at a time) and recursion policy; misuse deadlocks. Often a simple `lock` or immutable snapshot/`ConcurrentDictionary` is simpler and faster.

**Code**
```csharp
// Single-instance app across processes
using var m = new Mutex(initiallyOwned: true, name: @"Global\MyApp_SingleInstance", out bool createdNew);
Console.WriteLine(createdNew ? "first instance" : "already running");

// RWLS
var rw = new ReaderWriterLockSlim();
var dict = new Dictionary<int, string>();
string? Read(int k) { rw.EnterReadLock(); try { return dict.GetValueOrDefault(k); } finally { rw.ExitReadLock(); } }
void Write(int k, string v) { rw.EnterWriteLock(); try { dict[k] = v; } finally { rw.ExitWriteLock(); } }
Write(1, "a"); Console.WriteLine(Read(1)); // Output: a
```

### 13.4 [Hard] How does Monitor Wait/Pulse work? Implement a bounded signalling.
**Answer**
`Monitor.Wait(obj)` releases the lock and parks the thread in the wait queue; `Pulse`/`PulseAll` (called while holding the lock) move waiters to the ready queue; they re-acquire the lock before returning. Always wait in a `while(!condition)` loop (spurious/changed conditions).

**Explanation**
This is the building block of `BlockingCollection`-style producer/consumer. Prefer higher-level constructs (Channels, `BlockingCollection`) in new code.

**Code**
```csharp
var q = new Queue<int>();
var gate = new object();

var consumer = Task.Run(() =>
{
    lock (gate)
    {
        while (q.Count == 0) Monitor.Wait(gate);
        Console.WriteLine($"got {q.Dequeue()}");
    }
});
Thread.Sleep(100);
lock (gate) { q.Enqueue(42); Monitor.Pulse(gate); }
await consumer;
// Output: got 42
```

**Gotchas / follow-ups**
- `Monitor.Wait/Pulse` must be called inside `lock` or `SynchronizationLockException`.
- Lost wakeup if you Pulse before the consumer waits and check no condition.

### 13.5 [Tricky] What is SpinLock for, and what are its pitfalls? What about deadlock conditions in general?
**Answer**
`SpinLock` busy-waits instead of blocking — good only for extremely short sections with low contention. It's a mutable **struct**: never copy it or store it in a readonly field. Deadlock requires: mutual exclusion, hold-and-wait, no preemption, circular wait — prevent by consistent lock ordering or `TryEnter` with timeouts.

**Explanation**
Spinning wastes CPU and under oversubscription can starve the lock holder. Normal `lock` already spins briefly before blocking, so SpinLock rarely wins.

**Code**
```csharp
var sl = new SpinLock(enableThreadOwnerTracking: false);
int n = 0;
Parallel.For(0, 10_000, _ =>
{
    bool taken = false;
    try { sl.Enter(ref taken); n++; }
    finally { if (taken) sl.Exit(); }
});
Console.WriteLine(n); // Output: 10000

// Deadlock demo (avoid): thread1 locks A then B, thread2 locks B then A.
object A = new(), B = new();
// Fix: always lock in the same order (A then B), or use Monitor.TryEnter(B, timeout).
```

**Gotchas / follow-ups**
- `readonly SpinLock` field → each access works on a copy → no mutual exclusion.
- Detect deadlocks: dump (dotnet-dump → `syncblk`, `dumpasync`) or Parallel Stacks.

**Interview traps**
- Lock on private object, not `this`/`string`/`typeof`.
- `Mutex` and `lock` are thread-affine: don't hold across `await`.
- RWLS upgradeable lock: only one upgrader at a time.
- `System.Threading.Lock` is .NET 9, not available in .NET 8.

**Cheat sheet**
- `lock` = `Monitor` + try/finally; reentrant.
- `SemaphoreSlim` = throttling + async.
- `Mutex` = cross-process.
- RWLS = many readers, one writer.
- Order locks consistently to avoid deadlock.

---

## Topic 14: Interlocked, volatile & memory model basics

**Study notes**

- **Why a memory model?** CPUs and the JIT reorder/cache memory operations; without synchronization one thread may not see another's writes in order.
- **Atomicity**: reads/writes of ≤ native word-size primitives (int, reference) are atomic; `long`/`double` on 32-bit are not. `x++` is **not** atomic (read-modify-write).
- `Interlocked`: atomic `Increment`, `Decrement`, `Add`, `Exchange`, `CompareExchange`, `Read` (long on 32-bit), `And/Or` (.NET 5+). They imply **full memory fences**.
- `volatile` keyword: reads have *acquire* semantics, writes *release* semantics; prevents certain JIT hoisting (loop-invariant read caching). Does **not** make `++` atomic, nor provide a full fence (a write followed by a volatile read can still reorder — store-load).
- `Volatile.Read/Write`: same semantics as volatile, per-access.
- .NET memory model (CLR on x86/x64 strong-ish; ARM weaker; .NET 5+ makes writes release-like). Locks (`lock`, etc.) are full fences on enter/exit.
- Safe publication: construct, then publish via volatile write/lock/`Interlocked.Exchange`/`Lazy<T>`.
- CAS loop pattern for lock-free updates.

```
Thread A: data = 42; ready = true;         // release (volatile write)
Thread B: if (ready) use(data);            // acquire (volatile read) -> sees 42
```

### 14.1 [Easy] Why is `counter++` not thread-safe, and how does Interlocked fix it?
**Answer**
`counter++` is three steps (load, add, store); two threads can interleave and lose updates. `Interlocked.Increment` performs it as a single atomic CPU instruction.

**Explanation**
Use Interlocked for simple counters/flags; for compound invariants across multiple variables use `lock`.

**Code**
```csharp
int unsafeCnt = 0, safeCnt = 0;
Parallel.For(0, 1_000_000, _ => { unsafeCnt++; Interlocked.Increment(ref safeCnt); });
Console.WriteLine($"unsafe={unsafeCnt}, safe={safeCnt}");
// Output: unsafe=<less than 1000000, varies>, safe=1000000
```

### 14.2 [Follow-up] What does `volatile` do and not do?
**Answer**
It prevents the compiler/JIT/CPU from caching the field in a register or reordering in ways that break acquire/release (reads can't move earlier than prior ops; writes can't move later). It doesn't make compound operations atomic and doesn't give mutual exclusion.

**Explanation**
Classic use: stop flag polled in a loop. Without volatile, the JIT may hoist the read out of the loop → infinite loop in optimized builds.

**Code**
```csharp
class Worker
{
    private volatile bool _stop;
    public void Run() { while (!_stop) { /* work */ } Console.WriteLine("stopped"); }
    public void Stop() => _stop = true;
}
var w = new Worker();
var t = Task.Run(w.Run);
await Task.Delay(100);
w.Stop();
await t; // Output: stopped
```

### 14.3 [Medium] Explain Interlocked.CompareExchange and a lock-free update loop.
**Answer**
`CompareExchange(ref loc, newValue, comparand)` atomically sets `loc = newValue` only if `loc == comparand`, returning the original value. Loop: read current, compute new, CAS; retry if another thread changed it.

**Explanation**
Used for lock-free max/min, stack/queue nodes, lazy init of reference. Beware the ABA problem with pointer-like structures (less relevant with GC, where nodes aren't reused unless pooled).

**Code**
```csharp
static int AtomicMax(ref int target, int value)
{
    int initial, computed;
    do
    {
        initial = Volatile.Read(ref target);
        computed = Math.Max(initial, value);
    } while (Interlocked.CompareExchange(ref target, computed, initial) != initial);
    return computed;
}
int m = 0;
Parallel.For(0, 1000, i => AtomicMax(ref m, i));
Console.WriteLine(m); // Output: 999
```

### 14.4 [Hard] What is the double-checked locking pattern, and when is it safe in C#?
**Answer**
Check a field without lock, then lock and re-check before initializing. It's safe in .NET when the shared field is `volatile` (or published via `Volatile.Write`/`Interlocked`) because it ensures the constructed object is fully visible before the reference is. In practice use `Lazy<T>` or static initializers.

**Explanation**
Without proper publication a reader could see a non-null reference to a partially initialized object on weak memory models (ARM). `Lazy<T>(LazyThreadSafetyMode.ExecutionAndPublication)` is the default and safest. `LazyInitializer.EnsureInitialized` is another.

**Code**
```csharp
sealed class Singleton
{
    private static volatile Singleton? _inst;
    private static readonly object _gate = new();
    public static Singleton Instance
    {
        get
        {
            if (_inst is null)
                lock (_gate) { _inst ??= new Singleton(); }
            return _inst;
        }
    }
}
// Simpler & preferred:
sealed class Simple { public static readonly Lazy<Simple> Lazy = new(() => new Simple()); }
```

**Gotchas / follow-ups**
- `Lazy<T>` caches exceptions in `ExecutionAndPublication` mode; `PublicationOnly` retries.
- Static constructors/field initializers run once, thread-safe by the CLR.

### 14.5 [Tricky] Is `volatile` enough for Dekker/Peterson-style flags (store-load ordering)?
**Answer**
No. Volatile gives acquire/release, which still allows a write to be reordered *after* a subsequent read of a different variable (store→load reordering). Algorithms needing store-load ordering require a full fence: `Interlocked` ops or `Thread.MemoryBarrier()`.

**Explanation**
Classic example: two threads each set their own flag then read the other's; with volatile both may read false and both enter the critical section. `Thread.MemoryBarrier()` between write and read fixes it.

**Code**
```csharp
class Dekker
{
    private volatile bool _a, _b;
    public bool TryEnterA()
    {
        _a = true;
        Thread.MemoryBarrier();   // full fence: prevents store-load reordering
        if (_b) { _a = false; return false; }
        return true;
    }
}
```

**Gotchas / follow-ups**
- In real code, use `lock`/`Monitor`; hand-rolled lock-free is an interview topic, not a recommendation.
- `long` reads/writes on 32-bit not atomic: use `Interlocked.Read`.

**Interview traps**
- `volatile` ≠ atomic.
- `x++` and `x += y` are not atomic.
- Interlocked = atomic + full fence.
- DCL needs volatile/Lazy.

**Cheat sheet**
- Interlocked for atomic counters/flags.
- volatile = acquire/release visibility only.
- `lock` gives atomicity + visibility.
- CAS loop for lock-free updates.
- Prefer `Lazy<T>` for lazy singletons.

---

## Topic 15: Race conditions & thread-safety demos

**Study notes**

- **Race condition**: result depends on timing/interleaving. Types: *data race* (unsynchronized conflicting accesses), *check-then-act* (`if (!dict.ContainsKey(k)) dict.Add(k, v)`), *read-modify-write*, *lazy init*.
- **Thread-safe**: correct under concurrent use without external sync. Strategies: immutability, thread confinement, locking, atomics, concurrent collections, message passing (Channels), `Lazy<T>`.
- Non-thread-safe: `List<T>`, `Dictionary<K,V>`, `HashSet<T>`, `StringBuilder`, `Random` (instance; use `Random.Shared` in .NET 6+, which is thread-safe), `DbContext`, `HttpClient` *is* thread-safe for requests.
- Thread-safe individually ≠ compound operations atomic (two thread-safe calls together aren't).
- Symptoms: lost updates, `InvalidOperationException: Collection was modified`, corrupted `Dictionary` (infinite loop on resize in old versions), intermittent bugs.
- Tools: stress tests, `ThreadSanitizer` equivalents are limited in .NET; code review, Concurrency Visualizer, Coyote (Microsoft).

### 15.1 [Easy] Demonstrate a race condition with a shared counter and fix it.
**Answer**
Many threads incrementing a shared int without sync lose updates. Fix with `lock` or `Interlocked`.

**Explanation**
Result is non-deterministic; the bug may only show under load. Choose the lightest correct primitive.

**Code**
```csharp
int count = 0;
var tasks = Enumerable.Range(0, 8).Select(_ => Task.Run(() => { for (int i = 0; i < 100_000; i++) count++; }));
await Task.WhenAll(tasks);
Console.WriteLine(count); // Output: < 800000 (varies)

int fixedCount = 0;
await Task.WhenAll(Enumerable.Range(0, 8).Select(_ => Task.Run(() => { for (int i = 0; i < 100_000; i++) Interlocked.Increment(ref fixedCount); })));
Console.WriteLine(fixedCount); // Output: 800000
```

### 15.2 [Follow-up] Why is `Dictionary` unsafe for concurrent writes and what happens?
**Answer**
Concurrent mutation corrupts internal buckets/entries (lost entries, exceptions, even infinite loops while resizing). Concurrent reads with no writers are safe. Use `ConcurrentDictionary`, lock, or immutable snapshots.

**Explanation**
Even "read while another writes" is unsafe. .NET Core versions detect some concurrent misuse and throw `InvalidOperationException` ("Operations that change non-concurrent collections must have exclusive access").

**Code**
```csharp
var d = new Dictionary<int, int>();
try
{
    Parallel.For(0, 100_000, i => { d[i] = i; });
}
catch (Exception e) { Console.WriteLine(e.GetType().Name); }
Console.WriteLine(d.Count); // Output: often != 100000 or an exception (AggregateException/IndexOutOfRange/InvalidOperation)
```

### 15.3 [Medium] Check-then-act races with thread-safe collections.
**Answer**
`if (!cd.ContainsKey(k)) cd[k] = Create();` is a race even on `ConcurrentDictionary`. Use atomic APIs: `GetOrAdd`, `AddOrUpdate`, `TryAdd`, `TryUpdate`. Note that `GetOrAdd(key, factory)` may call `factory` multiple times and discard results; wrap in `Lazy<T>` for once-only side effects.

**Explanation**
The factory delegates run *outside* the dictionary's locks (to avoid deadlocks), so concurrent callers can each run it; only one result wins.

**Code**
```csharp
var cd = new System.Collections.Concurrent.ConcurrentDictionary<string, Lazy<ExpensiveObj>>();
ExpensiveObj Get(string k) => cd.GetOrAdd(k, key => new Lazy<ExpensiveObj>(() => new ExpensiveObj(key))).Value;

Parallel.For(0, 100, _ => Get("a"));
Console.WriteLine(ExpensiveObj.Created); // Output: 1

class ExpensiveObj
{
    public static int Created;
    public ExpensiveObj(string k) { Interlocked.Increment(ref Created); }
}
```

### 15.4 [Hard] Thread-safety of `Random`, `DateTime.Now`, static fields, singleton services in DI.
**Answer**
`new Random()` instance methods aren't thread-safe (can return 0s when corrupted in older frameworks); use `Random.Shared` (.NET 6+) or one per thread. Static mutable fields and singleton services share state across all requests — must be thread-safe. Scoped `DbContext` must never be used from parallel tasks concurrently.

**Explanation**
In ASP.NET Core, singleton lifetimes mean concurrency by default. Typical bug: caching per-request data in a singleton field. EF Core throws "A second operation was started on this context instance before a previous operation completed" on concurrent DbContext use.

**Code**
```csharp
var results = new System.Collections.Concurrent.ConcurrentBag<int>();
Parallel.For(0, 1000, _ => results.Add(Random.Shared.Next(100))); // thread-safe
Console.WriteLine(results.Count); // Output: 1000

// BUG in DI: singleton holding mutable state
public class Counter { public int N; public void Hit() => N++; }  // race when registered as AddSingleton
```

**Gotchas / follow-ups**
- `HttpClient` is thread-safe; don't mutate `DefaultRequestHeaders` concurrently.
- `async` code on the same DbContext via `Task.WhenAll` → exception. Use separate contexts via `IDbContextFactory<T>`.

### 15.5 [Tricky] Shared lambda-captured variable bug and closure races.
**Answer**
Closures capture variables, not values. Tasks started in a `for` loop capturing the loop variable `i` all see its later value. (`foreach` variables are per-iteration since C# 5; `for` loop variables are not.)

**Explanation**
Fix by copying into a local inside the loop. Another capture issue: modifying a captured variable from multiple tasks is a race.

**Code**
```csharp
var list = new List<Task>();
for (int i = 0; i < 3; i++)
    list.Add(Task.Run(() => Console.Write(i + " ")));
await Task.WhenAll(list);
Console.WriteLine();
// Output (likely): 3 3 3   (tasks run after loop ends; order/values vary)

list.Clear();
for (int i = 0; i < 3; i++) { int copy = i; list.Add(Task.Run(() => Console.Write(copy + " "))); }
await Task.WhenAll(list);
// Output: 0 1 2 (any order)
```

**Interview traps**
- Thread-safe collection ≠ atomic compound operations.
- `GetOrAdd` factory may run multiple times.
- `for` loop closure capture; `foreach` is safe since C# 5.
- Singleton + mutable state = race.

**Cheat sheet**
- Prefer immutability/confinement over locking.
- `Interlocked` for counters, `lock` for compound invariants.
- `ConcurrentDictionary` atomic methods; `Lazy` for once-only.
- `Random.Shared` is thread-safe.
- DbContext is not thread-safe.

---

## Topic 16: Concurrent collections & BlockingCollection

**Study notes**

`System.Collections.Concurrent`:
- `ConcurrentDictionary<K,V>`: lock striping (fine-grained locks per bucket group); lock-free reads. Atomic: `TryAdd`, `TryGetValue`, `TryRemove`, `TryUpdate`, `GetOrAdd`, `AddOrUpdate`. `Count`/`ToArray`/`Keys` lock everything (costly). Factory delegates may run more than once.
- `ConcurrentQueue<T>`: lock-free FIFO (segmented linked arrays). `TryDequeue`, `TryPeek`, `Enqueue`.
- `ConcurrentStack<T>`: lock-free LIFO; `PushRange/TryPopRange`.
- `ConcurrentBag<T>`: unordered; per-thread local lists; efficient when same threads add & take (work-stealing pools). Not for FIFO/LIFO.
- `BlockingCollection<T>`: wraps an `IProducerConsumerCollection<T>` (default queue); bounded capacity → back-pressure; `Add` blocks when full, `Take`/`GetConsumingEnumerable` blocks when empty; `CompleteAdding()` signals no more items. **Blocking** (threads) — for async use Channels.
- Enumeration of concurrent collections = snapshot/moment-in-time (no `Collection modified` exception).
- Immutable collections (`System.Collections.Immutable`) + `ImmutableInterlocked` — another safe pattern; `FrozenDictionary` (.NET 8) for read-only lookups.

### 16.1 [Easy] When do you use ConcurrentDictionary vs Dictionary + lock?
**Answer**
`ConcurrentDictionary` for frequent concurrent reads/writes with simple atomic operations (counters, caches). A plain `Dictionary` + `lock` is fine for low contention or when you need multi-step invariants across several collections.

**Explanation**
ConcurrentDictionary is not a drop-in speedup; `Count` and iteration are costly, and compound ops need its atomic methods.

**Code**
```csharp
var counts = new System.Collections.Concurrent.ConcurrentDictionary<string, int>();
Parallel.ForEach(new[] { "a", "b", "a", "c", "a" }, w => counts.AddOrUpdate(w, 1, (_, old) => old + 1));
Console.WriteLine(counts["a"]); // Output: 3
```

### 16.2 [Follow-up] What are BlockingCollection's key features and limits?
**Answer**
Thread-safe producer/consumer with optional bounded capacity (back-pressure), blocking `Add/Take`, `TryTake` with timeout, cancellation tokens, and `CompleteAdding`. Limit: it blocks threads (not async-friendly) — prefer `Channel<T>` in async apps.

**Explanation**
`GetConsumingEnumerable()` ends after `CompleteAdding` and queue drained. Dispose it when done.

**Code**
```csharp
using var bc = new BlockingCollection<int>(boundedCapacity: 2);
var producer = Task.Run(() =>
{
    for (int i = 1; i <= 5; i++) { bc.Add(i); Console.WriteLine($"produced {i}"); }
    bc.CompleteAdding();
});
var consumer = Task.Run(() =>
{
    foreach (var x in bc.GetConsumingEnumerable())
    { Thread.Sleep(50); Console.WriteLine($"consumed {x}"); }
});
await Task.WhenAll(producer, consumer);
// Output: produced/consumed interleaved; producer is throttled to at most 2 ahead
```

### 16.3 [Medium] ConcurrentQueue vs ConcurrentBag vs ConcurrentStack — choose.
**Answer**
Queue for FIFO pipelines, Stack for LIFO, Bag when order doesn't matter and producers/consumers are often the same threads (thread-local storage reduces contention). Bag can have poor behavior for pure producer/consumer split across threads.

**Explanation**
`BlockingCollection<T>` default backing = ConcurrentQueue. `ConcurrentBag` enumeration order undefined.

**Code**
```csharp
var q = new System.Collections.Concurrent.ConcurrentQueue<int>();
q.Enqueue(1); q.Enqueue(2);
if (q.TryDequeue(out var a)) Console.WriteLine(a); // Output: 1

var s = new System.Collections.Concurrent.ConcurrentStack<int>();
s.Push(1); s.Push(2);
if (s.TryPop(out var b)) Console.WriteLine(b); // Output: 2
```

### 16.4 [Hard] ConcurrentDictionary GetOrAdd/AddOrUpdate pitfalls: factories, `Count`, and enumeration.
**Answer**
Factory/update delegates can run multiple times under contention and run outside locks — they must be side-effect-free (or use `Lazy<T>`). `Count`, `Keys`, `Values`, `ToArray`, enumerating with `IsEmpty` have different cost: `Count` acquires all locks; `IsEmpty` also; enumeration is not an atomic snapshot (reflects state over time but safe).

**Explanation**
Overload `GetOrAdd(key, factoryArg-based factory, arg)` avoids closure allocations (.NET Core 2.0+). Also the `concurrencyLevel` constructor param defaults to processor count.

**Code**
```csharp
var cd = new System.Collections.Concurrent.ConcurrentDictionary<int, int>();
int calls = 0;
Parallel.For(0, 1000, _ => cd.GetOrAdd(1, k => { Interlocked.Increment(ref calls); return k * 10; }));
Console.WriteLine(cd[1]);        // Output: 10
Console.WriteLine(calls >= 1);   // Output: True  (calls may be > 1)
```

**Gotchas / follow-ups**
- `cd[key] += 1` is not atomic; use `AddOrUpdate` or `Interlocked` on a value holder.
- `Count` in hot paths → contention.

### 16.5 [Tricky] BlockingCollection pitfalls: forgetting CompleteAdding, deadlocks, and mixing with async.
**Answer**
Forgetting `CompleteAdding()` makes `GetConsumingEnumerable` block forever. Calling `Take()` inside an async method blocks pool threads (starvation). A bounded collection with a single thread doing both Add and Take deadlocks when full.

**Explanation**
Use `CompleteAdding` in a `finally` in the producer. For async code use `Channel<T>` (Topic 17), or wrap in `Task.Run` with dedicated long-running threads.

**Code**
```csharp
var bc = new BlockingCollection<int>(1);
bc.Add(1);
bool ok = bc.TryAdd(2, TimeSpan.FromMilliseconds(100)); // would block forever with Add(2)
Console.WriteLine(ok); // Output: False
bc.CompleteAdding();
try { bc.Add(3); } catch (InvalidOperationException) { Console.WriteLine("adding completed"); } // Output: adding completed
```

**Interview traps**
- Concurrent collections ≠ thread-safe compound operations.
- `GetOrAdd` factory may execute more than once.
- `Count` on ConcurrentDictionary is expensive.
- BlockingCollection blocks threads; use Channels for async.

**Cheat sheet**
- Dictionary → `ConcurrentDictionary` (atomic methods).
- FIFO → `ConcurrentQueue`; LIFO → `ConcurrentStack`; unordered → `ConcurrentBag`.
- Producer/consumer (sync) → `BlockingCollection` with bounded capacity.
- Always `CompleteAdding()`.
- Read-mostly: `FrozenDictionary`/immutable.

---

## Topic 17: System.Threading.Channels producer/consumer

**Study notes**

- `System.Threading.Channels` (in-box since .NET Core 3.0): async-first, thread-safe producer/consumer queues.
- Create: `Channel.CreateUnbounded<T>(UnboundedChannelOptions)` or `Channel.CreateBounded<T>(BoundedChannelOptions)`.
- Bounded options: `Capacity`, `FullMode` (`Wait` (default), `DropOldest`, `DropNewest`, `DropWrite`), `SingleReader`, `SingleWriter`, `AllowSynchronousContinuations`.
- Writer: `WriteAsync(item, ct)` (waits when full), `TryWrite`, `WaitToWriteAsync`, `Complete(Exception?)`.
- Reader: `ReadAsync(ct)` (throws `ChannelClosedException` when completed & drained), `TryRead`, `WaitToReadAsync` (returns false when completed), `ReadAllAsync(ct)` → `IAsyncEnumerable<T>`.
- Completion: `writer.Complete()`; `reader.Completion` task completes when drained.
- Compared to `BlockingCollection`: no blocked threads; back-pressure via awaiting.
- Used by SignalR, Kestrel internals, background queues in `BackgroundService`.

```
Producer(s) ──WriteAsync──► [ bounded buffer N ] ──ReadAllAsync──► Consumer(s)
        (awaits when full: back-pressure)       (awaits when empty)
```

### 17.1 [Easy] Show a basic producer/consumer with Channels.
**Answer**
Create a channel, have a producer `WriteAsync` items then `Complete()`, and a consumer loop `await foreach` over `reader.ReadAllAsync()`.

**Explanation**
`ReadAllAsync` ends when the writer completes and the buffer is drained.

**Code**
```csharp
using System.Threading.Channels;

var ch = Channel.CreateUnbounded<int>();
var producer = Task.Run(async () =>
{
    for (int i = 1; i <= 3; i++) await ch.Writer.WriteAsync(i);
    ch.Writer.Complete();
});
var consumer = Task.Run(async () =>
{
    await foreach (var x in ch.Reader.ReadAllAsync())
        Console.WriteLine($"got {x}");
});
await Task.WhenAll(producer, consumer);
// Output: got 1 / got 2 / got 3
```

### 17.2 [Follow-up] How do bounded channels provide back-pressure, and what are the FullMode options?
**Answer**
With `Channel.CreateBounded<T>(n)`, `WriteAsync` doesn't complete while the buffer is full (default `BoundedChannelFullMode.Wait`), slowing fast producers. Other modes drop items instead: `DropOldest`, `DropNewest`, `DropWrite` (`TryWrite` returns true while dropping in drop modes; `WriteAsync` completes).

**Explanation**
Unbounded channels can grow memory without limit if consumers lag. Bounded + `Wait` is the safest default for pipelines.

**Code**
```csharp
using System.Threading.Channels;
var ch = Channel.CreateBounded<int>(new BoundedChannelOptions(2) { FullMode = BoundedChannelFullMode.Wait });
Console.WriteLine(ch.Writer.TryWrite(1)); // True
Console.WriteLine(ch.Writer.TryWrite(2)); // True
Console.WriteLine(ch.Writer.TryWrite(3)); // False (full, Wait mode)
// Output: True True False
```

### 17.3 [Medium] Multiple consumers and graceful shutdown.
**Answer**
Start N consumer tasks reading from the same reader (each item goes to exactly one consumer). Producer calls `Complete()` when done; consumers exit when drained. Await `reader.Completion` or all consumer tasks.

**Explanation**
Multiple producers: complete only after *all* producers finish (e.g., `await Task.WhenAll(producers); writer.Complete();`). Pass the exception to `Complete(ex)` to propagate failure to readers.

**Code**
```csharp
using System.Threading.Channels;
var ch = Channel.CreateBounded<int>(10);
var producers = Enumerable.Range(0, 2).Select(p => Task.Run(async () =>
{
    for (int i = 0; i < 5; i++) await ch.Writer.WriteAsync(p * 100 + i);
})).ToArray();
var consumers = Enumerable.Range(0, 3).Select(c => Task.Run(async () =>
{
    int n = 0;
    await foreach (var _ in ch.Reader.ReadAllAsync()) n++;
    return n;
})).ToArray();
await Task.WhenAll(producers);
ch.Writer.Complete();
var counts = await Task.WhenAll(consumers);
Console.WriteLine(counts.Sum()); // Output: 10
```

### 17.4 [Hard] Channel vs BlockingCollection vs ConcurrentQueue; SingleReader/SingleWriter flags.
**Answer**
Channel = async waiting, back-pressure, completion semantics, high performance. `BlockingCollection` blocks threads. `ConcurrentQueue` has no waiting/signalling at all. Setting `SingleReader`/`SingleWriter = true` lets the implementation pick lower-overhead (lock-free) paths, but violating the promise is undefined behavior.

**Explanation**
`AllowSynchronousContinuations = true` makes continuations run inline on the writer thread (faster, risk of re-entrancy/stack dives) — default false.

**Code**
```csharp
using System.Threading.Channels;
var opts = new BoundedChannelOptions(100)
{
    SingleReader = true,
    SingleWriter = false,
    AllowSynchronousContinuations = false,
    FullMode = BoundedChannelFullMode.Wait
};
var ch = Channel.CreateBounded<string>(opts);
```

**Gotchas / follow-ups**
- `ReadAsync` on completed empty channel throws `ChannelClosedException`; `WaitToReadAsync` returns false.
- Writing after `Complete` throws `ChannelClosedException` (`TryWrite` returns false).
- `ReadAllAsync` throws the completion exception if `Complete(ex)` was used.

### 17.5 [Tricky] A background queue in ASP.NET Core: why use Channels with `BackgroundService`?
**Answer**
Request handlers `TryWrite`/`WriteAsync` work items into a bounded channel; a `BackgroundService` drains it. It decouples request latency from work, provides back-pressure, and avoids unobserved fire-and-forget Tasks. Caveat: in-memory — items are lost on shutdown/crash (use a durable queue when needed).

**Explanation**
Scoped services must be resolved via `IServiceScopeFactory` inside the worker. On shutdown, the stopping token cancels `ReadAllAsync`; decide whether to drain.

**Code**
```csharp
using System.Threading.Channels;
using Microsoft.Extensions.Hosting;

public sealed class WorkQueue
{
    private readonly Channel<Func<CancellationToken, ValueTask>> _ch =
        Channel.CreateBounded<Func<CancellationToken, ValueTask>>(100);
    public ValueTask EnqueueAsync(Func<CancellationToken, ValueTask> w) => _ch.Writer.WriteAsync(w);
    public IAsyncEnumerable<Func<CancellationToken, ValueTask>> ReadAllAsync(CancellationToken ct)
        => _ch.Reader.ReadAllAsync(ct);
}

public sealed class Worker(WorkQueue queue) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await foreach (var work in queue.ReadAllAsync(stoppingToken))
        {
            try { await work(stoppingToken); }
            catch (Exception ex) when (ex is not OperationCanceledException) { /* log */ }
        }
    }
}
```

**Interview traps**
- Forgetting `Complete()` → consumers wait forever.
- Multiple producers: complete after all done.
- `DropWrite` etc. silently lose data.
- Violating SingleReader/SingleWriter = UB.

**Cheat sheet**
- Channel = async producer/consumer.
- Bounded + `Wait` = back-pressure.
- `ReadAllAsync` + `Complete()`.
- Each item to exactly one consumer.
- Prefer over BlockingCollection in async code.

---

## Topic 18: Task.Run for CPU vs I/O bound

**Study notes**

- **CPU-bound**: work keeps a core busy (hashing, image processing, parsing). Parallelism helps; use `Task.Run` to offload from a UI/request thread (UI apps), or `Parallel`/PLINQ for data parallel.
- **I/O-bound**: waiting for network/disk/DB. Use truly async APIs (`HttpClient`, EF `ToListAsync`, `Stream.ReadAsync`) — no thread consumed. Wrapping sync I/O in `Task.Run` just moves the blocking to another pool thread ("fake async").
- In **ASP.NET Core**, `Task.Run` for CPU work usually does *not* help (request already on pool thread); it adds scheduling overhead and uses extra pool threads. Offload only to bound CPU contention (separate worker/queue) or for truly parallel pieces.
- In **UI apps**, `await Task.Run(() => HeavyCpu())` keeps UI responsive.
- Library authors should **not** expose async wrappers over sync methods (`Task.Run`) — let callers decide. Likewise no sync-over-async.
- Decision: `Is it waiting (I/O) → async API`; `Is it computing → Task.Run (UI) / Parallel (data-parallel) / just run sync (server)`.

### 18.1 [Easy] How do you decide between async I/O and Task.Run?
**Answer**
If the operation waits on external resources, use the native async API. If it burns CPU and you must keep a UI thread responsive or exploit multiple cores, use `Task.Run`/Parallel.

**Explanation**
Async I/O frees the thread while waiting; Task.Run just occupies a different thread.

**Code**
```csharp
// I/O-bound: no thread consumed while waiting
using var http = new HttpClient();
var body = await http.GetStringAsync("https://example.com");

// CPU-bound: offload (useful in UI apps)
var hash = await Task.Run(() => System.Security.Cryptography.SHA256.HashData(new byte[50_000_000]));
Console.WriteLine(hash.Length); // Output: 32
```

### 18.2 [Follow-up] Why is `await Task.Run(() => File.ReadAllText(path))` considered an anti-pattern?
**Answer**
It still blocks a thread (a pool thread) for the I/O duration; it only hides it from the caller. Use `File.ReadAllTextAsync` (which uses truly async OS I/O when opened asynchronously).

**Explanation**
For file I/O, `FileStream` must be opened with `useAsync: true` / `FileOptions.Asynchronous` to get real async I/O on Windows; `File.ReadAllTextAsync` handles it. In a UI app wrapping might still keep the UI responsive, but the scalable approach is native async.

**Code**
```csharp
var text = await File.ReadAllTextAsync("data.txt");           // preferred
// var text = await Task.Run(() => File.ReadAllText("data.txt")); // anti-pattern on servers
```

### 18.3 [Medium] Should I use Task.Run in an ASP.NET Core controller for CPU work?
**Answer**
Generally no: request code already runs on a pool thread, so `Task.Run` adds a thread hop and consumes another pool thread without adding capacity. Run CPU work inline if short; for heavy work, push to a background queue/worker (Channel + BackgroundService) or dedicated service, and return 202 Accepted.

**Explanation**
Parallelizing one request across cores steals cores from other requests; fine for low-traffic admin endpoints, bad at scale.

**Code**
```csharp
app.MapGet("/hash", () =>
{
    var h = System.Security.Cryptography.SHA256.HashData(new byte[1000]); // short CPU: inline
    return Convert.ToHexString(h);
});
```

### 18.4 [Hard] How do you keep a UI responsive and report progress while doing CPU work?
**Answer**
`await Task.Run(() => Work(progress, ct), ct)` with `IProgress<T>` (`Progress<T>` captures the UI SC and posts reports back) and a CancellationToken. Don't touch UI objects inside the Run delegate.

**Explanation**
`Progress<T>` captures `SynchronizationContext.Current` at construction, so create it on the UI thread. Report events are posted asynchronously (not necessarily ordered relative to completion).

**Code**
```csharp
static int Work(IProgress<int> progress, CancellationToken ct)
{
    for (int i = 1; i <= 100; i++)
    {
        ct.ThrowIfCancellationRequested();
        Thread.SpinWait(1_000_000);
        progress.Report(i);
    }
    return 100;
}
var progress = new Progress<int>(p => Console.WriteLine($"{p}%")); // posts to captured SC (or pool in console)
using var cts = new CancellationTokenSource();
int r = await Task.Run(() => Work(progress, cts.Token), cts.Token);
```

**Gotchas / follow-ups**
- In console apps, `Progress<T>` callbacks run on the pool and may arrive out of order.

### 18.5 [Tricky] Is `Task.Run(() => SomethingAsync())` useful? When does it help/hurt?
**Answer**
It helps when `SomethingAsync` does a lot of *synchronous* CPU work before its first await on a UI thread (moves that prefix off the UI), or to escape a SynchronizationContext. It hurts if used reflexively: extra hop, extra allocations, and no benefit for truly async methods.

**Explanation**
Also an anti-deadlock hack for sync-over-async (Topic 6). In libraries, don't do it on behalf of callers.

**Code**
```csharp
static async Task<int> HeavyPrefixAsync()
{
    long s = 0; for (int i = 0; i < 200_000_000; i++) s += i; // sync CPU prefix
    await Task.Delay(10);
    return (int)(s % 1000);
}
var r = await Task.Run(HeavyPrefixAsync);  // prefix runs on a pool thread, not the caller
Console.WriteLine(r >= 0); // Output: True
```

**Interview traps**
- Task.Run around I/O = fake async.
- Task.Run in ASP.NET Core rarely helps.
- Don't expose async wrappers over sync code in libraries.
- Don't touch UI from the Run delegate.

**Cheat sheet**
- I/O → real async API.
- CPU on UI → `Task.Run`.
- CPU data-parallel → `Parallel`/PLINQ.
- Server heavy CPU → background worker/queue.
- `Progress<T>` for UI progress.

---

## Topic 19: Thread starvation & pool tuning

**Study notes**

- **Starvation**: all pool threads are busy/blocked, queued work waits. Pool injects new threads when work is queued and throughput doesn't improve: below `MinThreads` immediate; above that ~1 thread per 0.5–1 s via hill-climbing (Gate thread).
- Causes: sync-over-async (`.Result/.Wait`), `Thread.Sleep`, blocking I/O, long-running CPU tasks, `lock` contention with long holds, `Parallel.For` on request threads, `async void` flooding.
- Symptoms: latency spikes with **low CPU**, growing request queue, `ThreadPool.PendingWorkItemCount` high, timeouts (HTTP, SQL).
- Diagnosis: `dotnet-counters monitor System.Runtime` → `threadpool-thread-count`, `threadpool-queue-length`, `threadpool-completed-items-count`; `ThreadPool.ThreadCount`, `ThreadPool.PendingWorkItemCount` (.NET Core 3+); `dotnet-dump` → `clrstack -all` / `dumpasync`; Parallel Stacks in VS.
- Mitigation: remove blocking (root fix); `ThreadPool.SetMinThreads` (band-aid; also `System.Threading.ThreadPool.MinThreads` runtimeconfig knob); use `TaskCreationOptions.LongRunning` for long loops; limit concurrency; separate resources (bulkheads).
- `Kestrel` + `HttpClient` + `SqlClient` all tied to pool completion callbacks, so starvation cascades.

### 19.1 [Easy] What is thread-pool starvation?
**Answer**
It's when all pool threads are occupied (usually blocked) so new work items, including the continuations that would unblock them, can't run promptly. Latency rises while CPU stays low.

**Explanation**
The pool is deliberately conservative in adding threads to avoid oversubscription, so bursts of blocking work take seconds to recover.

**Code**
```csharp
ThreadPool.SetMinThreads(2, 2);   // demo only
var blockers = Enumerable.Range(0, 8).Select(_ => Task.Run(() => Thread.Sleep(3000))).ToArray();
var sw = System.Diagnostics.Stopwatch.StartNew();
await Task.Run(() => Console.WriteLine($"ran after {sw.ElapsedMilliseconds} ms"));
// Output: ran after ~3000+ ms (queued behind blocked threads, new threads injected slowly)
```

### 19.2 [Follow-up] How do you diagnose it in production?
**Answer**
Watch `dotnet-counters` for rising `threadpool-queue-length` and `threadpool-thread-count` with low CPU; capture a dump/`dotnet-stack` to see many threads blocked in `Task.Wait`/`.Result`/`Monitor.Enter`/sync I/O; use App Insights/EventPipe thread-pool events.

**Explanation**
The common signature is dozens of threads whose stack ends in `Task.Wait`, `GetResult`, or `SqlClient.Execute...` (sync). Fix call sites to async.

**Code**
```csharp
Console.WriteLine($"Threads={ThreadPool.ThreadCount}, Pending={ThreadPool.PendingWorkItemCount}, Completed={ThreadPool.CompletedWorkItemCount}");
// Output: e.g., Threads=8, Pending=0, Completed=123 (values vary)
```

### 19.3 [Medium] SetMinThreads: when does it help, and what are its risks?
**Answer**
It lets the pool create threads immediately up to the minimum, which helps if you have unavoidable bursts of blocking. It masks the underlying issue, increases memory/context-switching, and may just shift the bottleneck (DB connections, downstream).

**Explanation**
Prefer fixing blocking. If you must, set via `runtimeconfig.json` (`System.Threading.ThreadPool.MinThreads`) or at startup. Measure before/after.

**Code**
```csharp
ThreadPool.GetMinThreads(out var minW, out var minIO);
if (!ThreadPool.SetMinThreads(Math.Max(minW, 100), minIO))
    Console.WriteLine("SetMinThreads failed (exceeds max)");
```

### 19.4 [Hard] Explain how sync-over-async cascades into a "death spiral".
**Answer**
Blocked threads wait on async operations whose continuations need pool threads; as more requests arrive, more threads block, queue grows, timeouts occur, clients retry (more load), and the pool's slow injection can't keep up — throughput collapses even with idle CPU.

**Explanation**
Each blocking call holds a thread for the duration of the *whole* async operation; with N cores and slow injection, concurrency capacity ≈ N + trickle. Retry storms amplify. Break the loop with async all the way, timeouts, load shedding/circuit breakers, and bulkhead limits.

**Code**
```csharp
// Anti-pattern
app.MapGet("/users/{id}", (int id, IUserRepo repo) => repo.GetAsync(id).Result);
// Fix
app.MapGet("/users/{id}", async (int id, IUserRepo repo, CancellationToken ct) => await repo.GetAsync(id, ct));

public interface IUserRepo { Task<string> GetAsync(int id, CancellationToken ct = default); }
```

**Gotchas / follow-ups**
- `GetAwaiter().GetResult()` is equally blocking.
- Blocking inside `lock` + async continuation requiring that lock → starvation + deadlock combos.

### 19.5 [Tricky] When to use `LongRunning`, and why can `Task.Run` loops starve the pool?
**Answer**
`Task.Factory.StartNew(..., TaskCreationOptions.LongRunning)` gets a dedicated thread — appropriate for loops that run for the app's lifetime or block for long periods. Using `Task.Run` for such loops permanently consumes a pool worker, shrinking capacity.

**Explanation**
In .NET `BackgroundService.ExecuteAsync` is async and awaited so it doesn't pin a thread unless you block in it. If you have a blocking loop (e.g., sync queue `Take()`), use LongRunning or a dedicated `Thread`.

**Code**
```csharp
var cts = new CancellationTokenSource();
var loop = Task.Factory.StartNew(() =>
{
    while (!cts.IsCancellationRequested) { Thread.Sleep(100); /* blocking work */ }
}, cts.Token, TaskCreationOptions.LongRunning, TaskScheduler.Default);
await Task.Delay(300);
cts.Cancel();
await loop;
Console.WriteLine(loop.Status); // Output: RanToCompletion
```

**Interview traps**
- Low CPU + high latency → suspect starvation.
- Raising MinThreads is a band-aid.
- `.GetAwaiter().GetResult()` blocks too.
- `Task.Run` infinite loops pin pool threads.

**Cheat sheet**
- Pool injection slow above min threads.
- Diagnose: counters, dumps (`Task.Wait` stacks).
- Fix: async all the way.
- `LongRunning` for dedicated blocking loops.
- Add timeouts/bulkheads/load shedding.

---

## Topic 20: Async locks & why `lock` can't await

**Study notes**

- `lock` (Monitor) is **thread-affine**: the thread that acquired must release. After an `await`, the continuation may run on a different thread, so holding a Monitor across await would break ownership. The compiler therefore forbids `await` inside a `lock` block (CS1996).
- Also, `lock` blocks a thread while waiting; async code should wait asynchronously.
- Solution: `SemaphoreSlim(1, 1)` with `await sem.WaitAsync()` and `Release()` in `finally`. Not thread-affine, not reentrant (re-entering from the same flow deadlocks).
- `Mutex`/`ReaderWriterLockSlim` are also thread-affine, can't span await.
- Alternatives: `Nito.AsyncEx.AsyncLock`, `System.Threading.Channels` serialization (single reader), `SemaphoreSlim` for throttling N>1.
- .NET 8 has no built-in AsyncLock; roll your own with SemaphoreSlim returning an `IDisposable` releaser.
- Keep critical sections small; avoid calling external code under the lock.

### 20.1 [Easy] Why can't you `await` inside a `lock` statement?
**Answer**
`lock` uses `Monitor`, which must be released by the same thread that acquired it, but after an `await` the code may resume on another thread. The compiler blocks it with error CS1996. Use `SemaphoreSlim.WaitAsync` instead.

**Explanation**
Even if it could work, holding a thread-blocking lock across a long async wait would hurt scalability and risk deadlocks.

**Code**
```csharp
// Does not compile:
// lock (gate) { await Task.Delay(10); }   // error CS1996

var sem = new SemaphoreSlim(1, 1);
await sem.WaitAsync();
try { await Task.Delay(10); Console.WriteLine("critical section"); }
finally { sem.Release(); }
// Output: critical section
```

### 20.2 [Follow-up] Write a reusable AsyncLock with `using` syntax.
**Answer**
Wrap a `SemaphoreSlim(1,1)`; `LockAsync()` awaits `WaitAsync` and returns an `IDisposable` that calls `Release`.

**Explanation**
Make the releaser idempotent-ish and a struct to avoid allocation if desired. Don't forget cancellation support.

**Code**
```csharp
public sealed class AsyncLock
{
    private readonly SemaphoreSlim _sem = new(1, 1);

    public async Task<Releaser> LockAsync(CancellationToken ct = default)
    {
        await _sem.WaitAsync(ct).ConfigureAwait(false);
        return new Releaser(_sem);
    }

    public readonly struct Releaser(SemaphoreSlim sem) : IDisposable
    {
        public void Dispose() => sem.Release();
    }
}
// usage
var al = new AsyncLock();
using (await al.LockAsync())
{
    await Task.Delay(10);
}
```

### 20.3 [Medium] SemaphoreSlim as throttle: limit concurrent HTTP calls.
**Answer**
`new SemaphoreSlim(N)`; each worker `await WaitAsync()` before the call and `Release()` in `finally`. This bounds concurrency while using `WhenAll`.

**Explanation**
Common bug: calling `Release()` when `WaitAsync` threw (cancellation) → `SemaphoreFullException` or count inflation; acquire *outside* the `try` and release only after acquired.

**Code**
```csharp
var throttle = new SemaphoreSlim(3);
int running = 0, peak = 0;
async Task Call(int i)
{
    await throttle.WaitAsync();
    try
    {
        var now = Interlocked.Increment(ref running);
        int p; while ((p = Volatile.Read(ref peak)) < now && Interlocked.CompareExchange(ref peak, now, p) != p) { }
        await Task.Delay(100);
        Interlocked.Decrement(ref running);
    }
    finally { throttle.Release(); }
}
await Task.WhenAll(Enumerable.Range(0, 10).Select(Call));
Console.WriteLine($"peak={peak}"); // Output: peak=3
```

### 20.4 [Hard] Why is SemaphoreSlim non-reentrant, and what deadlock does that cause?
**Answer**
It has no owner concept; a flow that already holds the single slot and calls another method that tries `WaitAsync` on the same semaphore waits forever (self-deadlock). Design public methods to take the lock and private "unlocked" variants to call internally.

**Explanation**
This differs from `lock`/Monitor which is reentrant. Structure code in layers: locked entry points → lock-free internals.

**Code**
```csharp
class Repo
{
    private readonly SemaphoreSlim _s = new(1, 1);
    public async Task OuterAsync()
    {
        await _s.WaitAsync();
        try { await InnerAsync(); }   // DEADLOCK: InnerAsync waits on same semaphore
        finally { _s.Release(); }
    }
    public async Task InnerAsync()
    {
        await _s.WaitAsync();
        try { await Task.Delay(1); } finally { _s.Release(); }
    }
    // Fix: private InnerCoreAsync() with no locking, called by both public methods.
}
```

**Gotchas / follow-ups**
- Fairness: SemaphoreSlim's async waiters are served roughly FIFO but not guaranteed.
- Dispose the semaphore only when no waiters remain.

### 20.5 [Tricky] Is it OK to use `lock` in async methods around synchronous sections? What about the thread-affinity of `Monitor` across `Task.Yield`?
**Answer**
Yes, if the `lock` body is purely synchronous (no await inside) — it's acquired and released on the same thread. The problem is only awaiting inside. Don't call async code and block on it under the lock.

**Explanation**
Also keep lock bodies short in async servers, since blocking a pool thread on contention contributes to starvation. If the guarded state is small, `Interlocked`/`ConcurrentDictionary` might avoid locks altogether.

**Code**
```csharp
class Cache
{
    private readonly object _gate = new();
    private readonly Dictionary<int, string> _d = new();
    public async Task<string> GetAsync(int k)
    {
        lock (_gate) { if (_d.TryGetValue(k, out var v)) return v; }  // sync-only critical section
        var val = await LoadAsync(k);                                   // await outside lock
        lock (_gate) { _d[k] = val; }
        return val;
    }
    private static async Task<string> LoadAsync(int k) { await Task.Delay(10); return $"v{k}"; }
}
```

**Gotchas / follow-ups**
- This allows duplicate loads (two threads miss simultaneously) — use `ConcurrentDictionary<K, Lazy<Task<V>>>` or per-key semaphore for once-only async init.

**Interview traps**
- CS1996: await inside lock.
- Don't `Release()` a semaphore you didn't acquire.
- SemaphoreSlim isn't reentrant.
- `Mutex`/RWLS across await break thread affinity.

**Cheat sheet**
- `lock` = sync-only.
- Async mutual exclusion = `SemaphoreSlim(1,1)` + try/finally.
- Throttle N = `SemaphoreSlim(N)`.
- No reentrancy → layer locked/unlocked methods.
- Per-key once-only async: `Lazy<Task<T>>`.

---

## Topic 21: TaskCompletionSource

**Study notes**

- `TaskCompletionSource<T>` (TCS): creates a "promise" Task you complete manually: `SetResult`, `SetException`, `SetCanceled` (and `TrySet*` variants that return false if already completed). Non-generic `TaskCompletionSource` exists in .NET 5+.
- Used to **bridge** callback/event/APM-based APIs (and legacy threads) into `await`able Tasks, and to implement signals/async events.
- The Task has no thread — Status stays `WaitingForActivation` until completed.
- **Always** pass `TaskCreationOptions.RunContinuationsAsynchronously` unless you need inline continuations; otherwise `SetResult` may synchronously run awaiting code on the caller's thread (under your locks, deep stacks, deadlock potential).
- `SetResult` throws `InvalidOperationException` if already completed → use `TrySetResult` when racing with cancellation/timeouts.
- Tie to cancellation with `ct.Register(() => tcs.TrySetCanceled(ct))`.

### 21.1 [Easy] What is TaskCompletionSource used for?
**Answer**
To create a Task whose completion you control manually — typically to wrap event/callback-based APIs so they can be awaited.

**Explanation**
`tcs.Task` is handed to consumers; the producer later calls `SetResult/SetException/SetCanceled`.

**Code**
```csharp
static Task<string> WaitForEventAsync(EventSource src)
{
    var tcs = new TaskCompletionSource<string>(TaskCreationOptions.RunContinuationsAsynchronously);
    void Handler(string s) { src.Fired -= Handler; tcs.TrySetResult(s); }
    src.Fired += Handler;
    return tcs.Task;
}
var es = new EventSource();
var t = WaitForEventAsync(es);
es.Raise("hello");
Console.WriteLine(await t); // Output: hello

class EventSource
{
    public event Action<string>? Fired;
    public void Raise(string s) => Fired?.Invoke(s);
}
```

### 21.2 [Follow-up] Why use `RunContinuationsAsynchronously`?
**Answer**
Without it, continuations of `await tcs.Task` may run synchronously inside `SetResult` on the completing thread. That means your `SetResult` call can unexpectedly execute arbitrary user code (long-running, blocking, re-entrant, or deadlock-prone if you hold a lock).

**Explanation**
With the flag, `SetResult` returns quickly; continuations are queued to the scheduler/pool.

**Code**
```csharp
var tcs = new TaskCompletionSource<int>();   // default: inline allowed
var waiter = Task.Run(async () => { await tcs.Task; Thread.Sleep(2000); });
await Task.Delay(100);
var sw = System.Diagnostics.Stopwatch.StartNew();
tcs.SetResult(1);                            // may take ~2 s: runs waiter's continuation inline!
Console.WriteLine($"SetResult took {sw.ElapsedMilliseconds}ms");
// Output (typical w/o flag): ~2000ms; with RunContinuationsAsynchronously: ~0ms
```

### 21.3 [Medium] Implement an async request/response correlation (e.g., over a socket or message bus).
**Answer**
Keep a `ConcurrentDictionary<Guid, TaskCompletionSource<Response>>`; sending registers a TCS under a correlation id; the receive loop looks up and completes it. Add timeout/cancel with `WaitAsync` or `ct.Register`, and remove the entry on completion.

**Explanation**
Use `TryRemove` before `TrySetResult` to avoid leaks. Also fail all pending TCS if the connection drops (`TrySetException`).

**Code**
```csharp
using System.Collections.Concurrent;

sealed class Rpc
{
    private readonly ConcurrentDictionary<Guid, TaskCompletionSource<string>> _pending = new();

    public async Task<string> CallAsync(string req, TimeSpan timeout)
    {
        var id = Guid.NewGuid();
        var tcs = new TaskCompletionSource<string>(TaskCreationOptions.RunContinuationsAsynchronously);
        _pending[id] = tcs;
        try
        {
            Send(id, req);                         // fire to transport
            return await tcs.Task.WaitAsync(timeout);
        }
        finally { _pending.TryRemove(id, out _); }
    }

    public void OnMessage(Guid id, string resp)
    {
        if (_pending.TryRemove(id, out var tcs)) tcs.TrySetResult(resp);
    }

    private void Send(Guid id, string req) => Task.Run(() => OnMessage(id, "echo:" + req)); // fake transport
}
// var rpc = new Rpc(); Console.WriteLine(await rpc.CallAsync("ping", TimeSpan.FromSeconds(1))); // Output: echo:ping
```

### 21.4 [Hard] How does TCS compare with `Task.Run`/`new Task`, and what states can it reach?
**Answer**
`Task.Run` creates *delegate* Tasks executed by a scheduler. TCS creates *promise* Tasks without any delegate; completion comes from external code. States: `WaitingForActivation` → `RanToCompletion` / `Faulted` / `Canceled`. A promise Task can't be `Start()`ed or `RunSynchronously()`.

**Explanation**
`async` methods also produce promise Tasks through the builder (the builder is effectively a TCS). `SetException` with multiple exceptions via `SetException(IEnumerable<Exception>)` produces an `AggregateException` with all.

**Code**
```csharp
var tcs = new TaskCompletionSource<int>();
Console.WriteLine(tcs.Task.Status);        // WaitingForActivation
tcs.SetException(new InvalidOperationException("x"));
Console.WriteLine(tcs.Task.Status);        // Faulted
Console.WriteLine(tcs.TrySetResult(1));    // False (already completed)
// tcs.SetResult(1);                       // would throw InvalidOperationException
try { await tcs.Task; } catch (InvalidOperationException e) { Console.WriteLine(e.Message); } // x
```

**Gotchas / follow-ups**
- `SetResult` race with cancel → use `TrySet*`.
- Faulted TCS with never-observed exception → `UnobservedTaskException`.

### 21.5 [Tricky] A TCS that's never completed — what's the impact? How do you avoid hung awaiters?
**Answer**
Awaiters wait forever (leak: state machine, closures, locals stay reachable via the pending Task). Avoid by always completing in all paths (`try/finally`, connection-closed handlers), adding timeouts (`WaitAsync`), and registering cancellation.

**Explanation**
Unlike a thread, an uncompleted Task consumes no CPU, so hung awaiters are silent — a favorite production mystery. Use diagnostics (`dotnet-dump dumpasync`) to find them.

**Code**
```csharp
var tcs = new TaskCompletionSource<int>(TaskCreationOptions.RunContinuationsAsynchronously);
using var cts = new CancellationTokenSource(200);
using var reg = cts.Token.Register(() => tcs.TrySetCanceled(cts.Token));
try { await tcs.Task; }
catch (OperationCanceledException) { Console.WriteLine("avoided hang"); } // Output: avoided hang
```

**Interview traps**
- Default TCS runs continuations inline → surprises.
- `SetResult` twice throws; use `TrySetResult`.
- Never-completed TCS = silent hang.
- Remember to deregister handlers/events after completing.

**Cheat sheet**
- TCS = manual promise Task.
- Use `RunContinuationsAsynchronously`.
- `TrySet*` for races.
- Bridge events/callbacks → await.
- Timeouts + cancellation registration.

---

## Topic 22: AsyncLocal / ThreadLocal

**Study notes**

- `ThreadLocal<T>` / `[ThreadStatic]`: value per **OS thread**. After `await`, you may be on another thread → value lost/different. Also pool threads are reused → stale data leaks between unrelated work.
- `AsyncLocal<T>`: value flows with the **ExecutionContext** across `await`, `Task.Run`, thread hops — logical call context. Child flows get a *copy*: changes in the child are not visible to the parent (copy-on-write), but changes made before the fork are seen by children.
- `AsyncLocal<T>(Action<AsyncLocalValueChangedArgs<T>>)` notification on changes.
- Suppress flow: `ExecutionContext.SuppressFlow()`; `UnsafeQueueUserWorkItem`/`UnsafeRegister` skip capture.
- Used by: `Activity.Current` (tracing), `IHttpContextAccessor` (`HttpContext` stored in AsyncLocal), logging scopes, ambient transaction (`TransactionScope` with `TransactionScopeAsyncFlowOption.Enabled`), culture flow.
- Mutations made inside an `async` method *do not propagate back to the caller* after the method returns (each async method that sets the value gets its own copy since the state machine's ExecutionContext is restored).

```
Main: AL.Value = "A"
  ├─ Task.Run → sees "A"; sets "B" (visible only inside this flow)
  └─ Main still sees "A"
```

### 22.1 [Easy] ThreadLocal vs AsyncLocal — what's the difference?
**Answer**
`ThreadLocal` stores a value per thread; `AsyncLocal` stores a value per logical async flow that follows `await` across threads. In async code use `AsyncLocal`.

**Explanation**
After `await`, `ThreadLocal`/`[ThreadStatic]` values can differ because the continuation may run on another thread.

**Code**
```csharp
var tl = new ThreadLocal<int>(() => 0);
var al = new AsyncLocal<int>();
tl.Value = 1; al.Value = 1;
await Task.Delay(10).ConfigureAwait(false);   // may switch threads
Console.WriteLine($"ThreadLocal={tl.Value}, AsyncLocal={al.Value}");
// Output: ThreadLocal=0 (if thread changed) , AsyncLocal=1
```

### 22.2 [Follow-up] Do changes inside a child task flow back to the parent?
**Answer**
No. Values flow downward (copied) from parent to child; changes in the child don't affect the parent. Likewise, an async method's changes to an AsyncLocal aren't visible to its caller after it returns.

**Explanation**
The ExecutionContext is immutable; setting a value creates a new context for the current flow. When an async method returns to its caller, the caller's ExecutionContext is restored.

**Code**
```csharp
var al = new AsyncLocal<string>();
al.Value = "parent";
await Task.Run(() => { Console.WriteLine(al.Value); al.Value = "child"; }); // Output: parent
Console.WriteLine(al.Value);                                                 // Output: parent

static async Task SetInside(AsyncLocal<string> a) { a.Value = "inner"; await Task.Yield(); }
await SetInside(al);
Console.WriteLine(al.Value);                                                 // Output: parent
```

### 22.3 [Medium] How does ambient context work in ASP.NET Core (HttpContextAccessor, logging scopes, Activity)?
**Answer**
`IHttpContextAccessor.HttpContext` reads from an `AsyncLocal<HttpContextHolder>`; the middleware sets it per request so it flows through awaits. `Activity.Current` is also AsyncLocal-based. It works as long as the flow isn't cut (e.g., `SuppressFlow`, custom threads without capture) and the request hasn't finished.

**Explanation**
Using `HttpContext` after the request completes (fire-and-forget) gives null/disposed objects. The holder indirection lets callees *clear* it when the request ends.

**Code**
```csharp
// Minimal API
app.MapGet("/who", (IHttpContextAccessor acc) => acc.HttpContext?.TraceIdentifier);
// registration: builder.Services.AddHttpContextAccessor();
```

### 22.4 [Hard] ThreadLocal + thread pool: the stale data bug and how to avoid it.
**Answer**
Pool threads are reused across requests; `[ThreadStatic]`/`ThreadLocal` values set by one work item can leak into the next work item on the same thread unless explicitly reset. In async code the value may also "disappear" after an await. Prefer AsyncLocal with explicit scoping (set and restore in `try/finally` or via `IDisposable` scope).

**Explanation**
Also `[ThreadStatic]` initializers run only for the first thread (field initializer runs once!) — other threads see default.

**Code**
```csharp
class Ambient
{
    private static readonly AsyncLocal<string?> _cur = new();
    public static string? Current => _cur.Value;
    public static IDisposable Begin(string v)
    {
        var prev = _cur.Value; _cur.Value = v;
        return new Scope(() => _cur.Value = prev);
    }
    private sealed class Scope(Action restore) : IDisposable { public void Dispose() => restore(); }
}
using (Ambient.Begin("req-1"))
{
    await Task.Delay(10);
    Console.WriteLine(Ambient.Current); // Output: req-1
}
Console.WriteLine(Ambient.Current ?? "null"); // Output: null
```

**Gotchas / follow-ups**
- `[ThreadStatic] static int x = 5;` → only the thread that ran the static initializer sees 5.
- `ThreadLocal<T>` needs `Dispose` (holds per-thread slots) — `ThreadLocal.Values` with `trackAllValues: true`.

### 22.5 [Tricky] Why might AsyncLocal value "leak" into unrelated work or disappear?
**Answer**
Leaks: values set in a flow are captured by anything started from it (timers, `Task.Run`, long-lived background loops started inside a request) — they keep the old value forever. Disappear: `ExecutionContext.SuppressFlow()`, work scheduled via `UnsafeQueueUserWorkItem`, or setting the value inside a callee and expecting it in the caller.

**Explanation**
Start long-lived background tasks from a clean context (`using (ExecutionContext.SuppressFlow()) { ... }`) so they don't hold request state (and memory).

**Code**
```csharp
var al = new AsyncLocal<string>();
al.Value = "request-data";
Task t;
using (ExecutionContext.SuppressFlow())
{
    t = Task.Run(() => Console.WriteLine(al.Value ?? "null"));
}
await t; // Output: null
```

**Interview traps**
- ThreadLocal after await = unreliable.
- Child changes don't flow back to parent.
- Background work started in a request inherits request AsyncLocals.
- `[ThreadStatic]` initializer runs once only.

**Cheat sheet**
- Thread-bound → ThreadLocal; flow-bound → AsyncLocal.
- Copy-on-write; flows down, not up.
- Scope with set/restore `IDisposable`.
- `SuppressFlow` to detach.
- Powers `Activity.Current`, `HttpContext`.

---

## Topic 23: Timers

**Study notes**

| Timer | Namespace | Threading | Notes |
|---|---|---|---|
| `System.Threading.Timer` | Threading | Callback on pool thread | Lightweight; can overlap if callback slower than period; must keep reference (GC) |
| `System.Timers.Timer` | Timers | `Elapsed` on pool (or SynchronizingObject) | Wrapper over Threading.Timer; exceptions in the handler should be caught inside the handler |
| `PeriodicTimer` (.NET 6+) | Threading | `await timer.WaitForNextTickAsync(ct)` | **Async, no overlap**, no callback; ideal for background loops |
| `DispatcherTimer`/WinForms Timer | UI | UI thread | UI updates |
| `Task.Delay` loop | — | — | Drift accumulates |

- `Timer` callbacks are `TimerCallback(object? state)` — a sync delegate; async lambdas become `async void`.
- Dispose timers; `Timer.DisposeAsync` waits for in-flight callbacks (`Dispose(WaitHandle)` older).
- .NET 8: `TimeProvider` abstraction (`TimeProvider.System.CreateTimer(...)`, `GetTimestamp`) + `FakeTimeProvider` (Microsoft.Extensions.TimeProvider.Testing) for testing.

### 23.1 [Easy] How do you run something periodically in modern .NET?
**Answer**
Use `PeriodicTimer` in an async loop: `while (await timer.WaitForNextTickAsync(ct)) { await DoWorkAsync(); }`. It's async-friendly, avoids overlapping ticks, and works well in `BackgroundService`.

**Explanation**
Ticks missed while the body runs coalesce (at most one pending tick) — no overlap, no pile-up.

**Code**
```csharp
using var timer = new PeriodicTimer(TimeSpan.FromMilliseconds(200));
using var cts = new CancellationTokenSource(700);
try
{
    while (await timer.WaitForNextTickAsync(cts.Token))
        Console.WriteLine($"tick {DateTime.UtcNow:HH:mm:ss.fff}");
}
catch (OperationCanceledException) { Console.WriteLine("stopped"); }
// Output: 3 ticks, then stopped
```

### 23.2 [Follow-up] What problems does `System.Threading.Timer` have?
**Answer**
Callbacks run on pool threads and can overlap if they take longer than the period; exceptions in callback crash the process; async lambdas become `async void`; and you must hold a reference or the GC can collect it and stop the timer.

**Explanation**
Fix overlap by using `Timeout.Infinite` period and re-arming via `Change` at the end of each callback, or use `PeriodicTimer`.

**Code**
```csharp
Timer? timer = null;
int n = 0;
timer = new Timer(_ =>
{
    try
    {
        Console.WriteLine($"tick {++n}");
        Thread.Sleep(300);   // slower than period; overlap prevented by one-shot re-arm
    }
    finally { if (n < 3) timer!.Change(100, Timeout.Infinite); }
}, null, 100, Timeout.Infinite);
await Task.Delay(1500);
await timer.DisposeAsync();
// Output: tick 1, tick 2, tick 3
```

### 23.3 [Medium] System.Timers.Timer vs Threading.Timer — differences and `AutoReset`.
**Answer**
`System.Timers.Timer` is a component-style wrapper using an `Elapsed` event, `AutoReset` (true = repeating), `Start/Stop`, and optional `SynchronizingObject` (UI marshalling). Like `Threading.Timer`, handlers can overlap on the pool.

**Explanation**
Set `AutoReset=false` and call `Start()` in handler end for non-overlap. Dispose to stop the underlying timer.

**Code**
```csharp
var t = new System.Timers.Timer(100) { AutoReset = true };
int c = 0;
t.Elapsed += (_, e) => { if (Interlocked.Increment(ref c) == 3) t.Stop(); };
t.Start();
await Task.Delay(500);
Console.WriteLine(c); // Output: 3
t.Dispose();
```

### 23.4 [Hard] Why does a `Task.Delay` loop drift, and how do you get accurate periods / testable time?
**Answer**
Each iteration's delay starts after the work finishes, so period = work + delay, plus scheduler latency; error accumulates. `PeriodicTimer` schedules from a fixed cadence. For testability, inject `TimeProvider` (.NET 8): `TimeProvider.CreateTimer`, `GetUtcNow`, `Task.Delay(delay, timeProvider, ct)`.

**Explanation**
Timer resolution on Windows is ~15.6 ms by default (can be 1 ms with `timeBeginPeriod`; .NET timers ≈ 1 ms-ish with high-res on newer OS) — don't expect sub-ms accuracy.

**Code**
```csharp
static async Task DriftyAsync(CancellationToken ct)
{
    while (!ct.IsCancellationRequested)
    {
        await DoWork();                       // takes 50ms
        await Task.Delay(1000, ct);           // period ~1050ms, drifts
    }
}
static Task DoWork() => Task.Delay(50);

// Testable: Task.Delay with TimeProvider (.NET 8)
await Task.Delay(TimeSpan.FromMilliseconds(10), TimeProvider.System);
```

**Gotchas / follow-ups**
- Use `Stopwatch.GetTimestamp()`/`TimeProvider.GetTimestamp()` for elapsed measurement, not `DateTime.Now` (clock changes).
- `FakeTimeProvider.Advance(...)` drives tests deterministically.

### 23.5 [Tricky] My Timer stopped firing after a while — why?
**Answer**
The Timer object was garbage-collected because only the timer's own callback referenced it (local variable out of scope). Keep it in a field (or `GC.KeepAlive`) and dispose on shutdown. Another cause: an unhandled exception killed the process, or `Change` re-arm was skipped due to exception without `finally`.

**Explanation**
In .NET, a `Threading.Timer` whose references are dropped can be finalized, which cancels it. `PeriodicTimer` also must remain referenced; the `using` scope keeps it alive in a loop.

**Code**
```csharp
class Service : IAsyncDisposable
{
    private readonly Timer _timer;                  // field keeps it alive
    public Service() => _timer = new Timer(_ => Console.WriteLine("tick"), null, 0, 1000);
    public ValueTask DisposeAsync() => _timer.DisposeAsync();
}
```

**Interview traps**
- Timer callbacks may overlap.
- Async lambda in Timer = `async void`.
- Dropped reference → GC stops timer.
- Task.Delay loops drift; PeriodicTimer doesn't (cadence-based).

**Cheat sheet**
- Async loops → `PeriodicTimer`.
- `Threading.Timer`: pool, overlap, needs reference.
- `Timers.Timer`: Elapsed event, AutoReset.
- Inject `TimeProvider` for tests (.NET 8).
- Dispose timers.

---

## Topic 24: Fire-and-forget pitfalls

**Study notes**

- "Fire-and-forget" = start an async operation and not await it. Problems:
  1. **Lost exceptions** (silently unobserved; `async void` crashes).
  2. **Lost work on shutdown**: process/host exits before it finishes; ASP.NET Core graceful shutdown doesn't track untracked tasks.
  3. **Scoped service disposal**: `DbContext`/scoped services disposed when the request ends → `ObjectDisposedException`. `HttpContext` used after the request → undefined.
  4. **Unbounded concurrency/memory**: no back-pressure.
  5. **Context capture**: inherits AsyncLocal (request data), holds memory.
  6. No retry, no observability.
- Better patterns: (a) background queue (`Channel<T>` + `BackgroundService`), (b) job systems (Hangfire/Quartz/Azure Service Bus/Storage Queues/outbox), (c) `IHostedService`/`IHostApplicationLifetime`, (d) if you truly must: a safe helper that catches/logs and uses a new DI scope.
- Discard `_ = DoAsync();` only silences the CS4014 warning, doesn't fix it.

### 24.1 [Easy] What's wrong with calling an async method without awaiting it in a controller?
**Answer**
Exceptions are lost, the request may finish before the work (and its scoped services such as `DbContext` are disposed), and nothing tracks completion on shutdown.

**Explanation**
Compiler warning CS4014 flags it inside async methods. Using `_ =` discards but doesn't solve those issues.

**Code**
```csharp
// BAD
app.MapPost("/order", (IServiceProvider sp, OrderDto dto) =>
{
    _ = SendEmailAsync(dto);    // fire-and-forget: errors lost, shutdown loses work
    return Results.Accepted();
});
static Task SendEmailAsync(OrderDto dto) => Task.Delay(100);
record OrderDto(int Id);
```

### 24.2 [Follow-up] If you must fire-and-forget, how to do it safely?
**Answer**
Wrap in a helper that catches and logs exceptions, creates its own DI scope, doesn't use request-bound objects, and is tracked (queue) so shutdown can drain. Better: enqueue to a Channel/BackgroundService or durable queue.

**Explanation**
A `SafeFireAndForget` still can't survive process crash; for must-not-lose work use outbox/queues.

**Code**
```csharp
static class TaskExtensions
{
    public static async void Forget(this Task task, Action<Exception> onError)
    {
        try { await task.ConfigureAwait(false); }
        catch (Exception ex) { onError(ex); }
    }
}
// usage:
// SendEmailAsync(dto).Forget(ex => Console.WriteLine($"email failed: {ex.Message}"));
```

### 24.3 [Medium] The DbContext-disposed bug in fire-and-forget.
**Answer**
A scoped `DbContext` is disposed at request end; a background continuation using it later throws `ObjectDisposedException`. Fix: create a new scope with `IServiceScopeFactory` (or `IDbContextFactory<T>`) inside the background work.

**Explanation**
Never capture request-scoped services into background tasks. Pass IDs/DTOs rather than entities tracked by a context.

**Code**
```csharp
using Microsoft.Extensions.DependencyInjection;

public sealed class Background(IServiceScopeFactory scopes)
{
    public async Task RunAsync(int orderId, CancellationToken ct)
    {
        using var scope = scopes.CreateScope();
        var svc = scope.ServiceProvider.GetRequiredService<IOrderService>();
        await svc.ProcessAsync(orderId, ct);
    }
}
public interface IOrderService { Task ProcessAsync(int id, CancellationToken ct); }
```

### 24.4 [Hard] How does graceful shutdown interact with un-awaited tasks in ASP.NET Core / Generic Host?
**Answer**
On SIGTERM/Ctrl+C the host stops accepting requests, runs `StopAsync` for hosted services within `ShutdownTimeout` (default 30 s in .NET 6+; 5 s earlier), then exits. Un-tracked fire-and-forget tasks are not awaited and get killed (pool threads are background).

**Explanation**
Hence implement work as `BackgroundService`s (or track tasks in a registry awaited in `StopAsync`) and honor `stoppingToken`. Configure `HostOptions.ShutdownTimeout`.

**Code**
```csharp
public sealed class TrackedRunner : IHostedService
{
    private readonly List<Task> _running = new();
    private readonly object _gate = new();

    public void Run(Func<Task> work)
    {
        var t = Task.Run(work);
        lock (_gate) _running.Add(t);
    }

    public Task StartAsync(CancellationToken ct) => Task.CompletedTask;

    public async Task StopAsync(CancellationToken ct)
    {
        Task[] snapshot;
        lock (_gate) snapshot = _running.ToArray();
        await Task.WhenAll(snapshot).WaitAsync(ct);   // drain within shutdown timeout
    }
}
```

**Gotchas / follow-ups**
- Prune completed tasks from the list to avoid leaks.
- Be wary: `WaitAsync(ct)` throws when shutdown timeout cancels `ct`.

### 24.5 [Tricky] What's the difference between `_ = Foo();`, `Foo();` and `await Foo();` for a method that throws?
**Answer**
`await Foo()` surfaces the exception to your code. `Foo();` (CS4014 warning) and `_ = Foo();` both drop the Task — the exception is stored and never observed (no crash for `Task`; process crash for `async void`). With `_ =` the warning is merely suppressed.

**Explanation**
If Foo is *not* async and throws synchronously before returning a Task, the exception **does** propagate immediately at call. Only async methods defer exceptions into the Task.

**Code**
```csharp
static Task Sync() => throw new InvalidOperationException("sync throw");        // throws at call time
static async Task Async() { await Task.Yield(); throw new InvalidOperationException("async throw"); }

try { _ = Sync(); } catch (Exception e) { Console.WriteLine("caught: " + e.Message); }  // Output: caught: sync throw
try { _ = Async(); Console.WriteLine("no exception here"); } catch { }                   // Output: no exception here
await Task.Delay(100);
```

**Interview traps**
- `_ =` hides the warning, not the bug.
- Scoped services + fire-and-forget = ObjectDisposed.
- No shutdown tracking → lost work.
- Non-async Task-returning methods can throw synchronously.

**Cheat sheet**
- Await it, or queue it.
- Channel + BackgroundService for in-process jobs.
- Durable queue/outbox for must-not-lose.
- New DI scope inside background work.
- Catch + log in any forget helper.

---

## Topic 25: Async output-prediction puzzles

**Study notes**

Rules of thumb for predicting output:
1. Code before the first *incomplete* await runs synchronously on the caller's thread.
2. Calling an async method returns to the caller at the first incomplete await; the caller continues.
3. Already-completed awaitables continue synchronously (no yield).
4. Continuations in a console app (no SC) run on pool threads; with `ConfigureAwait`/SC they may run elsewhere. Ordering between *independent* continuations isn't guaranteed — questions avoid it by using `await`/sequencing.
5. `Task.Run`/`Task.Delay` start immediately (hot tasks); LINQ `Select` is lazy.
6. `await` unwraps exceptions; `.Wait()` wraps.
7. `for`-loop closure capture vs `foreach`.

### 25.1 [Easy] What is printed?
**Answer**
`Start`, `Inside`, `End`, `After`. `Inside` prints synchronously before `End`, and since `Delay(100)` is incomplete, `End` prints before `After`.

**Explanation**
`M()` runs synchronously until the first await, so "Inside" appears before "End".

**Code**
```csharp
static async Task M()
{
    Console.WriteLine("Inside");
    await Task.Delay(100);
    Console.WriteLine("After");
}
Console.WriteLine("Start");
var t = M();
Console.WriteLine("End");
await t;
// Output: Start, Inside, End, After
```

### 25.2 [Follow-up] Sequential vs concurrent: what's the elapsed time?
**Answer**
Sequential awaits: ~300 ms (100+200). Starting both first then awaiting: ~200 ms.

**Explanation**
The tasks begin when called; awaiting later only waits for already-running work.

**Code**
```csharp
var sw = System.Diagnostics.Stopwatch.StartNew();
await Task.Delay(100); await Task.Delay(200);
Console.WriteLine($"seq ~{sw.ElapsedMilliseconds / 100 * 100}ms");   // Output: seq ~300ms

sw.Restart();
var a = Task.Delay(100); var b = Task.Delay(200);
await a; await b;
Console.WriteLine($"conc ~{sw.ElapsedMilliseconds / 100 * 100}ms");  // Output: conc ~200ms
```

### 25.3 [Medium] What's the output with exceptions and Wait/await?
**Answer**
`AggregateException` from `.Wait()`; `InvalidOperationException` from `await`.

**Explanation**
`Wait` wraps; `await` rethrows the first inner.

**Code**
```csharp
static Task Fail() => Task.FromException(new InvalidOperationException("x"));
try { Fail().Wait(); } catch (Exception e) { Console.WriteLine(e.GetType().Name); }       // Output: AggregateException
try { await Fail(); }  catch (Exception e) { Console.WriteLine(e.GetType().Name); }       // Output: InvalidOperationException
```

### 25.4 [Hard] Predict: `finally`, `return`, and await ordering.
**Answer**
`A`, `Main continues`, `finally`, `result=1`. The `finally` block runs after the delay, before the Task completes and the caller receives the value.

**Explanation**
`A` is printed synchronously; at `await Task.Delay` control returns to `Main`, which prints "Main continues". After the delay completes, `return 1` triggers `finally` before completing the Task.

**Code**
```csharp
static async Task<int> F()
{
    try
    {
        Console.WriteLine("A");
        await Task.Delay(50);
        return 1;
    }
    finally { Console.WriteLine("finally"); }
}
var t = F();
Console.WriteLine("Main continues");
Console.WriteLine($"result={await t}");
// Output: A, Main continues, finally, result=1
```

**Gotchas / follow-ups**
- If the caller *abandons* the Task, `finally` still runs when the delay completes.

### 25.5 [Hard] What does this print? Closure + Task.Run + loop variable.
**Answer**
`for`: usually `3 3 3` (value after the loop; any order). `foreach`: `0 1 2` in any order. Fixed with a copy local.

**Explanation**
C# 5+ gives `foreach` a fresh variable per iteration; `for` shares one.

**Code**
```csharp
var r1 = new List<int>();
var tasks = new List<Task>();
for (int i = 0; i < 3; i++) tasks.Add(Task.Run(() => { lock (r1) r1.Add(i); }));
await Task.WhenAll(tasks);
Console.WriteLine(string.Join(",", r1));    // Output (likely): 3,3,3 (racy; could include smaller values)

var r2 = new List<int>();
tasks.Clear();
foreach (var j in Enumerable.Range(0, 3)) tasks.Add(Task.Run(() => { lock (r2) r2.Add(j); }));
await Task.WhenAll(tasks);
Console.WriteLine(string.Join(",", r2.Order())); // Output: 0,1,2
```

### 25.6 [Tricky] `Task.Delay` vs `Thread.Sleep` inside async; and `ConfigureAwait` thread id puzzle.
**Answer**
`Thread.Sleep` in an async method blocks the caller until finished (since sync part), while `Task.Delay` returns control. So `Thread.Sleep(1000); await Task.Delay(1)` makes the *caller* wait 1 s before `var t = ...` returns.

**Explanation**
Common interview trick: the "async" method's first sync segment blocks the caller.

**Code**
```csharp
static async Task Bad()
{
    Thread.Sleep(500);        // blocks the caller!
    await Task.Delay(1);
}
var sw = System.Diagnostics.Stopwatch.StartNew();
var t = Bad();                                                      // returns after ~500ms
Console.WriteLine($"call returned after ~{sw.ElapsedMilliseconds / 100 * 100}ms"); // Output: call returned after ~500ms
await t;
```

**Interview traps**
- Sync part before first await blocks the caller.
- `Task.WhenAll` order = input order, not completion.
- Lazy LINQ + multiple enumeration doubles work.
- Loop variable capture in `for`.
- `finally` runs after the awaited continuation, before the Task completes.

**Cheat sheet**
- Trace: sync prefix → first incomplete await → caller resumes → continuation.
- Hot tasks start on creation.
- `await` vs `.Wait()` exception types differ.
- Completed awaits don't yield.
- `foreach` captures per iteration; `for` doesn't.

---

*End of A04 — C# Async, Tasks & Threading.*
