# Map, Set, WeakMap, WeakSet

> **In one line:** `Map` is a key-value store where keys can be any type, `Set` stores unique values, and their weak versions hold object keys "weakly" so the garbage collector can still free those objects, which helps avoid memory leaks.

## Key points
- [`Map`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map): keys of any type (objects, numbers), keeps insertion order, has `.size`, is directly iterable, and is optimised for frequent add/delete.
- [`Set`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Set): a collection of unique values (compared with SameValueZero, so `NaN` equals `NaN`, but two different objects are different).
- [`WeakMap`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakMap): keys must be objects (or non-registered symbols). If nothing else references the key, the entry can be garbage collected. Not iterable, no `.size`.
- `WeakSet`: same idea for a set of objects, e.g. "have I seen this object?".
- Weak collections are not iterable because entries can vanish at any time when GC runs.

## Example
```js
// Map: any key type, ordered, size
const lastPrice = new Map();
lastPrice.set('INFY', 1500).set('TCS', 4000);
lastPrice.get('INFY');        // 1500
lastPrice.size;               // 2
for (const [symbol, price] of lastPrice) console.log(symbol, price);

// Set: uniqueness
const symbols = ['INFY', 'TCS', 'INFY', 'HDFC'];
const unique = [...new Set(symbols)]; // ['INFY', 'TCS', 'HDFC']

// WeakMap: data attached to DOM nodes without leaking them
const tooltipData = new WeakMap();
const row = document.querySelector('#row-infy');
tooltipData.set(row, { symbol: 'INFY', updatedAt: Date.now() });
row.remove();
// once no other code references `row`, the node AND its tooltip data can be collected
```

## When to use it
- `Map`: price cache keyed by symbol, LRU cache (insertion order makes it easy), counting occurrences.
- `Set`: unique symbols in a watchlist, tracking selected row ids, de-duplicating WebSocket messages.
- `WeakMap`: caching results per object (memoize by object), private data for objects, metadata for DOM nodes.
- `WeakSet`: marking objects as "already processed" without keeping them alive.

## Likely questions
### When would you use a Map instead of a plain object?
When keys are not strings (objects, numbers kept as numbers), when you add and remove keys often, when you need the size (`map.size`) or insertion order, or when keys come from users (a plain object has prototype keys like `constructor`, and `__proto__` can be a security issue). Objects are still fine for fixed, known shapes like a config or a JSON record.

### How do you remove duplicates from an array?
`[...new Set(arr)]`. It works for primitives. For objects, dedupe by a key: `[...new Map(items.map((i) => [i.id, i])).values()]`.

### Why does WeakMap help avoid memory leaks?
With a normal `Map`, the map holds a strong reference to its keys, so an object used as a key can never be garbage collected while the map lives, even after the rest of the app is done with it. A `WeakMap` holds keys weakly: when nothing else references the object, the GC can remove it and its value. So caching data per DOM node or per object does not keep dead objects alive.

### Why can't you iterate a WeakMap or get its size?
Because entries can disappear at any moment when the garbage collector runs, so the result would be unpredictable. The API only has `get`, `set`, `has`, `delete`.

### What is the difference between WeakMap and WeakRef?
`WeakMap` attaches data to an object weakly. `WeakRef` holds a weak reference to an object you might want to use later (`ref.deref()` may return `undefined`). WeakRef is rarely needed in app code.

## Common mistakes
- Using `map[key] = value` on a Map; that sets an object property, not a map entry. Use `map.set`.
- Expecting `new Set([{ id: 1 }, { id: 1 }])` to dedupe; they are two different objects.
- `JSON.stringify(map)` gives `{}`; convert with `Object.fromEntries(map)` first.

## Resources
- [javascript.info: Map and Set](https://javascript.info/map-set) - clear examples of both
- [javascript.info: WeakMap and WeakSet](https://javascript.info/weakmap-weakset) - use cases like caching
- [MDN: Map](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map) - includes Map vs Object table
- [MDN: WeakMap](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakMap) - why it is not enumerable
