# Autocomplete / search

> **In one line:** An autocomplete waits until the user pauses typing (debounce), cancels any older request that is still running, caches results, and lets the user move through suggestions with the keyboard like a proper combobox.

## Requirements to confirm
- Data source: a server API or a local list? How big is it? (Local and small means filter in memory, no debounce needed.)
- Minimum characters before searching (often 1 or 2)? Debounce delay (250 to 300 ms is common)?
- What happens on select: fill the input, navigate to a stock page, or add to a watchlist?
- Max results to show? Should matches be highlighted?
- Show recent searches when the box is empty?

## Component breakdown
- `SymbolSearch.svelte`: input, listbox, all keyboard handling.
- `searchSymbols(query, signal)`: API function that accepts an [AbortSignal](https://developer.mozilla.org/en-US/docs/Web/API/AbortController).
- `splitMatch(text, query)`: pure helper that splits a label into matched and unmatched parts for highlighting. Pure functions are easy to unit test.

## State and data flow
- `query` (what is typed), `results`, `status` (`idle` / `loading` / `success` / `error`), `activeIndex` (highlighted option, `-1` for none), `open` (is the list shown).
- Typing changes `query`. An `$effect` waits 300 ms, then searches. If the user types again, the effect's cleanup clears the timer and aborts the old request. See [$effect](https://svelte.dev/docs/svelte/$effect).
- A `Map` cache stores results by normalised query, so going back from "rel" to "re" is instant.

## Implementation
```svelte
<script>
  let { onselect } = $props();

  const cache = new Map(); // query -> results (plain Map, no need to be reactive)

  let query = $state('');
  let results = $state([]);
  let status = $state('idle');
  let activeIndex = $state(-1);
  let open = $state(false);

  async function searchSymbols(q, signal) {
    const res = await fetch(`/api/symbols?q=${encodeURIComponent(q)}`, { signal });
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    return res.json(); // [{ symbol: 'RELIANCE', name: 'Reliance Industries' }]
  }

  $effect(() => {
    const q = query.trim().toLowerCase();
    activeIndex = -1;
    if (q.length < 2) {
      results = [];
      status = 'idle';
      return;
    }
    if (cache.has(q)) {
      results = cache.get(q);
      status = 'success';
      return;
    }
    const controller = new AbortController();
    const timer = setTimeout(async () => {
      status = 'loading';
      try {
        const data = await searchSymbols(q, controller.signal);
        cache.set(q, data);
        results = data;
        status = 'success';
      } catch (err) {
        if (err.name !== 'AbortError') status = 'error'; // aborts are expected, not errors
      }
    }, 300);
    // Runs before the next effect run and on unmount
    return () => {
      clearTimeout(timer);
      controller.abort();
    };
  });

  function splitMatch(text, q) {
    const needle = q.trim().toLowerCase();
    if (!needle) return [{ text, match: false }];
    const parts = [];
    const lower = text.toLowerCase();
    let i = 0;
    while (i < text.length) {
      const hit = lower.indexOf(needle, i);
      if (hit === -1) { parts.push({ text: text.slice(i), match: false }); break; }
      if (hit > i) parts.push({ text: text.slice(i, hit), match: false });
      parts.push({ text: text.slice(hit, hit + needle.length), match: true });
      i = hit + needle.length;
    }
    return parts;
  }

  function choose(item) {
    query = item.symbol;
    open = false;
    onselect?.(item);
  }

  function onkeydown(e) {
    if (e.key === 'ArrowDown') {
      e.preventDefault(); // stop the cursor jumping to the end of the input
      open = true;
      if (results.length) activeIndex = (activeIndex + 1) % results.length;
    } else if (e.key === 'ArrowUp') {
      e.preventDefault();
      if (results.length) activeIndex = (activeIndex - 1 + results.length) % results.length;
    } else if (e.key === 'Enter' && open && activeIndex >= 0) {
      e.preventDefault();
      choose(results[activeIndex]);
    } else if (e.key === 'Escape') {
      open = false;
      activeIndex = -1;
    }
  }
</script>

<label for="symbol-search">Search stocks</label>
<input
  id="symbol-search"
  role="combobox"
  autocomplete="off"
  aria-expanded={open && results.length > 0}
  aria-controls="symbol-listbox"
  aria-autocomplete="list"
  aria-activedescendant={activeIndex >= 0 ? `option-${activeIndex}` : undefined}
  bind:value={query}
  oninput={() => (open = true)}
  {onkeydown}
  onblur={() => setTimeout(() => (open = false), 100)}
/>

<p class="sr-only" role="status">
  {#if status === 'success'}{results.length} results{/if}
</p>

{#if open && query.trim().length >= 2}
  {#if status === 'loading'}
    <p>Searching...</p>
  {:else if status === 'error'}
    <p>Could not load results. Keep typing to retry.</p>
  {:else if status === 'success' && results.length === 0}
    <p>No stocks match "{query}".</p>
  {/if}
  <ul id="symbol-listbox" role="listbox">
    {#each results as item, i (item.symbol)}
      <!-- keys are handled on the input (combobox pattern), so this warning does not apply -->
      <!-- svelte-ignore a11y_click_events_have_key_events -->
      <li
        id={`option-${i}`}
        role="option"
        aria-selected={i === activeIndex}
        onpointerdown={(e) => e.preventDefault()}
        onclick={() => choose(item)}
      >
        {#each splitMatch(item.symbol, query) as part}
          {#if part.match}<mark>{part.text}</mark>{:else}{part.text}{/if}
        {/each}
        <small>{item.name}</small>
      </li>
    {/each}
  </ul>
{/if}
```

The cache plus abort logic, tested in node with a fake fetch where longer queries are slower:

```js
search('rel'); // starts, slow
search('re');  // aborts 'rel', starts 're'
// Output:
// rel -> AbortError
// re -> [ 'RE' ]
// cached: [ 'RE' ]   (search('RE') again hits the cache, no request)
```

## Edge cases and accessibility
- **Stale responses**: without abort, a slow "re" response can arrive after a fast "rel" response and overwrite it. Aborting the old request (or checking the request id) fixes this race.
- **Highlighting without XSS**: I split the string and render `<mark>` in markup. I never build HTML strings and use `{@html}`. A query like `(` is also safe because I use `indexOf`, not a `RegExp` built from user input.
- **Combobox pattern**: focus stays in the input; `aria-activedescendant` tells the screen reader which option is active. Follow the [ARIA combobox pattern](https://www.w3.org/WAI/ARIA/apg/patterns/combobox/).
- **Clicking an option**: `onpointerdown` with `preventDefault` stops the input from blurring before the click lands.
- **Result count** is announced through a `role="status"` live region.
- **Empty, loading, error** states are all different messages. Error should not wipe the old results if you prefer "last good results".
- **Cache size**: cap it (for example 50 entries, delete the oldest key from the Map) and give it a TTL if data changes.

## What interviewers look for
- Debounce done right, with cleanup on every keystroke and on unmount.
- Race condition awareness: AbortController or "ignore if not latest".
- Full keyboard support: Up, Down, Enter, Escape, wrap-around.
- Clear empty, loading and error states. Cache for repeated queries.

## Likely questions
### Debounce or throttle for search?
Debounce. I only want to search once the user pauses. Throttle would fire every N ms while typing, which still sends requests for half-typed words.

### How do you cancel stale requests?
I create an `AbortController` per request and call `abort()` when a new query starts. `fetch` then rejects with an `AbortError`, which I ignore. See [javascript.info: Fetch abort](https://javascript.info/fetch-abort). If the API cannot be aborted, I keep a request counter and only apply the response if its id is still the latest.

### Why is the cache a plain Map and not $state?
The UI never renders the cache directly. It only renders `results`. Making it reactive would add tracking cost for nothing.

### How would you cache across the whole app?
Move the cache into a module, key by normalised query, add a TTL, and cap the size (LRU). For a SvelteKit app, the server can also send `Cache-Control` headers so the browser caches the API response.

## Resources
- [W3C APG: Combobox pattern](https://www.w3.org/WAI/ARIA/apg/patterns/combobox/) - exact keyboard and ARIA rules
- [MDN: combobox role](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Reference/Roles/combobox_role) - attributes explained
- [MDN: AbortController](https://developer.mozilla.org/en-US/docs/Web/API/AbortController) - cancel fetch
- [Svelte: $effect](https://svelte.dev/docs/svelte/$effect) - cleanup functions for timers and requests
