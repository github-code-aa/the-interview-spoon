# React Interview Guide — State Management

## Table of Contents
- [Q73. useState vs global store](#q73)
- [Q74. Context API vs Redux](#q74)
- [Q75. Redux flow](#q75)
- [Q76. Redux Toolkit and Immer](#q76)
- [Q77. Thunks vs sagas vs listeners](#q77)
- [Q78. Zustand, Jotai, Recoil, MobX](#q78)
- [Q79. Selectors and Reselect](#q79)
- [Q80. Server state vs client state](#q80)
- [Q81. Normalizing state](#q81)
- [Q82. Persisting state](#q82)
- [Q83. Avoiding unnecessary context re-renders](#q83)
- [Q84. State design for a form wizard](#q84)
- [Q85. State machines (XState)](#q85)

---

<a id="q73"></a>
### Q73. When is useState enough, and when do you need a global store?

**Short answer**
`useState`/`useReducer` is enough when state is local to a component or a small subtree (lifting state up covers siblings). Reach for a global store when many distant components share and update the same client state, prop drilling or context re-renders become painful, or you need devtools, middleware, or persistence. Server data belongs in a query library, not a global store.

**Detailed explanation**
- **Ladder of escalation**: local `useState` → `useReducer` for complex transitions → lift state up → composition (pass components as children) → Context for low-frequency values → external store (Redux Toolkit, Zustand, Jotai) for high-frequency or widely shared state.
- **Signals you need a store**: the same state read in many unrelated places; cross-cutting concerns (cart, auth, theme, notifications); time-travel debugging; complex update logic shared across features.
- **Keep it colocated**: state should live as close as possible to where it's used. Form input values, open/closed toggles and hover state almost never belong in a global store.
- **URL state** (filters, tabs) is often better in the URL than in a store.

**Code example**
```jsx
import { useReducer } from "react";

// Local complex state: useReducer is enough — no global store needed
function cartReducer(state, action) {
  switch (action.type) {
    case "add": {
      const existing = state.find((i) => i.id === action.item.id);
      return existing
        ? state.map((i) => (i.id === existing.id ? { ...i, qty: i.qty + 1 } : i))
        : [...state, { ...action.item, qty: 1 }];
    }
    case "remove":
      return state.filter((i) => i.id !== action.id);
    default:
      return state;
  }
}

export function MiniCart({ products }) {
  const [cart, dispatch] = useReducer(cartReducer, []);
  return (
    <>
      {products.map((p) => (
        <button key={p.id} onClick={() => dispatch({ type: "add", item: p })}>Add {p.name}</button>
      ))}
      <ul>
        {cart.map((i) => (
          <li key={i.id}>
            {i.name} × {i.qty} <button onClick={() => dispatch({ type: "remove", id: i.id })}>x</button>
          </li>
        ))}
      </ul>
    </>
  );
}
```

**Common pitfalls / follow-up questions**
- *Is prop drilling 2–3 levels bad?* No — it's explicit and fine; don't add a store prematurely.
- *Can lifting state up be an issue?* Yes: the high parent re-renders everything; fix with composition or splitting state.
- *Global store for API data?* Avoid; use TanStack Query/RTK Query.

---

<a id="q74"></a>
### Q74. Context API vs Redux: tradeoffs.

**Short answer**
Context is a *dependency-injection* mechanism, not a state manager: any change to its value re-renders all consumers, and it has no middleware, devtools or selectors. Redux is a predictable, centralized store with selector-based subscriptions, middleware and excellent tooling, at the cost of boilerplate and concepts. Use Context for low-frequency values (theme, locale, auth user) and Redux (Toolkit) for complex, frequently-updated shared state.

**Detailed explanation**
- **Context**: built-in, zero dependencies, great for static or rarely-changing values. All consumers re-render when the value identity changes, even if they use only part of it. Must memoize the value and split contexts.
- **Redux**: `useSelector` only re-renders when the selected slice changes; Redux DevTools offer time travel and action logs; middleware for async/logging; strong conventions for large teams; testable pure reducers.
- **Combination**: Redux uses Context internally just to pass the store; the subscription happens outside React.
- **Alternatives**: Zustand/Jotai give selector-based updates with less ceremony.

| | Context | Redux Toolkit |
|---|---|---|
| Setup | Minimal | Moderate |
| Re-render control | Poor (no selectors) | Fine-grained |
| Devtools/middleware | None | Yes |
| Best for | Theme, locale, DI | Complex shared app state |

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
  return <ThemeContext.Provider value={value}>{children}</ThemeContext.Provider>;
}

export function useTheme() {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error("useTheme must be used within ThemeProvider");
  return ctx;
}

// Redux equivalent with fine-grained subscription:
// const theme = useSelector((s) => s.ui.theme);  // re-renders only if theme changes
```

**Common pitfalls / follow-up questions**
- *Is Context "state management"?* No; it transports a value. State still comes from `useState`/`useReducer`.
- *Passing a new object literal as `value`?* Re-renders all consumers every provider render; memoize it.
- *Can Context replace Redux?* For small apps yes; for frequent updates and debuggability, no.

---

<a id="q75"></a>
### Q75. Explain the Redux flow (store, actions, reducers, middleware).

**Short answer**
Redux has one-way data flow: the UI dispatches an *action* (plain object describing what happened); middleware can intercept it; *reducers* (pure functions) compute the next state from the current state and the action; the *store* saves it and notifies subscribers, so the UI re-renders. State is never mutated directly.

**Detailed explanation**
- **Store**: holds the single state tree; exposes `getState`, `dispatch`, `subscribe`.
- **Action**: `{ type: "cart/added", payload }`; action creators build them.
- **Reducer**: `(state, action) => newState`; must be pure — no side effects, no mutation, no randomness/Date.
- **Middleware**: sits between dispatch and reducer (`store => next => action => ...`); used for async (thunk), logging, analytics, API calls.
- **Selectors**: read derived data from state.
- **Three principles**: single source of truth, state is read-only (change via actions), changes via pure reducers.
- **Flow**: UI event → `dispatch(action)` → middleware chain → root reducer → new state → subscribers notified → `useSelector` re-evaluates → re-render if changed.

**Code example**
```jsx
import { createStore, applyMiddleware } from "redux";
import { Provider, useDispatch, useSelector } from "react-redux";

const logger = (store) => (next) => (action) => {
  console.log("dispatching", action);
  const result = next(action);
  console.log("next state", store.getState());
  return result;
};

function counterReducer(state = { value: 0 }, action) {
  switch (action.type) {
    case "counter/incremented":
      return { ...state, value: state.value + action.payload };
    default:
      return state;
  }
}

const store = createStore(counterReducer, applyMiddleware(logger));

function Counter() {
  const value = useSelector((s) => s.value);
  const dispatch = useDispatch();
  return <button onClick={() => dispatch({ type: "counter/incremented", payload: 1 })}>Count: {value}</button>;
}

export default function App() {
  return (
    <Provider store={store}>
      <Counter />
    </Provider>
  );
}
```
> Note: `createStore` is deprecated in favor of Redux Toolkit's `configureStore` (see Q76); shown here to illustrate the core concepts.

**Common pitfalls / follow-up questions**
- *Why must reducers be pure?* For predictability, time-travel, memoization and change detection by reference.
- *Why mutation breaks things?* `useSelector` compares by reference; mutated objects look unchanged so UI doesn't update.
- *Where do side effects go?* Middleware, thunks, sagas, listeners — not reducers.

---

<a id="q76"></a>
### Q76. What is Redux Toolkit? How does it use Immer?

**Short answer**
Redux Toolkit (RTK) is the official, opinionated way to write Redux: `configureStore`, `createSlice`, `createAsyncThunk`, and RTK Query remove boilerplate and encode best practices. It uses Immer so you can write "mutating" code in reducers that is actually translated into safe immutable updates.

**Detailed explanation**
- **`configureStore`**: combines reducers, adds thunk middleware, Redux DevTools, and dev-time checks (immutability and serializability).
- **`createSlice`**: generates action creators and action types from reducer functions; `name/reducerName` types.
- **Immer**: reducers receive a *draft* Proxy. Mutations are recorded and applied to produce a new immutable state with structural sharing (unchanged parts keep references). You can either mutate the draft **or** return a new value — never both.
- **`createAsyncThunk`**: generates `pending/fulfilled/rejected` actions, handled with `extraReducers`.
- **`createEntityAdapter`**: normalized state helpers (Q81). **`createListenerMiddleware`** for side effects (Q77).
- Immer also available standalone (`produce`) and via `useImmerReducer`.

**Code example**
```jsx
import { configureStore, createSlice, createAsyncThunk } from "@reduxjs/toolkit";
import { Provider, useDispatch, useSelector } from "react-redux";

export const fetchTodos = createAsyncThunk("todos/fetch", async (_, { signal }) => {
  const res = await fetch("/api/todos", { signal });
  if (!res.ok) throw new Error("Failed");
  return res.json();
});

const todosSlice = createSlice({
  name: "todos",
  initialState: { items: [], status: "idle" },
  reducers: {
    added(state, action) {
      state.items.push(action.payload); // safe: Immer draft
    },
    toggled(state, action) {
      const todo = state.items.find((t) => t.id === action.payload);
      if (todo) todo.done = !todo.done;
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchTodos.pending, (s) => { s.status = "loading"; })
      .addCase(fetchTodos.fulfilled, (s, a) => { s.status = "ready"; s.items = a.payload; })
      .addCase(fetchTodos.rejected, (s) => { s.status = "failed"; });
  },
});

export const { added, toggled } = todosSlice.actions;
export const store = configureStore({ reducer: { todos: todosSlice.reducer } });

function Todos() {
  const dispatch = useDispatch();
  const items = useSelector((s) => s.todos.items);
  return (
    <>
      <button onClick={() => dispatch(fetchTodos())}>Load</button>
      {items.map((t) => (
        <label key={t.id}>
          <input type="checkbox" checked={t.done} onChange={() => dispatch(toggled(t.id))} />
          {t.title}
        </label>
      ))}
    </>
  );
}

export default function App() {
  return <Provider store={store}><Todos /></Provider>;
}
```

**Common pitfalls / follow-up questions**
- *"Mutating" and returning a value in the same reducer?* Immer throws; do one or the other.
- *Non-serializable values (Dates, class instances, functions) in state?* RTK warns; store ISO strings/numbers.
- *Does Immer hurt performance?* Slight overhead; usually negligible, and structural sharing keeps selectors efficient. Avoid mutating huge arrays element-by-element in hot paths.

---

<a id="q77"></a>
### Q77. What are thunks vs sagas vs listeners?

**Short answer**
All three handle side effects in Redux. **Thunks** are functions dispatched instead of actions that can run async logic with `dispatch/getState` — simple and the default. **Sagas** (redux-saga) use generator functions and an effects API for complex, long-running, cancellable workflows. **Listener middleware** reacts to dispatched actions/state changes with callbacks, covering most saga use cases with less complexity.

**Detailed explanation**
- **Thunk**: `(dispatch, getState, extra) => Promise`. Great for data fetching, conditional dispatches. `createAsyncThunk` standardizes lifecycle actions and supports cancellation via `signal`. Limited for orchestration (debounce, race, cancel).
- **Saga**: `takeLatest`, `debounce`, `race`, `fork`, `call`, `put` — declarative effects make tests easy (step through generator, no mocking) and enable complex flows (retries, background tasks, websocket channels). Steeper learning curve, extra dependency.
- **Listener middleware** (`createListenerMiddleware`): `startListening({ actionCreator|matcher|predicate, effect })` with `listenerApi` offering `delay`, `condition`, `cancelActiveListeners`, `fork`. Handles debounce, "when X then Y" logic.
- **Guidance**: thunks for most; listeners for reactive side effects; sagas only when you truly need advanced orchestration or already have them. Observables (redux-observable) are another option.

**Code example**
```jsx
import {
  configureStore,
  createListenerMiddleware,
  createSlice,
  createAsyncThunk,
  isAnyOf,
} from "@reduxjs/toolkit";

// Thunk
export const saveSettings = createAsyncThunk("settings/save", async (settings) => {
  const res = await fetch("/api/settings", {
    method: "PUT",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(settings),
  });
  return res.json();
});

const settingsSlice = createSlice({
  name: "settings",
  initialState: { theme: "light", query: "" },
  reducers: {
    themeChanged: (s, a) => { s.theme = a.payload; },
    queryChanged: (s, a) => { s.query = a.payload; },
  },
});
export const { themeChanged, queryChanged } = settingsSlice.actions;

// Listener: debounced search side effect + persisting theme
const listeners = createListenerMiddleware();
listeners.startListening({
  actionCreator: queryChanged,
  effect: async (action, api) => {
    api.cancelActiveListeners(); // takeLatest behaviour
    await api.delay(300);        // debounce
    const res = await fetch(`/api/search?q=${encodeURIComponent(action.payload)}`, { signal: api.signal });
    console.log(await res.json());
  },
});
listeners.startListening({
  matcher: isAnyOf(themeChanged),
  effect: (_a, api) => localStorage.setItem("theme", api.getState().settings.theme),
});

export const store = configureStore({
  reducer: { settings: settingsSlice.reducer },
  middleware: (gdm) => gdm().prepend(listeners.middleware),
});
```

**Common pitfalls / follow-up questions**
- *Thunk vs saga for cancellation?* Sagas have first-class cancel; thunks use AbortSignal; listeners use `cancelActiveListeners`.
- *Is saga dead?* Not dead, but RTK recommends listeners/thunks for new code.
- *Where should middleware be prepended?* Listener middleware should be prepended so it sees actions before serializability checks, per RTK docs.

---

<a id="q78"></a>
### Q78. What are Zustand, Jotai, Recoil and MobX? Compare them.

**Short answer**
They are lighter alternatives to Redux. **Zustand**: a small store with hooks and selectors. **Jotai**: atomic, bottom-up state with derived atoms. **Recoil**: Meta's atoms/selectors model, now archived/unmaintained so avoid for new projects. **MobX**: observable, mutable state with automatic fine-grained reactivity.

**Detailed explanation**
- **Zustand**: `create((set, get) => ({...}))`; no provider needed; components subscribe via selectors (`useStore(s => s.count)`); middleware for persist, devtools, immer. Flux-like but minimal.
- **Jotai**: state is split into atoms; derived atoms compute from others; components re-render only for atoms they read; great for many independent pieces; async atoms integrate with Suspense.
- **Recoil**: similar atom/selector model with async selectors; development stopped (repo archived), so prefer Jotai for similar ergonomics.
- **MobX**: `makeAutoObservable` classes/objects; `observer` components track which observables they read and re-render precisely; you mutate directly; powerful for complex domain models; "magic" can be harder to debug.
- **Redux Toolkit** by contrast: explicit, predictable, best tooling, more ceremony.

| Library | Model | Provider | Re-render control | Notes |
|---|---|---|---|---|
| Zustand | Single/multiple stores | No | Selectors | Simple, popular |
| Jotai | Atoms | Optional | Per-atom | Fine-grained, small |
| Recoil | Atoms/selectors | Yes | Per-atom | Archived |
| MobX | Observables | Optional | Automatic tracking | Mutable, OOP-friendly |

**Code example**
```jsx
import { create } from "zustand";
import { atom, useAtom, useAtomValue } from "jotai";

// Zustand
const useCartStore = create((set) => ({
  items: [],
  add: (item) => set((s) => ({ items: [...s.items, item] })),
  clear: () => set({ items: [] }),
}));

function CartCount() {
  const count = useCartStore((s) => s.items.length); // re-renders only when length changes
  return <span>{count}</span>;
}

// Jotai
const priceAtom = atom(10);
const qtyAtom = atom(2);
const totalAtom = atom((get) => get(priceAtom) * get(qtyAtom)); // derived

function Total() {
  const total = useAtomValue(totalAtom);
  const [qty, setQty] = useAtom(qtyAtom);
  return <button onClick={() => setQty(qty + 1)}>Total: {total}</button>;
}
```

**Common pitfalls / follow-up questions**
- *Zustand selecting an object?* `useStore(s => ({a: s.a, b: s.b}))` creates a new object every time → re-renders; use `useShallow` or separate selectors.
- *Why avoid Recoil now?* No maintenance; Jotai offers a near-drop-in model.
- *MobX gotcha?* Forgetting `observer` or destructuring observables too early loses reactivity.

---

<a id="q79"></a>
### Q79. What are selectors and memoization (Reselect)?

**Short answer**
Selectors are functions that read and derive data from state (`state => visibleTodos`). Memoized selectors (Reselect's `createSelector`) cache the result and recompute only when their input selectors' results change by reference, avoiding expensive recalculation and keeping derived references stable to prevent re-renders.

**Detailed explanation**
- **Why**: store minimal state, compute derived data on read; centralize shape knowledge so state structure can change in one place.
- **`createSelector(inputs, resultFn)`**: runs input selectors; if outputs are `===` to the previous ones, returns the cached result. Recent Reselect (v5) uses `weakMapMemoize` by default and warns in dev about unstable inputs or identity-function results.
- **Reference stability**: `useSelector(s => s.items.filter(...))` returns a new array each time → re-render on every store update. Memoization fixes that.
- **Parameterized selectors**: create per-instance selectors (`useMemo(makeSelector)`) or use `weakMapMemoize` with args.
- **Not just Redux**: Zustand with `useShallow`, Jotai derived atoms, `useMemo` for local.
- Don't memoize trivial selectors (simple property reads).

**Code example**
```jsx
import { createSelector } from "@reduxjs/toolkit"; // re-exports Reselect
import { useSelector } from "react-redux";

const selectTodos = (s) => s.todos.items;
const selectFilter = (s) => s.todos.filter;

export const selectVisibleTodos = createSelector(
  [selectTodos, selectFilter],
  (todos, filter) => {
    // only recomputed if todos or filter references change
    return todos.filter((t) => (filter === "done" ? t.done : filter === "open" ? !t.done : true));
  }
);

export const selectOpenCount = createSelector([selectTodos], (todos) => todos.filter((t) => !t.done).length);

export function TodoList() {
  const todos = useSelector(selectVisibleTodos);
  const open = useSelector(selectOpenCount);
  return (
    <>
      <p>{open} open</p>
      <ul>{todos.map((t) => <li key={t.id}>{t.title}</li>)}</ul>
    </>
  );
}
```

**Common pitfalls / follow-up questions**
- *Selector returns new reference each call?* Causes needless re-renders and React-Redux dev warnings; memoize.
- *Cache size problem with args?* Older Reselect has cache size 1, so a shared selector with different args thrashes; use a selector factory or Reselect 5's weakMapMemoize.
- *Mutating state breaks memoization?* Yes, same reference means no recompute and stale results.

---

<a id="q80"></a>
### Q80. Server state vs client state: how do you separate them?

**Short answer**
Server state is remote data you don't own (async, shared, can go stale) — manage it with TanStack Query/SWR/RTK Query, which provide caching and sync. Client state is synchronous UI/app state you own (modals, wizard step, theme) — manage it with `useState`, Context or Zustand/Redux. Don't copy server data into a client store.

**Detailed explanation**
- **Server state traits**: persisted remotely, async, shared ownership, can change without you knowing → needs cache, invalidation, dedupe, retries.
- **Client state traits**: ephemeral or locally persisted, synchronous, fully controlled.
- **Anti-pattern**: fetching in an effect then `dispatch(setUsers(data))` — you've reinvented a cache poorly and now must handle staleness manually.
- **Boundary cases**: form drafts built from server data — keep the server value as the initial value, the draft as local state, and submit via mutation. Derive rather than sync (`select` option in queries).
- **URL state** is a third category: filters/pagination.
- Benefits: far less global state, simpler reducers, fewer bugs.

**Code example**
```jsx
import { useQuery } from "@tanstack/react-query";
import { create } from "zustand";

// Client state: UI only
const useUiStore = create((set) => ({
  selectedUserId: null,
  select: (id) => set({ selectedUserId: id }),
}));

// Server state: via query library, keyed by client state
function UserDetail() {
  const id = useUiStore((s) => s.selectedUserId);
  const { data: user, isPending } = useQuery({
    queryKey: ["user", id],
    queryFn: () => fetch(`/api/users/${id}`).then((r) => r.json()),
    enabled: id != null,
  });

  if (id == null) return <p>Select a user</p>;
  if (isPending) return <p>Loading…</p>;
  return <h2>{user.name}</h2>;
}

function UserList({ users }) {
  const select = useUiStore((s) => s.select);
  return users.map((u) => <button key={u.id} onClick={() => select(u.id)}>{u.name}</button>);
}
```

**Common pitfalls / follow-up questions**
- *Where do I store the selected item — id or object?* Store the id; derive the object from cache so it never goes stale.
- *Syncing query data into `useState`?* Only for editable drafts; otherwise derive.
- *Optimistic UI belongs to which?* It temporarily writes client intent into the server-state cache with rollback.

---

<a id="q81"></a>
### Q81. How do you normalize state?

**Short answer**
Normalizing means storing entities in a lookup table by id (`{ ids: [], entities: { id: item } }`) and referencing them by id instead of nesting or duplicating them. It makes updates O(1), avoids inconsistent copies, and keeps re-renders narrow. Redux Toolkit's `createEntityAdapter` automates this.

**Detailed explanation**
- **Problem with nested/duplicated data**: updating a user that appears in many posts requires touching every copy; deep spreads are error-prone.
- **Structure**: one table per entity type, relationships as id arrays/refs, ordering via `ids`.
- **`createEntityAdapter`**: provides `addOne`, `upsertMany`, `updateOne`, `removeOne`, plus `getSelectors` (`selectAll`, `selectById`) and optional `sortComparer`.
- **`normalizr`**: converts nested API responses into normalized form (less common now).
- **Server state**: TanStack Query caches per query key and doesn't normalize by default; you can update related caches manually or use `select`. Apollo/Relay normalize GraphQL by `__typename:id`.
- **Tradeoff**: more upfront shape work; denormalize in selectors for the UI.

**Code example**
```jsx
import { configureStore, createEntityAdapter, createSlice } from "@reduxjs/toolkit";
import { useSelector } from "react-redux";

const usersAdapter = createEntityAdapter({
  sortComparer: (a, b) => a.name.localeCompare(b.name),
});

const usersSlice = createSlice({
  name: "users",
  initialState: usersAdapter.getInitialState(),
  reducers: {
    usersReceived: usersAdapter.setAll,
    userUpdated: usersAdapter.updateOne, // { id, changes }
    userRemoved: usersAdapter.removeOne,
  },
});

export const { usersReceived, userUpdated } = usersSlice.actions;
export const { selectAll: selectAllUsers, selectById: selectUserById } =
  usersAdapter.getSelectors((state) => state.users);

export const store = configureStore({ reducer: { users: usersSlice.reducer } });

// state shape: { users: { ids: ["2","1"], entities: { 1: {...}, 2: {...} } } }

function UserName({ id }) {
  const user = useSelector((s) => selectUserById(s, id)); // only re-renders if this user changes
  return <span>{user?.name}</span>;
}
```

**Common pitfalls / follow-up questions**
- *When not to normalize?* Small, read-only, non-shared data — normalization is needless complexity.
- *How to keep relations consistent on delete?* Remove references (or use cascading logic) in related slices via `extraReducers`.
- *Order of items?* The `ids` array (or sortComparer) holds order; the object map doesn't.

---

<a id="q82"></a>
### Q82. How do you persist state?

**Short answer**
Serialize the relevant slice of state to a storage (localStorage, sessionStorage, IndexedDB, cookies, or the URL/server) and rehydrate it on startup. Libraries help: Zustand's `persist` middleware, `redux-persist`, TanStack Query's persister. Persist only what's needed, version it, and never store secrets in localStorage.

**Detailed explanation**
- **Where**: localStorage (small, sync, same-origin), IndexedDB (large/async), cookies (sent to server, useful for SSR theme/auth), URL (shareable), server (cross-device).
- **What**: user preferences, cart, drafts, auth session hints. Not transient UI or sensitive tokens (XSS risk; prefer httpOnly cookies).
- **Rehydration**: initial load may render before hydration completes → flash/mismatch. With SSR, reading localStorage during render causes hydration mismatches; load in an effect or use cookies.
- **Versioning/migrations**: schema changes break old persisted data; include `version` and `migrate`.
- **Cross-tab sync**: listen to the `storage` event.
- **Custom hook**: lazy initial state `useState(() => read())` plus an effect that writes.
- **Query cache persistence**: `persistQueryClient` for offline support.

**Code example**
```jsx
import { create } from "zustand";
import { persist, createJSONStorage } from "zustand/middleware";
import { useEffect, useState } from "react";

export const useSettings = create(
  persist(
    (set) => ({
      theme: "light",
      sidebarOpen: true, // ephemeral; excluded below
      setTheme: (theme) => set({ theme }),
    }),
    {
      name: "settings-v1",
      version: 2,
      storage: createJSONStorage(() => localStorage),
      partialize: (s) => ({ theme: s.theme }),
      migrate: (persisted, version) => (version < 2 ? { theme: persisted.theme ?? "light" } : persisted),
    }
  )
);

// Generic hook with cross-tab sync
export function useLocalStorage(key, initial) {
  const [value, setValue] = useState(() => {
    try {
      const raw = localStorage.getItem(key);
      return raw ? JSON.parse(raw) : initial;
    } catch {
      return initial;
    }
  });

  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(value));
  }, [key, value]);

  useEffect(() => {
    const onStorage = (e) => e.key === key && e.newValue && setValue(JSON.parse(e.newValue));
    window.addEventListener("storage", onStorage);
    return () => window.removeEventListener("storage", onStorage);
  }, [key]);

  return [value, setValue];
}
```

**Common pitfalls / follow-up questions**
- *Hydration mismatch in SSR?* Server can't read localStorage; render defaults then update in an effect, or use cookies.
- *Storage quota/serialization errors?* Wrap in try/catch (private mode, quota); only JSON-safe values.
- *Is storing a JWT in localStorage okay?* Risky (XSS); prefer httpOnly secure cookies.

---

<a id="q83"></a>
### Q83. How do you avoid unnecessary context re-renders?

**Short answer**
Every consumer re-renders whenever the context value changes by reference, so: memoize the value, split contexts (state vs dispatch, or by update frequency), keep state low in the tree, and pass `children` through providers. For selector-like granularity use an external store (Zustand, `use-context-selector`) instead.

**Detailed explanation**
- **Memoize provider value** with `useMemo`/`useCallback`; otherwise a new object each render triggers all consumers.
- **Split state and dispatch contexts**: `dispatch` from `useReducer` is stable, so components that only dispatch never re-render on state changes.
- **Split by domain/frequency**: `ThemeContext` vs `UserContext` vs fast-changing `MousePositionContext`.
- **Children pattern**: provider receiving `{children}` as props doesn't re-render children when the provider's own state changes (same element references).
- **Memoize consumers** with `React.memo` — note context changes bypass `memo` for components that consume it.
- **Selector subscription**: `useSyncExternalStore`-based stores or `use-context-selector` allow partial subscription. React Compiler helps memoization but doesn't change context semantics.
- Don't put rapidly-changing values (inputs, scroll) in context.

**Code example**
```jsx
import { createContext, useContext, useMemo, useReducer } from "react";

const CountStateContext = createContext(0);
const CountDispatchContext = createContext(null);

function reducer(state, action) {
  return action === "inc" ? state + 1 : state;
}

export function CountProvider({ children }) {
  const [count, dispatch] = useReducer(reducer, 0);
  return (
    <CountStateContext.Provider value={count}>
      <CountDispatchContext.Provider value={dispatch}>{children}</CountDispatchContext.Provider>
    </CountStateContext.Provider>
  );
}

export const useCount = () => useContext(CountStateContext);
export const useCountDispatch = () => useContext(CountDispatchContext);

// Does NOT re-render when count changes
export function IncrementButton() {
  const dispatch = useCountDispatch();
  return <button onClick={() => dispatch("inc")}>+1</button>;
}

export function Display() {
  return <p>{useCount()}</p>;
}

// When you have several values in one context, memoize
export function AuthProvider({ children, user, login, logout }) {
  const value = useMemo(() => ({ user, login, logout }), [user, login, logout]);
  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}
const AuthContext = createContext(null);
```

**Common pitfalls / follow-up questions**
- *Does `React.memo` stop context-triggered renders?* No; a component using `useContext` re-renders when the value changes regardless.
- *Why is my provider causing a full-tree re-render?* The provider's parent re-rendered and passed fresh `children`, or the value isn't memoized.
- *Best alternative for hot state?* Zustand/Redux with selectors or Jotai atoms.

---

<a id="q84"></a>
### Q84. How do you design state for a complex form wizard?

**Short answer**
Model the wizard as a single state object holding `step`, per-step data, and validity/touched info, managed by a reducer or store so steps can be navigated back/forth without losing data. Validate per step with schemas, validate everything again at final submit, and persist drafts (URL step, sessionStorage/server). Keep each step's form local (RHF) and commit to the wizard state on "Next".

**Detailed explanation**
- **Shape**: `{ step, data: { account, profile, payment }, completed: Set/array }`; one slice per step keeps schemas modular.
- **Flow control**: navigation rules (can't skip unvalidated steps), conditional steps (branching), derived `canGoNext`. For many branching rules a state machine (Q85) is cleaner.
- **Validation**: per-step Zod schema on Next; full-schema parse on submit; map server errors back to the correct step.
- **Persistence**: draft in sessionStorage or server for resume; sync the current step with the URL (`/signup/profile`) so refresh and back button work.
- **Performance/UX**: each step form uses RHF with `defaultValues` from wizard state; unmount-safe because data is committed on Next/Back; focus management and progress indicator for accessibility.
- **Submission**: final mutation with idempotency; disable buttons while pending; clear draft on success.
- Avoid one giant form that renders all steps (slow, hard to validate), and avoid losing data when steps unmount (`shouldUnregister: false` semantics).

**Code example**
```jsx
import { createContext, useContext, useReducer } from "react";
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";

const schemas = {
  account: z.object({ email: z.string().email() }),
  profile: z.object({ name: z.string().min(2) }),
};
const steps = ["account", "profile", "review"];

function reducer(state, action) {
  switch (action.type) {
    case "commit": // save step data and advance
      return {
        ...state,
        data: { ...state.data, [action.step]: action.values },
        step: Math.min(state.step + 1, steps.length - 1),
      };
    case "back":
      return { ...state, step: Math.max(state.step - 1, 0) };
    default:
      return state;
  }
}

const WizardContext = createContext(null);

function StepForm({ name, children }) {
  const { state, dispatch } = useContext(WizardContext);
  const { register, handleSubmit, formState: { errors } } = useForm({
    resolver: zodResolver(schemas[name]),
    defaultValues: state.data[name],
  });
  return (
    <form onSubmit={handleSubmit((values) => dispatch({ type: "commit", step: name, values }))}>
      {children({ register, errors })}
      <button type="button" onClick={() => dispatch({ type: "back" })} disabled={state.step === 0}>Back</button>
      <button>Next</button>
    </form>
  );
}

export function Wizard({ onSubmit }) {
  const [state, dispatch] = useReducer(reducer, { step: 0, data: {} });
  const current = steps[state.step];

  return (
    <WizardContext.Provider value={{ state, dispatch }}>
      <p>Step {state.step + 1} of {steps.length}</p>
      {current === "account" && (
        <StepForm name="account">
          {({ register, errors }) => (<><input placeholder="Email" {...register("email")} /><span>{errors.email?.message}</span></>)}
        </StepForm>
      )}
      {current === "profile" && (
        <StepForm name="profile">
          {({ register, errors }) => (<><input placeholder="Name" {...register("name")} /><span>{errors.name?.message}</span></>)}
        </StepForm>
      )}
      {current === "review" && (
        <>
          <pre>{JSON.stringify(state.data, null, 2)}</pre>
          <button onClick={() => onSubmit(state.data)}>Submit</button>
        </>
      )}
    </WizardContext.Provider>
  );
}
```

**Common pitfalls / follow-up questions**
- *Data lost when going back?* Steps unmount; commit values to wizard state on Back too (or use `getValues` before navigating) and pass as `defaultValues`.
- *Users jump via URL to later step?* Guard: redirect to the first incomplete step.
- *Branching steps?* Compute the step list from data (`steps = getSteps(data)`), or use a state machine.

---

<a id="q85"></a>
### Q85. What are state machines (XState) and when should you use them?

**Short answer**
A state machine models behavior as a finite set of states, events, and transitions, so only valid transitions can happen (impossible states become unrepresentable). XState is the JS library for statecharts (nested/parallel states, guards, actions, invoked actors). Use it for complex workflows — wizards, auth flows, media players, multi-step async processes — not for simple toggles.

**Detailed explanation**
- **Core concepts**: states (`idle`, `loading`, `success`, `failure`), events (`FETCH`, `RETRY`), transitions, guards (conditions), actions (side effects), context (extended data), and invoked services/actors (promises, callbacks).
- **Benefits**: eliminates boolean soup (`isLoading && !isError && ...`), explicit and visualizable (Stately editor), testable (model-based testing), predictable async handling with cancellation on state exit.
- **XState v5** API: `setup({...}).createMachine({...})`, `createActor`, and `useMachine`/`useActor`/`useSelector` from `@xstate/react`. Actor model supports spawning child machines.
- **Lighter alternatives**: `useReducer` with a `status` discriminated union, or `@xstate/store` for simple stores.
- **When not to use**: trivial UI state, CRUD with a query library handling status, teams unfamiliar and the logic is small.
- **Fits with**: form wizards, payments, drag-and-drop, retry/backoff logic, OAuth flows.

**Code example**
```jsx
import { setup, assign, fromPromise } from "xstate";
import { useMachine } from "@xstate/react";

const fetchUser = fromPromise(async ({ input }) => {
  const res = await fetch(`/api/users/${input.id}`);
  if (!res.ok) throw new Error("Failed");
  return res.json();
});

const userMachine = setup({
  actors: { fetchUser },
  guards: { canRetry: ({ context }) => context.retries < 3 },
}).createMachine({
  id: "user",
  initial: "idle",
  context: { id: 1, user: null, retries: 0 },
  states: {
    idle: { on: { FETCH: "loading" } },
    loading: {
      invoke: {
        src: "fetchUser",
        input: ({ context }) => ({ id: context.id }),
        onDone: { target: "success", actions: assign({ user: ({ event }) => event.output }) },
        onError: { target: "failure", actions: assign({ retries: ({ context }) => context.retries + 1 }) },
      },
    },
    success: { on: { FETCH: "loading" } },
    failure: { on: { RETRY: { guard: "canRetry", target: "loading" } } },
  },
});

export function UserCard() {
  const [snapshot, send] = useMachine(userMachine);

  return (
    <div>
      {snapshot.matches("idle") && <button onClick={() => send({ type: "FETCH" })}>Load</button>}
      {snapshot.matches("loading") && <p>Loading…</p>}
      {snapshot.matches("success") && <h2>{snapshot.context.user.name}</h2>}
      {snapshot.matches("failure") && (
        <button onClick={() => send({ type: "RETRY" })} disabled={!snapshot.can({ type: "RETRY" })}>
          Retry
        </button>
      )}
    </div>
  );
}
```

**Common pitfalls / follow-up questions**
- *State machine vs reducer?* A reducer allows any action in any state unless you code checks; a machine declares allowed transitions per state, making invalid transitions impossible.
- *Machine vs TanStack Query for fetching?* Query handles server-state caching; machines model workflow/UI logic. They complement each other.
- *Overengineering risk?* Yes for simple cases; adopt when booleans multiply or flows have many conditions.
