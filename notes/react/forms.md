# Controlled vs uncontrolled forms

> **In one line:** A controlled input gets its value from React state and updates it on every change, while an uncontrolled input keeps its own value in the DOM and you read it when needed (ref or `FormData`); React 19 adds form actions, `useActionState` and `useFormStatus` to handle submit, pending and errors with less code.

## Key points
- **Controlled**: `value={state}` + `onChange`. React is the single source of truth. Easy to validate live, format, or disable buttons. Costs a re-render per keystroke.
- **Uncontrolled**: `defaultValue` + read on submit via `ref` or [`FormData`](https://developer.mozilla.org/en-US/docs/Web/API/FormData). Less code, fewer re-renders, works great with native validation.
- **Validation**: use native HTML attributes (`required`, `min`, `type="email"`) first, then a schema (Zod) for rules, and always validate again on the server.
- **React Hook Form** uses uncontrolled inputs with refs, so typing doesn't re-render the whole form. Good for large forms.
- **React 19**: `<form action={fn}>`, [`useActionState`](https://react.dev/reference/react/useActionState) for result/error + pending state, [`useFormStatus`](https://react.dev/reference/react-dom/hooks/useFormStatus) for a submit button that knows the form is submitting.

## Example
Login form, controlled version, with errors and a disabled submit.

```tsx
import { useState, type FormEvent } from 'react';

export function LoginForm({ onLogin }: { onLogin: (e: string, p: string) => Promise<void> }) {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [touched, setTouched] = useState({ email: false, password: false });
  const [serverError, setServerError] = useState<string | null>(null);
  const [submitting, setSubmitting] = useState(false);

  // Derived during render, not stored in state
  const errors = {
    email: /^\S+@\S+\.\S+$/.test(email) ? '' : 'Enter a valid email',
    password: password.length >= 8 ? '' : 'At least 8 characters',
  };
  const isValid = !errors.email && !errors.password;

  async function handleSubmit(e: FormEvent) {
    e.preventDefault();
    if (!isValid || submitting) return;
    setSubmitting(true);
    setServerError(null);
    try {
      await onLogin(email, password);
    } catch {
      setServerError('Wrong email or password'); // don't reveal which one
    } finally {
      setSubmitting(false);
    }
  }

  return (
    <form onSubmit={handleSubmit} noValidate>
      <label htmlFor="email">Email</label>
      <input
        id="email"
        type="email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
        onBlur={() => setTouched((t) => ({ ...t, email: true }))}
        aria-invalid={touched.email && !!errors.email}
        aria-describedby="email-error"
      />
      {touched.email && errors.email && <p id="email-error">{errors.email}</p>}

      <label htmlFor="password">Password</label>
      <input
        id="password"
        type="password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
        onBlur={() => setTouched((t) => ({ ...t, password: true }))}
        aria-invalid={touched.password && !!errors.password}
        aria-describedby="password-error"
      />
      {touched.password && errors.password && <p id="password-error">{errors.password}</p>}

      {serverError && <p role="alert">{serverError}</p>}
      <button type="submit" disabled={!isValid || submitting}>
        {submitting ? 'Signing in...' : 'Sign in'}
      </button>
    </form>
  );
}
```

The same form with React 19 actions (uncontrolled inputs):

```tsx
import { useActionState } from 'react';
import { useFormStatus } from 'react-dom';

type State = { error: string | null };

async function loginAction(_prev: State, formData: FormData): Promise<State> {
  const email = String(formData.get('email'));
  const password = String(formData.get('password'));
  if (password.length < 8) return { error: 'At least 8 characters' };
  const res = await fetch('/api/login', { method: 'POST', body: JSON.stringify({ email, password }) });
  return res.ok ? { error: null } : { error: 'Wrong email or password' };
}

function SubmitButton() {
  const { pending } = useFormStatus(); // reads the parent <form>'s status
  return <button type="submit" disabled={pending}>{pending ? 'Signing in...' : 'Sign in'}</button>;
}

export function LoginFormV19() {
  const [state, formAction, isPending] = useActionState(loginAction, { error: null });
  return (
    <form action={formAction}>
      <input name="email" type="email" required />
      <input name="password" type="password" required minLength={8} />
      {state.error && <p role="alert">{state.error}</p>}
      <SubmitButton />
    </form>
  );
}
```

Svelte 5 equivalent: `bind:value={email}` gives a controlled input in one line; SvelteKit form actions with `use:enhance` play the role of React 19 actions.

## When to use it
Controlled: order ticket where qty and price update the "estimated cost" live, or a field that formats as you type. Uncontrolled / React Hook Form: long KYC or onboarding forms. React 19 actions: simple submit flows (login, add to watchlist), especially with server functions in frameworks like Next.js.

## Likely questions

### Controlled vs uncontrolled: which and why?
Controlled when the UI must react to every change: live validation, dependent fields, input masks, disabling the submit button. Uncontrolled when you only need values on submit, for performance in big forms, or for file inputs (`<input type="file">` is always uncontrolled). Don't switch one input between the two: going from `value={undefined}` to a string triggers React's "changing an uncontrolled input to be controlled" warning, so initialise with `''`.

### How do you approach validation?
Layered. Native constraints (`required`, `min`, `pattern`) give free accessibility and browser messages. Then schema validation (Zod or Yup) shared between client and server where possible. Show errors on blur or on submit, not on the first keystroke. Link errors to inputs with `aria-describedby` and set `aria-invalid`, and move focus to the first invalid field on submit. The server is the real authority; client validation is only for UX.

### What is React Hook Form at a high level?
A library that registers inputs as uncontrolled (via `register('email')`, which attaches a ref and `name`), so values live in the DOM and the form doesn't re-render on every keystroke. It gives `handleSubmit`, `formState.errors`, `isSubmitting`, and integrates with Zod via a resolver. Use `Controller` for custom components that need to be controlled (date pickers, selects).

### What are React 19 form actions, useActionState and useFormStatus?
You can pass a function to `<form action={fn}>`. React calls it with the `FormData` on submit, inside a transition, and resets uncontrolled fields after success. `useActionState(action, initialState)` wraps that action and returns `[state, formAction, isPending]`, where `state` is whatever the action last returned, like an error message. `useFormStatus()` from `react-dom` gives `{ pending, data, method, action }` for the parent form; it must be called in a component **rendered inside** the `<form>`, not in the component that renders the form.

### How do you prevent double submit?
Disable the button while pending (`submitting` state or `useFormStatus().pending`), guard in the handler, and for orders send an idempotency key so the server ignores a duplicate request.

## Common mistakes
- `value` without `onChange`, giving a read-only field warning.
- Initialising controlled inputs with `undefined`/`null`.
- Storing `isValid` or `errors` in state instead of computing them.
- Calling `useFormStatus` in the same component that renders the `<form>`.
- Trusting client-side validation for security.

## Resources
- [react.dev: input](https://react.dev/reference/react-dom/components/input) - controlled vs uncontrolled inputs explained
- [react.dev: useActionState](https://react.dev/reference/react/useActionState) - action result and pending state
- [react.dev: useFormStatus](https://react.dev/reference/react-dom/hooks/useFormStatus) - pending state for submit buttons
- [React Hook Form: Get started](https://react-hook-form.com/get-started) - register, errors and resolvers
