# React Interview Guide — 01 Fundamentals

## Table of Contents

1. [What is React, and why use it over vanilla JS?](#q1-what-is-react-and-why-use-it-over-vanilla-js)
2. [Virtual DOM vs real DOM](#q2-what-is-the-virtual-dom-how-does-it-differ-from-the-real-dom)
3. [JSX](#q3-what-is-jsx-can-browsers-run-it-directly)
4. [Components: function vs class](#q4-what-are-components-compare-function-and-class-components)
5. [Props](#q5-what-are-props-can-a-component-modify-its-own-props)
6. [State vs props](#q6-what-is-state-how-does-it-differ-from-props)
7. [One-way data flow](#q7-what-is-one-way-data-flow)
8. [Single root & Fragments](#q8-why-must-a-component-return-a-single-root-element-what-are-fragments)
9. [Rendering lists & keys](#q9-how-do-you-render-lists-why-are-keys-needed)
10. [Index as key](#q10-why-is-using-the-array-index-as-a-key-problematic)
11. [Conditional rendering](#q11-how-does-conditional-rendering-work-what-are-the-common-patterns)
12. [Events & SyntheticEvent](#q12-how-do-you-handle-events-in-react-what-is-a-syntheticevent)
13. [Controlled vs uncontrolled](#q13-controlled-vs-uncontrolled-components)
14. [children](#q14-what-is-children-and-how-is-it-used)
15. [defaultProps, PropTypes, TypeScript](#q15-what-is-defaultprops-how-do-proptypes-and-typescript-compare)
16. [Styling](#q16-how-do-you-style-components-css-modules-styled-components-tailwind)
17. [CRA vs Vite vs Next.js](#q17-what-is-create-react-app-vs-vite-vs-nextjs)
18. [Element vs component](#q18-what-is-the-difference-between-an-element-and-a-component)
19. [Prop drilling](#q19-what-is-prop-drilling)
20. [Lifting state up](#q20-what-is-lifting-state-up)

---

### Q1. What is React, and why use it over vanilla JS?

**Short answer**
React is a JavaScript library for building user interfaces from reusable components. You describe what the UI should look like for a given state, and React efficiently updates the DOM when state changes, instead of you manipulating the DOM by hand.

**Detailed explanation**
- **Declarative:** you write `UI = f(state)`. React figures out the DOM mutations. Vanilla JS is imperative: you find nodes, create/remove them, and keep them in sync with data manually, which becomes error-prone as an app grows.
- **Component-based:** UI is split into small, composable, reusable units that encapsulate markup, logic and (optionally) styling.
- **Efficient updates:** React reconciles a lightweight description of the UI (elements) against the previous one and applies minimal DOM changes.
- **Ecosystem:** Hooks, Context, Suspense, concurrent rendering, Server Components (React 19 via frameworks), DevTools, huge library ecosystem, React Native for mobile.
- **Trade-offs:** it is a library, not a framework (routing, data fetching, and build tooling are your choice or come from a framework); there is a learning curve (JSX, hooks rules, effects); for tiny static pages vanilla JS is simpler and lighter.

**Code example**
```jsx
import { useState } from "react";

export default function Counter() {
  const [count, setCount] = useState(0);
  // Declarative: the text always reflects `count`; no manual DOM updates.
  return (
    <button onClick={() => setCount((c) => c + 1)}>
      Clicked {count} times
    </button>
  );
}
```
Vanilla equivalent requires `document.querySelector`, an event listener, and `el.textContent = ...` after each change.

**Common pitfalls / follow-up questions**
- *Is React a framework?* No, it is a view library; frameworks like Next.js or Remix build on it.
- *Is React always faster than vanilla JS?* No. Hand-tuned vanilla can be faster; React's benefit is maintainability and predictable updates at scale.
- *Pitfall:* mixing direct DOM manipulation (`document.getElementById(...).innerHTML = ...`) with React-managed nodes causes desync; use refs when DOM access is truly needed.

---

### Q2. What is the virtual DOM? How does it differ from the real DOM?

**Short answer**
The virtual DOM is a lightweight in-memory tree of JavaScript objects (React elements) that describes the UI. On each update React builds a new tree, diffs it against the previous one (reconciliation), and applies only the necessary changes to the real DOM.

**Detailed explanation**
- **Real DOM:** the browser's actual tree of nodes; changing it can trigger style recalculation, layout and paint, and its API is comparatively heavy.
- **Virtual DOM:** plain objects like `{ type: 'div', props: {...} }`. Creating them is cheap.
- **Reconciliation heuristics:** elements of different types produce different trees (subtree is torn down and rebuilt); same-type elements are updated in place; `key`s identify children in lists.
- **Fiber:** React's reconciler (since 16) represents work as fibers, enabling interruptible/concurrent rendering and prioritization (React 18+).
- **Nuance:** the virtual DOM is not "faster than the DOM" by itself; it is a way to make declarative rendering affordable. Other libraries (Svelte, Solid) skip it using compile-time or fine-grained reactivity.
- **Phases:** *render* (pure calculation, may be paused/repeated) and *commit* (DOM mutations, layout effects, refs).

**Code example**
```jsx
import { useState } from "react";

export default function Clock() {
  const [now, setNow] = useState(() => new Date());

  return (
    <section>
      <h2>Static heading (DOM node untouched on re-render)</h2>
      {/* Only this text node is updated in the real DOM */}
      <p>{now.toLocaleTimeString()}</p>
      <button onClick={() => setNow(new Date())}>Refresh</button>
    </section>
  );
}
```

**Common pitfalls / follow-up questions**
- *Does a re-render mean the DOM is updated?* No. Render produces elements; React only commits real differences.
- *Is the virtual DOM the same as Shadow DOM?* No. Shadow DOM is a browser feature for encapsulation of web components.
- *Pitfall:* defining a component inside another component creates a new type each render, so React remounts it and loses state.

---

### Q3. What is JSX? Can browsers run it directly?

**Short answer**
JSX is a syntax extension that lets you write HTML-like markup inside JavaScript. Browsers cannot run it; a compiler such as Babel, SWC, or esbuild transforms it into JavaScript function calls (`jsx()` / `React.createElement`).

**Detailed explanation**
- `<h1 className="t">Hi</h1>` compiles (automatic runtime, React 17+) to `_jsx("h1", { className: "t", children: "Hi" })`. No need to `import React` just for JSX.
- JSX is an expression: it can be assigned, returned, and passed around.
- Embed JavaScript expressions (not statements) in `{}`.
- Differences from HTML: `className`, `htmlFor`, camelCase attributes/events (`onClick`, `tabIndex`), `style` takes an object, tags must be closed (`<img />`), and components must start with a capital letter.
- JSX escapes embedded values by default, which helps prevent XSS (except `dangerouslySetInnerHTML`).
- JSX is optional; you can use `createElement` directly.

**Code example**
```jsx
const user = { name: "Ada", avatar: "/ada.png" };

export default function Profile() {
  return (
    <header style={{ display: "flex", gap: 8 }}>
      <img className="avatar" src={user.avatar} alt={`${user.name}'s avatar`} />
      <h1>{user.name.toUpperCase()}</h1>
    </header>
  );
}

// Roughly what the compiler emits (automatic runtime):
// jsxs("header", { style: {...}, children: [jsx("img", {...}), jsx("h1", {...})] })
```

**Common pitfalls / follow-up questions**
- *Can you put `if` inside `{}`?* No, only expressions; use ternary, `&&`, or compute beforehand.
- *Why do I get "Adjacent JSX elements must be wrapped"?* Multiple siblings need a parent or Fragment.
- *Pitfall:* `{obj}` rendering a plain object throws ("Objects are not valid as a React child"); render its fields instead.

---

### Q4. What are components? Compare function and class components.

**Short answer**
Components are reusable, self-contained pieces of UI that accept props and return React elements. Function components with Hooks are the modern standard; class components still work but are legacy and needed mainly for error boundaries.

**Detailed explanation**
| Aspect | Function component | Class component |
|---|---|---|
| Syntax | Plain function | `class X extends Component` with `render()` |
| State | `useState`, `useReducer` | `this.state`, `setState` |
| Side effects | `useEffect` | Lifecycle methods (`componentDidMount`, ...) |
| `this` binding | Not an issue | Must bind handlers or use arrow fields |
| Logic reuse | Custom hooks | HOCs / render props |
| Error boundaries | Not yet possible (use a class or `react-error-boundary`) | Supported via `componentDidCatch`/`getDerivedStateFromError` |

- Hooks let related logic stay together rather than being split across lifecycle methods.
- Function components capture props/state values per render (closures), which avoids many stale-`this` bugs.
- Components must be pure with respect to props/state during render.

**Code example**
```jsx
import { Component, useState } from "react";

// Modern
export function Greeting({ name }) {
  const [excited, setExcited] = useState(false);
  return (
    <button onClick={() => setExcited((e) => !e)}>
      Hello, {name}{excited ? "!" : "."}
    </button>
  );
}

// Legacy equivalent
export class GreetingClass extends Component {
  state = { excited: false };
  toggle = () => this.setState((s) => ({ excited: !s.excited }));
  render() {
    return (
      <button onClick={this.toggle}>
        Hello, {this.props.name}{this.state.excited ? "!" : "."}
      </button>
    );
  }
}
```

**Common pitfalls / follow-up questions**
- *Are classes deprecated?* Not removed, but new code should use functions.
- *Why must component names be capitalized?* Lowercase names are treated as DOM tags.
- *Pitfall:* calling a component as a function (`Greeting()`) instead of `<Greeting />` breaks hook rules and identity.

---

### Q5. What are props? Can a component modify its own props?

**Short answer**
Props are the inputs passed from a parent to a child component, similar to function arguments. They are read-only; a component must never mutate its own props. To change data, the owner updates its state and passes new props.

**Detailed explanation**
- Props can be any value: strings, objects, arrays, functions (callbacks), JSX.
- Props are immutable from the child's perspective; React relies on this for pure rendering and memoization (`React.memo` compares props shallowly).
- Destructure in the parameter list; use default parameter values for defaults.
- Special props: `key` and (before React 19 for function components) `ref` are not passed through `props`. In React 19, `ref` is a normal prop for function components.
- Children communicate upward by calling callback props.

**Code example**
```jsx
function Price({ amount, currency = "USD", onBuy }) {
  const formatted = new Intl.NumberFormat("en-US", {
    style: "currency",
    currency,
  }).format(amount);

  // props.amount = 0; // WRONG: never mutate props
  return <button onClick={() => onBuy(amount)}>Buy for {formatted}</button>;
}

export default function Shop() {
  return <Price amount={19.99} onBuy={(a) => console.log("bought", a)} />;
}
```

**Common pitfalls / follow-up questions**
- *What if I mutate a prop object?* You mutate the parent's data, causing bugs and missed re-renders. Copy first.
- *Do props changing always re-render the child?* Yes, whenever the parent re-renders (unless memoized), regardless of prop equality.
- *Pitfall:* creating inline objects/functions as props defeats `React.memo` since identities change each render.

---

### Q6. What is state? How does it differ from props?

**Short answer**
State is data a component owns and remembers between renders; changing it triggers a re-render. Props are passed in from a parent and are read-only, while state is managed internally and updated via setters.

**Detailed explanation**
| | Props | State |
|---|---|---|
| Owner | Parent | The component itself |
| Mutable by component | No | Yes (via setter) |
| Purpose | Configuration/input | Changing data over time |
| Triggers re-render | When parent passes new values | When setter called with a new value |

- `useState` returns `[value, setValue]`. State is a snapshot per render; `setCount(count + 1)` three times in a handler yields +1, while updater form `setCount(c => c + 1)` yields +3.
- Updates are batched (automatic batching in React 18 everywhere) and asynchronous with respect to the current render.
- Never mutate state objects/arrays; create new ones. React bails out if `Object.is` says the value is unchanged.
- Don't store derived data in state; compute it during render.
- State is local to the component instance and position in the tree; it resets if the component unmounts or its `key` changes.

**Code example**
```jsx
import { useState } from "react";

export default function Todo() {
  const [items, setItems] = useState([]);
  const [text, setText] = useState("");

  const add = () => {
    if (!text.trim()) return;
    setItems((prev) => [...prev, { id: crypto.randomUUID(), text }]);
    setText("");
  };

  return (
    <>
      <input value={text} onChange={(e) => setText(e.target.value)} />
      <button onClick={add}>Add</button>
      <p>{items.length} items</p>
    </>
  );
}
```

**Common pitfalls / follow-up questions**
- *Why doesn't `console.log(count)` show the new value after `setCount`?* State is a snapshot; the new value appears on the next render.
- *Why did my array push not re-render?* Mutation keeps the same reference; use spread/`map`/`filter`.
- *Should derived values (e.g. `fullName`) live in state?* No, compute them in render; duplicated state drifts out of sync.

---

### Q7. What is one-way data flow?

**Short answer**
Data in React flows down the tree from parent to child via props, and changes flow up through callbacks that the parent provides. This unidirectional flow makes state changes predictable and easy to trace.

**Detailed explanation**
- A single owner holds each piece of state; children receive it as props and cannot write to it directly.
- To request a change, children call a function passed as a prop (`onChange`, `onSave`), and the owner updates its state, re-rendering the subtree.
- Contrast with two-way binding (Angular `ngModel`, Vue `v-model`) where the view and model sync automatically. In React you get the same effect explicitly with controlled inputs.
- Benefits: easier debugging, fewer cascading updates, clear ownership. Cost: some boilerplate and potential prop drilling (solved by Context/state libraries).

**Code example**
```jsx
import { useState } from "react";

function Toggle({ on, onToggle }) {
  return <button onClick={onToggle}>{on ? "ON" : "OFF"}</button>;
}

export default function Panel() {
  const [on, setOn] = useState(false); // owner
  return (
    <div>
      <Toggle on={on} onToggle={() => setOn((v) => !v)} /> {/* data down, events up */}
      <p>Status: {on ? "enabled" : "disabled"}</p>
    </div>
  );
}
```

**Common pitfalls / follow-up questions**
- *Can siblings communicate?* Via a shared parent (lift state up) or context/store.
- *Does Context break one-way flow?* No; values still flow down from a Provider, and updates go through functions you expose.
- *Pitfall:* mutating a shared object in a child bypasses the flow and causes unpredictable rendering.

---

### Q8. Why must a component return a single root element? What are Fragments?

**Short answer**
A function can return only one value, and JSX compiles to a single `jsx()` call, so multiple siblings must be wrapped in one parent. Fragments (`<>...</>`) group children without adding an extra DOM node.

**Detailed explanation**
- Without a wrapper, `return <h1/><p/>` is a syntax error since it would be two expressions.
- Wrapping in a `<div>` works but can pollute the DOM, break CSS layouts (flex/grid children, `<table>`/`<ul>` structure), and hurt semantics.
- Short syntax `<>...</>` doesn't accept attributes; the long form `<React.Fragment key={id}>` supports `key` and is required when rendering lists of fragments.
- Components can also return arrays, strings, numbers, `null`, or (React 18+) `undefined` (nothing rendered).

**Code example**
```jsx
import { Fragment } from "react";

function Row({ term, definition }) {
  return (
    <>
      <dt>{term}</dt>
      <dd>{definition}</dd>
    </>
  );
}

export default function Glossary({ entries }) {
  return (
    <dl>
      {entries.map((e) => (
        <Fragment key={e.term}>
          <Row term={e.term} definition={e.definition} />
        </Fragment>
      ))}
    </dl>
  );
}
```

**Common pitfalls / follow-up questions**
- *Can `<>` take a key?* No; use `<Fragment key>`.
- *Do fragments appear in the DOM?* No.
- *Pitfall:* extra wrapper `div`s inside `<tr>`/`<ul>` produce invalid HTML; fragments avoid that.

---

### Q9. How do you render lists? Why are keys needed?

**Short answer**
Use `Array.prototype.map` to turn data into elements and give each item a stable, unique `key`. Keys let React identify which items were added, removed, or reordered, so it can preserve state and update the DOM minimally.

**Detailed explanation**
- Keys must be unique among siblings (not globally), stable across renders, and not generated during render (`Math.random()` remounts everything).
- Best keys are IDs from data (database id, slug). If you create items locally, generate an ID once when the item is created.
- `key` is not passed as a prop; pass a separate prop if the child needs it.
- Keys also force a reset: changing a component's `key` remounts it, which is a deliberate way to reset state.
- Filter before mapping; keep the key on the outermost element returned from `map`.

**Code example**
```jsx
export default function UserList({ users }) {
  if (users.length === 0) return <p>No users.</p>;

  return (
    <ul>
      {users
        .filter((u) => u.active)
        .map((u) => (
          <li key={u.id}>
            {u.name} <small>({u.email})</small>
          </li>
        ))}
    </ul>
  );
}
```

**Common pitfalls / follow-up questions**
- *What warning appears without keys?* "Each child in a list should have a unique key prop."
- *Why not `Math.random()` as key?* New key each render means unmount/remount of every item, losing state and focus.
- *Where does the key go?* On the element directly inside `map`, e.g. the `Fragment` or component, not on its inner children.

---

### Q10. Why is using the array index as a key problematic?

**Short answer**
Index keys tie identity to position, not to the data. When items are inserted, removed, or reordered, React matches the wrong items, causing incorrect state retention, stale input values, and unnecessary DOM work.

**Detailed explanation**
- If you delete item 0, the item formerly at index 1 now has key 0; React reuses the old component instance (with its local state, uncontrolled input values, focus) for different data.
- Reordering/sorting/filtering/prepending all shift indexes.
- Index keys are acceptable only when the list is static, never reordered or filtered, and items have no local state/inputs. Even then, a stable ID is better.
- React uses index by default when no key is given (and warns).

**Code example**
```jsx
import { useState } from "react";

export default function Demo() {
  const [items, setItems] = useState([
    { id: "a", label: "Apple" },
    { id: "b", label: "Banana" },
    { id: "c", label: "Cherry" },
  ]);

  return (
    <>
      <button onClick={() => setItems((xs) => xs.slice(1))}>Remove first</button>
      {items.map((item) => (
        // With key={index}, typed text in an input would stay at position 0
        // and appear to "move" to the wrong item after removal.
        <div key={item.id}>
          {item.label} <input defaultValue="" />
        </div>
      ))}
    </>
  );
}
```

**Common pitfalls / follow-up questions**
- *When is index fine?* Static, non-reordered, stateless lists.
- *What if data has no ID?* Assign one at creation time (`crypto.randomUUID()`), not during render.
- *Pitfall:* composite keys like `${name}-${index}` still break on reorder; use a true unique identifier.

---

### Q11. How does conditional rendering work? What are the common patterns?

**Short answer**
React uses ordinary JavaScript: since JSX is an expression, you choose what to render with `if`, ternary, or `&&`. Returning `null` renders nothing.

**Detailed explanation**
Common patterns:
1. **Early return** with `if` for loading/error/empty states, clean and readable.
2. **Ternary** `cond ? <A/> : <B/>` for either/or inline.
3. **Logical AND** `cond && <A/>` for optional rendering.
4. **Variable assignment** of JSX before return for complex branching.
5. **Lookup object / switch** for multiple states.
6. **Return `null`** to hide a component.
7. Toggle via CSS (`hidden`) when you want to preserve DOM/state cheaply.

Notes: conditionally rendered components unmount and lose their state when hidden. Don't call hooks conditionally; put the condition inside or split into components.

**Code example**
```jsx
const STATUS_VIEW = {
  idle: () => <p>Start typing…</p>,
  loading: () => <p>Loading…</p>,
  error: () => <p role="alert">Something went wrong.</p>,
};

export default function Results({ status, items, user }) {
  if (status !== "success") return STATUS_VIEW[status]();

  return (
    <section>
      {user ? <h2>Welcome, {user.name}</h2> : <h2>Welcome, guest</h2>}
      {items.length > 0 && <p>{items.length} results</p>}
      {items.length === 0 && <p>No results.</p>}
    </section>
  );
}
```

**Common pitfalls / follow-up questions**
- *Why does `{count && <X/>}` render "0"?* `0` is a falsy number that React renders; use `count > 0 && ...`.
- *What do `false`, `null`, `undefined`, `true` render?* Nothing.
- *Pitfall:* hiding via conditional rendering resets state; keep it mounted with CSS if state must persist.

---

### Q12. How do you handle events in React? What is a SyntheticEvent?

**Short answer**
Pass a function to camelCase props like `onClick` and `onChange`. React wraps native events in a `SyntheticEvent`, a cross-browser wrapper with the same interface as native events (`preventDefault`, `stopPropagation`, `target`).

**Detailed explanation**
- Pass the function reference (`onClick={handle}`), not a call (`onClick={handle()}`).
- React attaches listeners at the root container (event delegation), not on each node.
- Since React 17 pooling was removed, so you can read the event asynchronously; use `e.nativeEvent` for the underlying event.
- Return `false` doesn't prevent default; call `e.preventDefault()`.
- Capture phase variants: `onClickCapture`.
- `onChange` in React fires on every input change (like native `input`), not on blur.
- Forms: `onSubmit` on `<form>`; React 19 also supports passing a function to `action`.

**Code example**
```jsx
import { useState } from "react";

export default function SearchForm({ onSearch }) {
  const [query, setQuery] = useState("");

  const handleSubmit = (e) => {
    e.preventDefault();
    onSearch(query.trim());
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        onKeyDown={(e) => e.key === "Escape" && setQuery("")}
      />
      <button type="submit">Search</button>
    </form>
  );
}
```

**Common pitfalls / follow-up questions**
- *`onClick={handle()}` bug?* It calls during render; use `onClick={handle}` or `() => handle(arg)`.
- *Does stopPropagation on a React event stop native listeners on `document`?* Not reliably; use `e.nativeEvent.stopImmediatePropagation()` in rare cases.
- *Is event pooling still a concern?* No, removed in React 17.

---

### Q13. Controlled vs uncontrolled components?

**Short answer**
A controlled component's form value lives in React state and is set through `value` plus an `onChange` handler. An uncontrolled component lets the DOM keep the value, and you read it when needed via a ref or `FormData`.

**Detailed explanation**
- **Controlled:** single source of truth is state; enables instant validation, formatting, conditional disabling, and dependent fields. Cost: re-render per keystroke.
- **Uncontrolled:** simpler, less re-rendering, good for simple forms and file inputs (`<input type="file">` is always uncontrolled). Use `defaultValue` for initial values.
- A `value` without `onChange` makes the input read-only (React warns). Switching between `undefined` and defined `value` triggers the controlled/uncontrolled warning, so initialize with `""`.
- Libraries like React Hook Form lean on uncontrolled inputs for performance.
- The idea extends beyond inputs: a component is "controlled" when the parent drives its state via props.

**Code example**
```jsx
import { useRef, useState } from "react";

export function ControlledName() {
  const [name, setName] = useState("");
  return (
    <label>
      Name
      <input value={name} onChange={(e) => setName(e.target.value.toUpperCase())} />
    </label>
  );
}

export function UncontrolledForm() {
  const formRef = useRef(null);

  const onSubmit = (e) => {
    e.preventDefault();
    const data = Object.fromEntries(new FormData(formRef.current));
    console.log(data);
  };

  return (
    <form ref={formRef} onSubmit={onSubmit}>
      <input name="email" type="email" defaultValue="" />
      <input name="avatar" type="file" />
      <button>Save</button>
    </form>
  );
}
```

**Common pitfalls / follow-up questions**
- *Warning "changing an uncontrolled input to be controlled"?* Initial value was `undefined`; default state to `""`.
- *Which to choose?* Controlled for live validation/derived UI; uncontrolled for simple submit-only forms.
- *Pitfall:* using `value` and `defaultValue` together on one input.

---

### Q14. What is children, and how is it used?

**Short answer**
`children` is a special prop containing whatever JSX is nested between a component's opening and closing tags. It enables composition: generic wrappers like layouts, cards, and providers that don't need to know their contents.

**Detailed explanation**
- Can be text, elements, arrays, functions (render props), or `undefined`.
- Composition via children avoids prop drilling and keeps components flexible ("containment").
- Children passed from a parent are created by the parent, so a state change inside the wrapper doesn't re-render them (they are the same element references). This is a useful performance pattern.
- Named "slots" can be implemented with additional props accepting JSX (`header`, `footer`).
- `React.Children` utilities (`map`, `count`, `toArray`) exist but are uncommon and considered fragile; prefer explicit props or context.

**Code example**
```jsx
import { useState } from "react";

function Card({ title, footer, children }) {
  return (
    <article className="card">
      <h3>{title}</h3>
      <div className="card-body">{children}</div>
      {footer && <footer>{footer}</footer>}
    </article>
  );
}

function Collapsible({ children }) {
  const [open, setOpen] = useState(true);
  return (
    <>
      <button onClick={() => setOpen((o) => !o)}>{open ? "Hide" : "Show"}</button>
      {open && children}
    </>
  );
}

export default function App() {
  return (
    <Card title="Profile" footer={<a href="/edit">Edit</a>}>
      <Collapsible>
        <p>Bio goes here.</p>
      </Collapsible>
    </Card>
  );
}
```

**Common pitfalls / follow-up questions**
- *Is `children` always an array?* No: undefined, single element, or array; don't assume.
- *Why use `React.Children.map` carefully?* It doesn't see through fragments/wrappers and breaks with custom components; prefer context.
- *Pitfall:* putting a function as children without documenting it (render-prop) makes APIs confusing.

---

### Q15. What is defaultProps? How do PropTypes and TypeScript compare?

**Short answer**
`defaultProps` supplied fallback values for undefined props. In modern React, use default parameter values instead; `defaultProps` on function components is removed in React 19. PropTypes validate props at runtime in development only, whereas TypeScript checks them statically at compile time.

**Detailed explanation**
- **defaultProps:** `Comp.defaultProps = { size: "md" }`. Still works for classes; for function components deprecated in 18.3 and removed in React 19. Use `function Comp({ size = "md" })`.
- Defaults apply only when the prop is `undefined`, not `null`.
- **PropTypes:** separate `prop-types` package (since React 15.5); warns in the console in development; no compile-time guarantees; React 19 removed built-in `propTypes` checking.
- **TypeScript:** static analysis in editor/CI, generics, discriminated unions, inference, autocompletion, zero runtime cost. It does not validate data at runtime (e.g., API responses); pair with Zod or similar for boundaries.
- Recommendation: TypeScript for new projects; PropTypes only for plain-JS libraries.

**Code example**
```jsx
// Modern JS (with default parameters)
function Button({ variant = "primary", size = "md", children }) {
  return <button className={`btn btn-${variant} btn-${size}`}>{children}</button>;
}
```
```tsx
// TypeScript version
type ButtonProps = {
  variant?: "primary" | "ghost";
  size?: "sm" | "md" | "lg";
  children: React.ReactNode;
};

export function Button({ variant = "primary", size = "md", children }: ButtonProps) {
  return <button className={`btn btn-${variant} btn-${size}`}>{children}</button>;
}
```

**Common pitfalls / follow-up questions**
- *Does a default apply for `null`?* No, only `undefined`.
- *Can TypeScript replace runtime validation?* No, types are erased at build time.
- *Pitfall:* object/array defaults (`items = []`) create a new reference each render; hoist a constant if identity matters.

---

### Q16. How do you style components (CSS modules, styled-components, Tailwind)?

**Short answer**
Common options: plain CSS/Sass files, CSS Modules for locally scoped class names, CSS-in-JS like styled-components or Emotion, utility-first Tailwind CSS, and inline `style` objects. The choice trades off scoping, performance, DX, and server-rendering support.

**Detailed explanation**
- **Global CSS/Sass:** simple but names collide; use naming conventions (BEM).
- **CSS Modules:** `Button.module.css` is compiled so class names become unique hashes; zero runtime, works with SSR and Server Components.
- **styled-components/Emotion:** styles co-located in JS with prop-based dynamic styling and theming. Runtime cost, and runtime CSS-in-JS has friction with React Server Components/streaming SSR; zero-runtime alternatives include vanilla-extract, Panda CSS, and Linaria. styled-components is in maintenance mode.
- **Tailwind:** utility classes in markup, design-token driven, tiny CSS output via JIT, no naming; markup can get verbose (mitigate with components and `clsx`/`cva`).
- **Inline styles:** dynamic values only; no pseudo-classes, media queries, or cascade.
- Dynamic classes: `clsx` helper. Dynamic values: CSS variables.

**Code example**
```jsx
// Button.module.css:  .btn { padding: 8px 12px } .primary { background: royalblue; color: #fff }
import styles from "./Button.module.css";
import clsx from "clsx";

export function ModuleButton({ primary, children }) {
  return <button className={clsx(styles.btn, primary && styles.primary)}>{children}</button>;
}

// Tailwind
export function TwButton({ children }) {
  return (
    <button className="rounded bg-blue-600 px-3 py-2 text-white hover:bg-blue-700">
      {children}
    </button>
  );
}

// styled-components
import styled from "styled-components";
const StyledButton = styled.button`
  padding: 8px 12px;
  background: ${(p) => (p.$primary ? "royalblue" : "#eee")};
`;
export const SButton = ({ primary, children }) => (
  <StyledButton $primary={primary}>{children}</StyledButton>
);
```

**Common pitfalls / follow-up questions**
- *Why `$primary` transient props?* Prevents unknown props from being forwarded to the DOM.
- *Biggest styled-components drawback?* Runtime overhead and SSR/RSC complications.
- *Pitfall:* creating styled components inside render recreates them each render and remounts the DOM.

---

### Q17. What is create-react-app vs Vite vs Next.js?

**Short answer**
Create React App (CRA) was a Webpack-based SPA starter that is now deprecated (the React team sunset it in 2025). Vite is a fast build tool/dev server using native ES modules and Rollup/Rolldown for production; Next.js is a full-stack React framework adding routing, SSR/SSG, Server Components, and API/server features.

**Detailed explanation**
| | CRA | Vite | Next.js |
|---|---|---|---|
| Type | Toolchain (SPA) | Build tool / dev server | Full framework |
| Status | Deprecated | Active, popular | Active, officially recommended |
| Dev speed | Slow (bundles everything) | Very fast (native ESM, HMR) | Fast (Turbopack) |
| Routing | Add React Router | Add React Router/TanStack Router | Built-in file-based (App Router) |
| Rendering | CSR only | CSR (SSR possible with setup) | SSR, SSG, ISR, RSC, streaming |
| Backend | None | None | Route handlers, Server Actions |

- Choose Vite for client-rendered SPAs, dashboards, and libraries; choose a framework (Next.js, React Router v7/Remix, TanStack Start) for SEO, performance, data fetching, and server rendering needs.
- The React docs now recommend starting with a framework, or Vite/Parcel/RSBuild if building from scratch.

**Code example**
```jsx
// Vite: npm create vite@latest my-app -- --template react
// src/main.jsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import App from "./App.jsx";

createRoot(document.getElementById("root")).render(
  <StrictMode>
    <App />
  </StrictMode>
);
```
```jsx
// Next.js App Router: app/page.jsx — a Server Component by default
export default async function Page() {
  const posts = await fetch("https://api.example.com/posts").then((r) => r.json());
  return (
    <ul>
      {posts.map((p) => (
        <li key={p.id}>{p.title}</li>
      ))}
    </ul>
  );
}
```

**Common pitfalls / follow-up questions**
- *Env variables?* Vite uses `import.meta.env.VITE_*`; CRA used `process.env.REACT_APP_*`; Next exposes `NEXT_PUBLIC_*` to the browser.
- *Why is a plain SPA bad for SEO?* Initial HTML is empty; crawlers/users wait for JS. SSR/SSG fixes this.
- *Pitfall:* using browser-only APIs (`window`) in server-rendered code throws; guard or use client components/effects.

---

### Q18. What is the difference between an element and a component?

**Short answer**
An element is a plain immutable object describing what to render (`{ type, props }`), created by JSX. A component is a function (or class) that receives props and returns elements; it is the blueprint, and elements are its output or usages.

**Detailed explanation**
- `<Button />` creates an element with `type: Button`; React later calls `Button(props)` to get more elements. `<div />` is an element with a string type (host element).
- Elements are cheap, immutable snapshots; you can't change their props after creation.
- A component *instance* (with state/hooks) exists in React's internal tree (fibers), distinct from the element objects.
- Reconciliation compares element `type` and `key` to decide update vs remount.
- Elements can be stored in variables, passed as props (`icon={<Icon />}`), and returned; components can be passed as values too (`as={Link}`), used with capitalized variable names.

**Code example**
```jsx
function Avatar({ src }) {
  return <img src={src} alt="" />;
}

// An element (just an object; Avatar hasn't run yet):
const element = <Avatar src="/a.png" />;
// ≈ { type: Avatar, props: { src: "/a.png" }, key: null, ... }

export default function Page({ Icon }) {
  // `Icon` is a component (passed as a value); <Icon /> creates an element
  return (
    <header>
      {element}
      <Icon />
    </header>
  );
}
```

**Common pitfalls / follow-up questions**
- *Is `<App />` a component or an element?* An element of type `App`; `App` is the component.
- *Why does an inline component definition cause remounts?* New function identity means a new `type` each render.
- *Pitfall:* `Component()` vs `<Component />`: only JSX creates a proper instance with its own hooks/state.

---

### Q19. What is prop drilling?

**Short answer**
Prop drilling is passing data through multiple intermediate components that don't use it just so a deeply nested component can receive it. It makes code verbose and tightly coupled, and can be resolved with composition, Context, or a state library.

**Detailed explanation**
- Small amounts over 1-2 levels are fine and keep data flow explicit; it's a problem when many layers merely forward props.
- Solutions, in order of preference:
  1. **Composition / children / slots:** pass already-built elements down so intermediates don't need the data.
  2. **Context API:** provide values to a subtree (theme, auth, locale). Consumers re-render on any value change, so split contexts and memoize values.
  3. **State management libraries:** Redux Toolkit, Zustand, Jotai with selector subscriptions for frequently-updating data.
  4. **Server data libraries:** TanStack Query/SWR let components fetch what they need.
- Context is not a replacement for all props; overuse hides dependencies and hampers reuse.

**Code example**
```jsx
import { createContext, use, useState } from "react";

const UserContext = createContext(null);

function Avatar() {
  const user = use(UserContext); // React 19; use useContext(UserContext) in 18
  return <img src={user.avatar} alt={user.name} />;
}

// Layout and Sidebar don't know about `user`
function Sidebar() { return <aside><Avatar /></aside>; }
function Layout() { return <main><Sidebar /></main>; }

export default function App() {
  const [user] = useState({ name: "Ada", avatar: "/ada.png" });
  return (
    <UserContext value={user}>{/* React 19: context itself is a provider */}
      <Layout />
    </UserContext>
  );
}
```

**Common pitfalls / follow-up questions**
- *Does Context fix performance?* No; all consumers re-render when the value changes. Memoize the value and split contexts.
- *Alternative without Context?* Composition: pass `<Avatar user={user} />` as a slot prop.
- *Pitfall:* putting rapidly-changing state (e.g., mouse position) in a global Context.

---

### Q20. What is lifting state up?

**Short answer**
Lifting state up means moving shared state to the closest common ancestor of the components that need it, then passing it down via props along with callbacks to update it. This keeps a single source of truth and keeps siblings in sync.

**Detailed explanation**
- If two siblings must reflect the same data, neither can own it; their parent owns it.
- Steps: (1) remove state from the children, (2) add it to the common parent, (3) pass value and setter/handler as props.
- Children become controlled components, simpler and more reusable.
- Lift only as high as necessary; too high causes unnecessary re-renders and prop drilling. The opposite move is "pushing state down" for state used by a single component.
- If lifting gets deep, consider Context, a reducer, or an external store.

**Code example**
```jsx
import { useState } from "react";

function TemperatureInput({ label, value, onChange }) {
  return (
    <label>
      {label}
      <input type="number" value={value} onChange={(e) => onChange(e.target.value)} />
    </label>
  );
}

const toF = (c) => (c * 9) / 5 + 32;
const toC = (f) => ((f - 32) * 5) / 9;
const round = (n) => (Number.isNaN(n) ? "" : Math.round(n * 100) / 100);

export default function Converter() {
  // Single source of truth: Celsius string
  const [celsius, setCelsius] = useState("");
  const fahrenheit = celsius === "" ? "" : round(toF(parseFloat(celsius)));

  return (
    <>
      <TemperatureInput label="°C" value={celsius} onChange={setCelsius} />
      <TemperatureInput
        label="°F"
        value={fahrenheit}
        onChange={(f) => setCelsius(f === "" ? "" : String(round(toC(parseFloat(f)))))}
      />
    </>
  );
}
```

**Common pitfalls / follow-up questions**
- *Why not sync two states with `useEffect`?* It causes extra renders and drift; derive one from the other.
- *How high should state go?* The nearest common parent of all consumers.
- *Pitfall:* lifting too far makes the whole subtree re-render on each change; use composition, `memo`, or move state back down.
