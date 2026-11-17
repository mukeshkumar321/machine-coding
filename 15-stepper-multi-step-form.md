# Stepper / Multi-Step Form — Machine Coding

> **Difficulty:** Medium | **Time:** 45–60 min | **Frequency:** ⭐⭐⭐⭐

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

Build a **multi-step (wizard) form** — e.g. a sign-up flow with 3 data-entry steps (**Account → Address → Plan**) and a final **Review** step. The user moves with **Next / Back**; each step is **validated before advancing**; values entered earlier are **preserved** when navigating back and forth; a **progress indicator** shows where the user is and lets them jump to earlier steps. The final step shows a summary and submits.

---

## 2. Requirements

### Functional

- Steps: Account (name, email, password) → Address (street, city, zip) → Plan (plan select, accept terms) → Review & submit.
- **Next** validates the current step only; on errors it stays, shows messages, and focuses the first invalid field.
- **Back** never validates and never loses data.
- **Progress indicator**: completed / current / upcoming states, `aria-current="step"`, plus a progress bar. Clicking an earlier step goes back; clicking a later step only works if all steps in between are valid.
- Enter key advances (it's a real `<form>`).
- Review step lists all values (password masked) with "Edit" shortcuts; **Submit** re-validates everything, shows a loading state, then a success screen.

### Non-functional

- Config-driven steps (add a step/field = add data, not new logic).
- One source of truth for values (parent) — steps are stateless views.
- Prevent double submission.
- Accessible errors, focus management, keyboard friendly.
- Plain CSS, no form library.

---

## 3. Clarifying Questions to Ask

- Is the flow strictly linear, or can users jump around?
- When should validation run: on Next only, on blur, on change?
- Should data persist across page refresh (sessionStorage/draft saving)?
- Are steps conditional (e.g. "Company details" only for the Team plan)?
- Submit once at the end, or save per step to the server?
- Do we need a progress bar, step labels, or both? Is "Edit" on review expected?
- Validation rules: sync only, or async (email availability)?

---

## 4. Approach & Component Design

**Key decision:** all form values live in the **parent** (`MultiStepForm`). Each step is just a view over `values` filtered by that step's field list. Because the parent never unmounts, going Back and then Next cannot lose data. (If each step held its own `useState`, unmounting a step would wipe it.)

**State shape**

```js
stepIndex: 0                     // which step is visible
values:    { name:'', email:'', password:'', street:'', city:'', zip:'', plan:'', terms:false }
errors:    { email: 'Enter a valid email' }   // only for the visible step
status:    'idle' | 'submitting' | 'success' | 'error'
attempt:   0                     // bump to trigger "focus first invalid field"
```

**Step config (data, not code)**

```js
{ id:'account', title:'Account', fields:[{ name, label, type, validate }] }
```

**Component tree**

```
<MultiStepForm>                       owns stepIndex / values / errors / status
  ├─ <StepIndicator />                nav + ol, progress bar, click to jump
  └─ <form noValidate>
       ├─ <h2 ref tabIndex=-1>        focus target on step change
       ├─ <Field /> x n               (or <Review /> on the last step)
       └─ [Back] [Next | Submit]
```

**Navigation logic (one function)**

```
goTo(target):
  target <= current  → clear errors, move (no validation)
  target >  current  → validate steps current … target-1 in order;
                       first failing step → show it + errors + focus; else move
```

| Action | Validation | Moves to |
| --- | --- | --- |
| Next / Enter | current step | `current + 1` |
| Back | none | `current - 1` |
| Click indicator ≤ current | none | clicked step |
| Click indicator > current | all steps in between | clicked step or first invalid |
| Submit | all data steps | success or first invalid |

---

## 5. Reference Solution

<details>
<summary><strong>Show Full Code</strong></summary>

**`steps.js`**

```js
const required = (label) => (v) => (String(v ?? '').trim() ? '' : `${label} is required`);

export const STEPS = [
  {
    id: 'account',
    title: 'Account',
    fields: [
      { name: 'name', label: 'Full name', type: 'text', autoComplete: 'name', validate: required('Full name') },
      {
        name: 'email',
        label: 'Email',
        type: 'email',
        autoComplete: 'email',
        validate: (v) => {
          if (!v.trim()) return 'Email is required';
          return /^\S+@\S+\.\S+$/.test(v.trim()) ? '' : 'Enter a valid email address';
        },
      },
      {
        name: 'password',
        label: 'Password',
        type: 'password',
        autoComplete: 'new-password',
        validate: (v) => (v.length >= 8 ? '' : 'Password must be at least 8 characters'),
      },
    ],
  },
  {
    id: 'address',
    title: 'Address',
    fields: [
      { name: 'street', label: 'Street address', type: 'text', autoComplete: 'street-address', validate: required('Street address') },
      { name: 'city', label: 'City', type: 'text', autoComplete: 'address-level2', validate: required('City') },
      {
        name: 'zip',
        label: 'ZIP code',
        type: 'text',
        autoComplete: 'postal-code',
        validate: (v) => (/^\d{5}$/.test(v.trim()) ? '' : 'ZIP must be 5 digits'),
      },
    ],
  },
  {
    id: 'plan',
    title: 'Plan',
    fields: [
      {
        name: 'plan',
        label: 'Plan',
        type: 'select',
        options: [
          { value: '', label: 'Select a plan…' },
          { value: 'free', label: 'Free' },
          { value: 'pro', label: 'Pro — $12/mo' },
          { value: 'team', label: 'Team — $30/mo' },
        ],
        validate: (v) => (v ? '' : 'Choose a plan'),
      },
      {
        name: 'terms',
        label: 'I accept the terms and conditions',
        type: 'checkbox',
        validate: (v) => (v ? '' : 'You must accept the terms'),
      },
    ],
  },
];

export const REVIEW_STEP = { id: 'review', title: 'Review', fields: [] };
export const ALL_STEPS = [...STEPS, REVIEW_STEP];

export const initialValues = {
  name: '', email: '', password: '',
  street: '', city: '', zip: '',
  plan: '', terms: false,
};

export function validateStep(step, values) {
  const errors = {};
  for (const field of step.fields) {
    const message = field.validate(values[field.name]);
    if (message) errors[field.name] = message;
  }
  return errors;
}

export const fakeSubmit = (values) =>
  new Promise((resolve) => setTimeout(() => resolve({ ok: true, id: values.email }), 900));
```

**`MultiStepForm.jsx`**

```jsx
import { useEffect, useRef, useState } from 'react';
import { ALL_STEPS, STEPS, fakeSubmit, initialValues, validateStep } from './steps';
import StepIndicator from './StepIndicator';
import Field from './Field';
import './stepper.css';

export default function MultiStepForm() {
  const [stepIndex, setStepIndex] = useState(0);
  const [values, setValues] = useState(initialValues);
  const [errors, setErrors] = useState({});
  const [attempt, setAttempt] = useState(0); // bumped whenever validation fails
  const [status, setStatus] = useState('idle');

  const formRef = useRef(null);
  const headingRef = useRef(null);
  const prevStep = useRef(stepIndex);

  const step = ALL_STEPS[stepIndex];
  const isReview = step.id === 'review';
  const submitting = status === 'submitting';

  // Focus the heading after the new step has rendered, so it's read with its new text.
  // Comparing with the previous index (not a "first run" flag) keeps this StrictMode-safe.
  useEffect(() => {
    if (prevStep.current !== stepIndex) {
      prevStep.current = stepIndex;
      headingRef.current?.focus();
    }
  }, [stepIndex]);

  // Declared after the effect above, so on a failed jump the invalid field wins the focus.
  useEffect(() => {
    if (attempt > 0) formRef.current?.querySelector('[aria-invalid="true"]')?.focus();
  }, [attempt]);

  const handleChange = (name, value) => {
    setValues((prev) => ({ ...prev, [name]: value }));
    setErrors((prev) => {
      if (!prev[name]) return prev;
      const next = { ...prev };
      delete next[name]; // clear the message as soon as the user edits the field
      return next;
    });
  };

  const handleBlur = (field) => {
    const message = field.validate(values[field.name]);
    if (message) setErrors((prev) => ({ ...prev, [field.name]: message }));
  };

  const failAt = (index, stepErrors) => {
    setStepIndex(index);
    setErrors(stepErrors);
    setAttempt((n) => n + 1);
  };

  const goTo = (target) => {
    if (submitting || target === stepIndex || target < 0 || target >= ALL_STEPS.length) return;

    if (target > stepIndex) {
      // Moving forward: every step we'd skip over must be valid.
      for (let i = stepIndex; i < target; i++) {
        const stepErrors = validateStep(ALL_STEPS[i], values);
        if (Object.keys(stepErrors).length > 0) return failAt(i, stepErrors);
      }
    }
    setErrors({});
    setStepIndex(target);
  };

  const submit = async () => {
    if (submitting) return;
    for (let i = 0; i < STEPS.length; i++) {
      const stepErrors = validateStep(STEPS[i], values);
      if (Object.keys(stepErrors).length > 0) return failAt(i, stepErrors);
    }
    setStatus('submitting');
    try {
      await fakeSubmit(values);
      setStatus('success');
    } catch {
      setStatus('error');
    }
  };

  const handleSubmit = (e) => {
    e.preventDefault(); // also fires on Enter inside any input
    if (isReview) submit();
    else goTo(stepIndex + 1);
  };

  const reset = () => {
    setValues(initialValues);
    setErrors({});
    setStepIndex(0);
    setStatus('idle');
  };

  if (status === 'success') {
    return (
      <div className="stepper" role="status">
        <h2>You're all set, {values.name.split(' ')[0]}! 🎉</h2>
        <p>Your {values.plan} plan is active. A confirmation was sent to {values.email}.</p>
        <button type="button" onClick={reset}>
          Start over
        </button>
      </div>
    );
  }

  return (
    <div className="stepper">
      <StepIndicator steps={ALL_STEPS} current={stepIndex} onSelect={goTo} />

      <form ref={formRef} onSubmit={handleSubmit} noValidate>
        <h2 ref={headingRef} tabIndex={-1} className="stepper__heading">
          Step {stepIndex + 1} of {ALL_STEPS.length}: {step.title}
        </h2>

        {isReview ? (
          <Review values={values} onEdit={goTo} />
        ) : (
          step.fields.map((field) => (
            <Field
              key={field.name}
              field={field}
              value={values[field.name]}
              error={errors[field.name]}
              onChange={handleChange}
              onBlur={handleBlur}
            />
          ))
        )}

        {status === 'error' && (
          <p role="alert" className="error">
            Something went wrong. Please try again.
          </p>
        )}

        <div className="actions">
          <button type="button" onClick={() => goTo(stepIndex - 1)} disabled={stepIndex === 0 || submitting}>
            Back
          </button>
          {isReview ? (
            <button key="submit" type="submit" className="primary" disabled={submitting}>
              {submitting ? 'Submitting…' : 'Submit'}
            </button>
          ) : (
            <button key="next" type="submit" className="primary">
              Next
            </button>
          )}
        </div>
      </form>
    </div>
  );
}

function Review({ values, onEdit }) {
  return (
    <div>
      {STEPS.map((s, index) => (
        <section key={s.id} className="review__section">
          <h3>
            {s.title}{' '}
            <button type="button" className="link" onClick={() => onEdit(index)}>
              Edit <span className="sr-only">{s.title}</span>
            </button>
          </h3>
          <dl>
            {s.fields.map((f) => {
              const raw = values[f.name];
              let display = raw;
              if (f.type === 'password') display = '•'.repeat(raw.length);
              if (f.type === 'checkbox') display = raw ? 'Yes' : 'No';
              if (f.type === 'select') display = f.options.find((o) => o.value === raw)?.label;
              return (
                <div key={f.name} className="review__row">
                  <dt>{f.type === 'checkbox' ? 'Terms accepted' : f.label}</dt>
                  <dd>{display || '—'}</dd>
                </div>
              );
            })}
          </dl>
        </section>
      ))}
    </div>
  );
}
```

**`Field.jsx`**

```jsx
export default function Field({ field, value, error, onChange, onBlur }) {
  const id = `field-${field.name}`;
  const errorId = `${id}-error`;

  const shared = {
    id,
    name: field.name,
    'aria-invalid': error ? 'true' : undefined,
    'aria-describedby': error ? errorId : undefined,
    onBlur: () => onBlur(field),
  };

  let control;
  if (field.type === 'select') {
    control = (
      <select {...shared} value={value} onChange={(e) => onChange(field.name, e.target.value)}>
        {field.options.map((o) => (
          <option key={o.value} value={o.value}>
            {o.label}
          </option>
        ))}
      </select>
    );
  } else if (field.type === 'checkbox') {
    control = (
      <input
        {...shared}
        type="checkbox"
        checked={value}
        onChange={(e) => onChange(field.name, e.target.checked)}
      />
    );
  } else {
    control = (
      <input
        {...shared}
        type={field.type}
        autoComplete={field.autoComplete}
        value={value}
        onChange={(e) => onChange(field.name, e.target.value)}
      />
    );
  }

  return (
    <div className={`field${field.type === 'checkbox' ? ' field--checkbox' : ''}`}>
      {field.type === 'checkbox' ? (
        <label htmlFor={id}>
          {control} {field.label}
        </label>
      ) : (
        <>
          <label htmlFor={id}>{field.label}</label>
          {control}
        </>
      )}
      {error && (
        <p id={errorId} role="alert" className="error">
          {error}
        </p>
      )}
    </div>
  );
}
```

**`StepIndicator.jsx`**

```jsx
export default function StepIndicator({ steps, current, onSelect }) {
  const percent = Math.round((current / (steps.length - 1)) * 100);

  return (
    <nav aria-label="Progress">
      <div
        className="progress"
        role="progressbar"
        aria-label="Form completion"
        aria-valuemin={0}
        aria-valuemax={100}
        aria-valuenow={percent}
      >
        <div className="progress__fill" style={{ width: `${percent}%` }} />
      </div>

      <ol className="steps">
        {steps.map((s, i) => {
          const state = i < current ? 'complete' : i === current ? 'current' : 'upcoming';
          return (
            <li key={s.id} className={`steps__item steps__item--${state}`}>
              <button
                type="button"
                className="steps__button"
                aria-current={i === current ? 'step' : undefined}
                onClick={() => onSelect(i)}
              >
                <span className="steps__marker" aria-hidden="true">
                  {state === 'complete' ? '✓' : i + 1}
                </span>
                <span className="steps__label">{s.title}</span>
                {state === 'complete' && <span className="sr-only">, completed</span>}
              </button>
            </li>
          );
        })}
      </ol>
    </nav>
  );
}
```

**`App.jsx`**

```jsx
import MultiStepForm from './MultiStepForm';

export default function App() {
  return <MultiStepForm />;
}
```

**`stepper.css`**

```css
* { box-sizing: border-box; }
body { font-family: system-ui, sans-serif; margin: 0; color: #111; }

.stepper { max-width: 520px; margin: 32px auto; padding: 0 16px; }

.progress { height: 6px; background: #e2e8f0; border-radius: 3px; overflow: hidden; margin-bottom: 16px; }
.progress__fill { height: 100%; background: #2563eb; transition: width 250ms ease; }

.steps { display: flex; justify-content: space-between; list-style: none; margin: 0 0 24px; padding: 0; }
.steps__button {
  display: flex; flex-direction: column; align-items: center; gap: 6px;
  background: none; border: 0; cursor: pointer; font: inherit; color: #64748b; padding: 4px 8px;
}
.steps__marker {
  width: 32px; height: 32px; border-radius: 50%;
  display: grid; place-items: center;
  border: 2px solid #cbd5e1; background: #fff; font-weight: 600;
}
.steps__item--current .steps__button { color: #111; font-weight: 600; }
.steps__item--current .steps__marker { border-color: #2563eb; color: #2563eb; }
.steps__item--complete .steps__marker { background: #2563eb; border-color: #2563eb; color: #fff; }
.steps__button:focus-visible { outline: 3px solid #93c5fd; border-radius: 6px; }

.stepper__heading { font-size: 20px; margin: 0 0 16px; }
.stepper__heading:focus { outline: none; }

.field { display: flex; flex-direction: column; gap: 4px; margin-bottom: 16px; }
.field label { font-weight: 600; font-size: 14px; }
.field input:not([type='checkbox']), .field select {
  padding: 10px 12px; font-size: 16px; border: 1px solid #cbd5e1; border-radius: 6px;
}
.field input:focus-visible, .field select:focus-visible { outline: 3px solid #93c5fd; border-color: #2563eb; }
.field [aria-invalid='true'] { border-color: #dc2626; }
.field--checkbox label { display: flex; gap: 8px; align-items: center; font-weight: 400; }

.error { color: #dc2626; font-size: 14px; margin: 0; }

.actions { display: flex; justify-content: space-between; margin-top: 24px; }
.actions button { padding: 10px 20px; font-size: 16px; border-radius: 6px; border: 1px solid #cbd5e1; background: #fff; cursor: pointer; }
.actions .primary { background: #2563eb; border-color: #2563eb; color: #fff; }
.actions button:disabled { opacity: 0.5; cursor: not-allowed; }

.review__section h3 { margin: 16px 0 4px; font-size: 16px; display: flex; gap: 8px; align-items: baseline; }
.review__row { display: flex; justify-content: space-between; padding: 4px 0; border-bottom: 1px solid #f1f5f9; }
.review__row dt { color: #64748b; }
.review__row dd { margin: 0; text-align: right; }
dl { margin: 0; }

.link { background: none; border: 0; color: #2563eb; text-decoration: underline; cursor: pointer; font-size: 14px; padding: 0; }

.sr-only {
  position: absolute; width: 1px; height: 1px; overflow: hidden;
  clip: rect(0 0 0 0); white-space: nowrap;
}
```

</details>

---

## 6. How It Works

- **Lifted state = preserved state.** `values` lives in `MultiStepForm`. Switching steps only changes `stepIndex`; the inputs for a step are controlled by `values[name]`, so Back/Next never lose data.
- **Config-driven.** `STEPS` describes fields and their `validate` functions. `validateStep(step, values)` is a pure function returning `{ field: message }`; the UI is a loop over `step.fields`. Adding a step is a data change.
- **Single navigation function.** `goTo(target)` handles Next, Back, indicator clicks and "Edit" from review. Forward moves validate each skipped step in order and stop at the first failure (`failAt` jumps *to that step* and shows its errors) — which also fixes the sneaky bug where a user goes back, breaks an earlier field, then clicks a later step in the indicator.
- **Errors only for the visible step**, cleared on navigation, and cleared per-field as the user edits it. `onBlur` validates a field for early feedback.
- **Focus management.** When `stepIndex` changes, an effect focuses the `<h2 tabIndex={-1}>` *after* the new step has rendered (so screen readers announce "Step 2 of 4: Address", not the old text). On failure, a counter `attempt` triggers a second effect which, *after* errors have rendered, focuses the first `[aria-invalid="true"]` element (declared later, so it wins when a failed jump also changes step). Using a counter (rather than a boolean) makes repeated failed attempts re-focus.
- **Real `<form>` with `noValidate`.** Enter submits → `handleSubmit` → Next or Submit. We use our own validation messages instead of the browser's tooltips.
- **Keyed Next/Submit buttons** (defensive) so React mounts a fresh element instead of mutating the Next node into the Submit node. Here both are `type="submit"` so it's harmless, but if Next were a `type="button"` with an `onClick` and React flipped that same node to `type="submit"` during the click, the browser's default action would submit the form immediately. Keys make that impossible.
- **Submit** re-validates *all* data steps (never trust that earlier steps are still valid), guards against double submission with `status` (and `goTo` ignores navigation while submitting), and handles failure with an error message.
- **Progress** is derived: `percent = current / (n-1)`, no extra state.

---

## 7. Edge Cases

- Press Enter in the first field → goes to the next step (validates first).
- User goes Back, clears the email, then jumps forward by clicking the last step in the indicator → validation catches it and returns them to the Account step.
- User edits a field on a step they already passed (via Edit/Back) → value is updated; later steps unaffected unless dependent.
- Double-click on Submit → `submitting` guard + disabled button → one request.
- Submit fails → status `error`, form data intact, retry possible.
- Whitespace-only values → `trim()` rejects them.
- Password shown masked on review; never render it in plain text.
- Browser autofill → React still fires `onChange`; `autoComplete` attributes help it target the right fields.
- Clicking the indicator for the current step is a no-op (doesn't wipe errors).
- Refreshing loses data (by design here) — see follow-up for sessionStorage drafts.
- Conditional steps (e.g. company info only for "Team") → compute the visible step list from `values` (`useMemo`) and clamp `stepIndex` when it shrinks.
- Browser Back button leaves the page — mention syncing `stepIndex` with the URL hash/query if the flow should be history-aware.

---

## 8. Accessibility

- Progress indicator is a `<nav aria-label="Progress">` with an ordered list; the current step uses **`aria-current="step"`**, completed steps get visually-hidden ", completed" text. Don't rely on colour alone — markers show ✓ / numbers.
- `role="progressbar"` with `aria-valuenow` for the bar (or hide it if it's purely decorative and the list already conveys it).
- Each control has a `<label htmlFor>`; errors are linked via `aria-describedby` and flagged with `aria-invalid`; messages use `role="alert"` so they're announced.
- **Focus moves** to the step heading on navigation and to the first invalid field on failure — otherwise keyboard/screen-reader users are stranded on a button that just disappeared.
- Native form semantics: Enter submits, Tab order follows DOM order, buttons are real `<button>`s. Disable (or hide) "Back" on step 1.
- Autocomplete tokens (`name`, `email`, `postal-code`, `new-password`) help assistive tech and password managers.
- Sufficient colour contrast on error text and focus rings.

---

## 9. Performance Considerations

- Only the active step's fields are mounted — tiny DOM, no cost for hidden steps.
- Any keystroke re-renders `MultiStepForm` and its few fields: negligible. For very large forms, store values in a ref/uncontrolled inputs (`FormData`) or use `React.memo` on `Field` with stable handlers (`useCallback`), or a library like React Hook Form.
- Validation is O(fields) and synchronous; for async checks (email taken) debounce, cancel stale requests, and don't block typing.
- Derived values (`percent`, `isReview`) are computed in render — no extra state or effects.
- Avoid effects for "sync" logic (clearing errors on step change) — do it in the event handlers where it's causal.

---

## 10. Follow-up Questions

<details>
<summary><strong>1. How would you persist progress across refreshes?</strong></summary>

Initialise state lazily from `sessionStorage` (`useState(() => load())`) and save `{ stepIndex, values }` in an effect on change. **Exclude the password** (and other secrets) from storage. Validate the shape on load and clear the draft on successful submit.

</details>

<details>
<summary><strong>2. How do you support conditional steps?</strong></summary>

Compute `visibleSteps = useMemo(() => ALL.filter(s => !s.when || s.when(values)), [values])` and use that array everywhere instead of the static one. Ensure `stepIndex` stays in range if the list shrinks (clamp or track the current step by `id` instead of index — tracking by id is more robust).

</details>

<details>
<summary><strong>3. Controlled vs uncontrolled inputs here?</strong></summary>

Controlled (state per keystroke) is simplest and enables instant validation/clearing errors; it re-renders the form each keystroke, which is fine for small forms. Uncontrolled inputs + `FormData`/refs reduce renders but then you must handle "preserve values across steps" explicitly (e.g. keep hidden inputs mounted or copy values to state on Next). React Hook Form takes the uncontrolled route with `shouldUnregister: false`.

</details>

<details>
<summary><strong>4. How would you add async validation (email already taken)?</strong></summary>

Run it on blur or on Next: `await checkEmail(email)`; keep a per-field `validating` flag; cancel/ignore stale results with an `AbortController` or a request-id ref; block Next while pending. Never rely solely on client checks — the server must re-validate on submit.

</details>

<details>
<summary><strong>5. Should each step be its own `<form>`?</strong></summary>

Either works. One `<form>` with conditionally rendered fields (as here) keeps Enter-to-advance and a single submit handler. Separate forms per step help with libraries and code-splitting. Do **not** keep all steps mounted inside one form with `display:none` fields: hidden required fields would block native validation, and they'd stay in the tab order unless `hidden`/`inert`.

</details>

<details>
<summary><strong>6. How do you sync the current step with the browser history?</strong></summary>

Store `step` in the URL (`?step=address`) using `pushState` on Next and listen to `popstate` so Back/Forward navigate steps. Guard direct access: if the URL asks for a step whose predecessors are invalid, redirect to the first invalid one.

</details>

<details>
<summary><strong>7. Why compute validation in a pure function outside the component?</strong></summary>

It's trivially unit-testable, reusable on the server, and reused by three callers (`goTo`, `submit`, blur) without duplicating rules. Schema libraries (Zod/Yup) do the same job declaratively.

</details>

<details>
<summary><strong>8. How would you prevent leaving the page with unsaved changes?</strong></summary>

Track `isDirty = values !== initialValues` (deep-compare or a flag set on change) and register a `beforeunload` handler in an effect (add when dirty, remove in cleanup). With a router, also use its navigation blocker.

</details>

---

## 11. Common Mistakes

- Each step keeping its own state → data lost on unmount when going Back.
- Validating only on the final submit (or only the current step on submit) — and not re-validating all steps at the end.
- Letting users jump forward through the indicator without validating the skipped steps.
- Not clearing stale errors when changing steps or editing a field.
- Using `type="button"` for Next and wiring Enter manually (or using separate `<form>`s inconsistently) so Enter does nothing or submits prematurely.
- Mutating state (`values[name] = v`) or using stale state in a handler (`setValues({...values, [name]: v})` in async code) — prefer the functional form.
- Forgetting focus management — focus stays on a removed button; errors aren't announced.
- Showing errors as colour only, or with `title` attributes.
- Putting the step *content* in `display:none` panels but leaving them focusable.
- Hard-coding step logic in JSX (`if (step === 2) …` everywhere) instead of config.
- Not disabling/guarding Submit → duplicate requests.
- Storing the password in persisted drafts.
- Swapping one button element between `type="button"` Next and `type="submit"` Submit without `key`s → the same click can submit the form on the final step transition.

