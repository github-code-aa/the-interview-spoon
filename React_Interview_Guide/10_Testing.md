# 10. Testing (Q142–Q152)

## Table of Contents
- [Q142. Testing pyramid](#q142-what-is-the-testing-pyramid-for-react)
- [Q143. Jest vs Vitest](#q143-jest-vs-vitest)
- [Q144. React Testing Library](#q144-what-is-react-testing-library-why-test-behavior-not-implementation)
- [Q145. Testing hooks and async code](#q145-how-do-you-test-hooks-and-async-code)
- [Q146. Mocking APIs with MSW](#q146-how-do-you-mock-apis-msw)
- [Q147. Context, routing and forms](#q147-how-do-you-test-context-routing-and-forms)
- [Q148. Snapshot tests](#q148-what-are-snapshot-tests-and-when-are-they-harmful)
- [Q149. E2E tests](#q149-what-are-e2e-tests-cypress-playwright-how-do-you-avoid-flakiness)
- [Q150. Visual regression and a11y testing](#q150-how-do-you-do-visual-regression-and-accessibility-testing)
- [Q151. Error boundaries and Suspense](#q151-how-do-you-test-error-boundaries-and-suspense)
- [Q152. Coverage target](#q152-what-coverage-target-is-reasonable)

---

### Q142. What is the testing pyramid for React?

**Short answer**
The pyramid says to have many fast, cheap unit tests at the base, a moderate number of integration tests in the middle, and few slow, expensive E2E tests at the top. For React apps, many teams favor the "testing trophy": static analysis (TypeScript/ESLint) at the bottom, and most value from integration tests that render components together, the way users use them.

**Detailed explanation**
- **Static**: TypeScript, ESLint catch typos and type errors for free.
- **Unit**: pure functions, reducers, selectors, utility hooks. Fast, isolated, precise failures.
- **Integration**: render a feature (page/component tree with providers, mocked network) using React Testing Library. Best confidence-to-cost ratio for UI.
- **E2E**: real browser, real (or staging) backend; covers critical user journeys (login, checkout). Slow and flakier, so keep few.
- Shape depends on app: UI-heavy apps lean integration; logic-heavy apps lean unit. The principle: push tests as low as possible while still giving confidence, and avoid duplicating coverage across layers.

**Code/config example**
```jsonc
// package.json scripts mapping to the layers
{
  "scripts": {
    "lint": "eslint . && tsc --noEmit",          // static
    "test": "vitest run",                         // unit + integration (RTL)
    "test:e2e": "playwright test"                 // E2E, run on CI against preview build
  }
}
```

**Common pitfalls / follow-up questions**
- *Ice-cream cone anti-pattern*: mostly E2E/manual tests → slow, flaky feedback. Fix by moving checks down to integration tests.
- *Why not only unit tests?* Components can each pass while the wiring (context, routing, data fetching) is broken; integration tests catch that.
- *What belongs in E2E?* Only critical happy paths and a few high-risk flows, not every validation message.

---

### Q143. Jest vs Vitest?

**Short answer**
Jest is the long-standing, batteries-included runner (CJS-first, babel/ts-jest transforms); Vitest is a Vite-native runner with a Jest-compatible API, native ESM/TypeScript, and fast watch mode that reuses the Vite config. Choose Vitest for Vite projects and new code; Jest remains fine for existing large suites, React Native, or Next.js setups configured around it.

**Detailed explanation**
- **Config reuse**: Vitest shares `vite.config` (aliases, plugins), so there's no duplicate transform pipeline. Jest needs babel/swc/ts-jest and `moduleNameMapper`.
- **Speed**: Vitest uses esbuild/Vite transforms and worker threads; HMR-like re-runs in watch mode.
- **API**: `describe/it/expect/vi.fn()` vs `jest.fn()`; `vi.mock` is hoisted like `jest.mock`. Set `globals: true` to mimic Jest.
- **Environment**: both support `jsdom`/`happy-dom`. Vitest also has browser mode (real browser) and in-source testing.
- **Ecosystem**: Jest has the larger plugin ecosystem and React Native support; snapshot format is mostly compatible.
- Migration is typically search/replace `jest` → `vi` plus config.

**Code/config example**
```ts
// vite.config.ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: './src/test/setup.ts',
    css: true,
    coverage: { provider: 'v8', reporter: ['text', 'lcov'] },
  },
});

// src/test/setup.ts
import '@testing-library/jest-dom/vitest';
import { afterEach } from 'vitest';
import { cleanup } from '@testing-library/react';
afterEach(cleanup);
```

**Common pitfalls / follow-up questions**
- *Fake timers/module mocking differences*: `vi.mock` factories must not reference outer variables unless via `vi.hoisted`. 
- *jsdom gaps*: neither implements layout, `IntersectionObserver`, `matchMedia`; polyfill/mocks in setup.
- *Why not migrate everything?* Cost/benefit; if Jest suite is stable and fast enough, migration offers little.

---

### Q144. What is React Testing Library? Why test behavior, not implementation?

**Short answer**
React Testing Library (RTL) renders components into a DOM and provides queries that mirror how users find elements (by role, label, text). Testing behavior rather than implementation keeps tests resilient to refactors and ensures they verify what users actually experience.

**Detailed explanation**
- Guiding principle: "The more your tests resemble the way your software is used, the more confidence they can give you."
- **Query priority**: `getByRole` (with `name`) > `getByLabelText` > `getByPlaceholderText` > `getByText` > `getByDisplayValue` > `getByAltText`/`getByTitle` > `getByTestId` (last resort). Role queries also validate accessibility.
- **Variants**: `getBy` (throws if missing), `queryBy` (null; for asserting absence), `findBy` (async, waits).
- **userEvent** simulates realistic interactions (focus, keydown, click sequence) vs. low-level `fireEvent`.
- Implementation-detail tests (checking state values, calling instance methods, shallow rendering, asserting child component props) break on refactor (false negatives) and miss real bugs (false positives).

**Code/config example**
```tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { Counter } from './Counter';

test('increments when the button is clicked', async () => {
  const user = userEvent.setup();
  render(<Counter />);

  await user.click(screen.getByRole('button', { name: /increment/i }));

  expect(screen.getByText(/count: 1/i)).toBeInTheDocument();
});
```

**Common pitfalls / follow-up questions**
- *Overusing `data-testid`*: bypasses accessibility; prefer roles/labels.
- *Wrapping everything in `act`*: RTL already wraps; use `await user...` and `findBy`/`waitFor` instead of manual `act`.
- *Is testing state ever OK?* Test observable outcomes (DOM, network calls, navigation); internal state is an implementation detail.

---

### Q145. How do you test hooks and async code?

**Short answer**
Test custom hooks with `renderHook` from RTL (use `act` for updates), or—better—through a component that uses them. For async code, use `findBy*`/`waitFor`, `await user.*`, and fake timers or mocked network; never use arbitrary sleeps.

**Detailed explanation**
- `renderHook(() => useX(args), { wrapper })` returns `result.current`, `rerender`, `unmount`. Provide a `wrapper` for context/query-client.
- State updates triggered outside RTL helpers need `act()`; `waitFor` polls until assertion passes or times out.
- **Timers**: `vi.useFakeTimers()` + `vi.advanceTimersByTime()` for debounce/polling; with userEvent use `userEvent.setup({ advanceTimers: vi.advanceTimersByTime })`.
- **Async data**: mock at the network layer (MSW) so hook code remains real. For React Query, create a fresh `QueryClient` per test with `retry: false`.
- Prefer testing hooks via a component if the hook is tightly coupled to rendering/DOM.

**Code/config example**
```tsx
import { renderHook, act, waitFor } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { useCounter } from './useCounter';
import { useUser } from './useUser';

test('useCounter increments', () => {
  const { result } = renderHook(() => useCounter(0));
  act(() => result.current.increment());
  expect(result.current.count).toBe(1);
});

const wrapper = ({ children }: { children: React.ReactNode }) => (
  <QueryClientProvider client={new QueryClient({ defaultOptions: { queries: { retry: false } } })}>
    {children}
  </QueryClientProvider>
);

test('useUser loads data', async () => {
  const { result } = renderHook(() => useUser(1), { wrapper });
  await waitFor(() => expect(result.current.isSuccess).toBe(true));
  expect(result.current.data?.name).toBe('Ada');
});
```

**Common pitfalls / follow-up questions**
- *"not wrapped in act" warnings*: usually an un-awaited async update; await `findBy`/`waitFor`.
- *Shared QueryClient between tests* leaks cache; create new per test.
- *Fake timers + `waitFor` hang?* Use `advanceTimers` option or `vi.runAllTimersAsync()`.

---

### Q146. How do you mock APIs (MSW)?

**Short answer**
Mock Service Worker intercepts requests at the network level (service worker in browser, request interception in Node), so app code uses real `fetch`/axios unchanged. Define handlers once and reuse them across unit, integration, Storybook and dev.

**Detailed explanation**
- Handlers use `http.get/post` and `HttpResponse.json` (MSW v2). 
- In tests: `setupServer(...handlers)` with `beforeAll(listen)`, `afterEach(resetHandlers)`, `afterAll(close)`. Use `onUnhandledRequest: 'error'` to catch unmocked calls.
- Override per test with `server.use(...)` to simulate errors, delays, empty states.
- Advantages over mocking `fetch`/modules: tests the real request code, headers, serialization, error handling; the same contract mocks are reusable.
- Combine with `@mswjs/data` or OpenAPI-generated handlers for consistency.

**Code/config example**
```ts
// src/mocks/handlers.ts
import { http, HttpResponse } from 'msw';
export const handlers = [
  http.get('/api/users/:id', ({ params }) =>
    HttpResponse.json({ id: params.id, name: 'Ada' })),
];

// src/mocks/server.ts
import { setupServer } from 'msw/node';
import { handlers } from './handlers';
export const server = setupServer(...handlers);

// setup.ts
beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

// in a test: simulate failure
server.use(http.get('/api/users/:id', () => new HttpResponse(null, { status: 500 })));
```

**Common pitfalls / follow-up questions**
- *Relative URLs in Node*: jsdom `fetch` needs absolute URLs or a base; configure base URL.
- *Forgetting `resetHandlers`* leaks overrides to other tests.
- *MSW vs mocking modules?* MSW is more realistic; module mocks are fine for isolated unit tests.

---

### Q147. How do you test context, routing and forms?

**Short answer**
Wrap the component in the same providers it has in production (via a custom `render` helper): the context provider, a `MemoryRouter`/`createMemoryRouter`, and a QueryClient. For forms, drive the UI with `userEvent`, then assert on validation messages, submitted payloads (via MSW or a mock handler), and resulting navigation.

**Detailed explanation**
- **Context**: render with a real provider and chosen initial value; test consumers through UI. Avoid mocking `useContext`.
- **Routing**: `MemoryRouter initialEntries` or `createMemoryRouter(routes, { initialEntries })` + `RouterProvider` for data routers (loaders/actions). Assert on rendered route content or `router.state.location`.
- **Forms**: type, select, check, submit; query by label; assert accessible errors (`role="alert"`). With React Hook Form/Formik, no special handling is needed—test as a user.
- Create a `renderWithProviders(ui, options)` helper to avoid repetition.

**Code/config example**
```tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { createMemoryRouter, RouterProvider } from 'react-router-dom';
import { ThemeProvider } from './theme';
import { routes } from './routes';

function renderApp(path = '/') {
  const router = createMemoryRouter(routes, { initialEntries: [path] });
  render(<ThemeProvider value="dark"><RouterProvider router={router} /></ThemeProvider>);
  return router;
}

test('login form validates and redirects', async () => {
  const user = userEvent.setup();
  const router = renderApp('/login');

  await user.click(screen.getByRole('button', { name: /sign in/i }));
  expect(await screen.findByRole('alert')).toHaveTextContent(/email is required/i);

  await user.type(screen.getByLabelText(/email/i), 'a@b.com');
  await user.type(screen.getByLabelText(/password/i), 'secret123');
  await user.click(screen.getByRole('button', { name: /sign in/i }));

  expect(await screen.findByRole('heading', { name: /dashboard/i })).toBeInTheDocument();
  expect(router.state.location.pathname).toBe('/dashboard');
});
```

**Common pitfalls / follow-up questions**
- *Using `BrowserRouter` in tests* mutates jsdom history across tests; use memory routers.
- *Missing provider errors*: use shared render helper.
- *How to test protected routes?* Render with auth context value unauthenticated and assert redirect.

---

### Q148. What are snapshot tests, and when are they harmful?

**Short answer**
Snapshot tests serialize output (DOM, JSON) to a file and fail when it changes. They're harmful when large or auto-updated blindly: they're brittle, noisy, and reviewers just press "update," so they stop catching real bugs.

**Detailed explanation**
- Good for: small, stable, serializable output—error messages, config objects, generated code, an icon's SVG, or `toMatchInlineSnapshot` on a small piece.
- Bad for: full component trees; any style/markup change breaks tests; they assert implementation, not behavior; diffs are huge and unreviewed (`-u` habit).
- Alternatives: explicit assertions on behavior; visual regression for appearance.
- Mitigations: keep snapshots small and focused, prefer inline, review snapshot diffs in PRs, fail CI on obsolete snapshots (`--ci`).

**Code/config example**
```tsx
// Harmful: huge, opaque
expect(container).toMatchSnapshot();

// Acceptable: small, intention-revealing
expect(formatPrice({ amount: 1999, currency: 'USD' })).toMatchInlineSnapshot(`"$19.99"`);

// Better for UI: behavior
expect(screen.getByRole('status')).toHaveTextContent('Saved');
```

**Common pitfalls / follow-up questions**
- *Snapshot includes volatile data* (dates, ids, generated classnames): mock or serialize-out.
- *Why do people say "snapshot fatigue"?* Developers update without reading.
- *Does Storybook/Chromatic replace them?* For visuals, yes—pixel/DOM visual diffs are a better fit.

---

### Q149. What are E2E tests (Cypress, Playwright)? How do you avoid flakiness?

**Short answer**
E2E tests drive a real browser through the full app (UI → API → DB or stubbed backend) to validate critical user journeys. Avoid flakiness by relying on auto-waiting, resilient role/label locators, isolated deterministic test data, network control, and no fixed sleeps.

**Detailed explanation**
- **Playwright**: multi-browser (Chromium/Firefox/WebKit), auto-waiting, parallel workers, trace viewer, multiple tabs/contexts, API testing. **Cypress**: great DX and time-travel debugger, runs in-browser; historically single-tab, now supports more browsers.
- **Flakiness causes**: race conditions, arbitrary timeouts, shared/stateful data, animations, third-party services, order-dependent tests, unstable selectors.
- **Remedies**: web-first assertions that retry; `getByRole`; seed data per test via API and log in via stored `storageState` (skip UI login); mock third parties with `page.route`; disable animations; run against a production build; retries on CI only with trace on first retry; quarantine and fix flaky tests, track flake rate.

**Code/config example**
```ts
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';
export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  retries: process.env.CI ? 2 : 0,
  use: { baseURL: 'http://localhost:4173', trace: 'on-first-retry' },
  webServer: { command: 'npm run build && npm run preview', url: 'http://localhost:4173' },
  projects: [{ name: 'chromium', use: { ...devices['Desktop Chrome'] } }],
});

// e2e/checkout.spec.ts
import { test, expect } from '@playwright/test';
test('user can add item and checkout', async ({ page }) => {
  await page.route('**/api/payments', r => r.fulfill({ json: { ok: true } }));
  await page.goto('/products');
  await page.getByRole('button', { name: 'Add to cart' }).first().click();
  await page.getByRole('link', { name: 'Cart' }).click();
  await page.getByRole('button', { name: 'Pay' }).click();
  await expect(page.getByText('Order confirmed')).toBeVisible(); // auto-retries
});
```

**Common pitfalls / follow-up questions**
- *`waitForTimeout`/`cy.wait(5000)`*: replace with assertions on state/network.
- *Retries hide flakiness*: monitor retried-pass tests.
- *Playwright vs Cypress?* Playwright for cross-browser/parallel/multi-tab; Cypress for component-testing DX and ecosystem—both valid.

---

### Q150. How do you do visual regression and accessibility testing?

**Short answer**
Visual regression compares screenshots (or DOM snapshots) against approved baselines using Playwright `toHaveScreenshot`, Storybook + Chromatic/Percy. Accessibility testing combines automated checks (eslint-plugin-jsx-a11y, jest-axe/axe-core, Lighthouse) with manual keyboard and screen-reader testing, since automation catches only roughly a third of issues.

**Detailed explanation**
- **Visual**: render components in isolation (Storybook) or pages in E2E; run in a consistent environment (same Docker image/OS, fonts) to avoid anti-aliasing diffs; mask dynamic regions; freeze time/animations; review diffs in PR.
- **A11y automation**: lint at author time; `jest-axe`/`vitest-axe` in unit tests; `@axe-core/playwright` in E2E; Storybook a11y addon; CI Lighthouse.
- **Manual**: keyboard-only, screen readers (NVDA, VoiceOver), zoom/reflow, high contrast, reduced motion.
- Target WCAG 2.2 AA.

**Code/config example**
```ts
// a11y unit test (vitest-axe)
import { axe } from 'vitest-axe';
test('form has no a11y violations', async () => {
  const { container } = render(<SignupForm />);
  expect(await axe(container)).toHaveNoViolations();
});

// Playwright: a11y + visual
import AxeBuilder from '@axe-core/playwright';
test('home is accessible and visually stable', async ({ page }) => {
  await page.goto('/');
  const results = await new AxeBuilder({ page }).withTags(['wcag2a', 'wcag2aa']).analyze();
  expect(results.violations).toEqual([]);
  await expect(page).toHaveScreenshot('home.png', { maxDiffPixelRatio: 0.01, mask: [page.locator('.clock')] });
});
```

**Common pitfalls / follow-up questions**
- *Flaky screenshots* due to fonts/animation/OS: pin environment, disable animations.
- *"Axe passes so it's accessible"?* No; it can't judge focus order, meaningful alt text, or screen-reader UX.
- *Where to run visual tests?* On stories (cheap, isolated) plus a few key pages.

---

### Q151. How do you test error boundaries and Suspense?

**Short answer**
For error boundaries, render a component that throws, assert the fallback UI appears (and recovery/reset works), silencing the expected `console.error`. For Suspense, render inside `<Suspense fallback>` and assert the fallback first, then use `findBy*` for resolved content, controlling data with MSW or a deferred promise.

**Detailed explanation**
- Error boundaries only catch render/lifecycle errors in descendants (not event handlers or async code). React logs the error; spy on `console.error` to keep output clean.
- Test `onError`/logging callback and reset (`resetErrorBoundary`/`resetKeys`) behavior; with `react-error-boundary`, the fallback receives `error` and `resetErrorBoundary`.
- Suspense: lazy components (`React.lazy`), `use()`, or Suspense-enabled libs (`useSuspenseQuery`). Check fallback → content transition. For React 18+, RTL `render` handles concurrent root; use `await screen.findBy...`.
- Per-test fresh cache/QueryClient to avoid stale resolved resources.

**Code/config example**
```tsx
import { ErrorBoundary } from 'react-error-boundary';
import { Suspense } from 'react';

function Bomb() { throw new Error('boom'); }

test('shows fallback and recovers', async () => {
  const user = userEvent.setup();
  const spy = vi.spyOn(console, 'error').mockImplementation(() => {});
  let shouldThrow = true;
  const Child = () => { if (shouldThrow) throw new Error('boom'); return <p>OK</p>; };

  render(
    <ErrorBoundary fallbackRender={({ resetErrorBoundary }) => (
      <div role="alert">Something went wrong
        <button onClick={resetErrorBoundary}>Retry</button></div>)}>
      <Child />
    </ErrorBoundary>
  );
  expect(screen.getByRole('alert')).toHaveTextContent(/went wrong/i);

  shouldThrow = false;
  await user.click(screen.getByRole('button', { name: /retry/i }));
  expect(screen.getByText('OK')).toBeInTheDocument();
  spy.mockRestore();
});

test('suspense shows fallback then data', async () => {
  render(<Suspense fallback={<p>Loading…</p>}><UserProfile id={1} /></Suspense>);
  expect(screen.getByText('Loading…')).toBeInTheDocument();
  expect(await screen.findByRole('heading', { name: 'Ada' })).toBeInTheDocument();
});
```

**Common pitfalls / follow-up questions**
- *Expecting boundaries to catch event-handler errors*: they don't; handle with try/catch or state.
- *Noisy console*: expected; spy/mock it but restore it.
- *How to test lazy loading?* Same pattern: `findBy` after dynamic import resolves.

---

### Q152. What coverage target is reasonable?

**Short answer**
A commonly reasonable target is roughly 70–80% line/branch coverage overall, with higher thresholds (90%+) on critical business logic and lower on glue/UI boilerplate. Coverage is a signal for untested code, not a measure of quality—100% is usually wasteful.

**Detailed explanation**
- Coverage shows what code executed, not whether assertions are meaningful. Tests without assertions can reach 100%.
- Prefer branch coverage over line coverage; track coverage on *changed code* (patch coverage, e.g., Codecov) to prevent regressions without chasing legacy numbers.
- Enforce thresholds in CI to avoid decline; exclude generated code, stories, types, config.
- Supplement with mutation testing (Stryker) for critical modules to check test strength.
- Focus effort on risk: payments, auth, data transforms.

**Code/config example**
```ts
// vitest.config.ts
export default defineConfig({
  test: {
    coverage: {
      provider: 'v8',
      reporter: ['text', 'lcov'],
      include: ['src/**/*.{ts,tsx}'],
      exclude: ['src/**/*.stories.tsx', 'src/**/*.d.ts', 'src/main.tsx'],
      thresholds: { lines: 75, branches: 70, functions: 75, statements: 75,
        'src/domain/**': { lines: 90, branches: 85 } },
    },
  },
});
```

**Common pitfalls / follow-up questions**
- *Goodhart's law*: mandating 100% encourages meaningless tests.
- *Is low coverage always bad?* Not if the covered parts are the risky ones, and E2E covers journeys.
- *How to improve quality beyond coverage?* Mutation testing, code review of tests, bug-driven regression tests.
