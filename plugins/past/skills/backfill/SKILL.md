---
name: backfill
description: Send the Codex history already on this machine to past.dev, after showing the estimate. Use only when the user asks to backfill or to send their existing Codex sessions to past.dev.
---

# Send the history already on this machine

Send every unsent Codex session already on this machine to past.dev. One session is `$past:ingest`.

1. Run `node ../../bin/past-hook.mjs backfill --host codex` first. It sends nothing, and runs inside the sandbox. It
   lists each session (date, id, prose size, credits, sittings, first prompt) and the total. Each
   sitting is one data point. The session you
   are in is always left out: it is sent when it goes idle or when Codex closes.
2. Show the user that list and **ask them to confirm**. Sending history to a server is not
   something to do on their behalf. If they want only some of it, `$past:ingest <id>` sends one.
   If the output says that sessions imported from another agent are left out, relay that line as
   it is: those are copies Codex made of another agent's sessions, dated the day of the import.
3. Only if they agree, run the same command with `--confirm` added. This command reaches past.dev over the network and writes to `~/.past`, and Codex's sandbox blocks both. Request escalated permissions when you run it. If it answers that it ran inside the sandbox, run it again with escalated permissions.

Run it from this skill's folder: the path is relative to it.

`--all` covers every project on the machine instead of the one this session works in. Include it
in step 1 as well, so that the estimate matches what step 3 sends.
