# Compatibility and breaking changes (0.75.0 → 1.5.0)

> Source: `CHANGELOG.md` in the `@earendil-works/pi-coding-agent` 1.5.0 npm tarball, sections "Breaking Changes" and
> "Removed" (Pi 1.5.0, verified 2026-11-04). Entries are paraphrased from the visible changelog text; the changelog is the
> authority for the full wording and migration steps. Entries marked "(inherited)" concern pi-ai / pi-agent-core types
> re-exported by Pi. Not every minor release between 0.75 and 0.99 was reviewed line by line: **unverified** for
> versions not listed here.

Use this when upgrading old code, extensions, `settings.json`, `models.json` or RPC/JSON consumers.

## Runtime

| Version / date | Change |
| --- | --- |
| 0.75.0 (2026-05-17) | Minimum Node.js is 22.19.0. `legacy-node20` dist-tag (0.74.2, a rescue release published with 0.75.5) only tells Node 20 users to upgrade Node |
| 1.0.1 | `npm-shrinkwrap.json` removed from the published package: npm installs no longer pin transitive dependencies; use the pi.dev installer for pinned installs |

## Configuration and providers

| Version | Change |
| --- | --- |
| 0.80.7 | `models.json`: `openai-responses` `compat.sendSessionIdHeader` removed; use `compat.sessionAffinityFormat` (`"openai"`, `"openai-nosession"`, …) |
| 1.0.3 (2026-10-05) | Provider `azure-openai-responses` renamed `azure`: update `auth.json` (or `/login` again), `models.json`, `settings.json` (`defaultProvider`, `enabledModels` patterns, `modelThinkingLevels`) |
| 1.2.0 (2026-10-12) | Generated schemas published at `https://pi.dev/schemas/*.schema.json` (models, settings, keybindings, themes); theme schema moved to `theme.schema.json` |
| 1.2.0 | `pi mcp list --json` reports `resources: true` instead of `resources`/`resourceTemplates` counts |

## JSON / RPC consumers

| Version | Change |
| --- | --- |
| 0.84.0 (2026-08-06) | JSON and RPC `message_update` events carry only `assistantMessageEvent` deltas; the cumulative `message` and `assistantMessageEvent.partial` fields are gone (quadratic growth). Clients must accumulate deltas themselves |
| 0.87.0 | `agent_settled` handlers' requested runs are deferred until all settled handlers finish (see `extensions.md`, `sdk-and-rpc.md`) |

## SDK and extension API

| Version | Change |
| --- | --- |
| 0.80.8 (2026-07-16) | SDK: `CreateAgentSessionOptions.authStorage`/`modelRegistry` replaced by async `modelRuntime`; `AuthStorage` no longer exported; `ModelRuntime.getAuth()` replaces `ModelRegistry.getApiKeyAndHeaders()`; `ModelRegistry.refresh()` is async for extensions (await it) |
| 0.83.0 | Bundled TypeBox aliases 1.3.7: deprecated APIs (`Type.Base`, `Type.Awaited`, `Type.Promise`, `Type.AsyncIterator`, `Type.Iterator`, `Type.Options`, `Value.Mutate`) removed |
| 0.84.0 | `ModelRegistry.getApiKeyAndHeaders()` header values may be `null` (deletion markers); `ModelRegistry.refresh()` takes options and returns a result; OAuth `refreshToken(credentials, signal)` must honor the signal; harness session model replaced by v4 lane-based `Session`/`SessionStorage`/`SessionRepo` (inherited); legacy JSONL/in-memory repo APIs removed (inherited) |
| 0.84.0 / 0.84.3 | Renamed inherited types `ModelsStreamTransforms` → `ModelsRequestTransforms`, `GoogleThinkingLevel` → `GoogleApiThinkingLevel` |
| 0.86.0 | Provider stream inputs are normalized `TranscriptContext`: read system prompt/tools with `getCurrentSystemPrompt(context.messages)` / `getCurrentTools(context.messages)`; `ToolCall.arguments` / `ToolResultMessage.details` must be JSON-compatible; **`user_bash` fails closed** (an error or invalid result aborts the command; return `undefined` to continue propagation) |
| 0.87.0 (2026-09-21) | `shouldStopAfterTurn` removed (use `finishTurn` returning `{ action: "end" }`); `ContextEditEntry` added to `SessionEntry` (handle `context_edit` in exhaustive switches); `SessionManager` is canonical for provider context (assigning `session.agent.state.messages` no longer replaces request history); `TurnEndEvent` expanded and `AgentBeforeSettleEvent` added to `ExtensionEvent` |
| 1.2.0 | `before_agent_start` also fires for `pi.sendMessage(..., { triggerTurn: true })` while idle and is a union on `messageType` (`"user"` events carry `prompt`…); a starting run counts as busy (`ctx.isIdle()` false, `pi.sendMessage()` queues into it) |
| 1.5.0 (2026-11-02) | Tool argument validation uses `@earendil-works/pi-schema`: TypeBox features beyond JSON Schema (`Type.Refine()`, `Type.BigInt()`, `Type.Undefined()`, codecs, `Format.Set()` formats) no longer apply to tool arguments |

## Removed

- 1.2.0: unreleased experimental server/client/plugin stack (`src/experimental`, `src/cli/experimental`, `./client` and
  `./experimental/plugin` exports; never part of the published package) and the dependency on `@earendil-works/chord`.
- 1.5.0: native OS `sandbox/` extension example and its dependency; use `examples/extensions/gondolin/` or
  `docs/containerization.md`.

## Unverified

- Behavior of releases 0.76–0.79, 0.81–0.82, 0.85, 0.88–0.99 beyond what is listed here: read their "Breaking Changes"
  sections in the changelog before relying on them.
