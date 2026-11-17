# File Explorer (Folder Tree) — Machine Coding

> **Difficulty:** Hard | **Time:** 60 min | **Frequency:** ⭐⭐⭐⭐⭐ (very common in product-company UI rounds)

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

Build a VS Code-style **file explorer** from a nested JSON tree. Users can expand/collapse folders, **add** a file/folder inside any folder, **rename** and **delete** any node. All tree operations must be **immutable**.

---

## 2. Requirements

### Functional

- Render folders and files recursively with indentation and icons.
- Click a folder to expand/collapse; click a file to select it.
- Add file / add folder to a folder (inline name input; the folder auto-expands).
- Rename inline (Enter confirms, Esc cancels).
- Delete a node (folder deletes its whole subtree).
- Validate names: not empty, no `/` or `\`, **no duplicate** among siblings (case-insensitive).
- Folders listed before files, alphabetical.

### Non-functional

- Immutable, pure tree helpers (testable in isolation).
- Arbitrary depth; expand state independent of the tree data.
- Keyboard accessible (Enter/Space, arrows, F2, Delete).

---

## 3. Clarifying Questions to Ask

- Is the data nested (`children`) or flat? Do nodes have unique ids?
- Can the root be renamed/deleted? *(Assume no.)*
- Duplicate names allowed? Case-sensitive?
- Delete confirmation? Undo?
- Drag and drop / move nodes? Search/filter? Lazy-load children?
- Should expansion state persist (localStorage)?

---

## 4. Approach & Component Design

**Node shape**

```js
{ id: 'n7', name: 'src', type: 'folder', children: [ ... ] }
{ id: 'n9', name: 'index.js', type: 'file' }
```

**State** (kept in a custom hook `useFileTree`)

| State | Type | Why separate |
|---|---|---|
| `tree` | root node (`useReducer`) | data — changes on add / rename / delete |
| `expanded` | `Set<id>` | UI state; survives rename, and lets us expand a folder when adding into it |
| `selectedId` | `id \| null` | UI state |
| `editing` | `null \| {mode:'rename', id} \| {mode:'create', parentId, type}` | only one inline input at a time |

```
App ── FileTreeContext.Provider(value = useFileTree(initialTree))
 └─ <ul role="tree">
     └─ TreeNode (recursive)  ◄──────────────────────┐
         ├─ row: icon · name | NameInput · [＋file ＋folder ✎ 🗑]
         └─ (if open) <ul role="group">              │
              ├─ create-row: NameInput (if creating here)
              └─ children.map → TreeNode ────────────┘
```

**Immutable ops** — each copies only the path root → target and reuses all other references.

| Op | Implementation |
|---|---|
| add | `updateNode(root, parentId, n => ({...n, children: [...n.children, child]}))` |
| rename | `updateNode(root, id, n => ({...n, name}))` |
| delete | `removeNode(root, id)` — filter at the parent level, recurse elsewhere |

---

## 5. Reference Solution

<details>
<summary><strong>Show Full Code (treeUtils.js, useFileTree.js, FileTreeContext.js, NameInput.jsx, TreeNode.jsx, App.jsx, styles.css)</strong></summary>

```jsx
// treeUtils.js — pure & immutable
let seq = 0;

export const makeNode = (type, name) => ({
  id: `n${++seq}`,
  name,
  type,
  ...(type === 'folder' ? { children: [] } : {}),
});

export function updateNode(node, id, updater) {
  if (node.id === id) return updater(node);
  if (node.type !== 'folder') return node;

  let changed = false;
  const children = node.children.map((child) => {
    const next = updateNode(child, id, updater);
    if (next !== child) changed = true;
    return next;
  });
  return changed ? { ...node, children } : node;
}

export function removeNode(node, id) {
  if (node.type !== 'folder') return node;

  const remaining = node.children.filter((c) => c.id !== id);
  let changed = remaining.length !== node.children.length;
  const children = remaining.map((child) => {
    const next = removeNode(child, id);
    if (next !== child) changed = true;
    return next;
  });
  return changed ? { ...node, children } : node;
}

export const addChild = (root, parentId, child) =>
  updateNode(root, parentId, (n) =>
    n.type === 'folder' ? { ...n, children: [...n.children, child] } : n
  );

export function validateName(name, siblings, ignoreId = null) {
  if (!name) return 'Name cannot be empty';
  if (/[\\/]/.test(name)) return 'Name cannot contain / or \\';
  const lower = name.toLowerCase();
  if (siblings.some((s) => s.id !== ignoreId && s.name.toLowerCase() === lower)) {
    return 'A file or folder with this name already exists';
  }
  return '';
}

export const sortChildren = (children) =>
  [...children].sort((a, b) =>
    a.type !== b.type
      ? a.type === 'folder' ? -1 : 1
      : a.name.localeCompare(b.name, undefined, { numeric: true, sensitivity: 'base' })
  );
```

```jsx
// FileTreeContext.js
import { createContext } from 'react';

export const FileTreeContext = createContext(null);
```

```jsx
// useFileTree.js
import { useReducer, useState } from 'react';
import { addChild, makeNode, removeNode, updateNode } from './treeUtils';

function treeReducer(tree, action) {
  switch (action.type) {
    case 'add':
      return addChild(tree, action.parentId, action.node);
    case 'rename':
      return updateNode(tree, action.id, (n) => ({ ...n, name: action.name }));
    case 'remove':
      return removeNode(tree, action.id);
    default:
      return tree;
  }
}

export function useFileTree(initialTree) {
  const [tree, dispatch] = useReducer(treeReducer, initialTree);
  const [expanded, setExpanded] = useState(() => new Set([initialTree.id]));
  const [selectedId, setSelectedId] = useState(null);
  const [editing, setEditing] = useState(null);

  const toggle = (id) =>
    setExpanded((prev) => {
      const next = new Set(prev);
      if (next.has(id)) next.delete(id);
      else next.add(id);
      return next;
    });

  const startCreate = (parentId, type) => {
    setExpanded((prev) => (prev.has(parentId) ? prev : new Set(prev).add(parentId)));
    setEditing({ mode: 'create', parentId, type });
  };
  const startRename = (id) => setEditing({ mode: 'rename', id });
  const cancelEdit = () => setEditing(null);

  const commitCreate = (name) => {
    const node = makeNode(editing.type, name);
    dispatch({ type: 'add', parentId: editing.parentId, node });
    setSelectedId(node.id);
    setEditing(null);
  };
  const commitRename = (name) => {
    dispatch({ type: 'rename', id: editing.id, name });
    setEditing(null);
  };

  const remove = (id, name) => {
    if (!window.confirm(`Delete "${name}"?`)) return;
    dispatch({ type: 'remove', id });
    setSelectedId((s) => (s === id ? null : s));
    setEditing(null);
  };

  return {
    tree, expanded, selectedId, editing,
    toggle, select: setSelectedId,
    startCreate, startRename, cancelEdit, commitCreate, commitRename, remove,
  };
}
```

```jsx
// NameInput.jsx
import { useState } from 'react';
import { validateName } from './treeUtils';

export default function NameInput({ initial = '', siblings, ignoreId, onSubmit, onCancel }) {
  const [value, setValue] = useState(initial);
  const [error, setError] = useState('');

  const submit = () => {
    const name = value.trim();
    if (initial && name === initial) return onCancel(); // unchanged rename
    const err = validateName(name, siblings, ignoreId);
    if (err) return setError(err);
    onSubmit(name);
  };

  return (
    <span className="name-input">
      <input
        autoFocus
        value={value}
        aria-label="Name"
        aria-invalid={Boolean(error)}
        onFocus={(e) => e.target.select()}
        onChange={(e) => {
          setValue(e.target.value);
          setError('');
        }}
        onClick={(e) => e.stopPropagation()}
        onKeyDown={(e) => {
          if (e.key === 'Enter') submit();
          if (e.key === 'Escape') onCancel();
        }}
        onBlur={onCancel}
      />
      {error && <span className="name-error" role="alert">{error}</span>}
    </span>
  );
}
```

```jsx
// TreeNode.jsx
import { useContext } from 'react';
import { FileTreeContext } from './FileTreeContext';
import NameInput from './NameInput';
import { sortChildren } from './treeUtils';

export default function TreeNode({ node, siblings, depth }) {
  const {
    expanded, selectedId, editing,
    toggle, select, startCreate, startRename, cancelEdit,
    commitCreate, commitRename, remove,
  } = useContext(FileTreeContext);

  const isFolder = node.type === 'folder';
  const isOpen = isFolder && expanded.has(node.id);
  const isRoot = depth === 0;
  const isRenaming = editing?.mode === 'rename' && editing.id === node.id;
  const isCreatingHere = editing?.mode === 'create' && editing.parentId === node.id;

  const handleRowClick = () => {
    select(node.id);
    if (isFolder) toggle(node.id);
  };

  const handleKeyDown = (e) => {
    if (e.target !== e.currentTarget) return; // ignore bubbled events from inputs/buttons/children
    switch (e.key) {
      case 'Enter':
      case ' ':
        e.preventDefault();
        handleRowClick();
        break;
      case 'ArrowRight':
        if (isFolder && !isOpen) toggle(node.id);
        break;
      case 'ArrowLeft':
        if (isFolder && isOpen) toggle(node.id);
        break;
      case 'F2':
        if (!isRoot) startRename(node.id);
        break;
      case 'Delete':
        if (!isRoot) remove(node.id, node.name);
        break;
      default:
    }
  };

  const stop = (fn) => (e) => {
    e.stopPropagation();
    fn();
  };

  return (
    <li
      role="treeitem"
      aria-level={depth + 1}
      aria-expanded={isFolder ? isOpen : undefined}
      aria-selected={selectedId === node.id}
      tabIndex={0}
      onKeyDown={handleKeyDown}
    >
      <div
        className={`row${selectedId === node.id ? ' selected' : ''}`}
        style={{ paddingLeft: depth * 16 + 8 }}
        onClick={handleRowClick}
      >
        <span className="icon" aria-hidden="true">
          {isFolder ? (isOpen ? '📂' : '📁') : '📄'}
        </span>

        {isRenaming ? (
          <NameInput
            initial={node.name}
            siblings={siblings}
            ignoreId={node.id}
            onSubmit={commitRename}
            onCancel={cancelEdit}
          />
        ) : (
          <span className="name">{node.name}</span>
        )}

        {!isRenaming && (
          <span className="actions">
            {isFolder && (
              <>
                <button aria-label={`New file in ${node.name}`} title="New file"
                  onClick={stop(() => startCreate(node.id, 'file'))}>＋📄</button>
                <button aria-label={`New folder in ${node.name}`} title="New folder"
                  onClick={stop(() => startCreate(node.id, 'folder'))}>＋📁</button>
              </>
            )}
            {!isRoot && (
              <>
                <button aria-label={`Rename ${node.name}`} title="Rename (F2)"
                  onClick={stop(() => startRename(node.id))}>✎</button>
                <button aria-label={`Delete ${node.name}`} title="Delete (Del)"
                  onClick={stop(() => remove(node.id, node.name))}>🗑</button>
              </>
            )}
          </span>
        )}
      </div>

      {isOpen && (
        <ul role="group">
          {isCreatingHere && (
            <li role="none">
              <div className="row" style={{ paddingLeft: (depth + 1) * 16 + 8 }}>
                <span className="icon" aria-hidden="true">
                  {editing.type === 'folder' ? '📁' : '📄'}
                </span>
                <NameInput
                  siblings={node.children}
                  onSubmit={commitCreate}
                  onCancel={cancelEdit}
                />
              </div>
            </li>
          )}
          {sortChildren(node.children).map((child) => (
            <TreeNode
              key={child.id}
              node={child}
              siblings={node.children}
              depth={depth + 1}
            />
          ))}
        </ul>
      )}
    </li>
  );
}
```

```jsx
// App.jsx
import { FileTreeContext } from './FileTreeContext';
import { useFileTree } from './useFileTree';
import TreeNode from './TreeNode';
import './styles.css';

const INITIAL_TREE = {
  id: 'root',
  name: 'my-project',
  type: 'folder',
  children: [
    {
      id: 'src', name: 'src', type: 'folder',
      children: [
        {
          id: 'components', name: 'components', type: 'folder',
          children: [
            { id: 'button', name: 'Button.jsx', type: 'file' },
            { id: 'modal', name: 'Modal.jsx', type: 'file' },
          ],
        },
        { id: 'index', name: 'index.js', type: 'file' },
        { id: 'appjsx', name: 'App.jsx', type: 'file' },
      ],
    },
    { id: 'public', name: 'public', type: 'folder', children: [] },
    { id: 'pkg', name: 'package.json', type: 'file' },
    { id: 'readme', name: 'README.md', type: 'file' },
  ],
};

export default function App() {
  const fileTree = useFileTree(INITIAL_TREE);

  return (
    <FileTreeContext.Provider value={fileTree}>
      <main className="explorer">
        <h1>Explorer</h1>
        <ul role="tree" aria-label="Project files">
          <TreeNode node={fileTree.tree} siblings={[]} depth={0} />
        </ul>
      </main>
    </FileTreeContext.Provider>
  );
}
```

```css
/* styles.css */
.explorer { max-width: 420px; margin: 24px auto; font: 14px system-ui, sans-serif; }
.explorer ul { list-style: none; margin: 0; padding: 0; }
.explorer li { outline: none; }
.row { display: flex; align-items: center; gap: 6px; min-height: 28px; padding-right: 8px; cursor: pointer; user-select: none; }
.row:hover { background: #f1f5f9; }
.row.selected { background: #dbeafe; }
li:focus-visible > .row { outline: 2px solid #2563eb; outline-offset: -2px; }
.name { flex: 1; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.actions { display: none; gap: 2px; margin-left: auto; }
.row:hover .actions, .row:focus-within .actions, li:focus-visible > .row .actions { display: flex; }
.actions button { border: 0; background: none; cursor: pointer; padding: 2px 4px; border-radius: 4px; font-size: 12px; }
.actions button:hover { background: #cbd5e1; }
.name-input { position: relative; flex: 1; }
.name-input input { width: 100%; box-sizing: border-box; padding: 2px 4px; font: inherit; }
.name-input input[aria-invalid='true'] { border-color: #dc2626; outline-color: #dc2626; }
.name-error { position: absolute; top: 100%; left: 0; z-index: 1; background: #fef2f2; color: #b91c1c; padding: 2px 6px; font-size: 12px; border-radius: 4px; white-space: nowrap; }
```

</details>

---

## 6. How It Works

- **Recursion** — `TreeNode` renders its own children when it's a folder and is open. Closed folders render nothing below them, so cost scales with *visible* nodes.
- **Expansion is separate from data** (`Set<id>`): renaming/adding doesn't reset what the user opened, and "add inside a collapsed folder" just adds the id to the set.
- **Immutable ops with structural sharing** — `updateNode`/`removeNode` return the *same node reference* when nothing below changed, so only the path to the target is copied. `React.memo` on `TreeNode` could then skip untouched branches (with the single context above, see the caveat in section 9).
- **One `editing` object** guarantees a single inline input (create *or* rename). Blur/Esc cancels; Enter validates then commits.
- **Validation lives in a pure function** (`validateName(name, siblings, ignoreId)`); `ignoreId` lets a rename keep its own name (or change casing).
- **Ids** come from a counter (`makeNode`) called in the event handler, not in the reducer, so the reducer stays pure.
- **Sorting at render** (`sortChildren` copies before sorting) – never `.sort()` state in place.
- **Keyboard** — `li` is the focusable `treeitem`; `e.target !== e.currentTarget` ignores bubbled key events from the input, buttons and nested items.

---

## 7. Edge Cases

- Duplicate name among siblings (case-insensitive); rename to the same name (no-op).
- Empty / whitespace name; names containing `/` or `\`.
- Delete the selected node or a node whose descendant is selected (selection of a descendant becomes dangling but harmless; clear it by walking the subtree if you want it strict).
- Delete a folder that is currently being renamed/created into → `editing` cleared on delete.
- Add into a collapsed folder → auto-expand.
- Empty folder (render nothing, keep `aria-expanded`).
- Root: cannot rename/delete.
- Very deep nesting → indentation via `paddingLeft: depth * 16`, not nested margins.
- Delete confirmation for non-empty folders (count of descendants is a nice touch).

---

## 8. Accessibility

- Follow the **tree pattern**: `role="tree"` → `treeitem` (`aria-level`, `aria-expanded`, `aria-selected`) → `role="group"` for children.
- Keyboard: Enter/Space toggles/selects, →/← expand/collapse, F2 rename, Delete remove. Full WAI-ARIA tree navigation (↑/↓ moving between visible items with roving `tabindex`, Home/End) is the stretch goal.
- Icon buttons have `aria-label`s; emoji icons are `aria-hidden`.
- Inline input: `aria-label`, `aria-invalid`, error with `role="alert"`; Esc cancels.
- After commit/cancel, return focus to the owning `treeitem` (not implemented above — mention it).
- Don't rely on hover to reveal actions — they also show on `:focus-within`.

---

## 9. Performance Considerations

- Only expanded folders render children → DOM size ∝ visible nodes.
- Structural sharing + `React.memo(TreeNode)` limits re-renders to the changed path; note the context value changes on every state update, so to benefit from `memo` split context (state vs stable actions) or pass props/selectors.
- Each `updateNode` is O(n) worst-case (search). With a **normalised** map (`byId`, `childIds`) updates become O(1).
- Very large trees: **virtualise** the flattened visible list (DFS over expanded nodes → rows with `depth`) and lazy-load children on expand.
- `new Set(prev)` copies are O(expanded) — fine; mention immutable structures only if asked.

---

## 10. Follow-up Questions

<details>
<summary><strong>1. How would you normalise the tree for O(1) updates?</strong></summary>

`{ nodes: { id: {id, name, type, parentId, childIds} }, rootId }`. Rename = replace one entry. Add = insert entry + append to parent's `childIds`. Delete = collect descendant ids with DFS and remove them all. Rendering reads `nodes[id]` per component, so only that node re-renders.

</details>

<details>
<summary><strong>2. How would you implement drag & drop to move nodes?</strong></summary>

`removeNode` extracting the node + `addChild` into the target, in one reducer action (so it's atomic). Block dropping a folder into itself or its descendants (walk up parent chain / check subtree contains target) and handle duplicate names at the destination.

</details>

<details>
<summary><strong>3. How do you add search/filter?</strong></summary>

Pure function `filterTree(node, q)` that keeps a node if it matches or any descendant matches (returns a new pruned tree), and auto-expands the ancestors of matches (derive an `expanded` override from matching paths). Memoize on `[tree, q]`.

</details>

<details>
<summary><strong>4. Lazy-load children from an API?</strong></summary>

Add `hasChildren` + `loaded` flags. On first expand dispatch `loading` → fetch → `updateNode` to set `children`. Show a spinner row; handle errors with retry; cancel stale requests on collapse.

</details>

<details>
<summary><strong>5. Implement undo/redo for add/rename/delete.</strong></summary>

Because updates are immutable, keep `past`, `present`, `future` arrays of tree snapshots (cheap due to structural sharing). Undo pops from `past`; any new action clears `future`. See the history pattern in the Tic-Tac-Toe file.

</details>

<details>
<summary><strong>6. Why `Set` for expanded state, and why immutably copy it?</strong></summary>

O(1) membership checks. React compares by reference, so mutating the same `Set` wouldn't trigger a re-render; always `new Set(prev)`. Alternative: a plain object `{[id]: true}`.

</details>

<details>
<summary><strong>7. How do you compute a node's path (`src/components/Button.jsx`)?</strong></summary>

DFS returning the path array when the id is found, or keep `parentId` (normalised) and walk up. Useful for breadcrumbs and for duplicate checks across full paths.

</details>

<details>
<summary><strong>8. Persist the tree and expansion state?</strong></summary>

Serialize `tree` and `[...expanded]` to `localStorage` in a `useEffect` (debounced), and lazily initialise state from it. Wrap `JSON.parse` in try/catch and validate shape.

</details>

---

## 11. Common Mistakes

- Mutating the tree (`node.children.push`) and calling `setState` with the same object.
- Storing `isOpen` inside each node's data, so renames/refreshes reset it — or local `useState` per node so "add into folder" can't expand it.
- Using `index` or `name` as the key — use a stable `id`.
- Allowing duplicate names, empty names, or deleting the root.
- Sorting `children` in place.
- Generating ids with `Math.random()` inside the reducer.
- Forgetting `stopPropagation` on action buttons (click also toggles the folder).
- Deleting a folder without removing its children (using a flat structure and leaving orphans).
- Not handling blur / Esc on the inline input (stuck input).

