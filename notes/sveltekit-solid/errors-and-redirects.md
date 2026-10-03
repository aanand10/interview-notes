# Errors and redirects

> **In one line:** In SvelteKit you call `error(status, message)` for errors you expect, like "stock not found", and `redirect(status, url)` to send the user elsewhere; any other crash is an unexpected error, and both kinds are shown by the nearest `+error.svelte`.

## Key points
- **Expected errors** are made with `error(404, 'Not found')` from `@sveltejs/kit`. Status and message go to the user as-is ([errors docs](https://svelte.dev/docs/kit/errors)).
- **Unexpected errors** are any other exception (a bug, a DB timeout). SvelteKit logs them, passes them to `handleError`, and shows the user a generic message (`"Internal Error"` by default) with status 500, so no secrets leak.
- **`+error.svelte`** is the error page for that route folder. SvelteKit walks up the folder tree to the nearest one. Read details with `page.error` and `page.status` from `$app/state`.
- **`redirect(status, location)`** stops the load or action and redirects. Use 303 after a form POST, 307 for a temporary redirect that keeps the method, 308 for permanent.
- In SvelteKit 2, `error()` and `redirect()` throw for you, so you do not write `throw` in front.

## Example
```ts
// src/routes/stocks/[symbol]/+page.server.ts
import { error, redirect } from '@sveltejs/kit';
import type { PageServerLoad } from './$types';

export const load: PageServerLoad = async ({ params, locals }) => {
  if (!locals.user) redirect(307, '/login');           // not logged in

  if (params.symbol !== params.symbol.toUpperCase()) {
    redirect(308, `/stocks/${params.symbol.toUpperCase()}`); // canonical URL
  }

  const stock = await db.getStock(params.symbol);       // if this throws -> unexpected (500)
  if (!stock) error(404, `No stock called ${params.symbol}`); // expected
  return { stock };
};
```

```svelte
<!-- src/routes/stocks/+error.svelte -->
<script lang="ts">
  import { page } from '$app/state';
</script>

<h1>{page.status}</h1>
<p>{page.error?.message}</p>
<a href="/stocks">Back to all stocks</a>
```

You can also add fields to errors by typing `App.Error` in `src/app.d.ts`, for example `{ message: string; errorId?: string }`.

## When to use it
- `error(404)` for an unknown symbol or order ID, `error(403)` when a user opens someone else's order.
- `redirect` for login gates, after placing an order (`303` to `/orders/123`), and old URLs.
- A custom `+error.svelte` per section so a broken chart page still shows the app shell.

## Likely questions
### What is the difference between expected and unexpected errors?
Expected errors are ones I create with `error()`, like 404 or 403. The message is safe and shown to the user. Unexpected errors are anything else that throws. SvelteKit hides the real message, returns 500, and calls `handleError` so I can log it and return a safe object, maybe with an error ID.

### How does `+error.svelte` get chosen?
SvelteKit uses the closest `+error.svelte` above the route that failed. One catch: if the error is in a `+layout` load, the `+error.svelte` next to that layout is not used (it renders inside that layout), so it goes to the parent's. If the root layout itself fails, SvelteKit uses the static `src/error.html` fallback.

### Which status code do you use for redirect?
303 after a POST (form action) so the browser does a GET. 307 for a temporary redirect that keeps the method. 308 for a permanent move, good for SEO. 301/302 also work but are less precise about the method.

### Can you catch a redirect in try/catch?
`redirect()` works by throwing a special object. If you wrap it in `try/catch`, your catch will grab it and the redirect never happens. Call it outside the try, or rethrow when `isRedirect(e)` is true.

## Common mistakes
- Calling `redirect()` inside `try { } catch { }` and swallowing it.
- Returning raw error messages or stack traces to users from `handleError`.
- Expecting `+error.svelte` to catch errors in event handlers. It only covers `load` and rendering; handle click/fetch errors yourself.

## Resources
- [SvelteKit: Errors](https://svelte.dev/docs/kit/errors) - expected vs unexpected, error pages
- [SvelteKit: Routing (+error)](https://svelte.dev/docs/kit/routing) - where `+error.svelte` sits in the tree
- [MDN: HTTP redirections](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Redirections) - what 301/302/303/307/308 mean
