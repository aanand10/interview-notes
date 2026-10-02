# Rendering modes

> **In one line:** By default SvelteKit server-renders the first page and then hydrates it into a client app, and you can switch each route to prerendered, client-only, or no-JavaScript with three exports: `prerender`, `ssr` and `csr`.

## Key points
- **SSR (server-side rendering):** the server builds full HTML for each request. Fast first paint, good SEO, works before JS loads.
- **CSR (client-side rendering):** the browser renders with JavaScript. After the first page, SvelteKit navigations are CSR by default (no full reload).
- **Prerender (SSG):** HTML is built once at build time and served as a static file. Fastest, but everyone sees the same content.
- **Hydration:** the browser runs the same components over the server HTML, attaches event listeners and makes it interactive, without rebuilding the DOM ([glossary](https://svelte.dev/docs/kit/glossary)).
- **Per-route config:** export `ssr`, `csr`, `prerender` from `+page.ts`, `+page.server.ts` or a `+layout` file. Layout values are defaults for children ([page options](https://svelte.dev/docs/kit/page-options)).

## Example
```ts
// src/routes/(marketing)/+layout.ts
export const prerender = true;   // landing, pricing, about: static HTML at build time
```

```ts
// src/routes/(app)/+layout.ts
export const ssr = true;         // default; logged-in pages rendered per request
export const prerender = false;  // personal data must never be prerendered
```

```ts
// src/routes/(app)/terminal/+page.ts
// Pro trading terminal: heavy chart lib touches window at import time,
// no SEO value, behind login. Render only in the browser.
export const ssr = false;
```

```ts
// src/routes/legal/terms/+page.ts
export const prerender = true;
export const csr = false;        // ship zero JS: pure HTML + CSS
```

Better than `ssr = false` for one widget: keep SSR and load the browser-only library lazily.

```svelte
<script lang="ts">
  let chartEl: HTMLDivElement;

  $effect(() => {                       // effects only run in the browser
    let chart: { destroy(): void } | undefined;
    import('$lib/charting').then(({ createChart }) => {
      chart = createChart(chartEl);
    });
    return () => chart?.destroy();      // cleanup on unmount
  });
</script>

<div bind:this={chartEl} class="chart"></div>
```

## When to use it
| Mode | Setting | Fintech example |
| --- | --- | --- |
| SSR + hydrate (default) | nothing | portfolio, order history, stock detail pages |
| Prerender | `prerender = true` | marketing site, help center, blog |
| Prerender some, SSR the rest | `prerender = 'auto'` + `entries` | top 100 stock pages static, long tail SSR |
| Client only (SPA) | `ssr = false` | internal admin, trading terminal behind login |
| No JS | `csr = false` | terms, privacy, static docs |

## Likely questions
### What is the difference between SSR, CSR and prerendering?
SSR builds HTML on every request on the server, so the user sees content quickly and search engines can read it. CSR renders in the browser with JavaScript, so the first paint waits for JS, but navigation after that is fast. Prerendering is SSR done once at build time; it is the fastest to serve but the content is the same for every user. SvelteKit mixes them: SSR for the first load, CSR for every navigation after, and prerender for whatever routes you mark.

### What is hydration?
The server sends HTML plus the JS for the page. In the browser, Svelte runs the components again against the existing DOM, attaches event listeners and sets up state, instead of rebuilding the DOM. Data from load functions is serialized into the page so the browser does not refetch it. If the server and client render different markup (for example using `Date.now()` or `window` during render), you get a hydration mismatch.

### What do `ssr`, `csr` and `prerender` do?
- `ssr = false`: the server sends an empty shell and the page renders in the browser. Universal loads run only in the browser.
- `csr = false`: no JavaScript is shipped. Links do full page loads, forms cannot be enhanced, `<script>` blocks are removed.
- `prerender = true`: render at build time. `'auto'` means prerender what is found but still allow SSR for other params.
- If both `ssr` and `csr` are false, nothing renders.

### When would you disable SSR?
When the page truly cannot render on the server, for example a library that touches `window` at import time, and the page has no SEO value, like a logged-in trading terminal. I prefer to keep SSR and move the browser-only code into `$effect` or a dynamic `import()`, because disabling SSR costs a blank first paint and an extra round trip for data. Setting `ssr = false` in the root layout turns the whole app into an SPA.

### When can a page NOT be prerendered?
When two users hitting the same URL should see different content (personal portfolio, account). Also pages with form actions, since a server must handle the POST, and pages that read `url.searchParams` during load. Dynamic routes like `[slug]` can be prerendered if the crawler finds links to them or you list them with `entries`.

### How do you check if code runs in the browser?
Import `browser` from `$app/environment` (SvelteKit 2), or put the code in `$effect` / `onMount`, which never run on the server.

## Common mistakes
- Touching `window`, `localStorage` or `document` at the top of a component script. It crashes SSR.
- Prerendering pages with user-specific data. Every user would see the same HTML.
- Disabling SSR for the whole app just to fix one widget.
- Thinking `csr = false` still allows `use:enhance`. It does not; there is no JS.
- **SvelteKit 3 note:** `$app/environment` is renamed to `$app/env`.

## Resources
- [SvelteKit docs: Page options](https://svelte.dev/docs/kit/page-options) - `ssr`, `csr`, `prerender`, `entries`
- [SvelteKit docs: Glossary](https://svelte.dev/docs/kit/glossary) - SSR, CSR, hydration, SPA, SSG defined
- [web.dev: Rendering on the Web](https://web.dev/articles/rendering-on-the-web) - trade-offs of each strategy
- [Tutorial: ssr](https://svelte.dev/tutorial/kit/ssr) - interactive practice
