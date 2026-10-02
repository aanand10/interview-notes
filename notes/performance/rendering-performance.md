# Rendering performance

> **In one line:** Fast rendering means doing as little DOM work as possible: update only what changed, give lists stable keys, render only what is on screen, and avoid making huge data deeply reactive.

## Key points
- **Avoid unnecessary updates.** Svelte compiles to direct DOM updates and tracks dependencies precisely, so the main risk is not "re-rendering" like React but triggering more reactive work than needed: big objects replaced every tick, `$effect`s that write state, and derived values recomputed too often.
- **Keyed lists.** In `{#each items as item (item.id)}` the key tells Svelte which DOM node belongs to which item, so reordering moves nodes instead of rewriting them. See [Svelte: each blocks](https://svelte.dev/docs/svelte/each).
- **Virtualization for long lists.** Only render the rows visible in the viewport (plus a few extra). 50 DOM rows instead of 50,000.
- **`$state.raw` for large data.** Normal `$state` wraps objects and arrays in a deep proxy so every nested property is reactive. `$state.raw` skips that; you update it by reassigning the whole value. Cheaper for large, mostly read-only data like API responses or chart series. See [Svelte: $state](https://svelte.dev/docs/svelte/$state).
- **Browser rendering pipeline:** JavaScript, then style, layout, paint, composite. Anything that triggers layout is expensive; `transform` and `opacity` can skip layout and paint. See [web.dev: rendering performance](https://web.dev/articles/rendering-performance).

## Example
Large API data with `$state.raw`, a `$derived` filter, and a keyed list:

```svelte
<script>
  let { initialRows } = $props();

  // 5,000 trade rows: no deep proxy, replaced as a whole when new data arrives
  let trades = $state.raw(initialRows);
  let query = $state('');

  // Recomputed only when `trades` or `query` change
  let visible = $derived(
    trades.filter((t) => t.symbol.includes(query.toUpperCase())).slice(0, 100)
  );

  async function refresh() {
    const res = await fetch('/api/trades');
    trades = await res.json(); // reassign, do not mutate (raw state is not deep)
  }
</script>

<input bind:value={query} placeholder="Filter symbol" />
<button onclick={refresh}>Refresh</button>

<ul>
  {#each visible as trade (trade.id)}
    <li>{trade.symbol} {trade.qty} @ {trade.price}</li>
  {/each}
</ul>
```

The trap with raw state:

```js
let trades = $state.raw([]);
trades.push(newTrade);           // NOT reactive: UI does not update
trades = [...trades, newTrade];  // reactive: new array, Svelte sees the change
```

## When to use it
Order books, trade history, watchlists and holdings tables are all large and update often. Use keyed each blocks everywhere a list can reorder (sorting by % change). Use `$state.raw` for snapshot data from the server, like a 5,000-row trade history or chart candles, that you replace rather than edit field by field. E.g. in my last project, cutting unnecessary updates is how we got about a 40% reduction in render time, and we confirmed it with before/after Performance traces.

## Likely questions
### How do you avoid unnecessary re-renders or updates?
First I confirm it in the profiler instead of guessing. Then I make sure state is as granular as needed: components read only the data they show, so a change in one price does not touch unrelated parts. I use `$derived` for computed values instead of `$effect` that writes to state, because effects that set state cause extra update cycles. I avoid replacing big objects when only one field changed, and I keep expensive calculations out of the template. In React the equivalents are `memo`, `useMemo` and stable props; in Svelte fine-grained reactivity does most of this for you.

### Why do keys matter in lists?
Without a key, Svelte updates list items by index. If you sort or insert at the top, every row's content is rewritten, and any local state, focus or animation sticks to the wrong row. With a stable unique key, like `trade.id`, Svelte moves the existing DOM nodes, which is faster and correct. Never use the array index as the key for lists that reorder.

### What is `$state.raw` and when would you use it?
`$state` makes objects deeply reactive using a proxy, which costs memory and time for large data. `$state.raw` stores the value as-is: it is only reactive when you reassign the variable. I use it for large datasets that I replace as a whole, like API results, chart data or a big table snapshot. The trade-off is you cannot mutate it (`push`, `obj.x = 1`) and expect the UI to update.

### What is virtualization?
It means rendering only the items in the visible viewport plus a small buffer, and using a tall spacer so the scrollbar still looks right. As you scroll, the same small set of DOM nodes shows different data. It turns a 50,000-row list into about 30 to 50 DOM nodes. See the [large lists and tables](large-lists-and-tables.md) note.

### What is layout thrashing?
It is when code alternates reading layout (like `offsetHeight`) and writing styles in a loop. Each read forces the browser to recalculate layout synchronously. The fix is to batch all reads first, then all writes, or do writes in `requestAnimationFrame`. See [web.dev: avoid layout thrashing](https://web.dev/articles/avoid-large-complex-layouts-and-layout-thrashing).

### How do you find rendering problems?
In the Performance panel, look for long purple (style/layout) and green (paint) blocks and "Forced reflow" warnings. Turn on "Paint flashing" in the Rendering drawer to see which areas repaint on every update. Large DOM size is also a red flag; Lighthouse warns about it.

## Common mistakes
- Using index as key in sortable lists.
- Using `$effect` to sync one state into another instead of `$derived`.
- Mutating `$state.raw` data and wondering why the UI did not update.
- Rendering thousands of rows "because Svelte is fast". The DOM is still the bottleneck.
- Animating `width`, `top` or `left` instead of `transform`.

## Resources
- [Svelte docs: $state (including $state.raw)](https://svelte.dev/docs/svelte/$state) - deep vs raw reactivity
- [Svelte docs: each blocks](https://svelte.dev/docs/svelte/each) - keyed each syntax
- [Svelte docs: $derived](https://svelte.dev/docs/svelte/$derived) - computed values without extra effects
- [web.dev: Rendering performance](https://web.dev/articles/rendering-performance) - the pixel pipeline explained
- [web.dev: DOM size and interactivity](https://web.dev/articles/dom-size-and-interactivity) - why big DOMs hurt INP
