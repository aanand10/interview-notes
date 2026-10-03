# Control flow (SolidJS)

> **In one line:** Because Solid components run only once, it uses special components for conditions and lists: `<Show>` for if/else, `<For>` and `<Index>` for lists, and `createStore` for nested state that updates field by field.

How I frame it: these map to Svelte's `{#if}` and `{#each}`, and `createStore` is close to a deep `$state` object.

## Key points
- **`<Show when={cond} fallback={...}>`** is the "if". The child can be a function that gets the truthy value ([Show](https://docs.solidjs.com/reference/components/show)).
- **`<For each={list}>{(item, index) => ...}`** is keyed by item reference. When the array changes, rows are moved, not re-created. `item` is the value, `index` is a signal (`index()`) ([list rendering](https://docs.solidjs.com/concepts/control-flow/list-rendering)).
- **`<Index each={list}>{(item, i) => ...}`** is keyed by position. `item` is a signal (`item()`), `i` is a plain number. Use it for primitives or fixed-length lists where values change in place.
- **Others:** `<Switch>`/`<Match>` (switch-case), `<Dynamic>`, `<ErrorBoundary>`, `<Suspense>`.
- **`createStore(obj)`** returns `[state, setState]`. Reads are plain property access (`state.orders[0].qty`), tracked per property. Updates use a path syntax ([stores](https://docs.solidjs.com/concepts/stores)).

## Example
```tsx
import { Show, For, Index } from 'solid-js';
import { createStore } from 'solid-js/store';

type Order = { id: string; symbol: string; qty: number; filled: boolean };

function Orders(props: { user?: { name: string } }) {
  const [state, setState] = createStore({ orders: [] as Order[], prices: [101, 102, 99] });

  const fill = (id: string) =>
    setState('orders', o => o.id === id, 'filled', true); // updates only that one field

  return (
    <>
      <Show when={props.user} fallback={<p>Please log in</p>}>
        {(user) => <h2>Hi {user().name}</h2>}
      </Show>

      {/* objects with ids: <For> */}
      <For each={state.orders} fallback={<p>No orders</p>}>
        {(order, i) => (
          <div>
            {i() + 1}. {order.symbol} x {order.qty} {order.filled ? 'filled' : 'open'}
            <button onClick={() => fill(order.id)}>Fill</button>
          </div>
        )}
      </For>

      {/* list of numbers that change in place: <Index> */}
      <Index each={state.prices}>{(price, i) => <span>#{i}: {price()} </span>}</Index>
    </>
  );
}
```

Svelte 5 equivalent of the list: `{#each orders as order, i (order.id)}...{/each}`.

## Likely questions
### Why not just use `.map()` and `&&` like React?
It works, but `.map` re-creates every row when the array changes, because the component does not re-run to diff. `<For>` caches rows by reference and only adds, removes or moves what changed. `<Show>` likewise avoids rebuilding the subtree.

### `<For>` vs `<Index>`?
`<For>` tracks items by reference: good for objects like orders that get added, removed or reordered. `<Index>` tracks by position: good for primitives like a list of prices where index 2 just gets a new number. In `For`, the index is a signal; in `Index`, the item is a signal.

### What is `createStore` and how is it different from a signal?
A signal holds one value; replacing an object re-triggers everything that read it. A store is a proxy over a nested object, so each property is tracked separately. Changing one order's `filled` flag updates only that cell. Use `produce` for mutable-style updates and `reconcile` to merge fresh server data.

## Resources
- [Solid: Conditional rendering](https://docs.solidjs.com/concepts/control-flow/conditional-rendering) - `<Show>`, `<Switch>`, `<Match>`
- [Solid: List rendering](https://docs.solidjs.com/concepts/control-flow/list-rendering) - `<For>` vs `<Index>`
- [Solid: Stores](https://docs.solidjs.com/concepts/stores) - `createStore`, path setters, `produce`
