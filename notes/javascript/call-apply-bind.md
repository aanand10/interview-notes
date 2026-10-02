# call, apply, bind

> **In one line:** All three let you choose what `this` is for a function; `call` and `apply` run the function right away (args as a list vs as an array), and `bind` returns a new function with `this` and any leading args locked in for later.

## Key points
- [`fn.call(thisArg, a, b)`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/call): runs now, arguments passed one by one.
- [`fn.apply(thisArg, [a, b])`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/apply): runs now, arguments passed as an array (or array-like). Memory trick: **A**pply = **A**rray, **C**all = **C**omma.
- [`fn.bind(thisArg, a)`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind): does **not** run. It returns a new "bound" function. Its `this` cannot be changed again by `call`, `apply` or another `bind`; only `new` overrides it.
- Bound args are **partial application**: you pre-fill the first arguments.
- None of these change `this` for arrow functions. Arrows keep their lexical `this`.

## Example
```js
const account = { owner: 'Asha' };
function describe(currency, amount) {
  return `${this.owner}: ${currency} ${amount}`;
}

describe.call(account, 'INR', 500);     // "Asha: INR 500"
describe.apply(account, ['INR', 500]);  // "Asha: INR 500"

const inINR = describe.bind(account, 'INR'); // nothing runs yet
inINR(500);                             // "Asha: INR 500"
```

## When to use it
- **bind:** passing a class method as a callback, for example `socket.onmessage = this.handleTick.bind(this)`. Also to pre-fill args: `const gst = tax.bind(null, 0.18)`.
- **call:** borrowing a method from another object or prototype, or calling a parent constructor in old-style inheritance (`Animal.call(this, name)`).
- **apply:** when your arguments are already in an array. Today spread (`fn(...args)`) usually replaces it, but `fn.apply(this, args)` is still common inside wrappers like debounce.

## Likely questions

### What is the difference between call, apply and bind?
`call` and `apply` invoke the function immediately with a given `this`. The only difference is how you pass arguments: a comma list for `call`, an array for `apply`. `bind` does not invoke; it returns a new function with `this` (and optionally some arguments) fixed, which you call later.

### When would you use each?
Use `call` to borrow a method or chain constructors. Use `apply` when args are in an array (for example inside a generic wrapper that forwards `args`). Use `bind` when you need to hand a function to someone else (an event listener, timer, or promise) and keep the right `this`.

### What is method borrowing?
Using a method from one object on another object that does not have it.
```js
function sumArgs() {
  // arguments is array-like but has no reduce, so borrow it from Array.prototype
  return Array.prototype.reduce.call(arguments, (a, b) => a + b, 0);
}
sumArgs(1, 2, 3);                                         // 6
Array.prototype.slice.call({ 0: 'a', 1: 'b', length: 2 }); // ['a', 'b']
Object.prototype.toString.call([]);                       // "[object Array]"
Math.max.apply(null, [3, 9, 2]);                          // 9 (same as Math.max(...[3, 9, 2]))
```

### How does bind do partial application?
Any arguments after `thisArg` are stored and placed before the arguments given at call time.
```js
function tax(rate, amount) { return +(amount * rate).toFixed(2); }
const gst = tax.bind(null, 0.18); // rate is fixed
gst(100); // 18
```
Pass `null` when the function does not use `this`.

### Write a polyfill for bind
Requirements: return a new function; fix `this`; prepend bound args; if the bound function is called with `new`, ignore the bound `this` and build a new instance whose prototype chain includes the original function's prototype.
```js
Function.prototype.myBind = function (thisArg, ...boundArgs) {
  if (typeof this !== 'function') {
    throw new TypeError('myBind must be called on a function');
  }
  const fn = this; // the original function

  function bound(...callArgs) {
    // If called with new, `this` is an instance of bound -> respect it
    const isNew = this instanceof bound;
    return fn.apply(isNew ? this : thisArg, [...boundArgs, ...callArgs]);
  }

  // Keep instanceof working for `new bound()` (arrow functions have no prototype)
  if (fn.prototype) bound.prototype = Object.create(fn.prototype);
  return bound;
};
```
Test it (output verified with node):
```js
const acct = { owner: 'Asha' };
function describe(currency, amount) { return `${this.owner}: ${currency} ${amount}`; }
const d = describe.myBind(acct, 'INR');
console.log(d(500));                              // Asha: INR 500
console.log(d.myBind({ owner: 'Other' })(10));    // Asha: INR 10  (first bind wins)

function Point(x, y) { this.x = x; this.y = y; }
const P = Point.myBind(null, 1);
const p = new P(2);
console.log(p.x, p.y, p instanceof Point);        // 1 2 true

try { Function.prototype.myBind.call(42); }
catch (e) { console.log(e.message); }             // myBind must be called on a function
```
Edge cases to mention: the `new` case, non-function callers, args merging order, and that `this instanceof bound` can be fooled in rare cases. The native version can detect `new` precisely, has `name` `"bound describe"`, and sets `length` to the remaining parameter count. Mention these as limits rather than coding them.

### Can you write call and apply polyfills too?
Attach the function to the target object under a unique key, call it as a method (implicit binding), then delete the key.
```js
Function.prototype.myCall = function (ctx, ...args) {
  ctx = ctx ?? globalThis;          // null/undefined -> global object (sloppy-mode behaviour)
  const key = Symbol('fn');         // Symbol avoids clashing with real properties
  ctx[key] = this;
  const result = ctx[key](...args);
  delete ctx[key];
  return result;
};
```
`myApply` is the same but takes `args` as an array. Note: primitives like `5` would need `Object(ctx)`.

## Common mistakes
- Writing `el.addEventListener('click', this.onClick.bind(this))` and later trying to remove it with another `.bind(this)`. Each `bind` returns a **new** function, so `removeEventListener` gets a different reference. Store the bound function once.
- Expecting `bind`/`call` to change `this` of an arrow function.
- Calling `bind` twice and expecting the second `this` to win.
- Forgetting that `bind` does not run the function.

## Resources
- [MDN: Function.prototype.bind](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind) - bound functions, partial application, new behaviour
- [MDN: Function.prototype.call](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/call) - call semantics
- [MDN: Function.prototype.apply](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/apply) - array arguments
- [javascript.info: Decorators and forwarding, call/apply](https://javascript.info/call-apply-decorators) - method borrowing and wrappers
