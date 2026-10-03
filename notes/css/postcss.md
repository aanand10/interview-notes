# PostCSS

> **In one line:** PostCSS is a tool that turns CSS into a tree (an AST), runs it through a chain of JavaScript plugins that each change something, and prints CSS back out - it does nothing by itself, the plugins do all the work.

## Key points
- **Plugin pipeline:** parse CSS -> AST (a tree of rules, selectors and declarations) -> plugin 1 -> plugin 2 -> ... -> output CSS. Plugins run in the order you list them.
- It is **not a preprocessor** like Sass with its own language. You write normal (or future) CSS, and plugins transform it. It can be used together with Sass.
- **[Autoprefixer](https://github.com/postcss/autoprefixer)** is the most famous plugin. It adds vendor prefixes like `-webkit-` only where your target browsers need them, based on [Can I Use](https://caniuse.com) data and your **browserslist** config.
- Other common plugins: `postcss-preset-env` (use modern CSS today), `postcss-nesting`, `cssnano` (minify), `postcss-import` (inline `@import`).
- Vite, SvelteKit, Next.js and webpack all run PostCSS automatically if a `postcss.config.js` exists.

## Example
```js
// postcss.config.js (Tailwind v3 era setup)
export default {
  plugins: {
    tailwindcss: {},   // 1. generate utilities from @tailwind directives
    autoprefixer: {},  // 2. add vendor prefixes for target browsers
  },
};
```

```js
// postcss.config.js (Tailwind v4 if you use PostCSS instead of the Vite plugin)
export default {
  plugins: {
    '@tailwindcss/postcss': {}, // v4 handles imports, nesting and prefixes itself
  },
};
```

```text
# .browserslistrc - which browsers Autoprefixer targets
> 0.5%
last 2 versions
not dead
```

```css
/* Input */
.chart { user-select: none; }

/* Output after Autoprefixer (for older Safari) */
.chart { -webkit-user-select: none; user-select: none; }
```

A tiny custom plugin, to show how the pipeline works:

```js
// Turns every "color: brand" into a real colour value
const brandColor = () => ({
  postcssPlugin: 'brand-color',
  Declaration: {
    color(decl) {
      if (decl.value === 'brand') decl.value = '#2563eb';
    },
  },
});
brandColor.postcss = true;
export default brandColor;
```

## When to use it
- Almost every modern frontend build already runs it under the hood (Vite supports it out of the box).
- Add it when you need prefixes for older Safari or embedded webviews, minification, or a design-token transform step.
- With Svelte, component `<style>` blocks also go through PostCSS when `vitePreprocess` is used.

## Likely questions
### What is PostCSS?
It is a CSS transformer. It parses CSS into an abstract syntax tree, passes that tree through a list of plugins written in JavaScript, then turns it back into CSS. PostCSS itself changes nothing; each plugin does one job, like adding prefixes, minifying or generating utilities.

### What does Autoprefixer do?
It reads your browserslist targets and Can I Use data, and adds only the vendor prefixes those browsers need. So you write standard CSS once, and you do not ship prefixes for browsers you no longer support.

### How does Tailwind use PostCSS?
In v3, Tailwind is a PostCSS plugin. It finds `@tailwind base/components/utilities` in your CSS, scans your files for class names, and replaces the directives with the generated CSS. Autoprefixer usually runs after it. In v4, Tailwind has its own engine built on Lightning CSS, recommends the Vite plugin, and offers `@tailwindcss/postcss` for PostCSS setups. It does prefixing itself, so Autoprefixer is not needed.

### PostCSS vs Sass?
Sass is a language with variables, mixins and functions that compiles to CSS. PostCSS is a platform for transforms on regular CSS. Many teams now use native CSS variables and nesting plus PostCSS, and drop Sass.

## Resources
- [PostCSS](https://postcss.org/) - official site and plugin list
- [Autoprefixer](https://github.com/postcss/autoprefixer) - how prefixing and browserslist work
- [Tailwind: Using PostCSS](https://tailwindcss.com/docs/installation/using-postcss) - Tailwind v4 PostCSS setup
