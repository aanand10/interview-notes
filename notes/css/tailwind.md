# Tailwind

> **In one line:** Tailwind is a utility-first CSS framework - you style with small single-purpose classes like `flex p-4 text-sm` right in the markup, the values come from a shared design-token theme, and the build only outputs the classes you actually used.

## Key points
- **Utility-first:** one class = one small style (`p-4` = `padding: 1rem`). You compose them instead of writing custom class names and CSS files.
- **Design tokens** (colours, spacing, fonts, breakpoints) live in one theme. In **Tailwind v4** you define them in CSS with `@theme`; in **v3** in `tailwind.config.js`. Every token becomes utilities **and** a CSS variable (v4).
- **Unused CSS is removed:** Tailwind scans your source files for class names and generates only those. v2 called this `purge`, v3 uses the `content` array, v4 detects sources automatically. Result: a small CSS file.
- **`@apply`** copies utilities into your own CSS class. Useful in small doses, but overusing it brings back the problems Tailwind was meant to fix.
- **Dark mode:** the `dark:` variant. By default it follows the OS (`prefers-color-scheme`); you can switch it to a class or data attribute for a manual toggle.

## Example
Tailwind v4 setup with Svelte (SvelteKit + Vite).

```css
/* src/app.css */
@import "tailwindcss";

/* Design tokens -> generates bg-brand, text-up, text-down, font-display ... */
@theme {
  --color-brand: oklch(0.55 0.2 260);
  --color-up: #16a34a;     /* price up */
  --color-down: #dc2626;   /* price down */
  --font-display: "Inter", sans-serif;
}

/* Manual dark mode toggle: dark: applies when .dark is on <html> */
@custom-variant dark (&:where(.dark, .dark *));

/* @apply for a repeated pattern that is not a component yet */
.btn {
  @apply rounded-md px-4 py-2 font-medium focus-visible:outline-2;
}
```

```svelte
<!-- PriceChip.svelte -->
<script lang="ts">
  let { symbol, change }: { symbol: string; change: number } = $props();
  let up = $derived(change >= 0);
</script>

<span
  class="inline-flex items-center gap-1 rounded px-2 py-1 text-sm
         bg-white text-gray-900 dark:bg-gray-900 dark:text-gray-100"
>
  {symbol}
  <!-- Write FULL class names, never build them like `text-${color}` -->
  <span class={up ? 'text-up' : 'text-down'}>{change.toFixed(2)}%</span>
</span>
```

```js
// Tailwind v3 equivalent (tailwind.config.js)
export default {
  content: ['./src/**/*.{html,js,ts,svelte}'], // files scanned for class names
  darkMode: 'class',                            // 'selector' in v3.4.1+
  theme: { extend: { colors: { up: '#16a34a', down: '#dc2626' } } },
};
```

## When to use it
- Product teams with a design system: tokens keep colours and spacing consistent across the order form, watchlist and charts.
- Svelte components already keep markup, logic and style in one file, so utilities fit well there.
- Less useful for heavily custom, art-directed pages, or when the team does not want long class lists.

## Likely questions
### What are the pros and cons of utility-first CSS?
Pros: no naming things, no dead CSS (deleting markup deletes its styles), consistent design tokens, tiny final CSS, and fast to build UI. Cons: long class strings that are harder to read, a learning curve for class names, and markup is tied to styling. I handle the cons by extracting **components** (a `Button.svelte`) rather than CSS classes.

### How does Tailwind remove unused CSS?
It scans source files as plain text for anything that looks like a class name, and generates CSS only for those. That is why you must write complete class names. `text-${color}-500` will not be found; use a map like `{ up: 'text-green-600', down: 'text-red-600' }` instead.

### When would you use `@apply`?
For small, repeated patterns in places where you cannot add classes easily, like styling Markdown output or third-party HTML. I avoid using it to rebuild BEM-style classes everywhere, because then you have custom CSS again plus Tailwind. In Svelte `<style>` blocks with v4, you need `@reference "../app.css";` before `@apply` can see your theme.

### How do you set up design tokens and dark mode?
In v4 I put tokens in `@theme` (they become utilities and CSS variables), and in v3 under `theme.extend` in the config. For dark mode, `dark:` follows the OS by default. For a user toggle, I change the dark variant to a `.dark` class, add or remove that class on `<html>`, and save the choice in `localStorage`. A small inline script in `<head>` sets the class before paint to avoid a flash of the wrong theme.

### How does Tailwind relate to PostCSS?
Tailwind v3 is a PostCSS plugin. Tailwind v4 has its own engine and ships a Vite plugin (`@tailwindcss/vite`) and a PostCSS plugin (`@tailwindcss/postcss`). v4 also handles vendor prefixes itself, so you do not need autoprefixer.

## Common mistakes
- Building class names dynamically so they get removed from the build.
- Overusing `@apply` and arbitrary values (`w-[317px]`) instead of tokens.
- Conflicting classes like `p-2 p-4` on the same element: the winner is decided by order in the generated CSS, not in the class attribute.

## Resources
- [Tailwind: Theme variables](https://tailwindcss.com/docs/theme) - design tokens with `@theme`
- [Tailwind: Dark mode](https://tailwindcss.com/docs/dark-mode) - OS vs manual toggle
- [Tailwind: Detecting classes in source files](https://tailwindcss.com/docs/detecting-classes-in-source-files) - how unused CSS is avoided
- [Tailwind: Functions and directives](https://tailwindcss.com/docs/functions-and-directives) - `@apply`, `@reference`, `@custom-variant`
