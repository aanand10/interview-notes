# Context and state management

> **In one line:** Context passes a value deep down the tree without prop drilling, but every consumer re-renders when that value changes, so I use it for low-frequency global data and pick a store (Zustand, Redux Toolkit) for busy client state and React Query for server state.

## Key points
- **Prop drilling** means passing props through many layers that don't use them, just to reach a deep child. It is fine for 2-3 levels; painful beyond that.
- [Context](https://react.dev/learn/passing-data-deeply-with-context): `createContext`, a `Provider` with a `value`, and `useContext` (or `use` in React 19) to read it.
- **Re-render cost**: when the provider's `value` changes (by `Object.is`), **every** component that reads that context re-renders, even if it only uses one field. `memo` does not stop this.
- **Split contexts** by how often they change (auth, theme, watchlist), and memoise the value object.
- **Server state** (data from APIs: quotes, positions, order history) is different from **client state** (UI choices: theme, open panels, form drafts). Use a server-state library for the first.

## Example
Auth + theme + watchlist in a trading app, each in its own context.

```tsx
import { createContext, useContext, useMemo, useState, type ReactNode } from 'react';

// Theme: changes rarely
type Theme = 'light' | 'dark';
const ThemeContext = createContext<{ theme: Theme; toggle: () => void } | null>(null);

export function ThemeProvider({ children }: { children: ReactNode }) {
  const [theme, setTheme] = useState<Theme>('dark');
  // Memoised so consumers don't re-render when the provider's parent re-renders
  const value = useMemo(
    () => ({ theme, toggle: () => setTheme((t) => (t === 'dark' ? 'light' : 'dark')) }),
    [theme],
  );
  return <ThemeContext.Provider value={value}>{children}</ThemeContext.Provider>;
}

export function useTheme() {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error('useTheme must be used inside ThemeProvider');
  return ctx;
}

// Auth: changes on login/logout only. Same pattern: AuthProvider + useAuth()
// Watchlist symbols: user's list, changes on add/remove. Same pattern.

export function AppProviders({ children }: { children: ReactNode }) {
  return (
    <AuthProvider>
      <ThemeProvider>
        <WatchlistProvider>{children}</WatchlistProvider>
      </ThemeProvider>
    </AuthProvider>
  );
}
```
Live **prices** are not in context. They change many times a second; putting them in context would re-render every consumer on every tick. They go in a store with selectors, or are fetched per row.

```tsx
// Zustand: components subscribe to a slice, so only the AAPL row re-renders on an AAPL tick
import { create } from 'zustand';

type PriceState = { prices: Record<string, number>; setPrice: (s: string, p: number) => void };

export const usePrices = create<PriceState>((set) => ({
  prices: {},
  setPrice: (symbol, price) =>
    set((state) => ({ prices: { ...state.prices, [symbol]: price } })),
}));

function PriceCell({ symbol }: { symbol: string }) {
  const price = usePrices((s) => s.prices[symbol]); // selector
  return <td>{price?.toFixed(2) ?? '-'}</td>;
}
```

Server data uses TanStack Query:

```tsx
import { useQuery } from '@tanstack/react-query';

function Positions() {
  const { data, isPending, error } = useQuery({
    queryKey: ['positions'],
    queryFn: () => fetch('/api/positions').then((r) => r.json()),
    staleTime: 10_000, // treat as fresh for 10s, then refetch in background
  });
  // ...
}
```

Svelte 5 equivalent: a shared `$state` object exported from a `.svelte.ts` module, or `setContext/getContext`; Svelte's fine-grained updates avoid the "all consumers re-render" problem.

## When to use it
- **Context**: theme, locale, auth user, feature flags, the current account. Low-frequency, read in many places.
- **Zustand**: small-to-medium client state that changes often (live prices, layout, selected symbol), with minimal boilerplate.
- **Redux Toolkit**: large teams and apps that want strict patterns, middleware, time-travel DevTools, and predictable reducers (complex order-entry workflows, audit-friendly state).
- **TanStack Query (React Query)** or RTK Query: anything from the server. It gives caching, dedupe, retries, refetch on focus, and loading/error states.

## Likely questions

### What is prop drilling and how do you avoid it?
Passing a prop through components that don't need it so a deep child can use it. First try **component composition**: pass JSX as `children` so the middle layers don't need the data. If many distant components need it, use context. If it also changes often, use a store.

### What is the re-render cost of context?
When the provider value changes, React re-renders every component that calls `useContext` for it, regardless of which field they use, and `memo` can't block it. A common hidden bug is `value={{ user, login }}`: a new object every render of the provider's parent, so all consumers re-render even if nothing changed. Fix with `useMemo`, split contexts, or a store with selectors.

### How do you split contexts?
Separate by topic and update frequency: `AuthContext`, `ThemeContext`, `WatchlistContext`. Another trick is to split **state** and **dispatch** into two contexts: components that only trigger actions read the dispatch context, which never changes, so they don't re-render when state changes.

### Redux Toolkit vs Zustand vs React Query?
They solve different problems. React Query manages **server state**: a cache of remote data plus fetching rules. Redux Toolkit and Zustand manage **client state**. Zustand is a tiny hook-based store, no providers, select what you need. Redux Toolkit is more structured: slices, reducers, middleware, great DevTools, good for big teams. In a modern app I'd use React Query for API data, context for auth/theme, and Zustand (or RTK if the team already uses Redux) for shared busy client state. Moving server data out of Redux usually shrinks the store a lot.

### Is context a state management tool?
Not really; it is a **transport** mechanism. The state still lives in `useState` or `useReducer` inside the provider. Context plus `useReducer` is fine for moderate needs, but it has no selectors, so it is a poor fit for high-frequency data.

### How would you design state for a trading app?
- Auth/session: context (rare changes), token kept out of JS-readable storage if possible.
- Theme/locale: context.
- Watchlist symbols: server state via React Query, with an optimistic update when adding/removing.
- Live prices: WebSocket feeding a Zustand store keyed by symbol; rows subscribe with selectors, and updates can be throttled to once per animation frame.
- Order form draft: local `useState` in the form.

## Common mistakes
- One giant context with everything in it.
- Inline object as provider `value` without `useMemo`.
- Putting server data in Redux and hand-writing loading/error/caching.
- Putting high-frequency data (ticks) in context.
- Reaching for global state for something only one screen uses.

## Resources
- [react.dev: Passing data deeply with context](https://react.dev/learn/passing-data-deeply-with-context) - context basics and alternatives to try first
- [react.dev: Scaling up with reducer and context](https://react.dev/learn/scaling-up-with-reducer-and-context) - state and dispatch split pattern
- [TanStack Query overview](https://tanstack.com/query/latest/docs/framework/react/overview) - why server state is different
- [Redux Toolkit: Getting started](https://redux-toolkit.js.org/introduction/getting-started) - modern Redux with slices
