---
name: otterbot-review
description: Perform a principal-level code review, producing a structured Markdown report with severity-tagged findings and a scorecard. Given a pull/merge request URL, reviews that PR and submits the report as a formal review — approving it when the recommendation is Approve or Comment only, requesting changes otherwise. Given no URL, reviews the current local code changes and presents the report in the conversation. Use this whenever the user asks to "review this PR", "review my diff", "analyze this code change", "do a code review", "check this pull request for issues", pastes a pull-request URL and asks for feedback, or wants a merge-readiness assessment. Works with any git hosting provider (GitHub, GitLab, Bitbucket, etc.).
version: 1.2.0
---

# Otterbot Review

Review a code change like a senior engineering manager and principal-level
reviewer would: friendly, concise, and high-signal. The output is a single
Markdown report, not a back-and-forth — read the change once, form a complete
judgment, and write it up.

This skill is intentionally agnostic about *how* you read the change and
*where* the report ends up. Use whatever tools you have available in the
current environment (CLI, API access, provided file contents, pasted diffs,
etc.) rather than assuming a specific one — the review process below is what
matters, not the plumbing.

## 1. Determine the mode

Check whether the user's request or the surrounding conversation includes a
pull/merge request URL (GitHub, GitLab, Bitbucket, or similar).

- **PR URL present → PR review mode.** Review that PR and, at the end,
  submit the report as a formal review whose verdict (approve vs. changes
  requested) follows the Merge Recommendation. See §6.
- **No PR URL → local review mode.** Review the current local code changes
  in this repository and present the report in the conversation. See §6.

If the user's intent is ambiguous in some other way (e.g. multiple PR links,
or a claim of local changes that don't actually exist), ask before
proceeding rather than guessing. Otherwise, don't ask which mode to use —
the presence or absence of a URL decides it.

### PR review mode

Fetch the PR's title, description, linked tickets, changed files, and diff
using whatever access you have in the current environment (a CLI for that
host, an API, a browsing tool). Use the PR's actual title in the review
summary line — never invent or assume one.

If the PR data can't be fetched (no access, no matching tool, auth error),
say so plainly and offer to review from a pasted diff instead of silently
falling back to something else.

### Local review mode

Review the full local change set, not just tracked-file edits:

1. Check for uncommitted changes, **including untracked (new) files** — a
   plain `git diff` only shows modifications to files git already tracks,
   so it will miss new files entirely. Check `git status` first; diff new
   files as additions (e.g. `git diff --no-index /dev/null <file>` per
   file, or stage them with `git add -N` before running `git diff`).
2. If there are no uncommitted changes, diff the current branch against its
   base/target branch instead.
3. If neither applies — e.g. a repository with no commits yet and nothing
   uncommitted, or a base branch that doesn't exist — treat every tracked
   and untracked file in the working tree as the change set.

Use the change description (commit messages, branch name, or context the
user has given you) in place of a PR title/description.

### Gathering context

Once you know the target, pull in enough surrounding context to judge the
change properly, not just the diff in isolation: the PR/commit description,
any linked tickets or requirements, the existing tests, and relevant parts of
the codebase the change touches. A diff without context produces a shallow
review.

## 2. Review focus

Evaluate the change for:

- **Correctness:** bugs, edge cases, broken logic, race conditions, error
  handling, and requirement fit.
- **Completeness:** missing states, validation, integrations, cleanup,
  tests, or partial implementation.
- **Regression risk:** side effects on existing flows, APIs, data, UI
  behavior, permissions, performance, deployment, and compatibility.
- **Code quality:** readability, structure, naming, duplication,
  abstractions, typing, and consistency with the project's existing
  patterns.
- **Testing:** whether coverage matches the risk level, including missing
  unit, integration, regression, migration, contract, or manual checks.
- **Security and data integrity:** authorization, unsafe inputs, secret
  leakage, validation gaps, data loss, and sensitive data handling.

## 3. Severity levels

Use exactly one severity per finding:

- 🔴 **Critical** — Must fix before merge: outage, data loss, security
  issue, or severe user impact.
- 🟠 **High** — Should fix before merge: clear bug, broken requirement,
  major regression risk, or significant maintainability issue.
- 🟡 **Medium** — Important but not always blocking: edge case, incomplete
  coverage, confusing design, or moderate risk.
- 🔵 **Low** — Minor improvement that reduces maintenance cost or improves
  polish.
- ⚪ **Optional** — Non-blocking suggestion, alternative approach, or future
  enhancement.

## 4. Review standards

For each finding, include:

- **Severity**
- **Issue**
- **Location**
- **Why it matters**
- **Recommended fix**

Include short code examples or suggested snippets only when they make the
issue clearer. Keep feedback specific, actionable, and non-duplicative.
Avoid vague comments like "clean this up" unless the impact and fix are
clear. Don't invent findings to fill out every severity bucket — an empty
section is a better signal than a padded one.

## 5. Output format

Produce the report in exactly this structure:

```markdown
# 🦦 OtterBot Code Review

## ✨ Summary — <actual PR/change title>

Briefly state overall quality, merge readiness, and the highest-risk
concern, if any.

## 📋 Requirements Check

State whether the change appears to satisfy the intended business/product
requirements. Note any ambiguity or missing acceptance criteria.

## 📊 Scorecard

Score each category from 0-100, where 100 means excellent and merge-ready
with no meaningful concerns.

| Criteria | Score | Notes |
| --- | ---: | --- |
| Correctness | 0-100 | Brief rationale |
| Completeness | 0-100 | Brief rationale |
| Regression Risk | 0-100 | Brief rationale |
| Code Quality | 0-100 | Brief rationale |
| Testing | 0-100 | Brief rationale |
| Security & Data Integrity | 0-100 | Brief rationale |

**Overall Score:** 0-100

**Merge Recommendation:** Choose one: ✅ Approve, ⚠️ Request changes, or
💬 Comment only.

Explain the recommendation in 1-2 concise sentences.

## 🔎 Findings

Group findings by severity. Omit empty severity sections unless there are
no findings at all.

Use this card format for every finding:

### 🟠 High

> **Issue:** Clear, concise statement of the problem.
>
> **Location:** `path/to/file.ext`, function/class/section, or changed
> behavior.
>
> **Why it matters:** Explain the user, system, data, security, or
> maintenance impact.
>
> **Recommended fix:** Describe the concrete change needed.

---

> **Issue:** Another separate finding.
>
> **Location:** `path/to/other-file.ext`
>
> **Why it matters:** Explain the risk.
>
> **Recommended fix:** Explain the fix.

### 🟡 Medium

> **Issue:** ...
>
> **Location:** ...
>
> **Why it matters:** ...
>
> **Recommended fix:** ...

### 🔵 Low

> **Issue:** ...
>
> **Location:** ...
>
> **Why it matters:** ...
>
> **Recommended fix:** ...

### ⚪ Optional

> **Issue:** ...
>
> **Location:** ...
>
> **Why it matters:** ...
>
> **Recommended fix:** ...

## 🧪 Testing Notes

Briefly summarize test coverage, missing cases, and any recommended manual
verification.
```

## 6. Delivering the review

Delivery follows the mode determined in §1:

- **PR review mode:** submit the completed report as a single formal review
  on the PR/MR — not just a plain comment — using whatever tool your
  environment provides for that host (e.g. a hosting-provider CLI or API).
  Set the review verdict from the **Merge Recommendation** in §5:

  - ✅ **Approve** or 💬 **Comment only** → submit the review as **approved**.
  - ⚠️ **Request changes** → submit the review as **changes requested**.

  The report Markdown is the body of that review in every case; only the
  verdict differs. Also show the report in the conversation so the user
  doesn't have to leave it to read your feedback. Submitting is the
  expected, default action for this mode — no need to ask for confirmation
  first, since providing a PR URL is the user's signal that they want it
  reviewed there. If the host or your access can't attach a verdict (e.g.
  the tool only supports plain comments, or you'd be reviewing your own PR
  where self-approval is disallowed), fall back to posting the report as a
  comment and state the recommendation in the report. If submitting fails
  entirely (no access, no such tool available, auth error), say so and fall
  back to presenting the report in the conversation.
- **Local review mode:** present the report in the conversation only. There
  is no PR to comment on, and nothing should be published anywhere else
  unless the user explicitly asks for that.

## 7. Completeness checklist

Before delivering the report, confirm all of these — they're drawn directly
from mistakes this skill has made in practice (see `references/examples.md`
for what a full pass looks like):

- [ ] Correct mode determined from presence/absence of a PR/MR URL (§1)
- [ ] Full change set gathered — for local mode, this includes untracked
      files, not just `git diff`'s tracked-file output (§1, "Local review
      mode")
- [ ] Surrounding context pulled in beyond the raw diff: description,
      linked tickets, existing tests, related code
- [ ] Actual PR/change title used in the summary line — never invented
- [ ] All six review focus areas considered (§2), even the ones that turn
      up nothing
- [ ] Every finding has all five fields: Severity, Issue, Location, Why it
      matters, Recommended fix
- [ ] No findings invented just to fill an empty severity bucket
- [ ] Output follows the exact §5 structure, with empty severity sections
      omitted
- [ ] Scorecard notes and Overall Score actually reflect the findings, not
      a generic/default number
- [ ] Delivery matches mode: PR mode submits a formal review **and** shows
      the report in-conversation; local mode shows it in-conversation only
- [ ] PR review verdict matches the Merge Recommendation: Approve or
      Comment only → approved; Request changes → changes requested (§6)
- [ ] Any fetch or post failure is stated explicitly, not silently worked
      around

## Examples

**PR review mode** — the user supplies a URL:

> "review https://github.com/acme/widgets/pull/42"

→ Fetch PR #42's title, description, and diff from the host; produce the
report in §5's format using that title; submit it as a formal review on
PR #42 — approved if the Merge Recommendation is Approve or Comment only,
changes requested otherwise; also show it in the conversation.

**Local review mode** — no URL, uncommitted changes exist:

> "review my changes before I open a PR"

→ Run `git status` to see tracked *and* untracked changes, diff them (per
the "Local review mode" steps above), produce the report using a
descriptive title (e.g. from the branch name or a one-line summary of the
diff), and show it in the conversation only.

**Local review mode** — no URL, no uncommitted changes:

> "can you review the current branch?"

→ No uncommitted changes, so diff the current branch against its
base/target branch and review that instead.

**Local review mode** — brand-new repo, nothing committed yet:

> "review what I've got so far" (repo has zero commits)

→ No uncommitted-changes diff and no base branch to compare against, so
treat every tracked and untracked file in the working tree as the change
set (step 3 of "Local review mode").

See `references/examples.md` for two complete, full-length worked
examples — one per mode — showing an entire sample report end to end,
including how the fixed untracked-files bug shows up in practice.

## Testing this skill

`evals/evals.json` has example prompts covering both modes — including the
branch-vs-base and fresh-repo fallback cases — for use with this project's
skill evaluation workflow.
