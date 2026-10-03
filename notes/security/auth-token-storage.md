# Auth token storage

> **In one line:** I keep the long-lived credential in an HttpOnly, Secure, SameSite cookie so JavaScript can never read it, keep any short-lived access token in memory, and use a refresh token to get new access tokens.

## Key points
- **localStorage** is readable by any script on the page. One XSS bug and the token is stolen and can be used from the attacker's machine.
- **HttpOnly cookies** cannot be read by JS at all. XSS can still make requests from the page, but cannot take the token away. The cost: cookies are sent automatically, so you must handle **CSRF** (SameSite + tokens).
- **Access token + refresh token:** the access token is short-lived (for example 5-15 minutes). The refresh token is long-lived, kept in an HttpOnly cookie, and only sent to the refresh endpoint.
- **Refresh token rotation:** every refresh returns a new refresh token and invalidates the old one. If an old one is reused, the server knows it was stolen and kills the session.
- **Logout across tabs:** tell other tabs with [`BroadcastChannel`](https://developer.mozilla.org/en-US/docs/Web/API/BroadcastChannel) or the `storage` event, and clear the server session so cookies become useless.

## Comparison

| Option | XSS can steal it? | CSRF risk? | Survives reload? | Notes |
|---|---|---|---|---|
| localStorage | Yes | No | Yes | Easy, but worst for XSS. Shared across tabs. |
| sessionStorage | Yes | No | Per tab only | Same XSS problem. |
| JS memory (a variable) | Harder (lost on reload, not in storage) | No | No | Good for access token; needs a refresh on reload. |
| HttpOnly Secure cookie | No (cannot be read) | Yes, mitigate with SameSite | Yes | Best for session / refresh token. |

## Example

```bash
# Server sets the refresh token (only sent to /auth paths)
Set-Cookie: rt=eyJ...; HttpOnly; Secure; SameSite=Strict; Path=/auth; Max-Age=2592000
```

```ts
// auth.ts - access token lives only in memory
let accessToken: string | null = null;
let refreshing: Promise<string> | null = null;

async function refresh(): Promise<string> {
  // Share one refresh call between many parallel 401s
  refreshing ??= fetch('/auth/refresh', { method: 'POST', credentials: 'include' })
    .then(async (r) => {
      if (!r.ok) throw new Error('session expired');
      const { accessToken: t } = await r.json();
      accessToken = t;
      return t;
    })
    .finally(() => { refreshing = null; });
  return refreshing;
}

export async function api(url: string, init: RequestInit = {}): Promise<Response> {
  const token = accessToken ?? (await refresh());
  const withAuth = (t: string) =>
    fetch(url, { ...init, headers: { ...init.headers, Authorization: `Bearer ${t}` } });

  let res = await withAuth(token);
  if (res.status === 401) res = await withAuth(await refresh()); // retry once
  return res;
}
```

Logout across tabs:

```ts
const channel = new BroadcastChannel('auth');

export async function logout() {
  await fetch('/auth/logout', { method: 'POST', credentials: 'include' }); // server revokes + clears cookie
  accessToken = null;
  channel.postMessage({ type: 'logout' });   // tell other tabs
  location.assign('/login');
}

channel.onmessage = (e) => {
  if (e.data?.type === 'logout') {
    accessToken = null;
    location.assign('/login');
  }
};

// Fallback for older setups: writing a key fires "storage" in OTHER tabs only
window.addEventListener('storage', (e) => {
  if (e.key === 'logout-at') location.assign('/login');
});
```

## When to use it
- **Trading app:** session or refresh token in an HttpOnly cookie, access token in memory, short expiry, and re-auth (PIN / 2FA) before placing orders or withdrawing.
- **Checkout in a WebView:** cookies in WebViews can be tricky (some WebViews isolate or block third-party cookies). Often the host app passes a short-lived, single-use token, and the page swaps it for a session cookie on its own domain.
- If one tab logs out, every tab must show the login screen, or a shared computer leaks the session.

## Likely questions

### HttpOnly cookie or localStorage for a JWT?
I prefer an HttpOnly, Secure, SameSite cookie. localStorage is readable by any script, so a single XSS or a compromised third-party script can copy the token and use it anywhere. With an HttpOnly cookie the attacker can still act inside the page while it is open, but cannot steal the token. The trade-off is CSRF, which I handle with SameSite and a CSRF token or Origin checks.

### Why do we need refresh tokens?
So the access token can be short-lived. If an access token leaks, it only works for a few minutes. The refresh token is long-lived but stored more safely (HttpOnly cookie, scoped to `/auth`) and only sent to one endpoint. With rotation, a stolen refresh token gets detected the moment both parties use it.

### Many API calls get 401 at once. What do you do?
I make sure only one refresh request runs. The first 401 starts the refresh and stores the promise; other calls await the same promise, then retry once. If refresh fails, I log the user out instead of looping.

### How do you log out in all tabs?
First, the server must revoke the session or refresh token and clear the cookie; that is the real logout. Then for the UI I broadcast a message with `BroadcastChannel`, and every tab clears its in-memory state and redirects to login. The `storage` event is an older way: writing a key in localStorage fires an event in other tabs of the same origin.

### Is putting the access token in memory enough against XSS?
It helps, because it is not sitting in storage, but a script running in my page can still hook `fetch` and see headers. Nothing on the client is fully safe from XSS. The real fix is preventing XSS (escaping, CSP), and limiting damage with short expiry.

## Common mistakes
- Storing a long-lived token in localStorage "because it's easy".
- Logging out only on the client (deleting the token) while the server session stays valid.
- No rotation, so a stolen refresh token works for 30 days.
- Infinite refresh loops when the refresh endpoint itself returns 401.
- Forgetting `credentials: 'include'` for cross-origin cookie calls.

## Resources
- [OWASP: Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html) - cookie flags and session lifecycle
- [MDN: Using HTTP cookies](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Cookies) - HttpOnly, Secure, SameSite basics
- [MDN: BroadcastChannel](https://developer.mozilla.org/en-US/docs/Web/API/BroadcastChannel) - messaging between tabs
- [OWASP: HTML5 Security Cheat Sheet (Local Storage)](https://cheatsheetseries.owasp.org/cheatsheets/HTML5_Security_Cheat_Sheet.html#local-storage) - why not to store secrets in localStorage
