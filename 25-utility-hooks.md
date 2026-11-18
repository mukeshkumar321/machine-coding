# Utility Hooks — Machine Coding

> **Difficulty:** Medium | **Time:** 60–90 min | **Frequency:** ⭐⭐⭐⭐

## Table of Contents

1. [useDebounce](#1-usedebounce)
2. [useThrottle](#2-usethrottle)
3. [useLocalStorage](#3-uselocalstorage)
4. [usePrevious](#4-useprevious)
5. [useFetch](#5-usefetch)
6. [useOnClickOutside](#6-useonclickoutside)
7. [useIntersectionObserver](#7-useintersectionobserver)
8. [useInterval](#8-useinterval)
9. [useToggle](#9-usetoggle)
10. [useEventListener](#10-useeventlistener)
11. [Practice Questions](#11-practice-questions)

---

> **How to use this file:** every hook is written as a standalone module (`// useXxx.js` on the first line of the code block). Each follows the same recipe that interviewers look for: **stable references (`useCallback`/refs)**, **cleanup in effects**, **latest-callback ref to avoid stale closures**, and **no work in render**.

---

## 1. useDebounce

Delays reacting to a fast-changing value (search box) or delays running a function (resize, autosave) until the user stops for `delay` ms.

### Code

**Value version** — returns a value that lags behind by `delay` ms.

```js
// useDebounce.js
import { useCallback, useEffect, useMemo, useRef, useState } from 'react';

export function useDebounce(value, delay = 300) {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id);
  }, [value, delay]);

  return debounced;
}

// Callback version — returns a stable debounced function with .cancel()
export function useDebouncedCallback(fn, delay = 300) {
  const fnRef = useRef(fn);
  const timerRef = useRef();

  useEffect(() => {
    fnRef.current = fn;
  });

  const cancel = useCallback(() => clearTimeout(timerRef.current), []);

  const debounced = useMemo(() => {
    const d = (...args) => {
      clearTimeout(timerRef.current);
      timerRef.current = setTimeout(() => fnRef.current(...args), delay);
    };
    d.cancel = cancel;
    return d;
  }, [delay, cancel]);

  useEffect(() => cancel, [cancel]); // no timer fires after unmount

  return debounced;
}
```

### Usage

```jsx
function Search() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 400);

  useEffect(() => {
    if (!debouncedQuery) return;
    const controller = new AbortController();
    fetch(`/api/search?q=${debouncedQuery}`, { signal: controller.signal });
    return () => controller.abort();
  }, [debouncedQuery]);

  return <input value={query} onChange={(e) => setQuery(e.target.value)} />;
}

function Editor() {
  const save = useDebouncedCallback((text) => api.save(text), 1000);
  return <textarea onChange={(e) => save(e.target.value)} onBlur={save.cancel} />;
}
```

### Gotchas / interview follow-ups

- **Debounce the value, not the input.** The input must stay controlled by the immediate `query`; only the *effect/API call* uses `debouncedQuery`.
- **Why not `const d = debounce(fn, 300)` in the component body?** It creates a *new* timer closure every render, so each keystroke has its own timer and nothing is ever cancelled. The debounced function must live in `useMemo`/`useRef` (stable identity).
- **Latest-callback ref** (`fnRef`) lets the debounced function stay stable while always calling the newest `fn` (fresh props/state) — avoids stale closures without re-creating the debounce.
- **Cancel on unmount** to avoid `setState` / API calls after the component is gone.
- First render: `useState(value)` means the debounced value equals the initial value immediately (no 300 ms delay on mount).
- **Debounce vs throttle:** debounce = run *after* quiet period (search, autosave); throttle = run *at most every N ms* (scroll, resize, drag).
- Changing `delay` re-creates the debounced function (and `useDebounce` restarts its timer) — acceptable, but know it.

---

## 2. useThrottle

Limits how often a value updates (or a function runs) — leading edge fires immediately, trailing edge guarantees the last value is eventually applied.

### Code

```js
// useThrottle.js
import { useCallback, useEffect, useRef, useState } from 'react';

export function useThrottle(value, interval = 500) {
  const [throttled, setThrottled] = useState(value);
  const lastRun = useRef(Date.now());

  useEffect(() => {
    const remaining = interval - (Date.now() - lastRun.current);

    if (remaining <= 0) {
      lastRun.current = Date.now();
      setThrottled(value);
      return;
    }

    const id = setTimeout(() => {
      lastRun.current = Date.now();
      setThrottled(value);
    }, remaining);
    return () => clearTimeout(id);
  }, [value, interval]);

  return throttled;
}

// Callback version: leading call + trailing call with the latest args
export function useThrottledCallback(fn, interval = 300) {
  const fnRef = useRef(fn);
  const lastRun = useRef(0);
  const timerRef = useRef();

  useEffect(() => {
    fnRef.current = fn;
  });

  useEffect(() => () => clearTimeout(timerRef.current), []);

  return useCallback(
    (...args) => {
      const remaining = interval - (Date.now() - lastRun.current);
      clearTimeout(timerRef.current);

      if (remaining <= 0) {
        lastRun.current = Date.now();
        fnRef.current(...args);
      } else {
        timerRef.current = setTimeout(() => {
          lastRun.current = Date.now();
          fnRef.current(...args);
        }, remaining);
      }
    },
    [interval]
  );
}
```

### Usage

```jsx
function MouseTracker() {
  const [pos, setPos] = useState({ x: 0, y: 0 });
  const throttledPos = useThrottle(pos, 200);
  return (
    <div onMouseMove={(e) => setPos({ x: e.clientX, y: e.clientY })}>
      {throttledPos.x}, {throttledPos.y}
    </div>
  );
}

function ScrollLogger() {
  const onScroll = useThrottledCallback(() => console.log(window.scrollY), 250);
  useEventListener('scroll', onScroll);
  return null;
}
```

### Gotchas / interview follow-ups

- Throttled **value** still re-renders the component on every `setPos` — it only slows the *downstream* update. To avoid the re-renders themselves, throttle the **event handler** (callback version).
- **Trailing edge matters:** without it, the final mouse position after the user stops moving would be dropped.
- Uses `Date.now()` for the elapsed check + `setTimeout` for the trailing call. `requestAnimationFrame` is the better throttle for visual updates (scroll/resize) — one call per frame.
- Throttle vs debounce in one sentence: throttle guarantees *regular* execution during activity; debounce waits for activity to *stop*.
- Cleanup the pending timer on unmount (both versions do).

---

## 3. useLocalStorage

State that persists in `localStorage`, survives reloads, and stays in sync across **browser tabs**.

### Code

```js
// useLocalStorage.js
import { useCallback, useEffect, useRef, useState } from 'react';

const resolve = (v) => (v instanceof Function ? v() : v);

// returns undefined when missing, corrupt, or storage is unavailable (private mode / SSR)
function readStorage(key) {
  try {
    const raw = window.localStorage.getItem(key);
    return raw === null ? undefined : JSON.parse(raw);
  } catch {
    return undefined;
  }
}

export function useLocalStorage(key, initialValue) {
  // lazy initializer: storage is read once, not on every render
  const [value, setValue] = useState(() => {
    const stored = readStorage(key);
    return stored !== undefined ? stored : resolve(initialValue);
  });

  // always holds the latest value so back-to-back updater calls compose
  const valueRef = useRef(value);

  const set = useCallback(
    (next) => {
      const v = next instanceof Function ? next(valueRef.current) : next;
      valueRef.current = v;
      setValue(v);
      try {
        window.localStorage.setItem(key, JSON.stringify(v));
      } catch {
        // quota exceeded / storage disabled: keep working in memory
      }
    },
    [key]
  );

  const remove = useCallback(() => {
    try {
      window.localStorage.removeItem(key);
    } catch {}
    const v = resolve(initialValue);
    valueRef.current = v;
    setValue(v);
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [key]);

  // cross-tab sync: the 'storage' event fires in *other* tabs only
  useEffect(() => {
    const onStorage = (e) => {
      if (e.storageArea !== window.localStorage) return;
      if (e.key !== null && e.key !== key) return; // e.key === null → storage.clear()
      let v;
      try {
        v = e.newValue === null ? resolve(initialValue) : JSON.parse(e.newValue);
      } catch {
        return;
      }
      valueRef.current = v;
      setValue(v);
    };
    window.addEventListener('storage', onStorage);
    return () => window.removeEventListener('storage', onStorage);
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [key]);

  return [value, set, remove];
}
```

### Usage

```jsx
function ThemeToggle() {
  const [theme, setTheme, resetTheme] = useLocalStorage('theme', 'light');
  return (
    <>
      <button onClick={() => setTheme((t) => (t === 'light' ? 'dark' : 'light'))}>{theme}</button>
      <button onClick={resetTheme}>Reset</button>
    </>
  );
}

const [cart, setCart] = useLocalStorage('cart', () => []); // lazy default
```

### Gotchas / interview follow-ups

- **Lazy initializer** `useState(() => read())` — `localStorage.getItem` + `JSON.parse` is synchronous and slow-ish; calling it as `useState(read())` runs it on every render.
- **Always `try/catch`:** `JSON.parse` throws on corrupt data; `getItem`/`setItem` throw in Safari private mode, when cookies are blocked, or when quota (~5 MB) is exceeded; `window` is undefined during SSR.
- **`storage` event fires only in *other* tabs/windows**, never in the tab that wrote. So same-tab sync between two components needs a custom event or `useSyncExternalStore` (see Practice Questions).
- **Functional updates** (`setX(prev => ...)`) must work — that's why `valueRef` is kept in sync; otherwise two quick `set(c => c + 1)` calls both read the same stale value.
- **Hydration mismatch (SSR):** server renders the default, client reads storage → mismatch. Read storage in an effect after mount, or use `useSyncExternalStore` with `getServerSnapshot`.
- `JSON.stringify` loses `Date`, `Map`, `Set`, `undefined`, functions — mention serializer options.
- **Changing `key` at runtime** does not re-read storage in this version; real-world fix: re-read in an effect keyed on `key`.
- `sessionStorage` is the same API but per-tab and does not fire cross-tab events.

---

## 4. usePrevious

Returns the value from the **previous render** — handy for comparing old vs new props (animations, "went up/down", analytics).

### Code

```js
// usePrevious.js
import { useEffect, useRef } from 'react';

export function usePrevious(value) {
  const ref = useRef();

  useEffect(() => {
    ref.current = value; // runs AFTER render, so render sees the old value
  });

  return ref.current;
}

// Variant: previous *distinct* value (ignores re-renders where value didn't change)
export function usePreviousDistinct(value) {
  const prev = useRef();
  const curr = useRef(value);

  if (curr.current !== value) {
    prev.current = curr.current;
    curr.current = value;
  }

  return prev.current;
}
```

### Usage

```jsx
function Price({ value }) {
  const prev = usePrevious(value);
  const trend = prev === undefined ? '' : value > prev ? '▲' : value < prev ? '▼' : '';
  return <span>{value} {trend}</span>;
}
```

### Gotchas / interview follow-ups

- **How it works:** `useRef` persists across renders; the effect updates it *after* the render commits, so during render `ref.current` is still the old value. The returned value is read **before** the effect runs.
- **Returns `undefined` on first render.**
- **Not "previous distinct value":** if the component re-renders for an unrelated reason, `usePrevious(count)` returns the *current* `count` (the effect already stored it). Use `usePreviousDistinct` when you need "last different value".
- Writing `ref.current` during render (as `usePreviousDistinct` does) is generally discouraged in concurrent React; it is safe here because the write is idempotent for the same inputs, but know the trade-off.
- Refs don't trigger re-renders — that's why the *next* render shows the new "previous".
- If a state update re-renders immediately after, you can see the same value for both — see Practice Question 3.

---

## 5. useFetch

Declarative data fetching with **loading/error/data**, **AbortController**, an in-memory **cache**, **retry with backoff**, **race safety**, and `refetch`.

### Code

```js
// useFetch.js
import { useCallback, useEffect, useState } from 'react';

const cache = new Map(); // module-level: shared by every component using the hook

export const clearFetchCache = () => cache.clear();

// resolves after ms, or immediately if the signal aborts
const wait = (ms, signal) =>
  new Promise((resolve) => {
    const id = setTimeout(resolve, ms);
    signal.addEventListener(
      'abort',
      () => {
        clearTimeout(id);
        resolve();
      },
      { once: true }
    );
  });

export function useFetch(url, { enabled = true, retries = 0, retryDelay = 300, useCache = true } = {}) {
  const [state, setState] = useState({
    data: undefined,
    error: null,
    status: url && enabled ? 'loading' : 'idle',
  });
  const [nonce, setNonce] = useState(0);

  useEffect(() => {
    if (!url || !enabled) return;

    if (useCache && cache.has(url)) {
      setState({ data: cache.get(url), error: null, status: 'success' });
      return;
    }

    const controller = new AbortController();
    const { signal } = controller;
    setState({ data: undefined, error: null, status: 'loading' });

    (async () => {
      for (let attempt = 0; ; attempt++) {
        try {
          const res = await fetch(url, { signal });
          if (!res.ok) throw Object.assign(new Error(`HTTP ${res.status}`), { status: res.status });
          const data = await res.json();
          if (signal.aborted) return; // a newer request/unmount owns the state now
          cache.set(url, data);
          setState({ data, error: null, status: 'success' });
          return;
        } catch (error) {
          if (signal.aborted) return; // AbortError is not a real failure
          const clientError = error.status >= 400 && error.status < 500;
          if (clientError || attempt >= retries) {
            setState({ data: undefined, error, status: 'error' });
            return;
          }
          await wait(retryDelay * 2 ** attempt, signal); // exponential backoff
          if (signal.aborted) return;
        }
      }
    })();

    return () => controller.abort();
  }, [url, enabled, retries, retryDelay, useCache, nonce]);

  const refetch = useCallback(() => {
    cache.delete(url);
    setNonce((n) => n + 1);
  }, [url]);

  return { ...state, loading: state.status === 'loading', refetch };
}
```

### Usage

```jsx
function User({ id }) {
  const { data, error, loading, refetch } = useFetch(`/api/users/${id}`, { retries: 2 });

  if (loading) return <Spinner />;
  if (error) return <button onClick={refetch}>Failed ({error.message}) — retry</button>;
  return <h1>{data.name}</h1>;
}

// conditional fetch: wait until we have a token
const { data } = useFetch('/api/me', { enabled: Boolean(token) });
```

### Gotchas / interview follow-ups

- **Race condition:** user switches `id` 1 → 2 quickly; response 1 arrives *after* response 2 and overwrites it. Fix = effect **cleanup aborts** the old request (+ `signal.aborted` check before `setState`). The alternative flag pattern: `let ignore = false; return () => { ignore = true }`.
- **Abort ≠ error:** `fetch` rejects with `AbortError` on abort — swallow it, never show it in the UI.
- **`fetch` does not reject on 404/500** — you must check `res.ok`. Don't retry 4xx (client errors won't fix themselves); do retry network errors and 5xx.
- **Backoff:** `retryDelay * 2 ** attempt` (300 → 600 → 1200 ms). Add jitter in real life to avoid thundering herd. The backoff wait itself is abortable.
- **Cache:** module-level `Map` keyed by URL (or `[url, params]`). Mention TTL/stale-while-revalidate, request **de-duplication** (store the in-flight *promise*), and that real apps use React Query / SWR.
- **StrictMode** (dev) runs the effect twice: mount → cleanup (abort) → mount. The abort makes the first request harmless; that is exactly why cleanup matters.
- **Dependencies:** options are destructured into primitives so an inline `{ retries: 2 }` object doesn't re-trigger the effect every render. Passing an object/function (`headers`, `parse`) needs a ref or stable identity.
- The state is reset to `loading` / no data when `url` changes (avoids showing user 1's data under user 2's id). Keeping stale data while revalidating is a conscious UX choice.
- Don't make the effect callback `async` (it would return a Promise instead of a cleanup function) — use an inner async IIFE.

---

## 6. useOnClickOutside

Calls a handler when the user presses outside an element — dropdowns, popovers, modals.

### Code

```js
// useOnClickOutside.js
import { useEffect, useRef } from 'react';

export function useOnClickOutside(ref, handler) {
  const handlerRef = useRef(handler);

  useEffect(() => {
    handlerRef.current = handler;
  });

  useEffect(() => {
    const listener = (event) => {
      const el = ref.current;
      if (!el || el.contains(event.target)) return; // inside click or not mounted
      handlerRef.current(event);
    };

    document.addEventListener('mousedown', listener);
    document.addEventListener('touchstart', listener);
    return () => {
      document.removeEventListener('mousedown', listener);
      document.removeEventListener('touchstart', listener);
    };
  }, [ref]);
}
```

### Usage

```jsx
function Dropdown() {
  const [open, setOpen] = useState(false);
  const ref = useRef(null);
  useOnClickOutside(ref, () => setOpen(false));

  return (
    <div ref={ref}>
      <button onClick={() => setOpen((o) => !o)}>Menu</button>
      {open && <ul>...</ul>}
    </div>
  );
}
```

### Gotchas / interview follow-ups

- **`mousedown`/`touchstart` vs `click`:** `click` fires after `mouseup`, so if the click *removes* the target from the DOM (e.g. a list item that unmounts), `contains(target)` is false and an inside click is misread as outside. `mousedown` fires before React re-renders. `pointerdown` covers mouse + touch + pen in one listener.
- **Latest-handler ref** means the listener is attached once, even if the caller passes an inline arrow function each render.
- **Portals:** a modal rendered through `createPortal` is outside `ref.current` in the DOM, so clicking inside it counts as "outside". Accept an array of refs, or check `event.composedPath()`.
- **Toggle button problem:** if the toggle button is *outside* `ref`, pressing it fires outside-handler (closes) and then `onClick` (reopens). Put the button inside the ref'd wrapper or ignore it.
- Add `Escape` key handling and focus return for accessibility.
- Use `document` (not `window`) listeners; capture phase (`true`) is needed if children call `stopPropagation`.

---

## 7. useIntersectionObserver

Tells you when an element enters/leaves the viewport — lazy images, infinite scroll sentinel, "scroll spy", analytics impressions.

### Code

```js
// useIntersectionObserver.js
import { useEffect, useState } from 'react';

export function useIntersectionObserver({
  root = null,
  rootMargin = '0px',
  threshold = 0,
  freezeOnceVisible = false,
} = {}) {
  const [node, setNode] = useState(null); // callback ref → effect re-runs when the node mounts
  const [entry, setEntry] = useState(null);

  const frozen = freezeOnceVisible && entry?.isIntersecting;
  const thresholdKey = Array.isArray(threshold) ? threshold.join(',') : threshold;

  useEffect(() => {
    if (!node || frozen || typeof IntersectionObserver === 'undefined') return;

    const observer = new IntersectionObserver(
      (entries) => setEntry(entries[entries.length - 1]),
      { root, rootMargin, threshold }
    );
    observer.observe(node);
    return () => observer.disconnect();
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [node, frozen, root, rootMargin, thresholdKey]);

  return [setNode, entry];
}
```

### Usage

```jsx
function LazyImage({ src, alt }) {
  const [ref, entry] = useIntersectionObserver({ rootMargin: '200px', freezeOnceVisible: true });
  return (
    <div ref={ref} style={{ minHeight: 200 }}>
      {entry?.isIntersecting && <img src={src} alt={alt} />}
    </div>
  );
}

// infinite scroll sentinel
const [sentinelRef, entry] = useIntersectionObserver();
useEffect(() => {
  if (entry?.isIntersecting && hasMore) loadMore();
}, [entry?.isIntersecting]);
```

### Gotchas / interview follow-ups

- **Why a callback ref (`setNode`) stored in state, not `useRef`?** With `useRef`, the effect runs once with whatever `ref.current` was at that time — if the element mounts later (conditional render) or is replaced, you never observe it. A state-backed callback ref re-runs the effect whenever the DOM node changes.
- **Cleanup = `disconnect()`**; otherwise observers leak.
- **`freezeOnceVisible`** stops observing after the first intersection — ideal for lazy-loading (one-shot).
- **Arrays in dependencies:** `threshold: [0, 0.5, 1]` is a new array each render; derive a primitive key (`thresholdKey`) so the observer isn't re-created every render.
- **`rootMargin`** is a CSS-margin string (`'200px'`) that grows/shrinks the root box — use it to prefetch *before* the element is visible.
- **Much cheaper than a scroll listener + `getBoundingClientRect()`:** the browser computes intersections off the main thread, with no layout thrash.
- jsdom has no `IntersectionObserver` — mock it in tests and guard for it in SSR (`typeof IntersectionObserver === 'undefined'`).

---

## 8. useInterval

`setInterval` that works correctly with React state — the classic Dan Abramov hook. Pass `null` to pause.

### Code

```js
// useInterval.js
import { useEffect, useRef } from 'react';

export function useInterval(callback, delay) {
  const savedCallback = useRef(callback);

  useEffect(() => {
    savedCallback.current = callback; // always the freshest closure
  });

  useEffect(() => {
    if (delay === null) return; // paused
    const id = setInterval(() => savedCallback.current(), delay);
    return () => clearInterval(id);
  }, [delay]);
}
```

### Usage

```jsx
function Counter() {
  const [count, setCount] = useState(0);
  const [running, setRunning] = useState(true);

  useInterval(() => setCount(count + 1), running ? 1000 : null);

  return (
    <>
      <h1>{count}</h1>
      <button onClick={() => setRunning((r) => !r)}>{running ? 'Pause' : 'Resume'}</button>
    </>
  );
}
```

### Gotchas / interview follow-ups

- **Stale closure:** `useEffect(() => { const id = setInterval(() => setCount(count + 1), 1000); ... }, [])` captures `count = 0` forever, so the counter goes `0 → 1` and stops (see Practice Question 2).
- Two valid fixes: (a) functional update `setCount(c => c + 1)` when you only need previous state; (b) the **latest-callback ref** (this hook) when the callback reads arbitrary props/state.
- **Why not put `callback` in the dependency array?** The interval would be torn down and recreated on every render, resetting the timing — with a fast re-render rate the tick may never fire.
- `delay` is the *only* dependency: changing it restarts the interval; `null` pauses (it does **not** reset the elapsed time of the current tick).
- `setInterval` **drifts** and is throttled in background tabs (≥1 s). For accurate timers (stopwatch) compute from `Date.now() - startTime` instead of counting ticks.
- A `useTimeout` is the same pattern with `setTimeout`/`clearTimeout`.

---

## 9. useToggle

Boolean state with stable helpers.

### Code

```js
// useToggle.js
import { useCallback, useState } from 'react';

export function useToggle(initialValue = false) {
  const [on, setOn] = useState(initialValue);

  // toggle() flips; toggle(true/false) sets explicitly
  const toggle = useCallback((next) => {
    setOn((prev) => (typeof next === 'boolean' ? next : !prev));
  }, []);

  return [on, toggle];
}
```

### Usage

```jsx
function Panel() {
  const [open, toggle] = useToggle(false);
  return (
    <>
      <button onClick={toggle}>{open ? 'Hide' : 'Show'}</button>
      <button onClick={() => toggle(false)}>Close</button>
      {open && <div>Content</div>}
    </>
  );
}
```

### Gotchas / interview follow-ups

- **`onClick={toggle}` passes the click event** as `next`. The `typeof next === 'boolean'` check protects against treating the event object as a truthy value.
- **Functional update** (`prev => !prev`) so rapid double-calls flip twice, not once. `setOn(!on)` reads a stale value in the same batch.
- **`useCallback` with `[]`** gives a stable `toggle` identity — safe in dependency arrays and for `React.memo` children.
- Lazy initial state is also supported by `useState`: `useToggle(() => expensive())` works as-is.
- Easy extensions: return `{ on, toggle, setOn, setTrue, setFalse }`, or generalize to `useToggle(['light', 'dark'])` cycling through options.

---

## 10. useEventListener

Attach a DOM event listener declaratively to `window`, `document`, or a ref'd element — with automatic cleanup and a fresh handler.

### Code

```js
// useEventListener.js
import { useEffect, useRef } from 'react';

export function useEventListener(eventName, handler, element, { capture = false, passive } = {}) {
  const handlerRef = useRef(handler);

  useEffect(() => {
    handlerRef.current = handler;
  });

  useEffect(() => {
    // undefined → window; null → nothing; ref → ref.current; otherwise the element itself
    const target =
      element === undefined ? window : element && 'current' in element ? element.current : element;
    if (!target?.addEventListener) return;

    const listener = (event) => handlerRef.current(event);
    target.addEventListener(eventName, listener, { capture, passive });
    return () => target.removeEventListener(eventName, listener, { capture });
  }, [eventName, element, capture, passive]);
}
```

### Usage

```jsx
function KeyLogger() {
  const [key, setKey] = useState('');
  useEventListener('keydown', (e) => setKey(e.key)); // window by default
  return <p>Last key: {key}</p>;
}

function Box() {
  const ref = useRef(null);
  const [hover, setHover] = useState(false);
  useEventListener('mouseenter', () => setHover(true), ref);
  useEventListener('mouseleave', () => setHover(false), ref);
  return <div ref={ref}>{hover ? 'Hovering' : 'Idle'}</div>;
}
```

### Gotchas / interview follow-ups

- **Same latest-ref trick** as `useInterval` / `useOnClickOutside`: the listener is added once per `(eventName, element)`; the handler can change every render without re-subscribing.
- **Removal needs the same function reference** (and the same `capture` flag) — `removeEventListener('x', () => ...)` with a new arrow does nothing. This is the #1 leak.
- **Ref targets:** `ref.current` is `null` during the first render, but effects run after commit, so reading it inside the effect is correct. Don't read `ref.current` while rendering.
- **`passive: true`** for `scroll`/`touchmove`/`wheel` tells the browser you won't call `preventDefault()` — smoother scrolling.
- Alternative to many hooks: pass `{ signal }` from an `AbortController` to `addEventListener` so a single `controller.abort()` removes everything.
- React's own `onClick` props are synthetic events attached at the root; use this hook for `window`/`document` events or when you need capture/passive control.

---

## 11. Practice Questions

**Q1.** A teammate writes this inside a component. Why does the search still fire on every keystroke?

```jsx
function Search() {
  const handle = debounce((q) => fetchResults(q), 300);
  return <input onChange={(e) => handle(e.target.value)} />;
}
```

<details>
<summary><strong>Show Answer</strong></summary>

Every render creates a **new** `debounce` closure with its own private timer. The parent re-renders on every keystroke (e.g. via state), so each call goes through a *different* debounced function and none cancels the previous timer — all of them fire 300 ms later.

Fix: keep one debounced function across renders: `useDebouncedCallback` (stable via `useMemo` + latest-callback ref), or debounce the **value** with `useDebounce` and fetch in an effect.

</details>

**Q2.** What does the counter show over time?

```jsx
function Counter() {
  const [count, setCount] = useState(0);
  useEffect(() => {
    const id = setInterval(() => setCount(count + 1), 1000);
    return () => clearInterval(id);
  }, []);
  return <h1>{count}</h1>;
}
```

<details>
<summary><strong>Show Answer</strong></summary>

It shows `0`, then `1` after one second, and **stays at `1` forever**.

The effect runs once (empty deps), so the interval callback closes over the first render's `count` (`0`). Every tick calls `setCount(0 + 1)`. Fixes: `setCount(c => c + 1)`, or `useInterval` with the latest-callback ref.

</details>

**Q3.** What does `usePrevious` return in each situation?

```jsx
function Demo() {
  const [count, setCount] = useState(0);
  const [other, setOther] = useState(0);
  const prev = usePrevious(count);
  console.log(`count=${count} prev=${prev}`);
  return (
    <>
      <button onClick={() => setCount((c) => c + 1)}>count</button>
      <button onClick={() => setOther((o) => o + 1)}>other</button>
    </>
  );
}
// Sequence: initial render → click "count" → click "other"
```

<details>
<summary><strong>Show Answer</strong></summary>

```
count=0 prev=undefined
count=1 prev=0
count=1 prev=1
```

Clicking `other` re-renders the component, and by then the effect from the previous render already stored `count = 1`. So `usePrevious` is "value at the *previous render*", **not** "previous *different* value". Use `usePreviousDistinct` for the latter.

</details>

**Q4.** In `useFetch`, how do you guarantee the UI never shows data for an old `id` when the user clicks through items quickly? Name two techniques and a pitfall.

<details>
<summary><strong>Show Answer</strong></summary>

1. **AbortController in effect cleanup** — the previous request is cancelled when `url` changes/unmounts; also check `signal.aborted` before `setState`.
2. **Ignore flag** — `let ignore = false; ... if (!ignore) setState(...); return () => { ignore = true; }` (works for non-abortable async work, but the request still runs).

Pitfalls: treating `AbortError` as a real error (flashing an error UI), forgetting `res.ok`, and showing the previous `id`'s data while the new one loads (reset state when the key changes). A request-id counter (`if (id !== latestId.current) return`) is a third option.

</details>

**Q5.** Two components on the same page call `useLocalStorage('theme')`. One changes the theme but the other doesn't update. Why, and how would you fix it?

<details>
<summary><strong>Show Answer</strong></summary>

The `storage` event is fired only in **other** documents (tabs/windows) — never in the one that called `setItem`. Each component holds its own `useState` copy, so nothing notifies the sibling.

Fixes:
- After `setItem`, dispatch a custom event (`window.dispatchEvent(new Event('local-storage'))`) and listen to both `storage` and the custom event in the hook.
- Better: model storage as an external store and read it with `useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot)` — it gives tear-free reads, same-tab and cross-tab sync, and SSR safety in one primitive. (`getSnapshot` must return a cached/stable value — for JSON, return the raw string and parse outside.)
- Or lift the state into Context.

</details>

**Q6.** `useOnClickOutside` is implemented with `click` instead of `mousedown`. A dropdown item removes itself from the list when clicked. What goes wrong, and what else breaks the hook for a modal rendered in a portal?

<details>
<summary><strong>Show Answer</strong></summary>

React processes the item's `onClick`, re-renders and removes the item from the DOM **before** the `document` `click` listener runs (listener runs in the bubble phase after React's root handler). `ref.current.contains(event.target)` is now `false` for a detached node, so the click is treated as *outside* and the dropdown closes unexpectedly. `mousedown`/`pointerdown` fire before the DOM changes.

Portals: the portal content is outside `ref.current` in the DOM tree (though inside it in the React tree), so clicks in it count as outside. Handle with multiple refs, `event.composedPath()`, or a `data-` attribute check.

</details>
