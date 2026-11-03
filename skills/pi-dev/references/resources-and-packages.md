# Skills, prompts, and Pi packages

> Sources: `docs/skills.md`, `docs/prompt-templates.md`, `docs/packages.md` from the
> `@earendil-works/pi-coding-agent` 1.5.0 npm tarball (mirrors https://pi.dev/docs/latest). Verified 2026-11-04.
> **[rec]** marks recommendations of this skill; everything else is stated in those pages.

## Skills

Pi implements the Agent Skills specification. A portable skill is a directory:

```text
my-skill/
├── SKILL.md
├── references/
├── scripts/
└── assets/
```

`SKILL.md` must start with a precise `name` and `description`; Pi advertises only its metadata at
startup and loads full instructions on demand. The description must say both what it does and when it
applies. Use `/skill:name` to force a skill; text after it becomes the appended request.

```md
---
name: pdf-tools
description: Extract text and tables from PDFs. Use when reading, converting, or inspecting PDFs.
---
```

Pi discovers skill directories recursively in `<agent-dir>/skills/`, `.pi/skills/`,
`~/.agents/skills/`, and `.agents/skills/` (the latter walking ancestors up to a repo root), plus
configured resource paths and packages. Project skills require project trust. Name collisions retain
the first discovered skill. `/reload` reloads resources in an active interactive session.

Portable frontmatter fields (Agent Skills specification): `name`, `description`, `license`, `compatibility`,
`metadata`, `allowed-tools` (experimental pre-approved tool list), `disable-model-invocation`. Names: lowercase
letters, digits, hyphens; no leading/trailing/consecutive hyphens; max 64 chars; description max 1024 chars.
Pi neither requires nor warns when `name` differs from the parent directory (other implementations may
enforce it, so matching is the portable choice). Malformed `SKILL.md` and skills without a description are not
loaded; most invalid fields only warn. Pi accepts some standalone Markdown skills, but a directory with
`SKILL.md` is the portable form. The model may fail to load a relevant skill; `/skill:name` forces it.
Run `/reload` after editing a skill. Skills can bundle scripts that the model runs: review them before
granting project trust.

Set `disable-model-invocation: true` only for a skill intended to be called explicitly. The
`enableSkillCommands` setting controls interactive command discovery; manually entered skill commands
still work.

## Prompt templates

Prompt-template Markdown files reside in `<agent-dir>/prompts/` or `.pi/prompts/` (project: after trust), and
become slash commands named after the file (`review.md` → `/review`). Conventional directories load **direct
`.md` children only**; settings and packages can select nested files. Use a template for reusable prompt text;
use a skill when the workflow needs references, scripts/assets, or detailed procedural instructions.

```markdown
---
description: Review staged git changes
argument-hint: "[focus]"
---
Review the staged changes. Focus on ${1:-correctness, security, and error handling}.
```

`description` is shown in completion (fallback: first non-empty line); `argument-hint` uses `<required>` and
`[optional]`. Substitutions: `$1`, `$2`…; `$@` / `$ARGUMENTS` (all args); `${1:-default}`; `${@:-default}`;
`${@:N}`; `${@:N:L}`. Arguments follow shell-like quoting (`/review "API compatibility"` is one argument).
Expansion happens before the text enters the agent; extensions see the raw input first through the `input`
event unless an extension command with the same name handles it. `/reload` picks up changes.

## Packages

A Pi package distributes extensions, skills, prompts, and themes via npm, git, URL, or local path:

```sh
pi install npm:@scope/pi-tools@1.0.0
pi install git:github.com/org/pi-tools@v1
pi install ./local-package
pi install --local npm:@scope/pi-tools   # write to .pi/settings.json
pi list
pi remove <source>
pi update --extensions
pi config
```

A minimal layout needs ordinary resource directories:

```text
my-pi-package/
├── package.json
├── extensions/
├── skills/
├── prompts/
└── themes/
```

For nonstandard paths, use `package.json`:

```json
{
  "name": "my-pi-package",
  "keywords": ["pi-package"],
  "pi": {
    "extensions": ["./src/extension.ts"],
    "skills": ["./resources/skills"],
    "prompts": ["./resources/prompts/*.md"],
    "themes": ["./resources/themes/*.json"]
  }
}
```

Source types: `npm:` (installed under the Pi npm directory; a versioned spec is pinned), `git:` (cloned and
reconciled to the selected ref; tags and commits are pinned, updates reconcile the checkout but do not move a
configured ref), a URL (treated as a git source), a local path (loaded in place, relative paths resolve from the
settings file containing them; a file loads one extension, a directory follows package discovery).
Personal installs go to `~/.pi/agent/settings.json`; `--local`/`-l` writes `.pi/settings.json`, read only
after project trust. If a configured package fails to install at startup, Pi warns, skips it, keeps running and
retries on the next start or `/reload`; `pi install` and `pi update --extensions` exit non-zero on failure.
`pi -e <source>` tries a package for one run: it is cached under `~/.pi/agent/tmp/extensions/` and checked on
every start and `/reload` (an exact-pinned npm version or an already-checked-out commit is not re-fetched;
offline or failed checks use the cached copy).

Scope and identity: the same package may appear in personal and project settings; a project entry normally
replaces the personal one (with `autoload: false` it acts as a filtering delta over the personal package). Pi
identifies npm packages by name, git packages by repository URL without the ref, local packages by resolved
absolute path, so equivalent declarations do not load twice. Installed packages load with separate module
roots: do not rely on two packages sharing a dependency instance.

Changelog: for packages that load at least one extension, Pi shows entries newer than the last seen version
under "Extension Updates" after an update (first sight only records the version). It reads `CHANGELOG.md` in
the package root; `pi.changelogPath` points elsewhere, `""` disables it. `pi.image` / `pi.video` add gallery
previews. `pi config` (or `pi config --local`) enables/disables discovered resources and built-in extensions.

The `pi-package` keyword makes an npm package eligible for the Pi package gallery. Pin npm versions
or git tags/commits where reproducibility matters. Project packages are trust-gated. Managed npm installs do not automatically install an extension package’s peer dependencies; declare Pi host packages as peers, but put other required runtime packages in `dependencies` and test a clean managed install. Review package source before installation: packages can load executable extensions.

## Dependencies and resource filters

Runtime imports of Pi's host packages must be peer dependencies, not dependencies:

```json
{
  "peerDependencies": {
    "@earendil-works/pi-ai": "*",
    "@earendil-works/pi-agent-core": "*",
    "@earendil-works/pi-coding-agent": "*",
    "@earendil-works/pi-tui": "*",
    "typebox": "*"
  }
}
```

Do not bundle them or list them under `dependencies`: a physical copy can bypass Pi's extension module mapping
in compiled ESM and create duplicate classes and registries (Pi reports an extension warning when it detects
this manifest). Pi suppresses automatic peer installation for managed npm packages and for git packages
installed with npm, pnpm or Bun; local packages are not installed or modified. Other Pi packages used as
dependencies must ship inside the published tarball and be referenced through their `node_modules` paths. Put actual package runtime dependencies in `dependencies`, because managed installs omit
dev dependencies.

Settings can narrow resources loaded from a package:

```json
{
  "packages": [{
    "source": "npm:@example/pi-tools",
    "extensions": ["extensions/*.ts", "!extensions/legacy.ts"],
    "skills": [],
    "prompts": ["prompts/review.md"]
  }]
}
```

Omit a key to load allowed resources, use `[]` to load none, and use `!`, `+`, `-` patterns as
appropriate. Filters only narrow package-provided paths.
