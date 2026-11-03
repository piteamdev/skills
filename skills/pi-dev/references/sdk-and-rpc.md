# SDK and RPC

> Sources: `docs/sdk.md`, `docs/rpc.md`, `docs/rpc-commands.md`, `docs/json.md`, `docs/cli-integration.md` from the
> `@earendil-works/pi-coding-agent` 1.5.0 npm tarball (mirrors https://pi.dev/docs/latest). Verified 2026-11-04.

## Choose the correct boundary

| Requirement | Use |
| --- | --- |
| Node.js/Bun process, direct typed API | `@earendil-works/pi-coding-agent` SDK |
| Python, Go, an IDE, custom UI, or a subprocess boundary | `pi --mode rpc` |
| One input, final output, shell integration | `pi --print` |

Never answer exact SDK shapes from memory. Read the installed `dist/*.d.ts` and current checked
examples, then compile a nontrivial TypeScript recipe. Pi's SDK evolves quickly.

## SDK baseline

```ts
import { createAgentSession, SessionManager } from "@earendil-works/pi-coding-agent";

const { session } = await createAgentSession({
  cwd: "/path/to/project",
  sessionManager: SessionManager.inMemory(),
  tools: ["read", "grep", "find", "ls"],
});

const unsubscribe = session.subscribe((event) => {
  if (event.type === "message_update" && event.assistantMessageEvent.type === "text_delta") {
    process.stdout.write(event.assistantMessageEvent.delta);
  }
});

try {
  await session.prompt("Explain this project");
  console.log(session.getLastAssistantText());
} finally {
  unsubscribe();
  session.dispose();
}
```

`createAgentSession()` normally discovers the current working directory's resources, stored settings,
credentials, default tools, and persistent session manager. Use overrides only when the host owns a
boundary deliberately:

- `cwd` controls tool paths, project discovery, context files, and session grouping.
- `SessionManager.inMemory()` prevents session files. Persistent sessions are an entry tree; assigning
  raw agent messages does not replace authoritative persisted context.
- Subscribe **before** `prompt()` when output must stream.
- A prompt during a running stream must explicitly steer or follow up; use `steer()`, `followUp()`, or
  the matching prompting behavior rather than making Pi guess.
- Always `dispose()` a finished session. When using `AgentSessionRuntime`, dispose the runtime and
  rebind subscriptions after it replaces the current session.

The CLI loads codemode, tool-search, and MCP as built-in extensions; an SDK session does not. Add the
relevant extension factories to a `DefaultResourceLoader` and bind extensions when an SDK application
needs this behavior.

## Tools, custom resources, and credentials

Use string tool names for a straightforward, cwd-safe selection:

```ts
await createAgentSession({ cwd, tools: ["read", "grep", "find", "ls"] });
```

When custom `Tool` instances are necessary, create them for the same target cwd with current factory
APIs. Do not reuse stale exported tool constants from old examples: they may be removed or capture an
incorrect working directory. `customTools` supplements the selected built-ins; ensure an allowlist also
includes the custom tool when the installed API requires it.

For deterministic examples/tests, use in-memory settings/session storage and a deliberate model/runtime
configuration. Never hard-code API keys. The precise model/auth services have changed across versions,
so consult the local declarations and official SDK example before writing setup code.

## Removed server stack

Do not design against old experimental `pi-client`, `pi-protocol`, `pi-server`, Chord, daemon, or
server-mode guidance: that stack was removed in Pi v1.2.0, 2026-10-12 (CHANGELOG 1.2.0, Removed: "Removed the unreleased experimental
server, client, and plugin stack ... The `@earendil-works/pi-client`, `@earendil-works/pi-protocol`, and
`@earendil-works/pi-server` packages it used are removed and no longer released"; the stack was not part of the
published package; `@earendil-works/chord` dependency also removed). The supported integration designs are an
in-process SDK session or a Pi child process controlled through RPC.

## RPC protocol

Start a long-lived child process:

```sh
pi --mode rpc --no-session --no-approve
```

It reads and writes strict JSONL: one JSON object per LF-terminated record. Reserve stdout for protocol
records and send client diagnostics to stderr. Do not use a reader that treats Unicode line separators
as framing; split only on LF (optionally handling preceding CR).

```json
{"id":"req-1","type":"prompt","message":"Review this repository"}
{"id":"req-1","type":"response","command":"prompt","success":true,"data":{"disposition":"started"}}
```

Commands are asynchronous. Use unique request IDs and correlate returned `response` records by ID;
never rely on response order. Continue reading events after a successful `prompt` response: success
means accepted/queued/handled, not that model work is finished. If disposition is not `handled`, wait
for `agent_settled`, not merely `agent_end`. Closing the child's stdin requests an orderly shutdown and Pi disposes the active runtime before exiting (`docs/rpc.md`, "Shutdown", 1.5.0); an extension can also request shutdown, which completes after the current command or after `agent_settled`. Still handle process signals, child
startup errors, stderr diagnostics, unexpected exits, client-side deadlines, and cancellation.

Use `--approve` or `--no-approve` in every headless integration based on its resource policy.

## RPC command catalog (verified, pi.dev/docs/latest/rpc-commands, 2026-11-04, Pi 1.5.0)

Each command and response is one JSON object; add `"id"` to correlate. Failures after acceptance arrive
as events, not as a second `response`.

| Group | Commands |
| --- | --- |
| Prompting | `prompt` (`message`, optional `images`, `streamingBehavior`), `steer`, `follow_up`, `abort`, `clear_queue`, `new_session` (optional `parentSession`) |
| State | `get_state`, `get_messages` |
| Model / thinking | `set_model` (`provider`, `modelId`), `cycle_model`, `get_available_models`, `set_thinking_level`, `cycle_thinking_level`, `get_available_thinking_levels` |
| Queue modes | `set_steering_mode`, `set_follow_up_mode` (`all` or `one-at-a-time`, default `one-at-a-time`) |
| Compaction / retry | `compact` (`customInstructions`), `set_auto_compaction`, `set_auto_retry`, `abort_retry` |
| Bash | `bash` (`command`, `excludeFromContext`), `abort_bash` |
| Session | `get_session_stats`, `export_html`, `switch_session`, `fork` (`entryId`), `clone`, `get_fork_messages`, `get_entries` (`since` cursor), `get_tree`, `get_last_assistant_text`, `set_session_name` |
| Discovery | `get_commands` (extension commands, prompt templates, skills; built-in TUI commands excluded) |

Semantics that matter:

- `prompt` while the agent streams **requires** `streamingBehavior`: `"steer"` (delivered after the current
  turn's tool calls, before the next LLM call) or `"followUp"` (only when the agent would stop); omitting
  it returns an error. `data.disposition` is `handled` (extension command or input handler consumed it),
  `queued`, or `started`.
- `steer`/`follow_up` reject extension commands; send those through `prompt` (they run immediately even
  while streaming). Skill commands and prompt templates expand before sending.
- `abort` does not clear the queue: for Esc-like behavior send `clear_queue` first, then `abort`.
- `bash` output is stored in the session and reaches the model on the **next** `prompt` unless
  `excludeFromContext: true`. Streamed chunks arrive as `bash_execution_update` events (use `id`).
- `get_entries` returns all entries in append order, including pre-compaction history and abandoned
  branches, with `leafId`; an unknown `since` returns `success: false`. `get_messages` does not.
- `fork`/`clone` create a new session and can be cancelled by a `session_before_fork` handler
  (`data.cancelled`); `new_session`/`switch_session` by `session_before_switch`.
- `--name` can be passed to `pi --mode rpc` to set the initial session name.

## JSON/RPC event stream (verified, pi.dev/docs/latest/json)

JSON mode writes one session header (`{"type":"session","version":3,"id","timestamp","cwd"}`) then events;
RPC emits the same event shapes without the header (use `get_state`). Framing: strict JSONL, LF-terminated;
Node `readline` is unsuitable because it also splits on Unicode line separators. Keep draining stdout.

- Run: `agent_start`, `agent_end` (`messages`, `willRetry`), `agent_settled` (`aborted`); turn:
  `turn_start`, `turn_end` (`message`, `toolResults`).
- Messages: `message_start`, `message_update` (`usage`, `assistantMessageEvent`, delta-only — no `partial`),
  `message_end` (authoritative final message). Nested types: `text_*`, `thinking_*`, `toolcall_*`
  (`start`/`delta`/`end`); replace reconstructed content with `*_end` and `message_end`.
- Tools: `tool_execution_start`/`_update`/`_end` (`toolCallId`, `toolName`, `args`, `result`, `isError`,
  `durationMs`).
- Other: `queue_update`, `entry_appended`, `session_info_changed`, `thinking_level_changed`,
  `compaction_start`/`compaction_end` (`reason`: `manual`|`threshold`|`overflow`; `result`, `aborted`,
  `willRetry`, `errorMessage`), `auto_retry_start`/`_end`, `summarization_retry_*`.
- RPC-only: `bash_execution_update`, `extension_error`; extension UI is a separate subprotocol
  (`/rpc-extension-ui`, not yet read).
- TypeScript: use the exported `JsonAgentSessionEvent` from `@earendil-works/pi-coding-agent`.
