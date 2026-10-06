---
name: connect
description: Connect Codex on this machine to a past.dev project with a project API key and an identity. Use only when the user asks to connect past.dev or to change its audience.
---

# Connect past.dev

The user gives a project API key (it starts with `past_sk_`), the identity every memory is
attributed to, optionally an API URL for a self-hosted deployment, and optionally an audience.

If a key and an identity are both given, run:

```
node ../../bin/past-hook.mjs connect <project-api-key> <identity> [api-url] [--audience <slug>] --host codex
```

If only an audience is given, run `node ../../bin/past-hook.mjs connect --audience <slug> --host codex`: it changes who
sees what is sent from now on and keeps the key. `--audience project` goes back to the whole project.

Run it from this skill's folder: the path is relative to it. This command reaches past.dev over the network and writes to `~/.past`, and Codex's sandbox blocks both. Request escalated permissions when you run it. If it answers that it ran inside the sandbox, run it again with escalated permissions.

Never repeat the key back, and never write it into a file in the repository.

If the key or the identity is missing, and this is not an audience-only change, do not ask for
the key in the conversation. Tell the user the ways to connect instead:

- **Already connected on this machine**: there is nothing to do. The connection lives in
  `~/.past/config.json`, which every past.dev plugin on the machine reads. `$past:status` shows it.
- **This skill with its arguments**: `$past:connect <project-api-key> <identity> [api-url]`. The
  key is redacted before a session is sent to past.dev, but it stays in this machine's local transcript.
- **Two environment variables** in the shell profile, which keeps the key out of every transcript:
  `PAST_API_KEY` and `PAST_IDENTITY` (and `PAST_API_URL` for a self-hosted deployment).

The **audience** decides who in the project sees the sessions sent from this machine. Without
one, everyone in the project does. It is an audience slug from the console (**Audiences**), and
the command checks that it exists. When the user connects without one, say so in one line, and
how to narrow it later: `$past:connect --audience <slug>`.

The **project API key** is created in the past.dev console under **Build > API keys** and shown
once. It is not the project's id or handle, and not the organization's management key
(`past_mk_...`). Recall stays off without an identity.

After it succeeds, tell the user three things: Codex runs the past.dev hooks only once they are
trusted (`/hooks` in a terminal session, or **Trust all** on the past.dev plugin's page in the ChatGPT
desktop app); new sessions are read from then on; and `$past:backfill` sends the Codex history
already on this machine.
