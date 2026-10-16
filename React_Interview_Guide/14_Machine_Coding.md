# 14. Machine Coding Round (Q188–Q200)

All solutions are single-file React 18/19 function components (`App.jsx` + optional `styles.css`), runnable in Vite/CRA/CodeSandbox with no extra dependencies. Mock APIs are included so they run offline.

## Table of Contents
- [Q188. Debounced search/autocomplete with cancellation](#q188-build-a-debounced-searchautocomplete-with-cancellation)
- [Q189. Infinite scroll / virtualized list](#q189-build-an-infinite-scroll-or-virtualized-list)
- [Q190. Modal/toast system with portals](#q190-build-a-modaltoast-system-with-portals)
- [Q191. Star rating and tabs](#q191-build-a-star-rating-and-a-tabs-component)
- [Q192. Multi-step form with validation](#q192-build-a-multi-step-form-with-validation)
- [Q193. Todo app with undo/redo](#q193-build-a-todo-app-with-undoredo)
- [Q194. Drag-and-drop Kanban board](#q194-build-a-drag-and-drop-kanban-board)
- [Q195. Data table: sort, filter, paginate](#q195-build-a-data-table-with-sorting-filtering-and-pagination)
- [Q196. Custom useFetch with caching](#q196-build-a-custom-usefetch-with-caching)
- [Q197. Tic-Tac-Toe or calculator](#q197-build-a-tic-tac-toe-game-or-a-calculator)
- [Q198. Accordion and carousel](#q198-build-an-accordion-and-a-carousel)
- [Q199. Chat UI with optimistic messages and reconnects](#q199-build-a-chat-ui-with-optimistic-messages-and-reconnects)
- [Q200. Theme switcher with context and persistence](#q200-build-a-theme-switcher-with-context-and-persistence)

---

### Q188. Build a debounced search/autocomplete with cancellation.

**Requirements**
- Input searches a (mock) API after the user pauses typing (300 ms).
- Stale requests must be cancelled; out-of-order responses must never overwrite newer results.
- Loading, empty and error states; keyboard navigation (↑ ↓ Enter Esc); ARIA combobox semantics.

**Approach / design thinking**
- Split the *immediate* input value from the *debounced* value (`useDebounce`).
- One `useEffect` keyed on the debounced query creates an `AbortController`; the cleanup aborts it — this handles both cancellation and race conditions.
- API accepts an `AbortSignal` and rejects with `AbortError`, which we ignore.
- Keyboard handling uses an `activeIndex`; the listbox uses `role="listbox"` and `aria-activedescendant`.

**Complete working solution**

```jsx
import { useEffect, useRef, useState } from 'react';

const DATA = ['Apple', 'Apricot', 'Avocado', 'Banana', 'Blueberry', 'Cherry', 'Cranberry', 'Date', 'Fig',
  'Grape', 'Grapefruit', 'Guava', 'Kiwi', 'Lemon', 'Lime', 'Mango', 'Orange', 'Papaya', 'Peach', 'Pear', 'Plum'];

// Mock API: random latency so responses can arrive out of order; honours AbortSignal.
function searchApi(query, signal) {
  return new Promise((resolve, reject) => {
    const timer = setTimeout(() => {
      resolve(DATA.filter(d => d.toLowerCase().includes(query.toLowerCase())));
    }, 200 + Math.random() * 600);
    signal.addEventListener('abort', () => {
      clearTimeout(timer);
      reject(new DOMException('Aborted', 'AbortError'));
    });
  });
}

function useDebounce(value, delay) {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id);
  }, [value, delay]);
  return debounced;
}

export default function App() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [status, setStatus] = useState('idle'); // idle | loading | success | error
  const [open, setOpen] = useState(false);
  const [active, setActive] = useState(-1);
  const debounced = useDebounce(query.trim(), 300);
  const wrapRef = useRef(null);

  useEffect(() => {
    if (!debounced) {
      setResults([]);
      setStatus('idle');
      return;
    }
    const controller = new AbortController();
    setStatus('loading');
    searchApi(debounced, controller.signal)
      .then(data => {
        setResults(data);
        setStatus('success');
        setActive(-1);
      })
      .catch(err => {
        if (err.name !== 'AbortError') setStatus('error');
      });
    return () => controller.abort(); // cancels stale request when query changes/unmounts
  }, [debounced]);

  useEffect(() => {
    const onDocClick = e => {
      if (wrapRef.current && !wrapRef.current.contains(e.target)) setOpen(false);
    };
    document.addEventListener('mousedown', onDocClick);
    return () => document.removeEventListener('mousedown', onDocClick);
  }, []);

  const select = item => {
    setQuery(item);
    setOpen(false);
  };

  const onKeyDown = e => {
    if (e.key === 'ArrowDown') {
      e.preventDefault();
      setOpen(true);
      setActive(i => (results.length ? (i + 1) % results.length : -1));
    } else if (e.key === 'ArrowUp') {
      e.preventDefault();
      setActive(i => (results.length ? (i <= 0 ? results.length - 1 : i - 1) : -1));
    } else if (e.key === 'Enter' && open && active >= 0) {
      e.preventDefault();
      select(results[active]);
    } else if (e.key === 'Escape') {
      setOpen(false);
    }
  };

  const showList = open && debounced && status !== 'idle';

  return (
    <div className="ac" ref={wrapRef}>
      <input
        role="combobox"
        aria-expanded={!!showList}
        aria-controls="ac-list"
        aria-autocomplete="list"
        aria-activedescendant={active >= 0 ? `ac-opt-${active}` : undefined}
        placeholder="Search fruits…"
        value={query}
        onChange={e => { setQuery(e.target.value); setOpen(true); }}
        onFocus={() => setOpen(true)}
        onKeyDown={onKeyDown}
      />
      {showList && (
        <ul id="ac-list" role="listbox" className="ac-list">
          {status === 'loading' && <li className="ac-msg">Loading…</li>}
          {status === 'error' && <li className="ac-msg">Something went wrong</li>}
          {status === 'success' && results.length === 0 && <li className="ac-msg">No results</li>}
          {status === 'success' && results.map((r, i) => (
            <li
              key={r}
              id={`ac-opt-${i}`}
              role="option"
              aria-selected={i === active}
              className={i === active ? 'ac-opt active' : 'ac-opt'}
              onMouseDown={e => e.preventDefault()}
              onClick={() => select(r)}
              onMouseEnter={() => setActive(i)}
            >
              {r}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

```css
.ac { position: relative; width: 320px; margin: 40px auto; font-family: sans-serif; }
.ac input { width: 100%; padding: 8px 10px; box-sizing: border-box; }
.ac-list { position: absolute; left: 0; right: 0; margin: 2px 0 0; padding: 0; list-style: none;
  background: #fff; border: 1px solid #ccc; max-height: 220px; overflow: auto; z-index: 10; }
.ac-opt, .ac-msg { padding: 8px 10px; }
.ac-opt { cursor: pointer; }
.ac-opt.active { background: #e8f0fe; }
.ac-msg { color: #666; }
```

**Edge cases & improvements**
- Empty/whitespace query clears results; aborting on unmount avoids "set state on unmounted" issues.
- Race condition: handled because cleanup aborts the previous request; even without abort support use an `ignore` flag.
- Add an in-memory cache (`Map<query, results>`), min length (2 chars), highlight matched text, and `useDeferredValue`/`useTransition` for heavy lists.
- Real APIs: `fetch(url, { signal })` works identically.

**Follow-up questions**
- Debounce vs throttle? Where would you use each?
- Why not debounce inside the `onChange` handler with a `useCallback`? (Stale closures, recreated timers; the effect-based approach is simpler.)
- How do you test it? (Jest fake timers + mock `searchApi`.)
- How would you handle IME composition events?

---

### Q189. Build an infinite scroll or virtualized list.

**Requirements**
- Infinite scroll: load pages of data when the user nears the bottom, with loading/end states.
- Virtualized list: render only visible rows for 10,000 fixed-height items with smooth scrolling.

**Approach / design thinking**
- Infinite scroll uses `IntersectionObserver` on a sentinel element (no scroll listeners, no throttling).
- Virtualization: container with fixed height and `overflow:auto`; an inner spacer of `total * rowHeight` px; render only items in `[start, end)` computed from `scrollTop`, with overscan; absolutely/transform-position the window.
- Guard against duplicate page fetches with a `loading` ref.

**Complete working solution**

```jsx
import { useCallback, useEffect, useRef, useState } from 'react';

/* ---------- Part A: Infinite scroll ---------- */
const PAGE_SIZE = 20;
const TOTAL = 100;

function fetchPage(page) {
  return new Promise(resolve =>
    setTimeout(() => {
      const start = page * PAGE_SIZE;
      const items = Array.from({ length: Math.max(0, Math.min(PAGE_SIZE, TOTAL - start)) }, (_, i) => ({
        id: start + i, title: `Item #${start + i + 1}`,
      }));
      resolve({ items, hasMore: start + PAGE_SIZE < TOTAL });
    }, 500)
  );
}

function InfiniteList() {
  const [items, setItems] = useState([]);
  const [page, setPage] = useState(0);
  const [hasMore, setHasMore] = useState(true);
  const [loading, setLoading] = useState(false);
  const loadingRef = useRef(false);
  const sentinelRef = useRef(null);

  const loadMore = useCallback(async () => {
    if (loadingRef.current) return;
    loadingRef.current = true;
    setLoading(true);
    const { items: next, hasMore: more } = await fetchPage(page);
    setItems(prev => [...prev, ...next]);
    setHasMore(more);
    setPage(p => p + 1);
    loadingRef.current = false;
    setLoading(false);
  }, [page]);

  useEffect(() => {
    const el = sentinelRef.current;
    if (!el || !hasMore) return;
    const io = new IntersectionObserver(
      entries => { if (entries[0].isIntersecting) loadMore(); },
      { rootMargin: '100px' }
    );
    io.observe(el);
    return () => io.disconnect();
  }, [loadMore, hasMore]);

  return (
    <div className="box">
      <ul>{items.map(i => <li key={i.id} className="row">{i.title}</li>)}</ul>
      {hasMore ? <div ref={sentinelRef} className="status">{loading ? 'Loading…' : ' '}</div>
               : <div className="status">— End —</div>}
    </div>
  );
}

/* ---------- Part B: Virtualized list ---------- */
const ROW_H = 36;
const VIEW_H = 360;
const OVERSCAN = 5;

function VirtualList({ count = 10000 }) {
  const [scrollTop, setScrollTop] = useState(0);
  const start = Math.max(0, Math.floor(scrollTop / ROW_H) - OVERSCAN);
  const end = Math.min(count, Math.ceil((scrollTop + VIEW_H) / ROW_H) + OVERSCAN);
  const rows = [];
  for (let i = start; i < end; i++) {
    rows.push(
      <div key={i} className="vrow" style={{ top: i * ROW_H, height: ROW_H }}>
        Row {i + 1}
      </div>
    );
  }
  return (
    <div className="viewport" style={{ height: VIEW_H }} onScroll={e => setScrollTop(e.currentTarget.scrollTop)}>
      <div style={{ height: count * ROW_H, position: 'relative' }}>{rows}</div>
    </div>
  );
}

export default function App() {
  const [tab, setTab] = useState('infinite');
  return (
    <div className="wrap">
      <button onClick={() => setTab('infinite')}>Infinite scroll</button>
      <button onClick={() => setTab('virtual')}>Virtualized (10k)</button>
      {tab === 'infinite' ? <InfiniteList /> : <VirtualList />}
    </div>
  );
}
```

```css
.wrap { width: 360px; margin: 20px auto; font-family: sans-serif; }
.box { height: 360px; overflow: auto; border: 1px solid #ccc; margin-top: 10px; }
.box ul { margin: 0; padding: 0; list-style: none; }
.row { padding: 10px; border-bottom: 1px solid #eee; }
.status { text-align: center; padding: 12px; color: #666; min-height: 20px; }
.viewport { overflow: auto; border: 1px solid #ccc; margin-top: 10px; }
.vrow { position: absolute; left: 0; right: 0; box-sizing: border-box; padding: 8px 10px;
  border-bottom: 1px solid #eee; }
```

**Edge cases & improvements**
- Because the sentinel stays in view after a short page, the effect re-subscribes after each load (deps change) so it fires again — avoiding a stall.
- Error + retry button for failed page loads; abort on unmount; dedupe items by id.
- Variable-height rows: measure with `ResizeObserver` and keep a prefix-sum/offset cache (binary search to find start index); or use `react-window`/`@tanstack/react-virtual`.
- Preserve scroll position when prepending (chat) by adjusting `scrollTop` by the height delta; use `will-change: transform` + `transform: translateY` for smoother painting.
- Accessibility: `aria-setsize`/`aria-posinset`; keyboard focus handling in virtualized lists is hard.

**Follow-up questions**
- Why IntersectionObserver instead of scroll events?
- What is the complexity of rendering with/without virtualization?
- How would you virtualize a grid or a table with sticky headers?
- Windowing vs pagination UX/SEO trade-offs?

---

### Q190. Build a modal/toast system with portals.

**Requirements**
- `useModal`/`useToast` available app-wide via context; modals and toasts render into `document.body`, not inside the component tree's DOM.
- Modal: overlay, close on Esc and backdrop click, focus trap, restore focus, scroll lock, `role="dialog" aria-modal`.
- Toasts: stack, auto-dismiss (with timer cleanup), manual dismiss, `aria-live` region.

**Approach / design thinking**
- `createPortal(children, document.body)` escapes `overflow:hidden`/z-index stacking contexts while keeping React event bubbling through the React tree.
- Toast state = array in a provider; each Toast owns its timeout in an effect.
- Modal focus management: remember `document.activeElement`, focus the dialog on open, trap Tab, restore on close.

**Complete working solution**

```jsx
import { createContext, useCallback, useContext, useEffect, useMemo, useRef, useState } from 'react';
import { createPortal } from 'react-dom';

/* ---------- Modal ---------- */
function Modal({ open, onClose, title, children }) {
  const dialogRef = useRef(null);

  useEffect(() => {
    if (!open) return;
    const previouslyFocused = document.activeElement;
    const prevOverflow = document.body.style.overflow;
    document.body.style.overflow = 'hidden';
    dialogRef.current?.focus();

    const onKey = e => {
      if (e.key === 'Escape') { onClose(); return; }
      if (e.key !== 'Tab') return;
      const focusables = dialogRef.current.querySelectorAll(
        'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
      );
      if (!focusables.length) { e.preventDefault(); return; }
      const first = focusables[0];
      const last = focusables[focusables.length - 1];
      if (e.shiftKey && (document.activeElement === first || document.activeElement === dialogRef.current)) {
        e.preventDefault(); last.focus();
      } else if (!e.shiftKey && document.activeElement === last) {
        e.preventDefault(); first.focus();
      }
    };
    document.addEventListener('keydown', onKey);
    return () => {
      document.removeEventListener('keydown', onKey);
      document.body.style.overflow = prevOverflow;
      previouslyFocused?.focus?.();
    };
  }, [open, onClose]);

  if (!open) return null;
  return createPortal(
    <div className="overlay" onMouseDown={e => { if (e.target === e.currentTarget) onClose(); }}>
      <div className="dialog" role="dialog" aria-modal="true" aria-label={title} tabIndex={-1} ref={dialogRef}>
        <h2>{title}</h2>
        {children}
        <button onClick={onClose}>Close</button>
      </div>
    </div>,
    document.body
  );
}

/* ---------- Toasts ---------- */
const ToastContext = createContext(null);
let nextId = 1;

function Toast({ toast, onDismiss }) {
  useEffect(() => {
    if (!toast.duration) return;
    const id = setTimeout(() => onDismiss(toast.id), toast.duration);
    return () => clearTimeout(id);
  }, [toast.id, toast.duration, onDismiss]);
  return (
    <div className={`toast ${toast.type}`} role="status">
      <span>{toast.message}</span>
      <button aria-label="Dismiss" onClick={() => onDismiss(toast.id)}>×</button>
    </div>
  );
}

function ToastProvider({ children }) {
  const [toasts, setToasts] = useState([]);
  const dismiss = useCallback(id => setToasts(t => t.filter(x => x.id !== id)), []);
  const show = useCallback((message, { type = 'info', duration = 3000 } = {}) => {
    const id = nextId++;
    setToasts(t => [...t, { id, message, type, duration }]);
    return id;
  }, []);
  const value = useMemo(() => ({ show, dismiss }), [show, dismiss]);

  return (
    <ToastContext.Provider value={value}>
      {children}
      {createPortal(
        <div className="toast-region" aria-live="polite">
          {toasts.map(t => <Toast key={t.id} toast={t} onDismiss={dismiss} />)}
        </div>,
        document.body
      )}
    </ToastContext.Provider>
  );
}

function useToast() {
  const ctx = useContext(ToastContext);
  if (!ctx) throw new Error('useToast must be used inside <ToastProvider>');
  return ctx;
}

/* ---------- Demo ---------- */
function Demo() {
  const [open, setOpen] = useState(false);
  const close = useCallback(() => setOpen(false), []);
  const toast = useToast();
  return (
    <div style={{ padding: 24, overflow: 'hidden', height: 120, border: '1px solid #ccc' }}>
      <button onClick={() => setOpen(true)}>Open modal</button>{' '}
      <button onClick={() => toast.show('Saved!', { type: 'success' })}>Success toast</button>{' '}
      <button onClick={() => toast.show('Failed!', { type: 'error', duration: 0 })}>Sticky error</button>
      <Modal open={open} onClose={close} title="Confirm">
        <p>Are you sure?</p>
        <input placeholder="Focus trapped inside" />
        <button onClick={() => { toast.show('Confirmed'); close(); }}>OK</button>
      </Modal>
    </div>
  );
}

export default function App() {
  return <ToastProvider><Demo /></ToastProvider>;
}
```

```css
.overlay { position: fixed; inset: 0; background: rgba(0,0,0,.5); display: grid; place-items: center; z-index: 1000; }
.dialog { background: #fff; padding: 20px; border-radius: 8px; min-width: 300px; outline: none; }
.toast-region { position: fixed; right: 16px; bottom: 16px; display: flex; flex-direction: column; gap: 8px; z-index: 2000; }
.toast { display: flex; gap: 12px; align-items: center; justify-content: space-between; padding: 10px 14px;
  border-radius: 6px; color: #fff; background: #333; min-width: 220px; }
.toast.success { background: #2e7d32; }
.toast.error { background: #c62828; }
.toast button { background: none; border: 0; color: inherit; font-size: 18px; cursor: pointer; }
```

**Edge cases & improvements**
- Effect deps include `onClose`; the demo passes a stable `useCallback`, otherwise the effect would re-run (and refocus) each render — store `onClose` in a ref to be safe.
- Nested modals: maintain a stack and only the top handles Esc; use `inert` on the background or `aria-hidden`.
- Toast pause-on-hover, max stack, animations (`transition` + exit state), dedupe, queueing, imperative API outside React.
- Use the native `<dialog>` element with `showModal()` for built-in focus trapping.
- SSR: `document` is undefined — render the portal only after mount.

**Follow-up questions**
- Do events from a portal bubble to React ancestors? (Yes, through the React tree.)
- Why `aria-live`? How would you avoid reading every toast twice?
- How do you stop background scroll on iOS Safari?

---

### Q191. Build a star rating and a tabs component.

**Requirements**
- Star rating: N stars, hover preview, click to set, click same star to clear (optional), read-only mode, keyboard accessible (arrow keys), controlled/uncontrolled.
- Tabs: accessible tablist with arrow-key/Home/End navigation, panels linked with ARIA, composable API.

**Approach / design thinking**
- Rating: derive display value `hover ?? value`; use `role="radiogroup"` with radio-like buttons using roving tabindex.
- Tabs: compound components with context (`Tabs`, `TabList`, `Tab`, `TabPanel`); roving tabindex; automatic activation on focus.

**Complete working solution**

```jsx
import { createContext, useContext, useId, useRef, useState } from 'react';

/* ---------- StarRating ---------- */
function StarRating({ value: controlled, defaultValue = 0, max = 5, readOnly = false, onChange }) {
  const [inner, setInner] = useState(defaultValue);
  const [hover, setHover] = useState(null);
  const value = controlled ?? inner;
  const display = hover ?? value;

  const set = v => {
    if (readOnly) return;
    if (controlled === undefined) setInner(v);
    onChange?.(v);
  };

  const onKeyDown = e => {
    if (readOnly) return;
    if (e.key === 'ArrowRight' || e.key === 'ArrowUp') { e.preventDefault(); set(Math.min(max, value + 1)); }
    if (e.key === 'ArrowLeft' || e.key === 'ArrowDown') { e.preventDefault(); set(Math.max(0, value - 1)); }
    if (e.key === 'Home') { e.preventDefault(); set(0); }
    if (e.key === 'End') { e.preventDefault(); set(max); }
  };

  return (
    <div
      role="radiogroup"
      aria-label="Rating"
      aria-readonly={readOnly}
      className="stars"
      onMouseLeave={() => setHover(null)}
      onKeyDown={onKeyDown}
    >
      {Array.from({ length: max }, (_, i) => {
        const n = i + 1;
        return (
          <button
            key={n}
            type="button"
            role="radio"
            aria-checked={value === n}
            aria-label={`${n} star${n > 1 ? 's' : ''}`}
            tabIndex={readOnly ? -1 : (value === n || (value === 0 && n === 1)) ? 0 : -1}
            className={n <= display ? 'star on' : 'star'}
            disabled={readOnly}
            onMouseEnter={() => !readOnly && setHover(n)}
            onClick={() => set(n === value ? 0 : n)}
          >
            ★
          </button>
        );
      })}
    </div>
  );
}

/* ---------- Tabs (compound) ---------- */
const TabsContext = createContext(null);

function Tabs({ defaultValue, children }) {
  const [value, setValue] = useState(defaultValue);
  const baseId = useId();
  return <TabsContext.Provider value={{ value, setValue, baseId }}>{children}</TabsContext.Provider>;
}

function TabList({ children, label }) {
  const ref = useRef(null);
  const onKeyDown = e => {
    const tabs = [...ref.current.querySelectorAll('[role="tab"]:not([disabled])')];
    const idx = tabs.indexOf(document.activeElement);
    if (idx < 0) return;
    let next = null;
    if (e.key === 'ArrowRight') next = tabs[(idx + 1) % tabs.length];
    else if (e.key === 'ArrowLeft') next = tabs[(idx - 1 + tabs.length) % tabs.length];
    else if (e.key === 'Home') next = tabs[0];
    else if (e.key === 'End') next = tabs[tabs.length - 1];
    if (next) { e.preventDefault(); next.focus(); next.click(); }
  };
  return <div role="tablist" aria-label={label} ref={ref} onKeyDown={onKeyDown} className="tablist">{children}</div>;
}

function Tab({ value, children, disabled }) {
  const { value: active, setValue, baseId } = useContext(TabsContext);
  const selected = active === value;
  return (
    <button
      role="tab"
      id={`${baseId}-tab-${value}`}
      aria-selected={selected}
      aria-controls={`${baseId}-panel-${value}`}
      tabIndex={selected ? 0 : -1}
      disabled={disabled}
      className={selected ? 'tab active' : 'tab'}
      onClick={() => setValue(value)}
    >
      {children}
    </button>
  );
}

function TabPanel({ value, children }) {
  const { value: active, baseId } = useContext(TabsContext);
  return (
    <div role="tabpanel" id={`${baseId}-panel-${value}`} aria-labelledby={`${baseId}-tab-${value}`}
         hidden={active !== value} tabIndex={0} className="panel">
      {active === value && children}
    </div>
  );
}

export default function App() {
  const [rating, setRating] = useState(3);
  return (
    <div style={{ width: 420, margin: '30px auto', fontFamily: 'sans-serif' }}>
      <h3>Rating: {rating}</h3>
      <StarRating value={rating} onChange={setRating} />
      <h4>Read-only</h4>
      <StarRating value={4} readOnly />

      <h3>Tabs</h3>
      <Tabs defaultValue="a">
        <TabList label="Demo tabs">
          <Tab value="a">Profile</Tab>
          <Tab value="b">Settings</Tab>
          <Tab value="c" disabled>Disabled</Tab>
          <Tab value="d">About</Tab>
        </TabList>
        <TabPanel value="a">Profile content</TabPanel>
        <TabPanel value="b">Settings content</TabPanel>
        <TabPanel value="d">About content</TabPanel>
      </Tabs>
    </div>
  );
}
```

```css
.stars { display: inline-flex; }
.star { background: none; border: 0; font-size: 28px; color: #ccc; cursor: pointer; padding: 0 2px; }
.star.on { color: #f5b301; }
.star:disabled { cursor: default; }
.tablist { display: flex; border-bottom: 1px solid #ccc; }
.tab { background: none; border: 0; padding: 8px 14px; cursor: pointer; border-bottom: 2px solid transparent; }
.tab.active { border-bottom-color: #1976d2; color: #1976d2; }
.tab:disabled { color: #aaa; cursor: not-allowed; }
.panel { padding: 12px; }
```

**Edge cases & improvements**
- Half-star support: compute from `e.nativeEvent.offsetX / width`; fractional values and `aria-valuenow` using `role="slider"`.
- Controlled vs uncontrolled pattern: `value ?? inner` as shown (don't switch modes mid-life).
- Tabs: lazy mount vs keep mounted (`hidden`), vertical orientation (`aria-orientation`, Up/Down keys), manual activation mode, URL-synced tab (`?tab=`), skip disabled tabs (done via selector).
- Use `useId` so multiple instances don't collide.

**Follow-up questions**
- What is roving tabindex and why use it?
- Compound components vs render props vs config-array API?
- How would you make the tab list overflow-scrollable?

---

### Q192. Build a multi-step form with validation.

**Requirements**
- 3 steps: Account (name, email, password) → Address (city, ZIP) → Review/Submit.
- Validate the current step before moving next; show inline errors after a field is touched; Back keeps entered data.
- Progress indicator; final submit with loading + success state; accessible error messaging.

**Approach / design thinking**
- Single state object for all data (lifted to the wizard), one `validators` map per step (pure functions → easy to test).
- `errors` are derived from `data` (no duplicated state); `touched` controls when errors display; "Next" marks all fields in the step touched.
- Steps are data-driven (`STEPS` array) so adding a step is trivial.

**Complete working solution**

```jsx
import { useState } from 'react';

const validators = {
  name: v => (v.trim().length < 2 ? 'Name must be at least 2 characters' : ''),
  email: v => (/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(v) ? '' : 'Enter a valid email'),
  password: v => (v.length < 8 ? 'Password must be at least 8 characters' : ''),
  city: v => (v.trim() ? '' : 'City is required'),
  zip: v => (/^\d{5,6}$/.test(v) ? '' : 'ZIP must be 5–6 digits'),
};

const STEPS = [
  { title: 'Account', fields: [
    { name: 'name', label: 'Name', type: 'text' },
    { name: 'email', label: 'Email', type: 'email' },
    { name: 'password', label: 'Password', type: 'password' },
  ] },
  { title: 'Address', fields: [
    { name: 'city', label: 'City', type: 'text' },
    { name: 'zip', label: 'ZIP', type: 'text' },
  ] },
  { title: 'Review', fields: [] },
];

const initial = { name: '', email: '', password: '', city: '', zip: '' };

function validateFields(fields, data) {
  const errors = {};
  fields.forEach(f => {
    const msg = validators[f.name](data[f.name]);
    if (msg) errors[f.name] = msg;
  });
  return errors;
}

export default function App() {
  const [step, setStep] = useState(0);
  const [data, setData] = useState(initial);
  const [touched, setTouched] = useState({});
  const [status, setStatus] = useState('idle'); // idle | submitting | done

  const current = STEPS[step];
  const errors = validateFields(current.fields, data); // derived, not stored
  const isLast = step === STEPS.length - 1;

  const onChange = e => setData(d => ({ ...d, [e.target.name]: e.target.value }));
  const onBlur = e => setTouched(t => ({ ...t, [e.target.name]: true }));

  const next = () => {
    if (Object.keys(errors).length) {
      setTouched(t => ({ ...t, ...Object.fromEntries(current.fields.map(f => [f.name, true])) }));
      return;
    }
    setStep(s => s + 1);
  };

  const submit = async e => {
    e.preventDefault();
    if (!isLast) return next();
    // re-validate everything before submit
    const all = validateFields(STEPS.flatMap(s => s.fields), data);
    if (Object.keys(all).length) {
      const firstBad = STEPS.findIndex(s => s.fields.some(f => all[f.name]));
      setTouched(Object.fromEntries(Object.keys(data).map(k => [k, true])));
      setStep(firstBad);
      return;
    }
    setStatus('submitting');
    await new Promise(r => setTimeout(r, 800)); // mock API
    setStatus('done');
  };

  if (status === 'done') {
    return <div className="wiz"><h2>🎉 Thanks, {data.name}!</h2><p>We sent a confirmation to {data.email}.</p></div>;
  }

  return (
    <form className="wiz" onSubmit={submit} noValidate>
      <ol className="progress" aria-label="Progress">
        {STEPS.map((s, i) => (
          <li key={s.title} className={i === step ? 'cur' : i < step ? 'done' : ''} aria-current={i === step ? 'step' : undefined}>
            {i + 1}. {s.title}
          </li>
        ))}
      </ol>

      {current.fields.map(f => {
        const showError = touched[f.name] && errors[f.name];
        return (
          <div key={f.name} className="field">
            <label htmlFor={f.name}>{f.label}</label>
            <input
              id={f.name} name={f.name} type={f.type} value={data[f.name]}
              onChange={onChange} onBlur={onBlur}
              aria-invalid={!!showError}
              aria-describedby={showError ? `${f.name}-err` : undefined}
            />
            {showError && <span id={`${f.name}-err`} role="alert" className="err">{errors[f.name]}</span>}
          </div>
        );
      })}

      {isLast && (
        <dl>
          <dt>Name</dt><dd>{data.name}</dd>
          <dt>Email</dt><dd>{data.email}</dd>
          <dt>City</dt><dd>{data.city}</dd>
          <dt>ZIP</dt><dd>{data.zip}</dd>
        </dl>
      )}

      <div className="actions">
        <button type="button" onClick={() => setStep(s => s - 1)} disabled={step === 0 || status === 'submitting'}>Back</button>
        {isLast
          ? <button type="submit" disabled={status === 'submitting'}>{status === 'submitting' ? 'Submitting…' : 'Submit'}</button>
          : <button type="button" onClick={next}>Next</button>}
      </div>
    </form>
  );
}
```

```css
.wiz { width: 360px; margin: 30px auto; font-family: sans-serif; }
.progress { display: flex; gap: 8px; list-style: none; padding: 0; color: #888; }
.progress .cur { color: #1976d2; font-weight: 600; }
.progress .done { color: #2e7d32; }
.field { display: flex; flex-direction: column; margin-bottom: 12px; }
.field input { padding: 8px; }
.field input[aria-invalid="true"] { border: 1px solid #c62828; }
.err { color: #c62828; font-size: 12px; margin-top: 4px; }
.actions { display: flex; justify-content: space-between; }
```

**Edge cases & improvements**
- Pressing Enter in a non-final step: `submit` routes to `next()` (handled by `if (!isLast)`).
- Persist draft to `sessionStorage`; warn on `beforeunload`; focus the first invalid field.
- Async validation (email uniqueness) with debounce + cancellation (see Q188).
- Production: `react-hook-form` + `zod`/`yup` schema; per-step schema using `trigger(fields)`.
- Passwords shouldn't be shown on a review screen (we omit it) and shouldn't be persisted.

**Follow-up questions**
- Controlled vs uncontrolled inputs here; which scales better?
- Where should validation state live — per step component or the wizard?
- How would you add a "conditional step" that depends on earlier answers?
- Route-based steps (`/signup/2`) pros and cons (back button, deep links).

---

### Q193. Build a todo app with undo/redo.

**Requirements**
- Add, toggle, edit (double-click), delete, and clear completed todos; filter All/Active/Done.
- Undo/Redo buttons plus Ctrl/Cmd+Z and Ctrl/Cmd+Shift+Z / Ctrl+Y; disabled when unavailable; persists to `localStorage`.

**Approach / design thinking**
- History pattern: `{ past: [], present, future: [] }` in a reducer. Every undoable action pushes `present` onto `past`, clears `future`. UNDO pops from `past`, pushes `present` to `future`; REDO is the reverse.
- Generic `useUndoable(reducer, initial)` wrapper keeps undo logic separate from todo logic; the history is capped (e.g. 100) to bound memory.
- Immutable snapshots mean structural sharing keeps this cheap (no deep clone).
- Filter is UI state — not part of history.

**Complete working solution**

```jsx
import { useEffect, useReducer, useRef, useState } from 'react';

const LIMIT = 100;

function undoable(reducer) {
  return function (state, action) {
    const { past, present, future } = state;
    switch (action.type) {
      case 'UNDO':
        if (!past.length) return state;
        return { past: past.slice(0, -1), present: past[past.length - 1], future: [present, ...future] };
      case 'REDO':
        if (!future.length) return state;
        return { past: [...past, present].slice(-LIMIT), present: future[0], future: future.slice(1) };
      default: {
        const next = reducer(present, action);
        if (next === present) return state; // no-op: don't pollute history
        return { past: [...past, present].slice(-LIMIT), present: next, future: [] };
      }
    }
  };
}

let uid = Date.now();
function todosReducer(todos, action) {
  switch (action.type) {
    case 'ADD': {
      const text = action.text.trim();
      return text ? [...todos, { id: ++uid, text, done: false }] : todos;
    }
    case 'TOGGLE': return todos.map(t => (t.id === action.id ? { ...t, done: !t.done } : t));
    case 'EDIT': {
      const text = action.text.trim();
      if (!text) return todos.filter(t => t.id !== action.id);
      return todos.map(t => (t.id === action.id && t.text !== text ? { ...t, text } : t));
    }
    case 'DELETE': return todos.filter(t => t.id !== action.id);
    case 'CLEAR_DONE': return todos.some(t => t.done) ? todos.filter(t => !t.done) : todos;
    default: return todos;
  }
}

const reducer = undoable(todosReducer);

function init() {
  try {
    const saved = JSON.parse(localStorage.getItem('todos-v1'));
    if (Array.isArray(saved)) return { past: [], present: saved, future: [] };
  } catch { /* ignore corrupt storage */ }
  return { past: [], present: [], future: [] };
}

function TodoItem({ todo, dispatch }) {
  const [editing, setEditing] = useState(false);
  const [draft, setDraft] = useState(todo.text);
  const commit = () => { setEditing(false); dispatch({ type: 'EDIT', id: todo.id, text: draft }); };
  return (
    <li className={todo.done ? 'done' : ''}>
      <input type="checkbox" checked={todo.done} onChange={() => dispatch({ type: 'TOGGLE', id: todo.id })} />
      {editing ? (
        <input
          autoFocus value={draft}
          onChange={e => setDraft(e.target.value)}
          onBlur={commit}
          onKeyDown={e => { if (e.key === 'Enter') commit(); if (e.key === 'Escape') { setDraft(todo.text); setEditing(false); } }}
        />
      ) : (
        <span onDoubleClick={() => { setDraft(todo.text); setEditing(true); }}>{todo.text}</span>
      )}
      <button aria-label={`Delete ${todo.text}`} onClick={() => dispatch({ type: 'DELETE', id: todo.id })}>✕</button>
    </li>
  );
}

export default function App() {
  const [{ past, present: todos, future }, dispatch] = useReducer(reducer, undefined, init);
  const [filter, setFilter] = useState('all');
  const [text, setText] = useState('');
  const inputRef = useRef(null);

  useEffect(() => { localStorage.setItem('todos-v1', JSON.stringify(todos)); }, [todos]);

  useEffect(() => {
    const onKey = e => {
      const mod = e.ctrlKey || e.metaKey;
      if (!mod) return;
      const k = e.key.toLowerCase();
      // don't hijack native text undo inside inputs
      if (e.target.tagName === 'INPUT' && e.target !== document.body && e.target.type === 'text') return;
      if (k === 'z' && !e.shiftKey) { e.preventDefault(); dispatch({ type: 'UNDO' }); }
      else if ((k === 'z' && e.shiftKey) || k === 'y') { e.preventDefault(); dispatch({ type: 'REDO' }); }
    };
    window.addEventListener('keydown', onKey);
    return () => window.removeEventListener('keydown', onKey);
  }, []);

  const visible = todos.filter(t => filter === 'all' || (filter === 'done' ? t.done : !t.done));

  return (
    <div className="todo">
      <h2>Todos</h2>
      <div>
        <button onClick={() => dispatch({ type: 'UNDO' })} disabled={!past.length}>↶ Undo ({past.length})</button>{' '}
        <button onClick={() => dispatch({ type: 'REDO' })} disabled={!future.length}>↷ Redo ({future.length})</button>
      </div>
      <form onSubmit={e => { e.preventDefault(); dispatch({ type: 'ADD', text }); setText(''); inputRef.current.focus(); }}>
        <input ref={inputRef} value={text} onChange={e => setText(e.target.value)} placeholder="What needs doing?" />
        <button>Add</button>
      </form>
      <ul>{visible.map(t => <TodoItem key={t.id} todo={t} dispatch={dispatch} />)}</ul>
      <div>
        {['all', 'active', 'done'].map(f => (
          <button key={f} onClick={() => setFilter(f)} aria-pressed={filter === f}>{f}</button>
        ))}
        <button onClick={() => dispatch({ type: 'CLEAR_DONE' })}>Clear completed</button>
      </div>
    </div>
  );
}
```

```css
.todo { width: 380px; margin: 30px auto; font-family: sans-serif; }
.todo ul { list-style: none; padding: 0; }
.todo li { display: flex; gap: 8px; align-items: center; padding: 6px 0; border-bottom: 1px solid #eee; }
.todo li span { flex: 1; cursor: text; }
.todo li.done span { text-decoration: line-through; color: #888; }
button[aria-pressed="true"] { font-weight: bold; }
```

**Edge cases & improvements**
- No-op actions return the same `present` so the reducer returns the same `state` (and React bails out) — no phantom undo steps.
- Typing in the add-box is intentionally not a history entry (only committed actions are).
- Hotkey handler ignores text inputs so the browser's native text-undo still works there.
- Alternatives: store *patches/inverse commands* (Immer `produceWithPatches`) for large state; coalesce rapid edits (group by time); cap memory.
- `crypto.randomUUID()` for ids in real apps; sync across tabs via the `storage` event.

**Follow-up questions**
- Snapshot history vs command pattern: trade-offs?
- Why wrap the reducer (higher-order reducer) rather than put history into each case?
- How would you persist the undo stack too, and should you?
- How would this behave with server-synced todos (optimistic updates + undo)?

---

### Q194. Build a drag-and-drop Kanban board.

**Requirements**
- Columns (To Do / In Progress / Done) with cards; drag a card to another column or reorder within a column, with a visual drop indicator.
- Add a card to a column; delete a card; state persists across reloads.
- No external DnD library (native HTML5 Drag & Drop), keyboard fallback via move buttons.

**Approach / design thinking**
- Normalised state: `{ columns: { colId: { title, cardIds: [] } }, cards: { id: {id,text} }, order: [colIds] }` — moves become simple array splices.
- `dragged = { cardId, fromCol }` kept in a ref/state; drop targets are a column (append) or a card (insert before it).
- A pure `moveCard(state, cardId, toCol, toIndex)` function handles same-column index shift correctly (remove first, then adjust index).
- Accessible alternative: ◀ ▶ buttons since pointer DnD is not keyboard-friendly.

**Complete working solution**

```jsx
import { useEffect, useState } from 'react';

const initialBoard = {
  order: ['todo', 'doing', 'done'],
  columns: {
    todo: { title: 'To Do', cardIds: ['c1', 'c2'] },
    doing: { title: 'In Progress', cardIds: ['c3'] },
    done: { title: 'Done', cardIds: [] },
  },
  cards: {
    c1: { id: 'c1', text: 'Write spec' },
    c2: { id: 'c2', text: 'Design UI' },
    c3: { id: 'c3', text: 'Build board' },
  },
};

function moveCard(board, cardId, toCol, toIndex) {
  const fromCol = board.order.find(c => board.columns[c].cardIds.includes(cardId));
  if (!fromCol) return board;
  const fromIds = board.columns[fromCol].cardIds;
  const fromIndex = fromIds.indexOf(cardId);

  if (fromCol === toCol) {
    // after removal, indexes after the old position shift left by 1
    let target = toIndex > fromIndex ? toIndex - 1 : toIndex;
    if (target === fromIndex) return board;
    const ids = fromIds.filter(id => id !== cardId);
    ids.splice(target, 0, cardId);
    return { ...board, columns: { ...board.columns, [fromCol]: { ...board.columns[fromCol], cardIds: ids } } };
  }
  const nextFrom = fromIds.filter(id => id !== cardId);
  const nextTo = [...board.columns[toCol].cardIds];
  nextTo.splice(toIndex, 0, cardId);
  return {
    ...board,
    columns: {
      ...board.columns,
      [fromCol]: { ...board.columns[fromCol], cardIds: nextFrom },
      [toCol]: { ...board.columns[toCol], cardIds: nextTo },
    },
  };
}

function load() {
  try { return JSON.parse(localStorage.getItem('kanban-v1')) || initialBoard; } catch { return initialBoard; }
}

export default function App() {
  const [board, setBoard] = useState(load);
  const [drag, setDrag] = useState(null);           // { cardId }
  const [over, setOver] = useState(null);           // { col, index }
  const [drafts, setDrafts] = useState({});

  useEffect(() => { localStorage.setItem('kanban-v1', JSON.stringify(board)); }, [board]);

  const addCard = col => {
    const text = (drafts[col] || '').trim();
    if (!text) return;
    const id = `c${Date.now()}${Math.random().toString(36).slice(2, 6)}`;
    setBoard(b => ({
      ...b,
      cards: { ...b.cards, [id]: { id, text } },
      columns: { ...b.columns, [col]: { ...b.columns[col], cardIds: [...b.columns[col].cardIds, id] } },
    }));
    setDrafts(d => ({ ...d, [col]: '' }));
  };

  const deleteCard = (col, id) => setBoard(b => {
    const { [id]: _removed, ...cards } = b.cards;
    return { ...b, cards, columns: { ...b.columns, [col]: { ...b.columns[col], cardIds: b.columns[col].cardIds.filter(x => x !== id) } } };
  });

  const drop = () => {
    if (drag && over) setBoard(b => moveCard(b, drag.cardId, over.col, over.index));
    setDrag(null);
    setOver(null);
  };

  const shift = (col, id, dir) => {
    const idx = board.order.indexOf(col) + dir;
    if (idx < 0 || idx >= board.order.length) return;
    const toCol = board.order[idx];
    setBoard(b => moveCard(b, id, toCol, b.columns[toCol].cardIds.length));
  };

  return (
    <div className="board">
      {board.order.map(colId => {
        const col = board.columns[colId];
        return (
          <section
            key={colId}
            className={`col ${over?.col === colId ? 'over' : ''}`}
            onDragOver={e => { e.preventDefault(); if (drag && (over?.col !== colId || over.index !== col.cardIds.length)) setOver({ col: colId, index: col.cardIds.length }); }}
            onDrop={e => { e.preventDefault(); drop(); }}
          >
            <h3>{col.title} <small>({col.cardIds.length})</small></h3>
            <ul>
              {col.cardIds.map((id, i) => (
                <li
                  key={id}
                  draggable
                  className={`card ${drag?.cardId === id ? 'dragging' : ''} ${over?.col === colId && over.index === i && drag?.cardId !== id ? 'indicator' : ''}`}
                  onDragStart={e => { e.dataTransfer.effectAllowed = 'move'; e.dataTransfer.setData('text/plain', id); setDrag({ cardId: id }); }}
                  onDragEnd={() => { setDrag(null); setOver(null); }}
                  onDragOver={e => {
                    e.preventDefault();
                    e.stopPropagation();
                    const r = e.currentTarget.getBoundingClientRect();
                    const index = e.clientY < r.top + r.height / 2 ? i : i + 1; // upper half -> before, lower half -> after
                    if (over?.col !== colId || over.index !== index) setOver({ col: colId, index });
                  }}
                  onDrop={e => { e.preventDefault(); e.stopPropagation(); drop(); }}
                >
                  <span>{board.cards[id].text}</span>
                  <span className="ctl">
                    <button aria-label="Move left" onClick={() => shift(colId, id, -1)}>◀</button>
                    <button aria-label="Move right" onClick={() => shift(colId, id, 1)}>▶</button>
                    <button aria-label="Delete" onClick={() => deleteCard(colId, id)}>✕</button>
                  </span>
                </li>
              ))}
            </ul>
            <form onSubmit={e => { e.preventDefault(); addCard(colId); }}>
              <input value={drafts[colId] || ''} onChange={e => setDrafts(d => ({ ...d, [colId]: e.target.value }))} placeholder="Add card…" />
            </form>
          </section>
        );
      })}
    </div>
  );
}
```

```css
.board { display: flex; gap: 16px; padding: 20px; font-family: sans-serif; align-items: flex-start; }
.col { flex: 1; background: #f1f3f5; border-radius: 8px; padding: 10px; min-height: 160px; }
.col.over { outline: 2px dashed #1976d2; }
.col ul { list-style: none; padding: 0; margin: 0 0 8px; min-height: 20px; }
.card { background: #fff; padding: 8px 10px; margin-bottom: 8px; border-radius: 6px; cursor: grab;
  display: flex; justify-content: space-between; gap: 8px; box-shadow: 0 1px 2px rgba(0,0,0,.15); }
.card.dragging { opacity: .4; }
.card.indicator { box-shadow: 0 -3px 0 #1976d2; }
.ctl button { border: 0; background: none; cursor: pointer; font-size: 11px; }
.col input { width: 100%; box-sizing: border-box; padding: 6px; }
```

**Edge cases & improvements**
- Dropping on the end of a column: column `onDragOver` sets index = length; card handlers `stopPropagation` so the column doesn't override them.
- Same-column move index adjustment (`toIndex > fromIndex ? toIndex - 1`) is the classic off-by-one; unit test `moveCard`.
- Dragging outside any target cancels (`onDragEnd` clears state).
- Touch devices don't fire HTML5 DnD — use Pointer Events or `@dnd-kit/core` (sensors, keyboard support, auto-scroll, collision detection), `react-beautiful-dnd`/`pragmatic-drag-and-drop`.
- Persist to a backend with optimistic update + rollback; use fractional indexing (LexoRank) so reorder updates a single row; column reorder; virtualization for huge columns.

**Follow-up questions**
- Why normalised state instead of nested arrays?
- How would you add undo (see Q193) or real-time collaboration?
- How do you give screen-reader users live announcements of moves (`aria-live`)?
- How do you avoid re-rendering all cards on every `dragover` (memoised `Card` + stable handlers, throttle)?

---

### Q195. Build a data table with sorting, filtering and pagination.

**Requirements**
- Columns: Name, Role, Age, Salary. Click header to cycle sort asc → desc → none (numeric vs string aware).
- Global text filter + role dropdown; page size selector; Prev/Next/page indicator and "showing X–Y of Z".
- Reset to page 1 when filter/sort/page-size changes; `aria-sort` on headers.

**Approach / design thinking**
- Pipeline: `rows → filter → sort → paginate`, computed with `useMemo` (derived state, not stored).
- Store only the *inputs*: `query`, `role`, `sort {key, dir}`, `page`, `pageSize`.
- Reset page by clamping (`Math.min(page, pageCount)`) rather than an effect, or do it inside the setters that change filters.
- Sort a copy (`[...rows].sort`) — never mutate props/state; stable sort is built in.

**Complete working solution**

```jsx
import { useMemo, useState } from 'react';

const ROLES = ['Engineer', 'Designer', 'Manager', 'Analyst'];
const FIRST = ['Ava', 'Ben', 'Chloe', 'Dev', 'Eli', 'Fay', 'Gus', 'Hana', 'Ivan', 'Jia'];
const LAST = ['Shah', 'Lee', 'Khan', 'Brown', 'Rao', 'Diaz', 'Kim'];

const ROWS = Array.from({ length: 87 }, (_, i) => ({
  id: i + 1,
  name: `${FIRST[i % FIRST.length]} ${LAST[(i * 3) % LAST.length]}`,
  role: ROLES[(i * 7) % ROLES.length],
  age: 22 + ((i * 5) % 30),
  salary: 40000 + ((i * 7919) % 90000),
}));

const COLUMNS = [
  { key: 'name', label: 'Name', numeric: false },
  { key: 'role', label: 'Role', numeric: false },
  { key: 'age', label: 'Age', numeric: true },
  { key: 'salary', label: 'Salary', numeric: true },
];

export default function App() {
  const [query, setQuery] = useState('');
  const [role, setRole] = useState('');
  const [sort, setSort] = useState({ key: null, dir: null });
  const [page, setPage] = useState(1);
  const [pageSize, setPageSize] = useState(10);

  const filtered = useMemo(() => {
    const q = query.trim().toLowerCase();
    return ROWS.filter(r =>
      (!role || r.role === role) &&
      (!q || Object.values(r).some(v => String(v).toLowerCase().includes(q)))
    );
  }, [query, role]);

  const sorted = useMemo(() => {
    if (!sort.key) return filtered;
    const { key, dir } = sort;
    const mult = dir === 'asc' ? 1 : -1;
    return [...filtered].sort((a, b) =>
      typeof a[key] === 'number' ? (a[key] - b[key]) * mult : a[key].localeCompare(b[key]) * mult
    );
  }, [filtered, sort]);

  const pageCount = Math.max(1, Math.ceil(sorted.length / pageSize));
  const current = Math.min(page, pageCount); // clamp instead of syncing with an effect
  const startIdx = (current - 1) * pageSize;
  const pageRows = sorted.slice(startIdx, startIdx + pageSize);

  const toggleSort = key => {
    setSort(s => {
      if (s.key !== key) return { key, dir: 'asc' };
      if (s.dir === 'asc') return { key, dir: 'desc' };
      return { key: null, dir: null };
    });
    setPage(1);
  };

  const ariaSort = key => (sort.key === key ? (sort.dir === 'asc' ? 'ascending' : 'descending') : 'none');

  return (
    <div className="dt">
      <div className="toolbar">
        <input placeholder="Search…" value={query} onChange={e => { setQuery(e.target.value); setPage(1); }} />
        <select value={role} onChange={e => { setRole(e.target.value); setPage(1); }}>
          <option value="">All roles</option>
          {ROLES.map(r => <option key={r}>{r}</option>)}
        </select>
        <select value={pageSize} onChange={e => { setPageSize(Number(e.target.value)); setPage(1); }}>
          {[5, 10, 25].map(n => <option key={n} value={n}>{n} / page</option>)}
        </select>
      </div>

      <table>
        <thead>
          <tr>
            {COLUMNS.map(c => (
              <th key={c.key} aria-sort={ariaSort(c.key)}>
                <button onClick={() => toggleSort(c.key)}>
                  {c.label} {sort.key === c.key ? (sort.dir === 'asc' ? '▲' : '▼') : '↕'}
                </button>
              </th>
            ))}
          </tr>
        </thead>
        <tbody>
          {pageRows.length === 0 && <tr><td colSpan={COLUMNS.length}>No matching rows</td></tr>}
          {pageRows.map(r => (
            <tr key={r.id}>
              <td>{r.name}</td><td>{r.role}</td><td>{r.age}</td><td>${r.salary.toLocaleString()}</td>
            </tr>
          ))}
        </tbody>
      </table>

      <div className="pager">
        <span>
          {sorted.length === 0 ? '0' : `${startIdx + 1}–${startIdx + pageRows.length}`} of {sorted.length}
        </span>
        <button onClick={() => setPage(current - 1)} disabled={current <= 1}>Prev</button>
        <span>Page {current} / {pageCount}</span>
        <button onClick={() => setPage(current + 1)} disabled={current >= pageCount}>Next</button>
      </div>
    </div>
  );
}
```

```css
.dt { width: 620px; margin: 30px auto; font-family: sans-serif; }
.toolbar { display: flex; gap: 8px; margin-bottom: 10px; }
.toolbar input { flex: 1; padding: 6px; }
table { border-collapse: collapse; width: 100%; }
th, td { border-bottom: 1px solid #ddd; padding: 8px; text-align: left; }
th button { background: none; border: 0; font-weight: 600; cursor: pointer; }
.pager { display: flex; gap: 12px; align-items: center; justify-content: flex-end; margin-top: 10px; }
```

**Edge cases & improvements**
- Page clamp handles filtered result shrinking; `Number` conversion for `pageSize`; empty result state.
- Debounce the search input for big datasets (Q188); `useDeferredValue(query)` keeps typing responsive.
- Server-side mode: send `{q, sort, page, pageSize}` to the API, keep `keepPreviousData`, and show total count from the server.
- Multi-column sort (shift-click), column visibility, row selection, sticky header, virtualization (Q189), URL-synced state (`useSearchParams`), CSV export; or use TanStack Table for headless logic.
- Locale-aware sort: `Intl.Collator` with `numeric: true`.

**Follow-up questions**
- Why derive filtered/sorted data instead of storing it in state?
- What is the complexity of this pipeline and which step would you optimise first?
- Client-side vs server-side pagination — when to choose which?
- Offset vs cursor pagination?

---

### Q196. Build a custom useFetch with caching.

**Requirements**
- `useFetch(url, options)` returns `{ data, error, loading, refetch }`.
- Module-level cache with TTL; stale-while-revalidate (show cached data immediately, refresh in background when stale).
- De-duplicate concurrent requests for the same URL; ignore results for outdated URLs (race safety); manual `refetch`; skip when `url` is falsy.

**Approach / design thinking**
- Two module-level `Map`s: `cache` (`url → {data, ts}`) and `inflight` (`url → Promise`). Multiple components asking for the same URL share one network call.
- Per-effect `cancelled` flag guarantees only the latest URL's result updates state (shared promises are not aborted because other consumers may need them).
- `fetcher` is injectable (tests, axios, GraphQL) and held in a ref so a new inline function doesn't retrigger the effect.
- This is a mini version of SWR/React Query; mention them for production.

**Complete working solution**

```jsx
import { useCallback, useEffect, useRef, useState } from 'react';

const cache = new Map();     // url -> { data, ts }
const inflight = new Map();  // url -> Promise

const defaultFetcher = async url => {
  const res = await fetch(url);
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return res.json();
};

export function useFetch(url, { ttl = 30000, fetcher = defaultFetcher } = {}) {
  const [state, setState] = useState({ data: undefined, error: null, loading: !!url });
  const [nonce, setNonce] = useState(0);
  const forceRef = useRef(false);
  const fetcherRef = useRef(fetcher);
  fetcherRef.current = fetcher;

  useEffect(() => {
    if (!url) {
      setState({ data: undefined, error: null, loading: false });
      return;
    }
    let cancelled = false;
    const entry = cache.get(url);
    const fresh = !!entry && Date.now() - entry.ts < ttl && !forceRef.current;
    forceRef.current = false;

    if (entry) setState({ data: entry.data, error: null, loading: !fresh }); // stale-while-revalidate
    else setState({ data: undefined, error: null, loading: true });
    if (fresh) return;

    let promise = inflight.get(url);
    if (!promise) {
      promise = fetcherRef.current(url)
        .then(data => { cache.set(url, { data, ts: Date.now() }); return data; })
        .finally(() => inflight.delete(url));
      inflight.set(url, promise);
    }
    promise.then(
      data => { if (!cancelled) setState({ data, error: null, loading: false }); },
      error => { if (!cancelled) setState(s => ({ data: s.data, error, loading: false })); }
    );
    return () => { cancelled = true; };
  }, [url, ttl, nonce]);

  const refetch = useCallback(() => { forceRef.current = true; setNonce(n => n + 1); }, []);
  return { ...state, refetch };
}

export const invalidate = url => (url ? cache.delete(url) : cache.clear());

/* ---------- Demo with a mock fetcher ---------- */
let networkCalls = 0;
const mockFetcher = url => new Promise((resolve, reject) => {
  networkCalls++;
  setTimeout(() => {
    const id = Number(url.split('/').pop());
    if (id === 13) reject(new Error('User 13 is unlucky'));
    else resolve({ id, name: `User ${id}`, fetchedAt: new Date().toLocaleTimeString() });
  }, 700);
});

function User({ id }) {
  const { data, error, loading, refetch } = useFetch(id ? `/api/users/${id}` : null, { ttl: 10000, fetcher: mockFetcher });
  return (
    <div className="card">
      {loading && !data && <p>Loading…</p>}
      {error && <p role="alert">Error: {error.message}</p>}
      {data && <p>{data.name} (fetched {data.fetchedAt}) {loading && '↻ refreshing'}</p>}
      <button onClick={refetch}>Refetch</button>
    </div>
  );
}

export default function App() {
  const [id, setId] = useState(1);
  const [calls, setCalls] = useState(0);
  return (
    <div style={{ fontFamily: 'sans-serif', margin: 30 }}>
      {[1, 2, 3, 13].map(n => <button key={n} onClick={() => { setId(n); setTimeout(() => setCalls(networkCalls), 0); }}>User {n}</button>)}
      <p>Network calls so far: {calls}. Switch back to a visited user within 10s → instant, no new call.</p>
      <User id={id} />
      <User id={id} /> {/* same URL -> deduped into one request */}
    </div>
  );
}
```

> Note: `useState` is already imported in the first line, so the demo `App` runs as-is.

**Edge cases & improvements**
- Errors are not cached; the previous `data` is kept on revalidation failure.
- Real cancellation: reference-count subscribers and `AbortController.abort()` when the last unsubscribes.
- Add: `staleTime` vs `cacheTime` (gc), retries with backoff, focus/reconnect revalidation, request keys with params (`['users', id]`), mutations + optimistic updates + `invalidate`, `useSyncExternalStore` for a shared cache so all consumers update when one refetches (here other mounted consumers of the same URL don't see a manual refetch until they re-render).
- SSR: cache must be per-request on the server (never a module-level global that leaks between users).
- JSON-serialisable keys; cache size limit (LRU).

**Follow-up questions**
- Why store the fetcher in a ref? What would break if it were in deps?
- How does React 19 `use(promise)` + Suspense change data fetching?
- Compare with React Query / SWR; what do they add?
- How would you prevent memory leaks in the cache?

---

### Q197. Build a Tic-Tac-Toe game or a calculator.

**Requirements**
- Tic-Tac-Toe: 3×3 board, alternating X/O, win/draw detection with winning line highlight, move history with time travel, restart.
- Calculator (bonus, shown too): digits, decimal, + − × ÷, chaining, equals, clear, backspace, ± ; no `eval`; divide-by-zero handling.

**Approach / design thinking**
- Tic-Tac-Toe: state is only `history` (array of boards) and `step`. Everything else (current board, next player, winner, status) is **derived**. Time travel = set `step` and truncate on a new move.
- Pure `calculateWinner(squares)` returns `{ player, line }`.
- Calculator: a reducer with a small state machine `{cur, prev, op, fresh}` — avoids parsing/eval; floating-point formatted with `toPrecision(12)`.

**Complete working solution**

```jsx
import { useReducer, useState } from 'react';

/* ---------- Tic-Tac-Toe ---------- */
const LINES = [[0,1,2],[3,4,5],[6,7,8],[0,3,6],[1,4,7],[2,5,8],[0,4,8],[2,4,6]];

function calculateWinner(sq) {
  for (const [a, b, c] of LINES) {
    if (sq[a] && sq[a] === sq[b] && sq[a] === sq[c]) return { player: sq[a], line: [a, b, c] };
  }
  return null;
}

function TicTacToe() {
  const [history, setHistory] = useState([Array(9).fill(null)]);
  const [step, setStep] = useState(0);

  const squares = history[step];
  const xIsNext = step % 2 === 0;
  const win = calculateWinner(squares);
  const draw = !win && squares.every(Boolean);
  const status = win ? `Winner: ${win.player}` : draw ? 'Draw!' : `Next: ${xIsNext ? 'X' : 'O'}`;

  const play = i => {
    if (squares[i] || win) return;
    const next = squares.slice();
    next[i] = xIsNext ? 'X' : 'O';
    setHistory([...history.slice(0, step + 1), next]); // drop "future" after time travel
    setStep(step + 1);
  };

  return (
    <div className="ttt">
      <h3 aria-live="polite">{status}</h3>
      <div className="board3">
        {squares.map((v, i) => (
          <button
            key={i}
            className={`sq ${win?.line.includes(i) ? 'win' : ''}`}
            onClick={() => play(i)}
            aria-label={`Square ${i + 1} ${v ?? 'empty'}`}
          >
            {v}
          </button>
        ))}
      </div>
      <ol start={0}>
        {history.map((_, m) => (
          <li key={m}>
            <button onClick={() => setStep(m)} disabled={m === step}>{m === 0 ? 'Start' : `Move #${m}`}</button>
          </li>
        ))}
      </ol>
      <button onClick={() => { setHistory([Array(9).fill(null)]); setStep(0); }}>Restart</button>
    </div>
  );
}

/* ---------- Calculator ---------- */
const initialCalc = { cur: '0', prev: null, op: null, fresh: false };

function compute(a, b, op) {
  switch (op) {
    case '+': return a + b;
    case '−': return a - b;
    case '×': return a * b;
    case '÷': return b === 0 ? null : a / b;
    default: return b;
  }
}
const fmt = n => String(parseFloat(n.toPrecision(12)));

function calcReducer(s, a) {
  switch (a.type) {
    case 'digit': {
      if (s.fresh) return { ...s, cur: a.d === '.' ? '0.' : a.d, fresh: false };
      if (a.d === '.' && s.cur.includes('.')) return s;
      if (s.cur.replace('-', '').length >= 15) return s;
      return { ...s, cur: s.cur === '0' && a.d !== '.' ? a.d : s.cur + a.d };
    }
    case 'op': {
      if (s.cur === 'Error') return s;
      if (s.fresh && s.op) return { ...s, op: a.op }; // change operator
      if (s.op !== null && s.prev !== null) {
        const r = compute(s.prev, Number(s.cur), s.op);
        if (r === null) return { cur: 'Error', prev: null, op: null, fresh: true };
        return { cur: fmt(r), prev: r, op: a.op, fresh: true };
      }
      return { ...s, prev: Number(s.cur), op: a.op, fresh: true };
    }
    case 'equals': {
      if (s.op === null || s.prev === null || s.cur === 'Error') return s;
      const r = compute(s.prev, Number(s.cur), s.op);
      if (r === null) return { cur: 'Error', prev: null, op: null, fresh: true };
      return { cur: fmt(r), prev: null, op: null, fresh: true };
    }
    case 'clear': return initialCalc;
    case 'back': {
      if (s.fresh) return s;
      const c = s.cur.length > 1 && !(s.cur.length === 2 && s.cur[0] === '-') ? s.cur.slice(0, -1) : '0';
      return { ...s, cur: c };
    }
    case 'negate': {
      if (s.cur === '0' || s.cur === 'Error') return s;
      return { ...s, cur: s.cur.startsWith('-') ? s.cur.slice(1) : '-' + s.cur };
    }
    default: return s;
  }
}

function Calculator() {
  const [s, dispatch] = useReducer(calcReducer, initialCalc);
  const d = x => <button onClick={() => dispatch({ type: 'digit', d: x })}>{x}</button>;
  const o = x => <button className="op" onClick={() => dispatch({ type: 'op', op: x })}>{x}</button>;
  return (
    <div className="calc">
      <div className="screen" aria-live="polite">
        <small>{s.prev !== null && s.op ? `${fmt(s.prev)} ${s.op}` : ' '}</small>
        <div>{s.cur}</div>
      </div>
      <div className="keys">
        <button onClick={() => dispatch({ type: 'clear' })}>C</button>
        <button onClick={() => dispatch({ type: 'back' })}>⌫</button>
        <button onClick={() => dispatch({ type: 'negate' })}>±</button>
        {o('÷')}
        {d('7')}{d('8')}{d('9')}{o('×')}
        {d('4')}{d('5')}{d('6')}{o('−')}
        {d('1')}{d('2')}{d('3')}{o('+')}
        <button className="zero" onClick={() => dispatch({ type: 'digit', d: '0' })}>0</button>
        {d('.')}
        <button className="op" onClick={() => dispatch({ type: 'equals' })}>=</button>
      </div>
    </div>
  );
}

export default function App() {
  return (
    <div style={{ display: 'flex', gap: 60, padding: 30, fontFamily: 'sans-serif' }}>
      <TicTacToe />
      <Calculator />
    </div>
  );
}
```

```css
.board3 { display: grid; grid-template-columns: repeat(3, 70px); gap: 4px; }
.sq { height: 70px; font-size: 32px; cursor: pointer; }
.sq.win { background: #c8e6c9; }
.calc { width: 220px; }
.screen { background: #222; color: #fff; padding: 8px; text-align: right; font-size: 28px; min-height: 60px; }
.screen small { color: #aaa; font-size: 12px; }
.keys { display: grid; grid-template-columns: repeat(4, 1fr); gap: 4px; margin-top: 4px; }
.keys button { padding: 14px 0; font-size: 18px; cursor: pointer; }
.keys .zero { grid-column: span 2; }
.keys .op { background: #ff9800; color: #fff; }
```

**Edge cases & improvements**
- Tic-Tac-Toe: clicking an occupied square or after a win is ignored; the key `step` trick makes "undo" trivial. Improvements: scoreboard, AI (minimax / pruning), N×N board with dynamic win lines, online play.
- Calculator: `0.1 + 0.2` → `0.3` via `toPrecision`; `5 ÷ 0` → "Error", which resets on next digit; operator change (`5 + ×`) replaces operator; chaining `2 + 3 × 4` evaluates left-to-right (like a basic pocket calculator). For true precedence, implement the shunting-yard algorithm; for money, use integer cents/BigInt or decimal libs.
- Keyboard support via a `keydown` listener mapping keys to actions; accessibility with `aria-live`.

**Follow-up questions**
- Why derive `winner`, `xIsNext` rather than storing them?
- How would you implement an unbeatable AI? Complexity of minimax for 3×3?
- How would you support expressions like `(2+3)*4`?
- How do you unit-test `calcReducer` and `calculateWinner` (pure functions)?

---

### Q198. Build an accordion and a carousel.

**Requirements**
- Accordion: single-open or multiple-open mode, animated expand/collapse, `aria-expanded`/`aria-controls`, Up/Down/Home/End keyboard navigation between headers.
- Carousel: previous/next, dot indicators, looping, autoplay that pauses on hover/focus, swipe/drag with pointer events, `aria-roledescription="carousel"`.

**Approach / design thinking**
- Accordion state: a `Set` of open ids (works for single and multiple modes — single mode replaces the set with one id). Height animation via CSS grid `0fr → 1fr` (no JS height measuring).
- Carousel: a flex track translated by `-index * 100%`; autoplay uses a `setTimeout` keyed on `index` so manual navigation resets the timer; swipe computed from pointer down/up delta.

**Complete working solution**

```jsx
import { useEffect, useRef, useState } from 'react';

/* ---------- Accordion ---------- */
function Accordion({ items, multiple = false }) {
  const [open, setOpen] = useState(() => new Set());
  const headers = useRef([]);

  const toggle = id => setOpen(prev => {
    const next = new Set(multiple ? prev : []);
    if (prev.has(id)) next.delete(id); else next.add(id);
    return next;
  });

  const onKeyDown = (e, i) => {
    const last = items.length - 1;
    const map = { ArrowDown: i === last ? 0 : i + 1, ArrowUp: i === 0 ? last : i - 1, Home: 0, End: last };
    if (e.key in map) { e.preventDefault(); headers.current[map[e.key]]?.focus(); }
  };

  return (
    <div className="acc">
      {items.map((it, i) => {
        const isOpen = open.has(it.id);
        return (
          <div key={it.id} className="acc-item">
            <h3>
              <button
                ref={el => (headers.current[i] = el)}
                id={`acc-h-${it.id}`}
                aria-expanded={isOpen}
                aria-controls={`acc-p-${it.id}`}
                onClick={() => toggle(it.id)}
                onKeyDown={e => onKeyDown(e, i)}
              >
                {it.title} <span aria-hidden>{isOpen ? '−' : '+'}</span>
              </button>
            </h3>
            <div className={`acc-panel ${isOpen ? 'open' : ''}`} id={`acc-p-${it.id}`} role="region" aria-labelledby={`acc-h-${it.id}`}>
              <div className="acc-inner"><p>{it.content}</p></div>
            </div>
          </div>
        );
      })}
    </div>
  );
}

/* ---------- Carousel ---------- */
function Carousel({ slides, autoPlay = true, interval = 3000 }) {
  const [index, setIndex] = useState(0);
  const [paused, setPaused] = useState(false);
  const startX = useRef(null);
  const n = slides.length;

  const go = i => setIndex(((i % n) + n) % n); // wraps both directions

  useEffect(() => {
    if (!autoPlay || paused || n < 2) return;
    const id = setTimeout(() => setIndex(i => (i + 1) % n), interval);
    return () => clearTimeout(id);
  }, [index, autoPlay, paused, interval, n]);

  const onPointerDown = e => { startX.current = e.clientX; };
  const onPointerUp = e => {
    if (startX.current === null) return;
    const dx = e.clientX - startX.current;
    startX.current = null;
    if (Math.abs(dx) > 50) go(index + (dx < 0 ? 1 : -1));
  };

  return (
    <div
      className="car"
      role="group"
      aria-roledescription="carousel"
      aria-label="Featured"
      onMouseEnter={() => setPaused(true)}
      onMouseLeave={() => setPaused(false)}
      onFocus={() => setPaused(true)}
      onBlur={() => setPaused(false)}
      onKeyDown={e => { if (e.key === 'ArrowRight') go(index + 1); if (e.key === 'ArrowLeft') go(index - 1); }}
    >
      <div className="car-view" onPointerDown={onPointerDown} onPointerUp={onPointerUp} onPointerCancel={() => (startX.current = null)}>
        <div className="car-track" style={{ transform: `translateX(-${index * 100}%)` }}>
          {slides.map((s, i) => (
            <div key={s.id} className="car-slide" role="group" aria-roledescription="slide" aria-label={`${i + 1} of ${n}`}
                 aria-hidden={i !== index} style={{ background: s.color }}>
              {s.text}
            </div>
          ))}
        </div>
      </div>
      <button className="prev" aria-label="Previous slide" onClick={() => go(index - 1)}>‹</button>
      <button className="next" aria-label="Next slide" onClick={() => go(index + 1)}>›</button>
      <div className="dots">
        {slides.map((s, i) => (
          <button key={s.id} aria-label={`Go to slide ${i + 1}`} aria-current={i === index} className={i === index ? 'dot on' : 'dot'} onClick={() => go(i)} />
        ))}
      </div>
    </div>
  );
}

export default function App() {
  const items = [
    { id: 'a', title: 'What is React?', content: 'A library for building user interfaces.' },
    { id: 'b', title: 'What is JSX?', content: 'A syntax extension that compiles to React.createElement calls.' },
    { id: 'c', title: 'What are hooks?', content: 'Functions that let you use state and lifecycle features in function components.' },
  ];
  const slides = [
    { id: 1, text: 'Slide 1', color: '#ffcdd2' },
    { id: 2, text: 'Slide 2', color: '#c8e6c9' },
    { id: 3, text: 'Slide 3', color: '#bbdefb' },
  ];
  return (
    <div style={{ width: 420, margin: '30px auto', fontFamily: 'sans-serif' }}>
      <h2>Accordion (single)</h2>
      <Accordion items={items} />
      <h2>Accordion (multiple)</h2>
      <Accordion items={items} multiple />
      <h2>Carousel</h2>
      <Carousel slides={slides} />
    </div>
  );
}
```

```css
.acc-item { border-bottom: 1px solid #ddd; }
.acc h3 { margin: 0; }
.acc button { width: 100%; text-align: left; padding: 12px; background: none; border: 0; cursor: pointer; font-size: 16px;
  display: flex; justify-content: space-between; }
.acc-panel { display: grid; grid-template-rows: 0fr; transition: grid-template-rows .25s ease; }
.acc-panel.open { grid-template-rows: 1fr; }
.acc-inner { overflow: hidden; padding: 0 12px; }
.acc-panel.open .acc-inner { padding: 0 12px 12px; }

.car { position: relative; }
.car-view { overflow: hidden; touch-action: pan-y; user-select: none; }
.car-track { display: flex; transition: transform .4s ease; }
.car-slide { flex: 0 0 100%; height: 160px; display: grid; place-items: center; font-size: 28px; }
.prev, .next { position: absolute; top: 50%; transform: translateY(-50%); background: rgba(0,0,0,.4); color: #fff; border: 0; width: 32px; height: 32px; cursor: pointer; }
.prev { left: 6px; } .next { right: 6px; }
.dots { display: flex; justify-content: center; gap: 6px; margin-top: 8px; }
.dot { width: 10px; height: 10px; border-radius: 50%; border: 0; background: #bbb; cursor: pointer; }
.dot.on { background: #1976d2; }
```

**Edge cases & improvements**
- Collapsed accordion panels are still in the DOM; add `inert` (or `hidden` after `transitionend`) so their links aren't focusable when closed.
- `prefers-reduced-motion`: disable transitions/autoplay; provide a pause button (WCAG 2.2.2).
- Carousel: infinite loop using cloned edge slides for seamless animation, drag-follow (translate with pointer move), lazy-load images, `scroll-snap` CSS alternative, multi-item per view, responsive resize.
- Accordion: controlled API (`value`, `onValueChange`), `defaultOpen`, nested accordions.

**Follow-up questions**
- Why a `Set` for open state? How do you enforce single mode?
- How do you animate to `height: auto`? (grid-rows trick, `ResizeObserver`, or `interpolate-size`.)
- Why is the carousel timer keyed on `index`, not a `setInterval`?
- CSS scroll-snap vs transform-based carousel trade-offs?

---

### Q199. Build a chat UI with optimistic messages and reconnects.

**Requirements**
- Messages appear instantly when sent (optimistic) in a `sending` state, switch to `sent` on server ack, or `failed` with a Retry button if no ack arrives.
- A WebSocket-like connection auto-reconnects with exponential backoff + jitter; messages composed while offline are queued and flushed on reconnect without duplicates.
- Connection status banner, auto-scroll to newest message, Enter to send.

**Approach / design thinking**
- **Client-generated `clientId`** per message is the idempotency key: the server de-dupes retries by that id and echoes it in the ack, so the optimistic row is reconciled (not duplicated).
- Keep a `pending` map (clientId → text) outside React state (refs) as the retry/outbox queue; React state holds what's rendered.
- Reconnect logic lives in one effect with closure variables (`attempt`, `disposed`, `retryTimer`) — clean teardown, StrictMode-safe.
- A mock socket (included) lets the demo run offline: it fails to connect 30% of the time, acks after 300 ms, replies as a bot, never acks messages containing the word "fail" (to demo the failed state), and has `drop()` to simulate a disconnect.
- In production swap `createMockSocket` for `new WebSocket(url)` (+ heartbeat/ping, auth token refresh, resume from `lastEventId`).

**Complete working solution**

```jsx
import { useCallback, useEffect, useRef, useState } from 'react';

/* ---------- Mock WebSocket server ---------- */
const seen = new Set(); // server-side idempotency by clientId
let botId = 0;

function createMockSocket({ onOpen, onClose, onMessage }) {
  let open = false, closed = false;
  const timers = new Set();
  const later = (fn, ms) => {
    const t = setTimeout(() => { timers.delete(t); fn(); }, ms);
    timers.add(t);
  };
  const teardown = () => { timers.forEach(clearTimeout); timers.clear(); };

  later(() => {
    if (Math.random() < 0.3) { closed = true; onClose(); } // connection attempt fails
    else { open = true; onOpen(); }
  }, 600);

  return {
    send({ clientId, text }) {
      if (!open) throw new Error('socket not open');
      if (/fail/i.test(text)) return; // server "loses" it -> client timeout -> failed
      later(() => {
        onMessage({ type: 'ack', clientId, id: `srv-${clientId}` });
        if (!seen.has(clientId)) {
          seen.add(clientId);
          later(() => onMessage({ type: 'message', id: `bot-${++botId}`, text: `Echo: ${text}` }), 700);
        }
      }, 300);
    },
    drop() { if (closed) return; open = false; closed = true; teardown(); onClose(); },
    close() { open = false; closed = true; teardown(); },
  };
}

/* ---------- useChat ---------- */
const ACK_TIMEOUT = 5000;

function useChat() {
  const [messages, setMessages] = useState([]);
  const [status, setStatus] = useState('connecting'); // connecting | open | reconnecting
  const sockRef = useRef(null);
  const openRef = useRef(false);
  const pending = useRef(new Map());     // clientId -> text (outbox)
  const ackTimers = useRef(new Map());   // clientId -> timeout id

  const transmit = useCallback(clientId => {
    const text = pending.current.get(clientId);
    if (text === undefined || !openRef.current) return; // stays queued until reconnect
    try { sockRef.current.send({ clientId, text }); } catch { return; }
    clearTimeout(ackTimers.current.get(clientId));
    ackTimers.current.set(clientId, setTimeout(() => {
      pending.current.delete(clientId);
      ackTimers.current.delete(clientId);
      setMessages(m => m.map(x => (x.clientId === clientId ? { ...x, status: 'failed' } : x)));
    }, ACK_TIMEOUT));
  }, []);

  useEffect(() => {
    let disposed = false;
    let attempt = 0;
    let retryTimer;

    const clearAckTimers = () => {
      ackTimers.current.forEach(clearTimeout);
      ackTimers.current.clear();
    };

    const connect = () => {
      setStatus(attempt === 0 ? 'connecting' : 'reconnecting');
      sockRef.current = createMockSocket({
        onOpen() {
          if (disposed) return;
          attempt = 0;
          openRef.current = true;
          setStatus('open');
          pending.current.forEach((_, id) => transmit(id)); // flush outbox
        },
        onClose() {
          if (disposed) return;
          openRef.current = false;
          clearAckTimers(); // unacked messages stay in outbox and get resent
          setStatus('reconnecting');
          const base = Math.min(1000 * 2 ** attempt, 10000);
          attempt += 1;
          retryTimer = setTimeout(connect, base * (0.5 + Math.random() / 2)); // jitter
        },
        onMessage(msg) {
          if (disposed) return;
          if (msg.type === 'ack') {
            pending.current.delete(msg.clientId);
            clearTimeout(ackTimers.current.get(msg.clientId));
            ackTimers.current.delete(msg.clientId);
            setMessages(m => m.map(x => (x.clientId === msg.clientId ? { ...x, status: 'sent', id: msg.id } : x)));
          } else {
            setMessages(m => (m.some(x => x.id === msg.id) ? m : [...m, { id: msg.id, from: 'bot', text: msg.text }]));
          }
        },
      });
    };

    connect();
    return () => {
      disposed = true;
      openRef.current = false;
      clearTimeout(retryTimer);
      clearAckTimers();
      sockRef.current?.close();
    };
  }, [transmit]);

  const messagesSnapshot = useRef(messages);
  messagesSnapshot.current = messages;

  const send = useCallback(text => {
    const clientId = `${Date.now()}-${Math.random().toString(36).slice(2)}`;
    setMessages(m => [...m, { clientId, from: 'me', text, status: 'sending' }]); // optimistic
    pending.current.set(clientId, text);
    transmit(clientId);
  }, [transmit]);

  const retry = useCallback(clientId => {
    const msg = messagesSnapshot.current.find(x => x.clientId === clientId);
    if (!msg) return;
    setMessages(m => m.map(x => (x.clientId === clientId ? { ...x, status: 'sending' } : x)));
    pending.current.set(clientId, msg.text);
    transmit(clientId);
  }, [transmit]);

  const drop = useCallback(() => sockRef.current?.drop(), []);
  return { messages, status, send, retry, drop };
}

/* ---------- UI ---------- */
export default function App() {
  const { messages, status, send, retry, drop } = useChat();
  const [text, setText] = useState('');
  const endRef = useRef(null);

  useEffect(() => { endRef.current?.scrollIntoView({ block: 'end' }); }, [messages]);

  const submit = e => {
    e.preventDefault();
    const t = text.trim();
    if (!t) return;
    send(t);
    setText('');
  };

  return (
    <div className="chat">
      <div className={`banner ${status}`} role="status">
        {status === 'open' ? '● Connected' : status === 'connecting' ? 'Connecting…' : 'Reconnecting…'}
        <button type="button" onClick={drop} disabled={status !== 'open'}>Simulate drop</button>
      </div>
      <ul className="msgs" aria-live="polite">
        {messages.map(m => (
          <li key={m.clientId ?? m.id} className={`msg ${m.from}`}>
            <span>{m.text}</span>
            {m.from === 'me' && (
              <small className={m.status}>
                {m.status === 'sending' && (status === 'open' ? 'Sending…' : 'Queued (offline)')}
                {m.status === 'sent' && '✓ Sent'}
                {m.status === 'failed' && <>Failed <button onClick={() => retry(m.clientId)}>Retry</button></>}
              </small>
            )}
          </li>
        ))}
        <li ref={endRef} aria-hidden />
      </ul>
      <form onSubmit={submit}>
        <input value={text} onChange={e => setText(e.target.value)} placeholder='Type a message (include "fail" to test failure)' />
        <button>Send</button>
      </form>
    </div>
  );
}
```

```css
.chat { width: 380px; margin: 20px auto; border: 1px solid #ccc; display: flex; flex-direction: column; height: 480px; font-family: sans-serif; }
.banner { padding: 6px 10px; font-size: 13px; display: flex; justify-content: space-between; background: #fff3cd; }
.banner.open { background: #d4edda; }
.msgs { flex: 1; overflow-y: auto; list-style: none; margin: 0; padding: 10px; display: flex; flex-direction: column; gap: 8px; }
.msg { max-width: 75%; padding: 8px 10px; border-radius: 10px; display: flex; flex-direction: column; }
.msg.me { align-self: flex-end; background: #dcf8c6; }
.msg.bot { align-self: flex-start; background: #eee; }
.msg small { font-size: 11px; color: #777; align-self: flex-end; }
.msg small.failed { color: #c62828; }
.chat form { display: flex; border-top: 1px solid #ccc; }
.chat form input { flex: 1; padding: 10px; border: 0; }
```

**Edge cases & improvements**
- Duplicates: client generates `clientId`; server stores it (idempotent), re-acks resends; client de-dupes incoming messages by `id`.
- Ordering: server timestamps/sequence numbers; on reconnect request messages after `lastEventId` to fill the gap.
- Backoff: `min(1s·2^n, 10s)` with jitter prevents thundering herds; reset `attempt` on success; also reconnect on `online` event / tab visibility.
- Heartbeats (ping/pong) to detect half-open sockets; close codes (don't retry on 4401 unauthorized).
- Persist outbox to IndexedDB for reload survival; typing indicators, read receipts, pagination of history (scroll up preserving position), virtualization (Q189), `useOptimistic` (React 19) for the optimistic layer, Server-Sent Events fallback.
- Retry when failed re-uses the same `clientId`, so a late ack of the first attempt still can't double-post.

**Follow-up questions**
- What is an idempotency key and why is it essential for retries?
- Optimistic UI: what to do when the server rejects (rollback vs mark failed)?
- How do you handle ordering when two devices send simultaneously?
- WebSockets vs SSE vs long polling; where does state like `status` belong (hook vs context vs store)?

---

### Q200. Build a theme switcher with context and persistence.

**Requirements**
- Light / Dark / System options available anywhere through context (`useTheme`).
- Persist the choice in `localStorage`; follow OS preference live in "system" mode; sync across tabs.
- Apply the theme through a `data-theme` attribute and CSS variables; avoid a flash of the wrong theme on load.

**Approach / design thinking**
- Store the *preference* (`light | dark | system`); derive the *resolved* theme (`light | dark`).
- OS preference is an external store → `useSyncExternalStore` with `matchMedia('(prefers-color-scheme: dark)')` (tear-free, SSR-safe via `getServerSnapshot`).
- Apply on `document.documentElement` in an effect; theme via CSS custom properties so zero components re-style in JS.
- Memoise the context value; wrap `localStorage` in try/catch (private mode, SSR).
- FOUC prevention: tiny inline script in `<head>` that sets `data-theme` before first paint (shown below).

**Complete working solution**

```jsx
import { createContext, useCallback, useContext, useEffect, useMemo, useState, useSyncExternalStore } from 'react';

const STORAGE_KEY = 'theme-preference';
const ThemeContext = createContext(null);

const readStored = () => {
  try {
    const v = localStorage.getItem(STORAGE_KEY);
    return v === 'light' || v === 'dark' || v === 'system' ? v : 'system';
  } catch {
    return 'system';
  }
};

// OS preference as an external store
const mq = () => window.matchMedia('(prefers-color-scheme: dark)');
const subscribeSystem = cb => {
  const m = mq();
  m.addEventListener('change', cb);
  return () => m.removeEventListener('change', cb);
};
const getSystemDark = () => mq().matches;

export function ThemeProvider({ children }) {
  const [preference, setPreference] = useState(readStored);
  const systemDark = useSyncExternalStore(subscribeSystem, getSystemDark, () => false);
  const resolved = preference === 'system' ? (systemDark ? 'dark' : 'light') : preference;

  useEffect(() => {
    const root = document.documentElement;
    root.dataset.theme = resolved;
    root.style.colorScheme = resolved; // native form controls/scrollbars
  }, [resolved]);

  // keep tabs in sync
  useEffect(() => {
    const onStorage = e => {
      if (e.key === STORAGE_KEY) setPreference(readStored());
    };
    window.addEventListener('storage', onStorage);
    return () => window.removeEventListener('storage', onStorage);
  }, []);

  const setTheme = useCallback(next => {
    setPreference(next);
    try { localStorage.setItem(STORAGE_KEY, next); } catch { /* storage unavailable */ }
  }, []);

  const value = useMemo(() => ({ preference, resolved, setTheme }), [preference, resolved, setTheme]);
  return <ThemeContext.Provider value={value}>{children}</ThemeContext.Provider>;
}

export function useTheme() {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error('useTheme must be used within <ThemeProvider>');
  return ctx;
}

function ThemeSwitcher() {
  const { preference, resolved, setTheme } = useTheme();
  return (
    <fieldset className="switcher">
      <legend>Theme (currently {resolved})</legend>
      {['light', 'dark', 'system'].map(opt => (
        <label key={opt}>
          <input type="radio" name="theme" value={opt} checked={preference === opt} onChange={() => setTheme(opt)} />
          {opt}
        </label>
      ))}
    </fieldset>
  );
}

function Page() {
  const { resolved, setTheme } = useTheme();
  return (
    <main className="page">
      <h1>Hello, {resolved} mode</h1>
      <p>Your choice is saved and applied on the next visit.</p>
      <button onClick={() => setTheme(resolved === 'dark' ? 'light' : 'dark')}>Toggle</button>
      <ThemeSwitcher />
    </main>
  );
}

export default function App() {
  return <ThemeProvider><Page /></ThemeProvider>;
}
```

```css
:root, [data-theme='light'] { --bg: #ffffff; --fg: #1a1a1a; --card: #f3f4f6; --accent: #1976d2; }
[data-theme='dark'] { --bg: #121212; --fg: #eeeeee; --card: #1e1e1e; --accent: #90caf9; }
body { background: var(--bg); color: var(--fg); margin: 0; transition: background .2s, color .2s; font-family: sans-serif; }
.page { max-width: 480px; margin: 40px auto; padding: 20px; background: var(--card); border-radius: 8px; }
button { background: var(--accent); color: var(--bg); border: 0; padding: 8px 14px; border-radius: 4px; cursor: pointer; }
.switcher { margin-top: 20px; }
.switcher label { margin-right: 12px; }
```

```html
<!-- index.html <head>: runs before first paint to avoid a flash of the wrong theme -->
<script>
  (function () {
    try {
      var p = localStorage.getItem('theme-preference') || 'system';
      var dark = p === 'dark' || (p === 'system' && matchMedia('(prefers-color-scheme: dark)').matches);
      document.documentElement.dataset.theme = dark ? 'dark' : 'light';
    } catch (e) {}
  })();
</script>
```

**Edge cases & improvements**
- The `storage` event fires only in *other* tabs, so there's no loop; same-tab changes go through `setTheme`.
- `useSyncExternalStore` with `() => false` server snapshot avoids hydration mismatch; the inline script handles the pre-hydration paint.
- Value is memoised so consumers re-render only when preference/resolved actually changes; split context if you need `setTheme`-only consumers.
- Add `prefers-contrast`, more themes (`data-theme="solarized"`), per-user server persistence (cookie so SSR can render the right theme), `<meta name="color-scheme">` / `theme-color`, and Tailwind `darkMode: 'class'`/`selector` integration.
- Disable transitions during the initial switch to avoid a flash of animated colours.

**Follow-up questions**
- Why store `preference` rather than the resolved theme?
- Context vs a tiny external store (Zustand) for theming: trade-offs?
- How do you get SSR (Next.js) to render the correct theme without flicker? (Cookie + `data-theme` on `<html>`, or `next-themes`.)
- How do you test `matchMedia`-dependent code in Jest? (Mock `window.matchMedia`.)
