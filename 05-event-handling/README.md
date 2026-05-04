# Topic 05 — Event Handling: Synthetic Events, Delegation, Patterns

> **What / Why / How** — React's event system looks like the DOM but works differently underneath.

---

## 1. What Is React's Event System?

### What

When you write `<button onClick={handler}>`, React does **not** attach a DOM listener to that button. Instead:

- **React 16 and earlier**: one listener per event type attached to `document`.
- **React 17+**: one listener per event type attached to the **root container** (the element you passed to `createRoot`). This was a deliberate change to make multi-version React on one page possible (e.g., embedding a React 18 widget inside a React 17 app).

When the event fires, React walks the Fiber tree to find which component's handler should run, then calls it with a **SyntheticEvent** — a cross-browser wrapper around the native event.

```tsx
function Button() {
  function handleClick(e: React.MouseEvent<HTMLButtonElement>) {
    console.log(e.target);     // SyntheticEvent — works the same in Chrome, Firefox, Safari
    console.log(e.nativeEvent); // The underlying native MouseEvent
  }
  return <button onClick={handleClick}>Click</button>;
}
```

### Why event delegation?

| Approach | Listeners on a 1,000-row table | Memory |
|----------|-------------------------------|--------|
| Native DOM (`row.addEventListener('click', …)` per row) | 1,000 listeners | High |
| React event delegation | 1 root listener that dispatches to the right handler | Low |

Beyond memory, delegation also means dynamically-added rows automatically get their handlers — no re-attachment needed.

### Why React 17 moved listeners from `document` to root container

Real bug it fixed: pre-React 17, calling `e.stopPropagation()` on a `document.addEventListener('click', …)` set up by a non-React library (like a `react-modal` portal in React 16) wouldn't actually stop React from receiving the event, because React was *also* listening on `document`. Moving to the root container made `stopPropagation` behave intuitively.

---

## 2. Synthetic Events vs Native Events

```tsx
function handleClick(e: React.MouseEvent<HTMLButtonElement>) {
  // SyntheticEvent — same API across browsers
  e.preventDefault();
  e.stopPropagation();
  e.target;        // the element clicked
  e.currentTarget; // the element the handler is attached to (typed!)

  // Escape hatch — the underlying browser event
  e.nativeEvent;
}
```

### React 16 event pooling — gone in React 17+

In React 16, SyntheticEvents were **pooled**: after the handler returned, the event object was reused, and async access (`setTimeout(() => console.log(e.target))`) returned `null`. You had to call `e.persist()` to opt out.

**React 17+**: pooling is removed. You can read event properties asynchronously without `persist()`. If you see `persist()` in old tutorials, ignore it.

---

## 3. Common Patterns

### Pattern A: Inline arrow vs named handler

```tsx
// Inline — concise, fine for simple handlers
<button onClick={() => setCount(c => c + 1)}>+</button>

// Named — required when you need to test the function or memoize it
const handleClick = useCallback(() => setCount(c => c + 1), []);
<button onClick={handleClick}>+</button>
```

**Inline arrow doesn't tank performance** for small components. The "always memoize handlers" advice is largely outdated — only matters when the child is wrapped in `React.memo`. Topic 09 covers this.

### Pattern B: Passing arguments

```tsx
// ✅ Standard pattern
function ItemList({ items, onRemove }) {
  return (
    <ul>
      {items.map(item => (
        <li key={item.id}>
          {item.name}
          <button onClick={() => onRemove(item.id)}>×</button>
        </li>
      ))}
    </ul>
  );
}

// ✅ Alternative — data attributes for very large lists (avoid creating N closures)
<button data-id={item.id} onClick={(e) => onRemove(e.currentTarget.dataset.id!)}>×</button>
```

The closure-per-row approach is fine until you're rendering 10K+ rows. For virtualized lists with `@tanstack/react-virtual@3`, the closure overhead disappears anyway.

### Pattern C: Form submit — not button click

```tsx
// ❌ Click handler skips Enter-key submission, validation, native form behavior
<button onClick={submit}>Submit</button>

// ✅ Form submit — works with Enter key, plays with HTML validation
function ContactForm() {
  function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault();
    const data = new FormData(e.currentTarget);
    console.log(Object.fromEntries(data));
  }
  return (
    <form onSubmit={handleSubmit}>
      <input name="email" type="email" required />
      <button type="submit">Submit</button>
    </form>
  );
}
```

In Next.js 14+ Server Actions, you don't even need `e.preventDefault()`:

```tsx
// app/contact/page.tsx
async function submit(formData: FormData) {
  'use server';
  await db.insert(contacts).values({ email: formData.get('email') as string });
}

export default function Page() {
  return (
    <form action={submit}>
      <input name="email" type="email" />
      <button type="submit">Submit</button>
    </form>
  );
}
```

---

## 4. TypeScript Event Types — The Cheat Sheet

```tsx
// Mouse
onClick: React.MouseEvent<HTMLButtonElement>
onMouseEnter: React.MouseEvent<HTMLDivElement>

// Keyboard
onKeyDown: React.KeyboardEvent<HTMLInputElement>
//   if (e.key === 'Enter') ...

// Form
onSubmit: React.FormEvent<HTMLFormElement>
onChange: React.ChangeEvent<HTMLInputElement>
//   const value = e.target.value

// Focus
onFocus: React.FocusEvent<HTMLInputElement>

// Drag
onDragStart: React.DragEvent<HTMLDivElement>
//   e.dataTransfer.setData(...)

// Generic handler types (used in component props)
type Props = {
  onClick: React.MouseEventHandler<HTMLButtonElement>;
  onChange: React.ChangeEventHandler<HTMLInputElement>;
};
```

**`e.target` vs `e.currentTarget`**: `currentTarget` is typed correctly in TS; `target` is always `EventTarget` (you may need a cast). Prefer `currentTarget` when reading the element the handler is attached to.

---

## 5. Stopping Propagation, Capture Phase, Native Listeners

### `e.stopPropagation()` — stops bubbling within React's tree

```tsx
function Card({ onCardClick, onCloseClick }) {
  return (
    <div onClick={onCardClick}>
      <p>Card body</p>
      <button onClick={(e) => {
        e.stopPropagation();   // prevents the card's onClick from firing
        onCloseClick();
      }}>Close</button>
    </div>
  );
}
```

### Capture-phase handlers — `onClickCapture`, `onMouseDownCapture`

Append `Capture` to listen during the capture phase (before the event bubbles down to the target):

```tsx
<div onClickCapture={(e) => console.log('captured first')}>
  <button onClick={(e) => console.log('then this')}>Click</button>
</div>
```

Use case: global analytics that should record clicks regardless of `stopPropagation` in deeper handlers.

### Native (non-React) listeners — when you need them

Some events React doesn't synthesize, or where React's order is wrong:

```tsx
useEffect(() => {
  function handleResize() { setWidth(window.innerWidth); }
  window.addEventListener('resize', handleResize);
  return () => window.removeEventListener('resize', handleResize);
}, []);
```

Other examples: `IntersectionObserver`, `MutationObserver`, `ResizeObserver` (from the `@juggle/resize-observer` polyfill or native), `document.addEventListener('keydown', …)` for global shortcuts.

---

## 6. Real-World Pattern: Outside-Click Detection

Used by every dropdown, modal, and tooltip library (Radix UI, Headless UI, shadcn/ui).

```tsx
import { useEffect, useRef } from 'react';

function useOutsideClick<T extends HTMLElement>(onOutside: () => void) {
  const ref = useRef<T>(null);
  useEffect(() => {
    function handle(e: MouseEvent) {
      if (ref.current && !ref.current.contains(e.target as Node)) {
        onOutside();
      }
    }
    document.addEventListener('mousedown', handle);
    return () => document.removeEventListener('mousedown', handle);
  }, [onOutside]);
  return ref;
}

// Usage
function Dropdown() {
  const [open, setOpen] = useState(false);
  const ref = useOutsideClick<HTMLDivElement>(() => setOpen(false));
  return (
    <div ref={ref}>
      <button onClick={() => setOpen(o => !o)}>Menu</button>
      {open && <ul>...</ul>}
    </div>
  );
}
```

`mousedown` is preferred over `click` because some buttons (e.g., shadcn/ui's `<Select>`) close on `mousedown` to avoid double-fire bugs.

---

## 7. Pointer Events vs Mouse + Touch

For modern apps, prefer `onPointerDown` / `onPointerMove` / `onPointerUp` — they unify mouse, touch, and pen input. Used internally by `@dnd-kit/core@6` and `framer-motion@11`.

```tsx
<div
  onPointerDown={(e) => e.currentTarget.setPointerCapture(e.pointerId)}
  onPointerMove={handleDrag}
  onPointerUp={handleEnd}
/>
```

`setPointerCapture` ensures the same element gets all subsequent pointer events even if the cursor leaves — essential for drag handles.

---

## Summary

| Concept | One-line definition |
|---------|---------------------|
| SyntheticEvent | Cross-browser wrapper React passes to handlers; access native via `e.nativeEvent` |
| Event delegation | React attaches one root listener per event type, not per element |
| Pooling | Removed in React 17 — async event access works without `persist()` |
| `stopPropagation()` | Stops bubbling within React's tree |
| `*Capture` handlers | Run during capture phase (before bubble) |
| Pointer events | Unify mouse + touch + pen — preferred for drag/gesture libraries |

---

## Further reading

- [React docs — Responding to Events](https://react.dev/learn/responding-to-events)
- [React 17 changes — event delegation](https://legacy.reactjs.org/blog/2020/10/20/react-v17.html#changes-to-event-delegation)
- [MDN — PointerEvent](https://developer.mozilla.org/en-US/docs/Web/API/PointerEvent)
