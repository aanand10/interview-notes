# Custom hooks

> **In one line:** A custom hook is a normal function whose name starts with `use` and that calls other hooks; it lets you reuse stateful logic between components, while each component still gets its own separate state.

## Key points
- **Rules of hooks**: call hooks only at the top level of a component or custom hook, never inside conditions, loops or nested functions, and never from regular functions.
- The rules exist because React identifies each hook by its **call order** in the render. If the order changes, state gets attached to the wrong hook.
- The `use` prefix tells React's lint rules (and readers) that the function may call hooks.
- Custom hooks **share logic, not state**. Two components calling `useWatchlist()` get two independent copies, unless the hook reads from context or an external store.
- Test hooks with `renderHook` from React Testing Library.

## Example
Four hooks interviewers love to ask for.

```tsx
import { useEffect, useRef, useState } from 'react';

// 1. useDebounce: value updates only after `delay` ms of no changes
export function useDebounce<T>(value: T, delay = 300): T {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id); // new keystroke cancels the old timer
  }, [value, delay]);
  return debounced;
}

// 2. useFetch: with abort, loading and error
export function useFetch<T>(url: string | null) {
  const [data, setData] = useState<T | null>(null);
  const [error, setError] = useState<Error | null>(null);
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    if (!url) return;
    const controller = new AbortController();
    setLoading(true);
    setError(null);
    fetch(url, { signal: controller.signal })
      .then((r) => {
        if (!r.ok) throw new Error(`HTTP ${r.status}`);
        return r.json() as Promise<T>;
      })
      .then((d) => { setData(d); setLoading(false); })
      .catch((e) => {
        if (e.name === 'AbortError') return; // stale request, ignore
        setError(e);
        setLoading(false);
      });
    return () => controller.abort();
  }, [url]);

  return { data, error, loading };
}

// 3. useLocalStorage: state that persists across reloads
export function useLocalStorage<T>(key: string, initial: T) {
  const [value, setValue] = useState<T>(() => {
    try {
      const raw = localStorage.getItem(key);
      return raw !== null ? (JSON.parse(raw) as T) : initial;
    } catch {
      return initial; // private mode, bad JSON, or SSR
    }
  });
  useEffect(() => {
    try {
      localStorage.setItem(key, JSON.stringify(value));
    } catch { /* quota full: ignore */ }
  }, [key, value]);
  return [value, setValue] as const;
}

// 4. useInterval: always calls the latest callback, no stale closure
export function useInterval(callback: () => void, delay: number | null) {
  const saved = useRef(callback);
  useEffect(() => {
    saved.current = callback; // keep the newest callback without restarting
  }, [callback]);
  useEffect(() => {
    if (delay === null) return; // null pauses
    const id = setInterval(() => saved.current(), delay);
    return () => clearInterval(id);
  }, [delay]);
}
```

Using them together in a symbol search:

```tsx
function SymbolSearch() {
  const [query, setQuery] = useState('');
  const debounced = useDebounce(query, 300);
  const { data, loading } = useFetch<string[]>(
    debounced ? `/api/search?q=${encodeURIComponent(debounced)}` : null,
  );
  const [recent, setRecent] = useLocalStorage<string[]>('recent-symbols', []);
  // ...
}
```

Svelte 5 equivalent: a plain function in a `.svelte.ts` file that uses runes; there are no rules about call order.

## When to use it
Whenever two components repeat the same effect plus state logic: `useLivePrice(symbol)` for a socket subscription, `useMarketOpen()` for market hours, `useDebounce` for search, `useMediaQuery` for responsive layouts, `useOnlineStatus` for showing an offline banner.

## Likely questions

### What are the rules of hooks and why do they exist?
Only call hooks at the top level, and only from React function components or custom hooks. React does not know hook names; it stores hook data in a list on the component and matches them by order: first `useState` call gets slot 1, and so on. If a hook is inside an `if`, the order can change between renders and React would hand you another hook's state. The `eslint-plugin-react-hooks` plugin enforces this. (The `use` API in React 19 is the one exception that can be called conditionally.)

### Write a useDebounce hook.
See above. The key points to say: store the debounced value in state, start a `setTimeout` in an effect whenever `value` changes, and clear it in cleanup so only the last change "wins". Include `delay` in the deps.

### Write a useFetch with abort.
See above. Mention: abort in cleanup to prevent race conditions and wasted requests; ignore `AbortError`; check `response.ok` because `fetch` does not reject on 404/500; accept `null` to skip fetching. In production I'd use TanStack Query because it adds caching, retries, dedupe and background refetch.

### Why does useInterval use a ref?
If the interval callback reads state directly, it captures the first render's values (stale closure). Restarting the interval on every change resets its timing. Storing the latest callback in a ref lets the interval stay running while always calling fresh code.

### Do two components using the same custom hook share state?
No. Each call is independent, like calling `useState` twice. Hooks share **logic**. To share **state**, lift it to a common parent, put it in context, or use an external store (Zustand, or `useSyncExternalStore` yourself) and have the hook read from there.

### How do you test a custom hook?
Use `renderHook` from `@testing-library/react`. It renders a tiny test component that calls your hook and gives you `result.current`. Wrap updates in `act`, and use fake timers for debounce or interval.

```tsx
import { renderHook, act } from '@testing-library/react';
import { vi, test, expect } from 'vitest';

test('useDebounce waits for the delay', () => {
  vi.useFakeTimers();
  const { result, rerender } = renderHook(({ v }) => useDebounce(v, 300), {
    initialProps: { v: 'A' },
  });
  rerender({ v: 'AAPL' });
  expect(result.current).toBe('A');      // not yet
  act(() => { vi.advanceTimersByTime(300); });
  expect(result.current).toBe('AAPL');   // updated after 300ms
  vi.useRealTimers();
});
```
For hooks that fetch, mock the network (for example with MSW) and use `waitFor`.

## Common mistakes
- Naming a hook without `use`, so lint rules don't check it.
- Calling hooks conditionally or after an early `return`.
- Expecting a custom hook to share state between components.
- Returning new object/function identities each render, which breaks memoised consumers. Wrap returned callbacks in `useCallback` if consumers depend on them.
- Forgetting cleanup inside the hook.

## Resources
- [react.dev: Reusing logic with custom hooks](https://react.dev/learn/reusing-logic-with-custom-hooks) - when and how to extract a hook
- [react.dev: Rules of Hooks](https://react.dev/reference/rules/rules-of-hooks) - the exact rules and why
- [Testing Library: renderHook](https://testing-library.com/docs/react-testing-library/api#renderhook) - testing hooks in isolation
- [MDN: setTimeout](https://developer.mozilla.org/en-US/docs/Web/API/Window/setTimeout) - timers used in debounce and interval
