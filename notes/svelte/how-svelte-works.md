# How Svelte works

> **In one line:** Svelte is a compiler: it turns my `.svelte` files into plain JavaScript that updates the exact DOM nodes that changed, so there is no virtual DOM and very little runtime code shipped to the browser.

## Key points
- **Compiler, not a big runtime.** React ships a library that runs in the browser and works out what changed at runtime. Svelte does most of that work at build time. The output is small JS that calls DOM APIs directly, plus a small shared runtime for things like signals and scheduling.
- **No virtual DOM.** A virtual DOM (VDOM) is a JS copy of the UI tree. React re-runs the component, builds a new VDOM tree and diffs it against the old one. Svelte skips this. The compiler already knows which parts of the template are dynamic, so it wires each one straight to its data.
- **Fine-grained updates (Svelte 5).** State made with `$state` is a *signal*: a value that knows who reads it. Each dynamic part of the template is wrapped in a tiny effect. When the signal changes, only those effects re-run. The component function itself runs only once.
- **Svelte 4 was different inside.** It tracked changes with `$$invalidate` and a "dirty" bitmask per component, then ran one update function `p(ctx, dirty)` for the whole component. Svelte 5 moved to signals, which is more precise and works outside components too.
- **Updates are batched.** Many state changes in one tick are grouped and flushed together in a microtask, so the DOM is touched once.

## Example
A tiny component and (trimmed) real output from the Svelte 5.57 compiler:

```svelte
<script>
  let price = $state(100);
</script>

<button onclick={() => price++}>Bump</button>
<p class="price">Price: {price}</p>

<style>
  .price { color: green; }
</style>
```

```js
// Compiled client output (svelte/compiler, generate: 'client')
import * as $ from 'svelte/internal/client';

// 1. Static HTML is created once from a template string (fast cloning)
var root = $.from_html(`<button>Bump</button> <p class="price svelte-4psngr"> </p>`, 1);

export default function Price($$anchor) {
  let price = $.state(100);                 // a signal
  var fragment = root();
  var button = $.first_child(fragment);
  var p = $.sibling(button, 2);
  var text = $.only_child(p);

  // 2. Only this text node is updated when `price` changes
  $.template_effect(() => $.set_text(text, `Price: ${$.get(price) ?? ''}`));

  // 3. Click handled through event delegation at the app root
  $.delegated('click', button, () => $.update(price));
  $.append($$anchor, fragment);
}
$.delegate(['click']);
```

So "how does an update reach the DOM?":
1. Click runs `$.update(price)`, which sets the signal.
2. The signal marks the effects that read it as dirty and schedules a flush (microtask).
3. On flush, the `template_effect` re-runs and calls `set_text`, which sets `text.nodeValue`. Nothing else runs.

## When to use it
- Live trading screens: a price ticker that updates many times per second. With signals, only the price text nodes change. No component re-render, no diff of the whole watchlist.
- Mobile web and low-end devices: small bundles and less JS work mean faster load and smoother scrolling.
- SvelteKit apps where you want SSR (server-side rendering) plus small client JS.

## Likely questions
### Is Svelte a framework or a compiler?
Both, in a way. Svelte the language is compiled by the Svelte compiler at build time, usually through Vite. The compiler emits plain JS plus imports from a small runtime (`svelte/internal/client`). So the "framework work" of tracking changes is decided at build time, and the runtime is small.

### Why doesn't Svelte need a virtual DOM?
A VDOM exists to answer "what changed?" at runtime by diffing two trees. Svelte answers that question at compile time: it sees `{price}` in the template and generates code that updates exactly that text node when `price` changes. Rich Harris wrote about this in [Virtual DOM is pure overhead](https://svelte.dev/blog/virtual-dom-is-pure-overhead). The VDOM is not slow, but the diffing is extra work Svelte can skip.

### Walk me through what happens when state changes.
`count++` compiles to a signal set. The signal knows which effects and deriveds read it (they subscribed automatically when they read it). Those are marked dirty. Svelte schedules a flush in a microtask. On flush, deriveds recompute lazily when read, and template effects update their DOM nodes. If many signals change in one event handler, they are batched into one flush.

### How is Svelte 5 reactivity different from Svelte 4 under the hood?
Svelte 4 used the compiler to find assignments (`count = ...`) and turned them into `$$invalidate(...)`, which set a bit in a dirty mask. Then the component's update function checked bits. It only worked inside `.svelte` files and only on assignment. Svelte 5 uses runtime signals created by runes, so reactivity is fine-grained, works with deep mutations through proxies, and works in `.svelte.js` / `.svelte.ts` files too.

### Svelte vs React: what are the trade-offs?
| | Svelte | React |
|---|---|---|
| Model | Compiler + signals, component runs once | Runtime + VDOM, component re-runs on every render |
| Updates | Only effects that read changed state | Re-render subtree, diff, commit (memo to limit) |
| Bundle | Small runtime | Larger runtime (react + react-dom) |
| Syntax | HTML-first templates, scoped CSS built in | JSX, CSS solution is your choice |
| Perf tuning | Rarely need `memo` / `useCallback` | Often need memoization |
| Ecosystem | Smaller: fewer libraries and devs | Huge ecosystem, hiring pool, React Native |
| Tooling | Needs the compiler (Vite plugin) | Works with many setups |

A fair answer: Svelte gives less code, faster updates and less tuning. React gives a bigger ecosystem and more hiring options. For a trading UI with lots of fast-changing numbers, fine-grained updates are a real win.

### Does Svelte have a runtime at all?
Yes, a small one. Signals, effect scheduling, transitions, hydration and event delegation live in `svelte/internal/client`. It is tree-shaken, so you only ship what you use. "No runtime" was never fully true; "no VDOM runtime" is accurate.

### What does the compiler do with `<style>`?
It scopes CSS by adding a hash class (like `svelte-4psngr`) to the elements and selectors of that component. It also removes unused selectors and warns about them. See [scoped styles](https://svelte.dev/docs/svelte/scoped-styles).

## Common mistakes
- Saying "Svelte has no runtime". Say "very small runtime, no virtual DOM".
- Thinking the component function re-runs like React. In Svelte 5 the `<script>` runs once per component instance. Only effects re-run.
- Saying Svelte 5 still uses the dirty bitmask. That was Svelte 3/4.
- Forgetting the compiler can only see one file. That is why exported `$state` that gets reassigned cannot be imported as a plain variable (see the `$state` note).

## Resources
- [Svelte docs: Overview](https://svelte.dev/docs/svelte/overview) - official intro to the compiler approach
- [Virtual DOM is pure overhead](https://svelte.dev/blog/virtual-dom-is-pure-overhead) - the classic argument against VDOM diffing
- [Introducing runes](https://svelte.dev/blog/runes) - why Svelte 5 moved to signals
- [Svelte playground](https://svelte.dev/playground) - open the "JS Output" tab to see compiled code
