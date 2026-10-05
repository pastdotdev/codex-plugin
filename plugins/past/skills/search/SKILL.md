---
name: search
description: Search this project's own history in past.dev: earlier decisions, why something was built a certain way, what was tried before. Use when the user asks what was decided, whether something was attempted already, or why the code looks the way it does, and the answer is not in the working tree.
---

# Searching past.dev

past.dev holds this team's earlier coding sessions as memory. The repository says what the code is.
past.dev says why it became that, and what was rejected on the way.

## When to search

Search when the question is about history rather than about the current code:

- "What did we decide about X?"
- "Have we tried this before?"
- "Why is this built this way?"
- "What was the reasoning behind Y?"

Do not search for things the working tree answers. Read the files for those. past.dev already puts
what it recalls into the context at the start of a session and on every prompt; search when that
is not enough.

## How to search

```
node ../../bin/past-hook.mjs search <the user's question, in their own words> --host codex
```

Run it from this skill's folder: the path is relative to it. This command reaches past.dev over the network and writes to `~/.past`, and Codex's sandbox blocks both. Request escalated permissions when you run it. If it answers that it ran inside the sandbox, run it again with escalated permissions.

Pass the question as it was asked. The recall runs on meaning, so a rewritten or shortened
question returns worse evidence than the original.

## How to use what comes back

Each result carries a date and the text of one memory. Cite the date when you use one, so the
user can judge how current it is. A memory is evidence of what was said at a time, not a
statement of what is true now. When it disagrees with the code, the code wins, and it is worth
telling the user that the two disagree.

If nothing comes back, say so plainly. Do not fill the gap with a guess.
