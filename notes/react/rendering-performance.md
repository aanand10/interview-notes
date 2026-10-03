# React performance optimisation

> **In one line:** I first measure with the React Profiler, then cut unnecessary re-renders (stable props, memo, state placed close to where it's used), render less (virtualize long lists, code-split routes), and keep the UI responsive with `useTransition` / `useDeferredValue` for heavy updates.

## Key points
- A component re-renders when **its state changes**, **its parent re-renders**, or a **context it reads changes**. Props changing is not the trigger; the parent re-rendering is.
- **Measure first** with the React DevTools [Profiler](https://react.dev/reference/react/Profiler): flamegraph, "why did this render?", render durations.
- **Render less**: virtualize long lists, lazy-load heavy screens with `lazy` + `Suspense`.
- **Re-render less**: keep state local, avoid inline objects/functions to memoised children, use `memo` and selectors.
- **Prioritise**: [`useTransition`](https://react.dev/reference/react/useTransition) and [`useDeferredValue`](https://react.dev/reference/react/useDeferredValue) mark expensive updates as low priority so typing and clicks stay smooth.

## Example
A big screener: typing stays fast while a 10,000-row table filters in the background.

```tsx
import { lazy, memo, Suspense, useDeferredValue, useMemo, useState } from 'react';

const ChartPanel = lazy(() => import('./ChartPanel')); // separate JS chunk

type Row = { symbol: string; price: number; volume: number };

const ResultsTable = memo(function ResultsTable({ rows, query }: { rows: Row[]; query: string }) {
  const filtered = useMemo(
    () => rows.filter((r) => r.symbol.includes(query.toUpperCase())),
    [rows, query],
  );
  return <VirtualList rows={filtered} />; // only visible rows in the DOM
});

export function Screener({ rows }: { rows: Row[] }) {
  const [query, setQuery] = useState('');
  // Input uses `query` (urgent). Table gets a lagging copy (low priority).
  const deferredQuery = useDeferredValue(query);
  const isStale = query !== deferredQuery;

  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <div style={{ opacity: isStale ? 0.6 : 1 }}>
        <ResultsTable rows={rows} query={deferredQuery} />
      </div>
      <Suspense fallback={<p>Loading chart...</p>}>
        <ChartPanel />
      </Suspense>
    </>
  );
}
```

List virtualization with [react-window](https://github.com/bvaughn/react-window) (1.x API shown; 2.x renames it to `List` with a `rowComponent` prop, same idea):

```tsx
import { FixedSizeList } from 'react-window';

function VirtualList({ rows }: { rows: Row[] }) {
  return (
    <FixedSizeList height={600} width="100%" itemCount={rows.length} itemSize={32}>
      {({ index, style }) => (
        // `style` positions the row absolutely; always apply it
        <div style={style}>{rows[index].symbol} {rows[index].price}</div>
      )}
    </FixedSizeList>
  );
}
```
10,000 rows become about 20 DOM nodes plus a few extra for smooth scrolling.

## When to use it
Order books, trade history and screeners (virtualize), heavy chart or settings pages (code split), symbol search over large lists (deferred value), live tickers (throttled store updates with per-row subscriptions).

## Likely questions

### What causes a component to re-render?
Three things: its own state changes (`setState` with a new value), its parent re-renders (by default all children re-render too, even with the same props), or a context it consumes changes. A re-render is not automatically a DOM update; React diffs and may change nothing, but the function still runs and that costs CPU.

### How do you use the React DevTools Profiler?
Open the Profiler tab, click record, do the slow interaction, stop. The flamegraph shows each commit and which components rendered and how long they took. Enable "Record why each component rendered" to see "props changed: onSelect" or "parent rendered". Also enable "Highlight updates" in settings to see re-renders live. Profile a production build with CPU throttling for realistic numbers. There is also a `<Profiler>` component for measuring in code.

### What is list virtualization?
Rendering only the rows that are visible in the scroll window, plus a small buffer, and using spacers so the scrollbar still looks right. DOM size stays constant regardless of data size, which fixes slow mounts, memory and scroll jank. Libraries: react-window, TanStack Virtual. Trade-offs: browser find (Ctrl+F) can't see hidden rows, dynamic row heights are harder, and screen readers need correct `aria-rowcount` / `aria-rowindex` on tables.

### How does code splitting with lazy and Suspense work?
`lazy(() => import('./ChartPanel'))` tells the bundler to put that component in a separate chunk, loaded on first render. While it loads, the nearest `<Suspense fallback>` shows the fallback. Split by route and by heavy, rarely used widgets (advanced charting, PDF statements). You can preload on hover by calling the import early.

### Why avoid inline object and function props?
`<Row style={{ color: 'green' }} onClick={() => buy(s)} />` creates a new object and function every render, so a `memo(Row)` always sees changed props and re-renders. Move constants outside the component, and wrap callbacks in `useCallback` and objects in `useMemo` when passing them to memoised children. Inline props on non-memoised plain elements are fine.

### useTransition vs useDeferredValue?
Both mark work as **non-urgent** so React can interrupt it for user input. `useTransition` wraps the **state update** you own: `startTransition(() => setTab('history'))`, and gives `isPending`. `useDeferredValue` is for when you only have the **value** (often a prop): you get a lagging copy to render the heavy part with. They do not make work faster; they keep the UI responsive while it happens. They are not debouncing: no fixed delay, and the background render is thrown away if a newer value arrives.

### How would you render a live price ticker without re-rendering everything?
- Keep prices outside React's top-level state: a WebSocket writes into a store (Zustand or `useSyncExternalStore`) keyed by symbol.
- Each row subscribes to **only its symbol** with a selector, so an AAPL tick re-renders only the AAPL row.
- **Batch/throttle**: buffer ticks and flush once per `requestAnimationFrame` or every 100-250 ms; humans can't read 50 updates a second.
- Memoise rows and give them stable keys; virtualize if there are many.
- For extreme rates, update the text node directly through a ref, skipping React for that cell.
- Pause updates when the tab is hidden (`visibilitychange`).

### Other quick wins?
Keep state as low in the tree as possible, move a slow subtree into `children` so it isn't re-rendered by the parent's state, avoid layout thrashing, use production builds, and check bundle size.

## Common mistakes
- Optimising without profiling, or profiling only in dev mode.
- Wrapping everything in `memo`/`useMemo` while still passing new objects.
- Putting fast-changing data at the top of the tree or in context.
- Using index keys in virtualized or sorted lists.
- Treating `useDeferredValue` as a debounce for network requests (it doesn't reduce requests).

## Resources
- [react.dev: useDeferredValue](https://react.dev/reference/react/useDeferredValue) - keeping input fast while heavy UI lags behind
- [react.dev: useTransition](https://react.dev/reference/react/useTransition) - non-blocking state updates
- [react.dev: lazy](https://react.dev/reference/react/lazy) - code splitting components with Suspense
- [web.dev: Virtualize large lists with react-window](https://web.dev/articles/virtualize-long-lists-react-window) - why and how to virtualize
