# Counter → Stopwatch / Timer — Machine Coding

> **Difficulty:** Easy → Medium | **Time:** 30–45 min | **Frequency:** ⭐⭐⭐⭐

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

Start with a simple **counter** (increment, decrement, reset). Then build a **stopwatch** (start / pause / resume / reset, showing `mm:ss.cs`) and a **countdown timer** (set a duration, start / pause / reset, stop automatically at zero). The interviewer will probe **interval cleanup**, **stale closures**, and **timer drift**.

---

## 2. Requirements

### Functional

- Counter: `+`, `−`, reset; optional step and min/max.
- Stopwatch: Start, Pause (keeps elapsed), Resume, Reset (works while running).
- Timer: input minutes/seconds, Start, Pause, Reset, stops at `00:00` and shows "Time's up".
- Time display formatted `mm:ss` (stopwatch also `.cs`).

### Non-functional

- No leaked intervals (unmount, pause, re-render).
- Accurate time — must not accumulate drift or run wrong when the tab is throttled.
- Reusable logic in a custom hook; UI is a thin layer.
- Buttons disabled when action is meaningless (e.g. Pause when stopped).

---

## 3. Clarifying Questions to Ask

- What precision is needed (seconds vs centiseconds)? Drives tick rate.
- Should the timer keep running if the tab is in the background? (Wall-clock based → yes.)
- Lap times? Sound/notification at zero? (Assume out of scope.)
- Should state persist on reload? (Would need to store `startedAt`/`elapsed`.)
- Multiple timers on the page? (Hook must be instance-safe — no module-level state.)

---

## 4. Approach & Component Design

**Key idea:** *don't count ticks; measure time.* `setInterval` only decides **when to re-render**; the **value** is computed from a clock (`performance.now()`).

```
elapsed = baseElapsed + (now - startedAt)       // while running
elapsed = baseElapsed                            // while paused
```

| Piece | Where it lives | Why |
| --- | --- | --- |
| `elapsed` (ms) | `useState` | drives rendering |
| `running` | `useState` | drives effect (start/stop interval) |
| `startedAt` | `useRef` | changes must not re-render |
| `baseElapsed` | `useRef` | time banked from previous runs |

```
App
 ├── Counter            (useState, functional updates)
 ├── Stopwatch          (useStopwatch)
 └── Timer              (useStopwatch + duration → remaining = duration − elapsed)
        └── shared: useStopwatch(), formatTime()
```

---

## 5. Reference Solution

<details>
<summary><strong>Show Full Code</strong></summary>

**`Counter.jsx`**

```jsx
import { useState } from 'react';

export default function Counter({ step = 1, min = -Infinity, max = Infinity }) {
  const [count, setCount] = useState(() => Math.min(max, Math.max(min, 0)));

  return (
    <section className="card">
      <h2>Counter</h2>
      <output aria-live="polite" className="display">{count}</output>
      <div className="row">
        <button onClick={() => setCount((c) => Math.max(min, c - step))} disabled={count <= min}>−</button>
        <button onClick={() => setCount(Math.min(max, Math.max(min, 0)))}>Reset</button>
        <button onClick={() => setCount((c) => Math.min(max, c + step))} disabled={count >= max}>+</button>
      </div>
    </section>
  );
}
```

**`useStopwatch.js`**

```jsx
import { useCallback, useEffect, useRef, useState } from 'react';

export function useStopwatch(tickMs = 10) {
  const [elapsed, setElapsed] = useState(0); // ms, for rendering
  const [running, setRunning] = useState(false);
  const startedAtRef = useRef(0); // performance.now() when current run began
  const bankedRef = useRef(0);    // ms accumulated by previous runs

  // The interval only triggers re-renders; the value comes from the clock.
  useEffect(() => {
    if (!running) return;
    const id = setInterval(() => {
      setElapsed(bankedRef.current + performance.now() - startedAtRef.current);
    }, tickMs);
    return () => clearInterval(id);
  }, [running, tickMs]);

  const start = useCallback(() => {
    if (running) return;
    startedAtRef.current = performance.now();
    setRunning(true);
  }, [running]);

  const pause = useCallback(() => {
    if (!running) return;
    bankedRef.current += performance.now() - startedAtRef.current;
    setElapsed(bankedRef.current);
    setRunning(false);
  }, [running]);

  const reset = useCallback(() => {
    bankedRef.current = 0;
    setElapsed(0);
    setRunning(false);
  }, []);

  return { elapsed, running, start, pause, reset };
}
```

**`formatTime.js`**

```jsx
const pad = (n, len = 2) => String(n).padStart(len, '0');

export function formatTime(ms, withCentis = false) {
  const totalCs = Math.floor(ms / 10);
  const cs = totalCs % 100;
  const totalSec = Math.floor(totalCs / 100);
  const sec = totalSec % 60;
  const min = Math.floor(totalSec / 60);
  const base = `${pad(min)}:${pad(sec)}`;
  return withCentis ? `${base}.${pad(cs)}` : base;
}
```

**`Stopwatch.jsx`**

```jsx
import { useStopwatch } from './useStopwatch';
import { formatTime } from './formatTime';

export default function Stopwatch() {
  const { elapsed, running, start, pause, reset } = useStopwatch(10);

  return (
    <section className="card">
      <h2>Stopwatch</h2>
      <output className="display" role="timer" aria-live="off">
        {formatTime(elapsed, true)}
      </output>
      <div className="row">
        {running ? (
          <button onClick={pause}>Pause</button>
        ) : (
          <button onClick={start}>{elapsed > 0 ? 'Resume' : 'Start'}</button>
        )}
        <button onClick={reset} disabled={!running && elapsed === 0}>Reset</button>
      </div>
    </section>
  );
}
```

**`Timer.jsx`**

```jsx
import { useEffect, useState } from 'react';
import { useStopwatch } from './useStopwatch';
import { formatTime } from './formatTime';

export default function Timer() {
  const [minutes, setMinutes] = useState(1);
  const [seconds, setSeconds] = useState(0);
  const { elapsed, running, start, pause, reset } = useStopwatch(100);

  const durationMs = (minutes * 60 + seconds) * 1000;
  const remaining = Math.max(0, durationMs - elapsed);
  const finished = durationMs > 0 && remaining === 0;
  const untouched = !running && elapsed === 0;

  useEffect(() => {
    if (running && remaining === 0) pause();
  }, [running, remaining, pause]);

  // Round up so the display shows 00:01 until time is truly up
  const shown = formatTime(Math.ceil(remaining / 1000) * 1000);

  const clamp = (v, max) => Math.min(max, Math.max(0, Number(v) || 0));

  return (
    <section className="card">
      <h2>Timer</h2>
      <div className="row">
        <label>
          Min
          <input
            type="number" min="0" max="999" value={minutes} disabled={!untouched}
            onChange={(e) => setMinutes(clamp(e.target.value, 999))}
          />
        </label>
        <label>
          Sec
          <input
            type="number" min="0" max="59" value={seconds} disabled={!untouched}
            onChange={(e) => setSeconds(clamp(e.target.value, 59))}
          />
        </label>
      </div>

      <output className="display" role="timer" aria-live="off">{shown}</output>
      <p role="status">{finished ? "Time's up!" : ''}</p>

      <div className="row">
        {running ? (
          <button onClick={pause}>Pause</button>
        ) : (
          <button onClick={start} disabled={durationMs === 0 || finished}>
            {elapsed > 0 ? 'Resume' : 'Start'}
          </button>
        )}
        <button onClick={reset} disabled={untouched}>Reset</button>
      </div>
    </section>
  );
}
```

**`App.jsx`**

```jsx
import Counter from './Counter';
import Stopwatch from './Stopwatch';
import Timer from './Timer';
import './styles.css';

export default function App() {
  return (
    <main className="app">
      <Counter />
      <Stopwatch />
      <Timer />
    </main>
  );
}
```

**`styles.css`**

```css
.app { display: grid; gap: 16px; max-width: 360px; margin: 32px auto; font-family: system-ui, sans-serif; }
.card { border: 1px solid #ddd; border-radius: 8px; padding: 16px; }
.display { display: block; font-size: 2.5rem; text-align: center; font-variant-numeric: tabular-nums; margin: 12px 0; }
.row { display: flex; gap: 8px; justify-content: center; align-items: end; }
.row input { width: 70px; display: block; }
button { padding: 6px 14px; cursor: pointer; }
button:disabled { cursor: not-allowed; opacity: 0.5; }
```

</details>

---

## 6. How It Works

- **Counter:** `setCount(c => c + 1)` (functional update) so rapid clicks / batched updates / intervals never read a stale `count`.
- **Elapsed from clock:** each tick recomputes `banked + (now − startedAt)`. If a tick is late (busy main thread, background tab throttled to ≥1 s), the next one is still **correct** — no drift.
- **Effect owns the interval:** it runs only while `running`, and the cleanup `clearInterval` fires on pause, reset, unmount, or a `tickMs` change. Never two intervals alive.
- **Refs vs state:** `startedAt`/`banked` are bookkeeping that must not cause renders → `useRef`. `elapsed` is what the UI shows → state.
- **Pause banks time in the handler, not in effect cleanup.** Doing it in cleanup makes `reset()` while running double-count (cleanup runs *after* the reset).
- **Timer = Stopwatch + subtraction.** `remaining = duration − elapsed`; an effect pauses when it hits 0. Handlers are `useCallback`-ed so the effect deps are honest.
- `Math.ceil` on display avoids showing `00:00` while 0.4 s still remain.
- `tabular-nums` stops the display from jittering as digits change.

---

## 7. Edge Cases

- Double-click Start → `if (running) return` prevents a second interval/`startedAt` reset.
- Reset while running → stops and zeroes; handler order avoids double counting.
- Unmount while running → cleanup clears interval (no "setState on unmounted" leaks).
- Timer duration `0` → Start disabled.
- Inputs: negative, empty, `NaN`, > 59 seconds → clamped.
- Editing duration mid-run → inputs disabled once started.
- Background tab throttling → wall-clock maths keeps value correct.
- React 18 StrictMode double-invoking effects → cleanup makes it safe.
- Timer hits zero between ticks → `Math.max(0, …)` clamps negative.

---

## 8. Accessibility

- Real `<button>`s with text labels; label the number inputs.
- `role="timer"` with `aria-live="off"` — announcing every 10 ms would be unusable; announce only milestones via a separate `role="status"` (e.g. "Time's up!"). Keep that status element mounted and only change its text — live regions inserted together with their content are often not announced.
- Disabled states reflect available actions.
- Counter value in `<output aria-live="polite">`.
- Don't rely on colour alone to indicate running/finished.

---

## 9. Performance Considerations

- Re-rendering at 10 ms = 100 renders/s. Pick the **tick rate to match the display** (1000 ms for seconds-only; ~250 ms is safe for "no skipped second").
- Keep the ticking component small so the rest of the tree doesn't re-render (state lives in `Stopwatch`, not `App`).
- For smooth millisecond displays use `requestAnimationFrame` (pauses in background tabs, aligned with paint).
- Avoid creating closures that read `elapsed` in the interval — read refs instead.
- Laps list: append immutably; key by lap id.

---

## 10. Follow-up Questions

<details>
<summary><strong>1. Why does <code>setInterval(() => setCount(count + 1), 1000)</code> only count to 1?</strong></summary>

The effect closed over `count = 0` on first render. Every tick computes `0 + 1`. Fix with `setCount(c => c + 1)` (and still clean up the interval), or include `count` in deps (re-creates interval each tick — not ideal).

</details>

<details>
<summary><strong>2. What is timer drift and how do you avoid it?</strong></summary>

`setInterval(fn, 1000)` isn't exact — delays accumulate (and background tabs throttle to ≥1 s). Counting ticks drifts. Instead store a timestamp (`Date.now()` / `performance.now()`) and derive elapsed from it. Optionally self-correct a `setTimeout` chain by subtracting the error.

</details>

<details>
<summary><strong>3. <code>Date.now()</code> vs <code>performance.now()</code>?</strong></summary>

`performance.now()` is monotonic (unaffected by system clock changes / NTP / DST) and high resolution — right for durations. `Date.now()` is wall-clock — needed if the timer must survive reloads (store an absolute `endAt`).

</details>

<details>
<summary><strong>4. How do you add lap times?</strong></summary>

`laps` state array; on Lap push `{ id, time: elapsed, split: elapsed − previousLapTime }` immutably (`setLaps(l => [...l, lap])`). Reset clears it.

</details>

<details>
<summary><strong>5. Make the timer survive a page reload.</strong></summary>

Persist `{ endAt: Date.now() + remaining, running }` (or `{ remainingMs, paused }`) in `localStorage`. On load compute `remaining = endAt − Date.now()`; if ≤ 0 show finished.

</details>

<details>
<summary><strong>6. Use <code>requestAnimationFrame</code> instead of <code>setInterval</code>?</strong></summary>

Yes for smooth sub-second UI: schedule `rAF` in an effect, compute from the clock, `cancelAnimationFrame` in cleanup. Downsides: paused in hidden tabs (fine because value is clock-based), renders at 60 fps even if only seconds are shown.

</details>

<details>
<summary><strong>7. Why not keep the timer state in a module-level variable?</strong></summary>

Module-level state is shared between component instances and survives unmount. Keeping it in hook refs/state makes each instance independent and cleans up with the component.

</details>

<details>
<summary><strong>8. How do you test it?</strong></summary>

Jest fake timers (`jest.useFakeTimers()`, `advanceTimersByTime`) and mock `performance.now`; assert display after start/pause/resume/reset and that unmount clears timers (`jest.getTimerCount()`).

</details>

---

## 11. Common Mistakes

- Not clearing the interval (or clearing in the wrong place) → multiple intervals, speed-up bug.
- `setCount(count + 1)` inside an interval → stale closure.
- Counting ticks (`seconds + 1`) instead of measuring time → drift.
- Starting a new interval on each Start click without a guard.
- Putting `startedAt` in state (extra renders, stale reads).
- Banking elapsed in effect cleanup → reset double-counts.
- Letting the timer go negative / not stopping the interval at zero.
- Not disabling inputs while running.
- Putting the ticking state in `App` so everything re-renders each tick.

