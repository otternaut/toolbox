# Contributing to toolbox

Thanks for helping improve this toolbox of AI agent skills. This guide covers
how to add or change a skill, the conventions to follow, and how to open a pull
request. For the full design rationale, read [AGENTS.md](./AGENTS.md).

## Ways to contribute

- **Add a new skill** — teach agents a new task.
- **Improve an existing skill** — sharpen its instructions, fix a bug, or add
  reference material and evals.
- **Improve the tooling or docs** — the installer, README, or these guides.

## Prerequisites

- `bash` and `coreutils` (already present on macOS and Linux).
- A supported AI tool to test against (Claude Code, Codex, or Cursor).
- No language runtime or package manager — this repo is intentionally plain
  Markdown and Bash.

## Development setup

```bash
git clone https://github.com/otternaut/toolbox.git
cd toolbox
./scripts/install --dry-run   # preview what would be linked
```

Because skills are symlinked into your tool directories, once you've run
`./scripts/install` your local edits are live immediately — no reinstall needed
while iterating.

## Adding a new skill

1. **Create the directory.** Use a kebab-case name that matches what the skill
   does: `skills/<skill-name>/`.

2. **Write `SKILL.md`** with YAML frontmatter and Markdown instructions:

   ```markdown
   ---
   name: my-skill              # kebab-case, MUST match the directory name
   description: >
     One paragraph describing when to use the skill and what it does. Lead with
     concrete trigger phrases the agent will match against, and write in the
     third person. This field is how the agent decides whether to load the skill.
   version: 1.0.0
   ---

   # My Skill

   Step-by-step instructions the agent follows when the skill is invoked.
   ```

   Requirements:
   - `name` must equal the directory name.
   - `description` must be specific and trigger-rich — it drives invocation.
   - `version` follows [semver](https://semver.org/), starting at `1.0.0`.

3. **Keep it tool-agnostic.** Describe the *process*, not a specific CLI or API.
   Let the agent use whatever tools its environment provides. Study
   [`skills/analyze-code/SKILL.md`](skills/analyze-code/SKILL.md) as the
   reference example.

4. **Keep `SKILL.md` focused.** Move long-form docs, worked examples, or
   templates into `references/` and link to them so the agent only loads detail
   when needed.

5. **Add evals (recommended).** Create `evals/evals.json` following the shape in
   [`skills/analyze-code/evals/evals.json`](skills/analyze-code/evals/evals.json):
   a top-level `skill_name` and an `evals` array of
   `{ id, prompt, expected_output, files }`. Cover the main path and the tricky
   edge cases.

6. **Link it.** Run `./scripts/install` and confirm the new skill is picked up.
   This also enables the pre-commit hook that keeps the README in sync.

7. **Documentation is automatic.** The "Available skills" table in
   [README.md](./README.md) is generated from each skill's `description` by
   `scripts/update-readme`. **Do not edit that table by hand** — the pre-commit
   hook regenerates and re-stages it for you on commit. To preview it now, run
   `./scripts/update-readme`. To make the entry read well, write a strong first
   sentence in the skill's `description` (that sentence becomes the table cell).

## Changing an existing skill

- Make the edit and bump the `version` in its `SKILL.md`:
  - **patch** (`1.0.0 → 1.0.1`) — wording clarifications, typo fixes.
  - **minor** (`1.0.0 → 1.1.0`) — new capability, backward-compatible.
  - **major** (`1.0.0 → 2.0.0`) — behavior change that could surprise existing
    users.
- Update the skill's evals if behavior changed.
- Keep README, AGENTS.md, and CONTRIBUTING.md consistent if the change affects
  the skill format or install behavior.

## Changing the installer

`scripts/install` must remain:

- **Dependency-free** — bash + coreutils only.
- **Idempotent** — safe to re-run; stale links refreshed, correct ones left be.
- **Non-destructive** — never clobber a real (non-symlink) file or directory.

Always validate before committing:

```bash
./scripts/install --dry-run
```

## Style

- **Markdown:** wrap prose at ~80 columns; match the existing tone.
- **Bash:** `set -euo pipefail`; support `--dry-run` and `--help` for anything
  with side effects; match the formatting and comment density of the existing
  script.

## Commit messages

This repo uses [Conventional Commits](https://www.conventionalcommits.org/):

```
feat:     a new skill or user-facing capability
fix:      a bug fix in a skill or script
docs:     documentation-only changes
refactor: restructuring without behavior change
chore:    tooling, meta, or housekeeping
```

Use the imperative mood ("add", not "added") and keep each commit focused.

## Pull request checklist

Before opening a PR, confirm:

- [ ] `SKILL.md` has valid frontmatter with `name` matching the directory.
- [ ] `description` is specific and trigger-rich.
- [ ] `version` bumped appropriately (for changes to existing skills).
- [ ] Evals added or updated where it makes sense.
- [ ] `./scripts/install --dry-run` runs clean and lists the skill.
- [ ] `./scripts/update-readme --check` passes (the README table is generated —
      let the hook or `./scripts/update-readme` maintain it; don't hand-edit).
- [ ] Commits follow Conventional Commits.

Open the PR against `main` with a clear description of what the skill does or
what changed and why. Thanks for contributing! 🧰
