# past.dev for Codex — working on the plugin

The past.dev plugin for Codex, and the marketplace that carries it. It runs on the developer's machine
and talks to past.dev's public Memory API. Nothing here is privileged: anyone can write the same
connector against the same endpoints with the same key. What a user reads is in `README.md`, and
in full in `plugins/past/README.md`; this file is the rules the code keeps, for whoever changes it.

## How it fits together

```
Codex
  hooks/hooks.json   five hooks          every one passes --host codex
  skills/*           six skills          every command passes --host codex
            |
  bin/past-hook.mjs <mode> --host codex
            |
  session start   brief        recall into context; sends idle sessions left behind
  every prompt    prompt       recall into context
  every turn      stop         stamps the activity file, keeps one waiter on the session
  compaction      precompact   sends the session so far
  session end     end          makes sure a waiter runs
  detached        idle         the waiter: sends once Codex's process is gone, or after idleMinutes
            |
  ~/.past/config.json          the connection
  ~/.past/state.json           what was sent and what is open, under hosts.codex
  ~/.past/activity/codex/      one file per open session
  ~/.past/hosts/codex.json     when the hooks last ran, for status
            |
  POST /api/v1/recall · POST /api/v1/ingest/batch · GET /api/v1/audiences/{slug}
```

```
.agents/plugins/marketplace.json  the catalog — this repository is the marketplace
plugins/past/
  .codex-plugin/plugin.json       the manifest: names the hooks file and the skills folder
  hooks/hooks.json                SessionStart, UserPromptSubmit, Stop, PreCompact, SessionEnd
  bin/past-hook.mjs               the whole connector, one file, no dependencies
  skills/                         search, status, connect, cut, ingest, backfill
  LICENSE                         the same license: an install copies this folder and nothing else
.github/workflows/check.yml       every pull request: the script parses and runs, the manifests parse
```

## The script

`bin/past-hook.mjs` is the whole plugin. The rules it keeps:

1. **A hook never breaks the session.** Every mode exits 0, prints valid JSON or nothing, and
   swallows its own failures. `$past:status` is where a person learns something is wrong, so a
   failed send is kept in `state.json` (`lastError`) for it.
2. **Only prose travels.** On disk a session is mostly tool output. Tool calls and their output,
   file contents, diffs, subagent threads and injected turns are cut: sending the transcript would
   cost far more and recall worse, with the conversation buried in it.
3. **The hash gates the send.** `state.json` holds, per session, the hash of the whole session
   rendered in one piece (`hash`) and one hash per sitting (`sittings`). A session whose whole hash
   has not changed is not sent; one that changed sends only the sittings whose hash differs. The
   server charges nothing for an unchanged re-send, but the call still spends a request from the
   key's per-minute budget.
4. **A session is sent when it is over, goes idle or compacts; never from `Stop`.** `Stop` fires
   every turn: it only stamps `~/.past/activity/codex/<session>.json` and keeps one detached waiter
   (`idle` mode) per session. The waiter sleeps in one-minute slices measured on the wall clock,
   since a timer does not count the time the machine is asleep, and sends once Codex's process is
   gone or the last turn is `idleMinutes` old (720 by default). `ensureWaiters` (`Stop`,
   `SessionStart`, `$past:status`) replaces a waiter that died or that an older `WAITER_VERSION`
   started (bump it whenever the waiter changes), and a replacement past its deadline sends at
   once. `PreCompact` sends before the context is summarised. `SessionStart` sends the idle
   sessions that never reached past.dev, and `tidyPending` drops an open session that is gone, empty,
   or in past.dev unchanged and idle. Why `SessionEnd` sends nothing is under "Codex" below.
5. **A recall query is redacted like the content it would send.** It crosses the network exactly
   as ingested content does, and both go through `redact()`.
6. **Backfill estimates before it sends, and never sends the session it runs in.** `backfill`
   lists; `backfill --confirm` sends them all, leaving out the current thread: the hooks send it,
   and a second send would pay for it twice. One session is `ingest`, which sends at once (running
   it is the consent). `cut` defaults to the current thread, then to the most recently written
   transcript.
7. **`state.json` changes only through `updateState`.** Hooks and waiters are separate processes
   running at once. `updateState` takes a lock (`~/.past/state.lock`, a directory, taken over after
   5 s, skipped after 3 s so a hook never hangs), re-reads the file, applies the change and renames
   the new file into place. Never save a copy read earlier: two waiters that sent together would
   erase each other's "sent". A file that does not parse is read again a few times and, if it still
   does not, left as it is: written over, it would lose every session, and the next backfill would
   send them all again.
8. **No dependencies.** Node 16 or later, standard library only, so installing is a clone. `fetch`
   is used where Node has it and the `http` modules where it does not: Codex runs hooks in a login
   shell, whose `node` is often an old one. On Node 16 a stalled `http` call is destroyed when its
   time is up, so it never holds a prompt or a waiter's send. Nothing in the script is newer than
   Node 14 can parse (no `||=`), so on an older Node the hooks stay silent rather than fail.
9. **What is sent for a session does not change by a byte.** The heading and the speaker names
   that `renderTranscript` prints are part of the content hash, and so are `SOURCE`, `AGENT` and
   `SPEAKER` at the top of the script: a changed byte makes `backfill` send every session already
   in past.dev again, as a paid revision. `sittingId` and the cut (`sittingsOf`, `silenceMeter`) are
   fixed too: changed, they move the boundaries of every session that grows afterwards, and the old
   sittings stay in past.dev beside the new ones.
10. **`~/.past` is the person's alone.** The folder is `700` and every file in it is created
    `600`, never written open and narrowed afterwards. A plain-http `apiUrl` on another machine
    sends the key unencrypted: `connect` warns, and `status` says so on its first line.
11. **A session is sent in sittings.** past.dev dates every memory at the time of its data point, and
    decides which of two statements is the later from those times. A sitting starts at the first
    turn after the thread was silent for `sittingMinutes` (30 by default, 5 at least), whoever
    speaks; tool calls and their output count as activity, so a prompt stays with its reply while
    Codex works. Each sitting is one data point timed at its first turn, and every changed sitting
    goes in one `/api/v1/ingest/batch` call. `state.json` pins `sittingMinutes` per session at its
    first send, so a changed value never moves the boundaries of a session already in past.dev.

12. **The product is written past.dev.** Every sentence a person reads says past.dev: the READMEs,
    the manifest's display names, the skills' text, what a mode prints. Identifiers keep `past`:
    the plugin and marketplace names (`past@past`, which every hook's trust is keyed on), the skills
    (`$past:status`), `~/.past`, the key prefix `past_sk_` and the data point ids. So does the recall
    block's header, `=== past · recalled from memory ===`: the script drops a block by that exact
    text from the sessions it sends, so a changed header would change how old sessions render and
    send them all again (rule 9). The hooks file keeps its status message, "Recalling from past", for
    the reason it is written once (see "The hooks file is written once").

## Beside other past.dev plugins

`~/.past` can be shared with another past.dev plugin on the same machine. `config.json` is then one
connection for both. This plugin keeps its sessions under `hosts.codex` in `state.json`, its
activity files in `activity/codex/` and its hook stamp in `hosts/codex.json`. The rest of
`state.json` is never read here, and every write puts it back as it was (rule 7).

## Codex

- **The hooks file is written once.** Codex runs no hook that a person has not trusted, and keeps
  the trust against a hash of each hook's definition, under a key made of the plugin's id, the
  hooks file's path in the package, the event and the hook's position
  (`past@past:hooks/hooks.json:stop:0:0`). A new version of the plugin and a changed script keep
  the hash. An edited definition changes it, and that one hook stops without a word until the
  person reviews it again. A renamed plugin or marketplace, a moved hooks file and a reordered
  entry change the key. So `hooks/hooks.json` holds the five events and a mode each, its path and
  the name `past@past` stay, and every change of behaviour goes into the script. Every hook in it
  passes `--host codex`, so the flag stays too: the script takes it out before reading its
  arguments, and the skills pass it as well, so a command reads the same wherever it comes from.
- **`SessionEnd` is a hint.** Codex also fires it when a thread has sat idle for half an hour. A
  send there would be a new revision at every pause, so `end` only makes sure a waiter runs, and
  the waiter decides: the owner is gone, or `idleMinutes` have passed. A session with no activity
  file had no turn since its last send, and gets no waiter.
- **Only Codex counts as the owner.** Any other process above a hook may last one turn, and a
  waiter that trusted it would send the session after every turn. A `codex exec` run ends with its
  session. The desktop app holds one Codex process for every thread, for as long as it is open, so
  there the owner goes when the app quits.
- **Codex reads only what its manifest names.** `hooks` and `skills` in
  `.codex-plugin/plugin.json` name the hooks file and the skills folder. A `plugin.json` at the
  package root would take precedence, and Codex would then load no hook.
- **A skill's command runs in the skill's folder.** So each skill calls the script by a relative
  path, `node ../../bin/past-hook.mjs <mode> --host codex`, and the project and the branch of a
  session come from its transcript (`ownProject`), since the directory says nothing. Codex
  substitutes nothing in a skill file.
- **The model's commands are sandboxed; hooks are not.** No network unless the person allowed it,
  no writing outside the workspace. A skill whose command reaches past.dev or writes `~/.past` tells
  Codex to ask for the permission. A mode that finds itself inside the sandbox
  (`CODEX_SANDBOX_NETWORK_DISABLED`) prints `NEEDS_OUTSIDE`, which tells the model to run it again
  outside: a recall that could not leave must never read as "nothing matches". `status` starts no
  waiter from inside the sandbox, since that waiter could never reach past.dev.
- **The three skills that take a key or send data run only when named** (`connect`, `ingest`,
  `backfill`: `allow_implicit_invocation: false` in their `agents/openai.yaml`).
- **The current session** is the thread Codex names in the environment of every command it runs
  (`CODEX_THREAD_ID`). `cut` and `ingest` act on it when no session is named; `backfill` leaves it
  out.
- **The reader** keeps `response_item` messages of the user and the assistant. It leaves out the
  developer's role (Codex's instructions, what hooks injected, past.dev's own recall block), the blocks
  Codex writes under the user's role, the prompts that only run one of the plugin's skills, and
  every line that is machinery. A recent Codex says what each user message is made of, and only
  typed text (`user.text`) counts. A transcript that does not say (an older Codex, a legacy
  thread) is filtered by `CODEX_SYNTHETIC`: the repository's AGENTS.md, a skill's whole file and a
  shell command a person ran, with its output, all arrive under the user's role. A compaction
  appends one line and rewrites nothing. A session is found by its thread id, in `sessions/` and
  in `archived_sessions/`, since archiving moves the file.
- **A subagent's thread is never read.** Its header names a subagent as its source, its prompts
  are the parent's instructions, and it is neither listed nor sent, from a hook or a command. A
  source that names a custom client is a conversation like any other.
- **A copied conversation is never sent.** Two kinds of thread open with somebody else's session,
  stamped with the one instant the copy was made. Codex can import another agent's sessions as
  threads of its own and lists those in `external_agent_session_imports.json`. A fork of a legacy
  thread repeats the parent's header and turns (a fork of a recent thread points at its parent and
  copies nothing). The reader drops what was written within `COPY_WRITE_MS` of that instant, and
  what a person says in the thread afterwards is this session. Only the thread's first header
  names its project and branch.

## Contracts this depends on

- `POST /api/v1/ingest/batch` — one item per sitting. `id` gives replacement (`codex:<thread id>`
  for the first sitting, `codex:<thread id>:<n>` for the others), `timestamp` dates every memory
  drawn from the item, `label` names the session, `metadata` carries `conversationId`,
  `sessionId`, `project`, `gitBranch`, `sitting`.
- `POST /api/v1/recall` — **`identity` is required**. Recall stays off until `$past:connect` sets
  one.
- `audience` on ingest — a slug; absent means the whole project. `connect --audience` checks it
  with `GET /api/v1/audiences/{slug}`.
- Codex gives a hook `session_id`, `transcript_path` and `cwd`, the prompt as `prompt`, and takes
  `hookSpecificOutput.additionalContext` at session start and on a prompt.

If any of these contracts changes, this file changes with it.

## Testing

By hand: there is no test suite. CI parses the manifests on every pull request, with the script
parsed on Node 14 and run on Node 16 and 22. From the repository root:

```bash
node plugins/past/bin/past-hook.mjs cut --host codex        # what would be sent, from a real transcript
node plugins/past/bin/past-hook.mjs backfill --host codex   # the estimate, sending nothing
```

For the hooks and the waiter, point a throwaway home at a local stand-in for the API, so nothing
touches your real `~/.past` or a real project: `HOME=/tmp/past-test` with a `config.json` whose
`apiUrl` is `http://127.0.0.1:<port>`, a few lines of Node answering `POST /api/v1/ingest/batch`, and the
mode under test fed its stdin JSON by hand.

On a real Codex:

```bash
codex plugin marketplace add .        # this folder is the marketplace
codex plugin add past@past
codex exec --dangerously-bypass-hook-trust "a prompt"     # runs the hooks for this one run, untrusted
```

`codex exec` waits on its input unless it is closed (`< /dev/null`), and fires `SessionEnd` at the
end of every run. `CODEX_HOME=<a folder>` gives Codex a home of its own, enough for everything that
calls no model: installing, and asking the app server what it thinks of the hooks (`codex
app-server`, then `hooks/list`), which is how to see a hook's key, its hash and whether it is
trusted. The ChatGPT desktop app carries the same `codex` inside its bundle, for a machine that has
no terminal install.
