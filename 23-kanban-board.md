# Kanban Board — Machine Coding

> **Difficulty:** Hard | **Time:** 60–75 min | **Asked at / Frequency:** Frequent for senior-ish rounds (Atlassian/Trello-style, Rippling, Postman, Razorpay, Uber) | **Tier:** 2 (high value)

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

Build a **Kanban board** with three columns (To Do, In Progress, Done). Users can add cards to a column, delete cards, and **drag & drop** cards between columns and to a specific position within a column. Use the **native HTML5 Drag and Drop API** (no `react-beautiful-dnd`, `dnd-kit`, etc.) and manage state with `useReducer`.

---

## 2. Requirements

### Functional

- 3 columns, each with a title, a list of cards, and an "Add card" form.
- Add a card (text required, trimmed); delete a card.
- Drag a card and drop it:
  - into another column, at the **exact position** indicated by a drop line;
  - within the same column to reorder.
- Visual feedback: dragged card is dimmed, drop indicator line shows where it will land, column highlights on hover.
- Keyboard-accessible alternative to dragging (move left / right buttons).
- State persists across reloads (localStorage).

### Non-functional

- Reducer is pure and immutable; one `MOVE_CARD` action handles all moves.
- Dragging over cards must not cause flicker or layout thrash.
- Ignore foreign drags (files, text) that aren't cards.
- No third-party libraries.

---

## 3. Clarifying Questions to Ask

- Fixed columns or can users add/rename/reorder columns?
- Reorder within a column, or only move across columns?
- Do we need touch support? (Native HTML5 DnD doesn't work on most touch devices.)
- Card content: just text, or title + description + labels?
- Persist where? Multi-user real-time? (then optimistic updates + ordering strategy matter)
- Keyboard accessibility expectations?
- Maximum cards per column (virtualization)?

---

## 4. Approach & Component Design

**State shape**

```js
{
  columns: [
    { id: 'todo',  title: 'To Do',       cards: [{ id: 'c1', text: 'Design login' }] },
    { id: 'doing', title: 'In Progress', cards: [] },
    { id: 'done',  title: 'Done',        cards: [] },
  ]
}
// UI-only (not in reducer): dragId, over = { columnId, index }
```

Nested arrays are fine for 3 columns. For large boards, normalise (`cardsById`, `column.cardIds`) so a move only touches two small id arrays.

**Component tree**

```
<Board>                    owns reducer + drag state (dragId, over)
  └─ <Column /> x 3        drop target; renders cards + indicator + <AddCardForm />
       └─ <Card /> x n     draggable; keyboard move buttons
```

**HTML5 DnD event map**

| Event | On | We do |
| --- | --- | --- |
| `dragstart` | card | `setData`, `effectAllowed='move'`, `setDragId` |
| `dragover` | card | `preventDefault()` (allows drop), compute before/after via mouse Y vs card midpoint → `over` |
| `dragover` | column | `preventDefault()`, if empty area → `over` = end of column |
| `dragleave` | column | clear `over` if leaving the column entirely |
| `drop` | column | dispatch `MOVE_CARD`, reset drag state |
| `dragend` | card | reset drag state (drop outside any column) |

> A drop target only accepts a drop if `dragover` calls `e.preventDefault()` — the #1 reason "drop never fires".

**Drop index maths:** hovering over card `i`: `index = i + (mouseY > midpoint ? 1 : 0)` → insertion index in the *current* list. The reducer corrects for the removed card when moving down within the same column.

---

## 5. Reference Solution

<details>
<summary><strong>Show Full Code</strong></summary>

**`boardReducer.js`**

```js
export const initialBoard = {
  columns: [
    {
      id: 'todo',
      title: 'To Do',
      cards: [
        { id: 'c1', text: 'Design login page' },
        { id: 'c2', text: 'Set up CI pipeline' },
        { id: 'c3', text: 'Write API docs' },
      ],
    },
    { id: 'doing', title: 'In Progress', cards: [{ id: 'c4', text: 'Build kanban board' }] },
    { id: 'done', title: 'Done', cards: [] },
  ],
};

export function boardReducer(state, action) {
  switch (action.type) {
    case 'ADD_CARD':
      return {
        columns: state.columns.map((col) =>
          col.id === action.columnId
            ? { ...col, cards: [...col.cards, { id: action.id, text: action.text }] }
            : col
        ),
      };

    case 'DELETE_CARD':
      return {
        columns: state.columns.map((col) => ({
          ...col,
          cards: col.cards.filter((c) => c.id !== action.cardId),
        })),
      };

    case 'MOVE_CARD': {
      const { cardId, toColumnId, toIndex } = action;
      // Derive the source from state, not the payload — it can't go stale.
      const from = state.columns.find((col) => col.cards.some((c) => c.id === cardId));
      const to = state.columns.find((col) => col.id === toColumnId);
      if (!from || !to) return state;

      const fromIndex = from.cards.findIndex((c) => c.id === cardId);
      const card = from.cards[fromIndex];
      let insertAt = Math.max(0, Math.min(toIndex, to.cards.length));

      if (from.id === to.id) {
        // Dropping right before/after itself is a no-op.
        if (insertAt === fromIndex || insertAt === fromIndex + 1) return state;
        // Removing the card shifts everything after it left by one.
        if (insertAt > fromIndex) insertAt -= 1;
      }

      return {
        columns: state.columns.map((col) => {
          let cards = col.cards;
          if (col.id === from.id) cards = cards.filter((c) => c.id !== cardId);
          if (col.id === to.id) {
            cards = [...cards.slice(0, insertAt), card, ...cards.slice(insertAt)];
          }
          return cards === col.cards ? col : { ...col, cards };
        }),
      };
    }

    default:
      return state;
  }
}

const STORAGE_KEY = 'kanban:v1';

export function loadBoard() {
  try {
    const data = JSON.parse(localStorage.getItem(STORAGE_KEY));
    const valid =
      data &&
      Array.isArray(data.columns) &&
      data.columns.every((c) => typeof c.id === 'string' && Array.isArray(c.cards));
    return valid ? data : initialBoard;
  } catch {
    return initialBoard;
  }
}

export function saveBoard(board) {
  try {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(board));
  } catch {
    /* ignore quota / privacy-mode errors */
  }
}
```

**`Board.jsx`**

```jsx
import { Fragment, useEffect, useReducer, useState } from 'react';
import { boardReducer, initialBoard, loadBoard, saveBoard } from './boardReducer';
import './kanban.css';

const uid = () => crypto.randomUUID?.() ?? `${Date.now()}-${Math.random().toString(36).slice(2)}`;

export default function Board() {
  const [board, dispatch] = useReducer(boardReducer, initialBoard, loadBoard);
  const [dragId, setDragId] = useState(null);
  const [over, setOver] = useState(null); // { columnId, index }

  useEffect(() => {
    saveBoard(board);
  }, [board]);

  const resetDrag = () => {
    setDragId(null);
    setOver(null);
  };

  const handleDragStart = (e, cardId) => {
    e.dataTransfer.setData('text/plain', cardId); // Firefox needs data to start a drag
    e.dataTransfer.effectAllowed = 'move';
    setDragId(cardId);
  };

  const handleCardDragOver = (e, columnId, index) => {
    if (!dragId) return; // not one of our cards
    e.preventDefault(); // required to allow dropping
    e.stopPropagation(); // the column handler must not override the precise index
    const rect = e.currentTarget.getBoundingClientRect();
    const after = e.clientY > rect.top + rect.height / 2;
    const next = index + (after ? 1 : 0);
    setOver((prev) =>
      prev && prev.columnId === columnId && prev.index === next ? prev : { columnId, index: next }
    );
  };

  // Hovering empty space / gaps / empty column => drop at the end.
  const handleColumnDragOver = (e, column) => {
    if (!dragId) return;
    e.preventDefault();
    setOver((prev) =>
      prev && prev.columnId === column.id ? prev : { columnId: column.id, index: column.cards.length }
    );
  };

  const handleColumnDragLeave = (e, columnId) => {
    if (e.currentTarget.contains(e.relatedTarget)) return; // moved onto a child
    setOver((prev) => (prev && prev.columnId === columnId ? null : prev));
  };

  const handleDrop = (e, column) => {
    e.preventDefault();
    if (dragId) {
      const toIndex = over && over.columnId === column.id ? over.index : column.cards.length;
      dispatch({ type: 'MOVE_CARD', cardId: dragId, toColumnId: column.id, toIndex });
    }
    resetDrag(); // the source node may unmount, so don't rely on dragend alone
  };

  return (
    <div className="board">
      {board.columns.map((column, i) => (
        <Column
          key={column.id}
          column={column}
          prevId={board.columns[i - 1]?.id}
          nextId={board.columns[i + 1]?.id}
          dragId={dragId}
          over={over?.columnId === column.id ? over : null}
          onDragStart={handleDragStart}
          onDragEnd={resetDrag}
          onCardDragOver={handleCardDragOver}
          onColumnDragOver={handleColumnDragOver}
          onColumnDragLeave={handleColumnDragLeave}
          onDrop={handleDrop}
          onAdd={(text) => dispatch({ type: 'ADD_CARD', columnId: column.id, id: uid(), text })}
          onDelete={(cardId) => dispatch({ type: 'DELETE_CARD', cardId })}
          onMove={(cardId, toColumnId) =>
            dispatch({
              type: 'MOVE_CARD',
              cardId,
              toColumnId,
              toIndex: board.columns.find((c) => c.id === toColumnId).cards.length,
            })
          }
        />
      ))}
    </div>
  );
}

function Column({
  column, prevId, nextId, dragId, over,
  onDragStart, onDragEnd, onCardDragOver, onColumnDragOver, onColumnDragLeave,
  onDrop, onAdd, onDelete, onMove,
}) {
  const dragIndex = column.cards.findIndex((c) => c.id === dragId);
  // Hide indicators that would describe a no-op move (directly above/below the dragged card).
  const showIndicatorAt = (i) =>
    over && over.index === i && !(dragIndex !== -1 && (i === dragIndex || i === dragIndex + 1));

  return (
    <section
      className={`column${over ? ' column--over' : ''}`}
      aria-labelledby={`col-${column.id}`}
      onDragOver={(e) => onColumnDragOver(e, column)}
      onDragLeave={(e) => onColumnDragLeave(e, column.id)}
      onDrop={(e) => onDrop(e, column)}
    >
      <h2 id={`col-${column.id}`} className="column__title">
        {column.title} <span className="column__count">{column.cards.length}</span>
      </h2>

      <ul className="cards">
        {column.cards.map((card, i) => (
          <Fragment key={card.id}>
            {showIndicatorAt(i) && <li className="drop-indicator" aria-hidden="true" />}
            <Card
              card={card}
              index={i}
              columnId={column.id}
              isDragging={dragId === card.id}
              prevId={prevId}
              nextId={nextId}
              onDragStart={onDragStart}
              onDragEnd={onDragEnd}
              onDragOver={onCardDragOver}
              onDelete={onDelete}
              onMove={onMove}
            />
          </Fragment>
        ))}
        {showIndicatorAt(column.cards.length) && <li className="drop-indicator" aria-hidden="true" />}
      </ul>

      <AddCardForm columnTitle={column.title} onAdd={onAdd} />
    </section>
  );
}

function Card({ card, index, columnId, isDragging, prevId, nextId, onDragStart, onDragEnd, onDragOver, onDelete, onMove }) {
  return (
    <li
      className={`card${isDragging ? ' card--dragging' : ''}`}
      draggable
      onDragStart={(e) => onDragStart(e, card.id)}
      onDragEnd={onDragEnd}
      onDragOver={(e) => onDragOver(e, columnId, index)}
    >
      <span className="card__text">{card.text}</span>
      <span className="card__actions">
        <button type="button" disabled={!prevId} aria-label={`Move "${card.text}" to previous column`} onClick={() => onMove(card.id, prevId)}>
          ←
        </button>
        <button type="button" disabled={!nextId} aria-label={`Move "${card.text}" to next column`} onClick={() => onMove(card.id, nextId)}>
          →
        </button>
        <button type="button" aria-label={`Delete "${card.text}"`} onClick={() => onDelete(card.id)}>
          ×
        </button>
      </span>
    </li>
  );
}

function AddCardForm({ columnTitle, onAdd }) {
  const [text, setText] = useState('');

  const handleSubmit = (e) => {
    e.preventDefault();
    const trimmed = text.trim();
    if (!trimmed) return;
    onAdd(trimmed);
    setText('');
  };

  return (
    <form className="add-card" onSubmit={handleSubmit}>
      <input
        value={text}
        onChange={(e) => setText(e.target.value)}
        placeholder="Add a card…"
        aria-label={`New card in ${columnTitle}`}
      />
      <button type="submit" disabled={!text.trim()}>
        Add
      </button>
    </form>
  );
}
```

**`App.jsx`**

```jsx
import Board from './Board';

export default function App() {
  return (
    <main>
      <h1 style={{ fontFamily: 'system-ui', margin: '16px' }}>Kanban</h1>
      <Board />
    </main>
  );
}
```

**`kanban.css`**

```css
* { box-sizing: border-box; }
body { margin: 0; font-family: system-ui, sans-serif; background: #f1f5f9; }

.board {
  display: grid;
  grid-template-columns: repeat(3, minmax(240px, 1fr));
  gap: 16px;
  padding: 16px;
  align-items: start;
}

.column {
  background: #e2e8f0;
  border-radius: 8px;
  padding: 12px;
  min-height: 160px;
  border: 2px solid transparent;
  transition: border-color 120ms, background 120ms;
}
.column--over { border-color: #2563eb; background: #dbeafe; }

.column__title { margin: 0 0 12px; font-size: 16px; display: flex; gap: 8px; align-items: center; }
.column__count { font-size: 12px; background: #cbd5e1; border-radius: 10px; padding: 1px 8px; }

/* Margins (not flex gap) so the zero-height indicator causes no layout shift. */
.cards { list-style: none; margin: 0 0 8px; padding: 0; min-height: 24px; }

.card {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  gap: 8px;
  margin-bottom: 8px;
  padding: 10px;
  background: #fff;
  border-radius: 6px;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.15);
  cursor: grab;
  user-select: none;
}
.card:active { cursor: grabbing; }
.card--dragging { opacity: 0.4; }
.card__text { word-break: break-word; }
.card__actions { display: flex; gap: 2px; flex: none; }
.card__actions button { border: 0; background: transparent; cursor: pointer; font-size: 16px; padding: 0 4px; }
.card__actions button:disabled { opacity: 0.25; cursor: default; }

.drop-indicator { height: 0; position: relative; list-style: none; }
.drop-indicator::after {
  content: '';
  position: absolute;
  left: 0; right: 0; top: -5px;
  height: 4px;
  border-radius: 2px;
  background: #2563eb;
  pointer-events: none;
}

.add-card { display: flex; gap: 6px; }
.add-card input { flex: 1; min-width: 0; padding: 6px 8px; }
```

</details>

---

## 6. How It Works

- **Two kinds of state.** The *board* (domain data) lives in a reducer and is persisted. The *drag session* (`dragId`, `over`) is transient UI state in `Board` and never saved.
- **One `MOVE_CARD` action** covers cross-column moves, reordering and keyboard moves. It finds the source from state (not payload), removes the card from the source, inserts it into the target — and for same-column downward moves subtracts 1 from the target index, because removing the card shifts later items left. No-op moves (drop directly above/below itself) return the same `state` reference → no re-render.
- **Why `preventDefault` in `dragover`?** By default elements reject drops; cancelling `dragover` marks the element as a valid target, otherwise `drop` never fires.
- **Computing the insertion index.** `dragover` on card `i` compares `clientY` to the card's vertical midpoint → insert before (`i`) or after (`i+1`). `stopPropagation` keeps the column's coarse handler (end-of-list) from overwriting it. The column handler only sets `over` when it isn't already set for that column, so hovering gaps doesn't flicker between "precise" and "end".
- **Zero-layout-shift indicator.** A `height: 0` `<li>` with an absolutely-positioned line; cards separated by `margin-bottom` (not `gap`) so inserting the indicator doesn't push cards under the cursor and cause `dragover` ping-pong.
- **`setData('text/plain', id)`** is required for Firefox to start a drag; we still read `dragId` from state (the `dataTransfer` payload isn't readable during `dragover`, only `drop`).
- **`dragleave` fires when entering child elements** — hence the `contains(relatedTarget)` check to avoid flicker.
- **Reset on both `drop` and `dragend`.** After a cross-column drop the original DOM node is unmounted (it's rendered in a different column), so `dragend` may never fire on it.
- **Keyboard buttons dispatch the same `MOVE_CARD`.**
- **IDs are generated in the event handler** (`uid()`), never inside the reducer, to keep it pure.

---

## 7. Edge Cases

- Dropping on the original position → reducer no-op (same reference); indicator hidden for those slots.
- Drop outside any column → `dragend` resets `dragId`/`over`.
- Drag a non-card payload (file from desktop) → `dragId` is `null`, handlers bail without `preventDefault`, so the browser keeps default behaviour.
- Dropping into an empty column → column `dragover` sets `index = 0` (length of empty list).
- Moving within the same column to the end → `insertAt = length` → adjusted to `length - 1` after removal.
- Very fast mouse movement across cards → `setOver` bails (returns `prev`) if the value is identical, avoiding render storms.
- Whitespace-only card text rejected.
- Delete while dragging (rare) → `MOVE_CARD` returns `state` since `from` isn't found.
- Corrupted localStorage → validated on load; falls back to the initial board.
- Safari/Firefox: `relatedTarget` on `dragleave` may be `null` → indicator flickers briefly but recovers on the next `dragover`.
- Touch devices: native DnD generally doesn't fire. Mention pointer-event based DnD (or the keyboard buttons) as a fallback.
- Scrolling a long column while dragging → browsers auto-scroll near edges for native DnD in scrollable containers.

---

## 8. Accessibility

- Native DnD is **not keyboard accessible**; the ← / → buttons provide an equivalent, with explicit `aria-label`s that include the card's text.
- Columns are `<section aria-labelledby>`; cards are `<li>` in a `<ul>` so screen readers announce "list, 3 items".
- Announce moves with a polite live region ("Moved 'Write docs' to Done, position 2 of 4") — an easy upgrade to mention.
- Drop indicator is `aria-hidden` (purely visual).
- Don't rely on colour alone for the over-column state; the border change helps and the indicator line is positional.
- Form input has an `aria-label` naming its column; the Add button is disabled for empty input.
- Maintain focus on the moved card after a keyboard move (a follow-up: refs keyed by card id, focus in `useEffect` after the move).

---

## 9. Performance Considerations

- `dragover` fires every few ms. We only `setOver` when the value changes (functional update returning `prev`), so most events cause no re-render.
- Wrap `Column`/`Card` in `React.memo` and stabilise callbacks (`useCallback`) if the board grows — currently inline lambdas in `Board` defeat memoization, which is fine for dozens of cards but worth calling out.
- The reducer preserves identity of untouched columns (`return cards === col.cards ? col : …`), so memoized columns skip re-render.
- For hundreds of cards, use a normalised state (`cardsById` + `cardIds`) and virtualize long columns.
- Avoid heavy work in `dragstart` (browsers snapshot the element for the drag image — it should be fast). Use `setDragImage` for custom previews.
- Debounce persistence if the save is expensive; writes occur once per committed change, not per `dragover`.

---

## 10. Follow-up Questions

<details>
<summary><strong>1. Why does `drop` never fire for me?</strong></summary>

Because `dragover` on the target didn't call `event.preventDefault()`. The default action of `dragover` is "not a valid drop target". Also check `draggable="true"` on the source and that `dataTransfer.setData` is called in `dragstart` (Firefox).

</details>

<details>
<summary><strong>2. How would you normalise the state for large boards?</strong></summary>

`{ cards: { [id]: {...} }, columns: { [id]: { id, title, cardIds: [] } }, columnOrder: [] }`. A move only rewrites two `cardIds` arrays; editing a card only replaces one entry in `cards`. Components select by id, so memoized cards don't re-render when unrelated cards move.

</details>

<details>
<summary><strong>3. How would you support touch devices?</strong></summary>

Native HTML5 DnD is unreliable on touch. Implement with Pointer Events: `pointerdown` → capture pointer, render a floating clone, on `pointermove` use `document.elementFromPoint` to find the target column/card, on `pointerup` dispatch `MOVE_CARD`. Or use `dnd-kit`, which does this plus keyboard support.

</details>

<details>
<summary><strong>4. How would you make column reordering draggable too?</strong></summary>

Differentiate drag types by `dataTransfer.setData('application/x-kanban', JSON.stringify({type:'column', id}))` (or a `dragType` state). Column `dragstart` sets `dragType='column'`; column `dragover`/`drop` compute index from `clientX`; add a `MOVE_COLUMN` action. Card handlers ignore column drags and vice-versa.

</details>

<details>
<summary><strong>5. How would you do optimistic updates with a backend?</strong></summary>

Dispatch `MOVE_CARD` immediately, send `PATCH /cards/:id {columnId, position}`, on failure dispatch a rollback with the snapshot (or refetch). For ordering, use fractional indexing (LexoRank-style) so a move changes one card's `position`, not every sibling's.

</details>

<details>
<summary><strong>6. Why is `dragend` not enough for cleanup?</strong></summary>

When a card is dropped into another column React unmounts the original element (different parent → new instance). Some browsers don't dispatch `dragend` on a detached node. Resetting in `drop` as well (and on `dragend` for drops outside) handles both.

</details>

<details>
<summary><strong>7. How do you add undo/redo?</strong></summary>

Wrap the reducer: `{ past: [], present, future: [] }`. Every undoable action pushes `present` to `past`; `UNDO` pops from `past` to `present` and pushes to `future`. Immutable updates + structural sharing make snapshots cheap.

</details>

<details>
<summary><strong>8. How would you add inline card editing?</strong></summary>

Add `EDIT_CARD {cardId, text}`; track `editingId` in `Board` state; render an input in `Card` when editing; turn off `draggable` while editing (otherwise text selection in the input drags the card); commit on Enter/blur, cancel on Escape.

</details>

---

## 11. Common Mistakes

- Forgetting `e.preventDefault()` in `dragover` → no drops.
- Not calling `setData` → drag doesn't start in Firefox.
- Mutating arrays (`splice` on state) when moving cards.
- Off-by-one when moving down within the same column (not adjusting after removal).
- Keeping drag state in the reducer / persisting it.
- Using array index as `key` for cards → wrong DOM reuse; dragged element state jumps.
- Using `flex gap` plus a visible placeholder that shifts layout → `dragover` oscillation/flicker.
- Handling `dragleave` naively → flicker when crossing child elements.
- Relying only on `dragend` to clear state.
- Not stopping propagation from card `dragover`, so the column handler overwrites the precise index.
- Generating ids (`Math.random()`) inside the reducer → impure under Strict Mode.
- No keyboard alternative.

