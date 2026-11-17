# Shopping Cart — Machine Coding

> **Difficulty:** Medium | **Time:** 60 min | **Frequency:** ⭐⭐⭐⭐⭐

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

Build a **shopping cart** with a product list and a cart panel. Users can add products, change quantities, remove items, apply a coupon code and see an order summary (subtotal, discount, shipping, tax, total). The cart must survive a page refresh.

State must be managed with **`useReducer` + Context** so any component (header badge, product card, cart page) can read and update the cart.

---

## 2. Requirements

### Functional

- Product list with "Add to cart".
- Adding an existing product increments quantity (no duplicate rows).
- Cart rows: increase / decrease quantity (decreasing from 1 removes it), remove item, line total.
- Quantity cap per item (e.g. 10).
- **Coupons**: `SAVE10` (10% off), `FLAT50` ($50 off, min subtotal $200), `FREESHIP` (free shipping). Show an error for invalid/ineligible codes; allow removing.
- **Order summary**: subtotal, discount, shipping (free ≥ $100 after discount), tax (8%), total.
- Header shows item count.
- **Persistence** across reloads (localStorage).

### Non-functional

- All money is calculated in **integer cents** — no floating-point errors.
- Totals are **derived**, never stored.
- Reducer is pure; context split so product cards don't re-render on cart changes.
- Robust against corrupted localStorage.

---

## 3. Clarifying Questions to Ask

- Is there a backend, or purely client state? Is the cart per-user?
- Can there be multiple coupons or only one? Stackable? Min-spend rules?
- Is the percentage discount applied before or after tax? Tax on shipping?
- Stock limits / max quantity? Variants (size/colour) → is a "line" keyed by `productId + variant`?
- Price currency and rounding rules?
- Should decreasing quantity to 0 remove the item?
- Persist where (localStorage vs server)? Multi-tab sync?

---

## 4. Approach & Component Design

**State shape (normalised minimal source of truth)**

```js
{
  items: [{ id: 'p1', name: 'Headphones', price: 5999, qty: 2 }], // price in cents
  coupon: 'SAVE10' | null                                        // just the code
}
// derived (never stored): subtotal, discount, shipping, tax, total, itemCount
```

**Component tree**

```
<CartProvider>                       reducer + persistence + 2 contexts
  <App>
    ├─ <Header>  (badge: itemCount)           useCart()
    ├─ <ProductList> → <ProductCard>          useCartActions() only
    └─ <Cart>
         ├─ <CartItem /> x n                  useCartActions()
         ├─ <CouponForm />
         └─ <Summary />                       totals
```

**Actions**

| Action | Payload | Effect |
| --- | --- | --- |
| `ADD` | product | qty+1 or push new line |
| `INCREMENT` / `DECREMENT` | id | qty ±1 (0 removes) |
| `REMOVE` | id | filter out |
| `APPLY_COUPON` / `REMOVE_COUPON` | code | set / clear `coupon` |
| `CLEAR` | — | reset |
| `HYDRATE` | state | replace (cross-tab sync) |

**Why two contexts?** `CartStateContext` changes on every cart update; `CartActionsContext` (stable `dispatch` wrappers) never changes. Components that only trigger actions (product cards) subscribe to the second and skip re-renders.

---

## 5. Reference Solution

<details>
<summary><strong>Show Full Code</strong></summary>

**`cartLogic.js`** — pure logic, easy to unit-test

```js
export const MAX_QTY = 10;
export const TAX_RATE = 0.08;
export const SHIPPING_FLAT = 599; // cents
export const FREE_SHIPPING_THRESHOLD = 10000; // cents
export const STORAGE_KEY = 'cart:v1';

export const COUPONS = {
  SAVE10: { type: 'percent', value: 10, label: '10% off' },
  FLAT50: { type: 'flat', value: 5000, minSubtotal: 20000, label: '$50 off orders over $200' },
  FREESHIP: { type: 'shipping', label: 'Free shipping' },
};

export const initialState = { items: [], coupon: null };

// hasOwn avoids matching inherited keys such as "constructor".
export const getCoupon = (code) => (Object.hasOwn(COUPONS, code) ? COUPONS[code] : undefined);

export const formatPrice = (cents) =>
  new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' }).format(cents / 100);

export function cartReducer(state, action) {
  switch (action.type) {
    case 'ADD': {
      const { id, name, price } = action.product;
      const existing = state.items.find((i) => i.id === id);
      if (existing) {
        return {
          ...state,
          items: state.items.map((i) =>
            i.id === id ? { ...i, qty: Math.min(i.qty + 1, MAX_QTY) } : i
          ),
        };
      }
      return { ...state, items: [...state.items, { id, name, price, qty: 1 }] };
    }
    case 'INCREMENT':
      return {
        ...state,
        items: state.items.map((i) =>
          i.id === action.id ? { ...i, qty: Math.min(i.qty + 1, MAX_QTY) } : i
        ),
      };
    case 'DECREMENT':
      return {
        ...state,
        items: state.items
          .map((i) => (i.id === action.id ? { ...i, qty: i.qty - 1 } : i))
          .filter((i) => i.qty > 0),
      };
    case 'REMOVE':
      return { ...state, items: state.items.filter((i) => i.id !== action.id) };
    case 'APPLY_COUPON':
      return { ...state, coupon: action.code };
    case 'REMOVE_COUPON':
      return { ...state, coupon: null };
    case 'CLEAR':
      return initialState;
    case 'HYDRATE':
      return action.state;
    default:
      return state;
  }
}

// Returns the normalised code plus an error message ('' when the coupon is OK).
export function validateCoupon(rawCode, subtotal) {
  const code = rawCode.trim().toUpperCase();
  if (!code) return { code, error: 'Enter a coupon code' };
  const coupon = getCoupon(code);
  if (!coupon) return { code, error: 'Invalid coupon code' };
  if (coupon.minSubtotal && subtotal < coupon.minSubtotal) {
    return { code, error: `Requires a subtotal of ${formatPrice(coupon.minSubtotal)}` };
  }
  return { code, error: '' };
}

export function getTotals({ items, coupon: code }) {
  const subtotal = items.reduce((sum, i) => sum + i.price * i.qty, 0);
  const itemCount = items.reduce((sum, i) => sum + i.qty, 0);
  const coupon = getCoupon(code);

  let discount = 0;
  let freeShipping = false;
  // A coupon can become ineligible after items are removed — recompute, don't trust.
  const couponActive = Boolean(coupon) && !(coupon.minSubtotal && subtotal < coupon.minSubtotal);

  if (couponActive) {
    if (coupon.type === 'percent') discount = Math.round((subtotal * coupon.value) / 100);
    if (coupon.type === 'flat') discount = Math.min(coupon.value, subtotal);
    if (coupon.type === 'shipping') freeShipping = true;
  }

  const taxable = subtotal - discount;
  const shipping =
    items.length === 0 || freeShipping || taxable >= FREE_SHIPPING_THRESHOLD ? 0 : SHIPPING_FLAT;
  const tax = Math.round(taxable * TAX_RATE);
  const total = taxable + shipping + tax;

  return { subtotal, itemCount, discount, shipping, tax, total, couponActive };
}

// ---------- persistence ----------
const isValidItem = (i) =>
  i && typeof i.id === 'string' && typeof i.name === 'string' &&
  Number.isInteger(i.price) && Number.isInteger(i.qty) && i.qty > 0 && i.qty <= MAX_QTY;

export function parseStored(json) {
  try {
    const data = JSON.parse(json);
    if (!data || !Array.isArray(data.items)) return initialState;
    return {
      items: data.items.filter(isValidItem),
      coupon: typeof data.coupon === 'string' && getCoupon(data.coupon) ? data.coupon : null,
    };
  } catch {
    return initialState;
  }
}

export function loadState() {
  try {
    const json = localStorage.getItem(STORAGE_KEY);
    return json ? parseStored(json) : initialState;
  } catch {
    return initialState; // storage blocked (private mode etc.)
  }
}

export function saveState(state) {
  try {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(state));
  } catch {
    /* quota exceeded or blocked — cart still works in memory */
  }
}
```

**`CartContext.jsx`**

```jsx
import {
  createContext,
  useContext,
  useEffect,
  useMemo,
  useReducer,
} from 'react';
import {
  STORAGE_KEY,
  cartReducer,
  getTotals,
  initialState,
  loadState,
  parseStored,
  saveState,
} from './cartLogic';

const CartStateContext = createContext(null);
const CartActionsContext = createContext(null);

export function CartProvider({ children }) {
  // Lazy init: read localStorage once, not on every render.
  const [state, dispatch] = useReducer(cartReducer, initialState, loadState);

  useEffect(() => {
    saveState(state);
  }, [state]);

  // Keep multiple tabs in sync.
  useEffect(() => {
    const onStorage = (e) => {
      if (e.key === STORAGE_KEY && e.newValue) {
        dispatch({ type: 'HYDRATE', state: parseStored(e.newValue) });
      }
    };
    window.addEventListener('storage', onStorage);
    return () => window.removeEventListener('storage', onStorage);
  }, []);

  const totals = useMemo(() => getTotals(state), [state]);
  const stateValue = useMemo(() => ({ ...state, totals }), [state, totals]);

  // dispatch is stable, so this object never changes.
  const actions = useMemo(
    () => ({
      addItem: (product) => dispatch({ type: 'ADD', product }),
      increment: (id) => dispatch({ type: 'INCREMENT', id }),
      decrement: (id) => dispatch({ type: 'DECREMENT', id }),
      removeItem: (id) => dispatch({ type: 'REMOVE', id }),
      applyCoupon: (code) => dispatch({ type: 'APPLY_COUPON', code }),
      removeCoupon: () => dispatch({ type: 'REMOVE_COUPON' }),
      clear: () => dispatch({ type: 'CLEAR' }),
    }),
    []
  );

  return (
    <CartActionsContext.Provider value={actions}>
      <CartStateContext.Provider value={stateValue}>{children}</CartStateContext.Provider>
    </CartActionsContext.Provider>
  );
}

export function useCart() {
  const ctx = useContext(CartStateContext);
  if (!ctx) throw new Error('useCart must be used within <CartProvider>');
  return ctx;
}

export function useCartActions() {
  const ctx = useContext(CartActionsContext);
  if (!ctx) throw new Error('useCartActions must be used within <CartProvider>');
  return ctx;
}
```

**`App.jsx`**

```jsx
import { memo, useState } from 'react';
import { CartProvider, useCart, useCartActions } from './CartContext';
import { COUPONS, MAX_QTY, formatPrice, validateCoupon } from './cartLogic';
import './cart.css';

const PRODUCTS = [
  { id: 'p1', name: 'Wireless Headphones', price: 5999 },
  { id: 'p2', name: 'Mechanical Keyboard', price: 8999 },
  { id: 'p3', name: 'USB-C Cable', price: 1299 },
  { id: 'p4', name: '27" Monitor', price: 22999 },
  { id: 'p5', name: 'Laptop Stand', price: 3499 },
];

// memo + stable `addItem` => cards don't re-render when the cart changes.
const ProductCard = memo(function ProductCard({ product }) {
  const { addItem } = useCartActions();
  return (
    <li className="product">
      <span className="product__name">{product.name}</span>
      <span className="product__price">{formatPrice(product.price)}</span>
      <button type="button" onClick={() => addItem(product)}>
        Add to cart
      </button>
    </li>
  );
});

function CartItem({ item }) {
  const { increment, decrement, removeItem } = useCartActions();
  return (
    <li className="cart-item">
      <div className="cart-item__info">
        <strong>{item.name}</strong>
        <span>{formatPrice(item.price)} each</span>
      </div>
      <div className="qty">
        <button type="button" aria-label={`Decrease quantity of ${item.name}`} onClick={() => decrement(item.id)}>
          −
        </button>
        <span aria-live="polite">{item.qty}</span>
        <button
          type="button"
          aria-label={`Increase quantity of ${item.name}`}
          disabled={item.qty >= MAX_QTY}
          onClick={() => increment(item.id)}
        >
          +
        </button>
      </div>
      <span className="cart-item__total">{formatPrice(item.price * item.qty)}</span>
      <button type="button" className="link" aria-label={`Remove ${item.name}`} onClick={() => removeItem(item.id)}>
        Remove
      </button>
    </li>
  );
}

function CouponForm() {
  const { coupon, totals } = useCart();
  const { applyCoupon, removeCoupon } = useCartActions();
  const [input, setInput] = useState('');
  const [error, setError] = useState('');

  const handleSubmit = (e) => {
    e.preventDefault();
    const result = validateCoupon(input, totals.subtotal);
    if (result.error) return setError(result.error);
    applyCoupon(result.code);
    setInput('');
    setError('');
  };

  return (
    <div className="coupon">
      {coupon ? (
        <p className={totals.couponActive ? 'coupon__applied' : 'coupon__inactive'}>
          <strong>{coupon}</strong> — {COUPONS[coupon].label}
          {!totals.couponActive && ' (not eligible for current subtotal)'}{' '}
          <button type="button" className="link" onClick={removeCoupon}>
            Remove
          </button>
        </p>
      ) : (
        <form onSubmit={handleSubmit}>
          <label htmlFor="coupon">Coupon code</label>
          <div className="coupon__row">
            <input
              id="coupon"
              value={input}
              onChange={(e) => {
                setInput(e.target.value);
                setError('');
              }}
              aria-invalid={Boolean(error)}
              aria-describedby={error ? 'coupon-error' : undefined}
              placeholder="SAVE10, FLAT50, FREESHIP"
            />
            <button type="submit">Apply</button>
          </div>
          {error && (
            <p id="coupon-error" role="alert" className="error">
              {error}
            </p>
          )}
        </form>
      )}
    </div>
  );
}

function Summary() {
  const { totals } = useCart();
  const rows = [
    ['Subtotal', formatPrice(totals.subtotal)],
    totals.discount > 0 && ['Discount', `−${formatPrice(totals.discount)}`],
    ['Shipping', totals.shipping === 0 ? 'Free' : formatPrice(totals.shipping)],
    ['Tax (8%)', formatPrice(totals.tax)],
  ].filter(Boolean);

  return (
    <dl className="summary" aria-live="polite">
      {rows.map(([label, value]) => (
        <div key={label} className="summary__row">
          <dt>{label}</dt>
          <dd>{value}</dd>
        </div>
      ))}
      <div className="summary__row summary__row--total">
        <dt>Total</dt>
        <dd>{formatPrice(totals.total)}</dd>
      </div>
    </dl>
  );
}

function Cart() {
  const { items } = useCart();
  const { clear } = useCartActions();

  if (items.length === 0) return <p>Your cart is empty.</p>;

  return (
    <>
      <ul className="cart-list">
        {items.map((item) => (
          <CartItem key={item.id} item={item} />
        ))}
      </ul>
      <CouponForm />
      <Summary />
      <button type="button" className="link" onClick={clear}>
        Clear cart
      </button>
    </>
  );
}

function Header() {
  const { totals } = useCart();
  return (
    <header className="header">
      <h1>Shop</h1>
      <span aria-live="polite">Cart: {totals.itemCount} item{totals.itemCount === 1 ? '' : 's'}</span>
    </header>
  );
}

export default function App() {
  return (
    <CartProvider>
      <Header />
      <main className="layout">
        <section aria-labelledby="products-h">
          <h2 id="products-h">Products</h2>
          <ul className="products">
            {PRODUCTS.map((p) => (
              <ProductCard key={p.id} product={p} />
            ))}
          </ul>
        </section>
        <section aria-labelledby="cart-h">
          <h2 id="cart-h">Your cart</h2>
          <Cart />
        </section>
      </main>
    </CartProvider>
  );
}
```

**`cart.css`**

```css
* { box-sizing: border-box; }
body { font-family: system-ui, sans-serif; margin: 0; color: #111; }

.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px 24px;
  border-bottom: 1px solid #e5e7eb;
}
.header h1 { margin: 0; font-size: 20px; }

.layout {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 32px;
  max-width: 960px;
  margin: 24px auto;
  padding: 0 16px;
}
@media (max-width: 720px) { .layout { grid-template-columns: 1fr; } }

ul { list-style: none; margin: 0; padding: 0; }
button { cursor: pointer; }
button:disabled { cursor: not-allowed; opacity: 0.5; }

.product, .cart-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 10px 0;
  border-bottom: 1px solid #f1f5f9;
}
.product__name { flex: 1; }
.cart-item__info { flex: 1; display: flex; flex-direction: column; font-size: 14px; }
.cart-item__info span { color: #64748b; }
.cart-item__total { min-width: 72px; text-align: right; font-variant-numeric: tabular-nums; }

.qty { display: flex; align-items: center; gap: 8px; }
.qty button { width: 28px; height: 28px; }

.link { background: none; border: 0; color: #2563eb; text-decoration: underline; padding: 0; }

.coupon { margin: 16px 0; }
.coupon__row { display: flex; gap: 8px; margin-top: 4px; }
.coupon__row input { flex: 1; padding: 6px 8px; }
.coupon__applied { color: #15803d; }
.coupon__inactive { color: #b45309; }
.error { color: #dc2626; margin: 4px 0 0; font-size: 14px; }

.summary { margin: 0 0 16px; }
.summary__row { display: flex; justify-content: space-between; padding: 4px 0; }
.summary dt, .summary dd { margin: 0; }
.summary__row--total { border-top: 2px solid #111; margin-top: 6px; padding-top: 8px; font-weight: 700; }
```

</details>

---

## 6. How It Works

- **Single source of truth, minimal.** State holds only `items` and the coupon *code*. `getTotals` derives subtotal / discount / shipping / tax / total each time `state` changes (memoised). Storing totals would create drift bugs (e.g. forgetting to recompute when a coupon's min-spend stops being met).
- **Integer cents.** `0.1 + 0.2 !== 0.3`. Prices are cents; rounding happens explicitly with `Math.round` at the percentage and tax steps; formatting uses `Intl.NumberFormat`.
- **Pure reducer.** Every case returns a new object/array via spread/`map`/`filter`; no `push`, no `item.qty++`. `DECREMENT` maps then filters out `qty <= 0`, so "decrement from 1 removes the row" needs no special branch.
- **Coupon validation lives outside the reducer** (`validateCoupon`) because it must produce an *error message* for the UI; the reducer only stores a code that has already been validated. `getTotals` still defends against ineligibility (items removed later) via `couponActive`.
- **Two contexts** (state vs. actions): `actions` is created once with `useMemo([])` because `dispatch` is stable, so `ProductCard` (wrapped in `memo`) never re-renders on cart updates.
- **Persistence:** lazy initializer `useReducer(reducer, initialState, loadState)` reads storage once; an effect writes on every change. Loading goes through `parseStored`, which validates shape and drops unknown coupons/items — a hand-edited or old-schema value can't crash the app. The `storage` event (fires only in *other* tabs) hydrates cross-tab changes; because writing an identical value doesn't re-fire the event, no infinite loop occurs.
- **Versioned key** (`cart:v1`) lets you change the schema later without breaking users.

---

## 7. Edge Cases

- Add the same product repeatedly → qty increments, capped at `MAX_QTY`; "+" button disabled at the cap.
- Decrement from 1 → item removed. Cart becomes empty → shipping shows 0 and the coupon UI hides (empty state); the stored coupon persists — decide whether `CLEAR`/empty should drop it.
- Coupon `FLAT50` applied, then user removes items so subtotal < $200 → discount becomes 0 and the UI flags it as ineligible (not silently still applied).
- Flat discount larger than subtotal → `Math.min(value, subtotal)` prevents negative totals.
- Invalid / lower-case / padded coupon (`" save10 "`) → trimmed and uppercased before lookup.
- Applying a coupon when another is already applied → this design allows one; the form is hidden until removed (alternatively replace it).
- Free-shipping threshold uses the **post-discount** amount; state this assumption.
- Corrupted/`null` localStorage, private mode throwing on `getItem`/`setItem` → try/catch with fallback to in-memory state.
- Price/rounding: `Math.round` on percent discount; tax computed on discounted amount.
- Two tabs: last write wins; `storage` event syncs.
- Product price changed on the server after the item was saved → persisted price is stale; in a real app re-fetch prices at cart load/checkout.

---

## 8. Accessibility

- Buttons have explicit `aria-label`s ("Increase quantity of Mechanical Keyboard") — "+" and "−" alone are meaningless.
- Totals (`<dl>`) and the header count use `aria-live="polite"` so changes are announced.
- Coupon input has a `<label>`, `aria-invalid`, and `aria-describedby` → error with `role="alert"`.
- Use real `<button>`s and `<form>` (Enter submits coupon) — never clickable `<div>`s.
- Disabled "+" at max quantity; communicate the limit in text if possible.
- When removing an item or applying a coupon (the form unmounts), move focus sensibly (next item / the cart heading / the applied-coupon text) so keyboard users aren't dropped at `<body>` — the reference code omits this for brevity.
- Use semantic lists, headings, and `<dl>` for summary rows.

---

## 9. Performance Considerations

- Split contexts + `memo` on `ProductCard` avoid re-rendering the catalogue on cart updates.
- `useMemo` for `totals` and context values prevents new object identities on unrelated renders. Consumers of the *state* context still re-render when anything in it changes — for large apps, use selector-based stores (Zustand, `use-context-selector`) or split further.
- Key cart rows by `id`, not index.
- `localStorage` is synchronous; writing on every change is fine for a small cart. For high-frequency updates, debounce the save and flush on `pagehide`.
- `Intl.NumberFormat` construction is relatively costly; create the formatter once at module level if profiling shows it (easy optimisation: hoist it out of `formatPrice`).
- Derived data computed in one place (`getTotals`) = one O(n) pass.

---

## 10. Follow-up Questions

<details>
<summary><strong>1. Why `useReducer` over several `useState`s here?</strong></summary>

Updates are multi-step and interdependent (add-or-increment, decrement-or-remove, clamping). A reducer centralises those transitions in a pure, testable function, makes the actions explicit, and gives a stable `dispatch` that doesn't need to be in dependency arrays.

</details>

<details>
<summary><strong>2. How would you support variants (size/colour)?</strong></summary>

Key a cart line by a composite `lineId` (`productId` + `:` + `variantId`). `ADD` matches on `lineId`; rows display variant info. Nothing else changes — all operations take `lineId` instead of `id`.

</details>

<details>
<summary><strong>3. How do you sync the cart with a backend?</strong></summary>

Keep the reducer as the optimistic local state; on each action, send a mutation and reconcile with the server response (server is the source of truth for prices/stock). Handle failure by rolling back (see the optimistic-update question). For guests, merge the local cart into the server cart on login.

</details>

<details>
<summary><strong>4. How would you handle stackable coupons or a priority order?</strong></summary>

Store `coupons: string[]`, define a deterministic application order (e.g. percent before flat), mark exclusive coupons with `stackable: false`, and compute all discounts in `getTotals` sequentially over the running subtotal, capping at 0.

</details>

<details>
<summary><strong>5. Why not store `total` in state?</strong></summary>

It's derivable; storing it creates two sources of truth that can disagree (e.g. after a coupon becomes ineligible). Compute on read with `useMemo`.

</details>

<details>
<summary><strong>6. How would you avoid re-rendering all context consumers when only the item count changes?</strong></summary>

Split into more contexts (items vs. totals), use a store with selectors (`useSyncExternalStore`-based: Zustand/Redux), or lift the badge into its own context fed by `itemCount`. Memoizing children doesn't help if they read the changing context themselves.

</details>

<details>
<summary><strong>7. How would you unit test it?</strong></summary>

Reducer and `getTotals` are pure — test them directly: add twice → qty 2; decrement to zero removes; `FLAT50` below minimum → discount 0; rounding (`$19.99 × 3` with 10% off); `parseStored` with garbage/invalid items. Then a couple of RTL tests for the UI flow.

</details>

<details>
<summary><strong>8. How would you implement undo for "Remove item"?</strong></summary>

Keep the last removed `{item, index}` in state (or a `past` stack in the reducer) and show a toast with an Undo button dispatching `RESTORE` that re-inserts it at its index; clear it after a timeout.

</details>

---

## 11. Common Mistakes

- Using floats for money (`19.99 * 3` → `59.97000000000001`).
- Storing derived values (`total`, `itemCount`) in state.
- Mutating state: `state.items.push(...)`, `item.qty += 1`.
- Duplicating rows instead of incrementing quantity.
- Allowing quantity ≤ 0 or beyond a cap; `NaN` from number inputs.
- Validating coupons inside the reducer (can't surface errors) or never re-validating after the cart changes.
- Reading `localStorage` on every render (`useReducer(reducer, JSON.parse(localStorage...))`) instead of the lazy initializer.
- Not wrapping `JSON.parse` in try/catch → white screen on corrupted storage.
- Putting everything into one context and wondering why the whole page re-renders.
- Index as key for cart rows → wrong row animates/focus jumps on removal.
- Not trimming/uppercasing coupon input.

---

## 12. Interview Tips & Time Plan

Say upfront: *"Money in cents, totals derived, reducer pure."* Build the reducer + totals first (the logic is the heart), then UI, then coupons, then persistence last. If short on time, skip the cross-tab sync and mention it.

| Minute | Activity |
| --- | --- |
| 0–5 | Clarify (coupon rules, tax/shipping, persistence) |
| 5–10 | State shape, actions table, component tree |
| 10–22 | `cartReducer` + `getTotals` (cents, derived) |
| 22–30 | `CartProvider`, `useCart` / `useCartActions` |
| 30–42 | Product list, cart rows, qty controls, summary |
| 42–50 | Coupon form + validation + ineligible handling |
| 50–55 | Persistence with lazy init + safe parse |
| 55–60 | a11y pass, edge cases, follow-ups |

---

## 13. Practice Checklist

- [ ] Reducer with ADD / INCREMENT / DECREMENT / REMOVE / CLEAR — all immutable
- [ ] Money in integer cents with `Intl.NumberFormat`
- [ ] `getTotals` derived with discount, shipping threshold, tax
- [ ] Context split into state + actions; explain why
- [ ] Coupon form with validation messages and removal
- [ ] Coupon re-evaluated when cart changes (min-spend)
- [ ] localStorage persistence with lazy init + safe parse + try/catch
- [ ] Cross-tab sync via the `storage` event
- [ ] Variant-aware line items
- [ ] Undo remove
- [ ] Unit tests for the reducer and `getTotals`
- [ ] Keyboard-only run-through with screen-reader labels
