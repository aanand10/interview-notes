# useEffect deep dive

> **In one line:** `useEffect` lets a component synchronise with something outside React (network, timers, WebSockets, the DOM API) after the render is painted, and its cleanup undoes that work before the next run or on unmount.

## Key points
- [`useEffect(setup, deps)`](https://react.dev/reference/react/useEffect) runs after the commit, usually after the browser paints. It is for **side effects**, not for computing values.
- **Dependency array**: no array = run after every render; `[]` = run once after mount; `[a, b]` = run when `a` or `b` changed (compared with `Object.is`). Every reactive value used inside must be listed.
- **Cleanup**: the function you return runs before the effect re-runs and when the component unmounts. Use it to unsubscribe, clear timers, abort fetches.
- In **StrictMode in development**, React mounts, unmounts and mounts again, so every effect runs setup, cleanup, setup. This is on purpose to find missing cleanups. It does not happen in production.
- Many effects are unnecessary. If you can **compute it during render** or do it **in an event handler**, do that instead.

## Example
Fetch a quote when the symbol changes, safely.

```tsx
import { useEffect, useState } from 'react';

type Quote = { symbol: string; price: number };

function QuoteCard({ symbol }: { symbol: string }) {
  const [quote, setQuote] = useState<Quote | null>(null);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const controller = new AbortController();
    setError(null);

    fetch(`/api/quote/${symbol}`, { signal: controller.signal })
      .then((r) => {
        if (!r.ok) throw new Error(`HTTP ${r.status}`);
        return r.json() as Promise<Quote>;
      })
      .then(setQuote)
      .catch((e) => {
        if (e.name !== 'AbortError') setError(e.message); // ignore our own aborts
      });

    // Runs when symbol changes or on unmount: the old request is cancelled,
    // so a slow AAPL response can't overwrite a newer TSLA one.
    return () => controller.abort();
  }, [symbol]);

  if (error) return <p role="alert">{error}</p>;
  if (!quote) return <p>Loading...</p>;
  return <p>{quote.symbol}: {quote.price}</p>;
}
```

Svelte 5 equivalent: `$effect(() => { ...; return () => cleanup(); })` with dependencies tracked automatically.

## When to use it
Subscribing to a price WebSocket, starting a polling interval, syncing with a chart library (`chart.setData`), listening to `window` resize or `visibilitychange`, logging analytics on page view. For data fetching in real apps, prefer a library such as TanStack Query or your framework's loader, which handles caching and races for you.

## Likely questions

### What are the rules for the dependency array?
List every value from the component scope that the effect reads: props, state, and functions or objects declared in the component. The `react-hooks/exhaustive-deps` lint rule checks this. Do not lie to the array to stop re-runs; instead, move the object or function inside the effect, use a functional state update, or move constants outside the component. Objects and functions created during render are new every time, so putting them in deps makes the effect run every render.

### What is the cleanup function for?
It undoes what setup did: close the socket, clear the interval, remove the listener, abort the request. React runs it with the **old** values before running setup with the new values, and once more on unmount. Without it you get memory leaks, duplicate subscriptions and "setState on unmounted component" style bugs.

```tsx
useEffect(() => {
  const ws = new WebSocket(`wss://feed.example.com/${symbol}`);
  ws.onmessage = (e) => setPrice(JSON.parse(e.data).price);
  return () => ws.close(); // old symbol's socket closes before the new one opens
}, [symbol]);
```

### Why does my effect run twice in development?
`<StrictMode>` in development deliberately mounts the component, runs effects, unmounts (runs cleanups), and mounts again. If your effect is correct, setup, cleanup, setup behaves the same as one setup. If you see two API calls or two sockets, your cleanup is missing. Do not "fix" it with a `useRef` flag; fix the cleanup. Production runs it once.

### How do you handle race conditions when fetching?
The race: user picks AAPL, then TSLA quickly. If AAPL's response arrives last, it overwrites TSLA. Two fixes: abort the old request with `AbortController` in cleanup (shown above), or use an `ignore` flag.

```tsx
useEffect(() => {
  let ignore = false;
  getQuote(symbol).then((q) => { if (!ignore) setQuote(q); });
  return () => { ignore = true; };
}, [symbol]);
```
Abort is better because it also saves bandwidth.

### When do you NOT need an effect?
- **Derived values**: `const fullName = first + ' ' + last` during render, not an effect that sets state.
- **Expensive calculations**: `useMemo`, not effect plus state.
- **User events**: submitting an order, showing a toast after a click. Put it in the event handler, because you know exactly what happened there.
- **Resetting state on prop change**: pass a `key` instead of an effect that clears state.
- **Notifying a parent**: call the parent's callback in the same handler instead of in an effect.
A good test: "Is this running because the component was **displayed**, or because the user **did** something?" Only the first belongs in an effect.

### useLayoutEffect vs useEffect?
[`useLayoutEffect`](https://react.dev/reference/react/useLayoutEffect) runs synchronously after DOM changes but **before the browser paints**. `useEffect` normally runs after paint. Use `useLayoutEffect` only when you must measure the DOM and update before the user sees it, such as positioning a tooltip next to a price cell to avoid a flicker. It blocks painting, so default to `useEffect`. It also does nothing on the server, which gives a warning in older SSR setups.

### Can the effect callback be async?
Not directly: an `async` function returns a promise, but React expects either nothing or a cleanup function. Define an async function inside the effect and call it.

## Common mistakes
- Missing dependencies, which gives stale values.
- Objects or functions in deps that are recreated every render, causing infinite loops when the effect sets state.
- No cleanup for sockets, timers or listeners.
- Using effects to sync state that should just be computed.
- Hiding the StrictMode double run with a ref instead of writing proper cleanup.

## Resources
- [react.dev: useEffect](https://react.dev/reference/react/useEffect) - full API, deps and cleanup rules
- [react.dev: You might not need an effect](https://react.dev/learn/you-might-not-need-an-effect) - the most asked-about guide on removing effects
- [react.dev: Synchronizing with effects](https://react.dev/learn/synchronizing-with-effects) - StrictMode double run and fetching patterns
- [MDN: AbortController](https://developer.mozilla.org/en-US/docs/Web/API/AbortController) - cancelling fetch requests
