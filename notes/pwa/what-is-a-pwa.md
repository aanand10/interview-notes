# What is a PWA

> **In one line:** A Progressive Web App is a normal website that uses HTTPS, a web app manifest and a service worker so it can be installed, open in its own window, load fast and keep working when the network is bad.

## Key points
- **Three building blocks:** served over **HTTPS** (secure context), a **[web app manifest](https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps/Manifest)** (a JSON file that describes name, icons, colours and how to open the app), and a **[service worker](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API)** (a background script that sits between the page and the network, used for caching, offline and push).
- **"Progressive"** means it works as a plain website in any browser, and gets extra powers (install, offline, push) where the browser supports them. This is progressive enhancement.
- **Installable:** Chromium browsers need HTTPS plus a manifest with `name` or `short_name`, 192px and 512px `icons`, `start_url`, `display` (for example `standalone`), and `prefer_related_applications` not set to true. A service worker is no longer strictly required to install in Chrome, but you want one for offline support.
- **Benefits:** one codebase, no app store review, instant updates, small download, linkable URLs, works offline, can send push notifications.
- **Limits:** less access to device features than native, and **iOS / Safari is the most limited platform** (details below).

## Example
```html
<!-- index.html -->
<link rel="manifest" href="/manifest.webmanifest" />
<meta name="theme-color" content="#0b1220" />
<!-- iOS still uses this for the home screen icon -->
<link rel="apple-touch-icon" href="/icons/apple-touch-icon.png" />
```

```json
{
  "name": "Acme Trade",
  "short_name": "Trade",
  "start_url": "/?source=pwa",
  "scope": "/",
  "display": "standalone",
  "background_color": "#0b1220",
  "theme_color": "#0b1220",
  "icons": [
    { "src": "/icons/192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icons/512.png", "sizes": "512x512", "type": "image/png" },
    { "src": "/icons/maskable-512.png", "sizes": "512x512", "type": "image/png", "purpose": "maskable" }
  ]
}
```

```js
// main.js - register the service worker after the page has loaded,
// so it does not compete with the first render for bandwidth.
if ('serviceWorker' in navigator) {
  window.addEventListener('load', () => {
    navigator.serviceWorker.register('/sw.js');
  });
}

// Custom install button (Chromium only - Safari does not fire this event)
let deferredPrompt;
window.addEventListener('beforeinstallprompt', (e) => {
  e.preventDefault();          // stop the default mini-infobar
  deferredPrompt = e;          // keep it, show our own "Install app" button
  showInstallButton();
});

async function onInstallClick() {
  deferredPrompt.prompt();
  const { outcome } = await deferredPrompt.userChoice; // 'accepted' | 'dismissed'
  deferredPrompt = null;
}

// Detect if we are running as an installed app
const isInstalled = window.matchMedia('(display-mode: standalone)').matches;
```

## When to use it
- A trading app where users want a home-screen icon, fast repeat loads and a usable shell (watchlist, last known portfolio) on a flaky mobile connection.
- Price alerts through web push without building separate native apps.
- In SvelteKit you add a `src/service-worker.js` file and SvelteKit registers and bundles it for you ([SvelteKit service workers](https://svelte.dev/docs/kit/service-workers)).

## Likely questions
### What makes a website a PWA?
Three things: it is served over HTTPS, it has a web app manifest, and it usually registers a service worker. HTTPS is needed because a service worker can intercept every request, so it must not be injectable by a man-in-the-middle. The manifest makes it installable, and the service worker gives offline support, caching and push. On top of that, a good PWA is responsive and fast.

### What does "installable" mean and how does install work?
Installable means the browser can add it to the home screen or desktop and open it in its own window without the browser UI. On Chrome and Edge, once the install criteria are met, the browser shows an install option and fires `beforeinstallprompt`, which lets me show my own "Install" button. On iOS there is no install prompt and no event; the user must tap Share, then "Add to Home Screen". So on iOS I show a small hint with those steps instead.

### What are the benefits of a PWA over a native app?
One codebase for web, Android, iOS and desktop. No app store review, so a fix ships as soon as I deploy. It is linkable and indexable by search engines. The install is tiny, and updates happen in the background through the service worker. For a fintech team this means faster releases and lower cost.

### What are the limits, especially on iOS?
On iOS, every browser uses WebKit, so Safari's rules apply everywhere. There is no `beforeinstallprompt`, so install is manual. [Web push only works since iOS 16.4](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/), and only after the user adds the app to the home screen. There is no Background Sync or Periodic Background Sync. Storage can be evicted, and in Safari tabs, script-written storage can be deleted after 7 days without user interaction (tracking prevention). In general PWAs also have weaker access to things like Bluetooth, NFC, contacts and background execution compared to native.

### Why is HTTPS required?
A service worker is a powerful proxy that can rewrite any response for the whole scope. If the page were on plain HTTP, an attacker on the network could inject their own service worker and keep control even after the user leaves that network. `localhost` is allowed for development.

### How do you know if the app is running installed?
Use the media query `(display-mode: standalone)` in CSS or `matchMedia` in JS. On iOS, `navigator.standalone` is also true for home-screen apps. I use this to hide "Install" banners and to track installed users in analytics.

## Common mistakes
- Thinking a service worker alone makes a PWA. Without a valid manifest it will not be installable.
- Forgetting a `maskable` icon, so Android crops the logo badly.
- Assuming `beforeinstallprompt` works everywhere. It is Chromium only.
- Caching live data (prices, balances) like static files. A PWA must never show stale money data as if it were fresh.
- Not testing on a real iPhone, where most PWA surprises happen.

## Resources
- [web.dev: Learn PWA](https://web.dev/learn/pwa/) - full free course, start here
- [MDN: What is a progressive web app?](https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps/Guides/What_is_a_progressive_web_app) - clear definition and building blocks
- [MDN: Making PWAs installable](https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps/Guides/Making_PWAs_installable) - exact install requirements
- [web.dev: Installation prompt](https://web.dev/learn/pwa/installation-prompt) - custom install button with `beforeinstallprompt`
- [WebKit: Web Push for Web Apps on iOS and iPadOS](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/) - what iOS supports
- [SvelteKit: Service workers](https://svelte.dev/docs/kit/service-workers) - PWA support in SvelteKit
