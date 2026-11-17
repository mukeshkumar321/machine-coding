# Like Button with Optimistic Update — Machine Coding

> **Difficulty:** Medium | **Time:** 30–45 min | **Asked at / Frequency:** Common (Meta/Instagram-style, Twitter/X, LinkedIn, Dream11, ShareChat) — tests async reasoning | **Tier:** 1 (must-know)

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

Build a **Like button** with a like count (like Instagram/Twitter). Clicking it must feel **instant**: the heart fills and the count changes immediately, *before* the server responds (**optimistic update**). If the request fails, **roll back** to the last known-good state and show an error. Users may click rapidly (double-click, spam) — the final state must always match what the server ends up with, and no stale response should overwrite a newer one.

You are given a mock API with latency and random failures.

---

## 2. Requirements

### Functional

- Toggle liked/unliked; count ± 1 instantly.
- Call the API (`PUT liked=true|false`) in the background.
- On success, reconcile with the server's returned `{ liked, count }`.
- On failure, **rollback** to the last server-confirmed state and show an error message.
- Rapid clicks: UI reacts to every click immediately; network requests are **never concurrent for the same item**; final server state == final UI state (or a rollback to confirmed).
- Do **not** disable the button while pending (that defeats "optimistic").

### Non-functional

- No lost updates, no out-of-order responses overwriting newer state.
- No state updates after unmount.
- Logic reusable: `useOptimisticLike` hook separate from the presentational button.

---

## 3. Clarifying Questions to Ask

- Is the API idempotent "set liked = X" (PUT) or a non-idempotent "toggle" (POST)? *(Idempotent is far safer for retries/races.)*
- Is the count authoritative from the server (other users like too)?
- On failure: silent rollback, toast, or retry with backoff?
- On rapid clicks: queue every click, **coalesce** to the latest intent, or debounce?
- Do we show a pending indicator?
- Should the state survive navigation (cache/store) — or is it local to the component?
- Offline behaviour?

---

## 4. Approach & Component Design

**Two sources of truth, deliberately separate:**

| Name | Lives in | Meaning |
| --- | --- | --- |
| `state` `{liked, count}` | `useState` | What the user **sees** (optimistic) |
| `confirmed` `{liked, count}` | `useRef` | Last state the **server acknowledged** (rollback target) |
| `desired` `boolean` | `useRef` | What the user **wants** right now (last click) |
| `inFlight` `boolean` | `useRef` | Is there a request running? |

Refs are used for values the async loop must read *fresh* and that don't need to trigger a render — this sidesteps stale closures.

**Algorithm — "serialize and coalesce":**

```
click:   desired = !desired;  UI flips instantly;  sync()
sync():  if inFlight → return            (the running loop will notice the new desire)
         inFlight = true
         while desired != confirmed.liked:
             result = await api(desired)   // one request at a time
             confirmed = result
         reconcile UI with confirmed
         on error: desired = confirmed.liked; UI = confirmed; show error
         inFlight = false
```

**Timeline (double-click while request #1 is pending, server initially `false`)**

```
t0  click  → UI true  (129)    desired=true   request(true) starts
t1  click  → UI false (128)    desired=false  (no new request: one is in flight)
t2  req#1 resolves → confirmed {true,129}
t3  loop sees desired(false) != confirmed(true) → request(false)
t4  req#2 resolves → confirmed {false,128} → UI reconciled
```

**Tree**

```
App
 └─ LikeButton   uses useOptimisticLike({ initialLiked, initialCount, request })
```

---

## 5. Reference Solution

<details>
<summary><strong>Show Full Code</strong></summary>

**`mockApi.js`**

```js
let serverState = { liked: false, count: 128 };

// Mutable so the demo UI can change it.
export const apiConfig = { latency: 800, failureRate: 0.3 };

// Idempotent "set" semantics: sending the same value twice is harmless.
export function setLikeOnServer(liked) {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      if (Math.random() < apiConfig.failureRate) {
        reject(new Error('Network error'));
        return;
      }
      if (liked !== serverState.liked) {
        serverState = { liked, count: serverState.count + (liked ? 1 : -1) };
      }
      resolve({ ...serverState });
    }, apiConfig.latency);
  });
}
```

**`useOptimisticLike.js`**

```js
import { useCallback, useEffect, useRef, useState } from 'react';

export function useOptimisticLike({ initialLiked, initialCount, request }) {
  const [state, setState] = useState({ liked: initialLiked, count: initialCount });
  const [error, setError] = useState('');
  const [isSyncing, setIsSyncing] = useState(false);

  const confirmed = useRef({ liked: initialLiked, count: initialCount }); // server truth
  const desired = useRef(initialLiked); // what the user wants right now
  const inFlight = useRef(false);
  const mounted = useRef(true);
  const requestRef = useRef(request);

  useEffect(() => {
    requestRef.current = request; // always call the latest function, never a stale one
  });

  useEffect(() => {
    mounted.current = true;
    return () => {
      mounted.current = false;
    };
  }, []);

  const sync = useCallback(async () => {
    if (inFlight.current) return; // the running loop will pick up the new `desired`
    inFlight.current = true;
    setIsSyncing(true);

    try {
      while (desired.current !== confirmed.current.liked) {
        const result = await requestRef.current(desired.current);
        confirmed.current = { liked: result.liked, count: result.count };
      }
      // Loop only exits when desired === confirmed, so the UI agrees on `liked`;
      // this just adopts the server's authoritative count.
      if (mounted.current) setState({ ...confirmed.current });
    } catch {
      // Roll back to the last state the server acknowledged — NOT to "the previous click".
      desired.current = confirmed.current.liked;
      if (mounted.current) {
        setState({ ...confirmed.current });
        setError('Could not update your like. Please try again.');
      }
    } finally {
      inFlight.current = false;
      if (mounted.current) setIsSyncing(false);
    }
  }, []);

  const toggle = useCallback(() => {
    setError('');
    desired.current = !desired.current;
    // Optimistic update: pure functional updater so rapid clicks never lose updates.
    setState((s) => ({ liked: !s.liked, count: s.count + (s.liked ? -1 : 1) }));
    sync();
  }, [sync]);

  return { liked: state.liked, count: state.count, error, isSyncing, toggle };
}
```

**`LikeButton.jsx`**

```jsx
import { useOptimisticLike } from './useOptimisticLike';
import { setLikeOnServer } from './mockApi';
import './like.css';

function Heart({ filled }) {
  return (
    <svg viewBox="0 0 24 24" width="24" height="24" aria-hidden="true" focusable="false">
      <path
        d="M20.84 4.61a5.5 5.5 0 0 0-7.78 0L12 5.67l-1.06-1.06a5.5 5.5 0 0 0-7.78 7.78l1.06 1.06L12 21.23l7.78-7.78 1.06-1.06a5.5 5.5 0 0 0 0-7.78z"
        fill={filled ? 'currentColor' : 'none'}
        stroke="currentColor"
        strokeWidth="2"
        strokeLinejoin="round"
      />
    </svg>
  );
}

export default function LikeButton({ initialLiked = false, initialCount = 128 }) {
  const { liked, count, error, isSyncing, toggle } = useOptimisticLike({
    initialLiked,
    initialCount,
    request: setLikeOnServer,
  });

  return (
    <div className="like-wrap">
      <button
        type="button"
        className={`like${liked ? ' like--on' : ''}`}
        aria-pressed={liked}
        aria-busy={isSyncing}
        onClick={toggle}
      >
        <Heart filled={liked} />
        <span className="sr-only">Like. </span>
        <span className="like__count">{count}</span>
      </button>
      <span className="like__status" aria-live="polite">
        {isSyncing ? 'Saving…' : ''}
      </span>
      {error && (
        <p role="alert" className="like__error">
          {error}
        </p>
      )}
    </div>
  );
}
```

**`App.jsx`**

```jsx
import { useState } from 'react';
import LikeButton from './LikeButton';
import { apiConfig } from './mockApi';

export default function App() {
  const [failures, setFailures] = useState(true);

  return (
    <main style={{ fontFamily: 'system-ui', padding: 24 }}>
      <h1>Post</h1>
      <LikeButton />
      <label style={{ display: 'block', marginTop: 16 }}>
        <input
          type="checkbox"
          checked={failures}
          onChange={(e) => {
            setFailures(e.target.checked);
            apiConfig.failureRate = e.target.checked ? 0.5 : 0;
          }}
        />{' '}
        Simulate random API failures
      </label>
    </main>
  );
}
```

**`like.css`**

```css
.like-wrap { display: flex; align-items: center; gap: 12px; }

.like {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 8px 14px;
  border: 1px solid #cbd5e1;
  border-radius: 999px;
  background: #fff;
  color: #475569;
  font-size: 16px;
  cursor: pointer;
  transition: color 150ms, border-color 150ms;
}
.like:focus-visible { outline: 3px solid #93c5fd; outline-offset: 2px; }

.like--on { color: #e11d48; border-color: #fda4af; }
.like--on svg { animation: pop 300ms ease-out; }

.like__count { min-width: 3ch; font-variant-numeric: tabular-nums; }
.like__status { font-size: 12px; color: #64748b; min-width: 60px; }
.like__error { margin: 0; color: #dc2626; font-size: 14px; }

.sr-only {
  position: absolute;
  width: 1px; height: 1px;
  overflow: hidden;
  clip: rect(0 0 0 0);
  white-space: nowrap;
}

@keyframes pop {
  0%   { transform: scale(1); }
  40%  { transform: scale(1.35); }
  100% { transform: scale(1); }
}

@media (prefers-reduced-motion: reduce) {
  .like--on svg { animation: none; }
}
```

</details>

---

## 6. How It Works

- **Instant feedback:** `toggle` flips UI state synchronously with a *functional* updater (`setState(s => …)`), so ten rapid clicks produce ten correct flips — never `setState({count: count + 1})` with a captured value.
- **Intent vs. truth:** `desired` (what the user wants) and `confirmed` (what the server acknowledged) are separate refs. The sync loop's only job is to make `confirmed` equal `desired`.
- **One request at a time.** `inFlight` guards re-entry. While a request is pending, extra clicks only update `desired` and the UI. When the response arrives, the `while` condition re-checks `desired` and, if it changed, sends *one* more request with the **latest** intent. Ten clicks → at most a couple of requests (**coalescing**), and out-of-order responses are impossible because requests are serialized.
- **Idempotent API.** Sending `liked=true` is a PUT-style set, not a toggle: a retry or duplicate can't flip the state accidentally, and the server returns the authoritative count.
- **Rollback target = `confirmed`**, not "previous UI state". If click 1 succeeded and click 2 failed, you roll back to the state after click 1 — exactly what the server has. Rolling back to a captured "previous" value breaks with multiple in-flight clicks.
- **Refs for async code.** The loop awaits across renders; reading `desired.current` / `confirmed.current` always gives the live value (no stale closure), and writing to them doesn't re-render.
- **`requestRef`** keeps the hook's identity-stable `sync` while still calling the newest `request` function.
- **`mounted` guard** avoids setting state after unmount; the in-flight request finishes harmlessly. (Use `AbortController` if your API supports it.)
- **Reconcile on success:** adopt the server's count (other users may have liked too) only when no newer click is pending — guaranteed because the loop exits only when `desired === confirmed.liked`.

---

## 7. Edge Cases

- **Double-click while pending** → handled by coalescing; verify the timeline in section 4.
- **Click → click → click (odd/even)**: the first click starts a request; later clicks only update `desired`. When that request resolves, an extra request is sent only if the final intent differs from what was just confirmed (an even number of clicks ends where it started, so the follow-up request restores it; an odd number matches the in-flight request, so no extra request is needed).
- **Failure on the first request** → rollback to initial; user sees the error; next click works.
- **Failure on a follow-up request** → rollback to the *last confirmed* state (which may differ from the pre-click state).
- **Error message** cleared on the next click.
- **Unmount mid-flight** → no state updates; loop ends naturally.
- **Strict Mode** double-invoking effects → `mounted` is reset to `true` in the effect body, so the guard isn't left `false`.
- **Offline** → request rejects → rollback; mention queueing + retry on `online` event as an extension.
- **Server count changes meanwhile** → we adopt the server's count when the loop finishes.
- **Multiple LikeButtons for the same post** (feed + modal) → local refs can't coordinate them; lift into a shared store/cache (React Query, Zustand).
- **Initial props change** (navigating to another post with the same component instance) → remount with `key={postId}`.

---

## 8. Accessibility

- A toggle button uses `aria-pressed` (true/false) — **don't also change the label** to "Unlike"; that creates "Unlike, pressed" confusion.
- The accessible name includes visible text "Like" plus the count via a visually-hidden span; the icon is `aria-hidden`.
- Error uses `role="alert"` (announced immediately); the saving status uses `aria-live="polite"`.
- Keep the button **enabled** while pending (don't trap keyboard users with a disabled-then-reenabled focus loss); `aria-busy` communicates the background work.
- Visible `:focus-visible` outline; ≥ 44px touch target in real designs.
- Respect `prefers-reduced-motion` for the pop animation.

---

## 9. Performance Considerations

- State updates are tiny: one object with two fields; only the button re-renders.
- Coalescing reduces network chatter: spam-clicking 20 times during one request results in 2 requests (the first, plus one follow-up if the intent changed), not 20.
- Refs avoid extra renders for bookkeeping (`inFlight`, `desired`, `confirmed`).
- In a feed of many posts, each `LikeButton` has its own hook instance — memoize the post list items; keep the optimistic state local so a like doesn't re-render the feed.
- Debouncing the request (e.g. 300 ms) is an alternative to serialization that further cuts traffic, but delays persistence and complicates failure rollback.

---

## 10. Follow-up Questions

<details>
<summary><strong>1. Why not just disable the button while the request is pending?</strong></summary>

It removes the race, but it also removes the *instant* feel — the whole point of optimistic UI. Users on slow networks would see a dead button. Serialize/coalesce instead.

</details>

<details>
<summary><strong>2. Alternative to serialization: request IDs ("latest wins")?</strong></summary>

Fire every request but tag each with an incrementing `requestId`; on response, ignore it if `requestId !== latestId`. This avoids stale UI writes, but the server may still process requests out of order (the *server* might end up with the older value). Serialization guarantees order end-to-end; or send a version/sequence number with each request so the server can discard stale ones.

</details>

<details>
<summary><strong>3. How would you do this with React Query?</strong></summary>

`useMutation` with `onMutate` (cancel queries via `queryClient.cancelQueries`, snapshot the previous cache, `setQueryData` optimistically, return the snapshot), `onError` (restore the snapshot), `onSettled` (`invalidateQueries` to refetch the truth). Same ideas: snapshot → optimistic write → rollback → reconcile.

</details>

<details>
<summary><strong>4. How does React 19's `useOptimistic` fit?</strong></summary>

`const [optimistic, addOptimistic] = useOptimistic(serverState, reducer)` inside a transition/action: the optimistic value is shown while the action is pending and automatically reverts to `serverState` when it settles (success or failure). It replaces the manual snapshot/rollback, but you still need to think about rapid-click ordering and surfacing errors.

</details>

<details>
<summary><strong>5. How do you implement retry with exponential backoff?</strong></summary>

Wrap the request: loop up to N attempts, `await sleep(base * 2 ** attempt + jitter)`, only retry on network/5xx errors (not 4xx). Because the API is idempotent, retries are safe. Roll back only after retries are exhausted; show a "Retrying…" state in between.

</details>

<details>
<summary><strong>6. What if the API were a non-idempotent toggle (`POST /toggle`)?</strong></summary>

Races become dangerous (duplicate request flips back; retry after a timeout may double-toggle). Prefer asking for a PUT/`liked` flag or an idempotency key. If you can't, you must serialize strictly and never auto-retry without reading the server state first.

</details>

<details>
<summary><strong>7. How would you cancel the in-flight request on unmount?</strong></summary>

Create an `AbortController` per request in the loop, pass `signal` to `fetch`, and call `abort()` in the unmount cleanup. Treat `AbortError` as non-error (don't roll back or show a message).

</details>

<details>
<summary><strong>8. How would you keep two like buttons of the same post in sync?</strong></summary>

Move `{liked, count}` into a shared store keyed by `postId` (Context + reducer, Zustand, or a React Query cache) and have the hook read/write it with `useSyncExternalStore`. The `confirmed`/`desired`/`inFlight` bookkeeping should live in the store (per post) too.

</details>

---

## 11. Common Mistakes

- `setCount(count + 1)` with a captured `count` → lost updates on rapid clicks; use functional updates.
- Rolling back to a captured "previous" value instead of the last **server-confirmed** state.
- Firing a request per click and applying responses in arrival order → stale response overwrites newer UI.
- Disabling the button during the request (not optimistic).
- Using non-idempotent toggle semantics with retries.
- Reading state inside the async loop via closure (`liked`) → stale; use refs.
- Setting state after unmount; forgetting the mounted guard in Strict Mode (setting `mounted=false` once and never back to `true`).
- No error feedback: the UI silently snaps back.
- Replacing the UI count with the server's count in the middle of a burst of clicks (flicker/back-jump).
- Mutating the state object (`state.count++`).
- Changing the label to "Unlike" *and* using `aria-pressed`.

---

## 12. Interview Tips & Time Plan

Narrate the invariant: *"UI shows intent; the server shows truth; one in-flight request at a time; rollback goes to the last confirmed state."* Draw the double-click timeline — it shows you've thought about races. Build the naive version first (optimistic + rollback), then explicitly add race handling.

| Minute | Activity |
| --- | --- |
| 0–4 | Clarify (idempotent API? failure UX? rapid clicks?) |
| 4–8 | State design: `state` vs `confirmed` vs `desired`, timeline sketch |
| 8–14 | Mock API (latency + random fail) |
| 14–24 | Naive optimistic toggle + rollback in a hook |
| 24–32 | Add `inFlight` serialization + coalescing loop |
| 32–38 | Button UI, CSS, aria-pressed, error + status |
| 38–45 | Edge cases (unmount, Strict Mode), follow-ups |

---

## 13. Practice Checklist

- [ ] Optimistic toggle with a functional state update
- [ ] Mock API with latency and random failures
- [ ] Rollback to last confirmed state; show an error
- [ ] Explain why the button must not be disabled
- [ ] Serialize requests with a ref flag; coalesce to the latest intent
- [ ] Draw and explain the double-click timeline
- [ ] Guard against unmount (and understand Strict Mode)
- [ ] Add retry with backoff for an idempotent API
- [ ] Implement the "latest request id wins" variant and compare trade-offs
- [ ] Rebuild with React Query `onMutate`/`onError`/`onSettled`
- [ ] Rebuild with React 19 `useOptimistic`
- [ ] Share state between two buttons via a store
