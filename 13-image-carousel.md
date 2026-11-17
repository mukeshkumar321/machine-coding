# Image Carousel (Autoplay, Infinite Loop, Swipe) — Machine Coding

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

Build an **image carousel / slider** with:

- Prev / Next arrows and **dot indicators**
- **Autoplay** (every N seconds) that **pauses on hover**
- **Infinite loop** — last → first (and first → last) without a visible rewind
- **Swipe / drag** support for touch and mouse

---

## 2. Requirements

### Functional

- Show one slide at a time with a sliding transition.
- Arrows move ±1; dots jump to a slide; the active dot is highlighted.
- Autoplay advances every `interval` ms; any manual navigation restarts the timer.
- Pause autoplay on hover, keyboard focus, hidden tab, and via an explicit Pause button.
- Seamless infinite loop in both directions.
- Swipe left/right (pointer events) to navigate; follows the finger while dragging; snaps back if the drag is short.
- Arrow keys (← →) navigate.

### Non-functional

- All timers/listeners cleaned up; no stale closures.
- Works with 0, 1, 2 … N slides.
- Respects `prefers-reduced-motion`; accessible controls (pause, labels, live region).
- Navigation logic isolated in a hook (`useCarousel`).

---

## 3. Clarifying Questions to Ask

- Infinite loop with a **slide animation**, or is wrap-around with a fade acceptable?
- Autoplay interval? Should user interaction permanently stop it or just restart it?
- Multiple slides visible at once? Variable slide widths?
- Lazy-load images? Fixed aspect ratio?
- Mobile swipe required? Vertical scroll must still work?
- Thumbnails / progress bar needed?

---

## 4. Approach & Component Design

**Track with cloned edges** — to loop seamlessly, render `[clone of last, ...slides, clone of first]`. The track is a flex row translated by `-pos * 100%`.

```
 pos:     0        1     2     3    ...   n       n+1
        [last']  [ 1 ] [ 2 ] [ 3 ]  ... [ n ]  [first']
           ▲ snap here (no animation)            ▲
           └─ after sliding onto a clone, jump (no transition) to the real slide: 0 → n, n+1 → 1
```

**State (inside `useCarousel`)** — one object so position and animation flag update atomically:

| State | Meaning |
|---|---|
| `pos` | index in the *extended* track (`1..n` are real slides; `0` and `n+1` are clones) |
| `animate` | CSS transition on/off (off for the invisible snap) |

**Derived:** `activeIndex = ((pos - 1) % n + n) % n` (works for the clone positions too).

**UI state in `Carousel`:** `hovered`, `focused` (focus-visible only), `userPaused`, `tabHidden`, `dragging`, `dragX`. `paused = any of them`.

```
Carousel
 ├─ useCarousel({ count, autoplay, interval, paused }) → { pos, animate, activeIndex, go, goTo, trackRef }
 ├─ viewport (overflow hidden, pointer handlers for swipe)
 │    └─ track (flex, transform: translateX(calc(-pos*100% + dragX px)))
 │         └─ slide × (n + 2)
 ├─ ‹ ›  arrow buttons
 ├─ dots  (one button per real slide)
 └─ pause / play button
```

---

## 5. Reference Solution

<details>
<summary><strong>Show Full Code (useCarousel.js, Carousel.jsx, App.jsx, styles.css)</strong></summary>

```jsx
// useCarousel.js
import { useCallback, useEffect, useLayoutEffect, useRef, useState } from 'react';

export const TRANSITION_MS = 400;

export function useCarousel({ count, autoplay = true, interval = 3000, paused = false }) {
  const loop = count > 1;
  // pos indexes the extended track [clone(last), ...slides, clone(first)]
  const [view, setView] = useState({ pos: loop ? 1 : 0, animate: true });
  const { pos, animate } = view;
  const trackRef = useRef(null);

  const atClone = loop && (pos === 0 || pos === count + 1);

  // While resting on a clone (waiting to snap) ignore input so we never run past it.
  const go = useCallback(
    (delta) => {
      if (!loop) return;
      setView((v) =>
        v.pos === 0 || v.pos === count + 1 ? v : { pos: v.pos + delta, animate: true }
      );
    },
    [loop, count]
  );

  const goTo = useCallback(
    (index) => {
      if (!loop) return;
      setView((v) =>
        v.pos === 0 || v.pos === count + 1 ? v : { pos: index + 1, animate: true }
      );
    },
    [loop, count]
  );

  // 1) after the slide-onto-clone animation finishes, jump to the real slide without animating
  useEffect(() => {
    if (!atClone) return;
    const id = setTimeout(() => {
      setView({ pos: pos === 0 ? count : 1, animate: false });
    }, TRANSITION_MS);
    return () => clearTimeout(id);
  }, [atClone, pos, count]);

  // 2) once the un-animated jump is committed, force a reflow and turn transitions back on
  useLayoutEffect(() => {
    if (animate) return;
    trackRef.current?.getBoundingClientRect();
    setView((v) => (v.animate ? v : { ...v, animate: true }));
  }, [animate]);

  // 3) autoplay: a fresh timeout per position, so manual navigation restarts the countdown
  useEffect(() => {
    if (!autoplay || paused || !loop) return;
    const id = setTimeout(() => go(1), interval);
    return () => clearTimeout(id);
  }, [autoplay, paused, loop, interval, pos, go]);

  const activeIndex = loop ? (((pos - 1) % count) + count) % count : 0;

  return { loop, pos, animate, activeIndex, trackRef, go, goTo };
}
```

```jsx
// Carousel.jsx
import { useEffect, useRef, useState } from 'react';
import { TRANSITION_MS, useCarousel } from './useCarousel';

function usePageHidden() {
  const [hidden, setHidden] = useState(() => document.hidden);
  useEffect(() => {
    const onChange = () => setHidden(document.hidden);
    document.addEventListener('visibilitychange', onChange);
    return () => document.removeEventListener('visibilitychange', onChange);
  }, []);
  return hidden;
}

export default function Carousel({ slides, autoplay = true, interval = 3000 }) {
  const count = slides.length;

  const [hovered, setHovered] = useState(false);
  const [focused, setFocused] = useState(false);
  const [userPaused, setUserPaused] = useState(false);
  const [dragging, setDragging] = useState(false);
  const [dragX, setDragX] = useState(0);
  const tabHidden = usePageHidden();

  const startX = useRef(0);
  const dxRef = useRef(0);

  const paused = hovered || focused || userPaused || tabHidden || dragging;
  const { loop, pos, animate, activeIndex, trackRef, go, goTo } = useCarousel({
    count, autoplay, interval, paused,
  });

  if (count === 0) return null;

  const items = loop ? [slides[count - 1], ...slides, slides[0]] : slides;
  const autoplayRunning = autoplay && loop && !paused;

  const onPointerDown = (e) => {
    if (!loop || (e.pointerType === 'mouse' && e.button !== 0)) return;
    startX.current = e.clientX;
    dxRef.current = 0;
    e.currentTarget.setPointerCapture(e.pointerId);
    setDragging(true);
    setDragX(0);
  };

  const onPointerMove = (e) => {
    if (!dragging) return;
    dxRef.current = e.clientX - startX.current;
    setDragX(dxRef.current);
  };

  const onPointerUp = (e) => {
    if (!dragging) return;
    const threshold = Math.min(e.currentTarget.offsetWidth * 0.2, 80);
    if (Math.abs(dxRef.current) > threshold) go(dxRef.current < 0 ? 1 : -1);
    setDragging(false);
    setDragX(0);
  };

  const onPointerCancel = () => {
    setDragging(false);
    setDragX(0);
  };

  const onKeyDown = (e) => {
    if (e.key === 'ArrowLeft') go(-1);
    if (e.key === 'ArrowRight') go(1);
  };

  return (
    <section
      className="carousel"
      aria-roledescription="carousel"
      aria-label="Featured images"
      onMouseEnter={() => setHovered(true)}
      onMouseLeave={() => setHovered(false)}
      onFocus={(e) => {
        if (e.target.matches(':focus-visible')) setFocused(true);
      }}
      onBlur={(e) => {
        if (!e.currentTarget.contains(e.relatedTarget)) setFocused(false);
      }}
      onKeyDown={onKeyDown}
    >
      <div
        className="viewport"
        onPointerDown={onPointerDown}
        onPointerMove={onPointerMove}
        onPointerUp={onPointerUp}
        onPointerCancel={onPointerCancel}
      >
        <div
          ref={trackRef}
          className="track"
          aria-live={autoplayRunning ? 'off' : 'polite'}
          style={{
            transform: `translateX(calc(${-pos * 100}% + ${dragX}px))`,
            transition: animate && !dragging ? `transform ${TRANSITION_MS}ms ease` : 'none',
          }}
        >
          {items.map((slide, idx) => {
            const isClone = loop && (idx === 0 || idx === count + 1);
            const realIndex = loop ? (idx - 1 + count) % count : idx;
            return (
              <div
                key={isClone ? `clone-${idx}` : slide.id}
                className="slide"
                role="group"
                aria-roledescription="slide"
                aria-label={`${realIndex + 1} of ${count}`}
                aria-hidden={isClone || realIndex !== activeIndex}
              >
                <div className="slide-inner" style={{ background: slide.bg }}>
                  <h2>{slide.title}</h2>
                </div>
              </div>
            );
          })}
        </div>
      </div>

      {loop && (
        <>
          <button className="arrow prev" aria-label="Previous slide" onClick={() => go(-1)}>‹</button>
          <button className="arrow next" aria-label="Next slide" onClick={() => go(1)}>›</button>

          <div className="dots" role="group" aria-label="Choose slide">
            {slides.map((slide, i) => (
              <button
                key={slide.id}
                className={`dot${i === activeIndex ? ' active' : ''}`}
                aria-label={`Go to slide ${i + 1}`}
                aria-current={i === activeIndex}
                onClick={() => goTo(i)}
              />
            ))}
          </div>

          {autoplay && (
            <button className="pause" onClick={() => setUserPaused((p) => !p)}>
              {userPaused ? '▶ Play' : '❚❚ Pause'}
            </button>
          )}
        </>
      )}
    </section>
  );
}
```

```jsx
// App.jsx
import Carousel from './Carousel';
import './styles.css';

const SLIDES = [
  { id: 1, title: 'Mountains', bg: 'linear-gradient(135deg,#4facfe,#00f2fe)' },
  { id: 2, title: 'Forest', bg: 'linear-gradient(135deg,#43e97b,#38f9d7)' },
  { id: 3, title: 'Desert', bg: 'linear-gradient(135deg,#fa709a,#fee140)' },
  { id: 4, title: 'Ocean', bg: 'linear-gradient(135deg,#667eea,#764ba2)' },
  { id: 5, title: 'City', bg: 'linear-gradient(135deg,#f093fb,#f5576c)' },
];

export default function App() {
  return (
    <main className="page">
      <h1>Image Carousel</h1>
      {/* remount with a new key if `slides` changes length */}
      <Carousel key={SLIDES.length} slides={SLIDES} autoplay interval={3000} />
    </main>
  );
}
```

```css
/* styles.css */
.page { max-width: 720px; margin: 24px auto; font-family: system-ui, sans-serif; }

.carousel { position: relative; }
.viewport { overflow: hidden; border-radius: 12px; touch-action: pan-y; user-select: none; cursor: grab; }
.viewport:active { cursor: grabbing; }
.track { display: flex; will-change: transform; }
.slide { flex: 0 0 100%; min-width: 0; }
.slide-inner { height: 320px; display: grid; place-items: center; color: #fff; text-shadow: 0 2px 6px rgba(0, 0, 0, 0.35); }
.slide-inner h2 { font-size: 40px; margin: 0; }
/* with real images: .slide img { width: 100%; height: 320px; object-fit: cover; display: block; } */

.arrow { position: absolute; top: 50%; transform: translateY(-50%); width: 40px; height: 40px; border: 0; border-radius: 50%; background: rgba(0, 0, 0, 0.45); color: #fff; font-size: 24px; cursor: pointer; }
.arrow:hover { background: rgba(0, 0, 0, 0.7); }
.prev { left: 10px; }
.next { right: 10px; }

.dots { display: flex; justify-content: center; gap: 8px; margin-top: 10px; }
.dot { width: 12px; height: 12px; padding: 0; border: 0; border-radius: 50%; background: #cbd5e1; cursor: pointer; }
.dot.active { background: #2563eb; transform: scale(1.25); }
.pause { position: absolute; top: 10px; right: 10px; padding: 4px 10px; border: 0; border-radius: 14px; background: rgba(0, 0, 0, 0.5); color: #fff; cursor: pointer; font-size: 12px; }

.carousel button:focus-visible { outline: 3px solid #f59e0b; outline-offset: 2px; }

@media (prefers-reduced-motion: reduce) {
  .track { transition: none !important; }
}
```

</details>

---

## 6. How It Works

- **Translate, don't scroll.** The track is `display: flex`; each slide is `flex: 0 0 100%`; `translateX(-pos * 100%)` moves by whole viewport widths (the `%` in `translate` refers to the *track's* own width = viewport width). It's GPU-accelerated and cheap.
- **Infinite loop with clones.** Sliding *forward* from the last real slide lands on a clone of slide 1 (looks identical). After the transition ends, we jump to the real slide 1 with `transition: none` — invisible. The reverse works for the first slide.
- **Atomic snap.** `{pos, animate}` live in one state object, so "jump + disable animation" is a single render. A `useLayoutEffect` then forces a reflow (`getBoundingClientRect`) and re-enables transitions *before paint*, so the jump isn't animated. (A `requestAnimationFrame` double-hop is the other common technique.)
- **Locked on clones.** While on a clone `go/goTo` are no-ops; otherwise rapid clicks could push `pos` beyond `n+1`.
- **Autoplay = one `setTimeout` per position**, with `pos` in deps: any manual navigation changes `pos` → cleanup clears the old timer and starts a new full countdown. Cleanup also covers unmount and pause. `go` uses the functional `setView` form, so there's no stale closure.
- **Pause sources are separate booleans** (hover, focus-visible, user, hidden tab, dragging) OR-ed together — easy to reason about and extend.
- **Swipe via Pointer Events** (covers mouse + touch + pen): `setPointerCapture` keeps events flowing outside the element; `dx` follows the pointer by adding `dragX px` to the transform with transitions disabled; on release, `|dx| > min(20% width, 80px)` triggers `go(±1)`, otherwise it animates back. `touch-action: pan-y` lets the browser keep vertical scroll but hands horizontal gestures to us. `dxRef` holds the latest value so `onPointerUp` never reads a stale state.
- **Simplest alternative** (mention it): `index = (index + 1) % n` with `translateX(-index*100%)`. Wrapping then visibly rewinds across all slides; fine for a quick MVP, not "seamless".

---

## 7. Edge Cases

- `slides.length === 0` → render nothing; `1` → no clones, no arrows/dots/autoplay; `2` → clones still work.
- Rapid clicking / holding the arrow key during the transition (locked while on a clone).
- Click on a dot for the slide that's already active → nothing changes (`pos` is the same, so the autoplay countdown is *not* restarted; add a counter to the timer's deps if you want that).
- Tab hidden: browsers throttle timers; we pause so slides don't pile up.
- Hover on touch devices (sticky hover after tap) — also pause on focus/drag; consider `(hover: hover)` media check.
- Dragging past the first/last slide (works because of the clones); starting a drag on a link inside a slide fires `click` afterwards — suppress it if movement > few px.
- Window resize: `%` based layout adapts automatically; px thresholds use `offsetWidth`.
- `slides` prop length changes at runtime → remount with `key` (or reset `pos`).
- `prefers-reduced-motion`: transitions off (the snap still happens after 400 ms, invisible because the clone looks identical); consider defaulting autoplay to off.
- Images not loaded yet → fixed height/aspect-ratio to avoid layout shift.

---

## 8. Accessibility

- Container: `aria-roledescription="carousel"` + label; each slide `role="group"`, `aria-roledescription="slide"`, `aria-label="2 of 5"`; clones and off-screen slides `aria-hidden`.
- **Autoplay must be pausable** (WCAG 2.2.2): visible Pause/Play button + pause on hover and keyboard focus.
- `aria-live="off"` while autoplay runs (otherwise screen readers get spammed), `"polite"` when it's paused/manual.
- Arrow keys navigate; arrows/dots are real `<button>`s with `aria-label`s; `aria-current` on the active dot.
- Meaningful `alt` on real images (`alt=""` if decorative); don't rely on dot colour alone (size/scale also changes).
- Respect `prefers-reduced-motion`.

---

## 9. Performance Considerations

- Animate only `transform` (compositor-only); add `will-change: transform` on the track.
- Pointer-move updates a single state value → one cheap re-render per move; for heavy slides wrap `Slide` in `React.memo` or write the transform straight to the DOM via `ref` while dragging and commit state on release.
- Lazy-load images (`loading="lazy"`), but eagerly load the neighbours of the active slide; use correct `srcset`/sizes and fixed aspect ratio.
- Don't render hundreds of slides — window them (active ± 1) when the list is long.
- Clean up timers/listeners (done in effect cleanups); one timeout at a time, never `setInterval` + manual resets.

---

## 10. Follow-up Questions

<details>
<summary><strong>1. Why `setTimeout` per slide instead of `setInterval`?</strong></summary>

With `setInterval` you must manually clear and recreate it whenever the user navigates, and a closure over `index` can go stale. A timeout keyed on `pos` is recreated by the effect automatically — navigation restarts the countdown, pause/unmount clear it, and the callback always uses the functional update.

</details>

<details>
<summary><strong>2. How would you implement the infinite loop without clones?</strong></summary>

Keep only the real slides but position each with `order`/computed offsets relative to the active index (circular distance), so the "next" slide is always placed to the right: `offset = ((i - active + n + half) % n) - half` → `translateX(offset*100%)`. Moving wraps by changing offsets; to avoid a slide flying across, disable its transition when its offset wraps. More complex than clones, but works for multi-slide views.

</details>

<details>
<summary><strong>3. Show 3 slides at once / variable widths?</strong></summary>

Slide width = `100% / perView`; translate by `-(pos * 100 / perView)%`; clones become `perView` items on each side. For variable widths measure slide offsets with `ref`s / ResizeObserver and translate by pixel offsets instead of percentages.

</details>

<details>
<summary><strong>4. How do you add a progress bar for the autoplay?</strong></summary>

Render the bar only while autoplay is running (`!paused`), keyed by `pos`, with a CSS animation `width: 0 → 100%` of duration `interval`. Keying by `pos` restarts it on every navigation, and unmounting it on pause makes it restart from 0 on resume — exactly like the timer, which also starts a full countdown again. (Using `animation-play-state: paused` instead would resume mid-way and drift out of sync.) No JS per frame.

</details>

<details>
<summary><strong>5. How would you support velocity-based swipes (flick)?</strong></summary>

Record timestamps in pointer-move; on release compute `velocity = dx / dt`. Navigate if `|dx| > threshold` **or** `|velocity| > ~0.5 px/ms`. For multi-slide jumps, multiply by velocity and clamp.

</details>

<details>
<summary><strong>6. How do you preload / lazy load images efficiently?</strong></summary>

`loading="lazy"` on non-visible slides, but preload `active ± 1` (set `loading="eager"` or create `Image()` objects in an effect). Use `decoding="async"`, responsive `srcset`, and reserve space (`aspect-ratio`) to avoid layout shift.

</details>

<details>
<summary><strong>7. Why `getBoundingClientRect()` in the layout effect?</strong></summary>

Browsers batch style changes; if we removed and re-added the transition in the same frame without the browser computing styles in between, it would still animate the jump. Reading layout forces a synchronous style/layout flush, so the jump is committed with `transition: none` before we turn transitions on again.

</details>

<details>
<summary><strong>8. Controlled vs uncontrolled carousel API?</strong></summary>

Uncontrolled (internal state + `onChange(index)` callback) is easiest. For a controlled version accept `index` and `onIndexChange`; the clone/snap logic still needs internal `pos`, so derive `pos = index + 1` and animate wraps explicitly. Expose `ref` methods (`next()`, `prev()`, `goTo(i)`) via `useImperativeHandle` for imperative control.

</details>

---

## 11. Common Mistakes

- Autoplay with `setInterval` and no cleanup → multiple timers after hover/unmount; or the callback closes over a stale `index`.
- Infinite loop done with `% n` only → the track visibly rewinds across all slides.
- Snap-back to the real slide while the transition is still enabled (animated jump) — or snapping on `transitionend` and never handling "transition didn't run".
- Allowing clicks to run past the clone (pos goes to `n+2`) → blank carousel.
- Not restarting the autoplay timer after manual navigation (slide changes immediately after a click).
- Using `onMouseMove`/`mousedown` only → no touch support; forgetting `touch-action` so vertical scrolling or the swipe breaks.
- Autoplay that can't be paused; `aria-live="polite"` while autoplaying.
- Layout via `left`/`margin` animation (janky) instead of `transform`.
- Forgetting the 1-slide / 0-slide cases.
- Pausing on focus for mouse clicks (use `:focus-visible`), so autoplay never resumes.

