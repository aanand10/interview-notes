# Components run once (SolidJS)

> **In one line:** A Solid component function runs only once, to set things up and build the DOM; after that, only the small reactive pieces inside it re-run, which is why destructuring props breaks reactivity.

How I frame it: I have not shipped Solid in production, but this matches Svelte 5, where the `<script>` block also runs once per component and runes do the updating.

## Key points
- **Setup, not render.** In React, the component function re-runs on every state change. In Solid, it runs once. The JSX is compiled into DOM nodes plus effects that update them.
- **Reactivity lives in getters.** Only code inside JSX expressions, `createMemo`, `createEffect` (tracking scopes) re-runs. A `console.log` in the component body prints once.
- **Props are reactive getters.** `props.price` is a getter under the hood. Reading it in JSX tracks it. Reading it once at the top and storing it in a variable does not ([props docs](https://docs.solidjs.com/concepts/components/props)).
- **Destructuring reads once.** `function Row({ price })` reads `props.price` at setup and stores a plain value. Later changes from the parent never reach it.
- **Fixes:** use `props.price` directly, `splitProps` to split props safely, `mergeProps` for defaults, or wrap in a function: `const price = () => props.price`.

## Example
```tsx
import { splitProps, mergeProps } from 'solid-js';

// BROKEN: price is read once, never updates
function PriceBad({ price }: { price: number }) {
  return <span>{price}</span>;
}

// WORKS: read through props each time
function PriceGood(props: { price: number }) {
  console.log('runs once');            // prints only one time
  return <span>{props.price}</span>;   // this expression re-runs when price changes
}

// WORKS: defaults + splitting without losing reactivity
function PriceTag(rawProps: { price: number; currency?: string; class?: string }) {
  const props = mergeProps({ currency: 'USD' }, rawProps);
  const [local, rest] = splitProps(props, ['price', 'currency']);
  return <span {...rest}>{local.price} {local.currency}</span>;
}

// BROKEN: early return runs once, so it never switches
function UserBad(props: { user?: { name: string } }) {
  if (!props.user) return <p>Loading</p>;
  return <p>{props.user.name}</p>;
}
// Fix: use <Show when={props.user} fallback={<p>Loading</p>}>...</Show>
```

## When to use it
Any time you pass live data down, like a ticking price into a row component. In a watchlist of 500 rows, each row component runs once, and a price tick only updates one text node. That is the performance win.

## Likely questions
### Why does a Solid component run only once?
Because Solid's reactivity is at the signal level, not the component level. The component is just a setup function that creates DOM and connects signals to it. When a signal changes, only the effects reading it re-run. React re-runs the whole function and diffs a virtual DOM instead.

### Why does destructuring props break reactivity?
Props are an object with getters. Destructuring calls the getter once during setup and copies the value into a local variable. Since the component never runs again, that variable never updates. Keep `props.x`, or use `splitProps`.

### How does that compare to Svelte 5?
Very similar idea. The Svelte `<script>` runs once. In Svelte 5, `let { price } = $props()` destructuring is fine because the compiler turns those into reactive reads, so this specific trap is a Solid thing.

### What about early returns and conditions?
An `if` in the body runs once, so it will not switch later. Put conditions in JSX with `<Show>` or a ternary inside `{}`.

## Common mistakes
- Destructuring props in the parameter list.
- `const p = props.price` at the top and using `p` in JSX.
- Early `return` for loading states.

## Resources
- [Solid: Props](https://docs.solidjs.com/concepts/components/props) - why not to destructure, `splitProps`, `mergeProps`
- [Solid: Intro to reactivity](https://docs.solidjs.com/concepts/intro-to-reactivity) - tracking scopes explained
- [Svelte: $props](https://svelte.dev/docs/svelte/$props) - Svelte 5 comparison
