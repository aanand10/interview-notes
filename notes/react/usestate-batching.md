# useState and state updates

> **In one line:** `useState` gives a component a value that survives re-renders and a setter that schedules a re-render; each render sees a fixed snapshot of state, so when the next value depends on the old one you use a functional update.

## Key points
- [`useState`](https://react.dev/reference/react/useState) returns `[value, setValue]`. Calling the setter does **not** change the variable now; it queues an update and React re-renders the component later with the new value.
- **State is a snapshot**: inside one render, `count` is a constant. Event handlers and timeouts created in that render keep seeing that value.
- **Functional updates** (`setCount(c => c + 1)`) get the latest queued value, so they are safe for multiple updates in a row and inside async code.
- **Automatic batching (React 18+)**: several `setState` calls in the same tick (events, promises, `setTimeout`, native listeners) are merged into one re-render.
- If the new value is the same as the old one (checked with [`Object.is`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/is)), React can skip the re-render. So never mutate objects or arrays; create new ones.

## Example
```tsx
import { useState } from 'react';

function QtyStepper() {
  const [qty, setQty] = useState(0);

  function addThreeWrong() {
    setQty(qty + 1); // qty is 0 in this render -> queue "set to 1"
    setQty(qty + 1); // still 0 -> "set to 1"
    setQty(qty + 1); // still 0 -> "set to 1"
    console.log(qty); // 0, the snapshot did not change
  }

  function addThreeRight() {
    setQty((q) => q + 1); // 0 -> 1
    setQty((q) => q + 1); // 1 -> 2
    setQty((q) => q + 1); // 2 -> 3
  }

  return (
    <>
      <p>Qty: {qty}</p>
      <button onClick={addThreeWrong}>+3 (shows 1)</button>
      <button onClick={addThreeRight}>+3 (shows 3)</button>
    </>
  );
}
```
Output after one click on each button starting from 0:
- `addThreeWrong` -> UI shows `1`, console logs `0`. Each call used the same snapshot `qty = 0`.
- `addThreeRight` -> UI shows `3`. Each updater gets the result of the previous one.
- Both cause only **one** re-render thanks to batching.

Svelte 5 equivalent: `let qty = $state(0); qty++` updates immediately, no setter and no snapshot.

## When to use it
Local UI state: an order form's quantity, a selected tab, an "is dropdown open" flag, a buy/sell toggle. Use the functional form for counters, toggles, and anything updated from a WebSocket callback or timer.

## Likely questions

### How does useState work under the hood?
React stores state on the component's fiber in a list, in the order hooks are called. On the first render it uses the initial value; on later renders it ignores the argument and returns the stored value. That is why hooks must be called in the same order every render. Calling the setter pushes an update onto a queue and schedules a render; during that render React processes the queue to get the new value.

### Why use functional updates like `setCount(c => c + 1)`?
Because `count` in your closure is the value from the render that created the handler, not the latest one. If you update several times in a row, or from a timer or a promise, using `count + 1` can lose updates. The updater function receives the most recent pending value, so it is always correct. Rule of thumb: if the next state is computed from the previous state, use the function form.

### What is automatic batching in React 18?
Batching means React groups several state updates into one re-render. Before React 18 this only happened inside React event handlers; updates in `setTimeout`, promises or native listeners each caused a separate render. With `createRoot` in React 18, batching is automatic everywhere.

```tsx
async function placeOrder() {
  const res = await api.submit(order);
  setStatus('filled');    // React 17: render
  setLastOrder(res);      // React 17: another render
  // React 18+: one render for both
}
```
If you really need the DOM updated right away, `flushSync` from `react-dom` forces a synchronous update, but it is rarely needed.

### What is stale state in closures?
A closure remembers the variables from when it was created. A `setInterval` created on mount keeps seeing the first render's state forever.

```tsx
useEffect(() => {
  const id = setInterval(() => {
    setSeconds(seconds + 1); // stale: always 0 + 1
    // fix: setSeconds((s) => s + 1);
  }, 1000);
  return () => clearInterval(id);
}, []);
```
Fixes: functional update, add the value to the dependency array, or keep the latest value in a ref.

### What does "state is a snapshot" mean?
Each render is a function call with its own props and state. When you call `setX`, the current render's `x` does not change; only the next render gets the new value. So `alert(count)` right after `setCount(5)` still shows the old count. This is very different from Svelte 5 where `$state` reads always give the current value.

### What is lifting state up?
When two components need the same data, move the state to their closest common parent and pass the value and a setter down as props. Example: a `SymbolSearch` and a `PriceChart` both need the selected symbol, so `TradePage` owns `selectedSymbol`. That gives one source of truth instead of two copies that can drift apart.

### What is the derived state anti-pattern?
Storing something in state that can be computed from props or other state. It creates two sources of truth that need syncing, usually with an extra effect and an extra render.

```tsx
// Anti-pattern
const [total, setTotal] = useState(0);
useEffect(() => setTotal(qty * price), [qty, price]);

// Better: compute during render (wrap in useMemo only if it is expensive)
const total = qty * price;
```
Same for copying a prop into state: `useState(props.price)` only uses the prop on first render and then ignores updates. If you need a reset when an id changes, use a `key`.

### How do you update objects and arrays in state?
Create a new copy, never mutate. `setOrder({ ...order, qty: 10 })` or `setList(list.filter(o => o.id !== id))`. Mutating and passing the same reference means `Object.is` sees no change and React may skip the render.

### Lazy initial state?
`useState(() => JSON.parse(localStorage.getItem('watchlist') ?? '[]'))` runs the function only on the first render. Passing `useState(expensive())` would call `expensive()` on every render and throw the result away.

## Common mistakes
- Reading state right after setting it and expecting the new value.
- `setCount(count + 1)` inside intervals or repeated calls.
- Mutating state (`list.push(x); setList(list)`).
- Mirroring props into state, or storing values you can compute.
- Too many separate `useState`s that always change together; use one object or `useReducer`.

## Resources
- [react.dev: useState](https://react.dev/reference/react/useState) - API reference with updater and lazy init
- [react.dev: State as a snapshot](https://react.dev/learn/state-as-a-snapshot) - why state does not change inside a render
- [react.dev: Queueing a series of state updates](https://react.dev/learn/queueing-a-series-of-state-updates) - batching and updater functions
- [react.dev: Choosing the state structure](https://react.dev/learn/choosing-the-state-structure) - avoid redundant and derived state
