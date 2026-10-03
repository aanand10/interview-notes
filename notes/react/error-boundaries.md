# Error boundaries

> **In one line:** An error boundary is a component that catches errors thrown while rendering its children, and shows a fallback UI instead of letting the whole app go blank.

## Key points
- Without a boundary, an error thrown during render **unmounts the whole React tree**. The user sees a white screen.
- A boundary catches errors from **rendering, lifecycle methods and constructors** of the components *below* it.
- It does **not** catch errors in event handlers, async code (`setTimeout`, promises, `fetch`), server-side rendering, or errors thrown inside the boundary itself.
- There is still **no hook** for this. You write a class component with `static getDerivedStateFromError` (to show the fallback) and `componentDidCatch` (to log). Most teams use the [react-error-boundary](https://github.com/bvaughn/react-error-boundary) library instead.
- Place several boundaries: one around the whole app, plus smaller ones around risky widgets (a chart, a news feed), so one broken widget does not kill the page.

## Example
A class-based boundary with a fallback, reset and logging.

```tsx
import { Component, type ErrorInfo, type ReactNode } from "react";
import * as Sentry from "@sentry/react";

type Props = { fallback: (reset: () => void) => ReactNode; children: ReactNode };
type State = { error: Error | null };

export class ErrorBoundary extends Component<Props, State> {
  state: State = { error: null };

  // 1. Called during render: switch to the fallback UI.
  static getDerivedStateFromError(error: Error): State {
    return { error };
  }

  // 2. Called after the error is committed: good place for side effects like logging.
  componentDidCatch(error: Error, info: ErrorInfo) {
    Sentry.captureException(error, {
      contexts: { react: { componentStack: info.componentStack } },
    });
  }

  reset = () => this.setState({ error: null }); // try rendering children again

  render() {
    if (this.state.error) return this.props.fallback(this.reset);
    return this.props.children;
  }
}

// Usage: if the chart crashes, the order form still works.
<ErrorBoundary fallback={(reset) => (
  <div role="alert">
    Chart failed to load. <button onClick={reset}>Retry</button>
  </div>
)}>
  <PriceChart symbol="AAPL" />
</ErrorBoundary>
```

## When to use it
- Around each independent panel of a trading dashboard: chart, watchlist, order book, news. A bad data point in the news feed should not hide the order form.
- At route level, so one broken page shows "Something went wrong" with a link home.
- Svelte 5 equivalent: the `<svelte:boundary>` element with a `failed` snippet.

## Likely questions
### What does an error boundary catch and not catch?
It catches errors thrown while React is rendering the children, in lifecycle methods and in constructors. It does not catch errors in event handlers, because those run outside rendering, so React still knows how to show the UI. It does not catch async errors from `setTimeout` or a rejected promise, it does not work during SSR, and it does not catch its own errors, only its children's. For event handlers I use a normal `try/catch` and set some error state.

### How do you send an async or event-handler error to a boundary?
You catch it yourself and push it into React. With react-error-boundary, the `useErrorBoundary` hook gives `showBoundary(error)`. A plain trick is to store the error in state and `throw` it during the next render. In React 19, errors thrown inside a transition (`startTransition` or a form Action) also go to the nearest boundary.

```tsx
import { useErrorBoundary } from "react-error-boundary";

function CancelOrderButton({ id }: { id: string }) {
  const { showBoundary } = useErrorBoundary();
  async function cancel() {
    try {
      await fetch(`/api/orders/${id}`, { method: "DELETE" });
    } catch (err) {
      showBoundary(err); // now the nearest boundary shows its fallback
    }
  }
  return <button onClick={cancel}>Cancel</button>;
}
```

### Why use the react-error-boundary library?
It saves writing the class and adds useful extras: `FallbackComponent` or `fallbackRender` for the UI, `onError` for logging, `onReset` plus `resetKeys` to reset automatically when some value changes, and the `useErrorBoundary` hook.

```tsx
import { ErrorBoundary } from "react-error-boundary";

<ErrorBoundary
  fallbackRender={({ error, resetErrorBoundary }) => (
    <div role="alert">
      <p>Could not load {symbol}: {error.message}</p>
      <button onClick={resetErrorBoundary}>Try again</button>
    </div>
  )}
  onError={(error, info) => Sentry.captureException(error)}
  resetKeys={[symbol]} // switching symbol clears the error
>
  <Quote symbol={symbol} />
</ErrorBoundary>
```

### How does reset work?
Reset just clears the error state so the boundary renders its children again. If nothing changed, it will crash again, so usually you also refetch or clear a cache in `onReset`. `resetKeys` is handy: when the selected symbol changes, the boundary resets by itself.

### How do you log errors to Sentry?
Call `Sentry.captureException` in `componentDidCatch` or in `onError`, and include the component stack. Sentry also ships its own `Sentry.ErrorBoundary` component that does this for you. In React 19 you can also pass `onCaughtError` and `onUncaughtError` to `createRoot` to log every error in one place.

## Common mistakes
- Expecting a boundary to catch a failed `fetch` in `useEffect`. It will not, unless you rethrow it during render.
- One boundary for the whole app only. Then any small error replaces the whole page.
- In development, React shows the error overlay even when a boundary caught it. Check the production build to see the real fallback.

## Resources
- [react.dev: Catching rendering errors with an error boundary](https://react.dev/reference/react/Component#catching-rendering-errors-with-an-error-boundary) - official class API
- [react-error-boundary on GitHub](https://github.com/bvaughn/react-error-boundary) - the library most teams use
- [Sentry React docs](https://docs.sentry.io/platforms/javascript/guides/react/) - logging and Sentry.ErrorBoundary
- [svelte.dev: svelte:boundary](https://svelte.dev/docs/svelte/svelte-boundary) - the Svelte 5 equivalent
