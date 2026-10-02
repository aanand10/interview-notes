# Interview notes

Content for my Knowledge Tracker app.

```
topics.json                       ← the list of topics
notes/<category>/<topic-slug>.md  ← one Markdown note per topic
```

## Add a topic

1. Add an entry to `topics.json`:

   ```json
   {
     "id": "js-event-loop",
     "title": "Event Loop",
     "category": "JavaScript",
     "priority": "high",
     "tags": ["async"],
     "note": "notes/javascript/event-loop.md",
     "resources": [{ "label": "javascript.info", "url": "https://javascript.info/event-loop" }]
   }
   ```

   - `id`: unique, letters/digits/`-`/`_` only. Your progress is stored under this id, so don't rename it.
   - `priority`: `high`, `medium` or `low` (defaults to `medium`).
   - `note`: optional path to a `.md` file in this repo.
   - `tags`, `resources`: optional.

2. Create the note file, commit and push. The app picks it up on the next load
   (GitHub's raw CDN may take a few minutes).

Headings (`##`, `###`) become the note's table of contents. Fenced code blocks
with a language (```` ```js ````) are syntax-highlighted. Relative links and
images resolve next to the note.
