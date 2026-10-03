# Proxy and Reflect

> **In one line:** A `Proxy` wraps an object and lets you intercept operations on it, like reading or writing a property, and `Reflect` gives you the default behaviour of those operations; reactivity systems such as Svelte 5's deep `$state` use proxies to know when data is read and changed.

## Key points
- `new Proxy(target, handler)`: the **handler** has **traps** such as `get`, `set`, `has`, `deleteProperty`, `apply` (function calls) and `construct` (`new`).
- If a trap is missing, the operation goes straight to the target as normal.
- [`Reflect`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Reflect) has a method for every trap (`Reflect.get`, `Reflect.set`, ...). Use it inside traps to do "the normal thing" correctly (it handles getters and the `receiver`, and `Reflect.set` returns `true`/`false`).
- The `set` trap must return `true` on success; returning `false` throws a TypeError in strict mode.
- Uses: validation, logging, default values, read-only views, and **reactivity**.

## Example
Logging and validation (checked with node):

```js
const order = new Proxy({ qty: 1 }, {
  get(target, key, receiver) {
    console.log('read', key);
    return Reflect.get(target, key, receiver);
  },
  set(target, key, value, receiver) {
    if (key === 'qty' && (!Number.isInteger(value) || value <= 0)) {
      throw new TypeError('qty must be a positive integer');
    }
    console.log('write', key, value);
    return Reflect.set(target, key, value, receiver);
  },
});

order.qty;        // read qty
order.qty = 5;    // write qty 5
// order.qty = -1 -> TypeError: qty must be a positive integer
```

A tiny reactivity system (the idea behind Vue 3 and Svelte 5 deep state):

```js
let activeEffect = null;
const subscribers = new Map(); // key -> Set of effects

function reactive(obj) {
  return new Proxy(obj, {
    get(t, key, r) {
      if (activeEffect) {
        if (!subscribers.has(key)) subscribers.set(key, new Set());
        subscribers.get(key).add(activeEffect);   // track: who read this key
      }
      return Reflect.get(t, key, r);
    },
    set(t, key, value, r) {
      const ok = Reflect.set(t, key, value, r);
      subscribers.get(key)?.forEach((fn) => fn()); // trigger: re-run readers
      return ok;
    },
  });
}

function effect(fn) { activeEffect = fn; fn(); activeEffect = null; }

const quote = reactive({ price: 100 });
effect(() => console.log('Price is', quote.price)); // Price is 100
quote.price = 101;                                  // Price is 101
```

## When to use it
- Understanding how Svelte 5 works: with `let cart = $state({ items: [] })`, `cart.items.push(x)` updates the UI because `cart` is a deep proxy that notices the change.
- Form/order objects that validate on write.
- Dev-only warnings when code reads a property that does not exist.

## Likely questions
### How does a Proxy work?
You give it a target object and a handler. Every operation on the proxy (get, set, delete, `in`, function call) checks the handler for a matching trap. If there is one, your code runs and decides what happens; if not, the operation is forwarded to the target. The original object is unchanged.

### Why use Reflect inside traps?
`Reflect.get(target, key, receiver)` does exactly what the default operation would do, including calling getters with the correct `this`. It also returns a boolean for `set`, which is what the trap must return. It is cleaner and more correct than `target[key] = value`.

### How do proxies relate to Svelte 5 reactivity?
In Svelte 5, `$state` on a plain object or array returns a **deep reactive proxy**. Reading a property inside a template, `$derived` or `$effect` records a dependency on that property; writing to it (even `arr.push()` or `obj.a.b = 1`) notifies only the things that read it. That is why mutation works in Svelte 5, while Svelte 4 needed reassignment (`arr = [...arr, x]`). Class instances and primitives are not proxied; `$state.raw` skips the proxy for large data you replace instead of mutate, which saves work for things like big candle arrays.

### Any downsides of proxies?
A small overhead on each access, identity issues (`proxy !== original`, so comparing with the raw object fails), and some built-ins like `Map`, `Set` or `Date` do not work through a plain proxy because their methods need the real object as `this`. Svelte provides `SvelteMap`, `SvelteSet` and `SvelteDate` in `svelte/reactivity` for this, and `$state.snapshot()` to get a plain copy (e.g. before `structuredClone` or logging).

## Resources
- [javascript.info: Proxy and Reflect](https://javascript.info/proxy) - every trap with examples
- [MDN: Proxy](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy) - reference and handler list
- [Svelte docs: $state](https://svelte.dev/docs/svelte/$state) - deep state proxies, `$state.raw`, `$state.snapshot`
