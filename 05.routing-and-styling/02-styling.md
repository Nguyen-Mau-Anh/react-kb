# Routing & Styling — 02. Styling: Tailwind CSS 3 vs CSS Modules vs styled-components 6

> **What / Why / How** — three real options. Pick by team, project size, and runtime cost.

---

## 1. The Five Real Options in 2026

| Approach | Real package | Style | Where it shines |
|----------|--------------|-------|-----------------|
| **Tailwind CSS 3** | `tailwindcss@3.4` + `autoprefixer@10` + `postcss@8` | Atomic utility classes | Most production apps; default in shadcn/ui, Vercel, Linear |
| **CSS Modules** | Built into Vite 5, Next.js 14, CRA | Locally-scoped class names | Teams that prefer "real CSS files"; design-system internals |
| **styled-components 6** | `styled-components@6.1` | CSS-in-JS, runtime injection | Legacy codebases; Next.js Pages Router styled-components apps |
| **Vanilla Extract** | `@vanilla-extract/css@1.15` | Type-safe CSS-in-TS, zero runtime | Strong-typing fans; Twilio Paste, Shopify Hydrogen |
| **Plain CSS / SCSS** | `sass@1.77` if SCSS | Global stylesheets | Tiny apps, marketing pages |

This file covers the top three by current adoption.

The **honorable mention** at the end covers Vanilla Extract briefly because it's the most interesting modern alternative.

---

## 2. Tailwind CSS 3 — The Default Pick

### Why Tailwind beats the alternatives in 2026

- **Zero runtime cost** — class names compile to a single static CSS file at build time.
- **No naming bikeshedding** — no need to invent `.user-card`, `.user-card__title`, `.user-card--featured` BEM names.
- **Dead-code elimination** — only utilities actually referenced in source end up in the output (Tailwind's JIT, default since v3).
- **Massive ecosystem** — `tailwindcss@3.4` is what `shadcn/ui`, `headlessui@2.0`, `next-themes@0.3`, `radix-ui` examples, `@vercel/examples`, and `create-t3-app@7` all assume.
- **Editor support** — `bradlc.vscode-tailwindcss` autocompletes every utility, shows the generated CSS on hover, and warns on duplicates/conflicts.

### Why people complain about Tailwind (and why it's mostly fine anyway)

- "It looks ugly in JSX." It does. You're trading visual ergonomics for build/runtime simplicity. With 100+ commercial design systems built on it, this trade is well-validated.
- "Long class strings." Use `clsx@2.1` (or `tailwind-merge@2.3` for resolving conflicts) and component abstractions.
- "I have to remember the names." `bradlc.vscode-tailwindcss` autocompletes everything; you're not memorizing.

### Setup (Vite 5)

```bash
npm i -D tailwindcss@3.4 postcss@8 autoprefixer@10
npx tailwindcss init -p
```

```js
// tailwind.config.js
export default {
  content: ['./index.html', './src/**/*.{js,ts,jsx,tsx}'],
  theme: { extend: {} },
  plugins: [],
};
```

```css
/* src/index.css */
@tailwind base;
@tailwind components;
@tailwind utilities;
```

Setup in Next.js 14 is identical except `content: ['./app/**/*.{js,ts,jsx,tsx}']`.

### Real component pattern — `clsx` + `tailwind-merge`

Conditional + override-safe class merging is the only hard part of using Tailwind in components:

```bash
npm i clsx@2.1 tailwind-merge@2.3
```

```ts
// utils/cn.ts — used by every shadcn/ui component
import { clsx, type ClassValue } from 'clsx';
import { twMerge } from 'tailwind-merge';

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

```tsx
import { cn } from './utils/cn';

type ButtonProps = {
  variant?: 'primary' | 'secondary';
  size?: 'sm' | 'md' | 'lg';
  className?: string;
  children: React.ReactNode;
} & React.ButtonHTMLAttributes<HTMLButtonElement>;

export function Button({ variant = 'primary', size = 'md', className, ...rest }: ButtonProps) {
  return (
    <button
      className={cn(
        'inline-flex items-center justify-center rounded font-medium transition-colors',
        // variants
        variant === 'primary'   && 'bg-blue-600 text-white hover:bg-blue-700',
        variant === 'secondary' && 'bg-gray-200 text-gray-900 hover:bg-gray-300',
        // sizes
        size === 'sm' && 'px-2 py-1 text-sm',
        size === 'md' && 'px-3 py-2 text-base',
        size === 'lg' && 'px-4 py-3 text-lg',
        // caller override last so it wins
        className
      )}
      {...rest}
    />
  );
}
```

Why `tailwind-merge`? Without it, `cn('p-4', 'p-2')` produces `"p-4 p-2"` and the cascade decides — ambiguous. `twMerge` resolves that to `"p-2"` deterministically. Used by `shadcn/ui`'s `cn` helper verbatim.

### Variant managers — `class-variance-authority@0.7`

For complex multi-variant components, `class-variance-authority@0.7` (CVA) is the idiomatic abstraction. `shadcn/ui`'s Button is built on it:

```ts
import { cva, type VariantProps } from 'class-variance-authority';

const buttonVariants = cva(
  'inline-flex items-center justify-center rounded font-medium transition-colors',
  {
    variants: {
      variant: {
        primary:   'bg-blue-600 text-white hover:bg-blue-700',
        secondary: 'bg-gray-200 text-gray-900 hover:bg-gray-300',
        ghost:     'hover:bg-gray-100',
      },
      size: {
        sm: 'px-2 py-1 text-sm',
        md: 'px-3 py-2 text-base',
        lg: 'px-4 py-3 text-lg',
      },
    },
    defaultVariants: { variant: 'primary', size: 'md' },
  }
);

type Props = React.ButtonHTMLAttributes<HTMLButtonElement> & VariantProps<typeof buttonVariants>;
export function Button({ variant, size, className, ...rest }: Props) {
  return <button className={cn(buttonVariants({ variant, size }), className)} {...rest} />;
}
```

### Theming — design tokens via CSS variables

```css
/* index.css */
:root {
  --color-bg: 255 255 255;
  --color-fg: 17 24 39;
}
.dark {
  --color-bg: 17 24 39;
  --color-fg: 255 255 255;
}
```

```js
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      colors: {
        bg: 'rgb(var(--color-bg) / <alpha-value>)',
        fg: 'rgb(var(--color-fg) / <alpha-value>)',
      },
    },
  },
};
```

`bg-bg`, `text-fg` now respect dark mode set by toggling the `.dark` class on `<html>`. Pair with `next-themes@0.3` for the toggle logic.

### Tailwind 4 — when it'll matter

`tailwindcss@4` is in alpha. It's a Lightning CSS rewrite, drops PostCSS, simplifies config to a single `@theme` block in CSS. **Don't migrate yet** — wait for stable. v3 stays the production answer through 2026.

---

## 3. CSS Modules — The "Real CSS" Option

### What

A `.module.css` file's classes are scoped: at build time, `.title` becomes `.UserCard_title__h2x9p`. Imports return an object of original-name → mangled-name.

```css
/* UserCard.module.css */
.title {
  font-size: 1.5rem;
  font-weight: bold;
}
.title.featured { color: gold; }
```

```tsx
import styles from './UserCard.module.css';

function UserCard({ featured }: { featured: boolean }) {
  return (
    <h2 className={`${styles.title} ${featured ? styles.featured : ''}`}>
      Hello
    </h2>
  );
}
```

### Why pick CSS Modules

- **Plain CSS** — no new syntax, no JS-to-CSS bridge, your designers can edit files directly.
- **Built into every modern bundler** — Vite 5, Next.js 14 (both routers), Webpack 5 — no install, no config beyond the `.module.css` extension.
- **Zero runtime** — same as Tailwind, just expressed differently.
- **No naming collisions** — file-scoped automatically.

### Why pick something else

- **Styles live in a separate file** from the component → context switching while editing.
- **No design-system primitives** — you re-invent spacing scale, color tokens, breakpoints from scratch.
- **No utility-first speed** — you write `.button { padding: 0.5rem 1rem; ... }` instead of `px-4 py-2`.

### When CSS Modules are still the right answer

- Your team has strong CSS expertise and views Tailwind as noise.
- You're shipping a design system *internals* that wraps every utility behind a semantic class.
- You're integrating with a legacy CSS architecture that already uses BEM.

### Conditional classes — same `clsx`

```tsx
import clsx from 'clsx';
import styles from './Card.module.css';

<div className={clsx(styles.card, isActive && styles.active, className)} />
```

---

## 4. styled-components 6 — Legacy CSS-in-JS

### Where styled-components fits in 2026

`styled-components@6.1` is **stable but no longer the recommended modern pick**. The two reasons:

1. **Runtime cost.** Every styled component injects styles at runtime via `<style>` tags. For a 200-component page, that's measurable parsing/serialization work. Tailwind's static CSS is faster.
2. **Next.js 14 App Router compatibility is awkward.** Styled-components requires special `StyleSheetManager` setup in Server Components, runs as a client boundary, and the official Next.js docs cover the workaround in detail because it's *not* automatic. Tailwind needs zero special handling.

### When it still makes sense

- You're maintaining a styled-components codebase. Migration is expensive; don't migrate without a reason.
- You need truly dynamic styles based on JS values (animation curves, computed colors). Tailwind handles this through arbitrary-value classes (`bg-[hsl(var(--accent))]`) but styled-components is more ergonomic for math-heavy cases.
- You're shipping a component library where consumers may not have Tailwind installed.

### Quick example

```bash
npm i styled-components@6.1
```

```tsx
import styled, { css } from 'styled-components';

const Button = styled.button<{ $variant?: 'primary' | 'secondary' }>`
  padding: 0.5rem 1rem;
  border-radius: 0.25rem;
  font-weight: 500;
  ${(p) => p.$variant === 'primary' && css`
    background: #2563eb;
    color: white;
  `}
  ${(p) => p.$variant === 'secondary' && css`
    background: #e5e7eb;
    color: #111827;
  `}
`;

<Button $variant="primary">Save</Button>
```

The `$`-prefix is v6's transient-prop convention — props starting with `$` aren't forwarded to the DOM. v5 used a `$` config; v6 made it the default.

### Why not `emotion` instead?

`@emotion/react@11` and `@emotion/styled@11` are very similar to styled-components, with slightly better performance and a smaller bundle. The decision is mostly cosmetic. If you have to pick one CSS-in-JS lib for a new project, pick `@emotion/styled@11` — but really, pick Tailwind.

---

## 5. Honorable Mention — Vanilla Extract 1.15

The most interesting non-Tailwind alternative for new projects. CSS-in-TypeScript that compiles to static CSS at build time:

```ts
// Button.css.ts
import { style } from '@vanilla-extract/css';

export const button = style({
  padding: '0.5rem 1rem',
  borderRadius: 4,
  ':hover': { opacity: 0.9 },
});
```

```tsx
import { button } from './Button.css';
<button className={button}>Save</button>
```

Pros: zero-runtime, type-safe (every class name is checked at compile time), supports themes via `createTheme`. Cons: smaller ecosystem than Tailwind, more boilerplate per component, fewer hire-able engineers know it.

Used by Twilio Paste, Shopify Hydrogen, and several internal tooling teams.

---

## 6. Combining Approaches

In real projects you often mix:

| Layer | Tool |
|-------|------|
| Layout/spacing/typography utilities | Tailwind |
| Animations and complex selectors | CSS Modules or `@layer` directives in Tailwind |
| One-off interactive states with computed values | inline `style={{}}` or `framer-motion@11` props |
| Global resets / fonts | Plain CSS `globals.css` |

Example: a `Modal` component using Tailwind for layout + `framer-motion@11` for the open/close animation, with one CSS Module file for an unusual scrollbar treatment.

---

## 7. Decision Tree

```
Are you on Next.js 14 App Router with a green-field codebase?
│   YES → Tailwind CSS 3.4 + shadcn/ui (the most-used combo in 2026)
│   NO  ↓
Are you on a Next.js Pages Router app with existing styled-components?
│   YES → Stay on styled-components 6 unless you have time to migrate
│   NO  ↓
Does the team strongly prefer "real CSS files"?
│   YES → CSS Modules
│   NO  ↓
Do you want type-safe CSS at build time, willing to accept smaller ecosystem?
│   YES → Vanilla Extract 1.15
│   NO  → Tailwind CSS 3.4
```

The 2026 default for any new React + Vite or React + Next.js project: **Tailwind CSS 3.4 + `clsx` + `tailwind-merge` + `class-variance-authority` + shadcn/ui** components.

---

## 8. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Tailwind classes not appearing in build output | Check `content` glob in `tailwind.config.js` covers your files |
| Dynamic class names like `bg-${color}` purged | Tailwind can't see the dynamic string. Use a lookup map: `{ red: 'bg-red-500', ... }[color]` |
| Two utilities conflict (`p-4 p-2`) and CSS-cascade-randomness wins | Use `tailwind-merge` |
| Long className strings unreadable | Extract into a CVA component variant |
| styled-components dynamic prop passed to DOM as attribute | Use `$`-prefixed transient props (v6 default) |
| CSS Modules typo `.titel` silently no-ops | Add `vite-plugin-typed-css-modules@1` or `typescript-plugin-css-modules` for IDE check |
| Tailwind dark mode with `class` strategy not toggling | Wire up `next-themes@0.3`, not raw `localStorage` |

---

## Summary

| Tool | Pick when | Avoid when |
|------|-----------|-----------|
| `tailwindcss@3.4` + `clsx@2.1` + `tailwind-merge@2.3` + `class-variance-authority@0.7` | Default for new React or Next.js projects | Team has hard CSS-Modules-only mandate |
| CSS Modules (built-in) | Team prefers real CSS files; design system internals | You'd rather not invent your own spacing/color scale |
| `styled-components@6.1` | Maintaining an existing codebase | New Next.js 14 App Router project |
| `@vanilla-extract/css@1.15` | Type-safe CSS, willing to accept smaller ecosystem | You want maximum hireability and tutorials |

| Rule | Why |
|------|-----|
| Use `tailwind-merge` for any class composition | Resolves conflicts deterministically |
| Wrap utilities in `cva` for components with multiple variants | Cleaner than chained ternaries |
| Don't migrate to Tailwind 4 until stable | v4 is alpha; v3 is the production version through 2026 |
| Use `next-themes@0.3` for dark mode | Don't reinvent the toggle/persist logic |

---

## Further reading

- [Tailwind CSS docs](https://tailwindcss.com/docs/installation)
- [shadcn/ui](https://ui.shadcn.com/) — the largest consumer of Tailwind + Radix UI
- [class-variance-authority](https://cva.style/docs)
- [tailwind-merge](https://github.com/dcastil/tailwind-merge)
- [Next.js — Styled Components setup](https://nextjs.org/docs/app/building-your-application/styling/css-in-js#styled-components)
- [Vanilla Extract docs](https://vanilla-extract.style/)
