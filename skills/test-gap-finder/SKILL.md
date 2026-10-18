---
name: test-gap-finder
description: Finds behaviour changed on the current branch that no test exercises, by mapping each changed function and branch in git diff to the tests that cover it, then lists the gaps by risk with file:line and a concrete test case to add. Use when asked what tests are missing, before merging a change to risky logic, or after a review flags weak coverage.
license: MIT
---

# Test gap finder

Line coverage shows which lines ran. It can't show whether any test checked the result. This skill asks, for each behaviour this branch adds or changes, is there a test that would fail if it broke?

## Step 1: list the changed behaviour

```bash
BASE=$(git merge-base HEAD origin/main)            # or the target branch
git diff --name-only "$BASE"...HEAD                # changed files
git diff -U0 "$BASE"...HEAD -- . ':!*test*' ':!*spec*'   # source hunks only
```

Adjust the exclude patterns to the project's test layout (`__tests__/`, `tests/`, `spec/`, `*_test.go`).

For each source hunk, write down:
- the function or method it sits in, with file:line
- each new or changed branch (`if`, `switch` case, early return, `catch`, guard clause)
- each new input it accepts, and each new error it can raise or return
- each side effect (write, network call, event, job)

## Step 2: find the tests that touch each one

1. Look for a test file that mirrors the source path (`src/a/b.ts` and `src/a/b.test.ts`, `app/models/user.rb` and `spec/models/user_spec.rb`, `pkg/x.go` and `pkg/x_test.go`).
2. Search tests for the function name: `git grep -n "functionName" -- '*test*' '*spec*'`.
3. Search for higher-level tests that reach it: the route, command, or public method that calls it.
4. Read the matching tests. A test only counts if it would fail when the branch breaks. A test that calls the function but asserts nothing about this branch doesn't count.

If the project has a coverage tool already configured, run it on the changed files to confirm. Use the project's command (for example `npx vitest run --coverage`, `pytest --cov`, `go test -cover ./...`, or `COVERAGE=true bundle exec rspec` with SimpleCov). Treat its result as a hint, since a covered line can still be unchecked.

## Step 3: rank the gaps

| Risk | Examples |
|---|---|
| High | Auth and permission checks, money, data deletion, migrations, input validation at a public boundary, error handling that decides retry or give up |
| Medium | Business rules with several branches, date and time logic, parsing, caching and invalidation |
| Low | Formatting, logging, simple getters, wiring code the type checker covers |

Tests for a high-risk gap are worth writing before merge. Low-risk gaps can be listed and left.

## Step 4: report

For each gap, give the location, the untested behaviour, and the test to add, phrased as a test name with its setup and assertion.

```
Test gaps: feature/refunds -> main (6 changed functions, 4 gaps)

High
1. app/services/refunds/issue.rb:34 Refund above the original charge is rejected; no test.
   Add: "rejects a refund larger than the charge" with a 100 charge and a 150 refund. Assert a failure result and no Refund row.
2. app/controllers/refunds_controller.rb:12 Non-admin access returns 403; only the admin path is tested.
   Add: a request spec as a regular user. Assert 403 and no change.

Medium
3. app/services/refunds/issue.rb:51 Partial refund rounds to cents; tested only with whole numbers.
   Add: refund 10.005 and assert 10.01 (or the rule the code intends; confirm with the author).

Low
4. app/services/refunds/issue.rb:70 Log line on success. Skip.

Covered: Refunds::Issue#call happy path (spec/services/refunds/issue_spec.rb:8).
```

## Rules

- Tie every gap to a changed line. Don't audit the whole codebase.
- Name existing tests that do cover a behaviour, so the user can trust the "covered" list.
- If the intended behaviour is unclear (as in the rounding example), say so and ask, rather than writing a test that locks in a guess.
- Only write the tests if the user asks. If they do, write each one so it fails on the current code first when it covers a bug, or when you can break the code to prove it.
