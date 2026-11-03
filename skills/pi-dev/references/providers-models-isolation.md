# Providers, models and isolation

> Sources: `docs/providers.md`, `docs/models.md`, `docs/containerization.md`, `docs/how-pi-works.md` from the
> `@earendil-works/pi-coding-agent` 1.5.0 npm tarball (mirrors https://pi.dev/docs/latest). Verified 2026-11-04.
> **[rec]** = recommendation of this skill; everything else is stated in those pages. The full per-field
> The per-field `models.json` schema is in `docs/models-configuration.md` and is not summarized here (out of scope, not checked).

## How Pi works in one paragraph **[doc]**

A session is a tree of entries (messages, tool calls/results, model changes, compactions). The branch ending at the
current entry is the active branch and supplies history for the next request. A turn = provider streams an assistant
message → Pi runs each tool call and records results; if results or queued messages need another request, another turn
starts, otherwise the run ends. Steering messages enter after the current assistant turn, follow-ups after pending work
is done, abort returns queued messages to the editor. The system prompt = base instructions + discovered context files;
requests also carry tool definitions and skill descriptions (full skill text loads on demand). Interactive, print, JSON,
RPC and SDK use the same agent and session mechanisms. Project trust is resolved **before** project settings/resources
load; context files load after that; tools run with the OS permissions of the Pi process, extensions inside it.

## Authentication and credential precedence **[doc]**

- `/login [provider]` (OAuth or API key) saves to `auth.json` (keep private, never commit); `/logout` removes the stored
  credential only (not env vars, not `models.json` auth, no revocation at the provider). On headless machines paste the
  final redirect URL / authorization code back.
- Precedence when several sources exist (`docs/models.md`): runtime `--api-key` (non-persistent; needs `--model`/`--models`)
  → stored `auth.json` credential → `apiKey` in `models.json` → provider environment variables or ambient cloud credentials.
  Provider extensions may define their own auth behavior. `pi auth check` shows what resolves on the machine.
- `auth.json` `key` may be `!command` (output cached for the process lifetime; empty output, timeout or non-zero exit
  leaves the key unresolved until restart). In `models.json`, `apiKey`/header values accept `$NAME`, `${NAME}`, a literal,
  or `!command` (run at request time, **not** cached); `$$` and `$!` give literal `$`/`!`.
- A stored API-key credential may carry an `env` object whose values override the process environment for that provider.
- Environment variables per provider (examples): `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GEMINI_API_KEY`,
  `COPILOT_GITHUB_TOKEN`, `MISTRAL_API_KEY`, `GROQ_API_KEY`, `OPENROUTER_API_KEY`, `AI_GATEWAY_API_KEY`, `HF_TOKEN`,
  `XAI_API_KEY`, `DEEPSEEK_API_KEY`, `RADIUS_API_KEY`, `TYPESAFE_API_KEY` (full table in `docs/providers.md`).
  Anthropic also reads `ANTHROPIC_OAUTH_TOKEN` (API credential) and `ANTHROPIC_AUTH_TOKEN` (bearer); with none set it uses
  workload identity federation when `ANTHROPIC_FEDERATION_RULE_ID`, `ANTHROPIC_ORGANIZATION_ID` and
  `ANTHROPIC_IDENTITY_TOKEN_FILE` are set (the identity-token file must stay fresh in long sessions).
- Provider specifics: **Azure** id `azure` (formerly `azure-openai-responses`; key in `auth.json`, `models.json`,
  `settings.json`, `--model azure/<id>`), env `AZURE_OPENAI_API_KEY` + `AZURE_OPENAI_BASE_URL` or
  `AZURE_OPENAI_RESOURCE_NAME`, `AZURE_OPENAI_DEPLOYMENT_NAME_MAP=model=deployment,…`, `AZURE_OPENAI_API_VERSION`
  (default `v1`); **Bedrock** bearer token (`AWS_BEARER_TOKEN_BEDROCK`) or AWS profile/IAM keys/ECS/IRSA, region via
  `AWS_REGION`/`AWS_DEFAULT_REGION`; **Vertex** `GOOGLE_CLOUD_API_KEY`, or ADC with `GOOGLE_CLOUD_PROJECT` +
  `GOOGLE_CLOUD_LOCATION`, or `GOOGLE_APPLICATION_CREDENTIALS`; **Cloudflare** AI Gateway needs `CLOUDFLARE_API_KEY`,
  `CLOUDFLARE_ACCOUNT_ID`, `CLOUDFLARE_GATEWAY_ID`, Workers AI the first two; **OpenRouter** lists only models the key may
  use (from `/models/user`, refreshed at startup in TUI, on `/model`, after `/login`, on `pi update --models`; saved list is
  used offline, in `-p` and RPC); **Radius** (`/login radius`) is Earendil Works' gateway, "early alpha" per the docs.

## Models **[doc]**

- `/model` shows models whose provider has usable auth; `Ctrl+S` saves the default; `/thinking` sets the level (limited to
  what the model supports); `Ctrl+P` cycles, `/scoped-models` controls the cycle. Model and thinking changes are recorded
  in the session; resuming restores them without changing defaults. Catalog: https://pi.dev/models; Pi ships a bundled
  catalog, may overlay newer data from pi.dev, caches it offline; `pi update --models` forces refresh.
- Connection choice: subscription → `/login`; API key → `/login` or env; local GGUF → llama.cpp router (`/llama`, built-in
  `builtin:llama.cpp`, see `docs/llama-cpp.md`); compatible endpoint → `models.json`; custom protocol/auth → provider
  extension (`extensions.md`).
- `models.json` (agent dir; reloaded when `/model` opens; add `"$schema": "https://pi.dev/schemas/models.schema.json"`):
  `providers.<name>` with `baseUrl`, `api` (e.g. `openai-completions`), `apiKey`, `models: [{ id, … }]`. A `models` entry
  adds or replaces the same ID; `modelOverrides` changes metadata of an existing built-in/extension model (unknown IDs are
  ignored); setting **only** `baseUrl` for a built-in provider keeps its models and auth (proxy use).
- `inputLimits.images.resize` (`maxWidth`, `maxHeight`, `maxBytes` on base64 payload, `jpegQuality`; defaults 2000×2000,
  4.5 MiB, quality 80); images are encoded once. `inputLimits.maxRequestBytes`, `images.maxPerMessage`/`maxPerRequest`
  may appear in the catalog but Pi does not yet rewrite or reject history based on them. `promptCache: { short, long }`
  (seconds; conservative end of the published range) is what makes a model eligible for `cacheWarming`.
- `samplingParams` and `samplingParamsByThinkingLevel` (keys `off…max`): Pi clamps unsupported thinking levels, then merges
  model `samplingParams`, the level's override, and request-level `samplingParams`, later wins per key; they apply only to
  `openai-completions`, `openai-responses`, `azure-openai-responses` (models.md names the API `azure-openai-responses`
  there even though the Azure *provider* id is now `azure`).
- Compatibility flags must describe **verified** endpoint differences, not just "OpenAI-compatible" claims.
- Classifier models (TypeSafe Jev, Cloudflare Clef variants, OpenAI GPT-6 Luna via the Decisions API) and image models
  (OpenRouter) do not appear in `/model`; they are reached through `codemode` (`models.classify`,
  `models.generateImages`; see `mcp-and-codemode.md`) or `ctx.modelRegistry.classify()` in extensions. GPT-6 Luna needs an
  API key, not `/login` ChatGPT credentials.
- **[rec]** For CI use environment variables or `!command` secrets instead of `/login`; do not bake `auth.json` into images.

## Isolation methods **[doc]** (`docs/containerization.md`)

| Method | Pi runs | Isolated | Credentials |
| --- | --- | --- | --- |
| Plain Docker | in container | Pi, built-in tools, `!` commands, extensions | passed into the container |
| Docker Sandboxes (`sbx`) | managed sandbox | same | stay on host, substituted by the proxy |
| OpenShell | local/remote sandbox | same | policy-controlled, inference routing |
| Gondolin extension | host | built-in tools and `!` commands only | host env vars are inherited by the VM |

- Whole-process isolation also isolates extensions; tool-only isolation (Gondolin) leaves other extension tools on the host.
- What you expose still counts: read-write mounts, mounting `~/.pi/agent` (credentials, settings, extensions, sessions),
  env vars, network. Use read-only mounts or copy files in/out.
- Plain Docker image: `node:24-bookworm-slim` + `bash ca-certificates git ripgrep`,
  `npm install -g --ignore-scripts @earendil-works/pi-coding-agent`, `docker run --rm -it -e ANTHROPIC_API_KEY -v "$PWD:/workspace" -v pi-agent-home:/root/.pi/agent`.
  Do not mount the host `~/.pi/agent` unless intended; check `!pwd` → `/workspace`.
- Docker Sandboxes: configure secrets first (`sbx secret set anthropic`, or `sbx secret set-custom` with a placeholder for a
  Claude token); do **not** run `/login` inside the sandbox; `sbx run --kit "docker.io/sbx/pi-kit:latest" pi`;
  `sbx exec <name> -- pi -p "…"`.
- OpenShell: `openshell gateway add/select`, `openshell sandbox create --name pi-sandbox --from pi -- pi`; remote gateways do
  not bind-mount the host folder (use `sandbox upload/download` or clone inside).
- Gondolin: local Linux micro-VM, needs Node.js 23.6+ and QEMU; example extension mounts the working folder at `/workspace`
  and overrides `read`, `write`, `edit`, `bash`, `grep`, `find`, `ls`; do not treat it as a credential boundary without
  scrubbing env vars. The native `sandbox/` example was removed in 1.5.0.
- **[rec]** Autonomous runs on untrusted code: whole-process container with only the working folder mounted and credentials
  injected per run; add `--no-mcp`/tool allowlists on top (`cli-and-automation.md`).
