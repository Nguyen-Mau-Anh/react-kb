# Ecosystem — 02. UI Libraries: shadcn/ui vs MUI 6 vs Headless UI 2 vs Mantine 7

> **What / Why / How** — pick by team philosophy: own-the-code (shadcn/ui), opinionated theme (MUI/Mantine), or roll-your-own (Headless UI / Ark UI).

---

## 1. The Real Choices in 2026

| Library | npm | Approach | Bundle impact | Used by |
|---------|-----|----------|---------------|---------|
| **shadcn/ui** | not an npm package — copy-paste from CLI | Radix UI primitives + Tailwind, source code in *your* repo | Per-component (typically ~2–6 KB each) | Vercel, Cal.com, T3 stack apps, most new SaaS in 2026 |
| **Radix UI Primitives 1.x** | `@radix-ui/react-*@1.1` | Headless, accessible primitives (no styles) | Per-primitive (~3–8 KB) | The base under shadcn/ui; many design systems |
| **Headless UI 2** | `@headlessui/react@2.0` | Tailwind Labs' headless components | ~10 KB | Tailwind UI, internal Tailwind apps |
| **Ark UI 4** | `@ark-ui/react@4` | Headless, framework-agnostic, state-machine driven | ~15–25 KB | Park UI, Chakra v3 internals |
| **MUI v6 (Material UI)** | `@mui/material@6` | Pre-styled Material Design 3 components | ~50–90 KB minimum | Enterprise apps, internal tools, Toolpad |
| **Mantine 7** | `@mantine/core@7` | Pre-styled, opinionated, batteries-included | ~60–100 KB | Indie SaaS, dashboards |
| **Chakra UI v3** | `@chakra-ui/react@3` | Pre-styled, themeable, built on Ark UI | ~70 KB | Mid-size apps switching from MUI |
| **Ant Design 5** | `antd@5` | Pre-styled, dense, enterprise-flavored | ~120 KB | Heavy data apps, China-market apps |
| **Park UI** | not an npm package — copy-paste | Ark UI + Panda CSS / Tailwind, "shadcn/ui for Ark" | Per-component | Teams that want shadcn-style ownership without Radix |

The two real defaults today:
- **shadcn/ui + Radix UI 1.x + Tailwind 3.4** — for new projects where you want full design control.
- **MUI v6** — for internal tools, dashboards, or any app where Material Design is the brief.

---

## 2. The Core Philosophical Split

### "Own the code" — shadcn/ui, Park UI

You run a CLI that **copies source files** of components into your repo. Then you own them: edit the JSX, swap classnames, change behavior. There is no `node_modules` package shipping the component — the file lives in `src/components/ui/button.tsx`, and you can do whatever you want.

Pros:
- Zero version-bump pain. Updating a library never breaks your design.
- No vendor lock-in. The components are yours; if shadcn/ui dies, your code keeps working.
- Customization is a code edit, not a theming gymnastics exercise.

Cons:
- You're responsible for keeping components up-to-date (Radix bumps, accessibility fixes).
- More files in your repo from day one.

### "Configure a theme" — MUI, Mantine, Chakra, Ant Design

Pre-styled components live in `node_modules`. You configure colors, spacing, typography via a `theme` object. To customize a component, you pass `sx={{}}`, override CSS, or use a theme `components` override.

Pros:
- Up-and-running in one `npm install`.
- Consistent design without picking colors yourself.
- Polished components with edge cases handled.

Cons:
- Theme escape hatches accumulate over time.
- Bundle size is per-package, not per-component.
- Updating major versions can break theme overrides.

### "Headless" — Radix Primitives, Headless UI, Ark UI

Components have **logic and accessibility but no styling**. You bring CSS / Tailwind. Behavior is fully built (focus traps, keyboard nav, ARIA roles); appearance is yours.

Pros:
- Maximum design freedom with zero accessibility work.
- Smaller bundle than full UI kits.
- Composes well with Tailwind.

Cons:
- You write more wrapper components than with shadcn/ui (which already wraps them for you).

---

## 3. shadcn/ui — The 2026 Default

### What

Not a library — a CLI that scaffolds React + Radix UI + Tailwind components into your project. Started as one developer's GitHub repo (`shadcn/ui`); is now the most-cloned UI starter in the React ecosystem.

### Setup (Next.js 14 + Tailwind already installed)

```bash
npx shadcn@2.1 init
```

Prompts you for:
- Style: `Default` or `New York` (different default looks)
- Base color: slate / gray / zinc / neutral / stone
- CSS variables for theming: yes (recommended)

The CLI creates `components.json`, `lib/utils.ts` (the `cn()` helper from `05.routing-and-styling/02-styling.md`), and a `components/ui/` folder.

### Add components one at a time

```bash
npx shadcn@2.1 add button input dialog dropdown-menu form
```

Each command writes a real `.tsx` file you now own:

```tsx
// components/ui/button.tsx — abridged real shadcn/ui source
import * as React from 'react';
import { Slot } from '@radix-ui/react-slot';
import { cva, type VariantProps } from 'class-variance-authority';
import { cn } from '@/lib/utils';

const buttonVariants = cva(
  'inline-flex items-center justify-center rounded-md text-sm font-medium ring-offset-background transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring disabled:pointer-events-none disabled:opacity-50',
  {
    variants: {
      variant: {
        default: 'bg-primary text-primary-foreground hover:bg-primary/90',
        destructive: 'bg-destructive text-destructive-foreground hover:bg-destructive/90',
        outline: 'border border-input bg-background hover:bg-accent hover:text-accent-foreground',
        secondary: 'bg-secondary text-secondary-foreground hover:bg-secondary/80',
        ghost: 'hover:bg-accent hover:text-accent-foreground',
        link: 'text-primary underline-offset-4 hover:underline',
      },
      size: { default: 'h-10 px-4 py-2', sm: 'h-9 px-3', lg: 'h-11 px-8', icon: 'h-10 w-10' },
    },
    defaultVariants: { variant: 'default', size: 'default' },
  }
);

export interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {
  asChild?: boolean;
}

export const Button = React.forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant, size, asChild = false, ...props }, ref) => {
    const Comp = asChild ? Slot : 'button';
    return (
      <Comp className={cn(buttonVariants({ variant, size, className }))} ref={ref} {...props} />
    );
  }
);
Button.displayName = 'Button';
```

This is **your code**. Want a different rounding? Edit `rounded-md` to `rounded-full`. Want a new variant? Add it to the `cva` config. No `theme.button.borderRadius` indirection.

### What ships with shadcn/ui (40+ components)

Buttons, inputs, dialogs, popovers, tooltips, toasts (`sonner@1.5`), drawers, tabs, accordion, dropdown menus, command palettes (`cmdk@1`), data tables (`@tanstack/react-table@8`), date pickers (`react-day-picker@8`), calendars, charts (`recharts@2.12` wrapped), forms (`react-hook-form` + `zod` integrated), navigation menus, sheets, sidebars (the `sidebar-07` block is widely cloned).

### Themes — copy-paste at [ui.shadcn.com/themes](https://ui.shadcn.com/themes)

Eight color themes, light/dark variants, CSS variables. Click a theme → copy the CSS variables block → paste into `globals.css`. Done.

### Why shadcn/ui won 2024–2026

- Design control without writing a design system from scratch.
- AI-assisted code (Claude, Cursor, GitHub Copilot) generates correct shadcn/ui code reliably because the component patterns are public, repeatable, and live in the user's repo.
- Maintained by Vercel; integrated with `v0.dev` (Vercel's AI UI generator outputs shadcn/ui code).
- Active community: 70K+ GitHub stars, 1000+ third-party blocks at [shadcn.studio](https://shadcn.studio).

---

## 4. Radix UI Primitives 1.x — The Foundation

### What

`@radix-ui/react-*@1.1` is a set of **headless, accessible** UI primitives. Each component (`Dialog`, `Popover`, `Tooltip`, `DropdownMenu`, `Tabs`, ...) ships:
- Full ARIA attributes
- Keyboard navigation (arrow keys, Esc, Tab cycling)
- Focus management (trap inside dialog, restore on close)
- Composition via the compound-component pattern (covered in `03.rendering-patterns/03-component-patterns.md`)

Without styling. You bring Tailwind, CSS Modules, styled-components, whatever.

### Direct Radix usage (without shadcn/ui wrapping)

```tsx
import * as Dialog from '@radix-ui/react-dialog';

function ConfirmDelete({ onConfirm }: { onConfirm: () => void }) {
  return (
    <Dialog.Root>
      <Dialog.Trigger className="rounded bg-red-600 px-3 py-1.5 text-white">
        Delete
      </Dialog.Trigger>
      <Dialog.Portal>
        <Dialog.Overlay className="fixed inset-0 bg-black/50 data-[state=open]:animate-in data-[state=open]:fade-in-0" />
        <Dialog.Content className="fixed left-1/2 top-1/2 -translate-x-1/2 -translate-y-1/2 rounded-lg bg-white p-6 shadow-lg">
          <Dialog.Title className="text-lg font-semibold">Are you sure?</Dialog.Title>
          <Dialog.Description className="text-sm text-gray-600">
            This action can&apos;t be undone.
          </Dialog.Description>
          <div className="mt-4 flex justify-end gap-2">
            <Dialog.Close className="rounded border px-3 py-1.5">Cancel</Dialog.Close>
            <button className="rounded bg-red-600 px-3 py-1.5 text-white" onClick={onConfirm}>
              Delete
            </button>
          </div>
        </Dialog.Content>
      </Dialog.Portal>
    </Dialog.Root>
  );
}
```

Using shadcn/ui? You get this exact `Dialog` wrapped with sensible defaults via `npx shadcn@2.1 add dialog`.

### When to use Radix directly without shadcn/ui

- Writing a public design system that consumers install via npm.
- Building a custom design language where shadcn/ui's defaults don't match.
- You only need 1–2 primitives and don't want shadcn's CLI footprint.

---

## 5. Headless UI 2 — Tailwind Labs' Alternative to Radix

### What

`@headlessui/react@2.0` is Tailwind Labs' headless component library. Powers Tailwind UI's official component examples.

```tsx
import { Menu, MenuButton, MenuItem, MenuItems } from '@headlessui/react';

<Menu>
  <MenuButton className="rounded bg-blue-600 px-3 py-1.5 text-white">Options</MenuButton>
  <MenuItems anchor="bottom end" className="rounded border bg-white shadow">
    <MenuItem>
      {({ focus }) => (
        <a href="/profile" className={focus ? 'bg-blue-100' : ''}>
          Profile
        </a>
      )}
    </MenuItem>
  </MenuItems>
</Menu>
```

### Headless UI vs Radix UI Primitives — the honest comparison

| | Radix UI 1.x | Headless UI 2 |
|--|--------------|---------------|
| Number of primitives | ~30 | ~15 |
| API style | Compound components per primitive | Compound, slightly different naming |
| Accessibility coverage | Excellent (used as a reference by other libs) | Excellent |
| Active development | Constant releases | Active, Tailwind-team-paced |
| Most-used wrapper | shadcn/ui | Tailwind UI examples |
| Niche missing pieces | Combobox, file upload (use `cmdk` / `react-dropzone`) | Tabs (the v2 one is decent), Tooltip (added in v2) |

For new projects: **Radix UI 1.x via shadcn/ui** is the higher-momentum pick. Headless UI 2 is a fine second choice if you're committed to Tailwind UI's ecosystem.

---

## 6. Ark UI 4 — The Headless State-Machine Library

### What

`@ark-ui/react@4` is the headless component library by **Chakra UI**'s creator (Segun Adebayo). Built on Zag.js state machines under the hood. **Park UI** is the shadcn-style copy-paste layer on top of Ark UI.

```tsx
import { Dialog } from '@ark-ui/react/dialog';

<Dialog.Root>
  <Dialog.Trigger>Open</Dialog.Trigger>
  <Dialog.Backdrop />
  <Dialog.Positioner>
    <Dialog.Content>
      <Dialog.Title>Title</Dialog.Title>
      <Dialog.Description>Description</Dialog.Description>
      <Dialog.CloseTrigger>Close</Dialog.CloseTrigger>
    </Dialog.Content>
  </Dialog.Positioner>
</Dialog.Root>
```

### When Ark UI beats Radix

- You're building a multi-framework design system (Ark UI ships React, Vue, Solid, and Svelte from the same state machines).
- You want **Chakra UI v3** under the hood (Chakra v3 is built on Ark).
- State-machine driven internals matter to you (predictable interactions, exhaustively-tested transitions).

### When Ark UI loses

- shadcn/ui has overwhelmingly more community examples and AI-generation support.
- Park UI (Ark's shadcn-equivalent) is real but has a smaller component catalog.

---

## 7. MUI v6 (Material UI) — The Enterprise Default

### What

`@mui/material@6` ships pre-styled Material Design 3 components. Battle-tested in enterprise apps. Created by the Material UI team (now at MUI Inc, which also makes MUI Joy and MUI X).

### Setup

```bash
npm i @mui/material@6 @emotion/react@11 @emotion/styled@11
```

```tsx
// app/providers.tsx
'use client';
import { ThemeProvider, createTheme, CssBaseline } from '@mui/material';
import { AppRouterCacheProvider } from '@mui/material-nextjs/v14-appRouter';

const theme = createTheme({
  palette: {
    primary: { main: '#1976d2' },
    secondary: { main: '#9c27b0' },
  },
});

export function Providers({ children }: { children: React.ReactNode }) {
  return (
    <AppRouterCacheProvider>
      <ThemeProvider theme={theme}>
        <CssBaseline />
        {children}
      </ThemeProvider>
    </AppRouterCacheProvider>
  );
}
```

`AppRouterCacheProvider` is the Next.js App Router compatibility layer for emotion-based MUI. Required, otherwise CSS-in-JS doesn't render correctly during SSR.

### Real example — a form

```tsx
'use client';
import { Box, TextField, Button, Stack } from '@mui/material';

export function LoginForm() {
  return (
    <Box component="form" sx={{ display: 'flex', flexDirection: 'column', gap: 2, maxWidth: 360 }}>
      <TextField label="Email" type="email" required fullWidth />
      <TextField label="Password" type="password" required fullWidth />
      <Stack direction="row" justifyContent="flex-end" spacing={1}>
        <Button variant="text">Forgot?</Button>
        <Button variant="contained" type="submit">Sign in</Button>
      </Stack>
    </Box>
  );
}
```

### Why pick MUI

- **MUI X** ($14/dev/mo for Pro) — the single best React data grid, premium charts, advanced date pickers. No competing library matches the data-grid feature set.
- **Toolpad Studio** — low-code admin builder by the same team.
- **Material Design defaults** — useful when "make it look professional" is the brief and design isn't your strong point.
- **Enterprise track record** — used by NASA, Amazon, Salesforce internal tools.

### Why MUI loses against shadcn/ui for new SaaS projects

- Material Design 3 looks "Google-y." Modern SaaS aesthetics (Linear, Notion, Vercel) are not Material Design.
- Theming a component to look not-Material requires fighting `sx` props, `styleOverrides`, and emotion specificity. With shadcn/ui you'd just edit the file.
- Per-component bundle size is high; tree-shaking helps but minimum overhead is ~50 KB.
- Less AI-generation friendly: AI tools produce more reliable shadcn/ui code than MUI code.

### MUI Joy and Base UI

Same company also ships:
- `@mui/joy@5` — non-Material, more modern aesthetic, smaller bundle. Mostly stagnant in 2026.
- `@mui/base@5` — headless primitives by MUI. Less momentum than Radix UI.

If considering MUI without Material Design, just use shadcn/ui.

---

## 8. Mantine 7 — Polished, Hooks-Heavy, Indie Favorite

### What

`@mantine/core@7` is a pre-styled component library plus a giant hooks library (`@mantine/hooks@7`). Modern flat-style aesthetic, dark mode by default, popular for indie SaaS dashboards.

### Setup

```bash
npm i @mantine/core@7 @mantine/hooks@7 @mantine/form@7
```

```tsx
import { MantineProvider, Button, TextInput } from '@mantine/core';

<MantineProvider>
  <TextInput label="Email" required />
  <Button>Submit</Button>
</MantineProvider>
```

### Why pick Mantine

- 130+ pre-styled components — more than MUI in some categories.
- 60+ hooks (clipboard, debounced value, hotkeys, color scheme, etc.).
- `@mantine/form@7` is a real `react-hook-form` alternative when you don't need that library's specifics.
- `@mantine/dates@7`, `@mantine/notifications@7`, `@mantine/spotlight@7` — first-party utility packages.

### Why Mantine loses

- Same problem as MUI: customization fights the theme system.
- Smaller community than shadcn/ui, fewer AI-generation patterns, fewer examples.
- Bundle adds up — even with tree-shaking, expect 60–100 KB on a typical app.

---

## 9. Chakra UI v3 — Re-Built on Ark UI

### What

`@chakra-ui/react@3` is the v3 rewrite. Ditched emotion-based runtime CSS-in-JS for static CSS via `panda-css`. Built on **Ark UI 4** internals.

### Why pick Chakra v3

- Familiar API for teams already on v2.
- Built-in dark mode, accessibility-first, decent default aesthetics.
- Lighter than v2 because of the Panda CSS rewrite.

### Why Chakra v3 has lost momentum

- Most v2 users either stayed on v2 (low migration appetite) or moved to shadcn/ui (clear winner).
- Chakra's "everything is a prop" API (`<Box bg="gray.100" p={4}>`) is less performant than Tailwind's atomic classes.

For new projects: **skip Chakra unless your team has strong Chakra v2 experience and wants the upgrade path**.

---

## 10. Ant Design 5 — When Density Wins

### What

`antd@5` ships ~100 dense, enterprise-flavored components. Strong tables, dense data grids, complex form layouts.

### When Ant Design beats everything

- **Heavy data apps** — back-office dashboards with 50-column tables, multi-step wizards, complex filter trees.
- **China-market apps** — Ant Design originated at Alibaba and has the best Chinese-language ecosystem.

### When it loses

- Aesthetics: looks distinctly "Alibaba enterprise." Customization is heavy work.
- Bundle: 120 KB+ without aggressive tree-shaking.
- Less dynamic in 2026 than shadcn/ui or MUI.

---

## 11. Quick Reference — When to Pick Which

```
Need a polished pre-styled kit and you don't want to design?
├── Material Design aesthetic + paid data grid? → @mui/material@6 + MUI X
├── Indie SaaS aesthetic + lots of hooks?       → @mantine/core@7
├── Enterprise data density (China especially)? → antd@5
└── Otherwise (most cases)                      → shadcn/ui

Want full design control with accessibility built in?
├── Tailwind + Radix UI primitives, owned source files? → shadcn/ui
├── Tailwind UI ecosystem, Tailwind Labs primitives?    → @headlessui/react@2 + Tailwind UI
├── State-machine internals, multi-framework dream?     → @ark-ui/react@4 (or Park UI for shadcn-style)
└── DIY everything                                       → @radix-ui/react-* directly

Only need 1–2 primitives (e.g. just a Dialog and a Tooltip)?
└── Install just those Radix UI packages — no need for the whole shadcn/ui setup.

Building a public design system you'll publish to npm?
└── Radix UI primitives directly + your styling layer (Tailwind, Vanilla Extract).
   shadcn/ui isn't designed to be re-distributed; it's per-app code.
```

---

## 12. Tables, Charts, and Other Specialized Components

| Need | The clear winner |
|------|------------------|
| Data table with sorting / filtering / column resizing | `@tanstack/react-table@8` (headless) — wrapped by shadcn/ui's `data-table` |
| Date picker | `react-day-picker@8` (headless) — wrapped by shadcn/ui's `calendar` |
| Charts | `recharts@2.12` (chart-of-the-week for shadcn/ui), `apache-echarts@5` for serious viz, `visx@3` for D3-power |
| Drag and drop | `@dnd-kit/core@6` (modern), `react-beautiful-dnd@13` (deprecated, don't use) |
| Rich text editor | `@tiptap/react@2` (ProseMirror-based, modern), `lexical@0.x` (Meta-built) |
| Code editor | `@codemirror/state@6` + `@codemirror/view@6`, or `monaco-editor@0.49` for VS Code-quality |
| Color picker | `@uiw/react-color@2`, `react-colorful@5` |
| Combobox / command palette | `cmdk@1` (used by shadcn/ui) |
| Toast notifications | `sonner@1.5` (the shadcn/ui default), `react-hot-toast@2` |
| File upload | `react-dropzone@14`, `uppy@4` for chunked / resumable |

These pair with any of the libraries above. Use specialized libs for the hard parts; let shadcn/ui or MUI handle the surrounding chrome.

---

## 13. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Using `react-helmet@6` to set meta tags in MUI Next.js apps | Use Next.js Metadata API (covered in `07.nextjs/07-seo-metadata.md`) |
| MUI flickers on first paint in Next.js App Router | Wrap with `AppRouterCacheProvider` from `@mui/material-nextjs` |
| shadcn/ui components don't update when registry changes | Re-run `npx shadcn@2.1 add <component>` to pull updates manually |
| Bundling all of MUI even with tree-shaking | Use named imports (`import Button from '@mui/material/Button'`), enable `babel-plugin-transform-imports` if not on Vite |
| Mixing MUI + Tailwind | Possible but theme conflicts pile up. Pick one. |
| `'use client'` boilerplate on every shadcn/ui-using page | Most shadcn/ui components are already client components; you only need `'use client'` when *your code* uses hooks |
| Headless UI v1 → v2 migration breaks | v2 renamed several components and slot props; check the [v1→v2 migration guide](https://headlessui.com/v2/migration) |
| Radix UI animations don't run | `data-[state=open]:animate-in` requires `tailwindcss-animate@1` plugin |

---

## 14. Migration Notes (Real Scenarios)

### "We're on MUI but want to migrate to shadcn/ui"

Don't migrate everything at once. The realistic path:
1. Install shadcn/ui in parallel (it's just files, doesn't conflict).
2. New components/screens: shadcn/ui.
3. Existing MUI screens: leave alone. Migrate only when touching them anyway.
4. Goal: 1–2 years of gradual replacement.

### "We're on Chakra v2 — do we upgrade to v3?"

Probably not. Chakra v3 is a near-rewrite (Ark UI internals, Panda CSS) and the migration cost is comparable to switching to shadcn/ui. Many teams use the v2-EOL trigger to switch UI libraries entirely.

### "We're on Ant Design and want a more modern look"

Hard. Ant Design's component aesthetics permeate everything. Either accept the look (it's actually fine for enterprise data apps) or commit to a full rewrite — there's no shortcut.

---

## Summary

| Library | Pick when |
|---------|-----------|
| **shadcn/ui** + Radix UI 1.x + Tailwind 3.4 | New projects in 2026; you want design control + accessibility |
| **Radix UI Primitives 1.x** directly | Public design systems, highly custom UI |
| **Headless UI 2** | Tailwind UI ecosystem |
| **Ark UI 4** / Park UI | Multi-framework design system; state-machine internals |
| **MUI v6** | Material Design brief; need MUI X data grid; enterprise app |
| **Mantine 7** | Indie SaaS dashboards; want pre-styled + tons of hooks |
| **Chakra UI v3** | Already on Chakra v2 with strong team buy-in |
| **Ant Design 5** | Dense enterprise data apps; China-market |

| Rule | Why |
|------|-----|
| Default to shadcn/ui for new projects | Highest momentum, AI-friendly, you own the code |
| Radix UI is the foundation under shadcn/ui | Same primitives if you skip the wrapper |
| Don't mix MUI + Tailwind in one app | Theme conflicts pile up |
| For data tables, use `@tanstack/react-table@8` headless | Then style with whichever kit you chose |
| Plan for AI-generated code | shadcn/ui produces more reliable AI completions than MUI/Mantine |

---

## Further reading

- [shadcn/ui docs](https://ui.shadcn.com/)
- [shadcn/ui themes](https://ui.shadcn.com/themes)
- [shadcn studio — community blocks](https://shadcn.studio)
- [Radix UI Primitives](https://www.radix-ui.com/primitives)
- [Headless UI v2](https://headlessui.com/)
- [Ark UI](https://ark-ui.com/)
- [MUI v6 docs](https://mui.com/material-ui/getting-started/)
- [MUI X (data grid, charts, date pickers)](https://mui.com/x/)
- [Mantine 7 docs](https://mantine.dev/)
- [Chakra UI v3](https://chakra-ui.com/)
- [Ant Design 5](https://ant.design/)
