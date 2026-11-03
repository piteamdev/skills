---
name: pi-dev
description: >-
  Use when installing, configuring, automating, extending, packaging, embedding, or debugging
  Pi Coding Agent from pi.dev / @earendil-works/pi-coding-agent. Covers the pi CLI, print/JSON/RPC
  modes, models and providers, settings, sessions, skills, prompt templates, TypeScript extensions,
  MCP, Pi packages, the Node SDK, and security. Trigger on "pi.dev", "Pi Coding Agent", "pi install",
  "pi extension", "pi.registerTool", "createAgentSession", "pi --mode rpc", or ".pi/settings.json".
---

# Pi Coding Agent

Pi is a minimal, extensible terminal coding agent. Its current public package is
`@earendil-works/pi-coding-agent`; `@mariozechner/pi-coding-agent` is deprecated.
Prefer the current `@earendil-works/*` imports and verify the installed package's types for API details.

## Version and ground truth

Verified against pi.dev/docs/latest and npm `@earendil-works/pi-coding-agent` 1.5.0 (latest as of
2026-11-04; `legacy-node20` dist-tag 0.74.2). Verified against the docs and CHANGELOG.md shipped inside the
1.5.0 npm tarball. Facts marked "unverified" (where present) were not confirmed by a source. `@mariozechner/pi-coding-agent` (0.73.1, repo `badlogic/pi-mono`) is
the deprecated predecessor.

Pi changes quickly. Treat the installed CLI and types as authoritative for version-sensitive details:

```sh
pi --version
pi --help
pi --list-models
```

Official docs: <https://pi.dev/docs/latest>. The upstream source/documentation is
<https://github.com/earendil-works/pi>. Do not invent flags, extension APIs, or SDK types from memory.

## Task routing

| Task | Read |
| --- | --- |
| Install, interactive use, noninteractive CLI, tool isolation | [references/cli-and-automation.md](references/cli-and-automation.md) |
| User/project configuration, providers, model choice, trust | [references/configuration-and-security.md](references/configuration-and-security.md) |
| Skills, prompt templates, packages and distribution | [references/resources-and-packages.md](references/resources-and-packages.md) |
| TypeScript extensions, lifecycle hooks, custom tools/commands | [references/extensions.md](references/extensions.md) |
| Node/Bun integration, streaming, lifecycle, SDK vs RPC | [references/sdk-and-rpc.md](references/sdk-and-rpc.md) |
| Session persistence, branching, compaction, exporting | [references/sessions.md](references/sessions.md) |
| Every setting with type and default (`settings.json`) | [references/settings-reference.md](references/settings-reference.md) |
| Provider auth, model selection, `models.json`, how a turn works, Docker/OpenShell/Gondolin isolation | [references/providers-models-isolation.md](references/providers-models-isolation.md) |
| Upgrading from older Pi versions, breaking changes, removed APIs | [references/compatibility.md](references/compatibility.md) |
| MCP servers, OAuth, tool exposure, codemode, tool search | [references/mcp-and-codemode.md](references/mcp-and-codemode.md) |

## Critical rules

1. **No built-in sandbox or per-tool approval.** Pi and its extensions execute with the OS permissions
   of the process. Use a container/VM/sandbox to form a real boundary; working directory and project
   trust do not constrain a tool's filesystem, process, or network access.
2. **Treat extensions and packages as executable code.** Review them before installing. Project-local
   `.pi` resources are trust-gated; `AGENTS.md`/`CLAUDE.md` context is not.
3. **Use `--approve` or `--no-approve` in print, JSON, and RPC automation.** Those modes cannot show
   the interactive trust prompt. State the intended decision explicitly.
4. **Keep stdout clean in JSON/RPC mode.** It is protocol output; diagnostics belong on stderr. In RPC,
   frame records as JSONL and correlate asynchronous responses with request `id`s.
5. **Choose the integration boundary deliberately.** Use the Node/Bun SDK for in-process TypeScript;
   use RPC for another language, an IDE/custom UI, or process isolation; use `-p` for a one-shot shell
   task.
6. **Tool allowlists replace defaults, but MCP is separate.** A plain `--tools` list replaces the default
   selection (`read`, `bash`, `edit`, `write`), so name every desired tool; a list of only `+name`/`-name`
   entries instead edits the defaults. `--tools` does **not** remove MCP tools unless an entry starts with
   `mcp__`; an unnamed MCP tool is not declared directly, but `codemode`/`tool_search`, if enabled, can still
   reach it. To remove MCP use `--no-mcp` or `--exclude-tools 'mcp__*'`. `--tools read,grep,find,ls` limits
   the built-ins only; it is not a sandbox.
7. **Dispose SDK sessions.** Call `session.dispose()` in `finally`; use `SessionManager.inMemory()`
   when a host must not write JSONL session history.
8. **Use `agent_settled` for completion.** `agent_end` (`messages`, `willRetry`) only finishes one
   low-level run; retry, overflow recovery, compaction, steering, and queued follow-ups may continue.
   `agent_settled` (`aborted`) means no automatic work remains.
9. **Keep host packages external.** Pi packages should declare Pi-provided packages in
   `peerDependencies` with `"*"`, not `dependencies`, or duplicate module instances can break runtime
   behavior.
10. **Match the Agent Skills standard.** Skills need a directory with `SKILL.md`, precise frontmatter
    `name`/`description`, and are discovered from configured global/project locations.

## Baseline workflow

1. Establish the installed version and inspect `pi --help` / local package types for exact behavior.
2. Select the smallest interface that meets the requirement: CLI, SDK, or RPC.
3. Decide trust, tools, persistence, model/provider, and resource scope before starting execution.
4. Implement in the appropriate scope: user agent directory for personal behavior; `.pi/` for
   project-specific behavior; a Pi package for shareable resources.
5. Test noninteractive flows with explicit `--approve`/`--no-approve`, tool restrictions, and a
   controlled working directory. Review diffs and session data before exporting or sharing.

## Quick commands

```sh
# Current install: safe installation uses the official installer; npm does not pin transitive deps.
curl -fsSL https://pi.dev/install.sh | sh
# Or: npm install -g --ignore-scripts @earendil-works/pi-coding-agent

# Interactive in the target project; authenticate with /login if needed.
pi

# One-shot automation with an explicit read-only built-in tool set.
pi --no-approve --tools read,grep,find,ls -p "Review this repository; do not modify files."

# Load trusted project-local resources for this one automation run.
pi --approve -p "Run the project's documented checks."

# Machine events / long-lived JSONL control protocol.
pi --mode json "Summarize this repository"
pi --mode rpc --no-session
```

For SDK code, extension code, or configuration changes, read the matching reference before writing
an implementation.
