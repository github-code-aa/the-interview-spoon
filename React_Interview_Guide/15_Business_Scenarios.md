# 15. Business Scenarios (Q201-222)

## Table of Contents
- [Q201. Dashboard slow with 10,000 rows](#q201)
- [Q202. App re-renders constantly](#q202)
- [Q203. Stale data after mutations](#q203)
- [Q204. Real-time notification system](#q204)
- [Q205. Large-scale e-commerce front end](#q205)
- [Q206. Search with filters, facets, deep links](#q206)
- [Q207. Flaky or slow network](#q207)
- [Q208. Multi-tenant, white-label UI](#q208)
- [Q209. Incremental legacy migration](#q209)
- [Q210. CRA to Vite / Next.js](#q210)
- [Q211. 2 MB bundle](#q211)
- [Q212. Role-based admin portal](#q212)
- [Q213. Concurrent edits](#q213)
- [Q214. Schema-driven forms](#q214)
- [Q215. Large file uploads](#q215)
- [Q216. Charting / analytics dashboard](#q216)
- [Q217. Multiple teams, one codebase](#q217)
- [Q218. Reduce production errors after release](#q218)
- [Q219. Payments and PCI](#q219)
- [Q220. Autosave](#q220)
- [Q221. Safe rollout of risky feature](#q221)
- [Q222. Hard production bug (STAR)](#q222)

---

<a id="q201"></a>
### Q201. The dashboard is slow with 10,000 rows. How do you fix it?

**Context / clarifying questions**
- Is it slow on initial load, scroll, sort/filter, or cell edit? Which devices?
- Do users truly need all 10k rows client-side, or is a summary + drill-down enough?
- Are rows editable, selectable, expandable? Fixed or variable row height?
- Is data server-paginatable/sortable?

**Diagnosis or approach (step by step)**
1. Measure first: React Profiler (commit time, why-did-render), Chrome Performance (long tasks, layout), count DOM nodes (`document.querySelectorAll('*').length`). 10k rows x 10 cells = 100k+ nodes is the usual culprit.
2. Classify: too many DOM nodes (render/layout cost) vs. too many re-renders (state at wrong level) vs. expensive per-row work (formatting, inline sorting in render) vs. network payload size.
3. Fix in order of payoff: reduce data (server-side pagination/filter/sort) -> virtualize DOM -> stop unnecessary re-renders -> move heavy compute off the main thread.

**Solution with code or architecture diagram**
```
API (paged/cursor, server sort+filter) -> TanStack Query cache
   -> table state (sort/filter in URL) -> virtualized list (only ~30 rows in DOM)
```
```tsx
import { useVirtualizer } from '@tanstack/react-virtual';

const Row = React.memo(function Row({ row, style }: { row: Order; style: React.CSSProperties }) {
  return <div style={style} role="row">{row.id} {row.customer} {row.total}</div>;
});

function Grid({ rows }: { rows: Order[] }) {
  const parentRef = useRef<HTMLDivElement>(null);
  const v = useVirtualizer({ count: rows.length, getScrollElement: () => parentRef.current, estimateSize: () => 36, overscan: 8 });
  return (
    <div ref={parentRef} style={{ height: 600, overflow: 'auto' }} role="table">
      <div style={{ height: v.getTotalSize(), position: 'relative' }}>
        {v.getVirtualItems().map(i => (
          <Row key={rows[i.index].id} row={rows[i.index]}
               style={{ position: 'absolute', top: 0, transform: `translateY(${i.start}px)`, height: i.size, width: '100%' }} />
        ))}
      </div>
    </div>
  );
}
```
Additional: stable keys (ids, not index); `useMemo` for derived sorted/filtered arrays; `useDeferredValue`/`startTransition` for filter input; selection state kept as a `Set` of ids in a store with per-row selectors (so only changed rows re-render); Web Worker for sorting/aggregation of huge sets; `content-visibility: auto` as a cheap fallback; ARIA `aria-rowcount`/`aria-rowindex` to keep accessibility with virtualization.

**Tradeoffs**
- Virtualization breaks Ctrl+F, print, and simple a11y; needs ARIA and care with variable heights.
- Server-side paging adds latency and API work but scales best; client-side is snappier for <=a few thousand rows.
- `memo` adds comparison cost; use only where profiling shows benefit.

**How to present this in an interview**
Lead with "measure, then reduce data, then reduce DOM, then reduce renders". Quote before/after numbers (e.g., INP 900ms -> 120ms, DOM nodes 120k -> 2k). Mention you'd ask whether users really need 10k rows.

---

<a id="q202"></a>
### Q202. The app re-renders constantly. How do you debug it?

**Context / clarifying questions**
- Which component/interaction? Is it visible (jank) or just noticed in DevTools?
- Dev only (StrictMode double render) or production too?
- Infinite loop (effect/state cycle) or just excess renders?

**Diagnosis or approach (step by step)**
1. Rule out StrictMode: in dev, components render twice by design and effects mount/unmount/mount.
2. React DevTools Profiler: record the interaction, enable "Record why each component rendered", read "Hooks changed / Props changed / Parent rendered".
3. Common causes: state too high (context/provider value changing each render), new object/array/function props each render breaking `memo`, context with frequently changing value, `useEffect` that sets state it depends on, unstable keys (remount), derived state mirrored in `useState`, store subscriptions selecting whole state.
4. Confirm with `console.count`, `why-did-you-render`, or `<Profiler onRender>`.
5. Fix the root cause, re-measure.

**Solution with code or architecture diagram**
```tsx
// Problem: new value every render re-renders all consumers
<Ctx.Provider value={{ user, setUser }}>

// Fix: memoize, and split rarely/frequently changing contexts
const value = useMemo(() => ({ user, setUser }), [user]);

// Problem: effect loop
useEffect(() => { setItems(items.filter(Boolean)); }, [items]); // loops
// Fix: derive during render
const clean = useMemo(() => items.filter(Boolean), [items]);

// Problem: store subscribes to everything
const state = useStore();                 // re-renders on any change
const count = useStore(s => s.cart.count); // selector: only on count change
```
Other fixes: move state down (colocate) or lift content up (`children` pattern); `React.memo` + stable props via `useCallback`; keys by id; React Compiler can auto-memoize; for hot inputs use `useDeferredValue`.

**Tradeoffs**
- Blanket `memo/useCallback` clutters code and can slow things; fix structure first.
- Splitting contexts/stores adds boilerplate but cuts blast radius.
- Many renders are fine if cheap; only optimize what the profile shows.

**How to present this in an interview**
Show a methodology: profile -> identify "why rendered" -> classify cause -> structural fix -> verify. Give one concrete example (context value identity) and mention StrictMode as a false positive.

---

<a id="q203"></a>
### Q203. Users see stale data after mutations. What do you do?

**Context / clarifying questions**
- What data layer (Redux, React Query, SWR, Apollo, plain fetch)? Is there a CDN/HTTP cache or Next.js cache?
- Is staleness within the same client (own writes) or across users?
- Does the mutation response return the updated entity?

**Diagnosis or approach (step by step)**
1. Reproduce: mutate, then inspect network. Did the refetch occur? Was it served from cache (browser HTTP cache, CDN, service worker, Next.js Data/Router cache)?
2. Check cache keys: does the query key include all params? Different keys for list vs detail leaves one stale.
3. Check invalidation: after mutation, are all affected queries invalidated (list, detail, counts)?
4. Check races: a slow older GET overwriting a newer one; optimistic update not rolled back.
5. Check server: read replicas lag, caching headers.

**Solution with code or architecture diagram**
```tsx
const qc = useQueryClient();
const m = useMutation({
  mutationFn: updateTodo,
  onMutate: async (next) => {
    await qc.cancelQueries({ queryKey: ['todos'] });      // avoid race overwrite
    const prev = qc.getQueryData<Todo[]>(['todos']);
    qc.setQueryData<Todo[]>(['todos'], old => old?.map(t => t.id === next.id ? { ...t, ...next } : t));
    return { prev };
  },
  onError: (_e, _v, ctx) => qc.setQueryData(['todos'], ctx?.prev),
  onSettled: () => qc.invalidateQueries({ queryKey: ['todos'] }),   // reconcile with server
});
```
Next.js: `revalidateTag`/`revalidatePath` in the Server Action, `router.refresh()`. For cross-user freshness: polling, `refetchOnWindowFocus`, WebSocket/SSE pushes that invalidate keys. Use `Cache-Control: no-cache` + ETag for API reads that must be fresh. Use hierarchical key factories (`['todos','list',filters]`).

**Tradeoffs**
- Optimistic updates feel instant but need rollback and can show wrong state on failure.
- Aggressive refetch costs bandwidth; tune `staleTime` per data volatility.
- Pushing server-returned entity into cache avoids an extra request but duplicates normalization logic.

**How to present this in an interview**
Say "stale data is a cache-invalidation problem: find which cache layer is stale". Walk the layers (browser, client cache, CDN, server) and finish with the optimistic + invalidate pattern.

---

<a id="q204"></a>
### Q204. How would you design a real-time notification system?

**Context / clarifying questions**
- Scale (concurrent users), latency target, delivery guarantees (at-least-once?), channels (in-app, push, email)?
- Do notifications need read/unread state, grouping, preferences, multi-device sync?
- Browser support / corporate proxies that block WebSockets?

**Diagnosis or approach (step by step)**
1. Pick transport: SSE for one-way server push (simple, auto-reconnect, HTTP-friendly); WebSocket for bidirectional; long-polling as fallback; Web Push for when tab is closed.
2. Backend: event producers -> queue (Kafka/Service Bus) -> notification service -> per-user fan-out via gateway (SignalR/Socket.IO/managed service); persist in DB for history.
3. Client: single connection per tab (or SharedWorker/BroadcastChannel leader), reconnect with backoff + `Last-Event-ID` to replay missed events, dedupe by id.
4. State: merge pushes into the query cache; unread count in a small store; mark-read is optimistic.
5. UX: toast for urgent, badge/inbox for rest; accessible `aria-live="polite"` region; respect preferences.

**Solution with code or architecture diagram**
```
Services -> Queue -> Notification Svc -> DB (history, read state)
                         |-> Gateway (SSE/WS) -> Browser (single connection)
                         |-> Web Push / Email (offline users)
```
```tsx
function useNotifications() {
  const qc = useQueryClient();
  useEffect(() => {
    const es = new EventSource('/api/notifications/stream', { withCredentials: true });
    es.onmessage = (e) => {
      const n: Notification = JSON.parse(e.data);
      qc.setQueryData<Notification[]>(['notifications'], (old = []) =>
        old.some(x => x.id === n.id) ? old : [n, ...old]);   // dedupe
    };
    return () => es.close();   // EventSource reconnects automatically; server uses Last-Event-ID
  }, [qc]);
}
```
Also: on reconnect, refetch the list to close gaps; throttle bursts (batch updates with `startTransition`); pause when tab hidden if appropriate.

**Tradeoffs**
- SSE: simple but one-directional and limited connections per domain on HTTP/1.1; WebSocket: flexible but needs sticky/stateful infra.
- Polling: simplest, robust, but latency and load.
- At-least-once delivery requires client dedupe.

**How to present this in an interview**
Draw the diagram, justify the transport choice from requirements, and call out reconnect/replay, dedupe, and offline users (push) — those are what differentiate senior answers.

---

<a id="q205"></a>
### Q205. How would you design a large-scale e-commerce front end (cart, checkout, inventory)?

**Context / clarifying questions**
- Traffic/peaks, SEO importance, markets/locales/currencies, team count?
- Headless commerce backend? Guest checkout? Third-party payment provider?
- How real-time must inventory be?

**Diagnosis or approach (step by step)**
1. Rendering strategy per page: catalog/PDP via SSR/ISR + CDN (SEO, speed); cart/checkout client-driven, no-store; account pages CSR.
2. Data: server state in TanStack Query/RSC fetches; cart is server-owned (source of truth) with client optimistic updates; guest cart via cookie/ID merged at login.
3. Inventory: show coarse availability on PDP (cached, "low stock"), authoritative check at add-to-cart and again at checkout/order placement; reservation with TTL server-side.
4. Checkout: stepper as a state machine (address -> shipping -> payment -> review), each step validated and persisted; idempotency key on order submit.
5. Performance/resilience: image CDN, critical-path JS minimal on PDP, prefetch cart/checkout, graceful degradation if recommendations fail (error boundaries per widget).
6. Observability and experimentation: analytics events, A/B via flags, Web Vitals.

**Solution with code or architecture diagram**
```
CDN -> Edge/Next.js (ISR PDP, SSR search) -> BFF -> Catalog | Cart | Inventory | Pricing | Orders | Payments
Browser: PDP (static+dynamic price/stock islands) -> Cart drawer (optimistic) -> Checkout machine -> Payment iframe
```
```tsx
const addToCart = useMutation({
  mutationFn: (v: {sku: string; qty: number}) => api.post('/cart/items', v, { headers: { 'Idempotency-Key': crypto.randomUUID() } }),
  onMutate: v => qc.setQueryData(['cart'], c => addLocal(c, v)),
  onError: (e, v) => { rollback(); toast(e.code === 'OUT_OF_STOCK' ? 'Sold out' : 'Try again'); },
  onSettled: () => qc.invalidateQueries({ queryKey: ['cart'] }),
});
```
Checkout state machine (XState or reducer) prevents illegal transitions; order submit disables button and uses idempotency key to avoid double charge; price shown is always re-confirmed by the server.

**Tradeoffs**
- ISR gives speed but prices/stock can be stale -> fetch volatile data client-side.
- Micro-frontends per domain (cart, checkout) give team autonomy but cost bundle size/consistency.
- BFF adds a layer to maintain but shields the UI from service sprawl.

**How to present this in an interview**
Structure by page type and data volatility. Emphasize correctness points interviewers love: server-authoritative cart/price, idempotent order, inventory re-check, accessibility, and graceful degradation.

---

<a id="q206"></a>
### Q206. How would you implement search with filters, facets and deep-linkable URLs?

**Context / clarifying questions**
- Backend engine (Elasticsearch/Algolia/SQL)? Do facet counts need to update with selected filters?
- SEO needed for result pages? Expected result volume?

**Diagnosis or approach (step by step)**
1. URL is the source of truth: `/search?q=shoe&brand=nike&brand=adidas&price=50-100&sort=price_asc&page=2`.
2. Parse URL -> typed filter object (validate with zod); unknown/invalid values fall back to defaults.
3. Query key = filter object; fetch results + facet counts in one call; `placeholderData: keepPreviousData` to avoid flicker.
4. Debounce text input (state local, URL updated after ~300ms); use `replace` for typing, `push` for discrete filter changes.
5. Reset `page` when filters change; canonicalize param order for cache/SEO.
6. SEO: SSR the result page for indexable combos, canonical + noindex for infinite combos.

**Solution with code or architecture diagram**
```tsx
const FiltersSchema = z.object({
  q: z.string().default(''),
  brand: z.array(z.string()).default([]),
  sort: z.enum(['relevance', 'price_asc', 'price_desc']).default('relevance'),
  page: z.coerce.number().int().min(1).default(1),
});

function useFilters() {
  const [sp, setSp] = useSearchParams();
  const filters = useMemo(() => FiltersSchema.parse({
    q: sp.get('q') ?? undefined, brand: sp.getAll('brand'),
    sort: sp.get('sort') ?? undefined, page: sp.get('page') ?? undefined,
  }), [sp]);
  const update = (patch: Partial<typeof filters>, replace = false) => {
    const next = { ...filters, page: 1, ...patch };
    const p = new URLSearchParams();
    if (next.q) p.set('q', next.q);
    next.brand.forEach(b => p.append('brand', b));
    p.set('sort', next.sort); p.set('page', String(next.page));
    setSp(p, { replace });
  };
  return [filters, update] as const;
}

const { data } = useQuery({ queryKey: ['search', filters], queryFn: () => api.search(filters), placeholderData: keepPreviousData });
```
Facets render from `data.facets` (counts per value), disabled when count is 0; "Clear all" resets params; abort in-flight requests via query cancellation.

**Tradeoffs**
- URL state is shareable/back-button friendly but long URLs and re-parsing cost; keep ephemeral UI state (open panels) out of the URL.
- Server-computed facets are accurate but add backend load.
- Debounce delays feedback; instant search needs a fast engine.

**How to present this in an interview**
Say "URL is the single source of truth" early, show schema validation, back-button behavior, debounce, and stale-response handling.

---

<a id="q207"></a>
### Q207. How do you handle a flaky or slow network (retries, offline queue)?

**Context / clarifying questions**
- Target users (mobile/field)? Which operations must work offline (read-only vs writes)?
- Are writes idempotent? Is conflict resolution needed?

**Diagnosis or approach (step by step)**
1. Classify errors: network failure/timeouts/5xx/429 (retry) vs 4xx (don't retry).
2. Retry with exponential backoff + jitter, cap attempts, honor `Retry-After`; timeouts via `AbortController`.
3. Make writes idempotent (Idempotency-Key) so retries are safe.
4. Offline: detect with `navigator.onLine` + failed request signals (not fully reliable); queue mutations in IndexedDB; replay in order on reconnect; show "pending/synced/failed" status.
5. Reads: cache (service worker / persisted query cache), stale-while-revalidate, skeletons and cached data instead of spinners.
6. UX: offline banner, per-item sync state, user-visible retry for terminal failures.

**Solution with code or architecture diagram**
```
UI -> mutation -> [online? send : enqueue in IndexedDB] -> retry/backoff -> server (idempotent)
Service Worker: cache static assets + API reads (SWR)  |  Background Sync replays queue
```
```ts
async function fetchRetry(url: string, init: RequestInit, max = 4) {
  for (let i = 0; ; i++) {
    const ctrl = new AbortController(); const t = setTimeout(() => ctrl.abort(), 10_000);
    try {
      const res = await fetch(url, { ...init, signal: ctrl.signal });
      if (res.ok || (res.status < 500 && res.status !== 429)) return res;
      if (i >= max) return res;
    } catch (e) { if (i >= max) throw e; }
    finally { clearTimeout(t); }
    await new Promise(r => setTimeout(r, Math.min(30_000, 2 ** i * 500) * (0.5 + Math.random())));
  }
}
```
With TanStack Query: `retry`, `retryDelay`, `networkMode: 'offlineFirst'`, `persistQueryClient` with IndexedDB, and `mutationCache` resume via `resumePausedMutations`.

**Tradeoffs**
- Retries amplify load during outages (retry storms) -> jitter, caps, circuit breaker.
- Offline writes introduce conflicts and complex UX; only support where the business needs it.
- Persisted caches risk storing sensitive data -> encrypt/clear on logout.

**How to present this in an interview**
Stress idempotency + jitter + which errors to retry, then the user-facing states (pending/failed). Mention you'd only build offline queues for flows that justify it.

---

<a id="q208"></a>
### Q208. How do you build a multi-tenant, white-label UI?

**Context / clarifying questions**
- Tenant identification: subdomain, path, or login? How many tenants (10 vs 10,000)?
- What varies: theme only, copy/locale, feature set, layout, or custom domains?
- Single deployment or per-tenant builds?

**Diagnosis or approach (step by step)**
1. Resolve tenant at the edge/server (host header -> tenant ID) and fetch a tenant config document.
2. Separate concerns: design tokens (colors, fonts, radius, logo), feature entitlements, copy/i18n overrides, and optional component overrides.
3. Theming via CSS variables so a theme switch needs no rebuild or re-render of the tree.
4. One build, runtime config. Per-tenant builds do not scale.
5. Isolation: tenant ID on every API call (server enforces authorization), cache keys include tenant, no cross-tenant data in shared caches/localStorage keys.
6. Test with a tenant matrix and visual regression for a few representative themes; validate contrast of tenant colors.

**Solution with code or architecture diagram**
```
Request(host) -> Edge middleware -> tenantId -> Config API (cached) -> SSR injects <style>:root{--brand:...} + TenantProvider
```
```tsx
const TenantContext = createContext<TenantConfig>(null!);

export function TenantProvider({ config, children }: { config: TenantConfig; children: React.ReactNode }) {
  const style = Object.fromEntries(Object.entries(config.tokens).map(([k, v]) => [`--${k}`, v]));
  return (
    <TenantContext.Provider value={config}>
      <div style={style as React.CSSProperties} data-tenant={config.id}>{children}</div>
    </TenantContext.Provider>
  );
}
// CSS: .btn { background: var(--brand-primary); }
const useFeature = (f: string) => useContext(TenantContext).features.includes(f);
```
Custom domains map to tenant IDs in the edge layer; logos/fonts served from tenant-scoped CDN paths; query keys `[tenantId, ...]`.

**Tradeoffs**
- Runtime config is flexible but needs validation and a safe fallback theme.
- Deep per-tenant component overrides explode maintenance; prefer tokens + slots + feature flags.
- Shared deployment risks blast radius; mitigate with per-tenant flags and canaries.

**How to present this in an interview**
Say "one codebase, one build, config-driven". Cover tokens via CSS variables, tenant isolation (security), and how you stop customization from fragmenting the product.

---

<a id="q209"></a>
### Q209. How do you migrate a legacy class or jQuery app incrementally?

**Context / clarifying questions**
- Size of app, test coverage, release cadence, can pages be served separately?
- Is there a router/state owned by the legacy app? Business deadline vs refactor budget?

**Diagnosis or approach (step by step)**
1. Don't rewrite; use the strangler fig pattern: new features and touched screens in React, legacy remains until replaced.
2. Establish a safety net: characterization/e2e tests around critical flows first.
3. Set up the build so React and legacy coexist (bundler, TS allowed with `allowJs`).
4. Choose seams: page-by-page behind the router/proxy, or widget-by-widget mounted into legacy DOM via `createRoot`.
5. Share state through a thin adapter (events, a tiny store, or URL) rather than reaching into DOM.
6. Class -> function components opportunistically when touching code (classes still work; error boundaries still need classes); no big-bang conversion.
7. Track progress (% of routes migrated), delete dead code, enforce lint rules to stop new legacy code.

**Solution with code or architecture diagram**
```
Reverse proxy / router:  /orders/* -> React app    /legacy/* -> jQuery app   (shared auth cookie, shared design tokens)
Within a page: legacy DOM #widget-root -> mounts <Widget/> via createRoot
```
```tsx
// Mount a React widget inside a legacy page
export function mountWidget(el: HTMLElement, props: WidgetProps) {
  const root = createRoot(el);
  root.render(<Widget {...props} />);
  return () => root.unmount();          // call when the legacy view is torn down
}
// Legacy: window.App.mountWidget($('#widget-root')[0], { orderId: 42 });
// Bridge: legacy $(document).trigger('cart:changed') <-> React useEffect listener
```
Convert a class lifecycle to hooks: `componentDidMount/Unmount` -> `useEffect` with cleanup; `setState` -> `useState`/`useReducer`.

**Tradeoffs**
- Two stacks increase bundle size and inconsistency in the interim; set an end date.
- Bridges (events/global) are hacky but cheap; keep them narrow and documented.
- Rewrite is cleaner on paper, but risky and delays value.

**How to present this in an interview**
Use the words "strangler fig", tests first, vertical slices, a measurable migration dashboard, and deleting legacy as you go.

---

<a id="q210"></a>
### Q210. How do you migrate CRA to Vite or Next.js with zero downtime?

**Context / clarifying questions**
- Is SEO/SSR a goal (Next.js) or just faster builds/DX (Vite)? Env var, proxy, Jest, SVG/aliases usage?
- Hosting/CDN setup, CI/CD, and how routing is done (React Router)?

**Diagnosis or approach (step by step)**
1. Pick target: Vite for a drop-in SPA with minimal change; Next.js if you need SSR/RSC/SEO, accepting routing/data-fetching changes. CRA is deprecated.
2. Do it on a branch with the old app still deploying. Zero downtime = the migration is just another release behind a switch.
3. Vite steps: add `vite` + `@vitejs/plugin-react`, move `index.html` to root and add `<script type="module" src="/src/index.tsx">`, rename `REACT_APP_*` -> `VITE_*` (`import.meta.env`), replace `process.env`, configure aliases/proxy, swap Jest for Vitest (or keep Jest), fix SVG imports (`vite-plugin-svgr`), rename JSX files to `.jsx/.tsx`.
4. Next.js steps: App/Pages router incrementally; move routes one by one (`next.config` rewrites / `basePath` or serve CRA catch-all as fallback); `'use client'` for existing components; replace `window` use at render; move env to `NEXT_PUBLIC_*`.
5. Validate parity: same e2e suite, bundle-size diff, Web Vitals comparison, source maps to error tracker.
6. Cutover: deploy to a parallel environment, shadow/canary with a percent of traffic at the CDN/LB, monitor errors/vitals, ramp to 100%, keep instant rollback (route back to old origin). Keep hashed asset filenames and long-cache headers so in-flight users on old HTML still load old chunks.

**Solution with code or architecture diagram**
```
CDN/LB --(95%)--> old CRA origin
       --( 5%)--> new Vite/Next origin  -> ramp 5->25->100 with error-rate gate; rollback = set weight 0
```
```ts
// vite.config.ts
export default defineConfig({
  plugins: [react(), svgr()],
  resolve: { alias: { '@': '/src' } },
  server: { proxy: { '/api': 'http://localhost:8080' } },
  build: { sourcemap: true },
});
```
Next.js incremental option: Next app serves migrated routes; unmatched paths are `rewrites()` fallback to the legacy CRA host.

**Tradeoffs**
- Vite: low effort, no SSR by default; Next: bigger gain (SSR, RSC) and bigger change.
- Parallel run costs double infra briefly but gives safe rollback.
- Test runner switch can change mocking behavior; migrate tests deliberately.

**How to present this in an interview**
Frame as release engineering: parallel origin, weighted traffic, metrics gates, rollback. Mention concrete gotchas (env prefix, `index.html`, SVG, Jest).

---

<a id="q211"></a>
### Q211. How do you handle a 2 MB bundle that hurts conversions?

**Context / clarifying questions**
- 2 MB compressed or raw? Which pages/devices/networks matter? What are current LCP/INP/TTI numbers and the conversion link?

**Diagnosis or approach (step by step)**
1. Analyze: `vite-bundle-visualizer` / `source-map-explorer` / webpack-bundle-analyzer. Find top offenders (moment, lodash full, charting, icon packs, duplicate versions).
2. Set budgets (e.g., <170 KB gz initial JS for critical route) and enforce in CI.
3. Fixes by payoff: route-based code splitting (`React.lazy`), lazy-load below-the-fold/heavy widgets (charts, editors, maps), replace heavy libs (date-fns/dayjs, lodash-es per-method), tree-shaking (ESM, `sideEffects:false`), dedupe deps, drop polyfills with modern browserslist, compression (Brotli), remove unused code/feature flags.
4. Loading strategy: preload critical chunks, prefetch likely next routes on hover/idle, SSR/RSC to ship less client JS, defer third-party scripts (analytics, chat) with `async`/after interaction.
5. Verify in RUM: LCP, INP, conversion before/after.

**Solution with code or architecture diagram**
```tsx
const Reports = lazy(() => import('./pages/Reports'));
const Chart = lazy(() => import(/* webpackChunkName: "chart" */ './Chart'));

<Suspense fallback={<Skeleton />}>
  <Routes><Route path="/reports" element={<Reports />} /></Routes>
</Suspense>

// prefetch on intent
<Link to="/reports" onMouseEnter={() => import('./pages/Reports')}>Reports</Link>
```
```ts
// vite.config.ts - budget guard
build: { chunkSizeWarningLimit: 250, rollupOptions: { output: { manualChunks: { vendor: ['react', 'react-dom'] } } } }
```
CI: `size-limit` fails PR if initial bundle exceeds budget.

**Tradeoffs**
- More chunks = more requests and possible waterfalls; mitigate with preload/prefetch and HTTP/2.
- Lazy loading shows loading states; use good skeletons.
- Replacing libraries costs engineering time; prioritize by KB saved per effort.

**How to present this in an interview**
Tie to business: "x00 ms LCP ~ y% conversion". Show analyze -> budget -> split -> enforce loop, with real KB numbers.

---

<a id="q212"></a>
### Q212. How would you build a role-based admin portal?

**Context / clarifying questions**
- Fixed roles or custom roles/permissions? Row/field-level rules? Multi-tenant? Audit requirements?
- Where is identity (OIDC/Entra/Auth0)? Token claims or permission API?

**Diagnosis or approach (step by step)**
1. Model permissions (not roles) in the UI: `orders:read`, `orders:refund`. Roles map to permission sets on the backend.
2. Server is the authority: every API enforces authorization; UI gating is only UX.
3. Load the user's permissions after login (token claim or `/me`), hold in context.
4. Gate three levels: routes (guards/loader redirects), navigation items, and individual actions/fields (`<Can>`).
5. Handle 401 (re-auth) vs 403 (show "no access") consistently via an HTTP interceptor.
6. Audit-sensitive actions (confirmation dialogs, reason fields), log to backend.
7. Test with a role matrix; code-split admin modules per permission so unauthorized users don't download them.

**Solution with code or architecture diagram**
```
Login (OIDC) -> /me {permissions[]} -> PermissionProvider -> Route guard -> Nav filter -> <Can> actions
API: JWT -> policy check per endpoint (source of truth)
```
```tsx
const PermCtx = createContext<Set<string>>(new Set());
export const useCan = (p: string) => useContext(PermCtx).has(p);
export const Can = ({ perm, children, fallback = null }: { perm: string; children: React.ReactNode; fallback?: React.ReactNode }) =>
  useCan(perm) ? <>{children}</> : <>{fallback}</>;

const routes = [
  { path: '/refunds', perm: 'orders:refund', element: lazy(() => import('./Refunds')) },
];
function Guard({ perm, children }: { perm: string; children: React.ReactNode }) {
  return useCan(perm) ? <>{children}</> : <Navigate to="/403" replace />;
}
// <Can perm="orders:refund"><Button>Refund</Button></Can>
```

**Tradeoffs**
- Permission-based UI is more granular but needs a maintained catalog; role-checks (`isAdmin`) are simple but brittle.
- Hiding vs disabling controls: hide when users should not know, disable with tooltip when discoverability helps.
- Stale permissions after role change: refresh on focus or short token TTL.

**How to present this in an interview**
Open with "UI hiding is not security". Then the permission model, guards at three levels, and consistent 401/403 handling.

---

<a id="q213"></a>
### Q213. How do you handle concurrent edits (conflict resolution, locking)?

**Context / clarifying questions**
- Document-style collaborative editing or record-style forms? How frequent are collisions? Is offline editing involved?
- Is losing a user's change acceptable?

**Diagnosis or approach (step by step)**
1. Pick strategy by collision likelihood:
   - Optimistic concurrency (default): entity carries `version`/ETag; save sends `If-Match`; server returns 409/412 on mismatch.
   - Pessimistic locking: lock with TTL + heartbeat for rare, high-cost edits (e.g., approval workflows).
   - Real-time merge: OT/CRDT (Yjs, Automerge) for collaborative text/canvases.
2. On conflict: fetch latest, show a diff/merge UI (field-level auto-merge when non-overlapping), let the user choose.
3. Presence indicators ("Priya is editing") reduce collisions via WebSocket.
4. Avoid silent last-write-wins on important data.

**Solution with code or architecture diagram**
```
Client A, B load v5 -> A saves (If-Match: 5) -> v6 -> B saves (If-Match: 5) -> 412 + latest v6 -> merge UI -> B saves (If-Match: 6)
```
```tsx
const save = useMutation({
  mutationFn: (draft: Doc) => api.put(`/docs/${draft.id}`, draft, { headers: { 'If-Match': String(draft.version) } }),
  onError: async (err: any, draft) => {
    if (err.status === 412) {
      const latest = await api.get<Doc>(`/docs/${draft.id}`);
      setConflict({ mine: draft, theirs: latest, base });   // open merge dialog
    }
  },
});
// Field-level 3-way merge: take theirs where mine == base; take mine where theirs == base; else ask user
```
Locks: `POST /docs/:id/lock` returns token + expiry; client heartbeats every 30s; server releases on expiry.

**Tradeoffs**
- Optimistic: scalable, needs good conflict UX. Locks: simple mental model, but stale locks and blocked users.
- CRDTs: seamless but heavier, storage growth, harder to apply business rules.

**How to present this in an interview**
Show the decision tree by collision rate and the version/412 flow; mention three-way merge and presence as UX mitigations.

---

<a id="q214"></a>
### Q214. How do you design dynamic, schema-driven forms?

**Context / clarifying questions**
- Who authors schemas (devs, admins, backend)? Complexity: conditional fields, repeating groups, cross-field validation, computed values?
- Need versioning and server-side validation parity?

**Diagnosis or approach (step by step)**
1. Define a schema (JSON Schema or a custom typed DSL): field type, label, validation, visibility conditions, options source.
2. Build a component registry mapping `type -> React component`; the renderer is generic.
3. Use a form library (React Hook Form) for state/perf; build validation from the schema (Ajv/zod generated at runtime).
4. Conditions (`showIf`) evaluated from watched values using a safe expression format (JSON logic), never `eval`.
5. Version schemas; store submissions with schema version; validate again on the server with the same schema.
6. Accessibility: labels, error association (`aria-describedby`), focus first error.
7. Consider whether a bespoke renderer is worth it versus existing (RJSF, SurveyJS, Formik+Yup, uniforms).

**Solution with code or architecture diagram**
```
Schema API -> validate schema -> Renderer -> registry[type] -> RHF controls -> submit -> server validates same schema
```
```tsx
type Field = { name: string; type: 'text' | 'select' | 'checkbox'; label: string; required?: boolean;
               options?: string[]; showIf?: { field: string; equals: unknown } };

const registry: Record<Field['type'], React.FC<{ field: Field }>> = { text: TextInput, select: SelectInput, checkbox: CheckboxInput };

function DynamicForm({ schema, onSubmit }: { schema: Field[]; onSubmit: (v: any) => void }) {
  const methods = useForm({ resolver: zodResolver(buildZod(schema)) });
  const values = methods.watch();
  return (
    <FormProvider {...methods}>
      <form onSubmit={methods.handleSubmit(onSubmit)}>
        {schema.filter(f => !f.showIf || values[f.showIf.field] === f.showIf.equals).map(f => {
          const C = registry[f.type];
          return <C key={f.name} field={f} />;
        })}
        <button type="submit">Submit</button>
      </form>
    </FormProvider>
  );
}
```
Hidden fields should be unregistered (`shouldUnregister`) so stale values are not submitted.

**Tradeoffs**
- Generic renderers speed delivery of many forms but make bespoke UX hard (escape hatch: custom widget types).
- Runtime-built validation is powerful, yet harder to type-check.
- Off-the-shelf libraries reduce build time but constrain styling.

**How to present this in an interview**
Describe schema + registry + condition engine, then call out server-side validation, versioning, and a11y.

---

<a id="q215"></a>
### Q215. How do you handle large file uploads (chunking, resume)?

**Context / clarifying questions**
- Max file size, file types, storage (S3/Azure Blob), security scanning requirement, mobile users?

**Diagnosis or approach (step by step)**
1. Avoid proxying bytes through your API: upload directly to object storage with pre-signed URLs/SAS tokens.
2. Chunk with `File.slice()` (e.g., 5-10 MB parts); S3 multipart / Azure block blobs / tus protocol.
3. Upload parts with limited concurrency (3-4), per-part retry with backoff, per-part checksum.
4. Resume: persist upload ID + completed part numbers (localStorage/IndexedDB or ask server via list-parts) and skip completed parts.
5. UX: progress bar (XHR `upload.onprogress` or per-part sums), pause/cancel (`AbortController`), drag and drop, validation of type/size client-side (and server-side).
6. After completion: server finalizes, virus scan, then marks file available; UI shows processing state.

**Solution with code or architecture diagram**
```
Client -> API: init (name,size) -> uploadId + presigned URLs per part
Client -> Storage: PUT part N (parallel, retry) -> ETag
Client -> API: complete(uploadId, parts[ETag]) -> scan -> available
```
```ts
async function upload(file: File, init: { uploadId: string; urls: string[] }, done = new Set<number>(), signal?: AbortSignal) {
  const size = 8 * 1024 * 1024; const parts = Math.ceil(file.size / size);
  const etags: Record<number, string> = {};
  const queue = [...Array(parts).keys()].filter(i => !done.has(i));
  const worker = async () => {
    for (let i = queue.shift(); i !== undefined; i = queue.shift()) {
      const res = await fetchRetry(init.urls[i], { method: 'PUT', body: file.slice(i * size, (i + 1) * size), signal }, 4);
      etags[i] = res.headers.get('ETag')!; done.add(i); saveProgress(init.uploadId, done);
    }
  };
  await Promise.all([worker(), worker(), worker()]);
  return etags;
}
```

**Tradeoffs**
- Direct-to-storage offloads servers but needs careful token scoping/expiry and CORS.
- Smaller chunks give finer resume but more requests; larger give throughput.
- Libraries (Uppy, tus-js-client) save time vs custom code.

**How to present this in an interview**
Mention presigned direct upload, multipart + resume bookkeeping, concurrency limits, and server-side validation/scanning.

---

<a id="q216"></a>
### Q216. How do you build a charting/analytics dashboard performantly?

**Context / clarifying questions**
- Number of charts and data points per chart? Live-updating or static? Interactivity (zoom, brush, cross-filter)? Export needs?

**Diagnosis or approach (step by step)**
1. Pick renderer by volume: SVG (Recharts/Visx/D3) up to a few thousand points; Canvas (ECharts, Chart.js, uPlot) for tens of thousands+; WebGL for millions.
2. Don't ship raw data: aggregate/downsample server-side (time buckets, LTTB) according to zoom level; fetch more detail on zoom.
3. Fetch panels in parallel with independent loading/error states; lazy-load charts when scrolled into view (IntersectionObserver) and code-split the chart lib.
4. Memoize transformed series; stable props to avoid chart re-instantiation; throttle live updates (batch via `requestAnimationFrame`).
5. Heavy transforms in a Web Worker.
6. Shared filter state (date range, segments) in URL; each panel keyed by filters; cancel stale requests.
7. Accessibility: data table alternative, color-blind-safe palette, text summaries.

**Solution with code or architecture diagram**
```
Filters (URL) -> panels (parallel queries) -> API (pre-aggregated, cached) -> chart (canvas) 
Live: WS -> buffer -> rAF flush (<=1 update/frame)
```
```tsx
const LazyChart = lazy(() => import('./TimeSeriesChart'));
function Panel({ metric, filters }: Props) {
  const { ref, inView } = useInView({ triggerOnce: true });
  const { data, isLoading } = useQuery({ queryKey: ['metric', metric, filters], queryFn: () => api.metric(metric, filters), enabled: inView, staleTime: 60_000 });
  return <div ref={ref} style={{ minHeight: 280 }}>{isLoading ? <Skeleton /> : <Suspense fallback={<Skeleton />}><LazyChart data={data!} /></Suspense>}</div>;
}
```

**Tradeoffs**
- Canvas is fast but less accessible/styleable than SVG.
- Downsampling can hide spikes; use min/max-preserving algorithms.
- Pre-aggregation needs backend investment but is the biggest win.

**How to present this in an interview**
Lead with "reduce data before drawing it", then renderer choice, lazy panels, and throttled live updates.

---

<a id="q217"></a>
### Q217. How do you support multiple teams in one front-end codebase?

**Context / clarifying questions**
- How many teams/engineers? Independent deploy needed? Shared design system? Current pain: merge conflicts, slow CI, inconsistent UI?

**Diagnosis or approach (step by step)**
1. Start with a modular monorepo (Nx/Turborepo + pnpm workspaces): `apps/*`, `packages/ui`, `packages/utils`, `packages/api-client`, `features/*`.
2. Ownership: CODEOWNERS per directory; teams own feature packages; shared packages have a platform/design-system owner and a contribution process.
3. Boundaries: enforce with lint (`@nx/enforce-module-boundaries`, eslint-plugin-boundaries) — features may import shared, not each other's internals; public API via `index.ts`.
4. CI speed: affected-only builds/tests, remote caching, parallelism.
5. Consistency: shared ESLint/TS/prettier configs, design tokens and component library with Storybook, conventional commits, changesets for versioning.
6. Only if independent deploys are truly needed: micro-frontends (Module Federation) with shared contracts, and a shared shell. Otherwise keep a single deployable.
7. Docs/ADRs, trunk-based dev with feature flags to avoid long-lived branches.

**Solution with code or architecture diagram**
```
repo/
  apps/web  apps/admin
  packages/ui  packages/design-tokens  packages/api-client  packages/config
  features/checkout  features/search  features/account   (each owned by a team)
CODEOWNERS: /features/checkout/ @team-checkout
```
```json
// .eslintrc - module boundary rule
"@nx/enforce-module-boundaries": ["error", { "depConstraints": [
  { "sourceTag": "type:feature", "onlyDependOnLibsWithTags": ["type:ui", "type:util", "type:data"] },
  { "sourceTag": "type:ui", "onlyDependOnLibsWithTags": ["type:ui", "type:util"] } ] }]
```

**Tradeoffs**
- Monorepo: easy refactors and sharing, needs tooling investment. Polyrepo/micro-frontends: autonomy, but version drift, duplicated deps, UX inconsistency.
- Strict boundaries slow quick hacks but prevent entanglement.

**How to present this in an interview**
Say "optimize for team autonomy without sacrificing consistency". Cover ownership, enforced boundaries, CI scaling, and when (not) to go micro-frontend.

---

<a id="q218"></a>
### Q218. How do you reduce production errors after a release?

**Context / clarifying questions**
- Which errors (JS exceptions, API failures, UX regressions)? Do we have error tracking, source maps, release tagging, RUM? What does the error budget look like?

**Diagnosis or approach (step by step)**
1. Make errors visible: Sentry/App Insights with source maps uploaded per release, release + user/session context, breadcrumbs, grouping.
2. Triage by impact (users affected, frequency, new in this release) — fix top regressions first; link to the deploy that introduced them.
3. Contain: error boundaries per feature (not one global), fallback UIs, retry affordances.
4. Prevent: TypeScript strict, runtime validation at API boundaries (zod), lint rules (`react-hooks/exhaustive-deps`), unit/integration tests (RTL + MSW), e2e smoke on critical journeys, contract tests with backend.
5. Release safety: feature flags, canary/staged rollout with automatic rollback on error-rate threshold, post-deploy smoke test.
6. Handle chunk-load errors after deploys (stale HTML requesting deleted chunks): keep old assets, retry/reload once.
7. Blameless postmortems feeding new tests/guards.

**Solution with code or architecture diagram**
```
Release -> staged rollout (5%) -> monitor error rate/vitals -> gate -> 100% | auto-rollback
```
```tsx
Sentry.init({ dsn, release: process.env.APP_VERSION, environment: 'prod', tracesSampleRate: 0.1,
  beforeSend: e => (isNoise(e) ? null : e) });

<ErrorBoundary fallbackRender={({ resetErrorBoundary }) => <Retry onClick={resetErrorBoundary} />}
               onError={(err, info) => Sentry.captureException(err, { extra: info })}>
  <CheckoutWidget />
</ErrorBoundary>

window.addEventListener('vite:preloadError', () => location.reload());  // stale chunk after deploy
```

**Tradeoffs**
- More tests/gates slow releases; invest in the highest-risk journeys.
- Noisy alerts cause fatigue; alert on rates/new issues, not every event.
- Sampling controls cost but may hide rare errors.

**How to present this in an interview**
Frame as detect -> contain -> prevent -> release safely, using metrics (crash-free sessions %, error rate per release).

---

<a id="q219"></a>
### Q219. How do you handle payment and PCI concerns in the UI?

**Context / clarifying questions**
- Provider (Stripe, Adyen, Braintree)? Cards only or wallets/3DS? Which PCI SAQ level is targeted? Regions (SCA/PSD2)?

**Diagnosis or approach (step by step)**
1. Principle: card data must never touch your servers or JS state. Use provider-hosted fields/iframes (Stripe Elements, Adyen Drop-in) or redirect/hosted checkout, which keeps you at SAQ A (lightest).
2. Client obtains a token/PaymentMethod from the provider; sends only that token + order ID to your backend; backend creates the charge using secret keys.
3. Never log, persist, or send card data to analytics/session replay (mask the payment iframe region, block inputs in replay tools).
4. Security headers: strict CSP (limit script sources), SRI for third-party scripts, HTTPS + HSTS; PCI DSS 4.0 requires managing/monitoring payment-page scripts (req. 6.4.3, 11.6.1) — inventory and authorize every script on checkout.
5. Idempotency keys on payment confirmation; handle 3DS/SCA redirect flows and return states; rely on webhooks (not the client) as truth for payment success.
6. UX: clear error messages, double-submit prevention, accessible forms, wallet options (Apple/Google Pay).

**Solution with code or architecture diagram**
```
Browser (provider iframe holds PAN) -> provider -> token
Browser -> Your API (token, orderId, idempotency key) -> provider (secret key) -> webhook -> order paid
```
```tsx
function PayForm({ clientSecret }: { clientSecret: string }) {
  const stripe = useStripe(); const elements = useElements();
  const [busy, setBusy] = useState(false);
  const pay = async (e: React.FormEvent) => {
    e.preventDefault(); if (!stripe || !elements || busy) return; setBusy(true);
    const { error } = await stripe.confirmPayment({ elements, confirmParams: { return_url: `${location.origin}/order/complete` } });
    if (error) { showError(error.message); setBusy(false); }
  };
  return <form onSubmit={pay}><PaymentElement /><button disabled={busy}>Pay</button></form>;
}
```
(Verify details against the provider's current docs.)

**Tradeoffs**
- Hosted fields: minimal PCI scope, less styling control. Custom card inputs: full control, but huge compliance burden — avoid.
- Redirect checkout is safest but leaves your site; embedded elements balance both.
- Third-party scripts on checkout add risk; minimize them.

**How to present this in an interview**
Key sentence: "We never touch card numbers; we use provider-hosted fields, tokens, CSP, and webhooks for truth." Note you are not a compliance officer, but know the scope-reduction strategy.

---

<a id="q220"></a>
### Q220. How would you design an autosave feature?

**Context / clarifying questions**
- Drafts or live-published content? Frequency of edits, document size, multi-device/collaboration? Is offline needed?

**Diagnosis or approach (step by step)**
1. Trigger: debounce after the last change (1-2 s) plus a max-wait (e.g., 10 s) and flush on blur/`visibilitychange`/route change.
2. Send only when dirty; send diffs/patches for large docs, or full draft for small.
3. Serialize saves: one in flight at a time; if changes occur meanwhile, save again after (avoid out-of-order overwrites). Use version numbers.
4. States: idle -> dirty -> saving -> saved / error; show "Saving… / Saved 10:42 / Retrying". Warn on `beforeunload` if dirty; use `fetch keepalive`/`sendBeacon` on exit.
5. Resilience: retry with backoff; local backup to localStorage/IndexedDB; restore prompt on reload.
6. Conflicts: version/ETag (see Q213).
7. Keep autosave drafts separate from "publish" action; validate only on publish.

**Solution with code or architecture diagram**
```
edit -> dirty -> debounce(1.5s, maxWait 10s) -> save queue (serial) -> PATCH /draft (If-Match) -> saved | retry
          \-> local backup (IndexedDB)
```
```tsx
function useAutosave<T>(value: T, save: (v: T, version: number) => Promise<number>, delay = 1500) {
  const [status, setStatus] = useState<'idle' | 'saving' | 'saved' | 'error'>('idle');
  const latest = useRef(value); latest.current = value;
  const version = useRef(0); const inFlight = useRef(false); const lastSaved = useRef(value);

  const flush = useCallback(async () => {
    if (inFlight.current || Object.is(latest.current, lastSaved.current)) return;
    inFlight.current = true; setStatus('saving');
    const snapshot = latest.current;
    try { version.current = await save(snapshot, version.current); lastSaved.current = snapshot; setStatus('saved'); }
    catch { setStatus('error'); }
    finally { inFlight.current = false; if (!Object.is(latest.current, lastSaved.current)) flush(); }
  }, [save]);

  useEffect(() => { const t = setTimeout(flush, delay); return () => clearTimeout(t); }, [value, delay, flush]);
  useEffect(() => {
    const onHide = () => document.visibilityState === 'hidden' && flush();
    document.addEventListener('visibilitychange', onHide);
    return () => document.removeEventListener('visibilitychange', onHide);
  }, [flush]);
  return status;
}
```

**Tradeoffs**
- Short debounce = more requests; long = more data lost on crash.
- Patches reduce bandwidth but complicate server logic.
- Autosave of invalid data requires lenient draft validation.

**How to present this in an interview**
Highlight the serialized save queue, flush on hide/unload, visible status, and conflict handling — the pitfalls most candidates miss.

---

<a id="q221"></a>
### Q221. How do you roll out a risky feature safely (flags, canary)?

**Context / clarifying questions**
- What's the blast radius (payments vs cosmetic)? Is there a data migration or API dependency? What metrics define "healthy"? Can we roll back instantly?

**Diagnosis or approach (step by step)**
1. Decouple deploy from release: merge behind a feature flag (LaunchDarkly, Unleash, GrowthBook, Azure App Config), default off.
2. Progressive exposure: internal staff -> beta customers -> 1% -> 5% -> 25% -> 50% -> 100%, with sticky bucketing (hash of user ID) so users get consistent experience.
3. Define guardrail metrics up front: error rate, p95 latency, Web Vitals, conversion/checkout success; set automatic halt thresholds. Compare cohort vs control.
4. Dark launch/shadow traffic for backend-heavy changes; keep old and new paths side by side.
5. Backward-compatible API/data changes first (expand/contract), so rollback is safe.
6. Kill switch: flag flip reverts without a deploy; test the off path in CI.
7. Clean up: remove flag and dead code after full rollout (flag debt).
8. Canary at infrastructure level (weighted routing) for the build itself in addition to flag-level exposure.

**Solution with code or architecture diagram**
```
Deploy (flag off) -> enable internal -> 1% -> 5% -> 25% -> 100%   (each step gated by metrics; kill switch anytime)
```
```tsx
function Checkout() {
  const newFlow = useFlag('checkout-v2');           // evaluated server-side or via SDK with stable user bucket
  return (
    <ErrorBoundary fallback={<CheckoutV1 />} onError={() => reportFlagError('checkout-v2')}>
      {newFlow ? <CheckoutV2 /> : <CheckoutV1 />}
    </ErrorBoundary>
  );
}
// Tag telemetry with flag variant so dashboards can compare cohorts
Sentry.setTag('checkout-v2', String(newFlow));
```

**Tradeoffs**
- Flags add branching complexity and test matrix; enforce expiry dates.
- Client-evaluated flags can flash/flicker and expose unreleased features; evaluate server-side for sensitive things.
- Slow ramp reduces risk but delays learning/value.

**How to present this in an interview**
"Deploy is not release." Give the ramp schedule, guardrail metrics with auto-rollback, backward-compatible data changes, and flag cleanup.

---

<a id="q222"></a>
### Q222. Tell me about a hard production bug you fixed in React.

**Context / clarifying questions**
This is a behavioral question: pick a real incident with measurable impact, a non-obvious root cause, and a lasting prevention outcome.

**Diagnosis or approach (step by step)**
1. Choose a story that shows: ownership, systematic debugging, collaboration, and prevention.
2. Prepare numbers: users affected, duration, metric deltas, time to detect/fix.
3. Practice a 2-minute version (STAR) and a deeper 5-minute version for follow-ups.

**Solution with code or architecture diagram — STAR model answer template**
- **Situation:** "In [product], after release [x], [N]% of [users/sessions] saw [symptom] — [business impact: failed checkouts, support tickets]."
- **Task:** "I was the [on-call / owning engineer] responsible for restoring service and finding root cause."
- **Action:**
  1. Stabilize first: "We rolled back / disabled the flag to stop the bleeding."
  2. Reproduce: "Used error tracking + session replay to find the pattern (browser, route, data shape); reproduced locally with production-like data."
  3. Isolate: "Used React Profiler / bisected the release / binary-searched commits to narrow it down."
  4. Root cause: "[e.g., a stale closure in a `useEffect` with missing deps caused duplicate API submissions] or [a context value created each render caused an infinite re-render on large lists] or [race condition: older response overwrote newer one]."
  5. Fix: "Specific, minimal change (cleanup + `AbortController`, dependency fix, memoization, keyed state)."
  6. Prevent: "Regression test, lint rule (`exhaustive-deps` as error), monitoring alert, postmortem."
- **Result:** "Error rate fell from X% to Y%, conversion recovered, MTTR was Z; the guard caught a similar bug later."

**Example talking points (sample bugs that are realistic)**
```tsx
// Race condition: slow response for old query overwrote newer results
useEffect(() => {
  const ctrl = new AbortController();
  fetch(`/api/search?q=${q}`, { signal: ctrl.signal }).then(r => r.json()).then(setResults).catch(ignoreAbort);
  return () => ctrl.abort();          // fix
}, [q]);
```
- Memory leak from un-cleaned subscription/interval causing tab slowdown after hours.
- Hydration mismatch from `Date.now()`/`window` use during SSR causing flicker/blank pages.
- Double submit of orders due to missing idempotency + button not disabled.

**Tradeoffs**
- Don't pick a story where you were the sole hero or caused it without learning; owning mistakes reads as maturity.
- Avoid too much code; keep technical depth for follow-ups.

**How to present this in an interview**
Keep to ~2 minutes: impact -> how you narrowed it -> root cause in one sentence -> fix -> prevention -> metric. Use "I" for your actions and "we" for team work. End with the lesson learned.
