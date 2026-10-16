# React Interview Guide — 06 Performance

## Table of Contents
- [Q86. Profiling React apps](#q86)
- [Q87. Finding and fixing unnecessary re-renders](#q87)
- [Q88. Code splitting and lazy loading](#q88)
- [Q89. Virtualizing large lists](#q89)
- [Q90. Debounce and throttle](#q90)
- [Q91. Cost of inline functions and objects](#q91)
- [Q92. React Compiler](#q92)
- [Q93. Images, fonts and bundle size](#q93)
- [Q94. Core Web Vitals](#q94)
- [Q95. Memory leaks](#q95)
- [Q96. Context and children pattern](#q96)
- [Q97. Web workers](#q97)
- [Q98. useTransition for expensive renders](#q98)

---

<a id="q86"></a>
### Q86. How do you profile React apps (React DevTools Profiler)?

**Short answer**
Use the React DevTools Profiler tab: record an interaction, then inspect the flamegraph/ranked views to see which components rendered, how long they took, and *why* they rendered. Always profile a production-like build and confirm with the browser Performance panel. Fix only measured hot spots.

**Detailed explanation**
- **Profiler tab**: click Record, interact, stop. Commits are shown as bars; each commit lists components that re-rendered.
- **Flamegraph** shows render duration per component (grey = did not render in this commit). **Ranked** sorts by self time.
- Enable *"Record why each component rendered"* in settings to see reasons (props changed, hooks changed, parent rendered).
- **Highlight updates when components render** (Components settings) gives quick visual feedback.
- Development builds are much slower and run StrictMode double renders; use a production build with profiling (`react-dom/profiling`) for realistic numbers.
- The **`<Profiler>` component** gives programmatic timing (`actualDuration`, `baseDuration`) and can be sent to analytics.
- Complement with Chrome Performance panel (long tasks, INP), Lighthouse, and the React Performance Tracks in newer React versions.

**Code example or diagram**
```jsx
import { Profiler } from "react";

function onRender(id, phase, actualDuration, baseDuration, startTime, commitTime) {
  if (actualDuration > 16) {
    navigator.sendBeacon("/metrics", JSON.stringify({ id, phase, actualDuration }));
  }
}

export default function App() {
  return (
    <Profiler id="Dashboard" onRender={onRender}>
      <Dashboard />
    </Profiler>
  );
}
```

**Common pitfalls / follow-up questions**
- *Why is profiling in dev misleading?* Dev adds warnings, double-invocation and unminified code; timings are inflated and uneven.
- *`actualDuration` vs `baseDuration`?* Actual is time spent rendering this commit (with memoization benefits); base is estimated worst-case cost of rendering the whole subtree without memoization.
- *Is `<Profiler>` free in production?* No, it is disabled in standard production builds unless you use the profiling build.

---

<a id="q87"></a>
### Q87. How do you find and fix unnecessary re-renders?

**Short answer**
Detect them with the Profiler's "why did this render" and highlight-updates, then fix by moving state down, lifting content up (children), stabilizing props, and using `memo`/`useMemo`/`useCallback` where measured. A re-render is only a problem if it is slow or frequent.

**Detailed explanation**
A component re-renders when its state changes, its parent re-renders, or a consumed context changes. Typical causes: state held too high, new object/array/function props each render, a context value that changes identity, and derived state stored in effects.

Fixes, in order of preference:
1. **Colocate state** – move it into the smallest component that needs it.
2. **Lift content up** – pass expensive subtrees as `children`; they keep identity when the wrapper's state changes.
3. **Derive instead of sync** – compute during render rather than `useEffect` + `setState`.
4. **`React.memo`** for pure components with stable props.
5. **Stabilize props** with `useMemo`/`useCallback` (or let React Compiler do it).
6. **Split contexts** or use selector-based stores (Zustand, Redux `useSelector`).
7. Use stable `key`s, avoid index keys on dynamic lists.

**Code example or diagram**
```jsx
// Before: typing re-renders ExpensiveTree
function Page() {
  const [text, setText] = useState("");
  return (
    <>
      <input value={text} onChange={(e) => setText(e.target.value)} />
      <ExpensiveTree />
    </>
  );
}

// After: state colocated
function SearchBox() {
  const [text, setText] = useState("");
  return <input value={text} onChange={(e) => setText(e.target.value)} />;
}
function Page() {
  return (
    <>
      <SearchBox />
      <ExpensiveTree />
    </>
  );
}
```

**Common pitfalls / follow-up questions**
- *Does `memo` always help?* No; if props change every render (inline objects) it adds comparison cost with no benefit.
- *Does a re-render mean DOM updates?* No. Rendering is calling the function; React only commits diffs.
- *How to log reasons?* DevTools "Record why", or `why-did-you-render` in dev.

---

<a id="q88"></a>
### Q88. How do you do code splitting and lazy loading?

**Short answer**
Use dynamic `import()` with `React.lazy` and `<Suspense>` to load components on demand, typically per route and for heavy, rarely used widgets. Bundlers (Vite, webpack) emit separate chunks automatically. Frameworks like Next.js do route splitting for you.

**Detailed explanation**
- Split at **route level** first (biggest win), then at heavy components (charts, editors, modals).
- `React.lazy` requires a default export; wrap in `Suspense` with a fallback. Pair with an Error Boundary for chunk load failures.
- **Prefetch/preload** chunks on hover/idle to hide latency (`import()` on `onMouseEnter`, or `<link rel="prefetch">` / webpackPrefetch magic comments).
- Libraries can be split too: `const { default: _ } = await import("lodash-es")`.
- In RSC frameworks, Server Components are never sent to the client, and `next/dynamic` handles client-only components.
- Use a bundle analyzer to find what to split.

**Code example or diagram**
```jsx
import { lazy, Suspense } from "react";
import { createBrowserRouter, RouterProvider } from "react-router-dom";

const Reports = lazy(() => import("./pages/Reports"));
const preloadReports = () => import("./pages/Reports");

const router = createBrowserRouter([
  { path: "/", element: <Home /> },
  {
    path: "/reports",
    element: (
      <Suspense fallback={<Spinner />}>
        <Reports />
      </Suspense>
    ),
  },
]);

export const ReportsLink = () => (
  <a href="/reports" onMouseEnter={preloadReports}>Reports</a>
);

export default () => <RouterProvider router={router} />;
```

**Common pitfalls / follow-up questions**
- *Waterfalls?* Lazy chunk then data fetch is sequential; preload both or use route loaders.
- *Chunk fails to load after deploy?* Catch with an Error Boundary and reload; keep old assets for a while.
- *Named exports?* `lazy(() => import("./X").then(m => ({ default: m.X })))`.

---

<a id="q89"></a>
### Q89. How do you virtualize large lists (react-window)?

**Short answer**
Virtualization renders only the rows visible in the viewport (plus a small overscan) and positions them absolutely inside a tall spacer. Libraries: `react-window`, `react-virtuoso`, `@tanstack/react-virtual`. It keeps DOM node count constant regardless of list size.

**Detailed explanation**
Rendering thousands of DOM nodes costs memory, layout and paint. A virtualized list computes which indexes are visible from `scrollTop` and item size, renders just those, and offsets them. `FixedSizeList` needs constant item height; `VariableSizeList` needs a size function (or measurement via `react-virtuoso`/TanStack Virtual for dynamic heights). Row components should be memoized and receive data through `itemData`. Trade-offs: browser find-in-page doesn't see off-screen rows, accessibility needs `aria-rowcount`/`aria-setsize`, and scroll restoration and focus management need care. Alternatives: pagination, infinite scroll, CSS `content-visibility: auto`.

**Code example or diagram**
```jsx
import { FixedSizeList as List } from "react-window";
import { memo } from "react";

const Row = memo(({ index, style, data }) => (
  <div style={style} className="row">
    {data[index].name}
  </div>
));

export default function BigList({ items }) {
  return (
    <List
      height={600}
      width="100%"
      itemCount={items.length}
      itemSize={48}
      itemData={items}
      overscanCount={5}
    >
      {Row}
    </List>
  );
}
```

**Common pitfalls / follow-up questions**
- *Why must the `style` prop be applied?* It carries the absolute positioning; omitting it breaks layout.
- *Dynamic heights?* Use `VariableSizeList` with cached measurements or TanStack Virtual's `measureElement`.
- *Note:* react-window v2 changed the API (`List` with `rowComponent`/`rowProps`); check the installed version.

---

<a id="q90"></a>
### Q90. How do you debounce and throttle inputs?

**Short answer**
Debounce delays an action until the user stops triggering it for N ms (search-as-you-type); throttle runs it at most once per N ms (scroll, resize). In React, keep the timer/function stable across renders with `useRef`/`useMemo` or a hook, and clean up on unmount.

**Detailed explanation**
Creating `debounce(fn)` in the render body makes a new debounced function each render, defeating it. Two common approaches: debounce the **value** (`useDebouncedValue`) and trigger effects/queries from the debounced value, or debounce the **callback** with a stable instance. Also cancel pending calls on unmount and handle race conditions (use `AbortController` or a query library that ignores stale responses). `useDeferredValue` is a related React tool, but it is not time-based: it defers rendering, not network calls, so it does not replace debounce for requests.

**Code example or diagram**
```jsx
import { useEffect, useState, useMemo, useRef } from "react";

function useDebouncedValue(value, delay = 300) {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id);
  }, [value, delay]);
  return debounced;
}

function Search() {
  const [q, setQ] = useState("");
  const dq = useDebouncedValue(q);
  useEffect(() => {
    if (!dq) return;
    const ctrl = new AbortController();
    fetch(`/api/search?q=${encodeURIComponent(dq)}`, { signal: ctrl.signal });
    return () => ctrl.abort();
  }, [dq]);
  return <input value={q} onChange={(e) => setQ(e.target.value)} />;
}

function useThrottledCallback(fn, ms) {
  const last = useRef(0);
  const fnRef = useRef(fn);
  fnRef.current = fn;
  return useMemo(
    () => (...args) => {
      const now = Date.now();
      if (now - last.current >= ms) {
        last.current = now;
        fnRef.current(...args);
      }
    },
    [ms]
  );
}
```

**Common pitfalls / follow-up questions**
- *Debounce vs throttle use cases?* Debounce: search, autosave. Throttle: scroll, mousemove, resize.
- *Stale closures?* Store latest callback in a ref as above.
- *Should the input itself be debounced?* No; keep the input controlled and immediate, debounce only the side effect.

---

<a id="q91"></a>
### Q91. What is the cost of inline functions and objects?

**Short answer**
Creating an inline function or object is cheap; the real cost is that it has a new identity every render, which breaks `memo`, dependency arrays, and context value equality, causing extra renders or effect re-runs. Optimize only where identity matters.

**Detailed explanation**
JS engines create closures very quickly, so inline handlers on plain DOM elements (`<button onClick={() => ...}>`) are fine. Problems occur when passing them to: memoized children (`React.memo` compares shallowly), hook dependency arrays (`useEffect`, `useMemo`), and `Context.Provider value={{...}}` (all consumers re-render). Fix with `useCallback`/`useMemo`, hoisting constants outside the component, or the React Compiler, which memoizes automatically.

**Code example or diagram**
```jsx
const ROW_STYLE = { padding: 8 }; // hoisted: stable identity

const Row = memo(function Row({ item, onSelect, style }) {
  return <li style={style} onClick={() => onSelect(item.id)}>{item.name}</li>;
});

function List({ items }) {
  const [selected, setSelected] = useState(null);
  const onSelect = useCallback((id) => setSelected(id), []);
  return (
    <ul>
      {items.map((i) => (
        <Row key={i.id} item={i} onSelect={onSelect} style={ROW_STYLE} />
      ))}
    </ul>
  );
}
```

**Common pitfalls / follow-up questions**
- *Should I wrap every handler in useCallback?* No; it adds overhead and noise unless the receiver is memoized or it's a dependency.
- *Inline style objects?* Fine on DOM nodes; hoist when passed to memoized components.
- *Provider value?* Memoize: `useMemo(() => ({ user, logout }), [user, logout])`.

---

<a id="q92"></a>
### Q92. What is the React Compiler, and does it replace memo/useMemo?

**Short answer**
React Compiler is a build-time tool (Babel/SWC/Vite plugin) that automatically memoizes components and hooks by analyzing code that follows the Rules of React. It removes most need for manual `memo`, `useMemo` and `useCallback`, but they remain useful as escape hatches (e.g. controlling effect dependencies).

**Detailed explanation**
The compiler (stable as v1.0, 2025) rewrites components to cache values and JSX at fine granularity, even conditionally after early returns, which manual hooks cannot do. It requires pure render functions, no mutation of props/state, and hooks used correctly; code that breaks the rules is skipped (bails out) rather than broken. Adoption: install `babel-plugin-react-compiler`, run `eslint-plugin-react-hooks` compiler rules, use `"use no memo"` to opt a component out, and `compilationMode: "annotation"` for incremental rollout. Existing manual memoization can remain; removing it may change compiled output and effect behaviour, so remove carefully. It does not fix algorithmic costs, large lists, or bundle size.

**Code example or diagram**
```jsx
// vite.config.js
import react from "@vitejs/plugin-react";
export default {
  plugins: [
    react({ babel: { plugins: [["babel-plugin-react-compiler", {}]] } }),
  ],
};

// Written without manual memoization; compiler caches filtered and handler
function Products({ products, query }) {
  const filtered = products.filter((p) => p.name.includes(query));
  const onAdd = (id) => addToCart(id);
  return filtered.map((p) => <ProductCard key={p.id} p={p} onAdd={onAdd} />);
}
```

**Common pitfalls / follow-up questions**
- *When still use `useMemo`?* Expensive computation needing guaranteed stability, or a value used as an effect dependency where you need explicit control.
- *What if code mutates props?* The compiler skips that component; fix with lint feedback.
- *Does it need React 19?* Works with 17/18 via `react-compiler-runtime`/target config, best on 19.

---

<a id="q93"></a>
### Q93. How do you optimize images, fonts and bundle size?

**Short answer**
Serve modern, correctly sized, lazy-loaded images (AVIF/WebP, `srcset`), self-host subsetted fonts with `font-display: swap` and preload, and shrink JS via analysis, tree shaking, code splitting, and lighter dependencies. Measure with a bundle analyzer and Lighthouse.

**Detailed explanation**
**Images**: use `srcset`/`sizes`, `loading="lazy"` for below-the-fold, `fetchpriority="high"` for the LCP image, explicit `width`/`height` to avoid CLS, and a CDN/optimizer (`next/image`). **Fonts**: WOFF2, subset glyphs, `font-display: swap|optional`, preload critical fonts, use `size-adjust` fallback metrics or variable fonts. **Bundle**: analyze (`rollup-plugin-visualizer`, `webpack-bundle-analyzer`, `source-map-explorer`); import specific modules (`lodash-es`, date-fns instead of moment); enable tree shaking (ESM, `sideEffects: false`); split by route; remove duplicate packages; lean on Server Components; compress with Brotli; set long-lived cache headers on hashed assets.

**Code example or diagram**
```jsx
export function Hero() {
  return (
    <picture>
      <source type="image/avif" srcSet="/hero-800.avif 800w, /hero-1600.avif 1600w" sizes="100vw" />
      <img
        src="/hero-800.jpg"
        width="1600"
        height="900"
        alt="Product hero"
        fetchPriority="high"
        decoding="async"
      />
    </picture>
  );
}

// index.html
// <link rel="preload" as="font" type="font/woff2" href="/fonts/inter-var.woff2" crossorigin>
```

**Common pitfalls / follow-up questions**
- *Lazy-load the LCP image?* No; it delays LCP.
- *Why `crossorigin` on font preload?* Fonts are fetched in CORS mode; without it the preload is wasted.
- *Barrel imports hurt tree shaking?* See Q117.

---

<a id="q94"></a>
### Q94. What are Core Web Vitals (LCP, INP, CLS) and how does React affect them?

**Short answer**
LCP (Largest Contentful Paint, good ≤ 2.5s) measures loading, INP (Interaction to Next Paint, good ≤ 200ms) measures responsiveness, and CLS (Cumulative Layout Shift, good ≤ 0.1) measures visual stability. React affects them through JS bundle size and hydration (LCP), render/event-handler cost (INP), and late-loading content (CLS).

**Detailed explanation**
- **LCP**: client-only rendering delays content until JS runs. Use SSR/SSG/RSC, streaming, preload the hero image, and trim bundles.
- **INP** (replaced FID in March 2024): long tasks block the main thread after input. Use `useTransition`/`useDeferredValue`, split work, virtualize, move compute to workers, keep handlers light, and avoid heavy hydration (selective hydration, islands).
- **CLS**: reserve space for images, ads, and async content (skeletons with fixed dimensions), avoid inserting content above existing content, use font fallbacks with matched metrics.
Measure with the `web-vitals` library (field data/RUM) and Lighthouse/CrUX; lab and field differ.

**Code example or diagram**
```jsx
import { useEffect } from "react";
import { onLCP, onINP, onCLS } from "web-vitals";

export function useWebVitals() {
  useEffect(() => {
    const send = (m) => navigator.sendBeacon("/vitals", JSON.stringify(m));
    onLCP(send);
    onINP(send);
    onCLS(send);
  }, []);
}

function Avatar({ src }) {
  return <img src={src} width={48} height={48} alt="" />; // reserves space, prevents CLS
}
```

**Common pitfalls / follow-up questions**
- *Why did INP replace FID?* FID measured only first input delay; INP covers all interactions across the page lifecycle.
- *Does hydration affect INP?* Yes; unhydrated or busy pages respond late.
- *Lab vs field?* Lighthouse is synthetic; real users (CrUX/RUM) determine the ranking signal.

---

<a id="q95"></a>
### Q95. How do you detect and fix memory leaks?

**Short answer**
Leaks usually come from subscriptions, timers, listeners, or in-flight requests not cleaned up, and from closures or caches holding references. Detect with Chrome DevTools Memory heap snapshots/allocation timelines; fix by returning cleanup functions from effects.

**Detailed explanation**
Common sources: `setInterval`/`setTimeout`, `addEventListener`, WebSocket/`EventSource`, observers (`ResizeObserver`, `IntersectionObserver`), third-party widgets not destroyed, global stores or module-level caches that grow unbounded, detached DOM nodes retained by refs, and fetches that set state after unmount. Detection workflow: take a heap snapshot, perform the action (mount/unmount route) several times, force GC, take another snapshot, compare for growing "Detached" DOM nodes and retained instances; check the Performance monitor for rising JS heap. StrictMode's double-invoked effects in dev reveal missing cleanup. Fixes: cleanup functions, `AbortController`, `WeakMap`/LRU caches, unsubscribe from stores.

**Code example or diagram**
```jsx
function useLiveFeed(url) {
  const [msgs, setMsgs] = useState([]);
  useEffect(() => {
    const ws = new WebSocket(url);
    ws.onmessage = (e) => setMsgs((m) => [...m.slice(-199), JSON.parse(e.data)]);
    return () => ws.close();
  }, [url]);
  return msgs;
}

function Profile({ id }) {
  const [user, setUser] = useState(null);
  useEffect(() => {
    const ctrl = new AbortController();
    fetch(`/api/users/${id}`, { signal: ctrl.signal })
      .then((r) => r.json())
      .then(setUser)
      .catch((e) => e.name !== "AbortError" && console.error(e));
    return () => ctrl.abort();
  }, [id]);
  return <div>{user?.name}</div>;
}
```

**Common pitfalls / follow-up questions**
- *Does "setState on unmounted component" warning indicate a leak?* It was removed in React 18; the real issue is the retained work, not the warning.
- *How does StrictMode help?* It mounts, unmounts, and remounts in dev to surface missing cleanup.
- *Unbounded arrays in state?* Cap size (as above) or window the data.

---

<a id="q96"></a>
### Q96. How do you tune the context and children pattern to avoid re-renders?

**Short answer**
Every consumer re-renders when a context value changes identity, so memoize the value, split context by concern (state vs dispatch, fast vs slow-changing), and pass subtrees as `children` so they are not re-created when the provider's state changes.

**Detailed explanation**
When a Provider component re-renders because of its own state, its `children` prop is the same element reference created by the parent, so React skips re-rendering them unless they consume the context. This "children as props" pattern lets a stateful wrapper avoid re-rendering the static subtree. For context: `value={{a, b}}` creates a new object each render; use `useMemo`. Split `StateContext` and `DispatchContext` so components that only dispatch don't re-render on state changes. For fine-grained subscriptions use `useSyncExternalStore`-based stores (Zustand, Redux selectors, `use-context-selector`). React Compiler memoizes values but does not split contexts for you.

**Code example or diagram**
```jsx
const StateCtx = createContext(null);
const DispatchCtx = createContext(null);

function CartProvider({ children }) {
  const [state, dispatch] = useReducer(cartReducer, initialCart);
  return (
    <DispatchCtx.Provider value={dispatch}>
      <StateCtx.Provider value={state}>{children}</StateCtx.Provider>
    </DispatchCtx.Provider>
  );
}

// children come from the parent: Layout is not re-rendered when cart state changes
function Root() {
  return (
    <CartProvider>
      <Layout />
    </CartProvider>
  );
}

const AddButton = memo(({ id }) => {
  const dispatch = useContext(DispatchCtx); // never re-renders on state change
  return <button onClick={() => dispatch({ type: "add", id })}>Add</button>;
});
```

**Common pitfalls / follow-up questions**
- *Can `memo` stop a context-triggered render?* No; consumers re-render regardless of memo.
- *Is context a state manager?* It is dependency injection; for frequently changing state use a store with selectors.
- *Why does children not re-render?* Same element reference, so React bails out.

---

<a id="q97"></a>
### Q97. What are web workers and offloading heavy computation?

**Short answer**
Web Workers run JavaScript on a separate thread, so CPU-heavy work (parsing, image processing, large sorting, crypto) doesn't block rendering and hurt INP. They communicate via `postMessage` and cannot touch the DOM. Libraries like Comlink make them feel like async function calls.

**Detailed explanation**
Data is copied via structured clone (or transferred with `Transferable` objects like `ArrayBuffer`, or shared via `SharedArrayBuffer`). Bundlers support `new Worker(new URL("./w.js", import.meta.url), { type: "module" })`. In React, create the worker in an effect or lazily in a ref, terminate on unmount, and store results in state. Use a pool for many tasks. Consider cost: serialization overhead, so use only for tasks taking tens of ms or more. Alternatives: chunk work with `scheduler.yield()`/`requestIdleCallback`, `useTransition`, WASM, or server-side compute.

**Code example or diagram**
```jsx
// worker.js
import { expose } from "comlink";
expose({
  heavySort(arr) {
    return [...arr].sort((a, b) => a - b);
  },
});

// Component
import { wrap } from "comlink";
import { useEffect, useRef, useState } from "react";

export function Sorter({ data }) {
  const apiRef = useRef(null);
  const [result, setResult] = useState([]);

  useEffect(() => {
    const worker = new Worker(new URL("./worker.js", import.meta.url), { type: "module" });
    apiRef.current = wrap(worker);
    return () => worker.terminate();
  }, []);

  useEffect(() => {
    let cancelled = false;
    apiRef.current?.heavySort(data).then((r) => !cancelled && setResult(r));
    return () => { cancelled = true; };
  }, [data]);

  return <p>{result.length} items sorted</p>;
}
```

**Common pitfalls / follow-up questions**
- *Can workers access the DOM or React state?* No; only messages.
- *Why not always use workers?* Messaging/serialization overhead and complexity for small tasks.
- *Transferables?* Moving an `ArrayBuffer` avoids copying; it becomes unusable in the sender.

---

<a id="q98"></a>
### Q98. How do you cut the cost of expensive renders using useTransition?

**Short answer**
`useTransition` marks a state update as non-urgent, so React keeps the UI responsive for urgent input while rendering the heavy update in the background, and can interrupt it if new input arrives. `isPending` lets you show feedback. `useDeferredValue` achieves the same for a value you don't own.

**Detailed explanation**
With concurrent rendering, updates wrapped in `startTransition` get lower priority and are interruptible. Typing stays immediate while a slow list/filter/tab rerenders with the new value later. It does not make rendering faster; it improves perceived responsiveness and INP. Transitions also keep the current UI visible instead of showing Suspense fallbacks again. React 19 allows async functions in transitions (Actions). Limitations: it can't wrap controlled input state itself (that must stay urgent), and it won't help with slow non-React work or a single component whose render is long-running without yielding (combine with memoization/virtualization).

**Code example or diagram**
```jsx
import { useState, useTransition, useDeferredValue, memo } from "react";

function Filterable({ items }) {
  const [query, setQuery] = useState("");
  const [filter, setFilter] = useState("");
  const [isPending, startTransition] = useTransition();

  return (
    <>
      <input
        value={query}
        onChange={(e) => {
          setQuery(e.target.value);                       // urgent
          startTransition(() => setFilter(e.target.value)); // deferred
        }}
      />
      {isPending && <span>Updating…</span>}
      <SlowList items={items} filter={filter} />
    </>
  );
}

// Alternative with deferred value
function Filterable2({ items }) {
  const [q, setQ] = useState("");
  const deferred = useDeferredValue(q);
  return (
    <>
      <input value={q} onChange={(e) => setQ(e.target.value)} />
      <SlowList items={items} filter={deferred} />
    </>
  );
}
const SlowList = memo(function SlowList({ items, filter }) {
  return items.filter((i) => i.includes(filter)).map((i) => <div key={i}>{i}</div>);
});
```

**Common pitfalls / follow-up questions**
- *`useTransition` vs `useDeferredValue`?* Transition wraps the setter you control; deferred value wraps a value from props/elsewhere. Memoize the slow child with deferred values so the stale render is skipped.
- *Can transitions debounce network requests?* No; they only schedule rendering.
- *Why doesn't my transition help?* The slow component renders synchronously for a long time, or the update isn't inside `startTransition` (e.g. after an `await` in older versions).
