# Solo and agent-only repos

The source article describes a team. With one person directing agents, there is no second human to approve, so the human-approval layer collapses into "the owner reads the evidence". Keep the rest of the layers: they matter more, not less, because nobody else is looking.

This adaptation is this skill's own, built from the article's layers.

## What changes

| Team layer | Solo version |
|---|---|
| Risk-tiered approvals (0/1/2 people) | Two tiers: **auto** (low risk, merges on green checks) and **owner reads it** (everything on the risky-path list). |
| CODEOWNERS + branch protection | A risky-path list in REVIEW.md and CLAUDE.md, plus a pre-push hook that stops pushes touching those paths until the owner confirms. Use branch protection too if the plan includes it. |
| AI review on every PR | Still worth it. If the repo doesn't use PRs, run a fresh-context `/code-review` and `/security-review` before every push instead, from CLAUDE.md or a hook. |
| Required CI on every PR | Run the fast test subset in a pre-push hook; let the full CI run on main. Configure the CI to also run on PRs if and when PRs are used. |
| Merge queue | Usually unnecessary with one writer at a time. Needed once several agents push in parallel. |

## Limited CI compute

When CI minutes are scarce, put the cheap checks (format, lint, fast unit tests) locally in hooks, and keep CI for the full suite on main or on PRs touching risky paths.

## Still keep

- Agents show check output instead of saying "done".
- A REVIEW.md, even for one person: it is what the AI reviewer reads.
- The review-round cap and nit limits from the main skill: an agent author and an agent reviewer loop fastest when nobody else is watching.
- Revert as the rollback, and small changes so reverts stay clean.
