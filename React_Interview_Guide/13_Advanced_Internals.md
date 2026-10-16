# 13. Advanced React Internals (Q174–Q187)

## Table of Contents
- [Q174. How does useState work internally?](#q174-how-does-usestate-work-internally-the-linked-list-of-hooks)
- [Q175. Lanes and the scheduler priority model](#q175-explain-lanes-and-the-scheduler-priority-model)
- [Q176. Time slicing](#q176-what-is-time-slicing-and-how-does-it-keep-the-ui-responsive)
- [Q177. Bailing out of rendering](#q177-how-does-react-decide-to-bail-out-of-rendering)
- [Q178. Synthetic events and delegation](#q178-how-does-the-synthetic-event-system-work-event-delegation)
- [Q179. Keys, identity and state preservation](#q179-how-do-keys-affect-component-identity-and-state-preservation)
- [Q180. Resetting state with a key](#q180-how-do-you-reset-state-with-a-key)
- [Q181. Effect events / useEffectEvent](#q181-what-is-an-effect-event--useeffectevent)
- [Q182. Tearing and concurrent-safe external stores](#q182-what-are-tearing-and-concurrent-safe-external-stores)
- [Q183. Object.is vs deep equality](#q183-what-is-the-difference-between-objectis-comparison-and-deep-equality-in-react)
- [Q184. Cost of context value changes](#q184-what-is-the-cost-of-context-value-changes)
- [Q185. Implementing a mini React](#q185-how-would-you-implement-a-mini-react-createelement-render-hooks)
- [Q186. Implementing useState, useEffect, memo](#q186-how-would-you-implement-your-own-usestate-useeffect-or-memo)
- [Q187. RSC payload serialization](#q187-how-do-server-components-serialize-the-rsc-payload)

---

### Q174. How does useState work internally (the linked list of hooks)?

**Short answer**
Each function component's Fiber has a `memoizedState` field that points to a singly linked list of hook objects, one per hook call, in call order. `useState` reads/creates the hook at the current position of that list, holds a queue of pending updates, and `dispatch` enqueues an update and schedules a render. Because identity is by *call order*, hooks must not be called conditionally.

**Detailed explanation**
- During render React sets a module-level "current dispatcher": `HooksDispatcherOnMount` for the first render, `HooksDispatcherOnUpdate` afterwards. Calling `useState` calls `dispatcher.useState`.
- **Mount** (`mountState`): creates a hook `{ memoizedState, baseState, baseQueue, queue, next }`, appends it to the work-in-progress fiber's list (`workInProgressHook`), initialises state (calling the lazy initializer if a function), creates `queue = { pending, lanes, dispatch, lastRenderedReducer, lastRenderedState }` and binds `dispatch = dispatchSetState.bind(null, fiber, queue)`.
- **Update** (`updateState` → `updateReducer`): clones the matching hook from the *current* fiber's list (walking in the same order), takes the pending circular queue of updates, processes those whose lane is included in the render lanes, applies reducer (`basicStateReducer` = `typeof action === 'function' ? action(state) : action`), and stores the result. Updates skipped due to insufficient priority stay in `baseQueue` and are rebased later.
- **dispatch**: creates `update = { lane, action, hasEagerState, eagerState, next }`, appends it to `queue.pending` (circular list), and calls `scheduleUpdateOnFiber`. If the fiber has no pending work, React *eagerly* computes the next state; if `Object.is(eager, current)` it can bail out without scheduling a render at all.
- `useReducer` is the primitive; `useState` is `useReducer` with `basicStateReducer`.
- Hook order mismatch triggers "Rendered more/fewer hooks than expected".
- Two copies of the tree (current / work-in-progress) exist; hooks are cloned from current to WIP so a render can be discarded safely (important for concurrent rendering).

**Code example or diagram**

```
FiberNode (App)
  memoizedState ─► Hook#1 (useState count) ─► Hook#2 (useEffect) ─► Hook#3 (useRef) ─► null
                    │  memoizedState: 3
                    │  queue.pending ─► update ─► update (circular)
                    └  next
```

```jsx
function Counter() {
  const [count, setCount] = useState(0);      // hook #1
  const [name, setName] = useState('a');      // hook #2
  // setCount(c => c + 1) -> enqueue {action: fn} on hook #1's queue
  return <button onClick={() => { setCount(c => c + 1); setCount(c => c + 1); }}>{count}</button>;
  // Both updates are queued, then processed in order in ONE render -> count + 2
}
```

**Pitfalls / follow-ups**
- Conditional hooks break the position-based matching (hence the Rules of Hooks / eslint plugin).
- `setCount(count + 1)` twice uses the stale closure value → +1; functional updates → +2.
- Setting the same value (by `Object.is`) may still render the component once more before bailing out (no children re-render, no effects).
- Follow-up: why are hooks not stored by name? (Order-based storage is cheap and needs no keys; trade-off is the rules.)
- Follow-up: how does StrictMode double-invoking help detect impure initializers/reducers?

---

### Q175. Explain lanes and the scheduler priority model.

**Short answer**
Lanes are a 31-bit bitmask model React uses to label each update with a priority ("lane"). Multiple lanes can be batched in one render, higher-priority lanes preempt lower ones, and sets of lanes can be cheaply combined/compared with bitwise ops. The separate `scheduler` package maps React's priorities to macrotask priorities (Immediate, UserBlocking, Normal, Low, Idle).

**Detailed explanation**
- Replaced the older `expirationTime` model, which could not express "this set of updates, not that one" (e.g. a transition shouldn't block on an unrelated Suspense retry).
- Examples of lanes (names from React source): `SyncLane` (discrete events like click), `InputContinuousLane` (mouse move, scroll), `DefaultLane` (setState in timeouts, network), `TransitionLanes` (a group of ~16 lanes used by `startTransition`), `RetryLanes` (Suspense), `IdleLane`, `OffscreenLane`.
- **Assigning**: `requestUpdateLane(fiber)` looks at the context: inside `startTransition` → a transition lane; else `getCurrentUpdatePriority()`; else the priority of the current window event (`click` → Discrete → SyncLane).
- **Picking work**: `getNextLanes(root, wipLanes)` chooses the highest-priority non-empty lane group from `pendingLanes`, excluding suspended ones. Bitwise: `lane = lanes & -lanes` gives the highest priority (lowest bit).
- **Starvation**: `markStarvedLanesAsExpired` promotes lanes that have waited too long so they're rendered synchronously/without yielding.
- **Entanglement**: related lanes (e.g. updates in the same transition on the same queue) are rendered together so state stays consistent.
- **Render phase**: updates that don't belong to the current render lanes are skipped but remain in the hook queue (rebasing), so a high-priority render shows the urgent update with old low-priority state, and the later render re-applies everything in original order.
- **Scheduler**: sync lanes flush in a microtask (or at the end of the event); concurrent lanes are scheduled through `scheduler` (`MessageChannel`), with timeouts: Immediate -1ms, UserBlocking 250ms, Normal 5s, Low 10s, Idle never.

**Code example or diagram**

```
SyncLane            0b0000000000000000000000000000010
InputContinuousLane 0b0000000000000000000000000001000
DefaultLane         0b0000000000000000000000000100000
TransitionLane1     0b0000000000000000000000001000000
...
highest = lanes & -lanes      // isolate lowest set bit
merge   = a | b ; subset = (set & subset) === subset
```

```jsx
function Search() {
  const [text, setText] = useState('');
  const [query, setQuery] = useState('');
  const [isPending, startTransition] = useTransition();
  return (
    <input
      value={text}
      onChange={e => {
        setText(e.target.value);                 // SyncLane (discrete event)
        startTransition(() => setQuery(e.target.value)); // Transition lane, interruptible
      }}
    />
  );
}
```

**Pitfalls / follow-ups**
- Lanes are internal; you influence them only via `startTransition`, `useDeferredValue`, `flushSync`, and event type.
- Don't confuse React's lanes with Scheduler priorities; there is a mapping (`lanesToEventPriority`).
- Follow-up: why bitmasks? O(1) set operations on many priorities. Follow-up: what is starvation protection?

---

### Q176. What is time slicing and how does it keep the UI responsive?

**Short answer**
In concurrent rendering React does the render phase in small chunks (~5 ms). After each unit of work (one fiber) it checks `shouldYield()`; if its time slice is used up it returns control to the browser, then continues in a new macrotask. This lets input, paint and animations run between chunks.

**Detailed explanation**
- The render phase is a loop: `while (workInProgress !== null && !shouldYield()) performUnitOfWork(workInProgress)`. Legacy/sync lanes use `workLoopSync` without the yield check.
- Because the render phase is pure (no DOM mutation), it can be paused, resumed, or thrown away. The **commit phase** (DOM mutations, layout effects) is synchronous and not sliceable.
- Scheduler yields using `MessageChannel` (macrotask, not clamped like `setTimeout`) and a `frameInterval` of 5ms. It also may use `navigator.scheduling.isInputPending()` in older versions.
- If a higher-priority update arrives while a low-priority render is in progress, React interrupts, discards the WIP tree (`prepareFreshStack`) and starts the urgent render, then restarts the low-priority one.
- Time slicing only applies to *transition/default concurrent* lanes when using `createRoot`. Sync updates (clicks, `flushSync`) aren't sliced.
- Single component rendering for 200 ms can't be sliced — granularity is one fiber. Slicing helps lots of moderately-sized components.

**Code example or diagram**

```
legacy:  |=========== render 120ms ===========|commit|   <- input blocked
sliced:  |5ms|yield|5ms|yield|5ms| ... |commit|          <- input handled in gaps
```

```jsx
import { useState, useDeferredValue, memo } from 'react';

const SlowList = memo(function SlowList({ text }) {
  const items = [];
  for (let i = 0; i < 500; i++) items.push(<SlowItem key={i} text={text} />);
  return <ul>{items}</ul>;
});
function SlowItem({ text }) {
  const t = performance.now();
  while (performance.now() - t < 1) {} // artificial 1ms each
  return <li>Result for {text}</li>;
}

export default function App() {
  const [text, setText] = useState('');
  const deferred = useDeferredValue(text); // lower-priority, interruptible render
  return (
    <>
      <input value={text} onChange={e => setText(e.target.value)} />
      <SlowList text={deferred} />
    </>
  );
}
```

**Pitfalls / follow-ups**
- Without `memo`, `SlowList` re-renders in the urgent pass as well, defeating the deferral.
- Effects/render may run more than once or be discarded: render must be pure.
- Follow-up: how is it different from debouncing? (Debounce delays; slicing doesn't delay, it interrupts, and adapts to device speed.)
- Follow-up: why `MessageChannel` rather than `requestIdleCallback`? (rIC fires inconsistently, e.g. in background tabs/Safari.)

---

### Q177. How does React decide to bail out of rendering?

**Short answer**
React skips work when it can prove nothing changed: same state (`Object.is`) for a state update, same element reference with no pending work/context change for a child (`oldProps === newProps`), `React.memo`/`PureComponent` shallow-equal props, `shouldComponentUpdate` returning false, or no pending lanes in a subtree (`childLanes` empty).

**Detailed explanation**
In `beginWork(current, workInProgress, renderLanes)`:
1. If `oldProps !== newProps` or legacy context changed → `didReceiveUpdate = true` (must render).
2. Otherwise check if this fiber has scheduled work in `renderLanes` (`checkScheduledUpdateOrContext`). If not → `bailoutOnAlreadyFinishedWork`.
3. In the bailout, if `workInProgress.childLanes` doesn't include renderLanes, return `null` — the **whole subtree is skipped**. Otherwise clone children and continue (this component bails but descendants have pending updates).

Cases:
- **State bail-out**: `setState(sameValue)` — eager-state check in `dispatchSetState`; if the fiber has no other pending lanes and `Object.is(eagerState, currentState)` no render is scheduled. Otherwise, the component renders once more, and in `updateReducer` if the state is unchanged `didReceiveUpdate` stays false and `bailoutHooks` skips children/effects.
- **Same element reference**: `props` object identity is the same when a parent returns the same JSX element (e.g. `children` passed through, or a hoisted/memoized element). The child is skipped even though the parent re-rendered.
- **`memo`**: compares prev and next props shallowly (`shallowEqual`, or custom compare) and bails out if equal and ref same.
- **Context**: consumers whose context value changed get lanes marked by `propagateContextChange` so they aren't bailed out.
- **React Compiler** auto-memoizes values/JSX to create more of these referential-equality bailouts.

**Code example or diagram**

```jsx
function Parent() {
  const [n, setN] = useState(0);
  return (
    <>
      <button onClick={() => setN(n + 1)}>{n}</button>
      <Expensive />            {/* new element each render -> Expensive re-renders */}
    </>
  );
}
// Fix 1: memo(Expensive)           (props shallow-equal -> bail out)
// Fix 2: move state into a child component, or pass <Expensive/> as `children`
function Wrapper({ children }) {
  const [n, setN] = useState(0);
  return <div onClick={() => setN(n + 1)}>{n}{children}</div>; // children element identity is stable
}
```

**Pitfalls / follow-ups**
- Inline objects/functions in props defeat `memo`.
- A bail-out of a component's output doesn't stop its effects from being compared—no render, no effects.
- Mutating state then `setState(sameRef)` will not re-render (a classic bug).
- Follow-up: difference between "render" and "commit" bail-out? (Even if a component renders, DOM updates only happen if the output differs.)

---

### Q178. How does the synthetic event system work (event delegation)?

**Short answer**
React attaches native listeners to the **root container** (not each node), for both capture and bubble phases. When a native event arrives, React finds the target fiber, collects handlers up the fiber tree (`onClick`, `onClickCapture`), and dispatches a `SyntheticEvent` wrapper — a cross-browser normalized event — calling those handlers in order, emulating capture/bubble over the React tree.

**Detailed explanation**
- Since React 17 listeners are on the root container (`createRoot(container)`), not `document`, enabling multiple React versions/roots to coexist. Registration happens when the root is created (`listenToAllSupportedEvents`).
- Flow: native event → `dispatchEvent` → find nearest fiber from `event.target` (via an internal key on DOM node) → `extractEvents` by plugin (SimpleEventPlugin, ChangeEventPlugin, EnterLeaveEventPlugin, SelectEventPlugin) → build ordered `dispatchQueue` (traversing fibers via `return` pointers, collecting `props.onX`) → execute listeners with the synthetic event.
- `SyntheticEvent` normalizes properties (`target`, `currentTarget`, `relatedTarget`, `key`, `button`) and offers `stopPropagation`, `preventDefault`, `isPropagationStopped`. `nativeEvent` gives the raw event. Pooling was removed in React 17 (no more `e.persist()` need).
- Events are tied to priorities: discrete (click, keydown) → SyncLane; continuous (mousemove, scroll) → InputContinuous. Updates inside a handler are batched (automatic batching in 18 for all contexts).
- Portals: events bubble through the **React tree**, not DOM tree.
- `onChange` for inputs maps to the native `input` event; `onMouseEnter/Leave` are synthesized from `mouseout/over`. `onScroll` doesn't bubble (since 17). `onFocus/onBlur` use `focusin/focusout`.
- `stopPropagation` in React stops React propagation and calls native stopPropagation at the root container — native listeners added on `document` still fire (they're above the root), while ones on ancestors of the root... are stopped only if they're beyond the root. Native listeners on inner DOM nodes fire *before* React's handlers.

**Code example or diagram**

```
click on <button>
  native capture @root ──► React capture handlers (onClickCapture) top→down
  native bubble  @root ──► React bubble handlers (onClick) bottom→up
```

```jsx
function App() {
  return (
    <div onClickCapture={() => console.log('1 capture parent')} onClick={() => console.log('4 bubble parent')}>
      <button
        onClickCapture={() => console.log('2 capture button')}
        onClick={e => { console.log('3 bubble button', e.nativeEvent instanceof MouseEvent); }}
      >Go</button>
    </div>
  );
}
```

**Pitfalls / follow-ups**
- Mixing native `addEventListener` on `document` with React handlers → ordering surprises; `e.stopPropagation()` in React doesn't prevent `document`-level listeners in React 17+ unless you use `e.nativeEvent.stopImmediatePropagation()` on a listener at the same root.
- Passive listeners: `onWheel/onTouchStart` are passive by default for root listeners, so `preventDefault` may warn.
- Follow-up: why move from `document` to the root? Multi-version embedding and gradual upgrades.

---

### Q179. How do keys affect component identity and state preservation?

**Short answer**
During reconciliation React matches old and new children by **type + key** (and position when no key). Same type and key at the same slot → same fiber → state preserved and props updated. Different key or type → unmount the old (state lost) and mount a new instance.

**Detailed explanation**
- State lives on the fiber, not in the component function. Identity = "this fiber survives".
- **Single child**: `reconcileSingleElement` compares `key` and `type`; if equal reuse fiber, else delete and create.
- **Arrays**: `reconcileChildrenArray` does a first pass by index while keys match; then builds a `Map<key|index, fiber>` of remaining old children to find moves. It tracks `lastPlacedIndex` so only elements that moved *forward-relative-to-others* get a `Placement` flag (a one-pass, no-reverse-iteration algorithm). Missing keys → fallback to index (with a dev warning).
- **Index as key** problem: when items are reordered/inserted/deleted, the fiber at index i is reused for a different item, so uncontrolled inputs, local state, focus, and animations get attached to the wrong data.
- **Type changes** at the same position (`<div>` → `<span>`, `<A/>` → `<B/>`) remount the whole subtree. Also, defining a component inside another component creates a new type every render → remount every render.
- Keys are only unique among siblings, aren't passed as props, and can be used on a single element (not only lists) to force identity change.
- Conditional rendering positions: `{cond ? <A/> : <A/>}` in the same slot keeps state; `{cond && <A/>}` with `false` placeholders keeps slot positions stable (that's why `null/false` take a slot).

**Code example or diagram**

```jsx
function Row({ label }) {
  const [text, setText] = useState('');
  return <li>{label}: <input value={text} onChange={e => setText(e.target.value)} /></li>;
}

function List({ items }) {
  return (
    <ul>
      {/* BAD: key={index}; deleting the first item shifts typed text to the wrong row */}
      {items.map(item => <Row key={item.id} label={item.name} />)}
    </ul>
  );
}
```

**Pitfalls / follow-ups**
- Never use `Math.random()` / `crypto.randomUUID()` in render as keys: remounts every render.
- Index keys are acceptable only for static, never reordered, never filtered lists with stateless items.
- Follow-up: Fragment keys — `<Fragment key={id}>` for keyed groups (the short `<>` can't take a key).
- Follow-up: What happens to effects on remount? Cleanup runs then setup runs again.

---

### Q180. How do you reset state with a key?

**Short answer**
Give the component a `key` derived from the identity of the data it represents (e.g. `userId`). When the key changes React unmounts the old instance and mounts a fresh one, so all state (and effects, refs) reset — no `useEffect` syncing needed.

**Detailed explanation**
- The recommended alternative to "reset state in an effect when a prop changes" (which causes an extra render with stale state visible and is bug-prone).
- Works for any subtree: the key on the topmost component whose whole state should reset.
- Reset only part of the state? Either split the component, or adjust state during render:
  `if (prevId !== id) { setPrevId(id); setSelection(null); }` (React re-renders immediately before committing, without a flash).
- Alternatively, store state keyed by id (`{[id]: draft}`) to *preserve* state per entity instead of reset.
- Also useful to restart animations, re-run mount effects (fetch on mount), or clear uncontrolled form inputs (`<form key={formVersion}>`).

**Code example or diagram**

```jsx
import { useState } from 'react';

function Profile({ userId }) {
  return <EditForm key={userId} userId={userId} />; // switching user remounts the form
}

function EditForm({ userId }) {
  const [draft, setDraft] = useState('');
  return (
    <label>
      Note for {userId}
      <input value={draft} onChange={e => setDraft(e.target.value)} />
    </label>
  );
}

// Reset a form after submit
function Contact() {
  const [version, setVersion] = useState(0);
  return (
    <form key={version} onSubmit={e => { e.preventDefault(); setVersion(v => v + 1); }}>
      <input name="msg" defaultValue="" />
      <button>Send</button>
    </form>
  );
}
```

**Pitfalls / follow-ups**
- Remounting is more expensive than updating; don't key large trees unnecessarily.
- The key must be on the *component that holds the state*, or its ancestor — keying the child of a state holder doesn't reset the holder.
- Follow-up: compare with `useEffect(() => setX(initial), [id])` — extra render, stale flash, and Strict Mode noise.

---

### Q181. What is an effect event / useEffectEvent?

**Short answer**
`useEffectEvent` (stable in React 19.2) wraps a function so it always sees the **latest props/state** but isn't a reactive dependency. Use it inside Effects for logic that should *read* fresh values without *re-triggering* the effect (e.g. log with current `theme` when a `roomId` connection event happens).

**Detailed explanation**
- Problem: effects mix *reactive* logic (should re-run when `roomId` changes) with *non-reactive* logic (what to do with `theme` when connected). Listing `theme` in deps reconnects on every theme change; omitting it gives a stale closure and lint errors.
- `const onConnected = useEffectEvent(() => showNotification('Connected', theme));` Then `useEffect(() => { conn.on('connected', () => onConnected()); ... }, [roomId])`. The lint rule knows effect events need not be in deps.
- Implementation idea: a stable function identity whose body is updated to the latest closure in a (layout-phase) commit, similar to the old "latest ref" pattern:
  `const ref = useRef(fn); useInsertionEffect(() => { ref.current = fn; }); return useCallback((...a) => ref.current(...a), []);`
- Rules: call only from inside Effects (or other effect events); don't pass them to children or other hooks, or put them in dependency arrays. It's not a general event-handler replacement (`useCallback`-ish for props).
- Don't use it to hide dependencies you *do* want to react to.

**Code example or diagram**

```jsx
import { useEffect, useEffectEvent } from 'react';

function ChatRoom({ roomId, theme }) {
  const onConnected = useEffectEvent(() => {
    showNotification('Connected!', theme); // always latest theme
  });

  useEffect(() => {
    const conn = createConnection(roomId);
    conn.on('connected', onConnected);
    conn.connect();
    return () => conn.disconnect();
  }, [roomId]); // theme not needed, onConnected isn't a dependency
}

// Pre-19.2 userland polyfill
function useLatest(fn) {
  const ref = useRef(fn);
  useInsertionEffect(() => { ref.current = fn; });
  return useCallback((...args) => ref.current(...args), []);
}
```

**Pitfalls / follow-ups**
- Calling during render throws/warns; the function identity changes per render intentionally so it must not be depended on.
- Requires `eslint-plugin-react-hooks` ≥ 6 to understand it.
- Follow-up: How is this different from `useRef` for latest value? Same idea, but standardised, with lint support and correct timing under concurrent rendering.

---

### Q182. What are tearing and concurrent-safe external stores?

**Short answer**
**Tearing** is a visual inconsistency where different parts of the UI show different values of the same external (non-React) store in one rendered frame, possible because concurrent rendering yields mid-render and the store can mutate in between. `useSyncExternalStore` (uSES) fixes it by making the render consistent: it reads a snapshot, re-checks during/after render, and if the store changed forces a synchronous re-render.

**Detailed explanation**
- React state is immutable per render lane, so React-owned state is tear-free. A mutable external store (Redux, Zustand, `window.matchMedia`, `localStorage`) can change *between* time slices, so component A (rendered early) shows v1 and component B (rendered after a yield) shows v2.
- `useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot?)`:
  - `getSnapshot` must return an **immutable/cached** value; same store state → same reference (otherwise infinite loop warning "getSnapshot should be cached").
  - React compares snapshots with `Object.is`; on subscription callback it schedules a sync update.
  - After render, in the commit phase it checks for snapshot changes (consistency check); if tearing detected, React re-renders **synchronously**, trading away time slicing for those updates. Updates from uSES stores can't be marked as transitions.
  - `getServerSnapshot` ensures hydration consistency (SSR value ≠ client store value avoided).
- Libraries (Redux 8+/react-redux, Zustand, Jotai, Valtio) use it internally.

**Code example or diagram**

```jsx
import { useSyncExternalStore } from 'react';

// minimal store
function createStore(initial) {
  let state = initial;
  const listeners = new Set();
  return {
    get: () => state,
    set(next) { state = typeof next === 'function' ? next(state) : next; listeners.forEach(l => l()); },
    subscribe(l) { listeners.add(l); return () => listeners.delete(l); },
  };
}
const counter = createStore({ n: 0 });

function useStore(store, selector = s => s) {
  return useSyncExternalStore(store.subscribe, () => selector(store.get()), () => selector(store.get()));
}

function Count() {
  const n = useStore(counter, s => s.n); // selector returns primitive -> stable
  return <button onClick={() => counter.set(s => ({ ...s, n: s.n + 1 }))}>{n}</button>;
}
```

**Pitfalls / follow-ups**
- Selector returning a new object each call (`s => ({a: s.a})`) → infinite re-render; memoize or return primitives.
- `subscribe` must be a stable function (define outside component or `useCallback`) or it resubscribes every render.
- Follow-up: why not `useEffect + useState` subscription? Tearing + a flash because the initial value could be stale between render and effect.

---

### Q183. What is the difference between Object.is comparison and deep equality in React?

**Short answer**
React uses **referential/identity comparison** via `Object.is` for state bail-outs, dependency arrays, and `useSyncExternalStore` snapshots, and **shallow** equality (one level of `Object.is` over keys) for `memo`/`PureComponent`. It never does deep equality, because deep compare is O(size of data), can be cyclic, and the immutability convention makes reference checks sufficient.

**Detailed explanation**
- `Object.is(a, b)`: like `===` except `Object.is(NaN, NaN) === true` and `Object.is(0, -0) === false`. Objects/arrays/functions compare by reference.
- Where used: `useState/useReducer` bail-out, `useMemo/useCallback/useEffect` deps, `useSyncExternalStore`, context value changes (`Object.is(old, new)`), `useDeferredValue`.
- `shallowEqual(a, b)`: same key set and `Object.is(a[k], b[k])` for each key. Used by `React.memo` default and `PureComponent`.
- Because of this, immutable updates are required: `setState(prev => ({...prev, x: 1}))` creates a new reference; mutating in place and setting the same reference doesn't re-render.
- Structural sharing (Immer, Redux Toolkit, React Query) keeps unchanged sub-objects referentially equal so memoization remains effective.
- Deep equality (`lodash.isEqual`) is sometimes passed as a custom `memo` comparator or used in `useMemo` for small structures, but is usually a smell; costs can exceed the render being avoided.

**Code example or diagram**

```jsx
Object.is(NaN, NaN);        // true   (=== false)
Object.is(0, -0);           // false  (=== true)
Object.is({a:1}, {a:1});    // false  different references

function shallowEqual(a, b) {
  if (Object.is(a, b)) return true;
  if (typeof a !== 'object' || !a || typeof b !== 'object' || !b) return false;
  const ka = Object.keys(a), kb = Object.keys(b);
  if (ka.length !== kb.length) return false;
  return ka.every(k => Object.prototype.hasOwnProperty.call(b, k) && Object.is(a[k], b[k]));
}

// Effect fires every render because the object is new each time:
useEffect(() => { fetchData(options); }, [options]);      // options = { page } created inline
useEffect(() => { fetchData({ page }); }, [page]);        // fix: depend on primitives
```

**Pitfalls / follow-ups**
- Mutating arrays (`push`, `sort`) then `setItems(items)` → no update.
- Deep equality in deps arrays isn't supported; use primitives, `useMemo`, or serialize (`JSON.stringify`) deliberately.
- Follow-up: why NaN case matters: state `NaN` set repeatedly wouldn't loop with `Object.is`.

---

### Q184. What is the cost of context value changes?

**Short answer**
When a Provider's `value` changes (by `Object.is`), React re-renders **every consumer** of that context (every `useContext(Ctx)` caller) regardless of which part of the value they use, even if intermediate components are memoized. There is no selector support. Cost = (number of consumers) × (cost of their renders + subtrees).

**Detailed explanation**
- Mechanism: `updateContextProvider` compares old/new value; if changed, `propagateContextChange` walks the Provider's subtree fibers looking for consumers (stored in `fiber.dependencies`) and marks their lanes (and parents' `childLanes`). Memo/bail-out doesn't protect consumers because the lane is set on them.
- The walk itself is O(subtree size) in the worst case.
- A new object literal as value (`value={{user, setUser}}`) changes every parent render → all consumers re-render each time the provider's parent renders.
- Mitigations:
  1. Memoize value with `useMemo`.
  2. **Split contexts** (state vs dispatch; rarely-changing vs frequently-changing).
  3. Put state in a component lower and pass `children` (children elements are stable, so only consumers re-render, not the whole tree).
  4. Context + `useReducer`; `dispatch` is stable.
  5. External store + `useSyncExternalStore` with selectors (Zustand, Redux, `use-context-selector`) for fine-grained subscription.
  6. `React.memo` on a child component that reads context and passes values as props to memoized grand-children ("context reader" pattern).
- Default value only used when no Provider exists.

**Code example or diagram**

```jsx
const StateCtx = createContext(null);
const DispatchCtx = createContext(null);

function Provider({ children }) {
  const [state, dispatch] = useReducer(reducer, { count: 0, theme: 'light' });
  const stateValue = useMemo(() => state, [state]);   // already stable per state; shown for clarity
  return (
    <DispatchCtx.Provider value={dispatch}>   {/* never changes -> buttons that only dispatch don't re-render */}
      <StateCtx.Provider value={stateValue}>{children}</StateCtx.Provider>
    </DispatchCtx.Provider>
  );
}
function IncrementButton() {
  const dispatch = useContext(DispatchCtx); // not re-rendered by state changes
  return <button onClick={() => dispatch({ type: 'inc' })}>+</button>;
}
```

**Pitfalls / follow-ups**
- A consumer reading only `state.theme` still re-renders on `state.count` change.
- `useContext` can be called conditionally (and after early return) when using the `use(Context)` API.
- Follow-up: how would you measure? React DevTools Profiler "why did this render" → "Context changed".

---

### Q185. How would you implement a mini React (createElement, render, hooks)?

**Short answer**
Three pieces: `createElement` builds plain objects (`{type, props:{children}}`); a **fiber-style render** turns them into DOM (here with a simple work loop, reconciliation with diffing, and a commit phase); **hooks** are stored in an array/list on the current fiber indexed by call order. This is the "Didact" approach.

**Detailed explanation**
1. `createElement(type, props, ...children)` → element tree; text becomes `TEXT_ELEMENT`.
2. `createDom(fiber)` creates DOM nodes, `updateDom` diffs props (events `on*`, removed, added).
3. **Render phase**: `performUnitOfWork(fiber)` handles function components (calling them with hooks context set) or host components; `reconcileChildren` compares old fiber children with new elements: same type → UPDATE, new type → PLACEMENT, leftover old → DELETION (collected in `deletions`).
4. **Commit phase**: after the whole tree is built, apply effects to DOM in one sync pass (so users never see half-updated UI).
5. **Work loop** uses `requestIdleCallback`/`MessageChannel` to yield (time slicing).
6. **Hooks**: global `wipFiber` and `hookIndex`; `useState` reads `wipFiber.alternate.hooks[hookIndex]`, queues actions; `setState` sets a new `wipRoot` and restarts rendering.

**Code example or diagram**

```jsx
// mini-react.js  (runnable in a browser module / bundler)
function createElement(type, props, ...children) {
  return {
    type,
    props: { ...props, children: children.flat().map(c => (typeof c === 'object' && c !== null ? c : createTextElement(c))) },
  };
}
function createTextElement(text) {
  return { type: 'TEXT_ELEMENT', props: { nodeValue: text, children: [] } };
}

const isEvent = k => k.startsWith('on');
const isProp = k => k !== 'children' && !isEvent(k);
const isNew = (p, n) => k => p[k] !== n[k];
const isGone = (p, n) => k => !(k in n);

function createDom(fiber) {
  const dom = fiber.type === 'TEXT_ELEMENT' ? document.createTextNode('') : document.createElement(fiber.type);
  updateDom(dom, {}, fiber.props);
  return dom;
}
function updateDom(dom, prev, next) {
  Object.keys(prev).filter(isEvent).filter(k => !(k in next) || isNew(prev, next)(k)).forEach(k =>
    dom.removeEventListener(k.slice(2).toLowerCase(), prev[k]));
  Object.keys(prev).filter(isProp).filter(isGone(prev, next)).forEach(k => { dom[k] = ''; });
  Object.keys(next).filter(isProp).filter(isNew(prev, next)).forEach(k => { dom[k] = next[k]; });
  Object.keys(next).filter(isEvent).filter(isNew(prev, next)).forEach(k =>
    dom.addEventListener(k.slice(2).toLowerCase(), next[k]));
}

let nextUnitOfWork = null, currentRoot = null, wipRoot = null, deletions = null;

function render(element, container) {
  wipRoot = { dom: container, props: { children: [element] }, alternate: currentRoot };
  deletions = [];
  nextUnitOfWork = wipRoot;
}

function commitRoot() {
  deletions.forEach(commitWork);
  commitWork(wipRoot.child);
  currentRoot = wipRoot;
  wipRoot = null;
}
function commitWork(fiber) {
  if (!fiber) return;
  let parentFiber = fiber.parent;
  while (!parentFiber.dom) parentFiber = parentFiber.parent;
  const parentDom = parentFiber.dom;
  if (fiber.effectTag === 'PLACEMENT' && fiber.dom) parentDom.appendChild(fiber.dom);
  else if (fiber.effectTag === 'UPDATE' && fiber.dom) updateDom(fiber.dom, fiber.alternate.props, fiber.props);
  else if (fiber.effectTag === 'DELETION') { commitDeletion(fiber, parentDom); return; }
  commitWork(fiber.child);
  commitWork(fiber.sibling);
}
function commitDeletion(fiber, parentDom) {
  if (fiber.dom) parentDom.removeChild(fiber.dom);
  else commitDeletion(fiber.child, parentDom);
}

function workLoop(deadline) {
  let shouldYield = false;
  while (nextUnitOfWork && !shouldYield) {
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
    shouldYield = deadline.timeRemaining() < 1;
  }
  if (!nextUnitOfWork && wipRoot) commitRoot();
  requestIdleCallback(workLoop);
}
requestIdleCallback(workLoop);

function performUnitOfWork(fiber) {
  if (fiber.type instanceof Function) updateFunctionComponent(fiber);
  else updateHostComponent(fiber);
  if (fiber.child) return fiber.child;
  let next = fiber;
  while (next) {
    if (next.sibling) return next.sibling;
    next = next.parent;
  }
  return null;
}

let wipFiber = null, hookIndex = null;
function updateFunctionComponent(fiber) {
  wipFiber = fiber; hookIndex = 0; wipFiber.hooks = [];
  reconcileChildren(fiber, [fiber.type(fiber.props)]);
}
function updateHostComponent(fiber) {
  if (!fiber.dom) fiber.dom = createDom(fiber);
  reconcileChildren(fiber, fiber.props.children);
}

function reconcileChildren(wip, elements) {
  let index = 0, oldFiber = wip.alternate && wip.alternate.child, prevSibling = null;
  while (index < elements.length || oldFiber) {
    const element = elements[index];
    const sameType = oldFiber && element && element.type === oldFiber.type;
    let newFiber = null;
    if (sameType) newFiber = { type: oldFiber.type, props: element.props, dom: oldFiber.dom, parent: wip, alternate: oldFiber, effectTag: 'UPDATE' };
    if (element && !sameType) newFiber = { type: element.type, props: element.props, dom: null, parent: wip, alternate: null, effectTag: 'PLACEMENT' };
    if (oldFiber && !sameType) { oldFiber.effectTag = 'DELETION'; deletions.push(oldFiber); }
    if (oldFiber) oldFiber = oldFiber.sibling;
    if (index === 0) wip.child = newFiber; else if (element && prevSibling) prevSibling.sibling = newFiber;
    prevSibling = newFiber;
    index++;
  }
}

function useState(initial) {
  const oldHook = wipFiber.alternate && wipFiber.alternate.hooks && wipFiber.alternate.hooks[hookIndex];
  const hook = { state: oldHook ? oldHook.state : initial, queue: [] };
  (oldHook ? oldHook.queue : []).forEach(action => { hook.state = typeof action === 'function' ? action(hook.state) : action; });
  const setState = action => {
    hook.queue.push(action);
    wipRoot = { dom: currentRoot.dom, props: currentRoot.props, alternate: currentRoot };
    nextUnitOfWork = wipRoot;
    deletions = [];
  };
  wipFiber.hooks.push(hook);
  hookIndex++;
  return [hook.state, setState];
}

export const MiniReact = { createElement, render, useState };

// usage (with /** @jsx MiniReact.createElement */ pragma)
// function Counter() { const [c, setC] = MiniReact.useState(0); return <button onClick={() => setC(x => x + 1)}>Count {c}</button>; }
// MiniReact.render(<Counter />, document.getElementById('root'));
```

**Pitfalls / follow-ups**
- Missing: keys (so reorders remount), effects, context, batching, priorities, Suspense, error boundaries, portals, fragments.
- Bug-prone: queued actions in `hook.queue` replay — this version drops actions that arrive between the `hook` creation and the next render; real React keeps updates on a persistent queue object.
- Follow-up: where would you add `useEffect`? Collect effects during render, run after commit (passive after paint).
- Follow-up: how to add keyed reconciliation? Map old children by key.

---

### Q186. How would you implement your own useState, useEffect or memo?

**Short answer**
Keep per-component storage in an array indexed by a call counter that resets every render: `useState` stores the value and returns a setter that schedules a re-render; `useEffect` stores previous deps and runs the callback (plus prior cleanup) when deps change, after render; `memo` caches the last props/result and re-uses it if a shallow compare says props are equal.

**Detailed explanation**
- A single global "component instance" with `hooks[]` and `idx` is enough to demonstrate; real React keeps them per fiber.
- `useState` must use the **functional update**, `Object.is` bail-out, and a stable setter.
- `useEffect`: deps `undefined` → every render; `[]` → once; otherwise shallow compare with `Object.is`. Run cleanups before the next effect and on unmount. Effects are *deferred* until after the DOM is updated (here, queued then flushed after render).
- `memo(Component, areEqual = shallowEqual)` returns a wrapper component; with hooks you can implement it with `useRef` for last props/last element.
- `useMemo`/`useCallback`/`useRef` follow the same slot pattern.

**Code example or diagram**

```jsx
// Self-contained demo: a tiny hooks runtime with real React DOM-free rendering to a string target.
const instance = { hooks: [], idx: 0, effects: [], render: null };

function useState(initial) {
  const i = instance.idx++;
  if (!(i in instance.hooks)) instance.hooks[i] = typeof initial === 'function' ? initial() : initial;
  const setState = action => {
    const prev = instance.hooks[i];
    const next = typeof action === 'function' ? action(prev) : action;
    if (Object.is(prev, next)) return;
    instance.hooks[i] = next;
    instance.render();
  };
  return [instance.hooks[i], setState];
}

function useEffect(cb, deps) {
  const i = instance.idx++;
  const prev = instance.hooks[i]; // { deps, cleanup }
  const changed = !prev || !deps || !prev.deps || deps.some((d, k) => !Object.is(d, prev.deps[k]));
  if (changed) {
    instance.effects.push(() => {
      if (prev && prev.cleanup) prev.cleanup();
      const cleanup = cb();
      instance.hooks[i] = { deps, cleanup };
    });
    if (!prev) instance.hooks[i] = { deps, cleanup: undefined };
  }
}

function useMemo(factory, deps) {
  const i = instance.idx++;
  const prev = instance.hooks[i];
  if (prev && deps && deps.every((d, k) => Object.is(d, prev.deps[k]))) return prev.value;
  const value = factory();
  instance.hooks[i] = { deps, value };
  return value;
}

function mount(Component, props, output) {
  instance.render = () => {
    instance.idx = 0;
    instance.effects = [];
    output(Component(props));
    instance.effects.forEach(run => run()); // flush effects after "commit"
  };
  instance.render();
}

// memo with shallow comparison
function shallowEqual(a, b) {
  const ka = Object.keys(a), kb = Object.keys(b);
  return ka.length === kb.length && ka.every(k => Object.is(a[k], b[k]));
}
function memo(Component, areEqual = shallowEqual) {
  let lastProps = null, lastResult = null;
  return function Memoized(props) {
    if (lastProps && areEqual(lastProps, props)) return lastResult;
    lastProps = props;
    lastResult = Component(props);
    return lastResult;
  };
}
// NOTE: the memo above uses closure state and skips Component's hooks if cached; fine for pure
// presentational components only. In real React, memo is handled by the reconciler (bail-out).

// Demo
function Counter() {
  const [n, setN] = useState(0);
  useEffect(() => { console.log('effect: n =', n); return () => console.log('cleanup: n =', n); }, [n]);
  globalThis.inc = () => setN(c => c + 1);
  return `count=${n}`;
}
mount(Counter, {}, out => console.log(out));
globalThis.inc(); // count=1 with cleanup then effect
```

**Pitfalls / follow-ups**
- Single global instance doesn't support multiple components; real implementations keep hooks per fiber.
- Skipping `Component(props)` in memo means hooks inside can't run → in real React, memo bails out at the fiber level and hooks stay intact.
- Stale closure: effect callback captures render-time values; that's why deps matter.
- Follow-up: implement `useRef` (`hooks[i] ||= {current: init}`) and `useReducer` (state + dispatch with reducer).

---

### Q187. How do Server Components serialize the RSC payload?

**Short answer**
Server Components render on the server into a **streamed, line-based serialization format (the "Flight" protocol)** — not HTML. It describes the React element tree as JSON-like rows, with references to Client Components as *module references* (chunk ids + export names) and *holes* for async/Suspense data that arrive later in the stream. The client's React runtime parses it and reconstructs elements, merging with existing client state.

**Detailed explanation**
- Output is a stream of rows: `<id>:<type?><json>\n`. Row 0 is the root model. Examples (simplified):
  - `1:I["app/Counter.js",["chunk-123.js"],"default"]` → an **I**mport row (client module reference).
  - `0:["$","div",null,{"children":[["$","h1",null,{"children":"Hello"}],["$","$L1",null,{"initial":5}]]}]` → element tuples `["$", type, key, props]`; `$L1` is a lazy reference to row 1 (client component).
  - `2:"$Sreact.suspense"` / `$@3` for promises, `$undefined`, `$D...` (Date), `$n...` (BigInt), `$S` symbols, `$F` server-function refs, `$Q` Map etc.
  - `3:{"data":...}` resolved later: the stream is flushed progressively as Suspense boundaries finish.
- Server Components themselves **don't appear** — their output (host elements and client refs) does. Their code and dependencies never ship to the client.
- **Props must be serializable**: primitives, plain objects/arrays, Date, Map/Set, typed arrays, Promises (consumed with `use`), JSX, and Server Function references. No functions (except Server Actions), classes, or DOM nodes.
- On the client, `createFromReadableStream` / `createFromFetch` parses the rows; client references are resolved via the bundler manifest (webpack/turbopack) and loaded lazily; React renders it as normal elements, reconciling with existing state (e.g. navigation keeps client state).
- With SSR: the same RSC stream is consumed by the server to render HTML (initial load) while also being inlined in `<script>` tags for hydration. Navigation fetches just the RSC payload.
- Security: because the payload is consumed on the client and Server Function IDs are in it, only expose intended data; treat actions as public endpoints (a Flight deserialization vulnerability was patched in React 19.x — keep React updated).

**Code example or diagram**

```jsx
// Server Component (app/page.js)
import Counter from './Counter'; // 'use client'
export default async function Page() {
  const posts = await db.posts();
  return (
    <main>
      <h1>Posts: {posts.length}</h1>
      <Counter initial={posts.length} />
    </main>
  );
}
```

```
Illustrative Flight stream (format varies by version):
1:I["./Counter.js",["static/chunks/counter.js"],"default"]
0:["$","main",null,{"children":[["$","h1",null,{"children":["Posts: ",3]}],["$","$L1",null,{"initial":3}]]}]
```

**Pitfalls / follow-ups**
- The exact wire format is an implementation detail and changes between versions; interviewers care about the *concepts*: streaming, module references, lazy holes, serializable props.
- Passing a function prop from server to client component errors at runtime.
- Follow-up: Difference between RSC payload and SSR HTML? (HTML for fast first paint/hydration; payload for the component tree/state-preserving updates.)
- Follow-up: what's `'use server'`? It marks Server Functions, serialized as references that the client calls via POST.
