# Topic 03 — State with useState: Local State, Batching, Derived State

> **What / Why / How** — `useState` is simple to use and easy to misuse. Master batching and derived state.

---

## 1. What is `useState`?

### What

`useState` is a React Hook (since React 16.8, 2019) that lets a function component hold a value across renders. React preserves the value between calls; updating it triggers a re-render.

```tsx
import { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);
  return (
    <button onClick={() => setCount(count + 1)}>
      Clicked {count} times
    </button>
  );
}
```

### Why React stores state outside the function

Function components run from top to bottom every render. If `count` were a normal `let count = 0` inside the function, it would reset on every render. React stores state in the **Fiber node** (covered in Topic 00) tied to that component instance — so `useState` reads back the latest value, not the initial one.

### Why NOT use class state (`this.state`, `this.setState`)?

- Classes can't use other hooks (`useEffect`, `useMemo`, `useTransition`).
- React Server Components (Next.js 14 App Router) require function components.
- See Topic 02 for the full comparison.

---

## 2. Updater Function vs Direct Value

```tsx
// ❌ Stale closure bug
function Counter() {
  const [count, setCount] = useState(0);

  function handleTripleClick() {
    setCount(count + 1);  // count = 0 in this closure
    setCount(count + 1);  // still 0
    setCount(count + 1);  // still 0 — final state: 1, not 3
  }
}

// ✅ Updater function — receives the latest state
function Counter() {
  const [count, setCount] = useState(0);

  function handleTripleClick() {
    setCount(c => c + 1);  // c = latest pending state
    setCount(c => c + 1);  // c = previous +1
    setCount(c => c + 1);  // c = previous +1 — final: 3
  }
}
```

**Rule of thumb**: if the new state depends on the previous state, use the updater function. Linters like `eslint-plugin-react-hooks` from `@reactjs/eslint-plugin-react-hooks` v5 won't catch this — you have to internalize it.

---

## 3. Batching — How React Groups State Updates

### What

React **batches** multiple `setState` calls inside a single event handler into one re-render. This is a performance optimization.

### Real example — multiple setState calls

```tsx
function Form() {
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');
  const [submitted, setSubmitted] = useState(false);

  function handleSubmit() {
    setName('');
    setEmail('');
    setSubmitted(true);
    // React 18: ONE re-render with all 3 updates applied.
    // React 17: ONE re-render in event handlers, but separate renders in promises/timeouts.
  }
}
```

### React 17 vs React 18 — automatic batching

| Scenario | React 17 | React 18 |
|----------|----------|----------|
| Inside React event handler (`onClick`, `onChange`) | ✅ Batched | ✅ Batched |
| Inside `setTimeout`, `Promise.then`, native event listener | ❌ NOT batched (one render per setState) | ✅ Batched |
| Opt-out of batching | n/a | `flushSync` from `react-dom` |

```tsx
// React 18 — automatic batching now works in async too
async function fetchAndUpdate() {
  const data = await fetch('/api/user').then(r => r.json());
  setUser(data);          // batched
  setLoading(false);      // batched
  // → ONE re-render in React 18; TWO in React 17.
}

// Force a synchronous render (rarely needed — DOM measurement, focus management)
import { flushSync } from 'react-dom';
flushSync(() => setX(1));   // forces a re-render right here
flushSync(() => setY(2));   // another re-render
```

`flushSync` is used by libraries like `@radix-ui/react-popover` 1.x to measure the DOM after a state change before paint.

---

## 4. Derived State — Don't Store What You Can Compute

### Anti-pattern: storing derived data in state

```tsx
// ❌ Bug factory — fullName goes stale every time firstName/lastName changes
function UserForm() {
  const [firstName, setFirstName] = useState('');
  const [lastName, setLastName] = useState('');
  const [fullName, setFullName] = useState(''); // ← derived state in state

  // Now you must remember to update fullName everywhere first/lastName changes.
}
```

### ✅ Compute on render

```tsx
function UserForm() {
  const [firstName, setFirstName] = useState('');
  const [lastName, setLastName] = useState('');
  const fullName = `${firstName} ${lastName}`.trim();
  // Always fresh, never stale.
}
```

### When to use `useMemo` for derived state

Only when the computation is expensive (sorting 10K rows, filtering large lists). For string concat or simple arithmetic, plain computation is faster than memoization (memo bookkeeping has its own cost). Topic 09 covers this in depth.

```tsx
// Real case — filtering 50K transactions
const filteredTxns = useMemo(
  () => transactions.filter(t => t.amount > minAmount),
  [transactions, minAmount]
);
```

---

## 5. State Object vs Multiple `useState` Calls

```tsx
// Approach A — multiple useState
const [name, setName] = useState('');
const [email, setEmail] = useState('');
const [age, setAge] = useState(0);

// Approach B — single object
const [form, setForm] = useState({ name: '', email: '', age: 0 });
setForm(f => ({ ...f, name: 'Anh' })); // must spread to keep other fields
```

**When approach A wins**: independent values that don't change together. Keeps re-renders surgical when paired with `React.memo`.

**When approach B wins**: tightly coupled fields (e.g., a form). Or use `useReducer` for complex updates — better than nested spreads.

**Real pick**: most React apps use approach A for 2–4 fields, then switch to `react-hook-form@7.51` (covered in Topic 07) for real forms.

---

## 6. Lazy Initial State — Avoid Expensive Init on Every Render

```tsx
// ❌ Runs JSON.parse on every render (only the first call's result is used, but the cost is paid)
const [todos, setTodos] = useState(JSON.parse(localStorage.getItem('todos') ?? '[]'));

// ✅ Lazy initializer — runs once
const [todos, setTodos] = useState(() => JSON.parse(localStorage.getItem('todos') ?? '[]'));
```

The function form is only called on mount.

---

## 7. State Reset Pattern — `key` Prop

To reset all state inside a component, change its `key`:

```tsx
function ProfilePage({ userId }: { userId: string }) {
  // When userId changes, React unmounts the old <Profile /> and mounts a fresh one.
  // All useState inside Profile resets to its initial value.
  return <Profile key={userId} userId={userId} />;
}
```

This is the official React-recommended pattern for resetting state on prop change ([react.dev — Resetting state with a key](https://react.dev/learn/preserving-and-resetting-state#resetting-state-with-a-key)). It avoids the old `componentDidUpdate` checks.

---

## 8. State Lives in the Tree — Where to Put It

| Location | When to use |
|----------|-------------|
| Component-local `useState` | UI-only state: open/closed, hovered, input value |
| Lifted to parent | Two siblings need to share it |
| Context (`useContext`) | Theme, current user — read by deep descendants |
| Zustand 4.5 / Jotai 2.x | Global state shared across many trees |
| TanStack Query v5 | **Server state** — never put server data in `useState` |

The single most common React mistake: storing fetched data in `useState` instead of TanStack Query. Topic 13 dives into why.

---

## Summary

| Concept | One-line definition |
|---------|---------------------|
| `useState` | Per-component-instance state, preserved across renders by Fiber |
| Updater function | `setX(prev => ...)` — required when new state depends on old |
| Automatic batching | React 18 batches setState calls in any context (incl. async) |
| Derived state | Compute on render, don't store; use `useMemo` only if expensive |
| `key` reset | Change a child's `key` to remount and reset all its state |
| Lazy init | `useState(() => expensiveInit())` runs once, not every render |

---

## Further reading

- [React docs — useState](https://react.dev/reference/react/useState)
- [React 18 blog — automatic batching](https://github.com/reactwg/react-18/discussions/21)
- [Kent C. Dodds — Don't sync state, derive it](https://kentcdodds.com/blog/dont-sync-state-derive-it)
