# Sessions and context

Source: <https://pi.dev/docs/latest/sessions>, `/json`, `/rpc-commands`, `/compaction` (checked 2026-11-04;
npm `@earendil-works/pi-coding-agent` 1.5.0). Installed `pi --help` wins on any difference.

Pi persists sessions automatically unless started with `--no-session`. Default location:
`~/.pi/agent/sessions/`, grouped by working directory. Storage override precedence (highest first):
`--session-dir`, `PI_CODING_AGENT_SESSION_DIR`, the `sessionDir` setting.

A session records the working directory it was created in and never changes it. Pi runs the session
there, so tools, `@path`, project configuration, and context files resolve from that directory.

```sh
pi --continue             # newest session for the current directory
pi --resume               # interactive selector (Tab includes other directories)
pi --session <path-or-id> # full/partial ID or file path; resumes in the session's own cwd
pi --session-id <id>      # open this directory's session with that ID, or create it
pi --fork <path-or-id>    # new session in the CURRENT directory from an existing one
pi --no-session           # in-memory, not resumable
pi --session-dir <dir>    # override storage and lookup
pi --name <name>          # display name (also /name; /session shows file, ID, tokens, cost)
```

| Entry point | Working directory used |
| --- | --- |
| `--continue` | current directory |
| `--resume`, `/resume`, `--session`, `/import` | the session's own directory (may differ from the current one) |
| `--fork`, or `--session <id>` from another directory that you choose to fork | current directory, in a new session |

If the stored directory no longer exists, interactive mode offers to run in the current directory and
asks again next time (the choice is not stored); other modes exit with an error.

Constraints: `--fork` cannot be combined with `--session`, `--continue`, `--resume`, or `--no-session`.
`--session-id` cannot be combined with `--session`, `--continue`, or `--resume` (combine it with
`--fork` to choose the new ID). IDs use letters, digits, `.`, `_`, `-` and must start and end with a
letter or digit.

## Branches

Entries are stored as a tree, so returning to an earlier point does not erase the branch you leave.

| Action | Result | Use when |
| --- | --- | --- |
| `/tree` | moves within the **same** session file | alternatives should stay together |
| `/fork` | creates a **new session** from an earlier user message | the alternative becomes separate work |
| `/clone` | copies the active branch into a **new session** | you want a separate copy of the current state |

`/fork` and `/clone` produce a new session; only `/tree` navigation stays in one file. In `/tree`,
selecting a user message puts its text back in the editor to create another branch. When you leave a
branch Pi can summarize it and attach the summary to the branch you enter.

## Context and compaction

The model receives the **active branch**, combined with the system prompt, discovered context files,
available tools, and loaded skill descriptions — not every branch in the file.

- Automatic compaction triggers when `contextTokens > contextWindow - reserveTokens`
  (default `reserveTokens` 16384; see <https://pi.dev/docs/latest/compaction>). It adds a summary and
  keeps recent messages; original entries are not deleted.
- `/compact [instructions]` compacts manually. Disabling automatic compaction does not disable it.
- Failures (provider down, summarization request rejected): fix the provider problem, run `/compact`.
- Compaction `reason` values in events: `manual`, `threshold`, `overflow`. Successful overflow
  recovery sets `willRetry: true` before Pi retries the prompt.
- Source files named by the docs: `packages/coding-agent/src/core/compaction/compaction.ts`,
  `branch-summarization.ts`, `utils.ts`, `session-manager.ts`.

## Completion semantics

`agent_end` closes one low-level agent run; automatic retry, overflow recovery, compaction retry,
steering, or follow-up work can still continue (`willRetry` flags a pending retry). `agent_settled`
(`{"aborted": boolean}`) means Pi has no remaining automatic work for that session-level run. Integrations
wait for `agent_settled`.

## Export, sharing, and bug reports

`/export` writes HTML or JSONL; `pi --export <input> [output]` converts a session file to HTML. `/share`
uploads and returns a viewer link: a Radius artifact when Radius authentication is configured, otherwise
a private GitHub gist; the `sessionShareDestination` setting (`gist` or `radius`) forces one. `/bug
[description]` prepares a private report (optional transcript, review first). All of these can disclose
prompts, model output, tool arguments, command output, file contents, and extension messages — review
before sending. `--no-session`/`SessionManager.inMemory()` avoids local persistence but does not stop
provider transmission or tool access.

## Session file format

Source: `docs/session-format.md` in the 1.5.0 npm tarball (verified 2026-11-04). Exact types live in
`packages/coding-agent/src/core/session-manager.ts` (repo `earendil-works/pi`); inspect
`node_modules/@earendil-works/pi-coding-agent/dist/` for the installed shape.

- JSONL, one JSON object per line with a `type`. Path:
  `~/.pi/agent/sessions/--<path>--/<timestamp>_<session-id>.jsonl`; `<path>` = cwd without the leading
  separator, with `/`, `\\` and `:` replaced by `-`. `<session-id>` is a UUID unless set via the SDK or
  `--session-id`.
- The first line is the header (`type: "session"`, `version`, `id`, `timestamp`, `cwd`, optional
  `parentSession`). It is not part of the tree. Version history: v1 linear (legacy), v2 tree with
  `id`/`parentId`, v3 renamed the `hookMessage` role to `custom`. Older files migrate to **v3** on load.
- Every other entry has `type`, `id` (usually 8-char hex, may fall back to a UUID), `parentId`
  (`null` for a root) and an ISO `timestamp`. Entry timestamps are ISO strings; the nested
  `message.timestamp` is Unix milliseconds. Entries form a tree, so branching happens in place.
- Entry types: `message` (an `AgentMessage`: system/user/assistant/toolResult), `model_change`,
  `thinking_level_change`, `usage` (model-attributed usage outside the context, e.g. `kind: "cache_warm"`;
  unknown `kind` values must be treated as normal usage), `compaction` (`summary`, required
  `firstKeptEntryId`, `tokensBefore`, optional `systemMessage`, `usage`, `details`, `fromHook`),
  `context_edit` (`targetId`, `replacement: null` omits the target from future context only; the latest edit
  on the active branch wins), `branch_summary` (`fromId`, `summary`), `custom` (extension state, **not** in
  LLM context), `custom_message` (extension message that **is** in context; `display`), `label`,
  `session_info` (name).
- System messages carry the prompt and tool loadout as `sections` patches plus `toolsAdded`/`toolsRemoved`;
  replaying them in order yields the current prompt and tools (no separate prompt-state entry). Newer
  assistant messages also record `thinkingLevel`.
- Virtual-model router state is a `custom` entry with `customType` `pi.virtual-model-state`.
- Context building: `buildContextEntries()` walks leaf → root honoring the latest compaction;
  `buildSessionProjection()` applies `context_edit`; `buildSessionContext()` produces the LLM messages.
  Entries can have several roots after `resetLeaf()` or `branchWithSummary(null, …)`.
- Delete sessions by removing the `.jsonl` file, or in `/resume` with Ctrl+D (uses the `trash` CLI when
  available).
- **[rec]** Parse defensively: switch on `type` with a default branch, since new entry types appear in minor
  releases (0.87.0 added `context_edit`).

## Unverified / open

- Whether the `version` header will move beyond 3 in later releases (not stated in the docs).
