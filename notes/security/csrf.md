# CSRF (Cross-Site Request Forgery)

> **In one line:** CSRF is when a different website makes the user's browser send a request to my site, and the browser automatically attaches the user's cookies, so the request looks real.

## Key points
- It only works because **browsers attach cookies automatically** to requests going to a site, even if another site started the request.
- The attacker **cannot read the response**. They can only cause a side effect: transfer money, change email, place an order.
- **[`SameSite` cookies](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Set-Cookie#samesitesamesite-value)** stop the browser from sending the cookie on cross-site requests. This is the main modern defence.
- **CSRF tokens** are a random secret the server puts in the page; the attacker's site cannot know it, so its forged requests fail.
- Extra checks: verify the `Origin` header, never change state with `GET`, and require re-auth (PIN, OTP) for high-risk actions.

## How the attack works

1. The user logs in to `bank.example`. The browser stores a session cookie.
2. The user visits `evil.example` in another tab.
3. That page has a hidden auto-submitting form:

```html
<form action="https://bank.example/transfer" method="POST" id="f">
  <input type="hidden" name="to" value="attacker" />
  <input type="hidden" name="amount" value="50000" />
</form>
<script>document.getElementById('f').submit();</script>
```

4. The browser sends the POST **with the bank's session cookie**. If the server only checks the cookie, the transfer happens.

## Example: the defences

```bash
# 1. SameSite on the session cookie
Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Lax; Path=/
```

| SameSite value | Cookie sent on cross-site requests? |
|---|---|
| `Strict` | Never. Even clicking a link from another site arrives logged out. |
| `Lax` | Only on top-level navigations with safe methods (clicking a link, GET). Not on cross-site POST, fetch, or iframes. |
| `None` | Always. Must also have `Secure`. Needed for cookies used inside a cross-site iframe or WebView-embedded flow. |

Chrome treats a cookie with no `SameSite` attribute as `Lax`, but do not rely on browser defaults; set it explicitly.

```js
// 2. CSRF token (synchronizer token pattern) on the client
// Server rendered: <meta name="csrf-token" content="r4nd0m-per-session">
const token = document.querySelector('meta[name="csrf-token"]').content;

await fetch('/api/orders', {
  method: 'POST',
  credentials: 'same-origin',
  headers: { 'Content-Type': 'application/json', 'X-CSRF-Token': token },
  body: JSON.stringify({ symbol: 'INFY', qty: 10, side: 'BUY' }),
});
// Server: compare X-CSRF-Token with the token stored in the session. Mismatch -> 403.
```

The **double-submit cookie** variant: the server sets a random value in a readable cookie, and the client copies it into a header. An attacker's site cannot read my cookies, so it cannot copy the value. Signing the value (tying it to the session) makes it stronger.

## When to use it
- Any app using **cookie-based sessions**: order placement, fund transfer, profile and bank-account changes.
- **Embedded checkout** inside an iframe on a merchant site: the cookie needs `SameSite=None; Secure` to work there, which removes SameSite protection. So a CSRF token or Origin check becomes required, not optional.
- If auth is a bearer token in the `Authorization` header (not a cookie), classic CSRF does not apply, because the browser does not add that header automatically.

## Likely questions

### How does CSRF work?
The user is logged in to my site with a cookie. They visit a malicious site, which submits a form or makes a request to my site. The browser attaches my cookie automatically, so my server thinks the user did it. The attacker cannot see the response, but they can trigger actions like a transfer.

### How does `SameSite` help? Is it enough?
`SameSite=Lax` or `Strict` tells the browser not to send the cookie on cross-site sub-requests like a POST form from another site. That blocks most CSRF. It is not always enough: `Lax` still allows top-level GET, so any GET that changes state is still exposed; same-site subdomains (another subdomain I do not control well) are not "cross-site"; and some flows need `SameSite=None`. So I combine it with tokens or Origin checks for sensitive actions.

### What is a CSRF token?
A random, unguessable value tied to the user's session. The server puts it in the page or a cookie, and the client sends it back in a header or hidden form field. The server checks it matches. The attacker's site cannot read my page or cookies because of the same-origin policy, so it cannot include the right token.

### Does CORS protect against CSRF?
Not fully. CORS controls who can *read* responses. A simple cross-site form POST does not need a preflight, so it is still sent. CORS helps only for requests that trigger a preflight, like JSON with custom headers, but I should not rely on it as the CSRF defence.

### How do SvelteKit form actions handle CSRF?
SvelteKit checks the `Origin` header on form submissions (POST, PUT, PATCH, DELETE with form content types) and rejects cross-origin ones by default. You can allow trusted origins with the [`csrf` config](https://svelte.dev/docs/kit/configuration#csrf).

## Common mistakes
- Changing state with `GET` (like `/logout` or `/cancel?id=1`). `Lax` cookies are still sent there.
- Thinking JSON APIs are immune. A form with `enctype="text/plain"` can send a body that looks like JSON if the server does not check `Content-Type`.
- Putting the CSRF token in the URL, where it leaks through logs and the `Referer` header.
- Forgetting that `SameSite=None` requires `Secure`, or the cookie is rejected.

## Resources
- [OWASP: CSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html) - all defences with trade-offs
- [MDN: Set-Cookie SameSite](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Set-Cookie#samesitesamesite-value) - exact cookie behaviour
- [web.dev: SameSite cookies explained](https://web.dev/articles/samesite-cookies-explained) - Lax vs Strict vs None with diagrams
- [SvelteKit: csrf config](https://svelte.dev/docs/kit/configuration#csrf) - built-in Origin check
