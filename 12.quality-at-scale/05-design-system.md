# Quality at Scale — 05. Design Systems: Tokens, Component Library, Versioning, Consumer Apps

> **What / Why / How** — start with **tokens, not components**. Build the library inside a **monorepo** with `turbo@2` + `pnpm@9`. Version with **changesets**. Ship as **source code through `transpilePackages`**, not as a bundled npm package, while everyone is internal.

---

## 1. What a Design System Actually Is

Three layers, in order of importance:

| Layer | What it is | Examples |
|-------|------------|----------|
| **1. Design tokens** | Named values for color, spacing, typography, radius, shadow | `--color-primary: #2563eb`, `space-4: 1rem` |
| **2. Components** | Typed, documented, accessible React components built from tokens | `<Button>`, `<Input>`, `<Dialog>` |
| **3. Patterns / app-level pieces** | Composed components for repeating layouts | `<PageHeader>`, `<DataTable>`, `<EmptyState>` |

**Most teams skip Layer 1 and start with Layer 2 — that's the mistake.** Without shared tokens, every component re-invents the color palette and spacing scale. The system stays inconsistent forever.

The discipline: tokens first, primitives second, patterns last. Every layer references the layer below it.

### When you need a design system

- Multiple apps share the same brand (web app, marketing, mobile, admin).
- Multiple teams ship UI work concurrently.
- A dedicated design team exists, separate from engineering.
- You're building a **public design system** to ship to npm (Twilio Paste, Adobe Spectrum, Shopify Polaris).

### When a design system is overkill

- Single-app project, single team, fewer than ~20 components.
- shadcn/ui already covers your needs (use it directly, no DS layer needed).
- You'd be building it just to have one. Don't.

For most production SaaS: **shadcn/ui IS your design system** (covered in `08.ecosystem/02-ui-libraries.md`). You own the source, customize tokens, and skip the build/publish overhead. The rest of this file applies when shadcn alone isn't enough — multi-app monorepo, mobile + web parity, dedicated design team.

---

## 2. The Real Choices for Building a Design System in 2026

| Approach | Best for | Effort |
|----------|----------|--------|
| **shadcn/ui owned in your repo** (`components/ui/`) | Single app or simple monorepo; design control via Tailwind tokens | Low |
| **Internal `packages/ui` workspace + Turborepo** | Multi-app monorepo (web + admin + mobile) | Medium |
| **Published npm package** (e.g. `@acme/ui`) | Consumed by external repos / contractors / OSS release | High |
| **Built on `@radix-ui/react-*@1.1`** | Want headless primitives + your styling layer | Medium |
| **Built on `react-aria@3` + `react-aria-components@1`** | Adobe-grade accessibility, multi-framework future | High |
| **Built on `@ark-ui/react@4`** + Park UI | State-machine internals; multi-framework (React + Vue + Solid) | High |
| **Built on Tamagui 1** | Cross-platform web + mobile (React Native) shared code | High |

This file uses **monorepo + `packages/ui` + shadcn/ui patterns** as the default — it's what most production teams settle on by 2026.

---

## 3. Tokens — The Foundation Everyone Skips

### What tokens are

Tokens are **named values** that map to design decisions. The naming system is the design system.

```css
/* ❌ Anti-pattern — hardcoded values everywhere */
.button { background: #2563eb; padding: 0.5rem 1rem; border-radius: 0.375rem; }
.alert  { background: #2563eb; padding: 0.5rem; border-radius: 0.25rem; }

/* ✅ Tokens — change once, propagate everywhere */
:root {
  --color-brand-500: 37 99 235;          /* RGB triplet for alpha blending */
  --space-2: 0.5rem;
  --space-4: 1rem;
  --radius-md: 0.375rem;
  --radius-sm: 0.25rem;
}
.button { background: rgb(var(--color-brand-500)); padding: var(--space-2) var(--space-4); border-radius: var(--radius-md); }
.alert  { background: rgb(var(--color-brand-500)); padding: var(--space-2); border-radius: var(--radius-sm); }
```

When the brand changes, you edit `--color-brand-500` once.

### Two-tier token model — global + semantic

The pattern that scales:

| Tier | Purpose | Example |
|------|---------|---------|
| **Global tokens** (primitives) | Raw color/scale values | `--color-blue-500: 37 99 235` |
| **Semantic tokens** (aliases) | Named uses of primitives | `--color-primary: var(--color-blue-500)` |

Components reference **semantic tokens only**. Semantic tokens reference globals. When you change the brand from blue to green, you update **one** semantic token (`--color-primary: var(--color-green-500)`); every component flips automatically.

```css
/* tokens/global.css — primitives */
:root {
  --color-slate-50: 248 250 252;
  --color-slate-100: 241 245 249;
  --color-slate-900: 15 23 42;
  --color-blue-500: 37 99 235;
  --color-blue-600: 29 78 216;
  --color-red-500: 239 68 68;
  --color-emerald-500: 16 185 129;
  --space-1: 0.25rem;
  --space-2: 0.5rem;
  --space-3: 0.75rem;
  --space-4: 1rem;
  --space-6: 1.5rem;
  --space-8: 2rem;
  --radius-sm: 0.25rem;
  --radius-md: 0.375rem;
  --radius-lg: 0.5rem;
  --shadow-sm: 0 1px 2px 0 rgb(0 0 0 / 0.05);
  --shadow-md: 0 4px 6px -1px rgb(0 0 0 / 0.1);
}

/* tokens/semantic.css — aliases */
:root {
  --color-bg: var(--color-slate-50);
  --color-fg: var(--color-slate-900);
  --color-primary: var(--color-blue-500);
  --color-primary-hover: var(--color-blue-600);
  --color-destructive: var(--color-red-500);
  --color-success: var(--color-emerald-500);
  --color-border: var(--color-slate-100);
  --color-muted-fg: var(--color-slate-500);
}

.dark {
  --color-bg: var(--color-slate-900);
  --color-fg: var(--color-slate-50);
  --color-border: var(--color-slate-700);
}
```

Switching to dark mode flips **only semantic tokens**, not the underlying scale. The same `<Button>` component works in both themes.

### Token categories worth defining

| Category | Examples |
|----------|----------|
| Color (semantic) | `bg`, `fg`, `primary`, `destructive`, `success`, `warning`, `border`, `muted`, `accent` |
| Color (global scale) | `slate-50`–`slate-900`, `blue-50`–`blue-900` |
| Spacing | `space-0`, `1`, `2`, `3`, `4`, `6`, `8`, `12`, `16`, `24` (4px scale) |
| Typography size | `text-xs`, `sm`, `base`, `lg`, `xl`, `2xl`, `3xl` |
| Typography weight | `font-normal`, `medium`, `semibold`, `bold` |
| Line height | `leading-tight`, `normal`, `relaxed` |
| Radius | `rounded-none`, `sm`, `md`, `lg`, `xl`, `full` |
| Shadow | `shadow-sm`, `md`, `lg`, `xl`, `2xl` |
| Z-index | `z-modal: 100`, `z-tooltip: 200`, `z-toast: 300` |
| Transition | `duration-150`, `300`, `500`; `ease-in`, `out`, `in-out` |
| Breakpoint | `sm`, `md`, `lg`, `xl`, `2xl` |

The **shadcn/ui defaults** (covered in `08.ecosystem/02-ui-libraries.md`) ship a sensible token set out of the box — `--background`, `--foreground`, `--primary`, `--muted`, etc. Most teams extend this rather than starting from scratch.

### Tools for tokens

| Tool | Best for |
|------|----------|
| **CSS custom properties** | Default. Works everywhere. Pair with Tailwind `theme.extend.colors`. |
| **Tailwind 3.4 / 4** | Utility classes that read from the CSS variables — covered in `05.routing-and-styling/02-styling.md` |
| **Style Dictionary 4** | `style-dictionary@4` — Amazon-built; converts tokens (JSON) to CSS / Sass / iOS / Android |
| **`@tokens-studio/sd-transforms@1`** | Bridge from Figma Tokens Studio to Style Dictionary |
| **Open-Color** | `open-color@1` — pre-built color palette as a starting scale |
| **`radix-colors@3`** | Radix's accessibility-first color scales |

For internal-only systems: CSS variables + Tailwind config is enough. For multi-platform (web + iOS + Android): **Style Dictionary** to fan out one source of truth. For Figma-as-source-of-truth: Figma's "Variables" feature exports JSON consumable by Style Dictionary.

---

## 4. Repository Structure — `packages/ui` in a Monorepo

The standard layout (covered in `08.ecosystem/03-monorepo.md`):

```
my-monorepo/
├── apps/
│   ├── web/                            ← Next.js consumer
│   ├── admin/                          ← Next.js consumer
│   └── mobile/                         ← Expo SDK 51 consumer (covered in 09.beyond-web/01)
├── packages/
│   ├── ui/                             ← the design system
│   │   ├── src/
│   │   │   ├── tokens/
│   │   │   │   ├── global.css
│   │   │   │   └── semantic.css
│   │   │   ├── primitives/
│   │   │   │   ├── Button.tsx
│   │   │   │   ├── Input.tsx
│   │   │   │   ├── Dialog.tsx
│   │   │   │   └── ...
│   │   │   ├── patterns/
│   │   │   │   ├── PageHeader.tsx
│   │   │   │   └── EmptyState.tsx
│   │   │   ├── styles.css              ← imports tokens + reset
│   │   │   └── index.ts                ← public API
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── tailwind.config.ts          ← shared Tailwind preset
│   └── tsconfig/                       ← shared TS configs
└── package.json
```

### `packages/ui/package.json` — the public API

```jsonc
{
  "name": "@acme/ui",
  "version": "0.0.0",
  "private": true,                      // not published while internal-only
  "type": "module",
  "main": "./src/index.ts",             // source mode, no build step
  "types": "./src/index.ts",
  "exports": {
    ".": "./src/index.ts",
    "./button": "./src/primitives/Button.tsx",
    "./styles.css": "./src/styles.css",
    "./tailwind.config": "./tailwind.config.ts"
  },
  "scripts": {
    "lint": "eslint .",
    "type-check": "tsc --noEmit"
  },
  "dependencies": {
    "react": "^18.3.0 || ^19.0.0",
    "@radix-ui/react-slot": "^1.1.0",
    "class-variance-authority": "^0.7.0",
    "clsx": "^2.1.0",
    "tailwind-merge": "^2.3.0"
  },
  "peerDependencies": {
    "react": "^18.3.0 || ^19.0.0",
    "tailwindcss": "^3.4.0"
  }
}
```

### Source mode vs build mode

| Mode | Setup | Trade-off |
|------|-------|-----------|
| **Source** (`main: src/index.ts`) | Consumers transpile via `transpilePackages` | No build step in the package; no stale-cache bugs |
| **Build** (`main: dist/index.js`) | Use `tsup@8` or Vite library mode | Required for npm publishing; needed for non-TS consumers |

For internal-only Next.js monorepos: **always source mode**. Consumer's Next.js compiles your TS package alongside the app:

```js
// apps/web/next.config.js
module.exports = {
  transpilePackages: ['@acme/ui'],
};
```

You skip an entire build step + watch process + cache invalidation problem. Edits to `Button.tsx` show up instantly in `apps/web` dev server.

When building goes from optional to mandatory:
- You publish to npm (consumers can't be assumed to use `transpilePackages`).
- You consume the package from non-Next.js apps that don't transpile node_modules.
- You need to ship type-stripped output (rare — even Vite handles TS in deps).

---

## 5. Building the Component Layer

### Use `class-variance-authority` for variant components

The `cva` pattern from `05.routing-and-styling/02-styling.md` and `08.ecosystem/02-ui-libraries.md` is the design-system standard:

```tsx
// packages/ui/src/primitives/Button.tsx
import { Slot } from '@radix-ui/react-slot';
import { cva, type VariantProps } from 'class-variance-authority';
import { forwardRef } from 'react';
import { cn } from '../utils/cn';

const buttonVariants = cva(
  'inline-flex items-center justify-center rounded-md text-sm font-medium ring-offset-background transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring disabled:pointer-events-none disabled:opacity-50',
  {
    variants: {
      variant: {
        primary:     'bg-primary text-primary-foreground hover:bg-primary/90',
        secondary:   'bg-secondary text-secondary-foreground hover:bg-secondary/80',
        destructive: 'bg-destructive text-destructive-foreground hover:bg-destructive/90',
        ghost:       'hover:bg-accent hover:text-accent-foreground',
      },
      size: {
        sm: 'h-9 px-3',
        md: 'h-10 px-4',
        lg: 'h-11 px-6',
        icon: 'h-10 w-10',
      },
    },
    defaultVariants: { variant: 'primary', size: 'md' },
  }
);

export interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {
  asChild?: boolean;
}

export const Button = forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant, size, asChild = false, ...props }, ref) => {
    const Comp = asChild ? Slot : 'button';
    return <Comp ref={ref} className={cn(buttonVariants({ variant, size, className }))} {...props} />;
  }
);
Button.displayName = 'Button';
```

This is the exact shape every shadcn/ui primitive uses. Copy it for your custom variants. Document each variant's intent in the Storybook stories (covered in `12.quality-at-scale/04-storybook.md`).

### Public API — `index.ts`

```ts
// packages/ui/src/index.ts
export { Button, type ButtonProps } from './primitives/Button';
export { Input, type InputProps } from './primitives/Input';
export { Dialog, DialogContent, DialogHeader, DialogTitle, DialogTrigger } from './primitives/Dialog';
// ...

export { cn } from './utils/cn';

// Re-export commonly used types so consumers don't need to dig
export type { VariantProps } from 'class-variance-authority';
```

Be **deliberate** about the public API. Every export is a commitment. **Internal helpers should not be re-exported.**

### Sub-path exports for tree-shaking

```ts
// Consumers can import per-component:
import { Button } from '@acme/ui';                    // grabs everything
import { Button } from '@acme/ui/button';             // tree-shakeable
```

The second form keeps the bundle small if a consumer only uses `Button`. Defined via `exports` in `package.json` (shown in §4).

---

## 6. Cross-Platform — Sharing Web + Native

The hardest part of design systems in 2026: web + React Native parity.

### Option A — Tamagui 1

`tamagui@1.108` (covered in `09.beyond-web/01-react-native-expo.md`) ships components that run **identically** on web (via React Native Web) and React Native. The compiler statically extracts CSS for web and StyleSheet for native.

```tsx
// One file works on both
import { Button } from '@acme/ui';
<Button theme="primary">Save</Button>
```

This is the ideal for teams with strong web ↔ native parity goals. The setup cost is real (~1–2 weeks of plumbing) but pays off for products like Discord, Coinbase, Shopify Mobile that ship on both.

### Option B — Per-platform variants

```
packages/ui/src/primitives/
├── Button.tsx                 ← web (default)
├── Button.native.tsx          ← React Native override
└── Button.types.ts            ← shared types
```

Metro and Webpack respect `.native.tsx` extensions. Import `@acme/ui/button` and the right file resolves at build time. Use shadcn/ui patterns on web; React Native primitives + Nativewind 4 on native.

This is the path most teams pick because **the design constraints differ enough** that one component for both is a compromise. A web `<Dialog>` is centered; a native one slides up from the bottom. Forcing them to share code creates worse versions of both.

For a single-platform DS (web only, or RN only): skip the cross-platform question entirely.

---

## 7. Versioning, Changelogs, and Release Discipline

### `changesets@2` is the standard

`@changesets/cli@2` (covered in `08.ecosystem/03-monorepo.md`) is the design-system release tool. Every PR that touches `packages/ui` should include a changeset:

```bash
pnpm changeset
# Choose: @acme/ui — patch / minor / major
# Write: "Added `destructive` variant to Button"
```

A `.md` file in `.changeset/` records the bump. When you merge to main, the [Changesets GitHub Action](https://github.com/changesets/action) opens a "Version Packages" PR. Merge that → versions bump, CHANGELOG updates, npm publishes (if you publish).

### Semver discipline for design systems

| Change type | Bump | Examples |
|-------------|------|----------|
| **Major** | x.0.0 | Removing a prop; renaming a component; breaking visual changes that consumers must adapt to |
| **Minor** | 1.x.0 | Adding a variant, prop, or component; non-breaking visual refinements |
| **Patch** | 1.0.x | Bug fix; tightening types; performance |

**Major bumps are expensive to consume.** Aim for one major per year on a stable design system. Use deprecation warnings for 1–2 minor cycles before removing:

```tsx
export interface ButtonProps {
  /** @deprecated Use `variant="destructive"` instead. Removing in v3.0. */
  isDanger?: boolean;
}
```

The deprecation comment shows up in TypeScript hover and IDE warnings. Consumers see them as they upgrade.

### CHANGELOG visibility

Changesets generates a per-package CHANGELOG.md. Surface it:
- Link from your Storybook docs (`/docs/changelog`).
- Link from your component reference page.
- Email summary to the design-system Slack channel on each release.

Consumers shouldn't have to dig into git history to see what changed.

---

## 8. Documentation — Storybook IS Your Docs

Per `12.quality-at-scale/04-storybook.md`: **Storybook is the design system's documentation site**. Don't build a separate docs site (Docusaurus, Nextra, custom) on top.

The pattern:

```
packages/ui/src/primitives/Button.tsx           ← code
packages/ui/src/primitives/Button.stories.tsx   ← examples + automated tests
packages/ui/src/primitives/Button.mdx           ← prose: when to use, when not to use, accessibility notes
```

Mix MDX and stories using `<Meta of={...} />` and `<Story of={...} />` blocks. The MDX files become the docs pages; stories become the live examples.

Real production rule: **if a component has prose docs but no stories, it's not in the design system.** Stories are the contract. Docs without working examples rot fast.

### Deploy Storybook per branch

For a design system, every PR should deploy a Storybook preview. Two patterns:

| Tool | Setup |
|------|-------|
| **Vercel preview** | Separate Vercel project pointing at `packages/ui` build |
| **Chromatic published Storybook** | Built-in with every Chromatic build; preview URL per commit |

Designers and PMs review against the deployed Storybook, not the merged main branch. The PR description always includes:

```markdown
- 📖 Storybook preview: https://acme-ui-pr-42.vercel.app
- 🎨 Chromatic review: https://www.chromatic.com/build?appId=...&number=42
```

---

## 9. Consumer Adoption — Make It Easy to Use

### Provide a Tailwind preset

Consumers shouldn't reinvent the token list:

```ts
// packages/ui/tailwind.config.ts
import type { Config } from 'tailwindcss';

export default {
  theme: {
    extend: {
      colors: {
        primary: 'rgb(var(--color-primary) / <alpha-value>)',
        secondary: 'rgb(var(--color-secondary) / <alpha-value>)',
        destructive: 'rgb(var(--color-destructive) / <alpha-value>)',
        background: 'rgb(var(--color-bg) / <alpha-value>)',
        foreground: 'rgb(var(--color-fg) / <alpha-value>)',
      },
      borderRadius: {
        lg: 'var(--radius-lg)',
        md: 'var(--radius-md)',
        sm: 'var(--radius-sm)',
      },
    },
  },
  plugins: [require('tailwindcss-animate')],
} satisfies Config;
```

```ts
// apps/web/tailwind.config.ts
import preset from '@acme/ui/tailwind.config';

export default {
  presets: [preset],
  content: [
    './app/**/*.{ts,tsx}',
    '../../packages/ui/src/**/*.{ts,tsx}',           // ← critical: scan UI package source
  ],
};
```

Without the `content` glob covering the UI package, Tailwind doesn't know about classes used inside the package and purges them.

### Provide a single CSS import

```css
/* apps/web/app/globals.css */
@import '@acme/ui/styles.css';
@tailwind base;
@tailwind components;
@tailwind utilities;
```

`@acme/ui/styles.css` defines all tokens, fonts, and the reset. Consumer apps add Tailwind layers on top.

### Provide a copy-paste starter

The `apps/_template` pattern: a minimal app pre-wired with `@acme/ui`, used as the starting point for new apps. Copy the folder, rename, customize. Saves hours per new app.

### Track consumer feedback

A real production team runs:
- **Slack channel** (`#design-system`) — engineers post issues + requests.
- **Quarterly survey** — what's missing, what's painful, what's clicking.
- **Office hours** — DS team available 1 hour/week for migration help.

**Component requests pile up. Triage ruthlessly.** Most requested ≠ most valuable. The most-used components (Button, Input, Dialog) deserve the most polish.

---

## 10. Governance — The Boring Discipline That Keeps a System Alive

### RFC process for new components

Before adding a new component to the system, an engineer writes a short doc:

```markdown
# RFC: Color picker

## Problem
Three different teams have implemented their own color picker (in app A, B, C).
None match the design system. Maintenance is fragmented.

## Proposal
Add `<ColorPicker>` to `@acme/ui`, built on `@uiw/react-color@2`.
API: `<ColorPicker value={...} onChange={...} format="hex" />`.

## Variants
- Hex / RGB / HSL formats
- With/without alpha channel
- With/without preset swatches

## Accessibility
- Keyboard navigable via Radix Slider primitives
- Color contrast checked at default render
- Screen-reader announces hex value on change

## Migration
Apps A, B, C will replace their custom pickers in PRs over 4 weeks.
```

PMs / designers / engineers review. Once approved, the engineer ships it and migrates the consumers. Without this gate, components proliferate without consistency.

### Component-status badges

Every component has a status:

| Status | Meaning | When to use |
|--------|---------|-------------|
| **Stable** | Production-ready, semver-stable API | Default for consumers |
| **Beta** | API may change before next major; gather feedback | New components in evaluation |
| **Deprecated** | Replacement exists; will be removed in next major | Migrate ASAP |
| **Internal** | Used by other components, not consumer-facing | Don't import directly |

Surface in Storybook:

```tsx
// Button.stories.tsx
const meta = {
  title: 'UI/Button',
  parameters: {
    componentStatus: 'stable',
  },
};
```

A custom Storybook addon renders the status badge in the sidebar. Consumers see at a glance which components are safe to depend on.

### Code ownership

```
# .github/CODEOWNERS
/packages/ui/  @acme/design-system-team
```

Every PR touching the UI package requires DS team approval. Slows things down by design — DS changes have N consumers and shouldn't be ad-hoc.

---

## 11. Anti-Patterns

### Anti-pattern 1 — building the system before consumers exist

> "Let's build a design system for our future products."

You don't know what consumers need until they exist. Build the second consumer, then extract the system. **Two real apps generate the right abstractions; one hypothetical doesn't.**

### Anti-pattern 2 — over-customizable components

> "Our `<Button>` accepts 27 props for every styling variation."

A button with 27 props is harder to use than 27 buttons. Prefer **compound components + variants**, not god-components.

```tsx
// ❌ The kitchen sink
<Button leftIcon={...} rightIcon={...} loading={...} colorScheme="..." size="..." rounded="..." shadowed={...} />

// ✅ Compose
<Button variant="primary" size="md">
  <Save /> Save
</Button>
```

### Anti-pattern 3 — system mirrors Figma 1:1

> "Figma has 47 buttons; the code system needs 47."

Figma has variants for every state combination. Code has variants for every distinct **decision**. Most Figma "variants" map to a single code component with props.

The DS team's job: collapse the Figma spread to a small, intentional API. **Code shouldn't have 47 buttons.**

### Anti-pattern 4 — design tokens that match the design

> "Our designer named the color `--color-cta-blue-bg-hover-2`."

That's not a token; it's a label for one element. Tokens are **reusable values**. If only one place uses it, it doesn't need a name. Promote to semantic only when 3+ places reference it.

### Anti-pattern 5 — multi-major-version-per-quarter releases

> "v2 → v3 → v4 in three months."

Major versions are expensive for consumers. The DS team forgets this; each consumer must coordinate, test, and migrate. **Aim for one major per year**, with deprecation warnings for ≥ 2 minor cycles.

### Anti-pattern 6 — Storybook as the only test surface

> "Visual regression in Storybook is enough."

It's not. Pair Storybook + Chromatic with `jest-axe@9` (covered in `11.capabilities/05-accessibility.md`), Playwright E2E (covered in `12.quality-at-scale/03-e2e-playwright.md`), and per-component Vitest unit tests. Different layers catch different bugs.

---

## 12. Real Roadmap — From Zero to Mature

| Stage | Effort | What you have |
|-------|--------|---------------|
| **0. shadcn/ui in your repo** | 1 week | A working Button, Dialog, Form. No DS layer. |
| **1. Internal `packages/ui`** | 2–4 weeks | Workspace package, source-mode imports, shared tokens via Tailwind config |
| **2. Storybook + tokens documented** | 1 week | Stories per component, MDX docs, design tokens in CSS variables |
| **3. Chromatic visual regression** | 1 week | Per-PR review by designers; baseline per branch |
| **4. Consumer feedback loop** | Ongoing | Slack channel, RFC process, component-status badges |
| **5. Multi-platform (web + native)** | 4–8 weeks | Tamagui or per-platform variants; iOS + Android +Web parity |
| **6. Public npm package** | 4 weeks | Build pipeline, semver, npm publish via changesets |
| **7. Multi-brand / theming** | Ongoing | Multiple semantic-token sets for sub-brands |

Most production teams in 2026 sit at stage 3–4 and don't need stages 5–7 unless their product roadmap demands it.

---

## 13. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Tailwind classes in `packages/ui` are purged | Add `../../packages/ui/src/**/*.{ts,tsx}` to consumer's Tailwind `content` glob |
| Two React versions in node_modules | Hoist `react` via `peerDependencies`; use pnpm's strict resolution |
| Consumers can't tree-shake; bundle huge | Define sub-path `exports` in package.json |
| Token changes require CSS-class rewrites | Use semantic tokens; rename them when concepts change, not values |
| New component added without RFC | Code review catches it; rotate DS team review for the workspace |
| Designer can't preview changes | Deploy Storybook per PR via Vercel or Chromatic |
| Breaking change ships without major bump | Changesets enforces semver; CI fails if PR doesn't include a changeset |
| Component used in 50 places, breaking change is impossible | Deprecate for 2+ minor cycles before removing |
| `next/font` doesn't load in Storybook | Use `@storybook/nextjs@8.3` framework; load fonts in `preview.tsx` |
| TS types break across major Next.js / React versions | Pin React + Next.js peerDependencies generously (`^18 \|\| ^19`) |
| Storybook stories drift from real usage | Pair with Playwright E2E; if it works in Storybook but breaks in app, the story has the wrong context |
| Design tokens out of sync with Figma | Style Dictionary 4 + Tokens Studio export; auto-PR on Figma change |

---

## 14. Decision Tree

```
Do you actually need a design system?
│
├── Single app, single team, < 20 components → use shadcn/ui directly. No DS layer needed.
├── Two apps sharing brand → extract `packages/ui` workspace
└── Public OSS / multi-tenant → published npm package + npm DX (build, types, codegen)

What scope?
│
├── Web-only → shadcn/ui + Radix UI 1.1 (covered in 08.ecosystem/02)
├── Web + native → Tamagui 1 (covered in 09.beyond-web/01)
└── Multi-framework (React + Vue + Solid) → Ark UI 4 + Park UI

How to ship?
│
├── Internal monorepo → source mode + transpilePackages
└── External consumers → tsup@8 build, npm publish, semver via changesets@2

How to document?
│
├── Storybook 8.3 + MDX (default)
└── Storybook + Chromatic for design review (production grade)

How to govern?
│
├── 1–3 maintainers → light touch; PR review only
└── 5+ maintainers + 50+ consumers → RFC process, status badges, CODEOWNERS, quarterly survey
```

---

## 15. What This Topic Connects To

- **`05.routing-and-styling/02-styling.md`** — Tailwind 3.4 + cva is the styling primitive used everywhere here.
- **`08.ecosystem/02-ui-libraries.md`** — shadcn/ui is the starting point.
- **`08.ecosystem/03-monorepo.md`** — Turborepo + pnpm is the monorepo foundation.
- **`09.beyond-web/01-react-native-expo.md`** — Tamagui for web ↔ native.
- **`11.capabilities/05-accessibility.md`** — accessibility is a design-system requirement, not a feature.
- **`12.quality-at-scale/04-storybook.md`** — Storybook is the documentation surface.
- **`12.quality-at-scale/03-e2e-playwright.md`** — consumers' Playwright tests verify DS components in real apps.

---

## 16. Summary

| Layer | Tool / Pattern |
|-------|----------------|
| **Tokens** | CSS variables + Tailwind theme.extend; Style Dictionary 4 for multi-platform |
| **Components** | shadcn/ui patterns + cva + Radix UI primitives |
| **Cross-platform** | Tamagui 1 (shared) or per-platform variants (`.native.tsx`) |
| **Repository** | Turborepo + pnpm monorepo; `packages/ui` workspace |
| **Build mode** | Source mode via `transpilePackages` (internal); tsup@8 (published) |
| **Docs** | Storybook 8.3 + MDX |
| **Visual regression** | Chromatic (production grade) or Playwright `toHaveScreenshot` (free) |
| **Versioning** | `@changesets/cli@2` |
| **Tokens source** | Figma Tokens Studio → Style Dictionary 4 |
| **Governance** | RFC docs, component-status badges, CODEOWNERS |

| Rule | Why |
|------|-----|
| Build the system after the second consumer exists | Two real apps generate the right abstractions |
| Two-tier tokens (global → semantic) | Brand changes touch one semantic token, not 100 components |
| Source mode (`transpilePackages`) for internal monorepos | No build step; instant updates in dev |
| One major version per year | Major bumps are expensive for consumers |
| Deprecation warnings for ≥ 2 minor cycles before removal | Gives consumers time to migrate |
| Storybook is the docs site, don't build a separate one | Stories are the contract; docs without examples rot |
| Chromatic for designer review | Closes the design-engineering loop |
| Component RFCs before merging | Avoids 50 different "Button" variants |
| Tailwind preset shipped with the package | Consumers don't reinvent the token list |
| Auto-publish via changesets GitHub Action | Eliminates manual release steps |

---

## Further reading

- [shadcn/ui](https://ui.shadcn.com/) — the most-cloned starter for design-system primitives
- [Style Dictionary docs](https://amzn.github.io/style-dictionary/) — multi-platform tokens
- [Figma Tokens Studio](https://tokens.studio/) — Figma → JSON tokens
- [Open-Color](https://yeun.github.io/open-color/) — pre-built color palette
- [Radix Colors](https://www.radix-ui.com/colors) — accessibility-first scales
- [Twilio Paste](https://paste.twilio.design/) — production design system reference
- [Adobe Spectrum](https://spectrum.adobe.com/) — production design system reference
- [Shopify Polaris](https://polaris.shopify.com/) — production design system reference
- [Brad Frost — Atomic Design](https://atomicdesign.bradfrost.com/) — the canonical taxonomy
- [Changesets docs](https://github.com/changesets/changesets)
