# Adapters and deployment

> **In one line:** An adapter is a small plugin that takes your SvelteKit build and turns it into the right output for where you deploy: a Node server, plain static files, or Vercel functions.

## Key points
- **You pick it in `svelte.config.js`.** The default `adapter-auto` detects common platforms (Vercel, Netlify, Cloudflare and others) and uses the matching adapter ([adapters docs](https://svelte.dev/docs/kit/adapters)).
- **SvelteKit 3 note:** the examples below use `svelte.config.js` (SvelteKit 2, what most production apps run). In SvelteKit 3 the same options, including `adapter`, are passed to `sveltekit({ ... })` in `vite.config.ts`. Worth saying if asked "what changed in Kit 3".
- **`adapter-node`** builds a standalone Node server. Run it with `node build`. Use it for Docker, a VM or Kubernetes. Configure with env vars like `PORT`, `HOST` and `ORIGIN`.
- **`adapter-static`** prerenders every page to HTML files, with no server at all. Host on S3, GitHub Pages or a CDN. Server `load`, form actions and `+server.ts` at runtime are not available.
- **`adapter-vercel`** turns routes into Vercel serverless functions, with options like edge runtime, regions and ISR (incremental static regeneration: cache a page and refresh it in the background).
- **SPA fallback.** `adapter-static` with `fallback: '200.html'` plus `ssr = false` gives a single-page app: one HTML shell, and the router renders everything in the browser.

## Example
```js
// svelte.config.js  -- Node server for Docker / Kubernetes
import adapter from '@sveltejs/adapter-node';

export default {
  kit: { adapter: adapter({ out: 'build' }) }
};
```

```bash
npm run build
ORIGIN=https://trade.example.com PORT=3000 node build
```

```js
// svelte.config.js  -- SPA mode with adapter-static
import adapter from '@sveltejs/adapter-static';

export default {
  kit: { adapter: adapter({ fallback: '200.html' }) } // host must serve this for unknown paths
};
```

```ts
// src/routes/+layout.ts  -- turn off SSR for the whole app
export const ssr = false;
```

You can also mix: keep SSR but set `export const prerender = true` on marketing pages so they are built once as HTML.

## When to use it
| Need | Adapter |
|---|---|
| Own infra, Docker, long-lived WebSockets next to the app | `adapter-node` |
| Docs site, landing page, or a pure SPA talking to a separate API | `adapter-static` |
| Fast setup on Vercel, edge functions, ISR | `adapter-vercel` (or `adapter-auto`) |

A trading app with auth, sessions and server-side API keys usually needs a server: `adapter-node` or a serverless adapter.

## Likely questions
### What is an adapter?
SvelteKit's build output is platform-neutral. The adapter is the last step that packages it for a target: a Node server, static files, or a platform's functions. Swapping the adapter is usually the only change when moving hosts.

### adapter-node vs adapter-static vs adapter-vercel?
Node gives a full server you run yourself, so everything works: SSR, actions, endpoints. Static prerenders everything, so it is cheap and fast but has no server code at runtime. Vercel splits routes into serverless or edge functions and adds platform features like ISR, but you have serverless limits like cold starts and no long-lived connections.

### What is SPA fallback and what are the downsides?
With `fallback`, the host serves one HTML file for any unknown URL, and the client router takes over. Downsides: blank page until JS loads (slower first paint), worse SEO, and no server `load`, actions or `+server.ts`. I use it only for apps behind login where SEO does not matter, or when the backend is a separate API.

### Why set `ORIGIN` with adapter-node?
SvelteKit needs the real public URL to build `url` correctly and to do its CSRF check on form posts. Behind a proxy, it can't guess, so you set `ORIGIN` (or `PROTOCOL_HEADER` and `HOST_HEADER`).

## Common mistakes
- Using `adapter-static` and then adding a form action. It will fail at build.
- Forgetting to configure the host to serve the fallback page, so deep links give 404.
- Forgetting `ORIGIN` behind a proxy, which breaks form posts with a cross-site error.

## Resources
- [SvelteKit: Adapters](https://svelte.dev/docs/kit/adapters) - overview of official adapters
- [SvelteKit: Node servers](https://svelte.dev/docs/kit/adapter-node) - env vars and options
- [SvelteKit: Static site generation](https://svelte.dev/docs/kit/adapter-static) - prerendering and fallback
- [SvelteKit: Single-page apps](https://svelte.dev/docs/kit/single-page-apps) - SPA mode trade-offs
