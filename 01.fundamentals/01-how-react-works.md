# Topic 00 — How React Works: VDOM, Fiber, Reconciliation

> **What / Why / How** — every concept explained with real alternatives compared by name

---

## 1. What is the Virtual DOM?

### What

The Virtual DOM (VDOM) is a lightweight JavaScript object tree that mirrors the structure of the real browser DOM. React keeps this tree in memory and uses it to calculate the minimal set of real DOM changes needed after a state update.

```js
// What you write (JSX):
<div className="card">
  <h1>{user.name}</h1>
  <p>{user.email}</p>
</div>

// What React holds in memory (simplified VDOM node):
{
  type: 'div',
  props: { className: 'card' },
  children: [
    { type: 'h1', props: {}, children: [user.name] },
    { type: 'p',  props: {}, children: [user.email] }
  ]
}
```

### Why — the real browser DOM is slow to touch

Direct DOM manipulation (e.g. `document.getElementById('name').textContent = newName`) is fast for a single operation, but:

- Triggering layout/reflow is expensive — the browser must recalculate geometry for every DOM change.
- Batch-updating is error-prone to do manually: you must track what changed across an entire render cycle.

**React's approach**: compute the diff entirely in JavaScript (cheap), then apply only the changed nodes to the real DOM (one surgical batch). This removes the bookkeeping burden from you.

### Why NOT always use a VDOM?

Not every framework does:

| Framework | Approach | Trade-off |
|-----------|----------|-----------|
| React 18 | VDOM + Fiber reconciler | Large runtime (~42 KB gzipped), predictable updates |
| Svelte 5 | Compiled reactive signals, no VDOM | ~10 KB runtime, faster fine-grained updates, smaller bundle |
| Vue 3 | VDOM + Proxy-based reactivity | Middle ground; smaller runtime than React |
| SolidJS 1.x | No VDOM, compiled signals directly to DOM | Fastest raw update speed, JSX-compatible |

**When React's VDOM wins**: large teams, massive ecosystems (npm packages, tooling), mature concurrent features. **When it loses**: extremely performance-sensitive UIs where every KB and microsecond matters (game UIs, huge data grids) — Svelte or SolidJS are better there.

---

## 2. What is React Fiber?

### What

Fiber (introduced in React 16, 2017) is React's internal reconciliation engine — the algorithm that decides *how* and *when* to update the UI. It replaced the old "stack reconciler" which was synchronous and blocking.

A **Fiber node** is a plain JavaScript object representing one unit of work (one component or DOM element). It forms a linked list (not a tree), which allows React to pause, resume, or abandon work mid-render.

```
FiberNode {
  type: MyComponent,        // function or class
  stateNode: ...,           // DOM node or component instance
  child: FiberNode,         // first child
  sibling: FiberNode,       // next sibling
  return: FiberNode,        // parent
  pendingProps: {},
  memoizedState: {},        // hooks live here
  lanes: ...,               // priority bits
}
```

### Why Fiber replaced the stack reconciler

The old reconciler (React 15 and earlier) walked the component tree synchronously in a single call-stack frame. If you had 1,000 components to update, it blocked the browser's main thread for the entire duration — causing visible jank (dropped frames below 60 fps).

**Fiber makes work interruptible:**

1. React breaks rendering into small units (one Fiber node at a time).
2. After each unit it checks: "Does the browser need to handle user input or paint?" — via `MessageChannel` scheduling (not `requestIdleCallback`, which has too coarse a granularity).
3. If yes, React pauses and resumes later.

Real scenario: A user types into a search box while React is re-rendering a large list. With the old reconciler, keystrokes feel laggy. With Fiber, React yields to the input event first.

### Two-phase rendering with Fiber

| Phase | Name | Interruptible? | What happens |
|-------|------|---------------|--------------|
| 1 | **Render / Reconcile** | ✅ Yes | React computes what changed (diffs the VDOM), building a "work-in-progress" Fiber tree |
| 2 | **Commit** | ❌ No | React flushes changes to the real DOM in one synchronous pass — interrupting here would leave the UI in a torn state |

---

## 3. What is Reconciliation?

### What

Reconciliation is the algorithm React uses to diff the old VDOM tree against the new one and determine the minimal set of DOM operations.

React uses two heuristics to make diffing O(n) instead of O(n³):

1. **Different element types → full subtree replace.** If `<div>` becomes `<section>`, React tears down the entire `<div>` subtree and builds a fresh `<section>` subtree. It doesn't try to reuse children.
2. **`key` prop → stable identity across renders.** For lists, React matches old and new nodes by `key`. Without `key`, React matches by index — causing bugs when list items are reordered.

### How — step by step in a real counter example

```jsx
// Before update:        After setState({ count: 1 }):
<div>                    <div>
  <p>Count: 0</p>   →     <p>Count: 1</p>   ← same type (p), update textContent only
  <button>+</button>      <button>+</button> ← unchanged, skip
</div>                   </div>
```

React's diff produces one operation: `p.textContent = "Count: 1"`. The `button` node is untouched.

### The `key` prop — why it exists and what happens without it

```jsx
// ❌ Bad — keyed by index
const items = ['Alice', 'Bob', 'Charlie'];
items.map((name, i) => <li key={i}>{name}</li>);
// If you remove 'Alice', React sees:
//   index 0: 'Alice' → 'Bob'   (updates DOM text)
//   index 1: 'Bob'   → 'Charlie' (updates DOM text)
//   index 2: 'Charlie' → removed  (removes DOM node)
// Result: 2 DOM mutations instead of 1. Input state inside <li> would also be wrong.

// ✅ Good — keyed by stable ID
items.map(item => <li key={item.id}>{item.name}</li>);
// React now matches by ID, detects 'Alice' was removed, does 1 DOM removal.
```

---

## 4. React 18 Concurrent Features — built on Fiber

Fiber's interruptibility unlocks React 18's concurrent features. These are real APIs, not theory:

### `startTransition` — mark a state update as non-urgent

```jsx
import { startTransition, useState } from 'react';

function SearchPage() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  function handleInput(e) {
    // Urgent: update input immediately
    setQuery(e.target.value);

    // Non-urgent: re-render the large results list — can be interrupted
    startTransition(() => {
      setResults(filterResults(e.target.value));
    });
  }

  return (
    <>
      <input value={query} onChange={handleInput} />
      <ResultsList results={results} />
    </>
  );
}
```

Without `startTransition`: typing feels laggy if `ResultsList` has 500 items. With it: the input stays responsive because React can interrupt the `ResultsList` render to handle keystrokes.

### `Suspense` + streaming SSR

React 18's streaming SSR (via `renderToPipeableStream` in Node.js) sends HTML to the browser in chunks. `<Suspense>` boundaries let React stream a placeholder first and fill in slow components later — without blocking the entire page.

```jsx
// Next.js 14 App Router uses this automatically.
// You get it for free when you use loading.tsx files.
export default function DashboardPage() {
  return (
    <div>
      <Header />                 {/* streams immediately */}
      <Suspense fallback={<Skeleton />}>
        <SlowDataComponent />    {/* streams when data is ready */}
      </Suspense>
    </div>
  );
}
```

---

## 5. Mental Model: what runs on every render?

```
setState() called
      ↓
React schedules a re-render (Fiber scheduler)
      ↓
[Render phase] — interruptible
  • React calls your component function again
  • Gets a new VDOM tree
  • Diffs new tree vs old tree (reconciliation)
  • Builds list of DOM changes ("effects")
      ↓
[Commit phase] — synchronous, not interruptible
  • Applies DOM mutations
  • Runs useLayoutEffect cleanup + setup
  • Paints to screen
  • Runs useEffect cleanup + setup
```

**Key insight**: your component function is just a pure function that returns a description of UI. React calls it; React decides when and how often. You don't call `render()` yourself — that mental shift is the core of learning React.

---

## Summary

| Concept | One-line definition |
|---------|---------------------|
| Virtual DOM | In-memory JS object tree; diffed to compute minimal real DOM ops |
| Fiber | React's internal unit-of-work node; enables interruptible rendering |
| Reconciliation | O(n) diffing algorithm using element type + `key` heuristics |
| `startTransition` | Marks update as non-urgent so urgent updates (input) aren't blocked |
| Commit phase | Synchronous DOM flush — never interrupted once started |

---

## Further reading

- [React source: ReactFiber.js](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiber.js)
- [React 18 release notes — concurrent features](https://react.dev/blog/2022/03/29/react-v18)
- [React docs — Rendering Lists (key prop)](https://react.dev/learn/rendering-lists)
