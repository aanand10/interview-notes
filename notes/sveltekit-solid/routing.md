# Routing

> **In one line:** SvelteKit uses file-based routing: every folder in `src/routes` is a URL segment, and special `+` files in that folder decide what the route renders, loads and serves.

## Key points
- **Folder = URL.** `src/routes/stocks/[symbol]` becomes `/stocks/AAPL`. Square brackets mark a dynamic param ([routing docs](https://svelte.dev/docs/kit/routing)).
- **`+` files have jobs.** `+page.svelte` is the UI, `+page.ts` / `+page.server.ts` load data, `+layout.svelte` wraps child pages, `+error.svelte` shows errors, `+server.ts` is an API endpoint.
- **Layouts nest.** A `+layout.svelte` wraps every page below its folder and must render `{@render children()}` where the page goes. Layouts stay mounted when you move between child pages.
- **Advanced params.** `[[optional]]`, `[...rest]` and matchers like `[id=integer]` cover the tricky URLs ([advanced routing](https://svelte.dev/docs/kit/advanced-routing)).
- **Route groups `(name)`** let you share a layout between routes without adding a URL segment.

## Example
A trading app folder tree:

```bash
src/routes/
├ +layout.svelte              # root shell: header, theme
├ +error.svelte               # root error page
├ (marketing)/                # group: not part of the URL
│ ├ +layout.svelte            # marketing layout (big footer)
│ ├ +page.svelte              # /
│ └ pricing/+page.svelte      # /pricing
├ (app)/                      # group: logged-in layout
│ ├ +layout.server.ts         # loads user for every app page
│ ├ +layout.svelte            # sidebar + watchlist
│ ├ portfolio/+page.svelte    # /portfolio
│ └ stocks/[symbol]/
│   ├ +page.ts                # load quote for this symbol
│   └ +page.svelte            # /stocks/AAPL
├ [[lang]]/help/+page.svelte  # /help and /en/help
├ docs/[...path]/+page.svelte # /docs/a/b/c -> path = "a/b/c"
└ api/quotes/+server.ts       # GET /api/quotes (JSON endpoint)
```

```svelte
<!-- src/routes/(app)/+layout.svelte -->
<script lang="ts">
  import type { LayoutProps } from './$types';
  let { data, children }: LayoutProps = $props();
</script>

<aside>Hi {data.user.name}</aside>
<main>{@render children()}</main>
```

```svelte
<!-- src/routes/(app)/stocks/[symbol]/+page.svelte -->
<script lang="ts">
  import type { PageProps } from './$types';
  let { data }: PageProps = $props();   // data comes from +page.ts load
</script>

<h1>{data.symbol}: {data.price}</h1>
```

```ts
// src/routes/(app)/stocks/[symbol]/+page.ts
import type { PageLoad } from './$types';

export const load: PageLoad = async ({ params, fetch }) => {
  const res = await fetch(`/api/quotes?symbol=${params.symbol}`);
  return { symbol: params.symbol, price: (await res.json()).price };
};
```

## When to use it
- Dynamic `[symbol]` for stock detail pages, `[orderId]` for order details.
- `(app)` vs `(marketing)` groups to give logged-in pages a sidebar layout and public pages a landing layout.
- `[...path]` for docs or a custom nested 404 page.
- `[[lang]]` for optional locale prefixes.

## Likely questions
### What is the difference between `+page.svelte` and `+layout.svelte`?
`+page.svelte` is the content for one URL. `+layout.svelte` wraps all pages in its folder and below, so it is the place for shared UI like a nav bar or a watchlist sidebar. The layout renders the page through the `children` snippet with `{@render children()}`. When I navigate between two pages under the same layout, the layout is not destroyed, so its state survives.

### How do dynamic routes work?
A folder named `[id]` matches any single segment, and the value shows up as `params.id` in `load` functions (and as the `params` prop on the page in newer 2.x versions). You can have several params in one path, like `[org]/[repo]`. You can even mix text and params in one segment, like `foo-[slug]`.

### What are optional and rest params?
- `[[lang]]` is optional: `/help` and `/en/help` both hit the same page; `params.lang` is `undefined` in the first case.
- `[...path]` is a rest param: it matches zero or more segments, so `/docs/a/b/c` gives `path = "a/b/c"`. It also matches `/docs` with an empty value, so validate it.
- An optional param cannot come right after a rest param, because rest is greedy.

### What are route groups and why use them?
A folder in parentheses like `(app)` is ignored in the URL. It only exists to group routes so they can share a `+layout.svelte` or `+layout.server.ts`. Typical use: `(auth)` pages with a minimal layout, `(app)` pages with the full dashboard layout, both at the root of the URL space.

### How does SvelteKit pick a route when two match?
More specific routes win. A static segment beats a param, a param with a matcher (`[id=integer]`) beats a plain param, and optional or rest params get the lowest priority. Ties go alphabetically. So `/stocks/new` hits `stocks/new` before `stocks/[symbol]`.

### How do you validate a param, for example only numbers?
Use a param matcher. In SvelteKit 2 you create `src/params/integer.ts` exporting `match(param)`, then name the folder `[id=integer]`. If the matcher returns false, SvelteKit tries other routes and finally returns 404.

```ts
// src/params/integer.ts  (SvelteKit 2)
import type { ParamMatcher } from '@sveltejs/kit';
export const match: ParamMatcher = (param) => /^\d+$/.test(param);
```

### Can a page escape its parent layout?
Yes. `+page@.svelte` resets to the root layout, and `+page@(app).svelte` resets to the `(app)` layout. Layouts can do the same with `+layout@.svelte`. Useful for a full-screen chart or embed page inside the app group.

## Common mistakes
- Forgetting `{@render children()}` in a layout, so pages never show. (Svelte 4 used `<slot />`.)
- Expecting `+layout.svelte` to apply to `+server.ts` endpoints. It does not; use the `handle` hook for that.
- Putting auth only in a layout `load`. Layout loads do not rerun on every client navigation; use `hooks.server.ts` for auth guards.
- Using a framework `<Link>`: SvelteKit uses plain `<a href>`.
- **SvelteKit 3 note** (released 1 Oct 2026): param matchers moved to a single `src/params.ts` with `defineParams`, and `$lib` became `#lib`. Most teams are still on 2.x, but it is a good thing to know.

## Resources
- [SvelteKit docs: Routing](https://svelte.dev/docs/kit/routing) - all the `+` files
- [SvelteKit docs: Advanced routing](https://svelte.dev/docs/kit/advanced-routing) - rest, optional, matchers, groups, `@` resets
- [Tutorial: Optional params](https://svelte.dev/tutorial/kit/optional-params) - interactive practice
- [Tutorial: Route groups](https://svelte.dev/tutorial/kit/route-groups) - interactive practice
