# Solo and agent-only repos

No second human, so approval becomes "the owner reads the evidence". Keep every other layer.

- **Tiers:** *routine* lands once local checks and local AI review pass; *owner reads it* for risky paths.
- **Review:** with PRs keep the AI reviewer; push-to-main runs `/code-review` and `/security-review` before every push.
- **CI:** on push-to-main it reports but can't block. Fast checks run pre-push; CI runs the full suite on main. CI outside the repo: set start conditions there.
- **Merge queue:** only once several agents push in parallel.
- **After a revert:** ask which layer should have caught it; add it to a small eval set.

## Risky-path list: `.claude/risky-paths.txt`

One glob per line; mirror in REVIEW.md and CLAUDE.md. It lists itself and the hooks so agents can't loosen the gate.

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

## Enforcing locally

Agents can skip git hooks (`--no-verify`), so pair a Claude Code `PreToolUse` hook with a committed git pre-push hook, sharing one script.

### Shared check script: `.githooks/check-push.sh`

Replace the fast-check commands with the repo's own.

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

Register in `.claude/settings.json` (committed):

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

- Only **exit 2** blocks (stderr goes to Claude); other codes don't, hence the mapping.
- `if` only matches commands starting `git push`; for `cd app && git push` or `git -C`, drop it.
- Needs `jq`; `chmod +x` and commit both scripts.
- Owner override: `ALLOW_RISKY_PUSH=1` from their own terminal. Agents get no override.

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

Then `chmod +x .githooks/*` and `git config core.hooksPath .githooks` (once per clone).

## Also

Formatter/linter before AI review (Swift: `swift-format`; formatter in a `PostToolUse` hook on `Edit|Write`, linter in `check-push.sh`). Keep: evidence not "done", a REVIEW.md, every finding fixed or dismissed with a reason, the three-round cap, flags, small revertable changes.
