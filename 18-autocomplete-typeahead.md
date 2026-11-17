# Autocomplete / Typeahead — Machine Coding

> **Difficulty:** Medium–Hard | **Time:** 45–60 min | **Asked at / Frequency:** Very common at product companies (Google, Amazon, Flipkart, Uber-style UI rounds) — ⭐⭐⭐⭐⭐ | **Tier:** 1 (must know)

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

Build an **Autocomplete (typeahead)** search box. As the user types, fetch suggestions from an API (**debounced**), show them in a dropdown, and let the user pick one with the **mouse or keyboard**. Handle **loading / empty / error** states, avoid **race conditions** (out-of-order responses) using `AbortController`, and **cache** previous queries.

---

## 2. Requirements

### Functional

- Controlled text input; suggestions appear after a minimum number of characters.
- Debounce API calls (~300 ms).
- Keyboard: `↓`/`↑` move highlight (wrap), `Enter` selects, `Esc` closes, mouse hover/click works.
- Loading, "No results", and error states.
- Highlight the matching part of each suggestion.
- Selecting an item fills the input, closes the list, and fires `onSelect`.

### Non-functional

- **No stale results**: slower earlier request must never overwrite a newer one.
- Cancel in-flight requests on new input/unmount.
- Cache by query to avoid refetching.
- Accessible combobox (ARIA 1.2 pattern).
- Search function injected (`searchFn`) → testable, API-agnostic.

---

## 3. Clarifying Questions to Ask

- Is the data local or remote? Is the API paginated? What's the max number of suggestions?
- Debounce delay and minimum characters?
- Free-text allowed (search box) or must pick from list (select/combobox)?
- Should selecting trigger a search or just fill the input?
- Recent searches / empty-state suggestions when focused with no text?
- Is the API `AbortSignal`-friendly (fetch)? What error behaviour do we want (retry)?
- Do we need multi-select / chips?

---

## 4. Approach & Component Design

**State shape**

```js
query:        string                       // what the user typed (immediate)
open:         boolean                      // dropdown visibility
activeIndex:  number                       // -1 = none highlighted
// inside useSearch:
debounced:    string                       // query after delay
{ status: 'idle'|'loading'|'success'|'error', results: [], error }
cache:        useRef(Map<queryLower, results>)
```

**Data flow**

```
keystroke → query (immediate, input stays responsive)
              └─ useDebounce(300ms) → debounced
                     └─ effect: cache hit? → show
                                else: new AbortController → searchFn(q, signal)
                                      cleanup: controller.abort()   ← kills stale request
                                      then(): ignore if aborted
```

```
Autocomplete
 ├── <input role="combobox" aria-activedescendant ...>
 ├── <ul role="listbox"> <li role="option"> × N </ul>
 ├── status rows (loading / empty / error)
 └── visually-hidden live region ("5 results available")
hooks: useDebounce, useSearch (debounce + abort + cache)
```

| Problem | Solution |
| --- | --- |
| Too many requests | Debounce |
| Out-of-order responses | Abort previous request in effect cleanup + `aborted` check |
| Repeated queries | `Map` cache in `useRef` |
| Blur before click on option | `onMouseDown={e => e.preventDefault()}` on the list |
| Stale highlight | Reset `activeIndex` when results change |

---

## 5. Reference Solution

<details>
<summary><strong>Show Full Code</strong></summary>

**`api.js`** (mock that honours `AbortSignal` and has random latency to expose races)

```jsx
const COUNTRIES = [
  'Argentina', 'Australia', 'Austria', 'Bangladesh', 'Belgium', 'Brazil', 'Canada', 'Chile',
  'China', 'Colombia', 'Denmark', 'Egypt', 'Finland', 'France', 'Germany', 'Ghana', 'Greece',
  'India', 'Indonesia', 'Ireland', 'Israel', 'Italy', 'Japan', 'Kenya', 'Malaysia', 'Mexico',
  'Nepal', 'Netherlands', 'New Zealand', 'Nigeria', 'Norway', 'Pakistan', 'Peru', 'Philippines',
  'Poland', 'Portugal', 'Russia', 'Singapore', 'South Africa', 'Spain', 'Sweden', 'Switzerland',
  'Thailand', 'Turkey', 'Ukraine', 'United Kingdom', 'United States', 'Vietnam',
];

const abortError = () => new DOMException('Aborted', 'AbortError');

export function searchCountries(query, signal) {
  return new Promise((resolve, reject) => {
    if (signal?.aborted) return reject(abortError());

    const timer = setTimeout(() => {
      const q = query.toLowerCase();
      const results = COUNTRIES.filter((c) => c.toLowerCase().includes(q))
        .slice(0, 8)
        .map((name) => ({ id: name, label: name }));
      resolve(results);
    }, 200 + Math.random() * 800); // random latency => responses can arrive out of order

    signal?.addEventListener(
      'abort',
      () => {
        clearTimeout(timer);
        reject(abortError());
      },
      { once: true }
    );
  });
}

// Real-API version: fetch rejects with AbortError automatically when aborted
export async function searchProducts(query, signal) {
  const res = await fetch(
    `https://dummyjson.com/products/search?q=${encodeURIComponent(query)}&limit=8`,
    { signal }
  );
  if (!res.ok) throw new Error(`Request failed: ${res.status}`);
  const data = await res.json();
  return data.products.map((p) => ({ id: p.id, label: p.title }));
}
```

**`useDebounce.js`**

```jsx
import { useEffect, useState } from 'react';

export function useDebounce(value, delay) {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id);
  }, [value, delay]);

  return debounced;
}
```

**`useSearch.js`**

```jsx
import { useEffect, useRef, useState } from 'react';
import { useDebounce } from './useDebounce';

const IDLE = { status: 'idle', results: [], error: null };

// `searchFn` must be a stable reference (module-level function or useCallback)
export function useSearch(query, { searchFn, delay = 300, minChars = 2, enabled = true }) {
  const trimmed = query.trim();
  const debounced = useDebounce(trimmed, delay);
  const cacheRef = useRef(new Map());
  const [state, setState] = useState(IDLE);

  useEffect(() => {
    if (!enabled || debounced.length < minChars) {
      setState(IDLE);
      return;
    }

    const key = debounced.toLowerCase();
    if (cacheRef.current.has(key)) {
      setState({ status: 'success', results: cacheRef.current.get(key), error: null });
      return;
    }

    const controller = new AbortController();
    setState((prev) => ({ status: 'loading', results: prev.results, error: null })); // keep old results visible

    searchFn(debounced, controller.signal)
      .then((results) => {
        if (controller.signal.aborted) return; // even if searchFn ignores abort
        cacheRef.current.set(key, results);
        setState({ status: 'success', results, error: null });
      })
      .catch((error) => {
        if (error.name === 'AbortError' || controller.signal.aborted) return;
        setState({ status: 'error', results: [], error });
      });

    return () => controller.abort(); // new query / unmount / disabled => cancel this request
  }, [debounced, minChars, enabled, searchFn]);

  // typed something new but debounce hasn't fired yet
  const isPending = enabled && trimmed.length >= minChars && trimmed !== debounced;

  return { ...state, loading: state.status === 'loading' || isPending };
}
```

**`Autocomplete.jsx`**

```jsx
import { useEffect, useId, useState } from 'react';
import { useSearch } from './useSearch';
import './autocomplete.css';

function Highlight({ text, query }) {
  const q = query.trim();
  const i = q ? text.toLowerCase().indexOf(q.toLowerCase()) : -1;
  if (i < 0) return text;
  return (
    <>
      {text.slice(0, i)}
      <mark>{text.slice(i, i + q.length)}</mark>
      {text.slice(i + q.length)}
    </>
  );
}

export default function Autocomplete({
  searchFn,
  onSelect,
  placeholder = 'Search…',
  minChars = 2,
  delay = 300,
  label = 'Search',
}) {
  const [query, setQuery] = useState('');
  const [open, setOpen] = useState(false);
  const [activeIndex, setActiveIndex] = useState(-1);
  const baseId = useId();
  const listboxId = `${baseId}-listbox`;
  const optionId = (i) => `${baseId}-option-${i}`;

  const { results, status, loading } = useSearch(query, {
    searchFn,
    delay,
    minChars,
    enabled: open,
  });

  const showPanel = open && query.trim().length >= minChars;
  const showOptions = showPanel && results.length > 0;
  const showEmpty = showPanel && !loading && status === 'success' && results.length === 0;
  const showError = showPanel && !loading && status === 'error';

  useEffect(() => setActiveIndex(-1), [results]);

  useEffect(() => {
    if (activeIndex >= 0) {
      document.getElementById(optionId(activeIndex))?.scrollIntoView({ block: 'nearest' });
    }
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [activeIndex]);

  const select = (item) => {
    setQuery(item.label);
    setOpen(false);
    setActiveIndex(-1);
    onSelect?.(item);
  };

  const handleKeyDown = (e) => {
    const count = results.length;
    switch (e.key) {
      case 'ArrowDown':
        e.preventDefault();
        if (!open) setOpen(true);
        else if (count) setActiveIndex((i) => (i + 1) % count);
        break;
      case 'ArrowUp':
        e.preventDefault();
        if (count) setActiveIndex((i) => (i <= 0 ? count - 1 : i - 1));
        break;
      case 'Enter':
        if (showOptions && activeIndex >= 0) {
          e.preventDefault();
          select(results[activeIndex]);
        }
        break;
      case 'Escape':
        if (open) setOpen(false);
        else setQuery('');
        break;
      default:
    }
  };

  return (
    <div className="autocomplete">
      <label htmlFor={`${baseId}-input`}>{label}</label>
      <input
        id={`${baseId}-input`}
        type="text"
        role="combobox"
        autoComplete="off"
        placeholder={placeholder}
        value={query}
        aria-expanded={showOptions}
        aria-controls={listboxId}
        aria-autocomplete="list"
        aria-activedescendant={showOptions && activeIndex >= 0 ? optionId(activeIndex) : undefined}
        onChange={(e) => {
          setQuery(e.target.value);
          setOpen(true);
        }}
        onFocus={() => setOpen(true)}
        onBlur={() => setOpen(false)}
        onKeyDown={handleKeyDown}
      />

      {showPanel && (
        <div className="autocomplete-panel">
          {showOptions && (
            <ul
              id={listboxId}
              role="listbox"
              aria-label={`${label} suggestions`}
              onMouseDown={(e) => e.preventDefault()} // keep input focused so blur doesn't close before click
            >
              {results.map((item, i) => (
                <li
                  key={item.id}
                  id={optionId(i)}
                  role="option"
                  aria-selected={i === activeIndex}
                  className={i === activeIndex ? 'active' : ''}
                  onMouseEnter={() => setActiveIndex(i)}
                  onClick={() => select(item)}
                >
                  <Highlight text={item.label} query={query} />
                </li>
              ))}
            </ul>
          )}
          {loading && <div className="autocomplete-status">Searching…</div>}
          {showEmpty && <div className="autocomplete-status">No results for “{query.trim()}”</div>}
          {showError && <div className="autocomplete-status error">Something went wrong. Try again.</div>}
        </div>
      )}

      <div className="sr-only" role="status" aria-live="polite">
        {showOptions ? `${results.length} suggestions available` : showEmpty ? 'No results' : ''}
      </div>
    </div>
  );
}
```

**`App.jsx`**

```jsx
import { useState } from 'react';
import Autocomplete from './Autocomplete';
import { searchCountries } from './api';

export default function App() {
  const [selected, setSelected] = useState(null);

  return (
    <main style={{ maxWidth: 420, margin: '40px auto', fontFamily: 'system-ui, sans-serif' }}>
      <Autocomplete searchFn={searchCountries} label="Country" onSelect={setSelected} />
      {selected && <p>Selected: {selected.label}</p>}
    </main>
  );
}
```

**`autocomplete.css`**

```css
.autocomplete { position: relative; display: flex; flex-direction: column; gap: 4px; }
.autocomplete input { padding: 10px 12px; font-size: 1rem; border: 1px solid #ccc; border-radius: 6px; }
.autocomplete input:focus-visible { outline: 2px solid #4f46e5; outline-offset: 1px; }

.autocomplete-panel {
  position: absolute; top: 100%; left: 0; right: 0; z-index: 10; margin-top: 4px;
  background: #fff; border: 1px solid #ddd; border-radius: 6px;
  box-shadow: 0 8px 20px rgba(0, 0, 0, 0.12); max-height: 280px; overflow-y: auto;
}
.autocomplete-panel ul { list-style: none; margin: 0; padding: 4px 0; }
.autocomplete-panel li { padding: 8px 12px; cursor: pointer; }
.autocomplete-panel li.active { background: #eef2ff; }
.autocomplete-panel mark { background: transparent; font-weight: 700; color: #4f46e5; }
.autocomplete-status { padding: 8px 12px; color: #666; font-size: 0.9rem; }
.autocomplete-status.error { color: #b91c1c; }

.sr-only {
  position: absolute; width: 1px; height: 1px; margin: -1px; padding: 0;
  overflow: hidden; clip: rect(0 0 0 0); white-space: nowrap; border: 0;
}
```

</details>

---

## 6. How It Works

- **Two copies of the input:** `query` updates on every keystroke (responsive typing); `debounced` is derived by `useDebounce` and is the *only* thing that triggers network work.
- **Race-condition fix:** every effect run creates its own `AbortController`; the cleanup (`abort()`) runs before the next effect run or on unmount. Slow response for `"ind"` can't land after `"india"`: it's rejected with `AbortError` and swallowed. The extra `controller.signal.aborted` check protects against `searchFn` implementations that ignore the signal.
- **Cache:** `useRef(new Map())` persists across renders without causing renders. Key is the lower-cased query. Errors are **not** cached.
- **No stale closures:** results are set from the promise with local variables; `searchFn` and `minChars` are listed in the deps (so `searchFn` must be stable).
- **Keep old results while loading:** `setState(prev => ({ ...prev, status: 'loading' }))` avoids dropdown flicker.
- **`isPending`:** covers the debounce gap between typing and the request start so "Searching…" shows immediately.
- **`enabled: open`:** after selecting an item the dropdown closes and the effect doesn't run another search for the filled-in label.
- **Keyboard / ARIA combobox:** DOM focus stays on the input; `aria-activedescendant` points to the highlighted option, which a screen reader announces. `scrollIntoView({ block: 'nearest' })` keeps it visible.
- **`onMouseDown` preventDefault** on the list prevents the input's `blur` (which closes the list) from firing before the option's `click`.
- **Highlight** uses `indexOf` + `<mark>` — no `dangerouslySetInnerHTML`, no regex-escaping bugs.

---

## 7. Edge Cases

- Query shorter than `minChars` → idle, panel hidden.
- Whitespace-only input → trimmed, treated as empty.
- Fast typing, slow API → earlier requests aborted; only the latest result applies.
- Same query again (also different case) → served from cache instantly.
- API error / network down → error row; next keystroke retries (errors aren't cached).
- Empty response → "No results" (only after success, not during loading).
- Clicking an option → no premature blur close.
- Backspace to a shorter query → effect returns idle and aborts the in-flight request.
- Unmount mid-request → abort in cleanup; no "setState on unmounted" work.
- Selecting an item re-fills the input — must not re-open or re-search (`enabled`).
- `Enter` with no highlighted item → submit / default behaviour left alone.
- Result list changes under the highlight → `activeIndex` reset to -1.
- Very long labels → CSS ellipsis; special regex characters in query are harmless (no regex used).
- IME composition → debounce naturally waits; for strictness ignore `isComposing` keydowns.
- React 18 StrictMode double-invoked effect → first request aborted, second runs; fine.

---

## 8. Accessibility

- ARIA 1.2 combobox: `role="combobox"` on the input with `aria-expanded`, `aria-controls`, `aria-autocomplete="list"`, `aria-activedescendant`.
- `role="listbox"` / `role="option"` with `aria-selected` on the highlighted option; unique ids via `useId`.
- Live region (`role="status"`) announces result counts / "No results".
- Visible `<label>` associated with the input.
- Keyboard: ↑/↓, Enter, Esc; `Tab` leaves the field (and the blur closes the panel).
- Don't move DOM focus into the list; keep it in the input.
- Contrast for highlight/active states; don't rely on bold alone for the active option.
- Error text is announced (`role="alert"` if you want it assertive).

---

## 9. Performance Considerations

- **Debounce** cuts requests; typical 250–400 ms. **Throttle** is for scroll/resize, not typing.
- **Abort** frees the connection and avoids wasted JSON parsing/render work.
- **Cache** (Map, optionally with TTL/LRU) removes repeat requests; also prefix-cache (reuse `"in"` results to answer `"ind"` locally when the dataset is small).
- Limit list size (`slice(0, 8)` / `limit=8` server-side) to bound DOM size.
- `memo` the option row if rendering hundreds of options; virtualise beyond that.
- Keep `searchFn` stable — otherwise the effect re-runs every render.
- Server should rank/limit; the client only displays.

---

## 10. Follow-up Questions

<details>
<summary><strong>1. Debounce vs throttle — which and why?</strong></summary>

Debounce waits until the user *stops* typing for N ms, so it fires once with the final value — perfect for search. Throttle fires at most once per interval during continuous events — suits scroll/resize/drag, but would send intermediate partial queries for typing.

</details>

<details>
<summary><strong>2. Walk me through the race condition and how AbortController fixes it.</strong></summary>

Requests for `"a"` (slow) and `"ab"` (fast) are sent; `"ab"` returns first, then `"a"` returns and overwrites the list with stale data. Aborting the previous request in the effect cleanup guarantees it never resolves into state. Without `AbortController` you can use a `let ignore = false` flag set in cleanup, or a request-id ref; abort additionally cancels the network work.

</details>

<details>
<summary><strong>3. What if the API doesn't support abort?</strong></summary>

Keep the guard: `let ignore = false; ... if (!ignore) setState(...); return () => { ignore = true; }`. The request still completes but its result is discarded. (Our hook does this too via `signal.aborted`.)

</details>

<details>
<summary><strong>4. Improve the cache (TTL, LRU, in-flight dedupe).</strong></summary>

Store `{ value, expiresAt }` and skip expired entries; cap size by deleting the oldest key (`Map` keeps insertion order → `map.keys().next().value`); for in-flight dedupe cache the *promise* so concurrent identical queries share one request.

</details>

<details>
<summary><strong>5. Replace the hook with a data-fetching library?</strong></summary>

React Query / SWR: `useQuery({ queryKey: ['search', debounced], queryFn: ({ signal }) => searchFn(debounced, signal), enabled, staleTime, placeholderData: keepPreviousData })` gives caching, abort, retries, and dedupe out of the box. Still need to debounce the key yourself and know how it works underneath.

</details>

<details>
<summary><strong>6. Support multi-select with chips / "creatable" values.</strong></summary>

State becomes `selected: Item[]`; selecting appends (dedupe by id) and clears the query while keeping the list open; Backspace on empty input removes the last chip; filter out already-selected items. "Creatable": show a "Create 'xyz'" option when no exact match.

</details>

<details>
<summary><strong>7. How would you build it for 100k local items?</strong></summary>

Skip the network: pre-normalise labels, filter in a **Web Worker** or use a **Trie**/prefix index; `useDeferredValue`/`startTransition` for the list; cap results; virtualise rendering.

</details>

<details>
<summary><strong>8. How to handle recent searches / show suggestions on focus?</strong></summary>

When `query` is empty and the input is focused, display `recent` items from `localStorage` (deduped, capped ~5). Selecting one behaves like a normal selection. Keep that source separate from the search hook.

</details>

---

## 11. Common Mistakes

- Calling the API on every keystroke (no debounce).
- Debouncing *inside* the event handler with a new function every render (`debounce(fn)` re-created → never debounces). Use an effect/`useRef`-held timer or `useDebounce`.
- **No cancellation → race condition** (stale results overwrite fresh ones).
- Forgetting `clearTimeout` / `abort()` in effect cleanup.
- Treating `AbortError` as a real error and showing an error message.
- Closing the dropdown on `blur` without preventing it during option clicks (click never registers).
- Missing `key` on options or using index when items can change.
- Highlight index not reset when the list changes; no wrap-around.
- Not handling empty/error/loading states.
- Search triggered by selecting an item (input value change) → re-opens the dropdown.
- Unstable `searchFn` (inline arrow) in effect deps → infinite refetch loop.
- Using `dangerouslySetInnerHTML` for highlighting (XSS).
- Skipping ARIA / keyboard support.

---

## 12. Interview Tips & Time Plan

Say the three keywords early: **debounce, abort/race, cache.** Build incrementally: static filter → debounced async → keyboard → ARIA → polish. Use a mock API with random delay so you can *demonstrate* the race fix.

| Minutes | Activity |
| --- | --- |
| 0–5 | Clarify requirements; sketch state + data flow |
| 5–12 | Input + dropdown with local filtering |
| 12–20 | `useDebounce` + async mock API + loading/empty states |
| 20–30 | `AbortController` race handling + error state |
| 30–36 | Cache with `useRef(Map)` |
| 36–46 | Keyboard navigation + select + blur/click handling |
| 46–54 | ARIA combobox attributes + live region + highlight |
| 54–60 | Edge cases, discuss follow-ups (React Query, worker, LRU) |

If time is short (45 min): skip highlight and live region, but *mention* them.

---

## 13. Practice Checklist

- [ ] Debounce hook written from memory (with cleanup).
- [ ] Async search effect with `AbortController` and `AbortError` handling.
- [ ] Reproduce the race condition without abort, then fix it.
- [ ] `Map` cache in `useRef`; explain why not state.
- [ ] Loading / empty / error states.
- [ ] ↑ ↓ Enter Esc with wrap-around and `scrollIntoView`.
- [ ] Mouse selection works (blur vs click solved).
- [ ] ARIA combobox + `aria-activedescendant`.
- [ ] Highlight match without `dangerouslySetInnerHTML`.
- [ ] Swap the mock for a real `fetch` API.
- [ ] Add LRU/TTL cache or React Query version.
- [ ] Add recent searches or multi-select.
