# `flatten` array / object

> **In one line:** Flattening an array turns nested arrays into one level (like `Array.prototype.flat`), and flattening an object turns nested keys into dotted paths like `{ "user.address.city": "Pune" }`; both are recursion (or an explicit stack) over the nested structure.

## Requirements
- **Array:** `flatten(arr, depth = Infinity)`. Keep the order. Respect `depth` like [`Array.prototype.flat`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/flat). Skip holes in sparse arrays (that is what `flat` does).
- **Object:** `flattenObject(obj, prefix = '', sep = '.')`. Nested plain objects become path keys. Ask: how should **arrays** be keyed (`a.0.b` or `a[0].b`)? What about **empty objects/arrays** (drop them or keep them as values)? `null`, `Date`, `Map`: treat as leaf values.
- Bonus: `unflatten` (the reverse), an iterative version (no recursion limit), and circular reference handling.

## Array: recursive
```js
function flatten(arr, depth = Infinity) {
  const out = [];
  for (const item of arr) {
    if (Array.isArray(item) && depth > 0) {
      out.push(...flatten(item, depth - 1));
    } else {
      out.push(item);
    }
  }
  return out;
}

flatten([1, [2, [3, [4]], 5]]);    // [1, 2, 3, 4, 5]
flatten([1, [2, [3, [4]], 5]], 1); // [1, 2, [3, [4]], 5]
```
`for...of` visits holes as `undefined`. To skip holes exactly like `flat`, use `arr.forEach` or check `i in arr`.

## Array: with `reduce` (common one-liner ask)
```js
const flattenReduce = (arr, depth = Infinity) =>
  arr.reduce(
    (acc, item) =>
      Array.isArray(item) && depth > 0
        ? acc.concat(flattenReduce(item, depth - 1))
        : acc.concat([item]), // wrap so a non-array item is never spread by concat
    []
  );
```

## Array: iterative with a stack (no recursion limit)
```js
function flattenIterative(arr) {
  const stack = [...arr];
  const out = [];
  while (stack.length) {
    const item = stack.pop();
    if (Array.isArray(item)) stack.push(...item); // unpack, keep order by reversing at the end
    else out.push(item);
  }
  return out.reverse();
}
```
Very deep nesting (tens of thousands of levels) can throw `RangeError: Maximum call stack size exceeded` with recursion. The stack version avoids that.

## Object flattening
```js
const isPlainObject = (v) =>
  v !== null && typeof v === 'object' && Object.getPrototypeOf(v) === Object.prototype;

function flattenObject(obj, prefix = '', sep = '.', out = {}) {
  for (const [key, value] of Object.entries(obj)) {
    const path = prefix ? `${prefix}${sep}${key}` : key;
    const isContainer = isPlainObject(value) || Array.isArray(value);
    if (isContainer && Object.keys(value).length > 0) {
      flattenObject(value, path, sep, out); // arrays give keys "0", "1", ...
    } else {
      out[path] = value; // leaf: primitive, null, Date, empty {} or []
    }
  }
  return out;
}

const user = {
  name: 'Asha',
  address: { city: 'Pune', geo: { lat: 18.52, lng: 73.85 } },
  tags: ['pro', 'beta'],
  prefs: {},
  joined: new Date('2024-01-01'),
};

flattenObject(user);
// {
//   name: 'Asha',
//   'address.city': 'Pune',
//   'address.geo.lat': 18.52,
//   'address.geo.lng': 73.85,
//   'tags.0': 'pro',
//   'tags.1': 'beta',
//   prefs: {},
//   joined: 2024-01-01T00:00:00.000Z
// }
```

## Unflatten (the reverse)
```js
function unflattenObject(flat, sep = '.') {
  const result = {};
  for (const [path, value] of Object.entries(flat)) {
    const keys = path.split(sep);
    let node = result;
    keys.forEach((key, i) => {
      if (i === keys.length - 1) {
        node[key] = value;
      } else {
        const nextIsIndex = /^\d+$/.test(keys[i + 1]);
        node[key] ??= nextIsIndex ? [] : {};
        node = node[key];
      }
    });
  }
  return result;
}

unflattenObject({ 'a.b': 1, 'a.c.0': 'x', 'a.c.1': 'y' });
// { a: { b: 1, c: ['x', 'y'] } }
```

## Circular references
```js
function flattenObjectSafe(obj, prefix = '', out = {}, seen = new WeakSet()) {
  if (seen.has(obj)) throw new TypeError(`Circular reference at "${prefix}"`);
  seen.add(obj);
  for (const [key, value] of Object.entries(obj)) {
    const path = prefix ? `${prefix}.${key}` : key;
    if ((isPlainObject(value) || Array.isArray(value)) && Object.keys(value).length) {
      flattenObjectSafe(value, path, out, seen);
    } else {
      out[path] = value;
    }
  }
  seen.delete(obj); // the same object may appear twice side by side; only a cycle is an error
  return out;
}
```

## When to use it
- **Forms:** map flat field names (`address.city`) to nested API payloads and back. Libraries like React Hook Form use the same path idea.
- **Validation errors** from the server (`{ "items.2.qty": "must be > 0" }`) to show next to the right field.
- **Analytics / logging / CSV export:** tools want flat key-value rows.
- **i18n files:** `{ home: { title: '...' } }` <-> `home.title`.
- **Diffing two objects:** flatten both, then compare keys.

## Likely questions
### Implement `Array.prototype.flat` as a polyfill.
```js
if (!Array.prototype.myFlat) {
  Array.prototype.myFlat = function (depth = 1) { // note: default depth is 1, not Infinity
    const out = [];
    const walk = (arr, d) => {
      arr.forEach((item) => { // forEach skips holes, like flat
        if (Array.isArray(item) && d > 0) walk(item, d - 1);
        else out.push(item);
      });
    };
    walk(this, depth);
    return out;
  };
}

[1, [2, [3]]].myFlat();         // [1, 2, [3]]
[1, [2, [3]]].myFlat(Infinity); // [1, 2, 3]
```

### What is the time and space complexity?
O(n) time where n is the total number of values at every level, because each value is visited once. Extra space is O(d) for the recursion stack (d = depth) plus the output. The `concat` version in `reduce` copies arrays repeatedly, so it can be closer to O(n * d); mention it if asked which is faster.

### How do you treat arrays inside objects?
Agree with the interviewer. Common choices are `tags.0` (simple, `lodash.set` understands it) or `tags[0]` (clearer). Also decide whether an empty array or object is kept as a value or dropped. Keeping it makes `unflatten` give back the same shape.

### Why not `JSON.stringify` tricks?
`JSON.stringify(arr).replace(/[[\]]/g, '')` only works for numbers, breaks on strings that contain brackets or commas, and loses types. Interviewers want to see recursion.

## Common mistakes
- Treating `null` as an object (`typeof null === 'object'`) and recursing into it.
- Recursing into `Date`, `Map` or class instances. Check for plain objects only.
- Mutating the input instead of building a new result.
- Forgetting `flat()` defaults to depth **1**.
- A shared default `out = {}` is fine as a parameter default (it is created fresh on each call), but a module-level `const out = {}` leaks between calls.

## Resources
- [MDN: Array.prototype.flat](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/flat) - depth and holes behaviour
- [MDN: Array.prototype.flatMap](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/flatMap) - map then flat one level
- [MDN: Object.entries](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/entries) - iterate own enumerable keys
- [javascript.info: Recursion and stack](https://javascript.info/recursion) - recursion over nested structures
