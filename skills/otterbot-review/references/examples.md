# Worked examples

Two complete, end-to-end examples of this skill's output — one per mode.
These are illustrative (the PR/repo don't exist); they show the level of
specificity and the exact formatting expected, not just the shape.

## Example 1: PR review mode

**User:** "review https://github.com/acme/payhub/pull/142"

**What happens:** a PR URL is present, so this is PR review mode (§1). The
PR's title, description, and diff are fetched from the host. The reviewer runs
or simulates the seven-category expert council, reconciles the internal notes,
and publishes only the final sanitized report. The report below is generated,
submitted as a formal review on PR #142, and also shown in the conversation
(§7).

**Report produced:**

```markdown
## 🦦 OtterBot · Code Review

Solid, focused change that adds a token-bucket limiter in front of the
webhook endpoint. Mergeable after the missing-lock issue below is fixed —
under concurrent requests the limiter can currently be bypassed.

<details open>
<summary><h3>⚠️ Recommendation · Request Changes</h3></summary>

The race condition means the rate limit isn't reliably enforced, which
defeats the ticket's purpose — worth a fix before merge, not a fast-follow.

#### 📋 Feedback · Requirements Specialist

- Concurrent requests can exceed the stated 50 requests/minute merchant limit, so the implementation does not satisfy the core ticket requirement. (High)

#### 🎯 Feedback · Correctness Specialist

- Concurrent requests can bypass the intended merchant limit, which maps to the atomicity finding. (High)

#### 🧪 Feedback · Testing Specialist

- Missing concurrency coverage maps to the regression-test gap for the same failure mode. (Medium)

</details><details>
<summary><h3>🦦 The Otter Council</h3></summary>

> 📋 **Requirements Specialist**
>
> **Score ·** 🟡 70
>
> - Ticket PAY-881 asks for no more than 50 requests/minute per merchant.
> - The implementation aims at the right requirement, but concurrent traffic can exceed the intended limit.

> 🎯 **Correctness Specialist**
>
> **Score ·** 🟡 65
>
> - Rate limiter has a race condition under concurrent requests.
> - The issue directly affects the core requirement because concurrent traffic can exceed the intended merchant limit.

> 🧩 **Completeness Specialist**
>
> **Score ·** 🟢 85
>
> - Covers the main rate-limiting path.
> - Redis-unavailable behavior is not defined, so the operational failure mode still needs a decision.

> 🛡️ **Regression Risk Specialist**
>
> **Score ·** 🟡 80
>
> - Limited blast radius because the change is scoped to webhook rate limiting.
> - Redis dependency behavior should be resolved before merge to avoid surprising webhook failures.

> 🧹 **Code Quality Specialist**
>
> **Score ·** 🟢 90
>
> - Implementation is readable and follows local structure.
> - The remaining concerns are behavioral rather than structural.

> 🧪 **Testing Specialist**
>
> **Score ·** 🟡 60
>
> - Unit tests cover the happy path only.
> - No concurrency test exercises the race condition.
> - No Redis-down test documents the intended fallback behavior.

> 🔒 **Security Specialist**
>
> **Score ·** 🟢 95
>
> - No authorization or sensitive-data concerns found in this change.
> - The limiter does not appear to expose secrets or user data.

</details><details>
<summary><h3>🔎 Findings</h3></summary>

#### 🟠 High · 1 Issue

> **The read-check-increment sequence in `checkRateLimit()` isn't atomic, so concurrent requests from the same merchant can all read the same counter value and all pass the check.**
> - **Location:** `src/webhooks/rateLimiter.ts`, `checkRateLimit()`
> - **Why it matters:** Under real traffic (the exact scenario this change exists to handle), the limiter can be bypassed, silently defeating the ticket's requirement.
> - **Fix:** Use an atomic Redis operation (`INCR` + `EXPIRE`, or a Lua script) instead of separate GET/SET calls.
>
> ```ts
> const count = Number(await redis.get(key)) || 0;
> if (count >= LIMIT) return false;
> await redis.set(key, count + 1, 'EX', 60); // gap between get and set
> return true;
> ```

#### 🟡 Medium · 1 Issue

> **No fallback behavior when Redis is unreachable — the handler currently throws, which would 500 all webhook traffic.**
> - **Location:** `src/webhooks/rateLimiter.ts`, `checkRateLimit()`
> - **Why it matters:** A transient Redis blip would take down payment webhook processing entirely, which is a worse outcome than a temporarily unenforced rate limit.
> - **Fix:** Fail open (allow the request, log a warning) on Redis errors, with an alert so the team notices.
>
> ```ts
> const allowed = await checkRateLimit(merchantId); // throws if Redis is down
> if (!allowed) return res.status(429).end();
> ```

#### 🔵 Low · 1 Issue

> **Magic number `50` for the rate limit is inlined directly in the handler.**
> - **Location:** `src/webhooks/rateLimiter.ts:23`
> - **Why it matters:** Makes it easy to miss when tuning the limit later, and inconsistent with how other limits are configured elsewhere in the repo.
> - **Fix:** Pull from the existing `config/limits.ts`, matching the pattern used for the API rate limiter.
>
> ```ts
> const LIMIT = 50;
> ```

</details><details>
<summary><h3>🧪 Testing</h3></summary>

> **Existing coverage.**
> - `rateLimiter.test.ts` covers a single request under the limit and a single request over it.
> - No coverage for concurrent requests hitting the same merchant key.

> **Coverage gaps.**
> - Concurrency: the race condition in the High finding above has no test guarding it — this is the most important gap to close.
> - Redis-down behavior: no test simulates a Redis connection failure to verify the handler's response.

> **Recommended manual verification.**
> - Load-test the endpoint with concurrent requests from a single merchant to confirm the limiter holds once the race is fixed.
> - Manually break the Redis connection in staging to observe current failure behavior before deciding on a fallback.

</details>
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
## 🦦 OtterBot · Code Review

Small, well-scoped addition. One real gap: the new export endpoint has no
authorization check, so any authenticated user could export any merchant's
transactions.

<details open>
<summary><h3>⚠️ Recommendation · Request Changes</h3></summary>

The missing authorization check is a data-exposure issue and should block
merge on its own; everything else here is minor.

#### 🔒 Feedback · Security Specialist

- Merchants could potentially export another merchant's transactions, which maps directly to the Critical data-exposure finding. (Critical)

#### 📋 Feedback · Requirements Specialist

- A merchant-facing export feature cannot meet the implied data-access requirement without merchant ownership checks. (Critical)

#### 🧩 Feedback · Completeness Specialist

- The missing authorization check blocks a complete implementation and supports the same merge gate. (Critical)

</details><details>
<summary><h3>🦦 The Otter Council</h3></summary>

> 📋 **Requirements Specialist**
>
> **Score ·** 🟡 65
>
> - No ticket was linked, so acceptance criteria are inferred from the branch name and diff.
> - A merchant export feature implies merchant-scoped data access, which the implementation does not enforce.

> 🎯 **Correctness Specialist**
>
> **Score ·** 🟢 90
>
> - Export logic is straightforward and appears to return the intended data.
> - No obvious data-shaping or pagination bug is visible from the changed code.

> 🧩 **Completeness Specialist**
>
> **Score ·** 🟡 70
>
> - Missing authorization check blocks a complete implementation.
> - Acceptance criteria are unclear without a linked ticket or spec.

> 🛡️ **Regression Risk Specialist**
>
> **Score ·** 🟢 90
>
> - New endpoint is isolated from existing flows.
> - The security flaw affects exposed data even though it is not a broad regression.

> 🧹 **Code Quality Specialist**
>
> **Score ·** 🟢 85
>
> - Code is simple and follows the surrounding handler style.
> - The implementation would benefit from colocated authorization handling matching adjacent endpoints.

> 🧪 **Testing Specialist**
>
> **Score ·** 🔴 50
>
> - No tests added for the new endpoint.
> - No authorization regression test exists for cross-merchant access.

> 🔒 **Security Specialist**
>
> **Score ·** 🔴 40
>
> - Missing authorization check is a real data-exposure risk.
> - A merchant could potentially export another merchant's transactions.

</details><details>
<summary><h3>🔎 Findings</h3></summary>

#### 🔴 Critical · 1 Issue

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

#### 🔵 Low · 1 Issue

> **No tests for the new endpoint.** (No code block — there's nothing to quote for a missing test.)
> - **Location:** `src/routes/exportController.ts` (new file, untracked)
> - **Why it matters:** Nothing currently guards against a regression here, and the missing-auth issue above is exactly the kind of thing a test would have caught.
> - **Fix:** Add a test asserting a user from merchant A gets a 403 when requesting merchant B's export.

</details><details>
<summary><h3>🧪 Testing</h3></summary>

> **Existing coverage.**
> - No automated tests exist for the new export endpoint (`exportController.ts` is untracked with no accompanying test file).
> - Existing CSV-format tests elsewhere in the repo (e.g. `transactionsController.test.ts`) don't cover this new route.

> **Coverage gaps.**
> - Authorization: no test exercises the missing ownership check from the Critical finding above — a test here would have caught it directly.
> - Edge cases: no test for a merchant with zero transactions or an invalid `:id`.

> **Recommended manual verification.**
> - Manually hit the endpoint as a user from merchant A requesting merchant B's `:id` to confirm the fix actually blocks it before merge.

</details>
```

Note how the missing-untracked-file bug this skill was fixed for shows up
directly in this example: `exportController.ts` is a **new, untracked**
file, and it's still reviewed — including a Critical finding inside it.
