---
name: cut
description: Show exactly what past.dev would send from this Codex session, or from one the user names. It sends nothing. Use when the user asks what past.dev sends, or wants to read a session before sending it.
---

# What past.dev would send

Show what would leave the machine for one session. It sends nothing.

If the user named a session (the last characters of its id, as the listings print them, or a
transcript path), run:

```
node ../../bin/past-hook.mjs cut <session-or-path> --host codex
```

Otherwise run it with nothing named, and it takes the session you are in:

```
node ../../bin/past-hook.mjs cut --host codex
```

Run it from this skill's folder: the path is relative to it. It only reads, so it runs inside the sandbox.

Show the user the result. This is what leaves the machine: the prompts and the replies, with
secrets redacted. Tool output, file contents, diffs and what Codex or a hook injected are not in
it. A session is sent in sittings, one data point each: a new one starts where the conversation
went quiet for `sittingMinutes` (30 by default), and each begins with its own heading. The first line names the
session and its sittings, and reports the size on disk, the size sent and what sending it now
costs. The last line says what happens next. Relay that line as it is.

Then, unless the last line says the session is already in past.dev, ask the user one yes/no
question: **"Send this session to past.dev now? About N credits."**, with N from the first line. If
they say yes, use `$past:ingest` with the session id from the first line. If they say no, stop
there; the session is still sent when it goes idle or when Codex closes.
