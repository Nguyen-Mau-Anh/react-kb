# Production — 01. Rich Text Editors: Tiptap 2 vs Lexical vs Slate vs BlockNote

> **What / Why / How** — pick **Tiptap 2** by default; **Lexical** if Meta's API style fits you; **BlockNote** when you want Notion-style blocks out of the box. Skip Quill in 2026.

---

## 1. The Real Choices in 2026

| Editor | npm | Built on | Used by | Best for |
|--------|-----|----------|---------|----------|
| **Tiptap 2** | `@tiptap/react@2.5`, `@tiptap/core@2.5`, `@tiptap/starter-kit@2.5` | ProseMirror | GitLab, Hashnode, ConvertKit, Medium clones, Outline | The 2026 default — most extensions, best DX, largest community |
| **Lexical** | `lexical@0.18`, `@lexical/react@0.18` | Custom (Meta) | Facebook, Instagram, WhatsApp Web, Threads | Meta-built, very fast, plugin model unfamiliar to most React devs |
| **BlockNote** | `@blocknote/react@0.21`, `@blocknote/mantine@0.21` | Tiptap 2 + ProseMirror | Notion-style apps, AI-builder products | Drop-in Notion-style block editor with slash menu |
| **Slate 0.103** | `slate@0.103`, `slate-react@0.111` | Custom | Discord (early years), some legacy editors | Truly headless, but the API churn is exhausting |
| **Plate 41** | `@udecode/plate-common@41`, `@udecode/plate-core@41` | Slate | shadcn/ui-aligned editor projects | Slate with batteries — alternative to Tiptap if Slate fits your model |
| **TinyMCE 7 / CKEditor 5** | `@tinymce/tinymce-react@5`, `@ckeditor/ckeditor5-react@9` | Custom | Wordpress, Drupal, enterprise CMSs | Word-style toolbars, license-required for some features |
| **ProseMirror direct** | `prosemirror-state@1.4`, `prosemirror-view@1.34` | — | Power users | When you want full control and no abstraction tax |
| **Quill 2** | `quill@2` | Custom | Old Slack composer, legacy apps | Don't pick for new code |

**Default rule for 2026:**
- Building a **Notion-clone block editor**? → `@blocknote/react@0.21`.
- Anything else (article editors, comment boxes, rich form fields, document collaboration)? → `@tiptap/react@2.5`.
- Already at Meta or strong preference for their plugin style? → `lexical@0.18`.

---

## 2. The Big Mental Split — Block vs Inline vs Document

Different editors are good at different shapes of content:

| Shape | Examples | Best fit |
|-------|----------|----------|
| **Inline** (one-line, formatting only) | Comment box, tweet composer, chat messages, table cell | Tiptap 2 with minimal extensions, or `contentEditable` + your own logic |
| **Article / document** (long-form, rich formatting, images, tables) | Blog post editor, internal wiki page | Tiptap 2 with full StarterKit, or Lexical |
| **Block-based** (each paragraph is a draggable block, slash menu, transformable types) | Notion, Coda, Outline | BlockNote 0.21 (Notion-style out of the box) |
| **Code-heavy** (mostly code blocks with prose around them) | Hashnode, dev.to | Tiptap 2 + `@tiptap/extension-code-block-lowlight` |
| **Collaborative** (multiple users editing simultaneously) | Google Docs, Notion | Tiptap 2 + Yjs + PartyKit (covered in `09.beyond-web/03-realtime.md`) |

The shape determines the editor more than personal preference does. **Slack chose Lexical for their new composer** because they needed inline + collaboration. **GitLab chose Tiptap** because they needed long-form articles with extensive plugin needs. Match the editor to the shape.

---

## 3. Tiptap 2 — The Default Pick

### What

`@tiptap/react@2.5` is a React wrapper around **ProseMirror** — a battle-tested document model and rendering engine that powers GitLab, Atlassian Confluence, the New York Times CMS, and many more. Tiptap removes the ProseMirror boilerplate and gives you an extension-based API.

### Why Tiptap won default-status by 2026

- **Largest extension catalog** in the React ecosystem (~80 official + ~hundreds of community).
- **Schema is data, not classes.** Define a node by extending `Node.create({...})` — no class-hierarchy ceremony.
- **Excellent TypeScript support** in v2.5+.
- **Yjs collaboration adapter** is first-party (`@tiptap/extension-collaboration@2.5`) — drops Google-Docs-style real-time editing into any Tiptap instance with one line.
- **shadcn/ui examples** (Vercel templates, Linear's editor demos) all assume Tiptap.
- **AI extensions** ship with the SDK — `@tiptap/extension-ai@2` for inline LLM completion, summarization, refactor commands.

### Setup — minimal article editor

```bash
npm i @tiptap/react@2.5 @tiptap/pm@2.5 @tiptap/starter-kit@2.5
```

```tsx
'use client';
import { useEditor, EditorContent } from '@tiptap/react';
import StarterKit from '@tiptap/starter-kit';

export function Editor({ initial }: { initial?: string }) {
  const editor = useEditor({
    extensions: [StarterKit],
    content: initial ?? '',
    immediatelyRender: false, // ⚠ required for SSR — see §6
    editorProps: {
      attributes: {
        class: 'prose prose-slate max-w-none focus:outline-none',
      },
    },
  });

  if (!editor) return null;
  return <EditorContent editor={editor} />;
}
```

`StarterKit` bundles paragraph, heading, bold, italic, code, lists, blockquote, horizontal rule, hard break, history (undo/redo), and a few more — the 80% of features every editor needs.

### Adding features — the extension model

```bash
npm i @tiptap/extension-link@2.5 @tiptap/extension-image@2.5 @tiptap/extension-placeholder@2.5
```

```tsx
import Link from '@tiptap/extension-link';
import Image from '@tiptap/extension-image';
import Placeholder from '@tiptap/extension-placeholder';

const editor = useEditor({
  extensions: [
    StarterKit,
    Link.configure({ openOnClick: false }),
    Image,
    Placeholder.configure({ placeholder: 'Start writing…' }),
  ],
});
```

Each extension is a small TypeScript file that registers nodes, marks, commands, keyboard shortcuts, and ProseMirror plugins. You can write your own:

```ts
// extensions/MentionExtension.ts
import { Node } from '@tiptap/core';

export const Mention = Node.create({
  name: 'mention',
  inline: true,
  group: 'inline',
  selectable: false,
  atom: true,
  addAttributes: () => ({ id: { default: null }, label: { default: null } }),
  parseHTML: () => [{ tag: 'span[data-mention]' }],
  renderHTML: ({ node }) => ['span', { 'data-mention': '', class: 'rounded bg-blue-100 px-1' }, `@${node.attrs.label}`],
});
```

### A real toolbar — using editor commands

```tsx
'use client';
import { useEditor, EditorContent } from '@tiptap/react';

function Toolbar({ editor }: { editor: Editor }) {
  if (!editor) return null;
  return (
    <div className="flex gap-1 border-b p-2">
      <ToolbarButton onClick={() => editor.chain().focus().toggleBold().run()} active={editor.isActive('bold')}>B</ToolbarButton>
      <ToolbarButton onClick={() => editor.chain().focus().toggleItalic().run()} active={editor.isActive('italic')}>I</ToolbarButton>
      <ToolbarButton onClick={() => editor.chain().focus().toggleBulletList().run()} active={editor.isActive('bulletList')}>•</ToolbarButton>
      <ToolbarButton onClick={() => editor.chain().focus().toggleHeading({ level: 2 }).run()} active={editor.isActive('heading', { level: 2 })}>H2</ToolbarButton>
    </div>
  );
}
```

The `.chain().focus().toggleX().run()` pattern is Tiptap's transaction API — every command is composable, returns the editor for chaining, and is wrapped in a single ProseMirror transaction.

### Reading the value — JSON or HTML

```ts
const json = editor.getJSON();   // ProseMirror JSON — the canonical representation
const html = editor.getHTML();   // HTML string — for SSR or simple display
const text = editor.getText();   // plain text — for search indexing
```

**Always store JSON, not HTML.** JSON is structured, lossless, version-tolerant. HTML is a one-way export. You'll thank yourself in 18 months when you add a new extension and need to programmatically transform existing content.

### Collaborative editing — Tiptap + Yjs + PartyKit

```bash
npm i yjs@13 @tiptap/extension-collaboration@2.5 y-partykit@0.x
```

```tsx
import * as Y from 'yjs';
import YPartyKitProvider from 'y-partykit/provider';
import Collaboration from '@tiptap/extension-collaboration';

const ydoc = new Y.Doc();
const provider = new YPartyKitProvider('localhost:1999', `doc-${docId}`, ydoc);

const editor = useEditor({
  extensions: [
    StarterKit.configure({ history: false }),    // Yjs handles history (CRDT undo)
    Collaboration.configure({ document: ydoc }),
  ],
});
```

Two browsers loading the same `docId` get full Google-Docs-style collaborative editing. Covered in detail in `09.beyond-web/03-realtime.md`.

### Tiptap 3 — what's coming

`@tiptap/react@3` is in beta as of mid-2026. Major changes:
- React 19 ref-as-prop migration done.
- Smaller core, opt-in extensions.
- Async loading for heavy extensions (code highlighting, math).

Don't migrate yet for production apps. v2.5 is stable and well-documented; v3 has 1–2 release cycles to settle.

---

## 4. BlockNote 0.21 — Notion-Style Out of the Box

### What

`@blocknote/react@0.21` is a Notion-clone editor built on top of Tiptap 2. It ships:
- Block-based content model (every paragraph is a block).
- Slash menu (`/h2`, `/quote`, `/image`).
- Drag-handle on every block (reorder by dragging the gutter).
- Side menu with quick-format actions.
- Default styling (with Mantine 7) — drop in and it looks right.

### Setup

```bash
npm i @blocknote/react@0.21 @blocknote/mantine@0.21 @blocknote/core@0.21
```

```tsx
'use client';
import '@blocknote/core/fonts/inter.css';
import '@blocknote/mantine/style.css';
import { useCreateBlockNote } from '@blocknote/react';
import { BlockNoteView } from '@blocknote/mantine';

export function Notes() {
  const editor = useCreateBlockNote({
    initialContent: [
      { type: 'heading', props: { level: 1 }, content: 'Project notes' },
      { type: 'paragraph', content: 'Write here…' },
    ],
  });
  return <BlockNoteView editor={editor} />;
}
```

That's a complete Notion-style editor — slash menu, drag handle, polished styling — in 8 lines. AI features (summarize, rewrite, brainstorm) are a paid add-on via `@blocknote/xl-ai@0.21`.

### When BlockNote beats raw Tiptap

- You need block-level reordering and slash menu out of the box.
- You're building a Notion-style note-taking app and don't want to design the chrome.
- You want AI features without integrating Tiptap's AI SDK manually.

### When BlockNote loses

- You're building an article-style editor (paragraphs flow, no block UI). Use Tiptap directly.
- You want full design control. BlockNote ships its own styled UI; customizing past a point is fighting it.

---

## 5. Lexical — Meta's Editor Framework

### What

`lexical@0.18` is Facebook's modern editor framework. Powers Facebook's main composer, Threads, Instagram (web), and WhatsApp Web. Built from scratch — does **not** use ProseMirror.

```bash
npm i lexical@0.18 @lexical/react@0.18
```

```tsx
'use client';
import { LexicalComposer } from '@lexical/react/LexicalComposer';
import { ContentEditable } from '@lexical/react/LexicalContentEditable';
import { RichTextPlugin } from '@lexical/react/LexicalRichTextPlugin';
import { HistoryPlugin } from '@lexical/react/LexicalHistoryPlugin';
import { LexicalErrorBoundary } from '@lexical/react/LexicalErrorBoundary';

const config = {
  namespace: 'editor',
  theme: { paragraph: 'mb-2' },
  onError(err: Error) { console.error(err); },
};

export function LexicalEditor() {
  return (
    <LexicalComposer initialConfig={config}>
      <RichTextPlugin
        contentEditable={<ContentEditable className="prose focus:outline-none" />}
        placeholder={<p>Start writing…</p>}
        ErrorBoundary={LexicalErrorBoundary}
      />
      <HistoryPlugin />
    </LexicalComposer>
  );
}
```

### Why pick Lexical

- **Performance**: explicitly designed for speed; benchmarks show 30–50% faster typing latency than ProseMirror at scale.
- **Smaller core** than Tiptap (the wrapper) — but you write more glue code yourself.
- **Strong accessibility defaults** out of the box.
- **Backed by Meta** — used in their core products, so it gets attention.

### Why Lexical loses against Tiptap for most teams

- **Plugin ecosystem is smaller.** ~30 official plugins; community plugins are maybe 50. Tiptap has 80+ official and many hundreds of community.
- **API style** is closer to Slate's "operations on a node tree" than Tiptap's "extensions add capabilities." Less familiar to React devs.
- **Documentation** is technically complete but harder to skim than Tiptap's. Real engineers report a steeper learning curve.
- **Collaboration story** is via `@lexical/yjs@0.18` but less battle-tested than Tiptap's.

### When Lexical is the right pick

- Performance budget at scale (Facebook's composer pattern).
- You're at a Meta-aligned shop or have engineers who already know it.
- You need very precise control over the editor's selection/insertion lifecycle (Lexical exposes more here than Tiptap).

---

## 6. SSR & React 18+ Gotchas

Editors run on `contentEditable`, which depends on `window` and `document`. SSR is tricky.

### Tiptap

```tsx
const editor = useEditor({
  extensions: [StarterKit],
  immediatelyRender: false,  // ⚠ required for Next.js App Router SSR
});
```

Without `immediatelyRender: false`, the server tries to render the editor and produces hydration mismatches. Tiptap 2.4+ added the flag as the explicit fix.

### Lexical

`@lexical/react@0.18` runs editor logic only on the client; the wrapper handles SSR by rendering a placeholder during server render. No special config needed.

### Render strategy in Next.js 14

```tsx
// app/editor/page.tsx
import dynamic from 'next/dynamic';

const Editor = dynamic(() => import('@/components/Editor'), { ssr: false });

export default function EditorPage() {
  return <Editor />;
}
```

For editors that don't *need* SSR (write-only workspaces), `ssr: false` is the simplest path. For viewer/edit hybrids (article preview that becomes editable), use `immediatelyRender: false` (Tiptap) or Lexical's built-in SSR-safe rendering.

---

## 7. Saving — Always JSON, Always Server-Validated

### Persisting

```ts
// On every change (debounced)
import { useDebouncedCallback } from 'use-debounce'; // or your own

const save = useDebouncedCallback(async (json: JSONContent) => {
  await fetch(`/api/notes/${id}`, {
    method: 'PUT',
    body: JSON.stringify({ content: json }),
  });
}, 1000);

const editor = useEditor({
  extensions: [StarterKit],
  onUpdate: ({ editor }) => save(editor.getJSON()),
});
```

Real-world rule: **debounce saves at 1–2 seconds**. Per-keystroke saves overwhelm your DB and provide no UX benefit (the user will keep typing anyway).

### Schema validation on the server

Trusting raw editor JSON from the client is a real XSS surface. Sanitize / validate:

```ts
// app/api/notes/[id]/route.ts
import { z } from 'zod';

const NodeSchema: z.ZodType = z.lazy(() =>
  z.object({
    type: z.string(),
    attrs: z.record(z.unknown()).optional(),
    content: z.array(NodeSchema).optional(),
    text: z.string().optional(),
    marks: z.array(z.object({ type: z.string() })).optional(),
  })
);

const Schema = z.object({ content: NodeSchema });

export async function PUT(req: Request, { params }: { params: { id: string } }) {
  const parsed = Schema.parse(await req.json());
  // ... persist
}
```

For **HTML rendering** of stored content, use `dompurify@3` (`isomorphic-dompurify@2` for SSR) on output. Tiptap's `generateHTML` from `@tiptap/html@2.5` is safe for the JSON it produces, but if you're displaying HTML from any external source, sanitize.

---

## 8. Output Strategies — JSON vs HTML vs Markdown

| Format | Stores well? | Renders well? | Searchable? | Real use |
|--------|---------------|---------------|-------------|----------|
| **JSON (ProseMirror schema)** | ✅ Best — structured, lossless | Need to convert to HTML for display | Need to extract text first | The default. Store this. |
| **HTML** | ⚠ Lossy when you re-import | ✅ Direct render | Awkward — strip tags | Use only for display caches or one-way exports |
| **Markdown** | ⚠ Loses non-Markdown features | Good for static rendering | ✅ Naturally indexable | Use for code-flavored editors (GitLab, Hashnode) where Markdown is the user's mental model |

**Real production pattern:** store JSON; generate an HTML cache on save for fast public rendering; strip-text the JSON for full-text search indexing.

```ts
import { generateHTML } from '@tiptap/html';
import StarterKit from '@tiptap/starter-kit';

const html = generateHTML(json, [StarterKit]);
const text = html.replace(/<[^>]+>/g, ''); // for search index
```

---

## 9. Extension Builder's Cheat Sheet

When you need a custom feature (mention, embed, AI button, custom block), Tiptap's three primitives:

| Primitive | What it is | Example |
|-----------|------------|---------|
| **Node** | A document element (paragraph, heading, image) | `Image`, custom `<Embed>` block |
| **Mark** | A range modifier (bold, italic, link) | `Bold`, custom `<Highlight>` |
| **Extension** | Behavior without a schema impact (keyboard shortcuts, plugins) | `History`, `KeyboardShortcuts`, drag-drop interceptor |

```ts
// Custom Node — embed a YouTube video
import { Node, mergeAttributes } from '@tiptap/core';

export const YouTube = Node.create({
  name: 'youtube',
  group: 'block',
  atom: true,
  addAttributes: () => ({ id: { default: null } }),
  parseHTML: () => [{ tag: 'iframe[src*="youtube.com"]' }],
  renderHTML: ({ HTMLAttributes }) => [
    'iframe',
    mergeAttributes(HTMLAttributes, {
      src: `https://www.youtube.com/embed/${HTMLAttributes.id}`,
      frameborder: 0,
      allowfullscreen: true,
      class: 'aspect-video w-full rounded',
    }),
  ],
});
```

`atom: true` means the node is opaque — users can't put their cursor inside it.

---

## 10. AI in the Editor — Real Patterns

The 2026 expectation: any editor lets the user select text and trigger AI actions. Three patterns:

### Pattern A — Built-in (Tiptap AI extension)

```bash
npm i @tiptap/extension-ai@2.5
```

```tsx
import Ai from '@tiptap/extension-ai';

const editor = useEditor({
  extensions: [
    StarterKit,
    Ai.configure({
      appId: process.env.NEXT_PUBLIC_TIPTAP_AI_APP_ID!,
      token: 'jwt-from-your-server',
    }),
  ],
});
```

Works with Tiptap's Pro AI service ($50+/mo). Provides slash commands, autocomplete, summarize, translate.

### Pattern B — Roll-your-own with Vercel AI SDK 4

```tsx
async function rewriteSelection() {
  const { from, to } = editor.state.selection;
  const selected = editor.state.doc.textBetween(from, to, '\n');

  const res = await fetch('/api/ai/rewrite', {
    method: 'POST',
    body: JSON.stringify({ text: selected }),
  });

  // Stream tokens into the editor (skipping for brevity)
  const rewritten = await res.text();
  editor.chain().focus().deleteRange({ from, to }).insertContent(rewritten).run();
}
```

The server uses `streamText` from `ai@4` (covered in `09.beyond-web/04-ai-integration.md`) to call Claude / GPT-4o-mini. Cheaper and more flexible than Pattern A.

### Pattern C — BlockNote AI (paid Pro tier)

`@blocknote/xl-ai@0.21` adds a "/" → "Ask AI" panel with summarize, rewrite, continue, brainstorm. Same model: server-routed via your own API.

---

## 11. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Hydration mismatch on Tiptap in Next.js | Add `immediatelyRender: false` to `useEditor` |
| Toolbar buttons don't reflect formatting | Component isn't re-rendering on selection change. Pass `editor` as a prop and use `editor.isActive(...)` — Tiptap re-renders the wrapper automatically |
| Saving HTML loses formatting on re-import | Switch to JSON. Don't store HTML. |
| `editor.commands.setContent(...)` triggers `onUpdate` infinitely | Pass `false` as the second arg to suppress the update event: `setContent(json, false)` |
| Custom block renders but is non-editable in the editor | Set `atom: false` (or remove `atom`) and add a `content` schema |
| Yjs collab doc grows huge | Snapshot periodically with `Y.encodeStateAsUpdateV2` and discard old updates (covered in `09.beyond-web/03-realtime.md`) |
| Lexical complains "useLayoutEffect requires DOM" | Wrap with `dynamic(..., { ssr: false })` in Next.js |
| Pasting Word/Google Docs HTML produces ugly output | Use `@tiptap/extension-text-style` + paste rules, or pre-process with `prosemirror-paste-rules` |
| Editor steals focus from form fields | Don't auto-focus on mount. Let the user click in. |
| Rich text content is XSS-able when rendered | Sanitize with `dompurify@3` if rendering HTML from any non-trusted source |

---

## 12. Decision Tree

```
What's the content shape?
│
├── Inline (chat, comments, single-line) → Tiptap 2 with minimal extensions
├── Article / long-form → Tiptap 2 + StarterKit + Image + Link + Placeholder
├── Notion-style blocks (slash menu, drag handle) → BlockNote 0.21
├── Code-heavy (devtools, dev.to-style) → Tiptap 2 + extension-code-block-lowlight
├── Real-time collaborative → Tiptap 2 + Yjs + PartyKit (or Liveblocks)
└── At Meta scale, performance is paramount → Lexical 0.18

Need AI in the editor?
├── Want managed, paid Tiptap Pro → @tiptap/extension-ai
├── Want full control, pay-per-token → roll your own with Vercel AI SDK 4
└── Want it pre-wired in a Notion-style UI → BlockNote 0.21 + xl-ai

Building a Wordpress-style CMS for a non-technical client?
└── TinyMCE 7 or CKEditor 5 — Word-style toolbars are what they expect
```

---

## 13. What This Topic Connects To

- **`08.ecosystem/01-animation.md`** — drag handles in BlockNote use Framer Motion patterns.
- **`08.ecosystem/02-ui-libraries.md`** — Tiptap toolbars built with shadcn/ui look natural in the rest of your app.
- **`09.beyond-web/03-realtime.md`** — collaborative editing pairs Tiptap + Yjs + PartyKit.
- **`09.beyond-web/04-ai-integration.md`** — selection → server-side AI rewrite pattern reuses `streamText` and `useChat` plumbing.
- **`07.nextjs/03-data-fetching.md`** — Server Action save pattern (debounced) for editor content.

---

## Summary

| Editor | Pick when |
|--------|-----------|
| `@tiptap/react@2.5` + StarterKit | Default for any rich text in a React/Next.js app |
| `@blocknote/react@0.21` + Mantine | Notion-style block editor, drop-in, slash menu out of the box |
| `lexical@0.18` | Performance-critical at Meta scale, or your team prefers it |
| Tiptap 2 + Yjs + PartyKit | Collaborative editing |
| TinyMCE 7 / CKEditor 5 | Wordpress-style CMS for non-technical writers |
| ProseMirror direct | You want zero abstraction tax and know the model |

| Rule | Why |
|------|-----|
| Always store JSON, not HTML | Lossless, version-tolerant, structured |
| Debounce saves at 1–2s, not per keystroke | Avoids overwhelming your DB and the network |
| Sanitize HTML on the way OUT, not just on the way IN | Defense in depth |
| `immediatelyRender: false` for Tiptap in Next.js App Router | Fixes hydration mismatch |
| Use Tiptap unless there's a specific reason for Lexical | Bigger ecosystem, better DX, faster onboarding |

---

## Further reading

- [Tiptap docs](https://tiptap.dev/docs)
- [Tiptap StarterKit](https://tiptap.dev/docs/editor/extensions/functionality/starterkit)
- [Tiptap + Yjs collaboration](https://tiptap.dev/docs/editor/extensions/functionality/collaboration)
- [BlockNote docs](https://www.blocknotejs.org/docs)
- [Lexical docs](https://lexical.dev/docs/intro)
- [ProseMirror reference](https://prosemirror.net/docs/)
- [DOMPurify](https://github.com/cure53/DOMPurify)
