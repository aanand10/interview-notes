# interface vs type

> **In one line:** Both describe the shape of an object, but an `interface` can be extended and re-opened (declaration merging), while a `type` alias can name anything, including unions, tuples and mapped types, so I use `type` by default for data and unions, and `interface` for public object contracts that others may extend.

## Key points
- **Same job for plain objects.** For `{ id: string; qty: number }` both work the same, and a class can `implements` either one. The [TypeScript handbook](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html) says the choice is mostly personal preference.
- **Extending.** An interface uses `extends`. A type uses an intersection `&`. If a property clashes, `extends` gives a clear error right away, while `&` quietly creates `never` for that property.
- **Merging.** Two `interface User` declarations with the same name merge into one ([declaration merging](https://www.typescriptlang.org/docs/handbook/declaration-merging.html)). Two `type User` declarations are an error ("Duplicate identifier"). Merging is how you add fields to `Window` or to a library's types.
- **Only `type` can do unions and more.** Unions (`'BUY' | 'SELL'`), tuples (`[number, number]`), primitives aliases, mapped types and conditional types all need `type`. An interface can only describe an object shape.
- **Small difference:** a `type` object is assignable to `Record<string, string>`, an interface is not (interfaces have no implicit index signature, because they could be merged later).

## Example
```ts
// 1. Same shape, two ways
interface Order { id: string; symbol: string; qty: number }
type OrderT = { id: string; symbol: string; qty: number };

// 2. Extending
interface LimitOrder extends Order { price: number }
type LimitOrderT = OrderT & { price: number };

// 3. Declaration merging (interfaces only)
interface User { id: string }
interface User { name: string }          // merged: { id; name }
const u: User = { id: "1", name: "Anand" };

type Account = { id: string };
// type Account = { name: string };      // Error: Duplicate identifier 'Account'

// 4. Things only `type` can express
type Side = "BUY" | "SELL";                          // union
type PricePoint = [time: number, price: number];     // tuple
type OrderStatus =                                    // discriminated union
  | { kind: "open" }
  | { kind: "filled"; avgPrice: number }
  | { kind: "rejected"; reason: string };

// 5. Clash: extends errors, & silently gives never
interface A { x: string }
// interface B extends A { x: number } // Error: types of property 'x' are incompatible
type C = A & { x: number };            // no error, but C["x"] is never
```

## When to use it
- **`type`** for API response shapes, props, unions like order status (`open | filled | rejected`), tuples for chart points `[time, price]`, and anything built with utility or mapped types.
- **`interface`** when I want something to be extended or augmented: a shared library contract, a class contract, or adding a global like `window.__APP_VERSION__`.

```ts
// Augment a global with interface merging
declare global {
  interface Window { __APP_VERSION__: string }
}
```

## Likely questions
### What is the difference between `interface` and `type`?
For object shapes they are almost the same. The real differences are: interfaces can be merged by declaring them twice, types cannot; types can describe unions, tuples, primitives, mapped and conditional types, interfaces cannot; and for extending, interfaces use `extends` while types use `&`. With `extends`, a conflicting property is an error, with `&` it silently becomes `never`.

### How do you extend each one? Can they mix?
An interface uses `interface B extends A {}`. A type uses `type B = A & { ... }`. They mix fine: an interface can extend a type alias as long as that type is an object shape, and a type can intersect an interface.

```ts
type Base = { ts: number };
interface Tick extends Base { price: number }   // OK
type Tick2 = Tick & { volume: number };          // OK
```

### What is declaration merging and when is it useful?
If I declare the same interface name twice in the same scope, TypeScript merges the members into one interface. It is useful to augment third-party or global types, for example adding a field to `Window`, or adding fields to SvelteKit's `App.Locals` in `app.d.ts`. The downside is that it can happen by accident if two files use the same global name.

### Can an interface describe a union?
No. A union like `type Side = 'BUY' | 'SELL'` or a discriminated union of order states must be a `type`. An interface can be one member of a union, though: `type Shape = Circle | Square` where both are interfaces.

### Which do you prefer and why?
I use `type` by default because it handles everything I need for app data: unions, tuples, and derived types with `Pick`, `Omit` or mapped types, and the code base stays consistent. I switch to `interface` when I write a public contract that others should extend or when I need merging, like augmenting globals. Many teams pick the opposite rule ("interface for objects, type for the rest"); the key is to be consistent, and I would follow the team's lint rule (`@typescript-eslint/consistent-type-definitions`).

### Is there a performance difference?
For big code bases the TypeScript team has said that `interface extends` can be checked a bit faster than large intersections, because interfaces are cached by name. For a normal frontend app this does not matter; readability matters more.

## Common mistakes
- Thinking interfaces cannot be extended by types or vice versa. They interoperate.
- Using `&` to "override" a property type. It does not override, it intersects, and you get `never`.
- Accidentally merging two global interfaces with the same name in different files.
- Trying to pass an interface value where `Record<string, unknown>` is expected and being surprised by the "index signature is missing" error.

## Resources
- [TS Handbook: Differences between type aliases and interfaces](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html) - official comparison table
- [TS Handbook: Object types](https://www.typescriptlang.org/docs/handbook/2/objects.html) - extends, intersections, optional and readonly props
- [TS Handbook: Declaration merging](https://www.typescriptlang.org/docs/handbook/declaration-merging.html) - how interface merging works
