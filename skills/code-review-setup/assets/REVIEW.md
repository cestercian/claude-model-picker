# Review guide

<!--
Owned by code owners (list this file in CODEOWNERS). Keep it short: the reviewer
reads it on every PR. Replace every example with this repo's own.
Code Review reads this file from the base branch, so edits apply to later PRs,
not to the PR that makes them.
When reviewers keep flagging the same thing, add a line here (or a lint rule).
When a line here produces comments people keep dismissing, cut or narrow it.
-->

## What failure looks like here

<!-- One phrase per area. e.g. docs: "it lies"; config: "it misroutes";
billing: "it charges wrong"; updater: "users get stuck on a broken build". -->

## Focus on

- Correctness bugs and regressions that would break production
- Whether the change does what the PR description says it does, and fits the rest of the system
- On bug fixes: whether the root cause is fixed, not just the symptom
- Security issues: auth checks, injection, secrets, unsafe input handling
- Changes that break a convention in CLAUDE.md, or make a line in it out of date
- <!-- repo-specific: e.g. "every migration has a matching rollback" -->

## Always needs a human

Flag clearly when a PR touches these, even if the change looks fine:

- <!-- e.g. auth/, billing/, .github/workflows/, release and signing config, installers, migrations/ -->

## Skip

- Style and formatting (the linter handles these)
- Missing tests, unless a change to risky code has none at all
- Generated, vendored and lock files: <!-- paths -->

## Rules for findings

- Cite a file and line as evidence. Don't guess from a name.
- Tag each finding **Important**, **Nit**, or **Pre-existing** (a bug this PR didn't introduce).
- At most 3 nits per review. No findings that are only about style or cleanup.
- If the same bug appears in several places, name the pattern once and list every location.
- Don't repeat a finding already posted on this PR. On re-review, look only at new commits.
- After three review rounds, stop listing small fixes and ask whether the approach should change.
- Start the summary with a count of findings. If there is nothing to fix, say so plainly.
