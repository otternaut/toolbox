# Worked examples

These examples show the eligibility boundary and the final, concise report.
The coordinator filters and schedules; each worker independently runs
`otterbot-review` and delivers its own review.

## Example 1: Two delivered reviews

**User:**

> `otterbot-review-orchestrator https://github.com/acme/widgets`

At `2026-07-20T15:00:00Z`, #7 and #104 are eligible. Five other candidates
are excluded: one draft, one approved, one `stale`-labeled, one inactive, and
one already reviewed at the same effective revision.

```markdown
## 🦦 Review Orchestrator · acme/widgets

Two reviews were posted and verified. PR #7 needs changes; PR #104 can ship.

### 📊 Review Summary

- **Checked:** 7 pull requests
- **Excluded:** 5
  - Draft: 1
  - Review not required: 1
  - `stale` label: 1
  - Inactive for 14+ days: 1
  - Already reviewed, unchanged: 1
- **Eligible:** 2
- **Workers started:** 2
- **Delivered:** 2

### 📋 Results

#### ⚠️ Request Changes · [PR #7 · Fix duplicate webhook delivery](https://github.com/acme/widgets/pull/7)

The webhook path can still deliver the same event twice after a retry.

- **Review:** [Open Otterbot review](https://github.com/acme/widgets/pull/7#pullrequestreview-501)
- **Findings:** 2 inline findings — 1 high, 1 medium
- **Verification:** Retry and duplicate-delivery behavior needs a regression
  test.

#### 🚢 Ship It! · [PR #104 · Remove legacy retry worker](https://github.com/acme/widgets/pull/104)

The removal preserves the remaining retry path.

- **Review:** [Open Otterbot review](https://github.com/acme/widgets/pull/104#pullrequestreview-503)
- **Verification:** Focused tests passed.

### 🧭 Follow-up

Address PR #7's two findings and add the regression test, then rerun the sweep.
```

The coordinator does not reproduce Council reports, compare scores, or make a
combined merge recommendation.

## Example 2: A skip and a failed worker

**User:**

> `otterbot-review-pipeline https://github.com/acme/payhub/`

Three PRs pass the snapshot gate. #22 becomes approved while queued, so it is
skipped before a worker starts. #30's worker cannot load `otterbot-review`.

```markdown
## 🦦 Review Orchestrator · acme/payhub

One review was posted. PR #22 no longer needs review, and PR #30 needs a retry.

### 📊 Review Summary

- **Checked:** 3 pull requests
- **Eligible:** 3
- **Workers started:** 2
- **Delivered:** 1
- **Skipped:** 1
- **Failed:** 1

### 📋 Results

#### 💬 Comment Only · [PR #21 · Add payout reconciliation](https://github.com/acme/payhub/pull/21)

The reconciliation path is acceptable; one non-blocking note was posted.

- **Review:** [Open Otterbot review](https://github.com/acme/payhub/pull/21#pullrequestreview-800)
- **Findings:** 1 low-priority inline finding
- **Verification:** Focused reconciliation tests passed.

#### ⏸️ Skipped · [PR #22 · Update settlement schedule](https://github.com/acme/payhub/pull/22)

`reviewDecision` changed to `APPROVED` before worker creation.

#### ❌ Failed · [PR #30 · Harden webhook verification](https://github.com/acme/payhub/pull/30)

The worker could not load `otterbot-review`. No review was posted; retry when
the skill is available.

### 🧭 Follow-up

Retry PR #30 after restoring the required review skill. No action is needed for
PR #22.
```

## Example 3: No eligible pull requests

**User:**

> `review eligible PRs in https://github.com/acme/quiet-repo with otterbot`

The queue contains two drafts, one approved PR, one inactive PR, and one PR
already reviewed at the same effective revision. No worker starts.

```markdown
## 🦦 Review Orchestrator · acme/quiet-repo

No pull requests require an Otterbot review.

### 📊 Review Summary

- **Checked:** 5 pull requests
- **Excluded:** 5
  - Draft: 2
  - Review not required: 1
  - Inactive for 14+ days: 1
  - Already reviewed, unchanged: 1
- **Eligible:** 0
- **Workers started:** 0

All candidates were filtered before review. Exiting successfully.
```

## Example 4: Invalid input

**User:**

> `otterbot-review-orchestrator`

No repository URL was supplied. The coordinator does not inspect the checkout,
configured remotes, or branch names, and does not create workers. It responds:

> Provide one GitHub repository URL, for example
> `https://github.com/acme/widgets`.
