# Solid vs Svelte vs React

> **In one line:** React re-runs whole components and diffs a virtual DOM, while Solid and Svelte 5 both use signals to update only the exact DOM nodes that changed; Solid does it with runtime functions you call (`count()`), Svelte does it with a compiler and runes (`$state`).

How I frame it: I use Svelte 5 daily. I have not shipped Solid in production, but its signal model maps almost one-to-one to runes, so I can read and reason about Solid code quickly.

## Key points
- **React:** state change -> component function re-runs -> new virtual DOM -> diff -> patch DOM. Needs `useMemo`, `useCallback`, `React.memo` and dependency arrays to avoid extra work (React Compiler now automates some of this).
- **Solid:** component runs once. Signals are read with getter calls. JSX compiles to real DOM plus small effects. No virtual DOM.
- **Svelte 5:** component script runs once. Runes (`$state`, `$derived`, `$effect`) are compiled into signal code. You read plain variables (`count`), no getter call. No virtual DOM.
- **Mapping:**

| Idea | Solid | Svelte 5 | React |
|---|---|---|---|
| State | `createSignal` | `$state` | `useState` |
| Derived | `createMemo` | `$derived` | `useMemo` |
| Side effect | `createEffect` | `$effect` | `useEffect` |
| Nested object state | `createStore` | `$state` (deep proxy) | `useState` + immutable copies |
| Props | `props.x` (do not destructure) | `let { x } = $props()` | function args |
| Templates | JSX + `<Show>`, `<For>` | `{#if}`, `{#each}` | JSX + `&&`, `.map` |

- **Dependency tracking:** Solid and Svelte track automatically at runtime. React relies on you listing dependencies.

## Example
The same counter with a derived value and an effect.

```tsx
// Solid
const [count, setCount] = createSignal(0);
const double = createMemo(() => count() * 2);
createEffect(() => console.log(double()));
<button onClick={() => setCount(count() + 1)}>{count()} / {double()}</button>
```

```svelte
<!-- Svelte 5 -->
<script>
  let count = $state(0);
  let double = $derived(count * 2);
  $effect(() => console.log(double));
</script>
<button onclick={() => count++}>{count} / {double}</button>
```

```tsx
// React
const [count, setCount] = useState(0);
const double = useMemo(() => count * 2, [count]);
useEffect(() => console.log(double), [double]);
<button onClick={() => setCount(c => c + 1)}>{count} / {double}</button>
```

## When to use it
- **Svelte/SvelteKit:** least code, great DX, full-stack framework built in. Good for a trading UI with many live values.
- **Solid:** similar speed, JSX feel for React developers, smaller ecosystem.
- **React:** biggest ecosystem and hiring pool, React Native, more manual performance tuning for high-frequency updates.

## Likely questions
### Compare the reactivity models.
React is coarse-grained: the component is the unit of update and it re-renders and diffs. Solid and Svelte 5 are fine-grained: the signal is the unit, and only effects reading that signal re-run. For a screen with hundreds of ticking prices, fine-grained means far less work per tick.

### How do Solid primitives map to Svelte runes?
Signal is `$state`, memo is `$derived`, effect is `$effect`, store is a deep `$state` object. The big syntax difference: Solid reads are `count()` calls, Svelte reads are plain `count` because the compiler adds the tracking.

### What does the compiler buy Svelte?
Cleaner syntax (no getters or setters), deep reactive objects with normal mutation (`todo.done = true`), and props you can destructure safely. The trade-off is the code you write is not exactly the code that runs.

### Any difference between `createMemo` and `$derived`?
Both are cached. `createMemo` computes right away when created. `$derived` is lazy: it computes when first read, then caches until a dependency changes.

## Common mistakes
- Saying Svelte has "no runtime". Svelte 5 has a small signals runtime; it just has no virtual DOM.
- Saying Solid has no compiler. It compiles JSX to DOM code, but reactivity is runtime signals.

## Resources
- [Solid: Intro to reactivity](https://docs.solidjs.com/concepts/intro-to-reactivity) - signals and tracking
- [Svelte: $derived](https://svelte.dev/docs/svelte/$derived) - runes side of the mapping
- [Svelte: $effect](https://svelte.dev/docs/svelte/$effect) - when effects run
- [Svelte blog: Introducing runes](https://svelte.dev/blog/runes) - why Svelte moved to signals
