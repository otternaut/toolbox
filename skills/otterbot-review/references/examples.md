# Worked examples

Two complete, end-to-end examples of this skill's output — one per mode.
These are illustrative (the PR/repo don't exist); they show the level of
specificity and the exact formatting expected, not just the shape.

## Example 1: PR review mode

**User:** "review https://github.com/acme/payhub/pull/142"

**What happens:** a PR URL is present, so this is PR review mode (§1). The
PR's title ("Add webhook rate limiting"), description, linked Jira or Linear
ticket when present, current active review-thread state, and diff are fetched
from the host. Prior comments are used only to avoid duplicate active feedback,
not as material to review. The reviewer runs
or simulates the seven-category expert council, reconciles the internal notes,
and publishes only the final sanitized report. The report below is generated,
submitted as a formal review on PR #142, and also shown in the conversation
(§7).

**Report produced:**

```markdown
### 🦦 Council Review &middot; Add webhook rate limiting

#### 📝 Summary

Solid, focused change that adds a token-bucket limiter in front of the
webhook endpoint. Mergeable after the missing-lock issue below is fixed —
under concurrent requests the limiter can currently be bypassed.

#### ⚠️ Verdict · Request Changes

The race condition means the rate limit isn't reliably enforced, which
defeats the ticket's purpose — worth a fix before merge, not a fast-follow.

<details>
<summary>🦦 <strong>Specialist Scores</strong></summary>

<br>

> 📋 **Requirements Specialist · 🟡 70**
>
> - Jira ticket PAY-881 asks for no more than 50 requests/minute per merchant.
> - Product intent is clear and the technical approach is close, but concurrent traffic can exceed the intended limit.

> 🎯 **Correctness Specialist · 🟡 65**
>
> - Rate limiter has a race condition under concurrent requests.
> - The issue directly affects the core requirement because concurrent traffic can exceed the intended merchant limit.

> 🧩 **Completeness Specialist · 🟢 85**
>
> - Covers the main rate-limiting path.
> - Redis-unavailable behavior is not defined, so the operational failure mode still needs a decision.

> 🛡️ **Regression Risk Specialist · 🟡 80**
>
> - Limited blast radius because the change is scoped to webhook rate limiting.
> - Redis dependency behavior should be resolved before merge to avoid surprising webhook failures.

> 🧹 **Code Quality Specialist · 🟢 90**
>
> - Implementation is readable and follows local structure.
> - The remaining concerns are behavioral rather than structural.

> 🧪 **Testing Specialist · 🟡 60**
>
> - Unit tests cover the happy path only.
> - No concurrency test exercises the race condition.
> - No Redis-down test documents the intended fallback behavior.

> 🔒 **Security Specialist · 🟢 95**
>
> - No authorization or sensitive-data concerns found in this change.
> - The limiter does not appear to expose secrets or user data.

</details>

<details>
<summary>🔎 <strong>Findings Overview</strong></summary>

<br>

🟠 **High - 1 Issue**

> - Non-atomic rate-limit update in `src/webhooks/rateLimiter.ts`; posted inline on `checkRateLimit()`.

🟡 **Medium - 1 Issue**

> - Redis-down behavior is undefined; posted inline on the rate-limit call site.

🔵 **Low - 1 Issue**

> - Limit constant should move to shared config; posted inline on the constant declaration.

</details>

<details>
<summary>🧪 <strong>Testing</strong></summary>

<br>

> **Result · Not run in this review.**
>
> - I inspected the tests and changed code but did not execute the suite in this example review.
> - Confidence: medium; the highest-risk behavior depends on concurrent timing and should be proven with a runtime test.

> **Evidence inspected.**
>
> - `rateLimiter.test.ts` covers a single request under the limit and a single request over it.
> - The existing assertions prove basic allow/deny behavior, but all calls are sequential.

> **Risk analysis.**
>
> - The tests do not exercise the High atomicity finding, so the change could still violate the 50 requests/minute requirement under real webhook traffic.
> - Redis-down behavior is undefined and untested, which leaves production failure handling to chance.

> **Coverage gaps.**
>
> - Concurrency: the race condition in the High finding above has no test guarding it — this is the most important gap to close.
> - Redis-down behavior: no test simulates a Redis connection failure to verify the handler's response.

> **Recommended verification.**
>
> - Add an automated concurrency test that fires parallel requests for one merchant and expects no more than 50 accepted responses in the window.
> - Add a Redis-error test that asserts the chosen fallback behavior and logging signal.
> - Load-test the endpoint in staging after the atomic fix to confirm the limiter holds under realistic request timing.

</details>
```

**Inline comments posted:**

`src/webhooks/rateLimiter.ts`, `checkRateLimit()`:

```markdown
> **The read-check-increment sequence in `checkRateLimit()` isn't atomic, so concurrent requests from the same merchant can all read the same counter value and all pass the check.**
>
> - **Severity:** 🟠 High
> - **Why it matters:** Under real traffic (the exact scenario this change exists to handle), the limiter can be bypassed, silently defeating the ticket's requirement.
> - **Fix:** Use an atomic Redis operation (`INCR` + `EXPIRE`, or a Lua script) instead of separate GET/SET calls.
```

`src/webhooks/rateLimiter.ts`, rate-limit call site:

```markdown
> **No fallback behavior when Redis is unreachable — the handler currently throws, which would 500 all webhook traffic.**
>
> - **Severity:** 🟡 Medium
> - **Why it matters:** A transient Redis blip would take down payment webhook processing entirely, which is a worse outcome than a temporarily unenforced rate limit.
> - **Fix:** Fail open (allow the request, log a warning) on Redis errors, with an alert so the team notices.
```

`src/webhooks/rateLimiter.ts`, limit constant:

```markdown
> **Magic number `50` for the rate limit is inlined directly in the handler.**
>
> - **Severity:** 🔵 Low
> - **Why it matters:** Makes it easy to miss when tuning the limit later, and inconsistent with how other limits are configured elsewhere in the repo.
> - **Fix:** Pull from the existing `config/limits.ts`, matching the pattern used for the API rate limiter.
```

## Example 2: Local review mode

**User:** "review my changes before I open a PR"

**What happens:** no PR URL, so this is local review mode (§1). `git
status` shows one modified file and one new (untracked) file — both are
included in the diff per the "Local review mode" steps, since a plain
`git diff` alone would have missed the untracked file. The reviewer runs or
simulates the seven-category expert council and shares only the final sanitized
report in the conversation; nothing is posted anywhere (§7).

**Report produced:**

```markdown
#### 📝 Summary

Small, well-scoped addition. One real gap: the new export endpoint has no
authorization check, so any authenticated user could export any merchant's
transactions.

#### ⚠️ Verdict · Request Changes

The missing authorization check is a data-exposure issue and should block
merge on its own; everything else here is minor.

<details>
<summary>🦦 <strong>Specialist Scores</strong></summary>

<br>

> 📋 **Requirements Specialist · 🟡 65**
>
> - No Jira or Linear ticket was linked, so acceptance criteria are inferred from the branch name and diff.
> - The product requirement for merchant-scoped exports implies a technical authorization requirement, which the implementation does not enforce.

> 🎯 **Correctness Specialist · 🟢 90**
>
> - Export logic is straightforward and appears to return the intended data.
> - No obvious data-shaping or pagination bug is visible from the changed code.

> 🧩 **Completeness Specialist · 🟡 70**
>
> - Missing authorization check blocks a complete implementation.
> - Acceptance criteria are unclear without a linked ticket or spec.

> 🛡️ **Regression Risk Specialist · 🟢 90**
>
> - New endpoint is isolated from existing flows.
> - The security flaw affects exposed data even though it is not a broad regression.

> 🧹 **Code Quality Specialist · 🟢 85**
>
> - Code is simple and follows the surrounding handler style.
> - The implementation would benefit from colocated authorization handling matching adjacent endpoints.

> 🧪 **Testing Specialist · 🔴 50**
>
> - No tests added for the new endpoint.
> - No authorization regression test exists for cross-merchant access.

> 🔒 **Security Specialist · 🔴 40**
>
> - Missing authorization check is a real data-exposure risk.
> - A merchant could potentially export another merchant's transactions.

</details>

<details>
<summary>🔎 <strong>Findings Overview</strong></summary>

<br>

🔴 **Critical - 1 Issue**

> **`GET /merchants/:id/export` doesn't verify the requesting user has access to `:id` — it only checks that the user is authenticated, not that they belong to that merchant.**
> - **Location:** `src/routes/exportController.ts` (new file, untracked)
> - **Why it matters:** Any authenticated user can export any other merchant's full transaction history by changing the `:id` in the URL — a direct data-exposure issue.
> - **Fix:** Add the same merchant-ownership check used in `src/routes/transactionsController.ts:getTransactions()` before generating the export.
>
> ```ts
> router.get('/merchants/:id/export', requireAuth, async (req, res) => {
>   const rows = await getTransactions(req.params.id); // no ownership check on :id
>   res.type('text/csv').send(toCsv(rows));
> });
> ```

🔵 **Low - 1 Issue**

> **No tests for the new endpoint.** (No code block — there's nothing to quote for a missing test.)
> - **Location:** `src/routes/exportController.ts` (new file, untracked)
> - **Why it matters:** Nothing currently guards against a regression here, and the missing-auth issue above is exactly the kind of thing a test would have caught.
> - **Fix:** Add a test asserting a user from merchant A gets a 403 when requesting merchant B's export.

</details>

<details>
<summary>🧪 <strong>Testing</strong></summary>

<br>

> **Result · Not run in this review.**
>
> - I inspected the diff and nearby tests but did not execute the route test suite.
> - Confidence: low-to-medium until an authorization regression test covers the Critical finding.

> **Evidence inspected.**
>
> - No automated tests exist for the new export endpoint (`exportController.ts` is untracked with no accompanying test file).
> - Existing CSV-format tests elsewhere in the repo (e.g. `transactionsController.test.ts`) don't cover this new route.

> **Risk analysis.**
>
> - The missing ownership check is a security boundary, so this needs a targeted negative test before merge rather than only manual spot-checking.
> - CSV formatting has some adjacent coverage, but the new route's access control and input handling are untested.

> **Coverage gaps.**
>
> - Authorization: no test exercises the missing ownership check from the Critical finding above — a test here would have caught it directly.
> - Edge cases: no test for a merchant with zero transactions or an invalid `:id`.

> **Recommended verification.**
>
> - Add a route test asserting a user from merchant A gets a 403 when requesting merchant B's export.
> - Add a positive same-merchant test so the ownership fix does not break valid exports.
> - Manually hit the endpoint as a user from merchant A requesting merchant B's `:id` to confirm the fix actually blocks it before merge.

</details>
```

Note how the missing-untracked-file bug this skill was fixed for shows up
directly in this example: `exportController.ts` is a **new, untracked**
file, and it's still reviewed — including a Critical finding inside it.
