# Navigation

> **In one line:** SvelteKit intercepts normal `<a>` clicks to navigate without a full reload, you navigate from code with `goto`, you can load the next page early with `preloadData` or the `data-sveltekit-preload-data` attribute, and `$app/state` tells you the current page and whether a navigation is in progress.

## Key points
- **`goto(url, options)`** from `$app/navigation` navigates in code. Options: `replaceState`, `noScroll`, `keepFocus`, `invalidateAll`, `state` ([docs](https://svelte.dev/docs/kit/$app-navigation)).
- **`preloadData(href)`** loads that route's code and runs its `load` functions early, so the click feels instant. `preloadCode(pathname)` only loads the code.
- **Link preloading attribute.** `data-sveltekit-preload-data="hover"` (the default on `<body>` in `app.html`) starts loading when the user hovers or touches a link. Values: `"hover"`, `"tap"`, `"off"`. There is also `data-sveltekit-preload-code` (`"eager"`, `"viewport"`, `"hover"`, `"tap"`) ([link options](https://svelte.dev/docs/kit/link-options)).
- **`$app/state`** (SvelteKit 2.12+, replaces `$app/stores`) exports `page` (`url`, `params`, `data`, `form`, `status`, `error`), `navigating` and `updated`. They are reactive objects, read them directly with no `$` ([docs](https://svelte.dev/docs/kit/$app-state)).

## Example
```svelte
<script lang="ts">
  import { goto, preloadData } from '$app/navigation';
  import { page, navigating } from '$app/state';

  let query = $state('');

  async function search() {
    await goto(`/stocks?q=${encodeURIComponent(query)}`, { keepFocus: true, replaceState: true });
  }
</script>

<!-- current URL is reactive -->
<p>Viewing {page.url.pathname}</p>

{#if navigating.to}<div class="progress-bar"></div>{/if}

<input bind:value={query} onkeydown={(e) => e.key === 'Enter' && search()} />

<!-- preload on tap only: hover on a dense table would fire too many requests -->
<table data-sveltekit-preload-data="tap">
  <tr><td><a href="/stocks/AAPL">AAPL</a></td></tr>
</table>

<button onmouseenter={() => preloadData('/orders')} onclick={() => goto('/orders')}>
  Orders
</button>
```

## When to use it
- `goto` after login, or to sync filters/search into the URL.
- Preload on hover for the main nav; switch to `tap` (or `off`) on big watchlist tables so hovering does not fire hundreds of API calls.
- `navigating` for a top progress bar on slow pages.

## Likely questions
### How do you navigate programmatically?
`goto('/path')` from `$app/navigation`. It returns a promise that resolves when navigation finishes. For external URLs I use `window.location.href` instead, since `goto` is for app routes.

### How does link preloading work?
On hover (desktop) or touchstart (mobile), SvelteKit fetches the route's JS and runs its `load`. If the user clicks, the data is already there. It is set with `data-sveltekit-preload-data` on a link or any parent element.

### `$app/state` vs `$app/stores`?
`$app/state` is the newer API built on Svelte 5 runes. `page` is a plain reactive object, so I write `page.url` instead of `$page.url`. `$app/stores` still works but is deprecated.

## Resources
- [SvelteKit: $app/navigation](https://svelte.dev/docs/kit/$app-navigation) - `goto`, `preloadData`, `invalidate`
- [SvelteKit: Link options](https://svelte.dev/docs/kit/link-options) - all `data-sveltekit-*` attributes
- [SvelteKit: $app/state](https://svelte.dev/docs/kit/$app-state) - `page`, `navigating`, `updated`
