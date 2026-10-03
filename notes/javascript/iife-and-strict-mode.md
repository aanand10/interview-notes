# IIFE and strict mode

> **In one line:** An IIFE is a function that runs right after it is defined, used before ES modules to create a private scope; strict mode is an opt-in, safer version of JavaScript that turns silent mistakes into errors.

## Key points
- [IIFE](https://developer.mozilla.org/en-US/docs/Glossary/IIFE) = Immediately Invoked Function Expression: `(function () { ... })();`
- **Why it was used**: before `let`/`const` and ES modules, `var` was function-scoped, so wrapping code in a function was the only way to avoid polluting the global scope and to keep variables private (the module pattern). It also fixed the `var` loop + `setTimeout` bug.
- Today ES modules and block scope cover most of this. You still see IIFEs for top-level `async` code in scripts and in bundler output.
- [Strict mode](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Strict_mode) is turned on with `'use strict'` at the top of a file or function. **ES modules and class bodies are always strict.**

## Example
```js
// IIFE: private scope, only the API leaks out
const priceStore = (function () {
  const prices = {}; // private
  return {
    set: (s, p) => { prices[s] = p; },
    get: (s) => prices[s],
  };
})();

// Async IIFE in a plain script (no top-level await there)
(async () => {
  const res = await fetch('/api/watchlist');
  console.log(await res.json());
})();
```

Strict mode turning silent bugs into errors (checked with node):
```js
'use strict';
try {
  undeclared = 1;            // typo or missing let
} catch (e) {
  console.log(e.constructor.name, e.message); // ReferenceError undeclared is not defined
}

function whoAmI() { return this; }
console.log(whoAmI());       // undefined (in sloppy mode it would be the global object)
```

## When to use it
- IIFE: a small standalone script tag (analytics, widget embed) that must not leak globals; `(async () => {...})()` for async startup code.
- Strict mode: always. In practice you get it for free with ES modules, Svelte components and classes.

## Likely questions
### Why were IIFEs used?
To create a private scope. Before ES6, only functions created scope, so libraries wrapped their code in an IIFE to avoid global variable clashes and to expose a single API object. They also captured loop variables correctly with `var`. Modules and `let` made most uses unnecessary.

### Why the wrapping parentheses?
`function () {}()` at the start of a statement is parsed as a function *declaration*, which cannot be called immediately and needs a name, so it is a syntax error. The parentheses make it an *expression*.

### What does strict mode change?
- Assigning to an undeclared variable throws a ReferenceError instead of creating a global.
- `this` inside a plain function call is `undefined`, not the global object.
- Writing to a read-only property or a non-writable/frozen object throws instead of failing silently.
- Duplicate parameter names are a syntax error, `with` is banned, and legacy octal literals like `010` are not allowed.
- `delete` on a non-deletable property throws; `eval` cannot add variables to the surrounding scope.
- Some future keywords (`implements`, `package`, etc.) are reserved.

### Do I need `'use strict'` in modern code?
Usually no. ES modules (`import`/`export`) and class bodies are automatically strict, and so is code compiled by Svelte or bundled as ESM.

## Resources
- [MDN: IIFE](https://developer.mozilla.org/en-US/docs/Glossary/IIFE) - definition and examples
- [MDN: Strict mode](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Strict_mode) - full list of changes
- [javascript.info: The modern mode, "use strict"](https://javascript.info/strict-mode) - short and easy intro
