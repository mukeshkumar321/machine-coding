# Data Table (Sort, Filter, Search, Pagination, Resize) — Machine Coding

> **Difficulty:** Hard · **Time:** 75 min · **Asked at / Frequency:** Commonly reported at product companies (Atlassian, Uber, Razorpay, Microsoft) — High · **Tier:** 2

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

Build a client-side **data table** over a list of employees with:

- **Sorting** by clicking column headers (asc → desc → none)
- A **column filter** (department) and a global **search**
- **Pagination** with a page-size selector
- **Resizable columns** (drag the header edge)
- Derived data computed with `useMemo` (filter → sort → paginate)

---

## 2. Requirements

### Functional

- Render rows from a column config (`key`, `label`, `type`, `width`, optional `render`).
- Click a sortable header to cycle asc → desc → unsorted; show an indicator; type-aware comparison (number / string / date).
- Search across searchable columns (case-insensitive); department dropdown filter; both combine (AND).
- Pagination (Prev/Next, "Page x of y", rows-per-page, "Showing a–b of N"); resets to page 1 when filters/sort change.
- Drag a header's right edge to resize; min width; keyboard resizable.
- Empty state when nothing matches.

### Non-functional

- Pipeline `rows → filter → sort → slice` as memoised derivations; **never mutate** `rows` (`[...rows].sort`).
- Table logic in a hook (`useDataTable`), column-width logic in another (`useColumnWidths`).
- Accessible table semantics (`aria-sort`, captions, labelled controls).
- Resizing must not re-render every row.

---

## 3. Clarifying Questions to Ask

- Client-side or server-side sort/filter/pagination? Data size (hundreds / millions)?
- Single-column or multi-column sort?
- Which columns are searchable? Per-column filters or only a global search?
- Should resizing persist? Reordering / hiding columns? Row selection? Sticky header?
- Stable sort required for ties? How to sort empty/null values?
- Is `<table>` markup required, or can I use CSS grid?

---

## 4. Approach & Component Design

**State (in `useDataTable`)** — only *inputs* are state:

| State | Shape |
|---|---|
| `search` | `string` |
| `department` | `'all' \| string` |
| `sort` | `{ key: string \| null, dir: 'asc' \| 'desc' \| null }` |
| `page`, `pageSize` | numbers (1-based page) |
| `widths` *(in `useColumnWidths`)* | `{ [columnKey]: px }` |

**Derived pipeline (each step memoised on its inputs)**

```
rows ──filter(search, department)──► filtered ──sortRows(sort)──► sorted ──slice(page, pageSize)──► pageRows
          deps: [rows,search,dept]       deps: [filtered, sort]          deps: [sorted, page, pageSize]
```

**Component tree**

```
App
 └─ DataTable
     ├─ Toolbar:  <input type=search/>  <select department/>  <status "N results"/>
     ├─ <div.table-wrap (overflow-x)>
     │    <table (width = Σ column widths, table-layout: fixed)>
     │      <colgroup> <col width=…/> × columns </colgroup>
     │      <thead> <th aria-sort> [sort button] [<ResizeHandle/>] </th> … </thead>
     │      <tbody> <Row (memo)/> × pageSize | empty state </tbody>
     └─ Pager: Prev · Page x of y · Next · rows-per-page
```

Resizing changes only `<col>` widths + table width → memoised `Row`s don't re-render.

---

## 5. Reference Solution

<details>
<summary><strong>Show Full Code (data.js, tableUtils.js, useDataTable.js, useColumnWidths.js, ResizeHandle.jsx, DataTable.jsx, App.jsx, styles.css)</strong></summary>

```jsx
// data.js
const FIRST = ['Aarav', 'Diya', 'Kabir', 'Meera', 'Rohan', 'Sana', 'Vikram', 'Isha', 'Arjun', 'Nisha'];
const LAST = ['Sharma', 'Verma', 'Khan', 'Iyer', 'Patel', 'Singh', 'Das', 'Reddy', 'Gupta', 'Nair'];
export const DEPARTMENTS = ['Engineering', 'Design', 'Marketing', 'Sales', 'HR'];
const LEVELS = ['Junior', 'Mid', 'Senior', 'Lead'];

const pad = (n) => String(n).padStart(2, '0');

export const ROWS = Array.from({ length: 137 }, (_, i) => {
  const first = FIRST[i % FIRST.length];
  const last = LAST[(i * 7 + 3) % LAST.length];
  return {
    id: i + 1,
    name: `${first} ${last}`,
    email: `${first}.${last}${i + 1}@example.com`.toLowerCase(),
    department: DEPARTMENTS[(i * 3) % DEPARTMENTS.length],
    level: LEVELS[(i * 5) % LEVELS.length],
    salary: 40000 + ((i * 7919) % 80) * 1000,
    joined: `${2015 + (i % 10)}-${pad(1 + (i % 12))}-${pad(1 + ((i * 3) % 28))}`,
  };
});

// module-level constant => stable reference for useMemo dependencies
export const COLUMNS = [
  { key: 'name', label: 'Name', type: 'string', width: 180, sortable: true, searchable: true },
  { key: 'email', label: 'Email', type: 'string', width: 260, sortable: true, searchable: true },
  { key: 'department', label: 'Department', type: 'string', width: 140, sortable: true, searchable: true },
  { key: 'level', label: 'Level', type: 'string', width: 100, sortable: true, searchable: true },
  {
    key: 'salary', label: 'Salary', type: 'number', width: 120, sortable: true, align: 'right',
    render: (v) => `₹${v.toLocaleString('en-IN')}`,
  },
  { key: 'joined', label: 'Joined', type: 'date', width: 120, sortable: true },
];
```

```jsx
// tableUtils.js — pure functions
export function filterRows(rows, columns, search, department) {
  const q = search.trim().toLowerCase();
  const searchable = columns.filter((c) => c.searchable);

  return rows.filter((row) => {
    if (department !== 'all' && row.department !== department) return false;
    if (!q) return true;
    return searchable.some((c) => String(row[c.key]).toLowerCase().includes(q));
  });
}

export function sortRows(rows, columns, { key, dir }) {
  if (!key || !dir) return rows; // unsorted: keep original order (same reference)
  const col = columns.find((c) => c.key === key);
  if (!col) return rows;

  const factor = dir === 'asc' ? 1 : -1;
  const compare =
    col.type === 'string'
      ? (a, b) => String(a).localeCompare(String(b), undefined, { numeric: true, sensitivity: 'base' })
      : (a, b) => (a < b ? -1 : a > b ? 1 : 0); // numbers and ISO dates

  // copy first - Array.prototype.sort mutates; sort is stable so ties keep their order
  return [...rows].sort((a, b) => factor * compare(a[key], b[key]));
}
```

```jsx
// useDataTable.js
import { useMemo, useState } from 'react';
import { filterRows, sortRows } from './tableUtils';

export function useDataTable({ rows, columns, initialPageSize = 10 }) {
  const [search, setSearchState] = useState('');
  const [department, setDepartmentState] = useState('all');
  const [sort, setSort] = useState({ key: null, dir: null });
  const [page, setPage] = useState(1);
  const [pageSize, setPageSizeState] = useState(initialPageSize);

  const filtered = useMemo(
    () => filterRows(rows, columns, search, department),
    [rows, columns, search, department]
  );
  const sorted = useMemo(() => sortRows(filtered, columns, sort), [filtered, columns, sort]);

  const totalItems = sorted.length;
  const totalPages = Math.max(1, Math.ceil(totalItems / pageSize));
  const currentPage = Math.min(page, totalPages);
  const startIndex = (currentPage - 1) * pageSize;
  const endIndex = Math.min(startIndex + pageSize, totalItems);

  const pageRows = useMemo(
    () => sorted.slice(startIndex, endIndex),
    [sorted, startIndex, endIndex]
  );

  // every input that changes the result set resets to page 1 in the same event
  const setSearch = (value) => {
    setSearchState(value);
    setPage(1);
  };
  const setDepartment = (value) => {
    setDepartmentState(value);
    setPage(1);
  };
  const setPageSize = (size) => {
    setPageSizeState(size);
    setPage(1);
  };
  const toggleSort = (key) => {
    setSort((s) => {
      if (s.key !== key) return { key, dir: 'asc' };
      if (s.dir === 'asc') return { key, dir: 'desc' };
      return { key: null, dir: null };
    });
    setPage(1);
  };
  const clearFilters = () => {
    setSearchState('');
    setDepartmentState('all');
    setPage(1);
  };

  return {
    search, department, sort, pageSize, currentPage, totalPages, totalItems, startIndex, endIndex,
    pageRows,
    setSearch, setDepartment, setPageSize, toggleSort, clearFilters,
    setPage: (p) => setPage(Math.min(Math.max(p, 1), totalPages)),
  };
}
```

```jsx
// useColumnWidths.js
import { useCallback, useState } from 'react';

export function useColumnWidths(columns, minWidth = 60) {
  const [widths, setWidths] = useState(() =>
    Object.fromEntries(columns.map((c) => [c.key, c.width]))
  );

  const resize = useCallback(
    (key, width) => setWidths((w) => ({ ...w, [key]: Math.max(minWidth, Math.round(width)) })),
    [minWidth]
  );

  const totalWidth = columns.reduce((sum, c) => sum + widths[c.key], 0);
  return { widths, resize, totalWidth };
}
```

```jsx
// ResizeHandle.jsx
import { useRef } from 'react';

export default function ResizeHandle({ label, width, onResize }) {
  const drag = useRef(null);

  const onPointerDown = (e) => {
    e.preventDefault();
    e.currentTarget.setPointerCapture(e.pointerId); // keep receiving moves outside the handle
    drag.current = { startX: e.clientX, startWidth: width };
  };

  const onPointerMove = (e) => {
    if (!drag.current) return;
    onResize(drag.current.startWidth + (e.clientX - drag.current.startX));
  };

  const endDrag = (e) => {
    drag.current = null;
    if (e.currentTarget.hasPointerCapture(e.pointerId)) {
      e.currentTarget.releasePointerCapture(e.pointerId);
    }
  };

  const onKeyDown = (e) => {
    if (e.key === 'ArrowLeft') {
      e.preventDefault();
      onResize(width - 10);
    } else if (e.key === 'ArrowRight') {
      e.preventDefault();
      onResize(width + 10);
    }
  };

  return (
    <span
      className="resizer"
      role="separator"
      aria-orientation="vertical"
      aria-label={`Resize ${label} column`}
      aria-valuenow={width}
      tabIndex={0}
      onPointerDown={onPointerDown}
      onPointerMove={onPointerMove}
      onPointerUp={endDrag}
      onPointerCancel={endDrag}
      onKeyDown={onKeyDown}
    />
  );
}
```

```jsx
// DataTable.jsx
import { memo } from 'react';
import { useDataTable } from './useDataTable';
import { useColumnWidths } from './useColumnWidths';
import ResizeHandle from './ResizeHandle';

const PAGE_SIZES = [5, 10, 25, 50];
const SORT_ICON = { asc: '▲', desc: '▼' };
const ARIA_SORT = { asc: 'ascending', desc: 'descending' };

// memoised: re-renders only when the row object changes, not while columns are being resized
const Row = memo(function Row({ row, columns }) {
  return (
    <tr>
      {columns.map((c) => (
        <td key={c.key} style={{ textAlign: c.align }}>
          {c.render ? c.render(row[c.key], row) : row[c.key]}
        </td>
      ))}
    </tr>
  );
});

export default function DataTable({ rows, columns, departments }) {
  const t = useDataTable({ rows, columns });
  const { widths, resize, totalWidth } = useColumnWidths(columns);
  const isFiltered = t.search !== '' || t.department !== 'all';

  return (
    <section className="dt">
      <div className="toolbar">
        <label>
          Search
          <input
            type="search"
            placeholder="Name, email, department…"
            value={t.search}
            onChange={(e) => t.setSearch(e.target.value)}
          />
        </label>

        <label>
          Department
          <select value={t.department} onChange={(e) => t.setDepartment(e.target.value)}>
            <option value="all">All departments</option>
            {departments.map((d) => (
              <option key={d} value={d}>{d}</option>
            ))}
          </select>
        </label>

        {isFiltered && (
          <button type="button" onClick={t.clearFilters}>Clear filters</button>
        )}

        <p className="status" role="status" aria-live="polite">
          {t.totalItems} {t.totalItems === 1 ? 'result' : 'results'}
        </p>
      </div>

      <div className="table-wrap" role="region" aria-label="Employees table" tabIndex={0}>
        <table style={{ width: totalWidth }}>
          <caption className="sr-only">Employees</caption>
          <colgroup>
            {columns.map((c) => (
              <col key={c.key} style={{ width: widths[c.key] }} />
            ))}
          </colgroup>
          <thead>
            <tr>
              {columns.map((c) => {
                const dir = t.sort.key === c.key ? t.sort.dir : null;
                return (
                  <th
                    key={c.key}
                    scope="col"
                    aria-sort={c.sortable ? ARIA_SORT[dir] || 'none' : undefined}
                    style={{ textAlign: c.align }}
                  >
                    {c.sortable ? (
                      <button type="button" className="sort-btn" onClick={() => t.toggleSort(c.key)}>
                        {c.label}
                        <span className="sort-icon" aria-hidden="true">
                          {SORT_ICON[dir] || '↕'}
                        </span>
                      </button>
                    ) : (
                      c.label
                    )}
                    <ResizeHandle
                      label={c.label}
                      width={widths[c.key]}
                      onResize={(w) => resize(c.key, w)}
                    />
                  </th>
                );
              })}
            </tr>
          </thead>
          <tbody>
            {t.pageRows.length === 0 ? (
              <tr>
                <td colSpan={columns.length} className="empty">No matching records</td>
              </tr>
            ) : (
              t.pageRows.map((row) => <Row key={row.id} row={row} columns={columns} />)
            )}
          </tbody>
        </table>
      </div>

      <div className="pager">
        <span>
          Showing {t.totalItems === 0 ? 0 : t.startIndex + 1}–{t.endIndex} of {t.totalItems}
        </span>
        <div className="pager-controls">
          <button type="button" onClick={() => t.setPage(t.currentPage - 1)} disabled={t.currentPage === 1}>
            ‹ Prev
          </button>
          <span>Page {t.currentPage} of {t.totalPages}</span>
          <button
            type="button"
            onClick={() => t.setPage(t.currentPage + 1)}
            disabled={t.currentPage === t.totalPages}
          >
            Next ›
          </button>
        </div>
        <label>
          Rows per page
          <select value={t.pageSize} onChange={(e) => t.setPageSize(Number(e.target.value))}>
            {PAGE_SIZES.map((s) => (
              <option key={s} value={s}>{s}</option>
            ))}
          </select>
        </label>
      </div>
    </section>
  );
}
```

```jsx
// App.jsx
import DataTable from './DataTable';
import { COLUMNS, DEPARTMENTS, ROWS } from './data';
import './styles.css';

export default function App() {
  return (
    <main className="page">
      <h1>Employees</h1>
      <DataTable rows={ROWS} columns={COLUMNS} departments={DEPARTMENTS} />
    </main>
  );
}
```

```css
/* styles.css */
.page { max-width: 980px; margin: 24px auto; padding: 0 12px; font-family: system-ui, sans-serif; }
.sr-only { position: absolute; width: 1px; height: 1px; overflow: hidden; clip: rect(0 0 0 0); white-space: nowrap; }

.toolbar { display: flex; flex-wrap: wrap; align-items: flex-end; gap: 12px; margin-bottom: 12px; }
.toolbar label, .pager label { display: flex; flex-direction: column; gap: 4px; font-size: 13px; }
.pager label { flex-direction: row; align-items: center; gap: 8px; }
.toolbar input, .toolbar select, .pager select { padding: 6px 8px; border: 1px solid #94a3b8; border-radius: 6px; font: inherit; }
.toolbar .status { margin: 0 0 0 auto; color: #64748b; font-size: 13px; }
button { font: inherit; }
.toolbar button, .pager button { padding: 6px 12px; border: 1px solid #94a3b8; border-radius: 6px; background: #fff; cursor: pointer; }
.pager button:disabled { opacity: 0.45; cursor: not-allowed; }

.table-wrap { overflow-x: auto; border: 1px solid #e2e8f0; border-radius: 8px; }
table { table-layout: fixed; border-collapse: collapse; }
th, td { padding: 8px 10px; border-bottom: 1px solid #eef2f7; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; text-align: left; }
thead th { position: relative; background: #f8fafc; font-size: 13px; user-select: none; }
tbody tr:hover { background: #f8fafc; }
td.empty { text-align: center; color: #64748b; padding: 24px; }

.sort-btn { display: inline-flex; gap: 6px; align-items: center; border: 0; background: none; padding: 0; cursor: pointer; font-weight: 600; color: inherit; }
.sort-icon { color: #64748b; font-size: 11px; }
th[aria-sort='ascending'] .sort-icon, th[aria-sort='descending'] .sort-icon { color: #2563eb; }

.resizer { position: absolute; top: 0; right: 0; width: 8px; height: 100%; cursor: col-resize; touch-action: none; }
.resizer::after { content: ''; position: absolute; right: 3px; top: 25%; height: 50%; width: 2px; background: #cbd5e1; }
.resizer:hover::after, .resizer:focus-visible::after { background: #2563eb; }

.pager { display: flex; flex-wrap: wrap; align-items: center; justify-content: space-between; gap: 12px; margin-top: 12px; font-size: 14px; }
.pager-controls { display: flex; align-items: center; gap: 10px; }
button:focus-visible, input:focus-visible, select:focus-visible, .table-wrap:focus-visible, .resizer:focus-visible { outline: 2px solid #2563eb; outline-offset: 2px; }
```

</details>

---

## 6. How It Works

- **Inputs are state; everything else is derived.** `search`, `department`, `sort`, `page`, `pageSize` are the only state. `filtered → sorted → pageRows` are `useMemo` chains, each depending only on the previous result and its own inputs — changing the sort doesn't re-filter; changing the page doesn't re-sort.
- **Stable dependencies matter.** `COLUMNS` and `ROWS` are module-level constants (or memoised by the parent), otherwise every `useMemo` would recompute each render.
- **No mutation.** `sortRows` copies (`[...rows].sort`) because `Array.prototype.sort` sorts in place; mutating `filtered` (a memoised array) or the original `rows` is a classic bug. `Array.sort` is stable, so equal keys retain their previous order. When unsorted we return the same reference (cheap).
- **Type-aware comparators** — numbers/ISO dates with `<`/`>`; strings with `localeCompare` (`numeric: true`, case-insensitive). Never `a - b` on strings; never default `.sort()` on numbers (lexicographic).
- **3-state sort cycle** (asc → desc → none) in a functional `setSort`.
- **Page resets in the same handler** that changes the result set (no `useEffect` → no wrong-page render). `currentPage` is also clamped as a safety net.
- **Column resize with Pointer Events** — `setPointerCapture` on the handle keeps the drag alive when the cursor leaves it; width = `startWidth + (clientX - startX)` captured at pointer-down (no accumulated drift), clamped to a minimum. `table-layout: fixed` + `<colgroup>` makes widths authoritative; the table's width is the sum of the columns, so the wrapper scrolls horizontally rather than squashing columns.
- **`Row` is memoised**, and widths live on `<col>` elements — dragging a handle re-renders the header/colgroup only, not N×M cells.

---

## 7. Edge Cases

- No results → empty row with `colSpan`; "Showing 0–0 of 0"; Prev/Next disabled.
- Filtering/searching while on page 5 → reset to 1 (and clamp if data shrinks elsewhere).
- Sorting ties (equal salaries) → stable sort keeps previous order; mention secondary sort by `id` for determinism.
- `null`/`undefined` cells: decide where they go (usually last in both directions); `String(null)` would sort as "null".
- Mixed-case / accented strings → `localeCompare` with `sensitivity: 'base'`.
- Dates: ISO strings compare correctly lexicographically; other formats must be parsed first.
- Resize below min width / beyond the viewport (wrapper scrolls); drag ending outside the window (pointer capture handles it).
- Pointer released while over another header → no accidental sort (handle is separate from the sort button).
- Page size change keeps data consistent (reset to page 1; or recompute page to keep the first row visible).
- Search input with regex characters — we use `includes`, not `RegExp`, so `(` or `*` is safe.
- Very wide cell text → `text-overflow: ellipsis` (+ `title` attribute if needed).

---

## 8. Accessibility

- Real `<table>` with `<caption>` (visually hidden), `<th scope="col">`, `<colgroup>`.
- `aria-sort="ascending | descending | none"` on sortable headers; the sort control is a real `<button>` inside the `<th>` (not `onClick` on the `th`).
- Search/filter/page-size controls have visible labels; results count in `role="status"` / `aria-live="polite"`.
- Resize handle: `role="separator"` + `aria-orientation`, focusable, **← →** change width by 10 px (mouse-only resizing excludes keyboard users).
- Scroll container is focusable (`tabIndex={0}` + `role="region"` + label) so keyboard users can scroll horizontally.
- Icons (▲▼↕) are `aria-hidden`; sort state is conveyed by `aria-sort`, not colour alone.
- Visible `:focus-visible` outlines.

---

## 9. Performance Considerations

- `useMemo` pipeline avoids re-filtering/sorting on unrelated updates (page, widths, hover).
- Sorting is O(n log n) — fine for 10⁴ rows. For larger: sort/filter on the server, or in a **Web Worker**; precompute lower-cased search strings once per row (`row._search`) instead of `toLowerCase()` on every keystroke.
- **Debounce** the search (200–300 ms) or use `useDeferredValue(search)` so typing stays responsive on large data.
- `React.memo` `Row` with stable props; `key={row.id}` (never index) so rows aren't remounted when sorting.
- Resizing at 60 fps: only `<col>`/`<table>` style changes; for extreme cases write widths to a CSS variable / `ref` during the drag and commit state on pointer-up.
- Only `pageSize` rows are rendered; for "infinite" tables use **virtualisation** (react-window) with fixed row heights.

---

## 10. Follow-up Questions

<details>
<summary><strong>1. How would you make it server-side (sort/filter/paginate on the API)?</strong></summary>

`sort`, `search`, `department`, `page`, `pageSize` become the **query key**; `useEffect` (or React Query) fetches `{rows, total}` whenever it changes, with `AbortController` to drop stale responses, a loading overlay and `keepPreviousData`. The `useMemo` pipeline disappears (server does it); `totalItems` comes from the response. Debounce the search.

</details>

<details>
<summary><strong>2. Multi-column sort?</strong></summary>

`sort` becomes an ordered array `[{key, dir}, …]` (shift+click adds). Comparator loops through the keys until a non-zero result. Show priority numbers on headers; `aria-sort` stays per column.

</details>

<details>
<summary><strong>3. How do you virtualise rows?</strong></summary>

Render only visible rows + overscan inside a scroll container with a spacer of `totalRows * rowHeight`; translate the visible slice. With a `<table>`, use padding rows (top/bottom `<tr style={{height}}>`) or switch to CSS-grid/`div` rows with `role="table"`/`row`/`cell`. Pagination and virtualisation are alternatives, not usually combined.

</details>

<details>
<summary><strong>4. Add row selection with "select all on this page"?</strong></summary>

Keep `selected: Set<id>` (not row indices — they change with sort/filter). Header checkbox: checked if all `pageRows` ids are selected, `indeterminate` if some. Decide semantics for "select all N results" (a banner like Gmail). Keep selection across pages.

</details>

<details>
<summary><strong>5. Column reordering and hiding?</strong></summary>

Keep `columnOrder` (array of keys) and `hidden` (Set) in state; derive `visibleColumns` with `useMemo` and map headers/cells from it. Reorder by drag-and-drop (HTML5 DnD or pointer events) updating the order array immutably. Widths are keyed by column key so they survive reordering.

</details>

<details>
<summary><strong>6. Why not `useEffect(() => setPage(1), [search])`?</strong></summary>

It renders once with the stale page (possibly out of range → empty table flash), then again after the effect. Resetting in the event handler batches both updates into one render. Effects are for syncing with external systems, not for deriving state from other state.

</details>

<details>
<summary><strong>7. How would you persist table state in the URL?</strong></summary>

Serialise `{q, dept, sortKey, dir, page, size}` into query params (`useSearchParams`), treat the URL as the source of truth, parse + validate on load (clamp page, ignore unknown sort keys). Makes views shareable and the back button meaningful.

</details>

<details>
<summary><strong>8. Sticky header and sticky first column?</strong></summary>

`thead th { position: sticky; top: 0; z-index: 1 }` works if the scroll container is the nearest scrolling ancestor with a fixed `max-height`. For a sticky first column use `position: sticky; left: 0` on the first `th/td` with a background and higher z-index for the header cell. Note `border-collapse: collapse` can drop borders on sticky cells — use `separate` + `border-spacing: 0`.

</details>

---

## 11. Common Mistakes

- Calling `.sort()` directly on state/props (mutation, and no re-render because the reference is unchanged).
- Sorting numbers as strings (default comparator) or using `a - b` for strings/dates.
- Storing `filteredRows` / `sortedRows` in state and syncing with effects.
- Inline `columns` array or inline `filter` functions creating new dependencies every render → `useMemo` useless.
- Not resetting the page when filters change → "No records" on page 7.
- Using the array index as a row `key` (rows remount/lose state on sort).
- Resizing logic using the previous width + delta each event (drift) instead of `startWidth + total delta`; losing the drag when the cursor leaves the handle (no pointer capture).
- Putting the sort `onClick` on the whole `<th>` (the resize handle then triggers sorts; not keyboard accessible).
- `table-layout: auto` so widths don't respect the resize.
- Forgetting empty state, or computing "Showing x–y" with 0-based indexes.
- Case-sensitive search; building a `RegExp` from user input.

