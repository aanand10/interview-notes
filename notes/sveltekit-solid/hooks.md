# Hooks

> **In one line:** Hooks are app-wide functions in `src/hooks.server.ts` that run on every request, like middleware: `handle` for auth, logging and headers, `handleError` for crashes, and `handleFetch` to change server-side `fetch` calls.

## Key points
- **`handle({ event, resolve })`** runs for every request (pages and `+server.ts` endpoints). You can read cookies, set `event.locals`, call `resolve(event)` to render, then change the response ([hooks docs](https://svelte.dev/docs/kit/hooks)).
- **`event.locals`** is a per-request object. Put the logged-in user there and every `load` and action can read it. Type it in `src/app.d.ts` under `App.Locals`.
- **`sequence(a, b, c)`** from `@sveltejs/kit/hooks` chains several `handle` functions in order.
- **`handleError`** runs for *unexpected* errors (bugs, crashes). Log them and return a safe object for the user. It does not run for `error(404, ...)` that you throw on purpose.
- **`handleFetch`** lets you change `fetch` requests made inside server `load` functions or actions, for example to add an auth header or point to an internal URL.

There are also `src/hooks.client.ts` (has `handleError` for the browser) and universal `src/hooks.ts` (`reroute`, `transport`).

## Example
```ts
// src/hooks.server.ts
import { redirect, type Handle, type HandleServerError, type HandleFetch } from '@sveltejs/kit';
import { sequence } from '@sveltejs/kit/hooks';

const logger: Handle = async ({ event, resolve }) => {
  const start = performance.now();
  const response = await resolve(event);
  console.log(`${event.request.method} ${event.url.pathname} ${response.status} ${Math.round(performance.now() - start)}ms`);
  return response;
};

const auth: Handle = async ({ event, resolve }) => {
  const sessionId = event.cookies.get('session');
  event.locals.user = sessionId ? await getUserFromSession(sessionId) : null;

  // protect all /portfolio pages in one place
  if (event.url.pathname.startsWith('/portfolio') && !event.locals.user) {
    redirect(303, `/login?next=${encodeURIComponent(event.url.pathname)}`);
  }
  return resolve(event);
};

const securityHeaders: Handle = async ({ event, resolve }) => {
  const response = await resolve(event);
  response.headers.set('X-Frame-Options', 'DENY');
  response.headers.set('Referrer-Policy', 'strict-origin-when-cross-origin');
  return response;
};

export const handle = sequence(logger, auth, securityHeaders);

export const handleError: HandleServerError = async ({ error, event, status, message }) => {
  const errorId = crypto.randomUUID();
  console.error(errorId, event.url.pathname, error);   // send to Sentry etc.
  return { message: 'Something went wrong', errorId }; // what the user sees in page.error
};

export const handleFetch: HandleFetch = async ({ event, request, fetch }) => {
  if (request.url.startsWith('https://api.mybroker.com/')) {
    request.headers.set('cookie', event.request.headers.get('cookie') ?? '');
  }
  return fetch(request);
};
```

## When to use it
- Read the session once and protect a whole section (`/portfolio`, `/orders`) instead of checking in every `load`.
- Add security headers (CSP, frame options) to every response.
- Request logging and error tracking with an error ID the support team can search for.

## Likely questions
### How would you implement auth in SvelteKit?
In `handle`, read the session cookie, look up the user, and put it on `event.locals.user`. Then load functions and actions read `locals.user`. For route protection I either redirect in `handle` for a path prefix, or check in a `+layout.server.ts` for that group of routes. The real check must be on the server.

### What does `resolve` do and what options does it take?
`resolve(event)` runs the actual route and returns the `Response`. Options include `transformPageChunk` (edit the HTML, for example set `lang` or a theme class), `filterSerializedResponseHeaders`, and `preload` (choose which files get preload links).

### When does `handleError` run?
Only for unexpected errors: something threw that was not created with `error()`. Its return value becomes `page.error`, so never return the raw stack or message to users. Expected errors from `error(404, ...)` skip it.

### What is `handleFetch` for?
It intercepts the `event.fetch` used on the server. Common uses: forward cookies to a different domain, add an API key, or rewrite a public URL to an internal one so the request skips the public internet.

## Common mistakes
- Forgetting to `return resolve(event)`, so the request hangs or errors.
- Putting heavy work in `handle` for every request, including static asset requests that hit the server.
- Leaking error details from `handleError`.

## Resources
- [SvelteKit: Hooks](https://svelte.dev/docs/kit/hooks) - every hook with examples
- [SvelteKit: Errors](https://svelte.dev/docs/kit/errors) - how `handleError` fits in
- [OWASP: HTTP Headers Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/HTTP_Headers_Cheat_Sheet.html) - which security headers to set in `handle`
