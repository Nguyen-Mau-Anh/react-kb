# Hooks — 04. useMemo & useCallback: When Memoization Actually Pays Off

> **What / Why / How** — most `useMemo`/`useCallback` calls in real codebases are pure overhead. The React Compiler is going to delete them.

---

## 1. What These Hooks Do

### What

```tsx
// useMemo — memoize the result of a computation
const sortedItems = useMemo(() => items.slice().sort(cmp), [items]);

// useCallback — memoize a function reference (same as useMemo for a function)
const handleClick = useCallback(() => doThing(id), [id]);
// equivalent to: useMemo(() => () => doThing(id), [id])
```

Both hooks cache a value across renders and only recompute when one of the deps changes (compared with `Object.is`, same rule as `useEffect`).

### Why React provides them

Three concrete situations where memoization avoids real work:

1. **Expensive computation** that would re-run every render (sorting 50K rows, parsing a 1MB JSON, computing a Three.js geometry).
2. **Stable reference** for an object/array passed to a memoized child or to a `useEffect` deps array.
3. **Stable callback** for the same reason — child wrapped in `React.memo` keeps the same handler ref across renders.

---

## 2. Why Most useMemo / useCallback Calls Are Pointless

### The hidden cost

Memoization is not free. Every `useMemo` / `useCallback` adds:
- A linked list entry on the Fiber's hook chain.
- A deps array allocation per render.
- An equality check per render.

For a `() => x + y` callback or a `useMemo(() => [a, b], [a, b])`, the bookkeeping costs more than the work it skips.

### The real condition for memoization to help

`useMemo` only saves time when **the work it skips is more expensive than the bookkeeping**, AND the deps actually stay equal across renders. Both conditions must be true.

### Real benchmark — when `useMemo` HURTS

```tsx
// A child component re-renders 1000 times. Each render:
const items = useMemo(() => [a, b, c], [a, b, c]);  // ❌ slower than literal
const items = [a, b, c];                            // ✅ faster
```

Allocating a 3-element array is ~20 nanoseconds. The hook bookkeeping is ~100 ns. Net loss every render.

---

## 3. When useMemo Actually Helps

### Case A: expensive computation

```tsx
function TransactionList({ transactions, minAmount }: Props) {
  // 50K transactions, filter is O(n) and runs on every render of the parent
  const filtered = useMemo(
    () => transactions.filter(t => t.amount > minAmount),
    [transactions, minAmount]
  );
  return <Table rows={filtered} />;
}
```

Without memo: 50K iterations on every parent re-render (e.g. every keystroke in a sibling input). With memo: 0 iterations until `transactions` or `minAmount` actually changes.

### Case B: stable reference for a memoized child

```tsx
const Row = React.memo(function Row({ data, onSelect }: RowProps) { ... });

function List({ items }: { items: Item[] }) {
  // ❌ New onSelect every render → React.memo on Row is useless, all rows re-render
  return items.map(item => (
    <Row data={item} onSelect={() => select(item.id)} />
  ));

  // ✅ Stable handler — Rows skip re-render when items haven't changed
  const onSelect = useCallback((id: string) => select(id), []);
  return items.map(item => (
    <Row data={item} onSelect={onSelect} />
  ));
}
```

The `useCallback` only matters because `Row` is wrapped in `React.memo`. If `Row` weren't memoized, `useCallback` would be wasted effort.

### Case C: stable deps for useEffect / context value

```tsx
// ❌ filters is a new object reference every render → effect runs every render
const filters = { status: 'active', priority: 'high' };
useEffect(() => loadData(filters), [filters]);

// ✅ Stable reference
const filters = useMemo(() => ({ status: 'active', priority: 'high' }), []);
useEffect(() => loadData(filters), [filters]);

// ✅✅ Or move it out of the component if it's static
const FILTERS = { status: 'active', priority: 'high' };
useEffect(() => loadData(FILTERS), []);
```

Same idea for context provider values — covered in `02.hooks/03-useref-usecontext.md`.

---

## 4. Don't Use useMemo for Object Identity Tricks

```tsx
// ❌ Cargo-cult — wrapping a primitive in useMemo for "stability"
const enabled = useMemo(() => count > 0, [count]);

// ✅ Just compute it
const enabled = count > 0;
```

Primitives are compared by value with `Object.is`. They're already "stable enough" for any deps array.

Only objects, arrays, functions, and class instances need `useMemo` for reference stability.

---

## 5. The React Compiler — Why This Topic Has a Shelf Life

The **React Compiler** (formerly "React Forget", official preview shipped May 2024) auto-memoizes function components and hook outputs at build time. With it enabled in `babel-plugin-react-compiler@0.x`:

```tsx
// You write this:
function Search({ items }: { items: Item[] }) {
  const filtered = items.filter(i => i.active);
  return <List items={filtered} />;
}

// The compiler emits roughly:
function Search({ items }: { items: Item[] }) {
  const filtered = $.cache(items, () => items.filter(i => i.active));
  return <List items={filtered} />;
}
```

Status (mid-2026):
- Used in production at Meta on instagram.com and other Meta surfaces.
- Stable RC for community use; recommended for **new** Next.js 15 / React 19 projects via `next.config.js`'s `experimental.reactCompiler`.
- React team's official guidance: **stop adding `useMemo` / `useCallback` manually** in projects where you intend to enable the compiler. Remove them once enabled.

Real implication: in a 2026 React 19 + compiler codebase, `useMemo` and `useCallback` should be rare and intentional. The compiler handles the boring 90%.

---

## 6. The `React.memo` HOC — Different Beast

`React.memo` wraps a component and skips its render if its props are shallow-equal to the previous render.

```tsx
const Row = React.memo(function Row({ data, onSelect }: RowProps) { ... });
```

### When React.memo helps

- Child renders frequently with the same props because the parent re-renders for unrelated reasons.
- Child is expensive (large subtree, heavy computation in render).

### When React.memo hurts

- Child is cheap (single `<div>`).
- Props are inline objects/arrays/functions that change every parent render anyway — `memo` does the comparison and always falls through.

### The shallow comparison gotcha

```tsx
// ❌ user is a new object every render → memo never bails out
<UserCard user={{ name, email }} />

// ✅ stable user reference (or pass primitives separately)
<UserCard name={name} email={email} />
```

For complex props you can pass a custom comparator:

```tsx
const Row = React.memo(RowImpl, (prev, next) => prev.data.id === next.data.id);
```

But once you're hand-writing comparators, you're past the point where the compiler would do better.

---

## 7. Decision Tree (Today, Without the Compiler)

```
Need to memoize something?
├─ Is it an expensive computation (sort 50K, parse big JSON, etc.)?
│   └─ YES → useMemo
├─ Is it a function/object/array passed to React.memo'd child?
│   └─ YES → useCallback / useMemo for reference stability
├─ Is it a function/object/array used in another hook's deps?
│   └─ YES → useMemo / useCallback (or move it out of the component)
└─ None of the above
    └─ Don't memoize. The bookkeeping costs more than it saves.
```

---

## 8. Profiling First — Don't Guess

Real workflow before adding memoization:

1. Open React DevTools → Profiler tab.
2. Click record, perform the slow interaction.
3. Look at the flame chart. Find components with high "self time" or marked yellow/red.
4. Only memoize what the profiler proves is hot.

Adding `useMemo` "just in case" is the React equivalent of premature optimization.

---

## Summary

| Hook | Use when | Don't use when |
|------|----------|----------------|
| `useMemo` | Computation > ~1ms; or stable reference for memo'd child / hook deps | Cheap primitives, simple values |
| `useCallback` | Function passed to `React.memo`'d child or hook deps | Inline handlers on plain DOM elements |
| `React.memo` | Child renders often with stable props and is expensive | Trivial children; props are new every render anyway |

| Rule | Why |
|------|-----|
| Profile first | Most memoization is wasted bookkeeping |
| Move static values outside the component | `const FILTERS = {...}` beats `useMemo(() => ({...}), [])` |
| Plan for the React Compiler | New code in React 19 + compiler shouldn't need manual memo |

---

## Further reading

- [React docs — useMemo](https://react.dev/reference/react/useMemo)
- [React docs — useCallback](https://react.dev/reference/react/useCallback)
- [React Compiler announcement (May 2024)](https://react.dev/blog/2024/05/15/react-19-upgrade-guide)
- [Kent C. Dodds — When to useMemo and useCallback](https://kentcdodds.com/blog/usememo-and-usecallback)
