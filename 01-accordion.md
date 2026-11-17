# Accordion — Machine Coding

> **Difficulty:** Easy | **Time:** 30–40 min | **Asked at / Frequency:** Very common (FAQ / settings-panel style question) — ⭐⭐⭐⭐⭐ | **Tier:** 1 (must know)

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

Build a reusable **Accordion** component that takes a list of `{ id, title, content }` items. Clicking a header expands/collapses its panel. It must support two modes:

- **Single-open** — opening one panel closes the others (and the open one can be collapsed).
- **Multi-open** — any number of panels can be open at once.

It must follow WAI-ARIA accordion practices (keyboard + screen reader support).

---

## 2. Requirements

### Functional

- Render items from props; header toggles its panel.
- `allowMultiple` prop switches between modes.
- Optional `defaultOpenIds` for the initial state.
- Keyboard: `Enter`/`Space` toggle, `↑`/`↓` move between headers, `Home`/`End` jump to first/last.
- Smooth expand/collapse animation (CSS only).

### Non-functional

- Reusable, data-driven API; no per-item hardcoding.
- Correct ARIA (`aria-expanded`, `aria-controls`, `aria-labelledby`).
- No layout jump / no JS height measuring.
- Respects `prefers-reduced-motion`.

---

## 3. Clarifying Questions to Ask

- In single mode, can the open item be collapsed (collapsible) or must one always stay open? (Assume collapsible.)
- Controlled or uncontrolled? (Assume uncontrolled with `defaultOpenIds`; mention controlled variant.)
- Can the content be arbitrary JSX? (Yes → `content` is a node.)
- Are there disabled items? Nested accordions?
- Animate? Should closed content remain in the DOM (for find-in-page/SEO)?
- Which heading level fits the page outline?

---

## 4. Approach & Component Design

**State shape:** one array of open ids — works for both modes.

```js
openIds: string[]      // single mode → length ≤ 1;  multi mode → any length
```

Why an array (not `openIndex`)? One representation handles both modes; ids survive reordering; `includes` is trivial for small lists (use a `Set` for very large).

**Toggle logic**

| Mode | Item closed | Item open |
| --- | --- | --- |
| single | `[id]` | `[]` |
| multi | `[...prev, id]` | `prev.filter(x => x !== id)` |

```
Accordion (openIds, buttonRefs)
 └── for each item
      ├── <h3>
      │     └── <button aria-expanded aria-controls>   ← header (focusable)
      └── <div role="region" aria-labelledby>          ← panel
            └── content
```

---

## 5. Reference Solution

<details>
<summary><strong>Show Full Code</strong></summary>

**`Accordion.jsx`**

```jsx
import { useId, useRef, useState } from 'react';
import './accordion.css';

export default function Accordion({
  items,
  allowMultiple = false,
  defaultOpenIds = [],
  headingLevel = 3,
}) {
  const baseId = useId();
  const buttonRefs = useRef([]);
  const [openIds, setOpenIds] = useState(defaultOpenIds);
  const Heading = `h${headingLevel}`;

  // If the mode is switched at runtime, never show more than one in single mode
  const effectiveOpenIds = allowMultiple ? openIds : openIds.slice(0, 1);

  const toggle = (id) => {
    const isOpen = effectiveOpenIds.includes(id);
    if (allowMultiple) {
      setOpenIds(isOpen ? effectiveOpenIds.filter((x) => x !== id) : [...effectiveOpenIds, id]);
    } else {
      setOpenIds(isOpen ? [] : [id]);
    }
  };

  const handleKeyDown = (e, index) => {
    const last = items.length - 1;
    let next;
    switch (e.key) {
      case 'ArrowDown': next = index === last ? 0 : index + 1; break;
      case 'ArrowUp':   next = index === 0 ? last : index - 1; break;
      case 'Home':      next = 0; break;
      case 'End':       next = last; break;
      default: return;
    }
    e.preventDefault();
    buttonRefs.current[next]?.focus();
  };

  return (
    <div className="accordion">
      {items.map((item, index) => {
        const isOpen = effectiveOpenIds.includes(item.id);
        const buttonId = `${baseId}-header-${item.id}`;
        const panelId = `${baseId}-panel-${item.id}`;

        return (
          <div className={`accordion-item ${isOpen ? 'open' : ''}`} key={item.id}>
            <Heading className="accordion-heading">
              <button
                type="button"
                id={buttonId}
                className="accordion-header"
                aria-expanded={isOpen}
                aria-controls={panelId}
                ref={(el) => (buttonRefs.current[index] = el)}
                onClick={() => toggle(item.id)}
                onKeyDown={(e) => handleKeyDown(e, index)}
              >
                <span>{item.title}</span>
                <span className="accordion-icon" aria-hidden="true">▾</span>
              </button>
            </Heading>

            <div
              id={panelId}
              role="region"
              aria-labelledby={buttonId}
              className="accordion-panel"
            >
              <div className="accordion-panel-inner">{item.content}</div>
            </div>
          </div>
        );
      })}
    </div>
  );
}
```

**`App.jsx`**

```jsx
import { useState } from 'react';
import Accordion from './Accordion';

const ITEMS = [
  { id: 'what', title: 'What is React?', content: <p>A library for building UIs.</p> },
  { id: 'hooks', title: 'What are hooks?', content: <p>Functions that let you use state and effects in function components.</p> },
  { id: 'css', title: 'Why plain CSS?', content: <p>Fewer moving parts in a machine-coding round.</p> },
];

export default function App() {
  const [multi, setMulti] = useState(false);

  return (
    <main style={{ maxWidth: 520, margin: '40px auto', fontFamily: 'system-ui, sans-serif' }}>
      <label>
        <input type="checkbox" checked={multi} onChange={(e) => setMulti(e.target.checked)} />{' '}
        Allow multiple open
      </label>
      <Accordion items={ITEMS} allowMultiple={multi} defaultOpenIds={['what']} />
    </main>
  );
}
```

**`accordion.css`**

```css
.accordion { border: 1px solid #ddd; border-radius: 8px; overflow: hidden; margin-top: 16px; }
.accordion-item + .accordion-item { border-top: 1px solid #ddd; }
.accordion-heading { margin: 0; font-size: inherit; }

.accordion-header {
  width: 100%; display: flex; justify-content: space-between; align-items: center;
  padding: 14px 16px; background: #fafafa; border: 0; font: inherit; font-weight: 600;
  text-align: left; cursor: pointer;
}
.accordion-header:hover { background: #f0f0f0; }
.accordion-header:focus-visible { outline: 2px solid #4f46e5; outline-offset: -2px; }

.accordion-icon { transition: transform 0.25s ease; }
.accordion-item.open .accordion-icon { transform: rotate(180deg); }

/* Animate height without measuring: 0fr -> 1fr */
.accordion-panel {
  display: grid;
  grid-template-rows: 0fr;
  visibility: hidden; /* removes closed content from tab order + a11y tree */
  transition: grid-template-rows 0.25s ease, visibility 0.25s;
}
.accordion-item.open .accordion-panel { grid-template-rows: 1fr; visibility: visible; }
.accordion-panel-inner { overflow: hidden; min-height: 0; padding: 0 16px; }
.accordion-item.open .accordion-panel-inner { padding: 12px 16px; }

@media (prefers-reduced-motion: reduce) {
  .accordion-panel, .accordion-icon { transition: none; }
}
```

</details>

---

## 6. How It Works

- **Single array of ids** covers both modes; the only difference is the toggle function.
- **Derived `effectiveOpenIds`** keeps single mode honest if `allowMultiple` flips at runtime without extra effects/sync.
- **Immutable updates** — `filter` and spread; never `push`.
- **Stable keys** — `item.id`.
- **ARIA wiring** — `useId` creates a unique prefix so two accordions on one page don't clash; button `aria-controls` → panel id, panel `aria-labelledby` → button id.
- **Native `<button>`** gives Enter/Space + focus for free; only arrow/Home/End navigation is custom, using a `refs` array to move focus.
- **CSS-only animation** — `grid-template-rows: 0fr → 1fr` animates to the content's natural height with no `scrollHeight` measuring; `visibility: hidden` ensures closed content isn't focusable or announced.
- **Heading wrapper** — the button sits inside `h3` (configurable level) so screen reader users can navigate by headings.

---

## 7. Edge Cases

- Empty `items` → renders an empty container; keyboard handler never runs.
- `defaultOpenIds` with several ids in single mode → only first shown.
- Duplicate ids → React key warnings + broken ARIA (validate / document unique ids).
- Item removed while open → id lingers in state; harmless, but could prune.
- Content containing links/inputs: closed panel's controls must not be tabbable (`visibility: hidden` / `hidden` / `inert`).
- Very long content → panel grows; no max-height magic number needed with grid trick.
- Rapid toggling mid-animation → CSS transitions interrupt cleanly.
- Nested accordions → `useId` keeps ids unique; arrow keys only move within the *outer* `items`.
- Disabled items → skip them in arrow navigation (see follow-ups).

---

## 8. Accessibility

- Header is a `<button>` in a heading with `aria-expanded` (true/false) and `aria-controls`.
- Panel `role="region"` + `aria-labelledby` (WAI-ARIA recommends **omitting** `region` if there are more than ~6 panels, to avoid landmark noise).
- Keyboard: Tab to each header, Enter/Space toggles, ↑/↓/Home/End are optional-but-expected enhancements.
- Visible `:focus-visible` style; contrast ≥ 4.5:1.
- Icon is `aria-hidden`; state isn't conveyed only through rotation/colour.
- `prefers-reduced-motion` honoured.
- If a variant requires one panel to always stay open, mark that open header `aria-disabled="true"` (per the APG pattern).

---

## 9. Performance Considerations

- Cheap: toggling re-renders the accordion; with big lists, wrap an `AccordionItem` in `memo` and pass `isOpen` + stable `onToggle(id)`.
- Closed panels are still rendered — for expensive content **lazy-render** (`isOpen && content`, or keep-mounted-after-first-open).
- Animating `grid-template-rows` triggers layout, but on a small subtree it's fine; avoid animating `height` via JS measurement on every frame.
- `includes` on a small array is faster than a `Set` in practice; switch only for hundreds of items.

---

## 10. Follow-up Questions

<details>
<summary><strong>1. Make it controlled.</strong></summary>

Accept `openIds` + `onOpenChange(nextIds)`. Compute `nextIds` with the same toggle logic and call the callback; use internal state only when `openIds` is `undefined` (the same controlled/uncontrolled hook used for Tabs).

</details>

<details>
<summary><strong>2. Lazy-render panel content / keep state of closed panels?</strong></summary>

Unmounting on close (`isOpen && children`) frees memory but loses inner state (form inputs). Compromise: render once on first open, then keep mounted (track `hasOpened`).

</details>

<details>
<summary><strong>3. How do you support disabled items in keyboard nav?</strong></summary>

Build `enabledIndexes = items.map((it,i)=>!it.disabled && i).filter(i=>i!==false)`, find the current position in it, move to the next/prev with wraparound. Use `disabled` on the button (not focusable) or `aria-disabled` if it should stay discoverable.

</details>

<details>
<summary><strong>4. Animate height with JS instead of the grid trick?</strong></summary>

Measure `scrollHeight`, set `height` to it, transition, then set to `auto` on `transitionend`. Needs refs and edge-case care (resizing content). Modern CSS alternatives: `grid-template-rows`, `interpolate-size: allow-keywords`, `<details>` + `::details-content`.

</details>

<details>
<summary><strong>5. Could you use native <code>&lt;details&gt;/&lt;summary&gt;</code>?</strong></summary>

Yes — free a11y and toggle. Exclusive "single-open" is available via the `name` attribute on `<details>` in modern browsers. Limitations: harder animation/styling, less control over state and analytics. Interviewers usually want the custom version to see state design.

</details>

<details>
<summary><strong>6. Persist open state / deep-link to a panel?</strong></summary>

Sync `openIds` to the URL hash/query (`#faq-hooks`) or `localStorage`; on mount read it as `defaultOpenIds`; optionally scroll the panel into view.

</details>

<details>
<summary><strong>7. Why store ids, not an index?</strong></summary>

Reordering/filtering the items shifts indexes and would open the wrong panel; ids are stable and allow multi-open with the same data shape.

</details>

---

## 11. Common Mistakes

- Using `<div onClick>` for headers (no keyboard / role / focus).
- Missing `aria-expanded` / `aria-controls`.
- Separate `useState` booleans per item — doesn't scale and can't enforce single mode.
- Storing `openIndex` and breaking multi-open mode.
- Mutating state: `openIds.push(id); setOpenIds(openIds)`.
- Hiding with `height: 0` only — hidden content still tabbable.
- Hardcoded ids (`id="panel1"`) breaking with multiple accordions.
- Animating `max-height: 1000px` hack → wrong timing for short/long content.
- Using index as key.

---

## 12. Interview Tips & Time Plan

Lead with the state decision ("array of open ids handles both modes") and mention a11y before they ask.

| Minutes | Activity |
| --- | --- |
| 0–4 | Clarify, define props and state shape |
| 4–12 | Render items, toggle for single + multi mode |
| 12–18 | CSS: layout, icon, open/close |
| 18–26 | ARIA attributes + `useId` |
| 26–32 | Keyboard navigation with refs |
| 32–36 | Animation (grid trick) + reduced motion |
| 36–40 | Edge cases, follow-ups (controlled, disabled items, lazy content) |

---

## 13. Practice Checklist

- [ ] Single-open mode with collapsible active item.
- [ ] Multi-open mode.
- [ ] `defaultOpenIds` handled for both modes.
- [ ] `aria-expanded`, `aria-controls`, `aria-labelledby`, heading wrapper.
- [ ] ↑ ↓ Home End navigation with wraparound.
- [ ] Closed content not focusable.
- [ ] CSS animation without JS measuring.
- [ ] Controlled variant (`openIds` + `onOpenChange`).
- [ ] Support disabled items.
- [ ] Re-implement with native `<details name>` and compare.
