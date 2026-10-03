# Output questions: types and coercion

> **In one line:** **Coercion** means JavaScript converts a value to another type on its own. `+` prefers strings if either side is a string, other math operators prefer numbers, and `==` converts types before comparing while `===` does not.

## Key points
- `+` with a string on either side joins strings. `-`, `*`, `/`, `%` and unary `+x` always convert to numbers. See [javascript.info: Type conversions](https://javascript.info/type-conversions).
- Objects and arrays are first turned into a **primitive** (a plain value like a string or number), usually via `toString()`: `[]` becomes `""`, `[1,2]` becomes `"1,2"`, `{}` becomes `"[object Object]"`.
- `==` (**loose equality**) converts types: booleans become numbers, and `null` only equals `undefined`. Use `===` in real code. See [MDN: Equality comparisons](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Equality_comparisons_and_sameness).
- Only 8 values are **falsy**: `false`, `0`, `-0`, `0n`, `""`, `null`, `undefined`, `NaN`. Everything else, including `[]`, `{}` and `"0"`, is truthy.
- Numbers are 64-bit floating point, so `0.1 + 0.2` is not exactly `0.3`.

All outputs below were run with Node.

## Drills

### Q1. Adding arrays and objects
```js
console.log([] + {});
console.log([1, 2] + [3]);
```
**Answer:**
```text
[object Object]
1,23
```
`+` turns both sides into strings: `"" + "[object Object]"`, and `"1,2" + "3"`. (Also: `[] + []` is an empty string.)

### Q2. [] == ![]
```js
console.log([] == ![]);
console.log([] == false);
console.log(!![]);
```
**Answer:**
```text
true
true
true
```
`![]` is `false` because `[]` is truthy. Then `[] == false` becomes `"" == 0`, then `0 == 0`, which is `true`. So an array is truthy and also `== false` at the same time.

### Q3. Minus vs plus with strings
```js
console.log("5" - 3);
console.log("5" + 3);
console.log("5" * "2");
console.log(true + 1);
console.log("b" + "a" + +"a" + "a");
```
**Answer:**
```text
2
53
10
2
baNaNa
```
`-` and `*` force numbers. `+` with a string joins. `true` becomes `1`. In the last one, `+"a"` is `NaN`, which gets joined as the text "NaN".

### Q4. null and undefined
```js
console.log(null == undefined);
console.log(null === undefined);
console.log(null == 0);
console.log(null >= 0);
console.log(undefined == 0);
```
**Answer:**
```text
true
false
false
true
false
```
`==` has a special rule: `null` and `undefined` equal each other and nothing else. But `>=` uses number conversion, where `null` is `0`, so `0 >= 0` is `true`. That is why `null == 0` is false but `null >= 0` is true.

### Q5. NaN
```js
console.log(NaN === NaN);
console.log(NaN == NaN);
console.log(Number.isNaN(NaN));
console.log(isNaN("abc"));
console.log(Number.isNaN("abc"));
```
**Answer:**
```text
false
false
true
true
false
```
`NaN` is the only value not equal to itself. Global `isNaN` converts first (`"abc"` becomes `NaN`). `Number.isNaN` does not convert, so it is the safe one.

### Q6. typeof
```js
console.log(typeof null);
console.log(typeof NaN);
console.log(typeof []);
console.log(typeof function () {});
console.log(typeof undefined);
console.log(Array.isArray([]));
```
**Answer:**
```text
object
number
object
function
undefined
true
```
`typeof null` is `"object"` because of an old bug kept for compatibility. `NaN` is a number type ("Not a Number" value). Arrays are objects, so use `Array.isArray`.

### Q7. 0.1 + 0.2
```js
console.log(0.1 + 0.2);
console.log(0.1 + 0.2 === 0.3);
console.log(Math.abs(0.1 + 0.2 - 0.3) < Number.EPSILON);
```
**Answer:**
```text
0.30000000000000004
false
true
```
0.1 and 0.2 cannot be stored exactly in binary floating point. Compare with a small tolerance (**epsilon**), or for money store integers (paise/cents).

### Q8. Object.is
```js
console.log(Object.is(NaN, NaN));
console.log(Object.is(0, -0));
console.log(0 === -0);
console.log(Object.is("a", "a"));
```
**Answer:**
```text
true
false
true
true
```
`Object.is` is like `===` except for two cases: `NaN` equals `NaN`, and `0` does not equal `-0`. React uses it to compare state. See [MDN: Object.is](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/is).

### Q9. Sorting numbers without a comparator
```js
console.log([10, 1, 5, 100, 25].sort());
console.log([10, 1, 5, 100, 25].sort((a, b) => a - b));
```
**Answer:**
```text
[ 1, 10, 100, 25, 5 ]
[ 1, 5, 10, 25, 100 ]
```
Default `sort` compares items as **strings**, so `"100"` comes before `"25"`. Always pass `(a, b) => a - b` for numbers (for example sorting prices). Also, `sort` changes the array in place; `toSorted` returns a new one.

### Q10. parseInt quirks
```js
console.log(parseInt("12px"));
console.log(parseInt("px12"));
console.log(parseInt("08"));
console.log(parseInt("0x1F"));
console.log(parseInt(0.0000005));
console.log(Number("12px"));
```
**Answer:**
```text
12
NaN
8
31
5
NaN
```
`parseInt` reads digits from the start and stops at the first bad character. It reads `0x` as hex. `0.0000005` becomes the string `"5e-7"` first, so it reads `5`. `Number()` is strict: any junk gives `NaN`.

### Q11. map(parseInt)
```js
console.log(["1", "2", "3"].map(parseInt));
console.log(["1", "2", "3"].map(Number));
```
**Answer:**
```text
[ 1, NaN, NaN ]
[ 1, 2, 3 ]
```
`map` passes `(value, index)`, and `parseInt` takes `(string, radix)`. So it calls `parseInt("2", 1)` (radix 1 is invalid) and `parseInt("3", 2)` ("3" is not a binary digit). Use `map(Number)` or `map(s => parseInt(s, 10))`.

### Q12. Unary plus
```js
console.log(+"");
console.log(+" ");
console.log(+"abc");
console.log(+[]);
console.log(+{});
console.log(+null);
console.log(+undefined);
console.log(+true);
```
**Answer:**
```text
0
0
NaN
0
NaN
0
NaN
1
```
Empty or whitespace-only strings become `0`. `null` becomes `0` but `undefined` becomes `NaN`. A form field with an empty value turning into `0` is a real bug source in order forms.

### Q13. Comparing strings and chaining comparisons
```js
console.log("2" > "12");
console.log(2 > "12");
console.log(1 < 2 < 3);
console.log(3 > 2 > 1);
```
**Answer:**
```text
true
false
true
false
```
Two strings compare letter by letter, and `"2"` > `"1"`. If one side is a number, both become numbers. `3 > 2 > 1` is `(true) > 1`, which is `1 > 1`, so `false`.

### Q14. Truthy and falsy
```js
console.log(Boolean("0"));
console.log(Boolean(""));
console.log(Boolean([]));
console.log(Boolean({}));
console.log(Boolean(0n));
console.log("0" == false);
```
**Answer:**
```text
true
false
true
true
false
true
```
`"0"` is a non-empty string, so it is truthy. But `"0" == false` converts both to `0`, so it is `true`. Truthiness and `==` follow different rules.

### Q15. Mixed operators
```js
console.log(1 + "2" - 1);
console.log("3" * [4]);
console.log([] == "");
console.log([0] == false);
```
**Answer:**
```text
11
12
true
true
```
Left to right: `1 + "2"` is `"12"`, then `"12" - 1` is `11`. `[4]` becomes `"4"` then `4`. `[]` becomes `""`. `[0]` becomes `"0"` then `0`, and `false` becomes `0`.

### Q16. Big integers
```js
console.log(9007199254740992 === 9007199254740993);
```
**Answer:**
```text
true
```
Above `Number.MAX_SAFE_INTEGER` (2^53 - 1) numbers lose precision, so two different literals become the same value. Use `BigInt` or strings for large IDs (for example order IDs from a backend).

## When it matters in real code
- Prices and quantities from `<input>` are strings. `"100" + 5` gives `"1005"`. Convert with `Number()` and check `Number.isNaN`.
- Money math: store paise as integers, format with `Intl.NumberFormat` only for display.
- Sorting a watchlist by price needs a numeric comparator.

## Common mistakes
- Using `==` and then being surprised by `"0" == false`. Use `===` (the only common exception is `x == null` to catch both null and undefined).
- Using `parseInt` without a radix, or with `map`.
- Treating `[]` as falsy in an `if`.

## Resources
- [javascript.info: Type conversions](https://javascript.info/type-conversions) - string, number and boolean conversion rules
- [MDN: Equality comparisons and sameness](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Equality_comparisons_and_sameness) - `==` vs `===` vs `Object.is`
- [MDN: parseInt](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/parseInt) - radix and parsing rules
- [MDN: Array.prototype.sort](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/sort) - default string ordering
