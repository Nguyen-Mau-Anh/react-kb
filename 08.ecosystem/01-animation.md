# Ecosystem — 01. Animation: Framer Motion 11 vs React Spring 9 vs CSS Transitions

> **What / Why / How** — start with CSS. Reach for Framer Motion when you need orchestration. Pick React Spring only for physics. View Transitions for navigation animations.

---

## 1. The Real Animation Stack in 2026

| Tool | npm | Bundle gzipped | Best for |
|------|-----|----------------|----------|
| **CSS transitions / animations** | built-in | 0 | Hover, focus, simple state-driven UI changes |
| **Tailwind utilities + `tailwindcss-animate`** | `tailwindcss-animate@1.0.7` | tiny | The same as CSS, but composable like any utility |
| **Framer Motion 11** | `framer-motion@11` | ~32 KB (LazyMotion: ~12 KB) | Orchestrated animations, layout animations, drag, gestures, page transitions |
| **React Spring 9** | `@react-spring/web@9.7` | ~16 KB | Physics-based animations (springs, decay), interactive simulations |
| **GSAP 3.12** | `gsap@3.12` | ~26 KB core | Complex timelines, SVG morphing, scroll-driven animation |
| **Motion One 10** | `motion@10` | ~5 KB (`animate` only) | Web Animations API wrapper, lightweight successor to Framer Motion's primitives |
| **CSS View Transitions API** | built-in (Chrome/Edge/Safari 18) | 0 | Cross-route animations triggered by navigation |
| **`auto-animate@0.8`** | `@formkit/auto-animate@0.8` | ~3 KB | "Add one line" animation for list/append/remove |

The honest 2026 default: **CSS first** (or Tailwind utilities), reach for **Framer Motion** when CSS can't express what you need. Skip React Spring unless you specifically want physics.

---

## 2. CSS First — The 80% Solution

A surprising number of "animations" are state changes. CSS transitions handle them with zero JS:

```tsx
// Tailwind CSS — toggle a class, get a transition
function Card({ open }: { open: boolean }) {
  return (
    <div
      className={`overflow-hidden transition-[max-height] duration-300 ease-in-out
                  ${open ? 'max-height-[500px]' : 'max-h-0'}`}
    >
      <p className="p-4">Body</p>
    </div>
  );
}
```

What CSS handles well:
- Hover, focus, active states
- Modal fade-in / fade-out
- Sidebar slide
- Button press scale
- Loading spinner (`@keyframes spin`)
- Mounted-on-page enter/exit (with the View Transitions API)

What CSS struggles with:
- Coordinating multiple elements with staggered delays
- Animating from `auto` height (you have to pick a max)
- Spring physics (CSS only does cubic-bezier easing)
- Interrupting and reversing mid-animation
- Drag, gestures, layout transitions when items reorder

When you hit the second list, reach for Framer Motion.

### `tailwindcss-animate@1.0.7` — the shadcn/ui default

`shadcn/ui` ships with this plugin enabled. It exposes Tailwind utilities for fade/slide/zoom variants used by every Radix-driven primitive:

```tsx
<div className="data-[state=open]:animate-in data-[state=closed]:animate-out
                data-[state=open]:fade-in-0 data-[state=open]:zoom-in-95
                duration-200">
  …
</div>
```

`data-[state=open]` is the standard Radix UI primitive convention; the animate utility automatically responds.

---

## 3. Framer Motion 11 — The Production Default

### What

`framer-motion@11` is a React animation library by the Framer team. It exposes a `<motion.*>` component for every HTML/SVG element, an `animate` function, and a small set of orchestration components.

```tsx
import { motion } from 'framer-motion';

<motion.div
  initial={{ opacity: 0, y: 12 }}
  animate={{ opacity: 1, y: 0 }}
  transition={{ duration: 0.25, ease: 'easeOut' }}
>
  Hello
</motion.div>
```

### Why pick it

- **Imperative-feeling API on top of declarative React.** You describe "from → to" instead of writing keyframes.
- **`AnimatePresence`** — cleanly animates components out before they unmount. CSS can't do this without DIY tricks.
- **`layout` prop** — automatic FLIP animations when items reorder, expand, or move between containers.
- **Drag, gestures, hover/tap variants** — built-in, accessibility-aware.
- **Production-grade**: used by Vercel, Linear, Cal.com, Resend's marketing site.

### Real example — list reorder with FLIP animation

```tsx
import { motion, AnimatePresence } from 'framer-motion';

function TodoList({ todos }: { todos: Todo[] }) {
  return (
    <ul>
      <AnimatePresence>
        {todos.map(todo => (
          <motion.li
            key={todo.id}
            layout                                              // animate position changes
            initial={{ opacity: 0, x: -20 }}
            animate={{ opacity: 1, x: 0 }}
            exit={{ opacity: 0, x: 20 }}
            transition={{ duration: 0.2 }}
          >
            {todo.text}
          </motion.li>
        ))}
      </AnimatePresence>
    </ul>
  );
}
```

When a todo is deleted, it fades right and the others slide up to fill the gap — no manual measurement, no `requestAnimationFrame` loop.

### Real example — drag to dismiss

```tsx
<motion.div
  drag="x"
  dragConstraints={{ left: 0, right: 0 }}
  dragElastic={0.2}
  onDragEnd={(_, info) => {
    if (Math.abs(info.offset.x) > 100) onDismiss();
  }}
  whileTap={{ scale: 0.98 }}
>
  Swipe me
</motion.div>
```

`whileTap`, `whileHover`, `whileDrag` are declarative variants that don't require event handlers.

### Reduce bundle with `LazyMotion`

The `<motion.*>` components import every feature. To trim the bundle, opt into a "domAnimation" or "domMax" feature set:

```tsx
import { LazyMotion, domAnimation, m } from 'framer-motion';

<LazyMotion features={domAnimation} strict>
  <m.div animate={{ opacity: 1 }} />
</LazyMotion>
```

`m` is the bare-bones component (~3 KB); features are loaded via `LazyMotion`. Total: ~12 KB instead of 32. The `strict` flag bans `<motion.*>` so you don't accidentally unify both.

### Variants — orchestrating multiple elements

```tsx
const list = {
  hidden: {},
  visible: { transition: { staggerChildren: 0.06 } },
};
const item = {
  hidden:  { opacity: 0, y: 10 },
  visible: { opacity: 1, y: 0 },
};

<motion.ul variants={list} initial="hidden" animate="visible">
  {items.map(i => <motion.li key={i.id} variants={item}>{i.text}</motion.li>)}
</motion.ul>
```

Children automatically inherit the parent's animation state and are staggered. This pattern is in every Vercel example app for hero-section reveals.

### Page transitions in Next.js 14 App Router

Next.js doesn't expose a built-in route-change event for App Router. The pattern that works:

```tsx
// app/template.tsx — re-mounts on every navigation
'use client';
import { motion } from 'framer-motion';

export default function Template({ children }: { children: React.ReactNode }) {
  return (
    <motion.div
      initial={{ opacity: 0, y: 8 }}
      animate={{ opacity: 1, y: 0 }}
      transition={{ duration: 0.2 }}
    >
      {children}
    </motion.div>
  );
}
```

`template.tsx` (covered in `07.nextjs/02-routing.md`) re-mounts on each navigation, so the animation runs.

For full cross-page **morph** transitions (e.g. a product card growing into a product page), the View Transitions API is now the better tool — see §6.

---

## 4. React Spring 9 — When You Want Physics

### What

`@react-spring/web@9.7` animates with **spring physics**: tension, friction, mass. You don't pick a duration; the spring resolves naturally.

```tsx
import { useSpring, animated } from '@react-spring/web';

function FadeIn() {
  const styles = useSpring({
    from: { opacity: 0, y: 20 },
    to:   { opacity: 1, y: 0 },
    config: { tension: 280, friction: 24 },
  });
  return <animated.div style={styles}>Hi</animated.div>;
}
```

### Why pick it

- **Physics feel**: bouncy menus, drag-snap, pull-to-refresh.
- **Interruptibility**: a spring already in motion smoothly retargets when state changes — no jump.
- **Imperative API option** (`useSpringRef`, `useChain`) for sequenced animations without React state.

### When NOT to pick React Spring

- Most UI animations don't need physics — duration-based easing reads fine, and Framer Motion's `type: 'spring'` covers the few that do.
- Smaller community than Framer Motion in 2026; fewer ecosystem examples for Next.js setups.

### Real fit

`react-use-gesture@9.x` (now `@use-gesture/react@10`) + React Spring is the classic combo for **drag handles, sortable lists with snap, pull-to-refresh**. If you're building a touch-heavy mobile-web UI, this pair is still the most polished option in 2026.

---

## 5. GSAP 3.12 — When You Need a Real Timeline

### What

GSAP is a non-React, framework-agnostic animation engine. `gsap@3.12` is the most powerful and battle-tested timeline tool on the web.

```tsx
import { useGSAP } from '@gsap/react';
import gsap from 'gsap';
import { ScrollTrigger } from 'gsap/ScrollTrigger';
gsap.registerPlugin(ScrollTrigger);

function HeroAnimation() {
  const ref = useRef<HTMLDivElement>(null);
  useGSAP(() => {
    gsap.timeline({ scrollTrigger: { trigger: ref.current, start: 'top 80%' } })
      .from('.hero-title', { y: 40, opacity: 0, duration: 0.6 })
      .from('.hero-sub',   { y: 20, opacity: 0, duration: 0.4 }, '-=0.3')
      .from('.hero-cta',   { scale: 0.9, opacity: 0, duration: 0.4 }, '-=0.2');
  }, { scope: ref });

  return (
    <div ref={ref}>
      <h1 className="hero-title">Headline</h1>
      <p className="hero-sub">Subhead</p>
      <button className="hero-cta">CTA</button>
    </div>
  );
}
```

### When pick GSAP

- **Long timelines** with many keyframes and overlap (`'-=0.3'` syntax).
- **ScrollTrigger** — by far the best scroll-driven animation tool on the web.
- **SVG morphing** — `MorphSVGPlugin` handles arbitrary path interpolation.
- **Marketing sites** — Awwwards-tier scroll experiences.

GSAP is **not free** for everything: `MorphSVGPlugin`, `SplitText`, `Flip` and a few others require a paid Club GreenSock membership ($99/yr). Core GSAP and `ScrollTrigger` are free under their own license (free for production use).

### When NOT to pick GSAP

- Component-state-driven UI animations — Framer Motion is more idiomatic in React.
- Anything Framer Motion's `layout` prop already does — GSAP's `Flip` plugin solves the same problem but is paid.

---

## 6. View Transitions API — Browser-Native Cross-Page Animation

### What

The CSS View Transitions API (Chrome 111+, Safari 18, Edge 111+; Firefox in development as of 2026) lets the browser take a snapshot of the old DOM, swap to the new DOM, and animate between them — including **cross-document transitions** for SPA-style navigations.

### Same-document transitions (in a SPA)

```ts
// Plain JS API
document.startViewTransition(() => {
  setState(newState);
});
```

CSS opts elements into specific transitions:

```css
::view-transition-old(root),
::view-transition-new(root) {
  animation-duration: 0.3s;
}

/* match elements by view-transition-name */
.product-card {
  view-transition-name: product-card-42; /* dynamic per ID */
}
```

### Next.js 14 + View Transitions

Next.js 14.2 added experimental support:

```js
// next.config.js
module.exports = {
  experimental: { viewTransition: true },
};
```

Then in your component:

```tsx
'use client';
import { unstable_ViewTransition as ViewTransition } from 'react';

<ViewTransition name="hero">
  <Image src="/hero.jpg" />
</ViewTransition>
```

Now navigating to a page with the same `name="hero"` triggers a browser-level morph between the two images. **No JavaScript animation** — the browser does it.

This is the future-direction for cross-page animations. Framer Motion's `LayoutGroup` + `layoutId` handled this case before, but at 30 KB; View Transitions does it at 0 KB once the browser support lands fully.

---

## 7. `auto-animate` — Zero-Effort List Animation

For 80% of the "animate this list when items appear/disappear/reorder" cases, `@formkit/auto-animate@0.8` is a one-liner:

```tsx
import { useAutoAnimate } from '@formkit/auto-animate/react';

function TodoList({ todos }: { todos: Todo[] }) {
  const [parent] = useAutoAnimate();
  return (
    <ul ref={parent}>
      {todos.map(t => <li key={t.id}>{t.text}</li>)}
    </ul>
  );
}
```

That's it. Insertions fade-slide in, removals fade out, reorders slide. ~3 KB. When you don't need the customization Framer Motion offers, this is the cheapest possible animation.

---

## 8. Accessibility — `prefers-reduced-motion`

Some users disable animations at the OS level (Windows, macOS, iOS, Android all support it). Always respect this.

### CSS

```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}
```

### Framer Motion

```tsx
import { useReducedMotion, motion } from 'framer-motion';

function Animated() {
  const reduce = useReducedMotion();
  return (
    <motion.div
      initial={reduce ? false : { opacity: 0 }}
      animate={{ opacity: 1 }}
    />
  );
}
```

Or set the global `MotionConfig`:

```tsx
import { MotionConfig } from 'framer-motion';
<MotionConfig reducedMotion="user">{children}</MotionConfig>
```

### React Spring

`useReducedMotion` exists in `@react-spring/web` too, with the same semantics.

**This is not optional.** Vestibular-disorder users get nauseated by parallax and large motion. Treat reduced-motion as a real production requirement.

---

## 9. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Animating `height` from `auto` to `0` doesn't work in CSS | Use Framer Motion's `<motion.div animate={{ height: 'auto' }}>` (it measures), or `react-collapsed@4` |
| `AnimatePresence` exit animation doesn't run | Children must have a stable `key`; `AnimatePresence` only sees keyed children |
| `layout` prop animates everything, even initial mount | Add `initial={false}` to skip the first render's animation |
| Janky animation on slow devices | Move from CSS `top/left` to `transform: translate()` — composited on the GPU |
| Bundle bloat from full `framer-motion` import | Use `LazyMotion` + `m` |
| Animation continues after unmount | Use `AnimatePresence` properly, or React Spring's `cancel` |
| Page-transition flash of unstyled content | `template.tsx` re-mounts; ensure the page element is the wrapper |
| GSAP scroll trigger fires on Next.js soft nav | Reset triggers on route change with `ScrollTrigger.killAll()` in a layout effect |
| `prefers-reduced-motion` ignored | Add the CSS media query AND use `useReducedMotion` in JS animations |

---

## 10. Decision Tree

```
Is it just a hover/focus/state-toggle effect?
│   YES → CSS or Tailwind utilities. Done.
│   NO  ↓
List items appearing / disappearing / reordering?
│   YES → @formkit/auto-animate@0.8 (cheapest)
│         OR framer-motion@11 with `layout` + `AnimatePresence` (more control)
│   NO  ↓
Cross-page morph (image → page hero)?
│   YES → React's experimental ViewTransition (Next.js 14.2+) when browser support is enough
│         OR framer-motion@11's `layoutId` everywhere else
│   NO  ↓
Drag, gestures, swipe-to-dismiss?
│   YES → framer-motion@11 (declarative) or @use-gesture/react@10 + @react-spring/web@9.7 (physics-feel)
│   NO  ↓
Long timeline, scroll-driven, SVG morph, marketing-site polish?
│   YES → gsap@3.12 + ScrollTrigger
│   NO  ↓
Complex orchestration, staggered children, layout animations?
│   YES → framer-motion@11
│   NO  → Reach for CSS first
```

---

## 11. Summary

| Tool | Pick when |
|------|-----------|
| CSS / Tailwind utilities | Hover, focus, simple toggles |
| `tailwindcss-animate@1` | Tailwind project using shadcn/ui Radix primitives |
| `@formkit/auto-animate@0.8` | List append/remove/reorder, one-line solution |
| `framer-motion@11` (with `LazyMotion`) | Production default for component animations, layout, drag, gestures |
| `@react-spring/web@9.7` + `@use-gesture/react@10` | Physics-feel mobile-web gestures |
| `gsap@3.12` + `ScrollTrigger` | Scroll-driven timelines, SVG morph, marketing polish |
| `motion@10` | Lightweight Web Animations API wrapper |
| React `<ViewTransition>` (Next.js 14.2+) | Cross-page browser-native morphs |

| Rule | Why |
|------|-----|
| Default to CSS first | 0 KB, GPU-accelerated, browser-optimized |
| Use `framer-motion@11` with `LazyMotion` | Cuts the bundle from 32 KB to ~12 KB |
| Always respect `prefers-reduced-motion` | Accessibility requirement, not optional |
| Animate `transform` and `opacity`, not `top/left/width` | Composited on the GPU; no layout reflow |
| `key` matters in `AnimatePresence` | Exit animations need stable identity |

---

## Further reading

- [Framer Motion docs](https://www.framer.com/motion/)
- [React Spring docs](https://www.react-spring.dev/)
- [GSAP docs](https://gsap.com/docs/v3/)
- [`@formkit/auto-animate` docs](https://auto-animate.formkit.com/)
- [MDN — View Transitions API](https://developer.mozilla.org/en-US/docs/Web/API/View_Transitions_API)
- [Next.js — experimental viewTransition](https://nextjs.org/docs/app/api-reference/config/next-config-js/viewTransition)
- [web.dev — High-performance animations](https://web.dev/articles/animations-guide)
