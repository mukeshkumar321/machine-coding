# Virtualized List — Machine Coding

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

Build a **virtualized (windowed) list** that stays smooth with **10,000+ rows** by mounting only the rows that are visible (plus a small overscan buffer), and a reusable **`useVirtualList`** hook behind it.

Do it in two stages:

1. **Fixed row height** — compute the visible range with plain arithmetic (`floor(scrollTop / itemHeight)`).
2. **Variable row height** — rows are measured after render, stored in a **prefix-sum** array, and the first visible row is found with **binary search**.

Also support **`scrollToIndex(i, { align })`** and a container whose height **changes at runtime** (resizable panel, window resize).

---

## 2. Requirements

### Functional

- Render 10,000 rows with a DOM that only contains ~(viewport / rowHeight + 2 × overscan) rows.
- The scrollbar behaves as if all rows existed (correct total height, correct thumb position).
- **Overscan** rows above and below the viewport so fast scrolling doesn't flash blank space.
- **Fixed mode** (`itemHeight` is a number): O(1) range calculation.
- **Variable mode** (no `itemHeight`): start from `estimateHeight`, measure real heights with `ResizeObserver`, rebuild prefix sums, find the start row with **binary search**.
- `scrollToIndex(index, { align: 'start' | 'center' | 'end' | 'auto' })`, also correct in variable mode where unmeasured rows only have estimates.
- Reacts to **container resize** (viewport height change) without a page reload.
- Stable **keys** (`getKey`) so measured heights follow the item, not the position.
- `onRangeChange` callback so the demo can show "rendering rows a–b".

### Non-functional

- Range / offset logic lives in **pure functions** that are easy to unit-test.
- No visible jump when a row *above* the viewport is measured (scroll anchoring).
- Rows are positioned with `transform: translateY` (no layout on scroll).
- No third-party libraries; plain CSS.

---

## 3. Clarifying Questions to Ask

- Are all rows the **same height**? Can the height change after render (images loading, expand/collapse, window resize re-wrapping text)?
- Do we know heights up front (server-provided / computable from text) or must we **measure**?
- How many rows — 10k, 1M? Is the data all in memory, or **paginated / infinite**?
- Do we need `scrollToIndex`, "stick to bottom" (chat), or restore scroll position on back-navigation?
- **Horizontal** and/or **grid** virtualization, or just a vertical list?
- Sticky headers, grouped sections, selection, drag-and-drop reorder?
- Is browser **Ctrl+F** inside the list required? (It can't find unmounted rows.)
- Is using `react-window` / `react-virtuoso` allowed, or must we build it?

---

## 4. Approach & Component Design

**Idea:** the scroll container has a tall inner "spacer" of `totalSize` px so the scrollbar is right; only the rows intersecting `[scrollTop, scrollTop + viewport]` (± overscan) are mounted, each placed absolutely with `translateY(offset)`.

```
┌ scroller (overflow:auto, height = viewport) ┐
│ ┌ spacer (height = totalSize) ───────────┐  │
│ │  … rows 0..97 NOT mounted …            │  │
│ │  row 98   translateY(1960px)  ← overscan  │
│ │  row 100  translateY(2000px)  ← scrollTop │
│ │  row 110  translateY(2200px)             │
│ │  row 112  translateY(2240px)  ← overscan  │
│ │  … rows 113..9999 NOT mounted …        │  │
│ └────────────────────────────────────────┘  │
└──────────────────────────────────────────────┘
```

**Stage 1 — fixed height (pure arithmetic)**

```js
start = max(0, floor(scrollTop / itemHeight) - overscan)
end   = min(count - 1, floor((scrollTop + viewport) / itemHeight) + overscan)
offset(i) = i * itemHeight          totalSize = count * itemHeight
```

**Stage 2 — variable height (prefix sums + binary search)**

```js
heights   = [100,  50, 100,  50, ...]      // measured, estimate until then
offsets   = [  0, 100, 150, 250, 300, ...] // offsets[i] = top of row i, offsets[count] = total
startRow  = largest i with offsets[i] <= scrollTop      // binary search, O(log n)
endRow    = largest i with offsets[i] <= scrollTop + viewport
```

| Strategy | Range lookup | Update one height | Notes |
|---|---|---|---|
| Fixed height | O(1) | n/a | simplest, fastest |
| Linear scan from 0 | O(n) | O(1) | fine at 100 rows, janky at 10k |
| **Prefix sums + binary search (this solution)** | **O(log n)** | O(n) rebuild | rebuild is batched once per `ResizeObserver` callback |
| Fenwick / segment tree | O(log n) | O(log n) | worth it at 100k+ rows with frequent resizes |

**Measuring:** every rendered row carries `data-index` and is observed by one shared `ResizeObserver`. A callback with N entries produces **one** state bump (`version`) → **one** prefix-sum rebuild.

**Scroll anchoring:** when a row whose top is *above* `scrollTop` changes from its estimate to its real height, everything below shifts. We accumulate that delta and add it to `scrollTop` in the same layout phase so the content under the user's eyes doesn't move. (`overflow-anchor: none` disables the browser's own anchoring so the two don't double-correct.)

**`scrollToIndex` in variable mode:** jump using the current (estimated) offsets, remember the target as `pending`; once the target row is measured, re-run the alignment once with the corrected offsets.

**Component tree**

```
App
 └─ Demo (×2)  index input · align select · Go/Top/Bottom · "rendering rows a–b"
      └─ VirtualList (forwardRef → { scrollToIndex })
           └─ useVirtualList({ count, itemHeight?, estimateHeight, overscan, getKey })
                ├─ virtualMath.js  createFixedMetrics · createVariableMetrics · findIndex
                │                  getVisibleRange · getScrollTarget   (pure)
                ├─ container ResizeObserver → viewport height
                └─ row ResizeObserver → measured heights → version++ → rebuild offsets
```

**`metrics` abstraction** — both modes expose the same `{ count, total, offsetOf, sizeOf, indexAt }`, so range calculation, `scrollToIndex` and rendering are written once.

---

## 5. Reference Solution

<details>
<summary><strong>Show Full Code</strong></summary>

**`virtualMath.js`**

```jsx
export const clamp = (n, min, max) => Math.min(Math.max(n, min), max);

export function createFixedMetrics(count, itemSize) {
  return {
    count,
    total: count * itemSize,
    offsetOf: (i) => i * itemSize,
    sizeOf: () => itemSize,
    indexAt: (y) => clamp(Math.floor(y / itemSize), 0, count - 1),
  };
}

// offsets[i] = top of item i, offsets[count] = total height (prefix sums)
export function buildOffsets(count, sizeOf) {
  const offsets = new Float64Array(count + 1);
  for (let i = 0; i < count; i++) offsets[i + 1] = offsets[i] + sizeOf(i);
  return offsets;
}

// largest i such that offsets[i] <= y  (the item that contains pixel y)
export function findIndex(offsets, y) {
  let lo = 0;
  let hi = offsets.length - 2;
  while (lo < hi) {
    const mid = (lo + hi + 1) >> 1;
    if (offsets[mid] <= y) lo = mid;
    else hi = mid - 1;
  }
  return Math.max(lo, 0);
}

export function createVariableMetrics(count, sizeOf) {
  const offsets = buildOffsets(count, sizeOf);
  return {
    count,
    total: offsets[count],
    offsetOf: (i) => offsets[i],
    sizeOf: (i) => offsets[i + 1] - offsets[i],
    indexAt: (y) => findIndex(offsets, y),
  };
}

export function getVisibleRange(metrics, scrollTop, viewport, overscan) {
  if (metrics.count === 0) return { start: 0, end: -1 };
  const top = clamp(scrollTop, 0, Math.max(0, metrics.total - viewport));
  const first = metrics.indexAt(top);
  const last = metrics.indexAt(top + viewport);
  return {
    start: Math.max(0, first - overscan),
    end: Math.min(metrics.count - 1, last + overscan),
  };
}

export function getScrollTarget(metrics, index, viewport, align = 'start', current = 0) {
  const i = clamp(index, 0, metrics.count - 1);
  const start = metrics.offsetOf(i);
  const size = metrics.sizeOf(i);
  const end = start + size;
  let top = start;
  if (align === 'center') top = start - (viewport - size) / 2;
  else if (align === 'end') top = end - viewport;
  else if (align === 'auto') {
    if (start < current) top = start;
    else if (end > current + viewport) top = end - viewport;
    else top = current;
  }
  return clamp(top, 0, Math.max(0, metrics.total - viewport));
}
```

**`useVirtualList.js`**

```jsx
import { useCallback, useLayoutEffect, useMemo, useRef, useState } from 'react';
import {
  clamp,
  createFixedMetrics,
  createVariableMetrics,
  getScrollTarget,
  getVisibleRange,
} from './virtualMath';

export function useVirtualList({ count, itemHeight, estimateHeight = 50, overscan = 3, getKey }) {
  const isFixed = typeof itemHeight === 'number';
  const containerRef = useRef(null);
  const [scrollTop, setScrollTop] = useState(0);
  const [viewport, setViewport] = useState(0);
  const [version, setVersion] = useState(0); // bumped when measured heights change

  const heights = useRef(new Map()); // key -> measured height
  const elements = useRef(new Set());
  const itemObserver = useRef(null);
  const adjust = useRef(0); // scrollTop correction for rows measured above the viewport
  const pending = useRef(null); // scrollToIndex target waiting for its real height

  const getKeyRef = useRef(getKey);
  getKeyRef.current = getKey;
  const keyOf = useCallback((i) => (getKeyRef.current ? getKeyRef.current(i) : i), []);

  const metrics = useMemo(
    () =>
      isFixed
        ? createFixedMetrics(count, itemHeight)
        : createVariableMetrics(count, (i) => heights.current.get(keyOf(i)) ?? estimateHeight),
    // `version` and `getKey` are deliberate triggers: the heights Map is mutated in place
    // eslint-disable-next-line react-hooks/exhaustive-deps
    [isFixed, count, itemHeight, estimateHeight, version, getKey, keyOf],
  );
  const metricsRef = useRef(metrics);
  metricsRef.current = metrics;

  const { start, end } = getVisibleRange(metrics, scrollTop, viewport, overscan);
  const virtualItems = useMemo(() => {
    const out = [];
    for (let i = start; i <= end; i++) {
      out.push({ index: i, key: keyOf(i), start: metrics.offsetOf(i), size: metrics.sizeOf(i) });
    }
    return out;
  }, [metrics, start, end, keyOf]);

  const onScroll = useCallback((e) => setScrollTop(e.currentTarget.scrollTop), []);

  // viewport height + container resize
  useLayoutEffect(() => {
    const el = containerRef.current;
    if (!el) return undefined;
    const read = () => {
      setViewport(el.clientHeight);
      setScrollTop(el.scrollTop);
    };
    read();
    if (typeof ResizeObserver === 'undefined') return undefined;
    const ro = new ResizeObserver(read);
    ro.observe(el);
    return () => ro.disconnect();
  }, []);

  // row measurement (variable mode only)
  useLayoutEffect(() => {
    if (isFixed || typeof ResizeObserver === 'undefined') return undefined;
    const ro = new ResizeObserver((entries) => {
      const el = containerRef.current;
      const m = metricsRef.current;
      let changed = false;
      for (const { target } of entries) {
        const index = Number(target.dataset.index);
        const height = target.offsetHeight;
        if (!height || index >= m.count) continue;
        const key = keyOf(index);
        if (heights.current.get(key) === height) continue;
        if (el && m.offsetOf(index) < el.scrollTop) adjust.current += height - m.sizeOf(index);
        heights.current.set(key, height);
        changed = true;
      }
      if (changed) setVersion((v) => v + 1);
    });
    itemObserver.current = ro;
    elements.current.forEach((node) => ro.observe(node));
    return () => {
      ro.disconnect();
      itemObserver.current = null;
    };
  }, [isFixed, keyOf]);

  const measureElement = useCallback((node) => {
    if (!node) return;
    elements.current.add(node);
    itemObserver.current?.observe(node);
  }, []);

  // drop unmounted rows from the observer
  useLayoutEffect(() => {
    elements.current.forEach((node) => {
      if (!node.isConnected) {
        itemObserver.current?.unobserve(node);
        elements.current.delete(node);
      }
    });
  });

  // after heights change: finish a pending jump, otherwise keep the visible content anchored
  useLayoutEffect(() => {
    const el = containerRef.current;
    if (!el) return;
    const target = pending.current;
    if (target && target.index >= metrics.count) pending.current = null;
    if (pending.current && heights.current.has(keyOf(pending.current.index))) {
      const { index, align } = pending.current;
      pending.current = null;
      adjust.current = 0;
      el.scrollTop = getScrollTarget(metrics, index, el.clientHeight, align, el.scrollTop);
      setScrollTop(el.scrollTop);
    } else if (adjust.current) {
      el.scrollTop += adjust.current;
      adjust.current = 0;
      setScrollTop(el.scrollTop);
    }
  }, [metrics, keyOf]);

  const scrollToIndex = useCallback(
    (index, { align = 'start', behavior = 'auto' } = {}) => {
      const el = containerRef.current;
      const m = metricsRef.current;
      if (!el || m.count === 0) return;
      const i = clamp(index, 0, m.count - 1);
      const top = getScrollTarget(m, i, el.clientHeight, align, el.scrollTop);
      if (behavior === 'smooth' && el.scrollTo) {
        el.scrollTo({ top, behavior });
        return;
      }
      el.scrollTop = top;
      setScrollTop(top);
      if (!isFixed && !heights.current.has(keyOf(i))) {
        pending.current = { index: i, align };
      }
    },
    [isFixed, keyOf],
  );

  return {
    containerRef,
    onScroll,
    virtualItems,
    totalSize: metrics.total,
    measureElement,
    scrollToIndex,
    range: { start, end },
    isFixed,
  };
}
```

**`VirtualList.jsx`**

```jsx
import { forwardRef, useCallback, useEffect, useImperativeHandle } from 'react';
import { useVirtualList } from './useVirtualList';

const VirtualList = forwardRef(function VirtualList(
  { items, itemHeight, estimateHeight, height = 400, overscan = 3, renderItem, getKey, label, onRangeChange },
  ref,
) {
  const keyAt = useCallback((i) => (getKey ? getKey(items[i], i) : i), [items, getKey]);
  const { containerRef, onScroll, virtualItems, totalSize, measureElement, scrollToIndex, range, isFixed } =
    useVirtualList({
      count: items.length,
      itemHeight,
      estimateHeight,
      overscan,
      getKey: getKey ? keyAt : undefined,
    });

  useImperativeHandle(ref, () => ({ scrollToIndex }), [scrollToIndex]);

  useEffect(() => {
    onRangeChange?.(range);
  }, [range.start, range.end, onRangeChange]);

  return (
    <div
      ref={containerRef}
      className="vl-scroller"
      style={{ height }}
      onScroll={onScroll}
      role="list"
      aria-label={label}
      tabIndex={0}
    >
      <div className="vl-spacer" style={{ height: totalSize }}>
        {virtualItems.map(({ index, key, start, size }) => (
          <div
            key={key}
            ref={measureElement}
            data-index={index}
            role="listitem"
            aria-setsize={items.length}
            aria-posinset={index + 1}
            className="vl-row"
            style={{ transform: `translateY(${start}px)`, height: isFixed ? size : undefined }}
          >
            {renderItem(items[index], index)}
          </div>
        ))}
      </div>
    </div>
  );
});

export default VirtualList;
```

**`App.jsx`**

```jsx
import { useRef, useState } from 'react';
import VirtualList from './VirtualList';
import './styles.css';

const SENTENCE = 'The quick brown fox jumps over the lazy dog. ';
const FIXED_ITEMS = Array.from({ length: 10000 }, (_, id) => ({ id, name: `Row ${id}` }));
const VARIABLE_ITEMS = Array.from({ length: 10000 }, (_, id) => ({
  id,
  text: SENTENCE.repeat(1 + ((id * 7) % 9)),
}));
const byId = (item) => item.id;

function Demo({ title, items, itemHeight, estimateHeight, renderItem }) {
  const listRef = useRef(null);
  const [target, setTarget] = useState('5000');
  const [align, setAlign] = useState('start');
  const [range, setRange] = useState({ start: 0, end: 0 });

  const jump = (e) => {
    e.preventDefault();
    const index = Number(target);
    if (Number.isInteger(index)) listRef.current?.scrollToIndex(index, { align });
  };

  return (
    <section className="panel">
      <h2>{title}</h2>
      <form className="toolbar" onSubmit={jump}>
        <label>
          Index{' '}
          <input type="number" min="0" max={items.length - 1} value={target} onChange={(e) => setTarget(e.target.value)} />
        </label>
        <label>
          Align{' '}
          <select value={align} onChange={(e) => setAlign(e.target.value)}>
            <option value="start">start</option>
            <option value="center">center</option>
            <option value="end">end</option>
            <option value="auto">auto</option>
          </select>
        </label>
        <button type="submit">Go</button>
        <button type="button" onClick={() => listRef.current?.scrollToIndex(0)}>Top</button>
        <button type="button" onClick={() => listRef.current?.scrollToIndex(items.length - 1, { align: 'end' })}>Bottom</button>
      </form>
      <p className="meta">
        Rendering rows {range.start}–{range.end} ({range.end - range.start + 1} of {items.length} in the DOM). Drag the
        bottom-right corner to resize.
      </p>
      <VirtualList
        ref={listRef}
        label={title}
        items={items}
        itemHeight={itemHeight}
        estimateHeight={estimateHeight}
        height={320}
        overscan={4}
        getKey={byId}
        renderItem={renderItem}
        onRangeChange={setRange}
      />
    </section>
  );
}

export default function App() {
  return (
    <main className="app">
      <h1>Virtualized List — 10,000 rows</h1>
      <Demo
        title="Fixed height (40px)"
        items={FIXED_ITEMS}
        itemHeight={40}
        renderItem={(item) => <div className="row">{item.name}</div>}
      />
      <Demo
        title="Variable height (measured)"
        items={VARIABLE_ITEMS}
        estimateHeight={60}
        renderItem={(item) => (
          <div className="msg">
            <strong>#{item.id}</strong> {item.text}
          </div>
        )}
      />
    </main>
  );
}
```

**`styles.css`**

```css
* { box-sizing: border-box; }
body { margin: 0; font-family: system-ui, sans-serif; background: #f1f5f9; color: #0f172a; }
.app { max-width: 720px; margin: 0 auto; padding: 24px 16px; }

.panel { margin: 16px 0; padding: 14px; background: #fff; border: 1px solid #cbd5e1; border-radius: 10px; }
.panel h2 { margin: 0 0 10px; font-size: 18px; }
.meta { font-size: 13px; color: #64748b; }

.toolbar { display: flex; flex-wrap: wrap; align-items: center; gap: 8px; margin-bottom: 10px; }
.toolbar input { width: 90px; padding: 5px 8px; border: 1px solid #94a3b8; border-radius: 6px; }
.toolbar select { padding: 5px 8px; border: 1px solid #94a3b8; border-radius: 6px; }
.toolbar button { padding: 6px 14px; border: 1px solid #94a3b8; border-radius: 6px; background: #fff; cursor: pointer; }
button:focus-visible, input:focus-visible, select:focus-visible, .vl-scroller:focus-visible { outline: 3px solid #f59e0b; outline-offset: 2px; }

.vl-scroller {
  position: relative;
  overflow: auto;
  overflow-anchor: none; /* we do our own scroll anchoring */
  resize: vertical;
  min-height: 120px;
  border: 1px solid #94a3b8;
  border-radius: 8px;
  background: #fff;
}
.vl-spacer { position: relative; width: 100%; }
.vl-row { position: absolute; top: 0; left: 0; right: 0; will-change: transform; }

.row { height: 100%; display: flex; align-items: center; padding: 0 12px; border-bottom: 1px solid #e2e8f0; }
.msg { padding: 10px 12px; border-bottom: 1px solid #e2e8f0; line-height: 1.4; }
```

</details>

---

## 6. How It Works

- **Spacer + absolute rows.** The spacer's height is `totalSize`, so the native scrollbar represents all 10,000 rows. Each mounted row is `position: absolute` and moved with `translateY(offset)` — a compositor-only property, no layout per scroll.
- **Range from `scrollTop`.** `getVisibleRange` finds the row at the top edge (`indexAt(scrollTop)`) and at the bottom edge (`indexAt(scrollTop + viewport)`), then widens by `overscan` and clamps to `[0, count-1]`. `scrollTop` is clamped to `total - viewport` first, so a stale value after the list shrinks can't ask for rows that no longer exist.
- **`metrics` is the seam between modes.** Fixed: `offsetOf(i) = i * h`, `indexAt(y) = floor(y / h)`. Variable: `offsets` is a `Float64Array` of prefix sums (`offsets[i+1] = offsets[i] + height(i)`), `indexAt` is a binary search for the largest `i` with `offsets[i] <= y`.
- **Binary search detail.** `mid = (lo + hi + 1) >> 1` (upper mid) with `lo = mid` / `hi = mid - 1` always terminates and returns the *last* offset `<= y`. `hi` starts at `count - 1` so `y` beyond the end returns the last row.
- **Measuring.** Rows register through the stable `measureElement` ref callback. One shared `ResizeObserver` receives a batch of entries; each real `offsetHeight` goes into a `Map` keyed by `getKey(i)`; if anything changed, `version` is bumped once and `useMemo` rebuilds `offsets`. Rows removed from the DOM are un-observed in a layout effect.
- **Scroll anchoring.** If a measured row's top is above `scrollTop`, its `measured − previous` delta is accumulated in `adjust`. After the re-render the layout effect adds it to `scrollTop` before paint, so nothing visibly shifts. The row at the viewport top keeps its on-screen position.
- **`scrollToIndex`.** `getScrollTarget` turns `(index, align)` into a clamped `scrollTop` (`start`: row top; `end`: row bottom − viewport; `center`; `auto`: scroll only if not fully visible). In variable mode the first jump uses estimates; if the target isn't measured yet it is stored in `pending`, and once its height arrives the layout effect re-aligns using the corrected offsets (and clears `adjust` so the two corrections don't stack).
- **Container resize.** A second `ResizeObserver` on the scroller re-reads `clientHeight` into state; the range is recomputed from the new viewport. The demo uses CSS `resize: vertical` to show it live.
- **Why `version` state + a mutable `Map`?** Measurements arrive outside React's render cycle and in bulk; mutating a `Map` and bumping a counter avoids copying a 10k-entry object per measurement.
- **Keys.** `getKey(items[i])` (the item id) is both the React key and the height-cache key, so reordering/prepending keeps heights attached to the right item.

---

## 7. Edge Cases

- **Empty list** → `getVisibleRange` returns `{ start: 0, end: -1 }`, zero rows, spacer height 0.
- **List shrinks while scrolled far down** → `scrollTop` is clamped against the new `total - viewport`; the browser also clamps and fires `scroll`.
- **Viewport height is 0 on the first render** (before layout) → we render `1 + overscan` rows, then the layout effect reads `clientHeight` and re-renders **before paint**. If the container is `display: none`, `clientHeight` is 0 until it's shown (the container `ResizeObserver` fires then).
- **Row height 0** (`display: none`, collapsed) → measurement ignored (`!height`) so a hidden row can't poison the cache with `0`.
- **Rows resize after first measure** (image loads, window narrows → text rewraps) → `ResizeObserver` fires again, offsets rebuild; deltas above the viewport are anchored.
- **`scrollToIndex` to an unmeasured row** → lands on an estimate, then self-corrects once measured. For huge estimate errors a second correction may be visible; real libraries accept the same trade-off.
- **`scrollToIndex` out of range** → index is clamped to `[0, count-1]`; the target `scrollTop` is clamped to `[0, total - viewport]` (last rows can't be aligned to `start`).
- **Fractional heights** → `offsetHeight` rounds; use `getBoundingClientRect().height` or `entry.borderBoxSize` if pixel-perfect accuracy matters (zoomed displays).
- **Margins on rows** → absolutely-positioned rows don't collapse/measure margins; use padding inside the row.
- **Browser native scroll anchoring** (`overflow-anchor`) would double-apply our correction → disabled on the scroller.
- **Smooth `scrollToIndex`** → `scroll` events drive the window; with estimated heights the final position can drift. We don't set `pending` for smooth scrolls.
- **Focus inside an unmounted row** → focus is lost when the row scrolls out; keep the focused index and keep that row mounted (or use roving tabindex with `scrollToIndex`).
- **Non-memoized `getKey`** in the hook → changes identity every render, rebuilding the prefix sums on every render. `VirtualList` memoizes it with `useCallback([items, getKey])`.
- **StrictMode** → the observer effects clean up and re-observe from the `elements` Set, so double-invoked effects don't lose measurements.

---

## 8. Accessibility

- The scroller is `role="list"` with an `aria-label`; each row is `role="listitem"` with **`aria-setsize`** (total count) and **`aria-posinset`** (index + 1) — without them a screen reader announces "1 of 14" for a 10,000-row list.
- The scroller has **`tabIndex={0}`** and a visible focus ring, so keyboard users can scroll it with Arrow / PageUp / PageDown / Home / End using the browser's native behaviour (no custom key handling needed).
- A "Jump to index" form, plus Top / Bottom buttons, gives a non-scroll way to navigate; the form uses real `<label>`s.
- Don't put a live region on the "rendering rows a–b" text — it would announce on every scroll tick.
- Interactive rows (links/buttons) need roving tabindex or a "mount the focused row" rule; otherwise Tab skips offscreen rows and focus disappears on unmount.
- Native Ctrl+F can't find unmounted rows — offer a search box that filters the data (and announce the result count), or use `aria-rowcount` + a data-grid pattern for tables.
- Respect `prefers-reduced-motion` before using `behavior: 'smooth'`.

---

## 9. Performance Considerations

- **DOM size is O(visible)**, not O(n): ~20 nodes instead of 10,000. That removes the layout/paint/style-recalc and memory cost that actually makes huge lists slow.
- **Fixed mode is O(1)** per scroll; **variable mode is O(log n)** per scroll (binary search) and O(n) per *measurement batch* (prefix-sum rebuild — 10k additions on a `Float64Array` is well under a millisecond). Batching all `ResizeObserver` entries into one `version` bump is what keeps it at one rebuild per frame.
- `translateY` + `will-change: transform` keeps scrolling on the compositor; avoid `top` (layout each time).
- **Overscan trade-off:** more overscan = fewer blank flashes during fast scroll but more nodes; 3–10 rows is typical. For heavy rows, overscan *less* but render rows cheaply.
- Every scroll event calls `setScrollTop`, so the list component re-renders per event. React 18 batches it to ≤ 1 render per frame; to cut renders further, store only `{start, end}` or throttle with `requestAnimationFrame`, and wrap row content in `React.memo` with stable props.
- `virtualItems` is memoized on `[metrics, start, end]`; the heights `Map` is mutated in place so measurements don't copy large objects.
- Don't create new `getKey` / `renderItem` identities that force a prefix-sum rebuild every render (`getKey` is in the memo's deps by design).
- **Scaling beyond ~100k rows with frequent resizes:** replace the prefix-sum array by a Fenwick tree (O(log n) point update + O(log n) search) or chunked sums.
- Browsers cap element height (~33M px in Chrome/Safari, ~17M px in Firefox) — a 1M-row list at 40px = 40M px breaks. Fix by scaling the scroll range (virtual scrollbar) or paginating.
- Measure with the Performance panel: scripting per scroll frame should be a few ms; check "Layout" isn't triggered by scroll.

---

## 10. Follow-up Questions

<details>
<summary><strong>1. Why not just paginate or use <code>content-visibility: auto</code>?</strong></summary>

Pagination changes the UX (no continuous scroll, no global scroll position). `content-visibility: auto` skips rendering work for offscreen rows but **keeps all nodes in the DOM** — memory and style-recalc still scale with n, and you must provide `contain-intrinsic-size`. It's a great zero-JS option for ~1–5k rows; windowing is for 10k+ or heavy rows. They can be combined.

</details>

<details>
<summary><strong>2. How do you implement a chat that sticks to the bottom and loads older messages at the top?</strong></summary>

Track "is at bottom" (`scrollHeight - scrollTop - clientHeight < threshold`); when new messages arrive and you were at the bottom, `scrollToIndex(last, { align: 'end' })`. When **prepending** older messages, keys must be stable (so heights stay attached) and the scroll anchoring step must add the prepended rows' total height to `scrollTop` so the viewport doesn't jump. Trigger the load from the top sentinel / `start === 0`.

</details>

<details>
<summary><strong>3. How would you support a grid (rows × columns) or horizontal list?</strong></summary>

Virtualize each axis independently: two `metrics` (row + column), two ranges, and render the cartesian product of visible rows × visible columns, positioned with `translate(x, y)`. A horizontal list is the same hook with `scrollLeft` / `clientWidth` / widths. For a grid of equal cards, compute `columns = floor(containerWidth / cardWidth)` and virtualize *rows of cards* — which is where container resize handling matters.

</details>

<details>
<summary><strong>4. Why binary search over a prefix-sum array instead of a loop?</strong></summary>

A loop from 0 is O(n) per scroll event — 10k iterations × 60 events/s is wasteful and grows with the list. Prefix sums are sorted by construction, so binary search finds the start row in ~14 comparisons for 10k rows (~20 for 1M). Rebuilding the sums is O(n) but only happens on measurement batches, not on every scroll.

</details>

<details>
<summary><strong>5. How does scroll anchoring work, and why disable <code>overflow-anchor</code>?</strong></summary>

When a row above the viewport grows by Δ, all content below moves down by Δ; if `scrollTop` doesn't change the user sees a jump. We accumulate Δ for rows with `offset < scrollTop` and add it to `scrollTop` in a layout effect (before paint). Chrome/Firefox also implement native anchoring via `overflow-anchor`; with absolutely-positioned, transformed rows it's unreliable and would stack with ours, so we set `overflow-anchor: none` and own the behaviour.

</details>

<details>
<summary><strong>6. How do you make <code>scrollToIndex</code> accurate when rows are unmeasured?</strong></summary>

Jump with estimates, render the neighbourhood, measure, then re-run alignment once the target is measured (our `pending` ref). Alternatives: pre-measure with an offscreen "ruler" element, compute heights from data (text length × font metrics, `canvas.measureText` / `pretext`-style layout), or iterate until offsets stop changing (bounded to a few passes). A good `estimateHeight` (average of measured rows so far) makes the first jump land close.

</details>

<details>
<summary><strong>7. react-window vs react-virtuoso vs TanStack Virtual vs this — which would you use in production?</strong></summary>

| | This solution | `react-window` | `react-virtuoso` | `@tanstack/react-virtual` |
|---|---|---|---|---|
| Fixed height | ✅ | ✅ `FixedSizeList` | ✅ | ✅ |
| Variable / **auto-measured** height | ✅ (ResizeObserver) | ⚠️ `VariableSizeList`: you must **supply** `itemSize(i)`; no auto-measure | ✅ automatic | ✅ `measureElement` |
| Scroll anchoring on measure | ✅ basic | ❌ | ✅ robust | ✅ |
| `scrollToIndex` with unmeasured rows | ✅ re-align once | ⚠️ exact only if sizes are known | ✅ | ✅ |
| Container resize | ✅ | needs `AutoSizer` | ✅ built-in | ✅ |
| Prepend / reverse (chat) | ❌ | ❌ | ✅ `firstItemIndex`, `followOutput` | ⚠️ manual |
| Grid / sticky / grouped | ❌ | grid ✅, sticky manual | ✅ groups, sticky headers | ✅ headless, you build it |
| Windowing for window scroll | ❌ | ❌ | ✅ | ✅ `useWindowVirtualizer` |
| Bundle size | ~2 KB | ~6 KB | ~17 KB+ | ~4 KB (headless) |
| Style | hook + component | components | components | headless hook |

Rule of thumb: known fixed sizes → `react-window`; unknown/dynamic sizes, chat, grouping → `react-virtuoso`; own markup/design system or tables → TanStack Virtual (headless, same idea as our hook). In an interview, build the hook; in production, use the library.

</details>

<details>
<summary><strong>8. How do you test it?</strong></summary>

Unit-test the pure functions: prefix sums, binary search vs a linear-scan oracle on random heights, range clamping, and `getScrollTarget` for each `align`. For the component (jsdom has no layout): stub `clientHeight` (viewport), set `scrollTop` and fire `scroll`, replace `ResizeObserver` with a fake you can trigger, and stub `offsetHeight` per `data-index`. Assert the rendered index range, `translateY` values, spacer height, and that `scrollTop` is corrected after measurement. Real scrolling/anchoring behaviour needs a browser test (Playwright).

</details>

---

## 11. Common Mistakes

- Rendering all rows and "hiding" them with CSS — that's not virtualization.
- Forgetting the **spacer** (or its height) → the scrollbar is wrong/absent.
- Computing the start row with a linear scan on every scroll event.
- Using `top` instead of `transform` (layout on every scroll) or re-creating inline style objects needlessly.
- No **overscan** → blank flashes on fast scrolls; **too much** overscan → no gain.
- Using the array **index as the key** in variable mode, so cached heights attach to the wrong item after insert/reorder.
- Re-measuring and re-rendering **per row** instead of batching a `ResizeObserver` callback into one update.
- Not compensating `scrollTop` when a row **above** the viewport changes height → content jumps while scrolling up.
- Using **margins** on rows (not included in the measured height) → overlapping rows.
- Reading `clientHeight` / `getBoundingClientRect` in the scroll handler (forced layout) — read the viewport once and on resize.
- Ignoring **container resize** → fixed-viewport math goes stale when the panel or window changes size.
- Treating `end` as exclusive in one place and inclusive in another (off-by-one — last visible row missing).
- Not clamping `scrollTop` / `scrollToIndex` when the data shrinks or the index is out of range.
- Letting browser `overflow-anchor` and manual anchoring both run.
- Forgetting a11y: no `aria-setsize` / `aria-posinset`, no keyboard-scrollable container.
