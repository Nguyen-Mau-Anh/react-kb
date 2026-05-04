# Topic 06 — Conditional Rendering & Lists: Key Prop, Reconciliation Impact

> **What / Why / How** — keys are not for you, they're for React. Get them wrong and you get silent bugs.

---

## 1. Conditional Rendering — Four Idioms

### Idiom 1: Ternary

```tsx
{isLoggedIn ? <Dashboard /> : <LoginForm />}
```

**Use when**: you need an else branch.

### Idiom 2: Logical AND (`&&`)

```tsx
{user && <UserCard user={user} />}
```

**Use when**: you only want to render something when truthy. **Watch the `0` trap**:

```tsx
// ❌ Renders the literal "0" when count === 0
{count && <Badge count={count} />}

// ✅ Coerce to boolean explicitly
{count > 0 && <Badge count={count} />}
{!!count && <Badge count={count} />}
{Boolean(count) && <Badge count={count} />}
```

### Idiom 3: Early return

```tsx
function UserPage({ user }: { user?: User }) {
  if (!user) return <Spinner />;
  if (user.banned) return <BanNotice />;

  return <Dashboard user={user} />;
}
```

**Use when**: there are guard conditions before the main render. Cleaner than nested ternaries.

### Idiom 4: Lookup object (replaces switch)

```tsx
const STATUS_VIEWS = {
  loading: <Spinner />,
  error: <ErrorBox />,
  success: <Results />,
  empty: <EmptyState />,
} as const;

function Page({ status }: { status: keyof typeof STATUS_VIEWS }) {
  return STATUS_VIEWS[status];
}
```

**Use when**: 4+ variants. More readable than chained ternaries; TypeScript narrows `status` to the union.

---

## 2. Lists — `Array.prototype.map`

### What

React renders arrays of React elements in order. The most common pattern is `.map()`:

```tsx
function TodoList({ todos }: { todos: Todo[] }) {
  return (
    <ul>
      {todos.map(todo => (
        <li key={todo.id}>{todo.text}</li>
      ))}
    </ul>
  );
}
```

### Why React requires `key` for list children

React's reconciliation algorithm (Topic 00) needs a way to match elements between the previous render and the next. Without `key`, React falls back to matching by **index** — which produces wrong results when items reorder, are inserted, or removed.

### The classic `key={index}` bug

```tsx
const [items, setItems] = useState(['Alice', 'Bob', 'Charlie']);

// ❌ Uses array index as key
{items.map((name, i) => (
  <li key={i}>
    {name}
    <input defaultValue={name} />
  </li>
))}

// User edits Charlie's input → "Charlie edited"
// User clicks "Remove Alice" → setItems(['Bob', 'Charlie'])
// What happens?
//   index 0: was Alice, now Bob → React keeps the Alice <li> mounted, swaps the text to "Bob"
//            BUT the <input> is the SAME DOM node — still shows the OLD text
//   index 1: was Bob, now Charlie → same bug
//   index 2: removed
// Result: Bob's input shows Alice's typing, Charlie's input shows Bob's typing.
```

### ✅ Fix: stable IDs

```tsx
type Item = { id: string; name: string };
const [items, setItems] = useState<Item[]>([
  { id: 'u1', name: 'Alice' },
  { id: 'u2', name: 'Bob' },
  { id: 'u3', name: 'Charlie' },
]);

{items.map(item => (
  <li key={item.id}>
    {item.name}
    <input defaultValue={item.name} />
  </li>
))}
```

Now React matches by `id`. When Alice is removed, the Bob and Charlie `<li>`s are kept (with their input state), and Alice's is unmounted.

### When `key={index}` is acceptable

Only when **all three** are true:
1. The list never reorders.
2. Items are never inserted or removed in the middle.
3. List items don't have internal state (no inputs, no animations, no `useState` inside).

In practice: a static rendered table from server data that you only display, never mutate. Even then, prefer real IDs.

---

## 3. Generating Keys When You Don't Have IDs

### From the data itself

```tsx
// ✅ Composite stable key
{events.map(e => (
  <li key={`${e.type}-${e.timestamp}`}>{e.message}</li>
))}
```

### `crypto.randomUUID()` at creation time, not render time

```tsx
// ❌ New UUID every render — destroys reconciliation
{items.map(item => <li key={crypto.randomUUID()}>{item}</li>)}

// ✅ Generate once when item is created
function addItem(text: string) {
  setItems(prev => [...prev, { id: crypto.randomUUID(), text }]);
}
```

### `nanoid` — when you need short collision-resistant IDs in browser

`nanoid@5` (15× smaller than uuid v4): `npm i nanoid` → `import { nanoid } from 'nanoid'; nanoid()` → `'V1StGXR8_Z5jdHi6B-myT'`. Used by tldraw, Excalidraw, NocoDB.

---

## 4. Fragment Keys for Multi-Element List Items

```tsx
// ❌ <Fragment> shorthand can't take a key
{items.map(item => (
  <>
    <dt>{item.term}</dt>
    <dd>{item.definition}</dd>
  </>
))}

// ✅ Long form supports key
{items.map(item => (
  <React.Fragment key={item.id}>
    <dt>{item.term}</dt>
    <dd>{item.definition}</dd>
  </React.Fragment>
))}
```

This is the only legitimate use of the long-form `<React.Fragment>` since React 16.2.

---

## 5. Filtering, Sorting, Empty States

```tsx
function TodoList({ todos, filter }: { todos: Todo[]; filter: 'all' | 'open' | 'done' }) {
  const visible = todos
    .filter(t => filter === 'all' || (filter === 'open' && !t.done) || (filter === 'done' && t.done))
    .sort((a, b) => a.createdAt.localeCompare(b.createdAt));

  if (visible.length === 0) {
    return <EmptyState message="No todos match this filter" />;
  }

  return (
    <ul>
      {visible.map(t => <TodoRow key={t.id} todo={t} />)}
    </ul>
  );
}
```

**Important**: `.filter` and `.sort` produce new array references every render. If you pass `visible` as a prop to a memoized child, wrap it in `useMemo`:

```tsx
const visible = useMemo(
  () => todos.filter(...).sort(...),
  [todos, filter]
);
```

This is covered in Topic 09.

---

## 6. Large Lists — When `.map()` Stops Scaling

For 1,000+ rows, rendering everything is wasteful. Real options:

| Library | Use case | Bundle | Notes |
|---------|----------|--------|-------|
| `@tanstack/react-virtual@3` | Virtualizing 10K–1M rows | ~5 KB gzipped | Headless, you build the markup |
| `react-window@1.8` | Simple virtualized lists | ~3 KB | Older, fewer features than tanstack |
| `react-virtuoso@4` | Auto-sized rows, chat/feed UIs | ~25 KB | Handles dynamic heights better than the others |
| Native CSS `content-visibility: auto` | "Skip rendering offscreen" | 0 KB | Browser-native, but no event/state benefit |

Real example with `@tanstack/react-virtual@3`:

```tsx
import { useVirtualizer } from '@tanstack/react-virtual';

function BigList({ items }: { items: Item[] }) {
  const parentRef = useRef<HTMLDivElement>(null);
  const v = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 48,
    overscan: 5,
  });

  return (
    <div ref={parentRef} style={{ height: 600, overflow: 'auto' }}>
      <div style={{ height: v.getTotalSize(), position: 'relative' }}>
        {v.getVirtualItems().map(vi => (
          <div
            key={items[vi.index].id}
            style={{ position: 'absolute', top: 0, transform: `translateY(${vi.start}px)`, height: vi.size }}
          >
            {items[vi.index].name}
          </div>
        ))}
      </div>
    </div>
  );
}
```

Only ~15 DOM nodes exist at any time even for a 100K-item list.

---

## 7. Avoid Index Math — Use the Item Itself

```tsx
// ❌ Array index gymnastics
{items.map((item, i) => (
  <Row item={item} onMove={() => move(i, i+1)} />
))}

// ✅ Pass the item or its ID directly
{items.map(item => (
  <Row item={item} onMove={() => move(item.id)} />
))}
```

Index-based handlers break when the array is filtered or sorted because the index a child captures may not match the source array after the next render.

---

## Summary

| Rule | Why |
|------|-----|
| Always use stable IDs as `key` | React matches list items by key for reconciliation |
| Never use `key={crypto.randomUUID()}` in render | New key every render = remount, lost state |
| Never use `key={index}` for mutable lists | Reordering or removal causes state to bind to wrong item |
| `<React.Fragment key={…}>` for multi-element items | Short `<>` syntax can't take props |
| Use `@tanstack/react-virtual@3` for 1K+ rows | Render only what's visible |
| Use `>` 0 not `&&` for numbers | `&&` renders the falsy `0` to the DOM |

---

## Further reading

- [React docs — Rendering Lists](https://react.dev/learn/rendering-lists)
- [React docs — Conditional Rendering](https://react.dev/learn/conditional-rendering)
- [TanStack Virtual docs](https://tanstack.com/virtual/latest)
- [Robin Wieruch — React keys](https://www.robinwieruch.de/react-list-key/)
