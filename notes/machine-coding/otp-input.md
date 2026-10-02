# OTP input

> **In one line:** An OTP input is N one-digit boxes that move focus forward on typing, back on Backspace, spread a pasted code across all boxes, and open the numeric keyboard on mobile.

## Requirements to confirm
- Length (4 or 6)? Digits only? Auto-submit when all boxes are filled, or a Verify button?
- Should SMS autofill work (iOS and Android suggest the code above the keyboard)?
- Resend OTP with a cooldown timer? Error state on a wrong code (clear boxes or keep them)?

## Component breakdown
- `OtpInput.svelte`: renders the boxes, owns `digits`, calls `oncomplete(code)`.
- `fillFrom(digits, start, text, length)`: pure helper used by both paste and multi-character input (SMS autofill puts the whole code into one box).
- Parent (`VerifyOtp`) does the API call and resend timer.

## State and data flow
- `digits`: array of strings, one per box. `code` is `$derived(digits.join(''))`.
- `inputs`: array of DOM refs (via `bind:this`) so we can call `.focus()`.
- When `code.length === length`, call `oncomplete`. The parent decides what to do.

## Implementation
```svelte
<script>
  let { length = 6, oncomplete } = $props();

  let digits = $state(Array(length).fill(''));
  let inputs = $state([]);
  const code = $derived(digits.join(''));

  function fillFrom(start, text) {
    const clean = text.replace(/\D/g, '').slice(0, length - start); // keep digits only
    for (let i = 0; i < clean.length; i++) digits[start + i] = clean[i];
    inputs[Math.min(start + clean.length, length - 1)]?.focus();
    if (digits.every(Boolean)) oncomplete?.(digits.join(''));
  }

  function oninput(e, i) {
    const value = e.currentTarget.value;
    if (value.length > 1) {
      fillFrom(i, value); // SMS autofill or fast typing
    } else if (/^\d$/.test(value)) {
      digits[i] = value;
      if (i < length - 1) inputs[i + 1].focus();
      if (digits.every(Boolean)) oncomplete?.(digits.join(''));
    } else {
      digits[i] = '';
    }
    e.currentTarget.value = digits[i]; // keep the DOM in sync if we rejected a letter
  }

  function onkeydown(e, i) {
    if (e.key === 'Backspace') {
      if (digits[i]) {
        digits[i] = ''; // first press clears this box
      } else if (i > 0) {
        digits[i - 1] = ''; // empty box: go back and clear the previous one
        inputs[i - 1].focus();
      }
      e.preventDefault();
    } else if (e.key === 'ArrowLeft' && i > 0) {
      inputs[i - 1].focus();
    } else if (e.key === 'ArrowRight' && i < length - 1) {
      inputs[i + 1].focus();
    }
  }

  function onpaste(e, i) {
    e.preventDefault();
    fillFrom(i, e.clipboardData.getData('text'));
  }
</script>

<div role="group" aria-label="One-time password">
  {#each digits as digit, i}
    <input
      bind:this={inputs[i]}
      value={digit}
      inputmode="numeric"
      autocomplete={i === 0 ? 'one-time-code' : 'off'}
      maxlength={i === 0 ? length : 1}
      aria-label={`Digit ${i + 1} of ${length}`}
      oninput={(e) => oninput(e, i)}
      onkeydown={(e) => onkeydown(e, i)}
      onpaste={(e) => onpaste(e, i)}
      onfocus={(e) => e.currentTarget.select()}
    />
  {/each}
</div>
<p class="sr-only" aria-live="polite">{code.length === length ? 'Code complete' : ''}</p>
```

The paste helper, tested in node:

```js
fillFrom(['', '', '', '', '', ''], 0, '12 34-56', 6);
// { next: ['1','2','3','4','5','6'], focusIndex: 5 }  spaces and dashes removed
fillFrom(['1', '', '', '', '', ''], 1, '98765432', 6);
// { next: ['1','9','8','7','6','5'], focusIndex: 5 }  extra digits dropped
fillFrom(['', '', '', '', '', ''], 0, 'abc', 6);
// { next: ['','','','','',''], focusIndex: 0 }       nothing valid to paste
```

## Edge cases and accessibility
- **Numeric keyboard**: use [inputmode="numeric"](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Global_attributes/inputmode), not `type="number"`. `type="number"` allows `e`, `+`, `-`, shows spinners, and can drop leading zeros.
- **SMS autofill**: `autocomplete="one-time-code"` on the first box. The OS fills the whole code into that box, so `maxlength` on it is the full length and `oninput` spreads it. See [web.dev: SMS OTP form](https://web.dev/articles/sms-otp-form).
- **Paste anywhere**: paste into box 3 starts filling from box 3. Non-digits are stripped.
- **Select on focus** so typing over a filled box replaces it.
- **Labels**: each box says "Digit 2 of 6"; the group has a name.
- **Auto-submit only once**: the parent should ignore `oncomplete` while a verify request is in flight.

## What interviewers look for
- Focus management that feels natural forward and backward.
- Paste and autofill handled, not just typing.
- Correct mobile keyboard and no `type="number"` traps.

## Likely questions
### Why not a single input?
A single input with `maxlength=6` and letter spacing is simpler and works best with autofill and screen readers. Many teams do that and draw boxes with CSS. Separate boxes are what designers often ask for, so I show I can handle the focus logic, but I would mention the single-input option.

### How does Backspace work?
If the current box has a digit, clear it and stay. If it is already empty, move to the previous box and clear that one. That matches what users expect.

### How do you handle the [WebOTP API](https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API)?
On Chrome Android, `navigator.credentials.get({ otp: { transport: ['sms'] } })` can read the code from a specially formatted SMS. I would feature-detect it and treat it as a bonus; `autocomplete="one-time-code"` is the baseline.

## Resources
- [web.dev: SMS OTP form best practices](https://web.dev/articles/sms-otp-form) - autocomplete and inputmode
- [MDN: inputmode](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Global_attributes/inputmode) - mobile keyboards
- [MDN: paste event](https://developer.mozilla.org/en-US/docs/Web/API/Element/paste_event) - reading clipboard data
- [MDN: WebOTP API](https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API) - reading the SMS code
