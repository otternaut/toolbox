# AGENTS.md

Guidance for AI coding agents (Claude Code, Codex, Cursor, and others) working
in this repository. Humans should read this too — it documents the conventions
the project is built on.

## What this repository is

`toolbox` is a collection of **portable AI agent skills**. Each skill is a
self-contained directory of Markdown (plus optional supporting files) that
teaches an AI agent how to perform a specific task well. The skills are written
to be tool-agnostic so the same directory can be used by Claude Code, Codex,
Cursor, or any other agent that supports the [Agent Skills](https://code.claude.com/docs/en/skills)
convention.

The `scripts/install` script publishes every skill into the global skills
directory of each supported tool, so a skill authored here is available
everywhere. Claude Code is symlinked (it follows links, so edits are live);
Codex and Cursor get real copies, because they don't traverse symlinks during
skill discovery.

## Repository layout

```
toolbox/
├── AGENTS.md              # This file — agent + contributor conventions
├── CLAUDE.md              # Points Claude Code at AGENTS.md
├── CONTRIBUTING.md        # How to add/change skills and open PRs
├── README.md              # Human-facing overview + install instructions
├── LICENSE                # MIT
├── .githooks/
│   └── pre-commit         # Regenerates the README skills table on commit
├── scripts/
│   ├── install            # Publishes skills (symlink/copy); exports rules; enables the git hook
│   ├── export-cursor-rules # Mirrors each skill into a global Cursor rule for cursor-agent
│   └── update-readme      # Regenerates the README skills table from frontmatter
└── skills/
    └── <skill-name>/
        ├── SKILL.md        # Required: frontmatter + instructions
        ├── references/     # Optional: long-form docs loaded on demand
        └── evals/          # Optional: eval prompts for the skill
```

## Anatomy of a skill

Every skill lives in `skills/<skill-name>/` and **must** contain a `SKILL.md`
file. The `scripts/install` script skips any directory without one.

`SKILL.md` starts with YAML frontmatter followed by Markdown instructions:

```markdown
---
name: <skill-name>          # kebab-case, must match the directory name
description: <one paragraph> # When to use the skill + what it does. This is
                            # what the agent reads to decide whether to invoke
                            # the skill, so make it specific and trigger-rich.
version: 1.0.0              # Semantic version
---

# <Human Title>

<Instructions the agent follows when the skill is invoked.>
```

Conventions:

- **`name`** is kebab-case and **must equal the directory name**. The installer
  and every host tool key off the directory name.
- **`description`** is the single most important field. Agents match a user's
  request against it to decide whether to load the skill, so lead with concrete
  trigger phrases ("review this PR", "analyze my diff") and say plainly what the
  skill does. Write it in the third person.
- **`version`** follows [semver](https://semver.org/). Bump it on every
  meaningful change (see CONTRIBUTING.md).
- Keep `SKILL.md` focused. Push long reference material into `references/` and
  link to it, so the agent only loads the detail when it needs it.
- Skills must stay **tool-agnostic**. Do not assume a specific CLI, API, or host
  is present; describe the *process* and let the agent use whatever tools its
  environment provides. See `skills/otterbot-review/SKILL.md` for the canonical
  example of this style.

### Optional supporting directories

- **`references/`** — long-form documentation, worked examples, or templates the
  skill links to but that would bloat `SKILL.md` if inlined.
- **`evals/evals.json`** — example prompts and expected outputs used to
  regression-test the skill. Follow the shape in
  `skills/otterbot-review/evals/evals.json`: a top-level `skill_name` plus an
  `evals` array of `{ id, prompt, expected_output, files }`.

## Installation model

`scripts/install` is a dependency-free Bash script (bash + coreutils only) that,
for each `skills/<name>/`, publishes an entry into every tool's skills
directory. **The publish mode differs by tool**, because only Claude Code
follows symlinks when it discovers skills — Codex and Cursor need real
directories:

```
~/.claude/skills/<name>  ->  <repo>/skills/<name>   (symlink)
~/.codex/skills/<name>       copy of <repo>/skills/<name>
~/.cursor/skills/<name>      copy of <repo>/skills/<name>
```

This is why the same symlink that works for Claude Code is invisible to Codex
and Cursor (including when they run as backends inside super.engineering /
Superconductor) — those tools skip symlinked entries during discovery. Copies
are stamped with a `.toolbox-managed` marker file so the installer can tell its
own copies apart from a real directory the user created.

Key properties to preserve when editing it:

- **Idempotent.** Re-running refreshes stale symlinks and drifted copies, leaves
  correct entries alone, and reports a summary. Never break this.
- **Non-destructive.** It never clobbers a real file or directory it didn't
  create — a symlink target it doesn't own, or a directory without the
  `.toolbox-managed` marker, is skipped with a warning.
- **`--dry-run`** must always show exactly what would happen without touching the
  filesystem.
- **Path resolution** is derived from the script's own location (`scripts/` is
  one level below the repo root). If you move the script, update `REPO_ROOT`.

Because Claude Code is symlinked, edits in this repo take effect immediately
there. Codex and Cursor use copies, so re-run `scripts/install` (a.k.a.
`--sync`) to refresh them after editing a skill. All three tools load skills
only at startup, so restart them (or the super.engineering session using them)
to pick up changes.

### The cursor-agent CLI reads rules, not skills

One important gap: the **Cursor Agent CLI** (`cursor-agent`) — which is what runs
when you select the "cursor" model inside super.engineering / Superconductor —
has **no Agent Skills feature at all**. Skills (`SKILL.md`, `~/.cursor/skills/`)
are a Cursor *app/IDE* feature. The CLI instead reads Cursor **rules** (`.mdc`
files), and it does read a **global** `~/.cursor/rules/` directory.

So `scripts/install` also runs `scripts/export-cursor-rules`, which mirrors each
skill into `~/.cursor/rules/<name>.mdc`:

- The rule's frontmatter carries the skill's `description` with
  `alwaysApply: false`, so cursor-agent loads it **on demand** by matching the
  request against the description — the same trigger model skills use, without
  bloating context.
- The rule body is the skill's instructions; a header points at
  `~/.cursor/skills/<name>/` for any `references/…` files.
- Rules are **generated artifacts** (a transform of `SKILL.md`), so there is no
  symlink shortcut — re-run `scripts/install` after editing a skill. The
  exporter is idempotent and only overwrites rules it generated (tagged with a
  `toolbox-generated-rule` marker); a hand-written `.mdc` is left alone.

This is why "keep the symlink version, it avoids syncs" doesn't hold: symlinks
never worked for Codex or the Cursor app (they don't traverse links), and the
cursor-agent backend needs a generated rule regardless — so copies + generated
rules are the only setup that covers every backend.

## Generated README skills table (keep it automatic)

The "Available skills" table in `README.md` is **generated, not hand-written**.
It lives between the `<!-- BEGIN SKILLS TABLE -->` / `<!-- END SKILLS TABLE -->`
markers and is produced by `scripts/update-readme`, which reads the `description`
field from each `skills/<name>/SKILL.md` and uses its first sentence as the row.

This is kept in sync automatically:

- `scripts/install` sets `core.hooksPath` to `.githooks`.
- `.githooks/pre-commit` runs `scripts/update-readme` on every commit and
  re-stages `README.md` if the table changed.

Rules for agents:

- **Never edit the skills table by hand.** Change a skill's `description` (or
  add/remove a skill) and let `scripts/update-readme` regenerate the table.
  Run `./scripts/update-readme` to preview, or `--check` to assert freshness.
- **Do not remove or rename the table markers** — the generator refuses to run
  (rather than silently dropping the table) if they're missing.
- If you extend the table's columns or format, update `scripts/update-readme`,
  the markers, and this section together.
- The first sentence of a skill's `description` is what shows in the table, so
  keep it a strong, self-contained summary.

## Working conventions for agents

- **Match the surrounding style.** Files here are prose-heavy Markdown and
  carefully formatted Bash. Match the existing tone, wrapping (~80 cols in
  Markdown prose), and comment density.
- **Bash:** keep scripts POSIX-friendly where practical, `set -euo pipefail`,
  no dependencies beyond bash + coreutils, and support `--dry-run`/`--help`
  for anything with side effects.
- **Don't add heavy toolchains.** This repo is intentionally lightweight — plain
  Markdown and Bash. Do not introduce a package manager, build step, or language
  runtime unless a change genuinely requires it, and flag it if you do.
- **Keep the three-doc contract in sync.** README.md (humans), AGENTS.md (this
  file), and CONTRIBUTING.md (process) should not contradict each other. If you
  change the skill format or install behavior, update all three.
- **Validate script changes** by running `./scripts/install --dry-run` before
  committing.

## Commit conventions

This repository uses [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>: <short summary>

feat:     a new skill or user-facing capability
fix:      a bug fix in a skill or script
docs:     documentation-only changes
refactor: restructuring without behavior change
chore:    tooling, meta, or housekeeping
```

Keep commits focused and the summary in the imperative mood ("add", not
"added"). See CONTRIBUTING.md for the full workflow.
