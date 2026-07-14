# Worked examples

Three complete, end-to-end examples of this skill's output — an initial PR
review, a PR re-review, and a local review.
These are illustrative (the PR/repo don't exist); they show the level of
specificity and the exact formatting expected, not just the shape.

## Example 1: PR review mode

**User:** "review https://github.com/acme/payhub/pull/142"

**What happens:** a PR URL is present, so this is PR review mode (§1). The
PR's title ("Add webhook rate limiting"), description, linked Jira or Linear
ticket when present, current active review-thread state, and diff are fetched
from the host. Prior comments are used only to avoid duplicate active feedback,
not as material to review. The reviewer runs or simulates the seven core
specialists plus the Reliability & Operability and Performance & Scalability
specialists activated by the external Redis dependency and traffic-sensitive
request path. It reconciles the internal notes and publishes only the final
sanitized report. The report below is generated, submitted as a formal review
on PR #142, and also shown in the conversation (§7).

**Report produced:**

```markdown
<!-- otterbot-review: council; head: 8b2c6e1d4a9f7c3b5e0d1a8c6f2b9e4d7a3c1f0e -->

#### 🦦 Council Review &middot; Add webhook rate limiting

#### 📝 Summary

Solid, focused change that adds a token-bucket limiter in front of the
webhook endpoint. Mergeable after the missing-lock issue below is fixed —
under concurrent requests the limiter can currently be bypassed.

#### ⚠️ Verdict · Request Changes

The race condition means the rate limit isn't reliably enforced, which
defeats the ticket's purpose — worth a fix before merge, not a fast-follow.

<details>
<summary>🦦 <strong>Scorecard</strong></summary>

<br>

> 📋 **Product Intent & Acceptance Specialist · 🟡 70**
>
> - Jira ticket PAY-881 asks for no more than 50 requests/minute per merchant.
> - Product intent is clear, but concurrent traffic can exceed the intended limit.

> 🎯 **Functional Correctness & State Specialist · 🟡 65**
>
> - Rate limiter has a race condition under concurrent requests.
> - The issue directly affects the core requirement because concurrent traffic can exceed the intended merchant limit.

> 🔗 **Integration & Contract Completeness Specialist · 🟢 85**
>
> - The endpoint is connected to the limiter and the Redis integration is present.
> - The Redis failure contract is not defined, so consumers cannot rely on a documented fallback.

> 🛡️ **Compatibility & Regression Specialist · 🟡 80**
>
> - Limited blast radius because the change is scoped to webhook rate limiting.
> - Redis dependency behavior should be resolved before merge to avoid surprising webhook failures.

> 🏗️ **Architecture & Maintainability Specialist · 🟢 90**
>
> - Implementation is readable and follows local structure.
> - The remaining concerns are behavioral rather than structural.

> 🧪 **Verification & Test Quality Specialist · 🟡 60**
>
> - Unit tests cover the happy path only.
> - No concurrency test exercises the race condition.
> - No Redis-down test documents the intended fallback behavior.

> 🔒 **Security & Privacy Specialist · 🟢 95**
>
> - No authorization or sensitive-data concerns found in this change.
> - The limiter does not appear to expose secrets or user data.

> 📡 **Reliability & Operability Specialist · 🔴 55**
>
> - Activated because: webhook handling now depends on Redis during every request.
> - Redis-unavailable behavior is undefined, so a transient dependency failure can return 500 for all webhook traffic.
> - No logging or alerting evidence shows operators can diagnose that failure.

> ⚡ **Performance & Scalability Specialist · 🟡 75**
>
> - Activated because: the change adds Redis I/O and concurrency-sensitive rate limiting to a production traffic path.
> - No load evidence shows the limiter maintains acceptable latency or enforces the threshold under concurrent requests.

</details>

<details>
<summary>🔎 <strong>Findings</strong></summary>

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
<!-- otterbot-finding: rate-limiter-atomicity -->

> **The read-check-increment sequence in `checkRateLimit()` isn't atomic, so concurrent requests from the same merchant can all read the same counter value and all pass the check.**
>
> - **Severity:** 🟠 High
> - **Why it matters:** Under real traffic (the exact scenario this change exists to handle), the limiter can be bypassed, silently defeating the ticket's requirement.
> - **Fix:** Use an atomic Redis operation (`INCR` + `EXPIRE`, or a Lua script) instead of separate GET/SET calls.
```

`src/webhooks/rateLimiter.ts`, rate-limit call site:

```markdown
<!-- otterbot-finding: redis-failure-contract -->

> **No fallback behavior when Redis is unreachable — the handler currently throws, which would 500 all webhook traffic.**
>
> - **Severity:** 🟡 Medium
> - **Why it matters:** A transient Redis blip would take down payment webhook processing entirely, which is a worse outcome than a temporarily unenforced rate limit.
> - **Fix:** Fail open (allow the request, log a warning) on Redis errors, with an alert so the team notices.
```

`src/webhooks/rateLimiter.ts`, limit constant:

```markdown
<!-- otterbot-finding: rate-limit-config -->

> **Magic number `50` for the rate limit is inlined directly in the handler.**
>
> - **Severity:** 🔵 Low
> - **Why it matters:** Makes it easy to miss when tuning the limit later, and inconsistent with how other limits are configured elsewhere in the repo.
> - **Fix:** Pull from the existing `config/limits.ts`, matching the pattern used for the API rate limiter.
```

## Example 2: PR re-review mode

**User:** "re-review https://github.com/acme/payhub/pull/142"

**What happens:** the same PR URL has an attributable Otterbot Council review
with the `<!-- otterbot-review: council -->` marker. The agent fetches the
current PR state, including the commits since the first review, full current
diff, active threads, and tests. It then runs the complete specialist council
again — it does not merely inspect the prior comments. The atomic update and
its concurrency test now directly address the original High finding, so the
agent resolves its own inline thread. Redis failure handling is still
unverified, and a new missing metric is found. The host cannot edit the
submitted review, so the agent posts exactly one re-review comment that links
to and supersedes it. It also submits a new Comment Only formal review so the
earlier Request Changes decision no longer represents the current revision.
After verifying the replacement and recreating the active Redis finding, it
uses `gh api graphql` to minimize every prior attributable Otterbot comment
with `classifier: OUTDATED`.

**Report produced:**

```markdown
<!-- otterbot-review: council; head: d7f4a9c2e8b1d6f0a3c5e7b9d1f2a4c6e8b0d3f5 -->

#### 🦦 Council Review &middot; Add webhook rate limiting

#### 📝 Summary

The original atomicity blocker is fixed and covered by a concurrent-request
test. The remaining Redis observability gap and new metric omission are
non-blocking, so this revision is mergeable with recorded follow-up work.

#### 💬 Verdict · Comment Only

This is a re-review of the Request Changes review from 2026-07-12T14:20:00Z:
the High atomicity finding is resolved, while the remaining Medium and new Low
findings do not block a safe merge.

<details>
<summary>🦦 <strong>Scorecard</strong></summary>

<br>

> 📋 **Product Intent & Acceptance Specialist · 🟢 92**
>
> - The revised implementation now enforces the 50 requests/minute requirement under concurrent traffic.

> 🎯 **Functional Correctness & State Specialist · 🟢 90**
>
> - The atomic Redis operation removes the prior check-then-set race.
> - The parallel-request test exercises the counterexample from the original review.

> 🔗 **Integration & Contract Completeness Specialist · 🟢 85**
>
> - The limiter remains connected to the webhook path and its Redis contract is unchanged.

> 🛡️ **Compatibility & Regression Specialist · 🟢 88**
>
> - The scoped limiter does not alter unrelated webhook routes.

> 🏗️ **Architecture & Maintainability Specialist · 🟢 90**
>
> - The atomic operation is localized and follows the existing Redis helper pattern.

> 🧪 **Verification & Test Quality Specialist · 🟢 88**
>
> - The added concurrency test directly covers the original blocking scenario.
> - Redis-unavailable behavior remains untested.

> 🔒 **Security & Privacy Specialist · 🟢 95**
>
> - No authorization, sensitive-data, or secret-handling regression found.

> 📡 **Reliability & Operability Specialist · 🟡 76**
>
> - Activated because: webhook handling still depends on Redis during every request.
> - The implementation logs Redis failures but does not emit the rate-limit fallback metric.

> ⚡ **Performance & Scalability Specialist · 🟢 84**
>
> - Activated because: the request path performs Redis I/O under concurrent production traffic.
> - The atomic operation removes race retries; staging load evidence is still recommended.

</details>

<details>
<summary>🔎 <strong>Findings</strong></summary>

<br>

> - ~~High: Non-atomic rate-limit update in `src/webhooks/rateLimiter.ts`.~~ **Resolved:** replaced by an atomic Redis operation and verified by the new concurrent-request test; original inline thread resolved.

🟡 **Medium - 1 Issue**

> - Redis-down behavior remains untested; recreated as a current-generation inline comment with its stable finding ID.

🔵 **Low - 1 Issue**

> - New in this re-review: the fail-open fallback has no metric, so operators cannot quantify how often protection is bypassed; posted inline on the fallback branch.

</details>

<details>
<summary>🧪 <strong>Testing</strong></summary>

<br>

> **Result · Passed.**
>
> - `rateLimiter.test.ts` passed, including the new parallel-request test that permits no more than 50 requests in the window.
> - Redis-unavailable behavior was inspected but not executed in this re-review.

> **Risk analysis.**
>
> - The original merge blocker is now directly tested.
> - The remaining Medium finding is a verification gap, not evidence of an unsafe fallback implementation.

> **Recommended verification.**
>
> - Add a Redis-error test that asserts the chosen fail-open response, warning log, and fallback metric.
> - Load-test the endpoint in staging before the next traffic increase.

</details>

<details>
<summary>🕰️ <strong>History</strong></summary>

<br>

> **Original review · 2026-07-12T14:20:00Z · Request Changes**
>
> - Review: https://github.com/acme/payhub/pull/142#pullrequestreview-100
> - Head: `8b2c6e1d4a9f7c3b5e0d1a8c6f2b9e4d7a3c1f0e`.
> - Findings: 1 High atomicity issue, 1 Medium Redis-down gap, 1 Low configuration suggestion.
> - Status: Superseded by re-review at 2026-07-14T15:45:00Z.

> **Re-review · 2026-07-14T15:45:00Z · Comment Only**
>
> - Review: https://github.com/acme/payhub/pull/142#issuecomment-200; supersedes the original because editing the submitted review was unavailable.
> - Head: `d7f4a9c2e8b1d6f0a3c5e7b9d1f2a4c6e8b0d3f5`.
> - Findings: 1 resolved, 1 active, 0 no-longer-applicable, and 1 newly discovered.
> - Status: Mergeable; no blocking findings remain.

</details>
```

The agent resolves only the original High inline thread, because it owns that
thread and the concurrent test directly verifies the fix. It recreates the
still-active Medium finding with its stable ID and posts the new Low finding
inline with a reference to the original review. After verifying those new
comments, it runs the required `gh api graphql` `minimizeComment` mutation with
`classifier: OUTDATED` once for every prior Otterbot Council root, inline
finding, and reply. It verifies `isMinimized: true` and
`minimizedReason: "OUTDATED"` for each one, then refetches the discussion to
confirm that only the current Otterbot generation remains expanded.

If this review is also delivered to the super.engineering in-app surface, the
agent uses its generation lifecycle there rather than copying the hosting
provider's thread lifecycle. It resolves every attributable prior
`otterbot-review` root, inline comment, and reply as superseded, posts one new
Council root containing the cumulative History above, then recreates the active
Medium and Low findings as current-generation inline comments. The fixed High
finding remains in History but does not get a new inline thread.

## Example 3: Local review mode

**User:** "review my changes before I open a PR"

**What happens:** no PR URL, so this is local review mode (§1). `git
status` shows one modified file and one new (untracked) file — both are
included in the diff per the "Local review mode" steps, since a plain
`git diff` alone would have missed the untracked file. The reviewer runs or
simulates the seven core specialists plus Data Integrity & Persistence because
the endpoint reads durable transaction records. It shares only the final
sanitized report in the conversation; nothing is posted anywhere (§7).

**Report produced:**

````markdown
#### 📝 Summary

Small, well-scoped addition. One real gap: the new export endpoint has no
authorization check, so any authenticated user could export any merchant's
transactions.

#### ⚠️ Verdict · Request Changes

The missing authorization check is a data-exposure issue and should block
merge on its own; everything else here is minor.

<details>
<summary>🦦 <strong>Scorecard</strong></summary>

<br>

> 📋 **Product Intent & Acceptance Specialist · 🟡 65**
>
> - No Jira or Linear ticket was linked, so acceptance criteria are inferred from the branch name and diff.
> - The product requirement for merchant-scoped exports implies a technical authorization requirement, which the implementation does not enforce.

> 🎯 **Functional Correctness & State Specialist · 🟢 90**
>
> - Export logic is straightforward and appears to return the intended data.
> - No obvious data-shaping or pagination bug is visible from the changed code.

> 🔗 **Integration & Contract Completeness Specialist · 🟡 70**
>
> - The route is registered and the CSV conversion path is connected.
> - The route does not integrate the merchant-ownership guard used by adjacent endpoints.

> 🛡️ **Compatibility & Regression Specialist · 🟢 90**
>
> - New endpoint is isolated from existing flows.
> - The security flaw affects exposed data even though it is not a broad regression.

> 🏗️ **Architecture & Maintainability Specialist · 🟢 85**
>
> - Code is simple and follows the surrounding handler style.
> - The implementation would benefit from colocated authorization handling matching adjacent endpoints.

> 🧪 **Verification & Test Quality Specialist · 🔴 50**
>
> - No tests added for the new endpoint.
> - No authorization regression test exists for cross-merchant access.

> 🔒 **Security & Privacy Specialist · 🔴 40**
>
> - Missing authorization check is a real data-exposure risk.
> - A merchant could potentially export another merchant's transactions.

> 🗄️ **Data Integrity & Persistence Specialist · 🟢 85**
>
> - Activated because: the endpoint reads durable merchant transaction records and serializes them into an export.
> - The change does not mutate records, so loss, duplication, and rollback risks are low.
> - Export completeness for large or empty transaction sets remains unproven because pagination behavior is not specified.

</details>

<details>
<summary>🔎 <strong>Findings</strong></summary>

<br>

🔴 **Critical - 1 Issue**

> **`GET /merchants/:id/export` doesn't verify the requesting user has access to `:id` — it only checks that the user is authenticated, not that they belong to that merchant.**
>
> - **Location:** `src/routes/exportController.ts` (new file, untracked)
> - **Why it matters:** Any authenticated user can export any other merchant's full transaction history by changing the `:id` in the URL — a direct data-exposure issue.
> - **Fix:** Add the same merchant-ownership check used in `src/routes/transactionsController.ts:getTransactions()` before generating the export.
>
> ```ts
> router.get("/merchants/:id/export", requireAuth, async (req, res) => {
>   const rows = await getTransactions(req.params.id); // no ownership check on :id
>   res.type("text/csv").send(toCsv(rows));
> });
> ```

🔵 **Low - 1 Issue**

> **No tests for the new endpoint.** (No code block — there's nothing to quote for a missing test.)
>
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
````

Note how the missing-untracked-file bug this skill was fixed for shows up
directly in this example: `exportController.ts` is a **new, untracked**
file, and it's still reviewed — including a Critical finding inside it.
