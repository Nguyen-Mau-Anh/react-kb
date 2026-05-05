# Beyond the Web — 04. AI Integration: Vercel AI SDK 4, Streaming Chat UIs, Tool-Calling Agents, RAG

> **What / Why / How** — pick the Vercel AI SDK 4 unless you have a specific reason. Stream tokens with `useChat`. Use tool calls for agents. RAG with `pgvector` is the right default in 2026.

---

## 1. The Real Choices for AI in a React App in 2026

| Layer | Tools |
|-------|-------|
| **Provider SDK (talk to the model)** | `ai@4` (Vercel AI SDK), `@anthropic-ai/sdk@0.30`, `openai@4`, `@google/generative-ai@0.x`, `@mistralai/mistralai@1` |
| **Multi-provider abstraction** | `ai@4` (the SDK's `@ai-sdk/anthropic`, `@ai-sdk/openai`, `@ai-sdk/google` adapters) |
| **React-streaming UI hooks** | `ai/react` (built into `ai@4`): `useChat`, `useCompletion`, `useObject`, `useAssistant` |
| **Vector DB** | `pgvector` (Postgres extension), Pinecone, Weaviate, Qdrant, `@upstash/vector@1` |
| **Embeddings** | `text-embedding-3-large` (OpenAI), `voyage-3` (Voyage AI), `@anthropic-ai/sdk` doesn't ship embeddings — use Voyage |
| **Agent framework** | `ai@4` `generateText` + `tools` + `maxSteps`, `langchain@0.3`, `@mastra/core@0.x` |
| **Observability** | `helicone@2`, `langsmith@0.x`, OpenTelemetry, Vercel AI Observability |

**The 2026 default for any React + Next.js app:** `ai@4` (Vercel AI SDK) with the model adapter for whichever provider you're using. It gives you streaming, tool calls, structured output, and React hooks in one package — and switching providers is a one-line change.

---

## 2. Why the Vercel AI SDK 4

### What

`ai@4` is a TypeScript SDK that abstracts over LLM providers. You write code against its API; under the hood it calls Anthropic, OpenAI, Google, Mistral, Cohere, Groq, etc.

```bash
npm i ai@4 @ai-sdk/anthropic@1
```

```ts
import { generateText } from 'ai';
import { anthropic } from '@ai-sdk/anthropic';

const { text } = await generateText({
  model: anthropic('claude-sonnet-4-6'),
  prompt: 'Summarize the React Compiler in one paragraph.',
});
```

### Why not call the provider SDK directly

You can use `@anthropic-ai/sdk@0.30` (Anthropic) or `openai@4` directly — they're well-maintained and zero-abstraction. But:

| | Provider SDKs (`@anthropic-ai/sdk`, `openai`) | Vercel AI SDK (`ai@4`) |
|--|----------------------------------------------|-------------------------|
| Switch model providers without rewriting | ❌ | ✅ One-line change |
| Stream to React without writing SSE plumbing | ❌ | ✅ Built-in: `toDataStreamResponse()` |
| `useChat` / `useCompletion` React hooks | ❌ | ✅ |
| Structured output via Zod schemas | Partial (manual JSON-mode prompting) | ✅ `generateObject` with Zod schema |
| Tool calls with type-safe schemas | Provider-specific shapes | ✅ Unified across providers |
| Observability (`onFinish` hooks, telemetry) | DIY | Built-in |

For a Next.js app: **always use `ai@4`** unless you need a feature only the provider SDK exposes (e.g., Anthropic's prompt caching beta headers). Cost is the same — both go through the same HTTP API.

### When the provider SDK still wins

- Heavy, advanced prompt caching with full control (Anthropic's `cache_control` blocks).
- Provider-specific features still in beta (Files API, Computer Use, Code Interpreter).
- Server-only batch jobs where the React-streaming hooks aren't needed.

---

## 3. The Streaming Chat UI — `useChat`

This is the workhorse pattern. ChatGPT-style streaming token UI in ~50 lines.

### Server route — Route Handler that streams

```ts
// app/api/chat/route.ts
import { streamText } from 'ai';
import { anthropic } from '@ai-sdk/anthropic';

export const runtime = 'edge';
export const maxDuration = 60;  // seconds — covered in 07.nextjs/06

export async function POST(req: Request) {
  const { messages } = await req.json();
  const result = await streamText({
    model: anthropic('claude-sonnet-4-6'),
    system: 'You are a concise React/Next.js teaching assistant.',
    messages,
  });
  return result.toDataStreamResponse();
}
```

### Client — `useChat` from `ai/react`

```tsx
'use client';
import { useChat } from 'ai/react';

export function Chat() {
  const { messages, input, handleInputChange, handleSubmit, isLoading, stop } = useChat({
    api: '/api/chat',
  });

  return (
    <div className="mx-auto max-w-2xl">
      <div className="space-y-3">
        {messages.map(m => (
          <div key={m.id} className={m.role === 'user' ? 'text-right' : 'text-left'}>
            <span className="inline-block rounded-md bg-muted px-3 py-2">{m.content}</span>
          </div>
        ))}
      </div>

      <form onSubmit={handleSubmit} className="mt-4 flex gap-2">
        <input
          className="flex-1 rounded border px-3 py-2"
          value={input}
          placeholder="Ask anything"
          onChange={handleInputChange}
        />
        <button disabled={isLoading} className="rounded bg-blue-600 px-3 text-white">
          Send
        </button>
        {isLoading && <button type="button" onClick={stop}>Stop</button>}
      </form>
    </div>
  );
}
```

What you get for free:
- **Token-by-token streaming** rendered as the model writes.
- **Auto-scroll**, message accumulation, role tracking.
- **Stop generation** — calls `AbortController` under the hood.
- **Reload last response** via `reload()` (if you destructure it).
- **Optimistic message append** — the user's message appears instantly.

This single hook is why the Vercel AI SDK is the 2026 default. The competing path of writing your own SSE parser + state machine is ~200 lines of tedious code.

### Switching to OpenAI

```bash
npm i @ai-sdk/openai
```

```ts
import { openai } from '@ai-sdk/openai';

const result = await streamText({
  model: openai('gpt-4o-mini'),  // ← only this line changed
  messages,
});
```

The client and the rest of the server code don't change. This is the abstraction's payoff.

---

## 4. Structured Output — `generateObject` with Zod

When you don't want freeform text but a typed object back, use `generateObject`. The model is constrained to produce JSON matching a Zod schema.

```ts
import { generateObject } from 'ai';
import { anthropic } from '@ai-sdk/anthropic';
import { z } from 'zod';

const Recipe = z.object({
  name: z.string(),
  ingredients: z.array(z.object({ item: z.string(), amount: z.string() })),
  steps: z.array(z.string()),
  prepMinutes: z.number(),
});

export async function POST(req: Request) {
  const { dish } = await req.json();

  const { object } = await generateObject({
    model: anthropic('claude-sonnet-4-6'),
    schema: Recipe,
    prompt: `Generate a recipe for: ${dish}`,
  });

  return Response.json(object);  // object is fully typed as z.infer<typeof Recipe>
}
```

Use cases:
- Form auto-fill ("turn this email into a calendar event").
- Data extraction from text.
- Generating shadcn/ui form configs.
- Function-call-style outputs without using tool calls.

The corresponding React hook is `useObject`:

```tsx
'use client';
import { experimental_useObject as useObject } from 'ai/react';
import { Recipe } from '@/schemas';

function RecipeGenerator() {
  const { object, submit, isLoading } = useObject({
    api: '/api/recipe',
    schema: Recipe,
  });

  return (
    <>
      <button onClick={() => submit({ dish: 'spicy carrot soup' })}>Generate</button>
      {object?.name && <h2>{object.name}</h2>}
      {object?.ingredients?.map((ing, i) => (
        <li key={i}>{ing.amount} {ing.item}</li>
      ))}
    </>
  );
}
```

`useObject` streams **partial** objects as the model writes — every key fills in progressively. Used by Vercel's own [v0.dev](https://v0.dev) UI.

---

## 5. Tool Calls — Letting the Model Use Your Functions

Tool calling lets the model invoke functions in your code. The SDK handles the JSON Schema generation from a Zod schema, executes the tool, feeds the result back to the model, and loops until the model has its answer.

```ts
// app/api/agent/route.ts
import { streamText, tool } from 'ai';
import { anthropic } from '@ai-sdk/anthropic';
import { z } from 'zod';
import { db } from '@/server/db';

export const runtime = 'edge';

export async function POST(req: Request) {
  const { messages } = await req.json();

  const result = await streamText({
    model: anthropic('claude-sonnet-4-6'),
    maxSteps: 5,                          // up to 5 tool-use rounds before stopping
    messages,
    tools: {
      listProjects: tool({
        description: 'List all projects owned by the current user',
        parameters: z.object({}),
        execute: async () => {
          return await db.project.findMany({ where: { ownerId: currentUserId } });
        },
      }),
      createTask: tool({
        description: 'Create a task in a project',
        parameters: z.object({
          projectId: z.string().cuid(),
          title: z.string().min(1),
          description: z.string().optional(),
        }),
        execute: async ({ projectId, title, description }) => {
          const task = await db.task.create({ data: { projectId, title, description, position: 0 } });
          return { id: task.id, ok: true };
        },
      }),
    },
  });

  return result.toDataStreamResponse();
}
```

The user types "create a 'review PR' task in my Acme project". The model:

1. Calls `listProjects` → gets project IDs.
2. Picks the one named Acme.
3. Calls `createTask` with that ID and the title.
4. Streams a confirmation message back to the user.

You write zero tool-router boilerplate.

### Render tool calls in the UI

```tsx
'use client';
import { useChat } from 'ai/react';

function AgentChat() {
  const { messages } = useChat({ api: '/api/agent' });

  return (
    <div>
      {messages.map(m => (
        <div key={m.id}>
          <strong>{m.role}:</strong>
          {m.toolInvocations?.map(t => (
            <div key={t.toolCallId} className="rounded border bg-muted p-2 text-xs">
              {t.toolName}({JSON.stringify(t.args)})
              {t.state === 'result' && (
                <pre>{JSON.stringify(t.result, null, 2)}</pre>
              )}
            </div>
          ))}
          {m.content}
        </div>
      ))}
    </div>
  );
}
```

`m.toolInvocations` is the typed list of tool calls the model made for that message. You can render them as collapsible debug rows or as polished UI cards (e.g., "I created task X for you").

### Why tool calls beat raw prompting

- The schema validation guarantees you get well-formed arguments — not an LLM hallucinating field names.
- The framework loops automatically until the model is done; you don't write a "is this done?" parser.
- Multi-tool agents become tractable — each tool is one Zod schema + one function.

### When NOT to use tool calls

- Single, deterministic operation ("translate this") — just `generateText` or `generateObject`.
- Latency-critical paths — each tool round-trip is a model call (~500–1500ms each).

---

## 6. The `useAssistant` Pattern — OpenAI Assistants API

If you're using OpenAI's Assistants API (server-side state, threads, retrieval, code interpreter), `useAssistant` is the equivalent of `useChat` for that surface. Different mental model: state lives on OpenAI's servers, not yours.

In 2026, **most teams skip OpenAI Assistants** in favor of `streamText` + tools + their own DB. Reasons:
- Vendor lock-in (state on OpenAI).
- Thread persistence is hard to migrate later.
- You can replicate it in your own DB in ~50 lines.

Use it only if you need OpenAI's hosted code interpreter or vector store and don't want to run your own.

---

## 7. RAG (Retrieval-Augmented Generation) — pgvector + Postgres

The "default" 2026 RAG stack: Postgres + `pgvector` extension + an embedding model (OpenAI `text-embedding-3-large` or Voyage AI `voyage-3`).

### Schema

```prisma
// prisma/schema.prisma
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["postgresqlExtensions"]
}

datasource db {
  provider   = "postgresql"
  url        = env("DATABASE_URL")
  extensions = [pgvector(map: "vector")]
}

model Document {
  id         String                @id @default(cuid())
  content    String                @db.Text
  embedding  Unsupported("vector(1536)")?
  source     String
  createdAt  DateTime              @default(now())
}
```

### Generate embeddings on insert

```ts
// server/embed.ts
import { embed } from 'ai';
import { openai } from '@ai-sdk/openai';

export async function embedText(text: string): Promise<number[]> {
  const { embedding } = await embed({
    model: openai.embedding('text-embedding-3-large'),
    value: text,
  });
  return embedding;
}
```

```ts
// server/ingest.ts
import { db } from './db';
import { embedText } from './embed';

export async function ingest(text: string, source: string) {
  // Chunk: 500-token chunks with 50-token overlap is the standard 2026 default
  const chunks = chunkText(text, { maxTokens: 500, overlap: 50 });
  for (const chunk of chunks) {
    const embedding = await embedText(chunk);
    await db.$executeRaw`
      INSERT INTO "Document" (id, content, embedding, source)
      VALUES (${cuid()}, ${chunk}, ${`[${embedding.join(',')}]`}::vector, ${source})
    `;
  }
}
```

### Retrieve and answer

```ts
// app/api/rag/route.ts
import { streamText } from 'ai';
import { anthropic } from '@ai-sdk/anthropic';
import { db } from '@/server/db';
import { embedText } from '@/server/embed';

export async function POST(req: Request) {
  const { messages } = await req.json();
  const lastUserMessage = messages[messages.length - 1].content;

  const queryEmbedding = await embedText(lastUserMessage);

  const docs = await db.$queryRaw<{ content: string; source: string }[]>`
    SELECT content, source
    FROM "Document"
    ORDER BY embedding <=> ${`[${queryEmbedding.join(',')}]`}::vector
    LIMIT 5
  `;

  const context = docs.map(d => `<doc source="${d.source}">${d.content}</doc>`).join('\n\n');

  const result = await streamText({
    model: anthropic('claude-sonnet-4-6'),
    system: `You are a helpful assistant. Answer the user's question using ONLY the information in the <doc> tags below. If the answer is not present, say "I don't know." Cite sources by their source attribute.

${context}`,
    messages,
  });

  return result.toDataStreamResponse();
}
```

`<=>` is `pgvector`'s cosine-distance operator. `LIMIT 5` is the standard top-k.

### Why pgvector over dedicated vector DBs

| | pgvector + Postgres | Pinecone / Weaviate / Qdrant |
|--|----------------------|-------------------------------|
| Setup | Already have Postgres → `CREATE EXTENSION vector;` | New service, separate auth, separate billing |
| Joins with relational data | Native (one query) | App-layer joins |
| Cost at small scale | $0 incremental | $20–70/mo minimum |
| Performance at 100M+ vectors | Slower than dedicated DBs | Better |
| 2026 popularity | Default for most RAG apps | Niche for very large scale |

For < 10M vectors: **pgvector**. Beyond that, Pinecone or Qdrant pay off.

### Real considerations

- **Chunk size matters more than the model.** 500-token chunks with 50-token overlap covers most cases.
- **Re-embed when source documents change.** Embeddings are stale once content changes.
- **Use a reranker** (`cohere/rerank-3` via the `cohere-ai@7` SDK, or `voyage-rerank-2`) on the top-20 from pgvector → top-5 to the model. Boosts answer quality by 10–30% in real benchmarks.
- **Cache embeddings.** Don't re-embed the same text twice.

---

## 8. Caching, Cost Control, Observability

### Anthropic prompt caching

Long system prompts (RAG context, examples) can be cached for 5 minutes per cache hit, cutting cost by ~90% on cache-hit calls.

The Vercel AI SDK 4 supports it via provider-specific options:

```ts
const result = await streamText({
  model: anthropic('claude-sonnet-4-6'),
  messages,
  providerOptions: {
    anthropic: {
      cacheControl: { type: 'ephemeral' },  // applies to large system prompts
    },
  },
});
```

Use this on system prompts containing RAG context, multi-shot examples, or tool definitions that don't change per request.

### OpenAI prompt caching

Automatic for prompts > 1024 tokens, ~50% cheaper on hits. No code change.

### Observability

Track cost, latency, errors per call:

```ts
import { streamText } from 'ai';

const result = await streamText({
  model: anthropic('claude-sonnet-4-6'),
  messages,
  experimental_telemetry: { isEnabled: true },
  onFinish: ({ usage, finishReason }) => {
    log({ tokens: usage, finishReason, durationMs: ... });
  },
});
```

For real production observability:
- **Helicone** (`helicone-async@2`) — open-source, self-hostable, drop-in proxy.
- **Langsmith** (`langsmith@0.x`) — commercial, deepest tracing.
- **Vercel AI Observability** — built into Vercel hosting, zero config.

---

## 9. Models — When to Pick Which (As of 2026)

| Need | Real pick |
|------|-----------|
| Default for chat, agents, code | `claude-sonnet-4-6` (Anthropic) — strongest 2026 default for software-engineering and reasoning |
| Fastest / cheapest for routine tasks | `claude-haiku-4-5` (Anthropic) or `gpt-4o-mini` (OpenAI) |
| Largest context (1M tokens) | `gemini-2.0-pro` (Google) |
| Strongest tool calling | `claude-sonnet-4-6` or `gpt-4o` |
| Cheapest embeddings | `text-embedding-3-small` (OpenAI) |
| Best embeddings (English) | `voyage-3` (Voyage AI) — measurably better at retrieval than `text-embedding-3-large` |
| Reranking | `voyage-rerank-2` or `cohere/rerank-3` |
| Local / on-device | `llama-3.3` via `ollama@0.x` (developer machines, not production for most) |

The model-pick rule: **start with `claude-sonnet-4-6`** as the workhorse. Drop to a smaller model only when you measure that the smaller one passes your evals.

---

## 10. Real-World UI Patterns

### Generative UI (Vercel's `streamUI`)

```tsx
// app/api/inventory/route.ts
import { streamUI } from 'ai/rsc';
import { anthropic } from '@ai-sdk/anthropic';
import { z } from 'zod';
import { ProductCard } from '@/components/product-card';

export async function POST(req: Request) {
  const { prompt } = await req.json();

  const result = await streamUI({
    model: anthropic('claude-sonnet-4-6'),
    prompt,
    text: ({ content }) => <p>{content}</p>,
    tools: {
      showProduct: {
        description: 'Show a product card to the user',
        parameters: z.object({ id: z.string() }),
        generate: async function* ({ id }) {
          yield <p>Loading…</p>;
          const product = await db.product.findUnique({ where: { id } });
          return <ProductCard product={product!} />;
        },
      },
    },
  });

  return result.toDataStreamResponse();
}
```

The model can return **React components** as part of its response, not just text. Used by `v0.dev` and many shopping-assistant chat UIs.

### Streaming voice (Whisper + ElevenLabs / OpenAI realtime)

`@ai-sdk/openai`'s upcoming realtime API support handles voice-in/voice-out streaming. Until that stabilizes, the standard pattern:

1. Browser captures audio → POSTs to `/api/transcribe` → OpenAI Whisper returns text.
2. Text goes to `streamText`.
3. Response text → ElevenLabs `text-to-speech` API → audio streams back.

`@ai-sdk/openai` exposes `transcribe()` and `speech()` helpers that wrap this.

---

## 11. Auth + AI — The Real Patterns

LLM endpoints are expensive. Always:
- **Authenticate the request.** Use `auth()` from Auth.js v5 (covered in `07.nextjs/04-auth.md`).
- **Rate-limit.** Use `@upstash/ratelimit@1` (covered in `07.nextjs/06-api-routes.md`).
- **Cap output tokens.** Pass `maxTokens` to `streamText` so a hostile prompt can't burn budget.
- **Log per-user usage.** Set a daily / monthly cap; alert when a user hits 80%.

```ts
import { auth } from '@/auth';
import { Ratelimit } from '@upstash/ratelimit';
import { Redis } from '@upstash/redis';

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(20, '1 h'),
});

export async function POST(req: Request) {
  const session = await auth();
  if (!session?.user) return new Response('Unauthorized', { status: 401 });

  const { success, remaining } = await ratelimit.limit(session.user.id);
  if (!success) return new Response('Rate limited', { status: 429 });

  // ... run streamText
}
```

---

## 12. Testing AI Code

You **cannot** unit-test the model output deterministically. Two real approaches:

### Mock the model

```ts
// __tests__/ai.test.ts
import { describe, it, expect, vi } from 'vitest';
import { streamText } from 'ai';

vi.mock('ai', () => ({
  streamText: vi.fn().mockResolvedValue({
    toDataStreamResponse: () => new Response('mocked'),
  }),
}));

it('chat route returns 200', async () => {
  const res = await POST(new Request('...', { method: 'POST', body: JSON.stringify({ messages: [] }) }));
  expect(res.status).toBe(200);
});
```

### Eval against real models

For agent quality regression: build an **eval set** of input/expected-behavior pairs. Run it nightly with a real model, score with another model, alert on drops.

Tools:
- `vercel/ai-evals@0.x` — Vercel's first-party eval framework.
- `@inngest/test-engine@0.x` — eval pipelines as Inngest jobs.
- `langsmith@0.x` — full eval + tracing platform.

---

## 13. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| `useChat` doesn't stream — UI updates only at end | Server didn't return `toDataStreamResponse()`; or you returned `result.text` instead of the stream |
| `maxDuration` exceeded on Vercel | Edge functions cap at 30s on Hobby, 60–300s on Pro; use `runtime: 'edge'` and split long calls |
| Anthropic 401 in production | `ANTHROPIC_API_KEY` not set; or you set `OPENAI_API_KEY` and used the wrong adapter |
| Tool call args are `null` | Zod schema is too loose; LLM omits the field. Make it required. |
| `pgvector` query is slow | Add an index: `CREATE INDEX ON "Document" USING hnsw (embedding vector_cosine_ops);` |
| RAG returns garbage answers | Chunks are too large; retrieved doc is irrelevant. Lower chunk size and add reranker. |
| Cost spikes overnight | Forgot to rate-limit + cap `maxTokens`. Add both. |
| `useChat` messages duplicate | You're sending `messages` array AND letting `useChat` manage state. Pick one source of truth. |
| Streaming stops mid-word | Edge function timed out. Lower model size or upgrade Vercel plan. |
| Tool returns a giant payload | The model then has to read it all on the next round. Truncate or summarize before returning. |

---

## 14. Decision Tree

```
What are you building?
│
├── ChatGPT-style chat with streaming?
│   → ai@4 streamText + useChat. Default to claude-sonnet-4-6 unless cost-constrained.
│
├── Q&A over your own documents (docs site, internal wiki)?
│   → RAG: pgvector + voyage-3 embeddings + reranker + streamText
│
├── Agent that calls your code (CRUD, search, web fetch)?
│   → streamText + tools + maxSteps. Pair with TanStack Query mutation invalidation.
│
├── Form auto-fill, data extraction, structured generation?
│   → generateObject with Zod schema, or useObject for streaming partial output
│
├── Generative UI (chat that returns React components)?
│   → streamUI from ai/rsc
│
└── Voice / realtime audio?
   → @ai-sdk/openai realtime helpers, or DIY via Whisper + ElevenLabs
```

---

## 15. What This Topic Connects To

- **`07.nextjs/06-api-routes.md`** — Route Handlers, Edge runtime, streaming responses, rate limiting.
- **`07.nextjs/04-auth.md`** — `auth()` for gating model calls.
- **`04.state-and-data/03-data-fetching.md`** — TanStack Query for caching tool-call results.
- **`05.routing-and-styling/03-typescript-with-react.md`** — Zod schemas for tool parameters and structured output.
- **`08.ecosystem/04-real-project.md`** — Add an AI assistant to the Task Manager: `useChat` + tools that call the existing tRPC procedures.

---

## Summary

| Tool | Use for |
|------|---------|
| `ai@4` (Vercel AI SDK) | Default for everything in a React/Next.js app |
| `useChat` from `ai/react` | Streaming chat UIs |
| `useObject` (`experimental_useObject`) | Streaming structured generation |
| `generateObject` server-side | Single-shot structured output with Zod |
| `streamText` + `tools` + `maxSteps` | Tool-calling agents |
| `streamUI` from `ai/rsc` | Generative UI (React components from the model) |
| pgvector + Postgres | Default RAG store for < 10M vectors |
| `voyage-3` embeddings + `voyage-rerank-2` | Best-quality retrieval pipeline |
| `claude-sonnet-4-6` | Workhorse model for chat, agents, code |
| Anthropic prompt caching | Cut cost on long system prompts |
| `@upstash/ratelimit@1` | Rate-limit AI endpoints per user |

| Rule | Why |
|------|-----|
| Default to `ai@4` over provider SDKs | Switch providers without rewriting; React hooks built-in |
| Always rate-limit and cap `maxTokens` | LLM endpoints are unbounded-cost without it |
| Use Zod schemas for every tool and structured output | Catches malformed model output at the framework level |
| RAG: chunk + reranker, not just embed + top-k | Reranking boosts answer quality 10–30% |
| Don't unit-test model output deterministically | Eval suites + monitoring instead |

---

## Further reading

- [Vercel AI SDK docs](https://sdk.vercel.ai/docs)
- [`ai` package — `streamText`](https://sdk.vercel.ai/docs/ai-sdk-core/generating-text)
- [`ai` package — `generateObject`](https://sdk.vercel.ai/docs/ai-sdk-core/generating-structured-data)
- [`useChat` reference](https://sdk.vercel.ai/docs/reference/ai-sdk-ui/use-chat)
- [`streamUI` (Generative UI)](https://sdk.vercel.ai/docs/ai-sdk-rsc/streaming-react-components)
- [pgvector docs](https://github.com/pgvector/pgvector)
- [Anthropic prompt caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
- [Voyage AI docs](https://docs.voyageai.com/)
