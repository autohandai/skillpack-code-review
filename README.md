# Code review skills

Three skills for checking a branch before it merges. A diff reviewer reports findings by severity with file:line, a checklist runs your own build and test commands, and a test gap finder lists changed behaviour that no test checks.
They work in any language, since they only rely on git and the commands your project already has.

## Install

From the root of your repo:

```bash
git clone https://github.com/autohandai/skillpack-code-review
mkdir -p .claude/skills
cp -R skillpack-code-review/skills/* .claude/skills/
```

Copy the skill folders themselves, so you get `.claude/skills/diff-review/SKILL.md` and not a nested pack folder. Some agents only look one level deep. These work well as personal skills too, in `~/.claude/skills/`.

Ask for a review and most agents will pick the right skill. To call one by name, mention it: `$diff-review` in Autohand Code, or `/diff-review` in Claude Code. For example: "$diff-review against develop".

## Skills

| Skill | What it does |
|---|---|
| `diff-review` | Reads `git diff` against the base branch and reports blocker, major, minor and nit findings, each with file:line and a fix |
| `pre-merge-checklist` | Runs build, tests, lint, secret and debug searches, and migration and config checks, then reports pass, fail or not run |
| `test-gap-finder` | Maps each changed branch to the test that would catch it breaking, and suggests the missing ones |

## Adapting it

- Change the base branch default (`main`) if yours is different.
- Add your team's own rules to the "look for these" list in `diff-review`, such as logging conventions or banned APIs.
- Add items from your PR template to `pre-merge-checklist`.
- Adjust the test path patterns in `test-gap-finder` to your layout.

Keep the `name` in the frontmatter the same as the folder name.

If a check gives you noise, please open an issue with an example.

Runs in Autohand Code (free CLI, any model): https://autohand.ai/cli/?utm_source=skillpack-code-review&utm_medium=github

Until 31 October 2026: first month of Pro for $10 with code FOUNDING10 (first-time customers).

MIT licensed. See LICENSE.
