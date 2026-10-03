# Push notifications

> **In one line:** The Notifications API shows a system notification, the Push API lets a server send a message to the browser's service worker even when the app is closed, and together they give push notifications: the user grants permission, the page subscribes, the server pushes, and the service worker's `push` event calls `showNotification`.

## Key points
- **Two different APIs:** the [Notifications API](https://developer.mozilla.org/en-US/docs/Web/API/Notifications_API) only *displays* a notification (title, body, icon). The [Push API](https://developer.mozilla.org/en-US/docs/Web/API/Push_API) *delivers* a message from your server through the browser's push service (for example Google's FCM for Chrome, Mozilla's for Firefox, Apple's for Safari) to your service worker. Push without a page open needs a service worker.
- **Permission flow:** `Notification.requestPermission()` returns `'granted'`, `'denied'` or `'default'` (dismissed). It must be called from a user gesture (a click) in modern browsers. Once denied, you cannot ask again; only the user can change it in site settings.
- **Subscribe:** `registration.pushManager.subscribe({ userVisibleOnly: true, applicationServerKey })` returns a subscription (an endpoint URL plus keys). Send it to your server and store it per user. The key is your **VAPID** public key, which proves pushes come from your server.
- **Service worker `push` event:** the server sends an encrypted message to the endpoint (usually with a library like `web-push`). The browser wakes the service worker, which must show a notification (`userVisibleOnly: true` is required; no silent pushes on the web).
- **iOS:** Safari supports Web Push from iOS 16.4, but only for web apps added to the Home Screen.

## Example
```js
// main.js - ask only after the user clicks "Enable price alerts"
async function enablePriceAlerts() {
  const permission = await Notification.requestPermission();
  if (permission !== 'granted') return showInlineHint('Alerts are off. You can turn them on in site settings.');

  const reg = await navigator.serviceWorker.ready;
  const sub = await reg.pushManager.subscribe({
    userVisibleOnly: true,
    applicationServerKey: VAPID_PUBLIC_KEY,   // base64url string or Uint8Array
  });
  await fetch('/api/push/subscribe', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(sub),                 // endpoint + keys
  });
}
```

```js
// sw.js
self.addEventListener('push', (event) => {
  const data = event.data?.json() ?? {};      // e.g. { symbol: 'INFY', price: 1610, url: '/stocks/INFY' }
  event.waitUntil(
    self.registration.showNotification(`${data.symbol} hit ${data.price}`, {
      body: 'Your price alert was triggered',
      icon: '/icons/192.png',
      tag: `alert-${data.symbol}`,             // replaces older alert for same symbol
      data: { url: data.url },
    })
  );
});

self.addEventListener('notificationclick', (event) => {
  event.notification.close();
  event.waitUntil(self.clients.openWindow(event.notification.data.url));
});
```

```js
// server (Node) - send a push with the web-push library
import webpush from 'web-push';
webpush.setVapidDetails('mailto:alerts@example.com', VAPID_PUBLIC, VAPID_PRIVATE);
await webpush.sendNotification(subscription, JSON.stringify({ symbol: 'INFY', price: 1610, url: '/stocks/INFY' }));
```

## When to use it
- Price alerts, order executed or rejected, margin call warnings, IPO allotment status. Things the user asked for and wants even when the app is closed.
- Not for marketing spam: users will block it and browsers may quiet your prompts.

## Likely questions
### What is the difference between the Notifications API and the Push API?
The Notifications API shows the notification on the device; a page can use it alone while it is open. The Push API receives messages from a server through the browser vendor's push service, waking the service worker even when no tab is open. A real push notification uses both: Push to deliver, Notifications to display.

### Walk me through the permission and subscription flow.
The user clicks something like "Enable alerts". I call `Notification.requestPermission()`. If granted, I get the service worker registration and call `pushManager.subscribe` with `userVisibleOnly: true` and my VAPID public key. I POST the subscription to my server. Later the server sends an encrypted message to the subscription endpoint, the service worker gets a `push` event and calls `showNotification` inside `waitUntil`.

### What is good permission UX?
Never prompt on page load; most users say no and that `denied` is permanent. Use a "pre-prompt": an in-app button or card explaining the value ("Get notified when INFY crosses your target") and only call the real prompt after they click. Ask in context, for example right after they create a price alert. Respect "not now", and give a settings page to turn alerts off. Chrome even shows quieter prompts for sites with low accept rates.

### What happens if the subscription expires?
The push service returns 404 or 410 to my server when sending. The server should delete that subscription. On the client I can check `pushManager.getSubscription()` on startup and re-subscribe if needed.

### Can I send silent pushes to update data?
Not on the web. `userVisibleOnly: true` is required, so every push must show a notification, otherwise browsers may show a generic one or revoke permission.

## Common mistakes
- Requesting permission on first visit.
- Forgetting `event.waitUntil` around `showNotification`, so the worker is stopped first.
- Putting sensitive data (account balances) in the notification body, which shows on the lock screen.
- Not removing expired subscriptions on the server.

## Resources
- [web.dev: Push notifications overview](https://web.dev/articles/push-notifications-overview) - how the pieces fit together
- [MDN: Push API](https://developer.mozilla.org/en-US/docs/Web/API/Push_API) - subscribe and the `push` event
- [MDN: Notifications API](https://developer.mozilla.org/en-US/docs/Web/API/Notifications_API/Using_the_Notifications_API) - permission and showing notifications
- [web.dev: Permission UX](https://web.dev/articles/push-notifications-permissions-ux) - when and how to ask
