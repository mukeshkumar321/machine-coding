# Toast Notification — Machine Coding

> **Difficulty:** Medium | **Time:** 45–60 min | **Frequency:** ⭐⭐⭐⭐

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

Build a **toast notification system** for a React app. Any component should be able to trigger a toast with a single call:

```jsx
const toast = useToast();
toast.success('Saved!');
toast.error('Something went wrong', { duration: 8000 });
```

Toasts stack in a corner of the screen, disappear automatically after a timeout, and stop the countdown while the user hovers over (or focuses) them. If too many toasts are fired at once, only a limited number are visible and the rest wait in a queue.

---

## 2. Requirements

### Functional

- `ToastProvider` + `useToast()` hook exposing `show`, `success`, `error`, `info`, `dismiss`, `clear`.
- Types: `success | error | info` (different colour/icon).
- **Auto-dismiss** after `duration` ms (default 4000). `duration: null` = sticky.
- **Pause on hover** (and on keyboard focus); resume with the *remaining* time, not a full restart.
- Manual close button.
- **Stacking**: newest toast at the bottom (or top, configurable), consistent gap.
- **Queue**: at most `MAX_VISIBLE` (3) on screen; extra toasts are queued and appear when a slot frees up.
- Rendered through a **portal** to `document.body` so z-index / overflow of parents never clips it.

### Non-functional

- Calling `useToast()` consumers must **not re-render** when toasts change (stable API object).
- No leaked timers after unmount / dismiss.
- Accessible: live region, keyboard-closable, respects `prefers-reduced-motion`.
- Plain CSS, no libraries.

---

## 3. Clarifying Questions to Ask

- Max number visible at once? Queue the rest or drop the oldest?
- Position — fixed corner, or configurable (top-right, bottom-center)?
- Should the timer pause on hover? On window blur / tab hidden too?
- Do we need enter/exit animations? Action buttons ("Undo") inside toasts?
- Dedupe identical messages?
- Do we need to trigger toasts **outside React** (e.g. from an axios interceptor)?
- Sticky/persistent toasts (errors)?

---

## 4. Approach & Component Design

**Key insight:** keep the *list* of toasts in the provider (reducer), but let each `Toast` own its *own timer*. A queued toast isn't rendered, so its timer doesn't start until it becomes visible — the queue falls out of `toasts.slice(0, MAX_VISIBLE)` for free.

**State shape**

```js
// provider state: array in arrival order
[{ id: 1, type: 'success', message: 'Saved', duration: 4000 }, ...]
```

**Component tree**

```
<ToastProvider>                 owns reducer state, exposes stable api via context
  {children}                    <- consumers call useToast() here
  createPortal(
    <ToastContainer>            role="region", aria-live="polite", fixed position
      <Toast />  x visible      owns timer + pause state
    </ToastContainer>, document.body)
</ToastProvider>
```

| Concern | Where it lives | Why |
| --- | --- | --- |
| List of toasts | `useReducer` in provider | Predictable add/remove |
| Public API | `useMemo` context value | Stable identity → no consumer re-renders |
| Timer / remaining time | `Toast` (`useRef` + effect) | Per-toast, auto-cleanup on unmount |
| Queue | `slice(0, MAX_VISIBLE)` | Hidden toasts have no timer |
| Hover/focus pause | `Toast` state `paused` | Effect re-runs on change |

---

## 5. Reference Solution

<details>
<summary><strong>Show Full Code</strong></summary>

**`ToastContext.jsx`**

```jsx
import {
  createContext,
  useCallback,
  useContext,
  useEffect,
  useMemo,
  useReducer,
  useRef,
  useState,
} from 'react';
import { createPortal } from 'react-dom';
import './toast.css';

const MAX_VISIBLE = 3;
const DEFAULT_DURATION = 4000;

const ToastContext = createContext(null);

function toastReducer(state, action) {
  switch (action.type) {
    case 'ADD':
      return [...state, action.toast];
    case 'REMOVE':
      return state.filter((t) => t.id !== action.id);
    case 'CLEAR':
      return [];
    default:
      return state;
  }
}

export function ToastProvider({ children }) {
  const [toasts, dispatch] = useReducer(toastReducer, []);
  const nextId = useRef(1);

  const dismiss = useCallback((id) => dispatch({ type: 'REMOVE', id }), []);

  const show = useCallback((message, options = {}) => {
    const id = nextId.current++;
    dispatch({
      type: 'ADD',
      toast: {
        id,
        message,
        type: options.type ?? 'info',
        duration:
          options.duration === undefined ? DEFAULT_DURATION : options.duration,
      },
    });
    return id;
  }, []);

  // Stable identity: consumers never re-render when the toast list changes.
  const api = useMemo(
    () => ({
      show,
      dismiss,
      clear: () => dispatch({ type: 'CLEAR' }),
      success: (msg, opts) => show(msg, { ...opts, type: 'success' }),
      error: (msg, opts) => show(msg, { ...opts, type: 'error' }),
      info: (msg, opts) => show(msg, { ...opts, type: 'info' }),
    }),
    [show, dismiss]
  );

  const visible = toasts.slice(0, MAX_VISIBLE);

  return (
    <ToastContext.Provider value={api}>
      {children}
      {createPortal(
        <div
          className="toast-container"
          role="region"
          aria-label="Notifications"
          aria-live="polite"
        >
          {visible.map((t) => (
            <Toast key={t.id} toast={t} onDismiss={dismiss} />
          ))}
        </div>,
        document.body
      )}
    </ToastContext.Provider>
  );
}

export function useToast() {
  const ctx = useContext(ToastContext);
  if (!ctx) throw new Error('useToast must be used inside <ToastProvider>');
  return ctx;
}

const ICONS = { success: '✓', error: '!', info: 'i' };

function Toast({ toast, onDismiss }) {
  const { id, message, type, duration } = toast;
  const [hovered, setHovered] = useState(false);
  const [focused, setFocused] = useState(false);
  const paused = hovered || focused;
  const remaining = useRef(duration);

  useEffect(() => {
    if (duration == null || paused) return;
    const startedAt = Date.now();
    const timer = setTimeout(() => onDismiss(id), remaining.current);
    return () => {
      clearTimeout(timer);
      // Remember how much time is left so resume doesn't restart the countdown.
      remaining.current = Math.max(0, remaining.current - (Date.now() - startedAt));
    };
  }, [paused, duration, id, onDismiss]);

  return (
    <div
      className={`toast toast--${type}`}
      role={type === 'error' ? 'alert' : 'status'}
      onMouseEnter={() => setHovered(true)}
      onMouseLeave={() => setHovered(false)}
      onFocus={() => setFocused(true)}
      onBlur={(e) => {
        if (!e.currentTarget.contains(e.relatedTarget)) setFocused(false);
      }}
    >
      <span className="toast__icon" aria-hidden="true">
        {ICONS[type]}
      </span>
      <p className="toast__message">{message}</p>
      <button
        type="button"
        className="toast__close"
        aria-label="Dismiss notification"
        onClick={() => onDismiss(id)}
      >
        ×
      </button>
      {duration != null && (
        <span
          className={`toast__bar${paused ? ' toast__bar--paused' : ''}`}
          style={{ animationDuration: `${duration}ms` }}
          aria-hidden="true"
        />
      )}
    </div>
  );
}
```

**`App.jsx`**

```jsx
import { ToastProvider, useToast } from './ToastContext';

function Demo() {
  const toast = useToast();
  return (
    <div style={{ padding: 24, display: 'flex', gap: 8, flexWrap: 'wrap' }}>
      <button onClick={() => toast.success('Saved successfully')}>Success</button>
      <button onClick={() => toast.error('Request failed', { duration: 8000 })}>
        Error (8s)
      </button>
      <button onClick={() => toast.info('Sticky info', { duration: null })}>
        Sticky
      </button>
      <button
        onClick={() => {
          for (let i = 1; i <= 6; i++) toast.info(`Toast #${i}`);
        }}
      >
        Fire 6 (queue)
      </button>
      <button onClick={toast.clear}>Clear all</button>
    </div>
  );
}

export default function App() {
  return (
    <ToastProvider>
      <Demo />
    </ToastProvider>
  );
}
```

**`toast.css`**

```css
.toast-container {
  position: fixed;
  top: 16px;
  right: 16px;
  z-index: 1000;
  display: flex;
  flex-direction: column;
  gap: 8px;
  width: min(360px, calc(100vw - 32px));
  pointer-events: none; /* container itself never blocks clicks */
}

.toast {
  --accent: #2563eb;
  position: relative;
  overflow: hidden;
  display: flex;
  align-items: flex-start;
  gap: 10px;
  padding: 12px 12px 14px;
  background: #fff;
  border-left: 4px solid var(--accent);
  border-radius: 6px;
  box-shadow: 0 6px 20px rgba(0, 0, 0, 0.18);
  pointer-events: auto;
  animation: toast-in 200ms ease-out;
}

.toast--success { --accent: #16a34a; }
.toast--error   { --accent: #dc2626; }

.toast__icon {
  flex: none;
  width: 22px;
  height: 22px;
  border-radius: 50%;
  background: var(--accent);
  color: #fff;
  font-size: 13px;
  font-weight: 700;
  display: grid;
  place-items: center;
}

.toast__message {
  flex: 1;
  margin: 0;
  font-size: 14px;
  line-height: 22px;
  word-break: break-word;
}

.toast__close {
  flex: none;
  border: 0;
  background: transparent;
  font-size: 20px;
  line-height: 1;
  cursor: pointer;
  color: #555;
}

.toast__bar {
  position: absolute;
  left: 0;
  bottom: 0;
  height: 3px;
  width: 100%;
  background: var(--accent);
  transform-origin: left;
  animation-name: toast-progress;
  animation-timing-function: linear;
  animation-fill-mode: forwards;
}

.toast__bar--paused {
  animation-play-state: paused;
}

@keyframes toast-in {
  from { opacity: 0; transform: translateX(24px); }
  to   { opacity: 1; transform: translateX(0); }
}

@keyframes toast-progress {
  from { transform: scaleX(1); }
  to   { transform: scaleX(0); }
}

@media (prefers-reduced-motion: reduce) {
  .toast { animation: none; }
}
```

</details>

---

## 6. How It Works

- **Provider owns data, `Toast` owns time.** The reducer only does add/remove. Each `Toast` runs its own `setTimeout` inside an effect, so cleanup on dismiss/unmount is automatic.
- **Queue = slice.** `toasts.slice(0, MAX_VISIBLE)` renders only the first N. When one is removed, the next one mounts and *then* starts its timer — queued toasts never silently expire.
- **Pause/resume with remaining time.** The effect depends on `paused`. When `paused` flips to `true`, the cleanup clears the timer and subtracts the elapsed time from `remaining` (a ref — it must survive re-renders without causing one). When it flips back, a new timer is started with what's left.
- **Stable API.** `api` is memoized with `useCallback` dispatchers, so components calling `useToast()` don't re-render on each toast. If we had put `toasts` in the same context, every consumer would re-render.
- **IDs from a ref counter**, generated outside the reducer so the reducer stays pure (Strict Mode double-invokes reducers).
- **Portal** mounts the container on `document.body`, escaping `overflow: hidden`/`transform` ancestors.
- **Progress bar** is pure CSS (`transform: scaleX`) and paused with `animation-play-state` — no JS per frame.

---

## 7. Edge Cases

- Dismiss clicked at the same moment the timer fires → `REMOVE` is idempotent (filter on a missing id is a no-op).
- Pointer is already over the toast when it mounts (a new toast appears under a stationary cursor): browsers usually fire `mouseenter` on the next pointer move, which then pauses it — acceptable; mention it.
- Toast removed while hovered → component unmounts, cleanup clears timer.
- Many toasts fired in a loop → unique ids from the ref counter; the queue handles overflow.
- Only `duration: null` is sticky. `duration: 0` is *not* sticky — `setTimeout(…, 0)` dismisses it immediately; decide whether you want `0` to mean sticky instead and document it.
- React 18 Strict Mode: effect runs → cleanup → runs again; elapsed ≈ 0 so `remaining` is essentially unchanged.
- Tab hidden: `setTimeout` is throttled but still fires; optionally pause on `visibilitychange`.
- Message very long → `word-break` + max-width.
- SSR: `document.body` doesn't exist on the server — guard or mount portal after `useEffect`.

---

## 8. Accessibility

- Container is a **live region** (`aria-live="polite"`); individual toasts use `role="status"`, errors use `role="alert"` (assertive). Nesting live regions can make some screen readers announce twice — if that shows up in testing, keep the roles on the toasts and drop `aria-live` from the container (or the reverse).
- The live region must exist **before** content is injected — that's why the container is always rendered.
- Pause on **keyboard focus** as well as hover (WCAG 2.2.1 Timing Adjustable) — users need enough time to read/act.
- Close button has an `aria-label`; decorative icon has `aria-hidden`.
- Don't put critical, must-act information in an auto-dismissing toast; use sticky or a dialog.
- `prefers-reduced-motion` disables slide-in.

---

## 9. Performance Considerations

- Memoized context value → consumers don't re-render on list changes.
- Only ≤ `MAX_VISIBLE` DOM nodes and timers exist at any time.
- Progress animation uses `transform` on the compositor, not `width`.
- Each `Toast` re-renders on its own hover/focus state and, because it isn't memoized, whenever the provider re-renders (add/remove).
- If toasts carried heavy content, wrap `Toast` in `React.memo` (props: `toast` object is stable per id, `onDismiss` is stable).

---

## 10. Follow-up Questions

<details>
<summary><strong>1. How would you add exit animations?</strong></summary>

Don't remove immediately. Mark the toast `closing: true` (new action), play a CSS `toast-out` animation, then remove on `onAnimationEnd`. Or use `react-transition-group`'s `TransitionGroup`/`CSSTransition`.

</details>

<details>
<summary><strong>2. How do you trigger a toast from outside React (e.g. an axios interceptor)?</strong></summary>

Expose a module-level store/event emitter: `toastStore.show(...)` pushes into a store, and the provider subscribes with `useSyncExternalStore`. Alternatively register the provider's `show` in a module variable via an effect (`setToastRef(api)`).

</details>

<details>
<summary><strong>3. How would you support different positions?</strong></summary>

Group toasts by `position` and render one container per position, each with its own modifier class (`.toast-container--bottom-center`). For bottom positions anchor the container with `bottom` instead of `top`; choose `column` vs `column-reverse` depending on whether the newest toast should sit nearest the screen edge.

</details>

<details>
<summary><strong>4. How would you dedupe identical toasts?</strong></summary>

In `show`, check for an existing toast with the same `message`+`type`; either ignore, or reset its timer by changing a `key`/`version` field so the timer effect restarts. A `toastId` option (like react-toastify) is the cleanest API.

</details>

<details>
<summary><strong>5. Why not keep the timers in the provider?</strong></summary>

Then you'd have to manage a `Map<id, timerId>` in a ref, clear them on remove, track remaining time and pause state per id, and handle queued ones. Putting the timer in the component ties its lifetime to the DOM node and makes cleanup automatic.

</details>

<details>
<summary><strong>6. How would you add an "Undo" action button?</strong></summary>

Accept `action: { label, onClick }` in options; render a button in the toast that calls `onClick` then `dismiss(id)`. Give such toasts a longer duration, and make sure the button is reachable by keyboard (focus pauses the timer).

</details>

<details>
<summary><strong>7. How do you test it?</strong></summary>

Use fake timers (`jest.useFakeTimers()` / `vi.useFakeTimers()`), render a component calling `toast.success`, advance time by `duration`, and assert removal. For pause: `userEvent.hover`, advance time, assert still present; unhover, advance the remaining time, assert gone. Queue: fire 5, assert 3 in DOM, dismiss one, assert 4th appears.

</details>

<details>
<summary><strong>8. What if the number of toasts must be unlimited but old ones scrolled?</strong></summary>

Remove the slice, make the container `max-height: 100vh; overflow-y: auto`, and keep newest at the end. Consider capping the array length in the reducer to prevent unbounded growth.

</details>

---

## 11. Common Mistakes

- Putting `toasts` in the same context as the dispatchers → every consumer re-renders.
- Starting timers in the provider for queued toasts → they expire before being seen.
- Restarting the full duration after hover instead of resuming with remaining time.
- `setTimeout` without cleanup → state update/dismiss for an unmounted toast.
- Using array index as `key` → wrong toast gets dismissed/animated. Use a unique id.
- Generating ids with `Date.now()` → collisions for toasts fired in the same millisecond.
- Mutating state (`state.push`) in the reducer.
- Stale closure: capturing `toasts` in `show` to compute ids/limits instead of using a ref/reducer.
- Rendering inline instead of a portal → clipped by parent `overflow`.
- Forgetting `pointer-events: none` on the container → invisible area blocks clicks on the page.

