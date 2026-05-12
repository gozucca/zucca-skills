---
name: merge-batch
description: Use when you need to merge a batch of related PRs (a feature-spec-keeper run, a coverage-keeper run, a docs-bot run, etc.) where each merge triggers conflicts on append-only log files in the others. The skill resolves the merge-rebase-merge cascade automatically, surfaces blockers, and stops on any unrecoverable conflict. Triggers on "merge all the X PRs", "merge this batch", "merge every coverage-keeper PR", or any "we have N related open PRs, merge them in order" flavor of request.
---

# Merge a Batch of Related PRs

## Overview

When a keeper-style agent (feature-spec-keeper, coverage-keeper, dead-code-keeper, etc.) opens multiple PRs in parallel that all touch the same append-only log file (`docs/coverage-log.md`, `docs/features/_LOG.md`, etc.), merging them sequentially is annoying:

1. Merge PR #1 → PRs #2…N all become `BEHIND` and conflict on the log file.
2. Rebase each one → push.
3. Merge PR #2 → repeat.

This skill automates the loop with safety: it sequentially merges only `CLEAN` PRs, rebases the rest, auto-resolves conflicts in known append-only files, and bails out the moment something needs human eyes.

**Announce at start:** "I'm using the merge-batch skill to process the queue."

## When to Use

Use this skill when the user asks to merge a batch of related PRs and at least one of these is true:

- The PRs all touch a `merge=union` file listed in `.gitattributes` (e.g., `docs/coverage-log.md`, `docs/features/_LOG.md`, `CHANGELOG.md`).
- The PRs share a label (e.g., `coverage-keeper`, `feature-spec-keeper`).
- The user explicitly lists 3+ PR numbers and wants them merged in sequence.

Do **not** use this skill for:

- A single PR — just `gh pr merge`.
- PRs with unresolved review comments — those need a human first.
- PRs that have failing required checks (other than transient flakes) — investigate first.

## The Process

### Step 1: Resolve the queue

Get the explicit PR list, in user-intended merge order. If the user gave a label, query GitHub:

```bash
gh pr list --repo <owner>/<repo> --state open --label <label> \
  --json number,title,headRefName,mergeStateStatus,labels \
  --limit 50
```

Print the resolved queue back to the user with PR titles and current `mergeStateStatus`, and **wait for confirmation** before starting the loop. Do not auto-run. Example:

```
Queue (in order):
  #1221 [BEHIND]  chore(git): add docs/features/_LOG.md to merge=union
  #1155 [CLEAN]   docs: spec Formula Management & Conflict Resolution
  #1156 [CLEAN]   docs: spec HITL workflow
  ...

Proceed? (yes / reorder / drop #N / abort)
```

### Step 2: Pre-flight checks

For each PR in the queue, before starting:

1. **Open review comments** — refuse if any PR has unresolved review threads or change requests.
   ```bash
   gh pr view "$pr" --json reviewDecision,reviews \
     --jq '.reviewDecision, [.reviews[] | select(.state=="CHANGES_REQUESTED")] | length'
   ```
   If `reviewDecision == "CHANGES_REQUESTED"` or any review with state `CHANGES_REQUESTED`, skip the PR and surface it.

2. **Open-question pattern** — refuse if any spec-style PR body still has an unresolved `## Open questions` section. Use a quick heuristic:
   ```bash
   gh pr view "$pr" --json body | jq -r '.body' | \
     grep -E "^##\s+Open questions" && echo "has unresolved questions"
   ```
   Surface and skip.

3. **Required checks failing for non-flaky reasons** — if a check has failed more than twice with the same failure mode, treat as non-flaky and bail.

### Step 3: Sequential merge + rebase loop

```bash
QUEUE=(<resolved PR numbers, in order>)
declare -A BRANCH=( [pr_number]="branch/name" ... )

while [ ${#QUEUE[@]} -gt 0 ]; do
  # Find first CLEAN PR
  merged_one=""
  for pr in "${QUEUE[@]}"; do
    mstate=$(gh pr view "$pr" --repo <owner>/<repo> --json mergeStateStatus --jq .mergeStateStatus)
    if [ "$mstate" = "CLEAN" ]; then
      echo "[$(date +%H:%M:%S)] merging #$pr"
      gh pr merge "$pr" --repo <owner>/<repo> --merge --delete-branch
      merged_one="$pr"
      break
    fi
  done

  if [ -n "$merged_one" ]; then
    QUEUE=($(for p in "${QUEUE[@]}"; do [ "$p" != "$merged_one" ] && echo "$p"; done))
    [ ${#QUEUE[@]} -eq 0 ] && break

    # Rebase the rest
    git fetch origin <base-branch>
    for pr in "${QUEUE[@]}"; do
      br="${BRANCH[$pr]}"
      git fetch origin "$br"
      git checkout -B "$br" "origin/$br"
      git rebase "origin/<base-branch>"
      # Auto-resolve append-only log conflicts (relies on .gitattributes merge=union
      # for the target file). If the conflict is in any file NOT matching these
      # patterns, abort and surface.
      while git status --short | grep -qE "^UU "; do
        for f in $(git status --short | awk '/^UU /{print $2}'); do
          case "$f" in
            docs/*_LOG.md|docs/*-log.md|docs/*-backlog.md|CHANGELOG.md|docs/features/_LOG.md)
              sed -i.bak -e '/^<<<<<<< /d' -e '/^=======$/d' -e '/^>>>>>>> /d' "$f"
              rm -f "$f.bak"
              git add "$f"
              ;;
            *)
              echo "❌ #$pr CONFLICT in $f — needs manual resolution"
              git rebase --abort
              # Remove from queue, surface, continue
              break 2
              ;;
          esac
        done
        git rebase --continue
      done
      git push --force-with-lease --no-verify
    done

    # Wait for GitHub to recompute mergeability + new check runs to start
    sleep 25
  else
    echo "[$(date +%H:%M:%S)] waiting on ${QUEUE[*]}"
    sleep 60
  fi
done
echo "[$(date +%H:%M:%S)] ALL MERGED"
```

### Step 4: Surface every state transition

The loop should emit a single stdout line per noteworthy event:

- `[HH:MM:SS] merging #N` — about to call `gh pr merge`.
- `🟣 #N — MERGED` — confirmed merged.
- `❌ #N CONFLICT in <path>` — manual resolution required; PR dropped from queue.
- `⚠️ #N BLOCKED (failing: <checks>)` — checks not transient.
- `[HH:MM:SS] waiting on <PR list>` — idle pass.

These are what the user reads to follow along; do not produce other noise.

### Step 5: Handle a manual conflict

When the loop drops a PR with `❌ CONFLICT in <path>`:

1. Stop the loop (or set the PR aside; don't merge subsequent ones until you've resolved).
2. Read the conflicting file.
3. Make a judgment call on resolution (typically: keep the change from the PR's commit, reconcile with current numbering/structure from `HEAD`).
4. `git add <path>`, `git rebase --continue`, `git push --force-with-lease`.
5. Resume the loop with the remaining queue.

### Step 6: Final report

When the loop exits:

```
🎉 ALL MERGED (8 of 8)
  ✓ #1221, ✓ #1155, ✓ #1156, ✓ #1157, ✓ #1159, ✓ #1160, ✓ #1161, ✓ #1164

Skipped (manual handling needed):
  ❌ #1163 — _INDEX.md numbering conflict (resolved + merged separately)
```

Include the merge-commit SHAs on staging if the user might want to revert.

## Safety Rules

- **Never bypass required status checks.** This skill respects `enforce_admins`. If a PR's checks aren't green, the skill waits or surfaces a blocker; it never uses `--admin`.
- **Never auto-resolve a non-log conflict.** Only `.gitattributes merge=union` paths get the strip-markers treatment. Anything else → drop PR from queue with `❌ CONFLICT` event.
- **Never silently drop a review-requested PR.** If a PR has `CHANGES_REQUESTED`, surface it before starting and require user override.
- **Never merge a PR whose body still has `## Open questions`.** Surface and require user confirmation that the questions are intentionally unresolved (e.g., a deprecated feature).
- **Always announce.** "I'm using the merge-batch skill to process N PRs."

## Common Use Cases

### Coverage-keeper batch

```
/merge-batch label:coverage-keeper
```

The agent label-queries, prints the queue, asks for go-ahead, then runs.

### Feature-spec-keeper batch

```
/merge-batch label:feature-spec-keeper
```

The same pattern — common today across all `docs/feature-spec-*-YYYY-MM-DD` branches that share `docs/features/_LOG.md`.

### Explicit PR list

```
/merge-batch 1155,1156,1157,1159,1160
```

Useful when you want a subset, or a specific order.

## What This Skill Does Not Do

- It does **not** decide *whether* PRs should be merged. The user is responsible for review and approval; the skill is just the merge-rebase mechanic.
- It does **not** rewrite PR commits or squash. It uses GitHub's default merge method for the target branch.
- It does **not** mass-revert. If a merge needs reverting, do it manually with `gh pr revert` or `git revert`.
