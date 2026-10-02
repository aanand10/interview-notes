# Modern syntax

> **In one line:** Destructuring pulls values out of objects and arrays, `...` spreads values out (spread) or gathers them in (rest) depending on where it sits, `?.` stops safely on `null`/`undefined`, and `??` only falls back on `null`/`undefined` while `||` falls back on any falsy value.

## Key points
- [Destructuring](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring) supports renaming (`bid: price`), defaults (`ask = 'n/a'`), nesting, skipping array items and a rest element. Defaults apply only when the value is `undefined`, not `null`.
- [Spread](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_syntax) **expands** an iterable or object: in calls `fn(...args)`, array literals `[...a, ...b]`, object literals `{ ...a, b: 2 }`. [Rest](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/rest_parameters) **collects** the leftovers into an array or object: in parameters `(...nums)` and destructuring `{ a, ...rest }`. Rest must be last.
- [Optional chaining](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Optional_chaining) `a?.b`, `a?.[key]`, `a.fn?.()` returns `undefined` instead of throwing if the left side is `null` or `undefined`. It short-circuits the rest of the chain.
- [`??` (nullish coalescing)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Nullish_coalescing) uses the right side only for `null`/`undefined`. [`||`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Logical_OR) uses it for any falsy value (`0`, `''`, `false`, `NaN` too).
- Logical assignment: `a ??= b`, `a ||= b`, `a &&= b` assign only when needed.

## Example
```js
const quote = { sym: 'AAPL', bid: 189.5, meta: { exchange: 'NASDAQ' }, volume: 0 };

// Object destructuring: rename, nested, default, rest
const { sym, bid: price, meta: { exchange }, ask = 'n/a', ...rest } = quote;
console.log(sym, price, exchange, ask, rest); // AAPL 189.5 NASDAQ n/a { volume: 0 }

// Array destructuring: skip, default, rest
const [first, , third = 'x', ...others] = [1, 2, undefined, 4, 5];
console.log(first, third, others); // 1 x [4, 5]

// Swap without a temp variable
let a = 1, b = 2;
[a, b] = [b, a]; // a = 2, b = 1

// Rest in parameters, spread in the call
function sum(...nums) { return nums.reduce((s, n) => s + n, 0); }
sum(...[1, 2, 3]);         // 6
Math.max(...[3, 9, 2]);    // 9

// Object spread: later keys win
({ ...{ a: 1, b: 2 }, ...{ b: 3 } }); // { a: 1, b: 3 }

// Optional chaining
const user = null;
user?.profile.name;        // undefined (stops at user, no error)
user?.getName?.();         // undefined
quote.meta?.['exchange'];  // 'NASDAQ'

// ?? vs ||
quote.volume || 100;       // 100  (0 is falsy, real value lost!)
quote.volume ?? 100;       // 0    (0 is kept)

// Logical assignment
const settings = { theme: '' };
settings.theme ||= 'dark'; // '' is falsy, so becomes 'dark'
settings.size ??= 12;      // undefined, so becomes 12
```

## When to use it
- **Svelte 5 props:** `let { symbol, qty = 1, ...rest } = $props();` uses destructuring with defaults and rest to forward other attributes.
- **Immutable updates:** `order = { ...order, qty: 10 }` or `list = [...list, newItem]` create new references.
- **API data:** `res.data?.quotes?.[0]?.price ?? 0` reads deep, possibly-missing fields without crashing.
- **Numbers where zero is valid:** quantity, price change, volume. Use `??` so `0` is not replaced by a default.

## Likely questions
### What is destructuring? Show a few forms.
It is a short way to unpack values from objects or arrays into variables. You can rename (`{ bid: price }`), give defaults (`{ qty = 1 }`), go nested (`{ meta: { exchange } }`), skip array items (`[a, , c]`) and collect the rest (`{ id, ...others }`). It also works in function parameters, which is how Svelte's `$props()` is normally used.

### What is the difference between spread and rest?
They use the same `...` but do opposite jobs. Spread expands one thing into many: `fn(...args)` or `[...a, ...b]`. Rest gathers many into one: `function f(...args)` or `const { a, ...others } = obj`. Simple rule: on the right side of `=` or in a call it's spread; on the left side or in a parameter list it's rest. Both spreads make only a **shallow** copy.

### How does optional chaining work?
`a?.b` checks if `a` is `null` or `undefined`. If so, the whole expression returns `undefined` and the rest of the chain is skipped; otherwise it reads `b` as normal. It has forms for brackets (`a?.[k]`) and calls (`a.fn?.()`). It only guards the part right before `?.`, so `a?.b.c` still throws if `b` is `undefined`. You can't use it on the left side of an assignment.

### What is the difference between `??` and `||`?
`||` returns the right side when the left is any falsy value, so `0 || 100` is `100`. `??` returns the right side only when the left is `null` or `undefined`, so `0 ?? 100` is `0`. For numbers, empty strings and booleans that can legitimately be falsy, `??` is the correct choice. You can't mix `??` with `||` or `&&` without parentheses; it is a syntax error.

### Does a destructuring default apply for `null`?
No. Defaults apply only for `undefined`. `const { x = 5 } = { x: null }` gives `x === null`. If the API sends `null`, use `x ?? 5` instead.

### What are template literals and tagged templates?
Backtick strings that allow `${expression}` interpolation and multi-line text. A tag function (`` sql`...` ``) receives the string parts and values separately, which libraries use for safe escaping.

## Common mistakes
- Using `||` for defaults on numeric fields and losing a real `0`.
- Thinking spread deep-copies nested objects.
- Overusing `?.` everywhere, which hides bugs where a value should never be missing.
- Destructuring from `undefined` (`const { a } = undefined` throws). Default the source: `({ a } = {})`.

## Resources
- [javascript.info: Destructuring assignment](https://javascript.info/destructuring-assignment) - every form with exercises
- [javascript.info: Rest parameters and spread syntax](https://javascript.info/rest-parameters-spread) - clear side-by-side comparison
- [javascript.info: Optional chaining](https://javascript.info/optional-chaining) - short-circuit rules
- [javascript.info: Nullish coalescing operator](https://javascript.info/nullish-coalescing-operator) - `??` vs `||` with examples
- [MDN: Nullish coalescing assignment (??=)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Nullish_coalescing_assignment) - logical assignment
