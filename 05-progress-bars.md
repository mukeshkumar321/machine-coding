# Progress Bars — Machine Coding

> **Difficulty:** Easy–Medium | **Time:** 30–45 min | **Frequency:** ⭐⭐⭐⭐

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

Build a page with an **"Add"** button. Every click adds a new progress bar that fills from 0 → 100% in 2 seconds.

- **Level 1:** bars fill **one at a time** (sequentially). A bar only starts when the previous one has finished.
- **Level 2:** bars may fill **concurrently**, but at most **N** at the same time (max concurrency). Extra bars wait in a queue.

---

## 2. Requirements

### Functional

- "Add" appends a bar; bar shows queued / running / done state.
- Bars start in FIFO order; at most `maxConcurrency` bars are running.
- When a bar finishes, the next queued bar starts automatically.
- Concurrency limit is configurable (1 = fully sequential).
- Reset button clears everything.

### Non-functional

- Smooth animation (no jank, GPU-friendly).
- No `setInterval` leaks; clicking Add 50 times quickly must work.
- Scheduling logic must not depend on a fragile chain of effects/timeouts.

---

## 3. Clarifying Questions to Ask

- Is the fill duration fixed (2s) or does it represent real work (upload progress)?
- Sequential only, or configurable concurrency? Can the limit change at runtime?
- Should the bar display a percentage number?
- Should a finished bar stay or be removed?
- Pause/cancel individual bars?
- Is a CSS transition acceptable, or must I drive progress from JS (rAF)?

---

## 4. Approach & Component Design

**Key insight:** don't store `status: 'queued' | 'running'` and schedule it with effects. Store only `{ id, done }` and **derive** which bars are *active* — the first `N` bars that aren't done. Completion of a bar (`transitionend`) flips `done`; the derived set automatically promotes the next bars. No effect chains, no race conditions.

**State shape**

```js
bars: [{ id: 1, done: true }, { id: 2, done: false }, { id: 3, done: false }]
maxConcurrency: 2
// derived: activeIds = first 2 bars where !done  -> {2, 3}
```

**Component tree**

```
App
 ├─ controls: [Add] [Max concurrency select] [Reset]
 └─ <Bar /> x n     props: active, done, onComplete
```

| Bar status | `active` | `done` | Visual |
| --- | --- | --- | --- |
| Queued | false | false | empty track |
| Running | true | false | fill animating |
| Done | false | true | full |

**CSS transition vs `requestAnimationFrame`**

| | CSS transition | rAF loop |
| --- | --- | --- |
| Code | Toggle a class, listen to `transitionend` | Hook with timestamps, `setState` each frame |
| Performance | Compositor thread (`transform`) | Main thread, re-render per frame |
| Percentage text | Not available | Easy |
| Pause / resume / real progress | Hard | Easy |
| Best for | Fixed-duration fill | Number display, pausing, data-driven progress |

---

## 5. Reference Solution

<details>
<summary><strong>Show Full Code</strong></summary>

**`App.jsx`** (CSS-transition version — the recommended answer)

```jsx
import { useMemo, useReducer, useRef, useState } from 'react';
import Bar from './Bar';
import './progress.css';

const DURATION = 2000;

function barsReducer(state, action) {
  switch (action.type) {
    case 'ADD':
      return [...state, { id: action.id, done: false }];
    case 'DONE':
      return state.map((b) => (b.id === action.id ? { ...b, done: true } : b));
    case 'RESET':
      return [];
    default:
      return state;
  }
}

export default function App() {
  const [bars, dispatch] = useReducer(barsReducer, []);
  const [maxConcurrency, setMaxConcurrency] = useState(1);
  const nextId = useRef(1);

  // The first N unfinished bars are "active" — everything else is queued.
  const activeIds = useMemo(() => {
    const ids = new Set();
    for (const bar of bars) {
      if (ids.size === maxConcurrency) break;
      if (!bar.done) ids.add(bar.id);
    }
    return ids;
  }, [bars, maxConcurrency]);

  return (
    <div className="pb-app">
      <div className="pb-controls">
        <button type="button" onClick={() => dispatch({ type: 'ADD', id: nextId.current++ })}>
          Add
        </button>
        <label>
          Max concurrency{' '}
          <select
            value={maxConcurrency}
            onChange={(e) => setMaxConcurrency(Number(e.target.value))}
          >
            {[1, 2, 3, 4].map((n) => (
              <option key={n} value={n}>
                {n}
              </option>
            ))}
          </select>
        </label>
        <button type="button" onClick={() => dispatch({ type: 'RESET' })}>
          Reset
        </button>
      </div>

      <div className="pb-list">
        {bars.map((bar, i) => (
          <Bar
            key={bar.id}
            label={`Bar ${i + 1}`}
            duration={DURATION}
            active={activeIds.has(bar.id)}
            done={bar.done}
            onComplete={() => dispatch({ type: 'DONE', id: bar.id })}
          />
        ))}
      </div>
    </div>
  );
}
```

**`Bar.jsx`**

```jsx
import { useEffect, useState } from 'react';

export default function Bar({ label, duration, active, done, onComplete }) {
  const [filled, setFilled] = useState(false);

  // A transition only runs if the browser paints the "empty" state first.
  // Two nested rAFs guarantee at least one paint before we flip the class.
  useEffect(() => {
    if (!active) return;
    let raf2;
    const raf1 = requestAnimationFrame(() => {
      raf2 = requestAnimationFrame(() => setFilled(true));
    });
    return () => {
      cancelAnimationFrame(raf1);
      cancelAnimationFrame(raf2);
    };
  }, [active]);

  const status = done ? 'Complete' : active ? 'Loading' : 'Queued';

  return (
    <div
      className="bar"
      role="progressbar"
      aria-label={label}
      aria-valuemin={0}
      aria-valuemax={100}
      aria-valuenow={done ? 100 : 0}
      aria-valuetext={status}
    >
      <div
        className={`bar__fill${filled ? ' bar__fill--full' : ''}`}
        style={{ transitionDuration: `${duration}ms` }}
        onTransitionEnd={(e) => {
          if (e.target === e.currentTarget && e.propertyName === 'transform') {
            onComplete();
          }
        }}
      />
      <span className="bar__status">{status}</span>
    </div>
  );
}
```

**`progress.css`**

```css
.pb-app {
  max-width: 520px;
  margin: 32px auto;
  font-family: system-ui, sans-serif;
}

.pb-controls {
  display: flex;
  gap: 12px;
  align-items: center;
  margin-bottom: 16px;
}

.pb-list {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.bar {
  position: relative;
  height: 24px;
  background: #e5e7eb;
  border-radius: 12px;
  overflow: hidden;
}

.bar__fill {
  position: absolute;
  inset: 0;
  background: #2563eb;
  transform: scaleX(0);
  transform-origin: left center;
  transition-property: transform;
  transition-timing-function: linear;
  will-change: transform;
}

.bar__fill--full {
  transform: scaleX(1);
}

.bar__status {
  position: relative;
  padding-left: 10px;
  font-size: 12px;
  line-height: 24px;
  color: #fff;
  text-shadow: 0 0 3px rgba(0, 0, 0, 0.7);
}
```

**Variant — rAF-driven progress with a percentage (`RafBar.jsx`)**

```jsx
import { useEffect, useRef, useState } from 'react';

function useRafProgress(active, duration, onDone) {
  const [progress, setProgress] = useState(0);
  const progressRef = useRef(0);
  const onDoneRef = useRef(onDone);

  useEffect(() => {
    onDoneRef.current = onDone; // always call the latest callback
  });

  useEffect(() => {
    if (!active || progressRef.current >= 1) return;
    // Subtract already-made progress so the loop can resume after pausing.
    const startedAt = performance.now() - progressRef.current * duration;
    let rafId;

    const tick = (now) => {
      const p = Math.min(Math.max((now - startedAt) / duration, 0), 1);
      progressRef.current = p;
      setProgress(p);
      if (p < 1) rafId = requestAnimationFrame(tick);
      else onDoneRef.current?.();
    };

    rafId = requestAnimationFrame(tick);
    return () => cancelAnimationFrame(rafId);
  }, [active, duration]);

  return progress;
}

export default function RafBar({ label, duration, active, onComplete }) {
  const progress = useRafProgress(active, duration, onComplete);
  const pct = Math.round(progress * 100);

  return (
    <div
      className="bar"
      role="progressbar"
      aria-label={label}
      aria-valuemin={0}
      aria-valuemax={100}
      aria-valuenow={pct}
    >
      <div className="bar__fill" style={{ transform: `scaleX(${progress})`, transition: 'none' }} />
      <span className="bar__status">{pct}%</span>
    </div>
  );
}
```

Swap `<Bar>` for `<RafBar>` in `App.jsx` — it takes the same props (`done` is simply ignored; the reducer still tracks it for scheduling). A finished bar keeps showing 100% because its progress state stays at 1.

</details>

---

## 6. How It Works

- **Derived scheduling.** `activeIds` is computed from `bars` + `maxConcurrency` on every render — a pure function. Adding a bar, finishing a bar, or changing the limit all use the same path, so there is no "who starts the next bar?" logic to get wrong.
- **Completion drives the pipeline.** `onTransitionEnd` → `DONE` → the bar leaves the "first N unfinished" window → the next queued bar becomes `active` → its effect flips `filled` → its transition starts.
- **Why two `requestAnimationFrame`s?** A bar can mount directly in the active state. If the class is applied in the same frame as mount, the browser never sees the `scaleX(0)` start state and jumps to 100% without animating. Waiting two frames guarantees a paint of the initial state.
- **`transform: scaleX` instead of `width`** — animated on the compositor, no layout per frame.
- **`e.target === e.currentTarget` + `propertyName`** guards against bubbled `transitionend` from children or other animated properties.
- **`filled` persists after done**, so a finished bar stays full; `done` is kept in the reducer so the scheduler knows it no longer occupies a slot.
- **rAF variant:** progress is computed from timestamps (not by adding a fixed increment per frame), so it's frame-rate independent and resumable.

---

## 7. Edge Cases

- Rapid clicks on Add (50 bars) → only N run; others queued; no timers to leak.
- Lowering concurrency while bars are running → running bars finish (their `filled` state is already true), new bars wait until the count drops. Say this explicitly; cancelling mid-flight is a design choice. (The rAF variant behaves differently: a bar that leaves the active window pauses and resumes later.)
- Raising concurrency while bars are queued → queued bars start immediately on the next render.
- Reset while bars are running → components unmount; pending rAFs are cancelled in cleanup; no `DONE` dispatch after unmount (the transition never ends).
- `transitionend` doesn't fire if the element becomes `display: none` or the transition is disabled (e.g. a blanket `prefers-reduced-motion` rule setting `transition: none`) → bar would hang. Provide a fallback (e.g. `setTimeout(duration)` backup or skip the animation and complete immediately).
- Background tab: rAF is paused while the tab is hidden, so the rAF variant stalls; timestamp-based math makes it jump to the right value when the tab returns. CSS transitions keep their timeline, but `transitionend` may only be delivered once the tab is visible again, so the next queued bar can start late.
- Strict Mode double effects → effect cleanup cancels rAF; second run re-schedules. Safe.

---

## 8. Accessibility

- `role="progressbar"` with `aria-valuemin/max/now` and `aria-label`.
- With a CSS transition the real percentage is unknown, so expose state via `aria-valuetext` (`Queued / Loading / Complete`) — children of `role="progressbar"` are presentational, so the visible status text isn't what screen readers read. The rAF variant can report the true `aria-valuenow`.
- Do not announce every percentage change to screen readers (too chatty); announce completion via a polite live region if needed.
- Don't convey state by colour alone — the text label ("Queued", "Loading", "Complete") does that.
- Provide visible focus styles for the buttons and a `<label>` for the select.

---

## 9. Performance Considerations

- **CSS transition on `transform`**: zero React renders per frame, runs on the compositor — the best option for a fixed-duration fill.
- **rAF + `setState` per frame**: re-renders each active bar ~60×/s. Fine for a handful; for dozens, write `style.transform` through a ref instead of state, or lift it out of React.
- `will-change: transform` hints the browser; remove it for long lists of finished bars.
- Derivation is O(n) per render with `useMemo`; trivial for realistic sizes. For thousands, store `activeIds` incrementally.
- Stable `key` (id) so a bar's `filled` state isn't lost when bars are added.

---

## 10. Follow-up Questions

<details>
<summary><strong>1. How would you make the bar reflect real upload progress?</strong></summary>

Replace the time-based animation with a controlled `progress` prop (0–100) set from `XMLHttpRequest.upload.onprogress` or `axios onUploadProgress`. Render `transform: scaleX(progress/100)` with a short `transition` to smooth updates. The scheduler is unchanged, except the *task* starts a request instead of a CSS class.

</details>

<details>
<summary><strong>2. How do you add pause/resume?</strong></summary>

With CSS: set `animation-play-state: paused` (use a keyframe animation instead of a transition). With rAF: stop the loop and keep `progressRef`; on resume compute `startedAt = now - progress * duration` (as in the variant above).

</details>

<details>
<summary><strong>3. How would you generalize this to a promise pool (run N async tasks at once)?</strong></summary>

Same derivation idea, or a classic pool: keep a queue and a `running` counter; `runNext()` shifts a task, increments `running`, awaits it, then decrements and calls `runNext()` again in `finally`. In React, keep the counter in a ref and mirror results in state.

</details>

<details>
<summary><strong>4. Why derive active bars instead of storing status and using `useEffect` to start the next?</strong></summary>

Effect-driven scheduling needs correct dependencies and reads state that might be stale/batched; two bars finishing in the same tick can both try to start "the next one" and over- or under-start. Derived state is a single source of truth and impossible to get out of sync.

</details>

<details>
<summary><strong>5. What if each bar has a different duration?</strong></summary>

Store `duration` on the bar object and pass it to `Bar`. Nothing else changes — the scheduler only cares about `done`.

</details>

<details>
<summary><strong>6. Add "cancel" for a bar.</strong></summary>

Add a `CANCEL` action that removes it (or sets `cancelled: true` and treat it like `done` for scheduling). Unmounting a running bar cancels its pending rAFs through cleanup; the transition simply never ends.

</details>

<details>
<summary><strong>7. How would you make the bars loop (infinite queue) or auto-add bars?</strong></summary>

Add an interval in an effect that dispatches `ADD`, with cleanup via `clearInterval`, and use a ref for the id counter. Cap the array length or prune done bars to avoid unbounded growth.

</details>

---

## 11. Common Mistakes

- Using `setInterval` to bump a percentage by a fixed step — drifts, isn't frame-rate independent, and is often left uncleared.
- Starting the next bar with `setTimeout(…, 2000)` chained from the previous one — breaks as soon as concurrency or duration changes.
- Applying the "filled" class on mount with no initial paint → bar jumps to 100% with no animation.
- Animating `width` (layout every frame) instead of `transform`.
- Not guarding `transitionend` → fires multiple times / from children, marking bars done twice (reducer should be idempotent anyway).
- Using the array index as `key`.
- Storing `status` in state *and* deriving it elsewhere → two sources of truth.
- Mutating state: `bars[i].done = true`.
- Not clearing rAF in effect cleanup.

