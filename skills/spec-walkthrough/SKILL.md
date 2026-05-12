---
name: spec-walkthrough
description: Use when a feature-spec-keeper (or similar reverse-engineering agent) opens a batch of spec PRs with `## Open questions for the rewrite` sections that need human answers before merge. The skill groups questions, applies the team's status-quo-for-features-fix-bad-architecture policy as default recommendations, walks PRs one at a time, captures answers, patches the spec doc, and pushes. Triggers on "walk the open questions on this PR", "answer the spec questions in batch", "let's resolve feature-spec PRs one by one", or any "I have N spec PRs with open questions, help me work through them" request.
---

# Spec PR Open-Questions Walkthrough

## Overview

When the `feature-spec-keeper` (or similar reverse-engineering agent) opens a batch of spec PRs, each typically ends with:

```markdown
## Open questions for the rewrite

1. Should X be configurable?
2. Should we add Y?
...
```

The human review work is answering those questions and patching them into the spec as "Questions resolved for the rewrite". This skill makes that work fast and consistent across PRs.

**Announce at start:** "I'm using the spec-walkthrough skill to process N spec PR(s)."

## The Policy

Default lens for every question (this is the team's standing decision, articulated 2026-05-12):

> **Status-quo for features; fix bad architecture only.**
>
> The rewrite ships current behavior unless something is genuinely poor architecture worth correcting. No feature creep. Architectural fixes that don't add new product behavior — just centralize, secure, or correct existing patterns — are in scope.

Apply this when proposing default answers. Each question gets one of:

- **No (status-quo)** — feature add or behavior change with no clear win.
- **YES (architectural fix)** — corrects bad architecture, security gap, or data-integrity issue without adding features.
- **Codebase question** — needs grep, not human input; resolve inline.
- **Out of scope** — belongs to a different spec or is tracked elsewhere; reference it.
- **Deprecated** — the feature is being shelved; document and skip.

## The Process

### Step 1: Resolve the PR queue

Get the explicit PR list (user-given or via label):

```bash
gh pr list --repo <owner>/<repo> --state open --label feature-spec-keeper \
  --json number,title,body,headRefName,reviewDecision --limit 50
```

For each PR, count open questions:

```bash
gh pr view <pr> --repo <owner>/<repo> --json body | \
  jq -r '.body' | awk '/^## Open questions/,0' | grep -cE "^[0-9]+\."
```

Print the queue with counts:

```
Queue (in walk order):
  #1157  8 questions  Product Workflow & Stages
  #1159 10 questions  AI Orchestrator
  #1160 10 questions  Auth & Permissions
  ...
```

Confirm with the user before starting.

### Step 2: For each PR, present questions clean

Fetch the PR's spec file and extract the open-questions section:

```bash
git fetch origin <pr-branch>
git show origin/<pr-branch>:docs/features/<slug>.md | awk '/^## Open questions/,0'
```

Then **for each question**, present:

- **Q#.** The question (bolded short title) + 1-line context.
- **Recommend:** the default answer under the status-quo+fix-arch policy.

Example format (this is the format that worked well in practice):

```
**Q1. User-configurable workflow phases?** → **No.** Currently hard-coded. Keep as-is.

**Q2. Global AI error handling / fallback strategy?** → **YES — adopt in the rewrite.** Per-callsite handling is bad architecture; one circuit breaker / fallback abstraction at the orchestrator boundary is the right shape and doesn't add features.

...
```

End the block with a summary:

```
Net change: N architectural fixes, M status-quo, K codebase greps I'll resolve inline.

Reply "yes" / "go" to apply, or call out specific overrides (e.g., "Q3 flip to yes, rest yes").
```

### Step 3: For codebase questions, grep before asking

If a question is factual ("Where does X live?", "Are draft milestones in a separate table?"), do the grep and inline the answer instead of asking the user. Examples worth grepping:

- "Is org isolation enforced in query Y?" — grep for `organizationId` / `orgId` joins in the queries dir.
- "What permissions does this surface use?" — grep `protect(` in the actions dir.
- "Are there special validation rules for X?" — read the validator dir.
- "What's the engine version constant?" — grep `const.*VERSION.*=`.

Mark these `(codebase resolution, no user input needed)` in the question list so the user knows you're not asking them.

### Step 4: Apply answers to the spec doc

Once the user confirms:

```bash
git checkout -B <pr-branch> origin/<pr-branch>
```

Edit the spec doc:

1. Replace `## Open questions for the rewrite` heading with `## Questions resolved for the rewrite`.
2. Add a one-paragraph note up top:
   ```markdown
   > Resolved YYYY-MM-DD by @<user>. Policy: **status-quo for features; fix bad architecture.**
   > Net change in scope: <N> architectural fixes (<Q list>); <M> status-quo.
   ```
3. Convert each question line to a confirmed answer with brief justification.
4. For deprecated features (whole feature being shelved), instead add a DEPRECATED banner at the top of the spec and a "intentionally unresolved" note over the still-open questions, then also flag the feature in `docs/features/_INDEX.md`.

Commit with a clear message:

```bash
git add docs/features/<slug>.md
git commit --no-verify -m "$(cat <<'EOF'
docs(spec): resolve open questions on <Feature Name>

All <N> questions answered. Policy: status-quo for features; fix bad
architecture only. Net change in scope:

- Q<X>: <decision> (<one-line why>)
- ...

Other Q's status-quo or codebase-resolved.

Co-Authored-By: <you>
EOF
)"
git push --no-verify
```

### Step 5: Move to the next PR

Don't try to merge — that's `merge-batch`'s job. Just push answer commits one PR at a time, then let the user say "merge all".

When the queue is exhausted, summarize:

```
🎉 All <N> spec PRs answered and pushed.
  ✓ #1157, ✓ #1159, ✓ #1160, ...

Net architectural fixes baked in:
  - <feature>: <fix>
  - ...

Ready to hand off to /merge-batch.
```

## Safety Rules

- **Never invent answers.** If a question doesn't have a clear status-quo default (i.e., the codebase doesn't tell you and the policy doesn't apply), surface it to the user — do not make it up.
- **Always offer the user override.** Default recommendations must be overridable per-question. The prompt format is: list defaults, then ask "yes / overrides?".
- **Never merge.** This skill answers + pushes; merging is `/merge-batch`.
- **Flag deprecation explicitly.** When the user says "we're killing this feature for now", do not auto-resolve the questions; instead, mark the spec DEPRECATED, leave questions intentionally unresolved, flag in `_INDEX.md`, commit.
- **Honor reverse-spec-keeper conventions.** Use the same `## Questions resolved for the rewrite` heading; preserve the per-question structure; don't reshape the doc.

## What This Skill Does Not Do

- It does **not** rewrite the spec body. Only the questions section + optional DEPRECATED banner + _INDEX.md entry.
- It does **not** trigger CI rebases or merges. Hand off to `/merge-batch` after.
- It does **not** make product/architecture decisions on the user's behalf when both policies could apply — surface it.
