# Solo and agent-only repos

The main skill describes a team. With one person directing agents there is no second human to approve, so the human-approval layer collapses into "the owner reads the evidence". Keep every other layer: they matter more, not less, because nobody else is looking.

This adaptation is this skill's own, built from the team process in SKILL.md and from how Claude Code hooks work. Everything needed is in this file.

## What changes

| Team layer | Solo version |
|---|---|
| Risk-tiered approvals (0/1/2 people) | Two tiers: **routine** (lands once local checks and the local AI review pass) and **owner reads it** (anything on the risky-path list). |
| CODEOWNERS + branch protection | A risky-path list in REVIEW.md and CLAUDE.md, enforced locally (below). Use branch protection too if the plan includes it. |
| AI review on every PR | If the repo uses PRs, keep it. If changes go straight to main, run a fresh-context `/code-review` and `/security-review` before every push. |
| Required CI on every PR | With push-to-main, CI runs only after code is already on main, so it can tell you but not stop you. Put the fast checks (format, lint, fast tests) before the push; let CI run the full suite on main. If you later use PRs, configure the CI to run on them too. |
| Merge queue | Usually unnecessary with one writer at a time. Needed once several agents push in parallel. |
| "Person directing the agent counts as a reviewer" | Always true here: the owner is accountable for everything the agents land, and signs off on risky-path changes by reading the evidence (test output, the command that ran, screenshots, review findings). |
| Incident review | Still ask "which layer should have caught this?" after every revert, and add the fix to REVIEW.md, CLAUDE.md, a lint rule or the hook. |

## The risky-path list

Write it once and use it in three places: REVIEW.md ("always needs the owner"), CLAUDE.md ("stop and ask the owner before changing these"), and the hooks below. Keep the machine-readable copy in `.claude/risky-paths.txt`, one shell glob per line:

```text
# Paths that need the owner to read the change before it is pushed.
.github/workflows/*
.githooks/*
.claude/settings.json
.claude/risky-paths.txt
REVIEW.md
CLAUDE.md
*auth*
*billing*
*payment*
migrations/*
fastlane/*
*.entitlements
*.xcconfig
```

Include the hook files and the list itself, so an agent can't quietly loosen its own gate.

## Enforcing locally

Git hooks alone are weak gates for agents: an agent can pass `--no-verify`, a pre-push hook can't prompt without `/dev/tty`, and `.git/hooks` isn't committed. Use two layers that share one script:

1. **A Claude Code `PreToolUse` hook on Bash**, committed in `.claude/settings.json`, filtered with `"if": "Bash(git push*)"`. It runs the fast checks and checks whether the commits about to be pushed touch a risky path. If they do, it blocks with **exit code 2**; Claude Code shows the hook's stderr to Claude, which should then stop and ask the owner. This stops agents even when they would skip git hooks, because it runs before the command does.
2. **A committed git pre-push hook** for pushes made outside Claude: put it in a tracked folder such as `.githooks/` and set `git config core.hooksPath .githooks` (each clone needs this once; mention it in the README or a setup script). Run the same checks; for risky paths, refuse and print what needs the owner, rather than prompting.

### Shared check script: `.githooks/check-push.sh`

Takes a commit range, runs fast checks, and fails if the range touches a risky path. Replace the fast-check commands with the repo's own.

```bash
#!/usr/bin/env bash
# Usage: check-push.sh <base-commit> <head-commit>
set -uo pipefail
base="$1"; head="$2"
root="$(git rev-parse --show-toplevel)"
cd "$root" || exit 1

# 1. Fast checks. Keep these under a minute; CI runs the full suite.
fast_checks() {
  # Examples, pick the repo's own:
  # npm run lint && npm run typecheck && npm test -- --changedSince="$base"
  # swift-format lint --recursive --strict Sources Tests
  # ruff check . && pytest -q -x tests/unit
  true
}
if ! out="$(fast_checks 2>&1)"; then
  echo "Fast checks failed; fix before pushing:" >&2
  echo "$out" | tail -40 >&2
  exit 1
fi

# 2. Risky paths.
list="$root/.claude/risky-paths.txt"
[ -f "$list" ] || exit 0
hits=()
while IFS= read -r file; do
  while IFS= read -r pat; do
    [[ -z "$pat" || "$pat" == \#* ]] && continue
    # shellcheck disable=SC2053  # glob match is intended
    if [[ "$file" == $pat ]]; then hits+=("$file  (matches $pat)"); break; fi
  done < "$list"
done < <(git diff --name-only "$base" "$head")

if [ ${#hits[@]} -gt 0 ]; then
  echo "These changes touch paths that need the owner to read them:" >&2
  printf '  %s\n' "${hits[@]}" >&2
  exit 3
fi
exit 0
```

### Claude Code hook: `.claude/hooks/pre-push.sh`

```bash
#!/usr/bin/env bash
# PreToolUse hook for Bash. Reads the tool call as JSON on stdin.
cmd="$(jq -r '.tool_input.command // empty')"
[[ "$cmd" =~ git.*push ]] || exit 0   # not a push

cd "${CLAUDE_PROJECT_DIR:-.}" || exit 0
base="$(git rev-parse --verify -q '@{u}' || git merge-base HEAD origin/HEAD 2>/dev/null)"
[ -n "$base" ] || exit 0              # nothing to compare against; let CI catch it

msg="$(.githooks/check-push.sh "$base" HEAD 2>&1)"
case $? in
  0) exit 0 ;;
  3) echo "$msg" >&2
     echo "Do not push. Stop and ask the owner to read this change (show the diff, test output and review findings). The owner pushes it themselves after reading." >&2
     exit 2 ;;
  *) echo "$msg" >&2
     echo "Fix the failing checks, then push again." >&2
     exit 2 ;;
esac
```

Register it in `.claude/settings.json` (committed):

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "if": "Bash(git push*)",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/pre-push.sh"
          }
        ]
      }
    ]
  }
}
```

Notes:

- **Exit code 2 blocks the tool call** and feeds stderr back to Claude. Any other non-zero exit is treated as a hook error and does not block, so the wrapper maps every failure to 2.
- The `if` filter keeps the hook from running on every Bash call. It matches commands that start with `git push`. If agents push through compound commands (`cd app && git push`, `git -C app push`), drop the `if` line; the script's own `git.*push` check then does the filtering on every Bash call.
- The hook needs `jq`. Make both scripts executable (`chmod +x`) and commit them.
- The owner's way through is to push from their own terminal after reading the change, with `ALLOW_RISKY_PUSH=1` (below). Don't give agents an override they can set themselves.

### Git pre-push hook: `.githooks/pre-push`

```bash
#!/usr/bin/env bash
# git passes: <local ref> <local sha> <remote ref> <remote sha> per line on stdin.
zero="0000000000000000000000000000000000000000"
status=0
while read -r _ local_sha _ remote_sha; do
  [ "$local_sha" = "$zero" ] && continue          # branch deletion
  if [ "$remote_sha" = "$zero" ]; then             # new branch
    base="$(git merge-base "$local_sha" origin/HEAD 2>/dev/null)" || continue
  else
    base="$remote_sha"
  fi
  "$(git rev-parse --show-toplevel)/.githooks/check-push.sh" "$base" "$local_sha"
  rc=$?
  if [ $rc -eq 3 ] && [ "${ALLOW_RISKY_PUSH:-}" = "1" ]; then
    echo "Risky paths pushed with ALLOW_RISKY_PUSH=1 (owner override)." >&2
  elif [ $rc -ne 0 ]; then
    [ $rc -eq 3 ] && echo "Owner: read the change, then push with ALLOW_RISKY_PUSH=1 git push" >&2
    status=1
  fi
done
exit $status
```

Then: `chmod +x .githooks/*` and `git config core.hooksPath .githooks`.

## Limited CI compute

When CI minutes are scarce, put the cheap checks in the hooks above and keep CI for the full suite on main, or on PRs touching risky paths. If CI is configured outside the repo (Xcode Cloud start conditions in App Store Connect, Bitrise workflows and so on), set its start conditions there; it can report on main even when nothing can make it a required check.

## Linting

If the language ships a formatter or linter, wire it in before adding an AI reviewer; it removes a class of review comments. For Swift, `swift-format` ships with the toolchain (`swift-format lint --recursive --strict`, `swift-format format --in-place --recursive`). Run the formatter from a Claude Code `PostToolUse` hook on `Edit|Write` so agent edits are formatted as they happen, and the linter in `check-push.sh`.

## Still keep

- Agents show check output instead of saying "done".
- A REVIEW.md, even for one person: it is what the AI reviewer reads. Use `assets/REVIEW.md`.
- A fresh-context review before every push (`/code-review`, `/security-review`), with every finding fixed or dismissed with a reason.
- The review-round cap and nit limits from the main skill: an agent author and an agent reviewer loop fastest when nobody else is watching. After three rounds, rethink the approach; after that, review only the new diff.
- Feature flags for anything user-facing that might need turning off.
- Revert as the rollback, and small changes so reverts stay clean.
- After a revert, add the case to a small eval set (the PR and the bug) and rerun the reviewer against it when REVIEW.md or the model changes.
- An explainer from Claude before touching a module you haven't looked at in a while, instead of reading old diffs.
