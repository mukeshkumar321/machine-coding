# JS Polyfills & Utilities — Machine Coding

> **Difficulty:** Medium | **Time:** 30–45 min | **Frequency:** ⭐⭐⭐⭐⭐

## Table of Contents

1. [Debounce](#1-debounce)
2. [Throttle](#2-throttle)
3. [Promise.all / allSettled / race / any](#3-promiseall--allsettled--race--any)
4. [Function.prototype.call / apply / bind](#4-functionprototypecall--apply--bind)
5. [Event Emitter](#5-event-emitter)
6. [Deep Clone](#6-deep-clone)
7. [Memoize](#7-memoize)
8. [Curry](#8-curry)
9. [Flatten](#9-flatten)
10. [Pipe / Compose](#10-pipe--compose)
11. [Array.prototype.map / filter / reduce](#11-arrayprototypemap--filter--reduce)
12. [Once](#12-once)
13. [Sleep + Retry with Exponential Backoff](#13-sleep--retry-with-exponential-backoff)
14. [Tricky Output Questions](#14-tricky-output-questions)

---

> **How to use this file:** every utility is a standalone ES module (`// name.js` on the first line of the code block). Interviewers are rarely checking whether you remember the API — they check **closures for state**, **`this`/arguments forwarding**, **edge cases (empty input, errors, holes, cycles)** and whether you can **explain the trade-off out loud**. Write the simple version first, then layer on options (leading/trailing, cancel, depth, …).

---

## 1. Debounce

Runs `fn` only after calls have stopped for `wait` ms (search box, autosave, resize). Supports `leading` / `trailing` edges, `cancel()` and `flush()`.

### Code

```js
// debounce.js
export function debounce(fn, wait = 300, { leading = false, trailing = true } = {}) {
  let timerId = null;
  let lastArgs = null;
  let lastThis = null;
  let result;

  function invoke() {
    const args = lastArgs;
    const ctx = lastThis;
    lastArgs = lastThis = null; // a call is consumed exactly once
    result = fn.apply(ctx, args);
    return result;
  }

  function debounced(...args) {
    lastArgs = args;
    lastThis = this;
    const isFirstCallInBurst = timerId === null;

    clearTimeout(timerId);
    timerId = setTimeout(() => {
      timerId = null;
      if (trailing && lastArgs) invoke();
      lastArgs = lastThis = null;
    }, wait);

    if (leading && isFirstCallInBurst) invoke();
    return result; // value of the last real invocation
  }

  debounced.cancel = () => {
    clearTimeout(timerId);
    timerId = null;
    lastArgs = lastThis = null;
  };

  debounced.flush = () => {
    if (timerId === null) return result;
    clearTimeout(timerId);
    timerId = null;
    if (trailing && lastArgs) invoke();
    lastArgs = lastThis = null;
    return result;
  };

  debounced.pending = () => timerId !== null;

  return debounced;
}
```

### Usage

```js
// trailing (default): fire once, 300 ms after the user stops typing
const search = debounce((q) => fetch(`/api?q=${q}`), 300);
input.addEventListener('input', (e) => search(e.target.value));

// leading only: fire immediately, ignore the burst that follows (double-click guard)
const submit = debounce(sendForm, 1000, { leading: true, trailing: false });

// autosave + flush before leaving the page + cancel on teardown
const save = debounce(saveDraft, 1000);
window.addEventListener('beforeunload', () => save.flush());
// component unmount: save.cancel();

// leading + trailing: first call runs now, the LAST call of the burst runs later
const both = debounce(log, 100, { leading: true, trailing: true });
both('a'); // runs now → log('a')
both('b'); // nothing yet
both('c'); // 100 ms after this → log('c')   ('b' is dropped)
```

### Gotchas / interview follow-ups

- **Debounce vs throttle:** debounce = wait for a *quiet period* (search, autosave); throttle = *at most once per N ms* (scroll, drag, resize).
- **Timer reset on every call** is the whole algorithm: `clearTimeout` + `setTimeout`. State (`timerId`, `lastArgs`) lives in the closure.
- **Preserve `this` and args** — use a `function` (not an arrow) for `debounced` and `fn.apply(ctx, args)`, otherwise it breaks when used as an object method.
- **Leading + trailing:** the leading call *consumes* its args (`invoke` clears `lastArgs`), so a single call fires **once**, not twice. Trailing fires only if there was another call in the burst.
- **`flush()`** runs the pending call immediately (e.g. `beforeunload`); **`cancel()`** drops it (e.g. unmount).
- **Stale value:** the return value is the result of the *previous* invocation — debounced functions can't return fresh results synchronously. Return a promise wrapper if callers need the result.
- **In React** the debounced function must be created once (`useMemo`/`useRef`), not in the render body — see `25-utility-hooks.md`.
- Lodash adds `maxWait` (guarantee a call at least every N ms during a long burst) — that is exactly a throttle/debounce hybrid; good follow-up to mention.

---

## 2. Throttle

Runs `fn` at most once per `wait` ms. Leading edge fires immediately; trailing edge guarantees the **last** call in the window is not lost.

### Code

```js
// throttle.js
export function throttle(fn, wait = 300, { leading = true, trailing = true } = {}) {
  let timerId = null;
  let lastInvokeTime = null;
  let lastArgs = null;
  let lastThis = null;
  let result;

  function invoke() {
    const args = lastArgs;
    const ctx = lastThis;
    lastArgs = lastThis = null;
    result = fn.apply(ctx, args);
    return result;
  }

  function throttled(...args) {
    const now = Date.now();
    if (lastInvokeTime === null && !leading) lastInvokeTime = now;

    const remaining = wait - (now - lastInvokeTime);
    lastArgs = args;
    lastThis = this;

    if (lastInvokeTime === null || remaining <= 0 || remaining > wait) {
      // window is open (or the clock jumped backwards): run now
      clearTimeout(timerId);
      timerId = null;
      lastInvokeTime = now;
      invoke();
    } else if (timerId === null && trailing) {
      // inside the window: schedule ONE trailing call with the latest args
      timerId = setTimeout(() => {
        timerId = null;
        lastInvokeTime = leading ? Date.now() : null;
        if (lastArgs) invoke();
      }, remaining);
    }
    return result;
  }

  throttled.cancel = () => {
    clearTimeout(timerId);
    timerId = null;
    lastInvokeTime = null;
    lastArgs = lastThis = null;
  };

  return throttled;
}
```

### Usage

```js
// scroll handler: at most once per 200 ms, and the final position is always processed
window.addEventListener('scroll', throttle(updateProgressBar, 200));

const t = throttle((x) => console.log('run', x), 100);
t('a'); // t=0   → runs immediately: run a
t('b'); // t=10  → ignored (window open until 100), remembered as lastArgs
t('c'); // t=20  → replaces 'b'; trailing timer fires at t=100: run c

// trailing-only: behaves like "sample the last value every N ms"
const sample = throttle(report, 1000, { leading: false });
```

### Gotchas / interview follow-ups

- **Two implementations:** timestamp-based (leading only, simplest) and timer-based (trailing only). The version above combines both: timestamp decides *whether the window is open*, the timer delivers the *trailing* call.
- **Always keep the latest args** — the trailing call must use the most recent arguments, not those of the first ignored call.
- **Single timer:** only schedule a trailing timer if none exists (`timerId === null`), otherwise every ignored call stacks another timeout.
- **`Date.now()` can go backwards** (system clock change) — `remaining > wait` handles that; `performance.now()` is monotonic and safer in browsers.
- **Throttle vs `requestAnimationFrame`:** for visual updates rAF-throttling (one run per frame) is usually better than a fixed 100 ms.
- **Testing:** use fake timers (`vi.useFakeTimers()` / `jest.useFakeTimers()`) — they also mock `Date.now()`.
- **Debounce = throttle with `leading: false` + resetting timer** — know the difference: throttle guarantees *progress during* a long burst, debounce waits for the burst to *end*.

---

## 3. Promise.all / allSettled / race / any

All four take an iterable (values or promises) and return a new promise; they differ only in **when** they settle and **what** they settle with.

### Code

```js
// promises.js
export function promiseAll(iterable) {
  return new Promise((resolve, reject) => {
    const items = Array.from(iterable);
    const results = new Array(items.length);
    let remaining = items.length;
    if (remaining === 0) return resolve(results);

    items.forEach((item, i) => {
      Promise.resolve(item).then((value) => {
        results[i] = value; // by index: output order = input order, not completion order
        if (--remaining === 0) resolve(results);
      }, reject); // first rejection wins; later settles are ignored
    });
  });
}

export function promiseAllSettled(iterable) {
  return new Promise((resolve) => {
    const items = Array.from(iterable);
    const results = new Array(items.length);
    let remaining = items.length;
    if (remaining === 0) return resolve(results);

    const done = (i, outcome) => {
      results[i] = outcome;
      if (--remaining === 0) resolve(results);
    };
    items.forEach((item, i) => {
      Promise.resolve(item).then(
        (value) => done(i, { status: 'fulfilled', value }),
        (reason) => done(i, { status: 'rejected', reason }),
      );
    });
  });
}

export function promiseRace(iterable) {
  return new Promise((resolve, reject) => {
    // empty iterable → never settles (same as native)
    Array.from(iterable).forEach((item) => Promise.resolve(item).then(resolve, reject));
  });
}

export function promiseAny(iterable) {
  return new Promise((resolve, reject) => {
    const items = Array.from(iterable);
    const errors = new Array(items.length);
    let remaining = items.length;
    if (remaining === 0) {
      return reject(new AggregateError([], 'All promises were rejected'));
    }

    items.forEach((item, i) => {
      Promise.resolve(item).then(resolve, (err) => {
        errors[i] = err;
        if (--remaining === 0) {
          reject(new AggregateError(errors, 'All promises were rejected'));
        }
      });
    });
  });
}
```

### Usage

```js
const delay = (v, ms, fail = false) =>
  new Promise((res, rej) => setTimeout(() => (fail ? rej(v) : res(v)), ms));

await promiseAll([delay('a', 30), delay('b', 10), 'plain']);
// → ['a', 'b', 'plain']            (input order)

await promiseAllSettled([delay('ok', 5), delay('bad', 5, true)]);
// → [{ status: 'fulfilled', value: 'ok' }, { status: 'rejected', reason: 'bad' }]

await promiseRace([delay('slow', 50), delay('fast', 10)]);
// → 'fast'   (first to settle, win OR lose)

await promiseAny([delay('e1', 10, true), delay('ok', 20)]);
// → 'ok'     (first FULFILLED; rejections ignored)

await promiseAny([delay('e1', 10, true), delay('e2', 20, true)]).catch((e) => e.errors);
// → ['e1', 'e2']   (AggregateError, errors in input order)
```

### Gotchas / interview follow-ups

- **Result order = input order** for `all`/`allSettled`/`any`'s `errors` — write by index, never `push`.
- **Wrap with `Promise.resolve(item)`** so plain values and thenables work (`promiseAll([1, Promise.resolve(2)])`).
- **Use a counter, not `results.length`** — assigning `results[2]` first makes `length` 3 even though slots 0–1 are empty.
- **Empty iterable:** `all` → `[]`, `allSettled` → `[]`, `any` → rejects with an empty `AggregateError`, `race` → pending forever.
- **`all` does not cancel** the other promises when one rejects — they keep running; their results are just ignored. Use `AbortController` for real cancellation.
- **Unhandled rejections:** after `reject` is called, later rejections are swallowed because each promise still has a handler attached (`.then(…, reject)`).
- **Non-iterable input** (`promiseAll(5)`) throws *inside* the executor, so the returned promise rejects with `TypeError` — same as native.
- **Pick the right one:** `all` = all-or-nothing, `allSettled` = report everything (batch uploads), `race` = timeouts (`race([fetch, timeout])`), `any` = first success (mirror servers).
- **Concurrency limit** (`pLimit`, "run at most 3 at a time") is the common follow-up: keep an index pointer and spawn N worker loops that each pull the next task.

---

## 4. Function.prototype.call / apply / bind

All three set `this`. `call` invokes now with an argument list, `apply` invokes now with an array, `bind` returns a **new** function with `this` (and optionally leading args) fixed.

### Code

```js
// fnMethods.js
const define = (name, impl) =>
  Object.defineProperty(Function.prototype, name, {
    value: impl,
    writable: true,
    configurable: true, // non-enumerable: keeps for...in clean
  });

define('myCall', function (context, ...args) {
  if (typeof this !== 'function') throw new TypeError('myCall must be called on a function');
  const ctx = context == null ? globalThis : Object(context); // box primitives like sloppy mode
  const key = Symbol('fn'); // unique key: can never clobber a real property
  Object.defineProperty(ctx, key, { value: this, configurable: true });
  try {
    return ctx[key](...args); // method call → this === ctx
  } finally {
    delete ctx[key];
  }
});

define('myApply', function (context, args) {
  if (args != null && typeof args !== 'object') {
    throw new TypeError('CreateListFromArrayLike called on non-object');
  }
  return this.myCall(context, ...(args ?? []));
});

define('myBind', function (context, ...boundArgs) {
  const target = this;
  if (typeof target !== 'function') throw new TypeError('myBind must be called on a function');

  function bound(...args) {
    // `new bound()` ignores the bound context and uses the freshly created object
    const self = new.target ? this : context;
    return target.apply(self, [...boundArgs, ...args]);
  }
  if (target.prototype) bound.prototype = Object.create(target.prototype);
  return bound;
});
```

### Usage

```js
function greet(greeting, punct) {
  return `${greeting}, ${this.name}${punct}`;
}
const user = { name: 'Asha' };

greet.myCall(user, 'Hi', '!');       // 'Hi, Asha!'
greet.myApply(user, ['Hello', '?']); // 'Hello, Asha?'

const hiAsha = greet.myBind(user, 'Hey'); // partial application
hiAsha('...');                            // 'Hey, Asha...'

// borrowing methods
Math.max.myApply(null, [3, 9, 4]);                       // 9
Array.prototype.slice.myCall({ 0: 'a', 1: 'b', length: 2 }); // ['a', 'b']

// new + bind
function Point(x, y) { this.x = x; this.y = y; }
const Point10 = Point.myBind(null, 10);
const p = new Point10(5);
p.x; p.y; p instanceof Point; // 10 5 true
```

### Gotchas / interview follow-ups

- **Core trick:** attach the function to the context under a unique key, call it as a *method* (so `this` = context), then delete the key. `Symbol()` avoids collisions with real properties.
- **`null`/`undefined` context** → `globalThis` in sloppy mode (the polyfill models that); in strict-mode functions the native keeps `null`. Say which one you implemented.
- **Primitives are boxed:** `fn.call(5)` gives `this` a `Number` object in sloppy mode → `Object(context)`.
- **Frozen / non-extensible context** makes `defineProperty` throw. The fully spec-compliant way needs `Reflect.apply(fn, ctx, args)` — which makes the polyfill pointless, so mention it but don't hide behind it.
- **`bind` rules:** (1) bound args are prepended; (2) calling with `new` ignores the bound `this`; (3) bind-twice cannot change `this` — the first bind wins; (4) bound functions have no own `prototype` natively.
- **Arrow functions ignore** `call/apply/bind` for `this` (lexical) — but bound *arguments* still work.
- **`bound.length` / `bound.name`:** native sets `length = max(0, target.length - boundArgs.length)` and `name = 'bound ' + target.name`. Not needed in an interview; mention if asked about fidelity.
- **Typical use:** `Function.prototype.call.bind(Array.prototype.slice)` — uncurrying `this`.
- **`apply` with `arguments`/array-likes** works because we only spread the array-like; for huge arrays spread can hit the engine's argument limit.

---

## 5. Event Emitter

Publish/subscribe: `on(event, fn)`, `off(event, fn)`, `once(event, fn)`, `emit(event, ...args)`. The core of Node's `EventEmitter`, DOM-like buses and state stores.

### Code

```js
// eventEmitter.js
export class EventEmitter {
  #events = new Map(); // event name → array of listeners

  on(event, listener) {
    if (typeof listener !== 'function') throw new TypeError('listener must be a function');
    if (!this.#events.has(event)) this.#events.set(event, []);
    this.#events.get(event).push(listener);
    return () => this.off(event, listener); // unsubscribe handle
  }

  off(event, listener) {
    const list = this.#events.get(event);
    if (!list) return this;
    // `once` wrappers keep a reference to the original so off(original) works
    const i = list.findIndex((l) => l === listener || l.original === listener);
    if (i !== -1) list.splice(i, 1);
    if (list.length === 0) this.#events.delete(event); // no leaked empty arrays
    return this;
  }

  once(event, listener) {
    const wrapper = (...args) => {
      this.off(event, wrapper); // remove first → safe even if listener throws or re-emits
      return listener.apply(this, args);
    };
    wrapper.original = listener;
    return this.on(event, wrapper);
  }

  emit(event, ...args) {
    const list = this.#events.get(event);
    if (!list) return false;
    [...list].forEach((l) => l.apply(this, args)); // iterate a copy
    return true;
  }

  listenerCount(event) {
    return this.#events.get(event)?.length ?? 0;
  }

  removeAllListeners(event) {
    if (event === undefined) this.#events.clear();
    else this.#events.delete(event);
    return this;
  }
}
```

### Usage

```js
const bus = new EventEmitter();

const onLogin = (user) => console.log('welcome', user);
const unsubscribe = bus.on('login', onLogin);
bus.once('login', () => console.log('first login only'));

bus.emit('login', 'asha'); // welcome asha, first login only   → true
bus.emit('login', 'ravi'); // welcome ravi                      → true
unsubscribe();             // same as bus.off('login', onLogin)
bus.emit('login', 'x');    // nothing                            → false

// React-style subscription with cleanup
useEffect(() => bus.on('cart:update', setCart), []); // returned fn = cleanup
```

### Gotchas / interview follow-ups

- **Iterate over a copy** in `emit`. If a listener calls `off`/`once` during emit, mutating the live array would skip the next listener.
- **`once` needs a wrapper** — and the wrapper must be removable via the *original* function (`l.original === listener`).
- **Remove before invoking** in `once`, otherwise a re-entrant `emit` inside the listener runs it twice.
- **Return an unsubscribe function** from `on` — easiest API for React effects and avoids the "anonymous arrow can't be removed" trap.
- **Duplicates:** the same function can be added twice; `off` removes one registration (Node removes the most recently added). Decide and state it.
- **Memory leaks:** forgotten listeners keep closures alive. Node warns above 10 listeners per event (`maxListeners`); clean up on unmount.
- **Errors in listeners:** here one throwing listener stops the rest. Node's behaviour is the same (sync); a more forgiving bus wraps each call in try/catch and reports the error.
- **Sync vs async emit:** sync like Node is predictable; queueing with `queueMicrotask` avoids re-entrancy but reorders logs.
- **Extensions:** wildcard events (`user:*`), `prependListener`, async `emitAsync` (`await Promise.all(listeners.map(...))`), typed events in TypeScript.

---

## 6. Deep Clone

Copy a value so **nothing** is shared with the original. Must handle nested objects/arrays, `Date`, `Map`, `Set`, `RegExp`, symbol keys and — crucially — **circular references**.

### Code

```js
// deepClone.js
export function deepClone(value, seen = new WeakMap()) {
  if (value === null || typeof value !== 'object') return value; // primitives + functions as-is

  if (seen.has(value)) return seen.get(value); // circular / shared reference

  if (value instanceof Date) return new Date(value.getTime());
  if (value instanceof RegExp) {
    const re = new RegExp(value.source, value.flags);
    re.lastIndex = value.lastIndex;
    return re;
  }
  if (value instanceof Map) {
    const copy = new Map();
    seen.set(value, copy); // register BEFORE recursing
    value.forEach((v, k) => copy.set(deepClone(k, seen), deepClone(v, seen)));
    return copy;
  }
  if (value instanceof Set) {
    const copy = new Set();
    seen.set(value, copy);
    value.forEach((v) => copy.add(deepClone(v, seen)));
    return copy;
  }

  const copy = Array.isArray(value) ? new Array(value.length) : Object.create(Object.getPrototypeOf(value));
  seen.set(value, copy);

  for (const key of Reflect.ownKeys(value)) { // strings AND symbols
    if (Object.prototype.propertyIsEnumerable.call(value, key)) {
      copy[key] = deepClone(value[key], seen);
    }
  }
  return copy;
}
```

### Usage

```js
const original = {
  name: 'cfg',
  created: new Date('2024-01-01'),
  tags: new Set(['a', 'b']),
  meta: new Map([['k', { deep: true }]]),
  list: [1, [2, 3]],
  re: /ab+/gi,
  [Symbol('id')]: 42,
};
original.self = original; // circular

const copy = deepClone(original);

copy !== original;                 // true
copy.self === copy;                // true   (cycle preserved, points to the CLONE)
copy.created instanceof Date;      // true
copy.created !== original.created; // true
copy.meta.get('k') !== original.meta.get('k'); // true (deep)
copy.list[1] !== original.list[1]; // true
```

### Gotchas / interview follow-ups

- **`JSON.parse(JSON.stringify(x))` pitfalls:** drops `undefined`, functions and symbols; turns `Date` into a string, `Map`/`Set` into `{}`, `NaN`/`Infinity` into `null`; throws on cycles and `BigInt`.
- **Circular refs:** keep a `WeakMap(original → clone)` and **register the clone before recursing** — otherwise infinite recursion. The same map also preserves *shared* references (two properties pointing to one object stay shared in the clone).
- **`structuredClone(value)`** is built in (Node 17+, modern browsers) and handles Date/Map/Set/RegExp/cycles/typed arrays — but throws on functions, DOM nodes, and drops prototypes/getters. In production prefer it; in the interview write the manual one.
- **Prototypes:** `Object.create(Object.getPrototypeOf(value))` keeps class instances (methods still work). `structuredClone` would give plain objects.
- **Not handled here:** typed arrays/`ArrayBuffer`, `Error`, `Blob`/`File`, non-enumerable props, getters/setters (we read the value, losing the accessor), `WeakMap`/`WeakSet` (can't be cloned). Mention them to show awareness.
- **Functions are copied by reference** — closures can't be cloned.
- **Recursion depth:** very deep structures can overflow the stack; an iterative version with an explicit stack avoids that.
- **Shallow alternatives:** spread `{...obj}`, `Object.assign`, `Array.from` — one level only. Nested objects stay shared.

---

## 7. Memoize

Cache a function's results keyed by its arguments so repeated calls with the same input skip the work. Only valid for **pure** functions.

### Code

```js
// memoize.js
export function memoize(fn, resolver = (...args) => JSON.stringify(args)) {
  const cache = new Map();

  function memoized(...args) {
    const key = resolver(...args);
    if (cache.has(key)) return cache.get(key); // has(), not truthiness: cached 0/''/undefined are valid
    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  }

  memoized.cache = cache;
  memoized.clear = () => cache.clear();
  return memoized;
}
```

### Usage

```js
let calls = 0;
const slowSquare = memoize((n) => {
  calls++;
  return n * n;
});
slowSquare(4); // 16   (computed)
slowSquare(4); // 16   (cache hit)
calls;         // 1

// recursion: the *memoized* name must be used inside, otherwise inner calls skip the cache
const fib = memoize((n) => (n < 2 ? n : fib(n - 1) + fib(n - 2)));
fib(50); // 12586269025 — instant; the naive version takes minutes

// custom key resolver for object arguments
const getUser = memoize(fetchUser, (user) => user.id);

// memoizing a promise-returning function caches the PROMISE (dedupes in-flight requests)
const loadConfig = memoize(() => fetch('/config').then((r) => r.json()), () => 'cfg');
```

### Gotchas / interview follow-ups

- **Cache key is the hard part.** `JSON.stringify(args)` is the common default but: `undefined`, `null` and `NaN` all become `null` (collisions), key order in objects matters (`{a:1,b:2}` ≠ `{b:2,a:1}`), functions/symbols vanish, cycles throw. Accept a `resolver` for anything non-trivial.
- **Single primitive arg fast path:** key the `Map` with the argument itself (`Map` keys keep type, so `1` and `'1'` differ).
- **Object-identity keys:** for object args use a `WeakMap` so entries are garbage-collected with the argument.
- **Memory growth:** an unbounded cache is a leak. Add a max size (LRU: delete the oldest key from the `Map` — `Map` preserves insertion order) or a TTL.
- **`has()` vs `get()`:** checking `cache.get(key) !== undefined` re-computes functions that legitimately return `undefined`/`0`/`false`.
- **Only pure functions.** Memoizing something that reads time, randomness, or global state returns stale data.
- **Async:** a rejected promise stays in the cache forever — evict it in `.catch` if failures should be retried.
- **`this`** is forwarded, but the default key ignores it — two receivers with the same args share a cache entry.
- **React tie-in:** `useMemo`/`useCallback`/`React.memo` are memoization with a cache size of **one** (previous deps), which is why they are cheap.

---

## 8. Curry

Transform `f(a, b, c)` into `f(a)(b)(c)` — each call takes **some** arguments and returns a function waiting for the rest.

### Code

```js
// curry.js
export function curry(fn) {
  return function curried(...args) {
    if (args.length >= fn.length) return fn.apply(this, args); // enough args → run
    return (...next) => curried.apply(this, [...args, ...next]); // else keep collecting
  };
}

// Variant: sum(1)(2)(3)() — arity unknown, an empty call ends the chain
export function curryUntilEmpty(fn) {
  return function collect(...args) {
    return function next(...more) {
      if (more.length === 0) return fn(...args);
      return collect(...args, ...more);
    };
  };
}

// Variant: placeholders — curryPlaceholder(fn)(_, 2)(1)
export const _ = Symbol('placeholder');
export function curryPlaceholder(fn) {
  return function curried(...args) {
    const ready = args.length >= fn.length && args.slice(0, fn.length).every((a) => a !== _);
    if (ready) return fn.apply(this, args);
    return (...next) => {
      const merged = args.map((a) => (a === _ && next.length ? next.shift() : a));
      return curried.apply(this, [...merged, ...next]);
    };
  };
}
```

### Usage

```js
const add = (a, b, c) => a + b + c;
const curried = curry(add);

curried(1)(2)(3);   // 6
curried(1, 2)(3);   // 6
curried(1)(2, 3);   // 6
curried(1, 2, 3);   // 6

// reusable specialisations
const log = curry((level, ts, msg) => `[${level}] ${ts}: ${msg}`);
const error = log('ERROR');
error('12:00')('disk full'); // '[ERROR] 12:00: disk full'

// unknown arity
const sum = curryUntilEmpty((...nums) => nums.reduce((a, b) => a + b, 0));
sum(1)(2, 3)(4)(); // 10

// placeholders
const sub = curryPlaceholder((a, b, c) => a - b - c);
sub(_, 2, 1)(10); // 7
```

### Gotchas / interview follow-ups

- **`fn.length` is the arity** — it counts parameters *before* the first default or rest parameter. `(a, b = 2, c) => …` has length `1`; `(...args) => …` has length `0` (use `curryUntilEmpty` or pass the arity explicitly: `curry(fn, arity = fn.length)`).
- **Currying vs partial application:** currying always yields unary-ish steps until arity is reached; `bind`/partial fixes *some* args once and runs on the next call.
- **Keep `this`** by using `fn.apply(this, …)` and arrow functions that capture it.
- **Extra arguments** are passed through (`args.length >= fn.length`), matching Lodash's `_.curry`.
- **Why interviewers like it:** tests closures (accumulating args), recursion and `Function.length`.
- **Real uses:** pre-configuring event handlers/loggers, functional pipelines (`map(curry(add)(1))`), Redux-style `connect(mapState)(Component)`.
- **Each call returns a new function** — don't expect `curried(1) === curried(1)`.

---

## 9. Flatten

Turn arbitrarily nested arrays into a single level (with an optional depth) — plus the object variant (`{a:{b:1}}` → `{'a.b':1}`).

### Code

```js
// flatten.js
export function flatten(arr, depth = 1) {
  const out = [];
  (function walk(items, d) {
    items.forEach((item) => { // forEach skips holes, like native .flat()
      if (Array.isArray(item) && d > 0) walk(item, d - 1);
      else out.push(item);
    });
  })(arr, depth);
  return out;
}

// iterative, unlimited depth — no recursion limit
export function flattenDeepIterative(arr) {
  const stack = [...arr];
  const out = [];
  while (stack.length) {
    const next = stack.pop();
    if (Array.isArray(next)) stack.push(...next);
    else out.push(next);
  }
  return out.reverse(); // we consumed from the back
}

// reduce version (the one-liner people expect)
export const flattenReduce = (arr) =>
  arr.reduce((acc, x) => acc.concat(Array.isArray(x) ? flattenReduce(x) : x), []);

export function flattenObject(obj, prefix = '', out = {}) {
  for (const [key, value] of Object.entries(obj)) {
    const path = prefix ? `${prefix}.${key}` : key;
    if (value !== null && typeof value === 'object' && !(value instanceof Date)) {
      flattenObject(value, path, out);
    } else {
      out[path] = value;
    }
  }
  return out;
}
```

### Usage

```js
const nested = [1, [2, [3, [4]]]];

flatten(nested);           // [1, 2, [3, [4]]]   depth 1 (same default as Array#flat)
flatten(nested, 2);        // [1, 2, 3, [4]]
flatten(nested, Infinity); // [1, 2, 3, 4]
flattenDeepIterative(nested); // [1, 2, 3, 4]
flattenReduce(nested);        // [1, 2, 3, 4]

flattenObject({ a: 1, b: { c: 2, d: { e: 3 } }, list: [10, 20] });
// { a: 1, 'b.c': 2, 'b.d.e': 3, 'list.0': 10, 'list.1': 20 }
```

### Gotchas / interview follow-ups

- **Native:** `arr.flat(depth)` / `arr.flat(Infinity)`; `flatMap(fn)` = map + flat(1). Interviewers still ask for the manual version.
- **Depth parameter:** decrement on each recursive step; `Infinity - 1` stays `Infinity`, so it works for "flatten all".
- **Order in the iterative version:** pop from the end and push children, so the output is built reversed — hence `.reverse()` (or use `shift()` for O(n²) simplicity).
- **Holes:** native `.flat()` removes holes; `forEach` does too. `for...of` / spread would turn them into `undefined`.
- **Recursion depth:** adversarial deeply-nested input overflows the call stack → iterative version.
- **`concat` in `reduce` is O(n²)** (allocates a new array every step). Use `push(...)` into one accumulator when performance matters.
- **`stack.push(...next)`** can hit the argument-count limit for enormous arrays — loop and push individually if needed.
- **`Array.isArray`**, not `instanceof Array` (cross-realm / iframes) and not `typeof` (`'object'` for everything).
- **Object flatten edge cases:** empty objects disappear, keys containing `.` are ambiguous, `Date`/`Map`/class instances need a decision — state your assumptions. The reverse (`unflatten`) is a common follow-up.

---

## 10. Pipe / Compose

Combine small functions into one. `pipe` runs **left → right**, `compose` runs **right → left**.

### Code

```js
// pipeCompose.js
export const pipe = (...fns) => (...args) => {
  if (fns.length === 0) return args[0]; // identity
  const [first, ...rest] = fns;
  return rest.reduce((acc, fn) => fn(acc), first(...args)); // first fn may take many args
};

export const compose = (...fns) => pipe(...[...fns].reverse());

// supports sync AND async steps; always returns a promise
export const pipeAsync = (...fns) => async (...args) => {
  if (fns.length === 0) return args[0];
  const [first, ...rest] = fns;
  let acc = await first(...args);
  for (const fn of rest) acc = await fn(acc);
  return acc;
};
```

### Usage

```js
const add1 = (x) => x + 1;
const double = (x) => x * 2;
const square = (x) => x * x;

pipe(add1, double, square)(3);    // ((3+1)*2)^2 = 64
compose(add1, double, square)(3); // add1(double(square(3))) = 19
pipe()(7);                        // 7  (identity)

// first function can take multiple arguments
const sumThenDouble = pipe((a, b) => a + b, double);
sumThenDouble(2, 3); // 10

// real-world: transform pipeline
const slugify = pipe(
  (s) => s.trim(),
  (s) => s.toLowerCase(),
  (s) => s.replace(/\s+/g, '-'),
);
slugify('  Hello World  '); // 'hello-world'

await pipeAsync(fetchUser, (u) => u.id, fetchOrders)(42);
```

### Gotchas / interview follow-ups

- **Same thing, opposite order:** `compose(f, g, h)(x) = f(g(h(x)))`, `pipe(f, g, h)(x) = h(g(f(x)))`. Implement `compose` via `reduceRight`, or by reversing and delegating to `pipe`.
- **Only the first function may be variadic.** Every later function receives exactly one value (the previous result) — that's why curry and pipe go together.
- **Empty pipeline → identity.** Don't throw; `reduce` with no initial value on an empty array does.
- **Async steps:** a sync `pipe` passes a *promise* to the next function. Either `await` between steps (`pipeAsync`) or chain with `.then`.
- **Error handling:** an exception short-circuits the whole chain; wrap steps (Either/Result) if you need per-step recovery.
- **Debugging tip:** insert a `tap(fn)` step (`x => { fn(x); return x; }`) to log intermediate values.
- **Where it shows up:** Redux middleware (`applyMiddleware` uses `compose`), Express-style middleware chains, Lodash `flow`/`flowRight`, RxJS `.pipe()`.

---

## 11. Array.prototype.map / filter / reduce

The three workhorses. Each takes a callback invoked with `(element, index, array)` and an optional `thisArg` (`map`/`filter`); each **skips holes** in sparse arrays.

### Code

```js
// arrayMethods.js
const define = (name, impl) =>
  Object.defineProperty(Array.prototype, name, {
    value: impl,
    writable: true,
    configurable: true, // non-enumerable → doesn't leak into for...in
  });

function assertCallable(fn) {
  if (typeof fn !== 'function') throw new TypeError(`${String(fn)} is not a function`);
}

define('myMap', function (callback, thisArg) {
  assertCallable(callback);
  const O = Object(this);
  const len = O.length >>> 0;
  const result = new Array(len); // holes stay holes
  for (let i = 0; i < len; i++) {
    if (i in O) result[i] = callback.call(thisArg, O[i], i, O);
  }
  return result;
});

define('myFilter', function (callback, thisArg) {
  assertCallable(callback);
  const O = Object(this);
  const len = O.length >>> 0;
  const result = [];
  for (let i = 0; i < len; i++) {
    if (i in O && callback.call(thisArg, O[i], i, O)) result.push(O[i]);
  }
  return result;
});

define('myReduce', function (callback, ...rest) {
  assertCallable(callback);
  const O = Object(this);
  const len = O.length >>> 0;
  let i = 0;
  let acc;

  if (rest.length > 0) {
    acc = rest[0]; // initial value supplied (even if undefined)
  } else {
    while (i < len && !(i in O)) i++; // first non-hole element is the seed
    if (i >= len) throw new TypeError('Reduce of empty array with no initial value');
    acc = O[i++];
  }

  for (; i < len; i++) {
    if (i in O) acc = callback(acc, O[i], i, O);
  }
  return acc;
});
```

### Usage

```js
[1, 2, 3].myMap((x) => x * 2);                 // [2, 4, 6]
[1, 2, 3].myMap(function (x) { return x * this.k; }, { k: 10 }); // [10, 20, 30]

[1, 2, 3, 4].myFilter((x) => x % 2 === 0);     // [2, 4]

[1, 2, 3, 4].myReduce((sum, x) => sum + x, 0); // 10
[1, 2, 3, 4].myReduce((max, x) => (x > max ? x : max)); // 4  (seeded by first element)

// group by with reduce
['apple', 'avocado', 'banana'].myReduce((acc, w) => {
  (acc[w[0]] ??= []).push(w);
  return acc;
}, {});
// { a: ['apple', 'avocado'], b: ['banana'] }

// map can be built from reduce
const mapViaReduce = (arr, fn) => arr.myReduce((acc, x, i) => (acc.push(fn(x, i, arr)), acc), []);
```

### Gotchas / interview follow-ups

- **Holes:** `[1, , 3].map(f)` calls `f` twice and keeps the hole; that is why the polyfill uses `i in O` instead of `O[i] !== undefined`.
- **`reduce` without an initial value** seeds with the first element and starts at index 1; on an **empty** array it throws `TypeError`. Detect "initial value provided" with `arguments.length`/rest, **not** `initialValue === undefined`.
- **`thisArg`** applies to `map`/`filter` (not `reduce`) and is ignored for arrow functions.
- **Callbacks get 3 arguments** — the classic trap `['1','2','3'].map(parseInt)` → `[1, NaN, NaN]` because `parseInt(str, radix=index)`.
- **Length is read once.** Elements appended during iteration are not visited; elements deleted are skipped by the `in` check.
- **Non-mutating:** `map`/`filter`/`reduce` return new values (but the callback can still mutate the objects inside).
- **Don't add enumerable methods** to built-in prototypes (`Array.prototype.myMap = …` shows up in `for...in`) — use `defineProperty` (non-enumerable) — and in real code, don't patch prototypes at all (polyfill only when `if (!Array.prototype.x)`).
- **`filter` result is sparse-free** (`push`), `map` result keeps holes (`new Array(len)`).
- **Performance:** a hand-written `for` loop is faster than `reduce` with spreading (`[...acc, x]` is O(n²)).

---

## 12. Once

Wrap a function so it runs **only the first time**; later calls return the first result without invoking it again.

### Code

```js
// once.js
export function once(fn) {
  let called = false;
  let result;

  return function (...args) {
    if (!called) {
      called = true; // set BEFORE calling → re-entrant calls can't run it twice
      result = fn.apply(this, args);
      fn = null; // release the closure so it can be GC'd
    }
    return result;
  };
}
```

### Usage

```js
const init = once(() => {
  console.log('connecting…');
  return { connected: true };
});

const a = init(); // logs 'connecting…'
const b = init(); // nothing logged
a === b;          // true — same cached result object

// lazy singleton
const getDb = once(() => new Database(process.env.DB_URL));

// event listener that must fire once (DOM has { once: true } natively)
button.addEventListener('click', once(handleFirstClick));
```

### Gotchas / interview follow-ups

- **State in the closure:** `called` + `result`. Using the *result* as the flag (`if (result === undefined)`) breaks when the function legitimately returns `undefined`.
- **Arguments of later calls are ignored** — `init(5)` then `init(10)` returns the result of `5`.
- **`this` is forwarded** (`fn.apply(this, args)`), so it works as a method — but only the *first* receiver counts.
- **If `fn` throws:** with the flag set first, later calls return `undefined` (lodash behaves the same). If you want "retry until success", set the flag only after `fn` returns.
- **Async functions:** `once(async () => …)` caches the **promise**, so concurrent callers share one in-flight request — usually exactly what you want for init/auth-refresh.
- **Generalisations:** `before(n, fn)` (first n calls), `after(n, fn)`, `limit(n)`; a `reset()` method for tests.
- **Native equivalents:** `addEventListener(…, { once: true })`, Node's `emitter.once`.

---

## 13. Sleep + Retry with Exponential Backoff

`sleep(ms)` turns `setTimeout` into something you can `await`. `retry` re-runs a failing async operation, waiting longer after each failure (`base · factor^attempt`) so a struggling server isn't hammered.

### Code

```js
// retry.js
export const sleep = (ms, { signal } = {}) =>
  new Promise((resolve, reject) => {
    if (signal?.aborted) return reject(signal.reason);
    const id = setTimeout(() => {
      signal?.removeEventListener('abort', onAbort);
      resolve();
    }, ms);
    function onAbort() {
      clearTimeout(id);
      reject(signal.reason);
    }
    signal?.addEventListener('abort', onAbort, { once: true });
  });

export async function retry(
  fn,
  {
    retries = 3, // retries AFTER the first attempt → up to retries + 1 calls
    baseDelay = 100,
    factor = 2,
    maxDelay = 10_000,
    jitter = true,
    shouldRetry = () => true,
    signal,
  } = {},
) {
  for (let attempt = 0; ; attempt++) {
    try {
      return await fn(attempt);
    } catch (err) {
      if (attempt >= retries || !shouldRetry(err)) throw err;
      const backoff = Math.min(maxDelay, baseDelay * factor ** attempt);
      const delay = jitter ? Math.random() * backoff : backoff; // "full jitter"
      await sleep(delay, { signal });
    }
  }
}
```

### Usage

```js
await sleep(500); // pause an async function for 500 ms

// delays (no jitter): 100 → 200 → 400 → give up
const data = await retry(() => fetchJson('/api/orders'), { retries: 3, baseDelay: 100, jitter: false });

// only retry transient failures, never 4xx
await retry(
  async () => {
    const res = await fetch('/api/pay', { method: 'POST' });
    if (!res.ok) throw Object.assign(new Error('HTTP ' + res.status), { status: res.status });
    return res.json();
  },
  { shouldRetry: (e) => e.status >= 500 || e.name === 'TypeError' },
);

// cancelable
const ctrl = new AbortController();
retry(job, { signal: ctrl.signal }).catch(() => {});
ctrl.abort(); // stops waiting and rejects

// timeout for any promise: race with sleep
await Promise.race([fetchJson('/slow'), sleep(3000).then(() => { throw new Error('timeout'); })]);
```

### Gotchas / interview follow-ups

- **Exponential formula:** `delay = min(maxDelay, base * 2 ** attempt)` → 100, 200, 400, 800 … Always **cap it**.
- **Jitter** (randomising the delay) stops thousands of clients retrying in lock-step (the "thundering herd"). *Full jitter* = `random(0, backoff)`; AWS recommends it.
- **Retry count semantics:** `retries: 3` means 1 initial try + 3 retries = 4 calls. Say it out loud — off-by-one is the usual bug.
- **Rethrow the last error** (not a generic one) so callers see the real cause; no `sleep` after the final failure.
- **Only retry retryable errors** — network failures, 5xx, 429 (honour `Retry-After`). Never blindly retry 4xx validation errors or non-idempotent `POST`s (double charges) without an idempotency key.
- **`await sleep` isn't blocking** — the event loop keeps running; `setTimeout` is only a *minimum* delay.
- **Cancellation:** clear the timer and reject on `AbortSignal`; remove the abort listener when the timer fires to avoid leaks.
- **`for (;;)` + `await` in a loop** is intentional here — attempts must be sequential. (In Q&A: contrast with `Promise.all`, which would run them in parallel.)
- **Testing:** inject `fn`/delays or use fake timers (`vi.advanceTimersByTimeAsync`) rather than waiting real seconds.
- **Related patterns:** circuit breaker, request timeouts, polling with backoff, React Query's `retry`/`retryDelay` options.

---

## 14. Tricky Output Questions

> Q1–Q13 use the implementations above (`debounce`, `throttle`, `promiseAll`, `myBind`, …). Q14 onwards are **self-contained** — copy a snippet into a file and run it with Node 18+ (as an ES module, so top-level code is in strict mode). Every output below was produced by actually running the code in Node.
>
> Topics: event loop (Q14–Q22) · promises & async/await (Q23–Q33) · `this`, `call`/`apply`/`bind` (Q34–Q42) · closures, memoize, curry, once, compose, array methods (Q43–Q54) · cloning, equality, coercion (Q55–Q60, Q71–Q77) · event emitter, debounce, throttle, timeout, retry (Q61–Q68) · prototypes and polyfill hygiene (Q69–Q70)

### Q1. Debounce timing

```js
const d = debounce((x) => console.log('fired', x), 100);

d(1);
d(2);
setTimeout(() => d(3), 50);
setTimeout(() => d(4), 200);
```

<details>
<summary><strong>Show Output</strong></summary>

```js
fired 3
fired 4
```

`d(1)` and `d(2)` are in the same burst — only the latest survives and its timer is reset by `d(3)` at t=50, so it fires at t=150 with `3`. `1` and `2` are dropped. `d(4)` arrives after the quiet period ended, so it starts a new burst and fires at t=300.

</details>

### Q2. Throttle: leading + trailing

```js
const t = throttle((x) => console.log('run', x), 100);

t('a');
t('b');
t('c');
setTimeout(() => t('d'), 150);
```

<details>
<summary><strong>Show Output</strong></summary>

```js
run a
run c
run d
```

`a` runs immediately (leading edge). `b` and `c` land inside the window — only the latest (`c`) is kept and runs on the trailing edge at t≈100, which also restarts the window. `d` arrives at t=150, still inside that new window, so it is scheduled and runs at t≈200. `b` is lost.

</details>

### Q3. Settling order of the Promise combinators

```js
const delay = (v, ms, fail = false) =>
  new Promise((res, rej) => setTimeout(() => (fail ? rej(v) : res(v)), ms));

promiseAll([delay('slow', 30), delay('fast', 10), 'plain']).then(console.log);
promiseAny([delay('e1', 10, true), delay('ok', 20)]).then(console.log);
promiseRace([delay('r1', 20), delay('r2', 10, true)]).catch((e) => console.log('race rejected:', e));
promiseAllSettled([delay('x', 5), delay('y', 5, true)]).then((r) => console.log(r.map((o) => o.status)));
```

<details>
<summary><strong>Show Output</strong></summary>

```js
[ 'fulfilled', 'rejected' ]
race rejected: r2
ok
[ 'slow', 'fast', 'plain' ]
```

Logs appear in the order the promises **settle**: `allSettled` at 5 ms, `race` at 10 ms (the first to settle wins — even if it is a rejection), `any` at 20 ms (skips the rejection and takes the first fulfilment), `all` at 30 ms (waits for the slowest). `all`'s array is still in **input** order, not completion order.

</details>

### Q4. bind, call and `new`

```js
function who() { return this.name; }

const a = who.myBind({ name: 'A' });
const b = a.myBind({ name: 'B' });
console.log(a(), b());
console.log(a.myCall({ name: 'C' }));
console.log({ name: 'O', get: a }.get());

function Pair(x, y) { this.x = x; this.y = y; }
const P1 = Pair.myBind({ ignored: true }, 1);
const p = new P1(2);
console.log(p.x, p.y, p instanceof Pair);
```

<details>
<summary><strong>Show Output</strong></summary>

```js
A A
A
A
1 2 true
```

A bound function's `this` is fixed forever: binding again (`b`), `call`, and even calling it as a method of another object cannot change it — the first bind wins. With `new`, the bound `this` is ignored (a fresh object is created) but the pre-filled argument `1` is still prepended, giving `x = 1, y = 2`, and the instance still passes `instanceof Pair`.

</details>

### Q5. Memoize with the default key

```js
let calls = 0;
const m = memoize((x) => {
  calls++;
  return x;
});

m(1);
m('1');
m(undefined);
console.log(calls);
console.log(m(null), m(NaN));
console.log(calls);
```

<details>
<summary><strong>Show Output</strong></summary>

```js
3
undefined undefined
3
```

`JSON.stringify` makes the keys `[1]`, `["1"]` and `[null]` for the first three calls, so `calls` is 3. Then `null` and `NaN` also stringify to `[null]` — a **cache collision** with `undefined` — so they return the cached `undefined` instead of `null` / `NaN` and the function is not called again. Fix: pass a custom `resolver`, or key a `Map` by the argument itself.

</details>

### Q6. `curry` and `fn.length`

```js
const f = curry((a, b = 2, c = 3) => [a, b, c]);
console.log(f(1));

const g = curry((...args) => args.length);
console.log(g(1));

const h = curry((a, b, c) => a + b + c);
console.log(h(1)(2)(3), typeof h(1)(2));
```

<details>
<summary><strong>Show Output</strong></summary>

```js
[ 1, 2, 3 ]
1
6 function
```

`fn.length` stops counting at the first default/rest parameter: the first function has length `1`, so `f(1)` already has "enough" arguments and runs immediately. `g` has length `0`, so it runs on the very first call. Only `h` (length 3) keeps returning functions until it has three arguments.

</details>

### Q7. `deepClone` vs `JSON`

```js
const src = { d: new Date(0), u: undefined, n: NaN, s: new Set([1]), re: /x/g, fn() {} };

console.log(JSON.parse(JSON.stringify(src)));

const c = deepClone(src);
console.log(c.d instanceof Date, c.s.has(1), 'u' in c, c.fn === src.fn);

src.self = src;
const cyclic = deepClone(src);
console.log(cyclic.self === src, cyclic.self === cyclic);
```

<details>
<summary><strong>Show Output</strong></summary>

```js
{ d: '1970-01-01T00:00:00.000Z', n: null, s: {}, re: {} }
true true true true
false true
```

The JSON round trip turns the `Date` into a string, drops `u` and `fn`, turns `NaN` into `null` and flattens `Set`/`RegExp` into `{}`. `deepClone` keeps all of them (functions are copied by reference, so `c.fn === src.fn`). After adding a cycle, the clone's `self` points to the **clone** (`=== cyclic`), not the original; `JSON.stringify(src)` would now throw `TypeError: Converting circular structure to JSON`.

</details>

### Q8. Mutating listeners while emitting

```js
const e = new EventEmitter();
const b = () => console.log('b');
const a = () => {
  console.log('a');
  e.off('x', b);
};

e.on('x', a);
e.on('x', b);
e.emit('x');
e.emit('x');

e.once('y', () => console.log('once'));
console.log(e.emit('y'), e.emit('y'));
```

<details>
<summary><strong>Show Output</strong></summary>

```js
a
b
a
once
true false
```

`emit` iterates over a **copy** of the listener array, so removing `b` from inside `a` does not stop `b` from running in the *current* emit — it is gone from the second one. For `once('y')`, the first `emit` returns `true` and runs the listener (printing `once` while the arguments of `console.log` are being evaluated); the second returns `false` because the wrapper already removed itself.

</details>

### Q9. map / reduce edge cases

```js
console.log(['1', '2', '3'].myMap(parseInt));
console.log([1, , 3].myMap((x) => x * 2));

try {
  [].myReduce((a, b) => a + b);
} catch (e) {
  console.log(e.constructor.name + ': ' + e.message);
}
console.log([].myReduce((a, b) => a + b, 'init'));
```

<details>
<summary><strong>Show Output</strong></summary>

```js
[ 1, NaN, NaN ]
[ 2, <1 empty item>, 6 ]
TypeError: Reduce of empty array with no initial value
init
```

`map` passes `(value, index, array)`, so `parseInt('2', 1)` and `parseInt('3', 2)` are invalid-radix/invalid-digit calls → `NaN`. The hole in `[1, , 3]` is skipped (callback not called) and stays a hole in the result. `reduce` on an empty array only throws when there is **no initial value**; with `'init'` it simply returns it.

</details>

### Q10. Flatten depth and holes

```js
const nested = [1, [2, [3, [4]]]];

console.log(flatten(nested));
console.log(flatten(nested, Infinity));
console.log(flatten([1, , [2, , 3]]));
console.log(flattenDeepIterative(nested));
```

<details>
<summary><strong>Show Output</strong></summary>

```js
[ 1, 2, [ 3, [ 4 ] ] ]
[ 1, 2, 3, 4 ]
[ 1, 2, 3 ]
[ 1, 2, 3, 4 ]
```

The default depth is `1` (same as `Array#flat`), so only the outermost nesting level is removed. `Infinity` flattens everything. `forEach` skips holes, so they vanish exactly like native `.flat()`. The iterative version pops from the end and reverses at the end to restore the original order.

</details>

### Q11. `once` and `this`

```js
const init = once(function () {
  console.log('init', this.id);
  return this.id;
});

const a = { id: 1, init };
const b = { id: 2, init };

console.log(a.init(), b.init());
```

<details>
<summary><strong>Show Output</strong></summary>

```js
init 1
1 1
```

The first call (with `this = a`) runs and caches `1`. `b.init()` never calls the function again, so `this.id` is not read for `b` — the cached result `1` is returned. Only the first receiver counts.

</details>

### Q12. Sleep and the event loop

```js
console.log('1 sync');

sleep(0).then(() => console.log('4 sleep(0)'));
Promise.resolve().then(() => console.log('3 microtask'));

(async () => {
  await sleep(10);
  console.log('5 after sleep(10)');
})();

console.log('2 sync end');
```

<details>
<summary><strong>Show Output</strong></summary>

```js
1 sync
2 sync end
3 microtask
4 sleep(0)
5 after sleep(10)
```

Synchronous code runs first. Then the microtask queue (promise callbacks) drains. `sleep(0)` is a `setTimeout` — a macrotask — so it runs after the microtask even with a 0 ms delay, and the 10 ms sleep runs last.

</details>

### Q13. Retry: attempts and the final error

```js
const makeFlaky = () => {
  let n = 0;
  return async () => {
    n++;
    if (n < 3) throw new Error(`fail ${n}`);
    return `ok after ${n}`;
  };
};
const opts = { baseDelay: 5, jitter: false };

(async () => {
  console.log(await retry(makeFlaky(), { ...opts, retries: 3 }));
  await retry(makeFlaky(), { ...opts, retries: 1 }).catch((e) => console.log('gave up:', e.message));
})();
```

<details>
<summary><strong>Show Output</strong></summary>

```js
ok after 3
gave up: fail 2
```

With `retries: 3` the function can run up to 4 times; it succeeds on attempt 3, so the result is `ok after 3`. With `retries: 1` there is the first try plus one retry (2 calls total), both fail, and the **last** error (`fail 2`) is rethrown — no sleep happens after the final failure.

</details>

### Q14. Event loop: sync, microtasks, macrotasks

```js
console.log('1');
setTimeout(() => console.log('2'), 0);
Promise.resolve().then(() => console.log('3'));
queueMicrotask(() => console.log('4'));
(async () => {
  console.log('5');
  await null;
  console.log('6');
})();
console.log('7');
```

<details>
<summary><strong>Show Output</strong></summary>

```js
1
5
7
3
4
6
2
```

Synchronous code runs first, including the body of the `async` function up to its first `await` (so `5` prints before `7`). Then the microtask queue drains in the order jobs were queued: the `.then` (`3`), `queueMicrotask` (`4`), and the continuation after `await` (`6`). The `setTimeout` macrotask (`2`) runs last, even with a 0 ms delay.

</details>

### Q15. Microtasks queued by microtasks still run before a timer

```js
setTimeout(() => console.log('timeout'), 0);

let i = 0;
function loop() {
  if (i++ < 3) {
    Promise.resolve().then(() => {
      console.log('micro', i);
      loop();
    });
  }
}
loop();
```

<details>
<summary><strong>Show Output</strong></summary>

```js
micro 1
micro 2
micro 3
timeout
```

The microtask queue is drained **completely** — including microtasks added while draining — before the event loop moves to the next macrotask. That is why an endless microtask chain can starve rendering and timers.

</details>

### Q16. `return promise` inside `async` costs extra ticks

```js
async function a() {
  return 1;
}
async function b() {
  return Promise.resolve(2);
}

a().then(() => console.log('a'));
b().then(() => console.log('b'));
Promise.resolve()
  .then(() => console.log('p1'))
  .then(() => console.log('p2'))
  .then(() => console.log('p3'));
```

<details>
<summary><strong>Show Output</strong></summary>

```js
a
p1
p2
b
p3
```

`a()` fulfils immediately, so its callback runs in the first tick. `b()` resolves with a *promise*, which has to be adopted first (a thenable job, then a `.then` on it) — roughly two extra ticks — so `b` lands between `p2` and `p3`. `return await x` or returning a plain value avoids the delay.

</details>

### Q17. Timer ordering with different delays

```js
setTimeout(() => console.log('A'), 10);
setTimeout(() => console.log('B'), 0);
setTimeout(() => console.log('C'), 5);
setTimeout(() => console.log('D'), 0);
```

<details>
<summary><strong>Show Output</strong></summary>

```js
B
D
C
A
```

Timers fire by due time; ties keep registration order, so the two 0 ms timers run first (`B` then `D`). The registration order of `A` and `C` is irrelevant — `C` (5 ms) is due before `A` (10 ms).

</details>

### Q18. Clearing an interval from inside its own callback

```js
let n = 0;
const id = setInterval(() => {
  n++;
  console.log('tick', n);
  if (n === 3) clearInterval(id);
}, 10);
```

<details>
<summary><strong>Show Output</strong></summary>

```js
tick 1
tick 2
tick 3
```

`id` is assigned before the first tick fires, so the callback can safely close over it and stop itself. The same pattern is how you build a `setTimeout`-chain or a polling loop that stops on a condition.

</details>

### Q19. `var` vs `let` in a loop with timers

```js
for (var i = 0; i < 3; i++) setTimeout(() => console.log('var', i), 0);
for (let j = 0; j < 3; j++) setTimeout(() => console.log('let', j), 0);
```

<details>
<summary><strong>Show Output</strong></summary>

```js
var 3
var 3
var 3
let 0
let 1
let 2
```

`var i` is one binding shared by every callback; the loop has finished (`i === 3`) before any timer fires. `let j` creates a **fresh binding per iteration**, so each closure captures its own value. The pre-ES6 fix was an IIFE or `setTimeout(fn, 0, i)`.

</details>

### Q20. `this` inside a `setTimeout` callback

```js
const o = {
  n: 1,
  f() {
    setTimeout(function () {
      console.log(this === o, this.n);
    }, 0);
  },
  g() {
    setTimeout(() => console.log(this === o, this.n), 0);
  },
};
o.f();
o.g();
```

<details>
<summary><strong>Show Output</strong></summary>

```js
false undefined
true 1
```

A regular function passed to `setTimeout` is invoked by the timer machinery, not as `o.f()`, so `this` is the `Timeout` object in Node (`window` in browsers) — never `o`. The arrow function has no `this` of its own and uses the one from `g`, which is `o`. Use an arrow function or `.bind(this)`.

</details>

### Q21. A Promise executor runs synchronously

```js
console.log('start');
const p = new Promise((resolve) => {
  console.log('executor');
  resolve('done');
  console.log('after resolve');
});
p.then((v) => console.log(v));
console.log('end');
```

<details>
<summary><strong>Show Output</strong></summary>

```js
start
executor
after resolve
end
done
```

The executor is run immediately and synchronously, and code after `resolve()` still executes (`resolve` is not `return`). Only the `.then` callback is deferred to a microtask.

</details>

### Q22. Only the first settle call wins

```js
new Promise((resolve, reject) => {
  resolve('first');
  reject('second');
  resolve('third');
  throw new Error('boom');
}).then(
  (v) => console.log('ok', v),
  (e) => console.log('err', e)
);
```

<details>
<summary><strong>Show Output</strong></summary>

```js
ok first
```

A promise settles exactly once. Later `resolve`/`reject` calls and even a thrown error after settlement are ignored (the throw is swallowed, not reported).

</details>

### Q23. then / catch / finally chain

```js
Promise.resolve(1)
  .then((v) => {
    console.log(v);
    return v + 1;
  })
  .then((v) => {
    console.log(v);
    throw new Error('x');
  })
  .then(() => console.log('skipped'))
  .catch((e) => {
    console.log('caught', e.message);
    return 'recovered';
  })
  .then((v) => console.log(v))
  .finally(() => console.log('finally'));
```

<details>
<summary><strong>Show Output</strong></summary>

```js
1
2
caught x
recovered
finally
```

A thrown error turns the chain into a rejected state and skips every `.then(onFulfilled)` until a handler is found. `catch` returning a value *recovers* the chain, so the next `.then` receives `'recovered'`. `finally` runs either way.

</details>

### Q24. Non-function arguments to `then`

```js
Promise.resolve(1).then(2).then(console.log);
Promise.resolve(1).then(null, null).then(console.log);
Promise.reject('e').then(console.log).catch((e) => console.log('c', e));
```

<details>
<summary><strong>Show Output</strong></summary>

```js
1
1
c e
```

If `then` receives something that is not a function, it is ignored and the value (or rejection) passes straight through to the next link. The rejected promise skips `.then(console.log)` and reaches `catch`.

</details>

### Q25. `finally` and the final value

```js
Promise.resolve('A').finally(() => 'B').then((v) => console.log('1', v));
Promise.reject('E').finally(() => 'B').catch((e) => console.log('2', e));
Promise.resolve('A').finally(() => { throw 'F'; }).catch((e) => console.log('3', e));
```

<details>
<summary><strong>Show Output</strong></summary>

```js
3 F
1 A
2 E
```

`finally` does not change the outcome: its return value is ignored (`A` and `E` pass through) — **unless it throws or returns a rejected promise**, which replaces the result (`F`). The pass-through versions take extra microtask ticks (`finally` waits on the callback's result), so the throwing version logs first.

</details>

### Q26. `Promise.all` — result order vs completion order, fail-fast

```js
const d = (v, ms, fail) =>
  new Promise((res, rej) => setTimeout(() => (fail ? rej(v) : res(v)), ms));

Promise.all([d('a', 30), d('b', 10), d('c', 20)]).then((r) => console.log('all', r));
Promise.all([d('a', 30), d('b', 10, true), d('c', 20, true)]).catch((e) => console.log('rejected', e));
```

<details>
<summary><strong>Show Output</strong></summary>

```js
rejected b
all [ 'a', 'b', 'c' ]
```

Results keep **input order** (`['a','b','c']`), not completion order, and the promise resolves only when the slowest (30 ms) finishes. With a rejection, `Promise.all` settles on the **first** rejection (`b` at 10 ms); later rejections (`c`) are handled internally, so there is no unhandled-rejection error — but the other work is *not* cancelled.

</details>

### Q27. race vs any vs allSettled

```js
const d = (v, ms, fail) =>
  new Promise((res, rej) => setTimeout(() => (fail ? rej(v) : res(v)), ms));

Promise.race([d('slow', 30), d('err', 10, true)]).catch((e) => console.log('race rejected', e));
Promise.any([d('err1', 10, true), d('ok', 30), d('err2', 20, true)]).then((v) => console.log('any', v));
Promise.any([d('e1', 5, true), d('e2', 8, true)]).catch((e) => console.log(e.constructor.name, e.errors));
Promise.allSettled([d('x', 5), d('y', 6, true)]).then((r) => console.log(JSON.stringify(r)));
```

<details>
<summary><strong>Show Output</strong></summary>

```js
[{"status":"fulfilled","value":"x"},{"status":"rejected","reason":"y"}]
AggregateError [ 'e1', 'e2' ]
race rejected err
any ok
```

`allSettled` never rejects and waits for everything (6 ms). The second `any` has no fulfilled input, so it rejects with an `AggregateError` whose `errors` array lists the reasons (8 ms). `race` adopts the first *settlement*, even a rejection (10 ms). The first `any` ignores rejections and takes the first *fulfilment* (30 ms).

</details>

### Q28. Edge cases: empty arrays and non-promises

```js
Promise.all([]).then((v) => console.log('all', v));
Promise.allSettled([]).then((v) => console.log('settled', v));
Promise.any([]).catch((e) => console.log(e.name, e.errors));
Promise.race([]).then(
  () => console.log('never'),
  () => console.log('never')
);
Promise.all([1, Promise.resolve(2), { then(r) { r(3); } }]).then((v) => console.log(v));
```

<details>
<summary><strong>Show Output</strong></summary>

```js
all []
settled []
AggregateError []
[ 1, 2, 3 ]
```

`all([])` and `allSettled([])` resolve immediately with `[]`; `any([])` rejects with an empty `AggregateError`; `race([])` **never settles** (nothing printed for it). Plain values and thenables are accepted — each element goes through `Promise.resolve`. A polyfill must handle all of these.

</details>

### Q29. Sequential `await` vs `Promise.all`

```js
const d = (v, ms) =>
  new Promise((r) => setTimeout(() => { console.log('done', v); r(v); }, ms));

(async () => {
  for (const [v, ms] of [['a', 30], ['b', 10]]) await d(v, ms);
  console.log('sequential finished');
  await Promise.all([['c', 30], ['d', 10]].map(([v, ms]) => d(v, ms)));
  console.log('parallel finished');
})();
```

<details>
<summary><strong>Show Output</strong></summary>

```js
done a
done b
sequential finished
done d
done c
parallel finished
```

`await` in a loop starts each task only after the previous one finished (≈ 40 ms total, source order). `Promise.all` over `map` starts them all at once (≈ 30 ms total) and they complete in time order (`d` before `c`).

</details>

### Q30. `forEach` does not await

```js
const sleep = (ms) => new Promise((r) => setTimeout(r, ms));

(async () => {
  [1, 2, 3].forEach(async (n) => {
    await sleep(10);
    console.log('each', n);
  });
  console.log('after forEach');
  for (const n of [4, 5]) {
    await sleep(10);
    console.log('for', n);
  }
  console.log('end');
})();
```

<details>
<summary><strong>Show Output</strong></summary>

```js
after forEach
each 1
each 2
each 3
for 4
for 5
end
```

`forEach` ignores the promises returned by its async callback, so execution continues immediately (`after forEach`) and the three sleeps run concurrently. `for…of` with `await` really pauses. Use `for…of` for sequential work and `Promise.all(arr.map(...))` for parallel work.

</details>

### Q31. `return` vs `return await` inside `try`

```js
async function f1() {
  try {
    return Promise.reject('r1');
  } catch (e) {
    return 'caught1';
  }
}
async function f2() {
  try {
    return await Promise.reject('r2');
  } catch (e) {
    return 'caught2';
  }
}
f1().then(console.log, (e) => console.log('rejected', e));
f2().then(console.log, (e) => console.log('rejected', e));
```

<details>
<summary><strong>Show Output</strong></summary>

```js
caught2
rejected r1
```

`return promise` hands the promise to the caller *without* awaiting it inside the `try`, so the `catch` never sees the rejection. `return await promise` awaits inside the `try`, so the error is catchable. This is a classic reason to write `return await` in `try` blocks.

</details>

### Q32. Promise identity

```js
const p = Promise.resolve(1);
console.log(Promise.resolve(p) === p, new Promise((r) => r(p)) === p, p.then() === p);
```

<details>
<summary><strong>Show Output</strong></summary>

```js
true false false
```

`Promise.resolve(p)` returns the *same* promise when given a native promise from the same realm. `new Promise` and `.then` always create a **new** promise.

</details>

### Q33. `await` on a thenable

```js
const thenable = {
  then(resolve) {
    console.log('then called');
    resolve('value');
  },
};

(async () => {
  console.log('before');
  const v = await thenable;
  console.log('after', v);
})();
console.log('sync end');
```

<details>
<summary><strong>Show Output</strong></summary>

```js
before
sync end
then called
after value
```

`await` wraps the value with `Promise.resolve`, which schedules a microtask to call the thenable's `then` — so `then called` comes *after* the synchronous code, and the function resumes only after `resolve` is called. This is how `Promise.all` and `await` treat any object with a `then` method.

</details>

### Q34. `bind` is permanent — except for `new`

```js
function who() {
  return this.name;
}
const a = { name: 'A' };
const b = { name: 'B' };
const bound = who.bind(a);

console.log(bound(), bound.call(b), bound.bind(b)(), new (who.bind(a))() instanceof who);
```

<details>
<summary><strong>Show Output</strong></summary>

```js
A A A true
```

A bound function ignores later `call`/`apply`/`bind` — `this` stays `a`. The one thing that overrides it is `new`: the bound function acts as a constructor for the *original* function with a fresh `this`, so the result is an instance of `who`. A correct `bind` polyfill must handle this.

</details>

### Q35. Arrow functions ignore `call` / `apply` / `bind`

```js
const o = {
  name: 'o',
  getArrow() {
    return () => this.name;
  },
};
const arrow = o.getArrow();

console.log(arrow(), arrow.call({ name: 'x' }), o.getArrow.call({ name: 'y' })());
```

<details>
<summary><strong>Show Output</strong></summary>

```js
o o y
```

An arrow function captures `this` **lexically** when it is created. `arrow.call({ name: 'x' })` cannot change it (still `'o'`). But calling the *outer* method with a different `this` creates a new arrow bound to `{ name: 'y' }`.

</details>

### Q36. Losing `this` when extracting a method

```js
'use strict';
const user = {
  name: 'Ada',
  hi() {
    return `hi ${this?.name}`;
  },
};
const { hi } = user;

console.log(user.hi(), hi(), (0, user.hi)(), (user.hi)());
```

<details>
<summary><strong>Show Output</strong></summary>

```js
hi Ada hi undefined hi undefined hi Ada
```

`this` is decided by **how** the function is called. `hi()` and `(0, user.hi)()` call the bare function (the comma operator returns the value, dropping the reference), so `this` is `undefined` in strict mode. `(user.hi)()` keeps the property-reference semantics, so `this` is `user`. Without strict mode, `this` would be the global object.

</details>

### Q37. `this` boxing for primitives, `null` and `undefined`

```js
function strictFn() {
  'use strict';
  return typeof this;
}
console.log(strictFn.call(5), strictFn.call(null), strictFn.call(undefined), strictFn.apply('s'));

const sloppy = new Function('return typeof this');
console.log(sloppy.call(5), sloppy.call(null), sloppy.call(undefined));
```

<details>
<summary><strong>Show Output</strong></summary>

```js
number object undefined string
object object object
```

In strict mode `this` is passed through untouched (`typeof null` is `'object'`, `undefined` stays `undefined`). In sloppy mode primitives are **boxed** (`5` → `Number` object) and `null`/`undefined` are replaced by the global object — which is why sloppy `call(null)` gives `'object'`. A `call` polyfill must wrap primitives with `Object(ctx)` to match sloppy semantics.

</details>

### Q38. A naive `bind` polyfill breaks `new`

```js
Function.prototype.naiveBind = function (ctx, ...a) {
  const fn = this;
  return function (...b) {
    return fn.apply(ctx, [...a, ...b]);
  };
};

function P(x, y) {
  this.x = x;
  this.y = y;
}
const NP = P.bind({ ignored: true }, 1);
const NB = P.naiveBind({ ignored: true }, 1);
const p1 = new NP(2);
const p2 = new NB(2);

console.log(p1.x, p1.y, p1 instanceof P);
console.log(p2.x, p2 instanceof P);
```

<details>
<summary><strong>Show Output</strong></summary>

```js
1 2 true
undefined false
```

Native `bind` partially applies arguments, ignores the bound `this` under `new`, and keeps the prototype chain. The naive version writes `x`/`y` onto the bound context object instead, and the instance it returns is a plain object created from the anonymous wrapper — wrong properties and `instanceof` fails. A correct polyfill checks `this instanceof boundFn` and sets `boundFn.prototype`.

</details>

### Q39. `this` in nested callbacks

```js
'use strict';
const counter = {
  count: 0,
  inc() {
    [1, 2, 3].forEach(function () {
      this.count++;
    });
  },
  inc2() {
    [1, 2, 3].forEach(function () {
      this.count++;
    }, this);
  },
  inc3() {
    [1, 2, 3].forEach(() => this.count++);
  },
};

try {
  counter.inc();
} catch (e) {
  console.log(e.constructor.name);
}
counter.inc2();
console.log(counter.count);
counter.inc3();
console.log(counter.count);
```

<details>
<summary><strong>Show Output</strong></summary>

```js
TypeError
3
6
```

A regular callback gets `this === undefined` in strict mode, so `this.count++` throws. Array methods accept a `thisArg` second parameter (`inc2`), and arrow functions capture the enclosing `this` (`inc3`). A `map`/`forEach` polyfill must honour `thisArg`.

</details>

### Q40. `apply` with array-likes, `Math.max` and empty inputs

```js
console.log(
  Math.max.apply(null, [1, 5, 3]),
  Math.max.call(null, 1, 5, 3),
  Math.max(...[]),
  Math.min(),
  Math.max.apply(null, { length: 2, 0: 4, 1: 9 })
);
console.log(Array.prototype.slice.call('abc'), Array.prototype.map.call('abc', (c) => c.toUpperCase()));
```

<details>
<summary><strong>Show Output</strong></summary>

```js
5 5 -Infinity Infinity 9
[ 'a', 'b', 'c' ] [ 'A', 'B', 'C' ]
```

`apply` accepts any array-like (an object with `length` and indices). `Math.max()` with no arguments is `-Infinity` and `Math.min()` is `Infinity` (identity values). Generic array methods work on strings and array-likes via `call` — the trick behind `Array.prototype.slice.call(arguments)`.

</details>

### Q41. `length` and `name` of functions and bound functions

```js
function f(a, b = 2, ...rest) {}
console.log(f.length, f.bind(null, 1).length, f.bind(null).name, (() => {}).name === '');

const g = function () {};
const h = () => {};
console.log(g.name, h.name);
```

<details>
<summary><strong>Show Output</strong></summary>

```js
1 0 bound f true
g h
```

`length` counts parameters **before** the first default or rest parameter (so `1`), which is exactly why `curry(fn)` based on `fn.length` fails with defaults. A bound function's `length` is reduced by the pre-bound arguments, and its name is prefixed with `bound `. Anonymous functions assigned to a variable infer the variable's name.

</details>

### Q42. A constructor that returns something

```js
function A() {
  this.v = 1;
  return { v: 2 };
}
function B() {
  this.v = 1;
  return 5;
}

console.log(new A().v, new B().v, new A() instanceof A, new B() instanceof B);
```

<details>
<summary><strong>Show Output</strong></summary>

```js
2 1 false true
```

If a constructor returns an **object**, that object replaces the new instance (so `instanceof A` is false). Returning a **primitive** is ignored and `this` is used. This rule is what a `new` polyfill (`myNew`) has to reproduce.

</details>

### Q43. Memoize with `JSON.stringify` keys

```js
const memo = (fn) => {
  const cache = new Map();
  return (...args) => {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key);
    const result = fn(...args);
    cache.set(key, result);
    return result;
  };
};

let calls = 0;
const f = memo((x) => {
  calls++;
  return x;
});
f(1); f('1'); f([1]); f(undefined);
console.log(f(null), calls);
f(NaN);
console.log(calls);
```

<details>
<summary><strong>Show Output</strong></summary>

```js
undefined 4
4
```

`JSON.stringify([undefined])`, `[null]` and `[NaN]` all produce `"[null]"`, so those inputs **collide** — `f(null)` returns the cached `undefined` from `f(undefined)` without calling the function. Stringifying also drops functions, symbols and cycles, and is slow for big arguments. Use a nested `Map` per argument or a resolver for production-grade keys.

</details>

### Q44. Memoize with `||` (falsy results are recomputed)

```js
const memoBad = (fn) => {
  const cache = {};
  return (x) => cache[x] || (cache[x] = fn(x));
};

let n = 0;
const g = memoBad((x) => {
  n++;
  return x * 0;
});
g(5); g(5); g(5);
console.log(n);

const key = memoBad((o) => o.id);
console.log(key({ id: 1 }), key({ id: 2 }));
```

<details>
<summary><strong>Show Output</strong></summary>

```js
3
1 1
```

The cached result `0` is falsy, so `||` misses it every time — use `key in cache` / `Map.has`. Also, a plain-object cache coerces keys to strings: both `{ id: 1 }` and `{ id: 2 }` become `"[object Object]"`, so the second call returns the first call's result. `Map` keeps keys by identity.

</details>

### Q45. `curry` relies on `fn.length`

```js
const curry = (fn) =>
  function c(...a) {
    return a.length >= fn.length ? fn.apply(this, a) : (...b) => c.apply(this, [...a, ...b]);
  };

const add3 = curry((a, b, c) => a + b + c);
console.log(add3(1)(2)(3), add3(1, 2)(3), add3(1)(2, 3), add3(1, 2, 3));

const withDefault = curry((a, b = 2, c) => [a, b, c]);
console.log(withDefault(1));
```

<details>
<summary><strong>Show Output</strong></summary>

```js
6 6 6 6
[ 1, 2, undefined ]
```

All call shapes work as long as the arity is known. With a default parameter, `fn.length` is `1`, so `curry` considers one argument enough and calls the function straight away — the fix is to let callers pass the arity (`curry(fn, 3)`).

</details>

### Q46. Infinite currying with `valueOf`

```js
function sum(a) {
  const f = (b) => sum(a + (b ?? 0));
  f.valueOf = () => a;
  return f;
}

console.log(+sum(1)(2)(3), sum(1)(2) + 0, `${sum(5)(5)}`);
```

<details>
<summary><strong>Show Output</strong></summary>

```js
6 3 (b) => sum(a + (b ?? 0))
```

Unary `+` and the `+` operator ask for a *number/default* hint, which calls `valueOf` (our sum). A template literal asks for a **string** hint, which calls `toString` first — the function's default `toString` returns its source code. Define `f.toString` as well (or use `Symbol.toPrimitive`) if both conversions must work.

</details>

### Q47. `once`: cached result and swallowed errors

```js
const once = (fn) => {
  let done = false;
  let result;
  return function (...a) {
    if (!done) {
      done = true;
      result = fn.apply(this, a);
    }
    return result;
  };
};

const init = once((x) => {
  console.log('init', x);
  return x * 2;
});
console.log(init(1), init(2), init(3));

const bad = once(() => {
  throw new Error('first');
});
try {
  bad();
} catch (e) {
  console.log('1st', e.message);
}
console.log(bad());
```

<details>
<summary><strong>Show Output</strong></summary>

```js
init 1
2 2 2
1st first
undefined
```

Arguments after the first call are ignored and the first result is returned forever. Because `done` is set **before** `fn` runs, a throwing function is never retried and later calls silently return `undefined`. Decide deliberately whether failures should reset the flag.

</details>

### Q48. `pipe` vs `compose`

```js
const pipe = (...fns) => (x) => fns.reduce((v, f) => f(v), x);
const compose = (...fns) => (x) => fns.reduceRight((v, f) => f(v), x);

const add1 = (x) => x + 1;
const dbl = (x) => x * 2;
const str = (x) => `<${x}>`;

console.log(pipe(add1, dbl, str)(3), compose(add1, dbl, str)(3));
console.log(pipe()(7), compose(str)(7));
```

<details>
<summary><strong>Show Output</strong></summary>

```js
<8> NaN
7 <7>
```

`pipe` runs left to right: `(3 + 1) * 2 = 8` → `"<8>"`. `compose` runs right to left: `str(3)` = `"<3>"`, `"<3>" * 2` is `NaN`, `NaN + 1` is `NaN`. With no functions, both return the input unchanged (identity).

</details>

### Q49. `reduce` with and without an initial value

```js
console.log([5].reduce((a, b) => a + b), [].reduce((a, b) => a + b, 'init'));
try {
  [].reduce((a, b) => a + b);
} catch (e) {
  console.log(e.constructor.name + ': ' + e.message);
}
console.log([1, 2, 3].reduce((acc, x, i, arr) => acc + x * i + arr.length, 0));
```

<details>
<summary><strong>Show Output</strong></summary>

```js
5 init
TypeError: Reduce of empty array with no initial value
17
```

Without an initial value, the first element is the accumulator and the callback starts at index 1; a single element is returned as-is without calling the callback, and an empty array throws. The last line: `0+0*0+3 = 3`, `3+2*1+3 = 8`, `8+3*2+3 = 17` (the callback gets `(acc, item, index, array)`).

</details>

### Q50. `map(parseInt)`

```js
console.log(['1', '2', '3'].map(parseInt), ['10', '10', '10'].map(Number), ['1', '2', '3'].map((s) => parseInt(s)));
```

<details>
<summary><strong>Show Output</strong></summary>

```js
[ 1, NaN, NaN ] [ 10, 10, 10 ] [ 1, 2, 3 ]
```

`map` passes `(value, index, array)`, so `parseInt('2', 1)` uses radix 1 (invalid → `NaN`) and `parseInt('3', 2)` is `NaN` (3 is not a binary digit). `Number` ignores extra arguments. Wrap in an arrow function to control exactly what is passed.

</details>

### Q51. Sparse arrays and the iteration methods

```js
const sparse = [1, , 3];
let visits = 0;
sparse.forEach(() => visits++);

console.log(visits, sparse.map((x) => x * 2), sparse.length, 1 in sparse, Object.keys(sparse));
console.log([...sparse].map((x) => x), Array.from(sparse, (x) => x));
```

<details>
<summary><strong>Show Output</strong></summary>

```js
2 [ 2, <1 empty item>, 6 ] 3 false [ '0', '2' ]
[ 1, undefined, 3 ] [ 1, undefined, 3 ]
```

`forEach`, `map`, `filter` and `reduce` **skip holes** (the index is not "in" the array) and `map` preserves them. Spread and `Array.from` turn holes into real `undefined` values. A faithful `Array.prototype.map` polyfill must check `i in this`.

</details>

### Q52. Mutating an array while iterating

```js
const arr = [1, 2, 3];
arr.forEach((x) => {
  if (x === 1) arr.push(99);
  console.log(x);
});
console.log(arr);

const arr2 = [1, 2, 3, 4];
arr2.forEach((x, i) => {
  if (i === 0) arr2.shift();
  console.log(x);
});
```

<details>
<summary><strong>Show Output</strong></summary>

```js
1
2
3
[ 1, 2, 3, 99 ]
1
3
4
```

The number of iterations is fixed from the **initial length** (3), so the pushed `99` is not visited. Elements removed before being reached are skipped: after `shift()` the array is `[2, 3, 4]`, index 1 is `3`, index 2 is `4`, and index 3 no longer exists (so it is skipped). Iterate over a copy when you mutate.

</details>

### Q53. `sort` defaults

```js
console.log(
  [10, 9, 1, 100].sort(),
  [10, 9, 1, 100].sort((a, b) => a - b),
  ['b', undefined, 'a'].sort(),
  [3, 1, 2].sort(() => 0)
);
```

<details>
<summary><strong>Show Output</strong></summary>

```js
[ 1, 10, 100, 9 ] [ 1, 9, 10, 100 ] [ 'a', 'b', undefined ] [ 3, 1, 2 ]
```

Without a comparator, elements are compared as **strings** (`"10" < "9"`). `undefined` always sorts to the end and the comparator is never called for it. `sort` is stable (ES2019), so an always-equal comparator keeps the original order. `sort` also **mutates** the array — copy first with `[...arr]` or `toSorted`.

</details>

### Q54. `flat` and `flatMap`

```js
console.log(
  [1, [2, [3, [4]]]].flat(),
  [1, [2, [3, [4]]]].flat(Infinity),
  [[1, 2], [3]].flatMap((x) => x),
  [1, 2].flatMap((x) => [[x]]),
  [1, , 3].flat()
);
```

<details>
<summary><strong>Show Output</strong></summary>

```js
[ 1, 2, [ 3, [ 4 ] ] ] [ 1, 2, 3, 4 ] [ 1, 2, 3 ] [ [ 1 ], [ 2 ] ] [ 1, 3 ]
```

`flat()` defaults to depth `1`; pass `Infinity` to flatten fully. `flatMap` is `map` + `flat(1)` — it removes only **one** level, so `[[x]]` becomes `[x]`. `flat` also drops holes. A recursive `flatten(arr, depth)` polyfill needs a depth parameter and hole handling.

</details>

### Q55. What `JSON.parse(JSON.stringify(x))` loses

```js
const src = {
  d: new Date(0),
  u: undefined,
  f() {},
  n: NaN,
  i: Infinity,
  m: new Map([[1, 2]]),
  s: new Set([1]),
  r: /x/g,
  nested: { a: [1, , 3] },
};
console.log(JSON.stringify(JSON.parse(JSON.stringify(src))));
```

<details>
<summary><strong>Show Output</strong></summary>

```js
{"d":"1970-01-01T00:00:00.000Z","n":null,"i":null,"m":{},"s":{},"r":{},"nested":{"a":[1,null,3]}}
```

`Date` becomes a string, `undefined` and functions disappear from objects (and become `null` inside arrays), `NaN`/`Infinity` become `null`, `Map`/`Set`/`RegExp` become `{}`, holes become `null`. It also throws on circular references and `BigInt`. That is why a hand-written `deepClone` needs type checks.

</details>

### Q56. Spread is shallow; `structuredClone` is deep

```js
const o = { a: { b: 1 }, d: new Date(0) };
const shallow = { ...o };
const deep = structuredClone(o);
shallow.a.b = 2;

console.log(o.a.b, deep.a.b, deep.d instanceof Date, deep.d === o.d);
try {
  structuredClone({ f() {} });
} catch (e) {
  console.log(e.name);
}
```

<details>
<summary><strong>Show Output</strong></summary>

```js
2 1 true false
DataCloneError
```

Spreading copies only the top level, so the nested object is shared (`o.a.b` becomes `2`). `structuredClone` deep-copies dates, maps, sets, typed arrays and cycles, but cannot clone **functions** (or DOM nodes, class methods/prototypes) — it throws `DataCloneError`.

</details>

### Q57. Circular references

```js
const a = { name: 'a' };
a.self = a;

try {
  JSON.stringify(a);
} catch (e) {
  console.log(e.constructor.name);
}
const copy = structuredClone(a);
console.log(copy.self === copy, copy === a);
```

<details>
<summary><strong>Show Output</strong></summary>

```js
TypeError
true false
```

`JSON.stringify` throws on cycles. A deep-clone polyfill handles them with a `WeakMap` from original → clone: before recursing, register the clone; if the node is seen again, return the existing clone. `structuredClone` does this internally.

</details>

### Q58. Loose equality

```js
console.log([] == false, [] == ![], null == undefined, null == 0, NaN == NaN, [1] == 1, '0' == false, {} == '[object Object]');
```

<details>
<summary><strong>Show Output</strong></summary>

```js
true true true false false true true true
```

`[] == false`: `false → 0`, `[] → '' → 0`. `[] == ![]`: `![]` is `false`, same as above. `null == undefined` is true but `null` equals nothing else under `==`. `NaN` is never equal to itself. `[1]` converts to `'1'` then `1`. Prefer `===`.

</details>

### Q59. `NaN`, `-0` and the different equality algorithms

```js
console.log(
  Object.is(NaN, NaN),
  Object.is(0, -0),
  0 === -0,
  [NaN].includes(NaN),
  [NaN].indexOf(NaN),
  new Set([NaN, NaN, 0, -0]).size,
  new Map([[NaN, 'x']]).get(NaN)
);
```

<details>
<summary><strong>Show Output</strong></summary>

```js
true false true true -1 2 x
```

`===` and `indexOf` use strict equality (`NaN !== NaN`, `0 === -0`). `Object.is` distinguishes `-0` and treats `NaN` as equal to itself. `includes`, `Set` and `Map` use **SameValueZero**: `NaN` equals `NaN` and `0` equals `-0`. This matters for `memoize` keys and `deepEqual`.

</details>

### Q60. Type checks used by a `deepClone`

```js
console.log(
  typeof null,
  typeof [],
  Array.isArray([]),
  Object.prototype.toString.call(new Date(0)),
  Object.prototype.toString.call(null),
  Object.prototype.toString.call(/x/),
  [] instanceof Object,
  Object.create(null) instanceof Object
);
```

<details>
<summary><strong>Show Output</strong></summary>

```js
object object true [object Date] [object Null] [object RegExp] true false
```

`typeof` cannot tell arrays, `null`, dates or regexes apart. `Object.prototype.toString.call(x)` returns a reliable tag, and `Array.isArray` is the safe array check. Objects created with `Object.create(null)` have no prototype, so `instanceof Object` is false — and `obj.hasOwnProperty` does not exist on them (use `Object.hasOwn`).

</details>

### Q61. Emitter: adding and removing listeners during `emit`

```js
class Emitter {
  constructor() {
    this.l = new Map();
  }
  on(e, f) {
    if (!this.l.has(e)) this.l.set(e, []);
    this.l.get(e).push(f);
    return this;
  }
  off(e, f) {
    this.l.set(e, (this.l.get(e) || []).filter((x) => x !== f));
  }
  emit(e, ...a) {
    (this.l.get(e) || []).slice().forEach((f) => f(...a));
  }
}

const em = new Emitter();
const second = () => console.log('second');
em.on('x', () => {
  console.log('first');
  em.off('x', second);
  em.on('x', () => console.log('added'));
});
em.on('x', second);

em.emit('x');
console.log('--');
em.emit('x');
```

<details>
<summary><strong>Show Output</strong></summary>

```js
first
second
--
first
added
```

`emit` iterates over a **snapshot** (`slice()`), so removing `second` during the first emit does not stop it from running, and the listener added during the emit does not run until the next one. Iterating the live array would skip listeners or loop forever.

</details>

### Q62. `once` and `off` with the original function

```js
const listeners = [];
const on = (f) => listeners.push(f);
const off = (f) => {
  const i = listeners.indexOf(f);
  if (i > -1) listeners.splice(i, 1);
};
const once = (f) => {
  const wrapper = (...a) => {
    off(wrapper);
    f(...a);
  };
  on(wrapper);
};

const handler = () => console.log('hit');
once(handler);
off(handler);
console.log(listeners.length);

listeners.slice().forEach((f) => f());
console.log(listeners.length);
```

<details>
<summary><strong>Show Output</strong></summary>

```js
1
hit
0
```

`once` stores a **wrapper**, so `off(handler)` cannot find it and the listener stays. Real emitters fix this by tagging the wrapper (`wrapper.listener = handler`) and comparing against it in `off`. Note the wrapper removes itself *before* calling `f`, so a re-entrant `emit` cannot fire it twice.

</details>

### Q63. `debounce` must preserve `this` and arguments

```js
function debounce(fn, ms) {
  let t;
  return function (...a) {
    clearTimeout(t);
    t = setTimeout(() => fn.apply(this, a), ms);
  };
}

const obj = {
  v: 'obj',
  m: debounce(function () {
    console.log('fn', this.v);
  }, 10),
  arrowM: debounce(() => console.log('arrow', this?.v), 10),
};
obj.m();
obj.arrowM();
```

<details>
<summary><strong>Show Output</strong></summary>

```js
fn obj
arrow undefined
```

The returned wrapper is a regular function, so it receives `this === obj` from `obj.m()` and forwards it with `apply`. The `setTimeout` callback is an arrow function, which keeps that `this`. The *callback* written as an arrow function inside the object literal uses the module-level `this` (`undefined`), no matter how it is called — don't write debounced methods as arrows.

</details>

### Q64. Leading-edge debounce

```js
function debounce(fn, ms, immediate = false) {
  let t;
  return (...a) => {
    const callNow = immediate && !t;
    clearTimeout(t);
    t = setTimeout(() => {
      t = null;
      if (!immediate) fn(...a);
    }, ms);
    if (callNow) fn(...a);
  };
}

const lead = debounce((x) => console.log('lead', x), 20, true);
lead(1);
lead(2);
lead(3);
setTimeout(() => lead(4), 50);
```

<details>
<summary><strong>Show Output</strong></summary>

```js
lead 1
lead 4
```

In leading mode the first call of a burst runs immediately and the rest of the burst is ignored; the timer only marks "quiet again" (it is reset by `lead(2)` and `lead(3)`). After 20 ms without calls the flag clears, so `lead(4)` at 50 ms starts a new burst. Search-as-you-type needs the trailing edge; "submit" buttons want the leading edge.

</details>

### Q65. Timestamp-based throttle (leading only)

```js
const throttle = (fn, ms) => {
  let last = 0;
  return (...a) => {
    const now = Date.now();
    if (now - last >= ms) {
      last = now;
      fn(...a);
    }
  };
};

const t = throttle((x) => console.log('run', x), 50);
t(1);
t(2);
setTimeout(() => t(3), 20);
setTimeout(() => t(4), 70);
```

<details>
<summary><strong>Show Output</strong></summary>

```js
run 1
run 4
```

Calls inside the window are dropped (`2` and `3`) and there is **no trailing call**, so the final burst can be lost entirely. If the last event must be delivered (a final scroll position, a resize), add a trailing timer as the `throttle` implementation above does.

</details>

### Q66. `cancel` and `flush`

```js
function debounce(fn, ms) {
  let t;
  let lastArgs;
  const d = (...a) => {
    lastArgs = a;
    clearTimeout(t);
    t = setTimeout(() => {
      t = null;
      fn(...a);
    }, ms);
  };
  d.cancel = () => {
    clearTimeout(t);
    t = null;
  };
  d.flush = () => {
    if (t) {
      clearTimeout(t);
      t = null;
      fn(...lastArgs);
    }
  };
  return d;
}

const d = debounce((x) => console.log('run', x), 20);
d('a');
d.flush();
d.flush();
d('b');
d.cancel();
d('c');
setTimeout(() => console.log('end'), 50);
```

<details>
<summary><strong>Show Output</strong></summary>

```js
run a
run c
end
```

`flush` runs a pending call right now (and only once — the second `flush` has nothing pending). `cancel` drops the pending call (`b`). `c` is scheduled normally and fires after 20 ms. Expose `cancel` so a React `useEffect` cleanup can stop a pending call on unmount.

</details>

### Q67. Timeout wrapper with `Promise.race`

```js
const sleep = (ms, v) => new Promise((r) => setTimeout(() => r(v), ms));
const timeout = (p, ms) =>
  Promise.race([p, new Promise((_, rej) => setTimeout(() => rej(new Error('timeout')), ms))]);

timeout(sleep(50, 'slow'), 20).then(console.log, (e) => console.log(e.message));
timeout(sleep(10, 'fast'), 40).then(console.log, (e) => console.log(e.message));
```

<details>
<summary><strong>Show Output</strong></summary>

```js
fast
timeout
```

`race` settles with whichever promise settles first: `fast` wins at 10 ms; the slow one loses to the 20 ms timer. Two caveats: the losing promise is **not cancelled** (the 50 ms sleep still runs), and the losing timer keeps running unless you `clearTimeout` it — use `AbortController` and clean up the timer in real code.

</details>

### Q68. Retry with exponential backoff — attempts and delays

```js
async function retry(fn, retries, base, log) {
  for (let i = 0; ; i++) {
    try {
      return await fn(i);
    } catch (e) {
      if (i >= retries) throw e;
      log.push(base * 2 ** i);
    }
  }
}

(async () => {
  const d1 = [];
  console.log(
    await retry(async (i) => {
      if (i < 3) throw new Error('e' + i);
      return 'ok' + i;
    }, 5, 10, d1),
    d1
  );

  const d2 = [];
  await retry(async (i) => {
    throw new Error('always' + i);
  }, 2, 10, d2).catch((e) => console.log(e.message, d2));
})();
```

<details>
<summary><strong>Show Output</strong></summary>

```js
ok3 [ 10, 20, 40 ]
always2 [ 10, 20 ]
```

Delays double each time (`base * 2 ** attempt`). The first function succeeds on its 4th call (index 3) after three recorded waits. With `retries = 2` the function runs 3 times, waits twice, and the **last** error is rethrown. (The sleep is omitted here so the example is deterministic; add `await sleep(delay)` — and jitter — in real code.)

</details>

### Q69. Polluting `Array.prototype`

```js
Array.prototype.myMap = function () {};
const arr = [1, 2];
for (const k in arr) console.log(k);
console.log(Object.keys(arr), arr.hasOwnProperty('myMap'));
delete Array.prototype.myMap;

Object.defineProperty(Array.prototype, 'myMap2', {
  value() {},
  writable: true,
  configurable: true,
});
for (const k in [1]) console.log(k);
delete Array.prototype.myMap2;
```

<details>
<summary><strong>Show Output</strong></summary>

```js
0
1
myMap
[ '0', '1' ] false
0
```

Assigning `Array.prototype.x = …` creates an **enumerable** property, so every `for…in` over any array now yields `myMap`. `Object.keys` shows only own properties. Polyfills should be added with `Object.defineProperty` (non-enumerable, writable, configurable) and guarded with `if (!Array.prototype.myMap)` so a native implementation is never overwritten.

</details>

### Q70. Hoisting and the temporal dead zone

```js
console.log(typeof hoisted, typeof expr, typeof arrow);
function hoisted() {}
var expr = function () {};
var arrow = () => {};

try {
  console.log(tdz);
} catch (e) {
  console.log(e.name);
}
let tdz = 1;
```

<details>
<summary><strong>Show Output</strong></summary>

```js
function undefined undefined
ReferenceError
```

Function **declarations** are hoisted with their body. `var` declarations are hoisted but initialised to `undefined`, so function *expressions* and arrows assigned to `var` are not callable yet. `let`/`const` are hoisted but sit in the temporal dead zone until their declaration runs, so reading them throws `ReferenceError`.

</details>

### Q71. Closure over a variable vs a copy

```js
function createStore() {
  let state = 0;
  const snapshot = state;
  const get = () => state;
  return {
    inc() {
      state++;
    },
    get,
    snapshot,
  };
}

const s = createStore();
s.inc();
s.inc();
console.log(s.get(), s.snapshot);
```

<details>
<summary><strong>Show Output</strong></summary>

```js
2 0
```

`get` closes over the **variable** `state`, so it always reads the current value. `snapshot` was a number copied into the returned object at creation time and never changes. This is the same reason a React effect that captured `count` sees a stale value — closures capture bindings, but state values copied into a render are frozen per render.

</details>

### Q72. `Object.freeze` is shallow

```js
const frozen = Object.freeze({ a: 1, nested: { b: 2 }, arr: [1] });
console.log(Reflect.set(frozen, 'a', 99));
frozen.nested.b = 99;
frozen.arr.push(2);
console.log(frozen.a, frozen.nested.b, frozen.arr, Object.isFrozen(frozen.nested));
```

<details>
<summary><strong>Show Output</strong></summary>

```js
false
1 99 [ 1, 2 ] false
```

`freeze` blocks adding, removing and changing the object's **own** properties (assignment fails silently in sloppy mode and throws in strict mode — `Reflect.set` just returns `false`). Nested objects and arrays stay mutable, so immutability for state needs a recursive `deepFreeze` or immutable updates.

</details>

### Q73. Spread, `Array.from` and strings

```js
console.log(
  [...'a😀b'].length,
  'a😀b'.length,
  Array.from({ length: 3 }, (_, i) => i * 2),
  Array(3).map((_, i) => i),
  [...Array(3)].map((_, i) => i)
);
```

<details>
<summary><strong>Show Output</strong></summary>

```js
3 4 [ 0, 2, 4 ] [ <3 empty items> ] [ 0, 1, 2 ]
```

Spread uses the string iterator, which walks **code points** (the emoji counts once); `.length` counts UTF-16 units (the emoji counts twice). `Array.from` accepts any array-like plus a mapper. `Array(3)` makes an array of *holes*, which `map` skips; spreading it creates real `undefined` entries that `map` visits.

</details>

### Q74. Property order

```js
const o = { b: 1, 2: 'x', a: 2, 1: 'y', [Symbol('s')]: 3 };
console.log(Object.keys(o), JSON.stringify(o));
```

<details>
<summary><strong>Show Output</strong></summary>

```js
[ '1', '2', 'b', 'a' ] {"1":"y","2":"x","b":1,"a":2}
```

Integer-like keys come first in ascending order, then string keys in insertion order, then symbols. Symbols are skipped by `Object.keys`, `for…in` and `JSON.stringify`. Because of the ordering rule, comparing objects with `JSON.stringify` works only if both were built with the same key order.

</details>

### Q75. `JSON.stringify` quirks that break naive equality

```js
console.log(
  JSON.stringify({ a: 1, b: 2 }) === JSON.stringify({ b: 2, a: 1 }),
  JSON.stringify({ a: undefined, b: () => {} }),
  JSON.stringify([undefined, () => {}]),
  JSON.stringify({ d: new Date(0) })
);
```

<details>
<summary><strong>Show Output</strong></summary>

```js
false {} [null,null] {"d":"1970-01-01T00:00:00.000Z"}
```

Key order matters, `undefined`/function values vanish from objects and become `null` in arrays, and `Date` goes through `toJSON`. Use a recursive `deepEqual` (comparing types, key sets and values) instead of stringifying.

</details>

### Q76. `Object` vs `Map` as a cache

```js
const objCache = {};
const k1 = { id: 1 };
const k2 = { id: 2 };
objCache[k1] = 'one';
objCache[k2] = 'two';
const mapCache = new Map([[k1, 'one'], [k2, 'two']]);

console.log(Object.keys(objCache), objCache[k1], mapCache.get(k1), mapCache.size);
```

<details>
<summary><strong>Show Output</strong></summary>

```js
[ '[object Object]' ] two one 2
```

Plain-object keys are converted to **strings**, so both objects collapse into the single key `"[object Object]"` and the second write overwrites the first. `Map` keys can be any value and are compared by identity. Use `WeakMap` when keys are objects that should be garbage-collected (e.g. memoizing per-DOM-node).

</details>

### Q77. Type coercion with `+`, `-` and `*`

```js
console.log([1 + '2', '3' - 1, '3' + 1 - 1, [] + [], [] + {}, +'', +' 12 ', +'1,2', true + true, [2] * [3]]);
```

<details>
<summary><strong>Show Output</strong></summary>

```js
[ '12', 2, 30, '', '[object Object]', 0, 12, NaN, 2, 6 ]
```

`+` concatenates if either side is a string (`1 + '2'` → `'12'`); `-` and `*` always convert to numbers (`'3' - 1` → `2`). `'3' + 1 - 1` is `('3' + 1) - 1` = `'31' - 1` = `30`. Arrays and objects are converted with `toString` (`[] → ''`, `{} → '[object Object]'`). `+''` is `0`, whitespace is trimmed, `'1,2'` is `NaN`, `true + true` is `2`, and `[2] * [3]` is `6`.

</details>
