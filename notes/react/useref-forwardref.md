# useRef and refs

> **In one line:** `useRef` gives you a box (`{ current }`) that keeps its value across renders without causing a re-render, used for DOM nodes and for mutable values like timer ids; to give a parent access to a child's DOM node you pass a ref down, which in React 19 is just a normal prop.

## Key points
- [`useRef(initial)`](https://react.dev/reference/react/useRef) returns the same object every render. Changing `ref.current` does **not** re-render.
- **DOM access**: `<input ref={inputRef} />`. React sets `inputRef.current` to the DOM node after commit and back to `null` on unmount.
- **Mutable values**: timer ids, previous values, the latest callback, a WebSocket instance, "has this run" flags.
- Don't read or write `ref.current` during render (except lazy init); do it in effects and handlers.
- **React 19**: function components can take `ref` as a regular prop. `forwardRef` is no longer needed (and is planned for deprecation). `useImperativeHandle` customises what the ref exposes.

## Example
```tsx
import { useEffect, useImperativeHandle, useRef, useState, type Ref } from 'react';

// 1. Focus an input on mount
function OrderQtyInput() {
  const inputRef = useRef<HTMLInputElement>(null);
  useEffect(() => {
    inputRef.current?.focus();
  }, []);
  return <input ref={inputRef} type="number" aria-label="Quantity" />;
}

// 2. Store the previous value (price went up or down?)
function usePrevious<T>(value: T) {
  const ref = useRef<T | undefined>(undefined);
  useEffect(() => {
    ref.current = value; // runs after render, so during render it's still the old value
  }, [value]);
  return ref.current;
}

function PriceCell({ price }: { price: number }) {
  const prev = usePrevious(price);
  const dir = prev === undefined ? '' : price > prev ? 'up' : price < prev ? 'down' : '';
  return <td className={dir}>{price.toFixed(2)}</td>;
}

// 3. Timer id in a ref: no re-render when it changes
function Stopwatch() {
  const [ms, setMs] = useState(0);
  const timerRef = useRef<number | null>(null);
  const start = () => {
    if (timerRef.current !== null) return;
    timerRef.current = window.setInterval(() => setMs((m) => m + 100), 100);
  };
  const stop = () => {
    if (timerRef.current !== null) clearInterval(timerRef.current);
    timerRef.current = null;
  };
  useEffect(() => stop, []); // clear on unmount
  return <><p>{ms} ms</p><button onClick={start}>Start</button><button onClick={stop}>Stop</button></>;
}

// 4. React 19: ref as a prop + useImperativeHandle
type SearchHandle = { focus: () => void; clear: () => void };

function SymbolSearch({ ref }: { ref?: Ref<SearchHandle> }) {
  const inputRef = useRef<HTMLInputElement>(null);
  // Parent gets only these two methods, not the whole DOM node
  useImperativeHandle(ref, () => ({
    focus: () => inputRef.current?.focus(),
    clear: () => { if (inputRef.current) inputRef.current.value = ''; },
  }), []);
  return <input ref={inputRef} placeholder="Search symbol" />;
}

function Toolbar() {
  const searchRef = useRef<SearchHandle>(null);
  return (
    <>
      <SymbolSearch ref={searchRef} />
      <button onClick={() => searchRef.current?.focus()}>Search (/)</button>
    </>
  );
}
```

Before React 19 the same component needed `forwardRef`:

```tsx
import { forwardRef } from 'react';

const FancyInput = forwardRef<HTMLInputElement, { label: string }>(function FancyInput(
  { label },
  ref,
) {
  return <label>{label}<input ref={ref} /></label>;
});
```

Svelte 5 equivalent: `bind:this={inputEl}` for DOM access, and a plain `let` (not `$state`) for values that shouldn't trigger updates.

## When to use it
Focusing the quantity field when an order ticket opens, scrolling a trade log to the bottom, measuring a cell for a tooltip, holding a chart library instance (`chartRef.current = createChart(el)`), storing a WebSocket or interval id, comparing the previous price for a green/red flash.

## Likely questions

### useRef vs useState?
Both persist across renders. Updating state triggers a re-render and the new value appears in the next render's snapshot. Updating a ref changes `current` immediately and does nothing to the UI. Use state for anything shown on screen; use a ref for values that only your handlers and effects need.

### How do you focus an input?
Create `const ref = useRef<HTMLInputElement>(null)`, attach `ref={ref}`, then call `ref.current?.focus()` in an effect or event handler. For focus on mount, the `autoFocus` prop also works, but refs give control (for example after validation fails, focus the first invalid field).

### How do you get the previous value of a prop?
The `usePrevious` hook above: store the value in a ref inside an effect. Since effects run after render, during render the ref still holds the last value. React docs also show a pattern that stores the previous value in state and compares during render, which avoids reading refs during render.

### Why keep timers in refs?
The interval id must survive re-renders so `stop` can clear it, but changing it shouldn't re-render. A local variable would be lost every render; state would cause extra renders.

### What is forwardRef and what changed in React 19?
By default, `ref` was not passed to function components as a prop, so to expose an inner DOM node you wrapped the component in `forwardRef((props, ref) => ...)`. In React 19, function components receive `ref` as a normal prop, so `forwardRef` is not needed for new code. Class components still don't get `ref` as a prop. React 19 also lets a ref callback return a cleanup function.

### What does useImperativeHandle do?
It lets a component decide what the parent's ref points to. Instead of exposing the raw `<input>`, you expose a small API like `{ focus, clear }`. Use it sparingly, for imperative actions like focus, scroll or play. Data should still flow through props.

## Common mistakes
- Using a ref for something displayed on screen, then wondering why the UI doesn't update.
- Reading `ref.current` during render for a DOM node (it is `null` on the first render).
- Putting `ref.current` in a dependency array: changing it doesn't trigger anything.
- Overusing `useImperativeHandle` instead of props.

## Resources
- [react.dev: useRef](https://react.dev/reference/react/useRef) - API and do's and don'ts
- [react.dev: Manipulating the DOM with refs](https://react.dev/learn/manipulating-the-dom-with-refs) - focus, scroll, exposing refs
- [react.dev: useImperativeHandle](https://react.dev/reference/react/useImperativeHandle) - exposing a custom ref API
- [react.dev: React 19 release](https://react.dev/blog/2024/12/05/react-19) - ref as a prop and ref cleanup functions
