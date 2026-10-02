# `$state`

> **In one line:** `$state` creates reactive state; for arrays and plain objects it returns a deeply reactive [Proxy](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy), so both reassigning and mutating (like `push` or `obj.x = 1`) update the UI.

## Key points
- `let count = $state(0)` gives you a normal-looking variable. Read it and write it like any variable. The compiler turns reads into `$.get(count)` and writes into `$.set(count, ...)`.
- **Deep reactivity:** arrays and plain objects are wrapped in a Proxy, recursively. Svelte intercepts `get` and `set` on each property, so `todos[0].done = true` and `todos.push(x)` are tracked. Each property is its own signal, so only the UI that reads that property updates.
- **Not proxied:** class instances, `Map`, `Set`, `Date`. For classes, put `$state` on class fields. For built-ins, use `SvelteMap`, `SvelteSet`, `SvelteDate`, `SvelteURL` from [`svelte/reactivity`](https://svelte.dev/docs/svelte/svelte-reactivity).
- **`$state.raw`:** not deep. You can only reassign it, not mutate it. Cheaper for big data you replace as a whole (API responses, chart series).
- **`$state.snapshot(x)`:** returns a plain, non-proxy copy. Use it before passing state to libraries, `structuredClone`, `postMessage` or logging.

## Example
Run in Node through the real Svelte 5.57 compiler (in a `.svelte.js` module with `$effect.root` and `flushSync`):

```js
import { flushSync } from 'svelte';

const cleanup = $effect.root(() => {
  let todos = $state([{ done: false }]);
  let count = $derived(todos.length);
  let raw = $state.raw({ price: 100 });

  $effect(() => console.log('effect: count =', count, 'first done =', todos[0].done));
  $effect(() => console.log('raw effect: price =', raw.price));
  flushSync();

  todos.push({ done: true });   flushSync(); // array mutation
  todos[0].done = true;         flushSync(); // deep mutation
  raw.price = 200;              flushSync(); // mutation of raw: ignored
  console.log('raw.price after mutate =', raw.price);
  raw = { price: 300 };         flushSync(); // reassignment: works

  console.log('snapshot:', JSON.stringify($state.snapshot(todos)));
  try { structuredClone(todos); } catch (e) { console.log('structuredClone error:', e.name); }
});
cleanup();
```

```bash
effect: count = 1 first done = false   # first run after mount
raw effect: price = 100
effect: count = 2 first done = false   # push() is tracked
effect: count = 2 first done = true    # deep property write is tracked
raw.price after mutate = 200           # value changed, but NO effect re-ran
raw effect: price = 300                # reassigning raw state is tracked
snapshot: [{"done":true},{"done":true}]
structuredClone error: DataCloneError  # a Proxy cannot be cloned; snapshot first
```

## When to use it
- Order form fields, toggles, selected tab: `$state` primitives.
- Watchlist array you add/remove from: `$state([...])` and just `push` / `splice`.
- 5,000-row trade history or candle data you fetch and replace: `$state.raw`, then assign a new array.
- Sending the form to an API or saving to IndexedDB: `$state.snapshot(form)`.

## Likely questions
### Why does `arr.push()` update the UI now, when it didn't in Svelte 4?
In Svelte 4, reactivity was triggered by **assignment**. The compiler only saw `items = ...` and inserted `$$invalidate`; `items.push(3)` was not an assignment, so nothing happened. In Svelte 5, `$state([])` returns a Proxy. `push` writes to index `n` and to `length` through the Proxy's `set` trap, so Svelte knows exactly what changed and updates anything that read `items` or `items.length`.

### How does deep reactivity work?
When you read a property of a state proxy inside an effect or template, Svelte creates (lazily) a signal for that property and subscribes the reader. When you write that property, the proxy's `set` trap updates that signal. New objects you push are proxied too. The original object you passed in is not mutated; the proxy holds its own copy.

### When would you use `$state.raw`?
When the data is large and you never mutate it, only replace it. Example: a list of 10,000 historical trades from the server, or chart data. Proxying every nested object costs memory and time. With `$state.raw`, you do `trades = await fetchTrades()`, and only the reassignment is tracked. Mutations like `trades.push()` silently do nothing to the UI.

### What is `$state.snapshot` for?
It returns a plain object copy of a state proxy at that moment. Use it when passing data to code that does not expect a Proxy: `structuredClone`, `postMessage` to a web worker, a charting library, or `console.log` (which otherwise prints `Proxy {...}`). If a value has `toJSON`, the snapshot uses that.

### Can I export `$state` from a `.svelte.ts` file?
Yes, but you cannot export a variable that you **reassign**. The compiler works one file at a time, so other files would get the signal object, not the value. Export an object and mutate its properties, or export getter functions:

```ts
// prices.svelte.ts
export const prices = $state<Record<string, number>>({}); // OK: we mutate, not reassign

export function setPrice(symbol: string, value: number) {
  prices[symbol] = value;
}
```

### How do you use `$state` in a class?
Use it on fields. The compiler turns them into getters and setters on the prototype.

```ts
class OrderForm {
  qty = $state(1);
  price = $state(0);
  total = $derived(this.qty * this.price);
  reset = () => { this.qty = 1; this.price = 0; }; // arrow keeps `this` in onclick
}
```

### Is destructuring state reactive?
No. `let { done } = todos[0]` reads the value once, like normal JS. Read `todos[0].done` where you need it, or use `$derived`.

## Common mistakes
- Using `$state.raw` and then calling `push`: no error, no update.
- Logging a proxy and being confused by `Proxy(Array)`: use `$state.snapshot` or `$inspect`.
- Putting a `Map` in `$state` and calling `.set()`: not reactive. Use `SvelteMap`.
- Passing `onclick={form.reset}` where `reset` is a normal method: `this` is lost. Use an arrow field or `() => form.reset()`.
- Exporting `export let count = $state(0)` and reassigning it: importers get an object, not a number.

## Resources
- [Svelte docs: $state](https://svelte.dev/docs/svelte/$state) - deep state, raw, snapshot, classes, modules
- [Tutorial: Deep state](https://svelte.dev/tutorial/svelte/deep-state) - interactive example of mutation
- [svelte/reactivity](https://svelte.dev/docs/svelte/svelte-reactivity) - reactive Map, Set, Date, URL
- [MDN: Proxy](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy) - how the underlying JS feature works
