# React scenario-based questions

> **In one line:** For any "what would you do if..." question, I clarify the problem, give a clear approach, mention the trade-offs, and say how I would measure that it worked.

## Key points
- **Clarify**: ask one or two questions first (how many users, which device, is data live, is SEO needed).
- **Approach**: give the main fix first, then extra options. Keep it practical.
- **Trade-offs**: every fix costs something (complexity, memory, freshness, bundle size). Say it out loud.
- **Measure**: name the tool. React DevTools Profiler, Chrome DevTools Performance and Network panels, Lighthouse, [Core Web Vitals](https://web.dev/articles/vitals) (LCP, INP, CLS), bundle analyzer.
- Link to the deeper notes on this site: React hooks overview, Routing, data loading and lazy routes, Redux and Redux Toolkit.

## Example
The 30-second shape of a good answer:

```text
Clarify:   "Is the data live? Which devices? Is SEO needed?"
Approach:  "First I would ..., then ..."
Trade-off: "This adds ..., so I would only do it if ..."
Measure:   "I would check the Profiler / Lighthouse before and after."
```

## When to use it
In every React or Svelte round where the interviewer says "Imagine..." or "How would you...". Tie answers to the trading domain: watchlists with live prices, order history tables, portfolio dashboards, chart pages.

## Likely questions

### 1) You get 10,000 records from an API. How do you render them efficiently without pagination?
**Clarify:** Do rows have a fixed height? Does the user need search, sort or Ctrl+F on all rows?
**Approach:** Use **virtualization** (windowing): only render the ~30 rows visible in the viewport plus a small buffer, and swap them as the user scrolls. Libraries: [TanStack Virtual](https://tanstack.com/virtual/latest) or react-window. Also give each row a stable `key` (the order id, not the index), wrap the row in `React.memo`, and do sorting and filtering with `useMemo`. If filtering is slow, wrap the filter update in `useTransition` or use `useDeferredValue` so typing stays smooth.
**Trade-offs:** Browser find (Ctrl+F) cannot see hidden rows, variable row heights need measuring, and screen readers need correct `aria-rowcount`. Infinite scroll is the other option if the API supports cursors.
**Measure:** React Profiler render time and Performance panel; DOM node count goes from 10,000 rows to about 30.

```tsx
import { useVirtualizer } from "@tanstack/react-virtual";

function OrdersList({ orders }: { orders: Order[] }) {
  const parentRef = useRef<HTMLDivElement>(null);
  const v = useVirtualizer({ count: orders.length, getScrollElement: () => parentRef.current, estimateSize: () => 40 });
  return (
    <div ref={parentRef} style={{ height: 600, overflow: "auto" }}>
      <div style={{ height: v.getTotalSize(), position: "relative" }}>
        {v.getVirtualItems().map((row) => (
          <div key={orders[row.index].id}
               style={{ position: "absolute", top: 0, width: "100%", transform: `translateY(${row.start}px)` }}>
            <OrderRow order={orders[row.index]} />
          </div>
        ))}
      </div>
    </div>
  );
}
```

### 2) A page takes 5 seconds to load. How do you investigate and optimize it?
**Clarify:** Is it slow on first load or on every navigation? On all devices or slow phones only?
**Approach:** Measure first, do not guess. Run Lighthouse and check the Network panel's waterfall and the Performance panel. Then fix by cause:
- **Big JavaScript**: code-split routes, lazy-load heavy widgets, remove unused libraries (see scenario 7).
- **Slow or chained API calls**: run them in parallel, fetch in route loaders instead of `useEffect` waterfalls, cache with TanStack Query, ask the backend for a smaller payload.
- **Slow rendering**: profile with React DevTools, memoize expensive parts, virtualize long lists.
- **Images and fonts**: compress, use modern formats, set width and height, lazy-load below-the-fold images, preload the hero image and key font.
- **Perceived speed**: show a skeleton fast; consider SSR or streaming (Next.js, or SvelteKit in our case).
**Trade-offs:** SSR adds server cost; caching can show stale prices, so set short stale times for market data.
**Measure:** [LCP](https://web.dev/articles/lcp) before and after, plus real-user monitoring, not just my fast laptop.

### 3) You need to call three independent APIs at the same time and handle failures. How?
**Clarify:** If one fails, should the page fail, or show partial data?
**Approach:** Start all three together, not one after another. Use [`Promise.allSettled`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/allSettled) when partial data is fine, so one failure does not hide the others; use `Promise.all` when all are required (it rejects fast on the first error). Add an `AbortController` to cancel on unmount, and maybe a timeout and retry. With TanStack Query, `useQueries` runs them in parallel and gives each its own error and retry.
**Trade-offs:** Partial UI needs per-section error states and a retry button. More parallel requests can hit rate limits.
**Measure:** The Network waterfall should show three bars starting together.

```tsx
const [profile, holdings, news] = await Promise.allSettled([
  fetch("/api/profile", { signal }).then((r) => r.json()),
  fetch("/api/holdings", { signal }).then((r) => r.json()),
  fetch("/api/news", { signal }).then((r) => r.json()),
]);
if (holdings.status === "rejected") showSectionError("holdings"); // others still render
```

### 4) How do you fetch data before a route component renders?
**Approach:** Use a React Router **loader** on the route (or the route's `lazy` to load code and loader together). The router waits for the loader, then renders; the component reads it with `useLoaderData()`. Errors go to the route's `errorElement`. Combine with TanStack Query by calling `queryClient.ensureQueryData` in the loader for caching. Prefetch on link hover to make it feel instant. In Next.js this is a server component or server fetch; in SvelteKit it is the `load` function.
**Trade-offs:** The user waits on the old page during navigation, so show a global progress bar (`useNavigation().state === "loading"`).
**Measure:** No loading spinners flash on the new page; no request waterfall. Deeper note: Routing, data loading and lazy routes.

### 5) A child component re-renders every time the parent updates. How do you fix it?
**Clarify:** Is it actually slow? Re-renders are often cheap. Check with the Profiler first.
**Approach:** By default, React re-renders all children when a parent renders. Fixes, in order:
1. **Move state down** so the parent that changes is smaller, or pass the expensive child as `children` so it is not re-created.
2. Wrap the child in [`React.memo`](https://react.dev/reference/react/memo) so it skips rendering when props are equal (shallow compare).
3. Keep props stable: `useCallback` for function props, `useMemo` for object and array props. A new `{}` or `() => {}` each render breaks `memo`.
4. If Context causes it, split the context or memoize the provider value.
The React Compiler can do much of this memoization automatically.
**Trade-offs:** `memo` adds a comparison cost and code noise; use it on expensive components only.
**Measure:** React DevTools "Highlight updates when components render" and the Profiler's "why did this render".

```tsx
const PriceRow = memo(function PriceRow({ symbol, onSelect }: Props) { /* ... */ });

function Watchlist({ symbols }: { symbols: string[] }) {
  const [selected, setSelected] = useState<string>();
  const onSelect = useCallback((s: string) => setSelected(s), []); // stable identity
  return symbols.map((s) => <PriceRow key={s} symbol={s} onSelect={onSelect} />);
}
```

### 6) Many components share the same state. How do you design state management?
**Clarify:** What kind of state is it? Server data, UI state, URL state, or form state?
**Approach:** Split by type. Server data (holdings, orders) goes in TanStack Query or RTK Query. Shareable UI state (filters, selected tab) goes in URL search params. Rarely changing global values (user, theme) go in Context. Frequently changing shared client state (watchlist, layout, selected symbol) goes in Redux Toolkit or Zustand with selectors so only affected components re-render. Everything else stays local with `useState`.
**Trade-offs:** Redux gives structure and DevTools but more boilerplate; Zustand is lighter; Context is simple but re-renders all consumers.
**Measure:** Re-render counts in the Profiler; how easy it is to trace a bug. Deeper note: Redux and Redux Toolkit.

### 7) The bundle size is large. What steps do you take to reduce it?
**Approach:**
1. **Measure**: run a bundle analyzer (`rollup-plugin-visualizer` for Vite, `webpack-bundle-analyzer`) and Chrome's Coverage tab.
2. **Code-split routes** with `React.lazy` or router `lazy`, and lazy-load heavy widgets (charts, editors, PDF export) with `import()`.
3. **Replace or trim big libraries**: `date-fns` instead of `moment`, import only what you use (`lodash-es/debounce`), drop unused polyfills by targeting modern browsers.
4. Make sure **tree shaking** works (ES modules, no side-effect imports).
5. Compress (Brotli), cache with hashed file names, and serve from a CDN.
**Trade-offs:** Too many tiny chunks means more requests; lazy parts show a loader the first time, so prefetch likely next routes.
**Measure:** KB of JS on first load (gzip/brotli), Lighthouse "Reduce unused JavaScript", LCP and INP.

### 8) An API takes a long time to respond. How do you keep the UI responsive?
**Clarify:** Seconds or minutes? Can the user keep working meanwhile?
**Approach:** Never block the UI. Show a skeleton or progress state, and keep the rest of the page usable. Use **optimistic updates** when success is likely (add to watchlist instantly, roll back on error). Show cached data first and refresh in the background (stale-while-revalidate with TanStack Query). Cancel outdated requests with `AbortController` and debounce search input. For heavy work on the client, use `useTransition` / `useDeferredValue` or move it to a Web Worker. For very long jobs (reports), use polling or a WebSocket and notify when done.
**Trade-offs:** Optimistic UI must handle rollback carefully; for placing real orders, do not fake success, show "Pending" until the server confirms.
**Measure:** [INP](https://web.dev/articles/inp) stays low, no long tasks in the Performance panel.

```tsx
const [query, setQuery] = useState("");
const deferredQuery = useDeferredValue(query); // input updates now, results catch up
const { data, isFetching } = useQuery({ queryKey: ["search", deferredQuery], queryFn: () => search(deferredQuery) });
```

### 9) How do you load a component only when the user navigates to its route?
**Approach:** Route-based code splitting. `const Portfolio = lazy(() => import("./Portfolio"))` and render it inside `<Suspense fallback={<Skeleton />}>`. With `createBrowserRouter`, use `lazy: () => import("./routes/portfolio")` so the component and its loader download together. The bundler makes a separate chunk that is fetched only on navigation.
**Trade-offs:** The first visit to that route waits for the chunk; prefetch on hover or when idle for routes users usually visit next.
**Measure:** In the Network panel, the chunk appears only when the route opens. Deeper note: Routing, data loading and lazy routes.

### 10) A large grid has images and complex components. How do you optimize rendering and scrolling?
**Clarify:** How many items? Fixed card sizes? Does it update live (prices)?
**Approach:**
- **Virtualize** rows and columns (TanStack Virtual supports grids).
- **Images**: `loading="lazy"`, `decoding="async"`, explicit `width`/`height` to avoid layout shift, responsive `srcset`, small thumbnails, modern formats (WebP/AVIF).
- **Memoize** each cell (`React.memo`) with stable props and keys, so a price tick re-renders one cell, not the grid. Subscribe each cell to its own symbol through a selector.
- **Cheap scrolling**: passive scroll listeners, animate only `transform` and `opacity`, avoid reading layout in scroll handlers, CSS `content-visibility: auto` for off-screen sections.
- **Batch live updates**: throttle price updates to animation frames (`requestAnimationFrame`) instead of rendering every tick.
**Trade-offs:** Virtualization complicates sticky headers, keyboard navigation and accessibility; throttling means prices update a few ms later.
**Measure:** Smooth 60 fps in the Performance panel, no long tasks, low CLS.

## Common mistakes
- Jumping to `useMemo` and `React.memo` before measuring.
- Giving only one fix with no trade-off and no way to measure.
- Using array index as `key` in lists that reorder or virtualize.
- Fetching in `useEffect` in nested components, which creates waterfalls.
- Faking success for money-moving actions (orders, payments).

## Interview tip
Always explain **why** (the root cause), the **trade-offs** (what it costs), and a **practical example** from a real app ("in our watchlist, a price tick re-rendered 200 rows; memoizing the row and selecting per symbol cut it to one"). Measuring before and after shows senior-level thinking.

## Resources
- [react.dev: memo](https://react.dev/reference/react/memo) - skipping re-renders and when it helps
- [react.dev: useDeferredValue](https://react.dev/reference/react/useDeferredValue) - keep the UI responsive during slow updates
- [web.dev: Virtualize large lists with react-window](https://web.dev/articles/virtualize-long-lists-react-window) - why and how to window long lists
- [web.dev: Optimize Largest Contentful Paint](https://web.dev/articles/optimize-lcp) - step-by-step fix for slow page loads
