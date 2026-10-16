# 11. Security, Accessibility & i18n (Q153–Q162)

## Table of Contents
- [Q153. XSS and dangerouslySetInnerHTML](#q153-how-does-react-protect-against-xss-what-is-dangerouslysetinnerhtml)
- [Q154. Token storage](#q154-where-should-you-store-tokens-localstorage-vs-httponly-cookies)
- [Q155. CSRF, CORS, CSP](#q155-what-are-csrf-cors-and-csp-and-how-do-they-affect-a-spa)
- [Q156. Supply-chain attacks](#q156-how-do-you-prevent-dependency-supply-chain-attacks)
- [Q157. Secrets in front-end apps](#q157-how-do-you-handle-secrets-in-front-end-apps)
- [Q158. RBAC in the UI](#q158-how-do-you-implement-rbac-in-the-ui-and-why-isnt-it-enough-on-its-own)
- [Q159. ARIA and semantic HTML](#q159-what-is-aria-what-is-semantic-html)
- [Q160. Focus management](#q160-how-do-you-manage-focus-modals-route-changes)
- [Q161. Keyboard nav and screen-reader testing](#q161-how-do-you-do-keyboard-navigation-and-screen-reader-testing)
- [Q162. i18n/l10n](#q162-how-do-you-implement-i18nl10n-react-i18next-rtl-pluralization-dates)

---

### Q153. How does React protect against XSS? What is dangerouslySetInnerHTML?

**Short answer**
React JSX escapes values interpolated into text and attributes by default, so injected strings are rendered as text, not HTML. `dangerouslySetInnerHTML` bypasses that and inserts raw HTML, so it must only receive sanitized content (e.g., via DOMPurify).

**Detailed explanation**
- Escaping covers `{userInput}` in children and attribute values; it prevents `<script>` injection via text.
- React does **not** protect against: `dangerouslySetInnerHTML`; `javascript:` URLs in `href`/`src` (`<a href={userUrl}>`); server-side rendering of unsanitized HTML in custom templates; direct DOM access (`ref.current.innerHTML`); `eval`/`new Function`; injecting user-controlled props via spread (`{...userObj}`); third-party libraries (markdown renderers with raw HTML); `style` or CSS injection.
- Defenses: sanitize with DOMPurify (allow-list), validate URL protocols (`http:`, `https:`, `mailto:`), strict CSP (no inline scripts), Trusted Types, prefer markdown renderers that don't allow raw HTML.

**Code/config example**
```tsx
import DOMPurify from 'dompurify';

export function RichText({ html }: { html: string }) {
  const clean = DOMPurify.sanitize(html, { USE_PROFILES: { html: true } });
  return <div dangerouslySetInnerHTML={{ __html: clean }} />;
}

// Safe URL helper
export function safeHref(url: string) {
  try {
    const u = new URL(url, window.location.origin);
    return ['http:', 'https:', 'mailto:'].includes(u.protocol) ? u.href : '#';
  } catch { return '#'; }
}
// <a href={safeHref(user.website)} rel="noopener noreferrer">…</a>
```

**Common pitfalls / follow-up questions**
- *Sanitizing before storage only*: sanitize at render (or both); config/library versions change.
- *Is `<a href={userInput}>` safe?* No, `javascript:` URLs execute on click.
- *Extra layer?* CSP plus Trusted Types (`require-trusted-types-for 'script'`).

---

### Q154. Where should you store tokens (localStorage vs HttpOnly cookies)?

**Short answer**
Prefer HttpOnly, Secure, SameSite cookies, because JavaScript (including injected XSS) can't read them; localStorage tokens are exfiltratable by any XSS. Cookies then require CSRF mitigations. Best-practice SPAs often use a BFF (backend-for-frontend) that holds tokens server-side and issues a session cookie.

**Detailed explanation**
- **localStorage/sessionStorage**: simple, no CSRF risk, but readable by any script on the origin → XSS = token theft (and long-lived refresh tokens are especially dangerous).
- **HttpOnly cookie**: not readable by JS; sent automatically → needs `SameSite=Lax/Strict`, CSRF token or custom header check; `Secure`; use `__Host-` prefix.
- **In-memory access token** (short-lived) + refresh token in HttpOnly cookie is a common compromise; the token is lost on reload and re-obtained silently.
- XSS can still *use* the session (make requests as the user) even with HttpOnly, so XSS prevention remains essential.
- OAuth for SPAs: Authorization Code + PKCE; never implicit flow.

**Code/config example**
```http
Set-Cookie: __Host-session=abc123; Path=/; Secure; HttpOnly; SameSite=Lax; Max-Age=3600
```
```ts
// Client: rely on cookie, no token handling
await fetch('/api/me', { credentials: 'include', headers: { 'X-Requested-With': 'XMLHttpRequest' } });
```

**Common pitfalls / follow-up questions**
- *"HttpOnly makes us safe from XSS"*: no, only from token theft.
- *Cross-site API with cookies?* Needs `SameSite=None; Secure`, CORS with credentials, and strong CSRF defenses.
- *Refresh token rotation?* Rotate on use and detect reuse to limit stolen-token impact.

---

### Q155. What are CSRF, CORS and CSP, and how do they affect a SPA?

**Short answer**
CSRF tricks a logged-in user's browser into sending an unwanted authenticated request (relevant with cookie auth). CORS is a browser mechanism that relaxes the same-origin policy for specific cross-origin reads—it is not a security control against attackers. CSP is a response header restricting what resources/scripts a page may load, limiting XSS impact.

**Detailed explanation**
- **CSRF**: mitigated by `SameSite` cookies, anti-CSRF tokens (double-submit or synchronizer), requiring custom headers/JSON content types, and checking `Origin`/`Referer`. Bearer tokens in headers aren't auto-sent, so they aren't CSRF-prone.
- **CORS**: server sends `Access-Control-Allow-Origin` (specific origin, not `*` with credentials), `-Allow-Credentials`, `-Allow-Methods/Headers`. Non-simple requests trigger a preflight `OPTIONS`. CORS errors are server config issues; proxy API under same origin in dev to avoid them. CORS doesn't stop requests being *sent* (CSRF still works for simple requests).
- **CSP**: e.g. `default-src 'self'; script-src 'self' 'nonce-…'; object-src 'none'; base-uri 'none'; frame-ancestors 'none'`. SPAs built with bundlers often avoid inline scripts; use nonces/hashes where needed; `connect-src` lists API hosts. Start with `Content-Security-Policy-Report-Only`.

**Code/config example**
```ts
// Express: CORS + CSP
import cors from 'cors';
import helmet from 'helmet';

app.use(cors({ origin: 'https://app.example.com', credentials: true }));
app.use(helmet.contentSecurityPolicy({
  directives: {
    defaultSrc: ["'self'"],
    scriptSrc: ["'self'"],
    connectSrc: ["'self'", 'https://api.example.com'],
    imgSrc: ["'self'", 'data:'],
    objectSrc: ["'none'"],
    frameAncestors: ["'none'"],
  },
}));
```

**Common pitfalls / follow-up questions**
- *`Access-Control-Allow-Origin: *` with credentials*: browsers reject it; never reflect arbitrary origins.
- *CSP breaking Vite/dev HMR or analytics*: scope dev policy separately; use nonces.
- *Does CORS prevent CSRF?* No.

---

### Q156. How do you prevent dependency supply-chain attacks?

**Short answer**
Minimize and vet dependencies, pin versions with a lockfile and install with `npm ci`, continuously scan for vulnerabilities, restrict install scripts, and protect the CI/publish pipeline. Add provenance and review automated updates before merging.

**Detailed explanation**
- **Lockfile + `npm ci`**: reproducible installs; review lockfile diffs.
- **Audit/scan**: `npm audit`, Dependabot/Renovate, Snyk, Socket, OSV-scanner; fail CI on high severity.
- **Install scripts**: `--ignore-scripts` or pnpm's `onlyBuiltDependencies` allow-list; typosquatting/dependency confusion defenses (scoped packages, registry config).
- **Reduce surface**: fewer, well-maintained packages; avoid tiny trivial packages; check maintainers, downloads, and recent releases.
- **Pipeline**: least-privilege CI tokens, 2FA for npm, publish with provenance (`npm publish --provenance`), SBOM generation, minimum release age for new versions.
- **Runtime**: CSP and Subresource Integrity (SRI) for third-party CDN scripts.

**Code/config example**
```yaml
# .github/workflows/security.yml (excerpt)
- run: npm ci --ignore-scripts
- run: npm audit --audit-level=high
- uses: actions/dependency-review-action@v4
```
```ini
# .npmrc
ignore-scripts=true
save-exact=true
```
```html
<script src="https://cdn.example.com/lib.js" integrity="sha384-…" crossorigin="anonymous"></script>
```

**Common pitfalls / follow-up questions**
- *Auto-merging all Dependabot PRs*: risk of malicious updates; use cooldowns and review.
- *Is `^` range dangerous?* Lockfile pins, but `npm install` may update; use `npm ci`.
- *Transitive dependencies?* Audit the full tree; use `overrides` to patch.

---

### Q157. How do you handle secrets in front-end apps?

**Short answer**
You can't: anything shipped to the browser is public. Keep real secrets (API keys with privileges, client secrets, DB credentials) on a server/BFF/serverless function, and expose only public identifiers; restrict public keys by domain/scope.

**Detailed explanation**
- Build-time env vars (`VITE_*`, `NEXT_PUBLIC_*`, `REACT_APP_*`) are inlined into the bundle—treat as public config.
- Proxy calls needing secrets through a backend that adds credentials; use short-lived tokens issued per user.
- Public keys (Stripe publishable key, Maps key) must be restricted (HTTP referrer, API scope, quotas).
- Prevent accidental commits: `.gitignore` for `.env`, secret scanning (GitHub push protection, gitleaks), pre-commit hooks; rotate immediately if leaked (git history is not safe).
- Source maps can reveal comments/code; never embed secrets in code even if "obfuscated".
- CI secrets stored in the platform's secret store, not repo.

**Code/config example**
```ts
// Vite: only VITE_-prefixed vars reach the client — still public
const apiBase = import.meta.env.VITE_API_URL;

// Serverless proxy (secret stays server-side)
export default async function handler(req, res) {
  const r = await fetch('https://api.vendor.com/v1/data', {
    headers: { Authorization: `Bearer ${process.env.VENDOR_SECRET}` },
  });
  res.status(r.status).json(await r.json());
}
```

**Common pitfalls / follow-up questions**
- *"We obfuscated the key"*: trivial to recover from network tab.
- *Committed a secret?* Revoke/rotate first; history rewrite is secondary.
- *Environment-specific config?* See Q168.

---

### Q158. How do you implement RBAC in the UI, and why isn't it enough on its own?

**Short answer**
In the UI, derive permissions from the authenticated user's roles/claims and use them to conditionally render routes, menus and actions via a guard component/hook. It's only UX: the client is untrusted, so the server must enforce authorization on every request.

**Detailed explanation**
- Model permissions (`can('invoice:delete')`) rather than checking role names everywhere; roles map to permissions (or use ABAC/policy libs like CASL).
- Provide permissions through context after login (from token claims or `/me` endpoint).
- Guard at: route level (redirect/403 page), component level (`<Can>`), and action level (disable vs hide, decided by UX).
- Don't trust localStorage-stored roles or JWT claims decoded on client for security; users can tamper with JS/DevTools and call APIs directly.
- Server enforces authN/authZ, object-level checks (prevent IDOR), and returns 401/403; UI handles those gracefully (e.g., global interceptor).

**Code/config example**
```tsx
const PermissionsContext = createContext<Set<string>>(new Set());
export const usePermission = (p: string) => useContext(PermissionsContext).has(p);

export function Can({ perm, children, fallback = null }: { perm: string; children: React.ReactNode; fallback?: React.ReactNode }) {
  return usePermission(perm) ? <>{children}</> : <>{fallback}</>;
}

function RequirePerm({ perm }: { perm: string }) {
  return usePermission(perm) ? <Outlet /> : <Navigate to="/403" replace />;
}

// routes: { element: <RequirePerm perm="admin:read" />, children: [{ path: 'admin', element: <Admin /> }] }
// <Can perm="invoice:delete"><DeleteButton /></Can>
```

**Common pitfalls / follow-up questions**
- *Hidden != secure*: admin bundles are still downloadable; keep sensitive logic/data server-side.
- *Stale permissions after role change?* Refresh on token renewal; handle 403 by refetching.
- *Code-split admin routes?* Reduces payload, not a security measure.

---

### Q159. What is ARIA? What is semantic HTML?

**Short answer**
Semantic HTML uses elements that convey meaning and built-in behavior (`button`, `nav`, `main`, `label`), giving accessibility for free. ARIA (Accessible Rich Internet Applications) attributes add roles, states and properties for custom widgets when native HTML can't express them. First rule of ARIA: don't use ARIA if a native element does the job.

**Detailed explanation**
- Semantic elements provide role, name, keyboard behavior, and focus handling automatically. `<div onClick>` lacks all of them.
- ARIA changes only the accessibility tree—not behavior. You must add keyboard handling and focus management yourself.
- Key attributes: `role`, `aria-label`/`aria-labelledby`/`aria-describedby` (names/descriptions), `aria-expanded`, `aria-controls`, `aria-selected`, `aria-live` (announcements), `aria-hidden`, `aria-invalid`, `aria-current`.
- Landmarks (`header`, `nav`, `main`, `footer`) and heading hierarchy enable navigation.
- In React: `htmlFor`, camelCase `aria-*` stays hyphenated; `useId` for associating labels/descriptions.
- Bad ARIA is worse than none (e.g., `role="button"` without keyboard support).

**Code/config example**
```tsx
function Disclosure({ title, children }: { title: string; children: React.ReactNode }) {
  const [open, setOpen] = useState(false);
  const id = useId();
  return (
    <section>
      <h3>
        <button aria-expanded={open} aria-controls={id} onClick={() => setOpen(o => !o)}>
          {title}
        </button>
      </h3>
      <div id={id} hidden={!open}>{children}</div>
    </section>
  );
}

// Form field with error
<label htmlFor="email">Email</label>
<input id="email" type="email" aria-invalid={!!error} aria-describedby={error ? 'email-err' : undefined} />
{error && <p id="email-err" role="alert">{error}</p>}
```

**Common pitfalls / follow-up questions**
- *`aria-label` on a div without role*: often ignored.
- *`aria-hidden` on focusable elements*: creates ghost focus; use `inert`.
- *When is ARIA needed?* Custom comboboxes, tabs, tree views, live regions—follow the WAI-ARIA Authoring Practices.

---

### Q160. How do you manage focus (modals, route changes)?

**Short answer**
Move focus deliberately when context changes: into a modal on open, trap it inside while open, and restore it to the trigger on close. On SPA route changes, move focus to the new page's heading or main container and announce the change, since browsers don't do it automatically.

**Detailed explanation**
- **Modals**: prefer native `<dialog>` with `showModal()` (built-in focus trap, Esc, inert background, top layer) or a vetted library (Radix, React Aria, Headless UI). Requirements: `role="dialog"`, `aria-modal`, accessible name, Esc closes, focus returns to the trigger.
- **Route changes**: focus an `<h1 tabIndex={-1}>` or the `<main>` after navigation; update `document.title`; optionally use an `aria-live` region. Provide a "skip to content" link.
- **Dynamic content**: after deleting an item, move focus to a logical neighbor; for inline errors, focus the first invalid field or an error summary.
- Never remove focus outlines without replacement; use `:focus-visible`.
- Use `inert` on background content.

**Code/config example**
```tsx
function Modal({ open, onClose, title, children }: Props) {
  const ref = useRef<HTMLDialogElement>(null);
  useEffect(() => {
    const d = ref.current!;
    if (open && !d.open) d.showModal();   // traps focus, inerts background
    if (!open && d.open) d.close();
  }, [open]);
  return (
    <dialog ref={ref} aria-labelledby="dlg-title" onClose={onClose}>
      <h2 id="dlg-title">{title}</h2>
      {children}
      <button onClick={onClose}>Close</button>
    </dialog>
  ); // browser restores focus to the previously focused element on close
}

// Route-change focus
function RouteFocus() {
  const { pathname } = useLocation();
  const h1 = useRef<HTMLHeadingElement>(null);
  useEffect(() => { document.querySelector<HTMLElement>('h1')?.focus(); }, [pathname]);
  return null;
}
// <h1 tabIndex={-1}>Page title</h1>
```

**Common pitfalls / follow-up questions**
- *Focus lost to `<body>` after content removal*: manually move focus.
- *Positive `tabIndex`*: avoid; use `0`/`-1` only.
- *Custom trap without Esc/restore*: keyboard users get stuck; use proven libs.

---

### Q161. How do you do keyboard navigation and screen-reader testing?

**Short answer**
Unplug the mouse and verify every interactive element is reachable with Tab, operable with Enter/Space/Arrows/Esc, has visible focus and logical order; then test with real screen readers (NVDA/JAWS on Windows, VoiceOver on macOS/iOS, TalkBack) to confirm names, roles, states and announcements. Automate what's possible (axe, testing-library) but treat manual checks as essential.

**Detailed explanation**
- **Keyboard checklist**: tab order matches visual order; no traps; custom widgets follow APA patterns (roving tabindex/`aria-activedescendant` for lists, tabs, menus); skip links; Esc closes overlays; focus visible.
- **Screen readers**: check headings/landmark navigation, form label announcements, live-region updates (toasts, loading states), dynamic content, table semantics, image alt text. Test at least NVDA+Firefox/Chrome and VoiceOver+Safari.
- **Automation**: `userEvent.tab()` + `toHaveFocus()` in unit tests; Playwright keyboard APIs; axe; Playwright ARIA snapshots (`toMatchAriaSnapshot`) assert the accessibility tree.
- Include a11y in the definition of done and test with users with disabilities when possible.

**Code/config example**
```tsx
test('menu is keyboard operable', async () => {
  const user = userEvent.setup();
  render(<Menu />);
  await user.tab();
  expect(screen.getByRole('button', { name: /actions/i })).toHaveFocus();
  await user.keyboard('{Enter}');
  const items = screen.getAllByRole('menuitem');
  expect(items[0]).toHaveFocus();
  await user.keyboard('{ArrowDown}');
  expect(items[1]).toHaveFocus();
  await user.keyboard('{Escape}');
  expect(screen.getByRole('button', { name: /actions/i })).toHaveFocus();
});
```

**Common pitfalls / follow-up questions**
- *Only testing with automation*: misses reading order, announcement quality.
- *Live regions not announcing*: region must exist in DOM before content changes.
- *Roving tabindex vs activedescendant?* Roving moves real focus; activedescendant keeps focus on container (comboboxes).

---

### Q162. How do you implement i18n/l10n (react-i18next, RTL, pluralization, dates)?

**Short answer**
Use a library such as react-i18next (or FormatJS/react-intl) with translation resource files per locale, a `useTranslation` hook, ICU-style pluralization, and `Intl` APIs for dates/numbers/currency. Support RTL by setting `dir` and `lang` on `<html>` and using CSS logical properties; lazy-load locale bundles.

**Detailed explanation**
- **Setup**: `i18next` + `react-i18next`; namespaces per feature, language detector, fallback language, Suspense-based lazy loading via backend plugin.
- **Plurals**: i18next uses `key_one`, `key_other` (CLDR categories; Arabic has six). Never concatenate strings; use interpolation and `<Trans>` for embedded markup.
- **Formatting**: `Intl.DateTimeFormat`, `Intl.NumberFormat`, `Intl.RelativeTimeFormat`, `Intl.PluralRules`, `Intl.ListFormat`; store dates in UTC/ISO and format by user's locale/timezone.
- **RTL**: `document.documentElement.dir = i18n.dir()`; use logical CSS (`margin-inline-start`, `padding-inline`, `text-align: start`), mirror directional icons, use `dir="auto"` for user content.
- **Process**: extract strings, translation management system, pseudo-localization to find hard-coded strings and layout overflow, allow text expansion (~30%), avoid text in images.

**Code/config example**
```ts
// i18n.ts
import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';
import HttpBackend from 'i18next-http-backend';
import LanguageDetector from 'i18next-browser-languagedetector';

i18n.use(HttpBackend).use(LanguageDetector).use(initReactI18next).init({
  fallbackLng: 'en',
  supportedLngs: ['en', 'ar', 'de'],
  ns: ['common'],
  defaultNS: 'common',
  interpolation: { escapeValue: false }, // React already escapes
});
i18n.on('languageChanged', lng => {
  document.documentElement.lang = lng;
  document.documentElement.dir = i18n.dir(lng);
});
export default i18n;
```
```json
// public/locales/en/common.json
{ "cart_one": "{{count}} item in cart", "cart_other": "{{count}} items in cart" }
```
```tsx
const { t, i18n } = useTranslation();
<p>{t('cart', { count: items.length })}</p>
<time>{new Intl.DateTimeFormat(i18n.language, { dateStyle: 'medium', timeZone: user.tz }).format(date)}</time>
<span>{new Intl.NumberFormat(i18n.language, { style: 'currency', currency: 'EUR' }).format(total)}</span>
```

**Common pitfalls / follow-up questions**
- *String concatenation* (`t('hello') + name`) breaks word order in other languages; use interpolation.
- *Hard-coded `left/right` CSS*: breaks RTL; use logical properties.
- *Locale vs language vs region?* `en-US` vs `en-GB` differ in date/number formats; keep locale for formatting, translations may share a language file.
