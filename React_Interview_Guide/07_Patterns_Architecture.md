# React Interview Guide — 07 Patterns & Architecture

## Table of Contents
- [Q99. HOCs and their problems](#q99)
- [Q100. Render props](#q100)
- [Q101. HOCs vs render props vs hooks](#q101)
- [Q102. Compound components](#q102)
- [Q103. Controlled vs uncontrolled](#q103)
- [Q104. Container/presentational, smart/dumb](#q104)
- [Q105. Provider and observer patterns](#q105)
- [Q106. State reducer pattern](#q106)
- [Q107. Headless components](#q107)
- [Q108. Building a design system](#q108)
- [Q109. Versioning, documenting, publishing](#q109)
- [Q110. Structuring a large project](#q110)
- [Q111. Feature-Sliced Design](#q111)
- [Q112. Monorepos](#q112)
- [Q113. Micro-frontends and Module Federation](#q113)
- [Q114. Clean/hexagonal architecture](#q114)
- [Q115. Separating UI, logic and data access](#q115)
- [Q116. SOLID in React](#q116)
- [Q117. Barrel files](#q117)
- [Q118. Dependency injection and service layers](#q118)
- [Q119. API client layer](#q119)
- [Q120. Anti-corruption layer](#q120)

---

<a id="q99"></a>
### Q99. What are HOCs? What problems do they have?

**Short answer**
A Higher-Order Component is a function that takes a component and returns a new component with added behavior (`withAuth(Page)`). They enable cross-cutting reuse but suffer from wrapper hell, prop collisions, lost static methods/refs, and opaque data flow. Hooks replaced most use cases.

**Detailed explanation**
HOCs are the decorator pattern for components. Issues: (1) **wrapper hell** deep trees in DevTools; (2) **prop name collisions** when the HOC injects props; (3) **implicit dependencies**, since the wrapped component's inputs aren't visible at the call site; (4) **refs** don't pass through without `forwardRef`; (5) **statics** must be hoisted (`hoist-non-react-statics`); (6) **never create HOCs inside render**, which remounts the subtree each render; (7) TypeScript typing is awkward. Still useful for things like `connect`, `withRouter` legacy, or wrapping with error boundaries/analytics.

**Code example or diagram**
```jsx
import { forwardRef } from "react";

function withAuth(Component) {
  const Wrapped = forwardRef(function Wrapped(props, ref) {
    const { user } = useAuth();
    if (!user) return <Navigate to="/login" replace />;
    return <Component ref={ref} {...props} user={user} />;
  });
  Wrapped.displayName = `withAuth(${Component.displayName ?? Component.name})`;
  return Wrapped;
}

// define outside render
export default withAuth(Dashboard);
```

**Common pitfalls / follow-up questions**
- *Why not call `withX(C)` inside render?* New component type each time causes unmount/remount and state loss.
- *Order of composition?* `withA(withB(C))` — B is innermost; order affects props and behavior.
- *Modern alternative?* A custom hook or a wrapper component using `children`.

---

<a id="q100"></a>
### Q100. What are render props?

**Short answer**
A render prop is a prop whose value is a function that a component calls to decide what to render, letting the component share its state/logic while the consumer controls the UI. `children` as a function is the common form.

**Detailed explanation**
The component owns behavior (mouse position, fetch state, virtualization window), and calls `render(data)`. It gives explicit data flow and avoids HOC name collisions, since the consumer names the arguments. Downsides: nesting ("callback hell") when combining several, and extra closures. Still prevalent in libraries where inversion of rendering is needed: Downshift, React Router `<Route render>` (legacy), Formik `<Field>`, react-virtualized, and TanStack `<Form.Field>`. Memoization caveat: a new inline function each render defeats `memo` on the provider.

**Code example or diagram**
```jsx
function MouseTracker({ children }) {
  const [pos, setPos] = useState({ x: 0, y: 0 });
  useEffect(() => {
    const onMove = (e) => setPos({ x: e.clientX, y: e.clientY });
    window.addEventListener("mousemove", onMove);
    return () => window.removeEventListener("mousemove", onMove);
  }, []);
  return children(pos);
}

<MouseTracker>
  {({ x, y }) => <p>Cursor at {x}, {y}</p>}
</MouseTracker>
```

**Common pitfalls / follow-up questions**
- *Is a render prop only `children`?* No, any function prop (`renderItem`, `renderEmpty`).
- *Performance?* Inline functions can bust memoization; hoist or memoize if needed.
- *Why less used now?* Hooks share logic without extra nesting.

---

<a id="q101"></a>
### Q101. Compare HOCs, render props and hooks.

**Short answer**
All three reuse stateful logic. HOCs wrap components, render props pass a render function, and hooks share logic by direct function call inside a component. Hooks are the default now: flatter, composable, type-friendly; HOCs/render props remain for wrapping or inversion of rendering control.

**Detailed explanation**
| Aspect | HOC | Render prop | Hook |
|---|---|---|---|
| Nesting | Wrapper layers | Nested callbacks | None |
| Data flow | Implicit injected props | Explicit args | Explicit return value |
| Name collisions | Possible | No | No |
| Composition | `compose(a, b)` | Nested | Multiple calls |
| TypeScript | Hard generics | Fine | Easiest |
| Controls rendering/markup | Can wrap | Yes | No (logic only) |
| Conditional/early return use | Yes | Yes | Rules of Hooks apply |

Choose hooks for logic; render props when the shared component must render markup around consumer-defined content (e.g. a list virtualizer); HOCs for cross-cutting wrappers applied at definition time (error boundary, route guard) or when you must wrap class components.

**Code example or diagram**
```jsx
// Hook
function useWindowSize() {
  const [s, setS] = useState({ w: innerWidth, h: innerHeight });
  useEffect(() => {
    const f = () => setS({ w: innerWidth, h: innerHeight });
    addEventListener("resize", f);
    return () => removeEventListener("resize", f);
  }, []);
  return s;
}
const A = () => { const { w } = useWindowSize(); return <p>{w}</p>; };

// Render prop
const WindowSize = ({ children }) => children(useWindowSize());
const B = () => <WindowSize>{({ w }) => <p>{w}</p>}</WindowSize>;

// HOC
const withWindowSize = (C) => (p) => <C {...p} size={useWindowSize()} />;
```

**Common pitfalls / follow-up questions**
- *Can hooks fully replace the others?* Not for class components or when you need to inject markup/boundaries.
- *Does sharing a hook share state?* No; each call has its own state.
- *Migration path?* Extract the HOC body into a hook, keep HOC as thin adapter.

---

<a id="q102"></a>
### Q102. What is the compound components pattern?

**Short answer**
A set of components that work together sharing implicit state through context, forming a flexible declarative API (`<Tabs><Tabs.List/><Tabs.Panel/></Tabs>`), like `<select>`/`<option>`. The consumer controls structure; the parent coordinates behavior.

**Detailed explanation**
The root holds state and exposes it via context; children read it. Prefer context over `React.Children.map`/`cloneElement`, which breaks with wrappers or fragments. Attach subcomponents as static properties or named exports. Benefits: less prop drilling, flexible layout/ordering, small API. Add accessibility (ARIA roles, keyboard nav), and throw a helpful error if a child is used outside its root. Support controlled and uncontrolled modes (Q103).

**Code example or diagram**
```jsx
const TabsCtx = createContext(null);
const useTabs = () => {
  const c = useContext(TabsCtx);
  if (!c) throw new Error("Tabs.* must be used within <Tabs>");
  return c;
};

function Tabs({ defaultValue, children }) {
  const [value, setValue] = useState(defaultValue);
  const ctx = useMemo(() => ({ value, setValue }), [value]);
  return <TabsCtx.Provider value={ctx}>{children}</TabsCtx.Provider>;
}
Tabs.List = ({ children }) => <div role="tablist">{children}</div>;
Tabs.Tab = ({ value, children }) => {
  const { value: active, setValue } = useTabs();
  return (
    <button role="tab" aria-selected={active === value} onClick={() => setValue(value)}>
      {children}
    </button>
  );
};
Tabs.Panel = ({ value, children }) =>
  useTabs().value === value ? <div role="tabpanel">{children}</div> : null;

<Tabs defaultValue="a">
  <Tabs.List><Tabs.Tab value="a">A</Tabs.Tab><Tabs.Tab value="b">B</Tabs.Tab></Tabs.List>
  <Tabs.Panel value="a">Panel A</Tabs.Panel>
  <Tabs.Panel value="b">Panel B</Tabs.Panel>
</Tabs>
```

**Common pitfalls / follow-up questions**
- *`cloneElement` vs context?* Context works through arbitrary nesting; cloneElement only direct children.
- *Rerenders?* Memoize context value; split if needed.
- *Ordering/IDs for a11y?* Use `useId` to link tab and panel.

---

<a id="q103"></a>
### Q103. What are the controlled vs uncontrolled component patterns?

**Short answer**
A controlled component's value lives in React state and is driven via props (`value` + `onChange`); an uncontrolled one keeps its value in the DOM and is read via refs or `FormData` (`defaultValue`). Reusable components often support both.

**Detailed explanation**
Controlled: single source of truth, instant validation/formatting, conditional UI; cost is a render per keystroke. Uncontrolled: less code and fewer renders, good for simple forms, file inputs (always uncontrolled), and integration with non-React code. React 19 form Actions work naturally with uncontrolled inputs and `FormData`. Pitfall: switching between controlled and uncontrolled (value `undefined` → defined) triggers warnings. For library components, implement a `useControllableState` hook: if `value` prop is provided use it, otherwise internal state, calling `onChange` either way.

**Code example or diagram**
```jsx
function useControllableState({ value, defaultValue, onChange }) {
  const [inner, setInner] = useState(defaultValue);
  const isControlled = value !== undefined;
  const current = isControlled ? value : inner;
  const set = useCallback((next) => {
    if (!isControlled) setInner(next);
    onChange?.(next);
  }, [isControlled, onChange]);
  return [current, set];
}

function Toggle(props) {
  const [on, setOn] = useControllableState({
    value: props.on, defaultValue: props.defaultOn ?? false, onChange: props.onChange,
  });
  return <button aria-pressed={on} onClick={() => setOn(!on)}>{on ? "On" : "Off"}</button>;
}

// Uncontrolled form
function Signup() {
  return (
    <form action={(fd) => save(Object.fromEntries(fd))}>
      <input name="email" defaultValue="" />
      <button>Submit</button>
    </form>
  );
}
```

**Common pitfalls / follow-up questions**
- *Why the warning "changing an uncontrolled input to be controlled"?* Initial `value` was `undefined`; use `""`.
- *`defaultValue` vs `value`?* `defaultValue` only sets the initial value.
- *Reset an uncontrolled component?* Change its `key`.

---

<a id="q104"></a>
### Q104. What are the container/presentational and smart/dumb patterns?

**Short answer**
Presentational (dumb) components render UI from props; container (smart) components fetch data and manage state, passing it down. It separates concerns and eases testing and Storybook use. Hooks now often replace the container with a custom hook.

**Detailed explanation**
Dan Abramov popularized it, later noting hooks make the split less necessary. Today the idea is retained as "UI component + logic hook" or, in RSC, server component fetches and passes props to client UI. Benefits: presentational components are pure, reusable, easy to snapshot/test; logic is testable independently. Avoid dogmatic splits that create needless files for trivial components.

**Code example or diagram**
```jsx
// Presentational
function UserList({ users, onSelect, loading }) {
  if (loading) return <Skeleton />;
  return <ul>{users.map((u) => <li key={u.id} onClick={() => onSelect(u.id)}>{u.name}</li>)}</ul>;
}

// Logic as a hook
function useUsers() {
  const { data = [], isLoading } = useQuery({ queryKey: ["users"], queryFn: fetchUsers });
  return { users: data, loading: isLoading };
}

// Container
function UserListContainer() {
  const { users, loading } = useUsers();
  const navigate = useNavigate();
  return <UserList users={users} loading={loading} onSelect={(id) => navigate(`/users/${id}`)} />;
}
```

**Common pitfalls / follow-up questions**
- *Is it still recommended?* As a guideline, yes; as a rule, no—hooks give the same separation.
- *Where do presentational components get state?* They can hold UI-only state (open/closed).
- *How does it map to RSC?* Server component = container, client component = presentational/interactive.

---

<a id="q105"></a>
### Q105. What are the provider and observer patterns?

**Short answer**
Provider pattern shares data through the tree with Context without prop drilling. Observer (pub/sub) lets subjects notify subscribers on change; in React it underlies stores and `useSyncExternalStore`, Redux, Zustand, and event emitters.

**Detailed explanation**
Providers suit low-frequency, tree-scoped values (theme, auth, i18n, DI). For high-frequency updates, an observable store with selectors avoids re-rendering all consumers. `useSyncExternalStore(subscribe, getSnapshot)` is the correct primitive to subscribe to external observables safely with concurrent rendering; snapshots must be cached/immutable to avoid infinite loops.

**Code example or diagram**
```jsx
import { useSyncExternalStore } from "react";

function createStore(initial) {
  let state = initial;
  const listeners = new Set();
  return {
    get: () => state,
    set(partial) {
      state = { ...state, ...partial };
      listeners.forEach((l) => l());
    },
    subscribe(l) { listeners.add(l); return () => listeners.delete(l); },
  };
}

const cartStore = createStore({ count: 0 });

function useCart(selector) {
  return useSyncExternalStore(
    cartStore.subscribe,
    () => selector(cartStore.get())
  );
}

const Badge = () => <span>{useCart((s) => s.count)}</span>;
```

**Common pitfalls / follow-up questions**
- *Why can selectors returning new objects loop forever?* `getSnapshot` must return a stable value when unchanged.
- *Provider vs store?* Provider distributes; store holds changing state.
- *Memory leaks?* Always return the unsubscribe.

---

<a id="q106"></a>
### Q106. What is the state reducer pattern?

**Short answer**
A component or hook exposes its internal reducer so consumers can pass a custom reducer that intercepts or overrides state transitions. It provides inversion of control without adding many props.

**Detailed explanation**
Popularized by Kent C. Dodds. The hook runs `useReducer(userReducer ?? defaultReducer)`, or passes the default reducer in so the user can delegate: `reducer(state, action, defaultReducer)`. Exporting action types lets consumers hook into specific transitions (e.g. prevent toggle after 4 clicks). It beats adding `shouldToggle`-style flag props, and plays well with headless libraries (Downshift's `stateReducer`).

**Code example or diagram**
```jsx
const actions = { toggle: "toggle", reset: "reset" };

function toggleReducer(state, action) {
  switch (action.type) {
    case actions.toggle: return { on: !state.on };
    case actions.reset: return { on: action.initialOn };
    default: throw new Error(`Unhandled ${action.type}`);
  }
}

function useToggle({ initialOn = false, reducer = toggleReducer } = {}) {
  const [state, dispatch] = useReducer(reducer, { on: initialOn });
  return {
    on: state.on,
    toggle: () => dispatch({ type: actions.toggle }),
    reset: () => dispatch({ type: actions.reset, initialOn }),
  };
}

function App() {
  const clicks = useRef(0);
  const { on, toggle } = useToggle({
    reducer(state, action) {
      if (action.type === actions.toggle && clicks.current >= 4) return state;
      if (action.type === actions.toggle) clicks.current++;
      return toggleReducer(state, action);
    },
  });
  return <button onClick={toggle}>{on ? "On" : "Off"}</button>;
}
```

**Common pitfalls / follow-up questions**
- *Reducer purity?* Mutating a ref in a reducer (as above for brevity) is impure and StrictMode double-invokes it; prefer tracking counts in state.
- *Stability?* An inline reducer changes each render; that is OK for `useReducer` but keep logic pure.
- *Versus controlled props?* State reducer customizes transitions, controlled props customize ownership of state.

---

<a id="q107"></a>
### Q107. What are headless components (Radix, Headless UI)?

**Short answer**
Headless components provide behavior, state, accessibility and keyboard handling with no styling or opinionated markup, so teams apply their own design. Examples: Radix Primitives, Headless UI, React Aria, Ark UI, TanStack Table, Downshift.

**Detailed explanation**
Building accessible dialogs, menus, comboboxes is hard (focus trap, ARIA, typeahead, portals, collision detection). Headless libraries solve this and let you style with Tailwind/CSS. APIs use compound components, `asChild`/slot patterns (Radix `Slot`) to merge props onto your element, and prop getters (`getInputProps`) or hooks. shadcn/ui copies styled Radix-based components into your repo. Trade-offs: more styling work, API learning, dependency on library a11y quality.

**Code example or diagram**
```jsx
import * as Dialog from "@radix-ui/react-dialog";

export function ConfirmDialog({ onConfirm }) {
  return (
    <Dialog.Root>
      <Dialog.Trigger asChild>
        <button className="btn">Delete</button>
      </Dialog.Trigger>
      <Dialog.Portal>
        <Dialog.Overlay className="fixed inset-0 bg-black/40" />
        <Dialog.Content className="fixed left-1/2 top-1/2 -translate-x-1/2 -translate-y-1/2 rounded bg-white p-6">
          <Dialog.Title>Are you sure?</Dialog.Title>
          <Dialog.Description>This cannot be undone.</Dialog.Description>
          <Dialog.Close asChild><button onClick={onConfirm}>Yes</button></Dialog.Close>
        </Dialog.Content>
      </Dialog.Portal>
    </Dialog.Root>
  );
}
```

**Common pitfalls / follow-up questions**
- *What does `asChild` do?* Renders the child element instead of a default one, merging props/refs.
- *Why headless in a design system?* Reuse tested a11y behavior while owning visuals.
- *Pitfall?* Child must forward refs and spread props to work with `asChild`.

---

<a id="q108"></a>
### Q108. How do you build a design system or component library?

**Short answer**
Start with design tokens, then accessible primitives, composed components, and documentation, packaged as a versioned library with consistent APIs, theming, and tests. Govern it with contribution guidelines and treat it as a product.

**Detailed explanation**
Layers: (1) **tokens** (color, spacing, typography as CSS variables/JSON, e.g. Style Dictionary); (2) **primitives** (headless/a11y behavior); (3) **components** (Button, Input, Dialog) with variants; (4) **patterns/templates**. Practices: consistent props API (`variant`, `size`, `asChild`), `forwardRef` (ref is a normal prop in React 19) and prop spreading, theming via CSS variables or Tailwind, TypeScript types, accessibility tests (axe, Testing Library), visual regression (Chromatic/Playwright), tree-shakable ESM output with `sideEffects` set, peer-dependency on React, and SSR/RSC compatibility (`"use client"` banners). Avoid over-abstraction; build from real product needs.

**Code example or diagram**
```jsx
import { cva } from "class-variance-authority";
import { forwardRef } from "react";

const button = cva("inline-flex items-center rounded font-medium focus-visible:ring-2", {
  variants: {
    variant: { primary: "bg-brand text-white", ghost: "bg-transparent" },
    size: { sm: "h-8 px-3 text-sm", md: "h-10 px-4" },
  },
  defaultVariants: { variant: "primary", size: "md" },
});

export const Button = forwardRef(function Button({ variant, size, className, ...rest }, ref) {
  return <button ref={ref} className={button({ variant, size, className })} {...rest} />;
});
```

**Common pitfalls / follow-up questions**
- *Build vs adopt (MUI, shadcn)?* Adopt when brand needs are standard; build/wrap when you need control.
- *How to prevent breaking changes?* Semver, deprecation cycles, codemods, visual tests.
- *Styling approach?* CSS variables + zero-runtime CSS scale better than runtime CSS-in-JS with RSC.

---

<a id="q109"></a>
### Q109. How do you version, document (Storybook) and publish it?

**Short answer**
Use semantic versioning automated with Changesets (or semantic-release), document components with Storybook stories and autodocs, and publish ESM+types to npm (public or private registry) via CI. Deploy Storybook as the living docs site.

**Detailed explanation**
- **Versioning**: semver (major = breaking). Changesets: contributors add a changeset file per PR; CI opens a "Version Packages" PR, then publishes. Conventional Commits + semantic-release is an alternative.
- **Docs**: Storybook CSF stories per component state, `autodocs` tag, controls, a11y addon, interaction tests (`play` functions), and MDX for guidelines. Publish with Chromatic or static hosting.
- **Publishing**: build with tsup/Vite library mode to ESM (+CJS if needed) with `.d.ts`; `package.json` `exports`, `types`, `sideEffects`, `files`, `peerDependencies` (react, react-dom). Use `npm publish --provenance`, check with `publint`/`arethetypeswrong`.
- Provide changelogs and migration guides/codemods.

**Code example or diagram**
```jsx
// Button.stories.jsx
import { Button } from "./Button";
import { expect, userEvent, within } from "storybook/test";

export default {
  title: "Components/Button",
  component: Button,
  tags: ["autodocs"],
  argTypes: { variant: { control: "select", options: ["primary", "ghost"] } },
};

export const Primary = { args: { children: "Save", variant: "primary" } };

export const Clickable = {
  args: { children: "Click", onClick: () => {} },
  play: async ({ canvasElement, args }) => {
    await userEvent.click(within(canvasElement).getByRole("button"));
    await expect(args.onClick).toHaveBeenCalled();
  },
};
```

```json
{
  "name": "@acme/ui",
  "version": "1.4.0",
  "type": "module",
  "exports": { ".": { "types": "./dist/index.d.ts", "import": "./dist/index.js" } },
  "sideEffects": ["**/*.css"],
  "peerDependencies": { "react": ">=18", "react-dom": ">=18" }
}
```

**Common pitfalls / follow-up questions**
- *Why peerDependencies for React?* Avoid duplicate React copies, which break hooks.
- *What counts as a breaking change?* Removing/renaming props, changing default behavior, dropping a React version.
- *Pre-releases?* Use `next`/`beta` dist-tags.

---

<a id="q110"></a>
### Q110. How do you structure a large React project (feature folders, layers)?

**Short answer**
Organize by feature (domain) rather than file type, with each feature owning its components, hooks, API and state, plus shared layers (`shared/ui`, `lib`) and enforced dependency direction (features don't import each other's internals). Expose a public API per feature.

**Detailed explanation**
Type-based folders (`components/`, `hooks/`) don't scale: changes scatter. Feature folders colocate related code, easing ownership, deletion, and code splitting. Guidelines: app shell (routing, providers) → pages/routes → features → shared. Enforce boundaries with ESLint (`eslint-plugin-boundaries`, `no-restricted-imports`) or Nx module-boundary rules. Colocate tests/styles; keep a feature `index.ts` as its public surface; avoid cyclical imports; keep global state minimal and server state in a query cache.

**Code example or diagram**
```jsx
// src/
//   app/            providers, router, entry
//   features/
//     orders/
//       api/        ordersApi.js
//       components/ OrderTable.jsx
//       hooks/      useOrders.js
//       index.js    <- public API
//     auth/
//   shared/
//     ui/           Button, Modal
//     lib/          http client, utils
//
// features/orders/index.js
export { OrdersPage } from "./components/OrdersPage";
export { useOrders } from "./hooks/useOrders";

// eslint: forbid deep imports
// "no-restricted-imports": ["error", { patterns: ["@/features/*/*"] }]
```

**Common pitfalls / follow-up questions**
- *Where does shared code go?* Only promote to `shared` after two or more real usages.
- *Feature-to-feature imports?* Prefer via public API or lift to a higher layer/composition in the page.
- *Is it overkill for small apps?* Yes, start flat and refactor when pain appears.

---

<a id="q111"></a>
### Q111. What is feature-sliced design?

**Short answer**
Feature-Sliced Design (FSD) is a frontend architecture methodology with standardized layers, slices, and segments, where a module may import only from layers below it. Layers: `app`, `processes` (deprecated), `pages`, `widgets`, `features`, `entities`, `shared`.

**Detailed explanation**
- **Layers** (top to bottom): app → pages → widgets → features → entities → shared. Import rule: higher layers import lower, never upward or sideways between slices of the same layer.
- **Slices** partition layers by business domain (`user`, `order`).
- **Segments** split slices by purpose: `ui`, `model`, `api`, `lib`, `config`.
- Each slice exposes a **public API** (`index.ts`); consumers must not deep-import.
Benefits: predictable structure, low coupling, onboarding. Costs: learning curve, boilerplate, debates over where things belong (feature vs entity). Tools: `@feature-sliced/steiger` linter.

**Code example or diagram**
```jsx
// src/
//  app/        providers, router
//  pages/      checkout/ { ui, model, index.js }
//  widgets/    header/
//  features/   add-to-cart/ { ui, model, api, index.js }
//  entities/   product/     { ui, model, api, index.js }
//  shared/     ui, lib, api, config

// features/add-to-cart/ui/AddToCartButton.jsx
import { ProductPrice } from "@/entities/product";   // lower layer OK
import { Button } from "@/shared/ui";                // lower layer OK
// import { Cart } from "@/features/cart";           // same layer: NOT allowed

export function AddToCartButton({ product }) {
  return <Button>Add — <ProductPrice value={product.price} /></Button>;
}
```

**Common pitfalls / follow-up questions**
- *Feature vs entity?* Entity = business noun (product); feature = user action with value (add to cart).
- *Cross-slice need?* Move down to a shared lower layer or compose in an upper layer.
- *Does it fit Next.js?* Yes, keep `app/` routing thin and re-export from `pages` layer.

---

<a id="q112"></a>
### Q112. What is a monorepo (Nx, Turborepo)? When is it useful?

**Short answer**
A monorepo holds multiple apps and packages in one repository, using workspaces (pnpm/npm/yarn) plus a build orchestrator (Nx, Turborepo) that caches and runs tasks by dependency graph. It's useful for sharing code (design system, utils), atomic cross-package changes, and consistent tooling.

**Detailed explanation**
Features: task pipelines (`build` depends on dependencies' `build`), local and remote caching, affected-only runs (`nx affected`), generators/boundaries (Nx), simple config (Turborepo). Benefits: one version of dependencies, atomic refactors, easy internal package consumption. Costs: CI complexity, repo size, tooling setup, need for ownership rules (CODEOWNERS), and permission granularity. Use it when multiple apps share libraries or teams collaborate closely; avoid for unrelated, independently released projects with different tooling.

**Code example or diagram**
```jsx
// turbo.json
// {
//   "tasks": {
//     "build": { "dependsOn": ["^build"], "outputs": ["dist/**", ".next/**"] },
//     "test": { "dependsOn": ["build"] },
//     "lint": {}
//   }
// }

// apps/web/package.json -> "dependencies": { "@acme/ui": "workspace:*" }

// apps/web/src/App.jsx
import { Button } from "@acme/ui"; // resolved to packages/ui through the workspace
export default () => <Button>Hello</Button>;
```

**Common pitfalls / follow-up questions**
- *Turborepo vs Nx?* Turborepo is lightweight task running/caching; Nx adds project graph, generators, plugins, boundary enforcement.
- *Internal packages: build or consume source?* Source ("just-in-time") is simpler; build for published packages.
- *Phantom dependencies?* pnpm's strict node_modules prevents them.

---

<a id="q113"></a>
### Q113. What are micro-frontends? Explain Module Federation.

**Short answer**
Micro-frontends split a large UI into independently developed and deployed pieces owned by separate teams, composed at build time, runtime, or the server. Module Federation (webpack 5 / Rspack / `@module-federation/vite`) lets a "host" load modules from "remotes" at runtime and share dependencies like React.

**Detailed explanation**
Integration options: iframes, Web Components, server-side composition/edge includes, single-spa, and Module Federation. In MF, each remote exposes modules (`./Checkout`), hosts declare remotes (URLs to `remoteEntry.js`), and `shared` configures singletons (React must be `singleton` to avoid duplicates and hook errors) with version ranges. Benefits: independent deploys, team autonomy, incremental migrations. Costs: operational complexity, duplicated payloads, inconsistent UX, versioning/contract drift, harder local dev and testing, cross-app state/communication (use events, URL, or small shared contracts). Don't adopt for small teams.

**Code example or diagram**
```jsx
// host: webpack.config.js
const { ModuleFederationPlugin } = require("webpack").container;
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: "shell",
      remotes: { checkout: "checkout@https://cdn.acme.com/checkout/remoteEntry.js" },
      shared: { react: { singleton: true, requiredVersion: "^18.0.0" }, "react-dom": { singleton: true } },
    }),
  ],
};

// remote: exposes: { "./CheckoutApp": "./src/CheckoutApp" }

// host usage
const CheckoutApp = lazy(() => import("checkout/CheckoutApp"));
export default () => (
  <ErrorBoundary fallback={<p>Checkout unavailable</p>}>
    <Suspense fallback={<Spinner />}><CheckoutApp /></Suspense>
  </ErrorBoundary>
);
```

**Common pitfalls / follow-up questions**
- *Why singleton React?* Two React copies break hooks/context.
- *Remote down?* Wrap in an Error Boundary with a fallback.
- *Sharing state across MFEs?* Prefer events/URL/props; avoid a shared global store.

---

<a id="q114"></a>
### Q114. What is clean/hexagonal architecture in the frontend?

**Short answer**
It organizes code so business rules (domain/use cases) depend on nothing, while UI, HTTP, and storage are adapters plugged in through interfaces ("ports"). Dependencies point inward, making core logic framework-agnostic and testable.

**Detailed explanation**
Layers: **domain** (entities, pure rules), **application** (use cases orchestrating via ports), **infrastructure** (adapters: REST client, localStorage), **presentation** (React components/hooks calling use cases). React becomes just one adapter. Gains: swap backends, test logic without React, reduce framework lock-in. Costs: extra indirection and boilerplate; for CRUD-heavy UIs it's overkill. Apply selectively to complex domains (pricing, workflows).

**Code example or diagram**
```jsx
// domain/cart.js  (pure)
export const addItem = (cart, item) =>
  cart.items.some((i) => i.id === item.id)
    ? { ...cart, items: cart.items.map((i) => (i.id === item.id ? { ...i, qty: i.qty + 1 } : i)) }
    : { ...cart, items: [...cart.items, { ...item, qty: 1 }] };

// application/addToCart.js  (use case, depends on a port)
export const makeAddToCart = ({ cartRepo }) => async (item) => {
  const cart = addItem(await cartRepo.load(), item);
  await cartRepo.save(cart);
  return cart;
};

// infrastructure/httpCartRepo.js  (adapter)
export const httpCartRepo = (http) => ({
  load: () => http.get("/cart"),
  save: (c) => http.put("/cart", c),
});

// presentation
function AddButton({ item }) {
  const { addToCart } = useServices();
  return <button onClick={() => addToCart(item)}>Add</button>;
}
```

**Common pitfalls / follow-up questions**
- *Port vs adapter?* Port is the interface the core needs; adapter implements it.
- *Is it worth it for CRUD?* Usually not; use feature folders plus a query library.
- *Where does React state live?* Presentation layer, wrapping use cases (e.g. in query mutations).

---

<a id="q115"></a>
### Q115. How do you separate UI, business logic and data access?

**Short answer**
Keep components for rendering, custom hooks for orchestration/UI-state, pure functions for business rules, and a data layer (API client + query hooks) for fetching and caching. Dependencies flow UI → hooks → domain/services → API client.

**Detailed explanation**
- **Data access**: API functions return typed data; React Query/SWR hooks wrap them and own server state caching.
- **Business logic**: pure functions in `domain/` (pricing, validation, permissions) — trivial to unit test with no React.
- **Hooks**: combine query and domain functions into view-models (`useCheckout`).
- **UI**: dumb components receiving props.
Avoid putting `fetch` or business calculations inside JSX/components, and don't store server data in Redux by hand.

**Code example or diagram**
```jsx
// data: api/orders.js
export const fetchOrders = (http) => http.get("/orders");

// domain: orders.js
export const totalOf = (order) => order.lines.reduce((s, l) => s + l.price * l.qty, 0);
export const isRefundable = (order, now = Date.now()) =>
  order.status === "paid" && now - order.paidAt < 30 * 864e5;

// hook: useOrders.js
export function useOrders() {
  const { data, isLoading } = useQuery({ queryKey: ["orders"], queryFn: () => fetchOrders(http) });
  const orders = useMemo(
    () => (data ?? []).map((o) => ({ ...o, total: totalOf(o), refundable: isRefundable(o) })),
    [data]
  );
  return { orders, isLoading };
}

// UI
export function OrdersTable() {
  const { orders, isLoading } = useOrders();
  if (isLoading) return <Skeleton />;
  return <Table rows={orders} />;
}
```

**Common pitfalls / follow-up questions**
- *Where to put validation?* Schema (zod) in domain, reused by forms and API boundary.
- *Testing?* Unit-test domain; test hooks with `renderHook` and mocked API (MSW).
- *Is a hook "business logic"?* It's orchestration; keep core rules framework-free.

---

<a id="q116"></a>
### Q116. How do you apply SOLID in React?

**Short answer**
S: one reason to change per component/hook. O: extend via props/composition, not editing. L: substitutable components honoring the same prop contract. I: small, focused prop interfaces. D: depend on abstractions (injected services/props/context), not concrete modules.

**Detailed explanation**
- **SRP**: split data fetching hook from presentational component; one hook per concern.
- **OCP**: use `children`, slots, render props, variants so new behavior doesn't require editing the component.
- **LSP**: a `PrimaryButton` should accept everything `button` does (`ComponentPropsWithoutRef<"button">`) and behave consistently; wrappers must forward props/refs.
- **ISP**: don't pass an entire `user` object when a component needs only `name`; avoid giant prop bags.
- **DIP**: components receive `onSave`/service via props or context so tests and environments can swap implementations.

**Code example or diagram**
```jsx
// OCP + ISP: list is closed for modification, open via renderItem
function List({ items, renderItem, getKey }) {
  return <ul>{items.map((i) => <li key={getKey(i)}>{renderItem(i)}</li>)}</ul>;
}

// LSP: wrapper keeps the native button contract
function IconButton({ icon, children, ...rest }) {
  return <button {...rest}>{icon}{children}</button>;
}

// DIP: depends on an injected abstraction
function SaveForm({ onSave }) {
  return <form action={(fd) => onSave(Object.fromEntries(fd))}><input name="title" /><button>Save</button></form>;
}
// production: <SaveForm onSave={api.saveDoc} />  test: <SaveForm onSave={vi.fn()} />
```

**Common pitfalls / follow-up questions**
- *Over-application?* Splitting everything creates indirection; apply when change/testing pain is real.
- *SRP for a component?* "Renders X" and "decides when/how data loads" are separate reasons to change.
- *Hooks and DIP?* Pass dependencies in, or read them from context.

---

<a id="q117"></a>
### Q117. What are barrel files, and what are their bundle-size costs?

**Short answer**
Barrel files are `index.js` files that re-export many modules (`export * from "./Button"`) for convenient imports. They can hurt bundle size, build/test/dev-server speed, and tree shaking, especially with side-effectful modules or CJS dependencies.

**Detailed explanation**
Importing one name from a barrel makes the bundler/Jest/dev server load and parse every module it re-exports. Production bundlers can tree shake pure ESM barrels, but fail when modules have side effects (or `sideEffects` isn't declared false) or when the library ships CJS (`lodash`, large icon packs). Dev servers (Vite) and Jest don't tree shake, so cold start and test time grow. Circular dependencies also arise through barrels. Mitigations: import from the specific path, set `"sideEffects": false`, use `optimizePackageImports` (Next.js) for big libs, keep barrels only as a feature's public API (small), or use lint rules (`eslint-plugin-barrel-files`, `no-restricted-imports`).

**Code example or diagram**
```jsx
// icons/index.js (barrel)
export * from "./Home";
export * from "./User";
// ...1,500 more

// Costly in dev/test and in non-tree-shakeable setups
import { Home } from "@/icons";

// Preferred
import { Home } from "@/icons/Home";

// next.config.js
// module.exports = { experimental: { optimizePackageImports: ["@mui/icons-material", "lodash-es"] } };
```

**Common pitfalls / follow-up questions**
- *Do barrels always bloat production bundles?* No; pure ESM + `sideEffects: false` tree-shakes fine, but it is fragile.
- *Why slow tests?* Jest resolves/transforms every re-exported file.
- *Are feature public-API barrels OK?* Yes, if small and without side effects.

---

<a id="q118"></a>
### Q118. How do you manage dependency injection or service layers in React?

**Short answer**
Use Context as the DI container: provide service instances (API client, analytics, feature flags) at the app root, and read them through typed hooks. Tests supply fakes via the same providers. Props injection and factory functions are lighter alternatives.

**Detailed explanation**
React has no built-in DI container, but Context is scoped, swappable, and tree-aware. Create services as plain objects/classes outside components (stable identity), wire them in a composition root, and expose `useServices()`. Avoid decorator-based IoC containers (InversifyJS) unless the team needs them; they add weight and hurt tree shaking. For non-React code (e.g., store middleware), pass dependencies as arguments (`createStore({ api })`, thunk `extraArgument`).

**Code example or diagram**
```jsx
const ServicesCtx = createContext(null);

export function ServicesProvider({ services, children }) {
  return <ServicesCtx.Provider value={services}>{children}</ServicesCtx.Provider>;
}
export function useServices() {
  const s = useContext(ServicesCtx);
  if (!s) throw new Error("ServicesProvider missing");
  return s;
}

// composition root
const services = { orders: createOrdersService(http), analytics: createAnalytics() };
root.render(<ServicesProvider services={services}><App /></ServicesProvider>);

// consumer
function useOrders() {
  const { orders } = useServices();
  return useQuery({ queryKey: ["orders"], queryFn: orders.list });
}

// test
render(<ServicesProvider services={{ orders: { list: async () => [] } }}><Orders /></ServicesProvider>);
```

**Common pitfalls / follow-up questions**
- *Creating services inside the provider component?* Create once outside or in `useState(() => ...)` to keep identity stable.
- *Context vs module imports?* Plain imports are simpler; use DI when you need swapping per test/environment/tenant.
- *Re-renders?* Services rarely change, so context is cheap.

---

<a id="q119"></a>
### Q119. How do you design an API client layer (interceptors, retry, error mapping)?

**Short answer**
Wrap `fetch`/axios in a single typed client that handles base URL, auth headers, token refresh, timeouts/cancellation, retries with backoff for idempotent requests, and maps failures to a consistent `ApiError`. Feature code then calls typed functions and never touches raw HTTP.

**Detailed explanation**
Concerns: **request interceptors** (auth, correlation IDs, locale), **response interceptors** (401 → refresh token once and replay; unwrap data), **retry** (only network errors, 502/503/504, 429 with `Retry-After`; exponential backoff + jitter; never auto-retry non-idempotent POSTs without idempotency keys), **timeouts and AbortSignal**, **error mapping** (HTTP/network/validation → `ApiError` with `kind`, `status`, `code`, user-safe message), **observability** (logging), and **runtime validation** (zod). Let TanStack Query handle caching and its own retry policy (avoid retry-on-retry multiplication). Axios has interceptors built in; with fetch, write a small wrapper or use `ky`/`ofetch`.

**Code example or diagram**
```jsx
export class ApiError extends Error {
  constructor(kind, status, message, details) {
    super(message);
    Object.assign(this, { kind, status, details });
  }
}

const RETRYABLE = new Set([502, 503, 504]);
const sleep = (ms) => new Promise((r) => setTimeout(r, ms));

export async function http(path, { method = "GET", body, signal, retries = 2 } = {}) {
  for (let attempt = 0; ; attempt++) {
    try {
      const res = await fetch(`${import.meta.env.VITE_API_URL}${path}`, {
        method,
        signal,
        headers: { "Content-Type": "application/json", Authorization: `Bearer ${getToken()}` },
        body: body ? JSON.stringify(body) : undefined,
      });
      if (res.status === 401 && attempt === 0 && (await refreshToken())) continue;
      if (!res.ok) {
        if (RETRYABLE.has(res.status) && method === "GET" && attempt < retries) {
          await sleep(2 ** attempt * 300 + Math.random() * 100);
          continue;
        }
        const problem = await res.json().catch(() => ({}));
        throw new ApiError(res.status >= 500 ? "server" : "client", res.status, problem.title ?? "Request failed", problem);
      }
      return res.status === 204 ? null : res.json();
    } catch (e) {
      if (e instanceof ApiError || e.name === "AbortError") throw e;
      if (method === "GET" && attempt < retries) { await sleep(2 ** attempt * 300); continue; }
      throw new ApiError("network", 0, "Network error");
    }
  }
}
```

**Common pitfalls / follow-up questions**
- *Concurrent 401s?* Share a single in-flight refresh promise to avoid refresh storms.
- *Why jitter?* Prevents synchronized retry spikes.
- *Retry POST?* Only with idempotency keys.

---

<a id="q120"></a>
### Q120. What is an anti-corruption layer for backend DTOs?

**Short answer**
An anti-corruption layer (ACL) is a mapping boundary that converts backend DTOs into frontend domain models, so UI code doesn't depend on API shape, naming, or quirks. When the backend changes, only the mapper changes.

**Detailed explanation**
DTOs often have snake_case, nullable fields, string dates/money, legacy flags, or inconsistent enums. Map them once at the data boundary into well-typed models (camelCase, `Date`/`Temporal`, value objects, defaults, discriminated unions), and validate with zod/valibot to fail fast on contract drift. Also map outbound models back to request DTOs. Place mappers beside the API function (or in query `select`), test them with fixtures. Costs: extra code and allocation; worth it for volatile or third-party APIs, and for large apps with multiple consumers. Tools: zod `.transform`, OpenAPI/GraphQL codegen for DTO types.

**Code example or diagram**
```jsx
import { z } from "zod";

const UserDto = z.object({
  user_id: z.string(),
  full_name: z.string().nullable(),
  created_at: z.string().datetime(),
  acct_status: z.enum(["A", "S", "D"]),
});

const STATUS = { A: "active", S: "suspended", D: "deleted" };

// ACL: DTO -> domain model
export const toUser = (dto) => {
  const d = UserDto.parse(dto);
  return {
    id: d.user_id,
    name: d.full_name ?? "Unknown",
    createdAt: new Date(d.created_at),
    status: STATUS[d.acct_status],
  };
};

export const fetchUser = async (id) => toUser(await http(`/users/${id}`));

// UI only knows the domain model
function UserCard({ user }) {
  return <p>{user.name} ({user.status})</p>;
}
```

**Common pitfalls / follow-up questions**
- *Where to map: fetcher or `select`?* In the fetcher keeps cache domain-shaped; `select` keeps raw DTO cached and maps per observer.
- *Strict vs lenient parsing?* Strict in dev/test to catch drift; in production log and degrade gracefully.
- *Overkill?* For stable, internal APIs with one consumer, light mapping suffices.
