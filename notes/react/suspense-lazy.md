# Suspense, lazy loading and code splitting

> **In one line:** `React.lazy` loads a component's code only when it is first rendered, and `<Suspense>` shows a fallback (like a spinner) while that code, or data, is still loading.

## Key points
- **Code splitting** means breaking one big JS bundle into smaller chunks. A dynamic `import()` tells the bundler (Vite, webpack) to make a separate chunk.
- [`lazy(() => import("./X"))`](https://react.dev/reference/react/lazy) turns that chunk into a component. The module must have a **default export**.
- [`<Suspense fallback={...}>`](https://react.dev/reference/react/Suspense) shows the fallback while anything inside it "suspends" (is waiting). The nearest Suspense above wins.
- Suspense also works for **data**, but only with Suspense-aware sources: React 19's `use(promise)`, frameworks like Next.js, or libraries like TanStack Query's `useSuspenseQuery`. A plain `fetch` in `useEffect` does not suspend.
- Suspense handles **loading**. Error boundaries handle **failure**. Use them together.

## Example
Route-based splitting with a loading state and error handling.

```tsx
import { lazy, Suspense } from "react";
import { BrowserRouter, Routes, Route } from "react-router-dom";
import { ErrorBoundary } from "react-error-boundary";

// Each page becomes its own chunk, downloaded on first visit.
const Dashboard = lazy(() => import("./pages/Dashboard"));
const Portfolio = lazy(() => import("./pages/Portfolio"));
const Reports = lazy(() => import("./pages/Reports")); // heavy charts, rarely used

export function App() {
  return (
    <BrowserRouter>
      <ErrorBoundary fallback={<p role="alert">Could not load this page.</p>}>
        <Suspense fallback={<PageSpinner />}>
          <Routes>
            <Route path="/" element={<Dashboard />} />
            <Route path="/portfolio" element={<Portfolio />} />
            <Route path="/reports" element={<Reports />} />
          </Routes>
        </Suspense>
      </ErrorBoundary>
    </BrowserRouter>
  );
}
```

## When to use it
- Split by **route** first: the biggest, easiest win. Users on the dashboard never download the reports page.
- Then split **heavy, rarely used widgets**: a charting library, a rich text editor, a KYC document uploader, an "advanced order" modal.
- Do not split tiny components. Each chunk is an extra network request.
- Svelte equivalent: SvelteKit splits per route automatically, and you can `await import()` a component yourself.

## Likely questions
### How do React.lazy and Suspense work together?
`lazy` takes a function that returns a promise of a module. The first time React renders the lazy component, the promise is not ready, so the component suspends. React then walks up to the nearest `<Suspense>` and shows its fallback. When the chunk arrives, React renders the real component. Later renders use the cached module, so there is no fallback again.

### What is Suspense for data, and how does use() fit in?
In React 19, [`use(promise)`](https://react.dev/reference/react/use) reads a promise inside render. If it is still pending, the component suspends and the nearest Suspense shows the fallback. If it rejects, the nearest error boundary shows. The key rule: the promise must be **stable**, created outside render (for example passed from a Server Component, a loader, or a cache), not created fresh on every render.

```tsx
import { use, Suspense } from "react";

function Holdings({ holdingsPromise }: { holdingsPromise: Promise<Holding[]> }) {
  const holdings = use(holdingsPromise); // suspends until resolved
  return <ul>{holdings.map((h) => <li key={h.symbol}>{h.symbol}: {h.qty}</li>)}</ul>;
}

// The parent (or a router loader) starts the request once.
<Suspense fallback={<p>Loading holdings...</p>}>
  <Holdings holdingsPromise={holdingsPromise} />
</Suspense>
```

### How do you design good loading states?
Put Suspense boundaries where a loading state makes sense to the user, not around every component. A skeleton for the whole portfolio page, then smaller boundaries for slow parts like news, so fast parts show first. Nested boundaries reveal content step by step. Siblings inside one boundary appear together, which avoids a "popcorn" effect.

### Why does the spinner replace content I already showed?
If an update suspends a boundary that is already showing content, React would hide it and show the fallback. Wrap the update in [`startTransition`](https://react.dev/reference/react/useTransition) so React keeps the old screen visible while the new one loads. Routers usually do this for navigation for you.

### How do errors work with Suspense?
If a lazy chunk fails to download (user went offline, or a new deploy removed the old chunk) or a `use()` promise rejects, the error is thrown to the nearest **error boundary**. So wrap Suspense in an error boundary with a "Retry" button. Order: error boundary outside, Suspense inside, is a common setup.

## Common mistakes
- Calling `lazy()` inside a component. It creates a new component every render and resets state. Always declare it at the top level of the module.
- Using a named export with `lazy`. Fix: `lazy(() => import("./X").then((m) => ({ default: m.X })))`.
- Creating the promise for `use()` inside the component body, which causes an endless loop of suspending.

## Resources
- [react.dev: lazy](https://react.dev/reference/react/lazy) - lazy loading API and pitfalls
- [react.dev: Suspense](https://react.dev/reference/react/Suspense) - fallbacks, nesting, transitions
- [react.dev: use](https://react.dev/reference/react/use) - reading promises and context in React 19
- [web.dev: Code splitting with React.lazy and Suspense](https://web.dev/articles/code-splitting-suspense) - why splitting helps performance
