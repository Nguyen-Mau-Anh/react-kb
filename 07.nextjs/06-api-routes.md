# Next.js — 06. API Routes & Route Handlers: route.ts, tRPC 11, Hono 4

> **What / Why / How** — three real ways to expose endpoints from Next.js. Pick by scope: internal mutation, public REST API, or end-to-end-typed RPC.

---

## 1. The Three Tools (and Where Each Fits)

| Tool | Best for | Type safety | Bundle on client |
|------|----------|-------------|------------------|
| **Route Handlers** (`app/api/.../route.ts`) | Public REST APIs, webhooks, file uploads, streaming responses | Manual (validate with Zod) | None — server only |
| **tRPC 11** (`@trpc/server@11`, `@trpc/client@11`, `@trpc/react-query@11`) | Internal API consumed by your own React app | End-to-end typed (server → client without codegen) | ~12 KB gzipped |
| **Hono 4** (`hono@4`) | Reusable HTTP app you might run on Cloudflare Workers, Bun, Deno, AWS Lambda — same code | Strong (typed routes via `hc()` client) | ~3 KB gzipped (if you use the `hc()` client) |

This file covers all three with named scenarios for when each beats the others. **Route Handlers and Server Actions** were introduced in `07.nextjs/03-data-fetching.md` — this is the deeper API-design view.

---

## 2. Route Handlers — The Built-In Default

### What

Files named `route.ts` (or `.js`) inside `app/api/.../` define backend endpoints. Each exported HTTP method becomes the handler for that method.

```ts
// app/api/posts/route.ts
import { NextResponse } from 'next/server';
import { auth } from '@/auth';
import { prisma } from '@/lib/prisma';
import { z } from 'zod';

const CreatePostSchema = z.object({
  title: z.string().min(1).max(200),
  body: z.string().min(1),
});

export async function GET(req: Request) {
  const url = new URL(req.url);
  const cursor = url.searchParams.get('cursor');
  const posts = await prisma.post.findMany({
    take: 20,
    cursor: cursor ? { id: cursor } : undefined,
    orderBy: { createdAt: 'desc' },
  });
  return NextResponse.json({ posts });
}

export async function POST(req: Request) {
  const session = await auth();
  if (!session?.user) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }
  const parsed = CreatePostSchema.safeParse(await req.json());
  if (!parsed.success) {
    return NextResponse.json(
      { error: parsed.error.flatten() },
      { status: 400 }
    );
  }
  const post = await prisma.post.create({
    data: { ...parsed.data, authorId: session.user.id },
  });
  return NextResponse.json(post, { status: 201 });
}
```

### Why pick Route Handlers

- **Public API contract.** Mobile apps, third-party integrators, webhooks all call HTTP, not RPC. They need a stable URL.
- **Streaming responses.** `ReadableStream` is straightforward (covered in §6).
- **Webhook receivers.** Stripe, GitHub, Slack send POSTs; your handler just receives them.
- **No client bundle cost.** The handler is server-only — nothing ships to the browser.

### Why NOT pick Route Handlers (and use Server Actions or tRPC instead)

- For internal mutations from your own React UI, the URL/contract is wasted ceremony. Server Actions are nicer.
- No automatic input/output type sync — you have to keep the client `fetch()` call's TS type aligned with the server schema by hand.

### Dynamic segments and edge runtime

```ts
// app/api/posts/[id]/route.ts
type Ctx = { params: { id: string } };

export async function DELETE(req: Request, { params }: Ctx) {
  await prisma.post.delete({ where: { id: params.id } });
  return new Response(null, { status: 204 });
}

// Run on the Edge runtime instead of Node.js (smaller, faster cold start)
export const runtime = 'edge';

// Force dynamic — never cache
export const dynamic = 'force-dynamic';
```

`runtime: 'edge'` is the right pick for handlers that don't need Node-only libraries (Prisma, native modules). For Prisma + Postgres, stay on `'nodejs'` (the default) or use Prisma Accelerate for Edge.

---

## 3. Real Patterns You'll Actually Write in Route Handlers

### Webhook receiver — Stripe with signature verification

```ts
// app/api/stripe/webhook/route.ts
import { NextResponse } from 'next/server';
import Stripe from 'stripe';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

export async function POST(req: Request) {
  const sig = req.headers.get('stripe-signature');
  const raw = await req.text();
  let event: Stripe.Event;

  try {
    event = stripe.webhooks.constructEvent(raw, sig!, process.env.STRIPE_WEBHOOK_SECRET!);
  } catch {
    return new Response('Invalid signature', { status: 400 });
  }

  switch (event.type) {
    case 'checkout.session.completed':
      await handleCheckout(event.data.object);
      break;
    // ...
  }

  return NextResponse.json({ received: true });
}
```

**Critical gotcha**: read the body as `req.text()`, not `req.json()`. Stripe's signature is computed against the **raw bytes**, and parsing as JSON re-serializes them, breaking verification. This is the #1 reason Stripe webhooks "mysteriously fail" on Vercel.

### File upload — multipart with `formData()`

```ts
// app/api/upload/route.ts
import { NextResponse } from 'next/server';
import { put } from '@vercel/blob';

export async function POST(req: Request) {
  const form = await req.formData();
  const file = form.get('file') as File | null;
  if (!file) return NextResponse.json({ error: 'No file' }, { status: 400 });

  const blob = await put(`uploads/${file.name}`, file, { access: 'public' });
  return NextResponse.json({ url: blob.url });
}
```

For large files, prefer client-direct uploads (signed S3 URLs) — covered in `08.ecosystem/04-real-project.md`.

### Streaming response — Server-Sent Events for live progress

```ts
// app/api/jobs/[id]/stream/route.ts
export const runtime = 'edge';

export async function GET(req: Request, { params }: { params: { id: string } }) {
  const stream = new ReadableStream({
    async start(controller) {
      const enc = new TextEncoder();
      for (const update of subscribeToJob(params.id)) {
        controller.enqueue(enc.encode(`data: ${JSON.stringify(update)}\n\n`));
        if (update.done) break;
      }
      controller.close();
    },
  });

  return new Response(stream, {
    headers: { 'Content-Type': 'text/event-stream' },
  });
}
```

The browser consumes with `new EventSource('/api/jobs/abc/stream')`. Used by ChatGPT-style streaming-token UIs (the `@ai-sdk/react@1` library wraps this).

### LLM streaming with the Vercel AI SDK

```ts
// app/api/chat/route.ts
import { streamText } from 'ai';
import { anthropic } from '@ai-sdk/anthropic';

export async function POST(req: Request) {
  const { messages } = await req.json();
  const result = await streamText({
    model: anthropic('claude-sonnet-4-6'),
    messages,
  });
  return result.toDataStreamResponse();
}
```

Pair with `useChat()` from `@ai-sdk/react@1` on the client — handles streaming, retries, and message state.

---

## 4. tRPC 11 — End-to-End-Typed Internal RPC

### What

`@trpc/server@11` lets you define server procedures. The client gets the **same TypeScript types** automatically — no codegen, no OpenAPI, no `fetch()` and `JSON.parse()`. Used by `create-t3-app@7`, Cal.com, and many production SaaS apps.

```ts
// server/trpc.ts
import { initTRPC, TRPCError } from '@trpc/server';
import superjson from 'superjson';
import { auth } from '@/auth';

const t = initTRPC.context<{ session: Awaited<ReturnType<typeof auth>> }>().create({
  transformer: superjson, // serializes Date, BigInt, Map, etc. correctly
});

export const router = t.router;
export const publicProcedure = t.procedure;

export const protectedProcedure = t.procedure.use(({ ctx, next }) => {
  if (!ctx.session?.user) throw new TRPCError({ code: 'UNAUTHORIZED' });
  return next({ ctx: { ...ctx, user: ctx.session.user } });
});
```

```ts
// server/routers/post.ts
import { z } from 'zod';
import { router, publicProcedure, protectedProcedure } from '../trpc';
import { prisma } from '@/lib/prisma';

export const postRouter = router({
  list: publicProcedure
    .input(z.object({ cursor: z.string().nullish() }))
    .query(async ({ input }) => {
      return prisma.post.findMany({
        take: 20,
        cursor: input.cursor ? { id: input.cursor } : undefined,
        orderBy: { createdAt: 'desc' },
      });
    }),

  create: protectedProcedure
    .input(z.object({ title: z.string().min(1), body: z.string().min(1) }))
    .mutation(async ({ ctx, input }) => {
      return prisma.post.create({
        data: { ...input, authorId: ctx.user.id },
      });
    }),
});
```

```ts
// server/routers/_app.ts
import { router } from '../trpc';
import { postRouter } from './post';

export const appRouter = router({ post: postRouter });
export type AppRouter = typeof appRouter; // exported TYPE only
```

### Mount tRPC inside a Route Handler

```ts
// app/api/trpc/[trpc]/route.ts
import { fetchRequestHandler } from '@trpc/server/adapters/fetch';
import { appRouter } from '@/server/routers/_app';
import { auth } from '@/auth';

const handler = (req: Request) =>
  fetchRequestHandler({
    endpoint: '/api/trpc',
    req,
    router: appRouter,
    createContext: async () => ({ session: await auth() }),
  });

export { handler as GET, handler as POST };
```

### Client setup with TanStack Query

```ts
// trpc/client.ts
import { createTRPCReact } from '@trpc/react-query';
import type { AppRouter } from '@/server/routers/_app';

export const trpc = createTRPCReact<AppRouter>();
```

```tsx
// app/providers.tsx
'use client';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { httpBatchLink } from '@trpc/client';
import { trpc } from '@/trpc/client';
import superjson from 'superjson';
import { useState } from 'react';

export function Providers({ children }: { children: React.ReactNode }) {
  const [qc] = useState(() => new QueryClient());
  const [client] = useState(() =>
    trpc.createClient({
      transformer: superjson,
      links: [httpBatchLink({ url: '/api/trpc' })],
    })
  );
  return (
    <trpc.Provider client={client} queryClient={qc}>
      <QueryClientProvider client={qc}>{children}</QueryClientProvider>
    </trpc.Provider>
  );
}
```

### Use in a Client Component

```tsx
'use client';
import { trpc } from '@/trpc/client';

function Posts() {
  const list = trpc.post.list.useQuery({ cursor: null });
  const create = trpc.post.create.useMutation();

  return (
    <>
      <ul>{list.data?.map(p => <li key={p.id}>{p.title}</li>)}</ul>
      <button onClick={() => create.mutate({ title: 'New', body: '...' })}>
        Create
      </button>
    </>
  );
}
```

You did not write a single fetch call. Renaming a server field updates the client's TypeScript type automatically. Misusing a procedure shape fails at compile time.

### Why pick tRPC over Route Handlers (when both could work)

| | Route Handlers | tRPC 11 |
|--|----------------|---------|
| Type sync between server and client | Manual (Zod schema in two places) | Automatic |
| Public consumers (mobile, partners) | ✅ Standard HTTP | ❌ Requires the tRPC client |
| Renaming a server endpoint | Breaks client until you update fetch URL | TypeScript error in client immediately |
| TanStack Query integration | DIY | First-class via `@trpc/react-query@11` |
| Bundle size impact on client | None | ~12 KB gzipped |

**Real rule of thumb**: tRPC for everything **inside your own app**. Route Handlers for **public APIs and webhooks**.

### tRPC + Server Components — direct server-side calls

In Next.js 14 App Router you can also call tRPC procedures directly in Server Components without going through HTTP:

```tsx
// app/page.tsx
import { appRouter } from '@/server/routers/_app';
import { auth } from '@/auth';

export default async function Page() {
  const caller = appRouter.createCaller({ session: await auth() });
  const posts = await caller.post.list({ cursor: null });
  return <PostList posts={posts} />;
}
```

This skips the network entirely — same procedures, zero HTTP overhead. The pattern is documented as `serverSideHelpers` and `createCaller` in the [tRPC v11 with Next.js App Router guide](https://trpc.io/docs/client/nextjs/server-components).

---

## 5. Hono 4 — When You Want Portable HTTP

### What

`hono@4` is a tiny, fast, web-standards (`Request`/`Response`) HTTP framework. It runs on:
- Cloudflare Workers
- Vercel Edge / Node functions
- Bun
- Deno
- AWS Lambda
- Node.js (`@hono/node-server`)

The single Hono app you write runs on all of them with a one-line adapter.

### Mount Hono inside a Next.js Route Handler

```ts
// app/api/[[...route]]/route.ts
import { Hono } from 'hono';
import { handle } from 'hono/vercel';
import { z } from 'zod';
import { zValidator } from '@hono/zod-validator';

const app = new Hono().basePath('/api');

app.get('/health', (c) => c.json({ ok: true }));

app.post(
  '/posts',
  zValidator('json', z.object({ title: z.string(), body: z.string() })),
  async (c) => {
    const data = c.req.valid('json'); // typed
    const post = await db.post.create({ data });
    return c.json(post, 201);
  }
);

export const GET = handle(app);
export const POST = handle(app);
export const PATCH = handle(app);
export const DELETE = handle(app);
```

The catch-all segment `[[...route]]` lets a single file own `/api/*` without manually creating one folder per endpoint.

### Why Hono over Route Handlers

- **Express-like ergonomics**: middleware chains, route groups, validators, JWT, CORS, basic auth — all built-in.
- **Portable**: same code runs on Cloudflare Workers (where Vercel doesn't reach) and on Bun (where Vercel doesn't reach).
- **Type-safe client via `hc()`**: similar to tRPC but for plain HTTP.

```ts
import { hc } from 'hono/client';
const client = hc<typeof app>('/');

// Fully typed:
const res = await client.api.posts.$post({ json: { title: 'Hi', body: '...' } });
const post = await res.json(); // typed!
```

### When Hono beats both Route Handlers and tRPC

- You want **one HTTP app** that runs on Next.js today and on Cloudflare Workers later (or vice versa).
- You're shipping a **mobile-app + web-app + admin-tool** and want a single typed REST API.
- You want **Express-style middleware** (CORS, rate-limiting, basic auth, body-size limits) without writing each one yourself.

### When Hono is overkill

- For a few endpoints that only your Next.js app calls — **tRPC** is cleaner.
- For pure webhook receivers — **Route Handlers** are simpler.

---

## 6. Edge Runtime vs Node Runtime — When to Pick Each

| | Edge runtime | Node runtime |
|--|--------------|--------------|
| Cold start | ~5ms | ~100–300ms |
| Available globals | Web standards (`fetch`, `Request`, `Response`, `crypto`) | Full Node + web standards |
| `fs`, `path`, native modules | ❌ | ✅ |
| Prisma | Limited (use Accelerate or Driver Adapters) | ✅ |
| Memory | 128 MB | 1024 MB+ |
| Pricing on Vercel | Cheaper per invocation | More expensive |
| Best for | Cheap, short-running endpoints; auth check; geolocation; A/B routing | Anything DB-heavy, image processing, long-running |

```ts
export const runtime = 'edge';   // or 'nodejs' (default)
```

**Default to `nodejs`.** Move to `edge` only when you've measured that the latency win matters and you've confirmed your DB driver works.

For Postgres on Edge: `@neondatabase/serverless@0.9` (Neon's HTTP-based driver), `@upstash/redis@1` for Redis, or `prisma@5.14 + @prisma/adapter-pg@5` with Driver Adapters.

---

## 7. CORS, Auth, and Other Real Cross-Cutting Concerns

### CORS for Route Handlers

```ts
const allowed = ['https://app.example.com', 'https://admin.example.com'];

export async function OPTIONS(req: Request) {
  const origin = req.headers.get('origin') ?? '';
  if (!allowed.includes(origin)) return new Response(null, { status: 403 });
  return new Response(null, {
    headers: {
      'Access-Control-Allow-Origin': origin,
      'Access-Control-Allow-Methods': 'GET, POST, PATCH, DELETE',
      'Access-Control-Allow-Headers': 'Content-Type, Authorization',
      'Access-Control-Max-Age': '86400',
    },
  });
}
```

For Hono, use the built-in `cors()` middleware:

```ts
import { cors } from 'hono/cors';
app.use('/api/*', cors({ origin: 'https://app.example.com' }));
```

### Rate limiting — `@upstash/ratelimit@1`

```ts
import { Ratelimit } from '@upstash/ratelimit';
import { Redis } from '@upstash/redis';

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(10, '60 s'),
});

export async function POST(req: Request) {
  const ip = req.headers.get('x-forwarded-for') ?? 'unknown';
  const { success } = await ratelimit.limit(ip);
  if (!success) return new Response('Too many requests', { status: 429 });
  // ...
}
```

Real production apps wrap this in a small helper or middleware.

### Auth in Route Handlers — `auth()` from Auth.js v5

```ts
import { auth } from '@/auth';

export const POST = auth(async (req) => {
  if (!req.auth?.user) return new Response('Unauthorized', { status: 401 });
  // req.auth is fully typed (covered in 07.nextjs/04-auth.md)
});
```

Same wrapping in tRPC via the `protectedProcedure` middleware (shown in §4).

---

## 8. OpenAPI — When You Need It

For public APIs consumed by external partners, you'll need OpenAPI/Swagger docs. Options:

| Tool | Approach | Use when |
|------|----------|----------|
| `zod-openapi@4` (or `@hono/zod-openapi@0.18` for Hono) | Generate OpenAPI from Zod schemas | You're already using Zod everywhere |
| `@asteasolutions/zod-to-openapi@7` | Generic Zod → OpenAPI converter | Same as above, framework-agnostic |
| `trpc-openapi@1.x` | Generate OpenAPI from tRPC procedures | You already use tRPC and want partners to call HTTP |
| Hand-written `openapi.yaml` | Just write it | Tiny APIs, full control |

For Hono, the cleanest path is `@hono/zod-openapi@0.18`:

```ts
import { OpenAPIHono, createRoute } from '@hono/zod-openapi';

const app = new OpenAPIHono();
const route = createRoute({
  method: 'get',
  path: '/posts/:id',
  request: { params: z.object({ id: z.string() }) },
  responses: { 200: { content: { 'application/json': { schema: PostSchema } } } },
});
app.openapi(route, (c) => c.json(getPost(c.req.valid('param').id)));
app.doc('/openapi.json', { info: { title: 'API', version: '1.0' }, openapi: '3.1.0' });
```

---

## 9. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Stripe webhook signature verification fails | Read body with `req.text()`, not `req.json()` |
| `req.json()` parses repeatedly fail | The body stream is single-use; clone with `req.clone()` if you need both |
| tRPC client throws `[object Object]` | Add `superjson` transformer on **both** sides |
| Route Handler caches when you didn't expect | Add `export const dynamic = 'force-dynamic'` or `cache: 'no-store'` on internal fetches |
| Edge runtime "Module not found" for Node API | The dependency uses `fs`, `path`, etc. Move to `runtime = 'nodejs'` |
| Prisma fails on Edge | Use Driver Adapters + Neon serverless driver, or stay on Node |
| CORS preflight not working | Need `OPTIONS` handler that returns `204` with the right headers |
| Hono `hc()` client missing types | Re-export the `app` type from a separate file the client imports |
| tRPC procedure renamed and it broke | Good — that's the feature. Update the call site. |

---

## 10. Decision Tree

```
Is this called by a public/external client (mobile, partner, third-party webhook)?
│   YES → Route Handler (route.ts) — stable URL, REST or webhook contract
│   NO  ↓
Is this called only from your own React app and you want full type safety?
│   YES → tRPC 11 (the cleanest internal API for Next.js)
│   NO  ↓
Will this HTTP layer also need to run on Cloudflare Workers / Bun / Deno later?
│   YES → Hono 4 — same code, multiple runtimes
│   NO  ↓
Is it actually a mutation triggered by a form/button in your own UI?
│   YES → Server Action (covered in 07.nextjs/03)
│   NO  → Route Handler is the safe default
```

---

## 11. Summary

| Tool | Pick when |
|------|-----------|
| Server Actions (Next.js) | Internal mutations from your forms/buttons |
| Route Handlers (`route.ts`) | Public APIs, webhooks, file uploads, streaming responses |
| tRPC 11 | Internal API for your own React app — end-to-end types |
| Hono 4 | Portable HTTP across Cloudflare Workers / Bun / Vercel |
| `runtime: 'edge'` | Short, cheap, latency-sensitive endpoints with Edge-compatible deps |
| `runtime: 'nodejs'` (default) | Anything DB-heavy, image processing, native modules |

| Rule | Why |
|------|-----|
| Don't expose tRPC to public consumers | Use Route Handlers or Hono with OpenAPI for that |
| Keep `superjson` configured on both sides of tRPC | Otherwise `Date`, `BigInt`, etc. silently break |
| For Stripe / GitHub / Slack webhooks, read raw body | Signature verification depends on byte-exact input |
| Put Edge-incompatible deps (Prisma classic) on Node runtime | Edge runtime errors are obscure |
| Use Zod for input validation everywhere | Same schema is reusable in client forms (Topic 04.state-and-data/01) |

---

## Further reading

- [Next.js — Route Handlers](https://nextjs.org/docs/app/building-your-application/routing/route-handlers)
- [tRPC v11 — Next.js App Router setup](https://trpc.io/docs/client/nextjs/setup)
- [Hono — Vercel adapter](https://hono.dev/docs/getting-started/vercel)
- [Vercel AI SDK — useChat with streaming](https://sdk.vercel.ai/docs/ai-sdk-ui/chatbot)
- [Stripe — verify webhook signatures](https://docs.stripe.com/webhooks/signatures)
- [Upstash Ratelimit](https://upstash.com/docs/oss/sdks/ts/ratelimit/overview)
