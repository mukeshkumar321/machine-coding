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

> Each question uses the implementations above (`debounce`, `throttle`, `promiseAll`, `myBind`, …). Every output below was produced by actually running the code in Node.

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
