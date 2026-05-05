# Beyond the Web — 03. Real-Time: PartyKit vs Liveblocks 2 vs Supabase Realtime vs Socket.IO 4

> **What / Why / How** — pick by what you're actually building: presence + cursors (Liveblocks), CRDT collab (PartyKit + Yjs), DB-driven dashboards (Supabase), or low-level rooms (Socket.IO).

---

## 1. The Real Choices in 2026

| Tool | npm / SDK | Best for | Hosting model | Pricing model |
|------|-----------|----------|---------------|---------------|
| **Liveblocks 2** | `@liveblocks/client@2`, `@liveblocks/react@2`, `@liveblocks/yjs@2` | Multiplayer cursors, presence, collaborative editors, comments | Managed (`liveblocks.io`) | Per-MAU, free tier 1k MAU |
| **PartyKit** | `partykit@0.x` + `partysocket@1` | CRDT collab (Yjs), low-level rooms, anything that needs a small server | Cloudflare Workers (managed) or self-hosted on Workers | Generous free tier on Cloudflare; self-host is free |
| **Supabase Realtime** | `@supabase/supabase-js@2` + `@supabase/realtime-js@2` | Postgres-row-change subscriptions, broadcast channels, presence | Managed Supabase | Free tier 200 concurrent, $25+/mo |
| **Socket.IO 4** | `socket.io@4` server, `socket.io-client@4` | Low-level WebSocket rooms, custom protocols, you run your own server | Self-host on any Node host | $0 (you pay for the host) |
| **Pusher Channels** | `pusher@5` server, `pusher-js@8` client | Pub/sub, simple presence | Managed | Per-message |
| **Ably** | `ably@2` | Pub/sub with delta compression, regional routing | Managed | Per-message |
| **AWS AppSync (GraphQL subscriptions)** | `@aws-amplify/api@6` | Existing AWS shop | Managed AWS | AWS pricing |
| **WebSocket directly via Hono 4 / Bun / Cloudflare Workers** | Built-in `WebSocket` API | DIY everything | Self-host | $0 |

**The 80% rule for new React apps in 2026:**
- Need cursors + presence + comments + named features? → **Liveblocks 2**.
- Need CRDT collab (Yjs/Tiptap-style document editing)? → **PartyKit + Yjs**.
- Already on Supabase Postgres? → **Supabase Realtime**.
- Need maximum control on your own infra? → **Socket.IO 4** or raw WebSockets in Hono on Bun/Workers.

---

## 2. The First Question — What Real-Time Pattern?

Real-time is not one thing. The four real patterns:

| Pattern | What it looks like | Examples |
|---------|---------------------|----------|
| **Pub/sub broadcast** | Server pushes a message; many clients receive | Stock ticker, score update, "someone published a new post" |
| **Presence** | A list of who's currently in a room/page, with optional metadata (cursor, avatar, status) | Figma's cursor list, Notion's "viewing now" |
| **Document collaboration (CRDT)** | Multiple clients edit the same document; conflicts auto-merge | Google Docs, Notion pages, Tiptap collaborative editor |
| **Database-driven** | Client subscribes to row changes; server pushes rows when they change | Live dashboards, Trello-style boards |

Each tool below is strongest at a different one of these. Picking the right tool is mostly picking the right pattern first.

---

## 3. Liveblocks 2 — Polished Multiplayer Out of the Box

### What

`@liveblocks/react@2` is a managed-service SDK for **rooms with presence, storage, comments, and Yjs-based document collaboration**. Built by the team that previously made [Liveblocks for Figma plugins](https://liveblocks.io/blog).

Setup:

```bash
npm i @liveblocks/client@2 @liveblocks/react@2
```

```ts
// liveblocks.config.ts
import { createClient } from '@liveblocks/client';
import { createRoomContext } from '@liveblocks/react';

const client = createClient({
  authEndpoint: '/api/liveblocks-auth', // your auth endpoint (next route handler)
});

type Presence = {
  cursor: { x: number; y: number } | null;
  selectedId: string | null;
};

type Storage = { /* persistent shared state */ };

export const {
  RoomProvider,
  useOthers, useMyPresence, useUpdateMyPresence,
  useStorage, useMutation, useEventListener,
} = createRoomContext<Presence, Storage>(client);
```

```tsx
// app/board/[id]/Board.tsx
'use client';
import { RoomProvider, useOthers, useUpdateMyPresence } from '@/liveblocks.config';

export function Board({ id }: { id: string }) {
  return (
    <RoomProvider id={`board-${id}`} initialPresence={{ cursor: null, selectedId: null }}>
      <Cursors />
      <Canvas />
    </RoomProvider>
  );
}

function Cursors() {
  const others = useOthers();
  return (
    <>
      {others.map(({ connectionId, presence }) =>
        presence.cursor ? (
          <div
            key={connectionId}
            style={{ position: 'absolute', left: presence.cursor.x, top: presence.cursor.y }}
          >
            ●
          </div>
        ) : null
      )}
    </>
  );
}

function Canvas() {
  const updateMyPresence = useUpdateMyPresence();
  return (
    <div
      onPointerMove={(e) => updateMyPresence({ cursor: { x: e.clientX, y: e.clientY } })}
      style={{ width: '100vw', height: '100vh' }}
    />
  );
}
```

That's a complete multiplayer cursor implementation in ~30 lines. Liveblocks handles the websocket, server-side state, presence diffing, and reconnection.

### Auth endpoint (Route Handler — covered in `07.nextjs/06-api-routes.md`)

```ts
// app/api/liveblocks-auth/route.ts
import { Liveblocks } from '@liveblocks/node';
import { auth } from '@/auth';

const liveblocks = new Liveblocks({ secret: process.env.LIVEBLOCKS_SECRET_KEY! });

export async function POST(req: Request) {
  const session = await auth();
  if (!session?.user) return new Response('Unauthorized', { status: 401 });

  const { room } = await req.json();
  const lb = liveblocks.prepareSession(session.user.id, {
    userInfo: { name: session.user.name!, avatar: session.user.image },
  });
  lb.allow(room, lb.FULL_ACCESS);
  return new Response(await lb.authorize(), { status: 200 });
}
```

The user-id-aware token means every connection is tied to your real auth user (covered in `07.nextjs/04-auth.md`).

### Why Liveblocks

- **Polished primitives**: cursors, presence, storage, comments (`@liveblocks/comments@2`), threads, notifications — all first-party.
- **Yjs adapter** (`@liveblocks/yjs@2`) for collaborative document editing without running your own server.
- **Battle-tested**: Linear's collaborative cursors, Vercel's preview comments, RoamHQ, Liveblocks' own demo apps.
- **DX**: hooks-first React API; minimal boilerplate.

### Why Liveblocks loses

- Per-MAU pricing scales fast at high traffic.
- Vendor lock-in: room state lives on Liveblocks' servers; migrating later means re-implementing on PartyKit/Socket.IO.

### When to pick Liveblocks

- You're building Figma-style cursors, Notion-style presence, or comment threads attached to UI elements.
- You don't want to run a websocket server.
- You're at < 50K MAU where the pricing is comfortable.

---

## 4. PartyKit + Yjs — CRDT Collaboration on Cloudflare Workers

### What

`partykit@0.x` is a tiny framework for writing **stateful WebSocket servers** that run on Cloudflare Durable Objects. Every "party" (room) is a Durable Object — a single-region serverless object with persistent state and a websocket interface.

```ts
// party/server.ts
import type * as Party from 'partykit/server';

export default class Server implements Party.Server {
  constructor(readonly room: Party.Room) {}

  onConnect(conn: Party.Connection) {
    conn.send(`Welcome to room ${this.room.id}. ${this.room.connections.size} people here.`);
    this.room.broadcast(`* ${conn.id} joined`, [conn.id]);
  }

  onMessage(message: string, sender: Party.Connection) {
    this.room.broadcast(message, [sender.id]); // exclude sender
  }
}
```

Deploy: `npx partykit deploy`. You get a live URL like `https://my-party.user.partykit.dev`.

Connect from React:

```tsx
'use client';
import usePartySocket from 'partysocket/react';
import { useState } from 'react';

function Chat({ room }: { room: string }) {
  const [messages, setMessages] = useState<string[]>([]);
  const socket = usePartySocket({
    host: 'https://my-party.user.partykit.dev',
    room,
    onMessage: (e) => setMessages(m => [...m, e.data]),
  });
  return (
    <div>
      <ul>{messages.map((m, i) => <li key={i}>{m}</li>)}</ul>
      <input
        onKeyDown={(e) => {
          if (e.key === 'Enter') {
            socket.send(e.currentTarget.value);
            e.currentTarget.value = '';
          }
        }}
      />
    </div>
  );
}
```

### Why PartyKit

- **CRDT-native**: pairs perfectly with `yjs@13` for collaborative editors. Tiptap and Lexical both have Yjs bindings; PartyKit is the most popular hosted backend for them.
- **Per-room state on the edge**: each room is its own Durable Object, runs near users, persists state to D1/R2 if you want.
- **Generous free tier**: 100K requests/day on Cloudflare.
- **Owned by Cloudflare since 2024**: stable, well-funded.

### Real example — collaborative Yjs document

```ts
// party/server.ts
import type * as Party from 'partykit/server';
import { onConnect } from 'y-partykit';

export default class Server implements Party.Server {
  constructor(readonly room: Party.Room) {}
  async onConnect(conn: Party.Connection) {
    return await onConnect(conn, this.room, {
      persist: { mode: 'snapshot' }, // save snapshots to Durable Object storage
    });
  }
}
```

```tsx
// React side — Tiptap + Yjs over PartyKit
'use client';
import { useEditor, EditorContent } from '@tiptap/react';
import StarterKit from '@tiptap/starter-kit';
import Collaboration from '@tiptap/extension-collaboration';
import * as Y from 'yjs';
import YPartyKitProvider from 'y-partykit/provider';

function CollaborativeDoc({ docId }: { docId: string }) {
  const ydoc = new Y.Doc();
  const provider = new YPartyKitProvider('localhost:1999', `doc-${docId}`, ydoc);

  const editor = useEditor({
    extensions: [
      StarterKit.configure({ history: false }), // Yjs handles history
      Collaboration.configure({ document: ydoc }),
    ],
  });

  return <EditorContent editor={editor} />;
}
```

Two browsers loading the same `docId` get full Google-Docs-style collaborative editing with conflict resolution. Total setup: one PartyKit `server.ts`, one React component.

### Why PartyKit loses

- More setup than Liveblocks for things Liveblocks gives you for free (cursors, comments, threads).
- Logic lives in your code (good for ownership, more work than Liveblocks' managed primitives).
- Smaller community than Liveblocks for "drop-in" features.

### When to pick PartyKit

- Collaborative editor (Tiptap, Lexical, ProseMirror, BlockNote) — Yjs + PartyKit is the canonical 2026 stack.
- You want to own the server logic (custom rate limits, custom presence semantics).
- Cloudflare Workers is already in your stack.

---

## 5. Supabase Realtime — When You're Already On Supabase

### What

`@supabase/realtime-js@2` (used through `@supabase/supabase-js@2`) gives you three primitives:
- **Postgres Changes** — subscribe to inserts/updates/deletes on a Postgres table.
- **Broadcast** — send a custom message to other clients in a channel.
- **Presence** — track who's connected to a channel and their metadata.

```ts
import { createClient } from '@supabase/supabase-js';

const supabase = createClient(process.env.NEXT_PUBLIC_SUPABASE_URL!, process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!);

// Subscribe to row changes
const channel = supabase.channel('public:tasks')
  .on('postgres_changes', { event: '*', schema: 'public', table: 'tasks' }, (payload) => {
    console.log('Task changed:', payload);
  })
  .subscribe();
```

```ts
// Presence
const presence = supabase.channel('room-42', {
  config: { presence: { key: userId } },
});

presence
  .on('presence', { event: 'sync' }, () => {
    const state = presence.presenceState();
    console.log('Currently online:', state);
  })
  .on('presence', { event: 'join' }, ({ key, newPresences }) => { /* ... */ })
  .subscribe(async (status) => {
    if (status === 'SUBSCRIBED') {
      await presence.track({ name: 'Anh', cursor: { x: 0, y: 0 } });
    }
  });
```

### Why Supabase Realtime

- **Free with Supabase.** No second vendor.
- **Postgres-Changes** is unique: subscribe to row events without writing publish logic. Your Postgres `INSERT` / `UPDATE` / `DELETE` becomes a real-time event automatically.
- **Same SDK** as auth, storage, RPC — one API surface.

### Why Supabase Realtime loses

- Postgres-Changes scales poorly past ~200 concurrent subscribers per channel — you'll need to denormalize or use Broadcast.
- Less polished primitives than Liveblocks for cursors/comments — you build them on top of Broadcast yourself.

### When to pick Supabase Realtime

- You already use Supabase for DB and auth.
- You want **DB row changes → UI updates** without writing a publishing layer.
- Your real-time needs are dashboard-style, not Figma-style.

---

## 6. Socket.IO 4 — The Old Faithful for Custom Servers

### What

`socket.io@4` is the classic WebSocket library: rooms, namespaces, automatic reconnect, fallback to long-polling, server-side state via your own logic.

```ts
// server.ts (Node.js)
import { Server } from 'socket.io';
import { createServer } from 'http';

const httpServer = createServer();
const io = new Server(httpServer, { cors: { origin: '*' } });

io.on('connection', (socket) => {
  socket.on('join', (room) => {
    socket.join(room);
    io.to(room).emit('user-joined', { id: socket.id });
  });
  socket.on('chat', ({ room, message }) => {
    io.to(room).emit('chat', { from: socket.id, message });
  });
  socket.on('disconnect', () => { /* ... */ });
});

httpServer.listen(3001);
```

```tsx
// React side
'use client';
import { useEffect, useState } from 'react';
import { io } from 'socket.io-client';

const socket = io('http://localhost:3001');

function ChatRoom({ room }: { room: string }) {
  const [messages, setMessages] = useState<string[]>([]);
  useEffect(() => {
    socket.emit('join', room);
    socket.on('chat', ({ message }) => setMessages(m => [...m, message]));
    return () => { socket.off('chat'); };
  }, [room]);

  // ...
}
```

### Why Socket.IO

- **Mature.** 13 years of bug fixing, every edge case discovered.
- **Fallbacks.** Auto-falls back from WebSocket to long-polling on hostile networks (corporate proxies).
- **Self-hosted.** No vendor pricing.
- **Works anywhere Node runs.** Bun, Deno (with adapters), AWS, your laptop.

### Why Socket.IO loses

- **You run a server.** That means a process, a host, scaling, monitoring, deploys.
- **State management is yours**: presence, room membership, persistence — all handcrafted.
- **Doesn't run on Vercel's Node functions.** Vercel functions are short-lived; you need a long-lived host (Railway, Fly.io, a VPS — covered in `07.nextjs/05-deployment.md`).
- **Larger client bundle** than the alternatives (~40 KB gzipped including the polling fallback).

### When to pick Socket.IO

- You already have a Node backend and want to add real-time features without picking a new vendor.
- You need **fallback to long-polling** for reliability on hostile networks.
- You want **maximum control** over the wire protocol and server logic.

---

## 7. Raw WebSockets via Hono 4 / Bun / Cloudflare — The Lightweight Option

If you don't need Socket.IO's fallbacks, native WebSocket is enough. The setup pattern with `hono@4` (covered in `07.nextjs/06-api-routes.md`):

### Bun

```ts
// server.ts
import { Hono } from 'hono';
import { upgradeWebSocket } from 'hono/bun';

const app = new Hono();

app.get('/ws', upgradeWebSocket((c) => ({
  onMessage(event, ws) { ws.send(`Echo: ${event.data}`); },
  onClose() { console.log('disconnect'); },
})));

export default app;
```

Run: `bun run server.ts`. Native WebSocket support; no library beyond Hono.

### Cloudflare Workers (with Durable Objects for state)

PartyKit is essentially a friendlier wrapper around this. If you want raw control:

```ts
export class Room {
  state: DurableObjectState;
  sockets = new Set<WebSocket>();

  constructor(state: DurableObjectState) { this.state = state; }

  async fetch(req: Request) {
    const pair = new WebSocketPair();
    const [client, server] = Object.values(pair);
    server.accept();
    this.sockets.add(server);

    server.addEventListener('message', (e) => {
      for (const ws of this.sockets) ws.send(e.data);
    });
    server.addEventListener('close', () => this.sockets.delete(server));

    return new Response(null, { status: 101, webSocket: client });
  }
}
```

Strong fit when you want **edge-region rooms with persistence** but don't need PartyKit's framework.

---

## 8. Real Side-by-Side: 50-User Cursor Tracker

The same feature implemented four ways:

| | Liveblocks 2 | PartyKit + custom | Supabase Realtime | Socket.IO |
|--|--------------|-------------------|---------------------|-----------|
| Lines of server code | 0 (managed) | ~30 (broadcast handler) | 0 (use Broadcast/Presence) | ~30 |
| Lines of client code | ~20 | ~20 | ~25 | ~25 |
| Latency at 50 connections | ~50ms (managed) | ~30ms (Cloudflare edge) | ~70ms (US-east default) | depends on host |
| Cost at 1k MAU | ~$0–$30 | ~$0 (Cloudflare free tier) | ~$0 (Supabase free tier) | ~$5/mo (host) |
| Scale ceiling on free tier | 1k MAU | 100k req/day | 200 concurrent | none (you scale your host) |
| Needs auth wiring | Yes | Yes | Yes (RLS) | Yes |
| Persistence of cursor state | Built-in (Storage) | Durable Object | Postgres row | DIY |

For cursors specifically: **Liveblocks** wins on dev speed; **PartyKit** wins on cost-at-scale.

---

## 9. Putting it Together — Real-Time Patterns in Common Apps

| App pattern | Tool |
|-------------|------|
| Notification dropdown ("you have 3 new messages") | Server-Sent Events (covered in `07.nextjs/06-api-routes.md`) or Supabase Broadcast |
| Live chat | Socket.IO 4 (self-host) or Liveblocks 2 (managed) |
| Live dashboard updating from DB | Supabase Realtime (Postgres Changes) |
| Trello / Linear-style board | Liveblocks 2 (cursors) + tRPC mutations + TanStack Query background refetch |
| Google Docs-style collaborative editor | Tiptap 2 + Yjs 13 + PartyKit |
| Multiplayer cursor on a marketing page | Liveblocks 2 (zero server logic) |
| Stock ticker / price feed | Pusher Channels or Ably (managed pub/sub) |
| Multiplayer game | Custom Socket.IO 4 server with binary protocol |
| Web push notifications (mobile-style) | `web-push@3` library + Service Worker — different problem from real-time |

---

## 10. Auth + Real-Time — The One Thing Everyone Skips

A websocket connection is **not authenticated by default**. Three real patterns:

### Pattern A — Token in connection URL

```ts
const ws = new WebSocket(`wss://api.example.com/ws?token=${jwt}`);
```

Server validates the JWT on `onConnect`. Liveblocks does this internally via `prepareSession`.

### Pattern B — Auth message after connect

```ts
ws.onopen = () => ws.send(JSON.stringify({ type: 'auth', token: jwt }));
```

Server expects an auth message; disconnects if not received within N seconds. Used by Socket.IO middleware.

### Pattern C — Cookie-based session

The browser already sends auth cookies on the handshake. Server reads `req.headers.cookie`, validates the session.

```ts
io.use((socket, next) => {
  const cookie = socket.handshake.headers.cookie;
  const session = parseCookie(cookie);
  if (!session) return next(new Error('unauthorized'));
  socket.data.userId = session.userId;
  next();
});
```

Works seamlessly with Auth.js v5 sessions (covered in `07.nextjs/04-auth.md`).

---

## 11. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Memory leak from unremoved listeners | Always remove in `useEffect` cleanup; `socket.off` matches `socket.on` |
| Reconnection storms after server restart | Use exponential backoff; PartyKit/Liveblocks handle it; Socket.IO has it built-in |
| Cursor positions update at 60fps and lag the network | Throttle with `lodash.throttle@4` or a custom 60→16fps debouncer |
| Stale presence state after tab close | Liveblocks/PartyKit detect close natively; Socket.IO needs `socket.on('disconnect', ...)` handling on the server |
| Postgres-Changes flood with 1000+ row inserts | Switch to a single batch broadcast event from the API, not row-level subscriptions |
| Liveblocks billing surprise | Check the "active connection minutes" metric, not just MAU |
| WebSocket fails on corporate proxy | Socket.IO's polling fallback is the safest; or `partysocket@1` which retries with backoff |
| Server Action triggers but UI doesn't update for OTHER users | Real-time event must be broadcast separately. Mutation → broadcast → other clients update via the channel |
| Yjs document grows huge over time | Run `Y.encodeStateAsUpdateV2` snapshots and discard old updates |
| TypeScript types of Liveblocks `useStorage` are `unknown` | Define `Storage` type in `createRoomContext<Presence, Storage>(client)` |

---

## 12. Decision Tree

```
Is the feature a multiplayer cursor / presence / comments / collaborative document?
├── Yes, and you want polished primitives, OK with managed pricing → Liveblocks 2
└── Yes, and you want CRDT (Tiptap, Lexical) on cheap edge infra → PartyKit + Yjs

Is the feature "DB row changed → push to clients"?
└── Use Supabase Realtime (Postgres Changes) if already on Supabase
   Otherwise: Server-Sent Events from a Route Handler + a publish channel (Redis)

Is it pub/sub (notifications, live numbers)?
├── Want managed, simple pricing → Pusher Channels or Ably
└── Want zero vendor → SSE from a Route Handler, or self-hosted Socket.IO

Custom multiplayer game / chat / anything bespoke?
├── Already have a Node backend → Socket.IO 4
├── Want Cloudflare edge + Durable Objects → Raw WS or PartyKit
└── Bun shop → Hono + native WebSocket

Existing AWS infrastructure → AWS AppSync subscriptions
```

---

## 13. What This Topic Connects To

- **`07.nextjs/06-api-routes.md`** — Route Handlers, Server-Sent Events, edge runtime.
- **`07.nextjs/04-auth.md`** — Auth.js sessions extend to WebSocket auth.
- **`04.state-and-data/03-data-fetching.md`** — Real-time complements TanStack Query's polling/refetch model.
- **`07.nextjs/05-deployment.md`** — Vercel doesn't host long-lived WebSocket processes; pick Railway/Fly/Workers.
- **`08.ecosystem/04-real-project.md`** — Adding presence to the Task Manager would be Liveblocks; adding collaborative description editing would be PartyKit + Yjs.

---

## Summary

| Tool | Pick when |
|------|-----------|
| **Liveblocks 2** | Cursors, presence, comments, threads, polished managed primitives |
| **PartyKit + Yjs** | CRDT collab editors (Tiptap, Lexical), Cloudflare-edge rooms |
| **Supabase Realtime** | Already on Supabase, want Postgres-Changes |
| **Socket.IO 4** | Self-host, mature library, network-fallback needed |
| **Hono + WebSocket** | Bun or Cloudflare Workers, lightweight, no library |
| **Pusher / Ably** | Pub/sub-only managed service |

| Rule | Why |
|------|-----|
| Pick by pattern (presence vs CRDT vs DB-changes vs pub/sub) | Each tool excels at one |
| Don't put long-lived WebSockets on Vercel functions | Vercel is short-lived; use Railway/Fly/Cloudflare |
| Authenticate the WebSocket connection | Or anyone can join your rooms |
| Throttle high-frequency events client-side | 60 mouse-moves per second flood the wire |
| Snapshot Yjs documents periodically | Updates accumulate forever otherwise |

---

## Further reading

- [Liveblocks docs](https://liveblocks.io/docs)
- [PartyKit docs](https://docs.partykit.io/)
- [Yjs docs](https://docs.yjs.dev/)
- [Supabase Realtime docs](https://supabase.com/docs/guides/realtime)
- [Socket.IO docs](https://socket.io/docs/v4/)
- [Hono WebSocket helper](https://hono.dev/docs/helpers/websocket)
- [Cloudflare Durable Objects + WebSockets](https://developers.cloudflare.com/durable-objects/api/websockets/)
- [Tiptap + Yjs collaboration](https://tiptap.dev/docs/editor/extensions/functionality/collaboration)
