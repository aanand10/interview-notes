# `Function.prototype.bind`

> **In one line:** `bind` returns a new function that always runs with a fixed `this` and some pre-filled arguments, except when it is called with `new`, where the fixed `this` is ignored and a new object is built instead.

## Requirements

What the real [`bind`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind), [`call`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/call) and [`apply`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/apply) do:

- `fn.call(ctx, a, b)` runs `fn` **now** with `this = ctx` and arguments listed one by one.
- `fn.apply(ctx, [a, b])` is the same, but the arguments come as an array.
- `fn.bind(ctx, a)` does **not** run `fn`. It **returns a new function**. When you call it later with `(b)`, it runs `fn` with `this = ctx` and arguments `(a, b)`. This is called partial application (pre-filling some arguments).
- `this` of a bound function can't be changed again: re-binding, `call` or `apply` on it won't change `this`.
- **With `new`**: `new boundFn(...)` ignores the bound `this`, creates a new instance of the original function, and still uses the pre-filled arguments. The result is `instanceof` both the original and the bound function.
- Throws `TypeError` if called on something that is not a function.

## Implementation

```js
Function.prototype.myCall = function (context, ...args) {
  if (typeof this !== 'function') throw new TypeError('myCall must be called on a function');
  // null/undefined -> globalThis (sloppy-mode behaviour), primitives -> wrapper object
  const ctx = context == null ? globalThis : Object(context);
  const key = Symbol('fn');          // unique key, can't clash with real props
  ctx[key] = this;                   // `this` is the function being called
  try {
    return ctx[key](...args);        // called as a method, so this === ctx
  } finally {
    delete ctx[key];                 // clean up even if fn throws
  }
};

Function.prototype.myApply = function (context, argsArray) {
  return this.myCall(context, ...(argsArray ?? []));
};

Function.prototype.myBind = function (context, ...boundArgs) {
  if (typeof this !== 'function') throw new TypeError('Bind must be called on a function');
  const originalFn = this;

  function boundFn(...callArgs) {
    // Called with `new`? Ignore `context` and construct a real instance.
    if (new.target) {
      return Reflect.construct(originalFn, [...boundArgs, ...callArgs], new.target);
    }
    // Normal call: fixed this + pre-filled args + new args
    return originalFn.apply(context, [...boundArgs, ...callArgs]);
  }

  // So `new boundFn()` objects inherit the original prototype (instanceof works)
  if (originalFn.prototype) {
    boundFn.prototype = Object.create(originalFn.prototype);
  }
  return boundFn;
};
```

Key ideas to say out loud:
- `call` trick: a function called as `obj.fn()` gets `this = obj`. So I temporarily put the function on the object under a [Symbol](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol) key, call it, then delete it.
- `bind` is a closure: it remembers `originalFn`, `context` and `boundArgs`.
- [`new.target`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/new.target) is `undefined` in a normal call and points at the function in a `new` call. [`Reflect.construct`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Reflect/construct) builds the object like `new` would, and also works for `class` constructors (which you cannot `.apply`).

Older ES5-style version of the `new` check, if the interviewer asks for no `Reflect`:

```js
function boundFn(...callArgs) {
  const isNew = this instanceof boundFn;   // `new` makes `this` an instance of boundFn
  return originalFn.apply(isNew ? this : context, [...boundArgs, ...callArgs]);
}
```

## Edge cases

- **Partial arguments** - `boundArgs` are put before `callArgs`, so `fn.bind(ctx, 1)(2)` calls `fn(1, 2)`.
- **Used with `new`** - `new.target` is set, so we construct a new instance and ignore `context`. `boundFn.prototype` inherits from the original, so `instanceof` works for both.
- **Re-binding** - `bound.myBind(other)()` still uses the first `context`, because the inner function calls `originalFn.apply(context, ...)` with the first context. Same as native.
- **`null` / `undefined` context in `call`** - falls back to `globalThis` (sloppy-mode rule). Primitives are boxed with `Object(context)`.
- **Property clash in `call`** - a `Symbol` key can't overwrite an existing property, and `finally` deletes it even if the function throws.
- **Called on a non-function** - throws `TypeError`.
- **Arrow functions** - binding an arrow does nothing to `this` (arrows ignore it), only partial args work. Same as native.

## Test it

```js
const account = { name: 'Anand', balance: 5000 };
function describe(currency, suffix) { return `${this.name}: ${currency}${this.balance}${suffix ?? ''}`; }

console.log(describe.myCall(account, '₹', '!'));   // Anand: ₹5000!
console.log(describe.myApply(account, ['$']));     // Anand: $5000
const bound = describe.myBind(account, '₹');
console.log(bound(' only'));                        // Anand: ₹5000 only   (partial args)
console.log(bound.myBind({ name: 'Other', balance: 1 })()); // Anand: ₹5000 (can't re-bind)

function Order(symbol, qty) { this.symbol = symbol; this.qty = qty; }
Order.prototype.total = function (p) { return this.qty * p; };
const BuyTCS = Order.myBind({ ignored: true }, 'TCS');
const o = new BuyTCS(10);
console.log(o, o instanceof Order, o instanceof BuyTCS, o.total(3));
// Order { symbol: 'TCS', qty: 10 } true true 30   (bound this ignored with new)

class Position { constructor(sym, qty) { this.sym = sym; this.qty = qty; } }
const P = Position.myBind(null, 'INFY');
console.log(new P(5));                              // Position { sym: 'INFY', qty: 5 }

console.log(Math.max.myApply(null, [3, 9, 2]));     // 9
```

Outputs checked with Node 24. Native `bind` gives the same results.

## Likely follow-ups

### What is the difference between call, apply and bind?
`call` and `apply` run the function immediately; the only difference is that `call` takes arguments one by one and `apply` takes an array. `bind` does not run anything; it returns a new function with `this` and some arguments locked in, which you call later. A memory trick: **a**pply = **a**rray.

### Why do we need bind in real code?
When you pass a method as a callback, it loses its object. `setTimeout(order.submit, 0)` calls `submit` with `this` undefined. `order.submit.bind(order)` fixes it. Today we often use an arrow function instead: `() => order.submit()`.

### What happens if you call a bound function with `new`?
The bound `this` is ignored. A fresh object is created from the original constructor, and the pre-filled arguments are still used. My polyfill detects this with `new.target` and uses `Reflect.construct`.

### Can you bind a function twice?
You can call `bind` again, but `this` stays the first one. Extra arguments do get added, so `fn.bind(a, 1).bind(b, 2)()` runs `fn` with `this = a` and args `(1, 2)`.

### How do you implement call without using apply or bind?
Attach the function to the context object under a unique Symbol key, call it as a method so `this` is set, store the result, delete the key, and return the result. That is my `myCall` above.

### What does `bind` do to `name` and `length`?
Native bound functions are named `"bound describe"` and their `length` is reduced by the number of pre-filled args. My polyfill does not copy that; you can add it with `Object.defineProperty(boundFn, 'name', { value: 'bound ' + originalFn.name })` if asked.

## Common mistakes

- Returning the result of `fn.apply(...)` from `bind` instead of returning a function.
- Forgetting partial arguments, or putting them after the call-time arguments.
- Writing `myBind` as an arrow function, so `this` is not the original function.
- Ignoring the `new` case. It is the main follow-up interviewers ask about.
- Using a plain string key like `ctx.fn = this` in `call`, which can overwrite a real `fn` property.

## Resources

- [MDN: Function.prototype.bind](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind) - bound functions and the `new` behaviour
- [MDN: Function.prototype.call](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/call) - how `this` is set
- [javascript.info: Function binding](https://javascript.info/bind) - losing `this` and partial functions, explained simply
- [javascript.info: Decorators and forwarding, call/apply](https://javascript.info/call-apply-decorators) - call vs apply in practice
