# Tricky JS concepts interviewers love

> **In one line:** Most JS "gotchas" come from a few rules: objects are shared by reference, arrays can have holes, semicolons are sometimes inserted for you, numbers are floating point, and `Date` months start at 0.

## Key points
- JS always passes **by value**, but for objects the value is a **reference** (an address). So a function can change an object's insides but cannot replace the caller's variable.
- Arrays can be **sparse** (have empty slots, called holes). Many array methods skip holes, `for...of` does not.
- **ASI** (Automatic Semicolon Insertion) adds a semicolon after `return` if a new line follows. See [MDN: ASI](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Lexical_grammar#automatic_semicolon_insertion).
- Money must be stored as integers (paise), not floats.
- `JSON.stringify` silently drops `undefined`, functions and symbols in objects.

All outputs below were run with Node.

## Drills

### Q1. Pass by value vs reference
```js
function update(order) {
  order.qty = 10;        // changes the shared object
  order = { qty: 99 };   // only changes the local variable
}
const o = { qty: 1 };
update(o);
console.log(o.qty);

let n = 1;
function inc(x) { x++; }
inc(n);
console.log(n);

const a = [1, 2];
const b = a;
b.push(3);
console.log(a, a === [1, 2, 3]);
```
**Answer:**
```text
10
1
[ 1, 2, 3 ] false
```
The function gets a copy of the reference, so mutation is visible but reassignment is not. Primitives are copied. `b = a` shares one array. `===` on objects compares identity, not contents.

### Q2. Default parameters are evaluated on every call
```js
function addItem(item, list = []) {
  list.push(item);
  return list;
}
console.log(addItem("A"), addItem("B"));

const shared = [];
function addItem2(item, list = shared) {
  list.push(item);
  return list;
}
console.log(addItem2("A"), addItem2("B"));

let calls = 0;
function f(x = ++calls) { return x; }
f(); f(); f(5);
console.log(calls);
```
**Answer:**
```text
[ 'A' ] [ 'B' ]
[ 'A', 'B' ] [ 'A', 'B' ]
2
```
Unlike Python, JS creates a fresh `[]` each call. The trap is when the default points to an outside object (`shared`): every call mutates the same array. The default expression runs only when the argument is `undefined`, so `f(5)` did not increment.

### Q3. Closures in event handlers
```js
const buttons = [];
for (var i = 0; i < 3; i++) buttons.push({ onclick: () => console.log("clicked", i) });
buttons[0].onclick();

const btns2 = [];
for (let i = 0; i < 3; i++) btns2.push({ onclick: () => console.log("clicked", i) });
btns2[0].onclick();

let price = 100;
const handler = () => console.log("price is", price);
price = 105;
handler();
```
**Answer:**
```text
clicked 3
clicked 0
price is 105
```
A handler reads the variable when it runs, not when it was made. `var` shares one `i`. The reverse trap: a **stale closure** (in React) happens when a handler holds an old variable from a past render. In Svelte 5, handlers read `$state` live, so this is rarer.

### Q4. for...in vs for...of
```js
const arr = ["a", "b"];
arr.extra = "x";
Array.prototype.bad = 1; // someone patched the prototype
for (const k in arr) console.log("in:", k);
for (const v of arr) console.log("of:", v);
delete Array.prototype.bad;
for (const k in "hi") console.log("in str:", k);
```
**Answer:**
```text
in: 0
in: 1
in: extra
in: bad
of: a
of: b
in str: 0
in str: 1
```
`for...in` loops over **keys** (as strings), including extra and inherited enumerable properties. `for...of` loops over **values** of an iterable (arrays, strings, Map, Set). Use `for...of` for arrays and `Object.entries(obj)` for objects. See [MDN: for...of](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/for...of).

### Q5. Sparse arrays
```js
const sp = [1, , 3];
console.log(sp, sp.length, 1 in sp);
sp.forEach((x) => console.log("forEach", x));
console.log(sp.map((x) => x * 2));
for (const x of sp) console.log("for-of", x);
const big = [];
big[5] = "x";
console.log(big.length, big);
```
**Answer:**
```text
[ 1, <1 empty item>, 3 ] 3 false
forEach 1
forEach 3
[ 2, <1 empty item>, 6 ]
for-of 1
for-of undefined
for-of 3
6 [ <5 empty items>, 'x' ]
```
A hole is not the same as `undefined`: the index does not exist (`1 in sp` is false). `forEach` and `map` skip holes; `for...of` gives `undefined`. Setting index 5 makes `length` 6.

### Q6. delete on arrays
```js
const d = [1, 2, 3];
delete d[1];
console.log(d, d.length, d[1]);

const s2 = [1, 2, 3];
s2.splice(1, 1);
console.log(s2, s2.length);

const t = [1, 2, 3, 4];
t.length = 2;
console.log(t);
```
**Answer:**
```text
[ 1, <1 empty item>, 3 ] 3 undefined
[ 1, 3 ] 2
[ 1, 2 ]
```
`delete` leaves a hole and keeps the length. Use `splice` (mutates) or `filter`/`toSpliced` (new array). Setting `length` smaller cuts the array.

### Q7. Array(3) vs [3]
```js
console.log(Array(3), Array(3).length, [3], Array(3, 4), Array.of(3), Array.from({ length: 3 }, (_, i) => i));
console.log(Array(3).map(() => 0), Array(3).fill(0));
try { Array(-1); } catch (e) { console.log(e.name + ": " + e.message); }
```
**Answer:**
```text
[ <3 empty items> ] 3 [ 3 ] [ 3, 4 ] [ 3 ] [ 0, 1, 2 ]
[ <3 empty items> ] [ 0, 0, 0 ]
RangeError: Invalid array length
```
With one number, `Array(n)` makes `n` holes. With other arguments it makes those items. `map` skips holes, so it does nothing; use `fill` or `Array.from`. `Array.of` always makes items.

### Q8. ASI: return on its own line
```js
function getConfig() {
  return
  {
    theme: "dark";
  }
}
console.log(getConfig());
```
**Answer:**
```text
undefined
```
JS inserts a semicolon right after `return`, so the function returns `undefined`. The `{ ... }` below becomes an unreachable block with a label `theme:`. Always start the returned value on the same line as `return`.

### Q9. ASI: line starting with [ or (, and labels
```js
const x1 = 1
const y1 = x1
;[1, 2].forEach((v) => console.log("asi", v))

outer: for (let i = 0; i < 3; i++) {
  for (let j = 0; j < 3; j++) {
    if (j === 1) continue outer;
    if (i === 2) break outer;
    console.log("label", i, j);
  }
}
```
**Answer:**
```text
asi 1
asi 2
label 0 0
label 1 0
```
Without the leading `;`, the code would parse as `x1[1, 2].forEach(...)` and crash. In no-semicolon style, put `;` before lines that start with `[`, `(` or a template literal. A **label** (`outer:`) lets `break`/`continue` target an outer loop.

### Q10. Floating point and money
```js
console.log(0.1 * 3, 1.005 * 100, Math.round(1.005 * 100) / 100);

const pricePaise = 10050; // Rs 100.50 stored as integer paise
const qty = 3;
const totalPaise = pricePaise * qty;
console.log(totalPaise, (totalPaise / 100).toFixed(2));
console.log(new Intl.NumberFormat("en-IN", { style: "currency", currency: "INR" }).format(totalPaise / 100));
console.log((1.005).toFixed(2), (0.1 + 0.2).toFixed(2));
```
**Answer:**
```text
0.30000000000000004 100.49999999999999 1
30150 301.50
₹301.50
1.00 0.30
```
Decimal fractions are not exact in binary, so even rounding goes wrong (`1.005` rounds to `1.00`). Do all math in integer paise, convert only for display with [Intl.NumberFormat](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/NumberFormat). For very large values use `BigInt` or a decimal library, and let the backend be the source of truth.

### Q11. Date month index
```js
const dt = new Date(2026, 9, 6);
console.log(dt.getMonth(), dt.getDate(), dt.toDateString());
console.log(new Date(2026, 1, 30).toDateString());
console.log(new Date(2026, 0, 31).getDay(), dt.getFullYear());
```
**Answer:**
```text
9 6 Tue Oct 06 2026
Mon Mar 02 2026
6 2026
```
Months are 0-based (9 is October) but days are 1-based. Invalid dates **roll over** silently: Feb 30 becomes Mar 2. `getDay()` is the weekday (0 is Sunday, 6 is Saturday), `getDate()` is the day of the month. Local-time constructors depend on the machine's timezone, which matters for market hours (use explicit timezones, for example `Asia/Kolkata`, with `Intl.DateTimeFormat`).

### Q12. JSON.stringify drops things
```js
console.log(JSON.stringify({
  a: undefined, b: () => 1, c: Symbol("s"), d: null, e: NaN, f: new Date(0), g: [undefined, () => 1],
}));
console.log(JSON.stringify(undefined), JSON.stringify(new Map([["k", 1]])));
try { JSON.stringify({ big: 10n }); } catch (e) { console.log(e.name + ": " + e.message); }
const c = { name: "c" };
c.self = c;
try { JSON.stringify(c); } catch (e) { console.log(e.name + ": " + e.message.split("\n")[0]); }
const back = JSON.parse(JSON.stringify({ when: new Date(0) }));
console.log(back.when, typeof back.when);
```
**Answer:**
```text
{"d":null,"e":null,"f":"1970-01-01T00:00:00.000Z","g":[null,null]}
undefined {}
TypeError: Do not know how to serialize a BigInt
TypeError: Converting circular structure to JSON
1970-01-01T00:00:00.000Z string
```
In objects, `undefined`, functions and symbols are dropped. In arrays they become `null`. `NaN` becomes `null`. Dates become strings and do not come back as Dates. `Map`/`Set` become `{}`. BigInt and circular references throw. That is why `JSON.parse(JSON.stringify(x))` is a bad deep clone; use `structuredClone`. See [MDN: JSON.stringify](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/stringify).

## When it matters in real code
- Order form: totals in paise, display with `Intl.NumberFormat`.
- Sending a form payload: optional fields set to `undefined` disappear from the JSON. Fine for PATCH, a surprise if the backend expects the key.
- Chart date axes: month off-by-one bugs from `new Date(y, m, d)`.

## Common mistakes
- Saying "JS passes objects by reference". More exact: it passes a reference by value.
- Using `delete arr[i]` to remove items.
- `new Array(n).map(...)` and expecting it to run.

## Resources
- [javascript.info: Object references and copying](https://javascript.info/object-copy) - reference vs value, shallow vs deep copy
- [MDN: Indexed collections (sparse arrays)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Indexed_collections#sparse_arrays) - which methods skip holes
- [MDN: Date](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date) - month index and rollover behaviour
- [MDN: JSON.stringify](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/stringify) - what gets dropped or converted
