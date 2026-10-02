# Data loading

> **In one line:** A route gets its data from a `load` function: `+page.ts` is a universal load that runs on the server and in the browser, `+page.server.ts` is a server load that only ever runs on the server.

## Key points
- **Universal load (`+page.ts`, `+layout.ts`)** runs on the server for the first SSR request, then in the browser for later client-side navigations. Good for public APIs. It can return anything, even classes or components ([load docs](https://svelte.dev/docs/kit/load)).
- **Server load (`+page.server.ts`, `+layout.server.ts`)** always runs on the server. Use it for databases, secrets, cookies and private APIs. Its return value must be serializable (JSON plus `Date`, `Map`, `Set`, `BigInt` and so on, using a library called devalue).
- **If both exist**, the server load runs first, and its result arrives as `data` inside the universal load.
- **Reruns are smart.** SvelteKit tracks what a load used (`params.x`, `url`, `fetch` URLs, `depends` keys). You can force a rerun with `invalidate(key)` or `invalidateAll()`.
- **Streaming.** A server load can return an un-awaited promise. The page renders right away and the promise streams in later.

## Example
A stock page: fast quote first, slow news streamed, and a refresh button.

```ts
// src/routes/stocks/[symbol]/+page.server.ts
import { error } from '@sveltejs/kit';
import { MARKET_API_KEY } from '$env/static/private';
import type { PageServerLoad } from './$types';

export const load: PageServerLoad = async ({ params, fetch, depends, setHeaders }) => {
  depends('app:quote');                       // custom key so the client can refresh it

  const res = await fetch(`https://api.example.com/quote/${params.symbol}`, {
    headers: { Authorization: `Bearer ${MARKET_API_KEY}` } // secret stays on server
  });
  if (!res.ok) error(404, `Unknown symbol ${params.symbol}`);

  setHeaders({ 'cache-control': 'private, max-age=5' });

  return {
    quote: await res.json(),                  // awaited: needed for first paint
    news: fetch(`https://api.example.com/news/${params.symbol}`)
      .then((r) => r.json())                  // NOT awaited: streamed later
      .catch(() => [])                        // always handle rejection
  };
};
```

```svelte
<!-- src/routes/stocks/[symbol]/+page.svelte -->
<script lang="ts">
  import { invalidate } from '$app/navigation';
  import type { PageProps } from './$types';
  let { data }: PageProps = $props();
</script>

<h1>{data.quote.symbol} {data.quote.price}</h1>
<button onclick={() => invalidate('app:quote')}>Refresh</button>

{#await data.news}
  <p>Loading news...</p>
{:then items}
  {#each items as item (item.id)}<p>{item.title}</p>{/each}
{/await}
```

## When to use it
- `+page.server.ts`: anything with a secret, a DB call, cookies or the session (portfolio, orders, account).
- `+page.ts`: public data you can fetch from the browser directly (public market list), or when you must return something not serializable (a chart class instance).
- `depends` + `invalidate`: a "refresh quote" button, or re-fetch after placing an order.
- Streaming: the slow parts of a page (news, analyst ratings) so the price shows instantly.

## Likely questions
### Where does each load run?
Server load: only on the server, both for the first request and for client navigations (the browser asks the server for JSON). Universal load: on the server during SSR, then again in the browser during hydration (reusing the fetch responses SvelteKit inlined into the HTML, so no duplicate request), and only in the browser for later navigations. If `ssr = false`, the universal load only runs in the browser.

### When would you pick `+page.ts` over `+page.server.ts`?
I default to `+page.server.ts` for anything private. I use `+page.ts` when the data comes from a public API that the browser can call directly, which saves a hop through my server on client navigations. I also use it when I need to return non-serializable things, like a component constructor. In rare cases I use both: the server load fetches private data and the universal load wraps it in a class.

### What is the `fetch` passed to load and why use it?
It is a wrapped `fetch`. It can use relative URLs on the server, forwards cookies for same-origin requests, and during SSR it records responses and inlines them in the HTML so the browser does not fetch again on hydration. It also registers the URL as a dependency, so `invalidate(url)` reruns the load.

### How do `depends` and `invalidate` work?
`depends('app:quote')` marks the load as depending on a custom key (it must look like `scheme:name`). Calling `invalidate('app:quote')` in the browser reruns every active load that depends on that key. `invalidate(url)` works for URLs fetched with the load's `fetch`, and `invalidateAll()` reruns every load on the page. Rerunning updates the `data` prop; the component is not recreated, so local state is kept.

### When does a load rerun on navigation?
When a `params` value it used changes, when a URL part it read changes (`url.pathname`, a search param), when it awaited `parent()` and the parent reran, when one of its dependencies was invalidated, or on `invalidateAll()`. Untouched layout loads do not rerun, which is why layout data is cheap. You can opt out of tracking with `untrack(() => ...)`.

### How does streaming work?
Since SvelteKit 2, top-level promises returned from a server load are not awaited. The HTML is sent with the resolved parts, then the promise values stream in over the same response. In the page you use `{#await}`. Two rules: always handle rejection (or the server can crash with an unhandled rejection), and you cannot set headers or redirect once streaming starts. Also, it needs JavaScript and a host that does not buffer responses.

### How do you avoid waterfalls?
All loads on a page run in parallel by default. `await parent()` makes a load wait for its parent, so call it as late as possible, after starting independent fetches. Use `Promise.all` for independent calls in one load.

## Common mistakes
- Returning a promise from a universal load on an SSR page and expecting it to stream. It does not; it reruns in the browser.
- Importing a private env var or DB client into `+page.ts`. Build fails, because that file also runs in the browser.
- Putting auth checks only in `+layout.server.ts`. Child loads run in parallel and may run anyway; use `hooks.server.ts`.
- Using the global `fetch` instead of the load's `fetch`, which causes a double request on hydration.
- **SvelteKit 3 note:** `invalidateAll` is deprecated in favour of `refreshAll`.

## Resources
- [SvelteKit docs: Loading data](https://svelte.dev/docs/kit/load) - universal vs server, rerun rules, streaming
- [Tutorial: Invalidation](https://svelte.dev/tutorial/kit/invalidation) - practice `invalidate`
- [Tutorial: Custom dependencies](https://svelte.dev/tutorial/kit/custom-dependencies) - practice `depends`
- [SvelteKit docs: $app/navigation](https://svelte.dev/docs/kit/$app-navigation) - `invalidate` API
