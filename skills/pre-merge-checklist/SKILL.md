---
name: pre-merge-checklist
description: Runs a pre-merge checklist on the current branch, covering a clean build, passing tests and linters, a scoped diff, no secrets or debug code, migrations, config and docs, and a clear description, then reports pass, fail or not applicable for each item with evidence. Use before opening or merging a pull request, when asked whether a branch is ready to merge, or as a final gate after a review.
license: MIT
---

# Pre-merge checklist

This skill checks readiness. It doesn't judge the code. For a line-by-line review, use a diff review. Run each check, record the evidence, and report.

## Step 1: find the project's own commands

Look in this order and use what you find. Don't invent commands.
1. `package.json` scripts, `Makefile`, `justfile`, `Taskfile.yml`, `Rakefile`, `pyproject.toml`, `tox.ini`, `noxfile.py`
2. CI config: `.github/workflows/*.yml`, `.gitlab-ci.yml`, `.circleci/config.yml`. The commands CI runs are the ones that matter.
3. `CONTRIBUTING.md` or the README

Write down the build, test, lint and type check commands before you run anything.

## Step 2: run the checks

```bash
BASE=$(git merge-base HEAD origin/main)    # or the target branch
git status --short                         # uncommitted work?
git diff --stat "$BASE"...HEAD             # scope
git log --oneline "$BASE"..HEAD            # commits
```

| # | Check | How | Pass when |
|---|---|---|---|
| 1 | Branch is current | `git rev-list --count HEAD..origin/main` | 0, or the gap has been merged in and retested |
| 2 | Working tree clean | `git status --short` | Empty, or the user says the leftovers stay out |
| 3 | Builds | The project's build command | Exit 0 |
| 4 | Tests pass | The project's test command | Exit 0, with no newly skipped or focused tests |
| 5 | Lint and types | The project's lint and type check commands | Exit 0, with no new suppressions |
| 6 | Scope | `git diff --stat` | Every file relates to the stated intent |
| 7 | No secrets | Search the diff (below) | No keys, tokens, passwords or private URLs |
| 8 | No debug leftovers | Search the diff (below) | No stray prints, `debugger`, focused tests or TODOs without an issue |
| 9 | Migrations | Look for new migration files | Reversible, and safe to run against live traffic |
| 10 | Config and env | New env vars or config keys | Documented, with defaults or deploy notes |
| 11 | Dependencies | Lockfile changes | Intended, and the lockfile matches the manifest |
| 12 | Docs and changelog | README, docs, CHANGELOG | Updated if behaviour or the interface changed |
| 13 | Description | PR text or commit messages | States what, why and how it was tested |

Searches for checks 7 and 8 run on added lines only:

```bash
git diff "$BASE"...HEAD | grep -nE '^\+' | grep -niE \
  '(api[_-]?key|secret|passw(or)?d|token|BEGIN (RSA|OPENSSH|EC) PRIVATE KEY|AKIA[0-9A-Z]{16})'

git diff "$BASE"...HEAD | grep -nE '^\+' | grep -nE \
  '(console\.log|debugger|binding\.pry|byebug|pdb\.set_trace|breakpoint\(\)|fit\(|fdescribe\(|\.only\(|test\.skip|xit\()'
```

The grep line numbers point into the diff, not the file. Open each hit and report the real file:line. Many hits are false positives, such as the word "token" in a variable name. Check each one before you report it.

If a command needs services you don't have (a database, network), mark the check "not run", say why, and give the command for the user to run. Never mark a check as passed without running it.

## Step 3: report

```
Pre-merge: feature/webhook-retries -> main

PASS  1 Branch current (0 commits behind)
PASS  3 Build: `npm run build` exit 0
FAIL  4 Tests: `npm test` 2 failing in src/webhooks/send.test.ts
PASS  5 Lint and types: `npm run lint`, `npm run typecheck` exit 0
FAIL  8 Debug leftovers: src/webhooks/send.ts:61 console.log(payload)
N/A   9 No migrations
NOT RUN 11 Lockfile check: no network
...
Ready to merge: no. Fix 4 and 8.
```

Keep it to one line per check. Put detail only under failures.

## Rules

- Run commands. Don't infer results from reading code.
- Report exit codes and the command used.
- A check you couldn't run is "not run", never "pass".
- Don't fix things during the checklist unless asked. List them.
- If the project has its own PR template or checklist, use its items as well and say which list you followed.
