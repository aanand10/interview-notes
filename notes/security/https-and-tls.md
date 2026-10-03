# HTTPS and TLS

> **In one line:** HTTPS is HTTP sent over TLS, which encrypts traffic, proves the server is who it claims to be, and stops tampering; powerful browser features like service workers and PWAs only work on HTTPS because of that.

## Key points
- **TLS** gives three things: **encryption** (nobody can read it), **integrity** (nobody can change it), **authentication** (the certificate proves the domain).
- During the **handshake**, the browser checks the server's certificate against trusted authorities and both sides agree on keys. TLS 1.3 needs only one round trip.
- **[Secure context](https://developer.mozilla.org/en-US/docs/Web/Security/Secure_Contexts):** many APIs only exist on HTTPS (or `localhost`): service workers, PWA install, geolocation, camera, clipboard write, Web Crypto `subtle`, push notifications.
- **[HSTS](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Strict-Transport-Security)** (`Strict-Transport-Security`) tells the browser to always use HTTPS for this site, which blocks downgrade attacks.
- Mixed content: an HTTPS page loading `http://` scripts gets them blocked.

## Example

```js
// Feature check before registering a service worker
if (window.isSecureContext && 'serviceWorker' in navigator) {
  navigator.serviceWorker.register('/sw.js');
} else {
  console.warn('Service worker needs HTTPS (or localhost)');
}
```

```bash
# Force HTTPS for a year, including subdomains
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

## Why service workers and PWAs need HTTPS
A service worker sits between the page and the network and can **rewrite every response**, and it keeps running across visits. If it could be installed over plain HTTP, an attacker on public Wi-Fi could inject a malicious service worker that lives on after the user leaves that network. HTTPS guarantees the worker script really came from the site. PWA installability also requires a secure origin, and offline support and push both depend on a service worker, so no HTTPS means no installable app, no offline mode and no push.

## When to use it
- Always, in production. For a payment or trading app it is non-negotiable, plus HSTS.
- Local dev: `localhost` counts as secure. Testing on a phone over your LAN IP is not secure, so use a tunnel or a local HTTPS cert.

## Likely questions

### Why is HTTPS required for service workers?
Because a service worker can intercept and change all requests for the site and persists after the page closes. Over HTTP, a man-in-the-middle could install one and keep control of the site for that user. HTTPS proves the script is genuine. `localhost` is allowed for development.

### What does TLS protect, and what not?
It protects data in transit between browser and server: encryption, integrity and server identity. It does not protect against XSS, a compromised server, or data stored insecurely on the device. The domain name is also still visible to the network in most setups.

## Resources
- [MDN: Secure contexts](https://developer.mozilla.org/en-US/docs/Web/Security/Secure_Contexts) - which APIs need HTTPS
- [MDN: Service Worker API](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API) - why they are HTTPS only
- [MDN: Strict-Transport-Security](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Strict-Transport-Security) - HSTS header
- [web.dev: Why HTTPS matters](https://web.dev/articles/why-https-matters) - plain-language overview
