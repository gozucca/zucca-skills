---
name: neon-prune
description: Use when CI is failing because Neon's create-branch action returns HTTP 422 (rate-limited at concurrent branch creation, or near a project quota) or when stale preview / CI / dev-local branches have piled up in a Neon project. The skill lists branches, filters by age + creation source + protection state, shows the user the deletion candidates for confirmation, and deletes in batches via the Neon MCP. Triggers on "prune Neon branches", "clean up Neon", "delete old previews on Neon", "Neon is at quota", or any "CI is failing on create-branch" investigation.
---

# Prune Stale Neon Branches

## Overview

The Neon Vercel integration auto-creates a preview branch per Vercel deployment. With multiple PRs open and active dev work, these accumulate fast — both `ready` and `archived` states still count against quota and contribute to concurrent-branch-create rate limits. CI's `db-data-integrity` / `db-schema-integrity` jobs also create short-lived `ci/*` branches; if these fail to clean up, they orphan.

This skill identifies safe-to-delete branches and removes them. **Defaults are conservative** — only branches that match clear stale-preview / orphaned-CI patterns are queued. Production snapshots, dev-named branches that look intentional, and branches with children are excluded.

**Announce at start:** "I'm using the neon-prune skill to clean up <project>."

## When to Use

- CI jobs are returning HTTP 422 from Neon's `create-branch-action` (rate limit or quota).
- A periodic cleanup is overdue (>2 weeks since last prune).
- The user says "Neon is acting up" or "previews are piling up".

Do not use when:

- A specific named branch needs deletion — just use `mcp__neon__delete_branch` directly.
- The user wants to delete production snapshots or anything protected — surface and refuse.

## The Process

### Step 1: Identify the project

```bash
# Use mcp__neon__list_projects with search="<org>" to find candidates
```

If multiple projects are returned (e.g., Zucca has `Zucca`, `Zucca A`, `Zucca Agents`, `zucca-dev`), confirm with the user which one to prune. For Zucca specifically, the main app database is `jolly-haze-38017780` (project `Zucca`).

### Step 2: Snapshot current state

Call `mcp__neon__describe_project` for the chosen project. The result includes all branches; jq-extract them to a working file:

```bash
jq -r '.[1].text' <describe-result> | sed 's/^It contains the following branches[^[]*//' > /tmp/branches.json
```

Print a summary the user can scan:

```
Project: <name> (<id>)
Total branches: <N>
By state: archived=<X>, ready=<Y>
By source: vercel=<A>, console=<B>, github=<C>
```

### Step 3: Build the deletion queue (conservative defaults)

Filter rules — a branch is a candidate only if **all** apply:

- `protected == false`
- `default == false`
- `primary == false`
- `created_at < <cutoff>` (default: 14 days ago; user can override)
- Name matches one of these patterns:
  - `preview/` — Vercel-created preview branches
  - `ci/` — CI-created integrity-check branches
- **Has no children** — `.id` does not appear as any other branch's `parent_id`

Explicitly exclude (even if old):

- Names matching `^(staging|develop|production|test|test-1|test-2|main|master)$`
- Names containing `production_old_`, `prod-backup-`, `test-migration_old_` (intentional snapshots)
- Bare developer-named branches (e.g., `spencer/staging`, `theo/local`) — these look intentional; user must explicitly opt-in

Implement with jq:

```jq
([.[] | .parent_id // empty] | unique) as $parents |
[.[] | select(
  .protected == false and
  .default == false and
  .primary == false and
  .created_at < $cutoff and
  (.name | test("^(preview/|ci/)")) and
  (.name | test("^(staging|develop|production|test|test-1|test-2|main|master)$") | not) and
  (.id as $id | $parents | index($id) | not)
)]
```

### Step 4: Show the user, get confirmation

Print the queue (or a sample if it's huge) with `created_at`, `current_state`, `creation_source`, and `name`:

```
Deletion candidates: 142
  Oldest:
    br-XXX 2025-07-02 archived vercel preview/...
    br-YYY 2025-07-15 archived vercel preview/...
  Newest:
    br-AAA 2026-04-25 ready    vercel preview/...
  By source: vercel=128, ci=14

Proceed? (yes / show-all / refine-filter / abort)
```

If the queue exceeds ~150 branches, also ask whether to batch in chunks (typical Neon throughput: ~20 deletes per parallel call is fine; the MCP handles concurrency).

### Step 5: Delete in batches

Use `mcp__neon__delete_branch` per branch. Issue calls in batches of 20 (parallel within one tool-call message) for throughput. Print one line per success or failure.

After each batch, the user should be able to interrupt cleanly — don't loop autonomously through hundreds without check-ins. Aim for 1–3 batches per turn, then summarize and ask.

### Step 6: Final report

```
🎉 Deleted <N> stale branches from <project>.
Remaining: <M> branches (was <M+N>).

If CI is still flaking, the issue isn't quota — investigate Neon API rate
limits or the create-branch retry behavior in the workflow.
```

## Safety Rules

- **Never delete protected, default, or primary branches.** Even if the user explicitly says "delete everything", refuse and surface.
- **Never delete production snapshots.** Names matching `production_old_*`, `prod-backup-*`, `test-migration_old_*` are intentional backups; the rewrite-team relies on them.
- **Never delete a branch with children.** A non-empty `parent_id` set referencing it means deleting it would orphan or break downstream branches. Skip.
- **Never auto-extend the cutoff.** If the user says "go more aggressive", get an explicit new cutoff (e.g., "7 days" or "3 days") — don't guess.
- **Always confirm the queue before deletion.** No silent runs. The user must see the candidates and approve.
- **Print one line per delete.** "Branch deleted successfully" output from the MCP is fine; don't suppress it. The user wants to see progress.

## Common Use Cases

### Routine cleanup (CI is fine, just hygienic)

```
/neon-prune
```

Defaults: Zucca main project, 14-day cutoff, preview/ + ci/ only.

### CI is failing on create-branch right now

```
/neon-prune --aggressive
```

Same project, 3-day cutoff (the user must confirm — aggressive means deleting active-recent previews). Adds: even ready-state preview branches if their corresponding PR is closed/merged.

### Specific project

```
/neon-prune --project=<name-or-id>
```

E.g., `--project="Zucca A"` to clean up the secondary app's previews.

## What This Skill Does Not Do

- It does **not** create or restore branches.
- It does **not** modify project settings (autoscaling, retention, etc.).
- It does **not** purge based on storage size — only by name + age + state.
- It does **not** delete from outside the Zucca org or a project the user hasn't explicitly named.
