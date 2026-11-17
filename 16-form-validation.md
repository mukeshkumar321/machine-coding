# Form Validation (Sign-up) — Machine Coding

> **Difficulty:** Medium · **Time:** 60 min · **Asked at / Frequency:** Commonly reported at product companies and fintech (Razorpay, PhonePe, Flipkart, Atlassian) — Very High · **Tier:** 1

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

Build a **sign-up form** (name, email, password, confirm password, accept-terms checkbox) with:

- Per-field validation and error messages
- Errors appear only after the user has *touched* a field (or tried to submit)
- A submit flow: `idle → submitting → success | error`, with a (fake) async API
- A **reusable `useForm` hook** that any form can use

---

## 2. Requirements

### Functional

- Validate on change/blur; show an error only if the field is **touched**.
- On submit: mark all fields touched, block submit if invalid, focus the first invalid field.
- While submitting: disable the submit button, show "Creating account…", ignore double submits.
- Show server errors (generic message and field-level like "Email already registered").
- Track **dirty** fields (value differs from initial); `reset()` restores everything.
- Confirm password must match password.

### Non-functional

- Validation is a **pure function** `values → errors` — independent of React.
- Hook is generic (not sign-up specific).
- Accessible errors (`aria-invalid`, `aria-describedby`, labels).
- No state updates after unmount.

---

## 3. Clarifying Questions to Ask

- Validate on change, on blur, or on submit? *(Default: show after blur/submit, re-validate live afterwards.)*
- Password rules? Show a strength meter?
- Is async validation needed (username/email availability)?
- Can I use a library (Formik / React Hook Form / Zod)? *(Usually "no" in machine coding; mention you would in production.)*
- Should the button be disabled while the form is invalid? *(Recommend no — keep it enabled so submit can reveal errors.)*
- What should happen on success — redirect, message, reset?

---

## 4. Approach & Component Design

**State shape (inside `useForm`)**

| State | Shape | Purpose |
|---|---|---|
| `values` | `{ name, email, password, confirmPassword, terms }` | controlled inputs |
| `touched` | `{ [field]: true }` | show error only after blur/submit |
| `serverErrors` | `{ [field]: string }` | errors returned by the API, cleared when that field changes |
| `status` | `'idle' \| 'submitting' \| 'success' \| 'error'` | submit lifecycle |
| `submitError` | `string` | generic API failure message |

**Derived (never stored):** `errors = validate(values)` (memo), `isValid`, `dirty`, `getError(name)`.

```
SignUpForm
 ├─ useForm({ initialValues, validate, onSubmit })
 ├─ <TextField  label="Name"  inputProps={getFieldProps('name')} error={getError('name')} />
 ├─ <TextField  ... email / password / confirmPassword />
 ├─ <CheckboxField ... terms />
 └─ <button type="submit" disabled={status === 'submitting'}>
```

```
 idle ──submit(valid)──► submitting ──ok──► success
   ▲  └─submit(invalid)─► (stay idle, all touched)   └─fail─► error ──edit field──► idle
```

---

## 5. Reference Solution

<details>
<summary><strong>Show Full Code (validators.js, useForm.js, Fields.jsx, SignUpForm.jsx, App.jsx, styles.css)</strong></summary>

```jsx
// validators.js  — pure, framework independent
const EMAIL_RE = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;

export const initialValues = {
  name: '',
  email: '',
  password: '',
  confirmPassword: '',
  terms: false,
};

// key order == visual order, so the first key is the first invalid field
export function validateSignUp(v) {
  const e = {};

  if (!v.name.trim()) e.name = 'Name is required';
  else if (v.name.trim().length < 2) e.name = 'Name must be at least 2 characters';

  if (!v.email) e.email = 'Email is required';
  else if (!EMAIL_RE.test(v.email)) e.email = 'Enter a valid email address';

  if (!v.password) e.password = 'Password is required';
  else if (v.password.length < 8) e.password = 'Password must be at least 8 characters';
  else if (!/[A-Z]/.test(v.password) || !/\d/.test(v.password))
    e.password = 'Use at least one uppercase letter and one number';

  if (!v.confirmPassword) e.confirmPassword = 'Please confirm your password';
  else if (v.confirmPassword !== v.password) e.confirmPassword = 'Passwords do not match';

  if (!v.terms) e.terms = 'You must accept the terms';

  return e;
}
```

```jsx
// api.js — fake backend
export function signUp(values) {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      if (values.email.toLowerCase() === 'taken@example.com') {
        const err = new Error('Please fix the highlighted fields.');
        err.fieldErrors = { email: 'This email is already registered' };
        reject(err);
      } else if (values.name.toLowerCase() === 'fail') {
        reject(new Error('Server error. Please try again.'));
      } else {
        resolve({ id: Date.now() });
      }
    }, 1000);
  });
}
```

```jsx
// useForm.js
import { useCallback, useEffect, useMemo, useRef, useState } from 'react';

export function useForm({ initialValues, validate, onSubmit }) {
  const initialRef = useRef(initialValues);
  const mountedRef = useRef(true);
  const submittingRef = useRef(false);

  const [values, setValues] = useState(initialValues);
  const [touched, setTouched] = useState({});
  const [serverErrors, setServerErrors] = useState({});
  const [status, setStatus] = useState('idle');
  const [submitError, setSubmitError] = useState('');

  useEffect(() => {
    mountedRef.current = true;
    return () => {
      mountedRef.current = false;
    };
  }, []);

  // `validate` must be a stable reference (module-level function or useCallback)
  const errors = useMemo(() => validate(values), [values, validate]);
  const isValid = Object.keys(errors).length === 0;

  const dirty = useMemo(() => {
    const init = initialRef.current;
    return Object.keys(values).reduce((acc, k) => {
      if (values[k] !== init[k]) acc[k] = true;
      return acc;
    }, {});
  }, [values]);
  const isDirty = Object.keys(dirty).length > 0;

  const setFieldValue = useCallback((name, value) => {
    setValues((v) => ({ ...v, [name]: value }));
    setServerErrors((s) => {
      if (!s[name]) return s;
      const { [name]: _removed, ...rest } = s;
      return rest;
    });
    setStatus((s) => (s === 'error' ? 'idle' : s));
  }, []);

  const handleChange = useCallback(
    (e) => {
      const { name, type, value, checked } = e.target;
      setFieldValue(name, type === 'checkbox' ? checked : value);
    },
    [setFieldValue]
  );

  const handleBlur = useCallback((e) => {
    const { name } = e.target;
    setTouched((t) => (t[name] ? t : { ...t, [name]: true }));
  }, []);

  const getFieldProps = (name) => {
    const isCheckbox = typeof initialRef.current[name] === 'boolean';
    return {
      name,
      onChange: handleChange,
      onBlur: handleBlur,
      ...(isCheckbox ? { checked: values[name] } : { value: values[name] }),
    };
  };

  const getError = (name) =>
    serverErrors[name] || (touched[name] ? errors[name] : undefined);

  const handleSubmit = async (e) => {
    e.preventDefault();
    if (submittingRef.current) return;

    setTouched(Object.fromEntries(Object.keys(values).map((k) => [k, true])));

    if (!isValid) {
      const firstInvalid = Object.keys(errors)[0];
      e.currentTarget.elements.namedItem(firstInvalid)?.focus();
      return;
    }

    submittingRef.current = true;
    setStatus('submitting');
    setSubmitError('');
    try {
      await onSubmit(values);
      if (mountedRef.current) setStatus('success');
    } catch (err) {
      if (mountedRef.current) {
        setServerErrors(err.fieldErrors ?? {});
        setSubmitError(err.message || 'Something went wrong');
        setStatus('error');
      }
    } finally {
      submittingRef.current = false;
    }
  };

  const reset = useCallback(() => {
    setValues(initialRef.current);
    setTouched({});
    setServerErrors({});
    setSubmitError('');
    setStatus('idle');
  }, []);

  return {
    values, errors, touched, dirty, isDirty, isValid,
    status, submitError,
    getFieldProps, getError, setFieldValue, handleSubmit, reset,
  };
}
```

```jsx
// Fields.jsx
import { useId } from 'react';

export function TextField({ label, type = 'text', error, hint, inputProps }) {
  const id = useId();
  const errorId = `${id}-error`;
  const hintId = `${id}-hint`;
  const describedBy = [error && errorId, hint && hintId].filter(Boolean).join(' ');

  return (
    <div className={`field${error ? ' invalid' : ''}`}>
      <label htmlFor={id}>{label}</label>
      <input
        id={id}
        type={type}
        aria-invalid={Boolean(error)}
        aria-describedby={describedBy || undefined}
        {...inputProps}
      />
      {hint && !error && <p id={hintId} className="hint">{hint}</p>}
      {error && (
        <p id={errorId} className="error" role="alert">
          {error}
        </p>
      )}
    </div>
  );
}

export function CheckboxField({ label, error, inputProps }) {
  const id = useId();
  const errorId = `${id}-error`;

  return (
    <div className={`field checkbox${error ? ' invalid' : ''}`}>
      <label htmlFor={id}>
        <input
          id={id}
          type="checkbox"
          aria-invalid={Boolean(error)}
          aria-describedby={error ? errorId : undefined}
          {...inputProps}
        />
        {label}
      </label>
      {error && (
        <p id={errorId} className="error" role="alert">
          {error}
        </p>
      )}
    </div>
  );
}
```

```jsx
// SignUpForm.jsx
import { useForm } from './useForm';
import { TextField, CheckboxField } from './Fields';
import { initialValues, validateSignUp } from './validators';
import { signUp } from './api';

export default function SignUpForm() {
  const {
    getFieldProps, getError, handleSubmit, reset,
    status, submitError, isDirty, values,
  } = useForm({ initialValues, validate: validateSignUp, onSubmit: signUp });

  if (status === 'success') {
    return (
      <div className="card" role="status">
        <h2>Welcome, {values.name}! 🎉</h2>
        <p>Your account has been created.</p>
        <button onClick={reset}>Create another</button>
      </div>
    );
  }

  const submitting = status === 'submitting';

  return (
    <form className="card" onSubmit={handleSubmit} noValidate>
      <h2>Create account</h2>

      <TextField
        label="Name"
        error={getError('name')}
        inputProps={{ ...getFieldProps('name'), autoComplete: 'name' }}
      />
      <TextField
        label="Email"
        type="email"
        error={getError('email')}
        inputProps={{ ...getFieldProps('email'), autoComplete: 'email' }}
      />
      <TextField
        label="Password"
        type="password"
        hint="8+ characters, one uppercase letter, one number"
        error={getError('password')}
        inputProps={{ ...getFieldProps('password'), autoComplete: 'new-password' }}
      />
      <TextField
        label="Confirm password"
        type="password"
        error={getError('confirmPassword')}
        inputProps={{ ...getFieldProps('confirmPassword'), autoComplete: 'new-password' }}
      />
      <CheckboxField
        label="I accept the terms and conditions"
        error={getError('terms')}
        inputProps={getFieldProps('terms')}
      />

      {status === 'error' && submitError && (
        <p className="error banner" role="alert">{submitError}</p>
      )}

      <div className="actions">
        <button type="submit" disabled={submitting}>
          {submitting ? 'Creating account…' : 'Sign up'}
        </button>
        <button type="button" onClick={reset} disabled={!isDirty || submitting}>
          Reset
        </button>
      </div>
    </form>
  );
}
```

```jsx
// App.jsx
import SignUpForm from './SignUpForm';
import './styles.css';

export default function App() {
  return (
    <main>
      <SignUpForm />
    </main>
  );
}
```

```css
/* styles.css */
body { font-family: system-ui, sans-serif; background: #f5f6f8; }
.card { max-width: 380px; margin: 40px auto; padding: 24px; background: #fff; border-radius: 10px; box-shadow: 0 2px 10px rgba(0,0,0,.08); }
.field { display: flex; flex-direction: column; margin-bottom: 14px; }
.field label { font-size: 14px; margin-bottom: 4px; font-weight: 600; }
.field input:not([type='checkbox']) { padding: 8px 10px; border: 1px solid #bbb; border-radius: 6px; font-size: 14px; }
.field input:focus-visible { outline: 2px solid #2563eb; outline-offset: 1px; }
.field.invalid input:not([type='checkbox']) { border-color: #d92d20; }
.field.checkbox label { display: flex; gap: 8px; align-items: center; font-weight: 400; }
.hint { margin: 4px 0 0; font-size: 12px; color: #667085; }
.error { margin: 4px 0 0; font-size: 12px; color: #d92d20; }
.error.banner { padding: 8px 10px; background: #fef3f2; border-radius: 6px; font-size: 13px; }
.actions { display: flex; gap: 8px; }
.actions button { padding: 9px 16px; border-radius: 6px; border: 1px solid #ccc; background: #fff; cursor: pointer; }
.actions button[type='submit'] { background: #2563eb; border-color: #2563eb; color: #fff; }
.actions button:disabled { opacity: .6; cursor: not-allowed; }
```

</details>

---

## 6. How It Works

- **Errors are derived** — `useMemo(() => validate(values))`. Storing errors in state causes "validation one keystroke behind" bugs. The *visibility* of an error is a separate concern: `touched[name] && errors[name]`.
- **`touched`** flips on blur; on submit we mark **every** field touched so all errors appear at once. That is the difference between touched (user interacted) and **dirty** (value ≠ initial).
- **Controlled inputs through `getFieldProps(name)`** — spreads `name/value/onChange/onBlur`. One `handleChange` handles every field via `e.target.name`; checkboxes use `checked`.
- **Functional updates** `setValues(v => ({...v, [name]: value}))` — no stale closures, no mutation.
- **Submit lifecycle** — a `submittingRef` guards double submit even before React re-renders; `mountedRef` prevents setting state after unmount; `finally` always releases the guard.
- **Server errors** live apart from client errors and are cleared when the user edits *that* field, so a stale "email taken" doesn't stick.
- **Focus management** — focus the first invalid control via `form.elements.namedItem(name)`; object key order in `validate` equals the visual order.
- `validate` must be referentially stable (defined at module level) or `useMemo` re-runs every render.

---

## 7. Edge Cases

- Submit pressed with an untouched empty form → all errors appear, focus goes to the first.
- Changing **password** after confirm was filled → confirm's error must update (it does, because errors derive from all values).
- Whitespace-only name; leading/trailing spaces in email (trim before validating/submitting).
- Double click / Enter pressed twice on submit.
- Component unmounted while the request is in flight.
- Server returns a field error → show under the field, clear on edit.
- Browser autofill fires `change` without blur — values update, touched doesn't; errors still gated until blur/submit.
- Reset after success / after error; reset clears touched and server errors.
- `type="email"` native validation bubbles competing with ours → use `noValidate`.

---

## 8. Accessibility

- Every input has a `<label htmlFor>` (via `useId`); placeholders are not labels.
- `aria-invalid` on invalid fields, `aria-describedby` linking to the error/hint.
- Errors use `role="alert"` (announced when they appear); success uses `role="status"`.
- Don't convey errors by colour alone — message text is always rendered.
- Keep the submit button enabled when invalid (a disabled button gives no feedback); disable only while submitting.
- Move focus to the first invalid field on failed submit.
- `autoComplete` attributes help password managers and autofill.

---

## 9. Performance Considerations

- Validation is O(fields) and cheap — running it on every keystroke is fine. For expensive/async rules (username availability) **debounce** and cancel stale requests.
- All fields re-render on each keystroke because state is in one parent. Fine for small forms; for large forms wrap fields in `React.memo` (needs stable props — `getFieldProps` returns new objects, so pass primitives) or use uncontrolled inputs (React Hook Form approach).
- `useCallback` for handlers keeps memoised children stable.
- Avoid creating the `validate` function inline.

---

## 10. Follow-up Questions

<details>
<summary><strong>1. Controlled vs uncontrolled forms — trade-offs?</strong></summary>

Controlled: every keystroke goes through React state → easy live validation/derived UI but more re-renders. Uncontrolled (refs/`FormData`, React Hook Form): fewer re-renders, simpler for big forms, but live UI needs subscriptions. Pick controlled for small forms.

</details>

<details>
<summary><strong>2. How would you add async validation (email availability)?</strong></summary>

Run on blur or debounced change. Keep `validating` status per field, use `AbortController` to cancel stale calls, ignore results whose value no longer matches current value, and treat "pending" as not-yet-valid on submit. Always re-validate on the server on submit.

</details>

<details>
<summary><strong>3. How do you validate on change vs blur vs submit?</strong></summary>

Best UX: **reward early, punish late** — show errors after first blur, then update live while typing; show success/valid state immediately. Implement via `touched`; the "mode" can be a hook option (`'onChange' | 'onBlur' | 'onSubmit'`).

</details>

<details>
<summary><strong>4. How would you support schema validation (Zod/Yup)?</strong></summary>

The hook only needs `validate(values) → errors`. Wrap the schema: `v => { const r = schema.safeParse(v); return r.success ? {} : flatten(r.error) }`. That's why validation is a pure function injected into the hook.

</details>

<details>
<summary><strong>5. How do you handle dynamic / array fields (add another phone number)?</strong></summary>

Store arrays in `values` and use paths (`phones.0.number`) with a `setIn` helper that clones along the path immutably; errors/touched mirror the same shape.

</details>

<details>
<summary><strong>6. Prevent the user from leaving with unsaved changes?</strong></summary>

Use `isDirty` to register a `beforeunload` listener (add in `useEffect`, remove in cleanup) and the router's blocker for in-app navigation.

</details>

<details>
<summary><strong>7. How would you add a password strength meter?</strong></summary>

A pure `scorePassword(pw)` (length, character classes, common-password list) → 0–4, derived from `values.password` in render; show with `<meter>`/progress bar and `aria-live="polite"` text. Don't block on it — it's guidance.

</details>

<details>
<summary><strong>8. Why not store `errors` in state?</strong></summary>

Two sources of truth: errors drift from values, you need effects to sync them, and you get an extra render. Derive them; only store what can't be derived (touched, server errors, status).

</details>

---

## 11. Common Mistakes

- Showing errors immediately on first render (before the user touched anything).
- Storing `errors` in state and syncing with `useEffect`.
- Mutating state (`values[name] = …`) or using `setValues({...values, …})` inside async code (stale).
- Disabling submit while invalid, so users can't discover what's wrong.
- No `preventDefault`, or forgetting `noValidate`.
- Not guarding double submit; no loading state.
- Setting state after unmount; forgetting to clear server errors.
- `validate` defined inline → `useMemo` recomputes every render.
- Confirm-password only validated when *its own* field changes.
- Checkbox uses `value` instead of `checked`.

---

## 12. Interview Tips & Time Plan

Say the words **touched**, **dirty**, **derived errors**, **pure validators** early — they signal experience. Do the form without the hook first, then extract.

| Minutes | Task |
|---|---|
| 0–5 | Clarify, define state shape and submit state machine |
| 5–15 | Controlled inputs + `validateSignUp` pure function |
| 15–25 | touched logic, show errors, submit marks all touched |
| 25–35 | Async submit, loading/success/error, double-submit guard |
| 35–47 | Extract `useForm`, `getFieldProps`, reusable `TextField` |
| 47–55 | a11y (labels, aria-invalid, focus first error), server field errors |
| 55–60 | Edge cases, walk through reset, discuss async validation |

---

## 13. Practice Checklist

- [ ] Validators are pure and tested with a few inputs
- [ ] Errors only visible after blur or submit
- [ ] Submit marks all touched and focuses first invalid field
- [ ] Submitting state disables button, blocks double submit
- [ ] Server field error shown and cleared on edit
- [ ] `dirty` vs `touched` explained
- [ ] Confirm password re-validates when password changes
- [ ] `useForm` is generic (works for a login form too)
- [ ] `aria-invalid` + `aria-describedby` + labels
- [ ] No state update after unmount
