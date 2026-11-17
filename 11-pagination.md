# Pagination — Machine Coding

> **Difficulty:** Medium · **Time:** 45 min · **Asked at / Frequency:** Commonly reported at product companies (Flipkart, Swiggy, Razorpay, Atlassian) — High · **Tier:** 2

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

Build a paginated list. Show a slice of a large dataset, with:

- Numbered page buttons with **ellipsis** (`1 … 4 5 6 … 20`)
- Prev / Next buttons
- A **page-size selector** (10 / 20 / 50)
- A reusable `usePagination` hook that owns all the math

---

## 2. Requirements

### Functional

- Render only the current page's items.
- Clicking a page number jumps to it; Prev/Next step by one and disable at the edges.
- Highlight the current page.
- Always show first and last page; collapse the rest with `…` when there are many pages.
- Changing page size recalculates total pages and keeps the user near the same items.
- Show "Showing 21–30 of 237".

### Non-functional

- Pagination logic separated from UI (hook + pure `getPageRange` function).
- No out-of-range page ever (clamp).
- Keyboard accessible, `aria-current="page"`.
- No unnecessary recomputation (`useMemo` for the page list).

---

## 3. Clarifying Questions to Ask

- Client-side (all data in memory) or server-side (API per page)? *(Assume client-side; design the hook so it works for both.)*
- How many page buttons visible at once? Siblings count around the current page?
- Should page / page size be in the URL (`?page=3&size=20`)?
- What happens to the current page when page size changes?
- Do we need "Go to page" input / First / Last buttons?
- Empty state when there are 0 items?

---

## 4. Approach & Component Design

**State** (only two pieces — everything else is derived):

| State / derived | Type | Notes |
|---|---|---|
| `page` | state | 1-based |
| `pageSize` | state | 10 / 20 / 50 |
| `totalPages` | derived | `max(1, ceil(totalItems / pageSize))` |
| `currentPage` | derived | `page` clamped into `[1, totalPages]` |
| `startIndex / endIndex` | derived | for `slice` and the "Showing x–y" label |
| `pages` | derived (memo) | array like `[1,'dots-left',4,5,6,'dots-right',20]` |

**Ellipsis algorithm** (`siblings = 1`): the slot count is fixed at `siblings*2 + 5` (first, last, current, 2 siblings, 2 dots) so the control never changes width while paging.

```
total <= 7           ->  1 2 3 4 5 6 7
current near start   ->  1 2 3 4 5 … 20
current in middle    ->  1 … 9 10 11 … 20
current near end     ->  1 … 16 17 18 19 20
```

**Component tree**

```
App
 ├─ usePagination({ totalItems, initialPageSize })
 ├─ <ul>  items.slice(startIndex, endIndex)
 └─ <Pagination
        pages, currentPage, pageSize, onPageChange, onPageSizeChange, ... />
```

---

## 5. Reference Solution

<details>
<summary><strong>Show Full Code (usePagination.js, Pagination.jsx, App.jsx, styles.css)</strong></summary>

```jsx
// usePagination.js
import { useCallback, useMemo, useState } from 'react';

const range = (from, to) => Array.from({ length: to - from + 1 }, (_, i) => from + i);

export function getPageRange(current, total, siblings = 1) {
  const totalSlots = siblings * 2 + 5;
  if (total <= totalSlots) return range(1, total);

  const left = Math.max(current - siblings, 1);
  const right = Math.min(current + siblings, total);
  const showLeftDots = left > 2;
  const showRightDots = right < total - 1;
  const edgeCount = 3 + siblings * 2;

  if (!showLeftDots && showRightDots) {
    return [...range(1, edgeCount), 'dots-right', total];
  }
  if (showLeftDots && !showRightDots) {
    return [1, 'dots-left', ...range(total - edgeCount + 1, total)];
  }
  return [1, 'dots-left', ...range(left, right), 'dots-right', total];
}

export function usePagination({
  totalItems,
  initialPageSize = 10,
  initialPage = 1,
  siblings = 1,
}) {
  const [page, setPage] = useState(initialPage);
  const [pageSize, setPageSizeState] = useState(initialPageSize);

  const totalPages = Math.max(1, Math.ceil(totalItems / pageSize));
  const currentPage = Math.min(Math.max(page, 1), totalPages);
  const startIndex = (currentPage - 1) * pageSize;
  const endIndex = Math.min(startIndex + pageSize, totalItems);

  const goTo = useCallback(
    (p) => setPage(Math.min(Math.max(p, 1), totalPages)),
    [totalPages]
  );
  const next = useCallback(
    () => setPage((p) => Math.min(p + 1, totalPages)),
    [totalPages]
  );
  const prev = useCallback(() => setPage((p) => Math.max(p - 1, 1)), []);

  // keep the first visible item visible after the size changes
  const setPageSize = useCallback(
    (size) => {
      setPageSizeState(size);
      setPage(Math.floor(startIndex / size) + 1);
    },
    [startIndex]
  );

  const pages = useMemo(
    () => getPageRange(currentPage, totalPages, siblings),
    [currentPage, totalPages, siblings]
  );

  return {
    currentPage, pageSize, totalPages, startIndex, endIndex, pages,
    goTo, next, prev, setPageSize,
    hasPrev: currentPage > 1,
    hasNext: currentPage < totalPages,
  };
}
```

```jsx
// Pagination.jsx
import { memo } from 'react';

const PAGE_SIZES = [10, 20, 50];

function Pagination({
  pages, currentPage, hasPrev, hasNext, pageSize,
  onPageChange, onPrev, onNext, onPageSizeChange,
}) {
  return (
    <nav className="pagination" aria-label="Pagination">
      <button onClick={onPrev} disabled={!hasPrev} aria-label="Previous page">
        ‹ Prev
      </button>

      <ul className="pages">
        {pages.map((p) =>
          typeof p === 'number' ? (
            <li key={p}>
              <button
                className={p === currentPage ? 'active' : ''}
                aria-current={p === currentPage ? 'page' : undefined}
                aria-label={`Page ${p}`}
                onClick={() => onPageChange(p)}
              >
                {p}
              </button>
            </li>
          ) : (
            <li key={p} className="dots" aria-hidden="true">…</li>
          )
        )}
      </ul>

      <button onClick={onNext} disabled={!hasNext} aria-label="Next page">
        Next ›
      </button>

      <label className="size">
        Rows per page
        <select value={pageSize} onChange={(e) => onPageSizeChange(Number(e.target.value))}>
          {PAGE_SIZES.map((s) => (
            <option key={s} value={s}>{s}</option>
          ))}
        </select>
      </label>
    </nav>
  );
}

export default memo(Pagination);
```

```jsx
// App.jsx
import { usePagination } from './usePagination';
import Pagination from './Pagination';
import './styles.css';

const ITEMS = Array.from({ length: 237 }, (_, i) => ({ id: i + 1, title: `Item #${i + 1}` }));

export default function App() {
  const {
    currentPage, pageSize, startIndex, endIndex, pages,
    goTo, next, prev, setPageSize, hasPrev, hasNext,
  } = usePagination({ totalItems: ITEMS.length, initialPageSize: 10 });

  const visible = ITEMS.slice(startIndex, endIndex);

  return (
    <main className="app">
      <h1>Pagination</h1>

      <p aria-live="polite">
        Showing {ITEMS.length === 0 ? 0 : startIndex + 1}–{endIndex} of {ITEMS.length}
      </p>

      <ul className="list">
        {visible.map((item) => (
          <li key={item.id}>{item.title}</li>
        ))}
      </ul>

      <Pagination
        pages={pages}
        currentPage={currentPage}
        pageSize={pageSize}
        hasPrev={hasPrev}
        hasNext={hasNext}
        onPageChange={goTo}
        onPrev={prev}
        onNext={next}
        onPageSizeChange={setPageSize}
      />
    </main>
  );
}
```

```css
/* styles.css */
.app { max-width: 640px; margin: 24px auto; font-family: system-ui, sans-serif; }
.list { list-style: none; padding: 0; margin: 12px 0; }
.list li { padding: 8px 12px; border-bottom: 1px solid #eee; }

.pagination { display: flex; align-items: center; gap: 8px; flex-wrap: wrap; }
.pages { display: flex; gap: 4px; list-style: none; margin: 0; padding: 0; }
.pagination button {
  min-width: 36px; height: 36px; padding: 0 8px;
  border: 1px solid #ccc; background: #fff; border-radius: 6px; cursor: pointer;
}
.pagination button:hover:not(:disabled) { background: #f3f3f3; }
.pagination button:disabled { opacity: 0.4; cursor: not-allowed; }
.pagination button.active { background: #2563eb; border-color: #2563eb; color: #fff; }
.pagination button:focus-visible { outline: 2px solid #2563eb; outline-offset: 2px; }
.dots { min-width: 24px; text-align: center; line-height: 36px; }
.size { margin-left: auto; display: flex; gap: 6px; align-items: center; }
```

</details>

---

## 6. How It Works

- **Derive, don't store.** Only `page` and `pageSize` are state. `totalPages`, `startIndex`, `endIndex`, `pages` are computed each render, so they can never go out of sync.
- **`getPageRange` is a pure function.** Easy to unit test (`getPageRange(1,10)`, `(5,10)`, `(10,10)`). The fixed slot count (`siblings*2+5`) keeps width stable.
- **Clamp on read** (`currentPage`) *and* on write (`goTo`). If the data shrinks (e.g. a filter is applied) the user is never stranded on page 9 of 3.
- **Page-size change** keeps the first visible item on screen: `floor(startIndex / newSize) + 1`. Resetting to page 1 is also acceptable — say which you chose.
- **Functional updates** (`setPage(p => …)`) in `next`/`prev` avoid stale closures and keep callbacks stable.
- **Ellipsis items** use string keys (`dots-left` / `dots-right`) so React keys stay unique and stable.

---

## 7. Edge Cases

- `totalItems = 0` → `totalPages = 1`, show "0 of 0" and an empty state; Prev/Next disabled.
- `totalPages <= slots` → no ellipsis at all.
- Current page = 1, 2, 3 or last-2 … last: the "near edge" branches (otherwise a lone `…` hides a single page).
- Page size larger than total items.
- Data shrinks while on a high page (clamp).
- Rapid double-click on Next past the last page (functional update + `Math.min`).
- Server-side: out-of-order responses (see follow-ups).

---

## 8. Accessibility

- Wrap in `<nav aria-label="Pagination">`.
- `aria-current="page"` on the active button; descriptive `aria-label="Page 5"`.
- Use real `<button>`s (Enter/Space + focus for free); `disabled` on Prev/Next at edges.
- Ellipsis is decorative → `aria-hidden`.
- `aria-live="polite"` on the "Showing x–y" text so screen-reader users hear the change.
- Visible `:focus-visible` outline; don't rely on colour alone for the active page (also bold / border).

---

## 9. Performance Considerations

- Render only `pageSize` rows — that's the point of pagination; for 100k rows in memory, slice is O(pageSize).
- `useMemo` the page list; `React.memo` on `Pagination` with stable callbacks (`useCallback`) so list re-renders don't re-render the control.
- Server-side: **debounce/cancel** requests (`AbortController`), cache pages by key, **prefetch** the next page.
- Keep `key={item.id}`, never the array index, so rows don't remount when page changes.
- For infinite datasets prefer cursor-based pagination (offset gets slow and inconsistent).

---

## 10. Follow-up Questions

<details>
<summary><strong>1. How would you convert this to server-side pagination?</strong></summary>

Hook receives `totalItems` from the API response; a `useEffect` (or React Query) fetches on `[currentPage, pageSize]`. Use an `AbortController` in cleanup so stale responses can't overwrite newer ones, show a loading state, and keep the previous page's data visible while loading (`keepPreviousData`).

</details>

<details>
<summary><strong>2. How do you sync page and size with the URL?</strong></summary>

Read initial values from `URLSearchParams` (`useSearchParams` in React Router) and write on change with `setSearchParams`. Make the URL the single source of truth, so refresh/back/share work. Validate/clamp the parsed values.

</details>

<details>
<summary><strong>3. Offset vs cursor pagination?</strong></summary>

Offset (`page`, `limit`) allows random page jumps but is slow on big tables and shows duplicates/skips when rows are inserted. Cursor (`after=<id>`) is stable and fast but only supports next/prev — better for feeds.

</details>

<details>
<summary><strong>4. Pagination vs infinite scroll vs "Load more"?</strong></summary>

Pagination: goal-oriented, shareable position, footer reachable (tables, search results). Infinite scroll: browsing feeds, but loses position, hurts a11y and the footer. "Load more": middle ground, keyboard friendly.

</details>

<details>
<summary><strong>5. How would you add a "Go to page" input?</strong></summary>

Controlled input with local string state; on Enter/blur parse `Number`, ignore `NaN`, call `goTo` (which clamps). Keep the local value in sync with `currentPage` when it changes elsewhere.

</details>

<details>
<summary><strong>6. How do you reset to page 1 when a filter/search changes?</strong></summary>

Call `goTo(1)` in the same handler that changes the filter (preferred — no extra render). Avoid `useEffect(() => setPage(1), [filter])`, which renders once with an invalid page. Clamping in the hook is the safety net.

</details>

<details>
<summary><strong>7. Support a different number of sibling pages / mobile layout?</strong></summary>

`siblings` is a hook param; on small screens pass `0` (`1 … 5 … 20`) via a `matchMedia` hook, or hide numbers and show "Page 5 of 20".

</details>

<details>
<summary><strong>8. How would you unit test it?</strong></summary>

`getPageRange` table-driven tests for every region (start/middle/end/small totals). `renderHook` for `usePagination`: clamping, `setPageSize` behaviour, next/prev at edges.

</details>

---

## 11. Common Mistakes

- Storing `totalPages` / `visibleItems` in state (goes stale).
- Not clamping the page → blank list after filtering.
- Ellipsis shown for a single hidden page (`1 … 3` where `…` is just 2).
- Page buttons jumping in width because the slot count isn't constant.
- `key={index}` on page buttons or list rows.
- Mutating with `.sort()` on the source array before slicing.
- Forgetting 1-based vs 0-based indexing (`slice(start, end)` is 0-based and end-exclusive).
- Prev/Next not disabled at edges.

---

## 12. Interview Tips & Time Plan

State assumptions out loud (client-side, siblings = 1). Build the **pure range function first** — it's the part interviewers probe.

| Minutes | Task |
|---|---|
| 0–5 | Clarify, sketch state (`page`, `pageSize`) + derived values |
| 5–10 | Render sliced list + Prev/Next working |
| 10–22 | `getPageRange` with ellipsis; walk through 3 examples by hand |
| 22–30 | Page-size selector, clamping, "Showing x–y" |
| 30–38 | Extract `usePagination`, memoize, polish CSS |
| 38–45 | a11y (`nav`, `aria-current`), edge cases, discuss server-side |

---

## 13. Practice Checklist

- [ ] Implement `getPageRange` from memory and test 5 inputs
- [ ] Prev/Next disable correctly at both edges
- [ ] Page size change keeps position (or resets) deliberately
- [ ] Clamp when data shrinks
- [ ] `aria-current`, `nav` landmark, keyboard usable
- [ ] Extract everything into `usePagination`
- [ ] Explain server-side version with `AbortController`
- [ ] Sync page/size with URL query params
- [ ] Re-do in 30 minutes without looking
