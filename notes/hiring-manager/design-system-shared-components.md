# Design system / shared components

> **In one line:** "We kept rebuilding the same buttons, inputs and modals with small differences, so I built a shared component library with design tokens, clear APIs and docs, drove adoption team by team, and measured the time saved on new screens."

## What the interviewer is really checking

- **Why:** did you solve a real problem (inconsistency, duplicated effort, accessibility bugs) or build it for fun?
- **API design:** can you design props that are simple, flexible and hard to misuse?
- **Adoption:** a library nobody uses is a failure. How did you get people to switch?
- **Measurement:** how did you get a number like "[fill in: X]% dev-time saving", and is it honest?
- **Maintenance thinking:** versioning, breaking changes, ownership, contributions. The job description mentions shared component libraries, so expect depth here.

## Key points

- **Design tokens** are named values (colours, spacing, font sizes, radii) stored once and used everywhere, usually as [CSS custom properties](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_cascading_variables/Using_CSS_custom_properties). Themes (dark mode) just swap token values.
- **Primitives first:** Button, Input, Select, Checkbox, Modal, Toast, Table. These cover most screens.
- **Accessibility built in:** keyboard support, focus management, ARIA, contrast. Every team gets it for free. See [MDN: Accessibility](https://developer.mozilla.org/en-US/docs/Web/Accessibility).
- **Docs and playground:** live examples, props table, do and don't. Without docs, adoption stalls.
- **Versioning:** semantic versioning, a changelog, deprecation warnings before breaking changes, codemods if possible.

## Example

A Svelte 5 Button that is typed, accessible and passes native attributes through:

```svelte
<!-- Button.svelte -->
<script lang="ts">
  import type { Snippet } from 'svelte';
  import type { HTMLButtonAttributes } from 'svelte/elements';

  type Props = HTMLButtonAttributes & {
    variant?: 'primary' | 'secondary' | 'danger';
    size?: 'sm' | 'md';
    loading?: boolean;
    children: Snippet;
  };

  let { variant = 'primary', size = 'md', loading = false, disabled, children, ...rest }: Props = $props();
</script>

<button class="btn {variant} {size}" disabled={disabled || loading} aria-busy={loading} {...rest}>
  {#if loading}<span class="spinner" aria-hidden="true"></span>{/if}
  {@render children()}
</button>

<style>
  .btn { padding: var(--space-2) var(--space-4); border-radius: var(--radius-md); font: inherit; }
  .primary { background: var(--color-brand); color: var(--color-on-brand); }
  .danger { background: var(--color-sell); color: var(--color-on-brand); }
  .sm { padding: var(--space-1) var(--space-3); }
</style>
```

```svelte
<!-- usage -->
<Button variant="danger" loading={placing} onclick={placeSellOrder}>Sell</Button>
```

## When to use it

When two or more teams or products build similar UI, when the brand needs consistency (a trading app with Buy green and Sell red everywhere), or when accessibility bugs keep repeating.

## Answer framework

1. **Problem:** duplication, inconsistency, accessibility bugs, slow new screens. Give evidence (for example "we had 6 button styles").
2. **Constraints:** teams on different stacks or versions, no dedicated design-system team, deadlines.
3. **Options:** adopt an open-source library, wrap one, or build our own. Pros and cons of each.
4. **Decision and why.**
5. **Implementation:** tokens, component list, API rules, docs, tests, release process.
6. **Adoption:** champions, migration guide, pairing, start with new screens, lint rules.
7. **Impact and how measured.**
8. **What I would change.**

## Fill-in template

```text
Why we built it: [fill in: evidence of the pain]
Options considered: [fill in: open-source / wrap / build] -> decision: [fill in] because [fill in]
What I built: [fill in: tokens, N components, docs site, test setup]
API rules we followed: [fill in: e.g. pass-through native attributes, controlled + uncontrolled, snippets for slots]
Adoption: [fill in: how many teams/screens, how you convinced them]
The [fill in: X]% dev-time saving was measured by: [fill in: method, see below]
What I would change: [fill in]
```

## Example answer (generic, for illustration only)

> EXAMPLE, not a real story.

"Three teams were building forms with their own inputs and modals. Design reviews kept finding inconsistent spacing and modals that trapped no focus. I proposed a small shared library: tokens as CSS variables plus 12 primitives. We considered an open-source kit, but theming it to our brand and bundle limits was hard, so we built thin, accessible components ourselves. I wrote the first components, a docs page with live examples, and component tests. For adoption, I paired with one engineer from each team to migrate one screen, and we made new screens use the library by default. To measure time saved, we compared story points and actual days for similar form screens before and after, over two sprints, and saw about [X]% less time. If I did it again, I would version it from day one and add visual regression tests earlier."

## Likely questions

### Why did you build it instead of using an existing library?
Give a real reason: brand theming, bundle size, framework fit, accessibility quality, or control over the roadmap. Acknowledge the cost: you now own maintenance. "Wrap an existing headless library" is often a good middle answer.

### How did you get teams to adopt it?
Make the right thing the easy thing: good docs, copy-paste examples, a migration guide. Find a champion in each team, migrate one screen together, and use it by default for new work. Collect feedback and fix pain points fast so people trust it.

### How did you measure the 30% dev-time saving?
Be honest about the method: compare similar tickets before and after (estimates vs actuals), time to build a standard screen, or number of lines or files per new form. Say it is an estimate and mention the other wins: fewer UI bugs, fewer accessibility issues, faster design reviews.

### How do you design a good component API?
Small, predictable props; sensible defaults; pass native attributes through (`...rest`); use [snippets](https://svelte.dev/docs/svelte/snippet) for flexible content instead of many string props; support both controlled and uncontrolled use where it makes sense; type everything with TypeScript.

### How do you handle breaking changes?
Semantic versioning, a changelog, deprecation warnings for at least one release, a migration guide, and ideally a codemod. Coordinate releases with consuming teams.

### What would you change?
Good answers: start with tokens and docs earlier, add visual regression tests, set up a contribution model so other teams can add components, or measure adoption automatically (count imports).

## Common mistakes

- Building 40 components before anyone uses one.
- Too many props ("prop explosion") instead of composition.
- Not handling focus and keyboard in Modal, Menu, Select.
- Breaking changes without notice, which kills trust.
- A big number with no method behind it.

## Resources

- [MDN: Using CSS custom properties](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_cascading_variables/Using_CSS_custom_properties) - the base of design tokens and theming
- [Svelte docs: Snippets](https://svelte.dev/docs/svelte/snippet) - flexible content in Svelte 5 components
- [Svelte docs: $props](https://svelte.dev/docs/svelte/$props) - typed props and rest props
- [MDN: Accessibility](https://developer.mozilla.org/en-US/docs/Web/Accessibility) - what every shared component must get right
