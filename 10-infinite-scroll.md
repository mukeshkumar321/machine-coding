# Infinite Scroll — Machine Coding

> **Difficulty:** Medium | **Time:** 40–50 min | **Asked at / Frequency:** Very common (feed / product-list / "design a Twitter timeline" style rounds) — ⭐⭐⭐⭐⭐ | **Tier:** 1 (must know)

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

Build a list that **loads the next page automatically as the user scrolls near the bottom**, using `IntersectionObserver` (not a scroll listener). Show **loading**, **error (with retry)**, and **end-of-list** states, never fetch the same page twice, and **dedupe** items that appear in more than one page.

---

## 2. Requirements

### Functional

- Initial page loads automatically; more pages load when the bottom sentinel enters (or nears) the viewport.
- Loading indicator while fetching.
- Error message with **Retry**; no automatic infinite retry loop.
- End-of-list message when the API says there is no more data.
- Empty state if the first page returns nothing.
- Items are deduplicated by `id`.

### Non-functional

- Use `IntersectionObserver` with a `rootMargin` prefetch buffer.
- No duplicate/overlapping requests; cancel on unmount (`AbortController`).
- Observer disconnected on cleanup — no leaks.
- Logic in a reusable hook; the UI component stays declarative.
- Works with offset or cursor pagination.

---

## 3. Clarifying Questions to Ask

- Pagination style: page number, offset/limit, or **cursor**? Does the API return `hasMore`/`nextCursor`?
- Page size? Total count known?
- Window scroll or a scrollable container?
- Can items be inserted at the top while scrolling (live feed) → duplicates/shifts?
- Do we need to retain scroll position when navigating away and back?
- Is virtualisation required (thousands of items)?
- Should there also be a manual "Load more" button (accessibility)?

---

## 4. Approach & Component Design

**State shape**

```js
items:   Item[]                    // accumulated, deduped by id
status:  'idle' | 'loading' | 'error'
hasMore: boolean                   // false once nextCursor === null
// refs (no re-render needed)
cursorRef:   number | string       // where the next page starts
inFlightRef: boolean               // synchronous guard against double fetch
abortRef:    AbortController
sentinelRef: HTMLDivElement        // empty div after the last item
```

**Flow**

```
render list + <div ref=sentinel/>
    │
IntersectionObserver(sentinel, rootMargin 200px)
    │ isIntersecting && status==='idle' && hasMore
    ▼
loadMore(): inFlight? return → fetchPage(cursor, signal)
    ├─ ok  → append (dedupe by id), cursor = next, hasMore = next !== null, status 'idle'
    └─ err → status 'error'  (observer paused until Retry)
status changes → effect re-creates observer → fires immediately if sentinel is still visible
```

```
App
 └── InfiniteList
      ├── <ul> items (key=id) </ul>
      ├── Loading… | Error + Retry | "You're all caught up"
      └── <div ref=sentinelRef>        ← observed
 hook: useInfiniteScroll(fetchPage)
```

---

## 5. Reference Solution

<details>
<summary><strong>Show Full Code</strong></summary>

**`api.js`** (mock; fails once at cursor 20 to demo Retry)

```jsx
const TOTAL = 53;
const PAGE_SIZE = 10;
const failedOnce = new Set();

const abortError = () => new DOMException('Aborted', 'AbortError');

// Returns { items, nextCursor } — nextCursor === null means "no more pages"
export function fetchItems(cursor, signal) {
  return new Promise((resolve, reject) => {
    if (signal?.aborted) return reject(abortError());

    const timer = setTimeout(() => {
      if (cursor === 20 && !failedOnce.has(cursor)) {
        failedOnce.add(cursor);
        return reject(new Error('Network error'));
      }
      const size = Math.min(PAGE_SIZE, TOTAL - cursor);
      const items = Array.from({ length: size }, (_, i) => {
        const id = cursor + i + 1;
        return { id, title: `Item #${id}`, body: 'Lorem ipsum dolor sit amet, consectetur adipiscing elit.' };
      });
      const next = cursor + size;
      resolve({ items, nextCursor: next < TOTAL ? next : null });
    }, 700);

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

// Real API equivalent
// export async function fetchPosts(cursor, signal) {
//   const res = await fetch(`https://jsonplaceholder.typicode.com/posts?_start=${cursor}&_limit=10`, { signal });
//   if (!res.ok) throw new Error(`HTTP ${res.status}`);
//   const items = await res.json();
//   return { items, nextCursor: items.length === 10 ? cursor + 10 : null };
// }
```

**`useInfiniteScroll.js`**

```jsx
import { useCallback, useEffect, useRef, useState } from 'react';

// `fetchPage` must be a stable reference (module-level or useCallback)
export function useInfiniteScroll(fetchPage, { rootMargin = '200px' } = {}) {
  const [items, setItems] = useState([]);
  const [status, setStatus] = useState('idle'); // 'idle' | 'loading' | 'error'
  const [hasMore, setHasMore] = useState(true);

  const cursorRef = useRef(0);
  const inFlightRef = useRef(false);
  const abortRef = useRef(null);
  const sentinelRef = useRef(null);

  const loadMore = useCallback(async () => {
    if (inFlightRef.current) return; // synchronous guard — state would be too late
    inFlightRef.current = true;
    setStatus('loading');

    const controller = new AbortController();
    abortRef.current = controller;

    try {
      const page = await fetchPage(cursorRef.current, controller.signal);
      if (controller.signal.aborted) return;

      setItems((prev) => {
        const seen = new Set(prev.map((item) => item.id));
        return [...prev, ...page.items.filter((item) => !seen.has(item.id))];
      });
      cursorRef.current = page.nextCursor;
      setHasMore(page.nextCursor !== null);
      setStatus('idle');
    } catch (err) {
      if (err.name === 'AbortError') return;
      setStatus('error');
    } finally {
      inFlightRef.current = false;
    }
  }, [fetchPage]);

  // Observe the sentinel only while we are able to load.
  // Re-creating the observer after each load re-fires the callback if the
  // sentinel is still visible (short page / tall screen) => keeps filling the viewport.
  useEffect(() => {
    const node = sentinelRef.current;
    if (!node || !hasMore || status !== 'idle') return;

    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) loadMore();
      },
      { rootMargin }
    );
    observer.observe(node);
    return () => observer.disconnect();
  }, [hasMore, status, loadMore, rootMargin]);

  useEffect(() => () => abortRef.current?.abort(), []);

  return { items, status, hasMore, loadMore, sentinelRef };
}
```

**`InfiniteList.jsx`**

```jsx
import { useInfiniteScroll } from './useInfiniteScroll';
import { fetchItems } from './api';
import './infinite-list.css';

export default function InfiniteList() {
  const { items, status, hasMore, loadMore, sentinelRef } = useInfiniteScroll(fetchItems);

  return (
    <main className="feed-page">
      <h1>Feed</h1>

      <ul className="feed" aria-busy={status === 'loading'}>
        {items.map((item) => (
          <li key={item.id} className="feed-item">
            <h2>{item.title}</h2>
            <p>{item.body}</p>
          </li>
        ))}
      </ul>

      {status === 'loading' && (
        <p className="feed-status" role="status">Loading…</p>
      )}

      {status === 'error' && (
        <div className="feed-status error" role="alert">
          Couldn't load more items.{' '}
          <button type="button" onClick={loadMore}>Retry</button>
        </div>
      )}

      {!hasMore && items.length > 0 && <p className="feed-status">You're all caught up 🎉</p>}
      {!hasMore && items.length === 0 && <p className="feed-status">Nothing here yet.</p>}

      {hasMore && <div ref={sentinelRef} className="sentinel" aria-hidden="true" />}
    </main>
  );
}
```

**`App.jsx`**

```jsx
import InfiniteList from './InfiniteList';

export default function App() {
  return <InfiniteList />;
}
```

**`infinite-list.css`**

```css
.feed-page { max-width: 560px; margin: 0 auto; padding: 16px; font-family: system-ui, sans-serif; }
.feed { list-style: none; margin: 0; padding: 0; display: grid; gap: 12px; }
.feed-item { border: 1px solid #e5e7eb; border-radius: 8px; padding: 16px; min-height: 120px; }
.feed-item h2 { margin: 0 0 8px; font-size: 1.05rem; }
.feed-item p { margin: 0; color: #555; }
.feed-status { text-align: center; color: #666; padding: 16px 0; }
.feed-status.error { color: #b91c1c; }
.sentinel { height: 1px; }
```

</details>

---

## 6. How It Works

- **`IntersectionObserver` > scroll events:** it's asynchronous, off the main thread for the geometry work, and needs no throttling or `getBoundingClientRect` reads. `rootMargin: '200px'` triggers the load *before* the user hits the bottom.
- **Sentinel pattern:** an empty `div` after the last item is the only thing observed — no need to attach observers to list items.
- **Double-fetch guard with a ref:** `inFlightRef` updates synchronously; relying on `status === 'loading'` state alone is racy because state updates are async and the observer can fire twice before a re-render.
- **Observer lifecycle tied to `status`:** while loading or in error the observer is torn down; when status returns to `idle` a *new* observer is created, which fires its initial callback immediately. If the sentinel is still on-screen (tall viewport) the next page loads automatically, otherwise it waits for scroll. This avoids the classic "only loaded one page and got stuck" bug.
- **Error does not loop:** in `error` no observer exists; the user presses Retry (`loadMore` directly).
- **Dedupe:** build a `Set` of existing ids in the functional `setItems` update and filter incoming items — immutable and safe against stale state.
- **Cursor in a ref:** it changes every page but doesn't affect rendering; `loadMore` stays stable and never reads a stale cursor.
- **Cancellation:** unmount aborts the in-flight request; the `aborted` check prevents state updates afterwards.
- **`hasMore`** comes from the API (`nextCursor === null`), not from guessing by page length.

---

## 7. Edge Cases

- Viewport taller than first page → keeps loading until the sentinel leaves the viewport (observer re-creation).
- Fast scrolling → observer fires many times; `inFlightRef` allows only one request.
- Network error mid-scroll → error row + Retry; already-loaded items stay.
- Retry succeeds → continues seamlessly with the same cursor (not skipped or duplicated).
- Overlapping pages (insertions shift offsets) → dedupe by `id`.
- Empty first page → empty state, sentinel removed (`hasMore=false`).
- Last page smaller than page size → `nextCursor: null` → end message.
- Unmount during request → abort; no state update warnings.
- React 18 StrictMode (effects run twice in dev) → observer set up/torn down cleanly; guards prevent duplicate fetches.
- Browser back/forward → state is lost unless cached (see follow-ups).
- Items with variable/unknown height → layout shift can push the sentinel back into view; reserve min-height/skeletons.
- Sentinel hidden via `display: none` → never intersects; keep it rendered with `height: 1px`.
- IntersectionObserver unsupported (very old browsers) → fall back to a "Load more" button.

---

## 8. Accessibility

- `aria-busy` on the list while loading; loading text in `role="status"` (polite announcement); error in `role="alert"`.
- Infinite scroll can make the footer unreachable by keyboard and disorients screen-reader users → offer a **"Load more" button** (or a toggle) and keep footer links reachable.
- Announce how many items loaded ("10 more items loaded") in a polite live region.
- Keep focus stable: appending items below must not steal or reset focus.
- Provide a "Back to top" control; avoid auto-loading forever without an end.
- Sentinel is `aria-hidden` and non-focusable.
- Don't use a spinner without an accessible text equivalent.

---

## 9. Performance Considerations

- `IntersectionObserver` avoids scroll-handler jank; no throttling code needed.
- `rootMargin` prefetch hides latency; tune it with page size and API speed.
- **DOM growth:** thousands of nodes slow layout/paint and memory → **virtualise** (`react-window`, `@tanstack/virtual`) or recycle/window old pages.
- `memo` the row component; use stable keys (`id`).
- Avoid storing duplicate derived arrays; dedupe in the append step only.
- Image-heavy feeds: `loading="lazy"`, explicit width/height to avoid layout shift (CLS).
- Cancel obsolete requests (e.g. when filters change) with `AbortController`.
- Prefer cursor pagination server-side — offset gets slow and unstable on large/changing datasets.

---

## 10. Follow-up Questions

<details>
<summary><strong>1. Why IntersectionObserver instead of a scroll listener?</strong></summary>

A scroll handler runs on every scroll event on the main thread and needs throttling plus layout reads (`scrollTop + clientHeight >= scrollHeight - threshold`), which can force reflow. `IntersectionObserver` delivers batched, async visibility changes calculated by the browser. Simpler, faster, and supports `rootMargin` prefetch and custom scroll containers (`root`).

</details>

<details>
<summary><strong>2. How do you avoid duplicate requests?</strong></summary>

A synchronous in-flight ref guard (state is too slow), disconnecting the observer while loading, and keeping the cursor in a ref so each request uses the latest value. With data libraries, `useInfiniteQuery` handles it (`fetchNextPage` is deduped via `isFetchingNextPage`).

</details>

<details>
<summary><strong>3. How would you do it in a scrollable container instead of the window?</strong></summary>

Pass the container node as `root` in the observer options (`{ root: containerRef.current, rootMargin }`), and make sure the container has a fixed height with `overflow-y: auto`. Re-create the observer if the root node changes.

</details>

<details>
<summary><strong>4. Handle thousands of items — virtualisation?</strong></summary>

Render only visible rows plus overscan (`react-window`/`@tanstack/virtual`); the "load more" trigger becomes "last virtual item index ≥ items.length − threshold" instead of a DOM sentinel. Variable heights need measuring (`measureElement`).

</details>

<details>
<summary><strong>5. Restore scroll position after navigating back.</strong></summary>

Cache `{ items, cursor, hasMore, scrollY }` in a store/`sessionStorage`/router state when leaving; on return, hydrate the state and `window.scrollTo(0, scrollY)` after the first paint (`useLayoutEffect`). With `history.scrollRestoration = 'manual'` you control it.

</details>

<details>
<summary><strong>6. Offset vs cursor pagination — which and why?</strong></summary>

Offset (`?page=3`) is simple but breaks when items are inserted/deleted during browsing (duplicates/skips) and gets slow for deep pages. Cursor (`?after=<id>`) is stable and efficient. The client treats it as an opaque token; dedupe is still a cheap safety net.

</details>

<details>
<summary><strong>7. Bi-directional infinite scroll (chat history)?</strong></summary>

Observe a top sentinel too; when prepending, preserve the scroll position by measuring `scrollHeight` before and after the update and adjusting `scrollTop` by the difference (in `useLayoutEffect`), or use `overflow-anchor`/column-reverse tricks.

</details>

<details>
<summary><strong>8. Reset when a filter/search changes.</strong></summary>

Abort the in-flight request, clear `items`, set `cursor=0`, `hasMore=true`, `status='idle'`, and key the hook's owner by the filter (`<InfiniteList key={filter}>`) or include `filter` in `fetchPage`'s identity with a reset effect. Beware late responses from the old filter — guard with the abort check.

</details>

---

## 11. Common Mistakes

- Scroll listener without throttling, reading layout on every event.
- Observer created once on mount: after the first load nothing triggers again (sentinel stays visible) or it fires repeatedly.
- Guarding with `status` state only → duplicate requests / double page append.
- `setItems([...items, ...newItems])` (stale closure) instead of functional update.
- Not deduping; using index as key (items shift → remount/flicker).
- Not disconnecting the observer / aborting requests on unmount.
- Retrying automatically in a loop when the API is down.
- No `hasMore` — requesting pages forever or showing "Loading…" at the end.
- Observing every item, or observing a sentinel hidden with `display: none`.
- Forgetting empty / error / end states.
- Putting `cursor` in state and reading it in a memoised callback → stale cursor, same page loaded twice.
- Unbounded DOM growth with no virtualisation discussion.

