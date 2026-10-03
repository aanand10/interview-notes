# API calls, loading and error states in React

> **In one line:** I run independent calls in parallel with `Promise.all` or `allSettled`, model each request as a clear status (idle, loading, success, error), cancel stale requests with `AbortController`, and in real apps let a data library like TanStack Query handle caching, dedupe and retries.

## Key points
- Independent calls should start **together**, not one after another. `await` in a loop runs them in series.
- [`Promise.all`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/all) fails fast on the first rejection. [`Promise.allSettled`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/allSettled) waits for all and tells you each result.
- `fetch` only rejects on network errors. A 404 or 500 still resolves, so always check `res.ok`.
- Keep request state as one union (`status: "loading" | "success" | "error"`), not three separate booleans that can disagree.
- Cancel requests that are no longer needed with [`AbortController`](https://developer.mozilla.org/en-US/docs/Web/API/AbortController), and dedupe identical requests that are already in flight.

## Example
A small `useFetch` hook with a status union, `res.ok` check and cancellation.

```tsx
import { useEffect, useState } from "react";

type State<T> =
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error"; error: Error };

export function useFetch<T>(url: string): State<T> {
  const [state, setState] = useState<State<T>>({ status: "loading" });

  useEffect(() => {
    const controller = new AbortController();
    setState({ status: "loading" });

    fetch(url, { signal: controller.signal })
      .then((res) => {
        if (!res.ok) throw new Error(`HTTP ${res.status}`); // fetch does not reject on 4xx/5xx
        return res.json() as Promise<T>;
      })
      .then((data) => setState({ status: "success", data }))
      .catch((error) => {
        if (error.name === "AbortError") return; // we cancelled it on purpose
        setState({ status: "error", error });
      });

    return () => controller.abort(); // runs when url changes or component unmounts
  }, [url]);

  return state;
}

function Quote({ symbol }: { symbol: string }) {
  const q = useFetch<{ price: number }>(`/api/quote/${symbol}`);
  if (q.status === "loading") return <p>Loading...</p>;
  if (q.status === "error") return <p role="alert">Could not load {symbol}. Try again.</p>;
  return <p>{symbol}: {q.data.price}</p>;
}
```

## When to use it
Every screen in a trading app: dashboard loading positions, orders and news together; a quote panel that refetches when the symbol changes; an order form that must not be submitted twice.

## Likely questions

### How do you make multiple API calls using Promises?
Start all the calls first, then wait for them together. Each `fetch` starts the moment you call it, so putting them in an array and passing it to `Promise.all` runs them at the same time.

```tsx
const [positions, orders] = await Promise.all([
  fetch("/api/positions").then((r) => r.json()),
  fetch("/api/orders").then((r) => r.json()),
]);
```

If call B needs data from call A, then they must be sequential: `await` A, then call B.

### Promise.all: what happens if one call fails?
`Promise.all` rejects as soon as the first promise rejects, with that error. This is called fail-fast. But the other requests are **not stopped**: they keep running and their results are just ignored. If you really want to stop them, pass one `AbortController` signal to all of them and call `abort()` in the `catch`. I ran this with node: "orders" fails at 50ms, "prices" and "news" are slower.

```js
// fake API call: logs when it finishes, then resolves or rejects
const sleep = (ms, v, fail) => new Promise((res, rej) =>
  setTimeout(() => { console.log("settled", v); fail ? rej(new Error(v + " failed")) : res(v); }, ms));

try {
  await Promise.all([sleep(100, "prices"), sleep(50, "orders", true), sleep(150, "news")]);
} catch (e) {
  console.log("caught:", e.message);
}
```
```text
settled orders
caught: orders failed
settled prices
settled news
```
- The catch runs right after "orders" fails, without waiting.
- "prices" and "news" still finish later, the work is not cancelled.

### Promise.all vs Promise.allSettled?
`Promise.all` gives you an array of values, or rejects on the first error. Use it when you need **all** results to continue, for example a checkout that needs price and balance. `Promise.allSettled` never rejects. It waits for every promise and gives you `{ status: "fulfilled", value }` or `{ status: "rejected", reason }` for each. Use it when parts are independent, for example a dashboard where the news widget failing should not hide your positions.

```js
const r = await Promise.allSettled([sleep(10, "a"), sleep(20, "b", true)]);
// [{ status: "fulfilled", value: "a" }, { status: "rejected", reason: Error("b failed") }]
```

Related: `Promise.race` settles with the first one to settle (good for timeouts), `Promise.any` resolves with the first success.

### How do you handle errors in async API calls?
- Wrap `await` in `try/catch`, and check `res.ok` because `fetch` resolves even on 500.
- Turn errors into a **user-facing message** ("Couldn't load orders. Retry"), not a raw stack trace. Log the real error to monitoring.
- **Retry** only safe requests (GET) with backoff, for example 3 tries at 0.5s, 1s, 2s. Never blindly retry a "place order" POST unless the API supports an idempotency key.
- [Error boundaries](https://react.dev/reference/react/Component#catching-rendering-errors-with-an-error-boundary) catch errors thrown **during render**, not inside event handlers or async code. To use them for fetch errors, rethrow in render (TanStack Query has `throwOnError`, and Suspense-style data fetching does this).
- Ignore `AbortError`, since that is a cancel you asked for.

### How do you execute independent API calls in parallel?
Don't `await` inside a `for` loop, that runs them one by one. Map to promises and await them together. If there are many (say 200 symbols), limit concurrency, for example batches of 10, so you don't flood the server or hit rate limits.

```tsx
// Slow: one by one
for (const s of symbols) quotes.push(await getQuote(s));
// Fast: all at once
const quotes = await Promise.all(symbols.map(getQuote));
```

### How do you avoid duplicate API requests?
- **In-flight cache (dedupe map)**: keep a `Map` from URL to the pending promise. A second caller gets the same promise. Remove it when it settles.
- **TanStack Query / SWR**: components using the same `queryKey` share one request and one cache entry, automatically.
- **Disable the submit button** while a mutation runs, so double-clicks don't place two orders. Add an idempotency key on the server side too.
- **AbortController on re-run**: when the input changes (search box), abort the old request so an old, slower response can't overwrite the new one. Debounce typing too.
- **StrictMode**: in development React 18+ runs effects twice (mount, unmount, mount) to find missing cleanups. You see two requests in dev only. Fix it with a proper cleanup (abort) or a cache, not by removing StrictMode.

```js
const inflight = new Map();
function dedupe(key, fn) {
  if (inflight.has(key)) return inflight.get(key);
  const p = fn().finally(() => inflight.delete(key));
  inflight.set(key, p);
  return p;
}
const [x, y] = await Promise.all([dedupe("/quote/AAPL", fake), dedupe("/quote/AAPL", fake)]);
// network calls: 1, same object: true
// a later call after it settled makes a new request (calls: 2)
```

### How do you manage loading and error states in React?
I use one state with a status union instead of `isLoading`, `isError` and `data` booleans, so impossible combinations like "loading and error" can't happen. I wrap it in a custom hook like `useFetch` above so components stay clean. In a real app I'd use [TanStack Query](https://tanstack.com/query/latest): `useQuery` gives `isPending`, `isError`, `data`, plus caching, dedupe, retries and background refetch for free.

```tsx
const { data, isPending, isError, refetch } = useQuery({
  queryKey: ["positions"],
  queryFn: () => fetch("/api/positions").then((r) => r.json()),
});
```

### An API is slow: how do you keep the UI responsive?
- Show a **skeleton** that matches the final layout, instead of a blank page or full-page spinner.
- **Show cached data first** and refresh in the background (stale-while-revalidate), with a small "updating" hint.
- **Optimistic UI**: for safe actions like adding to a watchlist, update the UI at once and roll back if the call fails. For placing real orders, show "pending" instead, since money is involved.
- **Timeouts and cancel**: `AbortSignal.timeout(8000)` and a Cancel button, then show a retry message.
- **useTransition**: mark the state update that triggers the slow screen as non-urgent, so the current UI stays interactive and you can show `isPending`.
- Load the critical part first (price), and lazy-load the rest (news, charts).

### Why use a library like TanStack Query instead of useEffect + fetch?
`useEffect` fetching means you rebuild caching, dedupe, retries, race handling and refetch-on-focus yourself. A server-state library does this and keeps server data separate from UI state. The [React docs](https://react.dev/learn/synchronizing-with-effects#fetching-data) themselves recommend a framework or data library over raw effects.

## Common mistakes
- Not checking `res.ok`, so a 500 HTML page gets parsed as JSON.
- Race condition: old response arrives after the new one and overwrites it. Fix with abort or an "ignore" flag in cleanup.
- Thinking `Promise.all` cancels the other requests. It doesn't.
- Expecting error boundaries to catch errors in `useEffect` or `onClick` async code.
- Retrying non-idempotent POSTs, which can place a duplicate order.

## Resources
- [MDN: Promise.all](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/all) - fail-fast behaviour explained
- [MDN: AbortController](https://developer.mozilla.org/en-US/docs/Web/API/AbortController) - cancelling fetch requests
- [react.dev: You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect) - data fetching and race conditions in effects
- [TanStack Query docs](https://tanstack.com/query/latest) - caching, dedupe, retries for server state
