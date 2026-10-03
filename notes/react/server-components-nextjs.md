# Server Components and Next.js basics

> **In one line:** Server Components run only on the server and send rendered output with zero JS for themselves, while Client Components (marked `"use client"`) ship JS to the browser so they can use state, effects and event handlers.

## Key points
- In the Next.js **App Router** (the `app/` folder), every component is a [Server Component](https://react.dev/reference/rsc/server-components) by default. It can be `async`, read a database or secret API key directly, and adds no JS to the bundle.
- [`"use client"`](https://react.dev/reference/rsc/use-client) at the top of a file marks a **boundary**: that file and everything it imports become Client Components. Props passed across must be serializable (no functions, except Server Functions).
- Server Components cannot use `useState`, `useEffect`, event handlers or browser APIs. Client Components cannot be `async` or import server-only code.
- Rendering strategies: **CSR** (render in the browser), **SSR** (render on each request), **SSG** (render at build time), **ISR** (static, but rebuilt in the background after a time).
- **Hydration** means React attaches event handlers to server HTML. If the client renders different HTML from the server, you get a **hydration mismatch**.

## Example
A stock page: server fetches data, a small client island handles the buy button.

```tsx
// app/stocks/[symbol]/page.tsx  (Server Component by default)
import { BuyButton } from "./BuyButton";

export const revalidate = 60; // ISR: regenerate this page at most once a minute

export default async function StockPage({ params }: { params: Promise<{ symbol: string }> }) {
  const { symbol } = await params; // params is a Promise in Next 15+
  const res = await fetch(`https://api.example.com/stocks/${symbol}`, {
    headers: { Authorization: `Bearer ${process.env.API_KEY}` }, // secret never reaches browser
  });
  const stock = await res.json();

  return (
    <main>
      <h1>{stock.name}</h1>
      <p>Last close: {stock.close}</p>
      <BuyButton symbol={symbol} /> {/* only this part ships JS */}
    </main>
  );
}
```

```tsx
// app/stocks/[symbol]/BuyButton.tsx
"use client";
import { useState } from "react";

export function BuyButton({ symbol }: { symbol: string }) {
  const [qty, setQty] = useState(1);
  return (
    <>
      <input type="number" value={qty} onChange={(e) => setQty(Number(e.target.value))} />
      <button onClick={() => placeOrder(symbol, qty)}>Buy {symbol}</button>
    </>
  );
}
```

## When to use it
- Server Components: static-ish content and data reads (company profile, research articles, account statements).
- Client Components: anything interactive or live (order form, live price ticker over WebSocket, charts).
- Keep `"use client"` as low (as close to the leaves) as possible so most of the tree stays server-only.

## Likely questions
### Server vs client components: what is the difference?
Server Components render on the server only. Their code never goes to the browser, so they are good for data fetching and keeping secrets, and they reduce bundle size. Client Components are the classic React components: they render on the server for the first HTML too, then hydrate in the browser and can be interactive. A Server Component can render a Client Component and pass it props or children, but a Client Component cannot import a Server Component.

### SSR vs SSG vs ISR vs CSR?
| Strategy | When HTML is made | Good for |
| --- | --- | --- |
| CSR | In the browser, after JS loads | Logged-in dashboards, highly interactive apps |
| SSR | On the server, every request | Personalised pages that need fresh data and SEO |
| SSG | At build time | Marketing pages, docs, blog |
| ISR | At build, then re-generated after N seconds | Product or stock pages that change slowly |

In Next.js, SSG uses `generateStaticParams`, ISR uses `export const revalidate = 60`, and calling dynamic APIs like `cookies()` makes a route dynamic (SSR). Caching defaults have changed between Next versions, so I check the version's docs.

### What is a hydration mismatch and how do you fix it?
The server HTML and the first client render must match. If they differ, React warns and may re-render that part on the client. Common causes: `Date.now()`, `Math.random()`, `typeof window` checks in render, locale-based number formatting, or invalid HTML like a `<div>` inside a `<p>`. Fixes: render browser-only values in `useEffect` after mount, use the same locale and timezone on both sides, or use `suppressHydrationWarning` for one unavoidable text node like a timestamp.

### App Router basics?
Folders are routes. `page.tsx` is the page, `layout.tsx` wraps child pages and keeps state across navigation, `loading.tsx` is an automatic Suspense fallback, `error.tsx` is an automatic error boundary (it must be a Client Component), and `route.ts` defines an API endpoint. `"use server"` marks Server Functions you can call from forms.

### When would you pick Next.js?
When I need SEO plus fast first load (public stock pages, marketing site), when I want server-side data fetching and secrets close to the UI, or when the team is already on React and wants routing, SSR and bundling decided for them. For a pure logged-in SPA behind auth, plain Vite + React can be simpler.

### How does this compare to SvelteKit load functions?
SvelteKit uses [`load` functions](https://svelte.dev/docs/kit/load): `+page.server.ts` runs only on the server (like a Server Component's data fetching), and `+page.ts` runs on server and client. The data is passed as a `data` prop to the page. The difference: in SvelteKit, every component still ships JS and hydrates, while React Server Components can skip shipping JS for whole parts of the tree. SvelteKit has page options `ssr`, `csr` and `prerender` that map to SSR, CSR and SSG.

## Common mistakes
- Putting `"use client"` in the root layout, which turns the whole app into client code.
- Passing a function or class instance from a Server to a Client Component.
- Thinking Server Components replace SSR. They are different: Client Components are still server-rendered for the first HTML.

## Resources
- [react.dev: Server Components](https://react.dev/reference/rsc/server-components) - the official model
- [react.dev: 'use client'](https://react.dev/reference/rsc/use-client) - how the boundary works
- [Next.js docs: App Router](https://nextjs.org/docs/app) - routing, rendering and caching
- [svelte.dev: Loading data](https://svelte.dev/docs/kit/load) - the SvelteKit comparison
