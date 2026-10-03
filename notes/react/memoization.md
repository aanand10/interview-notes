# useMemo, useCallback and React.memo

> **In one line:** `useMemo` caches a computed value, `useCallback` caches a function, and `React.memo` lets a component skip re-rendering when its props are the same; they only help when references stay stable and the skipped work is actually expensive.

## Key points
- [`useMemo(fn, deps)`](https://react.dev/reference/react/useMemo) re-runs `fn` only when a dependency changes; otherwise it returns the cached result.
- [`useCallback(fn, deps)`](https://react.dev/reference/react/useCallback) returns the same function reference until deps change. It is just `useMemo(() => fn, deps)`.
- [`memo(Component)`](https://react.dev/reference/react/memo) shallow-compares props with `Object.is`. If all are equal, the component skips rendering (its own state or context changes still re-render it).
- **Referential equality**: `{}` !== `{}` and `() => {}` !== `() => {}`. A new object or function on every render breaks `memo`.
- They are not free: they cost memory, comparison time and code noise. **Profile first**, then memoise. The **React Compiler** can now add this automatically.

## Example
A watchlist where only changed rows should re-render.

```tsx
import { memo, useCallback, useMemo, useState } from 'react';

type Stock = { symbol: string; price: number; change: number };

const StockRow = memo(function StockRow({
  stock,
  onRemove,
}: {
  stock: Stock;
  onRemove: (symbol: string) => void;
}) {
  return (
    <tr>
      <td>{stock.symbol}</td>
      <td>{stock.price.toFixed(2)}</td>
      <td><button onClick={() => onRemove(stock.symbol)}>Remove</button></td>
    </tr>
  );
});

function Watchlist({ stocks }: { stocks: Stock[] }) {
  const [filter, setFilter] = useState('');
  const [removed, setRemoved] = useState<Set<string>>(new Set());

  // Cached: only recomputed when stocks, filter or removed change,
  // not when some unrelated parent state changes.
  const visible = useMemo(
    () =>
      stocks
        .filter((s) => !removed.has(s.symbol))
        .filter((s) => s.symbol.includes(filter.toUpperCase()))
        .sort((a, b) => b.change - a.change),
    [stocks, filter, removed],
  );

  // Same function reference every render, so memo(StockRow) can skip.
  const handleRemove = useCallback((symbol: string) => {
    setRemoved((prev) => new Set(prev).add(symbol));
  }, []);

  return (
    <>
      <input value={filter} onChange={(e) => setFilter(e.target.value)} />
      <table>
        <tbody>
          {visible.map((s) => (
            <StockRow key={s.symbol} stock={s} onRemove={handleRemove} />
          ))}
        </tbody>
      </table>
    </>
  );
}
```
If a price tick creates new objects only for changed stocks (and keeps the same object for unchanged ones), only those rows re-render.

Svelte 5 equivalent: `$derived(...)` is memoised automatically, and Svelte has no "re-render the whole component" problem to fix.

## When to use it
- Big lists (watchlists, order books) with a memoised row component.
- Expensive calculations: sorting thousands of trades, computing portfolio P&L, building chart series.
- Passing callbacks or objects to a `memo` child, or using them as effect/hook dependencies.

## Likely questions

### What does each one do?
`useMemo` remembers the **result** of a calculation between renders. `useCallback` remembers the **function itself** so its identity stays the same. `React.memo` wraps a **component** and skips its render if the props are shallowly equal to last time. The first two are hooks used inside a component; `memo` is a higher-order component.

### When do they help and when do they hurt?
They help when the skipped work is expensive (heavy calculation or a large subtree) **and** the inputs really stay the same between renders. They hurt or do nothing when the computation is cheap (memo overhead is larger than the work), when deps change every render anyway, or when they make code harder to read. Wrapping every value in `useMemo` is premature optimisation.

### What is referential equality and why does it matter?
React compares props and deps with `Object.is`, which for objects, arrays and functions checks "same reference", not "same contents". `{ qty: 1 } === { qty: 1 }` is `false`. So a component that does `style={{ color: 'red' }}` creates a new object each render, and any `memo` child or effect depending on it sees a "change" every time.

### Why does my memoised child still re-render?
Common reasons:
1. A prop is a new object, array or inline function each render: `onClick={() => ...}`, `options={{...}}`, `items={list.filter(...)}`. Fix with `useCallback` / `useMemo` or move constants outside the component.
2. `children` is JSX, which is a new object every render.
3. The child uses a context whose value changed.
4. The child's own state changed.

```tsx
// Breaks memo: new function every render
<StockRow stock={s} onRemove={(sym) => remove(sym)} />
```

### What is the React Compiler?
A build-time tool (Babel plugin, stable since late 2025) that analyses components and **automatically memoises** values, callbacks and JSX where it is safe, based on the [Rules of React](https://react.dev/reference/rules). With it, you mostly stop writing `useMemo`, `useCallback` and `memo` by hand. It needs pure components and correct hook usage; it skips code that breaks the rules. Conceptually it moves React closer to what Svelte does at compile time.

### How do you decide what to optimise?
Measure first. Use the React DevTools **Profiler** to record an interaction, see which components rendered, why, and how long they took. Turn on "Highlight updates when components render". Optimise the slow ones, then profile again. Test with CPU throttling and a production build, since dev mode is slower.

### Does useMemo guarantee the value is kept?
No. React treats it as a performance hint and may throw the cache away. Code must still work without it, so don't use `useMemo` for things that must run once (use a ref or state for that).

## Common mistakes
- Memoising cheap things like `a + b`.
- `useCallback` on a function passed to a non-memoised child: no benefit.
- Missing deps in `useMemo` / `useCallback`, which returns stale values.
- Mutating arrays in place, so the reference never changes and memoised children don't update.
- Optimising without profiling.

## Resources
- [react.dev: useMemo](https://react.dev/reference/react/useMemo) - when caching a calculation is worth it
- [react.dev: memo](https://react.dev/reference/react/memo) - why props must be stable for memo to work
- [react.dev: useCallback](https://react.dev/reference/react/useCallback) - caching functions for memoised children
- [react.dev: React Compiler](https://react.dev/learn/react-compiler) - automatic memoisation overview
