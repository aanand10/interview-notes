# Modules

> **In one line:** ES modules (`import`/`export`) are the standard: they are static, async-friendly and tree-shakable, while CommonJS (`require`/`module.exports`) is Node's older, synchronous, dynamic system; I use named exports by default and dynamic `import()` to lazy-load heavy code.

## Key points
- **ES modules (ESM)**: `import`/`export` are static, meaning they must be at the top level, so tools know the dependency graph before running any code. Imports are **live, read-only bindings**. Modules run in strict mode, run once, and support top-level `await`. See [MDN: JavaScript modules](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules).
- **CommonJS (CJS)**: `require()` is a normal function call, runs synchronously, can be called anywhere (even in an `if`), and gives you a **copy** of whatever `module.exports` held at that time.
- **Named exports** (`export function buy()`) can be many per file and must be imported by exact name. A **default export** (one per file) can be imported with any name.
- [Dynamic `import()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/import) returns a promise for the module. Bundlers split it into a separate chunk, which is the basis of code splitting and lazy loading.
- [Tree shaking](https://developer.mozilla.org/en-US/docs/Glossary/Tree_shaking) is when the bundler removes exports nobody imports. It needs static ESM and code without side effects at import time (`"sideEffects": false` in `package.json` helps).

## Example
```js
// counter.mjs
export let count = 0;
export function inc() { count++; }
export default function hello() { return 'hi'; }

// counter.cjs
let count = 0;
module.exports = { count, inc() { count++; } };

// main.mjs
import greet, { count, inc } from './counter.mjs';
inc();
console.log(count, greet()); // 1 'hi'   live binding sees the update

const c = require('./counter.cjs'); // via createRequire in ESM
c.inc();
console.log(c.count);        // 0   CJS exported a copy of the number

// count = 5;                // TypeError: imports are read-only

// Dynamic import: loaded only when needed
const m = await import('./counter.mjs');
console.log(Object.keys(m)); // ['count', 'default', 'inc']
```

Lazy loading a heavy chart in Svelte 5:

```svelte
<script>
  let Chart = $state(null);
  async function showChart() {
    // separate chunk, downloaded only on click
    Chart = (await import('./CandleChart.svelte')).default;
  }
</script>

<button onclick={showChart}>Show chart</button>
{#if Chart}<Chart symbol="AAPL" />{/if}
```

## When to use it
- **Code splitting:** charting libraries, PDF statement export or an options-chain view can be loaded with `import()` only when the user opens them, which keeps the first load fast. SvelteKit already splits code per route.
- **Utility libraries:** import named functions (`import { debounce } from 'lodash-es'`) so the bundler can drop the rest. `import _ from 'lodash'` (CJS) pulls in everything.
- **Node tooling:** you still meet CJS in older packages and config files (`.cjs`).

## Likely questions
### What are the differences between ES modules and CommonJS?
ESM uses `import`/`export`, is static and analysed before running, loads asynchronously, gives live read-only bindings, is always strict mode, and supports top-level `await`. CJS uses `require`/`module.exports`, is a runtime function call, loads synchronously, and gives you a copy of the exported value. ESM works natively in browsers; CJS is Node-only. In Node, a file is ESM if it ends in `.mjs` or the `package.json` has `"type": "module"`.

### Named vs default exports: which do you prefer?
Named exports. The name is fixed, so auto-import, refactor/rename and search all work, typos fail at build time, and tree shaking is clearer. Defaults let each importer pick a name, which leads to inconsistent names across the codebase. Defaults make sense when a file has one main thing, like a Svelte component (which is always a default export).

### What is dynamic `import()` and when would you use it?
It is a function-like expression that loads a module at runtime and returns a promise of its namespace object. You can use it anywhere, with a variable path, and bundlers create a separate chunk for it. I use it for heavy, rarely-used features (charts, editors, admin screens), for loading a locale file based on the user's language, and for feature flags.

### What is tree shaking and what breaks it?
The bundler (Rollup, or Rolldown in newer Vite versions) follows static `import`s and removes exports that are never used. It breaks with CommonJS (because `require` is dynamic), with modules that run side effects at import time (like patching globals), and with "barrel" files that import everything if the package isn't marked side-effect-free. Importing a whole namespace and accessing properties by a dynamic key also prevents it.

### What are live bindings?
An ESM import is a live view of the exporting module's variable, not a copy. If the module changes `count`, every importer sees the new value. Importers can't assign to it. In CJS you get a snapshot of the value at `require` time.

### How do circular dependencies behave?
ESM handles them better because bindings are live, but you can still read a variable before it is initialised and get a `ReferenceError`. In CJS, you get a partially filled `module.exports`. Best fix: move shared code into a third module.

## Common mistakes
- Mixing `require` and `import` in the same file without knowing the module type.
- Using a default export plus re-exports and getting `.default.default` problems in CJS interop.
- Forgetting the file extension in browser or Node ESM imports (bundlers hide this).
- Barrel files (`index.js` re-exporting everything) that hurt tree shaking and slow dev servers.

## Resources
- [MDN: JavaScript modules guide](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules) - full guide including dynamic import
- [javascript.info: Export and Import](https://javascript.info/import-export) - every export/import form
- [javascript.info: Dynamic imports](https://javascript.info/modules-dynamic-imports) - short and clear
- [MDN: Tree shaking](https://developer.mozilla.org/en-US/docs/Glossary/Tree_shaking) - definition
