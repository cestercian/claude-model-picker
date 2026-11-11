---
name: code-review-setup
description: Set up a layered, agent-era code review process in a repository - verification before the PR, AI and security review on every PR, risk-tiered human approval, and merge and rollback safety - for teams or solo developers directing agents. Use when someone asks to set up or improve code review, add an AI reviewer, write a REVIEW.md or PR template, define risk tiers or CODEOWNERS, or cope with more pull requests than humans can read.
---

# Code review setup

Self-contained; no links need opening. Solo or agent-only repo: also read `references/solo.md` (replaces human-approval tiers, enforces locally with hooks).

## Why review changed

Agents multiply changes: Anthropic's monorepo went from ~2,500 to 30,000+ changes landing per day (May to Sept 2026); in the Claude Code repo (~90% written by Claude) merged PRs more than doubled while the revert share fell 40%+. Line-by-line human review can't scale, so ask not "did a human read this?" but **"what could this break, and which check, human or automated, is best placed to catch it?"** Human time is the scarce resource, counting both review time and cleanup after bad code ships. Target: most PRs merge with at most one human (possibly the author). Guardrail: reverts and incidents must not rise. Measure approvals per PR (median, p95), human comments per PR, time waiting on a person, and revert/incident rate.

## Workflow

1. **Establish the setup first** (ask or probe):
   - **Team or solo?** Solo: use `references/solo.md`.
   - **PRs at all?** Compare `gh pr list --state merged --limit 20` with `git log --oneline -20 main`. No PRs: add them, or move checks to a pre-push hook.
   - **Plan limits.** `gh api repos/{owner}/{repo}/branches/{branch}/protection` (and `/rulesets`). A 403 / "Upgrade to GitHub Pro" on a private repo means no branch protection: nothing can be *required*; use local fallbacks and say so. Code Review needs a Claude Team/Enterprise plan on GitHub; otherwise local `/code-review` + `/security-review`, or the Claude Code GitHub Action in CI.
   - **Which CI?** GitHub Actions, or external (Xcode Cloud, Bitrise, CircleCI...) configured in that system to run on PRs and report status.
   - **Risky paths:** auth, billing/payments, migrations, CI workflows, release/signing, installers/updaters, system prompts, caching, secrets. Feeds CODEOWNERS, REVIEW.md, CLAUDE.md, tiers.
   - **Linter/formatter?** If none, add the standard one before any AI reviewer.
2. **Report gaps:** a table of the seven layers: exists, missing, unenforceable on this plan, proposal. Keep it proportional to repo size.
3. **Add files** the way the repo lands changes: CLAUDE.md "Verify your work" section, `REVIEW.md` (from `assets/REVIEW.md`), `.github/pull_request_template.md` (from `assets/`), `.github/CODEOWNERS`, CI workflow if missing, security review, formatter hooks.
4. **Ask before changing settings**, even with admin rights: branch protection/ruleset (require PR, required checks, approvals, code-owner review, dismiss stale approvals, require conversation resolution, block force push), merge queue, external CI triggers, Code Review enablement, trusted-author list.
5. **Roll out gradually:** AI reviewer on one repo, tune REVIEW.md for a few weeks, then widen. Tiers start strict.

## The seven layers

| # | Layer | Set up | Why |
|---|---|---|---|
| 1 | Verification before the PR | CLAUDE.md lists exact test/build/lint/typecheck/UI-screenshot commands; agent pastes output, not "done"; `PostToolUse` formatter hook on `Edit\|Write`; fresh-context review via `/code-review`, `/security-review` or a subagent given only diff + intent | Self-verification makes results ~2-3x better; a fresh reviewer isn't anchored on the author's reasoning |
| 2 | AI review on every PR | Code Review (Quick / Standard default / Deep per repo by risk), Claude Code GitHub Action, or local `/code-review` | Substantive-comment share on internal PRs rose 16% to 54%; 1,000+ line PRs usually get findings |
| 3 | Blocking security review | `anthropics/claude-code-security-review` Action or `/security-review`; open findings from either review block merge | A general bug hunt misses security issues |
| 4 | Required CI + PR template | Lint, types, tests on all platforms, build as required checks; template with What and why, Testing (evidence), Risk, How to revert | Cheapest, most reliable layer; the template's intent is what the reviewer checks against |
| 5 | Risk-tiered human approval | Tiers below; CODEOWNERS + "require code owner review" | Uniform review wastes senior time on trivia or under-reviews danger |
| 6 | Merge queue, flags, staged release | Merge queue (CI must also trigger on `merge_group`); flags off by default, internal first, switchable without deploy; "latest" channel before "stable" | Two green PRs can combine into a broken main; some bugs only show in real use |
| 7 | Fast detection and rollback | One-step flag-off and revert; on-call; revert/incident dashboards; incident reviews ask "which layer should have caught this?" | Going faster at fixing beats going slower; goal: circuit breakers that tighten approvals when reverts rise |

**CLAUDE.md vs REVIEW.md.** CLAUDE.md: what the project is (stack, conventions, how to verify); read by every session; reviewer flags violations and lines a change makes outdated. REVIEW.md: what the reviewer should care about; owned by code owners (put it in CODEOWNERS). Both short. A good REVIEW.md names what failure looks like here ("it charges wrong"), caps nits (e.g. 3), lists paths to skip (generated, vendored, lockfiles), requires file:line evidence, and starts its summary with a finding count. Fix repeated findings at the source: lint rule, skill, CLAUDE.md or REVIEW.md.

**AI review findings** are tagged Important / Nit / Pre-existing (not this PR's fault). Every finding is fixed or resolved with a reason. Reviewer checks: does it solve the stated problem, fit the system, break anything else; root cause over symptom patches.

**Risk tiers:**

| Tier | Examples | Human approvals |
|---|---|---|
| Low | Docs, copy, flag-gated additive, internal-only | 0, once automated review and security are green |
| Medium | Ordinary product changes; caching, auto-updater, system prompts | 1 |
| High | Highest-risk code, short list of always-two paths | 2 |

- The change sets the tier, not the author; a classifier assigns it, and a second check can escalate (e.g. a "docs" PR that edits a script).
- Some paths always need a person: CI workflows and owner-marked sensitive paths.
- Lighter paths only for authors on a per-repo trusted list.
- A bot's approval never fills a human slot.
- Whoever directed the agent fills one reviewer slot and is accountable.
- Approval covers one commit: dismiss stale approvals on new pushes.
- Start strict; loosen only with revert/incident evidence.
- GitHub implementation: CODEOWNERS for always-human paths (include `.github/workflows/`, CODEOWNERS, REVIEW.md); a CI job labels `risk:low|medium|high`; a required gate check counts non-bot approvals on the head commit and checks the trusted list. No branch protection: put the list in REVIEW.md/CLAUDE.md as "stop and ask" and enforce locally (solo.md).

## Making the reviewer better

- **Incidents as eval:** collect PRs that caused incidents/reverts, have owners confirm the bug, run the reviewer on the set on a schedule (daily in the reference team), rescore on every prompt/REVIEW.md/model change; keep only improvements. Small repo: a handful of past reverts through `/code-review`.
- **Measure noise:** bucket comments by author response: fixed / wrong / intentional / didn't matter. Run on known-clean PRs to count harmful nudges. Priority: real bugs first, fewer rounds second. Noise costs more with agent authors, which fix every nit at the price of another round.
- **Convergence:** one PR ran 50+ overnight review-fix rounds; 59% of what was flagged had existed since the first pass. Fixes: cap at three rounds then rethink; diff-only re-review after the first few; flag the bug class, not one line; never repeat a finding; track rounds-to-pass; nightly sweeps of existing code. Result: one-round pass 65%→78%, 5+ rounds 8.9%→1.5%, mean reviews 2.08→1.32.
- When a review misses something, add what the model lacked to CLAUDE.md, a skill or a check.

## Humans and team norms

- **Humans stay in the loop** for whether to build it, risky paths, pattern-setting architecture, reading the evidence (if you can't say what checks missed, review properly), and owning the outcome.
- **Sharing context:** reading diffs used to spread understanding; replace it with a Claude explainer (or interactive diagram) of the slice you're about to change, and work in public channels.
- **You land it, you own it:** fix what you broke, backed by small reversible changes, fast signals and blameless incident reviews.
- **Change direction early:** raise "should we?" early in the thread; closing cheap agent work isn't a loss.

## Code Review for Claude Code

- Needs a Claude Team or Enterprise plan and GitHub (incl. Enterprise Server); not with ZDR, HIPAA or CMEK. An org Owner installs the GitHub App, picks repos, level and trigger (on open, every push, or on request; `@claude review` from write-access users).
- Levels Quick / Standard / Deep; model not configurable; billed per review with an optional monthly limit (reviews skip once hit).
- Focuses on correctness; reads code, never runs it; skips drafts and forks.
- REVIEW.md is read from the base branch, so a PR can't loosen its own review.
- **It never approves or blocks.** Its check run never fails; blocking comes from requiring resolved conversation threads in branch protection.

## Adoption checklist

- [ ] CLAUDE.md verify commands; formatter/linter hooks; evidence over "done"; fresh-context local review
- [ ] REVIEW.md owned by code owners
- [ ] Required CI; PR template (What and why, Testing, Risk, How to revert)
- [ ] AI + security review on every PR; threads resolved before merge; depth matched to risk; three-round cap
- [ ] Risk tiers, trusted authors, CODEOWNERS for always-human paths, stale approvals dismissed, no bot approvals
- [ ] Merge queue, flags, staged release, one-step rollback
- [ ] Incident eval set, noise and rounds-to-pass tracked, human-time metrics with revert/incident guardrail

## Gotchas

- Code Review doesn't block on its own; needs required conversation resolution and a plan with branch protection.
- REVIEW.md edits don't apply to the PR making them.
- Required checks need branch protection or rulesets; without them, enforce locally and tell the user.
- External CI must be set to run on PRs and report status.
- Merge queue needs CI on `merge_group` or PRs wait forever.
- Agent-reviewer loops don't converge on their own.
- Don't recommend what the plan can't do; don't change shared settings without asking.

## Files in this skill

- `references/solo.md`: solo developer or agent-only repos; enforcing locally with a Claude Code `PreToolUse` hook plus committed git hooks, with full example scripts; limited CI; linting.
- `assets/REVIEW.md`: REVIEW.md template to copy to the repo root and fill in.
- `assets/pull_request_template.md`: PR template to copy to `.github/pull_request_template.md`.

## Based on

Credit only; everything needed is above.

- "How the Claude Code team does code review", Addy Osmani, Morgan Westlee Lunt and Qing Wang, claude.dev Blog (Anthropic), Oct 15, 2026. https://claude.dev/blog/solving-code-review-at-anthropic/
- "Code Review", Claude Code docs (Anthropic). https://code.claude.com/docs/en/code-review
- anthropics/claude-code-action and anthropics/claude-code-security-review on GitHub. https://github.com/anthropics/claude-code-action, https://github.com/anthropics/claude-code-security-review
