# 09 – TypeScript with React (Q134–Q141)

## Table of Contents
- [Q134. Typing props, state, events and refs](#q134-how-do-you-type-props-state-events-and-refs)
- [Q135. FC vs plain function signatures](#q135-fc-vs-plain-function-signatures-which-do-you-prefer)
- [Q136. Typing generic components](#q136-how-do-you-type-generic-components)
- [Q137. Typing useReducer with discriminated unions](#q137-how-do-you-type-usereducer-with-discriminated-unions)
- [Q138. Typing context safely](#q138-how-do-you-type-context-safely)
- [Q139. Polymorphic components (`as` prop)](#q139-what-are-polymorphic-components-as-prop)
- [Q140. Typing custom hooks and HOCs](#q140-how-do-you-type-custom-hooks-and-hocs)
- [Q141. Sharing types between API and UI](#q141-how-do-you-share-types-between-the-api-and-ui-openapi-trpc-graphql-codegen)

---

### Q134. How do you type props, state, events and refs?

**Short answer**
Props use an `interface`/`type` on the function parameter; state is inferred from the initial value or given explicitly via `useState<T>()` when it can be `null` or a union; events use `React.ChangeEvent<HTMLInputElement>`-style types; refs use `useRef<HTMLDivElement>(null)` for DOM nodes. Lean on inference and let the compiler check JSX.

**Detailed explanation**
- **Props:** `type Props = { title: string; onSelect?: (id: string) => void; children?: React.ReactNode }`. Extend native element props with `React.ComponentProps<"button">` (or `React.ComponentPropsWithoutRef`) instead of re-declaring them. Use `React.ReactNode` for renderable children, `React.ReactElement` when it must be an element.
- **State:** `useState(0)` infers `number`. Use `useState<User | null>(null)` for nullable/async data, and a discriminated union for request status. Functional updates infer from the state type.
- **Events:** `React.MouseEvent<HTMLButtonElement>`, `React.ChangeEvent<HTMLInputElement>`, `React.FormEvent<HTMLFormElement>`, `React.KeyboardEvent`. Inline handlers are inferred contextually, so only annotate extracted handlers. Handler types: `React.ChangeEventHandler<HTMLInputElement>`.
- **Refs:** DOM refs: `useRef<HTMLInputElement>(null)` (read-only `.current`, nullable). Mutable instance value: `useRef<number>(0)` (in React 19 types, `useRef` always requires an argument). Forwarding: in React 19 `ref` is a regular prop; before that use `forwardRef<HTMLInputElement, Props>`. Expose methods with `useImperativeHandle`.
- Avoid `any` and non-null assertions; narrow instead.

**Code example**
```tsx
import { useRef, useState, type ChangeEvent, type ComponentProps, type FormEvent } from "react";

type Status =
  | { state: "idle" }
  | { state: "loading" }
  | { state: "error"; message: string };

type InputProps = Omit<ComponentProps<"input">, "onChange" | "value"> & {
  label: string;
  value: string;
  onValueChange: (value: string) => void;
};

function TextField({ label, value, onValueChange, ref, ...rest }: InputProps) {
  return (
    <label>
      {label}
      <input
        {...rest}
        ref={ref}
        value={value}
        onChange={(e: ChangeEvent<HTMLInputElement>) => onValueChange(e.target.value)}
      />
    </label>
  );
}

export function SearchForm({ onSearch }: { onSearch: (q: string) => Promise<void> }) {
  const [query, setQuery] = useState("");
  const [status, setStatus] = useState<Status>({ state: "idle" });
  const inputRef = useRef<HTMLInputElement>(null);

  async function handleSubmit(e: FormEvent<HTMLFormElement>) {
    e.preventDefault();
    setStatus({ state: "loading" });
    try {
      await onSearch(query);
      setStatus({ state: "idle" });
      inputRef.current?.focus();
    } catch (err) {
      setStatus({ state: "error", message: err instanceof Error ? err.message : "Unknown error" });
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      <TextField ref={inputRef} label="Search" value={query} onValueChange={setQuery} />
      {status.state === "error" && <p role="alert">{status.message}</p>}
    </form>
  );
}
```

**Common pitfalls / follow-up questions**
- *Why is `ref.current` possibly null?* The DOM node doesn't exist before mount/after unmount; use optional chaining or a guard.
- *`interface` vs `type` for props?* Either works; `interface` gives clearer errors and extension, `type` is needed for unions. Be consistent.
- *How do I type `e.target` vs `e.currentTarget`?* `currentTarget` is the typed element the handler is attached to; `target` is `EventTarget` and often needs narrowing.

---

### Q135. FC vs plain function signatures: which do you prefer?

**Short answer**
I prefer plain functions with typed props (`function Button(props: ButtonProps)`). `React.FC` historically added implicit `children` (removed in `@types/react` 18), makes generics, `defaultProps`, and function hoisting awkward, and adds little value. `FC` is not wrong, just unnecessary.

**Detailed explanation**
- `FC<Props>` gives return type `ReactNode`, static properties typing (`displayName`), and — formerly — implicit `children`. Since React 18 types the implicit children are gone, removing the main reason it caused bugs (and the main reason people used it).
- Drawbacks of `FC`: can't be generic (`const List: FC<ListProps<T>>` doesn't work), can't be a hoisted `function` declaration, and slightly noisier. Plain functions get return type inferred (`JSX.Element`) and work with generics and overloads naturally.
- Default values: use default parameters/destructuring defaults rather than `defaultProps` (deprecated for function components; removed in React 19).
- Return type annotation is usually unnecessary; if desired, `React.ReactNode`.
- Team consistency matters more than the choice; enforce via ESLint. Using `FC` is fine for typing function expressions assigned to variables, e.g. with `memo`.

**Code example**
```tsx
import type { ReactNode } from "react";

type CardProps = {
  title: string;
  variant?: "plain" | "outlined";
  children?: ReactNode;
};

// Preferred: plain function, explicit children, default via destructuring
export function Card({ title, variant = "plain", children }: CardProps) {
  return (
    <section className={variant}>
      <h2>{title}</h2>
      {children}
    </section>
  );
}

// FC works but can't express generics
import type { FC } from "react";
export const Badge: FC<{ label: string }> = ({ label }) => <span>{label}</span>;

// Generic requires a plain function
export function List<T>({ items, render }: { items: T[]; render: (item: T) => ReactNode }) {
  return <ul>{items.map((item, i) => <li key={i}>{render(item)}</li>)}</ul>;
}
```

**Common pitfalls / follow-up questions**
- *Did `FC` ever cause bugs?* Yes: implicit `children` let components accept children they never rendered (pre-18 types).
- *How do you type `memo` components?* `const Row = memo(function Row(props: RowProps) {...})` keeps the name and type; generics need a cast.
- *Should I annotate return types?* Rarely; inference is accurate, but for exported library components an explicit `React.JSX.Element`/`ReactNode` can stabilize API.

---

### Q136. How do you type generic components?

**Short answer**
Make the component a plain function with a type parameter (`function Select<T>(props: SelectProps<T>)`), and tie props together via `T` (items, `getKey`, `onChange`) so callers get inference. Add constraints with `extends` where needed. Wrapped components (`memo`, `forwardRef`) lose generics and need a cast helper.

**Detailed explanation**
- Generic components let the data type flow from `items` into render callbacks and handlers with no manual annotation.
- Constrain with `T extends { id: string }` to access fields safely, or use `keyof T` for column definitions (`Column<T>` with `key: keyof T`).
- For extra safety with relationships, use mapped/conditional types, e.g. `K extends keyof T` linking `key` and `render(value: T[K])`.
- In `.tsx` files, arrow generics need a trailing comma or constraint: `<T,>(props: P<T>) => ...`; function declarations avoid this.
- `forwardRef` and `memo` erase generics. Workaround: cast to the generic signature, e.g. `export const Table = memo(TableInner) as typeof TableInner;`. React 19 treating `ref` as a prop reduces the need for `forwardRef`.
- Provide sensible defaults (`T = unknown`) only when beneficial.

**Code example**
```tsx
import { memo, type ReactNode } from "react";

type Column<T> = {
  [K in keyof T]: {
    key: K;
    header: string;
    render?: (value: T[K], row: T) => ReactNode;
  };
}[keyof T];

type TableProps<T extends { id: string | number }> = {
  rows: readonly T[];
  columns: readonly Column<T>[];
  onRowClick?: (row: T) => void;
};

function TableInner<T extends { id: string | number }>({ rows, columns, onRowClick }: TableProps<T>) {
  return (
    <table>
      <thead>
        <tr>{columns.map(c => <th key={String(c.key)}>{c.header}</th>)}</tr>
      </thead>
      <tbody>
        {rows.map(row => (
          <tr key={row.id} onClick={() => onRowClick?.(row)}>
            {columns.map(c => (
              <td key={String(c.key)}>
                {c.render ? c.render(row[c.key], row) : String(row[c.key])}
              </td>
            ))}
          </tr>
        ))}
      </tbody>
    </table>
  );
}

// memo erases generics; restore them with a cast
export const Table = memo(TableInner) as typeof TableInner;

// Usage: T inferred as { id: number; name: string; price: number }
const products = [{ id: 1, name: "Pen", price: 2 }];
export const Demo = () => (
  <Table
    rows={products}
    columns={[
      { key: "name", header: "Name" },
      { key: "price", header: "Price", render: v => `$${v.toFixed(2)}` },
    ]}
    onRowClick={row => console.log(row.name)}
  />
);
```

**Common pitfalls / follow-up questions**
- *Why does `<T>(...) =>` fail in .tsx?* It parses as JSX; use `<T,>` or `<T extends unknown>`, or a function declaration.
- *Inference fails and gives `unknown`?* Ensure `T` appears in an inferable position (e.g. `items: T[]`) and avoid declaring it only in callbacks.
- *How to keep `memo` generic?* Cast as shown, or skip `memo` unless profiling shows a benefit.

---

### Q137. How do you type useReducer with discriminated unions?

**Short answer**
Define the state and a union of action types sharing a literal `type` discriminant, then write the reducer as `(state: State, action: Action): State`. TypeScript narrows `action` in each `switch` case, so payloads are type-checked and an exhaustive `never` check catches missing cases. `useReducer(reducer, initial)` infers everything.

**Detailed explanation**
- A discriminated union uses a common literal property (`type`) so the compiler narrows to the exact member within `case`.
- Model state itself as a discriminated union too (`status: "idle" | "loading" | "success" | "error"`) to make illegal states unrepresentable (no `data` while loading, etc.).
- Exhaustiveness: in `default`, assign `action` to a `never` variable; adding a new action causes a compile error until handled.
- Derive action creators or types with `as const`, or use `ReturnType`/helper types, but explicit unions are most readable.
- Return type annotation `: State` prevents accidentally returning partial objects.
- Dispatch type: `React.Dispatch<Action>`; pass `dispatch` through context safely, since its identity is stable.
- Reducers must be pure; no side effects in them.

**Code example**
```tsx
import { useReducer } from "react";

type Todo = { id: string; text: string; done: boolean };

type State =
  | { status: "idle"; todos: Todo[] }
  | { status: "saving"; todos: Todo[] }
  | { status: "error"; todos: Todo[]; error: string };

type Action =
  | { type: "added"; todo: Todo }
  | { type: "toggled"; id: string }
  | { type: "removed"; id: string }
  | { type: "saveStarted" }
  | { type: "saveFailed"; error: string };

function assertNever(x: never): never {
  throw new Error(`Unhandled action: ${JSON.stringify(x)}`);
}

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case "added":
      return { status: "idle", todos: [...state.todos, action.todo] };
    case "toggled":
      return { ...state, todos: state.todos.map(t => (t.id === action.id ? { ...t, done: !t.done } : t)) };
    case "removed":
      return { ...state, todos: state.todos.filter(t => t.id !== action.id) };
    case "saveStarted":
      return { status: "saving", todos: state.todos };
    case "saveFailed":
      return { status: "error", todos: state.todos, error: action.error };
    default:
      return assertNever(action);
  }
}

export function TodoApp() {
  const [state, dispatch] = useReducer(reducer, { status: "idle", todos: [] });
  return (
    <div>
      <button onClick={() => dispatch({ type: "added", todo: { id: crypto.randomUUID(), text: "Learn TS", done: false } })}>
        Add
      </button>
      {state.status === "error" && <p role="alert">{state.error}</p>}
      <ul>
        {state.todos.map(t => (
          <li key={t.id} onClick={() => dispatch({ type: "toggled", id: t.id })}>
            {t.done ? "✅" : "⬜"} {t.text}
          </li>
        ))}
      </ul>
    </div>
  );
}
```

**Common pitfalls / follow-up questions**
- *Why not a single action with optional payload?* It loses narrowing and allows invalid combinations (`{type:"added"}` with no `todo`).
- *How do you ensure exhaustiveness?* A `never` assignment in `default` (as above) or `satisfies never`.
- *Reducer vs `useState`?* Use a reducer for related state with many transitions or when next state depends on complex logic; it's also easy to unit test.

---

### Q138. How do you type context safely?

**Short answer**
Create the context with `createContext<Value | null>(null)` (or `undefined`), and expose a custom `useX` hook that throws if the context is missing, returning a non-nullable type. This avoids fake default values and `as` casts while giving consumers a clean API and a clear error if used outside the provider.

**Detailed explanation**
- A fake default like `{} as Value` hides missing providers and causes runtime errors later. A `null` default plus a guard hook makes misuse fail fast.
- Keep the provider and hook in one module; do not export the raw context unless necessary.
- Split state and dispatch into two contexts (or memoize the value) to avoid re-rendering consumers needlessly; `useMemo`/`useCallback` for the value object (React Compiler may handle it).
- In React 19, render `<Ctx value={...}>` directly (no `.Provider`) and read with `use(Ctx)`, which can be called conditionally.
- For complex contexts type the value with an explicit interface; use discriminated unions for states like `authenticated`/`anonymous`.
- A generic factory helper `createStrictContext<T>()` removes the boilerplate.

**Code example**
```tsx
import { createContext, useContext, useMemo, useState, type ReactNode } from "react";

type User = { id: string; name: string };
type AuthContextValue = {
  user: User | null;
  signIn: (user: User) => void;
  signOut: () => void;
};

const AuthContext = createContext<AuthContextValue | null>(null);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const value = useMemo<AuthContextValue>(
    () => ({ user, signIn: setUser, signOut: () => setUser(null) }),
    [user]
  );
  return <AuthContext value={value}>{children}</AuthContext>;
}

export function useAuth(): AuthContextValue {
  const ctx = useContext(AuthContext);
  if (ctx === null) throw new Error("useAuth must be used within <AuthProvider>");
  return ctx;
}

// Reusable strict-context factory
export function createStrictContext<T>(name: string) {
  const Ctx = createContext<T | null>(null);
  function useStrict(): T {
    const v = useContext(Ctx);
    if (v === null) throw new Error(`${name} hook used outside its provider`);
    return v;
  }
  return [Ctx, useStrict] as const;
}
```

**Common pitfalls / follow-up questions**
- *What if `null` is a legitimate value?* Use a unique sentinel (`Symbol`) or `undefined` as "missing" and `null` as value.
- *Why do all consumers re-render?* The provider's `value` identity changes; memoize it or split contexts by update frequency.
- *Context in Server Components?* Not supported; create the provider in a `"use client"` file and wrap children.

---

### Q139. What are polymorphic components (`as` prop)?

**Short answer**
A polymorphic component lets the caller choose the rendered element or component via an `as` prop (e.g. `<Text as="h1">`, `<Button as={Link} href="/">`). Typing it requires a generic `C extends ElementType` so props adapt to the chosen element, merging own props with `ComponentPropsWithoutRef<C>` and omitting conflicts.

**Detailed explanation**
- Benefits: reuse styling/behavior across semantic elements (heading, link, button) without wrappers, and keep accessibility correct.
- Core type: `type PolymorphicProps<C extends ElementType, Own = {}> = Own & { as?: C } & Omit<ComponentPropsWithoutRef<C>, keyof Own | "as">`.
- Defaults: `C extends ElementType = "span"` so omitting `as` still type-checks.
- Refs: add `ref?: ComponentPropsWithRef<C>["ref"]` (React 19: ref is a normal prop); previously needed `forwardRef` with a cast.
- Trade-offs: heavy type complexity, slower type checking, complicated error messages. Consider alternatives: the `asChild` pattern (Radix `Slot`) clones the child and merges props; or render props. Don't over-engineer for simple cases.
- When `as` is a custom component, its props are inferred (e.g. Next.js `Link` requires `href`).

**Code example**
```tsx
import type { ComponentPropsWithoutRef, ElementType, ReactNode } from "react";

type PolymorphicProps<C extends ElementType, Own extends object = {}> = Own & {
  as?: C;
} & Omit<ComponentPropsWithoutRef<C>, keyof Own | "as">;

type TextOwnProps = { size?: "sm" | "md" | "lg"; children?: ReactNode };

export function Text<C extends ElementType = "span">({
  as,
  size = "md",
  children,
  ...rest
}: PolymorphicProps<C, TextOwnProps>) {
  const Component: ElementType = as ?? "span";
  return (
    <Component data-size={size} {...rest}>
      {children}
    </Component>
  );
}

// Usage
export function Demo() {
  return (
    <>
      <Text>Default span</Text>
      <Text as="h1" size="lg" id="title">Heading</Text>
      <Text as="a" href="/docs" target="_blank">Link (href is valid here)</Text>
      {/* <Text as="span" href="/x" />  // ✗ error: href not allowed on span */}
    </>
  );
}
```

**Common pitfalls / follow-up questions**
- *Why omit `keyof Own`?* So your own prop (e.g. `size`, or `color` on native elements) doesn't conflict with the element's prop of the same name.
- *`as` vs `asChild`?* `asChild` (Radix Slot) composes onto your actual child element, better for custom components and ref/prop merging; `as` is simpler but harder to type.
- *Performance of types?* Polymorphic types are expensive; limit to design-system primitives.

---

### Q140. How do you type custom hooks and HOCs?

**Short answer**
Type hooks like normal functions: typed parameters and an explicit or inferred return type; use generics for reusable data hooks, and `as const` (or an explicit tuple type) when returning tuples. HOCs are typed with generics over the wrapped component's props, using `Omit` to remove injected props and `ComponentType<P>` for input/output. Prefer hooks over HOCs in new code.

**Detailed explanation**
- **Hooks:** return objects for many values (named, extensible) or `as const` tuples for `useState`-like APIs. Generic hooks (`useLocalStorage<T>`, `useFetch<T>`) accept `T` and expose `T | undefined`. Use discriminated union returns for async states. Type the setter as `Dispatch<SetStateAction<T>>`.
- Hooks can also take callbacks with typed args (`useEvent`-style), and refs (`RefObject<T | null>`).
- **HOCs:** signature `function withX<P extends InjectedProps>(Component: ComponentType<P>): ComponentType<Omit<P, keyof InjectedProps>>`. TypeScript struggles to prove `Omit<P, K> & K` is assignable to `P`, so a cast of props in the spread is a common (contained) compromise.
- Set `displayName` for devtools, hoist statics if needed (`hoist-non-react-statics`), and forward refs where relevant.
- HOCs have mostly been replaced by hooks, but remain useful for cross-cutting concerns (error boundaries, feature flags, legacy class components).

**Code example**
```tsx
import { useCallback, useEffect, useState, type ComponentType, type Dispatch, type SetStateAction } from "react";

// Generic custom hook returning a tuple
export function useLocalStorage<T>(key: string, initial: T): readonly [T, Dispatch<SetStateAction<T>>] {
  const [value, setValue] = useState<T>(() => {
    const raw = typeof window === "undefined" ? null : window.localStorage.getItem(key);
    return raw ? (JSON.parse(raw) as T) : initial;
  });
  useEffect(() => {
    window.localStorage.setItem(key, JSON.stringify(value));
  }, [key, value]);
  return [value, setValue] as const;
}

// Hook with discriminated union result
type AsyncState<T> =
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error"; error: Error };

export function useFetch<T>(url: string): AsyncState<T> & { refetch: () => void } {
  const [state, setState] = useState<AsyncState<T>>({ status: "loading" });
  const [tick, setTick] = useState(0);
  useEffect(() => {
    const ctrl = new AbortController();
    setState({ status: "loading" });
    fetch(url, { signal: ctrl.signal })
      .then(r => (r.ok ? (r.json() as Promise<T>) : Promise.reject(new Error(r.statusText))))
      .then(data => setState({ status: "success", data }))
      .catch(error => { if (!ctrl.signal.aborted) setState({ status: "error", error }); });
    return () => ctrl.abort();
  }, [url, tick]);
  const refetch = useCallback(() => setTick(t => t + 1), []);
  return { ...state, refetch };
}

// HOC injecting a `theme` prop
type WithTheme = { theme: "light" | "dark" };

export function withTheme<P extends WithTheme>(Component: ComponentType<P>) {
  function Themed(props: Omit<P, keyof WithTheme>) {
    const [theme] = useLocalStorage<WithTheme["theme"]>("theme", "light");
    return <Component {...(props as P)} theme={theme} />;
  }
  Themed.displayName = `withTheme(${Component.displayName ?? Component.name ?? "Component"})`;
  return Themed;
}
```

**Common pitfalls / follow-up questions**
- *Why does my tuple return widen to an array?* Without `as const` or an explicit tuple type, TS infers `(A | B)[]`.
- *Why the cast in the HOC?* TS can't prove `Omit<P, K> & {theme}` equals `P` for a generic `P`; the cast is localized and safe given the constraint.
- *Hooks vs HOCs?* Hooks compose without wrapper nesting, are easier to type and don't clash on prop names.

---

### Q141. How do you share types between the API and UI (OpenAPI, tRPC, GraphQL codegen)?

**Short answer**
Make a single source of truth and derive types from it: OpenAPI → generated TS types/client (`openapi-typescript`, `orval`), tRPC → infer types directly from server router (end-to-end TS without codegen), GraphQL → `graphql-codegen` generating typed operations/hooks. Optionally validate at runtime with Zod since TypeScript types vanish at runtime.

**Detailed explanation**
- **OpenAPI:** the backend (any language) publishes a spec. `openapi-typescript` generates `paths`/`components` types; `openapi-fetch` or `orval`/`hey-api` give a typed client (plus React Query hooks). Best for polyglot backends and public APIs. Run generation in CI and fail on drift; commit or generate at build.
- **tRPC:** TS monorepo (or shared package) only. Client imports `type AppRouter` (type-only import, no server code shipped) and gets inputs/outputs inferred; Zod validators define input at runtime. Excellent DX, tight coupling between client and server, not for public/third-party consumers.
- **GraphQL Codegen:** schema + `.graphql` documents → typed operation results/variables (`TypedDocumentNode`), fragments colocated with components. Works with Apollo/urql/React Query; the "client preset" gives type-safe `gql()`.
- **Zod/valibot shared schemas:** a shared package with schemas and `z.infer<typeof S>`; validate at boundaries (API responses, forms) to catch runtime drift.
- **Practices:** distinguish API DTO types from UI view models; map at the boundary; version APIs; use `Pick`/`Omit` rather than copy types; avoid hand-written duplicates; handle dates (JSON turns `Date` into string) with transformers like superjson or Zod coercion.

**Code example**
```tsx
// ---- tRPC (server) ----
// server/router.ts
import { initTRPC } from "@trpc/server";
import { z } from "zod";
const t = initTRPC.create();
export const appRouter = t.router({
  user: t.router({
    byId: t.procedure.input(z.object({ id: z.string() })).query(async ({ input }) => ({
      id: input.id,
      name: "Ada",
    })),
  }),
});
export type AppRouter = typeof appRouter;

// ---- tRPC (client) ----
// app/UserCard.tsx
import { createTRPCReact } from "@trpc/react-query";
import type { AppRouter } from "../server/router"; // type-only: no server code bundled
export const trpc = createTRPCReact<AppRouter>();

export function UserCard({ id }: { id: string }) {
  const { data, isPending } = trpc.user.byId.useQuery({ id }); // data: { id: string; name: string } | undefined
  if (isPending) return <p>Loading…</p>;
  return <p>{data?.name}</p>;
}

// ---- OpenAPI: generated types + typed client ----
// npx openapi-typescript https://api.example.com/openapi.json -o src/api/schema.d.ts
import createClient from "openapi-fetch";
import type { paths, components } from "./api/schema";

export type Product = components["schemas"]["Product"];
const client = createClient<paths>({ baseUrl: "https://api.example.com" });

export async function getProduct(id: string): Promise<Product> {
  const { data, error } = await client.GET("/products/{id}", { params: { path: { id } } });
  if (error) throw new Error("Failed to load product");
  return data;
}
```

**Common pitfalls / follow-up questions**
- *Do generated types guarantee runtime safety?* No; they're compile-time only. Validate responses with Zod (or generate Zod schemas via orval/hey-api) at trust boundaries.
- *tRPC vs OpenAPI vs GraphQL?* tRPC for full-stack TS monorepos; OpenAPI for language-agnostic/public REST; GraphQL when clients need flexible queries and fragment-driven data requirements.
- *How to prevent type drift?* Generate in CI, fail on a dirty diff, version the contract, and add contract tests.
