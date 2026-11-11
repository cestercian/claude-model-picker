---
name: code-review-setup
description: Set up a layered, agent-era code review process in a repository - verification before the PR, AI and security review on every PR, risk-tiered human approval, and merge and rollback safety - for teams or solo developers directing agents. Use when someone asks to set up or improve code review, add an AI reviewer, write a REVIEW.md or PR template, define risk tiers or CODEOWNERS, or cope with more pull requests than humans can read.
---

# Code review setup

Goal: spend human attention only where humans are needed, and let automated layers catch the rest, without raising revert or incident rates. Replace "did a human read it?" with "what could go wrong with this change, and which check, automated or human, should catch it?"

Each layer catches some problems and misses others. Set up several, not one gate.

## Workflow

1. **Work out the setup before recommending anything.** Each answer changes which layers apply:
   - **Who reviews?** A team, or one person directing agents? For solo or agent-only repos, use [references/solo.md](references/solo.md) instead of the human-approval tiers.
   - **Does the repo use PRs at all?** If changes go straight to main, the "On every PR" layer needs PRs first, or its checks move to a pre-push hook.
   - **What can the plan enforce?** Probe branch protection (`gh api repos/{owner}/{repo}/branches/{branch}/protection`). A 403 on a private repo usually means the plan doesn't include it: say so and use the fallbacks below rather than recommending settings that can't be turned on. Code Review for Claude Code needs a Team or Enterprise plan. Without it, run `/code-review` and `/security-review` locally; the Claude Code GitHub Action is an option only if the repo uses PRs and has Actions minutes to spare.
   - **Which CI?** GitHub Actions, or something configured outside the repo (Xcode Cloud start conditions live in App Store Connect; Bitrise, Jenkins…). "Required CI" means that system runs on every PR and reports a status the merge waits on. Making a status *required* needs branch protection, so on plans without it CI can report but not block.
2. **Report the gaps** against the layers below, before changing anything.
3. **Add files in the repo** the way the repo already lands changes: through a PR if it uses them, otherwise as a normal commit the user reviews. Add the items under "Before the PR" and "On every PR".
4. **List repo or CI settings the user must change** (branch protection, required checks, CI start conditions, merge queue, Code Review depth). These are shared settings, so ask before applying any.

## Layers

**Before the PR**
- CLAUDE.md lists the commands an agent runs to verify its own work (tests, build, lint). Keep it short; point to other docs.
- Hooks run formatters and linters. If the repo has no linter, add the language's standard one first; it removes a whole class of review comments.
- Agents show the output of those checks rather than claiming they're done, then a fresh-context subagent reviews the diff (`/code-review`, `/security-review`).
- `REVIEW.md`, owned by code owners, says what the reviewer should care about. Start from [assets/REVIEW.md](assets/REVIEW.md). When reviewers keep flagging the same thing, fix it at the source: REVIEW.md, CLAUDE.md, a skill, or a lint rule.

**On every PR**
- Required CI: lint, types, tests, build. Red blocks merge.
- PR template with Testing, Risk, and How to revert. Use [assets/pull_request_template.md](assets/pull_request_template.md). An agent can fill it; the person who asked for the change signs off.
- AI review: Code Review for Claude Code (Team/Enterprise; Quick, Standard or Deep per repo) or the Claude Code GitHub Action. Start on one repo, tune REVIEW.md for a few weeks, then widen.
- Automated security review (`/security-review` or the security review GitHub Action).
- Every review thread must be fixed or resolved with a reason before merge.

**Approval scaled by risk**

| Tier | Examples | Human approvals |
|---|---|---|
| Low | Docs, copy, internal-only, additive changes behind a flag | 0 once automated reviews are green |
| Medium | Ordinary product changes; sensitive areas like caching, updaters, system prompts | 1 |
| High | Highest-risk code, plus listed paths | 2 |

- The change sets the tier, not the author. Start strict and loosen with evidence.
- Lighter paths only for trusted authors (usually the owning team).
- CODEOWNERS plus branch protection so auth, billing, CI workflows, release, signing, and installers always need a person. Without branch protection, list these paths in REVIEW.md and CLAUDE.md as "always ask a human" and enforce it locally (see [references/solo.md](references/solo.md#enforcing-locally)).
- A bot's approval never fills a required human slot. The person directing an agent counts as a reviewer of its PR.
- Dismiss approvals when new unvetted commits land.

**Merge and after**
- Merge queue (re-runs checks against latest main), feature flags, internal use first, staged release, one-step rollback.
- Turn incidents and reverts into an eval set for the reviewer; rescore whenever its prompt, tools or model change.
- Measure human time per PR (approvals, comments, wait time) with revert and incident rates as the guardrail.

## Gotchas

- **Review loops don't converge on their own.** Agent author plus agent reviewer can loop for dozens of rounds, mostly re-finding old code. Cap rounds (after 3, ask the author to rethink the approach), re-review only the new diff after the first passes, flag the class of bug so every instance gets fixed, never repeat a posted finding.
- **Noise costs more with agent authors.** A person waves off a nit; an agent fixes it and triggers another round. Drop style-only findings and cap nits.
- **Measure reviewer noise**, not just recall: track whether each comment was fixed, called wrong, intentional, or unimportant, and run the reviewer on known-clean PRs.
- **Humans still own**: whether to build it, risky paths, pattern-setting architecture, reading the evidence, and the outcome.

## Sources

- Addy Osmani, Morgan Westlee Lunt and Qing Wang, ["How the Claude Code team does code review"](https://claude.dev/blog/solving-code-review-at-anthropic/), claude.dev Blog (Anthropic), Oct 15, 2026.
- [Code Review](https://code.claude.com/docs/en/code-review), [Security](https://code.claude.com/docs/en/security) and [Best practices](https://code.claude.com/docs/en/best-practices), Claude Code docs.
- [anthropics/claude-code-action](https://github.com/anthropics/claude-code-action) and [anthropics/claude-code-security-review](https://github.com/anthropics/claude-code-security-review).
