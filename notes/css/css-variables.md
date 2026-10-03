# CSS variables

> **In one line:** CSS variables (custom properties) like `--brand: blue` are real CSS values that live in the browser, cascade and inherit like any other property, and can be changed at runtime with JavaScript or a class - which makes them perfect for theming and dark mode, unlike Sass variables that disappear at build time.

## Key points
- **Syntax:** declare with `--name: value;`, read with `var(--name, fallback)`. Names are case-sensitive. See [MDN: Using CSS custom properties](https://developer.mozilla.org/en-US/docs/Web/CSS/Using_CSS_custom_properties).
- **They cascade and inherit.** Define them on `:root` for global tokens, override them on any element to change a subtree (a "scoped theme").
- **Runtime:** change them with a class, a media query, or JS (`el.style.setProperty('--x', '10px')`). Everything that uses them updates instantly.
- **Sass variables (`$brand`)** are replaced with fixed values at compile time. They cannot react to media queries, classes or JS, but they can be used in places CSS variables cannot, like selectors and media query conditions.
- **[`@property`](https://developer.mozilla.org/en-US/docs/Web/CSS/@property)** lets you give a variable a type (like `<color>`), which makes it animatable.

## Example
Light/dark theme with a toggle, plus a per-component override.

```css
:root {
  --bg: #ffffff;
  --text: #111827;
  --up: #16a34a;
  --down: #dc2626;
  --radius: 8px;
  color-scheme: light dark; /* form controls and scrollbars follow the theme */
}

/* 1. Follow the OS setting */
@media (prefers-color-scheme: dark) {
  :root:not([data-theme="light"]) {
    --bg: #0b0f19;
    --text: #e5e7eb;
  }
}

/* 2. Manual override from a toggle */
:root[data-theme="dark"] { --bg: #0b0f19; --text: #e5e7eb; }

body { background: var(--bg); color: var(--text); }

.change.up   { color: var(--up); }
.change.down { color: var(--down); }

/* Scoped override: this card uses a bigger radius, children inherit it */
.promo-card { --radius: 16px; border-radius: var(--radius); }

/* Fallback when a variable is missing */
.badge { padding: var(--badge-pad, 4px 8px); }
```

```svelte
<!-- ThemeToggle.svelte -->
<script lang="ts">
  let theme = $state<'light' | 'dark'>('light');

  function toggle() {
    theme = theme === 'light' ? 'dark' : 'light';
    document.documentElement.dataset.theme = theme; // CSS reacts at once
    localStorage.setItem('theme', theme);
  }

  let progress = $state(40);
</script>

<button onclick={toggle}>Theme: {theme}</button>

<!-- Svelte's style: directive sets a CSS variable from state -->
<div class="bar" style:--progress="{progress}%"></div>

<style>
  .bar { width: var(--progress); height: 4px; background: var(--up); }
</style>
```

## When to use it
- **Theming and dark mode** for a trading app: one set of semantic tokens (`--bg`, `--surface`, `--up`, `--down`), swapped per theme.
- **White-label or per-user settings:** load brand colours from an API and set them on `:root` at runtime.
- **Component APIs:** Svelte lets a parent pass CSS variables to a child: `<Slider --track-color="red" />`.

## Likely questions
### CSS variables vs Sass variables?
Sass variables are compiled away; the browser only sees the final value, so you cannot change them at runtime or per element. CSS variables exist in the browser, cascade, inherit, respond to media queries and classes, and can be updated from JS. I use Sass variables only for build-time things like breakpoint numbers in media queries; for themes I use CSS variables.

### How would you implement dark mode?
Define semantic tokens on `:root`, then override them in `@media (prefers-color-scheme: dark)` and under a `[data-theme="dark"]` attribute for a manual toggle. Save the user's choice in `localStorage` and set the attribute with a tiny inline script in `<head>` before first paint, so there is no flash of the wrong theme. Also set `color-scheme` so native inputs match. Newer browsers support [`light-dark()`](https://developer.mozilla.org/en-US/docs/Web/CSS/color_value/light-dark), e.g. `color: light-dark(black, white)`.

### How do you change a CSS variable from JavaScript?
`document.documentElement.style.setProperty('--brand', '#2563eb')` to set it, and `getComputedStyle(el).getPropertyValue('--brand')` to read it.

### Can you use a CSS variable in a media query?
No. `@media (min-width: var(--bp))` does not work, because media queries are not tied to an element. Use a fixed value, a Sass variable, or a container query.

## Common mistakes
- Forgetting `var()`: `color: --brand` is invalid.
- An invalid value does not fall back to the previous declaration; the property becomes "invalid at computed-value time" and uses its inherited or initial value.
- Transitions on plain variables do not animate smoothly; register them with `@property` first.

## Resources
- [MDN: Using CSS custom properties](https://developer.mozilla.org/en-US/docs/Web/CSS/Using_CSS_custom_properties) - full guide
- [MDN: prefers-color-scheme](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-color-scheme) - follow the OS theme
- [web.dev: prefers-color-scheme](https://web.dev/articles/prefers-color-scheme) - practical dark mode guide
- [Svelte docs: style directive](https://svelte.dev/docs/svelte/style) - setting CSS variables from state
