# Objects

> **In one line:** A shallow copy (spread, `Object.assign`) copies only the top level and still shares nested objects, a deep copy (`structuredClone`) copies everything, and `Object.freeze` / `Object.seal` lock an object to different degrees, but only one level deep.

## Key points
- **Shallow copy**: `{ ...obj }` or `Object.assign({}, obj)`. New outer object, but nested objects are the same references. See [javascript.info: Object copying](https://javascript.info/object-copy).
- **Deep copy**: [`structuredClone(obj)`](https://developer.mozilla.org/en-US/docs/Web/API/Window/structuredClone) (built into browsers and Node 17+). It keeps `Date`, `Map`, `Set`, typed arrays and circular references, but throws on functions and DOM nodes, and drops class prototypes (you get a plain object back).
- `JSON.parse(JSON.stringify(x))` is the old trick. It turns dates into strings, drops `undefined` and functions, turns `NaN` into `null`, and throws on circular references.
- [`Object.freeze`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/freeze): no add, no delete, no change. [`Object.seal`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/seal): no add, no delete, but you can still change existing values. Both are shallow. In strict mode (and so in ES modules), a blocked write throws a `TypeError`; in sloppy mode it fails silently.
- Every property has a [descriptor](https://javascript.info/property-descriptors): `value`, `writable`, `enumerable`, `configurable` (or `get`/`set` for accessors).

## Example
```js
'use strict';
const order = { sym: 'AAPL', price: { bid: 10, ask: 11 }, at: new Date(0) };

// Shallow copy shares the nested object
const shallow = { ...order };
shallow.price.bid = 99;
console.log(order.price.bid); // 99  (original changed!)

// Deep copy
const deep = structuredClone(order);
deep.price.bid = 1;
console.log(order.price.bid, deep.at instanceof Date); // 99 true

// JSON trick loses the Date
console.log(typeof JSON.parse(JSON.stringify(order)).at); // 'string'

// freeze vs seal
const f = Object.freeze({ a: 1, nested: { b: 2 } });
// f.a = 2;          -> TypeError: Cannot assign to read only property 'a'
f.nested.b = 3;      // allowed: freeze is shallow
const s = Object.seal({ a: 1 });
s.a = 2;             // allowed: seal lets you change values
// s.z = 1;          -> TypeError (cannot add)
// delete s.a;       -> TypeError (cannot delete)

// Property descriptors
const acct = {};
Object.defineProperty(acct, 'id', { value: 7 }); // flags default to false here
console.log(Object.getOwnPropertyDescriptor(acct, 'id'));
// { value: 7, writable: false, enumerable: false, configurable: false }
console.log(Object.keys(acct), JSON.stringify(acct)); // [] '{}'  (not enumerable)
```

Iterating keys (verified in Node):

```js
const proto = { inherited: true };
const obj = Object.create(proto);
obj.own = 1; obj[2] = 'two'; obj[1] = 'one'; obj[Symbol('s')] = 's';

for (const k in obj) console.log(k); // '1' '2' 'own' 'inherited'  (includes inherited)
Object.keys(obj);      // ['1', '2', 'own']  (own, enumerable, string keys)
Object.entries(obj);   // [['1','one'], ['2','two'], ['own',1]]
Reflect.ownKeys(obj);  // ['1', '2', 'own', Symbol(s)]  (all own keys, incl. symbols and non-enumerable)
Object.hasOwn(obj, 'inherited'); // false
```
Key order: integer-like keys first in ascending order, then string keys in insertion order, then symbols.

## When to use it
- **Undo or "reset form"** in an order ticket: `structuredClone(initialOrder)` so edits never touch the original.
- **Config constants** like a list of supported exchanges: `Object.freeze` stops accidental changes.
- **Svelte 5:** `$state` objects are proxies. Use [`$state.snapshot(obj)`](https://svelte.dev/docs/svelte/$state#$state.snapshot) to get a plain copy before passing it to `structuredClone`, a chart library or `postMessage`.
- **Hidden fields:** `defineProperty` with `enumerable: false` keeps an internal field out of `JSON.stringify` and `Object.keys`.

## Likely questions
### What is the difference between a shallow copy and a deep copy?
A shallow copy creates a new top-level object but copies nested objects by reference, so changing `copy.price.bid` also changes the original. A deep copy recursively copies every level, so the two are fully independent. Spread and `Object.assign` are shallow; `structuredClone` is deep.

### What does `structuredClone` support, and what are its limits?
It uses the [structured clone algorithm](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm), the same one used by `postMessage`. It handles nested objects, arrays, `Date`, `Map`, `Set`, `RegExp`, typed arrays, and circular references. It throws a `DataCloneError` for functions and DOM nodes, and it does not keep the prototype, so class instances come back as plain objects without their methods. Getters are run and saved as plain values.

### What is the difference between `Object.freeze` and `Object.seal`?
Both stop adding and deleting properties. Freeze also makes every value read-only; seal still allows changing existing values. Both are shallow, so nested objects stay editable unless you freeze them recursively. `Object.preventExtensions` is the weakest: it only blocks adding.

### What are property descriptors?
Each property has hidden flags. `writable` says if the value can change, `enumerable` says if it shows up in `for...in`, `Object.keys` and `JSON.stringify`, and `configurable` says if it can be deleted or its flags changed. You read them with `Object.getOwnPropertyDescriptor` and set them with `Object.defineProperty`. Note that `defineProperty` defaults all flags to `false`, while normal assignment sets them to `true`. Freeze works by setting `writable` and `configurable` to `false` on every property.

### What are the ways to iterate over an object's keys?
`for...in` loops over own and inherited enumerable string keys, so you usually pair it with `Object.hasOwn`. `Object.keys`, `Object.values` and `Object.entries` give only own enumerable string keys and are the normal choice. `Reflect.ownKeys` gives every own key, including symbols and non-enumerable ones.

### How would you write a simple deep clone yourself?
Recurse over own keys, create `[]` or `{}` for each nested object, and keep a `WeakMap` of already-cloned objects to handle cycles. In real code I would just use `structuredClone`.

## Common mistakes
- Thinking spread gives a deep copy.
- Thinking `const` or `Object.freeze` protects nested data.
- Using `JSON.parse(JSON.stringify())` on objects with dates, `undefined`, `Map` or cycles.
- Calling `structuredClone` on a class instance and expecting its methods to still work.
- Using `for...in` on arrays (it gives string indexes and inherited keys).

## Resources
- [javascript.info: Object references and copying](https://javascript.info/object-copy) - shallow vs deep with pictures
- [MDN: structuredClone](https://developer.mozilla.org/en-US/docs/Web/API/Window/structuredClone) - API and supported types
- [web.dev: Deep-copying in JavaScript using structuredClone](https://web.dev/articles/structured-clone) - why it beats the JSON trick
- [javascript.info: Property flags and descriptors](https://javascript.info/property-descriptors) - writable, enumerable, configurable, freeze and seal
- [MDN: Object.freeze](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/freeze) - shallow freeze and deep freeze example
