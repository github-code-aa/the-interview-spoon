# React Interview Guide — Forms, Data Fetching & Routing

## Table of Contents
- [Q61. Handling forms](#q61)
- [Q62. Validating forms (Zod, Yup)](#q62)
- [Q63. Fetching data & pitfalls](#q63)
- [Q64. TanStack Query, SWR, RTK Query](#q64)
- [Q65. Caching, SWR & invalidation](#q65)
- [Q66. Pagination, infinite scroll, optimistic updates](#q66)
- [Q67. React Router, nested routes, loaders, actions](#q67)
- [Q68. Protecting routes](#q68)
- [Q69. 404s and redirects](#q69)
- [Q70. URL state (search params)](#q70)
- [Q71. File uploads with progress](#q71)
- [Q72. WebSockets, SSE, polling](#q72)

---

<a id="q61"></a>
### Q61. How do you handle forms (React Hook Form, Formik)?

**Short answer**
Forms can be *controlled* (state drives inputs) or *uncontrolled* (the DOM holds values, read via refs/FormData). Libraries like React Hook Form (RHF) use uncontrolled inputs plus refs for minimal re-renders; Formik uses controlled state and is simpler conceptually but re-renders more. RHF is the modern default for performance and bundle size.

**Detailed explanation**
- **Controlled**: `value` + `onChange` per field; every keystroke re-renders the component. Fine for small forms, painful for large ones.
- **Uncontrolled**: let the browser own values; read on submit. React 19 also adds `<form action={fn}>`, `useActionState` and `useFormStatus`.
- **React Hook Form**: `register` attaches a ref and handlers; values live outside React state, so typing doesn't re-render the form. `formState` is a Proxy — only the properties you read subscribe to updates. Use `Controller`/`useController` for controlled UI components (MUI, custom selects). `useFieldArray` handles dynamic lists. `watch` re-renders; `useWatch` isolates re-renders.
- **Formik**: `useFormik`/`<Formik>` keeps `values`, `errors`, `touched` in state (reducer), re-rendering the form on each change. Maintenance has slowed; many new projects choose RHF.
- Both integrate with schema validators (Zod/Yup) through resolvers/adapters.
- Accessibility: label every input, link errors with `aria-describedby`, set `aria-invalid`.

**Code example**
```jsx
import { useForm } from "react-hook-form";

export function SignupForm({ onSubmitToApi }) {
  const {
    register,
    handleSubmit,
    reset,
    formState: { errors, isSubmitting },
  } = useForm({ defaultValues: { email: "", age: "" } });

  const onSubmit = async (data) => {
    await onSubmitToApi(data);
    reset();
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)} noValidate>
      <label htmlFor="email">Email</label>
      <input
        id="email"
        type="email"
        aria-invalid={!!errors.email}
        aria-describedby="email-err"
        {...register("email", { required: "Email is required" })}
      />
      {errors.email && <p id="email-err" role="alert">{errors.email.message}</p>}

      <label htmlFor="age">Age</label>
      <input
        id="age"
        type="number"
        {...register("age", { valueAsNumber: true, min: { value: 18, message: "Must be 18+" } })}
      />
      {errors.age && <p role="alert">{errors.age.message}</p>}

      <button disabled={isSubmitting}>{isSubmitting ? "Saving…" : "Sign up"}</button>
    </form>
  );
}
```

**Common pitfalls / follow-up questions**
- *Why doesn't my custom/MUI input work with `register`?* It doesn't forward a ref / isn't a native input; wrap it with `Controller` or `useController`.
- *Switching an input from uncontrolled to controlled warning?* `value` was `undefined` initially; always provide `defaultValues`.
- *Why does the whole form re-render?* Using `watch()` at the top level or reading `formState` props you don't need; use `useWatch` in a child component.

---

<a id="q62"></a>
### Q62. How do you validate forms (Zod, Yup)?

**Short answer**
Define a schema (Zod or Yup) describing valid data and plug it into the form library via a resolver (`zodResolver`, `yupResolver`). Zod is TypeScript-first with inferred types (`z.infer`), so one schema gives both runtime validation and static types. Always re-validate on the server — client validation is UX only.

**Detailed explanation**
- **Zod**: `z.object({...})`, composable, parses *and* transforms (`z.coerce.number()`), supports `refine`/`superRefine` for cross-field rules, `safeParse` returns a result instead of throwing.
- **Yup**: similar fluent API (`yup.object({...})`), async validation built-in, types inferred via `InferType` but less precise.
- **Timing**: RHF `mode` — `onSubmit` (default), `onBlur`, `onChange`, `onTouched`. Revalidation after submit is controlled by `reValidateMode`.
- **Cross-field rules** (password confirm): use `.refine(..., { path: ["confirm"] })` so the error lands on the right field.
- **Server errors**: map them back with `setError("email", { message })`.
- **Shared schema**: the same Zod schema can validate on client and server (Node, Server Actions).

**Code example**
```jsx
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";

const schema = z
  .object({
    email: z.string().email("Invalid email"),
    password: z.string().min(8, "Min 8 characters"),
    confirm: z.string(),
  })
  .refine((v) => v.password === v.confirm, {
    path: ["confirm"],
    message: "Passwords do not match",
  });

export function RegisterForm({ register: api }) {
  const {
    register,
    handleSubmit,
    setError,
    formState: { errors },
  } = useForm({ resolver: zodResolver(schema), mode: "onTouched" });

  const onSubmit = async (values) => {
    try {
      await api(values);
    } catch (e) {
      if (e.code === "EMAIL_TAKEN") setError("email", { message: "Email already in use" });
    }
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input placeholder="Email" {...register("email")} />
      <p>{errors.email?.message}</p>
      <input type="password" placeholder="Password" {...register("password")} />
      <p>{errors.password?.message}</p>
      <input type="password" placeholder="Confirm" {...register("confirm")} />
      <p>{errors.confirm?.message}</p>
      <button>Create account</button>
    </form>
  );
}
```

**Common pitfalls / follow-up questions**
- *Number inputs return strings.* Use `z.coerce.number()` or RHF `valueAsNumber`.
- *`refine` on an object doesn't run until all fields pass.* In Zod, object-level refinements only run if the base shape is valid; that's why confirm-mismatch may show late.
- *Zod vs Yup?* Zod: better TS inference, smaller runtime in v4, transforms; Yup: mature, async-friendly, familiar with Formik.

---

<a id="q63"></a>
### Q63. How do you fetch data in React? What are the pitfalls (race conditions, cancellation)?

**Short answer**
Classically in `useEffect` with `fetch`, storing `data/loading/error` in state — but this needs manual handling of cancellation, races, caching, and retries. Prefer a data library (TanStack Query, SWR, RTK Query) or framework loaders/Server Components. If you do it manually, use `AbortController` and an ignore flag in cleanup.

**Detailed explanation**
- **Race condition**: request A (old query) resolves after request B (new query) and overwrites newer state. Fix: abort in cleanup, or an `ignore` flag.
- **Cancellation**: `AbortController.signal` passed to `fetch`; abort in effect cleanup. An aborted fetch rejects with `AbortError` — ignore it.
- **StrictMode** (dev) mounts effects twice, which exposes missing cleanup and causes duplicate requests; that is intentional.
- **Other pitfalls**: setting state after unmount (harmless in React 18+, but wasteful), request waterfalls (child fetches only after parent renders), no dedupe/caching, forgetting `response.ok` (fetch doesn't reject on 404/500), missing error/empty states, unstable dependencies causing infinite loops.
- **Alternatives**: the `use()` hook with Suspense and promises cached by a framework, Server Components, router loaders (fetch before render, avoiding waterfalls).

**Code example**
```jsx
import { useEffect, useState } from "react";

function useUser(userId) {
  const [state, setState] = useState({ data: null, error: null, loading: true });

  useEffect(() => {
    const controller = new AbortController();
    setState({ data: null, error: null, loading: true });

    (async () => {
      try {
        const res = await fetch(`/api/users/${userId}`, { signal: controller.signal });
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        const data = await res.json();
        setState({ data, error: null, loading: false });
      } catch (error) {
        if (error.name === "AbortError") return;
        setState({ data: null, error, loading: false });
      }
    })();

    return () => controller.abort();
  }, [userId]);

  return state;
}

export function Profile({ userId }) {
  const { data, error, loading } = useUser(userId);
  if (loading) return <p>Loading…</p>;
  if (error) return <p role="alert">{error.message}</p>;
  return <h1>{data.name}</h1>;
}
```

**Common pitfalls / follow-up questions**
- *Why does my effect fetch twice in dev?* StrictMode double-invokes effects; with proper cleanup it's safe. Production runs once.
- *Can the effect callback be `async`?* No — it would return a promise instead of a cleanup function; define an inner async function.
- *How to avoid waterfalls?* Fetch in parallel (`Promise.all`), hoist to route loaders, or prefetch.

---

<a id="q64"></a>
### Q64. What are React Query/TanStack Query, SWR and RTK Query?

**Short answer**
They are *server-state* libraries that handle fetching, caching, deduping, background refetching, retries and mutations so you don't hand-write effects. TanStack Query is the most feature-rich and framework-agnostic; SWR is lighter and simpler; RTK Query is built into Redux Toolkit and is ideal if you already use Redux.

**Detailed explanation**
- **TanStack Query**: `useQuery({ queryKey, queryFn })`, `useMutation`, `useInfiniteQuery`, `QueryClient` cache, devtools, `staleTime`/`gcTime`, retries with backoff, window-focus/reconnect refetch, prefetching, SSR hydration, Suspense support.
- **SWR** (Vercel): `useSWR(key, fetcher)`; implements stale-while-revalidate; small API, `mutate` for revalidation/optimistic updates, `useSWRInfinite`.
- **RTK Query**: define an `api` with `createApi`; endpoints generate hooks (`useGetPostsQuery`); tag-based invalidation (`providesTags`/`invalidatesTags`); lives in the Redux store so it integrates with other slices; codegen from OpenAPI available.
- **Choosing**: TanStack Query for most apps; SWR for simple read-heavy apps; RTK Query when Redux is already the architecture.

**Code example**
```jsx
import { QueryClient, QueryClientProvider, useQuery } from "@tanstack/react-query";

const queryClient = new QueryClient({
  defaultOptions: { queries: { staleTime: 60_000, retry: 2 } },
});

async function fetchTodos({ signal }) {
  const res = await fetch("/api/todos", { signal });
  if (!res.ok) throw new Error("Failed to load todos");
  return res.json();
}

function Todos() {
  const { data, isPending, isError, error, isFetching } = useQuery({
    queryKey: ["todos"],
    queryFn: fetchTodos,
  });

  if (isPending) return <p>Loading…</p>;
  if (isError) return <p role="alert">{error.message}</p>;
  return (
    <>
      {isFetching && <small>Refreshing…</small>}
      <ul>{data.map((t) => <li key={t.id}>{t.title}</li>)}</ul>
    </>
  );
}

export default function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <Todos />
    </QueryClientProvider>
  );
}
```

**Common pitfalls / follow-up questions**
- *Query key mistakes?* Keys must include every variable the query depends on (`["todo", id]`); otherwise stale/shared data.
- *`isPending` vs `isFetching`?* Pending = no data yet; fetching = any in-flight request including background refetches.
- *Do I still need Redux?* Not for server data; keep Redux/Zustand for client state only.

---

<a id="q65"></a>
### Q65. Explain caching, stale-while-revalidate and invalidation.

**Short answer**
Caching stores fetched data by key so repeated requests are instant and deduplicated. Stale-while-revalidate returns cached (possibly stale) data immediately and refetches in the background to update it. Invalidation marks cached data as outdated (after a mutation, say) so it gets refetched.

**Detailed explanation**
- **Keyed cache**: identical keys share one request and one result across components.
- **TanStack Query timings**: `staleTime` — how long data is considered fresh (default 0, so refetch on mount/focus); `gcTime` — how long *unused* cache entries are retained (default 5 min) before garbage collection.
- **SWR flow**: serve cache → revalidate → update if different. Triggers: mount, window focus, reconnect, interval.
- **Invalidation strategies**: `queryClient.invalidateQueries({ queryKey })` (matches prefix), `setQueryData` for direct writes from mutation responses, `removeQueries`, RTK Query tags, SWR `mutate(key)`.
- **HTTP layer**: `Cache-Control`, `ETag`, and `stale-while-revalidate` directives are separate from the client cache but complementary.
- **Hard problem**: choosing freshness. Static reference data → long `staleTime`; volatile data (prices) → short or polling.

**Code example**
```jsx
import { useMutation, useQuery, useQueryClient } from "@tanstack/react-query";

function useProjects() {
  return useQuery({
    queryKey: ["projects"],
    queryFn: () => fetch("/api/projects").then((r) => r.json()),
    staleTime: 5 * 60_000, // fresh for 5 min: no refetch on remount
    gcTime: 30 * 60_000,
  });
}

function AddProject() {
  const qc = useQueryClient();
  const add = useMutation({
    mutationFn: (name) =>
      fetch("/api/projects", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ name }),
      }).then((r) => r.json()),
    onSuccess: () => qc.invalidateQueries({ queryKey: ["projects"] }),
  });

  return <button onClick={() => add.mutate("New project")} disabled={add.isPending}>Add</button>;
}
```

**Common pitfalls / follow-up questions**
- *Stale vs gc time confusion?* Stale = "should I refetch?"; gc = "should I delete from memory?".
- *Invalidate everything after every mutation?* Wasteful; target specific keys, or update cache directly with `setQueryData`.
- *Does invalidation refetch inactive queries?* It marks them stale; only active (mounted) queries refetch immediately.

---

<a id="q66"></a>
### Q66. How do you implement pagination, infinite scroll and optimistic updates?

**Short answer**
Pagination: include the page/cursor in the query key and use `placeholderData: keepPreviousData` for smooth transitions. Infinite scroll: `useInfiniteQuery` with `getNextPageParam` plus an IntersectionObserver sentinel. Optimistic updates: update the cache immediately in `onMutate`, roll back in `onError`, and re-sync in `onSettled`.

**Detailed explanation**
- **Offset vs cursor pagination**: offset is simple but unstable if data changes; cursor-based is robust and efficient for feeds.
- **Infinite scroll**: pages are stored as `data.pages`; `fetchNextPage`, `hasNextPage`, `isFetchingNextPage`. Use IntersectionObserver rather than scroll events; virtualize long lists (TanStack Virtual / react-window).
- **Optimistic update steps**: cancel outgoing refetches (`cancelQueries`) → snapshot previous data → write optimistic data → on error restore snapshot → on settle invalidate. React 19 also offers `useOptimistic` for local optimistic UI with actions.
- Keep URL page in sync (see Q70) for shareable/back-button-friendly pagination.

**Code example**
```jsx
import { useEffect, useRef } from "react";
import {
  useInfiniteQuery,
  useMutation,
  useQueryClient,
} from "@tanstack/react-query";

function usePosts() {
  return useInfiniteQuery({
    queryKey: ["posts"],
    queryFn: ({ pageParam, signal }) =>
      fetch(`/api/posts?cursor=${pageParam ?? ""}`, { signal }).then((r) => r.json()),
    initialPageParam: null,
    getNextPageParam: (last) => last.nextCursor ?? undefined,
  });
}

export function Feed() {
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage } = usePosts();
  const sentinel = useRef(null);

  useEffect(() => {
    const el = sentinel.current;
    if (!el) return;
    const io = new IntersectionObserver(([e]) => {
      if (e.isIntersecting && hasNextPage && !isFetchingNextPage) fetchNextPage();
    });
    io.observe(el);
    return () => io.disconnect();
  }, [hasNextPage, isFetchingNextPage, fetchNextPage]);

  return (
    <>
      {data?.pages.flatMap((p) => p.items).map((post) => <article key={post.id}>{post.title}</article>)}
      <div ref={sentinel} aria-hidden />
    </>
  );
}

export function useToggleLike() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (id) => fetch(`/api/posts/${id}/like`, { method: "POST" }),
    onMutate: async (id) => {
      await qc.cancelQueries({ queryKey: ["posts"] });
      const previous = qc.getQueryData(["posts"]);
      qc.setQueryData(["posts"], (old) => ({
        ...old,
        pages: old.pages.map((p) => ({
          ...p,
          items: p.items.map((i) => (i.id === id ? { ...i, liked: !i.liked } : i)),
        })),
      }));
      return { previous };
    },
    onError: (_e, _id, ctx) => qc.setQueryData(["posts"], ctx.previous),
    onSettled: () => qc.invalidateQueries({ queryKey: ["posts"] }),
  });
}
```

**Common pitfalls / follow-up questions**
- *Page flicker on page change?* Use `placeholderData: keepPreviousData`.
- *Duplicates in infinite lists?* Offset pagination with inserts; use cursors and dedupe by id.
- *Why cancel queries in `onMutate`?* An in-flight refetch could overwrite the optimistic value.

---

<a id="q67"></a>
### Q67. How does React Router work? What are nested routes, loaders and actions?

**Short answer**
React Router maps URL paths to components using the History API, rendering matched routes without full page reloads. Nested routes render child routes inside a parent's `<Outlet />`, giving shared layouts. In data routers (v6.4+/v7), `loader` fetches data before rendering and `action` handles form submissions/mutations, then the router revalidates loaders automatically.

**Detailed explanation**
- **Setup**: `createBrowserRouter(routes)` + `<RouterProvider>` (data mode). Declarative mode uses `<BrowserRouter><Routes>`.
- **Nested routes**: a route tree mirrors UI hierarchy; URL segments match nested levels; layout routes (no path) share UI; index routes render at parent's path.
- **Loaders**: run in parallel for all matched routes on navigation (avoids waterfalls), read with `useLoaderData`. Can `throw redirect()` or throw responses to reach `errorElement`.
- **Actions**: triggered by `<Form method="post">`, `useSubmit`, or `fetcher.Form`; after an action completes, loaders revalidate.
- **Other**: `useNavigation` for pending UI, `useFetcher` for non-navigating mutations, `lazy` routes for code splitting, `errorElement`. React Router v7 merges with Remix (framework mode, SSR).

**Code example**
```jsx
import {
  createBrowserRouter,
  RouterProvider,
  Outlet,
  Link,
  Form,
  useLoaderData,
  redirect,
} from "react-router-dom";

const router = createBrowserRouter([
  {
    path: "/",
    element: <Layout />,
    errorElement: <p>Something went wrong</p>,
    children: [
      { index: true, element: <h1>Home</h1> },
      {
        path: "contacts/:id",
        element: <Contact />,
        loader: async ({ params, request }) => {
          const res = await fetch(`/api/contacts/${params.id}`, { signal: request.signal });
          if (!res.ok) throw new Response("Not found", { status: 404 });
          return res.json();
        },
        action: async ({ params, request }) => {
          const form = await request.formData();
          await fetch(`/api/contacts/${params.id}`, {
            method: "PUT",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify(Object.fromEntries(form)),
          });
          return redirect(`/contacts/${params.id}`);
        },
      },
    ],
  },
]);

function Layout() {
  return (
    <>
      <nav><Link to="/">Home</Link></nav>
      <Outlet />
    </>
  );
}

function Contact() {
  const contact = useLoaderData();
  return (
    <Form method="post">
      <input name="name" defaultValue={contact.name} />
      <button>Save</button>
    </Form>
  );
}

export default function App() {
  return <RouterProvider router={router} />;
}
```

**Common pitfalls / follow-up questions**
- *Forgot `<Outlet />`?* Children never render.
- *`<a>` vs `<Link>`?* `<a>` triggers full reload; `<Link>` does client-side navigation.
- *Loader vs useEffect fetch?* Loaders start before render and in parallel, avoiding render-then-fetch waterfalls.

---

<a id="q68"></a>
### Q68. How do you protect routes (auth guards)?

**Short answer**
Wrap protected routes in a layout/guard component that checks auth state and renders `<Outlet />` or `<Navigate to="/login" />`, preserving the intended location. With data routers, do the check in a `loader` and `throw redirect("/login")` so nothing protected renders. Remember: client guards are UX — the server/API must enforce authorization.

**Detailed explanation**
- **Guard component**: works with declarative routing; handle the "auth loading" state to avoid flashing the login page.
- **Loader guard**: runs before render; ideal with data routers; can check cookies/session via an API call.
- **Return-to**: pass `state={{ from: location }}` or a `?redirect=` param and navigate back after login.
- **Role-based**: extend the guard with `allowedRoles`; show 403 page instead of redirecting when authenticated but unauthorized.
- **Security**: never rely on hidden UI; tokens in `httpOnly` cookies are safer than `localStorage`; JS bundles are public.

**Code example**
```jsx
import { Navigate, Outlet, useLocation, redirect } from "react-router-dom";
import { useAuth } from "./auth";

export function RequireAuth({ roles }) {
  const { user, status } = useAuth(); // status: "loading" | "ready"
  const location = useLocation();

  if (status === "loading") return <p>Checking session…</p>;
  if (!user) return <Navigate to="/login" replace state={{ from: location }} />;
  if (roles && !roles.includes(user.role)) return <Navigate to="/403" replace />;
  return <Outlet />;
}

// Usage: <Route element={<RequireAuth roles={["admin"]} />}><Route path="admin" element={<Admin />} /></Route>

// Data-router alternative
export async function protectedLoader({ request }) {
  const res = await fetch("/api/me", { credentials: "include" });
  if (res.status === 401) {
    const url = new URL(request.url);
    throw redirect(`/login?redirect=${encodeURIComponent(url.pathname + url.search)}`);
  }
  return res.json();
}
```

**Common pitfalls / follow-up questions**
- *Flash of login screen on refresh?* Not handling the initial "loading" auth state.
- *Open redirect risk?* Validate the `redirect` param is a same-origin relative path.
- *Is hiding a route secure?* No; enforce authorization on the API.

---

<a id="q69"></a>
### Q69. How do you handle 404s and redirects?

**Short answer**
Add a catch-all route (`path="*"`) for unknown URLs, and throw a 404 `Response` from loaders for missing resources so `errorElement` renders. For redirects use `<Navigate>` in components, `redirect()` in loaders/actions, or `useNavigate` imperatively. Use `replace` so redirects don't pollute history.

**Detailed explanation**
- **Unmatched URL**: `path: "*"` renders a NotFound component. Server must also fall back to `index.html` for SPAs (otherwise deep links 404 at the host) and ideally return real 404 status for SEO under SSR.
- **Resource not found**: loader throws `new Response("", { status: 404 })`; `useRouteError()` + `isRouteErrorResponse()` distinguishes it from crashes.
- **Redirects**: legacy paths (`/old` → `/new`), post-action redirects (Post/Redirect/Get), index redirects. `replace` prevents back-button loops.
- **Server-side**: permanent redirects (301) should be configured on the server/CDN for SEO.

**Code example**
```jsx
import {
  createBrowserRouter,
  Navigate,
  redirect,
  useRouteError,
  isRouteErrorResponse,
  Link,
} from "react-router-dom";

function RouteError() {
  const error = useRouteError();
  if (isRouteErrorResponse(error) && error.status === 404) {
    return <h1>Not found — <Link to="/">go home</Link></h1>;
  }
  return <h1>Unexpected error</h1>;
}

export const router = createBrowserRouter([
  {
    path: "/",
    errorElement: <RouteError />,
    children: [
      { path: "old-docs", loader: () => redirect("/docs") },
      { path: "home", element: <Navigate to="/" replace /> },
      {
        path: "posts/:id",
        element: <Post />,
        loader: async ({ params }) => {
          const res = await fetch(`/api/posts/${params.id}`);
          if (res.status === 404) throw new Response("Not Found", { status: 404 });
          return res.json();
        },
      },
      { path: "*", element: <h1>404 — Page not found</h1> },
    ],
  },
]);

function Post() { return null; }
```

**Common pitfalls / follow-up questions**
- *Refreshing a deep link gives a host 404?* Configure SPA fallback rewrites to `index.html`.
- *Redirect loop?* Guard and login redirecting to each other; add conditions and `replace`.
- *Soft 404 SEO issue?* Client-rendered 404 returns HTTP 200; use SSR to emit a real 404.

---

<a id="q70"></a>
### Q70. How do you manage URL state (search params)?

**Short answer**
Treat the URL as state for anything shareable or bookmarkable: filters, sort, page, tabs. Use `useSearchParams` from React Router (or `nuqs`/TanStack Router for typed params) and derive UI from it instead of duplicating in `useState`. The URL then works with back/forward, refresh and sharing.

**Detailed explanation**
- **Single source of truth**: read params, don't mirror them into state (avoids sync bugs).
- **Parsing**: all values are strings; parse/validate numbers and enums with defaults (Zod helps).
- **Updating**: `setSearchParams(prev => …)` functional form preserves other params; use `replace: true` for high-frequency changes (typing) so history isn't spammed.
- **Debounce** text search before writing to the URL.
- **Data**: include params in query keys/loader so data refetches when URL changes.
- **Path vs query**: identity in path (`/users/42`), view options in query (`?sort=name`).

**Code example**
```jsx
import { useSearchParams } from "react-router-dom";
import { useQuery } from "@tanstack/react-query";

export function ProductList() {
  const [params, setParams] = useSearchParams();
  const page = Math.max(1, Number(params.get("page")) || 1);
  const sort = ["name", "price"].includes(params.get("sort")) ? params.get("sort") : "name";
  const q = params.get("q") ?? "";

  const { data } = useQuery({
    queryKey: ["products", { page, sort, q }],
    queryFn: () => fetch(`/api/products?page=${page}&sort=${sort}&q=${encodeURIComponent(q)}`).then((r) => r.json()),
  });

  const update = (patch, opts) =>
    setParams((prev) => {
      const next = new URLSearchParams(prev);
      Object.entries(patch).forEach(([k, v]) => (v ? next.set(k, v) : next.delete(k)));
      return next;
    }, opts);

  return (
    <>
      <input
        value={q}
        onChange={(e) => update({ q: e.target.value, page: "1" }, { replace: true })}
        placeholder="Search"
      />
      <select value={sort} onChange={(e) => update({ sort: e.target.value, page: "1" })}>
        <option value="name">Name</option>
        <option value="price">Price</option>
      </select>
      <ul>{data?.items.map((p) => <li key={p.id}>{p.name}</li>)}</ul>
      <button disabled={page === 1} onClick={() => update({ page: String(page - 1) })}>Prev</button>
      <button onClick={() => update({ page: String(page + 1) })}>Next</button>
    </>
  );
}
```

**Common pitfalls / follow-up questions**
- *Losing other params on update?* Copy the previous params; don't build from scratch.
- *Back button stepping through each keystroke?* Use `replace: true`/debounce.
- *Arrays?* Use repeated keys (`getAll("tag")`) or comma-separated; stay consistent.

---

<a id="q71"></a>
### Q71. How do you upload files with progress?

**Short answer**
Use `<input type="file">`, send via `FormData`, and track progress with `XMLHttpRequest`'s `upload.onprogress` or Axios `onUploadProgress` — `fetch` has no upload progress event (streams are limited). For big files use presigned URLs and chunked/resumable uploads, and support cancel via abort.

**Detailed explanation**
- **Validation**: check type and size client-side (`file.size`, `file.type`) but re-validate on server.
- **Progress**: `event.lengthComputable`, `loaded / total`. Throttle UI updates if many files.
- **Cancel**: `xhr.abort()` or Axios `AbortController`.
- **Large files**: direct-to-storage via presigned S3/Azure SAS URLs, multipart/chunked uploads (tus protocol), retry per chunk.
- **UX**: drag-and-drop, previews with `URL.createObjectURL` (revoke afterwards), accessible status text.
- **Don't set `Content-Type` manually** with FormData — the browser adds the multipart boundary.

**Code example**
```jsx
import { useRef, useState } from "react";

function uploadFile(file, { onProgress, signal }) {
  return new Promise((resolve, reject) => {
    const xhr = new XMLHttpRequest();
    xhr.open("POST", "/api/upload");
    xhr.upload.onprogress = (e) => {
      if (e.lengthComputable) onProgress(Math.round((e.loaded / e.total) * 100));
    };
    xhr.onload = () =>
      xhr.status >= 200 && xhr.status < 300 ? resolve(JSON.parse(xhr.responseText)) : reject(new Error(`HTTP ${xhr.status}`));
    xhr.onerror = () => reject(new Error("Network error"));
    signal?.addEventListener("abort", () => {
      xhr.abort();
      reject(new DOMException("Aborted", "AbortError"));
    });
    const body = new FormData();
    body.append("file", file);
    xhr.send(body);
  });
}

export function Uploader() {
  const [progress, setProgress] = useState(0);
  const [status, setStatus] = useState("idle");
  const controller = useRef(null);

  const onChange = async (e) => {
    const file = e.target.files?.[0];
    if (!file) return;
    if (file.size > 10 * 1024 * 1024) return setStatus("File too large (max 10MB)");
    controller.current = new AbortController();
    setStatus("uploading");
    try {
      await uploadFile(file, { onProgress: setProgress, signal: controller.current.signal });
      setStatus("done");
    } catch (err) {
      setStatus(err.name === "AbortError" ? "cancelled" : "failed");
    }
  };

  return (
    <div>
      <input type="file" onChange={onChange} />
      <progress value={progress} max="100" aria-label="Upload progress" />
      <span aria-live="polite">{status} {progress}%</span>
      <button onClick={() => controller.current?.abort()}>Cancel</button>
    </div>
  );
}
```

**Common pitfalls / follow-up questions**
- *Why not `fetch`?* No standard upload-progress events (only download via streams).
- *Upload fails with 400 after setting `Content-Type`?* Boundary missing; let the browser set it.
- *Large files?* Chunk + resume, or presigned direct upload to avoid proxying through your server.

---

<a id="q72"></a>
### Q72. How do you handle WebSockets, SSE and polling?

**Short answer**
Polling repeatedly requests data on an interval (simple, works everywhere); SSE is a one-way server→client stream over HTTP with auto-reconnect; WebSockets are full-duplex for chat/collab/games. Manage connections in `useEffect` with cleanup, and feed incoming data into state or the query cache.

**Detailed explanation**
- **Polling**: `refetchInterval` in TanStack Query (pause when tab hidden via `refetchIntervalInBackground: false`). Wasteful but easy; long polling reduces latency.
- **SSE**: `EventSource` — text-only, automatic reconnection with `Last-Event-ID`, works through HTTP/proxies, limited connections per domain on HTTP/1.1. Great for notifications, dashboards, LLM token streaming.
- **WebSocket**: bidirectional, binary support; you must implement reconnection with exponential backoff, heartbeats, and auth (token in first message or short-lived ticket, since browser WS can't set headers).
- **React concerns**: open in effect, close in cleanup (StrictMode double-mount); don't set state after close; use refs for the socket; keep a single shared connection (context/store) rather than one per component; push updates into the query cache with `setQueryData`/invalidate.

**Code example**
```jsx
import { useEffect, useRef, useState } from "react";

export function useWebSocket(url, onMessage) {
  const [status, setStatus] = useState("connecting");
  const wsRef = useRef(null);
  const handlerRef = useRef(onMessage);
  handlerRef.current = onMessage;

  useEffect(() => {
    let retry = 0;
    let timer;
    let closedByUs = false;

    const connect = () => {
      const ws = new WebSocket(url);
      wsRef.current = ws;
      ws.onopen = () => { retry = 0; setStatus("open"); };
      ws.onmessage = (e) => handlerRef.current(JSON.parse(e.data));
      ws.onclose = () => {
        if (closedByUs) return;
        setStatus("reconnecting");
        timer = setTimeout(connect, Math.min(30_000, 500 * 2 ** retry++));
      };
    };
    connect();

    return () => {
      closedByUs = true;
      clearTimeout(timer);
      wsRef.current?.close();
    };
  }, [url]);

  const send = (data) => wsRef.current?.readyState === WebSocket.OPEN && wsRef.current.send(JSON.stringify(data));
  return { status, send };
}

export function useSSE(url, onEvent) {
  useEffect(() => {
    const es = new EventSource(url);
    es.onmessage = (e) => onEvent(JSON.parse(e.data));
    return () => es.close();
  }, [url]); // eslint-disable-line react-hooks/exhaustive-deps
}
```

**Common pitfalls / follow-up questions**
- *Stale closure in `onmessage`?* Store the handler in a ref (as above) or use functional state updates.
- *WS vs SSE?* SSE for server-push only (simpler, auto-reconnect); WS for bidirectional low-latency.
- *Scaling concerns?* Sticky sessions or pub/sub backplane (Redis), heartbeats, backpressure, and fallback to polling.
