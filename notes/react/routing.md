# Routing, data loading and lazy routes

> **In one line:** React has no built-in router, so we use a library like React Router, which maps the URL to a tree of components, can load data for a route before it renders, and can lazy-load each route's code.

## Key points
- [React Router](https://reactrouter.com/home) watches the URL (with the browser [History API](https://developer.mozilla.org/en-US/docs/Web/API/History_API)) and renders the matching route without a full page reload. That is client-side routing in a single-page app (SPA).
- Routes can be **nested**. A parent layout renders an `<Outlet />`, which is the slot where the matching child route appears. This is like `+layout.svelte` with `{@render children()}` in SvelteKit.
- **Dynamic params** like `/stocks/:symbol` are read with `useParams()`.
- In "data mode" (`createBrowserRouter`), each route can have a **loader** that fetches data before the route renders, read with `useLoaderData()`. This is React Router's version of SvelteKit's `load`.
- Each route's code can be split out with `React.lazy` + `Suspense`, or with the router's `lazy` property, so users only download the page they visit.

## Example
A small trading app with a protected layout, nested routes, a loader and a lazy route (React Router v7; in v6 the same APIs come from `react-router-dom`).

```tsx
import {
  createBrowserRouter, RouterProvider, Outlet, Link, NavLink,
  useParams, useLoaderData, useNavigate, redirect,
} from "react-router";

// Loader runs BEFORE the route renders. Same idea as SvelteKit's load.
async function stockLoader({ params }: { params: { symbol?: string } }) {
  const res = await fetch(`/api/stocks/${params.symbol}`);
  if (!res.ok) throw new Response("Not found", { status: 404 });
  return res.json();
}

// Protected routes: check auth in the parent loader and redirect if needed
async function requireAuth() {
  const me = await fetch("/api/me");
  if (me.status === 401) throw redirect("/login");
  return me.json();
}

function AppLayout() {
  return (
    <>
      <nav>
        <NavLink to="/watchlist">Watchlist</NavLink>
        <NavLink to="/portfolio">Portfolio</NavLink>
      </nav>
      <Outlet /> {/* child route renders here */}
    </>
  );
}

function StockPage() {
  const { symbol } = useParams();            // dynamic param
  const stock = useLoaderData() as { price: number };
  const navigate = useNavigate();            // navigate from code
  return (
    <div>
      <h1>{symbol}: {stock.price}</h1>
      <button onClick={() => navigate(`/trade/${symbol}`)}>Trade</button>
      <Link to="/watchlist">Back</Link>
    </div>
  );
}

const router = createBrowserRouter([
  { path: "/login", lazy: () => import("./routes/login") },
  {
    path: "/",
    loader: requireAuth,
    Component: AppLayout,
    children: [
      { path: "watchlist", lazy: () => import("./routes/watchlist") },
      { path: "stocks/:symbol", loader: stockLoader, Component: StockPage },
      // Route-level code splitting: the module exports `Component` and `loader`
      { path: "portfolio", lazy: () => import("./routes/portfolio") },
    ],
  },
]);

export default function App() {
  return <RouterProvider router={router} />;
}
```

The older "declarative" style still works and is common in interviews:

```jsx
<BrowserRouter>
  <Routes>
    <Route path="/" element={<AppLayout />}>
      <Route path="stocks/:symbol" element={<StockPage />} />
      <Route path="*" element={<NotFound />} />
    </Route>
  </Routes>
</BrowserRouter>
```

## When to use it
Any multi-page React SPA: watchlist, stock detail, orders, portfolio, settings. Use loaders when a page is useless without its data (stock detail). Lazy-load heavy pages like an advanced charting screen or an admin area that most users never open.

## Likely questions

### How does routing work in React?
React itself only renders components, so routing comes from a library, usually React Router. You define routes as a tree, either with `createBrowserRouter` plus `RouterProvider` (data mode, which supports loaders) or with `<BrowserRouter>` and `<Routes>/<Route>` (declarative mode). When the URL changes through a `<Link>` or `navigate()`, the router updates history with `pushState` and renders the matching route, without reloading the page. Dynamic segments like `:symbol` come from `useParams()`. Nested routes share a layout, and the child renders inside `<Outlet />`. For **protected routes**, I either check auth in a parent loader and `throw redirect("/login")`, or wrap routes in a component that returns `<Navigate to="/login" replace />` when there is no user. Real security still lives on the server; the client check is only for UX.

### How do you fetch API data before rendering a route?
In React Router data mode I add a `loader` to the route. The router calls it on navigation, waits for it, then renders the component, which reads the result with `useLoaderData()`. Loaders for parent and child routes run in parallel, so there is no waterfall. If the loader throws, the route's `errorElement` shows. This is the same idea as SvelteKit's `load` function in `+page.ts`. For better UX I can **prefetch on hover**: in framework mode `<Link prefetch="intent">` does it for you; otherwise I call `queryClient.prefetchQuery` in `onMouseEnter`. Many teams combine **TanStack Query with the router**: the loader calls `queryClient.ensureQueryData(...)`, and the component uses `useQuery` with the same key, so you get both "data before render" and caching, background refetch and dedupe.

```tsx
const stockQuery = (s: string) => ({
  queryKey: ["stock", s],
  queryFn: () => fetch(`/api/stocks/${s}`).then((r) => r.json()),
});

const loader = ({ params }) => queryClient.ensureQueryData(stockQuery(params.symbol));

function StockPage() {
  const { symbol } = useParams();
  const { data } = useQuery(stockQuery(symbol!)); // already in cache, no spinner
  return <h1>{data.price}</h1>;
}

<Link to="/stocks/AAPL" onMouseEnter={() => queryClient.prefetchQuery(stockQuery("AAPL"))}>AAPL</Link>
```

### What is dynamic component loading?
It means the code for a component is not in the main bundle. It is downloaded only when it is needed, for example when the user opens a route, clicks "Show chart" or opens a modal. The bundler (Vite, webpack) sees `import()` and puts that module in a separate chunk file. This keeps the first load small and fast.

### What is the difference between dynamic imports (import()) and lazy loading (React.lazy)?
[`import()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/import) is a plain JavaScript feature. It loads any module at runtime and returns a Promise. It works for anything: a utility, a chart library, a JSON file. `React.lazy` is a React wrapper that takes a function returning that `import()` Promise and turns it into a component you can render. React suspends while the code loads, so it must sit inside a `<Suspense fallback>`. `React.lazy` expects the module's **default export** to be a component. So: `import()` is the loading mechanism, `React.lazy` is how React renders a lazily loaded component.

```tsx
// Plain dynamic import: load a heavy lib only when needed
async function exportCsv(rows) {
  const { unparse } = await import("papaparse");
  return unparse(rows);
}

// React.lazy: lazily loaded component
const AdvancedChart = lazy(() => import("./AdvancedChart"));
<Suspense fallback={<Spinner />}><AdvancedChart symbol="AAPL" /></Suspense>
```

### How do you load a component only when the user navigates to a route?
That is route-based code splitting. With declarative routes I wrap each page in `lazy` and put a `Suspense` around the routes. With `createBrowserRouter` I use the route's `lazy` property, which can load the component and its loader together, so both download in parallel on navigation.

```tsx
const Portfolio = lazy(() => import("./pages/Portfolio"));

<Suspense fallback={<PageSkeleton />}>
  <Routes>
    <Route path="/portfolio" element={<Portfolio />} />
  </Routes>
</Suspense>

// Data router version: ./routes/portfolio exports `Component` and `loader`
{ path: "portfolio", lazy: () => import("./routes/portfolio") }
```
SvelteKit does this automatically: every route is its own chunk.

### What is the difference between Link and useNavigate?
`<Link>` renders a real `<a href>`, so it is accessible, supports open-in-new-tab and is the default choice. `useNavigate()` is for navigating from code, for example after an order is placed successfully. Pass `{ replace: true }` when the user should not go back, like after login.

## Common mistakes
- Using a plain `<a href>` for internal links, which reloads the whole app.
- Fetching in `useEffect` inside every nested route, which creates request waterfalls. Loaders run in parallel.
- Forgetting `Suspense` around `React.lazy` components, or lazy-loading with a named export (needs `.then(m => ({ default: m.Chart }))`).
- Calling `lazy()` inside a component. Declare it at module top level, or state resets on every render.
- Treating client-side protected routes as security. The API must still check auth.

## Resources
- [React Router docs](https://reactrouter.com/home) - modes, routing, loaders, lazy routes
- [react.dev: lazy](https://react.dev/reference/react/lazy) - React.lazy with Suspense, rules and pitfalls
- [MDN: import()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/import) - the JS feature behind code splitting
- [TanStack Query: Prefetching](https://tanstack.com/query/latest/docs/framework/react/guides/prefetching) - prefetch on hover and router integration
