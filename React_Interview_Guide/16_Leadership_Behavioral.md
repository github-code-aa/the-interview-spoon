# 16. Leadership & Behavioral (Q223-230)

## Table of Contents
- [How to use the STAR template](#star)
- [Q223. Code reviews for React code](#q223)
- [Q224. Tech choices: build vs buy, library selection](#q224)
- [Q225. Front-end standards and mentoring juniors](#q225)
- [Q226. Delivery speed vs technical debt](#q226)
- [Q227. Estimating front-end work](#q227)
- [Q228. Collaborating with designers and backend teams](#q228)
- [Q229. React vs Angular vs Vue vs Svelte](#q229)
- [Q230. Where React is heading](#q230)

---

<a id="star"></a>
## How to use the STAR template
**S**ituation (1-2 sentences, context and stakes) -> **T**ask (your responsibility) -> **A**ction (what *you* did, 3-5 concrete steps, the bulk of the answer) -> **R**esult (measurable outcome + lesson learned). Keep to ~2 minutes; have numbers ready; use real stories (the examples below are talking points, not scripts to memorize or fabricate).

---

<a id="q223"></a>
### Q223. How do you do code reviews for React code?

**Context / clarifying questions**
Interviewers want to see that reviews improve quality *and* team velocity and culture, not gatekeeping.

**Diagnosis or approach (step by step)**
1. Automate the boring stuff (ESLint with `react-hooks`, Prettier, TypeScript, tests, bundle-size checks) so humans review design.
2. Review in layers: correctness -> design/maintainability -> performance -> accessibility -> security -> tests -> naming/style.
3. Keep PRs small (<400 lines); require a clear description, screenshots/video for UI, and linked ticket.
4. Respond within an agreed SLA (e.g., 1 business day).

**Solution with code or architecture diagram — checklist and model answer**

React-specific checklist:
- Hooks: correct dependency arrays, cleanup of effects/subscriptions, no effects for derived state ("You Might Not Need an Effect"), no conditional hooks.
- State: colocated, minimal, single source of truth; server state in a query lib, not copied into `useState`.
- Rendering: stable keys, no inline object/function props on hot paths with `memo`, avoid unnecessary context churn.
- Components: single responsibility, composition over prop-drilling/boolean prop explosion, controlled vs uncontrolled used consistently.
- Accessibility: semantic elements, labels, keyboard support, focus management, alt text.
- Security: no `dangerouslySetInnerHTML` without sanitization, no secrets in client code, validate URLs.
- Types and tests: no `any`, behavior-focused RTL tests, error/loading/empty states covered.

```tsx
// Typical review comment - suggesting derived value instead of effect+state
// Instead of:
const [filtered, setFiltered] = useState<Item[]>([]);
useEffect(() => setFiltered(items.filter(match)), [items, match]);
// Prefer:
const filtered = useMemo(() => items.filter(match), [items, match]);
```
STAR-style template:
- **S:** "A teammate's PR touched a core table component, and it had subtle re-render and a11y issues."
- **T:** "I was the reviewer and wanted to fix quality without discouraging them."
- **A:** "I used prefixed comments (`nit:`, `question:`, `blocking:`), explained the 'why' with a link/snippet, offered to pair, and approved with follow-ups for non-blocking items. I also added an ESLint rule so the issue is caught automatically."
- **R:** "Review turnaround fell to <1 day, repeated issues dropped, and the teammate started catching the same patterns in others' PRs."

**Tradeoffs**
- Thorough reviews catch bugs but can bottleneck; balance with automation and trust (tiered scrutiny for risky areas).
- Style nitpicks waste goodwill; delegate to tooling.

**How to present this in an interview**
Show empathy plus rigor: "I review for the reader and the next maintainer". Mention tone (questions not commands), being reviewed yourself, and automation.

---

<a id="q224"></a>
### Q224. How do you decide on tech choices (build vs buy, library selection)?

**Context / clarifying questions**
Frame the answer as a repeatable decision process rather than preference.

**Diagnosis or approach (step by step)**
1. Define the problem and success criteria (requirements, constraints, timeline, team skills).
2. Is it core differentiating? Build only what differentiates; buy/adopt commodity (auth, payments, date pickers, grids, analytics).
3. Evaluate candidates (2-3) with a weighted matrix: fit, maintenance health (release cadence, issues, bus factor), bundle size, TypeScript support, accessibility, license, security record, community/hiring pool, upgrade path, escape hatches/lock-in.
4. Prove with a time-boxed spike/POC against the real hardest requirement.
5. Total cost of ownership: build cost + ongoing maintenance vs license + integration + lock-in.
6. Document in an ADR (context, options, decision, consequences) and set a review trigger; wrap third-party libs behind thin adapters for replaceability.

**Solution with code or architecture diagram — model answer**
```
Need -> Core/differentiating? --yes--> Build (own it)
                              \--no--> Shortlist 2-3 -> Matrix -> Spike -> ADR -> Adapter wrapper -> Review at 6-12 months
```
- **S:** "We needed an enterprise data grid with grouping, pinning and export."
- **T:** "I led the evaluation and recommendation."
- **A:** "Estimated custom build at ~4 engineer-months plus ongoing a11y/perf upkeep. Compared AG Grid, TanStack Table (headless) and MUI X using a matrix, and spiked each against our hardest case (50k rows, inline edit, keyboard nav). Checked license cost, bundle size, and accessibility. Wrote an ADR and wrapped the choice in our `DataGrid` component."
- **R:** "Shipped in 3 weeks instead of months; the wrapper let us swap a feature later without touching consumers."

Example talking points: "boring technology" bias, avoiding resume-driven development, checking npm download trends/maintainers/last release, license audits, security (`npm audit`, Snyk), exit strategy.

**Tradeoffs**
- Build: control, no lock-in, but maintenance forever. Buy: speed, but cost, constraints, vendor risk.
- Newest tech gives capability but risk and hiring difficulty.

**How to present this in an interview**
Emphasize criteria, a time-boxed spike, and ADRs. Show you can also say "no" to a shiny library and revisit decisions with data.

---

<a id="q225"></a>
### Q225. How do you set front-end standards and mentor juniors?

**Context / clarifying questions**
Show you create guardrails and growth, not rules for their own sake.

**Diagnosis or approach (step by step)**
1. Standards: written, short, enforced by tooling — ESLint/Prettier/TS strict configs, folder and naming conventions, component and testing guidelines, a11y checklist, PR template, Definition of Done. Keep them in the repo and evolve through RFC/ADR with team input.
2. Golden paths: starter templates, example features, Storybook, shared hooks/utilities.
3. Mentoring: assess level and goals; pair programming; stretch assignments with safety nets; code review as teaching (explain why); regular 1:1s; learning sessions/brown-bags; encourage them to present.
4. Gradually withdraw support (I do -> we do -> you do -> you teach).
5. Measure: PR cycle time, defect rate, onboarding time to first PR, junior autonomy.

**Solution with code or architecture diagram — model answer**
- **S:** "Our team of 8 had inconsistent patterns — three ways to fetch data and no shared testing approach — and new joiners took ~6 weeks to be productive."
- **T:** "As senior engineer I owned improving consistency and onboarding."
- **A:** "Ran a short RFC with the team to agree on conventions (TanStack Query for server state, RTL for tests, feature-folder layout). Encoded them in lint rules and a project template, wrote a 'how we build features' guide with a reference feature, and set up weekly pairing plus a 30-60-90 plan for a junior. Gave one junior ownership of a small feature end-to-end, reviewing design before coding."
- **R:** "Time to first merged PR dropped from ~2 weeks to 3 days, PR review rounds fell, and the junior later led the form-library upgrade."

Talking points: psychological safety, "teach to fish" (ask questions before giving answers), feedback that is specific/kind/actionable, celebrate learning from failures, avoid becoming a bottleneck.

**Tradeoffs**
- Strict standards reduce variation but can stifle innovation; allow documented exceptions.
- Mentoring costs short-term velocity but compounds.

**How to present this in an interview**
Give one concrete mentee story and one standards-by-automation story; show standards were co-created, not dictated.

---

<a id="q226"></a>
### Q226. How do you balance delivery speed against technical debt?

**Context / clarifying questions**
Show pragmatism: debt is a tool, not a sin, but it needs management.

**Diagnosis or approach (step by step)**
1. Distinguish deliberate/prudent debt (ship now, plan to fix) from accidental/reckless debt.
2. Make it visible: track debt items in the backlog with impact, cost and interest (what it slows down); tag in code (`TODO(ticket)`).
3. Prioritize by interest: debt in hot paths/frequently changed areas or causing incidents first; ignore stable ugly code.
4. Fund it continuously: reserve ~10-20% capacity, follow the boy-scout rule, bundle refactors with feature work touching the same area.
5. Communicate in business terms ("this adds 2 days to every checkout change", "causes 30% of bugs") and negotiate trade-offs with product explicitly.
6. Guardrails: tests around risky shortcuts, flags, time-boxed expiry for hacks, and quality gates that never get skipped (security, accessibility of critical flows).
7. Use metrics: lead time, change failure rate, bug rate, build time.

**Solution with code or architecture diagram — model answer**
- **S:** "Ahead of a launch, product wanted a pricing feature in 3 weeks; the existing pricing module was tangled."
- **T:** "I had to hit the date without making the module unmaintainable."
- **A:** "Proposed a scoped plan: ship the MVP by wrapping the old logic with a clean interface and tests, explicitly logging the shortcut as a ticket with an estimate and impact. Aligned with the PM that the next sprint would include the refactor, and used a flag to ship safely."
- **R:** "Launched on time with no incidents; the refactor reduced pricing change effort from ~5 days to ~1 and we kept the 15% debt budget thereafter."

Talking points: "debt interest", Quadrant (Fowler), Definition of Done, avoid big-bang rewrites, say no to debt that has security/compliance implications.

**Tradeoffs**
- Too much speed focus slows you down later; too much cleanup delays value and loses stakeholder trust.

**How to present this in an interview**
Show you negotiate transparently with product using data, and that you pay debt in small continuous installments tied to business value.

---

<a id="q227"></a>
### Q227. How do you estimate front-end work?

**Context / clarifying questions**
Show you handle uncertainty honestly. Ask about requirements, designs readiness, API availability, and who will do the work.

**Diagnosis or approach (step by step)**
1. Clarify scope and acceptance criteria; confirm designs/API contracts or call out as risks/assumptions.
2. Break down into tasks of <=1-2 days: component structure, states (loading/empty/error), responsive, a11y, i18n, analytics, tests, integration, QA fixes, code review, deployment/flags.
3. Include the "hidden" work: cross-browser testing, edge cases, design review iterations, accessibility, performance, documentation.
4. Estimate with ranges (best/likely/worst) or three-point (PERT), or relative sizing (story points) calibrated on past velocity; reference similar past work.
5. Add buffer for risks and unknown dependencies; list assumptions explicitly.
6. Spikes for high-uncertainty items (new library, unclear API) — time-boxed before committing.
7. Re-estimate as you learn; track actual vs estimate to improve.

**Solution with code or architecture diagram — model answer**
```
Estimate = sum(tasks, three-point) + risk buffer (15-30%) ;  Range communicated: "6-9 days, likely 7, assuming API ready by Mon"
```
- **S:** "PM asked for an estimate on a multi-step onboarding wizard with an unfinished API."
- **T:** "Provide a reliable estimate for planning."
- **A:** "Broke it into steps/states, did a 1-day spike for the form-state approach, flagged the API as the main risk, gave a range (8-12 days) with assumptions, and proposed delivering step 1-2 first to de-risk. Re-estimated after the API contract landed."
- **R:** "Delivered in 10 days, within the range; PM trusted ranges and assumptions more than a single number."

Talking points: estimates aren't commitments; avoid padding silently; cone of uncertainty; slice vertically; use historical data.

**Tradeoffs**
- More detail = more accuracy but more planning time; use lighter estimates early (T-shirt sizes) and refine.

**How to present this in an interview**
Emphasize ranges, assumptions, spikes, hidden work (states, a11y, tests) and learning from actuals.

---

<a id="q228"></a>
### Q228. How do you collaborate with designers and backend teams?

**Context / clarifying questions**
Show proactive, early, contract-based collaboration.

**Diagnosis or approach (step by step)**
**Designers**
1. Involve early in discovery; review designs for feasibility (states, responsiveness, edge cases, a11y, performance) before handoff.
2. Shared language: design tokens synced between Figma and code, a component library/Storybook matching the design system.
3. Ask for all states (loading, empty, error, long text, RTL) and interaction details; offer alternatives when something is costly.
4. Design QA before release; pair on pixel/interaction details.

**Backend**
1. Agree API contracts first (OpenAPI/GraphQL schema), generate types/clients, and mock with MSW so work proceeds in parallel.
2. Discuss pagination, error format, auth, idempotency, caching and versioning; consider a BFF for UI-shaped endpoints.
3. Contract tests in CI; communicate breaking changes with deprecation windows.
4. Joint incident reviews; shared observability (trace IDs from UI to API).

**Solution with code or architecture diagram — model answer**
```
Discovery (PM+Design+FE+BE) -> Contract (OpenAPI) + Figma states -> Parallel build (MSW mocks) -> Integrate -> Design QA -> Release
```
- **S:** "A new reporting feature had unclear data shape and a design that required real-time filters over large data."
- **T:** "Align design and backend so we could deliver in one sprint-set."
- **A:** "Held a kickoff to flag that instant filtering over raw data wasn't feasible; proposed server-side aggregation and debounced filters. Co-wrote the OpenAPI spec with the backend lead, generated typed clients, mocked it with MSW, and had the designer adjust loading/empty states and a skeleton design. Did design QA together before release."
- **R:** "Integration took 1 day instead of a week of rework, and the feature met performance targets."

Talking points: disagree-and-commit, assume good intent, bring data/prototypes to resolve debates, document decisions, be a "T-shaped" engineer who understands the other disciplines.

**Tradeoffs**
- Early involvement takes more meetings but avoids costly rework; contract-first needs discipline when specs change.

**How to present this in an interview**
Use a concrete cross-team story where you changed an outcome through early feasibility feedback and contracts.

---

<a id="q229"></a>
### Q229. React vs Angular vs Vue vs Svelte: when would you choose each?

**Context / clarifying questions**
Not behavioral, so no STAR needed; give a decision framework with honest tradeoffs. Ask: team skills, app type, longevity, SEO, performance targets, hiring market.

**Diagnosis or approach (step by step)**
Evaluate on: team experience and hiring, ecosystem maturity, app size/complexity, structure/opinionation needed, performance and bundle constraints, SSR/SEO, mobile/native needs, long-term support.

**Solution with code or architecture diagram — comparison**

| Framework | Strengths | Weaknesses | Choose when |
|---|---|---|---|
| **React** | Huge ecosystem and talent pool, flexible, React Native, mature SSR/RSC via Next.js/Remix, strong for large evolving apps | Unopinionated (many choices), hooks pitfalls, needs meta-framework for full-stack | Large/long-lived products, mixed web+mobile, hiring ease, rich ecosystem needs |
| **Angular** | Batteries-included (DI, router, forms, HttpClient, CLI), TypeScript-first, strong conventions for big teams, signals modernization | Steeper learning curve, more boilerplate/heavier | Enterprise apps with many teams needing uniform structure; teams with Java/C# backgrounds |
| **Vue** | Gentle learning curve, excellent docs, progressive adoption, Nuxt, single-file components | Smaller enterprise/talent pool than React, ecosystem fragmentation historically (Vue 2 -> 3) | Small/mid teams, incremental enhancement of server-rendered sites, fast prototyping |
| **Svelte/SvelteKit** | Compiler-based, small bundles, fast runtime, minimal boilerplate, great DX | Smaller ecosystem and hiring pool, fewer enterprise libraries | Performance-sensitive/embedded widgets, content sites, small focused teams, greenfield apps with low ecosystem needs |

(Also acknowledge alternatives: Solid, Qwik, HTMX for server-driven apps.)

Model answer: "There's no universally best choice. For a large product with web and mobile and a deep hiring need I'd choose React with Next.js. For a big enterprise team that wants enforced conventions, Angular. For a small team shipping quickly, Vue; for a lightweight, performance-critical app, Svelte. The biggest factor is usually team skills and ecosystem, not benchmarks."

**Tradeoffs**
- Differences in raw performance rarely decide outcomes; people, maintainability and hiring do.
- Switching frameworks is costly; avoid unless there's strong business justification.

**How to present this in an interview**
Be balanced and criteria-driven; show you know the trade-offs of React too (and don't bash the others). Mention you'd decide with a spike and team input.

---

<a id="q230"></a>
### Q230. Where do you see React heading (compiler, RSC, concurrent features)?

**Context / clarifying questions**
Speak from verified facts and label predictions as opinions; check the React blog for current status since details evolve.

**Diagnosis or approach (step by step)**
Key directions:
1. **React Compiler** (stable v1 released in 2025): build-time automatic memoization, reducing manual `useMemo/useCallback/memo`; relies on following the Rules of React; adopt incrementally with ESLint plugin and opt-in per directory.
2. **Server Components & Server Functions/Actions**: components that run on the server (zero client JS, direct data access), `'use client'` boundaries, streaming with Suspense; supported mainly through frameworks (Next.js App Router, others).
3. **Concurrent features**: `startTransition`, `useTransition`, `useDeferredValue`, Suspense for data, streaming SSR, selective hydration; React 19 features such as `use`, Actions, `useActionState`, `useOptimistic`, `useFormStatus`, ref as prop, document metadata support; `<Activity>` and view transitions are emerging.
4. **Framework-centric future**: React team recommends frameworks for new apps; the ecosystem moves toward server-first rendering with islands of interactivity.
5. **Ecosystem**: TanStack libraries, Signals-like fine-grained discussions, improved DevTools, better a11y/performance defaults.

**Solution with code or architecture diagram — talking points**
```tsx
// Server Component (no client JS) + client island
async function ProductPage({ id }: { id: string }) {
  const p = await db.products.get(id);          // runs on server
  return (<><ProductInfo p={p} /><AddToCart id={id} /></>);   // AddToCart is 'use client'
}
// Concurrent UX
const [isPending, startTransition] = useTransition();
startTransition(() => setQuery(next));           // keeps input responsive
```
Model answer: "I see React moving toward a server-first, framework-integrated model where the compiler removes performance boilerplate, RSC reduces shipped JS, and concurrent primitives plus Actions standardize async UI and mutations. As an engineer I'd adopt incrementally: enable the compiler on a few modules, use transitions where input lags, and use RSC through a framework when SEO/JS size matter, while keeping client-heavy apps fine as SPAs."

**Tradeoffs**
- RSC adds mental-model complexity (server/client boundary, serialization limits) and framework coupling.
- Compiler assumes pure components; legacy code with side effects in render may need cleanup.
- Concurrent features require idempotent renders and care with effects.

**How to present this in an interview**
Show you track the ecosystem, separate facts from predictions, and take a pragmatic incremental adoption stance rather than hype.
