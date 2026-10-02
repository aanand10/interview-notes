# `$derived`

> **In one line:** `$derived` declares a value computed from other state; Svelte tracks what it reads, recomputes it lazily only when a dependency changed and someone reads it, and caches it otherwise.

## Key points
- `let total = $derived(price * qty)` takes an **expression**. `$derived.by(() => { ... })` takes a **function** for multi-line logic (loops, `if`, early return). `$derived(expr)` is the same as `$derived.by(() => expr)`.
- **Dependencies are automatic:** anything read synchronously inside it. No dependency array like React's `useMemo`.
- **Lazy and cached (push-pull):** when state changes, deriveds are only *marked dirty* (push). They recompute the next time they are read (pull). If nobody reads it, it never runs.
- **Skips no-op updates:** if the new value is identical (`===`) to the old one, things downstream do not update.
- Must be **pure**: no side effects, and Svelte errors if you set state inside it. Since Svelte 5.25 you can **override** a derived by assigning to it (handy for optimistic UI); it goes back to computing when a dependency changes.

## Example
Run in Node with the real Svelte compiler (`.svelte.js` module):

```js
let price = $state(100);
let qty = $state(2);
let total = $derived.by(() => {
  console.log('  computing total');
  return price * qty;
});

console.log('before read');
console.log('total =', total);
console.log('total again =', total);   // cached, no recompute
price = 110;                           // only marks dirty
console.log('total after price change =', total);

// Overriding a derived (Svelte 5.25+)
let likes = $state(5);
let shown = $derived(likes);
shown = 6;  console.log('shown overridden =', shown);
likes = 10; console.log('shown after source change =', shown);
```

```bash
before read                 # declaring the derived did not compute it (lazy)
  computing total           # first read computes
total = 200
total again = 200           # cached: no "computing" line
  computing total           # recomputed only because price changed AND we read it
total after price change = 220
shown overridden = 6        # temporary override
shown after source change = 10   # dependency changed, so it recomputes
```

A Svelte component for an order ticket:

```svelte
<script>
  let qty = $state(10);
  let price = $state(182.5);
  let side = $state('buy');

  let notional = $derived(qty * price);
  let warnings = $derived.by(() => {
    const list = [];
    if (qty <= 0) list.push('Quantity must be positive');
    if (notional > 50_000) list.push('Large order, needs confirmation');
    return list;
  });
</script>

<input type="number" bind:value={qty} />
<p>{side.toUpperCase()} notional: {notional.toFixed(2)}</p>
{#each warnings as w}<p class="warn">{w}</p>{/each}
```

## When to use it
- Totals, P&L, filtered or sorted watchlists, validation messages, "is form valid" flags.
- Any value you could compute from other state. If you can compute it, derive it; don't store it.

## Likely questions
### What is the difference between `$derived` and `$derived.by`?
Only the input shape. `$derived` takes one expression, `$derived.by` takes a function so you can write statements. Use `$derived.by` for loops, `switch`, or several steps. Both are lazy, cached and auto-tracked.

### Why is `$derived` better than setting state inside an `$effect`?
Four reasons. First, it is always correct: the value is computed when read, so there is no moment where `doubled` is stale. With an effect, the effect runs *after* the change (in a microtask, after DOM work), so you get an extra update and briefly stale UI. Second, it is lazy: an effect runs even if nobody reads the value. Third, effects that write state can cause loops and are hard to follow. Fourth, effects do not run on the server, so effect-computed values are missing in SSR. The docs call `$effect` an escape hatch.

```svelte
<script>
  let count = $state(0);
  // Bad: extra state, runs after render, not on the server
  // let doubled = $state(0);
  // $effect(() => { doubled = count * 2; });

  // Good
  let doubled = $derived(count * 2);
</script>
```

### How are dependencies tracked? What if I read state inside a helper function?
Anything read **synchronously** during the computation counts, including reads inside functions you call. Reads after an `await` inside a helper are not tracked (the docs note `await` directly in the `$derived` expression is a special case in the new async mode). To read something without depending on it, wrap it in [`untrack`](https://svelte.dev/docs/svelte/svelte#untrack).

### Is the result of `$derived` deeply reactive?
No. `$derived` returns the value as is; it does not wrap it in a new proxy. But if it returns part of a `$state` object (like `items[index]`), that part is already a proxy, so mutating it updates the original.

### Can you write to a derived?
Since 5.25, yes, unless declared with `const`. It is a temporary override, useful for optimistic UI: bump `likes` immediately, call the server, and roll back on error. The next time a dependency changes, it recomputes.

### How do deriveds compare to React's `useMemo`?
Same idea (cached computed value), but no dependency array to get wrong, and nothing re-runs the whole component. In React, every render evaluates `useMemo` checks; in Svelte only the derived and its readers are involved.

## Common mistakes
- Writing `$derived(() => a + b)`. That makes the value a *function*. Use `$derived.by(() => a + b)` or `$derived(a + b)`.
- Side effects inside a derived (fetch, logging to analytics, setting state).
- Destructuring a non-derived: `let { a } = obj` is not reactive, but `let { a, b } = $derived(obj)` is.
- Using an effect to "sync" two values. Derive one from the other, or use a function binding.

## Resources
- [Svelte docs: $derived](https://svelte.dev/docs/svelte/$derived) - by, overriding, push-pull, destructuring
- [Tutorial: Derived state](https://svelte.dev/tutorial/svelte/derived-state) - hands-on
- [Svelte docs: When not to use $effect](https://svelte.dev/docs/svelte/$effect) - see the last section on syncing state
