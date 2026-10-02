# `this` keyword

> **In one line:** For a normal function, `this` is decided by *how the function is called*, not where it is written; arrow functions are the exception and take `this` from the surrounding code.

## Key points
- [`this`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this) is set at call time. Look at the call site, the part to the left of `(`.
- Four rules, from highest to lowest priority:
  1. **`new` binding:** `new Fn()` sets `this` to the brand new object.
  2. **Explicit binding:** `fn.call(obj)`, `fn.apply(obj)`, `fn.bind(obj)` set `this` to `obj`.
  3. **Implicit binding:** `obj.fn()` sets `this` to `obj` (the thing before the dot).
  4. **Default binding:** plain `fn()` gives `undefined` in strict mode (modules, classes, Svelte), or the global object (`window`) in sloppy mode.
- **Arrow functions** ignore all four rules. They use `this` from the enclosing scope at the time they were created (lexical `this`).
- Losing the dot loses `this`. Passing `obj.fn` as a callback is the same as calling `fn()` plainly.

## Example
Run as an ES module (strict mode), like code in a Svelte or Vite app. Output verified with node.
```js
const user = {
  name: 'Asha',
  greet() { return this?.name; },
  nested() { function inner() { return this; } return inner(); },
  nestedArrow() { const inner = () => this.name; return inner(); },
  arrowMethod: () => typeof this,
};

console.log(user.greet());        // Asha       -> implicit: called with user.
const g = user.greet;
console.log(g());                 // undefined  -> plain call, strict mode: this is undefined
console.log(user.nested());       // undefined  -> inner() is a plain call
console.log(user.nestedArrow());  // Asha       -> arrow takes this from nestedArrow
console.log(user.arrowMethod());  // undefined  -> arrow took this from module top level
console.log(g.call({ name: 'Ravi' })); // Ravi  -> explicit binding
```

## When to use it
- Class methods passed as callbacks (to `setInterval`, WebSocket `onmessage`, or event listeners) lose `this`. Use an arrow class field or `.bind` in the constructor.
- In Svelte 5 you rarely write `this`. You mostly see it in plain JS classes, for example a `PriceSocket` class or a class with `$state` fields used as a shared store.

## Likely questions

### Method call vs extracted function: what prints?
`user.greet()` prints `Asha` because the call has `user.` in front. `const g = user.greet; g()` is a plain call, so `this` is `undefined` in strict mode (and `this.name` would throw a TypeError without the `?.`). In sloppy browser code `this` would be `window`, so you would get `window.name`, often an empty string. The function is the same; only the call site changed.

### What is `this` inside a nested function in a method?
A normal nested function called as `inner()` uses default binding, so `this` is `undefined` (strict) or `window` (sloppy). It does not inherit the method's `this`. The fix is an arrow function (`const inner = () => this.name`), or the old `const self = this`.

### What is `this` inside a callback, like setTimeout?
```js
const user = {
  name: 'Asha',
  later() { setTimeout(function () { console.log(this.name); }, 0); },
  laterArrow() { setTimeout(() => console.log(this.name), 0); },
};
user.later();      // not "Asha": browser passes window as this, Node passes a Timeout object
user.laterArrow(); // Asha -> arrow captured this from laterArrow
```
The caller of the callback decides `this`, and timers do not know about `user`. Array methods like `map` accept a second `thisArg` argument for this reason.

### What about an arrow function used as an object method?
An object literal does not create a scope, so an arrow method takes `this` from the code around the object: `undefined` in a module, `window` in a classic script. So `arrowMethod: () => this.name` never sees the object. Use method shorthand `greet() {}` instead.

### How does `this` work in classes?
Inside class methods `this` is the instance when you call `obj.method()`. Class bodies are always strict, so an extracted method gets `undefined`.
```js
class BuyButton {
  label = 'Buy';
  handle() { return this?.label; }
  handleArrow = () => this.label; // arrow field: this fixed to the instance
}
const btn = new BuyButton();
const h = btn.handle, ha = btn.handleArrow;
console.log(h(), ha()); // undefined Buy
```
Arrow fields are created per instance (more memory) but are safe to pass around. Alternative: `this.handle = this.handle.bind(this)` in the constructor.

### What is `this` in a DOM event handler?
With `el.addEventListener('click', function () { ... })`, `this` is the element the listener is attached to (same as `event.currentTarget`). With an arrow function, `this` comes from outside, so use `event.currentTarget` instead. Inline HTML handlers like `onclick="..."` also get the element.

### Explain the four rules and their priority.
`new` beats explicit binding, explicit beats implicit, implicit beats default. Two proofs: `bind` cannot be overridden by another `bind` or `call` (the first bound `this` wins), but `new` on a bound function ignores the bound `this` and uses the new object.
```js
function show() { return this.name; }
const b1 = show.bind({ name: 'B1' });
console.log(b1.bind({ name: 'B2' })()); // B1

const Bound = function (n) { this.n = n; }.bind({ n: 'ignored' });
console.log(new Bound('new wins').n);    // new wins
```

## Common mistakes
- Using an arrow function as an object or prototype method and expecting `this` to be the object.
- Passing `this.method` to `setInterval`, `addEventListener` or a promise `.then` without binding.
- Assuming `this` is the function itself. It never is (unless you call it that way).
- Forgetting that modules and classes are strict, so the "default" `this` is `undefined`, not `window`.

## Resources
- [MDN: this](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this) - full rules including classes and callbacks
- [javascript.info: Object methods, "this"](https://javascript.info/object-methods) - beginner-friendly walkthrough
- [javascript.info: Function binding](https://javascript.info/bind) - losing this and fixing it with bind
- [MDN: Arrow functions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions) - lexical this
