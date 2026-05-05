# Capabilities — 05. Accessibility: axe-core, Focus Management, Keyboard Nav, Screen-Reader Testing

> **What / Why / How** — wire **`@axe-core/react@4`** in dev, **`jest-axe@9`** in tests, ship semantic HTML first, and test with a real screen reader. Most a11y bugs are **missing labels, broken keyboard traps, and color contrast** — not exotic ARIA gymnastics.

---

## 1. Why Accessibility Is a 2026 Production Requirement

The honest framing: a11y isn't optional. Three independent reasons:

| Reason | Reality in 2026 |
|--------|-----------------|
| **Legal exposure** | ADA / Section 508 / EAA (European Accessibility Act, in force June 2025). Real lawsuits filed against B2C sites; the EAA covers SaaS and e-commerce in the EU. |
| **Audience size** | ~16% of users globally have a disability (WHO). Permanent (vision/hearing/motor), temporary (broken arm), or situational (bright sunlight, noisy commute). |
| **SEO and product polish** | Google's Lighthouse weights a11y heavily; semantic HTML + proper headings boost rankings; keyboard-friendly UIs feel faster to power users too. |

The European Accessibility Act (EAA) became enforceable June 28, 2025. Every consumer-facing digital product sold in the EU now legally needs to meet **WCAG 2.1 level AA**. This is the current baseline; WCAG 2.2 (Oct 2023) is what new audits actually use.

### The single biggest insight

**Most a11y bugs come from skipping semantic HTML.** A `<div>` with `role="button"` and 30 lines of keyboard-handler code is the wrong fix. The right fix is a `<button>`. The browser already knows it's focusable, fires on Enter/Space, has the right ARIA role, and announces correctly to screen readers.

Treat every `role="..."` attribute as a yellow flag. You're either re-implementing what HTML already does for free, or you're papering over the wrong primitive.

---

## 2. The Real Tooling in 2026

| Layer | Tool | npm | Best for |
|-------|------|-----|----------|
| **Dev-time runtime audit** | `@axe-core/react@4.10` | Loaded only in dev; logs violations to console | Catch issues during development |
| **Unit/integration tests** | `jest-axe@9.0` (also runs on Vitest) | Assert no violations on rendered components | CI gate per component |
| **E2E tests** | `@axe-core/playwright@4` or `axe-playwright@2` | Run against the live page | CI gate per page |
| **CI a11y check** | Lighthouse CI (`@lhci/cli@0.14`) + Pa11y or `axe-core/cli@4` | Enforce score thresholds per build | PR gates |
| **Browser audit** | axe DevTools (Chrome/Firefox extension), Lighthouse panel, WAVE | Manual spot-check during development | Visual review |
| **Color contrast** | `colorjs.io@0.6`, browser DevTools "Issues" tab, WCAG Contrast Checker | Ensure AA/AAA contrast | Design tooling |
| **Focus management** | `react-focus-lock@2.13`, `focus-trap-react@10` | Modal/dialog focus traps | Already built into Radix UI Dialog |
| **Component library with built-in a11y** | shadcn/ui (Radix UI), Headless UI 2, React Aria 3 | First-class keyboard + screen-reader support | Default UI primitive choice (covered in `08.ecosystem/02-ui-libraries.md`) |
| **Screen-reader testing** | NVDA (Win, free), VoiceOver (macOS/iOS, built-in), TalkBack (Android), JAWS (Win, paid) | Real testing, not auto-detected | Manual QA |

**The 80% rule**: install `@axe-core/react@4` in dev today, add `jest-axe@9` to component tests, use Radix-UI-based primitives via shadcn/ui, run NVDA or VoiceOver once a quarter on critical flows. That covers ~80% of real a11y issues.

---

## 3. Set Up Runtime Audits in Dev

`@axe-core/react@4.10` runs the real `axe-core` engine in your browser during development and logs violations to the console.

### Setup

```bash
npm i -D @axe-core/react@4.10
```

```tsx
// app/components/axe-init.tsx
'use client';
import { useEffect } from 'react';

export function AxeInit() {
  useEffect(() => {
    if (process.env.NODE_ENV !== 'production' && typeof window !== 'undefined') {
      void Promise.all([import('react'), import('react-dom'), import('@axe-core/react')])
        .then(([React, ReactDOM, axe]) => {
          axe.default(React.default, ReactDOM.default, 1000);
        });
    }
  }, []);
  return null;
}
```

```tsx
// app/layout.tsx — mount in dev
import { AxeInit } from './components/axe-init';

<body>
  <AxeInit />
  {children}
</body>
```

Now every render runs axe and logs issues like:

```
[axe-core]: Element has insufficient color contrast of 3.11 (foreground: #888, background: #fff). Expected ≥ 4.5:1.
   <p class="muted-text">…</p>
   Fix: change foreground to #6b7280 or background to #f3f4f6
```

Two-second delay before re-running prevents log spam during typing. Only loaded in dev — production builds skip the import entirely.

### What axe catches automatically

- Missing `alt` on `<img>`.
- Form inputs without labels.
- Insufficient color contrast (WCAG AA = 4.5:1 for body text, 3:1 for large text).
- Duplicate IDs.
- Missing landmark roles (`<main>`, `<nav>`).
- Heading levels skipping (h1 → h3 with no h2).
- Empty buttons / links.
- Improper ARIA roles, states, properties.
- Repeated `aria-label`s on the same scope.

### What axe can't catch

- Whether your `aria-label` text is actually meaningful.
- Focus order (logical sequence of tab stops).
- Whether keyboard navigation actually works (only that the affordance exists).
- Whether a screen reader announces things in the order users expect.
- Whether color alone conveys meaning (axe can detect some, not all).

For those, you need component tests + manual screen-reader passes.

---

## 4. Test-Time Audits — `jest-axe`

```bash
npm i -D jest-axe@9.0
```

`jest-axe@9` works under both Jest 29 and Vitest 1.6 (covered in `06.testing-perf/01-testing.md`).

```ts
// vitest.setup.ts
import { expect } from 'vitest';
import { toHaveNoViolations } from 'jest-axe';

expect.extend(toHaveNoViolations);
```

```tsx
// components/SignupForm.test.tsx
import { render } from '@testing-library/react';
import { axe } from 'jest-axe';
import { describe, it, expect } from 'vitest';
import { SignupForm } from './SignupForm';

describe('SignupForm', () => {
  it('has no a11y violations', async () => {
    const { container } = render(<SignupForm />);
    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });
});
```

Run it as part of your normal test suite. Failures look like:

```
expect(received).toHaveNoViolations()

Expected the HTML found at $('input[name="email"]') to have no violations:

<input type="email" name="email">

Received:
form-field-multiple-labels: Form fields should not have multiple label elements
   https://dequeuniversity.com/rules/axe/4.10/form-field-multiple-labels
```

### Per-component, not per-page

Run `axe` on each significant component (forms, modals, navs, data tables) — not on the entire app at once. Per-component runs are faster, isolate failures cleanly, and let you `toHaveNoViolations` for components you've already fixed.

For E2E coverage of the rendered page after JavaScript runs:

```ts
// e2e/dashboard.spec.ts (Playwright)
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test('dashboard has no a11y violations', async ({ page }) => {
  await page.goto('/dashboard');
  const results = await new AxeBuilder({ page }).analyze();
  expect(results.violations).toEqual([]);
});
```

---

## 5. Semantic HTML — The 80% Solution

Before reaching for ARIA, pick the right element. The browser already wired up most of what you need.

### The cheat sheet

| Use case | ❌ Don't | ✅ Do |
|----------|----------|-------|
| Clickable thing | `<div onClick>` | `<button onClick>` |
| Navigation link | `<span onClick>` | `<a href>` |
| Toggleable disclosure | `<div onClick>` + ARIA | `<details><summary>` |
| Form input | `<div contentEditable>` | `<input>`, `<textarea>` |
| Choice from many | `<div>` list with click handlers | `<select>` + `<option>` |
| Yes/no choice | Custom toggle | `<input type="checkbox">` |
| Page section | `<div className="header">` | `<header>`, `<nav>`, `<main>`, `<aside>`, `<footer>` |
| Heading | `<div className="title">` | `<h1>`–`<h6>` |
| List of items | `<div>` per item | `<ul><li>` or `<ol><li>` |
| Progress | Animated div | `<progress max="100" value="40">` |
| Modal/dialog | `<div role="dialog">` | `<dialog>` element OR Radix UI Dialog (recommended) |

### Why semantic HTML beats ARIA

A real `<button>` gives you:

- ✅ Focusable in tab order
- ✅ `Enter` and `Space` trigger click
- ✅ Default `role="button"` for screen readers
- ✅ Focus ring matches the OS theme
- ✅ Disabled state via `disabled` attribute, not `aria-disabled`
- ✅ Form-submit behavior with `type="submit"`

A `<div role="button" tabIndex={0} onKeyDown={...}>` gives you all of those **only if you remember every detail**. Most teams forget two or three.

### The First Rule of ARIA

**The first rule of ARIA is: don't use ARIA.** ([W3C Authoring Practices](https://www.w3.org/TR/using-aria/#rule1).) Use the right HTML element first. Only reach for ARIA when no native element does what you need (e.g., a fully-custom combobox, a tab list with scroll-into-view).

When you do need ARIA, prefer composed Radix UI primitives via shadcn/ui — they encode the patterns from [WAI-ARIA Authoring Practices](https://www.w3.org/WAI/ARIA/apg/patterns/) correctly.

---

## 6. Forms — The Single Highest-Leverage Area

Real production forms get a11y wrong more than anywhere else. The basics:

### Every input needs a real label

```tsx
// ❌ Placeholder as label — disappears on focus, bad contrast
<input type="email" placeholder="Email" />

// ❌ aria-label only — no visual label hurts cognitive accessibility
<input type="email" aria-label="Email" />

// ✅ Visible <label> tied via for/htmlFor
<label htmlFor="email">Email</label>
<input id="email" type="email" />

// ✅ Wrapping pattern — same effect, no id needed
<label>
  Email
  <input type="email" />
</label>
```

### Errors must be announced

```tsx
'use client';
import { useState } from 'react';

export function SignupForm() {
  const [error, setError] = useState<string | null>(null);

  return (
    <form>
      <label htmlFor="email">Email</label>
      <input
        id="email"
        type="email"
        required
        aria-invalid={!!error}
        aria-describedby={error ? 'email-error' : undefined}
      />
      {error && (
        <p id="email-error" role="alert" className="text-red-600">
          {error}
        </p>
      )}
    </form>
  );
}
```

`role="alert"` makes screen readers announce the error the moment it appears. `aria-describedby` ties the input to the error message so the SR reads "Email, invalid, please enter a valid email" when the user focuses the input again.

`react-hook-form@7.51` (covered in `04.state-and-data/01-forms.md`) gives you the right `aria-invalid` and `aria-describedby` wiring out of the box if you use its `<Controller>` and integrate with `formState.errors`.

### Loading states

```tsx
<button type="submit" disabled={isPending} aria-busy={isPending}>
  {isPending ? <span aria-hidden="true">⏳</span> : null}
  {isPending ? 'Saving…' : 'Save'}
  <span className="sr-only">{isPending ? 'Submitting form' : ''}</span>
</button>
```

`aria-busy="true"` on a submit button signals "wait, this is processing." The `sr-only` class hides text visually but exposes it to screen readers — Tailwind ships this class out of the box.

### Inline validation should not interrupt typing

Don't fire `aria-invalid` on every keystroke — screen readers will announce errors as the user types each character. Wait until blur or submit. `react-hook-form`'s `mode: 'onBlur'` is the right default.

---

## 7. Keyboard Navigation — The Tab Test

Run this test on every page: **unplug your mouse and try to use the app.**

What should work:
- **`Tab`** moves to next focusable element in DOM order; **`Shift+Tab`** moves backward.
- **`Enter`** activates buttons and form submit; **`Space`** activates buttons and toggles checkboxes.
- **`Esc`** closes modals, dropdowns, and tooltips.
- Arrow keys navigate **within** widgets (radio groups, listboxes, menus, tabs) — this is where Radix UI does the heavy lifting.
- Every interactive element shows a visible focus ring.

### Skip links — the underused win

Sighted keyboard users on a long page tab through 50+ links to reach main content. Fix with a skip link:

```tsx
// app/layout.tsx
<body>
  <a
    href="#main"
    className="sr-only focus:not-sr-only focus:fixed focus:left-4 focus:top-4 focus:z-50 focus:rounded focus:bg-blue-600 focus:px-3 focus:py-2 focus:text-white"
  >
    Skip to main content
  </a>
  <header>...</header>
  <main id="main" tabIndex={-1}>{children}</main>
</body>
```

Hidden until focused; appears as the first Tab stop. `tabIndex={-1}` on `<main>` lets it receive programmatic focus from the link.

### Focus indicators — never `outline: none` without a replacement

```css
/* ❌ Bad — invisible focus ring breaks keyboard nav */
button:focus { outline: none; }

/* ✅ Good — replace with a visible custom ring */
button:focus-visible {
  outline: 2px solid #2563eb;
  outline-offset: 2px;
}
```

`:focus-visible` is the modern selector — only shows the ring on keyboard focus, not on click. shadcn/ui's `Button` already uses this:

```tsx
className="focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2"
```

### Focus order matches DOM order

Don't reorder visually with CSS in ways that break logical Tab order. If `flex-direction: row-reverse` puts the "Submit" button visually first but the user Tabs through "Cancel" first, that's a bug.

For grids: `display: grid` + `order: -1` does the same. **Visual order should match DOM order** for any element that takes focus.

---

## 8. Focus Management in Modals and Dialogs

The hardest accessibility problem in SPAs: **focus trapping**.

When a modal opens:
- Focus must move into the modal (typically the first focusable element).
- `Tab` and `Shift+Tab` must cycle within the modal — no escaping into the page behind.
- `Esc` should close it.
- On close, focus must return to the element that triggered the modal.

### Use a library, not custom code

This is exactly the kind of thing `@radix-ui/react-dialog@1.1` handles correctly:

```tsx
// shadcn/ui Dialog — built on Radix UI 1.1
import { Dialog, DialogContent, DialogHeader, DialogTitle, DialogTrigger } from '@/components/ui/dialog';

<Dialog>
  <DialogTrigger asChild>
    <Button>Edit profile</Button>
  </DialogTrigger>
  <DialogContent>
    <DialogHeader>
      <DialogTitle>Edit profile</DialogTitle>
    </DialogHeader>
    <ProfileForm />
  </DialogContent>
</Dialog>
```

What you get for free:
- Focus moves into the dialog on open.
- Tab cycles within.
- Esc closes.
- Focus returns to the trigger button on close.
- Background is `inert` — screen readers can't accidentally read content behind the modal.
- `aria-modal="true"` and `role="dialog"` set automatically.

If you don't use Radix: `react-focus-lock@2.13` or `focus-trap-react@10` are the standalone primitives. **Don't roll your own.**

---

## 9. Skip the Off-Screen, Show the On-Screen — `sr-only` Class

Some content should be readable by screen readers but invisible visually:

```css
/* Tailwind 3.4's built-in class */
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}
```

Real uses:

```tsx
// Icon-only button with screen-reader text
<button>
  <TrashIcon aria-hidden="true" />
  <span className="sr-only">Delete task</span>
</button>

// Status announcement that's not visually shown
<div className="sr-only" role="status">
  Saved.
</div>

// Visible label hidden but kept for SR
<label>
  <span className="sr-only">Search</span>
  <SearchIcon className="absolute left-3 top-3" />
  <input type="search" placeholder="Search…" />
</label>
```

The Tailwind `not-sr-only` class undoes it (used in skip links).

### `aria-hidden` for decorative icons

```tsx
<button>
  <CheckIcon aria-hidden="true" />
  Save
</button>
```

The icon is decorative. The text already says "Save". Without `aria-hidden`, screen readers read out the icon's name (whatever filename or unicode it resolves to) before "Save" — clutter.

---

## 10. Color Contrast — Just Math

WCAG 2.1 AA contrast ratios:

| Content | Minimum ratio |
|---------|---------------|
| Body text (≤ 24px or ≤ 18.66px bold) | **4.5:1** |
| Large text (≥ 24px or ≥ 18.66px bold) | **3:1** |
| UI components, graphical objects (icon, focus ring, form border) | **3:1** |

WCAG 2.1 AAA is stricter (7:1 / 4.5:1) — usually only required for content for users with low vision.

### Tools

- **Chrome DevTools** → Inspect element → Styles → click the color square → contrast ratio shown directly.
- **`@figma/contrast-plugin`** — for Figma reviews before code.
- **`colorjs.io@0.6`** programmatically:
  ```ts
  import Color from 'colorjs.io';
  new Color('#888').contrast('#fff', 'WCAG21'); // 3.5 — fails AA
  ```
- **Tailwind 3.4 + shadcn/ui** ships color tokens that hit AA at the documented combos. Don't override `--muted-foreground` to `#bbb` and forget — that's a common regression.

### Real production rule

Any **new color combination** must hit 4.5:1 for text, 3:1 for UI. Add a Storybook visual regression check (`@storybook/addon-a11y@8`) so PRs that introduce low-contrast pairs fail CI.

### Don't rely on color alone

A red error message without text or icon is invisible to color-blind users (~4% of men, ~0.5% of women have some form of color-vision deficiency).

```tsx
// ❌ Color-only signal
<span className="text-red-600">Failed</span>

// ✅ Color + icon + text
<span className="text-red-600">
  <AlertCircleIcon aria-hidden="true" /> Failed
</span>
```

---

## 11. Live Regions — Announcing Dynamic Updates

Screen readers don't automatically announce DOM changes. You opt in with **live regions**.

```tsx
// 'polite' — wait for current speech to finish, then announce
<div aria-live="polite">
  {message}
</div>

// 'assertive' — interrupt current speech (use sparingly)
<div aria-live="assertive">
  {error}
</div>

// role='status' is shorthand for aria-live='polite' + aria-atomic='true'
<div role="status">
  {savedAt && `Saved at ${savedAt}`}
</div>

// role='alert' is shorthand for aria-live='assertive' + aria-atomic='true'
<div role="alert">
  {error}
</div>
```

Real usage:
- **Toast notifications**: `role="status"` for success, `role="alert"` for errors. `sonner@1.5` (covered in `08.ecosystem/02-ui-libraries.md`) wires this correctly by default.
- **Loading state changes**: "5 tasks loaded" announced after fetch resolves.
- **Form validation errors after submit**: announce immediately so the SR user knows submit failed.

Don't put a live region on a chat log that updates 50 messages per second — you'll DoS the screen reader. Throttle, batch, or only announce critical changes.

---

## 12. Real Screen-Reader Testing

Auto-tools cover ~30–40% of real a11y issues. The other ~60% needs human testing with a real screen reader.

### NVDA (Windows, free)

[nvaccess.org](https://www.nvaccess.org/download/) — open-source, the most-used screen reader globally. Pair with **Firefox** (the historical "best supported" combination) or Chrome.

Critical commands:
- **`NVDA + space`** — toggle browse/focus mode (most navigation issues come from getting these confused).
- **`H`** — next heading; cycle `1`–`6` for specific levels.
- **`F`** — next form field.
- **`B`** — next button.
- **`K`** — next link.
- **`L`** — next list.
- **`R`** — next region (landmark).
- **`Tab`** — next focusable element.

### VoiceOver (macOS / iOS, built-in)

`Cmd + F5` toggles on macOS. `Settings → Accessibility → VoiceOver` triple-click on iOS. Critical for testing **Safari** behavior, which differs from Chrome in subtle ways — especially around form controls and ARIA.

`VO` is the modifier (`Ctrl + Option`). Browse with `VO + arrow keys`. Read all from current position with `VO + A`.

### TalkBack (Android, built-in)

`Settings → Accessibility → TalkBack`. Test on a real device, not the emulator.

### What to actually test

A pragmatic 30-minute screen-reader test of a critical flow:
1. Land on the home page. Does the screen reader announce the page title and main content?
2. Tab through the navigation. Are link names meaningful? ("Click here" is a fail.)
3. Open a modal. Does focus move in? Does Esc close it? Does focus return to the trigger?
4. Fill out a form with errors. Are errors announced? Does the SR say which fields are wrong?
5. Submit successfully. Is the success announced via a live region?
6. Navigate to a new page. Is the new heading announced?

If those six work end-to-end, you're already ahead of 90% of websites. Add coverage for one or two specific journeys per quarter.

### Don't just test "does it work" — test "is it usable"

A site can pass every automated test and still be miserable to navigate. The real metric: **could a blind user complete a checkout / signup / search task at the same speed as a sighted user?** If not, where do they get stuck?

---

## 13. Lighthouse and CI Gates

Lighthouse audits are part of `06.testing-perf/02-performance.md`'s Core Web Vitals story, but the **Accessibility** category is its own automated audit.

### Local check

Chrome DevTools → Lighthouse → Accessibility → Generate report. Score 0–100 with a list of failures.

### CI gate with Lighthouse CI

```yaml
# .github/workflows/ci.yml
- run: npm ci && npm run build
- run: npm run start &
- run: npx wait-on http://localhost:3000
- run: npx @lhci/cli@0.14 collect --url=http://localhost:3000 --url=http://localhost:3000/dashboard
- run: npx @lhci/cli@0.14 assert --preset=lighthouse:no-pwa --assertions.categories.accessibility=>=0.95
```

This fails the build if accessibility score drops below 95. Tune the threshold to your team's pace — `>=0.90` is a common starting target, climbing to `>=0.98` over time.

### Pa11y for site-wide crawl

`pa11y@8` crawls multiple pages and reports violations. Useful for catching "page X has a div as a button" on a deep route nobody tested manually.

```bash
npx pa11y https://example.com
# or
npx pa11y-ci --sitemap https://example.com/sitemap.xml
```

---

## 14. Prefers — Respecting User Preferences

Three CSS media queries the user controls at the OS level:

| Preference | Media query | What to do |
|------------|-------------|------------|
| Reduced motion (vestibular disorders) | `prefers-reduced-motion: reduce` | Disable or shorten animations (covered in `08.ecosystem/01-animation.md`) |
| Color scheme | `prefers-color-scheme: dark` | Respect dark mode preference unless user overrode |
| High contrast | `prefers-contrast: more` | Boost contrast in your CSS variables |
| Reduced data | `prefers-reduced-data: reduce` | Skip auto-playing video, lazy-load aggressively |

In Tailwind 3.4: `motion-reduce:`, `dark:`, `contrast-more:` variants:

```tsx
<div className="transition motion-reduce:transition-none">
  ...
</div>
```

`useReducedMotion` hook from `framer-motion@11`:

```tsx
import { useReducedMotion } from 'framer-motion';
const reduced = useReducedMotion();
<motion.div animate={{ opacity: 1 }} transition={{ duration: reduced ? 0 : 0.3 }} />
```

**This is not optional.** Users with vestibular disorders get nauseated by animations they didn't ask for. Treat reduced motion as a hard requirement, not a "nice to have."

---

## 15. The Accessibility Tree — What the Screen Reader Sees

Browsers expose two trees: the **DOM** (HTML elements) and the **accessibility tree** (what assistive tech reads). They overlap but aren't identical.

### How to inspect

Chrome DevTools → Elements panel → "Accessibility" sidebar. Click any element to see its **computed name**, **role**, **state**, and **properties** as the SR sees them.

The most common debugging session:
1. SR reads "button" instead of "Save".
2. Inspect → name is empty.
3. The element is `<button><img src="save.svg" /></button>` — no text or alt.
4. Fix: add `<button aria-label="Save">` or visible text.

### Computed name precedence

The browser computes a name by checking, in order:
1. `aria-labelledby` (highest priority)
2. `aria-label`
3. Associated `<label>` (for form fields)
4. `<title>` attribute
5. **Text content of the element**
6. Placeholder, alt, etc., depending on element type

Knowing the order helps debug "why is the SR reading the wrong thing" — usually you've set an `aria-label` that's overriding visible text incorrectly.

---

## 16. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Using `<div onClick>` for buttons | Use `<button>`; you get focus, keyboard, role, and disabled state for free |
| `outline: none` on focus | Replace with `:focus-visible` ring matching theme |
| Image without alt | Add `alt="..."` describing content; use `alt=""` only for purely decorative |
| Form input with no label | Add `<label>` or `aria-label`; placeholder alone fails |
| Modal not trapping focus | Use Radix UI Dialog or `react-focus-lock@2.13` |
| Color contrast ratio 3.5:1 on body text | WCAG AA requires 4.5:1; deepen the foreground or lighten the background |
| Loading state silent for screen readers | Add `role="status"` or `aria-live="polite"` region |
| Custom checkbox/toggle missing keyboard support | Replace with `<input type="checkbox">` + visual styling, or use Radix UI primitives |
| Heading levels skipped (h1 → h3) | Maintain logical hierarchy; visual size is decorative, headings are structural |
| Generic link text ("click here") | Make text describe the target ("View pricing") |
| `aria-hidden="true"` on focusable element | Hides it from SR but keeps it focusable — confusing. Use `inert` to remove from both, or remove from tab order with `tabIndex={-1}` |
| `tabindex="5"` (positive value) | Don't. Positive tabindex breaks logical tab order. Use `0` (focusable in DOM order) or `-1` (programmatic only) |
| Notifications announced too aggressively | Use `aria-live="polite"` for non-critical; reserve `assertive` for errors |
| iOS VoiceOver reads form duplicates | Likely both `<label>` and `aria-label` set; pick one |
| Navigation built with `<div role="navigation">` | Use `<nav>` — same role, no extra attribute, semantic HTML |
| Screen reader misses live-region update on first render | The region must exist in DOM **before** content changes; render the empty region, update its content |

---

## 17. Decision Tree

```
What's the a11y situation?
│
├── Just starting a project → install @axe-core/react@4 in dev + jest-axe@9 in tests + use shadcn/ui (Radix UI)
├── Adding a11y to an existing app → run Lighthouse + axe DevTools, fix top 5 issues per page, add CI gate
├── Need to certify for EAA / ADA → hire a certified WCAG auditor (Deque, AccessibilityWorks, Tenon)
└── Just need to ship — no time for "fancy" a11y? → semantic HTML + Radix UI + sr-only labels covers 80% with zero extra effort

Picking primitives?
├── Want a11y for free → shadcn/ui (Radix UI 1.1) — covered in 08.ecosystem/02-ui-libraries.md
├── Tailwind UI fan → Headless UI 2
├── Multi-framework / state-machine internals → Ark UI 4 / React Aria 3
└── Rolling your own → don't, unless you're publishing a primitives library

CI gate?
├── Per-component → jest-axe@9 in Vitest/Jest tests
├── Per-page → @axe-core/playwright@4 in E2E suite
├── Whole-site crawl → pa11y-ci@4 against sitemap
└── Score threshold → Lighthouse CI (@lhci/cli@0.14) with accessibility >= 0.95

Manual testing cadence?
├── New feature → tab-test with keyboard before merging
├── Quarterly → 30-minute NVDA / VoiceOver walkthrough of critical flows
└── Pre-launch → full WCAG 2.1 AA audit (internal or external)
```

---

## 18. What This Topic Connects To

- **`05.routing-and-styling/02-styling.md`** — Tailwind's `sr-only`, `not-sr-only`, `motion-reduce:`, and `:focus-visible` patterns.
- **`08.ecosystem/02-ui-libraries.md`** — shadcn/ui (Radix UI) gives you a11y for free.
- **`08.ecosystem/01-animation.md`** — `prefers-reduced-motion` handling.
- **`04.state-and-data/01-forms.md`** — `react-hook-form` + `aria-invalid` + `aria-describedby` patterns.
- **`06.testing-perf/01-testing.md`** — `jest-axe@9` plugs into your existing Vitest/Jest setup.
- **`06.testing-perf/02-performance.md`** — Lighthouse CI covers a11y category alongside performance.
- **`10.production/04-observability.md`** — track Lighthouse scores per deploy as a real metric.

---

## Summary

| Tool | Pick when |
|------|-----------|
| `@axe-core/react@4.10` | Dev-time runtime audit |
| `jest-axe@9.0` | Per-component a11y assertion in Vitest/Jest |
| `@axe-core/playwright@4` | E2E a11y check |
| `@lhci/cli@0.14` | Lighthouse CI threshold gate |
| `pa11y-ci@4` | Whole-site crawl for violations |
| shadcn/ui (Radix UI) | Default UI primitives with a11y built in |
| `react-focus-lock@2.13` | Modal focus trap when not using Radix Dialog |
| NVDA (Win) / VoiceOver (macOS/iOS) / TalkBack (Android) | Real screen-reader testing |

| Rule | Why |
|------|-----|
| Use semantic HTML before reaching for ARIA | The browser's built-in roles are correct by default |
| Visible labels on every form input | Placeholder is not a label |
| Visible focus rings, always | `:focus-visible` keeps mouse users happy too |
| Trap focus inside modals; return on close | Radix UI Dialog handles this; don't roll your own |
| Color contrast ≥ 4.5:1 for body text | WCAG AA requirement |
| Don't rely on color alone for meaning | Add icon + text |
| Run NVDA/VoiceOver quarterly on critical flows | Auto-tools miss ~60% of real issues |
| Respect `prefers-reduced-motion` | Vestibular disorders are not negotiable |
| Treat EAA/ADA as a hard launch requirement | Real legal exposure since June 2025 |

---

## Further reading

- [WCAG 2.2 quick reference](https://www.w3.org/WAI/WCAG22/quickref/)
- [WAI-ARIA Authoring Practices](https://www.w3.org/WAI/ARIA/apg/patterns/)
- [Using ARIA — W3C](https://www.w3.org/TR/using-aria/)
- [Deque axe-core rules](https://dequeuniversity.com/rules/axe/4.10)
- [`@axe-core/react` GitHub](https://github.com/dequelabs/axe-core-npm/tree/develop/packages/react)
- [`jest-axe` docs](https://github.com/nickcolley/jest-axe)
- [Radix UI accessibility](https://www.radix-ui.com/primitives/docs/overview/accessibility)
- [React Aria docs](https://react-spectrum.adobe.com/react-aria/)
- [NVDA download](https://www.nvaccess.org/download/)
- [VoiceOver primer (Apple)](https://www.apple.com/voiceover/info/guide/_1124.html)
- [European Accessibility Act overview](https://ec.europa.eu/social/main.jsp?catId=1202)
