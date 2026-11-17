# Todo List — Machine Coding

> **Difficulty:** Easy | **Time:** 40–45 min | **Frequency:** ⭐⭐⭐⭐⭐

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

Build a Todo List app in React. A user can add a todo, mark it complete, edit its text, delete it, and filter the list by **All / Active / Completed**. Todos must **persist across page reloads** using `localStorage`.

---

## 2. Requirements

### Functional

- Add a todo via input + Enter / "Add" button (ignore empty / whitespace-only input).
- Toggle complete / incomplete.
- Edit text inline (double-click or Edit button; Enter = save, Esc = cancel).
- Delete a todo.
- Filter: All / Active / Completed.
- Show "N items left" and a "Clear completed" button.
- Persist to `localStorage`; restore on load.

### Non-functional

- Functional components + hooks only, plain CSS.
- No state mutation; stable unique keys (not array index).
- Survives corrupted / unavailable `localStorage`.
- Keyboard accessible; no unnecessary re-renders of untouched rows.

---

## 3. Clarifying Questions to Ask

- Should edit be inline or in a modal? (Assume inline.)
- Should empty text on edit delete the todo or revert? (Assume delete, like TodoMVC.)
- Do I need duplicate detection or max length? (Assume no, mention it.)
- Is "filter" persisted too? (Assume no — only todos.)
- Do I need drag-to-reorder, due dates, or multi-tab sync? (Assume out of scope; mention as follow-ups.)
- Any browser support constraints? (`crypto.randomUUID` needs a secure context.)

---

## 4. Approach & Component Design

**State shape** (single source of truth lives in `App`):

```js
todos:  [{ id: 'uuid', text: 'Buy milk', completed: false }]   // persisted
filter: 'all' | 'active' | 'completed'                          // UI-only
// per-item local UI state (inside TodoItem): isEditing, draft
```

**Derived, not stored:** `visibleTodos` (filter applied), `activeCount`, `hasCompleted`. Never keep these in state — they would go out of sync.

```
App  (todos, filter, handlers)
 ├── TodoForm        (draft input; calls onAdd)
 ├── <ul>
 │    └── TodoItem * N   (memo; local isEditing/draft; calls onToggle/onEdit/onDelete)
 └── footer: count · FilterBar · Clear completed
```

| Concern | Decision |
| --- | --- |
| Persistence | `useLocalStorage` hook (lazy init + effect write) |
| Updates | Functional `setTodos(prev => ...)` — no stale closures |
| Edit UI state | Local to `TodoItem` so typing doesn't re-render siblings |
| IDs | `crypto.randomUUID()` (or `Date.now()+counter` fallback) |

---

## 5. Reference Solution

<details>
<summary><strong>Show Full Code</strong></summary>

**`useLocalStorage.js`**

```jsx
import { useEffect, useState } from 'react';

export function useLocalStorage(key, initialValue, isValid = () => true) {
  const [value, setValue] = useState(() => {
    try {
      const raw = localStorage.getItem(key);
      if (raw === null) return initialValue;
      const parsed = JSON.parse(raw);
      return isValid(parsed) ? parsed : initialValue; // valid JSON, wrong shape
    } catch {
      return initialValue; // corrupted JSON or storage blocked
    }
  });

  useEffect(() => {
    try {
      localStorage.setItem(key, JSON.stringify(value));
    } catch {
      /* quota exceeded / private mode — keep working in memory */
    }
  }, [key, value]);

  return [value, setValue];
}
```

**`TodoForm.jsx`**

```jsx
import { useState } from 'react';

export default function TodoForm({ onAdd }) {
  const [text, setText] = useState('');

  const handleSubmit = (e) => {
    e.preventDefault();
    const trimmed = text.trim();
    if (!trimmed) return;
    onAdd(trimmed);
    setText('');
  };

  return (
    <form className="todo-form" onSubmit={handleSubmit}>
      <input
        value={text}
        onChange={(e) => setText(e.target.value)}
        placeholder="What needs to be done?"
        aria-label="New todo"
        autoFocus
      />
      <button type="submit">Add</button>
    </form>
  );
}
```

**`TodoItem.jsx`**

```jsx
import { memo, useEffect, useRef, useState } from 'react';

function TodoItem({ todo, onToggle, onEdit, onDelete }) {
  const [isEditing, setIsEditing] = useState(false);
  const [draft, setDraft] = useState(todo.text);
  const inputRef = useRef(null);
  const skipBlur = useRef(false);

  useEffect(() => {
    if (isEditing) {
      inputRef.current?.focus();
      inputRef.current?.select();
    }
  }, [isEditing]);

  const startEdit = () => {
    setDraft(todo.text);
    skipBlur.current = false;
    setIsEditing(true);
  };

  const commit = () => {
    skipBlur.current = true; // Enter unmounts the input too; don't commit twice
    setIsEditing(false);
    const trimmed = draft.trim();
    if (!trimmed) onDelete(todo.id);
    else if (trimmed !== todo.text) onEdit(todo.id, trimmed);
  };

  const cancel = () => {
    skipBlur.current = true; // the blur fired by unmounting must not commit
    setIsEditing(false);
  };

  const handleKeyDown = (e) => {
    if (e.key === 'Enter') commit();
    else if (e.key === 'Escape') cancel();
  };

  return (
    <li className={`todo-item ${todo.completed ? 'completed' : ''}`}>
      {isEditing ? (
        <input
          ref={inputRef}
          className="edit-input"
          value={draft}
          aria-label={`Edit "${todo.text}"`}
          onChange={(e) => setDraft(e.target.value)}
          onKeyDown={handleKeyDown}
          onBlur={() => {
            if (!skipBlur.current) commit();
          }}
        />
      ) : (
        <>
          <input
            type="checkbox"
            checked={todo.completed}
            aria-label={todo.text}
            onChange={() => onToggle(todo.id)}
          />
          <span className="todo-text" onDoubleClick={startEdit}>
            {todo.text}
          </span>
          <button onClick={startEdit} aria-label={`Edit "${todo.text}"`}>
            Edit
          </button>
          <button onClick={() => onDelete(todo.id)} aria-label={`Delete "${todo.text}"`}>
            ✕
          </button>
        </>
      )}
    </li>
  );
}

export default memo(TodoItem);
```

**`App.jsx`**

```jsx
import { useCallback, useState } from 'react';
import { useLocalStorage } from './useLocalStorage';
import TodoForm from './TodoForm';
import TodoItem from './TodoItem';
import './styles.css';

const FILTERS = {
  all: () => true,
  active: (t) => !t.completed,
  completed: (t) => t.completed,
};

export default function App() {
  const [todos, setTodos] = useLocalStorage('todos:v1', [], Array.isArray);
  const [filter, setFilter] = useState('all');

  // setTodos from useState is stable -> these callbacks keep identity -> memo works
  const addTodo = useCallback(
    (text) => {
      const id = crypto.randomUUID(); // outside the updater, which must stay pure
      setTodos((prev) => [...prev, { id, text, completed: false }]);
    },
    [setTodos]
  );
  const toggleTodo = useCallback(
    (id) =>
      setTodos((prev) => prev.map((t) => (t.id === id ? { ...t, completed: !t.completed } : t))),
    [setTodos]
  );
  const editTodo = useCallback(
    (id, text) => setTodos((prev) => prev.map((t) => (t.id === id ? { ...t, text } : t))),
    [setTodos]
  );
  const deleteTodo = useCallback(
    (id) => setTodos((prev) => prev.filter((t) => t.id !== id)),
    [setTodos]
  );
  const clearCompleted = () => setTodos((prev) => prev.filter((t) => !t.completed));

  const visibleTodos = todos.filter(FILTERS[filter]);
  const activeCount = todos.filter(FILTERS.active).length;
  const hasCompleted = todos.length > activeCount;

  return (
    <main className="app">
      <h1>Todos</h1>
      <TodoForm onAdd={addTodo} />

      {visibleTodos.length === 0 ? (
        <p className="empty">{todos.length === 0 ? 'Nothing to do 🎉' : `No ${filter} todos`}</p>
      ) : (
        <ul className="todo-list">
          {visibleTodos.map((todo) => (
            <TodoItem
              key={todo.id}
              todo={todo}
              onToggle={toggleTodo}
              onEdit={editTodo}
              onDelete={deleteTodo}
            />
          ))}
        </ul>
      )}

      {todos.length > 0 && (
        <footer className="footer">
          <span aria-live="polite">
            {activeCount} item{activeCount === 1 ? '' : 's'} left
          </span>
          <div role="group" aria-label="Filter todos">
            {Object.keys(FILTERS).map((name) => (
              <button
                key={name}
                className={filter === name ? 'selected' : ''}
                aria-pressed={filter === name}
                onClick={() => setFilter(name)}
              >
                {name[0].toUpperCase() + name.slice(1)}
              </button>
            ))}
          </div>
          <button onClick={clearCompleted} disabled={!hasCompleted}>
            Clear completed
          </button>
        </footer>
      )}
    </main>
  );
}
```

**`styles.css`**

```css
.app { max-width: 480px; margin: 40px auto; font-family: system-ui, sans-serif; }
.todo-form { display: flex; gap: 8px; }
.todo-form input { flex: 1; padding: 8px 10px; }
.todo-list { list-style: none; padding: 0; margin: 16px 0; }
.todo-item { display: flex; align-items: center; gap: 8px; padding: 8px 0; border-bottom: 1px solid #eee; }
.todo-text { flex: 1; overflow-wrap: anywhere; }
.todo-item.completed .todo-text { text-decoration: line-through; color: #888; }
.edit-input { flex: 1; padding: 6px; }
.empty { color: #888; text-align: center; }
.footer { display: flex; justify-content: space-between; align-items: center; gap: 8px; font-size: 14px; }
.footer button.selected { font-weight: 700; outline: 2px solid #4f46e5; }
button:focus-visible, input:focus-visible { outline: 2px solid #4f46e5; outline-offset: 2px; }
```

</details>

---

## 6. How It Works

- **Lazy initial state** — `useState(() => read())` reads `localStorage` once, not on every render.
- **Write in an effect** — keeps render pure; runs whenever `todos` changes.
- **Functional updates** (`setTodos(prev => ...)`) make handlers independent of the current `todos` value, so `useCallback` with `[setTodos]` is always correct and `memo(TodoItem)` stays effective.
- **Immutable updates** — `map` + spread for edit/toggle, `filter` for delete, spread for add.
- **Derived data** — filtering/counting computed during render; nothing to keep in sync.
- **Local edit state** — `isEditing`/`draft` live in the row, so typing re-renders one row only.
- **`skipBlur` ref** — Enter/Esc unmount the input, which can fire `blur` → would commit twice (Enter) or commit the cancelled draft (Esc). The ref prevents that.
- **Keys** — `todo.id`, so toggling/deleting never remounts the wrong row or loses edit state.

---

## 7. Edge Cases

- Empty / whitespace-only add → ignored; edit to empty → deleted.
- Corrupted JSON, or valid JSON of the wrong shape (e.g. `{}`), in `localStorage` → `try/catch` + `isValid` fall back to `[]`.
- `localStorage` throws (Safari private mode, quota) → app still works in memory.
- Filtering to a state with no matches → show a contextual empty message.
- Deleting a todo while another is in edit mode → untouched thanks to stable keys.
- Same text twice → allowed (IDs differ); duplicate checking is a follow-up.
- Very long text → CSS `overflow-wrap: anywhere` on `.todo-text`.
- Schema changes later → versioned key (`todos:v1`).

---

## 8. Accessibility

- Real `<form>`, `<button>`, `<input type="checkbox">` — keyboard works for free; the checkbox is named by `aria-label={todo.text}`.
- `aria-label` on icon-only buttons including the todo text ("Delete "Buy milk"").
- Filter buttons use `aria-pressed` inside a `role="group"` with a label.
- Item count in `aria-live="polite"` so screen readers hear changes.
- Visible `:focus-visible` outlines; focus moves into the edit input on edit. (When editing ends the input unmounts and focus drops to `<body>` — restoring focus to the row's Edit button is a good extra.)
- Don't convey completion with colour alone (strike-through + checkbox state).

---

## 9. Performance Considerations

- `memo(TodoItem)` + stable callbacks → toggling one todo re-renders only that row (verify with React DevTools Profiler).
- `localStorage` writes are synchronous and serialise the whole list — fine for hundreds of items; debounce or use IndexedDB for large lists.
- `filter`/count are O(n) per render — negligible; `useMemo` only if profiling shows a need.
- Avoid inline object/array props on `TodoItem`.
- Virtualise (react-window) only for thousands of rows.

---

## 10. Follow-up Questions

<details>
<summary><strong>1. How would you sync todos across browser tabs?</strong></summary>

Listen to the `storage` event (fires in *other* tabs) and call `setValue(JSON.parse(e.newValue))` when `e.key === key`. Or use `BroadcastChannel`.

</details>

<details>
<summary><strong>2. Convert state to <code>useReducer</code>. Why/when?</strong></summary>

Actions `ADD | TOGGLE | EDIT | DELETE | CLEAR_COMPLETED` centralise transitions, are easy to unit test, and `dispatch` is stable (no `useCallback`). Worth it once there are 4+ related transitions.

</details>

<details>
<summary><strong>3. Add undo/redo.</strong></summary>

Keep `{ past: [], present, future: [] }`. Each change pushes `present` to `past` and clears `future`. Undo pops from `past`; redo pops from `future`. Works naturally with immutable updates.

</details>

<details>
<summary><strong>4. Add drag-and-drop reorder.</strong></summary>

HTML5 DnD or `dnd-kit`; on drop, compute new array by removing the item at `from` and inserting at `to` (immutably). Keep order in the array itself so persistence is free. Provide keyboard alternative (Move up/down buttons).

</details>

<details>
<summary><strong>5. Todos come from an API instead. What changes?</strong></summary>

Add `loading/error` states, optimistic updates with rollback on failure, temp client IDs replaced by server IDs, and a data-fetching library (React Query) for cache/invalidations. `localStorage` becomes an offline cache.

</details>

<details>
<summary><strong>6. Why <code>crypto.randomUUID()</code> rather than <code>Date.now()</code> or array index?</strong></summary>

Index changes on delete/reorder → wrong reconciliation. `Date.now()` can collide for two adds in the same ms. UUID is unique; needs secure context, so fallback to a counter ref for older environments.

</details>

<details>
<summary><strong>7. How do you test it?</strong></summary>

React Testing Library: type + Enter adds item; click checkbox toggles; filter hides items; reload (re-render with seeded `localStorage`) restores; corrupted JSON doesn't crash. Query by role/label, not class names.

</details>

---

## 11. Common Mistakes

- Using array index as `key` or generating `key={Math.random()}`.
- Mutating state (`todos.push`, `todo.completed = !todo.completed`).
- Storing `filteredTodos` / `count` in state.
- `setTodos([...todos, x])` inside memoised callbacks → stale closure.
- Writing to `localStorage` in render, or reading it on every render.
- No `try/catch` around `JSON.parse`.
- Forgetting `preventDefault()` on form submit.
- Committing on blur *and* Esc, causing cancelled edits to save.
- Not trimming input; allowing empty todos.

