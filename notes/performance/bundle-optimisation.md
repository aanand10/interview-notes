# Bundle optimisation

> **In one line:** Ship less JavaScript, and ship it only when it is needed: split code by route, lazy-load heavy features, let tree shaking drop unused code, replace heavy libraries, and target modern browsers.

## Key points
- **Code splitting by route:** each page gets its own chunk, so the login page does not download the charting code. SvelteKit does this automatically per route.
- **Dynamic `import()`:** load a heavy feature (chart library, PDF export, rich editor) only when the user needs it. See [MDN: import()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/import).
- **Tree shaking:** the bundler removes exports you never import. It works best with ES modules, named imports and packages marked side-effect free. See [MDN: Tree shaking](https://developer.mozilla.org/en-US/docs/Glossary/Tree_shaking).
- **Bundle analyzer:** a visual map of what is inside each chunk (for Vite/Rollup, `rollup-plugin-visualizer`; for webpack, `webpack-bundle-analyzer`). You cannot fix what you cannot see.
- **Modern builds:** target modern browsers so you ship native `async/await`, classes and modules instead of large transpiled code and polyfills.

## Example
Lazy-load a heavy chart only when the user opens the chart tab (Svelte 5):

```svelte
<script>
  let showChart = $state(false);
  let ChartModule = $state(null);

  async function openChart() {
    showChart = true;
    // Separate chunk, downloaded only on first click
    ChartModule ??= (await import('$lib/CandleChart.svelte')).default;
  }
</script>

<button onclick={openChart}>Show chart</button>

{#if showChart}
  {#if ChartModule}
    <ChartModule symbol="INFY" />
  {:else}
    <p>Loading chart...</p>
  {/if}
{/if}
```

Tree-shake friendly imports and lighter replacements:

```js
// Bad: pulls the whole library (CommonJS, not tree-shakeable)
import _ from 'lodash';
const total = _.sumBy(holdings, 'value');

// Better: ES module build, only this function is bundled
import { sumBy } from 'lodash-es';

// Best: no dependency at all
const total2 = holdings.reduce((sum, h) => sum + h.value, 0);

// Replace a heavy date/number library with the built-in Intl API
const inr = new Intl.NumberFormat('en-IN', { style: 'currency', currency: 'INR' });
inr.format(1234567.5); // "₹12,34,567.50"
```

Analyze the bundle in a Vite or SvelteKit project:

```js
// vite.config.js
import { sveltekit } from '@sveltejs/kit/vite';
import { visualizer } from 'rollup-plugin-visualizer';

export default {
  plugins: [sveltekit(), visualizer({ filename: 'stats.html', gzipSize: true })],
  build: { target: 'es2022' }, // modern output, no legacy transpiling
};
```

## When to use it
When Lighthouse flags "Reduce unused JavaScript", when the Coverage tab shows most loaded JS is unused, or when INP and load are poor on mid-range Android phones. In a trading app, the order form and watchlist must load fast, while charting libraries, report exports and KYC upload flows can be lazy-loaded. E.g. in my last project, splitting heavy code out of the first load was part of how we got about 300 ms off page load.

## Likely questions
### How would you reduce the bundle size of an app?
First I measure with a bundle analyzer and the Coverage tab to see what is big and what is unused. Then: split by route, lazy-load heavy features with dynamic `import()`, fix imports so tree shaking works, replace or remove heavy dependencies, and target modern browsers. Finally I add a size budget in CI so it does not grow back.

### What is code splitting and how does it work?
The bundler breaks the app into multiple chunks instead of one big file. Every dynamic `import()` becomes a split point: that module and its dependencies go into a separate file loaded on demand. Route-based splitting is the most common: frameworks like SvelteKit create a chunk per page and preload it when needed. See [web.dev: code splitting](https://web.dev/articles/reduce-javascript-payloads-with-code-splitting).

### What is tree shaking and when does it fail?
Tree shaking is dead-code removal based on ES module `import`/`export`, which are static so the bundler can see what is used. It fails with CommonJS (`require`), with `import *` used dynamically, and with modules that have side effects at the top level, because the bundler cannot safely remove them. Libraries help by shipping ESM and setting `"sideEffects": false` in `package.json`. See [web.dev: tree shaking](https://web.dev/articles/reduce-javascript-payloads-with-tree-shaking).

### How do you find and remove heavy dependencies?
Run the analyzer and sort by size. Common culprits: moment.js with all locales, full lodash, big icon packs imported as a whole, and chart libraries loaded on every page. Fixes: use native APIs (`Intl`, `Array` methods, `structuredClone`), import single icons, choose smaller libraries, or lazy-load. Also check for duplicate versions of the same package.

### What is a "modern build"?
It means compiling for browsers that support ES modules and modern syntax, so we do not ship transpiled code and polyfills for features every current browser has. This makes JS smaller and faster to parse. If old browsers matter, you can serve a separate legacy bundle only to them. See [web.dev: publish modern JavaScript](https://web.dev/articles/publish-modern-javascript).

### Can code splitting make things slower?
Yes, if overdone. Too many tiny chunks mean many requests and a waterfall where chunk A loads, then asks for chunk B. Lazy-loading something needed right away delays it. The fix is to split at meaningful boundaries (routes, heavy optional features) and preload chunks you know will be needed, for example on hover of a link.

## Common mistakes
- Lazy-loading things that are needed for first paint, which hurts LCP.
- Only looking at minified size. Look at gzip/brotli size for download and raw size for parse time.
- Assuming tree shaking works on any library.
- Optimising bundle size but ignoring that a small library can still be slow at runtime.

## Resources
- [web.dev: Reduce JavaScript payloads with code splitting](https://web.dev/articles/reduce-javascript-payloads-with-code-splitting) - how and where to split
- [web.dev: Reduce JavaScript payloads with tree shaking](https://web.dev/articles/reduce-javascript-payloads-with-tree-shaking) - why ESM matters
- [javascript.info: Dynamic imports](https://javascript.info/modules-dynamic-imports) - simple explanation of `import()`
- [SvelteKit: Performance](https://svelte.dev/docs/kit/performance) - what SvelteKit does for you out of the box
- [Chrome DevTools: Coverage](https://developer.chrome.com/docs/devtools/coverage) - find unused JS and CSS
