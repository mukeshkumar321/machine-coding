# Search with Filters (E-commerce) — Machine Coding

> **Difficulty:** Medium–Hard | **Time:** 60 min | **Frequency:** ⭐⭐⭐⭐⭐

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

Build an e-commerce **product listing page** with a text search box and filters (category, brand, price range, minimum rating) plus sorting. **All filter state must live in the URL query string** (`?q=phone&category=Phones&category=Watches&min=100&sort=price-asc`) so that:

- refreshing the page restores the exact view,
- a link can be shared/bookmarked,
- browser Back/Forward move through filter changes.

The text input must be **debounced** so we don't update the URL/results on every keystroke. Implement a small `useQueryParams` hook that mimics React Router's `useSearchParams` (no router library).

---

## 2. Requirements

### Functional

- Text search (case-insensitive, matches product name).
- Multi-select **category** and **brand** checkboxes (OR within a filter, AND across filters).
- **Price range** (min / max) and **min rating** select.
- **Sort**: relevance, price asc/desc, rating.
- "Clear all" resets everything.
- Result count + empty state.
- State ⇄ URL sync both ways (initial load, user changes, Back/Forward).
- Debounced search input (≈ 400 ms).

### Non-functional

- Single source of truth = the URL; no duplicated filter state that can drift.
- Invalid/garbage URL params must not crash (`?min=abc&sort=hack`).
- Typing in the search box must stay responsive and never lose characters.
- Back button shouldn't step through every typed letter.

---

## 3. Clarifying Questions to Ask

- Is filtering done client-side on a full dataset or server-side (then API calls, loading and error states, cancellation)?
- Can I use React Router, or implement the URL sync myself?
- Which filters are multi-select vs single? AND/OR semantics?
- Should each filter change create a new history entry (`pushState`) or replace it?
- Pagination / infinite scroll? Reset to page 1 when filters change?
- Should URL params be human-readable (`category=Phones`) or encoded (`c=3`)?
- Do filter option counts (e.g. "Phones (12)") need to update dynamically?

---

## 4. Approach & Component Design

**Single source of truth:** the URL. React state is *derived* from it on every render; user actions write to the URL.

```
user action ──► setParams() ──► history.pushState/replaceState ──► 'querychange' event
                                                                         │
 popstate (Back/Fwd) ───────────────────────────────────────────────────►│
                                                                         ▼
                              useSyncExternalStore re-reads location.search
                                                                         ▼
                      params ► parseFilters() ► filterProducts() ► render list
```

**Only exception — the text input.** It needs local `text` state so typing is instant; a debounced copy is committed to the URL (`replaceState`, so Back isn't polluted), and URL changes made elsewhere (Back button, "Clear all") flow back into the input.

**Component tree**

```
App
 └─ SearchPage                    useQueryParams → parseFilters → filterProducts (useMemo)
     ├─ SearchBox                 local text + debounce ⇄ URL `q`
     ├─ Filters                   category/brand checkboxes, price, rating, sort, clear
     └─ ProductList               cards, count (aria-live), empty state
```

**URL schema & history policy**

| Param | Example | Parse rule | History |
| --- | --- | --- | --- |
| `q` | `q=phone` | string | `replace` (debounced typing) |
| `category` (repeated) | `category=Phones&category=Watches` | `getAll` | `push` |
| `brand` (repeated) | `brand=Acme` | `getAll` | `push` |
| `min`, `max` | `min=100` | finite number else ignored | `replace` (typing) |
| `rating` | `rating=4` | number else 0 | `push` |
| `sort` | `sort=price-asc` | whitelist else `relevance` | `push` |

Rule of thumb: **discrete choices push; free-text typing replaces.**

---

## 5. Reference Solution

<details>
<summary><strong>Show Full Code</strong></summary>

**`useQueryParams.js`** — a minimal `useSearchParams` clone

```js
import { useCallback, useMemo, useSyncExternalStore } from 'react';

const QUERY_EVENT = 'querychange';

function subscribe(callback) {
  window.addEventListener('popstate', callback); // Back / Forward
  window.addEventListener(QUERY_EVENT, callback); // our own pushState/replaceState (they don't emit events)
  return () => {
    window.removeEventListener('popstate', callback);
    window.removeEventListener(QUERY_EVENT, callback);
  };
}

// A string is a stable, comparable snapshot (no new object each call).
const getSnapshot = () => window.location.search;
const getServerSnapshot = () => '';

export function useQueryParams() {
  const search = useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot);
  const params = useMemo(() => new URLSearchParams(search), [search]);

  // Same contract as React Router: pass init OR an updater `(prev) => next`
  // (the updater must return a new URLSearchParams, not mutate `prev`).
  const setParams = useCallback((update, { replace = false } = {}) => {
    const current = new URLSearchParams(window.location.search); // always read fresh
    const next = typeof update === 'function' ? update(current) : new URLSearchParams(update);

    const qs = next.toString();
    if (qs === current.toString()) return; // no-op: don't spam history

    const url = `${window.location.pathname}${qs ? `?${qs}` : ''}${window.location.hash}`;
    window.history[replace ? 'replaceState' : 'pushState'](null, '', url);
    window.dispatchEvent(new Event(QUERY_EVENT));
  }, []);

  return [params, setParams];
}
```

**`data.js`**

```js
const CATEGORIES = ['Laptops', 'Phones', 'Headphones', 'Cameras', 'Watches'];
const BRANDS = ['Acme', 'Globex', 'Initech', 'Umbrella'];

export const ALL_CATEGORIES = CATEGORIES;
export const ALL_BRANDS = BRANDS;

export const PRODUCTS = Array.from({ length: 48 }, (_, i) => ({
  id: i + 1,
  name: `${BRANDS[i % 4]} ${CATEGORIES[i % 5]} ${100 + i * 7}`,
  category: CATEGORIES[i % 5],
  brand: BRANDS[i % 4],
  price: 49 + ((i * 137) % 951),
  rating: (20 + ((i * 7) % 31)) / 10, // 2.0 – 5.0
}));
```

**`filters.js`** — pure parse / filter / sort (easy to unit-test)

```js
export const SORT_OPTIONS = [
  { value: 'relevance', label: 'Relevance' },
  { value: 'price-asc', label: 'Price: Low to High' },
  { value: 'price-desc', label: 'Price: High to Low' },
  { value: 'rating-desc', label: 'Rating' },
];

const toNumber = (raw) => {
  if (raw === null || raw.trim() === '') return null;
  const n = Number(raw);
  return Number.isFinite(n) ? n : null; // garbage in the URL is ignored, never thrown
};

export function parseFilters(params) {
  const sort = params.get('sort');
  return {
    q: params.get('q') ?? '',
    categories: params.getAll('category'),
    brands: params.getAll('brand'),
    minPrice: toNumber(params.get('min')),
    maxPrice: toNumber(params.get('max')),
    minRating: toNumber(params.get('rating')) ?? 0,
    sort: SORT_OPTIONS.some((o) => o.value === sort) ? sort : 'relevance',
  };
}

export function filterProducts(products, f) {
  const q = f.q.trim().toLowerCase();

  const result = products.filter(
    (p) =>
      (!q || p.name.toLowerCase().includes(q)) &&
      (f.categories.length === 0 || f.categories.includes(p.category)) &&
      (f.brands.length === 0 || f.brands.includes(p.brand)) &&
      (f.minPrice === null || p.price >= f.minPrice) &&
      (f.maxPrice === null || p.price <= f.maxPrice) &&
      p.rating >= f.minRating
  );

  // `result` is already a new array, so sorting in place doesn't mutate the source list.
  switch (f.sort) {
    case 'price-asc':
      return result.sort((a, b) => a.price - b.price);
    case 'price-desc':
      return result.sort((a, b) => b.price - a.price);
    case 'rating-desc':
      return result.sort((a, b) => b.rating - a.rating);
    default:
      return result;
  }
}
```

**`SearchPage.jsx`**

```jsx
import { useCallback, useEffect, useMemo, useRef, useState } from 'react';
import { useQueryParams } from './useQueryParams';
import { ALL_BRANDS, ALL_CATEGORIES, PRODUCTS } from './data';
import { SORT_OPTIONS, filterProducts, parseFilters } from './filters';
import './search.css';

function SearchBox({ urlValue, onCommit, delay = 400 }) {
  const [text, setText] = useState(urlValue);
  const lastCommitted = useRef(urlValue); // the last `q` we know the URL holds

  // text -> URL (debounced). Nothing to commit while text matches the URL,
  // so mounting never rewrites the URL (e.g. drops `page`) and no stray history writes happen.
  useEffect(() => {
    if (text === lastCommitted.current) return;
    const id = setTimeout(() => {
      lastCommitted.current = text;
      onCommit(text);
    }, delay);
    return () => clearTimeout(id); // each keystroke cancels the previous timer
  }, [text, delay, onCommit]);

  // URL -> text, but only for changes we didn't cause ourselves (Back button, Clear all).
  // Comparing with lastCommitted stops the URL echo from overwriting characters typed since.
  useEffect(() => {
    if (urlValue !== lastCommitted.current) {
      lastCommitted.current = urlValue;
      setText(urlValue);
    }
  }, [urlValue]);

  return (
    <input
      type="search"
      className="search-input"
      placeholder="Search products…"
      aria-label="Search products"
      value={text}
      onChange={(e) => setText(e.target.value)}
    />
  );
}

function CheckboxGroup({ legend, options, selected, onToggle }) {
  return (
    <fieldset className="filter">
      <legend>{legend}</legend>
      {options.map((opt) => (
        <label key={opt} className="filter__option">
          <input type="checkbox" checked={selected.includes(opt)} onChange={() => onToggle(opt)} />
          {opt}
        </label>
      ))}
    </fieldset>
  );
}

export default function SearchPage() {
  const [params, setParams] = useQueryParams();
  const filters = useMemo(() => parseFilters(params), [params]);
  const results = useMemo(() => filterProducts(PRODUCTS, filters), [filters]);

  const setParam = useCallback(
    (key, value, options) =>
      setParams((prev) => {
        const next = new URLSearchParams(prev);
        if (value === '' || value === null || value === undefined) next.delete(key);
        else next.set(key, value);
        next.delete('page'); // any filter change goes back to page 1
        return next;
      }, options),
    [setParams]
  );

  const toggleMulti = useCallback(
    (key, value) =>
      setParams((prev) => {
        const next = new URLSearchParams(prev);
        const current = next.getAll(key);
        next.delete(key);
        const updated = current.includes(value)
          ? current.filter((v) => v !== value)
          : [...current, value];
        updated.forEach((v) => next.append(key, v));
        next.delete('page');
        return next;
      }),
    [setParams]
  );

  const commitQuery = useCallback((q) => setParam('q', q, { replace: true }), [setParam]);
  const clearAll = () => setParams(() => new URLSearchParams());

  const hasFilters = params.toString() !== '';
  const priceConflict =
    filters.minPrice !== null && filters.maxPrice !== null && filters.minPrice > filters.maxPrice;

  return (
    <div className="page">
      <header className="page__header">
        <SearchBox urlValue={filters.q} onCommit={commitQuery} />
        <label className="sort">
          Sort by{' '}
          <select value={filters.sort} onChange={(e) => setParam('sort', e.target.value === 'relevance' ? '' : e.target.value)}>
            {SORT_OPTIONS.map((o) => (
              <option key={o.value} value={o.value}>
                {o.label}
              </option>
            ))}
          </select>
        </label>
      </header>

      <div className="page__body">
        <aside className="filters" aria-label="Filters">
          <CheckboxGroup
            legend="Category"
            options={ALL_CATEGORIES}
            selected={filters.categories}
            onToggle={(v) => toggleMulti('category', v)}
          />
          <CheckboxGroup
            legend="Brand"
            options={ALL_BRANDS}
            selected={filters.brands}
            onToggle={(v) => toggleMulti('brand', v)}
          />

          <fieldset className="filter">
            <legend>Price</legend>
            <div className="price">
              <input
                type="number"
                min="0"
                inputMode="numeric"
                placeholder="Min"
                aria-label="Minimum price"
                value={params.get('min') ?? ''}
                onChange={(e) => setParam('min', e.target.value, { replace: true })}
              />
              <span aria-hidden="true">–</span>
              <input
                type="number"
                min="0"
                inputMode="numeric"
                placeholder="Max"
                aria-label="Maximum price"
                value={params.get('max') ?? ''}
                onChange={(e) => setParam('max', e.target.value, { replace: true })}
              />
            </div>
            {priceConflict && <p className="hint">Min price is greater than max price.</p>}
          </fieldset>

          <label className="filter">
            <span className="filter__label">Minimum rating</span>
            <select
              value={String(filters.minRating)}
              onChange={(e) => setParam('rating', e.target.value === '0' ? '' : e.target.value)}
            >
              <option value="0">Any</option>
              <option value="3">3★ & up</option>
              <option value="4">4★ & up</option>
            </select>
          </label>

          <button type="button" onClick={clearAll} disabled={!hasFilters}>
            Clear all
          </button>
        </aside>

        <main>
          <p className="count" aria-live="polite">
            {results.length} result{results.length === 1 ? '' : 's'}
          </p>
          {results.length === 0 ? (
            <p className="empty">
              No products match your filters. <button type="button" className="link" onClick={clearAll}>Clear all</button>
            </p>
          ) : (
            <ul className="grid">
              {results.map((p) => (
                <li key={p.id} className="card">
                  <h3>{p.name}</h3>
                  <p>
                    {p.category} · {p.brand}
                  </p>
                  <p className="card__meta">
                    <strong>${p.price}</strong>
                    <span role="img" aria-label={`Rated ${p.rating} out of 5`}>★ {p.rating.toFixed(1)}</span>
                  </p>
                </li>
              ))}
            </ul>
          )}
        </main>
      </div>
    </div>
  );
}
```

**`App.jsx`**

```jsx
import SearchPage from './SearchPage';

export default function App() {
  return <SearchPage />;
}
```

**`search.css`**

```css
* { box-sizing: border-box; }
body { margin: 0; font-family: system-ui, sans-serif; color: #111; }

.page { max-width: 1100px; margin: 0 auto; padding: 16px; }
.page__header { display: flex; gap: 16px; align-items: center; margin-bottom: 16px; }
.search-input { flex: 1; padding: 10px 12px; font-size: 16px; border: 1px solid #cbd5e1; border-radius: 8px; }

.page__body { display: grid; grid-template-columns: 220px 1fr; gap: 24px; }
@media (max-width: 720px) { .page__body { grid-template-columns: 1fr; } }

.filters { display: flex; flex-direction: column; gap: 16px; }
.filter { border: 0; padding: 0; margin: 0; display: flex; flex-direction: column; gap: 6px; }
.filter legend, .filter__label { font-weight: 600; margin-bottom: 4px; padding: 0; }
.filter__option { display: flex; gap: 8px; align-items: center; cursor: pointer; }
.price { display: flex; gap: 6px; align-items: center; }
.price input { width: 100%; min-width: 0; padding: 6px; }
.hint { color: #b45309; font-size: 13px; margin: 0; }

.count { margin: 0 0 12px; color: #475569; }
.empty { padding: 32px 0; color: #475569; }

.grid { list-style: none; margin: 0; padding: 0; display: grid; grid-template-columns: repeat(auto-fill, minmax(200px, 1fr)); gap: 12px; }
.card { border: 1px solid #e2e8f0; border-radius: 8px; padding: 12px; }
.card h3 { margin: 0 0 4px; font-size: 15px; }
.card p { margin: 0 0 6px; font-size: 14px; color: #475569; }
.card__meta { display: flex; justify-content: space-between; color: #111 !important; }

.link { background: none; border: 0; color: #2563eb; text-decoration: underline; cursor: pointer; padding: 0; }
```

</details>

---

## 6. How It Works

- **URL as the store.** `useQueryParams` wraps `window.location.search` with `useSyncExternalStore`. `pushState`/`replaceState` **don't** fire any event, so `setParams` dispatches a custom `querychange` event; `popstate` covers Back/Forward. All components calling the hook re-render from the same source — no context needed.
- **String snapshot.** `getSnapshot` returns `location.search` (a string), so React can compare with `Object.is`; returning a new `URLSearchParams` each time would loop forever. `useMemo` builds the params object from the string.
- **Same API as React Router.** `setParams(updater, { replace })` with `updater(prev) → new URLSearchParams`. Swapping in `useSearchParams` later is a one-line import change. The hook reads `window.location.search` fresh inside `setParams`, so it never uses a stale `params` closure — important because the debounce fires 400 ms later.
- **Derive, don't copy.** `filters = parseFilters(params)` and `results = filterProducts(...)` are `useMemo` derivations; there is no `useState` mirroring the URL (the classic drift bug). `parseFilters` validates everything (finite numbers, sort whitelist), so `?min=abc&sort=hack` degrades gracefully.
- **Repeated params for multi-select** (`category=A&category=B`) via `getAll` / `append` — readable, and what `URLSearchParams` is designed for.
- **Debounce + URL echo problem.** The input is controlled by local `text` (instant typing). A timer effect in `SearchBox` commits `text` to the URL 400 ms after the last keystroke, via `replace` so Back doesn't walk through every query. `lastCommitted` remembers the `q` the URL holds: when the URL echoes our own commit back, `urlValue !== lastCommitted.current` is *false*, so we don't overwrite whatever the user typed in the meantime. When Back/Clear all changes `q`, the comparison is true → `setText(urlValue)` syncs the input. Because the timer only starts when `text` differs from `lastCommitted`, mounting never touches the URL.
- **Filter changes `push`**, so Back undoes a filter; **typing replaces**, so Back doesn't undo letters.
- **No-op guard** (`qs === current.toString()`) avoids duplicate history entries when a change produces the same URL (e.g. re-selecting the current sort).
- **Page reset.** Every filter change deletes `page` — the standard rule to avoid landing on an empty page 7.

---

## 7. Edge Cases

- Refresh / deep link with filters → `useSyncExternalStore` reads the URL on first render; `SearchBox` initialises `text` from it.
- Garbage params (`min=abc`, `rating=NaN`, `sort=foo`) → ignored/defaulted.
- Min > max → zero results + hint (don't silently swap).
- Same filter clicked twice quickly → `toggleMulti` reads the **fresh** URL each time, not a closed-over array.
- Back button after typing → input updates to the previous `q`; pending typing is discarded by design.
- "Clear all" while the debounce timer is pending → text is reset via URL echo, which re-runs the effect and cancels the pending timer (text now equals the URL, so nothing is committed).
- Special characters in `q` (`&`, `#`, `+`, unicode) → `URLSearchParams` encodes/decodes them correctly; never hand-build query strings with `+`/`&`.
- Very long query strings → cap `q` length or ignore unknown params.
- Empty `q` → param removed (not `q=`), so URLs stay clean.
- Unknown extra params (e.g. `utm_source`) are preserved because we copy `prev`.
- Hash fragments are preserved when rewriting the URL.
- SSR → `getServerSnapshot` returns `''`.
- Server-side filtering variant: add `AbortController` per request so slow, old responses can't overwrite newer results; show loading/error states.

---

## 8. Accessibility

- Search input has an `aria-label`; `type="search"` gives a clear button and the right semantics.
- Filter groups use `<fieldset>` + `<legend>`; each checkbox is wrapped by a `<label>`.
- Result count in `aria-live="polite"` so screen-reader users learn the effect of changing a filter without moving focus.
- Number inputs have `aria-label`s ("Minimum price"); the error hint is real text next to the field.
- Everything is native controls → keyboard works for free; don't build custom dropdowns for this exercise.
- Empty state offers an action ("Clear all"), not just a message.
- Focus isn't stolen on filter change (list updates in place).

---

## 9. Performance Considerations

- **Debounce** the free-text input so we don't push URL changes + recompute on every keystroke (but keep the input itself instant via local state).
- `useMemo` for `filters` and `results` keyed on `params` — recomputation only when the URL changes.
- Client-side filtering of thousands of items is fine; for tens of thousands, move to the server or a web worker, and virtualize the grid.
- Keep list items keyed by `id`; wrap `ProductCard` in `React.memo` if cards become heavy.
- For server-side search: debounce + `AbortController` (cancel stale requests), cache by query string, `keepPreviousData` to avoid flashing an empty state.
- `useDeferredValue`/`startTransition` can keep typing responsive when rendering the list is expensive.

---

## 10. Follow-up Questions

<details>
<summary><strong>1. Why `useSyncExternalStore` instead of `useState` + `useEffect` listening to `popstate`?</strong></summary>

It's the sanctioned way to subscribe to external mutable sources: it avoids tearing in concurrent rendering, reads the current value synchronously on first render (no flash of default filters), and handles subscription cleanup. The `useEffect` version renders once with stale data before the effect runs.

</details>

<details>
<summary><strong>2. How would you add pagination or infinite scroll?</strong></summary>

Add `page` to the URL; `setParam` already resets it on filter changes. For infinite scroll keep `page` out of the URL (or use a cursor) and accumulate results in state/React Query's `useInfiniteQuery`; make sure the "reset on filter change" rule clears accumulated pages.

</details>

<details>
<summary><strong>3. How would you move this to server-side filtering?</strong></summary>

Replace `filterProducts` with a fetch keyed by `params.toString()` (React Query: `queryKey: ['products', qs]`). Add loading/error UI, `AbortController` (or React Query's built-in cancellation) so out-of-order responses can't win, and prefetch/keep previous data for smooth transitions. The URL ⇄ state logic is unchanged.

</details>

<details>
<summary><strong>4. Why `replaceState` for typing and `pushState` for checkboxes?</strong></summary>

History entries should represent *meaningful* navigation steps. A checkbox toggle is a discrete decision users expect Back to undo; 12 intermediate strings from typing `"headphones"` are noise (and debouncing only reduces, not eliminates them).

</details>

<details>
<summary><strong>5. What's the bug if `getSnapshot` returns `new URLSearchParams(location.search)`?</strong></summary>

`useSyncExternalStore` compares snapshots with `Object.is`; a fresh object each call looks like "store changed" every render → infinite re-render loop (React warns "The result of getSnapshot should be cached"). Return a primitive (string) or a cached object.

</details>

<details>
<summary><strong>6. How would you show per-option counts ("Phones (12)") that update with other filters?</strong></summary>

For each facet, compute counts over products filtered by *all other* filters but not that facet itself (so users can still see alternatives). Do it in one pass with a `Map` per facet inside `useMemo`; on the server this is a facet aggregation.

</details>

<details>
<summary><strong>7. How do you migrate to React Router?</strong></summary>

Replace the import with `const [params, setParams] = useSearchParams()`. Because our `setParams(prev => next, { replace })` signature mirrors React Router v6.4+'s, nothing else changes. Remove the custom event and `popstate` listeners — the router owns history.

</details>

<details>
<summary><strong>8. Debounce vs throttle — and could you use `useDeferredValue` instead?</strong></summary>

Debounce waits for a pause (best for "search as you type" → fewer network calls). Throttle limits rate during continuous input. `useDeferredValue` doesn't reduce *calls*; it lowers *render priority*, so it's good for expensive client-side filtering but doesn't replace debouncing network requests or URL updates.

</details>

---

## 11. Common Mistakes

- Duplicating filters in `useState` **and** the URL → they drift (Back button shows stale UI).
- Writing the URL on every keystroke → history pollution + jank; or debouncing the input's `value` itself (typing feels laggy).
- The URL-echo bug: syncing `urlValue → text` unconditionally, so the user's newer characters get overwritten.
- Using `useEffect` to read the URL after the first render → flash of unfiltered results.
- Forgetting `popstate`, or forgetting that `pushState` doesn't trigger it → UI doesn't update.
- Building query strings by hand (`'?q=' + q`) → breaks on `&`, `#`, spaces.
- Not validating params (`Number('abc')` → `NaN` → everything filtered out / crashes).
- Mutating the source array with `.sort()` on `PRODUCTS`.
- Capturing a stale `params` inside the debounced callback; always read the fresh location inside `setParams`.
- Not resetting `page` when filters change.
- Using the array index as `key` for results.
- Treating every filter change as `pushState` (including typing) or every change as `replaceState` (Back does nothing).

---

## 12. Interview Tips & Time Plan

Open with the principle: *"The URL is the single source of truth; the UI state is derived."* Then build in layers: static list → pure `filterProducts` → URL hook → checkboxes → debounce → polish. The debounce + URL echo subtlety is where strong candidates stand out — mention it even if you run out of time.

| Minute | Activity |
| --- | --- |
| 0–5 | Clarify (client vs server filtering, router allowed?, history policy) |
| 5–10 | URL schema, data flow diagram, component tree |
| 10–18 | Mock data + pure `parseFilters` / `filterProducts` |
| 18–30 | `useQueryParams` (`useSyncExternalStore`, push/replace, popstate) |
| 30–42 | Filters UI: checkboxes, price, rating, sort, clear all |
| 42–50 | Debounced `SearchBox` with URL ⇄ text sync |
| 50–55 | Empty state, count, a11y |
| 55–60 | Edge cases, follow-ups (server-side, pagination) |

---

## 13. Practice Checklist

- [ ] Pure `filterProducts` (AND across filters, OR within) with unit tests
- [ ] `useQueryParams` using `useSyncExternalStore` + `popstate` + custom event
- [ ] Explain why `pushState` needs a manual event
- [ ] Multi-select params with `getAll` / `append`
- [ ] Push vs replace history policy
- [ ] Debounced input that doesn't lose characters (URL echo handled)
- [ ] Validate/ignore invalid params
- [ ] Clear all + empty state
- [ ] Reset `page` on filter change
- [ ] Server-side variant with `AbortController`
- [ ] Facet counts per option
- [ ] Replace the custom hook with React Router's `useSearchParams`
