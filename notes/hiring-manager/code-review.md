# Code review

> **In one line:** "In a review I check correctness and edge cases first, then readability and design, and I give kind, specific comments that say what to change and why, marking which ones block the merge."

## What the interviewer is really checking

- Do you review for things that matter (bugs, security, money paths), not just style?
- Can you give feedback that helps people grow, without being harsh?
- Can you handle disagreement like an adult and settle it with data or a quick call?
- Do you also make your OWN PRs easy to review?

## What I look for (in order)

1. **Correctness:** does it do what the ticket says? Edge cases: empty, loading, error, slow network, double click, back button.
2. **Money and data safety:** double submit protection, idempotency key on order/payment calls, number precision (no float math on money; use integer paise or a decimal library), race conditions with stale responses.
3. **Security:** no `{@html}` / `innerHTML` with user data (XSS), no secrets in client code, no tokens in localStorage if we can avoid it, input validated on the server too.
4. **Accessibility:** labels on inputs, keyboard support, focus management, colour is not the only signal (red/green prices need +/- or an icon).
5. **Performance:** unnecessary re-renders or effects, large imports, lists that need virtualisation, memory leaks (subscriptions or sockets not cleaned up).
6. **Tests:** the risky logic has tests; tests check behaviour, not implementation.
7. **Readability and design:** names, small functions, follows existing patterns. Style is for the linter, not for humans.

## Example: a comment I would leave

```svelte
<script lang="ts">
  let { onSubmit } = $props();
  let submitting = $state(false);

  async function handleClick() {
    // REVIEW (blocking): this can fire twice on a double tap and place two orders.
    // Suggest guarding with `if (submitting) return;`, setting submitting = true,
    // disabling the button, and sending an idempotency key so the server dedupes.
    await onSubmit();
  }
</script>

<button onclick={handleClick}>Place order</button>
```

How I write comments:
- **Label severity:** "blocking", "suggestion", "nit" (tiny style point), "question".
- **Explain why** and suggest a fix, ideally with code.
- **Ask, do not command:** "What happens if this request times out?" teaches more than "Handle timeout."
- **Praise good things** too, specifically.
- Big design concerns: talk early (on the design doc or a call), not as 40 comments on a finished PR.

## Handling disagreement

- First, make sure I understand their reasoning. Often they know a constraint I do not.
- Separate preference from problem. If it is only my taste, I approve and let it go.
- If it matters: bring data (a benchmark, a doc, a failing test case), or a 10-minute call instead of a long comment thread.
- If still stuck: follow the team's agreed conventions, or ask a tech lead to decide, then commit fully to the decision ("disagree and commit").
- Write the outcome into the team guidelines so we do not argue about it again.

## Making my own PRs easy to review

- Small PRs (roughly under 400 changed lines), one purpose each.
- Description: what, why, how to test, screenshots or a short video for UI.
- Self-review the diff before asking others.
- Automate style: Prettier, ESLint, type checks in CI.

## Fill-in template

```text
What I focus on in reviews: [fill in: your top 3]
A bug I caught in review: [fill in: generic, what it would have caused]
A time I disagreed in a review: [fill in: topic, how resolved, outcome]
Feedback I received that changed how I code: [fill in]
A team review practice I introduced: [fill in: e.g. PR template, checklist]
```

## Example answer (generic, for illustration only)

> EXAMPLE, not a real story. Replace with your own.

"In a checkout PR I noticed the pay button was not disabled while the request was in flight, so a double tap could create two payment attempts. I marked it blocking, explained the risk, and suggested a guard plus an idempotency key. Another time a teammate and I disagreed about putting server data in a global store versus loading it per route. We had a short call, I wrote a small prototype of both, and we chose per-route loading because it avoided stale data. I added the decision to our frontend guidelines."

## Likely questions

### What do you look for in a code review?
Correctness and edge cases first, then security and money safety, then accessibility and performance, then tests, then readability. Formatting is for tools.

### How do you give feedback to a junior?
Kind and specific, with the why, and often as a question so they think it through. For bigger issues I pair with them for 15 minutes instead of writing 20 comments. I point to docs or examples so they can learn the pattern.

### What if someone keeps ignoring your review comments?
Talk one-on-one first; maybe my comments are unclear or the timeline is tight. Agree on which items are blocking. If it is a pattern, we turn it into a team rule or lint check so it is not personal.

### How long should a review take?
Aim to respond within a working day so you do not block others. If a PR is too big to review well, I ask to split it.

## Common mistakes

- Only nitpicks, missing the real bug.
- Rewriting someone's PR in comments to your personal style.
- Approving huge PRs without reading ("LGTM").
- Long comment wars instead of a quick call.

## Resources

- [OWASP Code Review Guide](https://owasp.org/www-project-code-review-guide/) - security checks to include in reviews
- [OWASP XSS Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html) - the most common frontend security bug to catch
- [web.dev: Learn Accessibility](https://web.dev/learn/accessibility) - a11y points to check
- [Testing Library: Guiding principles](https://testing-library.com/docs/guiding-principles) - how to judge whether tests check behaviour
