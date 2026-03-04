---
name: work-on-issue
description: Start working on a GitHub issue — create a linked branch, plan implementation in phases, execute one commit per phase, and open a draft PR at the end. Never merges without explicit user approval. Works with any coding agent.
---

# Work on GitHub Issue

Given a GitHub issue number, create a linked branch, produce a phased implementation plan, execute each phase as a commit, and open a draft PR. The agent orchestrates this process — it does not implement alone without a plan.

> **oh-my-opencode users:** See [`work-on-issue-omo`](../work-on-issue-omo/SKILL.md) for a version with Oracle-driven planning and Hephaestus implementation, with ultrawork mode enabled by default.

## Trigger

Use this skill when the user says any of:
- "work on issue #N", "start issue N", "implement issue N"
- "pick up issue N", "work-on-issue N"

## Arguments

`issue_number` (required) — the GitHub issue number to work on.

## Step 0 — Fetch Issue Details

```bash
gh issue view {issue_number} \
  --json number,title,body,labels,assignees,comments \
  --jq '{number: .number, title: .title, body: .body, labels: [.labels[].name], comments: [.comments[].body]}'
```

Also get the repo context:

```bash
gh repo view --json owner,name,defaultBranchRef \
  --jq '{owner: .owner.login, name: .name, default_branch: .defaultBranchRef.name}'
```

Store: `{owner}`, `{repo}`, `{issue_number}`, `{issue_title}`, `{issue_body}`, `{default_branch}`.

## Step 1 — Create a Linked Branch

```bash
gh issue develop {issue_number} --checkout
```

This creates a branch linked to the issue on GitHub and checks it out automatically. GitHub names it from the issue title (e.g. `42-add-dark-mode-toggle`).

> If you need a custom branch name: `gh issue develop {issue_number} --checkout --name "feat/issue-{issue_number}-{slug}"`

Confirm you are on the new branch:

```bash
git branch --show-current
```

## Step 2 — Produce a Phased Implementation Plan

Analyze the issue body, labels, and any linked code context. Produce a plan where:

- **Each phase = one logical unit of work = one commit**
- Phases are ordered by dependency (foundational work first)
- Each phase specifies:
  - Goal (one sentence)
  - Files to create or modify
  - Acceptance criterion (how you know it's done)

Print the plan before starting any implementation:

```
Implementation Plan — Issue #{issue_number}: {issue_title}
══════════════════════════════════════════════════════════
Phase 1: {goal}
  Files:    {list of files}
  Done when: {acceptance criterion}

Phase 2: {goal}
  Files:    {list of files}
  Done when: {acceptance criterion}

...
```

**Stop here and ask the user to confirm the plan before proceeding.**

```
Does this plan look right? Reply "yes" to start, or tell me what to change.
```

## Step 3 — Execute Each Phase

For each phase, in order:

1. Implement the change
2. Verify it works (run tests, type-check, lint as appropriate)
3. Commit immediately with a descriptive message:

```bash
git add <affected files>
git commit -m "{type}({scope}): {description}

Phase {N}/{total} — Issue #{issue_number}"
```

**One commit per phase. No batching phases.**

Print a short status after each commit:

```
✅ Phase N/{total} complete — {commit_sha_short}: {commit message}
```

## Step 4 — Push the Branch

After all phases are committed:

```bash
git push -u origin HEAD
```

## Step 5 — Open a Draft PR

```bash
gh pr create \
  --title "{issue_title}" \
  --body "$(cat <<'EOF'
Closes #{issue_number}

## Summary

{2-3 sentence description of what was implemented and why}

## Phases

{numbered list of phases with their commit SHAs}

## Testing

{how to verify the changes work}
EOF
)" \
  --draft
```

Print the PR URL when done.

> **IMPORTANT:** Always create the PR as `--draft`. **Never merge without explicit user approval.** The user must review and approve before any merge action is taken.

## Step 6 — Summary

```
✅ Issue #{issue_number} ready for review.

Branch:  {branch_name}
PR:      {pr_url} (draft)
Commits: {N} phases

Phases committed:
  {sha} {phase 1 commit message}
  {sha} {phase 2 commit message}
  ...

Next: review the PR, request review, or ask me to address feedback.
```

## Commit Convention

```
{type}({scope}): {description}

Phase {N}/{total} — Issue #{issue_number}
```

Types: `feat`, `fix`, `refactor`, `test`, `docs`, `chore`

Examples:
- `feat(auth): add JWT token generation — Phase 1/3 — Issue #42`
- `test(auth): add unit tests for token expiry — Phase 2/3 — Issue #42`
- `docs(auth): update API reference for new endpoints — Phase 3/3 — Issue #42`

## Anti-Patterns

**NEVER:**
- Start implementing before the plan is confirmed by the user
- Bundle multiple phases into one commit — one phase = one commit
- Merge the PR — always leave it as draft; only the user merges
- Create the PR without `--draft` flag
- Skip the branch creation step — always work on a dedicated branch

**NEVER proceed alone if:**
- The issue body is ambiguous and you cannot infer intent — ask the user
- A phase requires knowledge you don't have — ask before guessing
