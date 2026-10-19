# Data Structures and Algorithms: A Practical Learning Guide

## Foundation to intermediate, for a .NET and React developer

Prepared: October 19, 2026

**Purpose:** build the ability to choose appropriate collections, explain the performance of code, solve common interview problems, and recognize inefficient patterns in real applications.

You already know how to build applications. This guide connects that experience to the vocabulary and reasoning behind data structures and algorithms. It is not a requirement to restart your career or become a competitive programmer.

**Learning target:** understand a problem, produce a correct solution, explain its cost, identify limitations, and improve it when the improvement is justified.

---

## Contents

1. How to study and run the examples
2. Step 1: Foundations and complexity
3. Step 2: Collections and core data structures
4. Step 3: Searching, sorting, and problem-solving patterns
5. Step 4A: Recursion and backtracking
6. Step 4B: Trees
7. Step 4C: Graphs
8. Step 4D: Dynamic programming
9. Step 5: Applying DSA to .NET and React
10. Testing correctness and measuring performance
11. Guided exercises, hints, and solutions
12. Practice sequence and capstone projects
13. A flexible 12-week learning schedule
14. Revision sheets and glossary
15. Official references

Find a topic using your Markdown viewer's outline or text search. Read in order initially; later, use the guide as a reference.

---

# 1. How to study and run the examples

## 1.1 The learning loop

For each topic:

1. Read the definition and draw the structure.
2. Trace a tiny example by hand.
3. Predict the output before running the code.
4. Type the solution yourself rather than copying immediately.
5. Test empty, one-element, duplicate, missing, and boundary inputs.
6. Explain the time and space complexity aloud.
7. Solve a slightly different problem without looking back.

Understanding a solution while reading is not the same as being able to derive it. Revisit a problem after two days and again after one week.

Keep a small personal record: problem, initial idea, mistake, corrected reasoning, and next review date. Record the lesson, not just the code.

## 1.2 Running the C# examples

The examples use C# features available with .NET 8. They do not require third-party packages. They also run on later compatible .NET SDKs.

From PowerShell, create a separate practice directory, not inside a production repository:

```powershell
Set-Location "$HOME\Desktop"
dotnet new console --name DsaPractice --framework net8.0
Set-Location .\DsaPractice
dotnet run
```

If your Desktop is redirected to OneDrive, navigate to the actual Desktop directory first instead of assuming `$HOME\Desktop` exists.

Replace `Program.cs` with **one C# code block at a time**. The normal console template enables implicit usings. If your project disables them, add the following namespaces as needed: `System`, `System.Collections.Generic`, `System.Linq`, `System.Text`, and `System.Diagnostics`.

Each `csharp` block is an independent example. Do not paste the entire guide into one file: some examples deliberately reuse names. Type declarations appear after top-level statements where necessary.

JavaScript blocks teach frontend concepts separately. Blocks marked `text` explain algorithms, traces, or context-dependent EF Core code; they are not standalone C# programs.

## 1.3 Conventions used throughout

- `n`: number of input elements unless defined otherwise.
- `m`: size of a second independent input.
- `V`: graph vertices; `E`: graph edges.
- `h`: tree height.
- `k`: window size, requested result count, or number of distinct values; the local section specifies which.
- **Auxiliary space:** additional working memory, including recursion frames, but excluding the original input and usually excluding returned output.
- **Output space:** memory needed to hold the returned answer.
- Hash-table complexities assume a reasonable hash distribution and bounded-cost hashing/equality, unless stated otherwise.
- Examples assume non-null inputs unless a null check or a different contract is explicitly shown.

Sentinels such as `-1` are used only where the method contract defines them. Invalid arguments are different from a valid input with no solution.

## 1.4 Two useful learning tracks

**Practical engineering:** emphasize collections, complexity, LINQ, correctness, database boundaries, and measurement.

**Interview preparation:** add more timed practice with two pointers, windows, trees, graphs, and dynamic programming.

Do not skip practical engineering to memorize advanced puzzle solutions. Conversely, production experience does not automatically prepare you for timed algorithm interviews.

---

# 2. Step 1: Foundations and complexity

## 2.1 What is a data structure?

A **data structure** is an organization of data that makes certain operations convenient or efficient.

Examples:

- Array: elements accessible by numerical position.
- Dictionary: values retrievable through keys.
- Queue: elements removed in arrival order.
- Tree: parent-child relationships.
- Graph: general connections between entities.

Think of the difference between a pile of papers, a numbered filing cabinet, and an alphabetical index. They may contain the same information but support different operations efficiently.

## 2.2 What is an algorithm?

An **algorithm** is a precise sequence of steps for solving a class of problems.

To find a customer:

1. Read each customer in sequence.
2. Compare the customer ID with the target.
3. Return the first match, or report no match.

This is linear search. The algorithm is independent of whether the customers represent banking clients, CRM records, or application users.

## 2.3 Abstract data type versus implementation

An **abstract data type**, or ADT, describes operations and behavior without dictating storage.

A queue promises FIFO behavior: the first item added is the first removed.

A queue could be implemented with:

- A linked list.
- A circular array.
- Another suitable internal representation.

`Queue<T>` is a .NET implementation of a queue ADT. A linked list is not automatically a queue: the way you use its operations determines the behavior.

**Learning note:** separate "what behavior do I need?" from "which concrete type provides it?"

## 2.4 Begin with a contract

Before writing an algorithm, define:

| Question | Example: finding two values with a target sum |
|---|---|
| Input | An integer array and target |
| Output | Two distinct indices, or no result |
| Duplicates allowed? | Yes |
| Same element twice? | No |
| Input sorted? | No |
| Can input be modified? | No |
| Multiple valid answers? | Any one pair is acceptable |
| Numeric limits? | Intermediate sums may require `long` |
| Empty input? | Return no result |

Changing any of these can change the correct algorithm.

For example, sorting first may improve search but lose original indices unless you preserve them. Returning values instead of indices is not an interchangeable answer.

## 2.5 Correctness, invariants, and termination

An **invariant** is a statement that remains true at a specific point in every iteration or recursive call.

For a running maximum:

> After processing the first `i` elements, `maximum` is the largest value among those elements.

```csharp
int FindMaximum(int[] values)
{
    if (values.Length == 0)
        throw new ArgumentException("At least one value is required.", nameof(values));

    int maximum = values[0];

    for (int i = 1; i < values.Length; i++)
        maximum = Math.Max(maximum, values[i]);

    return maximum;
}

Console.WriteLine(FindMaximum(new[] { -7, -2, -9 })); // -2
```

Trace:

| Processed value | Maximum |
|---|---:|
| -7 | -7 |
| -2 | -2 |
| -9 | -2 |

Why correct:

- **Initialization:** the first value is the maximum of the first one-element prefix.
- **Maintenance:** taking the larger of the old maximum and next value preserves the invariant.
- **Termination:** the index increases until all elements are covered.
- **Conclusion:** the maximum covers the entire input.

Time: O(n). Auxiliary space: O(1).

Common mistake: initializing the maximum to zero produces the wrong answer for an all-negative array.

## 2.6 What time complexity measures

Time complexity models how work grows as input size grows.

It does **not** directly predict milliseconds. Hardware, memory access, allocation, JIT compilation, network calls, and data distribution influence actual elapsed time.

If one solution examines every customer once, doubling customers roughly doubles that part of the work.

If another compares every customer with every other customer, doubling the input roughly quadruples the comparisons.

Choose a meaningful unit: comparisons, visited records, hash operations, or another relevant operation.

## 2.7 Big O, Big Omega, and Big Theta

- **Big O:** an asymptotic upper bound.
- **Big Omega:** an asymptotic lower bound.
- **Big Theta:** a matching upper and lower bound, describing a tight growth rate.

Suppose `T(n) = 3n + 10`. Its growth is Theta(n): both upper and lower bounds are linear for sufficiently large `n`.

It is also technically O(n squared), but that bound is less informative. In engineering conversations, people usually give the tightest useful bound when saying "Big O."

**Important distinction:** O does not mean "worst case" by definition. You can state an upper bound on best-case, average-case, or worst-case behavior. Specify the case being discussed.

## 2.8 Common growth rates

The following values are illustrations of operation-count growth, not benchmark predictions. Logarithms are base 2 here.

| Growth | n = 10 | n = 1,000 | n = 1,000,000 | Example |
|---|---:|---:|---:|---|
| O(1) | 1 | 1 | 1 | Valid array index access |
| O(log n) | about 4 | about 10 | about 20 | Binary search |
| O(n) | 10 | 1,000 | 1,000,000 | Full traversal |
| O(n log n) | about 33 | about 9,966 | about 19,931,569 | Efficient comparison sorting |
| O(n squared) | 100 | 1,000,000 | 1,000,000,000,000 | All ordered pairs |
| O(2 to the n) | 1,024 | enormous | impractical | Enumerating subsets |
| O(n factorial) | 3,628,800 | enormous | impractical | Enumerating permutations |

Logarithmic work is small because the remaining problem repeatedly shrinks by a factor. Exponential work grows quickly because the number of possibilities multiplies with each decision.

## 2.9 Dropping constants and combining work

Two independent passes:

```text
for every customer: validate
for every customer: summarize

Work: n + n = 2n
Growth: O(n)
```

Nested passes:

```text
for every order:
    scan every customer

Work: n * m
Growth: O(nm)
```

If both inputs have size `n`, O(nm) becomes O(n squared). Keep different input sizes separate until there is a reason to relate them.

A pass plus sorting:

```text
validate n values
sort n values

O(n + n log n) = O(n log n)
```

The dominant term describes long-run growth. Constants still matter for practical performance, especially for small inputs.

## 2.10 Nested loops are not automatically quadratic

Case A:

```text
for i = 0 to n - 1:
    for j = 0 to 4:
        work
```

Exactly five inner iterations per outer iteration: O(n).

Case B:

```text
for i = 0 to n - 1:
    for j = i + 1 to n - 1:
        work
```

Total: `(n - 1) + (n - 2) + ... + 1`, which is `n(n - 1)/2`: O(n squared).

Case C:

```text
left = 0
for right = 0 to n - 1:
    while the window needs shrinking:
        left++
```

If `left` only advances and never exceeds `n`, all inner iterations together total at most `n`. The combined work can be O(n), not O(n squared).

This is the key reasoning behind many sliding-window algorithms.

## 2.11 Best, average, worst, expected, and amortized

For linear search:

- Best case: target is first, O(1).
- Worst case: target is last or absent, O(n).
- Average case: under a suitable distribution, still O(n).

**Expected complexity** usually refers to averaging over randomness or a model, such as expected hash distribution.

**Amortized complexity** averages the cost over a sequence of operations, without requiring random inputs.

For a dynamic array, most appends are cheap, but occasional resizing copies existing elements. A geometric growth strategy makes a sequence of `n` appends O(n) total, or O(1) amortized per append. A particular resizing append can still cost O(n).

Average-case and amortized are not synonyms.

## 2.12 Space complexity and hidden allocations

```text
One running total: O(1) auxiliary space.
Dictionary containing up to n distinct IDs: O(n).
Recursive call chain of depth h: O(h) stack space.
Returning n result values: O(n) output space.
```

`ToList()`, `ToArray()`, sorting buffers, closures, and substring creation can allocate memory even if no `new` appears in your code.

An iterator may delay output allocation but still maintain substantial internal state. Deferred does not automatically mean streaming or constant space.

## 2.13 A small introduction to recurrences

A **recurrence** expresses a recursive algorithm's cost in terms of smaller inputs.

- `T(n) = T(n - 1) + O(1)`: a chain of `n` calls, O(n).
- `T(n) = T(n / 2) + O(1)`: halve the problem each time, O(log n).
- `T(n) = 2T(n / 2) + O(n)`: two halves plus a linear merge, O(n log n).
- `T(n) = T(n - 1) + T(n - 2) + O(1)`: naive Fibonacci, exponential.

For merge sort, each level processes O(n) elements and there are O(log n) levels. Multiplying gives O(n log n).

You do not need to memorize a formal theorem before understanding these examples.

## 2.14 Foundation checkpoint

Try before continuing:

- Explain why `3n + 100` is linear.
- Explain why a full scan inside another full scan can be quadratic.
- Give a nested-loop example that is linear.
- Distinguish a valid empty result from an invalid argument.
- Explain why an O(n) method can be slower than an O(n log n) method on a tiny input.
- Include recursive stack frames in a space analysis.

**Revision sentence:** complexity describes scaling; correctness describes whether the answer is right. Neither replaces the other.

---

# 3. Step 2: Collections and core data structures

The collection-selection guidance in reference S1 provides the .NET API context. The explanations below develop the algorithmic reasoning using original examples.

## 3.1 Arrays

An **array** stores a fixed number of elements, accessible by index.

```text
Index:   0    1    2    3
Value:  12   24   36   48
```

Analogy: numbered lockers. Knowing a locker number lets you go directly to it.

Typical costs:

| Operation | Cost |
|---|---|
| Read/write at valid index | O(1) |
| Find an arbitrary value in unsorted data | O(n) |
| Traverse | O(n) |
| Insert while preserving order | O(n) shifts/copying, with space available or a new array |
| Delete while preserving order | O(n) shifts, with separate logical count or a new array |

An array has fixed length. "Insert" and "delete" are conceptual operations on a logical sequence, not methods that change an existing array's length.

```csharp
int[] amounts = { 10, 20, 30 };
amounts[1] = 25;

long total = 0;
foreach (int amount in amounts)
    total += amount;

Console.WriteLine(amounts[1]); // 25
Console.WriteLine(total);     // 65

int[,] grid = new int[2, 3];
grid[1, 2] = 7;

int[][] rows =
{
    new[] { 1, 2 },
    new[] { 3, 4, 5 }
};

Console.WriteLine(grid[1, 2]); // 7
Console.WriteLine(rows[1][2]); // 5
```

A rectangular array has a uniform shape; a jagged array is an array of arrays and can have different row lengths.

For reference-type elements, the array contains references. The referenced objects are not necessarily adjacent in memory.

**Use for:** indexed access, fixed-size buffers, matrices, many algorithm problems.

**Avoid assuming:** "I have an array, so searching any value is O(1)." Position lookup and value search are different operations.

## 3.2 Dynamic arrays: `List<T>`

A `List<T>` presents a variable-size sequence backed by an internal array.

- **Count:** number of elements currently stored.
- **Capacity:** number of elements the current backing storage can hold before growth is needed.

```csharp
var numbers = new List<int>(capacity: 4);
numbers.Add(10);
numbers.Add(20);
numbers.Insert(0, 5);

Console.WriteLine(string.Join(", ", numbers)); // 5, 10, 20
Console.WriteLine(numbers.Count);               // 3
Console.WriteLine(numbers.Capacity >= numbers.Count); // True

numbers.RemoveAt(1);
Console.WriteLine(string.Join(", ", numbers)); // 5, 20
```

Costs:

- Index access: O(1).
- Append: O(1) amortized, O(n) for an individual growth operation.
- Insert/remove near the beginning: O(n) because elements shift.
- Search with `Contains` or a predicate scan: O(n).

When a reasonable size is known, an initial capacity can reduce allocations. Do not reserve enormous capacity speculatively.

Removing items from the front repeatedly can lead to quadratic total shifting. For FIFO consumption, consider a queue.

**Learning note:** a list is often a good default. A theoretically sophisticated structure is not automatically better for small, sequential workloads.

## 3.3 Strings and text

A C# string is immutable. An operation that changes its textual content produces another string rather than editing the existing instance.

Repeated concatenation in a growing loop can copy increasingly large prefixes:

```text
1 + 2 + 3 + ... + n copied units -> potentially quadratic copying
```

For incremental text construction, consider `StringBuilder`:

```csharp
using System.Text;

var builder = new StringBuilder();
for (int i = 1; i <= 3; i++)
{
    if (i > 1)
        builder.Append(", ");
    builder.Append(i);
}

Console.WriteLine(builder.ToString()); // 1, 2, 3
```

Do not mechanically replace every interpolation or small concatenation with a builder. Use the simplest expression unless repeated growth is an actual issue.

### Character counting is more subtle than it appears

In .NET, `char` represents a UTF-16 code unit. It is not guaranteed to represent an entire Unicode scalar value or a displayed character.

- `string.Length`: UTF-16 code-unit count.
- `Rune`: Unicode scalar value.
- Text elements/grapheme clusters: closer to user-perceived characters.

See S4 for the distinction.

The string algorithms in this guide explicitly operate on UTF-16 code units. This is useful for learning but may not match requirements for emoji, combining marks, or multilingual user-facing text.

For identifier comparisons, choose explicit ordinal semantics when appropriate. Human-language sorting may require culture-sensitive rules. Normalization and case handling are product decisions, not an automatic `.ToLower()` everywhere.

## 3.4 Hash tables: `Dictionary<TKey,TValue>`

A dictionary maps a unique key to a value.

```text
Customer ID -> Customer
1001        -> Alice
1002        -> Ben
```

Conceptually:

1. Compute a hash from the key.
2. Use the hash to identify a candidate location or bucket.
3. Use equality checks to distinguish matching and colliding keys.

A **collision** occurs when different keys map into the same hash/bucket space. Collision handling is a normal part of a hash table.

```csharp
var namesById = new Dictionary<int, string>
{
    [101] = "Alice",
    [102] = "Ben"
};

if (namesById.TryGetValue(102, out string? name))
    Console.WriteLine(name); // Ben
else
    Console.WriteLine("Customer not found.");

Console.WriteLine(namesById.TryAdd(101, "Another Alice")); // False
namesById[101] = "Alice Updated";
Console.WriteLine(namesById[101]); // Alice Updated
```

Important behaviors:

- `Add` rejects a duplicate key.
- `TryAdd` reports whether insertion succeeded.
- The indexer assignment inserts or replaces.
- Reading a missing key through the indexer throws.
- `TryGetValue` handles an expected miss without a separate lookup.

Expected lookup/insertion/removal is O(1) under the stated assumptions; pathological collisions can degrade operations toward O(n). Resizing also affects individual insertion cost. The official API context is S2.

String hashing/equality can depend on key length. Calling dictionary lookup O(1) usually treats key-processing cost as bounded. With long variable-length keys, discuss that dimension too.

### Equality and key safety

If two keys are equal under the chosen equality comparer, their hash codes must agree. The reverse is not required.

Avoid mutating fields that determine equality/hash identity while an object is used as a key.

Choose case semantics deliberately:

```csharp
var permissions = new Dictionary<string, bool>(StringComparer.OrdinalIgnoreCase)
{
    ["ReadReports"] = true
};

Console.WriteLine(permissions.ContainsKey("readreports")); // True
```

Do not depend on dictionary enumeration as a contractual sorted order. If the application requires ordering, represent or request that ordering explicitly.

**Use for:** repeated lookups, frequency counting, indexing records, memoization.

**Do not use automatically:** when there is only one lookup over a tiny list; building an index may add unnecessary allocation.

## 3.5 Sets: `HashSet<T>`

A set represents membership without storing duplicate elements.

```csharp
var permitted = new HashSet<int> { 1, 2, 3 };
Console.WriteLine(permitted.Add(2)); // False
Console.WriteLine(permitted.Add(4)); // True
Console.WriteLine(permitted.Contains(3)); // True

var requested = new HashSet<int> { 2, 4, 5 };
requested.IntersectWith(permitted);

Console.WriteLine(string.Join(", ", requested.OrderBy(x => x))); // 2, 4
```

Useful operations:

- Union: values in either set.
- Intersection: values in both.
- Difference: values in one but not the other.
- Subset check: every element of one exists in the other.

Membership is expected O(1), with hash/equality qualifications like a dictionary. Set operations depend on the number of elements processed and the concrete operands.

### Duplicate detection

```csharp
bool HasDuplicate(int[] values)
{
    var seen = new HashSet<int>();
    foreach (int value in values)
    {
        if (!seen.Add(value))
            return true;
    }

    return false;
}

Console.WriteLine(HasDuplicate(new[] { 5, 8, 5 })); // True
Console.WriteLine(HasDuplicate(Array.Empty<int>())); // False
```

Expected time: O(n). Auxiliary space: O(n) worst case.

Trace: add 5, add 8, attempt to add 5 again; the third add reports that it already exists.

Alternative: sort and compare neighbors. That costs O(n log n) and may modify the input, but has different memory tradeoffs.

## 3.6 Stacks: last in, first out

A stack supports:

- `Push`: add to top.
- `Pop`: remove top.
- `Peek`: inspect top.

Analogy: a stack of plates.

```csharp
var history = new Stack<string>();
history.Push("Open document");
history.Push("Type title");

Console.WriteLine(history.Pop());  // Type title
Console.WriteLine(history.Peek()); // Open document
```

Push is amortized O(1) for an array-backed growing stack; pop and peek are O(1). Empty pop/peek must be handled; use `TryPop`/`TryPeek` when emptiness is expected.

Use cases: undo operations, expression parsing, DFS, and matching delimiters.

### Balanced brackets

Contract: consider only `()[]{}` as brackets and ignore other characters.

```csharp
bool HasBalancedBrackets(string input)
{
    var openings = new Stack<char>();

    foreach (char current in input)
    {
        if (current is '(' or '[' or '{')
        {
            openings.Push(current);
        }
        else if (current is ')' or ']' or '}')
        {
            if (!openings.TryPop(out char opening))
                return false;

            bool matches =
                (opening == '(' && current == ')') ||
                (opening == '[' && current == ']') ||
                (opening == '{' && current == '}');

            if (!matches)
                return false;
        }
    }

    return openings.Count == 0;
}

Console.WriteLine(HasBalancedBrackets("{a + [b * (c)]}")); // True
Console.WriteLine(HasBalancedBrackets("([)]"));             // False
Console.WriteLine(HasBalancedBrackets(""));                 // True
```

Invariant: the stack contains unmatched opening brackets in nesting order.

Time O(n); auxiliary space O(n) worst case.

Why counts alone fail: `([)]` contains matching counts but invalid nesting.

## 3.7 Queues: first in, first out

A queue supports enqueue at the back and dequeue at the front.

Analogy: customers waiting in line.

```csharp
var jobs = new Queue<string>();
jobs.Enqueue("Send receipt");
jobs.Enqueue("Generate report");

while (jobs.TryDequeue(out string? job))
    Console.WriteLine(job);

// Send receipt
// Generate report
```

Enqueue is amortized O(1) with occasional resizing; dequeue and peek are O(1).

`Queue<T>` maintains a logical front without shifting every remaining element.

Use cases: BFS and simple in-process work ordering.

**Production boundary:** an in-memory queue is not a durable message broker. It does not automatically provide retries, acknowledgements, persistence, distributed coordination, or exactly-once processing.

`Queue<T>` is also not generally safe for concurrent mutation. Consider an appropriate concurrent collection or channel when concurrency is required.

## 3.8 Linked lists

A singly linked node stores a value and a reference to the next node:

```text
head -> [10 | next] -> [20 | next] -> [30 | null]
```

A doubly linked node stores previous and next references.

Costs:

- Access the element at index `i`: O(i), worst O(n).
- Find a value: O(n).
- Insert after an already-known node: O(1).
- Remove an already-known node from a doubly linked list: O(1).
- Find that node first and then remove: O(n) overall.

The "already-known node" condition is essential.

### Reverse a singly linked list

```csharp
var head = new Node(10, new Node(20, new Node(30)));
Node? reversed = Reverse(head);

for (Node? node = reversed; node is not null; node = node.Next)
    Console.WriteLine(node.Value);

// 30
// 20
// 10

static Node? Reverse(Node? head)
{
    Node? previous = null;
    Node? current = head;

    while (current is not null)
    {
        Node? next = current.Next;
        current.Next = previous;
        previous = current;
        current = next;
    }

    return previous;
}

sealed class Node
{
    public int Value { get; }
    public Node? Next { get; set; }

    public Node(int value, Node? next = null)
    {
        Value = value;
        Next = next;
    }
}
```

Trace:

```text
previous = null, current = 10
save 20; make 10 point to null
save 30; make 20 point to 10
save null; make 30 point to 20
return 30
```

Invariant: `previous` heads the reversed processed prefix; `current` heads the unprocessed suffix.

Time O(n); auxiliary space O(1). This mutates the original connections. Assumes an acyclic list.

Common mistake: overwriting `current.Next` before saving it loses the rest of the chain.

In application code, `LinkedList<T>` is available. Do not reimplement a general-purpose linked list without a real requirement.

Linked lists have node/reference overhead and weaker locality than arrays. Fast theoretical insertion does not mean faster overall workloads.

## 3.9 Heaps and priority queues

A **binary min-heap** is a complete binary tree in which each parent's value is no greater than its children's values.

It is not a fully sorted tree.

```text
        2
      /   \
     5     3
    / \
   9   7
```

With zero-based array indexing:

- Parent of index `i > 0`: `(i - 1) / 2`.
- Left child: `2i + 1`.
- Right child: `2i + 2`.

Insertion bubbles upward. Removing the minimum moves the final element to the root and restores order downward.

Typical heap operations: peek O(1), insert/remove O(log n) excluding occasional backing-array growth, bottom-up construction O(n).

.NET's `PriorityQueue<TElement,TPriority>` is an array-backed **quaternary** min-heap, not a binary heap. It removes the lowest priority first and does not promise FIFO behavior for equal priorities. See S3.

```csharp
var jobs = new PriorityQueue<string, int>();
jobs.Enqueue("Low urgency", 10);
jobs.Enqueue("High urgency", 1);
jobs.Enqueue("Medium urgency", 5);

while (jobs.TryDequeue(out string? job, out int priority))
    Console.WriteLine($"{priority}: {job}");

// 1: High urgency
// 5: Medium urgency
// 10: Low urgency
```

If equal-priority FIFO is required, make the priority a tuple such as `(Urgency, SequenceNumber)` and increment the sequence for each enqueue.

### Largest K values without sorting everything

Keep a min-heap containing at most `k` elements. Its smallest element is the least valuable currently retained candidate.

```csharp
int[] LargestK(int[] values, int k)
{
    if (k < 0 || k > values.Length)
        throw new ArgumentOutOfRangeException(nameof(k));

    if (k == 0)
        return Array.Empty<int>();

    var heap = new PriorityQueue<int, int>();

    foreach (int value in values)
    {
        heap.Enqueue(value, value);
        if (heap.Count > k)
            heap.Dequeue();
    }

    var result = new int[k];
    for (int i = 0; i < k; i++)
        result[i] = heap.Dequeue();

    return result;
}

Console.WriteLine(string.Join(", ", LargestK(new[] { 8, 1, 9, 3, 7 }, 3)));
// 7, 8, 9
```

Trace: retain 8; retain 1,8; retain 1,8,9; add 3 then discard 1; add 7 then discard 3.

Invariant: after each input value, the heap holds the largest up to `k` values seen so far, counting duplicates.

Time O(n log(k + 1) + k log(k + 1)); auxiliary space O(k), output O(k). Often summarized as O(n log k) for `k >= 2`. Draining gives ascending output.

Do not use this when you need every value sorted; ordinary sorting is clearer.

## 3.10 Ordered collections

`SortedDictionary<TKey,TValue>` and `SortedSet<T>` maintain comparer order with logarithmic lookup and updates.

Contrast:

| Requirement | Candidate |
|---|---|
| Fast expected exact-key lookup | `Dictionary<TKey,TValue>` |
| Sorted key traversal with ongoing updates | `SortedDictionary<TKey,TValue>` |
| Uniqueness and sorted traversal | `SortedSet<T>` |
| Mostly build once, sort, then read | Sorted array/list |
| Only the next minimum/maximum matters | Priority queue |

`SortedList<TKey,TValue>` uses array-based storage: logarithmic key lookup but potentially linear insertion/removal due to shifts. Its name does not mean it has the same update costs as `SortedDictionary`.

Sorting a dictionary's enumeration produces an ordered view/result, not a dictionary that maintains that sort order.

## 3.11 Collection checkpoint

Explain your choice for:

- Customer lookup by ID across thousands of orders.
- Maintaining unique selected IDs.
- Undo history.
- A queue of pending visits in a hierarchy.
- The next five highest scores.
- Sorted price updates.
- One sequential pass over 20 items.

**Revision sentence:** optimize the operations the application actually performs, including construction, mutation, memory, and ordering requirements.

---

# 4. Step 3: Searching, sorting, and problem-solving patterns

## 4.1 Linear search

Check candidates one at a time.

```csharp
int IndexOf(int[] values, int target)
{
    for (int i = 0; i < values.Length; i++)
    {
        if (values[i] == target)
            return i;
    }

    return -1;
}

Console.WriteLine(IndexOf(new[] { 9, 4, 7 }, 4)); // 1
Console.WriteLine(IndexOf(new[] { 9, 4, 7 }, 6)); // -1
```

Contract: return the first matching index, or `-1` for no match.

Worst-case time O(n), auxiliary space O(1).

Linear search is perfectly reasonable for a single search in small unsorted data. A dictionary has a build cost.

## 4.2 Binary search

Binary search discards half of a sorted search space at each step.

Prerequisite: data must be sorted under the same ordering used by the search, or the problem must have a suitable monotonic predicate.

```csharp
int BinarySearch(int[] sorted, int target)
{
    int left = 0;
    int right = sorted.Length - 1;

    while (left <= right)
    {
        int middle = left + (right - left) / 2;

        if (sorted[middle] == target)
            return middle;

        if (sorted[middle] < target)
            left = middle + 1;
        else
            right = middle - 1;
    }

    return -1;
}

Console.WriteLine(BinarySearch(new[] { 2, 5, 8, 12, 16 }, 12)); // 3
Console.WriteLine(BinarySearch(Array.Empty<int>(), 12));        // -1
```

Trace for target 12:

| Left | Right | Middle | Value | Action |
|---:|---:|---:|---:|---|
| 0 | 4 | 2 | 8 | Keep right half |
| 3 | 4 | 3 | 12 | Found |

Invariant: if the target exists and has not been found, at least one matching position remains inside `[left, right]`.

Time O(log n), auxiliary space O(1).

Why use `left + (right - left) / 2`? It avoids adding two potentially large nonnegative indices.

Common mistakes: using unsorted data, forgetting `+1`/`-1`, mixing inclusive and exclusive boundaries, or promising the first duplicate when the method returns any matching index.

Sorting first costs O(n log n). For only one search, sorting may cost more than a linear scan.

## 4.3 Lower bound: first element at least target

This variant uses a half-open range `[left, right)`.

```csharp
int LowerBound(int[] sorted, int target)
{
    int left = 0;
    int right = sorted.Length;

    while (left < right)
    {
        int middle = left + (right - left) / 2;
        if (sorted[middle] < target)
            left = middle + 1;
        else
            right = middle;
    }

    return left;
}

int[] values = { 1, 3, 3, 3, 8 };
Console.WriteLine(LowerBound(values, 3));  // 1
Console.WriteLine(LowerBound(values, 4));  // 4
Console.WriteLine(LowerBound(values, 10)); // 5
```

Returning `Length` means no element meets the condition. It is a valid insertion position, not an index to dereference.

Invariant: all positions before `left` contain values below the target; all positions at or after `right` contain values at least the target.

An **upper bound** returns the first value strictly greater than the target. With both, duplicate count is `upperBound - lowerBound`.

Time O(log n), auxiliary space O(1).

## 4.4 Sorting vocabulary

- **Stable:** elements with equal sort keys retain their original relative order.
- **In place:** usually means O(1) auxiliary array storage; some discussions permit logarithmic recursion space. State the convention.
- **Comparison sort:** determines order by comparing elements.
- **Adaptive:** can exploit an existing degree of order.

For payroll records sorted by department, stability may preserve a previous employee-name order within equal departments.

If you require a secondary ordering, explicitly specify it rather than relying accidentally on whatever order the input had.

## 4.5 Bubble sort and insertion sort

Bubble sort swaps neighboring out-of-order elements repeatedly. Large elements gradually move to the end.

Insertion sort builds a sorted prefix and inserts each next value into its place.

```csharp
void BubbleSort(int[] values)
{
    for (int end = values.Length - 1; end > 0; end--)
    {
        bool swapped = false;
        for (int i = 0; i < end; i++)
        {
            if (values[i] > values[i + 1])
            {
                (values[i], values[i + 1]) = (values[i + 1], values[i]);
                swapped = true;
            }
        }

        if (!swapped)
            break;
    }
}

void InsertionSort(int[] values)
{
    for (int i = 1; i < values.Length; i++)
    {
        int current = values[i];
        int j = i - 1;

        while (j >= 0 && values[j] > current)
        {
            values[j + 1] = values[j];
            j--;
        }

        values[j + 1] = current;
    }
}

int[] first = { 4, 2, 3, 1 };
int[] second = { 4, 2, 3, 1 };
BubbleSort(first);
InsertionSort(second);
Console.WriteLine(string.Join(", ", first));  // 1, 2, 3, 4
Console.WriteLine(string.Join(", ", second)); // 1, 2, 3, 4
```

Insertion-sort trace:

```text
[4 | 2,3,1]
[2,4 | 3,1]
[2,3,4 | 1]
[1,2,3,4]
```

Both shown implementations are stable because equal values are not moved past one another.

Worst-case time O(n squared), best case O(n), auxiliary space O(1).

Bubble sort is mainly educational. Insertion sort can be useful for small or nearly sorted ranges and inside hybrid algorithms.

## 4.6 Merge sort

Divide, solve each half, and merge sorted halves.

```csharp
void MergeSort(int[] values)
{
    var buffer = new int[values.Length];
    Sort(0, values.Length);

    void Sort(int start, int end)
    {
        if (end - start <= 1)
            return;

        int middle = start + (end - start) / 2;
        Sort(start, middle);
        Sort(middle, end);

        int left = start;
        int right = middle;
        int write = start;

        while (left < middle && right < end)
        {
            buffer[write++] = values[left] <= values[right]
                ? values[left++]
                : values[right++];
        }

        while (left < middle)
            buffer[write++] = values[left++];

        while (right < end)
            buffer[write++] = values[right++];

        Array.Copy(buffer, start, values, start, end - start);
    }
}

int[] values = { 7, 2, 5, 1 };
MergeSort(values);
Console.WriteLine(string.Join(", ", values)); // 1, 2, 5, 7
```

Trace:

```text
[7,2,5,1]
 [7,2] + [5,1]
 [7] [2] [5] [1]
 [2,7] + [1,5]
 [1,2,5,7]
```

The merge invariant: the written output prefix is sorted and contains the smallest consumed values from the two halves.

Time O(n log n) in best/average/worst cases for this version, auxiliary array O(n), recursion O(log n). Total auxiliary space O(n).

It is stable because ties select from the left half first.

## 4.7 Quicksort

Quicksort partitions around a pivot, then recursively sorts the partitions.

The following simple version deliberately chooses the last value as pivot. It is instructional, not a recommended replacement for the platform sort.

```csharp
void QuickSort(int[] values)
{
    Sort(0, values.Length - 1);

    void Sort(int left, int right)
    {
        if (left >= right)
            return;

        int pivot = values[right];
        int boundary = left;

        for (int i = left; i < right; i++)
        {
            if (values[i] <= pivot)
            {
                (values[boundary], values[i]) = (values[i], values[boundary]);
                boundary++;
            }
        }

        (values[boundary], values[right]) = (values[right], values[boundary]);
        Sort(left, boundary - 1);
        Sort(boundary + 1, right);
    }
}

int[] values = { 4, 1, 3, 2 };
QuickSort(values);
Console.WriteLine(string.Join(", ", values)); // 1, 2, 3, 4
```

For `[4,1,3,2]`, partition around 2 gives `[1,2,3,4]`, with 2 now in its final position. The two sides still need to be considered recursively.

Expected/average O(n log n) under suitable input or pivot assumptions. Worst O(n squared), including already sorted or all-equal input for this specific partition strategy. Stack space is typically O(log n) for balanced partitions, O(n) worst case.

Random pivots help avoid adversarial ordering; three-way partitioning helps with many equal values. Production implementations may use hybrid strategies to bound bad cases.

## 4.8 Prefer built-in sorting in application code

```csharp
var tickets = new List<(string Team, int Priority)>
{
    ("Payments", 2),
    ("Auth", 1),
    ("Payments", 1)
};

var ordered = tickets
    .OrderBy(t => t.Team, StringComparer.Ordinal)
    .ThenBy(t => t.Priority)
    .ToArray();

foreach (var ticket in ordered)
    Console.WriteLine($"{ticket.Team}: {ticket.Priority}");

// Auth: 1
// Payments: 1
// Payments: 2
```

`List<T>.Sort` mutates the list and is not stable. LINQ `OrderBy` is stable and buffers data when enumerated. See S5 and S6 for API behavior.

When writing a numeric comparer, prefer `a.CompareTo(b)` rather than `a - b`: subtraction can overflow and produce incorrect ordering.

**Learning note:** implement sorts to understand them; use supported built-ins for normal production work.

## 4.9 Two pointers

Two pointers track positions whose movement avoids repeated work.

Common forms:

- Left/right moving inward.
- Read/write indices for compaction.
- Fast/slow traversal.
- One pointer in each sorted sequence.

### Pair sum in sorted data

```csharp
(int Left, int Right)? FindPair(int[] sorted, long target)
{
    int left = 0;
    int right = sorted.Length - 1;

    while (left < right)
    {
        long sum = (long)sorted[left] + sorted[right];
        if (sum == target)
            return (left, right);

        if (sum < target)
            left++;
        else
            right--;
    }

    return null;
}

var pair = FindPair(new[] { 1, 2, 4, 7, 11 }, 9);
Console.WriteLine(pair); // (1, 3)
```

Trace: 1+11 too large, 1+7 too small, 2+7 matches.

Why moving left is safe when the sum is too small: pairing the same left value with any smaller right value cannot increase the sum enough.

Time O(n), auxiliary space O(1), assuming the input is already sorted.

Sorting an unsorted input adds O(n log n) and changes index meaning. For original indices without sorting, a hash map may fit better.

## 4.10 Fixed-size sliding window

A window is a contiguous range. Reuse information when moving it instead of recomputing from scratch.

### Maximum total across K consecutive values

```csharp
long MaximumWindowSum(int[] values, int k)
{
    if (k <= 0 || k > values.Length)
        throw new ArgumentOutOfRangeException(nameof(k));

    long window = 0;
    for (int i = 0; i < k; i++)
        window += values[i];

    long best = window;
    for (int right = k; right < values.Length; right++)
    {
        window += values[right];
        window -= values[right - k];
        best = Math.Max(best, window);
    }

    return best;
}

Console.WriteLine(MaximumWindowSum(new[] { 2, 1, 5, 1, 3, 2 }, 3)); // 9
Console.WriteLine(MaximumWindowSum(new[] { -5, -2, -8 }, 2));        // -7
```

Window sums: 8, 7, 9, 6.

Invariant: `window` equals the total of exactly the current `k` elements.

Brute force: O(nk). Sliding window: O(n), auxiliary space O(1).

Initializing `best` to zero would be wrong for all-negative data.

## 4.11 Variable-size sliding window

Grow the right boundary and adjust the left boundary to preserve a condition.

### Longest substring without repeated UTF-16 code units

```csharp
int LongestUniqueLength(string text)
{
    var lastSeen = new Dictionary<char, int>();
    int left = 0;
    int best = 0;

    for (int right = 0; right < text.Length; right++)
    {
        char current = text[right];
        if (lastSeen.TryGetValue(current, out int previous))
            left = Math.Max(left, previous + 1);

        lastSeen[current] = right;
        best = Math.Max(best, right - left + 1);
    }

    return best;
}

Console.WriteLine(LongestUniqueLength("abcabcbb")); // 3
Console.WriteLine(LongestUniqueLength("abba"));     // 2
Console.WriteLine(LongestUniqueLength(""));         // 0
```

Trace for `"abba"`:

| Right | Character | Left | Current window | Best |
|---:|---|---:|---|---:|
| 0 | a | 0 | a | 1 |
| 1 | b | 0 | ab | 2 |
| 2 | b | 2 | b | 2 |
| 3 | a | 2 | ba | 2 |

The `Math.Max` prevents moving left backward when an old occurrence is already outside the current window.

Expected time O(n); auxiliary space O(u), where `u` is the number of distinct code units seen, bounded by both input size and the UTF-16 code-unit alphabet.

**Caution:** a "shrink while sum is too large" strategy for numeric windows often relies on nonnegative values. With negatives, expanding can decrease a sum and shrinking can increase it. Do not apply the pattern without checking monotonicity.

## 4.12 Prefix sums

A prefix-sum array stores cumulative totals:

```text
values:  [3, 2, 5, 1]
prefix:  [0, 3, 5, 10, 11]
```

For a half-open range `[start, end)`, sum is `prefix[end] - prefix[start]`.

```csharp
long[] BuildPrefix(int[] values)
{
    var prefix = new long[values.Length + 1];
    for (int i = 0; i < values.Length; i++)
        prefix[i + 1] = prefix[i] + values[i];
    return prefix;
}

long RangeSum(long[] prefix, int start, int end)
{
    int length = prefix.Length - 1;
    if (start < 0 || end < start || end > length)
        throw new ArgumentOutOfRangeException(nameof(start), "Invalid range.");

    return prefix[end] - prefix[start];
}

long[] prefix = BuildPrefix(new[] { 3, 2, 5, 1 });
Console.WriteLine(RangeSum(prefix, 1, 3)); // 7
Console.WriteLine(RangeSum(prefix, 2, 2)); // 0
```

Build O(n), each query O(1), auxiliary storage O(n).

Why subtraction works: the larger prefix contains everything up to `end`; subtracting the prefix before `start` removes the unwanted beginning.

Use for many range-total queries on mostly static data. Updates can invalidate many prefix entries, so frequent updates require a different design.

## 4.13 Frequency maps

Count occurrences with a dictionary:

```csharp
var frequency = new Dictionary<char, int>();
foreach (char character in "banana")
{
    frequency.TryGetValue(character, out int count);
    frequency[character] = count + 1;
}

foreach (var entry in frequency.OrderBy(p => p.Key))
    Console.WriteLine($"{entry.Key}: {entry.Value}");

// a: 3
// b: 1
// n: 2
```

Expected counting time O(n); auxiliary space O(u) for distinct values. Sorting the displayed keys adds O(u log u).

Useful for anagrams, inventory counts, vote counts, and comparing multisets.

A **multiset** tracks multiplicity: `[a,a,b]` differs from `[a,b]`, even though their ordinary sets are equal.

## 4.14 Intervals

Intervals model bookings, date ranges, processing windows, or IP ranges.

Define endpoint semantics first:

- Closed `[start, end]`: endpoints included.
- Half-open `[start, end)`: start included, end excluded.

For half-open bookings, `[9,10)` and `[10,11)` do not overlap.

### Merge overlapping half-open intervals

Contract: intervals have integer endpoints with `Start < End`. Touching intervals remain separate.

```csharp
(int Start, int End)[] MergeIntervals((int Start, int End)[] intervals)
{
    foreach (var interval in intervals)
    {
        if (interval.Start >= interval.End)
            throw new ArgumentException("Intervals must have positive length.", nameof(intervals));
    }

    var sorted = intervals.OrderBy(x => x.Start).ThenBy(x => x.End).ToArray();
    var merged = new List<(int Start, int End)>();

    foreach (var current in sorted)
    {
        if (merged.Count == 0 || current.Start >= merged[^1].End)
        {
            merged.Add(current);
        }
        else
        {
            var last = merged[^1];
            merged[^1] = (last.Start, Math.Max(last.End, current.End));
        }
    }

    return merged.ToArray();
}

var result = MergeIntervals(new[] { (1, 4), (2, 6), (6, 8), (10, 12) });
Console.WriteLine(string.Join(", ", result)); // (1, 6), (6, 8), (10, 12)
```

Trace: `[1,4)` overlaps `[2,6)`, producing `[1,6)`. `[6,8)` only touches and remains separate.

Time O(n log n), auxiliary space O(n), output O(n).

If business rules merge adjacent ranges too, change the separation condition deliberately and update the tests.

## 4.15 Greedy reasoning

A **greedy algorithm** makes a locally preferred choice and does not revisit it.

Greedy is correct only when the problem has a supporting property or proof.

### A working example: maximize non-overlapping meetings

For one room with fixed, positive-length half-open meetings:

1. Sort by end time.
2. Select the meeting that ends earliest.
3. Continue with meetings starting at or after the last selected end.

Why it works: selecting the earliest-ending available meeting leaves at least as much remaining time as any other first selection. An optimal solution can exchange its first meeting for that one without reducing how many later meetings fit.

Time O(n log n), or O(n) if already ordered by end time.

### A failing example: arbitrary coin change

Coins `[1,3,4]`, target 6:

- Greedy largest-first: 4+1+1, three coins.
- Optimal: 3+3, two coins.

This counterexample is enough to disprove the general greedy strategy.

**Revision sentence:** recognizing a pattern is a starting hypothesis, not a correctness proof.

---

# 5. Step 4A: Recursion and backtracking

## 5.1 Recursion

A recursive function solves a problem by calling itself on a smaller or structurally simpler problem.

Every recursive design needs:

- A base case.
- A recursive case.
- Progress toward termination.
- A definition of what each call returns or accomplishes.

### Factorial

```csharp
long Factorial(int n)
{
    if (n < 0 || n > 20)
        throw new ArgumentOutOfRangeException(nameof(n), "Use a value from 0 through 20.");

    return Calculate(n);

    long Calculate(int value) =>
        value <= 1 ? 1 : checked(value * Calculate(value - 1));
}

Console.WriteLine(Factorial(5)); // 120
Console.WriteLine(Factorial(0)); // 1
```

Call expansion:

```text
Factorial(5)
5 * Factorial(4)
5 * 4 * Factorial(3)
5 * 4 * 3 * Factorial(2)
5 * 4 * 3 * 2 * Factorial(1)
5 * 4 * 3 * 2 * 1
```

Time O(n), stack space O(n).

The 20 limit prevents overflow for `long`. An iterative implementation can use O(1) auxiliary space.

Do not confuse the example's O(n) computational work with the value `n!` that it computes.

## 5.2 Call stack

Each active call has information such as local variables and where execution should resume.

```text
top -> Calculate(1)
       Calculate(2)
       Calculate(3)
       Calculate(4)
       Calculate(5)
```

The calls finish in reverse order.

Very deep recursion risks stack overflow. Do not rely on the runtime eliminating recursive frames. For unknown-depth external data, iterative traversal with an explicit stack can be safer.

An explicit stack also consumes memory; it changes control and capacity management, not the need to remember unfinished work.

## 5.3 Backtracking

Backtracking explores choices while undoing temporary state:

```text
choose
explore
undo
```

Use it when you need combinations, arrangements, or constraint-satisfying possibilities.

### Generate subsets of distinct values

```csharp
List<int[]> Subsets(int[] values)
{
    var result = new List<int[]>();
    var current = new List<int>();
    Visit(0);
    return result;

    void Visit(int index)
    {
        if (index == values.Length)
        {
            result.Add(current.ToArray());
            return;
        }

        Visit(index + 1);
        current.Add(values[index]);
        Visit(index + 1);
        current.RemoveAt(current.Count - 1);
    }
}

foreach (int[] subset in Subsets(new[] { 1, 2 }))
    Console.WriteLine($"[{string.Join(", ", subset)}]");

// []
// [2]
// [1]
// [1, 2]
```

Decision tree:

```text
                 []
           skip 1 / \ take 1
                []   [1]
       skip/take 2     skip/take 2
          [] [2]       [1] [1,2]
```

There are `2^n` subsets of `n` distinct items.

Because the code copies each completed subset, total time and stored output are O(n * 2^n). Auxiliary working space is O(n), excluding output.

Common mistake: adding `current` itself instead of a copy makes multiple results reference the same changing list.

If input values repeat, this explores positions and may produce identical value-subsets. Unique-value-subset requirements need duplicate handling.

## 5.4 Permutations versus combinations

- **Permutation:** order matters. `[1,2]` and `[2,1]` are different.
- **Combination:** order does not matter.
- **Subset:** any selection, including empty and full.

For permutations, maintain used positions:

```text
if path length equals n:
    copy path into results
for every unused position:
    mark used
    append value
    recurse
    remove value
    unmark used
```

There are `n!` full permutations of distinct values. Returning each as an `n`-element array needs O(n * n!) output work and space.

For combinations of exactly `k` positions, use an increasing start index so the same selection is not generated in different orders.

## 5.5 Pruning

**Pruning** stops a search branch once it cannot yield a valid solution.

For choosing exactly `k` items:

- Stop when `k` items have been chosen.
- Stop when too few items remain to reach `k`.

Pruning improves practical work, but do not claim polynomial worst-case complexity without a proof.

For a sum target, stopping when the partial sum exceeds the target is safe only under constraints such as nonnegative remaining values. Negative values can invalidate that reasoning.

## 5.6 Checkpoint

- Explain the base case before writing the recursive call.
- Trace the stack on three input values.
- Explain where state is restored.
- Separate auxiliary memory from output memory.
- Know when iterative traversal avoids an unsafe call depth.

---

# 6. Step 4B: Trees

## 6.1 Terminology

A tree models a hierarchy.

```text
          A
        /   \
       B     C
      / \
     D   E
```

- Root: A.
- Parent of D: B.
- Children of B: D and E.
- Leaves: D, E, C.
- Depth: number of edges from root to node; A has depth 0.
- Height: longest downward path in edges; A has height 2.
- Subtree: a node and its descendants.

This guide uses edge-based height. Other materials use node count; check the convention.

A rooted tree has no cycles and, except for the root, one parent per node. Hierarchical-looking business data can violate these rules and must be validated.

## 6.2 Binary tree versus binary search tree

A **binary tree** has at most two children per node.

A **binary search tree**, or BST, additionally maintains an ordering rule. For a strict, duplicate-free BST:

```text
every key in left subtree < node key < every key in right subtree
```

The rule applies to entire subtrees, not just immediate children.

A binary tree is not automatically ordered. A heap is not a BST.

## 6.3 Depth-first traversals

- Preorder: node, left, right.
- Inorder: left, node, right.
- Postorder: left, right, node.

```csharp
var root = new TreeNode(4,
    new TreeNode(2, new TreeNode(1), new TreeNode(3)),
    new TreeNode(6));

Console.Write("Preorder: ");
Preorder(root);
Console.WriteLine();
Console.Write("Inorder: ");
Inorder(root);
Console.WriteLine();
Console.Write("Postorder: ");
Postorder(root);
Console.WriteLine();

// Preorder: 4 2 1 3 6
// Inorder: 1 2 3 4 6
// Postorder: 1 3 2 6 4

static void Preorder(TreeNode? node)
{
    if (node is null) return;
    Console.Write($"{node.Value} ");
    Preorder(node.Left);
    Preorder(node.Right);
}

static void Inorder(TreeNode? node)
{
    if (node is null) return;
    Inorder(node.Left);
    Console.Write($"{node.Value} ");
    Inorder(node.Right);
}

static void Postorder(TreeNode? node)
{
    if (node is null) return;
    Postorder(node.Left);
    Postorder(node.Right);
    Console.Write($"{node.Value} ");
}

sealed record TreeNode(int Value, TreeNode? Left = null, TreeNode? Right = null);
```

Each visits `n` nodes once: O(n) time, O(h) recursive stack space. A skewed tree can have height n-1.

Inorder yields sorted keys **only when the tree satisfies the BST ordering**.

Practical intuition:

- Preorder: handle a parent before descendants.
- Postorder: calculate child results before the parent, or delete descendants first.
- Inorder: ordered BST traversal.

## 6.4 Breadth-first traversal

Visit one level at a time using a queue.

```csharp
var root = new TreeNode(4,
    new TreeNode(2, new TreeNode(1), new TreeNode(3)),
    new TreeNode(6));

var queue = new Queue<TreeNode>();
queue.Enqueue(root);

while (queue.TryDequeue(out TreeNode? node))
{
    Console.Write($"{node.Value} ");
    if (node.Left is not null) queue.Enqueue(node.Left);
    if (node.Right is not null) queue.Enqueue(node.Right);
}

Console.WriteLine();
// 4 2 6 1 3

sealed record TreeNode(int Value, TreeNode? Left = null, TreeNode? Right = null);
```

Queue trace: `[4]` -> `[2,6]` -> `[6,1,3]` -> `[1,3]` -> `[3]` -> empty.

Time O(n), auxiliary space O(w), where `w` is the maximum level width up to constant factors.

Use cases: level grouping, nearest-by-edge hierarchy queries, shallow-first exploration.

## 6.5 Search and insertion in a BST

```csharp
BstNode? root = null;
foreach (int value in new[] { 4, 2, 6, 1, 3 })
    root = Insert(root, value);

Console.WriteLine(Contains(root, 3)); // True
Console.WriteLine(Contains(root, 5)); // False

static BstNode Insert(BstNode? root, int value)
{
    if (root is null)
        return new BstNode(value);

    BstNode current = root;
    while (true)
    {
        if (value == current.Value)
            return root;

        if (value < current.Value)
        {
            if (current.Left is null)
            {
                current.Left = new BstNode(value);
                return root;
            }
            current = current.Left;
        }
        else
        {
            if (current.Right is null)
            {
                current.Right = new BstNode(value);
                return root;
            }
            current = current.Right;
        }
    }
}

static bool Contains(BstNode? root, int value)
{
    BstNode? current = root;
    while (current is not null)
    {
        if (value == current.Value)
            return true;
        current = value < current.Value ? current.Left : current.Right;
    }
    return false;
}

sealed class BstNode
{
    public int Value { get; }
    public BstNode? Left { get; set; }
    public BstNode? Right { get; set; }
    public BstNode(int value) => Value = value;
}
```

This implementation treats duplicate insertion as an intentional no-op: it stores a set of values.

Search/insertion time O(h), auxiliary working space O(1) for this iterative version, plus new-node storage for insertion.

For a balanced tree, h is O(log n). Inserting sorted values into an unbalanced BST can produce a chain with h = O(n).

## 6.6 Balanced trees

AVL trees and red-black trees maintain height through rules and structural adjustments such as rotations.

You should understand:

- Why height controls operation cost.
- What a rotation changes and preserves.
- Why maintaining balance adds implementation complexity.
- Why ordered library collections are preferable to homemade balancing in normal application code.

Implementing all balancing cases is outside this guide's core scope.

## 6.7 Business application: categories and menus

A general category tree can have any number of children:

```text
Products
  Hardware
    Laptops
    Monitors
  Software
    Productivity
```

For flat records containing `Id` and `ParentId`, build a parent-to-children lookup once, then traverse.

Repeatedly filtering the entire flat list for every node can become quadratic. Building a lookup takes expected O(n), then traversal can be O(n).

Validate duplicate IDs, missing parents, cycles, and maximum depth. Permissions also need clear inheritance and conflict rules; tree traversal alone is not an authorization policy.

---

# 7. Step 4C: Graphs

## 7.1 Graph vocabulary

A graph consists of vertices and edges.

- Vertex/node: entity.
- Edge: relationship.
- Directed: A -> B differs from B -> A.
- Undirected: connection works both ways.
- Weighted: edges have costs.
- Cycle: path returning to an earlier vertex.
- Connected component: connected region in an undirected graph.
- DAG: directed acyclic graph.

Examples: dependency ordering, routes, social connections, workflow transitions, and package dependencies.

Trees are a restricted kind of graph. Graph algorithms usually need explicit protection against revisiting nodes.

## 7.2 Representations

### Adjacency list

```text
0: [1,2]
1: [3]
2: [3]
3: []
```

Storage O(V + E); efficient enumeration of actual neighbors.

### Adjacency matrix

```text
     0 1 2 3
0    0 1 1 0
1    0 0 0 1
2    0 0 0 1
3    0 0 0 0
```

Storage O(V squared); direct edge-existence checks are O(1), but scanning all neighbors of a vertex takes O(V).

An adjacency-list representation does not automatically give O(1) edge-existence checks if each neighbor collection is a list.

Choose based on graph density and operations, not habit.

## 7.3 BFS: shortest paths in an unweighted graph

The graph below uses vertices 0 through 4 and valid adjacency indices.

```csharp
int[] BfsDistances(List<int>[] graph, int start)
{
    if (start < 0 || start >= graph.Length)
        throw new ArgumentOutOfRangeException(nameof(start));

    var distance = Enumerable.Repeat(-1, graph.Length).ToArray();
    var queue = new Queue<int>();
    distance[start] = 0;
    queue.Enqueue(start);

    while (queue.TryDequeue(out int vertex))
    {
        foreach (int neighbor in graph[vertex])
        {
            if (distance[neighbor] != -1)
                continue;

            distance[neighbor] = distance[vertex] + 1;
            queue.Enqueue(neighbor);
        }
    }

    return distance;
}

List<int>[] graph =
{
    new() { 1, 2 },
    new() { 3 },
    new() { 3 },
    new(),
    new()
};

Console.WriteLine(string.Join(", ", BfsDistances(graph, 0)));
// 0, 1, 1, 2, -1
```

Vertex 4 is unreachable; `-1` is the method's explicit unreachable sentinel.

Mark nodes visited **when enqueuing**, not only when dequeuing, to avoid multiple enqueues from different parents.

Why shortest paths: FIFO processing visits distance-0 nodes first, then distance-1 nodes, and so forth. First discovery is through a shortest edge-count path.

Time O(V + E) including initialization; auxiliary space O(V). This is not a general cheapest-weighted-path algorithm.

To reconstruct a route, store `parent[neighbor] = vertex` on first discovery, then walk backward from destination and reverse.

## 7.4 Iterative DFS

```csharp
List<int> Dfs(List<int>[] graph, int start)
{
    if (start < 0 || start >= graph.Length)
        throw new ArgumentOutOfRangeException(nameof(start));

    var visited = new bool[graph.Length];
    var stack = new Stack<int>();
    var result = new List<int>();

    visited[start] = true;
    stack.Push(start);

    while (stack.TryPop(out int vertex))
    {
        result.Add(vertex);
        foreach (int neighbor in graph[vertex])
        {
            if (!visited[neighbor])
            {
                visited[neighbor] = true;
                stack.Push(neighbor);
            }
        }
    }

    return result;
}

List<int>[] graph =
{
    new() { 1, 2 },
    new() { 3 },
    new() { 3 },
    new()
};

Console.WriteLine(string.Join(", ", Dfs(graph, 0))); // 0, 2, 3, 1
```

Neighbor order and the LIFO stack determine visitation order. This iterative reachability traversal need not produce the same DFS parent tree as a recursive implementation that descends immediately.

Time O(V + E), auxiliary space O(V), output O(V).

For disconnected graphs, loop over every vertex and start a traversal at each still-unvisited vertex.

## 7.5 Cycle detection

For a directed graph, a visited set alone is insufficient to distinguish a cycle from two paths reaching the same completed node.

Use three states:

- White/unseen.
- Gray/active in current recursion path.
- Black/fully processed.

An edge to a gray node proves a directed cycle.

For an undirected graph, encountering a visited neighbor is not automatically a cycle: the edge back to the parent is expected. Track the parent, and handle parallel edges according to the graph model.

## 7.6 Topological sorting

Topological sorting orders vertices so every directed edge `u -> v` places `u` before `v`.

Interpretation here: an edge goes **from prerequisite to dependent**.

Kahn's algorithm:

1. Compute each node's indegree.
2. Enqueue nodes with indegree zero.
3. Remove a node and append it to the order.
4. Decrement its dependents' indegrees.
5. Enqueue dependents that reach zero.
6. If not all nodes are processed, a cycle prevents a full ordering.

```csharp
int[] TopologicalSort(List<int>[] graph)
{
    var indegree = new int[graph.Length];
    foreach (List<int> neighbors in graph)
        foreach (int neighbor in neighbors)
            indegree[neighbor]++;

    var ready = new Queue<int>();
    for (int vertex = 0; vertex < graph.Length; vertex++)
        if (indegree[vertex] == 0)
            ready.Enqueue(vertex);

    var order = new List<int>();
    while (ready.TryDequeue(out int vertex))
    {
        order.Add(vertex);
        foreach (int neighbor in graph[vertex])
        {
            indegree[neighbor]--;
            if (indegree[neighbor] == 0)
                ready.Enqueue(neighbor);
        }
    }

    if (order.Count != graph.Length)
        throw new InvalidOperationException("The dependency graph contains a cycle.");

    return order.ToArray();
}

List<int>[] graph =
{
    new() { 2 },
    new() { 2 },
    new() { 3 },
    new()
};

Console.WriteLine(string.Join(", ", TopologicalSort(graph))); // 0, 1, 2, 3
```

Initial ready nodes: 0 and 1. Node 2 becomes ready only after both are processed.

Time O(V + E); auxiliary space O(V), excluding output.

There may be multiple valid orders. Tests should verify prerequisite relationships rather than require one arbitrary ordering unless deterministic tie-breaking is part of the contract.

## 7.7 Dijkstra's algorithm

Dijkstra finds shortest weighted distances from one source when **all edge weights are nonnegative**. See S7.

Core terms:

- `distance[v]`: best known cost to vertex `v`.
- **Relax an edge:** improve the destination distance if going through the current vertex is cheaper.
- Priority queue: process the currently smallest tentative distance first.

The example uses duplicate queue entries instead of decrease-key. Stale entries are discarded.

```csharp
long[] Dijkstra(List<(int To, int Weight)>[] graph, int start)
{
    if (start < 0 || start >= graph.Length)
        throw new ArgumentOutOfRangeException(nameof(start));

    foreach (var edges in graph)
    {
        foreach (var edge in edges)
        {
            if (edge.To < 0 || edge.To >= graph.Length)
                throw new ArgumentException("Invalid destination vertex.", nameof(graph));
            if (edge.Weight < 0)
                throw new ArgumentException("Dijkstra requires nonnegative weights.", nameof(graph));
        }
    }

    var distance = Enumerable.Repeat(long.MaxValue, graph.Length).ToArray();
    var frontier = new PriorityQueue<int, long>();
    distance[start] = 0;
    frontier.Enqueue(start, 0);

    while (frontier.TryDequeue(out int vertex, out long queuedDistance))
    {
        if (queuedDistance != distance[vertex])
            continue;

        foreach (var edge in graph[vertex])
        {
            long candidate = checked(queuedDistance + edge.Weight);
            if (candidate >= distance[edge.To])
                continue;

            distance[edge.To] = candidate;
            frontier.Enqueue(edge.To, candidate);
        }
    }

    return distance;
}

List<(int To, int Weight)>[] graph =
{
    new() { (1, 4), (2, 1) },
    new() { (3, 1) },
    new() { (1, 2), (3, 5) },
    new()
};

Console.WriteLine(string.Join(", ", Dijkstra(graph, 0))); // 0, 3, 1, 4
```

Trace:

1. Start: distance 0 to vertex 0.
2. From 0: tentative distance 4 to 1, distance 1 to 2.
3. Process 2: improve 1 to 3; set 3 to 6.
4. Process 1 at distance 3: improve 3 to 4.
5. Old queue entries for 1 at 4 and 3 at 6 become stale.

`long.MaxValue` means unreachable. The example assumes representable finite path costs stay below that sentinel.

With duplicate entries, the queue can contain O(E) items, not merely O(V). A conservative bound for this implementation is O(V + E log(E + 1)) time and O(V + E) auxiliary space. For simple graphs, the common O((V + E) log V) presentation is compatible up to the usual relationships between E and V, apart from degenerate conventions.

An indexed heap with decrease-key has a different storage bound. Do not copy a complexity claim from a different implementation without checking.

Use BFS for unweighted shortest paths. Negative weights require a different algorithm; Bellman-Ford is a next topic, not a silent extension of Dijkstra.

---

# 8. Step 4D: Dynamic programming

## 8.1 What dynamic programming means

Dynamic programming, or DP, solves related subproblems and reuses their results.

Two recurring properties:

- **Overlapping subproblems:** the same smaller question is asked repeatedly.
- **Optimal substructure:** an optimal answer can be constructed from suitable optimal subproblem answers.

DP is not simply "use a dictionary." The important work is defining a correct state and transition.

## 8.2 The five questions

Before coding:

1. What exactly does `dp[state]` mean?
2. Which earlier states determine it?
3. What are the base cases?
4. In which order can states be evaluated?
5. Where is the final answer, and do I need to reconstruct choices?

If you cannot answer the first question precisely, writing an array usually makes the confusion harder to see.

## 8.3 Climbing stairs

Problem: count ordered ways to climb `n` stairs using steps of size 1 or 2.

State: `ways(i)` counts ways to reach stair `i`.

Transition:

```text
ways(i) = ways(i - 1) + ways(i - 2)
```

The final move is either one step from `i-1` or two steps from `i-2`; these cases do not overlap.

Base cases: `ways(0)=1`, `ways(1)=1`. The empty sequence is one way to climb zero stairs.

### Top-down memoization

```csharp
long CountWaysMemo(int n)
{
    if (n < 0 || n > 91)
        throw new ArgumentOutOfRangeException(nameof(n), "Use a value from 0 through 91.");

    var memo = new long?[n + 1];
    return Solve(n);

    long Solve(int remaining)
    {
        if (remaining <= 1)
            return 1;

        if (memo[remaining] is long cached)
            return cached;

        long result = checked(Solve(remaining - 1) + Solve(remaining - 2));
        memo[remaining] = result;
        return result;
    }
}

Console.WriteLine(CountWaysMemo(5)); // 8
```

Time O(n), memo O(n), recursive stack O(n).

Without memoization, repeated calls expand exponentially. The solution still has only O(n) distinct states.

### Bottom-up tabulation and space optimization

```csharp
long CountWaysBottomUp(int n)
{
    if (n < 0 || n > 91)
        throw new ArgumentOutOfRangeException(nameof(n), "Use a value from 0 through 91.");

    long twoBack = 1;
    long oneBack = 1;

    for (int stair = 2; stair <= n; stair++)
    {
        long current = checked(oneBack + twoBack);
        twoBack = oneBack;
        oneBack = current;
    }

    return oneBack;
}

Console.WriteLine(CountWaysBottomUp(5)); // 8
Console.WriteLine(CountWaysBottomUp(0)); // 1
```

Trace: ways for stairs 0..5 are `1,1,2,3,5,8`.

Time O(n), auxiliary space O(1). We retain only the two values needed by the next transition.

The upper input bound is a numeric-representation limit, not an algorithmic principle. Larger answers need a suitable type, such as `BigInteger`, whose arithmetic cost is not constant with growing number length.

## 8.4 Coin change: minimum number of coins

Problem: unlimited copies of each positive denomination; find the minimum coins for a nonnegative amount.

State: `dp[a]` is the minimum count to form amount `a`, or infinity if unreachable.

Transition:

```text
dp[a] = min(dp[a - coin] + 1) over usable coins
```

```csharp
int MinimumCoins(int[] coins, int amount)
{
    if (amount < 0 || amount == int.MaxValue)
        throw new ArgumentOutOfRangeException(nameof(amount));
    if (coins.Any(coin => coin <= 0))
        throw new ArgumentException("Coin values must be positive.", nameof(coins));

    var dp = Enumerable.Repeat(int.MaxValue, amount + 1).ToArray();
    dp[0] = 0;

    for (int current = 1; current <= amount; current++)
    {
        foreach (int coin in coins)
        {
            if (coin <= current && dp[current - coin] != int.MaxValue)
                dp[current] = Math.Min(dp[current], dp[current - coin] + 1);
        }
    }

    return dp[amount] == int.MaxValue ? -1 : dp[amount];
}

Console.WriteLine(MinimumCoins(new[] { 1, 3, 4 }, 6)); // 2
Console.WriteLine(MinimumCoins(new[] { 2 }, 3));        // -1
Console.WriteLine(MinimumCoins(Array.Empty<int>(), 0)); // 0
```

For coins 1,3,4:

| Amount | Minimum |
|---:|---:|
| 0 | 0 |
| 1 | 1 |
| 2 | 2 |
| 3 | 1 |
| 4 | 1 |
| 5 | 2 |
| 6 | 2 |

Time O(amount * numberOfCoins), auxiliary space O(amount).

This is **pseudo-polynomial**: complexity depends on the numeric amount, not just the number of bits needed to represent it. An amount of a billion is not practical just because it fits in an integer. Set a resource budget in a real API.

To reconstruct the coins, record which denomination improved each state, then walk backward from the target.

## 8.5 Common DP mistakes

- A state omits information that affects future choices.
- A base case assumes the answer rather than proving it.
- A sentinel is incremented and overflows.
- Memoization is recreated for every recursive call.
- State is shared across independent requests accidentally.
- Loop order changes whether items can be reused.
- Counting combinations is confused with counting ordered sequences.
- A large numeric state space is treated as "small input."

For example, counting coin combinations and counting ordered coin sequences are different problems. Their iteration order and recurrence interpretation differ.

## 8.6 When DP is unnecessary

Do not use DP for a straightforward dictionary lookup, one-pass maximum, or a problem already solved by a valid greedy rule.

Start with a simple correct solution, identify repeated subproblems, and only then consider memoization/tabulation.

---

# 9. Step 5: Applying DSA to .NET and React

## 9.1 Refactoring repeated customer scans

Problem: create order summaries with customer names.

Business contract for the example:

- Customer IDs are unique.
- Missing customers display `"Unknown"`.
- Order sequence is preserved.
- Inputs are already in memory.

```csharp
var customers = new[]
{
    new Customer(10, "Alice"),
    new Customer(20, "Ben")
};

var orders = new[]
{
    new Order(100, 20),
    new Order(101, 10),
    new Order(102, 99)
};

var original = orders.Select(order =>
{
    Customer? customer = customers.FirstOrDefault(c => c.Id == order.CustomerId);
    return new Summary(order.Id, customer?.Name ?? "Unknown");
}).ToArray();

var customerById = customers.ToDictionary(c => c.Id);
var improved = orders.Select(order =>
{
    string name = customerById.TryGetValue(order.CustomerId, out Customer? customer)
        ? customer.Name
        : "Unknown";
    return new Summary(order.Id, name);
}).ToArray();

Console.WriteLine(original.SequenceEqual(improved)); // True
foreach (Summary summary in improved)
    Console.WriteLine($"{summary.OrderId}: {summary.CustomerName}");

// 100: Ben
// 101: Alice
// 102: Unknown

sealed record Customer(int Id, string Name);
sealed record Order(int Id, int CustomerId);
sealed record Summary(int OrderId, string CustomerName);
```

Original: up to O(nm), where n orders and m customers.

Improved: expected O(m) index construction + O(n) lookups, total O(n+m); auxiliary space O(m), output O(n).

### Preserve behavior, not just speed

If duplicate customer IDs are possible:

- `FirstOrDefault` picks the first matching customer.
- `ToDictionary` throws on duplicate keys.

Those are not equivalent behaviors. Validate uniqueness, explicitly preserve a first-match rule if intended, or choose a multivalue lookup. Do not silently pick a different record.

The `"Unknown"` value here is a defined display rule, not permission to hide corrupted required relationships in production.

## 9.2 Repeated membership checks

Suppose each of `n` products checks membership in `m` selected IDs:

```text
products.Where(p => selectedIdsList.Contains(p.Id))
```

Worst-case repeated scanning can be O(nm).

Build a set once:

```text
selectedIds = new HashSet<int>(selectedIdsList)
products.Where(p => selectedIds.Contains(p.Id))
```

Expected O(n+m), with O(m) extra memory.

But if there are only three selected IDs and one tiny list, the simpler list may already be sufficient. Evaluate the workload.

## 9.3 One-to-many lookups

For many orders per customer, the key maps to multiple values:

```csharp
var orders = new[]
{
    (Id: 1, CustomerId: 10),
    (Id: 2, CustomerId: 20),
    (Id: 3, CustomerId: 10)
};

ILookup<int, (int Id, int CustomerId)> ordersByCustomer =
    orders.ToLookup(order => order.CustomerId);

Console.WriteLine(string.Join(", ", ordersByCustomer[10].Select(o => o.Id))); // 1, 3
Console.WriteLine(ordersByCustomer[99].Count()); // 0
```

Use `ToLookup` or an appropriate grouped dictionary when duplicates are part of the model.

Building this lookup takes expected O(n) time and O(n) storage. A missing group is empty.

Producing all matching pairs in a many-to-many join can itself require large output. No index can eliminate the cost of returning required result rows.

## 9.4 LINQ: concise syntax does not remove work

For typical LINQ-to-Objects traversal:

| Expression | Important consideration |
|---|---|
| `Where` / `Select` | Usually deferred; enumeration performs predicate/projection work |
| `FirstOrDefault(predicate)` | Can scan until the first match; O(n) worst case |
| `Any(predicate)` | Stops once a match is found |
| `Count(predicate)` | Generally evaluates all candidates |
| `Count()` | May exploit a collection count; not always a full scan |
| `OrderBy` | Deferred but buffers/sorts when enumerated |
| `ToArray` / `ToList` | Enumerate and materialize now |
| `ToDictionary` / `ToLookup` | Build an in-memory index now |
| `SelectMany` | Cost depends on total inner output, not only outer count |

Implementation optimizations can vary by runtime and input type. Understand semantics before asserting exact internal operation counts. See S6.

### Deferred execution and repeated enumeration

```csharp
IEnumerable<int> EvenValues(IEnumerable<int> source)
{
    foreach (int value in source)
    {
        Console.WriteLine($"Inspect {value}");
        if (value % 2 == 0)
            yield return value;
    }
}

IEnumerable<int> query = EvenValues(new[] { 1, 2, 3, 4 });
Console.WriteLine("Created");
Console.WriteLine(query.Count());
Console.WriteLine(query.Sum());

// Created
// Inspect 1
// Inspect 2
// Inspect 3
// Inspect 4
// 2
// Inspect 1
// Inspect 2
// Inspect 3
// Inspect 4
// 6
```

The iterator runs twice because there are two enumerations.

Possible responses:

- Materialize once if repeated use and snapshot semantics are desired.
- Compute count and sum in one pass.
- Keep deferred execution if avoiding storage is more important.

Materialization changes timing and memory usage. If the source changes, a new enumeration can also produce different results.

## 9.5 `IEnumerable<T>` versus `IQueryable<T>`

`IEnumerable<T>` represents something enumerable. It does **not** guarantee that the source is fully in memory or cheap to enumerate.

`IQueryable<T>` supports an expression-tree query model that a provider can interpret, such as an EF Core provider translating supported expressions to SQL.

Context-dependent EF Core illustration:

```text
// Requires an application DbContext, entity model, and EF Core provider.
var rows = await db.Orders
    .Where(o => o.Status == "Open")
    .OrderBy(o => o.Id)
    .Select(o => new { o.Id, o.CustomerId })
    .Take(100)
    .ToListAsync();
```

Filtering and projection remain provider-translated when supported.

Contrast:

```text
var all = await db.Orders.ToListAsync();
var selected = all.Where(o => o.Status == "Open").Take(100);
```

This materializes all orders before the in-memory filter.

`AsEnumerable()` does not immediately fetch all rows, but subsequent LINQ operators bind to enumerable processing rather than provider translation. `AsQueryable()` over an existing list does not turn that list into a database query.

Do not optimize a database query by automatically loading the entire table into a dictionary. Inspect generated SQL, indexes, row counts, roundtrips, and the execution plan. See S8.

## 9.6 Database performance and DSA are related, not interchangeable

Useful distinctions:

- In-memory dictionary: helps repeated lookups in already-loaded bounded data.
- Database index: helps the database access suitable rows under its query plan.
- Batching: reduces repeated remote calls.
- Projection: avoids loading unused columns.
- Pagination: limits result size.
- Cache: avoids repeated computation/fetches but introduces invalidation, freshness, and memory concerns.

An O(n) loop making n remote requests can be much slower than an O(n log n) in-memory operation.

The N+1 query problem is often primarily a roundtrip/query-shape issue. Building a hash set after the requests finish does not fix it.

### Pagination

Offset pagination:

```text
OrderBy(Id).Skip(offset).Take(pageSize)
```

Large offsets may require substantial work.

Keyset pagination:

```text
Where(Id > lastSeenId).OrderBy(Id).Take(pageSize)
```

This can fit sequential navigation with an appropriate index and unique ordering. Compound sort orders need compound cursor logic. Concurrent updates and sort-key changes still require a defined consistency contract.

Neither approach should promise a stable snapshot unless the application actually provides one.

## 9.7 React: normalized data and lookup cost

Repeatedly searching an array for each displayed ID can multiply work.

```javascript
const users = [
  { id: 1, name: "Alice" },
  { id: 2, name: "Ben" }
];
const visibleIds = [2, 1];

const usersById = new Map(users.map(user => [user.id, user]));
const names = visibleIds.map(id => usersById.get(id)?.name ?? "Unknown");

console.log(names); // ["Ben", "Alice"]
```

Building the map has an initial cost. JavaScript `Map` implementations are required to provide average access that is sublinear in collection size; do not treat a particular hash-table implementation as a universal language guarantee.

For an immutable React data source, a component might memoize derived indexing:

```javascript
// Inside a React component; useMemo must be imported from React.
const usersById = useMemo(
  () => new Map(users.map(user => [user.id, user])),
  [users]
);
```

This assumes `users` changes reference when its data changes. Mutating the same array can make derived data stale and violates the intended state-update pattern.

`useMemo` is a performance optimization, not a correctness mechanism or a guarantee that a value is never recomputed. See S9 and S10.

## 9.8 Immutable updates have costs

```javascript
const updatedUsers = users.map(user =>
  user.id === targetId ? { ...user, name: newName } : user
);
```

This traverses the array and creates a new array: O(n).

Likewise:

```javascript
const nextSelection = new Set(previousSelection);
nextSelection.add(id);
```

The final insertion may be cheap, but copying the old set is still proportional to its size.

For an object-based normalized store:

```javascript
const nextById = {
  ...previousById,
  [id]: { ...previousById[id], name: newName }
};
```

Copying the container's properties also has a cost. Normalization improves access patterns and consistency; it does not make all immutable updates O(1).

Avoid storing duplicated derived data without a clear synchronization strategy.

## 9.9 Lookup improvements do not eliminate rendering cost

A map can speed up finding a record, but the UI may still:

- Filter a large list.
- Reconcile many elements.
- Render expensive child components.
- Perform layout and paint.
- Transfer or parse too much data.

Consider measuring first, then pagination, list virtualization, sensible memoization, and stable keys where appropriate.

React keys identify siblings across renders; they are not a substitute for an application lookup index.

## 9.10 Production optimization checklist

Before merging a "faster" implementation:

- Are duplicate and missing-key behaviors unchanged?
- Is output ordering preserved?
- Are comparison/case rules the same?
- Is the index rebuilt or updated when source data changes?
- Is the additional memory acceptable?
- Is the workload frequent or large enough to justify the change?
- Are database filtering and batching handled at the right boundary?
- Do concurrent operations require synchronization?
- Are invalid states visible rather than silently hidden?

---

# 10. Testing correctness and measuring performance

## 10.1 Boundary tests

For most sequence algorithms:

| Input category | Why it matters |
|---|---|
| Empty | Initialization and boundary safety |
| One element | Loop termination and same-element reuse |
| Duplicates | Equality and uniqueness contracts |
| All equal | Partitioning and duplicate handling |
| All negative | Incorrect zero initialization |
| Already sorted | Best/worst behavior assumptions |
| Reverse sorted | Shifts and pivot quality |
| Missing target | No-result contract |
| First/last target | Boundary handling |
| Numeric extremes | Overflow and comparer bugs |

For graphs: isolated vertices, disconnected components, cycles, multiple valid orders, zero-weight edges, and unreachable destinations.

For trees: empty, one-node, balanced, skewed, and malformed cyclic input if accepting external references.

## 10.2 Example-based tests versus property checks

Examples verify specific expected results.

Properties verify general relationships:

- Sorting preserves element multiplicities.
- Sorting produces nondecreasing output.
- Reversing twice restores a sequence.
- A valid returned pair uses different positions and sums to target.
- Every topological edge has its source before its destination.
- BFS distances increase by at most one across an outgoing reachable edge, with suitable reachability conditions.

A single property may not prove everything. A sorter returning an empty array is sorted but fails preservation.

## 10.3 Compare with a simple reference implementation

For small random inputs:

1. Run a simple brute-force solution.
2. Run the optimized solution.
3. Compare the required semantics.

For maximum window sum, enumerate every legal window for the reference. For pair sum, enumerate every pair of distinct indices.

Keep random seeds fixed when reproducing a failure. Test enough variety, not merely a large number of similar inputs.

## 10.4 A simple assertion without a test package

```csharp
void AssertEqual<T>(T expected, T actual, string description)
{
    if (!EqualityComparer<T>.Default.Equals(expected, actual))
        throw new InvalidOperationException(
            $"{description}: expected {expected}, actual {actual}");
}

AssertEqual(9L, new[] { 5, 1, 3 }.Sum(x => (long)x), "Total");
Console.WriteLine("Assertion passed.");
```

For production repositories, prefer the existing test framework rather than adding a custom assertion system.

## 10.5 Measurement basics

Separate:

- Index construction cost.
- Query-only cost.
- End-to-end cost.
- Memory allocation.
- Throughput and latency under actual application conditions.

A benchmark that builds a dictionary once but charges the competing method for all setup is answering a different question from a benchmark that includes both setups.

### An illustrative lookup comparison

```csharp
using System.Diagnostics;

int[] values = Enumerable.Range(0, 20_000).ToArray();
int[] queries = Enumerable.Range(10_000, 2_000).ToArray();

long Scan()
{
    long checksum = 0;
    foreach (int query in queries)
        if (Array.IndexOf(values, query) >= 0)
            checksum += query;
    return checksum;
}

long BuildAndLookup()
{
    var set = new HashSet<int>(values);
    long checksum = 0;
    foreach (int query in queries)
        if (set.Contains(query))
            checksum += query;
    return checksum;
}

_ = Scan();
_ = BuildAndLookup();

var watch = Stopwatch.StartNew();
long first = Scan();
watch.Stop();
Console.WriteLine($"Scan: {watch.Elapsed.TotalMilliseconds:F3} ms; checksum {first}");

watch.Restart();
long second = BuildAndLookup();
watch.Stop();
Console.WriteLine($"Build + lookup: {watch.Elapsed.TotalMilliseconds:F3} ms; checksum {second}");

if (first != second)
    throw new InvalidOperationException("The methods produced different results.");
```

The checksums are both 21,999,000. Timing values depend on your environment and should not be copied as performance promises.

Run release builds:

```powershell
dotnet run --configuration Release
```

This demonstration is not a rigorous benchmark: one measurement, runtime warmup effects, GC, CPU state, and execution order can distort it.

Use a dedicated benchmarking tool such as BenchmarkDotNet for serious microbenchmarks, and application profiling for real bottlenecks. Its official documentation is reference S11.

## 10.6 What to explain in a performance review

Use this format:

```text
Problem:
For each of n orders, the code scans up to m customers.

Change:
Build one customer index, then perform one lookup per order.

Complexity:
Worst-case repeated scans O(nm) -> expected O(n+m).

Tradeoff:
O(m) additional memory and index construction.

Behavior:
Unique IDs required; missing-customer behavior and order preserved.

Evidence:
Equivalent outputs on edge cases; representative workload measurements.
```

---

# 11. Guided exercises, hints, and solutions

Do not open a solution immediately. Write a contract and at least one example first.

## 11.1 Exercise A: first repeated value

Return the value whose repeated occurrence is encountered first while scanning left to right.

```text
[2,5,3,5,2] -> 5
[] -> no result
[1,2,3] -> no result
```

This is not "the earliest-positioned value that eventually repeats," which would be 2 in the first example.

<details>
<summary>Hint</summary>

Keep a set of values seen earlier. The first failed insertion identifies the answer.

</details>

<details>
<summary>Worked solution</summary>

```csharp
int? FirstRepeated(int[] values)
{
    var seen = new HashSet<int>();
    foreach (int value in values)
        if (!seen.Add(value))
            return value;
    return null;
}

Console.WriteLine(FirstRepeated(new[] { 2, 5, 3, 5, 2 })); // 5
Console.WriteLine(FirstRepeated(new[] { 1, 2, 3 }) is null); // True
```

Expected O(n) time, O(n) auxiliary space.

</details>

## 11.2 Exercise B: two sum with original indices

Input is unsorted. Return two different original indices with a target sum, or null.

```text
[3,3], target 6 -> (0,1)
[2,7,11,15], target 9 -> (0,1)
[5], target 10 -> no result
```

<details>
<summary>Hint</summary>

For the current value, search for its complement among earlier positions. Insert the current value only after the search.

</details>

<details>
<summary>Worked solution</summary>

```csharp
(int First, int Second)? TwoSum(int[] values, int target)
{
    var previousIndex = new Dictionary<long, int>();

    for (int i = 0; i < values.Length; i++)
    {
        long complement = (long)target - values[i];
        if (previousIndex.TryGetValue(complement, out int earlier))
            return (earlier, i);

        previousIndex.TryAdd(values[i], i);
    }

    return null;
}

Console.WriteLine(TwoSum(new[] { 3, 3 }, 6));       // (0, 1)
Console.WriteLine(TwoSum(new[] { 5 }, 10) is null); // True
```

Looking up before inserting prevents using the current element twice. A `long` complement avoids overflowing an `int` subtraction.

Expected O(n) time, O(n) auxiliary space.

</details>

## 11.3 Exercise C: compact sorted duplicates

Modify a sorted array so its prefix contains each distinct value once. Return the prefix length.

```text
[1,1,2,2,3] -> length 3, prefix [1,2,3]
[] -> length 0
```

Values beyond the returned length have no specified meaning.

<details>
<summary>Hint</summary>

Use a read pointer and a write pointer. Compare each candidate with the last accepted value.

</details>

<details>
<summary>Worked solution</summary>

```csharp
int CompactDistinct(int[] sorted)
{
    if (sorted.Length == 0)
        return 0;

    int write = 1;
    for (int read = 1; read < sorted.Length; read++)
    {
        if (sorted[read] != sorted[write - 1])
            sorted[write++] = sorted[read];
    }

    return write;
}

int[] values = { 1, 1, 2, 2, 3 };
int length = CompactDistinct(values);
Console.WriteLine(length); // 3
Console.WriteLine(string.Join(", ", values.Take(length))); // 1, 2, 3
```

Invariant: the prefix before `write` contains the distinct values from the processed input, in sorted order.

Time O(n), auxiliary space O(1).

</details>

## 11.4 Exercise D: validate a BST

Return whether every node satisfies the strict, duplicate-free BST rule.

Counterexample to a local-only check:

```text
      10
     /  \
    5    15
        /
       6
```

6 is smaller than its parent 15, but it violates the lower bound imposed by root 10.

<details>
<summary>Hint</summary>

Pass allowed lower and upper bounds down the recursion. Use a wider bound type than the stored integers.

</details>

<details>
<summary>Worked solution</summary>

```csharp
bool IsValidBst(TreeNode? root) => Check(root, long.MinValue, long.MaxValue);

bool Check(TreeNode? node, long lower, long upper)
{
    if (node is null)
        return true;
    if (node.Value <= lower || node.Value >= upper)
        return false;

    return Check(node.Left, lower, node.Value)
        && Check(node.Right, node.Value, upper);
}

var invalid = new TreeNode(10,
    new TreeNode(5),
    new TreeNode(15, new TreeNode(6)));

Console.WriteLine(IsValidBst(invalid)); // False
Console.WriteLine(IsValidBst(new TreeNode(int.MinValue))); // True

sealed record TreeNode(int Value, TreeNode? Left = null, TreeNode? Right = null);
```

Time O(n), stack O(h). Assumes a genuine finite tree, not cyclic references.

</details>

## 11.5 Exercise E: detect a directed cycle

Return true if any component contains a directed cycle.

<details>
<summary>Hint</summary>

Use unseen, active, and completed states. Search from all still-unseen vertices.

</details>

<details>
<summary>Worked solution</summary>

```csharp
bool HasDirectedCycle(List<int>[] graph)
{
    var state = new byte[graph.Length];
    for (int vertex = 0; vertex < graph.Length; vertex++)
        if (state[vertex] == 0 && Visit(vertex))
            return true;
    return false;

    bool Visit(int vertex)
    {
        state[vertex] = 1;
        foreach (int neighbor in graph[vertex])
        {
            if (state[neighbor] == 1)
                return true;
            if (state[neighbor] == 0 && Visit(neighbor))
                return true;
        }
        state[vertex] = 2;
        return false;
    }
}

List<int>[] cyclic =
{
    new() { 1 },
    new() { 2 },
    new() { 0 }
};

List<int>[] acyclic =
{
    new() { 1, 2 },
    new() { 2 },
    new()
};

Console.WriteLine(HasDirectedCycle(cyclic));  // True
Console.WriteLine(HasDirectedCycle(acyclic)); // False
```

Time O(V+E), auxiliary space O(V), including a worst-case recursive depth of V.

For very deep graphs, use an iterative state-machine traversal or Kahn's cycle test to avoid recursive stack limits.

</details>

## 11.6 Exercise F: anagrams

Determine whether two strings contain the same UTF-16 code units with the same counts.

Contract: ordinal, case-sensitive; spaces count; no normalization.

```text
"listen", "silent" -> true
"aab", "abb" -> false
"A", "a" -> false
```

<details>
<summary>Hint</summary>

Equal length is necessary. Increment counts for one input and decrement them for the other.

</details>

<details>
<summary>Worked solution</summary>

```csharp
bool AreAnagrams(string first, string second)
{
    if (first.Length != second.Length)
        return false;

    var counts = new Dictionary<char, int>();
    foreach (char value in first)
    {
        counts.TryGetValue(value, out int count);
        counts[value] = count + 1;
    }

    foreach (char value in second)
    {
        if (!counts.TryGetValue(value, out int count) || count == 0)
            return false;
        counts[value] = count - 1;
    }

    return true;
}

Console.WriteLine(AreAnagrams("listen", "silent")); // True
Console.WriteLine(AreAnagrams("aab", "abb"));       // False
Console.WriteLine(AreAnagrams("A", "a"));          // False
```

Expected O(n) time for equal lengths, O(u) auxiliary space for distinct code units.

Human-language anagrams require an explicit definition of characters, normalization, punctuation, and case. Do not silently expand this contract.

</details>

## 11.7 Additional exercises with reasoning targets

| Exercise | First approach | Improvement or important lesson |
|---|---|---|
| Count target occurrences in sorted data | Scan | Lower/upper bounds |
| Merge two sorted arrays | Concatenate and sort | Two pointers, O(n+m) |
| Move zeros to the end | Repeated deletion | Stable read/write compaction |
| Minimum-size subarray reaching a sum | Enumerate ranges | Positive-input window; inspect assumptions |
| Range sum queries | Sum each range | Prefix preprocessing |
| Reverse words | Split/reverse/join | Define whitespace preservation |
| Find kth largest | Sort | Bounded heap |
| Linked-list middle | Count then traverse | Fast/slow pointers |
| Linked-list cycle | Visited nodes | Floyd's fast/slow method |
| Tree height | Traverse | Recursive definition and height convention |
| Count graph components | Traversal from one node | Restart at every unvisited vertex |
| Shortest unweighted route | Enumerate paths | BFS plus parent tracking |
| Course prerequisites | Arbitrary ordering | Topological sort and cycle detection |
| Grid path count | Enumerate routes | DP state and obstacle rules |
| House robber | Try every subset | Include/exclude DP state |

For each, write: input contract, brute-force baseline, optimized invariant, complexity, and at least five edge tests.

---

# 12. Practice sequence and capstone projects

## 12.1 Curated progression

The order matters more than the number of problems solved.

| Stage | Problems to practice | Readiness condition |
|---|---|---|
| Arrays | Maximum, minimum, reverse, running totals | Can explain O(n) and boundaries |
| Hashing | Duplicates, frequencies, two sum, anagrams | Can distinguish set versus map |
| Stack/queue | Brackets, undo, FIFO simulation | Can choose correct processing order |
| Search | Binary search, lower bound, duplicate count | Can state and maintain boundaries |
| Pointers/windows | Sorted pair sum, compaction, fixed/variable windows | Can explain why pointer moves are safe |
| Sorting/intervals | Insertion, merge, custom ordering, merge ranges | Can state stability/mutation contract |
| Linked lists | Reverse, middle, cycle | Can trace references without losing nodes |
| Recursion | Factorial, tree traversal, subsets | Can identify base/progress/undo |
| Trees | Traversals, height, BST search, validation | Can reason about height and bounds |
| Graphs | BFS, DFS, components, cycles, dependencies | Can distinguish tree from general graph |
| Weighted paths | Dijkstra on tiny graphs | Can explain relaxation and constraints |
| DP | Stairs, coin change, grid paths, house robber | Can define state before coding |

Aim to solve a few representative problems per pattern and revisit mistakes. Solving many near-identical questions without explaining them can create false confidence.

## 12.2 Capstone A: order enrichment

Build a console application that:

- Accepts customers and orders.
- Enriches orders using indexed customer lookup.
- Reports duplicate customer IDs explicitly.
- Applies a documented missing-customer rule.
- Preserves order.
- Compares the output with a reference scan implementation for valid inputs.
- Measures build cost, lookup cost, and memory tradeoff.

Add tests for no customers, no orders, unknown IDs, duplicates, and large workloads.

Learning outcome: algorithms, business semantics, and performance are evaluated together.

## 12.3 Capstone B: dependency planner

Represent tasks and prerequisite edges.

Produce:

- A valid task execution order, if one exists.
- An explicit cycle failure otherwise.
- All isolated tasks too.
- Stable tie-breaking if required by the interface.

Optional extension: output ready-to-run groups rather than a single sequence. Do not assume this alone solves distributed scheduling, failure retries, or resource allocation.

Learning outcome: graph representation, validation, topological order, and realistic scope boundaries.

## 12.4 Capstone C: React selection explorer

Create a small UI containing a list of records and selected IDs.

Compare:

- Array-based membership and map/set-based lookup.
- Derived indexing recreated every render versus memoized from immutable inputs.
- Full rendering versus a bounded visible subset.

Preserve selection order if that is a requirement. Avoid mutating React state collections directly.

Learning outcome: data access and rendering are different performance layers.

---

# 13. A flexible 12-week learning schedule

This is a proposed schedule, not a promise that everyone reaches the same level in 12 weeks.

Suggested effort: 45-60 minutes on five days per week. If you have only 30 minutes, extend the schedule rather than rushing.

Daily template:

```text
10 minutes: recall yesterday's ideas without notes
15 minutes: learn/trace one concept
20 minutes: implement or solve
10 minutes: test and explain complexity
 5 minutes: record one mistake and one takeaway
```

| Week | Focus | Deliverable |
|---|---|---|
| 1 | Contracts, invariants, complexity | Analyze ten small snippets and implement maximum/search |
| 2 | Arrays, lists, strings | Explain capacity, copying, Unicode assumptions; solve traversal problems |
| 3 | Dictionary, set, frequency maps | Implement duplicates, two sum, anagrams, customer indexing |
| 4 | Stack, queue, linked list | Brackets, queue simulation, list reversal; draw every reference update |
| 5 | Binary search, sorting | Binary search/lower bound; trace insertion and merge sort |
| 6 | Two pointers, windows, prefix sums | Solve one example of each with stated assumptions |
| 7 | Intervals, greedy, recursion | Merge ranges, explain greedy counterexample, trace recursive calls |
| 8 | Backtracking and trees | Subsets, traversals, BST search/validation |
| 9 | Graph fundamentals | BFS, DFS, components, cycle detection |
| 10 | Dependencies and shortest paths | Topological sort and a hand-traced Dijkstra example |
| 11 | DP and practical refactoring | Stairs, coin change, LINQ enumeration, order-enrichment capstone |
| 12 | Review, capstone, interview explanation | Complete one capstone and solve mixed problems without pattern labels |

## 13.1 Weekly self-check

Score each from 0 to 2:

- Can I explain the idea without reading?
- Can I trace it on a small input?
- Can I implement it from a contract?
- Can I test boundaries?
- Can I explain time and space?
- Can I say when the technique does not apply?

A low score identifies what to revisit. It does not mean you should learn an advanced topic to compensate.

## 13.2 Interview practice after the foundation

Use a repeatable structure:

1. Clarify inputs, outputs, constraints, and allowed mutation.
2. Walk through an example.
3. Describe a simple correct baseline.
4. Identify the bottleneck.
5. Propose a structure or pattern with a correctness argument.
6. Implement with readable names.
7. Test boundaries aloud.
8. State time, auxiliary space, and output costs.

Start without strict timers. Add time limits after you can derive and explain solutions.

For senior .NET roles, continue studying system design, database queries, concurrency, reliability, and technical ownership. DSA supplements those skills.

---

# 14. Revision sheets and glossary

## 14.1 Collection selection cheat sheet

| Need | Start with | Important limitation |
|---|---|---|
| Fixed-size indexed storage | Array | Length does not grow |
| Flexible indexed sequence | `List<T>` | Middle/front updates shift elements |
| Repeated exact-key lookup | `Dictionary<TKey,TValue>` | Memory/build cost; equality rules |
| Membership/uniqueness | `HashSet<T>` | Not a sorted sequence |
| LIFO | `Stack<T>` | Empty access must be handled |
| FIFO | `Queue<T>` | In-memory and not durable |
| Insert/remove at known nodes | `LinkedList<T>` | Finding nodes may dominate |
| Repeated best-priority retrieval | `PriorityQueue<TElement,TPriority>` | Not fully sorted; ties need a rule |
| Ongoing sorted unique values | `SortedSet<T>` | Logarithmic updates |
| Ongoing sorted key-value mapping | `SortedDictionary<TKey,TValue>` | Different tradeoff from hashing |
| One key to many values | `ILookup<TKey,TElement>` / grouped mapping | Group/output size still matters |
| Concurrent mutation | Appropriate concurrent collection/channel | Compound workflows may still need coordination |

No single collection is universally best.

## 14.2 Complexity cheat sheet

| Algorithm/operation | Time | Auxiliary space |
|---|---|---|
| Array index access | O(1) | O(1) |
| Unsorted linear search | O(n) worst | O(1) |
| Iterative binary search | O(log n) | O(1) |
| List append | O(1) amortized; O(n) individual resize | Capacity storage |
| Hash lookup | Expected O(1); pathological O(n) | Table storage |
| Insertion sort | O(n squared) worst; O(n) best | O(1) |
| Merge sort shown | O(n log n) | O(n) |
| Quicksort shown | O(n log n) average; O(n squared) worst | O(log n) typical stack; O(n) worst |
| Fixed-size window | O(n) | O(1) |
| Prefix construction/query | O(n) / O(1) | O(n) preprocessing |
| Tree DFS | O(n) | O(h) |
| Tree BFS | O(n) | O(w) |
| Graph BFS/DFS, adjacency lists | O(V+E) | O(V), excluding graph/output |
| Topological sort | O(V+E) | O(V), excluding output |
| Lazy-entry Dijkstra shown | O(V + E log(E+1)) | O(V+E) |
| All subsets with copies | O(n * 2^n) | O(n), excluding output |
| Memoized stairs | O(n) | O(n) |
| Optimized bottom-up stairs | O(n) | O(1) |
| Minimum coin count | O(amount * coinCount) | O(amount) |

Complexities depend on contracts and implementations. A table is a reminder, not a substitute for reasoning.

## 14.3 Pattern recognition cheat sheet

| Clue | Candidate technique | Check before using |
|---|---|---|
| Repeated "have I seen this?" | Set | Equality and memory |
| Repeated "what value belongs to this key?" | Dictionary | Unique keys or multivalue model |
| Sorted data and target | Binary search | Same ordering, boundaries |
| Pair in sorted sequence | Two pointers | Pointer movement proof |
| Contiguous fixed-size range | Sliding window | Correct initial window |
| Longest/shortest valid contiguous range | Variable window | Monotonicity and shrink rule |
| Many static range totals | Prefix sums | Update frequency |
| Nested structure | Stack/recursion/tree traversal | Depth and cycles |
| Shortest number of edges | BFS | Unweighted/equal-cost edges |
| Prerequisite order | Topological sort | Edge direction and cycles |
| Nonnegative weighted shortest distance | Dijkstra | No negative edges |
| Enumerate valid arrangements | Backtracking | Output explosion and duplicate handling |
| Repeated smaller states | DP | Correct state and transition |
| Best few from a large stream | Bounded heap | Whether ties/order matter |

## 14.4 Common misconceptions to correct

**"Dictionary lookup is always O(1)."** Expected behavior has assumptions; collisions and key-processing costs matter.

**"Two loops means O(n squared)."** Sequential loops and bounded nested loops may be linear.

**"LINQ is slow."** Cost comes from operations, enumeration, provider translation, and allocation, not merely syntax.

**"Recursion is always elegant and safe."** Depth and stack space matter.

**"BFS finds every cheapest path."** It finds shortest edge-count paths in unweighted/equal-edge-cost graphs.

**"A BST guarantees logarithmic lookup."** Only when height is appropriately bounded.

**"A heap is sorted."** It guarantees a priority relationship, not complete ordering.

**"A hash set fixes all performance problems."** It addresses membership patterns, not network, database, or rendering costs.

**"The fastest code is the best code."** Correctness, clarity, memory, operational behavior, and maintainability also matter.

**"I need hundreds of hard problems before applying."** Preparation should match your target interview process, while retaining your existing engineering strengths.

## 14.5 Glossary

| Term | Meaning |
|---|---|
| ADT | Behavior and operations independent of storage |
| Algorithm | Defined steps for solving a class of problems |
| Amortized | Cost averaged over an operation sequence |
| Auxiliary space | Working memory beyond original input and usually output |
| Backtracking | Explore choices and undo temporary changes |
| Base case | Recursive case that terminates without further recursion |
| Big O | Asymptotic upper bound |
| Big Omega | Asymptotic lower bound |
| Big Theta | Asymptotically tight bound |
| Collision | Different keys share hash/bucket space |
| Comparator/comparer | Rule for ordering two values |
| DAG | Directed acyclic graph |
| Deferred execution | Work begins when results are evaluated/enumerated |
| DP | Reuse results of defined subproblems |
| Edge | Connection between graph vertices |
| Greedy | Make a locally preferred choice without revisiting it |
| Hash function | Maps a key to a hash used in lookup organization |
| Heap | Structure maintaining a parent-child priority property |
| Invariant | Statement preserved at a repeated program point |
| Materialization | Evaluate results into a stored collection |
| Memoization | Cache subproblem answers, usually top-down |
| Monotonic | Changes consistently in one direction under an ordering |
| Multiset | Collection model that tracks multiplicity |
| Prefix sum | Cumulative total enabling range subtraction |
| Pruning | Stop exploring a branch that cannot succeed |
| Relaxation | Improve a tentative path distance through an edge |
| Stable sort | Preserves original order among equal keys |
| Tabulation | Compute DP states in dependency order |
| Topological order | Vertex order respecting directed prerequisites |
| Vertex | Graph entity/node |
| Window | Contiguous input range tracked during traversal |

## 14.6 Final readiness checklist

You have a useful foundation when you can:

- Choose a collection and explain the tradeoff.
- Analyze repeated scans and hidden allocations.
- Write binary search without guessing boundaries.
- Explain why a window or pointer movement preserves correctness.
- Trace references and recursive calls.
- Traverse trees and graphs without confusing their assumptions.
- Recognize when BFS, topological sorting, or Dijkstra applies.
- Define a DP state clearly.
- Preserve semantics when optimizing existing code.
- Test edge cases and distinguish measurement from expectation.

You have an intermediate working level when you can combine these skills on an unfamiliar problem and explain why plausible alternatives are wrong or less suitable.

---

# 15. Official references

These are primary-source references for API semantics and selected algorithm details, checked while preparing this guide on October 19, 2026. The teaching examples and exercises are original. Documentation can evolve; select the version matching your project.

URLs are shown in code formatting so they can be copied from the file.

| ID | Reference | Relevant sections |
|---|---|---|
| S1 | Microsoft: Selecting a Collection Class | .NET collection selection and generic/concurrent options |
| S2 | Microsoft: `Dictionary<TKey,TValue>` | Dictionary keys, comparer, lookup behavior |
| S3 | Microsoft: `PriorityQueue<TElement,TPriority>` | Lowest-priority-first, quaternary heap, equal-priority caveat |
| S4 | Microsoft: Character encoding in .NET | UTF-16, `char`, `Rune`, grapheme clusters |
| S5 | Microsoft: `List<T>.Sort` | Sorting behavior, comparer, stability |
| S6 | Microsoft: Introduction to LINQ Queries | Deferred and immediate execution |
| S7 | Princeton Algorithms: Shortest Paths | Relaxation, nonnegative weights, Dijkstra variants |
| S8 | Microsoft: Efficient Querying in EF Core | Projection, indexes, pagination, N+1, buffering |
| S9 | React: Updating Arrays in State | Immutable state updates |
| S10 | React: `useMemo` | Dependencies, cache semantics, optimization caveats |
| S11 | BenchmarkDotNet official overview | Rigorous .NET microbenchmarking |
| S12 | ECMAScript: Map Objects | Required average access behavior |

S1: `https://learn.microsoft.com/en-us/dotnet/standard/collections/selecting-a-collection-class`

S2: `https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.dictionary-2`

S3: `https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.priorityqueue-2`

S4: `https://learn.microsoft.com/en-us/dotnet/standard/base-types/character-encoding-introduction`

S5: `https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.list-1.sort`

S6: `https://learn.microsoft.com/en-us/dotnet/csharp/linq/get-started/introduction-to-linq-queries`

S7: `https://algs4.cs.princeton.edu/44sp/index.php`

S8: `https://learn.microsoft.com/en-us/ef/core/performance/efficient-querying`

S9: `https://react.dev/learn/updating-arrays-in-state`

S10: `https://react.dev/reference/react/useMemo`

S11: `https://benchmarkdotnet.org/articles/overview.html`

S12: `https://tc39.es/ecma262/multipage/keyed-collections.html#sec-map-objects`

**Closing learning note:** the goal is not to memorize every algorithm. It is to build a reliable habit: define the problem, choose a suitable representation, justify the steps, test the boundaries, and measure the result when performance matters.
