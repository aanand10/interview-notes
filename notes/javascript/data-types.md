# Data types

> **In one line:** JavaScript has 7 primitive types that are stored and copied by value, and one reference type, `object` (which includes arrays and functions), where variables hold a reference to the same object in memory.

## Key points
- **Primitives** (7): `string`, `number`, `bigint`, `boolean`, `undefined`, `null`, `symbol`. They are immutable: you can't change a primitive, only replace it. See [MDN: Primitive](https://developer.mozilla.org/en-US/docs/Glossary/Primitive).
- **Objects** (reference types): plain objects, arrays, functions, dates, maps and so on. A variable holds a reference (like an address) to the object, not the object itself.
- JS is always **pass by value**. For objects, the value that is copied is the reference. People call this "pass by sharing": you can mutate the shared object, but reassigning the parameter does not affect the caller.
- [`typeof null`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/typeof) is `'object'`. It is a bug from the first version of JS that can never be fixed without breaking the web.
- Check arrays with [`Array.isArray(x)`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/isArray), because `typeof []` is also `'object'`.

## Example
```js
function update(n, order) {
  n = 99;          // changes the local copy only
  order.qty = 99;  // mutates the shared object
}
function replace(order) {
  order = { qty: 1 }; // points the local variable at a new object; caller unaffected
}

let count = 1;
const o = { qty: 10 };
update(count, o);
console.log(count, o); // 1 { qty: 99 }
replace(o);
console.log(o);        // { qty: 99 }

const a = { x: 1 }, b = { x: 1 }, c = a;
console.log(a === b, a === c); // false true  (compares references, not contents)

let s = 'hello';
s[0] = 'H';
console.log(s); // 'hello'  (strings are immutable)

console.log(typeof undefined, typeof undeclaredVar, typeof null);
// 'undefined' 'undefined' 'object'

console.log(0.1 + 0.2, 0.1 + 0.2 === 0.3); // 0.30000000000000004 false
console.log(2 ** 53 + 1);                   // 9007199254740992 (precision lost)
```

## When to use it
- **State updates in Svelte 5:** a deep `$state` object is a proxy, so mutating `order.qty = 5` is tracked. But with `$state.raw` or when passing data to other code, you need to know if you are sharing a reference or making a copy.
- **Money in a trading app:** `number` is a 64-bit float, so `0.1 + 0.2 !== 0.3`. Store money as integer paise/cents or use a decimal library. Use `bigint` for very large integer IDs beyond `Number.MAX_SAFE_INTEGER` (2^53 - 1).
- **Equality of objects:** two order objects with the same fields are not `===`. Compare by ID, not by object.

## Likely questions
### What is the difference between primitive and reference types?
Primitives hold the actual value and are immutable. Copying one gives you an independent copy. Objects are stored once in memory, and variables hold a reference to them. Copying the variable copies the reference, so both variables point to the same object, and a change through one is visible through the other.

### Is JavaScript pass by value or pass by reference?
Strictly, always pass by value. For objects the value is a reference, so the function can mutate the caller's object but cannot make the caller's variable point to a new object. If you reassign the parameter inside the function, the caller doesn't see it.

### Why is `typeof null` equal to `'object'`?
In the first JS engine, values had a type tag and the tag for objects was 0. `null` was the null pointer, also all zeros, so it was reported as an object. Fixing it would break old websites, so it stays. To check for null, use `x === null`.

### How do you check if something is an array?
Use `Array.isArray(x)`. `typeof` returns `'object'` for arrays. `x instanceof Array` mostly works but fails for arrays from another iframe or realm, because each has its own `Array` constructor. `Object.prototype.toString.call(x)` returns `'[object Array]'` and is the old reliable fallback.

### What is the difference between `null` and `undefined`?
`undefined` means "not set yet": a declared variable without a value, a missing property, a missing argument. `null` is an intentional "no value" that a developer sets. `null == undefined` is `true`, but `===` is `false`.

### How do you reliably get the type of any value?
`typeof` works for primitives and functions. For `null` check `=== null`, for arrays use `Array.isArray`, and for built-ins like `Date` use `Object.prototype.toString.call(x)`, which returns strings like `'[object Date]'`.

## Common mistakes
- Thinking `const` makes an object immutable. It only stops reassignment; you can still change properties.
- Comparing arrays or objects with `===` and expecting a content check.
- Using `typeof x === 'object'` as an "is object" check without excluding `null` and arrays.
- Doing money math with floats.

## Resources
- [MDN: JavaScript data types and data structures](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures) - the full list of types with details
- [MDN: typeof](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/typeof) - table of results and the `null` history
- [javascript.info: Data types](https://javascript.info/types) - simple intro to each type
- [MDN: Array.isArray](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/isArray) - why it beats `instanceof`
