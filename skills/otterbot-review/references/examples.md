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
## 🦦 OtterBot · Code Review

Solid, focused change that adds a token-bucket limiter in front of the
webhook endpoint. Mergeable after the missing-lock issue below is fixed —
under concurrent requests the limiter can currently be bypassed.

### 📋 Requirements

Ticket PAY-881 asks for "no more than 50 requests/minute per merchant." The
implementation matches that at the code level, but see the High finding
below — under load, the actual enforced rate can exceed the limit.

### 📊 Scorecard

| Category            | Score      |
| ------------------- | ---------: |
| Correctness         | 65         |
| Completeness        | 85         |
| Regression Risk     | 80         |
| Code Quality        | 90         |
| Testing             | 60         |
| Security            | 95         |

**Overall Score** · 79 / 100

**Score Notes**

- **Correctness:** Rate limiter has a race condition under concurrent requests.
- **Completeness:** Covers the main path; no handling for the limiter's Redis dependency being unavailable.
- **Testing:** Unit tests cover the happy path only; no concurrency or Redis-down test.

### ⚠️ Recommendation · Request Changes

The race condition means the rate limit isn't reliably enforced, which
defeats the ticket's purpose — worth a fix before merge, not a fast-follow.

**Blocking**

- Make the read-check-increment in `checkRateLimit()` atomic so concurrent requests can't all pass the limit. (High)
- Add a Redis-down fallback so a connection blip doesn't 500 all webhook traffic. (Medium)

### 🔎 Findings

#### 🟠 High · 1 Issue

> **The read-check-increment sequence in `checkRateLimit()` isn't atomic, so concurrent requests from the same merchant can all read the same counter value and all pass the check.**
>
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
>
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
>
> - **Location:** `src/webhooks/rateLimiter.ts:23`
> - **Why it matters:** Makes it easy to miss when tuning the limit later, and inconsistent with how other limits are configured elsewhere in the repo.
> - **Fix:** Pull from the existing `config/limits.ts`, matching the pattern used for the API rate limiter.
>
> ```ts
> const LIMIT = 50;
> ```

### 🧪 Testing

> **Existing coverage.**
>
> - `rateLimiter.test.ts` covers a single request under the limit and a single request over it.
> - No coverage for concurrent requests hitting the same merchant key.

> **Coverage gaps.**
>
> - Concurrency: the race condition in the High finding above has no test guarding it — this is the most important gap to close.
> - Redis-down behavior: no test simulates a Redis connection failure to verify the handler's response.

> **Recommended manual verification.**
>
> - Load-test the endpoint with concurrent requests from a single merchant to confirm the limiter holds once the race is fixed.
> - Manually break the Redis connection in staging to observe current failure behavior before deciding on a fallback.
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
## 🦦 OtterBot · Code Review

Small, well-scoped addition. One real gap: the new export endpoint has no
authorization check, so any authenticated user could export any merchant's
transactions.

### 📋 Requirements

No ticket was linked; based on the branch name (`csv-export`) and the diff,
this appears to implement a merchant-facing "export transactions" feature.
Can't confirm acceptance criteria without a ticket — worth double-checking
against whatever spec exists before merging.

### 📊 Scorecard

| Category            | Score      |
| ------------------- | ---------: |
| Correctness         | 90         |
| Completeness        | 70         |
| Regression Risk     | 90         |
| Code Quality        | 85         |
| Testing             | 50         |
| Security            | 40         |

**Overall Score** · 71 / 100

**Score Notes**

- **Completeness:** Missing authorization check (see Critical finding).
- **Testing:** No tests added for the new endpoint (new file, untracked).
- **Security:** Missing authorization check is a real data-exposure risk.

### ⚠️ Recommendation · Request Changes

The missing authorization check is a data-exposure issue and should block
merge on its own; everything else here is minor.

**Blocking**

- Add a merchant-ownership check to `GET /merchants/:id/export` before generating the export. (Critical)

### 🔎 Findings

#### 🔴 Critical · 1 Issue

> **`GET /merchants/:id/export` doesn't verify the requesting user has access to `:id` — it only checks that the user is authenticated, not that they belong to that merchant.**
>
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
>
> - **Location:** `src/routes/exportController.ts` (new file, untracked)
> - **Why it matters:** Nothing currently guards against a regression here, and the missing-auth issue above is exactly the kind of thing a test would have caught.
> - **Fix:** Add a test asserting a user from merchant A gets a 403 when requesting merchant B's export.

### 🧪 Testing

> **Existing coverage.**
>
> - No automated tests exist for the new export endpoint (`exportController.ts` is untracked with no accompanying test file).
> - Existing CSV-format tests elsewhere in the repo (e.g. `transactionsController.test.ts`) don't cover this new route.

> **Coverage gaps.**
>
> - Authorization: no test exercises the missing ownership check from the Critical finding above — a test here would have caught it directly.
> - Edge cases: no test for a merchant with zero transactions or an invalid `:id`.

> **Recommended manual verification.**
>
> - Manually hit the endpoint as a user from merchant A requesting merchant B's `:id` to confirm the fix actually blocks it before merge.
```

Note how the missing-untracked-file bug this skill was fixed for shows up
directly in this example: `exportController.ts` is a **new, untracked**
file, and it's still reviewed — including a Critical finding inside it.
