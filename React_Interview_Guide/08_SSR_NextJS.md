# 08 – SSR & Next.js (Q121–Q133)

## Table of Contents
- [Q121. CSR vs SSR vs SSG vs ISR](#q121-csr-vs-ssr-vs-ssg-vs-isr-when-do-you-use-each)
- [Q122. React Server Components vs SSR](#q122-what-are-react-server-components-how-do-they-differ-from-ssr)
- [Q123. "use client" and "use server" boundaries](#q123-what-are-use-client-and-use-server-boundaries)
- [Q124. Server Actions](#q124-what-are-server-actions)
- [Q125. Streaming SSR and Suspense](#q125-explain-streaming-ssr-and-suspense-boundaries)
- [Q126. Next.js routing: App Router vs Pages Router](#q126-how-does-nextjs-routing-work-app-router-vs-pages-router)
- [Q127. Caching, revalidation and fetch](#q127-how-do-caching-revalidation-and-fetch-work-in-nextjs)
- [Q128. Middleware and edge runtimes](#q128-what-are-middleware-and-edge-runtimes)
- [Q129. SEO, metadata and sitemaps](#q129-how-do-you-handle-seo-metadata-and-sitemaps)
- [Q130. Auth (Auth.js, sessions, JWT)](#q130-how-do-you-handle-auth-nextauthauthjs-sessions-jwt)
- [Q131. Islands architecture; Astro vs Remix vs Next.js](#q131-what-is-the-islands-architecture-compare-astro-remix-and-nextjs)
- [Q132. Hydration performance problems](#q132-what-are-the-common-hydration-performance-problems)
- [Q133. Deploying and scaling an SSR app](#q133-how-do-you-deploy-and-scale-an-ssr-app)

---

### Q121. CSR vs SSR vs SSG vs ISR: when do you use each?

**Short answer**
CSR renders in the browser after loading a JS bundle; SSR renders HTML on the server per request; SSG renders HTML at build time; ISR is SSG that regenerates pages in the background after a time or on demand. Use CSR for authenticated dashboards, SSG/ISR for mostly-static public content, and SSR for personalized or always-fresh, SEO-relevant pages.

**Detailed explanation**
- **CSR (Client-Side Rendering):** server sends a near-empty HTML shell; JS fetches data and renders. Pros: simple hosting, rich interactivity, cheap server. Cons: slow first contentful paint, weak SEO for crawlers that don't run JS, request waterfalls. Good for internal tools and behind-login apps.
- **SSR (Server-Side Rendering):** HTML generated on each request. Pros: fresh data, per-user content, good SEO and fast first paint. Cons: TTFB depends on server and data latency, compute cost, harder to cache. Good for search results, personalized feeds, pages needing request cookies/headers.
- **SSG (Static Site Generation):** HTML built once at deploy time and served from a CDN. Fastest and cheapest; but stale until rebuilt and build times grow with page count. Good for docs, marketing pages, blogs.
- **ISR (Incremental Static Regeneration):** static pages with `revalidate` window (stale-while-revalidate) or on-demand revalidation by tag/path. Good for e-commerce catalogs and CMS content with thousands of pages.
- In the Next.js App Router these are not separate modes but outcomes of what you use: a route is static unless it uses dynamic APIs (`cookies()`, `headers()`, `searchParams`) or uncached fetches. You can mix strategies per route, even per component (Partial Prerendering).

**Code example**
```tsx
// app/products/[id]/page.tsx — ISR-style: static, regenerated hourly
export const revalidate = 3600;

export async function generateStaticParams() {
  const ids: string[] = await fetch("https://api.example.com/products/ids").then(r => r.json());
  return ids.map(id => ({ id }));
}

export default async function ProductPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const product = await fetch(`https://api.example.com/products/${id}`).then(r => r.json());
  return <h1>{product.name}</h1>;
}

// app/feed/page.tsx — SSR: dynamic because it reads cookies
import { cookies } from "next/headers";
export default async function Feed() {
  const token = (await cookies()).get("session")?.value;
  const feed = await fetch("https://api.example.com/feed", {
    headers: { Authorization: `Bearer ${token}` },
    cache: "no-store",
  }).then(r => r.json());
  return <pre>{JSON.stringify(feed, null, 2)}</pre>;
}
```

**Common pitfalls / follow-up questions**
- *Can one app mix all four?* Yes, per route (and with PPR, per section of a route).
- *Why isn't SSR always better than CSR?* It adds server cost and TTFB, and you still pay hydration cost; for logged-in apps with no SEO need, CSR is often simpler.
- *What happens to ISR on a cache miss for a not-pregenerated path?* With `dynamicParams` enabled it's rendered on demand once, then cached.

---

### Q122. What are React Server Components? How do they differ from SSR?

**Short answer**
React Server Components (RSC) are components that run only on the server (at build or request time), can access backend resources directly, and send no component JS to the browser. SSR is about producing initial HTML for *all* components (which are then hydrated); RSC is about *where components execute* and what gets shipped. They are complementary: RSC output is serialized in the RSC payload, and SSR can turn it into HTML.

**Detailed explanation**
- Server Components can be `async`, read databases/filesystem/secrets, and their dependencies (e.g. a markdown parser, date library) never reach the client bundle.
- They cannot use state, effects, refs to DOM, browser APIs, or event handlers. Those belong in Client Components.
- The server renders the tree into the *RSC payload* (a streamable format describing rendered output, plus references to Client Components and their props). The client reconciles this payload into the React tree; on navigation, only new payload is fetched and client state is preserved.
- Classic SSR hydrates every component, so all component code is shipped to the client. With RSC, only Client Components are hydrated.
- Server Components re-render only on the server; they are not hydrated, never re-render in the browser.
- Props passed from server to client components must be serializable (no functions except Server Actions, no class instances).

**Code example**
```tsx
// app/posts/page.tsx — Server Component (default in App Router)
import { db } from "@/lib/db";
import { LikeButton } from "./LikeButton";

export default async function Posts() {
  const posts = await db.post.findMany({ take: 10 });
  return (
    <ul>
      {posts.map(p => (
        <li key={p.id}>
          {p.title} <LikeButton postId={p.id} initialLikes={p.likes} />
        </li>
      ))}
    </ul>
  );
}

// app/posts/LikeButton.tsx — Client Component
"use client";
import { useState } from "react";

export function LikeButton({ postId, initialLikes }: { postId: string; initialLikes: number }) {
  const [likes, setLikes] = useState(initialLikes);
  return <button onClick={() => setLikes(l => l + 1)}>♥ {likes}</button>;
}
```

**Common pitfalls / follow-up questions**
- *Are Client Components rendered on the server?* Yes, they are still pre-rendered to HTML by SSR, then hydrated; "client" means they ship JS and can hydrate.
- *Can I import a Server Component into a Client Component?* Not directly, but you can pass it as `children`/props from a Server parent.
- *Does RSC replace SSR?* No. RSC reduces JS and enables server data access; SSR provides initial HTML.

---

### Q123. What are "use client" and "use server" boundaries?

**Short answer**
`"use client"` at the top of a file marks the boundary where the module graph switches from Server to Client Components; that file and everything it imports become part of the client bundle. `"use server"` marks functions (Server Actions) callable from the client that execute only on the server. They are opposites in purpose: one moves code to the client, the other exposes server functions via RPC.

**Detailed explanation**
- In the App Router all components are Server Components by default. Add `"use client"` only to the entry point of interactive subtrees — you don't need it in every child since imports from a client module are already client.
- Push boundaries *down* the tree (leaf interactive widgets) to keep the bundle small.
- Props crossing the boundary must be serializable. Server Components can be passed as `children` (the "donut" pattern) to keep them server-rendered inside a client wrapper (e.g. a context provider or modal).
- `"use server"` can be at the top of a file (all exports are actions) or inside an async function body. Each action becomes a public HTTP POST endpoint, so authenticate and validate input inside it.
- Use `import "server-only"` to make the build fail if a module with secrets is imported into client code; `client-only` is the reverse.
- Context providers must be Client Components; wrap them in a small file and use them in the server layout.

**Code example**
```tsx
// app/providers.tsx
"use client";
import { ThemeProvider } from "next-themes";
export function Providers({ children }: { children: React.ReactNode }) {
  return <ThemeProvider attribute="class">{children}</ThemeProvider>;
}

// app/layout.tsx — Server Component passes server-rendered children through
import { Providers } from "./providers";
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body><Providers>{children}</Providers></body>
    </html>
  );
}

// lib/secrets.ts
import "server-only";
export const apiKey = process.env.PAYMENTS_KEY!;
```

**Common pitfalls / follow-up questions**
- *Is `"use client"` only for the file it's in?* It marks the boundary; everything imported from it is also in the client graph.
- *Does `"use server"` mean "server component"?* No — common confusion. Server Components need no directive; `"use server"` is for Server Actions.
- *Why do I get "Functions cannot be passed directly to Client Components"?* The prop is a non-serializable function; use a Server Action or move logic into the client component.

---

### Q124. What are Server Actions?

**Short answer**
Server Actions are async functions marked with `"use server"` that run on the server but can be invoked from forms or client code, replacing hand-written API routes for mutations. Next.js turns them into POST endpoints with encrypted IDs and integrates them with caching, revalidation, and progressive enhancement.

**Detailed explanation**
- Used as `<form action={fn}>`, they work even before JS hydrates (progressive enhancement) and receive `FormData`.
- After mutating, call `revalidatePath`/`revalidateTag` or `redirect` to refresh the UI; the response can include updated RSC payload in the same round trip.
- React 19 hooks: `useActionState` (state + pending for the action result), `useFormStatus` (pending inside a form), `useOptimistic` (instant UI).
- Security: treat as public endpoints. Always check auth/authorization and validate inputs (e.g. Zod) in the action itself. Next.js provides CSRF mitigation via Origin/Host comparison and POST-only, but not authorization.
- Actions are queued/serialized per client, so they are not ideal for data fetching/reads; use Server Components or route handlers for reads.

**Code example**
```tsx
// app/todos/actions.ts
"use server";
import { z } from "zod";
import { revalidatePath } from "next/cache";
import { auth } from "@/auth";
import { db } from "@/lib/db";

const schema = z.object({ title: z.string().min(1).max(200) });
export type State = { error?: string } | undefined;

export async function addTodo(_prev: State, formData: FormData): Promise<State> {
  const session = await auth();
  if (!session?.user) return { error: "Unauthorized" };
  const parsed = schema.safeParse({ title: formData.get("title") });
  if (!parsed.success) return { error: parsed.error.issues[0].message };
  await db.todo.create({ data: { title: parsed.data.title, userId: session.user.id } });
  revalidatePath("/todos");
}

// app/todos/AddTodo.tsx
"use client";
import { useActionState } from "react";
import { addTodo } from "./actions";

export function AddTodo() {
  const [state, action, pending] = useActionState(addTodo, undefined);
  return (
    <form action={action}>
      <input name="title" required />
      <button disabled={pending}>{pending ? "Adding…" : "Add"}</button>
      {state?.error && <p role="alert">{state.error}</p>}
    </form>
  );
}
```

**Common pitfalls / follow-up questions**
- *Are Server Actions secure by default?* No; they're public endpoints. Authorize and validate every call.
- *Server Action vs Route Handler?* Actions for UI-triggered mutations in your own app; route handlers for public/third-party APIs, webhooks, and non-React clients.
- *How do you do optimistic updates?* `useOptimistic` to show the expected state while the action is pending; it reverts automatically on failure/completion.

---

### Q125. Explain streaming SSR and Suspense boundaries.

**Short answer**
Streaming SSR sends HTML in chunks as parts of the page become ready instead of waiting for the whole page. `<Suspense>` boundaries define those chunks: the fallback is sent immediately and the real content streams in later and replaces it, improving TTFB and perceived speed.

**Detailed explanation**
- Traditional SSR is all-or-nothing: fetch all data → render all HTML → send → hydrate all. One slow query blocks everything.
- With React 18+ (`renderToPipeableStream` / Web Streams), React flushes the shell with fallbacks, then streams completed boundaries with small inline scripts that swap them in.
- Selective hydration: React hydrates boundaries independently, prioritizing the one the user interacts with, so a heavy section doesn't block interactive parts.
- In Next.js App Router, `loading.tsx` wraps a route segment in Suspense automatically; you can add finer `<Suspense>` boundaries in components. `error.tsx` handles failures per segment.
- Place boundaries around slow, independent data; avoid too many (layout shift, flicker) and design fallbacks (skeletons) matching final dimensions.
- Caveat: status code and headers are sent with the first chunk, so a streamed page can't later change to 404/redirect (Next.js handles `notFound()` before streaming starts when thrown in the shell). Streaming also needs hosting that doesn't buffer responses.

**Code example**
```tsx
// app/dashboard/page.tsx
import { Suspense } from "react";

async function Revenue() {
  const data = await fetch("https://api.example.com/revenue", { cache: "no-store" }).then(r => r.json());
  return <p>Revenue: {data.total}</p>;
}
async function Recent() {
  const rows: { id: string; name: string }[] =
    await fetch("https://api.example.com/orders", { cache: "no-store" }).then(r => r.json());
  return <ul>{rows.map(r => <li key={r.id}>{r.name}</li>)}</ul>;
}

export default function Dashboard() {
  return (
    <main>
      <h1>Dashboard</h1> {/* sent immediately */}
      <Suspense fallback={<p>Loading revenue…</p>}><Revenue /></Suspense>
      <Suspense fallback={<p>Loading orders…</p>}><Recent /></Suspense>
    </main>
  );
}
```

**Common pitfalls / follow-up questions**
- *Why isn't my page streaming?* Data is awaited above the Suspense boundary (e.g. in the page itself), or the host/proxy buffers the response.
- *Does streaming help SEO?* Crawlers like Googlebot handle streamed HTML; critical metadata is still emitted in `<head>` early. Keep SEO-critical content outside slow boundaries.
- *Streaming vs. parallel fetching?* Start fetches in parallel (`Promise.all` or separate components) so boundaries resolve independently rather than waterfalling.

---

### Q126. How does Next.js routing work (App Router vs Pages Router)?

**Short answer**
Both are file-system routers. The Pages Router (`pages/`) maps each file to a route and uses `getServerSideProps`/`getStaticProps`; the App Router (`app/`) maps folders containing `page.tsx` to routes, uses nested layouts, Server Components by default, streaming, and `fetch`-based data loading. App Router is the recommended modern approach; both can coexist during migration.

**Detailed explanation**
- **App Router special files:** `layout.tsx` (persistent, doesn't re-render on navigation), `page.tsx`, `loading.tsx`, `error.tsx`, `not-found.tsx`, `template.tsx`, `route.ts` (API handlers), `default.tsx` (parallel routes).
- **Route features:** dynamic segments `[id]`, catch-all `[...slug]`, optional `[[...slug]]`, route groups `(marketing)` (organization without URL impact), parallel routes `@slot`, intercepting routes `(.)photo` (modals), private folders `_lib`.
- **Navigation:** `<Link>` prefetches and does client-side transitions; `useRouter`, `usePathname`, `useSearchParams` from `next/navigation` (client-only).
- **Pages Router:** `_app.tsx`, `_document.tsx`, `pages/api/*`, `getStaticPaths`; layouts are per-page conventions; whole page is hydrated; data fetched at page level only.
- **Migration:** move incrementally; `app` takes precedence if both define the same route. Routes conflicting is a build error.
- Note: in recent Next.js versions `params` and `searchParams` are Promises in server components.

**Code example**
```tsx
// app/(shop)/products/[slug]/page.tsx  →  /products/:slug
import { notFound } from "next/navigation";
import Link from "next/link";

type Props = { params: Promise<{ slug: string }> };

export default async function Product({ params }: Props) {
  const { slug } = await params;
  const res = await fetch(`https://api.example.com/products/${slug}`);
  if (res.status === 404) notFound();
  const product: { name: string } = await res.json();
  return (
    <article>
      <h1>{product.name}</h1>
      <Link href="/products">Back</Link>
    </article>
  );
}

// app/(shop)/products/[slug]/loading.tsx
export default function Loading() { return <p>Loading product…</p>; }
```

**Common pitfalls / follow-up questions**
- *Why doesn't my layout re-render on navigation?* Layouts are preserved by design; use `template.tsx` or a `key` if you need reset.
- *`useRouter` import?* In App Router it's `next/navigation`, not `next/router`.
- *Parallel vs intercepting routes?* Parallel renders multiple slots simultaneously; intercepting shows another route within the current layout (e.g. modal) while direct URL loads the full page.

---

### Q127. How do caching, revalidation and fetch work in Next.js?

**Short answer**
Next.js has several cache layers: request memoization (dedupes identical fetches within one render), the Data Cache (persistent fetch results), the Full Route Cache (prerendered HTML/RSC payload), and the client Router Cache. You control them with `fetch` options (`cache`, `next.revalidate`, `next.tags`), route segment config, and `revalidatePath`/`revalidateTag`. Defaults changed across versions, so always verify for your version.

**Detailed explanation**
- **Version caveat:** Next.js 14 cached `fetch` by default (`force-cache`). Next.js 15 changed `fetch` and GET route handlers to *uncached by default*, and the client Router Cache no longer caches page segments by default (`staleTimes` = 0). Newer releases add Cache Components / `"use cache"` directive for explicit, opt-in caching.
- **Time-based revalidation:** `fetch(url, { next: { revalidate: 60 } })` — serve stale, regenerate in background after 60s.
- **On-demand revalidation:** tag fetches `next: { tags: ['posts'] }` and call `revalidateTag('posts')` (or `revalidatePath('/posts')`) from a Server Action or webhook route handler.
- **Opting out:** `cache: 'no-store'`, `export const dynamic = 'force-dynamic'`, or using dynamic APIs like `cookies()`.
- **Non-fetch data:** for ORMs/DB clients, wrap in `unstable_cache` (or `"use cache"`) with tags, and use React `cache()` for per-request memoization.
- Self-hosted multi-instance deployments need a shared cache handler (Redis/S3) or each instance has its own cache.

**Code example**
```tsx
// lib/data.ts
import { unstable_cache } from "next/cache";
import { cache } from "react";
import { db } from "@/lib/db";

export const getPost = cache(async (id: string) => {
  const res = await fetch(`https://api.example.com/posts/${id}`, {
    next: { revalidate: 300, tags: [`post-${id}`] },
  });
  return res.json() as Promise<{ id: string; title: string }>;
});

export const getTopUsers = unstable_cache(
  async () => db.user.findMany({ orderBy: { score: "desc" }, take: 10 }),
  ["top-users"],
  { revalidate: 600, tags: ["users"] }
);

// app/api/revalidate/route.ts
import { revalidateTag } from "next/cache";
import { NextRequest, NextResponse } from "next/server";

export async function POST(req: NextRequest) {
  const { tag, secret } = await req.json();
  if (secret !== process.env.REVALIDATE_SECRET) return NextResponse.json({ ok: false }, { status: 401 });
  revalidateTag(tag);
  return NextResponse.json({ revalidated: true });
}
```

**Common pitfalls / follow-up questions**
- *Why do I see stale data?* Likely Data Cache, Full Route Cache (static route) or client Router Cache; identify which layer and revalidate with tags/paths.
- *Does `revalidatePath` clear the browser cache?* It invalidates server caches and the client Router Cache for the current session after a Server Action.
- *Request memoization vs Data Cache?* Memoization lasts one render pass and dedupes calls; Data Cache persists across requests and deployments (until revalidated).

---

### Q128. What are middleware and edge runtimes?

**Short answer**
Middleware is code that runs before a request reaches a route, letting you rewrite, redirect, set headers/cookies, or block requests. Historically it ran on the lightweight Edge Runtime (V8 isolates with Web APIs, low latency, close to users, no Node APIs); the Edge Runtime can also be selected for route handlers/pages.

**Detailed explanation**
- Defined in `middleware.ts` at project root (renamed `proxy.ts` in Next.js 16, which runs on the Node.js runtime). Use `config.matcher` to limit which paths run it.
- Typical uses: auth gating (optimistic check of cookie/JWT), locale redirects, A/B testing, geolocation, rate limiting, bot filtering, security headers.
- **Edge runtime constraints:** Web standard APIs (`fetch`, `Request`, `Response`, `crypto.subtle`), no `fs`, no most Node built-ins, limited native packages, bundle size limits, short execution time. DB drivers using TCP often don't work (use HTTP drivers).
- **Benefits:** very low cold start and global distribution. **Costs:** can't use all npm packages; data may be far from the edge so DB calls add latency; runs on every matched request, so keep it fast.
- Middleware should not be the only security layer: do authorization close to data (in server components/actions) since middleware can be bypassed by misconfiguration (e.g. CVE-2025-29927 header bypass in older versions).

**Code example**
```tsx
// middleware.ts
import { NextResponse, type NextRequest } from "next/server";
import { jwtVerify } from "jose";

const secret = new TextEncoder().encode(process.env.AUTH_SECRET);

export async function middleware(req: NextRequest) {
  const token = req.cookies.get("session")?.value;
  try {
    if (!token) throw new Error("no token");
    await jwtVerify(token, secret);
    return NextResponse.next();
  } catch {
    const url = new URL("/login", req.url);
    url.searchParams.set("from", req.nextUrl.pathname);
    return NextResponse.redirect(url);
  }
}

export const config = { matcher: ["/dashboard/:path*", "/account/:path*"] };

// app/api/geo/route.ts — opt a handler into edge
export const runtime = "edge";
export async function GET(req: Request) {
  return Response.json({ country: req.headers.get("x-vercel-ip-country") ?? "unknown" });
}
```

**Common pitfalls / follow-up questions**
- *Can middleware query a database?* On the edge, only via HTTP-based drivers; better do just a token check there.
- *Is middleware enough for auth?* No; re-verify permissions in the data layer.
- *Edge vs Node runtime choice?* Edge for low-latency, simple logic; Node for full API compatibility, heavy libs, and DB access near your region.

---

### Q129. How do you handle SEO, metadata and sitemaps?

**Short answer**
In the App Router, export a static `metadata` object or an async `generateMetadata` function per layout/page for title, description, canonical, Open Graph and Twitter tags, and add `sitemap.ts`, `robots.ts`, and structured data (JSON-LD). Rendering content on the server/static ensures crawlers get real HTML.

**Detailed explanation**
- Metadata merges from root layout down; child values override. Use `title.template` for consistent titles and `metadataBase` for absolute URLs.
- `generateMetadata` can fetch data; Next.js dedupes identical `fetch` calls with the page (use React `cache()` for ORM calls).
- Canonical URLs via `alternates.canonical`; i18n via `alternates.languages` (hreflang).
- File conventions: `app/sitemap.ts`, `app/robots.ts`, `opengraph-image.tsx` (dynamic OG images with `ImageResponse`), `icon.png`, `manifest.ts`. Large sites split sitemaps with `generateSitemaps`.
- Core Web Vitals matter: `next/image`, `next/font`, avoiding layout shift, and streaming. Use semantic HTML, correct status codes (`notFound()` yields 404), and `noindex` for private pages.
- Add JSON-LD via a `<script type="application/ld+json">` (escape `<` to prevent XSS).

**Code example**
```tsx
// app/blog/[slug]/page.tsx
import type { Metadata } from "next";
import { cache } from "react";
import { notFound } from "next/navigation";

const getPost = cache(async (slug: string) => {
  const res = await fetch(`https://api.example.com/posts/${slug}`, { next: { revalidate: 3600 } });
  return res.ok ? ((await res.json()) as { title: string; summary: string; image: string }) : null;
});

export async function generateMetadata({ params }: { params: Promise<{ slug: string }> }): Promise<Metadata> {
  const { slug } = await params;
  const post = await getPost(slug);
  if (!post) return {};
  return {
    title: post.title,
    description: post.summary,
    alternates: { canonical: `/blog/${slug}` },
    openGraph: { title: post.title, description: post.summary, images: [post.image], type: "article" },
  };
}

export default async function Page({ params }: { params: Promise<{ slug: string }> }) {
  const post = await getPost((await params).slug);
  if (!post) notFound();
  const jsonLd = { "@context": "https://schema.org", "@type": "Article", headline: post.title };
  return (
    <article>
      <script type="application/ld+json" dangerouslySetInnerHTML={{ __html: JSON.stringify(jsonLd).replace(/</g, "\\u003c") }} />
      <h1>{post.title}</h1>
    </article>
  );
}

// app/sitemap.ts
import type { MetadataRoute } from "next";
export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const posts: { slug: string; updatedAt: string }[] = await fetch("https://api.example.com/posts").then(r => r.json());
  return posts.map(p => ({ url: `https://example.com/blog/${p.slug}`, lastModified: p.updatedAt }));
}
```

**Common pitfalls / follow-up questions**
- *Can I use `<Head>` in App Router?* No (that's Pages Router `next/head`); use the Metadata API.
- *Do CSR pages rank?* Google can render JS, but it is delayed and unreliable for other crawlers/social scrapers; prefer SSR/SSG for indexable content.
- *Duplicate content?* Set canonical URLs and handle trailing slashes/query params consistently.

---

### Q130. How do you handle auth (NextAuth/Auth.js, sessions, JWT)?

**Short answer**
Use a maintained library like Auth.js (NextAuth v5) with OAuth/credentials providers, choose between database sessions (revocable, server-side state) and stateless JWT sessions (scalable, harder to revoke), store them in HttpOnly, Secure, SameSite cookies, and enforce authorization on the server at every data access point.

**Detailed explanation**
- **Auth.js v5:** one `auth.ts` exporting `auth`, `handlers`, `signIn`, `signOut`. `auth()` works in server components, route handlers, actions, and middleware.
- **JWT sessions:** signed (and encrypted in Auth.js) cookie; no DB lookup, works at the edge, but can't be revoked before expiry without a denylist/short TTL + refresh rotation. **Database sessions:** session ID in cookie, looked up via an adapter (Prisma/Drizzle); instant revocation.
- Use `callbacks.jwt` and `callbacks.session` to add `id`/`role` to the session; extend types via module augmentation.
- Never store tokens in `localStorage` (XSS risk). Use HttpOnly cookies, CSRF protection (built in), short-lived access tokens + rotating refresh tokens.
- **Authorization layers:** middleware for optimistic redirects, but verify in a Data Access Layer (DAL) close to the data; check in each Server Action. Credentials provider requires you to handle password hashing (argon2/bcrypt), rate limiting, and doesn't support database sessions by default.
- Alternatives: Clerk, Auth0, Lucia-style custom sessions, Better Auth.

**Code example**
```tsx
// auth.ts
import NextAuth from "next-auth";
import GitHub from "next-auth/providers/github";
import { PrismaAdapter } from "@auth/prisma-adapter";
import { prisma } from "@/lib/prisma";

export const { handlers, auth, signIn, signOut } = NextAuth({
  adapter: PrismaAdapter(prisma),
  session: { strategy: "jwt", maxAge: 60 * 60 * 8 },
  providers: [GitHub],
  callbacks: {
    jwt({ token, user }) {
      if (user) token.role = (user as { role?: string }).role ?? "user";
      return token;
    },
    session({ session, token }) {
      session.user.id = token.sub!;
      (session.user as { role?: string }).role = token.role as string;
      return session;
    },
  },
});

// app/api/auth/[...nextauth]/route.ts
export { handlers as GET, handlers as POST } from "@/auth";

// app/admin/page.tsx — authorize at the data layer
import { auth } from "@/auth";
import { redirect } from "next/navigation";

export default async function Admin() {
  const session = await auth();
  if (!session) redirect("/api/auth/signin");
  if ((session.user as { role?: string }).role !== "admin") redirect("/403");
  return <h1>Welcome, {session.user?.name}</h1>;
}
```

**Common pitfalls / follow-up questions**
- *JWT or database sessions?* JWT for edge/scale and simplicity; DB sessions when you need immediate revocation or server-controlled state.
- *How do you protect against CSRF/XSS?* HttpOnly + SameSite cookies, CSRF tokens (built-in), CSP, escaping output.
- *Why not rely only on middleware?* It can be bypassed/misconfigured and doesn't cover Server Actions robustly; check in each action and data fetch.

---

### Q131. What is the islands architecture? Compare Astro, Remix and Next.js.

**Short answer**
Islands architecture renders pages as static HTML and hydrates only small, isolated interactive "islands," leaving the rest as zero-JS HTML. Astro is built around it; Remix (now React Router v7 framework mode) focuses on web-standard loaders/actions and nested routes; Next.js is a full-featured React framework with RSC, multiple rendering modes and a large ecosystem.

**Detailed explanation**
- **Islands:** each interactive component is hydrated independently, with directives for timing (`client:load`, `client:idle`, `client:visible`). Great for content sites (blogs, docs, marketing) with minimal JS. Astro can use React, Vue, Svelte islands in one page. Weakness: shared state across islands requires stores; less suited to highly interactive apps.
- **Remix / React Router 7:** nested routes with parallel `loader` data loading (avoids waterfalls), `action` + `<Form>` for mutations with automatic revalidation, progressive enhancement, runs on any JS runtime (Node, Cloudflare, Deno). Embraces web fundamentals (Request/Response, HTTP caching headers). Whole page is hydrated (no RSC in stable as of writing; RSC support is in progress).
- **Next.js:** App Router with RSC, Server Actions, streaming, ISR, PPR, image/font optimization, middleware; Vercel-optimized but self-hostable. Highest feature surface and complexity; caching model can be confusing.
- **Choosing:** content-heavy/low-interactivity → Astro; data-mutation-heavy apps wanting simple mental model and portability → Remix/React Router; large product apps needing mixed rendering strategies and ecosystem → Next.js.
- RSC and islands are related ideas (minimize client JS) but RSC is a single React tree with server/client boundary rather than independent roots.

**Code example**
```tsx
// Remix / React Router 7 route: loader + action, nested-route friendly
import { Form, useLoaderData } from "react-router";
import type { Route } from "./+types/notes";

export async function loader() {
  const notes: { id: string; text: string }[] = await fetch("https://api.example.com/notes").then(r => r.json());
  return { notes };
}

export async function action({ request }: Route.ActionArgs) {
  const form = await request.formData();
  await fetch("https://api.example.com/notes", {
    method: "POST",
    body: JSON.stringify({ text: form.get("text") }),
    headers: { "Content-Type": "application/json" },
  });
  return null; // loaders revalidate automatically
}

export default function Notes() {
  const { notes } = useLoaderData<typeof loader>();
  return (
    <>
      <ul>{notes.map(n => <li key={n.id}>{n.text}</li>)}</ul>
      <Form method="post"><input name="text" /><button>Add</button></Form>
    </>
  );
}

// Astro page equivalent for an island (in .astro file): <Counter client:visible />
```

**Common pitfalls / follow-up questions**
- *Is Next.js an islands framework?* Not strictly; Server Components give similar JS reduction, but client components are in one React tree and the app still hydrates its client portions together.
- *Why choose Remix over Next?* Simpler data-flow model, web-standard APIs, deployment portability, no implicit caching.
- *When does Astro struggle?* Rich app-like UIs with heavy cross-component state.

---

### Q132. What are the common hydration performance problems?

**Short answer**
Hydration attaches event handlers to server HTML by re-running components on the client, so large bundles, many hydrated components, and expensive render work delay interactivity (poor INP/TTI) even though content is visible. Hydration mismatches also force client re-rendering and warnings. Fix by shipping less JS, hydrating selectively, and keeping server/client output identical.

**Detailed explanation**
- **Large JS bundles:** everything marked client must download, parse, execute. Fix: Server Components, code splitting with `next/dynamic`/`React.lazy`, tree-shaking, remove heavy libraries.
- **Whole-page hydration:** hydrates even static parts. Fix: RSC, Suspense boundaries for selective hydration (React 18), islands, lazy hydration on visibility.
- **Hydration mismatches:** server HTML differs from first client render (e.g. `Date.now()`, `Math.random()`, `window` checks, locale/timezone formatting, browser extensions modifying the DOM, invalid HTML nesting). Results in errors and discarded server HTML. Fix: compute in `useEffect`, use `useId`, pass values from server, `suppressHydrationWarning` for unavoidable text like timestamps.
- **Uncanny-valley UI:** visible but unresponsive. Mitigate with progressive enhancement (forms working pre-hydration), smaller islands, prioritizing above-the-fold.
- **Large serialized props/state:** big JSON payloads (`__NEXT_DATA__`/RSC payload) bloat HTML and parse time; send only needed fields.
- **Third-party scripts** blocking main thread: `next/script` with `strategy="lazyOnload"`/`worker`.
- **Long tasks:** break up with `startTransition`, defer non-critical work. Measure via React Profiler, Lighthouse, Web Vitals (INP, TBT).

**Code example**
```tsx
"use client";
import dynamic from "next/dynamic";
import { useEffect, useState } from "react";

// Heavy widget is loaded only when needed, never in the hydration critical path
const Chart = dynamic(() => import("./Chart"), {
  ssr: false,
  loading: () => <div style={{ height: 300 }} aria-busy />,
});

export function LocalTime({ iso }: { iso: string }) {
  // Avoid server/client mismatch caused by timezone differences
  const [text, setText] = useState(() => iso);
  useEffect(() => {
    setText(new Date(iso).toLocaleString());
  }, [iso]);
  return <time dateTime={iso} suppressHydrationWarning>{text}</time>;
}

export function Analytics({ show }: { show: boolean }) {
  return show ? <Chart /> : null;
}
```

**Common pitfalls / follow-up questions**
- *What causes "Text content does not match server-rendered HTML"?* Non-deterministic or environment-dependent rendering (time, random, `window`, locale).
- *Does `ssr: false` solve hydration cost?* It avoids mismatch and server work, but the component renders only on client (no SEO content, layout shift) — use sparingly and with fixed-size placeholders.
- *How does React 18/19 help?* Streaming + selective hydration, automatic batching, and improved mismatch diffs in error messages.

---

### Q133. How do you deploy and scale an SSR app?

**Short answer**
Build a production bundle, run it as a Node server (container) or on a serverless/edge platform, put a CDN in front to cache static assets and cacheable HTML, and scale horizontally with stateless instances plus a shared cache and session store. Monitor Web Vitals and server metrics, and use ISR/SSG wherever possible to reduce compute.

**Detailed explanation**
- **Deployment options:** Vercel/Netlify/Cloudflare (managed, serverless/edge), or self-host with `next start`, Docker using `output: "standalone"` on Kubernetes/ECS/Azure Container Apps/App Service.
- **Statelessness:** no in-memory sessions or uploads on instance disk; use Redis/DB for sessions, object storage (S3/Blob) for files.
- **Caching:** hashed static assets (`/_next/static`) with `immutable` long cache; CDN caching for ISR/static pages with `s-maxage` and `stale-while-revalidate`; for multi-instance self-hosting, implement a custom `cacheHandler` (Redis) so ISR and `revalidateTag` are consistent across pods.
- **Scaling:** horizontal autoscaling on CPU/latency, load balancer health checks (`/healthz`), graceful shutdown, connection pooling for DB (PgBouncer/serverless drivers), keep-alive.
- **Performance:** prefer static/ISR, stream responses, cache data fetches, compress (brotli), use image optimization/CDN loader, regional deployment close to the database.
- **Reliability/observability:** `instrumentation.ts` with OpenTelemetry, structured logging, error tracking (Sentry), rate limiting, zero-downtime rollouts (blue/green), and deployment skew protection for server actions/assets (build IDs; `deploymentId`).
- **Environment config:** `NEXT_PUBLIC_*` are inlined at build time; server secrets via runtime env/secret manager.

**Code example**
```tsx
// next.config.ts
import type { NextConfig } from "next";

const config: NextConfig = {
  output: "standalone", // small self-contained server for Docker
  cacheHandler: require.resolve("./cache-handler.js"), // Redis-backed shared cache
  cacheMaxMemorySize: 0, // disable per-instance in-memory cache so Redis is source of truth
  async headers() {
    return [
      {
        source: "/:path*",
        headers: [{ key: "X-Content-Type-Options", value: "nosniff" }],
      },
    ];
  },
};
export default config;

// app/api/healthz/route.ts
export const dynamic = "force-dynamic";
export function GET() {
  return Response.json({ status: "ok", uptime: process.uptime() });
}

/* Dockerfile (multi-stage)
FROM node:22-alpine AS build
WORKDIR /app
COPY . .
RUN npm ci && npm run build
FROM node:22-alpine
WORKDIR /app
ENV NODE_ENV=production
COPY --from=build /app/.next/standalone ./
COPY --from=build /app/.next/static ./.next/static
COPY --from=build /app/public ./public
EXPOSE 3000
CMD ["node", "server.js"]
*/
```

**Common pitfalls / follow-up questions**
- *Why does ISR behave inconsistently behind multiple instances?* Each instance has its own file/memory cache unless a shared `cacheHandler` is configured.
- *How do you reduce server cost?* Maximize static/ISR, CDN caching, cache data fetches, and avoid unnecessary dynamic APIs that opt whole routes into SSR.
- *How do you handle deployment skew?* Keep old assets available during rollout and use deployment IDs/skew protection so clients on old builds don't call incompatible server actions.
