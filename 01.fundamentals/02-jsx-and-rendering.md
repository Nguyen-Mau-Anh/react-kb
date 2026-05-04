# Topic 01 — JSX & Rendering: What JSX Compiles to, ReactDOM.createRoot

> **What / Why / How** — JSX is not magic; it's a function call with a babel transform.

---

## 1. What JSX Actually Is

### What

JSX is **syntactic sugar** over `React.createElement(type, props, ...children)`. The browser cannot run JSX directly — a compiler (Babel 7 with `@babel/preset-react`, or SWC bundled in Next.js 14, or esbuild in Vite 5) transforms it into JavaScript before execution.

```jsx
// What you write
const ui = <h1 className="title">Hello {name}</h1>;

// What Babel 7 + classic runtime emits (pre-React 17)
const ui = React.createElement('h1', { className: 'title' }, 'Hello ', name);

// What Babel 7 + automatic runtime emits (React 17+, default in CRA 5 / Next.js / Vite)
import { jsx as _jsx } from 'react/jsx-runtime';
const ui = _jsx('h1', { className: 'title', children: ['Hello ', name] });
```

The output is a plain JavaScript object — a **React element** — not a DOM node:

```js
{
  $$typeof: Symbol(react.element),
  type: 'h1',
  props: { className: 'title', children: ['Hello ', name] },
  key: null,
  ref: null
}
```

### Why JSX over alternatives

| Option | Example | Why React picked JSX |
|--------|---------|---------------------|
| `React.createElement` calls | `createElement('h1', null, 'Hi')` | Verbose; nested trees are unreadable |
| Tagged template literals (lit-html 3) | `` html`<h1>${name}</h1>` `` | No static analysis; IDE autocomplete is weaker |
| HyperScript (`h('h1', ...)` — used by Mithril 2, Cycle.js) | `h('h1', {}, name)` | Concise but unfamiliar to HTML-trained devs |
| **JSX** | `<h1>{name}</h1>` | HTML-like syntax, full TS type-checking via `tsx`, IDE autocomplete |

JSX won because it preserves the visual tree structure of HTML while giving you the full power of JavaScript expressions inside `{}`.

### Why NOT use a runtime template language (Vue SFC, Angular templates)?

Vue 3's SFC template (`<template><h1>{{ name }}</h1></template>`) and Angular's HTML templates are parsed by a separate compiler. Trade-offs vs JSX:

- ✅ Vue/Angular templates can be statically optimized by the compiler (e.g., Vue 3's hoisting of static nodes).
- ❌ Conditional logic feels grafted on (`v-if`, `*ngIf`) instead of native JS.
- ❌ Type-checking inside templates requires extra tooling (Volar for Vue, Angular Language Service).

JSX is just JavaScript, so anything you can do in JS works inside `{}`: `.map()`, ternaries, function calls, optional chaining.

---

## 2. The Two JSX Runtimes

### Classic runtime (pre-React 17)

Required `import React from 'react'` in every JSX file because the output called `React.createElement`. If you forgot the import, you got the dreaded `'React' must be in scope` ESLint error.

### Automatic runtime (React 17+, the modern default)

Babel auto-imports from `react/jsx-runtime`. **You no longer need `import React`** unless you use `React.SomeAPI` directly.

Configured in:
- **Next.js 14**: automatic by default (`next.config.js` uses SWC).
- **Vite 5 + `@vitejs/plugin-react@4`**: automatic by default.
- **Babel 7**: set `"runtime": "automatic"` in `@babel/preset-react` options.
- **TypeScript 5.x**: set `"jsx": "react-jsx"` in `tsconfig.json`.

```json
// tsconfig.json — automatic runtime
{
  "compilerOptions": {
    "jsx": "react-jsx",        // emits jsx-runtime imports
    "jsxImportSource": "react" // or "@emotion/react", "preact", etc.
  }
}
```

---

## 3. From JSX to Pixels — The Full Render Pipeline

Real scenario: a Vite + React 18 SPA loading a `<TodoApp />` component.

```
[1] Build time (Vite + esbuild)
    src/App.tsx (JSX) ──→ esbuild transforms JSX ──→ dist/assets/index-abc123.js
                                                      (plain JS calling _jsx)

[2] Browser runtime
    index.html loads index-abc123.js
       │
       ▼
    main.tsx executes:
       ReactDOM.createRoot(document.getElementById('root')!).render(<App />)
       │
       ▼
    React creates a Fiber root, schedules initial render
       │
       ▼
    [Render phase] React calls App() → returns React elements (the JSX object tree)
       │
       ▼
    [Commit phase] React translates elements to real DOM nodes via document.createElement('div'), .appendChild, etc.
       │
       ▼
    Browser paints
```

### `ReactDOM.createRoot` vs the legacy `ReactDOM.render`

```jsx
// ❌ Legacy API — deprecated in React 18, removed in a future major
import ReactDOM from 'react-dom';
ReactDOM.render(<App />, document.getElementById('root'));

// ✅ Modern API — React 18+
import { createRoot } from 'react-dom/client';
const root = createRoot(document.getElementById('root')!);
root.render(<App />);
```

**Why the change?** `createRoot` enables React 18's concurrent rendering (`startTransition`, automatic batching, `<Suspense>` for data). The old `ReactDOM.render` runs in legacy mode where these features are disabled or partial.

### `createRoot` vs `hydrateRoot` — when each fits

| API | When to use | Real-world example |
|-----|-------------|--------------------|
| `createRoot` | Pure CSR (Vite SPA, Create React App) | Vite-built dashboard rendered fresh in the browser |
| `hydrateRoot` | SSR/SSG output, the server already produced HTML | Next.js Pages Router, Remix, Gatsby — browser attaches event listeners to existing DOM |

```jsx
// SSR hydration — Next.js Pages Router does this internally
import { hydrateRoot } from 'react-dom/client';
hydrateRoot(document.getElementById('root')!, <App />);
```

Mismatched HTML between server render and client hydration produces the famous `Hydration failed because the initial UI does not match` error — the #1 React 18 SSR bug. Causes: random IDs, `Date.now()`, `Math.random()`, browser-only APIs called during render.

---

## 4. Rules and Gotchas of JSX

### Must return a single root

```jsx
// ❌ JSX expressions must have one parent
return <h1>Hi</h1><p>Bye</p>;

// ✅ Wrap in a fragment (no extra DOM node)
return <><h1>Hi</h1><p>Bye</p></>;

// ✅ Or use React.Fragment for keyed lists
return <React.Fragment key={id}>...</React.Fragment>;
```

### Attribute name differences

JSX uses JavaScript property names, not HTML attribute names:

| HTML | JSX | Why |
|------|-----|-----|
| `class` | `className` | `class` is a JS reserved word |
| `for` | `htmlFor` | `for` is a JS reserved word |
| `tabindex` | `tabIndex` | DOM property is camelCase |
| `onclick` | `onClick` | React's synthetic event system |
| `style="color: red"` | `style={{ color: 'red' }}` | Style is an object, not a string |

### Children behavior

- Strings, numbers → rendered as text.
- `null`, `undefined`, `false`, `true` → rendered as nothing (great for conditionals).
- Arrays → rendered in order (each item needs a stable `key`).
- Objects → throw `Objects are not valid as a React child`.

```jsx
// Conditional rendering using JSX falsy behavior
{user && <UserCard user={user} />}      // renders only if user is truthy
{count > 0 && <Badge count={count} />}  // ⚠ if count === 0, renders "0"!

// Safer:
{count > 0 ? <Badge count={count} /> : null}
```

The `count > 0 && <Badge />` bug is one of the most common React mistakes — `0 && X` evaluates to `0`, and JSX renders the literal `0` to the DOM.

---

## 5. JSX Type-Checking with TypeScript

Setup:

```json
// tsconfig.json
{
  "compilerOptions": {
    "jsx": "react-jsx",
    "types": ["react", "react-dom"]
  }
}
```

You get:

- IntelliSense for every HTML attribute via `@types/react` (e.g., `<input type="numbr" />` errors).
- Component prop type checking (covered in topic 02).
- Event handler types: `onClick: React.MouseEventHandler<HTMLButtonElement>`.

---

## Summary

| Concept | One-line definition |
|---------|---------------------|
| JSX | Syntactic sugar that compiles to `_jsx(type, props)` calls returning React elements |
| React element | Plain JS object describing UI — not a DOM node |
| Automatic runtime | React 17+ default, removes the `import React` requirement |
| `createRoot` | React 18 entry point for CSR, enables concurrent features |
| `hydrateRoot` | React 18 entry point for SSR/SSG, attaches to server-rendered HTML |
| Hydration mismatch | When server HTML differs from client first render — fix nondeterministic code in render |

---

## Further reading

- [React docs — Writing Markup with JSX](https://react.dev/learn/writing-markup-with-jsx)
- [React 17 blog — New JSX Transform](https://legacy.reactjs.org/blog/2020/09/22/introducing-the-new-jsx-transform.html)
- [React 18 docs — createRoot](https://react.dev/reference/react-dom/client/createRoot)
