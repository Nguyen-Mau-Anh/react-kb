# React KB — Iteration Roadmap

This document defines every topic planned for the knowledge base, grouped by iteration theme.
Each session picks the next unfinished item — deeper related dives stay together inside the same group folder rather than each topic getting its own folder.
Mark items `[x]` as they are completed.

## Writing Standard (apply to every document)

Every claim about a library or tool must include:
- **Exact npm package and version** (`react-hook-form@7.51`, `@tanstack/react-query@5.28`) — or note that the version is dictated by the framework (e.g. Next.js 14 ships its own React build).
- **Why this library over alternatives** — not just what it does, but why it beats the named competitors in that scenario.
- **Named real scenarios** — "use X when Y" with Y being a concrete, recognizable situation (e.g. "a Next.js e-commerce site that needs ISR for product pages").
- **Links to official docs** — react.dev, nextjs.org, the library's own README — not random blog posts.
- Avoid: "some libraries", "certain tools", "various frameworks", "etc." — always name them.

Each topic must follow **What / Why / How** structure in that order.

---

## Group 1 — Fundamentals (Iteration 1) ✅
*Mental model and the language you write React in.*

- [x] `01.fundamentals/01-how-react-works.md` — VDOM, Fiber, reconciliation, concurrent features
- [x] `01.fundamentals/02-jsx-and-rendering.md` — JSX compilation, automatic runtime, createRoot vs hydrateRoot
- [x] `01.fundamentals/03-components-props.md` — Function components, prop typing, composition

---

## Group 2 — Hooks (Iteration 2) ✅
*Every built-in hook, with the pitfalls people actually hit.*

- [x] `02.hooks/01-usestate.md` — Updater fn, batching, derived state, lazy init, key reset
- [x] `02.hooks/02-useeffect.md` — When NOT to use it, deps array, cleanup, Strict Mode, race conditions
- [x] `02.hooks/03-useref-usecontext.md` — DOM refs, forwardRef, context perf, provider composition
- [x] `02.hooks/04-usememo-usecallback.md` — When memoization actually pays off, React Compiler future
- [x] `02.hooks/05-custom-hooks.md` — Extraction patterns, useFetch / useLocalStorage / useDebounce

---

## Group 3 — Rendering Patterns (Iteration 3) ✅
*How React builds UI from data.*

- [x] `03.rendering-patterns/01-event-handling.md` — SyntheticEvent, delegation, pointer events
- [x] `03.rendering-patterns/02-conditional-lists.md` — Key prop bugs, virtualization with `@tanstack/react-virtual@3`
- [x] `03.rendering-patterns/03-component-patterns.md` — Compound, render props, HOC, slot patterns — when each fits

---

## Group 4 — State & Data (Iteration 4) ✅
*Local state, global state, and server state.*

- [x] `04.state-and-data/01-forms.md` — Controlled vs uncontrolled, `react-hook-form@7.51` + `zod@3.23`
- [x] `04.state-and-data/02-state-management.md` — Context vs `zustand@4.5` vs `@reduxjs/toolkit@2`
- [x] `04.state-and-data/03-data-fetching.md` — `@tanstack/react-query@5.28` vs `swr@2.2` vs raw fetch

---

## Group 5 — Routing & Styling (Iteration 5) ✅
*Client routing and how to make things look right.*

- [x] `05.routing-and-styling/01-routing-react-router.md` — `react-router-dom@6.22` data routers, loaders, actions
- [x] `05.routing-and-styling/02-styling.md` — `tailwindcss@3.4` vs CSS Modules vs `styled-components@6.1`
- [x] `05.routing-and-styling/03-typescript-with-react.md` — Component types, generics, event types, utility types

---

## Group 6 — Testing & Performance (Iteration 6) ✅
*Confidence and speed.*

- [x] `06.testing-perf/01-testing.md` — `vitest@1.6` + `@testing-library/react@15` vs Jest 29
- [x] `06.testing-perf/02-performance.md` — `React.memo`, lazy/Suspense, profiler, bundle analysis with `rollup-plugin-visualizer`

---

## Group 7 — Next.js (Iterations 7–8) ✅
*The most common React framework today.*

- [x] `07.nextjs/01-fundamentals.md` — Next.js 14 App Router, file conventions, Server Components
- [x] `07.nextjs/02-routing.md` — Layouts, loading.tsx, error.tsx, parallel and intercepting routes
- [x] `07.nextjs/03-data-fetching.md` — fetch caching, revalidate, Server Actions, `useActionState`
- [x] `07.nextjs/04-auth.md` — `next-auth@5` (Auth.js), middleware, sessions, providers
- [x] `07.nextjs/05-deployment.md` — Vercel vs Docker on Railway/Fly.io vs self-host
- [x] `07.nextjs/06-api-routes.md` — Route Handlers, `trpc@11`, `hono@4` for serverless edge
- [x] `07.nextjs/07-seo-metadata.md` — Metadata API, Open Graph, JSON-LD, `next-sitemap`

---

## Group 8 — Ecosystem (Iterations 9+) ✅
*The libraries you reach for after the framework is set.*

- [x] `08.ecosystem/01-animation.md` — `framer-motion@11` vs `react-spring@9` vs CSS transitions
- [x] `08.ecosystem/02-ui-libraries.md` — `shadcn/ui` (Radix UI 1.x copy-paste) vs `@mui/material@6` vs Headless UI 2
- [x] `08.ecosystem/03-monorepo.md` — `turbo@2` vs `nx@19` for shared React component libraries
- [x] `08.ecosystem/04-real-project.md` — Full-stack Task Manager: Next.js 14 + Prisma 5 + tRPC 11 + NextAuth 5

---

## Group 9 — Beyond the Web (Iteration 10) 🚧
*React past the browser: native, real-time, AI, advanced state.*

- [x] `09.beyond-web/01-react-native-expo.md` — React Native with Expo SDK 51, `expo-router@3`, Tamagui 1 vs Nativewind 4
- [x] `09.beyond-web/02-react-19-features.md` — `use()`, Actions, `useOptimistic`, `useFormStatus`, React Compiler in production
- [x] `09.beyond-web/03-realtime.md` — `partykit@0.x` vs `liveblocks@2` vs Supabase Realtime vs `socket.io@4`
- [ ] `09.beyond-web/04-ai-integration.md` — Vercel AI SDK 4, streaming chat UIs, tool-calling agents, RAG patterns
- [ ] `09.beyond-web/05-state-machines.md` — `xstate@5` for wizards, multi-step checkouts, drag-snap interactions

When Group 9 is done, append five new topics and continue (likely: rich text editors, drag/kanban deep dive, charts, observability, mobile DX).
