# Clickjacking

> **In one line:** Clickjacking is when an attacker loads my page in an invisible iframe on their site and tricks the user into clicking buttons on it; I stop it by telling the browser who may frame my page, with CSP `frame-ancestors` and `X-Frame-Options`.

## Key points
- The attacker's page shows something innocent ("Click to win") and places my page in a **transparent iframe** on top. The user's click lands on my real "Confirm payment" button, with their real session.
- **[`X-Frame-Options`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/X-Frame-Options)** (older): `DENY` (never frame) or `SAMEORIGIN` (only my own origin). `ALLOW-FROM` is obsolete and ignored by modern browsers.
- **[CSP `frame-ancestors`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Security-Policy/frame-ancestors)** (modern): a list of who may frame me, e.g. `'none'`, `'self'`, or specific merchant origins. When both are present, modern browsers use `frame-ancestors`.
- Both must be sent as **HTTP response headers**. `frame-ancestors` does not work inside a `<meta>` CSP tag.
- JavaScript "frame-busting" (`if (top !== self) top.location = self.location`) is weak and can be bypassed; use headers.

## Example

```bash
# Page should never be framed (login, account settings, order confirm)
Content-Security-Policy: frame-ancestors 'none'
X-Frame-Options: DENY

# Only my own site may frame it
Content-Security-Policy: frame-ancestors 'self'
X-Frame-Options: SAMEORIGIN

# Embeddable widget: only approved partner sites
Content-Security-Policy: frame-ancestors https://shop.example https://*.partner.example
```

Setting it in SvelteKit for every response:

```ts
// src/hooks.server.ts
import type { Handle } from '@sveltejs/kit';

export const handle: Handle = async ({ event, resolve }) => {
  const response = await resolve(event);
  response.headers.set('Content-Security-Policy', "frame-ancestors 'none'");
  response.headers.set('X-Frame-Options', 'DENY'); // for very old browsers
  return response;
};
```

## When to use it
- **Default for every page:** `frame-ancestors 'none'` (or `'self'`).
- **Checkout that merchants embed in an iframe:** you cannot use `'none'`. Allow-list the registered merchant origins per merchant (look it up on the server and build the header), and add extra protection on the final pay button: require a fresh user action like OTP or a second step, so a single hidden click is not enough.
- **WebViews:** a native app loading your page directly in a WebView is not framing, so these headers do not apply there. They only control iframes.

## Likely questions

### What is clickjacking and how do you prevent it?
An attacker embeds my site in a hidden iframe and overlays it with bait, so the user clicks my real buttons without knowing. I prevent it with the response header `Content-Security-Policy: frame-ancestors 'none'` (or a list of allowed origins), and also `X-Frame-Options: DENY` for older browsers.

### X-Frame-Options vs frame-ancestors?
X-Frame-Options is the older header with only `DENY` or `SAMEORIGIN`; it cannot list several allowed sites. `frame-ancestors` is part of CSP, supports multiple origins and wildcards for subdomains, and wins if both are set. I send both for coverage.

### Our checkout must be embeddable. What now?
I use `frame-ancestors` with only the approved merchant origins, generated per request on the server. Then I make sensitive actions resistant to a single blind click: confirmation steps, OTP, and not auto-focusing the pay button. Browsers also offer the [IntersectionObserver v2 visibility check](https://web.dev/articles/intersectionobserver-v2) in Chromium to detect if the frame is obscured, though support is limited.

## Common mistakes
- Putting `frame-ancestors` in a `<meta http-equiv>` tag (ignored).
- Using `X-Frame-Options: ALLOW-FROM` (not supported by modern browsers).
- Protecting only the home page, not the sensitive pages like "confirm" or "settings".
- Relying on JS frame-busting code.

## Resources
- [OWASP: Clickjacking Defense Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Clickjacking_Defense_Cheat_Sheet.html) - headers and edge cases
- [MDN: CSP frame-ancestors](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Security-Policy/frame-ancestors) - syntax and examples
- [MDN: X-Frame-Options](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/X-Frame-Options) - DENY vs SAMEORIGIN
