# Service Stack — 04. Search: Algolia vs Typesense vs MeiliSearch vs Postgres Full-Text

> **What / Why / How** — start with **Postgres full-text + `pg_trgm`** for < 10K records. Move to **MeiliSearch** when you need typo-tolerance + facets. Pick **Typesense** when you need analytics + faceted filtering at scale. Pick **Algolia** when you need polished UX + the largest ecosystem and can afford it. Index **on event**, never inline in a request.

---

## 1. The Real Choices in 2026

| Tool | Self-hosted? | Best for | Pricing |
|------|--------------|----------|---------|
| **Postgres full-text + `pg_trgm`** | Yes (you already have Postgres) | < 10K records; simple needs; no extra infra | Free |
| **MeiliSearch 1.10** | Yes / SaaS (Meilisearch Cloud) | Indie SaaS; typo-tolerance; OSS; small ops surface | Free self-host; SaaS from $0 |
| **Typesense 27** | Yes / SaaS (Typesense Cloud) | OSS; faceted filtering; fast; good at 100K–10M records | Free self-host; SaaS from $19/mo |
| **Algolia v5** | No (managed only) | Polished UX; typo-tolerance; analytics; legal-grade SLAs | $0.50/1K searches + records; expensive at scale |
| **ElasticSearch 9 / OpenSearch 2.16** | Yes / SaaS | Logs + search at scale; complex full-text | Free self-host; SaaS via Elastic Cloud |
| **PGroonga** (Postgres extension) | Yes | Postgres + good Asian-language search | Free |
| **`pg_search` (ParadeDB)** | Yes | Postgres + BM25 ranking like ES | Free |
| **`tsv` materialized columns + `pg_trgm` GIN** | Yes | Same as Postgres FTS but more idiomatic | Free |
| **Vector search (RAG)** | Yes / SaaS | Semantic / AI search | Covered separately in `09.beyond-web/04-ai-integration.md` |

**The 2026 default decision tree**:
- Single-tenant SaaS, < 10K records, simple needs → **Postgres full-text** (`tsvector` + `pg_trgm`).
- 10K–10M records, want typo-tolerance + facets, OSS-friendly → **MeiliSearch 1.10** or **Typesense 27**.
- > 10M records, complex relevance tuning, multi-region, polished UI → **Algolia v5**.
- Already on Elasticsearch → keep it; otherwise don't pick ES for new product search.

For the typical "I have a project page with a search box" feature: **Postgres FTS for v1, MeiliSearch when v1 stops scaling**.

---

## 2. Postgres Full-Text — The Free Default

### What you get out of the box

Postgres has built-in full-text search via `tsvector` and `tsquery` types. Combined with the **`pg_trgm`** extension for fuzzy matching, it covers a remarkable amount of ground for free.

```sql
-- Enable extensions (one-time)
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- Schema with full-text + fuzzy indexes
ALTER TABLE projects
  ADD COLUMN search_vector tsvector GENERATED ALWAYS AS (
    setweight(to_tsvector('english', coalesce(name, '')), 'A') ||
    setweight(to_tsvector('english', coalesce(description, '')), 'B')
  ) STORED;

CREATE INDEX projects_search_idx ON projects USING GIN (search_vector);
CREATE INDEX projects_name_trgm_idx ON projects USING GIN (name gin_trgm_ops);
```

The `GENERATED ALWAYS AS ... STORED` keeps the `search_vector` automatically in sync with `name` and `description`. **No application code needed to maintain the index.**

### Querying

```sql
-- Full-text query — fast, exact-word match with stemming
SELECT id, name, ts_rank(search_vector, query) AS rank
FROM projects, plainto_tsquery('english', 'task management') query
WHERE search_vector @@ query
ORDER BY rank DESC
LIMIT 20;

-- Trigram fuzzy match — handles typos
SELECT id, name, similarity(name, 'tsk manager') AS sim
FROM projects
WHERE name % 'tsk manager'                          -- % is the trigram operator
ORDER BY sim DESC
LIMIT 20;

-- Combine — full-text for ranked match, trigram fallback for typos
SELECT id, name,
  ts_rank(search_vector, query) AS fts_rank,
  similarity(name, 'tsk') AS trgm_sim
FROM projects, plainto_tsquery('english', 'tsk') query
WHERE search_vector @@ query OR name % 'tsk'
ORDER BY (ts_rank(search_vector, query) + similarity(name, 'tsk')) DESC
LIMIT 20;
```

### Wiring through Prisma

Prisma 5 has limited tsvector support; use raw SQL via `$queryRaw`:

```ts
// server/search.ts
import { db } from './db';

export async function searchProjects(q: string, ownerId: string) {
  return db.$queryRaw<Array<{ id: string; name: string; rank: number }>>`
    SELECT id, name,
      ts_rank(search_vector, query) +
      coalesce(similarity(name, ${q}), 0) AS rank
    FROM projects, plainto_tsquery('english', ${q}) query
    WHERE owner_id = ${ownerId}
      AND (search_vector @@ query OR name % ${q})
    ORDER BY rank DESC
    LIMIT 20;
  `;
}
```

Combine with **tRPC** (covered in `13.service-stack/01-trpc-deep.md`) or a Server Action (covered in `07.nextjs/03-data-fetching.md`).

### Why Postgres FTS is enough for most apps

- **Zero new infrastructure** — your DB already runs.
- **Transactional consistency** — search results match what's written in the same transaction; no async sync delay.
- **Joins for free** — you can filter search results by user role, tenant, status without leaving SQL.
- **Multi-tenancy is trivial** — `WHERE owner_id = ...` in the same query.
- **Fuzzy + ranked + faceted** all expressible.

### When Postgres FTS hits its limits

| Symptom | What to do |
|---------|------------|
| Search becomes the slowest endpoint at 50K+ records | Look at `EXPLAIN ANALYZE`; tune indexes; or switch to MeiliSearch |
| Need typo tolerance beyond what trigrams handle | MeiliSearch / Typesense / Algolia |
| Need faceted filtering at high cardinality (1000+ tags) | MeiliSearch / Typesense |
| Need sub-50ms search latency at scale | Dedicated search engine |
| Need synonyms, custom relevance, instant search | Algolia / MeiliSearch / Typesense |
| Need typo-tolerance in non-English (CJK, Arabic) | PGroonga, MeiliSearch, or Algolia |

For the Task Manager capstone (covered in `08.ecosystem/04-real-project.md`): Postgres FTS is the right v1.

---

## 3. MeiliSearch 1.10 — The OSS Sweet Spot

### What

`meilisearch@1.10` is an open-source search engine written in Rust. Single binary, easy ops. Indexes documents (JSON), supports typo-tolerance, faceted filtering, and instant-search latency (< 50ms typical).

### Setup — self-host

```bash
docker run -p 7700:7700 -v $(pwd)/meili_data:/meili_data getmeili/meilisearch:v1.10 \
  meilisearch --master-key="your-master-key"
```

Or use **Meilisearch Cloud** ($0–$80/mo for typical SaaS scale).

### Index from your app

```bash
npm i meilisearch@0.45
```

```ts
// lib/search.ts
import { MeiliSearch } from 'meilisearch';

export const meili = new MeiliSearch({
  host: process.env.MEILI_HOST!,
  apiKey: process.env.MEILI_ADMIN_KEY!,
});

const projects = meili.index('projects');

await projects.updateSettings({
  searchableAttributes: ['name', 'description'],    // priority order
  filterableAttributes: ['ownerId', 'status', 'createdAt'],
  sortableAttributes: ['createdAt', 'updatedAt'],
  typoTolerance: { enabled: true, minWordSizeForTypos: { oneTypo: 4, twoTypos: 8 } },
});
```

### Index on event (NEVER inline)

```ts
// inngest/functions/index-project.ts (covered in 13.service-stack/03-background-jobs.md)
import { inngest } from '@/inngest/client';
import { meili } from '@/lib/search';

export const indexProject = inngest.createFunction(
  { id: 'index-project' },
  { event: 'project.changed' },
  async ({ event, step }) => {
    const project = await step.run('load', () =>
      db.project.findUniqueOrThrow({ where: { id: event.data.projectId } })
    );
    await step.run('index', () =>
      meili.index('projects').addDocuments([{
        id: project.id,
        name: project.name,
        description: project.description,
        ownerId: project.ownerId,
        status: project.status,
        createdAt: project.createdAt.getTime(),
      }])
    );
  }
);
```

After every project create/update/delete, your app sends `inngest.send({ name: 'project.changed', data: { projectId } })`. The Inngest function reads the latest version from the DB and updates Meili. **Search is eventually consistent** — usually within seconds — but never blocks the user's mutation.

### Searching

```ts
// server/routers/search.ts (tRPC procedure)
import { protectedProcedure } from '../trpc';
import { z } from 'zod';
import { meili } from '@/lib/search';

export const searchRouter = router({
  projects: protectedProcedure
    .input(z.object({ q: z.string(), limit: z.number().default(20) }))
    .query(async ({ ctx, input }) => {
      const result = await meili.index('projects').search(input.q, {
        filter: `ownerId = "${ctx.user.id}"`,        // multi-tenancy: filter by owner
        limit: input.limit,
        attributesToHighlight: ['name', 'description'],
        attributesToCrop: ['description'],
        cropLength: 200,
      });
      return result.hits;
    }),
});
```

`filter: 'ownerId = "..."'` makes Meili respect tenant boundaries. **Critical**: don't expose the master key to the client — use **tenant tokens**:

```ts
const token = meili.generateTenantToken({
  apiKey: process.env.MEILI_SEARCH_KEY!,
  searchRules: { projects: { filter: `ownerId = "${user.id}"` } },
  expiresAt: new Date(Date.now() + 60 * 60 * 1000),     // 1 hour
});
```

The tenant token can only search the user's own projects. Pass to a `<MeiliSearch>` React component in the browser; users never see other users' data even if they manipulate requests.

### Why MeiliSearch wins for indie SaaS

- **OSS + free** — self-host on a $5 Hetzner VPS handles 1M docs.
- **Single binary** — no JVM, no cluster, no operator.
- **Typo-tolerance** out of the box.
- **Instant search** UI library: `instant-meilisearch@0.21` adapter for `react-instantsearch@7` (Algolia's UI).
- **Migrating from Algolia** is straightforward — the adapter speaks Algolia's API.

### Why MeiliSearch can lose

- Single-node by default; clustering is a paid Cloud feature.
- Less mature analytics dashboard than Algolia.
- Larger memory footprint than Typesense at the same scale.

---

## 4. Typesense 27 — The Other OSS Option

### What

`typesense@1` (server v27) is another OSS search engine. Similar feature set to MeiliSearch with different trade-offs: multi-node clustering OSS, slightly better at faceted aggregations, lower memory usage.

```bash
docker run -p 8108:8108 -v $(pwd)/typesense-data:/data typesense/typesense:27.0 \
  --data-dir /data --api-key=your-key
```

```bash
npm i typesense@1.7
```

```ts
import Typesense from 'typesense';

const ts = new Typesense.Client({
  nodes: [{ host: 'localhost', port: 8108, protocol: 'http' }],
  apiKey: process.env.TYPESENSE_KEY!,
});

await ts.collections().create({
  name: 'projects',
  fields: [
    { name: 'name', type: 'string' },
    { name: 'description', type: 'string' },
    { name: 'ownerId', type: 'string', facet: true },
    { name: 'createdAt', type: 'int64' },
  ],
  default_sorting_field: 'createdAt',
});

await ts.collections('projects').documents().import([{ id: '1', name: 'Test', ... }]);
```

### MeiliSearch vs Typesense — practical differences

| | MeiliSearch 1.10 | Typesense 27 |
|--|-------------------|--------------|
| Clustering OSS | ❌ (paid Cloud) | ✅ |
| Default focus | Typo-tolerance, instant UX | Faceted filtering, analytics |
| Memory at 1M docs | ~3 GB | ~1.5 GB |
| Algolia API compat (for migration) | Yes (`instant-meilisearch`) | Yes (`typesense-instantsearch-adapter@2`) |
| Hybrid search (vector + keyword) | Yes (1.10+) | Yes |
| GraphQL API | No | No |
| Default best at | Indie SaaS, < 10M docs | Mid-scale, faceted catalogs |

For 95% of teams: **either works**. Pick the one whose docs feel clearer to your team.

For **e-commerce with heavy facets** (color × size × brand × price-range): **Typesense** has the edge.

For **Notion-style "search across all my notes"**: **MeiliSearch** has the more polished defaults.

---

## 5. Algolia v5 — When Polish + Ecosystem Matter

### What

`algoliasearch@5` + `react-instantsearch@7` is the premium hosted search service. Built by the same Algolia that's been doing this since 2012; the UX libraries are battle-tested and cover dozens of edge cases (debouncing, highlighting, autocomplete, geolocation, federated search, A/B tests on relevance).

```bash
npm i algoliasearch@5 react-instantsearch@7 react-instantsearch-nextjs@0.4
```

```tsx
'use client';
import { liteClient as algoliasearch } from 'algoliasearch/lite';
import { InstantSearch, SearchBox, Hits, Configure } from 'react-instantsearch';

const client = algoliasearch(process.env.NEXT_PUBLIC_ALGOLIA_APP_ID!, process.env.NEXT_PUBLIC_ALGOLIA_SEARCH_KEY!);

export function Search({ userId }: { userId: string }) {
  return (
    <InstantSearch indexName="projects" searchClient={client}>
      <Configure filters={`ownerId:${userId}`} />
      <SearchBox placeholder="Search projects…" />
      <Hits hitComponent={({ hit }) => <li>{hit.name}</li>} />
    </InstantSearch>
  );
}
```

The `<SearchBox>` debounces, highlights matches, handles typos, paginates — all out of the box.

### Why Algolia wins

- **Best React UI library** — `react-instantsearch@7` has dozens of pre-built widgets.
- **Polished UX defaults** — typo-tolerance, instant results, autocomplete, search analytics.
- **Multi-region by default** — sub-100ms latency globally.
- **A/B testing on relevance** built in.
- **Click + conversion analytics** that feed back into ranking.
- **Federated search** across multiple indexes in one query.

### Why Algolia loses

- **Cost grows fast** — $0.50 per 1K searches + $0.50 per 1K records/month. For 1M searches/month + 100K records: ~$550/mo. At 10× scale: thousands per month.
- **Vendor lock-in** — migrating off Algolia (to Meili/Typesense) is non-trivial despite their adapters.
- **Index must be public** — search keys are exposed to the browser; you mitigate with **secured API keys** but it's an extra step.

For startups < 50K MAU: **MeiliSearch is dramatically cheaper for similar UX**. Algolia earns its price tag at higher scale or when relevance tuning + analytics + multi-region are worth real money.

### Secured API keys for multi-tenancy

```ts
// server/algolia.ts
import { algoliasearch } from 'algoliasearch';

const admin = algoliasearch(process.env.ALGOLIA_APP_ID!, process.env.ALGOLIA_ADMIN_KEY!);

export function generateUserSearchKey(userId: string): string {
  return admin.generateSecuredApiKey({
    parentApiKey: process.env.ALGOLIA_SEARCH_KEY!,
    restrictions: {
      filters: `ownerId:${userId}`,                  // user can only search their own
      validUntil: Math.floor(Date.now() / 1000) + 3600,  // 1 hour
    },
  });
}
```

The server signs a key scoped to the user's records. The browser uses that scoped key. Other users' data is unreachable even with manipulated requests.

---

## 6. The "Index on Event" Pattern

The single most important pattern across all search engines: **never index inline in a request**.

```ts
// ❌ Slow + flaky — search service down → user's mutation fails
export async function POST(req: Request) {
  const project = await db.project.create({ data });
  await meili.index('projects').addDocuments([project]);   // 500ms; flaky network
  return Response.json(project);
}

// ✅ Decoupled — user's mutation completes; index updates async
export async function POST(req: Request) {
  const project = await db.project.create({ data });
  await inngest.send({ name: 'project.changed', data: { projectId: project.id } });
  return Response.json(project);
}
```

Pair with **Inngest** (covered in `13.service-stack/03-background-jobs.md`) or any other queue. Failures retry; user latency stays low.

### The outbox pattern — guaranteeing eventual consistency

```ts
await db.$transaction(async (tx) => {
  const project = await tx.project.create({ data });
  await tx.outbox.create({ data: { event: 'project.changed', payload: { projectId: project.id } } });
});

// A separate worker reads from outbox and dispatches to Inngest / Meili.
```

The outbox row is committed in the same transaction as the project. If the transaction rolls back, no event is emitted. **Guarantees the search index never sees a phantom project that doesn't exist in the DB.**

For most apps, simpler `inngest.send` after a successful `db.project.create` is fine. The outbox pattern matters when **dropping events is unacceptable** (audit, billing, compliance).

### Bulk reindex — when the schema changes

When you change the indexed fields (add a column, change tokenization), reindex everything:

```ts
inngest.createFunction(
  { id: 'reindex-projects', concurrency: { limit: 1 } },          // run one at a time
  { event: 'reindex.projects' },
  async ({ step }) => {
    let cursor: string | null = null;
    while (true) {
      const projects = await step.run(`load-${cursor}`, () =>
        db.project.findMany({ take: 1000, cursor: cursor ? { id: cursor } : undefined, orderBy: { id: 'asc' } })
      );
      if (projects.length === 0) break;
      await step.run(`index-${projects[0].id}`, () =>
        meili.index('projects').addDocuments(projects)
      );
      cursor = projects[projects.length - 1].id;
    }
  }
);
```

Pull in batches of 1000; index each batch. **Scales to millions of records without timing out** because each `step.run` is its own checkpointed unit.

---

## 7. Hybrid Search — Keyword + Vector

Modern search increasingly blends **keyword search** ("project management tool") with **vector search** ("something to organize my work"). Both MeiliSearch 1.10 and Typesense 27 support hybrid mode.

For RAG-style "find documents semantically similar to this question" search, **vector** is the right approach (covered in `09.beyond-web/04-ai-integration.md`).

For "find documents matching these keywords with typo tolerance," **keyword + trigram** wins.

For **product search**: hybrid usually beats either alone. A typical config:

```ts
// MeiliSearch 1.10 hybrid
await meili.index('projects').search(q, {
  hybrid: {
    embedder: 'openai',                              // configured via Meili UI
    semanticRatio: 0.5,                              // 50/50 keyword + vector
  },
});
```

Real launch path: **start keyword-only**. Add hybrid only when keyword search hits clear quality limits (users searching by intent, not exact words).

---

## 8. Real-World Patterns

### Pattern A — Search-as-you-type with React

```tsx
'use client';
import { useState } from 'react';
import { useDebouncedValue } from '@/hooks/use-debounced-value';
import { trpc } from '@/trpc/client';

export function ProjectSearch() {
  const [q, setQ] = useState('');
  const debounced = useDebouncedValue(q, 200);
  const { data } = trpc.search.projects.useQuery(
    { q: debounced },
    { enabled: debounced.length > 1 }
  );

  return (
    <>
      <input value={q} onChange={(e) => setQ(e.target.value)} />
      <ul>{data?.map((p: any) => <li key={p.id}>{p.name}</li>)}</ul>
    </>
  );
}
```

200ms debounce balances responsiveness against API hits. `enabled: q.length > 1` skips empty/single-char searches.

For Algolia: replace with `<SearchBox>` + `<Hits>` from `react-instantsearch@7` — less code, more polish.

### Pattern B — Faceted filter UI

```tsx
'use client';
import { useState } from 'react';

const STATUSES = ['open', 'in_progress', 'done'] as const;

export function FilteredProjectSearch() {
  const [q, setQ] = useState('');
  const [status, setStatus] = useState<typeof STATUSES[number] | 'all'>('all');

  const { data } = trpc.search.projects.useQuery({
    q,
    filter: status === 'all' ? undefined : { status },
  });

  return (
    <>
      <input value={q} onChange={(e) => setQ(e.target.value)} />
      <select value={status} onChange={(e) => setStatus(e.target.value as any)}>
        <option value="all">All</option>
        {STATUSES.map((s) => <option key={s} value={s}>{s}</option>)}
      </select>
      <ul>{data?.map((p: any) => <li key={p.id}>{p.name}</li>)}</ul>
    </>
  );
}
```

The tRPC procedure (server-side) maps `filter.status` to the engine's filter syntax (`status = "open"` for Meili, `filters: 'status:open'` for Algolia).

### Pattern C — Highlighting matches

Both Algolia and Meili return highlight markup:

```ts
const result = await meili.index('projects').search(q, {
  attributesToHighlight: ['name', 'description'],
  highlightPreTag: '<mark>',
  highlightPostTag: '</mark>',
});
// result.hits[0]._formatted.name === "<mark>React</mark> task manager"
```

Render with `dangerouslySetInnerHTML` (after sanitizing if the source can contain HTML — covered in `10.production/01-rich-text-editors.md`):

```tsx
<span dangerouslySetInnerHTML={{ __html: hit._formatted.name }} />
```

### Pattern D — Suggested completions

```ts
// MeiliSearch 1.10 has facet-distribution; combine with prefix to suggest
const suggestions = await meili.index('projects').search(q, {
  limit: 5,
  attributesToRetrieve: ['name'],
});
```

For dedicated typeahead at scale: Algolia's `Autocomplete.js@1` library + `react-instantsearch@7` Combobox patterns. For Meili: build it yourself with TanStack Query 5 + a debounced query.

---

## 9. Evals + Monitoring

Search quality is measurable. Real production teams track:

| Metric | What it tells you |
|--------|-------------------|
| **Click-through rate (CTR) per query** | Are top results relevant? |
| **No-results rate** | What words are users searching that find nothing? Add synonyms or content |
| **Time to first click** | Is the UX fast enough? |
| **Search-to-conversion** | Do searched products get added to cart? |
| **Long queries** (> 5 words) | Often a sign users couldn't find what they wanted via short queries |

Algolia has all of this built in. MeiliSearch and Typesense expose query logs you can ship to your analytics layer (covered in `10.production/04-observability.md`).

### "No results" is your roadmap

The single highest-leverage optimization: **review your top "no-results" queries weekly**. They tell you:
- Missing content (write it).
- Synonyms to add (`'iphone' → 'mobile'`).
- Typos in your data (a product name has a typo — fix it).
- Customer needs you don't yet serve (great product signal).

A SaaS that ignores no-results loses ~10–30% of search-driven activation. A team that mines them weekly stays competitive.

---

## 10. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Indexing inline in the request | Move to a queue (Inngest / BullMQ / QStash) |
| Search results show deleted records | Index deletes too: `inngest.send({ name: 'project.changed', data: { id, action: 'delete' } })` |
| Master/admin key exposed in browser | Use search-only keys (Algolia) or tenant tokens (Meili) |
| Search service down → app breaks | Catch errors in your search route; fall back to "search unavailable, try again" |
| Eventual consistency confuses users | Tell them ("Indexing...") OR optimistically inject the just-mutated row in the UI |
| `pg_trgm` indexes balloon disk | Limit to indexed columns; check `pg_stat_user_indexes` for unused indexes |
| MeiliSearch out of memory | Reduce `searchableAttributes`; raise machine size; or move to disk-based |
| Algolia bill spikes after a viral campaign | Set hard limits on records + searches via Algolia's quota controls |
| Tenant filter forgotten in one query | Always use a wrapper function: `searchWithTenancy(user, query)` |
| Reindex scripts run twice in parallel | `concurrency: { limit: 1 }` in Inngest; or DB lock |
| Updates lost during deploy | Use the outbox pattern; or queue at the engine level (Meili task queue) |
| Synonyms break tests after dictionary update | Version your search settings; review changes via PR like code |
| Search latency P99 spikes during reindex | Use Algolia's atomic reindex (replicas); Meili's task queue; or reindex during off-peak |

---

## 11. Decision Tree

```
What scale and features?
│
├── < 10K records, simple needs, single-tenant
│   → Postgres FTS + pg_trgm (free, no infra)
│
├── 10K–10M records, indie SaaS, OSS-friendly
│   ├── Want typo-tolerance + clean defaults → MeiliSearch 1.10
│   └── Want multi-node OSS + heavy faceting → Typesense 27
│
├── > 10M records OR want polished UX + analytics + global low latency
│   → Algolia v5
│
└── Existing Elastic shop → keep it; otherwise don't pick ES for new product search

Indexing strategy?
│
├── On every mutation, in the same request
│   → ❌ Don't. Slow. Flaky.
│
├── Async via Inngest event
│   → ✅ Default. Eventually consistent within seconds.
│
└── Outbox pattern (transactional)
   → ✅ When dropping events is unacceptable

Multi-tenancy?
│
├── Postgres → WHERE owner_id = ... (free)
├── MeiliSearch → tenant token with filter
├── Typesense → scoped API key
└── Algolia → secured API key

Hybrid keyword + vector?
│
├── Start keyword-only → covers 90%
└── Add hybrid when measurable wins (semantic queries on long content)
```

---

## 12. What This Topic Connects To

- **`07.nextjs/06-api-routes.md`** — search exposed as a tRPC query or Route Handler.
- **`13.service-stack/03-background-jobs.md`** — index-on-event via Inngest.
- **`09.beyond-web/04-ai-integration.md`** — vector + RAG for semantic search; pgvector pairs with Postgres FTS for hybrid.
- **`10.production/04-observability.md`** — track search no-results, latency, CTR.
- **`07.nextjs/05-deployment.md`** — where to host MeiliSearch / Typesense (Railway, Fly).
- **`08.ecosystem/04-real-project.md`** — adding search to the Task Manager is a Postgres-FTS-first feature.

---

## 13. Summary

| Tool | Pick when |
|------|-----------|
| Postgres FTS + `pg_trgm` | Default for v1; no extra infrastructure |
| `meilisearch@1.10` | Indie SaaS, OSS, 10K–10M records, typo tolerance |
| `typesense@27` | OSS, multi-node, faceted filtering at mid scale |
| Algolia v5 | Polished UX, analytics, > 10M records, global latency |
| ElasticSearch / OpenSearch | Existing logs + search shop |
| Vector search (pgvector / Pinecone) | Semantic / RAG (covered separately) |

| Rule | Why |
|------|-----|
| Always index on event, never inline | User latency + reliability |
| Always scope queries by tenant | Filter at the search engine, not after |
| Use the outbox pattern when events MUST emit | Transactional consistency with the DB |
| Review "no-results" weekly | Free product roadmap |
| Cap searchable attributes | Index size + memory grow with each one |
| Start keyword-only; add hybrid when measurable | Vector search is more expensive and slower |
| Don't expose admin / master keys | Use scoped tokens / signed keys |

---

## Further reading

- [Postgres docs — Full-text search](https://www.postgresql.org/docs/current/textsearch.html)
- [`pg_trgm` docs](https://www.postgresql.org/docs/current/pgtrgm.html)
- [MeiliSearch docs](https://www.meilisearch.com/docs)
- [Typesense docs](https://typesense.org/docs/)
- [Algolia docs](https://www.algolia.com/doc/)
- [`react-instantsearch@7`](https://www.algolia.com/doc/guides/building-search-ui/what-is-instantsearch/react/)
- [`instant-meilisearch`](https://github.com/meilisearch/instant-meilisearch) — Algolia UI on Meili backend
- [PGroonga (Asian-language search)](https://pgroonga.github.io/)
- [ParadeDB / `pg_search` (BM25 in Postgres)](https://www.paradedb.com/)
