# Data fetching in React

> **In one line:** You can fetch in `useEffect` yourself, but for real apps a server-state library like TanStack Query or SWR is better, because it gives you caching, deduping, retries and background refresh for free.

## Key points
- **fetch in useEffect** works, but you must handle loading, errors, race conditions, cancelling and caching by hand. See [You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect).
- **TanStack Query / SWR** treat server data as a cache keyed by a **query key**. Two components asking for the same key share one request (**dedupe**).
- **Stale-while-revalidate:** show cached (stale) data immediately, then refetch in the background and update. SWR is named after this.
- **Cancel** requests with an [`AbortController`](https://developer.mozilla.org/en-US/docs/Web/API/AbortController) so an old response cannot overwrite a newer one.
- For **live prices**, use a WebSocket (server pushes) and write updates into the cache; use polling only for slow-changing data.

## Example
### The manual way, done correctly

```tsx
function Quote({ symbol }: { symbol: string }) {
  const [data, setData] = useState<Quote | null>(null);
  const [error, setError] = useState<Error | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const controller = new AbortController();
    setLoading(true);
    setError(null);
    fetch(`/api/quote/${symbol}`, { signal: controller.signal })
      .then((r) => {
        if (!r.ok) throw new Error(`HTTP ${r.status}`); // fetch does not reject on 4xx/5xx
        return r.json();
      })
      .then(setData)
      .catch((e) => { if (e.name !== "AbortError") setError(e); })
      .finally(() => { if (!controller.signal.aborted) setLoading(false); });
    return () => controller.abort(); // symbol changed or unmounted: cancel old request
  }, [symbol]);

  if (loading) return <Spinner />;
  if (error) return <p role="alert">{error.message}</p>;
  return <p>{data!.symbol}: {data!.price}</p>;
}
```

### The same with TanStack Query

```tsx
import { useQuery } from "@tanstack/react-query";

function Quote({ symbol }: { symbol: string }) {
  const { data, isPending, error } = useQuery({
    queryKey: ["quote", symbol],
    queryFn: ({ signal }) => fetch(`/api/quote/${symbol}`, { signal }).then((r) => r.json()),
    staleTime: 5_000, // treat as fresh for 5s, no refetch in that window
  });
  if (isPending) return <Spinner />;
  if (error) return <p role="alert">{error.message}</p>;
  return <p>{data.symbol}: {data.price}</p>;
}
```

## When to use it
- TanStack Query: portfolio, order history, account balances, anything read from REST with mutations.
- WebSocket + cache: live order book and price ticks.
- Svelte equivalent: SvelteKit `load` functions, or TanStack Query's Svelte adapter.

## Likely questions
### Why not just fetch in useEffect?
It works for one simple screen, but you rewrite the same code everywhere: loading and error state, race conditions, no cache so going back refetches and shows a spinner, no dedupe so two widgets make two calls, no retry. In Strict Mode, effects run twice in development, so you also see double requests. A library solves all this in one place.

### What do caching, dedupe, retries and stale-while-revalidate mean in TanStack Query?
**Caching:** data is stored by query key, so returning to a page shows it instantly. **Dedupe:** several components with the same key share one in-flight request. **Retries:** failed queries retry 3 times with backoff by default. **Stale-while-revalidate:** with the default `staleTime` of 0, cached data shows immediately and is refetched on mount, window focus or reconnect. Unused data is garbage-collected after `gcTime`, 5 minutes by default.

### How do you cancel requests?
Pass an `AbortController.signal` to `fetch` and call `abort()` in the effect cleanup. TanStack Query gives you `signal` in the `queryFn` and aborts when the query is no longer needed. Also note `fetch` only rejects on network errors, so check `res.ok` yourself.

### How do you do an optimistic update?
Update the UI before the server confirms, then roll back on error.

```tsx
const qc = useQueryClient();
const addToWatchlist = useMutation({
  mutationFn: (symbol: string) => api.addToWatchlist(symbol),
  onMutate: async (symbol) => {
    await qc.cancelQueries({ queryKey: ["watchlist"] });       // stop refetch overwriting us
    const previous = qc.getQueryData<string[]>(["watchlist"]);
    qc.setQueryData<string[]>(["watchlist"], (old = []) => [...old, symbol]);
    return { previous };                                         // context for rollback
  },
  onError: (_err, _symbol, ctx) => qc.setQueryData(["watchlist"], ctx?.previous),
  onSettled: () => qc.invalidateQueries({ queryKey: ["watchlist"] }), // sync with server
});
```

I would never do this for placing a real order: money actions should show a pending state and wait for the server.

### Polling vs WebSocket for live prices?
Polling (`refetchInterval: 5000` in TanStack Query, `refreshInterval` in SWR) is simple and works through any proxy, but wastes requests and is always a bit late. A [WebSocket](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket) keeps one connection open and the server pushes ticks, so it is right for live prices. I open the socket in a hook or a store outside React, write ticks into the query cache with `setQueryData`, and throttle UI updates (for example once per animation frame) so 50 ticks a second do not cause 50 renders. I also handle reconnect with backoff and resubscribe on reconnect.

```tsx
useEffect(() => {
  const ws = new WebSocket("wss://stream.example.com/prices");
  ws.onopen = () => ws.send(JSON.stringify({ subscribe: ["AAPL", "TSLA"] }));
  ws.onmessage = (e) => {
    const tick = JSON.parse(e.data);
    qc.setQueryData(["quote", tick.symbol], tick); // components using this key re-render
  };
  return () => ws.close();
}, [qc]);
```

## Common mistakes
- Forgetting the cleanup, so a slow old response overwrites the new symbol's price (race condition).
- Putting server data in Redux or context and syncing it by hand, instead of a server-state cache.
- Query keys that miss a variable, for example `["quote"]` instead of `["quote", symbol]`, so all symbols share one cache entry.

## Resources
- [react.dev: Fetching data with effects](https://react.dev/learn/synchronizing-with-effects#fetching-data) - pitfalls of manual fetching
- [TanStack Query: Overview](https://tanstack.com/query/latest/docs/framework/react/overview) - caching, retries, mutations
- [SWR docs](https://swr.vercel.app/) - stale-while-revalidate library
- [MDN: AbortController](https://developer.mozilla.org/en-US/docs/Web/API/AbortController) - cancelling fetch
