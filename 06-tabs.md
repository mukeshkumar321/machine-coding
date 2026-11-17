# Tabs — Machine Coding

> **Difficulty:** Easy–Medium | **Time:** 35–45 min | **Asked at / Frequency:** Very common (UI-component / design-system style rounds) — ⭐⭐⭐⭐ | **Tier:** 1 (must know)

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

Build a reusable **Tabs** component. Given `items = [{ id, label, content, disabled? }]` it renders a tab list and the active panel. It must work as **uncontrolled** (manages its own state, `defaultValue`) and **controlled** (parent owns `value` + `onChange`). Support **keyboard navigation** (arrow keys, Home/End) and correct **ARIA** roles.

---

## 2. Requirements

### Functional

- Click a tab → show its panel.
- Uncontrolled: `defaultValue`; controlled: `value` + `onChange`.
- Keyboard: `←`/`→` (or `↑`/`↓` if vertical), wrap-around, `Home`, `End`; skip disabled tabs.
- Disabled tabs can't be selected.
- Default to the first enabled tab if no value is given.

### Non-functional

- Reusable API, no hardcoded tabs.
- ARIA tabs pattern (`tablist`, `tab`, `tabpanel`, roving tabindex).
- Only the active tab is in the page tab order.
- Re-render safe: switching tabs shouldn't remount the tab list.

---

## 3. Clarifying Questions to Ask

- Controlled, uncontrolled, or both? (Both.)
- **Automatic** activation (focus selects) or **manual** (Enter/Space selects)? (Automatic; mention manual for expensive panels.)
- Should inactive panels stay mounted (preserve state) or unmount? (Option `keepMounted`.)
- Horizontal only, or vertical too?
- Are disabled tabs possible? Overflow/scrolling for many tabs?
- URL sync (`?tab=`) needed?

---

## 4. Approach & Component Design

**State shape**

```js
activeId: string          // the only state; controlled → comes from props
tabRefs:  { [id]: HTMLButtonElement }   // ref map for programmatic focus
```

**Controlled / uncontrolled in one hook** — `useControllableState`:

| Mode | Source of truth | On select |
| --- | --- | --- |
| Uncontrolled (`value === undefined`) | internal `useState` | set state + call `onChange` |
| Controlled (`value !== undefined`) | `value` prop | call `onChange` only |

```
Tabs
 ├── <div role="tablist" aria-orientation>
 │     └── <button role="tab" aria-selected aria-controls tabIndex={0|-1}> × N
 └── <div role="tabpanel" aria-labelledby hidden={!selected}> × N
```

Keyboard = **roving tabindex**: selected tab has `tabIndex=0`, others `-1`; arrows move focus + selection.

---

## 5. Reference Solution

<details>
<summary><strong>Show Full Code</strong></summary>

**`useControllableState.js`**

```jsx
import { useCallback, useState } from 'react';

export function useControllableState({ value, defaultValue, onChange }) {
  const isControlled = value !== undefined;
  const [inner, setInner] = useState(defaultValue);
  const current = isControlled ? value : inner;

  const setValue = useCallback(
    (next) => {
      if (!isControlled) setInner(next);
      onChange?.(next);
    },
    [isControlled, onChange]
  );

  return [current, setValue];
}
```

**`Tabs.jsx`**

```jsx
import { useId, useRef } from 'react';
import { useControllableState } from './useControllableState';
import './tabs.css';

export default function Tabs({
  items,
  value,
  defaultValue,
  onChange,
  orientation = 'horizontal',
  keepMounted = false,
  label,
}) {
  const baseId = useId();
  const tabRefs = useRef({});
  const enabledIds = items.filter((i) => !i.disabled).map((i) => i.id);

  const [selected, setSelected] = useControllableState({
    value,
    defaultValue: defaultValue ?? enabledIds[0],
    onChange,
  });

  // Guard against an invalid/disabled value so one tab is always reachable
  const activeId = enabledIds.includes(selected) ? selected : enabledIds[0];

  const isHorizontal = orientation === 'horizontal';
  const prevKey = isHorizontal ? 'ArrowLeft' : 'ArrowUp';
  const nextKey = isHorizontal ? 'ArrowRight' : 'ArrowDown';

  const handleKeyDown = (e, id) => {
    const count = enabledIds.length;
    const index = enabledIds.indexOf(id);
    let nextId;

    if (e.key === nextKey) nextId = enabledIds[(index + 1) % count];
    else if (e.key === prevKey) nextId = enabledIds[(index - 1 + count) % count];
    else if (e.key === 'Home') nextId = enabledIds[0];
    else if (e.key === 'End') nextId = enabledIds[count - 1];
    else return;

    e.preventDefault();
    setSelected(nextId); // automatic activation
    tabRefs.current[nextId]?.focus();
  };

  return (
    <div className={`tabs tabs-${orientation}`}>
      <div role="tablist" aria-label={label} aria-orientation={orientation} className="tabs-list">
        {items.map((item) => {
          const isSelected = item.id === activeId;
          return (
            <button
              key={item.id}
              type="button"
              role="tab"
              id={`${baseId}-tab-${item.id}`}
              aria-selected={isSelected}
              aria-controls={`${baseId}-panel-${item.id}`}
              tabIndex={isSelected ? 0 : -1}
              disabled={item.disabled}
              ref={(el) => {
                if (el) tabRefs.current[item.id] = el;
                else delete tabRefs.current[item.id];
              }}
              className={`tab ${isSelected ? 'active' : ''}`}
              onClick={() => setSelected(item.id)}
              onKeyDown={(e) => handleKeyDown(e, item.id)}
            >
              {item.label}
            </button>
          );
        })}
      </div>

      {items.map((item) => {
        const isSelected = item.id === activeId;
        return (
          <div
            key={item.id}
            role="tabpanel"
            id={`${baseId}-panel-${item.id}`}
            aria-labelledby={`${baseId}-tab-${item.id}`}
            hidden={!isSelected}
            tabIndex={0}
            className="tab-panel"
          >
            {(isSelected || keepMounted) && item.content}
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
import Tabs from './Tabs';

const ITEMS = [
  { id: 'profile', label: 'Profile', content: <p>Profile settings…</p> },
  { id: 'billing', label: 'Billing', content: <p>Billing details…</p> },
  { id: 'beta', label: 'Beta', content: <p>Not available</p>, disabled: true },
  { id: 'security', label: 'Security', content: <p>Password & 2FA…</p> },
];

export default function App() {
  const [tab, setTab] = useState('billing');

  return (
    <main style={{ maxWidth: 560, margin: '40px auto', fontFamily: 'system-ui, sans-serif' }}>
      <h2>Uncontrolled</h2>
      <Tabs items={ITEMS} defaultValue="profile" label="Account settings" />

      <h2>Controlled</h2>
      <button onClick={() => setTab('security')}>Jump to Security</button>
      <Tabs items={ITEMS} value={tab} onChange={setTab} label="Account settings (controlled)" />
      <p>Current: {tab}</p>
    </main>
  );
}
```

**`tabs.css`**

```css
.tabs { display: flex; flex-direction: column; margin: 12px 0 24px; }
.tabs-vertical { flex-direction: row; }
.tabs-list { display: flex; border-bottom: 2px solid #e5e7eb; }
.tabs-vertical .tabs-list { flex-direction: column; border-bottom: 0; border-right: 2px solid #e5e7eb; }

.tab {
  padding: 10px 16px; background: none; border: 0; font: inherit; cursor: pointer;
  border-bottom: 2px solid transparent; margin-bottom: -2px; color: #555;
}
.tabs-vertical .tab { border-bottom: 0; margin-bottom: 0; border-right: 2px solid transparent; margin-right: -2px; text-align: left; }
.tab:hover:not(:disabled) { color: #111; }
.tab.active { color: #4f46e5; border-color: #4f46e5; font-weight: 600; }
.tab:disabled { color: #aaa; cursor: not-allowed; }
.tab:focus-visible, .tab-panel:focus-visible { outline: 2px solid #4f46e5; outline-offset: 2px; }
.tab-panel { padding: 16px; }
```

</details>

---

## 6. How It Works

- **`useControllableState`** abstracts the controlled/uncontrolled split: `value !== undefined` means controlled. Uncontrolled updates internal state *and* notifies `onChange`; controlled only notifies and lets the parent re-render with the new `value`.
- **`defaultValue` is only read on mount** (`useState` initial value) — exactly the React convention for uncontrolled inputs.
- **Roving tabindex** — only the active tab is `tabIndex=0`, so Tab key enters the tablist once, and goes to the panel next; arrows move within the list.
- **Keyboard handler** works on `enabledIds`, so disabled tabs are skipped and wrap-around is a modulo. `e.preventDefault()` stops page scrolling on arrows/Home/End.
- **Focus + select together** (automatic activation) via `tabRefs` map; ref callback cleans up on unmount.
- **Derived `activeId`** guards a bad `value`/`defaultValue` instead of syncing state with an effect.
- **ARIA ids** use `useId` so multiple Tabs instances stay unique; tab ↔ panel linked both directions.
- **Panels always exist, content optional** — `aria-controls` always points to a real element; `keepMounted` decides whether inactive content stays alive (preserves form state).

---

## 7. Edge Cases

- No `defaultValue`/`value` → first enabled tab.
- `value` pointing to a missing/disabled id → falls back to first enabled.
- All tabs disabled → `enabledIds` empty: nothing selected; `handleKeyDown` can't run because buttons are disabled (still avoid dividing/indexing `undefined`).
- Items change dynamically (tab removed while active) → derived `activeId` falls back; a controlled parent should update `value`.
- Switching controlled ⇄ uncontrolled at runtime → React-style warning; avoid.
- Only one tab → arrows are no-ops (modulo returns same).
- Many tabs overflow → horizontal scroll (`overflow-x: auto`) and `scrollIntoView` on select.
- Rapid arrow presses → each updates state; no async hazards.
- Heavy panel content + automatic activation → consider manual activation or lazy mount.

---

## 8. Accessibility

- Roles: `tablist` → `tab` → `tabpanel`; `aria-selected`, `aria-controls`, `aria-labelledby`.
- Roving tabindex; Tab leaves the tablist into the panel (`tabIndex=0` on the panel helps when it has no focusable content).
- `aria-orientation` for vertical lists with ↑/↓ keys.
- Provide a label for the tablist (`aria-label`/`aria-labelledby`).
- Disabled tabs: `disabled` removes them from focus; `aria-disabled` keeps them discoverable (then skip selection manually).
- Visible focus ring distinct from the "selected" style.
- Inactive panels `hidden` so they're out of the a11y tree.

---

## 9. Performance Considerations

- Mounting only the active panel avoids rendering heavy hidden content; `keepMounted` trades memory for state preservation.
- Memoise expensive panel components (`React.memo`) or lazy-load with `React.lazy` + `Suspense`.
- Ref callback on each render is cheap; the map avoids `querySelector`.
- Controlled `onChange` identity changes recreate `setValue` — harmless, but a `useEvent`-style ref can stabilise it.
- Avoid inline `content` JSX creation for dozens of tabs per render when content is heavy; pass render functions/components instead.

---

## 10. Follow-up Questions

<details>
<summary><strong>1. Automatic vs manual activation — when do you choose each?</strong></summary>

Automatic (arrow = select) is best when panels render instantly. Use manual (arrows only move focus; Enter/Space selects) when panel content is slow/expensive or triggers network calls, so keyboard users don't fire loads while scanning.

</details>

<details>
<summary><strong>2. Convert to a compound-component API (<code>&lt;Tabs&gt;&lt;Tabs.List&gt;…</code>).</strong></summary>

Create a context holding `{ activeId, setActive, baseId, registerTab }`; `Tabs.Tab` and `Tabs.Panel` read from it with `useContext` and take a `value` prop. Roving focus then needs tabs to register in order (ref list in context) or DOM queries within the tablist.

</details>

<details>
<summary><strong>3. Sync the active tab with the URL.</strong></summary>

Controlled mode: `value` from `useSearchParams().get('tab')`, `onChange` writes it. Fall back to default when the param is invalid; use `replace` if you don't want a history entry per tab.

</details>

<details>
<summary><strong>4. Lazy-load panels and keep them alive after first visit.</strong></summary>

Track `visited` ids in a `Set`/array state; render content when `selected || visited.has(id)`. Use `React.lazy` for code splitting.

</details>

<details>
<summary><strong>5. Handle overflow with many tabs.</strong></summary>

`overflow-x: auto` list, scroll the selected tab into view (`el.scrollIntoView({ inline: 'nearest', block: 'nearest' })`), or collapse extras into a "More" menu using `ResizeObserver` to measure.

</details>

<details>
<summary><strong>6. Support RTL?</strong></summary>

In RTL horizontal lists, `ArrowRight` should go to the *previous* tab. Detect `dir` (e.g. `getComputedStyle(el).direction`) and swap prev/next keys.

</details>

<details>
<summary><strong>7. Why do we need the controlled mode at all?</strong></summary>

So the parent can drive tabs from other UI (deep links, wizard "Next" buttons, validation that blocks switching) and keep a single source of truth. The parent can even reject a change by not updating `value`.

</details>

---

## 11. Common Mistakes

- Using `<div onClick>` instead of `<button role="tab">`.
- Index-based state with keys by index → breaks when tabs change/disable.
- Syncing prop → state with `useEffect` instead of a controllable hook.
- Forgetting roving tabindex (every tab in tab order) or `preventDefault` on arrows.
- Arrow navigation that doesn't skip disabled tabs or doesn't wrap.
- Missing `aria-controls`/`aria-labelledby`; hard-coded ids that collide.
- `defaultValue` not respected / re-applied on every render.
- Mutating `items` or storing the active *item object* (goes stale) instead of its id.
- Unmounting inactive panels and silently losing form input (when users expect it to be kept).

---

## 12. Interview Tips & Time Plan

State up front: "one piece of state, `activeId`, via a controllable hook." Do the click version first, then keyboard, then ARIA.

| Minutes | Activity |
| --- | --- |
| 0–5 | Clarify (controlled?, activation, vertical?), design API |
| 5–12 | Basic tabs: render list + panel, click to select |
| 12–20 | `useControllableState`; demo both modes |
| 20–30 | Keyboard nav with roving tabindex + refs, skip disabled |
| 30–36 | ARIA roles/ids with `useId` |
| 36–42 | CSS, vertical orientation, `keepMounted` |
| 42–45 | Edge cases, follow-ups (manual activation, compound API) |

---

## 13. Practice Checklist

- [ ] Basic tabs with click selection.
- [ ] Uncontrolled via `defaultValue`.
- [ ] Controlled via `value` + `onChange`.
- [ ] Arrow/Home/End with wrap-around, skipping disabled.
- [ ] Roving tabindex verified with keyboard only.
- [ ] `tablist`/`tab`/`tabpanel` ARIA wired with unique ids.
- [ ] Safe fallback for invalid `value`.
- [ ] `keepMounted` option.
- [ ] Vertical orientation + RTL handling.
- [ ] Rebuild as compound components with context.
