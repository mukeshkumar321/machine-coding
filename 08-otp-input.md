# OTP Input — Machine Coding

> **Difficulty:** Medium | **Time:** 35–45 min | **Frequency:** ⭐⭐⭐⭐

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

Build a reusable **OTP input** component: a row of N (default 6) single-digit boxes. The user types digits and focus moves forward automatically. Backspace moves backward, arrow keys navigate, and pasting a code fills all the boxes. When every box is filled the component calls `onComplete(code)`.

```jsx
<OtpInput length={6} onComplete={(code) => verify(code)} />
```

---

## 2. Requirements

### Functional

- Render `length` inputs, each accepting **one digit**.
- **Auto-focus** the first box on mount; typing a digit moves focus to the next box.
- **Backspace**: clears the current box; if the box is already empty, move to the previous box and clear it.
- **ArrowLeft / ArrowRight** (also Home / End) move focus.
- **Paste**: strip non-digits, distribute across boxes, focus the next empty (or last) box.
- Reject non-numeric characters.
- Fire `onComplete(code)` when all boxes are filled.
- Support SMS autofill on mobile (`autocomplete="one-time-code"`).

### Non-functional

- Controlled component state; handlers don't use stale values.
- Fully keyboard-operable and screen-reader friendly.
- No unnecessary re-renders beyond the boxes themselves.

---

## 3. Clarifying Questions to Ask

- Numeric only, or alphanumeric?
- Fixed length or configurable? Do we need to mask digits (like a PIN)?
- What happens on paste of a code longer/shorter than N?
- Should the user be allowed to click any box, or only the first empty one?
- Should `onComplete` fire on every completion or just the first? Auto-submit or explicit "Verify" button?
- Resend timer / error state / disabled state?
- Mobile: do we need the numeric keypad and SMS autofill?

---

## 4. Approach & Component Design

**State shape** — one array of single-character strings, plus an array of input refs.

```js
values: ['1', '2', '', '', '', '']
inputsRef.current: [<input>, <input>, ...]   // for imperative focus()
```

**Component tree**

```
App
 └─ OtpInput (length, onComplete)
     └─ <input> x length   (ref, value, onChange, onKeyDown, onPaste, onFocus)
```

**Event → behaviour table**

| Event | Condition | Result |
| --- | --- | --- |
| `change` | digits typed | fill from this index, focus next |
| `change` | value `''` | clear cell (stay) |
| `change` | non-digit | ignored (controlled input snaps back) |
| `keydown Backspace` | cell empty & i > 0 | clear previous, focus previous |
| `keydown ←/→` | — | focus i-1 / i+1 |
| `keydown Home/End` | — | focus first / last |
| `paste` | any | digits fill from 0 (full code) or from i |
| `focus` | — | select content so typing replaces it |

**One helper, `fillFrom(start, digits)`**, handles typing (1 digit), SMS autofill (many digits in one `change`) and paste — one code path, fewer bugs.

---

## 5. Reference Solution

<details>
<summary><strong>Show Full Code</strong></summary>

**`OtpInput.jsx`**

```jsx
import { useEffect, useRef, useState } from 'react';
import './otp.css';

export default function OtpInput({
  length = 6,
  onComplete,
  disabled = false,
  autoFocus = true,
  hasError = false,
}) {
  const [values, setValues] = useState(() => Array(length).fill(''));
  const inputsRef = useRef([]);

  useEffect(() => {
    if (autoFocus) inputsRef.current[0]?.focus();
  }, [autoFocus]);

  const focusAt = (index) => {
    const i = Math.max(0, Math.min(index, length - 1));
    const el = inputsRef.current[i];
    if (el) {
      el.focus();
      el.select();
    }
  };

  const commit = (next) => {
    setValues(next);
    if (next.every(Boolean)) onComplete?.(next.join(''));
  };

  // Single code path for typing, autofill and paste.
  const fillFrom = (start, digits) => {
    const next = [...values];
    let i = start;
    for (const d of digits) {
      if (i >= length) break;
      next[i++] = d;
    }
    commit(next);
    focusAt(i);
  };

  const handleChange = (e, index) => {
    const raw = e.target.value;
    const digits = raw.replace(/\D/g, '');

    if (!digits) {
      if (raw === '') {
        const next = [...values];
        next[index] = '';
        commit(next);
      }
      return; // non-numeric input is dropped
    }
    fillFrom(index, digits);
  };

  const handleKeyDown = (e, index) => {
    switch (e.key) {
      case 'Backspace':
        // Non-empty cell: the browser fires `change` with '' and we clear it.
        if (!values[index] && index > 0) {
          e.preventDefault();
          const next = [...values];
          next[index - 1] = '';
          commit(next);
          focusAt(index - 1);
        }
        break;
      case 'ArrowLeft':
        e.preventDefault();
        focusAt(index - 1);
        break;
      case 'ArrowRight':
        e.preventDefault();
        focusAt(index + 1);
        break;
      case 'Home':
        e.preventDefault();
        focusAt(0);
        break;
      case 'End':
        e.preventDefault();
        focusAt(length - 1);
        break;
      default:
    }
  };

  const handlePaste = (e, index) => {
    e.preventDefault();
    const digits = e.clipboardData.getData('text').replace(/\D/g, '');
    if (!digits) return;
    // A full-length code always replaces everything; a partial one fills from here.
    fillFrom(digits.length >= length ? 0 : index, digits.slice(0, length));
  };

  return (
    <div className="otp" role="group" aria-label="One-time password">
      {values.map((value, i) => (
        <input
          // Fixed positional slots: index is a stable identity here.
          key={i}
          ref={(el) => {
            inputsRef.current[i] = el;
          }}
          className={`otp__cell${hasError ? ' otp__cell--error' : ''}`}
          type="text"
          inputMode="numeric"
          pattern="\d*"
          autoComplete={i === 0 ? 'one-time-code' : 'off'}
          aria-label={`Digit ${i + 1} of ${length}`}
          aria-invalid={hasError || undefined}
          value={value}
          disabled={disabled}
          onChange={(e) => handleChange(e, i)}
          onKeyDown={(e) => handleKeyDown(e, i)}
          onPaste={(e) => handlePaste(e, i)}
          onFocus={(e) => e.target.select()}
        />
      ))}
    </div>
  );
}
```

**`App.jsx`**

```jsx
import { useState } from 'react';
import OtpInput from './OtpInput';

const CORRECT_CODE = '123456';

export default function App() {
  const [status, setStatus] = useState('idle'); // idle | success | error
  const [attempt, setAttempt] = useState(0);

  const handleComplete = (code) => setStatus(code === CORRECT_CODE ? 'success' : 'error');

  const reset = () => {
    setStatus('idle');
    setAttempt((n) => n + 1); // new key -> fresh, empty OtpInput
  };

  return (
    <main style={{ maxWidth: 420, margin: '48px auto', fontFamily: 'system-ui' }}>
      <h1>Enter verification code</h1>
      <p>We sent a 6-digit code to your phone. (Hint: 123456)</p>

      <OtpInput
        key={attempt}
        length={6}
        onComplete={handleComplete}
        hasError={status === 'error'}
        disabled={status === 'success'}
      />

      <p role="status" aria-live="polite" style={{ minHeight: 24 }}>
        {status === 'success' && 'Verified ✓'}
        {status === 'error' && 'Incorrect code, try again.'}
      </p>

      <button type="button" onClick={reset}>
        Reset
      </button>
    </main>
  );
}
```

**`otp.css`**

```css
.otp {
  display: flex;
  gap: 8px;
}

.otp__cell {
  width: 48px;
  height: 56px;
  text-align: center;
  font-size: 24px;
  font-weight: 600;
  border: 2px solid #cbd5e1;
  border-radius: 8px;
  outline: none;
  caret-color: transparent;
  transition: border-color 120ms, box-shadow 120ms;
}

.otp__cell:focus {
  border-color: #2563eb;
  box-shadow: 0 0 0 3px rgba(37, 99, 235, 0.25);
}

.otp__cell--error {
  border-color: #dc2626;
}

.otp__cell:disabled {
  background: #f1f5f9;
  color: #64748b;
}

@media (max-width: 420px) {
  .otp { gap: 6px; }
  .otp__cell { width: 40px; height: 48px; font-size: 20px; }
}
```

</details>

---

## 6. How It Works

- **`values` array + `inputsRef` array.** State drives what's shown; refs are only for the imperative `focus()`/`select()` calls — focus is not something you can express declaratively.
- **`fillFrom(start, digits)`** is the core. Typing one digit = `fillFrom(i, '5')`. Mobile SMS autofill dispatches one `change` with the whole code (`'123456'`) = `fillFrom(i, '123456')`. Paste = `fillFrom(0, code)` for a full-length code, `fillFrom(i, digits)` for a partial one. One function, three features.
- **We copy before writing** (`[...values]`) — no state mutation.
- **No `maxLength=1`** — it silently truncates autofill and some IME input before our handler sees it. We sanitise ourselves.
- **`onFocus → select()`** means typing over an existing digit *replaces* it instead of producing `"12"`.
- **Backspace logic.** On a non-empty cell, let the browser clear it (we receive `change('')`). On an empty cell there is nothing to delete, so we manually clear the previous one and move back — that's the behaviour users expect.
- **Handlers close over fresh `values`** because they're recreated each render; React flushes discrete events (typing) synchronously so two rapid keystrokes don't see stale state. If you used the functional `setValues(prev => …)` form you'd have to compute `onComplete` inside the updater — a side effect in an updater is wrong, which is why we compute `next` first and then `commit`.
- **`onComplete` is called from the event handler, not an effect** — effects reacting to "all filled" can fire on re-renders/Strict Mode and double-submit.
- **Reset via `key`** remounts the component with fresh state; no imperative `reset()` needed.

---

## 7. Edge Cases

- Paste with spaces/dashes (`"123 456"`, `"123-456"`) → non-digits stripped.
- Paste longer than `length` → truncated; shorter → fills from the current cell.
- Paste of non-numeric text → ignored (no state change).
- Typing a letter → controlled input rejects it (value stays).
- Backspace on first empty cell → nothing happens (no index −1).
- Typing in the last cell → focus stays on last (clamped), no crash on `inputsRef[length]`.
- Cell already filled and user types a different digit → replaced (thanks to `select()`).
- Clicking into the middle of a partially-filled OTP → allowed; typing continues forward from there.
- Editing a cell after completion → `onComplete` fires again when full; document or guard if the parent should only submit once.
- Cursor placement: `caret-color: transparent` hides the caret since each box has one char.
- Some Android soft keyboards report `e.key === 'Unidentified'` for Backspace, so the empty-cell Backspace branch may not fire there; a robust fix listens to `beforeinput` (`inputType === 'deleteContentBackward'`) instead. Test on a real device.
- Strict Mode: auto-focus effect runs twice — harmless.
- `length` prop changes at runtime: `values` length won't follow (initial state only). Remount with `key={length}`.

---

## 8. Accessibility

- Wrapper `role="group"` + `aria-label="One-time password"`; each input has `aria-label="Digit 3 of 6"`.
- `inputMode="numeric"` + `pattern="\d*"` → numeric keypad on mobile.
- `autoComplete="one-time-code"` on the first box enables iOS/Android SMS suggestion.
- `aria-invalid` on error plus a visible error message in an `aria-live="polite"` region.
- Don't `type="number"`: it adds spinners, allows `e`/`-`, and doesn't support the caret/selection-range APIs (`setSelectionRange`).
- Never remove focus outlines; provide a visible `:focus` style (done).
- Arrow keys give keyboard users the same navigation as mouse users; Tab still moves through the boxes in order.
- Alternative pattern worth mentioning: a **single** transparent input with N visual boxes — simpler for screen readers and autofill.

---

## 9. Performance Considerations

- N is tiny (4–8); re-rendering all boxes per keystroke is negligible. Don't reach for `memo` here.
- Refs avoid state for focus handling — no re-render when focus moves.
- Avoid creating new arrays in render beyond the `.map`.
- Avoid effects that watch `values` to move focus (extra render + flicker); do it directly in event handlers.

---

## 10. Follow-up Questions

<details>
<summary><strong>1. How would you restrict focus to the first empty box?</strong></summary>

In `onFocus`, find `values.findIndex(v => !v)`; if it's `-1` allow any, otherwise if the focused index is greater than the first empty index, redirect with `focusAt(firstEmpty)`. This mimics most banking UIs.

</details>

<details>
<summary><strong>2. How would you support alphanumeric codes?</strong></summary>

Change the sanitiser regex from `/\D/g` to `/[^a-z0-9]/gi`, drop `inputMode="numeric"`/`pattern`, and decide on case normalisation (`toUpperCase()`).

</details>

<details>
<summary><strong>3. How do you implement a masked (PIN) variant?</strong></summary>

Use `type="password"` on the inputs (keeps `value` real, hides the glyph) — or render a bullet and keep the real values in state. With `type="password"`, browsers/password managers may treat the field as a credential, so verify SMS autofill and `autoComplete` behaviour on real devices.

</details>

<details>
<summary><strong>4. How would you expose imperative methods like `reset()` / `focus()` to the parent?</strong></summary>

`forwardRef` + `useImperativeHandle(ref, () => ({ reset: () => setValues(Array(length).fill('')), focus: () => focusAt(0) }))`. Using `key` for reset is simpler when you don't need focus control.

</details>

<details>
<summary><strong>5. How do you add a "Resend code in 30s" timer?</strong></summary>

A `useEffect` with `setInterval` (or `setTimeout` chain) keyed on `secondsLeft`, with cleanup. Store the deadline timestamp rather than decrementing a counter to avoid drift in background tabs. On resend, reset the OTP via `key` and restart the timer.

</details>

<details>
<summary><strong>6. Why not `maxLength={1}`?</strong></summary>

It truncates multi-char input before JS sees it, breaking SMS autofill and some mobile keyboards. Sanitise in `onChange` instead.

</details>

<details>
<summary><strong>7. How would you use the Web OTP API?</strong></summary>

`navigator.credentials.get({ otp: { transport: ['sms'] }, signal })` (Chrome Android) resolves with the code from an SMS formatted with `@domain #code`. Feature-detect `'OTPCredential' in window`, abort via `AbortController` in effect cleanup, then call `fillFrom(0, code)`.

</details>

<details>
<summary><strong>8. How would you make this a controlled component (`value` / `onChange` props)?</strong></summary>

Accept `value` as a string, derive `values = Array.from({length}, (_, i) => value[i] ?? '')`, and call `onChange(next.join(''))`. Trade-off: with a joined string, a gap in the middle (empty cell 2, filled cell 3) can't be represented — use an array prop or a placeholder char.

</details>

---

## 11. Common Mistakes

- Using `type="number"` (spinners, `e`, `-`, scroll changes value).
- `maxLength={1}` breaking autofill/paste.
- Moving focus inside `useEffect` on `values` change → double renders, focus fights.
- Mutating state: `values[i] = d; setValues(values)` → no re-render.
- Forgetting Backspace on an empty box (the most-tested behaviour).
- Not handling paste, or pasting into one box only.
- Not clamping focus indices → `inputsRef.current[-1]` / `[length]` errors.
- Not stripping non-digits from pasted text.
- Using a single `useRef` object instead of an array of refs (or re-creating refs each render incorrectly).
- Firing `onComplete` from an effect → duplicate calls.
- Missing `aria-label`s — every box reads as "edit text".

