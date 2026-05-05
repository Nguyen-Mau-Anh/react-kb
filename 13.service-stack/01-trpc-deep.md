# Service Stack — 01. tRPC 11 Deep Dive: Middleware, Context, Batching, Links, Subscriptions, Transformers

> **What / Why / How** — `@trpc/server@11` is the most-used internal-RPC layer for React + Next.js in 2026. **Routers + procedures** scale to dozens of features; **middleware** carries cross-cutting concerns; **links** customize the wire; **batching + transformers** make it production-grade. `07.nextjs/06-api-routes.md` introduced tRPC; this file goes deep.

---

## 1. Quick Recap — What tRPC Already Solves

tRPC v11 (`@trpc/server@11`, `@trpc/client@11`, `@trpc/react-query@11`) gives you:

- **End-to-end type safety** with no codegen — change a server procedure, the client call site is a TS error.
- **`useQuery` / `useMutation`** wrappers built on TanStack Query 5 (covered in `04.state-and-data/03-data-fetching.md`).
- **Server Component direct calls** via `appRouter.createCaller(ctx)` — skip HTTP entirely (covered in `07.nextjs/06-api-routes.md`).
- **Zod validation** on every input (covered in `04.state-and-data/01-forms.md` and `05.routing-and-styling/03-typescript-with-react.md`).

This file covers the parts most teams skip: **middleware composition, layered context, batching, custom links (logging / retry / WebSocket / batched HTTP), Server-Sent Events subscriptions, and the SuperJSON transformer pattern** that handles `Date`, `BigInt`, `Map`, `Set`, `URL`.

---

## 2. The 2026 tRPC Project Layout

The shape that scales to ~30 procedures and beyond:

```
src/
├── server/
│   ├── trpc.ts                       ← initTRPC, procedure factories
│   ├── context.ts                    ← createContext function
│   ├── middlewares/
│   │   ├── auth.ts                   ← protectedProcedure
│   │   ├── logger.ts                 ← request logging
│   │   ├── ratelimit.ts              ← per-user rate limiting
│   │   └── audit.ts                  ← audit log writer
│   └── routers/
│       ├── _app.ts                   ← appRouter
│       ├── user.ts
│       ├── project.ts
│       ├── task.ts
│       └── billing.ts
├── trpc/
│   └── client.ts                      ← createTRPCReact<AppRouter>()
└── app/
    ├── api/trpc/[trpc]/route.ts       ← Next.js Route Handler
    └── providers.tsx                   ← QueryClientProvider + tRPC client
```

The `_app.ts` naming convention keeps the root router obvious. Domain routers (`user.ts`, `project.ts`) own one slice of the API each.

---

## 3. Context — The Per-Request Foundation

Every procedure gets a **context** value: the per-request bag of state that procedures use (auth session, DB client, IP, headers, request ID).

```ts
// server/context.ts
import { auth } from '@/auth';
import { db } from '@/server/db';
import { logger } from '@/lib/logger';
import type { FetchCreateContextFnOptions } from '@trpc/server/adapters/fetch';

export async function createContext(opts: FetchCreateContextFnOptions) {
  const session = await auth();                    // covered in 07.nextjs/04-auth.md
  const requestId = opts.req.headers.get('x-request-id') ?? crypto.randomUUID();
  const ip = opts.req.headers.get('x-forwarded-for')?.split(',')[0]?.trim() ?? 'unknown';

  return {
    session,
    user: session?.user ?? null,
    db,
    requestId,
    ip,
    log: logger.child({ requestId, userId: session?.user?.id }),
  };
}

export type Context = Awaited<ReturnType<typeof createContext>>;
```

### What goes in context

| Belongs in context | Doesn't belong |
|--------------------|----------------|
| Session / user (read once per request) | Long-lived singletons (use module imports) |
| DB client (already a singleton, but conventional) | Per-procedure side effects |
| Request ID, IP, locale, user-agent | Computed values that depend on inputs |
| Pre-fetched data shared across many procedures (rare) | Anything > a few KB |

**Don't put expensive lookups in context.** Context runs for every procedure in a batched call (see §6); a slow context creation slows every request. Defer expensive work to the procedure that needs it.

---

## 4. The `setup()` + Procedure Factories Pattern

```ts
// server/trpc.ts
import { initTRPC, TRPCError } from '@trpc/server';
import superjson from 'superjson';
import { ZodError } from 'zod';
import type { Context } from './context';

const t = initTRPC.context<Context>().create({
  transformer: superjson,                          // covered in §10
  errorFormatter({ shape, error }) {
    return {
      ...shape,
      data: {
        ...shape.data,
        zodError: error.cause instanceof ZodError ? error.cause.flatten() : null,
      },
    };
  },
});

export const router = t.router;
export const middleware = t.middleware;
export const publicProcedure = t.procedure;

// Auth-required procedure — covered in detail next
export const protectedProcedure = t.procedure.use(({ ctx, next }) => {
  if (!ctx.user) throw new TRPCError({ code: 'UNAUTHORIZED' });
  return next({ ctx: { ...ctx, user: ctx.user } });    // narrows user to non-null
});
```

The **factory pattern** (`publicProcedure`, `protectedProcedure`, future `adminProcedure`) keeps boilerplate out of every procedure definition. New procedures pick the factory; the right middleware chain is included automatically.

---

## 5. Middleware — Where Real Logic Lives

Middleware in tRPC v11 is **composable**, **typed**, and **per-procedure-chain**. Each middleware can:

- Inspect or modify the context.
- Run before/after the procedure (`next()` returns the result).
- Throw `TRPCError` to reject the call.
- Time the call, log, audit, rate-limit, transform.

### Auth middleware — narrows context types

```ts
// server/middlewares/auth.ts
import { TRPCError } from '@trpc/server';
import { middleware } from '../trpc';

export const isAuthed = middleware(({ ctx, next }) => {
  if (!ctx.user) throw new TRPCError({ code: 'UNAUTHORIZED' });
  return next({
    ctx: {
      ...ctx,
      user: ctx.user,                              // TypeScript narrows non-null
      session: ctx.session!,
    },
  });
});

export const isAdmin = middleware(({ ctx, next }) => {
  if (!ctx.user) throw new TRPCError({ code: 'UNAUTHORIZED' });
  if (ctx.user.role !== 'admin') throw new TRPCError({ code: 'FORBIDDEN' });
  return next({ ctx: { ...ctx, user: ctx.user } });
});
```

### Logger middleware — wraps the procedure

```ts
// server/middlewares/logger.ts
import { middleware } from '../trpc';

export const logProcedure = middleware(async ({ path, type, ctx, next }) => {
  const start = Date.now();
  const result = await next();
  const durationMs = Date.now() - start;

  ctx.log.info({
    procedure: path,
    type,
    ok: result.ok,
    durationMs,
    userId: ctx.user?.id,
  }, 'trpc.call');

  return result;
});
```

### Rate-limit middleware — per-user

```ts
// server/middlewares/ratelimit.ts
import { Ratelimit } from '@upstash/ratelimit';
import { Redis } from '@upstash/redis';
import { TRPCError } from '@trpc/server';
import { middleware } from '../trpc';

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(60, '1 m'),     // 60 calls/min/user
});

export const enforceRateLimit = middleware(async ({ ctx, next }) => {
  const key = ctx.user?.id ?? `ip:${ctx.ip}`;
  const { success, remaining } = await ratelimit.limit(key);
  if (!success) throw new TRPCError({ code: 'TOO_MANY_REQUESTS' });
  return next();
});
```

### Audit middleware — for mutations only

```ts
// server/middlewares/audit.ts
import { middleware } from '../trpc';

export const auditMutation = middleware(async ({ path, type, input, ctx, next }) => {
  const result = await next();
  if (type === 'mutation' && result.ok) {
    await ctx.db.auditLog.create({
      data: {
        userId: ctx.user?.id,
        action: path,
        input: input as object,                    // SuperJSON-serialized
        ip: ctx.ip,
        requestId: ctx.requestId,
      },
    });
  }
  return result;
});
```

### Compose middlewares with `.use()`

```ts
// server/trpc.ts (continued)
import { isAuthed, isAdmin } from './middlewares/auth';
import { logProcedure } from './middlewares/logger';
import { enforceRateLimit } from './middlewares/ratelimit';
import { auditMutation } from './middlewares/audit';

const baseProcedure = t.procedure.use(logProcedure);

export const publicProcedure  = baseProcedure;
export const rateLimitedPublic = baseProcedure.use(enforceRateLimit);

export const protectedProcedure = baseProcedure.use(enforceRateLimit).use(isAuthed);
export const adminProcedure     = protectedProcedure.use(isAdmin);
export const auditedMutation    = protectedProcedure.use(auditMutation);
```

Every procedure picks its factory based on the cross-cutting concerns it needs:

```ts
// server/routers/billing.ts
import { z } from 'zod';
import { router, adminProcedure, auditedMutation, protectedProcedure } from '../trpc';

export const billingRouter = router({
  myInvoices: protectedProcedure.query(({ ctx }) =>
    ctx.db.invoice.findMany({ where: { userId: ctx.user.id } })
  ),

  refund: auditedMutation
    .input(z.object({ invoiceId: z.string().cuid(), reason: z.string().min(1) }))
    .mutation(async ({ ctx, input }) => {
      return ctx.db.invoice.update({ where: { id: input.invoiceId }, data: { refunded: true } });
    }),

  listAllInvoices: adminProcedure.query(({ ctx }) =>
    ctx.db.invoice.findMany({ take: 100 })
  ),
});
```

The middleware chain runs **bottom-up on input, top-down on result**: log → rate-limit → auth → audit → procedure body → audit (after) → auth (no-op) → rate-limit (no-op) → log (with timing). You don't write any of that orchestration; tRPC composes it.

---

## 6. Batching — Multiple Calls In One HTTP Request

By default, **tRPC batches client calls within a 0–5ms window into a single HTTP request**. This is the most-underrated tRPC feature.

A page that renders 10 components, each calling `trpc.x.useQuery(...)`, fires **one** request to `/api/trpc/x.a,x.b,x.c,...` rather than 10. The server runs each procedure, returns an array of results.

### Setup — `httpBatchLink`

```ts
// trpc/client.ts (covered briefly in 07.nextjs/06-api-routes.md)
import { createTRPCReact } from '@trpc/react-query';
import type { AppRouter } from '@/server/routers/_app';
export const trpc = createTRPCReact<AppRouter>();
```

```tsx
// app/providers.tsx
'use client';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { httpBatchLink, loggerLink } from '@trpc/client';
import { trpc } from '@/trpc/client';
import superjson from 'superjson';
import { useState } from 'react';

export function Providers({ children }: { children: React.ReactNode }) {
  const [qc] = useState(() => new QueryClient());
  const [client] = useState(() =>
    trpc.createClient({
      links: [
        loggerLink({
          enabled: (op) => process.env.NODE_ENV === 'development' || (op.direction === 'down' && op.result instanceof Error),
        }),
        httpBatchLink({
          url: '/api/trpc',
          transformer: superjson,
          maxURLLength: 2083,                       // safe for IE; modern browsers handle more
        }),
      ],
    })
  );

  return (
    <trpc.Provider client={client} queryClient={qc}>
      <QueryClientProvider client={qc}>{children}</QueryClientProvider>
    </trpc.Provider>
  );
}
```

### Real impact

Before batching: page render = 10 round trips × 100ms latency = 1000ms total wait.
After batching: 1 round trip × 100ms = 100ms total.

For pages with many small data dependencies (analytics widgets, user info, settings, notifications), this is a **10× speedup** with zero code changes.

### When NOT to batch

- **Real-time / streaming data** — use a separate `httpLink` or `wsLink` for those procedures.
- **Long-running mutations** — batching can delay the response of fast ones waiting on slow ones in the same batch.
- **Different auth requirements** — public vs authenticated procedures shouldn't share a batch.

To disable per-procedure: use a `splitLink` (next section).

---

## 7. Links — Customizing the Wire

A **link** is a middleware for the client. Links transform requests/responses, add retries, switch transports, log, mock — they compose like server middleware.

### `loggerLink` — built-in dev logger

```ts
import { loggerLink } from '@trpc/client';

loggerLink({
  enabled: (op) =>
    process.env.NODE_ENV === 'development' ||
    (op.direction === 'down' && op.result instanceof Error),
});
```

Logs every call to the browser console with timing and result. Disable in prod by checking `NODE_ENV`. **The single best dev-tool for debugging tRPC issues.**

### `splitLink` — route to different transports

```ts
import { splitLink, httpBatchLink, httpLink, wsLink } from '@trpc/client';

const links = [
  splitLink({
    condition: (op) => op.type === 'subscription',
    true: wsLink({ url: 'wss://api.example.com/trpc' }),         // subscriptions over WebSocket
    false: splitLink({
      condition: (op) => Boolean(op.context.skipBatch),
      true: httpLink({ url: '/api/trpc' }),                       // un-batched
      false: httpBatchLink({ url: '/api/trpc' }),                 // batched (default)
    }),
  }),
];
```

Now subscriptions go over WS, mutations marked `skipBatch` go un-batched, everything else batches. **Per-call control without changing call sites:**

```ts
trpc.task.heavyExport.useMutation(undefined, {
  trpc: { context: { skipBatch: true } },           // op.context.skipBatch === true
});
```

### Custom retry link

```ts
import { TRPCLink } from '@trpc/client';
import { observable } from '@trpc/server/observable';

export function retryLink({ maxRetries = 3, retryDelay = 500 }): TRPCLink<any> {
  return () => ({ next, op }) =>
    observable((observer) => {
      let attempts = 0;
      const exec = () => {
        const sub = next(op).subscribe({
          next: (v) => observer.next(v),
          error: (err) => {
            attempts++;
            const isRetryable = err.data?.httpStatus >= 500 && err.data?.httpStatus < 600;
            if (isRetryable && attempts < maxRetries) {
              setTimeout(exec, retryDelay * 2 ** (attempts - 1));   // exponential backoff
            } else {
              observer.error(err);
            }
          },
          complete: () => observer.complete(),
        });
        return () => sub.unsubscribe();
      };
      exec();
    });
}
```

Most teams don't need this — TanStack Query's built-in `retry` config (covered in `04.state-and-data/03-data-fetching.md`) handles query retries. Use a custom retry link only when you need it for **all** procedures including ones not driven by TanStack Query.

### Custom auth-header link

```ts
import { TRPCLink } from '@trpc/client';
import { observable } from '@trpc/server/observable';

export function authHeaderLink(getToken: () => string | null): TRPCLink<any> {
  return () => ({ next, op }) => {
    const token = getToken();
    const newOp = token
      ? { ...op, context: { ...op.context, headers: { ...op.context.headers, authorization: `Bearer ${token}` } } }
      : op;
    return observable((observer) => {
      const sub = next(newOp).subscribe(observer);
      return () => sub.unsubscribe();
    });
  };
}
```

Almost never needed inside Next.js (cookies handle auth). Useful if your tRPC client lives in a React Native app talking to a separate Next.js backend (covered in `09.beyond-web/01-react-native-expo.md`).

---

## 8. Subscriptions — Real-Time Over Server-Sent Events

tRPC v11 added **first-class Server-Sent Events (SSE) subscriptions** as the default real-time transport. WebSockets are still supported via `wsLink`, but SSE is simpler, works through HTTP/2 multiplexing, and doesn't require a separate server process.

### Server — define a subscription

```ts
// server/routers/notifications.ts
import { router, protectedProcedure } from '../trpc';
import { observable } from '@trpc/server/observable';
import { eventEmitter } from '@/lib/events';

export const notificationsRouter = router({
  onNew: protectedProcedure.subscription(({ ctx }) => {
    return observable<{ id: string; message: string }>((emit) => {
      const handler = (notification: { userId: string; id: string; message: string }) => {
        if (notification.userId === ctx.user.id) {
          emit.next({ id: notification.id, message: notification.message });
        }
      };
      eventEmitter.on('notification', handler);
      return () => eventEmitter.off('notification', handler);
    });
  }),
});
```

`eventEmitter` is your in-process event bus, OR a Redis pub/sub adapter if multiple Node processes share state.

### Client — `useSubscription`

```tsx
'use client';
import { trpc } from '@/trpc/client';
import { useState } from 'react';

export function NotificationToaster() {
  const [items, setItems] = useState<{ id: string; message: string }[]>([]);

  trpc.notifications.onNew.useSubscription(undefined, {
    onData: (n) => setItems((prev) => [...prev, n]),
  });

  return (
    <ul role="status" aria-live="polite">
      {items.map((n) => <li key={n.id}>{n.message}</li>)}
    </ul>
  );
}
```

That's it. Server pushes; client renders.

### When SSE is enough vs when you need WebSockets

| Need | Pick |
|------|------|
| Server → client only (notifications, log streaming, AI token streaming) | SSE — tRPC default |
| Client → server *and* server → client | WebSocket — `wsLink` |
| Operates through corporate proxies that block WS | SSE wins (it's just HTTP) |
| Bi-directional CRDT collaboration (covered in `09.beyond-web/03-realtime.md`) | Use **PartyKit + Yjs** instead — tRPC isn't the right tool |

For most "live-update" needs in a Next.js app: **tRPC SSE subscriptions**. Reach for PartyKit / Liveblocks only when you need true CRDT collab.

### Vercel + SSE — works on Edge functions

The Vercel Edge runtime supports streaming responses. tRPC v11's SSE adapter (via `httpBatchStreamLink` on the client) plays nicely:

```ts
import { httpBatchStreamLink } from '@trpc/client';

httpBatchStreamLink({
  url: '/api/trpc',
  transformer: superjson,
});
```

Streams responses as JSON-NL chunks. Combined with React 19 Server Components + Suspense (covered in `09.beyond-web/02-react-19-features.md`), this gives **per-procedure streaming** — slow procedures don't block fast ones in the same batch.

---

## 9. SuperJSON — Why You Need a Transformer

Plain JSON loses fidelity:

```ts
JSON.stringify(new Date('2026-01-01'));     // '"2026-01-01T00:00:00.000Z"'
JSON.parse(JSON.stringify(new Date()));      // string, not Date
JSON.stringify(new Map());                    // '{}'
JSON.stringify(BigInt(10));                   // TypeError
```

The fix: a **transformer** that round-trips these types.

```bash
npm i superjson@2
```

```ts
// On the server
import superjson from 'superjson';
import { initTRPC } from '@trpc/server';

const t = initTRPC.context<Context>().create({ transformer: superjson });
```

```ts
// On the client (must match)
import { httpBatchLink } from '@trpc/client';

httpBatchLink({ url: '/api/trpc', transformer: superjson });
```

### What SuperJSON 2 handles automatically

| Type | Behavior |
|------|----------|
| `Date` | Round-trips as `Date` |
| `Map` / `Set` | Round-trips with full content |
| `BigInt` | Round-trips |
| `Buffer` (Node) | Round-trips |
| `URL` | Round-trips |
| `RegExp` | Round-trips |
| `Symbol` | Discarded (can't be serialized — by design) |
| `Error` instances | Become `{ name, message, stack }` (lossy) |
| `undefined` properties | Preserved (plain JSON drops them) |

Without SuperJSON, you spend a few hours each year debugging "why is `task.dueDate` a string here". With it, you don't.

### Alternative: `devalue@5`

`devalue@5` is a faster, smaller alternative to SuperJSON. Used internally by SvelteKit and some tRPC adopters who care about bundle size. Trade-off: less common, slightly different supported types.

For 95% of teams: **stick with SuperJSON 2**.

### Don't mix transformers across server and client

If you set `transformer: superjson` on the server but forget on the client, you get garbled `{ json: ..., meta: ... }` objects on the wire. **Test once locally**: log a `Date` field; if it arrives as a string, you have a transformer mismatch.

---

## 10. Server-Side tRPC in Server Components

Recap from `07.nextjs/06-api-routes.md`: skip HTTP, call procedures directly from Server Components.

```tsx
// app/dashboard/page.tsx (Server Component)
import { auth } from '@/auth';
import { appRouter } from '@/server/routers/_app';
import { db } from '@/server/db';
import { headers } from 'next/headers';

export default async function DashboardPage() {
  const session = await auth();
  const requestId = (await headers()).get('x-request-id') ?? crypto.randomUUID();

  const caller = appRouter.createCaller({
    session,
    user: session?.user ?? null,
    db,
    requestId,
    ip: 'server',
    log: logger.child({ requestId }),
  });

  const projects = await caller.project.list();
  return <ProjectList projects={projects} />;
}
```

`createCaller` runs every middleware in the same chain — auth, logging, audit — but skips network entirely. **Same code paths as the API; same authorization checks; same telemetry.** This is the killer pattern for Next.js App Router + tRPC.

For convenience, wrap it:

```ts
// server/server-caller.ts
import 'server-only';                              // Next.js: must not import from client
import { auth } from '@/auth';
import { db } from '@/server/db';
import { headers } from 'next/headers';
import { appRouter } from './routers/_app';
import { logger } from '@/lib/logger';

export async function getServerCaller() {
  const session = await auth();
  const requestId = (await headers()).get('x-request-id') ?? crypto.randomUUID();
  return appRouter.createCaller({
    session,
    user: session?.user ?? null,
    db,
    requestId,
    ip: 'server',
    log: logger.child({ requestId, userId: session?.user?.id }),
  });
}
```

```tsx
import { getServerCaller } from '@/server/server-caller';

export default async function Page() {
  const trpc = await getServerCaller();
  const me = await trpc.user.me();
  return <Profile user={me} />;
}
```

`'server-only'` (covered in `07.nextjs/03-data-fetching.md`) errors at build time if a Client Component imports this file — preventing accidental bundling of server code into the client.

---

## 11. Hydration — Pre-Fetching for Client Components

Pattern from `04.state-and-data/03-data-fetching.md`: server seeds the TanStack Query cache, client picks it up without re-fetching.

```tsx
// app/projects/page.tsx (Server Component)
import { dehydrate, HydrationBoundary, QueryClient } from '@tanstack/react-query';
import { getServerCaller } from '@/server/server-caller';
import { ProjectListClient } from './ProjectListClient';

export default async function ProjectsPage() {
  const qc = new QueryClient();
  const trpc = await getServerCaller();
  const projects = await trpc.project.list();

  // Seed the client cache with the query key the client component will use
  qc.setQueryData(['project', 'list'], projects);

  return (
    <HydrationBoundary state={dehydrate(qc)}>
      <ProjectListClient />
    </HydrationBoundary>
  );
}
```

```tsx
// app/projects/ProjectListClient.tsx
'use client';
import { trpc } from '@/trpc/client';

export function ProjectListClient() {
  // initialData isn't needed — HydrationBoundary already seeded the cache
  const { data } = trpc.project.list.useQuery();
  return <ul>{data?.map((p) => <li key={p.id}>{p.name}</li>)}</ul>;
}
```

For tRPC's typed query keys, use `getQueryKey`:

```ts
import { getQueryKey } from '@trpc/react-query';
const key = getQueryKey(trpc.project.list);     // ['project.list']
qc.setQueryData(key, projects);
```

### tRPC's experimental Server Actions integration

`@trpc/next@11` ships an experimental `experimental_nextAppDirCaller` that lets you expose tRPC procedures as Next.js Server Actions:

```ts
import { experimental_nextAppDirCaller } from '@trpc/server/adapters/next-app-dir';

export const appRouter = router({
  project: router({
    create: protectedProcedure
      .input(CreateProjectInput)
      .mutation(async ({ ctx, input }) => { ... })
      .experimental_caller(experimental_nextAppDirCaller),
  }),
});

// Use directly as a Server Action
<form action={appRouter.project.create}>
```

Status: stable for production-curious adoption; API may shift. For 2026 production code: **use tRPC HTTP for client calls, Server Actions for server-driven mutations** (covered in `07.nextjs/03-data-fetching.md`). Both can call the same internal logic — extract shared business code into a `services/` module.

---

## 12. Real-World Procedure Patterns

### Optimistic mutations

```tsx
'use client';
import { trpc } from '@/trpc/client';

function ToggleTask({ task }: { task: Task }) {
  const utils = trpc.useUtils();
  const toggle = trpc.task.toggle.useMutation({
    onMutate: async ({ id }) => {
      await utils.project.byId.cancel({ id: task.projectId });
      const prev = utils.project.byId.getData({ id: task.projectId });
      utils.project.byId.setData({ id: task.projectId }, (old) => ({
        ...old!,
        tasks: old!.tasks.map((t) => t.id === id ? { ...t, done: !t.done } : t),
      }));
      return { prev };
    },
    onError: (_e, _v, ctx) => {
      if (ctx?.prev) utils.project.byId.setData({ id: task.projectId }, ctx.prev);
    },
    onSettled: () => utils.project.byId.invalidate({ id: task.projectId }),
  });

  return <input type="checkbox" checked={task.done} onChange={() => toggle.mutate({ id: task.id })} />;
}
```

`utils.X.invalidate()` and `utils.X.setData()` are the typed TanStack Query helpers tRPC generates for every procedure. Used heavily in `08.ecosystem/04-real-project.md`.

### Infinite queries — cursor pagination

```ts
// server/routers/post.ts
import { z } from 'zod';
import { router, publicProcedure } from '../trpc';

export const postRouter = router({
  feed: publicProcedure
    .input(z.object({
      limit: z.number().min(1).max(100).default(20),
      cursor: z.string().nullish(),
    }))
    .query(async ({ ctx, input }) => {
      const items = await ctx.db.post.findMany({
        take: input.limit + 1,                    // +1 to detect next page
        cursor: input.cursor ? { id: input.cursor } : undefined,
        orderBy: { createdAt: 'desc' },
      });
      const nextCursor = items.length > input.limit ? items.pop()!.id : null;
      return { items, nextCursor };
    }),
});
```

```tsx
'use client';
import { trpc } from '@/trpc/client';

function Feed() {
  const query = trpc.post.feed.useInfiniteQuery(
    { limit: 20 },
    { getNextPageParam: (last) => last.nextCursor },
  );

  return (
    <>
      {query.data?.pages.flatMap((p) => p.items).map((post) => (
        <PostCard key={post.id} post={post} />
      ))}
      <button onClick={() => query.fetchNextPage()} disabled={!query.hasNextPage || query.isFetchingNextPage}>
        Load more
      </button>
    </>
  );
}
```

Same pattern as `04.state-and-data/03-data-fetching.md`; tRPC just types it for free.

### Long-running procedures — return early, finish in background

```ts
// server/routers/job.ts
import { z } from 'zod';
import { router, protectedProcedure } from '../trpc';
import { inngest } from '@/inngest/client';

export const jobRouter = router({
  startExport: protectedProcedure
    .input(z.object({ format: z.enum(['csv', 'xlsx']) }))
    .mutation(async ({ ctx, input }) => {
      const job = await ctx.db.exportJob.create({
        data: { userId: ctx.user.id, format: input.format, status: 'pending' },
      });

      await inngest.send({
        name: 'export.start',
        data: { jobId: job.id, format: input.format, userId: ctx.user.id },
      });

      return { jobId: job.id };
    }),

  getJob: protectedProcedure
    .input(z.object({ jobId: z.string().cuid() }))
    .query(({ ctx, input }) =>
      ctx.db.exportJob.findFirst({ where: { id: input.jobId, userId: ctx.user.id } })
    ),
});
```

Client polls `getJob` (TanStack Query `refetchInterval: 2000` while pending), or subscribes via SSE. Background-job platforms covered in `13.service-stack/03-background-jobs.md` (next topic in the queue).

### File uploads via tRPC

**Don't.** tRPC sends JSON; binary uploads through tRPC defeat its purpose. Use the **direct-to-storage with signed URLs** pattern from `11.capabilities/02-file-uploads.md`:

```ts
// server/routers/upload.ts
export const uploadRouter = router({
  getPresignedUrl: protectedProcedure
    .input(z.object({ filename: z.string(), contentType: z.string(), size: z.number().max(10_000_000) }))
    .mutation(async ({ ctx, input }) => {
      const key = `uploads/${ctx.user.id}/${randomUUID()}-${input.filename}`;
      const url = await getSignedUrl(s3, new PutObjectCommand({ Bucket, Key: key, ContentType: input.contentType }), { expiresIn: 60 });
      return { uploadUrl: url, key };
    }),

  finalize: protectedProcedure
    .input(z.object({ key: z.string() }))
    .mutation(async ({ ctx, input }) => { ... }),
});
```

tRPC issues the URL; the browser PUTs the bytes directly to S3/R2/Vercel Blob.

---

## 13. Error Handling — Beyond `try/catch`

tRPC errors are typed as `TRPCError` on the server, deserialize to `TRPCClientError` on the client.

### Server: throw with the right code

```ts
import { TRPCError } from '@trpc/server';

throw new TRPCError({
  code: 'NOT_FOUND',                              // maps to HTTP 404
  message: 'Project not found',
  cause: originalError,                            // optional — preserves stack trace in dev
});
```

The full code list (and HTTP mappings):

| Code | HTTP | When |
|------|------|------|
| `BAD_REQUEST` | 400 | Validation failed (Zod throws this automatically) |
| `UNAUTHORIZED` | 401 | No session |
| `FORBIDDEN` | 403 | Authenticated but not allowed |
| `NOT_FOUND` | 404 | Resource doesn't exist |
| `TIMEOUT` | 408 | Slow upstream |
| `CONFLICT` | 409 | Duplicate / version mismatch |
| `PRECONDITION_FAILED` | 412 | Stale ETag |
| `PAYLOAD_TOO_LARGE` | 413 | Input too big |
| `UNPROCESSABLE_CONTENT` | 422 | Semantic validation failed |
| `TOO_MANY_REQUESTS` | 429 | Rate limited |
| `CLIENT_CLOSED_REQUEST` | 499 | Client cancelled (rare to throw manually) |
| `INTERNAL_SERVER_ERROR` | 500 | Default for uncaught exceptions |

### Client: typed error handling

```tsx
const create = trpc.project.create.useMutation({
  onError: (err) => {
    if (err.data?.code === 'CONFLICT') {
      toast.error('A project with that name already exists');
    } else if (err.data?.zodError) {
      // Field-level validation errors from errorFormatter
      const fieldErrors = err.data.zodError.fieldErrors;
      Object.entries(fieldErrors).forEach(([field, errors]) => {
        setError(field as any, { message: errors?.[0] });
      });
    } else {
      toast.error('Something went wrong');
    }
  },
});
```

`err.data?.zodError` flows from the `errorFormatter` in §4. **Front-end form errors map directly from server-side Zod schemas — no duplication.**

### Sentry integration

In your error formatter, log unexpected errors to Sentry:

```ts
import * as Sentry from '@sentry/nextjs';

errorFormatter({ shape, error, ctx, path, type, input }) {
  if (shape.data?.httpStatus >= 500) {
    Sentry.captureException(error, {
      tags: { trpc_path: path, trpc_type: type },
      user: ctx?.user ? { id: ctx.user.id } : undefined,
      extra: { input },
    });
  }
  return { ...shape, data: { ...shape.data, zodError: ... } };
}
```

Combined with `10.production/04-observability.md`, this gives full observability per procedure.

---

## 14. Testing tRPC Procedures

### Server-side: createCaller with mock context

```ts
// __tests__/project.test.ts (Vitest — covered in 06.testing-perf/01-testing.md)
import { describe, it, expect, beforeEach, vi } from 'vitest';
import { appRouter } from '@/server/routers/_app';
import { dbMock, createMockContext } from '../helpers/mocks';

describe('project.create', () => {
  beforeEach(() => vi.clearAllMocks());

  it('rejects unauthenticated callers', async () => {
    const caller = appRouter.createCaller(createMockContext({ user: null }));
    await expect(caller.project.create({ name: 'x', description: '' }))
      .rejects.toMatchObject({ code: 'UNAUTHORIZED' });
  });

  it('creates with proper fields', async () => {
    const ctx = createMockContext({ user: { id: 'u1', email: 'a@b.c', role: 'user' } });
    dbMock.project.create.mockResolvedValue({ id: 'p1', name: 'My Project', ownerId: 'u1' });
    const caller = appRouter.createCaller(ctx);

    const result = await caller.project.create({ name: 'My Project', description: '' });

    expect(result.id).toBe('p1');
    expect(dbMock.project.create).toHaveBeenCalledWith({
      data: expect.objectContaining({ ownerId: 'u1', name: 'My Project' }),
    });
  });
});
```

### Client-side: MSW handlers

For component tests of tRPC-using components, mock at the network layer with MSW:

```ts
// __tests__/handlers.ts
import { http, HttpResponse } from 'msw';

export const handlers = [
  http.post('/api/trpc/project.list', () => HttpResponse.json([{ result: { data: { json: [{ id: 'p1', name: 'Test' }] } } }])),
];
```

Or use the official `msw-trpc@2` adapter for typed mocks:

```bash
npm i -D msw-trpc@2
```

```ts
import { httpLink } from '@trpc/client';
import { createTRPCMsw } from 'msw-trpc';
import type { AppRouter } from '@/server/routers/_app';

export const trpcMsw = createTRPCMsw<AppRouter>({ links: [httpLink({ url: '/api/trpc' })], transformer: { input: superjson, output: superjson } });

// In tests:
server.use(
  trpcMsw.project.list.query(() => [{ id: 'p1', name: 'Test' }]),
);
```

`msw-trpc` resolves the procedure path types automatically — no string paths. Renames in the router show up as test failures.

---

## 15. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| `superjson` on one side only → garbled `{ json, meta }` payloads | Configure transformer on both server and client identically |
| `apiVersion` doesn't exist; you confused tRPC with Stripe | Pin tRPC versions in package.json instead |
| Subscriptions don't fire | Verify `splitLink` routes them to `wsLink` or that SSE is supported |
| Batched calls failing with "URL too long" | Increase `maxURLLength` or split via `splitLink` |
| Calling tRPC procedure directly bypasses middleware | Always use the procedure factory (`protectedProcedure.use(audit)`); never re-build chains ad-hoc |
| Server Component import accidentally bundles into client | Add `'server-only'` to server-side files |
| Client receives stale data after mutation | Call `utils.X.invalidate()` in `onSettled` |
| File uploads through tRPC are slow / fail | Don't. Use signed URLs (covered in `11.capabilities/02-file-uploads.md`) |
| `errorFormatter` returns `undefined` Zod data on validation success | Check `error.cause instanceof ZodError` before flattening |
| Background-job mutation timeouts at Vercel's 10s edge limit | Return early from the mutation; enqueue via Inngest/Trigger.dev |
| Missing types after a server-side change | Restart TypeScript server (VS Code: Cmd+Shift+P → "Restart TS Server") |
| Accidentally exposing tRPC to public API consumers | Use Route Handlers (covered in `07.nextjs/06-api-routes.md`) for stable public contracts |
| `createCaller` fails because context creator depends on `headers()` outside a request | Build a separate context factory for jobs/cron that doesn't read request headers |
| Subscriptions over Vercel functions disconnect after 5min | Use Edge runtime + streaming; or move to a long-lived host (Railway/Fly — covered in `07.nextjs/05-deployment.md`) |
| Optimistic update fires twice for the same mutation | Pass a `mutationKey` to dedupe; or guard with `useMutation`'s `onMutate` snapshot |

---

## 16. Decision Tree

```
What kind of tRPC question?
│
├── How to share auth + session? → Custom context (§3) + protectedProcedure middleware
├── How to add cross-cutting concerns? → Middleware factories (§5) — log, ratelimit, audit
├── Slow page with many queries? → Default httpBatchLink already batches; verify with loggerLink
├── Need real-time updates? → Subscriptions over SSE (§8); switch to wsLink only for bi-directional
├── Need typed Date/Map/BigInt? → SuperJSON transformer (§9) on BOTH ends
├── Server Component fetching? → createCaller (§10) — same middleware, no HTTP
├── Pre-fetch into client? → HydrationBoundary + setQueryData (§11)
├── Optimistic mutation? → utils.X.cancel + setData + invalidate in onMutate/onError/onSettled (§12)
├── Long-running mutation? → Return early; queue via Inngest (next topic)
├── File upload through tRPC? → Don't; use signed URLs from 11.capabilities/02
└── Public API for mobile / partners? → Route Handlers (07.nextjs/06), not tRPC

Performance?
│
├── HTTP batching → enable httpBatchLink (default)
├── HTTP/2 streaming → httpBatchStreamLink for per-procedure streaming
├── Sub-50ms latency on Edge → put route handler on edge runtime
└── 100K+ users hitting same endpoint → cache via Redis in middleware

Auth model?
│
├── Cookie-based session → use auth() in context (Next.js)
├── JWT in header → custom authHeaderLink + verify in middleware
└── API keys for external clients → don't use tRPC; use Route Handlers
```

---

## 17. What This Topic Connects To

- **`07.nextjs/06-api-routes.md`** — the introductory tRPC topic; this file goes deep.
- **`04.state-and-data/03-data-fetching.md`** — TanStack Query integration patterns.
- **`07.nextjs/04-auth.md`** — Auth.js v5 session powering the tRPC context.
- **`11.capabilities/02-file-uploads.md`** — why tRPC isn't for file uploads.
- **`10.production/04-observability.md`** — Sentry integration via errorFormatter; `pino` logger in middleware.
- **`12.quality-at-scale/01-feature-flags.md`** — flag values can flow through context.
- **`08.ecosystem/04-real-project.md`** — full Task Manager project using these patterns end-to-end.

---

## 18. Summary

| Concept | Pattern |
|---------|---------|
| Per-request state | Context with `session`, `db`, `requestId`, `log` |
| Auth + cross-cutting concerns | Middleware factories (`protectedProcedure`, `auditedMutation`) |
| Multiple-call optimization | `httpBatchLink` (default) batches within 0–5ms windows |
| Per-call routing | `splitLink` routes to ws / http / un-batched |
| Real-time | SSE-based subscriptions via `useSubscription` |
| `Date`/`Map`/`BigInt` | `superjson@2` transformer on both ends |
| Server Component fetches | `createCaller(ctx)` — no HTTP, full middleware |
| Pre-fetch + hydrate | `HydrationBoundary` + `setQueryData` keyed by `getQueryKey(...)` |
| Optimistic mutations | `utils.X.cancel` → `setData` → `onError` rollback → `onSettled` invalidate |
| Errors | `TRPCError` codes map to HTTP; `errorFormatter` exposes Zod field errors |
| Testing | `createCaller(mockCtx)` server-side; `msw-trpc@2` client-side |

| Rule | Why |
|------|-----|
| Pin SuperJSON on server AND client | One side breaks → garbled wire format |
| Build procedure factories, not ad-hoc chains | Avoids missing middleware on new procedures |
| Use `createCaller` in Server Components | Same auth + audit + log; zero HTTP overhead |
| Route subscriptions through `wsLink` or use SSE | Don't try to subscribe over `httpBatchLink` |
| Don't push file uploads through tRPC | Signed URLs from `11.capabilities/02-file-uploads.md` |
| Long-running mutations return early + queue | Vercel's 10s cap will kill them otherwise |
| Tag Sentry exceptions with `trpc_path` | Diagnose problem procedures instantly |
| Always invalidate after a mutation | TanStack cache stays stale otherwise |

---

## Further reading

- [tRPC v11 docs](https://trpc.io/docs)
- [`@trpc/server` API](https://trpc.io/docs/server)
- [`@trpc/react-query` API](https://trpc.io/docs/client/react)
- [Server-Sent Events subscriptions](https://trpc.io/docs/server/subscriptions)
- [`createCaller` in App Router](https://trpc.io/docs/client/nextjs/server-components)
- [SuperJSON docs](https://github.com/blitz-js/superjson)
- [msw-trpc](https://github.com/marcosrjjunior/msw-trpc)
- [t3-stack docs](https://create.t3.gg/) — opinionated reference setup
- [Cal.com tRPC source](https://github.com/calcom/cal.com/tree/main/packages/trpc) — production reference at scale
