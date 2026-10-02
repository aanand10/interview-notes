# Stock watchlist

> **In one line:** A watchlist keeps the list of symbols and the live prices in separate, keyed state, batches incoming ticks so the UI updates at most once per frame, and only re-renders the row whose price changed.

## Requirements to confirm
- Where do prices come from: a WebSocket feed, or polling? (Assume WebSocket; mock it with `setInterval` in the round.)
- Columns: symbol, last traded price (LTP), change, change %. Anything else (volume, high/low)?
- Add a symbol how: a search box, or just a text input? Max list size (for example 50)? Duplicates allowed? (No.)
- Sort by which columns? Should the order stay stable while prices move, or re-sort live?
- Persist the list: `localStorage` for the round, server in real life.
- Green/red means: change vs previous close (day change), plus a short flash on each tick up or down?

## Component breakdown
- `watchlist.svelte.js`: a small store class with the symbols, the quotes map, and actions (`add`, `remove`, `applyTicks`).
- `Watchlist.svelte`: add form, sort controls, table.
- `PriceCell` logic inline: shows LTP and flashes on change.
- `priceFeed`: subscribes to the socket and calls `applyTicks` with a batch.

## State and data flow
- `symbols`: ordered array of strings, the user's list.
- `quotes`: object keyed by symbol, `{ ltp, prevClose, dir }`. Keyed by symbol means an update is O(1) and only the row that reads `quotes[sym]` re-renders. Svelte 5 [$state](https://svelte.dev/docs/svelte/$state) proxies objects deeply, so this is fine-grained for free.
- `sorted` is [$derived](https://svelte.dev/docs/svelte/$derived) from symbols, quotes and the sort key.
- Feed flow: socket message, push into a buffer `Map` (latest tick per symbol wins), flush once per [requestAnimationFrame](https://developer.mozilla.org/en-US/docs/Web/API/Window/requestAnimationFrame). 200 ticks a second becomes at most 60 UI updates a second.

## Implementation
```js
// watchlist.svelte.js
export class Watchlist {
  symbols = $state(JSON.parse(localStorage.getItem('watchlist') ?? '["RELIANCE","TCS","INFY"]'));
  quotes = $state({}); // { TCS: { ltp, prevClose, dir } }

  add(raw) {
    const sym = raw.trim().toUpperCase();
    if (!sym) return 'Enter a symbol.';
    if (this.symbols.includes(sym)) return `${sym} is already in your watchlist.`;
    if (this.symbols.length >= 50) return 'Watchlist is full (50 max).';
    this.symbols.push(sym);
    this.save();
    return '';
  }

  remove(sym) {
    this.symbols = this.symbols.filter((s) => s !== sym);
    delete this.quotes[sym];
    this.save();
  }

  // batch: Map<symbol, { symbol, ltp, prevClose }>
  applyTicks(batch) {
    for (const t of batch.values()) {
      if (!this.symbols.includes(t.symbol)) continue; // ignore late ticks for removed symbols
      const old = this.quotes[t.symbol];
      const dir = !old ? 'none' : t.ltp > old.ltp ? 'up' : t.ltp < old.ltp ? 'down' : old.dir;
      this.quotes[t.symbol] = { ltp: t.ltp, prevClose: t.prevClose, dir };
    }
  }

  save() {
    localStorage.setItem('watchlist', JSON.stringify(this.symbols));
  }
}

// Collect ticks, flush at most once per frame. Latest tick per symbol wins.
export function createBatcher(apply) {
  let pending = new Map();
  let scheduled = false;
  return (tick) => {
    pending.set(tick.symbol, tick);
    if (scheduled) return;
    scheduled = true;
    requestAnimationFrame(() => {
      const batch = pending;
      pending = new Map();
      scheduled = false;
      apply(batch);
    });
  };
}
```

```svelte
<script>
  import { Watchlist, createBatcher } from './watchlist.svelte.js';

  const list = new Watchlist();
  let input = $state('');
  let error = $state('');
  let sortKey = $state('none'); // 'none' | 'symbol' | 'changePct'
  let sortDir = $state(1); // 1 asc, -1 desc

  const fmt = new Intl.NumberFormat('en-IN', { minimumFractionDigits: 2, maximumFractionDigits: 2 });
  const changePct = (q) => (q ? ((q.ltp - q.prevClose) / q.prevClose) * 100 : 0);

  const rows = $derived.by(() => {
    const out = [...list.symbols];
    if (sortKey === 'symbol') out.sort((a, b) => a.localeCompare(b) * sortDir);
    if (sortKey === 'changePct') out.sort((a, b) => (changePct(list.quotes[a]) - changePct(list.quotes[b])) * sortDir);
    return out;
  });

  function sortBy(key) {
    sortDir = sortKey === key ? -sortDir : 1;
    sortKey = key;
  }

  // Mock feed. In real life: WebSocket, subscribe to list.symbols.
  $effect(() => {
    const push = createBatcher((batch) => list.applyTicks(batch));
    const id = setInterval(() => {
      for (const symbol of list.symbols) {
        const old = list.quotes[symbol];
        const base = old?.ltp ?? 1000;
        push({ symbol, ltp: +(base + (Math.random() - 0.5) * 4).toFixed(2), prevClose: old?.prevClose ?? 1000 });
      }
    }, 250);
    return () => clearInterval(id);
  });

  function onsubmit(e) {
    e.preventDefault();
    error = list.add(input);
    if (!error) input = '';
  }
</script>

<form {onsubmit}>
  <label for="add-symbol">Add symbol</label>
  <input id="add-symbol" bind:value={input} aria-describedby="add-error" />
  <button type="submit">Add</button>
  <p id="add-error" role="alert">{error}</p>
</form>

{#if rows.length === 0}
  <p>Your watchlist is empty. Add a stock to start tracking it.</p>
{:else}
  <table>
    <thead>
      <tr>
        <th aria-sort={sortKey === 'symbol' ? (sortDir === 1 ? 'ascending' : 'descending') : 'none'}>
          <button onclick={() => sortBy('symbol')}>Symbol</button>
        </th>
        <th>LTP</th>
        <th aria-sort={sortKey === 'changePct' ? (sortDir === 1 ? 'ascending' : 'descending') : 'none'}>
          <button onclick={() => sortBy('changePct')}>Change %</button>
        </th>
        <th><span class="sr-only">Actions</span></th>
      </tr>
    </thead>
    <tbody>
      {#each rows as sym (sym)}
        {@const q = list.quotes[sym]}
        {@const pct = changePct(q)}
        <tr>
          <td>{sym}</td>
          <td>
            {#key q?.ltp}
              <span class="flash-{q?.dir ?? 'none'}">{q ? fmt.format(q.ltp) : '--'}</span>
            {/key}
          </td>
          <td class={pct >= 0 ? 'gain' : 'loss'}>
            {pct >= 0 ? '+' : ''}{pct.toFixed(2)}%
            <span class="sr-only">{pct >= 0 ? 'up' : 'down'}</span>
          </td>
          <td><button onclick={() => list.remove(sym)} aria-label={`Remove ${sym}`}>x</button></td>
        </tr>
      {/each}
    </tbody>
  </table>
{/if}

<style>
  .gain { color: #0a7d32; }
  .loss { color: #c62828; }
  td { font-variant-numeric: tabular-nums; } /* digits same width, no jitter */
  .flash-up { animation: up 0.6s; }
  .flash-down { animation: down 0.6s; }
  @keyframes up { from { background: #c8f7d4; } }
  @keyframes down { from { background: #fbd0d0; } }
  @media (prefers-reduced-motion: reduce) { .flash-up, .flash-down { animation: none; } }
</style>
```

Batcher tested in node (3 ticks in one frame, then 1 later):

```js
push({ symbol: 'TCS', ltp: 4100 });
push({ symbol: 'INFY', ltp: 1500 });
push({ symbol: 'TCS', ltp: 4101 });
// 40 ms later: push({ symbol: 'INFY', ltp: 1501 })
// Output:
// flush TCS:4101 INFY:1500   <- one flush, TCS 4100 was replaced by the newer tick
// flush INFY:1501
```

## Edge cases and accessibility
- **Duplicates and bad input**: trim and uppercase, reject duplicates, cap list size.
- **Late ticks** for a removed symbol are ignored, so the row does not come back.
- **No price yet**: show `--`, not `0.00` (a zero price looks like a crash).
- **Colour is not the only signal**: add `+`/`-` sign and hidden "up"/"down" text for colour-blind and screen reader users.
- **Do not announce every tick**: no `aria-live` on the price table, it would spam screen readers.
- **Live re-sort** makes rows jump under the mouse. Offer "sort once" or re-sort on a slower interval (for example every 2 seconds).
- **Tab hidden**: pause UI flushes (or the whole socket subscription) using the [Page Visibility API](https://developer.mozilla.org/en-US/docs/Web/API/Page_Visibility_API).
- **Number format**: [Intl.NumberFormat](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/NumberFormat) with `en-IN` gives Indian grouping (1,00,000.00).

## What interviewers look for
- Normalised state: list of ids plus a map of quotes, not an array of objects you replace on every tick.
- Keyed `{#each}` so rows are reused, not recreated. See [each blocks](https://svelte.dev/docs/svelte/each).
- Batching or throttling of ticks; understanding that the socket can be faster than the screen.
- Cleanup of intervals and sockets in `$effect` return.

## Likely questions
### How do you make updates efficient with hundreds of ticks per second?
Three things. One, keep quotes in a map keyed by symbol so an update touches only one entry. Two, buffer ticks and flush once per animation frame, keeping only the latest tick per symbol. Three, use a keyed each block so Svelte updates just the text node in that row. If the list is huge, I add virtualization.

### How do you show green and red?
Day change colour comes from LTP vs previous close. The flash comes from comparing the new tick to the old one; I store `dir` and restart a CSS animation with `{#key}`. I respect `prefers-reduced-motion`.

### Where would the socket subscription live in a real app?
In a shared service, not in the component. The watchlist tells the service "I need these symbols", and the service manages one socket for the whole app and reference-counts subscriptions. See the trading dashboard note.

## Resources
- [MDN: WebSocket](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket) - the live feed API
- [MDN: requestAnimationFrame](https://developer.mozilla.org/en-US/docs/Web/API/Window/requestAnimationFrame) - flush once per frame
- [Svelte: $state](https://svelte.dev/docs/svelte/$state) - deep reactivity and classes
- [Svelte: each blocks](https://svelte.dev/docs/svelte/each) - keyed lists
