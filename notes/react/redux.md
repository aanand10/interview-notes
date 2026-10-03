# Redux and Redux Toolkit

> **In one line:** Redux keeps app-wide state in one central store; components dispatch actions describing what happened, pure reducers compute the new state, and every component subscribed to that slice re-renders.

## Key points
- **State management** means deciding where data lives, who can change it, and how the UI updates when it changes.
- **Local state** (`useState`) belongs to one component, like an input value. **Global state** is needed by many distant components, like the logged-in user or the watchlist.
- Redux has three parts: the **store** (one object holding state), **actions** (plain objects like `{ type: "watchlist/add", payload: "AAPL" }`), and **reducers** (pure functions `(state, action) => newState`).
- [Redux Toolkit (RTK)](https://redux-toolkit.js.org/) is the official, modern way to write Redux. `createSlice` makes actions and reducers together and uses Immer, so "mutating" code is safely turned into immutable updates.
- Server data (API responses) usually belongs in a cache like TanStack Query or RTK Query, not in hand-written Redux state.

## Redux data flow

```text
  UI (click "Add AAPL")
     |
     v
  dispatch({ type: "watchlist/add", payload: "AAPL" })
     |
     v
  reducer(oldState, action) -> newState     (pure, no side effects)
     |
     v
  store saves newState
     |
     v
  subscribers (useSelector) compare their selected value
     |
     v
  only components whose selected value changed re-render
```

One-way flow: data goes down, actions go up. Every change goes through dispatch, so it is predictable and traceable.

## Example
A watchlist slice with Redux Toolkit and typed hooks.

```tsx
// store.ts
import { configureStore, createSlice, type PayloadAction } from "@reduxjs/toolkit";
import { useDispatch, useSelector } from "react-redux";

type WatchlistState = { symbols: string[] };
const initialState: WatchlistState = { symbols: ["AAPL"] };

const watchlistSlice = createSlice({
  name: "watchlist",
  initialState,
  reducers: {
    add(state, action: PayloadAction<string>) {
      // Looks like mutation, but Immer makes it an immutable update
      if (!state.symbols.includes(action.payload)) state.symbols.push(action.payload);
    },
    remove(state, action: PayloadAction<string>) {
      state.symbols = state.symbols.filter((s) => s !== action.payload);
    },
  },
});

export const { add, remove } = watchlistSlice.actions; // action creators, auto-generated

export const store = configureStore({
  reducer: { watchlist: watchlistSlice.reducer }, // DevTools + thunk set up for you
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
export const useAppSelector = useSelector.withTypes<RootState>();
export const useAppDispatch = useDispatch.withTypes<AppDispatch>();
```

```tsx
// main.tsx: wrap the app once
import { Provider } from "react-redux";
<Provider store={store}><App /></Provider>;

// Watchlist.tsx: any component, any depth, no props passed down
function Watchlist() {
  const symbols = useAppSelector((s) => s.watchlist.symbols); // subscribe to a slice
  const dispatch = useAppDispatch();
  return (
    <ul>
      {symbols.map((sym) => (
        <li key={sym}>
          {sym} <button onClick={() => dispatch(remove(sym))}>Remove</button>
        </li>
      ))}
      <button onClick={() => dispatch(add("TSLA"))}>Add TSLA</button>
    </ul>
  );
}
```

## When to use it
Large apps with lots of shared, frequently changing client state and many developers: a trading terminal where the watchlist, open order panel, selected symbol and layout are read and changed from many places. Redux DevTools (time-travel, action log) is very useful for debugging "why did this order panel change?".

## Likely questions

### What is state management in React?
It is how we store data that changes and keep the UI in sync with it. React gives us `useState`, `useReducer` and Context for this. As an app grows, state must be shared between components that are far apart, so we decide where each piece lives and how updates flow. Libraries like Redux, Zustand or TanStack Query help when the built-in tools get messy.

### What is the difference between local and global state?
Local state is used by one component or a small subtree, like whether a dropdown is open or the quantity typed in an order form. Keep it in `useState` close to where it is used. Global state is needed by many unrelated parts of the app, like the user session, theme or watchlist. My rule: start local, lift up only when siblings need it, and go global only when many distant components need it.

### What is Redux and how does it work?
Redux is a predictable state container. All shared state lives in one store. To change it, a component dispatches an action, a plain object saying what happened. The store passes the current state and the action to a reducer, a pure function that returns the new state without mutating the old one. The store saves it and notifies subscribers; with React-Redux, `useSelector` re-renders only components whose selected value changed.

### Explain the Redux data flow.
UI -> `dispatch(action)` -> reducer -> store -> subscribers re-render. It is one-way: the view never changes state directly. Because reducers are pure and every change is an action, you can log, replay and time-travel through changes in Redux DevTools. Async work like API calls happens before the reducer, in a thunk (`createAsyncThunk`) or RTK Query, which then dispatches normal actions.

### What are actions, reducers and the store?
- **Action**: a plain object `{ type, payload }` that describes an event, like `"orders/placed"`.
- **Reducer**: a pure function `(state, action) => newState`. Same input, same output, no API calls, no randomness.
- **Store**: the single object that holds state, with `getState()`, `dispatch(action)` and `subscribe(listener)`. In RTK you create it with `configureStore`.

### What are the advantages of Redux?
Predictable updates through one path, a single source of truth, great DevTools with time-travel debugging, easy testing because reducers are pure functions, middleware for logging or analytics, fine-grained subscriptions with `useSelector`, and a well-known pattern that scales across big teams.

### How does Redux avoid prop drilling?
Prop drilling is passing props through many layers that do not need them, just to reach a deep child. With Redux, the store is provided once at the top with `<Provider>`, and any component at any depth reads what it needs with `useSelector` and updates it with `useDispatch`. The middle components stay clean.

### Why use Redux instead of building your own state system?
You could build a store with Context and `useReducer`, but you would need to solve problems Redux already solved: avoiding unnecessary re-renders (Context re-renders all consumers on every change), selector memoization, middleware, async handling, DevTools, SSR hydration, and TypeScript typing. Redux is battle-tested, documented and familiar to new team members, so you spend time on features, not on infrastructure.

### What are the limitations of Redux?
It adds boilerplate and concepts (less with RTK, but still more than `useState`). It is overkill for small apps. Everything goes through one global store, which can encourage putting local state there by mistake. Hand-writing server data caching (loading flags, refetch, invalidation) is painful, which is why RTK Query and TanStack Query exist. There is also a learning curve with immutability and selectors.

### When would you use the Context API instead of Redux?
Context is built in and great for values that change rarely and are needed widely: theme, locale, current user, feature flags. It is a way to pass data, not a full state manager. Every consumer re-renders when the value changes, so it is a poor fit for high-frequency data like live prices. If the shared state is large, changes often, or has complex update logic, I pick Redux or Zustand.

### When should you avoid global state?
When only one component or a small subtree uses the data, like form inputs, modal open state or hover state. When the data is really server data; use TanStack Query instead. When it can be derived from other state or from the URL; filters and selected tab often belong in URL search params so they are shareable. Global state makes components harder to reuse and test, so keep it small.

### What about Zustand and React Query?
[Zustand](https://zustand.docs.pmnd.rs/) is a tiny global store: `create((set) => ({ symbols: [], add: (s) => set(...) }))`, no Provider, selector-based subscriptions, very little boilerplate. Good for small to medium apps. [TanStack Query](https://tanstack.com/query/latest) (React Query) manages server state: fetching, caching, deduping, background refetch and retries. In many modern apps, TanStack Query handles API data and a small Zustand or Context store handles the little client state left. In Svelte, a `$state` object in a `.svelte.ts` module often replaces all of this.

## Common mistakes
- Mutating state in a plain reducer (`state.push(x)`). Outside of RTK's Immer, always return a new object.
- Selecting the whole state (`useSelector(s => s)`), which re-renders on every change. Select the smallest value you need.
- Returning a new array or object from a selector every time (`s.list.filter(...)`), which re-renders every time. Use `createSelector` to memoize.
- Putting API responses, loading flags and form inputs in Redux by default.
- Side effects (fetch, `Date.now()`) inside reducers.

## Resources
- [Redux Essentials: Overview and concepts](https://redux.js.org/tutorials/essentials/part-1-overview-concepts) - the data flow explained with diagrams
- [Redux Toolkit: createSlice](https://redux-toolkit.js.org/api/createSlice) - the slice API used above
- [react.dev: Passing data deeply with Context](https://react.dev/learn/passing-data-deeply-with-context) - when Context is enough
- [TanStack Query overview](https://tanstack.com/query/latest/docs/framework/react/overview) - server state vs client state
