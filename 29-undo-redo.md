# Undo / Redo (useHistory) — Machine Coding

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

Build a reusable **`useHistory(initialValue)`** hook that gives any piece of state **undo / redo**, then prove it with demos:

- a **sketch pad** (freehand strokes — one stroke = one undo step, even though it is hundreds of `pointermove`s),
- a **notes textarea** (typing is coalesced by a debounce — a pause = one undo step),
- a **todo list** that uses the other common flavour: a **reducer wrapper** `withUndo(reducer)`.

Keyboard shortcuts: **Ctrl/Cmd + Z** (undo), **Ctrl/Cmd + Shift + Z** and **Ctrl + Y** (redo).

---

## 2. Requirements

### Functional

- `useHistory` returns `present`, `set`, `undo`, `redo`, `reset`, `canUndo`, `canRedo` (+ `commit` for batching).
- `set(value)` accepts a value **or an updater function** (`set(prev => …)`), like `useState`.
- A new `set` after an undo **discards the redo branch**.
- **History limit** (default 100): the oldest entries are dropped.
- **Batching:** `set(v, { batch: true })` merges consecutive updates into one history entry until `commit()` (drag / draw). `set(v, { debounce: ms })` auto-commits after `ms` of inactivity (typing).
- `reset(value)` clears the history.
- Setting an identical value (`Object.is`) is a no-op — no junk history entry.
- Shortcuts work inside the demo panels (including inside the `<textarea>`, where we override the browser's native undo because the textarea is controlled).
- `withUndo(reducer, { limit, track })` — wraps any reducer and adds `UNDO` / `REDO` / `RESET` actions.

### Non-functional

- Pure, immutable updates; the history logic lives in a **pure reducer** that is easy to unit-test.
- Stable callbacks (`useCallback`) so consumers can memoize.
- Pending debounce timers are cleared on undo / reset / unmount.
- No third-party libraries.

---

## 3. Clarifying Questions to Ask

- Is undo **local to one component** or **global** (a single stack across the whole app)?
- What is "one step" — a keystroke, a word, a pause, a whole drag gesture?
- Should some changes (selection, zoom, scroll position) be **excluded** from history?
- Maximum history depth / memory budget? Large documents (images, 10k shapes)?
- Should history **persist** across reloads or sync between users (collaborative undo)?
- Do we need to display the history (a list of steps) and jump to any point?
- Snapshots or commands (operations that must be reversible)?

---

## 4. Approach & Component Design

**State shape — three-stack snapshot history**

```js
{ past: [v0, v1, v2], present: v3, future: [v4, v5], merging: false }
```

| Operation | Effect |
|---|---|
| `set(v)` | `past = [...past, present].slice(-limit)`, `present = v`, `future = []` |
| `undo()` | `present = past.at(-1)`, `past = past.slice(0,-1)`, `future = [present, ...future]` |
| `redo()` | `present = future[0]`, `past = [...past, present]`, `future = future.slice(1)` |
| `reset(v)` | `{ past: [], present: v, future: [] }` |
| `canUndo / canRedo` | derived: `past.length > 0` / `future.length > 0` |

**Batching with a `merging` flag** — the first `set(…, { batch: true })` pushes `present` into `past` and opens a batch; while open, later batched `set`s only *replace* `present`. `commit()` (pointer-up, debounce timer, undo, redo, reset) closes it. So a 300-event drag creates **one** history entry.

```
 pointerdown ──► set(batch) ─ push past ─┐
 pointermove ──► set(batch) ─ replace ───┤  1 history entry
 pointermove ──► set(batch) ─ replace ───┘
 pointerup   ──► commit()   ─ close batch
```

**Snapshot vs Command pattern**

| | Snapshot (this solution) | Command pattern |
|---|---|---|
| Stores | full previous states (shared structurally) | operations `{ do, undo }` / inverse patches |
| Undo | swap in the old state — trivially correct | run the inverse operation |
| Memory | O(states); cheap with immutable structural sharing, heavy for big blobs (images, canvases) | O(operations); tiny |
| Complexity | very low, works for *any* state | every action needs a correct inverse; easy to get wrong |
| Side effects | can't undo them (network, DOM) | can (compensating command) |
| Collaboration | hard (can't rebase snapshots) | natural (operational transforms / CRDT ops) |
| Use when | forms, todos, small editors, drawing of vectors | large docs, text editors, Figma-like tools, collaborative apps |

Middle road: store **patches** (Immer `produceWithPatches` → `patches` / `inversePatches`) — snapshot ergonomics with command-sized memory.

**Two ways to ship it**

- `useHistory(initial)` — a drop-in `useState` replacement (history owned by one component).
- `withUndo(reducer)` — wrap an existing reducer; every action it already handles becomes undoable and `track(action)` can opt actions out (e.g. `SELECT`).

**Component tree**

```
App
 ├─ SketchPad    useHistory([])         set(batch) on move, commit on pointerup
 │    └─ UndoToolbar (Undo · Redo · Clear)
 ├─ NotesEditor  useHistory('')         set(text, { debounce: 500 })
 │    └─ UndoToolbar
 └─ TodoList     useReducer(withUndo(todoReducer), initUndo(todos))
      └─ UndoToolbar

useUndoShortcuts({ undo, redo }) → onKeyDown handler attached to each panel
```

Shortcuts are attached to the **panel** (`onKeyDown`) rather than `window`, so three independent histories never fight over one keystroke. A global history would use one `window` listener.

---

## 5. Reference Solution

<details>
<summary><strong>Show Full Code</strong></summary>

**`useHistory.js`**

```jsx
import { useCallback, useEffect, useReducer, useRef } from 'react';

const resolve = (value, prev) => (typeof value === 'function' ? value(prev) : value);
const init = (present) => ({ past: [], present, future: [], merging: false });

export function historyReducer(state, action) {
  const { past, present, future, merging } = state;
  switch (action.type) {
    case 'SET': {
      const next = resolve(action.value, present);
      if (Object.is(next, present)) return state;
      if (action.merge && merging) return { ...state, present: next };
      return {
        past: [...past, present].slice(-action.limit),
        present: next,
        future: [],
        merging: action.merge,
      };
    }
    case 'COMMIT':
      return merging ? { ...state, merging: false } : state;
    case 'UNDO':
      if (past.length === 0) return state;
      return {
        past: past.slice(0, -1),
        present: past[past.length - 1],
        future: [present, ...future],
        merging: false,
      };
    case 'REDO':
      if (future.length === 0) return state;
      return {
        past: [...past, present],
        present: future[0],
        future: future.slice(1),
        merging: false,
      };
    case 'RESET':
      return init(action.value);
    default:
      return state;
  }
}

export function useHistory(initialValue, { limit = 100 } = {}) {
  const [state, dispatch] = useReducer(historyReducer, initialValue, init);
  const timer = useRef(null);

  useEffect(() => () => clearTimeout(timer.current), []);

  const set = useCallback(
    (value, { batch = false, debounce = 0 } = {}) => {
      clearTimeout(timer.current);
      dispatch({ type: 'SET', value, limit, merge: batch || debounce > 0 });
      if (debounce > 0) {
        timer.current = setTimeout(() => dispatch({ type: 'COMMIT' }), debounce);
      }
    },
    [limit]
  );

  const commit = useCallback(() => {
    clearTimeout(timer.current);
    dispatch({ type: 'COMMIT' });
  }, []);

  const undo = useCallback(() => {
    clearTimeout(timer.current);
    dispatch({ type: 'UNDO' });
  }, []);

  const redo = useCallback(() => {
    clearTimeout(timer.current);
    dispatch({ type: 'REDO' });
  }, []);

  const reset = useCallback((value) => {
    clearTimeout(timer.current);
    dispatch({ type: 'RESET', value });
  }, []);

  return {
    present: state.present,
    pastCount: state.past.length,
    futureCount: state.future.length,
    canUndo: state.past.length > 0,
    canRedo: state.future.length > 0,
    set,
    commit,
    undo,
    redo,
    reset,
  };
}
```

**`withUndo.js`**

```jsx
const UNDO = '@@history/UNDO';
const REDO = '@@history/REDO';
const RESET = '@@history/RESET';

export const undo = () => ({ type: UNDO });
export const redo = () => ({ type: REDO });
export const resetHistory = (present) => ({ type: RESET, present });
export const initUndo = (present) => ({ past: [], present, future: [] });

export function withUndo(reducer, { limit = 100, track = () => true } = {}) {
  return function undoable(state, action) {
    const { past, present, future } = state;
    switch (action.type) {
      case UNDO:
        if (past.length === 0) return state;
        return {
          past: past.slice(0, -1),
          present: past[past.length - 1],
          future: [present, ...future],
        };
      case REDO:
        if (future.length === 0) return state;
        return {
          past: [...past, present],
          present: future[0],
          future: future.slice(1),
        };
      case RESET:
        return initUndo(action.present);
      default: {
        const next = reducer(present, action);
        if (next === present) return state;
        if (!track(action)) return { ...state, present: next };
        return { past: [...past, present].slice(-limit), present: next, future: [] };
      }
    }
  };
}
```

**`useUndoShortcuts.js`**

```jsx
import { useCallback } from 'react';

export function useUndoShortcuts({ undo, redo }) {
  return useCallback(
    (e) => {
      if (!(e.ctrlKey || e.metaKey) || e.altKey) return;
      const key = e.key.toLowerCase();
      if (key === 'z') {
        e.preventDefault();
        if (e.shiftKey) redo();
        else undo();
      } else if (key === 'y') {
        e.preventDefault();
        redo();
      }
    },
    [undo, redo]
  );
}
```

**`UndoToolbar.jsx`**

```jsx
export default function UndoToolbar({ undo, redo, canUndo, canRedo, children }) {
  return (
    <div className="toolbar" role="toolbar" aria-label="History">
      <button type="button" onClick={undo} disabled={!canUndo} aria-keyshortcuts="Control+Z Meta+Z">
        Undo
      </button>
      <button
        type="button"
        onClick={redo}
        disabled={!canRedo}
        aria-keyshortcuts="Control+Shift+Z Control+Y Meta+Shift+Z"
      >
        Redo
      </button>
      {children}
    </div>
  );
}
```

**`SketchPad.jsx`**

```jsx
import { useRef } from 'react';
import { useHistory } from './useHistory';
import { useUndoShortcuts } from './useUndoShortcuts';
import UndoToolbar from './UndoToolbar';

export default function SketchPad() {
  const { present: strokes, set, commit, undo, redo, canUndo, canRedo, pastCount } = useHistory([]);
  const drawing = useRef(false);
  const nextId = useRef(1);
  const onKeyDown = useUndoShortcuts({ undo, redo });

  const toPoint = (e) => {
    const rect = e.currentTarget.getBoundingClientRect();
    return [Math.round(e.clientX - rect.left), Math.round(e.clientY - rect.top)];
  };

  const onPointerDown = (e) => {
    if (e.button !== 0) return;
    e.currentTarget.setPointerCapture?.(e.pointerId);
    drawing.current = true;
    const stroke = { id: nextId.current++, points: [toPoint(e)] };
    set((prev) => [...prev, stroke], { batch: true });
  };

  const onPointerMove = (e) => {
    if (!drawing.current) return;
    const p = toPoint(e);
    set(
      (prev) => {
        const last = prev[prev.length - 1];
        if (!last) return prev;
        return [...prev.slice(0, -1), { ...last, points: [...last.points, p] }];
      },
      { batch: true }
    );
  };

  const endStroke = () => {
    if (!drawing.current) return;
    drawing.current = false;
    commit();
  };

  return (
    <section className="panel" tabIndex={0} onKeyDown={onKeyDown} aria-labelledby="sketch-h">
      <h2 id="sketch-h">Sketch pad</h2>
      <UndoToolbar undo={undo} redo={redo} canUndo={canUndo} canRedo={canRedo}>
        <button type="button" onClick={() => set([])} disabled={strokes.length === 0}>
          Clear
        </button>
        <span className="meta" aria-live="polite">
          {strokes.length} stroke{strokes.length === 1 ? '' : 's'} · {pastCount} undo step
          {pastCount === 1 ? '' : 's'}
        </span>
      </UndoToolbar>
      <svg
        className="canvas"
        role="img"
        aria-label={`Drawing with ${strokes.length} strokes`}
        onPointerDown={onPointerDown}
        onPointerMove={onPointerMove}
        onPointerUp={endStroke}
        onPointerCancel={endStroke}
      >
        {strokes.map((s) => (
          <polyline
            key={s.id}
            points={(s.points.length === 1 ? [...s.points, s.points[0]] : s.points)
              .map((p) => p.join(','))
              .join(' ')}
          />
        ))}
      </svg>
    </section>
  );
}
```

**`NotesEditor.jsx`**

```jsx
import { useHistory } from './useHistory';
import { useUndoShortcuts } from './useUndoShortcuts';
import UndoToolbar from './UndoToolbar';

export default function NotesEditor() {
  const { present: text, set, undo, redo, canUndo, canRedo, pastCount } = useHistory('', {
    limit: 50,
  });
  const onKeyDown = useUndoShortcuts({ undo, redo });

  return (
    <section className="panel" aria-labelledby="notes-h">
      <h2 id="notes-h">Notes</h2>
      <UndoToolbar undo={undo} redo={redo} canUndo={canUndo} canRedo={canRedo}>
        <span className="meta">{pastCount} undo steps (a 500 ms pause starts a new step)</span>
      </UndoToolbar>
      <textarea
        aria-label="Note text"
        rows={6}
        value={text}
        onChange={(e) => set(e.target.value, { debounce: 500 })}
        onKeyDown={onKeyDown}
      />
    </section>
  );
}
```

**`todoReducer.js`**

```jsx
export const initialTodos = [
  { id: 1, text: 'Read about command pattern', done: false },
  { id: 2, text: 'Build useHistory', done: false },
];

export function todoReducer(todos, action) {
  switch (action.type) {
    case 'ADD':
      return [...todos, { id: action.id, text: action.text, done: false }];
    case 'TOGGLE':
      return todos.map((t) => (t.id === action.id ? { ...t, done: !t.done } : t));
    case 'REMOVE':
      return todos.filter((t) => t.id !== action.id);
    default:
      return todos;
  }
}
```

**`TodoList.jsx`**

```jsx
import { useReducer, useRef, useState } from 'react';
import { withUndo, undo, redo, initUndo } from './withUndo';
import { todoReducer, initialTodos } from './todoReducer';
import { useUndoShortcuts } from './useUndoShortcuts';
import UndoToolbar from './UndoToolbar';

const undoableTodos = withUndo(todoReducer, { limit: 100 });

export default function TodoList() {
  const [state, dispatch] = useReducer(undoableTodos, initialTodos, initUndo);
  const [text, setText] = useState('');
  const nextId = useRef(3);
  const todos = state.present;

  const onUndo = () => dispatch(undo());
  const onRedo = () => dispatch(redo());
  const onKeyDown = useUndoShortcuts({ undo: onUndo, redo: onRedo });

  const add = (e) => {
    e.preventDefault();
    const trimmed = text.trim();
    if (!trimmed) return;
    dispatch({ type: 'ADD', id: nextId.current++, text: trimmed });
    setText('');
  };

  return (
    <section className="panel" onKeyDown={onKeyDown} aria-labelledby="todo-h">
      <h2 id="todo-h">Todos (withUndo reducer)</h2>
      <UndoToolbar
        undo={onUndo}
        redo={onRedo}
        canUndo={state.past.length > 0}
        canRedo={state.future.length > 0}
      />
      <form onSubmit={add} className="add-todo">
        <input aria-label="New todo" value={text} onChange={(e) => setText(e.target.value)} />
        <button type="submit">Add</button>
      </form>
      <ul className="todos">
        {todos.map((t) => (
          <li key={t.id}>
            <label className={t.done ? 'done' : ''}>
              <input
                type="checkbox"
                checked={t.done}
                onChange={() => dispatch({ type: 'TOGGLE', id: t.id })}
              />
              {t.text}
            </label>
            <button
              type="button"
              aria-label={`Delete ${t.text}`}
              onClick={() => dispatch({ type: 'REMOVE', id: t.id })}
            >
              ✕
            </button>
          </li>
        ))}
      </ul>
    </section>
  );
}
```

**`App.jsx`**

```jsx
import SketchPad from './SketchPad';
import NotesEditor from './NotesEditor';
import TodoList from './TodoList';
import './styles.css';

export default function App() {
  return (
    <main className="app">
      <h1>Undo / Redo</h1>
      <p className="hint">
        <kbd>Ctrl</kbd>+<kbd>Z</kbd> undo · <kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>Z</kbd> /{' '}
        <kbd>Ctrl</kbd>+<kbd>Y</kbd> redo (focus a panel first)
      </p>
      <SketchPad />
      <NotesEditor />
      <TodoList />
    </main>
  );
}
```

**`styles.css`**

```css
* { box-sizing: border-box; }
body { margin: 0; font-family: system-ui, sans-serif; background: #f1f5f9; color: #0f172a; }
.app { max-width: 720px; margin: 0 auto; padding: 24px 16px; }
.hint { color: #475569; font-size: 14px; }
kbd { padding: 1px 6px; border: 1px solid #94a3b8; border-bottom-width: 2px; border-radius: 4px; background: #fff; font-size: 12px; }

.panel { margin: 16px 0; padding: 14px; background: #fff; border: 1px solid #cbd5e1; border-radius: 10px; }
.panel:focus-visible { outline: 3px solid #f59e0b; outline-offset: 2px; }
.panel h2 { margin: 0 0 10px; font-size: 18px; }

.toolbar { display: flex; flex-wrap: wrap; align-items: center; gap: 8px; margin-bottom: 10px; }
.toolbar button, .add-todo button { padding: 6px 14px; border: 1px solid #94a3b8; border-radius: 6px; background: #fff; cursor: pointer; }
.toolbar button:disabled { opacity: 0.45; cursor: not-allowed; }
button:focus-visible, input:focus-visible, textarea:focus-visible { outline: 3px solid #f59e0b; outline-offset: 2px; }
.meta { font-size: 13px; color: #64748b; }

.canvas { display: block; width: 100%; height: 240px; background: #f8fafc; border: 1px dashed #94a3b8; border-radius: 8px; touch-action: none; cursor: crosshair; }
.canvas polyline { fill: none; stroke: #2563eb; stroke-width: 3; stroke-linecap: round; stroke-linejoin: round; }

textarea { width: 100%; padding: 8px; font: inherit; border: 1px solid #94a3b8; border-radius: 8px; resize: vertical; }

.add-todo { display: flex; gap: 8px; margin-bottom: 8px; }
.add-todo input { flex: 1; padding: 6px 8px; border: 1px solid #94a3b8; border-radius: 6px; }
.todos { list-style: none; margin: 0; padding: 0; }
.todos li { display: flex; justify-content: space-between; align-items: center; padding: 6px 0; border-bottom: 1px solid #e2e8f0; }
.todos label.done { text-decoration: line-through; color: #64748b; }
.todos li button { border: 0; background: none; color: #dc2626; cursor: pointer; }
```

</details>

---

## 6. How It Works

- **Three stacks, one reducer.** `past` (oldest → newest), `present`, `future` (nearest redo first). Every transition is a pure function of the previous state, so the hook is a thin wrapper that adds timers and stable callbacks.
- **Snapshots are cheap** because updates are immutable: a new `present` shares unchanged sub-trees with older ones. Nothing is deep-cloned.
- **New action after undo clears `future`** — exactly like an editor; the old redo branch is unreachable.
- **`limit`**: `[...past, present].slice(-limit)` keeps only the newest entries; `canUndo` simply becomes `false` when the oldest has been dropped.
- **Functional updates** (`set(prev => …)`) are resolved *inside* the reducer against the latest `present`, so many `pointermove` dispatches in one tick never use a stale value.
- **Batching via `merging`.** First batched `set` pushes the pre-gesture value and sets `merging: true`; later batched sets only replace `present`. `COMMIT`, `UNDO`, `REDO`, `RESET` or a non-batched `set` closes the batch. Undoing in the middle of a batch rewinds the *whole* gesture to the state before it started.
- **Debounce = batch + timer.** `set(text, { debounce: 500 })` is a batched set that also schedules `COMMIT`. Each keystroke resets the timer, so a typing burst = one step, a pause = a step boundary. `undo/redo/reset/unmount` clear the timer so a stale commit never fires.
- **No-op protection.** `Object.is(next, present)` returns the *same* state object → no re-render, no history entry (important for `Clear` on an empty board or typing the same value).
- **`withUndo(reducer)`** is the same idea expressed as a higher-order reducer: it intercepts `UNDO/REDO/RESET`, delegates everything else to the inner reducer, and treats "inner reducer returned the same reference" as "nothing happened, don't record". `track(action)` lets you update `present` without creating a history entry (selection, hover…).
- **Shortcuts**: `e.ctrlKey || e.metaKey` covers Windows/Linux and macOS; `key.toLowerCase()` because Shift makes `e.key === 'Z'`. `preventDefault` stops the browser's native undo — required on the controlled textarea, otherwise native and custom history fight.
- **Pointer capture** on the SVG keeps `pointermove` flowing if the cursor leaves the canvas mid-stroke; `pointerup`/`pointercancel` both end the stroke.

---

## 7. Edge Cases

- Undo/redo with empty stacks → reducers return the same `state` (no re-render); buttons are disabled.
- `set` with the same value / `Clear` on an empty canvas → no entry.
- Typing, then immediately clicking Undo before the debounce fires → timer cleared; the whole partial burst is undone as one step.
- Undo while a stroke is in progress (keyboard during drag) → batch closes, stroke removed; further `pointermove`s must not crash — the guard `drawing.current` is cleared on `pointerup`, and the updater bails (`if (!last) return prev`) when the stroke list is empty.
- Missed `pointerup` (pointer released outside window) → capture + `pointercancel` + `lostpointercapture` (add if needed) prevent a stuck batch that would merge two strokes.
- History limit reached → oldest step silently dropped; document it (or show "history truncated").
- Undo after `reset` → nothing to undo by design.
- Identical-looking but new object (`set({...obj})`) → recorded; use `Object.is` or a custom `equals` if that's unwanted.
- Debounce + non-debounced `set` interleaved → the non-batched set closes the batch first (a new entry starts).
- StrictMode double-invoked reducers → fine because updaters and reducers are pure (IDs are generated in handlers, not reducers/updaters).
- Mac `Cmd+Z`, Windows `Ctrl+Z`, `Ctrl+Y` (Windows redo) and `Cmd+Shift+Z` (Mac redo) all supported.
- IME composition in the textarea (`isComposing`) → debounce commits fine, but a stricter editor should not undo mid-composition.

---

## 8. Accessibility

- Undo/Redo are real `<button>`s with `disabled` state and `aria-keyshortcuts`, so keyboard and screen-reader users have a non-shortcut path.
- Toolbar uses `role="toolbar"` with a label.
- Stroke/step counts live in `aria-live="polite"` so changes after undo are announced; the `<svg>` has `role="img"` with a text summary (a drawing is inherently visual — offer a text alternative or export).
- The panel is focusable (`tabIndex=0`) with a visible focus ring so keyboard users can reach the shortcut scope.
- Don't hijack Ctrl+Z when focus is in an unrelated input unless you replace its history (we scope shortcuts to the panel).
- Consider announcing what was undone ("Undid: Added 'Build useHistory'") — needs descriptions per history entry.

---

## 9. Performance Considerations

- **Memory** = `limit × size of snapshot`. Structural sharing makes todos/strokes cheap; for large documents store patches/commands, or cap `limit`.
- Batching matters: without it a single drag would push hundreds of entries and blow the limit in one gesture.
- `pointermove` fires up to once per frame; each batched `set` copies the current stroke's point array (O(points)). For long strokes keep points in a ref and commit once, or use `requestAnimationFrame` throttling / a mutable stroke while drawing.
- `past.slice(-limit)` and spread are O(limit) per commit — fine for ≤ a few hundred; a ring buffer or linked list is the optimisation.
- Callbacks are stable (`useCallback` with `[]`/`[limit]`) so memoized children don't re-render.
- Only expose derived booleans (`canUndo`) rather than the stacks to avoid consumers re-rendering on irrelevant changes.

---

## 10. Follow-up Questions

<details>
<summary><strong>1. Snapshots or commands — which would you pick and why?</strong></summary>

Snapshots for UI state of modest size: trivial, always correct, any state shape. Commands (or patches) when snapshots are large (canvas pixels, long documents), when actions have side effects that need compensating, or for **collaborative undo**, where you must undo *your* operation without undoing others'. Immer patches give most of the memory benefit with snapshot-like ergonomics.

</details>

<details>
<summary><strong>2. How do you implement a global undo stack across many components?</strong></summary>

Lift the history into a context/store (one `withUndo`-wrapped root reducer, or Zustand/Redux with a `history` slice). One `window` keydown listener dispatches `undo/redo`. If you need per-feature undo (editor vs. settings), keep separate stacks and route shortcuts to the focused feature.

</details>

<details>
<summary><strong>3. How do you exclude some changes (selection, zoom) from history?</strong></summary>

`withUndo`'s `track(action)` returns `false` for them, so they update `present` without pushing to `past`. Caveat: undo then restores the *old* selection too, since `present` is replaced wholesale — split state into `{ document, ui }` and put only `document` under history.

</details>

<details>
<summary><strong>4. How would you add "history list / jump to step"?</strong></summary>

Keep one array `states` and an index `cursor` instead of three stacks: `present = states[cursor]`; undo = `cursor - 1`; new `set` = `states.slice(0, cursor + 1).concat(next)`; jump = set `cursor`. Same asymptotics; the three-stack form is just a rotation of this.

</details>

<details>
<summary><strong>5. How would you persist history or sync it with the server?</strong></summary>

Persist `present` always; persist `past/future` only if they are small and serialisable (debounced write to localStorage/IndexedDB). With a server, send operations (commands) and treat undo as a *new* inverse operation appended to the log — rewriting shared history would break other clients.

</details>

<details>
<summary><strong>6. Why not just use the browser's native Ctrl+Z in the textarea?</strong></summary>

It works for uncontrolled inputs, but React-controlled values, programmatic `value` changes and cross-field state (e.g. a form with several inputs) break the native stack, and its granularity can't be controlled or shared with a toolbar button. Own the history and call `preventDefault`.

</details>

<details>
<summary><strong>7. How do you coalesce by <em>kind</em> of change (merge only typing, not formatting)?</strong></summary>

Replace the boolean `merging` with `mergeKey`: `set(v, { mergeKey: 'typing' })` merges only if `state.mergeKey === action.mergeKey`; a different key (or none) pushes a new entry. Time-based (debounce) and key-based coalescing can be combined, which is what real editors do.

</details>

<details>
<summary><strong>8. How do you test it?</strong></summary>

The reducer is pure: assert `set → undo → redo` round-trips, redo is cleared by `set`, the limit truncates the oldest, batched sets produce one entry, no-ops return the same reference. For the hook use `renderHook` with fake timers for debounce, and a component test firing `keyDown` with `{ ctrlKey: true, key: 'z' }` / `{ shiftKey: true }`.

</details>

---

## 11. Common Mistakes

- Mutating the present value (`push`, `splice`) → all snapshots share the mutation; undo "does nothing".
- Not clearing `future` on a new action → redo resurrects stale states.
- Pushing an entry on every `pointermove` / keystroke → 1 stroke = 300 undo steps, limit exhausted.
- Unbounded history → memory leak in long sessions.
- Storing `canUndo` / `canRedo` in state instead of deriving them.
- Recording no-op updates (same value) → "Undo" appears to do nothing.
- Computing the next value from a stale closure instead of an updater (`set(prev => …)`).
- Forgetting to clear the debounce timer on undo/unmount → stale commit closes the wrong batch / state update after unmount.
- Not calling `preventDefault` on Ctrl+Z → native and custom undo both run.
- Checking only `ctrlKey` (no `metaKey`) or comparing `e.key === 'z'` while Shift is held (`'Z'`).
- Putting side effects or ID generation (`Math.random`, `Date.now`) inside the reducer.
- Undoable and non-undoable state mixed together (undo reverts scroll position / selection).
