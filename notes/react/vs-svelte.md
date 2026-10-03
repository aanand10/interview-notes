# React vs Svelte

> **In one line:** React is a runtime library that re-runs your component function and diffs a virtual DOM to find changes, while Svelte is a compiler that turns components into code that updates exactly the DOM nodes that depend on changed state.

## Key points
- **Runtime vs compiler:** React ships a runtime (react + react-dom) that does diffing in the browser. Svelte does most work at build time and ships a small runtime plus compiled update code.
- **Reactivity model:** React uses hooks (`useState`, `useEffect`) and re-renders the whole component on change. Svelte 5 uses [runes](https://svelte.dev/docs/svelte/what-are-runes) (`$state`, `$derived`, `$effect`), which are **signals**: fine-grained, so only the exact reads update.
- **Re-render model:** in React, a parent re-render re-renders all children by default, unless memoized. In Svelte, components do not "re-render"; the component script runs once and only bindings update.
- **Ecosystem:** React's is much bigger (libraries, jobs, React Native, Next.js). Svelte's is smaller but growing, with SvelteKit as the official full-stack framework.
- **Mental model is close:** components, props, one-way data flow, composition, effects. Moving between them is mostly learning the API, not new ideas.

## Example
The same price card in both.

```tsx
// React
import { useState, useMemo, useEffect } from "react";

export function PriceCard({ symbol, qty }: { symbol: string; qty: number }) {
  const [price, setPrice] = useState(0);
  const value = useMemo(() => price * qty, [price, qty]); // derived value

  useEffect(() => {
    const ws = new WebSocket(`wss://stream.example.com/${symbol}`);
    ws.onmessage = (e) => setPrice(JSON.parse(e.data).price); // whole component re-runs
    return () => ws.close();
  }, [symbol]); // must list dependencies by hand

  return <p>{symbol}: {price} (position {value.toFixed(2)})</p>;
}
```

```svelte
<!-- Svelte 5 -->
<script lang="ts">
  let { symbol, qty }: { symbol: string; qty: number } = $props();
  let price = $state(0);
  let value = $derived(price * qty); // dependencies tracked automatically

  $effect(() => {
    const ws = new WebSocket(`wss://stream.example.com/${symbol}`);
    ws.onmessage = (e) => (price = JSON.parse(e.data).price); // only the two text nodes update
    return () => ws.close();
  });
</script>

<p>{symbol}: {price} (position {value.toFixed(2)})</p>
```

### Quick mapping

| Idea | React | Svelte 5 |
| --- | --- | --- |
| Local state | `useState` | `$state` |
| Derived value | compute in render / `useMemo` | `$derived` |
| Side effect | `useEffect` with deps array | `$effect` (auto-tracked) |
| Props | function argument | `$props()` |
| Two-way binding | `value` + `onChange` | `bind:value` |
| Children / slots | `children`, render props | snippets, `{@render}` |
| Shared state | Context, Zustand, Redux | `.svelte.ts` module with `$state`, context |
| Global store subscribe | `useSyncExternalStore` | stores or runes in modules |
| Error boundary | class / react-error-boundary | `<svelte:boundary>` |
| Full-stack framework | Next.js, Remix / React Router | SvelteKit |

## When to use it
- **Choose React** when: the team or hiring pool knows React, you need a specific library (big data grids, design systems), you want React Native for mobile, or you need Server Components.
- **Choose Svelte** when: performance and bundle size really matter (high-frequency live price updates, low-end phones in emerging markets), the team is small and values less boilerplate, or you want a fast, simple full-stack setup with SvelteKit.
- For a trading UI with many ticking cells, Svelte's fine-grained updates are a natural fit; in React you get there with memoization, external stores and careful component splitting.

## Likely questions
### What is the difference between a virtual DOM and a compiler approach?
React keeps a virtual DOM, a JS object tree of what the UI should look like. On each state change it re-runs the component, builds a new tree, diffs it with the old one, and applies the differences to the real DOM. Svelte's compiler knows at build time which DOM nodes depend on which state, so it generates direct update code like "set this text node". No diffing step at runtime, so less work per update and less JS shipped.

### How do hooks compare to runes and signals?
Hooks are tied to render: React calls your function again and hooks return the latest values in call order. That is why hooks have rules (top level only, same order) and why you list dependency arrays. Runes are signals: reading `price` inside `$derived` or `$effect` subscribes to it automatically, and writing it notifies only those readers. No dependency arrays, no stale closures, and the script runs once.

### How do bundle sizes compare?
React's baseline is react plus react-dom, roughly in the 40 to 50 KB gzipped range before your own code. Svelte has a much smaller runtime, so small and medium apps ship far less JS. The gap narrows in very large apps because Svelte's compiled output grows per component, but in practice Svelte apps are usually smaller.

### How does re-rendering differ, and what does that mean for performance?
In React, `setState` re-runs that component and, by default, all its children. To avoid wasted work you use `React.memo`, `useMemo`, `useCallback`, or now the React Compiler. In Svelte there is no component re-render, only targeted DOM updates, so most of that optimisation work is not needed. React's advantage is that its model is very predictable and its concurrent features (transitions, Suspense) can keep the UI responsive during big updates.

### What about the ecosystem?
React has the largest ecosystem: TanStack, React Hook Form, Radix, MUI, React Native, Next.js, and many developers to hire. Svelte has fewer ready-made libraries, but it works well with plain JS libraries because there is no virtual DOM to fight, and SvelteKit covers routing, SSR and data loading.

### "You know Svelte. Can you work in React?"
Sample answer: "Yes. I have used React with hooks before, and the core ideas are the same in both: components, props, one-way data flow, derived state, effects with cleanup, and composition. The main things I keep in mind in React are the re-render model, so I keep state close to where it is used and memoize only when profiling shows a need; dependency arrays and stale closures in effects; and that React prefers controlled inputs over `bind:value`. Svelte actually made me careful about derived state and effects, because runes push you to derive values instead of syncing them with effects, and that is exactly what React's docs recommend too. I would be productive in React quickly, and I can bring Svelte's habits, like small components and minimal effects, to a React codebase."

### When would React be a worse choice than Svelte?
When a screen updates hundreds of values many times a second, like an order book. React re-renders and diffs each time unless you carefully memoize or move updates outside React. Svelte updates only the changed text nodes. That said, a well-built React app with memoized rows and throttled updates handles it fine.

## Common mistakes
- Saying "virtual DOM is slow". It is fast enough for most apps; the point is that it is extra work Svelte avoids.
- Saying Svelte has "no runtime". Svelte 5 has a small runtime for signals; it is just much smaller.
- Copying Svelte habits into React: mutating state objects directly (`state.price = 5`) does nothing in React; you must call the setter with a new object.
- Using `useEffect` to sync derived state in React. Compute it during render instead, like `$derived`.

## Resources
- [react.dev: Thinking in React](https://react.dev/learn/thinking-in-react) - React's mental model
- [react.dev: You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect) - derive instead of sync
- [svelte.dev: What are runes?](https://svelte.dev/docs/svelte/what-are-runes) - Svelte 5 signals model
- [svelte.dev: $state](https://svelte.dev/docs/svelte/$state) - fine-grained reactivity details
