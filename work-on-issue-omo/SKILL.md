---
name: work-on-issue-omo
description: Start working on a GitHub issue in oh-my-opencode — create a linked branch, plan with Oracle agent (mandatory, synchronous), delegate implementation phases to Hephaestus one commit at a time, and open a draft PR. Ultrawork mode on by default. Never merges without explicit user approval.
---

# Work on GitHub Issue (oh-my-opencode)

Given one or more GitHub issue numbers, create a linked branch per issue, have Oracle produce a phased plan submitted via `submit_plan` (mandatory — no implementation starts without user approval), delegate each phase to Hephaestus as a separate commit, then open a draft PR. You are an orchestrator — you do not implement yourself.

> **Generic agents:** See [`work-on-issue`](../work-on-issue/SKILL.md) for a version that works with any coding agent (Claude Code, Codex, Cursor, etc.).

## Trigger

Use this skill when the user says any of:
- "work on issue #N", "start issue N", "implement issue N"
- "ulw work-on-issue N", "work-on-issue-omo N"
- `/work-on-issue N`
- "work on issues #9 and #11", "tackle issues 9, 11"

## Arguments

- `issue_numbers` (required) — one or more GitHub issue numbers. Examples: `42`, `9 11`, `9, 11`
- `comment` (optional) — free-text context or instructions to guide Oracle's plan. Takes priority over anything inferred from the issue.
- `ulw` (optional, **default: ON**) — ultrawork mode: Oracle planning is mandatory and synchronous, Hephaestus implements with maximum precision, no shortcuts or assumptions allowed

> Ultrawork mode is **on by default**. To disable it explicitly, the user must say `--no-ulw`.

Examples:
```
work-on-issue-omo 42
work-on-issue-omo 9 11
work-on-issue-omo 42 "focus on the API layer, skip UI changes for now"
work-on-issue-omo 9 11 "use the existing auth middleware pattern"
```

## Ultrawork Mode (ULW)

When active (default):
- Oracle planning is **mandatory and synchronous** — implementation cannot start until Oracle responds
- If Oracle times out or errors → **stop and return to user immediately**, do not proceed alone
- Hephaestus implements each phase with no shortcuts
- If Hephaestus fails or times out → **stop and return to user**, do not retry alone
- Every phase must pass verification before moving to the next

When disabled (`--no-ulw`):
- Oracle planning is still recommended but you may proceed with your own plan if Oracle is unavailable
- Hephaestus is still preferred for implementation but you may implement directly for trivial phases

## Multi-Issue Behavior

When multiple issue numbers are provided, process them **sequentially** — one full cycle (branch → Oracle plan → submit_plan approval → implement → PR) per issue, in the order given. Do not start issue N+1 until issue N has an open draft PR.

After completing all issues, print a final summary listing all PRs.

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
### Step 2 — Oracle Planning (MANDATORY)

Invoke Oracle synchronously. **Do not skip this step. Do not start implementing without Oracle's response.**

```typescript
task(
  subagent_type="oracle",
  run_in_background=false,
  load_skills=[],
  description="Plan implementation for issue #{issue_number}",
  prompt=`
You are planning the implementation of GitHub issue #${issue_number}.

ISSUE TITLE: ${issue_title}

ISSUE BODY:
${issue_body}

REPOSITORY: ${owner}/${repo}
DEFAULT BRANCH: ${default_branch}

${comment ? `USER GUIDANCE (takes priority over everything else):
${comment}

` : ""}Produce a phased implementation plan where:
- Each phase = one atomic, logically complete unit of work
- Each phase will become exactly ONE git commit
- Phases are ordered by dependency (foundational work first)
- Each phase must be implementable independently, in order

For each phase, provide:
1. Phase number and goal (one sentence)
2. Files to create or modify (be specific)
3. What the implementation should do (not how — leave that to the implementer)
4. Acceptance criterion: how to verify this phase is complete
5. Commit message in format: \`{type}({scope}): {description} — Phase {N}/{total} — Issue #{issue_number}\`

End your response with exactly: PLAN READY
  `
)
```

**If Oracle does not respond, times out, or errors:**
> Stop immediately. Tell the user: "Oracle is not responding. I cannot proceed without a plan. Please retry or let me know how to continue."

**Do not proceed to Step 3 until Oracle's response contains `PLAN READY`.**

### Step 3 — Submit Plan for User Approval

Take Oracle's plan and call `submit_plan` — **do not print the plan inline, do not ask via chat text**.

```
submit_plan(
  summary="Issue #{issue_number}: {issue_title}",
  plan="""
## Issue #{issue_number}: {issue_title}

{if comment: > **User guidance:** {comment}
>}

---

{oracle_plan_verbatim}
  """
)
```

**Wait for `submit_plan` to return an approval before proceeding. Do not start Step 4 until approved.**

If the user annotates the plan with changes:
1. Re-invoke Oracle with the updated requirements (do not modify Oracle's plan yourself)
2. Call `submit_plan` again with the revised Oracle output

### Step 4 — Execute Each Phase via Hephaestus

For each phase from the approved Oracle plan, invoke Hephaestus:

```typescript
task(
  subagent_type="hephaestus",
  run_in_background=false,
  load_skills=[],
  description="Implement Phase {N}/{total} — Issue #{issue_number}",
  prompt=`
TASK: Implement Phase ${N} of ${total} for GitHub Issue #${issue_number}.

PHASE GOAL: ${phase_goal}

FILES TO MODIFY/CREATE: ${phase_files}

WHAT TO DO:
${phase_description}

ACCEPTANCE CRITERION: ${phase_acceptance_criterion}

EXPECTED OUTCOME:
- All files from the phase files list are created or modified as described
- Acceptance criterion passes
- No regressions in existing tests
- Code compiles/type-checks cleanly

REQUIRED TOOLS: file reads, file writes, bash (for tests/lint/build only)

MUST DO:
- Implement ONLY what this phase specifies — nothing more
- Run existing tests after implementation
- Run type-check and lint if configured
- Report exactly what you changed and in which files

MUST NOT DO:
- Implement work from other phases
- Refactor code unrelated to this phase
- Suppress type errors with as any, @ts-ignore, or @ts-expect-error
- Commit anything — the orchestrator handles commits

CONTEXT:
- Repository: ${owner}/${repo}
- Issue: #${issue_number} — ${issue_title}
- Branch: ${branch_name}
- Phase ${N} of ${total} total phases
  `
)
```

**If Hephaestus does not respond, times out, or errors:**
> Stop immediately. Tell the user: "Hephaestus is not responding for Phase {N}. I cannot proceed. Please retry or let me know how to continue."

**After each Hephaestus response, verify the phase:**
1. Check the files it reported changing actually changed
2. Confirm acceptance criterion is met
3. If verification fails → tell the user, do not retry alone

**Commit the phase immediately after verification:**

```bash
git add <files changed in this phase>
git commit -m "{commit_message_from_oracle_plan}"
```

Print status:

```
✅ Phase N/{total} complete — {commit_sha_short}: {commit_message}
```

### Step 5 — Push the Branch

```bash
git push -u origin HEAD
```

### Step 6 — Open a Draft PR

```bash
gh pr create \
  --title "{issue_title}" \
  --body "$(cat <<'EOF'
Closes #{issue_number}

## Summary

{2-3 sentence description of what was implemented}

## Implementation Phases

{numbered list of phases with commit SHAs from Step 4}

## Testing

{how to verify the changes, derived from Oracle's acceptance criteria}
EOF
)" \
  --draft
```

Print the PR URL.

> **IMPORTANT:** Always `--draft`. **Never merge without explicit user approval.** This is hardcoded — do not offer to merge, do not merge automatically.

### Step 7 — Issue Summary

```
✅ Issue #{issue_number} — {issue_title}
   Branch:       {branch_name}
   PR:           {pr_url} (draft)
   Phases:       {N} commits
   Oracle plan:  yes
   ULW mode:     {on/off}
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

Next: review the PRs, request reviews, or ask me to address feedback with address-pr-comments-omo.
```

---

## Anti-Patterns

**NEVER (hard stops):**
- Start implementing before Oracle responds — no exceptions in ULW mode
- Skip `submit_plan` — never print the plan inline or ask for approval via chat
- Proceed after `submit_plan` rejection or without approval
- Implement phases yourself instead of delegating to Hephaestus
- Merge or push to main — always draft PR only
- Continue if any subagent (Oracle or Hephaestus) fails to respond — return to user
- Start the next issue before the current one has an open draft PR
- Cancel Oracle or Hephaestus mid-flight because "I already have enough context" — if you launched it, you collect it; "I have enough context" is never a valid cancellation reason

**NEVER (implementation quality):**
- Bundle multiple phases into one commit — one phase = one commit
- Let Hephaestus commit — the orchestrator always commits
- Skip phase verification before committing
- Modify Oracle's plan yourself — re-invoke Oracle if changes are needed
- Ignore the `comment` argument — pass it to Oracle as highest-priority guidance
