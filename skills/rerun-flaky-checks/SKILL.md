---
name: rerun-flaky-checks
description: Use when one or more open PRs are blocked on transient CI failures — typically `db-data-integrity` / `db-schema-integrity` flapping on Neon rate-limits, `promptfoo-eval` flaking on LLM rate-limits, or `e2e` cascade-failing because preview-env Clerk auth hiccupped. The skill identifies the latest FAILED run for each named check across a PR list, reruns just those failed jobs (not whole workflows), surfaces persistent failures so they get human attention, and stops cleanly. Triggers on "rerun the failed checks", "retry CI on these PRs", "rerun any failed promptfoo", "the db checks are flaking again", or any "CI flake retry on a batch" request.
---

# Rerun Transient CI Failures Across a PR Batch

## Overview

Zucca's CI has a few known flake patterns:

- **`db-data-integrity` / `db-schema-integrity`** — fail with HTTP 422 when Neon rate-limits concurrent branch creates (many CI jobs starting at once).
- **`promptfoo-eval`** — fails on LLM provider rate-limits or transient model outages.
- **`e2e`** — cascade-fails when Clerk auth on the preview deployment hiccupped (one test's sign-in fails → all dependent tests fail → cascade across the suite).

For these, the first-pass debugging is "rerun and see if it sticks." This skill does that across a PR list cleanly.

It does NOT rerun whole workflows blindly (that wastes minutes). It identifies the **latest** failing run of each named check on each PR and reruns only the failed jobs.

**Announce at start:** "I'm using the rerun-flaky-checks skill to retry failed checks on N PRs."

## When to Use

- A batch of PRs is BLOCKED on `db-*`, `promptfoo-eval`, or `e2e` failures after a Neon prune or preview-env hiccup.
- A monitor is reporting "⚠️ BLOCKED" on multiple PRs simultaneously.
- The user says "rerun any failed X across all PRs" or "the e2e checks are flaking".

Do not use when:

- A single PR has a single failed check — just rerun manually via `gh run rerun <id> --failed`.
- The failure mode is clearly a real bug, not a flake. Investigate the failure first.
- The same check has failed >2x in a row on the same PR with the same failure mode — that's not flaky, that's broken. Surface and stop.

## The Process

### Step 1: Resolve the PR list

```bash
# Explicit list
PRS=(1155 1156 1157 ...)

# Or by label
gh pr list --repo <owner>/<repo> --state open --label feature-spec-keeper \
  --json number --jq '[.[].number]'
```

### Step 2: For each PR, find the latest failing run of each watched check

Default watchlist: `db-data-integrity`, `db-schema-integrity`, `promptfoo-eval`, `e2e`. User can extend with `--checks <name1,name2>`.

For each PR:

```bash
gh pr view "$pr" --repo <owner>/<repo> --json statusCheckRollup > /tmp/s-$pr.json

# Latest failure per watched check name
jq -r --arg checks "db-data-integrity,db-schema-integrity,promptfoo-eval,e2e" '
  ($checks | split(",")) as $watch |
  [.statusCheckRollup[] | select(.name and (.name|type=="string") and ($watch | index(.name)))] |
  group_by(.name) | map(sort_by(.startedAt) | last) |
  .[] | select(.conclusion == "FAILURE") | "\(.name)\t\(.detailsUrl)"
' /tmp/s-$pr.json
```

Build a table of (PR, check, run_id) to rerun.

### Step 3: Detect non-flaky failures

For each candidate rerun, look at history: if the same check has FAILED on this PR ≥2 times in the recent rollup, mark as "likely real" and exclude from the rerun batch.

```bash
jq -r --arg name "<check>" '
  [.statusCheckRollup[] | select(.name == $name and .conclusion == "FAILURE")] | length
' /tmp/s-$pr.json
```

If count >= 2, surface and skip:

```
⚠️  #<pr> <check> — failed 2+ times. Likely a real failure; investigate instead of rerunning.
```

### Step 4: Show the user, get confirmation

```
Will rerun:
  #1162 db-schema-integrity  run-25761206503
  #1163 db-data-integrity    run-25761206504
  #1164 db-data-integrity    run-25761206505
  #1215 db-data-integrity    run-25761206506
  #1215 db-schema-integrity  run-25761206507

Skipped (likely real, not flaky):
  #1138 e2e — failed 3x in a row with same error

Proceed? (yes / details / abort)
```

### Step 5: Rerun the failed jobs

For each entry in the table:

```bash
gh run rerun <run_id> --repo <owner>/<repo> --failed
```

`--failed` reruns only the failed jobs within the workflow run, not the whole workflow. That's the cheap, correct retry.

Print one line per rerun.

### Step 6: Final report

```
🔄 Reran <N> failed checks across <M> PRs.
Skipped (need human review): <K> checks on <J> PRs.

The monitor (or you) will see fresh CI runs land. If the reruns also fail,
the failure isn't flaky — investigate.
```

## Safety Rules

- **Never use `gh run rerun` without `--failed`.** Rerunning a whole workflow wastes minutes and obscures which job actually flaked. `--failed` reruns just the failing jobs.
- **Never rerun a check that has failed 3+ times.** That's not transient; it's broken. Surface and stop.
- **Never rerun checks the user didn't list.** If the user asked for "failed promptfoo", do not also rerun e2e or db-* on the same pass. Respect the scope.
- **Always show the user the rerun table before firing.** Each `gh run rerun` consumes Actions minutes; the user should confirm.
- **Never rerun on closed/merged PRs.** Check PR state first.

## Common Use Cases

### Rerun any failed db-integrity across all open spec PRs

```
/rerun-flaky-checks label:feature-spec-keeper --checks db-data-integrity,db-schema-integrity
```

### Rerun any failed promptfoo across an explicit list

```
/rerun-flaky-checks 1155,1156,1157 --checks promptfoo-eval
```

### Default behavior — rerun all known-flake checks on a queue

```
/rerun-flaky-checks 1155,1156,1157,1159,1160
```

Watchlist defaults to `db-data-integrity, db-schema-integrity, promptfoo-eval, e2e`.

## What This Skill Does Not Do

- It does **not** investigate failure root causes. If reruns also fail, the user investigates.
- It does **not** rerun successful or in-progress runs. Only failures.
- It does **not** rerun checks outside its watchlist unless the user explicitly extends it.
- It does **not** merge anything — pairs with `/merge-batch` for that.

## Pairs Well With

- `/neon-prune` — if `db-*` failures are quota-driven, prune first then rerun.
- `/merge-batch` — after flaky reruns clear, run merge-batch to land the queue.
- `/spec-walkthrough` — same PR batches that need flaky-check reruns often also need spec answers.
