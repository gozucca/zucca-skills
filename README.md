# zucca-skills

A Claude Code marketplace of skills for Zucca's engineering and fleet operations.

## What's in here

| Skill | When to use |
|---|---|
| [`merge-batch`](skills/merge-batch/SKILL.md) | Merge a batch of related PRs (coverage-keeper, feature-spec-keeper, etc.) that all touch the same append-only log files. Auto-handles the merge → rebase → merge cascade with safety guards. |
| [`spec-walkthrough`](skills/spec-walkthrough/SKILL.md) | Walk the `## Open questions for the rewrite` section across a batch of spec PRs. Applies the team's status-quo-for-features-fix-bad-architecture policy as defaults; user overrides per-question; patches the spec doc + pushes. |
| [`neon-prune`](skills/neon-prune/SKILL.md) | Clean up stale preview / CI branches in a Neon project when CI is hitting create-branch rate limits or a periodic cleanup is overdue. Conservative defaults: only `preview/*` and `ci/*` branches older than 14 days that have no children. |
| [`rerun-flaky-checks`](skills/rerun-flaky-checks/SKILL.md) | Rerun the latest failure of each watched check (default: `db-*`, `promptfoo-eval`, `e2e`) across a batch of PRs. Skips checks that have failed 3+ times — those aren't flaky, they're broken. |

More candidates: `review-pr-batch`, `bump-versions`, `release-coordination`.

## Install (one-time, per teammate)

Add this marketplace to your `~/.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "zucca-skills": {
      "source": {
        "source": "github",
        "repo": "gozucca/zucca-skills"
      }
    }
  }
}
```

That's it. The skills become available in every Claude Code session, regardless of which repo you're working in. Updates flow automatically — Claude pulls the latest marketplace on each session.

If you already have an `extraKnownMarketplaces` entry, just add `zucca-skills` as a sibling key.

## Using a skill

In any Claude Code session, type `/<skill-name>` or describe the task in natural language — the skill will trigger based on its `description` frontmatter. Example:

```
/merge-batch label:coverage-keeper
```

or just:

```
"merge all the open coverage-keeper PRs"
```

## Contributing

Skills live in `skills/<name>/SKILL.md`. The format is a Claude Code skill file:

```markdown
---
name: <skill-name>
description: <when-to-use sentence — this drives triggering>
---

# <Skill Title>

## Overview
...

## The Process
...

## Safety Rules
...
```

To add a new skill:

1. Create `skills/<name>/SKILL.md`.
2. Add a row to the table above in this README.
3. Open a PR. After merge, teammates get the new skill on their next Claude Code session — no reinstall needed.

To version a breaking change, bump `version` in `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json`. Otherwise minor edits ship automatically.

## Relationship to other Zucca repos

- **`zucca-skills` (this repo)** — *interactive* Claude Code slash commands. What a human triggers in their IDE session.
- **`zucca-app-a/agents/`** — *autonomous* agents (coverage-keeper, feature-spec-keeper, release-keeper, …). What runs in CI on cron.
- **`zucca-app/.claude/agents/`** — agent prompts referenced by `zucca-app-a/agents/` runners.
- **`zucca-knowledge-base/docs/`** — engineering docs, architecture, patterns.

Skills and agents are different things. Don't conflate them.
