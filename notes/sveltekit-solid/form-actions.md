# Form actions

> **In one line:** Form actions are server functions in `+page.server.ts` that handle a `<form method="POST">`, so the form works even with no JavaScript, and `use:enhance` upgrades it to a smooth no-reload submit when JS is there.

## Key points
- **Actions live in `+page.server.ts`** as `export const actions = { ... }`. They only run on the server, so you can use secrets, cookies and the database ([form actions docs](https://svelte.dev/docs/kit/form-actions)).
- **Default vs named actions.** `default` handles a plain `<form method="POST">`. Named actions are called with `action="?/login"`. A page can not mix a `default` action with named ones.
- **`fail(status, data)`** returns a validation error. The page is re-rendered and `data` shows up in the `form` prop, so you can show the error and keep what the user typed.
- **Progressive enhancement** means the base feature works with plain HTML, and JS only makes it nicer ([MDN glossary](https://developer.mozilla.org/en-US/docs/Glossary/Progressive_Enhancement)). Without JS, the browser does a normal full-page POST.
- **`use:enhance`** (from `$app/forms`) intercepts the submit, sends it with `fetch`, updates `form`, re-runs `load` functions (`invalidateAll`), and resets the form on success. No page reload.

## Example
A buy-order form on a stock page.

```ts
// src/routes/stocks/[symbol]/+page.server.ts
import { fail, redirect } from '@sveltejs/kit';
import type { Actions } from './$types';

export const actions = {
  buy: async ({ request, params, locals }) => {
    if (!locals.user) redirect(303, '/login');      // not logged in

    const formData = await request.formData();
    const qty = Number(formData.get('qty'));

    if (!Number.isInteger(qty) || qty <= 0) {
      // 400 + data goes back to the page as `form`
      return fail(400, { qty: formData.get('qty'), error: 'Quantity must be a positive whole number' });
    }

    await placeOrder(locals.user.id, params.symbol, qty); // your DB / broker call
    return { success: true, message: `Bought ${qty} ${params.symbol}` };
  }
} satisfies Actions;
```

```svelte
<!-- src/routes/stocks/[symbol]/+page.svelte -->
<script lang="ts">
  import { enhance } from '$app/forms';
  import type { PageProps } from './$types';
  let { data, form }: PageProps = $props();
  let submitting = $state(false);
</script>

<form
  method="POST"
  action="?/buy"
  use:enhance={() => {
    submitting = true;                       // runs before the request
    return async ({ update }) => {
      await update();                        // default behaviour: update form, invalidate, reset
      submitting = false;
    };
  }}
>
  <label>Qty <input name="qty" value={form?.qty ?? ''} /></label>
  {#if form?.error}<p role="alert">{form.error}</p>{/if}
  {#if form?.success}<p>{form.message}</p>{/if}
  <button disabled={submitting}>Buy</button>
</form>
```

## When to use it
Any form that changes data on the server: login, place order, add to watchlist, update profile. In a trading app the server must validate anyway (never trust the client), so actions keep that logic in one place.

## Likely questions
### What is a form action and why use it instead of a fetch call?
It is a server function tied to a page that handles a POST from a `<form>`. The main win is that the form works without JavaScript, because it is a real HTML form. Validation and secrets stay on the server, and after the action SvelteKit re-runs `load` so the page shows fresh data.

### What does `use:enhance` do?
Without it, submit does a full page reload. With it, SvelteKit submits using `fetch`, then updates `form` and `page.status`, calls `invalidateAll()` so load data refreshes, resets the form on success, and follows redirects with `goto`. You can pass a callback to show a spinner, `cancel()` the submit, or call `update({ reset: false })` to keep input values.

### How do you return validation errors?
Return `fail(400, { ...data })`. The status is set on the response, and the object becomes the `form` prop. Send back the user's input (not passwords) so the fields stay filled.

### How do you redirect after a successful action?
Call `redirect(303, '/orders')`. 303 tells the browser to do a GET on the new page, which avoids "resubmit form?" on refresh.

### What is progressive enhancement?
Build the feature with plain HTML first so it always works, then add JS on top for a better experience. Form actions plus `use:enhance` is exactly this.

## Common mistakes
- Forgetting `method="POST"`. Actions only handle POST.
- Mixing `default` with named actions in the same file (SvelteKit throws).
- Doing validation only in the browser. Always validate in the action.
- Calling `redirect()` inside a `try/catch`. It works by throwing, so a catch swallows it.

## Resources
- [SvelteKit: Form actions](https://svelte.dev/docs/kit/form-actions) - full guide including `use:enhance` options
- [SvelteKit: $app/forms](https://svelte.dev/docs/kit/$app-forms) - `enhance`, `applyAction`, `deserialize` reference
- [MDN: Progressive enhancement](https://developer.mozilla.org/en-US/docs/Glossary/Progressive_Enhancement) - the idea behind it
