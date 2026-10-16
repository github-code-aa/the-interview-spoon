# React Interview Guide — 02: Hooks

## Table of Contents
- [Q21. Why were hooks introduced?](#q21-why-were-hooks-introduced)
- [Q22. Rules of Hooks](#q22-what-are-the-rules-of-hooks-and-why-do-they-exist)
- [Q23. useState and functional updates](#q23-how-does-usestate-work-what-is-the-functional-update-form)
- [Q24. Async / batched state updates](#q24-why-is-state-updating-asynchronous-or-batched)
- [Q25. Why not mutate state](#q25-why-cant-you-mutate-state-directly)
- [Q26. useEffect and dependency array](#q26-how-does-useeffect-work-what-is-the-dependency-array)
- [Q27. Empty / no / changing array](#q27-what-happens-with-an-empty-array-no-array-or-a-changing-array)
- [Q28. Effect cleanup](#q28-how-do-you-clean-up-effects-subscriptions-timers-fetch)
- [Q29. Infinite loops in useEffect](#q29-what-causes-infinite-loops-in-useeffect)
- [Q30. Stale closure](#q30-what-is-a-stale-closure-give-an-example)
- [Q31. useEffect vs useLayoutEffect](#q31-useeffect-vs-uselayouteffect)
- [Q32. useRef](#q32-what-is-useref-which-two-use-cases-does-it-have)
- [Q33. useMemo](#q33-what-is-usememo-and-when-does-it-hurt)
- [Q34. useCallback vs useMemo](#q34-what-is-usecallback-and-how-does-it-differ-from-usememo)
- [Q35. useReducer](#q35-what-is-usereducer-when-is-it-better-than-usestate)
- [Q36. useContext](#q36-what-is-usecontext-what-are-its-re-render-implications)
- [Q37. Custom hooks](#q37-what-are-custom-hooks-write-usefetch-usedebounce-and-uselocalstorage)
- [Q38. useImperativeHandle and forwardRef](#q38-what-are-useimperativehandle-and-forwardref)
- [Q39. useId, useTransition, useDeferredValue](#q39-what-are-useid-usetransition-and-usedeferredvalue)
- [Q40. useSyncExternalStore](#q40-what-is-usesyncexternalstore)
- [Q41. useOptimistic, useActionState, use()](#q41-what-is-useoptimistic-what-are-useactionstate-and-use-in-react-19)
- [Q42. Strict Mode double effects](#q42-why-does-useeffect-run-twice-in-strict-mode-dev)

---

### Q21. Why were hooks introduced?

**Short answer**
Hooks (React 16.8) let function components use state, lifecycle behavior and other React features without classes. They solve hard-to-reuse stateful logic, giant lifecycle methods mixing unrelated concerns, and the confusion of `this` and classes.

**Detailed explanation**
- **Reusing stateful logic:** With classes you needed HOCs or render props, causing "wrapper hell". Custom hooks share logic by plain function composition, with no extra tree nodes.
- **Complex components:** Lifecycle methods split related code (subscribe in `componentDidMount`, unsubscribe in `componentWillUnmount`) and lump unrelated code together. Hooks organize code by concern.
- **Classes are confusing:** `this` binding, and classes are harder for tooling to optimize/minify and hot-reload.
- Classes still work; hooks are opt-in and not a breaking change.

**Code example**
```jsx
import { useState, useEffect } from "react";

function OnlineStatus() {
  const [online, setOnline] = useState(navigator.onLine);

  useEffect(() => {
    const update = () => setOnline(navigator.onLine);
    window.addEventListener("online", update);
    window.addEventListener("offline", update);
    return () => {
      window.removeEventListener("online", update);
      window.removeEventListener("offline", update);
    };
  }, []);

  return <p>{online ? "Online" : "Offline"}</p>;
}
```

**Common pitfalls / follow-up questions**
- *Are classes deprecated?* No, but error boundaries still require classes (no hook equivalent).
- *Do hooks replace Redux/HOCs?* Not entirely; they replace many HOC/render-prop uses, and `useReducer`+context covers simple state management.
- *Do hook-based components share state between instances?* No — each call gets its own state; only the logic is shared.

---

### Q22. What are the Rules of Hooks, and why do they exist?

**Short answer**
Only call hooks at the top level (not in loops, conditions, or nested functions), and only from React function components or custom hooks. React tracks hooks by call order, so the order must be identical on every render.

**Detailed explanation**
- React stores hook state per component fiber as a linked list indexed by call order, not by name. If a hook is skipped conditionally, later hooks read the wrong slot.
- Custom hooks must start with `use` so linters (`eslint-plugin-react-hooks`: `rules-of-hooks`, `exhaustive-deps`) can enforce the rules.
- Conditional logic goes *inside* a hook, not around it. Early returns before hooks are also violations.
- Exception: `use()` (React 19) can be called conditionally and in loops.

**Code example**
```jsx
import { useState, useEffect } from "react";

// ❌ Wrong
function Bad({ enabled }) {
  if (enabled) {
    const [count, setCount] = useState(0); // conditional hook
  }
  return null;
}

// ✅ Right
function Good({ enabled }) {
  const [count, setCount] = useState(0);
  useEffect(() => {
    if (!enabled) return;
    document.title = `Count ${count}`;
  }, [enabled, count]);
  return <button onClick={() => setCount((c) => c + 1)}>{count}</button>;
}
```

**Common pitfalls / follow-up questions**
- *Can I call a hook after an early return?* No; move the return below all hooks, or split into child components.
- *How to run a hook per list item?* Extract an item component that calls the hook.
- *Why must custom hooks start with `use`?* Convention enables lint enforcement and signals hook semantics.

---

### Q23. How does useState work? What is the functional update form?

**Short answer**
`useState(initial)` returns `[value, setValue]`; React stores the value across renders and re-renders when you set a new value (compared with `Object.is`). The functional form `setValue(prev => next)` computes the next state from the latest queued state, avoiding stale values.

**Detailed explanation**
- The initial value is used only on the first render. Pass a function (`useState(() => expensive())`) for lazy initialization.
- State is a snapshot: within one render, `count` is constant even after calling `setCount`.
- Use the functional form whenever the new state depends on the previous one, especially when calling the setter multiple times, in async callbacks, or intervals.
- If the new value is `Object.is`-equal to the current, React bails out of re-rendering.
- Updates replace state (no merging like class `setState`); spread objects manually.

**Code example**
```jsx
import { useState } from "react";

function Counter() {
  const [count, setCount] = useState(() => Number(localStorage.getItem("c") ?? 0));

  const addThree = () => {
    setCount(count + 1);       // all three use the same snapshot -> +1 total
    setCount(count + 1);
    setCount(count + 1);
  };
  const addThreeCorrect = () => {
    setCount((c) => c + 1);    // queued updaters -> +3 total
    setCount((c) => c + 1);
    setCount((c) => c + 1);
  };

  return (
    <>
      <p>{count}</p>
      <button onClick={addThree}>+3 (buggy)</button>
      <button onClick={addThreeCorrect}>+3</button>
    </>
  );
}
```

**Common pitfalls / follow-up questions**
- *Why does `console.log(count)` after `setCount` show the old value?* State is a snapshot for that render.
- *Is `useState(expensive())` bad?* It runs every render (result ignored); use `useState(() => expensive())`.
- *Setting the same value?* React may skip re-render (`Object.is`), so objects need new references to update.

---

### Q24. Why is state updating asynchronous or batched?

**Short answer**
Setters don't change the variable immediately; they schedule a re-render. React batches multiple updates in the same event into one render for performance and consistency. React 18 batches automatically everywhere (timeouts, promises, native handlers), not only in React events.

**Detailed explanation**
- Rendering is expensive; batching avoids intermediate inconsistent UI and wasted renders.
- Each render sees a fixed snapshot of state; updates apply on the next render.
- React 17 batched only inside React event handlers; React 18 (`createRoot`) batches everything.
- To force synchronous DOM flush (rare), use `flushSync` from `react-dom`.
- Updates are processed in order; updater functions are queued and applied sequentially.

**Code example**
```jsx
import { useState } from "react";
import { flushSync } from "react-dom";

function Demo() {
  const [a, setA] = useState(0);
  const [b, setB] = useState(0);

  console.log("render"); // once per click in React 18

  async function handle() {
    await Promise.resolve();
    setA((x) => x + 1);
    setB((x) => x + 1); // batched with setA in React 18
  }

  function forced() {
    flushSync(() => setA((x) => x + 1)); // DOM updated here
    flushSync(() => setB((x) => x + 1)); // second render
  }

  return (
    <>
      <p>{a} / {b}</p>
      <button onClick={handle}>Batched</button>
      <button onClick={forced}>flushSync</button>
    </>
  );
}
```

**Common pitfalls / follow-up questions**
- *How to read the updated value?* Compute it in a local variable, use an effect, or the functional form.
- *Does `setState` return a promise/callback?* No; use `useEffect` on the state.
- *Is it truly "async"?* Not literally; it is deferred/scheduled, and the code runs synchronously up to the render.

---

### Q25. Why can't you mutate state directly?

**Short answer**
React detects changes by reference comparison (`Object.is`). Mutating in place keeps the same reference, so React may skip re-rendering, and it breaks memoization, effect deps, time-travel and concurrent rendering. Always create new objects/arrays.

**Detailed explanation**
- `state.items.push(x); setState(state)` → same reference → no re-render.
- `React.memo`, `useMemo`, `useEffect` deps all rely on reference equality; mutation yields stale UI.
- Concurrent features and Strict Mode assume state is immutable; mutation causes tearing and double-invocation bugs.
- Use spread, `map`, `filter`, `slice`, `toSorted`, `toSpliced`, `structuredClone`, or Immer (`use-immer`) for deep updates.

**Code example**
```jsx
import { useState } from "react";

function Todos() {
  const [todos, setTodos] = useState([{ id: 1, text: "Learn hooks", done: false }]);

  const toggleBad = (id) => {
    const t = todos.find((x) => x.id === id);
    t.done = !t.done;       // ❌ mutation
    setTodos(todos);        // same reference, no re-render
  };

  const toggle = (id) =>
    setTodos((list) => list.map((t) => (t.id === id ? { ...t, done: !t.done } : t)));

  const add = (text) =>
    setTodos((list) => [...list, { id: crypto.randomUUID(), text, done: false }]);

  return (
    <ul>
      {todos.map((t) => (
        <li key={t.id} onClick={() => toggle(t.id)}>
          {t.done ? "✅" : "⬜"} {t.text}
        </li>
      ))}
      <button onClick={() => add("New")}>Add</button>
    </ul>
  );
}
```

**Common pitfalls / follow-up questions**
- *Is shallow copy enough for nested state?* Copy every level you change, or use Immer.
- *Mutating a local variable during render is fine?* Yes, if created during that render ("local mutation").
- *Why does `array.sort()` bite?* It mutates; use `[...arr].sort()` or `toSorted()`.

---

### Q26. How does useEffect work? What is the dependency array?

**Short answer**
`useEffect(setup, deps)` runs side-effect code after React commits and paints the render. The dependency array lists reactive values the effect uses; React re-runs the effect (after running the previous cleanup) only when one of them changed by `Object.is`.

**Detailed explanation**
- Effects synchronize a component with an external system (network, DOM APIs, timers, subscriptions) — not for deriving state from props/state.
- Flow: render → commit → paint → cleanup of previous effect (if deps changed) → run new effect.
- Every reactive value (props, state, values derived in the component) used inside must be in deps; `exhaustive-deps` lint enforces this.
- Setters, `useRef` objects and module-level values need not be listed.
- Effects capture values from the render they were created in.

**Code example**
```jsx
import { useState, useEffect } from "react";

function UserCard({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    const controller = new AbortController();
    fetch(`/api/users/${userId}`, { signal: controller.signal })
      .then((r) => r.json())
      .then(setUser)
      .catch((e) => { if (e.name !== "AbortError") console.error(e); });
    return () => controller.abort();
  }, [userId]);

  return <h2>{user?.name ?? "Loading..."}</h2>;
}
```

**Common pitfalls / follow-up questions**
- *Should I omit deps to silence lint?* No; fix by moving logic, using functional updates, or `useCallback`/`useEffectEvent`.
- *Do effects run on the server?* No, only on the client.
- *Do I need an effect for event-driven logic?* No, put it in the event handler.

---

### Q27. What happens with an empty array, no array, or a changing array?

**Short answer**
`[]` runs the effect once after mount (cleanup on unmount). No array runs after every render. An array with values runs after mount and whenever any value changes.

**Detailed explanation**
- **`[]`:** Mount-only. Any reactive value used is captured from the first render (risk of stale values). In dev Strict Mode it runs setup→cleanup→setup once.
- **Omitted:** After every render; cleanup runs before each subsequent run. Rarely appropriate (e.g., measuring every render).
- **`[a, b]`:** Re-runs if `a` or `b` changed (`Object.is`). Objects/arrays/functions created in render are new each time, so they always "change".
- Changing array *length* between renders produces a warning/error; the size must be constant.

**Code example**
```jsx
import { useEffect, useState } from "react";

function Demo({ id }) {
  const [n, setN] = useState(0);

  useEffect(() => { console.log("mount only"); return () => console.log("unmount"); }, []);
  useEffect(() => { console.log("every render"); });
  useEffect(() => { console.log("id changed:", id); }, [id]);

  // Always re-runs: new object each render
  const options = { id };
  useEffect(() => { console.log("options effect"); }, [options]);

  return <button onClick={() => setN(n + 1)}>{n}</button>;
}
```

**Common pitfalls / follow-up questions**
- *Fix object deps?* Depend on primitives, or memoize with `useMemo`, or create the object inside the effect.
- *Does `[]` mean componentDidMount?* Similar, but think "synchronization", not lifecycle.
- *Are refs allowed in deps?* They're stable; `ref.current` changes don't trigger effects.

---

### Q28. How do you clean up effects (subscriptions, timers, fetch)?

**Short answer**
Return a cleanup function from the effect. React calls it before re-running the effect and on unmount. Use it to unsubscribe, clear timers, remove listeners, and abort in-flight requests.

**Detailed explanation**
- Cleanup should mirror setup symmetrically (connect/disconnect, subscribe/unsubscribe).
- For fetch use `AbortController`; for non-abortable async work use an `ignore` flag to prevent setting state from stale responses (race conditions).
- Cleanup runs with the values from the render that created it.
- Prefer data libraries (TanStack Query, SWR) or Suspense for fetching in production.

**Code example**
```jsx
import { useEffect, useState } from "react";

function Clock() {
  const [now, setNow] = useState(() => new Date());
  useEffect(() => {
    const id = setInterval(() => setNow(new Date()), 1000);
    return () => clearInterval(id);
  }, []);
  return <time>{now.toLocaleTimeString()}</time>;
}

function Search({ query }) {
  const [results, setResults] = useState([]);
  useEffect(() => {
    const controller = new AbortController();
    let ignore = false;
    (async () => {
      try {
        const res = await fetch(`/api/search?q=${encodeURIComponent(query)}`, {
          signal: controller.signal,
        });
        const data = await res.json();
        if (!ignore) setResults(data);
      } catch (e) {
        if (e.name !== "AbortError") throw e;
      }
    })();
    return () => { ignore = true; controller.abort(); };
  }, [query]);

  return <ul>{results.map((r) => <li key={r.id}>{r.title}</li>)}</ul>;
}
```

**Common pitfalls / follow-up questions**
- *What bug appears without fetch cleanup?* Race conditions: slow earlier response overwrites newer result.
- *Is aborting an error?* It throws `AbortError`; catch and ignore it.
- *When is cleanup called?* Before the next effect run and on unmount (not before the first run).

---

### Q29. What causes infinite loops in useEffect?

**Short answer**
An effect that sets state every time it runs, with a dependency array that changes as a result (or no array at all), re-triggers itself endlessly. Common causes are missing deps array, object/array/function deps recreated every render, and updating a state that is also a dep.

**Detailed explanation**
- `useEffect(() => setX(...))` with no deps → render → effect → setState → render…
- Deps like `{}`, `[]` literals, or inline functions are new each render, so the effect always re-runs.
- Effect sets state that appears in its own deps (`data` → `setData(data.map…)`).
- Fixes: add correct deps, use functional updates to drop the dependency, memoize (`useMemo`/`useCallback`), move objects inside the effect or outside the component, compute derived data during render instead of in an effect.

**Code example**
```jsx
import { useEffect, useMemo, useState } from "react";

function Bad({ userId }) {
  const [data, setData] = useState([]);
  const params = { userId };           // new object each render
  useEffect(() => {
    fetch(`/api?u=${params.userId}`).then((r) => r.json()).then(setData);
  }, [params]);                         // ❌ loop: setData -> render -> new params
  return null;
}

function Good({ userId }) {
  const [data, setData] = useState([]);
  useEffect(() => {
    let ignore = false;
    fetch(`/api?u=${userId}`).then((r) => r.json()).then((d) => !ignore && setData(d));
    return () => { ignore = true; };
  }, [userId]);                         // ✅ primitive dep
  const total = useMemo(() => data.reduce((s, x) => s + x.price, 0), [data]); // derive, don't effect
  return <p>{total}</p>;
}
```

**Common pitfalls / follow-up questions**
- *Is an infinite loop always from setState?* Yes, effect loops need a re-render trigger; React warns "Maximum update depth exceeded".
- *Function prop as dep?* Wrap in `useCallback` in the parent, or use `useEffectEvent` (experimental/React 19.2).
- *Derived state?* Compute during render; don't mirror props into state via an effect.

---

### Q30. What is a stale closure? Give an example.

**Short answer**
A stale closure is a function that captured variables from an older render and keeps using those outdated values. In hooks it typically occurs in intervals, listeners, or effects whose dependency array omits values they read.

**Detailed explanation**
- Each render creates new function instances closing over that render's props/state.
- An effect with `[]` deps keeps the first render's closure forever.
- Fixes: include proper deps (re-subscribe), use functional updates (`setN(n => n+1)`), or store the latest value in a ref, or use `useEffectEvent`.

**Code example**
```jsx
import { useEffect, useRef, useState } from "react";

function StaleCounter() {
  const [count, setCount] = useState(0);
  useEffect(() => {
    const id = setInterval(() => {
      console.log(count);        // always 0 — stale
      setCount(count + 1);       // ❌ goes to 1 once, then stuck
    }, 1000);
    return () => clearInterval(id);
  }, []);
  return <p>{count}</p>;
}

function FixedCounter() {
  const [count, setCount] = useState(0);
  const latest = useRef(count);
  latest.current = count;        // keep ref current (ideally inside an effect)

  useEffect(() => {
    const id = setInterval(() => {
      setCount((c) => c + 1);    // ✅ functional update
      console.log(latest.current); // ✅ latest via ref
    }, 1000);
    return () => clearInterval(id);
  }, []);
  return <p>{count}</p>;
}
```

**Common pitfalls / follow-up questions**
- *Can I just add `count` to deps?* Yes, but the interval resets every change; functional update is nicer.
- *Do refs solve everything?* Use sparingly; don't write refs during render in strict-purity code (assign in an effect).
- *Is it a React bug?* No, it's JavaScript closure semantics.

---

### Q31. useEffect vs useLayoutEffect?

**Short answer**
`useEffect` runs asynchronously after the browser paints; `useLayoutEffect` runs synchronously after DOM mutations but before paint, blocking it. Use `useLayoutEffect` only to measure/adjust layout to avoid visual flicker.

**Detailed explanation**
- Both have the same signature and cleanup behavior.
- `useLayoutEffect` state updates trigger a synchronous re-render before paint, so users never see the intermediate state.
- It blocks painting, so heavy work hurts performance.
- On the server (SSR) `useLayoutEffect` warns in older React versions; use `useEffect` or an isomorphic variant.
- Typical use: tooltips/popovers positioning, measuring `getBoundingClientRect`, scroll restoration.

**Code example**
```jsx
import { useLayoutEffect, useRef, useState } from "react";

function Tooltip({ anchorRef, children }) {
  const tipRef = useRef(null);
  const [pos, setPos] = useState({ top: 0, left: 0 });

  useLayoutEffect(() => {
    const a = anchorRef.current.getBoundingClientRect();
    const t = tipRef.current.getBoundingClientRect();
    setPos({ top: a.top - t.height - 8, left: a.left + (a.width - t.width) / 2 });
  }, [anchorRef, children]);

  return (
    <div ref={tipRef} style={{ position: "fixed", ...pos }}>
      {children}
    </div>
  );
}
```

**Common pitfalls / follow-up questions**
- *Which should be the default?* `useEffect`.
- *Symptom needing layout effect?* Flicker where UI renders at the wrong position first.
- *Order?* Layout effects run before passive effects; cleanups likewise.

---

### Q32. What is useRef? Which two use cases does it have?

**Short answer**
`useRef(initial)` returns a stable mutable object `{ current }` that persists across renders and doesn't trigger re-renders when changed. Use cases: (1) accessing DOM nodes/component handles, (2) storing mutable values that aren't part of rendering (timer IDs, previous values, latest callbacks).

**Detailed explanation**
- Same object identity every render; changing `.current` doesn't re-render.
- DOM refs: `<input ref={inputRef} />`; `inputRef.current` is set after commit (null before mount/after unmount).
- Instance variable: interval IDs, `isMounted`, previous values, WebSocket instances.
- Don't read/write `ref.current` during render (except lazy init) because it breaks purity; do it in handlers/effects.
- Callback refs `ref={(node) => ...}` (React 19 allows cleanup return) handle dynamic nodes.
- React 19: `ref` is a regular prop for function components.

**Code example**
```jsx
import { useEffect, useRef, useState } from "react";

function SearchBox() {
  const inputRef = useRef(null);          // use case 1: DOM
  const timerRef = useRef(null);          // use case 2: mutable value
  const prevRef = useRef("");
  const [text, setText] = useState("");

  useEffect(() => { inputRef.current.focus(); }, []);
  useEffect(() => { prevRef.current = text; });

  function onChange(e) {
    setText(e.target.value);
    clearTimeout(timerRef.current);
    timerRef.current = setTimeout(() => console.log("search", e.target.value), 300);
  }

  return (
    <>
      <input ref={inputRef} value={text} onChange={onChange} />
      <p>Now: {text} | Previous render: {prevRef.current}</p>
    </>
  );
}
```

**Common pitfalls / follow-up questions**
- *Ref vs state?* State for values affecting UI; ref for those that don't.
- *Why is `ref.current` null?* Accessed before mount or the element isn't rendered.
- *Does changing a ref re-render?* No.

---

### Q33. What is useMemo, and when does it hurt?

**Short answer**
`useMemo(fn, deps)` caches the result of a computation between renders, recomputing only when deps change. It hurts when used everywhere: the comparison and memory overhead can exceed the savings, and wrong deps cause stale values.

**Detailed explanation**
- Good for: expensive calculations (filtering/sorting large lists, ~1ms+), stabilizing object/array references passed to `React.memo` children or used as effect deps.
- It's a performance hint, not a semantic guarantee; React may discard the cache.
- Costs: extra memory, dependency comparison each render, code complexity.
- Don't use for cheap operations, or when result isn't passed to memoized consumers.
- React Compiler (React 19 ecosystem) auto-memoizes, reducing manual need.
- Measure first (React DevTools Profiler, `console.time`).

**Code example**
```jsx
import { memo, useMemo, useState } from "react";

const Row = memo(function Row({ item, style }) {
  return <li style={style}>{item.name}</li>;
});

function List({ items, query, color }) {
  const [sortAsc, setSortAsc] = useState(true);

  const visible = useMemo(() => {
    const q = query.toLowerCase();
    return items
      .filter((i) => i.name.toLowerCase().includes(q))
      .toSorted((a, b) => (sortAsc ? 1 : -1) * a.name.localeCompare(b.name));
  }, [items, query, sortAsc]);

  const style = useMemo(() => ({ color }), [color]); // stable ref for memo child

  return (
    <ul>
      {visible.map((i) => <Row key={i.id} item={i} style={style} />)}
    </ul>
  );
}
```

**Common pitfalls / follow-up questions**
- *Does useMemo guarantee no recompute?* No, only an optimization.
- *When harmful?* Trivial computations, unstable deps (always recomputes), premature optimization.
- *Alternatives?* Move state down, lift content up (`children`), React Compiler.

---

### Q34. What is useCallback, and how does it differ from useMemo?

**Short answer**
`useCallback(fn, deps)` returns a memoized function reference that changes only when deps change. It equals `useMemo(() => fn, deps)`: `useMemo` caches a computed value; `useCallback` caches the function itself.

**Detailed explanation**
- Useful when passing callbacks to `React.memo` children, or when a function is an effect dependency.
- Without `memo` on the child, `useCallback` gives no render benefit.
- Use functional state updates inside to keep deps minimal.
- It doesn't prevent function creation; it just reuses the earlier one.

**Code example**
```jsx
import { memo, useCallback, useState } from "react";

const TodoItem = memo(function TodoItem({ todo, onToggle }) {
  console.log("render", todo.id);
  return <li onClick={() => onToggle(todo.id)}>{todo.text}</li>;
});

function TodoList() {
  const [todos, setTodos] = useState([
    { id: 1, text: "A", done: false },
    { id: 2, text: "B", done: false },
  ]);
  const [filter, setFilter] = useState("");

  const onToggle = useCallback((id) => {
    setTodos((list) => list.map((t) => (t.id === id ? { ...t, done: !t.done } : t)));
  }, []); // stable thanks to functional update

  return (
    <>
      <input value={filter} onChange={(e) => setFilter(e.target.value)} />
      <ul>{todos.map((t) => <TodoItem key={t.id} todo={t} onToggle={onToggle} />)}</ul>
    </>
  );
}
```

**Common pitfalls / follow-up questions**
- *Is useCallback always good?* No; useless without memoized consumers.
- *Difference in one line?* `useMemo` memoizes the return value of `fn`; `useCallback` memoizes `fn`.
- *Missing deps?* Stale closures.

---

### Q35. What is useReducer? When is it better than useState?

**Short answer**
`useReducer(reducer, initialState)` manages state via `dispatch(action)` and a pure `reducer(state, action) => newState`. Prefer it when state is complex, has multiple sub-values, or next state depends on intricate transitions.

**Detailed explanation**
- Centralizes update logic, making it testable and predictable; actions describe *what happened*.
- `dispatch` identity is stable, so it's safe in deps and to pass down without `useCallback`.
- Reducers must be pure (no side effects, no mutation); Strict Mode calls them twice to verify.
- Combine with context for app-level state (Redux-lite).
- Use `useState` for simple independent values.

**Code example**
```jsx
import { useReducer } from "react";

const initial = { items: [], status: "idle" };

function reducer(state, action) {
  switch (action.type) {
    case "added":
      return { ...state, items: [...state.items, { id: action.id, text: action.text, done: false }] };
    case "toggled":
      return {
        ...state,
        items: state.items.map((i) => (i.id === action.id ? { ...i, done: !i.done } : i)),
      };
    case "removed":
      return { ...state, items: state.items.filter((i) => i.id !== action.id) };
    default:
      throw new Error(`Unknown action: ${action.type}`);
  }
}

export default function Todos() {
  const [state, dispatch] = useReducer(reducer, initial);
  return (
    <>
      <button onClick={() => dispatch({ type: "added", id: crypto.randomUUID(), text: "Task" })}>
        Add
      </button>
      <ul>
        {state.items.map((i) => (
          <li key={i.id} onClick={() => dispatch({ type: "toggled", id: i.id })}>
            {i.done ? "✅" : "⬜"} {i.text}
          </li>
        ))}
      </ul>
    </>
  );
}
```

**Common pitfalls / follow-up questions**
- *Can reducers fetch data?* No; do async in handlers/effects and dispatch results.
- *Lazy init?* Third arg: `useReducer(reducer, arg, init)`.
- *useReducer vs Redux?* Redux adds middleware, devtools, global store; `useReducer` is local.

---

### Q36. What is useContext? What are its re-render implications?

**Short answer**
`useContext(Ctx)` reads the nearest provider's value, avoiding prop drilling. Every consumer re-renders whenever the provider's `value` changes (by `Object.is`), even if it uses only part of it, and `React.memo` can't block it.

**Detailed explanation**
- Provider `value={{ a, b }}` creates a new object each render → all consumers re-render whenever the provider re-renders. Memoize the value.
- Split contexts by concern (state vs dispatch, or fast-changing vs static).
- Not a full state manager; for high-frequency updates use selector-based libs (Zustand, Redux, `useSyncExternalStore`).
- Memoizing children passed via `children` prop prevents unnecessary renders of non-consumers.
- React 19: `use(Context)` can read context conditionally; `<Context>` can be used directly as provider (no `.Provider`).

**Code example**
```jsx
import { createContext, useContext, useMemo, useState } from "react";

const ThemeContext = createContext(null);

export function ThemeProvider({ children }) {
  const [theme, setTheme] = useState("light");
  const value = useMemo(
    () => ({ theme, toggle: () => setTheme((t) => (t === "light" ? "dark" : "light")) }),
    [theme]
  );
  return <ThemeContext value={value}>{children}</ThemeContext>; // React 19 syntax
}

export function useTheme() {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error("useTheme must be used within ThemeProvider");
  return ctx;
}

function ThemeButton() {
  const { theme, toggle } = useTheme();
  return <button onClick={toggle}>Theme: {theme}</button>;
}
```

**Common pitfalls / follow-up questions**
- *Why do all consumers re-render?* Context has no selectors; any value change notifies all.
- *Default value used when?* Only when no provider is above.
- *Reduce renders?* Split contexts, memoize value, or use an external store with selectors.

---

### Q37. What are custom hooks? Write useFetch, useDebounce and useLocalStorage.

**Short answer**
A custom hook is a function starting with `use` that calls other hooks to encapsulate and reuse stateful logic. Each call has isolated state; they share logic, not state.

**Detailed explanation**
- Extract when logic (state + effects) repeats or when a component mixes concerns.
- Return values/functions that suit consumers; keep APIs small.
- Hooks must follow the Rules of Hooks.
- For production data fetching prefer TanStack Query/SWR; `useFetch` below illustrates cleanup and race handling.

**Code example**
```jsx
import { useEffect, useState, useCallback, useSyncExternalStore } from "react";

export function useFetch(url) {
  const [state, setState] = useState({ data: null, error: null, loading: !!url });

  useEffect(() => {
    if (!url) return;
    const controller = new AbortController();
    setState((s) => ({ ...s, loading: true, error: null }));
    fetch(url, { signal: controller.signal })
      .then((res) => {
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return res.json();
      })
      .then((data) => setState({ data, error: null, loading: false }))
      .catch((error) => {
        if (error.name !== "AbortError") setState({ data: null, error, loading: false });
      });
    return () => controller.abort();
  }, [url]);

  return state;
}

export function useDebounce(value, delay = 300) {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id);
  }, [value, delay]);
  return debounced;
}

export function useLocalStorage(key, initialValue) {
  const [value, setValue] = useState(() => {
    try {
      const raw = window.localStorage.getItem(key);
      return raw !== null ? JSON.parse(raw) : initialValue;
    } catch {
      return initialValue;
    }
  });

  useEffect(() => {
    try {
      window.localStorage.setItem(key, JSON.stringify(value));
    } catch { /* quota or private mode */ }
  }, [key, value]);

  useEffect(() => {
    const onStorage = (e) => {
      if (e.key === key && e.newValue !== null) setValue(JSON.parse(e.newValue));
    };
    window.addEventListener("storage", onStorage); // sync across tabs
    return () => window.removeEventListener("storage", onStorage);
  }, [key]);

  return [value, setValue];
}

// Usage
function Search() {
  const [query, setQuery] = useLocalStorage("q", "");
  const debounced = useDebounce(query, 400);
  const { data, loading, error } = useFetch(debounced ? `/api/search?q=${encodeURIComponent(debounced)}` : null);

  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      {loading && <p>Loading…</p>}
      {error && <p>{error.message}</p>}
      <ul>{data?.map((r) => <li key={r.id}>{r.title}</li>)}</ul>
    </>
  );
}
```

**Common pitfalls / follow-up questions**
- *Do two components using a hook share state?* No, each gets its own.
- *useFetch flaws?* No caching, dedupe, retry; stale data between URL changes — use TanStack Query.
- *SSR issue with useLocalStorage?* `window` is undefined on server; lazy init guard or `useSyncExternalStore` avoids hydration mismatch.

---

### Q38. What are useImperativeHandle and forwardRef?

**Short answer**
`forwardRef` lets a component pass a `ref` to a child DOM node/component (needed pre-React 19). `useImperativeHandle(ref, () => api, deps)` customizes what the parent receives, exposing a limited imperative API (`focus`, `reset`) instead of the whole DOM node.

**Detailed explanation**
- By default function components can't receive `ref` as a prop (React ≤18) — wrap with `forwardRef`.
- React 19: `ref` is a normal prop for function components; `forwardRef` is no longer required (and will be deprecated).
- Use `useImperativeHandle` sparingly; prefer declarative props. Valid for focus, scroll, play/pause, imperative animations.
- Exposes a narrow API, preserving encapsulation.

**Code example**
```jsx
import { useImperativeHandle, useRef } from "react";

// React 19: ref is a prop
function FancyInput({ ref, ...props }) {
  const inputRef = useRef(null);

  useImperativeHandle(ref, () => ({
    focus: () => inputRef.current.focus(),
    clear: () => { inputRef.current.value = ""; },
  }), []);

  return <input ref={inputRef} {...props} />;
}

// React 18 equivalent:
// const FancyInput = forwardRef(function FancyInput(props, ref) { ... });

export default function Form() {
  const fieldRef = useRef(null);
  return (
    <>
      <FancyInput ref={fieldRef} placeholder="Name" />
      <button onClick={() => fieldRef.current.focus()}>Focus</button>
      <button onClick={() => fieldRef.current.clear()}>Clear</button>
    </>
  );
}
```

**Common pitfalls / follow-up questions**
- *Why is my ref null?* Component didn't forward it (React 18) or the element isn't mounted.
- *Why not expose the whole DOM node?* Breaks encapsulation; parent could mutate anything.
- *Deps array?* Include values the handle uses to avoid stale closures.

---

### Q39. What are useId, useTransition and useDeferredValue?

**Short answer**
`useId` generates stable unique IDs consistent between server and client (for accessibility attributes). `useTransition` marks state updates as non-urgent so the UI stays responsive. `useDeferredValue` gives a lagging copy of a value that updates after urgent work.

**Detailed explanation**
- **useId:** Safe for SSR hydration; use for `htmlFor`/`id`, `aria-*`. Don't use for list keys. Don't use output in CSS selectors (contains colons).
- **useTransition:** `const [isPending, startTransition] = useTransition()`. Updates inside are interruptible low-priority; urgent input updates stay snappy. React 19 supports async functions in transitions (Actions).
- **useDeferredValue:** Use when you don't control the setter (prop from parent). The deferred value renders with the old value first, then re-renders in background. Pair with `memo` for the expensive child.
- Both rely on concurrent rendering and Suspense integration.

**Code example**
```jsx
import { memo, useDeferredValue, useId, useState, useTransition } from "react";

const BigList = memo(function BigList({ query }) {
  const items = Array.from({ length: 5000 }, (_, i) => `Item ${i}`).filter((x) => x.includes(query));
  return <ul>{items.map((x) => <li key={x}>{x}</li>)}</ul>;
});

export default function SearchPage() {
  const id = useId();
  const [text, setText] = useState("");
  const deferred = useDeferredValue(text);

  const [tab, setTab] = useState("home");
  const [isPending, startTransition] = useTransition();

  return (
    <>
      <label htmlFor={id}>Search</label>
      <input id={id} value={text} onChange={(e) => setText(e.target.value)} />
      <BigList query={deferred} />

      <button onClick={() => startTransition(() => setTab("reports"))}>
        Reports {isPending && "…"}
      </button>
      <p>Tab: {tab}</p>
    </>
  );
}
```

**Common pitfalls / follow-up questions**
- *Transition vs debounce?* Transitions have no fixed delay and are interruptible; debounce delays work artificially.
- *Can controlled inputs be in a transition?* No; keep the input update urgent.
- *Why not `Math.random()` for ids?* Server/client mismatch breaks hydration.

---

### Q40. What is useSyncExternalStore?

**Short answer**
`useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot?)` subscribes a component to an external (non-React) store in a concurrent-safe way, avoiding tearing. It's used by state libraries and for browser APIs like `navigator.onLine` or `matchMedia`.

**Detailed explanation**
- `subscribe(callback)` registers a listener and returns an unsubscribe function.
- `getSnapshot()` returns the current value; must return a cached/immutable result (same reference when unchanged) or it loops infinitely.
- `getServerSnapshot` provides the initial value for SSR/hydration.
- Updates from the store are synchronous (can't be deferred by transitions).
- Define `subscribe` outside the component or memoize it, otherwise it resubscribes every render.

**Code example**
```jsx
import { useSyncExternalStore } from "react";

function subscribe(callback) {
  window.addEventListener("online", callback);
  window.addEventListener("offline", callback);
  return () => {
    window.removeEventListener("online", callback);
    window.removeEventListener("offline", callback);
  };
}

export function useOnlineStatus() {
  return useSyncExternalStore(
    subscribe,
    () => navigator.onLine,
    () => true // server snapshot
  );
}

// Minimal custom store
let count = 0;
const listeners = new Set();
export const counterStore = {
  subscribe: (l) => (listeners.add(l), () => listeners.delete(l)),
  getSnapshot: () => count,
  increment() { count++; listeners.forEach((l) => l()); },
};

function Counter() {
  const value = useSyncExternalStore(counterStore.subscribe, counterStore.getSnapshot);
  return <button onClick={counterStore.increment}>{value}</button>;
}
```

**Common pitfalls / follow-up questions**
- *What is tearing?* Different parts of UI render different store values in one concurrent render.
- *Why infinite loop?* `getSnapshot` returns a new object each call; cache it.
- *Why not `useEffect`+`useState`?* Can tear in concurrent rendering and flashes on hydration.

---

### Q41. What is useOptimistic? What are useActionState and use() in React 19?

**Short answer**
`useOptimistic` shows an immediate, temporary UI state while an async action runs, reverting automatically if it fails. `useActionState` manages state produced by an action (form submit) with pending status. `use()` reads a Promise (suspending) or Context during render, and can be called conditionally.

**Detailed explanation**
- **useOptimistic(state, updateFn):** returns `[optimisticState, addOptimistic]`. Call `addOptimistic` inside a transition/action; when the action finishes, UI reflects real state.
- **useActionState(action, initialState):** returns `[state, formAction, isPending]`. `action(prevState, formData)` can be async; use as `<form action={formAction}>`. Replaces `useFormState` (react-dom). Works with Server Functions and progressive enhancement.
- **use(resource):** unwraps Promise via Suspense (pending → fallback; rejection → error boundary) or reads Context. Unlike hooks, it can be in conditionals/loops. Promises should be created outside render (passed from Server Components or cached), not newly created in the client component each render.
- Related: `useFormStatus` (from `react-dom`) reads parent form's pending state.

**Code example**
```jsx
import { Suspense, use, useActionState, useOptimistic, useState, useTransition } from "react";

// useActionState + form action
async function addComment(prev, formData) {
  const text = formData.get("text");
  if (!text) return { ...prev, error: "Required" };
  await fetch("/api/comments", { method: "POST", body: JSON.stringify({ text }) });
  return { error: null, comments: [...prev.comments, text] };
}

function Comments() {
  const [state, formAction, isPending] = useActionState(addComment, { error: null, comments: [] });
  const [optimistic, addOptimistic] = useOptimistic(state.comments, (cur, t) => [...cur, `${t} (sending…)`]);

  async function action(formData) {
    addOptimistic(formData.get("text"));
    await formAction(formData);
  }

  return (
    <form action={action}>
      <input name="text" />
      <button disabled={isPending}>Post</button>
      {state.error && <p role="alert">{state.error}</p>}
      <ul>{optimistic.map((c, i) => <li key={i}>{c}</li>)}</ul>
    </form>
  );
}

// use() with a promise created outside render
function Profile({ userPromise }) {
  const user = use(userPromise);
  return <h2>{user.name}</h2>;
}

export function Page({ userPromise }) {
  return (
    <Suspense fallback={<p>Loading…</p>}>
      <Profile userPromise={userPromise} />
    </Suspense>
  );
}
```

**Common pitfalls / follow-up questions**
- *Why shouldn't I create the promise inside the component calling `use`?* A new promise each render re-suspends forever; create/cache it above.
- *What happens when an optimistic action fails?* Optimistic state reverts to the real state once the action ends.
- *Can `use` replace `useContext`?* It can read context, with the bonus of conditional calls.

---

### Q42. Why does useEffect run twice in Strict Mode (dev)?

**Short answer**
In development, React 18+ Strict Mode deliberately mounts, unmounts and remounts each component (setup → cleanup → setup) to expose effects lacking proper cleanup. It does not happen in production.

**Detailed explanation**
- Simulates the future where React preserves state while hiding/showing UI (Offscreen/Activity), so effects must be resilient to being re-run.
- Strict Mode also double-invokes render functions, `useState`/`useMemo`/`useReducer` initializers and reducers to catch impure code.
- If double-run causes bugs (duplicate subscriptions, double analytics), the cleanup is missing or the logic belongs in an event handler.
- Fix by writing symmetric cleanup, not by removing Strict Mode or using a "ran once" ref hack.
- One-time app initialization (not tied to component) belongs at module level.
- React 19 additionally reuses memoized results from the first render during the double render.

**Code example**
```jsx
import { StrictMode, useEffect } from "react";
import { createRoot } from "react-dom/client";

function Chat({ roomId }) {
  useEffect(() => {
    const conn = createConnection(roomId);
    conn.connect();                       // dev: connect → disconnect → connect
    return () => conn.disconnect();       // proper cleanup makes it harmless
  }, [roomId]);
  return <h1>Room {roomId}</h1>;
}

function createConnection(room) {
  return {
    connect: () => console.log("connect", room),
    disconnect: () => console.log("disconnect", room),
  };
}

createRoot(document.getElementById("root")).render(
  <StrictMode>
    <Chat roomId="general" />
  </StrictMode>
);
```

**Common pitfalls / follow-up questions**
- *My fetch fires twice in dev. Bug?* Not a bug; add abort/ignore cleanup or use a data library with dedupe.
- *Should I use a ref flag to run once?* No, it hides the bug and breaks with remounts.
- *Does it happen in production?* No.
