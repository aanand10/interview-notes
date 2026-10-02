# Login form

> **In one line:** A good login form is a small state machine (idle, submitting, error, success) with clear validation, a submit button that cannot fire twice, and labels and errors that a screen reader can read.

## Requirements to confirm
Ask these in the first 2 minutes. It shows you think before you code.
- Fields: email or phone or client ID? Password or MPIN? Is there a "remember me"?
- When to validate: on blur, on submit, or on every keystroke? (Good default: on blur and on submit, then live once a field has shown an error.)
- What does the API return on failure? A field error ("wrong password") or a general error ("account locked", "server down")?
- What happens on success: redirect, show a message, or call a callback?
- Do we need a password visibility toggle? Rate limiting or captcha after N failures?

## Component breakdown
- `LoginForm.svelte`: owns the form state and the submit logic.
- `TextField.svelte` (optional, if time): label + input + error text, wired with `aria-describedby`.
- `PasswordField`: a text field plus a show/hide button.
- `login(email, password)`: a plain async function that calls the API. Keeping it outside the component makes it easy to mock in tests.

## State and data flow
- `values` (email, password), `touched` (which fields the user has left), `status` (`idle` / `submitting` / `error` / `success`), `serverError` (string).
- `errors` is **derived** from `values`. It is never stored separately, so it can never go stale. See [$derived](https://svelte.dev/docs/svelte/$derived).
- The submit button is disabled while `status === 'submitting'`. That is the main double-submit guard.
- Flow: user types, `values` change, `errors` recompute, user leaves a field, `touched` marks it, the error shows. On submit, mark all fields touched, stop if invalid, else call the API.

## Implementation
```svelte
<script>
  // Pretend API. Replace with fetch('/api/login', ...)
  async function login(email, password) {
    await new Promise((r) => setTimeout(r, 800));
    if (password !== 'secret123') throw new Error('Email or password is incorrect.');
    return { name: 'Anand' };
  }

  let values = $state({ email: '', password: '' });
  let touched = $state({ email: false, password: false });
  let status = $state('idle'); // 'idle' | 'submitting' | 'error' | 'success'
  let serverError = $state('');
  let showPassword = $state(false);
  let emailInput = $state(); // DOM ref via bind:this

  // Derived: always in sync with values, never stale
  const errors = $derived({
    email: !values.email.trim()
      ? 'Email is required.'
      : !/^\S+@\S+\.\S+$/.test(values.email)
        ? 'Enter a valid email, like name@example.com.'
        : '',
    password: !values.password
      ? 'Password is required.'
      : values.password.length < 8
        ? 'Password must be at least 8 characters.'
        : ''
  });
  const isValid = $derived(!errors.email && !errors.password);

  async function handleSubmit(event) {
    event.preventDefault();
    if (status === 'submitting') return; // second guard against double submit
    touched = { email: true, password: true };
    if (!isValid) {
      // Move focus to the first broken field so keyboard users know where to go
      if (errors.email) emailInput.focus();
      else document.getElementById('password').focus();
      return;
    }
    status = 'submitting';
    serverError = '';
    try {
      await login(values.email.trim(), values.password);
      status = 'success';
    } catch (err) {
      status = 'error';
      serverError = err.message || 'Something went wrong. Please try again.';
    }
  }
</script>

{#if status === 'success'}
  <p role="status">Welcome back! Redirecting...</p>
{:else}
  <form onsubmit={handleSubmit} novalidate aria-busy={status === 'submitting'}>
    {#if serverError}
      <p role="alert" class="form-error">{serverError}</p>
    {/if}

    <label for="email">Email</label>
    <input
      id="email"
      type="email"
      autocomplete="username"
      bind:this={emailInput}
      bind:value={values.email}
      onblur={() => (touched.email = true)}
      aria-invalid={touched.email && !!errors.email}
      aria-describedby={touched.email && errors.email ? 'email-error' : undefined}
    />
    {#if touched.email && errors.email}
      <p id="email-error" class="field-error">{errors.email}</p>
    {/if}

    <label for="password">Password</label>
    <div class="password-row">
      <input
        id="password"
        type={showPassword ? 'text' : 'password'}
        autocomplete="current-password"
        bind:value={values.password}
        onblur={() => (touched.password = true)}
        aria-invalid={touched.password && !!errors.password}
        aria-describedby={touched.password && errors.password ? 'password-error' : undefined}
      />
      <button
        type="button"
        onclick={() => (showPassword = !showPassword)}
        aria-pressed={showPassword}
        aria-controls="password"
      >
        {showPassword ? 'Hide' : 'Show'}<span class="sr-only"> password</span>
      </button>
    </div>
    {#if touched.password && errors.password}
      <p id="password-error" class="field-error">{errors.password}</p>
    {/if}

    <button type="submit" disabled={status === 'submitting'}>
      {status === 'submitting' ? 'Signing in...' : 'Sign in'}
    </button>
  </form>
{/if}
```

## Edge cases and accessibility
- **Real labels**: every input has a `<label for>`. Placeholder is not a label; it disappears when you type.
- **Errors linked to inputs**: [aria-describedby](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Reference/Attributes/aria-describedby) points to the error text and [aria-invalid](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Reference/Attributes/aria-invalid) marks the field, so a screen reader reads "Email, invalid, Enter a valid email".
- **Server error announced**: `role="alert"` makes it read out right away ([alert role](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Reference/Roles/alert_role)).
- **Focus first invalid field** on submit, so keyboard users are not lost.
- **Password managers**: `autocomplete="username"` and `"current-password"` let them fill the form ([autocomplete](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Attributes/autocomplete)).
- **Toggle button** is `type="button"` so it does not submit the form, and it uses `aria-pressed` to say its state.
- **Trim the email** but never trim the password; spaces can be part of a password.
- **Enter key** works for free because we use a real `<form>` and a `type="submit"` button.
- **Do not leak info**: show "Email or password is incorrect", not "this email does not exist" ([OWASP auth cheat sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)).

## What interviewers look for
- A clear status model instead of five unrelated booleans.
- Derived errors, not duplicated state.
- No double submit, a loading label, and the form stays usable after an error.
- Accessible labels, linked errors, and focus management.
- Separation: API call outside the UI, so it can be tested and mocked.

## Likely questions
### How do you prevent double submit?
I disable the submit button while `status` is `submitting`, and I also return early in the handler if a request is already in flight. The early return is defence in depth: the disabled attribute only protects the button, and the guard protects any other path that calls the handler. For payments or orders I would also send an idempotency key so the server ignores duplicates.

### When do you show validation errors?
Not while the user is typing their first character, that feels rude. I show an error after the field loses focus (blur) or on submit. Once an error is visible, it updates live as they fix it, so they see it disappear.

### Why `novalidate` on the form?
The browser's built-in bubbles look different in every browser and are hard to style. With `novalidate` I keep the semantic types (`type="email"`) for the mobile keyboard, but show my own consistent, accessible messages. See [MDN form validation](https://developer.mozilla.org/en-US/docs/Learn_web_development/Extensions/Forms/Form_validation).

### Is client-side validation enough?
No. Client validation is for user experience only. The server must validate again, because anyone can call the API directly.

### How would you do this in SvelteKit?
I would use a [form action](https://svelte.dev/docs/kit/form-actions) with `use:enhance`. The form works without JavaScript, the server returns `fail(400, { errors })`, and the session goes into an `httpOnly` cookie instead of `localStorage`, so JavaScript (and XSS) cannot read it.

## Resources
- [web.dev: Sign-in form best practices](https://web.dev/articles/sign-in-form-best-practices) - autocomplete, labels, password toggle
- [MDN: Client-side form validation](https://developer.mozilla.org/en-US/docs/Learn_web_development/Extensions/Forms/Form_validation) - built-in vs custom validation
- [OWASP: Authentication cheat sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html) - error messages and lockout
- [SvelteKit: Form actions](https://svelte.dev/docs/kit/form-actions) - progressive enhancement
