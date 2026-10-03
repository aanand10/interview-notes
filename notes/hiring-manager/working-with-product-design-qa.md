# Working with Product, Design, QA

> **In one line:** "I treat Product, Design and QA as partners: I clarify requirements early in writing, push back with trade-offs and data rather than opinions, and bring QA in before code is finished."

## What the interviewer is really checking

- Can you handle **ambiguity** without either freezing or guessing wildly?
- Can you **push back** respectfully, offering options instead of a flat "no"?
- Do you understand the user and business side, not only the code?
- Do you respect design intent while flagging what is costly or bad for performance or accessibility?

## Key points

- **Unclear requirements:** write my understanding as acceptance criteria, list open questions, get written confirmation. Ask "what problem are we solving and how will we measure success?"
- **Pushing back on scope:** never just "no". Offer options: "We can ship A by Friday, or A+B in two weeks. Here is the risk of each." Suggest an MVP (minimum viable product) plus a phase 2.
- **Design trade-offs:** explain cost in their terms (days, performance, accessibility). Propose a cheaper version that keeps the intent. Use existing design-system components where possible.
- **QA:** share test notes early (edge cases, how to test flags), agree on the device and browser matrix, treat bugs as shared work, not blame.
- **Close the loop:** demo early and often; a 5-minute demo avoids a 2-week rework.

## Example: turning a vague ask into acceptance criteria

```text
Ask from Product: "Show live P&L on the portfolio page."

My questions:
- Live = every tick, or every few seconds? (performance and battery on low-end phones)
- Market closed: show last close value or hide?
- Socket disconnects: show stale data with a "delayed" badge?
- Rounding: 2 decimals? Which currency format?
- Success metric: engagement? fewer support tickets?

Acceptance criteria (agreed):
- P&L updates at most once per second per holding (throttled)
- After 5 s without data, show a "Prices delayed" badge
- Positive values with "+" and green, negative with "-" and red (sign also shown, not only colour)
- Works on [fill in: lowest supported device] without dropped frames
```

## Fill-in template

```text
Unclear requirement story: [fill in: what was vague, questions you asked, outcome]
Scope pushback story: [fill in: what was asked, options you gave, what shipped, result]
Design trade-off story: [fill in: design ask, cost/risk you raised, compromise]
QA collaboration: [fill in: how you worked with QA, a bug caught early]
```

## Example answer (generic, for illustration only)

> EXAMPLE, not a real story. Replace with your own.

"Product wanted a redesigned checkout with animations and three new features in one sprint before a sale event. I broke it down: the payment-method change was the business priority, the animations added risk on low-end devices, and one feature needed backend work that was not ready. I offered two options with estimates. We shipped the payment-method change on time behind a flag and moved the rest to the next sprint. With Design, we kept a simpler transition that did not hurt INP. The sale went smoothly and phase 2 shipped two weeks later."

## Likely questions

### How do you handle unclear requirements?
Clarify before coding: write acceptance criteria and questions, confirm in writing. If some unknowns stay, build the clear part first, make the unclear part easy to change, and demo early.

### Tell me about a time you pushed back on scope.
STAR. Show you understood the business goal, offered options with trade-offs, and the outcome was good for users. Avoid sounding like you just wanted less work.

### Design wants something that is expensive to build. What do you do?
Ask what the user goal is, explain the cost (time, performance, accessibility), and propose an alternative that keeps the intent. If the design is worth it, plan it properly instead of hacking it.

### How do you work with QA?
Involve them at the design stage, share edge cases and how to toggle flags, agree on a device matrix, and automate repeated checks so QA can focus on exploratory testing.

### Product and Design disagree. You are in the middle. What do you do?
Bring both to one short discussion, frame it around the user and the metric, add the technical facts, and if needed suggest an A/B test to settle it with data.

## Common mistakes

- Saying "I just build what is in the ticket".
- Saying "no" without options.
- Blaming Product or QA in the story.
- Silently cutting scope and surprising people at the end.

## Resources

- [web.dev: Interaction to Next Paint (INP)](https://web.dev/articles/inp) - numbers to use when a design hurts responsiveness
- [web.dev: Learn Accessibility](https://web.dev/learn/accessibility) - basis for accessibility trade-off discussions
- [Playwright: Getting started](https://playwright.dev/docs/intro) - automating repeated QA checks
