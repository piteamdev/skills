# Configuration and security

Sources: <https://pi.dev/docs/latest/configuration>, `/environment-variables`, `/security`, `/settings`
(checked 2026-11-04, Pi 1.5.0). Items marked **unverified** were not confirmed against a source.

## Scope and paths

The user agent directory defaults to `~/.pi/agent`; override it with `PI_CODING_AGENT_DIR` or the
SDK `agentDir` option. Project configuration is `.pi/` under the working directory and usually loads
only after project trust is granted. The one exception is `sessionDir`: Pi reads it **before** resolving
trust so it can locate sessions, so a project `.pi/settings.json` can influence where sessions are stored
even for an untrusted checkout.

| Location | Purpose |
| --- | --- |
| `<agent-dir>/settings.json` | personal defaults, resource paths and packages |
| `<agent-dir>/auth.json` | saved API keys / OAuth credentials |
| `<agent-dir>/models.json` | compatible endpoints, models and overrides |
| `<agent-dir>/mcp.json` | user-level MCP servers |
| `<agent-dir>/extensions`, `skills`, `prompts`, `themes` | personal resources |
| `.pi/settings.json`, `.pi/mcp.json` | project settings and MCP servers |
| `.pi/extensions`, `skills`, `prompts`, `themes` | project resources |
| `AGENTS.md`, `CLAUDE.md`, `AGENTS.override.md` | cascading context files |
| `SYSTEM.md`, `APPEND_SYSTEM.md` | replace/append system prompt |

`<agent-dir>/keybindings.json` holds custom keybindings. In the agent directory the user instruction file
may be `AGENTS.override.md`, `AGENTS.md`, `AGENTS.MD`, `CLAUDE.md`, or `CLAUDE.MD`.

Use `/settings` for common interactive settings and `/reload` after manually changing settings,
keybindings, instructions, or resources.

### Precedence rules

- `SYSTEM.md` / `APPEND_SYSTEM.md`: a trusted project file (`.pi/`) takes precedence over the file of the
  same name in the agent directory. Files with the same name are **not** combined.
- Context files are loaded from the agent directory, the working directory, and its parent directories; a
  file applies in its directory and everything below it. `AGENTS.override.md` replaces `AGENTS.md` /
  `CLAUDE.md` **only in the same directory** and does not suppress context files from other directories.
- Context-file discovery does not require project trust.
- Settings merge rules and the full key list: `settings-reference.md`.

## Environment variables

Source: `/environment-variables`. Provider API-key variables are documented on the Providers page.

| Variable | Effect |
| --- | --- |
| `PI_CODING_AGENT_DIR` | override config directory (default `~/.pi/agent`) |
| `PI_CODING_AGENT_SESSION_DIR` | override session storage; `--session-dir` wins |
| `PI_PACKAGE_DIR` | override the package directory (Nix/Guix store paths) |
| `PI_OFFLINE` | disable automatic network activity, including model catalog refresh |
| `PI_SKIP_VERSION_CHECK` | disable the pi.dev latest-version request |
| `PI_SELF_UPDATE_COMMAND` | command `pi update` runs for Pi itself (split on whitespace; `{packageName}`, `{version}`, `{installSpec}`); beats the `selfUpdateCommand` setting |
| `PI_TELEMETRY` | override install/update telemetry and provider attribution headers (`1/true/yes`, `0/false/no`) |
| `PI_CODEMODE_MAX_WORKERS` | max codemode workers alive per process across sessions (default 2; each script up to 256 MB) |
| `PI_CACHE_RETENTION` | `long` for extended provider prompt caching where supported |
| `PI_SHARE_VIEWER_URL`, `PI_RADIUS_GATEWAY` | override `/share` viewer base URL; `/bug` upload gateway |
| `HTTP_PROXY`, `HTTPS_PROXY` | proxy outbound HTTP requests |
| `VISUAL`, `EDITOR` | external-editor fallback when `externalEditor` is unset |
| `PI_CLEAR_ON_SHRINK`, `PI_HARDWARE_CURSOR`, `PI_CLIPBOARD_OSC52`, `PI_HYPERLINKS`, `PI_PROGRAM_STATUS`, `PI_IMAGE_PROTOCOL`, `PI_TRUE_COLOR`, `PI_TUI_ESC_TIMEOUT` | terminal rendering/input tuning (see Terminal setup) |

Markers set by Pi: the CLI and RPC entry points set `AI_AGENT=pi` and `PI_CODING_AGENT=true`; child
processes inherit them. They are **not** set automatically when Pi is embedded via the SDK, and they are
not session-specific.

Session variables for the LLM-callable `bash`/`powershell` tools (resolved when each command starts, not
injected into user `!`/`!!` commands): `PI_SESSION_ID`, `PI_SESSION_FILE` (unset for ephemeral sessions),
`PI_PROVIDER`, `PI_MODEL`, `PI_REASONING_LEVEL` (`off|minimal|low|medium|high|xhigh|max`). They identify the
selected Pi model, not a model a router picks upstream. Custom tools from `createBashTool()` /
`createPowerShellTool()` expose them by default (set before `spawnHook`, so `ctx.env` has them); pass
`exposeSessionEnvironment: false` to disable, which also strips inherited values so nested Pi processes
do not leak stale parent metadata.

## Project trust

Trust prevents a checkout from silently loading project configuration and executable extensions.
Protected resources include `.pi/settings.json`, `.pi/mcp.json`, `.pi/extensions`, `.pi/skills`,
`.pi/prompts`, `.pi/themes`, `.pi/SYSTEM.md`, `.pi/APPEND_SYSTEM.md`, and project `.agents/skills`.
A bare `.pi` directory does not require a decision.

Context files (`AGENTS.md`, `CLAUDE.md`) load regardless of project trust unless context discovery is
disabled. Treat all repository prose, source, output, and model responses as potentially adversarial
input.

Trust selection precedence: command-line `--approve`/`--no-approve`; a user extension handling
`project_trust`; the nearest saved decision in `<agent-dir>/trust.json`; then
`defaultProjectTrust` (normally `ask`). Noninteractive `ask`/`never` skips protected project resources;
`always` loads them.

## Compatibility and upgrade notes

- Node.js **22.19+** is required (npm `engines.node` `>=22.19.0` in 1.5.0; CHANGELOG 0.75.0, 2026-05-17,
  Breaking Changes). The `legacy-node20` dist-tag (0.74.2) is only a rescue release telling Node 20 users to
  upgrade Node (CHANGELOG 0.75.5). The official pi.dev installer uses Pi's pinned install lock;
  global npm installation does **not** pin the transitive dependency graph. Use `--ignore-scripts` for
  normal npm installation.
- Since v1.0.3 (2026-10-05) the provider id `azure-openai-responses` is renamed to `azure` (CHANGELOG 1.0.3,
  Breaking Changes: "Renamed the Azure provider from `azure-openai-responses` to `azure`"). The `AZURE_OPENAI_*`
  environment variables are unchanged. Sessions that used the old provider fall back to another model on resume. When upgrading an
  affected setup, update `auth.json`, `models.json`, and relevant settings such as `defaultProvider`,
  `enabledModels`, and `modelThinkingLevels`, or reconnect through `/login`. Existing sessions under
  the old provider will not retain their prompt-cache path.
- Configuration JSON Schemas are published under `https://pi.dev/schemas/*.schema.json`. Verify the
  current theme schema before carrying forward an old `$schema` reference.
- `PI_SELF_UPDATE_COMMAND` and the global `selfUpdateCommand` customize self-update behavior. Pi
  deliberately ignores a project-level `selfUpdateCommand`: an untrusted checkout must not inject an
  updater command.

## Security model

Pi has **no built-in permission boundary**. The process, built-in tools, shell commands, extensions,
package installers, language servers, and child processes have the permissions of the launching user.
The working directory does not keep shell commands within it. Reviewing a diff and granting project
trust are useful controls, not isolation.

Use these controls:

- Run untrusted or high-impact tasks in a container, VM, or OS sandbox; expose only required files,
  network services, and short-lived scoped credentials.
- Use an explicit built-in tool allowlist for read-only work, and remove MCP with `--no-mcp` or
  `--exclude-tools 'mcp__*'` (a plain `--tools` list does not remove MCP tools).
- Review source of extensions/packages before installation; they can inspect credentials, prompts,
  files, tool calls, and session history.
- Use snapshots or version control before edits; review changes before deploying them elsewhere.
- Inspect session exports/shares: they can contain prompts, source contents, command output, arguments,
  credentials accidentally exposed to a run, and extension messages.

See Pi's `containerization.md` for Gondolin, plain Docker, and OpenShell patterns. Containerizing the
whole Pi process is stronger than routing only built-in tools into an isolated environment, because
extensions can otherwise remain on the host.
