# Large lists and tables

> **In one line:** To show 50,000 rows, I never put 50,000 rows in the DOM: I virtualize the list so only visible rows render, and when the data itself is too big I paginate and filter on the server.

## Key points
- **The DOM is the bottleneck,** not JavaScript arrays. 50,000 rows with 8 cells each is 400,000+ nodes: slow first render, heavy memory, slow style and layout on every update.
- **Virtualization (windowing):** render only rows in the viewport plus an "overscan" buffer, inside a container whose height equals `rows x rowHeight` so the scrollbar is correct.
- **Pagination or infinite scroll:** load data in pages (for example 100 at a time). Use cursor-based pagination for live data so rows do not shift between pages.
- **Server-side filtering, sorting and search:** if the full dataset is large, let the backend do it and send back only the page you need. Debounce the search input.
- **Combine them:** server pages + client virtualization is the usual answer for a big trade history.

## Example
A fixed-row-height virtual list in Svelte 5:

```svelte
<script>
  let { rows, rowHeight = 32, height = 600 } = $props();
  let scrollTop = $state(0);
  const overscan = 5;

  let start = $derived(Math.max(0, Math.floor(scrollTop / rowHeight) - overscan));
  let end = $derived(
    Math.min(rows.length, Math.floor(scrollTop / rowHeight) + Math.ceil(height / rowHeight) + overscan)
  );
  let visible = $derived(rows.slice(start, end));
</script>

<div
  class="viewport"
  style:height="{height}px"
  onscroll={(e) => (scrollTop = e.currentTarget.scrollTop)}
>
  <!-- Spacer gives the real total height so the scrollbar is right -->
  <div style:height="{rows.length * rowHeight}px" style:position="relative">
    <div style:transform="translateY({start * rowHeight}px)">
      {#each visible as row (row.id)}
        <div class="row" style:height="{rowHeight}px">{row.symbol} {row.qty} {row.price}</div>
      {/each}
    </div>
  </div>
</div>

<style>
  .viewport { overflow-y: auto; contain: strict; }
</style>
```

The range math, checked with node for 50,000 rows, 32 px rows, a 600 px viewport:

```js
function visibleRange(scrollTop, viewportHeight, rowHeight, total, overscan = 5) {
  const first = Math.floor(scrollTop / rowHeight);
  const count = Math.ceil(viewportHeight / rowHeight);
  const start = Math.max(0, first - overscan);
  const end = Math.min(total, first + count + overscan);
  return { start, end, offsetY: start * rowHeight, totalHeight: total * rowHeight };
}
console.log(visibleRange(0, 600, 32, 50_000));
console.log(visibleRange(32_000, 600, 32, 50_000));
console.log(visibleRange(1_599_800, 600, 32, 50_000));
```

```bash
{ start: 0, end: 24, offsetY: 0, totalHeight: 1600000 }          # top: rows 0-23 only
{ start: 995, end: 1024, offsetY: 31840, totalHeight: 1600000 }  # middle: ~29 rows in DOM
{ start: 49988, end: 50000, offsetY: 1599616, totalHeight: 1600000 } # bottom: clamped to total
```

At any scroll position, fewer than 30 rows exist in the DOM.

Server-side filter and cursor pagination:

```js
// GET /api/trades?symbol=INFY&sort=-time&limit=100&cursor=eyJpZCI6MTIzfQ
async function loadPage({ symbol, cursor }) {
  const params = new URLSearchParams({ symbol, sort: '-time', limit: '100' });
  if (cursor) params.set('cursor', cursor);
  const res = await fetch(`/api/trades?${params}`);
  return res.json(); // { items: [...], nextCursor: '...' | null }
}
```

## When to use it
Trade history, order book depth, the full list of NSE/BSE instruments for a search dropdown, transaction statements, and admin tables. Virtualize when you have the data but too many rows to render. Paginate and filter on the server when the data itself is too big to download (millions of historical trades).

## Likely questions
### How would you render a table with 50,000 rows?
I would not render all of them. If the data fits in memory (50k rows of small objects is fine), I keep it in a plain array, ideally `$state.raw` so Svelte does not deep-proxy it, and virtualize the table so only about 30 to 50 visible rows are in the DOM. Sorting and filtering happen on the array, with heavy work in a Web Worker if needed. If the data is larger or must be fresh, I move filtering, sorting and pagination to the server.

### Virtualization vs pagination: which one?
Pagination is simple, accessible, works with server-side data and lets users jump to "page 7", but it breaks the flow of scanning. Virtualization gives smooth scrolling over a big list but is harder: variable row heights, keyboard navigation, and "find in page" (Ctrl+F) only finds rendered rows. For trade history I often combine infinite scroll that fetches pages with virtualization of what is loaded.

### How do you handle variable row heights?
Either estimate a height, render, then measure each row (with `ResizeObserver`) and cache the real heights to correct offsets; or design rows with a fixed height, which is much simpler and faster. For tables, fixed height is usually acceptable.

### Why server-side filtering instead of client-side?
If the dataset is huge, sending it all wastes bandwidth and memory, especially on low-end phones. The database can use indexes to filter and sort quickly. The client sends query params (`symbol`, `sort`, `cursor`), debounces the search box by around 300 ms, and cancels stale requests with `AbortController` so old results never overwrite new ones.

### Offset vs cursor pagination?
Offset (`?page=3`) is easy but if new rows arrive at the top, items shift and users see duplicates or miss rows. Cursor pagination (`?after=<last id>`) asks for "items after this one", which stays stable while new trades keep arriving. For live financial data, cursor is safer.

### What about accessibility in a virtual list?
Only rendered rows exist for screen readers, so set `aria-rowcount` on the table and `aria-rowindex` on each row so assistive tech knows the full size. Keep keyboard focus working when rows are recycled, and keep a pagination or "load more" alternative if possible.

## Common mistakes
- Virtualizing without keys, so recycled rows show wrong state.
- Forgetting the spacer, so the scrollbar is tiny and jumps.
- Running sort and filter on 50k rows on every keystroke without debounce.
- Using offset pagination on a list that grows in real time.
- Making 50k rows deeply reactive with `$state` when `$state.raw` would do.

## Resources
- [web.dev: Virtualize large lists](https://web.dev/articles/virtualize-long-lists-react-window) - concept explained with a React example; same idea in Svelte
- [web.dev: DOM size and interactivity](https://web.dev/articles/dom-size-and-interactivity) - why large DOMs hurt
- [MDN: CSS contain](https://developer.mozilla.org/en-US/docs/Web/CSS/contain) - isolate the scroll container's layout
- [Svelte docs: $state.raw](https://svelte.dev/docs/svelte/$state) - avoid deep proxies on big arrays
