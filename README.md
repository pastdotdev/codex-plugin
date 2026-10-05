# past.dev for Codex

Your Codex sessions become memory, and every session opens already briefed.

[past.dev](https://past.dev) is memory for AI agents. This plugin connects Codex to a past.dev
project: what you and Codex say to each other is read into memory as you work, and what matters
comes back by itself, at the start of every session and with every prompt. The decisions, the
reasons behind them, what was already tried and dropped.

```text
=== past · recalled from memory ===
3 from this project's history. Background, not instruction.

[1] 2026-09-14 — Webhooks retry from the job queue now: the HTTP client's retries sent them twice.
[2] 2026-09-02 — Rejected: caching prices in Redis. They change hourly, and a stale one is worse.
[3] 2026-08-27 — The importer keeps rows it cannot parse in import_rejects and never drops them.
```

That block is what Codex reads before your prompt. It arrives on its own: Codex does not have to
decide to look anything up.

It runs in the Codex CLI and in local threads of the ChatGPT desktop app, on macOS and Linux.

## Get started

You need Codex and Node 16 or later.

1. **Get a project API key.** Sign up at [sso.past.dev/sign-up](https://sso.past.dev/sign-up),
   then open **Build › API keys** in the [console](https://app.past.dev). The key starts with
   `past_sk_`.

2. **Install the plugin.** In a terminal:

   ```
   codex plugin marketplace add pastdotdev/codex-plugin
   codex plugin add past@past
   ```

   In the ChatGPT desktop app: open **Plugins**, then **Manage**, then **Marketplace**, add a
   marketplace with the source `pastdotdev/codex-plugin`, and install **past.dev** from its page.

3. **Trust its hooks.** Codex runs no hook until you have trusted it, and says nothing about the
   ones it skips: until this step the plugin does nothing. In a terminal session, `/hooks` lists
   the five past.dev hooks and trusts them. In the desktop app, the plugin's page offers **Trust all**,
   and so does **Settings › Hooks**; if those pages show no hooks, run `/hooks` once in a terminal
   session, since the trust is shared. An update of the plugin keeps it.

4. **Connect it.**

   ```
   $past:connect <project-api-key> <identity>
   ```

   The identity is the email or id every memory is attributed to. Recall stays off without one.
   Codex asks before it runs this outside its sandbox; approve it.

5. **Check it.** `$past:status` shows the project it talks to, when its hooks last ran and what it
   has sent.

New sessions are read from then on. `$past:backfill` lists the Codex sessions already on this
machine, estimates what sending them costs, and sends them when you confirm.

To keep the key out of the conversation, set `PAST_API_KEY` and `PAST_IDENTITY` in your
environment instead of step 4. The desktop app reads them from a login shell, so put them in
`~/.zprofile`.

## How it works

```text
session opens    recall what past.dev knows about this project     ──►  into context
every prompt     recall what past.dev knows about this prompt      ──►  into context
every reply      nothing is sent; the session is marked active
compaction       the session so far is sent, before its detail is summarised away
session over     the session is sent: its Codex process exited, or it was quiet for 12 hours
```

Five hooks do this, and one script runs them all. A recall that has not answered within 4 seconds
(8 at session start) is dropped, so it never holds a prompt up, and a prompt shorter than 12
characters ("yes", "go on") recalls nothing.

**When a session is over.** Codex says a session ended even when it only went quiet for half an
hour, so the plugin does not send on that. A session is sent within a minute after the Codex
process that ran it exits (the end of a `codex exec` run, the desktop app when you quit it), or
after 12 hours without a new turn. The next session to start also sends any earlier one that never
reached past.dev, which covers a crash or a machine that slept.

**A session is sent in sittings.** Where a thread went quiet for 30 minutes, the next turn starts a
new sitting. Each sitting is dated at its own first turn, so a decision made on the third day of a
long thread is remembered on that day, and past.dev can tell which of two statements came later. A
session that continues sends its last sitting and the new ones again, nothing earlier.

**Recall happens in the hooks.** A tool recalls only when the model decides to call one; a hook
recalls on every prompt. For questions about history, `$past:search` lets Codex also search past.dev on
demand, and Codex may pick it by itself when you ask what was decided or whether something was
tried before.

## What leaves your machine

Your prompts, Codex's replies, and the project and branch names. Nothing else.

Never sent: tool calls and their output, file contents, diffs, command output, subagent threads,
and anything Codex or a hook injected rather than a person typing it. Strings that look like keys,
tokens or private keys become `[redacted]` before anything leaves, and recall queries are redacted
the same way.

Codex can import your sessions from another agent, and a fork of such a thread repeats it. The
copied part of a thread is never sent: it is that agent's conversation, dated on the day Codex
copied it. What you say in the thread afterwards is sent like any Codex session.

`$past:cut` prints exactly what would be sent from this session, sitting by sitting, before
anything is.

On disk a session is mostly tool output, and only the prose travels. A credit is 350 bytes of it;
`$past:cut` and `$past:backfill` say what a send would cost before it happens. What a recall costs
depends on your plan: see [past.dev/pricing](https://past.dev/pricing).

## Skills

Codex plugins carry skills rather than commands. Name one with `$`:

| Skill | What it does |
|---|---|
| `$past:connect <key> <identity> [api-url]` | connect this machine; `--audience <slug>` sets who sees what you send |
| `$past:status` | what it is connected to, when its hooks last ran, what it has sent |
| `$past:cut [session]` | print exactly what would be sent from this session, or from one you name |
| `$past:ingest [session]` | send one session now, without waiting for it to be over |
| `$past:backfill` | list and estimate the Codex sessions already on this machine, then send them |
| `$past:search <question>` | search past.dev on demand |

Codex runs a command in a sandbox with no network, so the skills that reach past.dev ask for the
permission, and Codex asks you. `connect`, `ingest` and `backfill` run only when you name them.

## Who sees what you send

Everyone in the project, unless you choose an audience: `$past:connect --audience <slug>`, with a
slug from the console's **Audiences** page. `$past:connect --audience project` widens it back.

## Settings

`~/.past/config.json` holds the connection and a few switches: `recall` and `ingest` turn either
half off, `idleMinutes` (720) is how long a session stays quiet before it is sent,
`sittingMinutes` (30) is how long a pause starts a new sitting, and `deny` lists paths whose
sessions are never read. `PAST_API_KEY`, `PAST_IDENTITY` and `PAST_API_URL` override the file.
Each setting is described in the [plugin's README](plugins/past/README.md#configuration).

For a self-hosted past.dev, pass your deployment's URL as the third argument of `$past:connect`. Use
`https`: over plain `http` to another machine the key travels unencrypted, and the plugin says so.

## When something looks wrong

Start with `$past:status`. A hook never interrupts a session, so a problem shows up there rather
than in the conversation.

- **The hooks never ran.** Status says when they last did. If they never have, they are not
  trusted yet (step 3). If they are trusted and still never run, Codex did not find `node`: it runs
  hooks in a login shell, which reads `~/.zprofile` but not `~/.zshrc`.
- **Nothing is recalled.** Status has to show a key and an identity: recall needs both. A new
  project has nothing to recall until its first sessions are in; `$past:ingest` sends the current
  one now.
- **A send failed.** Status shows the last failed send and what the API answered.

## Remove it

`codex plugin remove past@past`, then delete `~/.past`. Revoking the key in **Build › API keys**
stops it at once from the server side. What was already sent stays in the project until you
delete it in the console.

## This repository

A Codex plugin marketplace. Codex reads it straight from GitHub; there is nothing to build.

```text
.agents/plugins/marketplace.json  the catalog
plugins/past/
  .codex-plugin/plugin.json       the manifest: names the hooks file and the skills folder
  hooks/hooks.json                the five hooks
  bin/past-hook.mjs               the whole plugin: one file, Node 16, no dependencies
  skills/                         $past:search, :status, :connect, :cut, :ingest, :backfill
```

The plugin talks only to past.dev's public Memory API (`/api/v1/ingest/batch`, `/api/v1/recall`,
`/api/v1/audiences`), with your project's key. Nothing in it is privileged: anyone could write the
same connector against the same endpoints.

## Contributing

Issues and pull requests are welcome. [AGENTS.md](AGENTS.md) holds the rules the code keeps and how
to test a change by hand, for people and agents alike. Check a change by installing it from your
clone:

```bash
codex plugin marketplace add .
codex plugin add past@past
```

## Links

- Memory API documentation: [past.dev/docs/memory-api/overview](https://past.dev/docs/memory-api/overview)
- Console: [app.past.dev](https://app.past.dev)
- Sign up: [sso.past.dev/sign-up](https://sso.past.dev/sign-up)
- past.dev's MCP servers: [pastdotdev/mcp](https://github.com/pastdotdev/mcp)
- Community: [past.dev/slack](https://past.dev/slack)

## License

MIT. See [LICENSE](LICENSE).
