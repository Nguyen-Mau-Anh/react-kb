# Service Stack — 02. GraphQL with React: Apollo 3 vs urql 4 vs Relay 18, Codegen, Subscriptions

> **What / Why / How** — pick **`@apollo/client@3.11`** for big teams and ecosystem; **`urql@4`** for small bundles and simplicity; **`relay@18`** when you can model the whole client around fragments. Always run **`graphql-codegen@5`** for typed hooks. **For internal-only APIs, prefer tRPC** (covered in `13.service-stack/01-trpc-deep.md`); pick GraphQL when you have multiple consumers, complex relational reads, or a federated server.

---

## 1. The Real Choices in 2026

| Tool | npm | Best for |
|------|-----|----------|
| **Apollo Client 3.11** | `@apollo/client@3.11`, `graphql@16` | Default for most teams; deepest ecosystem; React Server Components support via `@apollo/client-integration-nextjs@0.12` |
| **urql 4** | `@urql/core@5`, `urql@4`, `@urql/exchange-graphcache@7` | Small bundle (~7 KB without graphcache); explicit cache control; great for SPAs |
| **Relay 18** | `relay-runtime@18`, `react-relay@18`, `relay-compiler@18` | Meta-grade performance; fragment-driven architecture; steepest learning curve |
| **GraphQL Yoga 5** | `graphql-yoga@5` (server) | Most-used self-host server in 2026 |
| **Apollo Server 4** | `@apollo/server@4` | Full Apollo stack; Federation v2 |
| **Hot Chocolate** (.NET) / **Strawberry** (Python) / **gqlgen** (Go) | various | When the backend isn't Node |
| **Apollo Router** | binary, Rust | GraphQL Federation gateway in front of subgraphs |
| **GraphQL Codegen 5** | `@graphql-codegen/cli@5`, `@graphql-codegen/typescript@4`, `@graphql-codegen/typescript-react-apollo@4`, `@graphql-codegen/client-preset@4` | Type-safe queries from the schema. **Required for any production GraphQL React app.** |
| **`graphql-request@7`** | `graphql-request@7` | One-off scripts, server-to-server queries; not a full client |
| **`tanstack-query` + `graphql-request`** | `@tanstack/react-query@5`, `graphql-request@7` | If you only need a few queries and already use TanStack Query |

**The 2026 default for most teams**: **Apollo Client 3.11 + GraphQL Codegen 5 with the client-preset**. Best documentation, largest community, best Next.js App Router support, and codegen is now first-class.

For OSS/lightweight projects: **urql 4** is a strong second place — smaller bundle, less ceremony, same TanStack-Query-style hooks.

For Meta-scale apps with strict performance budgets: **Relay 18**. Few teams need this, but when you do, nothing else competes.

---

## 2. Should You Use GraphQL At All?

The honest decision matrix in 2026:

| Situation | Pick |
|-----------|------|
| Internal API consumed only by your own React app(s) | **tRPC 11** (covered in `13.service-stack/01-trpc-deep.md`) — same type safety, less complexity |
| Public API consumed by mobile apps, partners, third parties | **REST via Route Handlers** (covered in `07.nextjs/06-api-routes.md`) or **Hono 4** for stable HTTP contracts |
| Multiple frontends (web + native + admin) hitting one backend | **GraphQL** wins here — one schema, every client picks fields it needs |
| Complex deeply-nested reads (project → tasks → comments → users) | GraphQL's batching + DataLoader pattern shines |
| Large company with multiple backend teams | **Federation v2** — each team owns a subgraph, Apollo Router stitches them |
| Need type-safe, evolution-friendly contract for non-tRPC consumers | GraphQL with `__typename` + nullable fields handles this gracefully |
| 5-engineer team building a single SaaS app | **Don't use GraphQL.** tRPC + Server Components is faster to ship. |

The 2026 reality: **GraphQL adoption peaked around 2021–2023 and has receded** in small/mid-size teams as tRPC and React Server Components covered the type-safety angle without GraphQL's overhead. Where GraphQL still wins: **shared schema across multiple consumers**, **federated architectures**, **deeply nested relational data with field-level access control**.

This file assumes you've already decided GraphQL is the right answer. If you haven't — re-read `13.service-stack/01-trpc-deep.md`.

---

## 3. Apollo Client 3.11 — The Default

### What

`@apollo/client@3.11` is the most-used GraphQL client for React. Wraps your queries in `useQuery` / `useMutation` hooks, manages a normalized cache, supports subscriptions over WS or SSE, and integrates with React Server Components via `@apollo/client-integration-nextjs@0.12`.

### Setup — Next.js 14+ App Router

```bash
npm i @apollo/client@3.11 graphql@16
npm i @apollo/client-integration-nextjs@0.12
```

```tsx
// lib/apollo-client.ts
'use client';
import { HttpLink, from } from '@apollo/client';
import {
  ApolloNextAppProvider,
  ApolloClient,
  InMemoryCache,
  SSRMultipartLink,
} from '@apollo/client-integration-nextjs';

function makeClient() {
  const httpLink = new HttpLink({
    uri: process.env.NEXT_PUBLIC_GRAPHQL_URL!,
    fetchOptions: { cache: 'no-store' },           // Next.js will cache via Apollo, not the platform
    credentials: 'include',
  });

  return new ApolloClient({
    cache: new InMemoryCache(),
    link:
      typeof window === 'undefined'
        ? from([
            // SSRMultipartLink streams chunks for React 19 streaming SSR
            new SSRMultipartLink({ stripDefer: true }),
            httpLink,
          ])
        : httpLink,
  });
}

export function ApolloWrapper({ children }: { children: React.ReactNode }) {
  return <ApolloNextAppProvider makeClient={makeClient}>{children}</ApolloNextAppProvider>;
}
```

```tsx
// app/layout.tsx
import { ApolloWrapper } from '@/lib/apollo-client';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <ApolloWrapper>{children}</ApolloWrapper>
      </body>
    </html>
  );
}
```

### Server Components — different client

For Server Components, use the dedicated server-side client:

```ts
// lib/apollo-rsc-client.ts
import 'server-only';
import { HttpLink } from '@apollo/client';
import { registerApolloClient, ApolloClient, InMemoryCache } from '@apollo/client-integration-nextjs';

export const { getClient, query, PreloadQuery } = registerApolloClient(() =>
  new ApolloClient({
    cache: new InMemoryCache(),
    link: new HttpLink({
      uri: process.env.GRAPHQL_URL!,
      credentials: 'include',
    }),
  })
);
```

```tsx
// app/projects/page.tsx (Server Component)
import { query } from '@/lib/apollo-rsc-client';
import { gql } from '@apollo/client';

const PROJECTS_QUERY = gql`
  query Projects {
    projects { id name }
  }
`;

export default async function ProjectsPage() {
  const { data } = await query({ query: PROJECTS_QUERY });
  return <ul>{data.projects.map((p: any) => <li key={p.id}>{p.name}</li>)}</ul>;
}
```

`registerApolloClient` deduplicates the client per request — no leaked state across users on the server.

### Reading and writing — `useQuery` / `useMutation`

```tsx
// app/projects/CreateProjectForm.tsx
'use client';
import { gql, useMutation } from '@apollo/client';
import { useState } from 'react';

const CREATE_PROJECT = gql`
  mutation CreateProject($name: String!) {
    createProject(name: $name) { id name createdAt }
  }
`;

const PROJECTS_QUERY = gql`
  query Projects { projects { id name } }
`;

export function CreateProjectForm() {
  const [name, setName] = useState('');
  const [create, { loading, error }] = useMutation(CREATE_PROJECT, {
    update(cache, { data: { createProject } }) {
      const existing: any = cache.readQuery({ query: PROJECTS_QUERY });
      cache.writeQuery({
        query: PROJECTS_QUERY,
        data: { projects: [...(existing?.projects ?? []), createProject] },
      });
    },
  });

  return (
    <form onSubmit={(e) => { e.preventDefault(); create({ variables: { name } }); setName(''); }}>
      <input value={name} onChange={(e) => setName(e.target.value)} disabled={loading} />
      {error && <p>{error.message}</p>}
    </form>
  );
}
```

The `update` callback writes the new project into the existing `Projects` query cache so the list updates without a refetch.

### Apollo's killer feature: `cache.modify` for surgical updates

For complex caches (project with many tasks, paginated lists), `cache.modify` is the cleanest API:

```ts
update(cache, { data: { addTask } }) {
  cache.modify({
    id: cache.identify({ __typename: 'Project', id: addTask.projectId }),
    fields: {
      tasks(existing = []) {
        const newRef = cache.writeFragment({
          data: addTask,
          fragment: gql`fragment NewTask on Task { id title done }`,
        });
        return [...existing, newRef];
      },
    },
  });
}
```

This is more verbose than tRPC's `utils.X.invalidate()`, but it's **deterministic** — no refetch, no flicker, no race condition. Apollo's normalized cache (every entity stored once by `__typename:id`) is what makes this work.

---

## 4. urql 4 — The Lightweight Alternative

### What

`urql@4` is a smaller, more explicit GraphQL client built by the Formidable team. Default exchanges (Apollo's "links" equivalent) are minimal; you opt into normalized caching via `@urql/exchange-graphcache@7`.

### Setup

```bash
npm i urql@4 @urql/core@5 graphql@16
npm i @urql/exchange-graphcache@7      # only if you want normalized cache
```

```tsx
// lib/urql-client.tsx
'use client';
import { Client, fetchExchange, Provider, cacheExchange } from 'urql';

const client = new Client({
  url: process.env.NEXT_PUBLIC_GRAPHQL_URL!,
  exchanges: [cacheExchange, fetchExchange],
  fetchOptions: { credentials: 'include' },
});

export function UrqlProvider({ children }: { children: React.ReactNode }) {
  return <Provider value={client}>{children}</Provider>;
}
```

### Use

```tsx
'use client';
import { gql, useQuery } from 'urql';

const PROJECTS = gql`
  query Projects { projects { id name } }
`;

export function ProjectList() {
  const [{ data, fetching, error }] = useQuery({ query: PROJECTS });
  if (fetching) return <p>Loading…</p>;
  if (error) return <p>{error.message}</p>;
  return <ul>{data.projects.map((p: any) => <li key={p.id}>{p.name}</li>)}</ul>;
}
```

### Why urql wins

- **Smaller bundle** (~7 KB without `graphcache`, ~25 KB with vs Apollo's ~40 KB).
- **Simpler mental model** — exchanges are just middleware, easier to compose than Apollo's `link` chain.
- **Explicit caching choice** — start with document cache (default), opt into normalized only when needed.
- **First-class file uploads** via `urql-multipart-fetch-exchange@5`.

### Why urql loses to Apollo for big teams

- Smaller community = fewer Stack Overflow answers.
- Less mature Server Component story (urql still mostly client-driven).
- No first-class Federation client.
- Codegen support exists but is less polished.

For a 5-engineer team shipping a SaaS: **urql is fine**. For a 50-engineer team where ecosystem and hiring matter: **Apollo wins**.

---

## 5. Relay 18 — Meta's Approach

### What

`relay@18` (specifically `relay-runtime@18` + `react-relay@18` + `relay-compiler@18`) is Meta's GraphQL client. It's used in production at Facebook, Instagram, and WhatsApp Web. The architecture is **fragment-first** — every component declares the fields it needs as a fragment; the parent composes them and fires one query.

```graphql
# components/UserCard.tsx — fragment
fragment UserCard_user on User {
  id
  name
  avatar { url }
}
```

```tsx
// pages/profile.tsx
const ProfileQuery = graphql`
  query ProfileQuery($id: ID!) {
    user(id: $id) {
      ...UserCard_user
      posts(first: 10) {
        edges {
          node {
            ...PostCard_post
          }
        }
      }
    }
  }
`;
```

Each component reads its own fragment via `useFragment`, never the whole user object. **At Meta scale, this prevents over-fetching at component boundaries**.

### Why Relay wins

- **Best perceived performance**: `relay-compiler` precompiles queries; runtime is tiny and fast.
- **Strict normalized cache** with sub-fragment granularity — re-renders only when *that component's* fields change.
- **Connection pagination spec** baked in (cursor-based, the spec everyone else follows).
- **Strict GraphQL schema discipline** — you can't write a query that breaks the rules.

### Why Relay is hard to adopt

- The compiler is a build step.
- The fragment-driven model is unfamiliar to most React engineers.
- Errors are confusing if you don't know Relay's mental model.
- Setup cost is days, not hours.
- Documentation, while improved, is still denser than Apollo's.

For an at-Meta-scale React team: Relay 18 is the right call. **For everyone else**: Apollo or urql.

---

## 6. GraphQL Codegen — Required for Production

Writing typed React hooks by hand is the failure mode of every GraphQL adoption. Use **`@graphql-codegen/cli@5`** with the **client-preset** to generate them automatically.

### Setup

```bash
npm i -D @graphql-codegen/cli@5 @graphql-codegen/client-preset@4
```

```ts
// codegen.ts
import type { CodegenConfig } from '@graphql-codegen/cli';

const config: CodegenConfig = {
  schema: process.env.GRAPHQL_URL ?? 'http://localhost:4000/graphql',
  documents: ['src/**/*.{ts,tsx}', '!src/**/*.generated.ts'],
  generates: {
    './src/__generated__/': {
      preset: 'client',
      plugins: [],
      presetConfig: {
        gqlTagName: 'gql',                          // matches the import below
        fragmentMasking: { unmaskFunctionName: 'getFragmentData' },
      },
    },
  },
  ignoreNoDocuments: true,
};

export default config;
```

```jsonc
// package.json
{
  "scripts": {
    "codegen": "graphql-codegen",
    "codegen:watch": "graphql-codegen --watch"
  }
}
```

Run `npm run codegen:watch` alongside `next dev`. Every time you write `gql\`...\`` in a `.tsx` file, the generated types update.

### Use the typed `gql`

```tsx
'use client';
import { useQuery } from '@apollo/client';
import { gql } from '@/__generated__';                // ← from generated types, NOT from @apollo/client

const PROJECTS_QUERY = gql(/* GraphQL */ `
  query Projects {
    projects { id name }
  }
`);

export function ProjectList() {
  const { data, loading } = useQuery(PROJECTS_QUERY);  // data is fully typed
  if (loading) return null;
  return <ul>{data?.projects.map((p) => <li key={p.id}>{p.name}</li>)}</ul>;
}
```

`gql` from `@/__generated__` returns a typed `TypedDocumentNode<TResult, TVariables>`. Apollo's `useQuery` infers both. **No more `data: any`.**

### Fragment masking

The client-preset enables **fragment masking** — components can only see fields they explicitly declared in their own fragment:

```tsx
// components/UserAvatar.tsx
import { FragmentType, getFragmentData, gql } from '@/__generated__';

const UserAvatarFragment = gql(/* GraphQL */ `
  fragment UserAvatar on User {
    name
    avatar { url }
  }
`);

export function UserAvatar({ user }: { user: FragmentType<typeof UserAvatarFragment> }) {
  const u = getFragmentData(UserAvatarFragment, user);
  return <img src={u.avatar.url} alt={u.name} />;
}
```

Parent that composes:

```tsx
const ProfileQuery = gql(/* GraphQL */ `
  query Profile {
    me {
      id
      ...UserAvatar
    }
  }
`);

const { data } = useQuery(ProfileQuery);
return data?.me ? <UserAvatar user={data.me} /> : null;
```

The parent doesn't accidentally read `me.avatar` directly — TypeScript blocks it. **Components stay encapsulated; refactors stay safe**. This is Relay's model retrofitted onto Apollo + urql.

---

## 7. Subscriptions — Real-Time

GraphQL subscriptions are typically delivered over **WebSocket** (`graphql-ws@5`) or **SSE** (`graphql-sse@2.5`).

### Apollo + `graphql-ws`

```bash
npm i graphql-ws@5
```

```ts
// lib/apollo-client.ts (extending the earlier example)
import { GraphQLWsLink } from '@apollo/client/link/subscriptions';
import { createClient } from 'graphql-ws';
import { split, HttpLink } from '@apollo/client';
import { getMainDefinition } from '@apollo/client/utilities';

const httpLink = new HttpLink({ uri: process.env.NEXT_PUBLIC_GRAPHQL_URL });

const wsLink =
  typeof window !== 'undefined'
    ? new GraphQLWsLink(
        createClient({
          url: process.env.NEXT_PUBLIC_GRAPHQL_WS_URL!,
          connectionParams: { authToken: getCookie('session') },
        })
      )
    : null;

const splitLink = wsLink
  ? split(
      ({ query }) => {
        const def = getMainDefinition(query);
        return def.kind === 'OperationDefinition' && def.operation === 'subscription';
      },
      wsLink,
      httpLink
    )
  : httpLink;
```

### Use a subscription

```tsx
'use client';
import { useSubscription } from '@apollo/client';
import { gql } from '@/__generated__';

const NEW_NOTIFICATION = gql(/* GraphQL */ `
  subscription OnNewNotification {
    notificationAdded {
      id
      message
      createdAt
    }
  }
`);

export function NotificationToaster() {
  const { data } = useSubscription(NEW_NOTIFICATION);
  return data ? <div role="status">{data.notificationAdded.message}</div> : null;
}
```

### When SSE wins over WS for GraphQL

The same trade-off as `13.service-stack/01-trpc-deep.md`:

- **SSE** (`graphql-sse@2.5`): server → client only; works through corporate proxies; same HTTP infrastructure.
- **WebSocket** (`graphql-ws@5`): bi-directional; lower per-message overhead at high frequency.

For 95% of "live notifications / live counter" use cases: **SSE is enough**.

For collaborative editing / multiplayer cursors: **don't use GraphQL subscriptions** — use **Liveblocks 2 / PartyKit** (covered in `09.beyond-web/03-realtime.md`). GraphQL subscriptions don't compete on CRDT collab.

---

## 8. Server-Side: GraphQL Yoga 5 + Pothos 4

For the actual GraphQL server in 2026, the most-used Node-side combo is:

| Tool | Purpose |
|------|---------|
| **GraphQL Yoga 5** (`graphql-yoga@5`) | The HTTP server that wraps your schema |
| **Pothos 4** (`@pothos/core@4`) | Code-first schema builder; type-safe; replaces SDL-first SchemaBuilder |
| **DataLoader 2** (`dataloader@2`) | N+1 query batching — **mandatory** at any scale |
| **GraphQL Armor** (`@escape.tech/graphql-armor@3`) | Security: max-depth, max-cost, disable introspection in prod |
| **Apollo Server 4** (alt) (`@apollo/server@4`) | The Apollo team's server — pick if you also need Federation |

```bash
npm i graphql-yoga@5 @pothos/core@4 dataloader@2 graphql@16
```

```ts
// server/schema.ts
import SchemaBuilder from '@pothos/core';
import { db } from '@/server/db';

const builder = new SchemaBuilder<{ Context: { userId: string | null } }>({});

builder.objectType('Project', {
  fields: (t) => ({
    id: t.exposeID('id'),
    name: t.exposeString('name'),
    tasks: t.field({
      type: ['Task'],
      resolve: (parent) => db.task.findMany({ where: { projectId: parent.id } }),
    }),
  }),
});

builder.queryType({
  fields: (t) => ({
    projects: t.field({
      type: ['Project'],
      authScopes: { user: true },
      resolve: (_, _args, ctx) => db.project.findMany({ where: { ownerId: ctx.userId! } }),
    }),
  }),
});

builder.mutationType({
  fields: (t) => ({
    createProject: t.field({
      type: 'Project',
      args: { name: t.arg.string({ required: true }) },
      resolve: (_, { name }, ctx) => db.project.create({ data: { name, ownerId: ctx.userId! } }),
    }),
  }),
});

export const schema = builder.toSchema();
```

```ts
// app/api/graphql/route.ts
import { createYoga } from 'graphql-yoga';
import { schema } from '@/server/schema';
import { auth } from '@/auth';

const yoga = createYoga({
  schema,
  graphqlEndpoint: '/api/graphql',
  fetchAPI: { Response, Request },
  context: async () => {
    const session = await auth();
    return { userId: session?.user?.id ?? null };
  },
});

export { yoga as GET, yoga as POST };
```

`graphql-yoga@5` runs on Vercel Edge or Node serverless; mount as a Route Handler (covered in `07.nextjs/06-api-routes.md`).

### N+1 — DataLoader is mandatory

```ts
// server/loaders.ts
import DataLoader from 'dataloader';
import { db } from './db';

export const userLoader = new DataLoader(async (ids: readonly string[]) => {
  const users = await db.user.findMany({ where: { id: { in: ids as string[] } } });
  return ids.map((id) => users.find((u) => u.id === id) ?? new Error(`User ${id} not found`));
});
```

Without DataLoader, a query like `projects { tasks { author { name } } }` fires:
- 1 query for `projects` (10 rows)
- 10 queries for `tasks` (one per project)
- 100 queries for `author` (one per task)

= 111 queries.

With DataLoader: **3 queries** (projects, tasks-by-project-id, users-by-id). Real production cost difference: ~30× faster, ~10× cheaper DB.

DataLoader is opt-in per loader; create a fresh set per request inside `context`.

---

## 9. Federation — Multi-Backend GraphQL

If you have 5+ backend teams each owning their own service, **GraphQL Federation v2** lets each team ship a **subgraph** that the **Apollo Router** stitches into one schema.

```
Client → Apollo Router → ┌→ Users subgraph (Node + Pothos)
                         ├→ Billing subgraph (Go + gqlgen)
                         ├→ Inventory subgraph (Python + Strawberry)
                         └→ Search subgraph (Rust + async-graphql)
```

Each subgraph extends shared types via `@key` directives:

```graphql
# Users subgraph
type User @key(fields: "id") {
  id: ID!
  name: String!
}

# Billing subgraph
type User @key(fields: "id") {
  id: ID! @external
  invoices: [Invoice!]!
}
```

The Router resolves a query like `query { user(id: "1") { name invoices { amount } } }` by calling Users for `name` and Billing for `invoices`, then merging.

When Federation pays off:
- **5+ teams** with clear domain boundaries.
- **Backwards-compat upgrade path** from a monolithic GraphQL server.
- **Polyglot backend** — Go + Rust + Node + Python services.

When it's overkill:
- Single team, single Node server. Just use Yoga + Pothos directly.

For most startups: **don't federate**. Let the schema grow inside one service until it's clearly painful, then split.

---

## 10. Real-World Patterns

### Optimistic mutation

```tsx
'use client';
import { useMutation } from '@apollo/client';
import { gql } from '@/__generated__';

const TOGGLE_TASK = gql(/* GraphQL */ `
  mutation ToggleTask($id: ID!) {
    toggleTask(id: $id) { id done }
  }
`);

export function TaskRow({ task }: { task: { id: string; done: boolean; title: string } }) {
  const [toggle] = useMutation(TOGGLE_TASK, {
    optimisticResponse: ({ id }) => ({
      __typename: 'Mutation',
      toggleTask: { __typename: 'Task', id, done: !task.done },
    }),
  });

  return (
    <label>
      <input
        type="checkbox"
        checked={task.done}
        onChange={() => toggle({ variables: { id: task.id } })}
      />
      {task.title}
    </label>
  );
}
```

`optimisticResponse` writes the predicted result into the cache immediately. Apollo's normalized cache merges by `__typename:id`, so any other component reading this task updates instantly. The real response replaces the optimistic one when it arrives.

### Pagination — Relay-style cursor connections

```graphql
type Query {
  posts(first: Int, after: String): PostConnection!
}

type PostConnection {
  edges: [PostEdge!]!
  pageInfo: PageInfo!
}

type PostEdge {
  node: Post!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  endCursor: String
}
```

```tsx
const POSTS = gql(/* GraphQL */ `
  query Posts($first: Int!, $after: String) {
    posts(first: $first, after: $after) {
      edges { node { id title } }
      pageInfo { hasNextPage endCursor }
    }
  }
`);

const { data, fetchMore } = useQuery(POSTS, { variables: { first: 20 } });

const loadMore = () =>
  fetchMore({
    variables: { after: data?.posts.pageInfo.endCursor },
  });
```

Apollo's `relayStylePagination` field policy auto-merges consecutive pages:

```ts
const cache = new InMemoryCache({
  typePolicies: {
    Query: {
      fields: {
        posts: relayStylePagination(),
      },
    },
  },
});
```

Now `useQuery(POSTS)` always returns the **merged** list across all loaded pages. Used by every Relay-style infinite scroll.

### File uploads

GraphQL Multipart Spec lets you POST files alongside variables. **Don't.** Same advice as in `13.service-stack/01-trpc-deep.md`: use **direct-to-S3/R2 with signed URLs** (covered in `11.capabilities/02-file-uploads.md`). The GraphQL mutation just creates the metadata row and returns a presigned URL.

```graphql
type Mutation {
  createUploadUrl(filename: String!, contentType: String!): UploadInstructions!
}

type UploadInstructions {
  uploadUrl: String!
  publicUrl: String!
  uploadId: ID!
}
```

The browser PUTs the file to `uploadUrl` directly. GraphQL only carries metadata.

---

## 11. Authentication

### Cookie-based session — most common

```ts
// In Apollo Client
const httpLink = new HttpLink({
  uri: '/api/graphql',
  credentials: 'include',                          // forwards cookies
});
```

The server reads cookies in its context creator:

```ts
context: async ({ request }) => {
  const cookie = request.headers.get('cookie');
  const session = await auth(); // Auth.js v5 (covered in 07.nextjs/04-auth.md)
  return { userId: session?.user?.id ?? null };
}
```

### Bearer-token (mobile, partner API)

```ts
import { setContext } from '@apollo/client/link/context';

const authLink = setContext((_, { headers }) => {
  const token = getAuthToken();
  return { headers: { ...headers, authorization: token ? `Bearer ${token}` : '' } };
});

const client = new ApolloClient({
  link: authLink.concat(httpLink),
  cache: new InMemoryCache(),
});
```

### Field-level authorization with Pothos

```ts
builder.objectType('Project', {
  fields: (t) => ({
    name: t.exposeString('name'),
    revenue: t.field({
      type: 'Float',
      authScopes: { admin: true },                 // only visible to admins
      resolve: (p) => p.revenue,
    }),
  }),
});
```

Non-admins querying `revenue` get `null` (or an error, configurable) — server controls visibility, not the client. **GraphQL's killer feature for shared schemas across roles.**

---

## 12. Caching Strategies

### Apollo's normalized cache — the default

Every entity is stored once by `__typename:id`. `User:1` exists in one place; `me { name }` and `users { name }` both reference it. Updates propagate everywhere.

```ts
const cache = new InMemoryCache({
  typePolicies: {
    User: { keyFields: ['id'] },                    // default
    Project: {
      fields: {
        tasks: { merge: false },                    // replace, don't merge
      },
    },
  },
});
```

### Per-query fetch policy

```ts
useQuery(POSTS, {
  fetchPolicy: 'cache-and-network',                // show cache instantly, fetch fresh in background
});
```

Common policies:
- `cache-first` (default) — cache hit returns immediately; only fetch on miss.
- `cache-and-network` — return cache, also fetch fresh, update when arrived.
- `network-only` — always fetch.
- `no-cache` — fetch and never write to cache.
- `cache-only` — never fetch (offline-first).

### urql's two cache modes

- **Document cache** (default, ~7 KB) — caches whole responses by query+variables. Simple. Doesn't auto-update across queries.
- **`@urql/exchange-graphcache@7`** (~18 KB) — normalized cache like Apollo. Opt-in.

For SPAs that don't share entities across many components: document cache is fine. For Apollo-like cache invalidation: graphcache.

---

## 13. Testing GraphQL Code

### MSW (the same library used in `06.testing-perf/01-testing.md`)

```ts
import { graphql, HttpResponse } from 'msw';
import { setupServer } from 'msw/node';

const handlers = [
  graphql.query('Projects', () =>
    HttpResponse.json({ data: { projects: [{ id: '1', name: 'Test' }] } })
  ),
  graphql.mutation('CreateProject', () =>
    HttpResponse.json({ data: { createProject: { id: '2', name: 'New' } } })
  ),
];

export const server = setupServer(...handlers);
```

MSW intercepts at the network layer; your component code runs unchanged.

### Apollo's `MockedProvider`

```tsx
import { MockedProvider } from '@apollo/client/testing';

const mocks = [
  {
    request: { query: PROJECTS_QUERY },
    result: { data: { projects: [{ id: '1', name: 'Test' }] } },
  },
];

render(
  <MockedProvider mocks={mocks} addTypename={false}>
    <ProjectList />
  </MockedProvider>
);
```

Pure in-process mocking — no network layer involved. Faster but only works for Apollo. **MSW is more reusable** since it handles tRPC, REST, and GraphQL with the same setup.

---

## 14. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Cache doesn't update after mutation | Either `update` callback, `refetchQueries`, or `optimisticResponse` + `update` |
| Same `__typename` but different ID structures | Define `keyFields` per type in `typePolicies` |
| Normalized cache stores entities you didn't intend | Apollo always normalizes; use `keyFields: false` to disable per type |
| N+1 queries server-side | Use DataLoader 2; create per-request, not module-level |
| Subscription disconnects on Vercel | Vercel functions don't support long-lived WS; use a long-lived host (Railway/Fly — `07.nextjs/05-deployment.md`) or SSE |
| Client bundle balloons with `graphql` package | Tree-shake; remove unused exports; consider `graphql-tag/loader` for compile-time stripping |
| Codegen fails on dev because schema URL unreachable | Use a local schema file: `schema: './schema.graphql'` |
| Generated types out of sync | Run codegen on save with `--watch`; CI step that fails if regenerated files differ |
| Optimistic update flickers | `optimisticResponse` must include all fields the cache reads |
| Subscriptions fire but state doesn't update | The subscription's data shape must match the cache normalization key |
| Public API exposes secrets via introspection | Disable introspection in production: `validationRules: [NoSchemaIntrospectionCustomRule]` |
| Server crashes on huge queries | Use `@escape.tech/graphql-armor@3` for max-depth + max-cost limits |
| Federated schema build fails | Composition errors mean two subgraphs disagree; use `rover supergraph compose` to validate |
| RSC client leaks data across users | Use `registerApolloClient` to dedupe per request |
| Non-null fields silently turn `null` | Server returned an error mid-resolution; check error array, not just `data` |

---

## 15. Decision Tree

```
Should you use GraphQL?
│
├── Single team, internal API → use tRPC 11 instead (13.service-stack/01)
├── Public REST API → use Route Handlers (07.nextjs/06-api-routes)
├── Multiple frontends sharing one backend → ✅ GraphQL
├── Polyglot backend with multiple teams → ✅ GraphQL + Federation
└── Deeply nested relational reads with field-level auth → ✅ GraphQL

Picking a client?
│
├── Default → @apollo/client@3.11 (largest ecosystem, best Next.js integration)
├── Smaller bundle, simpler mental model → urql@4
├── Meta-scale apps, fragment-driven discipline → relay@18
└── Just a few queries → @tanstack/react-query@5 + graphql-request@7

Picking a server?
│
├── Default → graphql-yoga@5 + @pothos/core@4 (code-first, type-safe)
├── Need Federation → @apollo/server@4 + Apollo Router
├── Backend isn't Node → use the language's GraphQL framework (Strawberry, gqlgen, async-graphql)
└── Expose existing REST API as GraphQL → don't; let consumers call REST directly

Real-time?
│
├── Server → client only (notifications, status) → graphql-sse@2.5
├── Bi-directional, low latency → graphql-ws@5 over WebSocket
└── Collaborative editing → don't use GraphQL; use PartyKit/Liveblocks (09.beyond-web/03)

Codegen?
│
├── Always. Use @graphql-codegen/cli@5 with client-preset.
└── No exceptions. Hand-typed hooks rot.
```

---

## 16. What This Topic Connects To

- **`13.service-stack/01-trpc-deep.md`** — when to pick tRPC over GraphQL.
- **`07.nextjs/06-api-routes.md`** — Yoga as a Route Handler.
- **`07.nextjs/04-auth.md`** — `auth()` powers the GraphQL context.
- **`11.capabilities/02-file-uploads.md`** — why GraphQL isn't for file uploads.
- **`09.beyond-web/03-realtime.md`** — when subscriptions vs PartyKit/Liveblocks.
- **`06.testing-perf/01-testing.md`** — MSW for GraphQL mocks.
- **`10.production/04-observability.md`** — Sentry tags per resolver path.

---

## 17. Summary

| Tool | Pick when |
|------|-----------|
| `@apollo/client@3.11` + codegen client-preset | Default for most React + Next.js GraphQL apps in 2026 |
| `urql@4` + `@urql/exchange-graphcache@7` | Smaller bundle, simpler mental model |
| `relay@18` + `relay-compiler@18` | Meta scale; fragment-driven discipline |
| `graphql-yoga@5` + `@pothos/core@4` | Default Node server in 2026 |
| Apollo Federation v2 + Apollo Router | Multi-team, polyglot backend |
| `graphql-ws@5` / `graphql-sse@2.5` | Real-time subscriptions |
| `dataloader@2` | Mandatory N+1 mitigation |
| `@graphql-codegen/cli@5` + `client-preset` | Always — typed hooks from schema |
| `@escape.tech/graphql-armor@3` | Production security: max-depth, max-cost |

| Rule | Why |
|------|-----|
| Don't use GraphQL for an internal-only app — use tRPC | Less complexity for the same type safety |
| Always run codegen with `client-preset` | Hand-typed queries rot fast |
| Always use DataLoader server-side | N+1 is the silent killer |
| Use `cache-and-network` for "fresh" lists | Instant cache hit + background refresh |
| Disable introspection in production | Prevents schema scraping |
| Don't push file uploads through GraphQL | Use signed URLs (covered in `11.capabilities/02-file-uploads.md`) |
| Don't try to do CRDT collab over subscriptions | Use PartyKit / Liveblocks instead |
| Use `__typename: id` normalized caching | The killer feature; Apollo + urql graphcache enable it |

---

## Further reading

- [Apollo Client docs](https://www.apollographql.com/docs/react/)
- [Apollo Next.js integration](https://github.com/apollographql/apollo-client-nextjs)
- [urql docs](https://commerce.nearform.com/open-source/urql/)
- [Relay docs](https://relay.dev/docs/)
- [GraphQL Yoga docs](https://the-guild.dev/graphql/yoga-server)
- [Pothos docs](https://pothos-graphql.dev/)
- [GraphQL Codegen docs](https://the-guild.dev/graphql/codegen)
- [Apollo Federation v2](https://www.apollographql.com/docs/federation/)
- [DataLoader](https://github.com/graphql/dataloader)
- [GraphQL Armor](https://escape.tech/graphql-armor/)
