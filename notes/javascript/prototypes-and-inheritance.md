# Prototypes and inheritance

> **In one line:** Every JavaScript object has a hidden link to another object called its prototype, and when a property is missing, the engine follows that link up the prototype chain; this is how objects share methods and how inheritance works, even with classes.

## Key points
- **Prototype chain:** `rex.speak()` looks on `rex`, then `Dog.prototype`, then `Animal.prototype`, then `Object.prototype`, then `null` (stop, `undefined`). See [MDN: inheritance and the prototype chain](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Inheritance_and_the_prototype_chain).
- **`__proto__` vs `prototype`:** `obj.__proto__` is the object's actual parent link (use [`Object.getPrototypeOf`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/getPrototypeOf) in real code). `Fn.prototype` is a property that only functions have; it is the object that becomes `__proto__` of anything created with `new Fn()`.
- **`Object.create(proto)`** makes a new empty object whose prototype is `proto`. `Object.create(null)` makes an object with no prototype at all (a clean dictionary).
- **Constructor functions** are normal functions called with `new`. Methods go on `Fn.prototype` so all instances share one copy.
- **ES6 classes** are mostly syntax on top of this same system.

## Example
Classic interview task: make `Dog` inherit from `Animal` without classes. Output verified with node.
```js
function Animal(name) {
  this.name = name;                         // own property, per instance
}
Animal.prototype.speak = function () {      // shared by all animals
  return `${this.name} makes a sound`;
};

function Dog(name, breed) {
  Animal.call(this, name);                  // 1. run parent constructor on this object
  this.breed = breed;
}
Dog.prototype = Object.create(Animal.prototype); // 2. link the prototypes
Dog.prototype.constructor = Dog;                 // 3. restore constructor pointer
Dog.prototype.speak = function () {              // 4. override, and call the parent
  return `${Animal.prototype.speak.call(this)} - woof`;
};

const rex = new Dog('Rex', 'Lab');
console.log(rex.speak());                         // Rex makes a sound - woof
console.log(rex instanceof Dog, rex instanceof Animal); // true true
console.log(Object.getPrototypeOf(rex) === Dog.prototype); // true
console.log(rex.hasOwnProperty('name'), rex.hasOwnProperty('speak')); // true false
```

## When to use it
- In daily code you write `class Dog extends Animal`, but knowing the chain helps debug "why is this method undefined", understand `instanceof`, and read library code.
- Shared methods on the prototype save memory when you create many objects, like thousands of `Trade` rows in a blotter.
- `Object.create(null)` is handy for lookup maps keyed by user input (no `toString` or `__proto__` keys to clash with), though `Map` is usually better.

## Likely questions

### What is the prototype chain?
Each object has an internal `[[Prototype]]` link. When you read a property, the engine checks the object itself first, then follows the link, and keeps going until it finds the property or reaches `null`. Writing a property always creates it on the object itself, not on the prototype. That is why `rex.hasOwnProperty('speak')` is false but `'speak' in rex` is true.

### What is the difference between `__proto__` and `prototype`?
`prototype` exists on functions (and classes). It is the blueprint for instances made with `new`. `__proto__` exists on every object and points to its actual parent. So `rex.__proto__ === Dog.prototype` is true, and `Dog.prototype.__proto__ === Animal.prototype` is true. An instance has no `prototype` property: `rex.prototype` is `undefined`. [`__proto__`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/proto) is a legacy accessor; prefer `Object.getPrototypeOf` and `Object.setPrototypeOf`.

### What does Object.create do?
[`Object.create(proto)`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/create) returns a new object whose prototype is `proto`, without running any constructor.
```js
const base = { greet() { return 'hi ' + this.n; } };
const kid = Object.create(base);
kid.n = 'K';
kid.greet(); // "hi K" -> found on base through the chain
```
We use it for inheritance because `Dog.prototype = Animal.prototype` would make them the **same** object (changing Dog would change Animal), and `Dog.prototype = new Animal()` would run the Animal constructor with no args.

### What does `new` do, in 4 steps?
1. Create a new empty object.
2. Set its prototype to `Constructor.prototype`.
3. Call the constructor with `this` set to that new object.
4. Return the new object, unless the constructor explicitly returns its own object (a returned primitive is ignored).
```js
function myNew(Ctor, ...args) {
  const obj = Object.create(Ctor.prototype);                 // steps 1 + 2
  const result = Ctor.apply(obj, args);                      // step 3
  const isObject = result !== null &&
    (typeof result === 'object' || typeof result === 'function');
  return isObject ? result : obj;                            // step 4
}
myNew(Dog, 'Bruno', 'Pug').speak(); // "Bruno makes a sound - woof"
```
Step 4 check: `function W() { this.a = 1; return { b: 2 }; }` gives `{ b: 2 }` with `new`, while `return 5` is ignored and gives `{ a: 1 }`.

### How does instanceof work? Can you write it?
[`a instanceof B`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/instanceof) walks up `a`'s prototype chain and returns true if any link equals `B.prototype`.
```js
function myInstanceOf(obj, Ctor) {
  let proto = Object.getPrototypeOf(obj);
  while (proto) {
    if (proto === Ctor.prototype) return true;
    proto = Object.getPrototypeOf(proto);
  }
  return false;
}
myInstanceOf(rex, Animal); // true
myInstanceOf(rex, Array);  // false
```
It checks the chain, not who constructed the object, so it can be fooled by changing prototypes. It also fails across iframes (each has its own `Array`); use `Array.isArray` for arrays.

### Why reset `Dog.prototype.constructor`?
`Object.create(Animal.prototype)` gives an object whose inherited `constructor` is `Animal`. Code that reads `rex.constructor` (or `new rex.constructor()`) would then get the wrong function. Setting it back to `Dog` keeps it correct.

### Why put methods on the prototype instead of inside the constructor?
`this.speak = function () {}` in the constructor creates a new function per instance. Methods on the prototype exist once and are shared, which uses less memory and lets you patch or override them in one place.

## Common mistakes
- Mixing up `prototype` (on functions) and `__proto__` (on all objects).
- `Dog.prototype = Animal.prototype`: no separate layer, overrides leak into Animal.
- Forgetting `Animal.call(this, name)`, so own fields like `name` are missing.
- Using `for...in` on instances and getting inherited enumerable props too. Use `Object.keys` or `hasOwnProperty`.
- Changing an object's prototype at runtime with `Object.setPrototypeOf`: it works but is slow in engines.

## Resources
- [MDN: Inheritance and the prototype chain](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Inheritance_and_the_prototype_chain) - the authoritative explanation
- [javascript.info: Prototypal inheritance](https://javascript.info/prototype-inheritance) - __proto__ and lookup rules
- [javascript.info: F.prototype](https://javascript.info/function-prototype) - prototype vs __proto__ for constructors
- [MDN: new operator](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/new) - the exact steps new performs
