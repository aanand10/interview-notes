# Svelte in a checkout product (what I built, why it fits, AI tools)

> **In one line:** "Svelte compiles components into small, direct DOM updates with very little runtime, which fits a checkout that must load fast on low-end phones inside other people's sites, and I use AI tools to go faster while still reviewing, testing and owning every line."

## What the interviewer is really checking

- That your Svelte experience is **real and current** (Svelte 5 runes, not only Svelte 3/4 syntax).
- That you can explain **why** a framework fits a product, with trade-offs.
- That you use **AI tools responsibly**: faster, but you understand and test the code.
- The team is Svelte-first, so expect follow-ups on runes, stores vs runes, and migration.

## Key points

- **Compiler, not virtual DOM:** Svelte turns components into JavaScript that updates the exact DOM nodes that changed. Less runtime code and less work per update. See [Svelte overview](https://svelte.dev/docs/svelte/overview).
- **Small bundles** matter for checkout: the page is often loaded inside a merchant site or WebView on slow networks, and every 100 KB costs conversions.
- **Runes (Svelte 5):** `$state`, `$derived`, `$effect`, `$props` make reactivity explicit and work in `.svelte.ts` files too. See [What are runes?](https://svelte.dev/docs/svelte/what-are-runes).
- **Trade-offs:** smaller ecosystem than React, fewer ready-made libraries, migration from Svelte 4 to 5 takes effort.
- **AI tools:** good for boilerplate, tests, refactors and explaining unknown code. I still review every diff, run tests, and never paste secrets or customer data into a tool.

## Example

A checkout summary with runes:

```svelte
<script lang="ts">
  type Item = { name: string; pricePaise: number; qty: number };
  let { items, couponPercent = 0 }: { items: Item[]; couponPercent?: number } = $props();

  // $derived recomputes only when items or couponPercent change
  const subtotal = $derived(items.reduce((sum, i) => sum + i.pricePaise * i.qty, 0));
  const discount = $derived(Math.round((subtotal * couponPercent) / 100));
  const total = $derived(subtotal - discount);

  const fmt = (paise: number) =>
    new Intl.NumberFormat('en-IN', { style: 'currency', currency: 'INR' }).format(paise / 100);
</script>

<dl>
  <dt>Subtotal</dt><dd>{fmt(subtotal)}</dd>
  {#if discount > 0}<dt>Discount</dt><dd>-{fmt(discount)}</dd>{/if}
  <dt>Total</dt><dd>{fmt(total)}</dd>
</dl>
```

## When to use it

Svelte shines for performance-sensitive, embeddable or mobile-web UIs: checkout, a trading watchlist with live prices, PWAs and WebViews on low-end Android devices.

## Answer framework

For "what you built": problem, constraints, options, decision, implementation, impact, improve. For "why Svelte": 3 reasons tied to the product, 1 honest trade-off. For "AI tools": how you use them, how you verify, one example where you caught a wrong suggestion.

## Fill-in template

```text
What I built with Svelte: [fill in: 1-2 features, described generically]
Svelte version and patterns: [fill in: Svelte 4 / 5, runes, stores, SvelteKit or not]
Why Svelte fits checkout: [fill in: bundle size, speed on low-end devices, embed/WebView, simple state]
Trade-off we felt: [fill in: ecosystem gap, migration cost, hiring]
AI tools I use: [fill in: tool names] for [fill in: tasks]
How I stay the owner: [fill in: review, tests, reading docs, no sensitive data]
Example where AI was wrong and I caught it: [fill in]
```

## Example answer (generic, for illustration only)

> EXAMPLE, not a real story.

"I built the payment-method selection and saved-cards screens of a checkout in Svelte. Svelte fits because the checkout is embedded in many merchant pages and opened on low-end phones, so a small bundle and fast updates directly help conversion. Its compiled output also keeps form interactions snappy. The trade-off is fewer ready-made libraries, so we wrote some components ourselves. I use an AI assistant to scaffold components and write first drafts of tests. Once it suggested an `$effect` to sync two pieces of state; I replaced it with `$derived`, because effects for derived values cause extra updates and bugs. I read every line, run the tests, and treat AI code like a junior teammate's PR."

## Likely questions

### What have you built with Svelte?
Name 1 to 2 features generically, the version (4 or 5), and one interesting technical detail (state model, a tricky form, performance). Be ready to write a small runes component live.

### Why does Svelte suit a checkout?
Small runtime and bundle, which matters on slow networks and inside WebViews and iframes. Fine-grained updates without a virtual DOM diff, so forms and timers feel instant. Simple component model, so new engineers ship quickly. Mention one trade-off to show balance.

### Svelte 4 vs Svelte 5: what changed?
Svelte 5 replaced `let` plus `$:` reactivity with runes: `$state`, `$derived`, `$effect`, `$props`. Slots became snippets, `on:click` became `onclick`, and reactivity works outside components in `.svelte.js/.ts` files. Stores still work but runes cover most uses. See the [v5 migration guide](https://svelte.dev/docs/svelte/v5-migration-guide).

### When do you use $effect vs $derived?
`$derived` for values computed from other state (most cases). `$effect` only for side effects that touch the outside world: analytics, a WebSocket subscription, a canvas draw, `localStorage`. Using `$effect` to set state is a common smell. See [$effect](https://svelte.dev/docs/svelte/$effect).

### How do you use AI tools while still owning the code?
Use them for boilerplate, tests, refactors and explanations. Then review the diff like any PR, check it against official docs (AI often writes Svelte 4 syntax), run tests and lint, and never share secrets or customer data. If I cannot explain a line, it does not get merged.

### Would you pick Svelte for everything?
No. For a team deep in React with many React-only libraries, the switch cost may not pay off. The choice depends on performance needs, team skills and ecosystem.

## Common mistakes

- Writing Svelte 4 syntax (`export let`, `$:`, `on:click`) when asked about Svelte 5.
- Saying "Svelte is faster" with no reason or trade-off.
- Using `$effect` to compute derived values.
- Sounding like AI writes your code and you just merge it.
- Naming internal tools or confidential product details.

## Resources

- [Svelte docs: Overview](https://svelte.dev/docs/svelte/overview) - what Svelte is and how it works
- [Svelte docs: What are runes?](https://svelte.dev/docs/svelte/what-are-runes) - the Svelte 5 reactivity model
- [Svelte 5 migration guide](https://svelte.dev/docs/svelte/v5-migration-guide) - every 4 to 5 change in one page
- [Svelte docs: $effect](https://svelte.dev/docs/svelte/$effect) - when not to use effects
