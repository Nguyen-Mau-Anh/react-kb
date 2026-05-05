# Production — 02. Drag & Kanban: dnd-kit 6 Deep Dive

> **What / Why / How** — `@dnd-kit/core@6` is the 2026 React drag-and-drop default. Skip `react-beautiful-dnd@13` (deprecated). Three patterns cover 95% of cases: sortable list, multi-list kanban, virtualized drag.

---

## 1. The Real Choice in 2026

| Library | npm | Status | Best for |
|---------|-----|--------|----------|
| **dnd-kit 6** | `@dnd-kit/core@6.1`, `@dnd-kit/sortable@8.0`, `@dnd-kit/utilities@3.2` | Active, hooks-first | The 2026 default. Sortable lists, kanban, free-form drag. |
| **react-beautiful-dnd 13** | `react-beautiful-dnd@13` | **Officially deprecated** by Atlassian (April 2024) | Don't pick for new code. Atlassian forked themselves into Pragmatic. |
| **Atlassian Pragmatic Drag and Drop** | `@atlaskit/pragmatic-drag-and-drop@1.4` | Active (Atlassian's RBD successor) | Jira's internal stack, lower-level than dnd-kit |
| **react-dnd 16** | `react-dnd@16`, `react-dnd-html5-backend@16` | Maintenance mode | Older codebases; class-component history |
| **Native HTML5 drag** | `draggable` attribute + events | Built-in | Simple file-drop zones (drag a file from desktop) |
| **`@dnd-kit/sortable`** + `@dnd-kit/modifiers` | Same family | Active | Multi-axis snap, distance-constrained drag |
| **Framer Motion `Reorder`** | `framer-motion@11` | Active | Tiny single-list reorder, animation-first |

**The 2026 default for any non-trivial drag/drop in React: `@dnd-kit/core@6`.** It's accessibility-first, hooks-based, supports keyboard + pointer + touch, and has explicit kanban/multi-list support via `@dnd-kit/sortable@8`.

For "swipe-to-delete" style single-axis gestures: prefer `@use-gesture/react@10` + `react-native-reanimated@3` (mobile, covered in `09.beyond-web/01-react-native-expo.md`) or `framer-motion@11`'s `drag` prop on web (covered in `08.ecosystem/01-animation.md`).

---

## 2. Why dnd-kit Won

### What

`@dnd-kit/core@6` is a hooks-based drag-and-drop toolkit by Claudéric Demers. It does **not** ship a Kanban component. It ships **primitives** (`useDraggable`, `useDroppable`, `<DndContext>`) that you compose into kanban, sortable lists, file drop zones, free-form rearrangement.

### Why dnd-kit beat react-beautiful-dnd

| | `react-beautiful-dnd@13` | `@dnd-kit/core@6` |
|--|---------------------------|--------------------|
| Status | **Deprecated April 2024** | Active, monthly releases |
| Class components support | Yes (legacy era) | Hooks-only (modern) |
| Multi-list kanban | Native, opinionated | Via `@dnd-kit/sortable` strategies |
| Virtualization | ❌ Hard — RBD doesn't play with `react-window` / virtualizers | ✅ Works with `@tanstack/react-virtual@3` |
| Accessibility | Good defaults | Excellent — full keyboard nav, screen-reader announcements |
| Bundle (gzipped) | ~30 KB | ~10 KB core + ~3 KB sortable |
| Touch support | Limited | First-class `PointerSensor` + `TouchSensor` |
| TypeScript | OK | Strong, generic-aware |

Atlassian's deprecation notice explicitly recommends migrating to **dnd-kit** or **Pragmatic Drag and Drop**. For most React apps: **dnd-kit**.

### When Pragmatic DnD beats dnd-kit

Atlassian's `@atlaskit/pragmatic-drag-and-drop@1.4` is lower-level than dnd-kit — closer to native HTML5 drag/drop with their own performance optimizations. Real fit:

- You're building Jira-scale drag (thousands of items, dozens of lists) and dnd-kit's `<DndContext>` becomes a perf bottleneck.
- You need cross-iframe drag (Pragmatic's strength).
- Your team is already at Atlassian or familiar with their patterns.

For everything else in 2026: **dnd-kit**.

---

## 3. dnd-kit's Three Real Primitives

```
┌─────────────────────────────────────────────────┐
│ <DndContext>                                    │ ← provides sensors, collision detection, fires events
│  ┌──────────────────────┐ ┌──────────────────┐ │
│  │ useDraggable(...)     │ │ useDroppable(...) │ │
│  │ — the thing you grab  │ │ — the target zone │ │
│  └──────────────────────┘ └──────────────────┘ │
└─────────────────────────────────────────────────┘
```

- **`<DndContext>`**: provider that owns the drag state and dispatches events.
- **`useDraggable`**: hook that turns any element into a draggable.
- **`useDroppable`**: hook that turns any element into a drop target.

For sortable lists, `@dnd-kit/sortable@8` adds `useSortable`, which combines `useDraggable` + `useDroppable` into one ergonomic hook.

---

## 4. Pattern A — Sortable List (the Most Common Case)

Real example: drag-reorder a list of tasks. This is what was wired into the capstone Task Manager (covered in `08.ecosystem/04-real-project.md`).

```bash
npm i @dnd-kit/core@6.1 @dnd-kit/sortable@8.0 @dnd-kit/utilities@3.2
```

```tsx
'use client';
import {
  DndContext,
  closestCenter,
  KeyboardSensor,
  PointerSensor,
  useSensor,
  useSensors,
  type DragEndEvent,
} from '@dnd-kit/core';
import {
  SortableContext,
  verticalListSortingStrategy,
  arrayMove,
  sortableKeyboardCoordinates,
  useSortable,
} from '@dnd-kit/sortable';
import { CSS } from '@dnd-kit/utilities';
import { useState } from 'react';

type Task = { id: string; title: string };

export function SortableTasks({ initial }: { initial: Task[] }) {
  const [tasks, setTasks] = useState(initial);
  const sensors = useSensors(
    useSensor(PointerSensor, { activationConstraint: { distance: 5 } }),
    useSensor(KeyboardSensor, { coordinateGetter: sortableKeyboardCoordinates }),
  );

  function handleDragEnd(event: DragEndEvent) {
    const { active, over } = event;
    if (!over || active.id === over.id) return;
    setTasks((items) => {
      const oldIndex = items.findIndex(i => i.id === active.id);
      const newIndex = items.findIndex(i => i.id === over.id);
      return arrayMove(items, oldIndex, newIndex);
    });
  }

  return (
    <DndContext sensors={sensors} collisionDetection={closestCenter} onDragEnd={handleDragEnd}>
      <SortableContext items={tasks.map(t => t.id)} strategy={verticalListSortingStrategy}>
        <ul className="space-y-2">
          {tasks.map(t => <SortableRow key={t.id} task={t} />)}
        </ul>
      </SortableContext>
    </DndContext>
  );
}

function SortableRow({ task }: { task: Task }) {
  const { attributes, listeners, setNodeRef, transform, transition, isDragging } = useSortable({ id: task.id });
  const style = {
    transform: CSS.Transform.toString(transform),
    transition,
    opacity: isDragging ? 0.5 : 1,
  };
  return (
    <li
      ref={setNodeRef}
      style={style}
      {...attributes}
      {...listeners}
      className="cursor-grab rounded-md border bg-card p-3"
    >
      {task.title}
    </li>
  );
}
```

### What's happening

- **`PointerSensor` with `distance: 5`** — drag activates only after the cursor moves 5px. Without this, every click registers as a drag and you can't click rows to open them.
- **`KeyboardSensor` + `sortableKeyboardCoordinates`** — the user can use Space/Enter to grab and arrow keys to reorder. Free accessibility.
- **`closestCenter`** — collision detection algorithm that picks the closest drop target by center point. The default for vertical lists.
- **`verticalListSortingStrategy`** — tells `SortableContext` how to compute slide animations as items reorder.
- **`arrayMove(items, old, new)`** — utility from `@dnd-kit/sortable` that handles the array-reordering logic.
- **`CSS.Transform.toString(transform)`** — converts the `{x, y, scaleX, scaleY}` object into a CSS `transform` string.

### Persisting to the server with optimistic update

Pair with TanStack Query (covered in `04.state-and-data/03-data-fetching.md`):

```tsx
const reorder = trpc.task.reorder.useMutation({
  onMutate: async ({ orderedIds }) => {
    await utils.project.byId.cancel({ id: projectId });
    const prev = utils.project.byId.getData({ id: projectId });
    if (prev) {
      utils.project.byId.setData({ id: projectId }, {
        ...prev,
        tasks: orderedIds.map(id => prev.tasks.find(t => t.id === id)!).filter(Boolean),
      });
    }
    return { prev };
  },
  onError: (_e, _v, ctx) => { if (ctx?.prev) utils.project.byId.setData({ id: projectId }, ctx.prev); },
  onSettled: () => utils.project.byId.invalidate({ id: projectId }),
});

function handleDragEnd(event: DragEndEvent) {
  const { active, over } = event;
  if (!over || active.id === over.id) return;
  const next = arrayMove(tasks, tasks.findIndex(t => t.id === active.id), tasks.findIndex(t => t.id === over.id));
  setTasks(next); // optimistic UI
  reorder.mutate({ projectId, orderedIds: next.map(t => t.id) }); // network
}
```

The user sees instant feedback; if the server rejects, `onError` rolls back.

---

## 5. Pattern B — Multi-List Kanban (Trello / Linear-Style)

The harder pattern: drag between columns. Different from a single sortable list because the drop target's container changes.

```tsx
'use client';
import {
  DndContext,
  closestCorners,
  PointerSensor,
  useSensor,
  useSensors,
  DragOverlay,
  type DragStartEvent,
  type DragOverEvent,
  type DragEndEvent,
} from '@dnd-kit/core';
import { SortableContext, useSortable, arrayMove } from '@dnd-kit/sortable';
import { CSS } from '@dnd-kit/utilities';
import { useState } from 'react';

type Card = { id: string; title: string; columnId: string };
type Column = { id: string; title: string };

const columns: Column[] = [
  { id: 'todo', title: 'To do' },
  { id: 'doing', title: 'In progress' },
  { id: 'done', title: 'Done' },
];

export function KanbanBoard({ initial }: { initial: Card[] }) {
  const [cards, setCards] = useState(initial);
  const [activeId, setActiveId] = useState<string | null>(null);
  const sensors = useSensors(useSensor(PointerSensor, { activationConstraint: { distance: 5 } }));

  function handleDragStart(event: DragStartEvent) {
    setActiveId(event.active.id as string);
  }

  function handleDragOver(event: DragOverEvent) {
    const { active, over } = event;
    if (!over) return;
    const activeCard = cards.find(c => c.id === active.id);
    if (!activeCard) return;

    // Dragging over a card in another column → move there
    const overCard = cards.find(c => c.id === over.id);
    if (overCard && overCard.columnId !== activeCard.columnId) {
      setCards((cs) =>
        cs.map(c => c.id === active.id ? { ...c, columnId: overCard.columnId } : c)
      );
      return;
    }

    // Dragging over an empty column → move there
    const overColumn = columns.find(col => col.id === over.id);
    if (overColumn && activeCard.columnId !== overColumn.id) {
      setCards((cs) => cs.map(c => c.id === active.id ? { ...c, columnId: overColumn.id } : c));
    }
  }

  function handleDragEnd(event: DragEndEvent) {
    const { active, over } = event;
    setActiveId(null);
    if (!over) return;
    if (active.id === over.id) return;
    const activeCard = cards.find(c => c.id === active.id);
    const overCard = cards.find(c => c.id === over.id);
    if (!activeCard || !overCard) return;

    // Same column: reorder within
    if (activeCard.columnId === overCard.columnId) {
      const colCards = cards.filter(c => c.columnId === activeCard.columnId);
      const oldIdx = colCards.findIndex(c => c.id === active.id);
      const newIdx = colCards.findIndex(c => c.id === over.id);
      const reordered = arrayMove(colCards, oldIdx, newIdx);
      setCards((cs) =>
        cs.filter(c => c.columnId !== activeCard.columnId).concat(reordered)
      );
    }
  }

  const activeCard = cards.find(c => c.id === activeId);

  return (
    <DndContext
      sensors={sensors}
      collisionDetection={closestCorners}
      onDragStart={handleDragStart}
      onDragOver={handleDragOver}
      onDragEnd={handleDragEnd}
    >
      <div className="grid grid-cols-3 gap-4">
        {columns.map(col => (
          <KanbanColumn
            key={col.id}
            column={col}
            cards={cards.filter(c => c.columnId === col.id)}
          />
        ))}
      </div>

      <DragOverlay>
        {activeCard ? <KanbanCard card={activeCard} dragging /> : null}
      </DragOverlay>
    </DndContext>
  );
}

function KanbanColumn({ column, cards }: { column: Column; cards: Card[] }) {
  const { setNodeRef } = useSortable({ id: column.id });
  return (
    <div ref={setNodeRef} className="rounded-lg bg-muted p-3">
      <h3 className="mb-2 text-sm font-semibold">{column.title}</h3>
      <SortableContext items={cards.map(c => c.id)}>
        <ul className="space-y-2">
          {cards.map(c => <KanbanCard key={c.id} card={c} />)}
        </ul>
      </SortableContext>
    </div>
  );
}

function KanbanCard({ card, dragging }: { card: Card; dragging?: boolean }) {
  const { attributes, listeners, setNodeRef, transform, transition, isDragging } =
    useSortable({ id: card.id });
  const style = {
    transform: CSS.Transform.toString(transform),
    transition,
    opacity: isDragging && !dragging ? 0 : 1,
  };
  return (
    <li
      ref={setNodeRef}
      style={style}
      {...attributes}
      {...listeners}
      className="rounded-md border bg-background p-2 shadow-sm"
    >
      {card.title}
    </li>
  );
}
```

### Why this works

- **`closestCorners`** instead of `closestCenter` — better for kanban because the user often hovers near the edges of cards/columns.
- **`onDragOver` moves the card across columns immediately** so the user sees real-time placement, not a "jump" at drop. This is the Trello/Linear feel.
- **`onDragEnd` does the within-column reorder** because by `dragEnd`, the column has already been changed by `onDragOver`.
- **`<DragOverlay>`** renders a clone of the dragging card on top of the cursor. Without it, the original card stays in its old position visually — jarring. The overlay is the smooth "ghost" that follows the cursor.
- **Each column also has `useSortable({ id: col.id })`** so the column itself is a drop target when empty.

### Persisting kanban state

Two server-side fields you need: `columnId` and `position` (covered in `08.ecosystem/04-real-project.md`'s schema). On every column move, update both. On every same-column reorder, update only `position`.

The `position: Int` step-of-1024 trick (e.g., `[1024, 2048, 3072]`) gives you room to insert at the average without rewriting every row. When two adjacent positions get within 1, run a re-pack `db.$transaction`.

---

## 6. Pattern C — Virtualized Drag (Long Lists)

For 1,000+ items, render only what's visible (covered in `06.testing-perf/02-performance.md` and `03.rendering-patterns/02-conditional-lists.md`). dnd-kit works with `@tanstack/react-virtual@3`:

```tsx
'use client';
import { useRef, useState } from 'react';
import { useVirtualizer } from '@tanstack/react-virtual';
import { DndContext, PointerSensor, useSensor, useSensors, type DragEndEvent } from '@dnd-kit/core';
import { SortableContext, useSortable, arrayMove, verticalListSortingStrategy } from '@dnd-kit/sortable';
import { CSS } from '@dnd-kit/utilities';

export function VirtualizedSortable({ items: initial }: { items: Item[] }) {
  const [items, setItems] = useState(initial);
  const parentRef = useRef<HTMLDivElement>(null);

  const v = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 56,
    overscan: 8,
  });

  const sensors = useSensors(useSensor(PointerSensor, { activationConstraint: { distance: 5 } }));

  function handleDragEnd(e: DragEndEvent) {
    const { active, over } = e;
    if (!over || active.id === over.id) return;
    setItems((arr) => arrayMove(arr, arr.findIndex(i => i.id === active.id), arr.findIndex(i => i.id === over.id)));
  }

  return (
    <DndContext sensors={sensors} onDragEnd={handleDragEnd}>
      <SortableContext items={items.map(i => i.id)} strategy={verticalListSortingStrategy}>
        <div ref={parentRef} style={{ height: 600, overflow: 'auto' }}>
          <div style={{ height: v.getTotalSize(), position: 'relative' }}>
            {v.getVirtualItems().map(vi => (
              <SortableRow
                key={items[vi.index].id}
                item={items[vi.index]}
                top={vi.start}
                height={vi.size}
              />
            ))}
          </div>
        </div>
      </SortableContext>
    </DndContext>
  );
}

function SortableRow({ item, top, height }: { item: Item; top: number; height: number }) {
  const { attributes, listeners, setNodeRef, transform, transition, isDragging } = useSortable({ id: item.id });
  const style = {
    position: 'absolute' as const,
    top: 0,
    transform: CSS.Translate.toString({
      x: transform?.x ?? 0,
      y: (transform?.y ?? 0) + top,
      scaleX: 1,
      scaleY: 1,
    }),
    transition,
    height,
    opacity: isDragging ? 0.5 : 1,
  };
  return (
    <div ref={setNodeRef} style={style} {...attributes} {...listeners} className="border-b bg-card p-3">
      {item.label}
    </div>
  );
}
```

### Critical details for virtualized drag

- **`SortableContext` must hold ALL item IDs**, not just the rendered ones. Otherwise drop targets disappear when scrolled off-screen.
- **`useVirtualizer`'s `overscan: 8`** renders 8 extra rows above and below — gives you a smoother drag-near-the-edge experience.
- **Combine the virtualizer's `top` with the drag transform** — that's why the `transform.y` adds `top` in the example.

For very long lists with multi-list kanban: the drag-overlay pattern from §5 handles the "ghost" without messing with virtualization.

---

## 7. Real Production Touches

### Drag handles vs whole-card drag

```tsx
// Whole row is draggable (default with the spread above)
<li {...attributes} {...listeners}>...</li>

// Only a handle area is draggable (better for cards with buttons/links inside)
<li {...attributes}>
  <span {...listeners} className="cursor-grab">⋮⋮</span>
  <button onClick={openModal}>Click me without dragging</button>
</li>
```

In Linear-style boards, the whole card is draggable. In Notion-style block editors, only the gutter handle is draggable. Pick by user expectation.

### Activation constraint — distance vs delay

```ts
useSensor(PointerSensor, { activationConstraint: { distance: 5 } })
// vs
useSensor(PointerSensor, { activationConstraint: { delay: 250, tolerance: 5 } })
```

| Constraint | When | Trade-off |
|-----------|------|-----------|
| `distance: 5` | Desktop, mouse-heavy UIs | Click and drag both work, no delay |
| `delay: 250, tolerance: 5` | Touch devices, mobile-first | Distinguishes tap from drag-to-reorder |

For mobile-first apps: use `delay`. The user expects a "long-press to grab" gesture; tapping once selects.

### Snap-to-grid and other modifiers

```tsx
import { restrictToVerticalAxis, snapCenterToCursor } from '@dnd-kit/modifiers';

<DndContext modifiers={[restrictToVerticalAxis, snapCenterToCursor]} ...>
```

Built-in modifiers: `restrictToVerticalAxis`, `restrictToHorizontalAxis`, `restrictToParentElement`, `restrictToWindowEdges`, `snapCenterToCursor`. Trivial to write your own (`(args) => ({ x: 0, y: args.transform.y })` for vertical-only).

### Auto-scroll while dragging

dnd-kit auto-scrolls the nearest scroll container when you drag near the edge. Configure via `<DndContext autoScroll={false}>` or pass `{ thresholds: { x: 0.2, y: 0.2 } }` to tune.

### Real-time collaboration (multiple users dragging)

Pair dnd-kit with Liveblocks or PartyKit (covered in `09.beyond-web/03-realtime.md`). Broadcast the dragged item's `id` and current position; render other users' drag overlays. Real implementation: ~150 lines using `@liveblocks/react@2`'s `useStorage` for the shared cards array.

---

## 8. Accessibility — What dnd-kit Gives You for Free

| Feature | How |
|---------|-----|
| **Keyboard navigation** | Tab to a draggable, Space to grab, arrows to move, Space to drop, Esc to cancel |
| **Screen reader announcements** | "Picked up Card X. It is in position 3 of 10." Each move and drop is announced |
| **Custom announcements** | `<DndContext accessibility={{ announcements: { ... } }}>` lets you customize messages |
| **Focus management** | Returns focus to the dragged element after drop |
| **`aria-roledescription`, `aria-disabled`** | Set automatically on draggable elements |

`react-beautiful-dnd@13` had similar features but they broke on virtualization. dnd-kit handles both correctly.

Real implementation for custom announcements:

```tsx
<DndContext
  accessibility={{
    announcements: {
      onDragStart: ({ active }) => `Picked up ${active.data.current?.title}.`,
      onDragOver: ({ active, over }) =>
        over ? `${active.data.current?.title} is over ${over.data.current?.title}.` : '',
      onDragEnd: ({ active, over }) =>
        over ? `Dropped ${active.data.current?.title} on ${over.data.current?.title}.` : 'Dropped outside.',
      onDragCancel: ({ active }) => `Cancelled drag of ${active.data.current?.title}.`,
    },
  }}
>
```

You pass titles via `useSortable({ id, data: { title } })` and read them from `active.data.current`.

---

## 9. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Click handlers fire when you try to drag | Add `activationConstraint: { distance: 5 }` so a tap doesn't activate drag |
| Card "snaps" back when dropped on another column | `onDragOver` isn't moving the card across columns. See Pattern B's `handleDragOver`. |
| Drag overlay shows the wrong style | Apply `style.opacity = isDragging ? 0 : 1` on the **original**, not the overlay |
| Items render in the wrong order during drag | `SortableContext` must hold all item IDs in their current order |
| Virtualized drag "loses" items past the viewport | The virtualizer is hiding them; ensure `SortableContext` lists all items even if not rendered |
| Drag works on desktop but not touch | Add `TouchSensor` or use `PointerSensor` with `delay` constraint |
| Server gets stale order after rapid drags | Debounce + always send the **full** ordered list, not deltas |
| `over` is always `null` on touch | Increase `tolerance` in the `delay` constraint; touch events have less precision |
| Long list's drag overlay janks at 60fps | Move heavy content (avatars, large text) out of the dragged element; render a simplified ghost |
| Auto-scroll fires when scroll-locking is intended | Set `autoScroll={false}` on `DndContext` |
| Keyboard nav doesn't move items | Ensure `KeyboardSensor` is in the sensors list AND `coordinateGetter: sortableKeyboardCoordinates` is set |
| `useSortable({ id })` warns "id must be unique" | An item is rendered twice; or your IDs aren't strings/numbers |

---

## 10. Decision Tree

```
What are you building?
│
├── Single sortable list (todos, settings reorder, file order)?
│   → Pattern A: SortableContext + verticalListSortingStrategy + arrayMove
│
├── Multi-list kanban (Trello, Linear, Asana)?
│   → Pattern B: closestCorners + onDragOver moves across, DragOverlay for ghost
│
├── 1,000+ rows in a sortable list?
│   → Pattern C: SortableContext over full list IDs + @tanstack/react-virtual for rendering
│
├── Free-form drag (Figma-like, design canvas)?
│   → DndContext + useDraggable directly (no SortableContext) + custom collision detection
│
├── File drop zone (drag from desktop)?
│   → Native HTML5 drag/drop or react-dropzone@14 (much simpler than dnd-kit)
│
├── Single-axis swipe-to-delete on mobile?
│   → @use-gesture/react@10 + framer-motion@11 (covered in 08.ecosystem/01-animation.md)
│
└── Real-time multi-user drag (collaborative kanban)?
   → dnd-kit local + Liveblocks/PartyKit for sync (covered in 09.beyond-web/03-realtime.md)
```

---

## 11. What This Topic Connects To

- **`03.rendering-patterns/02-conditional-lists.md`** — list virtualization with `@tanstack/react-virtual@3`.
- **`04.state-and-data/03-data-fetching.md`** — TanStack Query optimistic update pattern for drag-reorder server sync.
- **`08.ecosystem/01-animation.md`** — `framer-motion@11` for animation, `@use-gesture/react@10` for low-level gestures.
- **`08.ecosystem/04-real-project.md`** — the real Task Manager kanban uses Pattern A; extending to multi-column is Pattern B.
- **`09.beyond-web/03-realtime.md`** — broadcasting drag state for collaborative cursors.
- **`10.production/01-rich-text-editors.md`** — BlockNote's drag handle uses dnd-kit internally.

---

## Summary

| Pattern | Tool combination |
|---------|-------------------|
| Sortable list | `DndContext` + `SortableContext` + `verticalListSortingStrategy` + `arrayMove` |
| Multi-list kanban | `closestCorners` + `onDragOver` (across) + `onDragEnd` (within) + `DragOverlay` |
| Virtualized drag | `SortableContext` over full IDs + `@tanstack/react-virtual@3` for rendering |
| Free-form drag | `useDraggable` + custom collision detection + no `SortableContext` |
| File drop zone | `react-dropzone@14`, not dnd-kit |
| Touch-first | `PointerSensor` with `delay: 250, tolerance: 5` |

| Rule | Why |
|------|-----|
| Use `@dnd-kit/core@6` for new code | `react-beautiful-dnd@13` is officially deprecated |
| Set `activationConstraint: { distance: 5 }` (or delay/tolerance) | Distinguishes click from drag |
| Always include `KeyboardSensor` + `sortableKeyboardCoordinates` | Free accessibility win |
| Pair drag with `<DragOverlay>` for kanban | Smooth ghost; original card hides cleanly |
| `SortableContext` must list ALL ids, even when virtualized | Drop targets vanish otherwise |
| Use `position: Int` step-of-1024 in DB | Cheap inserts, periodic re-pack |

---

## Further reading

- [dnd-kit docs](https://docs.dndkit.com/)
- [dnd-kit storybook (real examples)](https://master--5fc05e08a4a65d0021ae0bf2.chromatic.com/)
- [Atlassian's Pragmatic Drag and Drop](https://atlassian.design/components/pragmatic-drag-and-drop/about)
- [TanStack Virtual + dnd-kit recipe](https://tanstack.com/virtual/latest/docs/examples/react/sortable-list)
- [Migrating from react-beautiful-dnd to dnd-kit (community)](https://github.com/clauderic/dnd-kit/discussions/1418)
- [WAI-ARIA drag-and-drop authoring practices](https://www.w3.org/WAI/ARIA/apg/patterns/)
