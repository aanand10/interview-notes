# Large lists, virtualization and big API responses

> **In one line:** For 10,000 rows I don't render 10,000 DOM nodes. I render only the rows you can see (virtualization), keep each row cheap with memoization, and move heavy data work off the main render path.

## Key points
- The cost is mostly **DOM nodes and layout**, not React itself. 10,000 rows x 8 cells = 80,000 nodes, and every scroll or style change is slow.
- **Virtualization (windowing)** renders only the visible rows plus a few extra (overscan). A tall spacer keeps the scrollbar correct.
- Make each row cheap: stable `key`, `React.memo` on the row, stable callbacks, no heavy work inside render.
- For big API responses: **normalize** the data, **memoize** derived data (`useMemo`), move heavy work to a [Web Worker](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Using_web_workers), and keep typing responsive with [`useDeferredValue`](https://react.dev/reference/react/useDeferredValue) or [`useTransition`](https://react.dev/reference/react/useTransition).
- In real apps use a library: [TanStack Virtual](https://tanstack.com/virtual/latest) (headless) or react-window. Hand-roll only to show you understand it.

## Example
A hand-written virtual list for a 10,000-row trade blotter. Fixed row height keeps the math simple.

```tsx
import { memo, useState, type UIEvent } from "react";

type Trade = { id: string; symbol: string; qty: number; price: number };

const ROW_H = 40;      // every row is exactly 40px tall
const VIEWPORT_H = 600; // visible height of the scroll box
const OVERSCAN = 5;     // extra rows above and below to avoid blank flashes

const Row = memo(function Row({ trade, top }: { trade: Trade; top: number }) {
  return (
    <div role="row" style={{ position: "absolute", top, height: ROW_H, left: 0, right: 0 }}>
      {trade.symbol} {trade.qty} @ {trade.price.toFixed(2)}
    </div>
  );
});

export function VirtualTradeList({ trades }: { trades: Trade[] }) {
  const [scrollTop, setScrollTop] = useState(0);

  const first = Math.floor(scrollTop / ROW_H);                 // first visible row
  const visible = Math.ceil(VIEWPORT_H / ROW_H);               // rows that fit (15)
  const start = Math.max(0, first - OVERSCAN);
  const end = Math.min(trades.length, first + visible + OVERSCAN);

  return (
    <div
      role="table"
      style={{ height: VIEWPORT_H, overflowY: "auto", position: "relative" }}
      onScroll={(e: UIEvent<HTMLDivElement>) => setScrollTop(e.currentTarget.scrollTop)}
    >
      {/* Spacer: as tall as ALL rows, so the scrollbar looks like 10,000 rows exist */}
      <div style={{ height: trades.length * ROW_H, position: "relative" }}>
        {trades.slice(start, end).map((t, i) => (
          <Row key={t.id} trade={t} top={(start + i) * ROW_H} />
        ))}
      </div>
    </div>
  );
}
```

The same math, run with node (10,000 rows, 40px each, 600px viewport, overscan 5):

```js
function getRange(scrollTop, viewportH, rowH, total, overscan = 5) {
  const first = Math.floor(scrollTop / rowH);
  const visible = Math.ceil(viewportH / rowH);
  const start = Math.max(0, first - overscan);
  const end = Math.min(total, first + visible + overscan);
  return { totalHeight: total * rowH, start, end, offsetY: start * rowH, rendered: end - start };
}
getRange(0, 600, 40, 10000);      // { totalHeight: 400000, start: 0,    end: 20,    offsetY: 0,      rendered: 20 }
getRange(20000, 600, 40, 10000);  // { totalHeight: 400000, start: 495,  end: 520,   offsetY: 19800,  rendered: 25 }
getRange(399400, 600, 40, 10000); // { totalHeight: 400000, start: 9980, end: 10000, offsetY: 399200, rendered: 20 }
```
- At the top there is no overscan above, so only 20 rows render.
- In the middle we render 15 visible + 5 above + 5 below = 25 rows.
- At the bottom `end` is clamped to 10,000, so again 20 rows.

## When to use it
A trade history, order book, or a watchlist with thousands of symbols. Also log viewers and admin tables. If the list is under a few hundred simple rows, plain rendering plus `memo` is usually enough.

## Likely questions

### How do you optimize a component rendering 10,000 rows?
First I measure with the React Profiler to see if the time is in React renders or in the browser's layout. Then: virtualize the list so only ~20-50 rows are in the DOM. Give rows a stable `key` (the id, not the index), wrap the row in `React.memo`, and pass stable props (`useCallback` for handlers, or one handler on the parent using the row id). Do sorting and filtering in `useMemo`, not on every render. If the data updates live (price ticks), update only the changed rows instead of replacing the whole array.

### How does virtualization (windowing) work?
The scroll container has a fixed height. Inside it I put one tall element whose height equals `rowCount x rowHeight`. On scroll I read `scrollTop`, work out which row indexes are visible, and render only those, positioned at their real offset. As you scroll, rows leaving the view are unmounted and new ones are mounted, so the DOM size stays constant no matter how big the list is.

### How does the scrollbar work when only ~50 rows of 10,000 are rendered?
The browser draws the scrollbar from the **content height**, not from how many rows exist. So I make a spacer that is `10,000 x 40px = 400,000px` tall. The scrollbar thumb is then the right size and position. Then I place the few real rows inside it:
- `first = floor(scrollTop / rowHeight)`, `count = ceil(viewportHeight / rowHeight)`.
- `start = max(0, first - overscan)`, `end = min(total, first + count + overscan)`.
- Each row goes at `top = index x rowHeight`, or I wrap the slice in one div with `transform: translateY(start x rowHeight)`.

Overscan is a few extra rows above and below so fast scrolling doesn't show blank gaps. For variable heights, libraries measure rows after render and keep a running offset table.

### Pagination vs virtualization vs infinite scroll: what are the trade-offs?
| | Pagination | Infinite scroll | Virtualization |
|---|---|---|---|
| DOM size | Small | Grows forever (unless also virtualized) | Small, constant |
| SEO | Best: each page has a URL | Weak: crawlers may not scroll | Weak: off-screen rows are not in HTML |
| Accessibility | Easy, clear position | Hard to reach the footer, focus can get lost | Screen readers only "see" rendered rows; set `aria-rowcount` / `aria-rowindex` |
| Find-in-page (Ctrl+F) | Works on current page | Works on loaded items | Broken for off-screen rows |
| Best for | Search results, reports, shareable links | Feeds | Big tables already in memory |

Often you combine them: infinite scroll for fetching, virtualization for rendering. For a trading app, order history with pagination is easier to share and audit; a live watchlist is better virtualized.

### How do you optimize rendering after receiving a large API response?
- **Normalize**: store items as `{ byId, allIds }` so updating one item doesn't copy or re-scan the whole list.
- **Memoize derived data**: `useMemo` for sort, filter, group, so it only reruns when inputs change.
- **Web Worker**: parse, sort, or aggregate 100k items in a worker so the main thread stays free for input.
- **Chunk the work**: process in batches and yield back to the browser between them with [`scheduler.yield()`](https://developer.chrome.com/blog/use-scheduler-yield) (not supported in every browser yet, so feature-detect) or `setTimeout(0)`. This keeps each task under ~50ms, which helps [INP](https://web.dev/articles/inp).
- **Lower the priority** of the heavy render with `useTransition` or `useDeferredValue`, so typing in a filter box stays instant.
- And of course, only render what's visible (virtualize), and ask the backend for less (pagination, fields filter).

```tsx
const [query, setQuery] = useState("");
const deferredQuery = useDeferredValue(query); // input updates now, list catches up later
const filtered = useMemo(
  () => trades.filter((t) => t.symbol.includes(deferredQuery.toUpperCase())),
  [trades, deferredQuery]
);
```

```js
async function processInChunks(items, fn, size = 500) {
  for (let i = 0; i < items.length; i += size) {
    items.slice(i, i + size).forEach(fn);
    // let the browser handle clicks and paint between chunks
    if (globalThis.scheduler?.yield) await scheduler.yield();
    else await new Promise((r) => setTimeout(r, 0));
  }
}
```

### How do you optimize a large grid with images and complex components?
- **Virtualize in two directions** (rows and columns), for example with TanStack Virtual.
- **Images**: `loading="lazy"`, `decoding="async"`, and always set `width`/`height` (or `aspect-ratio`) so nothing jumps (no layout shift). Serve small thumbnails via `srcset`.
- **Memoize cells** with `React.memo` and keep cell props primitive so they compare cheaply.
- For grids that are not virtualized, CSS [`content-visibility: auto`](https://web.dev/articles/content-visibility) with `contain-intrinsic-size` lets the browser skip layout and paint for off-screen items, while keeping them in the DOM (so Ctrl+F still works).
- Defer heavy pieces (charts, menus) until a cell is visible or hovered.

```tsx
<img src={logo.thumb} width={48} height={48} loading="lazy" decoding="async" alt={`${symbol} logo`} />
```

### Why not use the array index as the key in a virtual list?
When you scroll, the slice changes, so index 0 of the slice is a different item each time. With index keys React reuses the wrong component and its state (for example an open menu jumps to another row). Use the stable item id.

## Common mistakes
- Forgetting the spacer height, so the scrollbar is tiny and you can't scroll to the end.
- No overscan: blank flashes on fast scroll.
- Passing a new inline object or arrow function to each memoized row, which breaks `memo`.
- Virtualizing a list of 50 items: extra complexity for no gain.
- Breaking accessibility: no `role="row"` / `aria-rowcount`, and keyboard focus lost when the focused row unmounts.

## Resources
- [TanStack Virtual](https://tanstack.com/virtual/latest) - headless virtualizer for lists and grids
- [web.dev: Virtualize long lists](https://web.dev/articles/virtualize-long-lists-react-window) - why windowing helps, with react-window
- [react.dev: useDeferredValue](https://react.dev/reference/react/useDeferredValue) - keep input responsive during heavy renders
- [web.dev: content-visibility](https://web.dev/articles/content-visibility) - let the browser skip off-screen rendering
