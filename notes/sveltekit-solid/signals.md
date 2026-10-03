# Signals (SolidJS)

> **In one line:** In Solid, a signal is a reactive value you read by calling a getter; `createMemo` derives cached values from signals, `createEffect` runs side effects when signals change, and Solid updates just the exact DOM nodes that depend on them, with no virtual DOM.

How I frame it: I have not shipped Solid in production, but its signal model matches Svelte 5 runes, which I use every day. `createSignal` is like `$state`, `createMemo` like `$derived`, `createEffect` like `$effect`.

## Key points
- **`createSignal(initial)`** returns `[getter, setter]`. Read with `count()` (a function call), write with `setCount(5)` or `setCount(c => c + 1)` ([signals docs](https://docs.solidjs.com/concepts/signals)).
- **Auto-tracking.** When a getter is called inside a tracking scope (JSX, memo, effect), Solid records that dependency. No dependency arrays like React.
- **`createMemo(fn)`** caches a derived value and only re-computes when its inputs change. Its result is also a getter ([memos](https://docs.solidjs.com/concepts/derived-values/memos)).
- **`createEffect(fn)`** runs after rendering and re-runs when any signal it read changes. Use `onCleanup` for cleanup ([effects](https://docs.solidjs.com/concepts/effects)).
- **No virtual DOM.** JSX compiles to real DOM creation code once, plus tiny effects bound to each dynamic spot. A price change updates one text node, it does not re-run the component or diff a tree.

## Example
```tsx
import { createSignal, createMemo, createEffect, onCleanup } from 'solid-js';

function Position() {
  const [price, setPrice] = createSignal(100);
  const [qty, setQty] = createSignal(10);

  const value = createMemo(() => price() * qty());   // cached; recomputes when price or qty change

  createEffect(() => {
    document.title = `Value: ${value()}`;              // side effect, re-runs when value changes
  });

  const id = setInterval(() => setPrice(p => +(p + Math.random() - 0.5).toFixed(2)), 1000);
  onCleanup(() => clearInterval(id));

  return (
    <div>
      {/* only this text node updates each second */}
      <p>Price: {price()}</p>
      <input type="number" value={qty()} onInput={e => setQty(+e.currentTarget.value)} />
      <p>Value: {value()}</p>
    </div>
  );
}
```

Same thing in Svelte 5:

```svelte
<script>
  let price = $state(100);
  let qty = $state(10);
  let value = $derived(price * qty);
  $effect(() => { document.title = `Value: ${value}`; });
</script>
```

## When to use it
Live prices, order book updates, P&L that changes many times a second. Fine-grained updates mean only the changed cells repaint, which is exactly what a trading screen needs.

## Likely questions
### What is a signal?
A value plus a list of subscribers. Reading it inside a reactive scope subscribes that scope. Writing it notifies only those subscribers. In Solid you read it as `count()` because the read must be a function call for tracking to work.

### Memo vs plain function?
`const double = () => count() * 2` also works and is reactive, but it recomputes every time it is read. `createMemo` caches the result and only recomputes when `count` changes, so use it when the work is expensive or read in many places.

### How is this different from React?
React re-runs the whole component on state change and diffs a virtual DOM. Solid runs the component once and wires signals directly to DOM nodes. So no `useMemo`/`useCallback` tuning and no dependency arrays.

### Should you set signals inside an effect?
Usually no. If it is derived, use a memo. Effects are for talking to the outside world: DOM APIs, logging, sockets. Same rule as `$effect` in Svelte.

## Common mistakes
- Writing `count` instead of `count()` in JSX, which passes the function, not the value.
- Reading a signal outside a tracking scope (for example at the top of the component) and expecting it to update.
- Using effects to sync one signal into another instead of a memo.

## Resources
- [Solid: Signals](https://docs.solidjs.com/concepts/signals) - core concept with examples
- [Solid: Memos](https://docs.solidjs.com/concepts/derived-values/memos) - cached derived values
- [Solid: Effects](https://docs.solidjs.com/concepts/effects) - side effects and cleanup
- [Svelte: $state](https://svelte.dev/docs/svelte/$state) - the Svelte 5 equivalent
