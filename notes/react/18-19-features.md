# React 18 and 19 features

> **In one line:** React 18 made rendering interruptible (concurrent) so urgent updates like typing stay smooth, and React 19 built on that with Actions for forms and async work, plus `use()`, simpler refs and the React Compiler.

## Key points
- **React 18 (2022):** concurrent rendering via `createRoot`, automatic batching, `useTransition`, `useDeferredValue`, `useId`, `useSyncExternalStore`, streaming SSR with Suspense.
- **React 19 (Dec 2024):** Actions, `useActionState`, `useOptimistic`, `useFormStatus`, `use()`, `ref` as a normal prop, `<Context>` as a provider, document metadata tags, Server Components and Server Functions as stable features.
- **React Compiler:** a build step that adds memoization for you, so you rarely need `useMemo`, `useCallback` or `React.memo` by hand.
- The big idea: React can now **prioritise** work. Urgent updates (typing, clicking) go first; non-urgent ones (filtering a big list) can wait or be interrupted.

## Example
Each feature with a short example, in a trading context.

### Concurrent rendering
Rendering can be paused and resumed. You turn it on by using `createRoot`, then opt in per update with transitions.

```tsx
import { createRoot } from "react-dom/client";
createRoot(document.getElementById("root")!).render(<App />);
```

### Automatic batching
In React 18, several `setState` calls are merged into **one re-render** everywhere: event handlers, promises, timeouts. Before 18, only React event handlers were batched.

```tsx
async function refresh() {
  const quote = await fetchQuote("AAPL");
  setPrice(quote.price);   // no render yet
  setChange(quote.change); // still no render
  // one render here with both values (React 17 would render twice)
}
// Opt out when you must read the DOM right away: flushSync(() => setPrice(p));
```

### useTransition
Marks an update as non-urgent. The input stays responsive and you get an `isPending` flag.

```tsx
const [isPending, startTransition] = useTransition();
function onTabChange(tab: string) {
  startTransition(() => setTab(tab)); // heavy "All instruments" tab renders in the background
}
return <div style={{ opacity: isPending ? 0.6 : 1 }}>{/* tabs */}</div>;
```

### useDeferredValue
Gives you a "lagging" copy of a value. Use it when you do not own the setter (for example a prop).

```tsx
function SymbolSearch({ query }: { query: string }) {
  const deferredQuery = useDeferredValue(query); // updates after urgent renders finish
  return <SlowResultsList query={deferredQuery} />; // wrap list in memo so it can skip
}
```

### useId
Makes a unique, stable ID that matches between server and client. Use it for accessibility attributes, never for list keys.

```tsx
function QtyField() {
  const id = useId();
  return <><label htmlFor={id}>Quantity</label><input id={id} type="number" /></>;
}
```

### Actions and useActionState (React 19)
An **Action** is an async function run inside a transition. React tracks pending state, errors and form reset for you. `<form action={fn}>` accepts one.

```tsx
const [state, formAction, isPending] = useActionState(
  async (prev: { error?: string }, formData: FormData) => {
    const res = await placeOrder(formData.get("symbol"), Number(formData.get("qty")));
    return res.ok ? {} : { error: res.message };
  },
  {} // initial state
);

<form action={formAction}>
  <input name="symbol" /> <input name="qty" type="number" />
  <button disabled={isPending}>{isPending ? "Placing..." : "Buy"}</button>
  {state.error && <p role="alert">{state.error}</p>}
</form>
```

### useOptimistic (React 19)
Shows the expected result immediately while the request runs. It snaps back to the real state when the Action ends.

```tsx
const [optimisticList, addOptimistic] = useOptimistic(
  watchlist,
  (list, symbol: string) => [...list, symbol]
);
async function add(formData: FormData) {
  const symbol = String(formData.get("symbol"));
  addOptimistic(symbol);        // appears instantly
  await saveToWatchlist(symbol); // parent updates real `watchlist` after this
}
```

### use() (React 19)
Reads a promise (suspends until ready) or a context. Unlike hooks, it **can** be called inside `if` and loops.

```tsx
function Price({ pricePromise }: { pricePromise: Promise<number> }) {
  const price = use(pricePromise);
  const theme = use(ThemeContext);
  return <span className={theme}>{price}</span>;
}
```

### ref as a prop (React 19)
Function components get `ref` as a normal prop. No more `forwardRef`.

```tsx
function AmountInput({ ref, ...props }: { ref?: React.Ref<HTMLInputElement> }) {
  return <input ref={ref} {...props} />;
}
```

### React Compiler
A Babel/build plugin that analyses components and memoizes values and JSX automatically. It assumes you follow the Rules of React (pure render, no mutating props or state).

```tsx
// You write plain code; the compiler caches `filtered` and the row JSX for you.
function Watchlist({ items, query }: Props) {
  const filtered = items.filter((i) => i.symbol.includes(query));
  return filtered.map((i) => <Row key={i.symbol} item={i} />);
}
```

## When to use it
- `useTransition` / `useDeferredValue`: filtering a list of thousands of instruments while the user types.
- `useActionState` + `useOptimistic`: order forms and "add to watchlist" buttons that feel instant.
- Svelte 5 note: runes are fine-grained, so the problems `useDeferredValue` and the Compiler solve are smaller there.

## Likely questions
### What is concurrent rendering, in simple words?
Before React 18, once rendering started it could not stop, so a big render could freeze typing. Now React can pause a non-urgent render, handle the keystroke, then continue or throw away the stale work. You opt in with transitions; it is not automatic for every update.

### useTransition vs useDeferredValue?
Both mark work as low priority. `useTransition` wraps the **setter** call, so use it when you own the state update. `useDeferredValue` wraps a **value**, so use it when the value comes from props or a library. Neither is debouncing: there is no fixed delay, React just does the urgent work first.

### What problem does useOptimistic solve?
It removes the manual "set temp state, call API, roll back on error" code. The optimistic value only lives while the Action is pending, so if the request fails, the UI goes back to the real state by itself.

## Common mistakes
- Using `useId` for list keys. Keys must come from your data.
- Thinking transitions make code faster. They make it feel responsive; the work is the same.
- Expecting the React Compiler to fix code that breaks the Rules of React. It skips or mis-optimises such components.

## Resources
- [react.dev: React v18.0](https://react.dev/blog/2022/03/29/react-v18) - official list of React 18 features
- [react.dev: React v19](https://react.dev/blog/2024/12/05/react-19) - Actions, use(), ref as prop and more
- [react.dev: useTransition](https://react.dev/reference/react/useTransition) - transitions with examples
- [react.dev: React Compiler](https://react.dev/learn/react-compiler) - what it does and how to adopt it
