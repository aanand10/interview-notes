# Mentoring

> **In one line:** "I mentor by making people independent: clear onboarding docs, pairing on real tasks, reviews that explain the why, and slowly giving them bigger ownership."

## What the interviewer is really checking

- SDE 2 is expected to **multiply the team**, not only ship their own tickets.
- Do you have a real example of helping someone grow, with a visible result?
- Do you create things that scale (docs, templates, checklists), not just answer the same question ten times?
- Are you patient and respectful, or do you just take over the keyboard?

## Key points

- **Onboarding:** a written guide (setup, architecture map, how to run tests, how we deploy, who to ask) plus a "first good task" list.
- **Pairing:** short sessions on real work. They drive, I navigate. I ask questions instead of giving answers.
- **Reviews as teaching:** explain the why, link docs, praise good choices.
- **Gradual ownership:** small bug -> small feature -> a feature end to end with a design doc.
- **Measure it:** time to first PR, how fast they need less help, what they now own.

## Answer framework (STAR)

- **Situation:** who you helped and what they struggled with.
- **Task:** what you were responsible for (buddy, onboarding owner, tech lead for a feature).
- **Action:** the 2-3 specific things you did.
- **Result:** what changed, with a number if possible, plus what you learned about mentoring.

## Example: what a good onboarding doc covers

```text
1. Setup in 30 minutes: clone, env vars (where to get them), run dev server, run tests
2. Architecture map: folders, routing, state, API layer, design system, one diagram
3. How we work: branch naming, PR template, review rules, CI checks, release + flags
4. Debugging tips: common errors, how to read Sentry, useful dev tools
5. Glossary: domain words (for trading: order types, LTP, margin, settlement)
6. First tasks: 3 labelled "good first issue" tickets with a named buddy
7. Who to ask for what
```

## Fill-in template

```text
Person (no names): [fill in: e.g. "a new joiner fresh out of college"]
Their struggle: [fill in]
What I did: [fill in: onboarding doc / pairing / review style / design doc coaching]
Onboarding docs I wrote at a previous company: [fill in: what they covered, who used them]
Result: [fill in: time to first PR, what they own now, feedback received]
What I learned about mentoring: [fill in]
```

## Example answer (generic, for illustration only)

> EXAMPLE, not a real story. Replace with your own.

"When two new engineers joined, setup took them almost a week because the knowledge was in people's heads. I wrote an onboarding guide with setup steps, an architecture diagram, and our release process, and I made a list of starter tickets. I paired with each of them for 30 minutes a day in the first two weeks, letting them drive. In reviews I explained the reasoning behind patterns and linked docs. The next joiner had a PR merged on day [X] instead of [Y], and after a few months one of them owned a feature end to end."

## Likely questions

### How have you helped a junior engineer grow?
Give one STAR story with a visible result. Mention a scalable artefact (doc, checklist) and a personal habit (pairing, explaining the why).

### How do you balance mentoring with your own deliverables?
I block fixed time (for example a daily 30-minute slot), and I invest in docs so repeated questions answer themselves. I tell my manager about mentoring time so it is planned, not hidden.

### A junior keeps making the same mistake. What do you do?
Check if the knowledge is written down; if not, write it. Pair once on that exact issue. If it is a common trap, add a lint rule or test so the tool catches it, not a person.

### How do you mentor someone more senior in one area but new to the codebase?
Respect their experience, focus on context: domain knowledge, history of decisions, and who owns what. Ask them for feedback on our patterns, since fresh eyes spot problems.

## Common mistakes

- Only "I answered their questions". That is helping, not mentoring.
- Taking over the keyboard.
- No result or change to show.
- Naming the person or criticising them in the story.

## Resources

- [MDN: Learn web development](https://developer.mozilla.org/en-US/docs/Learn_web_development) - a solid curriculum to point new juniors to
- [Svelte tutorial](https://svelte.dev/tutorial) - interactive onboarding for engineers new to Svelte
- [web.dev: Learn](https://web.dev/learn) - structured courses (CSS, performance, accessibility) for growth plans
