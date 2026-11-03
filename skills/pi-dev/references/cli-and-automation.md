# CLI and automation

Run `pi --help` first: extensions may add flags and the installed version is the source of truth.
Pi requires Node.js 22.19 or newer (npm `engines.node` of 1.5.0 is `>=22.19.0`; CHANGELOG 0.75.0, 2026-05-17: "Raised the minimum supported Node.js version to 22.19.0"). npm dist-tag `legacy-node20` = 0.74.2, a rescue release that tells Node 20 users to upgrade Node (CHANGELOG 0.75.5: "Published a 0.74.2 rescue release that tells Node 20 users to upgrade Node before updating to newer Pi versions"). The official installer pins dependencies;
`npm install -g --ignore-scripts @earendil-works/pi-coding-agent` is an alternative but does not pin
transitive dependencies.

```sh
curl -fsSL https://pi.dev/install.sh | sh
# Windows: powershell -c "irm https://pi.dev/install.ps1 | iex"
pi                         # interactive TUI
pi -p "Explain src/"       # print final text and exit
pi --mode json "Inspect"   # JSONL events, then exit
pi --mode rpc --no-session  # persistent JSONL controller
```

## Modes and boundary selection

| Need | Interface | Notes |
| --- | --- | --- |
| A person working in a terminal | interactive `pi` | TUI, slash commands, session picker and prompts |
| One shell task | `pi -p` / `--print` | final assistant text on stdout, then exits |
| Shell/script needs streaming records | `--mode json` | stdout is JSONL only |
| Non-Node host, custom UI, process boundary | `--mode rpc` | JSONL commands/responses/events until stdin closes |
| Node/Bun host wants direct API | SDK | use `createAgentSession()`; see SDK reference |

`@path` attaches a text file/image to the initial prompt and resolves against the working directory.
RPC mode rejects `@file`; send content through protocol commands instead. Piped stdin is prepended to
the initial prompt. `--` ends option parsing for a prompt that starts with `-`.

## Exit status and automation results

A process exit code alone is not enough to decide whether model work succeeded:

- In print mode, truncated output exits with status **1**.
- In JSON mode, a failed or aborted message does not necessarily make Pi exit non-zero; consume the
  emitted records and decide from their message/run state.
- In RPC, a successful `prompt` response means only that Pi accepted, queued, or handled the request.
  When its disposition is not `handled`, consume events until `agent_settled`.

Set an application-level deadline, keep consuming stdout to avoid backpressure, preserve stderr for
human diagnostics, and report an explicit result based on events rather than guessing from a single
response.

## Model and provider choice

```sh
pi --list-models
pi --model sonnet:high
pi --provider <provider> --model <model>
pi --api-key "$KEY" --model <model>
```

`--provider` requires `--model`; `--api-key` is non-persistent and also requires an explicit model.
`--thinking` accepts supported levels (`off`, `minimal`, `low`, `medium`, `high`, `xhigh`, `max`) and
is clamped by model capability. Do not silently switch providers: identity, credentials, cost, and
model capabilities are provider-dependent.

## Tools and resource switches

Default built-ins are `read,bash,edit,write`. A plain `--tools` value **replaces** that selection:

```sh
# Read-only built-ins.
pi --tools read,grep,find,ls -p "Review the code"

# Keep defaults but add/remove named tools.
pi --tools +codemode,-write -p "Analyze safely"

# Remove MCP entirely for this run.
pi --no-mcp --tools read,grep,find,ls -p "Inspect"

# Clean analytical run; explicit `-e` still loads even with --no-extensions.
pi --no-extensions --no-skills --no-prompt-templates --no-context-files -p "Answer directly"
```

`--tools` selects the tools declared to the model. It keeps MCP tools unless an entry starts with
`mcp__` (then it filters MCP tools too, e.g. `--tools read,bash,codemode,'mcp__radius__*'`). An MCP tool
no entry names is never declared directly, but `tool_search` (if listed) can load it and `codemode` scripts
can call it. To remove MCP use `--exclude-tools 'mcp__*'` or `--no-mcp`. `-xt/--exclude-tools` applies after
all other selection and accepts patterns; `-nbt/--no-builtin-tools` drops default built-ins but keeps
extension/custom tools; `-nt/--no-tools` disables built-in, extension, custom, and MCP tools.
`+name`/`-name` entries take exact names (no `*`) and cannot be mixed with plain names.
`--extension`, `--skill`, `--prompt-template`, and `--theme` add transient resources.

Built-ins: `read`, `bash`, `powershell` (Windows), `edit`, `write`, `grep`, `find`, `ls`. Built-in
extensions add `codemode` (QuickJS-sandboxed JavaScript calling other tools via `tools.<name>()`) and
`tool_search`; both are off by default (`"defaultTools": ["+codemode"]` or `--tools +codemode`).

## Output selection and other options

With a terminal on stdin and stdout Pi opens the TUI unless `--print`, `--mode json`, or `--mode rpc` is
given; if either stream is redirected and the mode is not json/rpc, Pi uses print mode. `--mode text` does
not force one-shot execution on a terminal — use `--print`. JSON/RPC reserve stdout for protocol records.
Also: `--system-prompt <text|path>` (replace), `--append-system-prompt` (repeatable), `-nc/--no-context-files`,
`--offline` (= `PI_OFFLINE=1`, no model-catalog refresh), `--export <input> [output]`, `-n/--name`.

## Maintenance and credential commands

```sh
pi update                 # updates Pi itself (not packages)
pi update --extensions    # all packages; `pi update <source>` one package; --models; --all; --force
pi auth check --provider openai --json   # exit 0 ready, 1 not_ready, 2 invalid
pi auth print-api-key --provider openai  # prints a secret to stdout
pi mcp add|remove|list|login|logout ...  # edits mcp.json; `-l` = project .pi/mcp.json
```

`PI_SELF_UPDATE_COMMAND` (or the global `selfUpdateCommand` setting, an argv array) lets `pi update` run a
wrapper's own update command (e.g. Nix, mise). `pi mcp list` exits 1 when a config entry is invalid or an
enabled server is not connected. Project `.pi/mcp.json` is read only for trusted projects. Source:
<https://pi.dev/docs/latest/cli> (2026-11-04).

## Noninteractive trust

Print, JSON, and RPC runs cannot show Pi's built-in trust dialog. Make the policy explicit:

```sh
pi --approve -p "Use known project resources"
pi --no-approve -p "Inspect untrusted checkout without .pi resources"
```

`--approve` loads trusted project-local configuration/resources for that invocation. `--no-approve`
ignores them. Neither turns the process into a sandbox.

## Sessions

```sh
pi --continue
pi --resume
pi --session <path-or-id>
pi --fork <path-or-id>
pi --session-id <id>
pi --no-session
pi --session-dir <dir>
```

Sessions are persistent by default and grouped by working directory. `--no-session` is ephemeral.
See `sessions.md` for branch and context semantics.
