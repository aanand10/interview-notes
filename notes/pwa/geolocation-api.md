# Geolocation API

> **In one line:** `navigator.geolocation.getCurrentPosition(success, error, options)` asks the browser for the user's location; it only works on HTTPS, shows a permission prompt, and I must always handle the error callback because the user can deny it, the device can time out, or the position may be unavailable.

## Key points
- **Two main calls:** [`getCurrentPosition`](https://developer.mozilla.org/en-US/docs/Web/API/Geolocation/getCurrentPosition) for a one-time read, `watchPosition` for continuous updates (stop with `clearWatch(id)`). Both use callbacks, not Promises.
- **Result:** `position.coords` has `latitude`, `longitude`, `accuracy` (metres) and sometimes `altitude`, `speed`, `heading`; plus `position.timestamp`.
- **Options:** `enableHighAccuracy` (GPS, slower, more battery), `timeout` (ms to wait), `maximumAge` (accept a cached position this old).
- **Permissions:** needs a secure context (HTTPS). The prompt appears on the first call; you can check the state first with the [Permissions API](https://developer.mozilla.org/en-US/docs/Web/API/Permissions_API): `navigator.permissions.query({ name: 'geolocation' })` gives `granted`, `denied` or `prompt`.
- **Errors:** the error has a `code`: `1` PERMISSION_DENIED, `2` POSITION_UNAVAILABLE, `3` TIMEOUT.

## Example
```js
// Wrap in a Promise so it works with async/await
function getPosition(options = {}) {
  return new Promise((resolve, reject) => {
    if (!('geolocation' in navigator)) return reject(new Error('Geolocation not supported'));
    navigator.geolocation.getCurrentPosition(resolve, reject, {
      enableHighAccuracy: false,  // city-level is enough for us
      timeout: 10_000,
      maximumAge: 5 * 60_000,     // a 5-minute-old position is fine
      ...options,
    });
  });
}

async function suggestNearestBranch() {
  try {
    const { coords } = await getPosition();
    return fetchBranches(coords.latitude, coords.longitude);
  } catch (err) {
    if (err.code === 1) showCityPicker('Location is off. Pick your city instead.'); // denied
    else if (err.code === 3) showCityPicker('Could not get location in time.');      // timeout
    else showCityPicker();                                                            // unavailable
  }
}
```

## When to use it
- In a fintech app: suggest the nearest branch or KYC centre, pre-fill the city in a form, or a fraud signal (with clear consent). Always have a manual fallback like a city dropdown.

## Likely questions
### How does `getCurrentPosition` work?
It takes a success callback, an error callback and options. The browser checks permission, shows a prompt if needed, then gets a position from GPS, Wi-Fi or IP, and calls success with a `GeolocationPosition`. I usually wrap it in a Promise and set a `timeout`, because by default it can wait forever.

### How do you handle denial?
I always pass the error callback and check `error.code`. If it is `1` (denied), I do not ask again or nag; I show a manual option like a city picker and maybe a short note on how to enable it in settings. Once denied, the browser will not show the prompt again for that site.

### What about privacy?
Location is sensitive personal data. Ask only when the user does something that needs it (clicking "Find nearest branch"), not on page load. Use low accuracy if that is enough, do not store or send it more than needed, explain why you need it, and say so in the privacy policy. It only works on HTTPS, and iframes need `allow="geolocation"` from the parent.

## Common mistakes
- No error callback, so the feature silently does nothing on denial.
- No `timeout`, so the UI spins forever.
- Prompting on page load.
- Forgetting `clearWatch` for `watchPosition`, draining battery.

## Resources
- [MDN: Using the Geolocation API](https://developer.mozilla.org/en-US/docs/Web/API/Geolocation_API/Using_the_Geolocation_API) - guide with examples
- [MDN: getCurrentPosition()](https://developer.mozilla.org/en-US/docs/Web/API/Geolocation/getCurrentPosition) - options and errors
- [web.dev: User location](https://web.dev/articles/user-location) - best practices for asking
