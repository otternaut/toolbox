---
name: otterbot-review
description: Perform a principal-level code review, producing a structured Markdown report with severity-tagged findings and a scorecard. Given a pull/merge request URL, reviews that PR and submits the report as a formal review — approving it when the recommendation is Approve or Comment Only, requesting changes otherwise. Given no URL, reviews the current local code changes and presents the report in the conversation. Use this whenever the user asks to "review this PR", "review my diff", "analyze this code change", "do a code review", "check this pull request for issues", pastes a pull-request URL and asks for feedback, or wants a merge-readiness assessment. Works with any git hosting provider (GitHub, GitLab, Bitbucket, etc.).
version: 1.6.0
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
  requested) follows the Merge Recommendation. See §7.
- **No PR URL → local review mode.** Review the current local code changes
  in this repository and present the report in the conversation. See §7.

If the user's intent is ambiguous in some other way (e.g. multiple PR links,
or a claim of local changes that don't actually exist), ask before
proceeding rather than guessing. Otherwise, don't ask which mode to use —
the presence or absence of a URL decides it.

### PR review mode

Fetch the PR's title, description, linked tickets, changed files, and diff
using whatever access you have in the current environment (a CLI for that
host, an API, a browsing tool). Use this context to inform the review —
never invent or assume it.

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

## 3. Parallel specialist review

After gathering the change set and surrounding context, use subagents when the
current environment supports them. The goal is faster review without weakening
the judgment: specialists run in parallel, then their work is reconciled into
one final principal-level report.

Launch one specialist per scorecard category:

- **Correctness specialist:** bugs, edge cases, broken logic, races, error
  handling, and requirement fit.
- **Completeness specialist:** missing states, validation, integrations,
  cleanup, migrations, rollout needs, and partial implementation.
- **Regression Risk specialist:** side effects on existing flows, APIs, data,
  UI behavior, permissions, performance, deployment, compatibility, and
  rollback.
- **Code Quality specialist:** readability, structure, naming, duplication,
  abstractions, typing, local conventions, maintainability, and unnecessary
  complexity.
- **Testing specialist:** existing coverage, missing automated tests,
  regression tests, contract/integration coverage, and manual verification
  proportional to risk.
- **Security specialist:** authorization, unsafe inputs, secret leakage,
  validation gaps, data loss, sensitive data handling, supply-chain exposure,
  and privacy concerns.

Give every specialist the same factual packet: mode, PR or local change
description, full diff including untracked files, relevant surrounding code,
tests, requirements, linked tickets, and any repo-specific guidance. Tell each
specialist to stay inside its category but to report cross-category evidence
when it changes severity or merge readiness. Ask each specialist to return:

- Category score from 0-100 with a concise rationale.
- Candidate findings with severity, issue, location, why it matters,
  recommended fix, and evidence.
- Blocking/non-blocking recommendation for that category.
- Confidence level and the facts or assumptions the confidence depends on.
- Missing context that would materially change the conclusion.

Run these specialists concurrently when possible. If the environment has no
subagent capability, perform the same six specialist passes yourself in a
single thread; do not skip any category or change the report format.

### Specialist debate and adjudication

Once all specialist passes return, synthesize them through an explicit
challenge round before writing the report:

1. Compare overlapping findings and merge duplicates, keeping the clearest
   location, impact statement, and fix.
2. Challenge each potential blocker from the opposite direction: ask what
   evidence would make it non-blocking, and whether that evidence is present.
3. Challenge each approve/comment-only path from the failure direction: ask
   what user, data, security, deploy, rollback, or compatibility issue could
   still make the change unsafe to merge.
4. Reconcile score disagreements by tying scores to concrete risk, coverage,
   and findings. Do not average scores mechanically if one category found a
   severe issue that changes merge readiness.
5. Drop speculative findings that lack code, requirement, or operational
   evidence. Keep well-supported findings even if only one specialist found
   them.
6. Choose the final verdict from the reconciled evidence:
   - **Request Changes** when any Critical/High issue blocks safe merge, or
     when a Medium issue is directly tied to an unmet requirement, data loss,
     security exposure, or untested high-risk behavior.
   - **Comment Only** when the change is mergeable but has meaningful
     non-blocking concerns worth recording.
   - **Approve** when the change is merge-ready and remaining concerns, if
     any, are optional or low-risk.

The debate is an internal quality gate. The delivered output remains the
single report in §6; do not include raw specialist transcripts unless the user
explicitly asks for them.

## 4. Severity levels

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

## 5. Review standards

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

## 6. Output format

Produce the report in exactly this structure. Keep it clean and scannable:
plain section headings and each finding in its own callout card. Use a
top-level heading (`##`) for the title so its underline rule visually
separates it from the rest of the report. Do **not** add horizontal rules
anywhere else — not between top-level sections (Summary, Requirements,
Scorecard, Recommendation, Findings, Testing) and not between individual finding cards
within Findings — the section headings and the blockquote cards already
create enough visual separation on their own.

```markdown
## 🦦 OtterBot · Code Review

Briefly state overall quality, merge readiness, and the highest-risk
concern, if any. Leave the verdict itself to the Recommendation section —
the Summary sets up the narrative, not the decision.

### 📋 Requirements

State whether the change appears to satisfy the intended business/product
requirements. Note any ambiguity or missing acceptance criteria.

### 📊 Scorecard

Score each category from 0-100, where 100 means excellent and merge-ready
with no meaningful concerns.

Keep the table to just `Category` and `Score` — GitHub's renderer sizes
table columns by available width, not by source padding, so a wide `Notes`
cell squeezes the other columns until even short header words wrap
mid-word. Don't put the rationale in the table; collect it as a bullet list
under a **Score Notes** label below instead.

| Category            | Score      |
| ------------------- | ---------: |
| Correctness         | 0-100      |
| Completeness        | 0-100      |
| Regression Risk     | 0-100      |
| Code Quality        | 0-100      |
| Testing             | 0-100      |
| Security            | 0-100      |

Follow the table with the **Overall Score** line (a middot then `N / 100`,
so the scale is explicit and the style matches elsewhere), then a **Score
Notes** label and a bullet list — one bullet per category that needs context
(isn't self-evidently near-perfect). Lead each bullet with the bold category
name so it ties back to the table. Skip categories that don't need comment.

**Overall Score** · 0-100 / 100

**Score Notes**

- **Correctness:** Brief rationale.
- **Testing:** Brief rationale.
- **Security:** Brief rationale.

### ⚠️ Recommendation · Request Changes

The verdict lives in the heading itself: set the emoji dynamically to match
the call and name the verdict after a middot — `### ✅ Recommendation · Approve`,
`### ⚠️ Recommendation · Request Changes`, or `### 💬 Recommendation · Comment
Only`. This keeps the section identifiable while surfacing the decision in the
header's own underline rule, with no separate banner line in the body.

Open the body with the 1-2 sentence explanation of the call. When the verdict
is ⚠️ Request Changes, follow with a **Blocking** label and a bullet list of
the specific must-fix items that gate merge — each a one-liner that ends with
the severity of the finding it maps to, e.g. `(High)`, rather than restating
the whole finding. For ✅ Approve or 💬 Comment Only, omit **Blocking**;
optionally add a **Follow-ups** label with non-blocking suggestions worth
doing later.

Explain the recommendation in 1-2 concise sentences.

**Blocking**

- Concrete must-fix item that gates merge. (High)
- Another must-fix item. (Medium)

### 🔎 Findings

Group findings by severity, most severe first. Severity headings use the
exact emoji and labels from §4 (🔴 Critical, 🟠 High, 🟡 Medium, 🔵 Low,
⚪ Optional) followed by a middot and the count of findings in that
section, e.g. `#### 🟠 High · 2 Issues`. Use singular "Issue" for a count
of exactly one, e.g. `#### 🟡 Medium · 1 Issue`. Omit empty severity
sections unless there are no findings at all.

Present each finding as a blockquote card: a bold one-line issue statement,
then a tight bullet list, and finally — when you have the source and it makes
the problem concrete — a short fenced code block quoting the exact lines the
finding refers to. Tag the code block with the file's language and keep it to
the few relevant lines. Omit the code block when there's nothing useful to
show (e.g. a finding about missing code, absent tests, or a design-level
concern). Every line of the code block, **including its fences**, is
prefixed with `>` so it stays inside the card. Put a blank line between
consecutive cards so they render as separate callouts — no horizontal rules:

#### 🟠 High · 2 Issues

> **Clear, concise statement of the problem.**
>
> - **Location:** `path/to/file.ext`, function/class/section, or changed behavior.
> - **Why it matters:** The user, system, data, security, or maintenance impact.
> - **Fix:** The concrete change needed.
>
> ```ts
> // the exact lines from the change that the finding refers to
> const count = await redis.get(key);
> if (Number(count) >= LIMIT) await redis.set(key, Number(count) + 1);
> ```

> **Another separate finding, stated in one line.**
>
> - **Location:** `path/to/other-file.ext`
> - **Why it matters:** The risk.
> - **Fix:** The fix.

#### 🟡 Medium · 1 Issue

> **Issue stated in one line.**
>
> - **Location:** ...
> - **Why it matters:** ...
> - **Fix:** ...

### 🧪 Testing

Give a thorough, specific picture of testing — not a one-line note. Cover,
as applicable:

- **Existing coverage:** which tests exist (unit, integration, e2e, manual
  QA) and exactly what they exercise — file/test names when you have them.
- **Coverage gaps:** concrete scenarios that aren't covered — edge cases,
  error paths, concurrency, migrations, rollback, permission boundaries —
  especially any gap that relates directly to a finding above.
- **Recommended manual verification:** specific steps worth running by
  hand before or after merge when automated coverage doesn't fully address
  the risk (e.g. load-testing a race condition, exercising a failure mode
  that's hard to simulate in CI).

Use blockquote cards matching the Findings style: a bold one-line point,
then a tight bullet list of specifics underneath when there's more to say.
Blank line between cards, no horizontal rules.

> **Existing coverage.**
>
> - `rateLimiter.test.ts` covers a single request under the limit and a single request over it.
> - No coverage for concurrent requests hitting the same merchant key.

> **Coverage gaps.**
>
> - Concurrency: the race condition in the High finding above has no test guarding it.
> - Redis-down behavior: no test simulates a Redis connection failure to verify the handler's response.

> **Recommended manual verification.**
>
> - Load-test the endpoint with concurrent requests from a single merchant to confirm the limiter holds once the race is fixed.
> - Manually break the Redis connection in staging to observe current failure behavior before deciding on a fallback.
```

## 7. Delivering the review

Delivery follows the mode determined in §1:

- **PR review mode:** submit the completed report as a single formal review
  on the PR/MR — not just a plain comment — using whatever tool your
  environment provides for that host (e.g. a hosting-provider CLI or API).
  Set the review verdict from the **Recommendation** in §6:

  - ✅ **Approve** or 💬 **Comment Only** → submit the review as **approved**.
  - ⚠️ **Request Changes** → submit the review as **changes requested**.

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

## 8. Completeness checklist

Before delivering the report, confirm all of these — they're drawn directly
from mistakes this skill has made in practice (see `references/examples.md`
for what a full pass looks like):

- [ ] Correct mode determined from presence/absence of a PR/MR URL (§1)
- [ ] Full change set gathered — for local mode, this includes untracked
      files, not just `git diff`'s tracked-file output (§1, "Local review
      mode")
- [ ] Surrounding context pulled in beyond the raw diff: description,
      linked tickets, existing tests, related code
- [ ] All six review focus areas considered (§2), even the ones that turn
      up nothing
- [ ] One specialist pass completed for each scorecard category, using
      parallel subagents when available and an explicit serial fallback when
      they are not (§3)
- [ ] Specialist results were challenged and reconciled before scoring or
      choosing the final verdict (§3, "Specialist debate and adjudication")
- [ ] Every finding has all five fields: Severity, Issue, Location, Why it
      matters, Recommended fix
- [ ] No findings invented just to fill an empty severity bucket
- [ ] Output follows the exact §6 structure, with empty severity sections
      omitted, no horizontal rules, and each severity heading tagged with its
      finding count
- [ ] Scorecard notes and Overall Score actually reflect the findings, not
      a generic/default number
- [ ] Delivery matches mode: PR mode submits a formal review **and** shows
      the report in-conversation; local mode shows it in-conversation only
- [ ] PR review verdict matches the Merge Recommendation: Approve or
      Comment Only → approved; Request Changes → changes requested (§7)
- [ ] Any fetch or post failure is stated explicitly, not silently worked
      around

## Examples

**PR review mode** — the user supplies a URL:

> "review https://github.com/acme/widgets/pull/42"

→ Fetch PR #42's title, description, and diff from the host; produce the
report in §6's format; submit it as a formal review on PR #42 — approved
if the Merge Recommendation is Approve or Comment Only, changes requested
otherwise; also show it in the conversation.

**Local review mode** — no URL, uncommitted changes exist:

> "review my changes before I open a PR"

→ Run `git status` to see tracked *and* untracked changes, diff them (per
the "Local review mode" steps above), produce the report, and show it in
the conversation only.

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
