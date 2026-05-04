# react-kb

A personal knowledge base for React + Next.js — built one topic per session, written to read when you have free time.

Every topic follows the **What / Why / How** structure with real names, real use cases, and real code. No "some libraries", no generic descriptions.

## Curriculum

| # | Topic | Status |
|---|-------|--------|
| 00 | [How React Works: VDOM, Fiber, Reconciliation](./00-how-react-works/README.md) | ✅ Done |
| 01 | [JSX & Rendering: what JSX compiles to, ReactDOM.createRoot](./01-jsx-and-rendering/README.md) | ✅ Done |
| 02 | [Components & Props: function components, prop types, composition](./02-components-props/README.md) | ✅ Done |
| 03 | [State with useState: local state, batching, derived state](./03-state-usestate/README.md) | ✅ Done |
| 04 | [Side Effects with useEffect: deps array, cleanup, pitfalls](./04-useeffect/README.md) | ✅ Done |
| 05 | [Event Handling: synthetic events, delegation, patterns](./05-event-handling/README.md) | ✅ Done |
| 06 | [Conditional Rendering & Lists: key prop, reconciliation impact](./06-conditional-lists/README.md) | ✅ Done |
| 07 | [Forms: controlled vs uncontrolled, react-hook-form 7 vs Formik 2](./07-forms/README.md) | ⏳ Pending |
| 08 | [useRef & useContext: DOM access, context patterns, pitfalls](./08-useref-usecontext/README.md) | ⏳ Pending |
| 09 | [useMemo & useCallback: when to memoize, real perf impact](./09-usememo-usecallback/README.md) | ⏳ Pending |
| 10 | [Custom Hooks: extraction patterns, useFetch, useLocalStorage](./10-custom-hooks/README.md) | ⏳ Pending |
| 11 | [Component Patterns: compound, render props, HOC](./11-component-patterns/README.md) | ⏳ Pending |
| 12 | [State Management: Context vs Zustand 4 vs Redux Toolkit 2](./12-state-management/README.md) | ⏳ Pending |
| 13 | [Data Fetching: TanStack Query v5 vs SWR 2 vs plain useEffect](./13-data-fetching/README.md) | ⏳ Pending |
| 14 | [Routing with React Router v6: loaders, actions, nested routes](./14-routing-react-router/README.md) | ⏳ Pending |
| 15 | [Styling: Tailwind CSS 3 vs CSS Modules vs styled-components 6](./15-styling/README.md) | ⏳ Pending |
| 16 | [TypeScript with React: component types, generics, event types](./16-typescript-with-react/README.md) | ⏳ Pending |
| 17 | [Testing: Vitest + React Testing Library vs Jest](./17-testing/README.md) | ⏳ Pending |
| 18 | [Performance: React.memo, lazy/Suspense, profiler, bundle splits](./18-performance/README.md) | ⏳ Pending |
| 19 | [Next.js 14 Fundamentals: App Router, file conventions, RSC](./19-nextjs-fundamentals/README.md) | ⏳ Pending |
| 20 | [Next.js Routing: layouts, loading, error, parallel, intercepting](./20-nextjs-routing/README.md) | ⏳ Pending |
| 21 | [Next.js Data Fetching: fetch caching, revalidate, Server Actions](./21-nextjs-data-fetching/README.md) | ⏳ Pending |
| 22 | [Next.js Auth: NextAuth.js 5 (Auth.js), middleware, session](./22-nextjs-auth/README.md) | ⏳ Pending |
| 23 | [Next.js Deployment: Vercel vs Docker on Railway vs self-host](./23-nextjs-deployment/README.md) | ⏳ Pending |
| 24 | [Next.js API Routes & Route Handlers: REST + tRPC + Hono](./24-nextjs-api-routes/README.md) | ⏳ Pending |
| 25 | [Next.js SEO: Metadata API, Open Graph, JSON-LD, sitemap](./25-nextjs-seo-metadata/README.md) | ⏳ Pending |
| 26 | [Animation: Framer Motion 11 vs React Spring 9 vs CSS transitions](./26-animation/README.md) | ⏳ Pending |
| 27 | [UI Libraries: shadcn/ui vs Radix UI vs MUI 6 — trade-offs](./27-ui-component-libraries/README.md) | ⏳ Pending |
| 28 | [Monorepo with Nx or Turborepo: shared React component libs](./28-monorepo-nx/README.md) | ⏳ Pending |
| 29 | [Real Project: full-stack Task Manager (Next.js + Prisma + tRPC)](./29-real-project/README.md) | ⏳ Pending |

## Structure

Each topic lives in its own folder: `NN-slug/README.md`. Progress is tracked in `progress.json` at the root.

## Roadmap

- **Topics 00–18**: React core (hooks, patterns, state, testing, performance)
- **Topics 19–25**: Next.js 14 App Router in depth
- **Topics 26–29**: Ecosystem (animation, UI libs, monorepo, real project)
- **After 29**: Additional topics auto-defined per session (Zustand deep dive, React Native with Expo, React Query patterns, etc.)
