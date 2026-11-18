# Drag & Drop Sortable List — Machine Coding

> **Difficulty:** Medium | **Time:** 45 min | **Frequency:** ⭐⭐⭐⭐

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Requirements](#2-requirements)
3. [Clarifying Questions to Ask](#3-clarifying-questions-to-ask)
4. [Approach & Component Design](#4-approach--component-design)
5. [Reference Solution](#5-reference-solution)
6. [How It Works](#6-how-it-works)
7. [Edge Cases](#7-edge-cases)
8. [Accessibility](#8-accessibility)
9. [Performance Considerations](#9-performance-considerations)
10. [Follow-up Questions](#10-follow-up-questions)
11. [Common Mistakes](#11-common-mistakes)

---

## 1. Problem Statement

Build a **drag-to-reorder list** **without a DnD library**:

1. **Native HTML5 Drag & Drop** version — drag an item, a blue **drop indicator** line shows where it will land, drop to reorder.
2. A **Pointer Events** version that also works on **touch** screens (native DnD generally doesn't fire on touch), with a drag handle and a floating dragged row.
3. A **keyboard alternative**: focus a handle, **Space** to grab, **↑ / ↓** to move, **Space** to drop, **Esc** to cancel — with screen-reader announcements.
4. For comparison, a short **`dnd-kit`** version (`SortableContext` + `useSortable`).

---

## 2. Requirements

### Functional

- Render a vertical list of items (`{ id, label }`); reorder by dragging.
- Show a **drop indicator** (line) at the insertion point; dim the dragged item.
- Dropping at the item's original position (or right after it) is a no-op.
- Dropping outside the list cancels the drag.
- **Touch support** via Pointer Events (handle uses `touch-action: none`).
- **Keyboard reordering**: Space/Enter grab, Arrow Up/Down move, Space/Enter drop, Escape cancel (restores the original position), Tab drops.
- Announce grab / move / drop / cancel in a live region.

### Non-functional

- One pure helper (`moveItem`) shared by every input method — the list state is only ever changed through it.
- Drag session state (`dragId`, `slot`, `dy`) is transient UI state — never persisted.
- No layout shift while dragging (indicator is an absolutely-positioned pseudo-element) → no `dragover` flicker.
- Ignore foreign drags (files, text from other pages).
- No third-party libraries (except in the final comparison snippet).

---

## 3. Clarifying Questions to Ask

- Vertical list only, or grid / horizontal / multiple lists (Kanban)?
- Is **touch** required? (Decides native DnD vs Pointer Events immediately.)
- Is the whole row draggable or only a **handle**?
- Live reordering (items shift out of the way) or a **drop line** (items stay put)?
- Keyboard and screen-reader support expected?
- Persist order? Any backend ordering (fractional index / `position` column)?
- List size? (virtualization & auto-scroll when dragging near an edge)
- Nested lists / dragging between lists / multi-select drag?

---

## 4. Approach & Component Design

**State shape**

```js
items: [{ id, label }, ...]          // domain data (the only thing that is saved)
// transient UI state
dragId: string | null                // native version
drag:   { id, dy } | null            // pointer version (dy = vertical offset of the floating row)
slot:   number | null                // insertion slot 0..items.length (n+1 possible slots)
grabbedId: string | null             // keyboard version
```

**Insertion "slots".** For a list of `n` items there are `n + 1` gaps. Slot `i` means "insert *before* item `i`"; slot `n` means "after the last item".

```
        slot 0 ─────────────
               [ A ]            dragging C from index 2
        slot 1 ─────────────
               [ B ]
        slot 2 ─────────────    slot 2 and slot 3 are no-ops (C's own position)
               [ C ]
        slot 3 ─────────────
               [ D ]
        slot 4 ─────────────
```

```js
slotToIndex(from, slot) = slot > from ? slot - 1 : slot   // final index after the dragged item is removed
isNoop(from, slot)      = slot === from || slot === from + 1
```

**Slot from the pointer position** — compare `clientY` with each item's vertical midpoint; the first item whose midpoint is below the pointer gives the slot; if none, the slot is `n`.

**HTML5 DnD event map**

| Event | On | We do |
|---|---|---|
| `dragstart` | `<li draggable>` | `setData('text/plain', id)`, `effectAllowed = 'move'`, `setDragId` |
| `dragover` | `<li>` | ignore if `dragId == null` (foreign drag); `preventDefault()`; midpoint → `setSlot` |
| `dragover` | `<ul>` | `preventDefault()` so gaps between items stay valid drop targets |
| `dragleave` | `<ul>` | clear `slot` when the pointer leaves the list entirely |
| `drop` | `<ul>` | `moveItem(items, from, slotToIndex(from, slot))`, reset |
| `dragend` | `<li>` | reset (drop outside the list / Esc) |

**Pointer Events flow (touch + mouse + pen)**

```
pointerdown (handle) ─ setPointerCapture ─ drag = { id, dy: 0 }
pointermove (handle) ─ dy = clientY - startY ; slot = slotFromY(clientY)
pointerup            ─ moveItem(...) ; reset          pointercancel ─ reset (no commit)
```

**Component tree**

```
App
 ├─ NativeSortableList     draggable <li>, drop indicator via ::before/::after
 ├─ SortableList           pointer handle + keyboard (useKeyboardReorder)
 │    └─ <li> [handle button] label
 ├─ DndKitSortableList     (comparison) DndContext → SortableContext → Row(useSortable)
 └─ shared: reorder.js (moveItem, slotToIndex, isNoop), styles.css
```

**Which approach when**

| | Native HTML5 DnD | Pointer Events (own) | `dnd-kit` |
|---|---|---|---|
| Touch | ❌ unreliable | ✅ | ✅ |
| Keyboard | ❌ (add separately) | add separately | ✅ built in (`KeyboardSensor`) |
| Custom drag preview | limited (`setDragImage`) | full control | full (`DragOverlay`) |
| Auto-scroll, collision, animations | browser auto-scroll only | you write it | ✅ |
| Code / bundle | smallest | medium | library (~10–15 kB gz) |
| Interview scope | ✅ expected "no library" answer | ✅ senior bonus | mention only |

---

## 5. Reference Solution

<details>
<summary><strong>Show Full Code</strong></summary>

**`reorder.js`**

```jsx
export function moveItem(list, from, to) {
  if (from === to || from < 0 || to < 0 || from >= list.length || to >= list.length) return list;
  const next = list.slice();
  const [item] = next.splice(from, 1);
  next.splice(to, 0, item);
  return next;
}

export const slotToIndex = (from, slot) => (slot > from ? slot - 1 : slot);

export const isNoop = (from, slot) => slot === from || slot === from + 1;
```

**`NativeSortableList.jsx`**

```jsx
import { useState } from 'react';
import { moveItem, slotToIndex, isNoop } from './reorder';

export default function NativeSortableList({ initialItems }) {
  const [items, setItems] = useState(initialItems);
  const [dragId, setDragId] = useState(null);
  const [slot, setSlot] = useState(null);

  const from = items.findIndex((i) => i.id === dragId);
  const shownSlot = dragId !== null && slot !== null && !isNoop(from, slot) ? slot : null;

  const reset = () => {
    setDragId(null);
    setSlot(null);
  };

  const onDragStart = (e, id) => {
    e.dataTransfer.effectAllowed = 'move';
    e.dataTransfer.setData('text/plain', id);
    setDragId(id);
  };

  const onItemDragOver = (e, index) => {
    if (dragId === null) return;
    e.preventDefault();
    e.dataTransfer.dropEffect = 'move';
    const rect = e.currentTarget.getBoundingClientRect();
    const after = e.clientY > rect.top + rect.height / 2;
    setSlot(index + (after ? 1 : 0));
  };

  const onListDragOver = (e) => {
    if (dragId !== null) e.preventDefault();
  };

  const onListDragLeave = (e) => {
    if (!e.currentTarget.contains(e.relatedTarget)) setSlot(null);
  };

  const onDrop = (e) => {
    if (dragId === null) return;
    e.preventDefault();
    if (slot !== null && !isNoop(from, slot)) {
      setItems((list) => moveItem(list, from, slotToIndex(from, slot)));
    }
    reset();
  };

  return (
    <ul
      className="sortable"
      onDragOver={onListDragOver}
      onDragLeave={onListDragLeave}
      onDrop={onDrop}
    >
      {items.map((item, index) => {
        const cls = [
          'item',
          item.id === dragId && 'dragging',
          shownSlot === index && 'drop-before',
          shownSlot === items.length && index === items.length - 1 && 'drop-after',
        ]
          .filter(Boolean)
          .join(' ');
        return (
          <li
            key={item.id}
            className={cls}
            draggable
            onDragStart={(e) => onDragStart(e, item.id)}
            onDragOver={(e) => onItemDragOver(e, index)}
            onDragEnd={reset}
          >
            <span className="grip" aria-hidden="true">⋮⋮</span>
            {item.label}
          </li>
        );
      })}
    </ul>
  );
}
```

**`useKeyboardReorder.js`**

```jsx
import { useLayoutEffect, useRef, useState } from 'react';

export function useKeyboardReorder({ items, onMove, getLabel = (item) => item.label }) {
  const [grabbedId, setGrabbedId] = useState(null);
  const [message, setMessage] = useState('');
  const origin = useRef(0);
  const handles = useRef(new Map());

  // React moves the DOM node on reorder, which can drop focus — restore it.
  useLayoutEffect(() => {
    if (grabbedId !== null) handles.current.get(grabbedId)?.focus();
  }, [grabbedId, items]);

  const onKeyDown = (e, index) => {
    const item = items[index];
    const label = getLabel(item);
    const total = items.length;

    if (grabbedId === null) {
      if (e.key === ' ' || e.key === 'Enter') {
        e.preventDefault();
        origin.current = index;
        setGrabbedId(item.id);
        setMessage(
          `Grabbed ${label}. Position ${index + 1} of ${total}. Use arrow keys to move, Space to drop, Escape to cancel.`
        );
      }
      return;
    }
    if (item.id !== grabbedId) return;

    switch (e.key) {
      case 'ArrowUp':
      case 'ArrowDown': {
        e.preventDefault();
        const to = index + (e.key === 'ArrowUp' ? -1 : 1);
        if (to < 0 || to >= total) return;
        onMove(index, to);
        setMessage(`Moved ${label} to position ${to + 1} of ${total}.`);
        break;
      }
      case ' ':
      case 'Enter':
        e.preventDefault();
        setGrabbedId(null);
        setMessage(`Dropped ${label} at position ${index + 1} of ${total}.`);
        break;
      case 'Escape':
        e.preventDefault();
        if (origin.current !== index) onMove(index, origin.current);
        setGrabbedId(null);
        setMessage(`Cancelled. ${label} returned to position ${origin.current + 1} of ${total}.`);
        break;
      case 'Tab':
        setGrabbedId(null);
        break;
      default:
    }
  };

  const handleProps = (item, index) => ({
    ref: (el) => {
      if (el) handles.current.set(item.id, el);
      else handles.current.delete(item.id);
    },
    onKeyDown: (e) => onKeyDown(e, index),
    'aria-pressed': grabbedId === item.id,
  });

  return { grabbedId, message, handleProps };
}
```

**`SortableList.jsx`**

```jsx
import { useRef, useState } from 'react';
import { moveItem, slotToIndex, isNoop } from './reorder';
import { useKeyboardReorder } from './useKeyboardReorder';

export default function SortableList({ initialItems }) {
  const [items, setItems] = useState(initialItems);
  const [drag, setDrag] = useState(null);
  const [slot, setSlot] = useState(null);
  const startY = useRef(0);
  const itemEls = useRef(new Map());

  const keyboard = useKeyboardReorder({
    items,
    onMove: (from, to) => setItems((list) => moveItem(list, from, to)),
  });

  const from = drag ? items.findIndex((i) => i.id === drag.id) : -1;
  const shownSlot = drag && slot !== null && !isNoop(from, slot) ? slot : null;

  const slotFromY = (clientY) => {
    for (let i = 0; i < items.length; i++) {
      if (items[i].id === drag.id) continue;
      const rect = itemEls.current.get(items[i].id).getBoundingClientRect();
      if (clientY < rect.top + rect.height / 2) return i;
    }
    return items.length;
  };

  const endDrag = () => {
    setDrag(null);
    setSlot(null);
  };

  const onPointerDown = (e, id) => {
    if (e.button !== 0) return;
    e.currentTarget.setPointerCapture?.(e.pointerId);
    startY.current = e.clientY;
    setDrag({ id, dy: 0 });
  };

  const onPointerMove = (e) => {
    if (!drag) return;
    setDrag({ id: drag.id, dy: e.clientY - startY.current });
    setSlot(slotFromY(e.clientY));
  };

  const onPointerUp = () => {
    if (drag && slot !== null && !isNoop(from, slot)) {
      setItems((list) => moveItem(list, from, slotToIndex(from, slot)));
    }
    endDrag();
  };

  return (
    <>
      <p id="sort-help" className="sr-only">
        Press Space to grab an item, arrow keys to move it, Space to drop, Escape to cancel.
      </p>
      <ul className="sortable">
        {items.map((item, index) => {
          const isDragging = drag?.id === item.id;
          const cls = [
            'item',
            isDragging && 'floating',
            keyboard.grabbedId === item.id && 'grabbed',
            shownSlot === index && 'drop-before',
            shownSlot === items.length && index === items.length - 1 && 'drop-after',
          ]
            .filter(Boolean)
            .join(' ');
          return (
            <li
              key={item.id}
              className={cls}
              style={isDragging ? { transform: `translateY(${drag.dy}px)` } : undefined}
              ref={(el) => {
                if (el) itemEls.current.set(item.id, el);
                else itemEls.current.delete(item.id);
              }}
            >
              <button
                type="button"
                className="handle"
                aria-label={`Reorder ${item.label}`}
                aria-describedby="sort-help"
                {...keyboard.handleProps(item, index)}
                onPointerDown={(e) => onPointerDown(e, item.id)}
                onPointerMove={onPointerMove}
                onPointerUp={onPointerUp}
                onPointerCancel={endDrag}
              >
                ⋮⋮
              </button>
              {item.label}
            </li>
          );
        })}
      </ul>
      <div role="status" aria-live="assertive" className="sr-only">
        {keyboard.message}
      </div>
    </>
  );
}
```

**`App.jsx`**

```jsx
import NativeSortableList from './NativeSortableList';
import SortableList from './SortableList';
import './styles.css';

const FRUITS = ['Apple', 'Banana', 'Cherry', 'Durian', 'Elderberry'].map((label, i) => ({
  id: `f${i + 1}`,
  label,
}));

export default function App() {
  return (
    <main className="app">
      <h1>Sortable lists</h1>
      <section aria-labelledby="native-h">
        <h2 id="native-h">1. Native HTML5 DnD (mouse only)</h2>
        <NativeSortableList initialItems={FRUITS} />
      </section>
      <section aria-labelledby="pointer-h">
        <h2 id="pointer-h">2. Pointer Events + keyboard (touch friendly)</h2>
        <SortableList initialItems={FRUITS} />
      </section>
    </main>
  );
}
```

**`styles.css`**

```css
* { box-sizing: border-box; }
body { margin: 0; font-family: system-ui, sans-serif; background: #f1f5f9; color: #0f172a; }
.app { max-width: 420px; margin: 0 auto; padding: 24px 16px; }
h2 { font-size: 16px; margin: 24px 0 10px; }

.sortable { list-style: none; margin: 0; padding: 0; }
.item {
  position: relative;
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 8px;
  padding: 10px 12px;
  background: #fff;
  border: 1px solid #cbd5e1;
  border-radius: 8px;
  user-select: none;
}
.grip { color: #94a3b8; cursor: grab; }
.item[draggable='true'] { cursor: grab; }

.handle {
  padding: 4px 6px;
  border: 0;
  background: none;
  color: #64748b;
  font-size: 16px;
  letter-spacing: -2px;
  cursor: grab;
  touch-action: none;
}
.handle:focus-visible { outline: 3px solid #f59e0b; outline-offset: 2px; border-radius: 4px; }

.item.dragging { opacity: 0.5; }
.item.floating { z-index: 2; box-shadow: 0 8px 20px rgba(15, 23, 42, 0.25); }
.item.grabbed { border-color: #2563eb; box-shadow: 0 0 0 3px rgba(37, 99, 235, 0.35); }

.item.drop-before::before,
.item.drop-after::after {
  content: '';
  position: absolute;
  left: 0;
  right: 0;
  height: 4px;
  border-radius: 2px;
  background: #2563eb;
  pointer-events: none;
}
.item.drop-before::before { top: -6px; }
.item.drop-after::after { bottom: -6px; }

.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  margin: -1px;
  padding: 0;
  overflow: hidden;
  clip: rect(0 0 0 0);
  white-space: nowrap;
  border: 0;
}
```

**`DndKitSortableList.jsx`** — comparison (needs `npm i @dnd-kit/core @dnd-kit/sortable @dnd-kit/utilities`)

```jsx
import { useState } from 'react';
import {
  DndContext,
  closestCenter,
  KeyboardSensor,
  PointerSensor,
  useSensor,
  useSensors,
} from '@dnd-kit/core';
import {
  SortableContext,
  arrayMove,
  sortableKeyboardCoordinates,
  useSortable,
  verticalListSortingStrategy,
} from '@dnd-kit/sortable';
import { CSS } from '@dnd-kit/utilities';

function Row({ item }) {
  const { attributes, listeners, setNodeRef, setActivatorNodeRef, transform, transition, isDragging } =
    useSortable({ id: item.id });

  return (
    <li
      ref={setNodeRef}
      className={`item ${isDragging ? 'dragging' : ''}`}
      style={{ transform: CSS.Transform.toString(transform), transition }}
    >
      <button
        type="button"
        ref={setActivatorNodeRef}
        className="handle"
        aria-label={`Reorder ${item.label}`}
        {...attributes}
        {...listeners}
      >
        ⋮⋮
      </button>
      {item.label}
    </li>
  );
}

export default function DndKitSortableList({ initialItems }) {
  const [items, setItems] = useState(initialItems);
  const sensors = useSensors(
    useSensor(PointerSensor, { activationConstraint: { distance: 5 } }),
    useSensor(KeyboardSensor, { coordinateGetter: sortableKeyboardCoordinates })
  );

  const onDragEnd = ({ active, over }) => {
    if (!over || active.id === over.id) return;
    setItems((list) =>
      arrayMove(
        list,
        list.findIndex((i) => i.id === active.id),
        list.findIndex((i) => i.id === over.id)
      )
    );
  };

  return (
    <DndContext sensors={sensors} collisionDetection={closestCenter} onDragEnd={onDragEnd}>
      <SortableContext items={items} strategy={verticalListSortingStrategy}>
        <ul className="sortable">
          {items.map((item) => (
            <Row key={item.id} item={item} />
          ))}
        </ul>
      </SortableContext>
    </DndContext>
  );
}
```

</details>

---

## 6. How It Works

- **One state, one mutation function.** `items` only changes through `moveItem` (copy → `splice` out → `splice` in), whether the trigger is a native drop, a pointer release or an arrow key. Drag-session state is separate and thrown away on drop/cancel.
- **Slots, not "target item".** Hovering item `i` yields slot `i` (upper half) or `i + 1` (lower half). `slotToIndex` converts a slot in the *current* list to the final index after the dragged item is removed (`slot > from ? slot - 1 : slot`) — this is the off-by-one everyone gets wrong. `isNoop` hides the indicator and skips the update for the two slots adjacent to the dragged item.
- **Native DnD rules.** Elements reject drops unless `dragover` calls `preventDefault()`. `dataTransfer.setData` is needed for Firefox to start a drag. The *payload is unreadable during `dragover`* (only on `drop`), so the dragged id is kept in state. `dragover` fires every few ms — `setSlot` with an unchanged value bails out, so there are no render storms.
- **Foreign drags** (a file from the desktop) have `dragId === null`; handlers bail *before* `preventDefault`, so the browser keeps its default behaviour.
- **`dragleave` fires when entering children**, hence `contains(relatedTarget)` on the list. `dragend` resets the state for drops outside the list (it fires on the source element even without a valid drop target).
- **Zero layout-shift indicator.** A pseudo-element positioned in the item's margin gap (`top: -6px`) — nothing moves under the cursor, so `dragover` doesn't oscillate.
- **Pointer Events version.** `pointerdown` on the handle calls `setPointerCapture`, so all later `pointermove`/`pointerup` for this pointer go to the handle even when it leaves the element. `touch-action: none` on the handle stops the browser from converting the gesture into page scroll (and firing `pointercancel`). The dragged row follows the finger via `transform: translateY(dy)` (no layout), while the other rows stay put; their rects are used to compute the slot and the dragged row is skipped in the loop.
- **Keyboard version.** A *grab* state machine on the handle button: idle → grabbed (Space) → arrows call `moveItem(index, index±1)` immediately (live reorder, no indicator needed) → Space drops, Esc moves the item back to the remembered `origin`, Tab drops. A `useLayoutEffect` re-focuses the handle after each move because moving a DOM node can blur it. A `role="status"` region announces every transition.
- **Keys must be stable ids** — array indexes as keys would make React reuse DOM nodes and the dragged element's state would jump to another item.
- **dnd-kit** replaces all of the above plumbing: `useSortable` registers the row, sensors translate pointer/keyboard input into drag events, `closestCenter` does collision detection, and `transform`/`transition` animate siblings; we still own `items` and apply `arrayMove` in `onDragEnd`.

---

## 7. Edge Cases

- Dropping on the original position (slot `from` or `from + 1`) → no state change, no indicator.
- Drop outside the list → `dragend` resets; the list is unchanged. Pressing Esc mid native drag behaves the same.
- Drag from the file system / another page → `dragId` is `null`; nothing is prevented or highlighted.
- Moving down within the list → final index is `slot - 1` (the removed item shifts everything left).
- Dropping into the gap between items → the `<ul>` `dragover` keeps it a valid target; the last computed slot is used.
- Pointer leaves the list while dragging (pointer version) → capture keeps events flowing; the slot is clamped to `0 … n` by `slotFromY`.
- `pointercancel` (system gesture, incoming call, scroll takeover) → state reset without committing.
- Right-click / secondary mouse button on the handle → ignored (`e.button !== 0`).
- Single-item or empty list → `isNoop`/`moveItem` guard it; arrows at the ends do nothing.
- Keyboard: ↑ on the first / ↓ on the last item is ignored; Escape after several moves restores the *original* index; Tab while grabbed drops the item.
- Long lists that scroll → native DnD auto-scrolls near edges; the pointer version needs manual auto-scroll (follow-up).
- Touch: long-press on a draggable `<li>` may select text / open the context menu → `user-select: none` and a dedicated handle.
- StrictMode: all updates are pure functional `setItems` calls; no ids generated in updaters.
- Items re-fetched while dragging → `findIndex` returns `-1`; `moveItem` returns the list unchanged for out-of-range indexes.

---

## 8. Accessibility

- **Native HTML5 DnD is not keyboard accessible and not exposed to screen readers** — that is why the second list adds the Space / arrows / Esc interaction. Never ship drag-only reordering.
- The handle is a real `<button>` (focusable, Space/Enter work) with an explicit `aria-label` ("Reorder Banana"), `aria-pressed` for the grabbed state and `aria-describedby` pointing to the instructions.
- Rows are `<li>` inside `<ul>`: "list, 5 items" is announced and positions are meaningful.
- A visually hidden `role="status"` / `aria-live="assertive"` region announces *grabbed / moved to position X of N / dropped / cancelled* (assertive because the user is mid-interaction).
- Focus is **kept on the moved item's handle** after every keyboard move.
- Visible focus ring (`:focus-visible`) and a distinct "grabbed" style that isn't colour-only (outline + shadow).
- The drop indicator is a pseudo-element (not in the accessibility tree).
- Respect `prefers-reduced-motion` if you add sibling animations.
- Provide alternative controls (Move up / Move down buttons, or a position `<select>`) for users who can't perform drag or key chords — this is WCAG 2.5.7 "Dragging Movements".

---

## 9. Performance Considerations

- `dragover` / `pointermove` are high-frequency. Native: `setSlot` bails when the value is unchanged. Pointer: `dy` changes every move, so only the list re-renders — wrap `<li>` in `React.memo` with stable callbacks for hundreds of rows, or apply `dy` by writing `el.style.transform` through a ref (no re-render) and only `setState` when the slot changes.
- `slotFromY` reads `getBoundingClientRect` for up to `n` rows per move (forces layout once, then cached since nothing is written in between). For big lists cache rects at drag start, or binary-search them.
- Transforms (`translateY`) and pseudo-elements don't trigger layout of siblings → smooth 60 fps.
- Avoid heavy work in `dragstart`: the browser snapshots the element for the drag image. Use `setDragImage` for a custom, lighter preview.
- 1000+ items: virtualize (`react-window`) — then dragging beyond the viewport needs auto-scroll and index-based (not DOM-based) slot calculation.
- Persist (localStorage / API) once on commit, never during `dragover`/`pointermove`.
- Library comparison: dnd-kit adds ~10–15 kB gz but brings measuring, auto-scroll, sibling animation and a11y — build your own only when told "no libraries" or for tiny needs.

---

## 10. Follow-up Questions

<details>
<summary><strong>1. Why doesn't <code>drop</code> fire for me?</strong></summary>

`dragover` on the target didn't call `event.preventDefault()` — its default action is "not a valid drop target". Also check `draggable="true"` on the source and `dataTransfer.setData(...)` in `dragstart` (Firefox needs it). Remember that every element in between (the `<li>` and the gaps in the `<ul>`) must allow the drop.

</details>

<details>
<summary><strong>2. Why not use native DnD on mobile?</strong></summary>

Touch browsers historically don't fire the HTML5 drag events for touch input (iOS Safari only recently supports some cases, and behaviour varies), and long-press competes with text selection/context menus. Pointer Events give one API for mouse, pen and touch; `touch-action: none` plus `setPointerCapture` make it reliable.

</details>

<details>
<summary><strong>3. How would you add auto-scroll while dragging near the container edge?</strong></summary>

In `pointermove`, if `clientY` is within ~40 px of the scroll container's top/bottom, start a `requestAnimationFrame` loop that does `container.scrollTop += speed` (speed proportional to proximity) and recomputes the slot each frame (the rects move as it scrolls); stop the loop on `pointerup`/`cancel` or when the pointer leaves the zone. Account for scroll when positioning the floating row (store `startScrollTop`).

</details>

<details>
<summary><strong>4. How do you reorder between multiple lists (Kanban)?</strong></summary>

Keep `lists: { [listId]: items[] }`; the slot becomes `{ listId, index }`. Item handlers compute the in-list index, list containers handle empty-list drops (index 0). The move is "remove from source list, insert into target list" — same `slotToIndex` correction only when source and target lists are the same.

</details>

<details>
<summary><strong>5. How would you persist order on a backend efficiently?</strong></summary>

Send only the moved item with its new neighbours (`PATCH /items/:id { afterId }`) or use **fractional indexing** (`position = (prev + next) / 2` / LexoRank strings) so one row changes instead of rewriting every index. Apply the move optimistically and roll back (restore the previous `items`) if the request fails.

</details>

<details>
<summary><strong>6. Live reorder (items slide out of the way) instead of a drop line?</strong></summary>

Reorder `items` during the drag (call `moveItem` when the pointer crosses a sibling's midpoint) and animate with FLIP: record each row's `top` before the update, then in `useLayoutEffect` apply `transform: translateY(old - new)` and transition to `0`. Keep a snapshot of the original order to support cancel. dnd-kit's `SortableContext` does this with `transform`/`transition` from `useSortable`.

</details>

<details>
<summary><strong>7. Why the <code>slot &gt; from ? slot - 1 : slot</code> adjustment?</strong></summary>

The slot is measured against the *current* list that still contains the dragged item. After removing it, every index greater than `from` shifts left by one, so a downward move must target `slot - 1`. Upward moves (`slot <= from`) are unaffected. Forgetting it drops the item one position too far down.

</details>

<details>
<summary><strong>8. How would you test drag and drop?</strong></summary>

Unit-test `moveItem` / `slotToIndex` / `isNoop` exhaustively (they hold all the logic). For components, jsdom has no layout, so mock `getBoundingClientRect` per row and fire `dragStart` → `dragOver` (with `clientY`) → `drop` using `fireEvent` with a fake `dataTransfer`; for pointer and keyboard fire `pointerDown/Move/Up` and `keyDown` with `{ key: ' ' }`, `{ key: 'ArrowDown' }`. Use Playwright for real-browser drag interactions.

</details>

---

## 11. Common Mistakes

- Forgetting `e.preventDefault()` in `dragover` → `drop` never fires.
- Not calling `dataTransfer.setData` → drag doesn't start in Firefox.
- Off-by-one when moving an item down (not subtracting 1 after removal).
- Mutating the state array in place (`items.splice(...)`) instead of copying first.
- Using the array index as `key` → the DOM node "follows the index" rather than the item.
- Putting `dragId` / `slot` in persisted or global state; storing the dragged item's *index* and not its id (stale after reorder).
- Shifting layout for the indicator (inserting a real element / `flex gap`) → `dragover` ping-pong flicker.
- Handling `dragleave` naively → flicker when crossing child elements.
- Not bailing for foreign drags (files) → hijacking the browser's default behaviour.
- Pointer version without `setPointerCapture` / `touch-action: none` → drag stops at the element edge, touch scrolls the page and fires `pointercancel`.
- No cleanup on `pointercancel` / `dragend` → stuck "dragging" state.
- Drag-only UI with no keyboard alternative or live-region announcements.
- Losing focus after a keyboard move (the node is re-inserted) and not restoring it.
- Computing the target from `event.target` instead of the item's rect midpoint → jumpy drop positions on child elements.
