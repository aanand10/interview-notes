# Equality and coercion

> **In one line:** `===` compares value and type with no conversion, while `==` first converts (coerces) the two sides to a common type, so I use `===` by default and only use `== null` as a shortcut for "null or undefined".

## Key points
- **Coercion** means JavaScript quietly converts a value to another type, like a string to a number. It can be explicit (`Number(x)`) or implicit (`'5' - 3`). See [MDN: Type coercion](https://developer.mozilla.org/en-US/docs/Glossary/Type_coercion).
- `===` (strict equality) never converts. `==` (loose equality) follows the [Abstract Equality rules](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Equality): objects become primitives, booleans become numbers, strings become numbers.
- `+` is special: if either side is a string (after converting objects to primitives), it **joins strings**. Other math operators (`-`, `*`, `/`) always convert to **numbers**.
- There are only 8 [falsy](https://developer.mozilla.org/en-US/docs/Glossary/Falsy) values: `false`, `0`, `-0`, `0n`, `''`, `null`, `undefined`, `NaN`. Everything else is [truthy](https://developer.mozilla.org/en-US/docs/Glossary/Truthy), including `[]`, `{}` and `'0'`.
- `NaN` is not equal to anything, even itself. Use `Number.isNaN(x)` or [`Object.is`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/is), which also tells `0` and `-0` apart.

## Example
Classic output questions (run in Node 24):

```js
console.log([] == ![]);        // true
console.log('5' + 3);          // '53'
console.log('5' - 3);          // 2
console.log('5' * '2');        // 10
console.log(true + 1);         // 2
console.log([] + []);          // ''  (empty string)
console.log([] + {});          // '[object Object]'
console.log(null == undefined, null === undefined); // true false
console.log(null == 0, null >= 0);                  // false true
console.log(NaN == NaN, Object.is(NaN, NaN));       // false true
console.log(0 === -0, Object.is(0, -0));            // true false
console.log('' == 0, '0' == false);                 // true true
console.log(Boolean([]), Boolean('0'), Boolean('')); // true true false
console.log('b' + 'a' + +'a' + 'a');                 // 'baNaNa'
console.log(1 < 2 < 3, 3 > 2 > 1);                   // true false
```

Why each one:
- `[] == ![]`: `![]` is `false` (an array is truthy). Then `[] == false` becomes `'' == 0` becomes `0 == 0`, so `true`.
- `'5' + 3`: one side is a string, so `+` joins: `'53'`.
- `'5' - 3`: `-` only works on numbers, so `'5'` becomes `5`: `2`.
- `'5' * '2'`: both become numbers: `10`.
- `true + 1`: `true` becomes `1`: `2`.
- `[] + []`: both arrays become `''`, joined gives `''`.
- `[] + {}`: `''` + `'[object Object]'`.
- `null == undefined` is a special rule (they only loosely equal each other). `===` sees different types.
- `null == 0` is `false` because `null` is not converted for `==`, but `null >= 0` converts `null` to `0`, so `true`.
- `NaN` is never equal to itself with `==` or `===`; `Object.is` says `true`.
- `0 === -0` is `true`; `Object.is` tells them apart.
- `'' == 0`: `''` becomes `0`. `'0' == false`: both become `0`.
- Arrays and non-empty strings like `'0'` are truthy; only `''` is falsy.
- `+'a'` is `NaN`, which becomes the string `'NaN'` in the middle.
- `1 < 2` is `true`, `true < 3` is `1 < 3`. `3 > 2` is `true`, `true > 1` is `1 > 1`, `false`.

## When to use it
- In a trading app, an order form gives you strings from inputs. `qty + 1` on `'10'` gives `'101'`, a real bug. Convert first: `Number(input.value)` or use `valueAsNumber`.
- Use `x == null` to check "null or undefined" in one go, for example when an API price field may be missing.
- Be careful with truthy checks on numbers: `if (price)` treats a price of `0` as missing. Use `price != null` or `Number.isFinite(price)`.

## Likely questions
### What is the difference between `==` and `===`?
`===` checks type and value and never converts anything. `==` converts the values to a common type first and then compares. That conversion has surprising rules, so I always use `===`, with one exception: `value == null`, which is a clean way to match both `null` and `undefined`.

### What are truthy and falsy values?
When a value is used in a condition, JS converts it to a boolean. The falsy ones are `false`, `0`, `-0`, `0n`, `''`, `null`, `undefined` and `NaN`. Everything else is truthy, including empty arrays, empty objects and the string `'0'`. That is why `if (arr.length)` is fine but `if (arr)` is always true.

### Why is `[] == ![]` true?
First `![]` runs. Arrays are truthy, so `![]` is `false`. Now we have `[] == false`. The boolean becomes `0`, the array becomes the primitive `''`, and `''` becomes `0`. So it ends as `0 == 0`, which is `true`.

### Why is `'5' + 3` `'53'` but `'5' - 3` `2`?
The `+` operator does both addition and string joining. If either side is a string, it joins. `-` has only a math meaning, so it converts both sides to numbers.

### How does an object get converted to a primitive?
JS calls `Symbol.toPrimitive` if it exists, otherwise `valueOf()` and then `toString()` (order depends on the "hint", number or string). Plain arrays return themselves from `valueOf`, so `toString` is used: `[1,2]` becomes `'1,2'`, `[]` becomes `''`.

### What is the difference between `===` and `Object.is`?
They agree except in two cases: `Object.is(NaN, NaN)` is `true` and `Object.is(0, -0)` is `false`. Svelte and React use `Object.is`-style checks to decide if state changed.

## Common mistakes
- Using `if (value)` when `0` or `''` is a valid value (a zero balance, an empty search).
- Comparing with `NaN` using `===`. Use `Number.isNaN`.
- Forgetting that `typeof NaN` is `'number'`.
- `parseInt('08abc')` returns `8` (it stops at the first bad character), but `Number('08abc')` returns `NaN`. Pick the one you mean.
- Thinking `null >= 0` and `null == 0` should agree. They use different rules.

## Resources
- [MDN: Equality comparisons and sameness](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Equality_comparisons_and_sameness) - the full table of `==`, `===`, `Object.is`
- [MDN: Equality (==)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Equality) - step by step loose equality algorithm
- [javascript.info: Type conversions](https://javascript.info/type-conversions) - simple rules for string, number, boolean conversion
- [javascript.info: Comparisons](https://javascript.info/comparison) - covers the `null`/`undefined` traps
- [MDN: Falsy](https://developer.mozilla.org/en-US/docs/Glossary/Falsy) - the complete falsy list
