# Generics

> **In one line:** Generics are type parameters, like function arguments but for types, so one function or component can work with many types and still keep the exact type flowing from input to output.

## Key points
- A [generic](https://www.typescriptlang.org/docs/handbook/2/generics.html) `<T>` is a placeholder type. TypeScript usually **infers** it from the arguments, so callers rarely write `<T>` themselves.
- **Constraints** with `extends` limit what `T` can be: `<T extends { id: string }>` means "any type, as long as it has an `id`".
- [`keyof T`](https://www.typescriptlang.org/docs/handbook/2/keyof-types.html) is the union of a type's keys. `K extends keyof T` plus `T[K]` (an [indexed access type](https://www.typescriptlang.org/docs/handbook/2/indexed-access-types.html)) lets you write a type-safe "get property" or "sort by key".
- You can give a default: `<T = unknown>`. Use as few type parameters as you need; if `T` is used only once, you probably do not need it.
- In Svelte 5, a component becomes generic with `<script lang="ts" generics="T">`.

## Example
```ts
// 1. Basic: keep the element type
function first<T>(arr: T[]): T | undefined {
  return arr[0];
}
const n = first([1, 2, 3]);         // n: number | undefined
const s = first(["TCS", "INFY"]);   // s: string | undefined

// 2. Constraint with extends
function longest<T extends { length: number }>(a: T, b: T): T {
  return a.length >= b.length ? a : b;
}
longest("ab", "abc");     // OK, strings have length
// longest(1, 2);         // Error: number has no 'length'

// 3. keyof + indexed access: type-safe property getter
function getProp<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
const quote = { symbol: "TCS", ltp: 4123.5 };
const ltp = getProp(quote, "ltp");   // ltp: number
// getProp(quote, "price");          // Error: "price" is not "symbol" | "ltp"

// 4. Generic fetch helper
async function fetchJson<T>(url: string, init?: RequestInit): Promise<T> {
  const res = await fetch(url, init);
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return res.json() as Promise<T>;   // trust boundary: see "Typing API responses"
}
type Holding = { symbol: string; qty: number; avgPrice: number };
const holdings = await fetchJson<Holding[]>("/api/holdings");  // Holding[]
```

## When to use it
- Reusable data components: a `Table<T>` for holdings, orders and watchlists.
- Helpers: `fetchJson<T>`, `groupBy<T, K>`, `sortBy<T>(rows, key)`.
- Stores or state classes: a `createCache<T>()` for quotes or user settings.

## Likely questions
### Write a generic `useFetch<T>` (or Svelte equivalent).
In Svelte 5 I would write a small function in a `.svelte.ts` file that returns reactive state. The caller passes `T` once and gets typed `data`.

```ts
// fetcher.svelte.ts
export function createFetch<T>(url: () => string) {
  let data = $state<T | null>(null);
  let error = $state<string | null>(null);
  let loading = $state(false);

  $effect(() => {
    const controller = new AbortController();
    loading = true;
    error = null;
    fetch(url(), { signal: controller.signal })
      .then((r) => (r.ok ? (r.json() as Promise<T>) : Promise.reject(new Error(`HTTP ${r.status}`))))
      .then((json) => (data = json))
      .catch((e) => { if (e.name !== "AbortError") error = e.message; })
      .finally(() => (loading = false));
    return () => controller.abort();   // cancel when url changes or component unmounts
  });

  return {
    get data() { return data; },
    get error() { return error; },
    get loading() { return loading; },
  };
}
// usage in a component: const q = createFetch<Quote>(() => `/api/quote/${symbol}`);
```

### Write a generic `Table<T>` component.
The table does not know the row type. It takes `rows: T[]` and `columns` whose `key` must be a real key of `T`, so typos fail at compile time.

```svelte
<!-- Table.svelte -->
<script lang="ts" generics="T extends { id: string | number }">
  import type { Snippet } from "svelte";

  interface Column { key: keyof T; label: string }
  interface Props {
    rows: T[];
    columns: Column[];
    cell?: Snippet<[T, keyof T]>;     // optional custom cell renderer
    onRowClick?: (row: T) => void;
  }
  let { rows, columns, cell, onRowClick }: Props = $props();
</script>

<table>
  <thead><tr>{#each columns as c}<th>{c.label}</th>{/each}</tr></thead>
  <tbody>
    {#each rows as row (row.id)}
      <tr onclick={() => onRowClick?.(row)}>
        {#each columns as c}
          <td>{#if cell}{@render cell(row, c.key)}{:else}{String(row[c.key])}{/if}</td>
        {/each}
      </tr>
    {/each}
  </tbody>
</table>
```

Usage: `<Table rows={holdings} columns={[{ key: "symbol", label: "Symbol" }]} />`. Here `T` is inferred as `Holding`, so `{ key: "sybmol" }` is a compile error.

### What does `extends` mean in `<T extends ...>`?
It is a constraint, not class inheritance. It says `T` must be assignable to that shape. Inside the function I can then safely use the members of the constraint, like `.length` or `.id`.

### What is `keyof` and how do you use it with generics?
`keyof T` gives a union of the keys of `T`, for `{ symbol: string; ltp: number }` that is `"symbol" | "ltp"`. With `K extends keyof T` I can accept only valid keys, and `T[K]` gives the type of that key's value. That is how I write `sortBy(rows, "ltp")` safely.

```ts
function sortBy<T, K extends keyof T>(rows: T[], key: K): T[] {
  return rows.toSorted((a, b) => (a[key] < b[key] ? -1 : a[key] > b[key] ? 1 : 0));
}
```

### `any` vs generic: why not just use `any`?
`any` throws away the type, so the result is `any` too. A generic remembers the type: `first<number[]>` returns `number`. Callers get autocomplete and errors.

## Common mistakes
- Adding a type parameter that is used only once, like `function log<T>(x: T): void`. Just use `unknown`.
- Writing `fetchJson<User>()` and thinking the data is checked. The generic is only a promise to the compiler; validate at runtime.
- Forgetting the constraint and then getting "property does not exist on type T".
- Using arrow generics in `.tsx` files: write `<T,>` so it is not read as JSX.

## Resources
- [TS Handbook: Generics](https://www.typescriptlang.org/docs/handbook/2/generics.html) - constraints, defaults, generic classes
- [TS Handbook: keyof type operator](https://www.typescriptlang.org/docs/handbook/2/keyof-types.html) - key unions
- [TS Handbook: Indexed access types](https://www.typescriptlang.org/docs/handbook/2/indexed-access-types.html) - `T[K]`
- [Svelte docs: TypeScript (generic components)](https://svelte.dev/docs/svelte/typescript) - the `generics` attribute
