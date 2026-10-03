# API routes

> **In one line:** A `+server.ts` file turns a route into an API endpoint: you export functions named after HTTP methods, like `GET` and `POST`, and each one takes a request event and returns a standard `Response`.

## Key points
- **One function per HTTP method:** `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `OPTIONS`, `HEAD`, plus `fallback` for anything else ([routing docs: +server](https://svelte.dev/docs/kit/routing)).
- **Web standard objects.** You get a [`Request`](https://developer.mozilla.org/en-US/docs/Web/API/Request) and return a [`Response`](https://developer.mozilla.org/en-US/docs/Web/API/Response). Helpers `json()` and `text()` from `@sveltejs/kit` build responses for you.
- **Same event as `load`:** `params`, `url`, `cookies`, `locals`, `fetch`, `setHeaders`. `error()` and `redirect()` work too.
- **Living next to a page.** If a folder has both `+page.svelte` and `+server.ts`, `PUT/PATCH/DELETE/OPTIONS` always go to `+server.ts`. `GET/POST/HEAD` go to the page when the `accept` header prefers `text/html` (a browser visit), otherwise to `+server.ts`.
- They run only on the server, so secrets are safe here.

## Example
A watchlist API.

```ts
// src/routes/api/watchlist/+server.ts
import { json, error } from '@sveltejs/kit';
import type { RequestHandler } from './$types';

export const GET: RequestHandler = async ({ locals, setHeaders }) => {
  if (!locals.user) error(401, 'Login required');
  const items = await db.watchlist.findMany({ userId: locals.user.id });
  setHeaders({ 'cache-control': 'private, max-age=10' });
  return json(items);
};

export const POST: RequestHandler = async ({ request, locals }) => {
  if (!locals.user) error(401, 'Login required');
  const { symbol } = await request.json();
  if (typeof symbol !== 'string' || !/^[A-Z]{1,10}$/.test(symbol)) error(400, 'Bad symbol');

  const item = await db.watchlist.add({ userId: locals.user.id, symbol });
  return json(item, { status: 201 });
};
```

```ts
// src/routes/api/watchlist/[symbol]/+server.ts
import type { RequestHandler } from './$types';

export const DELETE: RequestHandler = async ({ params, locals }) => {
  await db.watchlist.remove({ userId: locals.user!.id, symbol: params.symbol });
  return new Response(null, { status: 204 });
};
```

Calling it from a component:

```ts
await fetch('/api/watchlist', {
  method: 'POST',
  headers: { 'content-type': 'application/json' },
  body: JSON.stringify({ symbol: 'AAPL' })
});
```

## When to use it
- JSON APIs for client-side code: a search box, a "star" button, polling.
- Webhooks from a payment or KYC provider.
- Streaming, like Server-Sent Events for live prices (return a `Response` with a `ReadableStream` body).
- Use **form actions** instead when it is a form on a page, because you get progressive enhancement for free.

## Likely questions
### How do you create a GET and POST endpoint in SvelteKit?
Create `src/routes/api/thing/+server.ts` and export `GET` and `POST` functions. Each receives a `RequestEvent` and returns a `Response`, often via `json(data)`. Read the body with `await request.json()` or `request.formData()`.

### Form action or `+server.ts`?
For a form on a page, I use actions: it works without JS and SvelteKit re-runs `load`. For a JSON API used by `fetch`, other clients, webhooks or streaming, I use `+server.ts`.

### How do you handle errors in an endpoint?
Call `error(400, 'message')`. SvelteKit returns a JSON error body if the client accepts JSON. Unexpected throws become 500 and go through `handleError`.

### Can endpoints be prerendered?
Yes, with `export const prerender = true`, if the output does not depend on the request. Useful for a static list of tickers.

## Common mistakes
- Forgetting to check auth in the endpoint (hooks set `locals.user`, but each handler must still check it).
- Trusting the request body without validation.
- Calling your own API with `fetch` from a server `load`. Just call the DB function directly.

## Resources
- [SvelteKit: Routing (+server)](https://svelte.dev/docs/kit/routing) - endpoints and content negotiation
- [SvelteKit: @sveltejs/kit reference](https://svelte.dev/docs/kit/@sveltejs-kit) - `json`, `text`, `error`, `RequestHandler`
- [MDN: Response](https://developer.mozilla.org/en-US/docs/Web/API/Response) - the object you return
