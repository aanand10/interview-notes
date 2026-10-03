# Env variables

> **In one line:** SvelteKit splits env variables into private ones (server only) and public ones (must start with `PUBLIC_`), and it will fail the build if browser code tries to import a private one, so secrets like API keys never reach the client.

## Key points
- **Four modules** ([env docs](https://svelte.dev/docs/kit/$env-static-private)):

| Module | When read | Who can import |
|---|---|---|
| `$env/static/private` | at build time, inlined | server code only |
| `$env/static/public` | at build time, inlined | anywhere |
| `$env/dynamic/private` | at runtime (`env.X`) | server code only |
| `$env/dynamic/public` | at runtime (`env.X`) | anywhere |

- **Public means `PUBLIC_` prefix.** Only variables starting with `PUBLIC_` appear in the public modules. Everything else is private.
- **Static** values are replaced in the code at build time. Fast and allows dead-code removal, but you must rebuild to change them.
- **Dynamic** values are read when the server runs. Use them when the same build is deployed to staging and production with different values.
- **Server-only guard.** Importing a private module (or anything in `$lib/server/` or a `*.server.ts` file) from code that can reach the browser is a build error.

## Example
```bash
# .env  (never commit this file)
BROKER_API_SECRET=sk_live_123
DATABASE_URL=postgres://...
PUBLIC_WS_URL=wss://prices.example.com
```

```ts
// src/routes/portfolio/+page.server.ts  (server only: OK)
import { BROKER_API_SECRET } from '$env/static/private';

export async function load({ fetch }) {
  const res = await fetch('https://api.broker.com/positions', {
    headers: { Authorization: `Bearer ${BROKER_API_SECRET}` }
  });
  return { positions: await res.json() }; // return data, never the key
}
```

```svelte
<!-- src/lib/LivePrice.svelte  (runs in the browser) -->
<script lang="ts">
  import { PUBLIC_WS_URL } from '$env/static/public';   // fine
  // import { BROKER_API_SECRET } from '$env/static/private'; // build error
  let { symbol } = $props();
  let price = $state<number | null>(null);

  $effect(() => {
    const ws = new WebSocket(`${PUBLIC_WS_URL}?s=${symbol}`);
    ws.onmessage = (e) => (price = JSON.parse(e.data).price);
    return () => ws.close();
  });
</script>
<span>{price ?? '...'}</span>
```

## When to use it
- Private: broker API secret, DB URL, JWT signing key, payment provider secret.
- Public: WebSocket URL, analytics ID, feature flags that are fine to show.
- Dynamic: one Docker image deployed to staging and production with different `DATABASE_URL`.

## Likely questions
### What is the difference between `$env/static/private` and `$env/static/public`?
Private variables can only be imported in server code, like `+page.server.ts`, `+server.ts` and hooks. Public ones must start with `PUBLIC_` and can be used in components that run in the browser. Both are inlined at build time.

### Why must secrets never reach the client?
Anything sent to the browser can be read by anyone: open DevTools, view the JS bundle, done. A leaked broker or payment key lets an attacker call the API as your company, place trades or read user data. So the secret stays on the server, and the browser calls your own server, which then calls the third party.

### Static vs dynamic?
Static is baked in at build, so it is fast and tree-shakeable, but needs a rebuild to change. Dynamic is read at runtime from the environment, so one build can run in many environments.

### Is a `PUBLIC_` variable a secret?
No. Treat it as visible to the world. Never put a key with write access there.

## Common mistakes
- Naming a secret `PUBLIC_...` by accident.
- Returning a secret from a `load` function. Load data is serialized into the HTML.
- Committing `.env`. Commit a `.env.example` with fake values instead.

## Resources
- [SvelteKit: $env/static/private](https://svelte.dev/docs/kit/$env-static-private) - build-time private vars
- [SvelteKit: $env/dynamic/private](https://svelte.dev/docs/kit/$env-dynamic-private) - runtime private vars
- [SvelteKit: Server-only modules](https://svelte.dev/docs/kit/server-only-modules) - how the build blocks leaks
