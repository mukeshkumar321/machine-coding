# Tooltip & Popover — Machine Coding

> **Difficulty:** Medium | **Time:** 45 min | **Frequency:** ⭐⭐⭐⭐

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

Build two reusable floating-UI components that share one positioning engine:

- **`<Tooltip>`** — a short hint that appears when the user **hovers or focuses** an element, with open/close **delays**, and dismisses with **Esc**.
- **`<Popover>`** — a richer panel opened by **clicking** a trigger and closed by an **outside click**, **Esc**, or tabbing away.

Both must stay on screen: when there isn't room on the preferred side they **flip**, when they'd overflow along the other axis they **shift**, and they keep following their anchor on **scroll / resize**. They render in a **portal** so `overflow: hidden` parents can't clip them.

---

## 2. Requirements

### Functional

- Tooltip opens on mouse hover (after `openDelay`) and on keyboard focus (immediately); closes after `closeDelay` on mouse leave / blur.
- If the pointer is over the trigger **or** the tooltip, or the trigger has focus, the tooltip stays open (hoverable content).
- `Esc` dismisses an open tooltip / popover without moving focus away from the trigger.
- Placement: `top | bottom | left | right`, an arrow pointing at the anchor.
- **Flip** to the opposite side when the preferred side doesn't fit; **shift** along the cross axis to remain inside the viewport (with padding).
- Reposition on window `scroll` (including scrolling ancestors) and `resize`.
- Popover: toggles on trigger click, closes on outside `pointerdown`, `Esc` (focus returns to the trigger) and when focus tabs out of it. Content may be a function receiving `close`.
- Rendered through a portal into `document.body`.

### Non-functional

- Pure, unit-testable `computePosition()` — no DOM access inside.
- No flicker: the panel is measured **before paint** (`useLayoutEffect`) and hidden until positioned.
- Listeners (scroll / resize / keydown / pointerdown) exist **only while open** and are cleaned up.
- Correct ARIA: `role="tooltip"` + `aria-describedby`; `aria-expanded` / `role="dialog"` for the popover.

---

## 3. Clarifying Questions to Ask

- Plain-text tooltips, or interactive/rich content? *(Plain text, but hoverable so users can reach it — WCAG 1.4.13.)*
- Which placements are required, and is **flip / shift** expected? *(Yes: all four, flip + shift.)*
- Touch devices: tooltips on long-press, or hide them? *(Hover tooltips are generally not for touch; popover is click-based.)*
- Default open / close delays? Is there a "warm-up" so moving between neighbouring tooltips is instant?
- Popover: modal (focus trap, scroll lock) or non-modal? *(Non-modal.)*
- Can the trigger be inside a scroll container, modal or transformed element?
- Controlled (`open` / `onOpenChange`) or uncontrolled? *(Uncontrolled for the exercise.)*
- Can I use a library (Floating UI, Radix)? *(Assume no — build the positioning.)*

---

## 4. Approach & Component Design

**State**

| State | Where | Notes |
|---|---|---|
| `open` | `Tooltip` / `Popover` | the only state the consumer-facing parts need |
| `hovered`, `focused`, `timer` | **refs** in `Tooltip` | changes must not re-render; used to decide *when* to close |
| `pos = { top, left, side, arrow } \| null` | `Floating` (inside the layer) | derived from DOM measurements; `null` = not measured yet. Lives in the component that is **mounted only while open**, so it resets for free |

**Positioning algorithm (`computePosition`)** — inputs are plain numbers, output is plain numbers.

```
1. space[side]  = room between the anchor and the viewport edge on that side
2. side = preferred
   if it doesn't fit (space - offset - size < padding) and the opposite side fits (or has more room) → FLIP
3. main axis:   top    → anchor.top  - offset - height      (bottom/left/right analogous)
   cross axis:  centre on the anchor, then CLAMP into [padding, viewport - size - padding]   ← SHIFT
4. arrow = anchor centre − popup origin, clamped inside the popup  (so it keeps pointing at the anchor after a shift)
```

```
  viewport
 ┌───────────────────────────────────────┐
 │ [anchor]      preferred 'top' doesn't │      anchor near the top edge:
 │               fit → flip to 'bottom'  │          ┌────────┐
 │   ┌─────────┐                         │          │ anchor │
 │   │ tooltip │ ◄── shifted right so it │          └───┬────┘
 │   └────▲────┘     stays on screen,    │          ▲ arrow     (flip)
 │        └ arrow still points at anchor │          tooltip below
 └───────────────────────────────────────┘
```

**Component tree**

```
Tooltip ─ <span class="tt-anchor"  hover/focus handlers>  { cloneElement(child, aria-describedby) }
        └ (open) Floating ──► createPortal → <div role="tooltip" class="floating tooltip">

Popover ─ <button aria-expanded aria-haspopup="dialog">        ← the anchor
        └ (open) Floating ──► createPortal → <div role="dialog" class="floating popover" tabIndex=-1>

Floating (shared): measure → computePosition → setPos; listens to scroll(capture) + resize
useOutsideClick (Popover only): document pointerdown, ignores trigger + panel
```

**Tooltip open/close rule:** open if *(pointer on trigger or tooltip)* **or** *(trigger focused)*. Leaving one trigger path only schedules a close when the **other** isn't active. Hover opens after `openDelay`; focus opens instantly; every close waits `closeDelay`, which also lets the pointer cross the gap onto the tooltip.

**Why `position: fixed` + viewport coordinates:** `getBoundingClientRect()` is viewport-relative, so no `scrollX/scrollY` arithmetic is needed in a body portal. The scroll listener keeps it glued to the anchor.

---

## 5. Reference Solution

<details>
<summary><strong>Show Full Code (computePosition.js, useOutsideClick.js, Floating.jsx, Tooltip.jsx, Popover.jsx, App.jsx, styles.css)</strong></summary>

```jsx
// computePosition.js — pure: numbers in, numbers out
const OPPOSITE = { top: 'bottom', bottom: 'top', left: 'right', right: 'left' };
const clamp = (v, min, max) => Math.max(min, Math.min(v, max));

/**
 * anchor:   { top, bottom, left, right, width, height }  (viewport coords, from getBoundingClientRect)
 * size:     { width, height }                            (the floating element)
 * viewport: { width, height }
 * returns   { top, left, side, arrow }   arrow = offset of the arrow along the cross axis
 */
export function computePosition(anchor, size, viewport, { placement = 'top', offset = 8, padding = 8 } = {}) {
  const space = {
    top: anchor.top,
    bottom: viewport.height - anchor.bottom,
    left: anchor.left,
    right: viewport.width - anchor.right,
  };
  const need = { top: size.height, bottom: size.height, left: size.width, right: size.width };
  const fits = (s) => space[s] - offset - need[s] >= padding;

  // flip only when it helps: the opposite side fits, or at least has more room
  let side = placement;
  if (!fits(side)) {
    const opp = OPPOSITE[side];
    if (fits(opp) || space[opp] > space[side]) side = opp;
  }

  let top;
  let left;
  let arrow;
  if (side === 'top' || side === 'bottom') {
    top = side === 'top' ? anchor.top - offset - size.height : anchor.bottom + offset;
    left = clamp(
      anchor.left + anchor.width / 2 - size.width / 2,
      padding,
      viewport.width - size.width - padding
    );
    arrow = clamp(anchor.left + anchor.width / 2 - left, 12, Math.max(12, size.width - 12));
  } else {
    left = side === 'left' ? anchor.left - offset - size.width : anchor.right + offset;
    top = clamp(
      anchor.top + anchor.height / 2 - size.height / 2,
      padding,
      viewport.height - size.height - padding
    );
    arrow = clamp(anchor.top + anchor.height / 2 - top, 12, Math.max(12, size.height - 12));
  }

  return { top: Math.round(top), left: Math.round(left), side, arrow: Math.round(arrow) };
}
```

```jsx
// useOutsideClick.js
import { useEffect, useRef } from 'react';

// refs: array of refs that count as "inside". Listener exists only while `enabled`.
export function useOutsideClick(refs, handler, enabled) {
  const latest = useRef({ refs, handler });

  // keep the latest values without re-subscribing the document listener
  useEffect(() => {
    latest.current = { refs, handler };
  });

  useEffect(() => {
    if (!enabled) return undefined;
    const listener = (e) => {
      const { refs: list, handler: fn } = latest.current;
      if (list.some((r) => r.current && r.current.contains(e.target))) return;
      fn(e);
    };
    document.addEventListener('pointerdown', listener);
    return () => document.removeEventListener('pointerdown', listener);
  }, [enabled]);
}
```

```jsx
// Floating.jsx — portal + measure + position + follow on scroll/resize
import { useCallback, useLayoutEffect, useState } from 'react';
import { createPortal } from 'react-dom';
import { computePosition } from './computePosition';

export default function Floating({
  anchorRef,
  floatRef,
  placement = 'top',
  offset = 8,
  className = '',
  children,
  ...rest // id, role, aria-*, handlers, tabIndex
}) {
  const [pos, setPos] = useState(null); // null until measured

  const update = useCallback(() => {
    const anchor = anchorRef.current;
    const el = floatRef.current;
    if (!anchor || !el) return;
    const next = computePosition(
      anchor.getBoundingClientRect(),
      { width: el.offsetWidth, height: el.offsetHeight },
      { width: window.innerWidth, height: window.innerHeight },
      { placement, offset }
    );
    // bail out when nothing moved, so scroll events don't cause pointless renders
    setPos((prev) =>
      prev &&
      prev.top === next.top &&
      prev.left === next.left &&
      prev.side === next.side &&
      prev.arrow === next.arrow
        ? prev
        : next
    );
  }, [anchorRef, floatRef, placement, offset]);

  // layout effect: measure + position before the browser paints (no flash at 0,0)
  useLayoutEffect(() => {
    update();
    window.addEventListener('resize', update);
    // capture: scroll events don't bubble, this also catches scrolling ancestors
    window.addEventListener('scroll', update, true);
    return () => {
      window.removeEventListener('resize', update);
      window.removeEventListener('scroll', update, true);
    };
  }, [update]);

  const side = pos ? pos.side : placement;
  const horizontal = side === 'top' || side === 'bottom';

  return createPortal(
    <div
      {...rest}
      ref={floatRef}
      className={`floating ${className}`}
      data-side={side}
      // opacity (not visibility) so the popover can still receive focus on its first frame
      style={{ top: pos ? pos.top : 0, left: pos ? pos.left : 0, opacity: pos ? 1 : 0 }}
    >
      {children}
      {pos && (
        <span
          className="arrow"
          aria-hidden="true"
          style={horizontal ? { left: pos.arrow - 5 } : { top: pos.arrow - 5 }}
        />
      )}
    </div>,
    document.body
  );
}
```

```jsx
// Tooltip.jsx
import { Children, cloneElement, useEffect, useId, useRef, useState } from 'react';
import Floating from './Floating';

export default function Tooltip({
  content,
  children,
  placement = 'top',
  openDelay = 300,
  closeDelay = 100,
}) {
  const id = useId();
  const anchorRef = useRef(null);
  const floatRef = useRef(null);
  const timer = useRef(null);
  const hovered = useRef(false); // pointer is on the trigger OR on the tooltip
  const focused = useRef(false);
  const [open, setOpen] = useState(false);

  const clear = () => clearTimeout(timer.current);
  const schedule = (next, delay) => {
    clear();
    if (delay > 0) timer.current = setTimeout(() => setOpen(next), delay);
    else setOpen(next);
  };
  useEffect(() => () => clearTimeout(timer.current), []);

  const onEnter = () => {
    hovered.current = true;
    schedule(true, open ? 0 : openDelay);
  };
  const onLeave = () => {
    hovered.current = false;
    if (!focused.current) schedule(false, closeDelay);
  };
  const onFocus = () => {
    focused.current = true;
    schedule(true, 0);
  };
  const onBlur = () => {
    focused.current = false;
    if (!hovered.current) schedule(false, closeDelay);
  };

  // Esc dismisses (WCAG 1.4.13) — listener only while open
  useEffect(() => {
    if (!open) return undefined;
    const onKey = (e) => {
      if (e.key === 'Escape') {
        clearTimeout(timer.current);
        setOpen(false);
      }
    };
    document.addEventListener('keydown', onKey);
    return () => document.removeEventListener('keydown', onKey);
  }, [open]);

  const child = Children.only(children);
  const describedBy =
    [child.props['aria-describedby'], open ? id : null].filter(Boolean).join(' ') || undefined;

  return (
    <>
      <span
        ref={anchorRef}
        className="tt-anchor"
        onMouseEnter={onEnter}
        onMouseLeave={onLeave}
        onFocus={onFocus}
        onBlur={onBlur}
      >
        {cloneElement(child, { 'aria-describedby': describedBy })}
      </span>
      {open && (
        <Floating
          id={id}
          role="tooltip"
          className="tooltip"
          anchorRef={anchorRef}
          floatRef={floatRef}
          placement={placement}
          onMouseEnter={() => {
            // moving from the trigger onto the tooltip must not close it
            hovered.current = true;
            clear();
          }}
          onMouseLeave={onLeave}
        >
          {content}
        </Floating>
      )}
    </>
  );
}
```

```jsx
// Popover.jsx
import { useEffect, useId, useRef, useState } from 'react';
import Floating from './Floating';
import { useOutsideClick } from './useOutsideClick';

const FOCUSABLE = 'a[href], button:not([disabled]), input:not([disabled]), select, textarea, [tabindex]:not([tabindex="-1"])';

export default function Popover({ trigger, label, children, placement = 'bottom' }) {
  const id = useId();
  const [open, setOpen] = useState(false);
  const triggerRef = useRef(null);
  const floatRef = useRef(null);

  const closeAndRestoreFocus = () => {
    setOpen(false);
    triggerRef.current?.focus();
  };

  // outside pointerdown closes (don't steal focus back: the user is going elsewhere)
  useOutsideClick([triggerRef, floatRef], () => setOpen(false), open);

  useEffect(() => {
    if (!open) return undefined;
    const onKey = (e) => {
      if (e.key === 'Escape') {
        setOpen(false);
        triggerRef.current?.focus();
      }
    };
    document.addEventListener('keydown', onKey);
    return () => document.removeEventListener('keydown', onKey);
  }, [open]);

  // move focus into the panel so keyboard users can reach its content
  useEffect(() => {
    if (open) floatRef.current?.focus({ preventScroll: true });
  }, [open]);

  // The portal sits at the end of <body>, so natural Tab order would skip it / lose the user.
  // Tab off either end -> close and hand focus back to the trigger.
  const onPanelKeyDown = (e) => {
    if (e.key !== 'Tab') return;
    const items = [...floatRef.current.querySelectorAll(FOCUSABLE)];
    const active = document.activeElement;
    const atEnd = !e.shiftKey && (items.length === 0 || active === items[items.length - 1]);
    const atStart =
      e.shiftKey && (items.length === 0 || active === items[0] || active === floatRef.current);
    if (atEnd || atStart) {
      e.preventDefault();
      closeAndRestoreFocus();
    }
  };

  return (
    <>
      <button
        ref={triggerRef}
        type="button"
        className="btn"
        aria-haspopup="dialog"
        aria-expanded={open}
        aria-controls={open ? id : undefined}
        onClick={() => setOpen((o) => !o)}
      >
        {trigger}
      </button>
      {open && (
        <Floating
          id={id}
          role="dialog"
          aria-label={label}
          tabIndex={-1}
          className="popover"
          anchorRef={triggerRef}
          floatRef={floatRef}
          placement={placement}
          onKeyDown={onPanelKeyDown}
        >
          {typeof children === 'function' ? children({ close: closeAndRestoreFocus }) : children}
        </Floating>
      )}
    </>
  );
}
```

```jsx
// App.jsx
import Tooltip from './Tooltip';
import Popover from './Popover';
import './styles.css';

export default function App() {
  return (
    <main className="page">
      <h1>Tooltip &amp; Popover</h1>

      <div className="row">
        <Tooltip content="Save your changes (Ctrl+S)">
          <button className="btn">Save</button>
        </Tooltip>
        <Tooltip content="Appears to the right" placement="right">
          <button className="btn">Right</button>
        </Tooltip>
        <Popover trigger="Share" label="Share this page">
          {({ close }) => (
            <div className="share">
              <strong>Share this page</strong>
              <button className="btn">Copy link</button>
              <button className="btn">Email</button>
              <button className="btn" onClick={close}>Done</button>
            </div>
          )}
        </Popover>
      </div>

      {/* edge cases: near the viewport corners the tooltip flips and shifts */}
      <div className="corner tl">
        <Tooltip content="Top-left: flips below and shifts right to stay on screen">
          <button className="btn">Top-left</button>
        </Tooltip>
      </div>
      <div className="corner br">
        <Tooltip content="Bottom-right: flips above and shifts left">
          <button className="btn">Bottom-right</button>
        </Tooltip>
      </div>
    </main>
  );
}
```

```css
/* styles.css */
.page { max-width: 640px; margin: 80px auto; font-family: system-ui, sans-serif; }
.row { display: flex; gap: 16px; align-items: center; }
.corner { position: fixed; }
.corner.tl { top: 8px; left: 8px; }
.corner.br { bottom: 8px; right: 8px; }
.btn { padding: 8px 14px; border: 1px solid #cbd5e1; border-radius: 6px; background: #fff; cursor: pointer; font: inherit; }
.btn:focus-visible { outline: 2px solid #2563eb; outline-offset: 2px; }
.tt-anchor { display: inline-block; }

.floating { position: fixed; z-index: 1000; box-sizing: border-box; width: max-content; max-width: min(260px, calc(100vw - 16px)); }
.tooltip { --bg: #111827; --bd: #111827; background: var(--bg); color: #fff; padding: 6px 10px; border-radius: 6px; font-size: 13px; line-height: 1.4; }
.popover { --bg: #fff; --bd: #cbd5e1; background: var(--bg); color: #111827; border: 1px solid var(--bd); border-radius: 8px; padding: 12px; box-shadow: 0 8px 24px rgba(0, 0, 0, 0.15); }
.popover:focus { outline: none; }
.share { display: flex; flex-direction: column; gap: 8px; }

.arrow { position: absolute; width: 10px; height: 10px; box-sizing: border-box; background: var(--bg); transform: rotate(45deg); border: 0 solid var(--bd); }
.floating[data-side='top'] .arrow { bottom: -5px; border-right-width: 1px; border-bottom-width: 1px; }
.floating[data-side='bottom'] .arrow { top: -5px; border-left-width: 1px; border-top-width: 1px; }
.floating[data-side='left'] .arrow { right: -5px; border-top-width: 1px; border-right-width: 1px; }
.floating[data-side='right'] .arrow { left: -5px; border-bottom-width: 1px; border-left-width: 1px; }
```

</details>

---

## 6. How It Works

- **Pure positioning.** `computePosition` takes the anchor rect, the floating element's size and the viewport and returns `{ top, left, side, arrow }`. Because it never touches the DOM it is trivial to unit-test (that's where the flip/shift cases are verified).
- **Flip.** `fits(side)` checks `space − offset − size ≥ padding`. If the preferred side doesn't fit, switch to the opposite one when it fits (or at least has more room). If *neither* fits, the roomier side wins instead of overflowing off-screen.
- **Shift.** Along the cross axis the popup is centred on the anchor and then **clamped** to `[padding, viewport − size − padding]`. The arrow offset is computed *after* the shift (`anchorCentre − left`) so it still points at the anchor.
- **Measure-then-show.** `Floating` mounts with `opacity: 0` at `(0,0)`; `useLayoutEffect` reads `getBoundingClientRect()` + `offsetWidth/Height`, calls `computePosition`, and sets `pos` **before paint** — the user never sees a jump. State lives in `Floating`, which only exists while open, so the next open starts fresh.
- **Following the anchor.** `scroll` is listened to on `window` with `capture: true` (scroll doesn't bubble, capture also catches scrolling ancestors) plus `resize`. The setState bails out when the numbers are unchanged. Everything is removed on unmount.
- **`position: fixed` + portal.** Rects are viewport-relative, so no scroll-offset math; the portal into `body` escapes `overflow: hidden` and stacking contexts of ancestors.
- **Tooltip timing.** Timer id, `hovered` and `focused` are **refs** (no re-render). Hover opens after `openDelay`, focus immediately; a close is only scheduled when the *other* trigger isn't active. Entering the tooltip itself cancels the pending close, so the content is hoverable. `Esc` is a document listener that exists only while open.
- **ARIA wiring.** `cloneElement` adds `aria-describedby="<id>"` (merged with an existing one) to the actual focusable child, only while the tooltip exists; the layer has `role="tooltip"` and that id.
- **Popover.** Click toggles `open`. `useOutsideClick` listens to `pointerdown` on `document` only while open and ignores the trigger (so the trigger's own click can toggle) and the panel. `Esc` closes and returns focus to the trigger; opening focuses the panel; Tab off either end closes and returns to the trigger because the portal is at the end of the DOM.

---

## 7. Edge Cases

- **Anchor near any viewport edge** → flip, then shift; arrow still points at the anchor.
- **No room on either side** → choose the side with more space rather than overflowing.
- **Floating element wider than the viewport** → clamp falls back to `padding`; `max-width: min(260px, 100vw − 16px)` keeps it bounded.
- **Scrolling** (page *or* any scrollable ancestor) / **resize** → repositioned via capture listener; **anchor scrolled out of view** → hide or close it (follow-up).
- **Content size changes while open** → measured only on open/scroll/resize; use `ResizeObserver` on the floating element for dynamic content.
- **Pointer crosses the gap** between trigger and tooltip → `closeDelay` plus entering the tooltip cancels the close.
- **Hover ends while the trigger is still focused** (or vice versa) → stays open; closes only when both are gone.
- **`Esc` while hovering** → closes and stays closed until the pointer leaves and re-enters or focus moves again.
- **Trigger unmounts / component unmounts with a pending timer** → cleared in the unmount effect (no setState on an unmounted component).
- **Disabled buttons** don't fire mouse/focus events in some browsers → wrap in the span anchor (already done) and don't rely on the disabled element itself.
- **Rapid enter/leave** → each `schedule` clears the previous timer; only the latest intent wins.
- **Touch devices** → hover tooltips don't exist; don't hide essential info in them.
- **Page scrollbar** → `innerWidth` includes it; use `documentElement.clientWidth` if tooltips tuck under the scrollbar.
- **Popover click on the trigger while open** → inside the trigger ref, so outside-click ignores it and the toggle closes it (no close-then-reopen flicker).
- **Nested popovers/tooltips inside a popover** → they are separate portals, so the outer outside-click would treat them as "outside"; register their refs as inside, or use a layer stack.
- **Modal parents** → portal to the modal's container (or raise `z-index`) so it's above the overlay.

---

## 8. Accessibility

- Tooltip: the layer is `role="tooltip"` with a unique `id`; the trigger gets `aria-describedby` pointing at it **while it is rendered**. It supplements the accessible name, it is not a replacement (don't put essential info only in a tooltip).
- Tooltip must be reachable by **keyboard focus** (shows on focus, hides on blur) and not require a mouse.
- **WCAG 1.4.13 (Content on Hover or Focus):** *dismissible* (Esc, without moving pointer/focus), *hoverable* (pointer can move onto the tooltip), *persistent* (stays until hover/focus is removed or the user dismisses it).
- Don't put interactive content in a tooltip — that's a popover.
- Popover: trigger has `aria-haspopup="dialog"`, `aria-expanded`, `aria-controls` (while open); panel is `role="dialog"` with `aria-label`, receives focus on open, and `Esc` returns focus to the trigger.
- Non-modal dialog: the portal is at the end of the DOM, so tabbing out must be handled explicitly (done in `onPanelKeyDown`) — otherwise focus escapes to the browser UI.
- The decorative arrow is `aria-hidden`.
- Respect `prefers-reduced-motion` if you add fade/scale animations; don't animate position changes while scrolling.
- Sufficient contrast (white on `#111827`) and visible `:focus-visible` rings on triggers.

---

## 9. Performance Considerations

- Listeners (`scroll`, `resize`, `keydown`, `pointerdown`) exist **only while open**; closed tooltips cost nothing (no layer mounted, no effects).
- `scroll` can fire very often: the `setPos` bail-out avoids re-renders when nothing moved; for heavier cases throttle with `requestAnimationFrame` (schedule one update per frame, cancel on cleanup).
- Reads (`getBoundingClientRect`, `offsetWidth`) are batched in one place and done in a layout effect, then **one** write (`setPos`) — no read/write interleaving → no layout thrashing.
- Timers and hover/focus flags are refs → hovering never re-renders the owner until `open` actually flips.
- For lists with hundreds of tooltips, don't mount a `Tooltip` state machine per item: use event delegation with one shared tooltip instance (`data-tooltip` attributes) and a single set of listeners.
- Use `transform: translate()` instead of `top/left` if you animate position (compositor-only); `top/left` is fine for static placement.
- Avoid recomputing on every render — `update` is memoised on `[placement, offset, refs]`.

---

## 10. Follow-up Questions

<details>
<summary><strong>1. Why a portal and `position: fixed` instead of `position: absolute` inside the parent?</strong></summary>

An absolutely positioned child is clipped by any ancestor with `overflow: hidden/auto` and trapped in its stacking context (a low `z-index` parent can sit under later siblings). A portal into `body` escapes both. With `fixed`, coordinates from `getBoundingClientRect()` can be used directly (no `scrollX/Y` math) — at the price of having to reposition on scroll. Because the layer lives directly under `body`, a transformed ancestor of the trigger can't break `fixed` positioning either.

</details>

<details>
<summary><strong>2. How would you add `start` / `end` alignment (e.g. `bottom-start`) and a middleware-style API?</strong></summary>

Add `align` to the cross-axis calculation: `start` → `left = anchor.left`, `end` → `left = anchor.right - size.width`, `center` as now — still followed by the clamp. Floating UI generalises this as a pipeline of middlewares (`offset → flip → shift → arrow`), each taking and returning `{x, y, placement}`; ours is the same thing hard-coded.

</details>

<details>
<summary><strong>3. How do you hide or close it when the anchor scrolls out of view?</strong></summary>

In `update`, compare the anchor rect to its scroll container / viewport (`IntersectionObserver` on the anchor with the scroll root). If it's not intersecting, either close, or keep it but set `visibility: hidden`. Otherwise the tooltip floats over unrelated content.

</details>

<details>
<summary><strong>4. How do you add a "warm-up" so moving between neighbouring tooltips is instant?</strong></summary>

Keep a module-level `lastClosedAt` timestamp. On enter, if another tooltip closed within ~300–500 ms use `openDelay = 0`. Also close the previously open tooltip first (a tiny singleton or context that stores the active id) so two never display at once.

</details>

<details>
<summary><strong>5. What changes for touch devices?</strong></summary>

There is no hover. Either show the tooltip on long-press (and block the context menu) or convert the hint into a click/tap popover, and never make a tooltip the only way to learn something. Focus-based open still works for keyboard and screen-reader users.

</details>

<details>
<summary><strong>6. Make the popover modal (focus trap, scroll lock, inert background).</strong></summary>

Add `aria-modal="true"`, a backdrop, a focus trap (Tab cycles inside; restore focus to the trigger on close), `overflow: hidden` on `body` or the `inert` attribute on the rest of the page. At that point you're building a dialog — usually a different component than a popover.

</details>

<details>
<summary><strong>7. Why `pointerdown` instead of `click` for outside detection, and why ignore the trigger?</strong></summary>

`pointerdown` fires before focus moves and before other handlers can remove the target from the DOM (which would make `contains` wrongly report "outside"); it also covers mouse, touch and pen. The trigger must be ignored, otherwise `pointerdown` closes the popover and the following `click` toggles it open again.

</details>

<details>
<summary><strong>8. How do you make `useLayoutEffect` safe with SSR?</strong></summary>

`useLayoutEffect` warns on the server and the portal needs `document`. Since the floating layer only renders when `open` (which is only set by client events), it never renders on the server. If you render it open initially, gate on a `mounted` flag (set in `useEffect`) or use an isomorphic layout-effect hook (`typeof window !== 'undefined' ? useLayoutEffect : useEffect`).

</details>

---

## 11. Common Mistakes

- Positioning with `absolute` inside an `overflow: hidden` parent → the tooltip is clipped.
- Measuring **before** the element is in the DOM / without a layout effect → flash at the wrong spot, or `0×0` size.
- Using page coordinates and viewport coordinates interchangeably (forgetting `scrollY` with `absolute`, or adding it with `fixed`).
- No flip/shift: tooltips cut off at the viewport edge.
- Clamping with `Math.min(Math.max(v, min), max)` when `max < min` (tooltip wider than viewport) → pushes the popup off the **left**.
- Forgetting to remove `scroll` / `resize` / `keydown` / `pointerdown` listeners; or attaching them while closed.
- Listening for `scroll` without `capture` → scrolling ancestors don't trigger updates.
- Closing instantly on `mouseleave` so the tooltip can't be hovered; or no `closeDelay` flicker between trigger and tooltip.
- Not clearing the timeout on unmount / on a new enter → tooltip opens after the mouse has already left.
- Tooltip only on hover — keyboard users never see it (no focus handler).
- Setting `aria-describedby` on the wrapper instead of the focusable element, or pointing at an id that doesn't exist.
- Popover: closing on `click` outside **and** on the trigger's own click → opens then immediately closes.
- Popover in a portal without handling Tab → focus gets lost at the end of the document.
- Storing hover/focus flags in `useState` → re-render on every mouse movement.
