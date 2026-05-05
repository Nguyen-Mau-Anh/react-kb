# react-kb

A personal knowledge base for **React + Next.js** — built one topic per session, written to read whenever you have free time.

Every topic follows the **What / Why / How** structure with real names, real use cases, and real code. No "some libraries", no generic descriptions.

📋 Full curriculum and progress tracker: [`00.meta/roadmap.md`](./00.meta/roadmap.md)

---

## Groups

Each group is a folder containing related topics as individual `.md` files (same convention as `springboot-kb`).

### 1. Fundamentals — `01.fundamentals/`
The mental model and the language you write React in.
- [01 — How React Works: VDOM, Fiber, Reconciliation](./01.fundamentals/01-how-react-works.md) ✅
- [02 — JSX & Rendering: what JSX compiles to, ReactDOM.createRoot](./01.fundamentals/02-jsx-and-rendering.md) ✅
- [03 — Components & Props: function components, prop typing, composition](./01.fundamentals/03-components-props.md) ✅

### 2. Hooks — `02.hooks/`
Every built-in hook with the pitfalls people actually hit.
- [01 — useState: batching, derived state, lazy init, key reset](./02.hooks/01-usestate.md) ✅
- [02 — useEffect: when NOT to use it, deps, cleanup, Strict Mode](./02.hooks/02-useeffect.md) ✅
- [03 — useRef & useContext: DOM access, forwardRef, context perf](./02.hooks/03-useref-usecontext.md) ✅
- [04 — useMemo & useCallback: when memoization pays off](./02.hooks/04-usememo-usecallback.md) ✅
- [05 — Custom hooks: extraction patterns, useFetch/useDebounce](./02.hooks/05-custom-hooks.md) ✅

### 3. Rendering Patterns — `03.rendering-patterns/`
How React builds UI from data.
- [01 — Event Handling: SyntheticEvent, delegation, pointer events](./03.rendering-patterns/01-event-handling.md) ✅
- [02 — Conditional rendering & lists: key prop, virtualization](./03.rendering-patterns/02-conditional-lists.md) ✅
- [03 — Component patterns: compound, render props, HOC, slots](./03.rendering-patterns/03-component-patterns.md) ✅

### 4. State & Data — `04.state-and-data/`
Local state, global state, and server state.
- [01 — Forms: controlled vs uncontrolled, react-hook-form 7 + Zod 3](./04.state-and-data/01-forms.md) ✅
- [02 — State management: Context vs Zustand 4 vs Redux Toolkit 2](./04.state-and-data/02-state-management.md) ✅
- [03 — Data fetching: TanStack Query v5 vs SWR 2](./04.state-and-data/03-data-fetching.md) ✅

### 5. Routing & Styling — `05.routing-and-styling/`
- [01 — Routing: react-router-dom 6, data routers, loaders, actions](./05.routing-and-styling/01-routing-react-router.md) ✅
- [02 — Styling: Tailwind 3 vs CSS Modules vs styled-components 6](./05.routing-and-styling/02-styling.md) ✅
- [03 — TypeScript with React: types, generics, utility types](./05.routing-and-styling/03-typescript-with-react.md) ✅

### 6. Testing & Performance — `06.testing-perf/`
- [01 — Testing: Vitest 1 + RTL 15 vs Jest 29](./06.testing-perf/01-testing.md) ✅
- [02 — Performance: React.memo, lazy/Suspense, profiler, bundle analysis](./06.testing-perf/02-performance.md) ✅

### 7. Next.js — `07.nextjs/`
The most common React framework today.
- [01 — Fundamentals: App Router, file conventions, Server Components](./07.nextjs/01-fundamentals.md) ✅
- [02 — Routing: layouts, loading, error, parallel, intercepting](./07.nextjs/02-routing.md) ✅
- [03 — Data fetching: fetch caching, revalidate, Server Actions](./07.nextjs/03-data-fetching.md) ✅
- [04 — Auth: NextAuth 5 (Auth.js), middleware, sessions](./07.nextjs/04-auth.md) ✅
- [05 — Deployment: Vercel vs Docker on Railway vs self-host](./07.nextjs/05-deployment.md) ✅
- [06 — API Routes: Route Handlers + tRPC 11 + Hono 4](./07.nextjs/06-api-routes.md) ✅
- [07 — SEO & Metadata: Metadata API, OG, JSON-LD](./07.nextjs/07-seo-metadata.md) ✅

### 8. Ecosystem — `08.ecosystem/`
Libraries you reach for after the framework is set.
- [01 — Animation: Framer Motion 11 vs React Spring 9](./08.ecosystem/01-animation.md) ✅
- [02 — UI libraries: shadcn/ui vs MUI 6 vs Headless UI 2](./08.ecosystem/02-ui-libraries.md) ✅
- [03 — Monorepo: Turbo 2 vs Nx 19 for shared component libs](./08.ecosystem/03-monorepo.md) ✅
- [04 — Real project: Next.js 14 + Prisma 5 + tRPC 11 + NextAuth 5](./08.ecosystem/04-real-project.md) ✅

### 9. Beyond the Web — `09.beyond-web/`
React past the browser: native, real-time, AI, advanced state.
- [01 — React Native with Expo SDK 51, expo-router, Tamagui vs Nativewind](./09.beyond-web/01-react-native-expo.md) ✅
- [02 — React 19 features: use(), Actions, useOptimistic, the Compiler](./09.beyond-web/02-react-19-features.md) ✅
- [03 — Real-time: PartyKit vs Liveblocks 2 vs Supabase Realtime vs Socket.IO 4](./09.beyond-web/03-realtime.md) ✅
- [04 — AI integration: Vercel AI SDK 4, streaming, tool-calling, RAG](./09.beyond-web/04-ai-integration.md) ✅
- [05 — State machines: xstate 5 for wizards and multi-step flows](./09.beyond-web/05-state-machines.md) ✅

### 10. Production Polish — `10.production/`
The work that turns "ships" into "scales."
- [01 — Rich text editors: Tiptap 2 vs Lexical vs Slate](./10.production/01-rich-text-editors.md) ⏳
- [02 — Drag & kanban: dnd-kit deep dive, sortable lists, virtualized drag](./10.production/02-drag-and-kanban.md) ⏳
- [03 — Charts & viz: Recharts 2 vs ECharts vs visx vs Nivo](./10.production/03-charts-and-viz.md) ⏳
- [04 — Observability: Sentry 8, OpenTelemetry, Vercel Analytics + Speed Insights](./10.production/04-observability.md) ⏳
- [05 — i18n: next-intl 3 vs react-i18next 15, ICU MessageFormat, RTL](./10.production/05-i18n.md) ⏳

✅ Done · ⏳ Pending

---

## Where to start

If you're picking this up fresh, read the groups in numerical order — earlier groups establish vocabulary used later. Inside a group, follow the file numbering.

If you already know React core and want only Next.js, skip to Group 7. Group 8 assumes Groups 1–7.

## Where to continue

Open [`00.meta/roadmap.md`](./00.meta/roadmap.md) — the next `[ ]` unchecked box is the next session.
