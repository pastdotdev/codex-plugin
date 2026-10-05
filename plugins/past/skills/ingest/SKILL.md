---
name: ingest
description: Send one Codex session to past.dev now: the one you are in, or one the user names. Use only when the user asks to send a session to past.dev.
---

# Send one session to past.dev

Send one session to past.dev right away, instead of waiting for it to go idle or for Codex to close.
It is usually the session just read with `$past:cut`.

If the user named a session (the last characters of its id, or a transcript path), run:

```
node ../../bin/past-hook.mjs ingest <session-or-path> --host codex
```

Otherwise send the session you are in:

```
node ../../bin/past-hook.mjs ingest --host codex
```

Run it from this skill's folder: the path is relative to it. This command reaches past.dev over the network and writes to `~/.past`, and Codex's sandbox blocks both. Request escalated permissions when you run it. If it answers that it ran inside the sandbox, run it again with escalated permissions.

The user asked for this send by invoking the skill, so do not ask again. Relay the output as it
is: what was sent and its cost, or why nothing was. A session still open is sent again when it
goes idle or when Codex closes, if it grew, and the output says so. To see the content first,
`$past:cut` shows exactly what would be sent.
