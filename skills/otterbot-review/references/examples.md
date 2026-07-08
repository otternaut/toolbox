# Worked examples

Two complete, end-to-end examples of this skill's output — one per mode.
These are illustrative (the PR/repo don't exist); they show the level of
specificity and the exact formatting expected, not just the shape.

## Example 1: PR review mode

**User:** "review https://github.com/acme/payhub/pull/142"

**What happens:** a PR URL is present, so this is PR review mode (§1). The
PR's title, description, and diff are fetched from the host. The report
below is generated, posted as a single comment on PR #142, and also shown
in the conversation (§6).

**Report produced:**

```markdown
# 🦦 OtterBot Code Review

## ✨ Summary — Add rate limiting to payment webhook handler

Solid, focused change that adds a token-bucket limiter in front of the
webhook endpoint. Mergeable after the missing-lock issue below is fixed —
under concurrent requests the limiter can currently be bypassed.

## 📋 Requirements Check

Ticket PAY-881 asks for "no more than 50 requests/minute per merchant." The
implementation matches that at the code level, but see the High finding
below — under load, the actual enforced rate can exceed the limit.

## 📊 Scorecard

| Criteria | Score | Notes |
| --- | ---: | --- |
| Correctness | 65 | Rate limiter has a race condition under concurrent requests. |
| Completeness | 85 | Covers the main path; no handling for the limiter's Redis dependency being unavailable. |
| Regression Risk | 80 | Isolated to the webhook handler; existing endpoints untouched. |
| Code Quality | 90 | Clean, well-named, consistent with existing handler patterns. |
| Testing | 60 | Unit tests cover the happy path only; no concurrency or Redis-down test. |
| Security & Data Integrity | 95 | No new sensitive data handling; rate limiting is itself a security improvement. |

**Overall Score:** 79

**Merge Recommendation:** ⚠️ Request changes

The race condition means the rate limit isn't reliably enforced, which
defeats the ticket's purpose — worth a fix before merge, not a fast-follow.

## 🔎 Findings

### 🟠 High

> **Issue:** The read-check-increment sequence in `checkRateLimit()` isn't
> atomic, so concurrent requests from the same merchant can all read the
> same counter value and all pass the check.
>
> **Location:** `src/webhooks/rateLimiter.ts`, `checkRateLimit()`
>
> **Why it matters:** Under real traffic (the exact scenario this change
> exists to handle), the limiter can be bypassed, silently defeating the
> ticket's requirement.
>
> **Recommended fix:** Use an atomic Redis operation (`INCR` + `EXPIRE`, or
> a Lua script) instead of separate GET/SET calls.

### 🟡 Medium

> **Issue:** No fallback behavior when Redis is unreachable — the handler
> currently throws, which would 500 all webhook traffic.
>
> **Location:** `src/webhooks/rateLimiter.ts`, `checkRateLimit()`
>
> **Why it matters:** A transient Redis blip would take down payment
> webhook processing entirely, which is a worse outcome than a temporarily
> unenforced rate limit.
>
> **Recommended fix:** Fail open (allow the request, log a warning) on
> Redis errors, with an alert so the team notices.

### 🔵 Low

> **Issue:** Magic number `50` for the rate limit is inlined directly in
> the handler.
>
> **Location:** `src/webhooks/rateLimiter.ts:23`
>
> **Why it matters:** Makes it easy to miss when tuning the limit later, and
> inconsistent with how other limits are configured elsewhere in the repo.
>
> **Recommended fix:** Pull from the existing `config/limits.ts`, matching
> the pattern used for the API rate limiter.

## 🧪 Testing Notes

Existing unit tests cover a single request under the limit and a single
request over it. Missing: a concurrency test that would have caught the
High finding, and a test for Redis-unavailable behavior. Recommend adding
both before merge.
```

## Example 2: Local review mode

**User:** "review my changes before I open a PR"

**What happens:** no PR URL, so this is local review mode (§1). `git
status` shows one modified file and one new (untracked) file — both are
included in the diff per the "Local review mode" steps, since a plain
`git diff` alone would have missed the untracked file. The report is shown
in the conversation only; nothing is posted anywhere (§6).

**Report produced:**

```markdown
# 🦦 OtterBot Code Review

## ✨ Summary — Add CSV export for merchant transactions

Small, well-scoped addition. One real gap: the new export endpoint has no
authorization check, so any authenticated user could export any merchant's
transactions.

## 📋 Requirements Check

No ticket was linked; based on the branch name (`csv-export`) and the diff,
this appears to implement a merchant-facing "export transactions" feature.
Can't confirm acceptance criteria without a ticket — worth double-checking
against whatever spec exists before merging.

## 📊 Scorecard

| Criteria | Score | Notes |
| --- | ---: | --- |
| Correctness | 90 | CSV generation logic is correct and handles empty result sets. |
| Completeness | 70 | Missing authorization check (see High finding). |
| Regression Risk | 90 | New endpoint, doesn't touch existing routes. |
| Code Quality | 85 | Consistent with existing route/controller structure. |
| Testing | 50 | No tests added for the new endpoint (new file, untracked). |
| Security & Data Integrity | 40 | Missing authorization check is a real data-exposure risk. |

**Overall Score:** 71

**Merge Recommendation:** ⚠️ Request changes

The missing authorization check is a data-exposure issue and should block
merge on its own; everything else here is minor.

## 🔎 Findings

### 🔴 Critical

> **Issue:** `GET /merchants/:id/export` doesn't verify the requesting
> user has access to `:id` — it only checks that the user is authenticated,
> not that they belong to that merchant.
>
> **Location:** `src/routes/exportController.ts` (new file, untracked)
>
> **Why it matters:** Any authenticated user can export any other
> merchant's full transaction history by changing the `:id` in the URL —
> a direct data-exposure issue.
>
> **Recommended fix:** Add the same merchant-ownership check used in
> `src/routes/transactionsController.ts:getTransactions()` before
> generating the export.

### 🔵 Low

> **Issue:** No tests for the new endpoint.
>
> **Location:** `src/routes/exportController.ts` (new file, untracked)
>
> **Why it matters:** Nothing currently guards against a regression here,
> and the missing-auth issue above is exactly the kind of thing a test
> would have caught.
>
> **Recommended fix:** Add a test asserting a user from merchant A gets a
> 403 when requesting merchant B's export.

## 🧪 Testing Notes

No automated tests exist for the new export endpoint. Recommend adding
authorization tests (see Low finding) alongside the existing CSV-format
tests before opening the PR.
```

Note how the missing-untracked-file bug this skill was fixed for shows up
directly in this example: `exportController.ts` is a **new, untracked**
file, and it's still reviewed — including a Critical finding inside it.
