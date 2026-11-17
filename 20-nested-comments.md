# Nested Comments (Reddit-style) — Machine Coding

> **Difficulty:** Hard · **Time:** 60 min · **Asked at / Frequency:** Commonly reported at product companies (Atlassian, Uber, Flipkart, Meesho) — Very High · **Tier:** 1

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
12. [Interview Tips & Time Plan](#12-interview-tips--time-plan)
13. [Practice Checklist](#13-practice-checklist)

---

## 1. Problem Statement

Build a Reddit-style comment thread with **unlimited nesting**. Users can:

- Add a top-level comment
- **Reply** to any comment (inline form)
- **Edit** and **delete** their own comments
- **Collapse / expand** a thread

The data is a tree; every update must be **immutable**.

---

## 2. Requirements

### Functional

- Render a recursive tree of comments (author, text, relative time).
- Reply opens an inline form under that comment; submit appends a child.
- Edit (only own comments) swaps text for a textarea; shows `(edited)`.
- Delete: leaf → removed; comment with replies → shown as `[deleted]` (replies preserved).
- Collapse hides the body and replies, shows "N replies hidden".
- Empty / whitespace-only comments are rejected.

### Non-functional

- Pure, immutable tree helpers (`updateNode`, `removeNode`) — unit-testable.
- Only the changed branch re-renders (structural sharing + `React.memo`).
- No prop drilling of handlers (context).
- Works for arbitrarily deep trees.

---

## 3. Clarifying Questions to Ask

- Is the data **nested** (`children: []`) or **flat** with `parentId`? *(Assume nested; mention normalisation.)*
- On delete with replies — remove the whole subtree or soft-delete (`[deleted]`)?
- Who can edit/delete? (current user only)
- Do we need votes, sorting, "load more replies", pagination?
- Max depth for nesting / mobile layout?
- Persist to a backend or local state only?

---

## 4. Approach & Component Design

**Data shape (nested tree)**

```js
{ id, author, text, createdAt, editedAt, deleted, children: [ ...same shape ] }
```

**State placement**

| State | Where | Why |
|---|---|---|
| comment tree | `useReducer` at the root | single source of truth, pure transitions |
| `collapsed`, `mode` (`'reply' \| 'edit' \| null`) | local to each `CommentNode` | purely UI; no one else needs it |
| form text | local to `CommentForm` | resets when the form unmounts |

```
CommentsApp  (useReducer(tree), Context: { currentUser, add, edit, remove })
 ├─ CommentForm                 (new top-level comment)
 └─ <ul>
     └─ CommentNode (memo)      ◄───────────────┐ recursion
         ├─ header  [–] author · time (edited)  │
         ├─ body | CommentForm (edit mode)      │
         ├─ actions  Reply · Edit · Delete      │
         ├─ CommentForm (reply mode)            │
         └─ <ul> children.map → CommentNode ────┘
```

**Immutable update rule:** copy every node *on the path* from root to the target; reuse (same reference) everything else.

---

## 5. Reference Solution

<details>
<summary><strong>Show Full Code (treeUtils.js, commentsReducer.js, CommentsContext.jsx, CommentForm.jsx, CommentNode.jsx, App.jsx, styles.css)</strong></summary>

```jsx
// treeUtils.js — pure helpers with structural sharing
export function updateNode(nodes, id, updater) {
  let changed = false;
  const next = nodes.map((n) => {
    if (n.id === id) {
      changed = true;
      return updater(n);
    }
    const kids = updateNode(n.children, id, updater);
    if (kids !== n.children) {
      changed = true;
      return { ...n, children: kids };
    }
    return n;
  });
  return changed ? next : nodes;
}

// Leaf -> removed. Node with replies -> soft-deleted. A soft-deleted node
// whose last reply is removed is pruned as well.
export function removeNode(nodes, id) {
  let changed = false;
  const next = [];
  for (const n of nodes) {
    if (n.id === id) {
      changed = true;
      if (n.children.length > 0) next.push({ ...n, deleted: true });
      continue;
    }
    const kids = removeNode(n.children, id);
    if (kids !== n.children) {
      changed = true;
      if (n.deleted && kids.length === 0) continue;
      next.push({ ...n, children: kids });
    } else {
      next.push(n);
    }
  }
  return changed ? next : nodes;
}

export const countAll = (nodes) =>
  nodes.reduce((sum, n) => sum + 1 + countAll(n.children), 0);

export function timeAgo(ts) {
  const s = Math.max(0, Math.floor((Date.now() - ts) / 1000));
  if (s < 60) return 'just now';
  const m = Math.floor(s / 60);
  if (m < 60) return `${m}m ago`;
  const h = Math.floor(m / 60);
  if (h < 24) return `${h}h ago`;
  return `${Math.floor(h / 24)}d ago`;
}
```

```jsx
// commentsReducer.js
import { updateNode, removeNode } from './treeUtils';

export function commentsReducer(tree, action) {
  switch (action.type) {
    case 'add': {
      const node = {
        id: action.id,
        author: action.author,
        text: action.text,
        createdAt: action.createdAt,
        editedAt: null,
        deleted: false,
        children: [],
      };
      return action.parentId === null
        ? [node, ...tree]
        : updateNode(tree, action.parentId, (n) => ({
            ...n,
            children: [...n.children, node],
          }));
    }
    case 'edit':
      return updateNode(tree, action.id, (n) => ({
        ...n,
        text: action.text,
        editedAt: action.editedAt,
      }));
    case 'delete':
      return removeNode(tree, action.id);
    default:
      return tree;
  }
}
```

```jsx
// CommentsContext.jsx
import { createContext, useContext } from 'react';

export const CommentsContext = createContext(null);

export function useComments() {
  const ctx = useContext(CommentsContext);
  if (!ctx) throw new Error('useComments must be used inside <CommentsContext.Provider>');
  return ctx;
}
```

```jsx
// CommentForm.jsx
import { useState } from 'react';

export default function CommentForm({
  initialText = '',
  submitLabel = 'Comment',
  placeholder = 'Write a comment…',
  autoFocus = false,
  onSubmit,
  onCancel,
}) {
  const [text, setText] = useState(initialText);
  const trimmed = text.trim();
  const canSubmit = trimmed.length > 0 && trimmed !== initialText.trim();

  const submit = (e) => {
    e.preventDefault();
    if (!canSubmit) return;
    onSubmit(trimmed);
    setText('');
  };

  const onKeyDown = (e) => {
    if (e.key === 'Escape' && onCancel) onCancel();
    if ((e.ctrlKey || e.metaKey) && e.key === 'Enter') submit(e);
  };

  return (
    <form className="comment-form" onSubmit={submit}>
      <textarea
        aria-label={placeholder}
        placeholder={placeholder}
        rows={3}
        value={text}
        autoFocus={autoFocus}
        onChange={(e) => setText(e.target.value)}
        onKeyDown={onKeyDown}
      />
      <div className="form-actions">
        <button type="submit" disabled={!canSubmit}>{submitLabel}</button>
        {onCancel && (
          <button type="button" className="link" onClick={onCancel}>Cancel</button>
        )}
      </div>
    </form>
  );
}
```

```jsx
// CommentNode.jsx
import { memo, useState } from 'react';
import { useComments } from './CommentsContext';
import CommentForm from './CommentForm';
import { countAll, timeAgo } from './treeUtils';

const CommentNode = memo(function CommentNode({ comment }) {
  const { currentUser, add, edit, remove } = useComments();
  const [collapsed, setCollapsed] = useState(false);
  const [mode, setMode] = useState(null); // 'reply' | 'edit' | null

  const { id, author, text, createdAt, editedAt, deleted, children } = comment;
  const isOwner = author === currentUser;

  const handleReply = (value) => {
    add(id, value);
    setMode(null);
  };
  const handleEdit = (value) => {
    edit(id, value);
    setMode(null);
  };
  const handleDelete = () => {
    if (window.confirm('Delete this comment?')) remove(id);
  };

  const hiddenCount = collapsed ? countAll(children) : 0;

  return (
    <li className="comment">
      <div className="comment-header">
        <button
          className="toggle"
          aria-expanded={!collapsed}
          aria-label={collapsed ? 'Expand thread' : 'Collapse thread'}
          onClick={() => setCollapsed((c) => !c)}
        >
          {collapsed ? '[+]' : '[–]'}
        </button>
        <strong>{deleted ? '[deleted]' : author}</strong>
        <span className="meta">
          {timeAgo(createdAt)}
          {editedAt && !deleted && ' (edited)'}
        </span>
        {collapsed && hiddenCount > 0 && (
          <span className="meta">
            · {hiddenCount} {hiddenCount === 1 ? 'reply' : 'replies'} hidden
          </span>
        )}
      </div>

      {!collapsed && (
        <div className="comment-body">
          {mode === 'edit' ? (
            <CommentForm
              initialText={text}
              submitLabel="Save"
              autoFocus
              onSubmit={handleEdit}
              onCancel={() => setMode(null)}
            />
          ) : (
            <p className={deleted ? 'deleted' : ''}>{deleted ? '[deleted]' : text}</p>
          )}

          {!deleted && mode !== 'edit' && (
            <div className="actions">
              <button className="link" onClick={() => setMode(mode === 'reply' ? null : 'reply')}>
                Reply
              </button>
              {isOwner && (
                <>
                  <button className="link" onClick={() => setMode('edit')}>Edit</button>
                  <button className="link danger" onClick={handleDelete}>Delete</button>
                </>
              )}
            </div>
          )}

          {mode === 'reply' && (
            <CommentForm
              submitLabel="Reply"
              placeholder={`Reply to ${author}…`}
              autoFocus
              onSubmit={handleReply}
              onCancel={() => setMode(null)}
            />
          )}

          {children.length > 0 && (
            <ul className="replies">
              {children.map((child) => (
                <CommentNode key={child.id} comment={child} />
              ))}
            </ul>
          )}
        </div>
      )}
    </li>
  );
});

export default CommentNode;
```

```jsx
// App.jsx
import { useMemo, useReducer } from 'react';
import { commentsReducer } from './commentsReducer';
import { CommentsContext } from './CommentsContext';
import CommentForm from './CommentForm';
import CommentNode from './CommentNode';
import './styles.css';

const CURRENT_USER = 'you';
const now = Date.now();
const HOUR = 3600 * 1000;

const SEED = [
  {
    id: 'c1', author: 'alice', text: 'What is everyone using for state management?',
    createdAt: now - 5 * HOUR, editedAt: null, deleted: false,
    children: [
      {
        id: 'c2', author: 'you', text: 'Context + useReducer is enough for most apps.',
        createdAt: now - 4 * HOUR, editedAt: null, deleted: false,
        children: [
          {
            id: 'c3', author: 'bob', text: 'Until you hit re-render problems.',
            createdAt: now - 3 * HOUR, editedAt: null, deleted: false, children: [],
          },
        ],
      },
      {
        id: 'c4', author: 'carol', text: 'Zustand, hands down.',
        createdAt: now - 2 * HOUR, editedAt: null, deleted: false, children: [],
      },
    ],
  },
  {
    id: 'c5', author: 'dave', text: 'Nice thread!',
    createdAt: now - HOUR, editedAt: null, deleted: false, children: [],
  },
];

let seq = 100;
const nextId = () => `c${++seq}`;

export default function App() {
  const [tree, dispatch] = useReducer(commentsReducer, SEED);

  // dispatch is stable, so this value never changes and memoised nodes stay put
  const api = useMemo(
    () => ({
      currentUser: CURRENT_USER,
      add: (parentId, text) =>
        dispatch({
          type: 'add', parentId, text,
          id: nextId(), author: CURRENT_USER, createdAt: Date.now(),
        }),
      edit: (id, text) => dispatch({ type: 'edit', id, text, editedAt: Date.now() }),
      remove: (id) => dispatch({ type: 'delete', id }),
    }),
    []
  );

  return (
    <CommentsContext.Provider value={api}>
      <main className="thread">
        <h1>Discussion</h1>
        <CommentForm
          submitLabel="Comment"
          onSubmit={(text) => api.add(null, text)}
        />
        <ul className="comments">
          {tree.map((c) => (
            <CommentNode key={c.id} comment={c} />
          ))}
        </ul>
      </main>
    </CommentsContext.Provider>
  );
}
```

```css
/* styles.css */
.thread { max-width: 680px; margin: 24px auto; font-family: system-ui, sans-serif; }
ul.comments, ul.replies { list-style: none; margin: 0; padding: 0; }
ul.replies { margin-left: 10px; padding-left: 14px; border-left: 2px solid #e5e7eb; }
.comment { margin-top: 14px; }
.comment-header { display: flex; align-items: center; gap: 8px; font-size: 14px; }
.meta { color: #6b7280; font-size: 12px; }
.comment-body p { margin: 4px 0 6px; white-space: pre-wrap; word-break: break-word; }
.comment-body p.deleted { color: #9ca3af; font-style: italic; }
.toggle { border: 0; background: none; cursor: pointer; color: #6b7280; font-family: monospace; padding: 0; }
.actions { display: flex; gap: 12px; }
button.link { border: 0; background: none; padding: 0; color: #2563eb; cursor: pointer; font-size: 13px; }
button.link.danger { color: #dc2626; }
button:focus-visible, textarea:focus-visible { outline: 2px solid #2563eb; outline-offset: 2px; }
.comment-form { margin: 8px 0; }
.comment-form textarea { width: 100%; box-sizing: border-box; padding: 8px; border: 1px solid #ccc; border-radius: 6px; font: inherit; resize: vertical; }
.form-actions { display: flex; gap: 12px; align-items: center; margin-top: 6px; }
.form-actions button[type='submit'] { padding: 6px 14px; background: #2563eb; color: #fff; border: 0; border-radius: 6px; cursor: pointer; }
.form-actions button[type='submit']:disabled { opacity: .5; cursor: not-allowed; }
```

</details>

---

## 6. How It Works

- **Recursive component:** `CommentNode` renders its own `children.map(c => <CommentNode />)`. Recursion ends naturally when `children` is empty — no depth limit needed.
- **Immutable path copy:** `updateNode` walks the tree; for the target it returns `updater(n)`, for ancestors `{...n, children: newKids}`, and for everything else it returns the *same reference*. If nothing changed it returns the *original array* (so React bails out).
- **Structural sharing + `memo`:** unchanged siblings keep identical `comment` props, so `React.memo` skips them. Editing a deep reply re-renders only its ancestors path.
- **Reducer + context:** all mutations are pure state transitions; ids and timestamps are generated in the **action creator** (not the reducer) so the reducer stays pure/deterministic (matters in StrictMode). The context value is memoised and stable → consumers don't re-render because of it.
- **Local UI state:** `collapsed` / `mode` live in each node. Because nodes are keyed by `id`, state survives siblings being added/removed.
- **Delete semantics:** leaf → removed; has replies → `deleted: true` keeps conversation context; a soft-deleted node left with no replies is pruned.
- **Keys:** always `comment.id` — never the index (state would jump between comments when one is removed).

---

## 7. Edge Cases

- Reply/comment text that is empty or whitespace → submit disabled.
- Edit with unchanged text → Save disabled.
- Delete a comment that has replies → `[deleted]` placeholder, replies kept; no Reply/Edit actions on it.
- Delete the last reply of a soft-deleted parent → parent pruned too (cascade up).
- Deleting while its reply form is open — the node unmounts, local state disappears.
- Collapse a thread with an open reply form → the form unmounts (typed text is lost) but `mode` is kept, so it reappears empty on expand. Acceptable; mention lifting draft text if asked.
- Very deep threads → indentation eats the viewport; cap visual indentation after ~5–6 levels and show "Continue this thread".
- Rapid double submit — form clears after first submit; second is blocked by `canSubmit`.
- Multi-line text must preserve newlines (`white-space: pre-wrap`); never use `dangerouslySetInnerHTML` (XSS).

---

## 8. Accessibility

- Collapse button has `aria-expanded` and a descriptive `aria-label`.
- Nested `ul/li` gives screen readers a list hierarchy ("list, 3 items, nested"). For tree semantics use `role="tree"/"treeitem"` with arrow-key navigation (follow-up).
- All actions are real `<button>`s; the reply/edit textarea is auto-focused when opened; `Esc` cancels, `Ctrl/Cmd+Enter` submits.
- Return focus to the Reply/Edit button after closing the form (improvement worth mentioning).
- Textareas have `aria-label`; don't rely on colour for "danger" — the text says "Delete".
- Announce new replies politely (`aria-live="polite"` region with "Reply added").

---

## 9. Performance Considerations

- `React.memo` + structural sharing → O(depth) component re-renders per update instead of O(n).
- Stable context value and stable `dispatch`; no inline handlers passed to memoised children (they read from context).
- `countAll` runs only for collapsed nodes; for huge trees precompute counts or store `replyCount`.
- Large threads (10k+): lazy "Load more replies" per node, or **virtualise** a flattened list (flatten tree → rows with `depth`; collapse is just filtering).
- Normalised state (`byId` + `childIds`) makes updates O(1) and avoids path-copying; trade-off: need a `buildTree` step for rendering.

---

## 10. Follow-up Questions

<details>
<summary><strong>1. How would you normalise the data instead of nesting?</strong></summary>

`{ byId: { c1: {…, childIds: ['c2','c4']} }, rootIds: ['c1','c5'] }`. Edit = replace one entry; add = insert entry + push id to the parent's `childIds`; delete = remove id (and descendants). Each node component selects its own entry by id, so only that node re-renders. API responses are often flat (`parentId`) — build the maps in one O(n) pass.

</details>

<details>
<summary><strong>2. How do you convert a flat list with `parentId` into a tree?</strong></summary>

Create a map `id → {...node, children: []}`, then loop once: if `parentId` is null push to roots, otherwise push to `map[parentId].children`. O(n). Guard against orphans (parent not found) by treating them as roots.

</details>

<details>
<summary><strong>3. Add upvote/downvote and sorting (best/new/top).</strong></summary>

Add `votes` + a `vote` action using `updateNode`. Sorting is a derived view: sort each `children` array at render (or `useMemo` per node) with a comparator — never mutate state. Re-sorting on every vote makes the UI jump; usually freeze order until refresh.

</details>

<details>
<summary><strong>4. How to implement optimistic updates with a backend?</strong></summary>

Dispatch the add with a temp client id and `pending: true`; on success replace the id with the server's; on failure remove (or mark `failed` with Retry). Keep the reducer pure and handle the async in an action creator/custom hook.

</details>

<details>
<summary><strong>5. How would you load replies lazily ("Load 12 more replies")?</strong></summary>

Store `replyCount` and `childrenLoaded` on the node. Show the button when `children.length < replyCount`; on click fetch and merge via `updateNode`. Cursor pagination per parent.

</details>

<details>
<summary><strong>6. Why generate the id/timestamp outside the reducer?</strong></summary>

Reducers must be pure: StrictMode (and React generally) may call them twice and expect identical output. `Date.now()` / random ids inside would create two different results and break replays/time-travel debugging.

</details>

<details>
<summary><strong>7. How do you handle deep nesting visually on mobile?</strong></summary>

Cap indentation (e.g. after depth 5 stop indenting and render a "Continue thread →" link that re-roots the view at that comment), and reduce padding per level.

</details>

<details>
<summary><strong>8. How would you virtualise a huge thread?</strong></summary>

Flatten the visible tree via DFS into `[{comment, depth}]` (skipping collapsed subtrees), render with a windowing library (react-window), and indent by `depth`. Collapsed state must then be lifted into a `Set` of ids at the root.

</details>

---

## 11. Common Mistakes

- **Mutating** the tree (`node.children.push(reply)`) then `setState(tree)` — same reference, no re-render (or subtle bugs).
- Deep-cloning the whole tree (`JSON.parse(JSON.stringify())`) on each update — slow, breaks memoisation, loses Dates.
- Using array index as `key`.
- Prop-drilling `onReply/onEdit/onDelete` through every level.
- Storing a form's text in the parent so every keystroke re-renders the whole tree.
- Calling `Date.now()` / random id generation inside the reducer.
- Deleting a node with replies by dropping the subtree without asking.
- Forgetting to reset/close the reply form after submit.
- Rendering user text with `dangerouslySetInnerHTML`.

---

## 12. Interview Tips & Time Plan

Draw the tree and say "I copy only the path to the node" before coding. Write `updateNode` first — then every feature is a one-liner using it.

| Minutes | Task |
|---|---|
| 0–5 | Clarify (nested vs flat, delete semantics), define node shape |
| 5–15 | Recursive `CommentNode` rendering seed data, indentation CSS |
| 15–25 | `updateNode` + reducer; add top-level comment and reply (inline form) |
| 25–35 | Edit + delete (leaf vs soft delete) |
| 35–42 | Collapse/expand with hidden-reply count |
| 42–50 | Context to avoid prop drilling, `React.memo`, structural sharing |
| 50–60 | Edge cases, a11y, discuss normalisation / virtualisation |

---

## 13. Practice Checklist

- [ ] Recursive component renders arbitrary depth
- [ ] `updateNode` returns new objects only along the path
- [ ] Add top-level comment and nested reply
- [ ] Edit own comment, shows `(edited)`
- [ ] Delete leaf vs delete with replies (soft delete + prune)
- [ ] Collapse shows hidden reply count
- [ ] Empty comments rejected; form closes after submit
- [ ] `key={id}`, context instead of prop drilling, `React.memo`
- [ ] Explain normalised alternative and flat → tree conversion
- [ ] Re-do from scratch in 45 minutes
