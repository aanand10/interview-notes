# React hooks overview

> **In one line:** Hooks are special functions, starting with `use`, that let a function component keep state, run side effects and reach into React features like context, without writing a class.

## Key points
- A hook "hooks into" React's internals. React stores the hook's data next to the component, not inside your function, so the data survives re-renders.
- React finds each hook's data **by call order**. That is why hooks must be called in the same order on every render (the rules of hooks).
- `useState` and `useReducer` hold data that should update the screen. `useRef` holds data that should NOT update the screen.
- `useEffect` syncs with things outside React (fetch, WebSocket, timers). `useMemo` and `useCallback` are only performance tools.
- You can build your own **custom hooks** (for example `useLivePrice(symbol)`) by combining built-in hooks.

## Example
A live price ticker using several hooks together.

```tsx
import { useState, useEffect, useRef, useMemo, useId } from "react";

function useLivePrice(symbol: string) {
  const [price, setPrice] = useState<number | null>(null);

  useEffect(() => {
    // Side effect: open a socket for this symbol
    const ws = new WebSocket(`wss://example.com/prices/${symbol}`);
    ws.onmessage = (e) => setPrice(JSON.parse(e.data).price);
    return () => ws.close(); // cleanup when symbol changes or component unmounts
  }, [symbol]);

  return price;
}

export function Ticker({ symbol, qty }: { symbol: string; qty: number }) {
  const price = useLivePrice(symbol);
  const renders = useRef(0); // survives renders, changing it does not re-render
  renders.current++;

  const value = useMemo(() => (price ?? 0) * qty, [price, qty]); // recompute only when inputs change
  const labelId = useId(); // stable unique id for accessibility

  return (
    <div aria-labelledby={labelId}>
      <span id={labelId}>{symbol}</span> {price ?? "--"} | Position: {value.toFixed(2)}
    </div>
  );
}
```

## When to use it
Every modern React component uses hooks. In a trading app: `useState` for form inputs on an order ticket, `useEffect` for the price WebSocket, `useRef` for the chart's DOM node, `useContext` for the logged-in user or theme, `useReducer` for a multi-step order flow, `useTransition` to keep typing smooth while a big watchlist filters.

## Quick table

| Hook | What it is for | One-line example |
| --- | --- | --- |
| `useState` | Local state that re-renders the UI | `const [qty, setQty] = useState(1)` |
| `useEffect` | Side effects after paint (fetch, subscribe, timers) | `useEffect(() => { const id = setInterval(poll, 5000); return () => clearInterval(id); }, [])` |
| `useLayoutEffect` | Effect that runs before the browser paints (measure DOM) | `useLayoutEffect(() => setH(ref.current!.offsetHeight), [])` |
| `useRef` | Mutable box or DOM reference, no re-render | `const inputRef = useRef<HTMLInputElement>(null)` |
| `useMemo` | Cache an expensive calculated value | `const total = useMemo(() => sum(rows), [rows])` |
| `useCallback` | Cache a function so its identity stays the same | `const onBuy = useCallback(() => buy(id), [id])` |
| `useContext` | Read a value from a Context provider | `const user = useContext(UserContext)` |
| `useReducer` | State with many actions, logic in a reducer | `const [state, dispatch] = useReducer(orderReducer, init)` |
| `useId` | Unique, SSR-safe id for labels and ARIA | `const id = useId()` |
| `useTransition` | Mark an update as low priority so the UI stays responsive | `startTransition(() => setFilter(text))` |

## Likely questions

### What are React Hooks?
Hooks are functions like `useState` and `useEffect` that let function components use React features: state, lifecycle-like side effects, context, refs. Before hooks (React 16.8) you needed class components for this. Hooks also let us share stateful logic through custom hooks, which is cleaner than higher-order components or render props. In Svelte terms, `useState` is close to `$state` and `useEffect` is close to `$effect`.

### What are the commonly used hooks?
- **useState**: holds a value; calling the setter re-renders the component.
- **useEffect**: runs code after render to sync with the outside world. Return a cleanup function.
- **useLayoutEffect**: same as useEffect but runs synchronously after DOM changes and before paint. Use it only to measure layout and avoid a flicker.
- **useRef**: gives a `{ current }` box that persists and does not trigger a render. Used for DOM nodes, timer ids, previous values.
- **useMemo**: caches a computed value until dependencies change.
- **useCallback**: caches a function; mainly useful when passing callbacks to `React.memo` children or effect dependencies.
- **useContext**: reads the nearest Provider's value; avoids prop drilling.
- **useReducer**: like useState but with a reducer `(state, action) => newState`; good for complex, related state.
- **useId**: generates a stable unique id that matches between server and client, for `htmlFor` and `aria-*`.
- **useTransition**: returns `[isPending, startTransition]`; updates inside `startTransition` are low priority and can be interrupted, so urgent updates like typing stay fast.

### Why do we use useState instead of a normal variable like let?
Two reasons. First, changing a `let` does not tell React to re-render, so the screen never updates. Second, a `let` is re-created on every render because the component function runs again from the top, so the value resets. `useState` fixes both: React stores the value outside the function, and the setter schedules a re-render.

```jsx
// Broken
function Counter() {
  let count = 0;
  return <button onClick={() => { count++; console.log(count); }}>{count}</button>;
  // Logs 1, 2, 3... but the button always shows 0: no re-render.
  // Even if something else re-renders it, count resets to 0.
}

// Works
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount((c) => c + 1)}>{count}</button>;
}
```

### What are the rules of hooks?
1. **Only call hooks at the top level**: not inside `if`, loops, nested functions or after an early `return`. React matches hooks by call order, so the order must be the same every render.
2. **Only call hooks from React functions**: function components or custom hooks (whose names start with `use`). Not from plain helpers or class components.
The ESLint plugin `eslint-plugin-react-hooks` checks both rules and also warns about missing effect dependencies. One exception: React 19's `use()` can be called inside conditions.

### What is the difference between useEffect and useLayoutEffect?
`useEffect` runs after the browser paints, so it does not block the screen. `useLayoutEffect` runs after React updates the DOM but before paint, so you can measure an element and set state without a visible jump. It blocks painting, so use it only for layout measuring, like positioning a tooltip under a price cell.

### What is the difference between useMemo and useCallback?
`useMemo` caches the **result** of a function, `useCallback` caches the **function itself**. `useCallback(fn, deps)` is the same as `useMemo(() => fn, deps)`. Both only help when something compares by reference, like a `React.memo` child or a dependency array. With the React Compiler, much of this manual memoization becomes automatic.

### What is a custom hook?
A function whose name starts with `use` and which calls other hooks, for example `useLivePrice(symbol)` or `useDebounce(value, 300)`. It shares logic, not state: each component that calls it gets its own copy of the state.

## Common mistakes
- Missing dependencies in `useEffect`, which causes stale values (a "stale closure"). Use the functional setter `setX(x => x + 1)` when the new value depends on the old one.
- Using `useEffect` to compute derived data. Just compute it during render, or use `useMemo` if it is expensive.
- Forgetting cleanup for sockets, intervals and listeners, which leaks memory and duplicates price updates.
- Expecting state to change immediately after `setState`. The new value shows up on the next render.
- Wrapping everything in `useMemo`/`useCallback`. It adds cost and noise; measure first.

## Resources
- [react.dev: Built-in React Hooks](https://react.dev/reference/react/hooks) - official list of every hook with examples
- [react.dev: Rules of Hooks](https://react.dev/reference/rules/rules-of-hooks) - the exact rules and why they exist
- [react.dev: State, a component's memory](https://react.dev/learn/state-a-components-memory) - the "why not a let variable" explanation
- [react.dev: You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect) - avoid the most common useEffect misuse
