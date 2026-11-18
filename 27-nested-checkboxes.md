# Nested Checkboxes (Tree Select) — Machine Coding

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

Build a **tree of checkboxes** (think "select permissions", "choose categories", file-tree pickers) with parent/child selection rules:

- Checking a **parent** checks **all** of its descendants (unchecking clears them).
- A parent whose children are only **partly** checked shows the **indeterminate** state.
- When **every** child gets checked, the parent **auto-checks** itself (and un-checks when the last one is cleared).
- Branches can be **expanded / collapsed**.

The data is an arbitrarily deep tree and every state update must be **immutable**.

---

## 2. Requirements

### Functional

- Render a recursive tree from `[{ id, label, children? }]`.
- Clicking a parent toggles its whole subtree: *checked → clear all*, *unchecked or indeterminate → check all*.
- Parent state is always consistent with its children: `checked` (all), `unchecked` (none), `indeterminate` (some).
- Expand / collapse per branch, plus "Expand all" / "Collapse all".
- Collapsing a branch must **not** change the selection.
- Show the current selection (e.g. the selected leaf ids).

### Non-functional

- Pure, unit-testable helpers (`computeStatus`, `toggleLeaves`) — no React inside.
- Single source of truth: **no** state that can drift out of sync (parent says checked, child says not).
- Controlled component: `checked` + `onChange(nextSet)`.
- Works for any depth; update cost proportional to the tree size, not exponential.

---

## 3. Clarifying Questions to Ask

- Is the tree **static** or loaded lazily (children fetched on expand)? *(Assume static; mention lazy.)*
- What should the **output** be — only selected leaves, or parents too when fully selected? *(Leaves; parents derivable.)*
- Clicking an **indeterminate** parent: select all or clear all? *(Select all — the common convention.)*
- Should collapsed branches still count toward selection? *(Yes.)*
- Disabled nodes? Max selection? Search/filter?
- Initially expanded or collapsed? Persist expansion?
- Is the data nested (`children`) or flat with `parentId`?

---

## 4. Approach & Component Design

**Data shape**

```js
{ id: 'frontend', label: 'Frontend', children: [ { id: 'react', label: 'React' }, … ] }
```

### Two ways to model the checked state

| | **A. Derive from a `Set` of leaf ids** *(recommended)* | **B. Propagate on toggle** (status stored on every node) |
|---|---|---|
| State | `checked: Set<leafId>` (+ `expanded: Set<id>`) | `status` field on every node (`checked / unchecked / indeterminate`) |
| Toggle a node | add/remove all its leaf ids in a **new Set** | set subtree to the new value, then **recompute every ancestor** from its children |
| Parent state | **derived** each render by one post-order pass | **stored**; must be kept consistent by the toggle code |
| Can it go out of sync? | No — parents have no state of their own | Yes, if any code path forgets to recompute ancestors (new child added, data reloaded…) |
| Output / persistence | trivial: the Set *is* the answer, JSON-friendly | need to walk the tree to extract selected ids |
| Data vs UI state | separate (data tree stays untouched) | mixed (selection lives inside the data tree) |
| Re-render story | whole tree re-derives (cheap); memoise rows if huge | structural sharing: only the changed path gets new objects → easy `memo` per node |
| Lazy-loaded children | selection of not-yet-loaded children is awkward | same problem; both need a rule for "select all" with unloaded kids |

**Recommendation: A.** Only *leaves* are real state; every parent value is a pure function of its leaves, so inconsistent states are impossible and the Set is exactly what a form/API wants. Choose B only when each node must carry extra per-node state and you want memoised, path-copied updates — it is included in the solution as `propagateAlt.js` for comparison.

**State placement**

| State | Where | Why |
|---|---|---|
| `checked: Set<leafId>` | parent (`App`), passed as controlled prop | the output of the component |
| `expanded: Set<id>` | `CheckboxTree` | purely UI; survives collapse/expand because it is not stored in unmounted nodes |
| `leafIds`, `parentIds`, `status` | **derived** with `useMemo` | never stored |

```
App  (checked: Set)
 └─ CheckboxTree  (expanded: Set · memo: leafIndex, status Map · Context)
     ├─ toolbar  [Expand all] [Collapse all]
     └─ <ul>
         └─ TreeNode ◄───────────────────────────────┐ recursion
             ├─ expander <button aria-expanded>      │
             ├─ TriStateCheckbox (ref → .indeterminate)
             └─ <ul> children.map → TreeNode ────────┘  (only when expanded)
```

**Toggle rule (Approach A):** `leafIds.get(node.id)` is the list of leaves under the node. If the node's status is `checked` → delete those ids; otherwise → add them. Always build a **new** `Set`.

**Approach B in one sentence:** `toggleNode(tree, id)` path-copies down to the node, overwrites the subtree's `status`, then on the way back up each ancestor recomputes its status from its children.

**Getting `indeterminate`:** it is a DOM *property* only (no HTML attribute, no React prop). Set it from a ref inside an effect: `ref.current.indeterminate = status === 'indeterminate'`.

---

## 5. Reference Solution

<details>
<summary><strong>Show Full Code (treeUtils.js, propagateAlt.js, TriStateCheckbox.jsx, CheckboxTree.jsx, App.jsx, styles.css)</strong></summary>

```jsx
// treeUtils.js — pure helpers for Approach A (Set of leaf ids)
const isLeaf = (n) => !n.children || n.children.length === 0;

// id -> ids of every leaf underneath it (a leaf maps to itself). Build once per data change.
export function buildLeafIndex(nodes) {
  const index = new Map();
  const walk = (n) => {
    const ids = isLeaf(n) ? [n.id] : n.children.flatMap((c) => walk(c));
    index.set(n.id, ids);
    return ids;
  };
  nodes.forEach((n) => walk(n));
  return index;
}

export function collectParentIds(nodes) {
  const out = [];
  const walk = (n) => {
    if (isLeaf(n)) return;
    out.push(n.id);
    n.children.forEach(walk);
  };
  nodes.forEach(walk);
  return out;
}

// One post-order pass -> Map<id, 'checked' | 'unchecked' | 'indeterminate'>
export function computeStatus(nodes, checked) {
  const status = new Map();
  const walk = (n) => {
    let s;
    if (isLeaf(n)) {
      s = checked.has(n.id) ? 'checked' : 'unchecked';
    } else {
      let on = 0;
      let off = 0;
      for (const child of n.children) {
        const cs = walk(child);
        if (cs === 'checked') on += 1;
        else if (cs === 'unchecked') off += 1;
      }
      if (on === n.children.length) s = 'checked';
      else if (off === n.children.length) s = 'unchecked';
      else s = 'indeterminate';
    }
    status.set(n.id, s);
    return s;
  };
  nodes.forEach((n) => walk(n));
  return status;
}

// checked -> clear the subtree; unchecked / indeterminate -> select the subtree. Returns a NEW Set.
export function toggleLeaves(checked, leafIds, currentStatus) {
  const next = new Set(checked);
  for (const id of leafIds) {
    if (currentStatus === 'checked') next.delete(id);
    else next.add(id);
  }
  return next;
}
```

```jsx
// propagateAlt.js — Approach B: `status` is stored on every node and propagated on toggle
export function initStatus(nodes) {
  return nodes.map((n) => ({
    ...n,
    status: 'unchecked',
    ...(n.children ? { children: initStatus(n.children) } : {}),
  }));
}

const setSubtree = (n, status) =>
  n.children
    ? { ...n, status, children: n.children.map((c) => setSubtree(c, status)) }
    : { ...n, status };

function summarise(children) {
  if (children.every((c) => c.status === 'checked')) return 'checked';
  if (children.every((c) => c.status === 'unchecked')) return 'unchecked';
  return 'indeterminate';
}

function findNode(nodes, id) {
  for (const n of nodes) {
    if (n.id === id) return n;
    const hit = n.children && findNode(n.children, id);
    if (hit) return hit;
  }
  return null;
}

// copy the path root -> target, overwrite the subtree, recompute each ancestor on the way back up
function update(node, id, status) {
  if (node.id === id) return setSubtree(node, status);
  if (!node.children) return node;
  let changed = false;
  const children = node.children.map((c) => {
    const next = update(c, id, status);
    if (next !== c) changed = true;
    return next;
  });
  return changed ? { ...node, children, status: summarise(children) } : node;
}

export function toggleNode(nodes, id) {
  const target = findNode(nodes, id);
  if (!target) return nodes;
  const next = target.status === 'checked' ? 'unchecked' : 'checked';
  return nodes.map((n) => update(n, id, next));
}
```

```jsx
// TriStateCheckbox.jsx
import { useLayoutEffect, useRef } from 'react';

export default function TriStateCheckbox({ status, label, onChange }) {
  const ref = useRef(null);

  // `indeterminate` is a DOM property, not an attribute, and React has no prop for it.
  // Sync after EVERY render (no deps): a native click clears it on its own, even if the parent
  // then ignores the change, so a deps-based effect could leave the DOM out of date.
  useLayoutEffect(() => {
    ref.current.indeterminate = status === 'indeterminate';
  });

  return (
    <label className="cb">
      <input
        ref={ref}
        type="checkbox"
        checked={status === 'checked'}
        onChange={onChange}
      />
      <span>{label}</span>
    </label>
  );
}
```

```jsx
// CheckboxTree.jsx
import { createContext, useCallback, useContext, useId, useMemo, useState } from 'react';
import TriStateCheckbox from './TriStateCheckbox';
import { buildLeafIndex, collectParentIds, computeStatus, toggleLeaves } from './treeUtils';

const TreeContext = createContext(null);

function TreeNode({ node }) {
  const { status, expanded, toggleCheck, toggleExpand } = useContext(TreeContext);
  const groupId = useId();
  const hasChildren = Boolean(node.children && node.children.length);
  const isOpen = hasChildren && expanded.has(node.id);

  return (
    <li className="tree-item">
      <div className="tree-row">
        {hasChildren ? (
          <button
            type="button"
            className="expander"
            aria-expanded={isOpen}
            aria-controls={isOpen ? groupId : undefined}
            aria-label={`Toggle ${node.label}`}
            onClick={() => toggleExpand(node.id)}
          >
            <span aria-hidden="true">▸</span>
          </button>
        ) : (
          <span className="expander-spacer" />
        )}
        <TriStateCheckbox
          status={status.get(node.id)}
          label={node.label}
          onChange={() => toggleCheck(node.id)}
        />
      </div>
      {isOpen && (
        <ul id={groupId} className="tree-list nested">
          {node.children.map((child) => (
            <TreeNode key={child.id} node={child} />
          ))}
        </ul>
      )}
    </li>
  );
}

export default function CheckboxTree({ data, checked, onChange, label = 'Options' }) {
  const leafIndex = useMemo(() => buildLeafIndex(data), [data]);
  const parentIds = useMemo(() => collectParentIds(data), [data]);
  const status = useMemo(() => computeStatus(data, checked), [data, checked]);
  const [expanded, setExpanded] = useState(() => new Set(data.map((n) => n.id)));

  const toggleCheck = useCallback(
    (id) => onChange(toggleLeaves(checked, leafIndex.get(id), status.get(id))),
    [checked, leafIndex, status, onChange]
  );

  const toggleExpand = useCallback((id) => {
    setExpanded((prev) => {
      const next = new Set(prev);
      if (next.has(id)) next.delete(id);
      else next.add(id);
      return next;
    });
  }, []);

  const ctx = useMemo(
    () => ({ status, expanded, toggleCheck, toggleExpand }),
    [status, expanded, toggleCheck, toggleExpand]
  );

  return (
    <TreeContext.Provider value={ctx}>
      <div className="tree">
        <div className="tree-toolbar">
          <button type="button" onClick={() => setExpanded(new Set(parentIds))}>
            Expand all
          </button>
          <button type="button" onClick={() => setExpanded(new Set())}>
            Collapse all
          </button>
        </div>
        <ul className="tree-list" aria-label={label}>
          {data.map((node) => (
            <TreeNode key={node.id} node={node} />
          ))}
        </ul>
      </div>
    </TreeContext.Provider>
  );
}
```

```jsx
// App.jsx
import { useState } from 'react';
import CheckboxTree from './CheckboxTree';
import './styles.css';

const DATA = [
  {
    id: 'frontend',
    label: 'Frontend',
    children: [
      {
        id: 'frameworks',
        label: 'Frameworks',
        children: [
          { id: 'react', label: 'React' },
          { id: 'vue', label: 'Vue' },
          { id: 'svelte', label: 'Svelte' },
        ],
      },
      { id: 'css', label: 'CSS' },
      { id: 'html', label: 'HTML' },
    ],
  },
  {
    id: 'backend',
    label: 'Backend',
    children: [
      { id: 'node', label: 'Node.js' },
      { id: 'go', label: 'Go' },
    ],
  },
  { id: 'devops', label: 'DevOps' },
];

export default function App() {
  const [checked, setChecked] = useState(() => new Set(['react']));

  return (
    <main className="page">
      <h1>Skills</h1>
      <CheckboxTree label="Skills" data={DATA} checked={checked} onChange={setChecked} />
      <p>
        Selected ({checked.size}): {checked.size ? [...checked].join(', ') : 'none'}
      </p>
    </main>
  );
}
```

```css
/* styles.css */
.page { max-width: 420px; margin: 40px auto; font-family: system-ui, sans-serif; }
.tree-toolbar { display: flex; gap: 8px; margin-bottom: 8px; }
.tree-toolbar button { padding: 4px 10px; border: 1px solid #cbd5e1; border-radius: 6px; background: #fff; cursor: pointer; font-size: 13px; }
.tree-list { list-style: none; margin: 0; padding: 0; }
.tree-list.nested { margin-left: 11px; padding-left: 14px; border-left: 1px solid #e2e8f0; }
.tree-row { display: flex; align-items: center; gap: 4px; min-height: 32px; }
.expander, .expander-spacer { width: 24px; height: 24px; flex: none; }
.expander { border: 0; background: none; cursor: pointer; color: #475569; border-radius: 4px; padding: 0; }
.expander span { display: inline-block; transition: transform 0.15s; }
.expander[aria-expanded='true'] span { transform: rotate(90deg); }
.cb { display: inline-flex; align-items: center; gap: 8px; cursor: pointer; user-select: none; padding: 4px 0; }
.cb input { width: 16px; height: 16px; margin: 0; cursor: pointer; }
button:focus-visible, input:focus-visible { outline: 2px solid #2563eb; outline-offset: 2px; }
```

</details>

---

## 6. How It Works

- **Only leaves are state.** `checked` holds leaf ids. A parent has no state of its own, so "parent says checked but a child isn't" cannot happen.
- **Status is derived in one pass.** `computeStatus` is a post-order DFS: leaves read from the Set, a parent counts how many children are `checked` / `unchecked` — all checked → `checked`, none → `unchecked`, otherwise `indeterminate`. That is also the *auto-check parent when all children are checked* rule, for free. The result is a `Map<id, status>` memoised on `[data, checked]`; each node reads its own entry in O(1).
- **Toggling a parent.** `buildLeafIndex` (computed once per `data`) gives every node's leaf list. `toggleLeaves` copies the Set and adds or deletes them: `checked` → delete, `unchecked` / `indeterminate` → add. A leaf maps to itself, so leaves and parents use the **same code path**.
- **Immutable updates.** Every change creates a new `Set`; React sees a new reference and re-renders. The data tree itself is never touched.
- **Indeterminate via ref.** `TriStateCheckbox` owns a ref to the `<input>` and assigns `ref.current.indeterminate` in a layout effect on every render. React cannot do this declaratively because it is a DOM property. `checked` is driven separately and is `false` while indeterminate, so the next click fires `onChange` and selects all.
- **Expand / collapse is independent.** `expanded` is a `Set` of ids kept in `CheckboxTree`, **not** in each node, so it survives re-renders. Collapsed children are simply not rendered, but their status is still computed from data, so a collapsed parent still shows the right tri-state.
- **Context** hands `status`, `expanded` and the handlers to any depth without prop drilling; the value is memoised so it only changes when something relevant does.
- **Approach B** (`propagateAlt.js`): `update` path-copies down to the target, `setSubtree` stamps the new status on all descendants, and on the way back each ancestor recalculates via `summarise`. It works, but correctness depends on *every* mutation going through this function.

---

## 7. Edge Cases

- **Click on an indeterminate parent** → selects everything beneath it (not a clear).
- **Last child unchecked** → parent flips from `checked` / `indeterminate` to `unchecked` automatically (derived).
- **Collapsed branch** → selection unchanged; collapsing never mutates `checked`.
- **Parent with an empty `children: []`** → treated as a leaf (`isLeaf` handles `undefined` and `[]`), otherwise it could never become checked.
- **Duplicate ids** → corrupt the Set and the Maps; ids must be unique across the whole tree.
- **`checked` contains an id that no longer exists** (data reloaded) → ignored by `computeStatus`, but still reported in `checked.size`; prune on data change.
- **Deep trees** → recursion depth equals tree depth, fine for UI-sized data; use an iterative DFS for pathological depth.
- **Data prop replaced** → `expanded` keeps old ids (harmless); new branches start collapsed unless handled.
- **Lazy-loaded children** → a parent with unloaded children can't know its true status; track `loaded`/`totalCount` or treat the server's "all selected" flag as the state.
- **Disabled nodes** → skip them in `toggleLeaves` and compute status only over enabled leaves.
- **Rapid clicks** → every toggle derives from the latest Set; use the functional `setChecked(prev => …)` form if toggles may batch.

---

## 8. Accessibility

- Use **native `<input type="checkbox">`** inside a `<label>`: free keyboard (Space), focus ring, and click target on the label text.
- Setting `el.indeterminate = true` makes browsers and screen readers expose the **mixed** state automatically — don't add a conflicting `aria-checked`.
- Expander is a real `<button>` with `aria-expanded` (+ `aria-controls` while the group exists). Keep its `aria-label` **fixed** (`Toggle Frontend`) — changing the label *and* using `aria-expanded` double-announces the state.
- Nested `ul/li` exposes the hierarchy ("list, 3 items, nested"). For a full ARIA tree use `role="tree"`, `treeitem`, `aria-checked="mixed"` and arrow-key navigation (Left/Right expand/collapse, Up/Down move) with roving `tabindex` — a follow-up, not required for the base task.
- Never rely on colour alone — the native indeterminate dash and the ▸/▾ rotation convey state.
- Visible `:focus-visible` outlines; click targets ≥ 24px (the expander) and the whole label is clickable.
- Announce bulk results if relevant (`aria-live="polite"` "5 selected").

---

## 9. Performance Considerations

- `computeStatus` is **O(n)** — one pass — and memoised on `[data, checked]`; `buildLeafIndex` is O(n·depth) worst case and runs only when `data` changes.
- A toggle does O(leaves under node) Set work; the render re-derives O(n). Fine up to a few thousand nodes.
- Each `TreeNode` re-renders on every toggle because the context value changes. If the tree is huge: `memo` rows and pass `status` per node instead of via a shared Map, or move to Approach B's structural sharing so unchanged subtrees keep identical props.
- Collapsed branches are **not rendered**, so large trees stay cheap; for 10k+ visible rows **virtualise** a flattened list (`[{node, depth}]` of expanded nodes).
- Precomputing `leafIds` per node avoids walking the subtree on every click.
- Don't store parent status in React state — extra state means extra renders **and** consistency bugs.

---

## 10. Follow-up Questions

<details>
<summary><strong>1. Why store only leaf ids instead of every node id?</strong></summary>

A parent's value is a pure function of its leaves; storing it duplicates information and creates a sync problem. With leaves only, the Set is minimal, consistent by construction, and is exactly the payload the backend wants. If you must send parents too, derive them from `computeStatus` (entries with `checked`).

</details>

<details>
<summary><strong>2. How do you make it controlled / work with forms?</strong></summary>

It already is: `checked` + `onChange(nextSet)`. For native forms, render hidden inputs per selected leaf (`<input type="hidden" name="skills" value={id}>`) or serialise `[...checked]` yourself. With form libraries wire `value` / `onChange` through a `Controller`.

</details>

<details>
<summary><strong>3. How would you add search / filter?</strong></summary>

Compute a filtered tree (keep a node if it or any descendant matches; auto-expand those branches). Status is still computed from the **full** tree so hidden leaves are respected — or decide explicitly that a parent "select all" affects only visible matches, in which case `toggleLeaves` gets the filtered leaf ids.

</details>

<details>
<summary><strong>4. What changes if the tree is huge or children load lazily?</strong></summary>

Virtualise a flattened visible list. For lazy children keep `totalLeafCount` from the server; a parent is `checked` when `selectedCount === totalLeafCount`, and "select all" on an unloaded branch stores a *branch id* ("everything under X") plus an exclusion list instead of enumerating leaves.

</details>

<details>
<summary><strong>5. Why `useLayoutEffect` without a dependency array for `indeterminate`?</strong></summary>

The property is imperative DOM state and the browser resets it on click. Running after every render guarantees the DOM matches `status` even when the parent rejects or ignores a change. Layout effect (vs `useEffect`) avoids a frame where the box shows the wrong mark. It's a trivial assignment, so running it every render is cheap.

</details>

<details>
<summary><strong>6. When would you pick Approach B (propagate on toggle) instead?</strong></summary>

When the node object must carry more per-node state (loading, disabled, metadata) and you want memoised rows: path-copying gives unchanged subtrees identical references, so `React.memo` can skip them. The cost is keeping ancestors consistent on **every** mutation (adds, removals, reloads) and extracting the selected ids by walking the tree.

</details>

<details>
<summary><strong>7. Add keyboard navigation like a real tree view.</strong></summary>

Use `role="tree"` / `treeitem` / `group`, `aria-expanded`, `aria-checked` (`true/false/mixed`), and roving `tabindex` (one tab stop). Flatten the visible nodes into an array; ↑/↓ move the index, → expands or moves to the first child, ← collapses or moves to the parent, Space toggles the checkbox, Home/End jump.

</details>

<details>
<summary><strong>8. How do you support "disabled" nodes or a max selection?</strong></summary>

Disabled: exclude disabled leaves from `leafIndex` when toggling and from the counts in `computeStatus` (a parent with only disabled leaves stays unchecked). Max N: guard in `toggleLeaves` — if the resulting Set would exceed N, ignore the toggle (or select only the first N) and show a message.

</details>

---

## 11. Common Mistakes

- Keeping a **separate boolean for every node** and updating them independently — parent and children drift apart.
- **Mutating** state (`checked.add(id); setChecked(checked)`) — same reference, no re-render.
- Trying to pass `indeterminate` as a JSX prop/attribute — it does nothing; it must be set on the DOM node via a ref.
- Running the `indeterminate` effect only with `[status]` deps and leaving the DOM stale after the native click.
- Treating an indeterminate click as "uncheck" (leaves the user unable to select everything in one click).
- Storing `expanded` inside each node component and losing it when the parent re-mounts or collapses.
- Deriving parent status from children **in the render of each row** by re-walking the subtree (O(n²)) instead of one memoised pass.
- Forgetting that `children: []` is a leaf.
- Using array index as `key`.
- Changing the expander's `aria-label` **and** `aria-expanded`, so state is announced twice.
- Recursion on the *rendered* children only (collapsed children not counted) when computing status.
