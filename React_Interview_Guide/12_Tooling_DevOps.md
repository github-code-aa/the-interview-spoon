# 12. Tooling & DevOps (Q163–Q173)

## Table of Contents
- [Q163. Webpack vs Vite vs Turbopack vs esbuild](#q163-webpack-vs-vite-vs-turbopack-vs-esbuild)
- [Q164. Tree shaking and code splitting](#q164-what-are-tree-shaking-and-code-splitting)
- [Q165. ESLint, Prettier, Husky](#q165-how-do-you-set-up-eslint-prettier-and-husky)
- [Q166. CI/CD pipeline](#q166-how-do-you-design-a-cicd-pipeline-for-a-react-app)
- [Q167. Feature flags and A/B tests](#q167-how-do-you-do-feature-flags-and-ab-tests)
- [Q168. Environment and runtime config](#q168-how-do-you-do-environment-configuration-and-runtime-config)
- [Q169. Production monitoring](#q169-how-do-you-monitor-production-sentry-rum-logging)
- [Q170. Source maps, caching, CDN](#q170-how-do-you-handle-source-maps-caching-and-cdn-strategies)
- [Q171. PWAs and offline](#q171-how-do-you-build-pwas-and-offline-support-service-workers)
- [Q172. Upgrading major React versions](#q172-how-do-you-upgrade-major-react-versions-in-a-big-codebase)
- [Q173. Codemod migration](#q173-how-do-you-do-a-codemod-migration)

---

### Q163. Webpack vs Vite vs Turbopack vs esbuild.

**Short answer**
Webpack is the mature, highly configurable bundler with the largest plugin ecosystem but slower builds. Vite uses native ESM and esbuild for a near-instant dev server and Rollup (Rolldown is emerging) for production builds. esbuild is an extremely fast Go-based bundler/transpiler with a limited plugin API, and Turbopack is Vercel's Rust-based incremental bundler, used mainly through Next.js.

**Detailed explanation**
- **Webpack**: loaders/plugins, Module Federation (micro-frontends), fine-grained chunking; complex config, slower cold start and HMR on large apps.
- **Vite**: dev server serves unbundled ESM, pre-bundles deps with esbuild; fast HMR; production via Rollup with good defaults; rich plugin ecosystem (Rollup-compatible). Default choice for new SPAs.
- **esbuild**: speed (10–100x), TS/JSX transforms, minification; lacks some features (no type-checking, limited code-splitting/CSS handling controls); often used as a building block (Vite, tsup).
- **Turbopack**: function-level incremental caching; default dev bundler in Next.js, production support maturing; not a general standalone tool.
- Others: Rspack (Webpack-compatible, Rust) is the easiest migration path from Webpack.
- Choose by ecosystem constraints: Next.js → Turbopack/webpack; new SPA → Vite; legacy/federation → Webpack/Rspack; libraries → tsup/esbuild/Rollup.

**Code/config example**
```ts
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react-swc';

export default defineConfig({
  plugins: [react()],
  server: { proxy: { '/api': 'http://localhost:3000' } },
  build: {
    target: 'es2020',
    sourcemap: 'hidden',
    rollupOptions: {
      output: { manualChunks: { react: ['react', 'react-dom'] } },
    },
  },
});
```

**Common pitfalls / follow-up questions**
- *Dev/prod parity in Vite*: dev uses esbuild/native ESM, prod uses Rollup; always test the production build (`vite preview`).
- *Type-checking*: none of the fast tools type-check; run `tsc --noEmit` separately.
- *Why not migrate Webpack → Vite always?* Custom loaders, Module Federation, and effort; consider Rspack.

---

### Q164. What are tree shaking and code splitting?

**Short answer**
Tree shaking removes unused exports at build time using static ES module analysis; code splitting breaks the bundle into chunks loaded on demand (routes, heavy components) to reduce initial load. Together they cut the JavaScript users must download and parse.

**Detailed explanation**
- **Tree shaking** needs ESM (`import/export`), side-effect-free modules (`"sideEffects": false` in package.json or annotated), and production mode minification. CommonJS libs, barrel files with side effects, and `import _ from 'lodash'` defeat it—use `lodash-es` or per-method imports.
- **Code splitting**: dynamic `import()` + `React.lazy`/`Suspense`; route-based first, then component-based (charts, editors, modals). Vendor/common chunks for caching. Preload/prefetch hints for likely next routes (`/* webpackPrefetch: true */`, `<link rel="modulepreload">`).
- **Analyze**: `rollup-plugin-visualizer`, `webpack-bundle-analyzer`, `source-map-explorer`; set performance budgets in CI.
- Related: use `React.lazy` only for default exports; handle load failures with error boundary + retry.

**Code/config example**
```tsx
import { lazy, Suspense } from 'react';
import { createBrowserRouter } from 'react-router-dom';

const Dashboard = lazy(() => import('./pages/Dashboard'));
const Reports = lazy(() => import('./pages/Reports'));

export const router = createBrowserRouter([
  { path: '/', element: <Suspense fallback={<Spinner />}><Dashboard /></Suspense> },
  { path: '/reports', element: <Suspense fallback={<Spinner />}><Reports /></Suspense> },
]);

// Named export lazy
const Chart = lazy(() => import('./Chart').then(m => ({ default: m.Chart })));
```
```json
{ "name": "my-lib", "sideEffects": ["*.css"], "module": "dist/index.mjs" }
```

**Common pitfalls / follow-up questions**
- *Barrel files (`index.ts`)* can pull entire modules in dev and hurt shaking; import directly or configure `optimizePackageImports`.
- *Too many tiny chunks*: waterfall requests; group sensibly and preload.
- *Why isn't my dependency shaken?* It's CJS or has side effects; inspect with the analyzer.

---

### Q165. How do you set up ESLint, Prettier and Husky?

**Short answer**
ESLint finds code-quality and correctness issues (React hooks rules, a11y), Prettier handles formatting only, and Husky runs Git hooks so `lint-staged` checks only staged files before commit. CI runs the same checks as the authoritative gate, since hooks can be bypassed.

**Detailed explanation**
- **ESLint** (v9 flat config `eslint.config.js`): `@eslint/js`, `typescript-eslint`, `eslint-plugin-react`, `react-hooks`, `jsx-a11y`, `import`. Keep rules about bugs, not style.
- **Prettier**: opinionated formatter; use `eslint-config-prettier` to disable conflicting rules; don't run Prettier as an ESLint rule (slow).
- **Husky + lint-staged**: `pre-commit` runs `lint-staged`; optionally `commit-msg` with commitlint for Conventional Commits, and `pre-push` for tests/type-check (keep fast).
- Add `.editorconfig`, `format` and `lint` scripts, and editor-on-save integration.

**Code/config example**
```js
// eslint.config.js
import js from '@eslint/js';
import tseslint from 'typescript-eslint';
import react from 'eslint-plugin-react';
import hooks from 'eslint-plugin-react-hooks';
import a11y from 'eslint-plugin-jsx-a11y';
import prettier from 'eslint-config-prettier';

export default tseslint.config(
  { ignores: ['dist', 'coverage'] },
  js.configs.recommended,
  ...tseslint.configs.recommended,
  react.configs.flat.recommended,
  react.configs.flat['jsx-runtime'],
  a11y.flatConfigs.recommended,
  { plugins: { 'react-hooks': hooks }, rules: { ...hooks.configs.recommended.rules },
    settings: { react: { version: 'detect' } } },
  prettier,
);
```
```json
// package.json
{
  "scripts": { "prepare": "husky", "lint": "eslint .", "format": "prettier --write ." },
  "lint-staged": {
    "*.{ts,tsx,js}": ["eslint --fix", "prettier --write"],
    "*.{json,md,css}": "prettier --write"
  }
}
```
```sh
# .husky/pre-commit
npx lint-staged
```

**Common pitfalls / follow-up questions**
- *Hooks bypassed with `--no-verify`*: enforce in CI.
- *ESLint/Prettier fighting*: add `eslint-config-prettier` last.
- *Slow hooks*: lint only staged files, move heavy tasks to CI.

---

### Q166. How do you design a CI/CD pipeline for a React app?

**Short answer**
On every PR: install with a cached lockfile, then lint, type-check, unit/integration test, build, and run E2E against a preview deployment. On merge to main: build once, deploy the immutable artifact to staging, run smoke tests, promote to production with rollback ability.

**Detailed explanation**
- **Stages**: install (`npm ci`, cache) → static checks → tests (parallel/sharded) → build → security scan → preview deploy (per-PR URL) → E2E/visual/Lighthouse → deploy.
- **Build once, promote**: same artifact across environments; inject environment config at runtime (Q168).
- **Deploy strategies**: static hosting + CDN (S3/CloudFront, Azure Static Web Apps, Vercel/Netlify); atomic deploys with content-hashed assets; blue/green or canary; instant rollback by re-pointing to previous build; keep old assets for users with open tabs.
- **Quality gates**: required checks, coverage thresholds, bundle-size budgets, a11y checks; protected branches.
- **Ops**: concurrency control to cancel stale runs, secrets via OIDC (no long-lived keys), release tagging, source map upload to Sentry, post-deploy health checks and notifications.

**Code/config example**
```yaml
# .github/workflows/ci.yml
name: CI
on: { pull_request: {}, push: { branches: [main] } }
concurrency: { group: ci-${{ github.ref }}, cancel-in-progress: true }
permissions: { contents: read, id-token: write }
jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: npm }
      - run: npm ci
      - run: npm run lint && npx tsc --noEmit
      - run: npm test -- --coverage
      - run: npm run build
      - uses: actions/upload-artifact@v4
        with: { name: dist, path: dist }
  e2e:
    needs: verify
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: npm }
      - run: npm ci && npx playwright install --with-deps chromium
      - run: npx playwright test
  deploy:
    if: github.ref == 'refs/heads/main'
    needs: [verify, e2e]
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/download-artifact@v4
        with: { name: dist, path: dist }
      - run: ./scripts/deploy.sh dist   # sync to bucket, set cache headers, invalidate index.html
```

**Common pitfalls / follow-up questions**
- *Rebuilding per environment*: breaks "what was tested is what ships"; use runtime config.
- *Deleting old hashed assets immediately*: users with stale index.html hit 404s; retain for a while.
- *Speeding up?* Caching, test sharding, affected-only runs (Nx/Turborepo).

---

### Q167. How do you do feature flags and A/B tests?

**Short answer**
Feature flags decouple deploy from release by toggling code paths at runtime via a flag service (LaunchDarkly, Unleash, GrowthBook, Statsig, or homegrown). A/B tests build on flags by deterministically assigning users to variants and measuring metric differences with statistical rigor.

**Detailed explanation**
- **Flag types**: release toggles (short-lived), ops kill switches, permission/entitlement, experiments. Keep release flags temporary and remove them (flag debt).
- **React integration**: a provider fetching/streaming flags, `useFlag('new-checkout')` hook; evaluate with user context (id, plan, country). Prefer server-side/edge evaluation or bootstrapped flags to avoid UI flicker; cache defaults for failure.
- **A/B**: sticky bucketing by hashed user id, exposure logging only when the user actually sees the variant, a pre-defined primary metric, sample size and duration, avoid peeking, guard metrics; SSR/edge assignment avoids layout shift.
- Test both flag states in CI; keep the code paths small and well-isolated; flags should not alter security checks.

**Code/config example**
```tsx
import { createContext, useContext } from 'react';

type Flags = Record<string, boolean | string>;
const FlagContext = createContext<Flags>({});

export const useFlag = <T extends boolean | string>(key: string, fallback: T) =>
  (useContext(FlagContext)[key] as T) ?? fallback;

export function Checkout() {
  const variant = useFlag('checkout-experiment', 'control');
  useEffect(() => { analytics.track('exposure', { experiment: 'checkout-experiment', variant }); }, [variant]);
  return variant === 'one-page' ? <OnePageCheckout /> : <MultiStepCheckout />;
}

// Deterministic bucketing
function bucket(userId: string, exp: string, variants = ['control', 'one-page']) {
  let h = 0; for (const c of `${exp}:${userId}`) h = (h * 31 + c.charCodeAt(0)) >>> 0;
  return variants[h % variants.length];
}
```

**Common pitfalls / follow-up questions**
- *Flicker/flash of default content*: bootstrap flags before render.
- *Stale flags forever*: add expiry owners and cleanup tickets.
- *Sample ratio mismatch?* Indicates bucketing/logging bug; verify before trusting results.

---

### Q168. How do you do environment configuration and runtime config?

**Short answer**
Build-time env vars (`import.meta.env.VITE_*`) are baked into the bundle, so they force a rebuild per environment. For "build once, deploy anywhere," load a runtime config (`/config.json` or an injected `window.__CONFIG__`) at startup. Never place secrets in either.

**Detailed explanation**
- **Build-time**: `.env`, `.env.production`; fine for values that don't differ by deployment (feature defaults, public keys).
- **Runtime**: generated at container start (envsubst into `config.js`) or served by backend/edge; fetched before rendering and validated with a schema (zod). Serve with `Cache-Control: no-store`.
- Typed access through a single `config` module; fail fast on missing values; separate dev proxy configs.
- In containers: Nginx image + entrypoint script writes config from env vars.
- Next.js: `NEXT_PUBLIC_*` is build-time; server components read runtime env.

**Code/config example**
```html
<!-- index.html: loaded before the app bundle -->
<script src="/config.js"></script>
```
```sh
# docker-entrypoint.sh
envsubst < /usr/share/nginx/html/config.template.js > /usr/share/nginx/html/config.js
exec nginx -g 'daemon off;'
```
```js
// config.template.js
window.__CONFIG__ = { API_URL: "${API_URL}", SENTRY_DSN: "${SENTRY_DSN}", ENV: "${APP_ENV}" };
```
```ts
// src/config.ts
import { z } from 'zod';
const schema = z.object({ API_URL: z.string().url(), SENTRY_DSN: z.string().optional(), ENV: z.enum(['dev', 'staging', 'prod']) });
export const config = schema.parse((window as any).__CONFIG__);
```

**Common pitfalls / follow-up questions**
- *Caching `config.js`*: serve no-store, or stale config persists.
- *Assuming `.env` is private*: it ends up in the bundle.
- *Why runtime config?* Same artifact promoted across environments, faster rollbacks.

---

### Q169. How do you monitor production (Sentry, RUM, logging)?

**Short answer**
Combine error tracking (Sentry) with release/source-map linkage, Real User Monitoring for Core Web Vitals and performance, and structured logging/analytics with correlation IDs. Add alerts, dashboards and SLOs so regressions are detected before users report them.

**Detailed explanation**
- **Errors**: Sentry SDK captures unhandled exceptions, promise rejections, React error boundaries (`Sentry.ErrorBoundary`), breadcrumbs, user/session context; upload source maps and tag `release` + `environment`; scrub PII; sample and filter noise (extensions, ResizeObserver).
- **RUM**: `web-vitals` (LCP, INP, CLS) sent to analytics/Datadog/New Relic/Grafana Faro; segment by device, route, release; compare to synthetic monitoring (Lighthouse CI).
- **Logging**: structured, leveled client logs sent via beacon with sampling; propagate `traceparent`/request IDs to correlate with backend traces (OpenTelemetry).
- **Operations**: alert on error rate/new issues per release, Web Vitals regressions; session replay for debugging (mask sensitive data); consent/GDPR compliance.

**Code/config example**
```tsx
import * as Sentry from '@sentry/react';
import { onLCP, onINP, onCLS } from 'web-vitals';

Sentry.init({
  dsn: config.SENTRY_DSN,
  release: import.meta.env.VITE_RELEASE,
  environment: config.ENV,
  tracesSampleRate: 0.1,
  replaysOnErrorSampleRate: 1.0,
  integrations: [Sentry.browserTracingIntegration(), Sentry.replayIntegration({ maskAllText: true, blockAllMedia: true })],
  beforeSend(event) { /* drop noisy/PII data */ return event; },
});

const send = (m: { name: string; value: number; id: string }) =>
  navigator.sendBeacon('/rum', JSON.stringify({ ...m, route: location.pathname }));
onLCP(send); onINP(send); onCLS(send);

root.render(
  <Sentry.ErrorBoundary fallback={<ErrorPage />}><App /></Sentry.ErrorBoundary>
);
```

**Common pitfalls / follow-up questions**
- *Minified stack traces*: upload source maps matching the release (Q170).
- *Alert fatigue*: group, rate-limit, alert on new/regressed issues.
- *Privacy*: never log tokens/PII; mask replays.

---

### Q170. How do you handle source maps, caching and CDN strategies?

**Short answer**
Generate source maps in production but don't expose them publicly: upload them to your error tracker and serve hidden. Use content-hashed filenames with long immutable caching for assets, `no-cache` for `index.html`, and a CDN for edge delivery with proper invalidation.

**Detailed explanation**
- **Source maps**: `sourcemap: 'hidden'` (no `sourceMappingURL` comment); upload via Sentry CLI/plugin in CI then delete from deploy artifact (or restrict access). Public maps expose source code.
- **Caching**: hashed assets (`app.3f2a1c.js`): `Cache-Control: public, max-age=31536000, immutable`. HTML and `config.js`/service worker: `no-cache` (revalidate with ETag). API responses: per-endpoint policy; `stale-while-revalidate` for tolerable staleness.
- **CDN**: put static assets behind CDN (CloudFront, Cloudflare, Fastly, Azure Front Door); enable Brotli/gzip, HTTP/2/3; invalidate only `index.html`; SPA fallback rewrite to index.html for routes; cross-origin assets need `crossorigin`/CORS and SRI.
- Keep prior build assets for a retention window to avoid 404s for stale clients; handle chunk-load errors with a reload prompt.
- Preconnect/preload critical resources; split vendor chunks for stable hashes.

**Code/config example**
```nginx
location ~* \.(js|css|woff2|png|svg)$ {
  add_header Cache-Control "public, max-age=31536000, immutable";
}
location = /index.html { add_header Cache-Control "no-cache"; }
location / { try_files $uri /index.html; }
gzip on;
```
```ts
// vite.config.ts
import { sentryVitePlugin } from '@sentry/vite-plugin';
export default defineConfig({
  build: { sourcemap: 'hidden' },
  plugins: [sentryVitePlugin({ org: 'acme', project: 'web',
    authToken: process.env.SENTRY_AUTH_TOKEN,
    sourcemaps: { filesToDeleteAfterUpload: ['dist/**/*.map'] } })],
});
```
```tsx
// Stale chunk handling
window.addEventListener('vite:preloadError', () => window.location.reload());
```

**Common pitfalls / follow-up questions**
- *Caching `index.html` long-term*: users never get new versions.
- *Source maps publicly served*: leak of source; use hidden + upload.
- *Chunk 404 after deploy?* Retain old assets and/or reload on preload error.

---

### Q171. How do you build PWAs and offline support (service workers)?

**Short answer**
A PWA needs HTTPS, a web app manifest, and a service worker that caches assets and handles fetches for offline use. Use Workbox (or `vite-plugin-pwa`) to precache the app shell and apply runtime caching strategies, plus a safe update flow.

**Detailed explanation**
- **Manifest**: name, icons (including maskable), `start_url`, `display: standalone`, theme color → installability.
- **Service worker** lifecycle: install (precache) → activate (clean old caches) → fetch (intercept). It runs off the main thread, with scope tied to its path.
- **Strategies**: precache app shell (cache-first); `StaleWhileRevalidate` for static/API reads; `NetworkFirst` for navigations/data needing freshness; `CacheFirst` for fonts/images with expiration.
- **Offline data**: IndexedDB (idb/Dexie), React Query persistence; queue mutations (Background Sync, Workbox `BackgroundSyncPlugin`) and reconcile conflicts; show offline UI via `navigator.onLine`/events.
- **Updates**: new SW waits; prompt user ("Reload to update") then `skipWaiting` + reload; never cache SW file aggressively.
- Test with Lighthouse, DevTools Application panel, offline mode.

**Code/config example**
```ts
// vite.config.ts
import { VitePWA } from 'vite-plugin-pwa';
export default defineConfig({
  plugins: [VitePWA({
    registerType: 'prompt',
    manifest: { name: 'Acme', short_name: 'Acme', start_url: '/', display: 'standalone',
      theme_color: '#0b5fff', icons: [
        { src: '/icon-192.png', sizes: '192x192', type: 'image/png' },
        { src: '/icon-512.png', sizes: '512x512', type: 'image/png', purpose: 'maskable' }] },
    workbox: {
      navigateFallback: '/index.html',
      runtimeCaching: [{
        urlPattern: ({ url }) => url.pathname.startsWith('/api/products'),
        handler: 'StaleWhileRevalidate',
        options: { cacheName: 'api', expiration: { maxEntries: 50, maxAgeSeconds: 3600 } },
      }],
    },
  })],
});
```
```tsx
import { useRegisterSW } from 'virtual:pwa-register/react';
function UpdatePrompt() {
  const { needRefresh: [needRefresh], updateServiceWorker } = useRegisterSW();
  return needRefresh ? <button onClick={() => updateServiceWorker(true)}>New version – reload</button> : null;
}
```

**Common pitfalls / follow-up questions**
- *Stuck on old version*: cached SW/HTML; use update prompts and `no-cache` for sw.js.
- *Caching authenticated API responses* can leak data across users; clear caches on logout.
- *Offline writes?* Queue with Background Sync and handle conflicts/idempotency.

---

### Q172. How do you upgrade major React versions in a big codebase?

**Short answer**
Upgrade incrementally: read the release notes/upgrade guide, update dependencies and libraries first for compatibility, run official codemods, fix warnings under `StrictMode`, and roll out behind tests and canary releases. Break the work into small, mergeable steps rather than a long-lived branch.

**Detailed explanation**
- **Prep**: get green CI, raise test coverage on risky areas, upgrade to the latest minor of the current major to surface deprecation warnings, audit dependencies' React peer-range support (UI kits, routers, testing libs).
- **Process**: (1) upgrade tooling (TypeScript, `@types/react`, bundler, test libs); (2) bump `react`/`react-dom`; (3) run codemods (`react-codemod`, `types-react-codemod`); (4) fix breaking changes; (5) adopt new features gradually.
- **React 18 example**: `ReactDOM.render` → `createRoot`; automatic batching; StrictMode double-invoked effects; Suspense behavior; update RTL. **React 19**: `ref` as prop (no `forwardRef` needed), removed `propTypes`/`defaultProps` for functions, `act` import from `react`, legacy context/string refs removed, new `use`, Actions, `useActionState`; TypeScript type changes.
- **Mitigation**: dual-version compatibility via aliasing in monorepo, feature branches per package, incremental rollout with flags/canary, monitoring error rates, rollback plan.
- Strangle legacy patterns (class components) over time, rather than rewriting.

**Code/config example**
```sh
# Upgrade (React 19 example)
npm i react@19 react-dom@19
npm i -D @types/react@19 @types/react-dom@19

# Official codemods
npx codemod@latest react/19/migration-recipe
npx types-react-codemod@latest preset-19 ./src
```
```tsx
// React 18 root API
import { createRoot } from 'react-dom/client';
createRoot(document.getElementById('root')!).render(<StrictMode><App /></StrictMode>);
```

**Common pitfalls / follow-up questions**
- *Big-bang upgrade branch*: merge conflicts and stalled; ship small PRs.
- *Third-party peer dependency conflicts*: check compatibility before; use `overrides` only temporarily.
- *StrictMode surfaced bugs*: effects without cleanup—fix, don't disable.

---

### Q173. How do you do a codemod migration?

**Short answer**
A codemod is a script that parses code into an AST, transforms it, and writes it back, automating large, repetitive refactors safely (jscodeshift, ts-morph, ast-grep, Babel). Write and test the transform on fixtures, run it incrementally on the codebase, review the diff, format, and verify with tests/type-check.

**Detailed explanation**
- **Tools**: `jscodeshift` (Babel-based, recast preserves formatting), `ts-morph` (type-aware TS), `ast-grep`/Comby (pattern-based), Codemod platform; ESLint autofix for simple rules.
- **Workflow**: define precise before/after; write transform; unit-test with input/output fixtures (`defineTest`); dry-run on a subset; run per directory/package in separate PRs; run Prettier; run type-check/tests; handle leftovers manually and add lint rule to prevent regressions (e.g., `no-restricted-imports`).
- Make transforms idempotent and conservative—skip ambiguous cases and log them for manual follow-up.
- Common uses: API renames, replacing deprecated components, converting `defaultProps`, import path changes, class → function components.

**Code/config example**
```js
// transforms/rename-button-prop.js  (<Button type="primary"> → <Button variant="primary">)
module.exports = function transformer(file, api) {
  const j = api.jscodeshift;
  const root = j(file.source);
  let changed = false;

  root.find(j.JSXOpeningElement, { name: { name: 'Button' } }).forEach(path => {
    j(path).find(j.JSXAttribute, { name: { name: 'type' } }).forEach(attr => {
      attr.node.name.name = 'variant';
      changed = true;
    });
  });
  return changed ? root.toSource({ quote: 'single' }) : null;
};
```
```sh
npx jscodeshift -t transforms/rename-button-prop.js src --extensions=tsx,jsx --parser=tsx --dry --print
npx jscodeshift -t transforms/rename-button-prop.js src --extensions=tsx,jsx --parser=tsx
npx prettier --write src && npx tsc --noEmit && npm test
```
```js
// __tests__/rename-button-prop.test.js
const { defineTest } = require('jscodeshift/dist/testUtils');
defineTest(__dirname, 'rename-button-prop', null, 'rename-button-prop', { parser: 'tsx' });
```

**Common pitfalls / follow-up questions**
- *Over-broad matching* (e.g., any `type` prop on a different `Button` import): scope by import source.
- *Formatting churn*: recast preserves most, but run Prettier in a separate commit.
- *Large repos?* Split by package/owner, run in CI-assisted batches, and add lint rules to stop reintroduction.
