# Sensitive data in the frontend

> **In one line:** Anything sent to the browser can be seen by the user and by any script on the page, so I never put secrets in client code, I mask sensitive numbers, and I never log personal data.

## Key points
- **PII** (personally identifiable information): name, phone, email, PAN or ID numbers, bank account, card number, address. Treat it as toxic in logs.
- **Never log PII:** `console.log`, error trackers, analytics events and session replay tools all ship data to other places. Scrub before sending.
- **Mask on display:** show `XXXX XXXX 1234`. Better, have the **server send only the masked value** so the full number never reaches the browser.
- **No secrets in client env variables:** in Vite anything with `VITE_`, and in SvelteKit anything in `$env/static/public` / `PUBLIC_`, is bundled into JS that anyone can read.
- Don't keep sensitive data longer than needed: avoid localStorage, clear forms after submit, set `autocomplete="off"` or proper values, and use `Cache-Control: no-store` on sensitive API responses.

## Example

```ts
// mask.ts
export function maskAccount(acc: string, visible = 4) {
  const digits = acc.replace(/\s/g, '');
  return digits.slice(-visible).padStart(digits.length, 'X').replace(/(.{4})/g, '$1 ').trim();
}
maskAccount('123456789012'); // "XXXX XXXX 9012"

// Scrub PII before anything leaves the app (logger, error tracker, analytics)
const PII_KEYS = /^(email|phone|pan|account|card|cvv|otp|password|token|address)$/i;

export function scrub(obj: unknown): unknown {
  if (Array.isArray(obj)) return obj.map(scrub);
  if (obj && typeof obj === 'object') {
    return Object.fromEntries(
      Object.entries(obj).map(([k, v]) => [k, PII_KEYS.test(k) ? '[REDACTED]' : scrub(v)])
    );
  }
  return obj;
}

logger.info('payment_failed', scrub({ orderId: 'o_1', card: '4111111111111111', reason: 'declined' }));
// -> { orderId: 'o_1', card: '[REDACTED]', reason: 'declined' }
```

Env variables in SvelteKit:

```ts
// OK: public, ends up in the browser bundle
import { PUBLIC_API_BASE } from '$env/static/public';

// Server only: SvelteKit refuses to build if a client module imports this
import { PAYMENT_SECRET_KEY } from '$env/static/private'; // use in +page.server.ts / +server.ts only
```

## When to use it
- **Checkout:** card fields are usually rendered in a payment-provider iframe or tokenised, so the merchant page never touches the raw number. Show only last 4 digits after.
- **Trading profile page:** bank account and ID numbers come masked from the API; a "reveal" button calls a separate endpoint after re-auth.
- **Error monitoring:** configure the tracker's `beforeSend` hook to run `scrub`, and mask inputs in session replay.
- **WebViews:** the host app may log URLs. Never put tokens or PII in query strings; use POST bodies or headers.

## Likely questions

### How do you make sure PII is not logged?
I route all logging through one logger with a scrub function that redacts known keys, and I set the same scrub in the error tracker's `beforeSend` and in analytics. I avoid putting PII in URLs, because URLs end up in server logs, browser history and the `Referer` header. And in code review we block raw `console.log` of API responses.

### Where should masking happen?
On the server, ideally. If the API returns the full account number and the UI masks it, the full value is still visible in DevTools and in any XSS. So the API sends `XXXX1234`, and only a separate, re-authenticated call reveals the full value if really needed.

### Can I put an API key in a `.env` file for the frontend?
Only if it is meant to be public, like a maps key restricted by domain. Anything in the client bundle is public; `.env` just feeds the build. Real secrets stay on the server, and the browser calls my backend, which calls the third party with the secret.

### What else leaks data on the client?
Browser autofill and form caching, storage that never gets cleared, screenshots in the app switcher on mobile, sensitive data cached by the HTTP cache or a service worker, and third-party scripts that can read the DOM.

## Common mistakes
- `console.log(user)` left in production.
- Secrets in `VITE_` or `PUBLIC_` env vars, or committed in source maps that are public.
- Masking only with CSS or only in the UI while the API returns full data.
- Tokens or emails in query strings.

## Resources
- [OWASP: Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html) - what never to log
- [SvelteKit: $env/static/private](https://svelte.dev/docs/kit/$env-static-private) - server-only env vars
- [SvelteKit: $env/static/public](https://svelte.dev/docs/kit/$env-static-public) - what is safe to expose
- [OWASP Top 10: Cryptographic Failures (sensitive data exposure)](https://owasp.org/Top10/A02_2021-Cryptographic_Failures/) - why exposure matters
