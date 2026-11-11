# Solo and agent-only repos

The source article describes a team. With one person directing agents, there is no second human to approve, so the human-approval layer collapses into "the owner reads the evidence". Keep the rest of the layers: they matter more, not less, because nobody else is looking.

This adaptation is this skill's own, built from the article's layers and the Claude Code docs.

## What changes

| Team layer | Solo version |
|---|---|
| Risk-tiered approvals (0/1/2 people) | Two tiers: **routine** (lands once local checks pass) and **owner reads it** (anything on the risky-path list). |
| CODEOWNERS + branch protection | A risky-path list in REVIEW.md and CLAUDE.md, enforced locally (below). Use branch protection too if the plan includes it. |
| AI review on every PR | If the repo uses PRs, keep it. If changes go straight to main, run a fresh-context `/code-review` and `/security-review` before every push. |
| Required CI on every PR | With push-to-main, CI runs only after code is already on main, so it can tell you but not stop you. Put the fast checks (format, lint, fast tests) before the push; let CI run the full suite on main. If you later use PRs, configure the CI to run on them too. |
| Merge queue | Usually unnecessary with one writer at a time. Needed once several agents push in parallel. |

## Enforcing locally

Git hooks alone are weak gates for agents: an agent can pass `--no-verify`, a pre-push hook can't prompt without `/dev/tty`, and `.git/hooks` isn't committed. Use two layers:

1. **A Claude Code `PreToolUse` hook on Bash**, with `"if": "Bash(git push*)"`, committed in `.claude/settings.json`. It runs the fast checks, and checks whether the pushed commits touch a risky path. If they do, it blocks (exit code 2) with a message telling Claude to stop and ask the owner. This stops agents even when they would skip git hooks.
2. **A committed git pre-push hook** for pushes made outside Claude: put it in a tracked folder such as `.githooks/` and set `git config core.hooksPath .githooks`. Run the same checks; for risky paths, refuse and print what needs the owner, rather than prompting.

## Limited CI compute

When CI minutes are scarce, put the cheap checks in the hooks above and keep CI for the full suite on main, or on PRs touching risky paths.

## Linting

If the language ships a formatter or linter, wire it in before adding an AI reviewer; it removes a class of review comments. For Swift, `swift-format` ships with the toolchain.

## Still keep

- Agents show check output instead of saying "done".
- A REVIEW.md, even for one person: it is what the AI reviewer reads.
- The review-round cap and nit limits from the main skill: an agent author and an agent reviewer loop fastest when nobody else is watching.
- Revert as the rollback, and small changes so reverts stay clean.

## Sources

- [Hooks reference](https://code.claude.com/docs/en/hooks) (`PreToolUse`, the `if` field, exit code 2), Claude Code docs.
