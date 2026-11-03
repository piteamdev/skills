# MCP, codemode and tool search

> Sources: `docs/mcp.md`, `docs/codemode.md`, `docs/settings.md`, `docs/cli.md`, `docs/sdk.md` from the
> `@earendil-works/pi-coding-agent` 1.5.0 npm tarball (mirrors https://pi.dev/docs/latest). Verified 2026-11-04.
> **[rec]** = recommendation of this skill. Anything else is stated in those pages.

## MCP servers

Pi connects to Model Context Protocol servers over **stdio** or **streamable HTTP** (legacy SSE is rejected).

```sh
pi mcp add filesystem -- npx -y @modelcontextprotocol/server-filesystem .
pi mcp add docs --url https://example.com/mcp --bearer-token-env-var DOCS_TOKEN
pi mcp add -l tools --env API_KEY='${TOOLS_KEY}' -- uvx tools-mcp   # -l/--local = project .pi/mcp.json
pi mcp list      # connects to every enabled server; exit 1 on an invalid entry or a disconnected enabled server
```

- Config: user `~/.pi/agent/mcp.json`, project `.pi/mcp.json` (read **only after project trust**); format
  `{ "mcpServers": { "<name>": {…} } }` as in other MCP clients. A project entry replaces a same-named user
  entry; a project entry without `command`/`url`/`type` overrides only `enabled`, `exposure`, `toolExposure`.
- stdio: `command` (one executable, not a shell string), `args`, `env`, `cwd` (relative → session directory;
  leading `~/` expands). HTTP: `url`, `headers`, `oauth`. Both: `timeout` (seconds, default 60, reset by
  progress notifications), `enabled`, `exposure`, `toolExposure`, `description`.
- `env`/`headers` values accept `${VAR}` and a whole-value `!command` (e.g. `"!echo Bearer $(gh auth token)"`).
- Names: letters, digits, `_`, `-`. Tools are `mcp__<server>__<tool>` (other characters → `_`; collisions get a
  hash suffix). Names differing only in `-`/`_` count as the same server.
- Invalid entries are reported and skipped; other servers still connect. Server logs go to
  `~/.pi/agent/mcp.log` (rotated to `mcp.log.1` past 5 MB).
- Startup: every enabled server connects in the background; the first prompt waits up to 10 s only for servers with
  `direct` tools. HTTP 408/429/5xx are retried twice; tool calls are **not** retried (the server may have acted);
  resource list/read retried once. Stdio shutdown: close stdin → SIGTERM → SIGKILL to the process group.
- `/mcp` (TUI) inspects, reconnects, signs in/out, changes exposure, enables/disables (per project override).
  Outside the TUI it prints status; `/mcp login|logout|reconnect <server>` work directly. `/reload` after edits
  made outside the session. `pi mcp add|remove|list|login|logout` work without a session and **do not load
  extensions**.
- Migrating: Claude Desktop/Claude Code/Cursor entries copy as-is; VS Code: move from `servers`, replace
  `${input:…}` with `${NAME}`; Codex TOML → JSON; OpenCode `local`/`remote` → stdio/URL (`environment` → `env`).

### OAuth (HTTP servers without an `Authorization` header)

`/mcp` shows "needs sign-in"; use "Sign in", `/mcp login <server>` or `pi mcp login <server>`. Tokens are stored in
`~/.pi/agent/mcp-auth.json` and refreshed; credentials belong to server name + URL. `oauth` options: `clientId`,
`clientSecret` (env/command allowed), `callbackPort` (`http://127.0.0.1:<port>/callback`), `callbackUrl`
(HTTP on `localhost`/`127.0.0.1`/`[::1]`), `scope`, `clientName` (default `pi`; sent only when registering),
`clientRegistration` (`auto` | `dcr` | `cimd`), `authServerMetadataUrl` (HTTPS except loopback; trusted as given).
Default identity is the Client ID Metadata Document `https://pi.dev/oauth/client.json` when the authorization
server supports it, else dynamic client registration. Over SSH paste the redirected URL into the sign-in screen.

### Tool exposure

| `exposure` | Meaning |
| --- | --- |
| `codemode` (default; alias `codemode-deferred`) | callable only from `codemode` scripts (`searchTools()`, `describeTool()`, `ALL_TOOLS`) |
| `deferred` | not declared until `tool_search` loads a match for the next model call |
| `direct` | declared like a built-in tool and also callable from codemode |
| `hidden` | registered but unreachable |

`toolExposure` overrides per tool (exact name beats pattern with `*`; first matching pattern wins), e.g. server
`deferred` with `search_code: direct`, `get_*: codemode`, `delete_*: hidden`. Pi activates `codemode` when a server
with `codemode` exposure connects and `tool_search` for `deferred` exposure; `"autoEnableCodemode": false` beside
`mcpServers` prevents that (project overrides user). Servers with non-direct tools are listed in the system prompt's
`mcp_servers` section (a changed section is appended as a message rather than altering tool declarations, so
earlier messages stay cached). Text results over 20 KB reach the model with the middle removed; the full text is in a
temp file.

- `--tools` keeps MCP tools unless an entry starts with `mcp__`; `--exclude-tools 'mcp__*'` or `--no-mcp` removes
  MCP. `"extensions": ["-builtin:mcp"]` (or `pi config` → Built-in) disables built-in MCP persistently.
- Resource tools (`list_mcp_resources`, `list_mcp_resource_templates`, `read_mcp_resource`) appear when a server
  offers resources; `ui://` MCP Apps resources are omitted.
- **Permissions**: every MCP call goes through Pi's tool pipeline, so extension `tool_call`/`tool_result` handlers
  and permission gates apply; codemode calls carry `parentToolCallId`. `pi.getAllTools()` exposes server
  annotations (`readOnlyHint`, `destructiveHint`, …); resource tools are read-only.
- An installed extension that registers `/mcp` (e.g. `pi-mcp-adapter`) **replaces** built-in MCP: Pi then does not
  read `mcp.json`. An extension registering `codemode` or `tool_search` replaces that built-in tool (Pi waits for
  servers its calls need only if it declares `reachesTools`). `pi mcp` shell commands always use the built-in code.
- Extensions add session-only servers with `pi.registerMcpServer()` (see `extensions.md`); a same-named file entry wins.
- **SDK**: SDK sessions load no built-in extensions. Add `createMcpExtension()`, `createCodemodeExtension()`,
  `createToolSearchExtension()` to `extensionFactories` of `DefaultResourceLoader` (`examples/sdk/14-codemode-mcp.ts`).
- **[rec]** Keep credentialed servers in user-level `mcp.json`; put only project-required servers in `.pi/mcp.json`
  and only trust projects you reviewed.

## Codemode

`codemode` lets the model write a JavaScript script that calls other tools; **only the script's output reaches
the model**, so calls can run in parallel and large results can be filtered first. It is registered **inactive**:
enable with `"defaultTools": ["+codemode"]` (settings) or `pi --tools +codemode`.

- Input is raw JavaScript (not JSON, no code fence), the body of an async function in a **QuickJS sandbox**: top-level
  `await`/`return`; no Node APIs, file system, network or timers. VM memory 256 MB.
- Optional first-line options: `// @options: {"max_output_tokens": 2000, "timeout_ms": 60000}`
  (`max_output_tokens` default 10000; `timeout_ms` unset by default, a hard deadline including worker wait).
- Globals: `tools.<name>(args)` (invalid identifier characters → `_`, e.g. `tools.mcp__dev_radius__search`),
  `text()`, `image()`, `console.*`, `return`, `exit()`, `store(key, value)`/`load(key)`, `ALL_TOOLS`,
  `searchTools(query, { limit?, namespace? })` (BM25, default 8), `describeTool()`, `describeNamespace()`, `models`.
- Tool calls are real; a failure does not undo earlier calls. Failed/blocked/invalid calls reject with an `Error`
  (use `Promise.allSettled`). `bash` resolves to `{ output, truncated, full_output_path?, exit_code,
  wall_time_seconds }` (up to 1 MiB of output); MCP tools resolve to `CallToolResult`; `read` to text or an image
  block. Scripts cannot start other `codemode` scripts.
- `store`/`load`: small JSON state persisted as `codemode-store` session entries and branch-aware; ≤262144
  characters per value, ≤1048576 in total.
- `models`: lists models and runs **non-LLM** models (classifiers, image generation) with the session's credentials;
  chat models are listed but cannot be run. At most four such calls at once per script.
- Limits: at most **two** codemode workers alive per Pi process (`PI_CODEMODE_MAX_WORKERS` changes it); a script
  with no worker within 60 s (or before its `timeout_ms`) fails without running; output beyond 16777216 characters or
  100000 `text()`/`image()`/`console` calls fails the script.
- Settings: `codemode.mode` `"on"` (default; declared tools stay declared) or `"only"` (tools hidden from the model
  and listed in the `codemode` description); `codemode.inlineBudget` default `3000` estimated tokens for declarations
  in the description (`0` lists only namespaces).
- **[rec]** Because scripts run real tools, apply the same policy gates (`tool_call` handlers) as for direct calls,
  and do not treat the QuickJS sandbox as a boundary for the tools it can call.

## Built-in extensions

`builtin:mcp`, `builtin:llama.cpp`, `builtin:codemode`, `builtin:tool-search` load by default; `-builtin:<name>` in
`settings.json` `extensions` disables one, a project `+/-builtin:<name>` entry overrides the user setting; `pi config`
lists them under Built-in; `--no-extensions` disables them too and `-e builtin:<name>` loads one explicitly
(`docs/settings.md`, "Resources"; `docs/cli.md`).

## Tool search

`tool_search` is off by default; the MCP extension turns it on when a server has `deferred` exposure, or name it in
`defaultTools`/`--tools`. It searches tools that are not declared to the model (`codemode` and `deferred` exposure, e.g.
MCP tools; ranking uses the server `description`) and declares the matches for the next model call; loaded tools are recorded in the transcript and stay declared on that branch (`docs/cli.md`,
"Tools"; `docs/mcp.md`, "Control tool exposure"). `--tools` never makes an MCP tool directly declared unless an entry names or matches it; only
`tool_search`, if listed, can load it. Models may declare `supportsToolSearch` in `models.json`
(`docs/models-configuration.md`: client-executed tool search for transcript-anchored additions, default `false`).

## Open points

- None known for this file as of 2026-11-04 (1.5.0); re-check `docs/mcp.md` and `docs/codemode.md` after upgrades.
