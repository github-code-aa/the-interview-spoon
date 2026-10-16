# React Interview Guide 03 — Lifecycle & Rendering

## Table of Contents
- [Q43. Class lifecycle methods and hook equivalents](#q43)
- [Q44. Mount, update, unmount phases](#q44)
- [Q45. Error boundaries](#q45)
- [Q46. What triggers a re-render?](#q46)
- [Q47. Parent re-render and children](#q47)
- [Q48. React.memo](#q48)
- [Q49. Reconciliation and diffing](#q49)
- [Q50. React Fiber](#q50)
- [Q51. Render vs commit phase](#q51)
- [Q52. Batching](#q52)
- [Q53. Concurrent rendering](#q53)
- [Q54. Suspense and lazy loading](#q54)
- [Q55. Portals](#q55)
- [Q56. Hydration and mismatches](#q56)
- [Q57. Refs vs state for DOM access](#q57)
- [Q58. flushSync](#q58)
- [Q59. Strict Mode](#q59)
- [Q60. Pure components and pure render](#q60)

---

<a id="q43"></a>
### Q43. Describe the class lifecycle methods and their hook equivalents.

**Short answer**
Class components have `constructor`, `render`, `componentDidMount`, `componentDidUpdate`, `componentWillUnmount` (plus `shouldComponentUpdate`, `getDerivedStateFromProps`, `getSnapshotBeforeUpdate`, `componentDidCatch`). In function components, `useState` initial value, `useEffect`/`useLayoutEffect` with dependency arrays and cleanup functions cover almost all of them.

**Detailed explanation**
| Class | Hook equivalent |
|---|---|
| `constructor` (init state) | `useState(initial)` / `useRef` |
| `render` | The function body |
| `componentDidMount` | `useEffect(() => {...}, [])` |
| `componentDidUpdate` | `useEffect(() => {...}, [deps])` (runs on mount too; skip with a ref if needed) |
| `componentWillUnmount` | Cleanup function returned from `useEffect` |
| `shouldComponentUpdate` | `React.memo` (+ `useMemo`/`useCallback`) |
| `getDerivedStateFromProps` | Compute during render, or set state during render conditionally |
| `getSnapshotBeforeUpdate` | No direct equivalent (closest: `useLayoutEffect`) |
| `componentDidCatch` / `getDerivedStateFromError` | No hook; still class-based error boundaries |

Hooks are organized by *synchronization* (what effect should stay in sync with which values), not by *time*. An effect with cleanup runs cleanup before re-running and on unmount.

```jsx
function Chat({ roomId }) {
  useEffect(() => {
    const conn = connect(roomId);       // didMount + didUpdate when roomId changes
    return () => conn.disconnect();     // willUnmount + before re-sync
  }, [roomId]);
  return <h1>Room {roomId}</h1>;
}
```

**Common pitfalls / follow-up questions**
- *Is `useEffect([])` exactly `componentDidMount`?* No: it syncs once, and in Strict Mode dev it mounts, unmounts, re-mounts to verify cleanup.
- *Which timing differs?* `componentDidMount` runs synchronously after commit (blocking paint); `useEffect` runs after paint; `useLayoutEffect` matches class timing.
- *Legacy methods?* `componentWillMount/ReceiveProps/Update` are deprecated (`UNSAFE_` prefix) because they break with concurrent rendering.

---

<a id="q44"></a>
### Q44. What are the mount, update and unmount phases?

**Short answer**
Mounting is when a component is first created and inserted into the DOM; updating is every re-render caused by props/state/context changes; unmounting is removal from the DOM. Each has associated hooks/methods for setup, reaction and cleanup.

**Detailed explanation**
- **Mount**: class `constructor → render → componentDidMount`; function: body runs, DOM inserted, layout effects then effects run.
- **Update**: `render → getSnapshotBeforeUpdate → componentDidUpdate`; function: body re-runs, previous effect cleanups run, then effects with changed deps re-run.
- **Unmount**: `componentWillUnmount`; function: all effect cleanups run.
- A component unmounts (and loses state) when its type or `key` at that position changes.

```
 Mount ──► Update ──► Update ──► ... ──► Unmount
 render     render     render            cleanup
 commit     commit     commit
 effects    cleanup+effects              cleanup
```

```jsx
function Timer() {
  const [n, setN] = useState(0);
  useEffect(() => {
    const id = setInterval(() => setN(x => x + 1), 1000);
    return () => clearInterval(id); // unmount phase
  }, []);
  return <p>{n}</p>;
}
```

**Common pitfalls / follow-up questions**
- *How do you force a remount?* Change the `key`.
- *Does state survive conditional rendering?* Not if the component is removed from the tree; yes if it stays at the same position with the same type.
- *Why forget cleanup is bad?* Leaks: timers, subscriptions, listeners keep running.

---

<a id="q45"></a>
### Q45. What are error boundaries? Why are they still class-based?

**Short answer**
Error boundaries are components that catch errors thrown during rendering, lifecycle methods and constructors of their descendants, and display a fallback UI instead of unmounting the whole tree. They remain classes because they rely on `getDerivedStateFromError` / `componentDidCatch`, which have no hook equivalent.

**Detailed explanation**
- `static getDerivedStateFromError(error)` returns state to render fallback (render phase, pure).
- `componentDidCatch(error, info)` is for side effects such as logging (commit phase).
- They do **not** catch: event handler errors, async code (setTimeout, promises), SSR errors, or errors in the boundary itself. Handle those with try/catch.
- No hook exists because the React team has not designed one; catching requires hooking into the internal render-error path. In practice use the `react-error-boundary` library, which wraps the class for you. React 19 adds `onCaughtError`/`onUncaughtError` root options.

```jsx
class ErrorBoundary extends React.Component {
  state = { error: null };
  static getDerivedStateFromError(error) { return { error }; }
  componentDidCatch(error, info) { logToService(error, info.componentStack); }
  render() {
    return this.state.error ? this.props.fallback : this.props.children;
  }
}

<ErrorBoundary fallback={<p>Something went wrong</p>}>
  <Dashboard />
</ErrorBoundary>
```

**Common pitfalls / follow-up questions**
- *Why didn't my boundary catch a click handler error?* Handlers aren't part of rendering; use try/catch.
- *How to reset?* Change `key` or clear state (`resetErrorBoundary` in react-error-boundary).
- *Where to place them?* Around route/feature sections so failures are isolated, plus one at top level.

---

<a id="q46"></a>
### Q46. What triggers a re-render?

**Short answer**
A component re-renders when its own state changes (`setState`/`useState`/`useReducer` with a different value), when a context it consumes changes, or when its parent re-renders. Props changing is really a consequence of the parent re-rendering.

**Detailed explanation**
- State update with a value that is not `Object.is`-equal to current.
- Parent re-render (children re-render by default).
- Context value change for every consumer of that context.
- External store change via `useSyncExternalStore`.
- `forceUpdate` in classes.
- Not triggers: mutating variables, refs (`ref.current = x`), or mutating state objects directly.
- Rendering means calling the function; it doesn't necessarily mean DOM changes.

```jsx
function Counter() {
  const [n, setN] = useState(0);
  const ref = useRef(0);
  return (
    <>
      <button onClick={() => setN(n + 1)}>state {n}</button>   {/* re-renders */}
      <button onClick={() => { ref.current++; }}>ref</button>  {/* no re-render */}
    </>
  );
}
```

**Common pitfalls / follow-up questions**
- *`setN(n)` with the same value?* React bails out (may render the component once more but won't render children or commit).
- *Mutating an array and calling setState with it?* Same reference → no re-render. Copy first.
- *Why does a context change re-render even memoized components?* `memo` doesn't block context updates.

---

<a id="q47"></a>
### Q47. Does a parent re-render always re-render its children? Why?

**Short answer**
Yes by default: when a parent renders, React re-invokes all child components it returns, regardless of whether props changed. This is because React can't know a child is pure; skipping requires `React.memo`, or passing the same element reference.

**Detailed explanation**
- Rendering is cheap and recursive: parent returns new element objects, and React renders each child element.
- Exceptions: child wrapped in `React.memo` with unchanged props; elements passed as `children`/props that keep the same reference (the "children as props" pattern); React Compiler auto-memoization.
- Moving state down or lifting content up avoids unneeded renders.

```jsx
function Parent() {
  const [n, setN] = useState(0);
  return (
    <>
      <button onClick={() => setN(n + 1)}>{n}</button>
      <Child />               {/* re-renders every time */}
    </>
  );
}

function Wrapper({ children }) {
  const [n, setN] = useState(0);
  return <div onMouseMove={() => setN(n + 1)}>{children}</div>; // children element is same reference → not re-rendered
}
```

**Common pitfalls / follow-up questions**
- *Does a re-render mean DOM update?* No; React diffs and only commits changes.
- *Is it always a perf problem?* Rarely; measure with Profiler first.
- *Do unchanged-prop children skip?* Only with `memo`.

---

<a id="q48"></a>
### Q48. What is React.memo? When does it fail (new object/function props)?

**Short answer**
`React.memo` wraps a component so it re-renders only if its props change by shallow comparison (`Object.is` per prop). It fails to help when props are new references each render, such as inline objects, arrays or functions.

**Detailed explanation**
- Shallow compare: `{a:1}` !== `{a:1}` by reference, so inline object/array/function props always break memoization.
- Fix with `useMemo` (values) and `useCallback` (functions), or hoist constants outside the component.
- `children` JSX is a new object each render, which also defeats memo.
- A custom comparator can be passed as 2nd arg, but is rarely worthwhile.
- Memo doesn't stop re-renders from own state or context changes.
- React Compiler (19) can automate memoization.

```jsx
const Row = React.memo(function Row({ item, onSelect }) {
  return <li onClick={() => onSelect(item.id)}>{item.name}</li>;
});

function List({ items }) {
  const [sel, setSel] = useState(null);
  const onSelect = useCallback(id => setSel(id), []); // stable
  return items.map(i => <Row key={i.id} item={i} onSelect={onSelect} />);
  // Broken: <Row style={{color:'red'}} onSelect={id => setSel(id)} />
}
```

**Common pitfalls / follow-up questions**
- *Should I memo everything?* No; comparison has a cost. Use for expensive subtrees or frequently re-rendered parents.
- *useCallback without memo child?* Pointless.
- *memo vs useMemo?* memo memoizes component output by props; useMemo memoizes a computed value.

---

<a id="q49"></a>
### Q49. What is reconciliation? Explain the diffing algorithm.

**Short answer**
Reconciliation is how React compares the new element tree with the previous one to compute the minimal set of DOM updates. It uses O(n) heuristics: elements of different types produce different trees, and `key`s identify stable items in lists.

**Detailed explanation**
General tree diff is O(n³); React assumes:
1. **Different type at same position** → tear down old subtree (unmount, state lost) and build new.
2. **Same type** → keep the instance/DOM node, update changed props/attributes, recurse into children.
3. **Lists** → compare by `key`; matching keys are moved/updated, new keys mounted, missing keys unmounted. Without keys, React matches by index.
- Comparison happens level by level; React doesn't detect moved subtrees across parents.

```
Before:  <div>            After:  <span>
           <Counter/>              <Counter/>     → different root type: Counter remounts (state lost)
```

```jsx
{items.map(i => <Item key={i.id} {...i} />)}  // stable id keys
// Avoid index keys when list reorders/inserts: state attaches to wrong item
```

**Common pitfalls / follow-up questions**
- *Why not use `Math.random()` as key?* New key each render → remount every time.
- *Defining a component inside another component?* New type every render → remount each time.
- *Reconciliation vs rendering library (react-dom)?* Reconciler is shared core; renderers (DOM, Native) apply the changes.

---

<a id="q50"></a>
### Q50. What is React Fiber?

**Short answer**
Fiber is the reimplementation of React's reconciler (React 16) that represents work as a linked list of units called fibers, making rendering incremental, interruptible and prioritizable. It is the foundation of concurrent features.

**Detailed explanation**
- Each fiber is a JS object for a component/DOM node holding type, props, state, `child`, `sibling`, `return` pointers, effect flags and lanes (priority).
- The old stack reconciler used recursion and couldn't pause. Fiber walks the tree iteratively, so it can yield to the browser every few ms (time slicing) and resume or discard work.
- **Double buffering**: `current` tree (on screen) and `workInProgress` tree; on commit they swap.
- Priorities (lanes): discrete input > default > transitions > idle.

```
        App (fiber)
        │ child
        ▼
      Header ── sibling ──► Main ── sibling ──► Footer
        ▲ return (parent) pointers on each
```

**Common pitfalls / follow-up questions**
- *Is Fiber an API you use?* No, internal architecture.
- *Why must render be pure?* Fiber may start, pause, restart or discard a render.
- *What does it enable?* Transitions, Suspense, selective hydration, error boundaries.

---

<a id="q51"></a>
### Q51. Explain the render phase vs the commit phase.

**Short answer**
The render phase calls your components to compute what the UI should look like; it's pure and can be paused, restarted or discarded. The commit phase applies the result to the DOM synchronously and runs layout effects, then effects.

**Detailed explanation**
- **Render phase**: component bodies, `useMemo`/`useReducer` initializers, `getDerivedStateFromProps`, `shouldComponentUpdate`. No side effects allowed (may run multiple times).
- **Commit phase** (synchronous, uninterruptible): 
  1. Before mutation (`getSnapshotBeforeUpdate`)
  2. Mutation: DOM changes, refs detached/attached; cleanups of layout effects
  3. Layout: `useLayoutEffect`, `componentDidMount/Update`, refs set
  4. After paint: passive effects (`useEffect`)
- Browser paints between layout and passive effects (usually).

```
Trigger → [Render: call components, diff] → [Commit: DOM mutate → layout effects] → paint → useEffect
```

```jsx
function Bad() {
  document.title = 'x';  // side effect in render: wrong
  useEffect(() => { document.title = 'x'; }); // correct
}
```

**Common pitfalls / follow-up questions**
- *Why can't render have side effects?* It may be invoked repeatedly or thrown away.
- *When does a ref become available?* During commit; don't read `ref.current` in render.
- *useEffect vs useLayoutEffect?* The latter blocks paint; use for measuring layout.

---

<a id="q52"></a>
### Q52. What is batching (automatic batching in React 18)?

**Short answer**
Batching groups multiple state updates into a single re-render. React 18 with `createRoot` batches automatically everywhere, including promises, timeouts and native event handlers; earlier versions batched only inside React event handlers.

**Detailed explanation**
- In React 17, `setTimeout(() => { setA(); setB(); })` caused two renders; in 18 only one.
- Updates are queued and processed together at the end of the event/task.
- State is a snapshot: after `setN(n+1)`, `n` in the same handler is still old; use updater functions for sequential updates.
- Opt out with `flushSync`.

```jsx
function handle() {
  setCount(c => c + 1);
  setFlag(f => !f);
  // one render
}

fetch('/api').then(() => {
  setData(d); setLoading(false); // React 18: single render
});
```

**Common pitfalls / follow-up questions**
- *Legacy `ReactDOM.render`?* Doesn't get automatic batching; migrate to `createRoot`.
- *Can I read updated state immediately?* No; use the value you computed or an effect.
- *Different events batched together?* No; each discrete event is flushed separately (e.g., two clicks).

---

<a id="q53"></a>
### Q53. What is concurrent rendering?

**Short answer**
Concurrent rendering lets React prepare multiple versions of the UI and interrupt, pause or abandon a render in favor of more urgent updates. It is enabled by `createRoot` and used via features like `useTransition`, `useDeferredValue` and Suspense.

**Detailed explanation**
- Not a feature to turn on per se; it's a capability: rendering becomes interruptible, so typing stays responsive while a heavy list re-renders.
- `startTransition` marks updates as non-urgent; React keeps showing the old UI until the new one is ready and can drop stale renders.
- `useDeferredValue` gives a lagging copy of a value.
- Requires pure render functions and safe external stores (`useSyncExternalStore`) to avoid tearing.
- React 19 adds Actions: `useActionState`, `useOptimistic`, async transitions.

```jsx
function Search() {
  const [q, setQ] = useState('');
  const [isPending, startTransition] = useTransition();
  const [results, setResults] = useState([]);

  function onChange(e) {
    setQ(e.target.value);                       // urgent
    startTransition(() => setResults(filter(e.target.value))); // non-urgent
  }
  return <>
    <input value={q} onChange={onChange} />
    {isPending && <Spinner />}
    <List items={results} />
  </>;
}
```

**Common pitfalls / follow-up questions**
- *Does it use multithreading?* No; single thread with time slicing.
- *Can render run twice?* Yes, so effects/side effects must not live in render.
- *Transition vs debounce?* Transition has no fixed delay and is interruptible; debounce reduces calls.

---

<a id="q54"></a>
### Q54. What are Suspense and lazy loading?

**Short answer**
`React.lazy` code-splits a component by loading it on demand via dynamic `import()`, and `<Suspense>` shows a fallback while a child isn't ready. Suspense also works with Suspense-enabled data fetching (`use`, frameworks, Relay) and streaming SSR.

**Detailed explanation**
- A suspending component throws a promise (or calls `use(promise)`); the nearest Suspense boundary shows `fallback` and retries when resolved.
- `lazy` requires default exports and should be declared at module top level.
- Combine with transitions so already-visible content isn't replaced by a fallback.
- Pair with Error Boundaries for load failures.
- Use route-level splitting first.

```jsx
const Chart = lazy(() => import('./Chart'));

function Page() {
  return (
    <ErrorBoundary fallback={<p>Failed</p>}>
      <Suspense fallback={<Spinner />}>
        <Chart />
      </Suspense>
    </ErrorBoundary>
  );
}

// React 19 data
function Comments({ promise }) {
  const comments = use(promise);
  return comments.map(c => <p key={c.id}>{c.text}</p>);
}
```

**Common pitfalls / follow-up questions**
- *Create the promise inside the suspending component?* No; it's recreated each render. Cache or pass from parent/framework.
- *Does Suspense detect `useEffect` fetches?* No.
- *Fallback flicker?* Wrap navigation in `startTransition`.

---

<a id="q55"></a>
### Q55. What are Portals? Give use cases.

**Short answer**
`createPortal(children, domNode)` renders children into a different DOM node outside the parent hierarchy while keeping them in the same React tree. Used for modals, tooltips, dropdowns and toasts that must escape overflow/z-index clipping.

**Detailed explanation**
- Context flows through portals as normal; events bubble through the React tree (not DOM tree), so a parent's `onClick` receives clicks from a portaled child.
- Solves `overflow: hidden`, stacking context and `position: fixed` issues from ancestors with transforms.
- Accessibility is your responsibility: focus trapping, `aria-modal`, Escape key, return focus. Native `<dialog>` is an alternative.
- Not usable for SSR on `document.body` at render without guarding for the DOM.

```jsx
import { createPortal } from 'react-dom';

function Modal({ open, onClose, children }) {
  if (!open) return null;
  return createPortal(
    <div role="dialog" aria-modal="true" className="backdrop" onClick={onClose}>
      <div onClick={e => e.stopPropagation()}>{children}</div>
    </div>,
    document.body
  );
}
```

**Common pitfalls / follow-up questions**
- *Do events bubble to React ancestors?* Yes, which can surprise; stop propagation if needed.
- *Does CSS inheritance work?* No; DOM-based inheritance follows the DOM location.
- *Other uses?* Tooltips, popovers, toasts, rendering into iframes.

---

<a id="q56"></a>
### Q56. What is hydration? What causes hydration mismatches?

**Short answer**
Hydration is attaching React to server-rendered HTML: React builds its tree, reuses the existing DOM and adds event handlers, via `hydrateRoot`. Mismatches occur when the client's first render produces different output than the server HTML.

**Detailed explanation**
- Causes: `Date.now()`, `Math.random()`, `typeof window` branching, locale/timezone formatting, browser-only state (localStorage), invalid HTML nesting (`<div>` in `<p>`), browser extensions altering DOM, differing data.
- Results: warnings in dev, React 18/19 re-renders client-side to recover, losing performance, possibly UI glitches.
- Fixes: render the same on first pass and update in `useEffect`; `useId` for IDs; `suppressHydrationWarning` for unavoidable text (one level); client-only components.
- React 18: streaming SSR + selective hydration with Suspense; React 19 improves mismatch diffs and third-party script tolerance.

```jsx
function Clock() {
  const [now, setNow] = useState(null);       // same on server and first client render
  useEffect(() => setNow(new Date().toLocaleTimeString()), []);
  return <p>{now ?? '—'}</p>;
}

hydrateRoot(document.getElementById('root'), <App />);
```

**Common pitfalls / follow-up questions**
- *Is hydration free?* No; JS must download and run. Server Components reduce this.
- *Use `useSyncExternalStore` `getServerSnapshot`?* Yes for external stores.
- *Does a mismatch patch attributes?* Not reliably; text/attributes may remain wrong.

---

<a id="q57"></a>
### Q57. What are Refs vs state for DOM access?

**Short answer**
Refs (`useRef`) give direct access to DOM nodes or mutable values without causing re-renders; state is for data that drives the UI and triggers re-renders. Use refs for imperative actions (focus, scroll, measure, media play) and state for anything displayed.

**Detailed explanation**
- `ref.current` is mutable, persists across renders, and is set during commit; changing it doesn't re-render.
- Don't read/write `ref.current` during render (except lazy init).
- Callback refs and (React 19) ref cleanup functions allow dynamic nodes; React 19 passes `ref` as a normal prop to function components (no `forwardRef` needed).
- Avoid manipulating React-managed DOM (adding/removing children) via refs.

```jsx
function SearchBox() {
  const inputRef = useRef(null);
  const [text, setText] = useState('');
  return (
    <>
      <input ref={inputRef} value={text} onChange={e => setText(e.target.value)} />
      <button onClick={() => inputRef.current.focus()}>Focus</button>
    </>
  );
}

function Fancy({ ref, ...props }) { return <input ref={ref} {...props} />; } // React 19
```

**Common pitfalls / follow-up questions**
- *`ref.current` is null?* Not yet mounted or conditionally rendered.
- *Controlled vs uncontrolled inputs?* Controlled use state; uncontrolled read value via ref/FormData.
- *Expose limited API?* `useImperativeHandle`.

---

<a id="q58"></a>
### Q58. What is flushSync?

**Short answer**
`flushSync` from `react-dom` forces React to synchronously process and commit updates inside its callback, opting out of batching. Use it rarely, when you need the DOM updated immediately (e.g., scrolling to a just-added item).

**Detailed explanation**
- Updates inside the callback are flushed to the DOM before `flushSync` returns.
- Hurts performance and can force pending effects/ Suspense fallbacks; can't be called during render or in lifecycle methods/effects (warns).
- Typical use: read layout after state change, integrating with third-party DOM libraries, `window.print` flows.

```jsx
import { flushSync } from 'react-dom';

function add() {
  flushSync(() => setTodos(t => [...t, newTodo]));
  listRef.current.lastChild.scrollIntoView();
}
```

**Common pitfalls / follow-up questions**
- *Alternative?* `useLayoutEffect` or `useEffect` reacting to state.
- *Does it make setState synchronous outside?* Only for updates inside the callback.
- *Why discouraged?* Defeats batching and concurrency.

---

<a id="q59"></a>
### Q59. What is Strict Mode?

**Short answer**
`<StrictMode>` is a development-only wrapper that surfaces bugs by double-invoking renders, state initializers and reducers, and by re-running effects (mount → unmount → mount). It has no effect on production builds or UI.

**Detailed explanation**
- Double render detects impure render functions.
- Extra effect setup/cleanup cycle verifies proper cleanup and resilience to remounts (needed for features like Offscreen/Activity and fast refresh).
- React 19 also double-invokes ref callbacks, reuses memoized results from first render on the second, and warns about deprecated APIs (legacy string refs, `findDOMNode`).
- Doesn't render visible UI.

```jsx
createRoot(document.getElementById('root')).render(
  <StrictMode>
    <App />
  </StrictMode>
);

useEffect(() => {
  const id = subscribe();
  return () => unsubscribe(id); // must fully undo setup
}, []);
```

**Common pitfalls / follow-up questions**
- *"My effect fires twice!"* Intentional in dev; fix missing cleanup, don't remove StrictMode or guard with a ref flag.
- *Does it affect production?* No.
- *Can scope be partial?* Yes, wrap only a subtree.

---

<a id="q60"></a>
### Q60. What are pure components and pure render functions?

**Short answer**
A pure render function returns the same JSX for the same props, state and context and causes no side effects. A "pure component" (`React.PureComponent` or `memo`) additionally skips re-rendering when props/state are shallowly equal.

**Detailed explanation**
- Purity rules: don't mutate props/state/outside variables during render, no network/DOM/timers/random in render; side effects go in handlers or effects.
- Local mutation of newly created values within render is fine.
- `PureComponent` implements `shouldComponentUpdate` with shallow comparison of props and state; function equivalent is `React.memo`.
- Purity enables concurrent rendering, Strict Mode checks, memoization and React Compiler.

```jsx
// Impure
let count = 0;
function Bad() { count++; return <p>{count}</p>; }

// Pure
function Good({ items }) {
  const sorted = [...items].sort();   // local copy, no mutation of props
  return <ul>{sorted.map(i => <li key={i}>{i}</li>)}</ul>;
}

class Row extends React.PureComponent { render() { return <li>{this.props.name}</li>; } }
```

**Common pitfalls / follow-up questions**
- *Why does PureComponent fail with mutated arrays?* Same reference → shallow equal → skipped update.
- *Is reading context pure?* Yes; context is an input.
- *Does purity forbid state?* No; state is an input managed by React.
