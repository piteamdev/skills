# Extensions

> Source: https://pi.dev/docs/latest/extensions (fetched as Markdown, 2026-11-04)
> Version: `@earendil-works/pi-coding-agent` 1.5.0 · Verified: 2026-11-04
> Tags: **[doc]** = stated in the documentation page; **[rec]** = recommendation of this skill; **unverified** = not confirmed.
> Installed `.d.ts` files and `pi --help` win over this file for exact signatures.

Extensions are TypeScript modules loaded in the Pi process. They can subscribe to lifecycle events,
register tools, commands, shortcuts, flags, model providers, virtual models, MCP servers, custom UI,
and renderers. They run with the same OS permissions as Pi and can read prompts, tool calls, files,
credentials and session history **[doc]**. Load only trusted sources.

## Location and testing

| Location | Scope |
| --- | --- |
| `~/.pi/agent/extensions/*.ts` or `*.js` | global |
| `~/.pi/agent/extensions/*/index.ts` or `index.js` | global, directory |
| `~/.pi/agent/extensions/*/package.json` with a `pi.extensions` field | global, directory with manifest |
| `.pi/extensions/...` (same layouts) | project-local, **loads only after the project is trusted** |
| `settings.json` `extensions` array (files or dirs), `builtin:<name>` entries | configured |
| `settings.json` `packages` (`npm:…`, `git:…`) | Pi packages (see `resources-and-packages.md`) |
| `pi -e ./extension.ts` / `--extension` | one invocation / quick test only |

- Discovery goes **one level deep**; larger packages declare entry points in `package.json` →
  `"pi": { "extensions": ["./src/index.ts"] }` **[doc]**.
- `/reload` hot-reloads extensions from auto-discovered locations **[doc]**. `-e` is not installation.
- Extensions load through jiti, so TypeScript needs no compile step **[doc]**.
- Use `CONFIG_DIR_NAME` (exported) instead of hard-coding `.pi` for project config paths (rebranded
  distributions may differ); use `ctx.isProjectTrusted()` before honoring project-local extension
  config **[doc]**.

## Minimal extension

```ts
import { type ExtensionAPI, isToolCallEventType } from "@earendil-works/pi-coding-agent";
import { Type } from "typebox";

export default function (pi: ExtensionAPI) {
  pi.on("session_start", async (_event, ctx) => {
    ctx.ui.notify("extension loaded", "info");
  });

  pi.on("tool_call", async (event, ctx) => {
    if (isToolCallEventType("bash", event) && event.input.command.includes("rm -rf")) {
      const ok = await ctx.ui.confirm("Dangerous!", "Allow rm -rf?");
      if (!ok) return { block: true, reason: "Blocked by user" };
    }
  });

  pi.registerTool({
    name: "greet",
    label: "Greet",
    description: "Greet someone by name",
    parameters: Type.Object({ name: Type.String({ description: "Name to greet" }) }),
    async execute(toolCallId, params, signal, onUpdate, ctx) {
      return { content: [{ type: "text", text: `Hello, ${params.name}!` }], details: {} };
    },
  });

  pi.registerCommand("hello", {
    description: "Say hello",
    handler: async (args, ctx) => ctx.ui.notify(`Hello ${args || "world"}!`, "info"),
  });
}
```

## Imports and dependencies **[doc]**

| Import | Purpose |
| --- | --- |
| `@earendil-works/pi-coding-agent` | `ExtensionAPI`, `ExtensionContext`, events, helpers |
| `@earendil-works/pi-agent-core` | agent message and tool types |
| `typebox`, `typebox/compile`, `typebox/value` | tool parameter schemas |
| `@earendil-works/pi-ai` | `StringEnum`, transcript helpers, model types; inside Pi it still exports `Type`/`Static`/`TSchema`, but the published package no longer re-exports them — import from `typebox` in new code |
| `@earendil-works/pi-ai/compat`, `/models`, `/oauth`, `/providers/all` | provider construction, built-in APIs, OAuth helpers |
| `@earendil-works/pi-tui` | TUI components for rendering |

- Pi maps these to its own copies; legacy `@mariozechner/*` and `@sinclair/typebox` names map to the
  same modules.
- Third-party npm deps: add `package.json` next to the extension, run `npm install`; `node_modules/`
  resolves automatically. Node built-ins (`node:fs`, …) work.
- For packages installed via `pi install`, runtime deps must be in `dependencies` (`devDependencies`
  are not installed) and host-provided packages must **not** be listed in `dependencies`.
  Declare the host-provided packages (`pi-ai`, `pi-agent-core`, `pi-coding-agent`, `pi-tui`, `typebox`) in
  `peerDependencies` with a `"*"` range and do not bundle them (packages.md "Declare dependencies", in the
  1.5.0 npm tarball; Pi suppresses automatic peer installation for managed npm and git packages).

### Tool-argument validation **[doc]**

Pi validates tool arguments as JSON Schema with `@earendil-works/pi-schema`, whatever built the schema.
Before validation, loose arguments are coerced (numeric strings → numbers, `"true"` → `true`, single
value → one-item array, `null` in an optional property dropped). Not applied to tool arguments:
`Type.Refine()` (check in `execute()`), `Type.BigInt/Undefined/Void` (reject everything), codecs
(`execute()` gets the JSON value), `Format.Set()` custom formats (standard formats + `url` and
`json-pointer-uri-fragment` are checked). Errors seen by the model look like
`- limit: expected number, got string`.

## Lifecycle **[doc]**

Input order for a prompt: (1) extension `/commands` run their handler and **skip** `input`
(`pi.sendUserMessage()` checks commands only with `expandPromptTemplates: true`); (2) `input` handlers;
(3) `/skill:name` and `/template` expansion; (4) `before_agent_start`; then the agent loop.

```
input → before_agent_start → agent_start → [turns] → agent_end → more work? ─yes→ agent_start
                                                               └no→ agent_before_settle → continue? ─yes→ agent_start
                                                                                                    └no→ agent_settled
```

- "More work" = retry, final-attempt recovery, compaction retry, queued messages. **`agent_end` is not
  final idle; use `agent_settled`.** An aborted run skips `agent_before_settle`
  (`agent_end` → `agent_settled`).
- Per turn: `turn_start` → model request → tool calls → `turn_end`.
- Model request hooks in order: `context` → `context_with_system` → `before_provider_headers` →
  `before_provider_request` → `after_provider_response` → `provider_stream_event` +
  `message_start/update/end`.
- Tool call hooks in order: `tool_execution_start` → `tool_call` (block/mutate input) →
  `tool_execution_update` → `tool_result` (modify) → `tool_execution_end`. Unknown tool or invalid args
  skip `tool_call`; blocked/aborted calls skip execution and `tool_result`; all still emit
  `tool_execution_end` with an error result — observe finalized calls there.
- Session start/replacement: `project_trust` (when needed) → `session_start`; `/new`,`/resume` →
  `session_before_switch`; `/fork`,`/clone` → `session_before_fork`; then `session_shutdown` for the
  old runtime → new runtime loaded → `session_start`. `/reload` and exit also emit `session_shutdown`.
  `session_start.reason`: `startup|reload|new|resume|fork`; `session_shutdown.reason`:
  `quit|reload|new|resume|fork`. Exit covers Ctrl+C, Ctrl+D, SIGHUP, SIGTERM.
- Without runtime replacement: `/compact` or auto-compaction → `session_before_compact` then
  `session_compact` or `session_compact_failed`; `/tree` → `session_before_tree` then `session_tree`;
  `/name` → `session_info_changed`; model change → `thinking_level_select` (if level changes/clamps)
  then `model_select`; `!`/`!!` → `user_bash`; blocking dialogs → `ui_prompt_start`/`ui_prompt_end`;
  idle cache refresh → `cache_warming_decision`; MCP registration after startup → `mcp_servers_change`.

### Handler composition **[doc]**

Handlers run in extension load order, then registration order. Patterns: *notification* (return ignored),
*in-place mutation* (`before_provider_headers.headers`, `before_agent_start.systemPromptOptions`,
`tool_call.input`), *chained transformation* (`input`, `context`, `context_with_system`, `message_end`,
`tool_result`, `before_provider_request`), *gate* (handled input / blocked tool call / cancelled session
op / handled `user_bash` stops the fallback), *actionable boundary* (`turn_end`, `agent_before_settle`),
*last override* (`cache_warming_decision`). Do not do slow work in streaming-event handlers; gate
handlers should fail closed.

## Event reference **[doc]**

| Event | Can return | Key fields / notes |
| --- | --- | --- |
| `project_trust` | `{ trusted: "yes"\|"no"\|"undecided", remember? }` | only user/global and `-e` extensions participate; first yes/no wins and suppresses the built-in prompt; limited ctx (`cwd`, `mode`, `hasUI`, select/confirm/input/notify) |
| `resources_discover` | `{ skillPaths, promptPaths, themePaths }` | fired after `session_start`; `reason: "startup"\|"reload"` |
| `mcp_servers_change` | – | `event.servers`; handling it marks your extension as the one connecting registered servers |
| `session_start` | – | `reason`, `previousSessionFile` |
| `session_info_changed` | – | `name` (undefined if cleared) |
| `session_before_switch` | `{ cancel }` | `reason: "new"\|"resume"`, `targetSessionFile` |
| `session_before_fork` | `{ cancel }` (`skipConversationRestore` reserved) | `entryId`, `position: "before"` (`/fork`) or `"at"` (`/clone`) |
| `session_before_compact` | `{ cancel }` or `{ compaction: { summary, firstKeptEntryId, tokensBefore, usage? } }` | `preparation`, `branchEntries`, `customInstructions`, `reason: manual\|threshold\|overflow`, `willRetry`, `signal` |
| `session_compact` / `session_compact_failed` | – | `compactionEntry`, `fromExtension`; failed: `errorMessage`, `aborted` |
| `session_before_tree` / `session_tree` | `{ cancel }` or `{ summary: { summary, details, usage? } }` | `preparation.{targetId, oldLeafId, commonAncestorId, entriesToSummarize, userWantsSummary, …}`; custom summary used only if `userWantsSummary` |
| `session_shutdown` | – | `reason`, `targetSessionFile`; close resources here |
| `before_agent_start` | `{ message, systemPrompt }` | union by `messageType: "user"\|"custom"` (`prompt`/`images` only for `"user"`); `systemPromptOptions` is mutable |
| `agent_start`, `agent_end` | – | `agent_end.messages` |
| `agent_before_settle` | `{ entries, continue }` | final actionable boundary; one continuation |
| `agent_settled` | – | `aborted`; `ctx.isIdle()` true; requested runs start after all settled handlers |
| `turn_start`, `turn_end` | `turn_end`: `{ entries, continue }` | `turn_end`: `turnIndex`, `message`, `toolResults`, `outcome`, `entries`; draft entry types `custom`, `custom_message`, `context_edit`, `compaction` |
| `message_start/update/end` | `message_end`: `{ message }` (same `role`) | `message_update.assistantMessageEvent` |
| `tool_execution_start/update/end` | – | `toolCallId`, `toolName`, `args`/`partialResult`/`result`, `isError`, `durationMs`, `parentToolCallId` |
| `context` | `{ messages }` | deep copy without system messages; Pi restores prompt and tool declarations |
| `context_with_system` | `{ messages }` | full transcript; must keep a system message at index 0 |
| `before_provider_headers` | mutate `event.headers` (`null` deletes) | once per request; retries reuse headers |
| `before_provider_request` | replacement payload | debugging serialization/caching; not reflected by `ctx.getSystemPrompt()` |
| `after_provider_response` | – | `status`, `headers` (availability depends on provider/transport) |
| `provider_stream_event` | – | read-only `data`; handlers are awaited, so slow ones delay the stream |
| `cache_warming_decision` | `{ action: "warm"\|"stop" }` | last handler wins |
| `model_select` | – | `model`, `previousModel`, `source: "set"\|"cycle"`; not fired on session restore |
| `thinking_level_select` | – | `level`, `previousLevel` |
| `tool_call` | `{ block, reason?, terminate? }` | `input` mutable, **no re-validation**; handler errors block the tool (fail-safe); sibling calls in parallel mode may not see each other's results |
| `tool_result` | partial patch of `content`/`details`/`structuredContent`/`isError`/`usage` | chained; replacing `content` without `structuredContent` drops structured content |
| `user_bash` | `{ operations }` or `{ result }` | `command`, `excludeFromContext` (`!!`), `cwd`; errors block the command |
| `input` | `{ action: "continue"\|"transform"\|"handled" }` | `text`, `images`, `source: "interactive"\|"rpc"\|"extension"`, `streamingBehavior` |

Notes **[doc]**: `before_agent_start` — prefer editing `sections`, `selectedTools`, `promptGuidelines`
(Pi appends a patch message and keeps the cached prefix on models that accept mid-conversation system
messages); returning `systemPrompt`/`forceSystemPrompt` replaces the whole prompt (cache miss).
`turn_end`/`agent_before_settle`: an unconditional `continue: true` can loop forever — guard it;
persistence of proposed entries is validated but not transactional. Use `isToolCallEventType("bash", event)`
and `isBashToolResult(event)` for typed narrowing; custom tools should export their input type and use
`isToolCallEventType<"name", Input>`.

## ExtensionContext

- `ctx.ui` (see Custom UI), `ctx.mode` (`"tui"|"rpc"|"json"|"print"`), `ctx.hasUI`, `ctx.cwd`,
  `ctx.isProjectTrusted()`, `ctx.signal` (abort signal; usually `undefined` outside active turn events —
  pass it to `fetch`/model calls), `ctx.isIdle()`, `ctx.abort()`, `ctx.hasPendingMessages()`,
  `ctx.shutdown()` (interactive: deferred until idle; RPC: immediate when idle, else after the current
  command/run; print: no-op), `ctx.getContextUsage()`, `ctx.compact({ customInstructions, onComplete, onError })`
  (non-blocking), `ctx.getSystemPrompt()` (excludes later `context` mutations and provider-payload rewrites).
- `ctx.sessionManager` is read-only: `getCwd`, `getSessionDir`, `getSessionId`, `getSessionFile`,
  `getSessionName`, `getHeader`, `getEntries`, `getEntry`, `getBranch`, `getTree`, `getLeafId`,
  `getLeafEntry`, `getLabel`, `buildContextEntries`, `buildSessionProjection`.
- `ctx.modelRegistry` / `ctx.model` / `ctx.thinkingLevel` / `ctx.scopedModels` (from `--models` and
  `enabledModels`; empty = no scoping). Use `ctx.modelRegistry.streamSimple()`/`complete()` for model calls
  — they see extension-registered providers; `pi-ai/compat` streaming does not.

### ExtensionCommandContext (commands only; can deadlock in event handlers) **[doc]**

`getSystemPromptOptions()` (may contain full context-file contents — treat as sensitive),
`waitForIdle()`, `newSession({ parentSession, setup, withSession })`,
`fork(entryId, { position: "before"|"at", withSession })`,
`navigateTree(targetId, { summarize, customInstructions, replaceInstructions, label })`
(**rejects** — not `cancelled` — while a response/compaction/navigation is active, without a model when
`summarize` is set, or for a missing target), `switchSession(path, { withSession })`, `reload()`.

Session replacement footguns **[doc]**: after `newSession`/`fork`/`switchSession`/`reload` the old
extension instance has already received `session_shutdown`; captured old `pi`/`ctx` session-bound objects
are stale and throw; use only the `ctx` passed to `withSession`; capture plain data (strings, ids), not
objects. Treat `await ctx.reload(); return;` as terminal. Tools cannot call `ctx.reload()` (tool context
differs and Pi cannot reload mid-run): record the request and run the command from `agent_settled` via
`pi.sendUserMessage("/cmd", { expandPromptTemplates: true })` (see `reload-runtime.ts`).

## ExtensionAPI **[doc]**

| Method | Notes |
| --- | --- |
| `pi.on(event, handler)` | returns an unsubscribe function |
| `pi.registerTool(def)` | works at load and later (e.g. in `session_start`); appears in `pi.getAllTools()` without `/reload` |
| `pi.sendMessage(msg, { deliverAs, triggerTurn })` | custom message that **participates in LLM context**; `deliverAs`: `steer` (default; after current turn's tools), `followUp` (after agent finishes), `nextTurn` (queued for next user prompt); idle session stores it without a turn unless `triggerTurn: true` |
| `pi.sendUserMessage(content, { deliverAs, expandPromptTemplates })` | real user message; while streaming, `deliverAs` is **required** (else rejected with an extension error) |
| `pi.appendEntry(customType, data)` | persisted, **not** sent to the LLM; pair with `registerEntryRenderer` |
| `pi.setSessionName/getSessionName`, `pi.setLabel(entryId, label)` | session display name; `/tree` bookmarks |
| `pi.registerCommand(name, { description, getArgumentCompletions, handler })` | same name from several extensions → suffixes `/review:1`, `/review:2` |
| `pi.getCommands()` | extensions, then templates, then skills (`skill:<name>`); built-in interactive commands (`/model`, `/settings`) are not included; use `sourceInfo` for provenance |
| `pi.registerMessageRenderer`, `registerEntryRenderer`, `registerMarkdownTransformer`, `registerToolRenderer` | display-only |
| `pi.registerShortcut(key, { description, handler })`, `pi.registerFlag(name, { type, default })` | `pi.getFlag()` returns the default inside the factory — read it in `session_start` |
| `pi.exec(cmd, args, { signal, timeout, cwd, maxBuffer })` | **no shell**; `maxBuffer` default 256 MiB; SIGKILL after 5 s if SIGTERM ignored; result `{ stdout, stderr, code, killed, truncated }` |
| `pi.getActiveTools/getAllTools/setActiveTools(names)` | ignores unknown and `hidden` tools; `codemode`/`deferred` tools stay callable from codemode scripts whether active or not |
| `pi.setModel(model)` / `pi.setThinkingLevel(level)` | recorded in session history; do not change the `defaultProvider`/`defaultModel` for new sessions; `setModel` returns `false` without auth; levels `off\|minimal\|low\|medium\|high\|xhigh\|max`, clamped to model capability |
| `pi.events` | shared bus between extensions |
| `pi.registerProvider(name, config)` / `unregisterProvider(name)` | immediate after load, no `/reload`; see below |
| `pi.registerVirtualModel(model)` / `unregisterVirtualModel(provider, id)` | selectable entry routing each request to a physical model (docs: `virtual-models.md`) |
| `pi.registerMcpServer(name, config)` / `unregisterMcpServer` / `getMcpServers()` | see MCP below |
| `pi.getSettings()` | copy of effective merged settings |

### Providers **[doc]**

`registerProvider(name, config)` — legacy options: `name`, `baseUrl`, `apiKey` (literal, `$ENV`/`${ENV}`,
or leading `!command`; `$$` escapes `$`, `$!` escapes `!`), `api` (`anthropic-messages`,
`openai-completions`, `openai-responses`, …), `headers`, `authHeader`, `models` (replaces all existing
models of that provider), `refreshModels`, `oauth` (adds the provider to `/login`), `streamSimple`.
Calls in the factory are queued until the runner initializes; later calls apply immediately. The object
form accepts a full pi-ai `Provider` (`createProvider` from `@earendil-works/pi-ai/compat`); `models.json`
overrides still apply above it. An async factory can fetch a model list at startup so `pi --list-models`
sees it; use `refreshModels` for lists that change at runtime (pass `context.signal` to I/O). Advanced
topics are in `docs/custom-provider.md` (see "Providers, virtual models and RPC UI" below).

### MCP servers **[doc]**

`pi.registerMcpServer(name, config)` takes an `mcp.json`-style entry (`command`/`args`/`env`/`cwd` for
stdio; `url`/`headers`/`oauth` for HTTP; plus `exposure`, `toolExposure`, `description`, `enabled`,
`timeout`). Registrations are **not saved** — register on every load. A same-named server in `mcp.json`
wins (`/mcp` shows the override). Same name again replaces your own earlier registration; names owned by
another extension, invalid names, and invalid configs throw. Built-in MCP support connects registered
servers; if another extension replaced it and nothing connects, each registration is reported as an
extension error.

## State, persistence and branching **[doc]**

- Store tool state in tool-result `details` and rebuild it from `ctx.sessionManager.getBranch()` (not
  `getEntries()`: abandoned branches are alternative histories) on both `session_start` and
  `session_tree`.
- Use `pi.appendEntry()` for durable data the model must not see; `pi.sendMessage()` for data that must
  reach the model.

## Custom tools **[doc]**

Required: `name`, `label`, `description`, `parameters`, `execute(toolCallId, params, signal, onUpdate, ctx)`
returning `{ content, details }` (partial results passed to `onUpdate` also need `details`; use
`details: undefined` if empty; the `details` type must be consistent across results).

- `promptSnippet` → one-line entry in the `<tools>` prompt section (custom tools are omitted from it if
  absent); `promptGuidelines` → bullets appended **flat** to `<rules>` while the tool is active, so each
  must name the tool ("Use my_tool when…", not "Use this tool when…").
- Errors: `throw` or return `isError: true` (a returned result keeps `details` and `structuredContent`).
- `terminate: true` skips the automatic follow-up LLM call only if **every** finalized result in the batch
  terminates.
- `outputSchema` + `structuredContent` for data results (codemode scripts receive it).
- `executionMode: "sequential"` when a tool must not run concurrently (default: parallel within one
  assistant message). `defaultActive: false` registers inactive. `prepareArguments(args)` maps old
  stored-call shapes before validation — keep the public schema strict. `defineTool()` keeps type
  inference. `constrainedSampling` is optional.
- `usage`: return nested LLM usage so footer, `/session` and RPC totals include it.
- Prefer `StringEnum` from `@earendil-works/pi-ai` (plain `enum` schema every provider accepts).
- Some models prefix path arguments with `@`; built-in tools strip it — normalize it in custom path tools.
- **File mutation queue**: tools run in parallel, so wrap read-modify-write in
  `withFileMutationQueue(absolutePath, fn)` (resolve to an absolute path first; existing files are
  canonicalized via `realpath`) to avoid lost updates against built-in `edit`/`write`.
- **Output truncation is mandatory**: built-in limit 50 KB / 2000 lines. Use `truncateHead` (search
  results, file reads), `truncateTail` (logs), `truncateLine`, `formatSize`, `DEFAULT_MAX_BYTES`,
  `DEFAULT_MAX_LINES`; tell the model where the full output is.
- **Overriding built-ins** (`read`, `bash`, `powershell`, `edit`, `write`, `grep`, `find`, `ls`): register
  the same name (no warning); renderers are inherited per slot (`renderCall`/`renderResult`) but
  `promptSnippet`/`promptGuidelines` are **not**; the result and `details` shape must match the built-in.
  `--no-builtin-tools` starts without built-ins but keeps extension tools.
- **Remote execution**: `create*Tool(cwd, { operations })` with `ReadOperations`, `WriteOperations`,
  `EditOperations`, `BashOperations`, `PowerShellOperations`, `LsOperations`, `GrepOperations`,
  `FindOperations`; `create*ToolDefinition()` returns full definitions; `createLocalBashOperations()` /
  `createLocalPowerShellOperations()` for `user_bash`; `createBashTool(cwd, { spawnHook, exposeSessionEnvironment })`
  (session env vars injected before `spawnHook`).
- **Rendering**: `renderCall`/`renderResult(result, { expanded, isPartial }, theme, context)` return a
  `Component`; `renderShell: "self"` for full control; context has `args`, `state`, `lastComponent`,
  `invalidate()`, `expanded`, `isPartial`, `isError`, `durationMs`, `outputPad`, `keyHint()`/`keyText()`.
  Use the context's `keyHint`/`keyText` (not imports) so user keybindings apply. Keybinding ids are
  namespaced `app.*` and `tui.*`.

### Exposure, dynamic loading and nested calls **[doc]**

| `exposure` | Declared to the model | Callable from other tools |
| --- | --- | --- |
| `direct` (default) | while active | while active |
| `model-only` | while active | never |
| `codemode` | only if activated explicitly | whenever registered; listed by `codemode` |
| `deferred` | no | like `codemode`, but findable via built-in `tool_search` |
| `hidden` | no | unreachable (re-register with `hidden` to withdraw; tools cannot be unregistered) |

- `namespace: { name, aliases, description, instructions }` groups tools (as MCP servers do).
- `annotations` follow MCP hints (`readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint`);
  Pi does not verify them (MCP tools carry what their server declares); a permission extension may use them in
  `tool_call`, and missing hints take the MCP defaults: not read-only, may be destructive, open world
  (`docs/extensions.md`, "Tool Exposure").
- `prepareLoadout(loadout)` / `reachesTools(input)` implement orchestrating tools (what `codemode` and
  `tool_search` use). Dynamic loading: register searchable tools with `defaultActive: false`, keep a loader
  active, call `pi.setActiveTools()` from it; the model sees the updated tool list on the next request.
- `ctx.executeTool(name, args, { signal, onUpdate })` / `ctx.tools` inside `execute()`: failures return
  `isError: true` results (never reject); nested calls pass validation and `tool_call`/`tool_result`
  handlers, carry `parentToolCallId`, are **not** transcript entries, are recorded boundedly as
  `nestedCalls`, and their `usage` rolls up into the caller.

## Custom UI **[doc]**

- Dialogs: `ctx.ui.select/confirm/input/editor` (optional `timeout` with live countdown, `AbortSignal`),
  `ctx.ui.notify(msg, "info"|"warning"|"error")`.
- Fire-and-forget: `setStatus`, `setWidget` (above editor by default; `{ placement: "belowEditor" }`;
  factory form; `undefined` clears), `setFooter`, `setHeader`, `setTitle`, `setEditorText`,
  `setWorkingMessage`, `onTerminalInput`, `setToolsExpanded`, `getAllThemes`.
- Custom components: `ctx.ui.custom()`, overlays (experimental), custom editor (`CustomEditor`,
  `setEditorComponent`), autocomplete providers. Components get an injected `keybindings` manager.
  Component model and rules: see "TUI components" below (`docs/tui.md`).

### Mode behavior **[doc]**

| Mode | `ctx.mode` | `ctx.hasUI` | Notes |
| --- | --- | --- | --- |
| Interactive | `"tui"` | `true` | full TUI |
| RPC (`--mode rpc`) | `"rpc"` | `true` | dialogs/notifications via JSON protocol; `custom()` returns `undefined`; component factories ignored (`rpc-extension-ui.md`) |
| JSON (`--mode json`) | `"json"` | `false` | event stream; UI methods are no-ops |
| Print (`-p`) | `"print"` | `false` | extensions run but cannot prompt |

Guard `custom()`/component factories with `ctx.mode === "tui"` and dialogs with `ctx.hasUI`.

### Error handling **[doc]**

Extension errors are logged and the agent continues; `tool_call` and `user_bash` handler errors **block**
the operation (fail-safe); a thrown `execute` error becomes an error result and the run continues.

## Safe design rules

- **[doc]** Do not start processes, sockets, watchers or timers in the factory (an invocation may never
  start a session); start them from `session_start` or a command/tool and close them in an idempotent
  `session_shutdown` handler. The factory may be async and is awaited before `session_start`,
  `resources_discover` and queued provider flushes — keep it to small one-time discovery.
- **[doc]** Use `agent_settled`, not `agent_end`, for cleanup/notifications that must see retries,
  compaction and queued work.
- **[rec]** Implement policy in `tool_call` (and `user_bash`); an extension is a policy layer, not a
  sandbox — process/VM isolation does that. Gate handlers should deny when they cannot decide.
- **[rec]** Check installed declarations for exact signatures before copying older examples
  (e.g. `before_agent_start` is a union by `messageType`; `turn_end` now carries `entries`/`continue`).
- **[rec]** Keep project-local extension config behind `ctx.isProjectTrusted()`.

## Shipped examples **[doc]**

Examples live in `examples/extensions/` of the repository. Policy-related: `permission-gate.ts`,
`project-trust.ts`, `protected-paths.ts`, `confirm-destructive.ts`, `dirty-repo-guard.ts`,
`gondolin/` (routes built-in tools and `!` commands into a Gondolin micro-VM), `ssh.ts`.
Other useful references: `tool-override.ts`, `truncated-tool.ts`, `structured-output.ts`,
`dynamic-tools.ts`, `todo.ts` (branch-aware state), `custom-compaction.ts`, `reload-runtime.ts`,
`input-transform.ts`, `subagent/`, `plan-mode/`, `preset.ts`, `jev-router.ts` (virtual model),
`custom-provider-anthropic/`, `with-deps/`.
The native OS `sandbox/` extension example and its dependency were removed in 1.5.0 (CHANGELOG 1.5.0, Removed:
"Use the Gondolin example or the containerization guide").

## Providers, virtual models and RPC UI **[doc]**

Sources: `docs/custom-provider.md`, `docs/virtual-models.md`, `docs/tui.md`, `docs/rpc-extension-ui.md` (1.5.0).

**Custom providers.** Pick the smallest integration: models behind a supported API → `models.json`; changed endpoint or
headers → `models.json` or a small provider extension; dynamic discovery → `refreshModels`; `/login` → OAuth config;
unsupported wire protocol → `stream`/`streamSimple`. Provider extensions run inside Pi and can see credentials, prompts
and responses: treat as trusted code, never log secrets or payloads.
- `pi.registerProvider()` is called from the factory; Pi waits for async factories, so providers are available to startup
  model selection and `pi --list-models`. Forms: a complete pi-ai `Provider` (preferred for new work) or a name plus
  legacy `ProviderConfig`. `models.json` overrides compose above a native provider.
- Only `baseUrl`/`headers` for an existing provider keeps its built-in models; supplying `models` in the legacy form
  **replaces** that provider's models (chat, image, classifier). Omitted `type` = `"chat"`; image/classifier models need
  explicit `type` and implementations in `images`/`classifiers` keyed by `api`. Model-level `baseUrl` wins.
  `pi.unregisterProvider()` restores replaced built-in behavior.
- Key values: `$NAME`/`${NAME}`, leading `!command`, `$$` literal `$`, `$!` literal leading `!`. OAuth providers appear in
  `/login`; credentials go to `~/.pi/agent/auth.json`. Honor abort signals; `promptCache.short|long` (seconds) keeps an idle
  cache warm.
- Reuse a built-in API implementation (Anthropic Messages, OpenAI Chat Completions/Responses, Google Generative AI/Vertex,
  Azure OpenAI Responses, Mistral Conversations, Bedrock Converse) instead of copying a stream. A custom `streamSimple` must
  emit one `start`, balanced content events, exactly one terminal `done`/`error`, convert abort to an aborted result, read
  prompt/tools via `getCurrentSystemPrompt(context.messages)` / `getCurrentTools(context.messages)`, and call
  `options.onPayload`, `options.onResponse`, `options.onProviderStreamEvent`. Normalize an unknown overflow message to
  `context_length_exceeded` only in a guarded `message_end` handler; do not relabel rate limits.
- Example: `examples/extensions/custom-provider-gitlab-duo/`.

**Virtual models** (`pi.registerVirtualModel({ provider, id, name, thinkingLevels, route })`): a selectable model whose
`route(request, ctx)` picks a physical `{ model, thinkingLevel }` (optionally `state`, stored per session branch) before every
request. Selection (`model_change`/`thinking_level_change` entries, `ctx.model`) and dispatch (assistant message
`provider/api/model/thinkingLevel`) are recorded separately; providers only see physical models. `request.reason` is
`user` | `continuation` | `retry` | `direct`; return `previous` for `continuation` and `failed` for `retry` to keep prompt
caches valid. A virtual model cannot route to another virtual model; a throw or a model without credentials ends the request
with an error. `id` must not equal a physical model id of that provider (it would hide it). `pi.unregisterVirtualModel()`
removes it (`unregisterProvider()` does not); SDK: `modelRuntime.registerVirtualModel()`.

**TUI components** (`@earendil-works/pi-tui`): use `ctx.ui` first, build a custom component only for own rendering,
input, focus or lifecycle; never create a second terminal renderer. A component renders lines that must fit the given width
(use `visibleWidth()`, `truncateToWidth()`, `sliceByColumn()`, `wrapTextWithAnsi()`), invalidates caches on state/theme
change and calls `tui.requestRender()`. Keys: `matchesKey()`/`Key`, configurable actions via the injected
`KeybindingsManager`; focusable components place `CURSOR_MARKER` before the cursor and propagate `focused` to child
`Input`/`Editor` (IME placement); extend `CustomEditor` when replacing the editor. Mouse events exist only in fullscreen mode;
always provide a keyboard path. Use the passed theme (`theme.style()`, `theme.fg()`/`bg()`), do not store themed strings
across `invalidate()`. Debug with `PI_TUI_WRITE_LOG`.

**RPC extension UI** (`docs/rpc-extension-ui.md`): dialog methods (`select`, `confirm`, `input`, `editor`) emit an
`extension_ui_request` and block for an `extension_ui_response` with the same `id` (a `timeout` auto-resolves agent-side);
`notify`, `setStatus`, `setWidget`, `setTitle`, `set_editor_text` are fire-and-forget. In RPC mode `ctx.mode` is `"rpc"` and
`ctx.hasUI` is `true`; `custom()` returns `undefined`, header/footer/editor-component/theme methods are no-ops or return empty
values. Guard TUI-only features with `ctx.mode === "tui"`.

## Unverified / open

- `keybindings.md` and `models.md` were not read for this file (keybinding ids and `models.json` schema are covered by
  `configuration-and-security.md` only as far as already stated there).
