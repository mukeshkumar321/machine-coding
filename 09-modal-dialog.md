# Modal / Dialog — Machine Coding

> **Difficulty:** Medium | **Time:** 40–45 min | **Frequency:** ⭐⭐⭐⭐⭐

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

Build a reusable **Modal (Dialog)** component. It is rendered through a **portal**, dims the page with an overlay, **traps keyboard focus** inside, closes on **Esc** and **outside click**, **locks background scroll**, and **restores focus** to the trigger when closed.

---

## 2. Requirements

### Functional

- `open`, `onClose`, `title`, `children` props.
- Rendered via `createPortal` into `document.body`.
- Close on Esc, overlay click, and a close button.
- Focus moves into the modal on open; Tab / Shift+Tab cycle inside it.
- Background doesn't scroll while open (no layout shift).
- Focus returns to the previously focused element on close.

### Non-functional

- Reusable (any content); no global state needed.
- Correct ARIA (`role="dialog"`, `aria-modal`, `aria-labelledby`).
- Clean-up of all listeners and body styles — even when unmounted while open.
- Works inside containers with `overflow: hidden` / `transform` (hence the portal).

---

## 3. Clarifying Questions to Ask

- Should overlay click close it? (Often configurable — not for destructive forms.)
- Can modals nest / stack?
- Is there enter/exit animation?
- Confirm dialogs (`role="alertdialog"`) too?
- What should receive initial focus (first control, close button, or the dialog)?
- Do we need to support mobile Safari scroll locking?

---

## 4. Approach & Component Design

**State:** `open` lives in the **parent** (controlled). The modal itself is stateless except for refs.

```
App (open state)
 └── <Modal open onClose title>               ← returns null if !open
       └── createPortal(...) → document.body
            └── ModalContent                  ← mounts only while open ⇒ effects = lifecycle
                 ├── useScrollLock()
                 ├── useFocusTrap(dialogRef)
                 ├── Esc key listener
                 └── div.overlay (outside-click)
                      └── div[role=dialog]  (title, close btn, children)
```

| Concern | Technique |
| --- | --- |
| Escape stacking/clipping | `createPortal(…, document.body)` |
| Focus trap | Keydown on Tab: wrap first ↔ last focusable |
| Focus restore | Save `document.activeElement` on mount, `.focus()` it in cleanup |
| Scroll lock | `body.style.overflow='hidden'` + `padding-right = scrollbar width`; restore in cleanup |
| Outside click | `mousedown` target === overlay and `click` target === overlay |
| Esc | `document` `keydown` listener (cleanup on unmount) |

**Trick:** split `Modal` (returns `null` when closed) from `ModalContent` (all hooks) so effects run exactly on open/close — no `if (open)` conditions inside effects.

---

## 5. Reference Solution

<details>
<summary><strong>Show Full Code</strong></summary>

**`useScrollLock.js`**

```jsx
import { useEffect } from 'react';

export function useScrollLock() {
  useEffect(() => {
    const { body, documentElement } = document;
    const prevOverflow = body.style.overflow;
    const prevPaddingRight = body.style.paddingRight;
    const scrollbarWidth = window.innerWidth - documentElement.clientWidth;

    body.style.overflow = 'hidden';
    if (scrollbarWidth > 0) body.style.paddingRight = `${scrollbarWidth}px`; // avoid layout jump

    return () => {
      body.style.overflow = prevOverflow;
      body.style.paddingRight = prevPaddingRight;
    };
  }, []);
}
```

**`useFocusTrap.js`**

```jsx
import { useEffect } from 'react';

const FOCUSABLE = [
  'a[href]',
  'button:not([disabled])',
  'input:not([disabled]):not([type="hidden"])',
  'select:not([disabled])',
  'textarea:not([disabled])',
  '[tabindex]:not([tabindex="-1"])',
].join(',');

export function useFocusTrap(containerRef) {
  useEffect(() => {
    const node = containerRef.current;
    if (!node) return;

    const previouslyFocused = document.activeElement;
    const getFocusable = () => Array.from(node.querySelectorAll(FOCUSABLE));

    (getFocusable()[0] ?? node).focus();

    const handleKeyDown = (e) => {
      if (e.key !== 'Tab') return;
      const items = getFocusable();
      const first = items[0] ?? node;
      const last = items[items.length - 1] ?? node;
      const active = document.activeElement;
      const atEdge = e.shiftKey ? active === first || active === node : active === last;

      // Also pull focus back if it escaped the dialog (e.g. after an overlay click).
      if (!node.contains(active) || atEdge) {
        e.preventDefault();
        (e.shiftKey ? last : first).focus();
      }
    };

    document.addEventListener('keydown', handleKeyDown);
    return () => {
      document.removeEventListener('keydown', handleKeyDown);
      previouslyFocused?.focus?.(); // return focus to the trigger
    };
  }, [containerRef]);
}
```

**`Modal.jsx`**

```jsx
import { useEffect, useId, useRef } from 'react';
import { createPortal } from 'react-dom';
import { useScrollLock } from './useScrollLock';
import { useFocusTrap } from './useFocusTrap';
import './modal.css';

export default function Modal({ open, onClose, title, children, closeOnOverlayClick = true }) {
  if (!open) return null;
  return createPortal(
    <ModalContent
      onClose={onClose}
      title={title}
      closeOnOverlayClick={closeOnOverlayClick}
    >
      {children}
    </ModalContent>,
    document.body
  );
}

function ModalContent({ onClose, title, children, closeOnOverlayClick }) {
  const dialogRef = useRef(null);
  const pressStartedOnOverlay = useRef(false);
  const titleId = useId();

  useScrollLock();
  useFocusTrap(dialogRef);

  useEffect(() => {
    const handleKeyDown = (e) => {
      if (e.key === 'Escape') onClose();
    };
    document.addEventListener('keydown', handleKeyDown);
    return () => document.removeEventListener('keydown', handleKeyDown);
  }, [onClose]);

  return (
    <div
      className="modal-overlay"
      // Require press AND release on the overlay, so dragging a text selection
      // out of the dialog doesn't close it.
      onMouseDown={(e) => {
        pressStartedOnOverlay.current = e.target === e.currentTarget;
      }}
      onClick={(e) => {
        if (closeOnOverlayClick && pressStartedOnOverlay.current && e.target === e.currentTarget) {
          onClose();
        }
      }}
    >
      <div
        ref={dialogRef}
        className="modal"
        role="dialog"
        aria-modal="true"
        aria-labelledby={titleId}
        tabIndex={-1}
      >
        <header className="modal-header">
          <h2 id={titleId}>{title}</h2>
          <button type="button" className="modal-close" aria-label="Close dialog" onClick={onClose}>
            ✕
          </button>
        </header>
        <div className="modal-body">{children}</div>
      </div>
    </div>
  );
}
```

**`App.jsx`**

```jsx
import { useState } from 'react';
import Modal from './Modal';

export default function App() {
  const [open, setOpen] = useState(false);

  return (
    <main style={{ fontFamily: 'system-ui, sans-serif', padding: 24 }}>
      <button onClick={() => setOpen(true)}>Open modal</button>
      <div style={{ height: 1500 }}>Scroll the page to test scroll lock…</div>

      <Modal open={open} onClose={() => setOpen(false)} title="Edit profile">
        <label>
          Name <input type="text" />
        </label>
        <p>Tab / Shift+Tab stays inside. Esc or clicking outside closes.</p>
        <div className="modal-actions">
          <button onClick={() => setOpen(false)}>Cancel</button>
          <button onClick={() => setOpen(false)}>Save</button>
        </div>
      </Modal>
    </main>
  );
}
```

**`modal.css`**

```css
.modal-overlay {
  position: fixed; inset: 0; z-index: 1000;
  display: flex; align-items: center; justify-content: center;
  background: rgba(0, 0, 0, 0.5); padding: 16px;
  animation: fade-in 0.15s ease-out;
}
.modal {
  background: #fff; border-radius: 8px; width: min(480px, 100%);
  max-height: 85vh; display: flex; flex-direction: column;
  box-shadow: 0 20px 50px rgba(0, 0, 0, 0.3);
  animation: pop-in 0.15s ease-out;
}
.modal:focus { outline: none; }
.modal-header { display: flex; justify-content: space-between; align-items: center; padding: 16px 20px; border-bottom: 1px solid #eee; }
.modal-header h2 { margin: 0; font-size: 1.15rem; }
.modal-close { border: 0; background: none; font-size: 1.1rem; cursor: pointer; padding: 4px 8px; }
.modal-body { padding: 16px 20px; overflow-y: auto; overscroll-behavior: contain; }
.modal-actions { display: flex; justify-content: flex-end; gap: 8px; margin-top: 16px; }
button:focus-visible, input:focus-visible { outline: 2px solid #4f46e5; outline-offset: 2px; }

@keyframes fade-in { from { opacity: 0; } to { opacity: 1; } }
@keyframes pop-in { from { transform: scale(0.96); opacity: 0; } to { transform: none; opacity: 1; } }
@media (prefers-reduced-motion: reduce) {
  .modal-overlay, .modal { animation: none; }
}
```

</details>

---

## 6. How It Works

- **Portal:** DOM lives under `document.body`, escaping parent `overflow: hidden`, stacking contexts and `transform` clipping; React events/context still bubble through the *React* tree.
- **`Modal` / `ModalContent` split:** the content component only exists while open, so "mount = open, unmount = close". Each hook's setup/cleanup then maps 1:1 to the modal lifecycle (and survives parent unmounting the modal).
- **Focus trap:** on mount save `activeElement`, focus the first focusable (or dialog via `tabIndex=-1`). On Tab at the last element → jump to first; Shift+Tab at first → jump to last. Cleanup removes the listener and **restores focus**.
- **Scroll lock:** `overflow: hidden` on body; compensate scrollbar width with `padding-right` to prevent layout shift; cleanup restores the *previous* inline values (not blanks), so nested/stacked locks unwind correctly in LIFO order.
- **Esc:** `document` listener, removed in cleanup; `onClose` in deps keeps it fresh (no stale closure).
- **Outside click:** compare `e.target === e.currentTarget` (overlay itself, not the dialog's children), and require the press to start on the overlay too.
- **ARIA:** `role=dialog`, `aria-modal`, `aria-labelledby` bound to the title via `useId`.

---

## 7. Edge Cases

- Parent unmounts the modal while open → cleanups still restore scroll + focus.
- Dialog with **no focusable** elements → focus stays on the dialog container, Tab is swallowed.
- Content changes while open (new inputs appear) → `getFocusable()` queried on every Tab, so it stays correct.
- Text-selection drag ending on overlay → doesn't close (press-start check).
- Page has no scrollbar (`scrollbarWidth = 0`) → no padding added.
- Trigger element removed from DOM before close → `focus()` on a detached node is a harmless no-op.
- Esc inside a `<select>`/IME composition → check `e.isComposing` if needed.
- Nested modals → both Esc listeners fire (see follow-ups); scroll lock restores in LIFO order.
- Long content → body scrolls inside the modal (`max-height` + `overflow-y: auto`), not the page.
- React 18 StrictMode double effect → setup/cleanup/setup is balanced.
- Server rendering → `document` is only touched inside effects / after `open`; guard portal target for SSR.
- Mobile Safari ignores `overflow: hidden` on body in some versions → use `position: fixed` technique or `overscroll-behavior`.

---

## 8. Accessibility

- `role="dialog"` + `aria-modal="true"` + `aria-labelledby` (and `aria-describedby` for descriptive text); `role="alertdialog"` for confirmations that demand a response.
- Initial focus inside the dialog; Tab cycles; **focus restored** to the invoking element on close.
- Esc always closes (unless intentionally blocked, e.g. unsaved changes → confirm).
- Close button has an accessible name (`aria-label`).
- Background should be inert: `aria-modal` hides it from most AT; for full robustness set `inert` (or `aria-hidden`) on the app root while open.
- Respect `prefers-reduced-motion`; maintain colour contrast; visible focus ring.
- Don't autofocus a destructive action; for a delete confirmation, focus Cancel.

---

## 9. Performance Considerations

- Mounting only when open avoids rendering hidden content (use `keepMounted` when state must persist).
- Focusable queries run only on Tab presses — cheap.
- Animations use `opacity`/`transform` (compositor-friendly), not `top/left/height`.
- Lazy-load heavy modal content (`React.lazy`) so the initial bundle stays small.
- Avoid re-registering the Esc listener unnecessarily: wrap `onClose` in `useCallback` (or hold it in a ref).
- To animate exit you must delay unmount (state machine: `open → closing → unmounted`) — adds complexity.

---

## 10. Follow-up Questions

<details>
<summary><strong>1. Why use a portal at all?</strong></summary>

To render outside parents that clip (`overflow: hidden`), create stacking contexts (`z-index`, `transform`, `filter`) or change `position: fixed` behaviour. It guarantees the overlay covers the viewport and sits at the top of the stack. Events still bubble through React ancestors.

</details>

<details>
<summary><strong>2. How do you support stacked/nested modals?</strong></summary>

Keep a module-level **stack** of open modal ids; only the top one handles Esc / owns the focus trap / overlay click. Scroll lock should be ref-counted. Each modal gets increasing z-index (or rely on DOM order).

</details>

<details>
<summary><strong>3. How would you animate the exit?</strong></summary>

Keep it mounted while playing the CSS animation: `const [mounted, setMounted] = useState(open)`; when `open` becomes false apply a `closing` class and unmount on `onAnimationEnd`/`transitionend` (with a timeout fallback). Alternatively use `<dialog>` + `@starting-style`.

</details>

<details>
<summary><strong>4. Could you use the native <code>&lt;dialog&gt;</code> element?</strong></summary>

Yes: `dialog.showModal()` gives top-layer rendering, focus trapping, Esc handling, `::backdrop`, and inert background natively. Still needs scroll lock handling, outside-click logic, and React ref syncing (`useEffect` calling `showModal()/close()`). Interviewers usually want the manual version first.

</details>

<details>
<summary><strong>5. Why <code>inert</code> on the background rather than just a focus trap?</strong></summary>

A trap only handles the Tab key. Screen reader virtual cursors, mobile swipe navigation, and clicks can still reach the page behind. `inert` (or `aria-hidden` + focus management) removes it from interaction and the accessibility tree.

</details>

<details>
<summary><strong>6. Prevent closing with unsaved changes.</strong></summary>

Make `onClose` a *request*: parent checks `isDirty` and opens a nested confirm modal, or ignores overlay/Esc. Expose `closeOnEsc` / `closeOnOverlayClick` props.

</details>

<details>
<summary><strong>7. Why `mousedown` + `click` for the overlay?</strong></summary>

If a user presses inside an input, drags to select text and releases over the overlay, the browser fires `click` on the common ancestor (the overlay) — a naive `onClick` closes the modal and loses their input. Requiring the press to start on the overlay avoids it.

</details>

<details>
<summary><strong>8. Build an imperative API (<code>confirm()</code> returning a Promise)?</strong></summary>

A `ModalProvider` holds a queue/state and exposes `openModal(config) → Promise` through context; the provider renders the active modal and resolves/rejects the promise from its buttons.

</details>

---

## 11. Common Mistakes

- Rendering inline (no portal) → clipped by parents / wrong stacking.
- Not removing the Esc listener / not restoring `body.style.overflow` on unmount.
- Setting `overflow = ''` in cleanup, wiping the previous value (breaks stacked modals).
- Using `onClick` on the overlay with children bubbling → every click inside closes the modal (missing `target === currentTarget`).
- No focus management: focus stays on the page behind; not restored after close.
- Focus trap that queries focusables once at mount (misses dynamic content).
- Hard-coded ids for `aria-labelledby`.
- Missing `type="button"` on buttons inside forms.
- Scroll jump when body overflow toggles (no scrollbar compensation).
- Returning `null` via CSS only (`display: none`) while effects keep running.

---

## 12. Interview Tips & Time Plan

Announce the checklist up front: *portal, focus trap, focus restore, Esc, outside click, scroll lock, ARIA.* Then deliver them in order of effort/value.

| Minutes | Activity |
| --- | --- |
| 0–4 | Clarify; list the behaviours; draw component tree |
| 4–12 | Portal + overlay + close button + `open` prop |
| 12–18 | Esc + outside click (with `target === currentTarget`) |
| 18–24 | Scroll lock hook with cleanup + scrollbar compensation |
| 24–34 | Focus trap + focus restore |
| 34–39 | ARIA, `useId`, close button name |
| 39–45 | CSS polish, edge cases, nested/animation/`<dialog>` discussion |

---

## 13. Practice Checklist

- [ ] Portal to `document.body` with overlay and centered dialog.
- [ ] Esc closes; listener cleaned up.
- [ ] Overlay click closes only when clicking the overlay itself.
- [ ] Scroll lock with scrollbar compensation + proper restore.
- [ ] Focus moves in on open; Tab/Shift+Tab wrap; focus returns on close.
- [ ] `role="dialog"`, `aria-modal`, `aria-labelledby`.
- [ ] Works when unmounted while open (no leaks).
- [ ] Animate exit with a `closing` state.
- [ ] Add modal stack for nested dialogs.
- [ ] Re-implement with native `<dialog>` and compare.
