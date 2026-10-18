---
name: diff-review
description: Reviews the changes on the current branch by reading git diff against the base branch, then reports concrete findings grouped by severity (blocker, major, minor, nit) with file:line references and a suggested fix for each. Use when asked to review a branch, a pull request, staged changes or work in progress before merge, or when someone asks what is wrong with a diff.
license: MIT
---

# Diff review

Read the diff, understand what it changes, and report only problems you can point to. A short list of real findings beats a long list of opinions.

## Step 1: get the right diff

Find the base branch. Use the one the user names. Otherwise try `main`, then `master`, then `develop`.

```bash
git fetch origin --quiet                      # skip if offline
BASE=$(git merge-base HEAD origin/main)       # or the named base
git diff --stat "$BASE"...HEAD                 # what changed, how much
git diff "$BASE"...HEAD                        # committed changes on this branch
git diff HEAD                                  # plus uncommitted work, if asked
git log --oneline "$BASE"..HEAD                # the intent, from commit messages
```

For staged changes only, use `git diff --cached`. For a single commit, use `git show <sha>`.

If the diff is over about 1,500 changed lines, say so. Review the riskiest files first (auth, payments, migrations, public APIs, concurrency), and list what you didn't cover.

## Step 2: understand before judging

1. Read the commit messages and the PR description, if there is one. State the intent of the change in one sentence.
2. For each changed file, open the whole file as well as the hunk. Most real bugs sit between the changed lines and the code around them.
3. For each changed function, search for its callers (`git grep -n "functionName"`). A changed signature or return value breaks callers outside the diff.
4. Note what the project's own tooling enforces (linters, formatters, type checkers). Don't report what they already catch.

## Step 3: look for these, in order

1. Correctness. Wrong condition, off by one, null or undefined paths, unhandled errors, wrong async handling (missing `await`, unhandled promise), state mutated in the wrong place, broken edge cases (empty list, zero, unicode, time zones).
2. Security. Missing auth or permission checks, injection (SQL, shell, HTML), secrets in code or logs, unsafe deserialization, user input reaching file paths or URLs, weakened CORS or CSRF settings.
3. Data and migrations. Destructive or locking schema changes, backfills without batches, data loss on rollback, changes not backwards compatible with code still running.
4. Contracts. Changed public APIs, response shapes, events or config keys without a version bump or migration path.
5. Concurrency and performance. Races, N+1 queries, unbounded loops or queries, work moved onto a hot path, missing indexes for new queries.
6. Tests. New behaviour with no test, tests that can't fail, deleted assertions.
7. Maintainability. Only when it will cause a bug or real confusion: duplicated logic that must stay in sync, misleading names, dead code left behind.

## Severity

| Level | Meaning | Merge? |
|---|---|---|
| Blocker | Will break production, lose data or open a security hole | No |
| Major | A likely bug, or a missing test for risky logic | Fix first, or get agreement |
| Minor | A real but low-impact issue | Can follow up |
| Nit | Style or naming the tooling doesn't catch | Optional |

When unsure between two levels, pick the lower one and say why.

## Step 4: report

Use this format. Cite the line in the new version of the file. Get line numbers from the file itself, or from the `+` side of the hunk header (`@@ -a,b +c,d @@` means new lines start at `c`).

```
Intent: Adds retry with backoff to the webhook sender.

Blocker
1. src/webhooks/send.ts:42 Retries also run for 4xx responses, so a bad payload is sent 5 times to the customer endpoint.
   Fix: retry only on network errors and 5xx: `if (res.status < 500) return res`.

Major
2. src/webhooks/send.ts:58 `await sleep(delay)` uses `2 ** attempt * 1000` with no cap. Attempt 10 waits 17 minutes inside a request handler.
   Fix: cap the delay (`Math.min(..., 30_000)`) or move retries to the job queue.
3. src/webhooks/send.test.ts No test covers the 4xx path.
   Fix: add a test asserting one call for a 400 response.

Minor
4. src/webhooks/config.ts:12 `MAX_RETRIES` is read from env without a default, so it's `NaN` when unset.

Not reviewed: generated files under src/gen/.
Verdict: changes requested (1 blocker).
```

## Rules

- Every finding has a file:line (or a file, for a missing test), what goes wrong, and a fix. If you can't say what goes wrong, drop it.
- Quote the code you mean when the line alone is ambiguous.
- Don't repeat the same finding for every occurrence. List it once and name the other locations.
- Don't praise, and don't summarise the diff back line by line.
- If you find nothing above nit level, say "No blocking issues found", list what you checked, and stop.
- Don't edit files during a review unless asked.
