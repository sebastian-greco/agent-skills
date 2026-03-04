---
name: work-on-issue-omo
description: Start working on a GitHub issue in oh-my-opencode — create a linked branch, plan with Oracle agent (mandatory, synchronous), delegate implementation phases to Hephaestus one commit at a time, and open a draft PR. Ultrawork mode on by default. Never merges without explicit user approval.
---

# Work on GitHub Issue (oh-my-opencode)

Given a GitHub issue number, create a linked branch, have Oracle produce a phased plan (mandatory — no implementation starts without it), delegate each phase to Hephaestus as a separate commit, then open a draft PR. You are an orchestrator — you do not implement yourself.

> **Generic agents:** See [`work-on-issue`](../work-on-issue/SKILL.md) for a version that works with any coding agent (Claude Code, Codex, Cursor, etc.).

## Trigger

Use this skill when the user says any of:
- "work on issue #N", "start issue N", "implement issue N"
- "ulw work-on-issue N", "work-on-issue-omo N"
- `/work-on-issue N`

## Arguments

- `issue_number` (required) — GitHub issue number to work on
- `ulw` (optional, **default: ON**) — ultrawork mode: Oracle planning is mandatory and synchronous, Hephaestus implements with maximum precision, no shortcuts or assumptions allowed

> Ultrawork mode is **on by default**. To disable it explicitly, the user must say `--no-ulw`.

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

## Step 0 — Fetch Issue Details

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

## Step 1 — Create a Linked Branch

```bash
gh issue develop {issue_number} --checkout
```

Confirm:

```bash
git branch --show-current
```

> If you need a custom name: `gh issue develop {issue_number} --checkout --name "feat/issue-{issue_number}-{slug}"`

## Step 2 — Oracle Planning (MANDATORY)

Invoke Oracle synchronously with the full issue context. **Do not skip this step. Do not start implementing without Oracle's response.**

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

Produce a phased implementation plan where:
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

## Step 3 — Confirm Plan with User

Present Oracle's plan to the user and ask for confirmation:

```
Oracle's Implementation Plan — Issue #{issue_number}: {issue_title}
══════════════════════════════════════════════════════════════════

{oracle_plan}

Does this plan look right? Reply "yes" to start, or tell me what to adjust.
```

If the user requests changes, re-invoke Oracle with the updated requirements (do not modify the plan yourself).

**Do not proceed to Step 4 until the user confirms the plan.**

## Step 4 — Execute Each Phase via Hephaestus

For each phase from Oracle's confirmed plan, invoke Hephaestus:

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

## Step 5 — Push the Branch

After all phases are committed:

```bash
git push -u origin HEAD
```

## Step 6 — Open a Draft PR

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

## Step 7 — Summary

```
✅ Issue #{issue_number} ready for review.

Branch:  {branch_name}
PR:      {pr_url} (draft)
Phases:  {N} commits

Commits:
  {sha} {phase 1 commit message}
  {sha} {phase 2 commit message}
  ...

Oracle plan used: yes
ULW mode: {on/off}

Next: review the PR, request review, or ask me to address feedback with address-pr-comments-omo.
```

## Anti-Patterns

**NEVER (hard stops):**
- Start implementing before Oracle responds — no exceptions in ULW mode
- Proceed without user plan confirmation
- Implement phases yourself instead of delegating to Hephaestus
- Merge or push to main — always draft PR only
- Continue if any subagent (Oracle or Hephaestus) fails to respond — return to user

**NEVER (implementation quality):**
- Bundle multiple phases into one commit — one phase = one commit
- Let Hephaestus commit — the orchestrator always commits
- Skip phase verification before committing
- Modify the Oracle plan without re-invoking Oracle
