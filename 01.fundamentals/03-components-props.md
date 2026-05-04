# Topic 02 — Components & Props: Function Components, Prop Types, Composition

> **What / Why / How** — function components are the only way forward in React 18+.

---

## 1. What is a Component?

### What

A React component is a **function** (or, historically, a class) that takes a `props` object and returns a React element (JSX). Components are the unit of reuse in React.

```tsx
// Function component — the only style you should write in 2026
type ButtonProps = {
  label: string;
  onClick: () => void;
  variant?: 'primary' | 'secondary';
};

export function Button({ label, onClick, variant = 'primary' }: ButtonProps) {
  return (
    <button onClick={onClick} className={`btn btn-${variant}`}>
      {label}
    </button>
  );
}

// Usage
<Button label="Save" onClick={handleSave} variant="primary" />
```

### Why function components beat class components

| Feature | Class component | Function component |
|---------|----------------|--------------------|
| Hooks (useState, useEffect, …) | ❌ Cannot use | ✅ Native |
| Bundle size | Larger (`extends React.Component`, prototype) | Smaller |
| `this` binding bugs | ❌ Common (`onClick={this.handleClick.bind(this)}`) | ✅ No `this` |
| Lifecycle methods | `componentDidMount`, `componentDidUpdate`, `componentWillUnmount` | One unified `useEffect` |
| Server Components (Next.js 14 RSC) | ❌ Not supported | ✅ Supported |
| Future React features (use, useFormStatus, useActionState) | ❌ Hooks-only | ✅ Hooks-only |

**The official React team position (since React 17, 2020)**: do not write new class components. Class components are not deprecated but receive no new features. React Server Components, Suspense for data, `useTransition`, `use()` — none of them work in classes.

### Why NOT use class components even for "stateful complex stuff"?

The argument that "classes are better for complex state" was true in 2018. With `useReducer` (built into React) and Zustand 4 / Redux Toolkit 2 / TanStack Query v5, hooks handle every state pattern classes did — usually with less code and no `this` confusion.

---

## 2. Props — The Component API

### What

Props are read-only inputs to a component. They flow **one-way: parent → child**. A child cannot modify its props.

```tsx
type UserCardProps = {
  user: { id: string; name: string; email: string };
  onEdit: (id: string) => void;
  isActive: boolean;
};

function UserCard({ user, onEdit, isActive }: UserCardProps) {
  return (
    <div className={isActive ? 'card active' : 'card'}>
      <h2>{user.name}</h2>
      <p>{user.email}</p>
      <button onClick={() => onEdit(user.id)}>Edit</button>
    </div>
  );
}
```

### Why props are read-only

React's mental model: a component is a pure function of its props (and internal state). Mutating props would break:
- **Predictability** — same props should always produce the same output.
- **Memoization** — `React.memo` and `useMemo` rely on prop reference equality.
- **Time-travel debugging** — Redux DevTools, React DevTools profiler.

If a child needs to change parent state, the parent passes a callback prop (`onEdit` in the example). This is "lifting state up".

### Real use case — controlled `<Modal>` component

```tsx
type ModalProps = {
  open: boolean;
  onClose: () => void;
  title: string;
  children: React.ReactNode;
};

export function Modal({ open, onClose, title, children }: ModalProps) {
  if (!open) return null;
  return (
    <div className="modal-backdrop" onClick={onClose}>
      <div className="modal" onClick={(e) => e.stopPropagation()}>
        <header>
          <h2>{title}</h2>
          <button onClick={onClose}>×</button>
        </header>
        {children}
      </div>
    </div>
  );
}

// Parent owns state; Modal is purely controlled by props.
function App() {
  const [isOpen, setIsOpen] = useState(false);
  return (
    <>
      <button onClick={() => setIsOpen(true)}>Open</button>
      <Modal open={isOpen} onClose={() => setIsOpen(false)} title="Confirm">
        <p>Are you sure?</p>
      </Modal>
    </>
  );
}
```

---

## 3. Typing Props — TypeScript Patterns

### Inline `type` vs `interface`

```tsx
// Both work — pick one and be consistent.
type CardProps = { title: string; children: React.ReactNode };
interface CardProps { title: string; children: React.ReactNode }
```

**Project convention used by `shadcn/ui`, `Vercel`, `Next.js` examples**: `type` for component props, `interface` for public library APIs that consumers might extend via declaration merging.

### `React.PropsWithChildren` — when accepting `children`

```tsx
import type { PropsWithChildren } from 'react';

type CardProps = PropsWithChildren<{ title: string }>;
// Equivalent to: { title: string; children?: ReactNode }
```

Or write it inline:

```tsx
type CardProps = {
  title: string;
  children: React.ReactNode;
};
```

### Extending native HTML props — the `shadcn/ui` pattern

Real example from how `shadcn/ui`'s `Button` is built:

```tsx
import { ButtonHTMLAttributes, forwardRef } from 'react';

interface ButtonProps extends ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'default' | 'destructive' | 'ghost';
  size?: 'sm' | 'md' | 'lg';
}

export const Button = forwardRef<HTMLButtonElement, ButtonProps>(
  ({ variant = 'default', size = 'md', className, ...rest }, ref) => (
    <button
      ref={ref}
      className={`btn btn-${variant} btn-${size} ${className ?? ''}`}
      {...rest}
    />
  )
);
```

This pattern lets consumers pass any native `<button>` attribute (`disabled`, `type`, `aria-label`, etc.) without you re-declaring all 50 of them.

### Discriminated unions for variant props

```tsx
type AlertProps =
  | { variant: 'success'; message: string }
  | { variant: 'error';   message: string; retry: () => void };

function Alert(props: AlertProps) {
  if (props.variant === 'error') {
    return <button onClick={props.retry}>{props.message}</button>;
  }
  return <span>{props.message}</span>;
}
```

TypeScript narrows the `props` type inside the `if` block — no need for non-null assertions.

---

## 4. Composition — How React Replaces Inheritance

React explicitly recommends composition over inheritance ([React docs — Thinking in React](https://react.dev/learn/thinking-in-react)).

### Pattern 1: `children` as the slot

```tsx
function Layout({ children }: { children: React.ReactNode }) {
  return (
    <div className="layout">
      <Header />
      <main>{children}</main>
      <Footer />
    </div>
  );
}

<Layout>
  <DashboardPage />
</Layout>
```

### Pattern 2: Named slots via props

When you need multiple insertion points:

```tsx
type PageProps = {
  header: React.ReactNode;
  sidebar: React.ReactNode;
  content: React.ReactNode;
};

function Page({ header, sidebar, content }: PageProps) {
  return (
    <div className="page">
      <div className="page-header">{header}</div>
      <div className="page-sidebar">{sidebar}</div>
      <div className="page-content">{content}</div>
    </div>
  );
}

<Page
  header={<NavBar />}
  sidebar={<FilterPanel />}
  content={<UserList />}
/>
```

This is exactly how Remix's `<Outlet />` and Next.js 14's parallel routes work conceptually.

### Pattern 3: Specialization via component composition

```tsx
function Dialog({ children }: { children: React.ReactNode }) { /* base */ }

function ConfirmDialog({ message, onConfirm }: { message: string; onConfirm: () => void }) {
  return (
    <Dialog>
      <p>{message}</p>
      <button onClick={onConfirm}>Confirm</button>
    </Dialog>
  );
}
```

`ConfirmDialog` doesn't extend `Dialog`; it uses it. This is React's idiomatic alternative to OOP inheritance.

---

## 5. Naming and File Conventions

| Convention | Rule | Why |
|------------|------|-----|
| Component name | PascalCase (`UserCard`) | React requires it — JSX uses lowercase for HTML, uppercase for components |
| File name | `UserCard.tsx` (PascalCase) — used by Next.js examples, shadcn/ui | Matches the export, easy to grep |
| Folder per component | `UserCard/index.tsx` + `UserCard.test.tsx` | Encapsulates tests, styles, types |
| Default vs named export | Named exports preferred (TanStack, shadcn/ui) | Better autocomplete, easier refactor |

### Default export — when Next.js requires it

Next.js App Router **requires default exports** for `page.tsx`, `layout.tsx`, `loading.tsx`, `error.tsx`. Everywhere else, prefer named exports.

```tsx
// app/dashboard/page.tsx — must be default
export default function DashboardPage() {
  return <h1>Dashboard</h1>;
}
```

---

## Summary

| Concept | One-line definition |
|---------|---------------------|
| Function component | A function from props → React element; the only style for new code |
| Props | Read-only inputs; one-way data flow parent → child |
| Lifting state up | When two siblings need shared state, move it to their common parent |
| Composition | Use `children` and named slots instead of class inheritance |
| `forwardRef` + `...rest` | Standard pattern (used by shadcn/ui) for components that wrap native elements |

---

## Further reading

- [React docs — Passing Props to a Component](https://react.dev/learn/passing-props-to-a-component)
- [shadcn/ui Button source](https://github.com/shadcn-ui/ui/blob/main/apps/www/registry/default/ui/button.tsx)
- [TypeScript + React cheatsheet](https://react-typescript-cheatsheet.netlify.app/)
