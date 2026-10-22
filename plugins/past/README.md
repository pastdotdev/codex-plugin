# past.dev for Codex

Your Codex sessions become memory, and every session opens already briefed.

It runs in the Codex CLI and in local threads of the ChatGPT desktop app, on macOS and Linux.

## Install

In a terminal:

```
codex plugin marketplace add pastdotdev/codex-plugin
codex plugin add past@past
```

In the ChatGPT desktop app, with no terminal: open **Plugins**, then **Manage**, then
**Marketplace**, add a marketplace with the source `pastdotdev/codex-plugin`, and install **past.dev**
from its page.

## Trust the hooks

Codex runs no hook until you have trusted it, and says nothing about the ones it skips, so until
this step the plugin does nothing. In a terminal session, `/hooks` lists the five past.dev hooks and
trusts them. In the desktop app, the past.dev plugin's page offers **Trust all**, and so does
**Settings › Hooks**; if those pages show no hooks, run `/hooks` once in a terminal session, since
the trust is shared. An update of the plugin keeps the trust.

## Connect

```
$past:connect <project-api-key> <identity> [api-url]
```

The project API key starts with `past_sk_` and comes from **Build › API keys** in the
[console](https://app.past.dev) (not the project's id, and not the organization's `past_mk_`
management key); the identity is who every memory is attributed to, and the API URL is for
self-host only. Codex asks before it runs this outside its sandbox; approve it.

To keep the key out of the conversation, set `PAST_API_KEY` and `PAST_IDENTITY` in your
environment instead. The desktop app reads them from a login shell, so put them in `~/.zprofile`.

The connection lives in `~/.past/config.json`. If this machine is already connected to past.dev, there
is nothing to do: `$past:status` shows it.

## What it does

| Hook | When | What |
|---|---|---|
| `SessionStart` | a session opens | recalls this project's history and puts it in context |
| `UserPromptSubmit` | every prompt | recalls the ten memories closest to the prompt and puts them in context |
| `Stop` | after each reply | stamps the session's activity; sends nothing itself |
| `PreCompact` | the context is about to be compacted | sends the session so far, before the detail is summarised away |
| `SessionEnd` | Codex unloads a session | makes sure the session will be sent; sends nothing itself |

**When a session is sent.** Codex says a session ended even when it only went quiet for half an
hour, so the plugin does not send on that. A session is sent within a minute after the Codex
process that ran it exits (the end of a `codex exec` run, the desktop app when you quit it), or
after 12 hours without a new turn (`idleMinutes`). It is sent before a compaction too.

`SessionStart` also sends any earlier session that never reached past.dev and has gone idle, which is
what covers a crash or a machine that slept.

**A session is sent in sittings.** A new sitting starts where the session went quiet for 30
minutes (`sittingMinutes`): nobody wrote and Codex ran nothing. Each sitting is dated at its own
first turn, and what past.dev remembers takes the date of the sitting it came from, so something said
on the third day of a thread that ran for three days is dated that day. A prompt and its reply stay
together while Codex works on it, however long. When a session continues, only its last sitting
and the new ones are sent again.

**Imported sessions.** Codex can import your sessions from another agent, and a fork of such a
thread repeats it. The plugin never sends the copied part of a thread: it is that agent's
conversation, and Codex dates the copy on the day it was made. What you say in the thread
afterwards is sent like any Codex session.

## What leaves this machine

The prompts, the replies, and the project and branch names. That is all.

Not sent: tool calls and their output, file contents, diffs, command output, subagent threads, and
anything Codex or a hook injected rather than a person typing it. Strings that look like keys,
tokens or private keys are replaced with `[redacted]` before anything is sent.

Run `$past:cut` to read exactly what would be sent, before you send it.

## Skills

Codex plugins carry skills rather than commands. Name one with `$`:

| Skill | Does |
|---|---|
| `$past:connect` | store the key, identity and audience in `~/.past/config.json`, mode `600` |
| `$past:status` | what it is connected to, when its hooks last ran, what it has sent |
| `$past:cut [session]` | print exactly what would be sent from this session, or from one you name |
| `$past:ingest [session]` | send one session now |
| `$past:backfill` | list and estimate, then send, the Codex sessions already on this machine |
| `$past:search` | search past.dev from the terminal, for you; Codex uses its `past_recall` tool for itself |

**And a recall tool.** The plugin also serves `past_recall`, on a small MCP server of its own
(`.mcp.json`, run by the same script). The block before a prompt holds what was recalled for it;
when the request depends on something that block does not hold, Codex calls `past_recall` with a
question of its own and reads up to a hundred dated memories, within 2,000 tokens. Each call is one
recall. The server runs outside Codex's sandbox, so it reaches past.dev without asking, and it only
reads, so its tool is approved in advance. `codex mcp list` shows it as `past`.

Codex runs a command in a sandbox with no network. The skills that reach past.dev ask for the
permission, and Codex asks you. `connect`, `ingest` and `backfill` run only when you name them.

Without an audience everyone in the project sees what you send. `$past:connect --audience <slug>`
narrows it to an audience from the console; `--audience project` widens it back.

## Configuration

`~/.past/config.json`:

```json
{
  "apiKey": "past_sk_…",
  "identity": "you@example.com",
  "apiUrl": "https://api.past.dev",
  "audience": "developers",
  "recall": true,
  "ingest": true,
  "idleMinutes": 720,
  "sittingMinutes": 30,
  "deny": ["/Users/you/clients/acme"]
}
```

- `audience` is an audience slug from the console; leave it out and the whole project sees what
  you send.
- `recall: false` stops the reading and keeps the sending. `ingest: false` does the opposite.
- `idleMinutes` is how long a session stays quiet before it is sent (720, twelve hours, by default).
- `sittingMinutes` is how long a session stays quiet before what follows starts a new sitting (30
  by default, 5 at least). Shorter dates what past.dev remembers more closely, in more and smaller
  data points. A session keeps the length it was first sent with.
- `deny` holds path fragments. A session whose working directory matches one is ignored
  completely — nothing is read and nothing is sent.
- `PAST_API_KEY`, `PAST_IDENTITY` and `PAST_API_URL` override the file.
- Self-host: point `apiUrl` at your own deployment. Nothing else changes. Use `https`: over plain
  `http` to another machine the key travels unencrypted, and `connect` and `status` warn about it.

## If nothing is recalled

Ask for `$past:status`. It says when the hooks last ran. If they never ran, they are not trusted
yet. If they are trusted and still never run, Codex did not find `node`: it runs hooks in a login
shell, which does not read `~/.zshrc`.

## Requirements

Node 16 or later, on the path of a login shell.

## Where it keeps things

Everything lives in `~/.past/`, a folder only you can open: `config.json` (the connection),
`state.json` (what was sent, as content hashes), `activity/` (one small file per open session, for
the idle timer) and `hosts/` (when the hooks last ran).

## Removing it

`codex plugin remove past@past`, then delete `~/.past`. Revoking the key in **Build › API keys**
stops it immediately from the server side. What was already sent stays until you delete it in the
console.

## License

MIT — see [LICENSE](LICENSE).
