---
name: work-on-issue
description: Start working on a GitHub issue — create a linked branch, plan implementation in phases, execute one commit per phase, and open a draft PR at the end. Never merges without explicit user approval. Works with any coding agent.
---

# Work on GitHub Issue

Given one or more GitHub issue numbers, create a linked branch per issue, produce a phased implementation plan (submitted for user approval via `submit_plan`), execute each phase as a commit, and open a draft PR. The agent orchestrates this process — it does not implement alone without an approved plan.

> **oh-my-opencode users:** See [`work-on-issue-omo`](../work-on-issue-omo/SKILL.md) for a version with Oracle-driven planning and Hephaestus implementation, with ultrawork mode enabled by default.

## Trigger

Use this skill when the user says any of:
- "work on issue #N", "start issue N", "implement issue N"
- "pick up issue N", "work-on-issue N"
- "work on issues #9 and #11", "tackle issues 9, 11"

## Arguments

- `issue_numbers` (required) — one or more GitHub issue numbers. Examples: `42`, `9 11`, `9, 11`
- `comment` (optional) — free-text context or instructions to guide the plan. Passed directly to the planner.

Examples:
```
work-on-issue 42
work-on-issue 9 11
work-on-issue 42 "focus on the API layer, skip UI changes for now"
work-on-issue 9 11 "use the existing auth middleware pattern"
```

## Multi-Issue Behavior

When multiple issue numbers are provided, process them **sequentially** — one full cycle (branch → plan → implement → PR) per issue, in the order given. Do not start issue N+1 until issue N has a merged-ready draft PR open.

After completing all issues, print a final summary listing all PRs created.

---

## Per-Issue Workflow

Repeat the following steps for **each** issue number.

### Step 0 — Fetch Issue Details

```bash
gh issue view {issue_number} \
  --json number,title,body,labels,assignees,comments \
  --jq '{number: .number, title: .title, body: .body, labels: [.labels[].name], comments: [.comments[].body]}'
```

```bash
gh repo view --json owner,name,defaultBranchRef \
  --jq '{owner: .owner.login, name: .name, default_branch: .defaultBranchRef.name}'
```

Store: `{owner}`, `{repo}`, `{issue_number}`, `{issue_title}`, `{issue_body}`, `{default_branch}`.

### Step 1 — Create a Linked Branch
Derive a slug from the issue title: lowercase, spaces and special characters replaced with hyphens, consecutive hyphens collapsed, truncated to 40 characters, no leading/trailing hyphens.

```
slug = issue_title
         .toLowerCase()
         .replace(/[^a-z0-9]+/g, '-')
         .replace(/^-+|-+$/g, '')
         .slice(0, 40)
branch_name = `feat/issue-{issue_number}-{slug}`
```

Examples:
- Issue #42 "Add dark mode toggle" → `feat/issue-42-add-dark-mode-toggle`
- Issue #9 "Fix: user can't log in with SSO" → `feat/issue-9-fix-user-can-t-log-in-with-sso`

```bash
gh issue develop {issue_number} --checkout --name "{branch_name}"
```

Confirm:

```bash
git branch --show-current
```

### Step 2 — Produce a Phased Implementation Plan

Analyze the issue body, labels, comments, and any linked code context. If a `comment` argument was provided, treat it as additional instructions that take priority over anything inferred from the issue.

Produce a plan where:

- **Each phase = one logical unit of work = one commit**
- Phases are ordered by dependency (foundational work first)
- Each phase specifies:
  - Goal (one sentence)
  - Files to create or modify
  - Acceptance criterion (how you know it's done)

**Call `submit_plan` with the plan — do not print it inline and do not ask via text.**

```
submit_plan(
  summary="Implementation plan for Issue #{issue_number}: {issue_title}",
  plan="""
## Issue #{issue_number}: {issue_title}

{if comment: > **User guidance:** {comment}}

---

### Phase 1: {goal}
- **Files:** {list of files}
- **Done when:** {acceptance criterion}

### Phase 2: {goal}
- **Files:** {list of files}
- **Done when:** {acceptance criterion}

...
  """
)
```

**Wait for the user to approve the plan before proceeding. Do not start Step 3 until `submit_plan` returns an approval.**

If the user annotates the plan with changes, revise and call `submit_plan` again with the updated plan.

### Step 3 — Execute Each Phase

For each approved phase, in order:

1. Implement the change
2. Verify it works (run tests, type-check, lint as appropriate)
3. Commit immediately:

```bash
git add <affected files>
git commit -m "{type}({scope}): {description}

Phase {N}/{total} — Issue #{issue_number}"
```

**One commit per phase. No batching.**

Print a short status after each commit:

```
✅ Phase N/{total} complete — {commit_sha_short}: {commit message}
```

### Step 4 — Push the Branch

```bash
git push -u origin HEAD
```

### Step 5 — Open a Draft PR

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

Print the PR URL.

> **IMPORTANT:** Always `--draft`. **Never merge without explicit user approval.**

### Step 6 — Issue Summary

```
✅ Issue #{issue_number} — {issue_title}
   Branch: {branch_name}
   PR:     {pr_url} (draft)
   Phases: {N} commits
```

If there are more issues in the list, proceed to the next one from Step 0.

---

## Final Summary (multi-issue only)

After all issues are processed:

```
✅ All issues complete.

#9  — {title}  →  {pr_url}
#11 — {title}  →  {pr_url}
...

Next: review the PRs, request reviews, or ask me to address feedback.
```

---

## Commit Convention

```
{type}({scope}): {description}

Phase {N}/{total} — Issue #{issue_number}
```

Types: `feat`, `fix`, `refactor`, `test`, `docs`, `chore`

Examples:
- `feat(auth): add JWT token generation — Phase 1/3 — Issue #42`
- `test(auth): add unit tests for token expiry — Phase 2/3 — Issue #42`

## Anti-Patterns

**NEVER:**
- Start implementing before `submit_plan` returns an approval
- Print the plan as text and ask via chat — always use `submit_plan`
- Bundle multiple phases into one commit — one phase = one commit
- Merge the PR — always `--draft`; only the user merges
- Start the next issue before the current one has an open draft PR
- Ignore the `comment` argument — it takes priority over inferred intent

**NEVER proceed alone if:**
- The issue body is ambiguous and you cannot infer intent — ask the user
- A phase requires knowledge you don't have — ask before guessing
