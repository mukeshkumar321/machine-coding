# Multi-Select Dropdown — Machine Coding

> **Difficulty:** Hard | **Time:** 60 min | **Frequency:** ⭐⭐⭐⭐

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

Build a reusable **multi-select dropdown** (a combobox with multiple values) that supports:

- **Search** to filter options
- Selected values shown as **chips** (removable)
- **Select all** (for the currently filtered options)
- Close on **outside click**
- Full **keyboard** control (↑ ↓ Enter Esc Backspace Tab)

---

## 2. Requirements

### Functional

- Click the control to open; type to filter (case-insensitive).
- Click / press Enter on an option to toggle it; the list stays open.
- Selected values appear as chips with a × button; Backspace in an empty input removes the last chip.
- "Select all" toggles every **filtered** option; shows checked / partial / unchecked state.
- "Clear all" button; empty state "No results".
- Escape and outside click close the list (and reset the query).
- Arrow keys move the active option (wrap-around); Enter toggles it; the active option scrolls into view.

### Non-functional

- **Controlled** component: `value: string[]`, `onChange(nextValues)`.
- Combobox/listbox ARIA pattern; focus stays in the input (`aria-activedescendant`).
- Efficient for a few thousand options (`Set` lookups, memoised filter).

---

## 3. Clarifying Questions to Ask

- Controlled (parent owns `value`) or internal state? *(Controlled, with `onChange`.)*
- Options static or loaded async (server-side search)?
- Select all: all options, or only filtered? Maximum selection limit?
- Should selected items stay in the list (with a checkmark) or disappear?
- How many chips before collapsing into "+N more"?
- Creatable options ("Add 'foo'")? Grouped options? Disabled options?
- Does the list close after each selection? *(Multi-select: no.)*

---

## 4. Approach & Component Design

**State (inside the component)**

| State | Notes |
|---|---|
| `open` | list visibility |
| `query` | search text |
| `active` | highlighted row index in the visible list (row 0 = "Select all") |
| *(prop)* `value: string[]` | selected values, owned by parent |

**Derived (never stored):** `selected = new Set(value)`, `filtered`, `allSelected`, `someSelected`, `rowCount`, `labelByValue`.

```
MultiSelect (rootRef — outside-click boundary)
 ├─ control (click → open + focus input)
 │    ├─ chips: value.map → [label ×]
 │    ├─ <input role="combobox" aria-activedescendant=…>   ← real focus always here
 │    └─ clear-all ✕  ·  caret ▾
 └─ panel (if open; onMouseDown preventDefault → keep input focus)
      ├─ <ul role="listbox" aria-multiselectable>
      │    ├─ row 0  "Select all (N)"       (aria-selected = allSelected)
      │    └─ rows 1…N  option ☑ label
      └─ "No results"
```

**Keyboard map**

| Key | Action |
|---|---|
| ↓ / ↑ | open if closed; else move active (wrap) |
| Enter | toggle active row (or open if closed) |
| Esc | close, clear query |
| Backspace (empty input) | remove last chip |
| Tab | close, let focus move on |

---

## 5. Reference Solution

<details>
<summary><strong>Show Full Code (useClickOutside.js, MultiSelect.jsx, App.jsx, styles.css)</strong></summary>

```jsx
// useClickOutside.js
import { useEffect, useRef } from 'react';

export function useClickOutside(ref, handler) {
  const handlerRef = useRef(handler);

  // always call the latest handler without re-subscribing the listener
  useEffect(() => {
    handlerRef.current = handler;
  });

  useEffect(() => {
    const listener = (e) => {
      if (ref.current && !ref.current.contains(e.target)) handlerRef.current(e);
    };
    document.addEventListener('pointerdown', listener);
    return () => document.removeEventListener('pointerdown', listener);
  }, [ref]);
}
```

```jsx
// MultiSelect.jsx
import { useEffect, useId, useMemo, useRef, useState } from 'react';
import { useClickOutside } from './useClickOutside';

export default function MultiSelect({
  options,
  value,
  onChange,
  label = 'Select options',
  placeholder = 'Select…',
}) {
  const uid = useId();
  const listId = `${uid}-list`;
  const optionId = (row) => `${uid}-opt-${row}`;

  const [open, setOpen] = useState(false);
  const [query, setQuery] = useState('');
  const [active, setActive] = useState(0);
  const rootRef = useRef(null);
  const inputRef = useRef(null);

  const close = () => {
    setOpen(false);
    setQuery('');
    setActive(0);
  };
  useClickOutside(rootRef, close);

  const selected = useMemo(() => new Set(value), [value]);
  const labelByValue = useMemo(
    () => new Map(options.map((o) => [o.value, o.label])),
    [options]
  );
  const filtered = useMemo(() => {
    const q = query.trim().toLowerCase();
    return q ? options.filter((o) => o.label.toLowerCase().includes(q)) : options;
  }, [options, query]);

  const selectedInView = filtered.filter((o) => selected.has(o.value)).length;
  const allSelected = filtered.length > 0 && selectedInView === filtered.length;
  const someSelected = selectedInView > 0 && !allSelected;

  // row 0 is "Select all"; rows 1..n map to filtered[row - 1]
  const rowCount = filtered.length > 0 ? filtered.length + 1 : 0;
  const activeRow = Math.min(active, Math.max(rowCount - 1, 0));

  const toggle = (v) =>
    onChange(selected.has(v) ? value.filter((x) => x !== v) : [...value, v]);

  const toggleAll = () => {
    if (allSelected) {
      const inView = new Set(filtered.map((o) => o.value));
      onChange(value.filter((v) => !inView.has(v)));
    } else {
      const toAdd = filtered.filter((o) => !selected.has(o.value)).map((o) => o.value);
      onChange([...value, ...toAdd]);
    }
  };

  const activate = (row) => (row === 0 ? toggleAll() : toggle(filtered[row - 1].value));

  useEffect(() => {
    if (!open) return;
    document.getElementById(`${uid}-opt-${activeRow}`)?.scrollIntoView({ block: 'nearest' });
  }, [open, activeRow, uid]);

  const onKeyDown = (e) => {
    switch (e.key) {
      case 'ArrowDown':
      case 'ArrowUp': {
        e.preventDefault();
        if (!open) {
          setOpen(true);
          return;
        }
        if (rowCount === 0) return;
        const step = e.key === 'ArrowDown' ? 1 : -1;
        setActive((activeRow + step + rowCount) % rowCount);
        break;
      }
      case 'Enter':
        e.preventDefault();
        if (open && rowCount > 0) activate(activeRow);
        else setOpen(true);
        break;
      case 'Escape':
        if (open) {
          e.preventDefault();
          close();
        }
        break;
      case 'Backspace':
        if (!query && value.length > 0) onChange(value.slice(0, -1));
        break;
      case 'Tab':
        close();
        break;
      default:
    }
  };

  return (
    <div className="ms" ref={rootRef}>
      <div
        className={`ms-control${open ? ' open' : ''}`}
        onClick={() => {
          setOpen(true);
          inputRef.current?.focus();
        }}
      >
        {value.map((v) => {
          const text = labelByValue.get(v) ?? v;
          return (
            <span className="chip" key={v}>
              {text}
              <button
                type="button"
                aria-label={`Remove ${text}`}
                onClick={(e) => {
                  e.stopPropagation();
                  toggle(v);
                  inputRef.current?.focus();
                }}
              >
                ×
              </button>
            </span>
          );
        })}

        <input
          ref={inputRef}
          className="ms-input"
          role="combobox"
          aria-label={label}
          aria-expanded={open}
          aria-controls={listId}
          aria-autocomplete="list"
          aria-activedescendant={open && rowCount > 0 ? optionId(activeRow) : undefined}
          placeholder={value.length ? '' : placeholder}
          value={query}
          onChange={(e) => {
            setQuery(e.target.value);
            setOpen(true);
            setActive(e.target.value ? 1 : 0); // first match, so Enter doesn't hit "Select all"
          }}
          onKeyDown={onKeyDown}
        />

        {value.length > 0 && (
          <button
            type="button"
            className="ms-clear"
            aria-label="Clear all selections"
            onClick={(e) => {
              e.stopPropagation();
              onChange([]);
              inputRef.current?.focus();
            }}
          >
            ✕
          </button>
        )}
        <span className="ms-caret" aria-hidden="true">▾</span>
      </div>

      {open && (
        // keep focus in the input when clicking inside the panel
        <div className="ms-panel" onMouseDown={(e) => e.preventDefault()}>
          <ul id={listId} role="listbox" aria-label={label} aria-multiselectable="true">
            {rowCount > 0 && (
              <li
                id={optionId(0)}
                role="option"
                aria-selected={allSelected}
                className={`ms-option select-all${activeRow === 0 ? ' active' : ''}`}
                onMouseEnter={() => setActive(0)}
                onClick={toggleAll}
              >
                <span className={`box${allSelected ? ' checked' : someSelected ? ' mixed' : ''}`} aria-hidden="true" />
                {allSelected ? 'Deselect all' : query ? 'Select all results' : 'Select all'} ({filtered.length})
              </li>
            )}
            {filtered.map((o, i) => {
              const row = i + 1;
              const isSelected = selected.has(o.value);
              return (
                <li
                  key={o.value}
                  id={optionId(row)}
                  role="option"
                  aria-selected={isSelected}
                  className={`ms-option${activeRow === row ? ' active' : ''}`}
                  onMouseEnter={() => setActive(row)}
                  onClick={() => toggle(o.value)}
                >
                  <span className={`box${isSelected ? ' checked' : ''}`} aria-hidden="true" />
                  {o.label}
                </li>
              );
            })}
          </ul>
          {rowCount === 0 && <div className="ms-empty" role="status">No results</div>}
        </div>
      )}
    </div>
  );
}
```

```jsx
// App.jsx
import { useState } from 'react';
import MultiSelect from './MultiSelect';
import './styles.css';

const OPTIONS = [
  'JavaScript', 'TypeScript', 'Python', 'Java', 'Go', 'Rust', 'C++', 'C#',
  'Kotlin', 'Swift', 'Ruby', 'PHP', 'Dart', 'Scala',
].map((name) => ({ value: name.toLowerCase(), label: name }));

export default function App() {
  const [langs, setLangs] = useState(['javascript']);

  return (
    <main className="page">
      <h1>Multi-select</h1>
      <MultiSelect
        label="Languages"
        placeholder="Pick languages…"
        options={OPTIONS}
        value={langs}
        onChange={setLangs}
      />
      <p>Selected: {langs.length ? langs.join(', ') : 'none'}</p>
    </main>
  );
}
```

```css
/* styles.css */
.page { max-width: 480px; margin: 40px auto; font-family: system-ui, sans-serif; }
.ms { position: relative; }
.ms-control { display: flex; flex-wrap: wrap; align-items: center; gap: 6px; min-height: 40px; padding: 4px 32px 4px 8px; border: 1px solid #94a3b8; border-radius: 8px; background: #fff; cursor: text; position: relative; }
.ms-control.open, .ms-control:focus-within { border-color: #2563eb; box-shadow: 0 0 0 2px rgba(37, 99, 235, 0.25); }
.ms-input { flex: 1; min-width: 80px; border: 0; outline: 0; padding: 6px 0; font: inherit; background: transparent; }
.chip { display: inline-flex; align-items: center; gap: 4px; padding: 2px 4px 2px 8px; background: #e0e7ff; color: #1e3a8a; border-radius: 12px; font-size: 13px; }
.chip button { border: 0; background: none; cursor: pointer; font-size: 16px; line-height: 1; color: inherit; border-radius: 50%; }
.chip button:hover { background: rgba(30, 58, 138, 0.15); }
.ms-clear { border: 0; background: none; cursor: pointer; color: #64748b; padding: 4px; }
.ms-caret { position: absolute; right: 10px; top: 50%; transform: translateY(-50%); color: #64748b; pointer-events: none; }

.ms-panel { position: absolute; z-index: 10; left: 0; right: 0; top: calc(100% + 4px); max-height: 260px; overflow-y: auto; background: #fff; border: 1px solid #cbd5e1; border-radius: 8px; box-shadow: 0 8px 24px rgba(0, 0, 0, 0.12); }
.ms-panel ul { list-style: none; margin: 0; padding: 4px; }
.ms-option { display: flex; align-items: center; gap: 8px; padding: 8px; border-radius: 6px; cursor: pointer; user-select: none; }
.ms-option.active { background: #eff6ff; }
.ms-option.select-all { font-weight: 600; border-bottom: 1px solid #e2e8f0; border-radius: 6px 6px 0 0; margin-bottom: 2px; }
.ms-option[aria-selected='true']:not(.select-all) { font-weight: 600; }
.ms-empty { padding: 12px; color: #64748b; text-align: center; }

.box { width: 16px; height: 16px; border: 2px solid #64748b; border-radius: 4px; box-sizing: border-box; position: relative; flex: none; }
.box.checked { background: #2563eb; border-color: #2563eb; }
.box.checked::after { content: ''; position: absolute; left: 3px; top: 0; width: 4px; height: 8px; border: solid #fff; border-width: 0 2px 2px 0; transform: rotate(45deg); }
.box.mixed { background: #2563eb; border-color: #2563eb; }
.box.mixed::after { content: ''; position: absolute; left: 2px; right: 2px; top: 5px; height: 2px; background: #fff; }
```

</details>

---

## 6. How It Works

- **Controlled component.** `value` is the single source of truth; every change calls `onChange(newArray)` with a new array — no mutation, parent decides what to do (validation, max limit).
- **Derived data.** `selected` (a `Set`) gives O(1) `has`; `filtered` is memoised on `[options, query]`; `allSelected`/`someSelected` are computed from `filtered`, so Select-all always reflects the *current search*.
- **Focus never leaves the input.** The ARIA *combobox* pattern: the input has real focus and points at the highlighted option with `aria-activedescendant`. Mouse clicks inside the panel call `preventDefault` on `mousedown`, otherwise the input would blur and the list would feel broken.
- **Row indexing.** One `active` index over the visible rows; row 0 is "Select all", rows `1..n` map to `filtered[row-1]`. `activeRow` is clamped, so shrinking the list while filtering can never point past the end.
- **Outside click via `pointerdown` on `document`** + `ref.contains(e.target)`. `useClickOutside` stores the handler in a ref so the listener is attached once and always calls the latest `close` (no stale closure, no re-subscribing) and is removed on unmount. `pointerdown` (not `click`) closes before other controls handle the click and covers touch.
- **Chips** come from `value` in selection order; label lookup via a `Map` (falls back to the raw value if an option disappeared).
- **Stays open after selecting** — multi-select expectation; Esc/Tab/outside click close it and reset the query.
- **Scroll into view** with `scrollIntoView({block:'nearest'})` when the active row changes via keyboard.

---

## 7. Edge Cases

- Query with no matches → "No results", arrow keys do nothing, Enter does nothing, "Select all" hidden.
- Select all while filtered → only filtered items change; previously selected items outside the filter remain.
- Deselect-all while filtered removes only the filtered ones.
- Backspace removes a chip only when the input is empty (otherwise it edits the query).
- `value` contains a value no longer in `options` → chip shows the raw value and can still be removed.
- Chip × and "clear" clicks must `stopPropagation` (the control's onClick would reopen the list).
- Duplicate option values (use unique `value` as key).
- Very long labels / many chips → wrap, or collapse into "+N".
- Disabled state / max selections (`maxSelected`) – guard in `toggle` and `toggleAll`.
- Clicking the scrollbar of the panel must not close it (it is inside `rootRef`).
- Window blur / Tab out closes via the Tab handler; Shift+Tab as well.

---

## 8. Accessibility

- **Pattern:** `role="combobox"` input (`aria-expanded`, `aria-controls`, `aria-autocomplete="list"`, `aria-activedescendant`) + `role="listbox"` with `aria-multiselectable="true"` + `role="option"` items with `aria-selected`.
- Checkboxes are visual only (`aria-hidden`); state is exposed through `aria-selected`.
- Chip remove buttons have `aria-label="Remove <label>"`; "Clear all" labelled.
- Everything works with the keyboard alone; visible focus ring (`:focus-within` on the control).
- Announce results count: add an `aria-live="polite"` region ("5 results", "No results") — `role="status"` is used for the empty state; extend it for counts.
- Don't rely on colour for selected state (checkbox + bold).
- Touch targets ≥ 32–44px for chips' × (increase padding on mobile).

---

## 9. Performance Considerations

- `Set` for `selected` → O(1) membership instead of `value.includes` per option (O(n·m)).
- `useMemo` for filtering; with thousands of options **virtualise** the list (react-window) or cap rendered rows and "show more". With virtualisation `scrollIntoView` by id no longer works — use the list's `scrollToItem(index)`.
- Debounce the query for server-side search (and cancel in-flight requests with `AbortController`).
- Memoise option rows (`React.memo`) with stable callbacks if the list is large, since hovering updates `active` and re-renders all rows.
- Only one document listener per mounted instance; removed in cleanup.

---

## 10. Follow-up Questions

<details>
<summary><strong>1. How would you load options asynchronously (server-side search)?</strong></summary>

Debounce `query` (250–300 ms), fetch with an `AbortController` (cancel the previous request in the effect cleanup), show a loading row, handle error/retry. Keep a cache of labels for already-selected values (`value` may reference options that are not in the current result page) so chips still render.

</details>

<details>
<summary><strong>2. Show "+N more" instead of unlimited chips?</strong></summary>

Render the first `maxVisible` chips and a `+{value.length - maxVisible}` badge (tooltip/popover lists the rest). A fancier version measures available width with `ResizeObserver` to decide how many chips fit.

</details>

<details>
<summary><strong>3. Support grouped options and disabled options?</strong></summary>

Flatten groups into a row list: `{type:'group', label}` headers (`role="presentation"`/`group`) and option rows; keyboard navigation skips headers and disabled rows (compute the list of navigable indices and move within it). "Select all" ignores disabled options.

</details>

<details>
<summary><strong>4. Creatable options ("Create 'foo'")?</strong></summary>

If `query` is non-empty and has no exact (case-insensitive) match, append a synthetic row "Create “query”". On activate call `onCreate(query)` (parent adds it to `options` and `value`). Trim, dedupe and validate.

</details>

<details>
<summary><strong>5. Why `pointerdown` instead of `click` for outside detection?</strong></summary>

`click` fires after `mouseup`; by then another component may have changed the DOM (target removed → `contains` is false → wrongly "outside"). `pointerdown` fires first, covers mouse + touch + pen, and lets the panel close before focus moves.

</details>

<details>
<summary><strong>6. How would you make it work inside a modal or `overflow: hidden` container?</strong></summary>

Render the panel in a **portal** (`createPortal(panel, document.body)`), position it with `getBoundingClientRect()` of the control (recompute on scroll/resize), and flip upward when there's no room. Outside-click logic must then also treat the portal node as "inside" (check both refs).

</details>

<details>
<summary><strong>7. Single-select version? What changes?</strong></summary>

`value` becomes a single value; selecting closes the list and shows the label in the input; remove chips/select-all; `aria-multiselectable` is dropped. Core combobox logic (active index, filtering, outside click) is shared — extract a `useCombobox` hook.

</details>

<details>
<summary><strong>8. How to integrate with native forms?</strong></summary>

Render hidden inputs (`<input type="hidden" name="langs" value={v}>` per selected value) or let the parent serialise `value`. For form libraries expose `value`/`onChange`/`onBlur`/`name` props (works with `Controller`).

</details>

---

## 11. Common Mistakes

- Dropdown closes when clicking an option because the input blurs first (missing `mousedown` `preventDefault` / using `onBlur` to close).
- Outside-click listener added without cleanup, or re-added every render; stale handler closure.
- Mutating `value` (`value.push`) instead of returning a new array.
- "Select all" ignoring the filter, or not showing a partial state.
- Using index as `key`, or using labels as values (duplicates).
- Active index not reset on query change / not clamped when the list shrinks (and Enter-after-typing hitting "Select all" instead of the first match).
- `Enter` submitting the surrounding form (forgot `preventDefault`).
- Tab doesn't close the list → floating menu left behind.
- Chip × click bubbling to the control and reopening the list.
- No `role`/`aria-*` → invisible to screen readers.
- Computing `filtered` and `selected` with `includes` on every render for large lists.

