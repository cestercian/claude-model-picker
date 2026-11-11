---
name: code-review-setup
description: Set up a layered, agent-era code review process in a repository - verification before the PR, AI and security review on every PR, risk-tiered human approval, and merge and rollback safety - for teams or solo developers directing agents. Use when someone asks to set up or improve code review, add an AI reviewer, write a REVIEW.md or PR template, define risk tiers or CODEOWNERS, or cope with more pull requests than humans can read.
---

# Code review setup

Everything needed to set this up is in this skill. You do not need to open any link.

**Goal:** spend human attention only where a human is actually needed, and let automated layers catch everything else, without raising the revert or incident rate.

**The reframe to carry through every decision:** stop asking "did a human read this code?" and ask "what could this change break, and which check, human or automated, is best placed to catch that?"

**The shape:** several overlapping layers, not one gate. Each layer catches some problems and misses others; few problems get through all of them, and fast rollback handles the ones that do.

Contents:

1. Why review has to change (the reasoning to explain to the user)
2. Workflow (the procedure to follow in a repo)
3. The life of a change (the whole pipeline at a glance)
4. The seven layers, each with what to set up and why
5. Making the AI reviewer better over time (evals, noise, convergence)
6. Where humans stay in the loop
7. Sharing context when machines do the reading
8. Team norms: "you land it, you own it", and changing direction early
9. Code Review for Claude Code: product reference for configuring it
10. Adoption checklist
11. Gotchas
12. Files in this skill, and Based on

For a solo developer or an agent-only repo, also read `references/solo.md` in this skill folder; it replaces the human-approval tiers and shows how to enforce the rules locally with hooks.

---

## 1. Why review has to change

Use this section to explain the setup to the user, and to judge trade-offs when the repo doesn't match the default case.

**Volume.** Agents multiply the number of changes. The reference case is the team that builds Claude Code, where Claude writes roughly 90% of the code:

- Across Anthropic's monorepo, changes landing per day rose from about 2,500 in May to more than 30,000 in September (2026).
- In the Claude Code repo itself, merged PRs more than doubled from June to September, while the share of merged PRs that later had to be reverted fell by more than 40%.

They did not get there by removing review. They moved most of the reading to machines, organised review around human time, and made mistakes fast and cheap to undo. Line-by-line human review of everything is not possible at that volume.

**Review has two jobs:**

1. Find and fix problems before they ship.
2. Spread shared understanding of the codebase across the team.

Most of this skill is about job 1. Job 2 needs a deliberate replacement once humans stop reading most diffs; see section 7.

**Review becomes continuous.** It used to be a gate at the end: write code, open a PR, wait for a colleague. Now it runs while the work happens, like a spellchecker. Most changes have been reviewed and fixed by Claude at least once before any human is asked to look. That clears routine mistakes and leaves people free for higher-level questions.

**Human time is the scarce resource.** Count both halves of review's cost: the time people spend reviewing, and the time they spend cleaning up after bad code ships. Cutting the first is worthless if it reappears as the second.

The target used by the reference team: most PRs merge with **at most one human in the loop (who may be the author)**, with **no rise in reverts or incidents**.

How to measure it (set this up as part of the rollout, see layer 7 and section 5):

| Metric | What it tells you |
|---|---|
| Human approvals per PR, median and 95th percentile | How much approval effort each change costs |
| Human review comments per PR | How much reading and discussion people do |
| Time a PR spends waiting on a person | Where humans are the bottleneck |
| Revert rate and incident rate | **Guardrail.** If these rise, you saved no human time; you moved it to cleanup |

---

## 2. Workflow

Follow this in order when asked to set up or improve review in a repo.

### Step 1. Work out the setup before recommending anything

Each answer changes which layers apply. Ask the user, or find out from the repo, before proposing changes.

- **Who reviews: a team, or one person directing agents?** For solo or agent-only repos, use `references/solo.md` instead of the human-approval tiers in layer 5. The other layers still apply and matter more, because nobody else is looking.
- **Does the repo use PRs at all?** Check `gh pr list --state merged --limit 20` against `git log --oneline -20 main`. If changes go straight to main, the "on every PR" layers either need PRs first, or their checks move to a pre-push hook (see `references/solo.md`).
- **What can the plan enforce?** Probe branch protection: `gh api repos/{owner}/{repo}/branches/{branch}/protection` (and `gh api repos/{owner}/{repo}/rulesets`). A 403 or "Upgrade to GitHub Pro" style error on a private repo usually means the GitHub plan doesn't include protected branches. Say so plainly and use the local fallbacks rather than recommending settings that cannot be turned on.
- **Which Claude plan?** Code Review for Claude Code (the managed PR reviewer) needs a Claude **Team or Enterprise** plan and GitHub hosting, and has other exclusions (section 9). Without it: run `/code-review` and `/security-review` locally before pushing (any plan), or run Claude in your own CI with the Claude Code GitHub Action (needs PRs, an API key or cloud credentials, and Actions minutes to spare).
- **Which CI?** GitHub Actions, or a system configured outside the repo (Xcode Cloud start conditions live in App Store Connect; Bitrise, CircleCI, Jenkins, Buildkite have their own settings). "Required CI" means that system runs on every PR and reports a commit status or check that the merge waits on. Making a status *required* needs branch protection or rulesets, so on plans without them CI can report but cannot block.
- **What is risky here?** Ask which paths would hurt most if they broke: auth, billing, payments, data migrations, CI workflow files, release and signing config, installers and auto-updaters, system prompts, caching, anything security-sensitive. This list feeds CODEOWNERS, REVIEW.md, CLAUDE.md and the risk tiers.
- **Is there a linter and formatter?** If not, adding the language's standard one comes before an AI reviewer.

### Step 2. Report the gaps

Before changing anything, show the user a table of the seven layers (section 4): what exists, what is missing, what the plan or CI can't enforce, and what you propose. Keep the proposal proportional: a small repo with one maintainer needs a fraction of this.

### Step 3. Add the files

Land them the way the repo already lands changes: through a PR if it uses PRs, otherwise as a normal commit the user reviews. Typical files:

| File | From | Notes |
|---|---|---|
| `CLAUDE.md` (add a "Verify your work" section) | Layer 1 | The exact commands; short |
| `REVIEW.md` at the repo root | `assets/REVIEW.md` | Fill in with this repo's failure modes, paths and rules |
| `.github/pull_request_template.md` | `assets/pull_request_template.md` | Testing, Risk, How to revert |
| `.github/CODEOWNERS` | Layer 5 | Risky paths mapped to owning people or teams |
| CI workflow for lint, types, tests, build | Layer 4 | Only if missing; match the existing CI system |
| Security review workflow or local command habit | Layer 3 | |
| Formatter/linter hooks | Layer 1 | `.claude/settings.json` hooks and/or git hooks |

### Step 4. List the settings the user must change

Repo and org settings are shared, so list them and **ask before applying any**, even if you have the permissions. Typical list:

- Branch protection or a ruleset on the default branch: require a PR, required status checks, required approvals, require review from code owners, dismiss stale approvals on new commits, require conversation resolution before merging, block force pushes.
- Merge queue on the default branch.
- CI start conditions (for CI configured outside the repo).
- Code Review: turn it on (Owner of the Claude organization), choose repos, review level, and when reviews run (section 9).
- Who is on the trusted-author list for lighter approval paths.

### Step 5. Roll out gradually

Start the AI reviewer on one repo. Tune REVIEW.md for a few weeks using what people do with its comments (section 5), then widen. Start risk tiers strict and loosen them only with evidence from the metrics in section 1.

---

## 3. The life of a change

The full pipeline from the author's machine (or a Claude session started from chat) to users:

| Stage | Step | Who | What happens |
|---|---|---|---|
| Before the PR | Work is verified | Automated | Claude runs tests, build and linter and shows the output; a second agent reviews the diff in a fresh context |
| On every PR | Required CI | Automated | Lint, types, tests across platforms, build and packaging. Red blocks the merge |
| On every PR | AI code review | Automated | Several agents hunt for bugs in parallel, verifiers vote on each finding, survivors are ranked by severity |
| On every PR | Security review | Automated | A separate pass. Any open finding from either review blocks the merge |
| On every PR | Risk-based approval | A person, scaled by risk | A classifier sets the tier: zero, one or two people. Some paths always need a person |
| Merging | Merge queue | Automated | Checks run again on the latest main with this change applied |
| After merge | Flag, internal use first | Recovery | New behaviour ships behind a flag to employees before everyone |
| After merge | Gradual release | Recovery | A "latest" channel ships first; the "stable" channel trails it |
| After merge | Watch, fix forward or roll back | Recovery | Author and on-call watch; flags turn off quickly; broken builds get reverted |

Automation covers most of the path, and people spend their attention where the risk sits.

---

## 4. The seven layers

For each layer: what it is for, what to set up, and the details that make it work.

### Layer 1. Verification before anyone else looks

**Why:** most review now happens before a PR exists. An agent that can run something to check its work produces much better results; Boris Cherny, who created Claude Code, says that letting Claude verify its own work is probably the single most important factor in getting good results, and estimates it makes the final result two to three times better. Verification differs by domain: a shell command, a test suite, a build, or exercising the app in a browser or phone simulator and comparing a screenshot to the design.

**Set up:**

1. **Give the agent checks it can run, listed in CLAUDE.md.** A short "Verify your work" section with the exact commands: test, build, typecheck, lint, format, and how to run the app or take a screenshot for UI work. Example:

   ```markdown
   ## Verify your work
   Before saying a change is done, run these and paste the output:
   - `npm run lint && npm run typecheck`
   - `npm test -- --related` (full suite: `npm test`)
   - `npm run build`
   - UI changes: run `npm run dev`, screenshot the changed screen, compare with the design.
   If anything fails, fix it or say plainly what still fails.
   ```

2. **Require evidence, not claims.** The agent shows the command and what it returned, instead of saying "done". This output later goes into the PR template's Testing section.
3. **Hooks run formatters and linters automatically.** A Claude Code `PostToolUse` hook on `Edit|Write` that runs the formatter, and/or a git pre-commit hook. If the repo has no linter, add the language's standard one first: it removes a whole class of review comments before any reviewer sees them.
4. **Fresh-context review of the diff.** A second agent that did not write the code reviews it, so it isn't anchored on the author's reasoning. Options: run `/code-review` (and `/security-review`) in Claude Code before opening the PR, or have the authoring agent spawn a subagent that is given only the diff, the stated intent and the repo, with an instruction like "find correctness bugs, regressions and security issues in this diff; cite file and line for each; say plainly if there is nothing to fix."
5. **Optional pre-PR skills.** The reference team runs a few of their own before opening a PR: one hunts for bugs, one simplifies the diff, one checks behaviour end to end, and UI changes are checked against design guidelines. Suggest equivalents only if the user wants them.

**Fix repeated findings at the source.** When the automated reviewer keeps flagging the same kind of thing, don't keep fixing it PR by PR. Move the rule to where it prevents the problem: a lint rule, a skill, CLAUDE.md, or REVIEW.md.

**CLAUDE.md versus REVIEW.md:**

| | CLAUDE.md | REVIEW.md |
|---|---|---|
| Says | What the project is: stack, conventions, goals, non-goals, how to verify work | What the reviewer should care about in this part of the code |
| Read by | Every Claude session, including every review | The reviewer |
| Written by | The team | Code owners (put REVIEW.md itself in CODEOWNERS) |
| Size | Short. In a large codebase it points to other docs instead of containing them | Short |
| Reviewer behaviour | The reviewer flags code that violates one of its conventions, and changes that leave one of its lines outdated | The reviewer follows it; check separately that it actually does |

**What a good REVIEW.md contains** (the template in `assets/REVIEW.md` has all of these):

- **What failure looks like in this repo,** in a phrase. For docs: "it lies". For config: "it misroutes". For billing: "it charges wrong".
- **A cap on nits** (for example, at most 3 per review).
- **Paths to skip:** generated code, vendored code, lockfiles, snapshots.
- **Evidence rules:** cite a file and line, not a guess from a function or variable name.
- **Summary format:** start with a count of findings, and say plainly when there is nothing to fix.
- Optionally: paths that always need a human, and the finding tags (layer 2).

### Layer 2. AI review on every pull request

**Why:** every PR gets a machine review before a person looks, so humans see a change that has already been through at least one round of bug-hunting.

**What the reviewer does:** it reads the diff against the whole repo and against the PR's stated intent, and asks three questions: does the change solve the problem it claims to, does it fit the system, and does it break anything else? On a bug fix, it looks for the root cause rather than accepting a patch to the symptom. (This is why the PR template's "What and why" matters: it is the intent the reviewer checks against.)

**Evidence it works:** when Code Review launched as a research preview inside Anthropic, the share of internal PRs that got substantive review comments rose from **16% to 54%**. PRs above 1,000 lines usually get findings; small PRs usually come back clean.

**How findings are produced (and how to emulate it if you build your own):**

1. **Find:** several agents look for bugs in parallel.
2. **Verify by vote:** a separate group of agents votes on each candidate finding; it is posted only if enough of them agree it is real.
3. **Filter:** findings that are only about style, diagnostics or cleanup are dropped. The reference team A/B tested these filters on their own PRs before keeping them.
4. **Rank and post:** what survives is posted as inline comments, sorted by severity.

**Finding tags.** Each finding is tagged:

| Tag | Meaning | Why it exists |
|---|---|---|
| Important | A real problem this PR should fix | Separates must-fix from the rest |
| Nit | Minor, optional | Lets people (and agents) skip it cheaply |
| Pre-existing | A bug the PR did not introduce | So the author isn't blamed for old code, and doesn't feel obliged to fix it in this PR |

**Every finding must be answered:** either fix it, or resolve the thread with a short reason. Often an agent picks up the fixes overnight, so the author returns to a PR that has already been through another round.

**Choose the review depth per repo by risk:** Quick, Standard (default) or Deep in Code Review. Deep is worth its extra time and cost where a single missed bug costs more than the review. Quick suits routine codebases.

**Which tool:**

| Situation | Use |
|---|---|
| Team or Enterprise Claude plan, GitHub | Code Review for Claude Code (section 9) |
| Any plan, PRs on GitHub or GitLab, CI minutes available | Claude Code in your own CI (Claude Code GitHub Action, or GitLab CI/CD) |
| Any plan, no PRs or no CI budget | `/code-review` locally before each push, enforced by hooks (`references/solo.md`) |
| Want a custom reviewer | A subagent or skill you write, run in a session |

Roll out on one repo first; tune REVIEW.md for a few weeks; then widen.

### Layer 3. Security review that can block

**Why:** security issues need their own focused pass; a general bug hunt misses them.

**Set up:** a separate automated security review on every PR: the open-source Claude Code security review GitHub Action (`anthropics/claude-code-security-review`), or `/security-review` in Claude Code locally.

**Blocking rule:** a PR cannot merge while any finding from either the bug review or the security review is still open. Clear a finding by fixing it, or by closing the thread with a short reason (this is how false positives get cleared). If a later commit fixes the issue, the finding clears on its own.

**How to enforce on GitHub:** turn on "Require conversation resolution before merging" in branch protection or a ruleset. Note that Code Review's own check run never concludes as a failure and a review never blocks or approves, so blocking comes from requiring threads resolved, not from the check.

### Layer 4. Automated checks that block

**Why:** the cheapest, most reliable layer. Anything a machine can check deterministically should never reach a human or an AI reviewer.

**Set up:**

- Required CI on every PR: lint, type checks, tests on every supported platform, build and packaging. A typical PR in the reference CLI repo runs dozens of checks before anyone looks. Red blocks the merge.
- Mark the checks as required status checks in branch protection or a ruleset.
- **PR template** with at least **Testing** (how it was tested, with evidence), **Risk**, and **How to revert**. Use `assets/pull_request_template.md`. Claude can fill it in, but the person who asked for the change signs off on what it says.

### Layer 5. Human approval, scaled by risk

**Why:** a docs typo and an auto-updater change carry different risk, so they should not get the same review. Uniform review either wastes senior attention on trivia or under-reviews the dangerous changes.

**Risk tiers:**

| Tier | Examples | Human approvals |
|---|---|---|
| Low | Docs, copy, flag-gated additive changes, internal-only features | None, once automated review and security checks are green |
| Medium | Ordinary product changes; changes to sensitive areas such as caching, the auto-updater, system prompts | One person |
| High | The highest-risk code, plus a short list of paths that always need two people | Two people |

**Approval rules:**

1. **The change sets the tier, not the author.** A senior author does not make a risky change low-risk.
2. **A classifier picks the tier.** It reads the change and assigns Low, Medium or High.
3. **A second check can escalate.** Separately ask: does *this particular* change need another person even though its tier says it doesn't? (For example, a "docs" PR that also edits a script.)
4. **Some paths always need a person:** any CI workflow file, and anything its owners have marked sensitive. Typical list: auth, billing and payments, CI and release workflows, signing, installers and updaters, secrets handling, migrations.
5. **Only trusted authors get the lighter paths.** Keep a trusted-author list per repo, usually the owning team. Everyone else needs a second person to approve.
6. **A bot's approval never fills a human slot.** Automated review can clear a Low change, but it cannot stand in where a person is required.
7. **Whoever prompted the change is accountable for it.** When an agent wrote the PR, whoever directed that agent fills one of the required reviewer slots, exactly as if they had typed the code.
8. **An approval covers one specific commit.** If new commits arrive that the system can't vouch for (an agent's push, a merge conflict someone resolved by hand), the approval must be given again. On GitHub: "Dismiss stale pull request approvals when new commits are pushed".
9. **Start strict, loosen with evidence.** Begin with more approvals than you think you need; reduce as revert and incident data show it is safe.

**Implementing tiers on GitHub** (GitHub has no native per-tier approval count; this is one practical way, not the reference team's internal system):

- **Always-human paths:** `.github/CODEOWNERS` maps them to people or teams; branch protection turns on "Require review from Code Owners". Include `.github/workflows/`, `.github/CODEOWNERS` and `REVIEW.md` themselves.
- **Classifier:** a CI job that labels each PR `risk:low|medium|high`, from paths touched (and optionally a Claude call on the diff that also answers the "needs another person anyway?" question).
- **Gate:** a required status check that reads the label, counts approvals on the current head commit from humans (exclude bot accounts), checks the author against the trusted list, and passes only when the tier's count is met. Re-run it on `pull_request_review` events. Branch protection's own required-approvals count can stay at 0 (if Low changes may merge on green checks) and the gate does the rest.
- **Without branch protection** (plan limits): put the always-human paths in REVIEW.md and CLAUDE.md as "stop and ask a human", and enforce locally (`references/solo.md`, "Enforcing locally").

### Layer 6. Merge queue, flags and gradual release

**Why:** two PRs that each pass on their own can combine into a broken main. And some problems only show up with real use.

**Set up:**

- **Merge queue:** re-runs every check against the latest main with the change applied, before it merges. On GitHub, enable the merge queue in branch protection or a ruleset and make CI trigger on the `merge_group` event as well as `pull_request`.
- **Feature flags:** new behaviour ships off by default, turned on internally first. Heavy internal use often surfaces regressions before users see them. Flags must be switchable without a deploy.
- **Staged release:** ship to a "latest" or beta channel first; the "stable" channel trails it.

### Layer 7. Fast detection, fast rollback

**Why:** some mistakes still get through. Slowing down does lower the error rate, but mostly because you do less of everything. It is better to get faster at noticing and fixing problems.

**Set up:**

- One-step rollback: flags turn off without a deploy; broken builds get reverted (keep changes small so reverts stay clean).
- An on-call rotation (weekly in the reference team) and dashboards for revert and incident rates.
- Incident reviews that ask **which layer should have caught this**, then fix that layer (a new check, a REVIEW.md line, a CLAUDE.md rule, an eval case).
- Dashboards alone won't spot a bad merge; the author and on-call still have to notice it too.
- **Not yet built, but wanted:** circuit breakers that automatically tighten approval requirements when revert or incident numbers rise. Engineers on the reference team want these before more code moves to zero-human merges. Recommend them as a goal if the user is pushing toward zero-human merges.

---

## 5. Making the AI reviewer better over time

Treat the reviewer as a system you tune with data, not a fixed tool.

### Incidents become the eval

1. Collect past PRs that caused incidents, together with their fix-forwards and reverts.
2. Have engineers who know that code confirm each bug and where it lived.
3. Add each confirmed case to an eval set; every new incident or revert adds a case.
4. Run the reviewer against the whole set on a schedule (daily in the reference team).
5. Whenever you change the reviewer (its prompt, tools, REVIEW.md, or model), rescore. Keep the change if the score went up; drop it if not.

For a small repo, a handful of past reverts run through `/code-review` after each REVIEW.md change is a reasonable start.

### Measure noise, not just recall

Catching more bugs is easy if you don't mind flagging lots of non-bugs. Balance recall with a noise measure:

- Sort every reviewer comment by how the PR's author responded: **fixed it**, **said it was wrong**, **said the behaviour was intentional**, or **decided it didn't matter**. The last three are noise signals; feed recurring ones into REVIEW.md.
- Run the reviewer on PRs known to be clean, and count how often its reports would have pushed someone into making a harmful change.
- Write down the priority order: **catch real bugs first, cut review rounds second.**

**Noise costs more with agent authors.** A person waves off a nit in seconds. An agent fixes it, which costs another commit plus another full review round.

### Review has to converge

When both author and reviewer are agents, they can keep each other going forever.

- In the reference team, one CLI PR went through **more than fifty** review-and-fix rounds overnight before its author stopped it.
- Running `git blame` on what the reviewer kept flagging showed about **59%** of it was code that had existed since the reviewer's first pass. Each round, the reviewer was mostly surfacing a few more old bugs rather than new ones.
- The team had made it worse by running several rounds by default, because each pass found something. Multiple rounds can pay off on a risky change, but not on every PR.

**Changes that fixed it** (apply all of them):

1. **Round cap:** once three rounds have passed, the reviewer tells the author to pause and rethink the approach rather than keep patching.
2. **Diff-only re-review:** after the first few passes, re-review only the new commits, not the whole PR.
3. **Flag the class of bug:** name the pattern so the author fixes every instance at once, not one line per round.
4. **No repeats:** never re-post a finding already posted.
5. **Track rounds-to-pass alongside recall** as a reviewer metric.
6. **Nightly sweeps:** review agents also run over existing code, catching old bugs before they surface in someone's PR.

**Results on internal PRs, before and after:**

| Metric | Before | After |
|---|---|---|
| Pass after one review-and-fix round | 65% | 78% |
| Need five or more rounds | 8.9% | 1.5% |
| Mean reviews to pass | 2.08 | 1.32 |

When a review misses something, ask what the model didn't have in front of it, and put that into CLAUDE.md, a skill, or a new check.

---

## 6. Where humans stay in the loop

An automated review is far cheaper than interrupting a senior engineer's deep work to catch a routine mistake. People remain responsible for:

- **Whether to build it at all:** direction, taste, scope. One team that considered its review bottleneck solved still kept humans reviewing changes from outside the team, so that someone still asks "should we even do this?"
- **Risky code:** billing, the auto-updater, CI workflows, and the rest of the always-human list.
- **Architecture that sets a pattern** others will copy.
- **Reading the evidence:** the test output, the command that ran and what it returned, screenshots, the reviewer's findings. If you can't say what the checks would have missed, ask for a real review instead of giving a quick approval.
- **Owning the outcome:** a named person is accountable for every change that ships, whoever (or whatever) typed it.

Note: Code Review itself never approves a PR; approving stays a human call. Some teams, including the reference team in their own repos, let owners decide which low-risk changes may merge on automated checks alone, and that line keeps moving as evidence accumulates.

---

## 7. Sharing context when machines do the reading

Review's second job was spreading understanding: reading a teammate's diff is how you learn what changed. With machines doing most of the reading, replace that deliberately:

- Nobody needs a complete mental model of the whole codebase. When someone needs to understand a part, they ask Claude for an **explainer of exactly that slice**. That takes minutes and matches the code as it stands now.
- Going further: have Claude build a small interactive page or diagram that visualises that part of the code, tailored to how that person learns.
- This beats reading trivial diffs. A one-line copy change teaches almost nothing; an explainer for the module you are about to change teaches a lot.
- **Work in the open:** default to public team channels so people see what others are building while they build it, can pick up threads, and give feedback on code they don't own.

Suggest to the user: when someone is about to change an unfamiliar module, generate an explainer first.

---

## 8. Team norms

### "You land it, you own it"

When a small or medium change you landed breaks, fixing it is usually your job. This works only with supporting conditions; set these up alongside:

- **Small changes:** easy to review, debug and revert.
- **Everything reversible:** most mistakes can be undone within minutes using flags, reverts and staged releases.
- **Blast radius sets the bar:** costly code gets more reviewers and more checks.
- **Fast signals:** telemetry and internal use surface regressions quickly.
- **Not blame:** the owner fixes it because they have the most context; incident reviews examine the system, not the person.
- **Shared ownership:** open channels mean anyone can spot a gap, point a Claude at it, or comment.

### Change direction early

Agents make starting work cheap, so some work shouldn't land. Usually someone asks why, cites an earlier decision, or proposes a simpler path, and the author then closes the PR or reshapes it. Because the code was fast to write, closing it isn't a loss. Raise a different direction **early in the thread**, not as a blocking comment on a finished PR.

---

## 9. Code Review for Claude Code: product reference

What an agent configuring the managed reviewer needs to know.

**What it does:** Claude reviews PRs in GitHub repositories. Several agents read each change with the surrounding code, looking for logic errors, security vulnerabilities, broken edge cases and similar. It focuses on **correctness** by default (bugs that would break production), not formatting preferences or missing test coverage; change that with REVIEW.md and CLAUDE.md.

**Pipeline**, run on Anthropic infrastructure:

1. **Find:** several agents examine the diff and surrounding code in parallel.
2. **Verify:** candidate findings are checked against the code; false positives are filtered out.
3. **Rank:** remaining findings are deduplicated and ranked by severity.
4. **Post:** results go to the PR.

**What appears on the PR:**

- Inline comments on the affected lines, plus an overall summary in the review body.
- A check run named **Claude Code Review** listing the findings in its details. **It never concludes as a failure.**
- When nothing is found, a note instead of findings, and the check run title reads **Code review found no issues**.
- **A review never approves or blocks a PR.** The merge decision stays with the team. To make findings blocking, require conversation resolution in branch protection (layer 3).

**Availability and requirements:**

| Requirement | Detail |
|---|---|
| Claude plan | Team or Enterprise |
| Code host | GitHub, including GitHub Enterprise Server, and GitHub Enterprise Cloud with data residency (the Owner enters the enterprise's `ghe.com` hostname during the GitHub Enterprise Server admin setup) |
| Not available with | Zero Data Retention; the HIPAA configuration (and not covered by the BAA); customer-managed encryption keys (CMEK), not yet |
| Cloud providers | Teams using Claude Code through Amazon Bedrock, Google Cloud's Agent Platform or Microsoft Foundry still need a Claude organization on Team or Enterprise |
| Who turns it on | An **Owner** of the Claude organization: installs the Claude GitHub App, picks repositories, the review level and when reviews run |

**Settings:**

| Setting | Options | Notes |
|---|---|---|
| Review level | **Quick** (lightest, fastest, cheapest), **Standard** (default, balanced), **Deep** (most thorough, slowest, most expensive) | Set once for the organization; override per repo, such as Quick for routine code and Deep for critical code. More thorough costs more |
| When reviews run (per repo) | Once when a PR opens; after every push; or only on request | Whatever the setting, anyone with write access can comment `@claude review` on a PR to start one |
| Model | Not configurable | Code Review picks the models; use the review level to change thoroughness |
| Spend | Billed per review; a monthly spend limit can be set, and usage is tracked | Once the limit is reached, reviews are skipped for budget; if a review didn't run, check this first |

**Behaviour to know:**

- Automatic reviews skip **draft PRs and PRs from forks**.
- After some pushes it **keeps the earlier result** instead of re-reviewing.
- A comment command starts a review only when it comes from someone with **write access**.
- **REVIEW.md is read from the base or default branch, never from the PR's own changes.** So a PR that edits REVIEW.md is reviewed with the old one, and a PR cannot loosen its own review. Land REVIEW.md changes first, then test them on a later PR.
- **Access:** inside a review, repo access is read-only and it never pushes commits. On the PR it posts comments and the check run, resolves its own earlier threads, and may react to a comment command with an eyes emoji. The GitHub App holds a broader permission set shared with other Claude features.
- **It reads, it doesn't run:** reviews work from the diff and surrounding code without building or running anything. Runtime verification still belongs to layers 1 and 4.
- Runs only in Anthropic's managed environment, not in self-hosted environments.
- A review with no findings does not mean the code is bug-free or secure.

**All the ways to get a Claude review:**

| Option | Started by | Runs on | Who can use it |
|---|---|---|---|
| Code Review | PR activity as configured per repo, or an `@claude review` comment | Anthropic infrastructure | Team or Enterprise orgs with GitHub repos, once an Owner turns it on |
| `/code-review` | You run it on your branch | Your Claude Code session | Anyone, any plan |
| Ultrareview | You start it from Claude Code | A cloud sandbox | Needs a claude.ai account sign-in; not available everywhere |
| Claude in your own CI (Claude Code GitHub Action, or GitLab CI/CD) | Pipeline events, such as every PR or merge request | Your CI runners | Anyone on GitHub or GitLab who has a Claude API key or cloud-provider credentials |
| Your own reviewer as a subagent or skill | You or Claude invoke it in a session | Wherever Claude Code runs | Anyone who writes one |

---

## 10. Adoption checklist

**Before the PR**

- [ ] Short CLAUDE.md lists the checks the agent runs (tests, build, lint, typecheck, how to see UI).
- [ ] Hooks run formatters and linters; a standard linter exists for each language.
- [ ] Agents show check output instead of claiming "done".
- [ ] `/code-review` and `/security-review` run locally, or a fresh-context subagent reviews the diff.
- [ ] REVIEW.md exists, owned by code owners; repeated review comments get moved into it (or into a lint rule, skill or CLAUDE.md).

**On every PR**

- [ ] Required CI: lint, types, tests across platforms, build. Red blocks merge.
- [ ] PR template has What and why, Testing, Risk, and How to revert.
- [ ] AI reviewer (Code Review for Claude Code, or the Claude Code GitHub Action) enabled on a single repo; REVIEW.md tuned for a few weeks; then widened.
- [ ] Automated security review on every PR.
- [ ] Every review thread fixed or resolved with a reason before merge (conversation resolution required).
- [ ] Review depth matches risk: Quick for routine repos, Deep where a missed bug is expensive.
- [ ] Review rounds capped (three, then rethink); after the first few, re-review only the new diff.

**Approval**

- [ ] Risk tiers written down, set by the change not the author; start strict, loosen with evidence.
- [ ] Lighter paths only for authors on a trusted list.
- [ ] CODEOWNERS plus branch protection so auth, billing, CI and release workflows, signing and installers always need a person.
- [ ] Bot approvals never replace a required human; approvals dismissed on new commits.
- [ ] The person directing an agent counts as a reviewer of its PR and signs off on the template.

**Merge and after**

- [ ] Merge queue, feature flags, internal use first, staged release, one-step rollback.
- [ ] Incidents and reverts collected into an eval set; reviewer rescored whenever its prompt, tools or model change.
- [ ] Reviewer noise measured (fixed / wrong / intentional / didn't matter; clean-PR false alarms) and rounds-to-pass tracked.
- [ ] Human time per PR measured (approvals median and p95, comments, wait time), with revert and incident rates as the guardrail.
- [ ] Incident reviews ask which layer should have caught it.

---

## 11. Gotchas

- **Code Review doesn't block on its own.** Its check run never fails and it never approves. Blocking needs "require conversation resolution" (and the plan to support branch protection).
- **REVIEW.md edits don't apply to the PR that makes them.** It is read from the base branch.
- **A required check needs branch protection or rulesets.** On plans without them, CI and reviewers can only advise; enforce locally instead and tell the user.
- **CI outside the repo** (Xcode Cloud, Bitrise, etc.) must be configured in that system to run on PRs and report status, or it can't gate anything.
- **Merge queue needs CI on `merge_group`** (GitHub Actions), otherwise queued PRs wait forever.
- **Review loops don't converge on their own.** Cap rounds at three, diff-only re-review, flag bug classes, no repeated findings.
- **Noise costs more with agent authors.** Drop style-only findings, cap nits, tag Pre-existing so agents don't "fix" unrelated old code in this PR.
- **Measure noise, not just recall,** and run the reviewer on known-clean PRs.
- **Bots don't count as humans,** and an agent's push after approval resets the approval.
- **Don't recommend what the plan can't do.** Probe first (Workflow step 1).
- **Don't change shared settings without asking,** even with admin rights.
- **Gradual is the point:** one repo first, strict tiers first.
- **Humans still own** whether to build it, risky paths, pattern-setting architecture, reading the evidence, and the outcome.

---

## 12. Files in this skill

- `references/solo.md`: solo developer or agent-only repos; enforcing locally with a Claude Code `PreToolUse` hook plus committed git hooks, with full example scripts; limited CI; linting.
- `assets/REVIEW.md`: REVIEW.md template to copy to the repo root and fill in.
- `assets/pull_request_template.md`: PR template to copy to `.github/pull_request_template.md`.

## Based on

Credit only; everything needed is above.

- "How the Claude Code team does code review", Addy Osmani, Morgan Westlee Lunt and Qing Wang, claude.dev Blog (Anthropic), Oct 15, 2026. https://claude.dev/blog/solving-code-review-at-anthropic/
- "Code Review", Claude Code docs (Anthropic). https://code.claude.com/docs/en/code-review
- anthropics/claude-code-action and anthropics/claude-code-security-review on GitHub. https://github.com/anthropics/claude-code-action, https://github.com/anthropics/claude-code-security-review
