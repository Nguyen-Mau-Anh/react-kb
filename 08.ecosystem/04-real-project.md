# Ecosystem — 04. Real Project: Full-Stack Task Manager (Next.js 14 + Prisma 5 + tRPC 11 + NextAuth 5)

> **What / Why / How** — a single end-to-end project that exercises every previous topic. This is the curriculum's capstone: read it, build it, ship it.

---

## 1. What We're Building

A real, deployable **multi-user task manager**:

- **Auth**: GitHub OAuth + email magic links via `next-auth@5`
- **Data**: Postgres (Neon) + `prisma@5.14` for schema and queries
- **API**: `@trpc/server@11` + `@trpc/react-query@11` for typed procedures
- **UI**: `next@14.2` App Router + `shadcn/ui` (Radix UI 1.x + Tailwind 3.4)
- **Forms**: `react-hook-form@7.51` + `zod@3.23`
- **Server cache**: TanStack Query 5.28 with the `HydrationBoundary` SSR pattern
- **Testing**: `vitest@1.6` + `@testing-library/react@15` + `msw@2.3`
- **Deployment**: Vercel + Neon Postgres branching per PR

Features:
- Sign in (GitHub OAuth, magic-link email).
- Create projects (a user belongs to many; projects belong to one user).
- Add tasks to a project; tasks have `title`, `description`, `dueDate`, `status` (`open` / `in_progress` / `done`).
- Drag-reorder tasks within a project (with `@dnd-kit/core@6`).
- Real-time sync of task updates across tabs (TanStack Query background refetch).
- A public `/project/[id]/share` route with a shareable read-only view.

This file walks through every architectural decision and the parts that surprise people.

---

## 2. Stack at a Glance

| Layer | Choice | Why this version |
|-------|--------|------------------|
| Framework | `next@14.2` | App Router stable, Server Actions stable, Turbopack dev |
| Language | `typescript@5.4` | `satisfies`, `using`, NoInfer<T> |
| Database | Postgres 16 on Neon | Branching for previews, free tier covers MVP |
| ORM | `prisma@5.14` + `@prisma/client@5.14` | Mature TS API, strong Next.js examples |
| Auth | `next-auth@5.0.0-beta` (Auth.js v5) | App Router-native, JWT sessions on Edge |
| API layer | `@trpc/server@11` + `@trpc/client@11` + `@trpc/react-query@11` | End-to-end typed RPC; covered in `07.nextjs/06-api-routes.md` |
| Server-state cache | `@tanstack/react-query@5.28` | Pairs with tRPC; covered in `04.state-and-data/03-data-fetching.md` |
| Forms | `react-hook-form@7.51` + `@hookform/resolvers@3.4` + `zod@3.23` | Covered in `04.state-and-data/01-forms.md` |
| UI primitives | `shadcn/ui` (`@radix-ui/react-*@1.1`) + `tailwindcss@3.4` | Covered in `08.ecosystem/02-ui-libraries.md` |
| Drag-and-drop | `@dnd-kit/core@6` + `@dnd-kit/sortable@8` | Modern, accessible, hooks-based |
| Animation | `framer-motion@11` (with `LazyMotion`) | Covered in `08.ecosystem/01-animation.md` |
| Toast | `sonner@1.5` | The shadcn/ui default |
| Validation | `zod@3.23` | Single schema reused on client + server |
| Error tracking | `@sentry/nextjs@8` | Real production observability |
| Tests | `vitest@1.6` + `@testing-library/react@15` + `msw@2.3` | Covered in `06.testing-perf/01-testing.md` |
| Deploy | Vercel + Neon (or Railway as $/mo alternative) | Covered in `07.nextjs/05-deployment.md` |

Everything in this list has its own deep-dive earlier in the KB.

---

## 3. Repository Layout

```
task-manager/
├── prisma/
│   ├── schema.prisma
│   └── seed.ts
├── src/
│   ├── app/
│   │   ├── (auth)/
│   │   │   └── login/page.tsx
│   │   ├── (app)/
│   │   │   ├── layout.tsx                     ← logged-in chrome
│   │   │   ├── page.tsx                        ← / (project list)
│   │   │   └── project/
│   │   │       ├── [id]/
│   │   │       │   ├── page.tsx                ← /project/:id (board)
│   │   │       │   ├── loading.tsx
│   │   │       │   └── error.tsx
│   │   │       └── new/page.tsx
│   │   ├── (public)/
│   │   │   └── project/[id]/share/page.tsx     ← read-only shareable view
│   │   ├── api/
│   │   │   ├── auth/[...nextauth]/route.ts
│   │   │   └── trpc/[trpc]/route.ts
│   │   ├── layout.tsx                          ← root layout
│   │   ├── globals.css
│   │   └── not-found.tsx
│   ├── components/
│   │   ├── ui/                                  ← shadcn/ui copy-paste lives here
│   │   ├── project-card.tsx
│   │   ├── task-row.tsx
│   │   ├── task-form.tsx
│   │   └── kanban-board.tsx
│   ├── server/
│   │   ├── trpc.ts                              ← initTRPC + procedure helpers
│   │   ├── routers/
│   │   │   ├── _app.ts                          ← appRouter
│   │   │   ├── project.ts
│   │   │   └── task.ts
│   │   └── db.ts                                ← Prisma singleton
│   ├── lib/
│   │   ├── utils.ts                             ← cn() helper
│   │   └── validators.ts                        ← shared Zod schemas
│   ├── trpc/
│   │   └── client.ts                            ← createTRPCReact<AppRouter>()
│   ├── auth.ts                                  ← Node-only NextAuth setup
│   ├── auth.config.ts                           ← Edge-safe NextAuth config
│   └── middleware.ts                            ← protects (app)/* routes
├── __tests__/
│   ├── handlers.ts                              ← MSW handlers
│   └── setup.ts
├── .env.example
├── package.json
├── tsconfig.json
├── tailwind.config.ts
├── components.json                              ← shadcn/ui config
├── vitest.config.ts
└── next.config.js
```

Two route groups split logged-in vs public chrome (covered in `07.nextjs/02-routing.md`).

---

## 4. The Domain Model

### Prisma schema

```prisma
// prisma/schema.prisma
generator client { provider = "prisma-client-js" }
datasource db { provider = "postgresql"; url = env("DATABASE_URL") }

// ---- Auth.js required models (covered in 07.nextjs/04-auth.md) ----
model User {
  id            String     @id @default(cuid())
  name          String?
  email         String     @unique
  emailVerified DateTime?
  image         String?
  accounts      Account[]
  sessions      Session[]
  projects      Project[]
  createdAt     DateTime   @default(now())
}
model Account { /* ... NextAuth standard ... */ }
model Session { /* ... NextAuth standard ... */ }
model VerificationToken { /* ... NextAuth standard ... */ }

// ---- Domain models ----
model Project {
  id          String   @id @default(cuid())
  name        String
  description String?
  ownerId     String
  shareToken  String?  @unique          // null = not shared, otherwise share-by-link
  owner       User     @relation(fields: [ownerId], references: [id], onDelete: Cascade)
  tasks       Task[]
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  @@index([ownerId])
}

enum TaskStatus { open in_progress done }

model Task {
  id          String     @id @default(cuid())
  projectId   String
  title       String
  description String?
  status      TaskStatus @default(open)
  dueDate     DateTime?
  position    Int                                     // for drag-reorder
  project     Project    @relation(fields: [projectId], references: [id], onDelete: Cascade)
  createdAt   DateTime   @default(now())
  updatedAt   DateTime   @updatedAt

  @@index([projectId, status])
  @@index([projectId, position])
}
```

### Why these decisions

- **`cuid()` not `uuid()`**: collision-resistant, URL-safe, sortable-ish. Prisma's default for new schemas.
- **`@@index([ownerId])`** on `Project`: every "list my projects" query filters by `ownerId`; without the index, it'd table-scan once you have 10K projects.
- **`@@index([projectId, status])`** on `Task`: the kanban view filters by both. Compound index is faster than two single-column indexes for the AND case.
- **`position: Int`**: simple integer ordering. Drag-reorder updates only the moved task's position to the average of its neighbors. When positions get too dense, run a re-pack background job.
- **`shareToken`**: nullable means "not shared." Setting a token enables a public read-only URL. Toggle by setting/clearing the column.
- **Cascade deletes**: deleting a `User` deletes their `Project`s and all their `Task`s. Saves you a DELETE-cascade hand-roll.
- **`enum TaskStatus`**: native Postgres enum. Cheap, indexable, `prisma generate` produces a TS union (`'open' | 'in_progress' | 'done'`).

---

## 5. Validators — Zod Schemas Used by Both Server and Client

```ts
// src/lib/validators.ts
import { z } from 'zod';

export const TaskStatus = z.enum(['open', 'in_progress', 'done']);
export type TaskStatus = z.infer<typeof TaskStatus>;

export const CreateProjectInput = z.object({
  name: z.string().min(1, 'Name is required').max(100),
  description: z.string().max(2000).optional(),
});
export type CreateProjectInput = z.infer<typeof CreateProjectInput>;

export const CreateTaskInput = z.object({
  projectId: z.string().cuid(),
  title: z.string().min(1).max(200),
  description: z.string().max(5000).optional(),
  status: TaskStatus.default('open'),
  dueDate: z.coerce.date().optional(),
});
export type CreateTaskInput = z.infer<typeof CreateTaskInput>;

export const UpdateTaskInput = CreateTaskInput.partial().extend({
  id: z.string().cuid(),
});
export type UpdateTaskInput = z.infer<typeof UpdateTaskInput>;
```

The `react-hook-form` setup uses `zodResolver(CreateTaskInput)`. The tRPC procedure uses `.input(CreateTaskInput)`. **One schema, two consumers, zero duplication.** Covered in `04.state-and-data/01-forms.md`.

---

## 6. Auth Setup (Reuses `07.nextjs/04-auth.md`)

```ts
// src/auth.config.ts — Edge-safe
import type { NextAuthConfig } from 'next-auth';

export default {
  pages: { signIn: '/login' },
  callbacks: {
    authorized({ auth, request: { nextUrl } }) {
      const isLoggedIn = !!auth?.user;
      const isOnApp = nextUrl.pathname === '/' || nextUrl.pathname.startsWith('/project') && !nextUrl.pathname.endsWith('/share');
      const isOnPublicShare = nextUrl.pathname.includes('/share');
      if (isOnApp && !isOnPublicShare) return isLoggedIn;
      return true;
    },
  },
  providers: [],
} satisfies NextAuthConfig;
```

```ts
// src/auth.ts — Node-only
import NextAuth from 'next-auth';
import GitHub from 'next-auth/providers/github';
import Email from 'next-auth/providers/nodemailer';
import { PrismaAdapter } from '@auth/prisma-adapter';
import { db } from './server/db';
import authConfig from './auth.config';

export const { handlers, auth, signIn, signOut } = NextAuth({
  adapter: PrismaAdapter(db),
  session: { strategy: 'jwt' },
  ...authConfig,
  providers: [
    GitHub({
      clientId: process.env.AUTH_GITHUB_ID!,
      clientSecret: process.env.AUTH_GITHUB_SECRET!,
    }),
    Email({
      server: process.env.EMAIL_SERVER!,
      from: process.env.EMAIL_FROM!,
    }),
  ],
});
```

```ts
// src/middleware.ts
import NextAuth from 'next-auth';
import authConfig from './auth.config';
export const { auth: middleware } = NextAuth(authConfig);
export const config = { matcher: ['/((?!api|_next/static|_next/image|favicon.ico).*)'] };
```

---

## 7. tRPC Server — Procedures Per Domain

```ts
// src/server/trpc.ts
import { initTRPC, TRPCError } from '@trpc/server';
import superjson from 'superjson';
import { auth } from '@/auth';

const t = initTRPC.context<{ session: Awaited<ReturnType<typeof auth>> }>().create({
  transformer: superjson,
  errorFormatter({ shape, error }) {
    return {
      ...shape,
      data: { ...shape.data, zodError: error.cause instanceof Error && 'flatten' in error.cause ? (error.cause as any).flatten() : null },
    };
  },
});

export const router = t.router;
export const publicProcedure = t.procedure;
export const protectedProcedure = t.procedure.use(({ ctx, next }) => {
  if (!ctx.session?.user?.id) throw new TRPCError({ code: 'UNAUTHORIZED' });
  return next({ ctx: { ...ctx, user: ctx.session.user } });
});
```

```ts
// src/server/routers/project.ts
import { router, protectedProcedure } from '../trpc';
import { db } from '../db';
import { CreateProjectInput } from '@/lib/validators';
import { z } from 'zod';
import { TRPCError } from '@trpc/server';
import { randomUUID } from 'crypto';

export const projectRouter = router({
  list: protectedProcedure.query(({ ctx }) =>
    db.project.findMany({
      where: { ownerId: ctx.user.id },
      orderBy: { updatedAt: 'desc' },
    })
  ),

  byId: protectedProcedure
    .input(z.object({ id: z.string().cuid() }))
    .query(async ({ ctx, input }) => {
      const project = await db.project.findFirst({
        where: { id: input.id, ownerId: ctx.user.id },
        include: { tasks: { orderBy: { position: 'asc' } } },
      });
      if (!project) throw new TRPCError({ code: 'NOT_FOUND' });
      return project;
    }),

  create: protectedProcedure
    .input(CreateProjectInput)
    .mutation(({ ctx, input }) =>
      db.project.create({ data: { ...input, ownerId: ctx.user.id } })
    ),

  toggleShare: protectedProcedure
    .input(z.object({ id: z.string().cuid() }))
    .mutation(async ({ ctx, input }) => {
      const p = await db.project.findFirstOrThrow({ where: { id: input.id, ownerId: ctx.user.id } });
      return db.project.update({
        where: { id: p.id },
        data: { shareToken: p.shareToken ? null : randomUUID() },
      });
    }),
});
```

```ts
// src/server/routers/task.ts
import { router, protectedProcedure } from '../trpc';
import { db } from '../db';
import { CreateTaskInput, UpdateTaskInput } from '@/lib/validators';
import { z } from 'zod';
import { TRPCError } from '@trpc/server';

async function assertProjectOwnership(userId: string, projectId: string) {
  const found = await db.project.findFirst({ where: { id: projectId, ownerId: userId } });
  if (!found) throw new TRPCError({ code: 'FORBIDDEN' });
}

export const taskRouter = router({
  create: protectedProcedure
    .input(CreateTaskInput)
    .mutation(async ({ ctx, input }) => {
      await assertProjectOwnership(ctx.user.id, input.projectId);
      const max = await db.task.aggregate({
        where: { projectId: input.projectId },
        _max: { position: true },
      });
      return db.task.create({
        data: { ...input, position: (max._max.position ?? 0) + 1024 },
      });
    }),

  update: protectedProcedure
    .input(UpdateTaskInput)
    .mutation(async ({ ctx, input }) => {
      const task = await db.task.findFirstOrThrow({ where: { id: input.id }, select: { projectId: true } });
      await assertProjectOwnership(ctx.user.id, task.projectId);
      const { id, ...patch } = input;
      return db.task.update({ where: { id }, data: patch });
    }),

  reorder: protectedProcedure
    .input(z.object({
      projectId: z.string().cuid(),
      orderedIds: z.array(z.string().cuid()),
    }))
    .mutation(async ({ ctx, input }) => {
      await assertProjectOwnership(ctx.user.id, input.projectId);
      // Re-pack positions in steps of 1024 to leave room for future inserts
      await db.$transaction(
        input.orderedIds.map((id, i) =>
          db.task.update({ where: { id }, data: { position: (i + 1) * 1024 } })
        )
      );
    }),

  delete: protectedProcedure
    .input(z.object({ id: z.string().cuid() }))
    .mutation(async ({ ctx, input }) => {
      const task = await db.task.findFirstOrThrow({ where: { id: input.id }, select: { projectId: true } });
      await assertProjectOwnership(ctx.user.id, task.projectId);
      await db.task.delete({ where: { id: input.id } });
    }),
});
```

```ts
// src/server/routers/_app.ts
import { router } from '../trpc';
import { projectRouter } from './project';
import { taskRouter } from './task';
export const appRouter = router({
  project: projectRouter,
  task: taskRouter,
});
export type AppRouter = typeof appRouter;
```

### Why these patterns are the real ones

- **`assertProjectOwnership`**: every task mutation re-verifies the user owns the project. **Never trust the client to send a `projectId` you assume they own.** This is the #1 horizontal-privesc pattern in real SaaS.
- **`position: ... + 1024` step**: gives drag-and-drop room to insert without re-packing every time. When two adjacent positions get within 1, run the re-pack `db.$transaction`.
- **`db.$transaction` for reorder**: all-or-nothing; either every position update succeeds or none do.
- **`findFirstOrThrow`**: cleaner than `findFirst({ ... }) || throw`. Prisma 5 has it natively.
- **`superjson` on both ends**: Date and BigInt would otherwise round-trip as strings.

---

## 8. tRPC Wired into Next.js App Router

```ts
// src/app/api/trpc/[trpc]/route.ts
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

```ts
// src/trpc/client.ts
import { createTRPCReact } from '@trpc/react-query';
import type { AppRouter } from '@/server/routers/_app';
export const trpc = createTRPCReact<AppRouter>();
```

```tsx
// src/app/providers.tsx
'use client';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { httpBatchLink } from '@trpc/client';
import { trpc } from '@/trpc/client';
import superjson from 'superjson';
import { useState } from 'react';

export function Providers({ children }: { children: React.ReactNode }) {
  const [qc] = useState(() => new QueryClient({
    defaultOptions: { queries: { staleTime: 30_000 } },
  }));
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

---

## 9. The Project List Page — Server Component With Hydration

```tsx
// src/app/(app)/page.tsx
import { auth } from '@/auth';
import { redirect } from 'next/navigation';
import { appRouter } from '@/server/routers/_app';
import { ProjectListClient } from '@/components/project-list-client';

export default async function HomePage() {
  const session = await auth();
  if (!session?.user) redirect('/login');

  // Direct server-side caller — no HTTP overhead (covered in 07.nextjs/06)
  const caller = appRouter.createCaller({ session });
  const initialProjects = await caller.project.list();

  return <ProjectListClient initialData={initialProjects} />;
}
```

```tsx
// src/components/project-list-client.tsx
'use client';
import { trpc } from '@/trpc/client';
import { ProjectCard } from './project-card';
import type { RouterOutputs } from '@/trpc/types'; // tiny helper exporting inferRouterOutputs

type Project = RouterOutputs['project']['list'][number];

export function ProjectListClient({ initialData }: { initialData: Project[] }) {
  const { data } = trpc.project.list.useQuery(undefined, { initialData });
  return (
    <div className="grid gap-4 md:grid-cols-2 lg:grid-cols-3">
      {data.map(p => <ProjectCard key={p.id} project={p} />)}
    </div>
  );
}
```

`initialData` seeds TanStack Query so the first render shows real data without a fetch round-trip. Subsequent navigations use the cached value; background refetches keep it fresh.

---

## 10. The Kanban Board With Drag-and-Drop

```tsx
// src/components/kanban-board.tsx
'use client';
import { trpc } from '@/trpc/client';
import { DndContext, closestCenter } from '@dnd-kit/core';
import { SortableContext, useSortable, arrayMove, verticalListSortingStrategy } from '@dnd-kit/sortable';
import { CSS } from '@dnd-kit/utilities';

type Task = RouterOutputs['project']['byId']['tasks'][number];

export function KanbanBoard({ projectId, initialTasks }: { projectId: string; initialTasks: Task[] }) {
  const utils = trpc.useUtils();
  const tasksQuery = trpc.project.byId.useQuery({ id: projectId }, { initialData: { /* ... */ } as any });
  const tasks = tasksQuery.data?.tasks ?? initialTasks;

  const reorder = trpc.task.reorder.useMutation({
    onMutate: async ({ orderedIds }) => {
      await utils.project.byId.cancel({ id: projectId });
      const prev = utils.project.byId.getData({ id: projectId });
      if (prev) {
        utils.project.byId.setData({ id: projectId }, {
          ...prev,
          tasks: orderedIds
            .map(id => prev.tasks.find(t => t.id === id)!)
            .filter(Boolean),
        });
      }
      return { prev };
    },
    onError: (_e, _v, ctx) => { if (ctx?.prev) utils.project.byId.setData({ id: projectId }, ctx.prev); },
    onSettled: () => utils.project.byId.invalidate({ id: projectId }),
  });

  function handleDragEnd(event: any) {
    const { active, over } = event;
    if (!over || active.id === over.id) return;
    const oldIndex = tasks.findIndex(t => t.id === active.id);
    const newIndex = tasks.findIndex(t => t.id === over.id);
    const next = arrayMove(tasks, oldIndex, newIndex);
    reorder.mutate({ projectId, orderedIds: next.map(t => t.id) });
  }

  return (
    <DndContext collisionDetection={closestCenter} onDragEnd={handleDragEnd}>
      <SortableContext items={tasks.map(t => t.id)} strategy={verticalListSortingStrategy}>
        <ul className="space-y-2">
          {tasks.map(t => <SortableTaskRow key={t.id} task={t} />)}
        </ul>
      </SortableContext>
    </DndContext>
  );
}

function SortableTaskRow({ task }: { task: Task }) {
  const { attributes, listeners, setNodeRef, transform, transition } = useSortable({ id: task.id });
  return (
    <li
      ref={setNodeRef}
      style={{ transform: CSS.Transform.toString(transform), transition }}
      {...attributes}
      {...listeners}
      className="rounded-md border bg-card p-3"
    >
      {task.title}
    </li>
  );
}
```

The `onMutate` / `onError` rollback pattern (covered in `04.state-and-data/03-data-fetching.md`) gives the user instant drag feedback while the network call settles.

---

## 11. Forms — Add Task With Real Validation

```tsx
// src/components/task-form.tsx
'use client';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { CreateTaskInput } from '@/lib/validators';
import { trpc } from '@/trpc/client';
import { toast } from 'sonner';

export function TaskForm({ projectId }: { projectId: string }) {
  const utils = trpc.useUtils();
  const create = trpc.task.create.useMutation({
    onSuccess: () => {
      utils.project.byId.invalidate({ id: projectId });
      toast.success('Task added');
      reset();
    },
    onError: (e) => toast.error(e.message),
  });

  const { register, handleSubmit, reset, formState: { errors, isSubmitting } } = useForm({
    resolver: zodResolver(CreateTaskInput),
    defaultValues: { projectId, status: 'open' as const },
  });

  return (
    <form
      onSubmit={handleSubmit((data) => create.mutateAsync(data))}
      className="space-y-3 rounded-md border p-4"
    >
      <input type="hidden" {...register('projectId')} />
      <div>
        <label className="text-sm font-medium">Title</label>
        <input {...register('title')} className="w-full rounded border px-2 py-1.5" />
        {errors.title && <p className="text-sm text-red-600">{errors.title.message}</p>}
      </div>
      <div>
        <label className="text-sm font-medium">Description</label>
        <textarea {...register('description')} className="w-full rounded border px-2 py-1.5" />
      </div>
      <button
        disabled={isSubmitting || create.isPending}
        className="rounded bg-blue-600 px-3 py-1.5 text-white disabled:opacity-50"
      >
        {create.isPending ? 'Adding…' : 'Add task'}
      </button>
    </form>
  );
}
```

Same `CreateTaskInput` schema validates client-side **and** server-side. The server's `protectedProcedure.input(CreateTaskInput)` rejects bad data even if a client bypasses the form.

---

## 12. The Public Share Page — A Different Mental Model

```tsx
// src/app/(public)/project/[id]/share/page.tsx
import { db } from '@/server/db';
import { notFound } from 'next/navigation';

export const revalidate = 60; // cache 60s — public, low-stakes

export async function generateMetadata({ params }: { params: { id: string } }) {
  const project = await db.project.findFirst({
    where: { id: params.id, shareToken: { not: null } },
    select: { name: true, description: true },
  });
  if (!project) return {};
  return {
    title: project.name,
    description: project.description ?? 'Shared project',
  };
}

export default async function ShareView({ params, searchParams }: {
  params: { id: string };
  searchParams: { t?: string };
}) {
  const project = await db.project.findFirst({
    where: { id: params.id, shareToken: searchParams.t },
    include: { tasks: { orderBy: { position: 'asc' } } },
  });
  if (!project) notFound();

  return (
    <main className="mx-auto max-w-2xl p-6">
      <h1 className="text-2xl font-bold">{project.name}</h1>
      {project.description && <p className="text-muted-foreground">{project.description}</p>}
      <ul className="mt-6 space-y-2">
        {project.tasks.map(t => (
          <li key={t.id} className="rounded-md border bg-card p-3">
            <span className={t.status === 'done' ? 'line-through' : ''}>{t.title}</span>
          </li>
        ))}
      </ul>
    </main>
  );
}
```

Three key facts:
- **Direct DB query, no tRPC.** Public views don't need auth context; the share-token check enforces access.
- **`searchParams.t` is the share token** — only matches when both `id` AND `t` are correct. Random URL guessing won't work.
- **`revalidate = 60`** — serves cached HTML for 60s; public pages shouldn't hit the DB on every request.

---

## 13. Tests — One Real Integration Test

```tsx
// __tests__/task-form.test.tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { TaskForm } from '@/components/task-form';
import { Providers } from '@/app/providers';
import { server } from './setup';
import { http, HttpResponse } from 'msw';

it('shows validation error and creates task', async () => {
  const user = userEvent.setup();
  render(<Providers><TaskForm projectId="abc" /></Providers>);

  // Submit empty: title is required
  await user.click(screen.getByRole('button', { name: /add task/i }));
  expect(await screen.findByText(/required/i)).toBeInTheDocument();

  // Mock the tRPC endpoint
  server.use(
    http.post('/api/trpc/task.create', () =>
      HttpResponse.json([{ result: { data: { json: { id: 'new', title: 'Hi' } } } }])
    )
  );

  await user.type(screen.getByLabelText(/title/i), 'Buy milk');
  await user.click(screen.getByRole('button', { name: /add task/i }));
  expect(await screen.findByText(/added/i)).toBeInTheDocument(); // sonner toast
});
```

Covered in detail in `06.testing-perf/01-testing.md`.

---

## 14. Deployment

### Vercel + Neon

1. Create a Neon project; note the pooled connection string.
2. `vercel link`, then add env vars: `DATABASE_URL`, `AUTH_SECRET`, `AUTH_GITHUB_ID`, `AUTH_GITHUB_SECRET`, `EMAIL_SERVER`, `EMAIL_FROM`.
3. Add a build step that runs Prisma generate + migrate:
   ```jsonc
   // package.json
   "scripts": {
     "build": "prisma generate && prisma migrate deploy && next build",
     "postinstall": "prisma generate"
   }
   ```
4. Push to `main` → deploy.
5. Install the Neon Vercel integration → every PR gets its own DB branch automatically.

### Real env file

```bash
# .env.example
DATABASE_URL="postgres://user:password@host/db?pgbouncer=true&connect_timeout=15"
DIRECT_URL="postgres://user:password@host/db"           # for migrations, bypasses pooler
AUTH_SECRET="<generate with: npx auth secret>"
AUTH_GITHUB_ID=""
AUTH_GITHUB_SECRET=""
EMAIL_SERVER="smtps://user:pass@smtp.resend.com:465"
EMAIL_FROM="hello@yourdomain.com"
NEXT_PUBLIC_APP_URL="https://yourdomain.com"
```

Covered in `07.nextjs/05-deployment.md`.

---

## 15. What This Project Tested From the Curriculum

| Topic | Used |
|-------|------|
| `01.fundamentals/01-how-react-works.md` | Understanding RSC + hydration + Fiber-driven concurrent renders |
| `01.fundamentals/02-jsx-and-rendering.md` | JSX in client components, automatic runtime |
| `01.fundamentals/03-components-props.md` | Function components, prop typing, `children`, composition |
| `02.hooks/01-usestate.md` | Local component state in client components |
| `02.hooks/02-useeffect.md` | Avoided — replaced with TanStack Query and Server Components |
| `02.hooks/03-useref-usecontext.md` | Trpc + QueryClient providers |
| `02.hooks/04-usememo-usecallback.md` | Sparingly — letting TanStack Query handle most stability |
| `02.hooks/05-custom-hooks.md` | tRPC's `useQuery`/`useMutation` are custom hooks under the hood |
| `03.rendering-patterns/*` | Event handlers, list keys, compound primitives via Radix |
| `04.state-and-data/01-forms.md` | `react-hook-form` + Zod resolver |
| `04.state-and-data/02-state-management.md` | Zustand wasn't needed; URL + server state covered everything |
| `04.state-and-data/03-data-fetching.md` | TanStack Query patterns, optimistic updates, hydration |
| `05.routing-and-styling/01-routing-react-router.md` | Skipped — using Next.js router instead |
| `05.routing-and-styling/02-styling.md` | Tailwind + cva + tailwind-merge via shadcn/ui |
| `05.routing-and-styling/03-typescript-with-react.md` | Native HTML attribute types, generics, Zod-inferred types |
| `06.testing-perf/01-testing.md` | Vitest + RTL + MSW |
| `06.testing-perf/02-performance.md` | Code splitting per route, virtualization for very long task lists |
| `07.nextjs/01-fundamentals.md` | App Router, Server vs Client Components |
| `07.nextjs/02-routing.md` | Route groups `(app)` and `(public)`, `loading.tsx`, `error.tsx` |
| `07.nextjs/03-data-fetching.md` | Server Components with direct Prisma queries; Server Actions optional |
| `07.nextjs/04-auth.md` | Auth.js v5, GitHub + email magic links, Edge-safe middleware |
| `07.nextjs/05-deployment.md` | Vercel + Neon with PR branching |
| `07.nextjs/06-api-routes.md` | tRPC mounted under `app/api/trpc/[trpc]/route.ts` |
| `07.nextjs/07-seo-metadata.md` | `generateMetadata` for the public share page |
| `08.ecosystem/01-animation.md` | Framer Motion for the toast/list-enter animations |
| `08.ecosystem/02-ui-libraries.md` | shadcn/ui + Radix UI + Tailwind |
| `08.ecosystem/03-monorepo.md` | Single-app project; if expanding to mobile (Expo) or admin app, switch to Turborepo |

Every choice traces back to a deeper topic. That's the whole point of the curriculum.

---

## 16. What to Build Next

Real extensions worth attempting:
- **Real-time collaboration** with `partykit@0.x` or `liveblocks@2` (covered in a future iteration).
- **Mobile** with Expo SDK 51 + Tamagui (covered in a future iteration).
- **AI features** with the `ai@4` SDK — "summarize this project's overdue tasks", "suggest a task title from a description".
- **Comments and threads** — adds a `Comment` model and a real-time channel per task.
- **File attachments** — Vercel Blob or S3 + pre-signed URLs from a Server Action.
- **Audit log** — every mutation appends to a `Log` table; show in the project sidebar.

Each one stresses a different part of the curriculum and is a solid week-long deep dive.

---

## 17. Common Pitfalls Specific to This Stack

| Pitfall | Fix |
|---------|-----|
| `prisma generate` fails on Vercel | Add `"postinstall": "prisma generate"` to `package.json` |
| Connection limits exceeded on Neon serverless | Use the **pooled** connection string (`?pgbouncer=true&connect_timeout=15`); set `DIRECT_URL` for migrations only |
| tRPC mutations don't appear in `tsServer` autocomplete | Restart TS server in your IDE after server-side changes |
| `superjson` not configured on one side | `Date` objects round-trip as strings → broken sort logic. Configure on both client and server transformer. |
| Drag reorder reverts after drop | The mutation didn't optimistically update the cache; add `onMutate` |
| Auth cookie absent on production | `AUTH_SECRET` not set; or `trustHost` not enabled for preview deploys |
| Public share page shows 404 with valid token | The token was passed in `searchParams.t`, query whichever name your URL uses |
| Sonner toasts don't appear | `<Toaster />` not mounted in root layout |
| Hydration mismatch on the project list | `initialData` typing differs from runtime fetch result; align with `inferRouterOutputs` |

---

## Summary

| Capstone Win | Where the Curriculum Earned It |
|--------------|--------------------------------|
| End-to-end type safety with no codegen | tRPC 11 + Zod + TS 5.4 |
| One auth call across RSC, actions, middleware | Auth.js v5 |
| Optimistic drag-reorder | TanStack Query + dnd-kit |
| Public share view that's safe and SEO-friendly | Server Components + `revalidate` + Metadata API |
| One Zod schema for client form + server input | `react-hook-form` + tRPC input validation |
| Free per-PR DB branches | Vercel + Neon integration |

| Rule | Why |
|------|-----|
| Never trust client-supplied `projectId` | Verify ownership server-side every time |
| Use `position` integers in steps for drag-reorder | Prevents constant rewrites of every row |
| Keep `superjson` configured on both ends of tRPC | Or `Date` and `BigInt` round-trip wrong |
| Seed TanStack Query with server data on first render | Eliminates the loading flash |
| Use route groups to separate logged-in vs public chrome | Different layouts at the same URL root |

---

## Further reading

- [T3 Stack docs](https://create.t3.gg/) — opinionated starter that wires the whole stack together
- [Cal.com source](https://github.com/calcom/cal.com) — production-grade real-world reference
- [Linear Engineering blog](https://linear.app/blog/engineering) — patterns for ambitious React + tRPC apps
- [TanStack Query — Optimistic Updates](https://tanstack.com/query/v5/docs/framework/react/guides/optimistic-updates)
- [Prisma — Connection Management](https://www.prisma.io/docs/orm/prisma-client/setup-and-configuration/databases-connections)
- [Auth.js v5 docs](https://authjs.dev/getting-started)
- [shadcn/ui blocks](https://ui.shadcn.com/blocks)
