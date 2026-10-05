---
name: status
description: Show what past.dev is connected to in Codex, whether its hooks run, and what it has sent. Use when the user asks whether past.dev is working, connected or sending.
---

# past.dev status

Run this and show the output to the user as it is:

```
node ../../bin/past-hook.mjs status --host codex
```

Run it from this skill's folder: the path is relative to it. It only reads, so it runs inside the sandbox.

If the key or the identity is missing, say which one and point at `$past:connect`. Do not print
the key beyond the prefix the command already shows.

If the output says the hooks have never run in Codex, that is the reason nothing is recalled or
sent: Codex runs no hook that a person has not trusted. Tell the user where to trust the five past.dev
hooks: `/hooks` in a terminal session, or the past.dev plugin's page in the ChatGPT desktop app, where
**Trust all** does it. Then they send one prompt and ask for the status again.
