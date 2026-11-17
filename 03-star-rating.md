# Star Rating — Machine Coding

> **Difficulty:** Easy–Medium | **Time:** 30–40 min | **Asked at / Frequency:** Common (e-commerce / review widgets, very frequent in UI-component rounds) — ⭐⭐⭐⭐ | **Tier:** 1 (must know)

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

Build a reusable **Star Rating** component. The user hovers over stars to **preview** a rating and clicks to **set** it. It supports a configurable number of stars (`count`), **half-star** precision, a **read-only** mode (for displaying averages like 3.7), and works controlled or uncontrolled.

---

## 2. Requirements

### Functional

- Render `count` stars (default 5).
- Hover previews the rating; leaving the component restores the committed rating.
- Click commits the rating and calls `onChange`.
- `precision={0.5}` enables half stars (decided by left/right half of the star under the pointer).
- `readOnly` shows the rating (including fractions) with no interaction.
- Keyboard: ←/→ (and ↑/↓) change the value, Home/End → 0/max.

### Non-functional

- Reusable props API; controlled & uncontrolled.
- Pure CSS rendering (no images); scalable via `size`.
- Accessible as a slider (or radio group) with proper labels.
- No re-render churn on every `mousemove`.

---

## 3. Clarifying Questions to Ask

- Whole stars only, or half stars too? Arbitrary fractions in read-only mode?
- Can the user clear the rating (click same star / Home key → 0)?
- Controlled or uncontrolled? Is `onChange` fired on hover too? (No — only on commit.)
- Touch device behaviour? (Click works; hover preview isn't meaningful.)
- Custom icon / colours / RTL?
- Should it be a form field (`name` + hidden input)?

---

## 4. Approach & Component Design

**State shape**

```js
rating: number       // committed (internal state, or `value` prop when controlled)
hover:  number|null  // preview while pointer is over a star; null otherwise
display = hover ?? rating     // derived — what gets painted
```

**Rendering each star** — two stacked glyphs; the top (filled) one is clipped by width:

```
 star i (0-based)       fill% = clamp(display − i, 0, 1) × 100

 ┌──────────────┐   display = 3.5
 │ ★ (grey)     │   star 0,1,2 → 100%   star 3 → 50%   star 4 → 0%
 │ ★ (gold) 50% │
 └──────────────┘
```

| Input | Behaviour |
| --- | --- |
| `mousemove` on star `i` | `hover = i + (leftHalf ? 0.5 : 1)` (precision 0.5) or `i + 1` |
| `click` | commit hover value |
| `mouseleave` container | `hover = null` |
| keyboard | adjust `rating` by `precision` |

```
StarRating (role=slider | img)
 └── Star × count   (bg glyph + width-clipped fg glyph)
```

---

## 5. Reference Solution

<details>
<summary><strong>Show Full Code</strong></summary>

**`StarRating.jsx`**

```jsx
import { useState } from 'react';
import './star-rating.css';

export default function StarRating({
  count = 5,
  value,
  defaultValue = 0,
  onChange,
  precision = 1, // 1 or 0.5
  readOnly = false,
  size = 28,
  label = 'Rating',
}) {
  const isControlled = value !== undefined;
  const [inner, setInner] = useState(defaultValue);
  const [hover, setHover] = useState(null);

  const rating = isControlled ? value : inner;
  const display = hover ?? rating;

  const commit = (next) => {
    if (!isControlled) setInner(next);
    onChange?.(next);
  };

  const valueFromPointer = (e, index) => {
    if (precision >= 1) return index + 1;
    const { left, width } = e.currentTarget.getBoundingClientRect();
    return index + (e.clientX - left < width / 2 ? 0.5 : 1);
  };

  const handleKeyDown = (e) => {
    let next;
    switch (e.key) {
      case 'ArrowRight':
      case 'ArrowUp':
        next = Math.min(count, rating + precision);
        break;
      case 'ArrowLeft':
      case 'ArrowDown':
        next = Math.max(0, rating - precision);
        break;
      case 'Home': next = 0; break;
      case 'End': next = count; break;
      default: return;
    }
    e.preventDefault();
    setHover(null);
    if (next !== rating) commit(next);
  };

  const stars = Array.from({ length: count }, (_, i) => {
    const fill = Math.min(1, Math.max(0, display - i)) * 100;
    const handlers = readOnly
      ? {}
      : {
          onMouseMove: (e) => setHover(valueFromPointer(e, i)), // same value => React bails out
          onClick: (e) => {
            commit(valueFromPointer(e, i));
            setHover(null);
          },
        };

    return (
      <span key={i} className="star" style={{ fontSize: size }} {...handlers}>
        <span className="star-bg" aria-hidden="true">★</span>
        <span className="star-fg" style={{ width: `${fill}%` }} aria-hidden="true">★</span>
      </span>
    );
  });

  if (readOnly) {
    return (
      <div
        className="star-rating readonly"
        role="img"
        aria-label={`${label}: ${rating} out of ${count} stars`}
      >
        {stars}
      </div>
    );
  }

  return (
    <div
      className="star-rating"
      role="slider"
      tabIndex={0}
      aria-label={label}
      aria-valuemin={0}
      aria-valuemax={count}
      aria-valuenow={rating}
      aria-valuetext={`${rating} out of ${count} stars`}
      onKeyDown={handleKeyDown}
      onMouseLeave={() => setHover(null)}
    >
      {stars}
    </div>
  );
}
```

**`App.jsx`**

```jsx
import { useState } from 'react';
import StarRating from './StarRating';

export default function App() {
  const [rating, setRating] = useState(3);

  return (
    <main style={{ maxWidth: 420, margin: '40px auto', fontFamily: 'system-ui, sans-serif' }}>
      <h3>Controlled (whole stars)</h3>
      <StarRating value={rating} onChange={setRating} label="Product rating" />
      <p>You rated: {rating}</p>

      <h3>Uncontrolled (half stars)</h3>
      <StarRating defaultValue={2.5} precision={0.5} size={36} onChange={(v) => console.log(v)} />

      <h3>Read-only average</h3>
      <StarRating value={3.7} readOnly size={20} label="Average rating" />
    </main>
  );
}
```

**`star-rating.css`**

```css
.star-rating { display: inline-flex; user-select: none; }
.star-rating:focus-visible { outline: 2px solid #4f46e5; outline-offset: 4px; border-radius: 4px; }

.star { position: relative; display: inline-block; line-height: 1; cursor: pointer; }
.readonly .star { cursor: default; }

.star-bg { color: #d1d5db; }
.star-fg {
  position: absolute; top: 0; left: 0; height: 100%;
  overflow: hidden; white-space: nowrap; color: #f59e0b; pointer-events: none;
}
```

</details>

---

## 6. How It Works

- **Two values, one display:** committed `rating` vs transient `hover`. `display = hover ?? rating` means "leave" just clears `hover` — no need to remember the previous value.
- **Fill percentage** `clamp(display − i, 0, 1) × 100` handles whole stars, halves, *and* arbitrary fractions (3.7 → 4th star 70%) with one formula.
- **Half-star detection** uses `getBoundingClientRect()` of the star under the pointer (`e.currentTarget`) and compares `clientX` to the midpoint.
- **Foreground glyph has `pointer-events: none`** so the star wrapper always receives the event, and its `currentTarget` rect stays stable.
- **Controlled/uncontrolled** with `value !== undefined`; `onChange` fires only on commit (click/keyboard), never on hover.
- **Keyboard support via `role="slider"`** — a rating is a bounded numeric value, and slider semantics natively express half values (`aria-valuenow=3.5`).
- **`setHover` with the same value** is a no-op (React bails out), so `mousemove` doesn't re-render continuously.
- **Index keys are fine** here: fixed-length, non-reorderable list.

---

## 7. Edge Cases

- `value` > `count` or negative → clamp (`Math.min/max`) before using; fill formula already clamps visually.
- `count` = 0 or non-integer → guard with `Math.max(0, Math.floor(count))`.
- Click on the **same** rating → optionally clear to 0 (`allowClear`).
- Moving the mouse across the gap between stars → `hover` stays on the last value (no flicker) because there's no gap in the markup.
- Mouse leaves while hovering → `onMouseLeave` resets preview.
- Touch: after a tap, the emulated hover persists — we clear `hover` in `onClick`.
- Keyboard change while hover is set → `setHover(null)` so the new value shows.
- Read-only with fractional values (3.7) → rendered correctly via percentage fill.
- RTL → fill should grow from the right (`right: 0`) and half detection flips.
- Float drift from steps → using 0.5 (exact in binary) avoids it; arbitrary steps need rounding.

---

## 8. Accessibility

- Interactive: `role="slider"`, `tabIndex=0`, `aria-valuemin/max/now`, `aria-valuetext` ("3.5 out of 5 stars"), `aria-label`.
- Arrow keys, Home/End implemented (matches WAI-ARIA slider pattern).
- Read-only: `role="img"` with descriptive `aria-label`, not focusable.
- Star glyphs are `aria-hidden` — screen readers announce the numeric value only.
- Don't depend on colour alone; for low-vision users consider outlines/numeric text ("3.5 / 5").
- Alternative pattern: radio group (`<input type="radio">` per star, visually hidden) — native keyboard, forms, and arrow key behaviour for free (but whole stars only).
- Visible focus ring.

---

## 9. Performance Considerations

- Tiny component; the only hot path is `mousemove`. Bail-out on equal state prevents extra renders; avoid `setHover(obj)` with a new object each time.
- `getBoundingClientRect` forces layout per move; fine for 5 stars. Alternative: `e.nativeEvent.offsetX / e.currentTarget.offsetWidth` (no forced layout — but offsetX is relative to the *target*, so keep `pointer-events: none` on children).
- If rendered in a long list (reviews), wrap in `memo` and keep `onChange` stable; read-only variants have no handlers, so are cheap.
- CSS (`width: %`) does the clipping — no canvas/SVG recomputation.

---

## 10. Follow-up Questions

<details>
<summary><strong>1. Radio-group vs slider semantics — which would you pick?</strong></summary>

Radios: free native keyboard/form submission, whole values only, needs visually hidden inputs and a `fieldset/legend`. Slider: supports half stars and fractions naturally, but needs custom keyboard handling and doesn't submit in forms unless you add a hidden input. Pick based on precision and form needs.

</details>

<details>
<summary><strong>2. Add "click again to clear".</strong></summary>

In click handler: `const next = v === rating ? 0 : v;`. Document it and make it opt-in (`allowClear`). Keyboard Home already gives 0.

</details>

<details>
<summary><strong>3. Support custom icons / SVG and per-star colours.</strong></summary>

Accept `renderIcon(filled)` or `icon` props; keep the layered technique (background layer + clipped foreground layer). For SVG use `<clipPath>` or the same absolute/overflow approach.

</details>

<details>
<summary><strong>4. RTL support?</strong></summary>

Read `dir`; in RTL anchor the filled layer with `right: 0` and invert the half calculation (`right - clientX`). Arrow keys: ArrowLeft increases.

</details>

<details>
<summary><strong>5. Use it inside a form.</strong></summary>

Add a `name` prop and render `<input type="hidden" name={name} value={rating} />`, or use radios. Also support `required` validation and a `disabled` state.

</details>

<details>
<summary><strong>6. Show rating counts / histogram (e.g. Amazon).</strong></summary>

Separate component: for each star level render a bar with width `count / total × 100%`; reuse the read-only star component for the header average.

</details>

<details>
<summary><strong>7. How would you debounce or optimise hover events?</strong></summary>

Usually unnecessary. If needed: use `onPointerMove` + `requestAnimationFrame` throttle, or compute value from `offsetX` to skip layout reads. Never debounce commit actions.

</details>

---

## 11. Common Mistakes

- Single `useState` for rating and hover → must remember "previous rating" to restore on leave.
- Firing `onChange` on hover.
- Using `onMouseOver` on the container and computing the index from `e.target` (child glyphs confuse it).
- Half stars via separate glyphs/images and branching (`★½`) instead of percentage fill.
- Not clearing hover on `mouseleave` (preview sticks).
- Read-only mode still attaching handlers / focusable.
- Missing keyboard support and ARIA (a plain `<div onClick>`).
- Not clamping `value` to `[0, count]`.
- Forgetting controlled mode, so parent can't reset the rating.

---

## 12. Interview Tips & Time Plan

Show the `hover ?? rating` idea early; interviewers love the one-formula fill percentage for half/fractional stars.

| Minutes | Activity |
| --- | --- |
| 0–4 | Clarify (half stars? clear? controlled?), state shape |
| 4–12 | Render stars, click to set, whole stars |
| 12–18 | Hover preview + mouseleave |
| 18–26 | Half-star via pointer position + layered fill |
| 26–30 | Read-only mode & controlled/uncontrolled |
| 30–36 | Keyboard + ARIA slider |
| 36–40 | CSS polish, edge cases, follow-ups |

---

## 13. Practice Checklist

- [ ] Whole-star rating with click.
- [ ] Hover preview that resets on leave.
- [ ] Derived `display = hover ?? rating` (no duplicate state).
- [ ] Half stars via pointer position.
- [ ] Fractional read-only display (e.g. 3.7).
- [ ] Controlled + uncontrolled support.
- [ ] Keyboard: arrows, Home, End.
- [ ] ARIA slider + read-only `img`.
- [ ] "Click to clear" option.
- [ ] Re-implement with radio inputs and compare trade-offs.
